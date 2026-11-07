# Guía Práctica del Kernel Linux

> **Fuente de verdad:** La especificación maestra y requisitos técnicos del kernel residen en [CosmicOS-spec.md#3-kernel](./CosmicOS-spec.md#3-kernel).

---

## 1. Purga Reproducible de Arquitecturas (`purge-arch.sh`)

CosmicOS soporta exclusivamente **ARM64** y **RISC-V**. Para purgar el árbol upstream de arquitecturas obsoletas y ajustar las referencias Kconfig rotas:

```bash
# Ejecutar desde la raíz del proyecto
./kernel/scripts/purge-arch.sh
```

El script conserva únicamente:
- `kernel/linux/arch/arm64`
- `kernel/linux/arch/riscv`
- `kernel/linux/arch/Kconfig`
- `kernel/linux/arch/.gitignore`

---

## 2. Generación de Configuración y Compilación

### Para ARM64:
```bash
# Opción A: Configuración estándar mínima
make -C kernel/linux ARCH=arm64 defconfig

# Opción B: Aplicar defconfig endurecido de CosmicOS
cp kernel/configs/cosmicos_defconfig_arm64 kernel/linux/.config
make -C kernel/linux ARCH=arm64 olddefconfig

# Compilar kernel comprimido
make -C kernel/linux ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image.gz -j$(nproc)
```

> **Nota:** La imagen UKI (*Unified Kernel Image*) se ensambla posteriormente empaquetando el binario del kernel (`Image.gz`), el initrd firmado con `cosmic-splash` y la línea de comandos fija.

### Para RISC-V (64-bit):
```bash
# Opción A: Configuración estándar mínima
make -C kernel/linux ARCH=riscv defconfig

# Opción B: Aplicar defconfig endurecido de CosmicOS
cp kernel/configs/cosmicos_defconfig_riscv64 kernel/linux/.config
make -C kernel/linux ARCH=riscv olddefconfig

# Compilar kernel
make -C kernel/linux ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- Image -j$(nproc)
```

---

## 3. Poda de Controladores (*Drivers*)

Siguiendo el principio de mínimo privilegio y reducción de superficie:
- **No se borran carpetas en `drivers/`:** La poda se realiza exclusivamente mediante Kconfig.
- **Medición de TCB:** Se evalúan los archivos de objeto `.o` compilados.
- **Sellado de módulos en tiempo de ejecución:** Tras el arranque, `cinitd` fija `kernel.modules_disabled = 1`.
