# SPEC: Desert Day-to-Night Scroll Animation

## 1. Descripción General

Página web single-section que presenta una escena de desierto en lienzo completo que transiciona de día a noche conforme el usuario hace scroll down. Sin texto, sin UI, solo la experiencia visual pura.

## 2. Estados de la Animación

| Scroll % | Momento del día | Descripción |
|-----------|----------------|-------------|
| 0% | Mediodía | Cielo azul brillante, sol alto, dunas iluminadas, sombras definidas |
| 25% | Tarde | Cielo más cálido (naranja tenue), sol descendiendo, sombras más largas |
| 50% | Atardecer | Cielo naranja/rojo, sol en el horizonte, siluetas, inicio de estrellas |
| 75% | Crepúsculo | Cielo púrpura oscuro, luna creciente, más estrellas, dunas oscuras |
| 100% | Noche | Cielo negro con estrellas, luna llena brillante, siluetas frías |

## 3. Elementos Visuales

Cada elemento se renderiza en Canvas 2D con interpolación paramétrica según el progreso `t ∈ [0, 1]`.

### 3.1 Cielo (Fondo degradado)
- **Día** `t=0.0`: `#4ab3eb` → `#87ceeb` (gradiente lineal vertical)
- **Atardecer** `t=0.5`: `#ff6b35` → `#f7c948`
- **Noche** `t=1.0`: `#0a0a2a` → `#1a1a3a`
- Transición vía interpolación RGB entre paletas clave

### 3.2 Sol
- Círculo con radio 60px
- Posición X: centro del viewport
- Posición Y: `80 - t * 120` (% del alto) → se oculta bajo el horizonte
- Color: amarillo (`#ffd700`) → naranja intenso al atardecer → se desvanece
- Glow radial con opacidad que decrece hasta `t=0.6`

### 3.3 Luna
- Círculo con radio 40px
- Aparece desde `t=0.4` con fade-in
- Posición: opuesta al sol (sube mientras el sol baja)
- Color: blanco suave `#f0f0ff` con glow plateado

### 3.4 Estrellas
- ~100 puntos distribuidos en la mitad superior
- Visibles desde `t=0.3`, opacidad máxima en `t=1.0`
- Tamaño variable (1-3px)
- Parpadeo sinusoidal individual

### 3.5 Montañas Lejanas (3 capas)
- Siluetas de montañas con perfil poligonal irregular
- Capa más lejana: más clara, menor desplazamiento parallax
- Capa media: tono medio
- Capa cercana: más oscura, mayor parallax
- Color varía con `t` (cálido de día, frío de noche)

### 3.6 Dunas (3 capas)
- Curvas suaves bezier para perfiles de duna
- Parallax: capas cercanas se mueven más que lejanas
- Sombreado con gradiente interno según posición del sol
- Color: ocre `#e8c97a` (día) → marrón oscuro `#3d2b1f` (noche)

### 3.7 Cactus (siluetas)
- 3-5 siluetas de cactus (saguaro) distribuidas
- Escala y posición fijas
- Color: silueta oscura con variación según luz
- Posicionados en capa frontal

## 4. Triggers y Timeline

| Trigger | Evento | Rango |
|---------|--------|-------|
| Scroll down | Avance de `t` de 0 a 1 | 0-100vh |
| Scroll up | Retroceso de `t` de 1 a 0 | 100-0vh |
| Resize | Re-render del canvas con nuevas dimensiones | cualquier momento |

## 5. Curvas de Interpolación

```js
// Paletas de cielo
const skyPalettes = [
  { pos: 0.0, top: '#4ab3eb', bottom: '#87ceeb' },
  { pos: 0.3, top: '#f4a460', bottom: '#ffd700' },
  { pos: 0.5, top: '#ff6b35', bottom: '#f7c948' },
  { pos: 0.7, top: '#6b2fa0', bottom: '#2d1b69' },
  { pos: 1.0, top: '#0a0a2a', bottom: '#1a1a3a' },
]
```

## 6. Accesibilidad

- `prefers-reduced-motion`: la animación se pausa en el estado inicial (día) sin scroll triggers
- No hay parpadeos rápidos que puedan causar fotosensibilidad
- La página no tiene contenido textual que requiera lectura

## 7. Rendimiento

- Canvas 2D (no DOM) para renderizado eficiente
- RequestAnimationFrame para el loop de pintado
- No re-paint de elementos DOM en scroll
- Resize del canvas con debounce (250ms)
- La complejidad de escena se mantiene constante (~100 estrellas, 3 capas de montañas, 3 dunas, 5 cactus)
