# Install Dependencies After Adding Them

Whenever you add, remove, or change a dependency in any `package.json` file, you MUST run the install command in that package's directory immediately — before making further code changes or running tests.

## Rules

1. **After editing `package.json`** in any workspace (`backend/`, `frontend/`, `cdk/`, or root), run `npm install` in that directory right away.
2. **Do not batch installs.** If you add a dependency to `backend/package.json` and then need to add one to `frontend/package.json`, install in backend first, then edit and install in frontend.
3. **Use exact versions** when adding new dependencies (e.g., `"zod": "3.23.8"` not `"zod": "^3.23.8"`). Pin to the latest stable version.
4. **Commit the lockfile.** The updated `package-lock.json` is part of the change — never gitignore it or skip it.

## Commands by Directory

| Directory | Command |
|-----------|---------|
| Root | `npm install` |
| `backend/` | `cd backend && npm install` |
| `frontend/` | `cd frontend && npm install` |
| `cdk/` | `cd cdk && npm install` |

## Why This Matters

- Local builds and tests will fail if dependencies aren't installed
- CI/CD uses `npm ci` which reads the lockfile — a stale lockfile breaks deploys
- TypeScript type checking fails without `@types/*` packages physically present in `node_modules`
