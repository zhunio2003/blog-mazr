# ADR-001 — Next.js App Router como Monolito Fullstack

**Fecha:** 20 de Marzo del 2026  
**Estado:** Aceptado

---

## Contexto  

Necesitamos un framework que soporte SSR, SSG, CSR, API routes, e i18n.

---

## Decisión

Usar Next.js 14+ App Router como monolito fullstack (frontend + backend + admin en un solo proyecto).

---

## Razón:

Para un solo developer, mantener repos separados para frontend y backend es overhead. Next.js unifica todo con zero-config. Server Components reducen JS en cliente. Route Handlers eliminan la necesidad de Express/Fastify.

---

## Consecuencias

Todo vive en un deploy. Si se necesita escalar backend independientemente del frontend en el futuro, se puede extraer src/app/api/ a un servicio separado