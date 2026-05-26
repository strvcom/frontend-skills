---
name: supply-chain-check
description: Audit JS supply-chain hygiene (Safe Chain on dev machine, .npmrc/equivalent in repo, GitHub Actions CI gates, CONTRIBUTING.md mention) and offer interactive fixes. INVOKE ONLY when the user explicitly types `/supply-chain-check` — do NOT auto-invoke based on project type, lockfile presence, security mentions, or any related context.
---

# Supply-chain hygiene audit

A defense-in-depth audit against npm-ecosystem supply-chain attacks (typosquatting, hijacked maintainer accounts, malicious postinstall scripts). Walks the user through their developer machine + the current project + CI workflows, then offers fixes one-by-one with explicit confirmation.

## Operating rules (read first)

1. **Audit before fix.** Always run the full audit and present findings before proposing any change.
2. **Confirm every write.** Show the file path and the exact content you will write. Ask y/n. Never batch-apply.
3. **Never execute remote installers.** If Safe Chain is missing, print the install command and link; do not run `curl ... | sh` yourself.
4. **Never auto-edit CI workflow files.** YAML structure varies (matrices, reusable workflows). Print the snippet and the suggested path; let the user place it.
5. **Render policy values at runtime.** Read `policy.json` from this skill's directory and substitute `{{VERSION}}` etc. into templates before showing them.
6. **Surface intentional overrides as warnings, not failures.** `ignore-scripts=false` is a deliberate choice — flag for review, recommend `@lavamoat/allow-scripts`, do not propose flipping.

## Phase 1 — Detect environment

Run these in parallel:

- `which safe-chain` — captures G1
- Read `~/.safe-chain/config.json` (may not exist) — captures G2
- `ls` the cwd for lockfiles to identify the package manager
- Check for `package.json` — if absent, exit with "Not a JS project, nothing to audit."

Lockfile priority (first match wins):

| Lockfile | Manager | Extra check |
|---|---|---|
| `bun.lockb` | bun | — |
| `pnpm-lock.yaml` | pnpm | Run `pnpm --version` to branch v9 vs v10+ |
| `yarn.lock` + `.yarnrc.yml` | yarn berry | — |
| `yarn.lock` alone | yarn classic v1 | Emit warning, skip P2, continue with P1/P3/CI |
| `package-lock.json` | npm | — |

For yarn classic: emit `⚠ yarn classic detected — no native min-release-age support. Recommend upgrading to Yarn Berry v3+ or switching to pnpm.` Do not propose P2 fixes.

Also check `.github/workflows/` — if it exists, run CI checks. If `.gitlab-ci.yml` / `azure-pipelines.yml` / etc. exist, emit one `⚠ non-GitHub CI detected — unchecked, see https://github.com/AikidoSec/safe-chain#cicd-integration`.

## Phase 2 — Audit checklist

Present findings as a table. Format: `[✓/✗/⚠] ID — short label — current → expected`.

**Global (machine-level):**

| ID | Check |
|---|---|
| G1 | `safe-chain` binary on PATH |
| G2 | `~/.safe-chain/config.json` has `minimumPackageAgeHours ≥ 72` |

**Project (repo-level):**

| ID | Check |
|---|---|
| P1 | Lockfile present and not gitignored |
| P2 | PM-specific config has policy values (see below) |
| P3 | `CONTRIBUTING.md` or `README.md` mentions Safe Chain install |

**P2 per manager** (load thresholds from `policy.json`):

- **npm:** `.npmrc` contains `ignore-scripts=true`, `min-release-age=3`, `engine-strict=true`
- **pnpm v10+:** `pnpm-workspace.yaml` contains `minimumReleaseAge: 4320` and an explicit `onlyBuiltDependencies:` key. Do **not** flag absence of `ignore-scripts` — scripts are disabled by default in v10+.
- **pnpm v9-:** treat as npm — expect `.npmrc` with `ignore-scripts=true`, `min-release-age=3`
- **yarn berry:** `.yarnrc.yml` contains `npmMinimalAgeGate: 3d` and `enableScripts: false`
- **bun:** `bunfig.toml` contains `[install]` section with `minimumReleaseAge = 259200`. Do **not** flag absence of script-disable — bun disables postinstalls by default.

**`ignore-scripts=false` handling:** if found in `.npmrc`, emit `⚠ intentional override detected — ensure @lavamoat/allow-scripts is configured as the allowlist (https://github.com/LavaMoat/LavaMoat/tree/main/packages/allow-scripts)`. Mark as warning, not failure. Do not propose flipping.

**CI (only if `.github/workflows/` exists):**

| ID | Check |
|---|---|
| C1 | At least one workflow runs a frozen-lockfile install (`npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable`, `bun install --frozen-lockfile`) AND has a Safe Chain install step before it |
| C2 | Safe Chain installer URL pins a specific version (matches `policy.json#safeChainInstallerVersion` or another concrete version), not `latest` |
| C3 | The install step does not use bare `npm install` / `yarn install` (without `--immutable`) |
| C4 | `SAFE_CHAIN_LOGGING: verbose` is set on the Safe Chain install step |

After the table, print: `N passing, M failing, K warnings`. If M+K > 0, ask: *"Walk through fixes interactively?"* — y/n. If no, stop here.

## Phase 3 — Interactive fix loop

Iterate failing checks in this order (least invasive first). For each: show the exact change, ask y/n, apply on yes, skip on no.

### Order

1. **G2 — Safe Chain global config**
   - If `~/.safe-chain/config.json` is missing: show the JSON, confirm, write it.
   - If present but `minimumPackageAgeHours < 72`: read existing, show diff, ask before overwriting that key only (preserve other keys).

2. **P2 — PM config**
   - Read existing config file if any.
   - Compare key-by-key against the policy.
   - For each missing/incorrect key: show old value vs new value, ask y/n.
   - Use `Edit` for surgical merges (preserve unrelated keys, comments, ordering). Only use `Write` if the file does not exist.
   - For `ignore-scripts=false` in `.npmrc`: do not propose change. Print the warning text from Phase 2 and move on.

3. **P3 — CONTRIBUTING/README mention**
   - If `CONTRIBUTING.md` exists: offer to append `templates/contributing.snippet.md` to it.
   - Else if `README.md` exists: offer to append to README, OR create `CONTRIBUTING.md` — ask which.
   - Else: offer to create `CONTRIBUTING.md` from the snippet.

4. **C1–C4 — GitHub Actions** *(never edit workflow files automatically)*
   - For each failing CI check, print the rendered `templates/github-actions.snippet.yml` (with `{{VERSION}}` substituted from `policy.json`).
   - State which file the user should edit (e.g. `.github/workflows/ci.yml`) and which step it should go before.
   - Do not run Edit or Write on workflow files.

5. **G1 — Safe Chain not installed**
   - Print: `Safe Chain is not on PATH. Install with:` followed by the official install command and the link `https://github.com/AikidoSec/safe-chain#installation`.
   - Do not execute.

After applying each fix, re-run that single check silently and confirm it now passes. If it still fails, surface why and stop the loop.

## Phase 4 — Final summary

Print:

- Updated pass/fail counts after fixes.
- A one-line list of files changed in the repo (use `git diff --stat` if the cwd is a git repo, else list paths you wrote to).
- Call out `~/.safe-chain/config.json` separately if you touched it — it is not in the repo.
- If C1–C4 had failures, remind the user that CI workflow edits are still pending (you printed the snippet but did not apply it).

## Templates and policy

Available in this skill directory:

- `policy.json` — version pins and threshold values. Always load before rendering templates.
- `templates/npmrc` — full recommended `.npmrc` body.
- `templates/pnpm-workspace.snippet.yaml` — keys to merge into `pnpm-workspace.yaml`.
- `templates/yarnrc.snippet.yml` — keys to merge into `.yarnrc.yml`.
- `templates/bunfig.snippet.toml` — keys to merge into `bunfig.toml`.
- `templates/contributing.snippet.md` — the "Security setup (required)" block.
- `templates/github-actions.snippet.yml` — Safe Chain + frozen install steps. Substitute `{{VERSION}}` from `policy.json#safeChainInstallerVersion` before showing.

Templates are written verbatim into user repos. Do not paraphrase them.
