# CosmicOS — Guías Prácticas y Documentación

Bienvenido al repositorio de documentación técnica de **CosmicOS**, desarrollado bajo la organización **OpenCosmicProject** (OCP).

> [!IMPORTANT]
> **Única fuente de verdad:**  
> La especificación maestra de arquitectura y diseño es [**`Docs/CosmicOS-spec.md`**](./CosmicOS-spec.md) (y sus versiones sucesivas).  
> Los documentos breves de esta carpeta son **manuales de procedimiento prácticos** (compilación, scripts, tablas de repositorios y flujos de trabajo) y enlazan a la sección correspondiente de la especificación para evitar divergencias.

---

## 1. Guías Prácticas Disponibles

| Guía Práctica | Objetivo Operativo | Sección de Referencia en la Spec |
|---|---|---|
| [**architecture.md**](./architecture.md) | Diagrama de flujo de procesos (`cinitd`, `kernel.root`, `clauncher`) e IPC. | [Sección 3.1](./CosmicOS-spec.md#31-kernelroot), [Sección 4](./CosmicOS-spec.md#4-modelo-de-aislamiento-de-aplicaciones) |
| [**repositories.md**](./repositories.md) | Catálogo de repositorios OCP, grupos de `repo` y convención de nombres (`c<nombre>d`, `cosmic-*`). | [Sección 7.5](./CosmicOS-spec.md#75-nombres-de-los-componentes), [Sección 8.5](./CosmicOS-spec.md#85-obtencion-del-codigo-fuente-repo-y-manifiesto-xml) |
| [**building.md**](./building.md) | Comandos para sincronizar con `repo`, compilar Cargo Workspace (Hito 1) y orquestador Go. | [Sección 7.3](./CosmicOS-spec.md#73-pruebas-ci-y-validacion-del-sandbox), [Sección 13](./CosmicOS-spec.md#13-hoja-de-ruta-propuesta) |
| [**kernel.md**](./kernel.md) | Manual del script `purge-arch.sh`, uso de Kconfig/defconfig y compilación para ARM64/RISC-V. | [Sección 3](./CosmicOS-spec.md#3-kernel) |
| [**sps-and-security.md**](./sps-and-security.md) | Guía rápida de empaquetado `.sps`, formato `Manifest.json` y verificación de sandbox. | [Sección 6](./CosmicOS-spec.md#6-sps-y-el-formato-sps) |
| [**v0.2.8-pending-changes.md**](./v0.2.8-pending-changes.md) | Borrador de cambios pendientes y registro de decisiones para la siguiente versión de la spec. | — |

---

## 2. Canales de Estabilidad

CosmicOS define cinco canales en su ciclo de vida y tags (ver [Sección 20.3 de la Spec](./CosmicOS-spec.md#203-tags-canales-de-estabilidad)):
- **EventHorizon:** Experimental.
- **Quasar:** Rolling continuo.
- **Pulsar:** Semiestable / Beta.
- **Nebula:** Canal de producción estable predeterminado.
- **Singularity:** Soporte extendido a largo plazo (LTS).
