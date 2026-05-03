Publish Instagram Reels from Notion with Claude captions and UploadToUrl

https://n8nworkflows.xyz/workflows/publish-instagram-reels-from-notion-with-claude-captions-and-uploadtourl-14658


# Publish Instagram Reels from Notion with Claude captions and UploadToUrl

# Workflow Documentation: Publish Instagram Reels from Notion with Claude Captions

### 1. Workflow Overview
This workflow automates the pipeline of publishing Instagram Reels by sourcing content from Notion, enhancing captions with AI, and implementing a human-in-the-loop approval process. It ensures that high-quality, branded content is posted only after a manager's review.

The logic is organized into five primary functional blocks:
- **1.1 Input & Validation:** Captures the request via a form and verifies Notion data.
- **1.2 Content Enhancement & Asset Hosting:** Generates AI captions and hosts media on a public CDN.
- **1.3 Instagram Stage 1 (Container):** Initializes the Reel upload on Meta's servers.
- **1.4 Approval Gate:** Sends a preview email and pauses execution until a manual decision is made.
- **1.5 Finalization & Notification:** Polls for encoding completion, publishes the Reel, and updates tracking systems.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
**Overview:** Handles the initial trigger and ensures the Notion page is in a valid state for publishing to prevent duplicates or errors.
- **Nodes Involved:** `n8n Form Submit Reel Request`, `HTTP Read Notion Page`, `Code Validate and Extract`.
- **Node Details:**
    - **n8n Form Submit Reel Request:** (Form Trigger) Collects `Notion Page ID` and `Caption Tone` (hype, minimal, storytelling).
    - **HTTP Read Notion Page:** (HTTP Request) Calls Notion API `GET /v1/pages/{id}`. Uses `NOTION_API_KEY`.
    - **Code Validate and Extract:** (Code) Extracts properties (Title, Video URL, etc.). **Edge Case:** Throws an error if the status is already "Published", "Failed", or "Rejected", or if the Video URL is missing.

#### 2.2 AI Processing & Asset Hosting
**Overview:** Generates the caption using Claude AI and converts Notion's internal links into public URLs required by Instagram.
- **Nodes Involved:** `Code Claude AI Caption`, `HTTP Download Video Binary`, `Upload to URL Video CDN`, `Code Store Video CDN URL`, `HTTP Download Cover Binary`, `Upload to URL Cover CDN`, `Code Merge CDN URLs`.
- **Node Details:**
    - **Code Claude AI Caption:** (Code) Uses Anthropic's `claude-haiku-4-5`. Sends title, description, and tone. **Edge Case:** Includes a fallback static template if the API call fails.
    - **HTTP Download Video Binary:** (HTTP Request) Fetches raw video from Notion into binary property `videoData`.
    - **Upload to URL Video CDN:** (UploadToUrl) Uploads binary to a public CDN to meet Instagram's HTTPS requirement.
    - **HTTP Download Cover Binary:** (HTTP Request) Fetches thumbnail binary.
    - **Upload to URL Cover CDN:** (UploadToUrl) Uploads thumbnail to CDN.
    - **Code Merge CDN URLs:** (Code) Consolidates the final public URLs and AI caption into a single JSON object.

#### 2.3 Instagram Container Creation
**Overview:** Creates a "media container" on Instagram. This is an asynchronous process where Meta prepares the video for publishing.
- **Nodes Involved:** `HTTP IG Create Reel Container`.
- **Node Details:**
    - **HTTP IG Create Reel Container:** (HTTP Request) POST to `/media` with `media_type=REELS`. Pass `video_url`, `cover_url`, and `caption`. Returns a `container_id`.

#### 2.4 Approval Gate
**Overview:** Stops the automated flow to allow a human to review the AI caption and video before it goes live.
- **Nodes Involved:** `Gmail Send Preview Email`, `Wait for Approval`, `IF Approved`.
- **Node Details:**
    - **Gmail Send Preview Email:** (Gmail) Sends an HTML email to `APPROVER_EMAIL` with "Approve" and "Reject" buttons using the `{{ $resumeUrl }}`.
    - **Wait for Approval:** (Wait) Pauses the workflow until the webhook URL is triggered.
    - **IF Approved:** (IF) Evaluates the query parameter `action`. If `approve`, it proceeds; otherwise, it routes to the rejection branch.

#### 2.5 Publishing & Post-Processing
**Overview:** Finalizes the upload, verifies the permalink, and notifies the team.
- **Nodes Involved:** `Wait Poll Buffer`, `HTTP Poll Container Status`, `Set Increment Retry`, `IF Encoding Finished`, `IG Publish Reel1`, `HTTP Fetch Reel Permalink`, `HTTP Notion Mark Published`, `Discord Notify Published`, `Code Retry or Fail`, `HTTP Notion Mark Failed`, `Discord Notify Encoding Failed`, `HTTP Notion Mark Rejected`, `Discord Notify Rejected`.
- **Node Details:**
    - **HTTP Poll Container Status:** (HTTP Request) Checks if `status_code` is `FINISHED`.
    - **Set Increment Retry / Code Retry or Fail:** Implements a loop that retries polling up to 15 times with 20-second intervals.
    - **IG Publish Reel1:** (HTTP Request) Calls `/media_publish` using the `container_id`.
    - **HTTP Fetch Reel Permalink:** (HTTP Request) Retrieves the live URL of the post.
    - **HTTP Notion Mark Published:** (HTTP Request) PATCHes the Notion page with the Permalink and "Published" status.
    - **Discord Notify Published:** (HTTP Request) Sends a formatted embed to Discord via webhook.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| n8n Form Submit Reel Request | Form Trigger | Entry point / Input | - | HTTP Read Notion Page | Nodes 1–3 — Form, Read Notion, Validate |
| HTTP Read Notion Page | HTTP Request | Data Retrieval | n8n Form... | Code Validate... | Nodes 1–3 — Form, Read Notion, Validate |
| Code Validate and Extract | Code | Data Validation | HTTP Read... | Code Claude AI... | Nodes 1–3 — Form, Read Notion, Validate |
| Code Claude AI Caption | Code | Caption Generation | Code Validate... | HTTP Download... | Nodes 4–6 — Claude Caption... |
| HTTP Download Video Binary | HTTP Request | File Retrieval | Code Claude AI... | Upload to URL... | Nodes 4–6 — Claude Caption... |
| Upload to URL Video CDN | UploadToUrl | Asset Hosting | HTTP Download... | Code Store Video... | Nodes 4–6 — Claude Caption... |
| Code Store Video CDN URL | Code | URL Management | Upload to URL... | HTTP Download... | Nodes 4–6 — Claude Caption... |
| HTTP Download Cover Binary | HTTP Request | File Retrieval | Code Store... | Upload to URL... | Nodes 7–9 — Download Cover... |
| Upload to URL Cover CDN | UploadToUrl | Asset Hosting | HTTP Download... | Code Merge CDN... | Nodes 7–9 — Download Cover... |
| Code Merge CDN URLs | Code | Data Consolidation | Upload to URL... | HTTP IG Create... | Nodes 7–9 — Download Cover... |
| HTTP IG Create Reel Container | HTTP Request | IG Initialization | Code Merge CDN... | Gmail Send... | Nodes 7–9 — Download Cover... |
| Gmail Send Preview Email | Gmail | Human Notification | HTTP IG Create... | Wait for Approval | Nodes 10–12 — Email Preview... |
| Wait for Approval | Wait | Human-in-the-loop | Gmail Send... | IF Approved | Nodes 10–12 — Email Preview... |
| IF Approved | IF | Decision Branch | Wait for Approval | Wait Poll Buffer / HTTP Notion Mark Rejected | Nodes 10–12 — Email Preview... |
| Wait Poll Buffer | Wait | Polling Delay | IF Approved / Code Retry... | HTTP Poll... | Nodes 10–12 — Email Preview... |
| HTTP Poll Container Status | HTTP Request | Status Check | Wait Poll Buffer | Set Increment Retry | Nodes 10–12 — Email Preview... |
| Set Increment Retry | Set | Loop Counter | HTTP Poll... | IF Encoding Finished | Nodes 10–12 — Email Preview... |
| IF Encoding Finished | IF | State Check | Set Increment Retry | IG Publish Reel / Code Retry... | Nodes 13–17 — Gate, Publish... |
| IG Publish Reel1 | HTTP Request | Final Publication | IF Encoding... | HTTP Fetch Reel... | Nodes 13–17 — Gate, Publish... |
| HTTP Fetch Reel Permalink | HTTP Request | Log Retrieval | IG Publish Reel1 | HTTP Notion Mark... | Nodes 13–17 — Gate, Publish... |
| HTTP Notion Mark Published | HTTP Request | Status Update | HTTP Fetch Reel... | Discord Notify... | Nodes 13–17 — Gate, Publish... |
| Discord Notify Published | HTTP Request | Team Alert | HTTP Notion... | - | Nodes 13–17 — Gate, Publish... |
| Code Retry or Fail | Code | Loop Control | IF Encoding... | Wait Poll Buffer / HTTP Notion Mark Failed | Nodes 13–17 — Gate, Publish... |
| HTTP Notion Mark Failed | HTTP Request | Status Update | Code Retry... | Discord Notify Encoding... | Error Branch — Encoding Failure |
| Discord Notify Encoding Failed | HTTP Request | Team Alert | HTTP Notion... | - | Error Branch — Encoding Failure |
| HTTP Notion Mark Rejected | HTTP Request | Status Update | IF Approved | Discord Notify Rejected | Error Branch — Encoding Failure |
| Discord Notify Rejected | HTTP Request | Team Alert | HTTP Notion... | - | Error Branch — Encoding Failure |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Environment Variables
Configure the following variables in n8n:
- `NOTION_API_KEY`, `IG_USER_ID`, `IG_ACCESS_TOKEN`, `ANTHROPIC_API_KEY`, `APPROVER_EMAIL`, `DISCORD_WEBHOOK_URL`.

#### Step 2: Input & Validation
1. Create an **n8n Form Trigger** with two fields: `Notion Page ID` (text) and `Caption Tone` (dropdown: storytelling, hype, minimal).
2. Connect to an **HTTP Request** node: `GET https://api.notion.com/v1/pages/{{ $json['Notion Page ID'] }}`. Add headers for `Authorization` (Bearer token) and `Notion-Version`.
3. Connect to a **Code Node**: Implement a JS function to extract properties and throw an Error if `Status === 'Published'` or if `Video URL` is empty.

#### Step 3: AI & Asset Management
1. Create a **Code Node** to call the Anthropic API via `fetch`. Use the `claude-haiku-4-5` model. Pass the title, description, and tone. Implement a fallback caption.
2. Create an **HTTP Request** node to download the video URL as a binary file (`videoData`).
3. Use the **UploadToUrl** node to push the binary to a public CDN.
4. Use a **Code Node** to capture the resulting `public_url`.
5. Repeat the Download $\rightarrow$ Upload process for the `Cover URL`.
6. Use a final **Code Node** to merge the Video CDN URL, Cover CDN URL, and AI Caption into one object.

#### Step 4: Instagram Staging
1. Create an **HTTP Request** node: `POST https://graph.facebook.com/v19.0/{{ ig_user_id }}/media`.
2. Set query parameters: `media_type=REELS`, `video_url`, `cover_url`, `caption`, `share_to_feed=true`, and `access_token`.

#### Step 5: Approval Logic
1. Create a **Gmail Node**: Send an HTML email to the approver. Include two links: `{{ $resumeUrl }}?action=approve` and `{{ $resumeUrl }}?action=reject`.
2. Create a **Wait Node**: Set resume type to `webhook`.
3. Create an **IF Node**: Check if `{{ $json.query.action }} === 'approve'`.

#### Step 6: Polling & Publishing
1. **True Branch:** 
   - Add a **Wait Node** (20 seconds).
   - Add an **HTTP Request** to poll the container status: `GET /{{ container_id }}`.
   - Add a **Set Node** to increment a `retry_count`.
   - Add an **IF Node** to check if `status_code === 'FINISHED'`.
   - If False: Route to a **Code Node** that checks if `retry_count < 15`. If yes, loop back to the Wait node; if no, route to "Mark Failed".
   - If True: Call **HTTP Request** `POST /media_publish`.
2. **Post-Publish:** 
   - Use an **HTTP Request** to fetch the `permalink`.
   - Use an **HTTP Request** to `PATCH` the Notion page (Set status to Published, add Permalink).
   - Use an **HTTP Request** (Discord Webhook) to notify the team.

#### Step 7: Rejection & Failure Handling
1. **False Branch (IF Approved):** `PATCH` Notion page status to "Rejected" $\rightarrow$ Discord alert.
2. **Failure Branch (Polling):** `PATCH` Notion page status to "Failed" $\rightarrow$ Discord alert.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Instagram Reels require direct public HTTPS links; binary uploads are not supported. | Technical Constraint |
| Use Claude-Haiku for cost-effective, fast caption generation. | AI Strategy |
| Ensure the Notion integration has "Read" and "Update" permissions for the specific database. | Permission Requirement |