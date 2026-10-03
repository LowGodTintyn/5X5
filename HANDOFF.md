# 5×5 Tracker — Project Handoff

Context pack for continuing work in a **new chat**. Upload this file **and `index.html`** into the new chat and say: *"Read HANDOFF.md and index.html, then continue."*

---

## What it is
A self-built **Stronglifts 5×5 workout tracker**: a single, self-contained `index.html` (vanilla JS + CSS, no build step, no framework). Runs as an installable standalone web app with no login.

- **Live app (use this):** https://lowgodtintyn.github.io/5X5/
- **GitHub repo:** `LowGodTintyn/5X5` — the file is `index.html` at the repo root, served by GitHub Pages.
- **Claude artifact copy (backup/dev):** https://claude.ai/artifact/1yYSjNnk3HfpzJE4C5BVDw
- **Current build:** `r39` (shown as a marker at the bottom of the Settings tab — **bump this string on every change** so the user can confirm they're on the new version: cached old copies are a recurring gotcha).

User: **Martyn** — returning lifter, every-second-day, kg, in Australia.

---

## How to edit & test (important conventions)
- **One file.** Everything lives in `index.html`. All JS is inside a single IIFE in the one `<script>` block.
- **Always bump the build marker** in `renderSettings` (search `build r39`) when you change anything.
- **Syntax check:** extract the script and run node:
  ```
  awk '/^<script>$/{f=1;next} /^<\/script>$/{f=0} f' index.html > /tmp/app.js && node --check /tmp/app.js
  ```
- **Smoke test with jsdom** (stub audio APIs; call `process.exit(0)` — the workout timer uses `setInterval`, which otherwise hangs node; wrap in `timeout 25`):
  ```js
  const {JSDOM}=require("jsdom");
  const d=new JSDOM(html,{runScripts:"dangerously",pretendToBeVisual:true,url:"https://ex.test/",beforeParse(w){
    w.AudioContext=function(){return{state:"running",resume(){},createOscillator(){return{type:"",frequency:{setValueAtTime(){}},connect(){},start(){},stop(){}}},createGain(){return{gain:{setValueAtTime(){},exponentialRampToValueAtTime(){}},connect(){}}},currentTime:0};};
    w.HTMLMediaElement.prototype.play=function(){return Promise.resolve()};w.HTMLMediaElement.prototype.pause=function(){};
  }});
  // give jsdom a real url (not about:blank) so localStorage works; drive via element.click()
  ```
- Deploy = user commits `index.html` to the repo (Claude can't push to their GitHub). GitHub Pages auto-deploys in ~1 min. User must **delete + re-add the home-screen icon** to clear the iOS standalone cache.

---

## Data model
`localStorage` key: **`sl5x5.v1`**. One JSON **container**:
```
{
  v:4, updatedAt:<ms>, theme:"auto"|"light"|"dark",
  currentProfile:"<id>", profileOrder:["<id>",...],
  profiles:{ "<id>": PROFILE }
}
PROFILE = {
  id, name, unit:"kg", bar:20, plates:[25,20,15,10,5,2.5,1.25],
  rest:180, next:"A"|"B",
  active:null | {workout,date,startedAt,restSec,steps:[{lift,kind:"warmup"|"work",weight,target,reps}],idx,restUntil,phase},
  sessions:[{d,workout,sec,intensity}], lastDate, nextDue,
  workouts:{A:["squat","bench","row"], B:["squat","ohp","deadlift"]},
  lifts:{ squat:LIFT, bench:LIFT, row:LIFT, ohp:LIFT, deadlift:LIFT }
}
LIFT = {name, sets, reps, inc, weight, fails, hist:[{d:"YYYY-MM-DD", w:<kg>, ok:<bool>}]}
```
Key functions: `load/migrate/validContainer/normalize`, `save/saveLocal`, `bindProfile` (sets global `P`=current profile), `render/safeRender`, `renderHome/renderRunner/renderIntensity/renderSummary/renderProgress/renderCalendar/renderSettings/renderModal`.

Backups to restore data are at the bottom of this file.

---

## Feature map
- **Home** – next workout (A/B), date = last workout + 2 days, lifts + target weights (tap weight to nudge), last-session line, Start.
- **Guided runner** – one movement per screen. Squat leads with warm-ups (Air Squats → empty bar → 5 kg-per-side plate ramps 30/40/50/60/70; last two ramp sets = 3 reps). Work sets logged via rep adjuster. **Rest only between sets of the same lift** (skips at movement changes). Rest screen = huge countdown + set dots + "up next". **Bell + 3-2-1 ticks**, **Wake Lock** keeps screen on. End → **intensity 1-10** → summary → saves session, flips A/B, advances due date.
- **Progress** – per-lift sparklines (dots mark weight increases) + **Excel export** (hand-rolled .xlsx with an embedded line chart; uses `downloads` capability in claude.ai, falls back to a normal browser download on GitHub).
- **Calendar** – month grid, red dot = worked out, dashed marker = next session, today outlined.
- **Settings** – People (profiles, each independent), Sync status, bar/rest, **Plates you own** (tickboxes — loading only uses owned plates), per-lift weight steps, theme, **Manual backup** (copy/paste export+import), Reset.
- **Profiles** – tap name chip → "who's lifting?" sheet. Each person fully separate. Sync (claude.ai only) keeps them on all the user's devices.
- **PWA** – apple-touch-icon + manifest (orientation `any`), full-screen standalone, landscape two-column runner + rest timer, zoom locked.

---

## Known constraints & gotchas (don't re-learn these)
- **`confirm()`/`alert()`/`prompt()` are blocked in the artifact/PWA sandbox** → all destructive actions (Quit / Delete profile / Erase / Load backup) use a **tap-again (armed) pattern**, not dialogs.
- **iOS standalone sizing:** use **`height:100dvh`** on `#app` (not `100%`/`100vh`/`position:fixed`) or the bottom bar floats. Safe-area via `env(safe-area-inset-*)` on `#view`/`#tabs`, **not on `:root`** (that pushes fixed bars up on iOS).
- **Bottom nav** is an in-flow flex child (not `position:fixed`).
- **iOS audio / mute switch:** bell is an embedded `<audio>` WAV (better than WebAudio in standalone), unlocked on first tap; WebAudio tones are the tick/fallback. The hardware **mute switch** can still silence it — not fixable from the web. (Status unconfirmed by user as of r12.)
- **Runtime capabilities** (`claude.use("db"/"user"/"downloads")`) only exist when the page is **framed inside claude.ai**. On GitHub Pages they're absent → sync is off (local-only), export uses the browser-download fallback. Code already handles null gracefully.
- **Excel chart** is hand-written OOXML (SpreadsheetML + DrawingML) zipped with a self-contained store-only ZIP writer (`crc32`/`zipStore`/`buildXlsx`) — validated to open in Excel without repair.
- **Plate diagram** (`barbellSVG`): solid colour plates, thick black outline, white number (dark-outlined for contrast), large plates same height with thickness by weight (`THK` map, 5→28kg scaling), micro plates (2.5/1.25) thin, equal, numbers rotated -90°. Colours: 25 red, 20 blue, 15 yellow, 10 green, 5 grey, 2.5 blue, 1.25 yellow (user's gym).
- **Default seed = Martyn's history** → "Erase everything" resets to that seed (history returns). A truly blank start needs loading the blank backup below.

---

## Build log (r1 → r14)
r1 base app → sync → profiles → guided runner/home/Excel → calendar → Spotify (removed) → next-date = last+2 → home last-line trimmed → big rest timer → Air Squats → 5kg plate-ramp warmups → plate graphics redesign → plate inventory tickboxes → **r+: quit button fix (two-tap)** → apple-touch-icon → standalone/download fallback → **dvh bottom-bar fix** → **r7 backup-import fix (two-tap)** → **r8 landscape allowed + wake lock + louder bell** → **r9 no rest between lifts** → **r10 set dots on rest** → **r11 landscape rest split + audio-element bell** → **r12 landscape rest timer fit** → **r13 landscape home fit** → **r14 landscape two-column runner + full width** → **r15 training days (per-profile `days`, 0=Sun; empty = every 2nd day), local-time date fix (no UTC off-by-one), silent looping audio keep-alive for the bell, 2-col landscape Progress/Settings, Clear history button** → **r16 bell = WebAudio chime** → **r17 "Forge" theme (ember/rose gradients, glass cards, rest progress bar), centred one-screen landscape layouts for home/runner/rest/intensity/summary, warm-up counter fix** → **r18 "Volt" theme: dark-only (lime→cyan neon on near-black, grid + violet glow, italic caps titles); theme setting is now dark/light** → **r19 six selectable skins (Settings → Style; `S.skin`: volt/ember/aurora/gold/neon/stealth, applied as `data-skin` on `<html>` overriding --go/--go2/--glow etc.)** → **r20 Paste button in Manual backup import (iOS wasn't offering paste)** → **r21 Carbon skin (7th; carbon-fibre bg, red/orange, chamfered buttons, striped rest bar; structural overrides keyed on [data-skin=carbon])** → **r22 Glass skin (frosted panes, drifting colour blobs via body::before) + Brutal skin (flat acid-yellow, 3px white outlines, hard offset shadows); 9 skins total** → **r23 skins pruned to Volt/Aurora/Neon/Glass/Brutal (Ember/Gold/Stealth/Carbon + light/dark toggle removed; skins live in the `SKINS` array) + 3 new Dribbble-inspired card-on-colour skins: Scan (food scanner: lime/green + stone cards + scan brackets), Zone (world time: orange + warm grey + Michroma numerals), Cortex (brain health: teal tiles on cream, lavender, segmented bar); vertical centring now applies in portrait too (#view is a flex column; .restmain/.runwrap/.homewrap/.sumwrap/.igwrap use auto margins)** → **r24 reworked the three Dribbble skins (card look only, own backgrounds): Scan = very light/simple white cards + lime scan brackets; Zone = pixel/dot-matrix lettering (Doto font), stepped pixel-cut corners, dotted card texture, pixel progress bar, warm-grey cards on graphite; Cortex = dark navy, minimal, odd-shaped notched cards (asymmetric radii + side notch masks). Fonts are preloaded in boot() via document.fonts.load** → **r25/r26: barbell diagram only shown when the weight changes (first set of each lift + warm-ups); repeat sets get class .rcard.solo = huge centred weight (font-size from --n char count and per-skin --cw); Zone numerals are diamond-halftone (SVG diamond pattern clipped to Chakra Petch 700 text); Zone cards carry a dotted globe canvas (land mask from Natural Earth 110m, 2deg bitmap embedded in `LAND`, orthographic projection centred near Brisbane with an orange marker, rotates during rest); new Luxe skin (black gradient capsules, champagne Outfit 200 type, glossy round buttons, pill button with knob + chevrons); 9 skins: volt/aurora/neon/glass/brutal/scan/zone/cortex/luxe** → **r27 Zone renamed "Pixel" in the UI (internal key stays `zone`); dotted globe REMOVED (clashed) incl. LAND mask/canvas code; ALL text in Pixel is pixel type via a `*` rule (Pixelify Sans for small text, Doto for display, diamond-halftone Chakra Petch for big numerals; svg + textarea excluded); verified by scanning computed font-family on every screen** → **r28 "Dark Pixels" skin (key `dpixel`): generated from the Pixel (`zone`) rules by copying every `:root[data-skin="zone"] ...` rule with colours flipped (black cards, light diamonds %23ece9e1, warm-grey page) and added to the shared `:is(scan,zone,dpixel,cortex)` card lists. If you change Pixel CSS, regenerate/mirror it for dpixel (block at the end of <style>)** → **r29 Luxe: warm-up label (.rcard .lbl) restyled champagne Outfit 300; ALL Luxe text now Outfit via a `*` rule (scan-verified on every screen)** → **r30 Pixel + Dark Pixels: diamond-halftone now ONLY on the rest clock and finish time (.rbnum, .sumbig .st); weight + wording (.rcard .bw e.g. "Air Squats") use the Doto dot-matrix pixel font; warm-up label (.lbl) uses ink at .75 opacity** → **r31 Cortex DELETED; Pixel = whole screen light (page #e6e2da, cards #dcd8d0, 2px outlines); Dark Pixels = whole screen black (page #000, cards #0b0b0b, light text); 8 skins now: volt/aurora/neon/glass/brutal/scan/zone(Pixel)/dpixel(Dark Pixels)/luxe (9 incl. luxe)**.

## Latest changes
**r32/r33: (a) Claude cloud sync REMOVED (app is standalone; localStorage + Manual backup only; Excel export is a plain browser download); (b) bell is ~9 dB louder (WebAudio: waveshaper-limited master chain `masterOut()`, sustained envelope, 3 rings; measured offline: old RMS 0.205 -> 0.58, peak 0.92, no clipping); (c) every skin now uses ONE font family on every screen (Volt/Aurora/Neon/Glass/Brutal = Barlow Condensed, Scan = Manrope, Luxe = Outfit, Pixel/Dark Pixels = Pixelify Sans + Doto + diamond clock digits) - scan-verified; (d) ORIENTATION LOCK: Settings -> Orientation has Lock portrait / Lock landscape toggles (`S.orient` = auto|portrait|landscape; both off = auto). All layout is now class-driven (`html.land`; the old `@media (orientation:landscape)` blocks were converted by script to `:root.land ...` rules) and viewport units are `calc(N*var(--vw))`/`--vh` so that, when the device is not in the locked orientation, `html.rot` + `rot-cw`/`rot-ccw` rotates `#app` 90deg and swaps --vw/--vh (iOS cannot truly lock, so this emulates it; it also calls screen.orientation.lock() where supported). `#modal` and the toast live inside `#app` so they rotate too. Real orientation = window shape, frozen while an input is focused (keyboard guard)**.

**r34: landscape Calendar = one centred 2-column group (.calwrap grid: month+legend left, grid right); landscape Progress = slim one-row header (.phead) + compact 2x2x1 card grid (.pgrid, last card centred, fits without scrolling); "Scan" skin renamed "Light White" in the UI (key stays `scan`); orientation is re-evaluated on every render + resize + orientation media-query change**.

**r36: repeat-set (.rcard.solo) screens show "reps × weight" (e.g. "5 × 75 kg") instead of just the weight; --n includes the prefix so the auto-sizing still fits (worst case "5 × 142.5 kg" verified in all skins, both orientations). Light White swatch is white.**

**r37: FORM GUIDE. A "Form" pill button (data-form=<lift id>) sits under the set dots on the workout screen and on the rest screen. It slides in `#formcard` from the right (inside `#app`, so it rotates with the orientation lock; scrim `#formscrim` closes it, as does the X and leaving the workout). The card has an animated side-view stick figure on a canvas (`formDraw`, drawn from CSS vars so it follows every skin), a dashed bar-path line, a phase caption, 5 numbered coaching steps and a "Watch a video" link (YouTube SEARCH url, so it cannot rot). Figures are built from keyframes in the `FORM` table (hip position, torso lean, hand offset) + a 2-bone IK solver `ik()` so limb lengths stay constant; add/adjust a lift by editing `FORM`. Landscape = two-column card. Verified poses for all five lifts via a frozen-clock contact sheet.**

**r38: Form card text lighter (title 500, steps 400, Luxe 200/300; loaded Barlow Condensed 300-700 + Manrope 400-800 so lighter weights exist). The Form button now lives as a direct child of `.runwrap` / `.restmain`, absolutely positioned bottom-left of the whole workout card (those containers reserve bottom padding with !important so nothing overlaps; card skins inset it 22px). Overlap-checked in landscape for volt/zone/luxe/scan/dpixel.**

**r39: Form button moved to the TOP-RIGHT of the whole workout card (absolute in `.runwrap`/`.restmain`; those reserve top padding with !important: 50px portrait, 52px landscape; card skins inset it 22px). Overlap-checked in all 9 skins, both orientations, set + rest screens.**

## Open / possible next items
- Confirm on device that the **bell** now sounds with the mute switch on (r15 added a silent looping audio keep-alive + `audioSession.type="playback"`).
- Landscape Progress/Settings are now 2 columns but may still scroll a little.
- Remove the build marker when done iterating.
- Repo/deploy: project lives in `C:\Claude Local Files\5x5-tracker`; `origin` = https://github.com/LowGodTintyn/5X5.git, signed in via Git Credential Manager, so deploy = `git add -A; git commit; git push origin main` (Pages redeploys in ~1 min). No `gh`/node on this PC; test by loading index.html in the preview browser.

---

## Backups (paste via Settings → Manual backup → Load from text; tap "Load from text", paste, tap "Load now")

**Full seeded history (through 29 Sep):**
```
{"v":4,"updatedAt":1,"theme":"auto","currentProfile":"martyn","profileOrder":["martyn"],"profiles":{"martyn":{"id":"martyn","name":"Martyn","unit":"kg","bar":20,"plates":[25,20,15,10,5,2.5,1.25],"rest":180,"next":"A","active":null,"sessions":[],"lastDate":"2026-09-29","nextDue":"2026-10-01","workouts":{"A":["squat","bench","row"],"B":["squat","ohp","deadlift"]},"lifts":{"squat":{"name":"Squat","sets":5,"reps":5,"inc":2.5,"weight":75,"fails":0,"hist":[{"d":"2026-08-18","w":40,"ok":true},{"d":"2026-08-20","w":42.5,"ok":true},{"d":"2026-08-22","w":45,"ok":true},{"d":"2026-08-24","w":47.5,"ok":true},{"d":"2026-08-26","w":50,"ok":true},{"d":"2026-09-07","w":52.5,"ok":true},{"d":"2026-09-09","w":55,"ok":true},{"d":"2026-09-11","w":57.5,"ok":true},{"d":"2026-09-14","w":60,"ok":true},{"d":"2026-09-16","w":62.5,"ok":true},{"d":"2026-09-19","w":65,"ok":true},{"d":"2026-09-22","w":67.5,"ok":true},{"d":"2026-09-24","w":70,"ok":true},{"d":"2026-09-29","w":72.5,"ok":true}]},"bench":{"name":"Bench Press","sets":5,"reps":5,"inc":2.5,"weight":50,"fails":0,"hist":[{"d":"2026-08-18","w":30,"ok":true},{"d":"2026-08-22","w":35,"ok":true},{"d":"2026-08-26","w":37.5,"ok":true},{"d":"2026-09-09","w":40,"ok":true},{"d":"2026-09-14","w":42.5,"ok":true},{"d":"2026-09-19","w":45,"ok":true},{"d":"2026-09-24","w":47.5,"ok":true}]},"row":{"name":"Barbell Row","sets":5,"reps":5,"inc":2.5,"weight":50,"fails":0,"hist":[{"d":"2026-08-18","w":30,"ok":true},{"d":"2026-08-22","w":35,"ok":true},{"d":"2026-08-26","w":37.5,"ok":true},{"d":"2026-09-09","w":40,"ok":true},{"d":"2026-09-14","w":42.5,"ok":true},{"d":"2026-09-19","w":45,"ok":true},{"d":"2026-09-24","w":47.5,"ok":true}]},"ohp":{"name":"Overhead Press","sets":5,"reps":5,"inc":2.5,"weight":47.5,"fails":0,"hist":[{"d":"2026-08-20","w":30,"ok":true},{"d":"2026-08-24","w":32.5,"ok":true},{"d":"2026-09-07","w":35,"ok":true},{"d":"2026-09-11","w":37.5,"ok":true},{"d":"2026-09-16","w":40,"ok":true},{"d":"2026-09-22","w":42.5,"ok":true},{"d":"2026-09-29","w":45,"ok":true}]},"deadlift":{"name":"Deadlift","sets":1,"reps":5,"inc":5,"weight":80,"fails":0,"hist":[{"d":"2026-08-20","w":50,"ok":true},{"d":"2026-08-24","w":45,"ok":true},{"d":"2026-09-07","w":50,"ok":true},{"d":"2026-09-11","w":60,"ok":true},{"d":"2026-09-16","w":65,"ok":true},{"d":"2026-09-22","w":70,"ok":true},{"d":"2026-09-29","w":75,"ok":true}]}}}}}
```

**Blank slate (empty charts, beginner weights):**
```
{"v":4,"updatedAt":1,"theme":"auto","currentProfile":"martyn","profileOrder":["martyn"],"profiles":{"martyn":{"id":"martyn","name":"Martyn","unit":"kg","bar":20,"plates":[25,20,15,10,5,2.5,1.25],"rest":180,"next":"A","active":null,"sessions":[],"lastDate":null,"nextDue":null,"workouts":{"A":["squat","bench","row"],"B":["squat","ohp","deadlift"]},"lifts":{"squat":{"name":"Squat","sets":5,"reps":5,"inc":2.5,"weight":20,"fails":0,"hist":[]},"bench":{"name":"Bench Press","sets":5,"reps":5,"inc":2.5,"weight":20,"fails":0,"hist":[]},"row":{"name":"Barbell Row","sets":5,"reps":5,"inc":2.5,"weight":30,"fails":0,"hist":[]},"ohp":{"name":"Overhead Press","sets":5,"reps":5,"inc":2.5,"weight":20,"fails":0,"hist":[]},"deadlift":{"name":"Deadlift","sets":1,"reps":5,"inc":5,"weight":40,"fails":0,"hist":[]}}}}}
```

