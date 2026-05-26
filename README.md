# frontend-skills

Shared AI-tool skills for the STRV frontend team. Works with Claude Code, Codex CLI, and Cursor — distributed as plain `SKILL.md` files via the open [Agent Skills spec](https://agentskills.io/specification).

## Install

**Common case** — Claude Code, project-local. You will be prompted to pick which skills to install:

```bash
npx skills add strvcom/frontend-skills -a claude-code
```

**Other variants:**

```bash
# Fully interactive — pick the agent too
npx skills add strvcom/frontend-skills

# All skills, no prompts (re-installs / CI)
npx skills add strvcom/frontend-skills -a claude-code -s '*' -y

# One specific skill, globally (across all your projects)
npx skills add strvcom/frontend-skills -a claude-code -s grill-me -g

# Everything everywhere (all skills, all agents, no prompts)
npx skills add strvcom/frontend-skills --all
```

Useful flags: `-a` (agent — `claude-code`, `codex`, `cursor`, or `*`; repeatable), `-s` (skill name; repeatable; `*` for all), `-g` (global / user-level), `-y` (skip prompts), `--copy` (copy instead of symlink). Run `npx skills --help` for the full CLI.

Files land in `.agents/skills/<name>/` with a symlink at `.claude/skills/<name>/` for Claude Code.

**Alternative install path:** you can also ask Claude Code in chat to install a skill — same files end up in `.claude/skills/`, but you lose the `npx skills update` story.

## Updates

```bash
npx skills update                  # refresh all installed skills
npx skills update <skill-name>     # refresh one
```

Skills track the latest commit on `main` — there's no version pinning yet. Breaking changes get announced in the team Slack channel and tagged as a [GitHub Release](https://github.com/strvcom/frontend-skills/releases). See [CONTRIBUTING.md](CONTRIBUTING.md) for the full update protocol.

## Available skills

| Skill | What it does | Owner |
|---|---|---|
| [`frontend-design`](skills/frontend-design/SKILL.md) | Distinctive, production-grade frontend UIs that avoid generic AI aesthetics. Vendored from [anthropics/skills](https://github.com/anthropics/skills). | @haidave |
| [`fixing-accessibility`](skills/fixing-accessibility/SKILL.md) | Fix a11y issues (ARIA, keyboard, focus, contrast, forms) with priority-ranked rules. Plays well with shadcn/Radix/Base UI primitives. Vendored from [ibelick/ui-skills](https://github.com/ibelick/ui-skills). | @haidave |
| [`grill-me`](skills/grill-me/SKILL.md) | Interview-style planning skill. Grills you on a plan or design, walking down each branch of the decision tree one question at a time until a shared understanding lands. Vendored from [mattpocock/skills](https://github.com/mattpocock/skills). | @haidave |
| [`compress-video`](skills/compress-video/SKILL.md) | Compress and re-encode a video file to a smaller MP4 (H.264 + AAC) via ffmpeg. Use when sharing videos through size-constrained channels (Signal, Slack, email). | @sebous |
| [`nutshell`](skills/nutshell/SKILL.md) | Distill anything to its essence — TLDR a concept, name a thing, condense a procedure. | @ZaweSK |
| [`the-fool`](skills/the-fool/SKILL.md) | Devil's advocate that stress-tests plans, decisions, and proposals via Socratic questioning, dialectic synthesis, pre-mortem, red team, or evidence audit. Vendored from [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills). | @gneutzling |
| [`logical-commits`](skills/logical-commits/SKILL.md) | Split working-tree changes into clean Conventional Commits. Auto-detects repo conventions, proposes a grouping, and gates every type/scope/approval through interactive selections. Never bypasses git hooks. | @gneutzling |
| [`supply-chain-check`](skills/supply-chain-check/SKILL.md) | Audit JS supply-chain hygiene — Safe Chain on the dev machine, `.npmrc`/equivalent in the repo, GitHub Actions CI gates, CONTRIBUTING.md mention — and offer interactive fixes one by one. | @monkey-denky |

More skills land as domain owners are recruited. See [CODEOWNERS](CODEOWNERS).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add a skill (vendor or author), commit conventions, breaking-change protocol, and review flow.

## License

Apache 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
