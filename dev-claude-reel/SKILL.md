---
name: dev-claude-reel
description: Build a short vertical (9:16) cinematic 3D reel of a conversation between a developer and a "Claude" character. Covers the full pipeline - script, ElevenLabs voices, a three.js scene rendered frame by frame in headless Chrome, shots, mix, mp4 - plus QA rules for buggy or weird movement. Characters, set and look are yours to design.
---

# Dev <-> Claude reel: how to make one

You will produce a **vertical video of about 50 seconds (1080x1920, 30 fps, H.264 + AAC)**: a 3D low-poly scene, two characters (a developer and a "Claude" character) who have a short funny conversation, real voices, subtitles, a camera that cuts on the lines, and a mix. The video is rendered **one frame at a time** from a three.js page in headless Chrome, then muxed with ffmpeg.

This file tells you the **mechanics** and the **rules that stop it from looking buggy**. It deliberately does **not** tell you what the characters look like, what the room looks like, or what the story is. Those are yours. If the user gave you a topic, character notes or a reference, use those. Otherwise decide, in a few lines, and go.

Work in this order and don't skip ahead: **brief -> script -> voices -> timeline -> scene -> stills -> motion check -> music/mix -> full render -> mux -> verify -> deliver.**

---

## 0. Rules of engagement (read first)

1. **Be gentle with the user's computer.** This is their everyday PC. Rendering must use **one headless browser, on the GPU, at idle (lowest) priority, no parallel workers**. If WebGL falls back to CPU rendering (the probe below returns SwiftShader / "software"), **stop and tell the user**. CPU rendering of a 3D film pegs every core for tens of minutes and can freeze the machine. Do not pass SwiftShader flags yourself.
2. **Never print secrets.** The ElevenLabs key is `ELEVENLABS_API_KEY` (look in the user's `.env` or environment). Load it inside scripts, never echo it, never put it in a log, a page or a file you show.
3. **Stills before film.** Never launch a full render until a contact sheet of stills looks right. Re-rendering the whole film for a one-shot fix is the most expensive mistake in this workflow.
4. **Background jobs die when your turn ends.** Render in **foreground chunks** (resumable, see section 8), each call comfortably under your tool timeout (aim for <= 8 minutes).
5. **One video per chat.** The scene file gets big. Keep it split into parts and don't re-read it needlessly. Start a new chat for the next video.
6. **Don't publish anything.** You deliver an mp4 file. Posting to Instagram/X/etc. is the user's call.
7. If something in this file conflicts with what the user asks, the user wins. If something is missing (no key, no ffmpeg, no Chrome), say exactly what and offer the closest alternative. Never fake success.

## 1. Requirements check (do this in the first minute)

- **Node 22+** (global `fetch` and `WebSocket` are used; no npm packages except three.js).
- **Chrome or Edge** installed (headless is used). **ffmpeg + ffprobe** on PATH (or at a known path).
- **three.js**: `npm i three@0.170` in the project folder. It is served from disk through a local static server (no CDN, no network at render time).
- **ElevenLabs API key**. The free plan works (about 10,000 credits a month, API access, premade voices; no commercial licence, so a paid plan is needed for monetised videos). Budget roughly 1 credit per character of text, and sound-generation calls cost credits too: write the script first, count its characters, keep alternate takes to the key lines only, and cache everything so nothing is generated twice. Use `eleven_v3`; if the API refuses it for this account, fall back to `eleven_multilingual_v2`, **remove the `[audio tags]` from the text**, and tell the user the delivery will be flatter. If the key is missing: say so, finish script + scene + silent test render as far as useful, and ask the user for the key. (Degraded fallback if they insist: OS text-to-speech, word timings estimated from character counts, mouth sync and subtitles will be rougher. Say so honestly.)
- A GPU. Probe it once (section 3).

## 2. Project layout

One folder per video, plain files, no build step:

```
reel/
  package.json            (three)         
  film.html               page: <canvas>, subtitle overlay, import map, loads scene.js
  scene.js                the 3D world + setT(t)  (split into parts if large)
  timeline.mjs            single source of truth for timing + shots (imported by Node AND the browser)
  gen-audio.mjs           takes.json -> ElevenLabs voices + ambience (disk-cached)
  words.mjs               word timings from the alignment json
  build-film.mjs          voice buses + film.json (envelopes, subtitles)
  music.mjs  mix.mjs      optional bed + the final mix
  cdp.mjs                 headless Chrome driver
  render.mjs              stills | full (chunked, resumable)
  vo/ sfx/ frames/ stills/
```

`make.mjs` (recommended): a tiny script that runs, in order, `gen-audio -> build-film -> music -> mix`. **Whenever `timeline.mjs` changes, rerun all of it.** Stale stems are the classic silent bug (a stray sound that belongs to an old timing).

---

## 3. The headless-Chrome driver (`cdp.mjs`)

Dependency-free Chrome DevTools Protocol client. Requirements, in priority order:

- Flags: `--headless=new --remote-debugging-port=<p> --user-data-dir=<tmp> --no-first-run --disable-extensions --disable-background-timer-throttling --disable-renderer-backgrounding --mute-audio --window-size=W,H --use-gl=angle --use-angle=<d3d11 (Windows) | metal (macOS) | gl (Linux)> --ignore-gpu-blocklist --enable-gpu`.
- After connecting: `Emulation.setDeviceMetricsOverride {width:1080,height:1920,deviceScaleFactor:1,mobile:false}`.
- **GPU probe**: evaluate `WEBGL_debug_renderer_info` -> `UNMASKED_RENDERER_WEBGL`. If it matches `/swiftshader|software|basic render/i` or is empty -> **abort and tell the user** (rule 1).
- **Idle priority**: `os.setPriority(os.constants.priority.PRIORITY_LOW)` at the top of every Node script, and re-lower the browser and *all* its child processes every ~5 s (Chrome raises its GPU process by itself). Child pids come from `SystemInfo.getProcessInfo` on the browser websocket (`/json/version` -> `webSocketDebuggerUrl`). On Windows also pin the browser to a single logical processor (`Get-Process -Id <pid>).ProcessorAffinity = <one bit>` via PowerShell) so a render can never take more than ~1/N of the machine.
- API: `goto(url)`, `eval(expr)` (with `awaitPromise`, `returnByValue`, throw on `exceptionDetails`), `waitFor(expr, timeout)`, `send(method, params)`, `console` (collect `Runtime.consoleAPICalled`, `Runtime.exceptionThrown`, `Log.entryAdded`), `close()` (kill process, delete the temp profile).

Core of it (adapt, don't copy blindly):

```js
import { spawn } from 'node:child_process'; import fs from 'node:fs'; import os from 'node:os'; import path from 'node:path';
const CHROME = ['C:/Program Files/Google/Chrome/Application/chrome.exe', 'C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe',
  '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome', '/usr/bin/google-chrome'].find(p => fs.existsSync(p));
const ANGLE = process.platform === 'win32' ? 'd3d11' : process.platform === 'darwin' ? 'metal' : 'gl';
export async function launch({ width = 1080, height = 1920, port = 9450 } = {}) {
  try { os.setPriority(os.constants.priority.PRIORITY_LOW); } catch {}
  const udd = fs.mkdtempSync(path.join(os.tmpdir(), 'reel-'));
  const proc = spawn(CHROME, [`--remote-debugging-port=${port}`, `--user-data-dir=${udd}`, '--headless=new', '--no-first-run', '--disable-extensions',
    '--disable-background-timer-throttling', '--disable-renderer-backgrounding', '--mute-audio', `--window-size=${width},${height}`,
    '--use-gl=angle', `--use-angle=${ANGLE}`, '--ignore-gpu-blocklist', '--enable-gpu', 'about:blank'], { stdio: 'ignore' });
  let list; for (let i = 0; i < 100 && !list?.length; i++) { try { list = await (await fetch(`http://127.0.0.1:${port}/json/list`)).json(); } catch { await new Promise(r => setTimeout(r, 150)); } }
  const ws = new WebSocket(list.find(t => t.type === 'page').webSocketDebuggerUrl); await new Promise((res, rej) => { ws.onopen = res; ws.onerror = rej; });
  let id = 0; const pending = new Map(), logs = [];
  ws.onmessage = ev => { const m = JSON.parse(ev.data);
    if (m.id && pending.has(m.id)) { const { res, rej } = pending.get(m.id); pending.delete(m.id); m.error ? rej(new Error(m.error.message)) : res(m.result); }
    else if (m.method === 'Runtime.exceptionThrown') logs.push({ type: 'exception', text: m.params.exceptionDetails.exception?.description || m.params.exceptionDetails.text });
    else if (m.method === 'Runtime.consoleAPICalled') logs.push({ type: m.params.type, text: m.params.args.map(a => a.value ?? a.description).join(' ') }); };
  const send = (method, params = {}) => new Promise((res, rej) => { const i = ++id; pending.set(i, { res, rej }); ws.send(JSON.stringify({ id: i, method, params })); });
  await send('Runtime.enable'); await send('Page.enable'); await send('Log.enable');
  await send('Emulation.setDeviceMetricsOverride', { width, height, deviceScaleFactor: 1, mobile: false });
  // + GPU probe, + polite() timer that lowers priority of proc.pid and every child pid every 5 s, + (Windows) affinity pin
  const api = { send, logs, proc,
    async eval(expr) { const r = await send('Runtime.evaluate', { expression: expr, returnByValue: true, awaitPromise: true }); if (r.exceptionDetails) throw new Error('eval failed: ' + (r.exceptionDetails.exception?.description || r.exceptionDetails.text)); return r.result.value; },
    async goto(url) { await send('Page.navigate', { url }); await api.waitFor('document.readyState === "complete"', 15000); },
    async waitFor(expr, ms = 10000) { const t0 = Date.now(); while (Date.now() - t0 < ms) { try { if (await api.eval(expr)) return true; } catch {} await new Promise(r => setTimeout(r, 100)); } throw new Error('timeout: ' + expr); },
    async close() { try { ws.close(); } catch {} try { proc.kill(); } catch {} setTimeout(() => fs.rmSync(udd, { recursive: true, force: true }), 500); } };
  return api;
}
```

---

## 4. The page contract (`film.html` + `scene.js`)

`film.html`: black body, no margins, `overflow:hidden`; an import map pointing `three` and `three/addons/` at the local `node_modules/three/...`; a `#subwrap`/`#sub` subtitle overlay; `<script type="module" src="/scene.js">`.

**The one rule that makes everything else work: `window.setT(t)` is a pure function of time.** Given `t` (seconds) it positions every character, prop, light and the camera, renders the whole frame, and returns. No clocks (`performance.now`, `requestAnimationFrame`), no accumulated state, no `Math.random()` (use a seeded hash of the index), no physics. Then any frame can be rendered in any order, chunks can resume, and a single still at any time is cheap. Also set `window.TOTAL` / `window.END_T` and `window.__ready = true` once the page is built and fonts/textures are loaded.

Renderer setup that works:

```js
const W = 1080, H = 1920;
const renderer = new THREE.WebGLRenderer({ antialias: false, preserveDrawingBuffer: true, powerPreference: 'high-performance' });
renderer.setPixelRatio(1); renderer.setSize(W, H); renderer.toneMapping = THREE.NoToneMapping; document.body.appendChild(renderer.domElement);
const depthTex = new THREE.DepthTexture(W, H); depthTex.type = THREE.UnsignedIntType;
const rtScene = new THREE.WebGLRenderTarget(W, H, { type: THREE.HalfFloatType, samples: 4, depthTexture: depthTex, colorSpace: THREE.LinearSRGBColorSpace });
const rt = (w, h) => new THREE.WebGLRenderTarget(w, h, { type: THREE.HalfFloatType, colorSpace: THREE.LinearSRGBColorSpace });
const rtDof = rt(W, H), rtB1 = rt(W >> 2, H >> 2), rtB1b = rt(W >> 2, H >> 2), rtB2 = rt(W >> 3, H >> 3), rtB2b = rt(W >> 3, H >> 3);
const quad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2)); quad.frustumCulled = false; const qScene = new THREE.Scene(); qScene.add(quad); const qCam = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
const pass = (mat, target) => { quad.material = mat; renderer.setRenderTarget(target); renderer.render(qScene, qCam); };
const VERT = 'varying vec2 vUv; void main(){ vUv = uv; gl_Position = vec4(position.xy, 0.0, 1.0); }';
camera = new THREE.PerspectiveCamera(30, W / H, 1, 500);   // near/far must match uNear/uFar in the DOF shader
```

Why this chain: the scene renders in **linear HDR** (half-float, 4x MSAA, depth texture) so lights and screens can exceed 1.0; a **depth-of-field** pass uses the depth to blur; **bloom** is added in HDR; the final composite does tone mapping + gamma. It is what makes the frames look filmic rather than "WebGL demo".

**Frame order inside `setT`:** pose characters -> `scene.updateMatrixWorld(true)` -> ground/attach fixes (props to hands, seat to floor) -> `updateMatrixWorld` again -> update **anchors** (world positions of heads, hands, props, screens) -> place camera from the shot list -> subtitle -> render scene -> DOF -> dust -> bright-pass -> blurs -> composite to canvas.

### Post chain shaders

DOF (depth-aware gather, 64 Vogel-disc taps; bright pixels bloom into bokeh discs; far background is not allowed to bleed over a sharp foreground):

```glsl
uniform sampler2D tColor,tDepth; uniform vec2 uRes; uniform float uNear,uFar,uFocus,uAperture,uMaxCoc; varying vec2 vUv;
float lin(float d){ float z = d*2.-1.; return 2.*uNear*uFar/(uFar+uNear - z*(uFar-uNear)); }
float cocOf(float z){ return min(uMaxCoc, uAperture*abs(1./uFocus - 1./z)); }
void main(){
  float z0 = lin(texture2D(tDepth,vUv).x); float c0 = cocOf(z0);
  vec3 acc = texture2D(tColor,vUv).rgb * (1.0/max(c0*c0,1.0)); float ws = 1.0/max(c0*c0,1.0);
  const int N = 64;
  for(int i=0;i<N;i++){
    float fi = float(i)+0.5; float r = sqrt(fi/float(N)); float a = fi*2.39996323;
    vec2 off = vec2(cos(a),sin(a))*r*uMaxCoc; float dist = length(off); vec2 uv = vUv + off/uRes;
    float zs = lin(texture2D(tDepth,uv).x); float cs = cocOf(zs);
    float eff = (zs > z0) ? min(cs, c0) : cs;
    float w = smoothstep(dist-1.0, dist+1.0, eff) / max(eff*eff, 1.0);
    vec3 s = texture2D(tColor,uv).rgb; float lum = dot(s, vec3(0.3,0.5,0.2));
    w *= 1.0 + 1.6*clamp(lum-0.8, 0.0, 6.0);
    acc += s*w; ws += w;
  }
  gl_FragColor = vec4(acc/ws, 1.0);
}
```
Uniforms: `uFocus` = camera-to-focus-point distance **along the view direction** (recompute each frame from the shot's focus anchor), `uAperture` ~ 1000-2500 (higher = shallower; close-ups higher, wides lower), `uMaxCoc` = 22 px. Default to a **moderate** aperture; heavy blur reads as a filter.

Bloom: bright-pass `c * clamp((max(r,g,b) - 1.5)/max(max(r,g,b),1e-3), 0, 1)` into a 1/4-res target, then separable 7-tap Gaussian (weights 0.2270, 0.1946, 0.1216, 0.0540, 0.0162, 0.0054, 0.0016, step 1.6 px) at 1/4 res and again at 1/8 res (two H+V rounds at 1/8). Composite adds `0.36*B1 + 0.55*B2`.

Composite: add bloom -> exposure (~1.05) -> ACES filmic -> optional subtle colour grade (your taste, keep it gentle) -> `pow(col, 1/2.2)` -> vignette -> a little film grain (hash of uv + time, amplitude ~1-2%) -> multiply by `uFade`. A very faint chromatic aberration (~0.2% at the edges) is fine. Everything is written to the canvas by the composite shader.

Optional: a few hundred dust/air particles as `THREE.Points` in a separate scene, drawn over the DOF output with the same camera and the same focus (they should blur like everything else). Subtle only.

### Debug switches (build them, you will need them)

Query flags: `?t=<seconds>` render one frame at load; `?nodof`, `?nobloom`, `?bloomonly` (shows only the glow, to see what smears), `?cam=x,y,z&look=x,y,z&fov=..` (free debug camera for checking clearances from outside). Also `window.dbg()` returning camera position, foot/seat heights and anchors, and `window.anchors()`.

### Subtitles (HTML overlay, captured in the screenshot)

`#subwrap` fixed, full width, **top ~600 px of 1920 (about 37% of the frame height), 224 px tall, flex-centred**; text 54 px, weight 600, `system-ui`/Segoe UI stack, near-white, centred, `max-width` ~880 px, `text-wrap: balance`, soft dark shadow (`0 2px 20px rgba(0,0,0,.72), 0 1px 4px rgba(0,0,0,.6)`) and a faint dark radial gradient behind. **At most 2 lines.** Different tint for the second character is fine. `subtitle(t)` finds the active chunk in `film.json`, sets `textContent`, fades in over 0.1 s and out over 0.12 s.

**The subtitle band is a no-go zone for faces.** Compose each shot so eyes and mouths are not under the text (use `camera.setViewOffset(W, H, 0, -shift*H, W, H)` to slide the framing up or down without changing the camera). Subtitles are part of the picture from **frame 1**: the first line of text must already be on screen at t=0.

---

## 5. Script, voices, timings

### 5.1 Voices (ElevenLabs)

```
POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}/with-timestamps?output_format=mp3_44100_128
headers: xi-api-key, content-type: application/json
body: { text, model_id: "eleven_v3", voice_settings: { stability, similarity_boost, style, use_speaker_boost } }
-> JSON { audio_base64, alignment: { characters[], character_start_times_seconds[], character_end_times_seconds[] } }
```

- **One take = one short line (one sentence or a beat).** Fine timing control comes from cutting lines apart, not from one long paragraph.
- v3 understands **audio tags in square brackets** at the start of the line: `[deadpan]`, `[excited, fast]`, `[softly, moved]`, `[flat]`, `[quietly]`, `[bright, cheerful]`. Tags are performance direction, they are not spoken, and they are **not** in the alignment as words (skip `[...]` when building word timings). Punctuation drives pacing: `...` is a pause, `!` pushes energy.
- Spell numbers as words in the TTS text (`sixty-three`, `one point seven million`). If the subtitle should show digits, give the cue a display override (see 6.2).
- Voice settings starting points: stability 0.3-0.75, similarity_boost 0.75-0.8, style 0.05-0.7 (style up = more acting, also more drift), `use_speaker_boost: true`. Lower stability + higher style for energetic lines, higher stability + low style for flat/deadpan lines.
- **Make two or three takes of every line that matters** (the hook, the punchline, any emotional beat) with different tags/settings. You cannot hear them, so choose by **duration, pacing and word timing fit**, keep the alternates on disk, and tell the user which take ids are alternates so they can ask for a swap.
- Pick **two clearly different voices** (list `GET /v1/voices`, use the labels: gender, age, accent, description, and let the user override). The contrast between the two voices carries the comedy.
- Cache everything on disk (`vo/<id>.mp3`, `vo/<id>.json` with `{text, alignment}`), never regenerate a cached take, never print the key. Convert each mp3 to mono 44.1 kHz wav: `ffmpeg -nostdin -y -i vo/x.mp3 -ar 44100 -ac 1 vo/x.wav`.
- Ambience and sfx: `POST https://api.elevenlabs.io/v1/sound-generation` `{text, duration_seconds (<= ~22), prompt_influence: 0.5}` -> mp3 bytes. Generate a room tone (and optionally an outside/city bed). Short one-off foley sounds are better made from this than faked.

### 5.2 Word timings

From the alignment: walk characters, skip everything between `[` and `]`, split on whitespace, each word gets `s` = first char start and `e` = last char end. `node words.mjs` should print every take as `word[start-end]`. You use this to trim takes and to cut the subtitles.

---

## 6. Timeline (`timeline.mjs`, shared by Node and the browser)

The **single source of truth.** Everything (audio bus, envelopes, subtitles, mouth sync, camera cuts, beat-synced motion) derives from it.

### 6.1 Cues

One row per spoken line, in playback order:

```js
// [cueId, takeId, src0, src1, silenceBefore, opts]    who = first letter convention or an explicit field
const RAW = [ ['g01', 'g01a', 0.10, 2.35, 0.00, {}], ['c01', 'c01a', 0.12, 1.20, 0.22, {}], /* ... */ ];
```
- `src0/src1` = trim inside the take, from the word timings: `src0 = first word start - 0.03`, `src1 = last word end + 0.05..0.15` (v3 leaves long tails of silence; trimming them is what makes the dialogue tight).
- `silenceBefore` = gap since the previous cue ended. **This is the rhythm control** (see 9.3). A negative value overlaps (interruptions).
- Resolve in a loop: `at = cursor + silenceBefore; cursor = at + (src1 - src0)`. Export `CUES` with `at`, `who`, `take`, `src0`, `src1`. Helpers `c0(id, off=0)` / `c1(id, off=0)` = start/end of a cue in film time: use them for **everything that should happen on a line** (a reaction, a gesture, a camera cut).
- `END_T` = the end of the last line plus a short tail (~0.3-0.8 s); `TOTAL = END_T + 0.1`. **The film hard-cuts at `END_T`** (see 9.4).
- Optional `MARK` table for named moments (`MARK.phoneUp`, `MARK.standUp`) so animation, sound and camera agree on one number.

### 6.2 Subtitles and mouth data (`build-film.mjs` -> `film.json`)

For every cue (except sfx):
1. **Bus gain**: take the trimmed segment, measure RMS over samples with `|x| > 0.02`, scale so speech RMS lands at about **-23 dBFS (dev) / -21 dBFS (Claude)**, add a per-cue `gain` (dB) for deliberate loud/quiet lines. Apply an 8 ms fade-in and a 30 ms fade-out (6 ms if the line is hard-cut). Write into one float32 bus **per speaker** (`bus_F.wav`, `bus_A.wav`, ...), at sample `at*44100`.
2. **Speech envelope**: 60 values per second, RMS over a 30 ms window around each step, times the cue gain. Per speaker, normalise so the 95th percentile = 1, then `min(1, v^0.8)`. Store `{id, who, t0, t1, env: [...]}`. In the scene, `env(who, t)` linearly interpolates it; `envAvg(who, t, w)` averages it over a window `w`. Use a **short window (~0.06-0.1 s) for a human mouth** and a **long one (~0.3 s) for anything that should move calmly** (see 8.2).
3. **Subtitles**: take the cue's words (with timings shifted to film time) and **chunk** them: break after a sentence end when the chunk is > 14 chars or the next gap > 0.35 s, break at any gap > 0.5 s, break when the chunk would exceed ~44-54 chars (so it stays on <= 2 lines). A chunk shows from `first word start - 0.03` to `last word end + 0.32` (the last chunk of a cue `+0.42`), clipped so it ends 0.03 s before the next chunk begins; **minimum on-screen time 0.7 s**. The first chunk of the film starts at `t=0`. A cue may carry `sub:` to override the displayed text (digits instead of spelled numbers, a typographic fix) and `nosub:` for lines that need none.

---

## 7. Shots and camera

### 7.1 Shot list (in `timeline.mjs`)

```js
// V(anchor, [dx,dy,dz]) = a point relative to a named anchor (a head, a prop, the pair, the screen...)
// shot: [id, start, from, to, opts]  from/to = { p: V(...), l: V(...), fov }
{ t0, t1, from: { p: ['Pair', [20, 6, 50]], l: ['Pair', [0, 2, 0]], fov: 28 }, to: { ...slightly closer... }, focus: 'Fhead', ap: 1500, shift: 0.06 }
```
- Anchors are **world-space points updated at the end of the pose step** (`ANCH.Fhead`, `ANCH.Ahead`, `ANCH.Pair` (midpoint), props, the screen, the floor under them). Cameras are written relative to anchors, never in absolute coordinates, so if a character moves, the shot follows.
- Interpolate `from -> to` over the shot with **smootherstep** (`x*x*x*(x*(x*6-15)+10)`), position, look-at and fov together. Most shots are a **slow push-in or drift** of 4-12% over their duration. Never a static locked-off frame for more than ~2 s; never a fast whip unless the script is a whip.
- Add tiny handheld life: a sum of two slow sines on camera x/y (amplitude ~0.07 and ~0.03 world units, scaled with distance), and ~0.006 rad of slow roll. It must be **barely visible**.
- `look` points slightly **above** the subject's centre (headroom), and the subject is never cropped at the joints (don't cut at the neck or knees without intent).
- `focus` = the anchor that must be sharp (the speaker's head, usually).
- A frame is `visibleHeight = 2 * distance * tan(fov/2)` tall and `0.5625 *` that wide. Think in **multiples of a character's height H** at the subject distance, then verify with a still: wide 4-6 H, two-shot 2.5-3 H, single (waist-up) ~1.5-2 H, face close-up ~0.7-1 H (head and shoulders), extreme close-up (eyes/hand/prop detail) ~0.3-0.5 H. Typical vertical fov: 22-31 degrees (longer lens = flatter faces, more compression); wides 28-34.
- Prefer **low side angles** (camera around chest height, 20-45 degrees off-axis) over top-down, and don't put the camera exactly on the axis of both characters (dead flat).

### 7.2 When to use which shot

| Shot | Use it for | Duration | Notes |
|---|---|---|---|
| **Hook shot** (medium two-shot or single) | The opening line. Frame 1 already shows the speaker and the first subtitle. | 2-3.5 s | Slow push, no fade-in from black. |
| **Wide / establishing** | Show where we are, **once**, early or at a reveal. | 2-3 s | Slow push-in. Look point above the surface; show the whole set in one frame. |
| **Far / very wide** | Isolation, awkward silence, the world being indifferent, a beat after something goes wrong. | 2-4 s, hold | Characters small in a big space, almost no motion. Great for a pause before a punchline. |
| **Two-shot** | Fast back-and-forth, both reactions at once, showing the contrast between the two characters. | 2-4 s | Keep both faces clear of the subtitle band, avoid tangents (an edge touching a head). |
| **Single on the speaker** | A line that needs attention (an emotional or important line). | 1.5-4 s | Cut to it **on the first word** of the line, not before. |
| **Reaction shot** (on the listener) | The listener's face after a line lands; the joke is often in the reaction, not the line. | 0.6-2 s | Cut **on the end of the line** or hold through the pause; the deadpan reaction should stay a half-beat longer than feels necessary. |
| **Close-up (face)** | The emotional beat, the sincere line, the punchline's setup. | 2-5 s | Slow push, smallest `fov` and shallow DOF (aperture up). Face stays out of the subtitle band. |
| **Insert** (hand, screen, prop) | Show a thing the story needs (the typed message, a notification). | 0.8-2 s | Never a long hold. Never a close-up of a phone/screen where you'd have to read it; the glow on the face tells the story. |
| **Pull-back on the last beat** | The closing line. | 2-4 s | Slow pull-out, then hard cut after the last word. |

Edit rules:
- **Cut on lines**: on a word start or just after a line ends, never mid-word, never on silence for no reason. Every cut needs a reason (new speaker, new information, a change of energy). If you cannot name the reason, delete the cut.
- Don't cut to the same framing twice in a row; change distance **or** angle by a clear margin (at least ~30% in distance or ~30 degrees).
- Rhythm: **short shots for fast banter, long shots for slow beats.** A 50 s reel is roughly 12-18 shots; some shots can last 5 s, most 2-4 s.
- Keep the **geography** consistent: the same character stays on the same side of the frame across cuts.
- Use `ap` (aperture) and `fov` to separate wides (deep focus, lower aperture) from close-ups (shallow focus). Always sharp on the speaker's face.

---

## 8. Characters, animation and the buggy-looking things to avoid

The characters, the set, the palette and the props are **yours to design** (ask the user for a one-line description or reference image of each character if they have one; otherwise invent). Constraints that apply no matter what you design:

- Build characters from **primitives in a hierarchy of named joints** (`root`, `pelvis`, `torso`, `head`, `armL/armR`, `legL/legR`, face parts). Pose = setting joint rotations from small key tables, never from physics. Expose their key joints as anchors.
- The "Claude" character must read as Claude at a glance (colour, shape, behaviour, name) without copying a logo asset. A simple memorable silhouette is better than a detailed one.
- Characters must be **readable at phone size**: strong silhouette, high contrast against the background, faces lit (a soft key + rim from behind + a faint fill; a face should never be a black hole).
- Everything is **low-poly with flat shading or very simple materials**; detail comes from light, DOF and bloom, not texture.

### 8.1 Motion rules (these are the ones that make a reel look buggy if you break them)

1. **Ease everything.** Motion between keyframes uses smootherstep (`key(t, [[t0,v0],[t1,v1],...])`, times **strictly increasing, never duplicated**; a duplicate time gives a divide-by-zero jump). No linear moves, no springs, no overshoot or wobble, no continuous morphing. Continuous life comes from **slow sines** (breathing: 0.2-0.3 Hz, tiny).
2. **Minimalist acting.** Move head, face and camera, plus **one slow looping motion** per character at a time. Don't animate the whole body to sell a line; large simultaneous motion looks like a glitch.
3. **No laughing, shaking or jittering.** No head shakes faster than ~2 Hz, nothing that oscillates with the voice envelope at full amplitude.
4. **Calm speech motion.** For a character with a mouth: jaw opening follows `env` through a short window (0.06-0.1 s), clamp the opening, never snap shut. For a character **without a mouth** (a mascot, a block): drive only a **tiny** squash/stretch (~1-2% of scale), eye height and a small head tilt from `envAvg(who, t, 0.3)` (long window), plus a bigger move **only** on 1-3 deliberate punchline words. Raw `env` on position/rotation = jitter. Always smooth.
5. **Ground every character in every frame.** Seat or feet touch the floor/ledge/chair at all times. If a leg swings, move the seat/pelvis with it. Compute the lowest world Y of the feet/seat each frame (`new THREE.Box3().setFromObject(...)`) and place the root so it sits exactly on the surface (do it in `setT`, after pose, before render). A character **floating above or sinking into the surface, even by a few hundredths of a unit, is the most noticeable bug.**
6. **Clearances are numbers, not eyeballs.** Limbs vs furniture, hands vs keyboard, head vs lamp. Check with Box3 across a sweep of times (every 0.1 s). If a character sits near a surface (keyboard, desk, ledge), give them real clearance (raise them, tilt the arms) rather than letting the geometry intersect on the cut-off frames.
7. **No coplanar surfaces.** Two faces at the same depth flicker (z-fighting). Offset by >= 0.03 world units, or use `polygonOffset`. Check close-ups: they reveal it.
8. **Props face the face, hands look like hands.** A phone/laptop/book held by a character: the screen points at the eyes, wrist angle plausible, hand = palm block + 4 wrapped finger blocks + thumb (even low-poly), prop parented to the hand (or placed from the hand each frame), never floating.
9. **Light from props is moderate.** A screen glow = point/spot lights parented to the prop, scaled by a `look` factor (how much the character is looking at it) with a gentle ramp. The glow on a face must be **dimmer than the screen**. Screens and logos in HDR bloom smear into blobs if they are much brighter than faces: check with `?bloomonly`. If the face washes out, dial it down.
10. **The camera never intersects anything** and never sees the back of the set: check the end of each push-in/pull-out, not only the middle.
11. **Subtitles never cover faces.** Subtitle band = ~31-43% from the top (see 4). 
12. **Frame 1 is a finished picture** (it becomes the cover frame): no black, no fade-in. The end: **hard cut** (see 9.4).

### 8.2 Checks to automate (write `check.mjs`, run it before any full render)

1. **Motion check**: with a `?norender` flag (skips the render passes, keeps pose + anchors), call `setT` for every frame time, collect the anchors (heads, hands, feet/seat, props, camera), and compute per-frame velocity and acceleration. **Flag** (a) teleports (a position delta > 3x the median speed, except on shot cuts), (b) spikes (acceleration > ~6x its 95th percentile), (c) reversals (direction flips within 3 frames at an amplitude > ~0.03 units: jitter). Fix each one by smoothing the driver (longer envelope window, fewer keys, add easing). Report counts, and **fix until none are left**.
2. **Grounding sweep**: at every 0.1 s, the lowest point of each seated/standing character vs its surface: |error| < 0.02 units.
3. **Penetration sweep**: Box3 intersection between each limb/prop and the furniture/keyboard/body across the sweep.
4. **Key-table lint**: strictly increasing times in every `key()` table, no NaN from `setT` at any frame (`isFinite` on every anchor), cut times fall on word starts/ends from the cue table.
5. **Subtitle lint**: every chunk <= 2 lines at 54 px in 880 px (measure with `getBoundingClientRect` for each chunk), min 0.7 s on screen, no overlap.
6. **Audio lint**: render the mix, analyse for clicks (a sample jump > 0.5 within 1 ms outside speech) and clipping; confirm no stem is older than `timeline.mjs` (compare mtimes in `make.mjs`).

### 8.3 Stills and contact sheet

`render.mjs stills 0.2,2.0,...` renders chosen times to `stills/t<time>.png`. Always include: **frame 1**, **every shot's first and last second**, every motion beat, the frame just before the end. Put them in one contact sheet (`ffmpeg -pattern_type glob -i 'stills/*.png' -vf "scale=270:-1,tile=8x2" sheet.png`) and **look at every one**: faces readable? nothing floating, intersecting, black or blown out? subtitle clear of faces? background not empty or messy? A good still-check is what saves the full render.

---

## 9. Writing the script (best practices, not content)

The format is **a conversation between a developer and a Claude character.** You write the lines yourself (or take them from the user). Our own scripts are not part of this file; write fresh ones.

1. **Open with "Claude" and a hook.** The first line should name the AI and put a situation, a claim or a confession on the table in one breath. By the end of the first line the viewer knows who is talking to whom and what the tension is.
2. **Each line answers or escalates the previous one.** The Claude character reacts to what the dev actually said (a literal reading, a dry correction, an unexpected agreement, an observation he didn't want), then the dev pushes back. Think ping-pong, not two monologues.
3. **Keep it relatable.** Pick moments any developer who works with an AI assistant has lived: late-night debugging, scope creep, tests that pass for the wrong reason, a confident wrong answer, rewriting a thing that worked, shipping, burnout, imposter feelings, naming things, the AI's habits (over-agreeing, over-explaining, over-engineering). Real specifics beat generic jokes. Don't explain the joke.
4. **Short lines.** Average <= 8 words, nothing over ~15. Total about 60-120 spoken words for about 50 s. Claude's lines are usually shorter than the dev's.
5. **Establish the two voices in the first 5-8 seconds and keep them consistent.** The contrast (energy vs flat, naive vs precise, emotional vs literal) is the engine of the comedy. Decide each character's persona in one line and stick to it.
6. **Vary the rhythm.** A series of quick exchanges, then **one held beat** (a 2-3 s silence or a slow close-up), then a snap. The punchline lands after the longest pause or immediately after a setup, never in the middle.
7. **Escalate in threes** when it is a list: the third item is the one that surprises.
8. **End on a funny line or a callback**, then **cut**. The last line should reframe the whole exchange or turn the first line around. No sign-off, no moral, no "thanks for watching", no end card.
9. **Give the performance direction.** Every take gets an audio tag; the contrast between two consecutive takes (flat then bright, small then loud) is part of the joke.
10. **Read it aloud in your head at speaking pace** and cut anything that does not earn its seconds. If the video is longer than about 55 s, cut a beat, don't speed it up.

---

## 10. Music and mix

Keep the sound **simple and clean**; the voices carry the video.

- **Room tone** (generated) as a bed at about **-47 dB mean**; optional outside ambience bed at about **-50 dB**; both looped with `acrossfade=d=2.5` (split the clip into 5 copies, chain the crossfades), high/low-passed (bed highpass ~45-60 Hz, lowpass ~4-6.5 kHz).
- **Music** is optional. If you make some: a slow synthesised pad (stacks of detuned sine/triangle, slow chord changes, gentle low-pass), mean level about **-37 dB** before the final normalisation, ducked a further ~6-8 dB under speech (use the speech mask from the cue table: fast attack, slow release). It is a bed, not a song. No copyrighted tracks.
- **Voice chain** per speaker (ffmpeg): `highpass=f=85, lowpass=f=9500, [optional gentle treble], acompressor=threshold=0.045:ratio=2.6:attack=8:release=140:makeup=2`, then a very small room send (`aecho=0.8:0.5:41|79|133:0.26|0.18|0.1, lowpass=f=3400, volume=0.6` mixed back at ~0.6) and a slight stereo offset between the two speakers (0.56/0.44 versus 0.44/0.56). Sounds that come from inside the set (a phone, a notification) get a narrower band-pass and quieter level. Match stem levels by **measured** mean volume (`volumedetect`) instead of guessing.
- Final: `amix=inputs=N:normalize=0:duration=longest`, trim to `TOTAL`, `afade` in 0.08 s, fade-out matching the visual fade, `apad=whole_dur=TOTAL`; then **two-pass `loudnorm`** to **I=-14 LUFS, TP=-1.5, LRA=9** (`print_format=json` pass, then `measured_*` + `offset` + `linear=true`) followed by `alimiter=limit=0.84`. Verify with `ebur128=peak=true`: integrated ~ -14.0, true peak <= -1.5.
- **Rebuild all stems whenever the timeline changes** (make.mjs). Delete or regenerate any foley (footsteps, ticks) that are timed from `MARK`; a stale tick track is heard as a mysterious click at an odd moment.

---

## 11. Rendering

`render.mjs` modes: `stills <times>`, `full [from] [to]`, both through `cdp.mjs`, serving the project folder from a local static server on `127.0.0.1:0` (no-store cache header) and opening `/film.html`.

- Open **one** browser. Wait for `window.__ready === true` (90 s timeout, dump the console on failure).
- Per frame: `await api.eval("window.setT(" + (f/30).toFixed(4) + ")")`, then `Page.captureScreenshot {format:'jpeg', quality:92}`, write `frames/f0000.jpg` (zero-padded to 4 digits).
- **Resumable**: skip frames whose file exists and is > 8 KB. Take an optional `[from] [to]` range so one tool call renders a chunk (~400-600 frames; adjust so the call is < 8 minutes), then call again with the next range until all frames exist.
- **Blank-frame guard**: if a screenshot is under ~20 KB (and the frame is not in the final fade), the WebGL context was lost: close the browser, relaunch, retry the frame (up to 3 attempts). Re-verify after the render: any frame file under 8 KB (except the last few frames) = re-render that range.
- Expect about 0.15-0.3 s per frame on a mid-range GPU (a 50 s film = 1500 frames = 4-7 minutes in total). **Do not "speed it up" with more workers** (rule 1).
- Don't change GPU/CPU mode mid-render. Finish a film in the mode it began in.

## 12. Mux, verify, deliver

```
ffmpeg -nostdin -y -framerate 30 -i frames/f%04d.jpg -i mix.wav -c:v libx264 -crf 25 -preset medium -maxrate 14M -bufsize 28M -pix_fmt yuv420p -profile:v high -c:a aac -b:a 192k -shortest -movflags +faststart "<Title>.mp4"
```

- **Audio-only fix later:** reuse the video stream: `-i old.mp4 -i mix.wav -map 0:v:0 -map 1:a:0 -c:v copy -c:a aac -b:a 192k -shortest -movflags +faststart new.mp4` (no re-render).
- **Verify before you tell the user it is done:**
  - `ffprobe`: 1080x1920, 30 fps, `yuv420p`, duration within one frame of `TOTAL`, audio duration within one frame of video.
  - Frame 1 is **not black** (`signalstats` YAVG clearly above 0) and equals the planned hook frame.
  - Loudness: integrated about -14 LUFS, true peak <= -1.5 dBTP.
  - Pull ~12 frames across the film from the **final mp4** (`ffmpeg -ss <t> -i out.mp4 -frames:v 1`) into a contact sheet and look at it once more (compression artifacts, blown-out glow, subtitles).
  - Spot-check a few cut points for audible clicks.
- **Deliver** to a normal user folder (for example `Videos/<project name>/`), never overwrite an earlier version (use `v2`, `v3`), **open the file in the default player** for the user, and give them a **short report**: title, duration, shot count, what is deliberately choice (voices, tags, takes with alternates), what you would push further, and **honestly** any known flaws you saw. Don't post it anywhere.

---

## 13. Quick checklist (copy this into your plan)

- [ ] Requirements, GPU probe passes (no SwiftShader), key present but never printed.
- [ ] Brief in 3-5 lines: topic, length, two characters, set, two voices.
- [ ] Script: Claude + hook first line, ping-pong, relatable, funny last line.
- [ ] Takes generated (2-3 alternates for key lines), wav + word timings printed.
- [ ] `timeline.mjs`: cues trimmed, gaps for rhythm, `END_T`, shots with reasons.
- [ ] `build-film.mjs`: buses, envelopes, subtitle chunks; `make.mjs` runs everything in order.
- [ ] `scene.js`: stateless `setT`, post chain, anchors, grounding, camera, subtitles.
- [ ] Stills contact sheet fixed; `check.mjs` (motion, grounding, penetration, lint) clean.
- [ ] Mix: levels measured, -14 LUFS, TP <= -1.5, no stale stems.
- [ ] Full render in foreground chunks, one browser, low priority; all frames > 8 KB.
- [ ] Mux, verify (ffprobe, frame 1, loudness, final contact sheet), deliver, short honest report.
