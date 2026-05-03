Generate multilingual AI video clips using WayinVideo and Google Drive

https://n8nworkflows.xyz/workflows/generate-multilingual-ai-video-clips-using-wayinvideo-and-google-drive-14707


# Generate multilingual AI video clips using WayinVideo and Google Drive

### 1. Workflow Overview

The **Multilingual AI Video Clip Distributor** is an automation designed to transform a single long-form video into multiple short-form, vertical clips translated into various target languages. The workflow leverages the WayinVideo API for AI-driven clipping and translation, and Google Drive for final storage.

The logic is organized into four primary functional blocks:
- **1.1 Input Reception & Data Splitting:** Collects project requirements via a form and iterates through a list of target languages.
- **1.2 AI Processing Submission:** Triggers the AI clipping process for each language identified.
- **1.3 Status Polling (The Wait Loop):** Periodically checks the AI job status until the video processing is complete.
- **1.4 File Retrieval & Storage:** Downloads the generated video files and uploads them to a specific Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Splitting
**Overview:** This block captures user requirements and ensures that if multiple languages are requested, the workflow creates a separate execution path for each language.

- **Nodes Involved:** `1. Form — Video URL and Languages`, `2. Code — Split Languages into Items`.
- **Node Details:**
    - **1. Form — Video URL and Languages (Form Trigger):**
        - *Role:* Entry point for the workflow.
        - *Configuration:* Captures `Video URL`, `Project / Brand Name`, `Target Languages` (comma-separated), and `Number of Clips Per Language`.
    - **2. Code — Split Languages into Items (Code):**
        - *Role:* Data transformation.
        - *Logic:* Splits the `Target Languages` string by commas into an array. It maps this array into individual n8n items, allowing subsequent nodes to run once per language.
        - *Key Variables:* `video_url`, `brand_name`, `target_lang`, `clip_limit`.

#### 2.2 AI Processing Submission
**Overview:** Sends the specific video and language requirements to the WayinVideo API to start the clipping and translation process.

- **Nodes Involved:** `3. WayinVideo — Submit Clipping Task`.
- **Node Details:**
    - **3. WayinVideo — Submit Clipping Task (HTTP Request):**
        - *Role:* API Integration.
        - *Configuration:* `POST` request to `/api/v2/clips`.
        - *Key Settings:* 
            - Ratio: `RATIO_9_16` (Vertical).
            - Duration: `DURATION_30_60`.
            - Resolution: `HD_720`.
            - Captions: Enabled with translation mode.
        - *Potential Failures:* Invalid API Token (Auth error), Invalid Video URL, or API Rate Limiting.

#### 2.3 Status Polling (Wait & Check)
**Overview:** Since AI video generation is asynchronous, this block implements a polling mechanism to wait until the processing status is "SUCCEEDED".

- **Nodes Involved:** `4. Wait — 45 Seconds1`, `5. WayinVideo — Get Clip Results`, `6. IF — Check Status SUCCEEDED`, `7. Wait — 30 Seconds Retry`.
- **Node Details:**
    - **4. Wait — 45 Seconds1 (Wait):** Initial pause to allow the AI to start processing.
    - **5. WayinVideo — Get Clip Results (HTTP Request):** 
        - *Role:* Polling.
        - *Configuration:* `GET` request using the job ID retrieved from node 3.
    - **6. IF — Check Status SUCCEEDED (IF):** 
        - *Logic:* Checks if `{{ $json.data.status }} === 'SUCCEEDED'`.
        - *True Path:* Proceeds to download.
        - *False Path:* Routes to the retry wait node.
    - **7. Wait — 30 Seconds Retry (Wait):** Pause before requesting the status again.
    - *Edge Case:* **Infinite Loop Risk.** If the job fails or hangs, the workflow will loop forever between nodes 5, 6, and 7.

#### 2.4 File Retrieval & Storage
**Overview:** Once the AI job is complete, this block iterates through the generated clips, downloads the binaries, and saves them to Google Drive.

- **Nodes Involved:** `8. Code — Extract Clips Array`, `9. HTTP — Download Clip File`, `10. Google Drive — Upload Clip`.
- **Node Details:**
    - **8. Code — Extract Clips Array (Code):**
        - *Role:* Flattens the `clips` array from the API response into individual n8n items so each clip can be processed.
    - **9. HTTP — Download Clip File (HTTP Request):**
        - *Role:* Binary retrieval.
        - *Configuration:* Uses the `export_link` from the AI result; response format set to `file`.
    - **10. Google Drive — Upload Clip (Google Drive):**
        - *Role:* File storage.
        - *Configuration:* Uploads the binary file using the AI-generated `title` as the filename to a specific `folderId`.
        - *Credentials:* Requires Google Drive OAuth2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form — Video URL... | Form Trigger | User Input | None | 2. Code | Section 1 — Form Input and Language Split |
| 2. Code — Split... | Code | Data Iterator | 1. Form | 3. WayinVideo | Section 1 — Form Input and Language Split |
| 3. WayinVideo — Submit... | HTTP Request | Job Submission | 2. Code | 4. Wait | Section 2 — Submit Clipping Job |
| 4. Wait — 45 Seconds1 | Wait | Initial Delay | 3. WayinVideo | 5. WayinVideo | Section 3 — Wait and Check Status |
| 5. WayinVideo — Get... | HTTP Request | Status Check | 4. Wait / 7. Wait | 6. IF | Section 3 — Wait and Check Status |
| 6. IF — Check Status... | IF | Logic Gate | 5. WayinVideo | 8. Code / 7. Wait | Section 3 — Wait and Check Status |
| 7. Wait — 30 Seconds... | Wait | Retry Delay | 6. IF | 5. WayinVideo | Section 3 — Wait and Check Status |
| 8. Code — Extract... | Code | Array Flattener | 6. IF | 9. HTTP | Section 4 — Download and Upload |
| 9. HTTP — Download... | HTTP Request | File Downloader | 8. Code | 10. Google Drive | Section 4 — Download and Upload |
| 10. Google Drive — Upload | Google Drive | File Storage | 9. HTTP | None | Section 4 — Download and Upload |

---

### 4. Reproducing the Workflow from Scratch

1.  **Input Trigger:**
    - Create a **Form Trigger** node.
    - Add fields: `Video URL` (Text), `Project / Brand Name` (Text), `Target Languages` (Text), `Number of Clips Per Language` (Number).
2.  **Language Splitting:**
    - Add a **Code** node. Use JavaScript to `.split(',')` the `Target Languages` field and return an array of objects containing the URL, Brand, Language, and Limit.
3.  **AI Submission:**
    - Add an **HTTP Request** node.
    - Method: `POST`. URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`.
    - Headers: `Authorization: Bearer [YOUR_TOKEN]`, `x-wayinvideo-api-version: v2`, `Content-Type: application/json`.
    - Body: JSON format including `video_url`, `project_name`, `target_duration: "DURATION_30_60"`, `limit`, `target_lang`, and `ratio: "RATIO_9_16"`.
4.  **The Polling Loop:**
    - Add a **Wait** node set to 45 seconds.
    - Add an **HTTP Request** node (`GET`) to `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{job_id}}` with the same headers as step 3.
    - Add an **IF** node checking if `data.status` equals `SUCCEEDED`.
    - **False Path:** Connect the IF node to another **Wait** node (30 seconds), then loop back to the "Get Clip Results" HTTP Request.
5.  **Data Processing:**
    - **True Path:** Connect the IF node to a **Code** node. Use JavaScript to map the `data.clips` array into a list of individual items (title, export\_link).
6.  **File Delivery:**
    - Add an **HTTP Request** node. Set URL to `{{export_link}}` and Response Format to `File`.
    - Add a **Google Drive** node. Action: `Upload`.
    - Set File Name to `{{title}}` and provide your `Folder ID` and OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Infinite Loop Risk** | Critical: If the API never returns "SUCCEEDED", the workflow loops forever. Implementation of a "Retry Counter" is recommended. |
| **Customization** | Change `target_duration` to `DURATION_15_30` for shorter clips or `ratio` to `RATIO_1_1` for square videos. |
| **Credential Setup** | Requires WayinVideo API Token and Google Drive OAuth2 API credentials. |