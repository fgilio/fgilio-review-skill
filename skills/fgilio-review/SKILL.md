---
name: fgilio-review
description: >
  Audit code through Franco's two lenses at once: the `coding` skill's style
  standards and the `polish` skill's simplification panel (Otwell, DHH, Wathan,
  Porzio), merged into one prioritized list. Three modes — report (default),
  address (apply the fixes), comment (post inline PR comments).
  Use when: reviewing the current changes, a branch, or a PR for both style
  compliance and simplification in one pass.
argument-hint: "[report|address|comment] [--scope=the current changes|the current branch|the entire project] [<PR number or URL>]"
user-invocable: true
disable-model-invocation: true
---

# fgilio-review

One audit, two lenses. Run Franco's `coding` standards and the `polish` reviewer panel over the same scope, merge their findings into a single prioritized list, then act on it according to the chosen mode.

## Parsing arguments

`$ARGUMENTS` carries the invocation. Resolve three things:

1. **Mode** — the first bare word. `report` (default) presents findings and changes nothing; `address` implements the fixes; `comment` posts findings as inline comments on a pull request. Accept synonyms: `fix` → address, `pr` / `comment-pr` → comment.
2. **Scope** — the value of `--scope=…` if present. Otherwise infer the most useful scope from context: the current uncommitted changes if the repo has them, else the current branch's diff against its base, else the entire project. Ask only if genuinely ambiguous.
3. **Target PR** (comment mode) — a PR number or URL if given; otherwise the PR for the current branch (`gh pr view --json number,url`).

State the resolved mode and scope in one line before you begin, so the choice is visible and correctable.

## The audit (all modes)

Run both lenses over the resolved scope and gather concrete, file-specific findings:

- **Style lens** — invoke the `coding` skill and apply its standards (expressive naming, fluent interfaces, guard clauses, Laravel idioms, comment and doc precision) to the scope.
- **Simplification lens** — invoke the `polish` skill in report mode over the same scope to convene its reviewer panel (Taylor Otwell, DHH, Adam Wathan, Caleb Porzio).

Merge both into one deduplicated, prioritized list. Each finding names the file or symbol, the problem (a style violation or a cost to clarity), and the concrete fix. Lead with the highest-leverage items. Where the two lenses disagree, say so and which way to lean.

## report (default)

Present the merged list. Write no code. End with the prioritized findings.

## address

Apply the fixes, but gate on risk. Auto-apply low-risk mechanical changes (renames, guard clauses, comment and doc fixes, local simplifications). For structural or ambiguous changes, confirm via **AskUserQuestion** before applying; skip anything declined.

If the session is non-interactive — a Claude Code routine with no user to prompt, so AskUserQuestion isn't viable — do not block and do not silently apply risky changes. Apply the safe fixes and leave each risky finding as a comment on the current branch's PR (inline where possible; see `comment`), so nothing is lost.

## comment

Post the merged findings as review comments on the target PR:

1. Pull the diff with `gh pr diff <pr>` to anchor findings to file and line.
2. Post one inline comment per finding via `mcp__github_inline_comment__create_inline_comment` (`confirmed: true`). Include a committable suggestion block only when it fixes the finding entirely; otherwise describe the fix in prose.
3. **Fallbacks** if that tool is unavailable: post inline via `gh api repos/{owner}/{repo}/pulls/{number}/comments` (path, line, body); if inline posting fails, consolidate everything into one `gh pr comment`.
4. One comment per unique finding — never duplicate. If there are no findings, post a single brief "no issues found" summary.
