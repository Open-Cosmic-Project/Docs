# CosmicOS — Especificación de diseño v0.2.9

> **Estado:** borrador de trabajo · **Fecha:** 7 de noviembre de 2026 · **Autor:** Ange^_^
> **Nombre:** CosmicOS, nombre definitivo. El autor conoce la coincidencia parcial con COSMIC, el entorno de escritorio de System76, y lo mantiene.

## Cómo leer este documento

Cada decisión lleva una etiqueta de estado:

| Etiqueta | Significado |
|---|---|
| **[Acordado]** | Decisión tomada por el autor del proyecto (incluye las recomendaciones técnicas que el autor aceptó el 18 y el 20 de octubre de 2026) |
| **[Propuesto]** | Sugerencia técnica nueva, pendiente de que el autor la confirme. Ahora mismo no hay ninguna (el autor aceptó todas el 21 de octubre de 2026); el estado se conserva para futuras sugerencias |
| **[Aplazado]** | Decisión de no decidir todavía, con la condición que la reactiva (dinero, hardware, código). Sigue viva en la tabla 14.1 |
| **[Abierto]** | Decisión pendiente. El detalle de cada una vive en el registro de la sección 14 (tabla 14.1); en el texto solo queda un puntero con su identificador `D-nn` |

La primera vez que aparece un término técnico, está enlazado a su definición en el [glosario](#glosario) (sección 22).

---

## 1. Visión y alcance

CosmicOS es un sistema operativo móvil nativo, pensado como alternativa a Android e iOS. Renuncia de forma deliberada a las [APKs](#g-apk), a [ART/JVM](#g-art) y a los servicios de Google ([GMS](#g-gms)), y a cambio ofrece:

- Un kernel Linux **[híbrido](#g-hibrido)**, estático y endurecido, solo para **ARM64 y RISC-V**.
- Aplicaciones **nativas y compiladas** (Rust preferido; también C y C++).
- Un gestor de paquetes propio, **[SPS](#g-sps) (Secure-Package-System)**, con paquetes [`.sps`](#g-sps-pkg) y manifiestos de permisos mínimos.
- Un **repositorio central público** y auditable como base de confianza.
- Su propio conjunto de servicios ("GMS propio"), pequeño, autoalojable y escrito en C/Rust.
- Una interfaz móvil propia: **[PhanteonWM](#g-phanteon)** ([compositor](#g-compositor)) y **[CreationDE](#g-creationde)** ([shell](#g-shell)).

**Público inicial:** desarrolladores y entusiastas de Linux. **Público general:** cuando el sistema sea estable.

**Arquitecturas** **[Propuesto]**: el kernel admite ARM64 y RISC-V (3), pero el objetivo de teléfono a corto plazo es ARM64. RISC-V queda para placas, tabletas y escritorio hasta que exista un teléfono con GPU y módem soportados upstream (13).

**Referencias:** OpenHarmony/HarmonyOS (arquitectura y kernel), Android ([GKI](#g-gki), [SELinux](#g-selinux), [sandbox](#g-sandbox) por UID), Flathub y F-Droid (modelo de repositorio), OpenBSD `pledge`/`unveil` (mínimo privilegio).

### 1.1 Lo que este documento no cubre

Para que quien se sume sepa qué falta, esta lista dice qué no está diseñado en detalle. Se actualiza en cada versión.

- **[Propuesto]** **Diseño visual:** de CreationDE y PhanteonWM (gestos concretos, animaciones, iconos, tipografía y temas).
- **[Propuesto]** **La tienda de apps:** su interfaz, el descubrimiento de apps y las páginas de cada app.
- **[Propuesto]** **El asistente de primer arranque**, incluida la pantalla para restaurar una copia (5.3).
- **[Propuesto]** **Las apps esenciales:** solo hay una lista (teléfono, mensajes, cámara, galería, mensajería, archivos, ajustes y navegador), sin diseño.
- **[Propuesto]** **Llamadas de emergencia:** no están diseñadas. Son un requisito regulatorio básico de un teléfono (en Colombia, la línea 123) y dependen del módem y del operador (11.3).
- **[Propuesto]** **Conectividad:** eSIM, varias SIM, compartir internet (*tethering*) y VPN no están diseñados; el cortafuegos solo está esbozado en 10.9.
- **[Propuesto]** **APIs concretas:** la API del SDK, la lista de permisos del broker y el protocolo del gateway push más allá de Web Push.
- **[Propuesto]** **Seguridad concreta:** la política SELinux real y el formato exacto del bundle OTA.
- **[Propuesto]** **Hardware y formatos:** sensores, cámara más allá del posprocesado, NFC, gestión térmica y de carga, pantallas plegables y tabletas. El contrato de interfaz con los demonios de hardware solo está esbozado en 16.6.
- **[Propuesto]** **Funciones de uso diario:** widgets, modo «no molestar», impresión y la interfaz del gestor de archivos.

## 2. Principios de diseño

1. **Mínimo privilegio:** cada componente recibe solo lo que declara y lo que se le concede.
2. **[TCB](#g-tcb) pequeño:** las piezas privilegiadas son pocas, pequeñas y en lenguajes seguros.
3. **Inmutabilidad:** sistema base de solo lectura, política sellada, binarios de sistema protegidos.
4. **Todo es un archivo (POSIX):** estructura de directorios clara y predecible.
5. **Nativo y compilado:** sin máquinas virtuales ni runtimes pesados.
6. **Abierto y auditable:** manifiestos, repositorio y revisiones públicas.
7. **Autoalojable:** ningún servicio central obligatorio.
8. **El dispositivo es del usuario:** solo se soportan dispositivos cuyo bootloader se pueda desbloquear y volver a bloquear con una clave del usuario (ver 11.4).
9. **Reutilizar antes de escribir** **[Propuesto]:** cada componente propio tiene que justificar por qué no sirve uno existente. El proyecto debe poder mantenerse durante años con un equipo pequeño; el agotamiento del equipo es el riesgo más probable, y mejor calidad que Android no exige que todo sea escrito desde cero (propuesta surgida en la comunidad, 6 de noviembre de 2026).

---

## 3. Kernel

| Estado | Decisión |
|---|---|
| **[Acordado]** | Kernel Linux **modificado (fork)** con soporte únicamente para ARM64 y RISC-V: `arch/` purgado, conservando `arm64`, `riscv`, `arch/Kconfig` y `arch/.gitignore`. Los parámetros de arranque van fijos en la compilación (decisión del autor, 6 de noviembre de 2026). |
| **[Acordado]** | El fork vive en un repositorio propio de la organización y el manifiesto lo baja con `clone-depth="1"` (8.5). La purga la hace el script reproducible `kernel/scripts/purge-arch.sh`, creado y probado: con él, el árbol compila para arm64 hasta `Image.gz` (7 de noviembre de 2026). |
| **[Propuesto]** | Ramas por versión upstream (`cosmic/6.x`): etiqueta upstream tal cual, un commit aparte «purga de `arch/`» y los parches encima. Así cada actualización es mecánica y se puede comprobar que el fork es upstream más la purga y nada más. |
| **[Propuesto]** | **Poda de drivers solo por Kconfig:** no se borran carpetas de `drivers/` (están entrelazados y es lo que más cambia upstream: cada actualización daría conflictos). Se parte de una configuración mínima, se guarda con `make savedefconfig`, se mide el TCB real con los objetos `.o` compilados y en cada versión se revisa `olddefconfig` y `listnewconfig`. Solo se podrían borrar carpetas hoja sin dependientes, con un script reproducible y CI en arm64 y riscv64. |
| **[Abierto]** | [D-43](#d-43): dónde viven los parámetros fijos, si hay variante libre en los canales de pruebas y qué compilador se usa (GCC o LLVM). |
| **[Acordado]** | Kernel **[híbrido](#g-hibrido)** en el sentido propio de CosmicOS: Linux monolítico con SELinux integrado en el binario y con algunos drivers **básicos y no críticos delegados al espacio de usuario**. Ver la nota sobre el término más abajo. |
| **[Acordado]** | Tras el arranque, todos los procesos **divergen del proceso raíz ([`kernel.root`](#g-kernel-root))**: cada uno nace en su propio sandbox y/o dominio SELinux. Si un proceso falla o es atacado, el sistema no cae y el proceso se reinicia dentro de su sandbox o se aísla temporalmente. Definición completa en 3.1. |
| **[Acordado]** | Procesos aislados con [namespaces](#g-namespaces), [cgroups](#g-cgroups) y [eBPF](#g-ebpf); sin máquinas virtuales pesadas. |
| **[Acordado]** | **Módulos solo durante el arranque:** se cargan los drivers como módulos y, al terminar el arranque, `init` cierra la puerta con `kernel.modules_disabled` (interruptor de un solo sentido: no se reactiva hasta reiniciar). Sustituye a la decisión anterior de `CONFIG_MODULES=n`. |
| **[Acordado]** | Con `CONFIG_MODULE_SIG_FORCE` solo se cargan módulos firmados. Lo necesario para llegar al sistema base (almacenamiento, dm-verity, EROFS) va compilado en el kernel o en el initrd de la UKI; el resto de módulos vive en el sistema base verificado (`/usr/lib/modules`, protegido por dm-verity) y `init` los carga al arrancar. Así la UKI no lleva los drivers de todos los dispositivos (las particiones de arranque de los móviles pueden ser pequeñas) y un driver se puede actualizar sin recompilar todo el kernel. |
| **[Acordado]** | `kernel.modules_disabled=1` bloquea tanto cargar como **descargar** módulos (lo dice la documentación del kernel), así que no hace falta `CONFIG_MODULE_UNLOAD=n` para este fin. |
| **[Acordado]** | Consecuencia de cerrar los módulos tras el arranque: los periféricos que se conecten después (USB, Bluetooth, NFC) no podrán cargar su driver. Hay que cargar durante el arranque todos los drivers que se espera usar (o compilarlos dentro) y asumir el coste en memoria y superficie; lo que no se cargó no funciona hasta reiniciar. |
| **[Acordado]** | Política SELinux **sellada solo en las compilaciones finales** (canal [Nebula](#g-nebula)): sin `CONFIG_SECURITY_SELINUX_DEVELOP`, sin modo permisivo, sin `setenforce`; `load_policy` permitido una sola vez a [init](#g-init). Las compilaciones de desarrollo (resto de canales) pueden dejarla sin sellar para depurar la política sin reflashear. |
| **[Acordado]** | Embeber la política en el initrd dentro de la UKI firmada, en lugar de parchear el kernel: es más limpio y la verificación de la UKI cubre también la política. |
| **[Acordado]** | Hardening: [kCFI](#g-hardening), shadow call stack (hasta donde sé, kCFI y la pila de llamadas sombra dependen de Clang: ver [D-43](#d-43)), PAC/BTI/MTE (ARM64), `init_on_alloc`/`init_on_free`, hardening de slab, lockdown en modo confidentiality, eBPF sin acceso sin privilegios (el kernel admite programas BPF firmados desde Linux 6.18; ver la regla en [D-22](#d-22)). |
| **[Acordado]** | **BPF:** ninguna app carga programas BPF; solo el sistema carga programas firmados con la clave de CosmicOS, y el modo Desarrollador avanzado permite cargarlos sin firmar ([D-22](#d-22), 22 de octubre de 2026). |
| **[Acordado]** | Drivers nuevos en Rust; módem aislado con [IOMMU](#g-iommu). |

**Notas técnicas**

- SELinux nunca fue un `.ko`: los [LSM](#g-lsm) se compilan dentro de `vmlinux`. Lo externo es la **política**, que userspace carga en `/sys/fs/selinux/load`; esa es la superficie a cerrar.
- `kernel.root` se define en la sección 3.1: es un concepto propio de CosmicOS que no existe en el Linux tradicional.
- **Sobre el término «híbrido».** En la literatura de sistemas operativos (NT, XNU) un núcleo híbrido tiene parte de sus servicios fuera del espacio del kernel. Un Linux monolítico con procesos bien aislados no encaja del todo; el autor mantiene el término porque algunos drivers básicos y no críticos irán a espacio de usuario. Hasta que existan, la descripción exacta es «Linux monolítico con aislamiento fuerte de procesos». Candidatos razonables a salir del kernel: dispositivos USB sencillos (libusb), I2C/SPI (`i2c-dev`, `spidev`), entrada virtual (`uinput`) y sensores simples. Coste: latencia y un demonio por driver; con DMA hace falta además IOMMU/VFIO.
- Purgar `arch/` no acelera la compilación (Kconfig solo construye la arquitectura elegida) ni cambia el binario: su razón es tener menos código que mantener y auditar. Ya está hecho (3, tabla), con dos cuidados aprendidos al compilar: el script debe tratar los residuos de la purga (por ejemplo, `scripts/checksyscalls.sh` busca la tabla de llamadas de x86 y avisa de que falta) y la configuración se calcula con las mismas variables (`ARCH`, `CROSS_COMPILE`) que la compilación; si no, el compilador detecta símbolos nuevos a mitad del proceso y los resuelve con valores por defecto sin que nadie los elija.
- **Límite asumido:** un bug de driver dentro del kernel no se contiene con namespaces, porque todos comparten el espacio de direcciones del kernel. Esa clase de fallos se mitiga con hardening y Rust, no con contenedores. Reiniciar componentes del kernel requeriría un microkernel o drivers en espacio de usuario (coste de rendimiento).

### 3.1 `kernel.root`

`kernel.root` es un concepto propio de CosmicOS: no existe en el Linux tradicional.

**Definición (del autor)**

- **[Acordado]** Es un proceso en segundo plano, uno de los primeros que arrancan, emparentado con el PID 2 original del kernel (`kthreadd`, que lanza los hilos del kernel). Es la **ventana** permanente por la que las apps del sistema se comunican con el kernel con más facilidad (no directamente) y, además, un **supervisor** de los procesos, que **reinicia los que se quedan colgados** dentro de su sandbox. Es la «cara parcial» del kernel en espacio de usuario.
- **[Acordado]** Es un **mero supervisor de solo lectura (RO, no RW)**: su binario y su configuración viven en el sistema base firmado, no tiene permiso de escritura sobre datos de usuario ni de apps, y su estado vive en memoria (tmpfs).
- **[Acordado]** No se puede matar, ni siquiera siendo root, salvo en el **[modo Desarrollador avanzado](#g-modo-dev)** o al apagar o reiniciar el dispositivo.
- **[Acordado]** Si un proceso se cae, `kernel.root` decide si lo reinicia en su sandbox o lo aísla según la configuración del supervisor; SELinux limita quién puede pedírselo (ver «Qué decide SELinux y qué no», más abajo).
- **[Acordado]** Todos los demás procesos y protocolos **divergen** de él: cada uno se separa en su propio sandbox y/o dominio SELinux. Si uno es atacado o entra en error, no compromete al sistema: se reinicia dentro de su sandbox o se aísla temporalmente.
- **[Acordado]** SELinux va integrado dentro de `vmlinux` (y sellado en las compilaciones Nebula) para que el sistema arranque con la política ya activa y `kernel.root` nazca bajo ella. (En Linux, SELinux no es un proceso sino un módulo de seguridad del kernel, así que la idea de "SELinux dentro de `kernel.root`" se interpreta de esta forma.)

**Cómo se materializa en Linux**

| Estado | Idea | Mecanismo propuesto |
|---|---|---|
| **[Acordado]** | Relación con `kthreadd` | En el arranque, el kernel crea `init` (PID 1) y `kthreadd` (PID 2) como hermanos, hijos del PID 0; los hijos de `kthreadd` son hilos del kernel, sin espacio de usuario, así que `kernel.root` no puede ser literalmente uno de ellos sin un parche. Se adopta la opción (a): `kernel.root` es el **primer proceso que `init` crea tras arrancar (el segundo proceso de usuario del sistema)**, identificado por su dominio SELinux (`kernel_root_t`) y por un `pidfd`. La alternativa (lanzarlo como *usermode helper*, con `kthreadd` como padre) exigía un parche y que `init` no pudiera recogerlo con `wait`; solo cambiaba lo que muestra `ps`. |
| **[Acordado]** | No se puede matar | La política SELinux no concede a ningún dominio, tampoco a root, los permisos de señal ni de `ptrace` sobre `kernel_root_t`. Solo desaparece al apagar o reiniciar. |
| **[Acordado]** | Salvo en modo Desarrollador | En las compilaciones selladas (Nebula) la política no cambia en caliente, así que el modo Desarrollador **no se activa en caliente**: es una variante de la política que se elige al arrancar y queda registrada en el estado de arranque (sección 12). |
| **[Acordado]** | Los demás divergen | Solo `kernel_root_t` puede hacer transiciones a otros dominios, y lo hace mediante el launcher: cada proceso nace con namespaces propios (montaje, PID, red, IPC, usuario y cgroup), contexto SELinux propio, [seccomp](#g-seccomp), [Landlock](#g-landlock) y cgroup, sin compartir los de su padre. |
| **[Acordado]** | Reinicio o aislamiento | `kernel.root` supervisa con *[backoff](#g-backoff)* exponencial y un contador de fallos. Tras N fallos seguidos, el servicio se aísla (no se reinicia más), se anota en el [registro de auditoría](#g-auditoria) y se avisa al usuario. |

**Cuidados**

- **[Acordado]** **Es la pieza más crítica del TCB.** Lo que es inmortal y lo gestiona todo es también el mayor punto único de fallo y de ataque. Debe ser un **supervisor pequeño** (arrancar, aislar, reiniciar, aplicar política) y no una navaja suiza: notificaciones, logging, teclado, etc. siguen siendo demonios separados. En Rust, sin parsear contenido de apps, con [fuzzing](#g-fuzzing) y revisión estricta.
- **[Acordado]** **Qué pasa si cae por un bug propio.** Al ser inmortal para el resto, solo `init` puede vigilarlo: si `kernel.root` termina de forma anormal, `init` fuerza un reinicio controlado en lugar de dejar el sistema sin supervisor.
- **[Acordado]** **Cómo detecta que algo está colgado.** Los servicios del sistema envían un *heartbeat* a `kernel.root` (al estilo `sd_notify`). Las apps con interfaz no: bloquearse un rato puede ser normal, así que el compositor las vigila con el ping/pong de `xdg_wm_base` (Wayland) y, si no responden, le pide a `kernel.root` reiniciarlas. Así `kernel.root` no parsea nada de las apps. Para señalar a todos los dominios necesita ese permiso en la política, pero ninguno de escritura.
- **[Acordado]** **El OOM killer.** Es lo único que podría matar a `kernel.root` sin que nadie lo decida. `init` fija `oom_score_adj = -1000` en `kernel.root` antes de lanzarlo (lo exime del OOM killer), la política SELinux impide que otro dominio lo cambie, se reserva memoria para su cgroup (`memory.min`) y cualquier demonio de memoria en espacio de usuario (tipo `earlyoom`) lo excluye. Si falta memoria, mueren antes las apps y los servicios.
- **[Acordado]** **Qué decide SELinux y qué no.** SELinux solo autoriza o niega acciones (quién puede señalar, reiniciar o crear un dominio); no tiene noción de «reiniciar un servicio». La política de reinicio (cuándo, cuántas veces, con qué *backoff*) vive en la configuración declarativa del supervisor, y SELinux limita quién puede pedirla o saltársela. Así se cumple lo acordado sin pedirle a SELinux algo que no hace.
- **[Acordado]** **La «ventana» necesita una API cerrada.** El kernel ya es accesible por syscalls, sysfs y netlink; `kernel.root` solo aporta valor en operaciones privilegiadas que ningún componente debería hacer por su cuenta (cgroups, montajes, política, energía). Lista cerrada de operaciones, cada una con su propio permiso SELinux; si crece sin límite, deja de ser un supervisor mínimo.
- **[Acordado]** **Colgado no es lo mismo que caído.** Vigilar con `pidfd` detecta que el proceso termina, no que se quede bloqueado pero vivo. Hace falta un *heartbeat*: `kernel.root` avisa a `init` cada pocos segundos y `init` alimenta el *watchdog* de hardware (`/dev/watchdog`) solo mientras lo recibe; si deja de llegar, el dispositivo se reinicia.
- **[Acordado]** **Qué hereda un `kernel.root` nuevo.** Sus hijos ya divergieron y no mueren con él, así que el sustituto heredaría un sistema cuyo estado no conoce. Para la primera versión: reinicio controlado de todo el sistema (simple y predecible); más adelante, readopción reconstruyendo el estado desde el árbol de cgroups.
- **[Acordado]** **Punto único de fallo aceptado.** Concentrar arrancar, aislar, reiniciar y aplicar política en un proceso inmortal es lo contrario de un TCB mínimo; se asume a cambio de un modelo mental simple, y por eso debe mantenerse pequeño.
- **[Acordado]** **Reiniciar automáticamente no es gratis.** Un proceso que se cae una y otra vez puede ser un atacante probando un exploit (por ejemplo, adivinando direcciones frente a [ASLR](#g-aslr)). Por eso se reinicia con un `exec` completo (que re-aleatoriza), con *backoff* y con escalada: al llegar al umbral se aísla en lugar de reiniciarse indefinidamente. Además, un servicio reiniciado pierde su estado en memoria, así que lo importante se guarda de forma segura.
- **[Acordado]** **Límite que sigue vigente:** esto contiene fallos de procesos de usuario; no contiene bugs dentro del propio kernel (ver "Límite asumido" arriba).
- **[Acordado]** **Relación con el launcher:** el launcher en Rust es el mecanismo con el que `kernel.root` crea cada proceso aislado, no una pieza aparte con privilegios propios.

**Puntos por decidir**

- **[Acordado]** Relación con `kthreadd`: opción (a). `kernel.root` es el primer proceso que `init` crea tras arrancar (el segundo proceso de usuario del sistema), con identidad por dominio SELinux y `pidfd`.
- **[Acordado]** **Operaciones de la ventana** (lista cerrada inicial; cada una con su propio permiso SELinux, y añadir una exige una propuesta pública, 11.6): (1) lanzar un proceso aislado a partir de un manifiesto validado; (2) reiniciar, aislar o congelar un servicio; (3) montar y desmontar la imagen de una app ya verificada (solo el instalador de SPS); (4) consultar el estado de un servicio (solo lectura); (5) apagar, reiniciar o suspender el dispositivo (solo el demonio de energía, 10.8).
- **[Acordado]** `init` ([CinitD](#d-25)) lanza `kernel.root`, recoge procesos huérfanos, lo vigila con heartbeat y alimenta el watchdog de hardware; nada más.
- **[Acordado]** Umbral y *backoff* por servicio: esperas de 1, 2, 4, 8 y 16 s entre reinicios y aislamiento al **quinto fallo seguido** dentro de una ventana de 10 minutos; el contador se reinicia tras 10 minutos estables. (Con un umbral de 3 fallos, una escalera de esperas hasta 60 s no llegaría a usarse: umbral y escalera deben ir juntos.) Los servicios esenciales de la interfaz (compositor, teclado, notificaciones) no se aíslan: tras el umbral se reinicia la sesión de interfaz y se avisa.
- **[Acordado]** Ningún canal para las apps hacia `kernel.root`: solo el launcher y los componentes del sistema, por un socket que verifica `SO_PEERSEC`.
- **[Acordado]** Tras una caída de `kernel.root`: reinicio controlado de todo el sistema en la primera versión; la readopción de procesos queda para más adelante.

---

## 4. Modelo de aislamiento de aplicaciones

- **[Acordado]** Las apps reciben solo los permisos **estrictamente necesarios**, declarados en [`Manifest.json`](#g-manifest).
- **[Acordado]** Directorios base del sistema (`/`, `/usr`, `/boot`, `/system`, etc.) estrictamente de solo lectura, garantizado por las políticas centrales de SELinux y el kernel.
- **[Acordado]** Mecanismo de aplicación en tiempo de ejecución, antes del `exec`:
  - Mount namespace con solo los directorios RO declarados.
  - **Landlock** + filtro **seccomp** base según la **clase de app** (ver 6.3), ajustado por el manifiesto.
  - `no_new_privs` y capabilities soltadas.
  - SELinux inmutable como red de seguridad debajo.
- **[Acordado]** Acceso a cámara, micrófono, galería y similares mediante un **[broker](#g-broker)** (modelo de portales), con consentimiento en runtime y revocable sin reinstalar.
- **[Acordado]** Verificación de integridad del sistema RO con [dm-verity](#g-dmverity)/fs-verity.
- **[Acordado]** Un filtro seccomp generado syscall a syscall desde cada manifiesto es frágil: libc llama a syscalls inesperadas según el locale, el DNS o `getrandom`, y las apps fallarían de formas difíciles de diagnosticar. Por eso se parte de un **filtro base fijo por clase de app** (es lo que Android hace con un filtro común) y el manifiesto solo ajusta.
- **[Acordado]** Para «sin red» se usa un **namespace de red vacío**, no solo Landlock: Landlock controla TCP por puerto (desde Linux 6.7) y las versiones recientes añaden UDP, pero no cubre los sockets raw. Aplica al teclado y a los demonios de accesibilidad.
- **[Acordado]** **Aislamiento dinámico con Landlock**, sin una política estática por app, y con SELinux debajo como red de seguridad; se descarta el modelo Android de categorías [MCS](#g-mcs) por app. ([D-01](#d-01), 22 de octubre de 2026)

> El manifiesto es una **solicitud**, nunca una concesión: lo que protege al usuario es el sandbox aplicado en el dispositivo.

---

## 5. Estructura de directorios

| Ruta | Contenido | ¿Se puede borrar? |
|---|---|---|
| `/usr/apps/<app-id>/` | Binarios y recursos de la app, montados RO | Solo al desinstalar |
| `~/.var/capps/<app>/{cache,data,config}` | **Única zona escribible por app** | `cache/` sí |
| `~/.var/daemons/<demonio>/` | Estado persistente de demonios (cifrado si es sensible) | No: es dato del usuario |
| `~/.cache/daemons/<demonio>/` | Temporales y archivos regenerables de demonios y protocolos | Sí, sin consecuencias |
| `~/.var/logs/<app>/{json,raw}` | Logs por app (ver sección 10.4) | Solo por rotación |
| `/run/user/<uid>/` ([tmpfs](#g-tmpfs)) | Memoria temporal en RAM, subcarpeta por app y demonio, con cuotas | Se pierde al reiniciar |

**Decisiones asociadas**

- **[Acordado]** Las apps pueden escribir únicamente en `~/.var/capps/<app>`.
- **[Acordado]** El resto del home (fotos, documentos) solo con permiso declarado y selector de archivos del sistema; no es el valor por defecto.
- **[Acordado]** Directorios ocultos compartidos (`.ssh`, `.gnupg`, `.config` general) fuera del alcance de las apps.
- **[Acordado]** Los archivos regenerables van en `.cache`; lo persistente, no. El caché del teclado se aprende y no es regenerable, por lo que vive en `~/.var/daemons/`, no en `.cache`.
- **[Acordado]** La carpeta `~/.cache/protocols` desaparece como zona aparte: sus restos pertenecen al demonio que usa cada protocolo, y los logs van a `~/.var/logs`.
- **[Acordado]** Nombre del tmpfs: `volatile` (evita la confusión de `flash` con el almacenamiento flash y con `/tmp`).
- **[Acordado]** Si se desea una vista `~/.cache/apps/<id>`, que sea un bind mount visible solo para esa app.

### 5.1 Sistema de ficheros

- **[Acordado]** En lugar de un FS único que lo haga todo, se **compone por roles** con sistemas existentes:

| Rol | FS / mecanismo | Motivo |
|---|---|---|
| Sistema base (`/usr`) | **[EROFS](#g-erofs) + dm-verity** | Solo lectura, comprimido, rápido; el kernel verifica cada bloque |
| Apps (`/usr/apps/<id>`) | Una imagen **EROFS firmada** por app y versión, montada RO | Instalar = descargar y montar; cuarentena = dejar de montar; [rollback](#g-rollback) de una app = conservar la imagen anterior |
| Datos del usuario (`/home`, `~/.var`) | **[F2FS](#g-f2fs)** (móvil) o **[Btrfs](#g-btrfs)** (escritorio) | F2FS: pensado para flash, [fscrypt](#g-fscrypt), compresión. Btrfs: snapshots, checksums de datos, compresión zstd |

- **[Acordado]** **ZFS descartado:** su licencia (CDDL) no es compatible con un kernel GPL estático y limpio.
- **[Acordado]** **Rollback del sistema con particiones [A/B](#g-ab)** y contador de arranque: la actualización se instala en el slot inactivo; si el arranque falla N veces, el [bootloader](#g-bootloader) vuelve al slot anterior. Se integra con el arranque verificado.
- **[Acordado]** El rollback afecta al **sistema, no a los datos del usuario**. Los formatos de datos se versionan, y antes de migrar uno se hace snapshot (Btrfs) o copia (F2FS), porque volver atrás tras una migración puede romper las apps.
- **[Acordado]** **Meta futura:** un FS propio (nombre tentativo **[AFS](#g-afs), Atomic File System**) como proyecto de investigación a largo plazo, prototipado con FUSE y desacoplado detrás del VFS para que sea reemplazable.
- **[Acordado]** F2FS para los datos en móvil y Btrfs en la fase de escritorio. Cerrar antes de la primera imagen de dispositivo.
- **[Acordado]** Una imagen EROFS por app implica un montaje por app. Con montajes respaldados por archivo (EROFS *file-backed*, desde Linux 6.12) no hace falta un dispositivo loop por imagen, lo que evita agotar `max_loop` con cientos de apps; un punto intermedio es composefs. Alternativa más simple: un único EROFS para `/usr/apps` con fs-verity por archivo, a costa del rollback por imagen. Confirmar con una prueba de carga.
- **[Acordado]** El almacén de apps usa una **partición propia**, separada del sistema base (que va en A/B), con `/usr/apps` como punto de montaje. Sustituye a la opción de remontar `/usr`.
- **[Acordado]** Contiene las imágenes EROFS firmadas; solo el instalador de SPS (su dominio SELinux) puede escribir en ella y el resto la ve en solo lectura. La integridad la da fs-verity sobre cada imagen (dm-verity no sirve en una partición que cambia). Como las apps no están en A/B, un rollback del sistema puede dejar apps compiladas contra una ABI más nueva; de ahí el interés de una ABI versionada (20.4).
- **[Acordado]** El punto de montaje del almacén de apps es `/usr/apps` (plural), con cada app en `/usr/apps/<app-id>/` (21 de octubre de 2026).

### 5.2 Formato de distribución y arranque

- **[Acordado]** Evitar la complejidad de las imágenes de Android (imágenes *sparse*, `super.img` con particiones dinámicas, `vbmeta`, `boot`/`init_boot`/`vendor_boot` y un conjunto distinto por dispositivo).
- **[Acordado]** Formato de distribución ([UKI](#g-uki) + bundle [OTA](#g-ota) elegidos sobre una ISO unificada por ser más simples de mantener):
  - **Una sola imagen de disco [GPT](#g-gpt) autodescriptiva**, con pocas particiones: [ESP](#g-esp)/arranque, sistema A, sistema B (EROFS + verity) y datos.
  - **Kernel como UKI** (*unified kernel image*): un único archivo EFI firmado con kernel, initrd y línea de comandos.
  - **Arranque [UEFI](#g-uefi)** siempre que el dispositivo lo permita (ARM SystemReady, [U-Boot](#g-uboot) con UEFI): así la imagen del sistema es la misma en todos los dispositivos y solo cambia un paquete pequeño de arranque/firmware por modelo.
  - **Actualizaciones OTA** como un bundle firmado único (manifiesto + `system.erofs` + árbol verity + firma + índice anti-rollback) que el agente de actualización escribe en el slot inactivo.
  - **ISO híbrida** solo como instalador para escritorio y placas (fase Atomic Linux), no como formato del sistema.
  - **Una única herramienta de flasheo** (nombre provisional `cosmic-flash`) que oculta las particiones al usuario.

### 5.3 Instantáneas previas a actualizar y copias de seguridad

Una copia en la misma memoria flash del teléfono ayuda a volver atrás, pero no es una copia de seguridad: si el dispositivo se pierde, se roba o falla la flash, se pierde todo junto. Por eso son dos cosas distintas.

- **[Acordado]** **Instantánea previa a la actualización** (ayuda al rollback, no es una copia de seguridad): antes de actualizar, y cuando el usuario quiera, se conserva lo que una migración de datos puede romper (bases de datos y configuración de apps y del sistema), en 1 o 2 generaciones. Fotos y vídeos no entran: pesan mucho y una migración no los toca. En Btrfs se hace con snapshots; en F2FS, con una copia de esos subconjuntos.
- **[Acordado]** **Compresión:** la hace el propio sistema de ficheros (F2FS y Btrfs soportan zstd y lz4), no una partición comprimida con XZ. XZ comprime algo mejor pero es lento y gasta batería, y sobre fotos, vídeo y audio Opus, que ya vienen comprimidos, casi no gana nada; zstd es el mejor equilibrio.
- **[Acordado]** **Dónde vive:** dentro de la partición de datos, con una cuota máxima, en lugar de una partición separada de tamaño fijo, que desperdicia espacio o se queda corta. Una partición propia para las instantáneas sigue siendo posible y serviría igual como ayuda de rollback.
- **[Acordado]** **Copia de seguridad real:** sale del dispositivo (USB, tarjeta SD, otro equipo o servidor autoalojado), va **cifrada con una clave del usuario** (si no, anularía el cifrado en reposo), es incremental y permite restaurar elementos sueltos. Para exportar todo, una imagen de solo lectura comprimida (EROFS); para copias periódicas, un formato incremental con deduplicación tipo restic o borg.
- **[Acordado]** **Desbloqueo OEM y formateo:** antes de desbloquear, Configuración ofrece la copia (externa recomendada). Desbloquear el bootloader formatea el teléfono (11.4); después se ofrece **restaurar datos** desde la copia **interna** (partición de copias) o **externa** (PC, USB, tarjeta SD, etc.).
- **[Acordado]** La pregunta y la restauración las hace el **asistente de primer arranque** del sistema (accesible, 18.3) o el modo recuperación, no el bootloader: este se mantiene mínimo, sin sistema de ficheros de datos ni criptografía de restauración.
- **[Acordado]** **La copia interna solo sirve tras un desbloqueo si su clave sobrevive.** El borrado al desbloquear existe para que quien tenga un teléfono robado no lea los datos. Una copia interna que se abra con el PIN de pantalla o con una clave del hardware no sirve: el borrado destruye la segunda, y el PIN permite fuerza bruta sin límite de intentos fuera del dispositivo. Por eso cada copia lleva una clave aleatoria envuelta de dos formas: una clave del hardware para el uso normal y una **clave de recuperación** larga que se muestra una sola vez al usuario; tras un desbloqueo, solo la de recuperación la abre.
- **[Acordado]** La copia interna solo sobrevive si el bootloader no formatea esa partición: en uno propio (U-Boot) se controla; en el del fabricante no está garantizado, y entonces la copia externa es la única fiable.
- **[Acordado]** **Copia externa incremental y cifrada**, tipo restic o borg (la herramienta concreta se elige al implementarla). Sin partición propia para las instantáneas al inicio: viven en la partición de datos. ([D-02](#d-02), 22 de octubre de 2026)

### 5.4 Pantalla de arranque

Es la primera impresión del sistema y, además, una pieza de seguridad: es donde el usuario ve el estado de arranque.

- **[Acordado]** CosmicOS tiene pantalla de arranque propia, con animación y sonido (21 de octubre de 2026).
- **[Acordado]** **Sin módulo de kernel propio.** Poner animación y audio en el espacio del kernel convierte cualquier bug en un pánico o en una vulnerabilidad; además, el audio no vive en el kernel (los drivers ALSA solo mueven muestras; decodificar y mezclar es de espacio de usuario) y los módulos se cargan tarde, desde el rootfs (3), cuando la pantalla de arranque ya tendría que estar mostrándose. Lo que hace falta ya existe sin módulo propio.
- **[Acordado]** **Cuatro etapas:**
  1. **Cargador** (U-Boot, 5.2 y 11.4): logo estático; U-Boot admite pantalla de bienvenida con imágenes BMP.
  2. **Kernel:** hereda el framebuffer del firmware con `simpledrm` y luego el driver gráfico real (KMS) toma el relevo, sin parpadeo. Aquí «DRM/KMS» es el subsistema gráfico del kernel, no la protección de contenido (11.2).
  3. **initrd:** un demonio propio mínimo, `cosmic-splash` (Rust), dibuja la animación con KMS dentro del initrd firmado de la UKI. Alternativa: [Plymouth](#g-plymouth), el estándar en Linux, con temas animados por script, aunque, hasta donde sé, no reproduce audio.
  4. **Sistema:** `cosmic-splash` cede la pantalla a PhanteonWM (9.1) sin parpadeo, traspasando el control de KMS.
- **[Acordado]** **Audio:** el sonido de inicio lo reproduce `cosmic-splash` directo por ALSA, con solo el driver de audio necesario cargado en el initrd (más initrd y más TCB), o espera a PipeWire (más tarde, pero sin cargar el initrd). Muchos teléfonos necesitan firmware y enrutado UCM para que suene el altavoz, así que el audio es lo último en estar listo. Sugerencia: audio opcional por dispositivo.
- **[Acordado]** **Seguridad:** la pantalla muestra el estado de arranque (modo, bootloader desbloqueado, huella de la clave inscrita y aviso de modo Desarrollador, 12) y no se puede falsificar: imágenes y animaciones viajan dentro de la UKI firmada, y ninguna app ni dato de usuario puede sustituirlas. Solo el modo Desarrollador avanzado admite temas propios.
- **[Acordado]** **Accesibilidad:** los avisos de arranque verificado y de recuperación se acompañan de tonos o voz (18.3): es un buen uso del audio de arranque.
- **[Acordado]** **Reglas:** la animación nunca retrasa el arranque (se dibuja en un proceso aparte y se corta al terminar), tiene un presupuesto de tamaño pequeño (las particiones de arranque lo son), gasta poco y, si falla KMS, cae a un logo estático.
- **[Acordado]** Animaciones como secuencia de imágenes o vectores; el tema lo diseña alguien de la comunidad con una guía de estilo.

### 5.5 Pérdida o robo del dispositivo

- **[Acordado]** **Recuperar los datos:** desde la copia de seguridad externa cifrada (5.3), con la clave de recuperación. Sin esa clave ni el dispositivo, nadie (tampoco el proyecto) puede abrirla: no hay puerta trasera.
- **[Acordado]** **Dispositivo nuevo:** el asistente de primer arranque ofrece restaurar desde un medio externo o un servidor autoalojado y pide la clave de recuperación. No hay cuenta central.
- **[Acordado]** **Copia en la nube:** solo hacia un destino que elija el usuario (un servidor autoalojado o un servicio suyo), cifrada en el dispositivo antes de salir. El proyecto no ofrece nube propia al inicio.
- **[Acordado]** **Mientras está perdido:** el cifrado en reposo (fscrypt), el arranque verificado y el bloqueo de pantalla protegen los datos.
- **[Acordado]** **Localizar, bloquear o borrar a distancia**: opcional, autoalojado y desactivado por defecto, con una orden firmada por el usuario que llega por el gateway push (10.2). Solo funciona si el dispositivo tiene red. ([D-03](#d-03), 22 de octubre de 2026)


### 5.6 Instalación y primer arranque

- **[Acordado]** **Opción A para el primer dispositivo** ([D-44](#d-44), 7 de noviembre de 2026): flasheo **por partición desde el PC**, guiado, y un **asistente de primer uso** en el teléfono. La «imagen única» es un **bundle**: un solo archivo con las imágenes por partición y un plan de flasheo (qué va en qué partición y en qué orden), que ejecutan `cosmic-flash` y `cosmic-fastboot` (11.12).
- **[Acordado]** El mismo origen produce además una **imagen de disco GPT completa** para QEMU y UEFI genérico (RISC-V y portátiles), donde se controla la tabla de particiones (5.2).
- **[Acordado]** Las opciones B (imagen única en una partición de paso con un instalador en el teléfono, `cosmic-install` más interfaz) y C (instalador arrancado desde la RAM con `fastboot boot`, con el sistema llegando por USB o red) quedan para cuando haya muchos dispositivos distintos o imágenes UEFI.
- **[Propuesto]** En teléfonos, hasta donde sé, el reparticionado se limita al área de datos (y `super` en los que la tienen); las particiones que el bootloader conoce no se mueven. Un instalador en el teléfono no puede bloquear el bootloader: el bloqueo con la clave de CosmicOS se hace con `cosmic-fastboot --oem-lock` desde el PC (11.12) o lo resuelve U-Boot.
- **[Propuesto]** `cosmic-fastboot` debe soportar imágenes dispersas (*sparse*) y trocear según `max-download-size`, para enviar particiones grandes.
- **[Propuesto]** Si se llegara a B o C: instalación repetible y escrita primero en la otra ranura (A/B), imagen de paso conservada hasta comprobar que el sistema nuevo arranca, verificación de firma antes de escribir nada, y un núcleo sin interfaz (`cosmic-install`) con la interfaz encima, para probarlo en QEMU y CI. La interfaz sería una aplicación de Qt sobre `eglfs` o `linuxfb`, sin compositor.

**Asistente de primer uso**

- **[Acordado]** Crea el usuario (nombre, contraseña y elección entre sudo o usuario normal, 12) y deja el sistema configurado. Todo lo que configura va a la zona de datos, nunca dentro de las imágenes firmadas del sistema (dm-verity).
- **[Acordado]** **Usuario temporal `temp-usr`:** vive en RAM (tmpfs) y desaparece al cerrar el asistente o reiniciar. No se renombra: al elegir el nombre, el asistente crea el usuario y el home definitivos y descarta el temporal.
- **[Acordado]** `temp-usr` no tiene contraseña ni shell ni inicio de sesión (tampoco por `cad`, terminal o serial) y tiene su propio dominio SELinux. Es un usuario de servicio del asistente, no una cuenta normal.
- **[Acordado]** **Un solo uso:** el asistente solo corre si no existe ningún usuario y, al terminar, queda desactivado de forma permanente; para volver a verlo hace falta un restablecimiento de fábrica.
- **[Acordado]** Si se interrumpe (por ejemplo, se apaga), al volver a arrancar no hay usuario y empieza de nuevo, sin dejar datos a medias.
- **[Propuesto]** El home definitivo nace cifrado: al elegir la contraseña, el asistente crea el directorio del usuario con **fscrypt** y una clave derivada de su credencial (12.1).

---

## 6. SPS y el formato `.sps`

**SPS (Secure-Package-System)** es el gestor de paquetes de CosmicOS.

### 6.1 Formato del paquete

- **[Acordado]** Un `.sps` contiene el **binario nativo** de la app y un `Manifest.json`.
- **[Acordado]** Lenguajes: Rust preferido; C y C++ aceptados por comodidad de los desarrolladores.
- **[Acordado]** Comando de creación: `sps --create-package <nombre-del-paquete> <directorio-del-paquete>`.

### 6.2 Generación del manifiesto

- **[Acordado]** SPS compila el código en la **máquina del desarrollador** (nunca en el dispositivo del usuario) y ejecuta el binario en un sandbox para analizar su comportamiento; con eso y la categoría de la app (por ejemplo, Cámara) genera el manifiesto con librerías, permisos mínimos y directorios RO.
- **[Acordado]** El binario de SPS se distribuye como inmutable (RO por petición al kernel).
- **[Acordado]** Tratar el análisis como **herramienta para el desarrollador** (al estilo `aa-genprof` de AppArmor), no como certificación: el análisis dinámico solo ve las rutas de código que ejecuta, y la máquina del desarrollador no es confiable por definición.
- **[Acordado]** Ejecutar la compilación en sandbox (sin red, sistema de archivos efímero), porque `build.rs`, proc macros y scripts de CMake ejecutan código arbitrario.
- **[Acordado]** Quién decide es el repositorio y el dispositivo: el manifiesto firmado por el repo es el que manda.
- **[Acordado]** Una actualización que **añade permisos** no se instala sola: el dispositivo vuelve a pedir consentimiento.
- **[Acordado]** Etiquetas sensibles (por ejemplo `<bancario>`) las asigna el repositorio tras revisión; no pueden autodeclararse en el manifiesto.

### 6.3 Estructura del manifiesto

- **[Acordado]** `Manifest.json` con tres partes: **encabezado**, **cuerpo** y **cola**.

| Parte | Contenido |
|---|---|
| **Encabezado** | Nombre de la app, desarrollador, URL del repositorio, identidad criptográfica ([YubiKey](#g-yubikey), firma GPG o SSH), lenguaje y frameworks con los que se escribió |
| **Cuerpo** | Funcionalidad, **tipo** (etiqueta: `system`, `bancario`, etc.), dependencias de librerías dinámicas centrales del sistema de destino, observaciones de SPS al ejecutar el binario en el primer sandbox, sugerencias de otros testers |
| **Cola** | Permisos declarados, tamaño de la app (KiB, MiB, GiB) y hash firmado por el servidor de origen que verifica que sea legítima |

- **[Acordado]** Validación con **esquema estricto** que rechaza campos desconocidos; el manifiesto se trata siempre como datos.
- **[Acordado]** El **tipo** lo asigna el repositorio tras revisión (ver 6.2); el valor del desarrollador es solo una propuesta.
- **[Acordado]** El manifiesto incluye un módulo de **clase de app** (cámara, mensajería, navegador, banco…). SPS trae preprogramado lo que ese tipo de apps suele requerir.
- **[Acordado]** Esa plantilla por clase es la base del filtro seccomp y de los permisos (ver 4); el manifiesto declara solo lo que se aparta de ella.
- **[Acordado]** Ajustes propuestos:
  - La firma va **separada** del contenido y se calcula sobre una serialización canónica ([JCS](#g-jcs), RFC 8785), con el identificador de clave en el encabezado.
  - Las **observaciones y sugerencias** viven en un archivo aparte, de solo anexar, por versión y firmado por quien observa: cambian después de publicar y obligarían a refirmar el manifiesto. Se tratan como datos no confiables y nunca deciden permisos.
  - El **tamaño** se guarda como entero en bytes; la interfaz lo muestra en KiB, MiB o GiB.
  - Campos explícitos para el algoritmo del hash y la versión del esquema (`schema_version`).
  - Las dependencias de librerías llevan rango de versión/[ABI](#g-abi).
  - Un bot de revisión que resalte en cada PR los cambios de permisos.
  - Claves del JSON en inglés, ASCII y `snake_case`, para estabilidad de las herramientas.

---

## 7. Arquitectura de software del sistema

| Pieza | Rol | Estado |
|---|---|---|
| **Lua** | Política declarativa (API de solo lectura, protegida por el kernel) | **[Acordado]** |
| **Go** | Orquestación del resto del sistema | **[Acordado]** |
| **Launchers en Rust** | Aplican namespaces, Landlock y seccomp, actualizan políticas y hacen `exec` | **[Acordado]** |
| **Python** | Descartado | **[Acordado]** |

**Cuidados identificados**

- El runtime de Go es multihilo y las restricciones de Landlock/seccomp se aplican por hilo, por eso las aplica un launcher pequeño en Rust (o C) y Go solo orquesta.
- El demonio en Go **corre sin privilegios** ([D-04](#d-04)): lo privilegiado lo hace el launcher en Rust. Aun así orquesta al resto del sistema, así que mantiene pocas dependencias, sin cgo si es posible, y dominio SELinux propio.
- **[Acordado]** «Lua protegida por el kernel» es ambiguo, porque el kernel no entiende Lua. Dos lecturas: (a) la política en Lua se compila en el repositorio a datos que aplican SELinux, Landlock y seccomp; (b) un intérprete dentro de un demonio privilegiado, con la librería estándar vaciada (sin `os`, `io`, `load`, `dofile` ni `package`). Se adopta (a), que deja el intérprete fuera del dispositivo.
- **[Acordado]** **Go orquesta sin privilegios y llama a un launcher en Rust**; todo lo privilegiado se escribe en Rust. Se asume explícitamente el coste de un segundo toolchain y de un runtime con recolector de basura. ([D-04](#d-04), 22 de octubre de 2026)
- Validar el manifiesto con esquema estricto; considerar parsearlo en Rust/Go, dejando Lua para la política declarativa.
- Firmas con protección contra rollback y verificación atómica (evitar [TOCTOU](#g-toctou) entre verificar e instalar).
- ABI estable para librerías del sistema, y paquetes por arquitectura (ARM64 y RISC-V).

### 7.1 libc y ABI

- **[Acordado]** libc: **[glibc](#g-glibc)**.
- **[Propuesto]** Excepción para el arranque: `cinit-early` y CinitD se enlazan **estáticamente con musl**, para ser binarios autónomos que sobreviven a un cambio de las librerías del sistema base. No es urgente: se puede empezar con glibc y cambiar después, porque es un binario separado y no mezcla ABI con el resto.
- **[Acordado]** La ABI se versiona como la documentación: `X.Y.Z`, con dígitos de 0 a 9 y acarreo (`0.0.9` → `0.1.0`). Es un número **independiente** de la versión del sistema (20.2).
- **[Acordado]** Qué cubre: el nivel base de símbolos de glibc, las versiones de Qt y de las librerías centrales, los protocolos de Wayland y la interfaz con SPS y los portales.
- **[Acordado]** Como el acarreo es mecánico, el número no dice por sí solo si algo se rompió (`0.0.9` → `0.1.0` puede no romper nada y `0.0.4` → `0.0.5` sí). Por eso cada versión de ABI publica `compatible_since`: la ABI más antigua cuyas apps siguen funcionando. Una app funciona si `compatible_since ≤ ABI de la app ≤ ABI del sistema`, y su manifiesto declara la ABI contra la que se compiló.
- **[Acordado]** glibc es compatible hacia atrás mediante el versionado de símbolos: una app compilada contra una versión antigua corre en una más nueva, no al revés. Como el repositorio recompila las apps contra un sysroot fijo (la glibc de la ABI de referencia), el problema de una app compilada contra una glibc más nueva que la del dispositivo no llega al usuario. Para apps fuera del repositorio central (11.5), el SDK ofrece ese mismo sysroot.
- **[Acordado]** Las apps viven en una partición propia fuera de A/B (5.1): tras un rollback del sistema, SPS compara las ABI y marca las apps que quedarían más nuevas que el sistema.

### 7.2 Comunicación entre procesos (IPC)

El documento no decía cómo se comunican los procesos ([IPC](#g-ipc)). Hoy ya hay piezas decididas por función: Wayland para la interfaz (9), PipeWire para audio, vídeo y cámara (10.5), `SO_PEERSEC` para identificar al emisor (10.4) y un broker de permisos para las apps (4). Falta decidir el transporte de los servicios del sistema, y lo obliga AT-SPI2, la pila de accesibilidad declarada, que corre sobre [D-Bus](#g-dbus).

- **[Acordado]** **Las apps no ven un bus general.** Piden recursos al broker (portales), que valida el permiso declarado (4); cada app solo ve sus propias conexiones.
- **[Acordado]** **D-Bus con dbus-broker como bus del sistema.** [BlueZ](#g-bluez), ModemManager, `iio-sensor-proxy`, `power-profiles-daemon` y AT-SPI2 ya lo usan, y reescribirlos no es realista. Hasta donde sé, dbus-broker puede mediar con SELinux (al adquirir nombres y al enviar mensajes). Corre en su propio dominio, sin acceso a datos de usuario, y solo lo conectan servicios del sistema.
- **[Acordado]** **Proxy filtrante por app** (al estilo de `xdg-dbus-proxy` de Flatpak) para lo poco de D-Bus que una app necesite: solo deja pasar los nombres que autoriza el manifiesto. El bus de accesibilidad de cada app pasa por un proxy que solo permite exportar su propio árbol, nunca leer el de otras.
- **[Acordado]** **Sockets Unix con `SO_PEERSEC`** para los servicios propios sencillos: logging (10.4), notificaciones (10.1) y la ventana de `kernel.root` (3.1). Protocolo mínimo e identidad del par verificada por el kernel. No se añade un cuarto protocolo.
- **[Acordado]** **Binder, a futuro** (21 de octubre de 2026). El driver [Binder](#g-binder) de Linux (con `binderfs`; desde Linux 6.18 también hay una versión en Rust) da identidad por transacción (UID, PID y contexto SELinux), llamadas síncronas con herencia de prioridad, aviso de muerte del par, paso de descriptores y ningún demonio de bus en el TCB. Frenos actuales: BlueZ, ModemManager, los portales y AT-SPI2 no hablan Binder y habría que escribir enlaces propios. Por eso los servicios propios se escriben detrás de una abstracción, para poder migrar.
- **[Acordado]** **Aislamiento del IPC:** cada bus con su dominio SELinux, sockets etiquetados en `/run`, límites de tamaño y de tasa por emisor, y todo mensaje tratado como entrada no confiable (los servicios propios, en Rust).
- **[Acordado]** **Nota sobre HongMeng.** HongMeng es el microkernel de HarmonyOS NEXT, que dejó atrás el kernel Linux. Su ventaja (servicios y drivers como procesos aislados) viene de la arquitectura de microkernel y no se traslada a un kernel Linux monolítico; lo que sí se aprovecha es la idea de aislar servicios, que aquí se logra con dominios SELinux y sandboxes. El equivalente en Linux es elegir bien el transporte, como arriba.
- **[Acordado]** Por ahora, **D-Bus y sockets Unix** (decisión del autor, 21 de octubre de 2026).

### 7.3 Pruebas, CI y validación del sandbox

- **[Acordado]** **Niveles:** pruebas unitarias (`cargo test` y equivalentes), de integración en QEMU (aarch64 y riscv64) con dispositivos virtuales, y pruebas en hardware real.
- **[Acordado]** **Suite de escape del sandbox**, pieza central del launcher (paso 1 de la hoja de ruta): apps de prueba hostiles que intentan leer `~/.ssh`, abrir sockets sin permiso, hacer `ptrace` a otros procesos, leer `/proc/<pid>/mem`, escribir en `/usr`, enviar señales a `kernel.root`, cargar módulos o usar `ioctl` de DRM no permitidos. Cada intento debe fallar y quedar registrado; cada escape que aparezca en el futuro se añade como prueba. Corre en cada PR.
- **[Acordado]** **Política SELinux:** reglas `neverallow` comprobadas al compilar la política (como Android) y pruebas de que ningún dominio de app alcanza dominios del sistema.
- **[Acordado]** **Fuzzing continuo** de todo lo que procesa entrada no confiable: manifiesto, logging, notificaciones, broker, proxy de D-Bus, `bluetoothd`.
- **[Acordado]** **Kernel:** comprobación automática del endurecimiento de la configuración (por ejemplo, con kernel-hardening-checker) en cada compilación.
- **[Acordado]** **Cadena de suministro:** reproducibilidad comprobada en CI (8.4), auditoría de dependencias (`cargo-audit`, `cargo-deny`) y análisis estático (`clippy`).
- **[Acordado]** **Auditoría externa** independiente de lo privilegiado (launcher, `kernel.root`, SPS, OTA) antes de la v1.0.
- **[Acordado]** **Laboratorio de hardware** para Bluetooth (10.7) y para regresiones de suspensión y consumo (10.8) en cada dispositivo soportado.
- **[Acordado]** **CI al inicio con runners ARM64 alojados (GitHub o similar) más QEMU**; un servidor propio ARM64 cuando haga falta y haya financiación. ([D-05](#d-05), 22 de octubre de 2026)

### 7.4 Init propio: CinitD

- **[Acordado]** systemd queda descartado. CosmicOS lleva un init propio, ligero y escrito en Rust, llamado **CinitD** (*Cosmic Init Driver*, binario `cinitd`), decisión del autor del 22 de octubre de 2026.
- **[Acordado]** Su alcance es el ya acordado en 3.1: ser el PID 1, lanzar `kernel.root`, recoger procesos huérfanos, vigilar a `kernel.root` con heartbeat y alimentar el watchdog de hardware. Cerrar la puerta de los módulos tras el arranque (3) también es suyo. Servicios, reinicios y aislamiento son de `kernel.root` y del launcher, no de CinitD.
- **[Propuesto]** Como el kernel entra en pánico si el PID 1 muere, CinitD debe ser lo más pequeño posible: enlazado estáticamente, sin dependencias dinámicas, sin runtime asíncrono, con el mínimo de código `unsafe` y probado en QEMU en cada cambio (7.3).
- **[Propuesto]** Antes de montar el sistema raíz, el initrd de la UKI (5.2) necesita su propio arranque mínimo: o el mismo binario en una fase temprana, o un `cinit-early` aparte. Lo defino al diseñar el arranque.
- **[Acordado]** **Sistema final:** el init del sistema final es CinitD, escrito desde cero en Rust. Lo único que falta decidir es la **estrategia de prototipado**, no la arquitectura.
- **[Abierto]** [D-25](#d-25): Estrategia de prototipado de CinitD: ¿escribirlo desde el primer prototipo, o usar temporalmente un init mínimo existente mientras se desarrollan `kernel.root` y el launcher? Conviene además buscar colisiones de nombre con proyectos que ya se llamen `cinit`.
- **[Propuesto]** **Reparto de tareas del PID 1** ([D-39](#d-39), refinado tras una revisión externa del 25 de octubre de 2026):

| Tarea | Dueño | Límite |
|---|---|---|
| Montajes del initrd (raíz con dm-verity, EROFS, descifrado) | `cinit-early` | No es CinitD del sistema final: cuando este arranca, la raíz ya está montada |
| Sistemas de ficheros virtuales del sistema final (`/proc`, `/sys`, `/dev`, `/run`, `volatile`) | CinitD | Solo estos; no monta apps |
| Montar y desmontar la imagen de una app | `kernel.root`, operación (3) | Ya acordado en 3.1 |
| Recoger huérfanos (`SIGCHLD`) | CinitD | Ya acordado en 3.1 |
| Eventos de dispositivos, `/dev`, reglas y etiquetas SELinux | `cdevd` (nuevo) | No carga módulos; privilegiado, así que entra en 7.6 |
| Botón de encendido, tapa y sensor térmico | `cpowerd` | Pide apagar o suspender a `kernel.root`, operación (5) |
| Orquestar el cierre de servicios (señalar, esperar, aislar) | `kernel.root` | Política de apagado |
| `sync`, desmontaje final y `reboot`/`poweroff` | CinitD | Mecanismo; solo tras la orden de `kernel.root` |
| Alimentar el watchdog de hardware | CinitD | Ya acordado en 3.1 |

- **[Propuesto]** **`cdevd` es un udev reducido:** gestiona `/dev` (nodos, permisos y enlaces), aplica reglas declarativas, etiqueta con SELinux y reacciona a eventos de dispositivos cuyo driver ya se cargó en el arranque. **No carga módulos:** lo impide `kernel.modules_disabled` (3) y lo repite la política SELinux de su dominio (`cdev_t`).
- **[Propuesto]** **El apagado tiene dos mitades.** La política es de `kernel.root`; el mecanismo final (`sync`, desmontar o remontar en solo lectura, `reboot`) es de CinitD, solo tras la orden de `kernel.root` por un socket verificado con `SO_PEERSEC`. Si `kernel.root` muere durante el apagado, CinitD **fuerza el reinicio** pasado un tiempo máximo: mejor un reinicio forzado que un bloqueo con el watchdog desatendido.
- **[Propuesto]** **Señales:** CinitD solo trata `SIGCHLD` y la orden interna de apagado. Hasta donde sé, el kernel puede mandar al PID 1 una señal al pulsar Ctrl+Alt+Supr; CinitD la desactiva con `reboot(2)`.

### 7.5 Nombres de los componentes

- **[Acordado]** Convención (confirmada el 22 de octubre de 2026), tomada de los nombres del autor (`cboard`, `cinitd`): los demonios llevan `c` delante y `d` detrás (`c<nombre>d`) y las herramientas y repositorios llevan prefijo `cosmic-` (como `cosmic-splash` y `cosmic-manifest`).

| Rol | Nombre propuesto | Estado |
|---|---|---|
| Init (PID 1) | `cinitd` (CinitD) | **[Acordado]** |
| Supervisor | `kernel.root` | **[Acordado]** |
| Compositor | PhanteonWM | **[Acordado]** |
| Shell | CreationDE | **[Acordado]** |
| Launcher en Rust | `clauncher` | **[Acordado]** |
| Orquestador en Go | `corchd` | **[Acordado]** |
| Demonio de notificaciones | `cnotifd` | **[Acordado]** |
| Demonio de logging | `clogd` | **[Acordado]** |
| Cliente de push | `cpushd` | **[Acordado]** |
| Teclado en pantalla | `cboard` | **[Acordado]** |
| Broker de permisos | `cbrokerd` | **[Acordado]** |
| Demonio de energía | `cpowerd` | **[Acordado]** |
| Gestor de dispositivos | `cdevd` | **[Propuesto]** |
| Servicio de actualizaciones | `cupdated` | **[Acordado]** |
| Pantalla de arranque | `cosmic-splash` | **[Acordado]** |
| Cortafuegos | Frontera FireWall (`cfronterad`) | **[Acordado]** el nombre; el binario, **[Propuesto]** |
| Gestor de paquetes | `sps` | **[Acordado]** |

- **[Acordado]** Convención y nombres confirmados por el autor ([D-27](#d-27), 22 de octubre de 2026).

### 7.6 Auditoría de código y pruebas ofensivas

- **[Propuesto]** **Revisión por pares:** todo cambio se revisa; lo privilegiado (launcher, `kernel.root`, CinitD, `cdevd`, SPS, OTA) exige dos revisores, como ya ocurre con los permisos de los manifiestos (8.1).
- **[Propuesto]** **Herramientas automáticas** en cada PR, además de las de 7.3: analizadores estáticos de seguridad para C y Go (por ejemplo, clang-tidy y gosec), y Miri para las zonas con `unsafe` en Rust.
- **[Propuesto]** **Auditoría externa independiente** de lo privilegiado antes de la v1.0 y en cada versión mayor, con el informe publicado.
- **[Propuesto]** **Pentesting (equipo rojo)** antes de la v1.0, con alcance en el sandbox, la cadena de actualización, el repositorio y la infraestructura de claves.
- **[Propuesto]** **Recompensas por vulnerabilidades:** con la política de divulgación coordinada ya acordada en 11.6 (`SECURITY.md`, 90 días). Hasta que haya financiación, el pago es el reconocimiento público.
- **[Abierto]** [D-29](#d-29): Pentesting externo y bug bounty (presupuesto y modelo).

### 7.7 Presupuesto de componentes propios

Los componentes de este documento ya pasan de veinte. Esta tabla separa lo que se escribe primero de lo que se reutiliza o se aplaza (principio 9).

| Grupo | Componentes | Qué se hace |
|---|---|---|
| **Primer hito (se escribe)** | CinitD, `kernel.root`, `clauncher`, `corchd`, `sps`, la biblioteca `cosmic-core`, un PhanteonWM mínimo, `cosmic-repo` para el índice firmado y `cosmic-build` (8.8) | Es lo mínimo para el modelo de seguridad y para instalar una app (13) |
| **Reutilizar al inicio** | `cdevd` (eudev o mdev), `csettingsd` (archivos de configuración simples), `cupdated` (una función de SPS), `cboard` (un teclado existente), `cnotifd` (un demonio de notificaciones existente) | Se sustituyen por algo propio solo si hay una razón concreta |
| **Aplazados** | `cfronterad` (tras su primer hito, 10.9), `cpushd`, `cpowerd`, `clogd` (hasta que el broker lo exija), CreationDE, el lector de pantalla propio, el almacén de secretos propio, y `cosmic-fastboot`, `cad` y `cosmic-usb` (11.12: hasta tener un teléfono físico) | Entran cuando el primer hito funciona |

- **[Acordado]** En el espacio de trabajo de Cargo, `default-members` solo lista el grupo del primer hito: `core/cinitd`, `core/kernel.root`, `core/clauncher`, `pkg/sps`, `libs/cosmic-core` y `tools/cosmic-repo` (ya configurado por el autor, 6 de noviembre de 2026). `corchd` está en Go y se compila aparte. Lo demás existe como andamiaje, pero no se compila por defecto ni exige mantenimiento hasta que le toque.
- **[Propuesto]** Cambiar un componente de grupo se anota en este documento con la razón.
- **[Abierto]** `cosmic-verifier` (C++20, dentro de `cosmic-build`) cuenta como componente propio. Si solo comprueba hashes y tamaños, `sha256sum` y `stat` lo sustituyen; si hace algo que ninguna otra pieza hace, debe documentarse qué (principio 9).

---

## 8. Repositorio, firmas y revocación

### 8.1 Repositorio central

- **[Acordado]** Repositorio central **desde el día 1**, público y estudiable por cualquiera.
- **[Acordado]** **Varios repositorios oficiales, a la manera de Arch** (22 de octubre de 2026): **[Main]** (el núcleo del sistema y las apps esenciales), **[Space]** (el resto de las apps de código abierto) y **[MTL]** (apps propietarias: cualquier empresa puede enlazar su propio repositorio para publicar apps `.sps`). Main y Space cumplen todas las reglas de esta sección; MTL las cumple de otra forma ([D-26](#d-26)). **MTL** significa *Main Third-party License* (aclaración del autor, 22 de octubre de 2026): el repositorio de apps bajo licencias de terceros, normalmente propietarias.
- **[Acordado]** Modelo de **pull requests**: los manifiestos de cada app se editan de forma colectiva y pública.
- **[Acordado]** CODEOWNERS por app; quitar permisos es fácil de aceptar, **añadirlos** exige revisión del mantenedor más un segundo revisor.
- **[Acordado]** Main y Space recompilan desde fuente en su infraestructura, en sandbox, ejecuta el análisis y firma; el dispositivo solo descarga, verifica firma y hash, e instala.
- **[Acordado]** Main y Space compilan siempre (modelo Flathub); los binarios de desarrolladores con verificación reproducible quedan para más adelante.
- **[Acordado]** Versión temporal sin infraestructura propia: repo en GitHub por app, GitHub Actions para compilar en sandbox, firma con [cosign](#g-cosign), provenance [SLSA](#g-slsa) e índice estático firmado con [TUF](#g-tuf) servido desde hosting estático.

### 8.2 Identidad y firmas

- **[Acordado]** Sin ligar cuenta de GitHub: se exige la **clave criptográfica del desarrollador**.
- **[Acordado]** La clave de publicación vive en **YubiKey** (hardware).
- **[Acordado]** Una segunda llave de respaldo guardada offline y un procedimiento de recuperación por mantenedores, para evitar un punto único de fallo.
- **[Acordado]** Dos claves: la del desarrollador (autoría) y la del repositorio (firma del binario que llega al dispositivo).
- **[Acordado]** Reputación asociada a la clave (antigüedad, historial de PRs aceptados); las claves nuevas no publican apps con permisos sensibles sin revisión humana.
- **[Acordado]** Expiración corta de claves con rotación periódica; metadatos con TUF; registro de transparencia estilo Sigstore.

### 8.3 Revocación y cuarentena ("criogenia")

- **[Acordado]** Si se detecta que una app llega con una clave inválida o comprometida, el sistema la **congela**: no se ejecuta (se muestra el aviso "aplicación en cuarentena por llave criptográfica comprometida"), pero **se conservan todos sus archivos y datos**, hasta recibir una actualización con una clave nueva.
- **[Acordado]** Tres estados distintos: **huérfano** (clave válida, sin mantenedor), **abandonado** (congelado, deprecado) y **comprometido** (revocado).
- **[Acordado]** La revocación afecta a las versiones firmadas **después** de la fecha de compromiso.
- **[Acordado]** Detección: reporte del desarrollador (clave de revocación offline), reportes de la comunidad, rotación periódica y cuarentena automática de versiones que de pronto piden más permisos o cambian de mantenedor.
- **[Acordado]** El archivo `timestamp` de vida corta de TUF permite al dispositivo saber si su lista de revocaciones está vieja.
- **[Acordado]** Los paquetes huérfanos son instalables con un aviso visible; no se congelan.
- **[Acordado]** **Suelo de seguridad por app.** El suelo de 17.5 cubre solo el sistema; las apps viven fuera de A/B y conservan la imagen anterior para rollback, así que un descuido (o un atacante con acceso al instalador) podría volver a montar una versión vieja y vulnerable. Cada versión firmada de una app lleva un `security_floor` en sus metadatos TUF, y el repositorio lo sube cuando una versión corrige una vulnerabilidad.
- **[Acordado]** Por debajo del suelo, la versión queda **congelada** igual que en la criogenia (no se ejecuta, se conservan sus datos) y SPS purga la imagen antigua. Si el usuario la necesita por compatibilidad, solo puede reactivarla con autorización local y un aviso de las vulnerabilidades conocidas (como en 17.5), y **nunca** para apps con etiquetas sensibles (`<bancario>`, `accessibility`, `system`).
- **[Acordado]** El índice de TUF protege los metadatos, pero no impide ejecutar una imagen antigua que el instalador conserva: por eso el suelo se comprueba **al ejecutar**, no solo al instalar.

### 8.4 Compilaciones reproducibles

- **[Acordado]** **Objetivo:** que cualquiera pueda recompilar un paquete y obtener exactamente el mismo binario que firmó el repositorio, de modo que no haga falta confiar en la infraestructura de compilación.
- **[Acordado]** **Proceso:**
  1. Entorno de compilación fijado por *digest* (imagen de contenedor con el sysroot de la ABI, ver 7.1).
  2. Sin red durante la compilación; dependencias congeladas y vendorizadas (`Cargo.lock` y fuentes en el repositorio).
  3. Fechas y rutas normalizadas (`SOURCE_DATE_EPOCH`) y orden determinista de archivos, también al generar las imágenes EROFS.
  4. Al menos dos compiladores independientes (la infraestructura del repositorio y un verificador de la comunidad) comparan los hashes.
  5. Con cada paquete se publican el hash, la procedencia SLSA y una entrada en el registro de transparencia.
- **[Acordado]** Si dos compilaciones no coinciden, el paquete no se publica; o se marca «no reproducible», con menos confianza y revisión humana reforzada.
- **[Acordado]** Proyectos enormes como Chromium no son fáciles de reproducir bit a bit: se aceptan marcados «no reproducible» mientras no se logre.
- **[Acordado]** Las apps de repositorios de terceros y de MTL se muestran como **«sin verificar»**: no pasan por la compilación reproducible ni por la revisión por PR de Main y Space, y la interfaz lo dice. ([D-06](#d-06), 22 de octubre de 2026)

### 8.5 Obtención del código fuente (`repo` y manifiesto XML)

- **[Acordado]** El código del sistema se obtiene con [`repo`](#g-repo) y un manifiesto XML, como en AOSP y OpenHarmony, en lugar de un `git clone` normal repositorio por repositorio (decisión del autor, 21 de octubre de 2026).
- **[Acordado]** **Por qué encaja:** CosmicOS son muchos repositorios (kernel, launcher, `kernel.root`, SPS, PhanteonWM, CreationDE, demonios, parches de Chromium y de Qt, paquetes de firmware por dispositivo). `repo sync` los baja a la vez y el manifiesto fija la revisión exacta de cada uno: es el **archivo de bloqueo del sistema entero**, justo lo que necesitan las compilaciones reproducibles (8.4).
- **[Acordado]** **Nombre:** el manifiesto de `repo` es un manifiesto de código fuente, no de permisos. Para no confundirlo con el `Manifest.json` de las apps (6.3), el repositorio se llama `cosmic-manifest` y su archivo principal `default.xml`.
- **[Acordado]** **Grupos** de `repo` (`launcher`, `ui`, `kernel`, `mobile`, `desktop-dev` y `milestone1`, con todo lo que el primer hito necesita para compilar) para bajar solo lo necesario, y clones parciales o superficiales para el kernel y Chromium, que son enormes.
- **[Acordado]** **Ramas del manifiesto = canales** (`nebula`, `pulsar`, `singularity`, `quasar`, `eventhorizon`, 20.3); una versión estable es un tag firmado del manifiesto.
- **[Acordado]** **El kernel viene de un fork propio**, no de kernel.org: el manifiesto apunta al repositorio `linux` de la organización (rama `cosmic/6.x`, `clone-depth="1"`), que lleva `arch/` purgado (3).
- **[Acordado]** **Firma:** hasta donde sé, `repo` verifica la firma de su propio código, no la del manifiesto. El tag firmado del manifiesto se verifica (`git verify-tag`) en el script de arranque y en el CI.
- **[Acordado]** **Contribuciones:** `repo upload` es el flujo de Gerrit; el proyecto usa PR (8.1), así que cada repositorio se contribuye con git normal y el manifiesto solo cambia para fijar revisiones, con un bot que las actualiza.
- **[Acordado]** **Las apps no usan `repo`:** cada app es un repositorio git propio (8.1) y SPS la referencia por su commit.
- **[Acordado]** `repo` está escrito en Python: es una herramienta de desarrollo, fuera del TCB, y por eso no contradice que Python esté descartado para el sistema (7).
- **[Acordado]** `repo` solo para el árbol del sistema; para el SDK y las apps, un `git clone` simple de un repositorio mínimo. ([D-07](#d-07), 22 de octubre de 2026)

### 8.6 Respuesta a incidentes (clave o servidor comprometido)

Procedimiento operativo para cuando algo de lo descrito en 8.2 y 8.3 falla. Cubre cuatro casos: clave de un desarrollador, clave del repositorio (los roles de TUF), clave raíz o de las OTA, y servidor de compilación.

- **[Propuesto]** 1. **Detectar y declarar:** quien lo detecta avisa a los mantenedores con clave y se abre un incidente en un canal privado.
- **[Propuesto]** 2. **Contener:** se congelan las publicaciones de lo afectado (dejar de firmar el índice, o detener las compilaciones si el servidor es sospechoso).
- **[Propuesto]** 3. **Revocar:** clave de desarrollador, criogenia de sus apps (8.3); claves del repositorio o de las OTA, rotación con el umbral de 8.2. Hasta donde sé, TUF contempla la rotación de claves y de la raíz; los dispositivos deben poder aceptar una raíz nueva sin reinstalar.
- **[Propuesto]** 4. **Avisar:** un aviso firmado dentro del índice (visible en Configuración), el boletín de seguridad (17.6) y un comunicado público. Los dispositivos sin conexión lo ven en su próxima consulta; no se usa un push dirigido, porque no hay identificadores (19).
- **[Propuesto]** 5. **Recuperar:** reconstruir lo afectado desde fuente limpia (las compilaciones reproducibles de 8.4 permiten comprobarlo) y volver a publicar con claves nuevas.
- **[Propuesto]** 6. **Aprender:** informe posterior público en un plazo razonable (propongo 14 días) y un **simulacro anual** de rotación de claves.
- **[Abierto]** [D-30](#d-30): Canal de contacto y página de estado para incidentes.


### 8.7 Estructura del repositorio de código

- **[Acordado]** **Organización:** `OpenCosmicProject` (OCP) es el nombre oficial de la organización (6 de noviembre de 2026).
- **[Acordado]** **Un repositorio git por carpeta.** Sustituye a la propuesta de monorepo: cada componente tiene su repositorio y `repo` los une (8.5). Hoy el autor trabaja en un único árbol local, sin publicar, que se parte al subirlo. El mismo esquema se aplicará a otros proyectos de la organización, por definir.
- **[Acordado]** Estructura de primer nivel del árbol local: `core/` (`cinitd`, `cdevd`, `clauncher`, `corchd`, `kernel.root`), `daemons/` (`cbrokerd`, `cfronterad`, `clogd`, `cnotifd`, `cpowerd`, `cpushd`, `csettingsd`, `cupdated`), `ui/` (`cboard`, `cosmic-splash`, `creationde`, `phanteonwm`), `pkg/sps`, `tools/` (`cosmic-flash`, `cosmic-repo`, `cosmic-build`, `fastboot`, `cad`), `libs/` (`cosmic-core`, `cosmic-usb`), `sdk/cosmic-sdk`, `kernel/` (`configs/`, `scripts/` y `linux/`), `patches/` (`chromium`, `qt`), `device/firmware`, `cosmic-manifest/` y `Docs/`.
- **[Acordado]** **No se versionan** `target/` (salida de Cargo) ni `kernel/linux/` (el kernel es su propio repositorio, 3). Un `.gitignore` lo deja escrito.
- **[Propuesto]** Añadir al `.gitignore` `out/`, `.ninja_log` y, si son salida de compilación, `tools/bin/`.
- **[Acordado]** `Docs/` se queda con mayúscula inicial: no forma parte del ecosistema del sistema, es solo la documentación (decisión del autor, 6 de noviembre de 2026). La regla de minúsculas de 5 es para las rutas del sistema.
- **[Acordado]** **`Docs/` es práctica, la especificación manda.** La especificación es la única fuente de verdad; los documentos de `Docs/` (arquitectura, repositorios, construcción, kernel, SPS) son guías breves que enlazan a la sección correspondiente en lugar de copiarla, para que no se desincronicen. Los enlaces deben ser relativos y apuntar a la versión vigente.
- **[Propuesto]** `cosmic-manifest` es un repositorio propio con `default.xml`. Falta decidir qué repositorio contiene los archivos de la raíz del espacio de trabajo (`Cargo.toml`, `Cargo.lock`, `.gitignore`): `cosmic-build`, que los coloca con `<linkfile>`, es la opción propuesta.
- **[Propuesto]** Nombres: el kernel como `cosmic-linux` (regla `cosmic-` para repositorios) y un único nombre para el gestor de paquetes (`sps`) en todos los documentos.
- **[Abierto]** [D-40](#d-40): Documentar lo que ya existe en el árbol (`libs/cosmic-core`, `tools/cosmic-flash`, `tools/cosmic-repo`) y su relación con el resto.

### 8.8 Sistema de construcción (`cosmic-build`)

- **[Acordado]** No se depende de un único Makefile: hay un sistema de construcción común, al estilo de OpenHarmony (que usa GN y Ninja, con un frontend fino), sin Python en nuestras herramientas ni en la imagen. ([D-41](#d-41), 7 de noviembre de 2026)
- **[Acordado]** **Frontend en Bash** (`cosmic-build`: producto, canal, arquitectura, número de trabajos y meta: `milestone1`, `kernel`, `check`, `clean`) y **Ninja** como motor, con un generador de `build.ninja` en Bash. Se reutiliza Ninja en lugar de escribir un motor propio (principio 9).
- **[Propuesto]** **GN** para describir el grafo cuando el proyecto crezca, con plantillas para Cargo, Go, CMake y el kernel. Un motor propio en C++ solo se reconsideraría con razones concretas, como en D-38.
- **[Acordado]** **Python solo dentro del contenedor de construcción**, y solo si lo exigen Chromium o Qt WebEngine; nunca en la imagen del sistema (7).
- **[Acordado]** Aceleración: `ccache` y `sccache` si están disponibles, `lld` donde el compilador lo permita y un `target/` compartido de Cargo.
- **[Acordado]** **Comprobado el 7 de noviembre de 2026:** `./cosmic-build all` compila `cinitd`, `corchd` (Go), `kernel.root`, `clauncher`, `sps` y `cosmic-repo` en 16 s; `milestone1` y `check` terminan bien; el kernel arm64 compila hasta `Image.gz` en 14,9 s (incremental) y, al borrar `Image.gz`, la regla de Ninja lo vuelve a compilar.
- **[Propuesto]** **Por corregir (importante primero):** (1) **compilación cruzada:** hoy `-target arm64` solo afecta al kernel; Cargo y Go compilan para el PC (los binarios salen en `target/release/`). Hay que pasar `--target aarch64-unknown-linux-gnu` (y `riscv64gc-unknown-linux-gnu`; CinitD, si adopta musl, con el target musl), configurar el enlazador cruzado en `.cargo/config.toml` y compilar Go con `GOOS=linux GOARCH=arm64 CGO_ENABLED=0`; (2) la regla del kernel debe pasar `ARCH` y `CROSS_COMPILE` también al paso de configuración y fallar si `make listnewconfig` devuelve algo (3); (3) las reglas de Cargo, Go y `make` se ejecutan siempre, con `restat`, y es la herramienta de cada una la que decide lo incremental; (4) `cargo build --locked` (y `cargo vendor` para compilaciones sin red); (5) una sola invocación de Cargo con varios `-p`, en lugar de una por componente; (6) un paso de instalación que copie los binarios a un directorio de montaje por producto y arquitectura (`out/<arquitectura>-<producto>/`), de donde sale el bundle (5.6); (7) `clean-kernel` aparte de `clean`, y no compilar nunca como root (hay archivos del árbol propiedad de root); (8) un contenedor con versiones de herramientas fijas (8.4); (9) pasar `shellcheck` a los scripts y probar riscv64; (10) el binario `logengine` no sigue la convención de nombres (7.5); (11) para armar la UKI, `ukify` está escrito en Python: usar `objcopy` con el stub de systemd o aceptarlo solo dentro del contenedor.

---

## 9. Interfaz de usuario

### 9.1 Componentes

| Componente | Rol | Estado |
|---|---|---|
| **PhanteonWM** | Compositor [Wayland](#g-wayland) (en Wayland, WM y compositor son lo mismo) | **[Acordado]** como nombre y rol |
| **CreationDE** | Shell: lanzador de apps, barra de estado, pantalla de bloqueo, centro de notificaciones | **[Acordado]** |
| **[Qt6](#g-qt) (kit completo)** | SDK oficial y toolkit recomendado | **[Acordado]** |
| **GTK4 + libadwaita** | Toolkit secundario, para aplicaciones básicas | **[Acordado]** |
| **Wayland** | Protocolo de pantalla | **[Acordado]** |
| **[Vulkan](#g-vulkan)** | API gráfica principal | **[Acordado]** |

> **Nota de nombre:** se escribe *Phanteon* (con «n»). La similitud con *Pantheon*, el entorno de elementary OS, es conocida y la acepta el autor; la grafía anterior *Phateon* era un error de ortografía (22 de octubre de 2026).

### 9.2 Decisiones y recomendaciones

- **[Acordado]** Base del compositor: **[Smithay](#g-smithay)** (Rust), por su seguridad y futuro. El proyecto contribuirá a su desarrollo upstream.
- **[Acordado]** Cada app recibe su **propio socket Wayland** mediante `wp-security-context-v1`; los protocolos privilegiados (captura de pantalla, teclado virtual, overlays) solo están disponibles para componentes del sistema.
- **[Acordado]** "Podar" GTK, Qt y Wayland por **configuración de compilación** (quitar backends X11, Broadway y plataformas no usadas), no mediante forks: son userspace sin código por arquitectura, y mantener forks implica rebases continuos.
- **[Acordado]** Vulkan como ruta principal del compositor, con fallback a GLES y, como último recurso, renderizado por software (llvmpipe), porque el soporte Vulkan en GPUs móviles es desigual.
- **[Acordado]** Coherencia visual: libadwaita impone mucho su estilo, así que hace falta una hoja de estilo propia para que las apps GTK y Qt se vean consistentes.
- **[Acordado]** Navegación por **gestos como modo predeterminado**, manteniendo los botones.
- **[Acordado]** Los gestos de borde pertenecen al compositor, con prioridad sobre las apps; ofrecer navegación alternativa por accesibilidad.
- **[Acordado]** Ciclo de vida de apps: congelar apps en segundo plano con el freezer de cgroups para ahorrar batería.
- **[Acordado]** Rotación, densidad y escalado fraccional desde el diseño.

### 9.3 GPU

La GPU es alcanzable desde todas las apps (Vulkan y GLES), así que sus drivers del kernel exponen una superficie de ataque mayor que la mayoría de los periféricos; hasta donde sé, los drivers de GPU móviles acumulan muchas vulnerabilidades. Merece el mismo tratamiento que el módem.

- **[Acordado]** Solo drivers abiertos y upstream (Mesa en espacio de usuario; en el kernel, por ejemplo, freedreno para Adreno y panfrost o panthor para Mali). Un dispositivo sin driver abierto utilizable no se soporta.
- **[Acordado]** Las apps usan únicamente nodos de render (`/dev/dri/renderD*`); `/dev/dri/card*` es solo del compositor. Cada nodo lleva etiqueta SELinux.
- **[Acordado]** El filtro seccomp base (4) para las clases con GPU incluye una **lista de `ioctl` permitidos por driver**, porque `ioctl` es una sola syscall y filtrar por syscall no basta; es la técnica que ya usan los navegadores.
- **[Acordado]** Espacio de direcciones propio por contexto de GPU (MMU e IOMMU de la GPU) y firmware de la GPU incluido en la lista de blobs de cada dispositivo (16.4).
- **[Acordado]** El WebView y los renderizadores pueden funcionar sin GPU (llvmpipe) si se quiere reducir la superficie, a costa de rendimiento.
- **[Acordado]** GPU al inicio: Adreno (freedreno) y Mali (panfrost y panthor); otras (PowerVR, por ejemplo) cuando tengan drivers abiertos utilizables.

### 9.4 Rendimiento: objetivos

- **[Acordado]** Objetivos iniciales, a calibrar midiendo el dispositivo con su sistema de fábrica; en el mismo hardware, nada peor que ese sistema:
  - Arranque en frío hasta la pantalla de bloqueo: **30 s** como máximo.
  - Apertura en frío de una app: **1 s** como máximo.
  - Interfaz: 60 fotogramas por segundo sin tirones (menos de 16,6 ms por fotograma) y latencia de toque a pantalla de **100 ms** como máximo.
  - Consumo en reposo: el criterio de 10.8.
- **[Acordado]** Se miden en pruebas reproducibles en hardware real; una regresión notable bloquea la versión.

### 9.5 Localización e idiomas

- **[Acordado]** Español e inglés al inicio. El sistema nace internacionalizable: ningún texto fijo en el código, y el linter de SPS avisa de los que queden.
- **[Acordado]** **Traducciones:** gettext (`.po`) como formato de intercambio, con conversión a los formatos de Qt (`.ts`), alojado en una plataforma de traducción de software libre (por ejemplo, Weblate).
- **[Acordado]** **Datos regionales** (fechas, números, monedas y orden) con CLDR/ICU; fuentes de amplia cobertura (Noto), escritura de derecha a izquierda y teclados e IME para otros alfabetos mediante `input-method-v2` (10.3).
- **[Acordado]** **Español e inglés hasta la v1.0.** Un idioma nuevo entra cuando tiene un traductor responsable y pasa una revisión de la comunidad. ([D-08](#d-08), 22 de octubre de 2026)

---

## 10. Servicios del sistema

### 10.1 Notificaciones

- **[Acordado]** Demonio de notificaciones **separado** del compositor (y del shell).
- **[Acordado]** La app genera un **token efímero** con caducidad de **10 a 30 segundos**; el demonio dibuja la notificación y esta deja de existir para el sistema, aunque el usuario puede seguir viéndola hasta que la borre.
- **[Acordado]** El compositor solo ofrece un protocolo mínimo para pintar overlays con prioridad (sobre apps a pantalla completa y la pantalla de bloqueo) y respeta el modo No molestar. Motivo: el compositor es la pieza más privilegiada de la UI y no debe parsear contenido de notificaciones.

### 10.2 Push remoto y "GMS propio"

- **[Acordado]** No se usa UnifiedPush como cliente oficial: el sistema tiene su propio gateway.
- **[Acordado]** El push es una **señal de despertar**: el mensaje real vive en el servidor de la app (Matrix u otro) y, al reconectar, la app sincroniza. Si el dispositivo está offline, la notificación se descarta; alternativa menos elegante: retenerla cifrada hasta 6 horas.
- **[Acordado]** El "GMS propio" es **pequeño y autoalojable**, escrito en C/Rust: push, repositorio de apps y sincronización de cuenta; más adelante, geolocalización por WiFi/celdas. Mapas con OpenStreetMap.
- **[Acordado]** El gateway habla **[Web Push](#g-webpush)** (RFC 8030/8291/VAPID): da compatibilidad con Matrix y ntfy; `TTL` y `Topic` para la retención y el reemplazo de avisos.
- **[Acordado]** El servidor de la app siempre envía al gateway, y este decide si entrega, retiene o descarta; así el servidor no conoce la presencia del dispositivo. P2P no encaja para retener: se necesita un nodo intermedio, y con contenido cifrado de [extremo a extremo](#g-e2e) no puede leerlo.
- **[Acordado]** Límites de tasa por app y por token, y tope de tamaño de payload.
- **[Propuesto]** Quién paga el gateway por defecto y cómo se sostiene: ver 11.11 ([D-34](#d-34)).

### 10.3 Teclado en pantalla

- **[Acordado]** Base parecida a GBoard, sobre Wayland.
- **[Acordado]** Proceso **separado** y con sus datos inaccesibles (sin lectura ni escritura) para cualquier app.
- **[Acordado]** Protocolos `input-method-v2` y `text-input-v3`; bases posibles: Squeekboard (Rust) o Maliit (Qt); el swipe typing y la predicción serían trabajo propio, con modelo **local**.
- **[Acordado]** Sin permiso de red (con un namespace de red vacío, ver 4), sin acceso al home; no aprender de campos de contraseña o dato sensible; datos cifrados en reposo y excluidos de copias; opción visible de "borrar lo aprendido".
- **[Acordado]** Ni siquiera acceso de solo escritura a las apps: permitirlo habilitaría envenenar las sugerencias (por ejemplo, un IBAN o una URL de phishing).

### 10.4 Logging

- **[Acordado]** **Un único demonio de logging** para todo el sistema.
- **[Acordado]** Estructura por app: `~/.var/logs/<app>/{json,raw}`.
  - `raw`: logs en crudo, accesibles **solo** para el protocolo de logging; inaccesibles para el resto.
  - `json`: generado por el demonio a partir del crudo, descartando datos sensibles poco relevantes al error.
- **[Acordado]** Las apps **solo envían log en crudo**.
- **[Acordado]** Objetivo: que el sistema, el usuario o un técnico de reparación puedan ver por qué falló algo.
- **[Acordado]** Etiqueta SELinux propia para el directorio de logs (dominio del demonio de logging), de modo que ni la app dueña ni otras puedan leer, borrar o editar.
- **[Acordado]** Identificar al emisor con `SO_PEERCRED` y `SO_PEERSEC` (UID, PID, etiqueta SELinux), no por el nombre que declare en el mensaje.
- **[Acordado]** El demonio parsea entrada no confiable: escribirlo en Rust, con límites duros (longitud de línea, ráfagas), fuzzing, separación de privilegios (el proceso que parsea no es el que escribe `raw`) y seccomp.
- **[Acordado]** Redacción por **lista de campos permitidos** (marca de tiempo, severidad, componente, plantilla del mensaje, códigos de error, trazas) y enmascarado por tipo (`<email>`, `<token>`, `<ip>`) en vez de borrado silencioso; reglas versionadas. La redacción es de mejor esfuerzo: `RAW` sigue siendo sensible y va cifrado en reposo con retención corta.
- **[Acordado]** Cuota, rotación y límite de tasa por app; hash encadenado en `raw`; cada entrada de `json` referencia el desplazamiento y hash de su línea en `raw`.
- **[Acordado]** SDK con formato mínimo estable (clave=valor para severidad, componente y código de error) y backends para `tracing`/`log` y Qt.
- **[Acordado]** **Registro de auditoría de seguridad** separado y fuera del alcance del usuario (denegaciones de SELinux, cuarentenas, revocaciones, cambios de política), solo-anexar y con hash encadenado, visible al usuario en solo lectura.
- **[Acordado]** **Paquete de diagnóstico** para técnicos: lo genera el usuario, con vista previa y redactado; los volcados de memoria solo con consentimiento explícito y cifrados. Los logs se conservan al desinstalar durante un tiempo corto y siempre en cuarentena.
- **[Acordado]** Carpetas de logs en minúsculas (más «Linux friendly») y un nivel menos que en el diseño original (21 de octubre de 2026).

### 10.5 Cámara

- **[Acordado]** Ninguna app accede directamente a `/dev/video*`: **[PipeWire](#g-pipewire)** concede el acceso (con el broker de permisos).
- **[Acordado]** Librería **propia en Rust**, especialmente para el posprocesado.
- **[Acordado]** Dividirla en dos: captura y control (3A/ISP) con **[libcamera](#g-libcamera)**, y posprocesado (reducción de ruido, HDR, modo nocturno) en la librería propia, idealmente con cómputo en GPU (wgpu/Vulkan).

### 10.6 Sensores, telefonía y energía

- **Sensores — [Acordado]:** subsistema **[IIO](#g-iio)** del kernel + `iio-sensor-proxy` (acelerómetro, luz, brújula, proximidad por D-Bus) al inicio; demonio propio en Rust después, si se necesita fusión o bajo consumo.
- **Telefonía — [Acordado]:** **[ModemManager](#g-ofono)** por estabilidad (mejor soporte de módems QMI/MBIM y más activo), detrás de una abstracción propia para poder cambiarlo. El firmware del módem es un [blob](#g-blob) cerrado con acceso DMA: aislar con IOMMU y asumirlo como excepción explícita al «sin blobs».
- **Energía — [Acordado]:** oFono no gestiona energía; usar suspensión del kernel con wakeup sources, el freezer de cgroups y un demonio de perfiles (`power-profiles-daemon` al inicio, uno propio en Rust después).

### 10.7 Audio, llamadas y Bluetooth

- **[Acordado]** **Audio:** PipeWire (ya acordado para la cámara) con WirePlumber como gestor de políticas. En un teléfono la voz de una llamada va del módem al códec de audio por un camino de hardware; PipeWire y ModemManager lo enrutan con perfiles ALSA UCM propios de cada dispositivo. Sin ese trabajo no hay llamadas con altavoz ni auriculares.
- **[Acordado]** **Bluetooth:** se parte de **BlueZ** y se mejora encima (21 de octubre de 2026), para que el Bluetooth no sea una ruleta rusa.
- **[Acordado]** **Mejorar sin bifurcar a ciegas:** los arreglos genéricos van a upstream (BlueZ es GPL-2.0 o posterior y una bifurcación obliga a rebasarla con cada versión); lo específico de CosmicOS va en capas propias encima.
- **[Acordado]** **Dónde está la ruleta rusa:** casi nunca en el protocolo, sino en la interoperabilidad con chips y dispositivos concretos. Plan: (1) laboratorio automatizado con dispositivos reales (auriculares, relojes, coches, audífonos, teclados) y una matriz de compatibilidad pública; (2) base de datos de *quirks* por chip y por dispositivo; (3) un gestor propio sobre BlueZ (estado de conexión, reintentos con *backoff* y reconexión) en lugar de dejar esa lógica en cada app; (4) registros por conexión, con el volcado HCI (`btmon`) en el paquete de diagnóstico (10.4).
- **[Acordado]** **Seguridad:** BlueZ está escrito en C y procesa datos que llegan por el aire (hasta donde sé, BlueBorne, en 2017, afectó a BlueZ y al kernel). `bluetoothd` va en su dominio SELinux sin más privilegios de los necesarios, con fuzzing continuo.
- **[Acordado]** **Perfiles y códecs:** BlueZ habla por D-Bus (7.2). En A2DP, SBC es obligatorio y libre; AAC, aptX y LDAC quedan como opcionales por las patentes (11.1). [LE Audio](#g-leaudio) (códec LC3, con implementación abierta, hasta donde sé) importa además por accesibilidad: audífonos y ayudas auditivas. El soporte en Linux todavía es desigual: verificarlo por dispositivo.
- **[Acordado]** **Superficie de ataque:** el chip Bluetooth lleva firmware cerrado y el kernel y BlueZ procesan datos que llegan por el aire. Se trata como el módem (16.4): controlador no confiable con IOMMU o aislamiento de interfaz, demonio en su dominio SELinux sin acceso al home y emparejamiento mediado por el broker.
- **[Acordado]** Perfiles Bluetooth al inicio: A2DP, HFP y HID; LE Audio cuando el hardware lo permita.

### 10.8 Suspensión y reanudación

Un teléfono que no duerme bien no es un teléfono; por eso la suspensión es un criterio de aceptación por dispositivo (11.7).

- **[Acordado]** Suspensión del kernel (suspend-to-idle o PSCI, según el hardware) con fuentes de despertar explícitas: módem (llamadas, SMS y datos), botón, RTC (alarmas), carga o USB, Wi-Fi (patrones de despertar) y la conexión del gateway push (10.2).
- **[Acordado]** Suspensión oportunista (`autosleep`) con *wakelocks*; el kernel ya ofrece ambos. El demonio de energía (10.6) ofrece a las apps, mediante el broker, un «mantener despierto» con permiso y cuota. `kernel.root` no interviene.
- **[Acordado]** Orden: congelar apps (freezer de cgroups, 9.2), luego drivers y luego suspender; al reanudar, a la inversa. Los drivers mal portados son la causa habitual de que un teléfono no duerma.
- **[Acordado]** El gateway push necesita una conexión de muy bajo consumo (keepalive largo y despertar por la red o por el módem); hay que medirlo.
- **[Acordado]** El consumo en reposo se mide en **porcentaje de batería por hora** con la pantalla apagada y el módem registrado, y el umbral se fija midiendo el dispositivo con su sistema de fábrica: no peor que ese sistema. Una cifra de 10 a 20 mW parece demasiado optimista para un teléfono con el módem registrado, hasta donde sé; conviene medirlo antes de prometerla.

### 10.9 Cortafuegos: Frontera FireWall

- **[Acordado]** CosmicOS lleva un cortafuegos propio escrito en Rust, llamado **Frontera FireWall** (nombre confirmado por el autor, 22 de octubre de 2026). Por ser un módulo de red, su licencia es AGPL-3.0 (11.6).
- **[Propuesto]** **Qué es:** no filtra paquetes por sí mismo. Es un demonio (`cfronterad`) que traduce las reglas de red de cada app (lo que declara el manifiesto, limitado por los ajustes del usuario) a reglas del kernel (nftables, eBPF asociado al cgroup de cada app) y las mantiene. Filtra el kernel; Frontera es la política. Así no parsea tráfico y el TCB sigue pequeño.
- **[Propuesto]** **Funciones iniciales:** (1) «sin red» sigue siendo un namespace de red vacío (4) y Frontera coordina el resto; (2) interruptores por app para datos móviles y Wi-Fi; (3) entrada bloqueada por defecto: nada escucha desde fuera salvo lo declarado; (4) registro de las conexiones de cada app en los logs (10.4).
- **[Propuesto]** **Regla de oro:** el usuario puede restringir más que el manifiesto, nunca ampliar.
- **[Propuesto]** **Lo que no hace al inicio:** filtrado por dominio (frágil por las CDN y el DNS), inspección profunda de paquetes y VPN.
- **[Propuesto]** **Primer hito:** escribir un modelo mínimo de reglas y probarlo con dos apps aisladas, una con red permitida y otra sin ella, antes de ampliar funciones. D-28 no se discute hasta que ese hito funcione.
- **[Abierto]** [D-28](#d-28): Alcance de Frontera FireWall (después del primer hito).

### 10.10 Ajustes del dispositivo

- **[Propuesto]** **Tres tipos de ajustes, tres sitios.** (1) **Ajustes de cada app:** dentro de su zona (`~/.var/capps/<app>/config`, sección 5), en el formato que la app quiera (por ejemplo, `QSettings`); no se comparten. (2) **Ajustes del sistema** (idioma, tema, brillo, red…): los guarda un servicio de Configuración (`csettingsd`, nombre propuesto) en `~/.var/daemons/csettingsd/`, con un **esquema tipado** que declara cada componente (clave, tipo, valor por defecto y quién puede leerla o escribirla); los cambios van por D-Bus con una señal de cambio. (3) **Preferencias que las apps necesitan leer** (idioma, esquema claro u oscuro, texto grande, reducción de movimiento): solo lectura, a través del broker y con la interfaz Settings de xdg-desktop-portal (`org.freedesktop.appearance`), para que las apps GTK y Qt existentes la entiendan sin cambios, hasta donde sé.
- **[Propuesto]** Las apps nunca escriben ajustes del sistema ni ven los de otras apps. No hay una base de datos de ajustes compartida: dconf, hasta donde sé, es una base única y exigiría un filtro por app; el esquema propio da el mismo resultado con permisos por clave.
- **[Propuesto]** Cada ajuste del sistema lleva una marca: **portable** (idioma, tema…) o **del dispositivo** (brillo, calibraciones). Los portables entran en la copia de seguridad (5.3) y se restauran también en otro modelo.
- **[Abierto]** [D-32](#d-32): Formato del almacén de ajustes del sistema.

### 10.11 Cuentas y sincronización

- **[Propuesto]** **No hay cuenta del proyecto** (sin identificadores, sección 19). Cada servicio externo es una «cuenta» que el usuario añade.
- **[Propuesto]** **Protocolos abiertos, nada propio:** contactos por CardDAV, calendario y tareas por CalDAV, notas y archivos por WebDAV y mensajes por Matrix. Funcionan con un servidor autoalojado (por ejemplo, Nextcloud o Radicale) o con un proveedor.
- **[Propuesto]** Contactos y calendario viven en proveedores del sistema que las apps leen por el broker con permiso; solo las apps esenciales de contactos y calendario hablan esos protocolos.
- **[Propuesto]** **Secretos y credenciales** (tokens, contraseñas): un almacén de secretos del sistema, cifrado con la clave del usuario y aislado por app. Para que las bibliotecas existentes funcionen, expone la API Secret Service de D-Bus (que usa libsecret, hasta donde sé), filtrada por app en el proxy de D-Bus (7.2). El inicio de sesión OAuth va por el navegador del sistema (11).
- **[Propuesto]** Las direcciones `@cmail.dev` de 11.10 son para quienes contribuyen al proyecto, no cuentas de los usuarios del sistema.
- **[Propuesto]** Hay una ambigüedad en 10.2: el «GMS propio» incluye «sincronización de cuenta». Con lo anterior, debería entenderse como una sincronización opcional de ajustes y copias, nunca como una cuenta central.
- **[Abierto]** [D-33](#d-33): Qué significa «sincronización de cuenta» en el GMS propio (10.2).

### 10.12 Periféricos

- **[Propuesto]** **El soporte lo marca el kernel.** Como los módulos se cierran tras el arranque (3), la lista de drivers integrados o cargados en el arranque es la lista de periféricos soportados; un periférico que necesite otro driver no funciona, y las excepciones se deciden con una propuesta pública.

| Categoría | Cómo se soporta (hasta donde sé) | Nota |
|---|---|---|
| Auriculares y audio USB-C | Clase de audio USB estándar, con PipeWire | Los dispositivos compatibles con la clase funcionan sin driver propio |
| Teclados y ratones | USB HID y Bluetooth HID | El teclado externo es necesario para la accesibilidad (18.5) |
| Mandos de juego | Drivers del kernel para mandos de Xbox, PlayStation y Nintendo, y HID genérico | Sin lista cerrada: depende de los drivers integrados |
| Impresoras | Impresión sin controladores (IPP) | Sin drivers propietarios |
| Almacenamiento USB | `usb-storage`/UAS con FAT, exFAT y ext4 | Restringido con la pantalla bloqueada (16.2) |
| Cámaras web | UVC | |
| Pantallas externas | DisplayPort por USB-C, según el hardware | El modo escritorio es posterior (13) |

- **[Propuesto]** **Lista de periféricos probados**, mantenida por la comunidad en la documentación y organizada por dispositivo y modelo, con informes de los usuarios; sustituye a una lista cerrada.
- **[Abierto]** [D-35](#d-35): Lista inicial de drivers integrados.

---

## 11. Compatibilidad y ecosistema

- **[Acordado]** Se asume el coste de **no tener APKs ni GMS**.
- **[Acordado]** Aplicaciones esenciales iniciales (lista corta): cámara, galería, Signal/Matrix, entre las imprescindibles.
- **[Acordado]** Sin apps bancarias ni Play Integrity desde el día 1.
- **[Acordado]** No habrá protocolo de pago propio: un banco solo adopta lo certificado por EMVCo y un adquirente con licencia. Se soportan los estándares existentes (línea siguiente).
- **[Acordado]** NFC: para tarjetas, tags y datáfonos hay que soportar los estándares (ISO 14443/EMV): `libnfc` en escritorio para experimentar y el subsistema NFC del kernel en el teléfono. Pagos mucho más adelante (requieren [elemento seguro](#g-se)).
- **[Acordado]** Navegador (versión inicial, sustituida por la decisión de abajo): candidato **Helium** (fork de ungoogled-chromium con Manifest V2 y uBlock Origin) por su compatibilidad (WebAuthn/biometría). Hay que vigilar su ritmo de parches tras cada release de Chromium y diseñar una excepción de sandbox explícita para el navegador. Gecko (el motor de Firefox) queda como alternativa.
- **[Acordado]** Aclaración sobre Helium y Titanium, según lo consultado: Helium de escritorio está en beta (v0.16.5), con dos mantenedores y builds para Windows, macOS y Linux. **Titanium** es una app Android que se publicó como «Helium Browser» y se renombró en julio de 2026 para evitar confusión: parte del navegador de GrapheneOS, añade soporte de extensiones MV2 y es experimental; la fuente la describe como un proyecto **distinto** del Helium de escritorio, no como su versión móvil. Como CosmicOS no es Android, ningún build Android sirve: hace falta un Chromium para Linux/Wayland con parches propios. Dependencia de riesgo: tener a Gecko como plan B.
- **[Acordado]** Otras bases Chromium (20 de octubre de 2026): **ungoogled-chromium** es la base de Helium y la candidata más madura. **Thorium** se centra en rendimiento con optimizaciones de compilación que, hasta donde sé, apuntan sobre todo a x86 y no aplican a ARM64 ni RISC-V; en lo consultado solo vi builds ARM64 para Android, y el repositorio de Android indica que el mantenedor principal anda corto de tiempo y que un colaborador cubrió la última actualización. En cualquier fork pesan el ritmo de parches de seguridad y **MV2**: Chrome 139 eliminó su soporte (y la Chrome Web Store retiró los últimos MV2 el 31 de agosto de 2026), así que uBlock Origin completo depende de que el fork lo mantenga con parches propios. Sugerencia: ungoogled-chromium con parches y ritmo propios; Thorium solo como fuente de ideas.
- **[Acordado]** Navegador: **ungoogled-chromium** (20 de octubre de 2026), con parches y ritmo de actualización propios; Gecko queda como plan B.
- **[Acordado]** **WebView** como **componente del sistema** (opción B, 20 de octubre de 2026). **Objetivo: un único motor web** para el navegador y el WebView, construido desde el árbol de ungoogled-chromium, con renderizador en procesos y dominio SELinux propios y actualizado aparte de las apps. **Transitoriamente** (etapa 1): Qt WebEngine empaquetado como componente del sistema, con su propia cadencia de parches. Es obligatorio en un sistema móvil desde el principio. Se descartan Qt WebEngine incrustado en cada app (cada una arrastraría su propio Chromium y su propio ritmo de parches) y no ofrecer WebView.
- **[Acordado]** **Riesgo de la etapa 1:** hay dos linajes de Chromium que parchear (ungoogled-chromium para el navegador y el Chromium de Qt para el WebView), con lo que se duplican la superficie de ataque y el trabajo de parches hasta llegar a la etapa 2.
- **[Acordado]** **Qué hay que construir.** Chromium no es una API de incrustación estable: el WebView de Android existe porque Google mantiene código propio de integración dentro del árbol de Chromium, y ese trabajo, aquí, sería de CosmicOS en cada actualización de Chromium. Por eso se propone ir por etapas: primero, **Qt WebEngine empaquetado como componente del sistema** (una sola copia, actualizada aparte de las apps y con los parches de privacidad de CosmicOS); más adelante, una integración propia sobre el árbol de ungoogled-chromium, detrás de la misma API para que las apps no cambien.
- **[Acordado]** **Seguridad: aislados entre sí y de las demás apps, dentro del límite de la app anfitriona.** Los procesos del WebView (renderizador, GPU y red) los crea el launcher de esa app y heredan su namespace de red, su Landlock y su dominio SELinux: pueden ser más estrictos que la app (por ejemplo, un subdominio sin acceso a sus archivos), pero nunca más laxos. Así el WebView no puede darle a una app permisos que no tiene, y el permiso de red de la app se aplica solo a su WebView. El sandbox interno de Chromium (namespaces y seccomp) lo sustituye o complementa el del launcher; por comprobar.
- **[Acordado]** **Apps GTK4 y de otros toolkits:** una app que no usa Qt no puede ni debe enlazar Qt WebEngine. **Ninguna app enlaza el motor:** las apps hablan con el componente por IPC mediante un cliente fino con ABI en C, neutral al toolkit, con adaptadores para Qt y GTK.
- **[Acordado]** **API neutral desde el primer día**, estable y versionada con la ABI (7.1). Eso permite cambiar Qt WebEngine por la integración propia sin tocar a nadie. Como cada app tiene su propio socket de Wayland (9.2), el componente no puede acoplar una subsuperficie a la ventana de otra app: entrega los fotogramas como buffers compartidos (dmabuf) que la app presenta en su superficie, y devuelve entrada, portapapeles y accesibilidad por IPC. Si Qt WebView admite un backend propio, las apps Qt lo usarían sin cambios; si no, un adaptador propio (por comprobar).
- **[Acordado]** El inicio de sesión (OAuth) se hace en el navegador del sistema, no en un WebView.
- **[Acordado]** [PWAs](#g-pwa) como ciudadanos de primera clase en SPS para cubrir servicios web.
- **[Acordado]** Cuentas de Google por protocolos abiertos: OAuth/OIDC, IMAP/SMTP con OAuth, CalDAV, CardDAV.
- **[Acordado]** Matrix primero (`matrix-rust-sdk`); Signal después, por la hostilidad hacia clientes de terceros. WhatsApp mediante WhatsApp Web como PWA, con un asterisco: históricamente exigía un teléfono vinculado, así que puede no servir a quien solo tenga CosmicOS (por verificar).
- **[Acordado]** Ninguna capa Android al inicio; Waydroid aislado solo si algún día hace falta.

### 11.1 Códecs y patentes

Un [códec](#g-codec) comprime y descomprime audio o vídeo. Que el código sea abierto no libra de patentes: la licencia del programa y las patentes del algoritmo son cosas independientes.

- **El kernel Linux casi no trae códecs.** Viven en bibliotecas de espacio de usuario (FFmpeg, libopus, dav1d, GStreamer). El kernel ofrece, como mucho, interfaces para decodificadores por hardware (V4L2).
- **Estado de las patentes, según lo consultado:** H.264 tiene un pool activo (Via LA, con tarifas de streaming reestructuradas en 2026) y para HEVC hay un pool activo (Access Advance) más titulares fuera de él; no se encontró fecha de caducidad confirmada para ninguno. Las patentes de MPEG-4 Parte 2 (DivX, Xvid) caducaron por completo el 19 de julio de 2026. Las de MP3 caducaron en 2017, hasta donde sé. AAC no pude verificarlo.
- **Quién paga en la práctica:** quien distribuye el software o el chip. El decodificador por hardware de un SoC suele venir licenciado por el fabricante del chip, hasta donde sé; el riesgo se concentra en distribuir decodificadores por software. Esto no es asesoría legal.
- **[Acordado]** [Opus](#g-opus) como códec de audio por defecto.
- **[Acordado]** Base sin regalías: Opus, Vorbis, FLAC, MP3, AV1 y VP9 (publicados como libres de regalías, aunque algunos pools ajenos a la alianza que los publicó reclaman patentes), con contenedores Ogg, Matroska y WebM.
- **[Acordado]** H.264, AAC y HEVC quedan fuera del núcleo: no se distribuyen por ahora: el paquete opcional que el usuario instalaría bajo su responsabilidad queda aplazado hasta tener asesoría legal; cuando exista, preferencia por decodificar con el hardware vía V4L2.
- **[Acordado]** **Solo códecs libres** en el sistema (Opus, AV1, VP9…). El paquete opcional con H.264, AAC y HEVC queda aplazado hasta tener asesoría legal (ver 11.6). ([D-09](#d-09), 22 de octubre de 2026)

### 11.2 DRM (Widevine)

El [DRM](#g-drm) es contenido cifrado más una licencia por dispositivo. El dominante es Widevine, de Google, que tiene dos niveles: **L3**, por software, y **L1**, con el descifrado dentro de un entorno de ejecución seguro (TEE), lo que exige aprovisionamiento ligado al fabricante y certificación. Los servicios de streaming suelen reservar la calidad alta para L1, hasta donde sé.

- Sin L1 no hay streaming en HD; es un hueco de ecosistema comparable al de las apps bancarias.
- El módulo de descifrado de Widevine es un binario cerrado de Google: ofrecer L3 chocaría con el principio de «sin blobs».
- **[Acordado]** **Sin DRM:** ni Widevine L3 ni L1. ([D-10](#d-10), 22 de octubre de 2026)

### 11.3 VoLTE / IMS

En 4G y 5G la red solo transporta datos; la voz es una llamada de datos que pasa por el subsistema IMS del operador, y a eso se le llama [VoLTE](#g-volte). Sin VoLTE el teléfono cae al 2G/3G, que muchos operadores están apagando.

- Cada operador configura IMS a su manera, y algunos solo permiten dispositivos que han certificado (por IMEI o por TAC). Por eso un teléfono con firmware correcto puede quedarse sin llamadas.
- En muchos teléfonos el cliente IMS es una pila cerrada del fabricante que no se puede integrar en un sistema alternativo.
- **[Acordado]** Es un bloqueo regulatorio-comercial, no solo de hardware: al elegir el dispositivo objetivo, comprobar que VoLTE funciona con los operadores locales antes de comprometerse con él. Las llamadas por Matrix o Signal sirven de respaldo, pero no sustituyen al teléfono.
- **[Abierto]** [D-11](#d-11): Calendario de apagado de 2G/3G; hay datos de la búsqueda del 22 de octubre de 2026 en la tabla 14.1.

### 11.4 Fabricante, bootloader y atestación

Para instalar CosmicOS hay que poder cambiar el sistema del dispositivo, y en la mayoría de los teléfonos eso exige desbloquear el bootloader, lo que borra los datos y suele romper la [atestación](#g-atestacion).

- **El enfoque de GrapheneOS** (a seguir como modelo): el bootloader se desbloquea, se instala el sistema, se inscribe la clave pública de ese sistema y se vuelve a **bloquear**. El arranque verificado ([AVB](#g-avb)) sigue activo, pero con la clave del proyecto, y la atestación por hardware informa de esa clave. Según su página, GrapheneOS lo soporta en la serie Pixel 6 a 10 y su app Auditor comprueba la clave, el bootloader bloqueado y los *downgrades*. Hasta donde sé, es posible porque Google permite la clave propia en los Pixel; la mayoría de fabricantes no.
- **Quién firma qué:** el fabricante controla la raíz del bootloader (grabada en el chip); Google controla la cadena de certificados de atestación de Android; el sistema alternativo aporta su propia clave. Lo que decide si un dispositivo sirve es si el fabricante permite volver a bloquearlo con esa clave.
- **[Acordado]** **Capa de compatibilidad con U-Boot** (20 de octubre de 2026): para los dispositivos cuyo bootloader no acepta el formato de CosmicOS (por ejemplo, un Pixel, que arranca imágenes de Android firmadas con AVB y no una UKI por UEFI), el bootloader del fabricante carga una imagen mínima con U-Boot y este arranca la UKI por UEFI.
- **[Acordado]** Detalles: la imagen de U-Boot va firmada con la clave que el usuario inscribe en el bootloader del fabricante (como en GrapheneOS) y U-Boot verifica a su vez la firma de la UKI (arranque seguro UEFI con claves de CosmicOS), de modo que la cadena no se rompe. La atestación del fabricante solo cubre hasta U-Boot, no la UKI, así que hace falta atestación propia y U-Boot pasa a formar parte del TCB. U-Boot es GPL-2.0 o posterior (11.6).
- **[Acordado]** La API de atestación por hardware es de Android (KeyMint). CosmicOS necesita su propia ruta para exponerla, y los bancos tendrían que aceptar su clave en una lista de permitidos.
- **[Acordado]** Seguir de cerca a [UnifiedAttestation](#g-unifiedattestation): iniciativa abierta (Apache 2.0) respaldada por Volla, Murena, iodé y Apostrophy, en busca de una fundación (Eclipse), sin versión publicada según lo consultado. Inventar una atestación propia no tiene sentido sin cooperación del fabricante.
- **[Acordado]** **Regla de dispositivos soportados:** un dispositivo cuyo bootloader no se pueda desbloquear ni volver a bloquear con una clave del usuario no se soporta, aunque el hardware sea bueno. Se distingue entre «bloqueado pero re-bloqueable con tu clave» (válido, como GrapheneOS) y «bloqueado para siempre por el fabricante» (excluido). La licencia no puede garantizar esto, porque el bootloader del fabricante no es código de CosmicOS: lo garantiza esta regla.
- **[Acordado]** **Sin socios fabricantes por ahora:** el usuario desbloquea el bootloader y lo vuelve a bloquear con la clave de CosmicOS. Se hablará con fabricantes cuando haya un prototipo. ([D-12](#d-12), 22 de octubre de 2026)

### 11.5 Sideloading y repositorios de terceros

- **[Acordado]** Los **repositorios de terceros** ([sideloading](#g-sideloading)) quedan aprobados (20 de octubre de 2026).
- **[Acordado]** Se habilitan desde el modo **Desarrollador básico** (sección 12), sin necesidad del avanzado y, por tanto, sin perder las apps `<bancario>`.
- **[Acordado]** Condiciones: cada repositorio de terceros se firma con una clave que el usuario inscribe expresamente (verificando la huella), con el aviso de que queda fuera de las garantías del repositorio central (revisión por PR y compilaciones reproducibles). Sus apps corren con el mismo sandbox, el manifiesto sigue siendo solo una solicitud, y nunca reciben las etiquetas sensibles (`<bancario>`, `accessibility`, `system`), reservadas al repositorio central. Configuración las marca como de terceros.

### 11.6 Licencia, gobernanza y relaciones

- **[Acordado]** Licencias (20 de octubre de 2026): **[LGPL-3.0](#g-agpl)** para el SDK, **AGPL-3.0** para los módulos de red y todo lo que toque internet o hardware de red, y **GPL-3.0** para el resto del código propio. **GPL-2.0 solo donde sea inevitable.** Nada de licencias permisivas (Apache, MIT, BSD) en código propio.
- **[Acordado]** **GPL-2.0 queda donde es obligatoria, y eso incluye el kernel:** el kernel Linux es GPL-2.0-only, con miles de autores, y no se puede relicenciar a GPL-3.0 ni a AGPL. Lo mismo vale para lo que entre en su árbol (incluidos los drivers en Rust y los de red) y para los módulos que se carguen en él. El kernel no puede ir en GPL-3.0; lo que sí se puede es que el **resto** de CosmicOS sea GPL-3.0. En el bootloader manda la licencia del proyecto de origen (U-Boot es GPL-2.0 o posterior, así que puede distribuirse como GPL-3.0). GPL-3.0 y GPL-2.0-only no se combinan en un mismo binario, pero como procesos separados no hay problema. La GPL-2.0 en espacio de usuario solo se acepta si una dependencia la impone.
- **[Acordado]** **Qué añade la AGPL.** Solo se nota sobre la GPL-3.0 cuando un usuario interactúa con el programa por red: gateway push, «GMS propio» y repositorio. Un demonio cliente de red en AGPL se comporta como GPL-3.0; no hace daño, pero el efecto real está en los servicios.
- **[Acordado]** **LGPL y arranque bloqueado.** LGPLv3 y GPLv3 exigen que el usuario pueda instalar versiones modificadas en dispositivos de consumo (cláusula anti-tivoización). Es compatible con el modo Desarrollador y con claves inscritas por el usuario (11.4 y 12); no lo sería en un dispositivo de un fabricante aliado que se envíe bloqueado sin esa vía. Además, el SDK debe enlazarse de forma dinámica.
- **[Acordado]** **Qt.** Qt cubre la mayoría de los módulos con LGPL-3, pero, hasta donde sé, algunos son solo GPL-3 o comerciales (por ejemplo, Qt Charts, Qt Virtual Keyboard y Qt Wayland Compositor); no lo pude verificar. Aceptado: los módulos solo GPL-3 pueden usarse en los componentes propios de CosmicOS (GPL-3.0), pero no entran en el SDK LGPL que usan las apps de terceros. Revisar módulo por módulo.
- **[Acordado]** Qt WebEngine está bajo LGPL-3.0 (no es de los módulos solo GPL), así que no contamina ni al componente del sistema ni a las apps; coherente con el SDK LGPL. Por confirmar en la revisión módulo por módulo: la página de módulos de Qt no indica licencias.
- **[Acordado]** **Revisión módulo por módulo** ([D-23](#d-23), 22 de octubre de 2026): antes de usar un módulo de Qt se revisa su licencia, y el SDK (LGPL) no incluye módulos solo GPL. Main y Space admiten cualquier licencia aprobada por la OSI; el repositorio muestra la licencia de cada app en sus metadatos, sin añadir campos al `Manifest.json` (6.3).
- **[Acordado]** La regla aplica al **código propio**. Las **dependencias** con licencias permisivas (Smithay es MIT, y gran parte del ecosistema Rust) son compatibles con un proyecto GPL-3.0; vetarlas dejaría fuera casi todo.
- **[Acordado]** Licencia del SDK: **LGPL**, de modo que las apps que lo enlacen dinámicamente no tienen que ser GPL.
- **[Acordado]** **Main y Space solo aceptan código abierto;** las apps propietarias van al repositorio MTL (8.1) o a repositorios de terceros (11.5). ([D-13](#d-13), 22 de octubre de 2026)
- **[Acordado]** **Gobernanza mínima** (adoptada el 20 de octubre de 2026):
  1. **Decisiones:** al principio decide el autor, pero todo cambio que toque el esquema del manifiesto, la ABI, la política de firmas o las licencias pasa por una propuesta pública (RFC) con plazo de comentarios.
  2. **Contribuciones:** firma de origen con [DCO](#g-dco) (`Signed-off-by`) y sin CLA: nadie, tampoco el autor, puede relicenciar el código de otros, lo que protege el copyleft.
  3. **Claves:** las claves raíz del repositorio y de las OTA usan umbral (por ejemplo, 3 de 5 mantenedores con YubiKeys y metadatos TUF con *threshold*) y nunca dependen de una sola persona (8.2).
  4. **Continuidad:** infraestructura, dominios y repositorios a nombre de una organización, no de una cuenta personal, todo documentado y reproducible para que otro pueda levantar un espejo; si el autor queda inactivo seis meses, los mantenedores con clave asumen la dirección.
  5. **Dinero y seguridad:** cuentas públicas de financiación e infraestructura, política de divulgación coordinada de vulnerabilidades (`SECURITY.md`, 90 días) y código de conducta.
- **[Aplazado]** [D-14](#d-14): Forma jurídica, marca y primeros mantenedores. Sin forma jurídica por ahora; fundación y asesoría legal más adelante.
- **[Acordado]** **Soberanía total, con una alianza parcial con [postmarketOS](#g-postmarketos)** para ganar experiencia con hardware que usa blobs. UnifiedAttestation se sigue de cerca (11.4), sin alianza formal. ([D-15](#d-15), 22 de octubre de 2026)
- **[Acordado]** Multiusuario: no es un objetivo inicial; el diseño no lo impide (rutas por usuario, sin `/home/<nombre>` fijo). La copia de seguridad se esboza en 5.3.
- **[Acordado]** Estrategia de adopción: primer borrador en 11.8.

### 11.7 Criterios del dispositivo objetivo

- **[Acordado]** Un dispositivo solo se soporta si cumple todo esto:
  1. Bootloader desbloqueable y re-bloqueable con una clave del usuario (principio 8, 11.4).
  2. VoLTE funcional con los operadores locales (11.3).
  3. Suspensión funcional y consumo en reposo aceptable (10.8).
  4. GPU con driver abierto upstream (9.3).
  5. Módem con IOMMU utilizable (16.4).
  6. Drivers upstream para el resto del hardware (ver el riesgo de hardware en 15).
- **[Acordado]** Familias de dispositivos candidatas al inicio: **Pixel y Motorola** (por ser fáciles de desbloquear y tener el mayor ecosistema de ROMs personalizadas) y **los teléfonos centrados en el usuario, como Fairphone** (21 de octubre de 2026). Todas se evalúan con los mismos criterios de arriba.
- **[Aplazado]** [D-16](#d-16): Modelo concreto de dispositivo objetivo. A la espera de confirmación y de dinero para probar en hardware real; orden tentativo: Motorola, después Pixel y Fairphone solo si se consigue a buen precio de importación.

#### Matriz de dispositivos (borrador del 21 de octubre de 2026)

Leyenda: **V** = verificado en una fuente consultada; **?** = no verificado (por rellenar con pruebas o con la documentación del proyecto correspondiente). Donde hay una frase en lugar de V o ?, es lo que sé sin haberlo comprobado hoy.

| Criterio | Pixel (Tensor: 6 a 10) | Motorola con Snapdragon | Motorola con MediaTek |
|---|---|---|---|
| 1. Desbloquear | V: sí, con el interruptor estándar de Android | V: con la web de Motorola; generalmente desde la serie G3x en adelante; la serie E y varios G de fabricantes ODM (G13, G23, G24) no tienen vía oficial; hay una cláusula sobre vender el equipo desbloqueado | V: igual vía, pero el estado de desbloqueo se comprueba contra RPMB y los equipos nuevos desactivan BROM por eFuse |
| 1. Re-bloquear con clave propia | Hasta donde sé, sí: soportan clave AVB personalizada (la usan GrapheneOS y similares); no lo verifiqué hoy | **Riesgo alto.** V: en algunos modelos Motorola no permite re-bloquear y puede dejar el equipo inservible. Hay trabajo en curso de GrapheneOS para algunos modelos recientes (Edge, Razr), no confirmado oficialmente | ? (misma política de Motorola; RPMB y eFuse agravan el riesgo) |
| 6. Soporte upstream del SoC | V: GS101 (Pixel 6) entró en Linux 6.8 solo para arrancar hasta un initramfs; Tensor G5 (Pixel 10) entra en 7.4 con soporte básico y archivos de árbol de dispositivo iniciales; el estado de pantalla, GPU y módem no consta | V: la familia está muy avanzada (por ejemplo, Snapdragon 8 Gen 3 arranca con pantalla desde Linux 6.8 y hay una charla en FOSDEM 2026 sobre su estado); lo concreto de GPU, suspensión, audio y cámara: ? | ? (generalmente menos avanzado que Qualcomm; no verificado) |
| 4. GPU abierta | ? Mali en Pixel 6 a 9 (Panfrost o Panthor según el modelo) y PowerVR en el Pixel 10 | Adreno con freedreno y Turnip en Mesa: sí, hasta donde sé; ? por modelo | ? (Mali o PowerVR según el chip) |
| 5. Módem con IOMMU | ? (módem Samsung Exynos) | ? (módem integrado en el SoC; acceso por remoteproc y QRTR) | ? |
| 2. VoLTE en Colombia | ? | ? Motorola es muy común en el mercado colombiano, lo que facilita comprobar con los operadores (Claro, Movistar, Tigo, WOM) | ? |
| 3. Suspensión y reposo | ? | ? | ? |

**Fairphone 5 (teléfonos centrados en el usuario)**

| Criterio | Fairphone 5 (Snapdragon QCM6490) |
|---|---|
| 1. Desbloquear y re-bloquear con clave propia | Desbloqueo: ? (la política es abierta, según tengo entendido). Re-bloqueo con clave AVB propia: **?**; la guía comunitaria de bootloader que consulté lo marca como pendiente de completar, así que no lo doy por hecho |
| 6. Soporte upstream | V: el propio fabricante envió al kernel principal el soporte inicial del Fairphone 5 (árbol de dispositivo, serie de parches de septiembre de 2023). Qué funciona hoy (pantalla, GPU, módem, cámara): ? |
| Vida útil del soporte | V: el fabricante anunció 8 años de actualizaciones de software (cnx-software, 2023); el sistema oficial es Android, con la dependencia del firmware del fabricante que eso implica (17.7) |
| 2. VoLTE en Colombia, 3. Suspensión, 4. GPU, 5. IOMMU del módem | ? |
| Disponibilidad en Colombia | ? (hasta donde sé, no se vende oficialmente en América Latina; habría que importarlo) |

**Cómo se entra a la matriz:** todo teléfono que se proponga (Fairphone, PinePhone, Volla, etc.) se evalúa con los seis criterios y la regla práctica de arriba; el que no complete la fila de verificaciones no se compra para el proyecto.

**Lectura del borrador:**
- **No hay candidato limpio hoy.** Pixel cumple el criterio 1 (re-bloqueo con clave propia) pero tiene el soporte upstream más flojo; Motorola con Snapdragon tiene el mejor soporte upstream de la tabla, pero el re-bloqueo con clave propia es el eslabón débil. Es justo el conflicto entre los criterios 1 y 6 señalado arriba.
- **Qué falta para decidir:** (1) comprobar, por modelo concreto de Motorola, si re-bloquea con una clave AVB propia en un equipo de prueba; (2) revisar la página del dispositivo en postmarketOS y el estado de su kernel principal; (3) elegir modelos de gama media-alta de los últimos 2 o 3 años para tener soporte del SoC.
- **Teléfonos centrados en el usuario (Fairphone y similares):** ver la tabla siguiente; son una familia objetivo y no solo una plataforma de desarrollo.
- **Regla práctica:** ningún equipo se compra para el proyecto sin haber comprobado antes el re-bloqueo con clave propia en esa variante y esa región, porque un error aquí puede dejar el equipo inservible.

### 11.8 Estrategia de adopción

Sin apps no hay usuarios, y sin usuarios no hay apps. Borrador de cómo romper el círculo:

- **[Acordado]** **Web primero:** las PWAs (11) con el WebView del sistema cubren los servicios web sin esperar a las apps nativas.
- **[Acordado]** **Apps esenciales propias:** teléfono y SMS, cámara, galería, mensajería, archivos, ajustes y navegador. Sin ellas el teléfono no es usable; marcan el primer hito móvil (13).
- **[Acordado]** **Reutilizar el ecosistema Linux móvil:** existen muchas apps adaptativas en GTK4/libadwaita y Qt/Kirigami que se pueden recompilar como `.sps` desde su fuente. Un **importador** que convierta los permisos del manifiesto de Flatpak (`finish-args`) en un borrador del `Manifest.json` ahorraría mucho trabajo; el borrador sigue siendo una solicitud que se revisa por PR (8.1), y la revisión humana es el cuello de botella.
- **[Acordado]** **Aliados:** clientes de Matrix y de Signal, y proyectos como postmarketOS, Phosh y Plasma Mobile, para apps y hardware.
- **[Acordado]** **Público inicial** (desarrolladores y entusiastas de Linux): ellos portan sus propias apps, así que un SDK, una documentación y un `sps --create-package` sencillos son el producto durante esta fase.
- **[Acordado]** **Criterio de «uso diario»:** cinco personas del equipo usan CosmicOS como teléfono principal durante un mes (llamadas, SMS, datos móviles, cámara, mensajería y mapas). ([D-17](#d-17), 22 de octubre de 2026)

### 11.9 Documentación para desarrolladores

- **[Propuesto]** **Documentación como código:** en Markdown, dentro del repositorio, versionada junto con la ABI (7.1) y en español e inglés (9.5).
- **[Propuesto]** **Contenido inicial:** (1) referencia del SDK (generada con rustdoc y con la documentación de Qt); (2) guía de empaquetado con `sps --create-package` y del `Manifest.json`; (3) tutorial «tu primera app» con Qt/QML y con Rust; (4) ejemplos mínimos por clase de app; (5) guía de seguridad (qué puede y qué no puede hacer una app dentro del sandbox, y cómo pedir permisos); (6) guía del entorno de prueba con QEMU ARM64 (13).
- **[Propuesto]** Un ejemplo que no compila en el CI no se publica.
- **[Abierto]** [D-31](#d-31): Plataformas de documentación y comunidad.

### 11.10 Comunidad

- **[Propuesto]** **Chat principal:** Matrix (abierto y coherente con el proyecto), con espacios por tema.
- **[Acordado]** **Las decisiones importantes nunca se toman en Discord:** se toman en Matrix y por correo (decisión del autor, 24 de octubre de 2026).
- **[Propuesto]** **Discord solo como puerta de entrada** ([D-36](#d-36)): el autor quiere llegar a una comunidad más grande y acepta que es propietario y poco seguro. Reglas: sin decisiones, sin informes de seguridad (esos van a `SECURITY.md`, 11.6), sin claves ni datos privados, un mensaje fijado que lleve a Matrix, al foro y a las listas, moderadores desde el primer día y un resumen de lo útil en el foro buscable, porque lo dicho en Discord no queda archivado de forma pública. Hasta donde sé existen puentes entre Matrix y Discord; no verifiqué cuál convendría.
- **[Propuesto]** **Correo del proyecto** ([D-37](#d-37)): direcciones `<usuario>@cmail.dev` para quienes desarrollan el sistema o contribuyen de forma ocasional, a la manera de las direcciones del kernel. Condiciones: (1) empezar con **alias de reenvío**, no con buzones, para no cargar con el alojamiento ni la entregabilidad del correo; (2) **nunca obligatorio** para contribuir (el `Signed-off-by` admite cualquier correo real, 11.6); (3) dominio y servidores a nombre de la organización, no de una persona (11.6); (4) claves PGP publicadas bajo el dominio con WKD (hasta donde sé, permite descubrir claves por dominio), en línea con una identidad sin cuenta de GitHub (8.2); (5) **no es una cuenta del sistema ni de los usuarios** (10.11). No verifiqué que el dominio `cmail.dev` esté disponible ni quién lo posee.
- **[Propuesto]** **«Interna» no debería significar privada:** las decisiones técnicas van en listas de correo públicas con archivo abierto, por el principio 6 (abierto y auditable) y porque, hasta donde sé, así lo hace el kernel. Lo privado queda para los incidentes de seguridad (8.6) y los asuntos personales.
- **[Propuesto]** **Foro libre** para lo que deba quedar buscable, y reuniones públicas periódicas (propongo mensuales) con acta publicada.
- **[Propuesto]** **Guía de contribución** y etiquetas de «buen primer issue»; el código de conducta ya está acordado (11.6).
- **[Propuesto]** **Transparencia:** las decisiones viven en el registro de la sección 14.
- **[Abierto]** [D-31](#d-31): Plataformas de documentación y comunidad.

### 11.11 Modelo de financiación

- **[Propuesto]** **Principio:** ni telemetría, ni publicidad, ni datos de los usuarios como fuente de ingresos (sección 19). Depender de una sola vía sería un riesgo para la continuidad (11.6).
- **[Propuesto]** **Vías posibles:** (1) donaciones recurrentes; (2) subvenciones y fondos para software libre (por ejemplo, NLnet, que hasta donde sé financia proyectos de software libre; no verifiqué sus condiciones); (3) servicios de pago opcionales: una cuota para las empresas que publican en MTL (por la verificación de identidad y la revisión de manifiestos), soporte para empresas y un gateway con límites más altos; (4) acuerdos con fabricantes para dispositivos con CosmicOS, más adelante (11.4); (5) una fundación como receptora del dinero (D-14).
- **[Propuesto]** **Gateway push:** el software es AGPL y autoalojable (10.2). El proyecto ofrece una instancia por defecto con límites generosos pero finitos, sostenida con las vías anteriores; cualquiera puede apuntar el dispositivo a otro gateway desde Configuración. El coste por mensaje debería ser bajo (es una señal de despertar), pero no lo he medido.
- **[Abierto]** [D-34](#d-34): Financiación y gateway push por defecto.

### 11.12 Herramientas USB: `cosmic-fastboot`, `cad` y `cosmic-usb`

- **[Acordado]** **Reparto** (7 de noviembre de 2026): `cosmic-usb` es solo transporte USB; `cosmic-fastboot` es el cliente del bootloader; `cad` (CosmicOS Advanced Driver, el equivalente de ADB) es el cliente del teléfono ya arrancado; `cosmic-flash` es la capa alta que conoce GPT, UKI y OTA y usa el protocolo a través de `cosmic-usb`.
- **[Acordado]** `cosmic-usb` usa `rusb` (libusb). El USB se esconde tras una interfaz pequeña, para poder cambiar a `nusb` (Rust puro) sin tocar los clientes.
- **[Acordado]** `cosmic-fastboot` (con `cfastboot` como alias opcional) es **parcialmente compatible con el fastboot de Android**: los comandos estándar (`getvar`, `flash`, `erase`, `reboot`, `flashing unlock` y `flashing lock`) se mantienen tal cual, y las extensiones van aparte: `--oem-unlock` (sirve para cualquier bootloader con camino de desbloqueo; si hace falta un código, lo pide), `--oem-get-data` y `--oem-lock`.
- **[Acordado]** `--oem-lock` pide el código de compilación exacto de la versión de CosmicOS instalada e inyecta las claves en el bootloader.
- **[Propuesto]** Solo se usan los caminos de desbloqueo que ofrece el propio fabricante (por ejemplo, en Motorola, el código que se obtiene en su web); sin exploits. Si el bootloader no tiene camino, el cliente lo dice. Antes de bloquear, comprueba que la clave de arranque está instalada y se niega si no, porque bloquear con una imagen no reconocida deja el teléfono inservible. Desbloquear borra los datos (11.4).
- **[Acordado]** **Limitaciones de `cad`:** desactivado por defecto y solo en modo Desarrollador (12); autenticación por clave; sin acceso por red (nada de ADB por TCP) al inicio; solo con el dispositivo desbloqueado por el usuario; demonio propio con nombre `c<nombre>d` y su propio dominio SELinux.
- **[Acordado]** Aplazados (7.7): no entran en el primer hito. Hasta tener un teléfono físico (D-16), `android-tools` sirve para flashear Motorola y Pixel.
- **[Abierto]** [D-42](#d-42): nombre del demonio de `cad`, reparto exacto con `cosmic-flash`, soporte de imágenes dispersas (5.6) y soporte de Windows y macOS en el cliente.

---

## 12. Usuarios y modos de dispositivo

- **[Acordado]** Al crear el usuario se elige entre **sudo (desarrollador)** o **usuario normal**.
- **[Acordado]** **El home lleva el nombre de usuario que la persona elige** en el asistente de primer uso (`/home/<usuario>`), a diferencia del «home raro» de los teléfonos Android (7 de noviembre de 2026).
- **[Propuesto]** **Nombre de usuario:** las reglas de Linux (minúsculas, números, `_` y `-`, empezando por letra, máximo 32 caracteres) y sin nombres reservados (`root`, usuarios del sistema). Aparte hay un **nombre para mostrar** libre (con tildes, mayúsculas y emojis), que es el que ve la interfaz.
- **[Propuesto]** **Privacidad frente a las apps:** dentro del sandbox, `$HOME` apunta a la carpeta de la propia app (`~/.var/capps/<app>`), no al home real, para que el nombre de usuario (a veces el nombre real) no se filtre a cada app (19).
- **[Propuesto]** **Rutas:** nada de rutas absolutas con el nombre en el código, la configuración ni los manifiestos; siempre `$HOME` y las variables XDG, como pide 12.1.
- **[Abierto]** [D-45](#d-45): el sistema base es de solo lectura, así que `passwd` y `shadow` deben vivir en la zona de datos o en una capa escribible. Falta decidir dónde y cómo se leen durante el arranque.
- **[Acordado]** El sudo no puede afectar a las apps con la etiqueta `<bancario>` o derivadas.
- **[Acordado]** Implementarlo con SELinux: el dominio de root no tiene permiso para `ptrace`, `/proc/<pid>/mem` ni para escribir en el directorio de esas apps.
- **[Acordado]** Modelarlo como **tres modos de dispositivo**, porque el dueño siempre puede reflashear y el banco confía en un estado verificado, no en una promesa:
  - **Estándar:** bootloader bloqueado, política sellada, admin con sudo acotado; las apps `<bancario>` pueden ejecutarse.
  - **Desarrollador básico:** se activa tocando siete veces el número de compilación en Configuración (como en Android) y es el primer paso hacia el modo avanzado. Muestra el botón de **desbloqueo OEM**, habilita los repositorios de terceros (11.5), permite abrir una **terminal local sin privilegios de root**, **instalar `.sps` desde un archivo** y **ver los logs `json`** ([D-24](#d-24)); las apps `<bancario>` siguen funcionando y los logs `raw` siguen inaccesibles.
  - **Desarrollador avanzado:** el usuario tiene casi todos los permisos; se activa de forma explícita y queda en el estado de arranque; las apps `<bancario>` se niegan a ejecutarse, avisando al usuario.
- **[Acordado]** Arranque verificado + atestación propia para sostener esa garantía.
- **[Acordado]** El modo Estándar solo es alcanzable en dispositivos cuyo bootloader pueda volver a bloquearse con la clave de CosmicOS o que un fabricante aliado envíe ya con CosmicOS y bloqueado (ver 11.4).
- **[Acordado]** En modo Desarrollador avanzado, `kernel.root` deja de ser inmortal para root (ver 3.1). Las menciones anteriores a «modo Desarrollador» sin más se refieren a este.
- **[Acordado]** El modo básico es un interruptor en tiempo de ejecución (cambia la configuración de SPS y de Configuración, no la política SELinux ni `kernel.root`) y se desactiva con otro toque. El avanzado sí cambia la política: como está sellada, se elige **al arrancar** (variante sellada, registrada en el estado de arranque) y no se activa en caliente.
- **[Acordado]** El botón de desbloqueo OEM solo habilita la posibilidad: el desbloqueo real exige la credencial de bloqueo, avisa de que se borrarán los datos y ofrece antes la copia externa (5.3); tras el formateo se ofrece restaurarla. Debe cumplir el principio 8.

### 12.1 Multiusuario: primeras ideas

- **[Acordado]** No es un objetivo inicial, pero el diseño no cierra la puerta: rutas por usuario (`~/` según el usuario, nunca `/home/<nombre>` fijo), cada usuario con su UID, su clave fscrypt derivada de su credencial y su propio `~/.var`.
- **[Acordado]** Las apps de `/usr/apps` se comparten en solo lectura y sus datos son por usuario; los servicios con estado (notificaciones, teclado) corren por usuario cuando haga falta.
- **[Acordado]** La primera forma razonable es un **modo invitado efímero** (datos en tmpfs, borrados al salir); los perfiles de trabajo, más adelante.

---

## 13. Hoja de ruta propuesta

1. **Launcher en Rust** *(acordado como primer paso)*: lee el manifiesto → namespaces + Landlock + seccomp → `exec`. Se prueba con una app que intente escapar (leer `~/.ssh`, abrir un socket).
2. Esquema de `Manifest.json` y sistema de firmas.
3. Repositorio mínimo (GitHub Actions + cosign + índice TUF estático).
4. Primeras apps nativas y demonios (notificaciones, logging, broker de permisos y WebView del sistema) en un entorno de desarrollo sobre **Atomic Linux de escritorio** (con emulación ARM64) y con pruebas en el móvil objetivo.
5. PhanteonWM y CreationDE sobre ese mismo entorno.
6. Poda del kernel y hardening *(acordado como paso posterior al launcher)*.
7. Plataforma objetivo desde el principio: un móvil ARM64 con buen soporte upstream, con pruebas en hardware real en paralelo al desarrollo (por ejemplo, Pixel o PinePhone); elegirlo también por VoLTE funcional y por poder volver a bloquear el bootloader (11.3 y 11.4).

**Estrategia de alcance**

- **[Acordado]** **Móvil primero, escritorio después** (decisión del autor, 21 de octubre de 2026). Con la restricción a ARM64 y RISC-V, un escritorio exigiría portátiles ARM, que son caros y escasos; los teléfonos son más fáciles de conseguir. El escritorio de Atomic Linux es el entorno de desarrollo y de pruebas (con emulación o placas ARM), no el producto.
- **[Acordado]** Para que «móvil primero» no dependa de que todos los muros externos caigan a la vez (VoLTE, bootloader, atestación, DRM), el primer hito en teléfono se define con criterios de salida: usable a diario por el autor, llamadas y SMS con VoLTE, mensajería (Matrix y Signal), actualizaciones A/B con rollback y copia de seguridad. Las apps bancarias y el streaming protegido quedan fuera del primer hito.
- **[Propuesto]** **Llamadas de emergencia como criterio de salida** del primer hito en teléfono (1.1): no se recomienda usar un dispositivo como teléfono principal sin tenerlas verificadas con el operador; mientras no lo estén, el sistema muestra un aviso permanente y la documentación lo dice.
- **[Propuesto]** **Plataformas de desarrollo antes del teléfono:** el modelo de seguridad (launcher, `kernel.root`, CinitD, SPS, Frontera) se prueba antes en QEMU ARM64 y en Atomic Linux de escritorio (paso 4) y, si hace falta, en una placa con buen soporte del kernel principal, sin esperar al teléfono. No cambia la decisión de móvil primero.
- **[Propuesto]** **RISC-V:** hasta donde sé no hay teléfonos RISC-V con GPU, módem, cámara y batería bien soportados. Se mantiene como arquitectura admitida, sin dispositivo móvil objetivo; se usa en QEMU y en placas.

### 13.1 Estado de implementación

A fecha del 7 de noviembre de 2026 el autor tiene un árbol local, sin publicar (8.7), con un crate o módulo por cada componente y los defconfig del kernel para arm64 y riscv64. **Andamiaje** significa que existe la estructura; no sé cuánto código funcional hay dentro de cada uno, y quien lo sepa debe corregir esta tabla.

| Componente | Estado | Primer hito comprobable | Paso |
|---|---|---|---|
| CinitD | Andamiaje | Arranca en QEMU, lanza `kernel.root` y recoge huérfanos | 1 (apoyo) |
| `kernel.root` | Andamiaje | Supervisa y reinicia un servicio de prueba en QEMU | 1 (apoyo) |
| Launcher en Rust | Andamiaje | Una app que intenta escapar del sandbox falla y queda registrada (7.3) | 1 |
| Orquestador en Go | Andamiaje | Lee un manifiesto validado y llama al launcher | 1 y 2 |
| SPS y `.sps` | Andamiaje | Empaqueta e instala una app de ejemplo | 2 |
| Repositorio y TUF | Andamiaje | Índice estático firmado | 3 |
| Frontera FireWall | Andamiaje | Modelo mínimo de reglas con dos apps aisladas, una con red y otra sin ella | 4 |
| Demonios (`clogd`, `cnotifd`, `cpushd`, `cbrokerd`, `cboard`) | Andamiaje | Cada uno funciona en el entorno de desarrollo | 4 |
| WebView del sistema | Solo diseño | Etapa 1 con Qt WebEngine | 4 |
| PhanteonWM y CreationDE | Andamiaje | Sesión mínima sobre Atomic Linux de escritorio | 5 |
| `cosmic-splash` | Andamiaje | Cede la pantalla al compositor sin parpadeo (5.4) | 5 |
| Kernel podado | Compila (fork con `arch/` purgado; arm64 hasta `Image.gz` el 7 de noviembre; riscv64 sin probar) | Configuración endurecida que arranca | 6 |
| Sistema de construcción (`cosmic-build`) | Funciona, pero los binarios de Rust y Go aún se compilan para el PC y no para ARM64 (8.8) | Compilación cruzada y reproducible en contenedor | 1 (apoyo) |
| Instalación y asistente de primer uso | Solo diseño (5.6) | Flashear el bundle en un teléfono y crear el usuario | 7 |
| `cosmic-fastboot`, `cad`, `cosmic-usb` | Andamiaje (C y Rust) | Flashear un teléfono real (11.12) | 7 |
| Dispositivo objetivo | Sin elegir | Primera imagen en hardware real | 7 |

**Por dónde empezar:** el paso 1 (el launcher) y, en paralelo, CinitD y `kernel.root` en QEMU, que no necesitan hardware ni dinero.

---

## 14. Registro de decisiones

Las decisiones pendientes viven aquí, no en el texto: la 14.1 es lo que sigue vivo (abierto o aplazado) y la 14.2 lo ya cerrado.

### 14.1 Vivas

| ID | Tema | Dónde | Estado | Detalle y sugerencia |
|---|---|---|---|---|
| <a id="d-11"></a>D-11 | Calendario de apagado de 2G/3G | 11.3 | Abierto | Búsqueda del 22 de octubre de 2026. **Colombia:** Tigo apagó el 2G en noviembre de 2022 y Claro estaba completando el suyo en febrero de 2023 (prensa de 2022 y 2023); un listado de cierres (thingsdata, octubre de 2026) da el apagado del **3G de Tigo Colombia el 1 de diciembre de 2027**, como fecha planificada y sujeta a cambios; para Claro y Movistar no encontré fecha publicada y no consulté WOM. No pude leer el documento de la CRC sobre modernización de redes. **Mundo:** en Europa está casi terminado (algunos operadores con fechas hasta 2028), en EE. UU. terminó (AT&T, T-Mobile y Verizon), en Asia-Pacífico está mayormente apagado, Canadá cierra el 1 de marzo de 2027 (Bell y Telus) y en Latinoamérica la mayoría de los operadores no tiene fecha. **Consecuencia:** con el 2G ya apagado en Colombia, la voz depende del 3G o de VoLTE; con el 3G de Tigo planificado para finales de 2027, VoLTE es obligatorio para el dispositivo objetivo. Falta confirmar con Claro, Movistar y WOM. |
| <a id="d-14"></a>D-14 | Forma jurídica, marca y primeros mantenedores | 11.6 | Aplazado | Sin forma jurídica por ahora; una fundación y asesoría legal más adelante. Los mantenedores se eligen cuando haya código que mantener. La misma consulta legal debería cubrir la marca (por la cercanía con COSMIC, de System76) y los códecs (D-09). |
| <a id="d-16"></a>D-16 | Modelo concreto de dispositivo objetivo | 11.7 | Aplazado | Hace falta confirmación y dinero para probar en hardware real. Orden tentativo: Motorola primero (más barato), Pixel después y Fairphone solo si se consigue a buen precio de importación. Antes de comprar: comprobar el re-bloqueo con clave propia en esa variante y esa región, y preferir un modelo Snapdragon (matriz en 11.7). |
| <a id="d-25"></a>D-25 | Estrategia de prototipado de CinitD | 7.4 | Abierto | **Esto no cambia la arquitectura final** (CinitD propio, acordado). Pregunta: para el primer prototipo, A) escribir CinitD ya (su alcance son pocas funciones y nada se acopla a otro init) o B) usar un init mínimo existente y sustituirlo después. Mi voto y el de las revisiones externas: A. Búsqueda del 22 de octubre de 2026: no encontré proyectos llamados `cinit` (solo salió `dumb-init`, que es otro), pero fue una búsqueda corta; antes de publicar el nombre, repetirla a fondo, incluido el registro de paquetes de Rust. Falta la respuesta del autor. |
| <a id="d-26"></a>D-26 | Cómo funciona el repositorio MTL | 8.1 | Abierto | Propuesta de partida: MTL es un índice del proyecto que enlaza repositorios que cada empresa aloja en su propio servidor y firma con su propia clave, inscrita tras verificar su identidad. El proyecto revisa los permisos del manifiesto (no el código), firma el índice y muestra las apps como «sin verificar»; las apps corren con el mismo sandbox. La etiqueta `<bancario>` solo se asigna tras revisar el manifiesto y verificar la identidad del emisor. **Quién responde:** la propuesta de la revisión externa, que adopto, es un acuerdo de responsabilidad y transparencia en el que la empresa asume la responsabilidad legal de su binario y el proyecto firma solo el índice, no el binario, de modo que no avala el contenido. Redactar ese acuerdo es trabajo para un abogado (no soy uno), así que depende de D-14. |
| <a id="d-28"></a>D-28 | Alcance de Frontera FireWall | 10.9 | Abierto | Qué hace además de lo básico: filtrado por dominio opcional con un DNS propio, aviso al usuario en la primera conexión de cada app, VPN integrada. Mi voto: nada de eso al inicio, y no discutirlo hasta superar el primer hito de 10.9 (dos apps aisladas, una con red y otra sin ella). |
| <a id="d-29"></a>D-29 | Pentesting externo y bug bounty | 7.6 | Abierto | Contratar un equipo, abrir un programa de recompensas o ambos; depende de la financiación. Mi voto: recompensa solo en reconocimiento público hasta tener dinero, y el pentest externo antes de la v1.0. |
| <a id="d-30"></a>D-30 | Canal de contacto y página de estado para incidentes | 8.6 | Abierto | Dónde se avisa a los usuarios y por dónde se reportan incidentes: dirección de correo y clave PGP en `SECURITY.md`, y una página de estado alojada fuera de la infraestructura principal. |
| <a id="d-31"></a>D-31 | Plataformas de documentación y comunidad | 11.9 | Abierto | Dónde se alojan las docs y el foro. Propuesta: Matrix para el chat (con Discord solo como puerta, D-36), un foro libre para lo que deba quedar buscable y listas de correo públicas para las decisiones técnicas. |
| <a id="d-32"></a>D-32 | Formato del almacén de ajustes del sistema | 10.10 | Abierto | Opción A: archivos de texto tipo TOML (legibles, auditables y fáciles de copiar en la copia de seguridad). Opción B: dconf/GVariant, por compatibilidad con GNOME. Mi voto: A. |
| <a id="d-33"></a>D-33 | «Sincronización de cuenta» del GMS propio | 10.11 | Abierto | 10.2 lo menciona y choca con «no hay cuenta central». Opción A: quitarlo. Opción B: sincronización opcional y autoalojada de ajustes y copias. Mi voto: B. |
| <a id="d-34"></a>D-34 | Financiación y gateway push por defecto | 11.11 | Abierto | Quién paga el gateway y de dónde sale el dinero del proyecto. Propuesta en 11.11: varias vías, sin telemetría ni publicidad, e instancia por defecto con límites, más cuota para empresas de MTL. Depende en parte de D-14. |
| <a id="d-35"></a>D-35 | Lista inicial de drivers integrados | 10.12 | Abierto | Qué periféricos se soportan al inicio, dado que los módulos se cierran tras el arranque. Se decide al podar el kernel (paso 6) y según el dispositivo elegido. |
| <a id="d-36"></a>D-36 | Discord como puerta de entrada | 11.10 | Abierto | Opción A: no tener servidor de Discord. Opción B: tenerlo como puerta, con reglas (sin decisiones ni informes de seguridad, enlace a Matrix y al foro, moderadores) y, tal vez, un puente con Matrix. Mi voto: B, como pide el autor; el riesgo es que la comunidad se fragmente, y se contiene resumiendo en el foro. |
| <a id="d-37"></a>D-37 | Correo del proyecto `@cmail.dev` | 11.10 | Abierto | Opción A: alias de reenvío. Opción B: buzones propios. Mi voto: A al inicio. Pendiente: confirmar que el dominio está disponible y registrarlo a nombre de la organización (11.6); definir quién puede tener dirección (¿también contribuidores ocasionales?) y decidir si las listas de correo de decisiones son públicas (mi propuesta). |
| <a id="d-42"></a>D-42 | Herramientas USB: demonio de `cad` y reparto con `cosmic-flash` | 11.12 | Abierto | Reparto y limitaciones de `cad` acordados. Falta el nombre del demonio, la frontera exacta con `cosmic-flash`, el soporte de imágenes dispersas y si el cliente soporta Windows y macOS. |
| <a id="d-45"></a>D-45 | Dónde viven `passwd` y `shadow` | 12 | Abierto | El sistema base es de solo lectura. Opciones: A) en la zona de datos, creados por el asistente; B) en una capa escribible superpuesta. Mi voto: A, con el arranque capaz de funcionar antes de que exista ningún usuario. |
| <a id="d-43"></a>D-43 | Parámetros fijos y compilador del kernel | 3 | Abierto | (a) ¿Los parámetros fijos viven solo en el `defconfig` (`CONFIG_CMDLINE`) o también en código fuente? (b) ¿Variante con línea de comandos libre en Quasar y EventHorizon, o fija en todos los canales? (c) GCC o LLVM: kCFI, la pila de llamadas sombra y LTO dependen de Clang. Mi voto: (a) `defconfig`, (b) variante libre solo en los canales de pruebas, (c) LLVM si se quieren esas defensas. |
| <a id="d-39"></a>D-39 | Tareas del PID 1 sin dueño | 7.4 | Abierto | Propuesta refinada en 7.4 (tabla de reparto entre `cinit-early`, CinitD, `kernel.root`, `cdevd` y `cpowerd`, con `cdevd` sin poder cargar módulos y el apagado en dos mitades). Falta la confirmación del autor. |
| <a id="d-40"></a>D-40 | Documentar lo que ya existe en el árbol | 8.7 | Abierto | `libs/cosmic-core` (con `ipc`, `manifest` y `service`), `tools/cosmic-flash` y `tools/cosmic-repo` no constan en la especificación. Falta saber qué hace cada uno y cómo se relaciona con SPS, con el flasheo de dispositivos (11.4) y con el repositorio (8.1). |

### 14.2 Cerradas

| ID | Tema | Decisión | Dónde |
|---|---|---|---|
| <a id="d-01"></a>D-01 | Aislamiento entre apps | Landlock dinámico con SELinux debajo | 4 |
| <a id="d-02"></a>D-02 | Copia externa | Incremental y cifrada; sin partición propia para instantáneas | 5.3 |
| <a id="d-03"></a>D-03 | Localizar, bloquear o borrar a distancia | Opcional, autoalojado, desactivado por defecto, orden firmada | 5.5 |
| <a id="d-04"></a>D-04 | Go en el TCB | Go sin privilegios; lo privilegiado, en Rust | 7 |
| <a id="d-05"></a>D-05 | Infraestructura de CI | Runners ARM64 alojados más QEMU; servidor propio cuando haya financiación | 7.3 |
| <a id="d-06"></a>D-06 | Apps de terceros y reproducibilidad | Se muestran como «sin verificar» | 8.4 |
| <a id="d-07"></a>D-07 | `repo` y colaboradores ocasionales | `repo` para el sistema; `git clone` simple para SDK y apps | 8.5 |
| <a id="d-08"></a>D-08 | Idiomas | Español e inglés hasta la v1.0 | 9.5 |
| <a id="d-09"></a>D-09 | Códecs con patentes | Solo códecs libres; paquete opcional aplazado hasta tener asesoría legal | 11.1 |
| <a id="d-10"></a>D-10 | DRM | Ninguno | 11.2 |
| <a id="d-12"></a>D-12 | Fabricantes y atestación | Sin socios por ahora | 11.4 |
| <a id="d-13"></a>D-13 | Apps propietarias | Main y Space solo código abierto; propietarias en MTL o en terceros | 8.1 |
| <a id="d-15"></a>D-15 | Alianzas | Soberanía total con alianza parcial con postmarketOS | 11.6 |
| <a id="d-17"></a>D-17 | Métrica de éxito | Uso diario: cinco personas, un mes | 11.8 |
| <a id="d-18"></a>D-18 | Licencias de firmware | Copiarlo de las particiones del fabricante del propio teléfono donde no se pueda redistribuir | 17.7 |
| <a id="d-19"></a>D-19 | Motor de voz | eSpeak NG; Piper, tal vez después | 18.4 |
| <a id="d-20"></a>D-20 | Accesibilidad fuera de la pantalla | Lector de pantalla primero; luego teclado externo e interruptores | 18.5 |
| <a id="d-21"></a>D-21 | Ramas LTS y backports | Tres posiciones congeladas; la cuarta cuenta parches; un bot asigna los números | 20.4 |
| <a id="d-22"></a>D-22 | BPF firmado | Solo el sistema, con firma de CosmicOS; sin firma solo en modo avanzado | 3 |
| <a id="d-23"></a>D-23 | Licencias de módulos y del repositorio | Revisión módulo por módulo; cualquier licencia OSI en Main y Space | 11.6 |
| <a id="d-24"></a>D-24 | Modo Desarrollador básico | Añade terminal sin root, instalar `.sps` desde archivo y ver logs `json` | 12 |
| <a id="d-27"></a>D-27 | Convención y nombres de los componentes | Demonios `c<nombre>d`, herramientas `cosmic-`; tabla de 7.5 confirmada; `cosmic-splash` se queda como está | 7.5 |
| <a id="d-38"></a>D-38 | «Servidor de hardware» tipo Halium | No por ahora; solo se reactiva si se cumplen las tres condiciones de 16.6 | 16.6 |
| <a id="d-41"></a>D-41 | Sistema de construcción común | Bash y Ninja ahora, GN cuando crezca, sin motor propio y sin Python fuera del contenedor | 8.8 |
| <a id="d-44"></a>D-44 | Instalación y primer arranque | Opción A en el primer dispositivo: flasheo por partición desde el PC (bundle como imagen única) y asistente de primer uso; B y C para más adelante | 5.6 |
| — | `kernel.root`: ventana, `init`, umbral y canal | Lista cerrada de operaciones; `init` solo recoge huérfanos y vigila; aislamiento al quinto fallo con espera creciente; ningún canal para las apps | 3.1 |
| — | Modelo de compilación del repositorio | Main y Space compilan siempre | 8.1 |
| — | Paquetes huérfanos | Instalables con aviso visible | 8.3 |
| — | Obtención del código fuente | `repo` con manifiesto XML (`cosmic-manifest`) | 8.5 |
| — | Capa Android | Ninguna al inicio | 11 |
| — | Campos finos del `Manifest.json` | Definitivos tal como están en 6.3 | 6.3 |
| — | Nombre del tmpfs | `volatile` | 5 |
| — | Carpetas de logs | `~/.var/logs/<app>/{json,raw}`, en minúsculas | 10.4 |
| — | Almacén de apps | Partición propia montada en `/usr/apps` | 5.1 |
| — | FS de datos | F2FS en móvil, Btrfs en escritorio | 5.1 |
| — | Copias de seguridad | Instantánea en la partición de datos más copia externa cifrada | 5.3 |
| — | Pantalla de arranque | `cosmic-splash` propio; audio opcional por dispositivo | 5.4 |
| — | Plazo máximo para actualizar el sistema | Decide siempre el usuario, con recordatorios persistentes | 17.4 |
| — | Compromiso de parcheo | Plazos por fases: 30 y 90 días al inicio; 7, 30 y 90 después | 17.6 |
| — | Vida útil del soporte | 2 años Nebula y 5 Singularity, limitados por kernel y fabricante | 17.6 |
| — | Lector de pantalla | Propio en Rust sobre [AT-SPI2](#g-atspi), empezando por un producto mínimo | 18.4 |
| — | Contadores de descargas | Agregados y públicos, sin identificadores | 19.4 |
| — | Mapas y búsquedas | Mapas sin conexión; búsquedas opcionales por proxy propio | 19.4 |
| — | Consumo en reposo | Porcentaje de batería por hora, medido en el dispositivo | 10.8 |
| — | Perfiles Bluetooth | A2DP, HFP y HID; LE Audio después | 10.7 |
| — | GPU soportadas | Adreno y Mali | 9.3 |
| — | Ruta de promoción entre canales | EventHorizon → Quasar → Pulsar → Nebula → Singularity | 20.4 |
| — | Primera implementación del WebView | Qt WebEngine primero, con dos linajes de Chromium durante la etapa 1 | 11 |
| — | Estrategia de adopción | Borrador acordado en 11.8 | 11.8 |
| — | Familias de dispositivos | Pixel, Motorola y teléfonos centrados en el usuario, como Fairphone | 11.7 |
| — | Nombre del init | CinitD | 7.4 |

---

## 15. Riesgos principales

- **Hardware:** con drivers upstream y sin blobs (salvo el módem), solo habrá soporte para dispositivos bien soportados por el kernel principal; el módem sigue siendo un blob cerrado.
- **Ecosistema:** sin ART ni APKs, parte de cero; las apps bancarias, push y pagos son el muro principal.
- **Navegador y WebView:** son las piezas con más superficie de ataque; su ritmo de parches pesa más que sus funciones. Mientras dure la etapa 1 del WebView (Qt WebEngine) hay dos linajes de Chromium que parchear, lo que duplica esa superficie; el motor único es el objetivo (sección 11).
- **TCB:** el launcher, el parser de logs, `cdevd` y `kernel.root` son piezas privilegiadas, y el demonio en Go, aunque sin privilegios, orquesta a los demás ([D-04](#d-04)); deben ser pequeñas, auditadas y con fuzzing. `kernel.root` es la más crítica: es inmortal para root y gestiona todo, por lo que debe ser un supervisor mínimo (sección 3.1).
- **Confianza del manifiesto:** el análisis en la máquina del desarrollador no es una garantía; la verificación real debe ocurrir en el repositorio y en el dispositivo.
- **Claves:** una clave de desarrollador comprometida o perdida; mitigada con YubiKey, llave de respaldo, rotación y revisión por PR.
- **Alcance:** es un proyecto enorme; priorizar lo diferencial (launcher, SPS, repositorio) antes del móvil completo.
- **Operadores (VoLTE/IMS):** sin VoLTE certificado no hay llamadas donde se apaguen el 2G y el 3G; es un bloqueo regulatorio-comercial (11.3).
- **Fabricantes y atestación:** el modo Estándar con bootloader bloqueado depende de que un fabricante lo permita o lo envíe de serie; conseguir un fabricante aliado es un requisito de producto, no un detalle de implementación (11.4).
- **Contenido protegido y códecs:** sin DRM L1 no hay streaming en HD, y los códecs con patentes tienen costes y riesgos legales (11.1 y 11.2).
- **Gobernanza, licencia y adopción:** financiación, continuidad del proyecto, licencia del SDK y el problema de las apps iniciales siguen abiertos (11.6).

---

## 16. Modelo de amenaza

Este modelo define de qué se protege CosmicOS y de qué no. Se revisa en cada versión mayor del documento.

### 16.1 Qué se protege

- **[Acordado]** Datos del usuario (mensajes, fotos, contactos), credenciales y claves (`~/.ssh`, claves de firma), privacidad de los sensores (micrófono, cámara, ubicación), integridad del sistema (base RO, política, arranque) y la cadena de suministro del repositorio.

### 16.2 Adversarios en alcance

| Estado | Adversario | Qué intenta | Defensas principales |
|---|---|---|---|
| **[Acordado]** | **Apps maliciosas** | Leer datos de otras apps, escalar privilegios, espiar (teclado, pantalla), persistir | Sandbox por app, escritura solo en `~/.var/capps/<app>`, broker de permisos, teclado aislado, manifiesto como solicitud (secciones 4, 5 y 10.3) |
| **[Acordado]** | **Procesos corruptos** (bugs explotados, servicios legítimos comprometidos) | Pasar de un proceso comprometido a todo el sistema | Divergencia desde `kernel.root`, SELinux sellado, seccomp y Landlock, reinicio con backoff o aislamiento, cuarentena (3.1 y 8.3) |
| **[Acordado]** | **Desarrolladores malintencionados** | Manifiestos engañosos, código malicioso (incluido en `build.rs`), bombas de tiempo, claves robadas o cuentas múltiples | Compilación en sandbox y revisión por PR en el repositorio, permisos mínimos, bloqueo de actualizaciones que añaden permisos, YubiKey y reputación de claves, firmas separadas de desarrollador y repositorio, [criogenia](#g-criogenia) (secciones 6 y 8) |
| **[Acordado]** | **Módem con blob** | Atacar desde el firmware cerrado del módem | Excepción asumida; ver 16.4 |
| **[Acordado]** | **Atacante de red** | Interceptar o alterar descargas y notificaciones | Firmas y TUF, TLS, cifrado de extremo a extremo, sin identificadores estables (sección 19) |
| **[Acordado]** | **Cadena de suministro** | Comprometer el repositorio, una dependencia o el servidor de actualizaciones | Compilaciones reproducibles, TUF, registro de transparencia, claves separadas, firma de OTA con índice anti-rollback (secciones 8 y 17) |
| **[Acordado]** | **Acceso físico breve** (teléfono perdido o robado) | Extraer datos o instalar algo en el arranque | Arranque verificado, cifrado en reposo (fscrypt), bloqueo de pantalla, lockdown del kernel, USB restringido con la pantalla bloqueada |
| **[Acordado]** | **Contenido web hostil** | Explotar el navegador | Sandbox del navegador, actualizaciones rápidas, PWAs aisladas (sección 11) |
| **[Acordado]** | **Periféricos y radios** (USB, Bluetooth, NFC, Wi-Fi) | Enviar entrada malformada a drivers del kernel | Drivers nuevos en Rust, IOMMU, USB y NFC restringidos con la pantalla bloqueada |

### 16.3 Fuera de alcance (por ahora)

- **[Acordado]** Ataques dirigidos de estados con cadenas de exploits de día cero contra el kernel: el diseño los dificulta, pero no los contiene (límite asumido en la sección 3).
- **[Acordado]** Ataques físicos invasivos (extraer chips, glitching, JTAG) y canales laterales de CPU más allá de las mitigaciones del kernel.
- **[Acordado]** El propietario del dispositivo como adversario: en modo Desarrollador el dueño puede hacer lo que quiera; las apps `<bancario>` confían en el estado de arranque verificado, no en que el usuario no pueda (sección 12).
- **[Acordado]** Otros firmwares cerrados (Wi-Fi, bootrom): se tratan como el módem (16.4) donde sea posible, pero no se pueden verificar. La GPU (9.3) y Bluetooth (10.7) tienen tratamiento propio.

### 16.4 Excepción: módem con blob

- **[Acordado]** Hay módems con firmware cerrado (blob). CosmicOS lo asume como **excepción explícita** al principio de "sin blobs", porque sin módem no hay teléfono.
- **[Acordado]** El módem se trata como un **dispositivo no confiable** que se supone comprometible:
  - **IOMMU obligatoria:** el módem solo accede por DMA a los buffers que se le asignan; un dispositivo sin IOMMU utilizable no se soporta.
  - **Interfaz mínima** (QMI/MBIM) tratada como entrada no confiable: el demonio de telefonía corre en su propio dominio SELinux, con los permisos mínimos y sin acceso al home.
  - **Modo avión real:** que corte la alimentación del módem cuando el hardware lo permita.
  - **Sin confianza en la red del operador:** las llamadas y los SMS no van cifrados de extremo a extremo por naturaleza; para confidencialidad, apps como Matrix o Signal.
  - **Firmware del módem** solo se actualiza dentro de paquetes firmados del dispositivo.
  - **Transparencia:** cada dispositivo soportado documenta qué blobs lleva ("lista de blobs").
- **[Acordado]** Límite aceptado: no se puede verificar qué hace ese firmware; se contiene, no se audita.

### 16.5 Supuestos de confianza

- **[Acordado]** Se confía en el hardware de arranque (bootrom y bootloader del dispositivo), en la clave raíz del repositorio y en el kernel compilado. `kernel.root` es de confianza total y por eso debe ser mínimo.

### 16.6 Frontera con el hardware del fabricante

- **[Propuesto]** **Regla:** ningún blob de fabricante se carga en el espacio de direcciones de una app. Vive en el kernel (como firmware) o en un demonio con dominio propio, y la app solo ve una API nativa. Generaliza lo dicho en 9.3 y en 16.4.

| Subsistema | Dónde vive el blob en Android (hasta donde sé) | En CosmicOS | ¿Entra en la app? |
|---|---|---|---|
| GPU | Driver de usuario enlazado dentro de la app | Mesa abierto (9.3); el firmware de la GPU lo carga el kernel | No, mientras el driver de usuario sea abierto; si fuera un blob, sí, y por eso ese dispositivo no se soporta |
| Cámara | Proveedor de cámara, un proceso del fabricante | Demonio propio, PipeWire y portal (10.5) | No |
| Módem | RIL del fabricante | ModemManager con dominio SELinux propio (16.4) | No |
| Audio | Firmware del DSP y configuración UCM | Driver del kernel y PipeWire (10.7) | No |
| Sensores | HAL del fabricante | IIO y un demonio (10.6) | No |
| Wi-Fi y Bluetooth | Firmware y driver | Kernel y BlueZ (10.7) | No |

- **[Propuesto]** **La GPU es el caso delicado.** Wayland aísla el acceso al dispositivo, pero para renderizar con aceleración la app enlaza el driver de usuario de la GPU y emite comandos desde su propio proceso. Si ese driver fuera un blob cerrado, entraría en cada app. Por eso la regla de GPU abierta (9.3) no es una preferencia: la arquitectura depende de ella. El firmware de la GPU sigue siendo un blob, pero vive en el kernel y consta en la lista de blobs (16.4).
- **[Propuesto]** **La regla traslada el blob, no lo borra.** Cumplir «ningún blob en la app» no implica «ningún blob en el sistema»: el firmware que queda (GPU, Wi-Fi, módem…) vive en el kernel o en su propio procesador y cuenta para el TCB (principio 2), aunque no se pueda auditar ni reescribir. Por eso el TCB real es mayor que el código propio, y la lista de blobs de cada dispositivo (16.4) forma parte de la seguridad.
- **[Propuesto]** **Qué se mata y qué se hereda.** Se descarta Android como plataforma de apps (ART, APK, GMS, framework Java). De Android se toman ideas (cómo usa SELinux y el sandbox por UID), no su código ni su kernel: SELinux es del kernel Linux y no de Android, y CosmicOS usa Linux upstream con su propio endurecimiento. Lo que sigue siendo del fabricante queda contenido: firmware y módem.
- **[Propuesto]** **Contrato de interfaz.** Cada subsistema de la tabla expone al sistema y a las apps una API nativa estable (D-Bus o socket, versionada con la ABI de 7.1), independiente de lo que haya detrás; se diseña antes de tocar hardware. Falta documentar cada API (1.1).
- **[Propuesto]** **Coste.** Lo que aporte el fabricante (módem, cámara, DSP) sigue gastando memoria y CPU aunque las apps sean nativas. La ganancia de eficiencia es de las apps, no necesariamente del sistema entero; se mide en el primer dispositivo (9.4) antes de prometer mejoras en teléfonos de gama baja. Las cifras de memoria que circulan no están verificadas.
- **[Propuesto]** **Mercado.** El filtro reduce los dispositivos candidatos (11.7); es el precio de la arquitectura.
- **[Acordado]** **Sin «servidor de hardware» tipo Halium por ahora** ([D-38](#d-38), 25 de octubre de 2026): se mantiene el criterio 6 de 11.7 (drivers upstream) y los dispositivos que lo necesiten no se soportan. **Condiciones para reactivarlo** (las tres a la vez, para que no vuelva por inercia):
  1. Ya hay un primer dispositivo objetivo con drivers upstream (9.3) y un subsistema crítico, típicamente el módem o la cámara, no tiene un camino upstream utilizable.
  2. El blob puede envolverse como servicio de hardware aislado que expone una API nativa (D-Bus, V4L2 o PipeWire) al resto del sistema, sin tocar procesos de apps y sin heredar framework Java, ART ni APK. Si el servicio expusiera interfaces HAL de Android (AIDL o HIDL) a las apps, rompería esta sección.
  3. Si el único camino es ejecutar Halium o Droidian completos, con su framework, la respuesta es no: el dispositivo no entra en soporte y se busca otro.

---

## 17. Modelo de actualización del sistema

Actualizar el sistema **no es lo mismo** que actualizar apps: las apps se actualizan desde la Store (SPS) y el sistema desde la app de **Configuración**, con claves, canales y garantías distintos.

### 17.1 Principios

- **[Acordado]** La actualización del sistema está **supervisada por el usuario**: nunca se aplica sin que el usuario lo decida.
- **[Acordado]** Se gestiona desde la **app de Configuración**, no desde la Store.
- **[Acordado]** Un proceso **descarga la nueva imagen** y solo la **aplica tras reiniciar**.

### 17.2 Flujo

**[Acordado]** Pasos del flujo de actualización:

| Paso | Qué ocurre |
|---|---|
| 1. Consulta | El dispositivo lee el índice firmado del canal (TUF), sin identificadores de dispositivo (sección 19). |
| 2. Aviso | El usuario ve versión, canal, tamaño, resumen de cambios (incluidos ABI, política y permisos del sistema) y la urgencia (seguridad crítica o normal). |
| 3. Descarga | En segundo plano, reanudable, por Wi-Fi por defecto; comprueba espacio libre y batería. |
| 4. Verificación | Firma del bundle, índice anti-rollback, hash del contenido (dm-verity) y compatibilidad con el hardware, antes de tocar nada. |
| 5. Instalación | Se escribe en el slot inactivo (A/B); el teléfono sigue usable y el sistema en uso no se modifica. |
| 6. Reinicio | El usuario elige cuándo: ahora o programado (por ejemplo, de noche con el cargador). |
| 7. Primer arranque a prueba | El slot nuevo arranca en estado "de prueba": se comprueba que `kernel.root`, la política, los demonios esenciales y la pantalla de bloqueo funcionan. |
| 8. Confirmación o rollback | Si todo va bien, se marca como bueno; si falla N veces, el bootloader vuelve solo al slot anterior. El usuario también puede revertir a mano desde Configuración mientras el slot anterior exista. |

### 17.3 Ideas adicionales

- **[Acordado]** **Actualizaciones delta** (diferencias entre imágenes EROFS) para descargar menos, con la imagen completa como alternativa.
- **[Acordado]** **Compatibilidad de apps:** la actualización declara su ABI; Configuración avisa de qué apps dejarían de funcionar y SPS las actualiza o las marca antes de reiniciar.
- **[Acordado]** **Recordatorios persistentes** para parches de seguridad críticos, sin forzar nada.
- **[Acordado]** **Modo recuperación:** si ningún slot arranca, un entorno mínimo en la ESP permite reflashear por USB con `cosmic-flash`, y debe ser accesible (sección 18).
- **[Acordado]** **Bootloader y UKI:** la UKI también se mantiene en doble copia (A/B), y el firmware de arranque se actualiza por separado y con más cuidado.
- **[Acordado]** **Historial:** cada actualización queda en el registro de auditoría (versión, firma, resultado y rollbacks).
- **[Acordado]** **Rotación de la clave raíz** de firma del sistema mediante TUF, igual que en el repositorio.
- **[Acordado]** **Sin actualizaciones silenciosas** y sin envío automático de resultados.

### 17.4 Puntos por decidir

- **[Acordado]** Decide siempre el usuario, con recordatorios persistentes para los parches críticos; no se fuerza nada.
- **[Acordado]** Cambiar de canal en cualquier dirección (por ejemplo, de Nebula a Singularity o a un canal de pruebas) está permitido y no cuenta como rollback; ver 17.5.

### 17.5 Anti-rollback sin fusibles

- **[Acordado]** Nada de eFUSE ni mecanismos irreversibles: el usuario debe poder bajar de versión (por compatibilidad) y cambiar de canal sin quedar atrapado. Aun así, un atacante no debe poder forzar una versión vieja y vulnerable.
- **[Acordado]** La diferencia entre ambos casos es **quién autoriza**: el usuario, presente, o un tercero. Diseño:
  1. **Índice de seguridad** monótono, asignado al firmar y **global a todos los canales**, sin depender del número de versión: cada release firmado, de cualquier canal, recibe el siguiente. Así pasar de Nebula a Singularity no es un rollback aunque el número de versión sea menor, y un LTS parcheado recibe un índice nuevo y alto.
  2. Cada release declara un **suelo de seguridad** (`security_floor`): el índice más bajo que sigue considerándose seguro. Los lanzamientos que corrigen fallos críticos suben el suelo.
  3. Por las rutas automáticas y remotas, el agente de actualización **nunca** instala por debajo del suelo.
  4. **Bajar del suelo** exige la **autorización local del usuario**, desde Configuración o desde recuperación: credencial de bloqueo, confirmación dibujada por el compositor (que ninguna app puede automatizar), aviso de los fallos conocidos de esa versión, imagen aún firmada y no marcada como **revocada** en los metadatos TUF.
  5. Los metadatos TUF aportan frescura (el `timestamp` de vida corta) y evitan que un servidor malicioso sirva un índice viejo. El suelo vive en una partición de estado que solo el agente de actualización puede escribir (política SELinux).
  6. **Datos:** si la versión destino no entiende el formato de datos actual, se restaura la instantánea previa o se restablecen los datos (5.1 y 5.3); nunca se migra hacia atrás en silencio.
- **[Acordado]** **Límite asumido:** sin contador de hardware, un atacante con acceso físico a la flash capaz de reescribir esa partición podría bajar el suelo. Ese caso cae en los ataques físicos invasivos, fuera de alcance (16.3); el acceso físico breve lo cubren el arranque verificado y el cifrado. Contra un atacante remoto, ninguna ruta automática baja de versión.
- **[Acordado]** Refuerzo opcional: guardar el suelo también en almacenamiento autenticado (RPMB) cuando el dispositivo lo ofrezca. No cambia el diseño, porque la autorización del usuario sigue pudiendo bajarlo.
- **[Acordado]** El mismo mecanismo se aplica a las apps: suelo de seguridad por app (8.3).

### 17.6 Compromiso de parcheo ([SLA](#g-sla))

Si un banco ha de confiar en el arranque verificado, hace falta poder responder con un número cuánto tarda un parche.

- **[Acordado]** Plazos objetivo (ver las fases más abajo), medidos desde que el arreglo es público (upstream) y no desde que se descubre el fallo:
  - **Crítico** (ejecución remota, escape del sandbox, salto del arranque verificado, o fallo del kernel, del navegador o del WebView con exploit conocido): hasta **7 días**.
  - **Alto:** hasta **30 días**.
  - **Medio y bajo:** hasta **90 días** o la siguiente versión.
  - El navegador sigue las versiones estables de Chromium. En la etapa 1 del WebView (Qt WebEngine, sección 11) el WebView sigue, hasta donde sé, la cadencia de Qt WebEngine, que se basa en una versión de Chromium anterior a la última estable y puede ir por detrás del navegador: los plazos de arriba se aplican al WebView solo cuando haya un motor único, y mientras tanto se publica su desfase respecto al navegador en el boletín de seguridad.
- **[Acordado]** Boletín de seguridad mensual y avisos fuera de ciclo para los críticos. El nivel de parche (el índice de seguridad de 17.5) se incluye en la atestación propia, para que un banco pueda comprobarlo.
- **[Acordado]** Plazos por fases: al inicio, críticos y altos hasta 30 días y el resto hasta 90; se baja a 7, 30 y 90 cuando el equipo y la automatización lo permitan. Solo se publica lo que se pueda cumplir, porque prometer un plazo que no se cumple es peor que no prometerlo.
- **[Acordado]** Vida útil: propuesta inicial de 2 años para una línea estable (Nebula) y 5 años para Singularity (LTS), siempre limitada por el kernel LTS elegido y por el soporte de firmware del fabricante; se cuenta por línea de versión y por dispositivo, no por canal.

### 17.7 Actualización del firmware

- **[Acordado]** **Tres tipos:** (1) firmware de periféricos que el kernel carga desde el sistema (Wi-Fi, Bluetooth, GPU, audio): viaja en la imagen del sistema (`/usr/lib/firmware`, EROFS con dm-verity), se actualiza con la OTA y revierte con el slot; (2) firmware del módem y del procesador de aplicaciones, que vive en particiones del fabricante y va firmado por él: no se puede recompilar; (3) firmware de arranque (bootrom, bootloader del fabricante, TrustZone): solo lo actualiza el fabricante.
- **[Acordado]** El bundle OTA lleva un **paquete de firmware por dispositivo** (extraído de imágenes oficiales del fabricante, con sus firmas intactas y listado en la lista de blobs, 16.4), que se escribe en las particiones de firmware (en A/B donde el hardware lo tenga) junto con el sistema. Configuración muestra las versiones.
- **[Acordado]** Muchos fabricantes aplican protección anti-rollback propia en el hardware, a veces con fusibles. Queda fuera del control de CosmicOS, se documenta por dispositivo y no afecta al índice de seguridad propio (17.5).
- **[Acordado]** Donde la licencia del firmware no permita redistribuirlo, el instalador **lo copia de las particiones del fabricante del propio teléfono**; el resto sigue el paquete por dispositivo descrito arriba. ([D-18](#d-18), 22 de octubre de 2026)

---

## 18. Accesibilidad

La accesibilidad es un componente de primer nivel, no un añadido.

### 18.1 Arquitectura

- **[Acordado]** Cada función de accesibilidad es un **demonio separado**, como el teclado: proceso propio, dominio SELinux propio y acceso mínimo.

| Estado | Componente | Rol |
|---|---|---|
| **[Acordado]** | Lector de pantalla | Narra la interfaz y permite explorar por tacto (tocar para explorar, doble toque para activar). Se apoya en AT-SPI2, que ya exponen Qt y GTK. |
| **[Acordado]** | Motor de voz ([TTS](#g-tts)) | Convierte texto en voz, **en local** y sin red; lo usan el lector y las apps (por ejemplo, eSpeak NG o Piper). |
| **[Acordado]** | Reconocimiento de voz | Dictado y control por voz, también en local (por ejemplo, Whisper.cpp o Vosk). |
| **[Acordado]** | Ayudas visuales | Lupa, texto grande, alto contraste, corrección de color y reducción de movimiento, en el compositor y el shell. |
| **[Acordado]** | Braille | Soporte de pantallas braille (BRLTTY). |
| **[Acordado]** | Audio y háptica | Subtítulos en directo locales, audio mono, avisos por flash y vibración. |
| **[Acordado]** | Acceso alternativo | Navegación por botones (ya acordada junto a los gestos) y control por interruptores. |

### 18.2 Seguridad de las APIs de accesibilidad

**[Acordado]** Las APIs de accesibilidad leen la pantalla e inyectan toques, y en Android son un vector clásico de malware. Por eso:

- No se obtienen por manifiesto: solo los componentes del sistema con la etiqueta `accessibility`, asignada por el repositorio, pueden usarlas.
- Cada demonio corre sin red y sin acceso al home.
- Si algún día se admiten lectores de terceros: revisión humana reforzada, activación explícita del usuario con una confirmación dibujada por el compositor (que ninguna app puede automatizar) e indicador permanente visible.
- La inyección de entrada solo se hace por un protocolo privilegiado del compositor y solo para esa etiqueta.

### 18.3 Continuidad y calidad

- **[Acordado]** **Primer arranque y recuperación accesibles:** el asistente de configuración inicial, el modo recuperación y los avisos de arranque verificado deben poder manejarse con lector de pantalla desde el primer encendido (activable con un gesto o un botón físico).
- **[Acordado]** **Preferencias globales** (tamaño de texto, contraste, movimiento) respetadas por todas las apps.
- **[Acordado]** **Requisito del repositorio:** el linter de SPS avisa a las apps que no exponen su árbol de accesibilidad, y el SDK incluye guías para desarrolladores.
- **[Acordado]** Diseño: objetivos táctiles de tamaño mínimo, no depender solo del color y gestos siempre con alternativa (ya acordado en 9.2).
- **[Acordado]** Pruebas con usuarios reales de tecnologías de apoyo.

### 18.4 Puntos por decidir

- **[Acordado]** Lector de pantalla propio en Rust sobre AT-SPI2, empezando por un producto mínimo (navegación por foco, narración y gestos táctiles básicos) y con usuarios reales de tecnologías de apoyo desde el principio. Que yo sepa, ninguno existente cubre bien la exploración táctil en móvil: Orca está pensado para escritorio con teclado.
- **[Acordado]** **eSpeak NG** como motor de voz base. Piper queda para más adelante, si se comprueba que su consumo de batería es aceptable. ([D-19](#d-19), 22 de octubre de 2026)

### 18.5 Accesibilidad más allá de la pantalla táctil

- **[Acordado]** **Control por interruptores:** barrido de elementos (por pasos o automático, con velocidad configurable) con uno o dos interruptores (botones físicos, interruptores por Bluetooth o USB, o gestos con la cámara frontal), apoyado en el árbol de AT-SPI2.
- **[Acordado]** **Teclado externo** (USB o Bluetooth): todo el sistema se maneja con foco visible, Tab y atajos, sin necesitar la pantalla táctil.
- **[Acordado]** **Audiodescripción:** el reproductor del sistema y el WebView admiten pistas de audiodescripción, y una preferencia global las activa.
- **[Acordado]** **El lector de pantalla va primero;** después, el teclado externo y el control por interruptores, probando siempre con usuarios reales. ([D-20](#d-20), 22 de octubre de 2026)

---

## 19. Telemetría y privacidad

### 19.1 Principios

- **[Acordado]** CosmicOS es **local-first**: casi nada sale del teléfono y la mayoría de los procesos se ejecutan en el dispositivo.
- **[Acordado]** **Sin telemetría por defecto:** ni analítica, ni identificador de publicidad, ni un identificador estable de dispositivo en ninguna petición.

### 19.2 Lo poco que sí sale

**[Acordado]** Todo el tráfico que sale del dispositivo se declara de forma explícita:

| Tráfico | Para qué | Qué puede ver el servidor | Mitigación |
|---|---|---|---|
| Índice de actualizaciones (TUF) | Saber si hay versiones nuevas | IP y hora, sin identidad | Sin ID ni cookies; espejos autoalojables |
| Descarga de apps y del sistema | Instalar | Qué paquete se descarga | Espejos y caché local; descarga del índice completo para no revelar qué se busca |
| Gateway de push | Despertar a las apps | Token de dispositivo y volumen de avisos | Contenido cifrado de extremo a extremo, `TTL` y sin conocer la presencia (10.2) |
| Hora (NTS) | Mantener el reloj | IP | NTS y servidores configurables |
| Resolución DNS ([DoH](#g-doh)/DoT) | Conectividad | Dominios consultados | Consultas cifradas y resolvedor configurable |
| Detección de portal cautivo | Saber si hay internet real | IP | Configurable y desactivable |
| Mapas | Ver mapas | Zonas consultadas | Mapas sin conexión por defecto |
| Geolocalización por WiFi/celdas | Ubicación rápida | Redes cercanas, si se consulta | Base de datos descargada y consultada en local |

### 19.3 Controles del usuario

- **[Acordado]** **Indicadores de privacidad** (micrófono, cámara o ubicación en uso) dibujados por el compositor: las apps no pueden falsificarlos ni ocultarlos.
- **[Acordado]** **Panel de privacidad** alimentado por el registro de auditoría: qué app accedió a qué y cuándo (10.4).
- **[Acordado]** **Red visible por app:** cada app declara en el manifiesto si usa red, y el usuario puede verlo y quitarlo.
- **[Acordado]** Portapapeles solo para la app en primer plano, con aviso al leerlo; ubicación aproximada como opción.
- **[Acordado]** Las apps no acceden a identificadores de hardware, y las direcciones MAC son aleatorias.
- **[Acordado]** Informes de fallos **solo voluntarios**, con el paquete de diagnóstico revisado por el usuario antes de salir (10.4).
- **[Acordado]** Aprendizaje local: teclado, predicción y voz procesan y guardan todo en el dispositivo, sin red (10.3 y sección 18).
- **[Acordado]** Cifrado en reposo con clave del usuario (fscrypt), protegida por un elemento seguro si el hardware lo permite.
- **[Acordado]** **Sin cuenta para descargar:** solo los desarrolladores tienen identidad (su clave); los usuarios no.

### 19.4 Puntos por decidir

- **[Acordado]** Contadores de descargas agregados y públicos, sin identificadores.
- **[Acordado]** Mapas sin conexión por defecto; búsquedas opcionales mediante un proxy propio.

---

## 20. Versionado

### 20.1 Documentación

- **[Acordado]** Formato `vX.Y.Z` (`v0.0.1`, `v0.0.2`, …). Cada dígito va de 0 a 9 y, al llegar a 10, el contador pasa al dígito siguiente y el anterior vuelve a 0 (`v0.0.9` → `v0.1.0`).
- **[Acordado]** Los **tags** son únicamente para el código fuente; la documentación solo usa su número de versión y el historial de la sección 21.

### 20.2 Código fuente

- **[Acordado]** Esquema de **cuatro posiciones** (confirmado; la mención anterior a cinco dígitos fue un error):

| Posición | Ejemplo | Significado |
|---|---|---|
| 1.ª | `1.` | Versión **completa** (generación del sistema); sigue en revisión, y la estabilidad la indica el tag |
| 2.ª | `1.1.` | Merge en el que fue confirmada |
| 3.ª | `1.1.1.` | Commit en el que fue firmada |
| 4.ª | `1.1.1.1` | Cuántos commits firmados lleva; al llegar a 20 vuelve a 0 y suma 1 a la 3.ª (ver la regla de acarreo) |

- **[Acordado]** **Regla de acarreo** (cerrada el 20 de octubre de 2026): la 4.ª posición cuenta commits firmados y, al llegar a 20, vuelve a 0 y suma 1 a la 3.ª; la 3.ª, al llegar a 10, vuelve a 0 y suma 1 a la 2.ª, y así hasta la 1.ª, que representa el merge final de la versión estable. Es un contador tipo cuentakilómetros: dos versiones distintas nunca comparten número.
- **[Acordado]** Detalles que el texto no fija y que se asumen: «al llegar a 20» significa que el commit 20 ya hace el acarreo (la 4.ª toma los valores 0 a 19, igual que los dígitos de la documentación pasan al llegar a 10); la 2.ª también acarrea al llegar a 10; la 1.ª no tiene tope. Si el 20 debía ser un valor válido, es un cambio de una línea.

### 20.3 Tags (canales de estabilidad)

- **[Acordado]** Los tags, solo en código fuente, indican la estabilidad:

| Tag | Significado | Nombre anterior |
|---|---|---|
| **Nebula** | Estable | Lazer |
| **Pulsar** | Semiestable (beta) | Meltdown |
| **Singularity** | LTS | Gecko |
| **Quasar** | Rolling: cada commit es una actualización | Core |
| **EventHorizon** | Ultraexperimental: aquí se forjan las ideas más locas y, una vez maduras, puede nacer de aquí un LTS | HardCore |

Los nombres se cambiaron el 19 de octubre de 2026. Los canales sirven tanto para el código fuente como para las vías de actualización del sistema (sección 17).

### 20.4 Puntos por decidir

- **[Acordado]** El acarreo de la 4.ª a la 3.ª posición queda resuelto en 20.2.
- **[Acordado]** La ABI se versiona aparte del sistema (ver 7.1).
- **[Acordado]** El índice anti-rollback es global, monótono e independiente del número de versión (ver 17.5).
- **[Acordado]** Las ramas LTS **congelan las tres primeras posiciones y la cuarta cuenta los parches**; un bot asigna los números desde la rama principal protegida. ([D-21](#d-21), 22 de octubre de 2026)
- **[Acordado]** Se mantienen las cuatro posiciones. Una revisión externa sugirió una alternativa más simple (semver para los releases, más el hash de `git describe` y el tag de canal, sin acarreos); queda anotada por si el esquema resulta incómodo en la práctica.
- **[Acordado]** Ruta de promoción entre canales: EventHorizon → Quasar → Pulsar → Nebula → Singularity, sin saltos directos a LTS; formato del identificador completo, por ejemplo `1.4.12.7-nebula`.

---

## 21. Historial de versiones

| Versión | Fecha | Cambios |
|---|---|---|
| v0 | 18 oct 2026 | Primer borrador: kernel, aislamiento, SPS, repositorio, UI, servicios, directorios, logging, hoja de ruta |
| v0.0.1 | 18 oct 2026 | Kernel híbrido (`kernel.root`); recomendaciones técnicas pasadas a acordadas; sistema de ficheros por roles (EROFS, F2FS/Btrfs, A/B) y AFS como meta futura; formato de distribución (imagen GPT, UKI y bundle OTA) |
| v0.0.2 | 18 oct 2026 | Smithay confirmado como base del compositor; estructura del `Manifest.json` (encabezado, cuerpo, cola); esquema de versionado y tags |
| v0.0.3 | 19 oct 2026 | Versionado de código confirmado a cuatro posiciones y canales renombrados (Nebula, Pulsar, Singularity, Quasar, EventHorizon); definición de `kernel.root`; modelo de actualización del sistema; modelo de amenaza; accesibilidad; telemetría y privacidad; glosario con enlaces; nuevo estado [Propuesto]; correcciones de formato en las decisiones abiertas |
| v0.0.4 | 19 oct 2026 | Decisiones del autor tras una revisión externa: módulos solo durante el arranque (sustituye a `CONFIG_MODULES=n`), SELinux sellado solo en Nebula, licencia GPL-3.0 o AGPL-3.0, nombre CosmicOS confirmado, «híbrido» justificado con drivers en espacio de usuario, clase de app en el manifiesto, fin del protocolo de pago propio. Nuevos: `kernel.root` con heartbeat y watchdog, filtro seccomp base por clase, namespace de red vacío, notas sobre Lua y Go, EROFS sin loop, sección 11.1 a 11.6 (códecs, DRM, VoLTE/IMS, fabricante y atestación, sideloading, licencia y gobernanza), riesgos externos, aclaración sobre Helium y Titanium, glosario ampliado |
| v0.0.5 | 20 oct 2026 | Licencias por capas (LGPL para el SDK, AGPL para red, GPL para kernel, bootloader y sistema de ficheros) con sus matices; `kernel.root` redefinido como hermano del PID 2 original, ventana hacia el kernel y supervisor parcial; ModemManager acordado; partición propia para el almacén de apps; regla de acarreo del versionado cerrada; nota sobre ungoogled-chromium, Thorium y MV2 |
| v0.0.6 | 20 oct 2026 | Licencias: LGPL-3.0 (SDK), AGPL-3.0 (red), GPL-3.0 (resto) y GPL-2.0 solo donde es obligatoria, con el kernel aclarado; ungoogled-chromium acordado; `kernel.root` como supervisor de solo lectura que reinicia procesos colgados, con detección de cuelgues y las dos formas de relacionarlo con `kthreadd`; principio «el dispositivo es del usuario» y regla de bootloader desbloqueable |
| v0.0.7 | 20 oct 2026 | glibc y ABI versionada con `compatible_since` (7.1); anti-rollback sin fusibles, con índice de seguridad global, suelo y autorización local para bajar de versión, y cambios de canal permitidos (17.5); repositorios de terceros aprobados; tres modos de dispositivo (Estándar, Desarrollador básico y avanzado); gobernanza mínima; compilaciones reproducibles (8.4); instantáneas previas a actualizar y copias de seguridad (5.3); multiusuario fuera del alcance inicial |
| v0.0.8 | 20 oct 2026 | Todas las propuestas anteriores pasan a acordadas; U-Boot como capa de compatibilidad con bootloaders de fabricante; tras el desbloqueo se ofrece restaurar la copia interna o externa; nuevas propuestas sobre dónde se restaura y cómo sobrevive la clave de la copia interna |
| v0.0.9 | 20 oct 2026 | Revisión editorial: fecha del encabezado, frase cortada de 11.4, redacción de `kernel.root` (primer proceso que `init` crea tras arrancar), reinicio de procesos según el supervisor y no según SELinux, sincronización de la tabla de decisiones abiertas con las secciones, Go como componente privilegiado por debajo de `kernel.root`; nuevos: protección frente al OOM killer, módulos del sistema base en el rootfs verificado, `modules_disabled` y la descarga de módulos, consecuencia para periféricos conectados tras el arranque, y WebView como punto abierto |
| v0.1.0 | 20 oct 2026 | WebView como componente del sistema (opción B) con propuesta de etapas, seguridad y API; nombre CosmicOS definitivo; el WebView se añade a la hoja de ruta y a los riesgos. Tras `v0.0.9` el contador de la documentación acarrea: `v0.1.0` |
| v0.1.1 | 21 oct 2026 | Revisión de la sección de WebView: el motor único pasa a ser el objetivo y Qt WebEngine la etapa 1 transitoria, con el riesgo de dos linajes de Chromium; ninguna app enlaza el motor (cliente fino con ABI en C y adaptadores Qt y GTK); los procesos del WebView heredan el límite de la app anfitriona; Qt WebEngine bajo LGPL-3.0 (por confirmar); suelo de seguridad por app (8.3) |
| v0.1.2 | 21 oct 2026 | Nuevo modelo de IPC (7.2: D-Bus con dbus-broker y proxy por app, sockets con `SO_PEERSEC`, Binder como alternativa y nota sobre HongMeng); GPU con tratamiento propio (9.3); audio, llamadas y Bluetooth (10.7); suspensión y reanudación (10.8); criterios del dispositivo objetivo (11.7); estrategia de alcance con el escritorio como posible producto (13); compromiso de parcheo con plazos (17.6) |
| v0.1.3 | 21 oct 2026 | IPC cerrado por ahora en D-Bus y sockets Unix, con Binder a futuro; Bluetooth sobre BlueZ con plan de mejora (interoperabilidad, gestor propio, seguridad); pantalla de arranque propia con animación y audio, sin módulo de kernel (5.4); móvil primero y escritorio después, con el entorno de escritorio como herramienta de desarrollo y primer hito móvil con criterios de salida (13) |
| v0.1.4 | 21 oct 2026 | `repo` y manifiesto XML para obtener el código (8.5); `/usr/apps` (plural) cerrado; puntos abiertos pasados a propuestas de cierre (`kernel.root`, nombres, FS, plazos, lector de pantalla, GPU, Bluetooth, consumo en reposo, ruta de canales, vida útil, y otros) con correcciones al umbral de reinicios, al consumo en reposo y al dispositivo objetivo; nuevos: pérdida o robo (5.5), pruebas y CI (7.3), rendimiento (9.4), localización (9.5), estrategia de adopción (11.8), multiusuario (12.1), firmware (17.7) y accesibilidad más allá de la pantalla (18.5) |
| v0.1.5 | 21 oct 2026 | Todas las propuestas pasan a [Acordado] por decisión del autor; carpetas de logs `~/.var/logs/<app>/{json,raw}` en minúsculas |
| v0.1.6 | 21 oct 2026 | Qt WebEngine como primera implementación del WebView y campos finos del manifiesto cerrados; familias Pixel y Motorola como dispositivos candidatos (modelo concreto abierto, con las dudas sobre re-bloqueo y soporte upstream); forma jurídica y mantenedores aplazados |
| v0.1.7 | 21 oct 2026 | Matriz de dispositivos (Pixel, Motorola con Snapdragon, Motorola con MediaTek) frente a los criterios de 11.7, con lo verificado y lo pendiente; los campos finos del manifiesto quedan definitivos |
| v0.1.8 | 21 oct 2026 | Los teléfonos centrados en el usuario (Fairphone y similares) pasan a ser familia objetivo; ficha del Fairphone 5 en la matriz de dispositivos |
| v0.1.9 | 22 oct 2026 | La sección 14 pasa a ser un registro: 14.1 reúne todos los puntos abiertos con identificador `D-nn` (el texto solo lleva punteros) y 14.2 las decisiones cerradas; se añaden como abiertos la política de BPF firmados, las licencias de módulos mixtos y el alcance del modo Desarrollador básico; leyenda sin referencia de versión; matiz del WebView en la etapa 1 en 17.6; referencia del criterio 6 de 11.7 corregida |
| v0.2.0 | 22 oct 2026 | Se cierran 21 de los 24 puntos abiertos según las respuestas del autor (la tabla 14 se divide en vivas y cerradas); repositorios oficiales Main, Space y MTL (8.1); CinitD como init propio (7.4) y propuesta de nombres de componentes (7.5); PhateonWM pasa a escribirse PhanteonWM; calendario de apagado de 2G/3G investigado (D-11); nuevos puntos D-25 a D-27; estado [Aplazado] |
| v0.2.1 | 22 oct 2026 | Convención y nombres de componentes confirmados (D-27 cerrado); MTL = *Main Third-party License*; Frontera FireWall (10.9); nuevas secciones propuestas: lo que el documento no cubre (1.1), auditoría y pruebas ofensivas (7.6), respuesta a incidentes (8.6), documentación para desarrolladores (11.9) y comunidad (11.10); nuevos puntos vivos D-28 a D-31 |
| v0.2.2 | 24 oct 2026 | CinitD: arquitectura final acordada y solo el prototipado abierto (D-25); Frontera con primer hito de modelo mínimo y dos apps; MTL: propuesta de responsabilidad legal (D-26); nuevas secciones propuestas: ajustes (10.10), cuentas y sincronización (10.11), periféricos (10.12), financiación (11.11) y estado de implementación (13.1); nuevos puntos vivos D-32 a D-35 |
| v0.2.3 | 24 oct 2026 | Comunidad (11.10) revisada: Matrix como chat principal, Discord solo como puerta de entrada (D-36) y sin decisiones allí; correo del proyecto `@cmail.dev` (D-37) y listas de correo públicas para las decisiones técnicas; aclaración en 10.11 de que esas direcciones no son cuentas del sistema |
| v0.2.4 | 25 oct 2026 | Frontera con el hardware del fabricante (16.6): regla de ningún blob dentro de una app, tabla por subsistema y contrato de interfaz; hoja de ruta (13): emergencias como criterio de salida, plataformas de desarrollo previas y nota sobre RISC-V; nuevos puntos vivos D-38 (servidor de hardware tipo Halium) y D-39 (tareas del PID 1) |
| v0.2.5 | 25 oct 2026 | D-38 cerrado (no hay servidor de hardware tipo Halium, con tres condiciones para reactivarlo); reparto de tareas del PID 1 refinado (7.4, `cdevd` sin módulos, apagado en dos mitades) y pendiente de confirmar (D-39); 16.6: el blob se traslada al kernel y cuenta para el TCB, y SELinux no es de Android; sección 1: ARM64 como objetivo de teléfono |
| v0.2.6 | 6 nov 2026 | Principio 9 (reutilizar antes de escribir) y presupuesto de componentes propios (7.7); estructura del árbol de código del autor, monorepo al inicio y qué no se versiona (8.7); estado de implementación actualizado a «andamiaje» (13.1); nuevo punto D-40 |
| v0.2.7 | 6 nov 2026 | `Docs/` se queda con mayúscula (8.7) |
| v0.2.8 | 7 nov 2026 | Organización `OpenCosmicProject` y un repositorio por carpeta (8.7); kernel como fork con `arch/` purgado, ramas, script reproducible y poda de drivers solo por Kconfig (3); musl estático para `cinit-early` y CinitD (7.1); `default-members` y `cosmic-build` en el primer hito (7.7); sistema de construcción Bash y Ninja, D-41 (8.8); herramientas USB `cosmic-fastboot`, `cad` y `cosmic-usb`, D-42 (11.12); `Docs/` como guías prácticas que enlazan a la spec; estado de implementación actualizado (13.1); nuevo D-43 (parámetros y compilador del kernel) |
| v0.2.9 | 7 nov 2026 | Instalación y primer arranque, D-44 cerrado con la opción A (5.6); `temp-usr` en RAM sin shell ni contraseña y con dominio SELinux propio; home con el nombre de usuario elegido, reglas del nombre y `$HOME` virtual en el sandbox (12); nuevo D-45 (dónde viven `passwd` y `shadow`); `cosmic-build` compila toda la base, con la lista de correcciones, la compilación cruzada primero (8.8); imágenes dispersas en `cosmic-fastboot` (D-42) |

---

<a id="glosario"></a>

## 22. Glosario

Orden alfabético. Cada término está enlazado desde su primera aparición en el documento.

- <a id="g-ab"></a>**A/B (particiones)** — Dos copias del sistema (A y B): se actualiza la inactiva y, si el arranque falla, el dispositivo vuelve a la anterior.
- <a id="g-abi"></a>**ABI** — Interfaz binaria de aplicaciones: el contrato, a nivel de binario, entre una app y las librerías del sistema. Si cambia, las apps compiladas contra la versión anterior pueden dejar de funcionar.
- <a id="g-afs"></a>**AFS (Atomic File System)** — Sistema de ficheros propio, previsto como meta de largo plazo.
- <a id="g-agpl"></a>**GPL / LGPL / AGPL** — Licencias copyleft: quien distribuye una versión modificada debe publicar su código con la misma licencia. La LGPL permite enlazar la biblioteca desde programas con otra licencia (de forma que el usuario pueda sustituirla); la AGPL extiende la obligación a quien ofrece el software como servicio por red.
- <a id="g-apk"></a>**APK** — Formato de paquete de aplicaciones de Android; CosmicOS no lo usa.
- <a id="g-art"></a>**ART / JVM** — Máquinas virtuales que ejecutan las apps de Android (Android Runtime y Java Virtual Machine); CosmicOS usa binarios nativos.
- <a id="g-aslr"></a>**ASLR** — Aleatorización de las direcciones de memoria de un proceso, para dificultar los exploits.
- <a id="g-atestacion"></a>**Atestación** — Mecanismo por el que el dispositivo demuestra a un tercero (por ejemplo, un banco) en qué estado de arranque está.
- <a id="g-atspi"></a>**AT-SPI2** — Interfaz de accesibilidad de Linux, sobre D-Bus, por la que las apps exponen su contenido a los lectores de pantalla; la usan Qt y GTK.
- <a id="g-avb"></a>**AVB (Android Verified Boot)** — Arranque verificado de muchos teléfonos: el bootloader comprueba que lo que arranca está firmado por una clave de confianza (la del fabricante o, en algunos dispositivos, una que el usuario inscribe).
- <a id="g-backoff"></a>**Backoff exponencial** — Esperar cada vez más entre reintentos tras un fallo, para no caer en un bucle de reinicios.
- <a id="g-binder"></a>**Binder** — Mecanismo de IPC del kernel Linux (usado por Android) en el que el kernel media cada llamada y entrega la identidad del emisor; con `binderfs` se puede usar sin Android.
- <a id="g-blob"></a>**Blob** — Firmware binario cerrado, sin código fuente auditable, como el del módem.
- <a id="g-bluez"></a>**BlueZ** — Pila Bluetooth oficial de Linux (demonio `bluetoothd` y librerías), que habla por D-Bus y se apoya en la parte Bluetooth del kernel.
- <a id="g-bootloader"></a>**Bootloader** — Programa que arranca el sistema operativo; en los teléfonos suele venir bloqueado por el fabricante.
- <a id="g-broker"></a>**Broker (de permisos)** — Intermediario del sistema que da a las apps acceso a recursos sensibles (cámara, micrófono, archivos) con consentimiento y revocación, sin acceso directo.
- <a id="g-btrfs"></a>**Btrfs** — Sistema de ficheros copy-on-write con checksums de datos, compresión y snapshots.
- <a id="g-nebula"></a>**Canales de actualización** — Vías de estabilidad del código y del sistema: Nebula (estable), Pulsar (semiestable, beta), Singularity (LTS), Quasar (rolling) y EventHorizon (ultraexperimental); ver 20.3.
- <a id="g-cgroups"></a>**cgroups** — Mecanismo del kernel para agrupar procesos, limitar sus recursos (CPU, memoria, E/S) y congelarlos.
- <a id="g-e2e"></a>**Cifrado de extremo a extremo** — Solo el emisor y el receptor pueden leer el contenido; ni siquiera el servidor intermedio.
- <a id="g-codec"></a>**Códec** — Programa (o bloque de hardware) que comprime y descomprime audio o vídeo, como Opus, AAC o H.264. Los códecs por software no viven en el kernel sino en bibliotecas de espacio de usuario.
- <a id="g-compositor"></a>**Compositor** — Programa que dibuja las ventanas en pantalla y gestiona la entrada; en Wayland también hace de gestor de ventanas.
- <a id="g-creationde"></a>**CreationDE** — Shell móvil de CosmicOS: lanzador de apps, barra de estado, pantalla de bloqueo y centro de notificaciones.
- <a id="g-criogenia"></a>**Cuarentena (criogenia)** — Estado de una versión de app con clave comprometida: no se ejecuta, pero conserva sus datos hasta recibir una actualización firmada con una clave nueva.
- <a id="g-dbus"></a>**D-Bus / dbus-broker** — Bus de mensajes de Linux por el que se hablan servicios y aplicaciones (BlueZ, ModemManager, AT-SPI2…); dbus-broker es una implementación moderna del bus.
- <a id="g-dco"></a>**DCO (Developer Certificate of Origin)** — Declaración que el contribuyente firma en cada commit (`Signed-off-by`) de que tiene derecho a aportar ese código bajo la licencia del proyecto; no transfiere derechos, a diferencia de un CLA.
- <a id="g-dmverity"></a>**dm-verity / fs-verity** — Verificación criptográfica, hecha por el kernel, de cada bloque (dm-verity) o archivo (fs-verity), para garantizar que lo leído no fue alterado.
- <a id="g-doh"></a>**DoH / DoT / NTS** — DNS sobre HTTPS o sobre TLS (cifra las consultas) y Network Time Security (sincronización de hora autenticada).
- <a id="g-drm"></a>**DRM / Widevine** — Protección de contenido: el vídeo viaja cifrado y solo un dispositivo con licencia puede descifrarlo. Widevine es el sistema de Google; su nivel L1 exige hardware certificado y L3 funciona por software.
- <a id="g-ebpf"></a>**eBPF** — Programas pequeños y verificados que se ejecutan dentro del kernel para observar o filtrar eventos.
- <a id="g-se"></a>**Elemento seguro** — Chip dedicado y resistente a manipulaciones que guarda claves y credenciales de pago.
- <a id="g-erofs"></a>**EROFS** — Sistema de ficheros de solo lectura, comprimido y rápido; Android lo usa en varias particiones del sistema.
- <a id="g-esp"></a>**ESP** — Partición del sistema EFI, donde viven los archivos de arranque (como la UKI).
- <a id="g-f2fs"></a>**F2FS** — Sistema de ficheros pensado para memoria flash, con compresión y cifrado por usuario (fscrypt).
- <a id="g-fscrypt"></a>**fscrypt** — Cifrado a nivel de sistema de ficheros, con claves por usuario.
- <a id="g-fuzzing"></a>**Fuzzing** — Probar un programa con entradas aleatorias o malformadas para encontrar fallos.
- <a id="g-gki"></a>**GKI (Generic Kernel Image)** — Kernel casi vanilla de Android; el soporte de cada fabricante va en módulos `.ko`.
- <a id="g-glibc"></a>**glibc** — Biblioteca C estándar de GNU, la de Arch y de la mayoría de distribuciones Linux; versiona sus símbolos para que lo compilado contra una versión antigua siga funcionando en una más nueva.
- <a id="g-gms"></a>**GMS** — Google Mobile Services: los servicios de Google en Android (push, mapas, inicio de sesión, verificación de apps).
- <a id="g-gpt"></a>**GPT** — Formato moderno de tabla de particiones de disco.
- <a id="g-hardening"></a>**Hardening (kCFI, PAC, BTI, MTE)** — Defensas que dificultan explotar bugs: kCFI y shadow call stack protegen el flujo de ejecución; PAC y BTI (ARM64) firman punteros y destinos de salto; MTE etiqueta la memoria para detectar errores.
- <a id="g-iio"></a>**IIO** — Subsistema del kernel para sensores (acelerómetro, luz, giroscopio).
- <a id="g-init"></a>**init (PID 1)** — Primer proceso de usuario que arranca el kernel; en CosmicOS es **CinitD** (Cosmic Init Driver), un init propio y mínimo en Rust, que arranca `kernel.root` y lo vigila.
- <a id="g-iommu"></a>**IOMMU** — Unidad de hardware que limita a qué memoria puede acceder cada dispositivo por DMA; contiene a periféricos poco fiables como el módem.
- <a id="g-ipc"></a>**IPC** — Comunicación entre procesos: sockets, buses de mensajes o memoria compartida con los que los procesos se piden cosas unos a otros.
- <a id="g-jcs"></a>**JCS (RFC 8785)** — Forma canónica de serializar JSON, para que dos programas obtengan los mismos bytes y, por tanto, la misma firma.
- <a id="g-hibrido"></a>**Kernel híbrido** — Término propio de CosmicOS: Linux compilado como monolítico en el que, tras el arranque, todos los procesos divergen de `kernel.root` y viven aislados en su sandbox, y con algunos drivers básicos en espacio de usuario.
- <a id="g-kernel-root"></a>**kernel.root** — Proceso supervisor mínimo y de solo lectura de CosmicOS, emparentado con el PID 2 del kernel: ventana en espacio de usuario hacia el kernel y supervisor parcial; inmortal para root, y de él divergen todos los demás (ver 3.1).
- <a id="g-landlock"></a>**Landlock** — Módulo de seguridad del kernel con el que un proceso restringe su propio acceso a archivos y red.
- <a id="g-leaudio"></a>**LE Audio** — Nueva generación de audio Bluetooth (códec LC3), con soporte para audífonos y ayudas auditivas y para compartir audio entre varios dispositivos.
- <a id="g-libcamera"></a>**libcamera** — Biblioteca de Linux para controlar cámaras y su procesado de imagen (ISP).
- <a id="g-lsm"></a>**LSM** — Linux Security Module: marco del kernel donde se enchufan módulos de seguridad como SELinux; se compilan dentro del kernel y no son `.ko`.
- <a id="g-manifest"></a>**Manifest.json** — Archivo de cada paquete `.sps` que declara qué es la app, qué librerías usa y qué permisos solicita (ver 6.3).
- <a id="g-mcs"></a>**MCS** — Multi-Category Security: categorías de SELinux con las que se separan las apps entre sí.
- <a id="g-modo-dev"></a>**Modo Desarrollador (básico / avanzado)** — Básico: se activa tocando siete veces el número de compilación, permite repositorios de terceros y muestra el botón de desbloqueo OEM; las apps `<bancario>` siguen funcionando. Avanzado: casi todos los permisos, se elige al arrancar (variante sellada de la política) y queda en el estado de arranque; las apps `<bancario>` no se ejecutan.
- <a id="g-namespaces"></a>**Namespaces** — Mecanismo del kernel que da a cada proceso su propia vista de archivos, procesos, red o usuarios.
- <a id="g-ofono"></a>**oFono / ModemManager** — Pilas de software de telefonía de Linux (llamadas, SMS y datos móviles).
- <a id="g-opus"></a>**Opus** — Códec de audio libre y sin regalías, usado en llamadas y música.
- <a id="g-ota"></a>**OTA** — Over-the-air: actualización que se descarga por red, sin cable.
- <a id="g-phanteon"></a>**PhanteonWM** — Compositor Wayland de CosmicOS, construido sobre Smithay.
- <a id="g-pipewire"></a>**PipeWire** — Servidor multimedia de Linux (audio, vídeo y cámaras) que además concede o niega a las apps el acceso a esos dispositivos.
- <a id="g-plymouth"></a>**Plymouth** — Programa estándar de Linux para mostrar una pantalla de arranque animada desde el initrd, con entrada de contraseña; usa DRM/KMS.
- <a id="g-postmarketos"></a>**postmarketOS** — Distribución Linux para teléfonos que busca usar un kernel lo más cercano posible al upstream; ya tiene trabajo de soporte de hardware.
- <a id="g-pwa"></a>**PWA** — Aplicación web progresiva: un sitio web que se instala y se comporta como una app.
- <a id="g-qt"></a>**Qt6 / GTK4 / libadwaita** — Toolkits para construir interfaces: Qt6 es el SDK oficial; GTK4 con libadwaita, el secundario.
- <a id="g-auditoria"></a>**Registro de auditoría** — Registro de eventos de seguridad (denegaciones de SELinux, cuarentenas, cambios de política) fuera del alcance del usuario y de las apps, de solo anexar y con hash encadenado.
- <a id="g-repo"></a>**repo** — Herramienta de Google que gestiona muchos repositorios git a la vez mediante un manifiesto XML que fija la revisión de cada uno; la usan AOSP y OpenHarmony.
- <a id="g-rollback"></a>**Rollback** — Volver a una versión anterior cuando una actualización falla.
- <a id="g-sandbox"></a>**Sandbox** — Entorno aislado en el que un proceso corre con acceso mínimo al resto del sistema.
- <a id="g-seccomp"></a>**seccomp** — Filtro del kernel que limita qué llamadas al sistema puede usar un proceso.
- <a id="g-selinux"></a>**SELinux** — Control de acceso obligatorio: una política que el kernel aplica incluso al usuario root.
- <a id="g-shell"></a>**Shell (de interfaz)** — Capa de interfaz sobre el compositor: lanzador, barra de estado, bloqueo y notificaciones.
- <a id="g-sideloading"></a>**Sideloading** — Instalar apps desde una fuente distinta del repositorio oficial.
- <a id="g-cosign"></a>**Sigstore / cosign** — Herramientas de firma de software con registro público de transparencia.
- <a id="g-sla"></a>**SLA** — Acuerdo de nivel de servicio: aquí, el plazo máximo que el proyecto se compromete a tardar en publicar un parche según la gravedad del fallo.
- <a id="g-slsa"></a>**SLSA** — Marco de niveles para garantizar la integridad y la procedencia de una cadena de compilación.
- <a id="g-smithay"></a>**Smithay** — Biblioteca en Rust para construir compositores Wayland.
- <a id="g-sps"></a>**SPS** — Secure-Package-System: el gestor de paquetes de CosmicOS.
- <a id="g-sps-pkg"></a>**.sps (paquete)** — Formato de paquete de CosmicOS: binario nativo más `Manifest.json`.
- <a id="g-tcb"></a>**TCB** — Trusted Computing Base: el conjunto de componentes en los que hay que confiar; cuanto más pequeña, menos superficie de ataque.
- <a id="g-tmpfs"></a>**tmpfs** — Sistema de ficheros en RAM; se borra al reiniciar.
- <a id="g-toctou"></a>**TOCTOU** — Time-of-check to time-of-use: ataque que aprovecha que algo cambia entre que se verifica y que se usa.
- <a id="g-tts"></a>**TTS** — Text-to-speech: síntesis de voz.
- <a id="g-tuf"></a>**TUF** — The Update Framework: diseño de metadatos firmados que protege las actualizaciones contra rollback, índices viejos y el compromiso de una sola clave.
- <a id="g-uboot"></a>**U-Boot** — Cargador de arranque abierto (GPL-2.0 o posterior), muy usado en placas y teléfonos Linux; puede arrancar imágenes UEFI y hacer de capa de compatibilidad con el bootloader del fabricante.
- <a id="g-uefi"></a>**UEFI** — Estándar moderno de firmware de arranque; ARM SystemReady es la certificación para que un dispositivo arranque imágenes estándar.
- <a id="g-uki"></a>**UKI** — Unified Kernel Image: un único archivo EFI firmado con kernel, initrd y línea de comandos.
- <a id="g-unifiedattestation"></a>**UnifiedAttestation** — Iniciativa abierta (Apache 2.0) respaldada por Volla, Murena, iodé y otros, que busca una alternativa abierta a Play Integrity; sin versión publicada según lo consultado.
- <a id="g-volte"></a>**VoLTE / IMS** — Llamadas de voz sobre la red 4G/5G (VoLTE) a través del subsistema IMS del operador; sin ellas el teléfono depende del 2G/3G, que muchos operadores están apagando.
- <a id="g-vulkan"></a>**Vulkan / GLES / llvmpipe** — API gráfica moderna; GLES es la alternativa más simple y llvmpipe, el renderizado por software.
- <a id="g-wayland"></a>**Wayland** — Protocolo moderno de pantalla; más seguro que X11 porque las apps no ven la entrada ni la pantalla de otras.
- <a id="g-webpush"></a>**Web Push** — Estándar (RFC 8030, 8291 y 8292/VAPID) para enviar notificaciones push cifradas.
- <a id="g-yubikey"></a>**YubiKey** — Llave de hardware (FIDO2) que guarda la clave privada y firma sin exponerla.

---

*Fin de la especificación v0.2.9. Documento vivo: versionar en el repositorio y actualizar las etiquetas de estado a medida que se cierren las decisiones.*
