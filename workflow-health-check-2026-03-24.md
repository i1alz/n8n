# Workflow Health Check Report

Date: 2026-03-24 (UTC)
Repository: `/workspace/n8n`
Scope: `packages/workflow`

## Commands run

1. `pushd packages/workflow >/dev/null && pnpm lint && pnpm typecheck && pnpm test && popd >/dev/null`
2. `pnpm install`
3. `pushd packages/workflow >/dev/null && pnpm lint && pnpm typecheck && pnpm test && popd >/dev/null`
4. `pushd packages/workflow >/dev/null && pnpm typecheck && popd >/dev/null`
5. `pushd packages/workflow >/dev/null && pnpm test && popd >/dev/null`
6. `pnpm build > build.log 2>&1` (partial run, log captured)

## Findings

### 1) Lint is failing to start
- `pnpm lint` fails because ESLint cannot load `@n8n/eslint-config/dist/configs/base.js`.
- Error: `ERR_MODULE_NOT_FOUND` from `packages/workflow/eslint.config.mjs` import resolution.

### 2) Typecheck reports widespread failures
- `pnpm typecheck` fails with many TypeScript errors.
- Major error clusters:
  - Missing workspace modules/types: `@n8n/errors`, `@n8n/expression-runtime`, `@n8n/config`, `@n8n/vitest-config/node`.
  - Error model mismatch (e.g. missing `message`, `level`, `name`, `stack` properties on error classes).
  - Test typing failures driven by the above model/type mismatches.

### 3) Tests fail to boot
- `pnpm test` (`vitest run`) fails before tests execute.
- Root cause: cannot resolve `@n8n/vitest-config/dist/node.js` in workflow package config.

### 4) Build log notes
- A partial monorepo build log was captured in `build.log`.
- Log includes repeated LightningCSS warnings about `:global(...)` pseudo-class usage from the chat/design-system CSS pipeline.

## Overall status

`packages/workflow` is **not healthy/runnable** in the current local state.

Primary blockers are dependency/module resolution for internal workspace packages and resulting TypeScript/test config failures.
