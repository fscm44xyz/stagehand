# Review rules (v3 branch)

Scope: everything under `packages/` and `.github/`. CI (`.github/workflows/ci.yml`) does not cover all of these; verify them by reading the diff.

## Published package surface
- `@browserbasehq/stagehand` ships two builds. `package.json#exports` maps `import` to `dist/esm` and `require` to `dist/cjs`; `files` lists both trees plus the `./cli` entry in both formats. Any edit to `exports`, `files`, `main`, `types`, `scripts/build-esm.ts` or `scripts/build-cjs.ts` changes what npm consumers receive: flag it, and confirm every path named in `exports` and `files` is produced by the build scripts, for both module formats.
- `packages/evals` and `packages/server-v3` are `private: true`. Flag anything that makes them publishable or adds `publishConfig`.
- `browse` (`packages/cli`) publishes from `v3` only, with a Docker image. Changes to its `bin`, `files` or manifest generation alter the published CLI.

## Generated files are not source
- `lib/version.ts`, `lib/v3/dom/build/**`, `packages/server-v3/openapi.v3.yaml` and `oclif.manifest.json` are build outputs (`turbo.json` lists them as outputs and excludes them from inputs). A hand edit to any of them is a bug: it is overwritten by `gen-version`, `build-dom-scripts`, `gen:openapi` or `manifest`. Ask for the change in the source.
- `openapi.v3.yaml` feeds Stainless SDK generation on every PR (`stainless.yml`). A route or schema change in `packages/server-v3/src` without the regenerated spec, or a spec edit without a source change, is a mismatch.

## Release
- `release.yml` publishes with npm trusted publishing: it needs `permissions.id-token: write` and an `actions/setup-node` step configured with `registry-url`. Removing, replacing or reordering either breaks publishing and nothing in PR CI will notice. Flag every edit to `release.yml` that touches `permissions`, the setup-node step, the npm upgrade step or the changesets step.
- Versions come from changesets. A behaviour change under `packages/core`, `packages/cli` or `packages/server-v3` without a `.changeset/*.md` will not be released: flag it. A changeset naming a `private` package is ignored by the release workflow.
- `browse` and `server-v3` have their own release paths (`release-cli.yml`, `stagehand-server-v3-release.yml`) keyed on their `package.json` versions; version bumps there must go through those workflows, not by hand.

## CI gating
- `run-lint` failure cancels the whole workflow. Lint is `prettier --check`, `eslint` and `tsc --noEmit` per package; anything that would fail formatting or type-checking is a bug, including test files.
- Jobs are selected by the `paths-filter` groups in `ci.yml` (`core`, `cli`, `evals`, `server`, `docs-only`). A change that adds a package or moves files between packages must update those filters, or its tests never run.
- Browserbase e2e tests and evals with secrets only run when `is_internal_head` is true. For a PR from a fork, nothing that needs API keys has executed: do not accept "tests pass" for e2e/bb or evals, review that code by hand.
- Eval categories are chosen by labels (`act`, `observe`, `extract`, `agent`, `combination`, `targeted-extract`); `regression` runs unless `skip-regression-evals` is set. Changes to `packages/evals` must keep `evals.config.json` defaults valid.
- `concurrency.cancel-in-progress` is on: a workflow edit that changes the `group` key can cancel unrelated runs.

## Code conventions
- ESLint forbids `eval`, `new Function`, implied eval and dynamic function construction (`security/detect-eval-with-expression`). Flag new dynamic code execution, including in DOM scripts under `lib/v3/dom`.
- `preserve-caught-error` is enforced: a rethrown error must carry the original as `cause`.
- Every package is `type: module` with `moduleResolution: node`. New imports must resolve for both the ESM and the CJS build.
- Node `^20.19 || >=22.12` and pnpm 9.15 are pinned. Flag syntax or APIs that need a newer Node, and lockfile changes with no matching `package.json` change.
- The PR template is `# why`, `# what changed`, `# test plan`. An empty section is a flag.
