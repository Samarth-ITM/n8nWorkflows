Parse Google Drive documents to RAG-ready Markdown with Landing.ai and Supabase cache

https://n8nworkflows.xyz/workflows/parse-google-drive-documents-to-rag-ready-markdown-with-landing-ai-and-supabase-cache-13244


# Parse Google Drive documents to RAG-ready Markdown with Landing.ai and Supabase cache

This document provides a technical breakdown of the n8n workflow designed to automate the conversion of Google Drive documents into RAG-ready Markdown using Landing.ai, featuring a Supabase-based caching mechanism to optimize API usage.

### 1. Workflow Overview

The workflow automates the ingestion, parsing, and storage of documents from Google Drive. It is designed for high-volume RAG (Retrieval-Augmented Generation) pipelines where processing costs and efficiency are critical.

**The logic is organized into four main functional blocks:**
1.  **File Acquisition & Metadata:** Monitors Google Drive for new files, downloads them, and establishes a canonical metadata schema.
2.  **Cache Verification:** Checks a Supabase database to see if the file (based on its unique ID) has already been processed to prevent duplicate API costs.
3.  **Asynchronous AI Parsing:** Submits new files to Landing.ai's document parsing engine and enters a polling loop to wait for completion.
4.  **Data Storage & Finalization:** Retrieves the Markdown result, handles large file outputs via external URLs if necessary, and caches the final result in Supabase.

---

### 2. Block-by-Block Analysis

#### 2.1 File Acquisition & Metadata
This block handles the entry point and ensures the binary data is ready for processing.
*   **Nodes Involved:** `Google Drive Trigger`, `Download File`, `Build Metadata`.
*   **Logic:** 
    *   The trigger polls a specific Google Drive folder every minute.
    *   `Build Metadata` uses a Code node to create a consistent JSON object (`file_id`, `document_name`, `mime_type`, etc.) that persists throughout the workflow, ensuring downstream nodes always have access to the original file context.

#### 2.2 Cache Verification
Determines if the workflow should proceed or terminate early.
*   **Nodes Involved:** `Check Cache`, `Debug Cache Results`, `If Cache Exists`, `STOP — Cached`.
*   **Logic:**
    *   Queries Supabase table `landing_parse_cache` using the `file_id`.
    *   If a record is found, the workflow moves to a "Success" stop node without calling the AI API, saving time and tokens.

#### 2.3 AI Processing Loop
Submits the document and manages the asynchronous nature of the Landing.ai API.
*   **Nodes Involved:** `Submit to Landing.ai`, `Merge Metadata + Job`, `Init Loop State`, `wait till interval set time`, `Poll Job Status`, `If Completed`, `If Timeout`.
*   **Logic:**
    *   **Submission:** Sends the binary file to Landing.ai using `multipart-form-data`.
    *   **Polling Logic:** Since parsing can take time (especially for 200+ page docs), the workflow enters a loop. It waits for a defined interval (default 10s) before checking the job status.
    *   **Loop Control:** `Init Loop State` sets a maximum of 20 retries and a 1-hour timeout safety net to prevent infinite loops.

#### 2.4 Retrieval & Storage
Handles the parsed Markdown output and saves it to the database.
*   **Nodes Involved:** `check-large-file`, `HTTP Request1`, `Save Parse Result`, `Log Timeout`, `STOP — Success`, `STOP — Timeout`.
*   **Logic:**
    *   **Large File Handling:** Landing.ai may return the Markdown directly or via an `output_url` for large files. `check-large-file` branches the logic to fetch data from the URL if needed.
    *   **Persistence:** The final Markdown content is saved to Supabase alongside the workflow execution ID and timestamps.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Google Drive Trigger | Google Drive Trigger | Entry Point (Polling) | - | Download File | Fetch file from Google drive |
| Download File | Google Drive | File Download | Google Drive Trigger | Build Metadata | Fetch file from Google drive |
| Build Metadata | Code | Data Normalization | Download File | Check Cache, Merge Metadata + Job | Fetch file from Google drive |
| Check Cache | Supabase | Cache Lookup | Build Metadata | Debug Cache Results | Check Cache |
| Debug Cache Results | Code | Cache Logic Helper | Check Cache | If Cache Exists | Check Cache |
| If Cache Exists | If | Branching Logic | Debug Cache Results | STOP — Cached, Submit to Landing.ai | - |
| STOP — Cached | Code | Early Exit | If Cache Exists | - | - |
| Submit to Landing.ai | HTTP Request | AI Submission | If Cache Exists | Merge Metadata + Job | Get Parse data |
| Merge Metadata + Job | Merge | Data Synchronization | Build Metadata, Submit to Landing.ai | Init Loop State | Get Parse data |
| Init Loop State | Code | Loop Config | Merge Metadata + Job | wait till interval set time | Get Parse data |
| wait till interval set time | Wait | Polling Delay | Init Loop State, If Timeout | Poll Job Status | Get Parse data |
| Poll Job Status | HTTP Request | Status Check | wait till interval set time | If Completed | Get Parse data |
| If Completed | If | Loop Exit Check | Poll Job Status | check-large-file, If Timeout | Get Parse data |
| If Timeout | If | Error Handling | If Completed | Log Timeout, wait till interval set time | Get Parse data |
| check-large-file | If | Output Format Check | If Completed | HTTP Request1, Save Parse Result | Get Parse data |
| HTTP Request1 | HTTP Request | Large File Fetch | check-large-file | Save Parse Result | Get Parse data |
| Save Parse Result | Supabase | Data Persistence | check-large-file, HTTP Request1 | STOP — Success | Get Parse data |
| Log Timeout | Supabase | Error Logging | If Timeout | STOP — Timeout | Get Parse data |
| STOP — Success | Code | Workflow Success | Save Parse Result | - | Get Parse data |
| STOP — Timeout | Code | Workflow Failure | Log Timeout | - | Get Parse data |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Google Drive Trigger** node. Set the event to `File Created` and point it to a specific Folder ID. Set polling to your preferred frequency (e.g., 1 minute).
2.  **File Download:** Connect a **Google Drive** node, set the operation to `Download`, and use the File ID from the trigger: `{{ $json.id }}`.
3.  **Metadata Prep:** Add a **Code** node to extract `file_id`, `name`, `mimeType`, and `size`. Return these as JSON while passing the binary data forward.
4.  **Database Connection:** Create a **Supabase** node. Configure it to `Get All` from table `landing_parse_cache` where `file_id` equals your metadata ID. Enable "Always Output Data" and "Continue on Fail".
5.  **Logic Branching:** Use an **If** node to check if the Supabase result contains a `file_id`. If true, route to a **Code** node that logs "File already in cache" and stops.
6.  **AI Submission:** On the "False" path, add an **HTTP Request** node.
    *   **Method:** POST
    *   **URL:** `https://api.va.landing.ai/v1/ade/parse/jobs`
    *   **Auth:** Bearer Token (Landing.ai API Key)
    *   **Body:** `multipart-form-data` with a parameter `document` (type: formBinaryData) and `model` set to `dpt-2-latest`.
7.  **Loop Initialization:** Add a **Code** node to set loop variables: `max_retries = 20` and `poll_interval_sec = 10`.
8.  **Wait & Poll:** Add a **Wait** node (using the interval variable) followed by an **HTTP Request** (GET) to `https://api.va.landing.ai/v1/ade/parse/jobs/{{job_id}}`.
9.  **Conditional Loop:** Use an **If** node to check if `status == "completed"`.
    *   If **No**: Check if retries are exceeded. If not, loop back to the **Wait** node.
    *   If **Yes**: Check if the response contains an `output_url`. If it does, use another **HTTP Request** to download the Markdown from that URL.
10. **Storage:** Add a **Supabase** node to `Insert` the final Markdown, `file_id`, and `document_name` into your cache table.
11. **Final Logs:** Use **Code** nodes at the end of both success and failure paths to print status messages to the console for debugging.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Landing.ai API Key Generation | [Landing.ai API Docs](https://docs.landing.ai/ade/agentic-api-key) |
| Supabase Table Schema | Required fields: `file_id`, `document_name`, `mime_type`, `file_size_bytes`, `job_id`, `job_status`, `markdown`, `uploaded_at`, `workflow_run_id` |
| Large Document Support | Handles files over 200 pages by using asynchronous polling and external URL retrieval. |
| Use Case: RAG | Optimized for building LLM-ready knowledge bases from unstructured Drive data. |