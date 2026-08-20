# Recipes

Ten patterns that cover most of what people ask Echoder for. **Every snippet
below was run with `echoder run` and, where it draws, rendered and looked at.**
Start from the closest one rather than from a blank file.

Two of them (2, 3) need an audio file at the path shown — that path is the only
thing to change. The GL recipes (4, 5) need `--renderer=browser`.

---

## 1. LFO-driven shape — the hello world

The reactive graph in four lines: one LFO drives radius, hue and opacity at once.

```js
sketch.bg("#0e1116");

const pulse = lfo.sine({ period: 3 }); // range [-1, 1]

sketch.add(
  shape2d.circle({
    x: sketch.width.div(2),
    y: sketch.height.div(2),
    radius: pulse.mapTo(40, 160),
    fill: color.hsl(pulse.mapTo(20, 55), 70, 60),
    opacity: pulse.mapTo(0.55, 1),
  })
);
```

`.mapTo()` on every use — an LFO is `[-1, 1]` raw.

## 2. Audio-reactive

`song.amplitude` is the source's own RMS, `[0, 1]`. `audio.amplitude` is the
master bus. Neither needs an analyser node.

```js
sketch.bg("#0a0c14");

const song = audio.file({ path: "./song.wav", loop: true });
song.to(audio.device()); // connections are deferred until setup() — this is fine here

const level = song.amplitude;

sketch.add(
  shape2d.circle({
    x: sketch.width.div(2),
    y: sketch.height.div(2),
    radius: level.mapTo(60, 260),
    fill: color.hsl(level.mapTo(190, 320), 75, 60),
  })
);
```

Headless this renders at `level = 0` — the smallest, coolest circle. That is the
sketch's idle state, not a bug, and it is all you can verify.

## 3. Spectrum bars with an idle floor

`bins` becomes an FFT size, so it **must be a power of two** (`64`, not `48`).
`.mapTo()` on a spectrum yields a `Signal<Float32Array>`; index it with `.map()`.

```js
sketch.bg("#080a10");

const song = audio.file({ path: "./song.wav", loop: true });
song.to(audio.device());

const BARS = 64;
const W = sketch.width.value ?? 800;
const H = sketch.height.value ?? 600;
const bw = W / BARS;

// The idle floor: without it every bar is zero-height when nothing is playing,
// and the frame is black.
const idle = lfo.sine({ period: 6 });
const bins = song.spectrum({ bins: BARS }).mapTo(0, H * 0.8);

for (let i = 0; i < BARS; i++) {
  const floor = idle.mapTo(18, 54).mul(0.6 + 0.4 * Math.sin(i * 0.4));
  const h = bins.map((v) => (v ? v[i] : 0)).add(floor);
  sketch.add(
    shape2d.rect({
      x: i * bw,
      y: h.map((v) => H - (v ?? 0)),
      width: bw - 2,
      height: h,
      fill: color.hsl(196 + i * 1.7, 72, 58),
    })
  );
}
```

## 4. Buffer feedback — trails (GL)

`vis.buffer()` reads as the *previous* frame. Route a chain that reads it back
into it and you have trails with no history bookkeeping. Decay (`brightness`
under 1) is what stops it saturating to white.

```js
const trail = vis.buffer();
const content = vis.noise.simplex({ scale: 2.5, speed: 0.4 });

trail
  .scale(1.02) // slight zoom grows trails outward
  .rotate(lfo.sine({ period: 24 }).mapTo(-3, 3))
  .effect("brightness", { amount: 0.94 }) // decay — below 1.0 or it blows out
  .layer(content, { blend: "add", opacity: 0.35 })
  .to(trail);

sketch.add(trail);
```

Verify with `echoder run trails.js --duration=2 -v --renderer=browser`.

## 5. Fragment shader field (GL)

`shader.fragment` is one fullscreen pass per frame — closed-form visuals. Built-in
uniforms: `resolution` (vec2), `time` (float seconds), `v_uv` (varying). User
uniforms accept numbers or `Signal<number>`.

```js
const drift = lfo.sine({ period: 18 }).mapTo(0, 6.2831853);

const field = shader.fragment({
  frag: `
    precision highp float;
    uniform vec2 resolution;
    uniform float time;
    uniform float drift;
    varying vec2 v_uv;

    vec3 palette(float t) {
      vec3 c = vec3(0.055, 0.063, 0.098);
      c = mix(c, vec3(0.180, 0.145, 0.360), smoothstep(0.00, 0.35, t));
      c = mix(c, vec3(0.545, 0.400, 0.980), smoothstep(0.30, 0.70, t));
      c = mix(c, vec3(0.960, 0.700, 0.290), smoothstep(0.75, 1.00, t));
      return c;
    }

    void main() {
      vec2 p = (v_uv - 0.5) * vec2(resolution.x / resolution.y, 1.0);
      float a = sin(p.x * 6.0 + drift) + sin(p.y * 5.0 - drift * 0.7);
      float b = sin(length(p) * 12.0 - time * 0.9);
      float t = clamp((a + b) * 0.25 + 0.5, 0.0, 1.0);
      gl_FragColor = vec4(palette(t), 1.0);
    }
  `,
  uniforms: { drift },
  resolution: [sketch.width.value ?? 800, sketch.height.value ?? 600],
});

sketch.bg("#0b0c12");
sketch.add(field);
```

For an *iterative* simulation (reaction-diffusion, cellular automata, fluid) use
`shader.compute` instead: it ping-pongs framebuffers and hands the previous
frame in as `uniform sampler2D state`, with `stepsPerFrame` and a one-shot
`seed: { frag }` pass for initial conditions.

## 6. Agent system — particles in a noise field

`system.spawn` owns per-agent state; `behavior` returns the next state, `render`
returns a drawable. Signals reach the behavior through `signals:` and arrive
sampled in `ctx.signals`.

```js
sketch.bg("#0a0b12");

const W = sketch.width.value ?? 800;
const H = sketch.height.value ?? 600;
const swirl = lfo.sine({ period: 11 }).mapTo(-0.35, 0.35);

const dust = system.spawn({
  count: 300,
  signals: { swirl },
  init: () => ({
    x: math.random(0, W),
    y: math.random(0, H),
    vx: math.random(-1, 1),
    vy: math.random(-1, 1),
    life: math.random(0.4, 1),
  }),
  behavior: (a, ctx) => {
    const n = math.noise(a.x * 0.003, a.y * 0.003, ctx.time * 0.15) * Math.PI * 4;
    return system.wrap(
      {
        ...a,
        vx: a.vx * 0.94 + Math.cos(n) * 0.5 + ctx.signals.swirl,
        vy: a.vy * 0.94 + Math.sin(n) * 0.5,
        x: a.x + a.vx,
        y: a.y + a.vy,
      },
      ctx.canvas.width,
      ctx.canvas.height
    );
  },
  render: (a) =>
    shape2d.circle({
      x: a.x,
      y: a.y,
      radius: a.life * 2.4,
      fill: color.hsl(210 + a.life * 90, 70, 55 + a.life * 15),
      opacity: 0.35 + a.life * 0.45,
    }),
});

sketch.add(dust);
```

Ranged randomness is `math.random(min, max)` — `random.range` does **not** exist
(the `random` namespace is `chance`, `oneOf`, `pick`, `seed`, `seeded`, `stream`).
`system.wrap`/`system.bounce` handle edges; `force.attract/repel/gravity/drag/
limitSpeed` compose into `system.flock({ forces: [...] })`.

## 7. MIDI-controlled, alive when idle

MIDI accessors return event **Streams**; `.value()` (CC) and `.velocity()`
(notes) cross into frame-sampled Signals. `.default()` supplies the value before
the first message — without it the sketch is blank on a machine with no
controller, which is every headless run.

```js
sketch.bg("#0d0a14");

const breath = lfo.sine({ period: 7 });
const size = midi.in().cc(74).value().mapTo(40, 220).default(90);
const hue = midi.in().cc(71).value().mapTo(180, 340).default(280);

sketch.add(
  shape2d.circle({
    x: sketch.width.div(2),
    y: sketch.height.div(2),
    radius: breath.mapTo(0.9, 1.1).mul(size),
    fill: color.hsl(hue, 70, 60),
    opacity: 0.85,
  })
);

const hit = midi.in().notes(36).velocity().mapTo(0, 1).default(0);
sketch.add(
  shape2d.rect({
    x: 0,
    y: sketch.height.sub(12),
    width: hit.mul(sketch.width),
    height: 12,
    fill: "#f4b942",
  })
);
```

## 8. Sequenced pattern

`seq.euclidean` / `seq.steps` / `seq.random` expose `.value` (velocity),
`.gate`, `.trigger` and `.step` as Signals. Subdivisions follow `time.bpm()`.

```js
time.bpm(124);
sketch.bg("#0b0e14");

const kick = seq.euclidean({ hits: 5, steps: 16, subdivision: "16n" });
const hats = seq.euclidean({ hits: 11, steps: 16, rotation: 2, subdivision: "16n" });

const W = sketch.width.value ?? 800;
const H = sketch.height.value ?? 600;

for (let i = 0; i < 16; i++) {
  const angle = (i / 16) * Math.PI * 2 - Math.PI / 2;
  const on = kick.step.map((s) => (s === i ? 1 : 0.15)); // lights the active step
  sketch.add(
    shape2d.circle({
      x: W / 2 + Math.cos(angle) * 190,
      y: H / 2 + Math.sin(angle) * 190,
      radius: on.mul(22).add(6),
      fill: color.hsl(28 + i * 4, 78, 58),
      opacity: on,
    })
  );
}

sketch.add(
  shape2d.circle({
    x: W / 2,
    y: H / 2,
    radius: kick.value.mapTo(30, 120).default(30),
    fill: "#7a5cf0",
    opacity: 0.7,
  })
);

sketch.add(shape2d.rect({ x: 0, y: H - 10, width: hats.gate.mul(W), height: 10, fill: "#3fc9c0" }));
```

## 9. Pointer-driven with an ambient fallback

Same discipline as MIDI: never *depend* on an input that may never arrive. Blend
it with an LFO orbit so the sketch moves on its own and the pointer steers it.

```js
sketch.bg("#0c1014");

const W = sketch.width.value ?? 800;
const H = sketch.height.value ?? 600;

// NOTE: `fx` is a DSL namespace — a local named `fx` throws at parse time.
const orbitX = lfo.sine({ period: 9 });
const orbitY = lfo.sine({ period: 9, phase: 0.25 });

const px = signal.mix(mouse.x.default(W / 2), orbitX.mapTo(W * 0.25, W * 0.75), 0.5);
const py = signal.mix(mouse.y.default(H / 2), orbitY.mapTo(H * 0.25, H * 0.75), 0.5);

for (let i = 0; i < 7; i++) {
  const lag = 1 - i * 0.12;
  sketch.add(
    shape2d.circle({
      x: px.mul(lag).add((W / 2) * (1 - lag)),
      y: py.mul(lag).add((H / 2) * (1 - lag)),
      radius: 60 - i * 7,
      fill: color.hsl(190 + i * 12, 65, 55),
      opacity: 0.25,
    })
  );
}
```

## 10. Multi-file project

Modules have the same full DSL access the entry does. Imports are
project-relative; there is no bundler and no bare specifiers.

`echoder.yaml`

```yaml
name: Recipe Project
version: 1
kind: sketch
main: main.js
canvas:
  width: 800
  height: 600
  backgroundColor: "#0b0e14"
audio:
  bpm: 120
runtime:
  fps: 60
```

`lib/palette.js` — no header, so it is library code, not a runnable sketch.

```js
export const palette = {
  bg: "#0b0e14",
  ink: "#f2e6d4",
  accent: "#e8b04b",
  cool: "#3fc9c0",
};
```

`lib/ring.js`

```js no-run — a module: its import only resolves inside the project laid out above
import { palette } from "./palette.js";

export function breathingRing({ count = 24, radius = 180, period = 6 } = {}) {
  const breath = lfo.sine({ period });
  const cx = sketch.width.div(2);
  const cy = sketch.height.div(2);

  for (let i = 0; i < count; i++) {
    const a = (i / count) * Math.PI * 2;
    sketch.add(
      shape2d.circle({
        x: breath.mapTo(0.85, 1.15).mul(Math.cos(a) * radius).add(cx),
        y: breath.mapTo(0.85, 1.15).mul(Math.sin(a) * radius).add(cy),
        radius: breath.mapTo(6, 14),
        fill: i % 3 === 0 ? palette.accent : palette.cool,
        opacity: 0.85,
      })
    );
  }
}
```

`main.js`

```js no-run — the entry of the multi-file project above; its imports need its siblings
import { palette } from "./lib/palette.js";
import { breathingRing } from "./lib/ring.js";

sketch.bg(palette.bg);
breathingRing({ count: 28, radius: 190 });

sketch.add(
  text({
    text: "recipes",
    x: sketch.width.div(2),
    y: sketch.height.div(2),
    fontSize: 44,
    textAlign: "center",
    fill: palette.ink,
  })
);
```

To make a second sketch runnable alongside `main.js`, give it a header:

```js
/* ---
name: Shader Field
kind: sketch
tags: [shader]
--- */
```

Then `echoder check <dir>` runs both. Without the header the file is library
code and `check` will not see it.

Both renderers resolve imports, so a multi-file project that needs GL verifies
under `--renderer=browser` like any other.
