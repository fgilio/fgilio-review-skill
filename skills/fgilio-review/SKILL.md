---
name: fgilio-review
description: >
  Audit code through Franco's two review lenses (the `coding` and `polish`
  skills) merged into one prioritized list.
  Use when: reviewing the current changes, a branch, or a PR in one pass.
argument-hint: "[report|address|comment] [--scope=changes|branch|project] [<PR number or URL>]"
user-invocable: true
disable-model-invocation: true
---

# fgilio-review

One audit, two lenses. Route the scope, review it in parallel against the `coding` rules and through the `polish` panel, verify each finding, merge the survivors into a single prioritized list, then act on it according to the chosen mode.

## Parsing arguments

`$ARGUMENTS` carries the invocation. Resolve three things:

1. **Mode**: the first word that names a mode. `report` (default) presents findings and changes nothing. `address` implements the fixes. `comment` posts findings as review comments on a pull request. Accept synonyms: `fix` → address, `pr` / `comment-pr` → comment. A word that names no mode never selects one, so the default stands.
2. **Scope**: the value of `--scope=…` if present: `changes` (the uncommitted diff), `branch` (the current branch's diff against its base), or `project` (the entire codebase). Accept the spelled-out forms too. Otherwise infer the scope from context:
   - `changes` when the repo has uncommitted work.
   - `branch` when the current branch has commits its base lacks.
   - `project` when the request is about the codebase as a whole, or when there is no diff to review.

   Ask only when two scopes fit equally well.

3. **Target PR** (comment mode): a PR number or URL if given, otherwise the open PR for the current branch.

State the resolved mode and scope in one line before you begin, with the reason for an inferred scope, so the choice is visible and correctable.

## Routing

List the files in scope, then route them:

- Skip lockfiles and generated, vendored, or binary files.
- The `coding` skill's `laravel.md` applies only to PHP files and files that are obviously Laravel.

Add the count of routed files by type to the opening line, for example: `report · branch (3 commits ahead of main) · 14 files: 9 PHP, 3 Blade, 1 test, 1 CLAUDE.md`.

## The audit (all modes)

Run two reviews in parallel over the routed files. Do not review the files in the main context.

- **Rules**: a subagent loads the `coding` skill and checks each routed file against it as a checklist, one finding per violation. On a large scope, shard it into several subagents, one per slice of the tree.
- **Judgment**: the `polish` skill runs its panel over the same files. Its comment findings can overlap with the checklist. The merge removes the duplicates.

Invoke a skill with the Skill tool when it appears in the available-skills list. A skill marked user-invocable only will not, so locate its SKILL.md instead (installed as a plugin or symlinked into the skills directory), follow it, and give its path to any subagent that needs it. Each skill owns its own review criteria. fgilio-review's job is to run both against the same files and reconcile the output, not to second-guess either one. Tell the rules subagent and each `polish` persona to return findings only, in the format below. fgilio-review owns what happens next, so neither review should modify code on its own.

Every finding uses one format, whichever lens produced it:

- **Location**: `file:line` or a line range.
- **Source**: the `coding` section or the `polish` persona behind it.
- **Class**: `defect` (wrong now: a bug or a broken contract), `design` (structure, naming, or ownership, possibly across files), or `mechanical` (local, same behavior, one clear fix: cut a comment, use the framework idiom, split a guard).
- **Behavior**: `unchanged` or `changes`.
- **Problem**: one sentence on what breaks or costs clarity.
- **Fix**: one sentence, or a committable suggestion.

Bug hunting belongs to `/code-review`. A defect either lens finds along the way still goes in the list.

## Verify

In every mode, give the findings to a fresh subagent before the merge. For each finding, it reads the cited lines and the cited rule, then returns keep or drop with a one-line reason. Split a long list across several subagents. Only kept findings reach the merge. Report how many were dropped alongside the merged list.

## Merge

Merge the findings into one list:

- **Duplicates**: findings on the same file, with overlapping lines and the same problem, become one finding that lists both sources.
- **Order**: `defect`, then `design`, then `mechanical`. Within a class, the fix that removes the most code or concepts comes first.
- **Disagreements**: where the two lenses disagree, say so and which way to lean.

## report (default)

Present the merged list. Write no code. End with the prioritized findings.

## address

Apply the fixes, gated on class and behavior. Apply `mechanical` findings with behavior `unchanged` directly. Confirm every other finding with the user before applying it, and skip anything declined.

If there is no interactive user to confirm with (a scheduled routine, say), do not block and do not silently apply risky changes. Apply the mechanical fixes and leave each other finding as a comment on the current branch's PR (see `comment`), so nothing is lost.

## comment

Post the merged findings as review comments on the target PR. Write everything you post (inline comments, suggestions' prose, and the review summary) in the language of the PR's description, not necessarily English: read the description, detect its language (fall back to the PR title if the description is empty), and match it.

1. Anchor each finding to its file and line in the PR diff.
2. Batch everything into a single PR review: start a pending review, add one inline comment per finding at the relevant line, then submit the review once with a brief summary. One notification for the whole audit, not one per finding. Include a committable suggestion only when it fixes the finding entirely. Otherwise describe the fix in prose.
3. If the session's tooling can't build a pending review, fall back to individual inline comments. If inline comments aren't possible at all, consolidate everything into a single summary comment on the PR.
4. One comment per unique finding, never a duplicate. If there are no findings, submit a single brief "no issues found" review.
