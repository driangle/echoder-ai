# Errors

Failure text → cause → fix. The last section is the dangerous one: sketches that
exit 0 and are still wrong.

## Loud failures

### `vis is not defined` (or `shader`, `shape3d`, `image`, `video`, `screen`, `compose`)

The GL namespaces do not exist in the Node-native 2D runtime, and
`--renderer=auto` resolves to 2D for `echoder run`.

Add `--renderer=browser` to both the `run` and the `render`.

### `The browser renderer needs Chromium. Install it with: echoder install-browser`

The headless browser is a one-time ~150 MB download that does not ship with the
CLI. Run `echoder install-browser` (add `--with-deps` on Linux) and retry — it
works the same from a Homebrew install and from a source checkout. If you cannot
run it, relay the command and say the GL sketch is **unverified** until the user
does. Do not fall back to the 2D renderer and report the sketch as checked.

Use exactly that command, not `npx playwright install chromium`: the CLI pins its
own Playwright and launches only the matching Chromium revision.

### `Import Error: Cannot find module './lib.js'`

A loose `.js` outside any project is a project of one file, so its siblings are
not loaded and no `import` can resolve. Give the directory an `echoder.yaml`
(`echoder init`) so the whole folder is one project. Both renderers behave the
same way here — `--renderer=browser` resolves imports exactly as the 2D one
does, so a multi-file GL sketch needs no special handling.

### `Identifier 'fx' has already been declared`

A local shadows a DSL global. Every namespace and colour constant is reserved —
`fx`, `seq`, `signal`, `text`, `system`, `force`, `random`, `log`, `color`,
`math`, `time`, `point`, `xy`, `vec2`, `red`, `gold`, `teal`, `plum`, `tan`, …

Rename the local.

### `X.y is not a function`

The API does not exist. The most common instance is `random.range(a, b)` — use
`math.random(a, b)`; the `random` namespace is `chance`, `oneOf`, `pick`, `seed`,
`seeded`, `stream`.

Check `echoder types path` first, but remember those definitions are
hand-maintained and lag the runtime, so the reverse — "not in the types" — is
*not* evidence a name is missing. Grep the project's own sketches too.

### `Signal.mapTo: this Signal (time.seconds) has no range defined`

`.mapTo()` remaps *from* a Signal's declared range, so it needs one. LFOs, MIDI
values and audio amplitudes carry a range; `time.seconds`, `time.frames` and
plain `.map()` results do not.

Use `.map()`, `.mul()`/`.add()`, or attach one with `.withRange([lo, hi])`.

### `Invalid fft size: 96 is not a power of two`

`spectrum({ bins })` becomes an FFT size. Use 32, 64, 128, 256, 512, 1024, 2048.

### `ENOENT: no such file or directory, open '…/song.wav'`

Media paths resolve **relative to the sketch file**, not the working directory.
Fix the path, or declare the asset in the sketch header's `files:` list.

### `Import Error: 'lodash' is not a project-relative path`

There are no external packages. Imports must be `./x.js`, `../x.js` or `/x.js`
(project root, not filesystem root).

### `Import Error: Cannot find module './lib/nope.js'`

The path is wrong — the message suggests the nearest real file. Note that
renaming a file never rewrites the specifiers pointing at it.

### `no sketch files matched: <dir>`

`echoder check` only collects **runnable** files: the manifest's `main`, files
with a `kind: sketch` header comment, and each subdirectory's `main.js`. A
directory of headerless `.js` files yields nothing.

Add headers, or name the files explicitly (`echoder run a.js b.js`) — an
explicitly named file is always taken at its word.

### `(regl) Error compiling fragment shader`

A GLSL syntax or type error. The reported line number points into the bundled
harness, **not** your shader, so it tells you nothing. Bisect: replace the body
with `gl_FragColor = vec4(v_uv, 0.0, 1.0);` and add your code back in pieces.
Usual suspects: a missing semicolon, `float`/`int` mixing (`2` vs `2.0`), and a
`for` loop bound that is not a compile-time constant.

### `⏭️ <name> — requires unavailable namespace(s): shader`

Not a failure — `echoder check` skipping a sketch it cannot run in this mode.
Re-run with `--renderer=browser` to actually verify it. A skip is *not* a pass;
say so if you leave one unresolved.

## Silent failures — exit 0, still wrong

These are why step 4 of the procedure exists.

### The frame is blank

- **An audio-reactive sketch with no idle floor.** Visual runs wire `audio` to
  an unrendered context, so `song.amplitude` and `.spectrum()` sit at zero and
  every bar has zero height. Add an LFO floor (recipe 3).
- **An input-driven sketch with no `.default()`.** MIDI, mouse and keyboard idle
  at rest headlessly. Give every input signal a `.default()`.
- **Geometry off-canvas.** Origin is top-left, Y increases *downward*, canvas is
  800×600 by default. A y computed from a bottom-up assumption lands outside.

### The shape is a single pixel

An `lfo.*` bound straight to a radius or a size. Raw LFO range is `[-1, 1]`, so
the value is sub-pixel and negative half the time — and nothing errors.
`.mapTo(lo, hi)` every LFO before it reaches a dimension.

### The audio file is silent

`echoder render <sketch> -o out.wav` prints `peak <n>` and exits **0 even at
`peak 0.000`**. Read the number. A zero peak usually means nothing was routed to
`audio.device()`, or a source was created and never connected.

### A `.loop()` / `.trim()` / `.speed()` / `.shift()` had no effect

These are immutable — they return a **new** node. `song.loop({ start: '16m' })`
on its own line does nothing; assign the result and use that.

### Frames were written but the command failed

`echoder render` can write frames *and* exit non-zero (`❌ … 1 error(s); wrote 8
frames`). A frames directory is not evidence of success — read the exit line.
