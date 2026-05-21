---
name: logical-commits
description: Split working-tree changes into multiple logical commits using Conventional Commits format. Use when the user says "split commits", "logical commits", "break this into commits", "commit my changes", "commit strategy", or has accumulated multi-concern changes on a branch. Auto-detects the repo's commit conventions from recent history and asks for type/scope/approval through interactive selections (AskUserQuestion) instead of free typing. Never bypasses git hooks. Do NOT use for amending existing commits, rewriting history, interactive rebase, or pushing as the primary goal — only for creating new local commits from current changes.
license: MIT
metadata:
  version: 1.0.0
---

# logical-commits

Turn the current working tree into a series of clean, conventional commits. Auto-detect the repo's commit conventions, propose a grouping, gate every decision through `AskUserQuestion`, and respect whatever hooks the repo has configured.

## Core principles

- **Propose, then gate.** You inspect the diff and propose the full split. The user approves or redirects through `AskUserQuestion` — never ask them to type out grouping decisions.
- **Selections over typing.** Use `AskUserQuestion` for type, scope, and approval gates. Free text is only for the commit description (and only when the user wants to edit yours).
- **Conventional Commits, hooks honored.** Format: `type(scope): description`. Never pass `--no-verify`, `--no-gpg-sign`, or `-c commit.gpgsign=false`.
- **Adapt to the repo.** Detect conventions from history. Don't assume a fixed scope list or a particular monorepo layout — let the data tell you.
- **Honest cohesion.** If the change is genuinely one thing, make one commit and say so. Don't fabricate splits.
- **One commit at a time.** Stage → commit → verify, group by group. If a commit fails a hook, stop and ask before continuing.

## Workflow

### Step 1 — Preflight

Run these checks. Abort with a clear message if any fail.

```bash
git rev-parse --is-inside-work-tree   # must print "true"
test -f .git/MERGE_HEAD && echo "merge in progress" && exit 1
test -d .git/rebase-merge -o -d .git/rebase-apply && echo "rebase in progress" && exit 1
test -f .git/CHERRY_PICK_HEAD && echo "cherry-pick in progress" && exit 1
```

If `git status --porcelain` is empty: tell the user there's nothing to commit and stop.

If there are **already-staged** changes, do not silently include them. Ask:

> AskUserQuestion: "You already have staged changes. How should I handle them?"
>
> - Include them in the planning (treat as part of working tree)
> - Unstage everything first and plan from a clean slate
> - Cancel

### Step 2 — Inspect

Get the change inventory:

```bash
git status --porcelain=v1
git diff --stat HEAD
```

For grouping decisions, fetch per-file diffs **lazily** — only when you need to read the content of a specific file to decide what bucket it belongs in:

```bash
git diff HEAD -- <file>
```

Don't dump every diff into context upfront. For most files the path + status (M/A/D/??) is enough.

### Step 3 — Discover conventions

Pull the repo's actual conventions from recent history:

```bash
git log -30 --pretty=%s
```

Extract type + scope from each subject with the pattern `^([a-z]+)(\(([^)]+)\))?:`. Build two deduped lists, sorted by frequency:

- **Types** actually used in this repo (e.g. some repos only use `feat`/`fix`/`chore`; others use the full set including `docs`, `perf`, `test`, `ci`, `style`, `build`, `refactor`).
- **Scopes** actually used in this repo (these become the scope options offered in Step 6).

If fewer than ~5 commits follow Conventional Commits, the repo may not use this convention. Surface the finding and ask:

```
AskUserQuestion:
  "This repo doesn't appear to use Conventional Commits consistently. How should we format commits?"
  - Use Conventional Commits anyway (type(scope): description)
  - Match the freeform style I see in recent history
  - Cancel
```

Optionally inspect `commitlint.config.*`, `.commitlintrc*`, `.czrc`, or `package.json` (commitlint key) at the repo root for explicit rules — if found, they override what's inferred from history.

### Step 4 — Propose the grouping

Group files by the **concern** they serve, not by physical location. Useful heuristics:

1. **Co-located feature code** — files under the same feature/module directory, or files that import each other, usually belong together. Read the diff content when path alone is ambiguous.
2. **Supporting changes follow the feature** — translations, fixtures, mocks, snapshots, assets, generated types, lockfile updates, and small consumer-site wiring that exist _only_ to support a feature should ride with that feature's commit, not a separate "i18n" or "assets" commit.
3. **Cross-cutting infrastructure** — tooling, CI, build config, dependency bumps, formatter/lint config — usually a separate `chore`/`build`/`ci` commit.
4. **Pure refactors with no behavior change** — their own `refactor` commit.
5. **Unrelated fixes tangled into a feature change** — separate `fix` commit, often via a hunk-level split (Step 7) when one file mixes concerns.
6. **Documentation-only changes unrelated to a code change** — separate `docs` commit.
7. **Genuinely cohesive change** — if everything serves one concern, propose **one** commit and explicitly say why. Don't fabricate splits.

Present the proposal as plain text. For each group show: index, draft type/scope/description, and the files (or hunks) it owns. Mark with `[+]` added, `[M]` modified, `[D]` deleted. If a file appears in multiple groups, flag it — that signals a hunk-level split (Step 7).

```
Proposed split (N commits):

[1] <type>(<scope>): <draft description>
    [+] path/to/added-file
    [M] path/to/modified-file
    [M] path/to/shared-file (hunks at lines X–Y only)

[2] <type>(<scope>): <draft description>
    [M] path/to/another-file
    [M] path/to/shared-file (remaining hunks)
```

### Step 5 — Approval gate

```
AskUserQuestion:
  "Does this grouping look right?"
  - Approve and start committing
  - Regroup (merge two groups or split one)
  - Edit a specific group's type/scope/description
  - Cancel
```

If "Regroup" or "Edit": ask follow-up `AskUserQuestion`s with 2–4 options each to drive the change (e.g. "Which two groups should merge?" with the group numbers as options). Loop back to Step 4 until approved or cancelled.

### Step 6 — Commit each group, in order

For each group in the approved proposal:

**6a. Pick the type.** Show the 3 types most likely to fit this specific change, with the strongest match flagged as Recommended. Pull from the types detected in Step 3 — never offer a type this repo hasn't used unless none of the detected types fit. Reserve the fourth slot for "Other" so the user can type a type manually if needed.

```
AskUserQuestion (header: "Type"):
  "What type fits this commit?"
  - <best-fit> (Recommended) — <one-line meaning>
  - <next-best>
  - <next-best>
```

(The user always has an implicit "Other" option to type freely.)

**6b. Pick the scope.** Show up to 3 detected scopes from Step 3 that plausibly fit this group + a "new scope" option:

```
AskUserQuestion (header: "Scope"):
  "Which scope?"
  - <best-fit detected scope> (Recommended)
  - <other detected scope>
  - <other detected scope>
  - New scope (I'll type it)
```

If "New scope", follow up:

```
AskUserQuestion (header: "New scope"):
  "Type the new scope:"
  → user provides via "Other"
```

Scope is optional per Conventional Commits. If the change is genuinely cross-cutting or the repo's recent commits often omit scope, offer "(no scope)" as one of the options instead of forcing one.

**6c. Confirm the description.** Draft a short imperative description (no trailing period, lowercase first word unless it's a proper noun). Match the casing/length conventions visible in recent history (e.g. some repos cap subjects at 50/72 chars; some prefer sentence case). Show it:

```
Proposed message:
  <type>(<scope>): <description>

AskUserQuestion:
  "Use this message?"
  - Use it
  - Edit (I'll rewrite the description)
  - Regenerate (try a different angle)
```

If "Edit", the user types via "Other". If "Regenerate", re-read the staged files' diffs and write a different description.

**6d. Stage.**

- File-level group: `git add -- <file1> <file2> ...`. Never use `git add -A` or `git add .` — only stage what's in this group.
- Hunk-level group: see Step 7.

**6e. Commit.** Use a HEREDOC. Do **not** add `Co-Authored-By` (this is the user's own work; only add a co-author trailer if the user explicitly asks).

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <description>
EOF
)"
```

**6f. Verify.** Run `git status` and `git log -1 --oneline`. Confirm the commit landed.

If the commit failed (commit-msg hook rejected the format, pre-commit hook rejected the content):

```
AskUserQuestion:
  "Commit failed. How should we proceed?"
  - Show me the hook output and let me decide
  - Fix and retry (I'll address the issue, then retry this same group)
  - Skip this group (unstage and move to the next)
  - Abort remaining groups (stop here)
```

**Never** retry with `--no-verify`. If the hook failure is a commit-message format issue, fix the message and retry — don't bypass.

### Step 7 — Hunk-level path

Triggered when a single file's diff cleanly splits between two groups — typical cases: a shared file (config, translation table, constants module) touched for two unrelated reasons, or a refactor entangled with an unrelated fix in the same source file.

For each such file:

1. `git diff HEAD -- <file>` to see the hunks (each `@@ -L,N +L,M @@` block)
2. Decide hunk-to-group assignment when proposing in Step 4 (show the decision in the proposal text)
3. At staging time for a group:
   - Generate a patch containing only that group's hunks
   - Save to a temp file: `/tmp/logical-commit-<n>.patch`
   - `git apply --cached /tmp/logical-commit-<n>.patch`
4. If the patch fails to apply (overlapping hunks, whitespace drift): fall back to file-level grouping for that file, surface the issue, and re-confirm via `AskUserQuestion`:

```
AskUserQuestion:
  "Hunk split failed for <file>. Fall back to putting the whole file in [group N]?"
  - Yes, whole file → group N
  - Whole file → a different group (I'll pick)
  - Skip this file for now
```

Clean up temp patches after each group.

### Step 8 — Push gate

After all groups committed successfully:

```bash
git log --oneline @{u}..HEAD 2>/dev/null || git log --oneline -N   # N = number of commits made
git status
```

Then:

```
AskUserQuestion:
  "All commits done. Push the branch now?"
  - Push now
  - Stay local (I'll push later)
```

If "Push now":

- Branch has upstream: `git push`
- Branch has no upstream:

```
AskUserQuestion:
  "This branch has no upstream. Set 'origin/<branch>' as upstream?"
  - Yes, push and set upstream
  - No, cancel push
```

Never use `--force` or `--force-with-lease` unless the user explicitly asked for it earlier in the session.

### Step 9 — Final report

Print a tight summary:

```
Committed N changes:
  <sha1> <type>(<scope>): <description>
  <sha2> <type>(<scope>): <description>

Pushed to origin/<branch> ✓   (or "Local only — push when ready")

Working tree: clean   (or list any leftover untracked/unstaged files)
```

## Examples

### Example A — cohesive feature with supporting files (one commit)

Working tree (illustrative — exact paths will differ per repo):

```
[M] app/screens/billing.ext
[+] app/features/billing/components/invoice-card.ext
[+] app/features/billing/components/usage-meter.ext
[+] app/features/billing/store.ext
[M] locales/en.json              (only billing.* keys added)
[M] locales/de.json              (only billing.* keys added)
[+] app/features/billing/assets/
```

Diff inspection shows: every new file is consumed by the billing feature; the locale changes only add `billing.*` keys; the assets are referenced by the new components.

Proposal:

> **One commit** — everything serves the new billing feature. Locale keys are exclusively `billing.*`; assets are consumed by the new components. Splitting into "feat + i18n + assets" would be artificial.

- Type: `feat` (Recommended), with `chore` and the next-most-common detected type also offered
- Scope: `billing` (Recommended, if seen in history) or "(no scope)" / "New scope"
- Draft description: `add billing screen with invoice and usage components`

### Example B — bug fix tangled with refactor (hunk split, two commits)

Working tree:

```
[M] src/list-view.ext     (contains both a perf refactor AND an unrelated bug fix)
[M] src/button.ext        (related to the refactor)
[M] config/constants.ext  (one constant tied to the fix, one tied to the refactor)
```

Diff inspection reveals two clean concerns. Proposal:

```
[1] fix(<detected-scope>): correct stale render when filter changes
    [M] src/list-view.ext        (hunks at lines 142–162 only)
    [M] config/constants.ext     (hunk for FILTER_DEBOUNCE only)

[2] refactor(<detected-scope>): replace inline handlers with memoized variants
    [M] src/list-view.ext        (remaining hunks)
    [M] src/button.ext
    [M] config/constants.ext     (hunk for RENDER_BATCH_SIZE only)
```

This requires hunk-level staging (Step 7) for `src/list-view.ext` and `config/constants.ext`. If a patch fails to apply, fall back to the AskUserQuestion in Step 7 and let the user pick the safer grouping.

### Example C — non-conventional repo

Step 3 finds the recent log doesn't use `type(scope): description` at all (e.g. subjects look like `Add billing flow`, `Fix render bug`). The skill asks via AskUserQuestion whether to introduce Conventional Commits or match the existing freeform style, then proceeds with the chosen format. All later prompts (type/scope) are skipped or adapted in freeform mode — only the approval and description-confirmation gates remain.

## Troubleshooting

### `commit-msg` hook rejects the message

A commit-message linter (commitlint, conform, gitlint, custom hook, etc.) found a rule violation. The hook output names the failing rule (subject-case, type-enum, scope-enum, header-max-length, etc.). Fix the message — do not retry with `--no-verify`. Common causes:

- Subject casing wrong (often must be lowercase first word) → rewrite description
- Type not in the repo's allowed list → re-pick type from Step 3's detected list
- Scope not in the repo's allowed list → re-pick scope from Step 3's detected list
- Header exceeds max length (often 72 or 100 chars) → tighten the description

### `pre-commit` hook rejects the content

The commit didn't happen. The hook (lint, typecheck, format, test, secrets scan, etc.) found something wrong with the staged content. Fix the underlying issue, re-stage the same files, then create a **new** commit — never `--amend`, because there's nothing to amend (the previous commit didn't land).

### Patch fails to apply (hunk split)

Usually whitespace drift or overlapping hunks. Fall back to file-level grouping for that file via the AskUserQuestion in Step 7. If even file-level grouping is ambiguous, surface the diff and ask the user to decide.

### Repo not detected

If `git rev-parse --is-inside-work-tree` doesn't return "true", abort and tell the user the current directory isn't a git repository.

### Branch in detached HEAD

Abort. Tell the user to checkout a branch first — making commits on detached HEAD orphans them.

### Working tree dirty after a partial commit

Expected when groups remain. Continue with the next group. The final report in Step 9 lists any leftover unstaged files at the end.

## Hard rules

- Never use `git add -A`, `git add .`, or `git add -u`. Always stage explicit paths.
- Never pass `--no-verify`, `--no-gpg-sign`, or `-c commit.gpgsign=false`.
- Never use `--amend` inside this workflow.
- Never push with `--force` or `--force-with-lease` unless the user explicitly authorized it.
- Never add `Co-Authored-By: Claude` to commits — the user is the author of these changes.
- Never use interactive flags (`git rebase -i`, `git add -i`, `git add -p` in interactive mode). Drive everything through non-interactive commands.
