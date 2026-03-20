# M A Z R — Arquitectura Detallada

**Fecha:** 20 de Marzo 2026  
**Versión:** 1.0  


<div align="center"><strong>Física • Inteligencia Artificial • Computación Cuántica • Neurociencia</strong></div>

---

## Índice

1. [Introducción](#1-introducción)
2. [Principios Arquitectónicos](#2-principios-arquitectónicos)
3. [Vista General del Sistema](#3-vista-general-del-sistema)
4. [Capa de Presentación (Frontend)](#4-capa-de-presentación-frontend)
5. [Capa de Aplicación (Backend / API)](#5-capa-de-aplicación-backend--api)
6. [Capa de Datos](#6-capa-de-datos)
7. [Pipeline de Contenido (MDX)](#7-pipeline-de-contenido-mdx)
8. [Sistema Interactivo](#8-sistema-interactivo)
9. [Sistema de Internacionalización (i18n)](#9-sistema-de-internacionalización-i18n)
10. [Sistema de Autenticación y Autorización](#10-sistema-de-autenticación-y-autorización)
11. [Sistema de Búsqueda](#11-sistema-de-búsqueda)
12. [Sistema de Comentarios](#12-sistema-de-comentarios)
13. [Sistema de Diseño](#13-sistema-de-diseño)
14. [Infraestructura y Deployment](#14-infraestructura-y-deployment)
15. [Observabilidad y Monitoreo](#15-observabilidad-y-monitoreo)
16. [Seguridad](#16-seguridad)
17. [Rendimiento y Optimización](#17-rendimiento-y-optimización)
18. [Decisiones Arquitectónicas (ADRs)](#18-decisiones-arquitectónicas-adrs)
19. [Flujos de Datos Principales](#19-flujos-de-datos-principales)
20. [Estrategia de Escalabilidad](#20-estrategia-de-escalabilidad)

---

## 1. Introducción

### 1.1 Propósito de este Documento

Este documento define la arquitectura técnica completa de MAZR. Sirve como referencia para todas las decisiones de diseño, la interacción entre componentes, los patrones utilizados, y la justificación detrás de cada elección. Es un documento vivo que evoluciona con el proyecto.

### 1.2 Alcance

Cubre la arquitectura de la versión 1.0 de MAZR: una plataforma de blog científico interactivo y bilingüe (español/inglés) con panel de administración custom, desplegada en VPS con Docker.

### 1.3 Contexto del Proyecto

MAZR es un proyecto solo-developer con ambición profesional. Las decisiones arquitectónicas priorizan productividad individual, rendimiento (Lighthouse >95), y extensibilidad futura sin sobre-ingeniería prematura.

---

## 2. Principios Arquitectónicos

Estos principios guían todas las decisiones técnicas del proyecto. Cuando haya conflicto, se resuelven en el orden de prioridad listado.

| # | Principio | Descripción | Ejemplo en MAZR |
|---|-----------|-------------|-----------------|
| 1 | **Ship over perfection** | Un sistema funcional que se puede iterar siempre gana sobre un diseño perfecto que nunca se entrega. | Fase 1 usa archivos MDX locales antes de implementar el admin panel. |
| 2 | **Monolito primero** | No distribuir hasta que el dolor lo justifique. La separación lógica interna permite extraer servicios después. | Next.js App Router como monolito fullstack: frontend + API + admin en un solo deploy. |
| 3 | **Type safety end-to-end** | Los tipos fluyen desde la base de datos hasta la UI sin conversiones manuales. | Prisma genera tipos → Route Handlers tipados → componentes React con props tipados. |
| 4 | **Rendimiento por defecto** | Las optimizaciones de rendimiento se aplican a nivel de arquitectura, no como parches después. | SSG para posts, dynamic imports para simulaciones, Server Components para reducir JS. |
| 5 | **Contenido como código** | Los artículos viven en el repositorio, con historial de cambios, revisión, y CI. | Archivos MDX en `content/`, versionados en Git, con pipeline de compilación. |
| 6 | **Separación de responsabilidades** | Cada módulo tiene una responsabilidad clara. Las dependencias fluyen hacia adentro. | `lib/` contiene lógica de negocio pura, `components/` solo presentación, `app/` solo routing. |

---

## 3. Vista General del Sistema

### 3.1 Diagrama de Arquitectura de Alto Nivel

```
                         ┌────────────────────────┐
                         │       INTERNET          │
                         └───────────┬─────────────┘
                                     │
                         ┌───────────▼─────────────┐
                         │       CLOUDFLARE         │
                         │  CDN · DNS · DDoS · SSL  │
                         │  Cache de assets estáticos│
                         └───────────┬─────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │        VPS (Docker Compose)      │
                    │                                  │
                    │  ┌──────────┐  ┌──────────────┐  │
                    │  │  Nginx   │─▶│   Next.js     │  │
                    │  │  :80/443 │  │   :3000       │  │
                    │  └──────────┘  └──────┬───────┘  │
                    │                       │          │
                    │            ┌──────────┼────────┐ │
                    │            │          │        │ │
                    │  ┌─────────▼──┐  ┌───▼────┐   │ │
                    │  │ PostgreSQL │  │ Redis  │   │ │
                    │  │   :5432    │  │ :6379  │   │ │
                    │  └────────────┘  └────────┘   │ │
                    │                               │ │
                    │  ┌────────────────────────────┘ │
                    │  │  MinIO / S3                   │
                    │  │  :9000 (media storage)        │
                    │  └──────────────────────────────┘│
                    └──────────────────────────────────┘
                    
                    ┌──────────────────────────────────┐
                    │         GitHub Actions            │
                    │  CI: lint + test + build          │
                    │  CD: Docker build → deploy via SSH│
                    └──────────────────────────────────┘
```

### 3.2 Componentes Principales

| Componente | Tecnología | Responsabilidad |
|------------|-----------|-----------------|
| **Reverse Proxy** | Nginx | Enrutar tráfico, SSL termination, servir assets estáticos, compresión gzip/brotli |
| **Aplicación** | Next.js 14+ (App Router) | Renderizar páginas, servir API REST, procesar MDX, manejar autenticación |
| **Base de datos** | PostgreSQL 16+ | Persistencia de datos estructurados, búsqueda full-text bilingüe |
| **Cache** | Redis 7+ | Caché de queries, rate limiting, sesiones |
| **Media Storage** | MinIO / S3 | Almacenamiento de imágenes y assets de artículos |
| **CDN** | Cloudflare | Cache global de assets, protección DDoS, DNS, SSL edge |
| **CI/CD** | GitHub Actions | Tests automatizados, build de Docker image, deploy a producción |

### 3.3 Estrategia de Renderizado

MAZR usa renderizado híbrido — cada tipo de página usa la estrategia óptima:

| Página | Estrategia | Razón |
|--------|-----------|-------|
| Blog post (publicado) | **SSG** (Static Site Generation) | Rendimiento máximo, cacheado en CDN. Se regenera al editar. |
| Listado de posts | **SSG + ISR** (Incremental Static Regeneration) | Estático con revalidación cada 5 minutos para nuevos posts. |
| Página de categoría | **SSG + ISR** | Mismo patrón que listado. |
| Búsqueda | **SSR** (Server-Side Rendering) | Depende del query del usuario, no se puede pre-generar. |
| Panel admin | **CSR** (Client-Side Rendering) | Contenido dinámico detrás de autenticación, no necesita SEO. |
| Preview de post | **SSR** | Editor necesita ver el resultado en tiempo real. |
| Homepage | **SSG + ISR** | Contenido semi-estático con posts recientes. Revalida cada 10 minutos. |

---

## 4. Capa de Presentación (Frontend)

### 4.1 Estructura de Páginas (App Router)

El App Router de Next.js 14+ organiza las páginas en una jerarquía de carpetas. Cada carpeta puede contener `layout.tsx`, `page.tsx`, `loading.tsx`, y `error.tsx`.

```
src/app/
├── layout.tsx                 ← Root Layout: providers globales, fonts, metadata base
├── globals.css                ← Tailwind imports + CSS custom properties
├── not-found.tsx              ← 404 global
├── error.tsx                  ← Error boundary global
│
├── [locale]/                  ← Segmento dinámico de idioma (es | en)
│   ├── layout.tsx             ← Layout con LocaleProvider (next-intl)
│   ├── page.tsx               ← Homepage
│   ├── blog/
│   │   ├── page.tsx           ← Listado con filtros y paginación
│   │   └── [slug]/
│   │       ├── page.tsx       ← Renderizado de post MDX + hidratación interactiva
│   │       └── loading.tsx    ← Skeleton mientras carga
│   ├── category/[slug]/
│   │   └── page.tsx           ← Posts filtrados por categoría
│   ├── tags/[slug]/
│   │   └── page.tsx           ← Posts filtrados por tag
│   ├── search/
│   │   └── page.tsx           ← Búsqueda full-text con resultados en vivo
│   └── about/
│       └── page.tsx           ← Sobre MAZR y autores
│
├── admin/                     ← Panel de administración (sin [locale], sin SEO)
│   ├── layout.tsx             ← Sidebar + auth guard middleware
│   ├── page.tsx               ← Dashboard con métricas
│   ├── posts/                 ← CRUD de posts
│   ├── categories/            ← Gestión de categorías
│   ├── tags/                  ← Gestión de tags
│   ├── comments/              ← Moderación
│   ├── media/                 ← Media library
│   ├── users/                 ← Gestión de roles
│   └── settings/              ← Configuración del sitio
│
└── api/                       ← Route Handlers (REST API)
    ├── auth/[...nextauth]/    ← Endpoints de NextAuth
    ├── posts/                 ← CRUD de posts
    ├── categories/            ← CRUD de categorías
    ├── tags/                  ← CRUD de tags
    ├── comments/              ← CRUD de comentarios
    ├── media/                 ← Upload y gestión
    ├── search/                ← Endpoint de búsqueda
    ├── feed/                  ← Generación de RSS
    └── health/                ← Health check
```

### 4.2 Jerarquía de Layouts

Los layouts en Next.js App Router se anidan automáticamente. MAZR define tres niveles:

```
Root Layout (src/app/layout.tsx)
│  → ThemeProvider (dark mode)
│  → Fonts (Space Grotesk, Source Serif 4, JetBrains Mono, Inter)
│  → Metadata base
│
├── Locale Layout (src/app/[locale]/layout.tsx)
│   │  → NextIntlClientProvider (traducciones)
│   │  → Navbar (categorías, buscador, idioma, dark mode)
│   │  → Footer
│   │  → ScrollProgress (barra de lectura)
│   │
│   ├── Blog Post Layout (implícito en page.tsx)
│   │   → PostReader (max-width 720px)
│   │   → TableOfContents (sidebar sticky)
│   │   → RelatedPosts
│   │   → CommentSection
│   │
│   └── Otras páginas públicas
│
└── Admin Layout (src/app/admin/layout.tsx)
    → Auth guard (redirect si no autenticado)
    → Sidebar (navegación admin)
    → Sin Navbar/Footer públicos
```

### 4.3 Componentes: Taxonomía y Responsabilidades

Los componentes se organizan por dominio, no por tipo técnico:

| Directorio | Responsabilidad | Server/Client | Ejemplo |
|------------|----------------|---------------|---------|
| `components/ui/` | Primitivos del design system. Sin lógica de negocio. | Ambos | `Button`, `Badge`, `Card`, `Modal`, `Skeleton` |
| `components/layout/` | Estructura visual de la aplicación. | Principalmente Client | `Navbar`, `Footer`, `Sidebar`, `LanguageSwitcher`, `ThemeProvider` |
| `components/blog/` | Todo lo relacionado con la visualización de posts. | Ambos | `PostCard`, `PostReader`, `TableOfContents`, `ReadingTime` |
| `components/admin/` | Componentes exclusivos del panel de administración. | Client | `MdxEditor`, `Toolbar`, `BilinguaEditor`, `DashboardStats` |
| `components/comments/` | Sistema de comentarios con hilos. | Client | `CommentSection`, `CommentThread`, `CommentForm` |
| `components/search/` | Búsqueda instantánea con filtros. | Client | `SearchBar`, `SearchResults`, `SearchFilters` |
| `components/interactive/` | Simulaciones y visualizaciones científicas embebidas en artículos. | Client (dynamic) | `PendulumSim`, `NeuralNetViz`, `BlochSphere` |
| `components/mdx/` | Componentes que MDX puede usar dentro de artículos. | Ambos | `CodeBlock`, `Callout`, `Equation`, `Figure` |
| `components/seo/` | Metadata, structured data, Open Graph. | Server | `MetaTags`, `JsonLd`, `OgImage` |

### 4.4 Server Components vs Client Components

Next.js App Router usa Server Components por defecto. MAZR los aprovecha estratégicamente:

**Server Components (por defecto, sin `"use client"`):**
Componentes que solo renderizan HTML sin interactividad. No envían JavaScript al navegador. Se usan para: layout de posts, listados, metadata, SEO, contenido estático de MDX.

**Client Components (`"use client"` explícito):**
Componentes que necesitan estado, efectos, o event handlers. Se usan para: simulaciones interactivas, buscador con debounce, comentarios, dark mode toggle, editor admin, selector de idioma.

**Patrón de composición:** Un Server Component puede importar Client Components, pero no al revés. Esto permite que la mayoría de la página sea estática (0 JS) con islas de interactividad:

```
PostPage (Server Component - 0 JS)
├── PostHeader (Server - 0 JS)
├── MDXContent (Server - renderiza HTML estático)
│   ├── <p>, <h2>, etc. (HTML puro)
│   ├── <Equation /> (Server - KaTeX pre-renderizado)
│   └── <PendulumSim /> (Client - dynamic import, JS solo aquí)
├── TableOfContents (Client - intersection observer)
└── CommentSection (Client - formulario interactivo)
```

### 4.5 Hooks Personalizados

| Hook | Propósito | Dependencia |
|------|-----------|-------------|
| `useTheme` | Toggle dark/light mode, persistencia en localStorage | ThemeProvider |
| `useLocale` | Acceso al idioma actual y función de cambio | next-intl |
| `useDebounce` | Debounce de valores (búsqueda, inputs) | Ninguna |
| `useIntersection` | Intersection Observer para lazy loading y TOC activo | Ninguna |
| `useReadingProgress` | Progreso de lectura del artículo (scroll position) | Ninguna |
| `useMediaQuery` | Breakpoints reactivos para diseño responsivo | Ninguna |

---

## 5. Capa de Aplicación (Backend / API)

### 5.1 Arquitectura de la API REST

La API vive en `src/app/api/` como Route Handlers de Next.js. Cada endpoint sigue el patrón REST estándar:

| Recurso | Método | Ruta | Descripción | Auth |
|---------|--------|------|-------------|------|
| Posts | `GET` | `/api/posts` | Listar posts (paginado, filtrable) | Público |
| Posts | `GET` | `/api/posts/[id]` | Detalle de un post | Público |
| Posts | `POST` | `/api/posts` | Crear post | Admin/Editor |
| Posts | `PUT` | `/api/posts/[id]` | Actualizar post | Admin/Editor/Autor |
| Posts | `DELETE` | `/api/posts/[id]` | Eliminar post | Admin |
| Categories | `GET` | `/api/categories` | Listar categorías | Público |
| Categories | `POST` | `/api/categories` | Crear categoría | Admin |
| Tags | `GET` | `/api/tags` | Listar tags | Público |
| Tags | `POST` | `/api/tags` | Crear tag | Admin/Editor |
| Comments | `GET` | `/api/comments?postId=X` | Comentarios de un post | Público |
| Comments | `POST` | `/api/comments` | Crear comentario | Público (rate limited) |
| Comments | `PUT` | `/api/comments/[id]` | Moderar comentario | Admin |
| Search | `GET` | `/api/search?q=X&lang=Y` | Búsqueda full-text | Público |
| Media | `POST` | `/api/media` | Subir archivo | Admin/Editor |
| Feed | `GET` | `/api/feed?lang=X&cat=Y` | RSS feed | Público |
| Health | `GET` | `/api/health` | Health check | Público |

### 5.2 Anatomía de un Route Handler

Cada Route Handler sigue un patrón consistente con validación, autenticación, manejo de errores, y caché:

```typescript
// src/app/api/posts/route.ts — Ejemplo de estructura

import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { z } from 'zod'
import { prisma } from '@/lib/db/prisma'
import { redis } from '@/lib/db/redis'
import { authConfig } from '@/lib/auth/config'
import { ApiError, handleApiError } from '@/lib/api/errors'
import { rateLimit } from '@/lib/api/rate-limit'

// 1. Schema de validación (Zod)
const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  slug: z.string().regex(/^[a-z0-9-]+$/),
  content_es: z.string().min(1),
  content_en: z.string().min(1),
  categoryId: z.string().uuid(),
  tags: z.array(z.string().uuid()).optional(),
  status: z.enum(['DRAFT', 'PUBLISHED', 'SCHEDULED']).default('DRAFT'),
})

// 2. GET — Listado público (con caché Redis)
export async function GET(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url)
    const page = parseInt(searchParams.get('page') ?? '1')
    const limit = parseInt(searchParams.get('limit') ?? '10')
    const category = searchParams.get('category')
    const lang = searchParams.get('lang') ?? 'es'

    // Intentar caché
    const cacheKey = `posts:list:${lang}:${category}:${page}`
    const cached = await redis.get(cacheKey)
    if (cached) return NextResponse.json(JSON.parse(cached))

    // Query a base de datos
    const posts = await prisma.post.findMany({
      where: {
        status: 'PUBLISHED',
        ...(category && { category: { slug: category } }),
      },
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { publishedAt: 'desc' },
      include: { category: true, tags: true, author: true },
    })

    const response = { data: posts, page, limit }

    // Guardar en caché (5 minutos)
    await redis.setex(cacheKey, 300, JSON.stringify(response))

    return NextResponse.json(response)
  } catch (error) {
    return handleApiError(error)
  }
}

// 3. POST — Crear post (autenticado, validado)
export async function POST(request: NextRequest) {
  try {
    const session = await getServerSession(authConfig)
    if (!session?.user || !['ADMIN', 'EDITOR'].includes(session.user.role)) {
      throw new ApiError(403, 'Insufficient permissions')
    }

    const body = await request.json()
    const data = createPostSchema.parse(body)

    const post = await prisma.post.create({
      data: {
        ...data,
        authorId: session.user.id,
        tags: data.tags
          ? { connect: data.tags.map(id => ({ id })) }
          : undefined,
      },
    })

    // Invalidar caché de listados
    await redis.del('posts:list:*')

    return NextResponse.json(post, { status: 201 })
  } catch (error) {
    return handleApiError(error)
  }
}
```

### 5.3 Manejo de Errores Estandarizado

Todos los errores de API pasan por un handler centralizado en `lib/api/errors.ts`:

| Código | Tipo | Cuándo |
|--------|------|--------|
| 400 | `ValidationError` | Zod parse falla, datos inválidos |
| 401 | `AuthenticationError` | No hay sesión activa |
| 403 | `AuthorizationError` | Rol insuficiente |
| 404 | `NotFoundError` | Recurso no existe |
| 409 | `ConflictError` | Slug duplicado, violación de unicidad |
| 429 | `RateLimitError` | Demasiadas requests (Redis counter) |
| 500 | `InternalError` | Error inesperado (loggeado a Sentry) |

La respuesta siempre tiene la estructura:

```json
{
  "error": {
    "code": 400,
    "type": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": [
      { "field": "slug", "message": "Must be lowercase alphanumeric with hyphens" }
    ]
  }
}
```

### 5.4 Rate Limiting

Se implementa con Redis usando el patrón sliding window:

| Endpoint | Límite | Ventana | Razón |
|----------|--------|---------|-------|
| `POST /api/comments` | 5 requests | 1 minuto | Anti-spam |
| `GET /api/search` | 30 requests | 1 minuto | Proteger PostgreSQL de queries pesadas |
| `POST /api/media` | 10 uploads | 5 minutos | Limitar uso de storage |
| `POST /api/auth/*` | 10 intentos | 15 minutos | Anti-brute-force |

---

## 6. Capa de Datos

### 6.1 Modelo de Datos Completo

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── USUARIOS ────────────────────────────────────────────

enum Role {
  ADMIN
  EDITOR
  AUTHOR
}

model User {
  id            String    @id @default(cuid())
  name          String
  email         String    @unique
  emailVerified DateTime?
  image         String?
  bio_es        String?
  bio_en        String?
  role          Role      @default(AUTHOR)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  posts         Post[]
  accounts      Account[]
  sessions      Session[]

  @@map("users")
}

// ─── AUTENTICACIÓN (NextAuth) ────────────────────────────

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
  @@map("accounts")
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("sessions")
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
  @@map("verification_tokens")
}

// ─── CONTENIDO ───────────────────────────────────────────

enum PostStatus {
  DRAFT
  PUBLISHED
  SCHEDULED
  ARCHIVED
}

model Post {
  id            String     @id @default(cuid())
  slug          String     @unique
  title_es      String
  title_en      String
  excerpt_es    String?
  excerpt_en    String?
  content_es    String     // MDX content en español
  content_en    String     // MDX content en inglés
  coverImage    String?
  status        PostStatus @default(DRAFT)
  publishedAt   DateTime?
  scheduledAt   DateTime?
  readingTime   Int?       // minutos estimados
  featured      Boolean    @default(false)
  createdAt     DateTime   @default(now())
  updatedAt     DateTime   @updatedAt

  authorId      String
  categoryId    String

  author        User       @relation(fields: [authorId], references: [id])
  category      Category   @relation(fields: [categoryId], references: [id])
  tags          Tag[]
  comments      Comment[]
  media         Media[]

  // Índices para búsqueda y rendimiento
  @@index([status, publishedAt(sort: Desc)])
  @@index([categoryId])
  @@index([authorId])
  @@index([slug])
  @@map("posts")
}

model Category {
  id            String  @id @default(cuid())
  slug          String  @unique
  name_es       String
  name_en       String
  description_es String?
  description_en String?
  color         String  // Hex color (ej: #10B981)
  icon          String? // Nombre del icono o SVG path
  order         Int     @default(0)

  posts         Post[]

  @@map("categories")
}

model Tag {
  id    String @id @default(cuid())
  slug  String @unique
  name  String

  posts Post[]

  @@map("tags")
}

// ─── COMENTARIOS ─────────────────────────────────────────

enum CommentStatus {
  PENDING
  APPROVED
  REJECTED
  SPAM
}

model Comment {
  id          String        @id @default(cuid())
  content     String
  authorName  String
  authorEmail String
  status      CommentStatus @default(PENDING)
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  postId      String
  parentId    String?       // Para hilos de respuestas

  post        Post          @relation(fields: [postId], references: [id], onDelete: Cascade)
  parent      Comment?      @relation("CommentReplies", fields: [parentId], references: [id])
  replies     Comment[]     @relation("CommentReplies")

  @@index([postId, status])
  @@index([parentId])
  @@map("comments")
}

// ─── MEDIA ───────────────────────────────────────────────

enum MediaType {
  IMAGE
  VIDEO
  DOCUMENT
  OTHER
}

model Media {
  id        String    @id @default(cuid())
  url       String
  altText   String?
  type      MediaType @default(IMAGE)
  size      Int       // bytes
  width     Int?
  height    Int?
  createdAt DateTime  @default(now())

  postId    String?
  post      Post?     @relation(fields: [postId], references: [id])

  @@map("media")
}

// ─── CONFIGURACIÓN DEL SITIO ─────────────────────────────

model SiteConfig {
  id    String @id @default("default")
  key   String @unique
  value String // JSON serializado

  @@map("site_config")
}
```

### 6.2 Extensiones PostgreSQL Requeridas

```sql
-- init.sql (ejecutado al crear la base de datos)

-- Full-text search con configuraciones de idioma
CREATE EXTENSION IF NOT EXISTS unaccent;

-- Búsqueda fuzzy (autocompletado, tolerancia a typos)
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Índices de búsqueda full-text (se crean post-migración)
CREATE INDEX IF NOT EXISTS idx_posts_search_es
  ON posts USING GIN (to_tsvector('spanish', title_es || ' ' || COALESCE(excerpt_es, '')));

CREATE INDEX IF NOT EXISTS idx_posts_search_en
  ON posts USING GIN (to_tsvector('english', title_en || ' ' || COALESCE(excerpt_en, '')));

-- Índice trigram para autocompletado
CREATE INDEX IF NOT EXISTS idx_posts_title_trgm_es ON posts USING GIN (title_es gin_trgm_ops);
CREATE INDEX IF NOT EXISTS idx_posts_title_trgm_en ON posts USING GIN (title_en gin_trgm_ops);
```

### 6.3 Estrategia de Caché (Redis)

| Clave | TTL | Contenido | Invalidación |
|-------|-----|-----------|--------------|
| `posts:list:{lang}:{category}:{page}` | 5 min | Listado paginado de posts | Al crear/editar/eliminar post |
| `posts:detail:{slug}` | 10 min | Post individual con relaciones | Al editar ese post |
| `categories:all` | 1 hora | Lista de categorías | Al crear/editar categoría |
| `tags:all` | 1 hora | Lista de tags | Al crear/editar tag |
| `search:{lang}:{query}:{page}` | 2 min | Resultados de búsqueda | Por TTL (no se invalida manualmente) |
| `rate:{ip}:{endpoint}` | Varía | Contador de requests | Auto-expira por TTL |
| `session:{token}` | 30 días | Sesión de usuario | Al cerrar sesión |

---

## 7. Pipeline de Contenido (MDX)

### 7.1 Flujo de Procesamiento

El pipeline MDX transforma archivos `.mdx` en componentes React renderizables. Es el corazón del sistema de contenido.

```
                   ┌─────────────────┐
                   │  archivo.mdx    │
                   │  (Markdown +    │
                   │   JSX + YAML)   │
                   └────────┬────────┘
                            │
                   ┌────────▼────────┐
                   │   Frontmatter   │   → Extrae metadata (título, categoría,
                   │   Parser        │     fecha, tags, componentes usados)
                   └────────┬────────┘
                            │
              ┌─────────────▼─────────────┐
              │     Remark Plugins         │
              │  (transforman el AST MD)   │
              │                            │
              │  • remark-gfm (tablas,     │
              │    strikethrough)           │
              │  • remark-math (ecuaciones │
              │    $...$ y $$...$$)         │
              │  • remark-toc (generar TOC) │
              │  • remark-reading-time     │
              └─────────────┬─────────────┘
                            │
              ┌─────────────▼─────────────┐
              │     Rehype Plugins         │
              │  (transforman el HTML AST) │
              │                            │
              │  • rehype-katex (renderizar │
              │    ecuaciones a HTML)       │
              │  • rehype-pretty-code      │
              │    (Shiki syntax highlight) │
              │  • rehype-slug (IDs en     │
              │    headings para TOC)       │
              │  • rehype-autolink-headings │
              └─────────────┬─────────────┘
                            │
              ┌─────────────▼─────────────┐
              │    MDX Compiler            │
              │  (mdx → React component)   │
              │                            │
              │  Resuelve imports de        │
              │  componentes interactivos   │
              │  via components-map.ts      │
              └─────────────┬─────────────┘
                            │
              ┌─────────────▼─────────────┐
              │   Componente React Final   │
              │                            │
              │  HTML estático (server) +   │
              │  Islands de interactividad  │
              │  (client, dynamic imports)  │
              └────────────────────────────┘
```

### 7.2 Estructura de un Archivo MDX

```mdx
---
title: "El Péndulo Simple: De Newton a la Simulación"
slug: "pendulo-simple"
category: "fisica"
tags: ["mecánica-clásica", "simulación", "oscilaciones"]
author: "mazr"
publishedAt: "2026-04-15"
excerpt: "Explora la física del péndulo simple con una simulación interactiva..."
coverImage: "/images/posts/pendulo-simple.jpg"
components: ["PendulumSim", "Equation"]
---

# El Péndulo Simple

La ecuación de movimiento de un péndulo simple es:

<Equation>
  \ddot{\theta} + \frac{g}{L} \sin(\theta) = 0
</Equation>

Ajusta los parámetros y observa cómo cambia el movimiento:

<PendulumSim
  defaultGravity={9.8}
  defaultLength={1.5}
  defaultAngle={45}
  showTrail={true}
/>

## Aproximación para ángulos pequeños

Cuando $\theta$ es pequeño, $\sin(\theta) \approx \theta$, lo que simplifica
la ecuación a un oscilador armónico simple...
```

### 7.3 Registry de Componentes Interactivos

El archivo `lib/mdx/components-map.ts` mapea nombres de componentes MDX a imports dinámicos:

```typescript
// lib/mdx/components-map.ts

import dynamic from 'next/dynamic'

export const interactiveComponents = {
  // Física
  PendulumSim: dynamic(() => import('@/components/interactive/physics/pendulum-sim')),
  WaveInterference: dynamic(() => import('@/components/interactive/physics/wave-interference')),
  EMFieldViz: dynamic(() => import('@/components/interactive/physics/em-field-viz')),
  ParticleSim: dynamic(() => import('@/components/interactive/physics/particle-sim')),

  // IA
  NeuralNetViz: dynamic(() => import('@/components/interactive/ai/neural-net-viz')),
  TransformerAttention: dynamic(() => import('@/components/interactive/ai/transformer-attention')),
  GradientDescent: dynamic(() => import('@/components/interactive/ai/gradient-descent')),

  // Cuántica
  QuantumCircuit: dynamic(() => import('@/components/interactive/quantum/quantum-circuit')),
  BlochSphere: dynamic(() => import('@/components/interactive/quantum/bloch-sphere')),

  // Neurociencia
  NeuronSim: dynamic(() => import('@/components/interactive/neuro/neuron-sim')),
  BrainRegions: dynamic(() => import('@/components/interactive/neuro/brain-regions')),
}

// Componentes MDX estáticos (no necesitan dynamic import)
export const staticComponents = {
  Equation: require('@/components/mdx/equation').default,
  CodeBlock: require('@/components/mdx/code-block').default,
  Callout: require('@/components/mdx/callout').default,
  Figure: require('@/components/mdx/figure').default,
}

// Merge para pasar a MDXRemote
export const mdxComponents = {
  ...staticComponents,
  ...interactiveComponents,
}
```

---

## 8. Sistema Interactivo

### 8.1 Arquitectura de Componentes Interactivos

Cada simulación o visualización sigue una arquitectura en capas:

```
┌─────────────────────────────────────────────────┐
│            InteractiveWrapper                    │
│  ┌───────────────────────────────────────────┐  │
│  │  Header: título + descripción + fullscreen │  │
│  ├───────────────────────────────────────────┤  │
│  │                                           │  │
│  │            Canvas / SVG / WebGL           │  │
│  │         (área de visualización)           │  │
│  │                                           │  │
│  ├───────────────────────────────────────────┤  │
│  │  ControlsPanel: sliders, inputs, toggles  │  │
│  └───────────────────────────────────────────┘  │
│  Footer: créditos + tooltip de ayuda            │
└─────────────────────────────────────────────────┘
```

### 8.2 Componentes Interactivos por Categoría

| Componente | Categoría | Tecnología | Descripción |
|------------|-----------|-----------|-------------|
| `<PendulumSim />` | Física | P5.js | Péndulo simple/doble con controles de gravedad, masa, longitud, fricción. Muestra trayectoria y energía. |
| `<WaveInterference />` | Física | P5.js / Canvas | Interferencia de ondas 2D con fuentes ajustables. Patrones constructivos y destructivos. |
| `<EMFieldViz />` | Física | Three.js / R3F | Visualización 3D de campos electromagnéticos. Líneas de campo ajustables. |
| `<ParticleSim />` | Física | Canvas / WebGL | Simulación N-body de partículas con diferentes fuerzas. |
| `<NeuralNetViz />` | IA | D3.js + Canvas | Red neuronal interactiva: ajustar pesos, ver forward/backward pass animado. |
| `<TransformerAttention />` | IA | D3.js | Mapa de atención de transformers. Input editable, colores por peso de atención. |
| `<GradientDescent />` | IA | Three.js / R3F | Descenso de gradiente animado sobre superficie 3D. Ajustar learning rate y ver convergencia. |
| `<QuantumCircuit />` | Cuántica | Custom SVG/Canvas | Editor drag-and-drop de circuitos cuánticos. Compuertas H, X, CNOT, etc. |
| `<BlochSphere />` | Cuántica | Three.js / R3F | Esfera de Bloch interactiva. Rotaciones, compuertas, visualización de estado. |
| `<NeuronSim />` | Neuro | P5.js / Canvas | Modelo Hodgkin-Huxley de potencial de acción. Ajustar corriente inyectada. |
| `<BrainRegions />` | Neuro | Three.js / R3F | Modelo 3D del cerebro con regiones clickeables e información contextual. |

### 8.3 Estrategia de Carga

Ninguna librería de visualización se incluye en el bundle inicial. Se cargan solo cuando el artículo las necesita:

```typescript
// Patrón de carga para todos los componentes interactivos

import dynamic from 'next/dynamic'

const PendulumSim = dynamic(
  () => import('@/components/interactive/physics/pendulum-sim'),
  {
    ssr: false,         // No renderizar en servidor (necesita canvas/WebGL)
    loading: () => (    // Placeholder mientras carga
      <InteractiveSkeleton
        height={400}
        label="Cargando simulación..."
      />
    ),
  }
)
```

**Impacto en bundle:**

| Librería | Tamaño (gzip) | Cuándo se carga |
|----------|---------------|-----------------|
| Three.js / R3F | ~150 KB | Solo artículos con visualizaciones 3D |
| P5.js | ~80 KB | Solo artículos con simulaciones 2D |
| D3.js | ~70 KB | Solo artículos con gráficas interactivas |
| KaTeX CSS | ~25 KB | Solo artículos con ecuaciones |
| **Bundle inicial** | **<150 KB** | **Siempre** |

---

## 9. Sistema de Internacionalización (i18n)

### 9.1 Estrategia

MAZR soporta español e inglés de forma nativa, con arquitectura preparada para agregar más idiomas (alemán, francés, portugués, italiano, japonés).

| Aspecto | Implementación |
|---------|---------------|
| Librería | `next-intl` 3.x (diseñada para App Router + Server Components) |
| Detección de idioma | Middleware analiza `Accept-Language` header → redirige a `/es/` o `/en/` |
| Persistencia | Cookie `NEXT_LOCALE` mantiene la preferencia del usuario |
| Rutas | Prefijo de locale: `/es/blog/pendulo-simple`, `/en/blog/simple-pendulum` |
| Contenido de artículos | Archivos MDX separados: `content/es/` y `content/en/` |
| UI strings | Archivos JSON: `lib/i18n/dictionaries/es.json` y `en.json` |
| SEO | `hreflang` tags, sitemaps separados por idioma, `alternate` links |

### 9.2 Flujo de Resolución de Idioma

```
Request entrante
    │
    ▼
Middleware (src/middleware.ts)
    │
    ├── ¿Tiene cookie NEXT_LOCALE?
    │   └── Sí → Usar ese idioma
    │
    ├── ¿Accept-Language header?
    │   └── Sí → Detectar idioma preferido
    │
    └── Fallback → español (es)
    │
    ▼
Redirect a /[locale]/... si no tiene prefijo
    │
    ▼
Layout [locale] carga NextIntlClientProvider
con el diccionario del idioma correspondiente
```

### 9.3 Agregar un Nuevo Idioma

Pasos para agregar un idioma (ejemplo: alemán):

1. Crear `lib/i18n/dictionaries/de.json` con todas las traducciones de UI.
2. Agregar `'de'` al array de locales en `lib/i18n/config.ts`.
3. Crear la carpeta `content/de/` con los archivos MDX traducidos.
4. Agregar campos `title_de`, `excerpt_de`, etc. al modelo de datos (migración Prisma).
5. Actualizar middleware para reconocer el nuevo locale.
6. Actualizar sitemap y `hreflang` tags.

---

## 10. Sistema de Autenticación y Autorización

### 10.1 Stack de Autenticación

| Componente | Tecnología | Función |
|-----------|-----------|---------|
| Framework | NextAuth.js v5 (Auth.js) | Manejo de sesiones, providers, callbacks |
| Storage | Prisma Adapter → PostgreSQL | Sesiones y cuentas almacenadas en la base de datos |
| Provider inicial | Credentials (email + password) | Login para el admin principal |
| Providers futuros | GitHub, Google OAuth | Para colaboradores |
| Sesiones | Database sessions (no JWT) | Revocables, auditables, almacenadas en PostgreSQL |

### 10.2 Modelo de Roles y Permisos

| Rol | Crear post | Editar propio | Editar otros | Eliminar post | Moderar comentarios | Gestionar usuarios | Config sitio |
|-----|-----------|---------------|-------------|--------------|--------------------|--------------------|-------------|
| **ADMIN** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **EDITOR** | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **AUTHOR** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

### 10.3 Protección de Rutas

```
Capa 1: Middleware (src/middleware.ts)
  → Intercepta todas las requests a /admin/*
  → Si no hay sesión → redirect a /login
  → Si hay sesión → continúa

Capa 2: Layout Admin (src/app/admin/layout.tsx)
  → getServerSession() verifica sesión en server
  → Si rol no es ADMIN/EDITOR/AUTHOR → redirect a /

Capa 3: Route Handlers individuales
  → Verifican rol específico para cada operación
  → AUTHOR puede crear pero no eliminar
```

---

## 11. Sistema de Búsqueda

### 11.1 Arquitectura de Búsqueda Full-Text

MAZR usa PostgreSQL native full-text search en lugar de Elasticsearch o Algolia. Para <1000 posts, PostgreSQL es más que suficiente y evita dependencias externas.

```
                    ┌──────────────┐
                    │  SearchBar   │ (Client Component)
                    │  con debounce│ (300ms)
                    └──────┬───────┘
                           │
                  GET /api/search?q=X&lang=Y
                           │
                    ┌──────▼───────┐
                    │ Route Handler │
                    │ rate limited  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐     ┌─────────┐
                    │  Redis Cache │◄───▶│  2 min  │
                    │  (hit/miss)  │     │  TTL    │
                    └──────┬───────┘     └─────────┘
                           │ (miss)
                    ┌──────▼───────────────────────┐
                    │       PostgreSQL              │
                    │                               │
                    │  1. Full-text search          │
                    │     to_tsvector + ts_rank     │
                    │     (idioma correcto)         │
                    │                               │
                    │  2. Fuzzy matching             │
                    │     pg_trgm + similarity()    │
                    │     (tolerancia a typos)      │
                    │                               │
                    │  3. Unaccent                   │
                    │     (ignorar acentos)         │
                    └──────────────────────────────┘
```

### 11.2 Query de Búsqueda (SQL)

```sql
-- Búsqueda en español con ranking y fuzzy fallback
SELECT
  p.id, p.slug, p.title_es AS title, p.excerpt_es AS excerpt,
  ts_rank(
    to_tsvector('spanish', unaccent(p.title_es || ' ' || COALESCE(p.excerpt_es, ''))),
    plainto_tsquery('spanish', unaccent($1))
  ) AS rank,
  similarity(p.title_es, $1) AS fuzzy_score
FROM posts p
WHERE
  p.status = 'PUBLISHED'
  AND (
    to_tsvector('spanish', unaccent(p.title_es || ' ' || COALESCE(p.excerpt_es, '')))
    @@ plainto_tsquery('spanish', unaccent($1))
    OR similarity(p.title_es, $1) > 0.3
  )
ORDER BY rank DESC, fuzzy_score DESC
LIMIT 20;
```

---

## 12. Sistema de Comentarios

### 12.1 Arquitectura

| Aspecto | Decisión |
|---------|----------|
| Almacenamiento | PostgreSQL (tabla `comments` con self-reference para hilos) |
| Autenticación | No requerida (nombre + email), con moderación obligatoria |
| Moderación | Cola de aprobación en admin panel. Estados: PENDING → APPROVED/REJECTED/SPAM |
| Anti-spam | Honeypot field + rate limiting (5 comentarios/min por IP) |
| Notificaciones | Email al autor cuando recibe respuesta (Resend / Nodemailer) |
| Formato | Markdown básico (bold, italic, code, links) — sanitizado con rehype-sanitize |
| Hilos | Un nivel de profundidad (respuestas directas, no sub-hilos infinitos) |

### 12.2 Flujo de un Comentario

```
Usuario escribe comentario
        │
        ▼
  Client-side validation
  (Zod: nombre, email, contenido)
        │
        ▼
  POST /api/comments
        │
        ├── Rate limit check (Redis) → 429 si excede
        ├── Honeypot check → silently discard si falla
        ├── Sanitize Markdown → rehype-sanitize
        │
        ▼
  Guardar en PostgreSQL (status: PENDING)
        │
        ├── Respuesta al usuario: "Comentario enviado, pendiente de aprobación"
        │
        └── (Async) Notificar al admin (email o dashboard badge)
```

---

## 13. Sistema de Diseño

### 13.1 Filosofía

Minimalismo funcional: cada elemento tiene un propósito. Espacios amplios, tipografía legible, y el contenido como protagonista. El verde aporta identidad sin saturar.

### 13.2 Tokens de Diseño

Los tokens se definen como CSS Custom Properties en `src/styles/tokens.css` y se mapean a clases de Tailwind en `tailwind.config.ts`.

**Colores:**

| Token | Light Mode | Dark Mode | Uso |
|-------|-----------|-----------|-----|
| `--primary` | `#1B5E20` | `#4CAF50` | Acentos, links, CTAs |
| `--primary-light` | `#E8F5E9` | `#1B3A1E` | Backgrounds de highlight |
| `--bg` | `#FFFFFF` | `#0F172A` | Fondo principal |
| `--bg-alt` | `#F8FAFC` | `#1E293B` | Fondo secundario |
| `--text` | `#111827` | `#F1F5F9` | Texto principal |
| `--text-muted` | `#6B7280` | `#94A3B8` | Texto secundario |
| `--border` | `#E5E7EB` | `#334155` | Bordes y separadores |
| `--cat-fisica` | `#10B981` | `#34D399` | Categoría Física |
| `--cat-ia` | `#3B82F6` | `#60A5FA` | Categoría IA |
| `--cat-cuantica` | `#8B5CF6` | `#A78BFA` | Categoría Cuántica |
| `--cat-neuro` | `#F43F5E` | `#FB7185` | Categoría Neurociencia |

**Tipografía:**

| Elemento | Font | Peso | Tamaño | Uso |
|----------|------|------|--------|-----|
| Display / Logo | Space Grotesk | 700-800 | 48-72px | Logo MAZR, hero titles |
| Headings | Space Grotesk | 600-700 | 24-36px | H1-H4 de artículos y páginas |
| Body text | Source Serif 4 | 400 | 18px / 1.75 lh | Texto de artículos |
| Code | JetBrains Mono | 400 | 15px | Bloques de código, inline code |
| UI / Labels | Inter | 400-500 | 14-16px | Botones, badges, nav, metadata |

**Spacing:** Sistema de 4px base (4, 8, 12, 16, 24, 32, 48, 64, 96, 128).

### 13.3 Dark Mode

Implementación con Tailwind `class` strategy, no `media`. El usuario controla el modo manualmente:

1. `ThemeProvider` lee preferencia de `localStorage` (o `prefers-color-scheme` como fallback).
2. Aplica clase `dark` al `<html>`.
3. Los tokens CSS cambian via `html.dark { --bg: #0F172A; ... }`.
4. Tailwind aplica variantes `dark:` automáticamente.

---

## 14. Infraestructura y Deployment

### 14.1 Entorno de Desarrollo

```yaml
# docker-compose.yml (desarrollo)
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mazr_dev
      POSTGRES_USER: mazr
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin

volumes:
  pgdata:
```

El desarrollador ejecuta Next.js localmente (`pnpm dev`) y los servicios de infraestructura en Docker.

### 14.2 Entorno de Producción

```yaml
# docker-compose.prod.yml
services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    restart: always
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://mazr:${DB_PASSWORD}@postgres:5432/mazr_prod
      - REDIS_URL=redis://redis:6379
    depends_on: [postgres, redis]

  nginx:
    image: nginx:1.25-alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./docker/nginx/sites/mazr.conf:/etc/nginx/conf.d/default.conf
    depends_on: [app]

  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_DB: mazr_prod
      POSTGRES_USER: mazr
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata_prod:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    restart: always
    command: redis-server --maxmemory 100mb --maxmemory-policy allkeys-lru

volumes:
  pgdata_prod:
```

### 14.3 Pipeline CI/CD (GitHub Actions)

```
Push a main / PR abierto
        │
        ▼
  ┌─────────────────────────────┐
  │  CI Pipeline (ci.yml)       │
  │                             │
  │  1. pnpm install            │
  │  2. TypeScript type-check   │
  │  3. ESLint                  │
  │  4. Prettier check          │
  │  5. Unit tests (Vitest)     │
  │  6. Build Next.js           │
  │  7. E2E tests (Playwright)  │
  └─────────────┬───────────────┘
                │ (solo si pasa + merge a main)
                ▼
  ┌─────────────────────────────┐
  │  CD Pipeline (cd.yml)       │
  │                             │
  │  1. Build Docker image      │
  │  2. Push a registry         │
  │  3. SSH al VPS              │
  │  4. Pull nueva imagen       │
  │  5. docker compose up -d    │
  │  6. Prisma migrate deploy   │
  │  7. Health check            │
  │  8. Notificar resultado     │
  └─────────────────────────────┘
```

### 14.4 Dockerfile (Multi-Stage Build)

```dockerfile
# docker/Dockerfile

# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable pnpm && pnpm install --frozen-lockfile

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN corepack enable pnpm && pnpm build

# Stage 3: Production
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 15. Observabilidad y Monitoreo

| Herramienta | Propósito | Costo |
|------------|-----------|-------|
| **Sentry** | Error tracking con stack traces, contexto, y breadcrumbs. SDK oficial de Next.js. | Gratis (5K eventos/mes) |
| **Cloudflare Analytics** | Métricas de tráfico sin JS adicional en el cliente. Privacy-friendly. | Gratis |
| **Uptime Kuma** (self-hosted) | Monitoreo de uptime. Ping al endpoint `/api/health` cada 60 segundos. Alertas via email/Telegram. | Gratis |
| **Next.js Analytics** (built-in) | Core Web Vitals reportados desde navegadores reales. | Gratis |
| **PostgreSQL logs** | Queries lentas (>500ms) loggeadas para optimización. | Incluido |

### Health Check Endpoint

```typescript
// src/app/api/health/route.ts
export async function GET() {
  const checks = {
    database: await checkPostgres(),
    cache: await checkRedis(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    timestamp: new Date().toISOString(),
  }

  const healthy = checks.database && checks.cache
  return Response.json(checks, { status: healthy ? 200 : 503 })
}
```

---

## 16. Seguridad

### 16.1 Medidas por Capa

| Capa | Medida | Implementación |
|------|--------|---------------|
| **Red** | DDoS protection | Cloudflare (automático) |
| **Red** | SSL/TLS | Cloudflare edge SSL + Nginx |
| **Red** | Rate limiting | Redis-based por IP y endpoint |
| **Aplicación** | Input validation | Zod schemas en todos los Route Handlers |
| **Aplicación** | CSRF protection | NextAuth.js CSRF tokens |
| **Aplicación** | XSS prevention | React escaping por defecto + rehype-sanitize para comentarios |
| **Aplicación** | SQL injection | Prisma ORM (queries parametrizadas) |
| **Aplicación** | Auth | NextAuth.js con database sessions (revocables) |
| **Datos** | Passwords | bcrypt hashing (via NextAuth credentials provider) |
| **Datos** | Secrets | Variables de entorno (`.env.local`, nunca en código) |
| **Datos** | Backups | pg_dump diario automatizado con cron |
| **Infra** | SSH hardening | Key-only auth, fail2ban, no root login |
| **Infra** | Docker** | Non-root containers, read-only filesystem donde posible |
| **Infra** | Dependencies | Dependabot + `pnpm audit` semanal |

### 16.2 Headers de Seguridad (Nginx)

```nginx
# Configurados en nginx.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```

---

## 17. Rendimiento y Optimización

### 17.1 Objetivos

| Métrica | Objetivo | Estrategia |
|---------|---------|------------|
| Lighthouse Performance | >95 | SSG, code splitting, image optimization |
| First Contentful Paint | <1.0s | Server-side rendering, CDN cache |
| Largest Contentful Paint | <2.0s | Priority loading de hero image, font preloading |
| Cumulative Layout Shift | <0.05 | Dimensiones explícitas en imágenes, font-display: swap |
| Total Blocking Time | <200ms | Minimizar JS en main thread, dynamic imports |
| Bundle Size (JS inicial) | <150KB | Tree shaking, dynamic imports de librerías pesadas |

### 17.2 Estrategias de Optimización

**Imágenes:**
Next.js `Image` component con automatic optimization, lazy loading, y formatos modernos (WebP/AVIF). Dimensiones explícitas para evitar CLS.

**Fonts:**
Self-hosted en `public/fonts/`, preloaded con `next/font`. `font-display: swap` para evitar FOIT (Flash of Invisible Text).

**JavaScript:**
Server Components por defecto (0 JS). Client Components solo donde hay interactividad. Dynamic imports con `next/dynamic` para simulaciones (Three.js, P5.js, D3.js). Tree shaking de Framer Motion (~30KB solo lo importado).

**CSS:**
Tailwind CSS purge automático en producción (<20KB). CSS Custom Properties para tokens (sin JS para dark mode).

**Caché:**
Cloudflare CDN para assets estáticos (max-age: 1 año con hashing). Redis para API responses frecuentes. ISR para páginas semi-estáticas. HTTP cache headers optimizados.

**Base de datos:**
Índices en columnas frecuentemente consultadas. Connection pooling via Prisma. Queries optimizadas con `select` explícito (no `select *`).

---

## 18. Decisiones Arquitectónicas (ADRs)

Cada decisión significativa se documenta como un Architecture Decision Record.

### ADR-001: Next.js App Router como Monolito Fullstack

**Contexto:** Necesitamos un framework que soporte SSR, SSG, CSR, API routes, e i18n.

**Decisión:** Usar Next.js 14+ App Router como monolito fullstack (frontend + backend + admin en un solo proyecto).

**Razón:** Para un solo developer, mantener repos separados para frontend y backend es overhead. Next.js unifica todo con zero-config. Server Components reducen JS en cliente. Route Handlers eliminan la necesidad de Express/Fastify.

**Consecuencias:** Todo vive en un deploy. Si se necesita escalar backend independientemente del frontend en el futuro, se puede extraer `src/app/api/` a un servicio separado.

### ADR-002: MDX como Formato de Contenido

**Contexto:** Los artículos necesitan contener componentes React interactivos (simulaciones, visualizaciones).

**Decisión:** Usar MDX como formato de contenido, almacenado en `content/` y versionado en Git.

**Razón:** MDX permite escribir `<PendulumSim />` directamente en el artículo. Ningún CMS headless soporta esto nativamente. Git da historial de cambios y revisión.

**Consecuencias:** El admin panel necesita un editor MDX custom (no se puede usar un CMS genérico). El pipeline de compilación tiene complejidad adicional (remark/rehype plugins).

### ADR-003: VPS Self-Hosted sobre Serverless

**Contexto:** Necesitamos hosting para la app + PostgreSQL + Redis con presupuesto <$25/mes.

**Decisión:** VPS (Hetzner o DigitalOcean) con Docker Compose.

**Razón:** Vercel requiere DB y Redis externos ($20-40/mes adicional). Un VPS de $10-20/mes aloja todo. Control total del entorno, sin vendor lock-in.

**Consecuencias:** Responsabilidad de mantenimiento del servidor (updates, backups, monitoreo). Se mitiga con Docker (entorno reproducible), GitHub Actions (deploy automático), y Uptime Kuma (alertas).

### ADR-004: PostgreSQL Full-Text Search sobre Elasticsearch

**Contexto:** Necesitamos búsqueda full-text bilingüe (español + inglés) con autocompletado.

**Decisión:** Usar PostgreSQL native full-text search con `tsvector`, `pg_trgm`, y `unaccent`.

**Razón:** Para <1000 posts, PostgreSQL es suficiente. Evita una dependencia adicional (Elasticsearch necesita 2GB+ RAM). Soporta configuraciones de idioma nativas (spanish, english). `pg_trgm` da fuzzy matching gratis.

**Consecuencias:** Si el volumen crece significativamente (+10K posts), se puede migrar a Elasticsearch/Meilisearch. El código de búsqueda está aislado en `lib/search/full-text.ts`.

### ADR-005: Custom Design System sobre Component Libraries

**Contexto:** MAZR tiene una identidad visual específica (minimalismo, verde primario, colores por categoría).

**Decisión:** Construir componentes UI custom con Tailwind CSS en lugar de usar shadcn/ui, Tailwind UI, o similar.

**Razón:** Una component library prearmada diluiría la identidad visual. MAZR necesita componentes especializados (interactive wrappers, category badges con colores dinámicos, post reader optimizado). Tailwind utility classes + tokens CSS custom dan el control necesario.

**Consecuencias:** Más tiempo de desarrollo inicial para componentes base. Se mitiga con scope limitado: solo se construyen los componentes que MAZR realmente necesita.

---

## 19. Flujos de Datos Principales

### 19.1 Lectura de un Post (Usuario Público)

```
Usuario visita /es/blog/pendulo-simple
        │
        ▼
  Cloudflare CDN
  ├── Cache HIT → Devolver página estática (instantáneo)
  └── Cache MISS ↓
        │
        ▼
  Nginx → Next.js
        │
        ▼
  Next.js App Router
  ├── Resuelve [locale] = "es"
  ├── Resuelve [slug] = "pendulo-simple"
  │
  ▼
  Server Component: PostPage
  ├── Fetch post de DB (o caché ISR)
  ├── Compilar MDX → React components
  ├── Renderizar Server Components a HTML
  ├── Inyectar metadata (OG, hreflang, JSON-LD)
  │
  ▼
  Enviar HTML al navegador
  ├── HTML completo renderizado (SEO ✅)
  ├── CSS inline (Tailwind, KaTeX)
  │
  ▼
  Hidratación en el cliente
  ├── React hidrata Server Components (0 JS adicional)
  ├── Client Components se activan (TOC, comentarios)
  ├── <PendulumSim /> se carga via dynamic import
  │   └── P5.js (~80KB) se descarga bajo demanda
  │
  ▼
  Artículo completamente interactivo
```

### 19.2 Publicación de un Post (Admin)

```
Admin escribe post en /admin/posts/new
        │
        ▼
  BilinguaEditor (side-by-side es/en)
  ├── MDX editor con toolbar
  ├── Preview en tiempo real
  ├── Insertar componente interactivo via toolbar
  │
  ▼
  Click "Publicar"
        │
        ▼
  POST /api/posts
  ├── Validar sesión + rol (ADMIN/EDITOR)
  ├── Validar payload con Zod
  ├── Guardar en PostgreSQL
  ├── Subir imágenes a MinIO/S3
  ├── Invalidar caché Redis (listados, categorías)
  │
  ▼
  Trigger ISR revalidation
  ├── Next.js regenera páginas afectadas
  ├── Cloudflare purge del path específico
  │
  ▼
  Post visible en el blog público
```

---

## 20. Estrategia de Escalabilidad

### 20.1 Estado Actual: Monolito Optimizado

MAZR v1.0 es un monolito. Toda la aplicación corre en un solo proceso Node.js, con PostgreSQL y Redis en el mismo VPS. Esto es apropiado para el tráfico esperado (500-5000 visitantes/mes).

### 20.2 Señales para Escalar

| Señal | Umbral | Acción |
|-------|--------|--------|
| CPU constante >80% | Sostenido por días | Escalar VPS verticalmente (más CPU/RAM) |
| RAM >90% | Sostenido | Mover PostgreSQL a managed DB (Supabase, Neon) |
| Response time >2s | P95 sostenido | Agregar segundo container de Next.js + Nginx load balancer |
| >10K posts | Búsqueda lenta | Evaluar Meilisearch o Elasticsearch |
| >3 colaboradores | Conflictos frecuentes | Evaluar monorepo con workspaces (Turborepo) |
| >50K visitantes/mes | CDN cache miss alto | Escalar a múltiples VPS con load balancer externo |

### 20.3 Camino de Escalabilidad

```
Fase actual                 Fase 2                    Fase 3
(monolito, 1 VPS)          (servicios separados)     (distribuido)

┌─────────────┐        ┌─────────────┐           ┌──────────────┐
│  Next.js    │        │  Next.js    │           │  Next.js x3  │
│  + API      │   →    │  (frontend) │      →    │  (frontend)  │
│  + Admin    │        ├─────────────┤           ├──────────────┤
│             │        │  API        │           │  API x2      │
│  PostgreSQL │        │  (Express)  │           │  (Express)   │
│  Redis      │        ├─────────────┤           ├──────────────┤
│             │        │  Managed DB │           │  Managed DB  │
│  1 VPS      │        │  + Redis    │           │  + Redis     │
└─────────────┘        └─────────────┘           │  + CDN       │
                                                  └──────────────┘
  $10-20/mes             $30-60/mes               $100+/mes
  <5K users/mes          5-50K users/mes          50K+ users/mes
```

La separación lógica interna (components, lib, api) permite esta evolución sin reescribir. Cada fase se activa solo cuando el dolor lo justifica.

---

*MAZR — Donde la ciencia cobra vida.*
