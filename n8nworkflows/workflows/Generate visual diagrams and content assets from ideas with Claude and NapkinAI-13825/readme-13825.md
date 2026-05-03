Generate visual diagrams and content assets from ideas with Claude and NapkinAI

https://n8nworkflows.xyz/workflows/generate-visual-diagrams-and-content-assets-from-ideas-with-claude-and-napkinai-13825


# Generate visual diagrams and content assets from ideas with Claude and NapkinAI

### 1. Workflow Overview

This n8n workflow automates the process of transforming raw text ideas into professional visual diagrams and multi-channel content assets. It leverages **Anthropic's Claude** to structure and enrich raw thoughts, and **NapkinAI** to generate actual visual diagram files (PNG/SVG). 

The workflow is designed for content creators, marketing teams, and startup founders who want to quickly turn a brief text description into a "visual-first" content package, including infographics, social media hooks, and structured diagrams.

**Logical Blocks:**
*   **1.1 Input Reception & Normalization:** Captures data via Webhook or Manual Trigger, then validates and formats the data for processing.
*   **1.2 AI Enrichment (Claude):** Uses Claude-3.5 Sonnet to expand the idea, suggest diagram types, and generate specific prompts for the visual engine.
*   **1.3 NapkinAI Generation Engine:** A multi-step process involving authentication, job submission, status polling (retry loop), and asset retrieval.
*   **1.4 Asset Assembly & Delivery:** Consolidates all AI-generated text and visual assets into a single package, then distributes it via Email, Slack, and Database logging.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block ensures the workflow has a consistent data structure regardless of whether it was triggered manually for testing or via an external API call.
*   **Nodes Involved:** `Webhook - Receive Idea`, `Manual Trigger - Test`, `Set Test Idea Data`, `Merge Input Sources`, `Validate & Normalize Input`.
*   **Node Details:**
    *   **Validate & Normalize Input (Code):** Normalizes fields like `title`, `rawIdea`, and `visualStyle`. It generates a unique `ideaId` and calculates metadata like word count.
    *   **Edge Cases:** If `rawIdea` is missing or under 10 characters, the node throws an error to prevent wasted API credits.

#### 2.2 AI Enrichment (Claude)
This block transforms the raw text into a structured technical blueprint for the diagram.
*   **Nodes Involved:** `AI - Enrich & Structure Idea`, `Wait For Data`, `Parse Claude Enrichment`.
*   **Node Details:**
    *   **AI - Enrich & Structure Idea (HTTP Request):** Sends a system prompt to Claude (Anthropic API) requesting a JSON response containing an `enrichedText`, `napkinPrompt`, and `contentAngles`.
    *   **Wait For Data (Wait):** A brief buffer to ensure API stability.
    *   **Parse Claude Enrichment (Code):** Extracts the JSON from Claude’s response. It includes a **fallback mechanism**; if the AI's JSON is malformed, it defaults to using the raw input to ensure the workflow doesn't crash.

#### 2.3 NapkinAI Visual Generation
The core technical integration that interacts with the NapkinAI API to produce images.
*   **Nodes Involved:** `NapkinAI - Authenticate`, `NapkinAI - Submit Idea for Visual`, `NapkinAI - Poll Generation Status`, `Check Job Completion`, `Wait 5s - Retry Poll`, `NapkinAI - Retrieve Generated Assets`.
*   **Node Details:**
    *   **NapkinAI - Authenticate:** Obtains a Bearer Token using an API key.
    *   **Submit Idea for Visual:** POSTs the enriched text and style preferences. Returns a `jobId`.
    *   **The Polling Loop:** `Poll Generation Status` checks the `jobId`. `Check Job Completion` (IF node) determines if the status is "completed". If "processing", it goes to `Wait 5s` and loops back to the poll.
    *   **Retrieve Generated Assets:** Once complete, this GET request fetches the final image URLs and metadata.

#### 2.4 Asset Delivery & Logging
The final phase that packages all data and notifies the user.
*   **Nodes Involved:** `Assemble Full Output Package`, `Send Email - Visual Assets Delivery`, `Send Slack - Quick Notification`, `Log to Database`, `Mark Idea as Processed`.
*   **Node Details:**
    *   **Assemble Full Output Package (Code):** Merges the original idea, Claude’s content strategy, and NapkinAI’s visual links into one clean object.
    *   **Send Slack:** Uses Slack Blocks to create a rich notification with "Download" buttons.
    *   **Mark Idea as Processed (Code):** Uses **n8n Static Data** to store a record of the processed `ideaId`. It includes a cleanup routine to remove entries older than 90 days.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Receive Idea | Webhook | External Trigger | - | Merge Input Sources | INPUT TRIGGERS |
| Manual Trigger - Test | Manual Trigger | Testing | - | Set Test Idea Data | INPUT TRIGGERS |
| Set Test Idea Data | Set | Test Data Setup | Manual Trigger | Merge Input Sources | INPUT TRIGGERS |
| Merge Input Sources | Merge | Flow Consolidation | Webhook, Set | Validate & Normalize | INPUT TRIGGERS |
| Validate & Normalize Input | Code | Data Sanitization | Merge | AI - Enrich | INPUT TRIGGERS |
| AI - Enrich & Structure Idea | HTTP Request | AI Analysis | Validate & Normalize | Wait For Data | AI ENRICHMENT (CLAUDE) |
| Wait For Data | Wait | API Buffer | AI - Enrich | Parse Claude | AI ENRICHMENT (CLAUDE) |
| Parse Claude Enrichment | Code | Response Parsing | Wait For Data | NapkinAI - Authenticate | AI ENRICHMENT (CLAUDE) |
| NapkinAI - Authenticate | HTTP Request | Authentication | Parse Claude | NapkinAI - Submit | NAPKINAI VISUAL GENERATION |
| NapkinAI - Submit Idea | HTTP Request | Task Submission | NapkinAI - Auth | NapkinAI - Poll | NAPKINAI VISUAL GENERATION |
| NapkinAI - Poll Status | HTTP Request | Progress Tracking | NapkinAI - Submit, Wait 5s | Check Job Completion | NAPKINAI VISUAL GENERATION |
| Check Job Completion | IF | Loop Logic | NapkinAI - Poll | Retrieve Assets, Wait 5s | NAPKINAI VISUAL GENERATION |
| Wait 5s - Retry Poll | Wait | Polling Delay | Check Job Completion | NapkinAI - Poll | NAPKINAI VISUAL GENERATION |
| NapkinAI - Retrieve Assets | HTTP Request | Data Retrieval | Check Job Completion | Assemble Output | NAPKINAI VISUAL GENERATION |
| Assemble Output Package | Code | Data Consolidation | Retrieve Assets | Email, Slack, Postgres | ASSET DELIVERY & LOGGING |
| Send Email | Email Send | Distribution | Assemble Output | Mark as Processed | ASSET DELIVERY & LOGGING |
| Send Slack | HTTP Request | Notification | Assemble Output | Mark as Processed | ASSET DELIVERY & LOGGING |
| Log to Database | Postgres | Persistent Storage | Assemble Output | Mark as Processed | ASSET DELIVERY & LOGGING |
| Mark Idea as Processed | Code | Internal Logging | Email, Slack, Postgres | - | ASSET DELIVERY & LOGGING |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Webhook** node (POST) for production and a **Manual Trigger** for testing.
    *   Connect a **Set** node to the Manual Trigger with sample keys: `body.title`, `body.rawIdea`, `body.diagramType`, and `body.visualStyle`.
    *   Add a **Merge** node (Mode: Choose Branch) to join them.

2.  **Input Logic:**
    *   Add a **Code** node to validate the text length and assign a unique ID (e.g., `IDEA-{{$now}}`).

3.  **Anthropic Integration:**
    *   Add an **HTTP Request** node. Method: POST. URL: `https://api.anthropic.com/v1/messages`.
    *   Headers: `x-api-key`, `anthropic-version: 2023-06-01`.
    *   Body: Send a prompt asking for JSON structure with keys: `enrichedText`, `napkinPrompt`, `diagramType`, and `contentAngles`.

4.  **NapkinAI Pipeline:**
    *   **Auth:** POST to `/v1/auth/token` with your API Key to get an `accessToken`.
    *   **Submit:** POST to `/v1/visuals/generate`. Pass the `enrichedText` from the AI step in the body.
    *   **Poll & Loop:** Use an HTTP GET node to `/v1/visuals/{{jobId}}/status`. Follow it with an **IF** node checking if `status == completed`. 
    *   **Loop Connection:** Connect the "False" branch of the IF node to a **Wait** node (5 seconds), then connect the Wait node back to the Status Poll node.
    *   **Retrieve:** Connect the "True" branch of the IF node to an HTTP GET node for `/v1/visuals/{{jobId}}/assets`.

5.  **Data Packaging:**
    *   Add a **Code** node to combine the original input, the AI's content strategy, and the NapkinAI download URLs into a single JSON object.

6.  **Delivery & Tracking:**
    *   Configure an **Email** node (SMTP) and a **Slack** node (Webhook) using the download URLs from the previous step.
    *   Add a **Postgres** node to insert the final object into a `napkin_visuals` table.
    *   Add a final **Code** node using `$getWorkflowStaticData('global')` to save the `ideaId` so you can track what has been processed.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **NapkinAI Documentation** | Refer to NapkinAI API docs for latest `diagramType` supported values (flowchart, mindmap, process). |
| **Claude Model Selection** | The workflow uses `claude-sonnet-4-20250514`. Ensure your Anthropic plan supports this version. |
| **Static Data Persistence** | Note that `getWorkflowStaticData` only persists between production executions, not manual test runs. |
| **Database Schema** | Ensure the Postgres table `content.napkin_visuals` exists before running the log node. |