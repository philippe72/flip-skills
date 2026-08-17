---
name: document-ticket
description: Generate a readable, self-contained document describing the outcome of a completed GitHub ticket. Produces a scannable overview, phase-based explanation, diagrams where they earn their place, a code-excerpts appendix, and links the doc back to the originating ticket.
disable-model-invocation: true
argument-hint: <ticket-number> [--since <ref>] [--no-code]
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(git:*), Bash(gh:*)
---

# Ticket Documentation

Generates documentation for a completed GitHub ticket, readable in one relaxed pass rather than exhaustively complete.

## Invocation

A ticket number is required. If none was supplied, stop and say so — never infer one from the branch name, the most recent commit, or the conversation.

- `--since <ref>` — take the change set from `<ref>..HEAD` instead of resolving it.
- `--no-code` — omit the code-excerpts appendix, which is otherwise included.

## Gathering the source material

### Resolving the change set

First confirm the number is a ticket and not a PR: `gh issue view <n> --json url` returns a `/pull/` path for a PR and an `/issues/` path for a ticket.

The document's subject is the code changes. Work down this ladder and stop at the first rung that isolates them:

1. **Explicit range** — if `--since <ref>` was supplied, it wins outright: diff `<ref>..HEAD`.
2. **Commits referencing the ticket** — `git log -E --grep '#<n>([^0-9]|$)'`, then take the union of their diffs. Keep the boundary group; without it `#1406` also matches `#14068`.
   - If the clone is shallow, `git fetch --unshallow` first.
   - If the search finds nothing, move to rung 3.
3. **The closing pull request** — `gh issue view <n> --json closedByPullRequestsReferences`, then `gh api repos/{owner}/{repo}/pulls/<pr>/commits` for exact SHAs.
   - **Filter the array by repository** — it includes fork and cross-repo PRs.
   - Use the PR *only* if it closes this ticket alone.
   - **Check each SHA is reachable before diffing it**: `git merge-base --is-ancestor <sha> HEAD`. After a squash merge they are not — grep the default branch for the **PR** number (`git log -E --grep '#<pr>([^0-9]|$)'`) and use the squash commit instead.
4. **Stop.** Report which rungs were tried and ask for `--since`. Do not fall back to a merge-base, "the last few commits," or the whole branch.

### Reading the ticket for context

Read the ticket body and comments in one call: `gh issue view <n> --json number,title,body,comments,url,state,labels`.

Follow linked tickets **one hop only**, and only structural links:

- Parent and sub-tickets: `gh issue view <n> --json parent,subIssues,subIssuesSummary`. `subIssues` is `{nodes: [...], totalCount: n}`, not a bare array. Never add `projectItems` to the same call — it needs `read:project` and fails the whole call.
- Cross-references written in the ticket **body**, not ones appearing only in comments.

Do not use the timeline API's `cross-referenced` events — they record mentions, not related work.

Linked tickets inform the overview and the *why*; **phases come only from the diff**.

## Output location

Write to `docs/tickets/<ticket-number>-<slug>/README.md`, where `<slug>` is roughly the first five meaningful words of the ticket title — lowercased, hyphen-separated, articles and filler words dropped. Ticket 412 "Fix rerouting when a toll road is excluded" becomes `docs/tickets/412-fix-rerouting-when-toll-road/`.

Create an `images/` folder inside that directory only if an image is actually produced.

If `docs/tickets/` does not exist but the repo has an obvious documentation root of its own, stop and ask once which to use rather than guessing.

**Never blind-overwrite.** If the target document already exists, read it, report that it exists, and overwrite only if the user confirms.

## Document skeleton

This spec is the sole authority on the document's shape. Do not look at existing documentation in the repo for templates, conventions, or prior art — tickets and their links supply context for the *content*, never the format.

The skeleton below defines structure and length. Treat its numbers as aims, not hard limits; the whole document should be readable in about five minutes.

````markdown
# <Ticket title>

[Ticket #<n>](<ticket-url>)

*Derived from <the resolved change set — a ref range, commit list, or PR>*

## Overview

<One paragraph of flowing prose — not labelled bullets — of at most 5 sentences: what was observed or motivated the work, what changed at a high level, and the measurable or practical outcome.>

## <Phase name — a coherent unit of change, not a chronological step>

<1–2 paragraphs. Diagrams where the diagram test says so; a worked example by default, unless a skip case applies.>

## <Phase name>

<1–2 paragraphs. Aim for 3–7 phases total.>

---

## Appendix: Code excerpts

### <Phase name — mirrors a phase heading above>

`path/to/file.ext:12-34`

```<lang>
<excerpt>
```

<1–2 sentences on what this code does.>
````

**Never cap coverage silently.** If the change set is too large to cover within these targets, cover what fits and state plainly in the document which areas were left out.

## Phases

Break the implementation into **logical phases** — conceptual chunks grouped by meaning and purpose, not a chronological "first I did X, then Y" narration, and not a file-by-file walkthrough. For an algorithm, this means its logical stages, not each line of pseudocode.

## Diagrams

Include a diagram only when it earns its place. **Include one** when the phase involves a **spatial, sequential, or structural relationship**: multiple components interacting, a chain of state changes, a before-and-after comparison, or a relationship between parts. **Skip it** when the phase is a single, linear cause-and-effect such as a value, threshold, or config change — even when that change fixes a bug.

**Choosing the representation**: prefer what a domain expert would sketch to explain the problem to a layperson — don't translate it into a generic flowchart because a flowchart is easy to draw.

- a sequence diagram for interactions between components
- a before-and-after comparison for structural changes or refactors
- an architecture diagram for how pieces fit together
- a domain-native view when the problem lives in a real coordinate space

**Choosing the format**:

- **Mermaid**, inline in the Markdown, when the relationship fits a standard graph vocabulary — sequence, flowchart, state, ER.
- **Hand-authored SVG** in `images/` when the meaning lives in spatial arrangement: side-by-side comparisons, annotated geometry, anything where position carries information.
- **Raster image** only when the truth is a real rendered frame that cannot be reconstructed. It is the only non-diffable format, so it needs a reason.

When a phase genuinely warrants a diagram but the honest format is a raster image that cannot be produced here, leave an explicit placeholder naming what should go there rather than substituting a Mermaid approximation of a rendered frame.

## Worked examples

A worked example is a concrete case shown as an image: specific inputs traced through the phase's change to a specific outcome — one route, one payload, definite numbers. Concreteness is the point, not authenticity: fully synthetic values are fine as long as they demonstrate the principle. One example per phase is the norm; add more only when the phase's behaviour genuinely forks — distinct branches, modes, or edge cases — and each extra example covers a branch the others don't. Variations of the same path don't qualify. When a phase has examples, they come last, after the prose and any diagram.

Whether a phase gets a worked example is independent of whether it gets a diagram. **Every phase gets one by default.** Skip it only in these three cases:

- the phase's diagram already depicts a concrete case — it *is* the worked example;
- the phase is already so concrete that an example would restate the prose with values filled in;
- the phase has no input-to-output behaviour to trace, such as a behaviour-preserving refactor, a rename, or a tooling or dependency change.

No other reason to skip exists.

Format and file rules are the same as for diagrams.

Prefer values from recorded material when it exists — repro steps in the ticket, test inputs and expected outputs in the diff, the PR description — obtained by reading, never by executing; running builds or tests is out of scope. When nothing recorded fits, invent a minimal case that shows the principle and caption it *illustrative*. The one hard rule is provenance honesty: never present invented values as recorded ones. Missing data is never a reason to omit an example — only the three skip cases are.

## Linking back to the ticket

After writing the document, link back from the ticket — but only once the link will actually resolve:

1. Build a permalink of the form `https://github.com/{owner}/{repo}/blob/{full-40-char-sha}/{path}`. Use the full SHA, not an abbreviation. Do not build it with `gh browse` unless passing a line range — without one it emits the `/tree/` form.
2. Verify the commit reached the remote by asking GitHub, not the local repo: `gh api repos/{owner}/{repo}/commits/<sha>`. A missing commit returns HTTP 422, not 404, so test the exit code. Do not use `git branch -r --contains` — it only reflects the last fetch.
3. If the commit resolves, post with `gh issue comment <n> --body-file -`. Always pass a body flag; with none, `gh` prompts and hangs. If the commit does not resolve, stop and print the exact command to run after pushing.

Do not commit or push on the user's behalf to make the link resolve.

## Workflow checklist

1. Require a ticket number; stop if absent. Confirm it is a ticket and not a PR.
2. Resolve the change set via the ladder; stop rather than guessing if no rung isolates it.
3. Read the ticket and its one-hop structural links for context.
4. Check the target path; if a document already exists, stop and confirm before overwriting.
5. Fill the skeleton: overview, phases, diagrams per the test, worked examples by default minus skip cases, appendix unless `--no-code`.
6. Note any omitted coverage in the document.
7. Post the link-back comment if the permalink resolves; otherwise print the command.
