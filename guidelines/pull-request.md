# Pull Request Guideline

Before opening a pull request (PR), follow the [GitHub Repository Guideline](github-repository.md).

## 1. Workflow

1. **Create or identify an issue.** Discuss the problem or feature with maintainers before building, and make sure it fits the project direction.
2. **Create a branch** for the change.
3. **Implement the solution.** Follow the coding standards and add tests and documentation.
4. **Test the changes.** Run the unit tests, integration tests and linters, and check the feature works.
5. **Open a PR** with a clear description and context.
6. **Review.** Reviewers give feedback and the author updates the PR.
7. **Merge** after approval and passing CI checks.

## 2. Communicate early

Before starting development:

- Comment on the related issue
- Confirm the implementation approach with maintainers
- Avoid duplicate work or misaligned solutions

## 3. Keep PRs small

Small PRs are faster to review, easier to roll back, and safer to merge.

- Solve **one problem per PR**
- Don't mix unrelated changes
- Prefer several small PRs over one large PR
- Aim for under 500 changed lines where possible

When small PRs depend on each other, stack them: each PR targets the branch of the one before it.

## 4. What a PR includes

Required:

- Ticket id in the PR title, e.g. `feat(auth): add JWT refresh token (#14)`
- A description of the change
- A linked issue
- Tests and documentation when applicable
- Environment variable or production config changes needed before deployment
- Passing CI checks

Recommended:

- Screenshots for UI changes
- Performance comparison, if relevant
- Migration instructions for breaking changes

## 5. PR description

Every PR explains the problem, the solution, how it was tested, its impact or breaking changes, and what reviewers should focus on:

```markdown
## Summary
Brief description of the change.

## Problem
What issue or limitation this PR addresses.

## Solution
How the change solves the problem.

## Test Evidence
A screenshot or recording covering the normal flow and some edge cases.

## Deployment Prerequisite / Breaking Changes
Env changes, DB migrations, deleted functions, etc.

## Notes
Additional context for reviewers.
```

A clear description lets reviewers understand the change without digging through the whole codebase.

## 6. Merge requirements

Before merging:

- All CI checks pass
- All discussions are resolved
- Required reviewers approve
- Commit messages follow the [guideline](github-repository.md#2-commit-messages)

If CI fails, update the PR before merging.
