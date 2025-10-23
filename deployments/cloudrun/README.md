# Deploy to Google Cloud Run (low-cost setup)

This guide pairs with `.github/workflows/deploy-cloudrun.yml` to deploy Plane API and Web to Cloud Run with Supabase (Postgres + S3-compatible storage), optional Celery worker, and Upstash Redis for cache. It targets the cheapest reliable setup for tiny usage and scales to zero.

## 1) Required GitHub Actions secrets
Set these in GitHub → Settings → Secrets and variables → Actions.

GCP + Container registry
- GCP_SA_KEY: Service Account JSON with roles: Artifact Registry Writer, Cloud Run Admin, Service Account User
- GCP_PROJECT_ID: your project id (e.g., my-project)
- GCP_REGION: Cloud Run region (e.g., us-central1)
- GAR_LOCATION: Artifact Registry location (e.g., us)
- GAR_REPOSITORY: Artifact Registry repo name (e.g., plane)

Supabase (DB + Storage)
- SUPABASE_DB_URL: Postgres connection string
- SUPABASE_S3_ENDPOINT_URL: S3 endpoint, e.g. https://<project-ref>.supabase.co/storage/v1/s3
- SUPABASE_S3_ACCESS_KEY: S3 access key
- SUPABASE_S3_SECRET_KEY: S3 secret key
- SUPABASE_BUCKET: Bucket name you created (e.g., uploads)

App config (recommended)
- SECRET_KEY: Django secret key

Optional (recommended for features/cheapest ops)
- REDIS_URL: Upstash Redis URL (rediss://...)
- AMQP_URL: Celery broker URL (CloudAMQP free tier). Required if you deploy the worker.
- CORS_ALLOWED_ORIGINS: Comma-separated list (e.g., https://<web-run-url>,https://<api-run-url>)

The workflow injects these as:
- DATABASE_URL ← SUPABASE_DB_URL
- AWS_S3_ENDPOINT_URL ← SUPABASE_S3_ENDPOINT_URL
- AWS_ACCESS_KEY_ID ← SUPABASE_S3_ACCESS_KEY
- AWS_SECRET_ACCESS_KEY ← SUPABASE_S3_SECRET_KEY
- AWS_S3_BUCKET_NAME ← SUPABASE_BUCKET
- SECRET_KEY ← SECRET_KEY
- REDIS_URL ← REDIS_URL (if provided)
- AMQP_URL ← AMQP_URL (if provided)

## 2) Supabase storage: create the bucket (one-time)
1. Choose a bucket name (e.g., uploads) — this must match SUPABASE_BUCKET.
2. Supabase Dashboard → Storage → Buckets → New bucket
   - Name: your bucket name
   - Visibility: Private (recommended)
3. Enable S3 compatibility: Project Settings → Storage → S3
   - Note the S3 Endpoint URL
   - Create or view an S3 Access Key and Secret
4. (Recommended) Configure S3 CORS to allow your Cloud Run Web/API URLs
   - Allowed origins: your Web and API service URLs
   - Methods: GET, POST, PUT
   - Allowed headers: *
   - Expose headers: ETag

With the bucket in place, the API’s startup check will succeed.

## 3) Upstash Redis setup (cheap cache)
1. Create an Upstash account and a Redis database in a region near your Cloud Run region.
2. Copy the Redis URL (rediss://...) — not the REST URL.
3. Save it as the GitHub secret REDIS_URL.

Benefits: near-zero idle cost, no GCP VPC connector needed, works directly from Cloud Run.

## 4) (Optional) Celery broker via CloudAMQP
1. Create a free instance at CloudAMQP.
2. Copy the AMQP URL (amqps://...).
3. Save it as GitHub secret AMQP_URL.

If you deploy the worker, AMQP_URL is required. Without it, the worker has no broker and won’t process tasks.

## 5) GCP prep checklist
- Enable APIs: Cloud Run, Artifact Registry
- Create an Artifact Registry repository named GAR_REPOSITORY in GAR_LOCATION
- Service account with roles: Artifact Registry Writer, Cloud Run Admin, Service Account User
- Create a JSON key for that SA and save it as GCP_SA_KEY

## 6) Deploy
1. Push to main or trigger manually:
   - GitHub → Actions → Deploy Plane to Cloud Run → Run workflow
   - Inputs:
     - deploy_api: true
     - deploy_web: true
     - deploy_worker: false (or true if you added AMQP_URL and want background tasks)
2. The workflow will:
   - Build/push API image
   - Run Django migrations against Supabase
   - Deploy API (injects Supabase + optional Redis/AMQP env)
   - Resolve API URL
   - Build/push Web image with NEXT_PUBLIC_API_BASE_URL baked in
   - Deploy Web (public)
   - (Optional) Deploy Worker as a private Cloud Run service (scale-to-zero)
3. Copy the printed service URLs at the end for API and Web.

## 7) Post-deploy tips
- CORS hardening: set CORS_ALLOWED_ORIGINS (comma-separated) to your Web and API URLs on the API service.
- Email: configure SMTP envs if you want email notifications processed by Celery.
- Live collab: if you plan to run the real-time server (`apps/live`), you’ll also need Redis and a new Cloud Run service; otherwise basic editing works without it.

That’s it. With min-instances=0 and small CPU/mem, this setup should run for a few dollars or less per month at very low usage.
