# ADR-0007: End-to-End Smoke Tests with Playwright, on Chromium, against a Production Build

- Status: Proposed
- Date: 2026-11-08
- Deciders: Legoin team
- Issue: #12
- Amends: [ADR-0001](0001-web-tech-stack.md), which left Playwright out

## Context and Problem

[ADR-0001](0001-web-tech-stack.md) chose Vitest with Testing Library and left Playwright out until there were "flows worth testing". The whole flow now exists in legoin-web, in English and Indonesian:

1. take or choose a photo and scan it;
2. check the results and correct pieces;
3. pick a theme and get three build ideas;
4. open an idea for its parts and steps;
5. come back to the scan in history.

Vitest covers the logic and each component's states, but nothing checks that the pages work together. That means navigation, scans kept in `localStorage` from one page to the next, the async Server Components and layouts Vitest can't render, the proxy and the not-found page. A broken link between two steps would pass every check we have. How should end-to-end tests be added, and how far should they go?

## Decision Drivers

- Catch what only a real browser and a real build show: pages linked together, data kept across navigations, Server Components, layouts and status codes.
- Keep PRs fast. The required check takes under a minute today, and every PR waits on CI before review.
- Few tools and dependencies to maintain ([ADR-0001](0001-web-tech-stack.md)).
- Tests that read like what a person does, and that break when the UI stops making sense to assistive technology, not when a class name changes.
- No flaky tests: a red check must mean something.
- The audience: students and families in Indonesia, mostly on Android phones, so 360px wide matters as much as a laptop screen.

## Considered Options

Browsers:

1. Chromium only, at a phone width and a desktop width
2. Chromium, plus WebKit with a mobile profile
3. Chromium, Firefox and WebKit

Server under test:

1. A production build: `pnpm build`, then `pnpm start`
2. The dev server: `pnpm dev`

Gating:

1. A second required check, in its own job
2. More steps in the existing check, "Lint, typecheck, test and build"
3. A check that runs but isn't required

## Decision

Add [Playwright](https://playwright.dev) (`@playwright/test`) to legoin-web as a **smoke suite that runs on Chromium against a production build, as a second required check**. Vitest stays the main safety net.

**Browsers.** Chromium only, in two projects:

- **phone:** 360 × 780, touch, mobile mode;
- **desktop:** 1280 × 800.

Every spec runs in both. WebKit stays out of CI for now:

- most of the audience is on Android, where Chromium-based browsers dominate;
- Playwright's WebKit on Linux isn't Safari on an iPhone, so it would cost a second engine's install and a doubled test run without testing the browser people actually use;
- a WebKit project is a few lines of config. Add it when a Safari-only bug turns up.

**Production build.** The suite runs against `pnpm build` and `pnpm start`, never `pnpm dev`:

- the dev server compiles each page on its first request, which makes timing uneven and timeouts likely;
- dev mode runs effects twice and shows the dev overlay;
- what Cache Components prerenders, the static shells and streamed parts, only exists in a build. That's what people get, and what the Next.js testing guide recommends.

The Playwright config starts the server itself on port 3100, out of the way of a dev server on 3000. Locally it builds first; in CI the build is its own step.

**Scope: a smoke suite.**

- **The whole flow,** from home to deleting the scan in history, in English and Indonesian, at both widths.
- **Each `?mock=fail-<reason>` failure,** with its message and the way to retry.
- **The empty state** after removing every piece.
- **The language switch** keeping the query on the ideas page.
- **The not-found page.**

A test belongs here only when it needs what a real browser and build give: more than one page, storage across navigations, Server Components, layouts, the proxy, status codes or metadata. A state of one component, or one more variation of a flow, goes in Vitest. When the suite grows past about a dozen tests per project, move what Vitest can cover.

**Test data.** Specs drive the mock backend through the demo options in the URL:

- `?mockDelay=0` skips the fake wait when a spec opens a scan or ideas page by URL;
- `?mock=fail-<reason>` makes a scan fail.

The app's own links don't carry these options, so the whole flow, clicked through from home, runs at the mock's normal pace: a few seconds per run.

Every test gets a fresh browser context, so it starts from empty storage, with only the mock's four demo scans. Tests never share state or depend on their order.

**Selectors: roles and labels.**

- Specs find elements by role and accessible name (`getByRole`) or by label (`getByLabel`). Narrowing a role with `hasText` is fine.
- Specs take their text from the dictionaries in `src/i18n/dictionaries`, so copy isn't repeated in the tests and every spec runs in both languages.
- No CSS or XPath selectors, and no `data-testid`.
- If a spec can't reach an element by role and name, the element usually has an accessibility gap. Fix the markup in its own commit, rather than adding a test id. A test id is the last resort, for something with no meaningful role.
- No fixed waits (`waitForTimeout`): assertions wait for the page to reach the expected state.

**Runtime budget.**

- The test run takes at most 1 minute in CI.
- The whole job takes at most 5 minutes, including install, browsers and build. It's stopped at 10.
- One test takes at most 30 seconds.

Going over the budget is a reason to trim the suite, not to raise the limits.

**Retries and flaky tests.**

- **In CI,** a failed test is retried once, which records a trace. A test that only passes on its retry still fails the check (`failOnFlakyTests`): against a local mock there's no network to blame, so a flaky test is a bug in the test or a race in the app. The HTML report and traces are uploaded when the job fails.
- **Locally,** there are no retries.
- **Quarantine.** A test that can't be fixed straight away is marked `test.fixme()`, with a link to an open issue, in its own commit. The PR that closes the issue brings the test back.

**Gating.**

- The suite runs in its own job, "End-to-end smoke", in the same workflow and in parallel with "Lint, typecheck, test and build". A failure then names what failed, the job can be re-run on its own, and the two together take about as long as the slower one.
- After the PR that adds the job merges, a maintainer adds it to the required status checks on `main`.
- The job caches Playwright's browser download, keyed by the Playwright version, and installs only Chromium's headless shell.

**What stays out:**

- Firefox, WebKit and real devices;
- visual regression (screenshot comparison);
- automated accessibility checks with axe. They can join this suite later under the same rules, in their own PR, without a new ADR;
- performance and Lighthouse budgets;
- runs against `pnpm dev`, or against Vercel previews, which sit behind a log-in;
- a real backend. There's no HTTP client yet. When there is, this suite keeps using the mock, and the API contract gets its own tests;
- Next.js's `instant()` navigation assertions (`@next/playwright`), until instant navigation becomes a requirement.

## Consequences

- Positive: a broken step between pages, a lost scan, or a page that only fails once built now fails a required check.
- Positive: async Server Components, layouts and the not-found page are tested in a real browser, closing the gap ADR-0001 left.
- Positive: role-and-name selectors make the suite an extra check on accessible names. If a button loses its name, a test fails.
- Positive: specs read the dictionaries, so a change of copy doesn't break them.
- Negative: one more dev dependency, and a browser download in CI and on each developer's machine (`pnpm exec playwright install chromium`).
- Negative: CI builds the app twice, once in each job. Each build takes about 10 seconds today.
- Negative: a maintainer has to add the new check to `main`'s required checks by hand.
- Negative: Safari and Firefox bugs still reach people. A WebKit project is the first thing to add if one does.
- Negative: since a flaky test fails the check, someone has to look into an occasional red check instead of re-running it.

## Pros and Cons of the Options

### Chromium only, at two widths

- Pro: one browser download and the shortest run.
- Pro: matches the browsers most of the audience uses.
- Con: no Safari or Firefox engine.

### Chromium and mobile WebKit

- Pro: catches some WebKit-only bugs, such as differences in View Transitions and CSS ([ADR-0006](0006-motion-and-interaction-language.md)).
- Con: doubles the test run and adds a heavier install with more system libraries.
- Con: Linux WebKit isn't iOS Safari, so it's a proxy for the browser that matters.

### Chromium, Firefox and WebKit

- Pro: the broadest engine coverage.
- Con: triples the run for a smoke suite whose job is to find broken flows, not engine quirks.

### Production build

- Pro: tests what people get, with the prerendering and streaming that Cache Components does.
- Pro: fast, even page timings, so fewer timeouts.
- Con: a code change needs a rebuild before the run (about 10 seconds).

### Dev server

- Pro: no build step, and it reloads as you edit.
- Con: compiles pages on their first request, runs effects twice and shows dev-only UI, so it's slower and behaves differently from production.

### A second required check

- Pro: runs in parallel, names what failed, and can be re-run alone.
- Con: a second build, and a one-time change to the branch's required checks.

### Steps in the existing check

- Pro: one build, and no change to branch protection.
- Con: the steps run one after another, so every PR waits longer, a failure is harder to read, and re-running a flaky test re-runs lint and unit tests too.

### Not required

- Pro: never blocks a merge.
- Con: a red check that doesn't block soon gets ignored.

## References

- [ADR-0001](0001-web-tech-stack.md), which this amends
- Next.js guide "How to set up Playwright with Next.js", which recommends testing the production build
- Playwright docs: [locators](https://playwright.dev/docs/locators), [retries](https://playwright.dev/docs/test-retries) and [CI](https://playwright.dev/docs/ci)
- Gap audit item C8 (end-to-end smoke tests)
