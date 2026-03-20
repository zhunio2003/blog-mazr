# ADR-002 — MDX como Formato de Contenido

**Fecha:** 20 de Marzo del 2026  
**Estado:** Aceptado

---

## Contexto  

Los artículos necesitan contener componentes React interactivos (simulaciones, visualizaciones).

---

## Decisión

Usar MDX como formato de contenido, almacenado en content/ y versionado en Git.

---

## Razón:

MDX permite escribir <PendulumSim /> directamente en el artículo. Ningún CMS headless soporta esto nativamente. Git da historial de cambios y revisión.

---

## Consecuencias

El admin panel necesita un editor MDX custom (no se puede usar un CMS genérico). El pipeline de compilación tiene complejidad adicional (remark/rehype plugins).