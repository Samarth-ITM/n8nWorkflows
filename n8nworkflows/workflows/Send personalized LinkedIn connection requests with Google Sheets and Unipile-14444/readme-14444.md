Send personalized LinkedIn connection requests with Google Sheets and Unipile

https://n8nworkflows.xyz/workflows/send-personalized-linkedin-connection-requests-with-google-sheets-and-unipile-14444


# Send personalized LinkedIn connection requests with Google Sheets and Unipile

# 1. Workflow Overview

This workflow automates personalized LinkedIn connection requests using lead data stored in Google Sheets and the Unipile API as the LinkedIn integration layer. It is designed for controlled outbound prospecting: it reads leads from a spreadsheet, skips rows already processed, extracts the LinkedIn username from each profile URL, sends a connection request with a personalized note, and writes the invitation result back into the sheet.

Typical use cases:
- Sales outreach to a curated lead list
- Founder-led prospecting
- Recruiter or partnership outreach
- Low-volume, trackable LinkedIn automation with spreadsheet-based control

## 1.1 Scheduled Input Reception
The workflow starts on a schedule and reads lead rows from a Google Sheet.

## 1.2 Lead Filtering and Volume Control
It removes leads already marked as invited and limits how many leads are processed per run.

## 1.3 Data Preparation
It parses each LinkedIn profile URL to extract the LinkedIn username, then assembles the fields and API settings required downstream.

## 1.4 Request Execution Loop
It iterates over leads one by one, resolves the LinkedIn internal provider ID through Unipile, sends the invitation, and waits between requests.

## 1.5 Tracking and Sheet Update
After each successful invitation, it records the invitation ID and status back into Google Sheets so future runs skip the same lead.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

### Overview
This block triggers the workflow automatically and retrieves the lead list from Google Sheets. It forms the ingestion layer for the entire process.

### Nodes Involved
- Schedule Trigger
- Get Leads

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger` — entry-point node that launches the workflow on a recurring schedule.
- **Configuration choices:** Uses the interval-based scheduling rule. In the exported JSON, the interval object is present but not visibly customized, so the exact schedule should be confirmed in the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none, this is a trigger node  
  - Output: `Get Leads`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Misconfigured schedule causing too frequent runs
  - Overlapping runs if execution time exceeds trigger cadence
  - Disabled workflow preventing automatic execution
- **Sub-workflow reference:** None.

#### Get Leads
- **Type and technical role:** `n8n-nodes-base.googleSheets` — reads lead rows from a Google Sheet.
- **Configuration choices:**
  - Uses a Google Sheets OAuth2 credential
  - Targets spreadsheet ID `1IM0qnZlhYK7Pbj__xwyPgEprW-5-P3RBnen1xGZjSSk`
  - Reads from sheet `gid=0`, labeled as `leads`
  - Default operation appears to be row retrieval
- **Key expressions or variables used:** None in the configured fields shown.
- **Input and output connections:**  
  - Input: `Schedule Trigger`  
  - Output: `Filter`
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - OAuth2 credential expiration or insufficient sheet permissions
  - Wrong document ID or sheet selection
  - Missing expected columns such as `linkedin_url`, `connection_note`, `connection_request_status`, `row_number`
  - Empty sheet resulting in no downstream processing
- **Sub-workflow reference:** None.

---

## 2.2 Lead Filtering and Volume Control

### Overview
This block removes already-contacted leads and caps the number of connection requests sent in a single run. It is the main safety mechanism for avoiding duplicate invites and excessive outreach volume.

### Nodes Involved
- Filter
- Limit Connection Request

### Node Details

#### Filter
- **Type and technical role:** `n8n-nodes-base.filter` — conditionally passes only rows that have not already been marked as invited.
- **Configuration choices:**
  - Uses a single string condition
  - Keeps items where `connection_request_status != UserInvitationSent`
  - Strict type validation is enabled
  - Case-sensitive comparison is enabled
- **Key expressions or variables used:**
  - `{{ $json.connection_request_status }}`
- **Input and output connections:**  
  - Input: `Get Leads`  
  - Output: `Limit Connection Request`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Rows with alternate statuses such as `Sent`, `Pending`, lowercase variants, or trailing spaces will still pass through
  - Empty/null status values will pass, which is intended
  - If the Google Sheet column is missing, expression behavior may be inconsistent depending on data shape
- **Sub-workflow reference:** None.

#### Limit Connection Request
- **Type and technical role:** `n8n-nodes-base.limit` — restricts the number of items processed in each workflow run.
- **Configuration choices:**
  - `maxItems = 10`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Filter`  
  - Output: `Get Linkedin Username`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If fewer than 10 leads match, all are processed
  - If more than 10 match, remaining rows are deferred to future runs
  - An overly high limit may increase LinkedIn restriction risk
- **Sub-workflow reference:** None.

---

## 2.3 Data Preparation

### Overview
This block converts each selected lead into the exact structure required for Unipile API calls. It extracts the LinkedIn username from the profile URL and combines lead fields with Unipile connection settings.

### Nodes Involved
- Get Linkedin Username
- Data Arrangement

### Node Details

#### Get Linkedin Username
- **Type and technical role:** `n8n-nodes-base.code` — parses a LinkedIn profile URL and extracts the `/in/{username}` segment.
- **Configuration choices:**
  - JavaScript code reads the first input item’s `linkedin_url`
  - Removes a trailing slash
  - Splits on `/in/`, then extracts the next path segment
  - Returns a single JSON object with `username`
- **Key expressions or variables used:**
  - Reads: `$input.first().json.linkedin_url`
  - Returns: `{ username: username }`
- **Input and output connections:**  
  - Input: `Limit Connection Request`  
  - Output: `Data Arrangement`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `linkedin_url` is missing, the code may throw because `replace()` is called on an undefined value
  - URLs not matching `/in/...` return `null`
  - Company pages, Sales Navigator URLs, or other LinkedIn URL formats are not handled
  - Query strings or unusual URL variants may produce malformed usernames
- **Sub-workflow reference:** None.

#### Data Arrangement
- **Type and technical role:** `n8n-nodes-base.set` — creates a normalized payload containing lead data and Unipile configuration values.
- **Configuration choices:**
  - Assigns:
    - `username` from previous node
    - `row_number` from `Filter`
    - `first_name` from `Filter`
    - `connection_note` from `Filter`
    - `unipile_DSN` as static text `unipile-DSN`
    - `unipile_api_key` as static text `your-api-key-here`
    - `unipile_account_ID` as static text `your-unipile-linkedin-account-id-here`
- **Key expressions or variables used:**
  - `{{ $json.username }}`
  - `{{ $('Filter').item.json.row_number }}`
  - `{{ $('Filter').item.json.first_name }}`
  - `{{ $('Filter').item.json.connection_note }}`
- **Input and output connections:**  
  - Input: `Get Linkedin Username`  
  - Output: `Loop Over Items`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Placeholder Unipile values must be replaced before production use
  - If paired-item lineage is broken, references like `$('Filter').item.json...` may fail or map incorrectly
  - Missing `row_number` prevents correct sheet updates later
  - Missing `connection_note` may result in blank invitation messages
- **Sub-workflow reference:** None.

---

## 2.4 Request Execution Loop

### Overview
This block processes each lead individually. It looks up the LinkedIn provider ID via Unipile, sends the connection request with the personalized note, then pauses before proceeding to the next item.

### Nodes Involved
- Loop Over Items
- Get Linkedin Provider ID
- Send Connection Request
- Wait

### Node Details

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches` — iterates through prepared lead items one at a time.
- **Configuration choices:**
  - Used as a loop controller with default options
  - Output 1 continues the loop body for the current item
  - The `Wait` node reconnects to this node to request the next item
- **Key expressions or variables used:** Downstream nodes reference this node using `$('Loop Over Items').item...`
- **Input and output connections:**  
  - Input: `Data Arrangement` and loop-back from `Wait`
  - Output:
    - Batch output to `Get Linkedin Provider ID`
    - Completion output unused
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If no items arrive, nothing is processed
  - Miswiring the loop can cause only one item to run or create an endless loop
  - Batch size defaults should be verified in the UI if changed
- **Sub-workflow reference:** None.

#### Get Linkedin Provider ID
- **Type and technical role:** `n8n-nodes-base.httpRequest` — calls Unipile to resolve a LinkedIn username into a `provider_id`.
- **Configuration choices:**
  - HTTP GET to:
    - `https://{{ $json.unipile_DSN }}/api/v1/users/{{ $json.username }}`
  - Query parameter:
    - `account_id = {{ $json.unipile_account_ID }}`
  - Headers:
    - `X-API-KEY = {{ $json.unipile_api_key }}`
    - `accept = application/json`
- **Key expressions or variables used:**
  - `{{ $json.unipile_DSN }}`
  - `{{ $json.username }}`
  - `{{ $json.unipile_account_ID }}`
  - `{{ $json.unipile_api_key }}`
- **Input and output connections:**  
  - Input: `Loop Over Items`  
  - Output: `Send Connection Request`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid DSN or API key causing 401/403 responses
  - Invalid or null username causing 404 or validation errors
  - Unipile rate limits or temporary service errors
  - Network/DNS/TLS failures
  - Response may not include `provider_id`, which would break the next node
- **Sub-workflow reference:** None.

#### Send Connection Request
- **Type and technical role:** `n8n-nodes-base.httpRequest` — sends the LinkedIn connection invitation through Unipile.
- **Configuration choices:**
  - HTTP POST to:
    - `https://{{ $('Loop Over Items').item.json.unipile_DSN }}/api/v1/users/invite`
  - Body parameters:
    - `provider_id = {{ $json.provider_id }}`
    - `account_id = {{ $('Data Arrangement').item.json.unipile_account_ID }}`
    - `message = {{ $('Loop Over Items').item.json.connection_note }}`
  - Headers:
    - `X-API-KEY = {{ $('Loop Over Items').item.json.unipile_api_key }}`
    - `accept = application/json`
- **Key expressions or variables used:**
  - `{{ $json.provider_id }}`
  - `{{ $('Data Arrangement').item.json.unipile_account_ID }}`
  - `{{ $('Loop Over Items').item.json.connection_note }}`
  - `{{ $('Loop Over Items').item.json.unipile_api_key }}`
- **Input and output connections:**  
  - Input: `Get Linkedin Provider ID`  
  - Output: `Update Connection Request Status and ID`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Missing `provider_id`
  - Empty or overlong invitation message
  - LinkedIn/Unipile restrictions on pending invitations or duplicate requests
  - Auth failures due to invalid key or account ID
  - HTTP 4xx/5xx responses not explicitly handled
- **Sub-workflow reference:** None.

#### Wait
- **Type and technical role:** `n8n-nodes-base.wait` — pauses execution between invitation sends, then resumes the loop.
- **Configuration choices:**
  - Unit is set to `minutes`
  - No explicit duration value is visible in the export, so this should be verified manually in the editor
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Update Connection Request Status and ID`
  - Output: loops back to `Loop Over Items`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Missing duration can result in unexpected behavior depending on UI defaults
  - Very short waits increase risk of LinkedIn restrictions
  - Long waits may cause backlog buildup
  - Execution resumption depends on n8n wait/resume infrastructure functioning correctly
- **Sub-workflow reference:** None.

---

## 2.5 Tracking and Sheet Update

### Overview
This block writes the result of each sent invitation back into the source spreadsheet. It ensures later runs can skip already-contacted leads and preserves the returned invitation ID for future follow-up logic.

### Nodes Involved
- Update Connection Request Status and ID

### Node Details

#### Update Connection Request Status and ID
- **Type and technical role:** `n8n-nodes-base.googleSheets` — updates an existing row in the Google Sheet based on a matching `row_number`.
- **Configuration choices:**
  - Operation: `update`
  - Spreadsheet: same Google Sheet as input
  - Sheet: `gid=0` / `leads`
  - Matching column: `row_number`
  - Writes:
    - `row_number = {{ $('Loop Over Items').item.json.row_number }}`
    - `invitation_id = {{ $json.invitation_id }}`
    - `connection_request_status = {{ $json.object }}`
- **Key expressions or variables used:**
  - `{{ $('Loop Over Items').item.json.row_number }}`
  - `{{ $json.invitation_id }}`
  - `{{ $json.object }}`
- **Input and output connections:**  
  - Input: `Send Connection Request`
  - Output: `Wait`
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - If `row_number` is missing or not unique, updates may fail or affect the wrong row
  - If Unipile response changes and `object` is absent, status may be blank
  - Google Sheets auth or permission issues
  - Data type mismatches if `row_number` formatting differs from source sheet
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts the workflow on a recurring schedule |  | Get Leads | ## 📥 LEAD COLLECTION & FILTERING<br><br>**Pulls leads from Google Sheets and prepares them for outreach.**<br><br>### 🔄 Flow:<br>1️⃣ **Schedule Trigger** — Runs on your set schedule<br>2️⃣ **Get Leads** — Reads all rows from the 'leads' sheet<br>3️⃣ **Filter** — Keeps only leads where `connection_request_status` is empty (not yet contacted)<br>4️⃣ **Limit** — Caps the batch size per run<br><br>### 💡 Key Points:<br>- Leads already sent a request are automatically skipped<br>- Adjust the Limit node to control daily volume<br>- Recommended limit: **10–15 per run** to stay safe<br><br>### 📊 Sheet Columns Needed:<br>`linkedin_url` \| `connection_note` \| `connection_request_status` |
| Get Leads | Google Sheets | Reads lead rows from the spreadsheet | Schedule Trigger | Filter | ## 📥 LEAD COLLECTION & FILTERING<br><br>**Pulls leads from Google Sheets and prepares them for outreach.**<br><br>### 🔄 Flow:<br>1️⃣ **Schedule Trigger** — Runs on your set schedule<br>2️⃣ **Get Leads** — Reads all rows from the 'leads' sheet<br>3️⃣ **Filter** — Keeps only leads where `connection_request_status` is empty (not yet contacted)<br>4️⃣ **Limit** — Caps the batch size per run<br><br>### 💡 Key Points:<br>- Leads already sent a request are automatically skipped<br>- Adjust the Limit node to control daily volume<br>- Recommended limit: **10–15 per run** to stay safe<br><br>### 📊 Sheet Columns Needed:<br>`linkedin_url` \| `connection_note` \| `connection_request_status` |
| Filter | Filter | Excludes leads already marked as invited | Get Leads | Limit Connection Request | ## 📥 LEAD COLLECTION & FILTERING<br><br>**Pulls leads from Google Sheets and prepares them for outreach.**<br><br>### 🔄 Flow:<br>1️⃣ **Schedule Trigger** — Runs on your set schedule<br>2️⃣ **Get Leads** — Reads all rows from the 'leads' sheet<br>3️⃣ **Filter** — Keeps only leads where `connection_request_status` is empty (not yet contacted)<br>4️⃣ **Limit** — Caps the batch size per run<br><br>### 💡 Key Points:<br>- Leads already sent a request are automatically skipped<br>- Adjust the Limit node to control daily volume<br>- Recommended limit: **10–15 per run** to stay safe<br><br>### 📊 Sheet Columns Needed:<br>`linkedin_url` \| `connection_note` \| `connection_request_status` |
| Limit Connection Request | Limit | Restricts how many leads are processed per run | Filter | Get Linkedin Username | ## 📥 LEAD COLLECTION & FILTERING<br><br>**Pulls leads from Google Sheets and prepares them for outreach.**<br><br>### 🔄 Flow:<br>1️⃣ **Schedule Trigger** — Runs on your set schedule<br>2️⃣ **Get Leads** — Reads all rows from the 'leads' sheet<br>3️⃣ **Filter** — Keeps only leads where `connection_request_status` is empty (not yet contacted)<br>4️⃣ **Limit** — Caps the batch size per run<br><br>### 💡 Key Points:<br>- Leads already sent a request are automatically skipped<br>- Adjust the Limit node to control daily volume<br>- Recommended limit: **10–15 per run** to stay safe<br><br>### 📊 Sheet Columns Needed:<br>`linkedin_url` \| `connection_note` \| `connection_request_status` |
| Get Linkedin Username | Code | Extracts the LinkedIn username from the profile URL | Limit Connection Request | Data Arrangement | ## ⚙️ DATA PREPARATION<br><br>**Extracts the LinkedIn username and arranges all data needed for the API calls.**<br><br>### 🔄 Flow:<br>1️⃣ **Get LinkedIn Username** — Parses the username from the LinkedIn URL<br>   - Input: `https://linkedin.com/in/john-doe/`<br>   - Output: `john-doe`<br><br>2️⃣ **Data Arrangement** — Bundles all required fields into one clean item:<br>   - `username` → for API lookup<br>   - `row_number` → to update the correct sheet row later<br>   - `connection_note` → personalized message per lead<br>   - `unipile_DSN`, `unipile_api_key`, `unipile_account_ID` → your API credentials<br><br>### ⚙️ Setup — Fill In the Data Arrangement Node:<br>- `unipile-DSN` → Your Unipile DSN<br>- `your-api-key-here` → Your Unipile API Key<br>- `your-unipile-linkedin-account-id-here` → Your LinkedIn Account ID |
| Data Arrangement | Set | Normalizes lead data and injects Unipile settings | Get Linkedin Username | Loop Over Items | ## ⚙️ DATA PREPARATION<br><br>**Extracts the LinkedIn username and arranges all data needed for the API calls.**<br><br>### 🔄 Flow:<br>1️⃣ **Get LinkedIn Username** — Parses the username from the LinkedIn URL<br>   - Input: `https://linkedin.com/in/john-doe/`<br>   - Output: `john-doe`<br><br>2️⃣ **Data Arrangement** — Bundles all required fields into one clean item:<br>   - `username` → for API lookup<br>   - `row_number` → to update the correct sheet row later<br>   - `connection_note` → personalized message per lead<br>   - `unipile_DSN`, `unipile_api_key`, `unipile_account_ID` → your API credentials<br><br>### ⚙️ Setup — Fill In the Data Arrangement Node:<br>- `unipile-DSN` → Your Unipile DSN<br>- `your-api-key-here` → Your Unipile API Key<br>- `your-unipile-linkedin-account-id-here` → Your LinkedIn Account ID |
| Loop Over Items | Split In Batches | Iterates through each lead one at a time | Data Arrangement, Wait | Get Linkedin Provider ID | ## 📨 CONNECTION REQUEST LOOP<br><br>**Sends connection requests one by one with a delay between each.**<br><br>### 🔄 Flow:<br>1️⃣ **Loop Over Items** — Processes leads one at a time<br>2️⃣ **Get LinkedIn Provider ID** — Calls Unipile API to resolve the username to LinkedIn's internal `provider_id`<br>3️⃣ **Send Connection Request** — Posts the invite with your personalized note<br>4️⃣ **Update Status & ID** — Writes `UserInvitationSent` + `invitation_id` back to Google Sheets<br>5️⃣ **Wait** — Pauses between requests to mimic human behavior<br>🔁 Loop repeats until all leads in the batch are processed<br><br>### ⚠️ LinkedIn Safety Tips:<br>- Set Wait to **3–5 minutes** for safest operation<br>- Keep daily total under **20 requests**<br>- Use a warmed-up LinkedIn account<br>- Vary your connection notes per lead |
| Get Linkedin Provider ID | HTTP Request | Resolves LinkedIn username into a Unipile/LinkedIn provider ID | Loop Over Items | Send Connection Request | ## 📨 CONNECTION REQUEST LOOP<br><br>**Sends connection requests one by one with a delay between each.**<br><br>### 🔄 Flow:<br>1️⃣ **Loop Over Items** — Processes leads one at a time<br>2️⃣ **Get LinkedIn Provider ID** — Calls Unipile API to resolve the username to LinkedIn's internal `provider_id`<br>3️⃣ **Send Connection Request** — Posts the invite with your personalized note<br>4️⃣ **Update Status & ID** — Writes `UserInvitationSent` + `invitation_id` back to Google Sheets<br>5️⃣ **Wait** — Pauses between requests to mimic human behavior<br>🔁 Loop repeats until all leads in the batch are processed<br><br>### ⚠️ LinkedIn Safety Tips:<br>- Set Wait to **3–5 minutes** for safest operation<br>- Keep daily total under **20 requests**<br>- Use a warmed-up LinkedIn account<br>- Vary your connection notes per lead |
| Send Connection Request | HTTP Request | Sends the personalized LinkedIn invitation | Get Linkedin Provider ID | Update Connection Request Status and ID | ## 📨 CONNECTION REQUEST LOOP<br><br>**Sends connection requests one by one with a delay between each.**<br><br>### 🔄 Flow:<br>1️⃣ **Loop Over Items** — Processes leads one at a time<br>2️⃣ **Get LinkedIn Provider ID** — Calls Unipile API to resolve the username to LinkedIn's internal `provider_id`<br>3️⃣ **Send Connection Request** — Posts the invite with your personalized note<br>4️⃣ **Update Status & ID** — Writes `UserInvitationSent` + `invitation_id` back to Google Sheets<br>5️⃣ **Wait** — Pauses between requests to mimic human behavior<br>🔁 Loop repeats until all leads in the batch are processed<br><br>### ⚠️ LinkedIn Safety Tips:<br>- Set Wait to **3–5 minutes** for safest operation<br>- Keep daily total under **20 requests**<br>- Use a warmed-up LinkedIn account<br>- Vary your connection notes per lead |
| Update Connection Request Status and ID | Google Sheets | Writes invitation result and status back into the sheet | Send Connection Request | Wait | ## ✅ TRACKING & STATUS UPDATE<br><br>**Keeps your Google Sheet up to date after every request.**<br><br>### 📝 What Gets Written Back:<br>\| Column \| Value \|<br>\|---\|---\|<br>\| `connection_request_status: ` \| `UserInvitationSent` \|<br>\| `invitation_id: ` \| Unique ID from Unipile \|<br><br>### 🎯 Why This Matters:<br>- The **Filter node** uses `connection_request_status` to skip leads on future runs<br>- `invitation_id` can be used for follow-up message workflows<br>- Matched by `row_number` — always updates the exact correct row<br><br>### 💡 Build On This:<br>You can extend this workflow to send follow-up messages to accepted connections using the stored `invitation_id`. |
| Wait | Wait | Delays between invitation requests and resumes the loop | Update Connection Request Status and ID | Loop Over Items | ## 📨 CONNECTION REQUEST LOOP<br><br>**Sends connection requests one by one with a delay between each.**<br><br>### 🔄 Flow:<br>1️⃣ **Loop Over Items** — Processes leads one at a time<br>2️⃣ **Get LinkedIn Provider ID** — Calls Unipile API to resolve the username to LinkedIn's internal `provider_id`<br>3️⃣ **Send Connection Request** — Posts the invite with your personalized note<br>4️⃣ **Update Status & ID** — Writes `UserInvitationSent` + `invitation_id` back to Google Sheets<br>5️⃣ **Wait** — Pauses between requests to mimic human behavior<br>🔁 Loop repeats until all leads in the batch are processed<br><br>### ⚠️ LinkedIn Safety Tips:<br>- Set Wait to **3–5 minutes** for safest operation<br>- Keep daily total under **20 requests**<br>- Use a warmed-up LinkedIn account<br>- Vary your connection notes per lead |
| 📖 Setup Instructions | Sticky Note | Workspace documentation and setup guidance |  |  | # 🔗 LinkedIn Auto Connection Request<br>**Automated LinkedIn outreach — personalized, safe, and trackable.**<br><br>### 🎯 What This Workflow Does:<br>✅ Reads leads from Google Sheets<br>✅ Filters out already-contacted leads<br>✅ Sends personalized connection requests via LinkedIn<br>✅ Tracks every invitation back in your sheet<br>✅ Adds delays to avoid LinkedIn restrictions<br><br>### 📋 Setup Required:<br><br>**1️⃣ Google Sheet**<br>Make sure your sheet has these columns:<br>`first_name` \| `last_name` \| `linkedin_url` \| `connection_note` \| `connection_request_status` \| `invitation_id` \| `row_number`<br><br>**2️⃣ Unipile Account**<br>- Sign up at: https://unipile.com<br>- Connect your LinkedIn account<br>- Get your: API Key, DSN, and Account ID<br>- Fill these in the **Data Arrangement** node<br><br>**3️⃣ Schedule**<br>- Set your preferred run frequency in the Schedule Trigger node<br>- Recommended: every 4–8 hours for safe outreach<br><br>**4️⃣ Set Your Daily Limit**<br>- Open the **Limit Connection Request** node<br>- Set max items to 10–15 per run (safe range)<br><br>### 🚀 Quick Start:<br>1. Complete setup steps above<br>2. Activate the workflow<br>3. Add leads to your Google Sheet<br>4. Watch connections go out automatically! |
| 📥 Lead Collection Path | Sticky Note | Visual explanation of the lead ingestion and filtering section |  |  | ## 📥 LEAD COLLECTION & FILTERING<br><br>**Pulls leads from Google Sheets and prepares them for outreach.**<br><br>### 🔄 Flow:<br>1️⃣ **Schedule Trigger** — Runs on your set schedule<br>2️⃣ **Get Leads** — Reads all rows from the 'leads' sheet<br>3️⃣ **Filter** — Keeps only leads where `connection_request_status` is empty (not yet contacted)<br>4️⃣ **Limit** — Caps the batch size per run<br><br>### 💡 Key Points:<br>- Leads already sent a request are automatically skipped<br>- Adjust the Limit node to control daily volume<br>- Recommended limit: **10–15 per run** to stay safe<br><br>### 📊 Sheet Columns Needed:<br>`linkedin_url` \| `connection_note` \| `connection_request_status` |
| ⚙️ Data Preparation | Sticky Note | Visual explanation of username extraction and payload shaping |  |  | ## ⚙️ DATA PREPARATION<br><br>**Extracts the LinkedIn username and arranges all data needed for the API calls.**<br><br>### 🔄 Flow:<br>1️⃣ **Get LinkedIn Username** — Parses the username from the LinkedIn URL<br>   - Input: `https://linkedin.com/in/john-doe/`<br>   - Output: `john-doe`<br><br>2️⃣ **Data Arrangement** — Bundles all required fields into one clean item:<br>   - `username` → for API lookup<br>   - `row_number` → to update the correct sheet row later<br>   - `connection_note` → personalized message per lead<br>   - `unipile_DSN`, `unipile_api_key`, `unipile_account_ID` → your API credentials<br><br>### ⚙️ Setup — Fill In the Data Arrangement Node:<br>- `unipile-DSN` → Your Unipile DSN<br>- `your-api-key-here` → Your Unipile API Key<br>- `your-unipile-linkedin-account-id-here` → Your LinkedIn Account ID |
| 📨 Connection Request Loop | Sticky Note | Visual explanation of the send-and-wait loop |  |  | ## 📨 CONNECTION REQUEST LOOP<br><br>**Sends connection requests one by one with a delay between each.**<br><br>### 🔄 Flow:<br>1️⃣ **Loop Over Items** — Processes leads one at a time<br>2️⃣ **Get LinkedIn Provider ID** — Calls Unipile API to resolve the username to LinkedIn's internal `provider_id`<br>3️⃣ **Send Connection Request** — Posts the invite with your personalized note<br>4️⃣ **Update Status & ID** — Writes `UserInvitationSent` + `invitation_id` back to Google Sheets<br>5️⃣ **Wait** — Pauses between requests to mimic human behavior<br>🔁 Loop repeats until all leads in the batch are processed<br><br>### ⚠️ LinkedIn Safety Tips:<br>- Set Wait to **3–5 minutes** for safest operation<br>- Keep daily total under **20 requests**<br>- Use a warmed-up LinkedIn account<br>- Vary your connection notes per lead |
| ✅ Tracking & Updates | Sticky Note | Visual explanation of sheet write-back and tracking |  |  | ## ✅ TRACKING & STATUS UPDATE<br><br>**Keeps your Google Sheet up to date after every request.**<br><br>### 📝 What Gets Written Back:<br>\| Column \| Value \|<br>\|---\|---\|<br>\| `connection_request_status: ` \| `UserInvitationSent` \|<br>\| `invitation_id: ` \| Unique ID from Unipile \|<br><br>### 🎯 Why This Matters:<br>- The **Filter node** uses `connection_request_status` to skip leads on future runs<br>- `invitation_id` can be used for follow-up message workflows<br>- Matched by `row_number` — always updates the exact correct row<br><br>### 💡 Build On This:<br>You can extend this workflow to send follow-up messages to accepted connections using the stored `invitation_id`. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure the interval according to your outreach cadence.
   - Practical recommendation from the workflow notes: every 4 to 8 hours.
   - This is the workflow entry point.

3. **Add a Google Sheets node named `Get Leads`**
   - Node type: `Google Sheets`
   - Connect `Schedule Trigger -> Get Leads`
   - Credential: create or select a `Google Sheets OAuth2 API` credential with access to the target spreadsheet.
   - Choose the spreadsheet containing your leads.
   - Select the sheet/tab named `leads` or the equivalent sheet at `gid=0`.
   - Configure the node to read rows.
   - Ensure the sheet contains at least these columns:
     - `first_name`
     - `last_name`
     - `linkedin_url`
     - `connection_note`
     - `connection_request_status`
     - `invitation_id`
     - `row_number`

4. **Add a Filter node named `Filter`**
   - Node type: `Filter`
   - Connect `Get Leads -> Filter`
   - Create one condition:
     - Left value: `{{ $json.connection_request_status }}`
     - Operator: `notEquals`
     - Right value: `UserInvitationSent`
   - Keep strict validation and case-sensitive matching if you want to match the original workflow exactly.

5. **Add a Limit node named `Limit Connection Request`**
   - Node type: `Limit`
   - Connect `Filter -> Limit Connection Request`
   - Set `Max Items` to `10`
   - You may increase to `10–15` if you accept higher outreach volume, but the notes recommend staying conservative.

6. **Add a Code node named `Get Linkedin Username`**
   - Node type: `Code`
   - Connect `Limit Connection Request -> Get Linkedin Username`
   - Set language to JavaScript.
   - Use logic equivalent to:
     - Read `linkedin_url`
     - Remove a trailing slash
     - Extract the string after `/in/`
     - Return it as `username`
   - The original logic is:
     - read from `$input.first().json.linkedin_url`
     - output `{ username: ... }`
   - Important: if you want a more resilient implementation, add null checks before calling string methods.

7. **Add a Set node named `Data Arrangement`**
   - Node type: `Set`
   - Connect `Get Linkedin Username -> Data Arrangement`
   - Create these fields:
     - `username` = `{{ $json.username }}`
     - `row_number` = `{{ $('Filter').item.json.row_number }}`
     - `first_name` = `{{ $('Filter').item.json.first_name }}`
     - `connection_note` = `{{ $('Filter').item.json.connection_note }}`
     - `unipile_DSN` = your Unipile DSN, for example `apiX.unipile.com` or the DSN provided by Unipile
     - `unipile_api_key` = your Unipile API key
     - `unipile_account_ID` = your Unipile LinkedIn account ID
   - Replace the placeholders from the template with real values before activation.

8. **Add a Split In Batches node named `Loop Over Items`**
   - Node type: `Split In Batches`
   - Connect `Data Arrangement -> Loop Over Items`
   - Leave batch behavior at one item per loop cycle, matching the original loop design.

9. **Add an HTTP Request node named `Get Linkedin Provider ID`**
   - Node type: `HTTP Request`
   - Connect `Loop Over Items -> Get Linkedin Provider ID` from the batch-processing output
   - Configure:
     - Method: `GET`
     - URL: `https://{{ $json.unipile_DSN }}/api/v1/users/{{ $json.username }}`
   - Enable query parameters:
     - `account_id` = `{{ $json.unipile_account_ID }}`
   - Enable headers:
     - `X-API-KEY` = `{{ $json.unipile_api_key }}`
     - `accept` = `application/json`
   - Expected output: a JSON object containing `provider_id`

10. **Add an HTTP Request node named `Send Connection Request`**
    - Node type: `HTTP Request`
    - Connect `Get Linkedin Provider ID -> Send Connection Request`
    - Configure:
      - Method: `POST`
      - URL: `https://{{ $('Loop Over Items').item.json.unipile_DSN }}/api/v1/users/invite`
    - Enable body parameters:
      - `provider_id` = `{{ $json.provider_id }}`
      - `account_id` = `{{ $('Data Arrangement').item.json.unipile_account_ID }}`
      - `message` = `{{ $('Loop Over Items').item.json.connection_note }}`
    - Enable headers:
      - `X-API-KEY` = `{{ $('Loop Over Items').item.json.unipile_api_key }}`
      - `accept` = `application/json`
    - Expected response should include at least:
      - `invitation_id`
      - a status/object field used later for sheet update

11. **Add a Google Sheets node named `Update Connection Request Status and ID`**
    - Node type: `Google Sheets`
    - Connect `Send Connection Request -> Update Connection Request Status and ID`
    - Use the same Google Sheets OAuth2 credential as `Get Leads`
    - Select the same spreadsheet and sheet (`leads`)
    - Set operation to `Update`
    - Configure matching column:
      - `row_number`
    - Map fields to write:
      - `row_number` = `{{ $('Loop Over Items').item.json.row_number }}`
      - `invitation_id` = `{{ $json.invitation_id }}`
      - `connection_request_status` = `{{ $json.object }}`
    - Disable automatic type conversion if you want behavior identical to the original export.

12. **Add a Wait node named `Wait`**
    - Node type: `Wait`
    - Connect `Update Connection Request Status and ID -> Wait`
    - Set unit to `Minutes`
    - Set a delay of `3` to `5` minutes for safer pacing, since the exported workflow shows the unit but not a visible numeric value.
    - This delay is important to avoid rapid consecutive LinkedIn actions.

13. **Close the loop**
    - Connect `Wait -> Loop Over Items`
    - This makes the workflow process the next lead after each pause.

14. **Validate the loop wiring**
    - `Loop Over Items` should send one item at a time into:
      - `Get Linkedin Provider ID`
      - `Send Connection Request`
      - `Update Connection Request Status and ID`
      - `Wait`
      - back to `Loop Over Items`
    - If this is wired incorrectly, the workflow may only process one item or may not iterate at all.

15. **Set up credentials**
    - **Google Sheets OAuth2**
      - Must have read and update access to the spreadsheet
    - **Unipile**
      - No dedicated credential node is used in this workflow
      - DSN, API key, and account ID are embedded as values in the `Data Arrangement` node
      - For stronger security, you may replace these with environment variables or credentials-backed expressions

16. **Prepare the spreadsheet data**
    - Each lead row should include:
      - a valid LinkedIn profile URL in `/in/...` form
      - a personalized connection note
      - a unique `row_number`
    - Example logic:
      - blank `connection_request_status` means eligible
      - `UserInvitationSent` means already processed and should be skipped

17. **Run a test with one or two rows first**
    - Temporarily reduce the Limit node to `1` or `2`
    - Verify:
      - username extraction works
      - Unipile lookup returns `provider_id`
      - invitation sends successfully
      - the Google Sheet updates the correct row

18. **Activate the workflow**
    - Once verified, activate it so the schedule can run automatically.

19. **Optional hardening improvements**
    - Add an IF node after username extraction to skip rows with invalid LinkedIn URLs
    - Add error handling branches for HTTP 4xx/5xx responses
    - Store secrets in credentials or environment variables instead of plain text in the Set node
    - Normalize status values in the sheet to avoid duplicates caused by inconsistent formatting

### Sub-workflow setup
This workflow does **not** use any Execute Workflow or sub-workflow nodes. There are no child workflows to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sign up for Unipile, connect your LinkedIn account, and collect your API Key, DSN, and Account ID before enabling the workflow. | https://unipile.com |
| Recommended schedule frequency from the notes: run every 4–8 hours for safer outreach pacing. | Workflow setup guidance |
| Recommended batch size from the notes: 10–15 requests per run, with conservative safety practices. | Workflow setup guidance |
| Recommended wait time from the notes: 3–5 minutes between requests. | LinkedIn safety guidance |
| Suggested LinkedIn safety rules from the notes: keep daily total under 20 requests, use a warmed-up account, and vary connection notes. | Operational best practice |
| The workflow stores Unipile settings directly in the `Data Arrangement` node. For production use, moving them to credentials or environment-backed expressions is safer. | Implementation note |
| The filter only excludes rows whose `connection_request_status` is exactly `UserInvitationSent`. Any other status value will still be processed unless additional conditions are added. | Important behavior note |
| The sheet update depends on `row_number` as the unique match key. Ensure each row has a stable, unique value. | Data integrity note |