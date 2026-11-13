# Mercería Plice

Tienda online premium de telas, botones, hilos, cintas y accesorios de costura.
Mendoza, Argentina. Diseño editorial, carrito persistente, checkout validado en
servidor y una capa de seguridad pensada para exponerse a internet.

> **Estado:** aplicación completa y compilable, con **pago y email simulados** y
> catálogo estático. Las integraciones reales (MercadoPago/Stripe, email,
> base de datos) están preparadas pero no conectadas. Ver
> [Limitaciones](#limitaciones-conocidas).

---

## Contenido

- [Stack](#stack)
- [Puesta en marcha](#puesta-en-marcha)
- [Scripts](#scripts)
- [Rutas y APIs](#rutas-y-apis)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Catálogo y precios](#catálogo-y-precios)
- [Carrito](#carrito)
- [Checkout](#checkout)
- [Seguridad](#seguridad)
- [Integrar pagos reales](#integrar-pagos-reales)
- [Email y newsletter](#email-y-newsletter)
- [Deploy](#deploy)
- [Rendimiento](#rendimiento)
- [Accesibilidad y SEO](#accesibilidad-y-seo)
- [Limitaciones conocidas](#limitaciones-conocidas)
- [Documentación](#documentación)

---

## Stack

| Capa | Tecnología |
| --- | --- |
| Framework | Next.js 15 (App Router, React Server Components) |
| UI | React 19 + TypeScript 5.7 |
| Estilos | Tailwind CSS 3.4 + sistema de diseño propio en `app/globals.css` |
| Animación | Framer Motion 11 + micro-interacciones CSS |
| Estado | Zustand 5 con `persist` (localStorage) |
| Validación | Zod 3 (cliente y servidor, mismo esquema) |
| Iconos | lucide-react |
| Tests | Vitest 2 |
| Carga | k6 y Artillery (scripts incluidos) |

Sin base de datos, sin CMS y sin imágenes remotas: el catálogo vive en
`lib/products.ts` y los visuales de producto son degradados y texturas CSS.
Eso mantiene el repositorio autocontenido y el build reproducible.

Requisitos: **Node.js 20+** y npm 10+.

---

## Puesta en marcha

```bash
npm install
cp .env.example .env.local   # opcional en desarrollo
npm run dev
```

Abrí <http://localhost:3000>.

Build de producción:

```bash
npm run build
npm start
```

> La app no necesita variables de entorno para funcionar en modo demo. Las
> variables de `.env.example` son el contrato previsto para las integraciones
> externas; hoy ninguna se consume salvo `NODE_ENV`.

---

## Scripts

| Comando | Qué hace |
| --- | --- |
| `npm run dev` | Servidor de desarrollo con hot reload. |
| `npm run build` | Compila la app para producción. |
| `npm start` | Sirve el build (requiere `npm run build`). |
| `npm run typecheck` | `tsc --noEmit`, sin emitir archivos. |
| `npm test` | Ejecuta la suite de Vitest una vez. |
| `npm run test:watch` | Vitest en modo watch. |
| `npm run test:load:k6` | Prueba de carga con k6 (requiere la CLI). |
| `npm run test:load:artillery` | Prueba de carga con Artillery (requiere la CLI). |

---

## Rutas y APIs

### Páginas

| Ruta | Tipo | Descripción |
| --- | --- | --- |
| `/` | Server Component | Home: hero con parallax, categorías, destacados, historia, beneficios, CTA. |
| `/catalogo` | Server + cliente | Catálogo con filtros por categoría, color, medida, precio y búsqueda; panel móvil. Acepta `?categoria=telas` y `?focus=buscar`. |
| `/producto/[slug]` | Server Component | Ficha de producto, selector de color/medida, stock, relacionados, JSON-LD `Product`. |
| `/checkout` | Cliente | Formulario en 3 pasos, resumen sticky, CSRF y confirmación animada. |
| `/contacto` | Server + cliente | Formulario seguro, datos del local, mapa estilizado y preguntas frecuentes con anclas `#envios`, `#mayorista`, `#talles`, `#cambios`. |
| `/_not-found` | Server Component | 404 con estética de la marca. |

Todas las páginas se renderizan **dinámicamente** (`ƒ`) porque el layout raíz lee
el nonce CSP con `headers()`. Es intencional: permite una CSP estricta por
request. Ver [ARCHITECTURE.md](ARCHITECTURE.md#renderizado).

### Endpoints

| Método | Endpoint | Límite por IP | Descripción |
| --- | --- | --- | --- |
| `GET` | `/api/products` | 120/min | Catálogo filtrable. Devuelve `{ total, items }`. |
| `POST` | `/api/checkout` | 8 / 10 min | Recalcula precios y stock, simula el pago, devuelve `201` con `orderId`. |
| `POST` | `/api/contacto` | 5 / hora | Valida, sanitiza y registra la consulta; devuelve `201` con referencia. |

Parámetros de `GET /api/products`: `category` (`telas`, `botones`, `hilos`,
`cintas`, `accesorios`, `kits`), `q`, `min`, `max`, `sort`
(`relevancia`, `precio-asc`, `precio-desc`, `novedades`) y `limit` (1–50,
default 24). Respuesta cacheable 60 s con `stale-while-revalidate=300`.

> El middleware aplica además **40 req/min** a cualquier `/api/*`. El límite
> efectivo por IP es siempre el más restrictivo de las dos capas.

Códigos de error relevantes:

| Código | Significado |
| --- | --- |
| `400` | JSON inválido o parámetros inválidos. |
| `403` | Origin no permitido o CSRF ausente/incorrecto. |
| `409` | Producto inexistente, stock insuficiente o subtotal manipulado. |
| `413` | Body mayor al límite (48 KB checkout, 32 KB contacto). |
| `415` | `Content-Type` distinto de `application/json`. |
| `422` | Validación Zod fallida (incluye `fields` por campo). |
| `429` | Rate limit excedido; incluye `Retry-After`. |

---

## Estructura del proyecto

```text
.
├── app/
│   ├── layout.tsx                 # Fuentes, metadata, header/footer, carrito, JSON-LD Store
│   ├── globals.css                # Tokens, utilidades y componentes del design system
│   ├── page.tsx                   # Home
│   ├── catalogo/page.tsx          # Catálogo (Suspense + skeleton)
│   ├── producto/[slug]/page.tsx   # Ficha + JSON-LD Product
│   ├── checkout/page.tsx          # Checkout
│   ├── contacto/page.tsx          # Contacto + FAQ
│   ├── not-found.tsx
│   └── api/
│       ├── products/route.ts
│       ├── checkout/route.ts
│       └── contacto/route.ts
├── components/
│   ├── home/hero.tsx              # Parallax, sello rotante, marquee
│   ├── catalog/catalog-view.tsx   # Filtros, orden, panel móvil
│   ├── checkout/checkout-view.tsx # Formulario 3 pasos + éxito
│   ├── cart-drawer.tsx            # Carrito lateral
│   ├── site-header.tsx / site-footer.tsx
│   ├── product-card.tsx / product-detail.tsx / product-visual.tsx
│   ├── contact-form.tsx
│   ├── newsletter-form.tsx
│   ├── reveal.tsx                 # Animaciones de entrada
│   └── logo.tsx
├── lib/
│   ├── products.ts                # Catálogo, precios, envío, helpers
│   ├── types.ts                   # Product, CartLine, OrderSummary
│   ├── validation.ts              # Esquemas Zod compartidos
│   ├── client-validation.ts       # Validaciones livianas de UI
│   ├── api.ts                     # JSON, body limitado, rate limit por endpoint
│   ├── utils.ts                   # cn, lineKey, orderId, CSRF client-side
│   ├── store/cart.ts              # Zustand + persist
│   └── security/
│       ├── csrf.ts                # Double-submit + Origin
│       ├── rate-limit.ts          # Ventana deslizante en memoria
│       └── sanitize.ts            # Strip, escape, detección de payloads
├── middleware.ts                  # CSP nonce, headers, CSRF, rate limit global
├── tests/                         # 53 tests unitarios (Vitest)
├── load-test/                     # k6.js, artillery.yml y helper
├── tailwind.config.ts / postcss.config.mjs / next.config.mjs
├── vitest.config.ts / tsconfig.json
├── .env.example
├── README.md / ARCHITECTURE.md / SECURITY.md / TESTING.md
```

---

## Catálogo y precios

Todo el catálogo está en `lib/products.ts`:

```ts
{
  id: "p-lino-sierras",          // identificador estable usado por el carrito
  slug: "lino-rustico-sierras",  // URL /producto/[slug]
  name: "Lino Rústico Sierras",
  category: "telas",             // CategorySlug
  description: "…",
  details: ["100% lino europeo", "Ancho 150 cm"],
  price: 8900,                   // ARS, entero
  unit: "metro",
  colors: [{ name: "Natural", hex: "#d9c7a7" }],
  size: [{ label: "1 m", value: 1, unit: "m" }], // opcional
  material: "Lino",
  stock: 42,
  rating: 4.9,
  reviews: 128,
  swatch: "linear-gradient(…)",  // visual CSS
  swatchAlt: "linear-gradient(…)",
  texture: "weave",              // weave | dots | thread | ribbon | pearl
}
```

Reglas al agregar un producto:

1. `id` y `slug` deben ser únicos (hay un test que lo verifica).
2. `price` es un entero en pesos; no usar decimales.
3. `colors[].name` y `size[].label` son el contrato que valida el checkout: el
   servidor rechaza un color o medida que no exista en el producto.
4. `stock` se descuenta solo en la simulación; no hay reserva real.
5. `featured: true` lo hace elegible para la home (máximo 8).

Envío en `lib/products.ts`:

```ts
export const SHIPPING = { freeFrom: 25000, cost: 2400, … };
```

- Retiro en local: sin cargo.
- Envío: `$2.400`, gratis desde `$25.000`.
- El servidor recalcula el envío; nunca confía en el total del cliente.

---

## Carrito

- Estado global con Zustand (`lib/store/cart.ts`), persistido en
  `localStorage` bajo la clave `plice-cart-v1`.
- Solo se persisten `lines`; el drawer abierto/cerrado no se guarda.
- La identidad de una línea es `lineKey(productId, color, size)`; cambiar de
  color o medida crea otra línea.
- Cantidad máxima por línea: **50**.
- El carrito no conoce el stock real ni sincroniza con el servidor. Si un precio
  o stock cambió, el checkout responde `409` y la UI muestra el error.
- `hasHydrated` evita parpadeos y permite mostrar skeletons hasta leer
  localStorage.
- El contador del header usa `selectCount` y el subtotal `selectSubtotal`.

---

## Checkout

1. Validación local con las mismas reglas que el servidor (`checkoutSchema` de
   Zod) y helpers de `lib/client-validation.ts`.
2. Envío de `POST /api/checkout` con `Content-Type: application/json` y header
   `x-csrf-token` leído de la cookie `plice_csrf`.
3. El servidor, en orden: rate limit → Origin → CSRF → body limitado →
   Zod → existencia de producto → color/medida → stock → **reprecio** →
   comparación de subtotal (±1) → envío → total.
4. Respuesta `201` con `orderId`, `total`, `estimatedDelivery` y un mensaje
   según el medio de pago. La UI limpia el carrito y muestra la confirmación.

Medios de pago ofrecidos en la UI: MercadoPago, tarjeta y transferencia. **No se
piden ni se almacenan datos de tarjeta**; en la demo el pago es simulado y el
pedido no se persiste.

---

## Seguridad

Resumen de lo implementado (detalle completo en [SECURITY.md](SECURITY.md)):

- CSP con nonce por request en producción; `strict-dynamic`; sin `unsafe-inline`
  para scripts.
- CSRF double-submit (`plice_csrf` + `x-csrf-token`) con comparación en tiempo
  constante y verificación de `Origin`.
- Rate limiting global (180 req/min en páginas, 40 req/min en APIs) y por
  endpoint (checkout 8/10 min, contacto 5/h, productos 120/min).
- Validación Zod + sanitización + detección de payloads + honeypot.
- Límite de body y `Content-Type` estricto.
- Headers: HSTS, `X-Frame-Options: DENY`, `nosniff`, `Referrer-Policy`,
  `Permissions-Policy`, COOP/CORP, `poweredByHeader: false`.
- Reprecio y verificación de stock en servidor.
- Logs sin PII: no se registran nombres, emails ni cuerpos de mensajes.

**No incluido:** autenticación, panel de administración, persistencia de
pedidos, antifraude, CAPTCHA ni WAF. Ver checklist de producción en
[SECURITY.md](SECURITY.md#checklist-para-producción).

---

## Integrar pagos reales

El punto de integración es `app/api/checkout/route.ts`, después del reprecio y
antes de la respuesta `201`. La simulación actual es:

```ts
await new Promise((resolve) => setTimeout(resolve, 550));
```

### MercadoPago (Argentina)

1. Crear una preferencia con el SDK oficial del lado servidor usando
   `MERCADOPAGO_ACCESS_TOKEN`, con `items` derivados de `verifiedItems` y
   `external_reference = orderId`.
2. Devolver `init_point` y redirigir al cliente; no confiar en el monto enviado
   por el navegador.
3. Implementar un webhook que valide la firma (`MERCADOPAGO_WEBHOOK_SECRET`),
   consulte el pago en la API y actualice el pedido de forma idempotente.
4. Guardar el pedido antes de redirigir para poder reconciliar el webhook.

### Stripe

1. Crear un `PaymentIntent` con el total recalculado y `metadata.orderId`.
2. Devolver `client_secret` y confirmar en el cliente con Stripe.js.
3. Verificar la firma del webhook (`STRIPE_WEBHOOK_SECRET`) y procesar
   `payment_intent.succeeded` / `payment_intent.payment_failed` con idempotencia.

En ambos casos: nunca loguear datos de tarjeta, no confiar en el `subtotal` del
cliente y mantener el reprecio actual como fuente de verdad.

---

## Email y newsletter

- **Contacto:** `POST /api/contacto` valida, sanitiza y registra solo
  `reference`, `subject` y si hay teléfono. No envía email todavía.
- **Newsletter:** el formulario del footer valida el email y lo guarda en
  `localStorage` (`plice-newsletter`). No hay endpoint ni proveedor conectado.
- Para producción, integrar un proveedor transaccional (por ejemplo Resend) en
  los route handlers y mover el newsletter a un endpoint con doble opt-in.

---

## Deploy

### Vercel

1. Importar el repositorio.
2. Configurar las variables de `.env.example` que correspondan.
3. Build command: `npm run build`; output: automático de Next.js.
4. Definir el dominio y actualizar `metadataBase` en `app/layout.tsx` (hoy está
   fijado a `https://merceriaplice.com.ar`).

> En serverless, el rate limiter en memoria es **por instancia**. Para producción
> multi-instancia, reemplazar el `store` de `lib/security/rate-limit.ts` por
> Redis/Upstash. Ver [SECURITY.md](SECURITY.md#escalado-horizontal).

### Servidor propio / contenedor

```bash
npm ci
npm run build
NODE_ENV=production npm start
```

Detrás de un proxy (Nginx/Caddy/Cloudflare):

- Terminar TLS en el proxy y redirigir HTTP → HTTPS.
- **Sobrescribir** `X-Forwarded-For` con la IP real del cliente; no aceptar el
  valor que envía el navegador.
- Mantener una sola instancia o compartir el estado del rate limiter.

No se incluye `Dockerfile` en este repositorio.

---

## Rendimiento

- Build de referencia: **103 kB** de JS compartido; home ~166 kB, checkout
  ~180 kB de First Load JS; middleware ~35 kB.
- Todas las rutas son dinámicas por el nonce CSP: no hay HTML estático
  cacheable. Es el costo de la CSP estricta.
- `GET /api/products` responde con `Cache-Control: public, max-age=60,
  stale-while-revalidate=300`.
- Sin imágenes remotas: los visuales son CSS, por lo que no hay descargas de
  producto ni CLS por imágenes.
- Fuentes autohospedadas por `next/font` (Fraunces y Manrope), sin requests a
  Google en runtime.
- No se ejecutaron benchmarks de latencia en este entorno; los scripts de carga
  están listos para medir en staging.

---

## Accesibilidad y SEO

- Skip link, landmarks (`header`, `main`, `footer`), labels asociados y
  `aria-pressed`/`aria-live` en filtros y resultados.
- Estados de error con `role="alert"` y `aria-invalid` en los formularios.
- Foco visible y navegación por teclado en carrito, filtros y checkout.
- Metadata por página, Open Graph, `lang="es-AR"` y JSON-LD `Store` y `Product`.
- No hay auditoría automatizada de accesibilidad ni `sitemap.xml`/`robots.txt`
  generados; ver [TESTING.md](TESTING.md#fuera-de-alcance).

---

## Limitaciones conocidas

- **Pago simulado:** no se cobra, no se crea preferencia ni se procesa webhook.
- **Sin persistencia de pedidos:** `orderId` se genera y se devuelve, pero no se
  guarda en ninguna base.
- **Catálogo estático:** los cambios requieren deploy; no hay panel ni CMS.
- **Stock no reservado:** dos compradores concurrentes pueden pasar la
  validación con la última unidad.
- **Rate limit en memoria:** por instancia; no coordinado entre regiones.
- **Email no enviado:** ni confirmación de pedido ni respuesta de contacto.
- **Newsletter local:** solo `localStorage`, sin backend.
- **Sin autenticación:** no hay cuentas, historial ni área de administración.
- **CSP dinámica:** impide HTML estático y caching de páginas en CDN.

---

## Documentación

- [ARCHITECTURE.md](ARCHITECTURE.md) — capas, renderizado, estado, flujos y
  contratos de API.
- [SECURITY.md](SECURITY.md) — modelo de amenazas, OWASP Top 10, controles,
  límites y checklist de producción.
- [TESTING.md](TESTING.md) — tests unitarios, verificación manual de seguridad,
  pruebas de carga y CI.

---

Proyecto privado de Mercería Plice. Todos los derechos reservados.
# web-merceria
