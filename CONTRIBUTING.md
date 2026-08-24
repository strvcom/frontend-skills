# Contributing

Toolkit for the STRV frontend team. Skills are plain `SKILL.md` files distributed via [`npx skills`](https://skills.sh/docs). Conventions below keep it lean and survivable.

## Add a skill

Two paths: vendor an existing skill, or author your own.

### Path A — vendor an existing skill

Most skills already exist somewhere. If a battle-tested one fits a STRV need, vendor it rather than rewriting.

```
skills/<name>/
  SKILL.md       # the skill itself
  LICENSE.txt    # original license, copied verbatim
```

Then update [NOTICE](NOTICE) with a block like:

```
* skills/<name>/
  Copyright (c) <year> <author>.
  Licensed under <license>.
  Source: <upstream-url>
  Original license retained at skills/<name>/LICENSE.txt.
```

Acceptable upstream licenses: **MIT, Apache 2.0, BSD, ISC**. Avoid GPL/copyleft (incompatible with our Apache 2.0 distribution).

### Path B — author a new skill

```
skills/<name>/
  SKILL.md
```

`SKILL.md` must have YAML frontmatter per the [Agent Skills spec](https://agentskills.io/specification):

```markdown
---
name: <name>
description: <one-line trigger description — Claude reads this to decide when to load the skill>
---

# <Title>

<body — guidance the agent follows when the skill is active>
```

The `description` field is the **most important thing in the file**. Claude Code auto-loads skill descriptions into context (~8k chars total) and uses them to decide when a skill activates. Lead with concrete triggers ("Use when adding interactive controls, forms, dialogs..."). Vague descriptions = dead skills.

Keep skills **bounded**: one concern per skill. If your skill is doing UI design AND a11y AND testing, split it.

## Naming

`skills/<name>/` — kebab-case, short, descriptive. Avoid `strv-` or `frontend-` prefixes (already implied by repo name).

Skills that only make sense as a set may live one level deeper under a group directory — `skills/docs/adr-authoring/`. The CLI discovers `SKILL.md` at any depth, and the group gets a single CODEOWNERS entry. Group only when the set is genuinely coupled; a lone skill stays at the top level.

## Commits

[Conventional Commits](https://www.conventionalcommits.org/). The relevant prefixes:

- `feat: add X skill` — new skill or new capability
- `fix: tighten Y skill description` — small fix, no breaking change
- `feat!: rewrite Z skill scope` — **breaking change** (rare; affects how/when skill triggers)
- `docs:`, `chore:` — non-skill changes

Why: lets us auto-generate changelogs from `git log` and lets reviewers spot breaking changes at a glance.

## Reviews

Every PR touching `skills/<dir>/` is reviewed by that dir's CODEOWNER. See [CODEOWNERS](CODEOWNERS).

If you're adding a new skill domain, add yourself to CODEOWNERS in the same PR.

## Breaking changes

A "breaking change" = the skill now triggers on different prompts, gives different recommendations, or has different scope than before. These propagate instantly to anyone who runs `npx skills update`.

When you ship one:

1. Use `feat!:` in the commit message
2. Cut a [GitHub Release](https://github.com/strvcom/frontend-skills/releases) with notes
3. Announce in the team Slack channel — give people a heads-up before they update

There is no semver enforcement at the CLI level. Discipline + communication is how this works for now.

## How updates reach the team

```bash
npx skills update                         # all installed skills, latest from main
npx skills update <skill-name>            # one skill
```

Updates are **opt-in**. Don't expect people to run `update` daily — that's why announcements matter.

## What we deliberately don't have (yet)

- **Lockfiles / version pinning** — `npx skills` has experimental `skills-lock.json` support; we'll adopt when stable
- **Custom validation CI** — we trust the spec's frontmatter check; rolling our own is maintenance burden
- **Issue templates** — add when patterns emerge
- **CHANGELOG.md** — GitHub Releases is the single source of truth

These are conscious omissions, not oversights. Re-evaluate as the toolkit grows.

