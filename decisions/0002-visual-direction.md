# ADR-0002: Instruction-Booklet Style with LEGO Red and Yellow

- Status: Accepted
- Date: 2026-10-28
- Deciders: Legoin team

## Context and Problem

BrickSense shows photos of the user's bricks, detection boxes over them, and inventories that label each piece with one of 10 MVP colors: red, blue, yellow, green, black, white, light and dark gray, orange and brown. The interface needs its own palette and style, and the palette must not be mistaken for a brick color in those screens. The audience includes children, teenagers, adult hobbyists and parents.

## Decision Drivers

- The accent must not be confused with a detected brick color in photos, boxes or inventory chips.
- WCAG AA contrast for text and controls.
- A friendly, playful feel that suits LEGO owners of all ages.
- Distinct from the team's other projects (aksesin's cobalt and lime, lokerin's and ngonlenin's orange).
- No confusion with LEGO's own brand, since the app is public.

## Considered Options

Palette:

1. Custom fuchsia, grape or teal accents. Fuchsia sat furthest from every brick color: 55° of hue from red.
2. A LEGO brick color the detector never labels. Magenta (#923978) is 50° from red with 6.8:1 for white text; Dark Turquoise and Medium Lavender need a darker step for button text.
3. Classic LEGO red (#C91A09) and yellow (#F2CD37).

Style:

1. "Buku instruksi": an instruction booklet with white pages, parts callouts with counts, and numbered steps.
2. "Papan dasar": a baseplate with stud-dot backgrounds and chunky brick-edge buttons.
3. "Lab pemindai": a dark scanner UI with viewfinder brackets and mono confidence readouts.

## Decision

Style: **"Buku instruksi"**, because the app's output is literally a booklet page: the parts used, the parts missing, and short steps.

Palette: **classic LEGO red and yellow**, chosen for recognisability and a playful feel. Red (`red-600` = LEGO Red) is the primary action; yellow (`yellow-300` = LEGO Yellow) is the highlight. Ink is LEGO Black, and the neutrals come from LEGO Light Bluish Gray. All scales are OKLCH ramps around these exact Rebrickable values.

These rules keep the known risks in check:

- Detection boxes and their labels are ink with a white halo, never red.
- Brick colors are data: they paint real bricks only, always with the color name.
- Status is a dot plus a label, never color alone.
- Destructive actions use an outlined button with confirmation, because danger and primary share red.
- The logo is a yellow 2×2 brick in ink line art; red and yellow never appear together in the logo or large brand areas. The UI name is BrickSense, without "LEGO".

## Consequences

- Positive: the app reads as "about bricks" at a glance, and the booklet layout maps directly onto inventories and recommendations.
- Positive: every text pair meets WCAG AA. White on LEGO Red is 5.8:1 and ink on LEGO Yellow is 12.1:1.
- Negative: red is also the most common brick color, so the primary button sits 3° of hue from the red-brick label. The rules above carry the separation instead of hue.
- Negative: red and yellow are LEGO's brand colors, so logo and marketing work must keep checking the trade-dress rule.
- Negative: danger and primary share a hue, so destructive actions depend on shape (outline) and wording, not color.

## Pros and Cons of the Options

### Custom fuchsia

- Pro: the largest hue distance from all 10 brick colors; magenta is a classic overlay color in computer vision.
- Con: not a LEGO color, and less recognisable.

### LEGO Magenta

- Pro: a real LEGO color that the detector never labels, with AA contrast for white text.
- Con: reads less like "LEGO" than red and yellow.

### Classic red and yellow

- Pro: instantly recognisable and playful.
- Con: collides with the red and yellow brick labels, and sits close to LEGO's trade dress.

### Papan dasar

- Pro: the most playful and tactile.
- Con: busy patterns next to photos where users judge brick colors.

### Lab pemindai

- Pro: shows off the CV pipeline in a demo.
- Con: the dark, technical look is the least friendly for children and parents.

## References

- PRD BrickSense v0.1 (28 October 2026)
- Live design system: `/desain` and `DESIGN.md` in [legoin-web](https://github.com/al-dioooo/legoin-web)
- Rebrickable color values: <https://rebrickable.com/colors/>
