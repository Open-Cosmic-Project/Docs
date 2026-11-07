# Organización de Repositorios y Fuentes

CosmicOS utiliza una arquitectura **multi-repositorio** coordinada mediante la herramienta `repo` de Google/AOSP, bajo la organización oficial:

> **Organización Oficial:** `OpenCosmicProject` (OCP)  
> **Repositorio de Manifiesto:** `cosmic-manifest`  
> **Archivo Central:** `default.xml`

---

## 1. Convención de Nombres de Componentes

La nomenclatura oficial (Sección 7.5 de la especificación) sigue reglas homogéneas:

1. **Demonios y servicios del sistema:** Llevan el prefijo `c` y el sufijo `d` (`c<nombre>d`).  
   *Ejemplos:* `cinitd`, `corchd`, `clogd`, `cnotifd`, `cpushd`, `cbrokerd`, `cpowerd`, `csettingsd`, `cupdated`, `cfronterad`, `cdevd`.
2. **Herramientas, paquetes y repositorios centrales:** Llevan el prefijo `cosmic-`.  
   *Ejemplos:* `cosmic-manifest`, `cosmic-splash`, `cosmic-flash`, `cosmic-repo`, `cosmic-sdk`, `cosmic-kernel`.
3. **Piezas especializadas del núcleo, gestor e interfaz:**
   - Gestor de paquetes: `sps`
   - Supervisor: `kernel.root`
   - Lanzador: `clauncher`
   - Compositor: `PhanteonWM`
   - Shell: `CreationDE`
   - Teclado: `cboard`

*(Nota: El repositorio auxiliar `cosmic-build` para contener el workspace raíz es actualmente una idea técnica **[Propuesta]** para la v0.2.8; no forma parte del catálogo activo).*

---

## 2. Catálogo de Repositorios Oficiales

| Repositorio OCP | Ruta Local (`path`) | Grupos `repo` | Descripción |
|---|---|---|---|
| `cosmic-manifest` | `cosmic-manifest/` | `default` | Manifiesto XML que fija el bloqueo de versiones del sistema. |
| `cosmic-core` | `libs/cosmic-core/` | `mobile,desktop-dev,milestone1` | Librería central compartida (IPC, manifiesto, políticas). |
| `cinitd` | `core/cinitd/` | `mobile,desktop-dev,milestone1` | Init PID 1 ligero en Rust. |
| `kernel.root` | `core/kernel.root/` | `launcher,mobile,desktop-dev,milestone1` | Supervisor de procesos y ventana segura de control. |
| `clauncher` | `core/clauncher/` | `launcher,mobile,desktop-dev,milestone1` | Launcher con sandbox Landlock/seccomp. |
| `corchd` | `core/corchd/` | `launcher,mobile,desktop-dev` | Orquestador en Go sin privilegios. |
| `cdevd` | `core/cdevd/` | `mobile,desktop-dev` | Gestor de dispositivos (`/dev` y reglas SELinux). |
| `clogd` | `daemons/clogd/` | `mobile,desktop-dev` | Demonio centralizado de logging seguro. |
| `cnotifd` | `daemons/cnotifd/` | `mobile,desktop-dev` | Demonio de notificaciones efímeras. |
| `cpushd` | `daemons/cpushd/` | `mobile` | Cliente nativo de Web Push (RFC 8030/8291). |
| `cbrokerd` | `daemons/cbrokerd/` | `mobile,desktop-dev` | Broker de permisos y mediador de portales. |
| `cpowerd` | `daemons/cpowerd/` | `mobile,desktop-dev` | Demonio de energía y estados de suspensión. |
| `csettingsd` | `daemons/csettingsd/` | `mobile,desktop-dev` | Servicio de configuración y ajustes del sistema. |
| `cupdated` | `daemons/cupdated/` | `mobile,desktop-dev` | Demonio de actualización OTA A/B. |
| `cfronterad` | `daemons/cfronterad/` | `mobile,desktop-dev` | Frontera FireWall (política declarativa nftables/eBPF). |
| `PhanteonWM` | `ui/phanteonwm/` | `ui,mobile,desktop-dev` | Compositor Wayland en Rust (Smithay). |
| `CreationDE` | `ui/creationde/` | `ui,mobile,desktop-dev` | Shell móvil en Qt6/QML. |
| `cboard` | `ui/cboard/` | `ui,mobile` | Teclado táctil en pantalla sin red. |
| `cosmic-splash` | `ui/cosmic-splash/` | `ui,mobile,desktop-dev` | Pantalla de arranque KMS en initrd. |
| `cosmic-kernel` | `kernel/linux/` | `kernel,mobile,desktop-dev` | Fork de Linux LTS con `arch/` purgado (`clone-depth="1"`). |
| `cosmic-kernel-configs` | `kernel/configs/` | `kernel,mobile,desktop-dev` | Archivos defconfig y scripts de purga del kernel. |
| `sps` | `pkg/sps/` | `mobile,desktop-dev,milestone1` | Gestor de paquetes Secure-Package-System. |
| `cosmic-build` | `tools/cosmic-build/` | `desktop-dev,milestone1` | Frontend unificado de compilación (Bash + Ninja, sin Python). |
| `tools/cosmic-flash` | `tools/cosmic-flash/` | `desktop-dev` | Herramienta de flasheo GPT / UKI / OTA. |
| `tools/cosmic-repo` | `tools/cosmic-repo/` | `desktop-dev,milestone1` | Herramientas de infraestructura TUF y firmas cosign. |
| `cosmic-chromium-patches` | `patches/chromium/` | `desktop-dev,mobile` | Parches y configuración de ungoogled-chromium / WebView. |
| `cosmic-qt-patches` | `patches/qt/` | `ui,desktop-dev,mobile` | Parches de poda y empaquetado para Qt6 móvil. |
| `cosmic-device-firmware` | `device/firmware/` | `mobile` | Paquetes de firmware aislado y árboles de dispositivos. |
| `cosmic-sdk` | `sdk/cosmic-sdk/` | `desktop-dev` | Kit de desarrollo oficial y cabecera C estable. |
| *(Documentación)* | `Docs/` | *(Raíz/OCP)* | Especificación maestra y guías técnicas prácticas. |

---

## 3. Grupos de Descarga en `repo`

Para optimizar ancho de banda y espacio de almacenamiento, el manifiesto clasifica los proyectos en grupos:
- `milestone1`: Crates esenciales para el primer hito de la hoja de ruta (`cinitd`, `kernel.root`, `clauncher`, `sps`, `cosmic-core`, `cosmic-repo`).
- `launcher`: Conjunto de lanzamiento de sandboxes (`clauncher`, `corchd`, `kernel.root`).
- `ui`: Pila gráfica y de interacción (`PhanteonWM`, `CreationDE`, `cboard`, `cosmic-splash`).
- `kernel`: Código fuente del kernel Linux modificado y configuraciones.
- `mobile`: Pila completa para compilación dirigida a teléfonos inteligentes.
- `desktop-dev`: Herramientas de desarrollo, emulación y testing en estación de trabajo.
