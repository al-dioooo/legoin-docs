# ADR-0004: English by Default, Indonesian as a Second Language

- Status: Accepted
- Date: 2026-10-28
- Deciders: Legoin team
- Issue: #5

## Context and Problem

The first version of legoin-web had its UI copy written straight into the components, in Bahasa Indonesia. The team wants the app to open in English. The PRD, though, names Indonesian-language recommendations as one of Legoin's differentiators. Which languages should the UI ship in, and how should a visitor get each one?

## Decision Drivers

- English is the default experience.
- Keep Indonesian, which the PRD lists as a differentiator.
- Shareable links that open in a specific language.
- Static pages, which Cache Components prerenders.
- No dependency on an i18n library while the copy is small.

## Considered Options

Languages:

1. English only
2. English by default, plus Indonesian

How visitors switch:

1. A URL prefix per language (`/en`, `/id`), the pattern in the Next.js internationalization guide
2. A switcher that stores the choice in a cookie, with the same URLs for every language

## Decision

**English by default, plus Indonesian, on URL prefixes.**

- Every page lives under `app/[lang]`. `generateStaticParams` prerenders both `/en` and `/id`.
- `proxy.ts` redirects any path without a language prefix to `/en`. It doesn't read `Accept-Language`, so English is what everyone gets first.
- UI copy lives in typed dictionaries in `src/i18n/dictionaries`. The English file defines the shape, and the Indonesian file is typed against it, so a missing key fails the typecheck. A test also compares their keys and list lengths.
- Brick color names and number formats follow the language: 0.94 in English, 0,94 in Indonesian. Part names (Brick 2×4, Plate 1×2) stay in the hobby's English in both.
- An EN/ID switcher in the header keeps the current page. `<html lang>` and `hreflang` alternate links are set per language.
- No i18n library. This is the dictionary pattern from the Next.js guide, plus `next/root-params` to read the language in Server Components.

## Consequences

- Positive: English-speaking reviewers and demo audiences land in English, and Indonesian is still one click or one link away.
- Positive: every page in both languages is static HTML; the proxy only adds a redirect for unprefixed paths.
- Positive: a missing translation can't ship, because the typecheck and tests catch it.
- Negative: every new string has to be written twice.
- Negative: the AI agent's recommendations must be requested in the visitor's language, so the backend has to accept a language parameter.
- Negative: unknown prefixes such as `/fr` are treated as English paths and end in a 404, not a language fallback.

## Pros and Cons of the Options

### English only

- Pro: no i18n layer, half the copy work.
- Con: drops the PRD's Indonesian differentiator.

### English and Indonesian

- Pro: covers the demo audience and the target users.
- Con: two sets of copy to keep in step.

### URL prefix per language

- Pro: shareable links per language, static pages, the Next.js-documented pattern.
- Con: URLs carry a prefix, and unprefixed paths need a redirect.

### Cookie switcher

- Pro: clean URLs.
- Con: pages become dynamic, and a shared link opens in whatever language the recipient last picked.

## References

- Next.js internationalization guide (`node_modules/next/dist/docs/01-app/02-guides/internationalization.md` in legoin-web)
- [ADR-0003](0003-ballers-style-lego-colors.md)
- PRD BrickSense v0.1, section 2 (Indonesian-language recommendations as a differentiator)
