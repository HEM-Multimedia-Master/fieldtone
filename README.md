# FIELDTONE

A fullscreen, generative browser instrument: six synthesizer pads, a draggable
ADSR envelope, and independent touch filters. Built with HTML, CSS, JavaScript,
and Web Audio in a single dependency-free `index.html`. No account, build step,
external service, or microphone is required.

**[Play FIELDTONE](https://hem-multimedia-master.github.io/fieldtone/)**

## Playing

Hold a pad or a number key (`1`–`6`) to play; release to fade out. Short taps
still play the envelope's minimum sustain duration. Multiple touches can hold
and filter different pads simultaneously. Audio starts after the first playing
gesture, as required by browser autoplay policies.

The wireframe interface fills the viewport without outer borders or padding.
Six pads occupy the upper 85%; the bottom 15% contains sound-set navigation
and the ADSR editor. Narrow screens use two columns of three pads; wider
screens use three columns of two.

### Sound sets

The green-bordered **R** button generates a new set of voices, pitches,
effects, stereo positions, and signal connections. **< / >** restore earlier
or later sets; **SET n / total** shows the position. Controls are available on
both touch devices and desktop.

History belongs to the open page session. Reloading starts over. Generating
from an earlier set replaces its future entries. Each pad's filter settings
stay with its set; the ADSR envelope and master volume are shared across sets.

### Envelope

Drag the **A, D, S, R** points horizontally to move their time boundaries:

- **Attack:** rise to the peak.
- **Decay:** fall to 60% of the peak.
- **Sustain:** minimum time at that level; holding a pad extends it.
- **Release:** fade to silence after release and the minimum sustain.

Each stage lasts **0.005–5 seconds**. Points cannot cross. Moving A, D, or S
redistributes time between neighbouring stages while keeping later boundaries
fixed; moving R changes the release duration. The drawing refits after dragging.

Drag **A or R vertically** to switch the interpolation curve: **up =
logarithmic**, **middle = linear**, **down = power**. The visible and audible
curves use the same interpolation functions. These are discrete curve modes,
not an amplitude control. Decay remains linear and sustain amplitude stays fixed.

For keyboard editing, Tab to a point and use **Left / Right** to change time
by 0.01 seconds (**Shift** changes it by 0.1 seconds). On A or R,
**Up / Down** changes the curve. These arrows edit the focused point instead
of navigating sets or changing volume.

Attack, decay, and minimum sustain are captured when a note starts. Release
time and curve are read when the note is released, so you can adjust R while
holding a sound.

### Filter gesture

Hold a pad, then drag anywhere across the **entire window**, including outside
the original pad:

- **Vertical:** low-pass cutoff frequency, logarithmically mapped from **40 Hz
  at the bottom** to **16 kHz at the top** (limited by the audio sample rate).
- **Horizontal:** filter **Q**, from **0.2 at the left** to **18 at the right**.

Q controls the resonance around the cutoff frequency. The crosshair and pad
readout display the values. Different fingers control their originating pads
independently, and filter changes are smoothed to avoid abrupt jumps.

### Cheatsheet

| Action | Control |
| --- | --- |
| Hold / release a voice | Pad or keys `1`–`6` |
| Generate a sound set | Green `R` button or `R` key |
| Previous set | `<`, `←`, or `Space` |
| Next set | `>`, `→`, or `Shift+Space` |
| Edit envelope times | Drag A/D/S/R horizontally |
| Change attack / release curve | Drag A/R vertically |
| Filter a sound | Hold a pad and drag across the window |
| Change volume | `↑` / `↓`, in 20% steps with a 500 ms fade |
| Open help | Bottom-left corner button or **`I`**, not L |
| Close help | Close button or `Esc` |

Master volume starts at **80%**; its six levels are 0, 20, 40, 60, 80, and
100%. A temporary notice reports changes. There is no top HUD, volume slider,
output meter, or dynamic-pan switch. The automatic stereo sweep is enabled.
Touch-only master-volume editing remains a possible improvement.

Held notes are released when the window loses focus, the page becomes hidden,
help opens, or the sound set changes. Cancelled touches release immediately.
Up to 24 voices can overlap; older voices are cleared at that limit. A master
compressor limits peaks from layered voices and resonant filters.

## Run and deploy

Open `index.html` in a modern browser with Web Audio support. For local HTTP
preview, run this from the repository and open `http://localhost:8080`:

```sh
python3 -m http.server 8080
```

GitHub Pages publishes the root of the `main` branch. Updating `index.html`
on `main` updates the instrument at the play link above. No package install or
compilation is required. No remote scripts, fonts, or audio files are loaded.

## Concepts applied

### HTML: structure and accessibility

`<main>` contains the decorative patch-cable SVG and the six-pad grid. Each
pad is a native button with a spoken sound description and number shortcut.
A `<nav>` groups sound-set controls; an `<output>` announces the set counter.
The envelope points are focusable buttons with slider roles, accessible time
values, and keyboard alternatives to dragging. A native `<dialog>` provides
help, focus management, and Escape dismissal.

The viewport metadata supports mobile sizing; `100dvh` follows the available
browser height. The page uses no sensor permissions or microphone tracks.

### CSS: layout and visual feedback

`.app` uses CSS Grid with `minmax(0, 1fr) 15%` rows. The bottom panel has a
compact navigation column and a flexible envelope editor. A media query at
650 px changes the pad grid from 3×2 to 2×3. A short-height media query reduces
envelope labels for landscape phones. Waveforms, cables, divider lines, and
point outlines provide the wireframe aesthetic; pads have no randomized
rotation. Visible keyboard focus and reduced-motion styles are included.

### JavaScript: state, rendering, and gestures

The inline module is organized into math helpers, the envelope editor, and
`initFieldtone()`. It uses a **state → render → interaction** pattern:

- `state.pads` and `state.links` describe the current sound set and routing.
- `state.history` and `state.historyIndex` track session snapshots.
- `makePatch()` generates a set; `showPatch()` restores and renders one.
- `renderPads()` and `renderWires()` project the same patch data visually.
- `setupEnvelopeEditor()` handles boundary dragging and keyboard edits.
- `moveEnvelopePoint()` enforces stage-duration and boundary constraints.
- `curveValue()` and `curveSamples()` share interpolation between SVG and audio.
- `filterFromPosition()` maps viewport coordinates to cutoff and Q.
- `beginGate()` / `endGate()` pair pointer or keyboard presses with releases.
- Pointer capture keeps a gesture attached to its original pad across the window.
- `ensureAudio()` lazily initializes Web Audio after a playing gesture.
- `releaseAll()` handles focus changes, dialogs, and sound-set changes safely.

`envelopeSettings` holds four durations plus attack and release curve modes.
The gate map tracks independently held inputs; `activeVoices` tracks voices
and effect tails until their nodes are disconnected. Abortable listeners,
resize observers, and timer cleanup keep lifecycle handling explicit.

### Audio: synthesis and signal graph

The six voice families are **additive, subtractive, FM, ring modulation,
supersaw, and pluck**. Each generated pad carries its pitch, stereo position,
delay/reverb settings, and gesture-filter values. The envelope peaks at 0.55
and decays to 0.33 (60%). Its release ends at zero.

Oscillators and synthesis filters feed an envelope gain, then a per-voice
low-pass gesture filter and stereo panner. Dry, delay, and reverb signals feed
pad buses. Randomized, quieter links connect buses in a directed signal graph;
the SVG cables visualize those connections. The master gain and compressor
feed the speakers. Automatic pan moves from the generated position toward its
opposite over attack plus release time.

Volume transitions cancel pending gain automation, hold the estimated current
level, and ramp to the requested level over half a second. Envelope audio
curves use `setValueCurveAtTime()` with the same mathematical functions as the
drawing. Releasing an input schedules the fade and oscillator stop; holding it
keeps sustain active. Node cleanup runs after the effects tail.

## Hyperphones

These principles describe the current instrument and its remaining limitations:

1. **Access:** HTTPS play link, no account, and no external runtime dependencies.
2. **Ubiquity:** Responsive pads, set navigation, and ADSR editing work with
   touch or mouse. Keyboard controls are also available. Master volume currently
   requires a keyboard or the device's own volume controls.
3. **Gesture:** Hold/release plays a voice; full-window dragging filters it;
   envelope points edit timing and curves. Multiple touches are independent.
4. **Feedback:** Active waveforms, filter readouts, a crosshair, the envelope
   graph, set counter, and status notices expose changes and audio-start errors.
5. **Control:** Audio requires a playing gesture. Release, cancellation,
   focus loss, and opening help end held notes. There is no microphone input
   and no dedicated global stop button yet.
6. **Decision:** Code randomizes patches; players choose sets, notes, timings,
   curves, filters, and volume. No learned behavior is used.
7. **Trace:** Sets and their filter settings can be revisited during a session.
   Recording, export, seeded reproduction, and history persistence are not implemented.
8. **Maintenance:** One readable source file, no build toolchain, and no remote
   API dependency. Playback still depends on browser Web Audio support.

## Verification

When changing the instrument, check:

- Desktop, portrait phone, tablet, and short landscape layouts: six pads,
  full viewport width, a 15% bottom panel, and reachable envelope points.
- R and previous/next navigation, disabled history boundaries, and branching
  history when generating from an earlier set.
- Horizontal envelope edits, all three A/R curves, and keyboard point editing
  without accidentally navigating sets or changing volume.
- Real audio output, long holds, release tails, full-window filter dragging,
  and two simultaneous touches controlling different pads.
- Release on blur, help opening, and touch cancellation; help focus and Escape.
- No browser errors or overlapping Web Audio automation errors.
