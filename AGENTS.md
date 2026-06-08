# AGENTS.md

## Build & Dev Commands

No build system. All files are Markdown documentation.

- Validate links: `find . -name "*.md" -exec markdown-link-check {} \;`
- Lint markdown: `npx markdownlint-cli2 "**/*.md"`

## Repository Structure

```
00-foundations/          — Core concepts (Linux, shell, networking)
01-version-control.../   — Git, code review, commits
02-ci-cd/               — CI/CD pipelines and artifact management
03-containers/          — Docker and container security
04-kubernetes/          — K8s concepts, practice, Helm
05-observability/       — Logging, metrics, alerting, incidents
06-capstone/            — Hands-on project
```

## Content Style

- 200-500 words per file
- YAML/Shell code examples
- Mermaid diagrams where helpful
- `[Entry]`/`[Mid]`/`[Senior]` difficulty badges
- No emojis, no filler

## Editing Guidelines

- Keep files short and self-contained
- Every code example must be runnable or clearly illustrative
- Cross-reference other modules with relative links
- Use conventional commit messages
