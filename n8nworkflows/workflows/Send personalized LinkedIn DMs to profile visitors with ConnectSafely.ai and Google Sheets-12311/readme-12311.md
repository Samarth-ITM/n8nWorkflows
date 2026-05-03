Send personalized LinkedIn DMs to profile visitors with ConnectSafely.ai and Google Sheets

https://n8nworkflows.xyz/workflows/send-personalized-linkedin-dms-to-profile-visitors-with-connectsafely-ai-and-google-sheets-12311


# Send personalized LinkedIn DMs to profile visitors with ConnectSafely.ai and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically fetch LinkedIn profile visitors (last 7 days) via ConnectSafely.ai, deduplicate against a Google Sheet, then send either a personalized DM (for 1st-degree connections) or a connection request (for others). Every action is logged back into Google Sheets, and the workflow loops through all visitors with a wait between actions.

**Target use cases:** Sales, recruiting, founders, community/networking teams that want lightweight, semi-personalized outreach to inbound profile views while preventing repeat messaging.

### Logical blocks
1. **1.1 Scheduling & Visitor Retrieval**: Trigger weekly, call ConnectSafely.ai API, split visitors into items.
2. **1.2 Deduplication**: Look up each visitor in Google Sheets and skip if already contacted.
3. **1.3 Profile Routing & Message Generation**: Extract LinkedIn profile ID, branch by connection degree, generate message text.
4. **1.4 Sending & Tracking + Loop Control**: Send DM or connection request, write/update a row in Google Sheets, wait, then continue batch loop.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Scheduling & Visitor Retrieval
**Overview:** Starts weekly, fetches the visitor list from ConnectSafely.ai for the past 7 days, and transforms the returned array into individual items for processing.

**Nodes involved:**
- Weekly Schedule Trigger
- Fetch Profile Visitors
- Split Visitors Array
- Loop Over Items

#### Node: Weekly Schedule Trigger
- **Type / role:** Schedule Trigger; entry point that runs on an interval.
- **Configuration:** Interval is set to **weeks** (no specific weekday/time shown in JSON; depends on n8n UI defaults).
- **Outputs:** Connects to **Fetch Profile Visitors**.
- **Edge cases / failures:** Workflow may run at an unintended time if timezone isn’t set as expected in n8n instance settings.

#### Node: Fetch Profile Visitors
- **Type / role:** HTTP Request; calls ConnectSafely.ai to retrieve profile visitors.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://api.connectsafely.ai/linkedin/profile/visitors`
  - **Auth:** Generic credential type → **HTTP Bearer Auth** (ConnectSafely.ai API key).
  - **JSON body:** `{"timeRange":"past_7_days","start":0,"fetchAll":true}`
    - `fetchAll:true` indicates pagination handled by API side (or by this endpoint).
- **Inputs:** From **Weekly Schedule Trigger**.
- **Outputs:** To **Split Visitors Array**.
- **Edge cases / failures:**
  - 401/403 if API key invalid/expired.
  - Non-JSON or schema change (missing `visitors` field) will break next node.
  - Rate limits / 429; transient failures need retry logic (not present).

#### Node: Split Visitors Array
- **Type / role:** Split Out; converts `visitors` array into one item per visitor.
- **Configuration:** `fieldToSplitOut = visitors`
- **Inputs:** From **Fetch Profile Visitors**.
- **Outputs:** To **Loop Over Items**.
- **Edge cases / failures:** If `visitors` is missing or not an array, node will output no items or error depending on n8n version.

#### Node: Loop Over Items
- **Type / role:** Split In Batches; iterates over visitors.
- **Configuration:** Batch options not set (defaults apply). With defaults, n8n typically uses batch size 1 unless configured.
- **Connections:**
  - **Main output (index 1)** is used to process each item: goes to **Check if Already Contacted**.
  - **Main output (index 0)** is unused in this workflow JSON.
- **Edge cases / failures:**
  - Misunderstanding outputs: SplitInBatches has distinct outputs (“items” vs “no items”). If connected incorrectly, loop may not execute or may stop early.

---

### Block 1.2 — Deduplication
**Overview:** For each visitor, checks Google Sheets for an existing row matching the visitor’s LinkedIn URL. If found, skip sending and proceed to wait/next visitor.

**Nodes involved:**
- Check if Already Contacted
- Skip if Already Contacted

#### Node: Check if Already Contacted
- **Type / role:** Google Sheets; lookup/filter rows.
- **Configuration choices:**
  - **Document ID:** `YOUR_GOOGLE_SHEET_ID` (must be replaced)
  - **Sheet:** `Sheet1` (`gid=0`)
  - **Filter:** lookup where **Linkedin URL** equals `{{$json.navigationUrl}}`
  - **alwaysOutputData:** true (important for downstream IF evaluation even if empty)
- **Inputs:** From **Loop Over Items** (per visitor item).
- **Outputs:** To **Skip if Already Contacted**.
- **Edge cases / failures:**
  - Auth/permission errors (Google OAuth credential).
  - Column name mismatch (must be exactly `Linkedin URL`).
  - If the sheet contains duplicates, multiple rows may match; downstream logic assumes “not empty” means already contacted.

#### Node: Skip if Already Contacted
- **Type / role:** IF; branches based on whether the lookup returned results.
- **Configuration (interpreted):**
  - Condition uses expression: `{{$json.isEmpty()}}`
  - Operator: boolean “false”
  - Meaning: **If result is NOT empty → already contacted → skip outreach**
- **Connections:**
  - **True branch (index 0):** Goes to **Extract Profile ID from URL** (i.e., proceed to outreach) — note: due to how the condition is written, this is effectively the “not empty” vs “empty” handling; verify in UI after import.
  - **False branch (index 1):** Goes to **Wait Between Messages** (skip and move on).
- **Edge cases / failures:**
  - Potential logic inversion risk: depending on how n8n evaluates `isEmpty()` on the returned structure, the “skip” branch might be reversed. You should test with one known-existing and one new URL.
  - If Google Sheets node returns a different structure (version/operation changes), `.isEmpty()` might not behave as expected.

---

### Block 1.3 — Profile Routing & Message Generation
**Overview:** Builds a ConnectSafely-compatible `profile_id` from the visitor LinkedIn URL, then routes based on connection degree. Generates a randomized message template appropriate for DM vs connection request.

**Nodes involved:**
- Extract Profile ID from URL
- Check if 1st Degree Connection
- Generate DM for Connected User
- Generate Message for New Connection

#### Node: Extract Profile ID from URL
- **Type / role:** Code; parses LinkedIn profile ID from a URL.
- **Key logic:**
  - Reads: `$('Loop Over Items').first().json.navigationUrl`
  - Removes query parameters and trailing slash.
  - Regex: `linkedin\.com\/in\/([^/]+)` → captures the slug after `/in/`.
  - Returns: `{ profile_id }`
- **Inputs:** From **Skip if Already Contacted** (the “proceed” branch).
- **Outputs:** To **Check if 1st Degree Connection**.
- **Edge cases / failures:**
  - URLs not matching `/in/` format (company pages, “/pub/…”, localized paths) will return `profile_id = null`.
  - If `navigationUrl` missing, returns null; downstream HTTP nodes may fail with validation or API error.

#### Node: Check if 1st Degree Connection
- **Type / role:** IF; routes by visitor’s connection degree.
- **Condition:**
  - Left value: `{{ $('Loop Over Items').item.json.connectionDegree }}`
  - Equals `"1st"`
- **True branch:** Generate DM for Connected User
- **False branch:** Generate Message for New Connection
- **Edge cases / failures:**
  - If `connectionDegree` is missing or uses a different label (e.g., `1st_degree`), routing will misclassify everyone to the “new connection” path.
  - Case sensitivity is enabled (strict).

#### Node: Generate DM for Connected User
- **Type / role:** Code; builds a DM for 1st-degree connections.
- **Configuration choices / logic:**
  - Name source: `$('Loop Over Items').first().json.name || 'there'`
  - Randomly selects one message body from a predefined array.
  - Adds footer: `— Your Name` (placeholder to customize).
  - Output: `{ message: finalMessage }`
- **Inputs:** From **Check if 1st Degree Connection** (true).
- **Outputs:** To **Send DM to Connected User**.
- **Edge cases / failures:**
  - If `name` contains unusual whitespace or non-string types, `.trim()` may error (rare, but possible).
  - Message templates include placeholders like `[YOUR PRODUCT]`; if not replaced, outreach quality suffers (not technical failure).

#### Node: Generate Message for New Connection
- **Type / role:** Code; builds a message for connection requests.
- **Same mechanics as above** but with message bodies intended for non-connected users.
- **Outputs:** `{ message }`
- **Inputs:** From **Check if 1st Degree Connection** (false).
- **Outputs:** To **Send Connection Request**.
- **Edge cases / failures:** Same as DM generator.

---

### Block 1.4 — Sending & Tracking + Loop Control
**Overview:** Executes the appropriate ConnectSafely.ai action (DM or connect), writes an “append or update” row in Google Sheets keyed by LinkedIn URL, then waits and continues with the next visitor.

**Nodes involved:**
- Send DM to Connected User
- Send Connection Request
- Log DM Sent to Sheet
- Log Connection Request to Sheet
- Continue to Next Visitor
- Wait Between Messages

#### Node: Send DM to Connected User
- **Type / role:** HTTP Request; sends a LinkedIn message via ConnectSafely.ai.
- **Configuration:**
  - **URL:** `https://api.connectsafely.ai/linkedin/message`
  - **Method:** POST
  - **Auth:** HTTP Bearer Auth
  - **Body params:**
    - `recipientProfileId` = `{{ $('Extract Profile ID from URL').item.json.profile_id }}`
    - `message` = `{{ $json.message }}` (from the preceding Code node)
    - `messageType` = `inmail`
- **Inputs:** From **Generate DM for Connected User**.
- **Outputs:** To **Log DM Sent to Sheet**.
- **Edge cases / failures:**
  - `profile_id` null → API likely rejects.
  - API may reject `messageType=inmail` depending on account/relationship; confirm ConnectSafely requirements.
  - Rate limits or LinkedIn restrictions may cause intermittent failures; no retry/backoff other than the later Wait.

#### Node: Send Connection Request
- **Type / role:** HTTP Request; sends a connection request via ConnectSafely.ai.
- **Configuration:**
  - **URL:** `https://api.connectsafely.ai/linkedin/connect`
  - **Method:** POST
  - **Auth:** HTTP Bearer Auth
  - **Body param:** `profileId` = `{{ $('Extract Profile ID from URL').item.json.profile_id }}`
- **Inputs:** From **Generate Message for New Connection**.
- **Outputs:** To **Log Connection Request to Sheet**.
- **Edge cases / failures:**
  - The generated message is not actually used in this request. If ConnectSafely supports a “note/message” field for connection invites, it is not configured here (functional gap).
  - `profile_id` null → API failure.

#### Node: Log DM Sent to Sheet
- **Type / role:** Google Sheets; append or update tracking row.
- **Configuration:**
  - **Operation:** Append or Update
  - **Matching column:** `Linkedin URL`
  - **Values:**
    - Name = `{{ $('Loop Over Items').item.json.name }}`
    - Linkedin URL = `{{ $('Loop Over Items').item.json.navigationUrl }}`
    - Status = `DONE`
- **Inputs:** From **Send DM to Connected User**.
- **Outputs:** To **Continue to Next Visitor**.
- **Edge cases / failures:**
  - Requires sheet columns: `Name`, `Linkedin URL`, `Status` exactly.
  - If matching column is blank in sheet, “update” behavior can be unpredictable; ensure URLs are always present.

#### Node: Log Connection Request to Sheet
- **Type / role:** Google Sheets; same as above but for connection requests.
- **Inputs:** From **Send Connection Request**.
- **Outputs:** To **Continue to Next Visitor**.
- **Edge cases / failures:** Same as Log DM node.

#### Node: Continue to Next Visitor
- **Type / role:** NoOp; used as a join point before waiting.
- **Inputs:** From both logging nodes.
- **Outputs:** To **Wait Between Messages**.

#### Node: Wait Between Messages
- **Type / role:** Wait; delays before continuing loop (rate limiting / safety).
- **Configuration:** Not specified (defaults). A `webhookId` exists (internal to n8n wait mechanism).
- **Inputs:** From **Continue to Next Visitor** and also from the “skip” branch of **Skip if Already Contacted**.
- **Outputs:** Back to **Loop Over Items** to continue processing.
- **Edge cases / failures:**
  - If duration is not configured, behavior depends on node defaults (may wait indefinitely until resumed manually in some configurations). Verify in UI that a concrete wait time is set (e.g., 30–120 seconds).
  - Large batches + wait can exceed execution time limits depending on hosting plan.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / context | — | — | ## Send Personalized DMs to LinkedIn Profile Visitors … Note: This template requires a ConnectSafely.ai account and API access. ([connectsafely.ai](https://connectsafely.ai)) |
| Section 1 - Fetch Visitors | Sticky Note | Documentation section header | — | — | ## 1. Fetch & Split Visitors Retrieves profile visitors and prepares them for individual processing. |
| Section 2 - Deduplicate | Sticky Note | Documentation section header | — | — | ## 2. Deduplicate Checks if visitor was already contacted to avoid duplicate outreach. |
| Section 3 - Send Messages | Sticky Note | Documentation section header | — | — | ## 3. Route & Send Messages Sends DMs to connected users or connection requests to new prospects. |
| Section 4 - Track Progress | Sticky Note | Documentation section header | — | — | ## 4. Track & Loop Logs outreach to Google Sheets and processes next visitor. |
| Weekly Schedule Trigger | Schedule Trigger | Entry point; weekly run | — | Fetch Profile Visitors | ## 1. Fetch & Split Visitors Retrieves profile visitors and prepares them for individual processing. |
| Fetch Profile Visitors | HTTP Request | Fetch visitors from ConnectSafely.ai | Weekly Schedule Trigger | Split Visitors Array | ## 1. Fetch & Split Visitors Retrieves profile visitors and prepares them for individual processing. |
| Split Visitors Array | Split Out | Explode visitors array into items | Fetch Profile Visitors | Loop Over Items | ## 1. Fetch & Split Visitors Retrieves profile visitors and prepares them for individual processing. |
| Loop Over Items | Split In Batches | Iterate visitor items | Split Visitors Array; Wait Between Messages | Check if Already Contacted | ## 1. Fetch & Split Visitors Retrieves profile visitors and prepares them for individual processing. |
| Check if Already Contacted | Google Sheets | Lookup by LinkedIn URL | Loop Over Items | Skip if Already Contacted | ## 2. Deduplicate Checks if visitor was already contacted to avoid duplicate outreach. |
| Skip if Already Contacted | IF | Branch: proceed vs skip | Check if Already Contacted | Extract Profile ID from URL; Wait Between Messages | ## 2. Deduplicate Checks if visitor was already contacted to avoid duplicate outreach. |
| Extract Profile ID from URL | Code | Parse LinkedIn profile slug to profile_id | Skip if Already Contacted | Check if 1st Degree Connection | ## 3. Route & Send Messages Sends DMs to connected users or connection requests to new prospects. |
| Check if 1st Degree Connection | IF | Route by connection degree | Extract Profile ID from URL | Generate DM for Connected User; Generate Message for New Connection | ## 3. Route & Send Messages Sends DMs to connected users or connection requests to new prospects. |
| Generate DM for Connected User | Code | Build randomized DM text | Check if 1st Degree Connection | Send DM to Connected User | ## 3. Route & Send Messages Sends DMs to connected users or connection requests to new prospects. |
| Send DM to Connected User | HTTP Request | Send LinkedIn DM via ConnectSafely.ai | Generate DM for Connected User | Log DM Sent to Sheet | ## 3. Route & Send Messages Sends DMs to connected users or connection requests to new prospects. |
| Generate Message for New Connection | Code | Build randomized connect text | Check if 1st Degree Connection | Send Connection Request | ## 3. Route & Send Messages Sends DMs to connected users or connection requests to new prospects. |
| Send Connection Request | HTTP Request | Send LinkedIn connection request | Generate Message for New Connection | Log Connection Request to Sheet | ## 3. Route & Send Messages Sends DMs to connected users or connection requests to new prospects. |
| Log DM Sent to Sheet | Google Sheets | Track DM outreach (append/update) | Send DM to Connected User | Continue to Next Visitor | ## 4. Track & Loop Logs outreach to Google Sheets and processes next visitor. |
| Log Connection Request to Sheet | Google Sheets | Track connect outreach (append/update) | Send Connection Request | Continue to Next Visitor | ## 4. Track & Loop Logs outreach to Google Sheets and processes next visitor. |
| Continue to Next Visitor | NoOp | Merge paths before waiting | Log DM Sent to Sheet; Log Connection Request to Sheet | Wait Between Messages | ## 4. Track & Loop Logs outreach to Google Sheets and processes next visitor. |
| Wait Between Messages | Wait | Throttle + resume loop | Continue to Next Visitor; Skip if Already Contacted | Loop Over Items | ## 4. Track & Loop Logs outreach to Google Sheets and processes next visitor. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Schedule Trigger** node:
   - Interval: **Weekly** (field: `weeks`).
   - Ensure timezone is correct in n8n settings.
3. **Add HTTP Request** node named **Fetch Profile Visitors**:
   - Method: **POST**
   - URL: `https://api.connectsafely.ai/linkedin/profile/visitors`
   - Authentication: **HTTP Bearer Auth** (Generic credential).
     - Create credential: paste your ConnectSafely.ai API key as bearer token.
   - Body: JSON
     - `timeRange`: `past_7_days`
     - `start`: `0`
     - `fetchAll`: `true`
4. **Connect** Schedule Trigger → Fetch Profile Visitors.
5. **Add Split Out** node named **Split Visitors Array**:
   - Field to split out: `visitors`
6. **Connect** Fetch Profile Visitors → Split Visitors Array.
7. **Add Split In Batches** node named **Loop Over Items**:
   - Use default batch size (or set to 1 explicitly).
8. **Connect** Split Visitors Array → Loop Over Items.
9. **Add Google Sheets** node named **Check if Already Contacted**:
   - Credential: Google Sheets OAuth2.
   - Document: set to your Sheet ID.
   - Sheet: select `Sheet1` (or your sheet tab).
   - Add a filter:
     - Lookup column: `Linkedin URL`
     - Lookup value: `{{$json.navigationUrl}}`
   - Enable **Always Output Data** (so downstream IF can run even when no rows are found).
10. **Connect** Loop Over Items (items output used for iteration) → Check if Already Contacted.
11. **Add IF** node named **Skip if Already Contacted**:
   - Condition uses expression on the Google Sheets output:
     - Use an “is empty” style check appropriate for your returned data.
     - Match the current workflow’s intent: **If found in sheet → skip**, else proceed.
   - Connect:
     - “Proceed” output → next step (Extract Profile ID from URL)
     - “Skip” output → Wait Between Messages
12. **Add Code** node named **Extract Profile ID from URL**:
   - Paste logic to parse `navigationUrl` and return `{ profile_id }`.
   - Ensure it reads from the current visitor item (or reference `Loop Over Items` item, as in the workflow).
13. **Connect** Skip if Already Contacted (“proceed”) → Extract Profile ID from URL.
14. **Add IF** node named **Check if 1st Degree Connection**:
   - Condition: `{{ $('Loop Over Items').item.json.connectionDegree }}` equals `1st`.
   - True: connected → DM path
   - False: not connected → connect path
15. **Connect** Extract Profile ID from URL → Check if 1st Degree Connection.
16. **Add Code** node **Generate DM for Connected User**:
   - Build `message` with randomized template + footer.
   - Return `{ message }`.
17. **Add HTTP Request** node **Send DM to Connected User**:
   - POST `https://api.connectsafely.ai/linkedin/message`
   - Bearer auth credential (same ConnectSafely credential)
   - Body parameters:
     - `recipientProfileId`: `{{ $('Extract Profile ID from URL').item.json.profile_id }}`
     - `message`: `{{ $json.message }}`
     - `messageType`: `inmail`
18. **Connect** Check if 1st Degree Connection (true) → Generate DM for Connected User → Send DM to Connected User.
19. **Add Code** node **Generate Message for New Connection**:
   - Build `message` similarly (note: current workflow does not actually send this text to the connect endpoint).
20. **Add HTTP Request** node **Send Connection Request**:
   - POST `https://api.connectsafely.ai/linkedin/connect`
   - Bearer auth credential
   - Body parameters:
     - `profileId`: `{{ $('Extract Profile ID from URL').item.json.profile_id }}`
21. **Connect** Check if 1st Degree Connection (false) → Generate Message for New Connection → Send Connection Request.
22. **Prepare Google Sheet** with columns exactly:
   - `Name`, `Linkedin URL`, `Status`
23. **Add Google Sheets** node **Log DM Sent to Sheet**:
   - Operation: **Append or Update**
   - Match on column: `Linkedin URL`
   - Values:
     - Name: `{{ $('Loop Over Items').item.json.name }}`
     - Linkedin URL: `{{ $('Loop Over Items').item.json.navigationUrl }}`
     - Status: `DONE`
24. **Add Google Sheets** node **Log Connection Request to Sheet** with identical configuration.
25. **Connect**:
   - Send DM to Connected User → Log DM Sent to Sheet
   - Send Connection Request → Log Connection Request to Sheet
26. **Add NoOp** node **Continue to Next Visitor** and connect both log nodes into it.
27. **Add Wait** node **Wait Between Messages**:
   - Set a concrete wait duration (recommended) to throttle requests.
28. **Connect** Continue to Next Visitor → Wait Between Messages.
29. **Connect loop-back:** Wait Between Messages → Loop Over Items (so the next visitor is processed).
30. **Also connect skip path:** Skip if Already Contacted (“skip”) → Wait Between Messages.
31. **Test run** with a small visitor set:
   - Confirm dedup logic branches correctly.
   - Confirm `profile_id` extraction works for your visitor URLs.
   - Confirm Google Sheets rows are created/updated as expected.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get your ConnectSafely.ai API key and configure HTTP Bearer Auth in n8n. | https://connectsafely.ai |
| Google Sheet must contain columns: `Name`, `Linkedin URL`, `Status` and the workflow matches on `Linkedin URL`. | Setup requirement from sticky note |
| Customize message templates in the Code nodes: “Generate DM for Connected User” and “Generate Message for New Connection”. | Message personalization |
| Visitor lookback window is controlled in “Fetch Profile Visitors” body (`timeRange: past_7_days`). | Adjust outreach window |
| Connection request path currently generates a message but does not send it to the connect endpoint (no “note/message” field configured). | Functional gap to review depending on ConnectSafely API capabilities |