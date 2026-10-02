# Estándar de documentación

> **Tipo:** Convención | **Estado:** Estable | **Fecha:** 2026-10-02

Estándar reutilizable para documentar cualquier proyecto. Documento fuente:
[`docs/ESTANDAR-DOCUMENTACION.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/ESTANDAR-DOCUMENTACION.md).

## Estructura mínima de la wiki

`Home` · `Arquitectura` · `Hardware-y-Conexiones` · `Decisiones` (ADR) ·
`Registro-de-cambios` · `Configuracion` · `API` · `Compilacion` · `Glosario` ·
`Evolucion` · `Mejoras`.

## Conexiones (hardware)

Cada componente con ficha:

```text
Interfaz · pines/dirección · resistencias (pull-up/divisor) ·
capacitores (desacople) · tensión · protecciones · estado seguro
```

## Registro de cambios por archivo

```markdown
### `ruta/al/archivo`
- **Qué:** …  ·  **Por qué:** …  ·  **Cómo:** …  ·  **Impacto:** …  ·  **Referencia:** …
```

## Publicación

- **Wiki nativa GitHub** (por repo).
- **Docs-as-code**: MkDocs Material + GitHub Pages.
- **Repo compartido** con carpeta por proyecto (este repo).
