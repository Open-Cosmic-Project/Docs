# Arquitectura del Sistema CosmicOS

> **Fuente de verdad:** Este documento es una guía práctica y resumen visual. La definición formal completa de la arquitectura reside en [CosmicOS-spec.md](./CosmicOS-spec.md) (Secciones 3, 3.1, 4, 7 y 10).

---

## 1. Diagrama de Arranque y Supervisión

En el arranque, el kernel crea como hermanos al **PID 1** (`cinitd`) y al **PID 2** (`kthreadd`, reservado a hilos del kernel). Siguiendo la opción (a) de la Sección 3.1, `kernel.root` es el **primer proceso que CinitD crea tras arrancar** (el segundo proceso de usuario del sistema), identificado por su dominio SELinux `kernel_root_t` y por un `pidfd`. De él divergen todos los demás servicios y aplicaciones en sandboxes independientes:

```
                     [ Kernel Linux (PID 0) ]
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
          [ PID 1: CinitD ]             [ PID 2: kthreadd ]
          • Montajes virtuales /proc...        │
          • Recolecta huérfanos         (Hilos del kernel)
          • kernel.modules_disabled=1
          • Heartbeat /dev/watchdog
                 │
                 ▼
          [ kernel.root ]
          (Supervisor inmutable, SELinux: kernel_root_t)
          • Ventana cerrada de 5 operaciones
          • Backoff exponencial y aislamiento
                 │
  ┌──────────────┴──────────────┐
  ▼                             ▼
[ clauncher ]             [ Demonios ]
(Aislamiento dinámico)    • clogd (logs)
• Mount NS + Landlock     • cnotifd (push/notif)
• Seccomp BPF (Syscalls)  • cpowerd (energía)
• Network NS vacío        • cbrokerd (portales)
• execve() binario app    • cfronterad (red)
```

---

## 2. Resumen de Componentes Clave

Para consultar el diseño completo y los compromisos técnicos, dirígete a las secciones de la especificación maestra:

- **CinitD (PID 1):** Alcance, recolección de huérfanos y sellado de módulos $\rightarrow$ [Sección 7.4](./CosmicOS-spec.md#74-init-propio-cinitd).
- **Supervisor `kernel.root`:** Primer proceso de usuario lanzado por CinitD, dominio `kernel_root_t`, ventana de 5 operaciones y mitigación del OOM killer $\rightarrow$ [Sección 3.1](./CosmicOS-spec.md#31-kernelroot).
- **Launcher (`clauncher`):** Aplicación de Landlock, namespaces y filtros seccomp base por clase de app $\rightarrow$ [Sección 4](./CosmicOS-spec.md#4-modelo-de-aislamiento-de-aplicaciones).
- **Orquestador (`corchd`):** Demonio sin privilegios en Go que coordina el sistema y delega en el launcher $\rightarrow$ [Sección 7](./CosmicOS-spec.md#7-arquitectura-de-software-del-sistema).
- **Comunicación IPC:** Sockets Unix con `SO_PEERSEC` y mediación por portales (`cbrokerd`) $\rightarrow$ [Sección 7.2](./CosmicOS-spec.md#72-comunicacion-entre-procesos-ipc).
- **Jerarquía de Almacenamiento:** Partición inmutable `/usr/apps` y único directorio escribible por app `~/.var/capps/<app>/` $\rightarrow$ [Sección 5](./CosmicOS-spec.md#5-estructura-de-directorios).
