Generate product demo highlight reels using WayinVideo Find Moments API

https://n8nworkflows.xyz/workflows/generate-product-demo-highlight-reels-using-wayinvideo-find-moments-api-14712


# Generate product demo highlight reels using WayinVideo Find Moments API

# Workflow Documentation: Generate Product Demo Highlight Reels

## 1. Workflow Overview
This workflow automates the extraction of key moments from video product demonstrations. It allows users to submit a video URL and a natural language query (e.g., "pricing discussion"), uses the WayinVideo AI API to locate those specific segments, and delivers a formatted email containing the clips, timestamps, and download links.

The logic is divided into four functional blocks:
- **1.1 Input Reception:** Captures user requirements via an n8n Form.
- **1.2 Moment Submission:** Interfaces with the WayinVideo API to initiate the AI search.
- **1.3 Processing and Poll:** Manages the asynchronous nature of AI video processing via a wait-and-retry loop.
- **1.4 Review Email:** Aggregates the results and notifies the user via Gmail.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
**Overview:** This block serves as the entry point, gathering all necessary parameters to start the AI processing.
- **Nodes Involved:** `1. Form — Demo URL + Query + Email`
- **Node Details:**
    - **Type:** `n8n-nodes-base.formTrigger`
    - **Configuration:** A public form with four required fields: Demo Recording URL, Product/Brand Name, "What Moments To Find" (Query), and Recipient Email.
    - **Input/Output:** Trigger $\rightarrow$ Outputs JSON object containing form responses.
    - **Edge Cases:** Submission of invalid URLs or empty fields (mitigated by "Required" field settings).

### 2.2 Moment Submission
**Overview:** Initiates the search process by sending the video and query to the WayinVideo Find Moments API.
- **Nodes Involved:** `2. WayinVideo — Submit Find Moments`
- **Node Details:**
    - **Type:** `n8n-nodes-base.httpRequest`
    - **Configuration:** 
        - **Method:** `POST`
        - **URL:** `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments`
        - **Body:** JSON containing `video_url`, `query`, `project_name`, and settings (HD_720, captions enabled, 5 clip limit).
        - **Headers:** Requires `Authorization: Bearer YOUR_TOKEN_HERE` and `x-wayinvideo-api-version: v2`.
    - **Input/Output:** Form Data $\rightarrow$ Outputs a Task ID (used in the polling phase).
    - **Failure Types:** Authentication errors (Invalid Token) or API timeouts.

### 2.3 Processing and Poll
**Overview:** AI video processing is not instantaneous. This block implements a polling mechanism to wait until the API status is "SUCCEEDED".
- **Nodes Involved:** `3. Wait — 90 Seconds`, `4. WayinVideo — Get Moments Result`, `5. IF — Status SUCCEEDED?`, `6. Wait — 30 Seconds Retry`.
- **Node Details:**
    - **3. Wait (Initial):** Pauses for 90 seconds to give the AI initial processing time.
    - **4. Get Moments Result:** `GET` request to the result endpoint using the Task ID from Node 2.
    - **5. IF Status:** Checks if `{{ $json.data.status }}` equals `SUCCEEDED`.
    - **6. Wait (Retry):** If status is not "SUCCEEDED", pauses for 30 seconds before looping back to Node 4.
    - **Edge Cases:** **Infinite Loop Risk.** If the API returns an error or never reaches "SUCCEEDED", the workflow will loop indefinitely.

### 2.4 Review Email
**Overview:** Transforms the raw JSON result from the API into a professional, human-readable HTML email.
- **Nodes Involved:** `7. Gmail — Send Review Email`
- **Node Details:**
    - **Type:** `n8n-nodes-base.gmail`
    - **Configuration:** Uses an HTML template that maps through the `clips` array. It calculates seconds from milliseconds (`begin_ms/1000`) and displays titles, scores, and download links.
    - **Input/Output:** SUCCEEDED Status $\rightarrow$ Sends email to the address provided in the form.
    - **Edge Cases:** **Link Expiration.** The `export_link` provided by WayinVideo is temporary (24h).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form — Demo URL... | Form Trigger | Input Collection | - | 2. WayinVideo — Submit... | User submits the demo recording URL, product name, moment query, and review email address. |
| 2. WayinVideo — Submit... | HTTP Request | API Task Initiation | 1. Form | 3. Wait — 90 Seconds | Sends the video URL and search query to WayinVideo Find Moments API. Returns a task ID. |
| 3. Wait — 90 Seconds | Wait | Initial Processing Delay | 2. WayinVideo — Submit... | 4. WayinVideo — Get... | Waits 90 seconds then checks if WayinVideo has finished. |
| 4. WayinVideo — Get... | HTTP Request | Status Polling | 3. Wait / 6. Wait | 5. IF — Status SUCCEEDED? | Polls the API for the completed result. |
| 5. IF — Status SUCCEEDED? | IF | Completion Check | 4. WayinVideo — Get... | 7. Gmail / 6. Wait | Checks if processing is done — retries every 30 seconds if not. |
| 6. Wait — 30 Seconds... | Wait | Loop Delay | 5. IF — Status... | 4. WayinVideo — Get... | If clips are not ready, waits 30 more seconds and checks again. |
| 7. Gmail — Send Review... | Gmail | Notification | 5. IF — Status... | - | Formats all clip results into a clean HTML email. |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Input Trigger
1. Create a **Form Trigger** node.
2. Add four required text fields: `Demo Recording URL`, `Product / Brand Name`, `What Moments To Find`, and `Send Review Email To`.
3. Set a descriptive form title and description.

### Step 2: Initialize AI Request
1. Create an **HTTP Request** node.
2. Set Method to `POST` and URL to `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments`.
3. In **Headers**, add `Authorization` (Bearer token), `x-wayinvideo-api-version` (`v2`), and `Content-Type` (`application/json`).
4. In **Body**, use JSON and map the form fields to the corresponding keys: `video_url`, `query`, and `project_name`. Set `limit: 5` and `enable_export: true`.

### Step 3: The Polling Loop
1. Create a **Wait** node: Set to 90 seconds.
2. Create a second **HTTP Request** node:
    - Method: `GET`.
    - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips/find-moments/results/{{ $node["2. WayinVideo — Submit Find Moments"].json["data"]["id"] }}`.
    - Headers: Same as Step 2.
3. Create an **IF** node:
    - Condition: String `{{ $json.data.status }}` equals `SUCCEEDED`.
4. Create a second **Wait** node: Set to 30 seconds.
5. **Connections:**
    - Wait (90s) $\rightarrow$ Get Results $\rightarrow$ IF.
    - IF (False) $\rightarrow$ Wait (30s) $\rightarrow$ Get Results.
    - IF (True) $\rightarrow$ Next Step.

### Step 4: Final Delivery
1. Create a **Gmail** node (requires Google OAuth2 credentials).
2. Set the "To" field using the expression from the Form node: `{{ $node["1. Form..."].json["Send Review Email To"] }}`.
3. In the **Message** field, enable HTML and use a template to map the `data.clips` array from the "Get Moments Result" node.
4. Ensure the template calculates timestamps by dividing `begin_ms` and `end_ms` by 1000.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Infinite Loop Risk:** If the API never returns `SUCCEEDED`, the workflow loops forever. Recommendation: Add a counter node to stop after X attempts. | Logic Safety |
| **Link Expiration:** Export links for video clips expire after 24 hours. | Data Retention |
| **API Version:** This workflow is built for WayinVideo API v2. | Technical Specification |