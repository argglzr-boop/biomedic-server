# Biomedic Edge Server — Raspberry Pi Zero 2 W & DietPi

Infraestructura como Código (IaC) para el aprovisionamiento, securización y operación de un nodo perimetral biomédico de bajo consumo basado en **DietPi**, orquestado mediante **Ansible** y respaldado localmente con **Btrfs** sobre **MicroSD High Endurance**.

> **Estado del Proyecto:** 🟡 **En desarrollo — Fase 00 (Diseño de Arquitectura y Especificación Técnica) completada.**  
> El código IaC (Ansible) y los scripts de aprovisionamiento se encuentran en proceso de implementación modular conforme al roadmap.

---

## 📋 Resumen del Proyecto

| Parámetro | Definición |
| :--- | :--- |
| **Plataforma Hardware** | Raspberry Pi Zero 2 W (SoC ARM Cortex-A53 quad-core, 512 MB LPDDR2) |
| **Almacenamiento Primario** | MicroSD High Endurance (Partición dual: ext4 OS + Btrfs datos) |
| **Sistema Operativo** | DietPi (Debian minimal con optimización de RAM-log y kernel afinado) |
| **Motor de Automatización**| Ansible (roles desacoplados, modo idempotente y trazable) |
| **Persistencia Transaccional**| SQLite (modo WAL optimizado para flash y compresión zstd) |
| **Conectividad Segura** | Túnel WireGuard (ChaCha20-Poly1305) y firewall estricto (UFW) |

---

## 🗺️ Roadmap y Checklist de Implementación

### 📐 00 — Arquitectura y Visión Transversal
- [x] **`00-architecture/overview.md`**: Topología funcional del nodo biomédico y arquitectura de 4 capas.
- [x] **`00-architecture/compliance-security.md`**: Cifrado en reposo (LUKS), segregación de datos PHI y diseño de auditoría.
- [ ] **`00-architecture/resilience-strategy.md`**: Objetivos RPO/RTO, modelo Btrfs 3-2-1 y retención de snapshots.

---

### 🟢 01 — Setup Base y Bootstrap
- [ ] **`01-setup/hardware.md`**: BOM, módulo RTC (DS3231), consumo eléctrico y disipación térmica.
- [ ] **`01-setup/installation.md`**: Flasheo de MicroSD High Endurance y configuración `dietpi.txt` headless.
- [ ] **`01-setup/network-time.md`**: IP estática/DHCP reservado, mitigación de drift temporal (NTP/Chrony + RTC).
- [ ] **`01-setup/bootstrap-ansible.md`**: Inyección de llave SSH temporal, control node handoff e inventario inicial.

---

### 🟡 02 — Endurecimiento Perimetral (Hardening)
- [ ] **`02-security/ssh-hardening.md`**: Llaves Ed25519, cambio de puerto, deshabilitación de root y contraseñas.
- [ ] **`02-security/firewall-ufw.md`**: Políticas de descarte por defecto (`DROP`) y segmentación LAN médica.
- [ ] **`02-security/fail2ban.md`**: Jaulas personalizadas para SSH y mitigación de fuerza bruta.
- [ ] **`02-security/auditd.md`**: Reglas de auditoría del kernel para accesos a datos clínicos y binarios sensibles.

---

### 💾 03 — Almacenamiento Seguro y Persistente
- [ ] **`03-storage/luks-encryption.md`**: Cifrado de volumen LUKS con desbloqueo controlado para datos médicos.
- [ ] **`03-storage/btrfs-subvolumes.md`**: Esquema de subvolúmenes (`@data`, `@db`, `@snapshots`), flags de fstab (`noatime`, `compress=zstd:1`).

---

### ⚙️ 04 — Servicios del Nodo Biomédico
- [ ] **`04-services/SQLite.md`**: Ingesta de variables fisiológicas, persistencia en `@data` y mitigación CoW.
- [ ] **`04-services/wireguard.md`**: Enlace VPN cifrado punto a punto para telemetría remota segura.
- [ ] **`04-services/monitoring.md`**: Despliegue de `node-exporter` para telemetría de hardware (temperatura, I/O, swap).
- [ ] **`04-services/grafana.md`**: Dashboards de salud del nodo y RBAC para personal médico/técnico.

---

### 🛡️ 05 — Resiliencia y Continuidad Operativa
- [ ] **`05-resilience/btrfs-snapshots.md`**: Creación de snapshots atómicos de solo lectura, purga y rollback.
- [ ] **`05-resilience/db-backups.md`**: Rutinas de volcado consistente de bases de datos antes de snapshots.
- [ ] **`05-resilience/etckeeper.md`**: Control de versiones de `/etc` automatizado con Git.
- [ ] **`05-resilience/remote-sync.md`**: Réplica periódica incremental cifrada hacia NAS o VPS central.

---

### 🤖 06 — Operaciones y Automatización
- [ ] **`06-operations/ansible-playbooks.md`**: Catálogo de roles, tags de ejecución (`--tags security,storage`) y dry-run.
- [ ] **`06-operations/maintenance-cron.md`**: Tareas programadas de scrubbing de Btrfs, trim y salud del storage.
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
* [Btrfs Administration Guide](https://btrfs.readthedocs.io/)
* [Ansible Core Documentation](https://docs.ansible.com/)
