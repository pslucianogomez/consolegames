# MATE.EXE — Diseño (spec)

- **Fecha:** 2026-05-25
- **Estado:** Aprobado para implementación
- **Repo:** `consolegames` (carpeta local: `mate`)

---

## 1. Visión

**consolegames** es una colección de juegos de terminal con identidad criolla
argentina y el humor como sello. Cada juego comparte el mismo "chasis" visual —
una ventana `XXXX.EXE` / `.SIM` con un lienzo pixel-art y una barra de estado
inferior con el formato fijo `<juego> · <métrica> · <día> · next: <lo que sigue>`.

**MATE.EXE** es el primer juego: un juego **entre narrativo y de habilidad**
sobre el ritual del mate. El jugador arma su equipo (que dispara humor y afecta
el puntaje), calienta el agua al punto justo y ceba sin quemar ni lavar la yerba.

La colección corre primero en el navegador (terminal emulada con xterm.js,
deployable a GitHub Pages sobre `consolegames.com.ar`) y, más adelante, como CLI
en la terminal real. El proyecto es **open source** desde el arranque.

---

## 2. Alcance

### v1 (esta entrega)
- **MATE.EXE** completo y pulido, con sus **dos modalidades**: **Mate** (caliente)
  y **Tereré** (frío).
- **Host web** únicamente (xterm.js + GitHub Pages / embebible).
- Motor (`engine`), chasis (`shell`) y motor de leyendas (`flavor`) compartidos,
  diseñados para soportar toda la colección.
- Documentación de contribución.

### Roadmap (fuera de v1, cada uno con su propio spec)
- **GAUCHO.EXE** — recorrido/distancia (arrear, pulpería, a caballo con perros).
- **ASADO.SIM** — habilidad/timing (manejar el fuego por corte).
- **TRUCO.EXE** — cartas (revivir el clásico noventoso de PC).
- **FULBITO.EXE** — habilidad (patear penales) + roast de equipos y jugadores.
- **Host CLI** — correr en la terminal real (`consolegames play mate`), reusando
  el mismo motor. Eventualmente carga remota desde registro propio.

---

## 3. Decisiones de arquitectura

| Decisión | Elección | Motivo |
|---|---|---|
| Estructura | **Monorepo liviano** (workspaces npm/pnpm, sin Nx/Turborepo) | Motor compartido entre juegos y hosts; estrellas/identidad concentradas; un solo deploy. |
| Lenguaje | **TypeScript** | Tipos como contrato/documentación; corre en browser y Node. |
| Render | **Motor propio con cell-buffer** + medio-bloque `▀` para pixel-art | Control total del look, cero deps pesadas, corre idéntico en web y CLI. (Descartados Ink y blessed por mal encaje con animación pixel-art y/o browser.) |
| Hosts | **Web primero**; CLI como host futuro | Cero instalación para descubrir; la arquitectura deja el CLI abierto como "otro host". |
| Licencia | **MIT** (propuesta) | Máxima amigabilidad para contribuciones open source. |

---

## 4. Capas y contratos

Cuatro capas, cada una con una única responsabilidad e interfaz clara:

```
packages/engine   → CÓMO se dibuja (loop, cell-buffer, pixel-art, input)
packages/shell    → CÓMO se enmarca y progresa (ventana, HUD, progresión, runtime)
packages/flavor   → el humor y los rangos (motor de leyendas data-driven)
games/<juego>     → QUÉ se juega (implementa la interfaz Game + sus datos)
apps/web          → arranque sobre xterm.js + deploy
```

**Dependencias:** `engine` no depende de nada; `shell` y `flavor` dependen de
`engine`/tipos comunes; un juego depende de `shell` + `flavor` + `engine`;
`web` arranca un `Game` dentro del `runtime`.

### 4.1 `engine`
- **`cell-buffer.ts`** — grilla de celdas `{ char, fg, bg }` con **doble buffer**;
  hace diff y escribe a la salida solo las celdas que cambiaron (sin parpadeo).
- **`pixel-canvas.ts`** — API de píxeles sobre la grilla usando el medio-bloque
  `▀` (color de frente = píxel superior, color de fondo = píxel inferior →
  doble resolución vertical). API: `setPixel`, `fillRect`, `drawSprite`.
- **`loop.ts`** — game loop con **tick fijo** (~30 fps, acumulador); `update(dt)`
  determinista, render después.
- **`input.ts`** — eventos de teclado normalizados en un `InputState` por tick
  (teclas apretadas + teclas recién apretadas / edge detection).
- **`index.ts`** — API pública (`createLoop`, `CellBuffer`, `PixelCanvas`, `InputState`).

El motor no sabe de mate, ventana ni HUD: solo píxeles, ticks y teclas.

### 4.2 `shell`
- **`window-frame.ts`** — marco con barra de título `XXXX.EXE` + botones `─ □ ✕`;
  entrega al juego el rectángulo interior como lienzo.
- **`hud.ts`** — barra inferior `<juego> · <métrica> · <día> · next: <…>` + barra
  de progreso. Sin lógica: pinta un `HudState`.
- **`progression.ts`** — días, métrica acumulada y progreso. Persiste vía interfaz
  `Storage` enchufable (web = `localStorage`; CLI futuro = archivo).
- **`game.ts`** — la interfaz `Game` (ver 4.4).
- **`runtime.ts`** — orquesta: corre `update`, compone lienzo + frame + HUD, manda
  al motor, enruta input. Cambiar de juego = pasarle otro `Game`.

### 4.3 `flavor` (ver Sección 6)

### 4.4 El contrato `Game`
```ts
interface Game {
  meta: { id: string; title: string; metricLabel: string }
  update(dt: number, input: InputState): void   // lógica por tick
  render(canvas: PixelCanvas): void              // dibuja SOLO su lienzo interior
  hud(): HudState                                // métrica actual + progreso
}
```
Un juego nuevo (gaucho, asado, truco, fulbito) = implementar esta interfaz y sus
datos. Motor y chasis no se tocan.

---

## 5. MATE.EXE — diseño del juego

### 5.1 Modalidades (no progresivas — el jugador elige)
- **Modo Mate** (caliente): calentar agua al punto, cebar.
- **Modo Tereré** (frío): agua fría/hielo, en vaso, con fruta o jugo en sobre.

Comparten mecánica y motor; difieren en el **target de temperatura**
(caliente vs. frío), los **ingredientes/elecciones** y las **tablas de contenido**.

### 5.2 Niveles = contexto social (dentro de un modo)
- **Solo** — cebás para vos; foco en tu técnica/puntaje.
- **Con compañía / ronda** — cebás para varios; cada invitado espera lo suyo.
  Restricción: no darle el mate frío, ni quemado, ni lavado a nadie.

### 5.3 Flujo de pantallas
1. **Selección (wizard tipo carrusel, una pantalla por elección):**
   - **1a — Mate/vaso:** imagen pixel-art + nombre + descripción; `← →` navega,
     `ENTER` confirma. Cada opción muestra su **leyenda** asociada.
   - **1b — Yerba.**
   - **1c — Compañía:** solo / ronda de 3 / con compañía (íconos de personas).
   - *(Tereré agrega elecciones: fruta o jugo en sobre.)*
2. **Calentar (timing, tiempo real):** la temperatura sube sola (barra); imagen de
   la pava. `[ESPACIO]` al entrar en la zona objetivo (caliente ~75–82° para mate;
   zona fría para tereré). Pasarse = quemar/arruinar.
3. **Cebar (timing, tiempo real):** `[ESPACIO]` sostenido vierte agua; soltar antes
   de inundar. Mucha agua = **lavado**; poca = no toma; punto justo = óptimo.
4. **Resultado:** nota (perfecto / quemado / lavado) + desglose de puntaje +
   **leyenda** de premio/burla + **rango** actual.

### 5.4 Modelo de puntaje
```
puntaje_cebada = Σ(modificadores de elección) + calidad_temperatura + calidad_cebado
```
- Las **elecciones** suman o restan (saber qué elegir es skill, no solo chiste).
- `calidad_temperatura` y `calidad_cebado` salen de qué tan cerca del punto justo
  estuvo el timing.
- El puntaje define la **nota** (con su leyenda), suma a la **métrica del día** y
  actualiza el **rango**.
- **Sin game over**: el objetivo es tu mejor mate; cebar mal baja el puntaje y
  dispara la burla.

---

## 6. Contenido data-driven: `flavor`

Paquete compartido (todos los juegos roastean). El **motor** es código; el
**humor es dato**. Tres tablas, por juego:

**1) Elecciones (`choices`)** — equipo/ingredientes con puntaje y leyenda:
```ts
{ id:"jugo-en-sobre", nombre:"Jugo en sobre", sprite:"jugo",
  modos:["terere"], puntaje:-20,
  leyendas:["Jugo en sobre. Valió. -20 y la vergüenza."] }
```

**2) Leyendas por resultado/combo (`lines`)** — disparan por contexto:
```ts
{ texto:"Ni Maradona cebaba así.", cuando:{ resultado:"perfecto" }, peso:3 }
```
El motor filtra las que matchean y **elige una al azar por peso** (rejugabilidad).

**3) Rangos (`ranks`)** — umbral de puntaje → título cómico regional:
```ts
[{ min:0, titulo:"Porteño bronce" }, { min:500, titulo:"Entrerriano oro" },
 /* … */, { min:5000, titulo:"Cacique guaraní" }]
```

**API:** `flavor.line(context)` recibe `{ modo, elecciones, resultado, puntaje, rango }`
y devuelve la frase apropiada. Los juegos **nunca** hardcodean frases.

Mate trae sus tablas en `games/mate/content/`. Agregar tereré o un juego nuevo =
**escribir datos, no mecánica**.

### 6.1 Sprites como datos
Cada sprite es un **mapa de píxeles + paleta** que el motor carga al renderizar
(igual que el `data-sprite` de los mockups). Ajustar forma/colores = editar el
dato del sprite, sin tocar lógica ni motor.

---

## 7. Host web

- **`apps/web`** monta el `runtime` sobre **xterm.js**: teclado del navegador →
  `input` del motor; salida del motor → xterm.
- Deploy estático a **GitHub Pages** con dominio `consolegames.com.ar` (CNAME).
- Empaquetable también como **componente embebible** (`<iframe>`/widget) para
  meter en otras páginas.
- Sin backend: persistencia en `localStorage`.

---

## 8. Documentación y contribución (parte de v1)

- **`README.md`** — vitrina: qué es, GIF, cómo jugar (web) y cómo correr local.
- **`ARCHITECTURE.md`** — las capas, el contrato `Game`, el flujo de datos.
- **`CONTRIBUTING.md`** — setup, convenciones y **checklist de PR**.
- **`docs/agregar-un-juego.md`** — receta para implementar `Game` + registrar.
- **`docs/escribir-contenido.md`** — cómo editar las tablas de leyendas/elecciones/rangos.
- **Tipos TS + JSDoc** como documentación viva: el esquema no se puede "romper"
  sin que el compilador avise.
- **`.github/`** con plantilla de PR e issue templates.
- **`games/_template/`** — juego mínimo de ejemplo para copiar.
- **`LICENSE`** — MIT.

La sostenibilidad viene de **fronteras claras + tipos estrictos + contenido como
datos**: un PR típico toca una sola cosa entendible (una tabla, un sprite, un
juego que implementa una interfaz conocida) → fácil de revisar y aprobar.

---

## 9. Testing

- **`engine`**: tests unitarios deterministas — dado un input y un estado, se
  verifica el cell-buffer resultante (sin render real).
- **`shell`**: composición frame + HUD a partir de un `HudState`/`Game` fake.
- **`flavor`**: el filtrado/selección por contexto es determinista con semilla
  fija (se verifica qué leyenda matchea cada contexto).
- **MATE.EXE**: la física de temperatura y de llenado es determinista (tick fijo),
  así que se testea el puntaje resultante para secuencias de input conocidas.

---

## 10. Decisiones pendientes (contenido, no bloquean implementación)

- **Lista completa de rangos** (los títulos regionales cómicos, de "Porteño bronce"
  a "Cacique guaraní", incluyendo variantes uruguayas, etc.).
- **Tablas de leyendas** de mate y tereré (elecciones + resultados).
- **Set de elecciones** definitivo de cada modo (mates/vasos, yerbas, frutas/jugos).
- **Qué muestra el `next:` del HUD** cuando no hay progresión entre modos
  (candidatos: próximo rango / nivel social / cierre del día). A resolver al
  implementar el HUD.

---

## 11. Estructura de carpetas

```
mate/                          (repo: consolegames)
├── packages/
│   ├── engine/
│   ├── shell/
│   └── flavor/
├── games/
│   ├── mate/
│   │   ├── src/
│   │   └── content/           (choices, lines, ranks, sprites)
│   └── _template/
├── apps/
│   └── web/
├── docs/
│   ├── agregar-un-juego.md
│   └── escribir-contenido.md
├── .github/
├── ARCHITECTURE.md
├── CONTRIBUTING.md
├── README.md
├── LICENSE
└── package.json               (workspaces)
```
