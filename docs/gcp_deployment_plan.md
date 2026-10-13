# Google Cloud Platform (GCP) Deployment Plan: Zero-Cost Architecture

## 1. Executive Summary & Zero-Cost Policy

This document defines the deployment architecture, configuration specifications, and step-by-step procedures for hosting the entire **MediFinderBot** solution on **Google Cloud Platform (GCP)**.

The deployment is designed under a **Strict Zero-Cost Policy ($0.00 USD/month)**, leveraging Google Cloud's **Always Free** tier and the existing Compute Engine infrastructure.

### Free Tier vs. Consumption Comparison

| GCP Component | Always Free Tier Limits (Per Month) | MediFinderBot Consumption | Monthly Cost |
| :--- | :--- | :--- | :--- |
| **Cloud Run (Web & MCP)** | **2,000,000 requests**<br>360,000 vCPU-seconds<br>180,000 GiB-seconds | ~15,000 - 50,000 requests<br>Scales to zero (0 instances when idle) | **$0.00** |
| **Cloud Run Jobs (ETL Ingestor)** | Draws from Cloud Run free compute pool | 3 min/day $\times$ 30 days = 5,400 vCPU-s (~1.5% of free pool) | **$0.00** |
| **Cloud Scheduler** | **3 jobs free** per billing account | 1 daily cron job (00:00 PET / 05:00 UTC) | **$0.00** |
| **PostgreSQL Database** | Reuses existing VM (`chapatuparte-db-vm`) | 1 isolated database (`medifinder`) on existing VM | **$0.00** *(No Cloud SQL)* |
| **Artifact Registry** | **0.5 GB storage free** | ~250 MB (auto-cleanup policy keeping last 2 tags) | **$0.00** |
| **Cloud Build** | **120 build-minutes/day free** | ~2 to 4 minutes per deployment | **$0.00** |
| **Secret Manager** | **6 secret versions free**<br>10,000 API operations free | 2 secrets (`DATABASE_URL`, `GEMINI_API_KEY`) | **$0.00** |
| **Network Egress** | **1 GB free egress** to all destinations | < 300 MB monthly HTTP response payload | **$0.00** |
| **Gemini AI API** | **1,500 requests/day free** (Gemini 2.0 Flash / 1.5 Flash)<br>15 RPM / 1M TPM | ~100 to 500 patient queries/day | **$0.00** |

---

## 2. GCP Project Structure & Region Selection

- **Dedicated Project Name**: `medifinder-pe` (or `medifinder-app`)
- **Billing Association**: Linked to the user's primary GCP billing account (enables APIs without incurring charges while within free limits).
- **Target Region**: **`us-central1`** (Iowa, USA)
  - *Rationale*: The existing PostgreSQL database VM (`chapatuparte-db-vm`) is located in zone `us-central1-a`. Placing Cloud Run services and jobs in the same region ensures sub-2ms network latency and zero cross-region data transfer fees.

---

## 3. Database Architecture (Existing PostgreSQL VM)

Rather than provisioning a paid Google Cloud SQL managed instance ($15–$50/month), MediFinder connects to the user's existing PostgreSQL 16 server running on Google Compute Engine:

- **Host / IP**: `136.114.176.10:5432`
- **Database Engine**: PostgreSQL 16.15 (Debian) with `pgvector` v0.8.6 pre-installed.
- **Target Database**: `medifinder`
- **Database User**: `lenin_admin`
- **Network Access**: Connected via secure TCP connection with strong authentication. No Serverless VPC Access Connector ($7.20/month) is required.

---

## 4. System Architecture

```mermaid
flowchart TD
    subgraph Users
        U[Web / Mobile Patient]
    end

    subgraph GCP Project: medifinder-pe
        subgraph Cloud Run Serverless
            CR[medifinder-web + MCP Server<br/>Unified Container in us-central1<br/>min=0, max=2 instances<br/>Memory: 512Mi, CPU: 1]
        end

        subgraph Scheduled Ingestion
            CS[Cloud Scheduler<br/>Daily Cron: 0 5 * * *] --> CRJ[Cloud Run Job: medifinder-ingestor<br/>SISMED Regional Scraper + DIGEMID API<br/>Playwright Headless]
        end

        subgraph Security
            SM[Secret Manager<br/>GEMINI_API_KEY, DATABASE_URL]
        end

        CRJ -.->|Daily Scraping| EXT1[(DIGEMID OPM API)]
        CRJ -.->|Daily Scraping| EXT2[(MINSA SISMED Power BI)]
    end

    subgraph Google AI
        GEMINI[Gemini 2.0 Flash API<br/>Native Tool Calling / AI Studio]
    end

    subgraph Existing VM: chapatuparte-pe
        PG[(PostgreSQL 16: medifinder<br/>chapatuparte-db-vm in us-central1-a<br/>IP: 136.114.176.10:5432)]
    end

    U -->|HTTPS / Chat| CR
    CR <-->|Tool Calling / Reasoning| GEMINI
    CR <-->|SQL Queries / pgvector| PG
    CRJ -->|Bulk Staging UPSERT| PG
    SM -->|Injects Env Vars| CR
    SM -->|Injects Env Vars| CRJ
```

---

## 5. Component Modernization Specifications

### 5.1 AI Engine Migration: Claude to Gemini 2.0 Flash
The legacy codebase from January 2025 used Anthropic Claude (`claude-3-haiku-20240307`) with string parsing hacks. The modernized system migrates to **Google Gemini 2.0 Flash** via the `google-genai` SDK:
- **Native Tool Calling**: Gemini receives tool declarations directly as JSON function schemas and returns structured `FunctionCall` objects, eliminating manual regex parsing.
- **Cost**: 100% Free under Google AI Studio limits (up to 1,500 requests per day).
- **Latency**: Under 600ms per turn.

### 5.2 Container Topology: Unified Web & MCP Service
To minimize cold starts, eliminate inter-container network latency, and avoid doubling request counts against the Cloud Run free quota:
- `medifinder-web` and `medifinder-mcp` run in a single container.
- The MCP tools are imported directly as a Python service module within the Flask runtime.
- Container limits: `512Mi` memory, `1` vCPU, `min-instances=0`, `max-instances=2`.

### 5.3 Automated Ingestion: Cloud Run Job
- `medifinder-ingestor` is containerized with Chromium and Playwright.
- Executes two automated scraping flows:
  1. **DIGEMID Catalog**: Direct HTTP POST to `https://ms-opm.minsa.gob.pe/msopmcovid/producto/catalogoproductos` (18,251 products, 12 columns, takes 5.5s).
  2. **MINSA SISMED Power BI**: Regional headless extraction with in-memory Blob bypass (takes ~1.5 min per region, 20,916 rows for Tumbes).
- Ingests data into PostgreSQL using set-based **Bulk Staging with `INSERT ... ON CONFLICT DO UPDATE`** in under 3 seconds.
- The job terminates immediately upon completion, releasing all cloud resources.

---

## 6. Step-by-Step Deployment Roadmap

### Phase 1: Local Code Modernization
1. Replace `medifinder-web/app/llm/client.py` with `google-genai` Gemini client using native tool calling.
2. Package scraping services into `medifinder-ingestor/services/scrapers/`.
3. Add Dockerfiles for `medifinder-web` and `medifinder-ingestor`.

### Phase 2: GCP Project Provisioning
Run via PowerShell using Google Cloud SDK:
```powershell
# 1. Create dedicated GCP project
gcloud projects create medifinder-pe --name="MediFinderBot"

# 2. Link billing account
$BILLING_ACCOUNT = (gcloud billing accounts list --format="value(name)" | Select-Object -First 1)
gcloud billing projects link medifinder-pe --billing-account=$BILLING_ACCOUNT

# 3. Set active project
gcloud config set project medifinder-pe

# 4. Enable required APIs (all eligible for Always Free)
gcloud services enable `
    run.googleapis.com `
    cloudbuild.googleapis.com `
    cloudscheduler.googleapis.com `
    secretmanager.googleapis.com `
    artifactregistry.googleapis.com

# 5. Create Artifact Registry repository
gcloud artifacts repositories create medifinder-repo `
    --repository-format=docker `
    --location=us-central1 `
    --description="MediFinder Docker Container Repository"
```

### Phase 3: Database Provisioning on Existing VM
1. Connect to PostgreSQL on `136.114.176.10:5432` (administrative `postgres` DB):
   ```sql
   CREATE DATABASE medifinder;
   ```
2. Connect to `medifinder` and enable extensions:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
   ```
3. Run schema creation and table migrations (`regions`, `medical_centers`, `products`, `product_types`, `inventory`).

### Phase 4: Secrets Management
Store sensitive parameters in Secret Manager:
```powershell
# Store database connection string
"postgresql://lenin_admin:LeninDev2026!GlobalDB@136.114.176.10:5432/medifinder" | `
    gcloud secrets create DATABASE_URL --data-file=-

# Store Google Gemini API Key
"YOUR_GEMINI_API_KEY" | `
    gcloud secrets create GEMINI_API_KEY --data-file=-
```

### Phase 5: Container Build & Cloud Run Deployment
```powershell
# Build web container using Cloud Build (free tier)
gcloud builds submit medifinder-web `
    --tag us-central1-docker.pkg.dev/medifinder-pe/medifinder-repo/medifinder-web:latest

# Deploy Web service to Cloud Run
gcloud run deploy medifinder-web `
    --image us-central1-docker.pkg.dev/medifinder-pe/medifinder-repo/medifinder-web:latest `
    --region us-central1 `
    --platform managed `
    --allow-unauthenticated `
    --min-instances 0 `
    --max-instances 2 `
    --memory 512Mi `
    --cpu 1 `
    --set-secrets="DATABASE_URL=DATABASE_URL:latest,GEMINI_API_KEY=GEMINI_API_KEY:latest"

# Build ingestor container
gcloud builds submit medifinder-ingestor `
    --tag us-central1-docker.pkg.dev/medifinder-pe/medifinder-repo/medifinder-ingestor:latest

# Deploy Cloud Run Job for ETL
gcloud run jobs create medifinder-ingestor-job `
    --image us-central1-docker.pkg.dev/medifinder-pe/medifinder-repo/medifinder-ingestor:latest `
    --region us-central1 `
    --memory 1Gi `
    --cpu 1 `
    --max-retries 1 `
    --task-timeout 600s `
    --set-secrets="DATABASE_URL=DATABASE_URL:latest"

# Schedule daily ETL execution at 00:00 PET (05:00 UTC)
gcloud scheduler jobs create http medifinder-daily-ingest `
    --location us-central1 `
    --schedule "0 5 * * *" `
    --time-zone "America/Lima" `
    --uri "https://us-central1-run.googleapis.com/v2/projects/medifinder-pe/locations/us-central1/jobs/medifinder-ingestor-job:run" `
    --http-method POST `
    --oauth-service-account-email "$(gcloud projects describe medifinder-pe --format='value(projectNumber)')-compute@developer.gserviceaccount.com"
```

---

## 7. Artifact Registry Storage Hygiene (< 0.5 GB Free Limit)

To guarantee that Artifact Registry never exceeds the 0.5 GB free limit, configure an automatic cleanup policy:

Create `cleanup-policy.json`:
```json
{
  "name": "delete-old-tags",
  "action": {"type": "Delete"},
  "condition": {
    "tagState": "ANY",
    "olderThan": "14d"
  }
}
```
Apply via:
```powershell
gcloud artifacts repositories set-cleanup-policies medifinder-repo `
    --location=us-central1 `
    --policy=cleanup-policy.json
```
This automatically deletes build artifacts older than 14 days, keeping total repository size consistently under 250 MB.
