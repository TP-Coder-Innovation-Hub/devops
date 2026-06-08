# Code Review

## Why Code Review

Code review catches bugs, spreads knowledge, and raises the baseline quality of the entire team. It is not gatekeeping — it is shared ownership.

**Goals:**
- Catch defects before production
- Share domain knowledge across the team
- Enforce consistent patterns
- Mentor junior engineers

**Not goals:**
- Find every bug (tests do that)
- Enforce personal style preferences
- Block releases over bikeshedding

## The PR Process

```yaml
# A healthy PR workflow
1. Author writes code + tests
2. Author self-reviews the diff before requesting review
3. Reviewer reads the PR description, then the code
4. Reviewer leaves comments (blocking or non-blocking)
5. Author addresses feedback
6. Reviewer approves
7. CI must pass
8. Author merges (not the reviewer)
```

## PR Size Matters

```bash
# Target PR sizes
Small:   50-200 lines changed   → 1-2 reviewers, < 1 hour review
Medium:  200-500 lines changed   → 2 reviewers, < 2 hours
Large:   500+ lines changed      → Break into smaller PRs

# Check PR size
git diff --stat main...feature/login
```

Large PRs get skimmed, not reviewed. If your PR is over 500 lines, split it.

## Reviewing for Maintainability

Do not just check "does this work?" Ask:

| Question | Why It Matters |
|----------|----------------|
| Can I understand this without asking the author? | Code is read more than written |
| Will this be easy to change in 6 months? | Requirements change |
| Are error cases handled? | Users find edge cases |
| Is there a simpler way? | Complexity is the enemy |
| Are names clear? | Naming is documentation |

## Comment Categories

Use prefixes so authors understand intent:

```
[blocking] — must fix before merge (bugs, security, broken tests)
[suggestion] — optional improvement (readability, performance)
[question] — seeking understanding (helps both reviewer and author)
[nit] — trivial (typo, formatting) — never block on these
```

## Writing Good PR Descriptions

```markdown
## What
Add rate limiting to the /api/search endpoint.

## Why
The endpoint is expensive (full-text search). During traffic spikes,
it overwhelms the database and causes cascading failures.

## How
- Token bucket algorithm (100 req/min per IP)
- Returns 429 with Retry-After header when exceeded
- Metrics emitted to Prometheus

## Testing
- Unit tests for rate limiter logic
- Integration test for 429 response
- Load tested at 500 req/min — confirmed throttling works
```

## Anti-Patterns

- **Rubber-stamping:** approving without reading the code. Defeats the purpose.
- **Nitpicking to death:** blocking on formatting when a linter can fix it. Automate style.
- **Ship-and-review:** merging before review under "urgency." Every time, it introduces bugs.
- **Comment-only reviews:** only catching typos, never engaging with logic.

## Reviewer Checklist

```yaml
review_checklist:
  correctness:
    - does the code do what the PR says?
    - are edge cases handled?
    - are error messages useful?
  security:
    - any hardcoded secrets?
    - user input sanitized?
    - authz checks present?
  testing:
    - are there tests for new behavior?
    - do tests cover failure paths?
  design:
    - does this fit existing patterns?
    - is the abstraction level right?
    - will this scale?
```
