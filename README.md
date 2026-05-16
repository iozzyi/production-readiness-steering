# Production Readiness Steering File

Opinionated production-readiness standards for AI-assisted development, designed for small teams (1-3 people) building serverless/container workloads on AWS.

Drop this file into your project's `.kiro/steering/` folder and every AI agent interaction will follow these standards automatically — no manual prompting required.

## What It Does

When included in a Kiro-powered workspace, this steering file ensures AI agents:

- Structure code in a 2-layer architecture (`app/` + `infra/`)
- Never hardcode secrets — always use Secrets Manager
- Validate all external inputs with schemas (Pydantic / Zod)
- Enforce tool allowlists when AI models invoke actions
- Use structured JSON logging with correlation IDs
- Implement circuit breakers on all external dependencies
- Write tests targeting 60% coverage on critical paths
- Deploy infrastructure via CDK or SST — never manual clicks
- Set up 5 essential CloudWatch alarms
- Avoid over-engineering (no Kubernetes for small workloads, no microservices for single domains)

## Installation

Copy the steering file into your project:

```bash
mkdir -p .kiro/steering
cp production-readiness.md .kiro/steering/
```

Or add as a git submodule:

```bash
git submodule add <repo-url> .kiro/steering/production-readiness
```

## Configuration

The file uses `inclusion: always` front matter, meaning it applies to every AI interaction in the workspace. To make it opt-in instead, change the front matter:

```yaml
---
inclusion: manual
description: Production-readiness standards for small teams on AWS.
---
```

With `manual` inclusion, users reference it explicitly via `#production-readiness` in chat.

## What's Covered

| Section | Summary |
|---------|---------|
| Architecture | 2 layers (`app/` + `infra/`), inward dependency flow, validated config at startup |
| 5 Non-Negotiables | Secrets Manager, input validation, tool allowlists, structured logging, circuit breakers |
| Security | KMS encryption, least-privilege IAM, WAF, auth on every request, no PII in logs |
| Error Handling | Typed error codes, structured responses, fail-fast on config errors |
| Testing | 60% coverage threshold, focus on business logic and validation |
| Observability | 5 CloudWatch alarms, structured logs, X-Ray only when needed |
| Resilience | Circuit breakers, retries with backoff, timeouts, graceful degradation |
| IaC | CDK or SST, one stack per environment, auto-scaling, private subnets |
| CI/CD | Lint → Test → Build → Deploy, coverage enforced, manual prod approval |
| Progressive Complexity | "What to add later" table — only take on overhead when the pain justifies it |
| Anti-Patterns | Explicit list of things to avoid (Kubernetes for small workloads, over-abstraction, etc.) |

## Philosophy

These standards exist at the intersection of two failure modes:

1. **Too little rigour** — no validation, no logging, no resilience. Works until it doesn't, then you're debugging blind at 2am.
2. **Too much ceremony** — 3-layer architecture, 80% coverage, SonarQube, OpenAPI specs, event sourcing. Appropriate for 50-person teams, paralysing for solo developers.

This file targets the middle ground: enough structure to sleep at night, not so much that you spend more time on infrastructure than product.

## Compatibility

- **Kiro** — drop into `.kiro/steering/`
- **Other AI coding tools** — the file is plain Markdown. Copy the content into your tool's system prompt, rules file, or context window.
- **Human developers** — readable as a standalone engineering standards document

## Customisation

Fork and modify freely. Common adjustments:

- Change coverage threshold (60% → 80% for regulated industries)
- Add language-specific tooling (Go, Rust, Java equivalents)
- Replace AWS services with GCP/Azure equivalents
- Add domain-specific non-negotiables (e.g., HIPAA, SOC 2)
- Remove the "What to Add Later" section if your team is already large

## License

MIT — use it however you want.
