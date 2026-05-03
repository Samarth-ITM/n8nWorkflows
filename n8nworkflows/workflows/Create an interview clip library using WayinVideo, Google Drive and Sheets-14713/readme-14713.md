Create an interview clip library using WayinVideo, Google Drive and Sheets

https://n8nworkflows.xyz/workflows/create-an-interview-clip-library-using-wayinvideo--google-drive-and-sheets-14713


# Create an interview clip library using WayinVideo, Google Drive and Sheets

### 1. Workflow Overview

The **Interview Clip Library Builder** is an automation designed for podcast producers and content teams to extract highlight clips from long-form interview recordings using AI. The workflow transforms a simple form submission into a curated library of video clips stored in Google Drive and indexed in Google Sheets.

The logic is organized into the following functional blocks:
- **1.1 Input Reception:** Captures recording metadata and the specific search query via an n8n Form.
- **1.2 AI Processing & Polling:** Interfaces with the WayinVideo API to identify relevant moments, implementing a wait-and-retry loop until the AI processing is complete.
- **1.3 Clip Download and Storage:** Iterates through the discovered clips, downloads the binary files, uploads them to Google Drive, and logs the metadata to a master Google Sheet.
- **1.4 Notification:** Sends a formatted HTML summary email to the user once the library is updated.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Collects all necessary parameters to start the AI search and identify where to send the results.
- **Nodes Involved:** `1. Form — Interview URL + Details`
- **Node Details:**
    - **Type:** Form Trigger
    - **Configuration:** Defines five required fields: Recording URL, Guest Name, Topic/Category, Moments to Extract (Query), and Recipient Email.
    - **Input/Output:** Trigger $\rightarrow$ `2. WayinVideo — Submit Find Moments`.
    - **Edge Cases:** If a user provides a non-public URL or a vague query, the subsequent AI step may fail or return empty results.

#### 2.2 WayinVideo Processing
- **Overview:** Handles the asynchronous nature of AI video processing by submitting a request and polling for completion.
- **Nodes Involved:** `2. WayinVideo — Submit Find Moments`, `3. Wait — 90 Seconds`, `4. WayinVideo — Get Clips Result`, `5. IF — Status SUCCEEDED?`, `6. Wait — 30 Seconds Retry`.
- **Node Details:**
    - **2. WayinVideo — Submit Find Moments (HTTP Request):** Sends a POST request to `/clips/find-moments`. It passes the video URL and query. Configuration includes HD_720 resolution and captioning enabled.
    - **3. Wait — 90 Seconds (Wait):** A hard pause to allow the AI to initialize processing.
    - **4. WayinVideo — Get Clips Result (HTTP Request):** A GET request using the `id` returned by the first node to check the current status of the clips.
    - **5. IF — Status SUCCEEDED? (If):** A conditional check. If `data.status` equals `SUCCEEDED`, it proceeds to extraction; otherwise, it triggers a retry.
    - **6. Wait — 30 Seconds Retry (Wait):** Prevents API rate-limiting during the polling loop before returning to node 4.
    - **Potential Failures:** Incorrect API tokens in headers or invalid video URLs will cause the HTTP request to fail.

#### 2.3 Clip Download and Storage
- **Overview:** Processes the array of found clips and moves them from the AI cloud to the user's permanent storage.
- **Nodes Involved:** `7. Code — Split Each Clip`, `8. HTTP — Download Clip File`, `9. Google Drive — Upload Clip`, `10. Google Sheets — Save to Library`.
- **Node Details:**
    - **7. Code — Split Each Clip (Code):** Uses JavaScript to map the `clips` array into individual n8n items, ensuring each clip is processed independently in the following nodes.
    - **8. HTTP — Download Clip File (HTTP Request):** Fetches the binary video file from the `export_link`. Response format is set to `file`.
    - **9. Google Drive — Upload Clip (Google Drive):** Uploads the binary file. The filename is dynamically generated using the Guest Name, Clip Index, and AI Score.
    - **10. Google Sheets — Save to Library (Google Sheets):** Appends a row to a sheet named "Interview Clip Library". It maps a wide range of data: timestamps (converted from ms to seconds), tags, scores, and the Google Drive link.
    - **Edge Cases:** Export links from WayinVideo expire after 24 hours. If the workflow is paused, the download will fail.

#### 2.4 Email Summary
- **Overview:** Notifies the requester that the process is complete.
- **Nodes Involved:** `11. Gmail — Send Summary Email`
- **Node Details:**
    - **Type:** Gmail Node
    - **Configuration:** Sends a professionally styled HTML email. It includes a link back to the original interview and a prompt to check the Google Sheet for the specific clip links.
    - **Input/Output:** `10. Google Sheets — Save to Library` $\rightarrow$ Gmail.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form — Interview URL + Details | Form Trigger | User Input | - | 2. WayinVideo... | ## Form Input... User submits... |
| 2. WayinVideo — Submit Find Moments | HTTP Request | AI Request | 1. Form... | 3. Wait — 90s | ## WayinVideo Processing... Submits video... |
| 3. Wait — 90 Seconds | Wait | Delay | 2. WayinVideo... | 4. WayinVideo... | ## WayinVideo Processing... |
| 4. WayinVideo — Get Clips Result | HTTP Request | Status Polling | 3. Wait / 6. Wait | 5. IF — Status... | ## WayinVideo Processing... |
| 5. IF — Status SUCCEEDED? | If | Logic Gate | 4. WayinVideo... | 7. Code / 6. Wait | ## WayinVideo Processing... |
| 6. Wait — 30 Seconds Retry | Wait | Retry Delay | 5. IF — Status... | 4. WayinVideo... | ## WayinVideo Processing... |
| 7. Code — Split Each Clip | Code | Data Formatting | 5. IF — Status... | 8. HTTP — Down... | ## Clip Download and Storage... |
| 8. HTTP — Download Clip File | HTTP Request | File Retrieval | 7. Code... | 9. Google Drive... | ## Clip Download and Storage... / ⚠️ Export Links Expire in 24h |
| 9. Google Drive — Upload Clip | Google Drive | File Storage | 8. HTTP — Down... | 10. Google Sheets... | ## Clip Download and Storage... |
| 10. Google Sheets — Save to Library | Google Sheets | Metadata Logging | 9. Google Drive... | 11. Gmail... | ## Clip Download and Storage... |
| 11. Gmail — Send Summary Email | Gmail | Notification | 10. Google Sheets... | - | ## Email Summary... Sends confirmation... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Input Setup:**
    - Create a **Form Trigger** node. Add five fields: `Interview Recording URL`, `Guest Name`, `Interview Topic / Category`, `What Moments To Extract`, and `Send Email To`.
2.  **AI Request:**
    - Create an **HTTP Request** node (POST).
    - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments`
    - Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `x-wayinvideo-api-version: v2`, `Content-Type: application/json`.
    - Body: JSON containing `video_url`, `query`, `project_name` (mapped from form), and `enable_export: true`.
3.  **Polling Loop:**
    - Add a **Wait** node set to 90 seconds.
    - Add an **HTTP Request** node (GET) to: `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments/results/{{ $json.data.id }}`.
    - Add an **If** node checking if `{{ $json.data.status }}` equals `SUCCEEDED`.
    - Connect the **False** output of the If node to a **Wait** node (30 seconds), then loop back to the "Get Clips Result" HTTP node.
4.  **Data Extraction:**
    - Add a **Code** node. Use JavaScript to return the `clips` array from the AI response as separate items.
5.  **Storage Pipeline:**
    - Add an **HTTP Request** node. Set URL to `{{ $json.export_link }}` and Response Format to `File`.
    - Add a **Google Drive** node. Operation: `Upload`. Set the folder ID and a dynamic filename (e.g., `{{ Guest Name }}_Clip_{{ index }}.mp4`).
    - Add a **Google Sheets** node. Operation: `Append`. Target a sheet named "Interview Clip Library". Map all metadata (Date, Tags, Score, Drive Link, etc.).
6.  **Final Notification:**
    - Add a **Gmail** node. Set the recipient to the email from the form. Compose an HTML body referencing the Guest Name and the original Interview URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Use descriptive queries (4-6 words) to avoid empty results. | Input Guideline |
| WayinVideo export links expire strictly after 24 hours. | Technical Constraint |
| Ensure Google Sheet tab is named exactly: "Interview Clip Library". | Sheet Configuration |