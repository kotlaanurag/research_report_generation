## Automated Research Report Generation — repository overview

This project is an autonomous research report generation system that uses LangChain/LangGraph-style workflows to generate structured reports (DOCX/PDF) from a given topic. It includes a small FastAPI-based UI to submit topics, track progress, provide human feedback, and download generated reports.

This README documents the project structure, how the frontend and backend work together, and clear, actionable steps to run locally, build Docker images, and deploy using CI/CD (Jenkins + Azure). It also covers environment variables, secrets, and troubleshooting tips.

## Table of contents
- Project summary
- Repository layout and responsibilities
- Frontend (templates & static)
- Backend (API, services, workflows)
- Data flow and report generation
- Running locally (dev)
- Docker (build & run)
- CI/CD with Jenkins (high level)
- Azure deployment (scripts included)
- Environment and secrets
- Troubleshooting & next steps

## Project summary
- Purpose: Accept a topic, run an autonomous multi-analyst workflow, compile content, and save final reports in DOCX and PDF formats.
- Tech stack: Python 3.11, FastAPI, Uvicorn, LangChain/LangGraph (langgraph), langchain adapters, langgraph memory checkpointing, Jinja2 templates, Docker, Jenkins, Azure Container Registry / Container Apps.

## Repository layout (high level)

- `main.py` — small project entrypoint (prints greeting). Primary service is in `research_and_analyst` package.
- `pyproject.toml`, `requirements.txt` — dependency metadata. Project targets Python >= 3.11.
- `Dockerfile`, `Dockerfile.jenkins` — images for app runtime and Jenkins builder image respectively.
- `Jenkinsfile` — CI pipeline to install deps, run a simple import test, login to Azure, and deploy to Azure Container Apps.
- `build-and-push-docker-image.sh`, `setup-app-infrastructure.sh`, `azure-deploy-jenkins.sh` — utility scripts for building/pushing images and creating Azure infrastructure.
- `research_and_analyst/` — main application code:
  - `api/` — FastAPI app, routes, templates. Key files: `main.py`, `routes/report_routes.py`, `services/report_service.py`.
  - `workflows/` — graph-based workflows. Key file: `report_generator_workflow.py` which composes LangGraph nodes to create analysts, run interviews, write sections, and finalize the report.
  - `prompt_lib/` — prompts used by the LLMs (prompt templates referenced by workflows).
  - `utils/` — helper modules such as model loader and config loader.
  - `schemas/` — dataclasses/models used by LangGraph and validation.
  - `logger/`, `exception/` — logging and custom exceptions.
- `static/` — CSS and JS served by FastAPI; templates live under `research_and_analyst/api/templates`.
- `generated_report/` — sample or produced reports (DOCX/PDF) organized per-topic and timestamp.

## Frontend (templates & static)

- Served by FastAPI (see `research_and_analyst.api.main`). The `StaticFiles` mount exposes `/static` for CSS/JS.
- Jinja2 templates are under `research_and_analyst/api/templates` and render pages:
  - `login.html`, `signup.html` — auth flow (very basic, backed by a local SQLite DB via SQLAlchemy in `database/db_config.py`).
  - `dashboard.html` — entry point after login; form to submit a topic.
  - `report_progress.html` — shows generation progress, allows feedback submission, and provides download links when ready.
- Static assets: `static/css/styles.css` and `static/js/app.js` — minimal styling and client-side behavior for the UI.

Frontend behavior overview:
- User logs in or signs up (simple username/password stored with hashing).
- From the dashboard, the user posts a `topic` to `/generate_report` which triggers the pipeline and returns a thread id used to check status and submit feedback.
- After pipeline completes, the UI shows links to download the generated DOCX/PDF via `/download/{file_name}`.

## Backend and workflows

Core components:
- `research_and_analyst/api/routes/report_routes.py` — hosts the HTTP endpoints (login/signup, dashboard, generate_report, submit_feedback, download). Routes use `ReportService` for report lifecycle operations.
- `research_and_analyst/api/services/report_service.py` — orchestrates the workflow: loads the LLM via `ModelLoader`, instantiates `AutonomousReportGenerator` (workflow class), triggers the LangGraph pipeline, stores a thread id, and exposes methods to submit feedback, check status, and download generated files.
- `research_and_analyst/workflows/report_generator_workflow.py` — implements the LangGraph graph nodes:
  - create_analyst: produce analyst personas using structured LLM outputs
  - conduct_interview: interview each persona to collect sections
  - write_report / write_introduction / write_conclusion: stitch content with LLM
  - finalize_report: combine sections and write final output
  - save_report: writes DOCX (python-docx) and PDF (reportlab) files into `generated_report/<topic>_<timestamp>/`

The service uses an in-memory/shared `MemorySaver` checkpoint from `langgraph` to keep state across nodes and to update the graph when human feedback is submitted.

Model loading:
- `research_and_analyst/utils/model_loader.py` (not expanded here) is responsible for configuring the LLM provider (OpenAI / Google / GROQ / Tavily) using env vars / secrets. The repo includes adapters for `langchain-openai`, `langchain-google-genai`, and `langchain-groq` in `pyproject`/`requirements`.

Database and auth:
- A small SQLAlchemy-based local DB (`research_and_analyst/database/db_config.py`) stores users for the UI login/signup flow. Passwords are hashed.

Error handling & logging:
- `research_and_analyst/exception/custom_exception.py` defines domain exceptions.
- `research_and_analyst/logger/custom_logger.py` provides `GLOBAL_LOGGER` for structured logs (used across the app).

## Data flow (short)
1. User submits a topic via POST `/generate_report`.
2. `ReportService.start_report_generation` creates a unique thread ID and streams the LangGraph pipeline to generate sections and final content.
3. The pipeline pauses for human feedback via the `human_feedback` node; the UI can submit feedback which updates the graph state.
4. When the graph reaches `final_report`, `ReportService.get_report_status` calls `AutonomousReportGenerator.save_report` to write DOCX and PDF files to `generated_report/` and returns file paths.
5. User can download via `/download/{file_name}`, which streams the file using FastAPI `FileResponse`.

## Run locally (developer quick-start)

Prerequisites:
- Python 3.11
- Recommended: create and activate a virtualenv

Install dependencies (on Windows PowerShell):

```powershell
python -m venv .venv; .\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install -e .
```

Run the FastAPI server (development):

```powershell
# from project root
uvicorn research_and_analyst.api.main:app --reload --host 127.0.0.1 --port 8000
```

Open the browser at http://127.0.0.1:8000/ to access the UI.

Notes for local dev:
- Ensure any LLM API keys (OpenAI, Google, GROQ, TAVILY) are set in environment variables before running model-related endpoints. If you don't have keys, some parts of the pipeline that call the LLM will fail; the UI still can load.
- Generated reports will appear under `generated_report/`.

## Docker: build and run

Build image (Linux/amd64 recommended for Azure deployments) and run locally:

```powershell
# build (if Docker Desktop is installed)
# Replace TAG with a version tag, e.g. 'latest'
docker build --platform linux/amd64 -t research-report-app:latest .

# run (map port 8000)
docker run --rm -p 8000:8000 -e OPENAI_API_KEY="<your_key>" research-report-app:latest
```

The repo includes `build-and-push-docker-image.sh` which builds an image and pushes it to an Azure Container Registry (ACR) configured by name in that script. On Windows, run the equivalent commands in PowerShell (or use WSL).

Health check: the container exposes `/health` which returns a JSON health status.

## CI/CD with Jenkins (overview)

- `Dockerfile.jenkins` builds a Jenkins image with Git, Python and Azure CLI preinstalled. It sets `git config --system --add safe.directory "*"` to avoid Git safe directory issues.
- `Jenkinsfile` demonstrates a pipeline that checks out `main`, installs dependencies, runs a simple import test, logs into Azure using stored Jenkins credentials, fetches the latest image tag from ACR, and updates/creates an Azure Container App with that image.

Recommended Jenkins credentials (as used in `Jenkinsfile`):
- Azure service principal (client id / secret / tenant / subscription)
- ACR username/password (or use managed identity/service principal)
- Storage account name/key (used by setup scripts)
- API keys for LLM providers stored as Jenkins credentials (OPENAI_API_KEY, GOOGLE_API_KEY, etc.)

### Running the pipeline
1. Use `setup-app-infrastructure.sh` to create an Azure Resource Group, ACR, storage account and container apps environment (it will echo ACR credentials — securely store them).
2. Build and push the app image to ACR using `build-and-push-docker-image.sh`.
3. Run the Jenkins pipeline (manually or via webhook). The pipeline will pick the latest ACR image and deploy it to Azure Container Apps.

## Azure deployment (scripts included)

- `setup-app-infrastructure.sh` — creates resource group, ACR, storage account, file share, and container apps environment.
- `build-and-push-docker-image.sh` — builds the Docker image and pushes it to the configured ACR.
- `azure-deploy-jenkins.sh` — example script that builds the custom Jenkins image (from `Dockerfile.jenkins`), pushes it to ACR, and deploys a Jenkins container to Azure Containers with an Azure file share for `/var/jenkins_home`.

High-level sequence to deploy app to Azure Container Apps:
1. Login to Azure and set subscription: `az login` / `az account set --subscription <id>`
2. Run `./setup-app-infrastructure.sh` to create ACR and container apps environment.
3. Build & push image with `./build-and-push-docker-image.sh <tag>`.
4. Run the Jenkins pipeline (or use `az containerapp create`/`update` as shown in the `Jenkinsfile` deploy stage) to update the running container app with the new image tag.

## Environment variables and secrets

The application and CI rely on the following external secrets (minimal set):
- OPENAI_API_KEY (or GOOGLE_API_KEY / GROQ_API_KEY / TAVILY_API_KEY depending on provider)
- LLM_PROVIDER (string to indicate provider used)
- For Azure: ACR credentials, service principal (AZURE_CLIENT_ID / AZURE_CLIENT_SECRET / AZURE_TENANT_ID / AZURE_SUBSCRIPTION_ID), storage account key.

Security notes:
- Never commit API keys to source control. Use Jenkins credentials store or Azure Key Vault.
- When deploying to Azure Container Apps, store keys as secrets and reference via secretRef in container app config (the `Jenkinsfile` shows an example).

## Troubleshooting

- If the LLM calls fail: verify provider env vars and network connectivity. Check `research_and_analyst/utils/model_loader.py` to match the expected env names.
- If Docker image push fails: ensure you `az acr login` and the ACR name in the script matches your registry.
- If the UI shows no progress: check the server logs; LangGraph pipelines stream data — exceptions can stop the graph. Check `generated_report/` for partial outputs.

## Next steps / recommended improvements

1. Add unit tests and CI stage that runs them (currently the Jenkinsfile runs an import-only sanity check).
2. Add a small health/readiness endpoint that verifies connectivity to required external services (LLM providers, storage) before declaring healthy.
3. Add feature flags or a configuration layer to select which LLM provider to use at runtime.
4. Introduce asynchronous job queue (Redis/RQ, Celery, or background worker) for large-generation tasks to avoid blocking request threads and to scale workers separately.

## Contact / authorship
Project author: Anurag Kotla

```
**Project Document Link:** https://docs.google.com/document/d/1VlHirN62sWE1CwXr4v2YM40sg8luskD6VY4A2gKOHK4/edit?usp=sharing
```

```
uvicorn research_and_analyst.api.main:app --reload
```