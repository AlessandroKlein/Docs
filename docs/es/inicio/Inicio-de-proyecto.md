# Inicio de proyecto (archivos de arranque)

> **Tipo:** Convención | **Estado:** Estable | **Fecha:** 2026-10-03

Conjunto de archivos de **convención** que se copian a un proyecto nuevo, en la
**primera** sesión, antes de escribir código. Definen cómo se trabaja, cómo se
versiona, cómo se documenta, cómo se diseña y cómo se reporta seguridad.

Cada plantilla tiene su **ruta exacta** y el **contenido completo copiable**.

---

## 1. Archivos a crear

| # | Archivo | Ruta en el proyecto | Para qué | Plantilla |
|:-:|---------|---------------------|----------|-----------|
| 1 | **Reglas de trabajo** | `docs/REGLAS-DE-TRABAJO.md` | Flujo tras cada cambio (build → release → docs), código, git, comunicación | [Abrir](Reglas-de-trabajo.md) |
| 2 | **Estándar de documentación** | `docs/ESTANDAR-DOCUMENTACION.md` | Cómo se escribe cada página de la wiki | [Abrir](Estandar-de-documentacion.md) |
| 3 | **Versionado** | `docs/VERSIONADO.md` | SemVer, `v`, `b`/beta, `rc`, cuándo MAJOR, `rev`, `schema` | [Abrir](Versionado.md) |
| 4 | **Design System** | `DESIGN-SYSTEM.md` | Identidad visual + arquitectura modular (módulos, bloques, paleta) | [Abrir](Design-system.md) |
| 5 | **Seguridad** | `SECURITY.md` | Política de seguridad y reporte de vulnerabilidades | [Abrir](Security.md) |

> Los cuatro primeros van dentro de `docs/`; `DESIGN-SYSTEM.md` y `SECURITY.md`
> van en la **raíz** del repositorio (así los detecta GitHub).

---

## 2. Orden recomendado

```text
1. SECURITY.md                      (raíz)   — primero, define el estado del repo
2. DESIGN-SYSTEM.md                 (raíz)   — antes de escribir UI o código
3. docs/VERSIONADO.md                        — antes del primer release
4. docs/REGLAS-DE-TRABAJO.md                 — antes del primer commit de código
5. docs/ESTANDAR-DOCUMENTACION.md            — antes de crear la wiki
```

---

## 3. Estructura resultante

```text
proyecto/
├── SECURITY.md                      ← política de seguridad
├── DESIGN-SYSTEM.md                 ← diseño + arquitectura
├── README.md                        ← qué es y cómo usarlo
├── CHANGELOG.md                     ← historial de versiones
├── .github/workflows/deploy.yml     ← CI (publica la wiki)
└── docs/
    ├── REGLAS-DE-TRABAJO.md         ← convenciones de trabajo
    ├── VERSIONADO.md                ← numeración de versiones
    ├── ESTANDAR-DOCUMENTACION.md    ← cómo documentar
    ├── MEJORAS.md                   ← roadmap y pendientes
    ├── IMPLEMENTACION.md            ← estado de la arquitectura
    └── DUDAS-Y-DECISIONES.md        ← decisiones (ADR)
```

---

## 4. Cómo usar las plantillas

Cada plantilla ofrece **dos formas**:

**A. Descarga directa** (recomendado; crea el archivo ya listo):

```bash
# desde la raíz del proyecto nuevo
mkdir -p docs
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/docs/REGLAS-DE-TRABAJO.md -o docs/REGLAS-DE-TRABAJO.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/docs/VERSIONADO.md -o docs/VERSIONADO.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/docs/ESTANDAR-DOCUMENTACION.md -o docs/ESTANDAR-DOCUMENTACION.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/DESIGN-SYSTEM.md -o DESIGN-SYSTEM.md
curl -fsSL https://raw.githubusercontent.com/AlessandroKlein/Invernadero/main/SECURITY.md -o SECURITY.md
```

**B. Copiar y pegar** el contenido de la sección 3 de cada plantilla en el
archivo correspondiente.

!!! tip "Ajustar al proyecto"
    Al copiarlos, revisá el nombre del proyecto y el correo de contacto en
    `SECURITY.md`; el resto del contenido es transversal y aplica tal cual.

---

## 5. Checklist de arranque

- [ ] `SECURITY.md` en la raíz.
- [ ] `DESIGN-SYSTEM.md` en la raíz.
- [ ] `docs/VERSIONADO.md` con la versión inicial definida (p. ej. `0.1.0`).
- [ ] `docs/REGLAS-DE-TRABAJO.md`.
- [ ] `docs/ESTANDAR-DOCUMENTACION.md`.
- [ ] `README.md` (qué es, cómo se compila, cómo se usa).
- [ ] `CHANGELOG.md` con la versión inicial.
- [ ] `docs/MEJORAS.md`, `docs/IMPLEMENTACION.md`, `docs/DUDAS-Y-DECISIONES.md`.
- [ ] Repositorio `Docs` con la carpeta del proyecto y el `nav` actualizado.
- [ ] Workflow de CI que publique la wiki.

Ver también: [Versionado](Versionado.md) · [Reglas de trabajo](Reglas-de-trabajo.md) ·
[Estándar de documentación](Estandar-de-documentacion.md).
