# ADR-0010: A Library of Checked Builds, Made and Rendered Offline with BrickLayer

- Status: Proposed
- Date: 2026-11-25
- Deciders: Legoin team
- Issue: #19

## Context and Problem

An idea is text: a theme, a difficulty, a title, a summary, steps, and a parts list that `cek_inventori` splits into `used` and `missing`. Nothing shows what the finished build looks like. Nothing checks that it can be built either, and some of the 18 templates can't be built as written:

- the delivery truck's 2×3 cab and 2×4 cargo box don't fit on its one 2×4 plate;
- the penguin's belly and the windmill's crossed sails need sideways studs, which the catalogue doesn't have;
- the house's door step and the bus stop's bench aren't attached to anything.

A teammate found [BrickWright](https://github.com/AvaLovelace1/brickwright) (MIT, research from CMU and Stanford, v0.1.0). It ships two Python packages:

- **BrickLayer** (`bricklayer-sdk`) builds models from real LDraw catalogue parts in Python. It checks them for collisions, floating parts and connectivity, and exports LDraw `.ldr` files. It covers all 13 catalogue parts and all 10 colors.
- **BrickWright** (`brickwright`) is an agent loop that has an LLM write BrickLayer code. Its "external agent" commands (`brickwright external init`, `validate`, `render`, `finalize`) let a coding agent write the build itself, with no API key.

How should Legoin use it to show a picture of each build, and to make sure the builds work, inside the 10-second budget for ideas?

## Decision Drivers

- **A picture of the finished build** on every template idea, and on Claude's ideas when the person's pieces fit a checked build.
- **Builds a person can follow:** checked for collisions, floating parts and connectivity, with steps that match the build.
- **The budget for ideas.** The contract's 10 s, which Sonnet 5.5 at `low` meets only just (ADR-0009), and about $0.02 a request.
- **Nothing heavy at runtime.** legoin-api's image, its start-up and CI stay as they are: no GPU, no 620 MB parts library, no new runtime dependency.
- **The licences** of what's used, credited where the pictures appear.

## Considered Options

1. A library of checked builds, made and rendered offline with BrickLayer; the service only serves the files
2. BrickWright writes a new build for each idea, in real time
3. Claude writes BrickLayer code for its own ideas, which the service validates and renders per request
4. No pictures

## Decision

Chosen option: "a library of checked builds, made and rendered offline", because it gives every template a real, checked picture at no cost per request. The service never runs BrickLayer or needs the LDraw parts library. Real-time generation with BrickWright is out of scope: it would take minutes and cost money for each idea.

### The library

It lives in legoin-api, in `builds/`, with one directory per build:

- `build.py`: the build in BrickLayer, kept deterministic, with its theme, difficulty, title, summary and steps in English and Indonesian;
- `model.ldr`: the build as an LDraw file;
- `preview.webp`: an isometric picture, 640 px square, on a transparent background.

`builds/manifest.json` lists each build with its parts, counted from the build itself, and each file's SHA-256.

- **18 builds to start, one per template,** with the template's id. The builds were written by a coding agent using BrickWright's external-agent workflow and BrickLayer, with no API key. `brickwright generate` wasn't used.
- **Every build passes BrickWright's validation:** no collisions, nothing floating, one connected assembly. Each is 3 to 15 pieces, from the catalogue's parts and colors only. The 18 are 3 to 9 pieces.
- **Each template matches its build:** the same theme, difficulty, title, summary, steps and parts, and a test checks it. Where they differed, the template changed:
  - the delivery truck, rubber duck, penguin, little house, small bridge, bus stop and windmill changed their parts, and their steps with them;
  - the race car's body moved a stud back, so its lights have studs to sit on;
  - the tractor, rocket, crab and puppy say more exactly where pieces go.

  The other six are unchanged. The details are in legoin-api's PRs.

### Authoring

- **`scripts/builds.py` in legoin-api,** a uv script with its own pinned `bricklayer-sdk` 0.1.0 and `brickwright` 0.1.0, locked in `scripts/builds.py.lock`. They're never installed with the service, and CI doesn't need them. For each build it:
  1. runs BrickWright's own validation, the code behind `brickwright external validate`;
  2. checks the pieces and the text;
  3. writes `model.ldr` and renders `preview.webp`;
  4. writes the manifest.
- **The tests** need neither BrickLayer nor LDraw. They check the manifest against the catalogue, every file against its hash and size limit (a preview under 150 KB), and each template against its build. A `build.py` changed without running the script fails its hash check.
- **Rendering** uses the LDraw parts library (about 620 MB unzipped), kept outside the repositories. It runs on a CPU (see the spike below).

### At runtime

- **legoin-api serves the files** as static files, at `/v1/builds/{buildId}/preview.webp` and `/v1/builds/{buildId}/model.ldr`. Each URL carries the file's hash (`?v=`) and is cached for a year as `immutable`. There's no device header: they're public catalogue content, not anyone's photos.
- **An idea gets two optional fields** in the [contract](../api/contract.md): `buildId`, the build it shows, and `preview: { image, model }`, absolute URLs of its picture and `.ldr`. Additive only: everything else stays as it is.
- **Template ideas** carry both.
- **Claude** gets a short summary of the library in the cached part of its system prompt: each build's id, theme, titles and parts.
  - **When the person owns every piece of a build, or all but one,** Claude can answer a slot with its `buildId`, an `enum` of the library's ids in the answer's schema. The service then uses that build's text, parts and steps, still runs `cek_inventori` for `used` and `missing`, and adds `preview`.
  - **A pick that's wrong** gets ADR-0009's single repair, then a template. Wrong means in another theme, already seen, picked twice, or missing more than one of the person's pieces.
  - **Claude's own builds** keep no preview.
- **One round of `cek_inventori` calls.** Claude checks every build in one turn, then answers: two API calls. ADR-0009 allowed three rounds, but with the library in the prompt, Claude sometimes took a second round, at about 4 s more (see the measurements below). The service runs `cek_inventori` on every final build anyway.
- **legoin-web** shows the picture on idea cards and the idea page, with a link to download the `.ldr`. Its mock has the same 18 pictures and the same templates.

### Credits

Wherever the pictures appear (the web app's footer, legoin-api's README, each `.ldr` file's header):

- BrickLayer and BrickWright, MIT: Pun, A., Dai, K., Chen, S., Jones, M., Liu, R., Deng, K., Liu, C., Agrawala, M., Ramanan, D., & Zhu, J.-Y. (2026). *BrickWright: Agentic Creation of Physically Feasible Brick Builds*. [arXiv:2610.31651](https://arxiv.org/abs/2610.31651).
- BrickLayer's part data, from the [LDraw.org Parts Library](https://library.ldraw.org/) (CC BY 2.0 and 4.0), [BrickNet](https://github.com/kulits/BrickNet) (MIT) and [Rebrickable](https://rebrickable.com/downloads/). The previews are rendered from the LDraw.org Parts Library, which the `.ldr` files refer to but don't include.
- "LEGO® is a trademark of the LEGO Group, which does not sponsor, authorize or endorse this project."

### The spike

It ran on 2026-11-25 on the project's server (AMD EPYC, 16 cores, no GPU, Ubuntu 24.04), in a scratch uv environment outside the repositories.

- **Install:** `bricklayer-sdk` 0.1.0 and `brickwright` 0.1.0 install from PyPI on Python 3.12. The LDraw parts library (`complete.zip`, 147 MB, 2026-05-29) unzips to 620 MB.
- **The catalogue:** all 13 parts (3005, 3004, 3622, 3010, 3003, 3002, 3001, 3023, 3022, 3020, 3039, 3069b, 3062b) and all 10 colors resolve in BrickLayer. The 10 colors' LDraw codes are their Rebrickable ids (4, 1, 14, 2, 0, 15, 71, 72, 25, 70). No part or color failed.
- **Building and validating:** a car, a duck and a house were written with `init`, an edited `build.py` and `validate`. Each `validate` took about 2 s, start-up included, and all three passed.
- **Rendering out of the box fails headless.** `brickwright external render` and `finalize` use pygfx on wgpu. On this server, wgpu's only adapter is llvmpipe through OpenGL, which lacks the `float32-filterable` feature pygfx asks for: `RuntimeError: Request device failed (3): Validation Error ... Unsupported features were requested: FLOAT32_FILTERABLE`. `finalize` then clears its exports. Without `LDRAW_LIBRARY_PATH`, `finalize` skips the render and writes the `.ldr`.
- **Two ways work on a CPU:**
  - BrickWright's own numpy rasterizer (`brickwright.tools.software_rendering`) renders a 640 px view in 0.12 s. But its flat headlight shading on black makes black parts hard to see.
  - The same pygfx pipeline with `float32-filterable` turned off (`pygfx.renderers.wgpu.enable_wgpu_features("!float32-filterable")`) renders on llvmpipe in 1.3 to 1.8 s a view, at 1280 px. Legoin uses this one. It adds an orthographic camera at 30° from the front left, a fixed key light and LDraw's edge lines, so the pictures read like an instruction booklet. Each is cropped and scaled to 640 px.
- **The library's numbers:**
  - Validating, exporting and rendering all 18 builds took 29 s, about 1.6 s a build, with 560 MB of memory at most.
  - The previews are 12 to 25 KB each (320 KB for all 18), and the `.ldr` files 370 to 640 bytes: plain LDraw part lines under a short header.
- **Renders aren't byte-for-byte repeatable.** On llvmpipe, two runs differ by a few pixels. The script keeps a build's files while its source still matches the manifest, so previews don't churn.

### Measured with Claude

- **Method:** the same as ADR-0009's, on 2026-11-25.
  - Requests went through `POST /v1/scans/{id}/ideas` in a container of legoin-api's branch for issue #19, before the one-round cap and the "all but one" rule.
  - A counting wrapper on the SDK client refused any call past 8.
  - The scan: the 31-piece test photo, scanned by the v0.2.0 models into the same 25 lines.
  - Sonnet 5.5 at `low`.
- **The cached prefix** grew from 2,910 to 4,577 tokens: about $0.0005 a call to read, $0.011 once to write.
- **Three requests, 8 API calls, $0.083:**

| Request | Latency | API calls (tool rounds) | Cost | Library picks |
| --- | --- | --- | --- | --- |
| `vehicles`, `en` | 13.3 s | 2 (1), the first call writing the cache | $0.029 | none |
| `animals`, `id` | 12.9 s | 3 (2) | $0.023 | the snail, with 1 of its 3 pieces owned |
| `surprise`, `en` | 15.3 s | 3 (2) | $0.031 | none |

- **Per call:** tool rounds took 3.8 to 4.7 s warm and answers 4.3 to 5.4 s, as in ADR-0009. The time over 10 s came from the second tool round and the first call's cache write.
- **Not yet measured:** the one-round cap and the "all but one" rule, which came after. From the per-call times, a warm request with one round would take about 8 to 9.5 s. legoin-api's PR for #19 asks for a few more real requests to confirm it.

## Consequences

- Positive: every template idea has a picture of a build that's been checked, and its steps and parts match that build.
- Positive: no cost and no latency per request. The service only serves small static files, cached for a year, and its image, start-up and CI don't change.
- Positive: Claude's ideas get a picture when the person's pieces fit a library build. The library summary sits in the prompt's cached prefix, so it costs a cache read.
- Positive: anyone can open the `.ldr` in an LDraw viewer, such as LDView or Studio, and turn the build around.
- Negative: only library builds have pictures. Claude's own builds, most of its answers, have none.
- Negative: the library is small, 18 builds, and grows by hand. Adding one needs the authoring setup: uv, the pinned packages and the LDraw parts library.
- Negative: the builds are 2-stud-wide micro models with 1×1 details off center, so some are more suggestive than exact.
- Negative: twelve templates changed, seven of them their parts, so legoin-web's mock and its copy of the templates have to change with them.
- Negative: BrickWright is v0.1.0 research code. Rendering relies on turning off a pygfx feature, and the edge lines read ldraw2mesh's scene. A new version may need the script to change, so the versions are pinned.
- Negative: more licences to credit: CC BY for the LDraw parts library, and MIT for BrickNet, BrickLayer and BrickWright.

## Pros and Cons of the Options

### A library of checked builds, made offline

- Pro: real, checked pictures for every template, at no cost or latency per request.
- Pro: the service only serves files. BrickLayer and the LDraw parts library stay on the authoring machine.
- Con: a fixed set. Free ideas get no picture.

### BrickWright writes a build for each idea, in real time

- Pro: every idea would get a picture of its own build.
- Con: BrickWright's loop takes minutes and many model calls per build, far over the 10 s budget, and costs money per idea.
- Con: the server would need the LDraw parts library, pygfx and a renderer at runtime.

### Claude writes BrickLayer code for its own ideas

- Pro: Claude's ideas would get pictures too.
- Con: one shot rarely passes validation: BrickWright reports baseline agents at 9% to 69% valid without its loop. Each repair is another model call.
- Con: running model-written code on the server needs a sandbox, and rendering needs the parts library at runtime.

### No pictures

- Pro: nothing to build or maintain.
- Con: ideas stay text only, and the templates that can't be built stay that way.

## References

- [API contract](../api/contract.md): the `preview` and `buildId` fields on `Idea`
- [ADR-0008](0008-backend-stack.md): the backend stack and the `IdeaGenerator`
- [ADR-0009](0009-anthropic-claude-writes-build-ideas.md): Claude writes ideas, with the templates as fallback
- [BrickWright and BrickLayer](https://github.com/AvaLovelace1/brickwright), and the paper, [arXiv:2610.31651](https://arxiv.org/abs/2610.31651)
- The [LDraw.org Parts Library](https://library.ldraw.org/) and its [licence](https://www.ldraw.org/legal-info)
- legoin-api issues #17, #18 and #19, and legoin-web issue #67
