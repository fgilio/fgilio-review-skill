---
name: fgilio-review
description: >
  Audit code through Franco's two lenses at once — the `coding` skill and the
  `polish` skill — run over the same scope and merged into one prioritized list.
  Three modes — report (default), address (apply the fixes), comment (post review
  comments on a PR).
  Use when: reviewing the current changes, a branch, or a PR in one pass.
argument-hint: "[report|address|comment] [--scope=the current changes|the current branch|the entire project] [<PR number or URL>]"
user-invocable: true
disable-model-invocation: true
---

# fgilio-review

One audit, two lenses. Run the `coding` skill and the `polish` skill over the same scope, merge their findings into a single prioritized list, then act on it according to the chosen mode.

## Parsing arguments

`$ARGUMENTS` carries the invocation. Resolve three things:

1. **Mode** — the first bare word. `report` (default) presents findings and changes nothing; `address` implements the fixes; `comment` posts findings as review comments on a pull request. Accept synonyms: `fix` → address, `pr` / `comment-pr` → comment.
2. **Scope** — the value of `--scope=…` if present. Otherwise infer the most useful scope from context: the current uncommitted changes if the repo has them, else the current branch's diff against its base, else the entire project. Ask only if genuinely ambiguous.
3. **Target PR** (comment mode) — a PR number or URL if given; otherwise the open PR for the current branch.

State the resolved mode and scope in one line before you begin, so the choice is visible and correctable.

## The audit (all modes)

Invoke both skills over the resolved scope and collect their findings. Each skill owns its own review criteria — fgilio-review's job is to run both against the same scope and reconcile the output, not to second-guess either one. Have them return findings only; fgilio-review owns what happens next, so neither sub-review should modify code on its own.

Merge the two sets into one deduplicated, prioritized list. Each finding names the file or symbol, the problem, and the concrete fix. Lead with the highest-leverage items. Where the two lenses disagree, say so and which way to lean.

## report (default)

Present the merged list. Write no code. End with the prioritized findings.

## address

Apply the fixes, but gate on risk. Apply low-risk, local, mechanical changes directly. For structural or ambiguous changes, confirm with the user before applying; skip anything declined.

If there is no interactive user to confirm with — a scheduled routine, say — do not block and do not silently apply risky changes. Apply the safe fixes and leave each risky finding as a comment on the current branch's PR (see `comment`), so nothing is lost.

## comment

Post the merged findings as review comments on the target PR:

1. Anchor each finding to its file and line in the PR diff.
2. Post one comment per finding, inline at the relevant line where possible. Include a committable suggestion only when it fixes the finding entirely; otherwise describe the fix in prose.
3. If inline comments aren't possible, consolidate everything into a single summary comment on the PR.
4. One comment per unique finding — never duplicate. If there are no findings, post a single brief "no issues found" summary.
