# Plataforma Propia — Arquitectura y Plan de Construcción

**Proyecto:** Reescritura completa de encambiodiario.mx como plataforma propietaria, sin WordPress ni CMS comercial.
**Fecha:** 10 de octubre de 2026
**Documento complementario:** `diagnostico-y-propuesta.md` (diagnóstico del sitio actual)

---

## 1. Premisa y alcance real

Objetivo: **control total del código, los datos y el roadmap** — cero dependencia de licencias de CMS, plugins premium o temas de terceros. Esto NO significa escribir todo desde cero (un servidor HTTP o una base de datos propios serían anti-productivos); significa usar frameworks open source bajo nuestro control, donde podemos leer, modificar y desplegar cada línea.

**Inventario a migrar (medido desde la API pública del sitio actual):**

| Entidad | Volumen | Notas |
|---|---|---|
| Artículos | **30,841** | Desde oct/2019 (~7 años de archivo) |
| Medios (imágenes/video) | **13,913** | Migrar a object storage + CDN |
| Categorías | 26 | Razonable |
| Etiquetas | **19,628** | Inflación severa — consolidar a <500 en la migración |
| Páginas estáticas | 6 | Trivial |

Este volumen define el proyecto: no es un micrositio, es una migración seria con **preservación de SEO como riesgo #1**.

---

## 2. Arquitectura objetivo

```
                        ┌─────────────────────────────────────┐
   Reportero (celular)  │           CLOUDFLARE CDN            │
   ┌──────────┐         │   caché HTML + imágenes + WAF       │
   │ PWA      │────────►└───────┬──────────────────┬──────────┘
   │ Redacción│                 │                  │
   └──────────┘          ┌──────▼──────┐    ┌──────▼──────────┐
   ┌──────────┐          │  FRONTEND   │    │  MEDIA PIPELINE │
   │ Larix/   │          │  Astro/Next │    │  imgproxy → AVIF│
   │ app live │          │  SSG + ISR  │    │  S3 compatible  │
   └────┬─────┘          └──────┬──────┘    └─────────────────┘
        │                       │
   ┌────▼────────┐      ┌──────▼──────────────────────────────┐
   │ MediaMTX    │      │  API EDITORIAL (Node/Fastify o Go)   │
   │ RTMP→HLS/   │      │  auth • posts • workflow • colas     │
   │ WebRTC      │      └──┬───────┬────────┬──────────┬───────┘
   └─────────────┘         │       │        │          │
                     ┌─────▼──┐ ┌──▼───┐ ┌──▼─────┐ ┌──▼────────┐
                     │Postgres│ │Redis │ │Search  │ │Web Push   │
                     │        │ │colas │ │Meili/  │ │VAPID propio│
                     └────────┘ └──────┘ │Typesense│└───────────┘
                                          └────────┘
                          ┌──────────────────────────┐
                          │  Distribución social     │
                          │  Meta/X/TikTok/YT APIs   │
                          │  WhatsApp Channel (man.) │
                          └──────────────────────────┘
```

### Principios de diseño

- **El sitio público es (casi) estático.** Las notas se pre-renderizan y se revalidan (ISR); la portada se regenera por evento, no por petición. Resultado: TTFB <100ms desde edge, CWV en verde por diseño, y el origen aguanta picos de viralización sin caerse.
- **La redacción es una app aparte.** Editor web + PWA de reportero hablan con la misma API. Nada de wp-admin.
- **Todo dato es nuestro:** PostgreSQL, backups versionados, exportación completa en JSON/Markdown en cualquier momento.

---

## 3. Stack recomendado

| Capa | Elección primaria | Alternativa | Justificación |
|---|---|---|---|
| Frontend público | **Astro** (+ islas React) | Next.js | Render por defecto a HTML puro, JS mínimo → perfecto para Discover/CWV |
| API / backend | **Fastify + TypeScript** | Go (Chi) o NestJS | Equipo JS único en todo el stack; Fastify es rápido y austero |
| Base de datos | **PostgreSQL** | — | JSONB para cuerpo de artículo, FTS nativo de respaldo |
| Caché/colas | **Redis + BullMQ** | NATS | Colas para auto-posteo social, regeneración ISR, newsletters |
| Búsqueda interna | **Meilisearch** (self-hosted) | Postgres FTS | Búsqueda instantánea tolerante a typos |
| Editor de texto | **TipTap** (headless) | Lexical | Editor propio sobre base OSS sólida, guarda JSON limpio |
| Storage media | **S3 compatible** (Backblaze/MinIO) | — | + imgproxy o sharp para AVIF/WebP/responsive |
| Live streaming | **MediaMTX** self-hosted | OvenMediaEngine | Ingesta RTMP desde el celular → LL-HLS/WebRTC al sitio; gratis |
| Push | **Web Push (VAPID)** implementado propio | — | Sin OneSignal ni suscripciones de terceros |
| Newsletter | **Listmonk** o módulo propio + SES | MailPoet propio | OSS, self-hosted |
| Auth | **Propia: passkeys + TOTP** | — | Roles: reportero, editor, admin, corresponsal |
| Infra | **Docker Compose → VPS** (Hetzner/Contabo) | Fly.io / k8s (sobre-ingeniería hoy) | 2 VPS: app+DB y streaming; ~$40–80 USD/mes |
| Analytics | **Umami** self-hosted + GA4 en transición | — | Métricas propias, sin cookies invasivas |

**Lo que NO construimos** (no reinventar): servidor HTTP, DB, motor de búsqueda, CDN, compresión de video. Ahí se usan proyectos OSS maduros bajo nuestro despliegue.

---

## 4. Modelo de datos esencial

```
posts (id, slug, title, subtitle, body_jsonb, excerpt, status,
       edition [qroo|tabasco|nacional|...], category_id, author_id,
       published_at, updated_at, breaking: bool, live_id?, seo_meta)
media_assets (id, kind [image|video|audio], storage_key, width, height,
              focal_point, credit, alt)
post_media (post_id, media_id, role [featured|inline|gallery])
authors (id, slug, name, bio, photo_id, social_links jsonb, active)
categories / tags (id, slug, name, parent_id)
revisions (post_id, version, body_jsonb, saved_by, saved_at)
live_events (id, title, status, rtmp_key, hls_url, started_at, post_id)
distribution_queue (post_id, network, payload, status, attempts)
push_subscriptions / newsletter_subscribers
comments (opcional — moderación propia)
```

Regla de oro del esquema: **`slug` + `category` determinan la URL pública**, idéntica a la actual (`/categoria/slug/`).

---

## 5. Migración desde WordPress (ETL)

La REST API actual (`/wp-json/wp/v2/`) permite extraer todo sin acceso al servidor:

1. **Extracción:** script paginado sobre `posts` (30,841), `media` (13,913), `categories`, `tags`, `users`. Respetar `per_page=100` + headers `X-WP-TotalPages`. Resumible por checkpoint.
2. **Transformación:**
   - `content.rendered` HTML → limpiar shortcodes/embeds de WP/JNews → TipTap JSON.
   - Re-mapear `categories` (26 → taxonomía nueva) y **colapsar 19,628 tags a una lista curada** — la inflación de tags daña SEO (thin archives).
   - Dedup de medios; descarga a S3 + regeneración de tamaños con sharp.
3. **Carga:** inserción en Postgres preservando `slug`, `date`, `author` → **URLs idénticas**.
4. **Verificación:** crawler comparativo viejo-vs-nuevo (status, canonical, título, imagen) sobre muestra + todas las URLs del sitemap.

## 6. Preservación de SEO — el riesgo #1

- **URLs byte-idénticas.** Mismo esquema `/categoria/slug/`. Cero redirects masivos = cero pérdida de rankings.
- **Sitemaps:** `sitemap_index.xml` + `news-sitemap.xml` con la misma nomenclatura que Google ya conoce (publicación: resolver marca QRoo/Tabasco antes del corte).
- **Schema `NewsArticle` completo:** `dateline`, `author:Person` con URL, `image`, `publisher` con `Organization` + logo. Mejor que lo que Yoast emite hoy.
- **Redirects solo donde cambie la URL:** mapa 301 explícito para tags consolidados y páginas legacy (`?p=`, feeds viejos).
- **Corte controlado:** DNS con TTL bajo, staging en subdominio verificado por Search Console, monitoreo de Coverage/Discover diario las primeras 2 semanas, rollback a WP detrás del mismo dominio si hay catástrofe.

## 7. Módulo Reportero Móvil (MoJo) — la pieza estrella

**PWA de redacción** (instalable, sin app stores en fase 1):

- Captura de foto/video directo (input `capture`), compresión en cliente, subida resumible por chunks — sobrevive a mala señal en cobertura.
- Borradores offline en IndexedDB; sincroniza al recuperar red.
- "Enviar a revisión" / "Publicar" según rol. Corrección de estilo básica asistida.
- Notificación al editor por push interno cuando entra una nota a revisión.
- Geolocalización opcional de la nota (`dateline` real → señal de calidad para Google).

**En vivo:**

- Botón "Transmitir" en la PWA → genera `live_event` con clave RTMP efímera → reportero apunta Larix/OBS móvil a nuestro MediaMTX → el sitio sirve LL-HLS con reproductor propio (hls.js) + **simulcast saliente a YouTube/FB** (restream propio o API).
- Al terminar: archivo del video → nota automática con `VideoObject` + corte de clip vertical para Reels/TikTok/Shorts.
- Schema `LiveBlogPosting` + `BroadcastEvent` → sello "EN VIVO" en resultados de Google.
- Chat en vivo propio (WebSocket) moderado, opcional.

**Distribución social automatizada:**

- Cola en Redis: al publicar → posts a Facebook Page (Graph API), X (API v2), TikTok (Content Posting API), YouTube (upload de clip). Cada reportero puede conectar sus cuentas con OAuth → su perfil amplifica la nota con su firma.
- Plantillas por red (copys, hashtags, crop vertical).
- WhatsApp Channel: difusión semimanual con plantilla (API broadcast no disponible para Channels aún).

## 8. Seguridad y operación

- Auth con **passkeys** + TOTP de respaldo; sesiones por dispositivo revocables.
- Rate limiting en API + Cloudflare WAF; CSP estricta; CORS cerrado.
- Backups: Postgres (WAL + dump diario) y S3 con versionado a segunda ubicación; prueba de restore mensual.
- Observabilidad: logs estructurados + alertas (uptime, cola social atascada, errores 5xx).
- CI/CD: repo Git → tests → build → deploy con rollback de un comando.

## 9. Roadmap (equipo de 2 devs + 1 editor técnico)

| Fase | Duración | Entregable |
|---|---|---|
| **0 — Cimientos** | 3 sem | Repo, CI, DB, API base, auth, modelo de datos, staging |
| **1 — CMS + Frontend** | 6 sem | Editor TipTap, workflow editorial, frontend Astro con diseño nuevo, sitemaps, schema |
| **2 — Migración** | 4 sem | ETL completo, verificación crawler, staging con los 30k artículos |
| **3 — Corte** | 2 sem | Redirects residuales, DNS, monitoreo SEO, rollback plan |
| **4 — PWA reportero** | 5 sem | Captura, offline, workflow móvil |
| **5 — En vivo** | 4 sem | MediaMTX, simulcast, sección En Vivo, clips verticales |
| **6 — Distribución** | 4 sem | Auto-posteo 4 redes, push propio, newsletter |
| **7 — Monetización** | 3 sem | Ad slots propios o Ad Manager, contenido patrocinado |

**MVP transitable (fases 0–3): ~15 semanas.** Plataforma completa: ~30 semanas.

## 10. Honestidad de ingeniería (lo que hay que decir en la junta)

- **Costo real:** ~7–8 meses de equipo pequeño antes de paridad total con el WP actual. La licencia de JNews/Yoast cuesta ~$200 USD/año; el ahorro no es el motivo — **el control y la diferenciación sí**.
- **Deuda permanente:** seguridad, actualizaciones y compatibilidad ahora son nuestras. Presupuestar 20–30% del tiempo en mantenimiento.
- **Secuencia correcta:** NO apagar WP hasta que el nuevo sitio esté en staging con los 30k artículos verificados. Migración reversible siempre.
- **Dónde SÍ conviene quedarse en terceros:** CDN (Cloudflare), email transaccional, y las APIs de las redes sociales — ahí la dependencia es inevitable y barata.

## 11. Siguientes pasos inmediatos

1. Aprobar stack y roadmap (sección 3 y 9).
2. Levantar repo + staging vacío; primer sprint: modelo de datos + ETL de muestra (500 notas).
3. Resolver identidad de marca para Publisher Center (una marca vs dos ediciones).
4. Inventario editorial: qué categorías/tags sobreviven la consolidación.
5. Definir quién mantiene el WP congelado durante la transición.
