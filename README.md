# FIELDTONE

## Synopsis

FIELDTONE is a fullscreen, generative browser instrument. Six touchable pads
trigger distinct synthesizer voices; each session randomizes their pitches,
effects, and signal connections. Play a pad by clicking or tapping it, or use
the visible `1`–`6` shortcuts on a keyboard. Press `R` to generate another
patch. The master slider controls output volume, and a small meter visualizes
the audio output. Generated patches are kept in a session history: use `Space`
or `←` to go back, and `Shift`+`Space` or `→` to go forward.

The instrument is built from one HTML page, CSS, and the browser's Web Audio
API. No account or external service is needed to play.

# Cheatsheet

| Action | How |
| --- | --- |
| Play a voice | Click/tap one of the six pads, or press its number, `1`–`6` |
| Generate a new patch | Click the HUD `R` button or press `R` |
| Revisit a patch | `Space` or `←` goes back; `Shift`+`Space` or `→` goes forward |
| Set output volume | Drag the master slider in the upper-left HUD |
| Change volume by a dynamic step | Press `↑` or `↓`; each change fades linearly over 500 ms |
| Shape a note | Adjust `A`, `D`, `S`, and `R` in the upper-left HUD; each stage ranges from 0 to 5 seconds |
| Toggle dynamic panic | Use the `DP` switch; on by default, it pans each note to the opposite side over attack + release (1 second if both are 0) |
| Start audio | Play a pad; browsers require a user gesture before audio starts |

Volume has six levels separated by five equal steps. The current level is
shown using only its dynamic mark, in bold italic serif type:

| Musical dynamic | Volume |
| --- | ---: |
| `n` | 0% |
| `ppp` | 20% |
| `pp` | 40% |
| `p` | 60% |
| `mf` | 80% |
| `f` | 100% |

# Concepts applied

## HTML (structure)

HTML gives the interface meaning and establishes parent-child relationships.
The `<main>` element contains the patch cables and pad grid, while each pad is
a native `<button>`: it is focusable, tappable, and works with assistive
technology without recreating button behavior in JavaScript.

```html
<main class="instrument-stage"
  aria-label="Six synthesizer pads; click or press keys 1 through 6 to play">
  <svg id="patch-wires" class="patch-wires" aria-hidden="true"></svg>
  <div id="pad-grid" class="pad-grid"></div>
</main>
```

The SVG is hidden from screen readers because it is decorative; the nearby
main label describes the instrument. Generated buttons also expose their
shortcut and a spoken description of their sound:

```html
<button type="button" aria-label="Play pad 1: C4 262Hz PAN+1 DLY 18% REB 32% SINE"
  aria-keyshortcuts="1">
  ...
</button>
```

The visible readout is marked `aria-hidden` because the button already has a
concise accessible name. The upper-left output meter uses `role="meter"` and
updates its accessible value as audio plays.

Metadata in the document head helps browsers interpret and display the page:

```html
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#08090a">
```

The character encoding supports musical text, the viewport makes the layout
fit mobile screens instead of using a desktop-sized virtual page, and the
theme color matches the dark instrument when the browser uses a colored
toolbar.

## CSS (appearance and interface)

The page uses a parent-to-child layout: `.app` fills and clips to the viewport,
`.instrument-stage` fills the app, and its `.pad-grid` divides that space into
equal tracks. Each pad fills one grid cell. There is no outer page padding or
grid gap, so the instrument reaches the window edges. A media query changes
from three columns by two rows to two columns by three rows on narrow screens.

```css
.app { width: 100%; height: 100%; overflow: hidden; }
.instrument-stage { position: relative; width: 100%; height: 100%; }
.pad-grid {
  display: grid;
  width: 100%;
  height: 100%;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  grid-template-rows: repeat(2, minmax(0, 1fr));
  gap: 0;
}
@media (max-width: 650px) {
  .pad-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
    grid-template-rows: repeat(3, minmax(0, 1fr));
  }
}
```

The selectors also express relationships and states, rather than styling only
isolated elements. For example, `.pad.is-playing .waveform` changes a waveform
only when its containing pad is active. The pad's accent color and rotation
are set as custom properties on that parent; its child shortcut, waveform,
and readout inherit or use those values. This keeps each generated pad's
decoration coordinated without writing a separate CSS rule for every pad.

```css
.pad.is-playing .waveform { opacity: 1; }
.pad-readout em { color: var(--pad-accent, var(--lime)); }
```

The SVG cables are layered between the pad backgrounds and the pad content.
Their coordinates are recalculated from the rendered pad positions, so the
cables remain attached at the midpoint of an edge when the window is resized.
Small, randomized CSS rotations give the grid an irregular, punk-inspired
character without taking the pads out of their grid cells.

## JavaScript (logic)

JavaScript creates the randomized patch, renders the six pads and their
connections, responds to pointer and keyboard input, and constructs the sound
with Web Audio nodes. The main mutable state keeps the current patch and audio
graph together:

```js
const state = {
  pads: [],
  links: [],
  audio: null,
  buses: [],
  reverbs: [],
  master: null,
  analyser: null,
  meterData: null,
  meterFrame: 0,
  noticeTimer: 0
};
```

### Variables

- `state.pads` holds each voice's synthesis type, pitch, pan, effects, color,
  and rotation. It is the source of truth for both rendering and playback.
- `state.links` stores pairs of pad indexes that describe the randomized
  connections between voices.
- `state.audio`, `state.buses`, `state.reverbs`, and `state.master` hold the
  Web Audio context and its per-voice and master signal nodes.
- `state.analyser` and `state.meterData` let the interface measure the output
  signal and draw the master meter.
- The `DP` switch controls dynamic panic, which pans each triggered voice from
  its patch position to the opposite side over the attack and release duration.
- `attackTime`, `decayTime`, `sustainTime`, and `releaseTime` are the selected
  envelope durations in seconds; their sliders are read when a note is
  triggered.
- `masterLevel` and `volumeRamp` track the target output level and current
  500 ms linear volume transition.
- `padGrid`, `wires`, `masterVolume`, and `notice` are references to HTML
  elements that JavaScript updates or listens to.
- `synthTypes`, `noteNames`, and `oscillatorLabels` are lookup data shared by
  patch generation, display, and sound synthesis.

### Functions

- `makePatch()` chooses six voices, randomizes their settings and connections,
  then calls the render and audio-routing functions.
- `renderPads()` builds each pad from the current data, including its number,
  compact readout, waveform, accessible label, and click handler.
- `renderWires()` uses the pads' current screen positions to draw SVG paths
  between the centers of the edges facing each other.
- `ensureAudio()` lazily creates and resumes the Web Audio graph. Delaying
  creation until a pad is triggered respects browser autoplay restrictions.
- `play(index)` creates an envelope and synthesis nodes for the selected
  voice, routes its dry and effect signals, and briefly marks the pad active.
- `updateMasterMeter()` measures the live output and updates both the meter
  display and its accessible value.

The four sliders shape an ADSR envelope: attack rises to the peak, decay falls
to a fixed sustain level, sustain holds that level, and release fades the note
to silence. Sustain is a duration here (rather than a level), so the four
stages all contribute to note length. Each 0–5 second slider is sampled when
the note is triggered:

```js
const attack = Number(attackTime.value);
const attackDuration = Math.max(attack, 0.005);
const decay = Number(decayTime.value);
const decayDuration = Math.max(decay, 0.005);
const sustain = Number(sustainTime.value);
const release = Number(releaseTime.value);
const releaseDuration = Math.max(release, 0.005);
const decayEnd = now + attackDuration + decayDuration;
const releaseStart = decayEnd + sustain;
const voiceDuration = attackDuration + decayDuration + sustain + releaseDuration;
envelope.gain.linearRampToValueAtTime(0.55, now + attackDuration);
envelope.gain.linearRampToValueAtTime(0.33, decayEnd);
envelope.gain.setValueAtTime(0.33, releaseStart);
envelope.gain.exponentialRampToValueAtTime(0.0001, releaseStart + releaseDuration);
```

The tiny minimum stage duration avoids scheduling zero-length automation
ramps when a slider is set to zero. Increasing any stage lengthens the
triggered sound by delaying its oscillator stop time.

Pan uses the stereo range `-1` (full left) to `+1` (full right). When dynamic
panic is on, each voice's pan ramps from its generated value to its opposite
over `attack + release` seconds (for example, `-0.3` to `+0.3`, or `+0.7` to
`-0.7`). If both controls are zero, the pan transition uses a one-second
duration.

The signal graph makes audio routing explicit. Each voice bus can connect to
the master output, and randomized links also send a quieter part of one voice
into another voice's bus:

```js
state.buses.forEach((bus) => {
  bus.disconnect();
  bus.connect(state.master);
});
state.links.forEach(([from, to]) => {
  const route = context.createGain();
  route.gain.value = 0.17;
  state.buses[from].connect(route);
  route.connect(state.buses[to]);
});
```

The master analyser is placed between the master gain and speakers. Measuring
there means the meter follows the actual combined output, including the
master-volume setting:

```js
state.master.connect(state.analyser);
state.analyser.connect(context.destination);
```

The slider changes the master gain without rebuilding the graph. Keyboard
shortcuts and pad clicks both call the same `play(index)` function, so the two
input methods share playback behavior.

The up/down shortcuts move between six fixed dynamic levels. When audio is
active, the master gain is held at its estimated current value and linearly
ramped to the next level over half a second. This also makes quick repeated
key presses continue smoothly from the in-progress transition:

```js
state.master.gain.cancelScheduledValues(now);
state.master.gain.setValueAtTime(start / 100, now);
state.master.gain.linearRampToValueAtTime(target / 100, now + 0.5);
```

Each generated patch is stored as a `{ pads, links }` snapshot in
`state.history`. Moving through that history restores the same voice settings
and connection graph; generating a new patch while viewing an older one
branches from that point and removes the now-unreachable future entries:

```js
state.history.splice(state.historyIndex + 1);
state.history.push({ pads, links });
showPatch(state.history.length - 1);
```

The history-position display stays visible on desktop; wireframe arrow
controls appear beside it on mobile. Both update whenever a snapshot is shown.
History currently lasts for the open page session; reloading the page starts a
new history.

## Design pattern

FIELDTONE uses a small **state → render → interaction** pattern. `state.pads`
and `state.links` are the current patch; `renderPads()` and `renderWires()`
turn that data into the visible interface; event handlers call functions that
change the patch or play its voices. This keeps what is displayed and what is
played based on the same data.

Audio is modeled separately as a **directed signal graph**. Oscillators
generate sound, gain nodes shape and route it, effect nodes add delay or
reverb, and the master node leads to the analyser and speakers. A patch link
is an audio connection as well as a line in the diagram. The SVG is a visual
projection of that routing data, not a second source of routing state.

# Hyperphones

These eight principles are the design and review checklist for FIELDTONE and
future Hyperphones examples. The notes below distinguish what this version
already does from what still needs work; a checklist item is a goal, not a
claim that every feature is implemented.

1. **Access:** Serve the page from an HTTPS URL and do not require an account
   to play. This static page has no sign-in flow; an HTTPS deployment still
   needs to be provided.
2. **Ubiquity:** Keep controls visible and usable on desktop and phone, without
   making any action depend on hovering. The pads and volume slider are
   visible and touchable at every size. Keyboard shortcuts are an additional
   input method, not a replacement for touch controls.
3. **Gesture:** Provide at least two identifiable musical gestures, with a
   slider or other UI alternative when a sensor is unavailable. Playing pads
   is implemented; patch rerolling is currently a keyboard-only `R` command.
   A visible reroll control and a second distinct musical gesture remain to
   be added.
4. **Feedback:** Show the current state, distinguish manual and learned
   modes, and report errors in understandable language. Active pads and the
   master meter provide feedback, and audio-start errors appear as a notice.
   The instrument is randomized by written rules, not learned from users; it
   has no explicit mode indicator or separate manual/learned modes.
5. **Control:** Start sound only after a user gesture; provide a way to stop
   sound and microphone tracks; make volume accessible. Audio starts on the
   first pad gesture and the master slider has an accessible label. There is
   no microphone input or explicit stop/pause control yet; setting volume to
   zero mutes output but does not stop active voices. All four ADSR stage
   durations are adjustable per note.
6. **Decision:** Make clear what is chosen by a person, set by a written rule,
   or learned. Pitch, effects, routing, and pad angles are randomized by
   code; the player chooses when to trigger pads and where to set master
   volume. No learned behavior is used. The interface should make this
   distinction more visible.
7. **Trace:** Support audio export and provide a notice that makes a scene
   reproducible. Generated patches can be revisited during the open page
   session, but there is no audio export or history persistence across reloads.
8. **Maintenance:** Keep sources readable, preserve versions, and provide
   basic behavior when an external API is unavailable. The app is a readable,
   dependency-free HTML file and displays an error if Web Audio is missing.
   It has no external API dependency; audio playback still requires browser
   Web Audio support.
