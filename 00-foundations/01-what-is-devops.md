# What Is DevOps

## The Problem DevOps Solves

Development teams ship code fast. Operations teams keep systems stable. These goals conflict.

```mermaid
graph LR
    subgraph "Before DevOps"
        DEV["Dev\n'throws code over wall'"] -->|"blame"| OPS["Ops\n'works on my machine'"]
        OPS -->|"blame"| DEV
    end
    subgraph "After DevOps"
        TEAM["Shared Ownership\nDev + Ops together"] --> AUTO["Automate everything"]
        AUTO --> FEED["Fast feedback loops"]
        FEED --> TEAM
    end
```

```mermaid
graph LR
    Dev[Dev: Ship fast] -->|Throws code over wall| Ops[Ops: Keep it stable]
    Ops -->|Change approval boards| Dev
    Dev -->|"Works on my machine"| Fire[Production fire]
```

The result: slow releases, blame cycles, broken production.

## DevOps Definition

DevOps is not a tool or a team. It is **culture + practices + tools** that break down the wall between development and operations.

- **Culture** — shared ownership of production, blameless collaboration
- **Practices** — CI/CD, infrastructure as code, automated testing
- **Tools** — Git, Docker, Kubernetes, Prometheus, Terraform

## CALMS Framework

The maturity model for DevOps adoption:

| Dimension | Meaning | Example |
|-----------|---------|---------|
| **C**ulture | Trust, shared responsibility | Devs carry pagers |
| **A**utomation | Eliminate manual toil | CI pipeline runs tests on every push |
| **L**ean | Small batches, fast feedback | Deploy hourly, not monthly |
| **M**easurement | Data-driven decisions | Track deployment frequency, MTTR |
| **S**haring | Open knowledge flow | Postmortems published to all |

## Key Metrics (DORA Four)

High-performing teams measure:

1. **Deployment Frequency** — how often code deploys to production
2. **Lead Time for Changes** — commit to production
3. **Change Failure Rate** — percentage of deployments causing failures
4. **Mean Time to Recover (MTTR)** — how fast you fix what breaks

These metrics reveal whether your process works, not whether your tools are trendy.

## The Feedback Loop

```mermaid
flowchart LR
    PLAN[Plan] --> CODE[Code]
    CODE --> BUILD[Build]
    BUILD --> TEST[Test]
    TEST --> RELEASE[Release]
    RELEASE --> DEPLOY[Deploy]
    DEPLOY --> OPERATE[Operate]
    OPERATE --> MONITOR[Monitor]
    MONITOR -->|"feedback"| PLAN
```

```mermaid
graph TD
    Plan --> Code --> Build --> Test --> Release --> Deploy --> Operate --> Monitor --> Plan
```

Every stage feeds the next. Speed through the loop = faster learning = better software.

## What DevOps Is Not

- A job title you assign to one person while everyone else works the old way
- Buying Jenkins and calling it done
- A separate "DevOps team" that becomes a new silo

DevOps succeeds when the entire organization optimizes for fast, safe delivery.
