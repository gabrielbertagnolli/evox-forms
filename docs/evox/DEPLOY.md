# Evox Forms — Deployment Guide (Coolify)

This guide documents the steps to deploy the whitelabeled Formbricks on Coolify using existing backup volumes.

## 1. Create Services in Coolify
In the Coolify dashboard, create the following services based on the provided `docker-compose.coolify.yml`:
1. Postgres (`formbricks-postgres`) - Image: `pgvector/pgvector:pg15`
2. Redis (`formbricks-redis`) - Image: `redis:alpine`
3. MinIO (`formbricks-minio`) - Image: `minio/minio:latest`

## 2. Set up Environment Variables
Populate the environment variables for these services based on the definitions in `env.template`.
**CRITICAL**: Do NOT regenerate `NEXTAUTH_SECRET`, `ENCRYPTION_KEY`, or `CRON_SECRET`. You MUST reuse the values from the old deployment to avoid losing access to encrypted data.

## 3. Attach Existing Volumes
Attach the real backup volumes to the corresponding services:
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
