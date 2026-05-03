Generate, review, and optimize code with Cursor AI, GitHub, Google, and Slack

https://n8nworkflows.xyz/workflows/generate--review--and-optimize-code-with-cursor-ai--github--google--and-slack-13827


# Generate, review, and optimize code with Cursor AI, GitHub, Google, and Slack

# Reference Document: n8n Cursor AI Coding Workflow

## 1. Workflow Overview
The **n8n Cursor AI Coding Workflow** is an automated pipeline designed to bridge the gap between high-level task descriptions and production-ready code. It leverages the advanced coding intelligence of Cursor AI to automate the entire lifecycle of a coding task: generation, review, optimization, and deployment.

By integrating directly with GitHub, Google Workspace, and Slack, this workflow transforms n8n into a continuous integration/continuous delivery (CI/CD) engine for AI-assisted development. It ensures that every piece of AI-generated code meets a specific quality threshold before being committed, documented, and logged.

### Logical Blocks
1.  **1.1 Input Reception & Configuration:** Captures incoming tasks via Webhook or Schedule and sets the operational environment variables.
2.  **1.2 Intelligence Phase (Classification):** Uses LLMs to categorize the request (e.g., Generate vs. Refactor) to select the correct prompt strategy.
3.  **1.3 Execution Phase (Cursor AI):** Calls the Cursor AI API to generate the code and subsequently perform a structured code review.
4.  **1.4 Quality Gate:** Evaluates the AI's self-review score against a user-defined threshold.
5.  **1.5 Deployment & Notification:** Commits approved code to GitHub, saves reports to Google Drive, logs data to Sheets, and alerts the team via Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Task Input
- **Overview:** Receives the coding task and initializes the workflow with repository settings, quality thresholds, and model selections.
- **Nodes Involved:** `Webhook: Receive Coding Task`, `Daily Batch (fallback)`, `Set Task Config`.
- **Node Details:**
    - **Webhook / Schedule Trigger:** Accepts a POST request with a `task` body or triggers every 24 hours.
    - **Set Task Config (Set Node):** 
        - Defines hardcoded variables: `repo_owner`, `repo_name`, `target_branch`, `language` (default: python), `quality_threshold` (default: 75), `storage_folder`, `log_sheet_id`, and `cursor_model` (cursor-fast).
        - *Edge Case:* If the JSON body in the webhook is missing the `task` key, subsequent AI nodes will fail.

### 2.2 Classification Phase
- **Overview:** Analyzes the user's prompt to determine the intent and technical requirements.
- **Nodes Involved:** `Create a conversation`, `Wait: Processing Delay`, `Parse Classification`.
- **Node Details:**
    - **Create a conversation (OpenAI):** Uses OpenAI to interpret the task and output a classification.
    - **Wait (Wait Node):** A short delay to ensure API stability and avoid rate limits.
    - **Parse Classification (Code Node):** 
        - Uses regex (`/```json|```/g`) to clean the AI's response and parse it into `task_type`, `language`, and `summary`.
        - *Failure Type:* Will error if the AI output is not valid JSON or contains unexpected text.

### 2.3 Cursor Phase
- **Overview:** The core processing engine where the code is authored and critiqued.
- **Nodes Involved:** `Cursor AI: Generate Code`, `Cursor AI: Code Review`.
- **Node Details:**
    - **Cursor AI: Generate Code (HTTP Request):** 
        - Sends a System Prompt defining the AI as an expert developer. 
        - Parameters: `model` (from config), `max_tokens` (2000). 
        - Requires: `cursorApiKey`.
    - **Cursor AI: Code Review (HTTP Request):** 
        - Sends the generated code back to Cursor for a "Senior Reviewer" audit. 
        - Instructions: Must return JSON with `score`, `issues`, and `suggestions`.

### 2.4 Quality Gate
- **Overview:** A logic check that decides whether the code is ready for production or requires an optimization pass.
- **Nodes Involved:** `Quality Score OK?`.
- **Node Details:**
    - **Quality Score OK? (If Node):** Compares the `score` from the review node against the `quality_threshold` (e.g., 75) defined in the Config node.

### 2.5 Output Phase (Deployment & Logging)
- **Overview:** Finalizes the task by pushing to Git and updating management tools.
- **Nodes Involved:** `GitHub: Commit Code`, `Cursor AI: Optimize Code`, `Google Drive: Save Report`, `Google Sheets: Log Result`, `Notify Slack: Success`, `Notify Slack: Optimized`.
- **Node Details:**
    - **GitHub: Commit Code (HTTP Request):** Base64 encodes the code and pushes it to GitHub using the GitHub REST API. Dynamic filename based on timestamp.
    - **Google Drive: Save Report:** Creates a Markdown file containing the review details in the specified folder.
    - **Google Sheets: Log Result:** Appends a row with the Task, Status, Language, Score, and GitHub link.
    - **Slack Nodes:** Sends formatted messages to the `ai-code-review` channel.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook: Receive Coding Task | Webhook | Entry Point | - | Set Task Config | 1. Trigger & Task Input |
| Daily Batch (fallback) | Schedule | Entry Point | - | Set Task Config | 1. Trigger & Task Input |
| Set Task Config | Set | Variable Init | Webhook, Schedule | Create a conversation | 1. Trigger & Task Input |
| Create a conversation | OpenAI | AI Intent Parsing | Set Task Config | Wait | 2. Task Classification |
| Wait: Processing Delay | Wait | Flow Control | Create a conversation | Parse Classification | 2. Task Classification |
| Parse Classification | Code | Data Cleanup | Wait | Cursor AI: Generate Code | 2. Task Classification |
| Cursor AI: Generate Code | HTTP Request | Code Writing | Parse Classification | Cursor AI: Code Review | 3. Cursor AI Code Generation |
| Cursor AI: Code Review | HTTP Request | Quality Audit | Cursor AI: Gen Code | Quality Score OK? | 3. Cursor AI Code Generation |
| Quality Score OK? | If | Logic Gate | Cursor AI: Review | GitHub, Cursor AI: Opt | 4. Quality Gate |
| Cursor AI: Optimize Code | HTTP Request | Self-Correction | Quality Score OK? | Notify Slack: Optimized | 3. Cursor AI Code Generation |
| GitHub: Commit Code | HTTP Request | Deployment | Quality Score OK? | Google Drive | 5. Commit, Store, Log & Notify |
| Google Drive: Save Report | Google Drive | Documentation | GitHub: Commit Code | Google Sheets | 5. Commit, Store, Log & Notify |
| Google Sheets: Log Result | Google Sheets | Auditing | Google Drive | Notify Slack: Success | 5. Commit, Store, Log & Notify |
| Notify Slack: Success | Slack | Communication | Google Sheets | - | 5. Commit, Store, Log & Notify |
| Notify Slack: Optimized | Slack | Communication | Cursor AI: Opt | - | 5. Commit, Store, Log & Notify |

---

## 4. Reproducing the Workflow from Scratch

1.  **Global Configuration:** Create a **Set Node** ("Set Task Config"). Add fields for GitHub username, repository name, target branch (e.g., `ai-generated`), and a `quality_threshold` integer (75).
2.  **Triggers:** Add a **Webhook Node** (POST) and a **Schedule Trigger**. Connect both to the Set Node.
3.  **Classification:** 
    *   Connect the Set node to an **OpenAI Node**. Configure it to output a JSON summary of the task.
    *   Add a **Wait Node** for 1 second.
    *   Add a **Code Node** to parse the OpenAI output. Use `$input.item.json.message.content.replace(/`json|`/g, '')`.
4.  **Cursor API Integration:**
    *   Create an **HTTP Request Node** ("Cursor AI: Generate Code"). URL: `https://api.cursor.sh/v1/chat/completions`. Method: POST. Use Headers for `Authorization` (Bearer Token).
    *   Create a second **HTTP Request Node** for "Code Review". The body should prompt the AI to return a JSON object with a `score`.
5.  **Quality Logic:**
    *   Add an **If Node**. Condition: `{{ $json.score }} >= {{ $('Set Task Config').item.json.quality_threshold }}`.
6.  **GitHub Deployment (True Path):**
    *   Add an **HTTP Request Node**. Method: PUT. URL: `https://api.github.com/repos/{{owner}}/{{repo}}/contents/{{path}}`.
    *   Body: Send the code as a Base64 string (use `Buffer.from().toString('base64')`).
7.  **Storage & Logging:**
    *   Add a **Google Drive Node** to upload the review report.
    *   Add a **Google Sheets Node** to append the row of results.
8.  **Optimization (False Path):**
    *   Add an **HTTP Request Node** to Cursor AI specifically for fixing issues found in the review.
9.  **Notifications:**
    *   Add **Slack Nodes** to both the "Success" and "Optimized" paths to alert the team.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Cursor AI API Access | Requires Cursor developer access or a compatible proxy endpoint. |
| GitHub Auth | Must use a Personal Access Token (PAT) with `repo` scopes. |
| Quality Threshold | Setting this too high (95+) may trigger frequent optimization loops. |
| Data Privacy | Ensure code sent to APIs follows your company's security policy. |