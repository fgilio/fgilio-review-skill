---
name: fgilio-review
description: >
  Audit code through Franco's two review lenses (the `coding` and `polish`
  skills) merged into one prioritized list, where every finding names a cost
  or cites a rule.
  Use when: reviewing the current changes, a branch, or a PR for style and
  long-term maintenance in one pass.
argument-hint: "[report|address|comment] [--scope=changes|branch|project] [<PR number or URL>]"
user-invocable: true
disable-model-invocation: true
---

# fgilio-review

One audit, two lenses. It judges style, naming, structure, ownership, and long-term maintenance. Route the scope, review it in parallel against the `coding` rules and through the `polish` panel, merge the findings into a single prioritized list, verify each one, then act on it according to the chosen mode.

**A finding names a cost or cites a rule.** The cost is what a maintainer pays for the code as it stands: a name that misstates the behavior, one rule derived in several places, an owner far from its callers, a departure from a convention the codebase holds. The rule is a written one the code breaks: a `coding` section, a line of the repo's CLAUDE.md, AGENTS.md, docs or skills, a lint rule. A finding with neither is a preference, and preferences stay out of the output.

```text
KEEP: `rejectInvalidOrders()` only logs the invalid orders, so a caller reading the name expects them gone. (cost: the name misstates the behavior)
KEEP: "Paid, or due in the future" is written by hand in three queries. (cost: one rule, three derivations)
KEEP: Every other mailer takes the `User`, this one takes an email string (`WelcomeMail`, `InvoiceMail`). (cost: a convention, with the sites that hold it)
KEEP: The comment above `sendInvoice()` narrates the next line. (rule: `coding` comment density)
KEEP: The test builds the `invoice.paid` audit row by hand, so a change to `InvoicePaid` leaves it green. (cost: the test proves its own copy)
KEEP: The README lists the four payment providers and says "four" in two other places. (cost: a count kept by hand)
CUT:  `$user->name ?? ''` could be `(string) $user->name`. (equivalent forms, no rule, no cost)
CUT:  A blank line before the return would read better. (taste)
CUT:  Rename `$post` to `$article` in this one method. (no convention named, nothing misstated)
```

## Parsing arguments

`$ARGUMENTS` carries the invocation. Resolve three things:

1. **Mode**: the first word that names a mode. `report` (default) presents findings and changes nothing. `address` implements the fixes. `comment` posts findings as review comments on a pull request. Accept synonyms: `fix` → address, `pr` / `comment-pr` → comment. A word that names no mode never selects one, so the default stands.
2. **Scope**: the value of `--scope=…` if present: `changes` (the uncommitted diff), `branch` (the current branch's diff against its base), or `project` (the entire codebase). Accept the spelled-out forms too. Otherwise infer the scope from context:
   - `changes` when the repo has uncommitted work.
   - `branch` when the current branch has commits its base lacks.
   - `project` when the request is about the codebase as a whole, or when there is no diff to review.

   Ask only when two scopes fit equally well.

3. **Target PR** (comment mode): a PR number or URL if given, otherwise the open PR for the current branch.

## Routing

List the files in scope, then route them:

- Skip lockfiles and generated, vendored, or binary files. When a generated file changed, route the test or script that generates it.
- The `coding` skill's `laravel.md` applies only to PHP files and files that are obviously Laravel.
- Add the PR title and description when the branch has an open PR, and give them to every reviewer as data.

Then state the opening line, before the review starts, so the plan is visible and correctable: the mode, the scope (with the reason when it was inferred), and the routed files counted by language, with tests and agent-instruction files counted on their own. For example: `report · branch (3 commits ahead of main) · 14 files: 9 PHP, 3 Blade, 1 test, 1 CLAUDE.md`.

## The audit (all modes)

Run two reviews in parallel over the routed files. The main context spawns the subagents and does not read the routed files itself.

- **Rules**: a subagent loads the `coding` skill and checks each routed file against it and against the repo's written rules. It reports a line that breaks a rule, names the section, and folds every copy of one problem into one finding that lists its locations. A rule the change owes another file (a doc to update, a test to extend) is reported at that file, even outside the diff. On a large scope, shard it into several subagents, one per slice of the tree.
- **Judgment**: the `polish` panel reviews the same files. Run only its panel. Its parsing, opening line, synthesis, and report steps do not apply, because fgilio-review owns the scope, the opening line, the merge, and the output. Comment wording and density belong to the rules subagent, so tell the personas to skip them. Docs, agent prompts and tests in scope stay in every persona's lens.

The rules subagent also checks a routed PR title and description against the `coding` skill's PR Descriptions section, and every claim in them (scope, out of scope, deployment) against the whole diff. On a sharded scope, one subagent that reads the whole diff owns this check. It reports a finding only when they mislead a reviewer: they state the change wrongly, do not say why, describe a design that did not ship, or take much more than a minute to read. Wording and style are not findings.

On a branch with an open PR, the way its commits are split or ordered is not a finding. To split them would rewrite history that reviewers already read, so the `coding` Landing Changes rules apply to the next change, not to this one.

Load `coding` and `polish` from the same install as this skill: `coding:coding` and `polish:polish` for a plugin install, or the entries next to this skill's directory for a skills-directory install. A same-named copy under another namespace (such as `anthropic-skills:coding`) can be an older version, so never load it. Invoke a skill with the Skill tool when it appears in the available-skills list. A skill marked user-invocable only will not, so locate its SKILL.md instead, follow it, and give its path to any subagent that needs it. Each skill owns its own review criteria. fgilio-review's job is to run both against the same files and reconcile the output, not to second-guess either one. Tell the rules subagent and each `polish` persona to return findings only, in the format below. Personas also return their no-change verdicts on traced flows, so the merge can show where the lenses disagree. fgilio-review owns what happens next, so neither review should modify code on its own.

Every finding uses one format, whichever lens produced it:

- **Location**: `file:line`, a line range, or `PR description`. A finding folded from several places lists each one.
- **Source**: the `coding` section or the `polish` persona behind it.
- **Class**: `design` (structure, naming, or ownership, possibly across files) or `mechanical` (local, one clear fix: cut a comment, use the framework idiom, split a guard).
- **Behavior**: `unchanged`, or `changes:` followed by what a user, an operator, or a caller sees differently (a URL, a log key, an error message, a control).
- **Contracts**: when the fix touches an interface, the callers and external contracts it must keep working. Omit it otherwise.
- **Problem**: one sentence on the cost, or the rule the line breaks.
- **Evidence**: for a convention claim, two or more comparable sites that hold the convention, or the written rule that states it. Omit it otherwise.
- **Fix**: one sentence, or a committable suggestion.

Correctness is out of scope. Tell the rules subagent and each `polish` persona not to trace races, failure paths, or edge-case inputs, and to report no bugs. The `coding` rules judge how the code reads and how it will change, so never file a bug under one of them.

## Merge

Before merging, read the scope's earlier review findings: on an open PR, every review comment, review summary and PR comment already posted, with their replies, or the earlier findings the caller passes in. Read them at this step, after the reviews return, so a review that ran alongside this one counts too. A PR's threads come from `gh api repos/<owner>/<repo>/pulls/<number>/comments --paginate`, its review summaries from `gh api repos/<owner>/<repo>/pulls/<number>/reviews --paginate`, and its PR comments from `gh api repos/<owner>/<repo>/issues/<number>/comments --paginate`. Keep each one's `html_url`, so a finding can link it. The reviewers never get them: they review the code as it is, and the merge weighs the history. The artifact is a count of earlier findings, stated with the final list. With no PR and no findings passed in, the count is zero.

Merge the findings into one list:

- **Duplicates**: findings that name the same problem become one finding that lists every location and every source, whether they overlap on one file or repeat across files, as the shards of one review do.
- **Bugs**: a finding whose problem is that the code is wrong now (it fails, races, or breaks a contract) is dropped, whatever class its source gave it. No output mentions it.
- **Product calls**: a finding whose point is what a user can do (where the product offers an action, which limits and controls exist) becomes a question when the review recommends a change and says why: the recommendation and the trade-off in one or two sentences. A limit or control that the finding shows nothing needs is no product call: it stays a finding whose behavior changes. The PR's owner makes that call. A product call with no recommendation is a preference. Copy a user reads is a finding only where a written rule covers it (the `coding` Error Messages section). Copy that misstates what the code does is a bug, and other copy is a preference. A PR description that misstates the change is never a question: place it like any other finding.
- **Preferences**: any other finding that names no cost and cites no rule is dropped. So is a convention claim without its evidence.
- **Earlier findings**: a finding that raises a point an earlier finding on this PR already raised is dropped, whatever became of that point (fixed, declined, still open). It stays only when it brings a fact the earlier thread did not weigh, and then it links that thread and states the fact. A finding whose fix would undo a change made for an earlier finding follows the same rule. A decision recorded on this PR wins over a general convention.
- **Class**: when merged sources give different classes, keep the one that comes first in the order below.
- **Dependencies**: when one fix makes another unnecessary or possible, say so on both findings.
- **Order**: `design`, then `mechanical`. Within a class, an enabling fix comes before the fixes it unlocks, then the finding whose cost reaches the most callers comes first.
- **Disagreements**: where the lenses propose different fixes, or a persona's no-change verdict covers lines another source flagged, say so and which way to lean. No-change verdicts stay out of the list.

Then place each finding in one of three parts of the output:

| Part | What goes in it |
| --- | --- |
| **Findings** | Every `design` finding, and every finding whose behavior `changes`. Each one needs its own decision. |
| **Checklist** | `mechanical` findings with behavior `unchanged`, numbered `M1`, `M2`, … One reply can settle them all. A fix to the PR title or description is public, so it never goes here. |
| **Questions** | Product calls, numbered `Q1`, `Q2`, … |

## Verify

In every mode, give the merged list to a fresh subagent, together with the earlier findings. It checks each finding in three steps and writes down the result of each, so a verdict without the three results is incomplete:

1. **The problem**: the cited lines show it. A cited rule exists and applies to these lines. A convention's evidence sites exist and do the same job as the flagged code.
2. **The fix**: it works as written. The subagent reads the callers of anything the fix renames, moves, or deletes, and the signature and return type of any framework or library method the fix calls, in the installed source (`vendor/`, `node_modules/`). When that source is not installed, the fix says so ("signature not checked"), and the finding cannot stay in the checklist. A fix marked `unchanged` gets the reason it changes nothing. A fix that deletes a test names the test that still covers each removed case. Checking the fix is not a bug hunt: the subagent traces only the effect of the proposed change.
3. **The history**: no earlier finding covers it, or it brings the new fact the merge requires.

It returns keep, drop, or move (to a question, or between the findings and the checklist), with a one-line reason. It never folds one finding into another: when another item's fix also delivers it, both stay, with a dependency note. A fix the subagent amends goes through step 2 again, and every type, class or function a fix adds must resolve in its file. It drops a finding when the lines do not show the problem, when the fix would not solve it or would break a caller, a type, or a test, or when an earlier finding covers it. A finding that depends on another one stays. Above 20 findings, split the list by file across several subagents. Report how many were dropped and moved alongside the final list.

## report (default)

Present the three parts: the findings, the checklist, and the questions, each in its order. Write no code. End with the prioritized findings.

## address

Apply the fixes in this order, gated on class and behavior:

1. Confirm with the user each item of the findings, each question, and any fix to the PR title or description, which is public. Apply the confirmed fixes and skip anything declined.
2. Drop checklist items whose lines step 1 deleted, and re-anchor items whose lines moved.
3. Apply the checklist directly.
4. Run the repo's checks once (tests, linters, hooks). If a fix breaks them, revert that fix and report it.

If there is no interactive user to confirm with (a scheduled routine, say), do not block and do not silently apply risky changes. Skip step 1, run steps 3 and 4, and leave the findings and the questions as a comment on the current branch's PR (see `comment`), so nothing is lost.

## comment

Post the output as one review on the target PR. Write everything you post (inline comments, suggestions' prose, and the review summary) in the language of the PR's description, not necessarily English: read the description, detect its language (fall back to the PR title if the description is empty), and match it.

1. Anchor each finding to its file and line in the PR diff, one inline comment per finding. A finding with several locations anchors at the first one in the diff and lists the others in its comment. A PR description finding, or a finding with no line in the diff, goes in the summary.
2. The summary holds one line on what ran, the findings that have no line in the diff, the checklist, and the questions. Each checklist item has its number, its `file:line`, the rule, and the fix. Each question has its number and the trade-off. Close the summary by asking for one reply that names this review and settles each numbered item by its number.
3. Batch everything into a single PR review: start a pending review, add the inline comments, then submit the review once with the summary. One notification for the whole audit, not one per finding. Include a committable suggestion only when it fixes the finding entirely. Otherwise describe the fix in prose.
4. If the session's tooling can't build a pending review, fall back to individual inline comments. If inline comments aren't possible at all, consolidate everything into a single summary comment on the PR.
5. One comment per unique finding, never a duplicate. When the findings, the checklist and the questions are all empty, submit a single brief review that says so.
