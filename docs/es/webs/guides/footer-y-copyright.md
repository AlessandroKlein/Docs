# Footer y copyright

El pie de página del sitio público se configura desde el panel de
administración, sin tocar código. Todo lo que se describe aquí se edita en
**Ajustes** (`/admin/settings`) y se renderiza automáticamente en la página
pública (`/`) mediante el componente `SiteFooter`
(`apps/web/src/components/SiteFooter.tsx`).

Los datos del pie se exponen de forma **pública** (sin autenticación) a través
de `GET /api/v1/public/footer`, que devuelve las secciones `site`, `footer` y
`social`.

---

## 1. Copyright: año en curso vs rango

La línea de copyright se compone automáticamente con el nombre del sitio y el
año (o rango de años). Se configura en la sección **Footer** con dos campos:

| Campo            | Tipo     | Descripción                                              |
| ---------------- | -------- | -------------------------------------------------------- |
| `copyrightYear`  | `integer`| Año de creación del sitio.                               |
| `copyrightMode`  | `enum`   | Modo de copyright: `current` o `range` (por defecto `range`). |

El **año en curso** se calcula siempre de forma automática a partir de la fecha
actual (`new Date().getFullYear()`). Según el modo:

- **`current`** — muestra solo el año en curso:

  ```text
  © 2026 Mi Sitio. Todos los derechos reservados.
  ```

- **`range`** — muestra `añoDeCreación–añoActual` solo si el año de creación es
  anterior al actual; en caso contrario muestra únicamente el año en curso:

  ```text
  © 2023-2026 Mi Sitio. Todos los derechos reservados.
  ```

El nombre que acompaña al copyright es el campo `name` de la sección **Sitio**
(`site`). Si el sitio está configurado en varios idiomas, el pie se renderiza
junto con el resto de la página pública, que respeta la localización activa
(ver [Idiomas](./idiomas.md)).

> Valor por defecto de la sección: `{ "copyrightMode": "range" }`. Si no se
> define `copyrightYear`, el pie muestra solo el año en curso.

---

## 2. Enlaces legales

Dentro de la sección **Footer**, el campo `legalLinks` es una lista de enlaces
legales. Cada entrada tiene dos propiedades:

| Propiedad | Tipo     | Descripción                                   |
| --------- | -------- | --------------------------------------------- |
| `label`   | `string` | Texto visible (ej. "Términos y condiciones"). |
| `url`     | `uri`    | Destino del enlace.                           |

Solo se muestran las entradas que tienen **ambos campos completos** (`label` y
`url`). Las incompletas se descartan automáticamente.

Ejemplo de `legalLinks`:

```json
[
  { "label": "Política de privacidad", "url": "/legal/privacidad" },
  { "label": "Términos de uso", "url": "/legal/terminos" },
  { "label": "Política de cookies", "url": "/legal/cookies" }
]
```

> La sección **Páginas legales** (`legal`) guarda el *contenido* de esas
> páginas (política de privacidad, términos de uso y política de cookies en
> texto enriquecido). Los `legalLinks` del footer son los enlaces que apuntan a
> ellas u a otras URLs.

---

## 3. Contacto

La sección **Footer** también expone tres campos de contacto opcionales:

| Campo             | Tipo     | Descripción            |
| ----------------- | -------- | ---------------------- |
| `contactEmail`    | `email`  | Correo de contacto.    |
| `contactPhone`    | `string` | Teléfono de contacto.  |
| `contactAddress`  | `string` | Dirección.             |

Al igual que con los enlaces legales, **solo se muestran los campos completos**.
Si ninguno está definido, la fila de contacto no aparece.

```text
✉ hola@misitio.com   ☎ +54 11 5555 0000   ⌖ Av. Siempre Viva 123
```

> No confundir con la sección **Contacto** (`contact`), que guarda datos de
> contacto generales del sitio (correo, teléfono, dirección y WhatsApp, algunos
> cifrados en reposo). El footer usa los campos `contactEmail`,
> `contactPhone` y `contactAddress` de la sección **Footer**.

---

## 4. Redes sociales

Las redes se configuran en la sección **Redes sociales** (`social`):

| Red         | Campo       | Tipo  |
| ----------- | ----------- | ----- |
| Facebook    | `facebook`  | `uri` |
| Instagram   | `instagram` | `uri` |
| X (Twitter) | `x`         | `uri` |
| LinkedIn    | `linkedin`  | `uri` |
| YouTube     | `youtube`   | `uri` |

El footer solo muestra las redes cuyo campo tiene una URL **no vacía**. Las
demás se omiten, junto con su icono.

```text
Facebook   Instagram   X   LinkedIn   YouTube
```

---

## 5. Comportamiento de "solo los completos"

El footer aplica la misma regla a todos sus bloques opcionales: **se filtran y
se muestran únicamente los valores completos**.

- Redes: se descartan las que no tienen URL.
- Enlaces legales: se descartan los que no tienen `label` **y** `url`.
- Contacto: se descarta cada campo individual vacío; si los tres están vacíos,
  la fila entera desaparece.

---

## 6. Ejemplo de respuesta pública

`GET /api/v1/public/footer` devuelve algo como:

```json
{
  "site": { "name": "Mi Sitio" },
  "footer": {
    "copyrightYear": 2023,
    "copyrightMode": "range",
    "contactEmail": "hola@misitio.com",
    "legalLinks": [
      { "label": "Privacidad", "url": "/legal/privacidad" }
    ]
  },
  "social": { "instagram": "https://instagram.com/misitio" }
}
```

---

## Ver también

- [Configuración desde la UI](./configuracion-desde-ui.md) — resto de secciones
  de Ajustes.
- [Idiomas](./idiomas.md) — localización del sitio.
- [Archivos y multimedia](./archivos.md) — subida de logos e imágenes.
