# Save the Nest — Project Context

A single-file, browser-based **Long / Short sorting game for young children**. The player drags
items (twigs, leaves, feathers) into two stone boxes labelled **Long** and **Short**, guided by a
friendly queen-bee mascot.

- **Entry point:** [`index.html`](index.html) — the *entire* game (HTML + CSS + vanilla JS, no build step).
- **Assets:** [`assets/`](assets/) — all raster art is **WebP**. **Audio:** [`audio/`](audio/) (looping BGM).
  > Asset filenames are **space-free** (hyphenated, e.g. `bee-1.webp`, `long-twig.webp`, `group-38510.webp`,
  > `audio/background-music.mp3`). Spaces in filenames 404 when the page is served over HTTP (Live Server,
  > `python -m http.server`, etc.), so keep new files hyphenated.
- **Run it:** open `index.html` in any modern browser. No server/dependencies required.

---

## Tech & architecture

- **Fixed stage:** everything lives in a `1920×1080` `#stage` that is uniformly `transform: scale()`d to
  fit any screen (`flex:none` prevents flex-shrink distortion). All coordinates below are in this 1920×1080 space.
- **Responsive / all devices:** `fitStage()` scales by `min(vw/1920, vh/1080)` using `visualViewport` (re-runs
  on `resize`/`orientationchange`/`load`/`pageshow`), the stage stays centred, and the sandy radial-gradient
  body fills the **letterbox** on any aspect ratio (4:3, ultrawide, etc.). The page is **locked** for mobile
  (`position:fixed`, `overflow:hidden`, `overscroll-behavior:none`, `user-scalable=no`) — no scroll, zoom,
  rubber-band or pull-to-refresh — and long-press menus are blocked over the stage. On **touch devices held
  in portrait**, a friendly **rotate-to-landscape** prompt (`#rotate`, pure CSS media query) covers the screen
  until the device is turned (the layout is fixed-landscape).
- **Drag & drop:** Pointer Events (`pointerdown/move/up`, `setPointerCapture`). `toStage(clientX,clientY)`
  converts screen → stage coords accounting for the scale.
- **Animation:** Web Animations API (`element.animate(...)`) for one-shot effects; CSS `@keyframes` for
  loops (hover, sway, pulse).
- **Audio:** procedural **WebAudio** SFX (pickup, correct arpeggio, wrong buzz, fanfare) in an `Audio` IIFE
  module + a looping `<audio>` BGM (`audio/Background music.mp3`, volume ~0.12). Audio is unlocked on the
  first user gesture (autoplay-safe); a `bgm.paused` guard prevents double-play.
- **Items** are `{art, cat}` — `art` = which PNG/WebP, `cat` = `'long'` | `'short'` (which box it belongs in).

---

## Game flow (start → play)

0. **Title / cover screen** (`#cover`, z-index 90): full-bleed `cover-page.webp` ("Sort the items")
   with a pulsing **Play button** (`play-button.svg`) in the lower-centre. Tapping the button (or
   anywhere on the cover) fades it out (`enterFromCover`), revealing the splash. This is the first
   screen shown; the old 6 s auto-start is gone (entry is an explicit tap).
1. **Splash screen** (`#intro`): bee lower-left (`.intro-bee`), the speech-bubble (`dialogue-box.svg`)
   **top-centre** typing out "Let us start sorting!", and the **nest pile** (`.intro-nest`, `pile.webp`)
   on the **right**. Once the greeting finishes typing, it holds for **~2 s**, then the bee flies out
   and the dialogue fades away (`startGame`) → gate → tutorial. Sandy background.
2. **The standing bee swaps to the flying bee** which zig-zags smoothly **off the top-right** of the
   screen (arc-length-matched keyframes, accelerating ease-in). The flying-bee art natively faces right — it must NOT
   be mirrored (the standing bee *is* mirrored, `scaleX(-1)`, to face her speech bubble).
3. **Honeycomb cell transition** (`honeycombGate`): the full-viewport `<div id="honeycomb">` has an
   **opaque honey background** so it covers the screen the **instant the transition starts** (the
   board behind never peeks while the cells assemble). A grid of honeycomb cells
   (`transition-bee-comb.svg`, one flat-top hexagon `<img>` per cell) pops in centre→out for texture;
   the **empty** board loads behind (`onCovered`); then the cells **dissolve centre→out and the honey
   backing fades**, revealing the game. Only after the reveal does `loadLevel` run — so the items are
   **seen flying out of the nest pile**; the tutorial starts shortly after. (z-index 9000; flat-top hex
   lattice, colStep ¾W, odd columns dropped ½H, cells scaled 1.06; covers the letterbox too.)
4. **PART 1 — Guided tutorial** (level 1 only, `runTutorial`) — **the player does all the
   sorting**; nothing is auto-played and there are no guide arrows. Pieces are **locked** (a
   `.locked` class blocks `makeDraggable`) until the bee names them — she unlocks **one at a time**:
   1. "Long items go here." — pieces stay locked while she types; once it finishes, **only the long
      twig** unlocks (the short stays locked). It **glows** (`.tut-target`), the Long box pulses,
      and the **ghost drag animation** loops (`showDragHint`/`replayHint`) until the player drags it in.
   2. "Short items go here." — then the short twig unlocks for its turn → Short box.
   `place()` dispatches an `itemplaced` event the tutorial listens for to advance; `finish` unlocks
   everything for free play. Wrong drops just snap home and the ghost keeps hinting.
5. **Tutorial → game hand-off**: after the tutorial celebration the **instructor bee quickly flies off and
   exits the RIGHT** of the play area (`flyInstructorAway`, ~0.56 s). Then the full-screen **"Now, it is
   your turn!"** scene appears (`showHandoff`, design slide 63 — game background): the **flying bee flies
   in** from off-screen left (~0.72 s) and settles centre-left; **then the speech bubble pops in**; **then
   the text types**. After she's spoken (or on tap), the bubble pops away and the **bee flies off the
   right**, and only **then** does the **honeycomb gate** transition into level 2, where the game begins.
6. **PART 2 — Free play** (levels 2-5): **no dialogue or text at all** — feedback is purely
   sounds, cheers and effects, plus the ~11 s idle ghost nudge if the player stalls.
7. **Play:** drag each item into the correct box.
   - **Correct:** box scale-bump flash, gold **sparkle stars**, the piece flies into the box and vanishes, bee cheers.
     Dragging a correct item over its box gives a subtle **scale lift** (no green glow — removed).
   - **Wrong:** the piece does a **gentle shake** and snaps home — **no red anywhere**. Both the
     wrong-box hover glow *and* the red drop-shadow on the dropped piece (`.item.wrong`) have been
     removed; only the **correct** box gives a subtle scale lift (`glow-ok`).
8. **Level complete:** **leaf confetti** (`spawnCelebration`/`fireConfetti` — the game's leaf artwork
   (`ART.leaf`/`ART.longleaf`) bursts up + inward from the lower corners + a centre pop, arcs over,
   then flutters/flips down past the bottom). A short beat, then the next level's items **fly out of
   the nest pile automatically**; the leaves keep fluttering down over the transition (not cleared by
   `clearBoard` — they self-remove).
9. **5 levels.** Finale (`showFinale`): after the last level, leaf confetti bursts, then the
   **honeycomb gate** reveals the **finale scene** (`#finale`, full-bleed `last-screen.webp` — the
   two filled stone boxes + nest). The **bee flies in** from off-screen, lands (swaps to her standing
   pose), says **"Now, that is some great sorting!"** in the speech bubble, and a **Play Again** button
   (`play-again.svg`) appears → tapping it gates back to a fresh game (level 0). (The old `#overlay`
   "Nest Saved!" screen is no longer used.)

---

## On-screen layout (1920×1080)

| Element            | Position |
|--------------------|----------|
| Bee instructor `#bee` | top-left (`left:185 top:40`), faces right, hover-bobs; **tapping her plays a bee buzz (`Audio.buzz`) + a cheer + a sparkle ring**. Speech bubble appears **only during the tutorial** (typewriter text via `setBubble`) — the main game has no dialogue/text, just cheers, sounds and effects |
| Nest pile `#nest` (`pile.webp`) | bottom-left (`left:-15 top:480`, 590px wide, `pointer-events:none`) — the **source of all materials**: every level's items fly out of it to the band, and it **thins out each level** (`updatePile`: scales 1→0.4 + fades toward the ground; on the **last level it shrinks away completely** — pile fully used up). Resets on Play Again |
| Items (2 per level) | upper-right band (`IX0=805, IX1=1705, ITEM_CY=255` → centres x≈1030 / x≈1480). **Sides are randomised in the game levels** (`flip = (i!==0) && Math.random()<0.5`) so the long/short piece isn't always the same side — the **tutorial (level 0) keeps its fixed layout** (short left, long right). Each item's **size + tilt is set per-level** in `LEVELS` (`w/h/rot`); pieces lie **nearly flat** (`rot` ≈ −66°…−90°) so the two lengths read side-by-side |
| `Long` box `#basketLong`  | stone frame `group-38510`, `left:676 top:408`, `570×435`; "Long" title centred **inside** the box — **Fredoka One** 92px, `#804206`, `opacity:.45` (written-into-sand look, no stroke) |
| `Short` box `#basketShort`| stone frame `group-38510`, `right:42 top:408`, `570×435`; "Short" title inside the box, same style |

Drop targets are invisible `.catch` rectangles inside each box.

---

## Levels (`LEVELS` in index.html)

Each item is `{art, cat, w, h, rot}` — the **size is per-level**, so the same art appears at different
lengths and the length *difference* within a pair is tuned (big diff = easy, small diff = harder).
Items lie nearly flat so the two lengths read side-by-side. Long-category → **Long** box; short → **Short**.

1. **L1 (tutorial):** long twig *(long)* + short twig *(short)* — clear, **easy** difference
2. **L2:** short leaf *(short)* + long leaf *(long)* — **very big** difference
3. **L3:** long leaf *(long)* + short feather *(short)* — **very big** difference
4. **L4:** short feather *(short)* + long twig *(long)* — **big** difference
5. **L5:** long twig *(long)* + short twig *(short)* — **less** difference
6. **L6:** short twig + short leaf *(short)* — **less** difference; **TWIST** — the (short) twig still goes in the **Long** basket (`cat:long`), the leaf in **Short**

(6 levels total. `cat` — not visual length — decides the basket; L6 leans on that. The tutorial item pair is unchanged.)

(The "short feather" reuses `long-feather.webp` sized small; the blue feather `feather2` is now unused by levels.)

---

## Assets (`assets/`, all WebP unless noted)

**In use**
- `background.webp` — sandy play area with leafy/rock corners (1920×1080; also the splash bg; converted from PNG, 2.8 MB → 141 KB)
- `group-38510.webp` — grey stone-pebble box frame (both drop boxes)
- `pile.webp` — twig nest pile with feathers (bottom-left decoration; converted from PNG, 540 KB → 178 KB)
- `cover-page.webp` — title-screen art ("Sort the items" with a nest in the "o"; converted from PNG, ~2.8 MB → 434 KB)
- `play-button.svg` — round orange ▶ Play button on the title screen
- `play-again.svg` — orange "Play Again" pill button on the finale screen
- `last-screen.webp` — finale scene background: both stone boxes **filled with the sorted items** + the
  nest in the corner (1681×935, ~16:9; shown full-bleed via `center/cover`; converted from PNG, 2.9 MB → 317 KB)
- `dialogue-box.svg` — **white speech bubble** with a thin gold-gradient border and a tail **bottom-left**
  (556×307, aspect ≈1.81). Used for the splash, hand-off + tutorial bee bubble, and the finale. Each
  bubble's box size + position is tuned per-screen so the **tail connects to the bee** without overlap and
  the text fits inside the **white area**; these sizes intentionally favour tail alignment over a strict
  aspect match. The bee bubble sits beside the bee (no overlap); the main game shows no text — bubbles
  appear only in the tutorial and finale.
- `bee-1.webp` *(animated)* — standing/instructor + splash bee
- `bee-flying.webp` *(animated)* — splash fly-out bee
- `cursor-pointer.svg` — **grab-hand** cursor (dark near-black fill + white outline + soft shadow),
  shown **only over `.item` pieces** (twigs/leaves/feathers); 60×63, hotspot at the fingertips (`28 4`)
- `transition-bee-comb.svg` — a single flat-top honeycomb hexagon (331×287), **tiled into a grid**
  for the cell transition (`honeycombGate`)
- Items: `long-twig`, `small-twig`, `short-leaf`, `long-leaf`, `long-feather`, `long-feather-2`
- `audio/background-music.mp3` — looping BGM

**Assets are clean** — every file in `assets/` is referenced by the game (verified: no orphans).
All raster art is **WebP**; vector art (cursor, dialogue box, transition hexagon, play buttons) is
**SVG** (kept as vectors — rasterizing them would blur when scaled). The legacy `.png`/`.gif` source
files and unused leftovers (`corsour*`, `leaf-gate.svg`, `let's gp.svg`, `Question bannner.webp`) have
all been **deleted**. New raster images get converted PNG→WebP via `sharp`.

> All raster assets were converted PNG→WebP (cwebp) and animated GIF→animated WebP (gif2webp).
> e.g. `background` 2.8 MB → 189 KB. References in `index.html` point to `.webp`.

---

## Notable features / effects

- **Typewriter text** (`typeWriter`) on splash title + all tutorial bubbles (supports `\n` line breaks).
  Slow, deliberate pace (~9-11 cps) with a soft per-character **tick SFX** (`Audio.type` — a short,
  quiet square blip with slight pitch jitter; silent until audio is unlocked, skips spaces).
- **Honeycomb cell** transition (`honeycombGate` — `transition-bee-comb.svg` hexagon tiled into a grid that pops in centre→out to cover, then dissolves to reveal).
- **Guided tutorial** with box highlights (`runTutorial`) — message-only, skippable by any tap/drag.
- **Custom cursor** — a themed golden **grab-hand** (`cursor-pointer.svg`) shows **only over the
  draggable pieces** (`.item` — twigs, leaves, feathers) to signal "grab & drag"; everywhere else
  keeps the normal cursor. Hotspot at the fingertips. The old top-down **bee cursor +
  instructor→cursor morph have been removed**.
- **Idle nudge** — after ~11 s of no interaction the drag-hint ghost replays (`armIdle`/`idleFire`); any tap resets it.
- **Juice:** sparkle-star burst + box scale-bump on correct drops (no green glow), **leaf confetti** on level complete,
  ambient falling leaves + floating air motes, bee hover/cheer.

---

## Key functions (in `index.html` `<script>`)

- Level/board: `loadLevel(i, skipHint)`, `makeItem`, `makeDraggable`, `place`, `clearBoard`, `levelComplete`
- Bee/text: `beeSay`, `beeCheer`, `typeWriter`
- Tutorial/hints: `runTutorial`, `showDragHint`, `armIdle`/`idleFire`
- Transitions: `startGame` (intro fly-out), `honeycombGate`, `enterGame`
- FX: `spawnConfetti`, `burst` (stars), `sparkleRing`, `spawnLeaf`, `spawnMote`
- Audio: `Audio` module — `ensure`, `startBgm`, `pickup`, `correct`, `wrong`, `fanfare`, `toggle`

---

## Ideas / possible next steps

- Spoken voice-over for instructions & praise (great for pre-readers).
- A "growing nest" goal that visibly fills as you sort, with chicks hatching at the end.
- Vary the sort rule per level (by colour / by type), star ratings, a level map, sticker rewards.
