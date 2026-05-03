Generate and validate bulk QR codes with Google Sheets and Google Drive

https://n8nworkflows.xyz/workflows/generate-and-validate-bulk-qr-codes-with-google-sheets-and-google-drive-13363


# Generate and validate bulk QR codes with Google Sheets and Google Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate and validate QR codes in bulk with Google Sheets and Google Drive

**Purpose:**  
This workflow generates QR-code images in bulk from rows in a Google Sheet, uploads the generated images to Google Drive, and updates each row’s status. It also exposes a validation webhook that is called when a QR code is scanned, validating the ID and marking it as used.

**Target use cases:**
- Event check-in / ticket validation
- Bulk badge / pass generation
- One-time QR access links tied to a unique identifier (here: an email)

**Logical blocks (by dependencies):**

### 1.1 Bulk QR generation orchestration (Manual trigger → Sheets → batch loop)
Reads a Google Sheet, filters rows that still need QR generation, then iterates them in batches.

### 1.2 QR creation + Drive upload + status update
Generates a QR image via an external QR API, uploads it to Google Drive, then updates the row status in Google Sheets to `QR_GENERATED`.

### 1.3 QR validation webhook (Scan → webhook → ID extraction)
Receives the scanned QR request, extracts the `id` from the URL query string.

### 1.4 Validation logic + “used” update + responses
Checks if the ID exists in the sheet, verifies it has not been used, updates status to `QR_USED`, and responds with a validation message.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Bulk QR generation orchestration

**Overview:**  
This block starts the generation process manually, reads rows from a Google Sheet, filters to only those needing QR generation, and sends them into a batch loop.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Get List
- Check is generated?
- Loop Over Items

#### Node: When clicking ‘Execute workflow’
- **Type / role:** `Manual Trigger` — starts the workflow on demand.
- **Configuration choices:** No parameters; used for manual runs/testing.
- **Inputs / outputs:**  
  - Output → **Get List**
- **Edge cases / failures:** None (only runs when manually executed).
- **Version notes:** typeVersion 1.

#### Node: Get List
- **Type / role:** `Google Sheets` — reads the spreadsheet rows to process.
- **Configuration choices (interpreted):**
  - Document and sheet are selected via UI resource locator fields (currently blank in exported JSON; must be set).
  - Operation is not explicitly shown in JSON; by default this node is used as a “read/get many rows” step in this template.
- **Key data expectations:** rows must contain at least:
  - `email` (unique identifier used to generate QR and used as validation `id`)
  - `status_qr` (controls generation/validation states)
- **Connections:**  
  - Input ← Manual Trigger  
  - Output → **Check is generated?**
- **Edge cases / failures:**
  - Missing/invalid Google credentials
  - Wrong sheet/document selection
  - Columns not present (`email`, `status_qr`) causing later expressions to fail
- **Version notes:** typeVersion 4.7.

#### Node: Check is generated?
- **Type / role:** `IF` — filters out rows already generated.
- **Configuration choices:**
  - Condition: `{{$json.status_qr}}` **notEquals** `"QR_GENERATED"`
  - If **true** → proceed to batch loop; if **false** → stop for that item.
- **Connections:**  
  - Input ← Get List  
  - Output (true) → **Loop Over Items**
- **Edge cases / failures:**
  - `status_qr` missing → comparison may behave unexpectedly (will likely pass as “not equals”)
  - Case sensitivity: enabled (strict validation mode)
- **Version notes:** typeVersion 2.2.

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — iterates over the filtered rows in controllable batches.
- **Configuration choices:** options object present but batch size not set in export (n8n defaults apply unless configured in UI).
- **Connections:**
  - Input ← Check is generated?
  - Output **index 1** → **QR Generator** (this is the “batch items” output path)
  - Receives loop-back input from **Update Row Status QR Generated** to continue processing next batch.
- **Edge cases / failures:**
  - Incorrect batch configuration can impact performance or rate limits.
  - If downstream nodes error, loop stops mid-way.
- **Version notes:** typeVersion 3.

**Sticky notes associated:**
- “Check whether a QR code has already been generated for this ID”

---

### Block 1.2 — QR creation + Drive upload + status update

**Overview:**  
For each row, this block generates a QR image that encodes a validation URL containing the unique ID, uploads the image to Google Drive, then updates the sheet row status to `QR_GENERATED`.

**Nodes involved:**
- QR Generator
- Upload file
- Prepare Status Update
- Update Row Status QR Generated
- Loop Over Items (loop-back)

#### Node: QR Generator
- **Type / role:** `HTTP Request` — calls an external QR generation service to create a QR image.
- **Configuration choices:**
  - URL template:  
    `https://api.qrserver.com/v1/create-qr-code/?size=300x300&data={{ $env.WEBHOOK_URL }}/webhook/read-qr?id={{$json.email}}`
  - Uses environment variable `$env.WEBHOOK_URL` as the base URL to your n8n instance.
  - Embeds the identifier as `id={{$json.email}}`.
- **Inputs / outputs:**
  - Input ← Loop Over Items
  - Output → Upload file
- **Edge cases / failures:**
  - `$env.WEBHOOK_URL` not set → QR encodes an invalid URL
  - `email` missing/empty → QR encodes an unusable ID
  - External API downtime / rate limiting / non-200 responses
  - Response is an image; depending on n8n defaults, you may need “Download”/binary handling (if not set, Drive upload can fail)
- **Version notes:** typeVersion 4.2.

#### Node: Upload file
- **Type / role:** `Google Drive` — uploads the generated QR code file to Drive.
- **Configuration choices:**
  - Name: `={{ $('Loop Over Items').item.json.email }}`
    - Uses the current loop item email as the file name.
  - Drive: “My Drive”
  - Folder: not selected (blank) → defaults to Drive root unless configured.
  - Upload content is expected from the previous HTTP node (binary).
- **Connections:**
  - Input ← QR Generator
  - Output → Prepare Status Update
- **Edge cases / failures:**
  - Missing Drive credentials / insufficient permissions
  - Folder not set (uploads to root unexpectedly)
  - If QR Generator doesn’t output binary data as expected, upload fails
  - File naming collisions if same email repeats
- **Version notes:** typeVersion 3.

#### Node: Prepare Status Update
- **Type / role:** `Set` — builds a minimal payload used to update Google Sheets.
- **Configuration choices:**
  - Sets:
    - `id` = `{{ $('Loop Over Items').item.json.email }}`
    - `status` = `"QR_GENERATED"`
- **Connections:**
  - Input ← Upload file
  - Output → Update Row Status QR Generated
- **Edge cases / failures:**
  - If loop item is unavailable (miswiring), expression fails.
- **Version notes:** typeVersion 3.4.

#### Node: Update Row Status QR Generated
- **Type / role:** `Google Sheets (update)` — updates the row for this ID to mark QR generated.
- **Configuration choices:**
  - Operation: `update`
  - Sheet name: `list_user` (cached name shown)
  - Document ID: must be selected (blank in JSON)
  - The mapping/key field isn’t visible in JSON export; you must configure how the update locates the row (commonly by row number or by matching a column like `email`).
- **Connections:**
  - Input ← Prepare Status Update
  - Output → Loop Over Items (loop-back to continue batches)
- **Edge cases / failures:**
  - Update mode misconfigured (e.g., no row match) → row not updated
  - Missing permissions / sheet protected
  - If the workflow relies on `row_number` later, you should ensure it exists and is preserved.
- **Version notes:** typeVersion 4.7.

**Sticky notes associated:**
- “Generate a QR code using the webhook validation URL and the unique ID”
- “Update the status of the ID in Google Sheets”

---

### Block 1.3 — QR validation webhook (scan entry)

**Overview:**  
This block receives incoming requests from scanned QR codes and extracts the `id` parameter from the query string for validation.

**Nodes involved:**
- QR Validation Webhook
- Extract ID from Webhook

#### Node: QR Validation Webhook
- **Type / role:** `Webhook` — public endpoint hit by scanning the QR code.
- **Configuration choices:**
  - Path: `read-qr`
  - Response mode: `responseNode` (responses are produced by Respond to Webhook nodes downstream)
- **Connections:**
  - Output → Extract ID from Webhook
- **Edge cases / failures:**
  - Workflow not active → production webhook won’t respond as expected
  - Missing `id` query parameter
  - Using wrong base URL in generated QR (points to dev URL or wrong instance)
- **Version notes:** typeVersion 2.

#### Node: Extract ID from Webhook
- **Type / role:** `Set` — normalizes inbound payload to a consistent field name.
- **Configuration choices:**
  - Sets `id = {{ $json.query.id }}`
- **Connections:**
  - Input ← QR Validation Webhook
  - Output → Check ID Exists
- **Edge cases / failures:**
  - If request does not include `query.id`, `id` becomes empty/undefined.
- **Version notes:** typeVersion 3.4.

**Sticky notes associated:**
- “Validate whether the scanned QR code is valid”

---

### Block 1.4 — Validation logic + update + responses

**Overview:**  
This block verifies the provided ID exists, checks whether it has already been used, updates the sheet to mark it used, and returns a text response indicating validity.

**Nodes involved:**
- Check ID Exists
- Get List ID
- Not used?
- Update QR used
- QR OK
- QR ID Already Used
- Not Valid

#### Node: Check ID Exists
- **Type / role:** `IF` — validates presence of `id` before looking it up.
- **Configuration choices:**
  - Condition: `{{$json.id}}` **exists**
  - True → Get List ID  
  - False → Not Valid (immediate response)
- **Connections:**
  - Input ← Extract ID from Webhook
  - Output (true) → Get List ID
  - Output (false) → Not Valid
- **Edge cases / failures:**
  - “exists” only checks presence, not whether the ID matches a row in Sheets.
- **Version notes:** typeVersion 2.2.

#### Node: Get List ID
- **Type / role:** `Google Sheets` — fetches row data for the given ID.
- **Configuration choices:**
  - Sheet name: `list_user`
  - Document ID must be set (blank in JSON)
  - The node name implies it retrieves a specific record, but the exported parameters don’t show the filter. In n8n, you must configure either:
    - “Lookup” by column (recommended), or
    - Read all rows then filter downstream (less efficient).
- **Connections:**
  - Input ← Check ID Exists (true)
  - Output → Not used?
- **Edge cases / failures:**
  - If lookup/filter isn’t configured, it may return all rows; downstream logic may behave incorrectly.
  - If ID not found, output could be empty (no items), preventing any response node from firing (client may hang).
- **Version notes:** typeVersion 4.5.

#### Node: Not used?
- **Type / role:** `IF` — determines whether the QR has already been used.
- **Configuration choices:**
  - Conditions (AND):
    1) `{{$json['status_qr']}}` is **empty**  
    2) `{{$json.row_number}}` **exists**
  - True branch → Update QR used  
  - False branch → QR ID Already Used
- **Interpretation:** This template treats “unused” as `status_qr` being empty (and row_number existing).
- **Connections:**
  - Input ← Get List ID
  - Output (true) → Update QR used
  - Output (false) → QR ID Already Used
- **Edge cases / failures:**
  - If your “unused” state is not actually empty (e.g., `QR_GENERATED`), this logic will mark valid QRs as “already used”. Consider changing the condition to `status_qr != 'QR_USED'`.
  - If `row_number` is not provided by the Sheets node, updates may fail or the IF may route incorrectly.
- **Version notes:** typeVersion 2.2.
- **Sticky note:** “Check if ID is valid” + “Update the QR status after it has been used”

#### Node: Update QR used
- **Type / role:** `Google Sheets (update)` — marks the QR/ID as used.
- **Configuration choices:**
  - Operation: `update`
  - Must be configured to update the specific row (commonly using `row_number` from the read step).
  - Intended new status: `"QR_USED"` (not visible in exported params; must be mapped in the node).
- **Connections:**
  - Input ← Not used? (true)
  - Output → QR OK
- **Edge cases / failures:**
  - If row targeting isn’t configured (row_number missing), update might fail or update wrong row.
  - Concurrency: two scans at the same time could both pass “not used?” before the sheet updates (race condition).
- **Version notes:** typeVersion 4.5.

#### Node: QR OK
- **Type / role:** `Respond to Webhook` — returns success response.
- **Configuration choices:**
  - Respond with: text
  - Body: `QR Valid!`
- **Connections:** Input ← Update QR used
- **Edge cases / failures:** If earlier steps return no item, this won’t run and webhook call may time out.
- **Version notes:** typeVersion 1.1.

#### Node: QR ID Already Used
- **Type / role:** `Respond to Webhook` — indicates the ID is not usable anymore.
- **Configuration choices:**
  - Response code: 200
  - Body: `QR ID Already Used`
- **Connections:** Input ← Not used? (false)
- **Edge cases / failures:** Same “no item produced” issue if Get List ID returns nothing.
- **Version notes:** typeVersion 1.1.

#### Node: Not Valid
- **Type / role:** `Respond to Webhook` — indicates missing/invalid request.
- **Configuration choices:**
  - Response code: 200
  - Body: `QR not valid`
- **Connections:** Input ← Check ID Exists (false)
- **Edge cases / failures:** Consider using HTTP 400/404 in production (optional).
- **Version notes:** typeVersion 1.1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start of bulk generation | — | Get List |  |
| Get List | Google Sheets | Read rows from sheet for QR generation | When clicking ‘Execute workflow’ | Check is generated? | Check whether a QR code has already been generated for this ID |
| Check is generated? | IF | Filter rows not yet marked QR_GENERATED | Get List | Loop Over Items | Check whether a QR code has already been generated for this ID |
| Loop Over Items | Split In Batches | Iterate over rows in batches | Check is generated?; Update Row Status QR Generated | QR Generator |  |
| QR Generator | HTTP Request | Generate QR image encoding webhook URL + ID | Loop Over Items | Upload file | Generate a QR code using the webhook validation URL and the unique ID |
| Upload file | Google Drive | Upload QR image to Drive | QR Generator | Prepare Status Update |  |
| Prepare Status Update | Set | Create payload to mark row as QR_GENERATED | Upload file | Update Row Status QR Generated | Update the status of the ID in Google Sheets |
| Update Row Status QR Generated | Google Sheets | Update sheet row status to QR_GENERATED | Prepare Status Update | Loop Over Items | Update the status of the ID in Google Sheets |
| QR Validation Webhook | Webhook | Entry point for scanned QR validation | — | Extract ID from Webhook | Validate whether the scanned QR code is valid |
| Extract ID from Webhook | Set | Extract `id` query parameter | QR Validation Webhook | Check ID Exists | Validate whether the scanned QR code is valid |
| Check ID Exists | IF | Ensure `id` is provided | Extract ID from Webhook | Get List ID; Not Valid | Check if ID is valid |
| Get List ID | Google Sheets | Retrieve row(s) for provided ID | Check ID Exists | Not used? | Check if ID is valid |
| Not used? | IF | Decide whether QR is unused vs already used | Get List ID | Update QR used; QR ID Already Used | Update the QR status after it has been used |
| Update QR used | Google Sheets | Update row status to QR_USED | Not used? | QR OK | Update the QR status after it has been used |
| QR OK | Respond to Webhook | Return “valid” response | Update QR used | — |  |
| QR ID Already Used | Respond to Webhook | Return “already used” response | Not used? | — |  |
| Not Valid | Respond to Webhook | Return “not valid” response | Check ID Exists | — |  |
| Sticky Note | Sticky Note | Comment | — | — | Validate whether the scanned QR code is valid |
| Sticky Note1 | Sticky Note | Comment | — | — | Check if ID is valid |
| Sticky Note2 | Sticky Note | Comment | — | — | Check whether a QR code has already been generated for this ID |
| Sticky Note3 | Sticky Note | Comment | — | — | Update the status of the ID in Google Sheets |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Generate and validate QR codes in bulk with Google Sheets and Google Drive… Contact: https://www.linkedin.com/in/dwicahyas/ |
| Sticky Note5 | Sticky Note | Comment | — | — | Update the QR status after it has been used |
| Sticky Note6 | Sticky Note | Comment | — | — | Generate a QR code using the webhook validation URL and the unique ID |
| Not used? (node name “Not used?”) | IF | (Included above already) | (Included above) | (Included above) | (Included above) |

> Note: Sticky Note nodes are included as nodes in the workflow export; they don’t connect to other nodes.

---

## 4. Reproducing the Workflow from Scratch

### Part A — Bulk QR generation

1. **Create node: Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

2. **Create node: Google Sheets (read)**
   - Name: `Get List`
   - Credentials: connect Google Sheets OAuth2 account.
   - Select:
     - **Document**: your spreadsheet
     - **Sheet**: the sheet containing your users (must include `email` and `status_qr`)
   - Configure to **read all rows** (or “Get Many”) depending on your node UI.

3. **Create node: IF**
   - Name: `Check is generated?`
   - Condition:
     - Value 1: `{{$json.status_qr}}`
     - Operation: `not equals`
     - Value 2: `QR_GENERATED`
   - Connect: `Get List` → `Check is generated?`

4. **Create node: Split In Batches**
   - Name: `Loop Over Items`
   - Set batch size as desired (e.g., 10–100 depending on Drive/API limits).
   - Connect: `Check is generated?` (true output) → `Loop Over Items`

5. **Create node: HTTP Request**
   - Name: `QR Generator`
   - Method: GET
   - URL:  
     `https://api.qrserver.com/v1/create-qr-code/?size=300x300&data={{ $env.WEBHOOK_URL }}/webhook/read-qr?id={{$json.email}}`
   - Important:
     - Ensure `$env.WEBHOOK_URL` is defined in your n8n environment (e.g., `https://n8n.yourdomain.com`)
     - Configure the node to **download the response as a file/binary** (so Drive upload can use it).

6. **Create node: Google Drive (upload)**
   - Name: `Upload file`
   - Credentials: connect Google Drive OAuth2.
   - Operation: Upload
   - File/Binary property: select the binary output from `QR Generator` (commonly `data`).
   - File name: `{{ $('Loop Over Items').item.json.email }}`
   - Choose Drive and target folder (recommended).
   - Connect: `QR Generator` → `Upload file`

7. **Create node: Set**
   - Name: `Prepare Status Update`
   - Add fields:
     - `id` (string): `{{ $('Loop Over Items').item.json.email }}`
     - `status` (string): `QR_GENERATED`
   - Connect: `Upload file` → `Prepare Status Update`

8. **Create node: Google Sheets (update)**
   - Name: `Update Row Status QR Generated`
   - Operation: Update
   - Select same Document + Sheet (`list_user` or your sheet name).
   - Configure row matching (choose one approach):
     - **Preferred:** “Update by matching column” (match `email` = `{{$json.id}}`) and set `status_qr` = `{{$json.status}}`
     - **Alternative:** Update by `row_number` if your read node provides it.
   - Connect: `Prepare Status Update` → `Update Row Status QR Generated`

9. **Close the loop**
   - Connect: `Update Row Status QR Generated` → `Loop Over Items` (so it continues with next batch).

---

### Part B — Validation webhook

10. **Create node: Webhook**
    - Name: `QR Validation Webhook`
    - Path: `read-qr`
    - Response mode: `Respond to Webhook node`
    - Activate workflow later to enable production URL.

11. **Create node: Set**
    - Name: `Extract ID from Webhook`
    - Add field:
      - `id` (string): `{{ $json.query.id }}`
    - Connect: `QR Validation Webhook` → `Extract ID from Webhook`

12. **Create node: IF**
    - Name: `Check ID Exists`
    - Condition: `{{$json.id}}` exists
    - Connect: `Extract ID from Webhook` → `Check ID Exists`

13. **Create node: Respond to Webhook (invalid)**
    - Name: `Not Valid`
    - Respond with: Text
    - Body: `QR not valid`
    - (Optional) Response code: 400
    - Connect: `Check ID Exists` (false output) → `Not Valid`

14. **Create node: Google Sheets (lookup/get)**
    - Name: `Get List ID`
    - Select Document + Sheet.
    - Configure a **lookup** so it returns the row matching the scanned ID:
      - Lookup column: `email`
      - Lookup value: `{{$json.id}}`
    - Connect: `Check ID Exists` (true output) → `Get List ID`

15. **Create node: IF**
    - Name: `Not used?`
    - Recreate current template logic (as exported):
      - `{{$json.status_qr}}` is empty
      - AND `{{$json.row_number}}` exists
    - Connect: `Get List ID` → `Not used?`

    > Recommended adjustment for real use: treat unused as `status_qr != 'QR_USED'` and ensure you have a reliable row reference for updating.

16. **Create node: Respond to Webhook (already used)**
    - Name: `QR ID Already Used`
    - Respond with: Text
    - Body: `QR ID Already Used`
    - Connect: `Not used?` (false output) → `QR ID Already Used`

17. **Create node: Google Sheets (update to used)**
    - Name: `Update QR used`
    - Operation: Update
    - Update the matched row to set:
      - `status_qr` = `QR_USED`
    - Configure row targeting:
      - Use `row_number` if available, or match by `email` = `{{$json.id}}`.
    - Connect: `Not used?` (true output) → `Update QR used`

18. **Create node: Respond to Webhook (ok)**
    - Name: `QR OK`
    - Respond with: Text
    - Body: `QR Valid!`
    - Connect: `Update QR used` → `QR OK`

19. **Finalize**
    - Set the environment variable `WEBHOOK_URL` to the public base URL of your n8n instance.
    - Activate the workflow.
    - Re-generate QR codes so they embed the correct production webhook URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Generate and validate QR codes in bulk with Google Sheets and Google Drive… Need Help? Contact me on LinkedIn!” | https://www.linkedin.com/in/dwicahyas/ |
| Sheet must contain `email` and `status_qr` columns | Required by expressions and IF conditions |
| Ensure `$env.WEBHOOK_URL` points to the correct public n8n base URL before generating QRs | Used in QR Generator URL |
| Validation race condition risk | Two scans in parallel can both validate before the sheet updates unless you enforce atomicity (not provided by Google Sheets) |

If you want, I can propose a safer validation condition/state model (e.g., `QR_GENERATED` → `QR_USED`) and the exact Google Sheets lookup/update configuration that avoids “empty status” ambiguity.