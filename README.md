# FIELDTONE

A fullscreen, generative browser instrument: six synthesizer pads, a draggable
ADSR envelope, manual FM cables, and independent touch filters. Built with HTML, CSS, JavaScript,
and Web Audio in a single dependency-free `index.html`. No account, build step,
external service, or microphone is required.

**[Play FIELDTONE](https://hem-multimedia-master.github.io/fieldtone/)** · [Play in the zztt Hyperphones suite](https://zztt.org/fieldtone/)

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

The square, green-bordered **R** button generates a new set of voices, pitches,
effects and stereo positions. FM cables are created only by the player, never randomly. **< / >** restore earlier
or later sets; **SET n / total** shows the position. **R, < and > always remain
square**, including on mobile and tablet. Controls are available on both touch
devices and desktop.

Generating or revisiting a set preserves notes that are already sounding,
including their original routing, delay, reverb, and scheduled release.
Held keys and touches still release their original voices when you let go.
New notes use the newly selected set. Each sounding set keeps its own effect
graph; it is reclaimed after its last voice and effects tail finish. The
existing 24-voice limit still applies: a new voice steals the oldest one only
when that limit is reached.

History belongs to the open page session. Reloading starts over. Generating
from an earlier set replaces its future entries. Each pad's filter settings
stay with its set; the ADSR envelope and master volume are shared across sets.

### Crossmix while holding

Keep a pad pressed and drag into another pad. After dwelling inside it for
**180 ms**, the voices crossfade over **one second** with complementary
sine/cosine gain curves. Briefly passing through a pad does not switch voices.
Continue dragging to transition again; release the pointer or finger to end
the current voice through its release envelope. Nothing stays latched after
release. Each simultaneous touch has its own transition state.

### Manual FM cables

New sets always start **without cables**. Each pad has an **OUT** terminal on the right and an **FM IN** terminal on the left. Drag a free OUT circle to a free FM IN circle to make a connection. The source sound modulates the frequency of the destination's carrier oscillators; play or hold both pads to hear the interaction. This does not mix a copy into the destination's audio output. The FM voice also retains its own internal modulator/carrier synthesis.

Connections use a gentle, frequency-relative depth (25% before the bounded modulation signal). Every endpoint is an outlined circle with its own touch target:

- Drag a connected **FM IN** endpoint to another inlet to change its carrier pad.
- Drag a connected **OUT** endpoint to another output to change its modulator pad.
- Drop a connected endpoint on a pad's **body/centre** to delete that cable. Dropping outside the pads or cancelling leaves it unchanged.
- For keyboard control, Tab to a terminal, press **Enter**, then focus the compatible terminal and press **Enter** again. **Delete / Backspace** removes its cable; **Esc** cancels selection.

Connections are stored with their set and can be edited while notes sound. Duplicate cables, self-connections, and direct feedback cycles are rejected. Filters, crossmix and other fingers remain playable while another finger edits a cable.

### Envelope

Drag the **A, D, S, R** points horizontally to move their time boundaries:

- **Attack:** rise to the peak.
- **Decay:** fall to 60% of the peak.
- **Sustain:** minimum time at that level; holding a pad extends it.
- **Release:** fade to silence after release and the minimum sustain.

**Attack and release reach 10 seconds**; decay and sustain reach 5 seconds.
Every stage has a 0.005-second minimum. Durations are independent: changing
one moves later boundaries without shortening its neighbours. The drawing
fits the current total duration, including long attacks and releases.

Drag **A vertically** to set master volume. A muted gray **ppp · pp · p · mf · f**
scale appears at the bottom-left inside the graph, in small Times italic,
with the current dynamic highlighted. The graph's peak follows the volume.

Hold **Shift while dragging A vertically**, or drag **R vertically**, to
switch interpolation: **up = logarithmic**, **middle = linear**,
**down = power**. The visible and audible curves use the same interpolation
functions. Decay remains linear; sustain stays at 60% of the peak.

For keyboard editing, Tab to a point and use **Left / Right** to change time
by 0.01 seconds (**Shift** changes it by 0.1 seconds). **Home / End** sets the
stage to its minimum / maximum. On A, **Up / Down** edits volume and
**Shift+Up / Down** changes the attack curve. On R, **Up / Down** changes its
curve. Focused-point keys do not navigate sound sets.

Attack, decay, and minimum sustain are captured when a note starts. Release
time and curve are read when the note is released, so you can adjust R while
holding a sound.

### Pad-local filter gestures

Hold a pad, then drag **inside its square**. Every pad maps its own full filter range:

- **Vertical:** low-pass cutoff frequency, logarithmically mapped from **40 Hz
  at the bottom of the pad** to **16 kHz at its top** (limited by the audio sample rate).
- **Horizontal:** filter **Q**, from **0.2 at the left edge of the pad** to **18 at its right edge**.

Q controls the resonance around the cutoff frequency. Each touch has its own numbered, coloured cursor and filter readout. Second and third touches work independently, including on the same pad. Releasing one finger removes only that cursor. Each pointer edits its own voice filter; the pad readout shows the most recent edit. When leaving a pad, its mapping clamps at the edge. After the crossfade dwell, that cursor uses the destination pad’s local coordinates. Filter changes are smoothed to avoid abrupt jumps.

### Cheatsheet

| Action | Control |
| --- | --- |
| Hold / release a voice | Pad or keys `1`–`6` |
| Generate a sound set | Green `R` button or `R` key |
| Previous set | `<`, `←`, or `Space` |
| Next set | `>`, `→`, or `Shift+Space` |
| Edit envelope times | Drag A/D/S/R horizontally |
| Change attack / release curve | Shift+drag A vertically / drag R vertically |
| Crossmix | Hold, drag into another pad, and stay inside for 180 ms |
| Filter a sound | Hold a pad and drag inside its square |
| Change volume | Drag A vertically, or `↑` / `↓` in 20% steps with a 500 ms fade |
| Open help | Bottom-left corner button or **`I`**, not L |
| Close help | Close button or `Esc` |

Master volume starts at **80%**; its six levels are 0, 20, 40, 60, 80, and
100%. The graph highlights the current dynamic; keyboard changes also show a temporary notice. There is no top HUD, volume slider,
output meter, or dynamic-pan switch. The automatic stereo sweep is enabled.
Touch and mouse can edit master volume directly through the attack point.

Held notes are released when the window loses focus, the page becomes hidden,
or help opens. Changing sound sets does not stop them. Cancelled touches release immediately.
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

- `state.pads`, `state.links`, and `state.patch` describe the current sound set and routing.
- `state.history` and `state.historyIndex` track session snapshots.
- `makePatch()` generates a set; `showPatch()` restores and renders one.
- `renderPads()` and `renderWires()` project the same patch data visually.
- `setupEnvelopeEditor()` handles boundary dragging and keyboard edits.
- `moveEnvelopePoint()` enforces stage-duration and boundary constraints.
- `curveValue()` and `curveSamples()` share interpolation between SVG and audio.
- `filterFromPad()` maps each pad’s local coordinates to cutoff and Q.
- `beginGate()` / `endGate()` pair pointer or keyboard presses with releases.
- Pointer capture keeps a gesture attached to its original pad across the window.
- `ensureAudio()` lazily initializes Web Audio after a playing gesture.
- `graphFor()` retains separate audio/effects and FM modulation nodes for each sounding set.
- `createFmRouter()` attaches manual FM links to each live carrier frequency and cleans them up with the voice.
- `setupPatching()` provides pointer/touch and keyboard cable creation, reconnection, and deletion.
- `connectFm()` validates manual graphs without duplicates, self-connections, or feedback cycles.
- `disposeUnusedGraphs()` frees retired routing only after its last voice and tail finish.
- `releaseAll()` handles focus changes and dialogs safely; set changes preserve active voices.

`envelopeSettings` holds four durations plus attack and release curve modes. Master volume controls the height of the graph, so the displayed envelope reflects the output level.
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
pad buses, each feeding the master directly. Manual cables connect a separate,
bounded modulation signal (after the source voice's envelope/filter/crossfade,
before stereo and effects) through a depth gain to the destination carrier's
`OscillatorNode.frequency` AudioParam. This is audio-rate FM; no cable copies
audio between pad output buses. Internal FM/ring modulators are not treated
as external carrier targets. The SVG cables visualize the manual FM graph. The master gain and compressor
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
   touch or mouse. Keyboard controls are also available; A provides touch-accessible master volume.
3. **Gesture:** Hold/release plays a voice; pad-local dragging filters it;
   envelope points edit timing, volume, and curves; held drags crossmix voices. Multiple touches are independent.
4. **Feedback:** Active waveforms, filter readouts, independent numbered cursors, the envelope
   graph, set counter, and status notices expose changes and audio-start errors.
5. **Control:** Audio requires a playing gesture. Release, cancellation,
   focus loss, and opening help end held notes. There is no microphone input
   and no dedicated global stop button yet.
6. **Decision:** Code randomizes voices and their parameters; players configure all FM cables; players choose sets, notes, timings,
   curves, filters, and volume. No learned behavior is used.
7. **Trace:** Sets and their filter settings can be revisited during a session.
   Recording, export, seeded reproduction, and history persistence are not implemented.
8. **Maintenance:** One readable source file, no build toolchain, and no remote
   API dependency. Playback still depends on browser Web Audio support.

## Verification

When changing the instrument, check:

- Manual OUT → FM IN connections, reconnection of either endpoint, body-drop deletion, cancellation, keyboard edits, and refusal of duplicate/feedback cables. Confirm that R starts with no cables and history restores manual links.
- Desktop, portrait phone, tablet, and short landscape layouts: six pads,
  full viewport width, a 15% bottom panel, square R / < / > buttons, and reachable envelope points/patch terminals.
- Hold a key or touch while generating with R; overlap old/new voices, verify a 10-second release survives set changes, and release the original input after its pad disappears.
- R and previous/next navigation, disabled history boundaries, and branching
  history when generating from an earlier set.
- Independent horizontal durations up to 10 seconds for A/R, A vertical volume, all three A/R curves, and keyboard point editing
  without accidentally navigating sets or changing volume.
- Real audio output, long holds, release tails, pad-local filter dragging,
  three simultaneous touches with independent cursors and filters, including two on one pad, and held crossmix transitions between pads.
- Release on blur, help opening, and touch cancellation; help focus and Escape.
- No browser errors or overlapping Web Audio automation errors.
