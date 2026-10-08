---
name: code_feature_builder
description: End-to-end checklist for building a new feature or fixing a bug in an existing codebase. Covers clarifying scope, reading the nearest existing code before writing any, implementing in the established pattern, writing and running tests, checking deployment gaps, writing docs, checking migrations and backward compatibility, recording risks, and opening a draft PR. Use when the user asks to build, add, implement, fix, or debug something, even if they don't name this skill. Scales down for trivial fixes, but says out loud which steps it skipped and why.
argument-hint: [description of the feature to build or bug to investigate]
allowed-tools: Read, Grep, Glob, Bash(git *), Bash(gh *), Bash(make *), Bash(pytest *), Bash(ruff *), Bash(npm *), Edit, Write, Agent, Skill, AskUserQuestion, TaskCreate, TaskUpdate
user-invocable: true
---

# Code Feature Builder

A checklist that stops feature work and bug fixes from skipping steps that are cheap now and expensive later: understanding the existing code before touching it, testing at the right levels, catching deploy, migration, and docs gaps, and landing the change through the repo's real PR flow.

**Usage:** `/code_feature_builder <description of the feature or bug>`

Treat each phase as a question to answer, not a box to tick. If a phase doesn't apply, say so and why, then move on.

---

## Phase 0: Size the task

Use `TaskCreate` to list the phases you're committing to, so progress is visible and nothing is dropped silently.

- **Trivial** (one-line fix, obvious root cause, no new files, no interface change): compress Phases 1–4, skip 5–8 with a stated reason, and still ship it through Phase 9. A trivial fix is not exempt from a PR.
- **Real feature or non-trivial fix** (new files, new service or endpoint, new data path, changed public interface, anything that touches deployment or persisted data): run every phase.

When unsure, ask. Over-scoping a trivial fix wastes a minute; under-scoping a real feature can miss a migration nobody tested.

---

## Phase 1: Clarify the ask

If the request already names a concrete change with enough detail to act on, don't invent questions. Otherwise pin down:

- Is this "build something new" or "investigate before deciding"? Debugging starts with reproducing and root-causing; building starts with reading the nearest analogous feature.
- What observable result counts as success?
- Which components does it touch? Check the repo's top-level docs and any per-component `CLAUDE.md` or `README.md` for the map.
- Are there hard constraints (must not break production, must not touch a frozen area, must ship by a date)?

If the user wants help deciding what to build, the main output is the code reading in Phase 2, not an implementation.

---

## Phase 2: Read before you write

Don't shortcut this phase.

0. **Fetch, then read the base you will branch from.** Searches like `grep`, `git log -S`, or "does this exist?" answer against your local objects. A stale checkout can say something doesn't exist when it landed last week, and that looks identical to a true negative.

   ```bash
   git fetch origin
   git rev-list --count HEAD..origin/<base-branch>                       # how far behind
   git log --oneline HEAD..origin/<base-branch> -- <paths you care about> # did anyone touch your area?
   ```

   If the count is meaningful and the paths overlap, read those commits before designing. Someone may already have built it, or changed the pattern you were about to copy. A stale base costs a rebase; a conclusion drawn from a stale base can cost the whole design.

1. **Find the nearest analog.** Most features have a sibling: a similar module, test, or script. Read it fully, not just its signature.
2. **Read the local agent or contributor docs** (per-component `CLAUDE.md`, `CONTRIBUTING.md`, lint and test config). They override generic assumptions.
3. **Trace the real data or control flow**, not just the function you expect to edit.
4. **For debugging,** reproduce the symptom or find the log signature before forming a hypothesis.

If the relevant code spans several directories, delegate the reading to parallel subagents, one per area. Read and synthesize their findings yourself before designing. Don't hand the design decision to a subagent.

You're ready to build when you can state, in your own words: where the new code goes, which existing pattern it follows, and what it must not break.

---

## Phase 3: Build

- Follow the pattern from Phase 2 and the repo's conventions (import style, typing, logging, dependency placement).
- Match the file placement of the analog. Don't invent a new location or abstraction the task doesn't need.
- Don't add speculative flexibility, config knobs, or error handling for cases that can't occur. Build what this change needs.

---

## Phase 4: Test

Every change gets tests; scale depth to the task size.

- Add unit tests in the convention the repo already uses (e.g. next to the source file, or in the matching test directory).
- If the change crosses a service boundary (API, message bus, external tool), add or extend an integration test, reusing the repo's existing harness rather than inventing one.
- Run the repo's test, lint, and format targets locally before claiming done. Run lint and format last, after test fixes, since those edits can introduce new lint issues.
- If something can't run locally (needs GPU, cloud resources, or an interactive TTY), say so explicitly. Don't claim it passed.
- For UI changes, drive the feature in a browser. Type-checking alone is not verification.
- For runtime behavior that tests can't establish, exercise it end to end.

---

## Phase 5: Deployment gap check

Ask directly: how does this change reach users or production, and does something need to be built to get it there?

- New service or container: needs deployment wiring, an image, monitoring or dashboards?
- Model or data pipeline change: needs an artifact migration, cache warm-up, or a config change in the job runner?
- Config or schema change: do existing deployments have the old shape stored somewhere?
- Pure library or script change with nothing to deploy? Say so. This phase surfaces real gaps; it doesn't manufacture work.

If a supporting script (migration, backfill, deploy helper) is needed to land the feature, build it now rather than leaving a note.

---

## Phase 6: Docs

Size docs to what a future maintainer needs:

- **Mechanism:** how it works, and the key design decision with its reason (not a restatement of the code).
- **Invariants:** what must hold for it to keep working.
- **User guide:** the concrete command or UI flow, with a copy-pasteable example.
- **Maintenance and debugging:** how to tell it's broken, where logs live, common failures and recovery.

Update an existing doc in preference to creating a new one when the existing one is more discoverable. Use Mermaid for diagrams if the repo renders it.

---

## Phase 7: Migration and backward compatibility

Check explicitly. Don't assume "probably fine":

- **Wire formats and schemas:** run the repo's compatibility check against the base branch. A breaking change needs a deprecation path.
- **Database or stored data:** is there a migration, and does it handle rows written in the old shape?
- **Saved artifacts** (models, checkpoints, caches): do old versions still load?
- **Interfaces:** grep for callers before assuming there are none.

For a real gap, either build the mitigation (dual-read, versioned field, deprecation window) or flag the decision to the user. Don't silently ship a breaking change, and don't silently add a compatibility shim nobody asked for.

---

## Phase 8: Risks and limitations

State plainly, in the PR description and to the user, what the change does not handle: known edge cases left out, scale or performance assumptions, anything that only works because of an environment detail, and what would have to change if that assumption breaks. This is information the next maintainer needs and won't otherwise have.

---

## Phase 9: PR and review

1. Branch from the freshly fetched base: `git fetch origin && git switch -c <branch> origin/<base-branch>`. If the branch was cut earlier and the base has moved, rebase now (see `/rebase-guide`) rather than at review time.
2. Push and open the PR as a draft (`gh pr create --draft`). Use the repo's PR template if it has one; otherwise cover Problem, Solution, Alternatives, Review Focus, Testing, and Risk Assessment. Fold Phase 8 into Risk Assessment.
3. If CI is gated by a label or manual trigger, tell the user how to start it. Add the trigger yourself only if the user asks.
4. Once CI passes and the PR is complete, mark it ready for review (`gh pr ready <number>`).
5. Hand ongoing review-comment and CI monitoring to `/babysit-pr <PR_NUMBER>`, rather than polling by hand.

A change isn't done until a PR exists, at least as a draft. A finished diff sitting in the working tree hasn't landed.
