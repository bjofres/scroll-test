# Architecture: Desert Day-to-Night Scroll Animation

## Tech Stack

| Capa | Tecnología | Justificación |
|------|-----------|---------------|
| Renderizado | Canvas 2D | Renderizado eficiente de escena completa sin DOM overhead |
| Animación scroll | GSAP + ScrollTrigger | Control preciso de progreso con scrub, pinning y easing |
| Smooth scroll | Lenis | Scroll suave de 60fps con inercia natural |
| Estilos | CSS (vanilla) | Solo layout básico y reset; sin framework necesario |
| Carga | CDN (jsdelivr) | GSAP, ScrollTrigger, Lenis vía script tags |

## Árbol de Archivos

```
desert-scroll/
├── index.html          # Punto de entrada único
├── SPEC.md             # Especificación de animación
├── ARCHITECTURE.md     # Decisiones arquitectónicas
├── PLAN.md             # Plan de implementación
├── REVIEW.md           # Revisión de código
├── QA_REPORT.md        # Reporte de calidad
├── README.md           # Documentación final
├── src/
│   ├── main.js         # Entry point: init Lenis, GSAP, ScrollTrigger
│   ├── scene.js        # SceneManager: canvas setup, render loop
│   ├── sky.js          # Sky gradient renderer
│   ├── sun.js          # Sun renderer
│   ├── moon.js         # Moon renderer
│   ├── stars.js        # Stars renderer with twinkle
│   ├── mountains.js    # Mountain silhouettes (3 layers)
│   ├── dunes.js        # Sand dunes (3 layers, bezier curves)
│   ├── cactus.js       # Saguaro cactus silhouettes
│   └── palette.js      # Color palettes and interpolation
└── docs/
    └── animation.md    # Documentación técnica de animación
```

## Flujo de Renderizado

```
Scroll del usuario
    │
    ▼
Lenis (smooth scroll, 60fps)
    │
    ▼
ScrollTrigger (GSAP) → actualiza `progress` (0 → 1)
    │
    ▼
SceneManager.render(progress)
    │
    ├─► sky.render(ctx, progress)
    ├─► stars.render(ctx, progress)
    ├─► sun.render(ctx, progress)
    ├─► moon.render(ctx, progress)
    ├─► mountains.render(ctx, progress)
    ├─► dunes.render(ctx, progress)
    └─► cactus.render(ctx, progress)
    │
    ▼
Canvas 2D → frame en pantalla
```

## ADRs

### ADR-001: Canvas 2D sobre Three.js

**Contexto**: Necesitamos renderizar una escena 2D con transiciones de color, formas geométricas y parallax.

**Decisión**: Usar Canvas 2D nativo en lugar de Three.js.

**Consecuencias**:
- + Sin dependencia pesada (Three.js ~500KB)
- + Control total sobre cada píxel
- + Sin overhead de WebGL
- - Más código manual para efectos de glow
- - Sin 3D (que no es necesario aquí)

### ADR-002: GSAP ScrollTrigger sobre IntersectionObserver

**Contexto**: Necesitamos sincronizar el progreso de scroll (0 a 1) con la animación de forma continua (scrub).

**Decisión**: Usar GSAP ScrollTrigger con scrub: 1 en lugar de IntersectionObserver.

**Consecuencias**:
- + Scrub continuo y suave, no por thresholds discretos
- + Easing integrado en la interpolación
- + Pin option disponible si se desea sección fija
- - Dependencia externa (GSAP ~30KB gzipped con ScrollTrigger)

### ADR-003: Lenis para smooth scrolling

**Contexto**: El scroll nativo del navegador puede tener jank en ciertos dispositivos.

**Decisión**: Usar Lenis para normalizar el scroll a 60fps con inercia natural.

**Consecuencias**:
- + Scroll fluido y consistente entre navegadores
- + Integración directa con GSAP ScrollTrigger via `lenis` plugin
- - Dependencia adicional (~8KB)

## Budget de Rendimiento

| Métrica | Target |
|---------|--------|
| FPS | 60fps sostenidos |
| Tamaño bundle | < 100KB (CDN) |
| Time to Interactive | < 2s |
| Sin reflow/layout en scroll | ✅ solo Canvas paint |
