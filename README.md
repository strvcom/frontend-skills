# frontend-skills

Shared AI-tool skills for the STRV frontend team. Works with Claude Code, Codex CLI, and Cursor — distributed as plain `SKILL.md` files via the open [Agent Skills spec](https://agentskills.io/specification).

## Install

Pick the agent(s) you use:

```bash
# All three tools, project-local
npx skills add strvcom/frontend-skills -a claude-code -a codex -a cursor

# Single tool, single skill, global (across all your projects)
npx skills add strvcom/frontend-skills -a claude-code --skill frontend-design -g
```

`npx skills` writes to `.agents/skills/<name>/` (the universal location for Codex / Cursor / others) and symlinks `.claude/skills/<name>/` for Claude Code. Default is symlink; pass `--copy` for files.

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

More skills land as domain owners are recruited. See [CODEOWNERS](CODEOWNERS).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add a skill (vendor or author), commit conventions, breaking-change protocol, and review flow.

## License

Apache 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
