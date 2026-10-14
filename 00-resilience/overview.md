# 📐 00 — Visión General y Topología del Nodo Biomédico

Descripción arquitectónica, modelo por capas y topología funcional del servidor biomédico perimetral desplegado sobre Raspberry Pi Zero 2 W con DietPi.

---

## 📋 Resumen del Sistema

| Parámetro | Definición |
| --- | --- |
| **Propósito** | Captura, persistencia local y tránsito seguro de telemetría médica |
| **Arquitectura de Cómputo** | Edge Computing ligero (ARM Cortex-A53 quad-core, 512 MB LPDDR2) |
| **Entorno Base** | DietPi (Debian minimal optimizado para memoria reducida) |
| **Estación de Gestión** | macOS (administración vía OpenSSH / Ansible) |
| **Destino de Persistencia** | USB externa formateada en Btrfs (aislamiento de MicroSD) |

---

## 🏗️ Las 4 Capas del Nodo Biomédico

Para garantizar contención de fallos, modularidad y bajo consumo de recursos en un sistema con restricciones estrictas de hardware, el diseño se organiza en cuatro capas desacopladas:

```text
┌─────────────────────────────────────────────────────────────┐
│  CAPA 4: APLICACIÓN & TELEMETRÍA (SQLite, Node-Exporter)    │
│  Servicios de ingesta, base de datos e instrumentación      │
├─────────────────────────────────────────────────────────────┤
│  CAPA 3: RED SEGURA & CONECTIVIDAD (WireGuard, SSH, UFW)     │
│  Túneles VPN cifrados y segmentación de puertos perimetral  │
├─────────────────────────────────────────────────────────────┤
│  CAPA 2: SISTEMA OPERATIVO & CONFIGURACIÓN (DietPi, etc)    │
│  Kernel afinado, RAM-disk para logs, control de estado /etc │
├─────────────────────────────────────────────────────────────┤
│  CAPA 1: HARDWARE & ALMACENAMIENTO PERSISTENTE (Pi + USB)   │
│  MicroSD (OS base) + USB Btrfs (@db_dumps, @dietpi_backup)  │
└─────────────────────────────────────────────────────────────┘

```

### Capa 1: Hardware y Almacenamiento Persistente

* **MicroSD:** Dedicada exclusivamente al arranque del sistema operativo y binarios en modo casi solo lectura.
* **USB Externa (Btrfs):** Desacopla las operaciones de escritura continuas de la MicroSD, gestionando las transacciones de bases de datos y la creación de snapshots inmutables mediante Copy-on-Write (CoW).

### Capa 2: Sistema Operativo y Control de Estado

* **DietPi Core:** Reduce el *footprint* de RAM a menos de 60 MB en arranque, redirigiendo bitácoras volátiles a RAM-disk (`dietpi-ramlog`) para evitar la degradación de medios flash.
* **Trazabilidad de Configuración:** Control de versiones del árbol `/etc` para registrar cualquier variación en el comportamiento del nodo a nivel de sistema operativo.

### Capa 3: Red Segura y Conectividad

* **Perímetro Restrictivo:** Rechazo total de tráfico entrante por defecto, habilitando únicamente los puertos indispensables.
* **Túnel Seguro de Telemetría (WireGuard):** Cifrado ChaCha20-Poly1305 para el enlace entre el nodo local y la infraestructura central (VPS o NAS de ingestión), aislando los datos clínicos de redes locales inseguras.

### Capa 4: Servicios y Persistencia de Telemetría

* **Motor Relacional Transaccional (SQLite):** Almacenamiento de variables fisiológicas y metadatos clínicos, operando con tablas y volcados vinculados directamente al almacenamiento USB externo.
* **Telemetría del Nodo (Monitoring):** Exposición de métricas de salud del nodo (temperatura del SoC, uso de swap, I/O) para supervisión remota.

---

## 🌐 Topología de Despliegue

```text
 [ Sensores / Equipos Médicos ]
               │
               ▼ (Red Local / USB Serial)
 ┌───────────────────────────────────────────────┐
 │       Raspberry Pi Zero 2 W (DietPi)          │
 │  ┌─────────────────┐    ┌──────────────────┐  │
 │  │ MicroSD (OS Base)│    │ USB Btrfs (CoW)  │  │
 │  └─────────────────┘    └──────────────────┘  │
 └──────────────────────┬────────────────────────┘
                        │
                        ▼ (WireGuard VPN / WAN)
 [ Servidor Central / VPS / Cloud / NAS ]

```

---

## 🔗 Referencias

* [DietPi Notas de lanzamiento](https://dietpi.com/docs/releases/v7_1/#dietpi-tools-new-notable-updates)
* [DietPi Configuración del sistema](https://dietpi.com/docs/dietpi_tools/system_configuration/#format-filesystem-types-ext4-default)
* [Btrfs Architecture Guide](https://btrfs.readthedocs.io/)

---

> **Siguiente**: [resilience-strategy.md — Estrategia de Resiliencia y Recuperación →](https://www.google.com/search?q=resilience-strategy.md)