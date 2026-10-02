# Reglas de trabajo

> **Tipo:** Convención | **Estado:** Estable | **Fecha:** 2026-10-02

Reglas que el asistente sigue en todos los proyectos. Documento fuente:
[`docs/REGLAS-DE-TRABAJO.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/REGLAS-DE-TRABAJO.md).

## 1. Flujo tras cada cambio de código

1. Compilar y validar (`pio run` → SUCCESS).
2. Bump semver (`feat`→minor, `fix`→patch, `BREAKING`→major).
3. Actualizar `firmware_manifest.json` (SHA-256 real).
4. Actualizar `CHANGELOG.md`.
5. Commit convencional (un cambio lógico = un commit).
6. Tag + push.
7. Release en GitHub con artefactos.
8. Actualizar wiki del proyecto + repo Docs.

## 2. Documentación

- Seguir el [Estándar de documentación](Estandar-de-documentacion.md).
- Mantener `README.md`, `CHANGELOG.md`, `IMPLEMENTACION.md`, `MEJORAS.md`.
- Verificar `mkdocs build` local antes de pushear al Docs.

## 3. Código

- Una clase por archivo (`.hpp`/`.cpp`).
- Compilar antes de entregar.
- No inventar APIs; verificar con el código real.
- No hardcodear secretos.

## 4. Git

- Conventional Commits (`feat`, `fix`, `docs`, `chore`, ...).
- Confirmar acciones destructivas.

## 5. Comunicación

- Reporte conciso (qué se hizo / validó / falta).
- Ser honesto ante bloqueos.
