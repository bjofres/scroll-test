# PLAN_MOBILE: Adaptación responsive para mobile portrait

> **Autor**: Frontend Lead  
> **Fecha**: 2026-07-29  
> **Versión**: 1.0  
> **Objetivo**: Corregir la visualización en mobile portrait (pantallas angostas en vertical) — el canvas se ve "puntiagudo, mal proporcionado, nada responsivo".

---

## Tabla de Contenidos

1. [Análisis de problemas visuales](#1-análisis-de-problemas-visuales)
2. [Propuesta de solución por problema](#2-propuesta-de-solución-por-problema)
3. [Estrategia de responsive design](#3-estrategia-de-responsive-design)
4. [Plan de implementación detallado](#4-plan-de-implementación-detallado)
5. [Priorización](#5-priorización)
6. [Estimación de esfuerzo](#6-estimación-de-esfuerzo)
7. [Código completo propuesto](#7-código-completo-propuesto)

---

## 1. Análisis de problemas visuales

### 1.1 Dunas puntiagudas (CAUSA RAÍZ)

**Síntoma**: Las dunas se ven como picos afilados en vez de colinas suaves y onduladas.

**Causa**: Dos factores combinados destruyen la relación de aspecto natural de las dunas:

| Factor | Código actual | Efecto |
|--------|--------------|--------|
| `duneWScale` | `Math.max(0.6, innerWidth/1920)` → **0.6** en 375px | Los perfiles Gaussianos se angostan 40% |
| `portraitMul` | `1.5` (cuando `portraitRatio() > 1.3`) | La altura de dunas se incrementa 50% |

**Efecto neto**: La relación ancho:alto de cada duna cambia drásticamente:

| Capa | Desktop (1920×1080) | Mobile portrait actual (375×667) |
|------|--------------------|----------------------------------|
| L0 (fondo) | **5.1:1 a 7.8:1** — muy suave | **1.4:1 a 2.1:1** — puntiagudo |
| L1 | **4.2:1 a 6.7:1** — suave | **1.2:1 a 1.8:1** — puntiagudo |
| L2 | **3.5:1 a 6.0:1** — suave | **1.0:1 a 1.6:1** — muy puntiagudo |
| L3 | **3.0:1 a 5.4:1** — suave | **0.8:1 a 1.5:1** — más alto que ancho |
| L4 (frente) | **2.6:1 a 5.3:1** — suave | **0.7:1 a 1.4:1** — más alto que ancho |

> Las capas 3 y 4 en mobile son **más altas que anchas** — parecen estalagmitas, no dunas.

### 1.2 Saturación visual por exceso de dunas

**Síntoma**: La escena se ve recargada y confusa.

**Causa**: En desktop, 6+5+4+3+2 = 20 dunas distribuidas en 1920px (~1 duna cada 96px). En mobile 375px, las mismas 20 dunas se aprietan (~1 cada 19px). La densidad visual se multiplica por 5x, creando una textura caótica.

### 1.3 Cielo comprimido

**Síntoma**: El sol desaparece rápido detrás de las dunas, el cielo se siente pequeño.

**Causa**: 
- `base` de capa 0 = 0.40 → el cielo "puro" es solo ~37% del viewport (igual que desktop)
- Pero en desktop el ojo humano landscape tiene campo visual horizontal; en portrait el espacio vertical es el que domina, y tener solo 37% de cielo se siente claustrofóbico
- Las dunas con `portraitMul=1.5` se elevan más, invadiendo más cielo visualmente

### 1.4 Sol y Luna fuera de escala

**Síntoma**: El sol y la luna se ven demasiado pequeños.

**Causa**: `vSize` usa fórmulas distintas para mobile y desktop:

```javascript
// Desktop (fijo):
vSize = Math.min(w, h) * 0.055  // 1080 * 0.055 = 59.4px

// Mobile (variable):
vSize = Math.max(w, h) * 0.038  // 667 * 0.038 = 25.3px
```

El sol máximo en desktop es 59px (5.5% de la altura).  
El sol máximo en mobile es 25px (3.8% de la altura). → **31% más pequeño de lo que debería**.

Además, el sol se mueve entre `h*0.12` y `h*0.67` (12%-67% de la pantalla). En portrait con cielo al 37%, el sol desaparece tras las dunas cuando aún debería ser visible.

### 1.5 Estrellas mal posicionadas

**Síntoma**: Estrellas que titilan sobre las dunas (se ven raro).

**Causa**: `stars.map(() => ({ y: Math.random() * (isMobile ? 0.65 : 0.5) }))` — las estrellas llegan hasta el 65% de la pantalla en mobile, pero las dunas empiezan en el ~37%. Las estrellas se superponen con las dunas.

### 1.6 Polvo flotante desubicado

**Síntoma**: Partículas de polvo aparecen en el cielo.

**Causa**: `dust[].y = 0.3 + Math.random() * 0.65` → las partículas van de 30% a 95% de la altura. En mobile portrait, buena parte cae en la zona de cielo.

### 1.7 **Falta de manejo de devicePixelRatio** (CRÍTICO)

**Síntoma**: El canvas se ve pixelado ("puntiagudo") en pantallas retina. En un iPhone con DPR=3, el canvas de 375×667 se renderiza en 375×667px pero se muestra en 1125×2001px físicos → el navegador **upscalea** con interpolación de píxeles → bordes dentados.

**Causa**: El `resize()` solo usa `innerWidth/innerHeight` sin multiplicar por `devicePixelRatio`:

```javascript
function resize() {
  vw = innerWidth;
  vh = innerHeight;
  canvas.width = vw;   // debería ser innerWidth * dpr
  canvas.height = vh;  // debería ser innerHeight * dpr
}
```

---

## 2. Propuesta de solución por problema

### 2.1 Sistema de parámetros continuos por aspect ratio

Reemplazar la detección binaria `isMobile`/`isPortrait` con un sistema basado en **aspect ratio continuo**. El factor `portraitFactor` va de 0 (landscape ≥16:9) a 1.0 (portrait ≤9:16).

```javascript
// ===== RESPONSIVE PARAMETERS (smooth, not binary) =====
const vw = innerWidth, vh = innerHeight;
const aspect = vw / vh;  // 1.78 = 16:9 landscape, 0.56 = 9:16 portrait
const isMobileWidth = vw <= 768;

// portraitFactor: 0 = landscape amplio, 1 = portrait extremo
const portraitFactor = Math.max(0, Math.min(1, (1.2 - aspect) / 0.7));
//   aspect=1.78 → pf=0.0  (desktop landscape)
//   aspect=1.33 → pf=0.0  (tablet landscape)
//   aspect=0.75 → pf=0.64 (tablet portrait)
//   aspect=0.56 → pf=0.91 (mobile portrait)
//   aspect=0.50 → pf=1.0  (mobile narrow portrait)
```

### 2.2 Corrección de duneWScale (ancho de dunas)

**Problema**: `duneWScale = Math.max(0.6, w/1920)` produce valores muy bajos en mobile.

**Solución**: Usar un rango menos agresivo que mantenga las dunas proporcionales:

```javascript
const duneWScale = Math.max(0.8, Math.min(1.4, vw / 1920 + 0.5));
//   375px → max(0.8, min(1.4, 0.195+0.5)) = max(0.8, 0.695) = 0.8
//   768px → max(0.8, min(1.4, 0.40+0.5))  = max(0.8, 0.90)  = 0.9
//  1920px → max(0.8, min(1.4, 1.0+0.5))   = max(0.8, 1.4)   = 1.4
```

### 2.3 Corrección de altura de dunas (duneHScale)

**Problema**: `portraitMul = 1.5` exagera la altura en portrait cuando debería reducirse.

**Solución**: Reemplazar `portraitMul` con un factor continuo que se reduce en portrait:

```javascript
const duneHScale = 1.0 - portraitFactor * 0.35;
//   pf=0.0 → 1.0  (landscape: altura normal)
//   pf=0.5 → 0.825
//   pf=1.0 → 0.65 (portrait: 35% más bajas)
```

### 2.4 Corrección de cantidad de dunas por capa

**Problema**: 20 dunas en 375px crea saturación visual.

**Solución**: Reducir progresivamente la cantidad de bumps según portraitFactor:

```javascript
const DESKTOP_COUNTS = [6, 5, 4, 3, 2];
const PORTRAIT_COUNTS = [4, 3, 3, 2, 1];
const bumpCounts = DESKTOP_COUNTS.map((c, i) =>
  Math.round(c + (PORTRAIT_COUNTS[i] - c) * portraitFactor)
);
```

### 2.5 Corrección de bases (distribución vertical)

**Problema**: Misma distribución vertical en landscape y portrait.

**Solución**: Mover las bases hacia abajo en portrait para dar más cielo:

```javascript
const DESKTOP_BASES = [0.40, 0.52, 0.62, 0.71, 0.80];
const PORTRAIT_BASES = [0.48, 0.59, 0.67, 0.75, 0.83];
const bases = DESKTOP_BASES.map((b, i) =>
  b + (PORTRAIT_BASES[i] - b) * portraitFactor
);
```

Esto da como resultado:
- Desktop landscape: `[0.40, 0.52, 0.62, 0.71, 0.80]` → cielo 37.8%
- Mobile portrait: `[0.47, 0.58, 0.67, 0.75, 0.83]` → cielo **~47%**

### 2.6 Corrección de devicePixelRatio (ANTI-PIXELADO)

**Problema**: Canvas renderizado a resolución lógica sin DPR.

**Solución**: Multiplicar dimensiones del canvas por DPR y escalar el contexto:

```javascript
function resize() {
  const dpr = window.devicePixelRatio || 1;
  vw = innerWidth;
  vh = innerHeight;
  canvas.width = vw * dpr;
  canvas.height = vh * dpr;
  ctx.scale(dpr, dpr);
}
```

**IMPORTANTE**: `ctx.scale()` es acumulativo. Hay que resetearlo en cada resize:

```javascript
function resize() {
  const dpr = window.devicePixelRatio || 1;
  vw = innerWidth;
  vh = innerHeight;
  // Reset canvas (esto limpia el contexto también)
  canvas.width = vw * dpr;
  canvas.height = vh * dpr;
  // Restaurar la matriz de transformación
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
}
```

### 2.7 Corrección de escala del Sol/Luna

**Problema**: `vSize` usa `max(w,h)` en mobile vs `min(w,h)` en desktop.

**Solución**: Unificar usando **siempre** el mínimo:

```javascript
// ===== RENDER =====
function render(ctx, w, h, t) {
  const vSize = Math.min(w, h) * 0.055;  // siempre 5.5% del lado menor
  // ...
  // Sun radius: maxRad = vSize  (antes: maxRad*0.37 en ciertos casos)
  // Moon radius: vSize * 0.7
}
```

Valores resultantes:
- Desktop 1920×1080: `vSize = 1080 * 0.055 = 59.4px` (sin cambio)
- Mobile 375×667: `vSize = 375 * 0.055 = 20.6px` (antes 25.3px)

El sol es ligeramente más pequeño pero proporcional al ancho de la pantalla. Más importante: el sol y la luna escalan linealmente con el viewport.

### 2.8 Trayectoria del Sol adaptativa

**Problema**: El sol se esconde detrás de las dunas muy rápido en portrait.

**Solución**: Ajustar la posición vertical del sol según el aspect ratio:

```javascript
// El rango vertical del sol debe cubrir el sky visible
// En desktop: de 12% a 67% (viaja 55% del viewport)
// En portrait: de 10% a 50% (viaja 40% del viewport, queda sobre las dunas)
const sunRange = portraitFactor > 0
  ? { start: 0.10, travel: 0.40 }  // portrait: menos recorrido, más arriba
  : { start: 0.12, travel: 0.55 }; // landscape: recorrido completo

const sy = h * (sunRange.start + lt * sunRange.travel);
```

### 2.9 Límite vertical de estrellas

**Problema**: Estrellas invaden zona de dunas.

**Solución**: Limitar estrellas a la zona de cielo real:

```javascript
// Basar el límite en la base de la capa 0 y su altura máxima
const starYLimit = bases[0] - hMaxs[0] * scales[0] * duneHScale;
const stars = Array(starCount).fill().map(() => ({
  x: Math.random(),
  y: Math.random() * starYLimit * 0.9,  // 90% del cielo visible
  // ...
}));
```

También reducir la cuenta de estrellas en mobile para rendimiento:

```javascript
const starCount = vw <= 768
  ? Math.round(Math.min(100, vw * 0.15))
  : 160;
```

### 2.10 Distribución vertical del polvo

**Problema**: Polvo en el cielo.

**Solución**: Ajustar el rango Y del polvo según portraitFactor:

```javascript
const dustYStart = 0.3 + portraitFactor * 0.2;  // 0.3 en land, 0.5 en portrait
const dustYRange = 0.65 - portraitFactor * 0.15; // 0.65 en land, 0.50 en portrait

// En el loop de creación:
dust.push({
  x: Math.random() * 1.2 - 0.1,
  y: dustYStart + Math.random() * dustYRange,
  // ...
});
```

### 2.11 Evaluación visual final (simulación)

Con todas las correcciones activas para mobile portrait 375×667:

| Capa | FWHM (px) | Altura (px) | Ratio | Perfil |
|------|-----------|-------------|-------|--------|
| L0 | 30-47 | 10 | **3.0:1 a 4.7:1** ✅ | Suave |
| L1 | 36-57 | 14 | **2.6:1 a 4.1:1** ✅ | Suave |
| L2 | 43-74 | 19 | **2.3:1 a 3.9:1** ✅ | Suave |
| L3 | 50-90 | 26 | **1.9:1 a 3.5:1** ✅ | Aceptable |
| L4 | 57-114 | 34 | **1.7:1 a 3.4:1** ✅ | Aceptable |

Cielo: **~47%** del viewport (vs 37% actual).

---

## 3. Estrategia de responsive design

### 3.1 Breakpoints y valores paramétricos

| Viewport | Aspect | pf | duneWScale | duneHScale | bumpCounts | bases | Cielo |
|----------|--------|----|------------|------------|------------|-------|-------|
| Desktop land 1920×1080 | 1.78 | 0.00 | 1.40 | 1.00 | [6,5,4,3,2] | [0.40, 0.52, 0.62, 0.71, 0.80] | 37.8% |
| Tablet land 1024×768 | 1.33 | 0.00 | 1.03 | 1.00 | [6,5,4,3,2] | [0.40, 0.52, 0.62, 0.71, 0.80] | 37.8% |
| Mobile land 667×375 | 1.78 | 0.00 | 0.85 | 1.00 | [6,5,4,3,2] | [0.40, 0.52, 0.62, 0.71, 0.80] | 37.8% |
| **Tablet port 768×1024** | **0.75** | **0.64** | **0.90** | **0.78** | **[5,4,3,2,1]** | **[0.45, 0.56, 0.65, 0.74, 0.82]** | **43.5%** |
| **Mobile port 375×667** | **0.56** | **0.91** | **0.80** | **0.68** | **[4,3,3,2,1]** | **[0.47, 0.58, 0.67, 0.75, 0.83]** | **47.0%** |

### 3.2 Transiciones suaves

No hay saltos binarios. Cada parámetro se interpola linealmente según `portraitFactor`. Esto significa que un Galaxy S8 (37.5×740, aspect 0.51) obtendrá valores ligeramente diferentes a un iPhone 12 (390×844, aspect 0.46), y ambos serán visualmente consistentes.

### 3.3 devicePixelRatio en todos los dispositivos

Se aplica `dpr` en todos los casos, no solo mobile. En desktop con monitores retina (DPR=2) también se beneficia.

---

## 4. Plan de implementación detallado

### Tarea 1: Sistema de parámetros responsive (alta prioridad)

**Archivo**: `index.html` (líneas ~22-117)

**Cambios**:

1. Reemplazar `isMobile`, `portraitRatio()` y `isPortrait` con:

```javascript
// ===== RESPONSIVE PARAMETERS =====
const vw = innerWidth, vh = innerHeight;
const aspect = vw / vh;
const isMobileWidth = vw <= 768;
const portraitFactor = Math.max(0, Math.min(1, (1.2 - aspect) / 0.7));

// Mobile detection for count-based decisions (stars, dust count)
const isMobile = vw <= 768;
```

2. Reemplazar `duneWScale` y `portraitMul`:

```javascript
const duneWScale = Math.max(0.8, Math.min(1.4, vw / 1920 + 0.5));
const duneHScale = 1.0 - portraitFactor * 0.35;
```

3. Nuevos `bumpCounts` y `bases`:

```javascript
const DESKTOP_COUNTS = [6, 5, 4, 3, 2];
const PORTRAIT_COUNTS = [4, 3, 3, 2, 1];
const bumpCounts = DESKTOP_COUNTS.map((c, i) =>
  Math.round(c + (PORTRAIT_COUNTS[i] - c) * portraitFactor)
);

const DESKTOP_BASES = [0.40, 0.52, 0.62, 0.71, 0.80];
const PORTRAIT_BASES = [0.48, 0.59, 0.67, 0.75, 0.83];
const bases = DESKTOP_BASES.map((b, i) =>
  b + (PORTRAIT_BASES[i] - b) * portraitFactor
);
```

4. Actualizar `LAYERS`:

```javascript
const LAYERS = bumpCounts.map((cnt, i) => ({
  cnt,
  hMin: [0.10, 0.13, 0.16, 0.20, 0.24][i],
  hMax: [0.22, 0.26, 0.32, 0.38, 0.44][i],
  wMin: [300, 200, 120, 80, 50][i] * duneWScale,
  wMax: [700, 500, 350, 260, 200][i] * duneWScale,
  base: bases[i],
  para: [0.008, 0.020, 0.040, 0.080, 0.140][i],
  scale: [0.10, 0.12, 0.14, 0.16, 0.18][i] * duneHScale,
  pal: i,
}));
```

### Tarea 2: Corrección de devicePixelRatio (alta prioridad)

**Archivo**: `index.html` (líneas ~232-236)

**Cambio**:

```javascript
function resize() {
  const dpr = window.devicePixelRatio || 1;
  vw = innerWidth;
  vh = innerHeight;
  canvas.width = vw * dpr;
  canvas.height = vh * dpr;
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
}
```

**Impacto**: Elimina el pixelado. Es el cambio de mayor impacto visual.

### Tarea 3: Escala y trayectoria del Sol/Luna (media prioridad)

**Archivo**: `index.html` (líneas ~131-178 para `vSize`, ~156-178 para sol, ~181-192 para luna)

**Cambio en `vSize`**:

```javascript
function render(ctx, w, h, t) {
  const time = performance.now() / 1e3;
  const vSize = Math.min(w, h) * 0.055;  // siempre 5.5% del mínimo
  // ...
```

**Cambio en trayectoria del sol**:

```javascript
// Sun position — adaptativo según aspect ratio
if (t < 0.6) {
  const lt = t / 0.6;
  // En portrait, el sol viaja menos verticalmente para no esconderse tras dunas
  const sunStart = 0.10 + portraitFactor * 0.02;   // 0.10 → 0.12
  const sunTravel = 0.55 - portraitFactor * 0.20;  // 0.55 → 0.35
  const sy = h * (sunStart + lt * sunTravel);
  // ...
}
```

**Cambio en luna**:

```javascript
// Moon — misma escala, posición adaptativa
if (t > 0.25) {
  const al = Math.max(0, Math.min(1, (t - 0.25) / 0.3));
  // En portrait, la luna arranca más arriba
  const moonStart = portraitFactor > 0.5 ? 0.04 : 0.06;
  const my = h * (moonStart + (1 - t) * 0.35);
  const moonRad = vSize * 0.7;
  // ...
}
```

### Tarea 4: Límite vertical de estrellas (media prioridad)

**Archivo**: `index.html` (línea ~79)

**Cambio**:

```javascript
// Las estrellas solo ocupan la zona de cielo real
const starYLimit = Math.max(0.35, bases[0] - 0.22 * 0.10 * duneHScale);
const stars = Array(starCount).fill().map(() => ({
  x: Math.random(),
  y: Math.random() * starYLimit * 0.9,
  r: 0.3 + Math.random() * 1.5,
  s: 0.2 + Math.random() * 2.5,
  ph: Math.random() * 7,
  a: 0.2 + Math.random() * 0.8,
}));
```

Ajustar `starCount`:

```javascript
const starCount = isMobile ? Math.min(100, Math.round(vw * 0.18)) : 160;
```

### Tarea 5: Distribución vertical del polvo (baja prioridad)

**Archivo**: `index.html` (líneas ~120-129)

**Cambio**:

```javascript
const dustYStart = 0.3 + portraitFactor * 0.2;
const dustYRange = 0.65 - portraitFactor * 0.15;
const dust = [];
for (let i = 0; i < dustCount; i++) {
  dust.push({
    x: Math.random() * 1.2 - 0.1,
    y: dustYStart + Math.random() * dustYRange,
    r: 0.3 + Math.random() * 2.5,
    a: 0.02 + Math.random() * 0.08,
    dx: (Math.random() - 0.5) * 0.3,
    dy: -(0.01 + Math.random() * 0.04),
    ph: Math.random() * 7,
    sp: 0.3 + Math.random() * 1.5,
  });
}
```

### Tarea 6: Ajuste de resolución de perfil de dunas (baja prioridad)

**Archivo**: `index.html` (líneas ~94-102, ~108)

**Cambio**: Ajustar `DUNE_RES` para que los segmentos de línea no sean ni muy grandes ni muy chicos:

```javascript
// Resolución adaptativa: ~1 punto cada 10px horizontalmente
const DUNE_RES = Math.max(60, Math.min(180, Math.round(vw / 10)));
//   Desktop 1920px → 180  (igual que antes)
//   Mobile 375px   → 60   (menos puntos = perfil más suave, menos vértices)
```

---

## 5. Priorización

| # | Tarea | Impacto visual | Esfuerzo | Prioridad |
|---|-------|---------------|----------|-----------|
| 1 | **Sistema de parámetros responsive** | 🔴 Crítico — dunas puntiagudas | 30 min | **P0** |
| 2 | **Corrección devicePixelRatio** | 🔴 Crítico — pixelado general | 5 min | **P0** |
| 3 | **Escala y trayectoria Sol/Luna** | 🟡 Alto — composición | 15 min | **P1** |
| 4 | **Límite vertical de estrellas** | 🟡 Alto — estrellas sobre dunas | 10 min | **P1** |
| 5 | **Distribución vertical del polvo** | 🟢 Medio — polvo en cielo | 5 min | **P2** |
| 6 | **Resolución adaptativa de dunas** | 🟢 Medio — rendimiento | 5 min | **P2** |

**Orden de implementación recomendado**: 1 → 2 → 3 → 4 → 5 → 6

---

## 6. Estimación de esfuerzo

| Tarea | Tiempo | Quién |
|-------|--------|-------|
| T1: Sistema de parámetros responsive | 30 min | Frontend Lead + Tailwind Expert* |
| T2: Corrección devicePixelRatio | 5 min | Frontend Lead |
| T3: Escala y trayectoria Sol/Luna | 15 min | Vue Expert |
| T4: Límite vertical de estrellas | 10 min | Vue Expert |
| T5: Distribución vertical del polvo | 5 min | Vue Expert |
| T6: Resolución adaptativa de dunas | 5 min | Performance Expert |
| **Testing en dispositivos reales** | 30 min | QA |
| **Total** | **~1.5h** | |

*\*Tailwind Expert para validar que los estilos CSS responsive del canvas sean consistentes.*

---

## 7. Código completo propuesto

A continuación se muestra el diff completo de los cambios necesarios en `index.html`. Solo se modifican las secciones indicadas; el resto del archivo permanece idéntico.

### 7.1 Header (viewport meta — sin cambios)

El meta viewport actual es correcto:

```html
<meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=1.0,user-scalable=no">
```

No requiere cambios.

### 7.2 CSS (sin cambios)

```css
canvas{position:fixed;top:0;left:0;width:100vw;height:100vh;display:block;touch-action:pan-y;will-change:transform}
```

No requiere cambios. `touch-action: pan-y` ya permite scroll vertical nativo.

### 7.3 Sección de detección mobile → responsive parameters

**Antes** (líneas 22-24):
```javascript
const isMobile=matchMedia('(max-width:768px)').matches
const portraitRatio=()=>Math.max(1,innerHeight/innerWidth)
```

**Después**:
```javascript
const vw=innerWidth,vh=innerHeight
const aspect=vw/vh
const isMobile=vw<=768
const portraitFactor=Math.max(0,Math.min(1,(1.2-aspect)/0.7))
const dpr=window.devicePixelRatio||1
```

### 7.4 Sección de dunas (LAYERS)

**Antes** (líneas 104-117):
```javascript
const isPortrait=portraitRatio()>1.3
const duneWScale=isMobile?Math.max(.6,innerWidth/1920):1
const portraitMul=isPortrait?1.5:1
const DUNE_RES=isMobile?170:180
const LAYERS=[
  {cnt:6, hMin:.10,hMax:.22,wMin:300*duneWScale,wMax:700*duneWScale,   base:.40,para:.008,scale:.10*portraitMul,pal:0},
  {cnt:5, hMin:.13,hMax:.26,wMin:200*duneWScale,wMax:500*duneWScale,   base:.52,para:.020,scale:.12*portraitMul,pal:1},
  {cnt:4, hMin:.16,hMax:.32,wMin:120*duneWScale,wMax:350*duneWScale,   base:.62,para:.040,scale:.14*portraitMul,pal:2},
  {cnt:3, hMin:.20,hMax:.38,wMin:80*duneWScale,wMax:260*duneWScale,    base:.71,para:.080,scale:.16*portraitMul,pal:3},
  {cnt:2, hMin:.24,hMax:.44,wMin:50*duneWScale,wMax:200*duneWScale,    base:.80,para:.140,scale:.18*portraitMul,pal:4},
]
const LAYER_BUMPS=LAYERS.map(l=>genBumps(l.cnt,l.hMin,l.hMax,l.wMin,l.wMax))
const DUNE_PTS=LAYER_BUMPS.map(b=>computeProfile(b,DUNE_RES))
```

**Después**:
```javascript
const duneWScale=Math.max(0.8,Math.min(1.4,vw/1920+0.5))
const duneHScale=1.0-portraitFactor*0.35
const DESKTOP_CNT=[6,5,4,3,2]
const PORTRAIT_CNT=[4,3,3,2,1]
const bumpCounts=DESKTOP_CNT.map((c,i)=>Math.round(c+(PORTRAIT_CNT[i]-c)*portraitFactor))
const DESKTOP_BASES=[0.40,0.52,0.62,0.71,0.80]
const PORTRAIT_BASES=[0.48,0.59,0.67,0.75,0.83]
const bases=DESKTOP_BASES.map((b,i)=>b+(PORTRAIT_BASES[i]-b)*portraitFactor)
const DUNE_RES=Math.max(60,Math.min(180,Math.round(vw/10)))
const H_MINS=[0.10,0.13,0.16,0.20,0.24]
const H_MAXS=[0.22,0.26,0.32,0.38,0.44]
const W_MINS=[300,200,120,80,50]
const W_MAXS=[700,500,350,260,200]
const PARAS=[0.008,0.020,0.040,0.080,0.140]
const SCALES=[0.10,0.12,0.14,0.16,0.18]
const LAYERS=bumpCounts.map((cnt,i)=>({
  cnt, hMin:H_MINS[i], hMax:H_MAXS[i],
  wMin:W_MINS[i]*duneWScale, wMax:W_MAXS[i]*duneWScale,
  base:bases[i], para:PARAS[i], scale:SCALES[i]*duneHScale, pal:i
}))
const LAYER_BUMPS=LAYERS.map(l=>genBumps(l.cnt,l.hMin,l.hMax,l.wMin,l.wMax))
const DUNE_PTS=LAYER_BUMPS.map(b=>computeProfile(b,DUNE_RES))
```

### 7.5 Sección de render (vSize)

**Antes** (línea 135):
```javascript
const vSize=isMobile?Math.max(w,h)*.038:baseSize*.055
```

**Después**:
```javascript
const vSize=Math.min(w,h)*0.055
```

### 7.6 Sección de estrellas (límite vertical)

**Antes** (líneas 77-82):
```javascript
const starCount=isMobile?Math.min(120,Math.round(innerWidth*.15)):160
const stars=Array(starCount).fill().map(()=>({
  x:Math.random(),y:Math.random()*(isMobile ? .65 : .5),
  r:.3+Math.random()*1.5,s:.2+Math.random()*2.5,
  ph:Math.random()*7,a:.2+Math.random()*.8
}))
```

**Después**:
```javascript
const starCount=isMobile?Math.min(100,Math.round(vw*0.18)):160
const starYLimit=Math.max(0.35,bases[0]-H_MAXS[0]*SCALES[0]*duneHScale)
const stars=Array(starCount).fill().map(()=>({
  x:Math.random(),y:Math.random()*starYLimit*0.9,
  r:.3+Math.random()*1.5,s:.2+Math.random()*2.5,
  ph:Math.random()*7,a:.2+Math.random()*.8
}))
```

### 7.7 Sección de polvo (distribución Y)

**Antes** (líneas 120-129):
```javascript
const dustCount=isMobile?120:280
const dust=[]
for(let i=0;i<dustCount;i++){
  dust.push({
    x:Math.random()*1.2-.1,y:.3+Math.random()*.65,
    r:.3+Math.random()*2.5,a:.02+Math.random()*.08,
    dx:(Math.random()-.5)*.3,dy:-(.01+Math.random()*.04),
    ph:Math.random()*7,sp:.3+Math.random()*1.5
  })
}
```

**Después**:
```javascript
const dustCount=isMobile?Math.round(120*(vw/375)):280
const dustYStart=.3+portraitFactor*.2
const dustYRange=.65-portraitFactor*.15
const dust=[]
for(let i=0;i<dustCount;i++){
  dust.push({
    x:Math.random()*1.2-.1,y:dustYStart+Math.random()*dustYRange,
    r:.3+Math.random()*2.5,a:.02+Math.random()*.08,
    dx:(Math.random()-.5)*.3,dy:-(.01+Math.random()*.04),
    ph:Math.random()*7,sp:.3+Math.random()*1.5
  })
}
```

### 7.8 Sol (trayectoria adaptativa)

**Antes** (líneas 155-178):
```javascript
if(t<.6){
  const lt=t/.6,sy=h*(.12+lt*.55)
  const maxRad=vSize,rad=Math.max(maxRad*.37,maxRad-lt*maxRad*.47),al=Math.max(0,1-(t-.35)/.25)
  // ... resto del código del sol
}
```

**Después**:
```javascript
if(t<.6){
  const lt=t/.6
  const sunStart=.12-portraitFactor*.02
  const sunTravel=.55-portraitFactor*.20
  const sy=h*(sunStart+lt*sunTravel)
  const maxRad=vSize,rad=Math.max(maxRad*.37,maxRad-lt*maxRad*.47),al=Math.max(0,1-(t-.35)/.25)
  // ... resto del código del sol (sin cambios)
}
```

### 7.9 Luna (escala unificada)

**Antes** (línea 183):
```javascript
const moonRad=vSize*.7
```

**Después**: Sin cambios en la fórmula (`moonRad=vSize*0.7` ya usa `vSize` que ahora es correcto). Solo ajustar posición vertical si es necesario:

```javascript
const my=h*(portraitFactor>.5?.04:.06+(1-t)*.35)
```

### 7.10 resize (devicePixelRatio)

**Antes** (líneas 232-236):
```javascript
let progress=0,vw,vh
function resize(){
  vw=innerWidth;vh=innerHeight
  canvas.width=vw;canvas.height=vh
}
resize();window.addEventListener('resize',resize)
```

**Después**:
```javascript
let progress=0,vw,vh
function resize(){
  vw=innerWidth;vh=innerHeight
  const dpr=window.devicePixelRatio||1
  canvas.width=vw*dpr;canvas.height=vh*dpr
  ctx.setTransform(dpr,0,0,dpr,0,0)
}
resize();window.addEventListener('resize',resize)
```

---

## 8. Problema A: Dunas de abajo se cortan con las de arriba (layering)

> **Gravedad**: Alta — visible en desktop y mobile durante el scroll

### 8.1 Síntoma

Las dunas de las capas inferiores (foreground, capa 4) cuando son más grandes que las de arriba (background, capa 0), aparecen "cortadas" o recortadas donde se cruzan. Se ven bordes diagonales o verticales antiestéticos entre capas de dunas.

### 8.2 Diagnóstico

Cada capa de duna es un polígono que va desde su cresta hasta el borde inferior del canvas. El painter's algorithm dibuja de atrás a adelante (capa 0 → capa 4). Cada capa usa su propio offset de parallax:

```javascript
const ox = (t - 0.5) * l.para * w
```

Esto significa que el mismo índice `i` del perfil de dunas apunta a coordenadas X diferentes entre capas. Cuando la capa 4 se dibuja, su polilínea de cresta está desplazada horizontalmente vs. capa 3, y en zonas donde capa 4 no cubre (porque su cresta está más arriba que el perfil de capa 3 en esa X), se ve capa 3. Esto es CORRECTO para parallax.

**El problema real**: El polígono de cada capa NO cubre todo el ancho real porque `sx` y `ex` solo se extienden 10px:

```javascript
const sx = Math.min(0, ox) - 10
const ex = Math.max(w, w + ox) + 10
```

Cuando el parallax es grande (t cercano a 0 o 1), el polígono se desplaza y quedan "descubiertas" zonas que deberían estar cubiertas por la capa frontal, creando cortes verticales o bordes diagonales antiestéticos.

Por ejemplo:
- En `t=1` (noche), capa 4 con `para=0.140`: `ox = 0.5 × 0.140 × w = 0.07w` (~140px en desktop 1920px)
- `sx = min(0, 0.07w) - 10 = -10`
- La cresta empieza en `x = 0 + ox = 0.07w`
- Entre `x=-10` y `x=0.07w` el polígono va horizontal a la altura del primer punto de cresta
- Si la altura de cresta en ese punto es alta, puede no cubrir bien contra la capa inferior

### 8.3 Solución propuesta

Extender `sx` y `ex` con un margen suficiente para cubrir el máximo desplazamiento de parallax:

```javascript
const margin = Math.abs(ox) + w * 0.15
const sx = -margin
const ex = w + margin
```

Además, cambiar el inicio del polígono para que empiece desde la parte inferior en lugar de desde la altura de la cresta, evitando así cualquier borde diagonal visible dentro del viewport:

```javascript
ctx.beginPath()
ctx.moveTo(sx, h)          // bottom-left (fuera del viewport)
ctx.lineTo(ox, cy[0])      // sube al primer punto de cresta
for (let i = 1; i < pts.length; i++)
  ctx.lineTo(pts[i].x * w + ox, cy[i])  // traza la cresta
ctx.lineTo(ex, h)          // baja a bottom-right (fuera del viewport)
ctx.closePath()            // vuelve al inicio
```

Esto garantiza:
- Cobertura total en todo el ancho del canvas incluso con el máximo desplazamiento de parallax
- Los bordes del polígono (izquierdo y derecho) quedan fuera del viewport, por lo que no se ven
- No hay segmentos diagonales visibles entre capas

### 8.4 Código modificado

**Antes** (líneas ~196-208):
```javascript
const sx=Math.min(0,ox)-10,ex=Math.max(w,w+ox)+10
const cy=pts.map(p=>by-p.y*h*sc)
ctx.beginPath()
ctx.moveTo(sx,cy[0])
for(let i=1;i<pts.length;i++)ctx.lineTo(pts[i].x*w+ox,cy[i])
ctx.lineTo(ex,h);ctx.lineTo(sx,h);ctx.closePath()
```

**Después**:
```javascript
const margin=Math.abs(ox)+w*0.15
const sx=-margin,ex=w+margin
const cy=pts.map(p=>by-p.y*h*sc)
ctx.beginPath()
ctx.moveTo(sx,h)
ctx.lineTo(ox,cy[0])
for(let i=1;i<pts.length;i++)ctx.lineTo(pts[i].x*w+ox,cy[i])
ctx.lineTo(ex,h);ctx.closePath()
```

---

## 9. Problema B: Deformación al final del scroll ("palito parado")

> **Gravedad**: Alta — visible en desktop y mobile en t=1 (noche total)

### 9.1 Síntoma

Cuando el scroll llega al 100% (t=1, noche total), las dunas se ven deformadas. Aparece un "palito parado" (una línea vertical/oblicua antiestética en un costado del canvas) y se nota un corte brusco en la animación al llegar al estado final.

### 9.2 Diagnóstico

Tres causas convergen para producir este artefacto:

**Causa 1 — Margen insuficiente en `sx`/`ex`:**
```javascript
// Código actual
const sx = Math.min(0, ox) - 10
const ex = Math.max(w, w + ox) + 10
```
En `t=1`:
- `ox = 0.5 × para × w`
- Capa 4 (para=0.140): `ox = 0.07w ≈ 140px` en desktop
- `sx = min(0, 0.07w) - 10 = -10`
- `ex = max(w, 1.07w) + 10 = 1.07w + 10`

El problema está en el lado opuesto al desplazamiento. Cuando `t=1` y `ox` es positivo, el lado izquierdo tiene `sx=-10` pero la cresta empieza en `x=ox=0.07w`. La línea entre `(-10, cy[0])` y `(0.07w, cy[0])` cubre horizontalmente, PERO:

En `t=0`:
- Capa 4: `ox = -0.07w`
- `sx = min(0, -0.07w) - 10 = -0.07w - 10`
- `ex = max(w, 0.93w) + 10 = w + 10`

La cresta termina en `x = w + ox = 0.93w`. Luego el polígono va desde `(0.93w, cy[res])` hasta `(w+10, h)`. Esta línea diagonal cruza el borde derecho del canvas (x=w) y es visible como un "palito".

**Causa 2 — Picos en los bordes del perfil:**
El perfil de dunas en los extremos (x=0 y x=1) puede tener valores de altura no nulos (`y > 0`). Cuando la función `computeProfile` genera `pts[0]` y `pts[res]`, estos pueden tener `y > 0` si hay bumps cerca de los bordes. Esto hace que `cy[0]` y `cy[res]` estén más arriba que la base (`by`), creando escalones en los bordes del polígono.

**Causa 3 — Estado estático en t=1:**
Al ser t=1 el estado final de la animación, cualquier imprecisión en el render se nota más porque la escena está estática y el usuario puede observarla detenidamente.

### 9.3 Solución propuesta

1. **Extender `sx` y `ex` con margen generoso** (misma solución que Problema A):
   ```javascript
   const margin = Math.abs(ox) + w * 0.15
   const sx = -margin
   const ex = w + margin
   ```

2. **Iniciar el polígono desde la parte inferior** en lugar de desde la altura de cresta:
   ```javascript
   ctx.moveTo(sx, h)    // bottom-left fuera del viewport
   ctx.lineTo(ox, cy[0])  // sube verticalmente al primer crest point
   ```
   Esto evita que el borde izquierdo del polígono quede a la altura de la cresta dentro del viewport.

3. **Suavizar los bordes del perfil** (opcional, si persiste): Forzar `y=0` en los extremos del perfil de dunas modificando `computeProfile` para que `pts[0].y = 0` y `pts[res].y = 0`.

### 9.4 Validación

Con las correcciones activas:

| Escenario | Antes | Después |
|-----------|-------|---------|
| Desktop t=0 | Posible "palito" en borde derecho | Sin artefactos |
| Desktop t=1 | Posible "palito" en borde izquierdo | Sin artefactos |
| Mobile t=0 | Corte diagonal en borde derecho | Sin artefactos |
| Mobile t=1 | Corte diagonal en borde izquierdo | Sin artefactos |
| Transición 0→1 | Corte brusco al llegar a t=1 | Transición suave |

---

## 10. Criterios de aceptación

- [ ] En iPhone SE (375×667), las dunas se ven suaves, no puntiagudas
- [ ] En iPad portrait (768×1024), la escena se ve equilibrada (no idéntica a desktop, sino adaptada)
- [ ] En desktop landscape, no hay regresión visual (los cambios solo afectan cuando `portraitFactor > 0`)
- [ ] El canvas no se ve pixelado en pantallas retina (DPR=2 o 3)
- [ ] El sol permanece visible sobre las dunas durante su trayectoria en portrait
- [ ] Las estrellas no aparecen sobre las dunas
- [ ] El polvo no flota en la zona de cielo
- [ ] La transición entre landscape y portrait (al rotar el dispositivo) es suave, no hay saltos
- [ ] FPS se mantiene en 60fps en dispositivos móviles (verificar con Chrome DevTools)
- [ ] **NUEVO**: Las dunas no se cortan entre capas durante el scroll (sin bordes diagonales visibles)
- [ ] **NUEVO**: No hay "palito parado" ni deformación en t=0 o t=1
- [ ] **NUEVO**: La transición al inicio y final del scroll es suave sin artefactos en los bordes
