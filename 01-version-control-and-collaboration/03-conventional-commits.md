# Conventional Commits `[Entry]`

## Why Structured Messages

```
# Bad — what changed?
fix stuff
update things
wip
asdfasdf

# Good — why and what
fix(auth): redirect to login on expired session
feat(payments): add Stripe webhook handler
chore(deps): bump express to 4.19.2
```

Structured commit messages enable:
- **Automated changelogs** — generate release notes from commit history
- **Semantic versioning** — determine major/minor/patch bumps automatically
- **Git bisect** — find the commit that introduced a bug faster
- **Code archaeology** — understand why a change was made months later

## The Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

**Types:**

| Type | Meaning | Version Impact |
|------|---------|----------------|
| `feat` | New feature | Minor |
| `fix` | Bug fix | Patch |
| `docs` | Documentation only | None |
| `style` | Formatting, whitespace | None |
| `refactor` | Code restructure, no behavior change | None |
| `perf` | Performance improvement | Patch |
| `test` | Adding or fixing tests | None |
| `chore` | Build, CI, dependencies | None |
| `ci` | CI configuration | None |
| `revert` | Revert a previous commit | Varies |

## Examples

```bash
# Feature — minor version bump
git commit -m "feat(api): add pagination to /users endpoint"

# Fix — patch version bump
git commit -m "fix(auth): handle expired refresh tokens gracefully"

# Breaking change — major version bump
git commit -m "feat(api): redesign user response format

BREAKING CHANGE: /users response no longer includes 'profile' nested object.
Fields are now flattened to top level."

# Multiple paragraphs
git commit -m "fix(payments): retry failed webhook deliveries

Webhook deliveries to Stripe were silently failing on network timeouts.
Added exponential backoff retry with 3 attempts.

Closes #421"
```

## Automation

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate release
        uses: semantic-release/semantic-release@v24
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Semantic release reads commit history, determines version bump, generates changelog, creates GitHub release — all from commit messages.

## Commitlint

Enforce the convention in CI:

```yaml
# .commitlintrc.yml
extends:
  - "@commitlint/config-conventional"

rules:
  type-enum:
    - 2
    - always
    - - feat
      - fix
      - docs
      - style
      - refactor
      - perf
      - test
      - chore
      - ci
      - revert
  subject-max-length:
    - 2
    - always
    - 72
```

```yaml
# GitHub Action to validate commits
- name: Lint commits
  uses: wagoid/commitlint-github-action@v6
```

## Squash Merging

When merging a PR with 15 commits, squash into one well-formed commit:

```bash
# Squash merge via GitHub
gh pr merge 42 --squash --subject "feat(api): add rate limiting (#42)"

# In local git
git merge --squash feature/rate-limiting
git commit -m "feat(api): add rate limiting (#42)"
```

The commit history on `main` stays clean: one commit per feature, conventional format.
