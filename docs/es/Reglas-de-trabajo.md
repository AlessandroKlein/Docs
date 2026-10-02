# Reglas de trabajo

> **Tipo:** Convención | **Estado:** Estable | **Fecha:** 2026-10-02

Reglas que el asistente sigue en **todos** los proyectos. Documento fuente:
[`docs/REGLAS-DE-TRABAJO.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/REGLAS-DE-TRABAJO.md).

Prioridad: estas reglas mandan sobre cualquier petición que entre en conflicto
(excepto que el usuario pida lo contrario explícitamente).

---

## 1. Flujo obligatorio tras cada cambio de código

En este orden, **sin omitir pasos**:

| # | Paso | Verificación |
|---|------|--------------|
| 1 | Compilar y validar | `pio run` → **SUCCESS** |
| 2 | Bump de versión | ver [Versionado](Versionado.md) |
| 3 | Actualizar `firmware_manifest.json` | **SHA-256 real** del `.bin` |
| 4 | Actualizar `CHANGELOG.md` | formato Keep a Changelog |
| 5 | Commit convencional | un cambio lógico = un commit |
| 6 | Tag + push | `v` minúscula, SemVer completo |
| 7 | Release en GitHub | con artefactos adjuntos |
| 8 | **Actualizar documentación** | wiki del proyecto **+ repo Docs** |
| 9 | Verificar el sitio | `python -m mkdocs build` → SUCCESS |

> **Regla de oro:** no se da por terminada una tarea de código sin
> **release + documentación actualizados**.

---

## 2. Documentación obligatoria en el repo Docs

El repositorio `AlessandroKlein/Docs` es la **documentación unificada** de todos
los proyectos. Se actualiza en **cada** ciclo de cambios, no "cuando haya tiempo".

### 2.1 Documentos que se mantienen siempre

| Documento | Ubicación | Cuándo se actualiza |
|-----------|-----------|---------------------|
| `README.md` | repo del código | Cada release (estado, versión, características) |
| `CHANGELOG.md` | repo del código **y** Docs | Cada release |
| `docs/MEJORAS.md` | repo del código **y** Docs | Cuando cambia el estado de un ítem |
| `docs/IMPLEMENTACION.md` | repo del código | Cuando cambia la arquitectura |
| `docs/DUDAS-Y-DECISIONES.md` | repo del código | Con cada decisión nueva |
| `docs/ESTANDAR-DOCUMENTACION.md` | repo del código | Si cambia el estándar |
| `docs/REGLAS-DE-TRABAJO.md` | repo del código | Si cambia una regla |
| **Wiki del proyecto** | `Invernadero.wiki` | Cada release |
| **Repo `Docs`** | este repositorio | Cada release |

### 2.2 Mapa: qué cambió → qué página actualizar

Esta tabla es **obligatoria**: si tocás el código, buscá la fila y actualizá la
página correspondiente.

| Cambio en el código | Página del Docs a actualizar |
|---------------------|------------------------------|
| **Nuevo endpoint REST** | `API-REST.md` + `Referencia-de-codigo.md` |
| **Endpoint eliminado/renombrado** | idem + revisar `Versionado.md` (¿MAJOR?) |
| **Cambio en MQTT/WebSocket** | `MQTT-y-WebSocket.md` |
| **Campo nuevo de configuración** | `Referencia-configuracion.md` + `Variables-Modificables.md` |
| **Cambio de `schema_version`** | `Referencia-configuracion.md` + `Versionado.md` |
| **Sensor nuevo** | `Sensores.md` + `Compatibilidad.md` + `Referencia-de-codigo.md` |
| **Actuador/rol nuevo** | `Actuadores-y-Salidas.md` |
| **Pin nuevo o cambiado** | `Guia-de-pines.md` + `Hardware-y-Conexiones.md` |
| **Ficha de conexión nueva** | `Hardware-y-Conexiones.md` |
| **Clase/módulo nuevo** | `Referencia-API-interna.md` + `Referencia-de-codigo.md` |
| **Enum o valor nuevo** | `Enumeraciones-y-tipos.md` |
| **Nueva limitación o bloqueo** | `Mejoras.md` (+ la página afectada) |
| **Release nuevo** | `CHANGELOG.md` + `Estadisticas.md` + `Mejoras.md` + `Evolucion.md` |
| **Decisión de arquitectura** | `Decisiones.md` (ADR) |
| **Cambio de UI/frontend** | `Frontend.md` |
| **Cambio en OTA** | `OTA-y-Actualizacion.md` |
| **Cambio en seguridad/auth** | `Seguridad.md` |
| **Diagrama nuevo** | `Diagramas.md` (Mermaid) |
| **Convención nueva** | `Reglas-de-trabajo.md` / `Versionado.md` / `Estandar-de-documentacion.md` |
| **Cualquier cambio de comportamiento** | `Registro-de-cambios.md` (ficha por archivo) |

### 2.3 Reglas de contenido

1. **Seguir el [Estándar de documentación](Estandar-de-documentacion.md)**:
   frontmatter con tags, cabecera (Tipo/Estado/Fecha), secciones numeradas y
   cierre con enlaces.
2. **Verificar cada dato contra el código** antes de escribirlo (defaults, pines,
   endpoints, enums). **Nunca inventar.**
3. **Tablas completas**: si es una referencia, no debe faltar ninguna entrada ni
   usar "etc.".
4. **Citar la versión** a la que aplica el dato (p. ej. "v3.29.0").
5. **Documentar también lo que falta**: los bloqueos y pendientes van a
   `Mejoras.md` con el motivo y la vía de solución.
6. **Sincronizar**: si el documento existe en el repo del código (`docs/…`) y en
   el Docs, se actualizan **los dos** en el mismo ciclo.

### 2.4 Navegación e idioma

- Toda **página nueva** se agrega al `nav` de `mkdocs.yml` (si no, queda huérfana).
- El orden del `nav` es lógico: empezar → referencia → hardware → uso → operación
  → soporte → proyecto.
- Contenido en `docs/es/`; las traducciones van a `docs/en/` con **la misma ruta**.
- Si no hay traducción, `fallback_to_default` muestra el español (no rompe nada).

### 2.5 Verificación antes de pushear

```bash
cd <repo Docs>
python -m mkdocs build     # debe terminar en SUCCESS
```

Comprobar que:

- No haya errores ni *warnings* nuevos.
- No queden páginas fuera del `nav` (salvo las `Home.md` huérfanas conocidas).
- Los enlaces internos entre páginas resuelvan.

Recién después: `git add` → `git commit -m "docs: …"` → `git push origin main`.
El workflow publica automáticamente en la rama `gh-pages` (GitHub Pages).

!!! warning "No dejar la documentación para después"
    El sitio tarda unos minutos en actualizarse tras el push (gh-pages). Si algo
    parece "no actualizado", primero verificá el **archivo en el repositorio**
    (`git pull` / vista de GitHub) antes de asumir que falta el cambio.

### 2.6 Qué NO hacer

| ❌ No hacer | ✅ Hacer |
|-------------|---------|
| Documentar de memoria o "de lo que me acuerdo" | Leer el fuente y copiar el dato exacto |
| Poner "etc." en una tabla de referencia | Completar todas las entradas |
| Actualizar solo la wiki y no el Docs (o al revés) | Ambos, en el mismo ciclo |
| Crear una página sin agregarla al `nav` | Agregarla siempre |
| Pushear al Docs sin compilar | `mkdocs build` primero |
| Duplicar la misma info en dos páginas | Una página + enlace desde la otra |
| Dejar el dato sin versión | Citar la versión aplicable |

---

## 3. Código

- **Una clase por archivo**: `.hpp` en `include/<módulo>/`, `.cpp` en `src/<módulo>/`.
- **Compilar antes de entregar**; no entregar código roto.
- **No inventar APIs, librerías, flags ni comandos**: verificar contra el código
  real o la documentación oficial. Ante la duda, preguntar.
- **No hardcodear secretos** (tokens, claves): usar NVS o `.env`.
- **No refactorizar** código que no se está tocando (salvo limpieza local < 10 líneas).
- Comentar el **por qué**, no el **qué**. Usar `// TODO:` / `// FIXME:` con fecha.

---

## 4. Git

- **Conventional Commits** en inglés: `feat`, `fix`, `docs`, `refactor`, `test`,
  `chore`, `style`, `perf`, `ci`.
- Un commit por cambio lógico; no mezclar funcionalidades independientes.
- No usar `push --force` en ramas protegidas sin confirmación.
- **Acciones destructivas** (borrar archivos/ramas, mover recursos) → pedir
  confirmación y verificar la ruta absoluta antes de ejecutar.

---

## 5. Comunicación

- Reporte **conciso**: qué se hizo, qué se validó, qué falta.
- **Ser honesto ante bloqueos**: detenerse y explicar, no improvisar una solución
  que no funciona.
- No repetir innecesariamente lo ya hecho; un resumen breve al final basta.
- Avisar cuando un dato **no se pudo verificar** en lugar de asumirlo.

---

## 6. Convenciones relacionadas

| Documento | Qué define |
|-----------|------------|
| [Versionado](Versionado.md) | Numeración, `v`, `b`, `rc`, `rev`, `schema`, MAJOR |
| [Estándar de documentación](Estandar-de-documentacion.md) | Cómo se escribe cada página |
| [Mejoras](invernadero/Mejoras.md) | Roadmap y pendientes |
| [CHANGELOG](invernadero/CHANGELOG.md) | Historial de versiones |

---

## 7. Checklist de un ciclo completo

- [ ] Código compila (`pio run` → SUCCESS).
- [ ] Versión bumpeada según [Versionado](Versionado.md).
- [ ] `firmware_manifest.json` con SHA-256 real.
- [ ] `CHANGELOG.md` actualizado.
- [ ] Commit convencional + tag `vX.Y.Z` + push.
- [ ] Release de GitHub con artefactos.
- [ ] `README.md` actualizado.
- [ ] Páginas del Docs actualizadas según §2.2.
- [ ] `Registro-de-cambios.md` con la ficha por archivo.
- [ ] `mkdocs build` en SUCCESS.
- [ ] Wiki del proyecto actualizada.
- [ ] Push a `AlessandroKlein/Docs`.
