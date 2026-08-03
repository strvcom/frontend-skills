---
name: aikido-jira
description: Turn Aikido security findings into Jira tickets — one ticket per root cause, deduplicated against tickets that already exist, behind a single review gate. Use when the user wants to triage Aikido findings, file security tickets from a scan, asks which vulnerabilities still need tickets, or types `/aikido-jira`. The first run in a repo configures itself (verifies the Aikido and Jira MCP connections, harvests the repo's existing ticket conventions, writes a committed config file); later runs go straight to triage. Requires an Aikido MCP server and an Atlassian MCP server. Do NOT use to run a new code scan (that is Aikido's own scan tool), to fix the findings themselves, or to file non-security tickets.
metadata:
  version: 1.0.0
---

# Aikido → Jira triage

Reads the Aikido security feed for the current repository, groups findings by root cause,
skips anything already ticketed, and files the rest in a configured Jira project. Aims at
**one question per run**: the review gate. Everything else comes from a committed per-repo
config file, so one person configures the repo and the whole team just runs it.

Works in any repo and any Jira project — all specifics live in the config, never in this
file.

## Operating rules (read first)

1. **Nothing is written before the gate.** No Jira ticket, no Aikido mutation, until the
   user has answered the review table. Reading is free; writing is not.
2. **One ticket per root cause**, never one per finding. Aikido's `issue_group_id` is the
   unit — sixteen CVEs fixed by one dependency bump are one ticket, not sixteen.
3. **Jira is the only source of truth for dedup.** Every created ticket carries the label
   `aikido-<issue_group_id>`; re-runs query for those labels. Never keep a local state
   file — it drifts.
4. **Follow every paginated response to the end.** Both the Aikido feed and Jira's search
   paginate. A half-read page silently becomes a duplicate ticket or a missed finding.
5. **Never commit, push, or open a PR.** Writing the config file is fine; landing it is
   the user's call.
6. **Ticket keys are never bare.** Print full URLs (`<site_url>/browse/<KEY>`) — Jira does
   not auto-link bare keys and terminals cannot linkify them.
7. **Findings detail outranks house style.** If the repo's convention forbids file paths
   in tickets, every other section obeys it, but the Findings section keeps `file:line`
   — a security fix is unactionable without locations. Note the exception in the config.
8. **Ignoring a finding needs a stated reason**, always from the user, never inferred.

## Phase 0 — Load the config

Look for the config file, in order: `.agents/aikido-jira.md`, then
`.claude/aikido-jira.md`. If neither exists, or the file lacks `jira.cloud_id`,
`jira.project_key`, or `aikido.repo_name`, say so and **run [Setup](#setup) now**, then
continue into Phase 1 in the same run. Do not make the user re-invoke the skill.

The frontmatter holds the machine constants; the body holds the repo's ticket-writing
conventions and must be honored when drafting.

Arguments override the config for this run only:

```
/aikido-jira            # scope = severity_threshold from config
/aikido-jira all        # every severity, including low
/aikido-jira high       # high (and critical) only
/aikido-jira setup      # reconfigure, then triage
```

## Phase 1 — Fetch the whole feed

Call the Aikido issues-list tool (`aikido_issues_list`) with `repo_name` from config —
plus `workspace_name` when the config sets one — starting at `page: 0`, then increment
`page` until a page returns fewer than 25 findings. **Never stop at page 0**; real feeds
span several pages.

Empty feed → report the repo is clean and stop.

## Phase 2 — Group by root cause

Group findings by `issue_group_id`. Per group record: the **highest** member severity, the
finding count, the issue type (`open_source`, `leaked_secret`, `sast`, `iac`, `mobile`,
`cloud`, …), and every member's file, line, and Aikido link.

Apply the scope (run argument, else `severity_threshold`). Groups below scope are **not
dropped silently** — they collapse into one summary line in the table, naming how many
findings and groups were held back and which argument surfaces them.

## Phase 3 — Dedup against Jira

Query in chunks of roughly 40 group ids:

```
searchJiraIssuesUsingJql
  jql: project = <project_key> AND labels in (aikido-<gid1>, aikido-<gid2>, ...)
  fields: ["labels", "status", "summary"]
  maxResults: 100
```

**Paginate:** while the response has `isLast: false`, re-issue the same query with the
returned `nextPageToken`, copied verbatim (a mistyped token errors out), until `isLast` is
`true`. Stopping early hides existing tickets and files duplicates.

Then classify each group:

| Situation | Action |
|---|---|
| No ticket carries the label | Propose creation |
| Ticket exists, status category not Done | Already tracked — list its URL, propose nothing |
| Ticket exists and is **Done**, group still in the feed | **Needs attention** — report it; never silently re-ticket |

The Done-but-still-reported case is a judgment call (regression, incomplete fix, or just a
stale scan), so it belongs to the user, not to this skill.

## Phase 4 — Draft the tickets

**Title** — name the remediation, not the symptom:

| Issue type | Title shape |
|---|---|
| `open_source` | `Bump <package> from <installed> to <target> (<n> findings)` — target = highest patched version recommended across members |
| `leaked_secret` | `Verify & rotate <what> in <file>` (or `across <n> files`) |
| `iac` / `cloud` | Imperative fix, e.g. `Restrict bastion SSH ingress to trusted IPs` |
| `sast` / `mobile` / other | The group's issue title, plus ` (<n> locations)` when it spans files |

**Body** — this template is fixed; the config's conventions may adjust tone, labels, and
extra fields, but must not remove the Findings section or its `file:line` detail:

```markdown
**Goal:** Resolve <n> <severity> Aikido finding(s): <one-line root cause>.

**Findings:**
- `<file>:<line>` — <finding title or CVE id> — [Aikido](<issue_link>)
  <dependency groups: installed → patched version>

**Remediation:** <Aikido's remediation text, deduplicated across members, trimmed>

**Done when:**
- <the observable fix, e.g. "<package> is at <target> on the default branch">
- Aikido no longer reports this issue group for the repo.
```

Add a caveat line where the fix carries risk the ticket should not hide — a major-version
jump, native binaries, a finding that is plausibly a false positive worth confirming
before touching code.

If the config names a story-points field, propose a value per ticket using the conversion
recorded in the config body.

## Phase 5 — The review gate (the only question)

Show one table, ordered by severity, then **stop and wait**:

| # | Sev | Proposed ticket | Findings | Existing | Recommendation |
|---|-----|-----------------|----------|----------|----------------|

Follow it with the already-tracked list, the needs-attention list, and the out-of-scope
summary line. Then invite a single reply combining any of:

- `all` · `all except 3 5` · `only 1 4` — what to **create**
- `skip 2` — leave it in the feed for a later run
- `ignore 7: sample tokens in API examples` — **ignore in Aikido**, reason required

Point out groups that look like false positives (example tokens in fixtures or generated
files, rules that do not apply to the module they fired on) so the user can weigh
ignoring them — but never pre-select an ignore.

## Phase 6 — Execute

**Count first, reconcile after.** Write down the approved group ids as an explicit
checklist before the first call and work through it. Long batches are easy to abandon
halfway, and a run that files 6 of 11 tickets and then reports success is a failure the
user should never be the one to catch.

Per approved group:

```
createJiraIssue
  cloudId: <jira.cloud_id>
  projectKey: <jira.project_key>
  issueTypeName: <jira.issue_type>
  summary: <title>
  description: <body>
  contentFormat: "markdown"
  assignee_account_id: <jira.assignee, omit when null>
  additional_fields: {
    "labels": [<jira.labels...>, "aikido-<gid>"],
    "parent": { "key": "<jira.epic>" },        // omit when null
    "<story_points_field>": <points>           // omit unless configured and approved
  }
```

If a create call rejects `parent` or a custom field inline, create the issue first and set
the field with `editJiraIssue`.

Per ignored group: the Aikido ignore tool takes **one finding id**, so loop over every
member finding of the group, passing the user's stated reason each time.

A failed call does not stop the batch — carry on, record it, and report it. Label-based
dedup makes a re-run pick up exactly the gaps.

**Before writing the report**, re-run the Phase 3 dedup query and confirm the number of
tickets found matches the number approved. Fill any gap, then report.

## Phase 7 — Report

- **Created** — one full URL per line.
- **Already tracked** — group → existing ticket URL.
- **Ignored in Aikido** — group + the reason given.
- **Needs attention** — Done tickets whose groups are still reported.
- **Out of scope / failed** — the held-back summary line, and any call that errored.

## Setup

Runs on demand (`/aikido-jira setup`) and automatically when Phase 0 finds no config.
Writes the config file but **never commits it** — ask the user to review and commit so the
team inherits the setup.

Reconfiguring an existing repo: show the current values, ask what should change, and
re-run only the affected steps.

**1. Verify Aikido.** Derive a candidate repo name from `git remote get-url origin`
(basename, `.git` stripped) and call `aikido_issues_list` with it.

| Failure | What it means |
|---|---|
| Tool not available | The Aikido MCP server is not connected — have the user install/enable it, then retry |
| Auth error | Sign in via the Aikido login tool (`aikido_login`), then retry |
| `400 Feature is disabled for workspace …` | The workspace blocks MCP feed access; an Aikido admin must enable it at <https://app.aikido.dev/settings/integrations/ide/mcp/permissions>. Stop until it is on |
| Empty result | Aikido's repo name likely differs from the git remote — ask the user for the name as it appears in Aikido and retry. Capture `workspace_name` too if the org runs several workspaces |

**2. Verify Jira.** Call `getAccessibleAtlassianResources` for the `cloudId` and site URL;
ask which site when several come back. A failure here means the Atlassian MCP server is
not connected.

**3. Harvest existing conventions** before asking the user anything. Read any skill in the
repo that already creates Jira tickets (`.claude/skills/*/SKILL.md`, `.agents/skills/*/`),
plus `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md`. Pull out the project key, custom-field
ids (story points, sprint), label taxonomy, epic conventions, title and key-linking style,
and estimate rules. Reuse rather than re-ask — and never delegate to that other skill at
runtime; its interactive per-ticket menus would defeat batch triage.

**4. Interview only the gaps**, in one compact round, each with a recommendation:
project (confirm the harvested key or pick from `getVisibleJiraProjects`), default issue
type (recommend `Task`), default labels (recommend `security`), default epic or parent,
default assignee (recommend unassigned), and the severity threshold for a bare run
(recommend `medium`). Ask about story points only if a story-points field was harvested.

**5. Write the config** — to `.agents/aikido-jira.md` when `.agents/` exists, else
`.claude/aikido-jira.md`:

```markdown
---
jira:
  cloud_id: "<uuid>"
  site_url: "https://<site>.atlassian.net"
  project_key: "<KEY>"
  issue_type: "Task"
  labels: ["security"]
  epic: null               # "<KEY-123>" or null
  assignee: null           # accountId, or null for unassigned
  story_points_field: null # e.g. "customfield_10602"; null = never set
aikido:
  repo_name: "<name as it appears in Aikido>"
  workspace_name: null     # only for multi-workspace orgs
severity_threshold: "medium"  # bare-run scope: high | medium | low (low = everything)
---

# Ticket conventions

<Harvested prose: title style, how ticket keys are referenced, tone, estimate
conversion, epic guidance — plus the standing exception that triage tickets keep the
Findings table with file:line detail whatever the house style says.>
```

Show the finished file, then hand it to the user to commit.

## Requirements

- An **Aikido** MCP server, signed in, with MCP feed access enabled for the workspace.
- An **Atlassian** MCP server with Jira write scope.

Tool names above are given unprefixed; the actual names may carry an MCP namespace
(`aikido-mcp:aikido_issues_list`, `mcp__…__createJiraIssue`). Match on the tool name.
