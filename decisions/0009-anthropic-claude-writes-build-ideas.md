# ADR-0009: Anthropic Claude Writes Build Ideas

- Status: Proposed
- Date: 2026-11-25
- Deciders: Legoin team
- Issue: #17

## Context and Problem

The [API contract](../api/contract.md) left the LLM provider open (open question 1). [ADR-0008](0008-backend-stack.md) shipped legoin-web's 18 idea templates behind an `IdeaGenerator` interface, so a provider could plug in later. The team has now chosen Anthropic. Which model and settings write the ideas, and how do they keep the contract's guarantees? What happens when the model can't answer? How is the API key kept safe, and what does a request cost and how long does it take?

## Decision Drivers

- **What open question 1 asks for:** tool calling for `cek_inventori`, good Indonesian, and an answer within the 10-second budget for ideas.
- **The contract's guarantees hold whatever the model does:**
  - only catalogue parts and colors;
  - `used` and `missing` from the backend's own `cek_inventori`;
  - `exclude` honoured, and `409 no_pieces`;
  - the 30-second deadline and the response shape unchanged.
- **The person always gets ideas.** A refusal, an outage or a slow answer mustn't turn into an error screen.
- **Builds for the person's own pieces,** in both of the app's languages ([ADR-0004](0004-english-default-with-indonesian.md)).
- **A small cost per request** for a course demo, and no key anywhere it could leak: the repositories, CI, the container image or the logs.
- **No change to the routes or the contract,** as ADR-0008 planned.

## Considered Options

The provider is the team's choice: Anthropic. Within it:

Model:

1. Claude Opus 5.5 (`claude-opus-5-5`)
2. Claude Sonnet 5.5 (`claude-sonnet-5-5`)
3. Claude Haiku 5.5 (`claude-haiku-5-5`)

How the ideas come back:

1. Structured outputs for the ideas, with `cek_inventori` as a strict tool the model may call
2. A forced call to an "ideas" tool
3. JSON in free text, parsed by hand

When Claude can't answer:

1. Ideas from the templates instead
2. `503 unavailable`

## Decision

Claude writes the ideas, through the official Anthropic Python SDK, as an `AnthropicIdeaGenerator` behind ADR-0008's `IdeaGenerator`. The templates stay, as the fallback. legoin-api's [PR #16](https://github.com/al-dioooo/legoin-api/pull/16) implements it.

### The request

- **Model:** `LEGOIN_IDEAS_MODEL`, default `claude-sonnet-5-5`. `claude-opus-5-5` takes the same request unchanged.
- **Effort:** `LEGOIN_IDEAS_EFFORT`, default `low`. It's always set in `output_config`, since the models' own defaults differ: `high` on Sonnet 5.5, `medium` on Opus 5.5.
- **Thinking:** adaptive. Neither model accepts `disabled`; effort is the control. Sonnet 5.5's lowest setting, `between_tools`, was measured and was slower, so it isn't used.
- **Server-side fallbacks:** the beta `server-side-fallback-2026-07-01` with `fallbacks: "default"`. A request the model declines is re-run inside the same call on the model Anthropic recommends for that kind of refusal. On Sonnet 5.5 that covers `cyber` and `frontier_llm` declines, which re-run on Claude Sonnet 5. A `bio`, `reasoning_extraction` or `general_harms` decline isn't re-run, and the templates answer.
- **`stop_reason`** is checked before the content is read. `refusal`, `max_tokens` and any other stop that isn't `end_turn` count as no answer.
- **The ideas** come back as structured output (`output_config.format`): one JSON object with an `ideas` array. Each idea has its theme, difficulty, title, summary, steps and full parts list, with part and color ids limited to the catalogue by the schema.
- **`cek_inventori`** is a `strict` tool with `tool_choice: auto`. The prompt asks Claude to check every build with it, all in one turn, before answering. There are at most 3 tool rounds; the last request then offers no tools (`tool_choice: none`). Forcing a tool call isn't possible: Sonnet 5.5 and Opus 5.5 both answer a forced `tool_choice` with a 400.
- **`max_tokens`** is 8,000, room for adaptive thinking and three builds.

### The prompt

- **The stable part comes first and is cached** (`cache_control`, 5-minute TTL). It holds:
  - the catalogue's 13 parts and 10 colors, with their ids, design ids and names in English and Indonesian;
  - the rules;
  - one example.

  With the tool definition, that's 2,910 tokens, the same bytes on every request.
- **The per-request part follows** in the user message: the inventory, the theme of each slot, the locale and the titles already excluded.
- **The rules:**
  - one build per slot, in order;
  - builds made mostly from the person's own pieces;
  - about 3 to 15 pieces and 2 to 5 short steps;
  - different from every idea already seen;
  - in the requested language. Indonesian is written naturally, with Indonesian color names and the hobby's English part names, as the contract says ("brick 2×4 merah").
- **What Anthropic receives:** the inventory, the themes, the locale and earlier idea titles. Never the photo, and never the device id.

### What the service still enforces

Unchanged from ADR-0008, around whichever generator runs:

- **Rule-breaking builds.** A build with a part or color outside the catalogue, a quantity under 1, the wrong theme, or no title or steps gets **one repair request**. Slots still wrong after that are filled from the templates.
- **`cek_inventori`** runs on every final build, against the scan's inventory, and writes `used` and `missing`. Claude's own split is never used.
- `exclude`, `409 no_pieces`, the 30-second deadline and the response shape don't change.

### Time and failure

- **Timeouts:** the SDK's timeout is 20 s per request, with 1 retry. All of Claude's requests share a budget of 80% of the ideas deadline, 24 of the 30 seconds, so the templates can still answer in time.
- **No answer gives the templates.** A timeout, an API error, a refusal the fallback model also declines, a cut-off answer or output that isn't the ideas JSON gives the whole set from the templates. The reason is logged: the error class, status and request id, never the key.
- **`503 unavailable` isn't used for the provider.** The contract allows it, but the person is better served by template ideas than by an error.
- **Logging:** each answer logs its latency, the number of requests and its tokens.

### Switch and key

- **`LEGOIN_IDEAS_GENERATOR`** is `anthropic` or `templates`. It defaults to `anthropic` when `ANTHROPIC_API_KEY` is set, and to `templates` otherwise. Asking for `anthropic` without a key refuses to start.
- **`/health`** reports `ideas`, and `ideasModel` when Claude runs.
- **The key** is `ANTHROPIC_API_KEY`, read from the environment by the SDK only. It's never logged, committed, baked into the image or stored in GitHub secrets.
  - **Locally:** `uv run --env-file <file outside the repo>`.
  - **In a container:** `docker run --env-file`.
  - **CI** has no key. Unit tests use a fake client, and the one test that calls the real API is skipped without a key.

### Measured cost and latency

These were measured with legoin-api's [PR #16](https://github.com/al-dioooo/legoin-api/pull/16) on the project's server, on 2026-11-25. Each request for ideas went through `POST /v1/scans/{id}/ideas`, with a counting wrapper on the SDK client.

- **Opus 5.5:** a real 31-piece photo, 26 inventory lines, scanned by the v0.2.0 models. 6 requests, in `en` and `id` across `vehicles` and `animals`, plus one more by a live test: 14 API calls, about $0.36. Prices: $4 per million input tokens, $20 per million output tokens, $0.20 per million cache reads, $5 per million cache writes.
- **Sonnet 5.5:** a real 31-piece photo from the detector's test split (`photos_31_IMG_20201127_192001.jpg`), scanned by the v0.2.0 models into 25 inventory lines. 8 requests: 4 at `low`, 2 at `medium`, and 2 at `low` with `between_tools` thinking. 16 API calls, $0.17. Prices: $2 per million input tokens, $10 per million output tokens, $0.10 per million cache reads, $2.50 per million cache writes.

Every request took 2 API calls: one turn of three parallel `cek_inventori` calls, then the answer. Every answer came from the requested model. No request needed a fallback model or the templates, and no Sonnet 5.5 request needed a repair.

**The cache was read.** The cached prefix of tools and system prompt is 2,910 tokens on both models. Caches are per model. Opus 5.5's was written once, by the live test, for $0.015. Sonnet 5.5 wrote its own on its first call, for $0.007, and the `between_tools` pair wrote another. Every other call read it.

| Model | Effort | Requests | Request latency, p50 / max | API call latency, p50 / max | Tokens per call: input + cache read / output | Cost per call | Cost per request |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Opus 5.5 | `low` | 4 | 16.5 s / 19.3 s | 8.3 s / 9.8 s | 1,233 + 2,910 / 938 | $0.024 | $0.049 ($0.036–$0.056) |
| Opus 5.5 | `medium` | 2 | 20.7 s / 21.5 s | 10.7 s / 12.2 s | 1,476 + 2,910 / 1,189 | $0.030 | $0.061 |
| **Sonnet 5.5** | **`low`** | 4 | **9.6 s / 10.3 s** | 5.0 s / 6.3 s | 1,014 + 2,910 / 705 | $0.010 | **$0.020** ($0.018–$0.025) |
| Sonnet 5.5 | `medium` | 2 | 10.7 s / 10.7 s | 5.4 s / 7.0 s | 1,076 + 2,910 / 786 | $0.010 | $0.021 |
| Sonnet 5.5 | `low`, `between_tools` | 2 | 12.8 s / 13.5 s | 6.0 s / 8.5 s | 1,063 + 2,909 / 732 | $0.012 | $0.023 ($0.021–$0.025) |

The highest cost in a range includes the one-time cache write. On Sonnet 5.5 at `low`, the tool round took 3.2–4.6 s and the answer 5.5–6.3 s; on Opus 5.5 the answer alone took 7.3–9.5 s. The Opus live test, at `low` on the contract's 4-piece inventory, took 11.1 s.

**Quality.** Every Sonnet 5.5 build kept to the catalogue and to its slot's theme without a repair. No Opus 5.5 build came from the templates either.

- All 12 Opus 5.5 builds at `low` used only the person's own pieces (`missing` empty).
- All 18 Sonnet 5.5 builds at `low` and `medium` did too. With `between_tools`, 5 of 6 did; one asked for two black 1×1 bricks where the scan has one.
- The Indonesian reads naturally on both: Indonesian color names, the hobby's English part names, everyday verbs (*letakkan*, *pasang*, *tempel*) and the right animal words (*cula*, *belalai*). On Sonnet 5.5, one step said "chassis" where "sasis" is usual, and one summary put a car's lights at the back where its step put them at the front.

**The default is Sonnet 5.5 at `low`.** The team's rule was to switch to Sonnet 5.5, at its fastest effort that passes, if that effort met all of: a request p50 of 10 s or less, every build valid, most builds with `missing` empty, and natural Indonesian. `low` met all four. `medium` was slower, at 10.7 s, and no better; `between_tools` was slower still.

**Latency is inside the budget, but only just.** The p50 of 9.6 s is under the contract's 10 s for ideas; with a scan of under a second, photo to ideas is about 10–11 s, under the 15 s target. But one of the four requests took 10.3 s, and four is a small sample. If the deployment runs over, the next steps are:

1. **Dropping the `cek_inventori` round.** The answer alone took 5.5–6.3 s at `low`, so this would save the 3–5 s tool round. But the contract's rules say the agent must call the tool, so that rule would change. The backend's own `cek_inventori` already guarantees `used` and `missing` (contract decision 12).
2. **`claude-haiku-5-5`:** the fastest and cheapest. It has no server-side fallback, so a refusal goes to the templates, and its builds and Indonesian are unmeasured.

Opus 5.5 is still one environment variable away, at about 17 s and $0.05 a request.

### The answer to open question 1

The LLM provider is Anthropic, with Claude as above. The contract itself doesn't change.

## Consequences

- Positive: ideas are written for each person's pieces, in English or natural Indonesian, and "Try other ideas" no longer runs out after six per theme.
- Positive: the contract and the web app don't change. A failure of any kind still answers with template ideas within the deadline, and the backend's own checks still hold.
- Positive: about $0.02 per request with Sonnet 5.5 at `low`, under half Opus 5.5's $0.05. The stable prompt is cached, so most input is read at a twentieth of the input price.
- Positive: the effort and the model are configuration, so going back to Opus 5.5 or trying Haiku 5.5 needs no code change.
- Negative: at `low`, a request takes about 10 s, at the edge of the contract's 10 s budget for ideas. One measured request in four went over, at 10.3 s. It's well within the 30 s deadline.
- Negative: on Sonnet 5.5, server-side fallbacks only re-run `cyber` and `frontier_llm` declines. Any other decline gives template ideas.
- Negative: a paid external service and a secret to manage. Ideas depend on the network and on Anthropic being up; when either fails, the person gets templates.
- Negative: answers aren't deterministic. Unit tests use a fake client, and only a by-hand live test calls the real API.
- Negative: server-side fallbacks are a beta feature, so the beta header may change.
- Negative: the inventory and earlier idea titles are sent to Anthropic. No photo or device id is sent.

## Pros and Cons of the Options

### Claude Opus 5.5

- Pro: the most capable of the three, and good at Indonesian. It has server-side fallbacks for refusals.
- Pro: per-token prices of $4 in, $20 out, and $0.20 for cache reads.
- Con: about 8–10 s per API call at `low`, so a request took 16.5 s (p50), over the 10 s budget. Thinking can't be turned off.
- Con: $0.049 a request at `low`, over twice Sonnet 5.5's.

### Claude Sonnet 5.5

- Pro: measured at `low`: 9.6 s per request (p50), $0.020, every build valid and from the person's own pieces, and natural Indonesian.
- Pro: half Opus 5.5's per-token price ($2 / $10), and half its cache-read price ($0.10). Server-side fallbacks for `cyber` and `frontier_llm` declines.
- Con: only just inside the 10 s budget. Its effort defaults to `high`, so effort must be set, and its lowest thinking setting, `between_tools`, was slower here than adaptive thinking.

### Claude Haiku 5.5

- Pro: the fastest, at $0.10 / $0.50 per million tokens.
- Con: no server-side fallback, so a refusal goes to the templates. Its quality on these builds and in Indonesian is unmeasured.

### Structured outputs with a strict `cek_inventori` tool

- Pro: the answer always parses, and part and color ids are limited to the catalogue by the schema. The tool call the PRD and the contract ask for stays the model's choice, which Sonnet 5.5 and Opus 5.5 require.
- Con: the tool round is a second request, which roughly doubles the latency.

### A forced tool call for the ideas

- Pro: a common way to get JSON from older models.
- Con: Sonnet 5.5 and Opus 5.5 answer a forced `tool_choice` with a 400.

### JSON in free text

- Pro: works on any model.
- Con: it can fail to parse or leave the catalogue, so it needs more repairs and more fallbacks.

### Templates when Claude can't answer

- Pro: the person always gets three valid ideas in time, and the screen never shows an error for a provider problem.
- Con: a template set can look like Claude's answer, so the log is the only place that says which wrote it.

### `503 unavailable`

- Pro: the contract allows it, and the client knows the provider failed.
- Con: legoin-web shows it as a network error, which is worse for the person than template ideas.

## References

- [API contract](../api/contract.md), open question 1 and the rules on languages and `cek_inventori`
- [ADR-0004](0004-english-default-with-indonesian.md): English by default, Indonesian as a second language
- [ADR-0008](0008-backend-stack.md): the backend stack and the `IdeaGenerator` interface
- legoin-api [PR #16](https://github.com/al-dioooo/legoin-api/pull/16): the implementation, the measurements and sample ideas
- Anthropic's [models](https://platform.claude.com/docs/en/about-claude/models/overview), [pricing](https://platform.claude.com/docs/en/about-claude/pricing) and [refusals and fallbacks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
