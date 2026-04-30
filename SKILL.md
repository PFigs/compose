---
name: compose
description: >
  Plan and execute work with Jazz-era agents. Activates on "/compose",
  "let's plan", "let's compose", "plan this", "design this",
  or when entering plan mode for non-trivial work.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
  - AskUserQuestion
  - TaskCreate
  - TaskUpdate
  - TaskGet
---

# Compose

A Jazz-themed planning and execution skill. Ella Fitzgerald runs the show.

## The Ensemble

| Name | Legend | Role |
|------|--------|------|
| **Ella** | Ella Fitzgerald | Orchestrator -- you, running this skill |
| **Bird** | Charlie Parker | Explorer -- fast codebase reconnaissance |
| **Duke** | Duke Ellington | Architect -- structures the implementation plan |
| **Billie** | Billie Holiday | Reviewer -- code review with unflinching honesty |
| **Monk** | Thelonious Monk | Test writer -- finds edge cases nobody expects |
| **Count** | Count Basie | Dispatcher -- coordinates the executing ensemble |
| **Hawk** | Coleman Hawkins | Implementer -- does the actual work |
| **Django** | Django Reinhardt | Spec compliance -- verifies Hawk built what was asked |

## Setup

Ensure redliner is installed:
```bash
redliner --version
```

If not installed: `uv tool install redliner` or `pip install redliner`.

## Engineering principles

Compose enforces two non-negotiable disciplines wherever it touches code. These are rigid, not flexible -- if a phase would skip them, stop and reconcile.

- **TDD (red-green-refactor).** Tests come first and MUST fail before implementation. Implementation makes them pass with no extra scope. Refactor only when green. Monk writes the gates (Phase 5), Hawk obeys them (Phase 7), Django verifies tests were not silently rewritten (Phase 7 Step 2), Billie verifies the discipline held.
- **SOLID + KISS / DRY / YAGNI.** Duke designs interfaces with single-responsibility, stable abstractions, dependency inversion at boundaries, and open-for-extension seams (Phase 4). Billie reviews every change against SOLID *and* DRY/KISS/YAGNI (Phase 7 Step 2). Reject premature abstraction and god-classes equally.

## Composing with other available skills

Compose is an orchestrator -- it does not duplicate work that specialist skills already do well. The user's environment may expose other skills that are stronger for specific phases (planning, testing, implementing, reviewing, debugging, PR work, etc.). Compose discovers these dynamically rather than hardcoding any names.

### Session-start discovery

At the start of every `/compose` session, before Phase 0:

1. **Inventory available skills.** Read the current available-skills list surfaced by the harness.
2. **Filter to plausibly relevant ones** based on each skill's description. Look for matches against the work compose does at each phase: preparing/exploring, planning, writing tests, implementing, reviewing, debugging, verifying, PR / branch operations, writing PR descriptions.
3. **Ask the user, once, which to wire in.** Use a single AskUserQuestion (or two if the list is large) grouped by phase:

   ```
   "I see these skills are available that may slot into compose's phases. Which should I use?"
   - Plan / spec phase: <skill A>, <skill B>, none
   - Test-writing phase: <skill C>, none
   - Implementation phase: <skill D>, none
   - Review phase: <skill E>, <skill F>, none
   - Wrap-up / PR phase: <skill G>, <skill H>, none
   ```

   Multi-select per group. The user can pick none for any phase to use compose's built-in flow.

4. **Record the chosen wiring for this session.** Reuse it for every phase rather than re-asking. If the user says "skip this question for future sessions" or similar, save the choices to memory.

### Invoking the wired skills

When entering a phase that has a wired skill:
- Invoke the skill via the Skill tool with the relevant context for that phase.
- The skill's flow takes precedence; compose's built-in flow becomes a fallback.
- For phases with no wired skill, run compose's built-in flow as defined below.

### Mid-session changes

If a new skill becomes relevant later (e.g. a debugging skill once a bug appears), or if the user asks compose to swap skills, treat it the same way: ask once, record, proceed. Do not re-prompt at every phase.

## Path conventions

Plans go to one of two roots, picked at the start of every session:

1. **`<repo>/docs/plans/`** — preferred when this directory exists in the repo root. Treat plans as project artifacts visible in PRs.
2. **`<repo>/.claude/plans/`** — fallback when there's no `docs/plans/`. Plans stay agent-private.

Pick once per session and use the same root for spec, impl, and tests files. If `docs/plans/` exists but is empty, still prefer it.

## Complexity Assessment

Assess early, gate phases accordingly:

| Level | Signals | Phases |
|-------|---------|--------|
| **Light** | 1-2 files, clear pattern, isolated | Explore, Interview (2-3 Qs), Spec |
| **Medium** | 3-10 files, touches interfaces | + Impl Plan, Test Plan |
| **Heavy** | 10+ files, new patterns, cross-cutting | + Review Plan, full Execute |

## Phase 0: Detect and Resume

**Run before Phase 1 on every /compose entry.** Prevents loops where a fresh session re-specs work that already has a plan.

### Where to look

1. The plans root for this session (per **Path conventions** above): `<repo>/docs/plans/` or `<repo>/.claude/plans/`.
2. Both roots if they coexist.

### What signals existing work

In priority order:

1. **A subdirectory containing `STATUS.md`** — strongest signal. This is a multi-PR initiative split into per-PR plan files; STATUS.md tracks which PRs are pending / in-progress / done.
2. **A `*-spec.md` and matching `*-impl.md`** — single-file initiative. Status implicit from git history of the impl file.
3. **A `*-spec.md` alone** — initiative that stalled at spec; resume by writing the impl plan.

Match against the user's request by topic keyword in filenames + recent mtime. Don't false-positive on stale plans from months ago.

### Decision

If something matches:

```
AskUserQuestion:
  "I see existing plans for <topic> (last touched <date>). What would you like to do?"
  Options:
    - "Resume — next pending PR is PR<n> (<title>)"
    - "View status first" (read STATUS.md / the relevant plan and report, then re-ask)
    - "Start a new initiative" (proceed to Phase 1)
```

If user picks **resume** for a split-impl initiative:
- Read STATUS.md, find the first `pending` or `in_review` PR
- Skip to **Phase 7 (Execute)** with **just-in-time per-PR review** (see Phase 7)

If user picks **resume** for a single-impl initiative:
- Re-open the impl plan in redliner, do a quick review pass for any new context, then proceed to Phase 7

If user picks **new** or nothing matches: proceed to Phase 1 normally.

### Don't loop

If the user pivots mid-session to an unrelated request (e.g. "actually let's do X instead"), treat it as a fresh task — don't re-trigger Phase 0 detection on the new request unless they say `/compose` explicitly again. Phase 0 fires on `/compose` invocation, not on every message.

## Phase 1: Explore (Bird)

Dispatch 1-3 Bird agents (Explore subagent type) based on scope. Bird scouts:
- Relevant files, existing patterns, recent changes
- Conventions and constraints
- Similar past implementations

Bird is read-only. No user questions yet -- work from what's been said.

```
Agent(subagent_type="Explore", prompt="Bird reporting. Scout the codebase for <context>...")
```

## Phase 2: Interview (Ella)

Ella runs the interview directly. Rules:

1. **Always happens.** Even trivial tasks get 1-2 questions.
2. **One question at a time.** Multiple choice preferred.
3. **Depth scales with complexity:**
   - Light: 2-3 questions
   - Medium: 4-6 questions
   - Heavy: 6-10+ questions
4. **Always ask:**
   - What's in scope? What's out?
   - Where does relevant context live? (docs, other repos, team decisions)
   - What does "done" look like?
5. **Use Bird's findings** to ask informed questions, not generic ones.

Use AskUserQuestion for structured choices, plain text for open discussion.

## Phase 3: Spec (Ella writes, user reviews)

Write the spec to `<plans-root>/<YYYY-MM-DD>-<topic>-spec.md` (the chosen `<plans-root>` from **Path conventions**).

**Skip-spec mode:** if the user signals the spec should stay local-only ("don't commit the spec", "skip the spec from the PR", or similar), still write the file in the same location and run the review loop on it. The downstream impl plan is the durable artifact; the spec drives the conversation. When PR0 lands the impl plans, leave the spec uncommitted unless asked otherwise.

### Spec format

```markdown
# <Topic> Spec

## Goal
One paragraph: what and why.

## Scope
**In:** ...
**Out:** ...

## Approach
High-level how. Not implementation details.

## Key Decisions
Anything non-obvious, with rationale.

## Success Criteria
Concrete, verifiable outcomes.

## Open Questions
Anything unresolved.
```

### Review loop

After writing the spec, open the review UI in the background:

```bash
redliner open <absolute-path>
```

Run this with `run_in_background: true` via Bash. The browser opens automatically for the user.

Then use AskUserQuestion to wait:
- Question: "The review is open in your browser. Let me know when you're done."
- Options: "Done reviewing" / "Still working on it"

When the user signals done (or if the user told Ella to proceed automatically on approval):

1. `redliner status <spec-file>` -- check `approved_at` timestamp
2. If approved with no pending comments: proceed to next phase
3. If has pending comments:
   - `redliner list <spec-file>` -- read all comments
   - Address each (update spec, ask follow-up, or explain), then resolve via CLI
4. If not yet approved: the review UI is still running -- ask the user to continue reviewing
5. Repeat until redliner status shows `"status": "approved"`

**Auto-proceed:** If the user has said "proceed automatically when I approve" or similar,
skip the AskUserQuestion wait. Instead, poll `redliner status <file>` periodically
(every ~10s) and proceed as soon as `approved_at` is set.

Use the same review loop for the implementation plan and test plan reviews.

For **Light** complexity: spec is the final deliverable. Proceed to execution if the change is straightforward enough to not need a formal impl plan.

## Phase 4: Implementation Plan (Duke)

Medium+ only. Dispatch a Plan agent (Duke) with Bird's findings and the approved spec.

### Output shape: single file vs split directory

Pick based on the impl plan's natural shape:

- **Single file** (default for Medium, smaller Heavy): `<plans-root>/<YYYY-MM-DD>-<topic>-impl.md`. Use when the work is one PR or a few tightly-coupled tasks.
- **Split directory** (Heavy + multi-PR initiatives, ~5+ PRs): `<plans-root>/<YYYY-MM-DD>-<topic>-impl/` containing:
  - `00-overview.md` — architecture, pre-flight investigations, cross-cutting concerns, test strategy, PR table-of-contents
  - `STATUS.md` — progress tracker (template below); auto-generated by compose
  - `pr01-<slug>.md`, `pr02-<slug>.md`, ... — one file per PR with files, depends-on, tasks, code sketches

Trigger split when the impl plan would have 5+ independent shippable PRs.

### STATUS.md template (split-directory only)

When generating a split directory, write `STATUS.md` with this shape:

```markdown
# Status — <Topic>

> **Fresh Claude session?** Read this file, then `00-overview.md` for architecture. Find the first row with status `pending` (or `in_review` if mid-review).
>
> 1. Open the per-PR plan in redliner: `redliner open <plan-file>`
> 2. Wait for review/approval; address feedback; resolve comments before code work begins.
> 3. Update this file: status -> `in_progress`, fill in `Started` date.
> 4. Implement per the approved plan.
> 5. When the GH PR is opened, fill in `GH PR` with `#NNNN`.
> 6. When merged, status -> `done`, fill `Done` date, append a "what we learned" note.

## Status legend
| Value | Meaning |
|-------|---------|
| `pending` | Plan drafted, not yet reviewed/started. |
| `in_review` | Plan open in redliner; awaiting approval before code. |
| `in_progress` | Plan approved; PR work underway. |
| `done` | GH PR merged. |
| `blocked` | See Notes for reason. |

## PR table
| # | Title | Plan | Status | GH PR | Started | Done |
|---|-------|------|--------|-------|---------|------|
| PR0 | Plans landed | this directory | `in_progress` | — | <today> | — |
| PR1 | <title> | [pr01-<slug>.md](pr01-<slug>.md) | `pending` | — | — | — |
| ...

## Dependency graph
<ASCII or list of PR -> PR dependencies>

## Per-PR notes
> Append `### PRn — <title>` with 1-3 bullets per shipped PR. Feeds the next PR's just-in-time review.
```

The PR0 row tracks the plans-landing PR itself.

### Just-in-time per-PR review (split-directory default)

For split-directory plans, **do NOT redliner-review every per-PR file upfront**. Only:

- Review `00-overview.md` upfront in Phase 4.
- Each per-PR file is reviewed **immediately before that PR starts**, in Phase 7.

This lets earlier PR learnings shape later plans. State this rule in the overview's intro so a fresh session knows the contract.

### Single-file format

```markdown
# <Topic> Implementation Plan

## Architecture
Brief overview of the approach and key design decisions.

## Tasks

### Task 1: <description> [sequential]
**Files:** `path/to/file.py` (create|modify), `path/to/test.py` (create)
**Depends on:** none
<what to do, with code sketches for non-obvious parts>

### Task 2: <description> [parallel with Task 3]
**Files:** `path/to/other.py` (modify)
**Depends on:** Task 1
<description>

### Task 3: <description> [parallel with Task 2]
**Files:** `path/to/another.py` (create)
**Depends on:** Task 1
<description>
```

### Format

```markdown
# <Topic> Implementation Plan

## Architecture
Brief overview of the approach and key design decisions.

## Tasks

### Task 1: <description> [sequential]
**Files:** `path/to/file.py` (create|modify), `path/to/test.py` (create)
**Depends on:** none
<what to do, with code sketches for non-obvious parts>

### Task 2: <description> [parallel with Task 3]
**Files:** `path/to/other.py` (modify)
**Depends on:** Task 1
<description>

### Task 3: <description> [parallel with Task 2]
**Files:** `path/to/another.py` (create)
**Depends on:** Task 1
<description>
```

### Dependency markers

- `[sequential]` -- must run after all dependencies complete
- `[parallel with Task N, M]` -- can run alongside those tasks
- `**Depends on:** Task X` -- cannot start until X completes

### Interface design

When tasks share boundaries (APIs, function signatures, data contracts):
- Duke proposes the interface against the **SOLID checklist**: single responsibility per type, narrow interfaces (ISP), dependencies pointing at abstractions (DIP), seams that allow extension without modification (OCP), substitutable subtypes (LSP)
- Ella discusses it with the user in detail: method signatures, return types, error handling, usage examples
- The agreed interface goes into the impl plan as the source of truth
- Hawk and Django both reference it

Run the same redliner review loop on the impl plan.

## Phase 5: Test Plan (Monk)

Medium+ only. Dispatch Monk with the spec and impl plan.

Write test plan as a section in the impl plan (not a separate file) OR as `.claude/plans/<YYYY-MM-DD>-<topic>-tests.md` for heavy changes.

This phase is the **red** half of red-green-refactor. Monk's tests are not documentation -- they are the failing gate that Hawk's implementation must turn green. If a TDD-focused skill was wired in at session start (per **Composing with other available skills**), defer to it and skip Monk; otherwise run this phase as defined below.

### Monk's rules

- **Test what matters**: business logic, data transformations, edge cases
- **Skip trivialities**: `assert obj is not None`, dataclass defaults, config objects
- **No mocks**: test real functions, use fixtures if needed
- **Free functions**: `test_` prefix, no test classes
- **Full values**: `assert result == expected_list`, not `assert len(result) == 3`
- **Test-first gates**: tests that can be written before implementation, becoming acceptance criteria

### Test-first gates

Monk writes tests that:
1. Call functions from the target module (even if they don't exist yet)
2. Assert expected behavior based on the spec
3. Will FAIL before Hawk implements (by design)
4. Become the acceptance gate: implementation is done when Monk's tests pass

The test plan is reviewed BEFORE execution starts. AskUserQuestion if there are concerns about test coverage.

## Phase 6: Review Plan (Billie)

Heavy only. Billie identifies additional focus mandates for the execution phase:

- **Critical areas**: new behavior, changed behavior, security-sensitive code
- **Focus mandates**: specific review question per area (e.g., "verify discount calc handles negatives")
- **Regression risks**: existing behavior that could break

Written as a section in the impl plan:

```markdown
## Review Plan

### Focus Area 1: <description>
**Files:** `path/to/file.py`
**Mandate:** <specific question Billie should answer>
**Risk:** <what could go wrong>
```

These mandates supplement -- not replace -- the per-task quality reviews that happen for every task during execution.

## Phase 7: Execute (Count)

Count reads the impl plan and orchestrates using an **agent-per-task** model.
Every task gets three dedicated agents: Hawk (implement), Django (spec compliance), Billie (code quality).

### Resume-aware entry

Two execution shapes, picked from the impl plan's structure:

**Single-file impl plan:**
- Read the impl plan in full.
- Run task-by-task per the **Task loop** below.

**Split-directory impl plan (with STATUS.md):**
- Read `STATUS.md` in the impl directory. Find the first row with status `pending` or `in_review`.
- This is the **next-up PR**. Don't read the other per-PR files yet.
- Open that PR's plan file in redliner for **just-in-time review**:
  ```bash
  redliner open <plan-file> &
  ```
  Wait for user approval (same review loop as Phase 3 spec). Address any feedback. Resolve comments.
- Once approved, update STATUS.md: status -> `in_progress`, fill `Started` date.
- Run the **Task loop** below over the tasks defined in that single PR file.
- On PR completion (after `gt submit` and merge), update STATUS.md: status -> `done`, fill `Done` date, append a 1-3 bullet "what we learned" note in the **Per-PR notes** section.
- Return to the top of this section to find the next pending PR.

### Pre-flight

Before any implementation:
1. Verify working directory and branch state
2. Run baseline tests (if applicable)
3. If Monk wrote test-first gates: run them, verify they FAIL

### Task loop

Each task follows a write-review-iterate cycle. Three agents, one concern each:

```
Hawk (write) -> Billie (quality) + Django (compliance) -> fix if needed -> repeat
```

#### Step 1: Write (Hawk)

Spawn **Hawk** (general-purpose agent) with:
- Full task text from impl plan
- Relevant context (file paths, interface specs, test gates)
- Hawk's prompt template from `prompts/hawk.md`

Hawk follows TDD discipline: when test gates exist, run them first, confirm RED, implement to GREEN, refactor only when green. Never silently rewrite tests to make them pass. No scope beyond what makes the failing test pass.

#### Step 2: Review (Billie + Django, in parallel)

After Hawk reports DONE, spawn two agents in a single message:

**Billie** (general-purpose agent) -- code quality:
- Changed files and git diff
- General quality review scope: **SOLID** (single-responsibility, open-closed, Liskov, interface segregation, dependency inversion), DRY, KISS, YAGNI, security, regressions, **TDD discipline** (tests not weakened to make impl pass; new behavior covered by a failing-then-passing test)
- If Heavy and review plan has a focus mandate for this area, include it as additional scope
- Billie's prompt template from `prompts/billie.md`

**Django** (general-purpose agent) -- spec/impl compliance:
- Task spec and expected behavior from the impl plan
- Files Hawk changed
- Django's prompt template from `prompts/django.md`

#### Step 3: Iterate (if needed)

If either reviewer reports issues:
1. Collect findings from both Billie and Django
2. Spawn a new Hawk with the combined findings and the files to fix
3. After Hawk fixes, re-run Step 2 (both reviewers on the updated files)
4. Repeat until both Billie and Django report PASS

A task is done when both reviewers pass.

### Parallel execution

Tasks marked `[parallel]` get dispatched concurrently:
- Each task gets its own Hawk agent (separate Agent calls, launched together)
- After each Hawk completes, its Django + Billie agents run in parallel
- Conflicts (if any) are resolved by Count after all parallel tasks complete

### User feedback

At natural checkpoints (after each task group, or when user interrupts):
- Present status: completed, in-progress, blocked
- Relay any Hawk NEEDS_CONTEXT or BLOCKED escalations
- Accept course corrections and relay to active agents

### Wrap-up

After all tasks:
1. Run full test suite
2. If Monk wrote test gates: verify all pass
3. Before claiming done: produce evidence (command output, test results) -- never assert success without it
4. If a verification or completion-check skill was wired in at session start, defer to it for the final pass
5. Present summary: what was done, what changed, any concerns
6. Offer: commit / create branch / keep as-is / discard

## Prompts

Agent prompt templates live in `.claude/skills/compose/prompts/`:
- `hawk.md` -- Implementer
- `django.md` -- Spec compliance reviewer
- `billie.md` -- Code reviewer
- `monk.md` -- Test writer

When dispatching an agent, read the relevant prompt template and include it in the agent's prompt along with the task-specific context.
