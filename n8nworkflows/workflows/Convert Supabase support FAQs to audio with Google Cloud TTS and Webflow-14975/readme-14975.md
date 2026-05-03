Convert Supabase support FAQs to audio with Google Cloud TTS and Webflow

https://n8nworkflows.xyz/workflows/convert-supabase-support-faqs-to-audio-with-google-cloud-tts-and-webflow-14975


# Convert Supabase support FAQs to audio with Google Cloud TTS and Webflow

# Technical Analysis: Convert Supabase Support FAQs to Audio Workflow

## 1. Workflow Overview
This workflow automates the transformation of text-based Frequently Asked Questions (FAQs) stored in a Supabase database into high-quality audio files using Google Cloud Text-to-Speech (TTS). The generated audio is hosted on a CDN and automatically embedded into a Webflow CMS collection, enabling a multi-modal support experience for end-users.

The workflow is logically divided into five functional blocks:
- **1.1 Input Reception & Data Fetching:** Triggering the process and retrieving pending FAQ records.
- **1.2 Data Refinement & Preparation:** Cleaning text, removing duplicates, and generating SSML (Speech Synthesis Markup Language).
- **1.3 Audio Generation & Hosting:** Converting text to speech and uploading the binary file to a permanent URL.
- **1.4 CMS & Database Synchronization:** Updating the Webflow front-end and marking the record as processed in Supabase.
- **1.5 Notification & Response:** Sending a summary report to Microsoft Teams and responding to the triggering webhook.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Data Fetching
**Overview:** Handles the incoming request to start audio generation and fetches the required records from the database.
- **Nodes Involved:** `Webhook — Receive FAQ Audio Job Request`, `Supabase — Read Unprocessed FAQs`, `Code — Filter & Split FAQ Rows`.
- **Node Details:**
    - **Webhook:** Listens for `POST` requests at `/faq-audio-generate`. It can accept `category` filters or a `force_regenerate` flag.
    - **Supabase (HTTP Request):** Queries the `/rest/v1/faqs` endpoint. It filters for items where `status` is not `audio_published` and limits the batch to 50 items.
    - **Code (Filter & Split):** A JavaScript node that converts the Supabase array into individual n8n items. It implements business logic to skip rows with answers shorter than 20 characters and handles the `force_regenerate` logic.

### 2.2 Data Refinement & Preparation
**Overview:** Ensures data quality and prepares the specific formatting required for natural-sounding AI speech.
- **Nodes Involved:** `Item Lists — Deduplicate by FAQ ID`, `Item Lists — Sort by Category then ID`, `Code — Build SSML for Each FAQ`.
- **Node Details:**
    - **Item Lists (Deduplicate):** Removes duplicate IDs to prevent redundant API calls.
    - **Item Lists (Sort):** Organizes the queue by category and ID.
    - **Code (Build SSML):** Transforms raw text into SSML. It strips HTML, adds `<break>` tags for pauses between questions and answers, and wraps the text in `<prosody>` tags to control the speaking rate (0.95x). It also generates a SEO-friendly filename.

### 2.3 Audio Generation & Hosting
**Overview:** The core processing engine that converts text to binary audio and stores it online.
- **Nodes Involved:** `Loop Over Items`, `Google Cloud TTS — Synthesize FAQ Audio`, `Code — Decode Base64 Audio to Binary`, `UploadToURL — Host FAQ MP3 on CDN`, `Code — Capture Audio URL + Timestamp`.
- **Node Details:**
    - **Loop Over Items:** A Split-in-Batches node that ensures sequential processing to avoid Google Cloud API rate limits.
    - **Google Cloud TTS:** Sends a `POST` request to the synthesize endpoint using the `en-US-Wavenet-D` voice.
    - **Code (Decode Base64):** Converts the `audioContent` (base64 string) returned by Google into a binary buffer compatible with file uploads.
    - **UploadToURL:** Uploads the binary MP3 via `multipart-form-data` to a CDN.
    - **Code (Capture URL):** Extracts the resulting URL from the upload response and adds an ISO timestamp.

### 2.4 CMS & Database Synchronization
**Overview:** Propagates the generated audio URL to the public-facing website and the internal database.
- **Nodes Involved:** `Webflow — Patch FAQ CMS Item with Audio URL`, `Supabase — Write Audio URL & Status Back`.
- **Node Details:**
    - **Webflow (HTTP Request):** Performs a `PATCH` request to the Webflow CMS v2 API. It updates the `audio-embed-url` and sets `audio-published` to true.
    - **Supabase (HTTP Request):** Performs a `PATCH` request to the specific FAQ row to store the `audio_url` and update the status to `audio_published`, preventing the item from being picked up in the next run.

### 2.5 Notification & Response
**Overview:** Finalizes the workflow by alerting the team and confirming the execution to the caller.
- **Nodes Involved:** `Code — Build Teams Card & Summary Object`, `Microsoft Teams — Send FAQ Audio Summary Card`, `Respond to Webhook — Return JSON Summary`.
- **Node Details:**
    - **Code (Build Teams Card):** Aggregates results from all iterations of the loop to create a JSON structure for an Adaptive Card.
    - **Microsoft Teams:** Sends the Adaptive Card via an Incoming Webhook to a specified channel.
    - **Respond to Webhook:** Returns a `200 OK` response containing the count of processed FAQs and the list of generated URLs.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook — Receive... | Webhook | Entry Point | - | Supabase — Read... | Step 1 — Webhook Intake & Supabase FAQ Fetch |
| Supabase — Read... | HTTP Request | Data Fetching | Webhook | Code — Filter... | Step 1 — Webhook Intake & Supabase FAQ Fetch |
| Code — Filter... | Code | Data Validation | Supabase — Read... | Item Lists — Dedup... | Step 1 — Webhook Intake & Supabase FAQ Fetch |
| Item Lists — Dedup... | Item Lists | De-duplication | Code — Filter... | Item Lists — Sort... | Step 2 — Dedup, Sort & Loop Setup |
| Item Lists — Sort... | Item Lists | Sequencing | Item Lists — Dedup... | Code — Build SSML... | Step 2 — Dedup, Sort & Loop Setup |
| Code — Build SSML... | Code | SSML Formatting | Item Lists — Sort... | Loop Over Items | Step 2 — Dedup, Sort & Loop Setup |
| Loop Over Items | Split In Batches | Flow Control | Code — Build SSML... | Google Cloud TTS | Step 2 — Dedup, Sort & Loop Setup |
| Google Cloud TTS | HTTP Request | Voice Synthesis | Loop Over Items | Code — Decode... | Step 3 — Google Cloud TTS & UploadToURL |
| Code — Decode... | Code | Base64 Conversion | Google Cloud TTS | UploadToURL | Step 3 — Google Cloud TTS & UploadToURL |
| UploadToURL | HTTP Request | File Hosting | Code — Decode... | Code — Capture... | Step 3 — Google Cloud TTS & UploadToURL |
| Code — Capture... | Code | Metadata Extraction | UploadToURL | Webflow — Patch... | Step 3 — Google Cloud TTS & UploadToURL |
| Webflow — Patch... | HTTP Request | CMS Update | Code — Capture... | Supabase — Write... | Step 4 — Webflow CMS Update & Supabase Write-Back |
| Supabase — Write... | HTTP Request | DB Update | Webflow — Patch... | Loop Over Items | Step 4 — Webflow CMS Update & Supabase Write-Back |
| Code — Build Teams... | Code | Report Generation | Supabase — Write... | Microsoft Teams | Step 5 — Teams Summary Card & Webhook Response |
| Microsoft Teams | HTTP Request | Notification | Code — Build Teams... | Respond to Webhook | Step 5 — Teams Summary Card & Webhook Response |
| Respond to Webhook | Respond to Webhook | API Response | Microsoft Teams | - | Step 5 — Teams Summary Card & Webhook Response |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger & Retrieval
1. Create a **Webhook Node**: Set HTTP Method to `POST` and path to `faq-audio-generate`. Set Response Mode to `Response Node`.
2. Create an **HTTP Request Node (Supabase Fetch)**: 
    - URL: `https://[PROJECT_REF].supabase.co/rest/v1/faqs`
    - Method: `GET`. 
    - Query Params: `select` (columns list), `status=neq.audio_published`, `order=category.asc,id.asc`, `limit=50`.
    - Headers: `apikey` and `Authorization: Bearer [KEY]`.
3. Create a **Code Node**: Use JavaScript to filter the array, check for `force_regenerate` from the webhook body, and ensure answer length $\ge$ 20 chars.

### Step 2: Processing Queue
4. Create an **Item Lists Node**: Select "Remove Duplicates" based on the `id` field.
5. Create an **Item Lists Node**: Select "Sort" and add `category` and `id` as sort fields.
6. Create a **Code Node**: Implement the SSML wrapper. Use a template string to add `<speak>`, `<prosody>`, and `<break>` tags. Generate a filename using a slugified version of the question.

### Step 3: The Processing Loop
7. Create a **Split In Batches (Loop)** node: Set batch size to 1.
8. Create an **HTTP Request Node (Google TTS)**:
    - URL: `https://texttospeech.googleapis.com/v1/text:synthesize`
    - Method: `POST`.
    - Body: JSON containing `input: { ssml: ... }`, `voice: { languageCode: "en-US", name: "en-US-Wavenet-D" }`, and `audioConfig`.
    - Headers: `X-Goog-Api-Key: [YOUR_KEY]`.
9. Create a **Code Node**: Use `Buffer.from(json.audioContent, 'base64')` to convert the response into a binary file.
10. Create an **HTTP Request Node (UploadToURL)**:
    - URL: `https://upload.uploadtourl.com/api/upload`
    - Method: `POST`.
    - Body Content Type: `multipart-form-data`.
    - Parameters: Add a binary field named `file`.
11. Create a **Code Node**: Extract the `url` from the upload response and add `new Date().toISOString()`.

### Step 4: Updates & Closing
12. Create an **HTTP Request Node (Webflow)**:
    - URL: `https://api.webflow.com/v2/collections/[COLLECTION_ID]/items/[ITEM_ID]`
    - Method: `PATCH`.
    - Body: JSON mapping `audio-embed-url` to the CDN URL.
    - Headers: `Authorization: Bearer [TOKEN]`.
13. Create an **HTTP Request Node (Supabase Update)**:
    - URL: `.../rest/v1/faqs?id=eq.[ID]`
    - Method: `PATCH`.
    - Body: JSON updating `audio_url` and `status` to `audio_published`.
14. Connect the output of the Supabase Update node **back to the Loop Over Items node**.
15. Create a **Code Node**: After the loop, use `.all()` on the "Capture URL" node to aggregate a summary.
16. Create an **HTTP Request Node (Teams)**: `POST` the summary JSON to the Teams Incoming Webhook URL.
17. Create a **Respond to Webhook Node**: Return the summary JSON with a `200` status code.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires `en-US-Wavenet-D` for a professional male voice profile. | Google Cloud TTS Configuration |
| Ensure the `faqs` table in Supabase has the column `webflow_item_id` to map records to the CMS. | Database Schema Requirement |
| Webflow CMS must have a field named `audio-embed-url` (slug) for the integration to work. | Webflow CMS Setup |
| The workflow uses `multipart-form-data` for binary uploads via UploadToURL. | Hosting Integration |