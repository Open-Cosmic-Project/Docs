# Guía Práctica de SPS y Empaquetado

> **Fuente de verdad:** La especificación completa de paquetes `.sps`, el esquema de `Manifest.json` y el modelo de aislamiento residen en [CosmicOS-spec.md#6-sps-y-el-formato-sps](./CosmicOS-spec.md#6-sps-y-el-formato-sps) y [Sección 4](./CosmicOS-spec.md#4-modelo-de-aislamiento-de-aplicaciones).

---

## 1. Creación de un Paquete `.sps`

El comando oficial para construir un paquete desde la máquina del desarrollador es:

```bash
sps --create-package <nombre-del-paquete> <directorio-de-origen>
```

SPS genera una imagen EROFS firmada que contiene el binario nativo (Rust/C/C++) y el archivo `Manifest.json`.

---

## 2. Plantilla de `Manifest.json`

Conforme a la Sección 6.3 de la especificación:
- El campo `app_type` lo asigna exclusivamente el repositorio tras revisión (etiquetas sensibles como `system`, `bancario`, etc.).
- Las dependencias de librerías llevan rango de versión/ABI.

```json
{
  "header": {
    "name": "Cosmic Camera",
    "developer": "OpenCosmicProject",
    "repository_url": "https://github.com/OpenCosmicProject/camera",
    "signing_key_id": "0xABC12345DEF67890",
    "language": "Rust",
    "framework": "Qt6",
    "schema_version": 1,
    "target_abi": "0.1.0"
  },
  "body": {
    "functionality": "Captura y procesamiento fotográfico con libcamera",
    "app_type": "system",
    "app_class": "camera",
    "system_dependencies": [
      "libcamera.so.0.3 (>= 0.3.0, < 0.4.0)",
      "libpipewire-0.3.so.0 (>= 0.3.0)"
    ]
  },
  "tail": {
    "declared_permissions": [
      "camera.capture",
      "storage.photos.write"
    ],
    "binary_size_bytes": 1048576,
    "origin_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "hash_algorithm": "sha256"
  }
}
```

---

## 3. Comprobación del Sandbox en Desarrollo

- **Directorios de usuario:** La aplicación únicamente puede escribir en `~/.var/capps/<app-id>/{cache,data,config}`.
- **Acceso a hardware:** Se realiza mediante portales (`cbrokerd`), nunca por acceso directo a `/dev/video*`.
- **Ejecución manual de prueba:**
  ```bash
  cargo run -p clauncher -- /ruta/a/Manifest.json
  ```
