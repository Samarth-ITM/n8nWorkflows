Generate AI songs from text prompts with Suno, OpenAI, Google Drive and Slack

https://n8nworkflows.xyz/workflows/generate-ai-songs-from-text-prompts-with-suno--openai--google-drive-and-slack-13826


# Generate AI songs from text prompts with Suno, OpenAI, Google Drive and Slack

# Reference Document: Suno AI Music Generation Workflow

## 1. Workflow Overview
This workflow automates the end-to-end process of creating custom AI-generated music. It transforms a simple text prompt into a fully produced song, handles the asynchronous generation process, stores the resulting audio file in the cloud, and logs the metadata for tracking.

The logic is divided into four functional stages:
*   **1.1 Input Reception & Configuration:** Captures the initial prompt via Webhook or Schedule and sets global variables (genre, mood, storage paths).
*   **1.2 AI Prompt Enrichment:** Utilizes OpenAI to expand the user's basic prompt into a detailed musical description optimized for the Suno API.
*   **1.3 Suno Generation & Polling:** Submits the generation request to the Suno API (via RapidAPI) and waits for the processing to complete.
*   **1.4 Storage & Notification:** Downloads the resulting audio, uploads it to Google Drive, logs entry details in Google Sheets, and sends a final confirmation to Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Prompt Input
This block handles the entry point of the workflow and defines the parameters used throughout the execution.

*   **Nodes Involved:** Webhook: Receive Prompt, Daily Batch (fallback), Set Music Config.
*   **Node Details:**
    *   **Webhook / Schedule Trigger:** Standard entry points. The Webhook accepts POST requests at the `/suno-generate-song` path.
    *   **Set Music Config (Set):** Defines static variables like `genre`, `mood`, `storage_folder`, and the `log_sheet_id`. It also provides a `default_prompt` if the trigger doesn't provide one.
    *   **Edge Cases:** If the Webhook body is empty, the downstream AI enrichment might fail unless the `default_prompt` is correctly referenced.

### 2.2 Enrichment Phase
Enhances the raw user input into a format that Suno can better interpret (e.g., adding "style tags").

*   **Nodes Involved:** OpenAI: Enrich Prompt.
*   **Node Details:**
    *   **OpenAI: Enrich Prompt (OpenAI):** Uses a Chat/Conversation resource to transform the input prompt into a rich musical description.
    *   **Configuration:** Requires OpenAI credentials.
    *   **Output:** A JSON object containing the enriched text used as the "message content" for the Suno API.

### 2.3 Suno Phase
The core integration with the music generation engine.

*   **Nodes Involved:** Suno: Submit Generation Job, Wait 30s (Polling Delay), Suno: Poll Song Status, Song Ready?.
*   **Node Details:**
    *   **Suno: Submit Generation Job (HTTP Request):** Sends a POST request to `suno-api.p.rapidapi.com/generate`.
        *   **Key Params:** `prompt`, `tags`, `make_instrumental: false`.
        *   **Auth:** Requires a RapidAPI Key.
    *   **Wait 30s (Wait):** A mandatory pause to allow the AI model to process the audio.
    *   **Suno: Poll Song Status (HTTP Request):** A GET request to check the status of the specific Job ID returned by the previous node.
    *   **Song Ready? (If):** Checks if the `status` field equals "complete".
    *   **Failure Case:** If the status is not "complete" after the first poll, the workflow proceeds to "Notify Failure". *Note: This implementation does not currently loop back to poll again.*

### 2.4 Storage & Output
Finalizes the workflow by moving the binary data to cloud storage and notifying the team.

*   **Nodes Involved:** Download Audio File, Google Drive: Save Song, Google Sheets: Log Song, Notify Slack, Notify Failure.
*   **Node Details:**
    *   **Download Audio File (HTTP Request):** Performs a GET request on the `audio_url` provided by Suno, set to "File" response format.
    *   **Google Drive: Save Song (Google Drive):** Uploads the binary file to a specific folder. Filename is generated using the song title or a timestamp.
    *   **Google Sheets: Log Song (Google Sheets):** Appends a row with the Title, Prompt, Song ID, Duration, and URLs for future reference.
    *   **Notify Slack (Slack):** Sends a formatted message to the `#music-generation` channel with the song link and confirmation of storage.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook: Receive Prompt | Webhook | Trigger | None | Set Music Config | 1. Trigger & Prompt Input |
| Daily Batch (fallback) | Schedule | Trigger | None | Set Music Config | 1. Trigger & Prompt Input |
| Set Music Config | Set | Configuration | Webhook, Daily Batch | OpenAI: Enrich Prompt | 1. Trigger & Prompt Input |
| OpenAI: Enrich Prompt | OpenAI | Enrichment | Set Music Config | Suno: Submit Job | 2. Prompt Enrichment |
| Suno: Submit Job | HTTP Request | Generation | OpenAI | Wait 30s | 3. Suno Generation & Polling |
| Wait 30s | Wait | Delay | Suno: Submit Job | Suno: Poll Status | 3. Suno Generation & Polling |
| Suno: Poll Status | HTTP Request | Polling | Wait 30s | Song Ready? | 3. Suno Generation & Polling |
| Song Ready? | If | Logic Gate | Suno: Poll Status | Download / Notify Failure | 3. Suno Generation & Polling |
| Download Audio File | HTTP Request | Data Retrieval | Song Ready? | Google Drive | 4. Store, Log & Notify |
| Google Drive: Save Song | Google Drive | Storage | Download Audio | Google Sheets | 4. Store, Log & Notify |
| Google Sheets: Log Song | Google Sheets | Logging | Google Drive | Notify Slack | 4. Store, Log & Notify |
| Notify Slack | Slack | Success Alert | Google Sheets | None | 4. Store, Log & Notify |
| Notify Failure | Slack | Error Alert | Song Ready? | None | 4. Store, Log & Notify |

---

## 4. Reproducing the Workflow from Scratch

1.  **Triggers:** Create a **Webhook** node (POST) and a **Schedule** node (24h interval).
2.  **Config:** Connect both to a **Set** node. Add string values for `genre`, `mood`, `storage_folder` (folder ID), and `log_sheet_id`.
3.  **Enrichment:** Add an **OpenAI** node. Use a "Chat" resource. Configure the prompt to take the input from the Set node and request "style tags" and a "refined song description."
4.  **Suno Generation:** 
    *   Add an **HTTP Request** node. Set Method to `POST`, URL to `https://suno-api.p.rapidapi.com/generate`.
    *   Headers: Add `x-rapidapi-key` and `x-rapidapi-host`.
    *   Body: Pass the enriched prompt from OpenAI.
5.  **Polling:** 
    *   Add a **Wait** node set to 30 seconds.
    *   Add another **HTTP Request** node. Set Method to `GET`, URL to `https://suno-api.p.rapidapi.com/get?ids={{ $node["Suno: Submit Job"].json[0].id }}`.
6.  **Logic:** Add an **If** node. Condition: `{{ $json[0].status }}` equals `complete`.
7.  **File Retrieval:** Connect the "True" output of the If node to an **HTTP Request** node. Method: `GET`. URL: `{{ $json[0].audio_url }}`. Set Response Format to "File".
8.  **Storage:** 
    *   Add a **Google Drive** node (Upload). Map the Folder ID from the Set node and the file from the previous HTTP node.
    *   Add a **Google Sheets** node (Append). Map fields for Title, Prompt, and the URL.
9.  **Alerts:** 
    *   Connect the Sheets node to a **Slack** node for success.
    *   Connect the "False" output of the **If** node to a second **Slack** node for failure notifications.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Guide** | Ensure Suno API access via RapidAPI is active before testing. |
| **Storage Note** | The Google Drive folder must exist and be accessible via the provided Folder ID. |
| **Polling Limitation** | This workflow polls once after 30 seconds. For longer songs, consider adding a recursive loop. |
| **Prompt Enrichment** | OpenAI is used to ensure the Suno API receives highly specific "tags" for better musical quality. |

***

*disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n. Ce traitement respecte strictement les politiques de contenu en vigueur.*