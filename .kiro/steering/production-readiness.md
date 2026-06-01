---
inclusion: always
description: Production-readiness standards covering application code quality, security, testing, resilience, and observability. For CI/CD pipeline patterns, deploy scripts, and GitHub Actions configuration, see the github-iac skill.
---

# Production Readiness Standards

These standards cover **application-level** concerns: code architecture, security, testing, error handling, resilience, and observability. For CI/CD pipeline structure, deploy scripts, container builds, and GitHub Actions configuration, activate the `github-iac` skill.

## PCI-DSS

These standards do NOT override PCI-DSS requirements. If handling card data, apply PCI compliance standards separately.

## Architecture (Hexagonal / Ports & Adapters)

- `backend/src/domain/` — Pure business logic, ports (interfaces), models — ZERO external dependencies
- `backend/src/adapters/driven/` — Outbound adapters (Bedrock, Cognito, DynamoDB, SSM, session pool, tenant resolver, rate limiter)
- `backend/src/adapters/driving/` — Inbound adapter (WebSocket server with HTTP /health endpoint)
- `backend/src/config.ts` — Validates all environment variables at startup with Zod
- `backend/src/main.ts` — Composition root, wires ports to adapters
- Domain logic MUST NOT import from adapters — dependency flows inward only
- Every adapter implements a domain port interface
- Tenant context threaded through all service calls (never global state)

## 5 Non-Negotiables (Never Skip)

1. **Secrets in Secrets Manager** — never in environment variables, never in source code, never in git. Use AWS Secrets Manager or SSM Parameter Store. CDK wires secrets to the runtime.
2. **Input validation** — every external input (WebSocket message, tenant YAML) validated with Zod before processing. Reject malformed input immediately.
3. **Tool/action allowlist** — explicit tool registry with rate limiting and risk classification (LOW/MEDIUM/HIGH). HIGH-risk tools require human approval.
4. **Structured logging** — JSON format via pino, correlation IDs + tenant ID on every log line, PII redacted. Never `console.log()` in production code.
5. **Circuit breaker on external dependencies** — any call to Bedrock, DynamoDB, or third-party APIs must have a circuit breaker (3 failures in 60s → OPEN → 30s cooldown). For voice: never leave the user in silence.

## Security

- All data encrypted at rest (KMS) and in transit (TLS 1.2+)
- Four narrow IAM roles for tenant lifecycle — never one broad admin role
- WAF on public-facing endpoints (feature-flagged)
- No hardcoded credentials, API keys, or tokens anywhere in the codebase
- Validate authentication on every WebSocket connection (JWT via per-tenant Cognito pool)
- Per-tenant data isolation: own DynamoDB table, own Cognito pool
- Never log secrets, tokens, or full PII — log references only
- VPC with private subnets for compute; public subnets only for load balancers

## Multi-Tenant Application Design

Design for multi-tenancy from the start in application code:

### Tenant Resolution
- Per-tenant Cognito pools (physical isolation, no shared pool)
- JWT `iss` claim → pool ID → DynamoDB lookup → full tenant config
- No caching — fresh read per connection (~5-10ms, negligible)
- Tenant config drives: system prompt, model, limits, theme, features

### Rate Limiting (Application Layer)
- Per-tenant, not per-user (tenant's users share the tenant's quota)
- Atomic DynamoDB counters for concurrent sessions and daily calls
- Config-driven limits from tenant config (not hardcoded)
- Fail closed on DynamoDB errors (reject connection)
- WebSocket close codes: 4029 (concurrent), 4030 (daily), 4031 (duration)

### Session Pool (Cross-Region Capacity)
- DynamoDB TTL-based leases for external service session management
- Distribute across available regions by priority (closest first)
- Self-healing: crashed sessions expire via TTL (no manual cleanup)
- Capacity validated at provisioning time (sum of tenant limits ≤ platform max)

## Error Handling

- Define domain errors with typed error codes (`ErrorCode` string union in `errors.ts`)
- Never expose stack traces or internal details to external consumers
- Log errors with context (session ID, tenant ID, operation, input summary)
- Return structured error responses: `{ code, message }` — never raw exception strings
- Fail fast on configuration errors at startup — don't discover missing config mid-call

## Testing (80% Coverage — No Exclusions)

- Test ALL source files — never exclude adapters or services from coverage to fake thresholds
- Test the domain/orchestration layer thoroughly (this is where bugs live)
- Test input validation (ensure malformed input is rejected)
- Test tenant resolver (valid/unknown/inactive tenant, DynamoDB failure)
- Test session pool (acquire/release/capacity exhaustion/TTL calculation)
- Test rate limiter (under limit/at limit/counter edge cases)
- Test adapters with mocked AWS SDK clients (`vi.mock("@aws-sdk/client-*")`)
- Test WebSocket adapter with mocked `ws` module (`vi.mock("ws")` with EventEmitter-based mocks)
- Use `vitest` with coverage threshold enforced in CI — build fails below 80%
- CDK assertions tests for all stacks
- Cypress E2E tests for all frontend flows
- Tenant YAML validation tests (Zod schema + platform-wide constraints)

## Observability (Feature-Flagged)

All observability resources are feature-flagged via environment variables.

### Always deployed (all environments):
- **5 CloudWatch alarms:** Error rate, latency p99, circuit breaker OPEN, connection capacity, auth failures
- **Structured logs** → CloudWatch Log Groups (14-day dev, 30-day staging, 90-day prod)
- **VPC Flow Logs** → CloudWatch Logs
- **Per-tenant alarms** for dedicated tier (CPU, task count, unhealthy targets)

### Feature-flagged (staging + prod only):
| Feature | Toggle | Cost |
|---------|--------|------|
| CloudWatch Dashboard | `ENABLE_DASHBOARD=true` | ~$3/mo |
| Synthetics Canary | `ENABLE_CANARY=true` | ~$5/mo |
| Load Test | `ENABLE_LOAD_TEST=true` | Free |

### Never deployed to dev:
Observability resources never deployed to dev regardless of toggle values. Enforced in env-config defaults.

## Resilience

- Circuit breaker on every external dependency (3 failures in 60s → OPEN → 30s cooldown)
- Retry transient errors once with 500ms delay, then circuit breaker
- Timeout on all external calls (Bedrock: 10s, DynamoDB: 5s)
- Graceful degradation: if the primary path fails, provide a degraded but functional experience
- For voice: never leave the user in silence — always speak a fallback message on failure
- Connection renewal for long-lived streams (renew before platform timeout)
- ECS circuit breaker with auto-rollback on deployment failure
- Session pool TTL self-heals on crashes (no manual reconciliation)
- Config-driven drain period for tier changes (max_call_duration + 1 min buffer)

## Configuration

- All config validated at startup with Zod
- Fail fast if required config is missing — don't start the service
- Tenant config in DynamoDB (not environment variables)
- Resource IDs discovered via SSM (never passed as arguments)
- System prompt per tenant in DynamoDB (tunable without redeployment)
- Feature flags for expensive resources via environment variables
- Region default in shared deploy config — single source of truth

## Three Environments

| Environment | Purpose | Observability |
|-------------|---------|---------------|
| **dev** | Developer iteration | Alarms + logs only |
| **staging** | Pre-prod validation | Full (when toggled on) |
| **prod** | Live traffic | Full (when toggled on) |

## Anti-Patterns (Application Level)

- ❌ Shared Cognito pool for multi-tenant — use per-tenant pools
- ❌ Cache tenant config in memory — fresh read per connection
- ❌ Hardcode rate limits — read from tenant config
- ❌ One broad IAM admin role — use narrow roles per lifecycle step
- ❌ Exclude files from coverage to fake passing thresholds
- ❌ `console.log()` in production code — use structured logger
- ❌ Expose stack traces to clients — return error codes only
- ❌ Global mutable state for tenant context — thread through calls
- ❌ Manual reconciliation for leaked sessions — use TTL self-healing
- ❌ Hardcode timeouts for drain periods — read from config
