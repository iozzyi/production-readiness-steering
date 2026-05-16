---
inclusion: always
description: Production-readiness standards for small teams on AWS. Enforces 5 non-negotiables (secrets management, input validation, tool allowlists, structured logging, circuit breakers), a 2-layer architecture, 60% test coverage, 5 CloudWatch alarms, CDK/SST for IaC, and simple CI/CD. Includes guidance on what to defer until complexity demands it.
---

# Production Readiness Standards — Balanced for Small Teams

These standards balance production-readiness with low operational overhead for small teams (1-3 people) building serverless/container workloads on AWS.

## PCI-DSS

These standards do NOT override PCI-DSS requirements. If handling card data, apply PCI compliance standards separately.

## Architecture (2 Layers, Not 3)

- Use **two layers**: `app/` (domain logic + handlers) and `infra/` (external integrations)
- `app/` contains: business logic, models, orchestration, error definitions
- `infra/` contains: AWS SDK clients, WebSocket/HTTP servers, auth, metrics
- Domain logic in `app/` must NOT import from `infra/` — dependency flows inward only
- A `config.py` (or `config.ts`) at the root validates all environment variables at startup
- A `main.py` (or `main.ts`) at the root wires everything together (composition root)
- Do NOT create ports/adapters/interfaces unless the project has 3+ implementations of the same contract

## 5 Non-Negotiables (Never Skip)

1. **Secrets in Secrets Manager** — never in environment variables, never in source code, never in git. Use AWS Secrets Manager or SSM Parameter Store. CDK/SST wires secrets to the runtime.
2. **Input validation** — every external input (HTTP, WebSocket, event) validated with a schema before processing. Use Pydantic (Python) or Zod (TypeScript). Reject malformed input immediately.
3. **Tool/action allowlist** — if an AI model can invoke tools, maintain an explicit allowlist. Rate-limit tool calls per session. Classify tools by risk level (LOW/MEDIUM/HIGH). HIGH-risk tools require human approval.
4. **Structured logging** — JSON format, correlation IDs on every log line, PII redacted. Use structlog (Python) or pino (TypeScript). Never `print()` or `console.log()` in production code.
5. **Circuit breaker on external dependencies** — any call to an external service (Bedrock, PSP, third-party API) must have a circuit breaker. A voice call with 30 seconds of silence is worse than a graceful fallback message.

## Security

- All data encrypted at rest (KMS) and in transit (TLS 1.2+)
- IAM roles with least-privilege — scope to specific resources, not `*`
- WAF on any public-facing endpoint (rate limiting + geo-restriction at minimum)
- No hardcoded credentials, API keys, or tokens anywhere in the codebase
- Validate authentication on every request (JWT, API key, or webhook signature)
- Never log secrets, tokens, card numbers, or full PII — log references only

## Error Handling

- Define domain errors with typed error codes (enum/string union)
- Never expose stack traces or internal details to external consumers
- Log errors with context (session ID, operation, input summary) — not just the exception message
- Return structured error responses: `{ code, message }` — never raw exception strings
- Fail fast on configuration errors at startup — don't discover missing config mid-call

## Testing (60% Coverage on Critical Paths)

- Test the orchestration/business logic layer thoroughly (this is where bugs live)
- Test input validation (ensure malformed input is rejected)
- Test tool allowlist enforcement (ensure blocked tools stay blocked)
- Test circuit breaker state transitions
- Integration tests for external adapters are optional for MVP — add when the adapter has failed in production
- Use `pytest` (Python) or `vitest` (TypeScript) — match the project language
- Coverage enforced in CI — build fails below 60%

## Observability (5 Alarms, Not 50)

- **5 CloudWatch alarms minimum:**
  1. Error rate > threshold (5xx responses or unhandled exceptions)
  2. Latency p99 > acceptable threshold
  3. Circuit breaker in OPEN state
  4. Active sessions/connections approaching capacity
  5. Failed authentication attempts spike
- Structured logs flow to CloudWatch Log Groups with 14-day retention (dev) or 90-day (prod)
- Add X-Ray tracing only when you have a latency problem to diagnose — not by default
- Add custom metrics only for business-critical measurements (e.g., payment success rate)

## Resilience

- Circuit breaker on every external dependency (3 failures in 60s → OPEN → 30s cooldown)
- Retry transient errors once with 500ms delay, then circuit breaker
- Timeout on all external calls (Bedrock: 10s, PSP: 30s, DynamoDB: 5s)
- Graceful degradation: if the primary path fails, provide a degraded but functional experience
- For voice: never leave the user in silence — always speak a fallback message on failure
- Connection renewal for long-lived streams (Nova Sonic 8-min limit: renew at 7m30s)

## Infrastructure as Code

- Use **CDK** (TypeScript) or **SST** for all infrastructure — never manual console clicks
- One stack per environment (dev + prod minimum)
- Auto-deploy to dev on push to main; manual approval for prod
- Private subnets for compute; public subnets only for load balancers
- Auto-scaling configured (scale on CPU or concurrent connections)
- Secrets referenced from Secrets Manager in task definitions — not passed as plain env vars

## CI/CD (Simple Pipeline)

- **Lint** → **Test** → **Build** → **Deploy**
- Use GitHub Actions or GitLab CI (whichever the repo uses)
- Lint: ruff (Python) or eslint+prettier (TypeScript) — enforced, not advisory
- Test: pytest/vitest with coverage threshold
- Build: Docker image or CDK synth
- Deploy: `cdk deploy` or `sst deploy` with environment parameter
- No SonarQube, no Spectral, no oasdiff — add these when the team grows beyond 3 people

## Configuration

- All config validated at startup with typed schemas (Pydantic Settings or Zod)
- Fail fast if required config is missing — don't start the service
- Separate config per environment via environment variables (not config files per env)
- System prompts and AI configuration stored in Secrets Manager (not source code) — allows tuning without redeployment

## What to Add Later (When It Hurts)

| Trigger | Then Add |
|---------|----------|
| Second developer joins | PR reviews, CODEOWNERS, stricter CI gates |
| Latency complaints from users | X-Ray tracing, provisioned throughput |
| >1000 requests/day sustained | DynamoDB persistence, multi-AZ, scaling tuning |
| PCI audit required | Tier 1/Tier 2 account separation, full audit logging, PCI compliance controls |
| Third service in the ecosystem | Shared libraries, event bus, API contracts |
| Compliance/governance review | OpenAPI docs, ADRs, formal test strategy, SonarQube |
| Team grows to 5+ | Full Clean Architecture (3 layers), ArchUnit tests, shared CI templates |

## Anti-Patterns (Never Do)

- ❌ Kubernetes for a single service with <10 concurrent users
- ❌ Microservices when you have one team and one domain
- ❌ Terraform when you're AWS-only and CDK/SST gives better DX
- ❌ Multi-region for a service that doesn't need it yet
- ❌ Custom observability dashboards before you have users
- ❌ Event sourcing / CQRS for a simple CRUD or request-response service
- ❌ Spending more time on infrastructure than on the product
- ❌ Copying an enterprise platform's full architecture for a greenfield with 1/100th the traffic
