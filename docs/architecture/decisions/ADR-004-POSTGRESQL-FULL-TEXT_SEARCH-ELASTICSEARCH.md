# ADR-004 — PostgreSQL Full-Text Search sobre Elasticsearch

**Fecha:** 20 de Marzo del 2026  
**Estado:** Aceptado

---

## Contexto  

Necesitamos búsqueda full-text bilingüe (español + inglés) con autocompletado.

---

## Decisión

Usar PostgreSQL native full-text search con tsvector, pg_trgm, y unaccent.
---

## Razón:

Para <1000 posts, PostgreSQL es suficiente. Evita una dependencia adicional (Elasticsearch necesita 2GB+ RAM). Soporta configuraciones de idioma nativas (spanish, english). pg_trgm da fuzzy matching gratis.

---

## Consecuencias

Si el volumen crece significativamente (+10K posts), se puede migrar a Elasticsearch/Meilisearch. El código de búsqueda está aislado en lib/search/full-text.ts.