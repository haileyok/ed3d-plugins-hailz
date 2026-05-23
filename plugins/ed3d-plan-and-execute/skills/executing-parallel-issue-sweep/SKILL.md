---
name: executing-parallel-issue-sweep
description: Use when the user asks to address multiple independent issues (bugs OR feature tickets) in parallel - dispatches one self-contained owner subagent per issue, where each owner runs its own implement/review/fix/PR loop independently, and the orchestrator only synthesizes final reports. Not for single issues or for issues that should land as one PR.
user-invocable: false
---

# Executing a Parallel Issue Sweep

Run a coordinated sweep of several independent issues in one session.
Each issue gets its own worktree, branch, and **end-to-end owner subagent**.
The owner does the work, reviews its own work (by dispatching code-reviewer),
fixes, re-reviews, pushes, and opens a draft PR — all without waiting on
the orchestrator or on other issues.

**Core principle:** Nested fanout. The orchestrator spawns issue-owners.
Each issue-owner spawns its own reviewer + bug-fixer. Issues iterate
independently — issue A's third review cycle doesn't wait for issue B
to start its first.

**Why this shape:** When the orchestrator owns the review loop, every
issue has to wait for the slowest one in the wave before any can move
to fix-and-retry. With nested fanout, A's review/fix/PR runs the
moment A's implementation lands, regardless of B's state.

**Required reading before applying this skill:**
- `using-git-worktrees` — worktree setup conventions
- `requesting-code-review` — the review-fix-rereview loop (issue-owners run this)
- `verification-before-completion` — the gate before claiming done

## When to use

- User points at a backlog / project board / issue list and asks to
  "fix these" or "ship these" together.
- Items are mostly independent (different files / surfaces / subsystems).
- Each item is small-to-medium scope.

## When NOT to use

- Single issue. Use `executing-an-implementation-plan` or just do the work.
- Items that should land as ONE PR (a coherent refactor sweep). That's
  one feature, not a sweep.
- Items with hard dependencies between them (A blocks B blocks C). Use
  sequential execution with a real implementation plan.
- Items where the right answer is "design first." Don't fanout to
  implementation when the design isn't pinned down.

## The process

### 1. Pre-flight — DO NOT FANOUT YET

The orchestrator owns this phase. It is where most of the value is and
where most failures happen.

#### 1a. Verify the input set

- If the user pointed at a label/tag (e.g. "all the bugs"), confirm
  that label actually exists. Don't infer.
- For each candidate, fetch the body / read the description. Don't
  trust the title.
- For each one, decide: in scope for this sweep? Some candidates will be:
  - Already fixed (verify against current state of the repo).
  - Too large / needs design.
  - Environmental (only reproduces on a setup the operator can't drive).
  - Duplicates of each other.

Use `AskUserQuestion` to lock the final set with the operator before
fanout.

#### 1b. Look for shared root causes

Two items with similar symptom wording often share a root cause.
Investigate the simplest one first and gate the related ones on its
outcome. Don't spawn owners for issues that might be duplicates.

#### 1c. Identify shared-resource conflicts

Some work needs exclusive access to a resource: docker compose stack
with fixed container names, host ports, shared databases, the user's
running services. **If two items both need the same exclusive
resource, they cannot truly parallelize.** Either:
- Sequence them in waves.
- Or instruct their owner agents to skip the conflicting verification
  step and have the orchestrator run it serially at the end.

#### 1d. Confirm with the operator

Surface concretely:
- Final list of issues in scope
- Any items requiring operator-side verification (e.g. Safari testing
  on a Mac the cloud workstation can't drive)
- PR strategy (default: one draft PR per item)

Don't fanout until the operator confirms.

### 2. Worktree setup

One worktree per item. Convention:

```
.worktrees/<prefix>-<id>-<short-slug>/    # e.g. bug-251-write-outputs
branch: claude/<prefix>-<id>-<short-slug>
```

Create all worktrees in parallel (multiple Bash calls in one message).

### 3. Fanout: dispatch one **opus owner** per issue

Each issue gets a `ed3d-plan-and-execute:task-issue-owner` subagent
(opus, end-to-end scope baked into the agent definition). The
orchestrator does NOT run a separate review pass — each owner runs its
own review/fix loop, pushes its own branch, and opens its own draft PR.

**Why opus and not the cheaper task-implementor-fast (haiku):** the
owner role demands judgment under conditions where smaller models have
observably failed — TDD red→green discipline, multi-pattern grep for
orphan consumers in producer/consumer migrations, verbatim cycle
reporting, and nuanced root-cause finding. Use task-implementor-fast
only inside a phased plan where the spec is tight and the agent's job
is mechanical execution.

**Dispatch all owners in one message so they run in parallel.**

Because the agent's own system prompt covers the workflow, your dispatch
prompt only needs the per-issue context. Template:

```
## Worktree
Operate in: <absolute worktree path>
Branch: <branch name>

## Issue
Issue: #<n> <title>
Repo: <owner/repo>

<full issue body / reproducer pasted inline>

## Verified root cause (if any)
<paste the orchestrator's pre-investigation findings, OR omit if the
owner needs to investigate>

## Out of scope
- <files NOT to touch>
- <constraints>
- <project conventions worth surfacing: arrow-function style, no
  PR-context comments in code, etc.>

## Shared-resource note (if applicable)
Other owners in this sweep also need <resource — e.g. docker compose
stack>. Skip <specific verification step> in your workflow; the
orchestrator will run it serially at the end.

## Repo-specific PR workflow
<e.g. "use the smite-pr skill" or "use gh pr create --draft with the
processing_small template">
```

The agent's own prompt covers the rest (implement → verify → self-
review → fix loop → push → draft PR → verbatim cycle log). You don't
need to repeat the workflow.

### 4. Orchestrator's job while owners are running

Almost nothing. You waited for all owners to finish, you don't poll
or sleep. The Agent tool fires a notification when a subagent
completes. While owners are working:
- Don't open speculative reviews
- Don't second-guess decisions
- Don't fetch their files

If multiple owners need the same exclusive resource (compose stack),
their prompts should already instruct them to skip that verification
step. You'll run the shared verification serially at the end.

### 5. Synthesis

When all owners return:
- **Print each owner's full report verbatim** to the operator. Per the
  human-transparency rule, the operator cannot see subagent output —
  your relay is their window.
- Synthesize a final summary table: issue → PR → cycles → risk →
  operator action items.
- Surface any owner that hit the three-strike rule or escalated.

## Owner-internal discipline (defaulted in the agent)

The `task-issue-owner` agent's own system prompt enforces:

- Full pre-commit gate before review (not piecemeal tool checks)
- Red→green TDD (verify the test fails on base before claiming it as a
  regression test)
- Multi-pattern grep for producer/consumer migrations + a test-sweep
  cross-check before declaring no orphans
- Three-strike rule for the review loop
- Verbatim cycle log in the final report
- Draft PRs only — operator promotes

The orchestrator can override any of these in the dispatch prompt
(e.g. for trivial config-only fixes where TDD isn't applicable), but
the defaults are designed around the failure modes that bit prior
parallel sweeps.

## What an owner's final report looks like (good)

```
# Issue #158 owner report

Outcome: PR https://github.com/.../pull/291, ready for operator review.
Cycles: 3
Risk: Medium (10 files, but engine sweep matches main baseline)

## Cycle 1
- Implemented: <diff summary>
- Verification: <pre-commit output>
- Reviewer findings: <verbatim>
- Outcome: BLOCKED — Critical: tests don't go red on base.

## Cycle 2
- Bug-fixer addressed: <list>
- Reviewer findings: <verbatim>
- Outcome: BLOCKED — Critical: CallExecutor orphan, 360 test failures.

## Cycle 3
- Bug-fixer addressed: <list, including discovered VariablesMustBeDefined orphan>
- Reviewer findings: APPROVED
- Push: <hash>
- PR opened: #291

## Operator action items
None.
```

## What an owner's final report looks like (bad — fix the prompt)

```
Done. Fixed the bug. PR is up.
```

If owners return reports like the second, the dispatch prompt was too
loose. Tighten the "final report" section.

## Common rationalizations and what they really mean

| Rationalization | Reality |
|---|---|
| "I'll run the reviews from the orchestrator so I can see them mid-flight" | Throws away the parallelism. Pick: visibility OR speed. For independent issues, speed wins; the owner's final report gives you visibility after the fact. |
| "I'll skip the verification gate to save a cycle" | The reviewer will catch the lint/format issues — burns the cycle anyway, plus the reviewer's time on style instead of substance. |
| "Owner has reported BLOCKED, let me just fix it myself" | Owners hit three-strike for a reason. Question the architecture or the issue's scope before grabbing the keyboard. |
| "I'll dispatch all owners and then dispatch all reviewers" | That's the OLD pattern. Wave-based reviews from the orchestrator. Use this skill for nested fanout instead. |

## Anti-patterns

| Anti-pattern | Why it bites |
|---|---|
| Fanout before locking scope with the operator | Wasted owner-agent context on out-of-scope items |
| One mega-PR with all fixes | Loses per-issue review focus, hard to revert one |
| Parallel work on the same compose stack without coordination | Container-name collisions break everyone's verification |
| Orchestrator running its own review loop on top of owner's | Doubles the review work; trust the owner's cycle log |
| Owner summarizes instead of reporting verbatim | Operator loses visibility; can't catch missed issues |
| Owner pushes a non-draft PR | Orchestrator/operator loses the merge gate; default to draft |
| Owner skips the verification gate to "save time" | Reviewer finds the format issues, cycle wasted |

## Quick reference

| Phase | Owner of work |
|---|---|
| 1. Pre-flight (scope, conflicts, operator confirm) | Orchestrator |
| 2. Worktrees | Orchestrator |
| 3. Fanout (one owner per issue) | Orchestrator dispatches; owners run independently |
| 4. Implement / verify / review / fix / re-review / push / PR | **Each owner, end-to-end** |
| 5. Synthesis: print each report, summarize | Orchestrator |
| 6. Operator handoff | Orchestrator |

## When this fails

If owners consistently report incomplete cycle logs, vague summaries,
or skipped verification, the dispatch prompt needs tightening — not
the agent. The owner agent (`task-implementor-fast`) defaults to
implementation scope; the **end-to-end ownership** comes from the
prompt you hand it. Treat the dispatch template as the contract.

If the orchestrator's context is filling fast from synthesis output,
that's a signal owners are reporting too verbosely OR the sweep is
larger than one session can hold. Checkpoint with the operator: clear
context, hand off the PR list as the new entry point for follow-up.
