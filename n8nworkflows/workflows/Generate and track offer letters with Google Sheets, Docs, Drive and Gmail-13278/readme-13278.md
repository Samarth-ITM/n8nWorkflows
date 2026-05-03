Generate and track offer letters with Google Sheets, Docs, Drive and Gmail

https://n8nworkflows.xyz/workflows/generate-and-track-offer-letters-with-google-sheets--docs--drive-and-gmail-13278


# Generate and track offer letters with Google Sheets, Docs, Drive and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automate the end-to-end lifecycle of offer letters: generate an offer letter PDF from a Google Docs template when a new “Pending” candidate row is added to Google Sheets, store/share the PDF in Google Drive, email the candidate with Accept/Decline links, capture the response via webhook, validate deadline/lock, update Google Sheets status, and send follow-up emails to candidate/HR.

**Primary use cases**
- HR teams generating standardized offer letters at scale.
- Tracking candidate decisions in a single Google Sheet with an enforceable response window (10 days).
- Automatic follow-ups on acceptance/rejection/timeout.

### Logical Blocks
**1.1 Sheet Intake & Eligibility Check (Pending only)**  
Google Sheets trigger → gate on Status = “Pending”.

**1.2 Offer Letter Generation & Delivery**  
Copy Google Docs template → replace placeholders → export to PDF → save to Drive → share link → send candidate email with response links → mark sheet as “Sent”.

**1.3 Response Capture & Validation**  
Webhook receives response → look up candidate row by Id → block if already locked → compute accepted/rejected/timeout based on decision and days since Sent date.

**1.4 Status Update & Notifications**  
Switch based on computed result → update sheet status (Accepted/Rejected/TimeOut) + lock → send appropriate emails (candidate + HR or closure message).

---

## 2. Block-by-Block Analysis

### 2.1 Sheet Intake & Eligibility Check (Pending only)

**Overview:** Polls Google Sheets for newly added rows and proceeds only if the row’s Status is “Pending” (prevents generating letters for rows not ready).

**Nodes involved:**  
- Google Sheets Trigger  
- If4

#### Node: Google Sheets Trigger
- **Type / role:** `googleSheetsTrigger` — polls a sheet and emits data when a new row is added.
- **Configuration (interpreted):**
  - Event: **rowAdded**
  - Polling: every **1 minute**
  - Spreadsheet: “Offer Letter” (documentId provided)
  - Sheet tab: “Sheet1” (gid=0)
- **Key data emitted:** row fields such as `Id, Name, Address, Email, Role, Salary (Lpa), Location, Joining Date, Status, ...`
- **Output connections:** to **If4**
- **Potential failures / edge cases:**
  - Google credential scopes/permission issues.
  - Polling delay or missed events if n8n is down.
  - New row missing required columns (e.g., Status, Name, Email).
  - Duplicate entries if sheet edits create additional “row added” events depending on sheet usage patterns.

#### Node: If4
- **Type / role:** `if` — gatekeeper; only allow “Pending”.
- **Condition:** `{{$json.Status}} == "Pending"`
- **Input:** row from Google Sheets Trigger
- **Outputs:**
  - **True** → Copy Template1
  - **False** → no path (workflow ends for that item)
- **Potential failures / edge cases:**
  - Status casing/spacing mismatches (e.g., “pending”, “ Pending ”) will not pass.
  - If Status is empty, the branch stops silently.

---

### 2.2 Offer Letter Generation & Delivery

**Overview:** Creates a personalized Google Doc from a template, replaces placeholders with candidate data, exports as PDF, uploads to a Drive folder, shares a public link, emails candidate with the link and response buttons, then updates the sheet as “Sent”.

**Nodes involved:**  
- Copy Template1  
- Update a offer letter1  
- Document to PDF1  
- Save PDF to drive1  
- Assign Sharing Rights1  
- Send a message1  
- Update status in sheet1

#### Node: Copy Template1
- **Type / role:** `googleDrive` — copies a template file to a new document per candidate.
- **Operation:** Copy
- **Key configuration:**
  - Template fileId: placeholder `YOUR_GOOGLE_DRIVE_TEMPLATE_FILE_ID`
  - New name: `{{$json.Name}}_offer_letter`
- **Input:** candidate row JSON (from If4 True branch)
- **Output:** new Drive file metadata (includes `id`)
- **Output connections:** to Update a offer letter1
- **Edge cases / failures:**
  - Template fileId invalid or not accessible.
  - Candidate `Name` missing -> odd filenames.
  - Drive quota or permission restrictions.

#### Node: Update a offer letter1
- **Type / role:** `googleDocs` — performs text replacements in the copied Google Doc.
- **Operation:** Update (replaceAll actions)
- **Document reference:** `documentURL: {{$json.id}}` (uses the copied file id from previous node)
- **Replacements (placeholders → values):**
  - `[Candidate Name]` → `{{$('Google Sheets Trigger').item.json.Name}}`
  - `[Candidate address]` → `{{$('Google Sheets Trigger').item.json.Address}}`
  - `[Date]` → `new Date($now).toLocaleDateString('en-In').replace(/\//g,'-')`
  - `[Role]` → `{{$('Google Sheets Trigger').item.json.Role}}`
  - `[Salary]` → `{{$('Google Sheets Trigger').item.json['Salary (Lpa)'].toString()}}`
  - `[Location]` → `{{$('Google Sheets Trigger').item.json.Location}}`
  - `[Joining Date]` → `{{$('Google Sheets Trigger').item.json['Joining Date']}}`
  - `[Last date]` → computed `$now + 10 days` formatted `en-IN` with `-` separators
- **Output connections:** to Document to PDF1
- **Edge cases / failures:**
  - Placeholders not present in template: replacement does nothing.
  - Salary field null/non-string: `.toString()` may throw if undefined.
  - Locale formatting inconsistencies (`en-In` vs `en-IN`) across nodes.
  - Google Docs API limits/timeouts for large documents.

#### Node: Document to PDF1
- **Type / role:** `googleDrive` — downloads the document, converting to PDF.
- **Operation:** Download with conversion
- **FileId:** `{{$json.documentId}}` (note: depends on what Google Docs node outputs)
- **Binary output:** stored in binary property `data`
- **Output connections:** to Save PDF to drive1
- **Edge cases / failures:**
  - **Potential mismatch risk:** the node uses `documentId` from prior output; some Google Docs nodes output `documentId`, others return `id`. If `documentId` is missing, download will fail.
  - Conversion errors if file is not a Google Doc.
  - Large file export timeouts.

#### Node: Save PDF to drive1
- **Type / role:** `googleDrive` — uploads the PDF binary to a specific Drive folder.
- **Operation:** Upload (implicit by providing binary input field)
- **Input binary field:** `data` (configured as `inputDataFieldName: =data`)
- **Filename:** `Name + "_" + currentDate(en-IN)`
- **Destination folder:** placeholder `YOUR_GOOGLE_DRIVE_FOLDER_ID` (“Offer_Letter_PDF”)
- **Output:** uploaded file metadata including `webViewLink`
- **Output connections:** to Assign Sharing Rights1
- **Edge cases / failures:**
  - Folder ID not accessible.
  - Binary property missing if prior step failed.
  - Filename contains forbidden characters for Drive (rare but possible from Name).

#### Node: Assign Sharing Rights1
- **Type / role:** `googleDrive` — sets sharing permissions.
- **Operation:** Share
- **Permission:** `type=anyone`, `role=reader` (public read access)
- **FileId:** `{{$json.id}}` (the uploaded PDF file id)
- **Output connections:** to Send a message1
- **Edge cases / failures:**
  - Organization policy may block “anyone with link”.
  - Permission inheritance conflicts or insufficient rights.

#### Node: Send a message1
- **Type / role:** `gmail` — emails the candidate the offer letter link + response buttons.
- **To:** `{{$('Google Sheets Trigger').item.json.Email}}`
- **Subject:** `Job Offer from Your Company, Inc.`
- **Body highlights:**
  - Includes offer link: `{{$('Save PDF to drive1').item.json.webViewLink}}`
  - Includes two call-to-action links (Accept/Decline) pointing to localhost test URLs:
    - `http://localhost:5678/webhook-test/YOUR_N8N_WEBHOOK_ID?Id={{ $('If4').item.json.Id}}&decision=accepted`
    - `http://localhost:5678/webhook-test/YOUR_N8N_WEBHOOK_PATH?Id={{ $('If4').item.json.Id}}&decision=rejected`
- **Output connections:** to Update status in sheet1
- **Edge cases / failures:**
  - **Critical mismatch risk:** webhook links use `webhook-test` and placeholders; in production, they must point to the **production webhook URL** and correct path.
  - The webhook expects more query params later in the workflow (`email`, `name`), but these links only send `Id` and `decision`.
  - Gmail sending limits, OAuth token expiration, “From” alias restrictions.

#### Node: Update status in sheet1
- **Type / role:** `googleSheets` — updates the same row by Id to indicate offer sent and stores link/date.
- **Operation:** Update with matching column
- **Match:** `Id` column
- **Values written:**
  - `Id` → `{{$('If4').item.json.Id}}`
  - `Date` → today formatted `en-In` with `-`
  - `Status` → `"Sent"`
  - `Offer Letter` → `{{$('Save PDF to drive1').item.json.webViewLink}}`
  - `row_number` present in schema but set to 0 in values (not typically needed; may be ignored)
- **Output connections:** none
- **Edge cases / failures:**
  - If `Id` not unique, wrong row could be updated.
  - If sheet uses different header names, update fails.
  - Date format must match later parsing logic (see Code node).

---

### 2.3 Response Capture & Validation

**Overview:** Receives candidate’s decision via webhook, retrieves the candidate row from the sheet, prevents multiple responses using “Response Locked”, and computes final outcome (accepted/rejected/timeout) using a 10-day window from the “Sent” date.

**Nodes involved:**  
- Webhook1  
- Get row(s) in sheet2  
- Check Response is Locked?1  
- Code in JavaScript3  
- Switch1

#### Node: Webhook1
- **Type / role:** `webhook` — entry point for candidate response.
- **Path:** `b04a8eb4-a672-41bd-9c8a-8cb538aaf3b4`
- **Expected query parameters (based on downstream usage):**
  - `Id` (used to look up row)
  - `decision` (`accepted` / `rejected`)
  - The email templates also reference `query.email` and `query.name` (but not actually provided in the offer email links as currently written)
- **Output connections:** to Get row(s) in sheet2
- **Edge cases / failures:**
  - If links still point to localhost/test URLs, real candidates cannot reach your n8n instance.
  - Missing query parameters will break downstream nodes (lookup and emails).

#### Node: Get row(s) in sheet2
- **Type / role:** `googleSheets` — retrieves candidate row by Id.
- **Operation:** Read rows with filter
- **Filter:** lookupColumn `Id` = `{{$json.query.Id}}` (Id from webhook query)
- **Spreadsheet:** same “Offer Letter” docId, sheet “Sheet1” (note cachedResultUrl shows placeholder in one place, but docId is real)
- **Output connections:** to Check Response is Locked?1
- **Edge cases / failures:**
  - If Id not found: returns no items; downstream nodes won’t run.
  - If multiple rows share same Id: may return multiple items; lock/status updates could apply to multiple rows unintentionally.
  - Permission or API quota issues.

#### Node: Check Response is Locked?1
- **Type / role:** `if` — prevents repeated/late processing if response already locked.
- **Condition:** `{{$json['Response Locked']}} == true`
- **Outputs:**
  - **True branch:** no connection (workflow stops; effectively ignores response)
  - **False branch:** to Code in JavaScript3
- **Edge cases / failures:**
  - “Response Locked” column might be absent; then expression evaluates undefined vs boolean and may route to false.
  - Values stored as strings (“True”) vs boolean `true` can cause mismatch. (Later updates set `"True"` as a string.)

#### Node: Code in JavaScript3
- **Type / role:** `code` — derives the outcome: accepted/rejected/timeout.
- **Key logic:**
  - Reads `decision` from `$('Webhook1').first().json.query.decision`
  - Reads sent date string from `$input.first().json.Date`
  - Parses date as **DD-MM-YYYY**
  - Computes `diffDays` between today and sendDate
  - If `diffDays >= 10` → `"timeout"`
  - Else if decision == `"accepted"` → `"accepted"`
  - Else → `"rejected"`
- **Output:** `{ result: "accepted" | "rejected" | "timeout" }`
- **Output connections:** to Switch1
- **Edge cases / failures:**
  - If `Date` is missing or not in `DD-MM-YYYY`, `split('-')` fails or produces invalid date.
  - Uses `Math.abs(today - sendDate)`: if a future date exists, it still counts as valid; you may want directional logic instead.
  - Decision missing defaults to “rejected” due to `else`.

#### Node: Switch1
- **Type / role:** `switch` — routes based on `{{$json.result}}`.
- **Rules:**
  - equals `accepted` → Update Status to Accepted1
  - equals `rejected` → Update Status to Rejected1
  - equals `timeout` → Update Status Timeout1
- **Output connections:** to status update nodes
- **Edge cases / failures:**
  - If result is any other value, nothing happens (no default route configured).

---

### 2.4 Status Update & Notifications

**Overview:** Writes final status back to Google Sheets and sends corresponding emails. Acceptance triggers two emails (candidate + HR). Rejection and timeout send candidate-only messages.

**Nodes involved:**  
- Update Status to Accepted1  
- Ack. Candidate1  
- Ack. Hr1  
- Update Status to Rejected1  
- Thank you to Candidate2  
- Update Status Timeout1  
- Thank you to Candidate3

#### Node: Update Status to Accepted1
- **Type / role:** `googleSheets` — updates row by Id with acceptance and locks further responses.
- **Match:** `Id` = `{{$('Webhook1').item.json.query.Id}}`
- **Values written:** `Status="Accepted"`, `Response Locked="True"`
- **Output connections:** to Ack. Candidate1 and Ack. Hr1 (both run)
- **Edge cases / failures:**
  - Stores `"True"` as string; lock check expects boolean `true` → may not lock properly.
  - If Id not found, no update occurs but emails might still send depending on node execution/data.

#### Node: Ack. Candidate1
- **Type / role:** `gmail` — sends acceptance acknowledgment to candidate.
- **To:** `{{$('Get row(s) in sheet2').item.json.Email}}`
- **Body uses:** `Name` from the fetched sheet row
- **Edge cases:**
  - If Get row(s) returns multiple items, email may be sent multiple times.
  - HTML body contains a stray `</strong>.` and a malformed paragraph segment (formatting issue, not fatal).

#### Node: Ack. Hr1
- **Type / role:** `gmail` — informs HR that candidate accepted.
- **To:** hardcoded placeholder `YOUR_GMAIL_RECIPIENT_EMAIL`
- **Body uses:** `{{$('Webhook1').item.json.query.name}}`
- **Edge cases:**
  - Offer email links do not supply `name`, so HR email may show blank unless query includes it.
  - Replace placeholder recipient with real HR mailbox/group.

#### Node: Update Status to Rejected1
- **Type / role:** `googleSheets` — updates row to Rejected and locks.
- **Values:** `Status="Rejected"`, `Response Locked="True"`
- **Output connections:** to Thank you to Candidate2
- **Edge cases:** same “True” string vs boolean lock mismatch.

#### Node: Thank you to Candidate2
- **Type / role:** `gmail` — sends rejection follow-up.
- **To:** `{{$('Webhook1').item.json.query.email}}`
- **Body uses:** `query.name`
- **Edge cases:**
  - Offer response links do not include `email`/`name`, so these may be empty unless you add them.

#### Node: Update Status Timeout1
- **Type / role:** `googleSheets` — updates row to TimeOut and locks.
- **Values:** `Status="TimeOut"`, `Response Locked="True"`
- **Output connections:** to Thank you to Candidate3

#### Node: Thank you to Candidate3
- **Type / role:** `gmail` — sends timeout closure message.
- **To:** `{{$('Webhook1').item.json.query.email}}`
- **Body uses:** `query.name`
- **Edge cases:** same missing query parameters issue.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | googleSheetsTrigger | Detect new candidate rows | — | If4 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| If4 | if | Gate: only Status=Pending | Google Sheets Trigger | Copy Template1 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Copy Template1 | googleDrive | Copy Google Docs template | If4 | Update a offer letter1 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Update a offer letter1 | googleDocs | Replace placeholders in doc | Copy Template1 | Document to PDF1 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Document to PDF1 | googleDrive | Export/download as PDF | Update a offer letter1 | Save PDF to drive1 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Save PDF to drive1 | googleDrive | Upload PDF to folder | Document to PDF1 | Assign Sharing Rights1 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Assign Sharing Rights1 | googleDrive | Share PDF publicly (anyone reader) | Save PDF to drive1 | Send a message1 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Send a message1 | gmail | Email offer letter + accept/decline links | Assign Sharing Rights1 | Update status in sheet1 | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Update status in sheet1 | googleSheets | Mark as Sent + store date/link | Send a message1 | — | ## 1. Generate & Send Offer Letter<br>• Watches Google Sheets for new Pending entries<br>• Creates offer letter from template<br>• Converts to PDF, saves to Drive<br>• Sends email with Accept / Decline buttons |
| Webhook1 | webhook | Capture candidate decision | — | Get row(s) in sheet2 | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Get row(s) in sheet2 | googleSheets | Fetch candidate row by Id | Webhook1 | Check Response is Locked?1 | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Check Response is Locked?1 | if | Stop if already locked | Get row(s) in sheet2 | Code in JavaScript3 (false branch) | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Code in JavaScript3 | code | Compute accepted/rejected/timeout | Check Response is Locked?1 | Switch1 | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Switch1 | switch | Route by result | Code in JavaScript3 | Update Status to Accepted1 / Update Status to Rejected1 / Update Status Timeout1 | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Update Status to Accepted1 | googleSheets | Set Accepted + lock | Switch1 | Ack. Candidate1, Ack. Hr1 | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Ack. Candidate1 | gmail | Acceptance email to candidate | Update Status to Accepted1 | — | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Ack. Hr1 | gmail | Acceptance notice to HR | Update Status to Accepted1 | — | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Update Status to Rejected1 | googleSheets | Set Rejected + lock | Switch1 | Thank you to Candidate2 | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Thank you to Candidate2 | gmail | Rejection email to candidate | Update Status to Rejected1 | — | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Update Status Timeout1 | googleSheets | Set TimeOut + lock | Switch1 | Thank you to Candidate3 | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Thank you to Candidate3 | gmail | Timeout closure email | Update Status Timeout1 | — | ## 2. Capture Response & Follow-Up<br>• Receives candidate response via webhook<br>• Checks response validity & deadline<br>• Updates final status in Google Sheets<br>• Sends follow-up emails to candidate & HR |
| Sticky Note | stickyNote | Documentation block (canvas note) | — | — | ## Automated Offer Letter & Response Tracking Workflow<br>### How it works<br>This workflow automates the complete offer letter lifecycle.<br>... (as provided in the note) |
| Sticky Note1 | stickyNote | Documentation block (canvas note) | — | — | (content as shown: “1. Generate & Send Offer Letter…”) |
| Sticky Note2 | stickyNote | Documentation block (canvas note) | — | — | (content as shown: “2. Capture Response & Follow-Up…”) |

---

## 4. Reproducing the Workflow from Scratch

1) **Prepare Google assets**
   1. Create a Google Sheet (e.g., “Offer Letter”) with headers at minimum:  
      `Id, Name, Address, Email, Role, Salary (Lpa), Location, Joining Date, Status, Date, Offer Letter, Response Locked`
   2. Ensure new candidates are added as new rows with `Status = Pending` and unique `Id`.
   3. Create a Google Docs template containing placeholders exactly matching:  
      `[Candidate Name]`, `[Candidate address]`, `[Date]`, `[Role]`, `[Salary]`, `[Location]`, `[Joining Date]`, `[Last date]`
   4. Create a Google Drive folder for generated PDFs.

2) **Credentials**
   1. In n8n, configure credentials for:
      - **Google Sheets** (OAuth2)
      - **Google Drive** (OAuth2)
      - **Google Docs** (OAuth2)
      - **Gmail** (OAuth2)
   2. Ensure the Google account has access to the sheet, template, and destination folder.

3) **Block 1: Trigger and Pending gate**
   1. Add node **Google Sheets Trigger**:
      - Event: `Row added`
      - Poll every: `1 minute`
      - Select Spreadsheet and Sheet tab.
   2. Add node **If** named **If4**:
      - Condition: String equals → `{{$json.Status}}` equals `Pending`
   3. Connect: **Google Sheets Trigger → If4 (main)**.

4) **Block 2: Generate offer, save, share, email, update sheet**
   1. Add **Google Drive** node **Copy Template1**:
      - Operation: `Copy`
      - File ID: your Docs template file id
      - Name: `{{$json.Name}}_offer_letter`
   2. Add **Google Docs** node **Update a offer letter1**:
      - Operation: `Update`
      - Document: use expression pointing to copied file id (as your node output provides)
      - Add Replace All actions for each placeholder (use the same expressions as in the workflow).
   3. Add **Google Drive** node **Document to PDF1**:
      - Operation: `Download`
      - Enable Google file conversion: Docs → `application/pdf`
      - Binary property name: `data`
      - File ID: reference the updated doc’s id/documentId output.
   4. Add **Google Drive** node **Save PDF to drive1**:
      - Operation: upload (by providing binary field)
      - Input binary field: `data`
      - Folder: select your destination folder
      - Name: `{{$('Google Sheets Trigger').item.json.Name + "_" + new Date().toLocaleDateString('en-IN')}}`
   5. Add **Google Drive** node **Assign Sharing Rights1**:
      - Operation: `Share`
      - File ID: `{{$json.id}}`
      - Permissions: `anyone` + `reader`
   6. Add **Gmail** node **Send a message1**:
      - To: `{{$('Google Sheets Trigger').item.json.Email}}`
      - Subject/body: include `{{$('Save PDF to drive1').item.json.webViewLink}}`
      - Replace accept/decline URLs with your real webhook URL (see step 5).
      - Make sure query includes at least `Id` and `decision` (and ideally `name` and `email`).
   7. Add **Google Sheets** node **Update status in sheet1**:
      - Operation: `Update`
      - Match column: `Id`
      - Set `Status=Sent`, `Date=today`, `Offer Letter=webViewLink`
   8. Connect in order:
      - If4 (true) → Copy Template1 → Update a offer letter1 → Document to PDF1 → Save PDF to drive1 → Assign Sharing Rights1 → Send a message1 → Update status in sheet1

5) **Block 3: Webhook response intake and row lookup**
   1. Add **Webhook** node **Webhook1**:
      - Path: generate a path (n8n provides one)
      - Method: GET (default works for query links)
   2. Update the email links in **Send a message1** to:  
      `https://<your-n8n-domain>/webhook/<your-path>?Id={{...}}&decision=accepted&name={{...}}&email={{...}}`  
      and similarly for rejected.
   3. Add **Google Sheets** node **Get row(s) in sheet2**:
      - Operation: Read rows
      - Filter: `Id` equals `{{$json.query.Id}}`
   4. Add **If** node **Check Response is Locked?1**:
      - Condition: boolean equals → `{{$json['Response Locked']}}` equals `true`
      - Use the **false** branch to continue processing (true branch stops).
   5. Connect: Webhook1 → Get row(s) in sheet2 → Check Response is Locked?1 (main)

6) **Block 4: Compute decision & route**
   1. Add **Code** node **Code in JavaScript3** with the same logic (deadline 10 days; parse DD-MM-YYYY from sheet Date).
   2. Add **Switch** node **Switch1**:
      - Value: `{{$json.result}}`
      - Cases: accepted, rejected, timeout
   3. Connect: Check Response is Locked?1 (false) → Code in JavaScript3 → Switch1

7) **Block 5: Update final status and send emails**
   1. Add **Google Sheets** node **Update Status to Accepted1**:
      - Update by Id: `{{$('Webhook1').item.json.query.Id}}`
      - Set `Status=Accepted`, `Response Locked=True`
   2. Add **Gmail** node **Ack. Candidate1**:
      - To: candidate email (prefer from looked-up row)
   3. Add **Gmail** node **Ack. Hr1**:
      - To: HR email/group
   4. Add **Google Sheets** node **Update Status to Rejected1** + **Gmail** node **Thank you to Candidate2**
   5. Add **Google Sheets** node **Update Status Timeout1** + **Gmail** node **Thank you to Candidate3**
   6. Connect:
      - Switch1(accepted) → Update Status to Accepted1 → (Ack. Candidate1 and Ack. Hr1)
      - Switch1(rejected) → Update Status to Rejected1 → Thank you to Candidate2
      - Switch1(timeout) → Update Status Timeout1 → Thank you to Candidate3

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated Offer Letter & Response Tracking Workflow: generates offer letters from a template, sends email with Accept/Decline, captures response via webhook, validates deadline, updates sheet, notifies HR/candidate. | From workflow sticky note “Automated Offer Letter & Response Tracking Workflow” |
| Setup reminders: connect Google Sheets/Docs/Drive/Gmail credentials; replace placeholder IDs; ensure required sheet columns; initial Status must be Pending. | From sticky note content |
| Customization: adjust deadline logic in Code node; edit email tone; add Slack/WhatsApp notifications. | From sticky note content |

