Send validated outreach emails with Gmail & Google Sheets anti-spam controls

https://n8nworkflows.xyz/workflows/send-validated-outreach-emails-with-gmail---google-sheets-anti-spam-controls-10972


# Send validated outreach emails with Gmail & Google Sheets anti-spam controls

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Send validated outreach emails with Gmail & Google Sheets anti-spam controls  
**Workflow name (in n8n):** Automated Outreach Emails with Gmail & Google Sheets Safety Controls  
**Purpose:** Automatically send **initial cold outreach emails** to contacts stored in **Google Sheets**, with multiple “safety controls” to reduce spam risk and prevent duplicates: email cleanup/validation, spam flag exclusion, deliverability gate, human-like delays, weekday-only sending, daily quota enforcement, and post-send status updates back into Sheets.

**Primary use cases**
- Low-volume initial outreach campaigns where you must avoid re-sending to the same contact.
- Controlled Gmail sending with quotas and randomized delays.
- Sheet-driven pipeline: “who to contact” + “status tracking” remains in Google Sheets.

### 1.1 Scheduled Input & Data Fetch
Runs daily at 9am (Africa/Lagos), pulls rows from a Google Sheet.

### 1.2 Contact Eligibility Filtering (Email present + status exclusions)
Filters out contacts lacking emails and contacts with excluded statuses (e.g., “spam”, “do not contact”).

### 1.3 Email Cleanup/Validation + Spam Flag Gate
Normalizes email strings, blocks certain domains/keywords, and skips rows marked as SPAM.

### 1.4 Verification/Deliverability + “Already Sent” Protection
Requires verification result fields to be present and “REACHABLE…”, then checks the “STATUS/SENT” fields to ensure the initial email hasn’t already been sent.

### 1.5 Human-like Delay + Daily Quota Enforcement (Weekdays only)
Adds random wait per email and enforces a daily global quota with category breakdown.

### 1.6 Content Generation, Final Duplicate Check, Send, and Sheet Update
Generates randomized subject/body lines, re-checks “not already sent”, downloads an attachment (currently not configured), sends Gmail email, then updates the Sheet with “SENT ✅” and “DATE SENT”.

---

## 2. Block-by-Block Analysis

### Block A — Scheduled Trigger & Source Sheet Read
**Overview:** Starts the workflow daily and loads contact rows from a specific Google Sheet tab.  
**Nodes involved:** `9am Daily`, `EMAILS SHEET`, `10 SECONDS WAIT`

#### Node: 9am Daily
- **Type / role:** Schedule Trigger; entry point.
- **Config:** Runs every day at **09:00**.
- **Connections:** Outputs to `EMAILS SHEET`.
- **Version:** 1.2
- **Failure modes:** n8n schedule disabled; timezone mismatch with business hours.

#### Node: EMAILS SHEET
- **Type / role:** Google Sheets; reads rows (operation not explicitly shown in parameters, so it uses node defaults for “read/get many” depending on UI state).
- **Config choices (interpreted):**
  - **Document:** Spreadsheet “JOB” (`1ua7aVmykESCwx6JBHBZHDmNyg7tmTkGnaL4WeoptHSE`)
  - **Sheet tab:** “PORTFOLIO” (gid `1160858465`)
  - **Auth:** Service Account (`googleApi` credential)
- **Connections:** Outputs to `10 SECONDS WAIT`.
- **Version:** 4.7
- **Failure modes / edge cases:**
  - Service account not shared on the spreadsheet → 403.
  - Sheet name/gid changed → read fails or returns empty.
  - Large sheet sizes can slow executions.

#### Node: 10 SECONDS WAIT
- **Type / role:** Wait; throttling before processing.
- **Config:** Fixed wait **10 seconds**.
- **Connections:** Outputs to `Get Contacts with Emails`.
- **Version:** 1.1
- **Failure modes:** None typical; may delay execution beyond desired window.

---

### Block B — Initial Filtering: “Contact has email and acceptable status”
**Overview:** Keeps only rows that have an email and are not in a list of invalid statuses.  
**Nodes involved:** `Get Contacts with Emails`

#### Node: Get Contacts with Emails
- **Type / role:** Code; filters items.
- **Logic:**
  - Reads `item.json.EMAIL` and trims it.
  - Reads `item.json.status` (lowercased).
  - Excludes statuses: `hot lead`, `unresponsive`, `spam`, `do not contact`.
  - Keeps rows where `EMAIL` exists AND status is not in excluded list.
- **Connections:** Outputs to `VALID EMAIL`.
- **Version:** 2
- **Edge cases / failure types:**
  - Uses `status` (lowercase field name) while later nodes use `STATUS` (uppercase). If your sheet column is `STATUS`, this filter may not exclude correctly.
  - If `status` is undefined, it passes as eligible (unless email missing).

---

### Block C — Email Cleanup + Validation Gate
**Overview:** Normalizes emails, validates syntax, blocks known bad domains/keywords, and routes only valid emails forward.  
**Nodes involved:** `VALID EMAIL`, `VALID EMAIL?`

#### Node: VALID EMAIL
- **Type / role:** Code; transforms each item by adding validation fields.
- **Key behavior:**
  - Lowercases and trims `EMAIL`, removes trailing dots/commas.
  - Validates syntax using regex:
    - Disallows double dots, requires `local@domain.tld`.
  - Extracts domain (match group 2).
  - Blocks:
    - Exact domains (e.g., `facebook.com`, `linkedin.com`, `tiktok.com`, plus crypto sites listed).
    - Keywords (e.g., `"pumpfun"`, and specifically `"@x.com"`).
  - Writes:
    - `valid` (boolean)
    - `blocked` (boolean)
    - `reason` (“Invalid format” / “Blocked keyword/domain” / “Valid email”)
- **Connections:** Outputs to `VALID EMAIL?`.
- **Version:** 2
- **Edge cases:**
  - Regex allows many valid emails but not all edge-valid forms; may reject uncommon valid addresses.
  - Blocking `@x.com` will reject legitimate X addresses; intended.
  - Domain extraction depends on regex match; if no match domain becomes `""`.

#### Node: VALID EMAIL?
- **Type / role:** IF; routes only valid emails forward.
- **Condition:** `$json.valid === true`
- **Connections:**
  - **True** → `Is SPAM?`
  - **False** → stops (no connected node)
- **Version:** 2.2
- **Failure modes:** If `valid` missing (unexpected data), it routes to false and drops the item.

---

### Block D — Spam Flag Exclusion
**Overview:** Skips contacts explicitly flagged as SPAM to protect sender reputation.  
**Nodes involved:** `Is SPAM?`

#### Node: Is SPAM?
- **Type / role:** IF; checks a SPAM marker column.
- **Condition:** `$json.SPAM` **contains** `"YES"`.
- **Connections (as wired):**
  - **True (spam)** → no output connected (drops item)
  - **False (not spam)** → `EMAIL VERIFICATION DONE?`
- **Version:** 2.2
- **Edge cases:**
  - If `SPAM` column is empty, it won’t “contain YES” → treated as not spam.
  - Different conventions (“Yes”, “TRUE”, “1”) won’t match unless they still contain “YES”.

---

### Block E — Verification/Reachability + Name + Initial “Already Sent” Check
**Overview:** Requires email verification fields to exist and indicates reachability, ensures a name exists, and prevents sending if the initial email was already sent.  
**Nodes involved:** `EMAIL VERIFICATION DONE?`, `undelivered email?`, `Has Name?`, `Initial Email NOT Sent?`, `Initial Email`

#### Node: EMAIL VERIFICATION DONE?
- **Type / role:** IF; ensures verification was completed.
- **Condition:** `$json['EMAIL VERIFIED']` is **not empty**.
- **Connections:**
  - **True** → `undelivered email?`
  - **False** → drops item
- **Version:** 2.2
- **Edge cases:** If you don’t populate `EMAIL VERIFIED` in the sheet, nothing proceeds.

#### Node: undelivered email?
- **Type / role:** IF; checks deliverability status.
- **Condition:** `$json["EMAIL VERIFIED"]` **starts with** `"REACHABLE"`.
- **Connections:**
  - **True** → `Has Name?`
  - **False** → drops item
- **Version:** 2.2
- **Edge cases:** If your verifier uses “Valid”, “Deliverable”, etc., this will drop all.

#### Node: Has Name?
- **Type / role:** IF; ensures personalization field exists.
- **Condition:** `$json.Name` is **not empty**.
- **Connections:**
  - **True** → `Initial Email NOT Sent?`
  - **False** → drops item
- **Version:** 2.2
- **Edge cases:** If you want to allow missing names, you’d route false forward and use fallback `"there"`.

#### Node: Initial Email NOT Sent?
- **Type / role:** IF; avoids duplicates.
- **Condition:** `$json.STATUS` is **empty** (operator “empty” on `STATUS`).
  - Note: `rightValue` is `"SENT "` but is unused for “empty” checks.
- **Connections:**
  - **True** → `Initial Email` (SplitInBatches)
  - **False** → drops item
- **Version:** 2.2
- **Edge cases / data consistency:**
  - This assumes a `STATUS` column exists and is blank for unsent contacts.
  - But the sheet update node writes to column `SENT`, not `STATUS`. If you don’t also maintain `STATUS`, this duplicate-prevention may fail.

#### Node: Initial Email
- **Type / role:** Split In Batches; processes contacts in batch loop (typical for pacing).
- **Config:** Options empty → defaults to batch size (commonly 1 unless set in UI).
- **Connections:**
  - Output index 1 (the “continue” branch) goes to `Human wait time1` (as wired).
  - The other output is not used.
- **Version:** 3
- **Edge cases:**
  - If batch size > 1, delays/quota checks may not behave as intended.
  - Incorrect wiring can lead to never-ending loops if “next batch” is mishandled; here, the loop is closed later by `UPDATE EMAIL SHEET` → `Initial Email`.

---

### Block F — Human-like Delay + Email Type + Daily Quota Gate (Weekday only)
**Overview:** Adds a randomized wait, sets email type to “initial”, and enforces daily quotas using global static data, skipping weekends.  
**Nodes involved:** `Human wait time1`, `Wait`, `Initial`, `Check Daily Email Counter`

#### Node: Human wait time1
- **Type / role:** Code; generates delay value.
- **Logic:**
  - Random integer `delay` between 0 and 120 seconds.
  - Returns `{ delay: delay * 1000 }`.
- **Connections:** Outputs to `Wait`.
- **Version:** 2
- **Edge cases:** Comment says “0–60 seconds” but code is **0–120 seconds**.

#### Node: Wait
- **Type / role:** Wait; sleeps per item.
- **Config:** `amount = Number($json.delay) / 1000`
- **Connections:** Outputs to `Initial` (Set).
- **Version:** 1.1
- **Failure modes:** If `delay` missing/not numeric, expression becomes `NaN` and may error.

#### Node: Initial
- **Type / role:** Set; categorizes the email.
- **Config:** Adds `emailType = "initial"` while keeping other fields.
- **Connections:** Outputs to `Check Daily Email Counter`.
- **Version:** 3.4

#### Node: Check Daily Email Counter
- **Type / role:** Code; daily quota + weekend exclusion + category quotas.
- **Key logic:**
  - If Saturday/Sunday → returns `[]` (hard stop, nothing sent).
  - `totalLimit = 100` emails/day.
  - Uses `$getWorkflowStaticData('global')`:
    - Resets counters when date changes.
    - Tracks `sentCount` and `breakdown` per type (`initial`, `retarget1`, `retarget2`, `reply`).
  - Enforces:
    - Total daily quota (`sentCount >= totalLimit` → stop).
    - Per-category quota using ratios (initial 55%, etc.).
  - If allowed, increments counters and returns items.
- **Connections:** Outputs to `INITIAL EMAIL SHEET`.
- **Version:** 2
- **Edge cases / operational notes:**
  - Static data is shared across workflows (“global” scope) and persists across executions; this is intended.
  - Concurrency: multiple executions in parallel can race and exceed quotas (no locking).
  - Returning `[]` silently stops downstream processing; useful but can look like “workflow did nothing”.

---

### Block G — Sheet Re-check, Content Generation, Final Sent Check
**Overview:** Re-reads the sheet dataset and ensures the currently processed email still matches, then generates randomized subject/body content, then does a second “not sent” check right before sending.  
**Nodes involved:** `INITIAL EMAIL SHEET`, `ONLY CHECK EMAIL NEEDED`, `SUBJECT LINES`, `Initial Email NOT Sent? SECOND CHECK`

#### Node: INITIAL EMAIL SHEET
- **Type / role:** Google Sheets; reads same document/tab again.
- **Purpose (implicit):** Provide a fresh sheet snapshot to avoid mismatches/updated statuses.
- **Connections:** Outputs to `ONLY CHECK EMAIL NEEDED`.
- **Version:** 4.7
- **Failure modes:** Same as earlier Sheets node.

#### Node: ONLY CHECK EMAIL NEEDED
- **Type / role:** Filter; keeps only the row matching the current batch email.
- **Condition:** `$json.EMAIL == $('Initial Email').item.json.EMAIL`
- **Connections:** Outputs to `SUBJECT LINES`.
- **Version:** 2.2
- **Edge cases:**
  - If the email address in the sheet differs by casing/spacing from processed email, it may not match (though earlier normalization helps).
  - If duplicates exist in sheet with same email, filter returns multiple rows → may send multiple times.

#### Node: SUBJECT LINES
- **Type / role:** Code; creates randomized subject + content components.
- **Config / behavior:**
  - Randomly picks from:
    - Subject pool (5 items)
    - Opening pool (3)
    - Body pool (3)
    - Closing pool (2)
  - Sets:
    - `Subject`, `Greeting`, `Opening`, `Body`, `Closing`, `Signature`
  - Signature uses `Sender Name` field or fallback `"Your Name"`.
  - Outputs merged JSON with original fields preserved.
- **Connections:** Outputs to `Initial Email NOT Sent? SECOND CHECK`.
- **Version:** 2
- **Edge cases:** HTML signature is included; Gmail node sends HTML body using `<br>` and `<p>` tags (works, but ensure correct content-type in Gmail node).

#### Node: Initial Email NOT Sent? SECOND CHECK
- **Type / role:** IF; final duplicate-prevention gate before sending.
- **Condition:** `$json.STATUS` is **empty** (same issue as earlier).
- **Connections:**
  - **True** → `Download file`
  - **False** → drops item
- **Version:** 2.2
- **Edge cases:** If you only update column `SENT` (not `STATUS`), this may remain empty and allow re-sends.

---

### Block H — Attachment Download, Send Gmail, Update Sheet, Loop Next Batch
**Overview:** Downloads an attachment (currently not configured), sends the Gmail message, updates the sheet row to mark as sent, then loops back to process the next batch item.  
**Nodes involved:** `Download file`, `SEND INITIAL EMAIL`, `UPDATE EMAIL SHEET`, `Initial Email`

#### Node: Download file
- **Type / role:** Google Drive; downloads a file to attach.
- **Config:** `operation: download`, but **fileId is empty** (URL mode with empty value).
- **Connections:** Outputs to `SEND INITIAL EMAIL`.
- **Version:** 3
- **Failure modes / edge cases:**
  - As configured, will fail or download nothing (missing fileId).
  - If you intend to attach, you must set `fileId` or drive URL and ensure it outputs **binary** data.

#### Node: SEND INITIAL EMAIL
- **Type / role:** Gmail; sends the outreach email.
- **Key configuration:**
  - **To:** `{{$json.EMAIL}}`
  - **Subject:** from `$('SUBJECT LINES').item.json.Subject`
  - **Body:** HTML template using `Opening`, `Body`, `Closing`, `Signature` and greeting fallback.
  - **Sender name:** “Your Name - Highlight”
  - **Attachments:** `attachmentsBinary` present but not mapped to a binary property (empty object).
  - `appendAttribution: false`
  - `maxTries: 2`
- **Connections:** Outputs to `UPDATE EMAIL SHEET`.
- **Version:** 2.1
- **Failure modes:**
  - Gmail OAuth expired/insufficient scope → auth error.
  - Sending limits / reputation throttling by Google.
  - Attachment misconfiguration (binary missing) can error depending on node behavior.

#### Node: UPDATE EMAIL SHEET
- **Type / role:** Google Sheets; updates status fields for the sent email.
- **Operation:** `update` with matching on `EMAIL`.
- **Columns written:**
  - `SENT = "SENT ✅"`
  - `EMAIL = $('Initial Email NOT Sent? SECOND CHECK').item.json.EMAIL`
  - `DATE SENT = today ISO date (YYYY-MM-DD)`
- **Matching columns:** `EMAIL`
- **Connections:** Outputs back to `Initial Email` (to continue next batch).
- **Version:** 4.7
- **Failure modes / edge cases:**
  - If EMAIL duplicates exist, update may affect multiple rows or an unintended row depending on Sheets node behavior.
  - If `EMAIL` is altered (normalized) but sheet stores a different string, match may fail and not update, enabling re-sends later.
  - Note mismatch: workflow gates on `STATUS` but updates `SENT`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 9am Daily | Schedule Trigger | Daily start trigger at 9am | — | EMAILS SHEET | ## 📧 AUTOMATED INITIAL COLD OUTREACH EMAILS WITH SAFETY CONTROLS / HOW? / SETUP STEPS / GMAIL PAYLOAD… |
| EMAILS SHEET | Google Sheets | Load contacts from sheet | 9am Daily | 10 SECONDS WAIT | ## STEP 1:  Get Contats From Sheets / This step pulls contact records… |
| 10 SECONDS WAIT | Wait | Initial throttling | EMAILS SHEET | Get Contacts with Emails | ## STEP 1:  Get Contats From Sheets / This step pulls contact records… |
| Get Contacts with Emails | Code | Filter rows with emails and acceptable status | 10 SECONDS WAIT | VALID EMAIL | ## STEP 1:  Get Contats From Sheets / This step pulls contact records… |
| VALID EMAIL | Code | Clean, validate, and block certain emails | Get Contacts with Emails | VALID EMAIL? | ## STEP 2:  Clean and Validate Emails / Fixes formatting… |
| VALID EMAIL? | IF | Route valid emails only | VALID EMAIL | Is SPAM? | ## STEP 2:  Clean and Validate Emails / Fixes formatting… |
| Is SPAM? | IF | Skip rows marked as spam | VALID EMAIL? | EMAIL VERIFICATION DONE? (false branch) | ## STEP 3: Spam Check / Spam records are skipped… |
| EMAIL VERIFICATION DONE? | IF | Require verification field present | Is SPAM? | undelivered email? | ## STEP 4: Initial Email Status Check / checks if an initial email was already sent… |
| undelivered email? | IF | Require “REACHABLE…” | EMAIL VERIFICATION DONE? | Has Name? | ## STEP 4: Initial Email Status Check / checks if an initial email was already sent… |
| Has Name? | IF | Ensure contact has Name | undelivered email? | Initial Email NOT Sent? | ## STEP 4: Initial Email Status Check / checks if an initial email was already sent… |
| Initial Email NOT Sent? | IF | Prevent duplicate initial sends (STATUS empty) | Has Name? | Initial Email | ## STEP 4: Initial Email Status Check / checks if an initial email was already sent… |
| Initial Email | Split In Batches | Batch pacing + loop control | Initial Email NOT Sent?, UPDATE EMAIL SHEET | Human wait time1 | ## STEP 5: Human Like Delay / random delay added… |
| Human wait time1 | Code | Generate random delay (0–120s) | Initial Email | Wait | ## STEP 5: Human Like Delay / random delay added… |
| Wait | Wait | Sleep for delay | Human wait time1 | Initial | ## STEP 5: Human Like Delay / random delay added… |
| Initial | Set | Set emailType=initial | Wait | Check Daily Email Counter | ## STEP 6: Daily Status Check / blocks weekends, limits… |
| Check Daily Email Counter | Code | Enforce daily quotas and weekend exclusion | Initial | INITIAL EMAIL SHEET | ## STEP 6: Daily Status Check / blocks weekends, limits… |
| INITIAL EMAIL SHEET | Google Sheets | Re-read sheet for latest row state | Check Daily Email Counter | ONLY CHECK EMAIL NEEDED | ## STEP 7: Initial Email Check / prevents mismatched data… |
| ONLY CHECK EMAIL NEEDED | Filter | Keep only row matching the current email | INITIAL EMAIL SHEET | SUBJECT LINES | ## STEP 7: Initial Email Check / prevents mismatched data… |
| SUBJECT LINES | Code | Random subject/opening/body/closing/signature | ONLY CHECK EMAIL NEEDED | Initial Email NOT Sent? SECOND CHECK | ## STEP 8: Subject and Content Generation / generates personalized subject… |
| Initial Email NOT Sent? SECOND CHECK | IF | Final duplicate check (STATUS empty) | SUBJECT LINES | Download file | ## STEP 8: Subject and Content Generation / generates personalized subject… |
| Download file | Google Drive | Download attachment for email | Initial Email NOT Sent? SECOND CHECK | SEND INITIAL EMAIL | ## STEP 9: Send Email / includes subject, opening, signature… |
| SEND INITIAL EMAIL | Gmail | Send the actual email | Download file | UPDATE EMAIL SHEET | ## STEP 9: Send Email / includes subject, opening, signature… |
| UPDATE EMAIL SHEET | Google Sheets | Mark row as sent + date | SEND INITIAL EMAIL | Initial Email | ## STEP 10: Update Sheet / records sent status, date… |
| Sticky Note | Sticky Note | Documentation | — | — | ## 📧 AUTOMATED INITIAL COLD OUTREACH EMAILS WITH SAFETY CONTROLS … |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## STEP 1:  Get Contats From Sheets … |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## STEP 2:  Clean and Validate Emails … |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## STEP 3: Spam Check … |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## STEP 4: Initial Email Status Check … |
| Sticky Note5 | Sticky Note | Documentation | — | — | ## STEP 6: Daily Status Check … |
| Sticky Note6 | Sticky Note | Documentation | — | — | ## STEP 5: Human Like Delay … |
| Sticky Note7 | Sticky Note | Documentation | — | — | ## STEP 8: Subject and Content Generation … |
| Sticky Note8 | Sticky Note | Documentation | — | — | ## STEP 9: Send Email … |
| Sticky Note9 | Sticky Note | Documentation | — | — | ## STEP 10: Update Sheet … |
| Sticky Note10 | Sticky Note | Documentation | — | — | ## STEP 7: Initial Email Check … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name: “Automated Outreach Emails with Gmail & Google Sheets Safety Controls”
   - Set timezone: **Africa/Lagos**

2. **Add trigger**
   - Node: **Schedule Trigger**
   - Configure: run daily at **09:00**
   - Name it: `9am Daily`

3. **Read contacts from Google Sheets**
   - Node: **Google Sheets**
   - Name: `EMAILS SHEET`
   - Auth: **Service Account**
     - Create/choose `googleApi` credential.
     - Share the spreadsheet with the service account email.
   - Select Spreadsheet ID: `1ua7aVmykESCwx6JBHBZHDmNyg7tmTkGnaL4WeoptHSE`
   - Select Sheet tab: `PORTFOLIO`
   - Operation: read rows (default “Get Many” / “Read”)
   - Connect: `9am Daily` → `EMAILS SHEET`

4. **Add initial pacing**
   - Node: **Wait**
   - Name: `10 SECONDS WAIT`
   - Amount: `10` seconds
   - Connect: `EMAILS SHEET` → `10 SECONDS WAIT`

5. **Filter contacts with emails and acceptable status**
   - Node: **Code**
   - Name: `Get Contacts with Emails`
   - Paste logic to keep rows with `EMAIL` and exclude `status` in the invalid list.
   - Connect: `10 SECONDS WAIT` → `Get Contacts with Emails`

6. **Clean and validate email addresses**
   - Node: **Code**
   - Name: `VALID EMAIL`
   - Paste the cleanup/validation code (lowercase/trim, regex validation, blocked domains/keywords, set `valid/blocked/reason`).
   - Connect: `Get Contacts with Emails` → `VALID EMAIL`

7. **Route only valid emails**
   - Node: **IF**
   - Name: `VALID EMAIL?`
   - Condition: Boolean equals `{{$json.valid}}` = `true`
   - Connect: `VALID EMAIL` → `VALID EMAIL?`
   - Connect **true** output → next node

8. **Spam flag check**
   - Node: **IF**
   - Name: `Is SPAM?`
   - Condition: String contains `{{$json.SPAM}}` contains `YES`
   - Connect: `VALID EMAIL? (true)` → `Is SPAM?`
   - Use **false branch** (not spam) for continuation.

9. **Require verification to be present**
   - Node: **IF**
   - Name: `EMAIL VERIFICATION DONE?`
   - Condition: String not empty `{{$json["EMAIL VERIFIED"]}}`
   - Connect: `Is SPAM? (false)` → `EMAIL VERIFICATION DONE?`

10. **Deliverability gate**
   - Node: **IF**
   - Name: `undelivered email?`
   - Condition: String startsWith `{{$json["EMAIL VERIFIED"]}}` starts with `REACHABLE`
   - Connect: `EMAIL VERIFICATION DONE? (true)` → `undelivered email?`

11. **Require name**
   - Node: **IF**
   - Name: `Has Name?`
   - Condition: String not empty `{{$json.Name}}`
   - Connect: `undelivered email? (true)` → `Has Name?`

12. **Prevent sending if already sent (first gate)**
   - Node: **IF**
   - Name: `Initial Email NOT Sent?`
   - Condition: String empty `{{$json.STATUS}}`
   - Connect: `Has Name? (true)` → `Initial Email NOT Sent?`

13. **Batch loop**
   - Node: **Split In Batches**
   - Name: `Initial Email`
   - Set batch size (recommended **1** for safest pacing).
   - Connect: `Initial Email NOT Sent? (true)` → `Initial Email`

14. **Random human delay generator**
   - Node: **Code**
   - Name: `Human wait time1`
   - Generate `delay` (ms) randomly.
   - Connect: `Initial Email` → `Human wait time1`

15. **Wait for randomized time**
   - Node: **Wait**
   - Name: `Wait`
   - Amount expression: `={{ Number($json.delay) / 1000 }}`
   - Connect: `Human wait time1` → `Wait`

16. **Set email type**
   - Node: **Set**
   - Name: `Initial`
   - Add field: `emailType` = `initial` (string), keep other fields.
   - Connect: `Wait` → `Initial`

17. **Enforce weekday + daily quota**
   - Node: **Code**
   - Name: `Check Daily Email Counter`
   - Paste the quota code using `$getWorkflowStaticData('global')`.
   - Connect: `Initial` → `Check Daily Email Counter`

18. **Re-read the sheet (fresh state)**
   - Node: **Google Sheets**
   - Name: `INITIAL EMAIL SHEET`
   - Same spreadsheet + tab as earlier.
   - Connect: `Check Daily Email Counter` → `INITIAL EMAIL SHEET`

19. **Filter to the exact email being processed**
   - Node: **Filter**
   - Name: `ONLY CHECK EMAIL NEEDED`
   - Condition: `{{$json.EMAIL}}` equals `{{ $('Initial Email').item.json.EMAIL }}`
   - Connect: `INITIAL EMAIL SHEET` → `ONLY CHECK EMAIL NEEDED`

20. **Generate subject and email components**
   - Node: **Code**
   - Name: `SUBJECT LINES`
   - Paste the random subject/opening/body/closing/signature code.
   - Connect: `ONLY CHECK EMAIL NEEDED` → `SUBJECT LINES`

21. **Second “not sent” gate**
   - Node: **IF**
   - Name: `Initial Email NOT Sent? SECOND CHECK`
   - Condition: String empty `{{$json.STATUS}}`
   - Connect: `SUBJECT LINES` → `Initial Email NOT Sent? SECOND CHECK`

22. **(Optional) Download attachment from Drive**
   - Node: **Google Drive**
   - Name: `Download file`
   - Operation: **Download**
   - Configure a valid **File ID or Drive URL**.
   - Ensure it outputs binary data (default for download).
   - Connect: `Initial Email NOT Sent? SECOND CHECK (true)` → `Download file`
   - If you do not need attachments, remove this node and connect directly to Gmail send.

23. **Send via Gmail**
   - Node: **Gmail**
   - Name: `SEND INITIAL EMAIL`
   - Credential: `gmailOAuth2` (OAuth2 with Gmail send scope).
   - To: `={{ $json.EMAIL }}`
   - Subject: `={{ $('SUBJECT LINES').item.json.Subject }}`
   - Message body: HTML template using `Opening/Body/Closing/Signature`.
   - Sender name: “Your Name - Highlight”
   - Attachments:
     - Map the downloaded binary property (from `Download file`) into Gmail attachments (must reference the correct binary field name).
   - Connect: `Download file` → `SEND INITIAL EMAIL`

24. **Update sent status in Google Sheets**
   - Node: **Google Sheets**
   - Name: `UPDATE EMAIL SHEET`
   - Operation: **Update**
   - Match column: `EMAIL`
   - Set values:
     - `SENT` = `SENT ✅`
     - `DATE SENT` = `={{ new Date().toISOString().split('T')[0] }}`
     - `EMAIL` = `={{ $('Initial Email NOT Sent? SECOND CHECK').item.json.EMAIL }}`
   - Connect: `SEND INITIAL EMAIL` → `UPDATE EMAIL SHEET`

25. **Loop to next batch**
   - Connect: `UPDATE EMAIL SHEET` → `Initial Email` (so SplitInBatches continues to next row)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AUTOMATED INITIAL COLD OUTREACH EMAILS WITH SAFETY CONTROLS… Setup steps: Connect Google Sheets, Gmail, NocoDB (optional), Review email limits before activating.” | Sticky note overview (top-left) |
| Gmail payload example with n8n expressions (To/Subject/Body/Signature) | Sticky note “GMAIL PAYLOAD” block |
| Steps 1–10 notes describing each phase (Sheets read, validation, spam check, status check, delay, quota, re-check, generation, send, update) | Sticky notes STEP 1 … STEP 10 |

**Important implementation warning (from node logic):** Duplicate-prevention gates check `STATUS` being empty, but the sheet update writes to `SENT`. For reliable idempotency, align these (either gate on `SENT` or update `STATUS`).