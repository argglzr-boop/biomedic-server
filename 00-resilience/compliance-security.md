# Cumplimiento Normativo, Cifrado en Reposo y Auditoría de Seguridad

## 1. Alcance y Modelo de Amenazas

Este documento define la arquitectura de seguridad física, criptográfica y de trazabilidad para el **Biomedic Edge Server**. El nodo opera como un dispositivo perimetral dedicado a un único paciente (*single-tenant*), en el que se procesan variables fisiológicas intermitentes (glucosa capilar/intersticial, presión arterial y pulso cardíaco) con almacenamiento transaccional local.

### 1.1 Clasificación de los Datos Tratados

Los datos procesados por este nodo **no se clasifican estrictamente como PHI (*Protected Health Information*)** en el sentido legal del término, dado que el sistema no actúa como entidad cubierta (*Covered Entity*) ni como socio comercial (*Business Associate*) bajo HIPAA, y no media una relación clínica regulada. No obstante, **se consideran datos críticos y totalmente confidenciales**: constituyen información de salud personal de carácter sensible cuya divulgación, alteración o pérdida podría afectar directamente la integridad y la privacidad del paciente. Por consiguiente, **deben ser protegidos con estándares equivalentes o superiores a los exigidos para PHI**, y el diseño del nodo se alinea voluntariamente con los principios técnicos de HIPAA y RGPD como marco de referencia de buenas prácticas.

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

## 5. Matriz de Alineación con Principios HIPAA y RGPD

| Requisito Normativo | Implementación en Nodo Perimetral | Verificación Técnica |
| --- | --- | --- |
| **Control de Acceso (§ 164.312(a)(1))** | Se deshabilita el acceso por contraseña; solo se admite autenticación mediante llaves SSH Ed25519; los servicios se ejecutan bajo usuarios sin privilegios de root (`uid/gid` dedicado). | `sshd_config`, `sudoers` auditado. |
| **Cifrado en Reposo (§ 164.312(a)(2)(iv))** | Se emplea un contenedor LUKS2 sobre la partición que aloja la base de datos SQLite y los snapshots. | `lsblk`, `cryptsetup status`. |
| **Cifrado en Tránsito (§ 164.312(e)(1))** | Se establece un túnel WireGuard punto a punto con cifrado simétrico ChaCha20 y autenticación Poly1305. | `wg show`. |
| **Control de Integridad (§ 164.312(c)(1))** | Se utiliza el sistema de archivos Btrfs con sumas de verificación CRC32C/xxhash automáticas por bloque; SQLite en modo WAL. | `btrfs scrub status`. |
| **Registro de Auditoría (§ 164.312(b))** | Se aplica auditoría granular del kernel vía `auditd` sobre el descriptor de archivo de la base de datos. | `ausearch -k phi_db_modification`. |

---
