## Instalación, pruebas y ejecución headless

```bash
npm install          # instala dependencias (Vitest)
npm test             # corre las pruebas unitarias y de validación analítica
npm run headless     # corre el escenario fijo sin interfaz y escribe output/headless.csv
```

### Pruebas de validación analítica (tolerancias declaradas)

| ID | Escenario | Referencia analítica | Tolerancia | Error medido | Justificación |
|----|-----------|----------------------|-----------|--------------|---------------|
| V1 | Caída libre desde h = 100 m | t = √(2h/g) | 0.5 % | 0.01 % | dt = 1 ms; el error viene del paso del integrador y de la variación de g con la altura (despreciable en 100 m). |
| V2 | Proyectil horizontal (h = 100 m, v₀ = 50 m/s) | x = v₀·√(2h/g) | 0.5 % | 0.002 % | Mismo origen que V1; la tolerancia deja un margen amplio sobre el error del integrador. |
| V3 | Caída radial h = 1000 km (gravedad variable) | fórmula de Kepler radial | 1 % | 0.05 % | dt = 0.5 s; es más laxa porque la caída termina con velocidad alta y el cruce con la superficie se interpola linealmente. |
| V4 | Órbita circular a 300 km | T = 2π√(r³/μ), radio constante | 0.5 % | 0.0001 % (T), 0.06 % (r) | dt = 1 s sobre un período de unos 5400 s; el radio oscila levemente por la discretización de Euler semi-implícito. |
| V5 | Δv de circularización | 2.º impulso de Hohmann | 1e-6 | 2.6e-14 | Es una igualdad algebraica entre dos fórmulas cerradas: solo hay error de redondeo. |
| V6 | Apogeo de la órbita de transferencia | ecuación vis-viva | 0.5 % | 0.0004 % | dt = 1 s; el máximo se toma de muestras discretas, no del instante exacto del apogeo. |
| C1 | Energía mecánica, 10 órbitas | E constante | 1 % | 0.005 % | Es el 1 % que pide la consigna. Euler semi-implícito es simpléctico: la energía oscila acotada, no se desvía. |
| C2 | Momento angular, sin arrastre | L constante | 1e-9 | 6.4e-15 | La fuerza es central y el esquema conserva L salvo redondeo; 1e-9 es holgado frente a eso. |
| M1 | Circularización en el apogeo | v tras el impulso = v_circ(r) y Δv = v_circ − v_apo | 1e-12 (v), 1e-9 (Δv) | 1.3e-16 (v), 2.3e-14 (Δv) | Comparación contra fórmulas cerradas: solo hay error de redondeo. |
| C3 | Con atmósfera | la energía total disminuye | — | — | Prueba cualitativa: el arrastre solo puede disipar energía. |
| E1–E5 | Velocidad de escape, clasificación y barrido | E = 0 en v_esc; fronteras coherentes | 1e-12 (E1), ±0.1 % (E2), ±1.5 % (E5) | 0 (E1) | Igualdades de fórmulas cerradas (redondeo) y fronteras de clasificación. |
| H1–H4 | Escenario headless (CSV) | Cabecera y columnas correctas; mismo CSV en dos corridas; evento de circularización; dictamen "órbita conforme" | exacto / 2 % (radio final) / 0.1 % (Δv) | 0.0001 % (radio final), 0.002 % (Δv) | El 2 % es la tolerancia del dictamen del Avance 1. El Δv simulado se compara con el analítico de vis-viva. |
| P1–P3, I | Pureza, determinismo e independencia de la interfaz | — | exacto | — | Comparaciones exactas: no hay error numérico que tolerar. |

Los errores medidos corresponden al código y a los pasos de tiempo de estas pruebas; si se cambia el integrador o algún `dt`, hay que volver a medirlos.

### Escenario headless

Tierra (M = 5.972×10²⁴ kg, R = 6371 km), torre a 300 km, órbita objetivo a 1200 km, dt = 1 s.
Se lanza con la v₀ de la transferencia, se circulariza en el apogeo y se simulan 2 vueltas más.
El CSV (`output/headless.csv`) trae t, x, y, vx, vy, r, altura, rapidez, Ek, Ep, E total (J/kg), L (m²/s) y el evento
(`lanzamiento`, `circularizacion`, `fin`). La consola imprime v₀, Δv_circ, Δv_total y el dictamen.

## API pública de la librería de física (`src/physics/`)

La librería no depende de la interfaz (no usa `document`, `window` ni `canvas`). Todas las funciones son **puras y deterministas** (mismo estado + mismo `dt` ⇒ mismo resultado; no hay aleatoriedad) y **todo está en unidades del SI**: metros, segundos, kilogramos, m/s, J/kg. Las conversiones (km, km/s, 10²⁴ kg) ocurren solo en la interfaz.

Se importa todo desde un único punto de entrada:

```js
import { step, createInitialState, computeEnergy /* … */ } from './src/physics/index.js';
```

### Convenciones

**Estado** `s` (objeto que `step` devuelve nuevo, sin modificar el anterior):

| Campo | Significado | Unidad |
|-------|-------------|--------|
| `x`, `y` | posición respecto al centro del planeta | m |
| `vx`, `vy` | velocidad | m/s |
| `t` | tiempo transcurrido | s |

**Parámetros** `p` (según la función, se usa solo un subconjunto):

| Campo | Significado | Unidad |
|-------|-------------|--------|
| `M` | masa del planeta | kg |
| `R` | radio del planeta | m |
| `h` | altura de la torre sobre la superficie | m |
| `v0` | rapidez de lanzamiento (horizontal) | m/s |
| `atmosfera` | `true` activa el arrastre | — |
| `altAtmosfera` | altura donde termina la atmósfera | m |
| `rho0` | densidad del aire a nivel del suelo | kg/m³ |
| `escalaAltura` | altura de escala de la densidad exponencial | m |
| `Cd` | coeficiente de arrastre | adimensional |
| `areaSobreMasa` | área frontal / masa del proyectil | m²/kg |

### Constantes

| Nombre | Valor | Unidad |
|--------|-------|--------|
| `G` | 6.674×10⁻¹¹ | N·m²/kg² |

### Simulación

| Función | Recibe | Devuelve |
|---------|--------|----------|
| `createInitialState(p)` | `p`: `{R, h, v0}` | Estado inicial en la cima de la torre, lanzado en horizontal: `{x:0, y:R+h, vx:v0, vy:0, t:0}` |
| `step(s, p, dt)` | `s`: estado, `p`: `{M, R, atmosfera, …}`, `dt` (s) | Nuevo estado tras un paso de Euler semi-implícito (simpléctico). Suma gravedad y, si `p.atmosfera`, arrastre. No modifica `s`. |
| `computeAcceleration(s, M)` | `s`: `{x, y}` (m), `M` (kg) | `{ax, ay}` en m/s², aceleración gravitatoria dirigida al centro |
| `computeDragAcceleration(s, p)` | `s`: estado, `p`: `{atmosfera, R, altAtmosfera, rho0, escalaAltura, Cd, areaSobreMasa}` | `{ax, ay}` en m/s². `{0, 0}` si la atmósfera está apagada o el proyectil está fuera de ella (o bajo la superficie). Modelo provisional: densidad exponencial. |

### Energía y velocidades características

| Función | Recibe | Devuelve |
|---------|--------|----------|
| `computeEnergy(s, M)` | estado `s`, `M` (kg) | `{kinetic, potential, total, r, v}`: energías específicas en J/kg (por unidad de masa del proyectil), `r` en m, `v` en m/s. La energía potencial es negativa (cero en el infinito). |
| `escapeVelocity(M, r)` | `M` (kg), `r` (m, distancia al centro) | Velocidad de escape en m/s |
| `circularVelocity(M, r)` | `M` (kg), `r` (m) | Velocidad de órbita circular en m/s |
| `angularMomentum(s)` | estado `s` | Momento angular específico `x·vy − y·vx` en m²/s |

### Maniobras y clasificación (soluciones analíticas)

| Función | Recibe | Devuelve |
|---------|--------|----------|
| `transferGeometry(p)` | `p`: `{M, R, h, v0}` | `{escape, E, L, r0, a, rApo, rPeri, ecc}`: geometría de la órbita de transferencia (el punto de lanzamiento es el perigeo). `E` en J/kg, `L` en m²/s, `r0`, `a`, `rApo`, `rPeri` en m, `ecc` adimensional. Si `E ≥ 0`: `escape: true`, `a = rApo = Infinity`, `ecc = 1`. |
| `circularizationDeltaV(p)` | `p`: `{M, R, h, v0}` | `{deltaV, vApo, vCirc, geo}`: Δv necesario en el apogeo para circularizar (vis-viva), en m/s. Si hay escape: `deltaV = NaN`. |
| `launchVelocityForApogee(M, r0, rApo)` | `M` (kg), `r0` (m, radio de lanzamiento), `rApo` (m, radio del apogeo deseado) | `v0` en m/s que da esa órbita de transferencia |
| `applyCircularization(s, M)` | `s`: estado en el apogeo, `M` (kg) | `{newState, deltaV}`: estado con la velocidad escalada a la circular (misma dirección) y el Δv aplicado en m/s |
| `classifyOutcome(p, tol = 0.02)` | `p`: `{M, R, h, v0}`, `tol`: excentricidad máxima para llamarla circular | `"impacto"`, `"eliptica"`, `"circular"` o `"escape"` |
| `sweepLaunchVelocities(p, vMin, vMax, n = 120)` | `p`: `{M, R, h}`, `vMin`/`vMax` (m/s), `n` puntos | Arreglo `[{v0, outcome}]` con `v0` en m/s y `outcome` como en `classifyOutcome` |

### Ejemplo de uso

```js
import {
  createInitialState, step, computeEnergy, launchVelocityForApogee
} from './src/physics/index.js';

const M = 5.972e24, R = 6.371e6, h = 300e3;          // Tierra, torre a 300 km
const v0 = launchVelocityForApogee(M, R + h, R + 1200e3);
const p = { M, R, h, v0, atmosfera: false };

let s = createInitialState(p);
for (let i = 0; i < 600; i++) s = step(s, p, 1);      // 600 s con dt = 1 s
console.log(s.t, computeEnergy(s, M).total);          // s, J/kg
```
