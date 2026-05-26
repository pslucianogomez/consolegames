# consolegames — Plan 1: Monorepo + Engine

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Montar el monorepo liviano y el paquete `@consolegames/engine` (cell-buffer, pixel-canvas, input, loop) con tests pasando.

**Architecture:** Monorepo con pnpm workspaces + TypeScript. El `engine` es el núcleo de render: una grilla de celdas con diff, una capa de pixel-art sobre medio-bloque `▀`, manejo de input por tick, y un game loop de tick fijo. No depende de ningún otro paquete y se testea de forma aislada con Vitest.

**Tech Stack:** TypeScript, pnpm workspaces, Vitest.

---

## Plan series (contexto)

Este es el **Plan 1 de 5**. Los siguientes se escriben después, cada uno sobre lo anterior:

1. **Monorepo + Engine** ← este plan
2. Shell (window-frame, HUD, progression, runtime, contrato `Game`)
3. Flavor (motor de leyendas + rangos, data-driven)
4. MATE.EXE (modos mate/tereré, flujo de pantallas, puntaje) + contenido
5. Web host (xterm.js + deploy a GitHub Pages)

Spec de referencia: `docs/superpowers/specs/2026-05-25-mate-exe-design.md`.

---

## File Structure

```
mate/
├── package.json                 # raíz: workspaces + scripts + devDeps
├── pnpm-workspace.yaml          # define los workspaces
├── tsconfig.base.json           # config TS compartida
├── vitest.config.ts             # corre tests de todos los paquetes
└── packages/
    └── engine/
        ├── package.json         # @consolegames/engine
        ├── tsconfig.json        # extiende la base
        └── src/
            ├── cell-buffer.ts   # grilla de celdas + diff
            ├── cell-buffer.test.ts
            ├── pixel-canvas.ts  # pixel-art sobre medio-bloque ▀
            ├── pixel-canvas.test.ts
            ├── input.ts         # InputState por tick
            ├── input.test.ts
            ├── loop.ts          # game loop de tick fijo
            ├── loop.test.ts
            └── index.ts         # API pública (barrel)
```

Responsabilidades: cada archivo `src/*.ts` tiene una sola pieza del motor. Los tests viven al lado del código que prueban.

---

## Task 0: Scaffolding del monorepo

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `tsconfig.base.json`
- Create: `vitest.config.ts`
- Create: `packages/engine/package.json`
- Create: `packages/engine/tsconfig.json`

- [ ] **Step 1: Crear `pnpm-workspace.yaml`**

```yaml
packages:
  - "packages/*"
  - "games/*"
  - "apps/*"
```

- [ ] **Step 2: Crear `package.json` (raíz)**

```json
{
  "name": "consolegames",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc -b"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "vitest": "^2.0.0",
    "@types/node": "^20.0.0"
  }
}
```

- [ ] **Step 3: Crear `tsconfig.base.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "declaration": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "verbatimModuleSyntax": true
  }
}
```

- [ ] **Step 4: Crear `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["packages/**/*.test.ts", "games/**/*.test.ts", "apps/**/*.test.ts"],
  },
});
```

- [ ] **Step 5: Crear `packages/engine/package.json`**

```json
{
  "name": "@consolegames/engine",
  "version": "0.0.0",
  "type": "module",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  }
}
```

- [ ] **Step 6: Crear `packages/engine/tsconfig.json`**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

- [ ] **Step 7: Instalar dependencias**

Run: `pnpm install`
Expected: instala typescript, vitest, @types/node sin errores; crea `pnpm-lock.yaml`.

- [ ] **Step 8: Commit**

```bash
git add package.json pnpm-workspace.yaml tsconfig.base.json vitest.config.ts packages/engine/package.json packages/engine/tsconfig.json pnpm-lock.yaml
git commit -m "chore: scaffolding del monorepo + paquete engine vacío

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 1: CellBuffer (grilla de celdas + diff)

**Files:**
- Create: `packages/engine/src/cell-buffer.ts`
- Test: `packages/engine/src/cell-buffer.test.ts`

- [ ] **Step 1: Escribir el test que falla**

```ts
import { describe, it, expect } from "vitest";
import { CellBuffer } from "./cell-buffer";

describe("CellBuffer", () => {
  it("set/get escribe y lee una celda", () => {
    const buf = new CellBuffer(3, 2);
    buf.set(1, 1, { char: "X", fg: "#fff", bg: "#000" });
    expect(buf.get(1, 1)).toEqual({ char: "X", fg: "#fff", bg: "#000" });
  });

  it("clear deja todas las celdas en espacio con el bg dado", () => {
    const buf = new CellBuffer(2, 1);
    buf.clear("#123456");
    expect(buf.get(0, 0)).toEqual({ char: " ", fg: "#ffffff", bg: "#123456" });
  });

  it("set fuera de rango no rompe", () => {
    const buf = new CellBuffer(2, 2);
    expect(() => buf.set(-1, 5, { char: "Z", fg: "#fff", bg: "#000" })).not.toThrow();
  });

  it("diff devuelve solo las celdas que cambiaron", () => {
    const prev = new CellBuffer(2, 1);
    const next = new CellBuffer(2, 1);
    next.set(1, 0, { char: "A", fg: "#fff", bg: "#000" });
    const changes = next.diff(prev);
    expect(changes).toEqual([{ x: 1, y: 0, cell: { char: "A", fg: "#fff", bg: "#000" } }]);
  });
});
```

- [ ] **Step 2: Correr el test para verificar que falla**

Run: `pnpm vitest run packages/engine/src/cell-buffer.test.ts`
Expected: FAIL — `Cannot find module './cell-buffer'`.

- [ ] **Step 3: Implementación mínima**

```ts
export interface Cell {
  char: string;
  fg: string;
  bg: string;
}

export interface CellChange {
  x: number;
  y: number;
  cell: Cell;
}

const DEFAULT_FG = "#ffffff";
const DEFAULT_BG = "#000000";

export class CellBuffer {
  readonly width: number;
  readonly height: number;
  private cells: Cell[];

  constructor(width: number, height: number) {
    this.width = width;
    this.height = height;
    this.cells = new Array(width * height);
    this.clear();
  }

  private index(x: number, y: number): number {
    return y * this.width + x;
  }

  private inBounds(x: number, y: number): boolean {
    return x >= 0 && y >= 0 && x < this.width && y < this.height;
  }

  clear(bg: string = DEFAULT_BG): void {
    for (let i = 0; i < this.cells.length; i++) {
      this.cells[i] = { char: " ", fg: DEFAULT_FG, bg };
    }
  }

  set(x: number, y: number, cell: Cell): void {
    if (!this.inBounds(x, y)) return;
    this.cells[this.index(x, y)] = cell;
  }

  get(x: number, y: number): Cell {
    const cell = this.cells[this.index(x, y)];
    if (!cell) throw new RangeError(`celda fuera de rango: (${x}, ${y})`);
    return cell;
  }

  diff(prev: CellBuffer): CellChange[] {
    const changes: CellChange[] = [];
    for (let y = 0; y < this.height; y++) {
      for (let x = 0; x < this.width; x++) {
        const a = this.get(x, y);
        const b = prev.get(x, y);
        if (a.char !== b.char || a.fg !== b.fg || a.bg !== b.bg) {
          changes.push({ x, y, cell: a });
        }
      }
    }
    return changes;
  }
}
```

- [ ] **Step 4: Correr el test para verificar que pasa**

Run: `pnpm vitest run packages/engine/src/cell-buffer.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add packages/engine/src/cell-buffer.ts packages/engine/src/cell-buffer.test.ts
git commit -m "feat(engine): CellBuffer con set/get/clear/diff

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: PixelCanvas (pixel-art sobre medio-bloque ▀)

**Files:**
- Create: `packages/engine/src/pixel-canvas.ts`
- Test: `packages/engine/src/pixel-canvas.test.ts`

- [ ] **Step 1: Escribir el test que falla**

```ts
import { describe, it, expect } from "vitest";
import { PixelCanvas } from "./pixel-canvas";

describe("PixelCanvas", () => {
  it("exige altura par", () => {
    expect(() => new PixelCanvas(4, 3)).toThrow();
  });

  it("toCellBuffer mapea píxel superior a fg e inferior a bg con ▀", () => {
    const c = new PixelCanvas(1, 2);
    c.setPixel(0, 0, "#ff0000"); // arriba
    c.setPixel(0, 1, "#00ff00"); // abajo
    const buf = c.toCellBuffer("#000000");
    expect(buf.width).toBe(1);
    expect(buf.height).toBe(1);
    expect(buf.get(0, 0)).toEqual({ char: "▀", fg: "#ff0000", bg: "#00ff00" });
  });

  it("los píxeles vacíos usan el color de fondo dado", () => {
    const c = new PixelCanvas(1, 2);
    const buf = c.toCellBuffer("#222222");
    expect(buf.get(0, 0)).toEqual({ char: "▀", fg: "#222222", bg: "#222222" });
  });

  it("drawSprite copia solo los píxeles no nulos", () => {
    const c = new PixelCanvas(2, 2);
    c.drawSprite(0, 0, { width: 2, height: 1, pixels: ["#abc", null] });
    const buf = c.toCellBuffer("#000000");
    expect(buf.get(0, 0).fg).toBe("#abc");
    expect(buf.get(1, 0).fg).toBe("#000000");
  });

  it("fillRect pinta un bloque", () => {
    const c = new PixelCanvas(2, 2);
    c.fillRect(0, 0, 2, 2, "#999");
    const buf = c.toCellBuffer();
    expect(buf.get(0, 0)).toEqual({ char: "▀", fg: "#999", bg: "#999" });
    expect(buf.get(1, 0)).toEqual({ char: "▀", fg: "#999", bg: "#999" });
  });
});
```

- [ ] **Step 2: Correr el test para verificar que falla**

Run: `pnpm vitest run packages/engine/src/pixel-canvas.test.ts`
Expected: FAIL — `Cannot find module './pixel-canvas'`.

- [ ] **Step 3: Implementación mínima**

```ts
import { CellBuffer } from "./cell-buffer";

export type Color = string;

export interface Sprite {
  width: number;
  height: number;
  pixels: (Color | null)[]; // row-major, longitud = width*height
}

const HALF_BLOCK = "▀";

export class PixelCanvas {
  readonly width: number;
  readonly height: number;
  private px: (Color | null)[];

  constructor(width: number, height: number) {
    if (height % 2 !== 0) {
      throw new Error("PixelCanvas: la altura debe ser par (medio-bloque ▀)");
    }
    this.width = width;
    this.height = height;
    this.px = new Array<Color | null>(width * height).fill(null);
  }

  setPixel(x: number, y: number, color: Color): void {
    if (x < 0 || y < 0 || x >= this.width || y >= this.height) return;
    this.px[y * this.width + x] = color;
  }

  fillRect(x: number, y: number, w: number, h: number, color: Color): void {
    for (let dy = 0; dy < h; dy++) {
      for (let dx = 0; dx < w; dx++) {
        this.setPixel(x + dx, y + dy, color);
      }
    }
  }

  drawSprite(x: number, y: number, sprite: Sprite): void {
    for (let dy = 0; dy < sprite.height; dy++) {
      for (let dx = 0; dx < sprite.width; dx++) {
        const c = sprite.pixels[dy * sprite.width + dx];
        if (c != null) this.setPixel(x + dx, y + dy, c);
      }
    }
  }

  toCellBuffer(bg: Color = "#000000"): CellBuffer {
    const rows = this.height / 2;
    const buf = new CellBuffer(this.width, rows);
    for (let cy = 0; cy < rows; cy++) {
      for (let x = 0; x < this.width; x++) {
        const top = this.px[cy * 2 * this.width + x] ?? bg;
        const bot = this.px[(cy * 2 + 1) * this.width + x] ?? bg;
        buf.set(x, cy, { char: HALF_BLOCK, fg: top, bg: bot });
      }
    }
    return buf;
  }
}
```

- [ ] **Step 4: Correr el test para verificar que pasa**

Run: `pnpm vitest run packages/engine/src/pixel-canvas.test.ts`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add packages/engine/src/pixel-canvas.ts packages/engine/src/pixel-canvas.test.ts
git commit -m "feat(engine): PixelCanvas con medio-bloque, sprites y fillRect

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: InputState (teclado por tick con edge detection)

**Files:**
- Create: `packages/engine/src/input.ts`
- Test: `packages/engine/src/input.test.ts`

- [ ] **Step 1: Escribir el test que falla**

```ts
import { describe, it, expect } from "vitest";
import { InputState } from "./input";

describe("InputState", () => {
  it("keyDown marca la tecla como apretada y recién apretada", () => {
    const input = new InputState();
    input.keyDown("Space");
    expect(input.isDown("Space")).toBe(true);
    expect(input.justPressed("Space")).toBe(true);
  });

  it("endTick limpia el edge pero mantiene la tecla apretada", () => {
    const input = new InputState();
    input.keyDown("Space");
    input.endTick();
    expect(input.justPressed("Space")).toBe(false);
    expect(input.isDown("Space")).toBe(true);
  });

  it("keyUp suelta la tecla", () => {
    const input = new InputState();
    input.keyDown("ArrowLeft");
    input.endTick();
    input.keyUp("ArrowLeft");
    expect(input.isDown("ArrowLeft")).toBe(false);
  });

  it("repetir keyDown sin soltar no vuelve a marcar justPressed tras endTick", () => {
    const input = new InputState();
    input.keyDown("Space");
    input.endTick();
    input.keyDown("Space"); // auto-repeat del SO
    expect(input.justPressed("Space")).toBe(false);
  });
});
```

- [ ] **Step 2: Correr el test para verificar que falla**

Run: `pnpm vitest run packages/engine/src/input.test.ts`
Expected: FAIL — `Cannot find module './input'`.

- [ ] **Step 3: Implementación mínima**

```ts
export class InputState {
  private down = new Set<string>();
  private pressed = new Set<string>();

  keyDown(key: string): void {
    if (!this.down.has(key)) {
      this.pressed.add(key);
      this.down.add(key);
    }
  }

  keyUp(key: string): void {
    this.down.delete(key);
  }

  isDown(key: string): boolean {
    return this.down.has(key);
  }

  justPressed(key: string): boolean {
    return this.pressed.has(key);
  }

  /** Llamar al FINAL de cada tick para limpiar los edges. */
  endTick(): void {
    this.pressed.clear();
  }
}
```

- [ ] **Step 4: Correr el test para verificar que pasa**

Run: `pnpm vitest run packages/engine/src/input.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add packages/engine/src/input.ts packages/engine/src/input.test.ts
git commit -m "feat(engine): InputState con edge detection por tick

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: createLoop (game loop de tick fijo)

**Files:**
- Create: `packages/engine/src/loop.ts`
- Test: `packages/engine/src/loop.test.ts`

- [ ] **Step 1: Escribir el test que falla**

```ts
import { describe, it, expect } from "vitest";
import { createLoop } from "./loop";

describe("createLoop", () => {
  it("avanza exactamente un update por paso a fps=100 (step=10ms)", () => {
    let updates = 0;
    let renders = 0;
    const loop = createLoop(
      { update: () => updates++, render: () => renders++ },
      100,
    );
    loop.advance(10);
    expect(updates).toBe(1);
    expect(renders).toBe(1);
    expect(loop.tickCount).toBe(1);
  });

  it("acumula el tiempo sobrante entre advances", () => {
    let updates = 0;
    const loop = createLoop({ update: () => updates++, render: () => {} }, 100);
    loop.advance(25); // 2 updates, 5ms sobran
    expect(updates).toBe(2);
    loop.advance(5); // 5+5 = 10ms -> 1 update más
    expect(updates).toBe(3);
  });

  it("dt se pasa en segundos", () => {
    const dts: number[] = [];
    const loop = createLoop({ update: (dt) => dts.push(dt), render: () => {} }, 100);
    loop.advance(10);
    expect(dts).toEqual([0.01]);
  });

  it("render se llama una vez por advance aunque no haya updates", () => {
    let renders = 0;
    const loop = createLoop({ update: () => {}, render: () => renders++ }, 100);
    loop.advance(3); // menos que un step
    expect(renders).toBe(1);
  });
});
```

- [ ] **Step 2: Correr el test para verificar que falla**

Run: `pnpm vitest run packages/engine/src/loop.test.ts`
Expected: FAIL — `Cannot find module './loop'`.

- [ ] **Step 3: Implementación mínima**

```ts
export interface LoopCallbacks {
  update(dt: number): void; // dt en segundos
  render(): void;
}

export interface Loop {
  /** Avanza el loop por el tiempo transcurrido (ms). Drive manual: rAF o tests. */
  advance(elapsedMs: number): void;
  readonly tickCount: number;
}

export function createLoop(callbacks: LoopCallbacks, fps = 30): Loop {
  const step = 1000 / fps;
  let acc = 0;
  let ticks = 0;

  return {
    advance(elapsedMs: number): void {
      acc += elapsedMs;
      while (acc >= step) {
        callbacks.update(step / 1000);
        acc -= step;
        ticks++;
      }
      callbacks.render();
    },
    get tickCount(): number {
      return ticks;
    },
  };
}
```

- [ ] **Step 4: Correr el test para verificar que pasa**

Run: `pnpm vitest run packages/engine/src/loop.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add packages/engine/src/loop.ts packages/engine/src/loop.test.ts
git commit -m "feat(engine): createLoop con tick fijo y acumulador

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: API pública (barrel) + verificación final

**Files:**
- Create: `packages/engine/src/index.ts`

- [ ] **Step 1: Crear el barrel `index.ts`**

```ts
export { CellBuffer } from "./cell-buffer";
export type { Cell, CellChange } from "./cell-buffer";

export { PixelCanvas } from "./pixel-canvas";
export type { Color, Sprite } from "./pixel-canvas";

export { InputState } from "./input";

export { createLoop } from "./loop";
export type { Loop, LoopCallbacks } from "./loop";
```

- [ ] **Step 2: Typecheck del paquete**

Run: `pnpm typecheck`
Expected: sin errores de tipos.

- [ ] **Step 3: Correr toda la suite**

Run: `pnpm test`
Expected: PASS — los 17 tests (4 cell-buffer + 5 pixel-canvas + 4 input + 4 loop) en verde.

- [ ] **Step 4: Commit**

```bash
git add packages/engine/src/index.ts
git commit -m "feat(engine): API pública del motor (barrel)

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Self-review (cobertura del spec)

- **Spec §4.1 `engine`** — cubierto: `cell-buffer` (Task 1), `pixel-canvas` con medio-bloque (Task 2), `input` (Task 3), `loop` tick fijo (Task 4), `index` (Task 5). ✅
- **Spec §3 monorepo liviano (pnpm workspaces, TS)** — Task 0. ✅
- **Spec §9 testing del engine** (input + estado → buffer determinista) — los tests de cada task lo cubren. ✅
- Fuera de alcance de este plan (planes 2-5): `shell`, `flavor`, `games/mate`, `apps/web`. La salida ANSI real a terminal (escribir el diff a xterm) vive en el host web (Plan 5); el engine entrega `CellBuffer`/`diff` y no asume destino.
- Sin placeholders; tipos consistentes entre tasks (`Cell`, `Sprite`, `Color`, `Loop`, `LoopCallbacks` definidos antes de usarse; `endTick` usado igual en input).
