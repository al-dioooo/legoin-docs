# ADR-0006: Motion and Interaction Language: an Instruction Booklet on a Baseplate, Built on CSS and View Transitions

- Status: Proposed
- Date: 2026-11-07
- Deciders: Legoin team
- Issue: #10
- Extends: [ADR-0003](0003-ballers-style-lego-colors.md)

## Context and Problem

[ADR-0003](0003-ballers-style-lego-colors.md) fixed how Legoin looks: paper, ink outlines, hard shadows, stickers, LEGO Red and Yellow, Dela Gothic One and DM Sans. It said little about how Legoin moves. The screens were centered cards in a column with a few unrelated transitions: a hover tilt, a pulse and a scan line that animated `top`. The team asked for a bolder, less mainstream layout with motion and interaction of its own. What should that language be, and what should it be built with?

## Decision Drivers

- Keep the ADR-0003 identity exactly: palette, type, tokens and the rules that keep UI colors apart from brick colors.
- One recognizable idea, reused, rather than a different effect on every screen.
- The scan should be the moment people remember: it's the product's magic, and it's where they wait.
- Readable at 360px and smooth (60 fps) on a mid-range phone.
- Accessible: `prefers-reduced-motion` respected everywhere, focus always visible, motion never blocks input.
- Few dependencies: the web app's runtime is next, react and react-dom ([ADR-0001](0001-web-tech-stack.md)).
- Stay clear of LEGO's trade dress: a generic building-instructions feel, never official instruction-booklet artwork or logos.

## Considered Options

Direction:

1. **"Instruction booklet on a baseplate":** screens are booklet pages laid on a stud grid; they assemble step by step, and every piece clicks into place.
2. **Generic polish:** fade-and-rise entrances, hover lifts and a spinner, applied evenly.
3. **"Lab pemindai" motion:** a dark, technical scanner aesthetic with readouts and glitch effects.

Implementation:

1. **CSS and platform APIs:** CSS keyframes and transitions driven by tokens, React 19's `<ViewTransition>` on the browser's View Transitions API, and `requestAnimationFrame` for the pointer.
2. **Motion (formerly Framer Motion):** springs, layout animations and `AnimatePresence` for exits.
3. **GSAP:** timelines and sequencing.

## Decision

Direction: **"Instruction booklet on a baseplate"**, because it's literally what Legoin produces (a parts list and steps) and the brick is the one physical gesture everyone recognizes: pressing a piece onto studs. Implementation: **CSS and platform APIs, no new dependency.**

**Baseplate layout.**

- A stud grid drawn with CSS gradients is the page background and the layout module. One stud (`--stud`) is 16px on phones and 24px from 640px.
- The app column is 49 studs wide, and the grid starts at its left edge, so plates sit between studs.
- Steps are composed as asymmetric booklet pages: the title and main plate on the left, supporting plates down the right, overlapping by a stud or two.
- Sticker headings sit a couple of degrees off-axis.
- The current step's number, outlined and filled with studs, hangs off the page edge.

**One signature motion: the click-in.** A piece drops onto the studs, squashes a little on impact and settles. It's defined once as tokens in `tokens.css`:

- **Durations:** press 80ms, release 280ms, lift 260ms, click-in 420ms, exit 200ms, morph 440ms, odometer 900ms.
- **Stagger:** a 55ms step, 8 steps at most.
- **Distances:** a 0.75rem drop and a 1.04 × 0.92 squash.
- **Curves:** `ease-drop` (falls like a weight), `ease-settle` (settles with a small overshoot) and `ease-press`.

Everything reuses it:

- page entrances, which assemble child by child;
- button presses, where filled buttons sink exactly into their 4px hard shadow;
- selection, where a detection box lifts and tilts over a dotted stud footprint, and its inventory tile slides out like a part callout;
- removal, which pops a piece off; undo clicks it back.

**The scan is the showcase.**

- A scanner beam sweeps the photo, and the studs it passes light up, then fade behind it.
- Detection boxes and inventory tiles click in one by one while the counts roll up like an odometer.
- When a scan fails, a small build wobbles and tumbles apart.

**Between steps,** two `<ViewTransition>` elements carry continuity:

- the photo is one shared element from the preview, through the scanner, to the results;
- the yellow current-step marker slides to the next step.

The rest of the page only fades. The capture flow hands the scan it just received to the results screen, so the photo is there in the navigation's first frame and the pair can morph.

**Pointer and touch.**

- With a mouse, studs near the pointer light up: a mask moved once per animation frame.
- At most one primary button per screen is magnetic; today that's "Scan pieces".
- On phones, selecting a piece and undoing a removal give an 8ms vibration where supported.
- Touch screens and reduced motion get neither the glow nor the magnet.

**Rules.**

- Animate `transform` and `opacity`, plus `clip-path` and `filter` where cheap. Never animate layout.
- Never block input: clicks pass through view transitions, and exits don't disable the page.
- Loops run only while needed: the scanner only during a scan, demos only while on screen.
- Reduced motion keeps every fade and state change and drops the travel. The tokens set distances, tilts and stagger to zero, so click-ins become short fades. Counts change at once, view transitions swap instantly, and the beam, glow, magnet and haptics are off.
- `data-motion="reduce"` previews the same rules on a subtree.

`/desain/gerak` shows all of it live, and `DESIGN.md` documents it under "Motion & interaction".

## Consequences

- Positive: Legoin moves like it looks. One motion, learned once, ties entrances, presses and selection together, and the scan becomes something worth watching rather than a spinner.
- Positive: no new runtime dependency. Everything is CSS, React and browser APIs, so there's no bundle cost and no animation-library upgrade path to maintain.
- Positive: it degrades well. Without View Transitions support (older Firefox and Safari), steps simply swap; without JavaScript, CSS entrances still play.
- Positive: timings live in one place, so tuning the feel is a token change, and reduced motion is a token override rather than per-component code.
- Negative: the photo only morphs when the results render in the same commit as the navigation. A `loading.tsx` or a slow fetch on that route turns the morph into a plain fade. The scan handoff exists for this reason, and it must not outlive the API's own copy of the scan.
- Negative: CSS can't animate an element after React removes it. Exits keep the element for the exit duration (a `leaving` state plus a short wait), where a library would offer `AnimatePresence`.
- Negative: the pointer glow repaints a masked layer on each frame the mouse moves. It's limited to desktop pointers and to bare baseplate, but it isn't compositor-only.
- Negative: the stud grid is the "Papan dasar" pattern that [ADR-0002](0002-visual-direction.md) passed over as busy next to photos. To keep that risk in check:
  - the rings are 9% ink;
  - opaque plates cover the grid wherever content sits;
  - studs only ever cross a photo during a scan, while nobody is judging colors yet;
  - the glow lights only bare baseplate.
- Negative: the reduced-motion token values are written twice, once for the media query and once for the `data-motion` preview.
- Negative: the home page and the ideas screens don't use the booklet layout yet, so the app is inconsistent until a follow-up brings them along.

## Pros and Cons of the Options

### Instruction booklet on a baseplate

- Pro: specific to Legoin; no other app could use it unchanged.
- Pro: the click-in is a single, physical, recognizable gesture.
- Con: asymmetric pages take more care at each breakpoint than a single column.

### Generic polish

- Pro: quick, safe and familiar.
- Con: interchangeable with any app; it doesn't answer the request for something less mainstream.

### Lab pemindai motion

- Pro: makes the computer-vision pipeline feel powerful in a demo.
- Con: a dark technical look conflicts with ADR-0003's paper and stickers, and it's the least friendly for children and parents (also noted in ADR-0002).

### CSS and platform APIs

- Pro: no dependency, nothing to load, and compositor-friendly by default.
- Pro: React 19.3 and Next 16 support `<ViewTransition>` with no configuration.
- Con: exits and interruption need small hand-written patterns, and View Transitions styling is newer and varies a little between browsers.

### Motion

- Pro: springs, layout animations and exit animations out of the box.
- Con: a sizeable runtime dependency for effects CSS already covers, and its layout animations would overlap with View Transitions.

### GSAP

- Pro: precise timelines for long choreographed sequences.
- Con: an imperative layer beside React for a product that needs short, state-driven motion; its license terms would be a dependency to watch.

## References

- [ADR-0003](0003-ballers-style-lego-colors.md), the visual identity this extends
- [ADR-0002](0002-visual-direction.md), where "Buku instruksi" and "Papan dasar" were first compared
- Implementation: al-dioooo/legoin-web#32; live at `/desain/gerak`
- Next.js guide "Designing view transitions" and React's `<ViewTransition>` reference
