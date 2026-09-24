---
name: pr-daily
description: Oversight-first daily PR maintenance for the current repo. Use when the user wants to triage and maintain their pull requests — "do my PR rounds", "pr daily", "review my PRs", "check my open PRs", "rebase and fix my PRs", daily PR cleanup. Resolves the right GitHub account first, fans out read-only parallel code reviews on PRs awaiting the user's review (never auto-submitted), and single-threads verification-gated fixes (rebase, review comments, failing CI) on the user's own PRs. Works on any repo and any stack — discovers conventions and checks from the repo itself. Do NOT use for a one-off review of a single specific PR, or to merge PRs.
metadata:
  version: 1.1.0
---

# pr-daily

You orchestrate a manual, oversight-first PR maintenance pass for the current repo. Invoke this skill when the user asks to do their PR rounds (also reachable as `/pr-daily`).

**Design (do not deviate):**
- The **review phase is read-only and parallel** — fan out one subagent per PR I review. Never submit a review without my explicit go-ahead.
- The **fix phase is single-threaded and leashed** — handle my own PRs one at a time, in the main context. Every push must be preceded by a passing verification check. Stop and ask on anything destructive or ambiguous (merge conflicts, force-push to shared branches, unclear review intent).
- **Resolve the right GitHub account first** (I may have multiple). See Phase 0.

**Models (set per subagent via the Agent tool's `model` option):**
- **Code review** (Phase 1 review subagents) → `opus` (the strongest model; reviews are correctness-critical).
- **All other work** (discovery, fix-phase subagents, everything else) → `sonnet` (general work).
- Note: this only controls *subagent* models. The main orchestrating loop runs on whatever model the session is set to — run the session on `sonnet` for the orchestration and Phase 1 will still escalate review subagents to `opus`.

---

## Phase 0 — Resolve account & discover (main context, no writes)

Figure out which GitHub account to use for *this* repo before anything else — don't assume a single global identity (I may have several `gh` accounts, e.g. one per org):

1. Get the repo owner from the remote: `git remote get-url origin`, parse the owner segment (e.g. `…:SomeOrg/some-repo.git` → `SomeOrg`).
2. List logged-in accounts: `gh auth status`.
   - **One account** → use it.
   - **Several** → pick the one that can access this owner. Prefer an account already tied to that org; if unclear, test each candidate (`gh api repos/<owner>/<repo> --jq .full_name`, or `gh api orgs/<owner> --jq .login`) and choose the one that succeeds. If still ambiguous, STOP and ask me which account to use — don't guess and run discovery against the wrong one.
3. If the chosen account isn't active, switch: `gh auth switch --user <account>`.
4. Confirm the active identity: `gh api user --jq .login` — that login is `me`. If it isn't the account you intended, STOP and tell me.

> **Token-override caveat:** if a `GITHUB_TOKEN` (or `GH_TOKEN`) env var is set, `gh` uses it and **silently ignores `gh auth switch`** (it prints "the value of the GITHUB_TOKEN environment variable is being used"). Check up front (`printenv GITHUB_TOKEN GH_TOKEN`). If either is set, **prefix every `gh` call** in all phases — and in subagents — with `env -u GITHUB_TOKEN -u GH_TOKEN` so account switching takes effect. If neither is set, plain `gh` is fine.

Then build the worklist:

- My open PRs: `gh pr list --author "@me" --state open --json number,title,headRefName,isDraft,mergeable`
- PRs awaiting my review: `gh pr list --search "review-requested:@me state:open" --json number,title,headRefName,author`

Print two numbered lists (My PRs / PRs to review). If both are empty, say so and stop.

---

## Phase 1 — Review phase (PARALLEL, thorough, read-only)

For each PR awaiting my review, spawn **one subagent on the `opus` model** (code review is correctness-critical — see Models). Run them concurrently — multiple Agent calls in a single message. This is a **real, deep code review**, not a status glance.

Each review subagent must:

1. **Set up the PR for review in an isolated worktree** (so the review sees the full code in context, not just a flat diff):
   - `gh pr checkout <n>` inside a fresh worktree for the PR branch (locate an existing worktree first via `git worktree list`; create one with `git worktree add` if none), based on the repo's default branch (`origin/HEAD` → usually `main`, but detect it — don't assume).
   - Capture context: `gh pr view <n> --json title,body,reviews,statusCheckRollup` (has the author/me already reviewed? CI green/red — if red, which checks).
2. **Discover the target repo's own conventions** so the review is tailored, not generic: read whatever convention sources the repo ships — `CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `CONTRIBUTING.md`, `docs/` — for its framework, state/styling/data layers, and perf or architecture rules. Don't assume any particular stack; learn it from the repo. Then review the PR's diff vs the default branch at **high effort**: if a `/code-review` skill is available, use it (do **NOT** pass `--comment` or `--fix` — this review must not post anything or touch the branch); otherwise perform the review inline yourself, following the same structure below. Either way it only analyzes and reports.
3. **Write the full review to a markdown file**: `.pr-reviews/pr-<n>-review.md` in the repo root. Structure it as: PR title + one-line summary; CI status; "already reviewed by me?"; then findings grouped **Correctness / bugs** → **Conventions** (the target repo's own conventions discovered in step 2) → **Reuse / simplification / efficiency**, each finding with `file:line` (or `file:start-end` for a range), severity, and a concrete suggested fix. End with an overall recommendation (approve / comment / request-changes) and rationale.

   Every finding must carry the exact anchor needed to post it inline later: `path`, `line` (and `start_line` if it spans lines), and `side` (`RIGHT` unless the point is about a deleted line). Findings that genuinely have no single location (e.g. "the PR is missing tests") go in a short summary-body section instead. The `.md` is for me to read — but write each finding's text as the comment you'd actually post, in the style described in Phase 3, so posting is copy-paste, not a rewrite.
4. Return a compact line for the orchestrator: `PR #n | <title> | CI: green/red/pending | my review: done/none | <n> findings (x critical) | report: .pr-reviews/pr-<n>-review.md`.

Collect all report lines. Do **not** run `gh pr review` / `gh pr comment` for anything yet — hold for Phase 3. The point of the `.md` is that **I read it and tell you which findings to actually post and which to drop** before any review is submitted.

---

## Phase 2 — Fix phase (worktree-isolated, leashed)

Work my own PRs that need changes. Skip drafts unless I ask. **All work happens in the branch's git worktree, never in the main checkout** — I usually created the PR from a worktree, so one likely already exists.

**Delegation:** because each PR lives in its own isolated worktree, the per-PR work below is safe to delegate to one **subagent per PR** on the `sonnet` model (general work — see Models; parallel writes don't collide). Default to single-threaded if I have only a couple of PRs; fan out subagents when there are several. Either way the leash rules hold: a subagent must NOT `git push` without passing the verification gate (step 4), and must report back rather than force-push through conflicts or ambiguity.

For each PR:

1. **Locate or create the worktree**
   - `git fetch origin` first.
   - `git worktree list` — if an existing worktree is already checked out to `headRefName`, use it (operate with `git -C <path>` / run commands from that dir). Do not disturb the main checkout's working tree.
   - If none exists, create one: `git worktree add <path> <headRefName>` (e.g. under a sibling `../<repo>-worktrees/<branch>` or the project's usual worktree location if there is one).
   - Then rebase **inside that worktree** onto the repo's default branch (detect it via `git symbolic-ref refs/remotes/origin/HEAD`; usually `origin/main`): `git -C <path> rebase origin/<default>`.
   - **If conflicts you cannot resolve trivially → STOP, report the conflicting files, leave the worktree as-is, do not force-push.** Move to the next PR.

2. **Address review comments**
   - `gh pr view <n> --comments` and `gh api repos/{owner}/{repo}/pulls/<n>/comments` for inline threads.
   - Apply fixes that are clear and in scope. For anything ambiguous, note it for the summary instead of guessing.
   - **After a comment is addressed, 👍 it** so the reviewer sees what's been handled. Capture each comment's `id` when fetching, then react once the fix is in:
     - inline review comment → `gh api -X POST repos/{owner}/{repo}/pulls/comments/<comment-id>/reactions -f content=+1`
     - PR-level (issue) comment → `gh api -X POST repos/{owner}/{repo}/issues/comments/<comment-id>/reactions -f content=+1`
     - Prefix with `env -u GITHUB_TOKEN -u GH_TOKEN` if Phase 0 found a token override. Only react to comments you actually addressed — never to ones you skipped, deferred, or noted as ambiguous. Re-reacting is idempotent (already-reacted returns 200), so no need to check first.
   - **Then re-request review from that reviewer**, so the PR lands back in their queue instead of sitting silently. Do this per reviewer, only once **all** of their comments are 👍'd and the fixes are pushed (step 4) — never with unaddressed comments outstanding or with the fixes still local:

     ```sh
     gh api -X POST repos/{owner}/{repo}/pulls/<n>/requested_reviewers -f 'reviewers[]=<login>'
     ```

     If some of a reviewer's comments were skipped or ambiguous, leave the re-request alone and put it on the "needs your attention" list instead — reply to them first, that's my call to make.

3. **Fix failing CI**
   - `gh pr checks <n>`. If red, fetch the failing job log (`gh run view <run-id> --log-failed`) and fix the root cause locally.

4. **VERIFY before any push (mandatory gate)**
   - Detect the project's checks (don't assume a stack): look at `package.json` scripts (`typecheck`/`lint`/`test`), `Makefile`, `justfile`, or the repo's CI workflow, and run the relevant ones. Run tests if you touched code they cover.
   - **Only if they pass** → push from the worktree: `git -C <path> push --force-with-lease`.
   - If they fail and you can't fix in a couple of iterations → STOP, report, leave the branch unpushed.

5. Record the outcome for this PR (and the worktree path used), then proceed to the next. Leave worktrees in place — they're likely mine; don't `git worktree remove` anything I didn't create this run unless I ask.

---

## Phase 3 — Synthesize & gate

Print one table:

| PR | mine/review | action taken | CI | status | needs me |
|----|-------------|--------------|----|--------|----------|

Then, for the review phase, **point me to each `.pr-reviews/pr-<n>-review.md` and give a 2–3 line digest** (finding count, criticals, your recommended verdict). **Wait for me to say which findings to post and which to drop**, and whether to submit as comment / approve / request-changes. Post only after I confirm, including only the findings I greenlit.

### How to post (inline first, short prose)

**Anchor to code, not to a wall of text.** Anything that has a location goes as an inline comment on that line or line range — not as a paragraph in the review body that makes the reader hunt for the file. Post all findings as one review in a single call:

```sh
gh api -X POST repos/{owner}/{repo}/pulls/<n>/reviews \
  -f event=COMMENT -f body="<1–3 line summary>" \
  -f 'comments[][path]=src/foo.ts' -F 'comments[][line]=42' -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=…'
```

For a range, add `start_line` (and `start_side`) alongside `line` — `line` is the **last** line of the range. Prefix with `env -u GITHUB_TOKEN -u GH_TOKEN` if Phase 0 found a token override.

The review **body** stays 1–3 lines: overall take plus anything with no location. Everything else is inline.

**Comment style — write like a colleague, not a linter:**

- Short. 1–3 sentences per comment. One point per comment; two problems on the same line = two comments.
- Plain words. No jargon or coined phrases, no "footgun", "code smell", "leaky abstraction", "non-idiomatic", "cognitive load", "consider extracting a seam". Say what breaks and when: "if `items` is empty this throws" beats "this violates the guard-clause pattern".
- Concrete over abstract. Name the variable, the input, the case that fails.
- Suggested code goes in a GitHub `suggestion` block when the fix is a small edit — a diff the author can click-apply beats a paragraph describing it.
- No praise padding, no restating the diff back at the author, no "great work but". Skip preamble entirely.
- No severity labels or emoji prefixes in posted comments (those are for the `.md`). If something is optional, one word does it: "nit:".
- If a comment needs more than ~4 lines to explain, it's probably several findings or a design discussion — put that one in the body as a short paragraph and say so.

End with a short "needs your attention" list: unresolved conflicts, ambiguous review comments, PRs left unpushed, and any review awaiting my go-ahead.
