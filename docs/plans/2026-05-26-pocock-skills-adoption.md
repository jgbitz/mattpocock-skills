# Plan: Pocock Skills Adoption + multibilling-infra GLOSSARY Seed

**Date:** 2026-05-26
**Status:** Active
**Authors:** Jamie Gaines, Claude (Opus 4.7)

---

## Goal

Adopt the high-value subset of Matt Pocock's skills
(`mattpocock-skills/skills/engineering/` + `skills/productivity/`) into
`~/src/claude-config` in an order that earns maximum compounding value with
minimum throwaway work. The headline deliverables:

1. The `infra:follow-up` ticket backlog under active triage rather than accumulating.
2. A canonical domain glossary (`multibilling-infra/docs/standards/GLOSSARY.md`) and a decision register (`DECISION-REGISTER.md`) — the artifacts that make every future agent session more aligned.
3. `grill-with-docs` imported with the fork-on-import edits that respect the `multibilling-docs` framework (no separate ADRs, glossary lives as a standard).
4. The remaining shared and personal skills imported in batches, with a `just sync-upstream` recipe that surfaces upstream drift per skill.

---

## Scope

### In scope

- Phased import of 10 Pocock skills into `~/src/claude-config` per the per-skill decisions in [`sync-from-upstream-skills.md`](../runbooks/sync-from-upstream-skills.md)
- Creating `multibilling-infra/docs/standards/GLOSSARY.md` and `multibilling-infra/docs/standards/DECISION-REGISTER.md` as fresh standards
- Building `just sync-upstream` in `~/src/claude-config/justfile` after the first imported skill exists
- First real-world validation of each imported skill before the next batch

### Out of scope

- Adopting `handoff` and `write-a-skill` from upstream — superseded by existing `claude-config` skills (per the runbook)
- Reversing the `multibilling-docs` "no separate ADRs" rule — confirmed sound (see runbook § Framework Decisions)
- Replacing `superpowers:brainstorming` with `grill-with-docs` — confirmed they serve different phases; both stay in use
- AGENT_SYS-539 implementation work itself (the ticket is filed; landing it is a separate engineering task)
- Reshaping `improve-codebase-architecture` for IaC use cases — that skill is import-as-is but consciously not invoked against `multibilling-infra`

---

## Design

### Three logic threads driving the order

1. **Earn the artifact before scaling the skill.** GLOSSARY.md gets seeded *before* `grill-with-docs` is imported, so the skill has something to read on its first session. Importing the skill first would mean every first-session prompts "want to create GLOSSARY?" — friction with no immediate payoff.
2. **Validate one import before doing the batch.** `grill-with-docs` is the first import because it's the highest-value and most complex (multiple fork-on-import edits). If the fork shape has a bug, it's caught on one skill rather than across eight.
3. **Don't build infrastructure before the first user.** `just sync-upstream` is deferred until at least one skill is imported and tracked, so the recipe has a real use case to design around.

### Compounding value model

The plan's payoff is non-linear. The first few sessions of `grill-with-docs`
produce a sparse GLOSSARY — modest value. Six months in, every agent that
touches muni-infra reads a dense glossary and stops re-deriving terminology. The
compounding starts the moment GLOSSARY.md exists with even 25 entries, which is
why Phase 2 is gated on the seed pass, not on any skill import.

---

## Implementation Steps

### Phase 1: Use `triaging-followups` on a real slice (parallel to everything)

**Goal:** Validate the deployed skill against actual backlog, surface gaps
before polishing.

**Tasks:**

- Pick one parent epic (suggest: the oldest one with the most follow-ups)
- Run the skill end-to-end on that epic's follow-ups
- For each follow-up: read PR diff, recommend outcome, apply outcome with `ticket-cli`
- Note any friction in the jq workaround, the outcome table, the comment templates

**Acceptance criteria:**

- [ ] At least one epic's follow-ups fully processed (each ticket either Cancelled / Promoted / Deferred / Bundled / Scheduled)
- [ ] Each processed ticket carries `infra:triaged` and a triage comment
- [ ] Friction notes captured in a comment on the skill's commit or as a follow-up ticket
- [ ] Cancellation reasons inspected for patterns (Step 3 of the skill)

**Dependencies:** None. Can start immediately.

**Effort:** ~1-2 hours per epic. Process one epic per session.

---

### Phase 2: Seed `GLOSSARY.md` and create `DECISION-REGISTER.md` in `multibilling-infra`

**Goal:** Make the highest-compounding artifact exist. After this phase, every
agent session against muni-infra has a canonical vocabulary to read.

**Tasks:**

1. Dispatch a research agent against `multibilling-infra/docs/standards/`, `docs/registry/`, and the most recent 5 plan files in `docs/plans/`. Agent extracts:
   - Project-specific terms (filter out general k8s, Terraform, SQL terminology)
   - Per-term provenance: which standard or registry file each term is defined or used in
   - `_Avoid_` candidates: places where multiple synonyms get used for the same concept

2. Review the candidate list. Reject general-programming noise. Refine definitions to one or two sentences each. Target ~25-40 entries for the first commit.

3. Draft `multibilling-infra/docs/standards/GLOSSARY.md`:
   - Use the `multibilling-docs` Standard template (metadata block, Context, Decision, etc.)
   - The "Decision" is the rule: *use the canonical term, avoid the listed synonyms*
   - The "Best Practices" section is the term list itself, in `**Term**:\n One-sentence definition.\n _Avoid_: synonym1, synonym2` format

4. Draft `multibilling-infra/docs/standards/DECISION-REGISTER.md` as an empty scaffold:
   - Standard metadata block
   - One-paragraph header explaining what the register is and how to add entries
   - Empty markdown table with columns: Date, Decision, Standard

5. Commit both files to `main` (muni convention: plans and docs go straight to main).

**Acceptance criteria:**

- [ ] `GLOSSARY.md` exists with at least 25 entries, each citing the standard/registry file it came from
- [ ] `DECISION-REGISTER.md` exists with the scaffold ready for first append
- [ ] Both follow the `multibilling-docs` framework template (metadata block, no separate ADR concept)
- [ ] At least 5 of the 25 entries have `_Avoid_` mappings (the place where confused vocabulary already exists)
- [ ] Committed and pushed to `multibilling-infra` main

**Dependencies:** None. Highest-priority single action in this plan.

**Effort:** 1-2 hours total. Research agent dispatch ~15 min, triage and refine
~60 min, commit + push ~15 min.

---

### Phase 3: Import `grill-with-docs` with fork-on-import edits

**Goal:** Bring the first upstream skill into `claude-config` with all the
muni-framework-respecting edits applied.

**Tasks:**

1. Copy `~/src/mattpocock-skills/skills/engineering/grill-with-docs/` into `~/src/claude-config/skills/shared/grill-with-docs/` (SKILL.md + CONTEXT-FORMAT.md + ADR-FORMAT.md)

2. Apply fork-on-import edits per [`sync-from-upstream-skills.md`](../runbooks/sync-from-upstream-skills.md) § `grill-with-docs`:
   - **Glossary detection**: if `docs/standards/` exists, target `docs/standards/GLOSSARY.md`; otherwise fall back to root `CONTEXT.md`
   - **ADR redirect**: when a decision crystallizes, prompt for (a) land in existing standard, (b) become new standard, (c) skip — never create `docs/adr/` in a muni repo. After user picks (a) or (b), append a one-row entry to `DECISION-REGISTER.md`
   - **Drop the `CONTEXT-MAP.md` multi-context branch** — muni repos don't use it

3. Write the `.upstream` stamp file:

   ```text
   ~/src/claude-config/skills/shared/grill-with-docs/.upstream
   mattpocock/skills@<sha-from-fork> path=skills/engineering/grill-with-docs
   ```

4. Run `just sync` from `~/src/claude-config/` to deploy to `~/.claude/`

5. Commit + push the change in `claude-config`. Commit message follows the `feat(skills/shared):` pattern.

**Acceptance criteria:**

- [ ] `grill-with-docs` visible in skill list after `just sync`
- [ ] `.upstream` stamp file present
- [ ] Fork-on-import edits all applied (verified by reading the SKILL.md diff against upstream)
- [ ] No `docs/adr/` references remain in the imported SKILL.md (replaced by register-append flow)
- [ ] Committed to `claude-config` main

**Dependencies:** Phase 2 complete (GLOSSARY.md must exist before the skill
reads it).

**Effort:** ~45 minutes. Most of the time is the fork-on-import edits.

---

### Phase 4: First real use of `grill-with-docs`

**Goal:** Validate the fork-on-import edits work end-to-end on a real muni-infra
plan.

**Tasks:**

1. Wait until the next muni-infra plan comes up naturally (don't fabricate one)
2. Run the existing pipeline: `superpowers:brainstorming` → `superpowers:writing-plans` → `grill-with-docs` → `writing-plan-tickets`
3. During the `grill-with-docs` session, observe:
   - Does the skill detect `docs/standards/` and target `GLOSSARY.md`?
   - When a new term resolves, does it append to `GLOSSARY.md` correctly?
   - If a decision crystallizes, does the skill prompt for (a)/(b)/(c) and skip `docs/adr/`?
   - If outcome is (a) or (b), does it append to `DECISION-REGISTER.md`?
4. Capture any issues — fix in `claude-config/skills/shared/grill-with-docs/SKILL.md`, re-sync, commit

**Acceptance criteria:**

- [ ] One real plan walked through the full pipeline including `grill-with-docs`
- [ ] At least one new term added to `GLOSSARY.md` from the grilling session
- [ ] No `docs/adr/` directory appears in `multibilling-infra` after the session
- [ ] If a decision crystallized: it landed in a standard AND a row appears in `DECISION-REGISTER.md`
- [ ] Any fork-edit bugs fixed and re-committed

**Dependencies:** Phase 3 complete.

**Effort:** Embedded in normal plan work — no dedicated session needed.

---

### Phase 5: Build `just sync-upstream` recipe

**Goal:** Surface upstream drift on adopted skills so updates aren't missed.

**Tasks:**

1. Add a `sync-upstream` recipe to `~/src/claude-config/justfile`. Behavior:
   - Walk every `.upstream` stamp file under `skills/`
   - For each one, run `git -C ~/src/mattpocock-skills log <sha>..upstream/main -- <path>` to compute drift
   - Print per-skill digest: `<skill-name> → up to date` or `<skill-name> → N commits, M files changed since <sha>`

2. Verify `~/src/mattpocock-skills` has the `upstream` remote configured. If not, add it:

   ```bash
   cd ~/src/mattpocock-skills
   git remote add upstream https://github.com/mattpocock/skills.git
   git fetch upstream main
   ```

3. Test the recipe against the `grill-with-docs` import. Should report "up to date" if nothing has changed upstream since the import.

**Acceptance criteria:**

- [ ] `just sync-upstream` runs cleanly and produces a digest
- [ ] Upstream remote configured on the fork
- [ ] Recipe handles the "no `.upstream` stamps yet" case gracefully (early skills imported before this phase shouldn't crash it)

**Dependencies:** Phase 3 complete (at least one `.upstream` stamp must exist).

**Effort:** ~30 minutes.

---

### Phase 6: Batch-import the no-edit shared skills

**Goal:** Cheap wins. Three skills that need no fork-on-import edits.

**Skills:** `caveman`, `zoom-out`, `prototype`

**Tasks per skill:**

1. Copy `mattpocock-skills/skills/<area>/<skill>/` into `claude-config/skills/shared/<skill>/`
2. Write the `.upstream` stamp
3. Verify SKILL.md needs no edits (run `diff` against upstream — should be byte-identical)
4. `just sync`
5. Commit

**Acceptance criteria:**

- [ ] All three skills appear in skill list after `just sync`
- [ ] `.upstream` stamps written
- [ ] `just sync-upstream` reports all three as "up to date"
- [ ] One commit per skill (or one commit covering all three with clear scope notation)

**Dependencies:** Phase 5 complete (so the sync recipe can validate).

**Effort:** ~30 minutes total for all three.

---

### Phase 7: Batch-import the GLOSSARY-reading shared skills

**Goal:** Bring in the remaining skills that need glossary-detection edits.

**Skills:** `diagnose`, `tdd`, `improve-codebase-architecture`

**Tasks per skill:**

1. Copy from upstream
2. Apply glossary-detection edits — same logic as `grill-with-docs` Phase 3 step 2, item 1 (target `docs/standards/GLOSSARY.md` if present, fall back to root `CONTEXT.md`)
3. For `improve-codebase-architecture` additionally: apply the same redirect-to-standards + register-append logic for the rare case where it offers to record an ADR
4. Write `.upstream` stamps
5. `just sync` + commit

**Acceptance criteria:**

- [ ] All three skills imported with glossary detection
- [ ] No `docs/adr/` references remain in `improve-codebase-architecture/SKILL.md`
- [ ] `.upstream` stamps written for all three
- [ ] Conscious note (in `improve-codebase-architecture`'s description or a comment) that the skill is intentionally not run against IaC

**Dependencies:** Phase 6 complete.

**Effort:** ~1 hour. `improve-codebase-architecture` is the heaviest because of
its ADR-redirect edits.

---

### Phase 8: Personal-side skills

**Goal:** Import the personal-only skills so the personal-repo experience
improves.

**Skills:** `setup-matt-pocock-skills` (renamed), `to-prd`, `to-issues`,
personal `triage`

**Tasks:**

1. **`setup-matt-pocock-skills`**:
   - Rename to drop the branding (decide between `bootstrapping-personal-repo` and `setting-up-skill-config` at import time)
   - Remove the GitLab and "Other" issue-tracker branches — keep only GitHub
   - Mark explicitly personal-only in the description
2. **`to-prd`, `to-issues`, `triage`**: copy as-is (no fork-on-import edits needed — they target GitHub Issues which is the personal use case)
3. Write `.upstream` stamps
4. `just sync` + commit each

**Acceptance criteria:**

- [ ] Four skills present in `claude-config/skills/personal/`
- [ ] Renamed setup skill has the new name reflected in frontmatter and description
- [ ] All four marked clearly as personal-only
- [ ] `.upstream` stamps written

**Dependencies:** Phase 7 complete (or run in parallel — these don't affect
work-side flow).

**Effort:** ~1 hour total.

---

### Phase 9: AGENT_SYS-539 cleanup

**Goal:** Once the `--tag` / `--not-tag` ticket lands, strip the jq workaround
from `triaging-followups`.

**Tasks:**

1. Wait for AGENT_SYS-539 to merge and deploy
2. Update `claude-config/skills/work/triaging-followups/SKILL.md`:
   - Replace the jq pipeline in Step 1 with `ticket-cli list --tag infra:follow-up --not-tag infra:triaged --status "To Do"`
   - Remove the Known Gaps entry about missing flags
3. `just sync` + commit

**Acceptance criteria:**

- [ ] `ticket-cli list --tag` / `--not-tag` confirmed working (run the new commands manually first)
- [ ] Skill Step 1 uses native flags, no jq
- [ ] Known Gaps section updated
- [ ] Committed

**Dependencies:** AGENT_SYS-539 merged and deployed (external dependency).

**Effort:** ~15 minutes once the ticket lands.

---

## Risks and Mitigations

| Risk                                                                    | Impact                                                                                     | Mitigation                                                                                                                                                         |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `grill-with-docs` fork-on-import edits have a subtle bug                | First muni-infra session creates a stray `docs/adr/` directory or writes to the wrong file | Phase 4 is explicitly a validation pass — caught on one skill, not eight                                                                                           |
| GLOSSARY seed captures too much general-programming noise               | Glossary bloats with terms agents don't need; signal-to-noise drops                        | Phase 2 step 2 is human triage; the framework rule "only project-specific concepts" governs the cut                                                                |
| Teammates push back on `GLOSSARY.md` / `DECISION-REGISTER.md` appearing | Adoption friction; standards become contentious                                            | Both fit the existing `multibilling-docs` framework as new Standards — same friction as adding any other standard. Loop in the team in the PR description          |
| GLOSSARY.md never gets used because nothing reads it before Phase 3     | Wasted Phase 2 effort                                                                      | Phase 3 is gated to follow Phase 2 directly; intentional sequencing                                                                                                |
| Phase 6/7 batches expose more fork-on-import edits than expected        | Schedule slips on shared skills                                                            | Apply learnings from Phase 4 to harden the fork-edit recipe; each batch can pause for refinement                                                                   |
| AGENT_SYS-539 lands but introduces a regression elsewhere               | The CLI breaks for existing flows                                                          | Standard testing on the ticket itself (the Acceptance Criteria already covers regression)                                                                          |
| Backlog grows faster than triaging can keep up                          | The original problem reasserts                                                             | Phase 1 runs in parallel with Phase 2-5 work, not after; Step 3 of the triaging skill includes pattern-detection that should reduce upstream finding-creation rate |

---

## Related Documents

- [`docs/research/mattpocock-skills-adoption.md`](../research/mattpocock-skills-adoption.md) — the point-in-time evaluation that drove the adopt/skip/fork decisions feeding this plan
- [`docs/runbooks/sync-from-upstream-skills.md`](../runbooks/sync-from-upstream-skills.md) — the living runbook of per-skill fork-on-import decisions and framework decisions (GLOSSARY-as-standard, decisions-inline-with-register, no separate ADRs)
- AGENT_SYS-539 — the ticket-cli `--tag` / `--not-tag` filter feature; Phase 9 strips the jq workaround from `triaging-followups` once it lands
- `~/src/claude-config/skills/work/triaging-followups/SKILL.md` — already-deployed skill used in Phase 1
