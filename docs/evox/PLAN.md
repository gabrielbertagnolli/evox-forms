# Evox Forms — Implementation Plan (Formbricks Whitelabel)

This document is the single source of truth for finishing the Evox whitelabel
deployment of Formbricks. It is written for autonomous AI coding agents that
have **no memory of any prior conversation** about this project. Read this
entire file, then read `AGENTS_PROGRESS.md`, before touching anything.

If something in this plan seems ambiguous or you are tempted to improvise: **stop**.
Mark your lane `BLOCKED` in `AGENTS_PROGRESS.md` with the exact question, and do not
proceed past it. Do not guess. Do not "helpfully" expand scope.

## 0. Project summary (read first)

Evox is a marketing agency. They ran a self-hosted Formbricks instance
(`form.evox.com.uy`) and have backups of it (Postgres dump + Docker volumes).
The goal is:

1. Run this exact fork with all **respondent-facing** Formbricks branding
   removed (the admin panel keeps Formbricks branding — that's fine and
   intentional).
2. Deploy it on Coolify, restoring the old surveys/responses/files from backup.
3. Keep the fork easy to update when upstream Formbricks ships new releases.

Repo: `https://github.com/gabrielbertagnolli/evox-forms`
Working branch: **`evox-whitelabel`** — branch off upstream tag **`5.1.4`**,
which was chosen because it is the oldest stable tag whose migration set
already includes the old instance's last-applied migration
(`20260204083137_added_data_type_and_value_fields`, taken from the backup dump's
`_prisma_migrations` table). Do not rebase this branch onto `main` or any
other tag without going through Lane 7.

## 1. Golden rules (apply to every lane)

1. **Never touch admin-facing UI, copy, or branding.** Only respondent-facing
   surfaces (the public `/s/...` survey pages and the `@formbricks/surveys`
   package rendered inside them) are in scope for whitelabel work. If a lane's
   task requires touching anything under `apps/web/app/(app)/` or
   `apps/web/modules/.../settings/` for whitelabel purposes, that is out of
   scope — stop and flag it.
2. **Never commit real secrets.** No passwords, API keys, or `.env` files with
   live values go into git, ever — not even in a "temporary" commit. Templates
   with placeholder values are fine (see `docs/evox/env.template`).
3. **Never operate on original backup files.** Any time a lane touches the
   Postgres dump or the Docker volumes described in Lane 4/5, work on a **copy**.
   The originals are the only copy of Evox's real production data — there is no
   second backup.
4. **Never force-push.** Never rewrite history on `evox-whitelabel` once other
   agents may have based work on it. New work is new commits.
5. **One lane at a time, in order.** Lanes are numbered and sequential — each
   one lists its preconditions. Do not start lane N if lane N-1 is not marked
   `DONE` (or explicitly waived) in `AGENTS_PROGRESS.md`.
6. **Every lane ends with a log entry in `AGENTS_PROGRESS.md`**, using the exact
   template in that file. No exceptions, even if the lane failed.
7. **Every lane has a Review Gate.** A lane is not `DONE` until its Review Gate
   checklist has been run and recorded by either the same agent (self-review,
   for low-risk lanes) or a second agent/human (for the lanes marked
   "requires independent review" below). Do not mark a lane `DONE` without
   running its Review Gate.
8. If you are an AI agent picking up this project, your first action is always
   to open `AGENTS_PROGRESS.md`, read the status table, and claim a lane per
   the protocol described there — before reading further code.

## 2. Lane index

| Lane | Name | Requires independent review? | Depends on |
|---|---|---|---|
| 0 | Orientation | No | — |
| 1 | Whitelabel code changes | No (already done, audit only) | 0 |
| 2 | Build & local verification | No | 1 |
| 3 | Coolify deployment artifacts | No | 2 |
| 4 | Data migration rehearsal (sandbox) | **Yes** | 3 |
| 5 | Production deployment (Coolify) | **Yes** | 4 |
| 6 | Post-deploy verification & sign-off | **Yes** | 5 |
| 7 | Update / maintenance playbook | No | 6 |

---

## Lane 0 — Orientation

**Objective:** confirm you understand the repo state before changing anything.

**Steps:**

1. `git fetch --all --tags`
2. `git log --oneline -5 evox-whitelabel` — confirm the tip is a whitelabel or
   deployment-artifact commit, and that its ancestor is tag `5.1.4`
   (`git merge-base --is-ancestor 5.1.4 evox-whitelabel` should print nothing
   and exit 0).
3. Read `AGENTS.md` at repo root (repo-wide coding conventions — these still
   apply to any code you write in Lanes 2-3).
4. Read `docs/evox/PLAN.md` (this file) in full.
5. Read `AGENTS_PROGRESS.md` in full, including closed-out lanes, before
   claiming a new one.

**Review Gate:** self-review. Confirm items 1-2 above produced the expected
output before proceeding. Log a `DONE` entry even for this lane — it's cheap
and it proves the next agent that orientation actually happened.

---

## Lane 1 — Whitelabel code changes (COMPLETE — audit only, do not redo)

**Status as of this writing: DONE.** Commits `043e95f6` and `1f5368ec` on
`evox-whitelabel` already implement this. This section exists so any agent can
audit *why* the code looks the way it does, and so Lane 7 (future upstream
merges) knows exactly what to re-apply.

**Objective (already met):** remove every piece of Formbricks branding a
survey **respondent** (not an admin user) can see, without touching the admin
panel, and with a diff small enough to survive future `git merge` from
upstream.

**Respondent-facing branding surface, and how it was neutralized:**

| # | What a respondent sees | File | How it's neutralized |
|---|---|---|---|
| 1 | "Powered by Formbricks" watermark on every survey | `packages/surveys/src/components/general/formbricks-branding.tsx` | Component body replaced with `return null;` |
| 2 | Formbricks logo in the survey preloader | `apps/web/modules/survey/link/components/survey-loading-animation.tsx` (unchanged file — reads `isBrandingEnabled` prop, which traces back to `workspace.linkSurveyBranding`) | Fed `false` from the loader, see #5 |
| 3 | Formbricks logo + link on the "survey already answered" screen | `apps/web/modules/survey/link/components/survey-completed-message.tsx` | Guard changed from `(!workspace \|\| workspace.linkSurveyBranding)` to `Boolean(workspace?.linkSurveyBranding)` — i.e. default to hidden, not shown, when no workspace data is available |
| 4 | Formbricks logo + "Create your own survey" CTA on paused/expired/invalid-link screens | `apps/web/modules/survey/link/components/survey-inactive.tsx` | Same guard fix as #3, plus `showCTA` hardcoded to `const showCTA = false;` |
| 5 | (root cause of #2-4) `workspace.linkSurveyBranding` read from the DB, which respects the org's real (often EE-gated) setting | `apps/web/modules/survey/link/lib/workspace.ts` — both `getWorkspaceContextForLinkSurvey` and `getWorkspaceById` | Both functions force `linkSurveyBranding: false` on their return value, regardless of what's in the database |

**Why this approach and not something else:**

- The edits are scoped to files that are **only** imported by the public
  `/s/...` survey route tree (`apps/web/modules/survey/link/**`) and the
  standalone `@formbricks/surveys` package. Verified during Lane 1 by
  `grep -rln 'survey/link/lib/workspace"' apps/web` — only survey-link pages
  and their tests import it. The admin panel has its own, separate
  `linkSurveyBranding` toggle in project/workspace settings that is untouched
  and still fully functional (Evox staff can still see/use it internally; it
  simply no longer has any effect on what respondents see).
- Forcing the flag at the **data-loader** level (`workspace.ts`) rather than
  only patching each component was deliberate: it collapses 3 separate
  render-guard bugs (preloader, completed screen, inactive screen) into a
  single source of truth, and it is far less likely to be touched by upstream
  refactors than component internals.
- In self-hosted deployments, `IS_FORMBRICKS_CLOUD` is always `false`, which
  already suppresses the `| Formbricks` title suffix and the "report survey"
  footer link (see `apps/web/modules/survey/link/lib/metadata-utils.ts` and
  `apps/web/modules/survey/link/components/legal-footer.tsx`) — no changes
  were needed there.
- The default favicon (`apps/web/public/favicon.ico`) was deliberately **left
  alone**. It's shared between the admin panel and the public survey pages;
  per explicit instruction, admin-visible branding is out of scope, and
  changing the global favicon would touch the admin panel's tab icon too.
  If a future request specifically asks for a neutral favicon on `/s/...`
  pages only, that must be done via a page-level metadata override in
  `apps/web/modules/survey/link/lib/metadata-utils.ts` / `metadata.ts`
  (`icons` field), not by replacing the shared file.

**Review Gate (self-review, since this is audit-only):** re-read the diff with
`git show 043e95f6` and confirm it matches the table above exactly. If it
doesn't, something is wrong with the branch — stop and flag in
`AGENTS_PROGRESS.md`, do not silently "fix" the discrepancy.

---

## Lane 2 — Build & local verification

**Precondition:** Lane 1 is `DONE`.

**Objective:** prove the Lane 1 diff actually builds and behaves as intended —
this has **not** been done yet as of this writing. Nobody has run `pnpm build`,
`pnpm test`, or looked at a rendered page since the whitelabel edits landed.

**Known test breakage — fix these, do not work around them:**

`apps/web/modules/survey/link/lib/workspace.test.ts` has two tests that assert
the *old* (pass-through) behavior of `linkSurveyBranding` and will fail
against the Lane 1 code, which now always returns `false`:

1. Test `"should return workspace data when found"` (around line 49-63): the
   mock input `mockWorkspace` has `linkSurveyBranding: true` (line 53) — leave
   that alone, it's simulating what Prisma returns. But the assertion at line
   63, `expect(result).toEqual(mockWorkspace)`, will fail because the real
   function now overrides the field. Fix: change the assertion to
   `expect(result).toEqual({ ...mockWorkspace, linkSurveyBranding: false })`.
2. Test `"should successfully fetch workspace context with all required data"`
   (around line 115-193): the expected-result object passed to
   `expect(result).toEqual({...})` has `linkSurveyBranding: true` at line 152.
   Change that one occurrence to `false`. Do **not** touch the `mockData`
   input at line 122, and do **not** touch the `select: {...}` assertion at
   line 174 — those describe the Prisma call shape, not our override, and are
   correct as-is.
3. A third test, `"should handle workspace with minimal data"` (around line
   245-294), already expects `linkSurveyBranding: false` in both the mock
   input and the result. That one needs no change — it already happens to
   pass. Do not "fix" it.

Do not change `apps/web/modules/survey/link/metadata.test.ts` — it mocks
`getWorkspaceContextForLinkSurvey` as a dependency and never asserts on the
`linkSurveyBranding` value itself, so it is unaffected.

If `pnpm test` turns up *other* failing tests unrelated to the two above,
do not fix them as part of this lane unless they are a direct, obvious
consequence of the Lane 1 diff (i.e. they mention `linkSurveyBranding`,
`FormbricksBranding`, `showCTA`, or the `survey-inactive`/
`survey-completed-message` guard). If a failure looks pre-existing/unrelated,
record it as a note in `AGENTS_PROGRESS.md` and move on — do not chase
unrelated flakiness in this lane.

**Steps:**

1. `pnpm install` (workspace root).
2. `pnpm db:up` — brings up local Postgres/Redis dev containers (see
   `docker-compose.dev.yml`). This is a throwaway dev DB, not the Evox backup —
   do not confuse the two.
3. Fix the two test assertions above.
4. `rm -rf packages/surveys/dist apps/web/public/js/surveys.* node_modules/.cache/turbo`
   then `pnpm build --filter=@formbricks/surveys... --force` — required per
   `AGENTS.md`: the surveys package is pre-compiled and the Next.js app reads
   the built bundle, not the source.
5. `pnpm test` — must pass (module: `apps/web` and `packages/surveys` at
   minimum; see note above about pre-existing unrelated failures).
6. `pnpm build` — full workspace build must succeed with no errors.
7. `pnpm dev`, then manually, in a browser:
   - Create (or use a seeded/dev) survey of type "link", open its public URL
     (`/s/<surveyId>`). Confirm: no "Powered by Formbricks" text anywhere, no
     Formbricks logo during the loading spinner.
   - Trigger the "already answered" state (submit a single-use survey twice,
     or view a paused survey's link) and confirm no logo, no "Create your own
     survey" button.
   - Separately, log into the admin panel (`/environments/.../settings/...`)
     and confirm the existing Formbricks branding/naming in the admin UI is
     **unchanged** — this proves scope was respected, not just that branding
     disappeared.
8. `pnpm db:down` when finished, to avoid leaving a stray dev DB running.

**Deliverable:** a new commit on `evox-whitelabel` containing only the two
test-assertion fixes (nothing else — if `pnpm build`/`pnpm test` caused any
lockfile or generated-file diffs, review them individually before staging;
don't blanket `git add -A`).

**Review Gate (self-review is acceptable for this lane):**
- [ ] `pnpm test` exit code 0 (or only the pre-approved unrelated failures,
      explicitly listed in the `AGENTS_PROGRESS.md` entry)
- [ ] `pnpm build` exit code 0
- [ ] Manual browser check above completed and described in the log entry
      (what you saw, not just "looks good")
- [ ] Diff reviewed with `git diff` before commit — confirms no unrelated
      files were swept in

---

## Lane 3 — Coolify deployment artifacts

**Precondition:** Lane 2 is `DONE`.

**Objective:** produce the deployment configuration Coolify needs, without
touching Coolify itself yet (that's Lane 5). Everything in this lane is
git-tracked, reviewable, non-destructive documentation/config work.

**Context on the previous deployment** (from the old, non-git-tracked
`docker-compose.yml`/`.env` the operator supplied — summarized here, no
secrets included):

- App image was `ghcr.io/formbricks/formbricks:latest` (unpinned — this is
  *why* Lane 0's job of confirming the exact base tag mattered).
- Services: `formbricks-postgres` (`pgvector/pgvector:pg15`),
  `formbricks-redis` (`redis:alpine`), `formbricks-minio` (`minio/minio:latest`,
  **not** Cloudflare R2), and the `formbricks` app container, all on an
  external Traefik network (`traefikproxy`).
- Named volumes to preserve and re-attach: `formbricks_pg` (Postgres data),
  `formbricks_uploads` (app-side uploads mount), `formbricks_minio_data`
  (MinIO object storage — this is where actual survey file uploads live).
- Public domains: `form.evox.com.uy` (app), `files.evox.com.uy` (MinIO/S3
  public URL for serving uploaded files).
- `STORAGE_TYPE=s3`, MinIO in S3-compatible mode, `S3_FORCE_PATH_STYLE=1`.

**Steps:**

1. Create `docs/evox/docker-compose.coolify.yml`: a Coolify-compatible compose
   file modeled on the old one, but:
   - App image built from this repo/branch (Coolify builds it from
     `apps/web/Dockerfile` when you point a Coolify "Application" resource at
     this GitHub repo + `evox-whitelabel` branch — do not hardcode a
     `ghcr.io/formbricks/formbricks` image reference, we are running our own
     fork).
   - Keep the **exact same volume names** (`formbricks_pg`,
     `formbricks_uploads`, `formbricks_minio_data`) so the operator's existing
     backup volumes can be attached by name in Coolify without renaming
     anything.
   - Do not hardcode Traefik labels tied to the *old* server's Traefik
     instance — Coolify manages its own reverse proxy; domain/TLS binding
     happens in the Coolify UI (documented in Lane 5), not in this compose
     file. Leave a comment saying so where the old file had Traefik labels.
   - Reference all secrets as `${VARIABLE_NAME}` — no literal values, ever.
2. Create `docs/evox/env.template`: every environment variable the app needs,
   with a one-line comment on what it is and where its real value must come
   from (either "reuse from the old deployment's secret store" or "generate
   fresh"). No real values in this file — placeholders only (e.g.
   `CHANGE_ME_GENERATE_RANDOM`). At minimum, document:
   - `NEXTAUTH_SECRET`, `ENCRYPTION_KEY`, `CRON_SECRET` — **must be reused
     from the old deployment, not regenerated.** `ENCRYPTION_KEY` in
     particular encrypts data already sitting in the Postgres backup;
     regenerating it will make existing encrypted fields unreadable.
   - `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` — safe to regenerate
     (these are just DB-access credentials, not tied to encrypted data).
   - `S3_ACCESS_KEY`, `S3_SECRET_KEY` — safe to regenerate (MinIO
     credentials, not tied to encrypted data), but note that if regenerated,
     MinIO's own user/bucket policy must be updated to match on first boot,
     since the *data* in the MinIO volume doesn't care about the access key,
     only MinIO's internal user store does (which lives in the same
     `formbricks_minio_data` volume — see Lane 4 rehearsal for how this plays
     out in practice).
   - SMTP vars, `S3_BUCKET_NAME`, `S3_REGION`, `S3_FORCE_PATH_STYLE`,
     `S3_PUBLIC_URL`, `NEXTAUTH_URL`, `WEBAPP_URL` — reuse the old values
     (domains, bucket name) as-is; SMTP password should be rotated if
     convenient but is not blocking.
   - The actual filled-in `.env` file must **never** be committed. It lives
     either directly in Coolify's environment-variables UI, or in a local
     file outside the repo on the operator's machine. State this explicitly
     in the template's header comment.
3. Write `docs/evox/DEPLOY.md`: step-by-step instructions for a human
   operator to create the Coolify resources (Postgres, Redis, MinIO or S3-
   compatible storage, and the app itself pointed at this repo/branch),
   attach the three volumes, set env vars from `env.template`, and bind the
   two domains. This is documentation for a human — Lane 5 is where an agent
   (with the operator present, since production/DNS changes need human
   sign-off per this project's safety rules) actually executes it.

**Review Gate (self-review):**
- [ ] `docs/evox/docker-compose.coolify.yml` contains zero literal secret
      values (grep it for anything that looks like a password/key)
- [ ] `docs/evox/env.template` documents every var the app actually reads
      (cross-check against `apps/web/Dockerfile` and the app's own env
      validation, e.g. `apps/web/lib/env.ts` or similar, if present)
- [ ] `docs/evox/DEPLOY.md` steps are concrete enough that a human with no
      other context could follow them (no "configure appropriately")

---

## Lane 4 — Data migration rehearsal (sandbox)

**Precondition:** Lane 3 is `DONE`. **Requires independent review before
Lane 5 starts** — a second agent or the human operator must review this
lane's log entry and explicitly approve it in `AGENTS_PROGRESS.md`.

**Objective:** prove, on disposable copies, that the real backup restores
cleanly under Formbricks 5.1.4 *before* anyone touches the real production
volumes. This is the highest-risk lane in the whole plan if done carelessly —
follow the copy-first rule exactly.

**Inputs (operator-supplied, paths are illustrative — confirm the current
location with the operator, do not assume):**
- Postgres dump: a `.sql` file (last known good copy is dated 2026-06-18).
- Docker volumes: `formbricks_pg`, `formbricks_uploads`,
  `formbricks_minio_data` from the old server.

**Steps:**

1. **Copy, don't touch originals.** Make working copies of the dump file and
   the three volumes (e.g. `docker volume create --name evox_rehearsal_pg`
   then copy data across with a throwaway container, or restore the dump into
   a brand-new local Postgres container — never point anything at the
   original volume/file paths directly).
2. Stand up a local sandbox: Postgres 15 + pgvector, MinIO, Redis (see
   `docker-compose.dev.yml` for reference versions), using the **copies**.
3. Restore the copied dump into the sandbox Postgres:
   `psql <connection-string> < backup_copy.sql`.
4. Point a Formbricks 5.1.4 (or the `evox-whitelabel` branch's built image, if
   Lane 2's build artifact is available) container at the sandbox DB and
   MinIO. On boot, Prisma will run every migration between the dump's last
   recorded migration (`20260204083137_added_data_type_and_value_fields`) and
   5.1.4's newest migration automatically. **Let it run — do not manually
   edit `_prisma_migrations` or skip migrations.**
5. Confirm boot succeeds with no migration errors in the logs.
6. Log into the sandboxed app's admin panel. Confirm: existing
   organizations/users are visible, at least one pre-existing survey opens
   and shows its questions correctly, and at least one pre-existing response
   is visible in that survey's results.
7. Confirm file storage: find a survey response or upload that references a
   file, and confirm it's fetchable from the sandboxed MinIO (not a 404/403).
8. Tear down the sandbox. **Do not delete the copies you made in step 1** —
   keep them until Lane 6 sign-off, in case Lane 5 needs to re-run this
   rehearsal.

**Review Gate (independent — a second agent or the human operator must
confirm, not just the agent who ran it):**
- [ ] Migration run log shows every migration between the dump's baseline and
      5.1.4 applying with no errors (paste the relevant log excerpt into the
      `AGENTS_PROGRESS.md` entry, not just "it worked")
- [ ] At least one real (pre-existing) survey + response + uploaded file was
      independently confirmed visible/fetchable, by name/ID, in the log entry
      — not a generic "data looks fine"
- [ ] Confirm explicitly, in writing in the log entry, that no command in
      this lane wrote to the *original* dump file or the *original* volumes
      (only copies)

---

## Lane 5 — Production deployment (Coolify)

**Precondition:** Lane 4 is `DONE` and independently reviewed/approved.
**Requires independent review** (human sign-off mandatory — this lane touches
real production data, real DNS, and is hard to reverse).

**Objective:** deploy for real, using Lane 3's artifacts and Lane 4's proven
migration path, with the operator present for anything irreversible.

**Hard stop conditions — do not proceed past these without the human operator
explicitly confirming in the chat/session, even if you have general
permission to operate Coolify:**
- Attaching the **real** (not copied) `formbricks_pg`, `formbricks_uploads`,
  `formbricks_minio_data` volumes to a new Coolify service.
- Any DNS cutover for `form.evox.com.uy` / `files.evox.com.uy`.
- Anything that could overwrite the current production instance still
  serving traffic, if one still exists.

**Steps:**

1. In Coolify, create the Postgres, Redis, and MinIO/S3 services per
   `docs/evox/DEPLOY.md`, using **new** credentials from `env.template`
   (except `ENCRYPTION_KEY`/`NEXTAUTH_SECRET`/`CRON_SECRET`, which must be the
   operator's real old values — obtained out-of-band, never from this repo).
2. Attach the real backup volumes (after the human confirms this is the
   intended moment to do so).
3. Create the Coolify "Application" resource pointed at
   `https://github.com/gabrielbertagnolli/evox-forms`, branch
   `evox-whitelabel`, building from `apps/web/Dockerfile`.
4. Set all env vars per `env.template`.
5. Deploy. Watch the build/boot logs for migration errors — if any appear,
   **stop and do not retry blindly**; this is exactly what Lane 4 was
   supposed to have already ruled out, so a failure here means something
   differs between the rehearsal and production (different volume snapshot,
   different env var, etc.) — diagnose the difference before re-attempting.
6. Bind domains + TLS in Coolify for `form.evox.com.uy` and, if MinIO is
   exposed directly, `files.evox.com.uy`.
7. Do not tear down or decommission the old server/instance until Lane 6 is
   signed off.

**Review Gate (independent, human-inclusive):**
- [ ] Human operator explicitly confirmed each hard-stop item above, with a
      timestamp/note in the log entry
- [ ] Build + boot logs attached/summarized in the log entry, confirming no
      migration errors
- [ ] Both domains resolve and serve TLS correctly (paste the check command
      and output, e.g. `curl -I https://form.evox.com.uy`)

---

## Lane 6 — Post-deploy verification & sign-off

**Precondition:** Lane 5 is `DONE` and independently reviewed/approved.
**Requires independent review.**

**Objective:** prove the production deployment actually preserved the old
data and actually shows the whitelabel changes, end to end, on the real
domains — not the sandbox.

**Checklist (perform against the real `form.evox.com.uy`, log actual
observations, not assumptions):**
- [ ] Admin login works with a pre-existing user account.
- [ ] At least 2 pre-existing surveys are listed and open correctly.
- [ ] At least 1 pre-existing survey shows its previously-collected
      responses (not zero, unless the operator confirms the real survey
      genuinely had none).
- [ ] A file uploaded before the migration (attached to some response) opens
      via its public URL without error.
- [ ] A **new** test response can be submitted end-to-end through a public
      survey link (`https://form.evox.com.uy/s/...`), and it shows up in the
      admin's response list afterward.
- [ ] The public survey link shows **no** Formbricks branding: no "Powered by
      Formbricks" text, no logo in the preloader, no logo/CTA on a
      paused/completed/inactive survey screen.
- [ ] The admin panel still shows Formbricks naming/branding wherever it did
      before (org settings, "remove branding" toggle, etc.) — confirming
      scope was respected end-to-end, in production.
- [ ] Outbound email (password reset or a survey notification) sends
      successfully via the configured SMTP.

**Review Gate (independent, human-inclusive):** the human operator (or a
second agent acting on the operator's behalf) must tick each box above
personally, with what they observed, before this lane — and the whole
project — can be marked `DONE`.

---

## Lane 7 — Update / maintenance playbook

**Precondition:** Lane 6 is `DONE`.

**Objective:** document, once, how to pull a future Formbricks release into
this fork without losing the whitelabel changes, so nobody has to
re-derive Lane 1's reasoning from scratch.

**Steps:**

1. Write `docs/evox/UPGRADING.md` covering:
   - `git fetch origin --tags` (upstream Formbricks remote — add it if not
     already present: `git remote add origin https://github.com/formbricks/formbricks.git`
     if this clone doesn't have it).
   - Pick the new target tag (check its migration set is a superset of the
     currently-running instance's last-applied migration, the same way Lane 0
     picked `5.1.4` — don't just grab `main` or the newest tag blindly).
   - `git checkout -b evox-whitelabel-vNEXT <new-tag>`
   - `git cherry-pick 043e95f6 1f5368ec` (the Lane 1 commits) plus any Lane 2
     test-fix commit — resolve conflicts by re-reading the Lane 1 table
     above and re-applying the same *intent* (hide the same 5 surfaces) even
     if the exact code has moved.
   - Re-run all of Lane 2 (build + tests + manual browser check) against the
     new branch before it replaces `evox-whitelabel` as the deployed branch.
   - Re-run Lane 4's rehearsal if the new tag's migration set changed
     significantly, before touching production again.
2. This playbook doc itself does not need a Review Gate beyond a self-review
   that it's actually followable — but the *first real execution* of an
   upgrade should be treated as a fresh pass through Lanes 2, 4 (if needed),
   5, and 6.
