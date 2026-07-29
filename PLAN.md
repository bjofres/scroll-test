# PLAN: Desert Day-to-Night Scroll Animation

## Fases

### Fase 1: Setup del proyecto (15min)
- Crear index.html con canvas fullscreen, imports CDN
- Inicializar Lenis, GSAP, ScrollTrigger
- Configurar canvas resize handler

### Fase 2: Paleta de colores y cielo (20min)
- Implementar palette.js con interpolación RGB entre paletas clave
- Implementar sky.js con gradiente vertical interpolado

### Fase 3: Sol y Luna (20min)
- Implementar sun.js: posición, color, glow
- Implementar moon.js: fade-in, glow plateado

### Fase 4: Estrellas (15min)
- Implementar stars.js: distribución procedural, parpadeo sinusoidal

### Fase 5: Montañas y Dunas (30min)
- Implementar mountains.js: 3 capas de siluetas poligonales con parallax
- Implementar dunes.js: 3 capas con curvas bezier y parallax

### Fase 6: Cactus y detalles (15min)
- Implementar cactus.js: siluetas de saguaro

### Fase 7: Integración y ajustes (20min)
- Conectar SceneManager con ScrollTrigger
- Ajustar curvas de interpolación
- Probar en viewports

## Hitos de Revisión

| Hito | Punto de control |
|------|------------------|
| H1 | Cielo transiciona suavemente entre paletas |
| H2 | Sol se pone y luna aparece fluidamente |
| H3 | Estrellas brillan con parpadeo natural |
| H4 | Montañas y dunas tienen parallax correcto |
| H5 | Escena completa funciona a 60fps |
