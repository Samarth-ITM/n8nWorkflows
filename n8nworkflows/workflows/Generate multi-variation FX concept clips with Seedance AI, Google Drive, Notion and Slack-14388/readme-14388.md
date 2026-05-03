Generate multi-variation FX concept clips with Seedance AI, Google Drive, Notion and Slack

https://n8nworkflows.xyz/workflows/generate-multi-variation-fx-concept-clips-with-seedance-ai--google-drive--notion-and-slack-14388


# Generate multi-variation FX concept clips with Seedance AI, Google Drive, Notion and Slack

# 1. Workflow Overview

This workflow is an AI-driven FX concept generation pipeline built in n8n. It receives an FX brief through a webhook, expands that single brief into four creative variations, submits each variation to Seedance AI for video generation, waits asynchronously for rendering to complete, stores the results in Google Drive and Notion, then notifies the FX team in Slack with consolidated review information. A separate error branch catches workflow failures and posts alerts to Slack.

## 1.1 Input Reception and Validation
The workflow starts with a webhook that accepts the FX brief payload. A Code node then validates and normalizes the incoming data so later nodes can rely on consistent field names and formats.

## 1.2 Prompt Expansion and Generation Submission
The validated brief is expanded into four concept variants. Each variation is converted into a request body for Seedance AI and submitted as a separate generation job.

## 1.3 Asynchronous Render Polling
After job submission, the workflow stores job IDs alongside variation metadata, checks render status via polling, and loops with a 20-second wait until each render is complete.

## 1.4 Asset Packaging, Storage, and Logging
When a render completes, the workflow builds structured metadata, creates a Notion record, downloads the generated video, and uploads it to Google Drive.

## 1.5 Review Aggregation and Team Notification
Once result data is available from storage/logging branches, the workflow aggregates the variation outputs and sends a Slack notification to the FX team for review.

## 1.6 Global Error Handling
A dedicated error trigger branch catches workflow-level failures and posts an alert to Slack so issues are visible immediately.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Validation

### Overview
This block accepts the incoming FX brief and prepares it for downstream processing. It is responsible for transforming external webhook input into a normalized internal schema.

### Nodes Involved
- Webhook: FX Brief Input
- Validate & Extract FX Brief

### Node Details

#### Webhook: FX Brief Input
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for external POST requests.
- **Configuration choices:** The JSON shows a webhook node with no explicit parameters configured in the export. In practice, this means the node must be configured in the editor for the desired HTTP method, path, response mode, and payload expectations.
- **Key expressions or variables used:** None visible in the export.
- **Input and output connections:** No input; outputs to **Validate & Extract FX Brief**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Webhook method/path mismatch
  - Missing request body
  - Invalid JSON payload
  - Timeout if response mode is configured incorrectly for long-running executions
- **Sub-workflow reference:** None.

#### Validate & Extract FX Brief
- **Type and technical role:** `n8n-nodes-base.code`; parses, validates, and standardizes incoming brief fields.
- **Configuration choices:** The code content is not present in the export, but based on the workflow description this node likely:
  - checks required fields such as brief text and shot code
  - extracts optional fields like FX type, intensity, scale, FPS, frame range, camera movement, speed, and reference image
  - assigns defaults where values are omitted
  - normalizes values into a consistent structure
- **Key expressions or variables used:** Likely based on `$json` from webhook input.
- **Input and output connections:** Input from **Webhook: FX Brief Input**; output to **Fan-Out: 4 FX Variations**.
- **Version-specific requirements:** Type version `2`, which uses JavaScript in the Code node.
- **Edge cases or potential failure types:**
  - Required field missing
  - Invalid numeric parsing for FPS or frame ranges
  - Invalid enum-like values for FX type or camera mode
  - Expression/code runtime errors
- **Sub-workflow reference:** None.

---

## Block 2 — Prompt Expansion and Generation Submission

### Overview
This block takes a single validated FX brief and turns it into four separate generation-ready concept variations. It then prepares and submits each variation to Seedance AI.

### Nodes Involved
- Fan-Out: 4 FX Variations
- Build FX Request Body
- Seedance: Generate FX Clip

### Node Details

#### Fan-Out: 4 FX Variations
- **Type and technical role:** `n8n-nodes-base.code`; clones one normalized brief into four distinct creative variants.
- **Configuration choices:** Based on the description, this node likely creates:
  - primary balanced variation
  - high-intensity variation
  - subtle variation
  - alternate camera variation
  Each output item probably contains variation labels, prompt text, and inherited technical parameters.
- **Key expressions or variables used:** Likely reads normalized fields from `$json`.
- **Input and output connections:** Input from **Validate & Extract FX Brief**; output to **Build FX Request Body**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Fan-out logic returning malformed items
  - Missing prompt text in one or more variations
  - Inconsistent handling of optional reference image data
- **Sub-workflow reference:** None.

#### Build FX Request Body
- **Type and technical role:** `n8n-nodes-base.code`; builds the exact API payload required by Seedance.
- **Configuration choices:** This node likely chooses between:
  - text-to-video mode when no reference image is present
  - image-to-video mode when a reference image is provided
  It probably maps prompt, model parameters, duration, camera guidance, and technical metadata into the expected request body.
- **Key expressions or variables used:** Likely references fields such as prompt, variation name, reference image URL or binary source, shot code, intensity, FPS, frame range.
- **Input and output connections:** Input from **Fan-Out: 4 FX Variations**; output to **Seedance: Generate FX Clip**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Invalid request schema for Seedance
  - Reference image field present but unusable
  - Missing prompt after transformation
- **Sub-workflow reference:** None.

#### Seedance: Generate FX Clip
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends generation jobs to the Seedance API.
- **Configuration choices:** Parameters are empty in the export, so the exact endpoint/method/auth are not preserved. In a working setup, this node would normally:
  - use `POST`
  - call the Seedance generation endpoint
  - send JSON built by the previous node
  - authenticate with API key or bearer token
  - parse the JSON response and extract a job ID
- **Key expressions or variables used:** Request body likely built from previous item JSON.
- **Input and output connections:** Input from **Build FX Request Body**; output to **Merge FX Job ID + Metadata**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Authentication failure
  - Invalid payload
  - API rate limiting
  - Seedance service timeout or 5xx error
  - Response schema mismatch if job ID field name changes
- **Sub-workflow reference:** None.

---

## Block 3 — Asynchronous Render Polling

### Overview
This block tracks each submitted Seedance render asynchronously. It persists metadata alongside the job ID, checks render status, and loops every 20 seconds until the job is done.

### Nodes Involved
- Merge FX Job ID + Metadata
- Poll: Check FX Job Status
- FX Render Complete?
- Wait 20s

### Node Details

#### Merge FX Job ID + Metadata
- **Type and technical role:** `n8n-nodes-base.code`; combines the Seedance response with upstream variation context.
- **Configuration choices:** Likely merges:
  - Seedance job ID
  - variation label
  - prompt text
  - shot code
  - FX parameters
  This is essential because polling responses often contain less context than the original request.
- **Key expressions or variables used:** Probably accesses both current response fields and incoming metadata fields in `$json`.
- **Input and output connections:** Input from **Seedance: Generate FX Clip**; output to **Poll: Check FX Job Status**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Job ID missing in API response
  - Merge logic overwriting important fields
  - Null/undefined access in code
- **Sub-workflow reference:** None.

#### Poll: Check FX Job Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; queries Seedance for render job status.
- **Configuration choices:** In practice this would usually:
  - use `GET` or status endpoint-specific method
  - inject the job ID into the URL or query/body
  - authenticate with the same Seedance credentials
  - return status fields such as pending/running/completed/failed and output asset URL when ready
- **Key expressions or variables used:** Likely references the merged job ID from `$json`.
- **Input and output connections:** Inputs from **Merge FX Job ID + Metadata** and loopback from **Wait 20s**; output to **FX Render Complete?**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Job not found
  - Temporary API failures
  - Polling too aggressively and hitting rate limits
  - Status schema changes
  - Completion response missing video URL
- **Sub-workflow reference:** None.

#### FX Render Complete?
- **Type and technical role:** `n8n-nodes-base.if`; branching node that decides whether to continue waiting or proceed to asset handling.
- **Configuration choices:** Parameters are not included, but it likely checks a status field for a success/completed value.
- **Key expressions or variables used:** Likely a comparison on something like `$json.status`.
- **Input and output connections:** Input from **Poll: Check FX Job Status**; true branch to **Build FX Asset Metadata**; false branch to **Wait 20s**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Incorrect status comparison causing infinite loop
  - Failed jobs treated as incomplete rather than errored
  - Null status field
- **Sub-workflow reference:** None.

#### Wait 20s
- **Type and technical role:** `n8n-nodes-base.wait`; pauses execution before the next status check.
- **Configuration choices:** The node name indicates a 20-second wait, but the actual wait settings are not present in the export. It also includes a webhook ID because Wait nodes can resume executions through n8n’s internal mechanism.
- **Key expressions or variables used:** None visible.
- **Input and output connections:** Input from the false branch of **FX Render Complete?**; output back to **Poll: Check FX Job Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Wait duration misconfigured
  - Resume/webhook issues if n8n instance URL is not configured properly
  - Long polling across many items increasing execution load
- **Sub-workflow reference:** None.

---

## Block 4 — Asset Packaging, Storage, and Logging

### Overview
After each render finishes, this block prepares a structured asset record, logs it to Notion, downloads the video, and uploads the file to Google Drive.

### Nodes Involved
- Build FX Asset Metadata
- Notion: Save FX Asset Record
- Download FX Video
- Google Drive: Upload FX Clip

### Node Details

#### Build FX Asset Metadata
- **Type and technical role:** `n8n-nodes-base.code`; builds the final asset metadata object for storage and review.
- **Configuration choices:** Likely assembles:
  - variation label
  - output video URL
  - shot code
  - FX type
  - intensity
  - scale
  - camera movement
  - FPS and frame range
  - tags
  - file naming convention
- **Key expressions or variables used:** Likely combines render output fields with original metadata.
- **Input and output connections:** Input from the true branch of **FX Render Complete?**; outputs in parallel to **Notion: Save FX Asset Record** and **Download FX Video**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Missing final asset URL despite completed status
  - Metadata schema inconsistency
  - Filename/path construction errors
- **Sub-workflow reference:** None.

#### Notion: Save FX Asset Record
- **Type and technical role:** `n8n-nodes-base.notion`; writes asset metadata to a Notion database.
- **Configuration choices:** Parameters are not visible, but this node is clearly intended to create a database page/record representing the FX asset. It likely maps metadata fields into database properties.
- **Key expressions or variables used:** Likely property mappings from `$json`, such as title, shot code, links, tags, and technical settings.
- **Input and output connections:** Input from **Build FX Asset Metadata**; output to **Aggregate All FX Variations**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Notion credential issues
  - Database ID misconfiguration
  - Property type mismatch
  - URL/title/rich-text formatting errors
- **Sub-workflow reference:** None.

#### Download FX Video
- **Type and technical role:** `n8n-nodes-base.httpRequest`; retrieves the rendered video file from the returned URL.
- **Configuration choices:** In practice this node should:
  - call the final video URL
  - return binary data
  - assign a binary property name for upload
- **Key expressions or variables used:** Likely references the completed asset URL from `$json`.
- **Input and output connections:** Input from **Build FX Asset Metadata**; output to **Google Drive: Upload FX Clip**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Expired or protected asset URL
  - Large file download timeout
  - Binary output misconfiguration
  - Content-type mismatch
- **Sub-workflow reference:** None.

#### Google Drive: Upload FX Clip
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads the generated video binary into Google Drive.
- **Configuration choices:** Parameters are not shown, but this typically includes:
  - operation: upload file
  - binary property input
  - target folder
  - file name from metadata naming convention
- **Key expressions or variables used:** Likely file name and folder expressions using shot code and variation label.
- **Input and output connections:** Input from **Download FX Video**; output to **Aggregate All FX Variations**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - OAuth credential issues
  - Insufficient Drive permissions
  - Missing binary property
  - Duplicate file naming collisions
  - Folder ID/path misconfiguration
- **Sub-workflow reference:** None.

---

## Block 5 — Review Aggregation and Team Notification

### Overview
This block combines outputs from storage and logging operations into a review-friendly structure and sends the team a Slack message.

### Nodes Involved
- Aggregate All FX Variations
- Slack: Notify FX Team

### Node Details

#### Aggregate All FX Variations
- **Type and technical role:** `n8n-nodes-base.code`; consolidates results from Notion and Google Drive branches into a single final payload.
- **Configuration choices:** Because both **Notion: Save FX Asset Record** and **Google Drive: Upload FX Clip** feed into this node, it likely reconciles:
  - Notion page information
  - Drive file links/IDs
  - variation metadata
  - render URLs or review URLs
  A robust implementation would also deduplicate or pair branch outputs correctly by variation ID.
- **Key expressions or variables used:** Likely uses incoming JSON arrays/items and matches them on a shared key such as shot code + variation name.
- **Input and output connections:** Inputs from **Notion: Save FX Asset Record** and **Google Drive: Upload FX Clip**; output to **Slack: Notify FX Team**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Branch synchronization mismatch
  - Duplicate or incomplete aggregated records
  - One branch succeeding while the other fails
  - Incorrect pairing of Notion and Drive results
- **Sub-workflow reference:** None.

#### Slack: Notify FX Team
- **Type and technical role:** `n8n-nodes-base.slack`; sends a review notification to the FX team.
- **Configuration choices:** Parameters are not exported, but expected setup includes:
  - operation to post a message
  - target channel
  - formatted message containing variation summaries, links, and technical metadata
- **Key expressions or variables used:** Likely references aggregated variation arrays, Drive links, Notion URLs, and technical fields.
- **Input and output connections:** Input from **Aggregate All FX Variations**; no downstream node.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Slack auth or permission issues
  - Channel ID misconfiguration
  - Message formatting errors
  - Exceeding Slack block/message limits if payload is too large
- **Sub-workflow reference:** None.

---

## Block 6 — Global Error Handling

### Overview
This independent branch listens for workflow failures and sends an error alert to Slack. It is important because the main workflow contains several external dependencies and asynchronous steps.

### Nodes Involved
- On Workflow Error
- Slack: Error Alert

### Node Details

#### On Workflow Error
- **Type and technical role:** `n8n-nodes-base.errorTrigger`; triggers when the workflow fails.
- **Configuration choices:** Standard error trigger entry point; no visible parameters in export.
- **Key expressions or variables used:** Typically exposes error metadata through `$json`.
- **Input and output connections:** No upstream input; output to **Slack: Error Alert**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Only catches errors according to n8n’s error trigger behavior
  - If the workflow is disabled or error workflow linkage is not appropriate in deployment, alerts may not fire as expected
- **Sub-workflow reference:** None.

#### Slack: Error Alert
- **Type and technical role:** `n8n-nodes-base.slack`; posts failure details to Slack.
- **Configuration choices:** Likely posts workflow name, failed node, execution ID, and error message to an ops/review channel.
- **Key expressions or variables used:** Likely references error trigger payload fields.
- **Input and output connections:** Input from **On Workflow Error**; no downstream node.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Slack auth or channel issues
  - Missing error detail fields in message template
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook: FX Brief Input | Webhook | Receives incoming FX brief payload |  | Validate & Extract FX Brief |  |
| Validate & Extract FX Brief | Code | Validates request fields and normalizes the FX brief | Webhook: FX Brief Input | Fan-Out: 4 FX Variations |  |
| Fan-Out: 4 FX Variations | Code | Expands one brief into four prompt variations | Validate & Extract FX Brief | Build FX Request Body |  |
| Build FX Request Body | Code | Builds the Seedance API payload for each variation | Fan-Out: 4 FX Variations | Seedance: Generate FX Clip |  |
| Seedance: Generate FX Clip | HTTP Request | Submits AI video generation job to Seedance | Build FX Request Body | Merge FX Job ID + Metadata |  |
| Merge FX Job ID + Metadata | Code | Combines Seedance job ID with prompt/variation metadata | Seedance: Generate FX Clip | Poll: Check FX Job Status |  |
| Poll: Check FX Job Status | HTTP Request | Polls Seedance for render status | Merge FX Job ID + Metadata, Wait 20s | FX Render Complete? |  |
| FX Render Complete? | If | Branches between completion and continued polling | Poll: Check FX Job Status | Build FX Asset Metadata, Wait 20s |  |
| Wait 20s | Wait | Delays the next status check | FX Render Complete? | Poll: Check FX Job Status |  |
| Build FX Asset Metadata | Code | Assembles final metadata for storage, logging, and review | FX Render Complete? | Notion: Save FX Asset Record, Download FX Video |  |
| Notion: Save FX Asset Record | Notion | Saves asset metadata into a Notion database | Build FX Asset Metadata | Aggregate All FX Variations |  |
| Download FX Video | HTTP Request | Downloads completed rendered video as binary | Build FX Asset Metadata | Google Drive: Upload FX Clip |  |
| Google Drive: Upload FX Clip | Google Drive | Uploads rendered FX clip to Google Drive | Download FX Video | Aggregate All FX Variations |  |
| Aggregate All FX Variations | Code | Merges logging and storage outputs into a final consolidated payload | Notion: Save FX Asset Record, Google Drive: Upload FX Clip | Slack: Notify FX Team |  |
| Slack: Notify FX Team | Slack | Sends review message with links and metadata | Aggregate All FX Variations |  |  |
| On Workflow Error | Error Trigger | Captures workflow failures |  | Slack: Error Alert |  |
| Slack: Error Alert | Slack | Sends Slack notification when the workflow fails | On Workflow Error |  |  |
| Sticky: Workflow Overview | Sticky Note | Canvas annotation |  |  |  |
| Sticky: Trigger & Validation | Sticky Note | Canvas annotation |  |  |  |
| Sticky: Prompt Fan-Out & Generation | Sticky Note | Canvas annotation |  |  |  |
| Sticky: Polling & Render Wait | Sticky Note | Canvas annotation |  |  |  |
| Sticky: Asset Storage & Logging | Sticky Note | Canvas annotation |  |  |  |
| Sticky: Review Notification | Sticky Note | Canvas annotation |  |  |  |
| Sticky: Credentials & Security | Sticky Note | Canvas annotation |  |  |  |
| Section: Error Handler | Sticky Note | Canvas annotation |  |  |  |

Note: all sticky note contents are empty in the provided workflow JSON, so no node-level sticky note text can be assigned.

---

# 4. Reproducing the Workflow from Scratch

Below is a manual rebuild plan based on the provided JSON and the descriptive behavior. Because many node parameter bodies are empty in the export, some implementation details must be reconstructed from the stated intent.

## 1. Create the webhook entry node
1. Add a **Webhook** node.
2. Name it **Webhook: FX Brief Input**.
3. Configure it to accept a **POST** request.
4. Set a path such as `fx-concept-webhook-001` or another environment-appropriate route.
5. Choose a response mode:
   - If you want an immediate acknowledgment, use an early response and process asynchronously.
   - If you want the caller to wait, be aware the workflow contains polling and may take a long time.
6. Expect a JSON payload with fields such as:
   - `brief`
   - `shotCode`
   - optional: `fxType`, `scale`, `intensity`, `cameraMovement`, `frameStart`, `frameEnd`, `fps`, `speed`, `referenceImageUrl`

## 2. Add validation and normalization logic
7. Add a **Code** node after the webhook.
8. Name it **Validate & Extract FX Brief**.
9. In JavaScript, validate required fields:
   - `brief`
   - `shotCode`
10. Normalize optional values:
   - default `fxType` if omitted
   - default `fps`
   - default frame range
   - default camera movement
   - normalize intensity/scale values to controlled labels or ranges
11. Return a clean JSON structure, for example with fields like:
   - `shotCode`
   - `brief`
   - `fxType`
   - `scale`
   - `intensity`
   - `cameraMovement`
   - `frameStart`
   - `frameEnd`
   - `fps`
   - `speed`
   - `referenceImageUrl`
12. Connect **Webhook: FX Brief Input** → **Validate & Extract FX Brief**.

## 3. Create the fan-out variation generator
13. Add a **Code** node.
14. Name it **Fan-Out: 4 FX Variations**.
15. Write code that emits four items from one input item:
   - `primary_balanced`
   - `high_intensity`
   - `subtle_controlled`
   - `alternate_camera`
16. For each item, include:
   - `variationName`
   - adjusted prompt text
   - inherited technical metadata
   - optional camera modifications for the alternate camera variation
17. Connect **Validate & Extract FX Brief** → **Fan-Out: 4 FX Variations**.

## 4. Build the Seedance request payload
18. Add a **Code** node.
19. Name it **Build FX Request Body**.
20. Write code that converts each variation item into the Seedance API schema.
21. Include logic:
   - if `referenceImageUrl` exists, use image-to-video mode
   - otherwise, use text-to-video mode
22. Build request fields such as:
   - prompt
   - model or mode
   - duration or clip settings
   - camera guidance
   - style or motion instructions
   - any reference image parameter
23. Preserve metadata fields alongside the request body so later nodes can still access them.
24. Connect **Fan-Out: 4 FX Variations** → **Build FX Request Body**.

## 5. Configure Seedance generation request
25. Add an **HTTP Request** node.
26. Name it **Seedance: Generate FX Clip**.
27. Configure:
   - **Method:** `POST`
   - **URL:** Seedance generation endpoint
   - **Authentication:** Header auth or predefined credential, depending on Seedance requirements
   - **Send Body:** JSON
28. Map the request body from the previous Code node.
29. Enable full response parsing if useful.
30. Confirm the response includes a job identifier, such as `jobId`.
31. Connect **Build FX Request Body** → **Seedance: Generate FX Clip**.

## 6. Merge job ID with variation metadata
32. Add a **Code** node.
33. Name it **Merge FX Job ID + Metadata**.
34. In code, combine:
   - Seedance response job ID
   - variation name
   - original prompt
   - shot code
   - all technical fields
35. Return one item per variation job.
36. Connect **Seedance: Generate FX Clip** → **Merge FX Job ID + Metadata**.

## 7. Add polling request
37. Add an **HTTP Request** node.
38. Name it **Poll: Check FX Job Status**.
39. Configure:
   - **Method:** likely `GET`
   - **URL:** Seedance job status endpoint
   - include the job ID in the URL, query, or body
   - same authentication as the generation call
40. Make sure the response returns fields like:
   - `status`
   - output URL when complete
   - optional error reason if failed
41. Connect **Merge FX Job ID + Metadata** → **Poll: Check FX Job Status**.

## 8. Add completion branching
42. Add an **If** node.
43. Name it **FX Render Complete?**
44. Configure a condition on the job status, such as:
   - `status` equals `completed`
45. Ideally add handling for failed/cancelled states too. Since the JSON only shows two outputs, a common approach is:
   - true = completed
   - false = not complete yet
   For production, consider replacing this with a Switch node or additional logic to fail explicitly on `failed`.
46. Connect **Poll: Check FX Job Status** → **FX Render Complete?**

## 9. Add the wait loop
47. Add a **Wait** node.
48. Name it **Wait 20s**.
49. Configure a **20-second** pause.
50. Connect the **false** output of **FX Render Complete?** → **Wait 20s**.
51. Connect **Wait 20s** → **Poll: Check FX Job Status** to create the loop.

## 10. Build final asset metadata
52. Add a **Code** node.
53. Name it **Build FX Asset Metadata**.
54. On completed renders, construct a metadata object containing:
   - `shotCode`
   - `variationName`
   - `videoUrl`
   - `fxType`
   - `scale`
   - `intensity`
   - `cameraMovement`
   - `frameStart`
   - `frameEnd`
   - `fps`
   - tags
   - a generated file name
55. Connect the **true** output of **FX Render Complete?** → **Build FX Asset Metadata**.

## 11. Log the asset in Notion
56. Add a **Notion** node.
57. Name it **Notion: Save FX Asset Record**.
58. Create/connect a **Notion credential**.
59. Configure the node to **create a page/database record** in your target database.
60. Map properties from metadata, for example:
   - Title = shot code + variation name
   - Shot Code
   - FX Type
   - Intensity
   - Scale
   - Camera Movement
   - FPS
   - Frame Range
   - Video URL
   - Tags
61. Connect **Build FX Asset Metadata** → **Notion: Save FX Asset Record**.

## 12. Download the rendered video
62. Add an **HTTP Request** node.
63. Name it **Download FX Video**.
64. Configure:
   - **Method:** `GET`
   - **URL:** use the completed `videoUrl`
   - **Response Format:** File/Binary
   - **Binary Property:** e.g. `data`
65. Increase timeout if rendered clips are large.
66. Connect **Build FX Asset Metadata** → **Download FX Video**.

## 13. Upload the file to Google Drive
67. Add a **Google Drive** node.
68. Name it **Google Drive: Upload FX Clip**.
69. Create/connect a **Google Drive OAuth2 credential**.
70. Configure the node to **upload a file**.
71. Set:
   - input binary property = `data`
   - file name from metadata, e.g. `{{shotCode}}_{{variationName}}.mp4`
   - target Drive folder ID/path
72. Connect **Download FX Video** → **Google Drive: Upload FX Clip**.

## 14. Aggregate storage and logging outputs
73. Add a **Code** node.
74. Name it **Aggregate All FX Variations**.
75. Connect:
   - **Notion: Save FX Asset Record** → **Aggregate All FX Variations**
   - **Google Drive: Upload FX Clip** → **Aggregate All FX Variations**
76. In code, reconcile Notion and Drive outputs for each variation.
77. Build a final summary payload containing:
   - variation name
   - Drive file link
   - Notion page link
   - original render URL if needed
   - technical metadata
78. If you expect all four variations to complete before one Slack message is sent, add logic to collect all items and format them together. In some n8n builds, this may require more explicit merge/aggregation patterns than a single Code node if branch timing becomes inconsistent.

## 15. Send the team review message in Slack
79. Add a **Slack** node.
80. Name it **Slack: Notify FX Team**.
81. Create/connect a **Slack OAuth2 credential**.
82. Configure it to post a message to the target FX review channel.
83. Format the message with:
   - shot code
   - summary of the brief
   - one section per variation
   - links to Drive and Notion
   - technical settings such as FPS, intensity, scale, camera movement
84. Connect **Aggregate All FX Variations** → **Slack: Notify FX Team**.

## 16. Add workflow-level error alerting
85. Add an **Error Trigger** node.
86. Name it **On Workflow Error**.
87. Add a **Slack** node.
88. Name it **Slack: Error Alert**.
89. Connect **On Workflow Error** → **Slack: Error Alert**.
90. Configure the Slack node to post:
   - workflow name
   - failed node
   - error message
   - execution ID
   - timestamp
91. Use a different Slack channel from the review channel if desired, such as an ops/automation-alerts channel.

## 17. Add credentials
92. Configure credentials for:
   - **Seedance API** in the HTTP Request nodes
   - **Google Drive OAuth2**
   - **Notion API**
   - **Slack OAuth2**
93. Test each credential independently before running the full workflow.

## 18. Add optional canvas annotations
94. Add Sticky Note nodes if you want the same canvas structure as the provided workflow:
   - Workflow Overview
   - Trigger & Validation
   - Prompt Fan-Out & Generation
   - Polling & Render Wait
   - Asset Storage & Logging
   - Review Notification
   - Credentials & Security
   - Error Handler

## 19. Test with sample input
95. Send a POST request to the webhook with a sample payload.
96. Verify:
   - four variation items are created
   - Seedance returns four job IDs
   - polling loops properly
   - completed renders produce metadata
   - Notion records are created
   - videos upload to Drive
   - Slack message contains all expected links and details

## 20. Recommended hardening before production
97. Add explicit handling for render statuses like `failed` or `cancelled`.
98. Add retry/backoff policies to HTTP nodes.
99. Add idempotency or duplicate-detection if the same shot code can be submitted twice.
100. Consider storing the original request and execution IDs in Notion for auditability.
101. If a single Slack message must only be sent after all four variations are done, ensure aggregation explicitly waits for all four completed variation records rather than relying on branch arrival timing alone.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow title in JSON: Seedance AI FX Concept Pipeline with Multi-Variation Rendering, Notion Logging, and Slack Review | Internal workflow name |
| User-facing title: Generate multi-variation FX concept clips with Seedance AI, Google Drive, Notion and Slack | Provided title |
| The exported JSON does not include the internal code for Code nodes, nor configured parameter bodies for HTTP Request, Slack, Notion, Google Drive, Webhook, If, or Wait nodes. Reconstruction therefore relies on the workflow description and node naming. | Implementation constraint |
| All sticky note nodes exist but their content is empty in the provided JSON. | Canvas annotation note |
| The workflow contains two entry points: the main webhook and a separate error trigger branch. | Architecture note |
| No sub-workflow or Execute Workflow node is present. | Architecture note |