# Biomedic Edge Server

Infraestructura como Código (IaC) para el aprovisionamiento, securización y operación de un nodo perimetral biomédico de bajo consumo basado en **DietPi**, orquestado mediante **Ansible** y respaldado localmente mediante **dumps lógicos de la base de datos** y **Golden Image** sobre **MicroSD High Endurance**.

> **Estado del Proyecto:** 🟡 **En desarrollo — Fases 00 (Arquitectura) y 01 (Setup Base y Bootstrap) completadas. Fase 02 (Endurecimiento Perimetral) en curso.**
> El código IaC (Ansible) y los scripts de aprovisionamiento se encuentran en proceso de implementación modular conforme al roadmap.

---

## 📋 Resumen del Proyecto

| Parámetro | Definición |
| :--- | :--- |
| **Plataforma Hardware** | Raspberry Pi Zero 2 W (SoC ARM Cortex-A53 quad-core, 512 MB LPDDR2) |
| **Almacenamiento Primario** | MicroSD High Endurance (Partición dual: ext4 OS + ext4 datos con `noatime`) |
| **Sistema Operativo** | DietPi (Debian minimal con optimización de RAM-log y kernel afinado) |
| **Motor de Automatización**| Ansible (roles desacoplados, modo idempotente y trazable) |
| **Persistencia Transaccional**| SQLite (modo WAL optimizado para flash y compresión zstd) |
| **Conectividad Segura** | Túnel WireGuard (ChaCha20-Poly1305) y firewall estricto (UFW) |

---

## 🗺️ Roadmap y Checklist de Implementación

### 📐 00 — Arquitectura y Visión Transversal
- [x] **[`00-resilence/overview.md`](00-resilience/001-overview.md)**: Topología funcional del nodo biomédico y arquitectura de 4 capas.
- [x] **[`00-resilence/compliance-security.md`](00-resilience/002-compliance-security.md)**: Cifrado en reposo granular (LUKS sobre la partición de datos sensibles en ext4), segregación de datos PHI y diseño de auditoría.
- [x] **[`00-resilence/resilience-strategy.md`](00-resilience/003-resilience-strategy.md)**: Objetivos RPO/RTO, modelo de respaldo basado en dumps lógicos transaccionales e IaC, y retención de los mismos.

---

### 🟢 01 — Setup Base y Bootstrap
- [x] **[`01-setup/01.1-hardware.md`](01-setup/01.1-hardware.md)**: BOM, módulo RTC (DS3231), consumo eléctrico y disipación térmica.
- [x] **[`01-setup/01.2-installation.md`](01-setup/01.2-installation.md)**: Flasheo de MicroSD High Endurance y configuración `dietpi.txt` headless.
- [x] **[`01-setup/01.3-provisioning.md`](01-setup/01.3-provisioning.md)**: *(Aquí deberías poner la descripción de este archivo nuevo, ej: Aprovisionamiento inicial)*
- [x] **[`01-setup/01.4-network.md`](01-setup/01.4-network.md)**: IP estática/DHCP reservado, mitigación de drift temporal (NTP/Chrony + RTC).
- [x] **[`01-setup/01.5-golden-image.md`](01-setup/01.5-golden-image.md)**: *(Aquí deberías poner la descripción de este archivo nuevo, ej: Creación de imagen maestra)*
- [x] **[`01-setup/01.6-bootstrap-ansible.md`](01-setup/01.6-bootstrap-ansible.md)**: Inyección de llave SSH temporal, control node handoff e inventario inicial.

---

### 🟡 02 — Endurecimiento Perimetral (Hardening)
- [ ] **`02-security/ssh-hardening.md`**: Llaves Ed25519, cambio de puerto, deshabilitación de root y contraseñas.
- [ ] **`02-security/firewall-ufw.md`**: Políticas de descarte por defecto (`DROP`) y segmentación LAN médica.
- [ ] **`02-security/fail2ban.md`**: Jaulas personalizadas para SSH y mitigación de fuerza bruta.
- [ ] **`02-security/auditd.md`**: Reglas de auditoría del kernel para accesos a datos clínicos y binarios sensibles.

---

### 💾 03 — Almacenamiento Seguro y Persistente
- [ ] **`03-storage/luks-encryption.md`**: Cifrado granular con LUKS (dm-crypt) aplicado exclusivamente sobre la partición física de datos sensibles (base de datos), formateada en ext4 sobre el mapeador (`/dev/mapper/...`) para no degradar la CPU en lecturas del sistema operativo.
- [ ] **`03-storage/ext4-data-partition.md`**: Particionado físico directo sobre la MicroSD (`/dev/mmcblk0`) con ext4 y opción `noatime`; descarte de LVM por capas intermedias y de Btrfs/XFS por amplificación de escritura (CoW) y huella de RAM/CPU en 512 MB.

---

### ⚙️ 04 — Servicios del Nodo Biomédico
- [ ] **`04-services/SQLite.md`**: Ingesta de variables fisiológicas y persistencia en la partición de datos ext4 (`noatime`).
- [ ] **`04-services/wireguard.md`**: Enlace VPN cifrado punto a punto para telemetría remota segura.
- [ ] **`04-services/monitoring.md`**: Despliegue de `node-exporter` para telemetría de hardware (temperatura, I/O, swap).
- [ ] **`04-services/grafana.md`**: Dashboards de salud del nodo y RBAC para personal médico/técnico.

---

### 🛡️ 05 — Resiliencia y Continuidad Operativa
- [ ] **`05-resilience/db-backups.md`**: Dumps lógicos transaccionales (volcados `.sql` comprimidos) previos a mantenimientos; descarte de snapshots locales de disco en favor de IaC para configuraciones y de la Golden Image para el aprovisionamiento base.
- [ ] **`05-resilience/etckeeper.md`**: Control de versiones de `/etc` automatizado con Git.
- [ ] **`05-resilience/remote-sync.md`**: Réplica periódica incremental cifrada hacia NAS o VPS central.

---

### 🤖 06 — Operaciones y Automatización
- [ ] **`06-operations/ansible-playbooks.md`**: Catálogo de roles, tags de ejecución (`--tags security,storage`) y dry-run.
- [ ] **`06-operations/maintenance-cron.md`**: Tareas programadas de mantenimiento de ext4, trim y salud del storage.
- [ ] **`06-operations/disaster-recovery-runbook.md`**: Protocolo paso a paso ante pérdida física del hardware o corrupción.

---

### 🔧 07 — Diagnóstico
- [ ] **`07-troubleshooting.md`**: Matriz de resolución de incidentes (degradación I/O, fallas de sincronización RTC, cuellos de botella RAM).

---

## 🚀 Flujo de Trabajo con Git y Ansible

1. **Bootstrap del nodo:** Flasheo de DietPi con variables preconfiguradas para conexión desatendida.
2. **Aprovisionamiento:**
   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/site.yml --diff
   ```
3. **Control de cambios en documentación:**
Cada fase completada se documenta y sube de forma atómica:
   ```bash
   git add docs/01-setup/
   git commit -m "docs(setup): documentar bootstrap headless y sincronización RTC"
   git push origin main
   ```
---

## 🔗 Referencias Técnicas

* [DietPi Documentation](https://dietpi.com/docs/)
* [ext4 Documentation](https://www.kernel.org/doc/html/latest/filesystems/ext4/)
* [Ansible Core Documentation](https://docs.ansible.com/)
