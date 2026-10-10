# Diagnóstico y Propuesta — encambiodiario.mx

**Fecha:** 10 de octubre de 2026
**Objetivo:** Elevar la visibilidad del periódico digital y habilitar publicación móvil / transmisiones en vivo por parte de los reporteros.

> **Actualización:** la decisión de proyecto evolucionó a construir una plataforma propia (sin WordPress). Ver **`plataforma-propia-arquitectura.md`** para la arquitectura, migración de los ~30,800 artículos y roadmap de construcción.

---

## 1. Resumen ejecutivo

En Cambio Diario ya tiene una base técnica sólida y sorprendentemente bien equipada para un medio regional: WordPress con el tema **JNews** (diseñado para medios), **Yoast SEO + News SEO** (sitemap de Google News activo), **Google Site Kit** (GA4 + Search Console), **Cloudflare** y conversión a **WebP**. Publica varias notas al día en dos ediciones (Quintana Roo y Tabasco) y ya tiene presencia en Facebook, Instagram, TikTok y X.

El problema no es la plataforma, es la **estrategia de distribución**: el sitio funciona como destino, pero las audiencias de hoy están en redes, video corto y Google Discover. La propuesta convierte a cada reportero en un **punto de distribución** (publica desde el celular, transmite en vivo, amplifica en sus propias redes) y convierte al sitio en el **hub** que concentra y monetiza esa atención.

---

## 2. Diagnóstico técnico

### 2.1 Stack detectado (verificado)

| Componente | Estado | Observación |
|---|---|---|
| CMS | WordPress (REST API expuesta en `/wp-json/`) | Base ideal para app móvil |
| Tema | JNews | Completo pero pesado; vigilar Core Web Vitals |
| SEO | Yoast SEO + **Yoast News SEO** | `news-sitemap.xml` activo — ya elegible para Google News |
| Analytics | Google Site Kit + gtag | GA4 y probablemente Search Console conectados |
| Infraestructura | Cloudflare, HTTP/2 + HTTP/3, TTFB ~0.5s | Buen tiempo de respuesta del origen |
| Caché de página | `cf-cache-status: DYNAMIC` | **No hay caché en el edge** — cada visita golpea el servidor |
| Imágenes | Plugin WebP converter | Correcto |
| Seguridad | AIOS (All In One Security) | Correcto |
| AMP | No (redirige 301 al canónico) | Correcto, AMP ya no es necesario |
| Schema | 1 bloque `ld+json` (Yoast básico) | Falta `NewsArticle` enriquecido, `VideoObject`, `LiveBlogPosting` |
| Ads | No se detectó AdSense en homepage | Posible oportunidad de monetización sin explotar |

### 2.2 Presencia en redes (detectada en el sitio)

- **Facebook:** `facebook.com/encambiodiario/`
- **Instagram:** `@encambiodiarioquintanaroo` — con Reels embebidos en el sitio
- **TikTok:** `@encambiodiarioo`
- **X:** `@EnCambioDiario`
- **YouTube:** no enlazado desde la portada — canal ausente o invisible
- **WhatsApp Channel / newsletter:** no detectado

### 2.3 Hallazgos y brechas

- **Identidad de marca dividida.** El news-sitemap declara la publicación como *"En Cambio Diario de Quintana Roo"* pero cubre Tabasco. Riesgo: Google News puede indexar solo una edición. Decidir: una sola marca con secciones, o dos propiedades en Publisher Center.
- **Sin caché de página en Cloudflare.** Con un pico de tráfico (nota viral, transmisión en vivo embebida) el origen puede saturarse. Activar "Cache Everything" con reglas para excluir wp-admin, o un plugin tipo WP Rocket / Super Page Cache.
- **Peso del front-end.** El HTML de portada pesa ~175 KB sin contar CSS/JS de JNews. Revisar Core Web Vitals en PageSpeed — la elegibilidad para Google Discover exige buena experiencia móvil.
- **Distribución pasiva.** Los artículos se republican manualmente (o no) en redes. No hay auto-posteo, no hay canal de YouTube enlazado, no hay WhatsApp Channel ni notificaciones push visibles.
- **Sin video propio.** El modelo de negocio de medios regionales en México se mueve a video en vivo y corto (FB Reels/TikTok) — el sitio solo embebe Reels de Instagram.
- **Schema mínimo.** Sin `VideoObject` ni `LiveBlogPosting`, el contenido audiovisual no puede aparecer en los carruseles de video ni de "En vivo" de Google.
- **Autoridad de autor (E-E-A-T).** Las notas firmadas "Por En Cambio Diario" diluyen la autoría. Google valora páginas de autor con bio, foto y redes.

---

## 3. Propuesta

### 3.1 Estrategia general: "Reportero Móvil + Hub Central"

```
   Reportero con celular
   ┌─────────┴──────────┐
   │                    │
 Nota/foto          Transmisión en vivo
 (app móvil)        (Larix/Restream → YT+FB+TikTok)
   │                    │
   ▼                    ▼
 WP REST API ──► encambiodiario.mx ◄── video embebido
   │                    │
   └── auto-posteo ──►  FB / X / IG / TikTok / WhatsApp Channel
                       de la marca y del reportero
```

### 3.2 Publicación desde celular

Opciones, de menor a mayor inversión:

- **Opción A — App oficial de WordPress/Jetpack (0–2 semanas, costo ~$0).** Cada reportero con rol `Autor` (publica directo) o `Colaborador` (requiere revisión de un editor). Subida de fotos, categorías, borradores offline. **Recomendada para el piloto.**
- **Opción B — PWA de redacción propia (4–8 semanas).** La REST API ya está expuesta y soporta *Application Passwords* (`/wp-admin/authorize-application.php`). Una PWA ligera con: título, foto destacada desde cámara, categoría por edición (QRoo/Tabasco), botón "Enviar a revisión" / "Publicar". Sin app store, se instala desde el navegador.
- **Opción C — App nativa (3+ meses).** Solo si el piloto lo justifica.

**Flujo editorial propuesto:**
1. Reportero captura nota + foto/video en el celular.
2. Envía como `pending` (revisión) si es nota sensible, o `publish` si es cobertura de agenda.
3. Editor recibe notificación (JNews ya tiene notificaciones internas; complementar con un canal de Slack/Telegram del equipo).
4. Al publicarse, **auto-posteo** a redes de la marca con Jetpack Social o plugin equivalente.

### 3.3 Transmisiones en vivo

- **Infraestructura recomendada:** app **Larix Broadcaster** (iOS/Android, gratuita) o las apps nativas de cada red. Para multistream simultáneo (YouTube + Facebook + TikTok) usar **Restream** (~$16–40 USD/mes) — el reportero transmite una vez, llega a todas las redes.
- **En el sitio:** página/sección "En Vivo" con el embed de YouTube Live o Facebook Live. Marcar la página con schema `LiveBlogPosting` + `BroadcastEvent` para que Google muestre el sello "EN VIVO" en resultados.
- **Post-transmisión:** el video queda en YouTube → se embebe en la nota → se marca con `VideoObject` → elegible para carruseles de video. Cortar clips de 30–60s para Reels/TikTok/YT Shorts.
- **Ritmo sugerido:** 2–3 transmisiones programadas por semana (conferencias mañaneras, cobertura de eventos, noticiero breve) + lives de última hora.

### 3.4 Identidad del reportero (multiplicador de alcance)

- Cada reportero con **página de autor completa** en el sitio (bio, foto, redes verificadas). JNews ya soporta esto; hay que llenarlo.
- Cada nota con **firma nominal** (no "Por En Cambio Diario") y schema `author` tipo `Person`.
- Auto-etiquetado: al republicarse en X/Facebook, mencionar la cuenta del reportero. El reportero republica desde su perfil personal → efecto red.
- Kit por reportero: gimbal + micrófono lavalier inalámbrico (tipo DJI Mic, ~$2,500–4,000 MXN) + lineamientos de estilo.

### 3.5 Visibilidad en Google (donde está el tráfico "gratis")

- **Google News / Publisher Center:** registrar la publicación (ya tienen news-sitemap, falta el registro editorial y verificar la etiqueta de sección). Resolver el tema de la doble edición QRoo/Tabasco.
- **Google Discover:** es la fuente #1 de tráfico para noticiarios regionales. Requisitos: imágenes destacadas ≥1200px (`max-image-preview:large`), titulares informativos (no clickbait), buen rendimiento móvil, contenido fresco y con autoría clara.
- **Caché en Cloudflare:** regla "Cache Everything" para HTML público (o APO for WordPress, ~$5 USD/mes). Reduce TTFB y protege el origen en picos.
- **Schema adicional:** `NewsArticle` con `dateline`, `VideoObject` en notas con video, `LiveBlogPosting` en coberturas minuto a minuto, `BreadcrumbList`.
- **Web Stories:** JNews tiene soporte nativo de Google Web Stories — formato visual que aparece en Discover. Bajo costo, buen retorno para notas de espectáculos/turismo.

### 3.6 Distribución y audiencia propia

- **Canal de WhatsApp** (gratuito, formato nativo de difusión en México): resumen matutino con links. Altísima tasa de apertura.
- **Notificaciones push:** OneSignal o el módulo push de JNews → alertas de última hora al navegador.
- **Newsletter:** JNews ya incluye `newsletterSubscribe` — activar el módulo con Mailchimp/MailPoet.
- **YouTube:** crear/activar el canal y enlazarlo desde el sitio (hoy está ausente en la portada). Es prerrequisito para la estrategia de lives.

### 3.7 Monetización (habilitada por mayor visibilidad)

- Google AdSense/Ad Manager (no se detectó en homepage) — inventario en notas + preroll en lives.
- Pauta local directa: formatos "auspiciado por" en transmisiones y esquelas comerciales.
- Contenido patrocinado marcado con `rel="sponsored"`.

---

## 4. Roadmap sugerido

| Fase | Duración | Contenido | Inversión aprox. |
|---|---|---|---|
| **0 — Fundaciones** | 2 semanas | Caché en Cloudflare, auditoría Core Web Vitals, Publisher Center, schema, páginas de autor, canal YouTube | Tiempo interno |
| **1 — Piloto móvil** | 4 semanas | App WordPress para 2–3 reporteros, flujo de revisión, auto-posteo social, kit móvil | ~$5,000–8,000 MXN (equipo) |
| **2 — En vivo** | 4 semanas | Restream, sección En Vivo + `LiveBlogPosting`, 2 lives/semana, clips verticales | ~$500–900 MXN/mes |
| **3 — Audiencia propia** | 6 semanas | Canal WhatsApp, push, newsletter, Web Stories | ~$0–600 MXN/mes |
| **4 — Escala** | continuo | PWA propia si el piloto lo justifica, Ad Manager, programa de corresponsales | a definir |

## 5. KPIs

- Tráfico orgánico y clics desde Google Discover (Search Console → filtro "Discover").
- % de notas publicadas desde móvil y tiempo nota→publicación.
- Espectadores concurrentes en lives y vistas del archivo en YouTube.
- Suscriptores: canal WhatsApp, push, newsletter.
- Ingresos por ads/pauta por cada 1,000 sesiones (RPM).

## 6. Información necesaria para afinar el diagnóstico

- Acceso a **Google Search Console** (¿aparece el reporte de Discover? ¿errores de indexación?).
- Acceso a **GA4** (fuentes de tráfico actuales: ¿% social vs orgánico?).
- Hosting actual y configuración de Cloudflare (plan, reglas de página).
- ¿Ya están dados de alta en Google News / Publisher Center?
- Número de reporteros y si ya existe flujo de revisión editorial.

---

*Diagnóstico basado en inspección externa del sitio (robots.txt, REST API, headers HTTP, sitemaps, markup de portada) realizada el 10/oct/2026. Las cifras de rendimiento son mediciones puntuales; confirmar con PageSpeed Insights y CrUX.*
