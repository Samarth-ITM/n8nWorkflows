Generate clean plates and automate object removal with Seedance AI

https://n8nworkflows.xyz/workflows/generate-clean-plates-and-automate-object-removal-with-seedance-ai-14387


# Generate clean plates and automate object removal with Seedance AI

# 1. Workflow Overview

This workflow automates AI-assisted clean plate generation for VFX object removal using Seedance AI. It receives a shot request through a webhook, validates and enriches the input, generates four render-pass variants from a reference plate image, polls Seedance until each render finishes, performs a simplified QC decision, then stores, logs, and distributes the results.

Typical use cases:
- Generating clean plates for paint/comp teams
- Creating multiple AI-assisted removal interpretations for a shot
- Producing a QC-oriented difference preview
- Automatically notifying artists and supervisors when output is ready or needs review

## 1.1 Input Reception and Validation
The workflow begins with an HTTP POST webhook. Incoming payload fields are validated, normalized, and supplemented with defaults such as `projectId`, `sequenceCode`, `frameRange`, and `qcThreshold`.

## 1.2 Pass Generation and Seedance Submission
The validated request is expanded into four parallel pass definitions:
- Primary clean plate
- Alternative clean plate
- Foreground removal
- Difference map preview

Each pass gets a tailored prompt and is converted into a Seedance image-to-video request body using the source plate image.

## 1.3 Polling and Render Completion
After submission, each Seedance job ID is stored with the original metadata. The workflow then polls the render job repeatedly every 20 seconds until the status becomes `succeeded`.

## 1.4 QC and Compositing Preparation
When a render completes, the workflow extracts the output video URL, computes a simplified QC score, and decides whether the result is automatically approved. Approved passes receive an auto-generated Nuke script template for downstream compositing.

## 1.5 Asset Storage, Logging, and Aggregation
Approved outputs are downloaded, uploaded to Google Drive, and logged to a Notion database. The workflow then aggregates the completed passes into a summary payload.

## 1.6 Team Notifications
A final summary is sent to Slack and Telegram with pass-level QC information, links, Nuke script metadata, and delivery folder structure.

## 1.7 Global Error Handling
A separate error-trigger branch catches workflow-level failures and sends an alert to Slack.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

**Overview:**  
This block accepts the clean plate generation request and ensures the minimum required fields are present. It also standardizes optional values so downstream nodes can assume a complete payload.

**Nodes Involved:**  
- Webhook: Clean Plate Request
- Validate & Extract Input

### Node: Webhook: Clean Plate Request
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point node that exposes an HTTP POST endpoint.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `clean-plate-request`
  - Webhook ID is defined for stable endpoint handling.
- **Key expressions or variables used:**  
  None in the node itself.
- **Input and output connections:**  
  - No input
  - Outputs to `Validate & Extract Input`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Incorrect HTTP method
  - Malformed JSON body
  - Upstream caller not sending expected `body` content
- **Sub-workflow reference:**  
  None

### Node: Validate & Extract Input
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the webhook payload, validates required fields, and emits a normalized object.
- **Configuration choices:**  
  The code expects request data in `$input.first().json.body`.
- **Key expressions or variables used:**  
  Validates required:
  - `plateImageUrl`
  - `removalBrief`
  - `shotCode`

  Produces:
  - `plateImageUrl`
  - `removalBrief`
  - `shotCode`
  - `sequenceCode` from payload or `shotCode.split('_')[0]` fallback
  - `projectId` default `PROJ-001`
  - `objectType` default `unwanted_object`
  - `frameRange` default `1-100`
  - `frameRate` default `24`
  - `qcThreshold` default `0.85`
  - `notionDatabaseId`
  - `supervisorEmail`
  - `requestTimestamp`
- **Input and output connections:**  
  - Input: `Webhook: Clean Plate Request`
  - Output: `Fan-Out: 4 Clean Plate Passes`
- **Version-specific requirements:**  
  Code node type version `2`
- **Edge cases or potential failure types:**  
  - Throws explicit error if a required field is missing or blank
  - `shotCode.split('_')[0]` may produce unexpected `sequenceCode` if naming convention differs
  - `objectType` default is `unwanted_object`, but downstream prompt mapping expects `wire`, `sign`, `crew`, `rig`, or general fallback
- **Sub-workflow reference:**  
  None

---

## 2.2 Pass Generation and Seedance Submission

**Overview:**  
This block creates four pass variants for the same shot and sends each as an image-to-video generation request to Seedance AI. It also preserves metadata needed for later QC and delivery.

**Nodes Involved:**  
- Fan-Out: 4 Clean Plate Passes
- Build Clean Plate Request
- Seedance: Generate Clean Pass
- Merge Job ID + Metadata

### Node: Fan-Out: 4 Clean Plate Passes
- **Type and technical role:** `n8n-nodes-base.code`  
  Expands a single validated request into four items, one per render pass.
- **Configuration choices:**  
  Uses `objectType` to inject more specific removal instructions into the prompt.
- **Key expressions or variables used:**  
  Builds:
  - `variantId`
  - `passType`
  - `label`
  - `safePrompt`

  Pass variants:
  - `CP-V1` / `clean_plate`
  - `CP-V2` / `clean_plate_alt`
  - `CP-V3` / `foreground_removal`
  - `CP-V4` / `difference_preview`

  Prompt construction includes:
  - `removalBrief`
  - object-specific instruction
  - `shotCode`
  - `frameRange`
  - `frameRate`
  - static camera flags such as `--duration 5 --camerafixed true`
- **Input and output connections:**  
  - Input: `Validate & Extract Input`
  - Output: `Build Clean Plate Request`
- **Version-specific requirements:**  
  Code node type version `2`
- **Edge cases or potential failure types:**  
  - Unknown `objectType` falls back to the general instruction
  - Prompt text could become too long depending on API limits
  - JSON escaping is intentionally handled using `JSON.stringify(...).slice(1,-1)`
- **Sub-workflow reference:**  
  None

### Node: Build Clean Plate Request
- **Type and technical role:** `n8n-nodes-base.code`  
  Builds the actual Seedance API JSON payload for image-to-video generation.
- **Configuration choices:**  
  - Model: `seedance-1-5-pro-251215`
  - Uses content array with:
    - text prompt
    - image URL reference
  - `generate_audio: false`
  - `ratio: adaptive`
  - `duration: 5`
  - `watermark: false`
  - Hardcodes mode as `image_to_video`
- **Key expressions or variables used:**  
  Consumes:
  - `safePrompt`
  - `plateImageUrl`

  Produces:
  - `requestBody` as JSON string
  - `mode`
- **Input and output connections:**  
  - Input: `Fan-Out: 4 Clean Plate Passes`
  - Output: `Seedance: Generate Clean Pass`
- **Version-specific requirements:**  
  Code node type version `2`
- **Edge cases or potential failure types:**  
  - Invalid or inaccessible `plateImageUrl`
  - Model identifier may become deprecated if Seedance changes versions
- **Sub-workflow reference:**  
  None

### Node: Seedance: Generate Clean Pass
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the render request to Seedance.
- **Configuration choices:**  
  - Method: `POST`
  - Sends JSON body parsed from `requestBody`
  - Sends headers:
    - `Authorization`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `={{ JSON.parse($json.requestBody) }}`
- **Input and output connections:**  
  - Input: `Build Clean Plate Request`
  - Output: `Merge Job ID + Metadata`
- **Version-specific requirements:**  
  HTTP Request type version `4.3`
- **Edge cases or potential failure types:**  
  - Missing URL configuration in exported JSON; must be set manually
  - Authorization header has no value in this node and must be supplied
  - 401/403 auth failures
  - 400 payload schema errors
  - API timeout or rate limiting
- **Sub-workflow reference:**  
  None

### Node: Merge Job ID + Metadata
- **Type and technical role:** `n8n-nodes-base.code`  
  Combines the Seedance job response with the original per-pass metadata.
- **Configuration choices:**  
  Reads HTTP response from current input and metadata from `Build Clean Plate Request`.
- **Key expressions or variables used:**  
  - `$('Build Clean Plate Request').first().json`
  - Exposes:
    - `id`
    - `jobStatus`
- **Input and output connections:**  
  - Input: `Seedance: Generate Clean Pass`
  - Output: `Poll: Check Job Status`
- **Version-specific requirements:**  
  Code node type version `2`
- **Edge cases or potential failure types:**  
  - Assumes response includes `id` and `status`
  - `.first()` can be problematic in multi-item scenarios if item linking is not preserved as intended
- **Sub-workflow reference:**  
  None

---

## 2.3 Polling and Render Completion

**Overview:**  
This block polls Seedance repeatedly until each render job finishes. If the render is still in progress, execution waits 20 seconds and retries.

**Nodes Involved:**  
- Poll: Check Job Status
- Render Complete?
- Wait 20s

### Node: Poll: Check Job Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries Seedance for the latest job state.
- **Configuration choices:**  
  - Sends headers with bearer authorization
  - URL is currently configured as `=`, so it is incomplete and must be replaced
- **Key expressions or variables used:**  
  None effectively usable in current exported state
- **Input and output connections:**  
  - Input: `Merge Job ID + Metadata` and loopback from `Wait 20s`
  - Output: `Render Complete?`
- **Version-specific requirements:**  
  HTTP Request type version `4.3`
- **Edge cases or potential failure types:**  
  - This node is not functional as exported because the URL is blank
  - Bearer token is hardcoded as `Bearer YOUR_TOKEN_HERE`
  - If polling endpoint schema differs, downstream `status` check will fail
- **Sub-workflow reference:**  
  None

### Node: Render Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes completed jobs to metadata/QC and incomplete jobs back into the wait loop.
- **Configuration choices:**  
  Checks whether `{{$json.status}} === "succeeded"`.
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Poll: Check Job Status`
  - True output: `Build Clean Plate Metadata + QC`
  - False output: `Wait 20s`
- **Version-specific requirements:**  
  IF node type version `2`
- **Edge cases or potential failure types:**  
  - Jobs that fail, cancel, or return `error` status are not handled explicitly; they will loop forever unless status changes
  - Missing `status` field causes false branch behavior
- **Sub-workflow reference:**  
  None

### Node: Wait 20s
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution before the next poll attempt.
- **Configuration choices:**  
  Wait amount: `20` seconds
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Render Complete?` false branch
  - Output: `Poll: Check Job Status`
- **Version-specific requirements:**  
  Wait node type version `1.1`
- **Edge cases or potential failure types:**  
  - Long-running jobs may accumulate many executions
  - No max retry limit is defined
- **Sub-workflow reference:**  
  None

---

## 2.4 QC and Compositing Preparation

**Overview:**  
This block extracts the output video URL, calculates a simplified QC score, and determines if the pass is auto-approved. Approved passes receive a Nuke compositing script; rejected ones trigger Slack review alerts.

**Nodes Involved:**  
- Build Clean Plate Metadata + QC
- QC Check: Passes Threshold?
- Generate Nuke Comp Script
- Slack: QC Failed — Artist Review

### Node: Build Clean Plate Metadata + QC
- **Type and technical role:** `n8n-nodes-base.code`  
  Standardizes finished render metadata and computes a simplified QC decision.
- **Configuration choices:**  
  Attempts to derive `videoUrl` from several possible response fields.
- **Key expressions or variables used:**  
  Inputs from:
  - current poll response
  - `$('Merge Job ID + Metadata').first().json`

  `videoUrl` fallback order:
  - `input.content.video_url`
  - `input.output_url`
  - `input.video_url`
  - fallback text with job ID

  QC logic:
  - `1080p` => `1.0`
  - `720p` => `0.85`
  - else `0.7`
  - `qcPassed = qcScore >= qcThreshold`
- **Input and output connections:**  
  - Input: `Render Complete?` true branch
  - Output: `QC Check: Passes Threshold?`
- **Version-specific requirements:**  
  Code node type version `2`
- **Edge cases or potential failure types:**  
  - QC is simulated, not image-based
  - Missing resolution may force low score
  - `.first()` reference may mismatch items in concurrent fan-out
  - If no URL field exists, later download step will fail
- **Sub-workflow reference:**  
  None

### Node: QC Check: Passes Threshold?
- **Type and technical role:** `n8n-nodes-base.if`  
  Separates approved outputs from failed outputs.
- **Configuration choices:**  
  Checks whether `{{$json.qcPassed}}` is `true`.
- **Key expressions or variables used:**  
  - `={{ $json.qcPassed }}`
- **Input and output connections:**  
  - Input: `Build Clean Plate Metadata + QC`
  - True output: `Generate Nuke Comp Script`
  - False output: `Slack: QC Failed — Artist Review`
- **Version-specific requirements:**  
  IF node type version `2`
- **Edge cases or potential failure types:**  
  - Any non-boolean or missing `qcPassed` may route unexpectedly under strict validation
- **Sub-workflow reference:**  
  None

### Node: Generate Nuke Comp Script
- **Type and technical role:** `n8n-nodes-base.code`  
  Creates a ready-to-use Nuke script template and folder metadata for delivery.
- **Configuration choices:**  
  Generates:
  - Nuke `.nk` text content
  - script filename
  - script path
  - folder structure object
- **Key expressions or variables used:**  
  Uses:
  - `shotCode`
  - `sequenceCode`
  - `objectType`
  - `passType`
  - `qcScore`
  - `qcPassed`
  - `plateImageUrl`
  - `videoUrl`

  Produces:
  - `nukeScript`
  - `nukeScriptName`
  - `nukeScriptPath`
  - `folderStructure.cleanPlates`
  - `folderStructure.differenceMaps`
  - `folderStructure.nukeScripts`
- **Input and output connections:**  
  - Input: `QC Check: Passes Threshold?` true branch
  - Outputs to:
    - `Download Clean Plate Video`
    - `Create a database page`
- **Version-specific requirements:**  
  Code node type version `2`
- **Edge cases or potential failure types:**  
  - File paths are illustrative and may not match infrastructure
  - Assumes Nuke v15 syntax and Linux-style paths
  - Generated script is not actually saved anywhere in this workflow
- **Sub-workflow reference:**  
  None

### Node: Slack: QC Failed — Artist Review
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends an alert for passes that did not meet QC threshold.
- **Configuration choices:**  
  - OAuth2 authentication
  - Sends formatted review message to Slack channel `social`
- **Key expressions or variables used:**  
  Uses:
  - `shotCode`
  - `objectType`
  - `removalBrief`
  - `videoUrl`
  - `generatedAt`
- **Input and output connections:**  
  - Input: `QC Check: Passes Threshold?` false branch
  - No downstream connection
- **Version-specific requirements:**  
  Slack node type version `2.3`
- **Edge cases or potential failure types:**  
  - Slack auth or channel permission errors
  - Broken `videoUrl`
- **Sub-workflow reference:**  
  None

---

## 2.5 Asset Storage and Logging

**Overview:**  
Approved videos are downloaded and uploaded to Google Drive, while metadata is written to Notion. These steps create both file storage and pipeline traceability.

**Nodes Involved:**  
- Download Clean Plate Video
- Google Drive: Upload Clean Plate
- Create a database page

### Node: Download Clean Plate Video
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the rendered video as binary data.
- **Configuration choices:**  
  - URL from `{{$json.videoUrl}}`
  - Response format: file
- **Key expressions or variables used:**  
  - `={{ $json.videoUrl }}`
- **Input and output connections:**  
  - Input: `Generate Nuke Comp Script`
  - Output: `Google Drive: Upload Clean Plate`
- **Version-specific requirements:**  
  HTTP Request type version `4.3`
- **Edge cases or potential failure types:**  
  - Invalid or expiring video URL
  - Large file download timeout
  - If `videoUrl` contains fallback text instead of a URL, request fails
- **Sub-workflow reference:**  
  None

### Node: Google Drive: Upload Clean Plate
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the downloaded clean plate video to Google Drive.
- **Configuration choices:**  
  - Drive: `My Drive`
  - Fixed folder ID
  - File name pattern:
    `shotCode_passType_variantId.mp4`
  - Uses Google Drive OAuth2 credential `Automations.ai`
- **Key expressions or variables used:**  
  - `{{ $('Generate Nuke Comp Script').first().json.shotCode }}`
  - `{{ $('Generate Nuke Comp Script').first().json.passType }}`
  - `{{ $('Generate Nuke Comp Script').first().json.variantId }}`
- **Input and output connections:**  
  - Input: `Download Clean Plate Video`
  - Output: `Aggregate All Passes`
- **Version-specific requirements:**  
  Google Drive node type version `3`
- **Edge cases or potential failure types:**  
  - Folder ID may not exist or credential may lack access
  - Uses `.first()` which may not be item-safe in parallel execution
  - Binary property selection is not explicit in export; verify upload source in n8n UI
- **Sub-workflow reference:**  
  None

### Node: Create a database page
- **Type and technical role:** `n8n-nodes-base.notion`  
  Logs approved pass metadata to a Notion database.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Title: `AI-Assisted Clean Plate`
  - Fixed database ID in node config
  - Writes selected fields into Notion properties
- **Key expressions or variables used:**  
  Properties mapped:
  - `plateImageUrl`
  - `projectId`
  - `sequenceCode`
  - `variantId` as title
  - `videoUrl`
- **Input and output connections:**  
  - Input: `Generate Nuke Comp Script`
  - Output: `Aggregate All Passes`
- **Version-specific requirements:**  
  Notion node type version `2.2`
- **Edge cases or potential failure types:**  
  - Database schema must already contain matching property names/types
  - Node ignores dynamic `notionDatabaseId` from webhook payload and uses a hardcoded database
  - Auth/scopes may fail if integration is not shared to the database
- **Sub-workflow reference:**  
  None

---

## 2.6 Aggregation and Team Notifications

**Overview:**  
This block consolidates all approved passes into one summary and dispatches it to team communication channels. It is intended to provide comp/paint teams with links, QC results, and Nuke metadata in one message.

**Nodes Involved:**  
- Aggregate All Passes
- Slack: Notify Paint/Comp Team
- Send Telegram1

### Node: Aggregate All Passes
- **Type and technical role:** `n8n-nodes-base.code`  
  Collects pass results and formats a combined summary.
- **Configuration choices:**  
  Builds a human-readable `passLines` summary using all items from `Generate Nuke Comp Script`.
- **Key expressions or variables used:**  
  - `$input.all()`
  - `$('Generate Nuke Comp Script').all()`

  Produces:
  - `passLines`
  - `totalPasses`
  - `allQcPassed`
  - first pass's `nukeScript`, `nukeScriptName`, `folderStructure`
- **Input and output connections:**  
  - Inputs from:
    - `Create a database page`
    - `Google Drive: Upload Clean Plate`
  - Outputs to:
    - `Slack: Notify Paint/Comp Team`
    - `Send Telegram1`
- **Version-specific requirements:**  
  Code node type version `2`
- **Edge cases or potential failure types:**  
  - This node may run more than once because it has two incoming branches and no explicit merge node
  - It only aggregates approved passes, not QC-failed ones
  - Uses first generated Nuke script only, even if each pass has a distinct script name
- **Sub-workflow reference:**  
  None

### Node: Slack: Notify Paint/Comp Team
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the aggregated delivery summary to Slack.
- **Configuration choices:**  
  - OAuth2 authentication
  - Channel `social`
  - Includes large formatted message with pass details and embedded Nuke script text
- **Key expressions or variables used:**  
  Uses:
  - `shotCode`
  - `allQcPassed`
  - `sequenceCode`
  - `objectType`
  - `removalBrief`
  - `totalPasses`
  - `passLines`
  - `folderStructure`
  - `nukeScriptName`
  - `nukeScript`
  - `generatedAt`
- **Input and output connections:**  
  - Input: `Aggregate All Passes`
  - No downstream connection
- **Version-specific requirements:**  
  Slack node type version `2.3`
- **Edge cases or potential failure types:**  
  - Message may exceed Slack length limits if all script text is included
  - Channel access/auth issues
- **Sub-workflow reference:**  
  None

### Node: Send Telegram1
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a compact summary to Telegram.
- **Configuration choices:**  
  - Fixed `chatId`
  - `parse_mode: Markdown`
  - Uses Telegram credential `Saurabh Telegram account 2`
- **Key expressions or variables used:**  
  Similar summary fields as Slack, excluding embedded Nuke script body.
- **Input and output connections:**  
  - Input: `Aggregate All Passes`
  - No downstream connection
- **Version-specific requirements:**  
  Telegram node type version `1.1`
- **Edge cases or potential failure types:**  
  - Markdown escaping issues if fields contain special characters
  - Bot permission or chat ID errors
- **Sub-workflow reference:**  
  None

---

## 2.7 Global Error Handling

**Overview:**  
This branch listens for workflow execution failures and posts an error message to Slack. It provides a basic operational alerting mechanism.

**Nodes Involved:**  
- On Workflow Error
- Slack: Error Alert

### Node: On Workflow Error
- **Type and technical role:** `n8n-nodes-base.errorTrigger`  
  Starts a separate execution when the workflow fails.
- **Configuration choices:**  
  No custom parameters.
- **Key expressions or variables used:**  
  None directly
- **Input and output connections:**  
  - No input
  - Output: `Slack: Error Alert`
- **Version-specific requirements:**  
  Error Trigger type version `1`
- **Edge cases or potential failure types:**  
  - Only catches workflow-level errors visible to n8n
  - Does not replace explicit handling for partial-pass failures
- **Sub-workflow reference:**  
  None

### Node: Slack: Error Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a simple failure notification to Slack.
- **Configuration choices:**  
  - OAuth2 authentication
  - Fixed channel `social`
  - Includes error message and timestamp
- **Key expressions or variables used:**  
  - `{{ $json.message }}`
  - `{{ new Date().toISOString() }}`
- **Input and output connections:**  
  - Input: `On Workflow Error`
  - No downstream connection
- **Version-specific requirements:**  
  Slack node type version `2.3`
- **Edge cases or potential failure types:**  
  - Slack auth/channel issues
  - Error payload structure could vary; `message` may be absent in some failure types
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook: Clean Plate Request | Webhook | Receives POST clean plate requests |  | Validate & Extract Input | ## 📡 Trigger & Validation<br>Accepts incoming shot requests via POST webhook and validates required fields (`plateImageUrl`, `removalBrief`, `shotCode`). Enriches the payload with sensible defaults for optional fields like frame range, frame rate, project ID, and QC threshold before handing off downstream. |
| Validate & Extract Input | Code | Validates required input and applies defaults | Webhook: Clean Plate Request | Fan-Out: 4 Clean Plate Passes | ## 📡 Trigger & Validation<br>Accepts incoming shot requests via POST webhook and validates required fields (`plateImageUrl`, `removalBrief`, `shotCode`). Enriches the payload with sensible defaults for optional fields like frame range, frame rate, project ID, and QC threshold before handing off downstream. |
| Fan-Out: 4 Clean Plate Passes | Code | Generates four pass variants and prompts | Validate & Extract Input | Build Clean Plate Request | ## 🎞️ Pass Generation & AI Render<br>Fans out into four distinct render passes and builds image-to-video requests using the plate image as reference. Each pass is submitted to the Seedance API and the returned job ID is merged with the original metadata for downstream polling. |
| Build Clean Plate Request | Code | Builds Seedance request payload | Fan-Out: 4 Clean Plate Passes | Seedance: Generate Clean Pass | ## 🎞️ Pass Generation & AI Render<br>Fans out into four distinct render passes and builds image-to-video requests using the plate image as reference. Each pass is submitted to the Seedance API and the returned job ID is merged with the original metadata for downstream polling. |
| Seedance: Generate Clean Pass | HTTP Request | Submits generation job to Seedance | Build Clean Plate Request | Merge Job ID + Metadata | ## 🎞️ Pass Generation & AI Render<br>Fans out into four distinct render passes and builds image-to-video requests using the plate image as reference. Each pass is submitted to the Seedance API and the returned job ID is merged with the original metadata for downstream polling. |
| Merge Job ID + Metadata | Code | Combines job response with original pass metadata | Seedance: Generate Clean Pass | Poll: Check Job Status | ## 🎞️ Pass Generation & AI Render<br>Fans out into four distinct render passes and builds image-to-video requests using the plate image as reference. Each pass is submitted to the Seedance API and the returned job ID is merged with the original metadata for downstream polling. |
| Poll: Check Job Status | HTTP Request | Polls Seedance job status | Merge Job ID + Metadata, Wait 20s | Render Complete? | ## ⏳ Polling & Job Completion<br>Polls the Seedance API every 20 seconds until the render job reports `succeeded`. If still processing, execution waits and retries automatically. Once complete, metadata and QC scores are assembled for the next stage. |
| Render Complete? | If | Routes succeeded jobs or loops pending jobs | Poll: Check Job Status | Build Clean Plate Metadata + QC, Wait 20s | ## ⏳ Polling & Job Completion<br>Polls the Seedance API every 20 seconds until the render job reports `succeeded`. If still processing, execution waits and retries automatically. Once complete, metadata and QC scores are assembled for the next stage. |
| Wait 20s | Wait | Delays next polling attempt | Render Complete? | Poll: Check Job Status | ## ⏳ Polling & Job Completion<br>Polls the Seedance API every 20 seconds until the render job reports `succeeded`. If still processing, execution waits and retries automatically. Once complete, metadata and QC scores are assembled for the next stage. |
| Build Clean Plate Metadata + QC | Code | Extracts output URL and computes QC | Render Complete? | QC Check: Passes Threshold? | ## 🔍 QC Check & Nuke Script<br>Scores each rendered pass against the configured QC threshold. Passes that meet the bar get a Nuke compositing script auto-generated with Read, Merge, Grade, and Write nodes pre-wired. Failed passes trigger an immediate artist-review alert in Slack. |
| QC Check: Passes Threshold? | If | Splits approved vs failed QC passes | Build Clean Plate Metadata + QC | Generate Nuke Comp Script, Slack: QC Failed — Artist Review | ## 🔍 QC Check & Nuke Script<br>Scores each rendered pass against the configured QC threshold. Passes that meet the bar get a Nuke compositing script auto-generated with Read, Merge, Grade, and Write nodes pre-wired. Failed passes trigger an immediate artist-review alert in Slack. |
| Generate Nuke Comp Script | Code | Builds Nuke comp script and delivery paths | QC Check: Passes Threshold? | Download Clean Plate Video, Create a database page | ## 🔍 QC Check & Nuke Script<br>Scores each rendered pass against the configured QC threshold. Passes that meet the bar get a Nuke compositing script auto-generated with Read, Merge, Grade, and Write nodes pre-wired. Failed passes trigger an immediate artist-review alert in Slack. |
| Slack: QC Failed — Artist Review | Slack | Notifies team of failed QC pass | QC Check: Passes Threshold? |  | ## 🔍 QC Check & Nuke Script<br>Scores each rendered pass against the configured QC threshold. Passes that meet the bar get a Nuke compositing script auto-generated with Read, Merge, Grade, and Write nodes pre-wired. Failed passes trigger an immediate artist-review alert in Slack. |
| Download Clean Plate Video | HTTP Request | Downloads rendered video file | Generate Nuke Comp Script | Google Drive: Upload Clean Plate | ## ☁️ Asset Storage & Logging<br>Downloads the rendered video and uploads it to Google Drive under the correct shot folder. Simultaneously logs the pass metadata — variant ID, video URL, plate URL, project ID — to a Notion database for pipeline tracking and review. |
| Google Drive: Upload Clean Plate | Google Drive | Uploads approved video to Drive | Download Clean Plate Video | Aggregate All Passes | ## ☁️ Asset Storage & Logging<br>Downloads the rendered video and uploads it to Google Drive under the correct shot folder. Simultaneously logs the pass metadata — variant ID, video URL, plate URL, project ID — to a Notion database for pipeline tracking and review. |
| Create a database page | Notion | Logs approved pass metadata to Notion | Generate Nuke Comp Script | Aggregate All Passes | ## ☁️ Asset Storage & Logging<br>Downloads the rendered video and uploads it to Google Drive under the correct shot folder. Simultaneously logs the pass metadata — variant ID, video URL, plate URL, project ID — to a Notion database for pipeline tracking and review. |
| Aggregate All Passes | Code | Builds final summary from approved passes | Create a database page, Google Drive: Upload Clean Plate | Slack: Notify Paint/Comp Team, Send Telegram1 | ## 📣 Team Notifications<br>Aggregates all four passes into a single summary and sends it to both Slack and Telegram. The message includes per-pass QC scores, video links, Nuke script content, and folder paths — giving the paint and comp team everything they need at a glance. |
| Slack: Notify Paint/Comp Team | Slack | Sends final delivery summary to Slack | Aggregate All Passes |  | ## 📣 Team Notifications<br>Aggregates all four passes into a single summary and sends it to both Slack and Telegram. The message includes per-pass QC scores, video links, Nuke script content, and folder paths — giving the paint and comp team everything they need at a glance. |
| Send Telegram1 | Telegram | Sends final delivery summary to Telegram | Aggregate All Passes |  | ## 📣 Team Notifications<br>Aggregates all four passes into a single summary and sends it to both Slack and Telegram. The message includes per-pass QC scores, video links, Nuke script content, and folder paths — giving the paint and comp team everything they need at a glance. |
| On Workflow Error | Error Trigger | Catches workflow-level failures |  | Slack: Error Alert | ## ⚠️ Error Handler<br>Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. |
| Slack: Error Alert | Slack | Sends failure alert to Slack | On Workflow Error |  | ## ⚠️ Error Handler<br>Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. |
| 📋 Overview | Sticky Note | Documentation note |  |  | ## 🎬 AI Clean Plate Generator<br><br>### How it works<br>This workflow automates AI-assisted clean plate generation for VFX production. When triggered via webhook, it fans out four parallel render passes — primary clean plate, an alternative take, a foreground-removal pass, and a difference map — using the Seedance AI video model with your original plate image as reference.<br><br>Each pass is polled until complete, then scored against a configurable QC threshold. Passes that clear the threshold are logged to Notion, uploaded to Google Drive, and shipped with a ready-to-use Nuke compositing script. The whole team gets notified via Slack and Telegram when results are ready.<br><br>### Setup steps<br>1. **Webhook** — Copy the webhook URL from the trigger node and use it in your shot management tool or pipeline script.<br>2. **Seedance API** — Replace the `Authorization` header value in `Seedance: Generate Clean Pass` and `Poll: Check Job Status` with your own Seedance API key stored as an n8n credential.<br>3. **Google Drive** — Connect a Google Drive OAuth2 credential and update the target folder ID in `Google Drive: Upload Clean Plate`.<br>4. **Notion** — Connect a Notion integration credential and set the correct database ID for your clean plate log.<br>5. **Slack & Telegram** — Connect credentials for both notification nodes and update the channel/chat IDs to match your team's setup.<br>6. **QC Threshold** — Send `qcThreshold` (0–1) in the webhook payload to control auto-approval sensitivity. Defaults to `0.85`. |
| Section: Trigger & Validation | Sticky Note | Section annotation |  |  | ## 📡 Trigger & Validation<br>Accepts incoming shot requests via POST webhook and validates required fields (`plateImageUrl`, `removalBrief`, `shotCode`). Enriches the payload with sensible defaults for optional fields like frame range, frame rate, project ID, and QC threshold before handing off downstream. |
| Section: Pass Generation & AI Render | Sticky Note | Section annotation |  |  | ## 🎞️ Pass Generation & AI Render<br>Fans out into four distinct render passes and builds image-to-video requests using the plate image as reference. Each pass is submitted to the Seedance API and the returned job ID is merged with the original metadata for downstream polling. |
| Section: Polling & Job Completion | Sticky Note | Section annotation |  |  | ## ⏳ Polling & Job Completion<br>Polls the Seedance API every 20 seconds until the render job reports `succeeded`. If still processing, execution waits and retries automatically. Once complete, metadata and QC scores are assembled for the next stage. |
| Section: QC Check & Nuke Script | Sticky Note | Section annotation |  |  | ## 🔍 QC Check & Nuke Script<br>Scores each rendered pass against the configured QC threshold. Passes that meet the bar get a Nuke compositing script auto-generated with Read, Merge, Grade, and Write nodes pre-wired. Failed passes trigger an immediate artist-review alert in Slack. |
| Section: Asset Storage & Logging | Sticky Note | Section annotation |  |  | ## ☁️ Asset Storage & Logging<br>Downloads the rendered video and uploads it to Google Drive under the correct shot folder. Simultaneously logs the pass metadata — variant ID, video URL, plate URL, project ID — to a Notion database for pipeline tracking and review. |
| Section: Team Notifications | Sticky Note | Section annotation |  |  | ## 📣 Team Notifications<br>Aggregates all four passes into a single summary and sends it to both Slack and Telegram. The message includes per-pass QC scores, video links, Nuke script content, and folder paths — giving the paint and comp team everything they need at a glance. |
| Security Notes | Sticky Note | Security annotation |  |  | ## 🔐 Credentials & Security<br>Use n8n credential store for all secrets — Seedance API key, Google Drive OAuth2, Notion API, Slack OAuth2, and Telegram Bot token. Never hardcode tokens in node headers. Replace all sample IDs and folder paths with values from your own project. |
| Section: Error Handler | Sticky Note | Error-handling annotation |  |  | ## ⚠️ Error Handler<br>Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Generate Clean Plates with Seedance and Automate Object Removal, QC, and Delivery`.
   - Leave it inactive until credentials and URLs are configured.

2. **Add the webhook trigger**
   - Add a **Webhook** node.
   - Name it `Webhook: Clean Plate Request`.
   - Set:
     - Method: `POST`
     - Path: `clean-plate-request`
   - Keep default response behavior unless you want a custom immediate response.

3. **Add input validation**
   - Add a **Code** node after the webhook.
   - Name it `Validate & Extract Input`.
   - Paste logic equivalent to:
     - Read `body` from the incoming webhook payload
     - Require:
       - `plateImageUrl`
       - `removalBrief`
       - `shotCode`
     - Set defaults:
       - `sequenceCode`: payload value or prefix from `shotCode`
       - `projectId`: `PROJ-001`
       - `objectType`: `unwanted_object`
       - `frameRange`: `1-100`
       - `frameRate`: `24`
       - `qcThreshold`: `0.85`
       - `notionDatabaseId`: `null`
       - `supervisorEmail`: `null`
       - `requestTimestamp`: current ISO timestamp
   - Connect `Webhook: Clean Plate Request` → `Validate & Extract Input`.

4. **Add the pass fan-out**
   - Add a **Code** node.
   - Name it `Fan-Out: 4 Clean Plate Passes`.
   - Configure it to create four items from one input:
     - `CP-V1` / `clean_plate` / `Clean Plate – Primary`
     - `CP-V2` / `clean_plate_alt` / `Clean Plate – Alternative`
     - `CP-V3` / `foreground_removal` / `Foreground Removal Pass`
     - `CP-V4` / `difference_preview` / `Difference Map Preview`
   - Build prompts from:
     - `removalBrief`
     - `objectType`
     - `shotCode`
     - `frameRange`
     - `frameRate`
   - Include static camera prompt suffix such as `--duration 5 --camerafixed true`.
   - Connect `Validate & Extract Input` → `Fan-Out: 4 Clean Plate Passes`.

5. **Build the Seedance request payload**
   - Add a **Code** node.
   - Name it `Build Clean Plate Request`.
   - Configure it to output:
     - `model: seedance-1-5-pro-251215`
     - `content` array:
       - text prompt
       - `image_url` pointing to `plateImageUrl`
     - `generate_audio: false`
     - `ratio: adaptive`
     - `duration: 5`
     - `watermark: false`
     - `mode: image_to_video`
   - Store the request body as JSON or a stringified JSON field.
   - Connect `Fan-Out: 4 Clean Plate Passes` → `Build Clean Plate Request`.

6. **Create the Seedance submission node**
   - Add an **HTTP Request** node.
   - Name it `Seedance: Generate Clean Pass`.
   - Set:
     - Method: `POST`
     - URL: your Seedance generation endpoint
     - Send Body: `true`
     - Body Content Type: `JSON`
     - Body: parse or directly send the generated request object
   - Add headers:
     - `Authorization: Bearer <Seedance API key>`
     - `Content-Type: application/json`
   - Best practice: create a generic HTTP header or API credential in n8n instead of hardcoding.
   - Connect `Build Clean Plate Request` → `Seedance: Generate Clean Pass`.

7. **Merge submission response with metadata**
   - Add a **Code** node.
   - Name it `Merge Job ID + Metadata`.
   - Configure it to combine:
     - current HTTP response fields such as job `id` and `status`
     - original pass metadata from `Build Clean Plate Request`
   - Connect `Seedance: Generate Clean Pass` → `Merge Job ID + Metadata`.

8. **Add polling request**
   - Add another **HTTP Request** node.
   - Name it `Poll: Check Job Status`.
   - Set:
     - Method: typically `GET`
     - URL: your Seedance status endpoint, usually using the job ID from the prior node
   - Example pattern:
     - `https://.../jobs/{{ $json.id }}`
   - Add header:
     - `Authorization: Bearer <Seedance API key>`
   - Connect `Merge Job ID + Metadata` → `Poll: Check Job Status`.

9. **Add completion check**
   - Add an **If** node.
   - Name it `Render Complete?`.
   - Configure condition:
     - left value: `{{$json.status}}`
     - operator: `equals`
     - right value: `succeeded`
   - Connect `Poll: Check Job Status` → `Render Complete?`.

10. **Add the wait loop**
    - Add a **Wait** node.
    - Name it `Wait 20s`.
    - Set wait duration to `20 seconds`.
    - Connect:
      - `Render Complete?` false branch → `Wait 20s`
      - `Wait 20s` → `Poll: Check Job Status`

11. **Add metadata normalization and QC**
    - Add a **Code** node.
    - Name it `Build Clean Plate Metadata + QC`.
    - Configure it to:
      - Resolve `videoUrl` from possible API response fields
      - Keep:
        - `variantId`
        - `passType`
        - `label`
        - `shotCode`
        - `sequenceCode`
        - `projectId`
        - `plateImageUrl`
        - `removalBrief`
        - `objectType`
        - `frameRange`
        - `frameRate`
        - `notionDatabaseId`
        - `supervisorEmail`
      - Add render metadata:
        - `jobId`
        - `resolution`
        - `duration`
        - `generatedAt`
      - Compute simplified QC:
        - `1080p = 1.0`
        - `720p = 0.85`
        - else `0.7`
      - Compare to `qcThreshold`
      - Set `qcPassed`
      - Build a `tags` object for downstream logging
   - Connect `Render Complete?` true branch → `Build Clean Plate Metadata + QC`.

12. **Add QC decision**
    - Add an **If** node.
    - Name it `QC Check: Passes Threshold?`.
    - Condition:
      - `{{$json.qcPassed}}` equals `true`
    - Connect `Build Clean Plate Metadata + QC` → `QC Check: Passes Threshold?`.

13. **Add QC failure alert**
    - Add a **Slack** node.
    - Name it `Slack: QC Failed — Artist Review`.
    - Authentication: Slack OAuth2
    - Select the target channel.
    - Compose a message including:
      - shot code
      - object type
      - brief
      - result URL
      - generated timestamp
    - Connect `QC Check: Passes Threshold?` false branch → `Slack: QC Failed — Artist Review`.

14. **Generate the Nuke script**
    - Add a **Code** node.
    - Name it `Generate Nuke Comp Script`.
    - Build an `.nk` script string containing:
      - original plate `Read`
      - AI clean plate `Read`
      - difference map merge
      - grade node
      - final merge
      - color correction
      - two write nodes
    - Also emit:
      - `nukeScriptName`
      - `nukeScriptPath`
      - `folderStructure.cleanPlates`
      - `folderStructure.differenceMaps`
      - `folderStructure.nukeScripts`
    - Connect `QC Check: Passes Threshold?` true branch → `Generate Nuke Comp Script`.

15. **Download the rendered video**
    - Add an **HTTP Request** node.
    - Name it `Download Clean Plate Video`.
    - Set URL to `{{$json.videoUrl}}`
    - Configure response format as **File**
    - Connect `Generate Nuke Comp Script` → `Download Clean Plate Video`.

16. **Upload to Google Drive**
    - Add a **Google Drive** node.
    - Name it `Google Drive: Upload Clean Plate`.
    - Create/connect a **Google Drive OAuth2** credential.
    - Operation: upload file
    - Select target drive and folder.
    - File name pattern:
      - `{{shotCode}}_{{passType}}_{{variantId}}.mp4`
    - Ensure the node uses the binary file from the previous HTTP Request node.
    - Connect `Download Clean Plate Video` → `Google Drive: Upload Clean Plate`.

17. **Log metadata to Notion**
    - Add a **Notion** node.
    - Name it `Create a database page`.
    - Create/connect a **Notion API** credential.
    - Resource: `Database Page`
    - Select your Notion database.
    - Map properties matching your schema, for example:
      - `variantId` as title
      - `plateImageUrl`
      - `projectId`
      - `sequenceCode`
      - `videoUrl`
    - Connect `Generate Nuke Comp Script` → `Create a database page`.

18. **Aggregate approved passes**
    - Add a **Code** node.
    - Name it `Aggregate All Passes`.
    - Configure it to:
      - collect generated pass items
      - format a `passLines` summary per pass
      - calculate `totalPasses`
      - calculate `allQcPassed`
      - expose one representative `nukeScript`, `nukeScriptName`, and `folderStructure`
    - Connect:
      - `Google Drive: Upload Clean Plate` → `Aggregate All Passes`
      - `Create a database page` → `Aggregate All Passes`

19. **Add Slack summary notification**
    - Add a **Slack** node.
    - Name it `Slack: Notify Paint/Comp Team`.
    - Authentication: Slack OAuth2
    - Target channel: your paint/comp or production channel
    - Compose a summary including:
      - shot and sequence
      - object type and brief
      - total passes
      - per-pass QC lines
      - folder structure
      - Nuke script name
      - optionally the full script body
    - Connect `Aggregate All Passes` → `Slack: Notify Paint/Comp Team`.

20. **Add Telegram summary notification**
    - Add a **Telegram** node.
    - Name it `Send Telegram1`.
    - Create/connect a **Telegram Bot** credential.
    - Set `chatId`
    - Enable Markdown parse mode
    - Use a similar summary message, but keep it shorter than Slack
    - Connect `Aggregate All Passes` → `Send Telegram1`.

21. **Add global error handling**
    - Add an **Error Trigger** node.
    - Name it `On Workflow Error`.
    - Add a **Slack** node named `Slack: Error Alert`.
    - Configure Slack OAuth2 and send errors to an ops/admin channel.
    - Message should include:
      - workflow error message
      - timestamp
    - Connect `On Workflow Error` → `Slack: Error Alert`.

22. **Add documentation sticky notes**
    - Create sticky notes for:
      - overview
      - trigger & validation
      - pass generation & AI render
      - polling & job completion
      - QC check & Nuke script
      - asset storage & logging
      - team notifications
      - error handler
      - security notes

23. **Configure credentials**
    - **Seedance API**
      - Use an HTTP credential or header-based auth
      - Replace all placeholder Authorization values
    - **Google Drive OAuth2**
      - Must have upload permission to the target folder
    - **Notion API**
      - Integration must be invited to the database
    - **Slack OAuth2**
      - Must be able to post to selected channels
    - **Telegram Bot**
      - Must be authorized to message the target chat

24. **Test with sample payload**
    - Send a POST request to the webhook with JSON like:
      - `plateImageUrl`
      - `removalBrief`
      - `shotCode`
      - optional `sequenceCode`, `projectId`, `objectType`, `frameRange`, `frameRate`, `qcThreshold`
    - Verify:
      - 4 pass items are created
      - Seedance jobs are submitted
      - polling returns `succeeded`
      - approved passes are downloaded/uploaded/logged
      - notifications are sent

25. **Recommended hardening before production**
    - Replace hardcoded IDs with environment variables or workflow variables
    - Add explicit handling for `failed`, `cancelled`, or timed-out Seedance jobs
    - Add a retry limit in polling
    - Use a Merge node before aggregation if you need deterministic synchronization
    - Save the generated Nuke script as an actual file if downstream tools need it directly
    - If you want dynamic Notion routing, use `notionDatabaseId` from the webhook instead of a fixed database ID

**Sub-workflow setup:**  
This workflow does **not** invoke any sub-workflow nodes. There are multiple entry points:
- Main entry point: `Webhook: Clean Plate Request`
- Error entry point: `On Workflow Error`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Clean Plate Generator overview explains the workflow automates four AI render passes, QC, Notion logging, Google Drive upload, Nuke script generation, and Slack/Telegram notifications. | Workflow overview sticky note |
| Setup guidance: copy the webhook URL into your shot management or pipeline tooling. | Webhook configuration |
| Setup guidance: replace Authorization headers in `Seedance: Generate Clean Pass` and `Poll: Check Job Status` with your Seedance API key stored in n8n credentials. | Seedance integration |
| Setup guidance: connect Google Drive OAuth2 and update the destination folder in `Google Drive: Upload Clean Plate`. | Google Drive integration |
| Setup guidance: connect Notion and point the Notion node to the clean plate logging database. | Notion integration |
| Setup guidance: connect Slack and Telegram credentials and update channel/chat IDs. | Notification setup |
| Setup guidance: `qcThreshold` can be sent in the webhook payload to control auto-approval sensitivity; default is `0.85`. | Runtime behavior |
| Use n8n credential store for all secrets; do not hardcode API keys or tokens. | Security note |
| Replace all sample IDs, folder IDs, and file paths with project-specific values. | Security / deployment note |