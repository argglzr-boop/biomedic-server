# Cumplimiento Normativo, Cifrado en Reposo y Auditoría de Seguridad

## 1. Alcance y Modelo de Amenazas

Este documento define la arquitectura de seguridad física, criptográfica y de trazabilidad para el **Biomedic Edge Server**. El nodo opera como un dispositivo perimetral dedicado a un único paciente (*single-tenant*), en el que se procesan variables fisiológicas intermitentes (glucosa capilar/intersticial, presión arterial y pulso cardíaco) con almacenamiento transaccional local.

### 1.1 Clasificación de los Datos Tratados

Los datos procesados por este nodo (glucosa capilar/intersticial, presión arterial, pulso cardíaco y toma de medicamentos) constituyen **datos personales sensibles** conforme a la **Ley Federal de Protección de Datos Personales en Posesión de los Particulares (LFPDPPP)** de México, publicada en el Diario Oficial de la Federación el 20 de marzo de 2025 y en vigor desde el 21 de marzo de 2025. La ley incluye explícitamente el **"estado de salud presente o futuro"** dentro de la categoría de datos personales sensibles.

* **No se aplica HIPAA** (*Health Insurance Portability and Accountability Act*), dado que el sistema no opera como entidad cubierta (*Covered Entity*) ni como socio comercial (*Business Associate*) bajo la jurisdicción de Estados Unidos.
* **Sí se aplican las obligaciones de la LFPDPPP**: el responsable del tratamiento debe implementar medidas de seguridad administrativas, técnicas y físicas para proteger los datos contra daño, pérdida, alteración, destrucción o acceso no autorizado, y el nivel de seguridad debe corresponder a la sensibilidad de los datos. Al tratarse de datos de salud, se aplica el nivel más alto de protección.
* La autoridad supervisora en materia de protección de datos personales en el sector privado es la **Secretaría Anticorrupción y Buen Gobierno**, a la que fueron transferidas las funciones del extinto INAI.

En consecuencia, los datos deben ser protegidos con estándares equivalentes o superiores a los exigidos para información de salud protegida, y el diseño del nodo se alinea con los requisitos de seguridad para datos sensibles de la LFPDPPP, así como con los principios técnicos de NIST como marco de referencia de buenas prácticas.

### 1.2 Modelo de Amenazas Considerado

| Vector de Ataque | Escenario de Riesgo | Contramedida Implementada |
| :--- | :--- | :--- |
| **Extracción física del medio** | Sustracción de la MicroSD para lectura directa en otro hardware. | Se cifra la partición de datos mediante LUKS2 con llaves no almacenadas en texto plano en la flash. |
| **Interceptación de tráfico** | Captura de paquetes en la red LAN local durante el transporte. | Se emplea encapsulado obligatorio punto a punto vía WireGuard (ChaCha20-Poly1305). |
| **Acceso no autorizado al SO** | Intrusión remota o escalamiento de privilegios para consultar la BD. | Se aplica aislamiento por permisos POSIX, llaves Ed25519 y auditoría de llamadas al sistema con `auditd`. |
| **Pérdida de integridad de datos** | Manipulación silenciosa o corrupción de lecturas médicas históricas. | Se verifica la integridad mediante checksums nativos de bloques con Btrfs y modo transaccional WAL en SQLite. |

---

## 2. Segregación de Datos Clínicos Sensibles

Para cumplir con el principio de mínimo privilegio y minimizar la superficie expuesta, se desacopla la capa de operación del sistema operativo de la capa de almacenamiento clínico:

```
[ MicroSD High Endurance ]
├── Partición 1 (ext4, ~3-4 GB): Sistema Base DietPi (Sin datos clínicos)
|
└── Partición 2 (Cifrada con LUKS2)
    └── Dispositivo Mapeado: /dev/mapper/secure_data
        └── Sistema de Archivos Btrfs
            ├── Subvolumen @data (Ficheros crudos, configuración cifrada)
            ├── Subvolumen @db   (Instancia SQLite de telemetría médica)
            └── Subvolumen @snapshots (Copias atómicas de solo lectura)
```

### 2.1 Identificación y Anonimización
* **Identificador Opaco:** La base de datos no indexa nombres, documentos de identidad ni direcciones. Se utiliza un `Subject-UUID` generado de forma determinista o pseudoaleatoria en el enrolamiento.
* **Separación de Contexto:** La correspondencia entre el `Subject-UUID` y la ficha del paciente reside exclusivamente en el nodo central/clínico remoto; nunca se almacena en el almacenamiento del edge server.

---

## 3. Cifrado en Reposo: Especificación LUKS2

La partición secundaria de la MicroSD aloja el contenedor criptográfico de bloques gestionado por `cryptsetup` (dm-crypt).

### 3.1 Parámetros de Cifrado
* **Formato:** LUKS2.
* **Cifrado (*Cipher*):** `aes-xts-plain64` con tamaño de clave de 512 bits (o 256 bits). El procesador ARM Cortex-A53 de la Raspberry Pi Zero 2 W dispone de instrucciones criptográficas ARMv8 por hardware, lo que permite cifrar y descifrar sin saturar los núcleos de cómputo.
* **Función de Derivación de Clave (KDF):** `argon2id` o `pbkdf2`, ajustado en iteraciones para evitar desbordar los 512 MB de memoria RAM LPDDR2 durante el proceso de apertura.

### 3.2 Ciclo de Desbloqueo Asistido (SSH / Red)
Dado que el nodo es *headless* (sin teclado ni pantalla física), se sigue el siguiente procedimiento:

1. **Arranque Base:** El nodo enciende e inicia el sistema operativo base en memoria (DietPi en partición no cifrada).
2. **Conectividad Mínima:** El demonio SSH se levanta escuchando únicamente mediante autenticación de clave pública (`Ed25519`).
3. **Inyección de Llave / Frase de Paso:**
   - Vía Ansible o sesión interactiva del administrador por red interna/VPN, se ejecuta:
     ```bash
     cryptsetup open /dev/mmcblk0p2 secure_data
     mount -o subvol=@db,noatime,compress=zstd:1 /dev/mapper/secure_data /opt/biomedic/data
     systemctl start biomedic-ingestion.service
     ```
4. **Protección en Apagado:** Al reiniciar o perder energía de forma imprevista, la clave desaparece de la memoria volátil y la partición queda sellada.

---

## 4. Trazabilidad y Auditoría de Seguridad con `auditd`

En sistemas con 512 MB de RAM y tarjetas de memoria flash, un demonio `auditd` con reglas genéricas provocaría saturación de I/O y agotamiento de memoria. Por tanto, la auditoría se acota exclusivamente a accesos sobre los archivos de telemetría médica y a modificaciones de la configuración del servicio.

### 4.1 Reglas Críticas de Auditoría (`/etc/audit/rules.d/biomedic.rules`)

```ini
## Descartar eventos innecesarios del sistema para ahorrar RAM
-f 1
-b 128

## Monitorear intentos de apertura/lectura no autorizados de la base de datos médica
-w /opt/biomedic/data/telemetry.db -p wa -k phi_db_modification
-w /opt/biomedic/data/telemetry.db-wal -p wa -k phi_wal_modification

## Monitorear binarios del servicio de ingesta médica
-w /usr/local/bin/biomedic-collector -p x -k collector_execution

## Monitorear intentos de acceso a llaves criptográficas locales
-w /etc/wireguard/wg0.conf -p rwa -k vpn_credentials_access
```

### 4.2 Almacenamiento y Ciclo de Vida de Logs

* **Logs Operativos (DietPi RAMlog):** Los logs del kernel y syslog residen en un buffer en RAM (`/var/log` en tmpfs), lo que evita el desgaste prematuro de las celdas flash de la MicroSD.
* **Logs de Seguridad:** Los eventos etiquetados por `auditd` (`phi_db_modification`) se registran con retención circular acotada (máximo 10 MB rotados localmente) y quedan listos para ser transmitidos periódicamente por el enlace seguro.

---

## 5. Matriz de Alineación con LFPDPPP y Principios Técnicos de Referencia

| Requisito Normativo (LFPDPPP) | Implementación en Nodo Perimetral | Verificación Técnica |
| --- | --- | --- |
| **Medidas de Seguridad para Datos Sensibles** (la ley exige medidas administrativas, técnicas y físicas para proteger datos personales sensibles) | Se deshabilita el acceso por contraseña; solo se admite autenticación mediante llaves SSH Ed25519; los servicios se ejecutan bajo usuarios sin privilegios de root (`uid/gid` dedicado); se emplea un contenedor LUKS2 sobre la partición que aloja la base de datos SQLite y los snapshots; se establece un túnel WireGuard punto a punto con cifrado simétrico ChaCha20 y autenticación Poly1305. | `sshd_config`, `sudoers` auditado, `lsblk`, `cryptsetup status`, `wg show`. |
| **Principio de Confidencialidad** (obligación de confidencialidad para quienes tratan datos, subsistente tras finalizar la relación) | Sistema *single-tenant* con autenticación exclusiva por llave pública; ausencia de indexación de nombres, documentos de identidad o direcciones en la base de datos; correspondencia `Subject-UUID` ↔ ficha del paciente exclusivamente en el nodo central/clínico remoto. | `sshd_config`, esquema de la base de datos SQLite. |
| **Control de Integridad** (protección contra alteración o destrucción indebida) | Se utiliza el sistema de archivos Btrfs con sumas de verificación CRC32C/xxhash automáticas por bloque; SQLite en modo WAL. | `btrfs scrub status`. |
| **Registro de Auditoría** (mecanismos que registren y permitan examinar la actividad en sistemas que contienen datos sensibles) | Se aplica auditoría granular del kernel vía `auditd` sobre el descriptor de archivo de la base de datos y sobre los binarios del servicio de ingesta. | `ausearch -k phi_db_modification`. |
| **Notificación de Incidentes de Seguridad** (obligación de notificar al titular ante vulneraciones que afecten de forma significativa sus derechos) | Monitoreo continuo de logs de `auditd` y verificación periódica de integridad mediante `btrfs scrub`; los eventos se transmiten por el enlace seguro para su revisión. | `ausearch`, `btrfs scrub status`, revisión periódica de logs. |

---

## 6. 🔗 Referencias

* [LFPDPPP — Ley Federal de Protección de Datos Personales en Posesión de los Particulares (DOF, 20 de marzo de 2025)](https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf)

* [NIST — SP 800-111, Guide to Storage Encryption Technologies for End User Devices](https://www.nist.gov/publications/guide-storage-encryption-technologies-end-user-devices)

* [NVD — CVE-2025-58356 (cryptsetup/LUKS2)](https://nvd.nist.gov/vuln/detail/cve-2025-58356)
