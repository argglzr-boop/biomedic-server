# 📐 001 — Visión General y Topología del Nodo Biomédico

> **Siguiente**: [compliance-security →](002-compliance-security.md)

Descripción arquitectónica, modelo por capas y topología funcional del servidor biomédico perimetral desplegado sobre Raspberry Pi Zero 2 W con DietPi.

---

## 1. 📋 Resumen del Sistema

| Parámetro | Definición |
| --- | --- |
| **Propósito** | Captura, persistencia local y tránsito seguro de telemetría médica |
| **Arquitectura de Cómputo** | Edge Computing ligero (ARM Cortex-A53 quad-core, 512 MB LPDDR2) |
| **Entorno Base** | DietPi (Debian minimal optimizado para memoria reducida) |
| **Estación de Gestión** | macOS (administración vía OpenSSH / Ansible) |
| **Destino de Persistencia** | MicroSD High Endurance (Partición dual: ext4 OS + ext4 datos con `noatime`) |

---

## 2. 🏗️ Las 4 Capas del Nodo Biomédico

Para garantizar contención de fallos, modularidad y bajo consumo de recursos en un sistema con restricciones estrictas de hardware y almacenamiento en medio único de alta durabilidad, el diseño se organiza en cuatro capas desacopladas:

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
│  CAPA 1: HARDWARE & ALMACENAMIENTO PERSISTENTE (Pi Zero 2W) │
│  MicroSD High Endurance: ext4 (/boot, OS) + ext4 (@data)    │
└─────────────────────────────────────────────────────────────┘

```

### 2.1 Hardware y Almacenamiento Persistente

* **MicroSD High/Max Endurance:** Unidad flash de grado industrial o videovigilancia (NAND de alta resistencia con controlador avanzado de *wear leveling*).
* **Esquema de Particionado y ext4:**
* Partición base para el sistema operativo optimizada para lecturas.
* Partición dedicada de datos en **ext4** con opción de montaje `noatime` para reducir ciclos de escritura física.
* Descarte de Btrfs por amplificación de escritura (CoW) y desgaste de la MicroSD, y de LVM por capas intermedias y consumo innecesario de metadatos en hardware limitado.



### 2.2 Sistema Operativo y Control de Estado

* **DietPi Core:** Reduce el *footprint* de RAM a menos de 60 MB en arranque, manteniendo bitácoras volátiles en memoria RAM (`dietpi-ramlog`) para mitigar escrituras repetitivas sobre la MicroSD.
* **Trazabilidad de Configuración:** Control de versiones del árbol `/etc` mediante herramientas de auditoría y Git para registrar cualquier variación en el sistema base.

### 2.3 Red Segura y Conectividad

* **Perímetro Restrictivo:** Rechazo total de tráfico entrante por defecto mediante UFW, habilitando únicamente los puertos indispensables (SSH por llaves, VPN).
* **Túnel Seguro de Telemetría (WireGuard):** Cifrado ChaCha20-Poly1305 de bajo impacto de CPU para el enlace directo entre la Pi Zero 2 W y la infraestructura central (VPS o NAS clínico), blindando los datos en tránsito.

### 2.4 Servicios y Persistencia de Telemetría

* **Motor Relacional Transaccional (SQLite):** Ingesta de variables fisiológicas y metadatos clínicos con journaling optimizado (modo WAL), persistiendo en la partición de datos ext4 configurada.
* **Telemetría del Nodo (Monitoring):** Exposición de métricas de salud (temperatura del SoC, uso de swap, I/O wait) para alertar antes de cualquier degradación térmica o de memoria.

---

## 3. 🌐 Topología de Despliegue

```text
 [ Sensores / Equipos Médicos ]
               │
               ▼ (Red Local / UART / BLE / I2C)
 ┌────────────────────────────────────────────────────────┐
 │            Raspberry Pi Zero 2 W (DietPi)              │
 │  ┌──────────────────────────────────────────────────┐  │
 │  │      MicroSD High Endurance (NAND Industrial)    │  │
 │  │  ├── Partición OS (ext4 / dietpi-ramlog)         │  │
 │  │  └── Partición Datos (ext4 con noatime)          │  │
 │  └──────────────────────────────────────────────────┘  │
 └───────────────────────────┬────────────────────────────┘
                             │
                             ▼ (WireGuard VPN / WAN)
      [ Servidor Central / VPS / Cloud / NAS Clínico ]

```

---

## 5. 🔗 Referencias

* [DietPi Documentación Oficial](https://dietpi.com/docs/)
* [DietPi Optimización de Almacenamiento RAM-Log](https://www.google.com/search?q=https://dietpi.com/docs/software/system_stats/%23dietpi-ramlog)
* [ext4 Documentation](https://www.kernel.org/doc/html/latest/filesystems/ext4/)


