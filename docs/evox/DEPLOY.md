# Evox Forms — Deployment Guide (Coolify)

This guide documents the steps to deploy the whitelabeled Formbricks on Coolify
using existing backup volumes. It assumes the backup volumes currently sit on
the operator's local Windows machine and must be transferred to the remote
Coolify host — that transfer is the part most likely to go wrong, so it gets
its own section (§0) before anything else.

Read `docs/evox/MIGRATION_NOTES.md` first — it documents findings from the
Lane 4 sandbox rehearsal (in particular, the MinIO credential gotcha in §3)
that this guide assumes you already know.

## 0. Get the backup volumes onto the Coolify host

Do this **before** creating any Coolify resources, so the volumes exist and
are populated the moment the containers first start.

1. Deploy the compose stack once with **empty** volumes first (skip to §1-§4
   below), so Coolify creates the Docker volumes and you learn their real
   names/paths on the host (Coolify usually prefixes volume names with the
   resource UUID — it will not be exactly `formbricks_pg` on disk even if
   that's the name inside the compose file).
2. **Stop** the Postgres, MinIO, and app containers in Coolify (stop, don't
   delete — deleting a resource in Coolify can delete its volumes too).
3. On the Coolify host, find the real volume paths:
   `docker volume inspect <volume-name> --format '{{ .Mountpoint }}'` for
   each of the three volumes.
4. From your Windows machine, copy each backup `_data` folder to the matching
   path on the host, e.g. with `scp -r` or `rsync` over SSH:
   ```
   scp -r "C:\...\evox_full_backup\var\lib\docker\volumes\formbricks_formbricks_pg\_data\." user@host:/tmp/restore_pg/
   ```
   then on the host, move it into the real mountpoint and fix ownership:
   ```
   sudo rsync -a /tmp/restore_pg/ <postgres-volume-mountpoint>/
   sudo chown -R 999:999 <postgres-volume-mountpoint>
   sudo chmod 700 <postgres-volume-mountpoint>
   ```
   Repeat for `formbricks_formbricks_minio_data` → the MinIO volume, and
   `formbricks_formbricks_uploads` → the uploads volume (no special
   ownership needed for the latter two beyond being readable by the
   container's runtime user).
5. Do not delete or modify the original backup folders on the Windows
   machine during this process — copy from them, never move.
6. Restart the containers in Coolify. This is the point where Prisma will run
   the pending migrations against the restored database — watch the app's
   deploy/runtime logs for migration errors before moving on.

## 1. Create Services in Coolify
In the Coolify dashboard, create the following services based on the provided `docker-compose.coolify.yml`:
1. Postgres (`formbricks-postgres`) - Image: `pgvector/pgvector:pg15`
2. Redis (`formbricks-redis`) - Image: `redis:alpine`
3. MinIO (`formbricks-minio`) - Image: `minio/minio:latest`

## 2. Set up Environment Variables
Populate the environment variables for these services based on the definitions in `env.template`.
**CRITICAL**: Do NOT regenerate `NEXTAUTH_SECRET`, `ENCRYPTION_KEY`, or `CRON_SECRET`. You MUST reuse the values from the old deployment to avoid losing access to encrypted data.
**Also critical (confirmed in Lane 4 — see MIGRATION_NOTES.md §3):** if you are
reusing the original `formbricks_minio_data` volume contents (which this
deployment does), `S3_ACCESS_KEY`/`S3_SECRET_KEY` must also match the
original MinIO root credentials exactly, not fresh ones — MinIO's root user
is persisted inside the volume itself, not read fresh from env vars once a
volume already has an identity.

Set every real value directly in Coolify's environment-variables UI for the
relevant service/application. Never put real values in a file inside this
git repository.

## 3. Attach Existing Volumes
Attach the real backup volumes (populated per §0) to the corresponding services:
- `formbricks_pg` for Postgres data
- `formbricks_uploads` for the App uploads
- `formbricks_minio_data` for MinIO storage

## 4. Deploy the Application
1. Create a new "Application" resource in Coolify.
2. Point it to this repository (`https://github.com/gabrielbertagnolli/evox-forms`) and branch (`evox-whitelabel`).
3. Set the build to use the Dockerfile at `apps/web/Dockerfile`.
4. Ensure all environment variables from `env.template` are added to the Application's configuration.
5. Deploy.

## 5. Domain and TLS Binding
Configure the domains in Coolify:
- Bind `form.evox.com.uy` to the Application with TLS enabled.
- Bind `files.evox.com.uy` to the MinIO service (if exposing MinIO directly) with TLS enabled.

## 6. Verification

Do not consider this deploy finished until you've completed
`docs/evox/PLAN.md` Lane 6's checklist against the real, live domains — not
just "the app booted." In particular: confirm a pre-existing survey and its
responses are visible, confirm an old uploaded file still opens, and confirm
the public survey pages show no Formbricks branding while the admin panel is
unaffected.
