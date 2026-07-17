# Agents Progress Ledger — Evox Forms Whitelabel

This file is the shared, append-only work log for `docs/evox/PLAN.md`. Multiple
AI agents (and humans) may work on this project over time, in separate
sessions, with no shared memory except this file and the git history. Read
`docs/evox/PLAN.md` in full before reading further.

## How to use this file (protocol — read before editing)

1. **Claim before you work.** Before starting a lane, edit the status table
   below to set that lane's status to `IN_PROGRESS` and put your agent
   name/id in the "Owner" column. Commit **only that table edit** first, by
   itself, and push it, before doing any other work. This is how two agents
   avoid starting the same lane at the same time.
   - If you `git pull` and find a lane you were about to start already shows
     `IN_PROGRESS` with a recent timestamp, do not start it — pick the next
     unclaimed lane, or help review a lane that's `IN_REVIEW`.
2. **Log entries are append-only.** Never edit or delete another agent's log
   entry. If something in an old entry turns out to be wrong, add a new entry
   that says so — don't rewrite history.
3. **Every log entry uses the template in §3, verbatim structure.** Fill in
   every field. "N/A" is an acceptable value; a missing field is not.
4. **Status values:** `NOT_STARTED`, `IN_PROGRESS`, `BLOCKED`, `IN_REVIEW`,
   `DONE`. A lane goes `IN_PROGRESS` → `IN_REVIEW` when its own work is
   finished but its Review Gate (see `PLAN.md`) hasn't run yet, then → `DONE`
   once the Review Gate passes. Lanes marked "requires independent review" in
   `PLAN.md` must not go straight from `IN_PROGRESS` to `DONE`.
5. **If blocked, say exactly why**, in the log entry, with status `BLOCKED`.
   Do not silently work around a blocker by improvising outside the lane's
   documented scope — escalate instead (leave it blocked for a human or
   another agent to unblock).
6. Update the status table and append your log entry **in the same commit**
   whenever possible, so the table and the detailed history never drift out
   of sync.

## 1. Status table

| Lane | Name | Status | Owner | Last updated (UTC) | Notes |
|---|---|---|---|---|---|
| 0 | Orientation | DONE | Codex (GPT-5) | 2026-07-17 | Local orientation complete; remote sync explicitly delegated to another agent |
| 1 | Whitelabel code changes | DONE | claude (chat session, 2026-07-16/17) | 2026-07-17 | Pre-dates this ledger; see §2 entry below for retroactive record |
| 2 | Build & local verification | IN_PROGRESS | Codex (GPT-5) | 2026-07-17 | Local build and verification in progress |
| 3 | Coolify deployment artifacts | NOT_STARTED | — | — | |
| 4 | Data migration rehearsal (sandbox) | NOT_STARTED | — | — | Requires independent review before Lane 5 |
| 5 | Production deployment (Coolify) | NOT_STARTED | — | — | Requires human sign-off — see PLAN.md hard-stop list |
| 6 | Post-deploy verification & sign-off | NOT_STARTED | — | — | Requires independent review |
| 7 | Update / maintenance playbook | NOT_STARTED | — | — | |

## 2. Log entries

### Lane 1 — Whitelabel code changes — DONE (retroactive entry)

- **Agent/session:** Claude (Sonnet 5), interactive chat session with the
  Evox operator, 2026-07-16/17. This entry is written retroactively because
  this ledger didn't exist yet when the work happened — from this point
  forward, all lanes must have a real-time entry, not a retroactive one.
- **What was done:** Cloned upstream Formbricks, determined the correct base
  version by cross-referencing the operator's Postgres backup's
  `_prisma_migrations` table against upstream tags (landed on tag `5.1.4`),
  created branch `evox-whitelabel` from that tag, and applied the 4-file diff
  described in `docs/evox/PLAN.md` Lane 1. Also removed `.github/workflows/`
  (commit `1f5368ec`) because the operator's GitHub PAT lacked the `workflow`
  scope needed to push those files, and they're not needed for a self-hosted
  fork anyway.
- **Commits:** `043e95f6` (whitelabel diff), `1f5368ec` (remove CI workflows).
  Both pushed to `evox-whitelabel` on
  `https://github.com/gabrielbertagnolli/evox-forms`.
- **Verification performed:** static analysis only (grep for every
  `linkSurveyBranding`/`FormbricksBranding`/`footerLogo`/`formbricks.com`
  occurrence reachable from the public survey pages, confirmed each is now
  neutralized or was already dead in self-hosted mode). **No build, no test
  run, no browser check was done.** That is exactly Lane 2's job — do not
  assume Lane 1's code works until Lane 2 says so.
- **Known follow-up required:** two assertions in
  `apps/web/modules/survey/link/lib/workspace.test.ts` will fail against this
  diff (see `docs/evox/PLAN.md` Lane 2 for the exact line numbers and fix).
  This was identified during Lane 1 but deliberately left for Lane 2 to fix,
  so that "make it build and pass tests" stays a single reviewable unit of
  work.
- **Review Gate:** self-review only (per PLAN.md, Lane 1 doesn't require
  independent review). Re-checked the diff against the intended scope table
  in PLAN.md immediately before writing this entry; matches.
- **Status:** DONE.

<!--
Template for every future entry — copy this block, fill it in, append below
this comment (do not insert above it, keep entries in chronological order):

### Lane <N> — <Name> — <STATUS>

- **Agent/session:** <who/what ran this, and when>
- **What was done:** <concrete summary — files touched, commands run>
- **Commands run and key output:** <paste the actually-relevant output, not
  just "ran successfully" — e.g. test pass counts, migration names applied,
  curl status codes>
- **Commits:** <hashes + one-line description, or "none yet">
- **Deviations from PLAN.md, if any:** <what, and why — if you deviated
  without a documented reason, that's a problem, not a footnote>
- **Blockers (if status is BLOCKED):** <exact question/obstacle, and what you
  need from a human or another agent to unblock>
- **Review Gate:** <for lanes NOT requiring independent review: what you
  checked, self-attested. For lanes requiring independent review: leave this
  as "PENDING INDEPENDENT REVIEW" until a second agent/human fills in the
  reviewer sub-block below>
  - **Reviewer (independent-review lanes only):** <who, when>
  - **Reviewer's findings:** <what they checked, what they found, approve or
    reject with reasons>
- **Status:** <NOT_STARTED | IN_PROGRESS | BLOCKED | IN_REVIEW | DONE>
-->

### Lane 0 — Orientation — DONE

- **Agent/session:** Codex (GPT-5), local workspace session, 2026-07-17.
- **What was done:** Read `docs/evox/PLAN.md`, `AGENTS_PROGRESS.md`, and the repository-root `AGENTS.md`; verified the local `evox-whitelabel` history and its tag ancestry.
- **Commands run and key output:** `git log --oneline -5 evox-whitelabel` showed `d48fba11`, `3b27794c`, `1f5368ec`, and `043e95f6`; `git merge-base --is-ancestor 5.1.4 evox-whitelabel` exited 0.
- **Commits:** `d48fba11` — claim Lane 0; this commit closes the lane locally.
- **Deviations from PLAN.md, if any:** Did not run `git fetch --all --tags` and did not push the claim commit. The operator explicitly directed this agent to work only with existing local files; remote synchronization is delegated to another agent.
- **Blockers (if status is BLOCKED):** N/A.
- **Review Gate:** Self-review passed: local history is consistent with the documented whitelabel commits and tag `5.1.4` is an ancestor of `evox-whitelabel`.
  - **Reviewer (independent-review lanes only):** N/A.
  - **Reviewer's findings:** N/A.
- **Status:** DONE.