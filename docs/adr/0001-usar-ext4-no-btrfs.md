# ADR-0001: Usar ext4 en lugar de Btrfs para la partición de datos

- **Estado:** Aceptado
- **Fecha:** 2026-10-27
- **Decisores:** Arquitecto y operador del nodo

## Contexto

El nodo perimetral corre sobre una Raspberry Pi Zero 2 W con **512 MB de RAM** y almacenamiento en **MicroSD High Endurance**. La arquitectura inicial contemplaba Btrfs para la partición de datos, con subvolúmenes (`@data`, `@db`, `@snapshots`) y compresión `zstd`.

Btrfs es un sistema de archivos CoW (Copy-on-Write) con snapshots nativos, pero su modelo de escritura genera **amplificación de escritura** (cada bloque modificado se reescribe en una ubicación nueva) y un consumo de RAM/CPU considerable para su gestión de metadatos y checksums.

## Decisión

Se adopta **ext4** como sistema de archivos para la partición de datos, montada con la opción `noatime` para reducir escrituras innecesarias de metadatos.

Se descarta Btrfs por:

1. **Amplificación de escritura (CoW):** acelera el desgaste de la MicroSD, que es el componente más frágil del nodo.
2. **Huella de RAM/CPU:** incompatible con los 512 MB disponibles y con el perfil de bajo consumo del SoC Cortex-A53.
3. **Complejidad innecesaria:** los snapshots locales que justificaban Btrfs se sustituyen por IaC (Git) para configuraciones y por dumps lógicos `.sql` comprimidos para la base de datos.

Se descarta también **XFS** por su rigidez y sobrecarga innecesaria en un nodo edge con un único disco.

## Consecuencias

- **Positivas:** menor desgaste de la MicroSD, menor consumo de RAM/CPU, sistema de archivos maduro y ampliamente soportado en DietPi/Debian.
- **Negativas:** se pierden los snapshots atómicos nativos de Btrfs; la reversión de configuraciones depende del repositorio Git y la recuperación de la BD depende de los dumps lógicos.
- **Mitigación:** IaC para configuraciones y servicios, dumps transaccionales previos a mantenimientos, y Golden Image para el aprovisionamiento base del nodo.

## Alternativas consideradas

| Alternativa | Motivo de descarte |
| :--- | :--- |
| Btrfs | Amplificación de escritura (CoW) y alta huella de RAM/CPU |
| XFS | Rigidez y sobrecarga innecesaria en edge |
| F2FS | Optimizado para flash, pero menor madurez y soporte en el stack DietPi/Ansible del proyecto |
