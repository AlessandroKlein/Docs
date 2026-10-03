# AlessandroKlein · Documentación

Documentación unificada de los proyectos. Seleccioná un proyecto en el menú.

## Proyectos

| Proyecto | Descripción |
|----------|-------------|
| [Invernadero](invernadero/Home.md) | Plataforma modular de automatización de invernadero (ESP32) |
| [SEMA](sema/Home.md) | Sistema de Estación Meteorológica Autónoma |

---

## Archivos de arranque (se copian al crear cada proyecto)

Antes de escribir código, todo proyecto nuevo recibe estos **5 archivos de
convención**. Cada uno tiene su **ruta exacta** y su **contenido completo
copiable** (o comando de descarga). Son la base sobre la que se trabaja.

| # | Archivo | Ruta en el proyecto | Para qué |
|:-:|---------|---------------------|----------|
| 1 | [Reglas de trabajo](inicio/Reglas-de-trabajo.md) | `docs/REGLAS-DE-TRABAJO.md` | Flujo tras cada cambio: build → release → docs; código, git, comunicación |
| 2 | [Estándar de documentación](inicio/Estandar-de-documentacion.md) | `docs/ESTANDAR-DOCUMENTACION.md` | Cómo se escribe cada página de la wiki |
| 3 | [Versionado](inicio/Versionado.md) | `docs/VERSIONADO.md` | SemVer, prefijo `v`, `b`/beta, `rc`, cuándo MAJOR, `rev`, `schema` |
| 4 | [Design System](inicio/Design-system.md) | `DESIGN-SYSTEM.md` (raíz) | Identidad visual + arquitectura modular (módulos, bloques, paleta) |
| 5 | [Seguridad](inicio/Security.md) | `SECURITY.md` (raíz) | Política de seguridad y reporte de vulnerabilidades |

### Cómo obtenerlos

```bash
# desde la raíz del proyecto nuevo
mkdir -p docs
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/docs/REGLAS-DE-TRABAJO.md -o docs/REGLAS-DE-TRABAJO.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/docs/ESTANDAR-DOCUMENTACION.md -o docs/ESTANDAR-DOCUMENTACION.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/docs/VERSIONADO.md -o docs/VERSIONADO.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/DESIGN-SYSTEM.md -o DESIGN-SYSTEM.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/SECURITY.md -o SECURITY.md
```

O copiá el contenido de la sección "Contenido completo" de cada plantilla.

> Los tres primeros van dentro de `docs/`; `DESIGN-SYSTEM.md` y `SECURITY.md` van
> en la **raíz** del repositorio (así los detecta GitHub).

### Estructura resultante

```text
proyecto/
├── SECURITY.md                  ← política de seguridad
├── DESIGN-SYSTEM.md             ← diseño + arquitectura
├── README.md                    ← qué es y cómo usarlo
├── CHANGELOG.md                 ← historial de versiones
└── docs/
    ├── REGLAS-DE-TRABAJO.md
    ├── VERSIONADO.md
    └── ESTANDAR-DOCUMENTACION.md
```
