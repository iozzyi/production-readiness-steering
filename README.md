# AWS CDK + TypeScript Steering Files

A set of AI steering files that encode production-grade standards for TypeScript-everywhere AWS projects deployed with CDK and GitHub Actions. These files guide AI coding assistants toward battle-tested patterns and away from common pitfalls.

## What's Included

| File | Purpose |
|------|---------|
| `architecture-decisions.md` | Design principles for compute, routing, multi-tenancy, data, IAM, and cost optimization |
| `github-cicd.md` | CI/CD pipeline structure, deploy scripts, container builds, OIDC, and failure recovery |
| `production-readiness.md` | Application-level standards: hexagonal architecture, security, testing, resilience, observability |
| `install-dependencies.md` | Dependency management rules to prevent stale lockfiles and broken builds |

## How They Work Together

```
architecture-decisions.md    → WHY we make certain choices (principles)
production-readiness.md      → WHAT the application code must satisfy (standards)
github-cicd.md               → HOW we build, test, and deploy (pipeline mechanics)
install-dependencies.md      → WHEN to install deps (operational discipline)
```

- `architecture-decisions.md` sets the strategic direction — TypeScript everywhere, CDK for IaC, ARM64 Fargate, ALB with host-based routing, DynamoDB for state, SSM for discovery.
- `production-readiness.md` defines what "done" looks like at the application layer — hexagonal architecture, 80% test coverage, circuit breakers, structured logging, per-tenant isolation.
- `github-cicd.md` covers the delivery pipeline — image builds, CDK deploys, OIDC roles, self-healing scripts, multi-tenant lifecycle, and a comprehensive failure-pattern reference table.
- `install-dependencies.md` prevents a class of "works on my machine" bugs by enforcing immediate installs after any `package.json` edit.

## Usage

### With Kiro

Place these files in `.kiro/steering/` at your workspace root. Kiro loads them automatically on every interaction (they use `inclusion: always` front-matter).

### With Other AI Assistants

Feed these files as system context or reference documents. They're written as declarative rules that any LLM-based coding assistant can follow.

## Key Patterns Encoded

- **Build once, deploy everywhere** — Docker images built in a dedicated workflow, pushed to ECR, referenced by tag during CDK deploy
- **Hexagonal architecture** — domain logic has zero external dependencies; adapters implement port interfaces
- **Multi-tenant by default** — per-tenant Cognito pools, DynamoDB isolation, config-driven behavior (not code branches)
- **SSM for resource discovery** — CDK writes IDs at creation time, scripts read from SSM (never hardcoded, never passed as arguments)
- **Feature-flagged cost** — observability, WAF, canaries toggled via environment variables
- **Self-healing deploys** — scripts detect and recover stuck CloudFormation stacks
- **Narrow IAM roles** — separate roles per lifecycle step, OIDC federation for CI/CD

## Target Stack

These steering files assume:

- **Language:** TypeScript (backend, frontend, IaC)
- **Infrastructure:** AWS CDK
- **Compute:** ECS Fargate (ARM64)
- **CI/CD:** GitHub Actions with OIDC
- **Testing:** Vitest (unit), Cypress (E2E)
- **Validation:** Zod (runtime), TypeScript (compile-time)
- **Logging:** Pino (structured JSON)
- **State:** DynamoDB (ephemeral), SSM Parameter Store (config)

## Adapting for Your Project

These files are opinionated but modular. Common adaptations:

- **Single-tenant?** Remove the multi-tenancy sections from `production-readiness.md` and `github-cicd.md`.
- **Different CI provider?** Replace GitHub Actions specifics in `github-cicd.md` with your provider's equivalents (the CDK and deploy script patterns still apply).
- **Different compute?** Swap ECS/Fargate references for Lambda, App Runner, or EKS — the architecture principles and production standards remain valid.
- **No frontend?** The frontend versioning and Lambda@Edge sections can be removed without affecting backend/infra guidance.

## License

These steering files are project documentation. Use and adapt them freely.
