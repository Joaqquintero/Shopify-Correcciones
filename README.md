# Viriathus · Carrito de Shopify con pedido por WhatsApp

Personalización del carrito lateral (`cart-drawer.liquid`) de la tienda en línea de **Viriathus · Objetos con historia**.

En lugar de pasar por el checkout de Shopify, el cliente completa sus datos y la compra se cierra **por WhatsApp** con un asesor. Al mismo tiempo, el pedido queda registrado en Shopify como **borrador (Draft Order)**.

---

## ¿Qué hace?

En el carrito hay dos botones:

| Botón | Para qué |
|---|---|
| **COTIZA TU ENVÍO** | El cliente deja su dirección para que le coticen el envío. |
| **RETIRAR EN SHOWROOM** | El cliente pasa a recoger a la tienda (Mérida 22, Roma Norte, CDMX). |

Los dos abren un formulario:

1. **Tus datos:** nombre, teléfono y correo. Se piden siempre.
2. **Datos para el envío:** solo en «Cotiza tu envío». Incluye calle, números, piso, elevador de carga, colonia, CP, municipio, estado, país y notas.
3. **Facturación (opcional):** si el cliente necesita factura, adjunta su **Constancia de Situación Fiscal (PDF)**. El navegador la lee y llena solo el **RFC**, la **razón social**, el **código postal fiscal** y el **régimen fiscal**. El cliente elige el **uso del CFDI**, escribe el correo para la factura y confirma que los datos son correctos.
4. **Continuar por WhatsApp:**
   - se crea el **borrador del pedido en Shopify** (a través de un Worker de Cloudflare);
   - se abre **WhatsApp** con un mensaje listo: modalidad, número de pedido de Shopify, productos, cantidades, importes, total, datos del cliente, envío y si requiere factura.

---

## Cómo está armado

```
Carrito (Shopify, cart-drawer.liquid)
   │  1. lee el carrito          → /cart.js
   │  2. lee la Constancia (PDF) → pdf.js, en el navegador
   │  3. envía el pedido         → Worker de Cloudflare
   ▼
Worker de Cloudflare  ──►  API de Shopify: crea el Draft Order
   │
   ▼
Carrito: abre WhatsApp con el mensaje y el número de pedido
```

- **pdf.js** se descarga **solo cuando el cliente adjunta su Constancia** (carga diferida), no en cada página de la tienda.
- La Constancia **no se sube a ningún servidor**: se lee en el navegador del cliente y solo se usan los datos extraídos.
- El Worker recibe del carrito **solo el producto (variante) y la cantidad**. Los precios los toma Shopify, así nadie puede crear un pedido con precios modificados.

---

## Archivos

| Archivo | Qué es |
|---|---|
| `sections/cart-drawer.liquid` | El carrito, el formulario, los estilos y el script. |
| `worker.js` | Worker de Cloudflare que crea el Draft Order en Shopify (se publica aparte, en Cloudflare). |

---

## Instalación

1. **Respaldo:** en Shopify → *Tienda online → Temas → ⋯ → Duplicar*.
2. **Editar el tema:** *⋯ → Editar código → `sections/cart-drawer.liquid`*. Reemplazar el contenido y guardar.
3. **Worker:** publicarlo en Cloudflare con sus variables de entorno (ver «Configuración»).
4. **Probar** con la lista de «Pruebas».

---

## Configuración

Dentro de `cart-drawer.liquid`:

| Qué | Dónde | Valor actual |
|---|---|---|
| URL del Worker | `fetch('https://…workers.dev')` en `sendToWhatsApp()` | El Worker de la tienda |
| Número de WhatsApp | `const whatsappNumber` | Número de ventas, con código de país, sin `+` |
| Correo para Constancias escaneadas | Textos de ayuda y mensajes de error | Correo de facturación |
| Versión de pdf.js | `cargarPdfjs()` | 3.11.174 (cdnjs) |

En el Worker (variables de entorno de Cloudflare, **nunca en el código**):

| Variable | Para qué |
|---|---|
| Dominio de la tienda | La tienda de Shopify a la que se le crean los pedidos |
| Token de la API de Shopify | Permiso para crear Draft Orders |

> ⚠️ No subas tokens ni llaves a GitHub. Si alguna vez se subió uno por error, revócalo en Shopify y genera uno nuevo.

### Lo que el carrito le envía al Worker

```json
{
  "cart": { "items": [ { "merchandise_id": "gid://shopify/ProductVariant/123", "quantity": 1 } ] },
  "orderType": "shipping | showroom",
  "customer": { "fullName": "", "phone": "", "email": "" },
  "shipping": { "street": "", "colonia": "", "postalCode": "" },
  "fiscal": { "requiresInvoice": true, "rfc": "", "fiscalName": "", "fiscalPostalCode": "", "fiscalRegime": "", "fiscalCFDI": "", "fiscalEmail": "" }
}
```

Respuesta esperada: `{ "ok": true, "draftOrder": { "name": "#D123" } }`, o `{ "ok": false, "error": "…" }`.

---

## Pruebas

- [ ] **Cotiza tu envío**, sin factura: el mensaje de WhatsApp trae tus datos y la dirección, y el borrador aparece en Shopify.
- [ ] **Retirar en showroom**, sin factura: pide nombre, teléfono y correo, no pide dirección, y el borrador aparece con el cliente.
- [ ] **Con factura** y una Constancia original del SAT: se muestran RFC, razón social, CP y régimen.
- [ ] **Con factura** y una Constancia escaneada o en foto: aparece el aviso de enviarla por correo.
- [ ] Faltan datos obligatorios: el navegador marca el campo y no deja continuar.
- [ ] Carrito vacío: aparece el error «Tu carrito está vacío».
- [ ] En celular: el formulario se ve en una columna y se puede desplazar.

---

## Limitaciones conocidas

- **Borradores huérfanos.** El borrador se crea antes de abrir WhatsApp. Si el cliente no manda el mensaje, el borrador queda igual en Shopify.
- **Constancias escaneadas o en foto.** pdf.js solo lee PDF con texto. Esas Constancias se procesan a mano por correo.
- **Lectura de la Constancia.** Depende del formato actual del SAT: si el SAT cambia el diseño del PDF, hay que revisar las expresiones que buscan el RFC, la razón social, el CP y el régimen.

---

## Pendientes / ideas

- [ ] Que el Worker acepte pedidos **solo desde el dominio de la tienda** (hoy acepta cualquier origen) y limite la cantidad de productos por pedido.
- [ ] Avisarle al **píxel de Meta** cuando alguien toca «Continuar por WhatsApp», para que los anuncios de catálogo aprendan quién escribe de verdad.
- [ ] Pasar el número de WhatsApp y el correo a la **configuración del tema**, para cambiarlos sin tocar código.

---

## Tecnologías

- Shopify **Liquid** (tema), HTML, CSS y JavaScript
- **pdf.js** (Mozilla), para leer la Constancia de Situación Fiscal en el navegador
- **Cloudflare Workers** + **Shopify Admin API**, para crear los Draft Orders
- **WhatsApp** (enlaces `wa.me`), para cerrar la venta con un asesor

---

Hecho para **Viriathus · Objetos con historia**, CDMX.
