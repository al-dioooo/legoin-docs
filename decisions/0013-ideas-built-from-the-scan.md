# ADR-0013: Ideas Built From the Scan: Claude Builds Each Set From the Scan's Photo and Pieces, Three at Once

- Status: Proposed
- Date: 2026-11-25
- Deciders: Legoin team
- Issue: #27
- Supersedes: [ADR-0012](0012-picture-to-lego-builds.md)

## Context and Problem

[ADR-0012](0012-picture-to-lego-builds.md) made picture-to-LEGO a page of its own. A person uploaded any picture, and Claude built its subject from the whole catalogue, ignoring their bricks. Ideas that Sonnet wrote as text also got a preview, built afterwards from the idea's words, without the photo.

The maintainer says image-to-LEGO was always meant to **continue the scan**. After a scan, Claude should take the scan's photo and the pieces the scan found, and build models with tool calls, from those pieces only. Neither the page nor the previews do that.

How should a scan's ideas be built from its photo and pieces? They have to come quickly enough to wait for on the ideas page, fit the daily budget, and always leave the scan with ideas.

## Decision Drivers

- **The person's own pieces only:** the part and color pairs the scan found, as the person corrected them, within their counts. The tools must enforce this, not the prompt alone.
- **Valid builds, no model-written code,** as in ADR-0012: strict tools, every build checked by BrickLayer.
- **Waiting time.** The person is on the ideas page. The first idea and the whole set should come as soon as they can.
- **Cost.** $3 a day for every account together, counted from real usage.
- **A scan always gets ideas,** whatever is off, spent or broken.
- **A shared production host.** Memory stays bounded, and the API can be deployed before builds are turned on.

## Considered Options

How to make a set's three builds:

1. One conversation that builds the three models one after another.
2. Three conversations at the same time, in one worker process, each asked for a different kind of thing.
3. Three jobs in the queue, run one after another.
4. Three worker processes at the same time.

## Decision

Chosen option: **"Three conversations at the same time, in one worker process"**, with each build after the first starting 5 seconds later. In the spike, this got the whole set in 50 seconds for $0.30. One conversation took 105 seconds for $0.32.

### What happens

1. `POST /scans/{scanId}/ideas` plans the set's three slots, one per theme slot as before:
   - **A library build** ([ADR-0010](0010-library-of-checked-builds.md)) fills a slot when the person owns every piece of one in the slot's theme, or all but one, and hasn't seen it. That's the rule Claude followed when it wrote ideas, now applied in code. It's an idea with its picture at once, at no cost.
   - **Every other slot is built.** The answer lists it in `building`, with the id its idea will have.
2. The set's builds are **one job** in the existing queue, run by the existing worker process (ADR-0012's dispatcher, runner, memory and time limits). In the worker, each build is a conversation of its own with Opus 5.5 at `medium`, in a thread. The builds share one renderer and one set of BrickLayer checks, behind a lock.
3. Each conversation gets:
   - the cached system prompt;
   - the scan's stored photo (upright, at most 1,024 px, a JPEG with no metadata);
   - the scan's pieces from `to_inventory`, with the person's corrections, ending at a cache breakpoint;
   - what to build.

   Within a theme, the three builds are asked for three different kinds of thing, such as a car, something that flies, and a boat or a train, so they differ without seeing each other. With `surprise`, each slot has its own theme. "Try other ideas" passes the titles the person has already seen.
4. The tools are ADR-0012's: `place_pieces`, `remove_pieces`, `validate`, `render`, `finish`. `reject_picture` is gone, since a scan photo is never off-topic.
   - **`place_pieces` refuses** any pair the scan didn't find, and any piece past its count ("the person has only 2 white brick-1x1, and all are in the build").
   - `validate` lists the pieces left.
   - A build uses at most 40 pieces, and never more than the scan has.
5. **As soon as one build finishes,** the worker reports it, and the API makes it the idea:
   - the title, summary and steps Claude wrote;
   - a difficulty from its piece count (`easy` up to 8, `medium` up to 16, `hard` above, like the library's);
   - its `buildId` (`gen_…`) and preview;
   - `used` from `cek_inventori` of its parts against the scan as it is then. `missing` is empty.

   The idea leaves `building` and joins `items`. The others keep building.

### Pending ideas

The answer lists the slots still being built in `IdeaSet.building`, as `{ id, theme }` (contract section 13). They aren't `Idea`s with a status, so a client never sees an idea without its text.

The web app shows a placeholder card for each, with legoin-web#81's placeholder: a dashed plate with pulsing studs. It reads `GET /scans/{scanId}` every 3 seconds while `building` is there, and the card clicks in when its idea arrives, following the motion ADR. An idea still building can't be opened.

### When builds can't run: the fallback

Ideas are written as text by today's generator (Sonnet 5.5 at `low`, ADR-0009, with the templates behind it) in these cases:

- **Up front, for the whole set,** when builds are off on the server, the account has started its sets for the day, the day's budget can't take the set, or too many sets are waiting. The answer is then exactly as before, with no `building`.
- **For one slot,** when its build fails: a cap with no valid build, a refusal, an error from the API, the worker killed, or the job's deadline. The generator writes that slot's idea in the background, with the same id, and it replaces the placeholder.

Every set ends with its ideas. No limit is ever an error for the client.

### The account's limit

ADR-0012's 2 picture builds a day per account were for the page. Now each **set** of builds counts, and an account may start **3 sets a day**:

- A set costs about $0.30 to $0.39, so three sets are about $1 of the $3. At least three people a day get builds.
- "Try other ideas" and a new theme each start a set. Past the limit, the person still gets ideas at once, as text.
- A set cancelled before it starts doesn't count.

### Corrections

A correction that changes the pieces drops the scan's ideas, as it always has. It also cancels the set's builds and deletes the finished ones. The next request for ideas builds again from the corrected pieces, and counts as a set. A deleted scan deletes its builds too.

### The spike

Before any of this was built, it ran on 2026-11-25 on the project's server (AMD EPYC, 16 cores, no GPU, Ubuntu). The setup:

- **Scans** came from the v0.2.0 models, run on two photos:
  - the detector's 31-piece test photo, a real photo;
  - `legoin-demo-09.png`, an AI-made photo of 30 loose bricks. The scan matched its 30-piece list exactly.
- **Builds:** Opus 5.5 at `medium`, with a $0.40 cap, 12 calls and 180 seconds per build, and 4,000 output tokens per call.

| Run | Photo, theme | Valid, within the counts | Set cost | First idea | Whole set | Pieces | Calls |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Three at once | 31-piece photo, `vehicles` | 3 of 3, 3 of 3 | $0.390 | 29 s | 62 s | 10, 14, 14 | 2, 3, 2 |
| One conversation | 31-piece photo, `vehicles` | 3 of 3, 3 of 3 | $0.323 | 35 s | 105 s | 12, 14, 17 | 3, 3, 2 |
| Three at once, 5 s apart | demo-09, `surprise` | 3 of 3, 3 of 3 | $0.303 | 34 s | 50 s | 16, 15, 22 | 2, 2, 2 |

- **The builds:**
  - "Little Blue Car", "Little Propeller Plane" and "Little Steam Locomotive" from the 31-piece photo;
  - "Chunky Red Hot Rod", "Little Red Dog" and "Little Red Cottage" from demo-09.

  Every one read as what it was named. They're blocky at this size.
- **Each build** cost $0.08 to $0.18, with a median of $0.10.
- **Without the stagger,** each of the three conversations wrote the photo and the pieces to the cache, 10,000 to 13,000 tokens each. Five seconds apart, the second and third read them instead: their writes fell to 5,000 to 6,000 tokens, and they read about 10,000.
- **One conversation** was the cheapest without the stagger. But it was the slowest: its context grew to 108,000 tokens read, and each build waited for the one before.
- **Spending:** the spike spent $1.02 in all.

### Why three at once

- **Waiting:** the whole set came in 50 to 62 seconds, against 105 seconds for one conversation, and the first idea about as fast. The person is looking at the placeholders all that time.
- **Cost:** with the stagger, it's as cheap as one conversation ($0.30 against $0.32).
- **Independence:** a build that fails or hits a cap fails alone and falls back alone, and each has its own cap. In one conversation, a cap or an error part-way ends the builds still to come.
- **Memory:** it's one process with one renderer, about the same as a single build. Option 4 would triple the renderer's memory on the shared host. Option 3 would take as long as one conversation.
- **Diversity,** the one thing one conversation does better (it sees what it built), comes from asking each build for a different kind of thing.

### Configuration

The new variables replace ADR-0012's `LEGOIN_PICTURE_*` and `LEGOIN_IDEA_PREVIEW*`, which are no longer read.

| Variable | Default | What it sets |
| --- | --- | --- |
| `LEGOIN_SCAN_BUILDS` | `off` | `anthropic` builds ideas from scans, if the key, the `picture` extra and the LDraw library are there too |
| `LEGOIN_SCAN_BUILD_MODEL`, `LEGOIN_SCAN_BUILD_EFFORT` | `claude-opus-5-5`, `medium` | The model and effort |
| `LEGOIN_SCAN_BUILD_DAILY_BUDGET_USD` | `3.00` | Dollars a day for every build together |
| `LEGOIN_SCAN_BUILD_DAILY_SETS` | `3` | Sets each account may start a day |
| `LEGOIN_SCAN_BUILD_COST_USD` | `0.35` | Dollars one build may spend; a set reserves this much per build while it waits or runs |
| `LEGOIN_SCAN_BUILD_SECONDS`, `LEGOIN_SCAN_BUILD_CALLS` | `150`, `12` | One build's design time and calls |
| `LEGOIN_SCAN_BUILD_MAX_PIECES` | `40` | The most pieces in a build, besides what the scan has |
| `LEGOIN_SCAN_BUILD_STAGGER_S` | `5` | How long each build after the first waits to start |
| `LEGOIN_SCAN_BUILD_QUEUE_SIZE` | `3` | Sets waiting or running |
| `LEGOIN_SCAN_BUILD_WORKER_MEMORY_MB` | `1024` | The worker's resident memory before it's killed |

- **Per call:** at most 4,000 output tokens. That keeps each call's worst case, which is checked before the call, at about $0.15.
- **A whole set** reserves $1.05 of the day's budget until it ends. The worker's deadline is the build time, plus the stagger, plus a minute.
- **The budget's end:** a set is refused once the day's spending and the running sets' reservations leave less than its whole reservation. So the last dollar or so of a day may go unused, and in practice the budget gives about seven full sets.

### What goes

- **The page and its routes:**
  - the web app's Picture to LEGO page, its link and copy;
  - the API's `/picture-quota`, `/picture-jobs` and `/picture-builds`;
  - their shapes and error codes;
  - idea previews, with `previewStatus`.

  `/picture-files` stays, for built ideas' files. The contract goes to version 0.3, keeping `/v1`, since it's a draft and nothing else used them.
- **The data:** on its first start, the API migrates `picture.db` to version 3. That deletes the standalone picture builds and their files, the uploaded pictures included. Their owners could no longer see or delete them. Spending and jobs stay, so the day's budget carries on.

### Privacy

The scan's photo now goes to Anthropic whenever builds run, with the pieces list. Before, only the pieces did, for Sonnet's ideas.

The web app says so before upload, in both languages, on the scan step, where the person chooses their photo. The photo is the same one already stored for the scan ([ADR-0008](0008-backend-stack.md)), and nothing else leaves the server.

### Deploying

- **Merge order:** legoin-web#82's pull request first, then legoin-api#38's.
  - The web app works with both APIs. It handles `building` when it's there, and no longer calls the removed routes.
  - The API lands with builds off until `LEGOIN_SCAN_BUILDS=anthropic` is set, so the old web app keeps working too, except for its picture page, which needs the old routes.
- **Then on the server:**
  - set `LEGOIN_SCAN_BUILDS=anthropic`;
  - keep the `picture` extra and `LDRAW_LIBRARY_PATH`;
  - drop the old `LEGOIN_PICTURE_*` and `LEGOIN_IDEA_PREVIEW*` variables, which do nothing now.

## Consequences

- **Positive:**
  - Every idea Claude makes is built from the person's own pieces: they can build it now, with a picture, an `.ldr` and steps.
  - The first idea comes in about 30 seconds and the set in about a minute, while library ideas show at once.
  - A scan always gets ideas.
  - One feature instead of two, and less code: the page, its routes and the previews are gone.
- **Negative:**
  - A set costs $0.30 to $0.40, against $0.017 for Sonnet's text ideas. The $3 budget gives about seven sets a day across all accounts, and three per account, then text.
  - Ideas take a minute instead of 10 seconds; the person watches placeholders.
  - The scan's photo goes to Anthropic.
  - Builds are blocky at this size. They read as what they're named, but they're small.
  - The standalone picture builds on the server are deleted.

## Pros and Cons of the Options

### One conversation for three builds

- Pro: the cheapest without caching tricks, with one cached prefix.
- Pro: Claude sees what it built, so the builds differ by themselves.
- Con: 105 seconds for the set, as each build waits for the one before. The context grows to more than 100,000 tokens read.
- Con: one cap and one failure path for all three. A problem part-way ends the builds after it.

### Three conversations at once, in one worker (chosen)

- Pro: the set in 50 to 62 seconds, and each build fails and falls back alone.
- Pro: as cheap as one conversation with a 5-second stagger, which lets the second and third read the first one's cached photo and pieces.
- Pro: one process and one renderer: the memory of one build.
- Con: the builds don't see each other, so they're told different kinds of thing to make.
- Con: renders wait for each other behind the lock, at about 2 seconds each.

### Three jobs in the queue

- Pro: no change to the worker.
- Con: as slow as one conversation, and each job writes the cache again.

### Three worker processes

- Pro: no lock around rendering.
- Con: three renderers' memory, about 1.6 GB, on a host that also runs scans and other sites.
- Con: a second concurrency limit to manage.

## References

- [ADR-0009](0009-anthropic-claude-writes-build-ideas.md): Sonnet's ideas, the fallback
- [ADR-0010](0010-library-of-checked-builds.md): the library, and the rule for picking a library build
- [ADR-0012](0012-picture-to-lego-builds.md): the engine this keeps: strict tools, BrickLayer, the worker, the budget
- [Contract](../api/contract.md), section 13
- legoin-api#38 and legoin-web#82
