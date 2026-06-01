---
inclusion: always
description: CI/CD pipeline patterns, deploy scripts, container builds, CDK infrastructure, and GitHub Actions configuration. Works alongside production-readiness.md which covers application-level concerns.
---

# GitHub Actions + Infrastructure as Code (CDK)

Generic rules and patterns for CI/CD pipelines deploying AWS infrastructure via CDK with GitHub Actions.

---

## Pipeline Architecture

### Recommended Pipeline Structure

```
Push to main:
  build-image (if backend/** changed — separate workflow)
      ↓ (image in ECR)
  lint-test-build (quality gate — must pass before any deploy)
      ↓
  deploy-dev-iac (SharedInfra + NetworkStack + AppStack)
      ↓
  deploy-dev-backend ──┐ (force new ECS deployment)
  deploy-dev-frontend ─┘ (versioned S3 path + manifest update)
      ↓
  ┌─ e2e-test (non-WebSocket — starts immediately)
  └─ e2e-test-websocket (WebSocket tests — 60s warmup + retries)
      ↓ (both must pass)
  deploy-staging (IAC + backend + frontend, sequential)
      ↓
  e2e-test-staging
      ↓
  deploy-prod (manual approval gate)
      ↓
  e2e-test-prod (smoke only)
```

### Separate Image Build from Deploy

**Critical pattern:** Build Docker images in a dedicated workflow, push to ECR, then reference by tag during CDK deploy. Never build Docker during `cdk deploy`.

```yaml
# build-backend-image.yml — triggers on backend/** changes
# Tags: sha-<short>, v<semver>, latest
# Pushes to ECR, stores tag in SSM

# ci.yml — triggers on all changes
# Reads image tag from SSM, passes to CDK via context
# CDK uses ContainerImage.fromEcrRepository(repo, tag)
```

Benefits:
- Deploy time drops 3-5 minutes (no Docker build)
- Same image deployed to all environments (immutable)
- Tenants can pin specific image versions
- Rollback = change tag reference (no rebuild)

### PR Environments (Ephemeral)

```
Push to PR:
  lint-test → deploy PR<number> (dev config) → e2e-test → comment PR with URLs

PR close:
  Auto-destroy all PR stacks
```

---

## Core Rules

### 1. Job Dependency Order

- IAC deploys before backend and frontend (they depend on stack outputs)
- Backend deploys before frontend (frontend reads config from backend stack outputs)
- Frontend should NEVER trigger IAC as a fallback (causes race conditions)
- E2E tests depend on ALL deploy jobs completing
- Staging/prod deploys should always run after dev E2E passes (no change-detection gating)

### 2. Concurrency

- Add workflow-level `concurrency` with `cancel-in-progress: true` — prevents two runs deploying to the same environment simultaneously
- Use `group: ${{ github.workflow }}-${{ github.ref }}` for branch-level concurrency
- PR environments use `group: pr-${{ github.event.pull_request.number }}`
- Tenant lifecycle workflows: `cancel-in-progress: false` (never interrupt provisioning)

### 3. CDK Diff Gate (Skip Unnecessary Deploys)

Run `cdk diff --no-lookups` before deploying. If no changes detected, exit early. This saves 10-15 minutes on non-infrastructure changes.

Place the diff check inside the deploy script (not the workflow) so it works for all environments automatically.

### 4. Container Image Strategy (ECR)

**Build once, deploy everywhere:**
- Dedicated workflow builds ARM64 image on `backend/**` changes
- Tags: `sha-<short>` (immutable), `v<semver>` (from package.json), `latest` (mutable)
- CDK references image via `ContainerImage.fromEcrRepository(repo, tag)` — never `fromAsset()`
- Image tag stored in SSM parameter for deploy scripts to read
- QEMU only needed in the image build workflow — NOT in deploy workflows

**First-deploy chicken-and-egg:**
- On first push, ECR is empty but CDK tries to deploy ECS service referencing `latest`
- Fix: deploy script checks if ECR has images before deploying AppStack
- If empty: deploy only SharedInfra + NetworkStack, skip AppStack
- AppStack deploys on next run after image workflow completes

**Dockerfile rules for CI:**
- All `npm install` in builder stage (QEMU SIGILL in production stage)
- `COPY node_modules` from builder to production (no npm execution in final stage)
- Self-contained tsconfig (no `extends: ../tsconfig.base.json` — outside Docker context)
- Use `npm install` not `npm ci` (workspace lockfile lives at repo root, not in subdirectory)

### 5. CloudFormation / CDK

- No cross-stack exports for resources that change — use SSM parameters
- Custom Resources need explicit `node.addDependency()` for ordering
- Never deploy the same stack from multiple parallel jobs
- Handle `UPDATE_ROLLBACK_COMPLETE` as deployable (not fatal)
- Handle `CREATE_IN_PROGRESS` as recoverable (wait and retry)
- Deploy stacks in explicit order: SharedInfra → Network → App
- Self-healing deploy scripts should handle stuck stacks (ROLLBACK_COMPLETE, DELETE_FAILED)
- `cdk.context.json` must be committed to git (CI needs it for `--no-lookups`)
- Add `.gitignore` exception if a global `*.json` rule would exclude it

### 6. ARM64 / Docker on x86_64 CI Runners

- Add `docker/setup-qemu-action` ONLY in the image build workflow
- Do NOT add QEMU to deploy workflows (CDK no longer builds Docker)
- Dockerfile should use explicit `--platform=linux/arm64`
- All npm installs in the builder stage, COPY node_modules to production stage

### 7. GitHub Actions Configuration

- Top-level `permissions: id-token: write, contents: read, pull-requests: write` for OIDC
- `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true` env var to suppress Node 20 deprecation warnings
- All secrets must be full ARNs (not just names) — validate format in setup scripts
- `prepare` script in package.json should use `husky || true` (CI doesn't have husky during npm ci)
- **Never run workspace-level `npm ci` after root `npm ci`** — corrupts hoisted `node_modules/.bin` symlinks
- CDK synth should NOT suppress errors (`2>/dev/null || true` hides real failures)

### 8. Secrets, Variables & OIDC Roles

**Three levels of GitHub configuration:**

| Level | Scope | Used by |
|-------|-------|---------|
| Repo-level secrets | All workflows | `build-backend-image.yml`, `frontend-deploy.yml` |
| Repo-level variables | All workflows | `AWS_REGION`, `PROJECT_PREFIX` |
| Per-environment secrets | Workflows with `environment:` key | `ci.yml` deploy jobs |

**Critical: workflows without `environment:` can only read repo-level secrets.**

If a workflow doesn't specify `environment:`, it CANNOT read per-environment secrets. Either:
- Add `environment: dev` to the job, OR
- Set the secret at repo level

**OIDC trust policy scoping:**
- Per-environment roles: trust condition `repo:ORG/REPO:environment:ENV_NAME`
- Repo-level role: trust condition `repo:ORG/REPO:*` (any job on any branch)
- Both are needed — per-env for deploy jobs, repo-level for build/utility workflows

**Setup script must configure:**
1. OIDC Identity Provider in AWS
2. Per-environment IAM roles (dev, staging, prod) with environment-scoped trust
3. Repo-level IAM role with broad trust (for non-environment workflows)
4. Per-environment secrets: `AWS_ROLE_ARN`
5. Repo-level secret: `AWS_ROLE_ARN` (pointing to repo-level role)
6. Repo-level variables: `AWS_REGION`, `PROJECT_PREFIX`
7. Per-environment variables: `AWS_REGION`, `PROJECT_PREFIX`, feature flags

### 9. Testing in CI

- All tests visiting protected routes must seed auth tokens in `onBeforeLoad`
- WebSocket-dependent tests go in a separate parallel job with 60s warmup + `--config retries=2`
- Integration tests (real Cognito) excluded from local CI runs — only run against deployed environments
- Use `testID` selectors, not hardcoded text (i18n strings and config values change)
- Conversation/WebSocket test scripts need retry loops (3 attempts, 15s between)

### 10. Code Coverage

- `babel.config.js` must be tracked in git (check `.gitignore` for `*.js` rules)
- Use static export (not dev server) for coverage — deferred scripts break coverage collection
- Remove `defer` from exported HTML script tags
- Always include istanbul in babel config (env var approach fails with Metro worker threads)

### 11. Region Configuration

- **Never hardcode regions** in deploy scripts or workflows
- Use a single source of truth: `common.sh` sets `AWS_REGION` from `.env` or defaults
- All workflows use `${{ vars.AWS_REGION || 'eu-west-2' }}` pattern
- CDK app reads region from context or environment variable
- CloudFront prefix lists are region-specific — use a mapping or `anyIpv4()` for ALB SGs
- ACM certificates for CloudFront must be in us-east-1 regardless of primary region

---

## Multi-Tenant Architecture

### Shared Infrastructure Model

- **One Docker image** serves all tenants (configured via DynamoDB, not env vars)
- **One ECS cluster** with shared tasks (tenant identified by JWT `iss` claim)
- **One ALB** with host-based routing (dedicated tenants get own routing rule)
- **One CloudFront distribution** with multiple CNAMEs
- **Lambda@Edge SSR** injects tenant config into HTML at request time
- **Per-tenant**: Cognito pool, DynamoDB data table, SSM parameters, DNS record

### Isolation Tiers

| Tier | Compute | Routing | Cost |
|------|---------|---------|------|
| Shared | Shared ECS tasks | Default ALB rule | ~$0/tenant |
| Dedicated | Own ECS tasks + auto-scaling | Host-based ALB rule | ~$37/month |

### Tenant Lifecycle (Git-Driven)

```
New YAML in tenants/ → push → validate → CDK deploy → seed config → verify health
Modified YAML → push → validate → update DynamoDB config
Destroy → manual workflow_dispatch with double-confirmation
```

- YAML deletion does NOT trigger destroy (safety)
- Sequential execution (one tenant at a time)
- Each step uses a different narrow IAM role

### Frontend Versioning (Manifest-Based)

- Frontend deploys to immutable versioned S3 path: `s3://bucket/v1.4.0/`
- `manifest.json` maps `"latest"` to concrete version
- Lambda@Edge resolves version per tenant (pinned or latest)
- Rollback = update manifest (no file operations)
- Old versions remain indefinitely for pinned tenants
- CloudFront invalidation targets only `/manifest.json` (not `/*`)

### Narrow IAM Roles

Use separate roles for each lifecycle step (not one broad admin):
- CDK deploy role (CloudFormation + CalledVia condition)
- Auth management role (Cognito only)
- Config seeding role (DynamoDB + SSM only)
- Role creation role (IAM scoped to specific prefix)

### Resource Discovery via SSM

- CDK writes resource IDs to SSM at creation time
- Deploy/seed scripts read from SSM (never accept IDs as arguments)
- Eliminates wrong-value bugs from copy-paste or stale variables
- Pattern: `/project/shared/*` for platform resources, `/project/tenant-{id}/*` for per-tenant

---

## Deploy Script Patterns

### Self-Healing Deploy

```bash
# Pre-flight: check stack health, heal stuck stacks
# Install: root npm ci only (never workspace-level)
# Resolve: read image tag from SSM
# Safety: check ECR has images before deploying AppStack
# Diff check: skip if no changes
# Deploy with retry: handle transient errors, wait for IN_PROGRESS
# Post-deploy: validate stack health, verify ECS service
```

### Frontend Deploy (Versioned)

```bash
# Determine version from package.json
# Build static export
# Sync to s3://bucket/v{version}/ (no --delete, immutable)
# Update manifest.json atomically
# Invalidate CloudFront for /manifest.json only
```

### Destroy Script

```bash
# Safety gate for prod (require CONFIRM=yes)
# Empty S3 buckets BEFORE stack deletion
# Delete in reverse dependency order (App → Network)
# Never destroy shared infrastructure (ECR, DynamoDB, SSM)
# Retry on DELETE_FAILED (find failed resources, empty them, retry)
```

---

## Common Failure Patterns

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `eslint: not found` | Redundant workspace npm ci after root install | Remove workspace-level npm ci |
| `exec format error` | ARM64 image on x86_64 runner | Add docker/setup-qemu-action to BUILD workflow only |
| Exit code 132 (SIGILL) | npm install in ARM64 production stage via QEMU | Move all npm to builder stage |
| `Cannot delete export...in use by` | Cross-stack CloudFormation reference | Use SSM parameters instead |
| `Stack is in UPDATE_IN_PROGRESS` | Parallel jobs deploying same stack | Sequence with `needs:` |
| `Credentials could not be loaded` | Missing `id-token: write` permission | Add top-level permissions |
| `AccessDenied` on role assume | Workflow missing `environment:` key | Add environment or use repo-level role |
| `Source Account ID is needed` | Role name instead of full ARN | Use full ARN format |
| ECS can't pull image | ECR empty on first deploy | Skip AppStack if no images in ECR |
| Coverage 0/0 = 100% | babel.config.js gitignored by `*.js` rule | Add exception to .gitignore |
| WebSocket test timeout | ECS cold start after deploy | Warmup delay + retries |
| Two runs fighting over same stack | No concurrency control | Add cancel-in-progress |
| S3 bucket DELETE_FAILED | Bucket not empty | Empty before delete |
| Frontend overwrites other versions | `s3 sync --delete` to bucket root | Sync to versioned path, no --delete |
| CDK synth fails silently in CI | `2>/dev/null \|\| true` hides errors | Remove error suppression |
| `cdk.context.json` missing in CI | File gitignored | Add exception: `!cdk/cdk.context.json` |
| SG rule limit exceeded | CloudFront prefix list has 130+ CIDRs | Use `anyIpv4()` for ALB SG (DNS is private) |
| Deploy script uses wrong region | Hardcoded default in shared script | Single source of truth in `common.sh` |
| Tenant lifecycle can't assume role | OIDC trust scoped to environment only | Create repo-level role with broad trust |

---

## Quick Validation Commands

```bash
# Verify secrets exist (repo + environment levels)
gh secret list
gh secret list --env dev
gh variable list

# Check stack health
aws cloudformation describe-stacks --region <region> \
  --query 'Stacks[?contains(StackName,`<prefix>`)].{Name:StackName,Status:StackStatus}' \
  --output table

# Check ECS service health
aws ecs describe-services --cluster <cluster> --services <service> \
  --query 'services[0].{running:runningCount,desired:desiredCount}'

# Check ECR has images (first-deploy validation)
aws ecr describe-images --repository-name <repo> \
  --query 'length(imageDetails)' --output text

# Check what caused a rollback
aws cloudformation describe-stack-events --stack-name <stack> \
  --query 'StackEvents[?contains(ResourceStatus,`FAILED`)].[LogicalResourceId,ResourceStatusReason]'

# Force ECS redeployment (after image push)
aws ecs update-service --cluster <cluster> --service <service> --force-new-deployment

# Verify OIDC role trust policy
aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument' --output json
```
