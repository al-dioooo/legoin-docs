# GitHub Repository Guideline

## 1. Commit size and frequency

Every commit is:

- **Small and focused**, roughly 5 to 100 lines
- **One logical change**
- **Easy to review and revert**

Commit whenever:

- a function is done
- a bug is fixed
- a logical step is done

Rule of thumb: if you pause work, you should be able to commit first.

A good commit has one concern and can be explained in one sentence. It never mixes a refactor with a feature, or a fix with formatting.

Good:

```text
feat: add login API integration
fix: handle empty email validation
refactor: extract auth service
```

Bad:

```text
update code
fix bugs
final fix for real
```

Avoid:

- Large commits (1000+ lines, many files, many concerns)
- WIP commits on shared branches
- "Panic coding" commits pushed without cleanup
- Many changes without intermediate commits

## 2. Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): description`. The scope is optional.

| Type | Use for |
| --- | --- |
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code restructuring |
| `docs` | Documentation updates |
| `chore` | Maintenance tasks |
| `test` | Tests |

Examples:

```text
feat: add payment gateway integration
fix: handle null response on login API
refactor: simplify auth middleware
feat(auth): add JWT refresh token
```

Mark breaking changes with `!`:

```text
feat(api)!: change response format
```

### Before committing

- All tests pass
- New features come with tests
- No lint errors
- The change works locally

Passing tests let reviewers focus on design and correctness instead of basic failures.

## 3. Branching

Use [trunk-based development](https://trunkbaseddevelopment.com/). `main` is production-ready and can be released at any time. There is no `develop`, `staging`, `release` or similar branch.

Name branches after the change:

```text
feature/<short-description>
fix/<bug-description>
chore/<maintenance>
```

For example `feature/payment-integration` or `fix/login-validation-error`.

Rules:

- No direct commits to `main`, except an emergency hotfix during an incident.
- Every change goes through a branch and a [pull request](pull-request.md).

## 4. Documentation

Every repository's README covers:

- Project overview
- Tech stack
- Setup and run
- Environment variables
- Deployment (optional)

Decisions are recorded as ADRs in [decisions/](../decisions).

## 5. Security

Never commit:

- `.env` files
- API keys, tokens or other credentials
- Logs

Commit only `.env.example`, with empty or sample values.
