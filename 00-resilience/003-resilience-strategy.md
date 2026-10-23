# Estrategia de Resiliencia, Continuidad Operativa y Respaldo 3-2-1

> **Navegación**: [← compliance-security](002-compliance-security.md) | [hardware →](../01-setup/01.1-hardware.md)

Definición de la arquitectura de respaldo, recuperación ante desastres y continuidad operativa del nodo biomédico perimetral, basada en la regla 3-2-1 y en la verificación periódica de integridad de los datos clínicos sensibles.

## 1. Definición de Objetivos de Recuperación (RPO / RTO)

En el contexto de un nodo biomédico perimetral monopaciente desplegado sobre una **Raspberry Pi Zero 2 W** con **64 GB MicroSD High Endurance**, los umbrales de tolerancia operativa son:

| Parámetro | Objetivo | Justificación Técnica |
| :--- | :--- | :--- |
| **RPO (*Recovery Point Objective*)** | **≤ 30 minutos** (Local)<br>**≤ 24 horas** (Offsite) | SQLite opera en modo WAL con `synchronous = NORMAL`, minimizando escrituras pero persistiendo transacciones atómicas al cerrarse cada toma. La réplica externa diaria absorbe fallas catastróficas del hardware. |
| **RTO (*Recovery Time Objective*)** | **≤ 3 minutos** (Reinicio)<br>**≤ 30 minutos** (Fallo total de hardware) | El arranque base de DietPi y desbloqueo de LUKS toma menos de 180 segundos. La reconstrucción de nodo completo se realiza de forma desatendida mediante Ansible e inyección de la última réplica Btrfs. |

---

## 2. Topología de Subvolúmenes Btrfs

Sobre el contenedor cifrado LUKS (`/dev/mapper/secure_data`), el sistema de archivos Btrfs se particiona lógicamente en subvolúmenes desacoplados:


```

/dev/mapper/secure_data (Btrfs, ~58 GB disponibles)
├── @data       -> /opt/biomedic/data      (Configuraciones locales, certificados, scripts)
├── @db         -> /opt/biomedic/db        (telemetry.db, telemetry.db-wal)
└── @snapshots  -> /opt/biomedic/snapshots (Puntos de restauración de solo lectura)

```

### 2.1 Opciones de Montaje Críticas (`/etc/fstab`)
```text
UUID=<UUID_LUKS_MAPPER> /opt/biomedic/db btrfs subvol=@db,noatime,compress=zstd:1,space_cache=v2 0 0

```

* **`noatime`:** Elimina escrituras por metadatos de lectura, prolongando la vida de la tarjeta flash.
* **`compress=zstd:1`:** Compresión transparente ligera que reduce el I/O en flash sin impactar perceptiblemente los 4 núcleos Cortex-A53.
* **`space_cache=v2`:** Asignación de espacio optimizada y tolerante a cortes repentinos de energía.

---

## 3. Consistencia Transaccional: SQLite + Snapshots

Para garantizar que un snapshot no capture un estado transaccional corrupto o a medio escribir en SQLite, el proceso de respaldo sigue un pipeline síncrono:

1. **Flush del WAL:** Se fuerza la consolidación del journal de transacciones a la base de datos principal:
```bash
sqlite3 /opt/biomedic/db/telemetry.db "PRAGMA wal_checkpoint(TRUNCATE);"

```


2. **Generación del Snapshot Atómico (-r):**
```bash
btrfs subvolume snapshot -r /opt/biomedic/db /opt/biomedic/snapshots/db-$(date +%Y%m%d_%H%M%S)

```


3. **Liberación:** El servicio de ingesta reanuda escrituras normales inmediatamente (el snapshot CoW toma milisegundos).

---

## 4. Política de Retención Local de Snapshots

Dado el medio físico de 64 GB, se aplica una rotación automatizada por temporizadores `systemd` para prevenir la sobrecarga de metadatos en Btrfs:

* **Snapshots Cada 6 Horas:** Retención de las últimas 24 horas (máximo 4 snapshots activos).
* **Snapshots Diarios:** Retención de los últimos 7 días (máximo 7 snapshots activos).
* **Purga Automática (FIFO):** Tarea programada que elimina snapshots con más de 7 días de antigüedad mediante `btrfs subvolume delete`.

```
[Cron/Timer] ──> Snapshot c/6h ──> Retener 4 (24h)
             ──> Snapshot diario ──> Retener 7 días ──> Purga de >7 días

```

---

## 5. Arquitectura de Respaldo 3-2-1 Adaptada al Edge

```
+-------------------------------------------------------------+
| COPIA 1 (Primaria Activa)                                   |
| /opt/biomedic/db/telemetry.db (Subvolumen @db en MicroSD)   |
+-------------------------------------------------------------+
                              │
                  (Snapshot CoW local)
                              ▼
+-------------------------------------------------------------+
| COPIA 2 (Local en Frío / RTO Inmediato)                    |
| /opt/biomedic/snapshots/db-YYYYMMDD_HHMMSS (Read-Only)      |
+-------------------------------------------------------------+
                              │
        (Túnel WireGuard + btrfs send/receive incremental)
                              ▼
+-------------------------------------------------------------+
| COPIA 3 (Offsite Remota / Recuperación de Desastres)        |
| Servidor Central / NAS Médico (Almacenamiento Redundante)  |
+-------------------------------------------------------------+

```

### 5.1 Rutina de Sincronización Remota Diaria (Offsite)

Cada medianoche, una tarea programada envía únicamente los bloques diferenciales respecto al día anterior:

```bash
# Envío diferencial seguro a través del túnel VPN WireGuard
btrfs send -p /opt/biomedic/snapshots/db-PREV /opt/biomedic/snapshots/db-CURRENT \
  | ssh -i /etc/wireguard/backup_key user@10.100.0.1 "btrfs receive /srv/backup/edge-node-01/"

```

---

## 6. Procedimiento de Mitigación ante Corrupción o Fallo

### Escenario A: Fallo Lógico o Corrupción de Datos

1. Detener el servicio: `systemctl stop biomedic-ingestion`.
2. Renombrar el subvolumen corrupto: `mv /opt/biomedic/db /opt/biomedic/db_corrupt`.
3. Restaurar snapshot íntegro: `btrfs subvolume snapshot /opt/biomedic/snapshots/db-YYYYMMDD_HHMMSS /opt/biomedic/db`.
4. Iniciar servicio: `systemctl start biomedic-ingestion`.

### 6.1 Escenario B: Muerte Física de la MicroSD

1. Insertar una nueva MicroSD High Endurance (64 GB).
2. Flashear DietPi base y ejecutar el playbook inicial de Ansible (`site.yml`).
3. Recrear el contenedor LUKS y el esquema de subvolúmenes Btrfs.
4. Restaurar la última réplica remota desde el servidor central vía `btrfs receive`.

## 7. Referencias

* [Btrfs Wiki - Subvolumes and Snapshots](https://btrfs.readthedocs.io/en/latest/Subvolumes.html)
* [SQLite - Checkpoint a database](https://www.sqlite.org/c3ref/wal_checkpoint.html)
* [Btrfs Send/Receive for Incremental Off-Site Backups](https://shop.zimaspace.com/blogs/support-tips/btrfs-send-receive-incremental-offsite-backups)
* [SentinelOne - ¿Qué es la estrategia de respaldo 3-2-1?](https://www.sentinelone.com/es/cybersecurity-101/cybersecurity/3-2-1-backup-strategy/)
* [Arch Wiki - Btrfs](https://wiki.archlinux.org/title/Btrfs)
* [Arch Wiki - Systemd/Timers](https://wiki.archlinux.org/title/Systemd/Timers)
* [WireGuard - Official Documentation](https://www.wireguard.com/)
* [Raspberry Pi Documentation - SD Cards and Storage](https://www.raspberrypi.com/documentation/computers/getting-started.html#sd-cards)
