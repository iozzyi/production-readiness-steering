---
inclusion: always
description: Architecture decision principles for TypeScript-everywhere AWS projects using CDK. These guide design choices toward patterns that scale without costly refactoring.
---

# Architecture Decision Principles

These principles guide design choices for TypeScript-everywhere AWS projects (backend, frontend, IaC all in TypeScript). They prefer AWS CDK for infrastructure and patterns that avoid costly retrofitting.

## Language & Tooling

- **TypeScript everywhere** — backend (Node.js), frontend (React/React Native), infrastructure (AWS CDK). One language, one type system, shared interfaces.
- **AWS CDK over CloudFormation/Terraform** — type-safe constructs, same language as application code, composable patterns.
- **Zod for runtime validation** — schema validation at boundaries (API input, config files, environment variables). Compile-time types + runtime checks.
- **Vitest for testing** — fast, ESM-native, compatible with CDK assertions. Use across backend and CDK packages.

## Compute & Containers

- **Build images once, deploy everywhere** — push to ECR with immutable tags, reference by tag in CDK. Never rebuild during deploy.
- **Never use `ContainerImage.fromAsset()`** — always `fromEcrRepository(repo, tag)`. Asset-based builds add 3-5 minutes per deploy and prevent image pinning.
- **ARM64 Fargate** — 20% cheaper than x86. All npm installs in Docker builder stage (QEMU SIGILL in production stage).
- **Separate build workflow from deploy workflow** — image build triggers on source changes, deploy references the built image.
- **HTTP health endpoint required** — ALB health checks need HTTP (not TCP). Add `GET /health` even to WebSocket-only services.

## Load Balancing & Routing

- **ALB over NLB for application routing** — ALB inspects HTTP headers (Host, path). Required for host-based tenant routing. NLB cannot differentiate tenants.
- **Host-based routing for tenant isolation** — dedicated tenants get ALB rules (`IF Host == tenant.domain THEN forward to tenant target group`). Shared tenants hit the default rule.
- **CloudFront prefix list exceeds SG limits** — the managed prefix list has 130+ CIDRs. Use `anyIpv4()` for ALB security groups (ALB DNS is private, only CloudFront knows it).
- **ALB priority assignment via atomic writes** — use DynamoDB conditional writes to prevent race conditions when provisioning tenants in parallel.

## Multi-Tenancy

- **Per-tenant auth pools (not shared)** — physical isolation by construction. No cross-tenant data leakage risk, no pre-token-generation Lambda needed.
- **Tenant identified by JWT issuer** — `iss` claim contains pool ID → lookup tenant config from DynamoDB. No custom claims needed.
- **No config caching** — fresh DynamoDB read per connection (~5-10ms). Always current, no cache invalidation complexity.
- **Shared table for routing, per-tenant tables for data** — routing config (domain, pool ID, limits) in one shared table. User data in isolated per-tenant tables (IAM-level isolation, clean GDPR deletion).
- **Config-driven, not code-driven** — tenant differences expressed as configuration (DynamoDB), not code branches. Same image serves all tenants.

## Frontend & SSR

- **Custom HTML template with explicit placeholder** — `<!-- TENANT_CONFIG -->` replaced at request time. Never rely on string-replacing `</head>` (fragile across framework upgrades).
- **Manifest-based versioning** — deploy to immutable versioned paths (`s3://bucket/v1.4.0/`). `manifest.json` maps `"latest"` to concrete version. Rollback = update manifest.
- **Never `s3 sync --delete` to bucket root** — destroys other versions. Sync to versioned path without `--delete`.
- **Lambda@Edge for SSR injection** — injects tenant config into HTML at origin-request time. Frontend reads `window.__TENANT_CONFIG__` synchronously (no flash, no loading state).

## IAM & Security

- **Narrow roles over broad admin** — separate roles for each lifecycle step (CDK deploy, auth management, config seeding, role creation). Limits blast radius.
- **OIDC for CI/CD** — no long-lived credentials. GitHub Actions assumes roles via OIDC federation.
- **Environment-scoped trust policies** — per-environment roles trust `repo:ORG/REPO:environment:ENV`. Repo-level role trusts `repo:ORG/REPO:*` for utility workflows.
- **Resource IDs from SSM, never arguments** — CDK writes IDs to SSM at creation time. Scripts read from SSM. Eliminates wrong-value bugs.

## Data & State

- **DynamoDB for ephemeral state** — session leases, rate limit counters, tenant config. TTL for automatic cleanup.
- **TTL-based leases for capacity management** — write lease on acquire, delete on release. Crashes self-heal via TTL expiry (no manual reconciliation).
- **Atomic counters for rate limiting** — DynamoDB `UpdateExpression` with `if_not_exists`. Thread-safe without distributed locks.
- **Date-partitioned counters for daily limits** — key format `{tenantId}#daily#{YYYY-MM-DD}` with TTL = end of day + 1 hour. Auto-expires, no cron job.

## Infrastructure Patterns

- **Region parameterized, never hardcoded** — single source of truth in shared config. All scripts and CDK read from one place.
- **SharedInfra stack for cross-environment resources** — ECR, DynamoDB tables, SSM params. Never destroyed by per-environment destroy scripts.
- **SSM Parameter Store for cross-stack sharing** — no CloudFormation exports (block subsequent deploys). SSM is free, always readable, no coupling.
- **Feature-flag expensive resources** — observability, WAF, canaries toggled via environment variables. Deploy the code, toggle the cost.
- **Removal policy RETAIN for stateful resources** — Cognito pools, data tables survive `cdk destroy`. Scheduled cleanup for orphans (6+ months inactive).

## Deployment & Lifecycle

- **Git-driven tenant provisioning** — push YAML to `tenants/` → workflow validates → deploys → seeds config → verifies health.
- **Manual destruction only** — YAML deletion does NOT trigger teardown. Require explicit workflow dispatch with double-confirmation.
- **Sequential tenant lifecycle** — one provisioning at a time (concurrency group, `cancel-in-progress: false`). Eliminates shared-resource conflicts.
- **Config-driven drain period** — read `max_call_duration + 1 min` from tenant config. Never hardcode timeouts.
- **Tier changes are zero-downtime** — ALB routing is instant. Drain existing connections on old tier, then teardown.

## CI/CD Pipeline

- **Single `npm ci` at root** — workspace hoisting handles all packages. Never run `npm ci` in subdirectories after root install (corrupts `node_modules/.bin`).
- **CDK synth must not suppress errors** — never `2>/dev/null || true`. If synth fails, the build fails.
- **First-deploy safety** — check if ECR has images before deploying AppStack. Skip if empty (image workflow hasn't completed yet).
- **Self-healing deploy scripts** — detect and recover stuck stacks (ROLLBACK_COMPLETE, UPDATE_ROLLBACK_FAILED, DELETE_FAILED).
- **`cdk.context.json` committed to git** — CI needs availability zone context for `--no-lookups`. Add `.gitignore` exception if needed.

## Cost Optimization

- **Shared infrastructure for shared tenants** — marginal cost per tenant ≈ $0 (Cognito free tier + SSM free + DNS).
- **Dedicated tier only when justified** — own ECS tasks (~$37/month). Recommend only for compliance, guaranteed capacity, or custom scaling.
- **Scale to zero when not testing** — ECS desired count = 0 for idle environments.
- **Cost allocation tags on dedicated resources** — `tenant-id`, `isolation-tier`, `cost-center` for per-tenant billing visibility.
- **Observability never deployed to dev** — enforced in env-config defaults regardless of toggle values.
