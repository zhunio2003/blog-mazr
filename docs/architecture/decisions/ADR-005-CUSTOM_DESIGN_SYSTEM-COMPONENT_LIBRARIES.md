# ADR-005 — Custom Design System sobre Component Libraries

**Fecha:** 20 de Marzo del 2026  
**Estado:** Aceptado

---

## Contexto  

MAZR tiene una identidad visual específica (minimalismo, verde primario, colores por categoría).

---

## Decisión

Construir componentes UI custom con Tailwind CSS en lugar de usar shadcn/ui, Tailwind UI, o similar.

---

## Razón:

Una component library prearmada diluiría la identidad visual. MAZR necesita componentes especializados (interactive wrappers, category badges con colores dinámicos, post reader optimizado). Tailwind utility classes + tokens CSS custom dan el control necesario.

---

## Consecuencias

Más tiempo de desarrollo inicial para componentes base. Se mitiga con scope limitado: solo se construyen los componentes que MAZR realmente necesita.