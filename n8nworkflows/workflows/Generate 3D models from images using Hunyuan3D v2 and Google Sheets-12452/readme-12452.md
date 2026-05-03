Generate 3D models from images using Hunyuan3D v2 and Google Sheets

https://n8nworkflows.xyz/workflows/generate-3d-models-from-images-using-hunyuan3d-v2-and-google-sheets-12452


# Generate 3D models from images using Hunyuan3D v2 and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
Automate the conversion of 2D images (via URL) into 3D `.glb` models using **Fal.ai’s Hunyuan3D v2 queue API**, and write the resulting `.glb` download URL back to **Google Sheets**.

**Target use cases:** rapid asset prototyping for games, e-commerce 3D viewers, AR/VR asset generation, and batch processing of product or concept images.

### 1.1 Triggers & Input Reception
Two entry points (manual + scheduled) fetch rows from Google Sheets to find images to process.

### 1.2 Data Normalization
Maps the sheet’s `IMAGE_URL` column into a predictable internal field used by the API request.

### 1.3 AI Job Submission (Fal.ai Queue)
Submits an async generation request to Fal.ai (Hunyuan3D v2), receiving a `request_id`.

### 1.4 Polling Loop (Wait → Status → IF)
Waits 30 seconds, checks job status, loops until `status === "COMPLETED"`.

### 1.5 Result Retrieval & Persistence
Fetches the final job output and updates the originating Google Sheets row with the `.glb` link.

---

## 2. Block-by-Block Analysis

### Block 1 — Triggers & Sheet Row Retrieval
**Overview:** Starts the workflow either manually or on a schedule, then reads data from Google Sheets to obtain candidate images (typically rows where `RESULT_GLB` is empty).  
**Nodes involved:**  
- Schedule Trigger  
- When clicking ‘Test workflow’  
- Get new image  

#### Node: **Schedule Trigger**
- **Type / role:** `Schedule Trigger` — time-based entry point.
- **Configuration (interpreted):**
  - Runs on an interval measured in **minutes**.
  - The UI note says “15-minute schedule”, but the JSON only indicates `field: "minutes"` and does not include an explicit numeric value. You must set the actual interval in n8n.
- **Outputs:** Connects to **Get new image**.
- **Failure modes / edge cases:**
  - Misconfigured schedule interval (too frequent may hit API limits; too slow delays processing).

#### Node: **When clicking ‘Test workflow’**
- **Type / role:** `Manual Trigger` — manual entry point for testing.
- **Configuration:** No parameters.
- **Outputs:** Connects to **Get new image**.
- **Edge cases:** None (except no downstream items if the sheet read returns nothing).

#### Node: **Get new image**
- **Type / role:** `Google Sheets` — retrieves rows from a spreadsheet.
- **Configuration (interpreted):**
  - **Document** and **Sheet** are not selected in the JSON (empty values). You must choose them.
  - The node’s **operation is not specified** in the provided JSON (n8n defaults vary by version/UI). Practically, this node must be configured to **read/get rows**.
  - Intended behavior (per description/sticky notes): fetch rows where `RESULT_GLB` is empty and `IMAGE_URL` is present.
- **Inputs:** From **Schedule Trigger** or **Manual Trigger**.
- **Outputs:** To **Set Data**.
- **Version notes:** Node typeVersion `4.5`.
- **Failure modes / edge cases:**
  - Missing Google Sheets OAuth2 credentials.
  - Sheet headers not matching expected names (`IMAGE_URL`, `RESULT_GLB`) → expressions will fail later.
  - No filtering configured → may reprocess already completed rows unless you add a filter.
  - Empty results → downstream nodes receive no items (no-op run).

---

### Block 2 — Data Normalization
**Overview:** Renames/mirrors the sheet column into a stable field `image_url` for API usage.  
**Nodes involved:**  
- Set Data

#### Node: **Set Data**
- **Type / role:** `Set` — creates/overwrites fields on each item.
- **Configuration (interpreted):**
  - Adds a string field `image_url` with expression:  
    - `image_url = {{$json['IMAGE_URL']}}`
- **Inputs:** From **Get new image**.
- **Outputs:** To **Submit to Hunyuan3D**.
- **Failure modes / edge cases:**
  - If `IMAGE_URL` column is missing or blank, `image_url` becomes empty → Fal.ai request will fail or generate an error.

---

### Block 3 — Submit to Fal.ai (Hunyuan3D v2)
**Overview:** Starts an asynchronous 3D generation job on Fal.ai’s queue endpoint and receives a `request_id` used for subsequent polling and retrieval.  
**Nodes involved:**  
- Submit to Hunyuan3D

#### Node: **Submit to Hunyuan3D**
- **Type / role:** `HTTP Request` — POST to Fal.ai queue to create a job.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://queue.fal.run/fal-ai/hunyuan3d/v2`
  - **Body:** JSON (explicit), sent as request body:
    - `input_image_url`: `{{$json.image_url}}`
    - `num_inference_steps`: `50`
    - `guidance_scale`: `7.5`
    - `octree_resolution`: `256`
    - `textured_mesh`: `true`
  - **Authentication:** `genericCredentialType` → `httpHeaderAuth`
    - Intended credential: Header `Authorization: Key YOUR_API_KEY`
- **Inputs:** From **Set Data**.
- **Outputs:** To **Wait 30 sec**.
- **Version notes:** Node typeVersion `4.2`.
- **Failure modes / edge cases:**
  - 401/403 if Authorization header is missing/invalid.
  - 4xx if `input_image_url` is invalid/unreachable.
  - Rate limits / 429 from Fal.ai if too many requests.
  - If Fal.ai response doesn’t include `request_id`, later nodes referencing it will fail.

---

### Block 4 — Polling Loop Until Completed
**Overview:** Waits 30 seconds between checks, requests the current job status, and loops until it becomes `COMPLETED`.  
**Nodes involved:**  
- Wait 30 sec  
- Check Status  
- Is Completed?

#### Node: **Wait 30 sec**
- **Type / role:** `Wait` — pause execution.
- **Configuration:** Wait `amount = 30` seconds.
- **Inputs:** From **Submit to Hunyuan3D** or from the “not completed” branch of **Is Completed?**
- **Outputs:** To **Check Status**.
- **Failure modes / edge cases:**
  - Long-running jobs could cause many loop iterations; consider n8n execution time limits and Fal.ai queue durations.

#### Node: **Check Status**
- **Type / role:** `HTTP Request` — GET job status from Fal.ai.
- **Configuration (interpreted):**
  - **URL (expression):**  
    `https://queue.fal.run/fal-ai/hunyuan3d/v2/requests/{{ $('Submit to Hunyuan3D').item.json.request_id }}/status`
  - **Authentication:** same `httpHeaderAuth` credential as submission.
- **Inputs:** From **Wait 30 sec**.
- **Outputs:** To **Is Completed?**
- **Version notes:** Node typeVersion `4.2`.
- **Key dependency:** Uses `$('Submit to Hunyuan3D').item.json.request_id`
  - This requires the original submission node to have produced `request_id` and that the item linkage remains valid.
- **Failure modes / edge cases:**
  - Expression failure if `request_id` missing.
  - 404 if request_id not found/expired.
  - Status could be other terminal states (e.g., `FAILED`)—workflow currently doesn’t handle this explicitly.

#### Node: **Is Completed?**
- **Type / role:** `IF` — routes flow based on status.
- **Configuration (interpreted):**
  - Condition: `{{$json.status}}` **equals** `"COMPLETED"` (strict validation).
- **Inputs:** From **Check Status**.
- **Outputs:**
  - **True branch (Completed):** to **Get Final Result**
  - **False branch:** to **Wait 30 sec** (loop)
- **Version notes:** Node typeVersion `2.2`.
- **Failure modes / edge cases:**
  - If Fal.ai returns a status like `IN_PROGRESS`, loop continues (expected).
  - If status is `FAILED` (or similar), loop will continue indefinitely. You likely want an additional condition to break and write an error to the sheet.

---

### Block 5 — Fetch Final Output & Update Google Sheet
**Overview:** Once completed, retrieves the final job payload (which should contain an asset URL) and writes it back to the originating row in Google Sheets.  
**Nodes involved:**  
- Get Final Result  
- Update Result

#### Node: **Get Final Result**
- **Type / role:** `HTTP Request` — GET final job result from Fal.ai.
- **Configuration (interpreted):**
  - **URL (expression):**  
    `https://queue.fal.run/fal-ai/hunyuan3d/v2/requests/{{ $('Submit to Hunyuan3D').item.json.request_id }}`
  - **Authentication:** `httpHeaderAuth` (Fal.ai key).
- **Inputs:** From **Is Completed?** (true branch).
- **Outputs:** To **Update Result**.
- **Failure modes / edge cases:**
  - If final payload structure differs (asset URL field name not what you expect), update mapping will fail unless configured carefully.
  - If job completed but assets are missing/empty, you’ll write blank or error.

#### Node: **Update Result**
- **Type / role:** `Google Sheets` — updates an existing row with the `.glb` URL.
- **Configuration (interpreted):**
  - Operation: **update**
  - **Document** and **Sheet** are not selected in JSON (must be configured).
  - The node is missing visible field mappings in JSON; you must configure:
    - How to identify the target row (usually via a row ID from the initial read, or by matching a unique column).
    - Which column to update (`RESULT_GLB`) and what value to write (the `.glb` URL from Fal.ai’s final result).
- **Inputs:** From **Get Final Result**.
- **Outputs:** End of workflow.
- **Version notes:** Node typeVersion `4.5`.
- **Failure modes / edge cases:**
  - Update fails if row identifier is not provided.
  - Header mismatch (`RESULT_GLB`) prevents writing to the right column.
  - Concurrent runs could update wrong rows if you don’t use a stable row ID.

---

### Block 6 — Documentation / Canvas Notes (Sticky Notes)
**Overview:** These nodes are non-executable annotations to explain the workflow on the n8n canvas. They don’t affect runtime but are part of the workflow JSON.  
**Nodes involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

#### Node: **Sticky Note**
- **Type / role:** `Sticky Note` — describes “How it works” and “Setup steps”.
- **Content highlights:** Triggers, sheet headers, credentials, and configuration reminders.

#### Node: **Sticky Note1**
- **Type / role:** `Sticky Note` — labels Block 1 (“Input & Data Setup”).

#### Node: **Sticky Note2**
- **Type / role:** `Sticky Note` — labels Block 2 (“Generate & Poll Status”).

#### Node: **Sticky Note3**
- **Type / role:** `Sticky Note` — labels Block 3 (“Save Result”).

#### Node: **Sticky Note4**
- **Type / role:** `Sticky Note` — provides a general description and includes an image link:  
  https://res.cloudinary.com/dfzmjcdxt/image/upload/v1767540624/WhatsApp_Image_2026-01-04_at_8.28.26_PM_dtxhjt.jpg

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point | — | Get new image | ### 1. Input & Data Setup<br>Watches for new data on schedule or manual trigger, then fetches empty rows from Google Sheets. |
| When clicking ‘Test workflow’ | n8n-nodes-base.manualTrigger | Manual entry point for testing | — | Get new image | ### 1. Input & Data Setup<br>Watches for new data on schedule or manual trigger, then fetches empty rows from Google Sheets. |
| Get new image | n8n-nodes-base.googleSheets | Read rows / fetch image URLs | Schedule Trigger; When clicking ‘Test workflow’ | Set Data | ## How it works<br>1. **Trigger:** Starts manually or on a 15-minute schedule.<br>2. **Fetch Data:** Reads image URLs (e.g. fashion sketches, product photos) from Google Sheets.<br>3. **Generate 3D:** Sends the image to Fal.ai (Hunyuan3D v2) for processing.<br>4. **Poll Status:** Loops every 30 seconds to check if the 3D model is ready.<br>5. **Update Sheet:** Retrieves the `.glb` file URL and saves it back to Google Sheets.<br><br>## Setup steps<br>1. **Google Sheet:** Create a sheet with headers `IMAGE_URL` and `RESULT_GLB`.<br>2. **Credentials:** Set up Fal.ai (API Key) and Google Sheets OAuth2 credentials.<br>3. **Configuration:** Select your specific Sheet ID in the 'Get new image' and 'Update Result' nodes.<br><br>### 1. Input & Data Setup<br>Watches for new data on schedule or manual trigger, then fetches empty rows from Google Sheets. |
| Set Data | n8n-nodes-base.set | Normalize field name to `image_url` | Get new image | Submit to Hunyuan3D | ### 2. Generate & Poll Status<br>Submits the image to Hunyuan3D v2, then enters a 30s wait-loop until the generation status is 'COMPLETED'. |
| Submit to Hunyuan3D | n8n-nodes-base.httpRequest | Start Fal.ai async job (queue) | Set Data | Wait 30 sec | ### 2. Generate & Poll Status<br>Submits the image to Hunyuan3D v2, then enters a 30s wait-loop until the generation status is 'COMPLETED'. |
| Wait 30 sec | n8n-nodes-base.wait | Delay between status polls | Submit to Hunyuan3D; Is Completed? (false) | Check Status | ### 2. Generate & Poll Status<br>Submits the image to Hunyuan3D v2, then enters a 30s wait-loop until the generation status is 'COMPLETED'. |
| Check Status | n8n-nodes-base.httpRequest | Poll Fal.ai job status | Wait 30 sec | Is Completed? | ### 2. Generate & Poll Status<br>Submits the image to Hunyuan3D v2, then enters a 30s wait-loop until the generation status is 'COMPLETED'. |
| Is Completed? | n8n-nodes-base.if | Branch: completed vs keep waiting | Check Status | Get Final Result (true); Wait 30 sec (false) | ### 2. Generate & Poll Status<br>Submits the image to Hunyuan3D v2, then enters a 30s wait-loop until the generation status is 'COMPLETED'. |
| Get Final Result | n8n-nodes-base.httpRequest | Retrieve final Fal.ai output | Is Completed? (true) | Update Result | ### 3. Save Result<br>Fetches the final .glb URL and updates the Google Sheet row. |
| Update Result | n8n-nodes-base.googleSheets | Write `.glb` URL back to sheet | Get Final Result | — | ### 3. Save Result<br>Fetches the final .glb URL and updates the Google Sheet row. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — | (This node is itself the note content shown at left) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas block label | — | — | (This node is itself the note content shown at left) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas block label | — | — | (This node is itself the note content shown at left) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas block label | — | — | (This node is itself the note content shown at left) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation + image link | — | — | (This node is itself the note content shown at left) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger (manual)**
   1) Add node: **Manual Trigger**  
   2) Name: `When clicking ‘Test workflow’`

2. **Create Trigger (scheduled)**
   1) Add node: **Schedule Trigger**  
   2) Set an interval (e.g., every **15 minutes** if desired).  
   3) Name: `Schedule Trigger`

3. **Create Google Sheets “read” node**
   1) Add node: **Google Sheets**  
   2) Name: `Get new image`  
   3) Credentials: configure **Google Sheets OAuth2** (n8n credential).  
   4) Select:
      - **Document ID** (your spreadsheet)
      - **Sheet name** (the worksheet tab)
   5) Configure operation to **Read/Get Many rows** (wording depends on n8n UI).
   6) Configure filtering so you only process pending rows:
      - `RESULT_GLB` is empty (and optionally `IMAGE_URL` is not empty).
      - If your node version supports filters, set them there; otherwise, add an **IF** node after reading to filter items.
   7) Connect **Schedule Trigger → Get new image** and **Manual Trigger → Get new image**.

4. **Normalize URL field**
   1) Add node: **Set**  
   2) Name: `Set Data`  
   3) Add field:
      - `image_url` (String) = `{{$json['IMAGE_URL']}}`
   4) Connect: **Get new image → Set Data**

5. **Create Fal.ai credential (Header Auth)**
   1) In n8n credentials, create: **HTTP Header Auth**  
   2) Header name: `Authorization`  
   3) Header value: `Key YOUR_FAL_AI_API_KEY`

6. **Submit job to Hunyuan3D v2**
   1) Add node: **HTTP Request**  
   2) Name: `Submit to Hunyuan3D`  
   3) Method: `POST`  
   4) URL: `https://queue.fal.run/fal-ai/hunyuan3d/v2`  
   5) Authentication: **Generic Credential** → **HTTP Header Auth** (select the credential from step 5)  
   6) Body: JSON, for example:
      - `input_image_url`: `{{$json.image_url}}`
      - `num_inference_steps`: `50`
      - `guidance_scale`: `7.5`
      - `octree_resolution`: `256`
      - `textured_mesh`: `true`
   7) Connect: **Set Data → Submit to Hunyuan3D**

7. **Wait node**
   1) Add node: **Wait**  
   2) Name: `Wait 30 sec`  
   3) Wait: `30 seconds`  
   4) Connect: **Submit to Hunyuan3D → Wait 30 sec**

8. **Poll status**
   1) Add node: **HTTP Request**  
   2) Name: `Check Status`  
   3) Method: `GET`  
   4) Authentication: same **HTTP Header Auth** credential  
   5) URL (expression):  
      `https://queue.fal.run/fal-ai/hunyuan3d/v2/requests/{{ $('Submit to Hunyuan3D').item.json.request_id }}/status`
   6) Connect: **Wait 30 sec → Check Status**

9. **IF node to detect completion**
   1) Add node: **IF**  
   2) Name: `Is Completed?`  
   3) Condition (String):
      - Left: `{{$json.status}}`
      - Operator: equals
      - Right: `COMPLETED`
   4) Connect: **Check Status → Is Completed?**
   5) Connect loop:
      - **Is Completed? (false) → Wait 30 sec**

10. **Fetch final result**
   1) Add node: **HTTP Request**  
   2) Name: `Get Final Result`  
   3) Method: `GET`  
   4) Authentication: same **HTTP Header Auth** credential  
   5) URL (expression):  
      `https://queue.fal.run/fal-ai/hunyuan3d/v2/requests/{{ $('Submit to Hunyuan3D').item.json.request_id }}`
   6) Connect: **Is Completed? (true) → Get Final Result**

11. **Update Google Sheet with `.glb` link**
   1) Add node: **Google Sheets**  
   2) Name: `Update Result`  
   3) Credentials: Google Sheets OAuth2  
   4) Select same Document ID / Sheet name as the read node  
   5) Operation: **Update row**
   6) Configure row targeting:
      - Prefer using the row identifier returned by the “Get new image” node (n8n often provides something like `rowNumber` or an internal row id, depending on the operation).
   7) Map field:
      - Set `RESULT_GLB` to the `.glb` URL found in **Get Final Result** output (you must select the correct JSON path based on Fal.ai’s response structure in your runs).
   8) Connect: **Get Final Result → Update Result**

12. **(Recommended hardening) Add failure handling**
   - Add another IF branch for terminal failure states (e.g., `FAILED`) to avoid infinite loops and optionally write an error message into the sheet.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Fal.ai signup and API key generation | https://fal.ai |
| Workflow canvas image reference | https://res.cloudinary.com/dfzmjcdxt/image/upload/v1767540624/WhatsApp_Image_2026-01-04_at_8.28.26_PM_dtxhjt.jpg |
| Sheet schema expectation | Headers: `IMAGE_URL`, `RESULT_GLB` |
| Key operational constraint | If you do not filter out rows with an existing `RESULT_GLB`, the workflow may reprocess completed items. |
| Polling behavior | Current logic loops forever unless status becomes `COMPLETED`; consider handling `FAILED`/timeouts. |