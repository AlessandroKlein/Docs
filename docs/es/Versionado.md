# Versionado (cómo numerar las versiones)

> **Tipo:** Convención | **Estado:** Estable | **Fecha:** 2026-10-02

Guía completa de la nomenclatura de versiones del ecosistema: firmware, servidor,
frontend, hardware, configuración y las **fases del proyecto** (`V8`, `V9`).

---

## 1. Regla base: Semantic Versioning 2.0.0

Toda versión del proyecto sigue **SemVer**:

```text
MAJOR.MINOR.PATCH[-preRelease][+buildMeta]

  3   .   29   .   0    -beta.1     +build.7
  │        │        │       │            └── metadatos de build (no afectan orden)
  │        │        │       └── pre-release: alpha / beta / rc
  │        │        └── PATCH: corrección de errores
  │        └── MINOR: funcionalidad nueva (retrocompatible)
  └── MAJOR: cambio incompatible
```

**Se incrementa solo el número que corresponde; los de la derecha vuelven a 0.**

```text
3.28.5 + feat  → 3.29.0   (MINOR sube, PATCH vuelve a 0)
3.28.5 + fix   → 3.28.6   (PATCH sube)
3.28.5 + break → 4.0.0    (MAJOR sube, el resto vuelve a 0)
```

## 2. El prefijo `v` — regla clave

| Contexto | Formato | Ejemplo |
|----------|---------|---------|
| **Etiqueta (tag) de Git** | `v` + versión | `v3.29.0` |
| **Release de GitHub** | `v` + versión | `v3.29.0` |
| **Variable del firmware** | **sin** `v` | `GH_FW_VERSION "3.29.0"` |
| **Manifest JSON** | **sin** `v` | `"version": "3.29.0"` |
| **CHANGELOG** | `[` versión `]` sin `v` | `## [3.29.0]` |
| **README / docs** | se usa `v` al hablar del release | "Avance v3.29.0" |

!!! warning "Siempre minúscula"
    La convención adoptada es **`v` minúscula**. Históricamente se usó `V`
    mayúscula (`V1.16.0`, `V2.30`) y por eso hay etiquetas mezcladas (ver §9).
    Los tags nuevos van **siempre en minúscula**.

## 3. ¿Qué número toca? (tabla de decisión)

| Cambio | Número | Ejemplos del proyecto |
|--------|:------:|----------------------|
| Rompe compatibilidad (config, API, protocolo) | **MAJOR** | subir `GH_CONFIG_SCHEMA_VERSION`; renombrar/quitar un endpoint |
| Funcionalidad nueva retrocompatible | **MINOR** | `feat(...)`: nuevo endpoint, nuevo pool, nueva página web |
| Corrección de errores | **PATCH** | `fix(...)`: bug de lectura, timeout mal calculado |
| Solo documentación / comentarios | **ninguno** | un commit `docs:` no genera release |
| Refactor sin cambio de comportamiento | **ninguno** o PATCH | depende de si el usuario nota algo |

### Cuándo NO bumpear

- Cambios solo en `docs/`, comentarios, README sin cambio funcional.
- Reordenar código, renombrar variables internas, formateo.
- Subir la versión solo porque hubo un commit.

## 4. Pre-releases: `alpha`, `beta` (`b`) y `rc`

Cuando la versión **no está lista para uso general**:

| Tipo | Notación | Uso |
|------|----------|-----|
| Alfa | `3.30.0-alpha.1` | Experimental, puede romper |
| Beta | `3.30.0-beta.1` | Funcionalmente completa, en pruebas |
| Beta corta | `3.30.0-b1` | Abreviatura aceptada de `beta.1` |
| Release candidate | `3.30.0-rc.1` | Candidata final, solo correcciones |

**Orden de precedencia** (de menor a mayor):

```text
3.30.0-alpha.1 < 3.30.0-alpha.2 < 3.30.0-beta.1 < 3.30.0-rc.1 < 3.30.0
```

> Una pre-release **siempre es menor** que su versión final. `3.30.0-beta.1` < `3.30.0`.

### La `b` en este proyecto

La `b` se interpreta como **beta** (`3.30.0-b1` = `3.30.0-beta.1`). En la industria
también se usa `b` como *build* (`b7`), pero **en este proyecto `b` = beta**; para
build se usa el metadato `+build.N` (§5).

## 5. Metadatos de build (`+`)

No cambian el orden de versiones; sirven para trazabilidad:

```text
3.30.0+build.42
3.30.0+sha.1a2b3c4
3.30.0-beta.1+sha.1a2b3c4
```

El **SHA-256 del binario** no va en la versión: va en `firmware_manifest.json`
(campo `sha256`) y en las notas del release.

## 6. Canales de actualización

El firmware tiene un canal (`update_channel`), que se combina con el tipo de versión:

| Canal | Acepta | Uso |
|-------|--------|-----|
| `stable` | `X.Y.Z` | Producción |
| `beta` | `X.Y.Z` y `-beta.N` | Pruebas |
| `development` | todo, incluido `-alpha.N` | Desarrollo |

```text
stable     ← solo versiones finales
beta       ← finales + betas
development ← todo
```

## 7. Otras versiones del proyecto (¡no confundir!)

El firmware lleva **cuatro** números distintos. No son intercambiables:

| Constante | Ejemplo | Qué versiona | Cuándo sube |
|-----------|---------|--------------|-------------|
| `GH_FW_VERSION` | `3.29.0` | Firmware | Cada release |
| `GH_HW_VERSION` | `rev0` | Revisión del hardware | Cambio de placa |
| `GH_CONFIG_SCHEMA_VERSION` | `2` | Estructura del JSON de config | Cambio incompatible de config |
| `GH_PROTOCOL_VERSION` | `1` | Protocolo con el servidor | Cambio incompatible de protocolo |

### Versión de hardware: `revN`

```text
rev0  → primera revisión del PCB
rev1  → corrección de una pista / componente
rev2  → rediseño mayor
```

Los enteros **no** vuelven a cero: es acumulativa.

### Esquema de configuración: entero incremental

```text
schema 1 → estructura original
schema 2 → configuración por capas / net_interface / eth_*   (subida en v3.14.0)
schema 3 → (cuando haya un cambio incompatible de config)
```

Al subir el esquema hay que agregar la migración en
`ConfigManager::migrate()`.

## 8. Versiones por subsistema (prefijos)

Cada artefacto del ecosistema se versiona **de forma independiente** con un prefijo:

| Artefacto | Etiqueta | Ejemplos reales |
|-----------|----------|-----------------|
| Firmware del invernadero | `vX.Y.Z` | `v3.29.0` |
| Servidor central | `server-vX.Y.Z` | `server-v0.1.0` … `server-v0.5.0` |
| Frontend (preview) | `frontend-preview-vX.Y.Z` | `frontend-preview-v1.0.0` … `v1.1.0` |
| Documentación | `vX.Y.Z` (repo Docs) | `v1.0.0` |

> Así el servidor puede ir en `0.x` (inestable) mientras el firmware va en `3.x`,
> sin confundir al usuario sobre cuál se actualizó.

### `0.y.z` = desarrollo inicial

Mientras el número **MAJOR sea `0`**, la API puede cambiar en cualquier momento
(es lo que indica SemVer). El servidor está en `0.5.0`; el firmware ya en `3.x`
(estable).

## 9. Las fases del proyecto (`V8`, `V8.1`, `V9`) NO son versiones

Cuidado con la ambigüedad: el proyecto tiene **fases de arquitectura** en
mayúscula, que **no** son versiones de firmware:

| Fase | Significado | Versión de firmware donde se entregó |
|------|-------------|--------------------------------------|
| `V8` | Base de la plataforma configurable (registros, buses) | `3.8.0` |
| `V8.1` | SpiManager, 74HC165, MCP23S17, ADC | `3.10.0` |
| `V8.4` | StorageManager (LittleFS/SPIFFS/SD) | `3.10.0` |
| `V9` | Perfiles Modbus, capa CAN, gateway RS485 | `3.10.0` / `3.17.0` |
| `V10` | (futuro) | — |

!!! tip "Cómo distinguirlas"
    - **Mayúscula + sin prefijo `v`** → fase del proyecto (`V8`, `V9`).
    - **Minúscula + prefijo `v`** → etiqueta de release (`v3.29.0`).

## 10. Historial real de etiquetas (y sus inconsistencias)

Las etiquetas existentes en el repositorio, para referencia histórica:

| Etiqueta | Observación |
|----------|-------------|
| `V1.3.0`, `V1.4.0`, `V1.16.0` | **Mayúscula**; convención antigua |
| `V2.00`, `V2.30` | Mayúscula y con **cero relleno** (`V2.00`) — no es SemVer |
| `v3.0.0` … `v3.29.0` | Convención actual (minúscula, SemVer) |
| **`v3.8.0` faltante** | El CHANGELOG tiene la entrada 3.8.0 pero **no se creó el tag** |
| `server-v0.x`, `frontend-preview-v1.x` | Versionado por subsistema |

**Reglas adoptadas a partir de ahora:**

- ✅ Siempre `v` **minúscula** y SemVer completo (`v3.30.0`).
- ❌ No usar cero relleno (`2.00` → `2.0.0`).
- ❌ No saltear números: si existe la entrada en el CHANGELOG, **crear el tag**.
- ✅ Prefijo de subsistema solo para artefactos distintos (`server-v…`).

## 11. Flujo de release (paso a paso)

```text
1. Decidir MAJOR/MINOR/PATCH según §3
2. Editar include/core/Version.hpp  →  #define GH_FW_VERSION "X.Y.Z"
3. Actualizar CHANGELOG.md          →  ## [X.Y.Z] - AAAA-MM-DD
4. pio run                           →  verificar SUCCESS
5. Calcular SHA-256 del firmware.bin →  firmware_manifest.json
6. git commit -m "feat(...): ..."    →  Conventional Commits
7. git tag -a vX.Y.Z -m "Release vX.Y.Z: <resumen>"
8. git push origin main --tags
9. gh release create vX.Y.Z --title "vX.Y.Z — <título>" --notes "..." firmware.bin
10. Actualizar wiki del proyecto + repo Docs
```

## 12. Ejemplos prácticos

| Situación | Versión |
|-----------|---------|
| Se agregó el endpoint `/api/v1/modbus/gateway` | `3.17.0` (MINOR) |
| Se corrigió que el OTA no cerrara el socket | `3.17.1` (PATCH) |
| Se cambió el formato del JSON de configuración | `4.0.0` (MAJOR) |
| Beta del próximo release con nueva API | `3.30.0-beta.1` |
| Candidata final de esa beta | `3.30.0-rc.1` |
| Release final | `3.30.0` |
| Se actualizó solo la documentación | sin release |

## 13. Glosario de notación

| Símbolo | Significado |
|---------|-------------|
| `X.Y.Z` | `MAJOR.MINOR.PATCH` |
| `v` | Prefijo de etiqueta de Git/release (minúscula) |
| `V8`, `V9` | Fase del proyecto (mayúscula, no es versión) |
| `-alpha.N` | Pre-release experimental |
| `-beta.N` / `-bN` | Pre-release en pruebas |
| `-rc.N` | Release candidate |
| `+build.N`, `+sha.xxxx` | Metadatos de build (no afectan el orden) |
| `revN` | Revisión de hardware |
| `schema N` | Versión del esquema de configuración |
| `0.y.z` | Desarrollo inicial (API inestable) |

Ver también: [Reglas de trabajo](Reglas-de-trabajo.md) ·
[CHANGELOG](invernadero/CHANGELOG.md) · [Mejoras](invernadero/Mejoras.md) ·
[Evolución](invernadero/Evolucion.md).
