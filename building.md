# Guía de Construcción y Compilación

Esta guía describe el entorno, herramientas y pasos necesarios para compilar y validar los componentes de **CosmicOS**.

---

## 1. Requisitos Previos

Para compilar el árbol completo del sistema en una distribución Linux moderna (x86_64 o ARM64):

| Herramienta | Versión Mínima Requerida | Rol |
|---|---|---|
| **Rust / Cargo** | 1.85+ (la edición 2024 exige 1.85+) | Compilación de `core/`, `daemons/`, `ui/` y `pkg/sps`. |
| **Go** | 1.22+ | Compilación del orquestador `core/corchd`. |
| **GCC / Clang** | GCC 13+ o Clang 17+ | Compilación del kernel Linux, cabeceras C y SDK. |
| **CMake** | 3.20+ | Construcción del Shell móvil `ui/creationde` (Qt6). |
| **Repo** | 2.x+ (Google repo tool) | Sincronización multi-repositorio. |

---

## 2. Sincronización con `repo`

Una vez publicado el manifiesto en la organización `OpenCosmicProject`, la obtención del código se realiza mediante:

```bash
# Inicializar repositorio en el canal estable (nebula)
repo init -u https://github.com/OpenCosmicProject/cosmic-manifest.git -b nebula

# Sincronizar todos los repositorios
repo sync -j$(nproc)

# Sincronizar únicamente los repositorios del Hito 1 (milestone1)
repo sync -j$(nproc) -g milestone1
```

---

## 3. Compilación Unificada con `cosmic-build` (Recomendado)

CosmicOS cuenta con un frontend unificado en Bash y motor de grafos Ninja (`tools/cosmic-build`), libre por completo de dependencias de Python:

```bash
# Compilar los componentes del Hito 1 (Rust + Go)
./cosmic-build milestone1

# Compilar para otra arquitectura soportada (ej. riscv64)
./cosmic-build --target riscv64 milestone1

# Compilar el kernel Linux LTS endurecido
./cosmic-build kernel

# Verificar artefactos en el directorio out/ con el verificador C++
./cosmic-build check

# Limpieza completa
./cosmic-build clean
```

---

## 4. Compilación Manual (Bajo Nivel)

Si se desea invocar los toolchains de bajo nivel de forma independiente:

### 4.1 Componentes en Rust (Cargo)
El proyecto utiliza un **Cargo Workspace** centralizado con `default-members` optimizado para el **Hito 1** ([Sección 7.7 y 13.1 de la Spec](./CosmicOS-spec.md#131-estado-de-implementacion)):

```bash
# Comprobar sintaxis y tipos de los crates del Hito 1
cargo check

# Compilar binarios de lanzamiento para el Hito 1
cargo build --release

# Compilar todos los crates del workspace (incluidos UI y herramientas)
cargo build --workspace --release
```

### 4.2 Orquestador en Go (`corchd`)
```bash
cd core/corchd
go build -v -ldflags="-s -w" -o ../../target/bin/corchd .
```

---

## 5. Validación y Pruebas en QEMU

Para validar el sandbox de `clauncher` y el PID 1 `cinitd` sin necesidad de hardware físico:

```bash
# Ejecución de pruebas unitarias existentes
cargo test --workspace

# Prueba de escape del sandbox (Sección 7.3 de la Spec)
# [Pendiente de implementación en el Paso 1 de la hoja de ruta]
# cargo test -p clauncher --test sandbox_escape
```
