Extract customer testimonial clips using WayinVideo AI and Google Drive

https://n8nworkflows.xyz/workflows/extract-customer-testimonial-clips-using-wayinvideo-ai-and-google-drive-14641


# Extract customer testimonial clips using WayinVideo AI and Google Drive

# Workflow Analysis: Extract customer testimonial clips using WayinVideo AI and Google Drive

### 1. Workflow Overview
The purpose of this workflow is to automate the extraction of short, high-impact vertical clips from long-form customer testimonial videos. It leverages WayinVideo's AI to identify the best moments, processes them into social-media-ready formats, and automatically archives the resulting files into a specific Google Drive folder.

The logic is organized into four functional blocks:
- **1.1 Input Reception:** Captures video URLs and client metadata via an n8n hosted form.
- **1.2 AI Processing Submission:** Sends the video to the WayinVideo API and implements a mandatory processing delay.
- **1.3 Result Retrieval:** Fetches the processed clip metadata using the job ID from the submission phase.
- **1.4 File Extraction & Archiving:** Iterates through the detected clips, downloads the binary video files, and uploads them to Google Drive with descriptive filenames.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Acts as the entry point for the workflow, providing a user interface to collect necessary data for the AI processing.
- **Nodes Involved:** `1. Form — Testimonial Video and Details`
- **Node Details:**
    - **Type:** Form Trigger
    - **Configuration:** A public form collecting four fields: Testimonial Video URL, Client/Customer Name, Industry/Niche, and Intended Usage.
    - **Input/Output:** Trigger $\rightarrow$ WayinVideo Submit node.
    - **Edge Cases:** If the user provides an invalid or unsupported URL (e.g., a private video), the subsequent API call will fail.

#### 2.2 Submit and Wait
- **Overview:** Initiates the AI clipping process and ensures the AI has enough time to process the video before the workflow attempts to retrieve results.
- **Nodes Involved:** `2. WayinVideo — Submit Clipping Task`, `3. Wait — 90 Seconds`
- **Node Details:**
    - **Node 2 (HTTP Request):** 
        - **Technical Role:** POST request to `/api/v2/clips`.
        - **Configuration:** Sets target duration (30-60s), limit (5 clips), ratio (9:16 vertical), and enables captions and AI reframing.
        - **Key Expressions:** Uses `{{ $json['Testimonial Video URL'] }}` and `{{ $json['Client / Customer Name'] }}`.
        - **Potential Failures:** Authentication errors if the Bearer token is incorrect or missing.
    - **Node 3 (Wait):** 
        - **Technical Role:** Execution delay.
        - **Configuration:** Fixed pause of 90 seconds.
        - **Edge Cases:** For videos longer than 30 minutes, 90 seconds may be insufficient, leading to an empty result in the next step.

#### 2.3 Fetch Results
- **Overview:** Retrieves the list of generated clips once the AI processing is complete.
- **Nodes Involved:** `4. WayinVideo — Get Clip Results`
- **Node Details:**
    - **Type:** HTTP Request (GET)
    - **Configuration:** Requests results from the endpoint using the ID generated in Node 2.
    - **Key Expressions:** `{{ $('2. WayinVideo — Submit Clipping Task').item.json.data.id }}`.
    - **Input/Output:** Wait node $\rightarrow$ Code node.
    - **Potential Failures:** If the job is still "processing" after the wait period, this node may return an incomplete list or an error.

#### 2.4 Extract and Upload
- **Overview:** Transforms the API response into individual items, downloads the actual video files, and saves them to the cloud.
- **Nodes Involved:** `5. Code — Extract Clips Array`, `6. HTTP — Download Clip File`, `7. Google Drive — Upload Clip`
- **Node Details:**
    - **Node 5 (Code):** 
        - **Technical Role:** Data transformation (Array to Items).
        - **Configuration:** Maps the `data.clips` array into a list of objects containing `title`, `export_link`, and `score`.
    - **Node 6 (HTTP Request):** 
        - **Technical Role:** Binary file download.
        - **Configuration:** Set to "File" response format using the `export_link` from the previous node.
    - **Node 7 (Google Drive):** 
        - **Technical Role:** File upload.
        - **Configuration:** Uploads the binary file to a specific `folderId`. The filename is dynamically set to the AI-generated clip title.
        - **Input/Output:** Download node $\rightarrow$ End of workflow.
        - **Potential Failures:** Google Drive API quota limits or incorrect Folder ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form — Testimonial Video and Details | Form Trigger | User input collection | - | 2. WayinVideo — Submit... | Section 1 — Input |
| 2. WayinVideo — Submit Clipping Task | HTTP Request | AI processing initiation | 1. Form... | 3. Wait... | Section 2 — Submit and Wait |
| 3. Wait — 90 Seconds | Wait | Processing delay | 2. WayinVideo — Submit... | 4. WayinVideo — Get... | Section 2 — Submit and Wait |
| 4. WayinVideo — Get Clip Results | HTTP Request | Retrieve clip metadata | 3. Wait... | 5. Code... | Section 3 — Fetch Results |
| 5. Code — Extract Clips Array | Code | Flatten results for iteration | 4. WayinVideo — Get... | 6. HTTP — Download... | Section 4 — Extract, Download and Upload |
| 6. HTTP — Download Clip File | HTTP Request | Download binary video | 5. Code... | 7. Google Drive... | Section 4 — Extract, Download and Upload |
| 7. Google Drive — Upload Clip | Google Drive | Archive file to cloud | 6. HTTP — Download... | - | Section 4 — Extract, Download and Upload |

---

### 4. Reproducing the Workflow from Scratch

1.  **Input Trigger:** Create a **Form Trigger** node. Add four required text fields: "Testimonial Video URL", "Client / Customer Name", "Industry / Niche", and "Where Will Clips Be Used".
2.  **AI Submission:** Create an **HTTP Request** node.
    - **Method:** POST
    - **URL:** `https://wayinvideo-api.wayin.ai/api/v2/clips`
    - **Headers:** `Authorization: Bearer [YOUR_TOKEN]`, `x-wayinvideo-api-version: v2`, `Content-Type: application/json`.
    - **Body:** JSON format. Map the video URL and project name from the Form trigger. Set `limit: 5`, `ratio: "RATIO_9_16"`, and `enable_caption: true`.
3.  **The Pause:** Add a **Wait** node. Set the amount to `90` seconds.
4.  **Result Retrieval:** Create an **HTTP Request** node.
    - **Method:** GET
    - **URL:** Use an expression to append the ID from Node 2: `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ $json.data.id }}`.
    - **Headers:** Include the same Authorization and Version headers used in step 2.
5.  **Data Parsing:** Add a **Code** node. Use JavaScript to map `$json.data.clips` into a new array of objects, extracting `title` and `export_link`.
6.  **File Download:** Add an **HTTP Request** node.
    - **URL:** Use expression `{{ $json.export_link }}`.
    - **Response Format:** Change "Response Data" to "File".
7.  **Cloud Storage:** Add a **Google Drive** node.
    - **Operation:** Upload a file.
    - **File Name:** Use expression `{{ $('5. Code — Extract Clips Array').item.json.title }}`.
    - **Folder ID:** Provide your specific Google Drive folder ID.
    - **Credentials:** Connect a Google Drive OAuth2 credential.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Warning:** Fixed Wait Risk — 90s may not be enough for videos > 30 mins. Consider a loop with an IF node for long content. | Technical limitation/Edge case |
| **Customization:** Change `RATIO_9_16` to `RATIO_16_9` for landscape videos. | Configuration Tip |
| **Customization:** Adjust the `limit` value in node 2 to extract more or fewer clips. | Configuration Tip |
| **Expansion:** Add Gmail or Slack nodes after Node 7 for team notifications. | Scalability Suggestion |