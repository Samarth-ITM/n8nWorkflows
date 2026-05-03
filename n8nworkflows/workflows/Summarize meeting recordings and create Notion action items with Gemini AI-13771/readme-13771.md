Summarize meeting recordings and create Notion action items with Gemini AI

https://n8nworkflows.xyz/workflows/summarize-meeting-recordings-and-create-notion-action-items-with-gemini-ai-13771


# Summarize meeting recordings and create Notion action items with Gemini AI

This document provides a technical breakdown and implementation guide for the **Summarize meeting recordings and create Notion action items with Gemini AI** n8n workflow.

---

### 1. Workflow Overview

This workflow automates the transition from a raw meeting recording to a structured knowledge base. It eliminates manual note-taking by leveraging Google Gemini's multimodal capabilities to "listen" to audio/video files, extract key insights, and distribute them to productivity tools.

**Logical Blocks:**
*   **1.1 Input Reception:** A user-facing web form to ingest the recording file and meeting metadata.
*   **1.2 AI Processing:** A two-stage interaction with Google Gemini API to upload the file and perform a structured analysis.
*   **1.3 Data Transformation:** A logic layer that parses the AI's raw response into clean, formatted data structures.
*   **1.4 Storage & Notification:** Synchronous updates to Notion (for documentation) and Slack (for immediate team visibility).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Captures the source material via an n8n-hosted form.
*   **Nodes Involved:** `Upload meeting recording` (Form Trigger).
*   **Node Details:**
    *   **Type:** Form Trigger (v2).
    *   **Configuration:** Defines two fields: a required file upload (`Meeting Recording`) and a text field (`Meeting Title`).
    *   **Edge Cases:** Large file uploads may hit n8n instance limits or timeout settings. Ensure the file type is supported by Gemini (MP3, WAV, MP4, etc.).

#### 2.2 AI Processing
*   **Overview:** Handles the communication with Google Gemini’s File and Generative APIs.
*   **Nodes Involved:** `Upload recording to Gemini`, `Analyze recording with Gemini`.
*   **Node Details:**
    *   **Upload recording to Gemini (HTTP Request):** 
        *   **Role:** Performs a POST request to the Google Files API (`upload/v1beta/files`).
        *   **Configuration:** Uses `googlePalmApi` credentials. Sends binary data using `X-Goog-Upload-Command: upload, finalize`.
        *   **Failures:** Invalid credentials or unsupported file MIME types.
    *   **Analyze recording with Gemini (HTTP Request):**
        *   **Role:** Prompts the `gemini-2.0-flash` model to analyze the uploaded file URI.
        *   **Configuration:** Sends a JSON payload requesting a summary, decisions, action items (with assignees), and follow-ups.
        *   **Key Expression:** `{{ $json.file.uri }}` passed from the previous node.
        *   **Failures:** Model safety filters triggering on meeting content or token limit exhaustion for very long meetings.

#### 2.3 Data Transformation
*   **Overview:** Converts the AI's JSON-string output into usable variables for downstream integrations.
*   **Nodes Involved:** `Format meeting notes` (Code).
*   **Node Details:**
    *   **Type:** Code Node (JavaScript).
    *   **Configuration:** Extracts the text from the Gemini response object, parses the inner JSON string, and builds a Markdown checklist for action items.
    *   **Logic:** Includes `try/catch` blocks to handle instances where Gemini might return malformed JSON or empty responses. It also calculates the current timestamp for the Notion entry.

#### 2.4 Storage & Notification
*   **Overview:** Distributes the processed information to the team.
*   **Nodes Involved:** `Create notes in Notion`, `Notify team on Slack`.
*   **Node Details:**
    *   **Create notes in Notion:**
        *   **Role:** Appends a new page to a specified Database.
        *   **Requirement:** Requires a Notion Database ID and mapping for Title, Summary, and Action Items.
    *   **Notify team on Slack:**
        *   **Role:** Sends a formatted summary and the action item list to a specific channel.
        *   **Configuration:** Uses a block-style text format including a link to the Notion page.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Upload meeting recording | Form Trigger | Trigger / Data Input | None | Upload recording to Gemini | Upload recording: Collects the audio file and meeting title from a web form. |
| Upload recording to Gemini | HTTP Request | File Hosting (API) | Upload meeting recording | Analyze recording with Gemini | AI processing: Uploads the file to Gemini and generates a structured meeting summary. |
| Analyze recording with Gemini | HTTP Request | LLM Inference | Upload recording to Gemini | Format meeting notes | AI processing: Uploads the file to Gemini and generates a structured meeting summary. |
| Format meeting notes | Code | Data Parsing | Analyze recording with Gemini | Create notes in Notion | Save and notify: Creates a Notion page with the summary and alerts the team on Slack. |
| Create notes in Notion | Notion | Documentation | Format meeting notes | Notify team on Slack | Save and notify: Creates a Notion page with the summary and alerts the team on Slack. |
| Notify team on Slack | Slack | Team Alert | Create notes in Notion | None | Save and notify: Creates a Notion page with the summary and alerts the team on Slack. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Form Trigger** node.
    *   Set the Title to "Meeting Notes Processor".
    *   Add a File field (Label: "Meeting Recording") and a Text field (Label: "Meeting Title").
2.  **Google Gemini Integration (Step 1):**
    *   Create an **HTTP Request** node.
    *   Method: `POST`. URL: `https://generativelanguage.googleapis.com/upload/v1beta/files`.
    *   Authentication: Predefined Credential Type (`googlePalmApi`).
    *   Headers: `X-Goog-Upload-Command: upload, finalize`, `X-Goog-Upload-Protocol: raw`.
    *   Body: Select `Binary Data` and set "Input Data Field Name" to `Meeting Recording`.
3.  **Google Gemini Integration (Step 2):**
    *   Create an **HTTP Request** node.
    *   Method: `POST`. URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`.
    *   Body: JSON. Include the `fileUri` from the previous node and a prompt requesting specific JSON keys: `summary`, `decisions`, `actionItems`, `followUps`.
4.  **Formatting Logic:**
    *   Create a **Code** node.
    *   Use JavaScript to parse `candidates[0].content.parts[0].text`.
    *   Map `actionItems` into a Markdown string (e.g., `- [ ] Task Name (@assignee)`).
5.  **Notion Setup:**
    *   Create a **Notion** node. Resource: `Database Page`, Operation: `Create`.
    *   Map the "Title" to the meeting title and "Summary" to the parsed summary.
6.  **Slack Notification:**
    *   Create a **Slack** node. Operation: `Post Message`.
    *   Select a channel. Format the text to include the summary (truncated if necessary) and the full action item list.
7.  **Connections:**
    *   Connect the nodes linearly as per the sequence above.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup requirements** | Add Google Gemini API, Notion API, and Slack OAuth2 credentials. |
| **Notion Database Structure** | Ensure columns exist: Title (Title), Date (Date), Summary (Text), Action Items (Text). |
| **Gemini API Documentation** | For file upload limits: [Google AI Studio Docs](https://ai.google.dev/gemini-api/docs/prompting_with_media) |