# Sync From Upstream Skills

> Use this runbook every time the `mattpocock-skills` fork pulls new commits from upstream, to decide what to promote into `~/src/claude-config` and to remember every fork-on-import decision so we don't re-derive them.

## When to Use This

- Right after `git fetch upstream && git merge upstream/main` on the fork
- Before importing a new upstream skill into `~/src/claude-config`
- When `just sync-upstream` flags drift on an already-adopted skill
- When an upstream skill we previously skipped looks worth reconsidering

## Prerequisites

- Fork at `~/src/mattpocock-skills` with `upstream` remote configured (`git remote add upstream https://github.com/mattpocock/skills.git`)
- `~/src/claude-config` checked out and clean
- `just sync-upstream` recipe in `claude-config` (build target — see [Open Procedural Items](#open-procedural-items))

## Sync Procedure

### 1. Refresh the fork from upstream

```bash
cd ~/src/mattpocock-skills
git fetch upstream
git merge upstream/main          # or rebase if the fork has no local commits worth preserving
```

### 2. Run the drift digest

```bash
cd ~/src/claude-config
just sync-upstream
```

Expected output: one line per adopted skill, either `up to date` or `N commits,
M files changed since <sha>`.

### 3. For each skill with drift, decide

For each skill flagged, open this runbook to its [Per-Skill Decisions](#per-skill-decisions) entry and re-read the prior reasoning. Then:

| Drift type                                      | Action                                                                                                                                        |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Upstream renamed or restructured the skill      | Read the upstream diff. If our fork-on-import edits still apply, port them forward. If the restructuring obsoletes them, update this runbook. |
| Upstream added new content to a section we kept | Merge in unless it conflicts with the fork-on-import edits                                                                                    |
| Upstream changed a section we replaced          | Skip the upstream change. Note in this runbook that we considered and skipped it (with the upstream sha).                                     |
| Upstream deprecated the skill                   | Move our adopted copy to `claude-config/skills/deprecated/` and update this runbook                                                           |

### 4. Apply changes in claude-config

```bash
# Edit the adopted skill in place
cd ~/src/claude-config
# ...edit ~/src/claude-config/skills/<bucket>/<skill>/SKILL.md...

# Update the .upstream stamp to the new upstream sha
echo "mattpocock/skills@<new-sha> path=skills/<area>/<skill>" > skills/<bucket>/<skill>/.upstream

# Deploy and commit
just sync
git add skills/<bucket>/<skill>/
git commit -m "sync(<skill>): pull upstream changes through <new-sha>"
```

### 5. Update this runbook

Append a dated note to the affected skill's entry below. Keep the prior notes —
the history is the point.

---

## Per-Skill Decisions

The status column means:

- **Adopted** — copied into `~/src/claude-config` and live
- **Adopted (forked)** — copied with non-trivial edits at import; see "Fork-on-import edits"
- **Pending** — decision direction set in [research doc][research], not yet imported
- **Skipped** — evaluated and rejected for reasons listed
- **Open** — still under discussion

### `grill-with-docs` — Pending (forked)

> Stress-tests a draft plan against the existing domain model, sharpens terminology, captures decisions inline.

**Target bucket:** `claude-config/skills/shared/`

**Role in workflow:** Used as a second pass after `superpowers:brainstorming`
produces a plan file. Brainstorming handles the front-end "what should we
build?" phase; `grill-with-docs` handles the "poke holes in this draft plan"
phase. Both stay in use.

**Fork-on-import edits:**

1. **Glossary file location** — Replace upstream's "`CONTEXT.md` at repo root" with detection logic:
   - If `docs/standards/` exists (MultiBilling framework signal), target `docs/standards/GLOSSARY.md`.
   - Otherwise (personal repos), fall through to upstream's `CONTEXT.md` at repo root.
2. **ADR side effect** — When a decision crystallizes meeting the "all three" criteria (hard to reverse + surprising + real trade-off):
   - If `docs/standards/` exists, prompt the user: *"This decision is worth recording. Should it (a) land in an existing standard's Decision section, (b) become a new standard, or (c) skip for now?"* Never create `docs/adr/` in a muni repo.
   - After the user confirms (a) or (b), the skill MUST also append a one-row entry to `docs/standards/DECISION-REGISTER.md` (date, one-line summary, link to the standard that holds the full decision). See [Framework Decisions](#framework-decisions) for why this register exists.
   - Otherwise (personal repo, no `docs/standards/`), fall through to upstream's `docs/adr/<NNNN>-slug.md` creation.
3. **Drop the `CONTEXT-MAP.md` multi-context branch.** Muni repos aren't structured that way and the complexity buys nothing.

**Why:** See [research doc § Q4][research], the in-conversation discussion about
CONTEXT.md and ADR conflicts with the `multibilling-docs` framework, and
[Framework Decisions](#framework-decisions) below.

**Open decisions:** Glossary filename (`GLOSSARY.md` vs `VOCABULARY.md` vs
`TERMS.md`) — TBD before import.

### `grill-me` — Skipped

> Lightweight grilling interview, no doc side effects.

**Reason for skip:** Upstream is deprecating in favor of `grill-with-docs`.
Importing it would create immediate drift work for no benefit.

**Revisit if:** Upstream un-deprecates, or if `grill-with-docs` proves too heavy
for quick sanity-checks.

### `handoff` — Skipped

> Compacts the current conversation into a handoff doc.

**Reason for skip:** Superseded by
`claude-config/skills/work/handing-off-session/`, which is richer (git branch
tracking, structured Current Focus, `.claude/handoffs/` convention).

**Revisit if:** Upstream evolves it past parity with `handing-off-session`.

### `write-a-skill` — Skipped

> Meta-skill for authoring new skills.

**Reason for skip:** Superseded by `claude-config/skills/shared/naming-skills/`
plus the `superpowers:writing-skills` plugin.

### `caveman`, `zoom-out`, `prototype`, `tdd` — Pending (clean adopt)

**Target bucket:** `claude-config/skills/shared/`

**Fork-on-import edits:** None expected for `caveman` and `zoom-out`.
`prototype` and `tdd` reference `CONTEXT.md` for vocabulary, but degrade
gracefully when absent — leave as-is.

### `diagnose`, `improve-codebase-architecture` — Pending (forked)

**Target bucket:** `claude-config/skills/shared/`

**Fork-on-import edits:**

1. Same glossary-file detection logic as `grill-with-docs` — read `docs/standards/GLOSSARY.md` in muni repos, `CONTEXT.md` elsewhere.
2. `improve-codebase-architecture` may offer to create ADRs when a refactor candidate is rejected with a load-bearing reason. Apply the same redirect-to-standards + register-append logic as `grill-with-docs`. Never create `docs/adr/` in a muni repo.

### `setup-matt-pocock-skills` — Pending (forked, personal-only)

**Target bucket:** `claude-config/skills/personal/`

**Fork-on-import edits:**

1. Rename to drop "matt-pocock" branding. Candidates: `bootstrapping-personal-repo`, `setting-up-skill-config`. **Decide at import.**
2. Remove the GitLab and "Other" issue-tracker branches — keep only GitHub (personal use case).
3. Mark explicitly as personal-only in the skill description so it doesn't get invoked in muni repos.

### `to-prd`, `to-issues` — Pending (personal-only, no fork)

**Target bucket:** `claude-config/skills/personal/`

**Reason no fork needed:** These skills publish to GitHub issues, which is the
right behavior for personal repos. Work-side equivalent already exists
(`writing-plan-tickets` + MultiBilling Plans).

### `triage` — Personal copy pending, work side replaced by net-new skill

**Personal copy:**

- **Target bucket:** `claude-config/skills/personal/`
- **Fork-on-import edits:** None — runs against GitHub issues as upstream intends.
- **Status:** Pending import.

**Work side — `triaging-followups` (NOT a fork of Pocock's `triage`):**

- **Location:** `~/src/claude-config/skills/work/triaging-followups/SKILL.md`
- **Status:** Drafted 2026-05-26.
- **Why net-new instead of a fork:** Pocock's triage assumes a
  maintainer-vs-reporter split (his state machine includes `needs-info`
  for waiting on the reporter). Jamie's situation is symmetric —
  `orchestrating-batches`
  generates the follow-up tickets, Jamie is both reporter and maintainer.
  The state machine doesn't translate, so the work-side skill is a fresh
  design that borrows Pocock's process discipline (per-ticket triage notes,
  resume-aware queries, read-the-code-before-deciding) but uses
  ticket-cli primitives (Cancelled status, batches, priority) instead of
  GitHub-style labels.
- **What carries over from Pocock:** "Show what needs attention" entry
  point; reproduce-before-deciding step (adapted to "read the PR diff");
  triage-notes-as-comments discipline; resume via tag query so the same
  ticket isn't re-triaged.
- **Net-new shape:** Five triage outcomes (Cancel / Promote / Defer /
  Bundle / Schedule), four namespaced tags on top of the existing
  `infra:follow-up` (`infra:triaged`, `infra:wontfix`, `infra:promoted`,
  `infra:deferred`), plus a Pattern Report step that feeds findings back
  into `orchestrating-batches` (cancellation-rate signals, recurring
  findings → standards proposals).
- **Open items the skill calls out:**
  - `ticket-cli list` has no `--tag` / `--not-tag` flags — needs jq
    workaround until a CLI feature lands. **Effort assessed 2026-05-26:
    Small** (~30-60 min). The backend CRUD layer
    (`agent-system/backend/app/crud/ticket.py:227-233`) already accepts
    `tag_ids: list[UUID]` and applies AND-logic via subqueries. Only
    three places need touching:
    1. MCP tool schema in `internal_mcp_server.py:329` (ticket_list)
       and `:626` (ticket_search) — add `tag_names` and
       `not_tag_names` inputs (~10 lines each).
    2. MCP handler — resolve names → UUIDs via `get_tag_by_name`,
       forward to `get_tickets(tag_ids=..., not_tag_ids=...)` (~8 lines).
    3. CLI argparse + `cmd_list` / `cmd_search` in `bin/ticket-cli`
       (lines 836, 1181, 3884, 3923) — add the flags and pass through
       (~6 lines per command).
    NOT-tag support requires one extra CRUD parameter (~5 lines) since
    the existing implementation only does AND, not exclusion. Recommend
    landing both flags in one PR — eliminates the jq workaround from
    the triaging skill entirely.
    **Filed as `AGENT_SYS-539` on 2026-05-26.** Draft body retained
    below in [Tag-Filter Ticket Draft](#tag-filter-ticket-draft) for
    reference.
  - `ticket-cli batch-add` semantics need verification (does adding to a
    new batch remove from the old?).

---

## Adding a New Upstream Skill

When upstream adds a skill we haven't evaluated:

1. Read the new SKILL.md in `~/src/mattpocock-skills/skills/<area>/<skill>/`
2. Check the four questions from the [research doc][research]:
   - Does it depend on the issue tracker? If so, GitHub-only (→ personal) or reshapeable to ticket-cli (→ work fork)?
   - Does it write files to the consuming repo? If so, do those files conflict with the muni framework?
   - Does it overlap with an existing claude-config skill? If yes, lean toward skip.
   - Is it tracker-agnostic and conflict-free? If so, candidate for `shared/`.
3. Add a new entry to [Per-Skill Decisions](#per-skill-decisions) — status `Open` — and capture the answers
4. Import (or skip) based on the answers
5. Update the entry status

## Framework Decisions

Decisions made about how the imported skills integrate with the existing
`multibilling-docs` framework. These bind every fork-on-import edit that
touches a muni repo's docs directory.

### Domain glossary lives at `docs/standards/GLOSSARY.md`, not `CONTEXT.md` at repo root

**Why:** The `multibilling-docs` framework already organizes prescriptive docs
under `docs/standards/`. A glossary is prescriptive ("use this term, not that
one"), so it slots in as a standard rather than introducing a new top-level
convention. Avoids confusion next to `CLAUDE.md` and keeps teammate friction
near zero — teammates already know where standards live.

**Applies to:** `grill-with-docs`, `diagnose`, `improve-codebase-architecture`,
and any future imported skill that reads or writes domain vocabulary.

### Decisions land in standards inline, with one-row entries in `DECISION-REGISTER.md`

The `multibilling-docs` framework explicitly rejects separate ADRs (see
`multibilling-infra/docs/standards/DOCUMENTATION.md`) because the Standard
template already has `## Context / ## Decision / ## Rationale` sections — a
separate ADR file would duplicate that content. The stated reason is team size
and duplication overhead for teams under 5 engineers.

We keep that rule. Decisions captured by imported skills (e.g. when
`grill-with-docs` recognizes one) flow into the relevant standard, not into
`docs/adr/`.

To preserve a single browsable view of every decision made — the genuine
benefit a flat `docs/adr/` directory would have given us — we maintain
**`docs/standards/DECISION-REGISTER.md`**: a one-row-per-decision index that
links to the standard holding the full decision text. The register is purely
an index; the standard is the source of truth.

Reconsider this rule if any of these happen:

1. Team grows past 5 engineers and parallel architecture decisions become common
2. Any single standard accumulates more than 3 inline decisions and bloats
3. Decisions start getting reversed/superseded frequently and the supersedes-chain becomes hard to follow via git history alone

### Standards rejection for forced-on-import edits is teammate-impact, not purism

When deciding whether to fork an upstream skill on import, the operative
question is "would this skill write files into a shared muni repo in a way
teammates would find confusing?" If yes, fork to gate or redirect. If no
(personal repo only, or writes only to `$TMPDIR`), no fork needed.

## Open Procedural Items

These are infrastructure pieces the runbook depends on but that don't exist yet.
Build them before the first real sync session.

- `just sync-upstream` recipe in `~/src/claude-config/justfile` — walks every `.upstream` stamp file and prints per-skill drift
- `.upstream` stamp file written into each adopted skill's directory
- `claude-config/skills/deprecated/` bucket — already in convention but not yet used
- `multibilling-infra/docs/standards/DECISION-REGISTER.md` — create lazily on first import of `grill-with-docs` (or sooner if a decision needs recording before then). Template: a single markdown table with `Date | Decision | Standard` columns and a one-paragraph header explaining what the register is.
- `multibilling-infra/docs/standards/GLOSSARY.md` — create lazily on first term resolved during a `grill-with-docs` session. Format derived from upstream `CONTEXT-FORMAT.md` but skinned as a Standard (metadata block, Context, Decision, Best Practices, etc.).

## Tag-Filter Ticket Draft

Draft body for the `ticket-cli` tag-filter ticket. File against the
`AGENT_SYS` project as a Feature. Has not been filed yet — confirm
before running `ticket-cli create`.

**Title:** `Add --tag and --not-tag filters to ticket-cli list and search`

**Type:** Feature

**Project:** `AGENT_SYS`

**Assignee:** Jamie (`--assign 2`)

**Priority:** default (5) — useful quality-of-life but not blocking

**Body:**

```markdown
## Summary

Add `--tag <name>` (AND filter) and `--not-tag <name>` (exclusion
filter) flags to `ticket-cli list` and `ticket-cli search`. Both flags
repeatable.

## Motivation

The `triaging-followups` workflow needs a primary query of "tickets
tagged `infra:follow-up` AND NOT tagged `infra:triaged`" to find the
untriaged backlog. The backend already supports tag-AND filtering at
the CRUD layer — only the MCP tool schema and CLI argparse need to
forward it. The exclusion case (`--not-tag`) requires one small CRUD
addition. Without these flags, every backlog query runs through a
brittle `--json | jq` pipe that's hard to script around and fragile to
schema drift.

This unblocks the triaging skill's core workflow and is broadly useful
for any future skill that needs tag-scoped queries.

## Context

Pointers verified 2026-05-26 on `agent-system` main:

- `bin/ticket-cli:836` — `cmd_list` builds the params dict for the
  `ticket_list` MCP call. No tag handling.
- `bin/ticket-cli:1181` — `cmd_search` does the same for
  `ticket_search`. No tag handling.
- `bin/ticket-cli:3884` — argparse subparser for `list`. Has
  `--project`, `--status`, `--type`, `--assigned`, `--queue`, `--batch`,
  `--all`, `--limit`, `--search`, `--updated-since`, `--updated-before`,
  `--json`. No tag flags.
- `bin/ticket-cli:3923` — argparse subparser for `search`. Same gap.
- `backend/app/services/internal_mcp_server.py:329` — `ticket_list`
  tool input schema. No `tag_names` / `tag_ids` properties.
- `backend/app/services/internal_mcp_server.py:626` — `ticket_search`
  tool input schema. Same gap.
- `backend/app/crud/ticket.py:111` — `get_tickets(tag_ids: list[UUID]
  | None = None, ...)` parameter exists already.
- `backend/app/crud/ticket.py:227-233` — applies tag-AND filter:

  ```python
  if tag_ids and len(tag_ids) > 0:
      for tag_id in tag_ids:
          subquery = select(TicketTag.ticket_id).where(TicketTag.tag_id == tag_id)
          conditions.append(Ticket.id.in_(subquery))
  ```

  No `not_tag_ids` parameter yet — needs to be added for `--not-tag`.

Tags themselves are first-class entities (`Tag` model with `id` + `name`,
`TicketTag` join table) per `backend/app/schemas/ticket.py:317-346`.

## Tasks

1. Add `not_tag_ids: list[UUID] | None = None` parameter to
   `get_tickets` in `crud/ticket.py`. After the existing tag-AND block,
   apply exclusion via `Ticket.id.not_in(subquery)` for each
   `not_tag_id`.

2. Add the same `tag_ids` / `not_tag_ids` filter to `search_tickets` in
   `crud/ticket.py` (or whichever function backs `ticket_search` — if
   it doesn't already accept the params, mirror the implementation
   from `get_tickets`).

3. Add `tag_names` and `not_tag_names` to the `ticket_list` input
   schema in `internal_mcp_server.py:329`. Mirror the existing
   `status_name` resolve-server-side pattern.

4. In the MCP handler for `ticket_list`: resolve each tag name to a
   UUID via `tag_crud.get_tag_by_name`. If any name fails to resolve,
   return a clear error (don't silently filter). Forward as
   `tag_ids=...` and `not_tag_ids=...` to `get_tickets`.

5. Repeat steps 3-4 for `ticket_search` (schema at line 626).

6. CLI: add `s.add_argument("--tag", action="append", help="Filter
   tickets with this tag (repeatable, AND logic)")` and `s.add_argument(
   "--not-tag", action="append", dest="not_tag", help="Exclude
   tickets with this tag (repeatable)")` to both the `list` (line 3884)
   and `search` (line 3923) subparsers.

7. CLI: in `cmd_list` (line 836) and `cmd_search` (line 1181), pass the
   parsed flags through as `tag_names=...` / `not_tag_names=...` in
   the params dict sent to `mcp_call`.

8. Update `ticket-cli list --help` and `search --help` examples in any
   docs that show common queries (probably `docs/` in the repo).

## Constraints

- Don't break existing callers. All new params are optional.
- The CLI is stdlib-only — keep it that way. No new dependencies.
- Server-side name → UUID resolution must be case-sensitive (tags
  like `infra:follow-up` vs `Infra:Follow-Up` are deliberately
  distinct strings).
- AND-only semantics for multi-`--tag`: passing `--tag a --tag b`
  returns tickets tagged with BOTH. This matches the existing CRUD
  behavior; document it in `--help`.

## Acceptance Criteria

- [ ] `ticket-cli list --tag infra:follow-up` returns tickets carrying that tag
- [ ] `ticket-cli list --tag infra:follow-up --not-tag infra:triaged` returns the untriaged backlog
- [ ] `ticket-cli list --tag a --tag b` returns tickets carrying BOTH a and b (AND semantics)
- [ ] `ticket-cli search "<query>" --tag infra:follow-up` works the same way for full-text search
- [ ] `ticket-cli list --tag does-not-exist` returns a clear error, not a silent empty result
- [ ] `--help` output documents both flags with examples
- [ ] No existing test regresses (`pytest backend/tests`)
- [ ] New tests cover: single-tag AND, multi-tag AND, NOT-tag exclusion, combined AND+NOT, non-existent tag name → error

```

## Related Documentation

- [`docs/research/mattpocock-skills-adoption.md`][research] — the point-in-time evaluation that drove the initial adopt/skip/fork decisions
- Upstream source: `~/src/mattpocock-skills/` (this repo)
- Destination: `~/src/claude-config/skills/`
- Ticket source files: `~/src/muni/agent-system/` (per the Tag-Filter Ticket Draft)

[research]: ../research/mattpocock-skills-adoption.md
[research-triage]:
../research/mattpocock-skills-adoption.md#triage--multibilling-equivalent
