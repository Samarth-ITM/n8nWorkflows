Handle approval requests with SendGrid email and Baserow records

https://n8nworkflows.xyz/workflows/handle-approval-requests-with-sendgrid-email-and-baserow-records-12972


# Handle approval requests with SendGrid email and Baserow records

## 1. Workflow Overview

**Workflow name:** Approval Workflow Handler – SendGrid & Baserow  
**Purpose:** Orchestrates an end-to-end approval lifecycle: validate incoming request data, notify an approver by email (SendGrid) with an approval link, wait for the approver’s response, then update a Baserow record and notify stakeholders of the final decision.

**Target use cases:**
- Purchase/expense approvals (amount + department-based routing)
- HR/Finance internal approval gates
- Any process where a human decision is required via an emailed link

### 1.1 Trigger & Validation
Manual start, basic input validation, invalid submissions are rejected early with an alert email.

### 1.2 Notify & Wait
Determine approver, email them a link, create a **Pending** record in Baserow, then suspend execution until the approval link is called.

### 1.3 Post-Processing & Storage
On resume, interpret the approver response, update Baserow status to **Approved**/**Rejected**, email stakeholders, then merge end paths and finalize.

---

## 2. Block-by-Block Analysis

### Block A — Trigger & Validation
**Overview:** Starts the workflow and ensures required request fields are present. Invalid requests are notified and routed to a clean end path.  
**Nodes involved:** Start Workflow → Validate Request Data → Is Request Valid? → (invalid path) Send Invalid Input Alert → Join End Paths

#### Node: **Start Workflow**
- **Type / role:** Manual Trigger (entry point for testing / manual execution)
- **Configuration:** No parameters.
- **Inputs / outputs:**  
  - **Output →** Validate Request Data
- **Edge cases:** None (manual trigger always fires when run).
- **Version:** 1

#### Node: **Validate Request Data**
- **Type / role:** Code node (data validation + normalization)
- **Configuration choices (interpreted):**
  - Reads the incoming item (`$json`)
  - Checks required fields: `requestId`, `requesterName`, `department`, `amount`
  - Adds:
    - `invalid` (boolean)
    - `timestamp` (ISO string)
- **Key variables/logic:**
  - `invalid = true` if any required field is missing/empty
- **Inputs / outputs:**
  - **Input:** From Start Workflow
  - **Output →** Is Request Valid?
- **Edge cases / failures:**
  - If `amount` is `0`, this code treats it as invalid (`!item[field]` is true for `0`). If zero is valid, adjust the check (e.g., `item[field] === undefined || item[field] === null || item[field] === ''`).
- **Version:** 2

#### Node: **Is Request Valid?**
- **Type / role:** IF node (routing)
- **Configuration choices (interpreted):**
  - Condition: `{{ $json.invalid }}` **equals** `true`
  - **True branch** = invalid input
  - **False branch** = valid input
- **Inputs / outputs:**
  - **Input:** From Validate Request Data
  - **True →** Send Invalid Input Alert
  - **False →** Determine Approver
- **Edge cases:**
  - If `invalid` is missing (upstream changed), expression may evaluate to `false`/null and incorrectly route.
- **Version:** 2

#### Node: **Send Invalid Input Alert**
- **Type / role:** SendGrid (email notification)
- **Configuration choices (interpreted):**
  - Node is present but **parameters are empty** in the JSON, so it will not send meaningful emails until configured.
- **Inputs / outputs:**
  - **Input:** From Is Request Valid? (true/invalid branch)
  - **Output →** Join End Paths (input2)
- **Common required setup (must be added):**
  - SendGrid credentials (API key)
  - From address (verified in SendGrid)
  - To address (likely requester or admin)
  - Subject/body including validation failure details
- **Edge cases / failures:**
  - Missing/invalid SendGrid credentials
  - Unverified sender identity
  - Rate limiting or SendGrid API errors
- **Version:** 1

---

### Block B — Notify Approver, Create Record, and Wait
**Overview:** For valid requests, chooses an approver email, notifies them, records a “Pending” entry in Baserow, then waits for the approver decision via the Wait node resume webhook.  
**Nodes involved:** Determine Approver → Notify Approver → Create Pending Record → Wait for Approval Response → Extract Approval Result

#### Node: **Determine Approver**
- **Type / role:** Code node (routing logic)
- **Configuration choices (interpreted):**
  - Sets `approverEmail` based on `$json.department`
  - Currently uses placeholder `user@example.com` for all departments
- **Key variables/logic:**
  - `dept === 'Finance'` → approverEmail set
  - `dept === 'HR'` → approverEmail set
- **Inputs / outputs:**
  - **Input:** From Is Request Valid? (false/valid path)
  - **Output →** Notify Approver
- **Edge cases / failures:**
  - Department values are case-sensitive; “finance” won’t match “Finance”
  - No default mapping beyond placeholder (should be expanded)
- **Version:** 2

#### Node: **Notify Approver**
- **Type / role:** SendGrid (approval request email)
- **Configuration choices (interpreted):**
  - Parameters empty in JSON; must be configured to send:
    - To: `{{$json.approverEmail}}`
    - Subject/body containing request details
    - **Approval link** pointing to the Wait node resume URL
- **Inputs / outputs:**
  - **Input:** From Determine Approver
  - **Output →** Create Pending Record
- **Critical implementation note (approval link):**
  - The Wait node provides a **resume webhook URL** (in the node UI when executed/active). The email should include that URL with a payload mechanism (depending on your pattern: query params or POST JSON).
- **Edge cases / failures:**
  - Same as other SendGrid node + broken/missing resume link
- **Version:** 1

#### Node: **Create Pending Record**
- **Type / role:** Baserow node (create database record)
- **Configuration choices (interpreted):**
  - Operation: **create**
  - Uses placeholders:
    - `databaseId: {{YOUR_DATABASE_ID}}`
    - `tableId: {{YOUR_TABLE_ID}}`
  - No field mapping is shown in JSON; you must map fields (Request ID, Requester, Department, Amount, Status, SubmittedAt, etc.).
- **Inputs / outputs:**
  - **Input:** From Notify Approver
  - **Output →** Wait for Approval Response
- **Edge cases / failures:**
  - Invalid Baserow credentials / token
  - Wrong databaseId/tableId
  - Field names mismatch (Baserow column names must match what you send)
- **Version:** 1

#### Node: **Wait for Approval Response**
- **Type / role:** Wait node (pauses execution until resumed)
- **Configuration choices (interpreted):**
  - `resume: true` meaning it expects to be resumed via its webhook/resume URL
  - Has a stored `webhookId` (internal identifier)
- **Inputs / outputs:**
  - **Input:** From Create Pending Record
  - **Output →** Extract Approval Result
- **Edge cases / failures:**
  - If the workflow is not active/published appropriately, resume links may not work as expected
  - Resume called multiple times (duplicate approvals) can create inconsistent outcomes unless protected
  - Long-running waits: retention/timeout depends on n8n configuration (executions pruning)
- **Version:** 1

#### Node: **Extract Approval Result**
- **Type / role:** Set node (shape/normalize resumed payload)
- **Configuration choices (interpreted):**
  - Node currently has no explicit field mappings (parameters empty besides options).
  - Intended to extract fields like `approved`, `comments`, and set decision timestamps.
- **Inputs / outputs:**
  - **Input:** From Wait for Approval Response (resume payload)
  - **Output →** Approved?
- **Edge cases / failures:**
  - If the resume payload doesn’t contain `approved`, the next IF may route incorrectly
- **Version:** 3.4

---

### Block C — Decision Branching, Updates, Notifications, Merge, Finalize
**Overview:** Determines whether the request was approved, updates the Baserow record accordingly, notifies the requester, merges all terminal branches, and finalizes execution.  
**Nodes involved:** Approved? → (approved path) Update Record – Approved → Notify Requester – Approved → Join End Paths → Finalize  
…and (rejected path) Update Record – Rejected → Notify Requester – Rejected → Join End Paths → Finalize  
…and (invalid path) Send Invalid Input Alert → Join End Paths → Finalize

#### Node: **Approved?**
- **Type / role:** IF node (decision routing)
- **Configuration choices (interpreted):**
  - Condition: `{{ $json.approved }}` equals `true`
- **Inputs / outputs:**
  - **Input:** From Extract Approval Result
  - **True →** Update Record – Approved
  - **False →** Update Record – Rejected
- **Edge cases:**
  - If approved value is a string `"true"` instead of boolean `true`, condition fails unless normalized
- **Version:** 2

#### Node: **Update Record – Approved**
- **Type / role:** Baserow node (update record status)
- **Configuration choices (interpreted):**
  - Operation: **update**
  - `rowId: {{ $json.requestId }}`
  - Uses placeholders `databaseId` / `tableId`
- **Inputs / outputs:**
  - **Input:** From Approved? (true)
  - **Output →** Notify Requester – Approved
- **Critical data-model note (likely issue):**
  - Baserow `rowId` is typically the **internal row numeric ID**, not your business `requestId`.  
  - Since you **create** the record earlier, you should capture the returned Baserow row id (e.g., `baserowRowId`) and use that for updates.
- **Edge cases / failures:**
  - Update fails if `rowId` does not exist or is not a valid Baserow row id
- **Version:** 1

#### Node: **Notify Requester – Approved**
- **Type / role:** SendGrid (approval confirmation)
- **Configuration choices (interpreted):**
  - Parameters empty; must be configured (To: requester, include status).
- **Inputs / outputs:**
  - **Input:** From Update Record – Approved
  - **Output →** Join End Paths (input1)
- **Edge cases:** SendGrid auth/sender verification issues.
- **Version:** 1

#### Node: **Update Record – Rejected**
- **Type / role:** Baserow node (update record status)
- **Configuration choices:** Same structure as “Approved”, but for rejection path.
- **Inputs / outputs:**
  - **Input:** From Approved? (false)
  - **Output →** Notify Requester – Rejected
- **Same critical note:** `rowId` should likely be Baserow row id, not `requestId`.
- **Version:** 1

#### Node: **Notify Requester – Rejected**
- **Type / role:** SendGrid (rejection confirmation)
- **Configuration choices:** Parameters empty; must be configured.
- **Inputs / outputs:**
  - **Input:** From Update Record – Rejected
  - **Output →** Join End Paths (input1)
- **Version:** 1

#### Node: **Join End Paths**
- **Type / role:** Merge node (converge branches)
- **Configuration choices (interpreted):**
  - Mode: **mergeInput1And2**
  - Input1 is used by approved/rejected notification paths
  - Input2 is used by invalid-input alert path
- **Inputs / outputs:**
  - **Input1:** from Notify Requester – Approved / Notify Requester – Rejected
  - **Input2:** from Send Invalid Input Alert
  - **Output →** Finalize
- **Edge cases:**
  - Merge behavior depends on arrival of items; with some configurations, it may wait for both inputs. Here it’s set to “mergeInput1And2”, which can be sensitive to missing second input depending on n8n merge semantics and number of items.
- **Version:** 2.1

#### Node: **Finalize**
- **Type / role:** Code node (final housekeeping)
- **Configuration choices:**
  - Returns items unchanged; placeholder for audit logs/metrics.
- **Inputs / outputs:**
  - **Input:** From Join End Paths
  - **Output:** End of workflow
- **Edge cases:** None (unless later expanded).
- **Version:** 2

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📝 Workflow Overview | Sticky Note | Documentation / overview |  |  | ## How it works\nThis workflow orchestrates a complete approval lifecycle from submission to resolution. A manual trigger kicks things off, after which incoming request data is verified for completeness.  Invalid submissions immediately notify the requester and terminate gracefully.  Valid requests are routed to the correct approver, logged as **Pending** in Baserow, and the approver is emailed via SendGrid with a unique approval-link.  The workflow then suspends on a *Wait* node until that link is called, resuming with the approver’s decision.  Depending on the response, the record is updated to **Approved** or **Rejected**, and all stakeholders receive a status email.  Finally, execution metadata is merged and the run is cleanly closed.\n\n## Setup steps\n1. Create a Baserow database/table with fields: Request ID, Requester, Department, Amount, Status, SubmittedAt, DecisionAt.\n2. Add SendGrid credentials in n8n and verify sender addresses.\n3. Replace all placeholder IDs (databaseId, tableId) and email addresses inside the nodes.\n4. Publish the workflow, set it *active*, and manually trigger once to generate sample data.\n5. Provide real request data to the Manual Trigger (or call via REST, e.g., the *Execute Workflow* API).\n6. Share the generated Approval Link with approvers—clicking it will resume the workflow.\n7. Monitor Baserow for live status or extend with additional analytics as needed. |
| Section • Trigger & Validation | Sticky Note | Section label |  |  | ## Trigger & Validation\nThis group covers the initial kick-off and data hygiene.  The Manual Trigger acts as the entry point—ideal for testing or external API calls via the *Execute Workflow* endpoint.  Immediately after triggering, the **Validate Request Data** Code node inspects the payload for required properties (requestId, requesterName, department, amount).  An IF node branches the flow: valid data proceeds, while invalid data triggers a SendGrid alert and gracefully ends.  Feel free to add more sophisticated validation (regex, schema checks) or transform raw inputs in the Set node if your upstream system sends fields in different shapes. |
| Section • Notify & Wait | Sticky Note | Section label |  |  | ## Notification & Await Response\nOnce a request is verified, we calculate the correct approver (e.g., based on department rules) and send a nicely formatted SendGrid email containing request details and a unique approval link.  The **Wait for Approval Response** node suspends the workflow execution, freeing resources until an approver clicks their link.  That link points to the automatically generated resume-URL of the Wait node, sending back JSON like `{ approved: true, comments: \"Looks good\" }`.  When the workflow resumes, the data is cleaned up and routed to the proper branch. |
| Section • Post-Processing | Sticky Note | Section label |  |  | ## Post-Processing & Storage\nAfter resumption, an IF node evaluates the `approved` flag.  Approved requests update the Baserow row to **Approved**, rejected ones to **Rejected**.  Each path sends a follow-up email via SendGrid to inform both requester and approver of the final status.  A Merge node converges all terminal branches for unified logging or further actions (analytics, archival, etc.).  You can expand this section with Slack, Teams, or CRM hooks as business needs evolve. |
| Start Workflow | Manual Trigger | Entry point | — | Validate Request Data |  |
| Validate Request Data | Code | Validate/normalize payload | Start Workflow | Is Request Valid? |  |
| Is Request Valid? | IF | Route valid vs invalid | Validate Request Data | Determine Approver; Send Invalid Input Alert |  |
| Send Invalid Input Alert | SendGrid | Email on invalid submission | Is Request Valid? | Join End Paths |  |
| Determine Approver | Code | Select approver email | Is Request Valid? | Notify Approver |  |
| Notify Approver | SendGrid | Send approval request + link | Determine Approver | Create Pending Record |  |
| Create Pending Record | Baserow | Create “Pending” row | Notify Approver | Wait for Approval Response |  |
| Wait for Approval Response | Wait | Pause until resume webhook called | Create Pending Record | Extract Approval Result |  |
| Extract Approval Result | Set | Normalize approval response | Wait for Approval Response | Approved? |  |
| Approved? | IF | Route approved vs rejected | Extract Approval Result | Update Record – Approved; Update Record – Rejected |  |
| Update Record – Approved | Baserow | Update row status approved | Approved? | Notify Requester – Approved |  |
| Notify Requester – Approved | SendGrid | Email approval outcome | Update Record – Approved | Join End Paths |  |
| Update Record – Rejected | Baserow | Update row status rejected | Approved? | Notify Requester – Rejected |  |
| Notify Requester – Rejected | SendGrid | Email rejection outcome | Update Record – Rejected | Join End Paths |  |
| Join End Paths | Merge | Converge terminal branches | Notify Requester – Approved; Notify Requester – Rejected; Send Invalid Input Alert | Finalize |  |
| Finalize | Code | Final pass-through / housekeeping | Join End Paths | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Approval Workflow Handler – SendGrid & Baserow*.
2. **Add “Manual Trigger”** node named **Start Workflow**.
3. **Add “Code”** node named **Validate Request Data** and connect:  
   **Start Workflow → Validate Request Data**  
   Paste logic that:
   - checks required fields: `requestId`, `requesterName`, `department`, `amount`
   - outputs `invalid` boolean and `timestamp`
   - (recommended) handle `amount = 0` correctly if needed
4. **Add “IF”** node named **Is Request Valid?** and connect:  
   **Validate Request Data → Is Request Valid?**  
   Condition:
   - Boolean: `={{ $json.invalid }}` equals `true`
   - True path = invalid
5. **Invalid path:** Add **SendGrid** node **Send Invalid Input Alert** and connect:  
   **Is Request Valid? (true) → Send Invalid Input Alert**  
   Configure SendGrid:
   - Credentials: SendGrid API key in n8n credentials
   - From: verified sender
   - To: requester or admin (e.g. `={{$json.requesterEmail}}` if you have it)
   - Subject/body: explain missing fields and include `requestId` + timestamp
6. **Valid path:** Add **Code** node **Determine Approver** and connect:  
   **Is Request Valid? (false) → Determine Approver**  
   Implement mapping rules to produce `approverEmail`.
7. Add **SendGrid** node **Notify Approver** and connect:  
   **Determine Approver → Notify Approver**  
   Configure:
   - To: `={{ $json.approverEmail }}`
   - Subject/body: include request details
   - Include an **approval link** that points to the Wait node resume URL (next steps).  
     - Recommended: include two links (Approve/Reject) that call the resume URL with `approved=true/false` plus optional `comments`.
8. Add **Baserow** node **Create Pending Record** and connect:  
   **Notify Approver → Create Pending Record**  
   Configure:
   - Credentials: Baserow token/host in n8n credentials
   - Operation: **Create**
   - Database ID + Table ID: your real IDs
   - Map fields (example):
     - Request ID ← `={{$json.requestId}}`
     - Requester ← `={{$json.requesterName}}`
     - Department ← `={{$json.department}}`
     - Amount ← `={{$json.amount}}`
     - Status ← `"Pending"`
     - SubmittedAt ← `={{$json.timestamp}}`
   - **Important:** store the returned Baserow row id into a field (e.g. via a Set node) for later updates.
9. Add **Wait** node **Wait for Approval Response** and connect:  
   **Create Pending Record → Wait for Approval Response**  
   Configure:
   - Resume mode enabled (default in this pattern)
   - After saving, run once to generate an execution and identify the resume URL in the node UI (or use the node’s provided resume webhook URL).
10. Go back to **Notify Approver** email and insert the correct resume URL. Decide how you pass decision data:
    - Option A: query parameters (if your resume endpoint parses them)
    - Option B (common): an intermediate “Approve” web page/service that POSTs JSON `{ approved: true, comments: "..." }` to the resume URL
11. Add **Set** node **Extract Approval Result** and connect:  
    **Wait for Approval Response → Extract Approval Result**  
    Configure to output normalized fields, for example:
    - `approved` (boolean)
    - `comments` (string, optional)
    - `decisionAt` (ISO timestamp)
    - `baserowRowId` (carried from earlier steps)
12. Add **IF** node **Approved?** and connect:  
    **Extract Approval Result → Approved?**  
    Condition: boolean `={{$json.approved}}` equals `true`.
13. **Approved branch:**  
    a) Add **Baserow** node **Update Record – Approved**, connect: **Approved? (true) → Update Record – Approved**  
    - Operation: **Update**
    - Row ID: `={{ $json.baserowRowId }}` (recommended; do not rely on `requestId` unless it truly matches Baserow row ids)
    - Set Status = `"Approved"`, DecisionAt = `={{$json.decisionAt}}`  
    b) Add **SendGrid** node **Notify Requester – Approved**, connect: **Update Record – Approved → Notify Requester – Approved**
14. **Rejected branch:**  
    a) Add **Baserow** node **Update Record – Rejected**, connect: **Approved? (false) → Update Record – Rejected**  
    - Status = `"Rejected"`, DecisionAt set  
    b) Add **SendGrid** node **Notify Requester – Rejected**, connect: **Update Record – Rejected → Notify Requester – Rejected**
15. Add **Merge** node **Join End Paths** and connect:
    - **Notify Requester – Approved → Join End Paths (Input 1)**
    - **Notify Requester – Rejected → Join End Paths (Input 1)** (either is fine as long as merge mode matches your intent)
    - **Send Invalid Input Alert → Join End Paths (Input 2)**
    Configure merge mode consistent with your desired behavior (some merge modes wait for both inputs; choose one that won’t block when only one branch runs).
16. Add **Code** node **Finalize** and connect:  
    **Join End Paths → Finalize**  
    Keep it as pass-through or extend with logging/metrics.
17. **Credentials checklist:**
    - SendGrid credential (API key), verified sender
    - Baserow credential (token + base URL if self-hosted)
18. **Activate the workflow** once configuration is complete, then test:
    - Valid request (goes to approver email + wait)
    - Invalid request (sends invalid alert)
    - Resume with approved=true/false payload and verify Baserow updates + requester emails

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow expects a Baserow table with fields: Request ID, Requester, Department, Amount, Status, SubmittedAt, DecisionAt. | From sticky note “📝 Workflow Overview” |
| Replace placeholder IDs (`{{YOUR_DATABASE_ID}}`, `{{YOUR_TABLE_ID}}`) and placeholder emails (`user@example.com`). | Baserow + approver routing code |
| The approval link must target the Wait node resume URL and provide a payload like `{ approved: true, comments: "Looks good" }`. | From sticky note “Section • Notify & Wait” |
| Consider improving validation (schema/regex) and normalizing department names to avoid routing mismatches. | Validation + Determine Approver blocks |
| Important: Baserow update uses `rowId`. Ensure you use Baserow’s actual row id (captured from the create operation), not a business `requestId`, unless they are guaranteed to match. | Update Record – Approved / Rejected nodes |

