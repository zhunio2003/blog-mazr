# M A Z R — Stack Tecnológico

**Fecha:** 18 de Marzo 2026  
**Versión:** 1.0  

<div align="center"><strong>Física • Inteligencia Artificial • Computación Cuántica • Neurociencia</strong></div>

---

## 1. Resumen Ejecutivo

Este documento define y justifica cada tecnología seleccionada para construir MAZR: una plataforma de blog científico interactivo y bilingüe. Cada decisión responde a los requerimientos específicos del proyecto: contenido con simulaciones embebidas, internacionalización nativa, panel de administración custom, rendimiento élite (Lighthouse >95), y presupuesto de infraestructura <$25/mes.

Las tecnologías se organizan en 7 capas: Frontend, Backend, Contenido Interactivo, Infraestructura, Calidad de Código, Tooling de Desarrollo, y Observabilidad.

---

## 2. Criterios de Selección

Toda tecnología incluida en este stack fue evaluada bajo estos criterios, priorizados para el contexto de MAZR (proyecto solo-developer, contenido científico interactivo, presupuesto limitado):

| Prioridad | Criterio | Descripción |
|-----------|----------|-------------|
| 1 | **Fit funcional** | ¿Resuelve directamente un requerimiento de MAZR? |
| 2 | **DX (Developer Experience)** | ¿Maximiza la productividad de un solo desarrollador? |
| 3 | **Rendimiento** | ¿Contribuye al objetivo de Lighthouse >95 y carga sub-segundo? |
| 4 | **Ecosistema y comunidad** | ¿Tiene documentación sólida, comunidad activa, y mantenimiento regular? |
| 5 | **Costo** | ¿Se alinea con el presupuesto <$25/mes? |
| 6 | **Escalabilidad futura** | ¿Soporta la evolución del proyecto (más idiomas, más autores, más tráfico)? |

---

## 3. Stack por Capas

### 3.1 Frontend

| Tecnología | Versión | Propósito | Alternativas Descartadas | Justificación |
|-----------|---------|-----------|--------------------------|---------------|
| **Next.js (App Router)** | 14+ | Framework fullstack | Astro, Remix, SvelteKit | Renderizado híbrido SSR/SSG/CSR en un solo framework. SSG para posts publicados (rendimiento máximo), SSR para previews del admin, CSR para simulaciones interactivas. React Server Components reducen el JS enviado al cliente. API Routes integradas eliminan la necesidad de un backend separado. Soporte nativo de rutas `[locale]` para i18n. |
| **React** | 18+ | Librería UI | Svelte, Vue, Solid | El ecosistema de visualización científica vive en React: React Three Fiber (3D), react-p5 (simulaciones), wrappers de D3. Los componentes interactivos embebidos en MDX son componentes React — cambiar de librería eliminaría esta capacidad. Server Components + Suspense para carga progresiva. |
| **TypeScript** | 5.x (strict) | Lenguaje | JavaScript | Type safety end-to-end: desde Prisma (tipos auto-generados del schema) hasta los props de componentes interactivos (`<PendulumSim gravity={9.8} />`). Para un solo developer sin equipo que revise PRs, TypeScript actúa como revisión automática. Refactoring seguro a medida que el proyecto crece. |
| **Tailwind CSS** | 3.x | Framework de estilos | CSS Modules, Styled Components, Vanilla Extract | Utility-first alinea con el diseño minimalista de MAZR. Dark mode con strategy `class` (toggle manual). Tokens customizados en `tailwind.config.ts` para colores de categoría (`text-cat-fisica`, `bg-cat-ia`). CSS estático — sin conflicto con Server Components (a diferencia de CSS-in-JS). Purge automático mantiene el CSS <20KB en producción. |
| **Framer Motion** | 11+ | Animaciones UI | CSS transitions, GSAP, React Spring | `AnimatePresence` para animar mount/unmount (page transitions, modals). Layout animations para reordenación de listas (filtrado de posts). API declarativa React (`<motion.div>`). Para hover/focus simples se usa CSS puro — Framer Motion se reserva para animaciones que CSS no puede hacer. Tree-shakeable (~30KB solo lo que se importa). |

### 3.2 Backend / API

| Tecnología | Versión | Propósito | Alternativas Descartadas | Justificación |
|-----------|---------|-----------|--------------------------|---------------|
| **Next.js Route Handlers** | 14+ | API REST | Express.js, Fastify, tRPC | Colocados con el frontend en el mismo proyecto. Zero configuración adicional. Un solo deploy. Para un developer solo, mantener un backend separado es overhead innecesario. Los Route Handlers soportan streaming, middleware, y todo lo que MAZR necesita para CRUD de posts, comentarios, y media. |
| **PostgreSQL** | 16+ | Base de datos relacional | MySQL, SQLite, MongoDB, Supabase | Full-text search bilingüe nativo con `tsvector` (configuraciones `spanish` y `english`). Modelo relacional natural para Posts → Categories → Tags → Comments. Extensiones: `pg_trgm` para búsqueda fuzzy/autocompletado, `unaccent` para ignorar acentos en español. Self-hosted en VPS = $0 adicional. |
| **Prisma** | 5.x | ORM | Drizzle, Kysely, SQL directo | Schema como fuente de verdad única: genera tipos TypeScript, migraciones SQL, y cliente tipado automáticamente. Prisma Studio para explorar datos visualmente. `$queryRaw` disponible para queries complejas (full-text search) — lo mejor de ambos mundos. Migraciones declarativas con `prisma migrate dev`. |
| **NextAuth.js (Auth.js)** | v5 | Autenticación | Lucia Auth, Clerk, Auth0, custom JWT | Integración nativa con Next.js: middleware para proteger `/admin/*`, session helpers en Server Components y Route Handlers. Prisma Adapter almacena sesiones en PostgreSQL (sin servicio externo). Se inicia con credentials; preparado para OAuth (GitHub/Google) cuando haya colaboradores. Campo `role` en User para permisos (ADMIN, EDITOR, AUTHOR). $0 costo. |
| **Redis** | 7+ | Caché y rate limiting | In-memory Map, Upstash, sin caché | Rate limiting para API de comentarios y búsqueda (patrón `INCR` + `EXPIRE`). Caché de queries frecuentes con TTL configurable (posts: 5min, categorías: 1h). Self-hosted en Docker = $0 adicional. 50-100MB RAM es suficiente. Futuro: pub/sub para notificaciones. |
| **Zod** | 3.x | Validación de datos | Yup, Joi, validación manual | Validación type-safe de inputs en Route Handlers. Los schemas Zod se definen una vez y generan tanto validación runtime como tipos TypeScript. Ideal para validar payloads de creación/edición de posts, comentarios, y configuración. Ligero (~13KB). |

### 3.3 Contenido y Authoring

| Tecnología | Versión | Propósito | Alternativas Descartadas | Justificación |
|-----------|---------|-----------|--------------------------|---------------|
| **MDX** | 3.x | Formato de contenido | Markdown + shortcodes, CMS headless (Sanity/Strapi), HTML en DB | Permite escribir `<PendulumSim gravity={9.8} />` directamente en un artículo — el diferenciador fundamental de MAZR. Ecosistema remark/rehype para TOC autogenerado, math rendering, syntax highlighting. Bilingüe por archivo (`content/es/` y `content/en/`). Git-friendly con historial de cambios. |
| **next-intl** | 3.x | Internacionalización | next-i18next, react-intl, i18n manual | Diseñado para App Router con Server Components (mensajes resueltos en servidor). Middleware para detección automática de idioma. Type-safe message keys. Escalable: agregar idioma = nuevo JSON + config. SEO: genera `hreflang` y `alternate` links. |
| **KaTeX** | 0.16+ | Ecuaciones matemáticas | MathJax | Significativamente más rápido que MathJax en renderizado — crítico para artículos con decenas de ecuaciones (Schrödinger, Maxwell, backpropagation). Compatible con server-side rendering. Se integra via `remark-math` + `rehype-katex` en el pipeline MDX. |
| **Shiki** | 1.x | Syntax highlighting | Prism, Highlight.js | Usa los mismos grammars y temas de VS Code — colores precisos y soporte para más lenguajes. Server-side rendering (no envía JS al cliente para highlighting). Se integra via `rehype-pretty-code` en el pipeline MDX. Temas custom alineados con la paleta de MAZR. |

### 3.4 Contenido Interactivo (Visualización y Simulación)

| Tecnología | Propósito en MAZR | Alternativas Descartadas | Justificación | Estrategia de Carga |
|-----------|-------------------|--------------------------|---------------|---------------------|
| **Three.js / React Three Fiber** | Visualizaciones 3D: modelos atómicos, esferas de Bloch, redes neuronales 3D, espacios de Hilbert | Babylon.js, raw WebGL | R3F permite escribir escenas 3D como componentes React declarativos — se integran naturalmente con MDX. Babylon.js es más gaming-oriented. Comunidad y documentación superiores para visualización científica. | Dynamic import, `ssr: false`, ~150KB solo cuando se usa |
| **P5.js** | Simulaciones 2D: péndulos, ondas, campos, partículas, cellular automata | Matter.js, PhysicsJS | API simple y expresiva ideal para simulaciones educativas. Canvas nativo, rendering rápido. Gran comunidad de creative coding con ejemplos científicos. Matter.js es más para juegos con motor de física. | Dynamic import, `ssr: false`, ~80KB solo cuando se usa |
| **D3.js** | Gráficas interactivas: árboles de decisión, visualizaciones de atención (transformers), datos estadísticos | Chart.js, Recharts, Plotly | Control total sobre el SVG — necesario para visualizaciones custom como mapas de atención de transformers o árboles de decisión interactivos. Recharts/Chart.js son demasiado alto nivel para este tipo de visualizaciones. | Dynamic import, ~70KB solo cuando se usa |
| **Canvas API / WebGL** | Simulaciones de alto rendimiento: fluidos, N-body, cellular automata | Ninguna (es la API nativa) | Para simulaciones que requieren >60fps con miles de partículas, canvas/WebGL directo evita el overhead de abstracción. Se usa solo cuando P5.js o Three.js no son suficientemente performantes. | Custom wrapper component, `ssr: false` |
| **Custom SVG/Canvas** | Circuitos cuánticos interactivos (editor drag-and-drop) | Qiskit.js | Qiskit.js es limitado para rendering interactivo con drag-and-drop. Un editor custom con SVG/Canvas da control total sobre UX: arrastrar compuertas, conectar qubits, ver estado en tiempo real. | Dynamic import, `ssr: false` |
| **Mermaid** | Diagramas de flujo, arquitecturas, workflows dentro de artículos | Draw.io embed, custom SVG | Se integra directamente con MDX via plugin. Ideal para diagramas explicativos en artículos (arquitectura de redes neuronales, flujo de algoritmos cuánticos). Sintaxis simple de texto. | Renderizado server-side cuando posible |

**Nota sobre bundle size:** Ninguna de estas librerías se incluye en el bundle inicial. Todas se cargan con `next/dynamic` + `ssr: false` solo cuando un artículo las necesita. Esto mantiene el JS inicial <150KB.

### 3.5 Infraestructura

| Tecnología | Propósito | Alternativas Descartadas | Justificación | Costo Estimado |
|-----------|-----------|--------------------------|---------------|----------------|
| **VPS (Hetzner / DigitalOcean)** | Servidor principal | Vercel, Railway, Render, AWS EC2 | Control total del entorno. Costo fijo y predecible ($5-20/mes para 4GB RAM). Sin vendor lock-in — migración = mover Docker containers. Aloja app + PostgreSQL + Redis en una sola máquina. Vercel escala en costo rápido y requiere DB/Redis externos pagados. | $5-20/mes |
| **Docker + Docker Compose** | Containerización | Instalación directa en VPS, Kubernetes | Un solo `docker-compose.prod.yml` define toda la infra. Entorno reproducible entre dev y prod. Simplifica el deploy a un `docker compose up -d`. Kubernetes es overkill para un blog. | Incluido |
| **Nginx** | Reverse proxy + SSL + static files | Traefik, Caddy | Probado en producción por décadas. Configuración bien documentada. Sirve assets estáticos directamente sin pasar por Node.js. SSL via Cloudflare o Let's Encrypt/Certbot. | Incluido |
| **Cloudflare** | CDN + DNS + DDoS protection + SSL | AWS CloudFront, Fastly | Tier gratuito incluye CDN global, protección DDoS, DNS rápido, y SSL. Cache de assets estáticos en edge. Page Rules para optimizar cacheo de posts. Analytics básico incluido. | Gratis |
| **GitHub Actions** | CI/CD | GitLab CI, Jenkins, CircleCI | Integrado con el repo. Pipeline: push a `main` → lint + test → build Docker image → deploy al VPS via SSH. Preview deploys en PRs. Gratis para repos públicos, 2000 min/mes para privados. | Gratis |
| **S3 / MinIO** | Almacenamiento de imágenes y media | Cloudinary, local filesystem | MinIO (self-hosted) para $0 si el VPS tiene espacio. S3 como alternativa managed si se necesita. Separar media del filesystem de la app simplifica backups y permite CDN directo. | $0-5/mes |

**Costo total estimado de infraestructura: $6-25/mes**

### 3.6 Calidad de Código

| Tecnología | Propósito | Justificación |
|-----------|-----------|---------------|
| **ESLint** | Linting de código | Reglas de Next.js (`eslint-config-next`), accesibilidad (`jsx-a11y`), imports ordenados. Detecta errores antes de compilación. |
| **Prettier** | Formateo de código | Formato consistente automático. Elimina debates de estilo. Integrado con ESLint via `eslint-config-prettier`. |
| **Husky + lint-staged** | Git hooks | Pre-commit: lint y format solo archivos modificados. Pre-push: type-check completo. Previene código con errores en el repo. |
| **Jest / Vitest** | Unit testing | Tests de utilidades (`reading-time`, `slugify`, `date`), lógica de API, y helpers de MDX. Vitest es más rápido y compatible con ESM. |
| **Playwright** | E2E testing | Tests del flujo completo: navegación del blog, cambio de idioma, búsqueda, panel admin. Cross-browser (Chromium, Firefox, WebKit). |

### 3.7 Tooling de Desarrollo

| Tecnología | Propósito | Justificación |
|-----------|-----------|---------------|
| **pnpm** | Package manager | Almacenamiento eficiente (content-addressable store) — importante con dependencias pesadas como Three.js y D3. Instalación más rápida que npm. Strict mode previene imports de dependencias no declaradas. |
| **Prisma Studio** | Explorador de base de datos | UI visual para inspeccionar y editar datos durante desarrollo. Gratis, incluido con Prisma CLI. |
| **Docker Compose (dev)** | Entorno de desarrollo | PostgreSQL + Redis levantados con un solo comando. Entorno idéntico entre dev y prod. No requiere instalar servicios localmente. |
| **VS Code** | Editor | Extensions: Tailwind CSS IntelliSense, Prisma, ESLint, Pretty TypeScript Errors, MDX. Configuración compartida en `.vscode/settings.json`. |

### 3.8 Observabilidad (Post-Launch)

| Tecnología | Propósito | Justificación | Costo |
|-----------|-----------|---------------|-------|
| **Cloudflare Analytics** | Tráfico y rendimiento | Incluido con Cloudflare free. Métricas de tráfico sin JS adicional en el cliente (server-side). Privacy-friendly. | Gratis |
| **Sentry** | Error tracking | Captura errores en producción con stack trace, contexto, y breadcrumbs. Free tier: 5K eventos/mes — más que suficiente para MAZR. SDK de Next.js oficial. | Gratis (free tier) |
| **UptimeRobot / Uptime Kuma** | Monitoring de uptime | Uptime Kuma self-hosted en el VPS o UptimeRobot free tier. Alertas cuando el sitio cae. Health check endpoint en `/api/health`. | Gratis |

---

## 4. Diagrama de Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLOUDFLARE                                 │
│                   CDN · DNS · DDoS · SSL                            │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                        VPS (Hetzner / DO)                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     Docker Compose                             │ │
│  │                                                                │ │
│  │  ┌──────────┐   ┌──────────────────────────────────────────┐  │ │
│  │  │  Nginx   │──▶│           Next.js 14+ (App Router)       │  │ │
│  │  │  Proxy   │   │                                          │  │ │
│  │  │  + SSL   │   │  ┌─────────────┐  ┌──────────────────┐  │  │ │
│  │  └──────────┘   │  │   Frontend   │  │    Backend        │  │  │ │
│  │                  │  │ React 18     │  │  Route Handlers   │  │  │ │
│  │                  │  │ Tailwind CSS │  │  NextAuth.js v5   │  │  │ │
│  │                  │  │ Framer Motion│  │  Prisma ORM       │  │  │ │
│  │                  │  │ next-intl    │  │  Zod              │  │  │ │
│  │                  │  │ MDX + KaTeX  │  │                   │  │  │ │
│  │                  │  │ Shiki        │  │                   │  │  │ │
│  │                  │  └─────────────┘  └────────┬─────┬────┘  │  │ │
│  │                  │                            │     │       │  │ │
│  │                  │  ┌─────────────────────┐   │     │       │  │ │
│  │                  │  │ Interactive Layer    │   │     │       │  │ │
│  │                  │  │ Three.js/R3F · P5.js │   │     │       │  │ │
│  │                  │  │ D3.js · Canvas/WebGL │   │     │       │  │ │
│  │                  │  │ (dynamic imports)    │   │     │       │  │ │
│  │                  │  └─────────────────────┘   │     │       │  │ │
│  │                  └────────────────────────────┘     │       │  │ │
│  │                                                     │       │  │ │
│  │  ┌──────────────────────┐    ┌─────────────────────┐│       │  │ │
│  │  │   PostgreSQL 16+     │◀───│      Redis 7+       ││       │  │ │
│  │  │   Full-text search   │    │   Caché + Rate      ││       │  │ │
│  │  │   pg_trgm · unaccent │    │   Limiting          ││       │  │ │
│  │  └──────────────────────┘    └─────────────────────┘│       │  │ │
│  │                                                     │       │  │ │
│  │  ┌──────────────────────┐                           │       │  │ │
│  │  │   S3 / MinIO         │◀──────────────────────────┘       │  │ │
│  │  │   Media storage      │                                   │  │ │
│  │  └──────────────────────┘                                   │  │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        CI/CD & Tooling                              │
│  GitHub Actions · pnpm · ESLint · Prettier · Husky · Playwright    │
│  Sentry · Cloudflare Analytics · UptimeRobot                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Versiones y Compatibilidad

| Tecnología | Versión mínima | Notas |
|-----------|---------------|-------|
| Node.js | 20 LTS | Requerido por Next.js 14+ |
| pnpm | 9.x | Content-addressable store |
| Next.js | 14.2+ | App Router estable, Server Actions |
| React | 18.3+ | Server Components, Suspense |
| TypeScript | 5.4+ | Strict mode habilitado |
| Tailwind CSS | 3.4+ | `dark:` class strategy |
| Prisma | 5.10+ | PostgreSQL full-text support |
| PostgreSQL | 16+ | `tsvector`, `pg_trgm`, `unaccent` |
| Redis | 7+ | Compatibilidad con ioredis/redis client |
| Docker | 24+ | Compose V2 integrado |
| Nginx | 1.24+ | HTTP/2, reverse proxy |

---

## 6. Restricciones y Decisiones Explícitas de Exclusión

Tecnologías evaluadas y deliberadamente **no incluidas** en el stack:

| Tecnología | Razón de exclusión |
|-----------|-------------------|
| **WordPress** | MAZR es custom por diseño. WordPress no soporta componentes React interactivos embebidos en contenido. |
| **CMS headless (Sanity, Strapi, Contentful)** | No soportan JSX/React components en contenido. Agregan dependencia externa y costo. MDX cubre esta necesidad. |
| **Vercel (hosting)** | Costo escala rápido con tráfico. PostgreSQL y Redis requieren servicios externos pagados ($20-40/mes adicional). VPS es más económico y da control total. |
| **Elasticsearch / Algolia** | PostgreSQL full-text search es suficiente para el volumen de contenido de MAZR (<1000 posts). Agrega complejidad y costo innecesario. |
| **GraphQL** | REST es suficiente para el modelo de datos de MAZR. GraphQL agrega complejidad (schema, resolvers, codegen) sin beneficio claro para un blog con panel admin. |
| **Kubernetes** | Overkill total. Docker Compose es suficiente para un solo VPS. K8s se justifica con múltiples servicios a escala — MAZR es un monolito. |
| **Tailwind UI / shadcn/ui** | MAZR tiene un sistema de diseño propio y minimalista. Usar un component library prearmada diluiría la identidad visual. Componentes UI se construyen custom. |
| **tRPC** | Requiere acoplamiento end-to-end entre cliente y servidor. Route Handlers estándar son más flexibles si en el futuro se expone la API a terceros. |

---

## 7. Estrategia de Actualización

| Frecuencia | Acción |
|-----------|--------|
| **Semanal** | `pnpm audit` para vulnerabilidades de seguridad. Dependabot PRs para patches. |
| **Mensual** | Actualizar dependencias menores. Revisar changelogs de Next.js, Prisma, y Tailwind. |
| **Trimestral** | Evaluar actualizaciones mayores. Revisar si alguna tecnología del stack tiene mejor alternativa. Actualizar este documento. |

---