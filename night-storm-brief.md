# Night Storm — Project Brief

An original, real-time thunderstorm scene that runs in the browser. Lightning strikes when you **click** or when the **microphone hears a sudden loud sound** (a clap, a shout, a desk slap). Everything is written from scratch: no engine, no libraries, no copied shader code.

This brief has two parts:
1. **Reference study.** What the Shadertoy shader I found (llB3W1) actually does, so we understand the techniques. **None of its code goes into this project.**
2. **Build spec.** What to build, how, in what order, and how we'll know it works.

---

## Part 1: Reference study (Shadertoy llB3W1)

### What it is
It's an audio-reactive fractal visualizer. It draws three overlapping **Julia set fractals** whose shapes morph over time and pulse with the music or microphone.

### How it works, piece by piece

**`fractal(p, point)`: a cubic Julia set**
- Each pixel becomes a point `p` on the 2D plane, treated as a complex number.
- The loop applies `z = z³ + c` repeatedly. The first line squares `p`, and the second multiplies the result by the original `r`, which gives a cube. Then it adds the `seed` (that's `c`).
- If the point escapes (`length(p) > 2`), the function returns how many steps it took, and that count picks the color. Points that never escape return 0, which comes out black.
- `seed` (the `c` value) is nudged by `point`. Small changes to `c` reshape the whole fractal, and that's what makes it "breathe."

**`color(i)`: a sine palette**
- It turns the step count into RGB with `sin(f*2)`, `sin(f*3)` and `abs(sin(f*7))`. The red and green channels can go negative. That's a quick hack, not a deliberate palette.

**`sampleMusicA()`: audio input**
- On Shadertoy, a microphone or music input in `iChannel0` arrives as a 512×2 texture. The row at `y≈0.25` is the **frequency spectrum (FFT)** and the row at `y≈0.75` is the **waveform**.
- This function samples two low-to-mid frequency bins and averages them. The result is effectively a "bass pulse" value.

**`mainImage`: the composition**
- It builds three fractal layers. The main one uses `position`. A mirrored one uses `pos2`, with flipped coordinates and a different scale. A zoomed one uses `position/1.6`. Each layer animates `c` with `sin`/`cos` of `iTime`, plus the audio pulse.
- `t3` samples the spectrum at `x = distance from center`. Low frequencies drive the middle of the screen and high frequencies drive the edges, which produces audio-reactive rings.
- The final mix, `c/t3 + c*t3 + layer2 + layer3`, is not clamped. The division by `t3` blows colors out to pure white or saturated values. It looks intense, but it's uncontrolled.

### Observations
- `mouse` and `uv` are computed but **never used**, so it's dead code.
- It runs up to **450 fractal iterations per pixel** (3 × 150), which is expensive on laptop GPUs.
- It has no concept of a "trigger." The audio continuously modulates the image, and nothing is event-based.

### What we take from it (ideas only)
- Reading microphone frequency/volume data and feeding it to a shader as a number.
- Driving shader parameters from time and audio together.
- Layering several procedural passes into one image.

### What we do differently
- **Event-based triggers** (a strike happens) instead of continuous wobble.
- A **controlled, physically plausible look**: tone mapping, no blown-out division.
- Our own scene, our own math, our own code.

---

## Part 2: Build spec

### Goal
A single `index.html` (inline JS and GLSL, no build step, no dependencies) showing a **dark night thunderstorm**:
- Heavy clouds drifting slowly, barely visible until lightning lights them up
- Falling rain
- A dark landscape silhouette (hills and a few trees or a distant town's lights)
- **Lightning:** a branching bolt, a sky flash, clouds lit from inside, then a delayed **thunder** sound
- Triggers: **mouse click** (the bolt lands near the click position) and **microphone** (a sudden loud sound causes a strike, and louder means a bigger strike)

### Tech choices (and why)
- **WebGL2, not WebGPU.** It works on almost every browser and laptop GPU without forcing the dedicated GPU. WebGPU would rule out a lot of devices for no visual gain at this scale.
- **One full-screen fragment shader** for the sky, clouds, rain and ground, plus a small JS layer for input, audio and bolt generation.
- **Web Audio API** for both microphone analysis and synthesized thunder (no audio files).

### Architecture
```
index.html
├── <canvas> full screen
├── Fragment shader (GLSL ES 3.0)
│   ├── sky gradient + faint stars through cloud gaps
│   ├── clouds: fbm (fractal Brownian motion) noise, 4–5 octaves, slow drift
│   ├── lightning: glow from distance-to-line-segments (segments passed as uniforms)
│   ├── cloud illumination: brighten clouds near the bolt + global flash
│   ├── ground silhouette: 1D noise ridge line, darker foreground layer
│   ├── rain: 2–3 layers of hashed diagonal streaks, parallax by layer
│   └── tone mapping (ACES or Reinhard) + subtle vignette + film grain
└── JavaScript
    ├── WebGL2 setup, resize handling, render loop (requestAnimationFrame)
    ├── Bolt generator (CPU): recursive midpoint displacement with random branches
    ├── Strike manager: flash envelope, cooldown, queued thunder
    ├── Mic input: getUserMedia → AnalyserNode → onset detection
    ├── Thunder synth: noise buffer → filters → envelopes, delayed by "distance"
    └── Small settings panel (toggle with H)
```

### Lightning details
- **Bolt shape (JS):** start at a random point in the clouds (or above the click x), end at the ground near the click/target. Recursively split each segment at its midpoint and offset the midpoint sideways by a random amount that halves at each level (5–6 levels). At some splits, spawn a shorter, dimmer branch.
- Pass up to ~64 segments to the shader as a `uniform vec4 uSeg[64]` (xy = start, zw = end) plus a per-segment brightness.
- **Shader glow:** for each pixel, take the minimum distance to the segments. Combine a sharp core (`exp(-d * big)`) with a wide soft halo (`1/(1 + d*k)`), tinted blue-white.
- **Flash envelope:** real lightning flickers. Use 2–4 quick pulses over about 300–600 ms (random gaps), then fade. `uFlash` drives the overall sky and cloud brightness and `uBolt` drives the bolt visibility.
- **Strength (0–1)** scales bolt brightness, number of branches, flash intensity and thunder loudness. A click gives a fixed strength, which can also vary by click duration. The mic maps onset loudness to strength.

### Microphone details
- Start the mic only after the user clicks a **"Enable mic"** button. Browsers require a user gesture before audio can start, and asking on page load feels hostile.
- Compute RMS volume from `getFloatTimeDomainData` every frame.
- **Onset detection, not a fixed threshold.** Keep a slow running average of background noise and trigger when `current RMS > background × sensitivity` AND it's above a minimum floor. A fixed threshold breaks in a noisy room or with a quiet laptop mic.
- **Cooldown** of about 1.5 s between strikes so one clap doesn't fire five bolts.
- **Feedback loop problem:** the thunder sound coming from the speakers will be heard by the mic and trigger another strike, endlessly. Fix: ignore the mic while thunder is playing (plus about 300 ms), and offer a "Headphones mode" toggle that disables that suppression.
- Show a small live mic-level meter with the threshold line drawn on it, so the user can tune sensitivity.

### Thunder sound (synthesized)
- **Delay:** pick a random "distance" per strike. Thunder plays `distance × ~0.3 s` after the flash (sound travels about 3 s per km). Close strikes get almost no delay.
- **Close strike:** a sharp crack (a short burst of high-passed noise), then a loud rumble.
- **Distant strike:** only a low rumble (noise through a lowpass filter around 100–300 Hz) with a long, uneven decay of 3–6 s. Modulate gain with slow random wobbles so it "rolls."
- Optional: a quiet looping rain-noise bed, with a mute toggle (M).

### Controls
| Input | Action |
|---|---|
| Click | Strike near the cursor's x position |
| Enable mic button | Turns on mic triggering |
| H | Show/hide settings panel |
| M | Mute all audio |
| Panel | Mic sensitivity, rain amount, cloud density, headphones mode, render scale |

### Performance targets
- 60 FPS on a typical laptop integrated GPU at 1080p.
- A render-scale option (0.5–1.0) that renders to a smaller canvas and upscales.
- Keep fbm at ≤5 octaves. Skip the segment loop for pixels far from the bolt's bounding box.

### Build phases (do one at a time, verify, then continue)
1. **Skeleton:** an HTML page, WebGL2 context, a full-screen shader showing a dark gradient sky, a resize handler and an FPS counter.
2. **Atmosphere:** fbm clouds, ground silhouette, rain layers and tone mapping. Static mood check: does it feel like a dark stormy night?
3. **Click lightning:** the bolt generator, segment uniforms, glow, flash flicker and cloud illumination.
4. **Thunder audio:** the synth with distance delay, plus mute.
5. **Microphone trigger:** the enable button, level meter, onset detection, cooldown and feedback suppression.
6. **Polish:** the settings panel, render scale, vignette/grain, and a short on-screen hint ("Click or clap to summon lightning").

### Acceptance criteria
- Opens and runs in Chrome and Edge on Windows with no console errors.
- The scene is moody and dark between strikes, not a flat blue screen.
- Every click produces a different bolt shape that lands near the cursor.
- A clap triggers a strike. Normal talking at a quiet sensitivity setting does not. The speakers' own thunder never re-triggers a strike.
- If mic permission is denied, the app keeps working with click only and shows a clear message.
- Holds steady FPS at the default settings on integrated graphics.

### How to run locally (Windows)
The mic API only works on `https://` or `localhost`, not by double-clicking the file (`file://`). From the project folder in a terminal:
```
python -m http.server 8000
```
Then open `http://localhost:8000` in Chrome. (`python -m http.server` starts a tiny web server serving the current folder. `8000` is the port number.)

### Instructions for Claude Code
- Write all code from scratch. **Do not copy code from Shadertoy or any other shader.** The reference study above is for understanding concepts only.
- Work phase by phase. After each phase, stop and explain what you built and how each new part works in plain language, because I'm learning graphics programming. Then wait for my go-ahead.
- Comment the shader generously: what each function does and why the math works.
- Keep it to one `index.html` file unless it grows past about 1,000 lines. If it does, ask before splitting.
- Use `git` and commit after each working phase with a clear message.

### Stretch goals (only after everything above works)
- Wet ground that reflects the lightning flash
- Bolt hits that briefly light up the trees or house silhouettes
- Strike position from mic stereo balance (left/right), if a stereo mic is available
- Deploy to Vercel or GitHub Pages and add it to the portfolio at mishrakeshav.in
