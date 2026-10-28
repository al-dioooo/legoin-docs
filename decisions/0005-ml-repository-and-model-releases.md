# ADR-0005: Models in legoin-ml, Weights as GitHub Releases

- Status: Accepted
- Date: 2026-10-28
- Deciders: Legoin team
- Issue: #8

## Context and Problem

A Legoin scan needs trained models: a YOLO11 brick detector now, a part classifier and a color step later. They are trained in notebooks on Colab GPUs, from a 6 GB dataset, and each run produces a weights file. legoin-web is frontend only, and the backend hasn't started. Where do the notebooks and weights live, and how does the backend get a model?

## Decision Drivers

- Model work starts before the backend's stack is decided.
- Notebooks open in Colab straight from GitHub.
- Diffs stay reviewable: no datasets, weights or bulky notebook outputs in git.
- The backend pins an exact model version.
- The team guidelines apply: branches, pull requests and CI.

## Considered Options

1. A separate `legoin-ml` repository
2. A folder in the future backend repository
3. A `research/` folder in legoin-docs

## Decision

Chosen option: "a separate legoin-ml repository", because model work can start now and its heavy dependencies and data rules stay apart from service code.

- Notebooks live in `notebooks/` with outputs cleared. Metrics and plots go in the PR's test evidence and in release notes.
- CI runs on every pull request: Ruff lint, notebook validation, cleared outputs, and no datasets or weights in the repository. Training doesn't run in CI.
- Weights are attached to GitHub Releases tagged `vMAJOR.MINOR.PATCH`, with the training config and test metrics in the release notes. There is no `release` branch; `main` stays the only long-lived branch.
- The repository was bootstrapped with one direct commit to `main` (README and `.gitignore`), since a new repository has no branch to open a pull request against. Every later change goes through a pull request.
- The backend downloads weights from a pinned release tag.

## Consequences

- Positive: a notebook opens in Colab from its README badge, the repository stays small, and a model version is a tag.
- Positive: model work goes ahead without waiting for the backend.
- Negative: settings the backend must match, such as image size and the confidence threshold chosen on validation, are copied by hand from the release notes.
- Negative: one more repository to set up and review.

## Pros and Cons of the Options

### A separate legoin-ml repository

- Pro: starts now; ML dependencies and data rules stay out of the backend.
- Con: settings shared with the backend are copied, not imported.

### A folder in the backend repository

- Pro: training and serving can share code.
- Con: needs the backend repository and its stack decision first, and mixes notebooks with service CI.

### research/ in legoin-docs

- Pro: no new repository.
- Con: legoin-docs holds documents, not code or weights, and has no Python CI.

## References

- [legoin-ml](https://github.com/al-dioooo/legoin-ml)
- Detection notebook: al-dioooo/legoin-ml#2 and al-dioooo/legoin-ml#4
- Dataset: Zawora et al. (2021), *Tagged images with LEGO bricks*, <https://doi.org/10.34808/anq4-rn44> (CC BY 4.0)
