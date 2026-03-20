# ADR-003 — VPS Self-Hosted sobre Serverless

**Fecha:** 20 de Marzo del 2026  
**Estado:** Aceptado

---

## Contexto  

Necesitamos hosting para la app + PostgreSQL + Redis con presupuesto <$25/mes.

---

## Decisión

VPS (Hetzner o DigitalOcean) con Docker Compose.

---

## Razón:

Vercel requiere DB y Redis externos ($20-40/mes adicional). Un VPS de $10-20/mes aloja todo. Control total del entorno, sin vendor lock-in.

---

## Consecuencias

Responsabilidad de mantenimiento del servidor (updates, backups, monitoreo). Se mitiga con Docker (entorno reproducible), GitHub Actions (deploy automático), y Uptime Kuma (alertas).