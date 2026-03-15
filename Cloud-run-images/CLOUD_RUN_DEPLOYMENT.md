# Cloud Run Automated Deployment

## Overview

The TrendSync Brand Factory backend is deployed as **three serverless microservices** on Google Cloud Run. The deployment is fully automated — a single Docker image is built once and deployed three times with different environment variables, allowing each service to start independently without manual intervention.

## Architecture: One Image, Three Services

Instead of maintaining separate Dockerfiles and build pipelines for each microservice, we use a **single container image** combined with a dynamic entrypoint that routes to the correct service at runtime based on the `SERVICE` environment variable.

```
┌──────────────────────────────────────────────────┐
│          Single Docker Image (python:3.11-slim)  │
│                                                  │
│   entrypoint.py reads SERVICE env var            │
│        │                                         │
│        ├── SERVICE=main-backend     → port 8080  │
│        ├── SERVICE=video-gen-service → port 8080  │
│        └── SERVICE=voice-companion  → port 8080  │
└──────────────────────────────────────────────────┘
```

Cloud Run deploys each service from the same image in Google Container Registry:

```
gcr.io/project-ca52e7fa-d4e3-47fa-9df/trendsync-backend@sha256:aaa03c...
```

## Files Used for Deployment

### `trendsync-backend/Dockerfile`

Defines the container image shared by all three services:

- **Base image:** `python:3.11-slim` — lightweight production Python runtime
- **System dependency:** Installs `ffmpeg` (required by the video generation pipeline for Veo 3.1 video stitching)
- **Python dependencies:** Installs from `requirements.txt` with `--no-cache-dir` for smaller image size
- **Source copy:** Copies the entire backend source tree into `/app`
- **Service selection:** Sets `ENV SERVICE=main-backend` as default; overridden per Cloud Run service
- **Entrypoint:** `CMD ["python", "entrypoint.py"]`

### `trendsync-backend/entrypoint.py`

The dynamic service router that makes the single-image pattern work:

1. Reads the `SERVICE` environment variable (defaults to `main-backend`)
2. Reads the `PORT` environment variable (defaults to `8080`, the Cloud Run standard)
3. Locates the service module at `/app/services/{SERVICE}/main.py`
4. Uses `importlib` to dynamically load the module (necessary because service directory names contain hyphens, which are invalid in Python import paths)
5. Starts a `uvicorn` ASGI server on `0.0.0.0:{PORT}` with the loaded FastAPI `app`

### `trendsync-backend/.dockerignore`

Keeps the Docker build context clean and the image small by excluding: `.venv`, `__pycache__`, `.env`, `.git`, `.pytest_cache`, `*.pyc`, `.DS_Store`.

### `trendsync-backend/requirements.txt`

Python dependencies installed into the image, including: `google-adk`, `google-cloud-storage`, `fastapi`, `uvicorn`, `httpx`, `redis`, `websockets`, `Pillow`, `python-docx`, and others.

## The Three Cloud Run Services

### 1. `trendsync-main` (SERVICE=main-backend)

- **Role:** API gateway — all REST endpoints, ADK design companion ("Lux"), pipeline orchestration, WebSocket proxy to voice service
- **Source:** `trendsync-backend/services/main-backend/main.py`
- **Region:** us-central1
- **Resources:** 2 CPU, 2GB RAM, 600s request timeout
- **Concurrency:** 160
- **Autoscaling:** 1–5 instances
- **Deployed by:** `claude-code-setup-bot@project-ca52e7fa-d4e3-47fa-9df.iam.gserviceaccount.com` using gcloud

### 2. `trendsync-video` (SERVICE=video-gen-service)

- **Role:** Veo 3.1 video generation with FFmpeg composition, outputs stored to Google Cloud Storage
- **Source:** `trendsync-backend/services/video-gen-service/main.py`
- **Region:** us-central1
- **Resources:** 2 CPU, 2GB RAM, 600s request timeout
- **Concurrency:** 160
- **Autoscaling:** up to 3 instances
- **Deployed by:** same service account via gcloud

### 3. `trendsync-voice` (SERVICE=voice-companion)

- **Role:** Bidirectional WebSocket audio streaming with Gemini Live 2.5 Flash Native Audio
- **Source:** `trendsync-backend/services/voice-companion/main.py`
- **Region:** us-central1
- **Resources:** 2 CPU, 2GB RAM, 3600s request timeout (longer for sustained audio sessions)
- **Concurrency:** 160
- **Autoscaling:** up to 3 instances
- **Deployed by:** same service account via gcloud

## Environment Variables (set per service in Cloud Run)

All three services share these common environment variables configured directly in Google Cloud Run (not baked into the image):

| Variable | Purpose |
|----------|---------|
| `SERVICE` | Selects which microservice to start |
| `GOOGLE_CLOUD_PROJECT` | GCP project ID |
| `GOOGLE_CLOUD_LOCATION` | `global` (required for Gemini 3 Preview) |
| `GOOGLE_CLOUD_REGION` | `us-central1` |
| `GOOGLE_GENAI_USE_VERTEXAI` | `TRUE` — routes AI calls through Vertex AI |
| `GCS_BUCKET` | `trendsync-brand-factory-media` |
| `VEO_MODEL` | `veo-3.1-generate-preview` |
| `GEMINI_PRO_MODEL` | `gemini-3-pro-preview` |
| `GEMINI_FLASH_MODEL` | `gemini-2.5-flash` |
| `GEMINI_FLASH_IMAGE_MODEL` | `gemini-3-pro-image-preview` |
| `GEMINI_VISUAL_MODEL` | `gemini-3-flash-preview` |
| `VOICE_MODEL` | `gemini-live-2.5-flash-native-audio` |
| `REDIS_URL` | Managed Redis instance on AWS (ElastiCache) |
| `FOXIT_CLIENT_ID` / `FOXIT_CLIENT_SECRET` | Foxit Cloud PDF conversion API |
| `VIDEO_GEN_SERVICE_URL` | Internal Cloud Run URL for inter-service calls |

## How Deployment Works (Automated)

The deployment is triggered via `gcloud` CLI commands executed by the `claude-code-setup-bot` service account. The process:

1. **Build:** `docker build` creates the image from `trendsync-backend/Dockerfile`
2. **Push:** Image is pushed to Google Container Registry (`gcr.io/project-xxx/trendsync-backend`)
3. **Deploy (×3):** Three `gcloud run deploy` commands deploy the same image with different `SERVICE` values:

```bash
# Main backend
gcloud run deploy trendsync-main \
  --image gcr.io/$PROJECT/trendsync-backend \
  --set-env-vars SERVICE=main-backend \
  --region us-central1 --memory 2Gi --cpu 2 --timeout 600

# Video generation
gcloud run deploy trendsync-video \
  --image gcr.io/$PROJECT/trendsync-backend \
  --set-env-vars SERVICE=video-gen-service \
  --region us-central1 --memory 2Gi --cpu 2 --timeout 600

# Voice companion
gcloud run deploy trendsync-voice \
  --image gcr.io/$PROJECT/trendsync-backend \
  --set-env-vars SERVICE=voice-companion \
  --region us-central1 --memory 2Gi --cpu 2 --timeout 3600
```

4. **Traffic routing:** Cloud Run automatically routes 100% of traffic to the latest healthy revision
5. **Health checks:** Each service exposes health check endpoints that Cloud Run monitors

## Screenshots

The screenshots in this directory show the live Cloud Run console for each deployed service:

| File | Description |
|------|-------------|
| `trendsync-main1.jpg` | Main backend service — general config, autoscaling (1–5 instances), container image reference |
| `trendsync-main2.jpg` | Main backend service — all 16 environment variables configured in Cloud Run |
| `trendsync-video1.jpg` | Video generation service — container config, autoscaling (up to 3 instances) |
| `trendsync-video2.jpg` | Video generation service — environment variables including `SERVICE=video-gen-service` |
| `trendsync-voice1.jpg` | Voice companion service — 3600s timeout for long audio sessions, autoscaling (up to 3 instances) |
| `trendsync-voice2.jpg` | Voice companion service — environment variables including `SERVICE=voice-companion` |

All services show deployment date of **February 27, 2026** with two revisions each, deployed by the automated service account.
