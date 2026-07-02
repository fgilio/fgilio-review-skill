# fgilio-review skill

A code-review skill for Claude Code that audits a scope through **both** of Franco's other skills at once: the [`coding`](https://github.com/fgilio/coding-skill) skill and the [`polish`](https://github.com/fgilio/polish-skill) skill. It runs both lenses over the same scope and synthesizes their output into one deduplicated, prioritized list.

Everything lives in [SKILL.md](skills/fgilio-review/SKILL.md).

## Modes

One skill, three modes — selected by the first argument:

- **report (default)** — `/fgilio-review` presents the merged findings. Writes no code.
- **address** — `/fgilio-review address` applies the fixes: low-risk changes directly, risky ones only after confirming with you. In a non-interactive routine it posts the risky ones as PR comments instead of blocking.
- **comment** — `/fgilio-review comment` posts the findings as review comments on the pull request, inline at the relevant lines where possible.

Optionally scope the review:

```
/fgilio-review --scope=changes      # default
/fgilio-review address --scope=branch
/fgilio-review comment 65           # target a specific PR
```

## Requirements

The `coding` and `polish` skills must be available in the session — this skill orchestrates them. Both ship from the same [fgilio marketplace](https://github.com/fgilio/claude-plugins). `comment` mode needs whatever GitHub tooling the session provides for posting PR review comments.

## Installation

### As a plugin

Installs through the [fgilio marketplace](https://github.com/fgilio/claude-plugins) and receives updates as the skill evolves:

```
/plugin marketplace add fgilio/claude-plugins
/plugin install fgilio-review@fgilio
```

### Manual clone

Clone and symlink into your Claude Code skills directory:

```bash
git clone https://github.com/fgilio/fgilio-review-skill.git ~/dev/skills/fgilio-review-skill
ln -s ~/dev/skills/fgilio-review-skill/skills/fgilio-review ~/.claude/skills/fgilio-review
```

This path also works with any agent that supports the open [Agent Skills](https://agentskills.io) format. Updates require a manual `git pull`.

## License

[MIT](LICENSE)
