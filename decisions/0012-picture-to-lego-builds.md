# ADR-0012: Picture-to-LEGO Builds: Claude Places Pieces Through Tools, in a Budgeted Background Job

- Status: Proposed
- Date: 2026-11-25
- Deciders: Legoin team
- Issue: #25

## Context and Problem

*Updated with a scope addition: previews of ideas, in "Idea previews" below.*

A teammate asked about [BrickWright](https://github.com/AvaLovelace1/brickwright)'s image-to-LEGO: an agent turns a text or image prompt into a physically valid brick build. The maintainer now wants it in Legoin. A person uploads a picture, such as a car, an animal or a house. Claude designs a LEGO build of it from the app's catalogue, and the person gets a rendered picture, an `.ldr` model, a parts list, steps, and what they own or are missing against one of their scans.

[ADR-0010](0010-library-of-checked-builds.md) turned down live BrickWright generation for ideas. It takes minutes and many model calls per build, it needs the LDraw parts library and a renderer at runtime, and BrickWright's agent writes Python that has to run on the server. Those are still the risks here:

- the cost per build;
- minutes of latency;
- running AI-written code on a shared production host.

How should Legoin design builds from pictures so that a build is valid, nothing the model writes is run, the cost has a hard ceiling, and scans and ideas never wait for it?

## Decision Drivers

- **No model-written code runs anywhere.** The production host also runs the API's scans and other sites.
- **Physically valid builds** a person can follow: no collisions, nothing floating, one connected assembly, from the 13 catalogue parts and 10 colors only.
- **A hard ceiling on spending,** with the team's Anthropic key: a daily budget, a limit per account and a cap per job. Spending has to be counted from real usage and survive restarts.
- **Scans and ideas come first.** Scans have a 5-second budget and ideas 10 seconds ([contract](../api/contract.md), "Rules"). A build of a minute or more must not slow them down or take their memory.
- **Faithful enough to recognise,** in both of the app's languages ([ADR-0004](0004-english-default-with-indonesian.md)).
- **Privacy.** The picture leaves the server for Anthropic, so the person must know before uploading. It's kept like a photo ([ADR-0008](0008-backend-stack.md)) and deleted with the build.
- **Optional.** The base install, the container image's default and CI stay as they are, without BrickLayer, a renderer or the 620 MB parts library.

## Considered Options

How Claude makes the build:

1. **Strict tools:** Claude places and removes catalogue pieces on a stud grid, asks for checks and renders, and finishes; the backend applies each call with BrickLayer
2. **BrickWright's own loop:** Claude writes BrickLayer code in `build.py`, which the backend runs in a sandbox
3. **One shot:** Claude answers with a JSON list of pieces, and the backend checks it once
4. **Library builds only** (ADR-0010): no new builds, pick the closest checked build

Where the job runs:

1. **A queue in the API, with each job in a worker process** of its own, one at a time, with memory and time limits
2. **In the API process**, on a thread, like scan jobs
3. **A separate service** with its own queue, such as Celery with Redis

The model: Claude Sonnet 5.5 or Claude Opus 5.5, measured in a spike.

## Decision

Chosen: **strict tools, in a budgeted background job that runs each design in its own worker process,** with **Claude Opus 5.5 at `medium` effort**. Nothing Claude writes is run: it can only name catalogue pieces and grid positions, and BrickLayer checks the result. In the spike, Opus 5.5 made 5 valid builds of 5 pictures, at a median of $0.14 and 48 seconds each.

legoin-api#28 implements it in `src/legoin_api/picture/`, and legoin-web#75 adds the page.

### The agent

BrickWright's guidance for its agents (`general_instructions.md`) is kept, translated from code to tools:

- **The requirements:** no collisions, nothing floating, one connected assembly, and faithful to the subject;
- **the method:** plan the defining features first, build in batches, validate and repair, and look at renders before finishing, since a valid build isn't proof that it looks right.

Claude gets a cached system prompt with that guidance, the grid's conventions, the 13 parts with their sizes and tops, the 10 colors, one worked example and a few small-scale tricks (wheels, eyes, windows, roofs). The picture comes after it, with one line naming the language. The person types nothing, so no text of theirs reaches the prompt. The prompt also tells Claude to ignore any instructions written in the picture.

The tools are strict, with `tool_choice: auto`, since Opus 5.5 and Sonnet 5.5 answer a forced tool choice with a `400`:

| Tool | Input | What the backend does |
| --- | --- | --- |
| `place_pieces` | pieces, each a catalogue `part` and `color` (enums), whole `x`, `y` and `z` (in studs and plates), and `rotation` of 0, 90, 180 or 270 | Adds each piece that's in the catalogue and inside the grid (20 × 20 studs, 48 plates high), up to 60 in all, and answers each one's id or why it couldn't be placed |
| `remove_pieces` | piece ids | Removes them |
| `validate` | nothing | BrickLayer's checks, the ones BrickWright's `TestContext` runs: collisions, floating pieces and connected assemblies, naming the pieces at fault. Also lists the pieces, the size and the turns left |
| `render` | a view: `front`, `back`, `left`, `right`, `top`, `iso` or `all` | Renders the build and answers with the image, for Claude to compare with the picture |
| `finish` | a title, a summary and 2 to 8 steps, in the request's language | Ends the job, but only with a build that passes the checks and that Claude has seen in a render since its last change |
| `reject_picture` | `no_subject` or `unsuitable` | Ends the job with `picture_unusable` or `picture_refused` |

The grid is BrickLayer's own: `x` to the viewer's right as they face the model, `y` toward the back, `z` up in plates. A piece is placed by its corner cell with BrickLayer's `LegoBuild.add`, so the checks and the `.ldr` are exactly BrickLayer's. The `brickwright` package itself isn't installed, since it brings litellm. Its two helpers the renderer needs are a few lines, copied.

**The request:**

- thinking is adaptive, the only setting Opus 5.5 takes;
- effort is set, at `medium`;
- `max_tokens` is 8,000 per call;
- server-side fallbacks are on (`server-side-fallback-2026-07-01`, `fallbacks: "default"`), as in ADR-0009;
- the system prompt carries a cache breakpoint, and top-level automatic caching follows the growing conversation;
- every turn goes back whole, thinking blocks included;
- `stop_reason` is checked first: `refusal` ends the job with `picture_refused`, and a reply cut off at `max_tokens` is told to place smaller batches.

### The job

It works like a scan job, polled by the client:

1. `POST /picture-jobs` takes a picture and a language, and answers `202` with a job at the stage `queued`.
2. The job moves through `designing`, `checking` and `rendering`, then ends `done` with a saved build or `failed` with a code.
3. `DELETE` cancels it. The contract's new section 13 describes the routes and shapes.

- **One job at a time.** A single dispatcher thread in the API process takes jobs from a bounded queue (3 by default, counting the running one). A full queue answers `503 unavailable` with `Retry-After`.
- **Each job runs in a worker process of its own** (`python -m legoin_api.picture.worker`), started fresh for that job and gone when it ends:
  - it runs at `nice` 10, with llvmpipe and NumPy kept to 2 threads, so it can't starve scans, which run torch on 4 threads in the API process;
  - **memory:** the dispatcher reads the worker's resident memory twice a second and kills it past `LEGOIN_PICTURE_WORKER_MEMORY_MB` (1,024 MB by default; it peaked at 546 MB in the spike). The worker also sets an 8 GiB address-space limit on itself as a backstop. That limit can't be the cap: llvmpipe reserves about 5 GB of address space it never uses;
  - **time:** the dispatcher kills a worker that's still running 6 minutes after it started. The job fails with `timeout`;
  - it reports its stages and every call's cost as JSON lines on its stdout. The dispatcher alone writes the database.
- **The worker owns nothing:** the API key comes from its environment, and it writes the picture build's preview and `.ldr` to a scratch folder. When the job is `done`, the dispatcher moves them to their final names and saves the build. A failed or cancelled job leaves nothing behind.
- **Scans and ideas don't share it.** The scan worker, the ideas route and the picture dispatcher are separate, and the picture worker runs in another process. The ideas flow never waits for a picture job.
- **Ended jobs are kept 10 minutes,** as scan jobs are. A restart forgets queued and running jobs. Their builds were never saved, and the client gives up after 3 failed polls.

### Budgets

All of them are configuration, listed in the README and `.env.example`:

| Variable | Default | What it limits |
| --- | --- | --- |
| `LEGOIN_PICTURE_BUILDS` | `off` | Picture builds are on only when this is `anthropic`, `ANTHROPIC_API_KEY` is set, the `picture` extra is installed and the LDraw parts library is found. Otherwise the routes answer `503 unavailable` and the quota says `off`. The service logs what's missing; it still starts, so scans keep working. |
| `LEGOIN_PICTURE_MODEL`, `LEGOIN_PICTURE_EFFORT` | `claude-opus-5-5`, `medium` | The model and effort |
| `LEGOIN_PICTURE_DAILY_BUDGET_USD` | `3.00` | Dollars a day (UTC) for every picture job together |
| `LEGOIN_PICTURE_DAILY_LIMIT` | `2` | Picture jobs a day per account |
| `LEGOIN_PICTURE_JOB_COST_USD` | `0.50` | Dollars one job may spend |
| `LEGOIN_PICTURE_JOB_SECONDS` | `300` | Seconds one job may design for |
| `LEGOIN_PICTURE_JOB_CALLS` | `24` | API calls one job may make |
| `LEGOIN_PICTURE_QUEUE_SIZE` | `3` | Jobs waiting or running |
| `LEGOIN_PICTURE_WORKER_MEMORY_MB` | `1024` | The worker's resident memory |

How they're kept:

- **Cost comes from real usage.** Each response's `usage` is priced with the model's per-million-token prices:

  | Model | Input | Output | Cache read | Cache write (5 min) |
  | --- | --- | --- | --- | --- |
  | Opus 5.5 | $4 | $20 | $0.20 | $5 |
  | Sonnet 5.5 | $2 | $10 | $0.10 | $2.50 |

  A response served by a fallback model is priced at the dearer of the two models, and an unknown model at Claude Opus 5's prices ($5 and $25), so spending is never under-counted.
- **Before every call,** the worker works out the most the call could cost: the last call's input plus 4,000 tokens, all written to the cache, and 8,000 output tokens. It calls only if that fits under the job's cap.
- **The daily budget is reserved per job.** A job is accepted only if today's spending, plus the cap of every job still queued or running, plus its own cap fits under the daily budget. So the budget holds even if every job spends its whole cap. Otherwise `POST` answers `503 picture_budget` with `Retry-After` until midnight UTC.
- **A worker killed in the middle of a call** is charged that call's worst case, since the call may still be billed.
- **Spending is kept in SQLite** (`<data dir>/picture.db`): every call's tokens and cost, and every job with its account, day and whether it counts toward the limit. It survives restarts and deploys.
- **Per account,** a job counts toward the daily limit as soon as it's accepted. It's given back if the job fails with `unavailable`, `internal` or `picture_budget`, or is cancelled while queued. Over the limit, `POST` answers `429 picture_limit` with `Retry-After`.
- **When a job's cap is hit** (cost, calls or time), the job ends with the last valid build Claude made, under a plain title, summary and steps that list the pieces level by level (`plain: true`). With no valid build, it fails with `no_valid_build`.
- **`GET /health`** reports `pictureBuilds` (`anthropic` or `off`), and `pictureModel` when it's on.

### Results and ownership

- **Each build belongs to an account,** as scans do since [ADR-0011](0011-accounts-with-email-and-password.md), through the same `CurrentAccount` dependency. That's the one seam that says whose builds these are. When this work began, ADR-0011 was still a proposal and scans were kept by `Legoin-Device-Id`; it was accepted meanwhile, so picture builds use accounts from the start.
- **A build keeps:**
  - its title, summary, steps, parts and pieces;
  - the picture, as a JPEG with no EXIF metadata, under a random 128-bit name;
  - the preview, a 640 px WebP rendered from the library's angle, and the `.ldr`, with the library's credit header naming Claude as designer.

  Files are served at `/v1/picture-files/{name}`, with no header, like photos. Deleting the build deletes them.
- **`used` and `missing` aren't kept.** `GET /picture-builds/{buildId}?scanId=` runs the existing `cek_inventori` against the scan as it is at that moment.
- **Credits:** the same as the library's pictures (ADR-0010), wherever the pictures appear.

### Privacy and safety

- **Anthropic receives the picture and the language,** nothing else: no account id or email. The web app says so on the upload screen, before a picture is chosen, in both languages.
- **The picture is checked like a scan's photo:**
  - JPEG, PNG or WebP, at most 10 MB and 4,096 × 4,096 pixels;
  - upright, with its EXIF metadata removed;
  - sent at most 1,024 px on its longest side.
- **A picture Claude can't use** ends with `picture_unusable`. One it declines, or judges unsuitable for a family building app, ends with `picture_refused`.

### Optional dependencies

- **The `picture` extra** pins what `scripts/builds.py` renders the library with:
  - `bricklayer-sdk` 0.1.0;
  - `ldraw2mesh` 0.1.1 and `gltflib` 1.0.13;
  - `pygfx` 0.18.0, `rendercanvas` 3.0.1 and `wgpu` 0.33.0.
- **Rendering needs the LDraw parts library** (about 620 MB unzipped), outside the repository, at `LDRAW_LIBRARY_PATH`. It uses the same headless workaround as ADR-0010: pygfx on llvmpipe, with `float32-filterable` turned off.
- **The host needs Mesa's llvmpipe,** which wgpu falls back to without a GPU (`libgl1-mesa-dri` and `libegl1` on Ubuntu).
- **What stays the same:** `uv sync` without extras, the container image's default and CI. A deployment that wants picture builds adds `--extra picture`, the parts library and the variables above.
- **Tests** use fakes for Claude, BrickLayer and the renderer. A slow test, skipped without the extra, runs the real checks.

### The spike (Phase 0)

It ran on 2026-11-25 on the project's server (AMD EPYC, 16 cores, no GPU, Ubuntu 24.04), in a scratch folder outside the repositories, before any pull request.

- **The pictures:** five simple pictures drawn for it with Pillow, in flat colors: a red car, a yellow duck on water, a house with a red roof, a blue boat with a funnel, and a green turtle. They're not committed.
- **The code:** the same loop and tools as above. The duck and the boat asked for Indonesian, the others English.
- **A spend guard** refused any call whose worst case would take the spike's total past $5. The key came from the environment and was never printed.
- **Effort:** `medium` for both models. It's the documented starting point for multistep tool use on Sonnet 5.5, and the default on Opus 5.5.
  - A first run of Sonnet 5.5 at `low`, on the car, made a 9-piece car with no wheels.
  - It never looked at a render: it finished as soon as the build was valid.
  - After that run, `finish` was changed to need a render seen since the last change, as BrickWright advises its agents.
  - The table's other runs all ran with that rule.
- **Total spent:** $0.97 for 35 API calls, 11 builds.
- **After the spike,** one live job ran through legoin-api's implementation: the real routes, the worker process and the dispatcher, on the project's server, with a $1.00 daily budget for the test.
  - The house picture, in `id`, went `queued` → `designing` → `rendering` → `done` in 72 s, for 3 calls and $0.227.
  - It came back as 57 pieces with 8 Indonesian steps.
  - The stored spending matched the worker's log.

| Model, effort | Picture | Valid | Pieces | API calls | Input + cache read + cache write / output tokens | Cost | Wall time |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Sonnet 5.5, `low` | car | yes | 9 | 4 | 10 + 17,284 + 7,539 / 1,346 | $0.034 | 17 s |
| Sonnet 5.5, `medium` | car | yes | 20 | 6 | 14 + 43,575 + 11,688 / 3,549 | $0.069 | 34 s |
| Sonnet 5.5, `medium` | duck (`id`) | yes | 10 | 2 | 6 + 9,152 + 3,790 / 2,443 | $0.035 | 22 s |
| Sonnet 5.5, `medium` | house | yes | 29 | 4 | 10 + 27,378 + 5,448 / 3,256 | $0.049 | 27 s |
| Sonnet 5.5, `medium` | boat (`id`) | yes | 24 | 3 | 8 + 17,586 + 4,563 / 2,689 | $0.040 | 24 s |
| Sonnet 5.5, `medium` | turtle | yes | 29 | 5 | 12 + 39,051 + 7,373 / 3,350 | $0.056 | 30 s |
| **Sonnet 5.5, `medium`, median** | | **5 of 5** | 24 | 4 | | **$0.049** | **27 s** |
| Opus 5.5, `medium` | car | yes | 34 | 2 | 6 + 4,984 + 11,441 / 4,589 | $0.150 | 48 s |
| Opus 5.5, `medium` | duck (`id`) | yes | 28 | 3 | 8 + 19,532 + 8,176 / 4,669 | $0.138 | 49 s |
| Opus 5.5, `medium` | house | yes | 42 | 2 | 6 + 9,137 + 7,615 / 4,860 | $0.137 | 48 s |
| Opus 5.5, `medium` | boat (`id`) | yes | 43 | 2 | 6 + 9,152 + 7,377 / 4,370 | $0.126 | 42 s |
| Opus 5.5, `medium` | turtle | yes | 30 | 2 | 6 + 9,137 + 7,526 / 4,958 | $0.139 | 51 s |
| **Opus 5.5, `medium`, median** | | **5 of 5** | 34 | 2 | | **$0.138** | **48 s** |

**How the time and money went:**

- The wall time runs from the picture to a saved preview, rendering included (about 2.5 s for the preview).
- Opus 5.5 placed the whole build in its first turn, validated and rendered it in that same turn, and finished in the second. Its first call wrote about 7,500 tokens to the cache, and its output was 4,400 to 5,000 tokens.
- Sonnet 5.5 took more, shorter turns to repair what `validate` reported.
- Every call came from the requested model; no fallback ran.
- One validation of up to 60 pieces took under 0.6 s, and a four-view render 2.5 to 3.7 s.
- The worker peaked at 546 MB of resident memory with 60 pieces, rendered.

**Quality,** judged on the previews:

- **Opus 5.5** made the more faithful builds: a car with four wheels, a sloped cabin with windows and lights at both ends; a house with a door step, two windows, a chimney and a lawn; a boat with a tall funnel; a turtle with a sloped shell and two eyes. Its steps are longer and say where each piece goes, and its Indonesian reads naturally.
- **Sonnet 5.5's builds** are recognisable but plainer, its car's wheels sit oddly, and two of its English steps say "bbricks".
- **The duck was the weakest for both,** a blocky body with a head.

**The gate:** continue only if the better model's median was at most $1.00 and 5 minutes per build, with at least 4 of 5 builds valid. **Opus 5.5 passed,** at $0.138, 48 s and 5 of 5 valid. Sonnet 5.5 passed too, at a third of the cost.

**The defaults that follow:**

- **Opus 5.5 at `medium`.** At about $0.14 a build, the $3 daily budget pays for about 21 builds a day, more than the course demo needs. Sonnet 5.5 at `medium` is one variable away, at about $0.05.
- **A job's cap of $0.50** is over three times the median, so a build that needs repairs still fits.
- **The 300 s design limit** is six times the median.

### Idea previews

A scope addition from the maintainer: every build idea should show a picture of what it looks like, made with this engine, not only the ideas that are library picks ([ADR-0010](0010-library-of-checked-builds.md)). legoin-api#36 and legoin-web#78 implement it.

**What happens:**

- `POST /scans/{scanId}/ideas` answers as before, around ADR-0009's 10 seconds. Library ideas keep their picture.
- **After it answers,** each idea Claude wrote itself gets a preview job, in the same queue and worker as picture builds:
  - Claude gets the idea as text: its title, summary, steps and listed parts. It builds it with the same tools, system prompt and render-before-finish rule.
  - `place_pieces` refuses any part and color pair the idea doesn't list.
  - The build may use more or fewer of each, up to twice the pieces the idea lists, and 20 at most.
  - It keeps the idea's title, and writes a summary and steps that match the build.
- **A valid build becomes the idea:**
  - its `buildId` (`gen_…`) and `preview`;
  - its steps and summary;
  - `used` and `missing` from `cek_inventori` of the build's own parts, against the scan as it is then.

  The text and the picture never disagree.
- **The idea carries a `previewStatus`:** `pending`, `on_open`, `ready` or `unavailable` (contract section 13, "Idea previews"). The web app shows a placeholder while it's `pending`, polls the routes it already reads, and swaps the picture in with the click-in.
- **Stale sets are dropped.** A preview belongs to its set:
  - asking for ideas again, or a correction that drops the ideas, cancels the old set's queued and running previews and deletes their builds;
  - deleting the scan deletes them too;
  - a preview that finishes for an idea no longer in the scan's set is thrown away.

  "Try other ideas" never stacks up work.
- **It never blocks or fails the ideas request.** A limit, the budget or a failed build leaves the idea text-only, as `unavailable`.

**The configuration:**

| Variable | Default | What it sets |
| --- | --- | --- |
| `LEGOIN_IDEA_PREVIEWS` | `all` | `all` builds the three previews after the ideas are answered; `on_open` only when someone opens that idea's page; `off` makes none |
| `LEGOIN_IDEA_PREVIEW_MODEL`, `LEGOIN_IDEA_PREVIEW_EFFORT` | `claude-opus-5-5`, `medium` | The model and effort |
| `LEGOIN_IDEA_PREVIEW_DAILY_LIMIT` | `9` | Previews each account may start a day: three sets with `all` |
| `LEGOIN_IDEA_PREVIEW_COST_USD` | `0.25` | Dollars one preview may spend; it reserves this much of the daily budget while it waits or runs |
| `LEGOIN_IDEA_PREVIEW_SECONDS`, `LEGOIN_IDEA_PREVIEW_CALLS` | `120`, `10` | One preview's time and calls |

**Budgets and prerequisites:**

- Previews draw on the same `LEGOIN_PICTURE_DAILY_BUDGET_USD` as picture builds. Each reserves its cap until it ends, and each call is checked against its worst case, as for picture builds.
- A preview's calls ask for at most 4,000 output tokens, since its turns are short. A per-preview cap of $0.25 then still allows three to four calls in the worst case.
- Previews need what picture builds need (the key, the `picture` extra, the LDraw parts library) and ideas from Claude. They don't need `LEGOIN_PICTURE_BUILDS` to be on.

**Priority: picture builds first.** The worker runs one job at a time.

- A picture build was asked for on purpose, counts against a small quota, and has someone watching its progress.
- A preview is extra on an idea that's already useful as text, and it's short (about 15 s).
- So queued picture builds run before queued previews, and previews run in the order they were asked for.
- A running preview isn't stopped for a picture build. Previews are short, so a picture build waits about 15 s more at worst.

**The default is `all`.** The spike below measured a median preview at $0.036 and 14 s, about a tenth of the decision rule's $0.35 and 2 minutes.

- **Cost:** a whole set of three cost $0.12, less than one picture build.
- **Wait:** it arrives in about 45 s, while the person is still reading the cards.
- **`on_open`** would cost about a third as much, but the cards stay text-only, and the person waits about 15 s on each idea's page instead.
- **The daily budget** bounds both modes, and `on_open` is one variable away for a tighter one.

#### The spike for idea previews

It ran on 2026-11-25, on the same server, after the picture spike and before this was built.

- **The ideas:** legoin-api ran with the v0.2.0 models and Claude ideas (Sonnet 5.5 at `low`, the defaults), and scanned the detector's 31-piece test photo (`photos_31_IMG_20201127_192001.jpg`). `surprise` in `en` gave three ideas Claude wrote itself, each 4 or 5 pieces, with no library pick:
  - "Blue buggy";
  - "Brown bird";
  - "Stone hut".

  That request took 9.9 s and $0.017.
- **The previews:** built in-process with the agent above, at `medium`, with a $0.40 cap each.
- **Timing:** the times leave out the final 640 px render (about 2 s) and a worker's start-up (about 2 s).
- **The spend guard:** the Sonnet run went through the picture spike's spend guard. The Opus run didn't: the maintainer asked for no spend guard on it, and each preview's own cap bounded it.
- **Total:** the spikes spent about $1.39 in all, including the live picture job above.

| Model, `medium` | Idea | Listed pieces | Built pieces | Valid | API calls | Input + cache read + cache write / output tokens | Cost | Time | Looks like the idea |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Sonnet 5.5 | Blue buggy | 5 | 5 | yes | 2 | 6 + 4,795 + 5,976 / 788 | $0.023 | 11 s | partly: red rounds on top, not wheels |
| Sonnet 5.5 | Brown bird | 4 | 4 | yes | 2 | 6 + 9,212 + 1,332 / 625 | $0.011 | 7 s | yes, blocky |
| Sonnet 5.5 | Stone hut | 5 | 5 | yes | 3 | 8 + 15,035 + 1,646 / 878 | $0.014 | 12 s | partly: half a roof |
| **Sonnet 5.5, median / set of 3** | | | | 3 of 3 | 2 | | **$0.014** / $0.048 | **11 s** / 30 s | |
| Opus 5.5 | Blue buggy | 5 | 7 | yes | 2 | 6 + 4,795 + 6,235 / 1,020 | $0.053 | 14 s | yes: four red wheels under the body |
| Opus 5.5 | Brown bird | 4 | 4 | yes | 2 | 6 + 9,212 + 2,174 / 1,094 | $0.035 | 15 s | yes, blocky |
| Opus 5.5 | Stone hut | 5 | 8 | yes | 2 | 6 + 9,244 + 2,544 / 1,097 | $0.036 | 14 s | yes: walls, two-slope roof, chimney |
| **Opus 5.5, median / set of 3** | | | | 3 of 3 | 2 | | **$0.036** / $0.124 | **14 s** / 42 s | |

**Which model:** Opus 5.5 made the more faithful previews.

- **Buggy:** Opus stood the red rounds under the body as four wheels; Sonnet put them on top.
- **Hut:** Opus built walls under a two-slope roof; Sonnet's slope covers half.
- **Bird:** a blocky body with a head and a beak, the same from both.

Every preview was valid and used every pair its idea listed. Opus added pieces where the idea needed them (two more wheels, two more walls and a second slope), which is what the source-of-truth rule is for.

**The decision rule:** build `all` or `on_open` only if the better model's median preview costs at most $0.35 and takes at most 2 minutes, with at least 2 of 3 valid. **Opus 5.5 passed:** $0.036, 14 s, 3 of 3.

## Consequences

- Positive: a person gets a build of their own picture, checked for collisions, support and connection, with a picture, an `.ldr`, its parts and steps, and what they own or are missing against a scan.
- Positive: nothing the model writes is run. Its only output that does anything is a list of catalogue part and color enums with whole grid positions, which the backend validates before BrickLayer sees it.
- Positive: spending has a hard ceiling that survives restarts: a daily budget reserved per job, 2 jobs per account a day, and a cap per job checked before every call. At the defaults, a day costs at most $3.
- Positive: scans and ideas keep their budgets. The design runs in another process, one at a time, at a lower priority, with its memory and time watched.
- Positive: the feature is optional. The base install, the container image's default and CI don't change, and the routes answer `503` when it's off.
- Negative: about a minute per build, and about $0.14 with Opus 5.5. That's why it's a job with a quota, not part of the 15-second flow.
- Negative: builds are small and blocky, from 13 parts and no sideways studs, so some subjects, such as the duck, only just read as what they are.
- Negative: the steps are Claude's own words. BrickLayer checks the build, not whether the steps match it. Plain level-by-level steps are only used when a cap ends the job.
- Negative: the picture is sent to Anthropic, which the person has to accept before uploading.
- Negative: a deployment that wants it needs the `picture` extra (about 245 MB of packages, 117 MB of them BrickLayer's part data), Mesa's llvmpipe on the host (`libgl1-mesa-dri` and `libegl1` on Ubuntu, already on the project's server), the 620 MB LDraw parts library, and the variables above.
- Negative: queued and running jobs live in memory, as scan jobs do, so a restart loses them, and the service stays one process.
- Negative: BrickLayer 0.1.0 is research code, and rendering relies on turning off a pygfx feature, as in ADR-0010. The versions are pinned.
- Negative: five drawn pictures is a small spike. Real photos, with backgrounds and several subjects, are unmeasured.
- Positive: with idea previews, every idea Claude writes gets a picture of a checked build, and its steps and parts are that build's.
- Negative: at the default `all`, each request for ideas costs about $0.12 more in previews, on top of about $0.02 for the ideas themselves, until the daily budget or the account's limit stops them.
- Negative: an idea's steps and parts can change after it's first shown, when its preview is built. The web app shows the new ones with the picture.
- Negative: the idea-preview spike was three small ideas from one scan. Bigger ideas, up to 15 pieces, are unmeasured.

## Pros and Cons of the Options

### Strict tools, applied with BrickLayer

- Pro: nothing the model writes is run; its tool inputs are enums and whole numbers that the schema and the backend both check.
- Pro: every check is BrickLayer's own, so a saved build is as valid as the library's.
- Pro: measured at a median of 2 calls and $0.14 with Opus 5.5.
- Con: Claude can't use loops or helpers to build repeated parts, so it lists every piece, which costs output tokens.

### BrickWright's own loop

- Pro: the published method, with helpers, sub-models and BrickLayer's whole API.
- Con: it runs model-written Python on the production host, which would need a real sandbox (a container or a VM per job), more than this host and team can operate.
- Con: the `brickwright` package brings litellm and its dependencies.

### One shot

- Pro: one call, the cheapest.
- Con: no repair and no look at the result. ADR-0010 cites BrickWright's measurements: baseline agents make valid builds 9% to 69% of the time without the loop.

### Library builds only

- Pro: no cost per request, already built.
- Con: 18 builds don't cover what people photograph; this isn't a build of *their* picture.

### A worker process per job

- Pro: memory, time and priority belong to one process that can be killed. Rendering's heavy imports never load into the API.
- Con: about a second to start Python and import BrickLayer and pygfx per job.

### In the API process, on a thread

- Pro: no process to start.
- Con: a job that grows in memory or hangs takes the API, and its scans, with it; Python threads can't be killed.

### A separate service

- Pro: jobs survive restarts and could run on another host.
- Con: Redis and a worker service to deploy and watch, for a feature limited to a few builds a day.

### Claude Opus 5.5

- Pro: more faithful builds in two calls, detailed steps and natural Indonesian.
- Con: about three times Sonnet 5.5's cost per build ($0.14 against $0.05), and 48 s against 27 s.

### Claude Sonnet 5.5

- Pro: valid every time in the spike, at $0.05 and 27 s.
- Con: plainer builds and steps, with more repair turns.

## References

- [API contract](../api/contract.md), section 13, "Picture builds" (Proposed)
- [ADR-0008](0008-backend-stack.md): the backend, scan jobs and photos
- [ADR-0009](0009-anthropic-claude-writes-build-ideas.md): Claude, its request settings and the key
- [ADR-0010](0010-library-of-checked-builds.md): BrickLayer, the rendering workaround and the credits
- [ADR-0011](0011-accounts-with-email-and-password.md): accounts, which own picture builds
- [BrickWright and BrickLayer](https://github.com/AvaLovelace1/brickwright), and the paper, [arXiv:2610.31651](https://arxiv.org/abs/2610.31651)
- Anthropic's [pricing](https://platform.claude.com/docs/en/about-claude/pricing), [tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), [prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) and [refusals and fallbacks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
- legoin-api#28 and legoin-web#75
