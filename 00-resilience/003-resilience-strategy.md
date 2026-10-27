# 003. Estrategia de Resiliencia, Continuidad Operativa y Respaldo 3-2-1

> **Navegación**: [← compliance-security](002-compliance-security.md) | [hardware →](../01-setup/01.1-hardware.md)

Definición de la arquitectura de respaldo, recuperación ante desastres y continuidad operativa del nodo biomédico perimetral, basada en la regla 3-2-1, en dumps lógicos transaccionales de la base de datos y en IaC (repositorio Git) para configuraciones y servicios.

## 1. Definición de Objetivos de Recuperación (RPO / RTO)

En el contexto de un nodo biomédico perimetral monopaciente desplegado sobre una **Raspberry Pi Zero 2 W** con **64 GB MicroSD High Endurance**, los umbrales de tolerancia operativa son:

| Parámetro | Objetivo | Justificación Técnica |
| :--- | :--- | :--- |
| **RPO (*Recovery Point Objective*)** | **≤ 30 minutos** (Local)<br>**≤ 24 horas** (Offsite) | SQLite opera en modo WAL con `synchronous = NORMAL`, minimizando escrituras pero persistiendo transacciones atómicas al cerrarse cada toma. La réplica externa diaria de dumps lógicos absorbe fallas catastróficas del hardware. |
| **RTO (*Recovery Time Objective*)** | **≤ 3 minutos** (Reinicio)<br>**≤ 30 minutos** (Fallo total de hardware) | El arranque base de DietPi y desbloqueo de LUKS toma menos de 180 segundos. La reconstrucción de nodo completo se realiza de forma desatendida mediante Ansible e inyección del último dump lógico de la base de datos. |

---

## 2. Esquema de Particionado y Almacenamiento de Datos

Sobre el contenedor cifrado LUKS (`/dev/mapper/secure_data`), el sistema de archivos **ext4** se monta con opción `noatime` para reducir escrituras de metadatos en la MicroSD.

```

/dev/mapper/secure_data (ext4, ~58 GB disponibles, noatime)
└── /opt/biomedic/data      (Base de datos SQLite, configuraciones locales, certificados, scripts)

```

### 2.1 Opciones de Montaje Críticas (`/etc/fstab`)
```text
UUID=<UUID_LUKS_MAPPER> /opt/biomedic/data ext4 noatime 0 0

```

* **`noatime`:** Elimina escrituras por metadatos de lectura, prolongando la vida de la tarjeta flash.

---

## 3. Consistencia Transaccional: SQLite + Dumps Lógicos

Para garantizar que un respaldo no capture un estado transaccional corrupto o a medio escribir en SQLite, el proceso de respaldo sigue un pipeline síncrono basado en volcados lógicos:

1. **Flush del WAL:** Se fuerza la consolidación del journal de transacciones a la base de datos principal:
```bash
sqlite3 /opt/biomedic/data/telemetry.db "PRAGMA wal_checkpoint(TRUNCATE);"

```


2. **Generación del Dump Lógico Transaccional (comprimido):**
```bash
sqlite3 /opt/biomedic/data/telemetry.db ".backup '/opt/biomedic/backups/db-$(date +%Y%m%d_%H%M%S).sql'"
gzip /opt/biomedic/backups/db-$(date +%Y%m%d_%H%M%S).sql

```


3. **Liberación:** El servicio de ingesta reanuda escrituras normales inmediatamente (el dump se realiza sobre una copia consistente).

---

## 4. Política de Retención Local de Dumps

Dado el medio físico de 64 GB, se aplica una rotación automatizada por temporizadores `systemd` para prevenir el agotamiento del espacio:

* **Dumps Cada 6 Horas:** Retención de las últimas 24 horas (máximo 4 dumps activos).
* **Dumps Diarios:** Retención de los últimos 7 días (máximo 7 dumps activos).
* **Purga Automática (FIFO):** Tarea programada que elimina dumps con más de 7 días de antigüedad mediante `find` + `rm`.

```
[Cron/Timer] ──> Dump c/6h ──> Retener 4 (24h)
             ──> Dump diario ──> Retener 7 días ──> Purga de >7 días

```

---

## 5. Arquitectura de Respaldo 3-2-1 Adaptada al Edge

```
+-------------------------------------------------------------+
| COPIA 1 (Primaria Activa)                                   |
| /opt/biomedic/data/telemetry.db (ext4 sobre LUKS en MicroSD)|
+-------------------------------------------------------------+
                              │
                  (Dump lógico transaccional)
                              ▼
+-------------------------------------------------------------+
| COPIA 2 (Local en Frío / RTO Inmediato)                    |
| /opt/biomedic/backups/db-YYYYMMDD_HHMMSS.sql.gz             |
+-------------------------------------------------------------+
                              │
        (Túnel WireGuard + rsync/scp del dump comprimido)
                              ▼
+-------------------------------------------------------------+
| COPIA 3 (Offsite Remota / Recuperación de Desastres)        |
| Servidor Central / NAS Médico (Almacenamiento Redundante)  |
+-------------------------------------------------------------+

```

### 5.1 Rutina de Sincronización Remota Diaria (Offsite)

Cada medianoche, una tarea programada envía el dump lógico comprimido del día a través del túnel VPN WireGuard:

```bash
# Envío seguro del dump comprimido a través del túnel VPN WireGuard
rsync -avz -e "ssh -i /etc/wireguard/backup_key" \
  /opt/biomedic/backups/db-$(date +%Y%m%d)*.sql.gz \
  user@10.100.0.1:/srv/backup/edge-node-01/

```

---

## 6. Procedimiento de Mitigación ante Corrupción o Fallo

### Escenario A: Fallo Lógico o Corrupción de Datos

1. Detener el servicio: `systemctl stop biomedic-ingestion`.
2. Mover la base corrupta: `mv /opt/biomedic/data/telemetry.db /opt/biomedic/data/telemetry.db_corrupt`.
3. Restaurar el último dump íntegro: `gunzip -c /opt/biomedic/backups/db-YYYYMMDD_HHMMSS.sql.gz | sqlite3 /opt/biomedic/data/telemetry.db`.
4. Iniciar servicio: `systemctl start biomedic-ingestion`.

### 6.1 Escenario B: Muerte Física de la MicroSD

1. Insertar una nueva MicroSD High Endurance (64 GB).
2. Flashear la **Golden Image** del proyecto (aprovisionamiento base documentado) y ejecutar el playbook inicial de Ansible (`site.yml`).
3. Recrear el contenedor LUKS sobre la partición de datos y formatear en ext4 con `noatime`.
4. Restaurar el último dump lógico remoto desde el servidor central vía `rsync`/`scp` e importarlo con `sqlite3`.

## 7. Referencias

* [ext4 Documentation](https://www.kernel.org/doc/html/latest/filesystems/ext4/)
* [SQLite - Checkpoint a database](https://www.sqlite.org/c3ref/wal_checkpoint.html)
* [SQLite - Backup API](https://www.sqlite.org/backup.html)
* [SentinelOne - ¿Qué es la estrategia de respaldo 3-2-1?](https://www.sentinelone.com/es/cybersecurity-101/cybersecurity/3-2-1-backup-strategy/)
* [Arch Wiki - Systemd/Timers](https://wiki.archlinux.org/title/Systemd/Timers)
* [WireGuard - Official Documentation](https://www.wireguard.com/)
* [Raspberry Pi Documentation - SD Cards and Storage](https://www.raspberrypi.com/documentation/computers/getting-started.html#sd-cards)
