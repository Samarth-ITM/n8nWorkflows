Upload videos to Facebook Pages and Groups with Facebook Graph API

https://n8nworkflows.xyz/workflows/upload-videos-to-facebook-pages-and-groups-with-facebook-graph-api-13366


# Upload videos to Facebook Pages and Groups with Facebook Graph API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Upload videos to Facebook Pages and Groups with Facebook Graph API

**Purpose:**  
This workflow uploads a video to a Facebook **Page or Group** using the **Facebook Graph Video API** (chunked upload protocol). It supports two entry modes:
- **Manual/UI submission** via an n8n Form (upload file + parameters)
- **Programmatic invocation** from another workflow (Execute Workflow Trigger)

**Core logical blocks**
- **1.1 Input Reception (two entry points):** Collects message, file, target_id, access_token.
- **1.2 Preparation / Normalization:** Extracts and standardizes metadata and keeps binary file data.
- **1.3 Facebook Upload Session Start:** Calls Graph API `upload_phase=start` to obtain session and offsets.
- **1.4 Combine Data for Transfer:** Merges metadata + session info.
- **1.5 Chunk Transfer:** Calls Graph API `upload_phase=transfer` with the binary file.
- **1.6 Finish / Publish:** Calls Graph API `upload_phase=finish` to finalize upload and publish with description.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Two Entry Points)

**Overview:**  
This block provides two ways to start the workflow: a user-facing form (file upload) or a trigger that allows another workflow to pass the same payload.

**Nodes Involved:**
- Trigger: Video Upload Form
- Triggered by Another Workflow

#### Node: **Trigger: Video Upload Form**
- **Type / Role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) — interactive entry point in n8n UI.
- **Configuration (interpreted):**
  - Form title: “Upload Video to Facebook”
  - Description: “Submit video, message, target_id, access_token”
  - Fields:
    - `message` (textarea, required)
    - `File` (file upload, single file, required) → stored in binary as `File0`
    - `target_id` (text, required) — Facebook Page ID or Group ID
    - `access_token` (text, required) — token used in Graph API calls (expects Page/Group token with video publishing permission)
  - Webhook path is auto-set by n8n.
- **Key variables produced:**
  - `$json.message`, `$json.target_id`, `$json.access_token`
  - `$binary.File0` (video file)
- **Connections:**
  - Output → **Prepare Video Metadata**
- **Failure / edge cases:**
  - Missing required fields prevents submission.
  - Very large files may hit n8n instance upload/body limits depending on hosting/reverse proxy settings.
  - Binary field name mismatch (expects `File0` downstream).
- **Version notes:** Form Trigger `typeVersion: 2` (ensure your n8n supports Form Trigger v2).

#### Node: **Triggered by Another Workflow**
- **Type / Role:** `Execute Workflow Trigger` (`n8n-nodes-base.executeWorkflowTrigger`) — entry point when called as a sub-workflow.
- **Configuration (interpreted):**
  - `inputSource: passthrough` — forwards incoming JSON/binary as-is.
- **Expected input contract:**
  - JSON fields: `message`, `target_id`, `access_token`
  - Binary: `File0` containing the video
- **Connections:**
  - Output → **Prepare Video Metadata**
- **Failure / edge cases:**
  - If caller does not pass `File0` binary, upload will fail later (transfer step).
  - If caller passes different binary property name, downstream expressions won’t find it.
- **Version notes:** `typeVersion: 1.1` (requires relatively recent n8n).

---

### 2.2 Preparation / Normalization

**Overview:**  
Standardizes the incoming payload into consistent fields and ensures binary data is carried forward.

**Nodes Involved:**
- Prepare Video Metadata

#### Node: **Prepare Video Metadata**
- **Type / Role:** `Set` (`n8n-nodes-base.set`) — maps/renames fields, keeps binary.
- **Configuration (interpreted):**
  - Creates/sets:
    - `message` = `{{$json.message}}`
    - `target_id` = `{{$json.target_id}}`
    - `access_token` = `{{$json.access_token}}`
    - `file_size` = `{{ $binary.File0.bytes }}` (byte length used by Graph API start phase)
    - `fileSize` = `{{$binary.File0.fileSize}}` (appears informational; not used later)
  - Option: **includeBinary: true** (critical so the file survives into HTTP nodes)
- **Connections:**
  - Output → **Initiate Video Upload**
  - Output → **Merge Upload Paths** (input 0)
- **Failure / edge cases:**
  - If `$binary.File0` is missing, expressions for `file_size`/`fileSize` fail and/or become undefined.
  - `file_size` is stored as a string; Facebook usually accepts numeric strings, but strictness may vary.
- **Version notes:** `typeVersion: 3` Set node behavior differs across major versions; includeBinary must be explicitly enabled as here.

---

### 2.3 Facebook Upload Session Start (upload_phase=start)

**Overview:**  
Creates a Facebook upload session and obtains upload offsets and a session ID.

**Nodes Involved:**
- Initiate Video Upload

#### Node: **Initiate Video Upload**
- **Type / Role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — calls Facebook Graph Video API to start upload.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://graph-video.facebook.com/v24.0/{{$json.target_id}}/videos`
  - Content-Type: `multipart-form-data`
  - Body parameters:
    - `upload_phase` = `start`
    - `file_size` = `{{$json.file_size}}`
    - `access_token` = `{{$json.access_token}}`
- **Expected output (from Facebook):**
  - Typically includes `upload_session_id`, `start_offset`, `end_offset` (depending on API behavior)
- **Connections:**
  - Output → **Merge Upload Paths** (input 1)
- **Failure / edge cases:**
  - Invalid/expired token → 400/401 with Graph API error payload.
  - Missing permissions (e.g., Page/Group publishing permissions) → Graph API permission error.
  - Wrong `target_id` type (page vs group) or posting restrictions → Graph API error.
  - API version pinned to `v24.0`; future deprecations may require bumping version.
- **Version notes:** HTTP Request `typeVersion: 4`.

---

### 2.4 Combine Data for Transfer

**Overview:**  
Merges the metadata (including binary file) with the session data returned by the start call, so the transfer step has everything it needs.

**Nodes Involved:**
- Merge Upload Paths

#### Node: **Merge Upload Paths**
- **Type / Role:** `Merge` (`n8n-nodes-base.merge`) — combines two incoming items into one.
- **Configuration (interpreted):**
  - Mode: `combine`
  - Combine by: `combineAll` (pairs/combines all inputs)
- **Inputs:**
  - Input 0 from **Prepare Video Metadata**
  - Input 1 from **Initiate Video Upload**
- **Output:**
  - A merged item containing:
    - session fields like `upload_session_id`, `start_offset`…
    - metadata fields like `access_token`, `target_id`, message
    - binary `File0`
- **Connections:**
  - Output → **Upload Video Chunk**
- **Failure / edge cases:**
  - If one branch produces zero items (e.g., Initiate fails), merge may output nothing and downstream won’t run.
  - If multiple items are produced unexpectedly, combineAll can create a Cartesian product (rare here but important if upstream is ever changed).
- **Version notes:** `typeVersion: 3.2`.

---

### 2.5 Chunk Transfer (upload_phase=transfer)

**Overview:**  
Transfers the actual video file content to Facebook using the session info. This workflow performs a single “transfer” request using the provided binary; it does not iterate offsets in a loop.

**Nodes Involved:**
- Upload Video Chunk

#### Node: **Upload Video Chunk**
- **Type / Role:** `HTTP Request` — sends video file as multipart binary.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://graph-video.facebook.com/v24.0/{{$node['Prepare Video Metadata'].json.target_id}}/videos`
  - Content-Type: `multipart-form-data`
  - Body parameters:
    - `upload_phase` = `transfer`
    - `upload_session_id` = `{{$json.upload_session_id}}` (from merged session output)
    - `start_offset` = `{{$json.start_offset}}`
    - `access_token` = `{{$node['Prepare Video Metadata'].json.access_token}}`
    - `video_file_chunk` = binary form field from `File0`
- **Input / output connections:**
  - Input ← **Merge Upload Paths**
  - Output → **Complete Video Upload**
- **Failure / edge cases:**
  - **Not a true chunk loop:** Facebook’s chunked upload is typically iterative until `start_offset == end_offset`. This workflow only sends one transfer request; for large files Facebook may require multiple transfer calls.
  - If `start_offset` is expected to change between calls, a single call may fail or upload partially.
  - If binary is too large for a single request given your n8n hosting limits, transfer can fail (413 Payload Too Large, socket hangup, timeouts).
  - Token/permission errors as above.
- **Version notes:** HTTP Request `typeVersion: 4`.

---

### 2.6 Finish / Publish (upload_phase=finish)

**Overview:**  
Finalizes the upload session and publishes the video with the provided description/message.

**Nodes Involved:**
- Complete Video Upload

#### Node: **Complete Video Upload**
- **Type / Role:** `HTTP Request` — finalizes the upload.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://graph-video.facebook.com/v24.0/{{$node['Prepare Video Metadata'].json.target_id}}/videos`
  - Content-Type: `multipart-form-data`
  - Body parameters:
    - `upload_phase` = `finish`
    - `upload_session_id` = `{{$node['Initiate Video Upload'].json.upload_session_id}}`
    - `description` = `{{$node['Prepare Video Metadata'].json.message}}`
    - `access_token` = `{{$node['Prepare Video Metadata'].json.access_token}}`
- **Connections:**
  - Input ← **Upload Video Chunk**
  - Output → (none; end of workflow)
- **Failure / edge cases:**
  - If `upload_session_id` isn’t available (start phase failed), finish fails.
  - If transfer did not complete successfully, finish may return errors about incomplete upload.
  - Posting restrictions (group settings, page restrictions) may prevent publishing.
- **Version notes:** HTTP Request `typeVersion: 4`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: Video Upload Form | Form Trigger | UI form entry point with file upload | — | Prepare Video Metadata | Purpose: Accepts input via n8n UI<br>Notes: Required fields must be filled; you can prefill defaults for testing |
| Triggered by Another Workflow | Execute Workflow Trigger | Entry point for being called by another workflow | — | Prepare Video Metadata | Allows workflow to be called from another workflow |
| Prepare Video Metadata | Set | Normalize fields; keep binary; compute file size | Trigger: Video Upload Form; Triggered by Another Workflow | Initiate Video Upload; Merge Upload Paths | Organizes input data for upload |
| Initiate Video Upload | HTTP Request | Graph API `upload_phase=start` | Prepare Video Metadata | Merge Upload Paths | Calls Facebook Graph API to start the upload session |
| Merge Upload Paths | Merge | Combine metadata + session info | Prepare Video Metadata; Initiate Video Upload | Upload Video Chunk | Combines metadata and upload session info for transfer |
| Upload Video Chunk | HTTP Request | Graph API `upload_phase=transfer` with binary | Merge Upload Paths | Complete Video Upload | Uploads video in chunks |
| Complete Video Upload | HTTP Request | Graph API `upload_phase=finish` and publish | Upload Video Chunk | — | Completes the upload and posts the video |
| Sticky Note | Sticky Note | Annotation | — | — | Purpose: Accepts input via n8n UI<br>Notes: Required fields must be filled; you can prefill defaults for testing |
| Sticky Note1 | Sticky Note | Annotation | — | — | Allows workflow to be called from another workflow |
| Sticky Note2 | Sticky Note | Annotation | — | — | Completes the upload and posts the video |
| Sticky Note3 | Sticky Note | Annotation | — | — | Uploads video in chunks |
| Sticky Note4 | Sticky Note | Annotation | — | — | Combines metadata and upload session info for transfer |
| Sticky Note5 | Sticky Note | Annotation | — | — | Calls Facebook Graph API to start the upload session |
| Sticky Note6 | Sticky Note | Annotation | — | — | Organizes input data for upload |
| Sticky Note7 | Sticky Note | Annotation / Overview panel | — | — | ## Facebook Video Upload Workflow<br>**Purpose:** Uploads videos to Facebook Pages or Groups automatically, either via a Form Trigger or triggered by another workflow.<br>**Features:** - Accepts video file, message, target_id (Page/Group ID), and access_token. - Prepares metadata and handles chunked uploads for large videos. - Initiates, transfers, and completes upload using Facebook Graph API. - Modular and reusable for integration into larger automation pipelines.<br>**Who it’s for:** Content creators, social media managers, agencies, or SaaS builders wanting automated Facebook video posting.<br>**Setup:** 1. Import workflow into n8n (cloud or self-hosted). 2. Activate workflow. 3. Fill in target_id and Page Access Token in the form or pass via sub-workflow. 4. Upload video and message. 5. Video is posted automatically.<br>**Requirements:** - n8n instance (cloud or self-hosted) - Facebook Page/Group Access Token with `publish_video` permission - Video file to upload |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add entry point #1: Form Trigger**
   - Add node: **Form Trigger**
   - Set:
     - Form Title: `Upload Video to Facebook`
     - Form Description: `Submit video, message, target_id, access_token`
   - Add fields (all required):
     1) Textarea: label `message`
     2) File: label `File`, single file (ensure it results in binary property `File0`)
     3) Text: label `target_id` (placeholder “Page or Group ID”)
     4) Text: label `access_token` (placeholder “Page Access Token”)
   - Keep default webhook/path generated by n8n.

3. **Add entry point #2: Execute Workflow Trigger**
   - Add node: **Execute Workflow Trigger**
   - Set **Input Source** to `passthrough`
   - This allows other workflows to call this workflow and pass `{ message, target_id, access_token }` plus binary `File0`.

4. **Add “Prepare Video Metadata” (Set node)**
   - Add node: **Set**
   - Add fields:
     - `message` = expression `{{$json.message}}`
     - `target_id` = `{{$json.target_id}}`
     - `access_token` = `{{$json.access_token}}`
     - `file_size` = `{{ $binary.File0.bytes }}`
     - `fileSize` = `{{$binary.File0.fileSize}}` (optional; not required by downstream nodes)
   - In options, enable **Include Binary Data**.

5. **Connect triggers to Set**
   - Connect **Form Trigger → Prepare Video Metadata**
   - Connect **Execute Workflow Trigger → Prepare Video Metadata**

6. **Add “Initiate Video Upload” (HTTP Request)**
   - Add node: **HTTP Request**
   - Configure:
     - Method: `POST`
     - URL: `https://graph-video.facebook.com/v24.0/{{$json.target_id}}/videos`
     - Send Body: enabled
     - Content Type: `multipart-form-data`
     - Body parameters:
       - `upload_phase`: `start`
       - `file_size`: `{{$json.file_size}}`
       - `access_token`: `{{$json.access_token}}`
   - Credentials:
     - This workflow uses a raw `access_token` field, not an n8n Facebook credential. No OAuth credential is required in-node; the token is sent as form data.

7. **Add “Merge Upload Paths” (Merge node)**
   - Add node: **Merge**
   - Mode: `Combine`
   - Combine By: `Combine All`

8. **Wire Set + Initiate into Merge**
   - Connect **Prepare Video Metadata → Merge Upload Paths** (to **Input 0**)
   - Connect **Initiate Video Upload → Merge Upload Paths** (to **Input 1**)
   - Also connect **Prepare Video Metadata → Initiate Video Upload**

9. **Add “Upload Video Chunk” (HTTP Request)**
   - Add node: **HTTP Request**
   - Configure:
     - Method: `POST`
     - URL: `https://graph-video.facebook.com/v24.0/{{$node['Prepare Video Metadata'].json.target_id}}/videos`
     - Content Type: `multipart-form-data`
     - Body parameters:
       - `upload_phase`: `transfer`
       - `upload_session_id`: `{{$json.upload_session_id}}`
       - `start_offset`: `{{$json.start_offset}}`
       - `access_token`: `{{$node['Prepare Video Metadata'].json.access_token}}`
       - `video_file_chunk`: **Binary** → select *Form Binary Data*, input field name `File0`
   - Connect **Merge Upload Paths → Upload Video Chunk**

10. **Add “Complete Video Upload” (HTTP Request)**
   - Add node: **HTTP Request**
   - Configure:
     - Method: `POST`
     - URL: `https://graph-video.facebook.com/v24.0/{{$node['Prepare Video Metadata'].json.target_id}}/videos`
     - Content Type: `multipart-form-data`
     - Body parameters:
       - `upload_phase`: `finish`
       - `upload_session_id`: `{{$node['Initiate Video Upload'].json.upload_session_id}}`
       - `description`: `{{$node['Prepare Video Metadata'].json.message}}`
       - `access_token`: `{{$node['Prepare Video Metadata'].json.access_token}}`
   - Connect **Upload Video Chunk → Complete Video Upload**

11. **Operational constraints to set (recommended)**
   - If self-hosted behind a reverse proxy, increase max upload sizes/timeouts (e.g., Nginx `client_max_body_size`, proxy timeouts).
   - Consider adding retry/error handling nodes around HTTP calls.

12. **Sub-workflow usage (if called by another workflow)**
   - In the parent workflow, use **Execute Workflow** node targeting this workflow.
   - Pass input JSON with `message`, `target_id`, `access_token`.
   - Pass the video as binary with property name **`File0`** (must match this workflow’s expectations).
   - Expect final output to be whatever Facebook returns on `finish` (typically includes the video id or success payload).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Facebook Video Upload Workflow — purpose, features, setup steps, requirements (as described in the workflow canvas note). | Embedded sticky note content (overview panel) |
| API endpoints used are `graph-video.facebook.com/v24.0/{target_id}/videos` with phases `start`, `transfer`, `finish`. | Facebook Graph Video API usage in HTTP nodes |
| Current design performs a single `transfer` call; true chunked upload may require looping until offsets indicate completion. | Important implementation constraint for large videos |

