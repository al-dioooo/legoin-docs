# ADR-0001: Next.js 16 with Biome and Vitest for the Web Frontend

- Status: Accepted
- Date: 2026-10-28
- Deciders: Legoin team

## Context and Problem

legoin-web needs a starting stack: framework, styling, linting, testing and package manager. The [repository guideline](../guidelines/github-repository.md#before-committing) requires lint, tests and a local check to pass before every commit, so the stack has to provide all of them from the first commit.

## Decision Drivers

- Lint, typecheck and tests runnable before every commit
- pnpm as the package manager
- Few tools to configure and keep in sync

## Considered Options

Linting and formatting:

1. ESLint (the `create-next-app` default)
2. Biome

Testing:

1. Vitest with Testing Library
2. Vitest with Testing Library, plus Playwright for end-to-end tests
3. No test runner yet

## Decision

- **Framework:** Next.js 16 App Router with TypeScript, code under `src/`, Cache Components and the React Compiler enabled.
- **Styling:** Tailwind CSS 4.
- **Linting and formatting:** Biome, because one fast tool covers both and ships `next` and `react` rule domains.
- **Testing:** Vitest with Testing Library and jsdom, following the Next.js Vitest guide. Vite's built-in `resolve.tsconfigPaths` replaces the `vite-tsconfig-paths` plugin. Playwright is left out for now.
- **Package manager:** pnpm.

`pnpm typecheck` runs `next typegen` before `tsc --noEmit`, because global route types such as `LayoutProps` are generated, not checked in.

## Consequences

- Positive: `pnpm lint`, `pnpm typecheck` and `pnpm test` all work from the first commit.
- Positive: one tool for linting and formatting instead of ESLint plus Prettier.
- Negative: Biome has fewer Next.js-specific rules and plugins than `eslint-config-next`.
- Negative: Vitest cannot render `async` Server Components, and there is no end-to-end runner yet, so those components need Playwright or similar once they hold real logic.
- Negative: Cache Components makes data dynamic unless cached explicitly with `use cache`, which differs from older Next.js examples.

## Pros and Cons of the Options

### ESLint

- Pro: Next.js default with the official `eslint-config-next` rules and the largest plugin ecosystem.
- Con: needs a separate formatter, and is slower.

### Biome

- Pro: lints and formats in one fast tool.
- Con: smaller plugin ecosystem and fewer Next.js-specific rules.

### Vitest with Testing Library

- Pro: documented Next.js setup, fast, no browser install.
- Con: no `async` Server Component rendering, no real browser.

### Adding Playwright

- Pro: covers `async` Server Components and full user flows in a real browser.
- Con: browser downloads and slower runs before there are flows worth testing.

## References

- [Next.js Vitest guide](https://nextjs.org/docs/app/guides/testing/vitest)
- [Biome](https://biomejs.dev)
