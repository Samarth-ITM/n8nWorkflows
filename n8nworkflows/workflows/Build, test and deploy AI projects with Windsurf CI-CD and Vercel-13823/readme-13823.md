Build, test and deploy AI projects with Windsurf CI/CD and Vercel

https://n8nworkflows.xyz/workflows/build--test-and-deploy-ai-projects-with-windsurf-ci-cd-and-vercel-13823


# Build, test and deploy AI projects with Windsurf CI/CD and Vercel

# Workflow Reference: n8n Windsurf Deployment (Build, Test & Deploy)

This document provides a detailed technical breakdown of the n8n workflow designed to automate CI/CD pipelines for AI projects. It leverages Windsurf’s AI capabilities to handle testing and evaluation, followed by containerization and deployment to Vercel or similar platforms.

---

### 1. Workflow Overview

The purpose of this workflow is to provide a low-code alternative to traditional CI/CD tools (like GitHub Actions) specifically for AI-driven projects. It automates the lifecycle from code commit to production deployment, ensuring that model evaluations and unit tests are passed before the code reaches the target environment.

**Logical Blocks:**
- **1.1 Trigger & Input:** Capture events from Git (Webhooks) or scheduled intervals and set project metadata.
- **1.2 Repository Sync:** Fetch the latest commit information from the remote repository.
- **1.3 Windsurf AI Processing:** Trigger Windsurf-powered tasks (linting, testing, model evaluation).
- **1.4 Containerization:** (Conditional) Build and push Docker images if tests pass.
- **1.5 Deployment & Notification:** Deploy the build to a target platform (Vercel) and send status updates to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input
This block initiates the pipeline. It handles both real-time events and scheduled fallbacks.

*   **Nodes Involved:** `Webhook: Git Push / Tag`, `Daily Check (fallback)`, `Set Project Config`.
*   **Node Details:**
    *   **Webhook (v2):** Listens for `POST` requests at `/windsurf-deploy-hook`. Used for Git push/tag events.
    *   **Schedule Trigger (v1.3):** Configured as a daily fallback to ensure the project stays up to date even without push events.
    *   **Set (v2):** Hardcodes project variables including `repo_url`, `branch`, `windsurf_profile`, and `deploy_platform`. These variables are used in subsequent HTTP requests.

#### 1.2 Repository Sync
Validates the state of the repository before starting expensive AI build processes.

*   **Nodes Involved:** `Git: Get Latest Commit`, `Wait For Data`.
*   **Node Details:**
    *   **HTTP Request (v4.2):** Calls the GitHub API to retrieve metadata for the latest commit on the specified branch. Uses an expression to extract the owner/repo from the URL: `{{ $json.repo_url.split('/').slice(-2).join('/') }}`.
    *   **Wait (v1.1):** Acts as a buffer/pause if needed to ensure repository consistency or rate-limit handling.

#### 1.3 Windsurf AI Processing
The core intelligence phase where AI-powered tests are executed.

*   **Nodes Involved:** `Windsurf: Build & Test`, `Build & Tests OK?`.
*   **Node Details:**
    *   **HTTP Request (v4.2):** Connects to `https://api.windsurf.dev/v1/run`. This initiates the Windsurf profile (AI-powered linting and model evaluation).
    *   **IF Node (v1):** Checks the boolean logic: `{{ $json.build_success === true || $json.tests_passed === true }}`.
        *   **True Path:** Proceeds to Docker build.
        *   **False Path:** Redirects immediately to Slack for failure notification.

#### 1.4 Containerization
Prepares the application for cloud-native deployment.

*   **Nodes Involved:** `Docker: Build & Push (placeholder)`.
*   **Node Details:**
    *   **HTTP Request (v4.2):** A placeholder request to Docker Hub (or a private registry). In a production environment, this would typically trigger a remote build or interact with a Docker daemon via API.

#### 1.5 Deployment & Notification
The final stage where code is moved to production and stakeholders are informed.

*   **Nodes Involved:** `Deploy → Vercel (example)`, `Notify Slack`.
*   **Node Details:**
    *   **HTTP Request (v4.2):** Sends a deployment request to the Vercel API (`/v13/deployments`).
    *   **Slack (v1):** Sends a formatted message to the `ci-cd-notifications` channel. 
        *   **Dynamic Message:** `{{ $json.success ? '🚀 AI Project Deployed successfully!\nURL: ' + $json.deploy_url : '❌ Build / Deploy failed: ' + $json.error_message }}`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook: Git Push / Tag | Webhook | Entry Point | None | Set Project Config | ## 1. Trigger & Input Git event or schedule → prepare repo info |
| Daily Check (fallback) | ScheduleTrigger | Entry Point | None | Set Project Config | ## 1. Trigger & Input Git event or schedule → prepare repo info |
| Set Project Config | Set | Config Management | Webhook, Daily Check | Git: Get Latest Commit | ## 1. Trigger & Input Git event or schedule → prepare repo info |
| Git: Get Latest Commit | HTTP Request | Data Retrieval | Set Project Config | Wait For Data | ## 2. Windsurf Build & Test AI-powered lint, test, eval via Windsurf |
| Wait For Data | Wait | Flow Control | Git: Get Latest Commit | Windsurf: Build & Test | ## 2. Windsurf Build & Test AI-powered lint, test, eval via Windsurf |
| Windsurf: Build & Test | HTTP Request | AI Processing | Wait For Data | Build & Tests OK? | ## 2. Windsurf Build & Test AI-powered lint, test, eval via Windsurf |
| Build & Tests OK? | IF | Logic Gate | Windsurf: Build & Test | Docker Build (True), Slack (False) | ## 2. Windsurf Build & Test AI-powered lint, test, eval via Windsurf |
| Docker: Build & Push | HTTP Request | Containerization | Build & Tests OK? | Deploy → Vercel | ## 3. Container & Push Docker build → registry push |
| Deploy → Vercel | HTTP Request | Deployment | Docker: Build & Push | Notify Slack | ## 4. Deploy & Notify Target platform + alerts |
| Notify Slack | Slack | Notification | Deploy, Build OK (False) | None | ## 4. Deploy & Notify Target platform + alerts |

---

### 4. Reproducing the Workflow from Scratch

1.  **Triggers:** 
    *   Create a **Webhook** node (POST) for Git provider integration.
    *   Create a **Schedule Trigger** set to your preferred frequency (e.g., daily).
2.  **Configuration:** 
    *   Add a **Set** node. Define strings for `repo_url`, `branch`, `windsurf_profile`, and `deploy_platform`.
3.  **Repository Sync:** 
    *   Add an **HTTP Request** node to fetch commit data from GitHub/GitLab. 
    *   Add a **Wait** node (set to a small duration or "Wait for Webhook Call" if using external CI runners).
4.  **Windsurf Integration:** 
    *   Add an **HTTP Request** node calling the Windsurf API (`/v1/run`). Use your Windsurf API Key in the Header (Authorization: Bearer).
5.  **Logic Gate:** 
    *   Add an **IF** node. Check if `build_success` or `tests_passed` is true from the Windsurf response.
6.  **Container Phase:** 
    *   Add an **HTTP Request** node to trigger your Docker build service (e.g., Docker Hub, GitHub Packages).
7.  **Deployment:** 
    *   Add an **HTTP Request** node for **Vercel**. Use the POST method to `/v13/deployments`. Include your Vercel Token in the Headers.
8.  **Communication:** 
    *   Add a **Slack** node. Connect it to both the `Deploy` node (Success path) and the `IF` node (Failure path).
9.  **Credentials:** 
    *   Ensure credentials are created for: **GitHub API**, **Windsurf API**, **Vercel API**, and **Slack OAuth2**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Windsurf API Documentation** | For advanced AI testing profiles, refer to [Windsurf Docs](https://api.windsurf.dev). |
| **Vercel Deployment API** | Instructions for deployment tokens: [Vercel API Reference](https://vercel.com/docs/rest-api). |
| **Security Note** | Ensure all API keys are stored as n8n Credentials and not hardcoded in the Set node. |
| **Setup Guide** | See the large "Overview & Instructions" sticky note in the n8n canvas for manual setup steps. |