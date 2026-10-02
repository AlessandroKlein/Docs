# Docs — Documentación unificada de AlessandroKlein

Repositorio único de documentación para todos los proyectos, siguiendo el estándar
[`ESTANDAR-DOCUMENTACION.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/ESTANDAR-DOCUMENTACION.md).

## Estructura

```text
docs/
├── index.md            ← índice global
├── invernadero/        ← carpeta por proyecto
│   ├── Home.md
│   ├── Arquitectura.md
│   ├── Hardware-y-Conexiones.md
│   ├── Decisiones.md
│   └── Registro-de-cambios.md
├── sema/
│   └── Home.md
└── (futuros proyectos)/
```

## Cómo funciona

- La documentación se escribe en **Markdown** dentro de `docs/<proyecto>/`.
- **MkDocs Material** la compila en un sitio estático con búsqueda global.
- Un **GitHub Action** publica el sitio en **GitHub Pages** en cada push.

## Desarrollo local

```bash
pip install mkdocs-material
mkdocs serve          # http://127.0.0.1:8000
```

## Publicación

El workflow `.github/workflows/deploy.yml` hace `mkdocs build` y despliega a
GitHub Pages automáticamente al pushear a `main`.

> Reglas: una página por tema · cabecera `> Tipo | Estado | Fecha` · documentar
> conexiones con resistencias/capacitores · registrar cambios por archivo
> (qué/por qué/cómo/impacto).
