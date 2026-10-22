---
name: dev-claude-reel
description: Make a ~50 s vertical (1080x1920) 3D reel where a developer and a "Claude" character have a short, funny conversation. Handles the script, ElevenLabs voices, a three.js scene rendered frame by frame in headless Chrome, camera and cuts, the audio mix and the final mp4, plus a QA pass that catches glitchy motion before the full render.
---

# dev-claude-reel

**Output:** one mp4, 1080x1920, 30 fps, H.264 + AAC, about 50 seconds. A low-poly 3D scene, two characters (a developer and a Claude character), real voices, burned-in subtitles, a camera that cuts on the dialogue, and a clean mix.

**How:** a three.js page is driven by a single `setT(t)` function. Headless Chrome renders it one frame at a time, ffmpeg glues frames and audio together.

**What this skill does not decide:** what the characters look like, where they are, and what they talk about. Take that from the user if they gave it. Otherwise pick something in a few lines and get going.

**Pipeline (in this order, no skipping):**
brief → script → voices → timeline → scene → stills → motion check → mix → full render → mux → verify → deliver

---

## 1. Ground rules

These override convenience. They don't override the user.

- **Don't hurt the machine.** It's the user's daily computer. One headless browser, GPU rendering, lowest process priority, no parallel workers. If WebGL reports a software renderer (SwiftShader / "software" / "Basic Render"), stop and tell the user. Never force software GL.
- **Secrets stay secret.** The ElevenLabs key lives in `ELEVENLABS_API_KEY` (env or `.env`). Read it inside scripts only. Never print, log or write it anywhere visible.
- **Stills first, film second.** No full render until a contact sheet of stills looks right. A full re-render for a one-shot mistake is the most expensive error in this workflow.
- **Render in foreground chunks.** Background jobs die when the turn ends. Keep each render call under ~8 minutes and make it resumable (section 11).
- **One video per chat.** The scene file grows fast. Split it into parts, don't re-read it for no reason, start a fresh chat for the next reel.
- **Deliver a file, never post it.** Publishing is the user's call.
- **Be honest about gaps.** Missing key, ffmpeg or Chrome? Say exactly what's missing and offer the closest workable option. Never fake a result.

## 2. Setup check (first minute)

| Need | Notes |
|---|---|
| Node 22+ | Uses built-in `fetch` and `WebSocket`. Only npm dependency: three.js. |
| Chrome or Edge | Run headless. |
| ffmpeg + ffprobe | On PATH or a known location. |
| three.js | `npm i three@0.170`, served from disk by a local static server. No CDN, no network during render. |
| ElevenLabs key | Free tier works (~10k credits/month, API access, premade voices, no commercial licence). ~1 credit per character, sound generation costs extra. |
| GPU | Probe once (section 3). |

**ElevenLabs budget:** finish the script before generating anything, count its characters, only make alternate takes for key lines, and cache every result so nothing is paid for twice.

**Model:** `eleven_v3`. If the account can't use it, fall back to `eleven_multilingual_v2`, strip the `[audio tags]` from the text, and warn the user the acting will be flatter.

**No key?** Say so. Build script, scene and a silent test render as far as useful, then ask for the key. If the user insists on going without: OS text-to-speech, word timings estimated from character counts. Tell them lip sync and subtitles will be rougher.

### Folder layout

One folder per video, plain files, no build step.

```
reel/
  package.json      three
  film.html         canvas + subtitle overlay + import map, loads scene.js
  scene.js          3D world + setT(t)  (split into parts when it gets big)
  timeline.mjs      timing + shots, imported by Node AND the browser
  gen-audio.mjs     takes.json -> ElevenLabs voices + ambience (cached on disk)
  words.mjs         word timings from alignment json
  build-film.mjs    per-speaker voice buses + film.json (envelopes, subtitles)
  music.mjs         optional music bed
  mix.mjs           final mix
  cdp.mjs           headless Chrome driver
  render.mjs        stills | full (chunked, resumable)
  check.mjs         automated QA
  make.mjs          runs gen-audio -> build-film -> music -> mix
  vo/ sfx/ frames/ stills/
```

**Any change to `timeline.mjs` means rerunning `make.mjs` end to end.** Stale audio stems are the classic invisible bug: a sound that belongs to an old timing shows up as a random click.

---

## 3. Driving headless Chrome (`cdp.mjs`)

A tiny Chrome DevTools Protocol client, no dependencies.

**Launch flags:**
`--headless=new --remote-debugging-port=<p> --user-data-dir=<tmp> --no-first-run --disable-extensions --disable-background-timer-throttling --disable-renderer-backgrounding --mute-audio --window-size=W,H --use-gl=angle --use-angle=<backend> --ignore-gpu-blocklist --enable-gpu`
Backend: `d3d11` on Windows, `metal` on macOS, `gl` on Linux.

**After connecting:**
- `Emulation.setDeviceMetricsOverride { width: 1080, height: 1920, deviceScaleFactor: 1, mobile: false }`
- **GPU probe:** read `UNMASKED_RENDERER_WEBGL` via `WEBGL_debug_renderer_info`. Empty, or matching `/swiftshader|software|basic render/i` → abort and tell the user.
- **Stay polite:** `os.setPriority(os.constants.priority.PRIORITY_LOW)` at the top of every Node script. Every ~5 s, lower the browser and every child process again (Chrome bumps its GPU process back up). Get child pids from `SystemInfo.getProcessInfo` on the browser websocket (`/json/version` → `webSocketDebuggerUrl`). On Windows, also pin the browser to one logical core via PowerShell (`(Get-Process -Id <pid>).ProcessorAffinity = <single bit>`).

**API surface:** `goto(url)`, `eval(expr)` (awaitPromise + returnByValue, throws on `exceptionDetails`), `waitFor(expr, timeout)`, `send(method, params)`, a `logs` array (console calls, exceptions, log entries), `close()` (kill process, delete temp profile).

Starting point (adapt it, don't paste blindly):

```js
import { spawn } from 'node:child_process';
import fs from 'node:fs'; import os from 'node:os'; import path from 'node:path';

const BROWSERS = [
  'C:/Program Files/Google/Chrome/Application/chrome.exe',
  'C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe',
  '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome',
  '/usr/bin/google-chrome',
];
const BIN = BROWSERS.find(p => fs.existsSync(p));
const ANGLE = { win32: 'd3d11', darwin: 'metal' }[process.platform] ?? 'gl';
const sleep = ms => new Promise(r => setTimeout(r, ms));

export async function launch({ width = 1080, height = 1920, port = 9450 } = {}) {
  try { os.setPriority(os.constants.priority.PRIORITY_LOW); } catch {}
  const profile = fs.mkdtempSync(path.join(os.tmpdir(), 'reel-'));
  const proc = spawn(BIN, [
    `--remote-debugging-port=${port}`, `--user-data-dir=${profile}`, '--headless=new',
    '--no-first-run', '--disable-extensions', '--disable-background-timer-throttling',
    '--disable-renderer-backgrounding', '--mute-audio', `--window-size=${width},${height}`,
    '--use-gl=angle', `--use-angle=${ANGLE}`, '--ignore-gpu-blocklist', '--enable-gpu', 'about:blank',
  ], { stdio: 'ignore' });

  let targets;
  for (let i = 0; i < 100 && !targets?.length; i++) {
    try { targets = await (await fetch(`http://127.0.0.1:${port}/json/list`)).json(); }
    catch { await sleep(150); }
  }
  const ws = new WebSocket(targets.find(t => t.type === 'page').webSocketDebuggerUrl);
  await new Promise((ok, fail) => { ws.onopen = ok; ws.onerror = fail; });

  let seq = 0; const waiting = new Map(); const logs = [];
  ws.onmessage = ({ data }) => {
    const msg = JSON.parse(data);
    if (msg.id && waiting.has(msg.id)) {
      const { ok, fail } = waiting.get(msg.id); waiting.delete(msg.id);
      msg.error ? fail(new Error(msg.error.message)) : ok(msg.result);
    } else if (msg.method === 'Runtime.exceptionThrown') {
      const d = msg.params.exceptionDetails;
      logs.push({ type: 'exception', text: d.exception?.description || d.text });
    } else if (msg.method === 'Runtime.consoleAPICalled') {
      logs.push({ type: msg.params.type, text: msg.params.args.map(a => a.value ?? a.description).join(' ') });
    }
  };
  const send = (method, params = {}) => new Promise((ok, fail) => {
    const id = ++seq; waiting.set(id, { ok, fail });
    ws.send(JSON.stringify({ id, method, params }));
  });

  await send('Runtime.enable'); await send('Page.enable'); await send('Log.enable');
  await send('Emulation.setDeviceMetricsOverride', { width, height, deviceScaleFactor: 1, mobile: false });
  // TODO: GPU probe, 5 s priority-lowering timer for proc.pid + children, Windows affinity pin

  const api = {
    send, logs, proc,
    async eval(expression) {
      const r = await send('Runtime.evaluate', { expression, returnByValue: true, awaitPromise: true });
      if (r.exceptionDetails) throw new Error('eval failed: ' + (r.exceptionDetails.exception?.description || r.exceptionDetails.text));
      return r.result.value;
    },
    async goto(url) { await send('Page.navigate', { url }); await api.waitFor('document.readyState === "complete"', 15000); },
    async waitFor(expr, ms = 10000) {
      for (const t0 = Date.now(); Date.now() - t0 < ms; await sleep(100)) {
        try { if (await api.eval(expr)) return true; } catch {}
      }
      throw new Error('timeout: ' + expr);
    },
    async close() {
      try { ws.close(); } catch {} try { proc.kill(); } catch {}
      setTimeout(() => fs.rmSync(profile, { recursive: true, force: true }), 500);
    },
  };
  return api;
}
```

---

## 4. The film page (`film.html` + `scene.js`)

### The golden rule

**`window.setT(t)` is a pure function of time.** Give it seconds, it poses every character, prop, light and the camera, renders the frame, done. That means:

- no `performance.now()`, no `requestAnimationFrame`, no accumulated state, no physics
- no `Math.random()`: use a seeded hash of an index instead

Because of this, frames render in any order, chunks resume cleanly, and a single still at any timestamp is cheap. Once everything (fonts, textures) is loaded, set `window.TOTAL`, `window.END_T` and `window.__ready = true`.

### `film.html`

Black body, zero margin, `overflow: hidden`. An import map pointing `three` and `three/addons/` at `node_modules/three/...`. A `#subwrap` / `#sub` subtitle overlay. `<script type="module" src="/scene.js">`.

### Renderer

```js
const W = 1080, H = 1920;
const renderer = new THREE.WebGLRenderer({ antialias: false, preserveDrawingBuffer: true, powerPreference: 'high-performance' });
renderer.setPixelRatio(1); renderer.setSize(W, H); renderer.toneMapping = THREE.NoToneMapping;
document.body.appendChild(renderer.domElement);

const depthTex = new THREE.DepthTexture(W, H); depthTex.type = THREE.UnsignedIntType;
const linear = { type: THREE.HalfFloatType, colorSpace: THREE.LinearSRGBColorSpace };
const rtScene = new THREE.WebGLRenderTarget(W, H, { ...linear, samples: 4, depthTexture: depthTex });
const rt = (w, h) => new THREE.WebGLRenderTarget(w, h, linear);
const rtDof = rt(W, H);
const rtB1 = rt(W >> 2, H >> 2), rtB1b = rt(W >> 2, H >> 2);   // bloom 1/4
const rtB2 = rt(W >> 3, H >> 3), rtB2b = rt(W >> 3, H >> 3);   // bloom 1/8

const quad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2)); quad.frustumCulled = false;
const qScene = new THREE.Scene(); qScene.add(quad);
const qCam = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
const pass = (mat, target) => { quad.material = mat; renderer.setRenderTarget(target); renderer.render(qScene, qCam); };
const VERT = 'varying vec2 vUv; void main(){ vUv = uv; gl_Position = vec4(position.xy, 0.0, 1.0); }';

camera = new THREE.PerspectiveCamera(30, W / H, 1, 500);   // near/far must equal uNear/uFar in the DOF shader
```

The scene renders into a **linear HDR** target (half-float, 4x MSAA, depth texture), so screens and lights can go above 1.0. Then depth of field, then bloom in HDR, then one composite pass that tone-maps and gamma-corrects. That chain is the difference between "filmic" and "WebGL demo".

### Order of work inside `setT`

1. pose characters
2. `scene.updateMatrixWorld(true)`
3. grounding and attachments (props to hands, seats to floor)
4. `updateMatrixWorld` again
5. update anchors (world positions of heads, hands, props, screens)
6. place camera from the shot list
7. update subtitle
8. render scene → DOF → dust → bright-pass → blurs → composite to canvas

### Post-processing

**Depth of field.** Depth-aware gather with 64 Vogel-disc taps. Bright samples get extra weight so highlights turn into bokeh discs. A blurry background is not allowed to bleed over a sharp foreground.

```glsl
uniform sampler2D tColor, tDepth;
uniform vec2 uRes;
uniform float uNear, uFar, uFocus, uAperture, uMaxCoc;
varying vec2 vUv;

float linDepth(float d) { float z = d * 2. - 1.; return 2. * uNear * uFar / (uFar + uNear - z * (uFar - uNear)); }
float coc(float z) { return min(uMaxCoc, uAperture * abs(1. / uFocus - 1. / z)); }

void main() {
  float z0 = linDepth(texture2D(tDepth, vUv).x);
  float c0 = coc(z0);
  float wsum = 1.0 / max(c0 * c0, 1.0);
  vec3 acc = texture2D(tColor, vUv).rgb * wsum;
  const int N = 64;
  for (int i = 0; i < N; i++) {
    float fi = float(i) + 0.5;
    float r = sqrt(fi / float(N));
    float a = fi * 2.39996323;
    vec2 off = vec2(cos(a), sin(a)) * r * uMaxCoc;
    float dist = length(off);
    vec2 uv = vUv + off / uRes;
    float zs = linDepth(texture2D(tDepth, uv).x);
    float cs = coc(zs);
    float eff = (zs > z0) ? min(cs, c0) : cs;            // background can't cover sharp foreground
    float w = smoothstep(dist - 1.0, dist + 1.0, eff) / max(eff * eff, 1.0);
    vec3 s = texture2D(tColor, uv).rgb;
    w *= 1.0 + 1.6 * clamp(dot(s, vec3(0.3, 0.5, 0.2)) - 0.8, 0.0, 6.0);   // bokeh boost
    acc += s * w; wsum += w;
  }
  gl_FragColor = vec4(acc / wsum, 1.0);
}
```

- `uFocus`: distance from camera to the focus anchor, measured **along the view direction**, recomputed every frame.
- `uAperture`: ~1000–2500. Higher = shallower. Close-ups high, wides low. Default to moderate; heavy blur looks like an Instagram filter.
- `uMaxCoc`: 22 px.

**Bloom.**
- Bright-pass: `c * clamp((max(r,g,b) - 1.5) / max(max(r,g,b), 1e-3), 0, 1)` into a 1/4-res target.
- Blur: separable 7-tap Gaussian, weights `0.2270, 0.1946, 0.1216, 0.0540, 0.0162, 0.0054, 0.0016`, step 1.6 px. Once at 1/4 res, then two H+V rounds at 1/8 res.
- Composite adds `0.36 * B1 + 0.55 * B2`.

**Composite**, in this order: add bloom → exposure (~1.05) → ACES filmic → optional gentle colour grade → `pow(col, 1/2.2)` → vignette → film grain (hash of uv + time, ~1–2%) → multiply by `uFade`. A whisper of chromatic aberration at the edges (~0.2%) is okay.

**Dust (optional).** A few hundred `THREE.Points` in their own scene, drawn over the DOF result with the same camera and focus so they blur like everything else. Keep it subtle.

### Debug hooks (build them early)

- URL flags: `?t=<sec>` (render one frame on load), `?nodof`, `?nobloom`, `?bloomonly` (see what glows and smears), `?cam=x,y,z&look=x,y,z&fov=..` (free camera to inspect clearances from outside), `?norender` (pose + anchors only, for `check.mjs`).
- `window.dbg()` → camera position, foot/seat heights, anchors. `window.anchors()` → anchors only.

### Subtitles

HTML overlay, captured together with the canvas in each screenshot.

- `#subwrap`: fixed, full width, top at ~600 px of 1920 (~31%), 224 px tall, flex-centred.
- Text: 54 px, weight 600, `system-ui` / Segoe UI, near-white, centred, `max-width` ~880 px, `text-wrap: balance`.
- Shadow: `0 2px 20px rgba(0,0,0,.72), 0 1px 4px rgba(0,0,0,.6)` plus a faint dark radial gradient behind.
- Max 2 lines. The second character may use a different tint.
- `subtitle(t)` looks up the active chunk in `film.json`, sets `textContent`, fades in over 0.1 s and out over 0.12 s.

**Faces never go in the subtitle band (~31–43% from the top).** Shift the framing without moving the camera: `camera.setViewOffset(W, H, 0, -shift * H, W, H)`. The first subtitle is already on screen at `t = 0`.

---

## 5. Writing the script

The format is fixed: **a developer talks with a Claude character.** Write fresh lines (or use the user's).

- **First line = "Claude" + hook.** Name the AI and drop a situation, claim or confession in one breath. By the end of line one the viewer knows who's talking and what the tension is.
- **Ping-pong, not monologues.** Every line answers or escalates the last. Claude reacts to what the dev actually said: takes it literally, corrects it dryly, agrees unexpectedly, notices something the dev didn't want noticed. The dev pushes back.
- **Relatable over clever.** Late-night debugging, scope creep, tests passing for the wrong reason, confidently wrong answers, rewriting working code, shipping, burnout, imposter feelings, naming things, AI habits (over-agreeing, over-explaining, over-engineering). Specific beats generic. Never explain the joke.
- **Short lines.** Average ≤ 8 words, max ~15. Total 60–120 spoken words for ~50 s. Claude usually says less than the dev.
- **Lock the two personas in the first 5–8 seconds.** One-line persona each, kept consistent. The contrast (energetic vs flat, naive vs precise, emotional vs literal) is the comedy engine.
- **Vary the rhythm.** Quick exchanges, then one held beat (2–3 s silence or a slow close-up), then snap. Punchlines land after the longest pause or right after the setup, never mid-run.
- **Lists come in threes.** The third item is the surprise.
- **End on the laugh, then cut.** A callback or a line that flips the opening. No sign-off, no moral, no end card.
- **Direct the performance.** Every take gets an audio tag. Contrast between consecutive takes (flat then bright, quiet then loud) is part of the joke.
- **Read it at speaking pace.** Over ~55 s? Cut a beat. Don't speed up.

---

## 6. Voices and word timings

### Text to speech

```
POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}/with-timestamps?output_format=mp3_44100_128
headers: xi-api-key, content-type: application/json
body: { text, model_id: "eleven_v3",
        voice_settings: { stability, similarity_boost, style, use_speaker_boost } }
→ { audio_base64, alignment: { characters[], character_start_times_seconds[], character_end_times_seconds[] } }
```

- **One take = one short line.** Timing control comes from cutting lines apart, not from one long paragraph.
- **Audio tags** go at the start of the line in square brackets: `[deadpan]`, `[excited, fast]`, `[softly, moved]`, `[flat]`, `[quietly]`, `[bright, cheerful]`. They're direction, not spoken, and don't show up as words in the alignment. Punctuation steers pacing: `...` pauses, `!` adds energy.
- **Numbers as words** in TTS text (`sixty-three`). Use a subtitle override if you want digits on screen.
- **Settings:** stability 0.3–0.75, similarity_boost 0.75–0.8, style 0.05–0.7, `use_speaker_boost: true`. Energetic = lower stability, higher style. Deadpan = higher stability, low style.
- **2–3 takes for lines that matter** (hook, punchline, emotional beat) with different tags/settings. You can't listen, so pick by duration, pacing and word-timing fit. Keep alternates on disk and tell the user their ids.
- **Two clearly different voices.** List them with `GET /v1/voices`, choose by labels (gender, age, accent, description), let the user override.
- **Cache everything:** `vo/<id>.mp3` + `vo/<id>.json` (`{ text, alignment }`). Never regenerate a cached take. Convert to mono wav: `ffmpeg -nostdin -y -i vo/x.mp3 -ar 44100 -ac 1 vo/x.wav`.

### Ambience and foley

`POST https://api.elevenlabs.io/v1/sound-generation` with `{ text, duration_seconds (≤ ~22), prompt_influence: 0.5 }` → mp3 bytes. Make a room tone, optionally an outside/city bed. Short one-off sounds come out better generated than faked.

### Word timings (`words.mjs`)

Walk the alignment characters, skip everything between `[` and `]`, split on whitespace. Each word: `s` = first character start, `e` = last character end. Print every take as `word[start-end]`; you'll use it to trim takes and cut subtitles.

---

## 7. Timeline (`timeline.mjs`)

**Single source of truth**, imported by both Node and the browser. Audio buses, envelopes, subtitles, lip sync, camera cuts and beat-timed motion all derive from it.

### Cues

One row per spoken line, in playback order:

```js
// [cueId, takeId, src0, src1, silenceBefore, opts]
const RAW = [
  ['g01', 'g01a', 0.10, 2.35, 0.00, {}],
  ['c01', 'c01a', 0.12, 1.20, 0.22, {}],
  // ...
];
```

- `src0` / `src1`: trim inside the take. `src0 = first word start − 0.03`, `src1 = last word end + 0.05..0.15`. v3 leaves long silent tails; trimming them is what makes dialogue snappy.
- `silenceBefore`: gap after the previous cue. **This is your rhythm knob.** Negative = overlap (interruptions).
- Resolve with a cursor: `at = cursor + silenceBefore; cursor = at + (src1 − src0)`. Export `CUES` with `at`, `who`, `take`, `src0`, `src1`.
- Helpers `c0(id, off = 0)` / `c1(id, off = 0)` give a cue's start/end in film time. Use them for anything that happens on a line: reactions, gestures, cuts.
- `END_T` = last line end + 0.3–0.8 s. `TOTAL = END_T + 0.1`. The film **hard-cuts** at `END_T`.
- Optional `MARK` table (`MARK.phoneUp`, `MARK.standUp`) so animation, sound and camera share one number.

### `build-film.mjs` → `film.json`

For every speech cue:

1. **Voice bus.** Measure RMS of the trimmed segment over samples with `|x| > 0.02`. Scale to about **−23 dBFS (dev)** / **−21 dBFS (Claude)**, plus an optional per-cue `gain` in dB. Fade in 8 ms, fade out 30 ms (6 ms if hard-cut). Write into a float32 bus per speaker (`bus_F.wav`, `bus_A.wav`, ...) at sample `at * 44100`.
2. **Speech envelope.** 60 values/s, RMS over a 30 ms window, times cue gain. Normalise per speaker so the 95th percentile = 1, then `min(1, v^0.8)`. Store `{ id, who, t0, t1, env: [...] }`. In the scene, `env(who, t)` interpolates; `envAvg(who, t, w)` averages over a window. Short window (0.06–0.1 s) for a human jaw, long (~0.3 s) for anything that should move calmly.
3. **Subtitle chunks.** Shift word timings to film time and group them:
   - break after a sentence end if the chunk is > 14 chars or the next gap > 0.35 s
   - break at any gap > 0.5 s
   - break before exceeding ~44–54 chars (keeps it ≤ 2 lines)
   - show from `first word start − 0.03` to `last word end + 0.32` (+0.42 for a cue's last chunk), clipped to end 0.03 s before the next chunk
   - minimum 0.7 s on screen; the film's first chunk starts at `t = 0`
   - cue options: `sub:` overrides displayed text (digits, typography), `nosub:` hides it

---

## 8. Camera and editing

### Shot list

Shots live in `timeline.mjs` and are written relative to **anchors**, never absolute coordinates, so the camera follows if a character moves.

```js
// V(anchor, [dx, dy, dz]) = offset from a named anchor (head, prop, pair midpoint, screen...)
{ t0, t1,
  from: { p: ['Pair', [20, 6, 50]], l: ['Pair', [0, 2, 0]], fov: 28 },
  to:   { /* a bit closer */ },
  focus: 'Fhead', ap: 1500, shift: 0.06 }
```

- Anchors (`ANCH.Fhead`, `ANCH.Ahead`, `ANCH.Pair`, props, screen, floor) update at the end of the pose step.
- Interpolate position, look-at and fov together with **smootherstep**: `x*x*x*(x*(x*6-15)+10)`.
- Most shots are a **slow push-in or drift of 4–12%**. Nothing locked off for more than ~2 s. No whip pans unless the script calls for it.
- **Handheld life:** two slow sines on camera x/y (~0.07 and ~0.03 units, scaled by distance) plus ~0.006 rad slow roll. Barely visible.
- Look point slightly **above** the subject centre (headroom). Don't crop at joints by accident.
- `focus` = the anchor that must be sharp, usually the speaker's head.

**Framing math.** Visible height = `2 × distance × tan(fov/2)`, width = `0.5625 ×` that. Think in character heights (H) at subject distance:

| Framing | Height |
|---|---|
| Wide | 4–6 H |
| Two-shot | 2.5–3 H |
| Single (waist up) | 1.5–2 H |
| Close-up (head + shoulders) | 0.7–1 H |
| Extreme close-up (eyes, hand, prop) | 0.3–0.5 H |

Vertical fov 22–31° for most shots, 28–34° for wides. Prefer low side angles (chest height, 20–45° off-axis) over top-down, and avoid the dead-flat axis through both characters.

### Shot types

| Shot | When | Length | Notes |
|---|---|---|---|
| Hook | Opening line; frame 1 already shows speaker + subtitle | 2–3.5 s | Slow push, no fade from black |
| Establishing wide | Show the place, once, early or at a reveal | 2–3 s | Whole set in frame, look point above the surface |
| Very wide | Isolation, awkward silence, a beat before a punchline | 2–4 s | Small characters, almost no motion |
| Two-shot | Fast back-and-forth, both reactions at once | 2–4 s | Both faces clear of subtitles, no tangents |
| Single | A line that needs focus | 1.5–4 s | Cut in on the first word |
| Reaction | Listener's face after a line lands | 0.6–2 s | Cut on line end; hold the deadpan a half-beat too long |
| Close-up | Emotional or sincere beat, punchline setup | 2–5 s | Smallest fov, shallow DOF |
| Insert | Hand, screen, prop the story needs | 0.8–2 s | Never long, never a readable screen; let the glow on the face tell it |
| Pull-back | Closing line | 2–4 s | Slow pull-out, hard cut after the last word |

### Cutting rules

- **Cut on dialogue:** on a word start or just after a line ends. Never mid-word, never on silence without a reason. If you can't name the reason for a cut, delete it.
- **No near-duplicate framings back to back:** change distance by ~30% or angle by ~30°.
- **Rhythm:** short shots for banter, long shots for slow beats. ~12–18 shots per 50 s, mostly 2–4 s.
- **Geography:** each character stays on the same side of frame across cuts.
- **Depth:** wides deep focus (lower aperture), close-ups shallow. The speaker's face is always sharp.

---

## 9. Characters, motion and QA

### Building characters

Look and set are free. Constraints:

- Primitives in a **named joint hierarchy**: `root`, `pelvis`, `torso`, `head`, `armL/armR`, `legL/legR`, face parts. Poses come from small key tables, never physics. Expose key joints as anchors.
- The Claude character reads as Claude at a glance (colour, shape, behaviour, name) **without copying a logo**. A simple memorable silhouette beats detail.
- **Phone-readable:** strong silhouette, high contrast with the background, lit faces (soft key, back rim, faint fill). No black-hole faces.
- **Low-poly, flat or simple materials.** Light, DOF and bloom provide the richness.

### Motion rules (break these and it looks buggy)

1. **Ease everything.** `key(t, [[t0, v0], [t1, v1], ...])` with smootherstep. Times strictly increasing, never duplicated (a duplicate = divide by zero = a jump). No linear moves, springs, overshoot or continuous morphing. Idle life = slow sines (breathing 0.2–0.3 Hz, tiny).
2. **Act small.** Head, face and camera, plus **one** slow looping motion per character at a time. Big simultaneous body motion reads as a glitch.
3. **No laughing, shaking or jitter.** Nothing faster than ~2 Hz, nothing driven at full amplitude by the voice envelope.
4. **Calm speech motion.** With a mouth: jaw follows `env` through a 0.06–0.1 s window, clamped, never snapping shut. Without a mouth (mascot, block): only a ~1–2% squash/stretch, eye height and small head tilt from `envAvg(who, t, 0.3)`, plus a bigger move on 1–3 chosen punchline words. Raw `env` on position/rotation = jitter.
5. **Always grounded.** Feet or seat touch the surface in every frame. If a leg swings, the pelvis moves with it. Each frame, find the lowest world Y (`new THREE.Box3().setFromObject(...)`) and place the root exactly on the surface, after posing and before rendering. Floating or sinking by even a few hundredths is the most visible bug.
6. **Clearances are measured.** Limbs vs furniture, hands vs keyboard, head vs lamp: Box3 checks every 0.1 s. Near a surface, give real clearance (raise the character, angle the arms).
7. **No coplanar faces.** Same-depth faces flicker. Offset ≥ 0.03 units or use `polygonOffset`. Close-ups expose it.
8. **Props face the eyes, hands look like hands.** Screens point at the face, wrist angle plausible, hand = palm block + 4 wrapped fingers + thumb. Props parented to (or placed from) the hand every frame, never floating.
9. **Prop light stays moderate.** Screen glow = point/spot lights on the prop, scaled by a `look` factor with a gentle ramp. Face glow dimmer than the screen. Check `?bloomonly`; dial down anything that smears.
10. **Camera never clips** into geometry or sees the back of the set. Check the end of every push and pull, not just the middle.
11. **Subtitles never cover faces.**
12. **Frame 1 is a finished picture** (it's the cover). No black, no fade-in. End on a **hard cut**.

### `check.mjs`: run before any full render

1. **Motion:** with `?norender`, call `setT` for every frame, collect anchors (heads, hands, feet/seat, props, camera), compute velocity and acceleration. Flag:
   - teleports: frame delta > 3× median speed (ignore shot cuts)
   - spikes: acceleration > ~6× its 95th percentile
   - jitter: direction flip within 3 frames at > ~0.03 units
   Fix by smoothing the driver (longer window, fewer keys, more easing). Repeat until zero.
2. **Grounding:** every 0.1 s, lowest point vs surface, |error| < 0.02 units.
3. **Penetration:** Box3 intersection of limbs/props vs furniture/keyboard/body across the sweep.
4. **Lint:** key times strictly increasing, every anchor `isFinite` at every frame, cuts land on word starts/ends.
5. **Subtitles:** every chunk ≤ 2 lines at 54 px in 880 px (measure with `getBoundingClientRect`), ≥ 0.7 s, no overlaps.
6. **Audio:** scan the mix for clicks (sample jump > 0.5 within 1 ms outside speech) and clipping; `make.mjs` confirms no stem is older than `timeline.mjs`.

### Stills and contact sheet

`render.mjs stills 0.2,2.0,...` → `stills/t<time>.png`. Always include frame 1, the first and last second of every shot, every motion beat, and the frame right before the end. Tile them:

```
ffmpeg -pattern_type glob -i 'stills/*.png' -vf "scale=270:-1,tile=8x2" sheet.png
```

Look at every tile: faces readable? Anything floating, clipping, black or blown out? Subtitles clear of faces? Background neither empty nor messy?

---

## 10. Sound

The voices carry the video. Keep everything else simple and clean.

- **Beds:** room tone at ~**−47 dB** mean, optional outside ambience at ~**−50 dB**. Loop with `acrossfade=d=2.5` (5 copies, chained). Highpass ~45–60 Hz, lowpass ~4–6.5 kHz.
- **Music (optional):** slow synth pad (detuned sine/triangle stacks, slow chord changes, gentle low-pass) at ~**−37 dB** mean before normalisation, ducked another 6–8 dB under speech using the cue mask (fast attack, slow release). A bed, not a song. No copyrighted tracks.
- **Voice chain per speaker:** `highpass=f=85, lowpass=f=9500, [optional gentle treble], acompressor=threshold=0.045:ratio=2.6:attack=8:release=140:makeup=2`. Small room send: `aecho=0.8:0.5:41|79|133:0.26|0.18|0.1, lowpass=f=3400, volume=0.6`, mixed back at ~0.6. Slight stereo offset between speakers (0.56/0.44 vs 0.44/0.56). In-world sounds (phone, notification) get a narrower band and lower level. Match stems by **measured** level (`volumedetect`), not guesswork.
- **Final mix:** `amix=inputs=N:normalize=0:duration=longest`, trim to `TOTAL`, `afade` in 0.08 s, fade-out matching the picture, `apad=whole_dur=TOTAL`. Then **two-pass `loudnorm`** to `I=-14, TP=-1.5, LRA=9` (first pass `print_format=json`, second with `measured_*`, `offset`, `linear=true`), then `alimiter=limit=0.84`. Confirm with `ebur128=peak=true`: integrated ≈ −14.0, true peak ≤ −1.5.
- **Timeline changed → rebuild every stem.** Regenerate any foley timed from `MARK`; a stale tick track sounds like a mystery click.

---

## 11. Render, mux, deliver

### Render (`render.mjs`)

Modes: `stills <times>` and `full [from] [to]`. Both use `cdp.mjs`, serve the project folder from a local static server on `127.0.0.1:0` (with a no-store cache header) and open `/film.html`.

- **One browser.** Wait for `window.__ready === true` (90 s timeout; dump console on failure).
- **Per frame:** `await api.eval("window.setT(" + (f / 30).toFixed(4) + ")")`, then `Page.captureScreenshot { format: 'jpeg', quality: 92 }` → `frames/f0000.jpg` (4-digit zero padding).
- **Resumable:** skip frames that exist and are > 8 KB. Render ~400–600 frames per call so each call stays under 8 minutes; repeat with the next range.
- **Blank-frame guard:** a screenshot under ~20 KB (outside the final fade) means the WebGL context died. Close, relaunch, retry the frame (max 3). After rendering, any frame under 8 KB (except the last few) gets its range redone.
- **Speed:** ~0.15–0.3 s/frame on a mid-range GPU, so 1500 frames ≈ 4–7 minutes. Don't add workers to go faster.
- Never switch GPU/CPU mode mid-film.

### Mux

```
ffmpeg -nostdin -y -framerate 30 -i frames/f%04d.jpg -i mix.wav -c:v libx264 -crf 25 -preset medium -maxrate 14M -bufsize 28M -pix_fmt yuv420p -profile:v high -c:a aac -b:a 192k -shortest -movflags +faststart "<Title>.mp4"
```

Audio-only fix later, without re-rendering:

```
ffmpeg -i old.mp4 -i mix.wav -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -b:a 192k -shortest -movflags +faststart new.mp4
```

### Verify before saying "done"

- `ffprobe`: 1080x1920, 30 fps, `yuv420p`, duration within one frame of `TOTAL`, audio and video within one frame of each other.
- Frame 1 isn't black (`signalstats` YAVG clearly > 0) and matches the planned hook frame.
- Loudness ≈ −14 LUFS integrated, true peak ≤ −1.5 dBTP.
- ~12 frames pulled from the **final mp4** (`ffmpeg -ss <t> -i out.mp4 -frames:v 1`) into a contact sheet: compression artifacts, blown glow, subtitles.
- Check a few cut points for clicks (via analysis).

### Deliver

Save to a normal user folder (e.g. `Videos/<project>/`), never overwrite earlier versions (`v2`, `v3`, ...), open it in the default player, and give a short report: title, duration, shot count, deliberate choices (voices, tags, which takes have alternates), what you'd push further, and any flaws you noticed, honestly. Don't post it anywhere.

---

## 12. Checklist

- [ ] Requirements met, GPU probe passes, key present and never printed
- [ ] 3–5 line brief: topic, length, both characters, set, both voices
- [ ] Script: "Claude" + hook up front, ping-pong, relatable, ends on the laugh
- [ ] Takes generated (alternates for key lines), wavs + word timings printed
- [ ] `timeline.mjs`: trimmed cues, rhythm gaps, `END_T`, every shot has a reason
- [ ] `build-film.mjs`: buses, envelopes, subtitle chunks; `make.mjs` runs it all in order
- [ ] `scene.js`: stateless `setT`, post chain, anchors, grounding, camera, subtitles
- [ ] Stills sheet approved, `check.mjs` clean
- [ ] Mix measured: −14 LUFS, TP ≤ −1.5, no stale stems
- [ ] Full render in foreground chunks, one browser, low priority, all frames > 8 KB
- [ ] Mux, verify, deliver, honest short report
