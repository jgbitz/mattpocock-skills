# Adopting Matt Pocock's Skills Into claude-config

**Research Date:** 2026-05-26
**Context:** Evaluation of the 14 skills in
`mattpocock-skills/skills/engineering/` and
`mattpocock-skills/skills/productivity/` for adoption into
`~/src/claude-config`. Answers four questions: (1) how to keep adopted skills in
sync with upstream, (2) how to translate every GitHub-issue dependency to the
MultiBilling `ticket-cli` flow, (3) which bucket each skill belongs in (`shared`
/ `work` / `personal`), and (4) whether the PRD concept should fold into the
existing `/document` skill.

---

## Table of Contents

- [Source Repo](#source-repo)
- [Skill Inventory](#skill-inventory)
- [Findings](#findings)
  - [Q1: Upstream-sync workflow](#q1-upstream-sync-workflow)
  - [Q2: GitHub-issue dependencies and ticket-cli mapping](#q2-github-issue-dependencies-and-ticket-cli-mapping)
  - [Q3: Shared / Work / Personal bucket assignments](#q3-shared--work--personal-bucket-assignments)
  - [Q4: PRD concept vs `/document`](#q4-prd-concept-vs-document)
- [Recommendations](#recommendations)
- [Open Questions](#open-questions)
- [Sources](#sources)

---

## Source Repo

| Field                 | Value                                                                           |
| --------------------- | ------------------------------------------------------------------------------- |
| Fork location         | `~/src/mattpocock-skills`                                                       |
| Upstream              | `https://github.com/mattpocock/skills`                                          |
| Skill count evaluated | 14 (10 engineering + 4 productivity)                                            |
| Out-of-scope          | `skills/misc/`, `skills/personal/`, `skills/in-progress/`, `skills/deprecated/` |

---

## Skill Inventory

Single-page summary of all 14 evaluated skills with the three properties that
drove every downstream recommendation.

| #   | Skill                           | `disable-model-invocation` | GitHub-issue dependency                        | PRD / doc concept                           | Composes with                                       |
| --- | ------------------------------- | -------------------------- | ---------------------------------------------- | ------------------------------------------- | --------------------------------------------------- |
| 1   | `setup-matt-pocock-skills`      | yes                        | configures the tracker for the suite           | configures domain doc layout                | bootstrap for skills 2-10                           |
| 2   | `diagnose`                      | no                         | none                                           | reads `CONTEXT.md` and ADRs                 | optional handoff to `improve-codebase-architecture` |
| 3   | `grill-with-docs`               | no                         | none                                           | writes `CONTEXT.md` + ADRs inline           | called by `triage`, `improve-codebase-architecture` |
| 4   | `triage`                        | no                         | **heavy** — `gh issue view/comment/edit/close` | reads `CONTEXT.md`, writes `.out-of-scope/` | calls `grill-with-docs`; consumes setup config      |
| 5   | `improve-codebase-architecture` | no                         | none (writes HTML to `$TMPDIR`)                | reads `CONTEXT.md`/ADRs, may write ADRs     | optional handoff from `diagnose`                    |
| 6   | `tdd`                           | no                         | none                                           | reads `CONTEXT.md`/ADRs                     | none                                                |
| 7   | `to-issues`                     | no                         | **heavy** — `gh issue create` per slice        | consumes plan/PRD                           | downstream of `to-prd`                              |
| 8   | `to-prd`                        | no                         | **heavy** — PRD published as a single GH issue | **produces PRDs**                           | feeds `to-issues`                                   |
| 9   | `zoom-out`                      | yes                        | none                                           | uses `CONTEXT.md` vocab                     | none                                                |
| 10  | `prototype`                     | no                         | none                                           | none                                        | none                                                |
| 11  | `caveman`                       | no                         | none                                           | none                                        | none                                                |
| 12  | `grill-me`                      | no                         | none                                           | none                                        | lightweight cousin of `grill-with-docs`             |
| 13  | `handoff`                       | no                         | none (writes to OS tmpdir)                     | none                                        | none                                                |
| 14  | `write-a-skill`                 | no                         | none                                           | none                                        | none                                                |

GitHub-issue coupling is concentrated in three skills (`triage`, `to-issues`,
`to-prd`) plus the optional `setup-matt-pocock-skills` config layer. The
remaining 10 are tracker-agnostic.

---

## Findings

### Q1: Upstream-sync workflow

The Pocock repo is a normal GitHub repo and the fork at
`~/src/mattpocock-skills` is already in place. The cleanest import model is
**per-skill copy with provenance stamps**, not `git subtree`, not `git
submodule`, and not a wholesale fork merge.

**Why not `git subtree` or `submodule`:**

- Skills are re-bucketed on import (the upstream `skills/engineering/triage/` lands in `shared/`, `work/`, or `personal/` depending on per-skill analysis).
- Some skills are edited heavily (`s/gh issue/ticket-cli/g`, `s/issue tracker/MultiBilling/g`).
- Both tools fight the existing `just sync` source-of-truth model that `claude-config` uses for every other skill.

**Workflow (one-time setup, then per-skill imports, then refresh on demand):**

1. Track upstream on the fork. Run once in `~/src/mattpocock-skills`:

   ```bash
   git remote add upstream https://github.com/mattpocock/skills.git
   git fetch upstream main
   ```

   Refresh with `git fetch upstream && git merge upstream/main` (or rebase)
   periodically. The fork is the staging ground; `claude-config` is the source
   of truth for the deployed copy.

2. Copy each adopted skill into `~/src/claude-config/skills/<bucket>/<skill>/` and drop a one-line provenance stamp:

   ```text
   ~/src/claude-config/skills/<bucket>/<skill>/
     SKILL.md
     ...
     .upstream
   ```

   `.upstream` contents (one line):

   ```text
   mattpocock/skills@<commit-sha> path=skills/engineering/<skill>
   ```

3. Add a `just sync-upstream` recipe to `claude-config`. It walks every `.upstream` stamp, runs `git -C ~/src/mattpocock-skills log <sha>..upstream/main -- <path>` per skill, and prints a digest:

   ```text
   diagnose          → 3 commits, 2 files changed since 2026-05-26
   tdd               → up to date
   grill-with-docs   → 1 commit, 1 file changed since 2026-05-26
   ```

   No automatic merging — the recipe surfaces drift; the human decides what to
   pull.

4. Update an adopted skill via the existing `claude-config` workflow:
   - Edit the file in `~/src/claude-config/skills/<bucket>/<skill>/`.
   - Refresh the `.upstream` stamp to the new upstream commit if the edit reflects an upstream change.
   - Run `just sync` to deploy to `~/.claude/`.
   - Commit and push in `claude-config`.
   - Never edit `~/.claude/skills/...` directly — `just sync` will overwrite it.

5. Use `skills/in-progress/` for experimental or uncertain adoptions until they earn a permanent bucket. This convention already exists in `claude-config`.

**Cost:** one stamp file per skill, one `just sync-upstream` recipe.

**Payoff:** upstream drift is visible per skill, local edits are preserved, no
rebase pain, no `git subtree pull` foot-guns.

### Q2: GitHub-issue dependencies and ticket-cli mapping

Three skills are deeply coupled to GitHub Issues. Each has a different
translation strategy for work-side adoption.

#### `to-prd` → MultiBilling equivalent

**What Pocock does:** synthesizes the current conversation into a PRD with these
sections, then publishes it as a single GitHub issue with the `ready-for-agent`
label:

- Problem Statement
- Solution
- User Stories
- Implementation Decisions
- Testing Decisions
- Out of Scope
- Further Notes

**Existing MultiBilling equivalent:** a Plan committed to
`multibilling-infra/docs/plans/YYYY-MM-DD-slug.md`. Plans land directly on
`main` (no PR, per the muni XCLAUDE.md "Plans and Docs Go Straight to Main"
rule), and `writing-plan-tickets` already turns the Plan file into a batch of
`ticket-cli` tickets.

| Axis                  | Pocock `to-prd`       | MultiBilling Plan + `writing-plan-tickets`                                                                |
| --------------------- | --------------------- | --------------------------------------------------------------------------------------------------------- |
| Output                | A single GitHub issue | A committed file in `docs/plans/`                                                                         |
| Audit trail           | Issue history only    | Git history + the batch of derived tickets                                                                |
| Cross-references      | none                  | Plan cites `docs/standards/` and `docs/runbooks/`                                                         |
| Standards enforcement | none                  | `writing-plan-tickets` step 4 queries `muni-kb` MCP and cites governing standards in every derived ticket |

**Decision:** do **not** port `to-prd` for work. The existing Plan +
`writing-plan-tickets` chain is strictly more capable.

For personal projects, port `to-prd` mostly as-is — GitHub issues are the user's
stated preference there.

#### `to-issues` → MultiBilling equivalent

**What Pocock does:** reads a plan or PRD, breaks it into vertical-slice tickets
(HITL or AFK), publishes each as a GitHub issue with `## What to build / ##
Acceptance criteria / ## Blocked by`.

**Existing MultiBilling equivalent:** `writing-plan-tickets` reads a
`*-plan.md`, calls `ticket-cli` via `creating-tickets`, and emits `## Context /
## Tasks / ## Constraints / ## Acceptance Criteria` per ticket.

| Concept          | Pocock `to-issues`                             | `writing-plan-tickets`                                                   |
| ---------------- | ---------------------------------------------- | ------------------------------------------------------------------------ |
| Source           | PRD or freeform plan                           | `docs/plans/*-plan.md` committed to main                                 |
| Slice contract   | Tracer-bullet vertical slices, HITL/AFK tagged | One ticket per Task heading                                              |
| Body sections    | What to build, AC, Blocked by                  | Context, Tasks, Constraints, AC                                          |
| Dependency model | "Blocked by" in body                           | `--parent` (hierarchy) + "Depends On" (sequencing), explicitly separated |
| Standards block  | absent                                         | mandatory `Relevant Standards` block sourced from `muni-kb`              |

**Decision:** do **not** port `to-issues` for work — `writing-plan-tickets` is a
clear superset.

**One borrowable idea:** the explicit HITL vs AFK tagging. Adding "is this
human-gated or AFK-pickable?" as a field in `writing-plan-tickets` step 3 would
tighten the ticket batches without disrupting the rest of the flow.

For personal, port `to-issues` as-is; it pairs with `to-prd` and uses GitHub
issues.

#### `triage` → MultiBilling equivalent

This is the only skill that needs **real reshaping** for work-side adoption.
Pocock's triage is a label-driven state machine; `ticket-cli` uses statuses,
types, and assignment as its primitives.

**Pocock state machine:**

- Category labels: `bug`, `enhancement`
- State labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`
- Side effect for rejected enhancements: write to `.out-of-scope/<slug>.md`

**`ticket-cli` primitives observed in `creating-tickets` and
`writing-plan-tickets`:**

- Status (e.g. "In Progress" set by `/ticket start`) — not labels
- Type (`bug`, `feature`, `task`) — field, not label
- Assignment (default `--assign 2`, the user)
- No `ready-for-agent` distinction today — work is implicitly AFK-pickable

**Recommended port (work-side, renamed `triaging-tickets`):**

1. Replace the label state machine with status-driven gates:
   - "incoming" maps to `needs-triage` — created by `ticket-cli create` without `/ticket start`.
   - "blocked / waiting on reporter" maps to `needs-info` — capture this in a body comment + a custom status if `ticket-cli` supports one, otherwise a body tag.
   - "ready" maps to `/ticket start` candidacy.
   - "won't fix" maps to close with a comment.
2. Map the `ready-for-agent` vs `ready-for-human` distinction to **AFK / HITL tagging in the ticket body's Constraints section** — the same field the HITL/AFK borrow from Q2 would carry.
3. Map `wontfix-enhancement` to a new `multibilling-infra/docs/out-of-scope/<slug>.md` convention. Reference the file from the ticket's close comment.
4. Keep the reproduce step verbatim — it's solid and language-agnostic. The grilling sub-step calls `riz-requirements-builder` or `grill-me` instead of `grill-with-docs`.

**Rename rationale:** the work-side variant diverges enough from upstream that
calling it `triage` would mislead. `triaging-tickets` makes the divergence
honest.

For personal, port `triage` as-is using GitHub issues.

#### `setup-matt-pocock-skills` — what to do with it

This skill is the per-repo config layer that writes
`docs/agents/issue-tracker.md`, `docs/agents/triage-labels.md`, and
`docs/agents/domain.md`.

| Context        | Decision                                                                                                                                                                                                                                               |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Personal repos | Adopt. Run once per repo to scaffold the config the other personal skills read.                                                                                                                                                                        |
| Work repos     | Skip. `multibilling-docs` + the `CLAUDE.md` hierarchy + the `docs/standards/` and `docs/registry/` conventions already establish where docs and the tracker live. Forcing `docs/agents/` into every muni service repo would add noise without benefit. |

**Decision:** personal-only.

### Q3: Shared / Work / Personal bucket assignments

Bucket rules used:

- **shared** — no tracker dependency, no muni-specific or personal-repo-specific assumptions baked in
- **work** — uses `ticket-cli`, muni standards, muni-kb MCP, or other MultiBilling-specific tooling
- **personal** — uses GitHub Issues or assumes a personal-repo workflow

| Skill                           | Recommended bucket                                                              | Reasoning                                                                                                                                                                   |
| ------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `diagnose`                      | **shared**                                                                      | No tracker dependency. Reads `CONTEXT.md`/ADRs which neither work nor personal repos use today — soften that requirement on import (see Open Questions).                    |
| `grill-with-docs`               | **shared**                                                                      | No tracker dependency. The `CONTEXT.md` and ADR write side is repo-agnostic.                                                                                                |
| `improve-codebase-architecture` | **shared**                                                                      | Writes HTML to `$TMPDIR`. Pure code-analysis skill, repo-agnostic.                                                                                                          |
| `tdd`                           | **shared**                                                                      | Pure red/green/refactor discipline. No tracker dependency.                                                                                                                  |
| `zoom-out`                      | **shared**                                                                      | One-liner. Trivially universal.                                                                                                                                             |
| `prototype`                     | **shared**                                                                      | Throwaway-code discipline. No tracker dependency.                                                                                                                           |
| `caveman`                       | **shared**                                                                      | Communication mode. Fully universal.                                                                                                                                        |
| `grill-me`                      | **shared**                                                                      | Lightweight cousin of `grill-with-docs`. No dependencies.                                                                                                                   |
| `handoff`                       | **skip**                                                                        | Superseded by the existing `handing-off-session` skill (richer state capture, `.claude/handoffs/` convention, git branch tracking). Pocock's version is a 17-line skeleton. |
| `write-a-skill`                 | **skip**                                                                        | Superseded by the existing `naming-skills` skill plus the `superpowers:writing-skills` plugin.                                                                              |
| `setup-matt-pocock-skills`      | **personal**                                                                    | Per-repo config layer the work side does not need. Likely rename to drop the "matt-pocock" branding.                                                                        |
| `to-prd`                        | **personal**                                                                    | Publishes to GitHub issues. Work side already covered by Plan + `writing-plan-tickets`.                                                                                     |
| `to-issues`                     | **personal**                                                                    | Same as `to-prd` — work side already covered by `writing-plan-tickets`.                                                                                                     |
| `triage`                        | **split** — personal as-is, work as `triaging-tickets` with the reshape from Q2 | Different enough that two skills is honest.                                                                                                                                 |

**Net adoption count:**

| Bucket          | Count                                                           |
| --------------- | --------------------------------------------------------------- |
| `shared/`       | 8                                                               |
| `work/`         | 1 (the reshaped `triaging-tickets`)                             |
| `personal/`     | 4 (`setup-matt-pocock-skills`, `to-prd`, `to-issues`, `triage`) |
| Dropped         | 2 (`handoff`, `write-a-skill`)                                  |
| Total evaluated | 14 (with one split → 15 total skill copies created)             |

### Q4: PRD concept vs `/document`

**Decision:** keep them separate. Do not add a fifth "PRD" doc type to
`multibilling-docs`.

The Pocock PRD and the `multibilling-docs` framework look similar from a
distance but solve different problems.

| Axis                 | Pocock PRD (`to-prd`)                           | MultiBilling Plan (`multibilling-docs`)            |
| -------------------- | ----------------------------------------------- | -------------------------------------------------- |
| Output medium        | A GitHub issue                                  | A committed file in `docs/plans/`                  |
| Audience             | A future AFK agent reading the issue            | A reviewer + the executing agent + future readers  |
| Lifecycle            | Closed when the work ships                      | Archived after build                               |
| Content emphasis     | User stories, problem statement, solution shape | Implementation blueprint, decisions, ordered tasks |
| Standards crosslinks | none                                            | core — research → standard → plan workflow         |

**Three reasons not to fold PRD into `multibilling-docs`:**

1. The framework's strength is that each of Standard, Research, Plan, Runbook answers a **distinct** question. PRD's question — "what user need are we solving and what's the shape of the solution?" — is upstream of Plan, not the same shape. Adding it would dilute the framework.
2. The "upstream of Plan" slot is already partially filled by `riz-requirements-builder`, which produces 5-phase yes/no specs in `requirements/YYYY-MM-DD-HHMM-<slug>/`. That covers the requirements-gathering need for work.
3. The Pocock PRD's output is a GitHub issue, not a file. Folding an issue-emitting workflow into a file-emitting framework would force one to morph into the other; both lose.

**Conclusion:** the PRD concept lives entirely in `personal/to-prd` and partners
with `personal/to-issues`. `multibilling-docs` is untouched.

---

## Recommendations

### Phase 1: Sync infrastructure (one-time)

1. Add the upstream remote on the fork:

   ```bash
   cd ~/src/mattpocock-skills
   git remote add upstream https://github.com/mattpocock/skills.git
   git fetch upstream main
   ```

2. Add the `just sync-upstream` recipe to `~/src/claude-config/justfile`. The recipe walks every `.upstream` stamp file under `skills/` and prints a per-skill drift digest.

### Phase 2: Adopt the 8 shared-bucket skills

Import order (lowest-risk first, to validate the workflow before touching
anything contentious):

1. `caveman`
2. `zoom-out`
3. `prototype`
4. `grill-me`
5. `tdd`
6. `diagnose`
7. `grill-with-docs`
8. `improve-codebase-architecture`

For each: copy into `~/src/claude-config/skills/shared/<skill>/`, write the
`.upstream` stamp, soften any `CONTEXT.md` requirement to "optional, ask if
missing" (skills 5–8), run `just sync`, commit.

### Phase 3: Adopt the 4 personal-bucket skills

Order:

1. `setup-matt-pocock-skills` (rename to drop the branding — candidate: `bootstrapping-personal-repo` or `setting-up-skill-config`)
2. `to-prd`
3. `to-issues`
4. `triage`

These four are interdependent and assume GitHub Issues. Adopt them together.

### Phase 4: Build the work-side `triaging-tickets` skill

This is net-new work, not a copy-paste. Use the reshape spec in Q2 as the
starting point. Create
`~/src/claude-config/skills/work/triaging-tickets/SKILL.md` from scratch, citing
`mattpocock/skills` as inspiration in a header comment, but build the body
around `ticket-cli` primitives.

### Phase 5: Tighten `writing-plan-tickets` with HITL/AFK tagging

Borrow the one good idea from `to-issues`: add an explicit HITL-or-AFK tag to
each ticket in `writing-plan-tickets` step 3, surfaced in the ticket body's
Constraints section. Small change, sharpens the batch.

### Do not touch

- `multibilling-docs` — PRD does not belong as a fifth doc type
- `handing-off-session`, `naming-skills` — Pocock's handoff and write-a-skill are skipped

---

## Open Questions

1. **`CONTEXT.md` and ADRs in muni repos.** Four shared-bucket skills (`diagnose`, `grill-with-docs`, `improve-codebase-architecture`, `tdd`) reference `CONTEXT.md` (domain glossary) and `docs/adr/` (architectural decisions). Neither exists in muni repos today. Two options:

   - **(a)** Adopt `CONTEXT.md` as a glossary file in muni repos. It pairs well with the existing `docs/standards/` and could capture domain terms the registry doesn't.
   - **(b)** Soften the requirement on import to "if `CONTEXT.md` exists, use it; otherwise ask the user where domain language lives."

   Recommend **(b)** as the default — adopt the skills, soften the dependency.
   Revisit `CONTEXT.md` adoption separately if a need emerges.

2. **`.out-of-scope/` convention in work.** Pocock's `triage` writes rejected enhancements to `.out-of-scope/`. The work-side `triaging-tickets` would write to `multibilling-infra/docs/out-of-scope/`. Confirm before building: is `docs/out-of-scope/` the right place, or does this belong elsewhere (Research? a runbook section)?

3. **Rename of `setup-matt-pocock-skills`.** If adopted into `personal/`, the "matt-pocock" branding makes no sense. Candidates: `bootstrapping-personal-repo`, `setting-up-skill-config`, `setting-up-agent-docs`. Pick during import.

---

## Sources

All Pocock skill SKILL.md files read during evaluation:

- `skills/engineering/setup-matt-pocock-skills/SKILL.md`
- `skills/engineering/diagnose/SKILL.md`
- `skills/engineering/grill-with-docs/SKILL.md`
- `skills/engineering/triage/SKILL.md`
- `skills/engineering/improve-codebase-architecture/SKILL.md`
- `skills/engineering/tdd/SKILL.md`
- `skills/engineering/to-issues/SKILL.md`
- `skills/engineering/to-prd/SKILL.md`
- `skills/engineering/zoom-out/SKILL.md`
- `skills/engineering/prototype/SKILL.md`
- `skills/productivity/caveman/SKILL.md`
- `skills/productivity/grill-me/SKILL.md`
- `skills/productivity/handoff/SKILL.md`
- `skills/productivity/write-a-skill/SKILL.md`
- `skills/engineering/setup-matt-pocock-skills/issue-tracker-github.md` (the GitHub seed config)
- `CONTEXT.md` (repo domain glossary)
- `README.md` (repo overview and skill reference index)

Existing `claude-config` skills read for overlap analysis:

- `skills/work/creating-tickets/SKILL.md`
- `skills/work/writing-plan-tickets/SKILL.md`
- `skills/work/multibilling-docs/SKILL.md`
- `skills/work/riz-requirements-builder/SKILL.md`
- `skills/work/handing-off-session/SKILL.md`

Framework references:

- `~/.claude/skills/multibilling-docs/references/templates.md` (Research template)
