Bulk WhatsApp & Gmail messenger with Google Sheets status tracking via InboxPlus

https://n8nworkflows.xyz/workflows/bulk-whatsapp---gmail-messenger-with-google-sheets-status-tracking-via-inboxplus-11921


# Bulk WhatsApp & Gmail messenger with Google Sheets status tracking via InboxPlus

## 1. Workflow Overview

**Purpose:**  
This n8n workflow performs **bulk multi-channel outreach** by sending **WhatsApp template messages** and **Gmail emails** to contacts stored in **Google Sheets**, while tracking delivery status back into the same sheet. It is designed to be **safe to re-run** by only sending on channels still marked **Pending**.

**Typical use cases:** marketing outreach, customer notifications, internal announcements, follow-ups where WhatsApp and Email must be executed independently with status tracking and retry capability.

### 1.1 Input Reception & Contact Loading
- Manually starts, reads all contacts from Google Sheets, then processes contacts in batches.

### 1.2 WhatsApp Channel (status-aware)
- Validates phone number presence → checks “Message Sent” is Pending → sends WhatsApp template → checks response status → writes success/failure to the sheet.

### 1.3 Email Channel (status-aware)
- Validates email presence → checks “Mail Sent” is Pending → prepares email via InboxPlus → builds HTML → downloads an image from Google Drive → sends Gmail → checks send label → writes success/failure to the sheet.

### 1.4 Central Status Persistence & Looping
- Updates Google Sheets with the outcome and loops back to continue batch processing.

---

## 2. Block-by-Block Analysis

### Block A — Startup, Sheet Read, and Batch Control
**Overview:** Loads contacts from Google Sheets and processes them in controlled batches to reduce rate-limit risk and improve stability.

**Nodes involved:**
- Manual Trigger
- Get Contacts (Google Sheets)
- Split In Batches

#### Node: Manual Trigger
- **Type / Role:** `n8n-nodes-base.manualTrigger` — manual entry point.
- **Config:** No parameters.
- **Inputs/Outputs:** Outputs to **Get Contacts**.
- **Edge cases:** None (manual start only).
- **Version notes:** v1.

#### Node: Get Contacts
- **Type / Role:** `n8n-nodes-base.googleSheets` — reads rows from a sheet.
- **Config (interpreted):**
  - Document: Google Sheet with ID `1B-ban2DAJLUzlf85zRkpP-QrdGsoVdQoBvCNhRW7sgs`
  - Sheet/tab (gid): `372585685` (named “Bulk Messenger” in cached metadata)
  - Operation implied by missing explicit operation: **read/get many rows** (standard Google Sheets node default behavior in many exports, but confirm in UI).
- **Outputs:** Feeds rows into **Split In Batches**.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - OAuth expiry / insufficient permissions
  - Sheet/tab ID mismatch (gid changed)
  - Missing expected columns: `Phone Number`, `Email`, `Message Sent`, `Mail Sent`, `Name`
- **Version notes:** v4.

#### Node: Split In Batches
- **Type / Role:** `n8n-nodes-base.splitInBatches` — throttles/controls throughput.
- **Config:**
  - `batchSize: 3`
- **Connections:**
  - Receives from **Get Contacts**
  - Outputs simultaneously to **Has Phone Number** and **Has Email Address**
  - Receives loop-back from **Update Sheet** to continue next batch
- **Edge cases:**
  - If downstream errors stop loop-back, the workflow may stop early
  - Large datasets: execution time & memory constraints
- **Version notes:** v1.

---

### Block B — WhatsApp Send Path (Pending-only)
**Overview:** Sends a WhatsApp template message only if the row has a phone number and WhatsApp status is Pending, then updates the sheet based on send acceptance.

**Nodes involved:**
- Has Phone Number (IF)
- IF WhatsApp Pending (IF)
- Send template (WhatsApp)
- Sent (IF)
- Update Sheet (Google Sheets)
- Whatsapp Failure (Google Sheets)

#### Node: Has Phone Number
- **Type / Role:** `n8n-nodes-base.if` — data validation gate.
- **Condition:** `Phone Number` **not empty**  
  Expression: `={{ $json['Phone Number'] }}`
- **Routing:**
  - **True** → IF WhatsApp Pending
  - **False** → no connection (WhatsApp path ends silently)
- **Edge cases:**
  - Phone numbers stored as text with spaces/“+”: condition uses “notEmpty” so formatting won’t block, but invalid phone format will fail later in WhatsApp API.
- **Version notes:** v2.2.

#### Node: IF WhatsApp Pending
- **Type / Role:** `n8n-nodes-base.if` — idempotency control.
- **Condition:** `Message Sent` equals `"Pending"`  
  Expression: `={{ $json['Message Sent'] }}`
- **Routing:**
  - **True** → Send template
  - **False** → no connection (prevents duplicates)
- **Edge cases:**
  - Case sensitivity: `"pending"` vs `"Pending"` will not match
  - Empty/undefined column: won’t pass
- **Version notes:** v2.2.

#### Node: Send template
- **Type / Role:** `n8n-nodes-base.whatsApp` — sends WhatsApp Cloud API template.
- **Config:**
  - Template: `hello_world|en_US`
  - Phone Number ID: `986729097850591`
  - Recipient: `={{ String($json['Phone Number']) }}`
- **Output:** Sends response to **Sent** IF node.
- **Credentials:** WhatsApp API credentials (Cloud API).
- **Edge cases / failures:**
  - Invalid recipient format (missing country code, spaces)
  - Template not approved / wrong language code
  - Rate limiting (Meta)
  - Auth/permission issues on phoneNumberId
- **Version notes:** v1.1.

#### Node: Sent
- **Type / Role:** `n8n-nodes-base.if` — checks WhatsApp API response status.
- **Condition:** `{{ $json.messages[0].message_status }}` equals `"accepted"`
- **Routing:**
  - **True** → Update Sheet
  - **False** → Whatsapp Failure
- **Edge cases:**
  - Response shape changes: if `messages[0]` is missing, expression may error or evaluate undefined (n8n IF usually treats as not matching, but can fail depending on runtime)
  - “accepted” means accepted by API, not necessarily delivered/read
- **Version notes:** v2.2.

#### Node: Whatsapp Failure
- **Type / Role:** `n8n-nodes-base.googleSheets` — writes WhatsApp failure status.
- **Operation:** Update row by matching `Phone Number`.
- **Values written:**
  - `Message Sent` = `"Failed"`
  - `Phone Number` = `={{$json['Phone Number']}}`
- **Matching:** `matchingColumns: ["Phone Number"]`
- **Edge cases:**
  - If phone number in response item is not the original sheet item, match may fail (here it uses current `$json`, which at this point is WhatsApp response, not guaranteed to contain `Phone Number`)
  - Multiple identical phone numbers → ambiguous update
- **Version notes:** v4.

#### Node: Update Sheet
- **Type / Role:** `n8n-nodes-base.googleSheets` — writes success status (both channels set to Sent).
- **Operation:** Update by `Phone Number`.
- **Values written:**
  - `Mail Sent` = `"Sent"`
  - `Message Sent` = `"Sent"`
  - `Phone Number` = `={{ $('Get Contacts').item.json['Phone Number'] }}`
- **Connections:**
  - Receives from **Sent (true)** and also from **Delivered (true)**
  - Outputs back to **Split In Batches** (loop continuation)
- **Important behavior note (logic coupling):**
  - This node sets **both** `Mail Sent` and `Message Sent` to `"Sent"` regardless of which channel succeeded. If only WhatsApp succeeded, it will still mark Mail Sent as Sent, and vice-versa.
- **Edge cases:**
  - Same as other sheet updates (matching ambiguity, permission, schema mismatch)
- **Version notes:** v4.

---

### Block C — Email Send Path (Pending-only with InboxPlus + Gmail)
**Overview:** Sends an email only if an email address exists and Mail Sent is Pending. Uses InboxPlus node (template-driven) plus a custom HTML body and an image download, then verifies Gmail result.

**Nodes involved:**
- Has Email Address (IF)
- IF Mail Pending (IF)
- PrepareEmail email (InboxPlus)
- Build HTML Email (Set)
- Fetch Email Image (Google Drive)
- Send Gmail (Gmail)
- Delivered (IF)
- Update Sheet (Google Sheets)
- Mail Failure (Google Sheets)

#### Node: Has Email Address
- **Type / Role:** `n8n-nodes-base.if` — validation gate.
- **Condition:** checks not empty using:  
  `={{ $('Split In Batches').item.json.Email }}`
- **Routing:**
  - **True** → IF Mail Pending
  - **False** → ends email path silently
- **Edge cases:**
  - Using `$('Split In Batches').item.json.Email` instead of `$json.Email` can break if item linking differs or if paired-item context is lost
- **Version notes:** v2.2.

#### Node: IF Mail Pending
- **Type / Role:** `n8n-nodes-base.if` — idempotency control.
- **Condition:** `Mail Sent` equals `"Pending"` via `={{ $json['Mail Sent'] }}`
- **Routing:** **True** → PrepareEmail email
- **Edge cases:** Same as WhatsApp pending check (case sensitivity, missing column).
- **Version notes:** v2.2.

#### Node: PrepareEmail email
- **Type / Role:** `@itechnotion/n8n-nodes-inboxplus.inboxPlus` — prepares email content based on an InboxPlus template.
- **Config:**
  - Template ID: `111ce91e-b0c2-4513-8cfa-845979431223`
  - Recipient email: `={{ $json.Email }}`
- **Output:** To **Build HTML Email**.
- **Credentials:** InboxPlus API credential.
- **Edge cases / failures:**
  - TemplateId invalid/deleted
  - API auth failure / quota
  - Output fields may not match downstream expectations (e.g., subject)
- **Downstream dependency:** **Send Gmail** expects `$('PrepareEmail email').item.json.subject`.
- **Version notes:** v1 (community/custom node).

#### Node: Build HTML Email
- **Type / Role:** `n8n-nodes-base.set` — constructs `gmailBodyHtml`.
- **Config:**
  - Adds field `gmailBodyHtml` containing a full HTML document
  - Includes personalization: `Hi {{ $('Get Contacts').item.json.Name }},`
  - Includes image URL: `https://drive.google.com/uc?id=1tc--ftXJE9dCvfq0yW3lvGvbOEnZviqP`
- **Output:** To **Fetch Email Image**.
- **Edge cases:**
  - Uses `$('Get Contacts').item.json.Name` which may not resolve correctly per-item (should usually reference the current item, e.g., `$json.Name`, to avoid cross-item mismatches)
  - Gmail typically accepts HTML, but full `<html><head>...` wrappers are optional; some clients may alter rendering
- **Version notes:** v3.4.

#### Node: Fetch Email Image
- **Type / Role:** `n8n-nodes-base.googleDrive` — downloads an image file intended for email attachment/inline usage.
- **Config:**
  - Operation: download
  - File ID: `1tc--ftXJE9dCvfq0yW3lvGvbOEnZviqP` (named “images (2).jpeg” in cached metadata)
- **Output:** To **Send Gmail**
- **Credentials:** Google Drive OAuth2.
- **Edge cases:**
  - File permissions (must be accessible to the credential)
  - Large file size → execution memory
  - MIME type/filename issues for email attachment
- **Version notes:** v3.

#### Node: Send Gmail
- **Type / Role:** `n8n-nodes-base.gmail` — sends email via Gmail.
- **Error handling:** `onError: continueRegularOutput` (workflow continues even if Gmail send fails; downstream logic must detect failure).
- **Config:**
  - To: `={{ $('Get Contacts').item.json.Email }}`
  - Subject: `={{ $('PrepareEmail email').item.json.subject }}`
  - Message body: `={{ $json.gmailBodyHtml }}`
  - Attachments UI: `attachmentsBinary` present but not configured with a specific binary property name (likely incomplete).
  - Append attribution: false
- **Output:** To **Delivered**
- **Credentials:** Gmail OAuth2.
- **Edge cases / failures:**
  - If `subject` missing from InboxPlus output → empty subject or expression failure
  - Attachment not actually attached (binary mapping not set)
  - “continueRegularOutput” can mask failures unless you check response content
- **Version notes:** v2.

#### Node: Delivered
- **Type / Role:** `n8n-nodes-base.if` — checks Gmail API result.
- **Condition:** `labelIds` array contains `"SENT"`  
  Expression: `={{ $json.labelIds }} contains SENT`
- **Routing:**
  - **True** → Update Sheet
  - **False** → Mail Failure
- **Edge cases:**
  - Gmail send response may not include `labelIds` in the expected way depending on node operation/version
  - If Gmail node errors but continues output, the structure may not include `labelIds` → false branch and sheet marked failed (desired, but depends)
- **Version notes:** v2.2.

#### Node: Mail Failure
- **Type / Role:** `n8n-nodes-base.googleSheets` — writes failure status (but currently writes to “Message Sent”).
- **Operation:** Update row by matching `Phone Number`.
- **Values written (as configured):**
  - `Message Sent` = `"Failed"`
  - `Phone Number` = `={{$json['Phone Number']}}`
- **Important misconfiguration:**
  - For email failures, this likely should update `Mail Sent` instead of `Message Sent`.
  - Also it matches by `Phone Number`, but at this stage `$json` is Gmail response, which likely does **not** contain `Phone Number`.
- **Edge cases:** High chance of “no row updated”.
- **Version notes:** v4.

---

### Block D — Status Writeback and Re-run Safety
**Overview:** Ensures each contact row acts as the source of truth. When statuses are updated correctly, re-running will skip non-pending channels.

**Nodes involved:**
- Update Sheet
- Whatsapp Failure
- Mail Failure

**Key edge case (design):**  
Because **Update Sheet** sets both statuses to “Sent”, the workflow’s “safe re-run” logic becomes channel-coupled; a success in one channel may prevent future sending on the other.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual start entry point | — | Get Contacts | ## Bulk WhatsApp + Gmail Sender… (workflow description + setup steps) |
| Get Contacts | Google Sheets | Read contact rows | Manual Trigger | Split In Batches | ## Step 1: Fetch contacts & batch processing… |
| Split In Batches | Split In Batches | Process contacts in batches (size 3) | Get Contacts, Update Sheet | Has Phone Number; Has Email Address | ## Step 1: Fetch contacts & batch processing… |
| Has Phone Number | IF | Validate phone exists | Split In Batches | IF WhatsApp Pending (true) | ## Step 3: WhatsApp message sending… |
| IF WhatsApp Pending | IF | WhatsApp idempotency gate (“Pending”) | Has Phone Number | Send template (true) | ## Step 3: WhatsApp message sending… |
| Send template | WhatsApp | Send WhatsApp template message | IF WhatsApp Pending | Sent | ## Step 3: WhatsApp message sending… |
| Sent | IF | Check WhatsApp API acceptance | Send template | Update Sheet (true); Whatsapp Failure (false) | ## Step 3: WhatsApp message sending… |
| Whatsapp Failure | Google Sheets | Write WhatsApp failure back to sheet | Sent (false) | — | ## Step 4: Delivery status updates… |
| Has Email Address | IF | Validate email exists | Split In Batches | IF Mail Pending (true) | ## Step 2:  Email preparation & sending… |
| IF Mail Pending | IF | Email idempotency gate (“Pending”) | Has Email Address | PrepareEmail email (true) | ## Step 2:  Email preparation & sending… |
| PrepareEmail email | InboxPlus | Fetch/prepare email subject/template | IF Mail Pending | Build HTML Email | ## Step 2:  Email preparation & sending… |
| Build HTML Email | Set | Build HTML body into `gmailBodyHtml` | PrepareEmail email | Fetch Email Image | ## Step 2:  Email preparation & sending… |
| Fetch Email Image | Google Drive | Download image for email (binary) | Build HTML Email | Send Gmail | ## Step 2:  Email preparation & sending… |
| Send Gmail | Gmail | Send HTML email via Gmail | Fetch Email Image | Delivered | ## Step 2:  Email preparation & sending… |
| Delivered | IF | Check Gmail result includes SENT label | Send Gmail | Update Sheet (true); Mail Failure (false) | ## Step 2:  Email preparation & sending… |
| Mail Failure | Google Sheets | Write email failure status (misconfigured) | Delivered (false) | — | ## Step 4: Delivery status updates… |
| Update Sheet | Google Sheets | Write “Sent” statuses and continue batches | Sent (true), Delivered (true) | Split In Batches | ## Step 4: Delivery status updates… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Manual Trigger**
   - Node type: *Manual Trigger*
   - Connect to next node.

3. **Add node: Google Sheets → “Get Contacts”**
   - Credentials: **Google Sheets OAuth2**
   - Select Spreadsheet: `InfographAI` (the sheet with ID `1B-ban2DAJLUzlf85zRkpP-QrdGsoVdQoBvCNhRW7sgs`)
   - Select Sheet/Tab: `Bulk Messenger` (gid `372585685`)
   - Operation: read rows (e.g., “Get Many” / “Read” depending on node UI)
   - Ensure columns exist: `Name`, `Email`, `Phone Number`, `Message Sent`, `Mail Sent`
   - Connect from **Manual Trigger** to **Get Contacts**.

4. **Add node: Split In Batches**
   - Batch size: `3`
   - Connect **Get Contacts → Split In Batches**
   - This node will also receive a loop-back later from **Update Sheet**.

---

### WhatsApp branch

5. **Add node: IF → “Has Phone Number”**
   - Condition: `Phone Number` → “not empty”
   - Expression: `{{$json['Phone Number']}}`
   - Connect **Split In Batches → Has Phone Number**

6. **Add node: IF → “IF WhatsApp Pending”**
   - Condition: `Message Sent` equals `Pending`
   - Expression: `{{$json['Message Sent']}}`
   - Connect **Has Phone Number (true) → IF WhatsApp Pending**

7. **Add node: WhatsApp → “Send template”**
   - Credentials: WhatsApp Cloud API
   - Phone Number ID: `986729097850591`
   - Template: `hello_world` language `en_US` (as `hello_world|en_US`)
   - Recipient phone number: `{{ String($json['Phone Number']) }}`
   - Connect **IF WhatsApp Pending (true) → Send template**

8. **Add node: IF → “Sent”**
   - Condition: `{{$json.messages[0].message_status}}` equals `accepted`
   - Connect **Send template → Sent**

9. **Add node: Google Sheets → “Whatsapp Failure”**
   - Operation: **Update**
   - Match column: `Phone Number`
   - Set: `Message Sent = Failed`
   - (Recommended when recreating: reference the original item’s phone number reliably, e.g. from Split In Batches item)
   - Connect **Sent (false) → Whatsapp Failure**

---

### Email branch

10. **Add node: IF → “Has Email Address”**
   - Condition: Email “not empty”
   - Prefer using: `{{$json.Email}}` (the provided workflow used `$('Split In Batches').item.json.Email`)
   - Connect **Split In Batches → Has Email Address**

11. **Add node: IF → “IF Mail Pending”**
   - Condition: `Mail Sent` equals `Pending`
   - Expression: `{{$json['Mail Sent']}}`
   - Connect **Has Email Address (true) → IF Mail Pending**

12. **Add node: InboxPlus → “PrepareEmail email”** (community node `@itechnotion/n8n-nodes-inboxplus`)
   - Credentials: InboxPlus API
   - Template ID: `111ce91e-b0c2-4513-8cfa-845979431223`
   - Recipient: `{{$json.Email}}`
   - Connect **IF Mail Pending (true) → PrepareEmail email**

13. **Add node: Set → “Build HTML Email”**
   - Add field `gmailBodyHtml` (string) containing your HTML template
   - Include personalization (ideally `{{$json.Name}}` from the current row)
   - Connect **PrepareEmail email → Build HTML Email**

14. **Add node: Google Drive → “Fetch Email Image”**
   - Credentials: Google Drive OAuth2
   - Operation: **Download**
   - File ID: `1tc--ftXJE9dCvfq0yW3lvGvbOEnZviqP`
   - Connect **Build HTML Email → Fetch Email Image**

15. **Add node: Gmail → “Send Gmail”**
   - Credentials: Gmail OAuth2
   - Operation: Send
   - To: contact email (e.g., `{{$json.Email}}` from the sheet item)
   - Subject: from InboxPlus output: `{{ $('PrepareEmail email').item.json.subject }}`
   - Message: `{{$json.gmailBodyHtml}}`
   - (Optional) Attachments: map the binary property produced by Google Drive download
   - Set node error handling to **Continue on Fail** (equivalent to `onError: continueRegularOutput`)
   - Connect **Fetch Email Image → Send Gmail**

16. **Add node: IF → “Delivered”**
   - Condition: `labelIds` contains `SENT`
   - Connect **Send Gmail → Delivered**

17. **Add node: Google Sheets → “Mail Failure”**
   - Operation: **Update**
   - Match column: `Phone Number` (or preferably a stable row identifier/email)
   - Should set: `Mail Sent = Failed` (the provided workflow sets `Message Sent`, which is likely wrong)
   - Connect **Delivered (false) → Mail Failure**

---

### Shared status update + looping

18. **Add node: Google Sheets → “Update Sheet”**
   - Operation: **Update**
   - Match column: `Phone Number`
   - Set statuses (as in the provided workflow):
     - `Mail Sent = Sent`
     - `Message Sent = Sent`
   - Connect **Sent (true) → Update Sheet**
   - Connect **Delivered (true) → Update Sheet**

19. **Loop continuation**
   - Connect **Update Sheet → Split In Batches** to process the next batch.

20. **Credentials checklist**
   - Google Sheets OAuth2: read + update access
   - WhatsApp Cloud API: phone number ID, template approval
   - InboxPlus API key/credential: template exists and returns subject (and any other used fields)
   - Google Drive OAuth2: permission to download the specified file
   - Gmail OAuth2: permission to send email

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Bulk WhatsApp + Gmail Sender… safe to re-run… Emails support three image delivery modes… batch processing… updates written back to Google Sheets.” | Sticky note describing the overall workflow intent and setup prerequisites |
| “Step 1: Fetch contacts & batch processing” | Sticky note describing the batching/rate-limit control block |
| “Step 2: Email preparation & sending” | Sticky note describing the InboxPlus + HTML + Drive + Gmail block |
| “Step 3: WhatsApp message sending” | Sticky note describing WhatsApp pending-check + template send + status tracking |
| “Step 4: Delivery status updates” | Sticky note describing writing results back to Sheets |

**Notable implementation warnings (from the current configuration):**
- **Update Sheet marks both channels as Sent** even if only one channel succeeded.
- **Mail Failure writes `Message Sent = Failed`** (likely should be `Mail Sent`), and it matches by `Phone Number` using `$json` from Gmail response (likely missing that field), so updates may fail silently.