# Estándar de documentación

> **Tipo:** Convención | **Estado:** Estable | **Fecha:** 2026-10-02

Estándar **obligatorio** para documentar cualquier proyecto de este ecosistema.
Define qué se documenta, cómo se estructura cada página y qué plantillas usar.

Documento fuente en el código:
[`docs/ESTANDAR-DOCUMENTACION.md`](https://github.com/AlessandroKlein/Invernadero/blob/main/docs/ESTANDAR-DOCUMENTACION.md).

---

## 1. Principios

1. **Documentar hasta el detalle más pequeño.** Si alguien puede preguntarse
   "¿y esto?", debe estar en la wiki. No existe "es obvio".
2. **Verificar contra el código, nunca inventar.** Cada dato (default, pin,
   endpoint, enum) se lee del fuente real antes de escribirlo.
3. **Explicar el por qué, no solo el qué.** Una tabla dice *qué*; una frase dice
   *por qué* se hizo así.
4. **Una página = una pregunta.** Si la página responde a dos preguntas distintas,
   conviene dividirla.
5. **Todo dato tiene una fuente.** Al final, link a la página relacionada o al
   archivo del código.
6. **Documentación viva.** Se actualiza en el mismo commit que el código.

---

## 2. Anatomía de una página

Toda página tiene **obligatoriamente** estos elementos:

### 2.1 Frontmatter (tags)

```yaml
---
tags:
  - invernadero
  - hardware
---
```

El primer tag es el **proyecto**; los siguientes son **temáticas** (van al índice
de tags).

### 2.2 Cabecera de bloque

```markdown
> **Tipo:** Referencia | **Estado:** Estable | **Fecha:** AAAA-MM-DD
```

| Campo | Valores |
|-------|---------|
| **Tipo** | `Guía` · `Referencia` · `Concepto` · `API` · `Configuración` · `Soporte` · `Roadmap` · `Convención` |
| **Estado** | `Estable` · `En desarrollo` · `Especificación` · `Obsoleto` |
| **Fecha** | última revisión real |

### 2.3 Secciones numeradas

Los títulos usan `## 1.`, `## 2.`, … para poder citarlos ("ver §10"). Las
subsecciones usan `###`.

### 2.4 Cierre con enlaces

```markdown
Ver también: [Página A](A.md) · [Página B](B.md).
```

---

## 3. Tipos de página y plantillas

### 3.1 Guía (paso a paso)

Para "cómo hacer X". Numerar los pasos, cada uno con **una acción**.

```markdown
# Inicio rápido
> **Tipo:** Guía | **Estado:** Estable | **Fecha:** …

## 1. Conectar el hardware
1. Paso concreto.
2. Paso concreto.

## 2. Primer arranque
…
```

### 3.2 Referencia (tablas exhaustivas)

Para "todos los X". **Toda** entrada debe estar: si falta una, la página no sirve.

```markdown
# Referencia de configuración
| Clave | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| …     | …    | …       | …           |
```

### 3.3 Concepto (explicación técnica)

Para "cómo funciona X" (p. ej. PID). Incluir: qué es, diagrama, fórmula,
aplicación en el proyecto y cuándo **no** usarlo.

### 3.4 Soporte (diagnóstico)

Para resolver problemas. **Siempre** con el comando/endpoint que confirma el
diagnóstico.

```markdown
## N. Síntoma
1. Qué revisar (`GET /api/v1/…`).
2. Causa probable.
3. Solución.
```

### 3.5 Roadmap / estado

Tablas con ✅ / ⚠️ / ❌ y la **versión** donde se entregó cada ítem.

---

## 4. Contenido obligatorio por área

### 4.1 Ficha de conexión de hardware

**Cada componente** (sensor, expansor, driver) lleva una ficha con **todos** estos
campos:

| Campo | Ejemplo |
|-------|---------|
| Interfaz | `I²C, 0x44` |
| Pines / dirección | `SDA→GPIO21` · `SCL→GPIO22` |
| Tensión | `2,4–5,5 V (a 3,3 V)` |
| **Resistencias** | `pull-up 4,7 kΩ` · `divisor 1k/2k` |
| **Capacitores** | `100 nF de desacople` |
| **Protecciones** | `diodo flyback`, `optoacoplador`, `TVS`, `120 Ω terminación` |
| **Estado seguro** | qué hace ante fallo (p. ej. "nivel bajo → corta bomba") |

```markdown
### SHT31 — temperatura y humedad interior
| Elemento | Valor / Detalle |
|----------|-----------------|
| Interfaz | I²C, 0x44 |
| Tensión | 2,4–5,5 V (alimentar a 3,3 V) |
| Resistencia | pull-up 4,7 kΩ en SDA y SCL |
| Capacitor | 100 nF de desacople entre VDD y GND |
| Conexión | VDD→3,3 V · GND→GND · SDA→GPIO21 · SCL→GPIO22 |
| Estado seguro | No aplica (solo lectura) |
```

### 4.2 Referencia de pines

| Columna | Contenido |
|---------|-----------|
| Clave JSON | nombre editable |
| Default | valor de fábrica |
| Función | para qué sirve |
| Dirección | entrada / salida / bidi |
| Notas | restricciones y **conflictos** |

Además: restricciones del MCU (solo-entrada, strapping, conflictos de ADC) y
**conflictos entre valores por defecto**.

### 4.3 Referencia de API interna

Tabla **método → descripción** por clase, con la firma real:

```markdown
| Método | Descripción |
|--------|-------------|
| `void begin(cfg, pins, SensorRegistry*)` | Inicializa buses y drivers |
```

### 4.4 Enumeraciones y tipos

Tabla **valor → código numérico → significado**. Documentar **todos** los valores
(incluidos `NONE`/`UNKNOWN`).

### 4.5 Referencia de código

Mapa módulo → archivos → responsabilidad → **métricas** (líneas). Incluir: árbol
del repo, flujo de arranque, tareas y puntos de extensión.

### 4.6 Endpoints REST

Agrupados por área, con método, ruta, descripción y **si requiere auth**. Añadir
ejemplos `curl` de los casos importantes.

---

## 5. Registro de cambios por archivo

**Cada archivo modificado** lleva una ficha:

```markdown
### `ruta/al/archivo` (vX.Y.Z)
- **Qué:** qué cambió.
- **Por qué:** motivación.
- **Cómo:** implementación.
- **Impacto:** qué se ve afectado.
- **Referencia:** vX.Y.Z.
```

> Se actualiza en el **mismo commit** que el cambio.

---

## 6. Diagramas (Mermaid)

Usar Mermaid (habilitado en el sitio) para:

| Tipo | Cuándo |
|------|--------|
| `flowchart` | Arquitectura, flujo de datos, lazo de control |
| `sequenceDiagram` | Diálogo entre componentes (sensor → ESP32 → MQTT → DB) |
| `stateDiagram-v2` | Máquinas de estado (provisioning, dispositivo) |

Reglas: nodos con **nombres del código** real; un diagrama = una idea.

---

## 7. Nomenclatura de archivos

| Regla | Ejemplo |
|-------|---------|
| PascalCase con guiones | `Referencia-API-interna.md` |
| Sin espacios ni acentos | `Guia-de-pines.md` (no `Guía de pines.md`) |
| Nombre = tema de la página | `Versionado.md` |
| Un archivo por página | — |

---

## 8. Navegación y tags

- **`mkdocs.yml` → `nav`**: orden lógico (empezar → referencia → hardware → uso →
  operación → soporte → proyecto).
- **Tags**: primer tag = proyecto; luego temáticas (`hardware`, `api`,
  `configuracion`, `sensores`, `soporte`, `referencia`, `desarrollo`…).
- Un tag nuevo solo si no existe uno que sirva.

---

## 9. Multi-idioma

- Contenido en `docs/es/` (default) y `docs/en/`.
- `fallback_to_default: true`: lo no traducido muestra el español.
- Al traducir, mantener **la misma ruta y nombre** de archivo.
- **No** usar `navigation.instant` (rompe el selector de idioma del plugin i18n).

---

## 10. Publicación

| Canal | Uso |
|-------|-----|
| **Este repo (Docs)** | Documentación unificada de todos los proyectos |
| **Wiki nativa de GitHub** | Notas rápidas por repo |
| **MkDocs Material** | Sitio publicado en GitHub Pages |

```text
push a main → workflow .github/workflows/deploy.yml
            → mkdocs build → rama gh-pages → GitHub Pages
```

- Páginas en `docs/es/<proyecto>/`.
- `site_url` y `custom_dir: overrides` configurados en `mkdocs.yml`.
- **Verificar siempre** con build local antes de pushear:

```bash
python -m mkdocs build
```

---

## 11. Checklist antes de publicar

- [ ] Frontmatter con tags.
- [ ] Cabecera (Tipo / Estado / Fecha).
- [ ] Secciones numeradas.
- [ ] Datos **verificados contra el código**.
- [ ] Tablas completas (sin "etc." que esconda información).
- [ ] Enlaces internos que resuelven (sin `404`).
- [ ] `mkdocs build` en **SUCCESS**.
- [ ] `nav` actualizado si hay página nueva.
- [ ] Referencia cruzada al final.

---

## 12. Errores a evitar

| Error | Por qué |
|-------|---------|
| Inventar APIs, pines o defaults | Rompe la confianza y produce fallos reales |
| Escribir "etc." en una tabla de referencia | Esconde justo lo que se busca |
| Dejar números redondeados sin el dato exacto | No permite verificar |
| Documentar solo lo que funciona | Los **bloqueos** y limitaciones son igual de importantes |
| No citar la versión | En 3 releases el dato ya no aplica |
| Duplicar contenido en dos páginas | Se desincronizan; mejor linkear |
| Páginas huérfanas (fuera del `nav`) | Nadie las encuentra |
| Documentar en pasado sin fecha | No se sabe si sigue vigente |

---

## 13. Qué documentar siempre (lista mínima)

Para cualquier proyecto nuevo:

1. **Identidad**: qué es, para qué sirve, qué resuelve.
2. **Inicio rápido**: de cero a funcionando.
3. **Arquitectura** + diagramas de datos.
4. **Hardware**: compatibilidad, pines, fichas de conexión, BOM.
5. **Configuración**: todas las claves con tipo/default.
6. **API**: endpoints/tópicos, con ejemplos.
7. **Conceptos**: los algoritmos usados (PID, histéresis…).
8. **Operación**: OTA, seguridad, estados, diagnóstico.
9. **Soporte**: solución de problemas + FAQ.
10. **Proyecto**: decisiones (ADR), evolución, mejoras, CHANGELOG,
    registro de cambios por archivo, glosario, métricas.
11. **Convenciones**: reglas de trabajo, versionado, este estándar.

---

## 14. Referencias cruzadas obligatorias

| Desde | Hacia |
|-------|-------|
| Guía de pines | Hardware y conexiones · Compatibilidad |
| Referencia de configuración | Variables modificables · API REST |
| API interna | Referencia de código · Enumeraciones |
| Soporte | Identidad y estados · FAQ |
| CHANGELOG | Versionado · Mejoras |

---

## 15. Estructura recomendada de la wiki

```text
Inicio (índice del proyecto)
├── Empezar        → Inicio rápido · Arquitectura · Diagramas · Compilación
├── Referencia     → Código · API interna · Enumeraciones · Configuración (JSON)
├── Hardware       → Compatibilidad · Pines · Conexiones · Materiales
├── Uso            → Sensores · Actuadores · Conceptos · Configuración
├── Operación      → API REST · MQTT · OTA · Seguridad · Estados
├── Soporte        → Solución de problemas · FAQ · Glosario
└── Proyecto       → Decisiones · Evolución · Mejoras · CHANGELOG · Registro
```

Ver también: [Versionado](Versionado.md) · [Reglas de trabajo](Reglas-de-trabajo.md).
