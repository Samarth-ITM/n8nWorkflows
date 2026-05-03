Send personalized LinkedIn InMails to CEOs and founders with Google Sheets and Unipile

https://n8nworkflows.xyz/workflows/send-personalized-linkedin-inmails-to-ceos-and-founders-with-google-sheets-and-unipile-14445


# Send personalized LinkedIn InMails to CEOs and founders with Google Sheets and Unipile

# 1. Workflow Overview

This workflow automates outbound LinkedIn InMail outreach to CEOs and founders using a lead list stored in Google Sheets and message delivery through Unipile. It is designed for recurring prospecting runs where already-contacted leads are skipped, a limited number of fresh leads are processed per execution, and each successful send is written back to the spreadsheet for tracking.

The workflow is organized into four logical blocks:

## 1.1 Triggering and Lead Intake
The workflow starts on a schedule, reads rows from a Google Sheet, and filters out leads that have already been marked as contacted.

## 1.2 Lead Preparation
For each remaining lead, the workflow extracts the LinkedIn username from the profile URL and assembles all required operational fields, including Unipile connection details.

## 1.3 InMail Delivery Loop
The workflow iterates through leads one by one, resolves the LinkedIn provider ID through Unipile, sends an InMail via the Sales Navigator API context, then waits before moving to the next lead.

## 1.4 Status Tracking
After each successful InMail, the corresponding row in Google Sheets is updated with `Sent` status and the returned chat ID so the lead is not messaged again and can be used in follow-up automations.

---

# 2. Block-by-Block Analysis

## 2.1 Triggering and Lead Intake

**Overview:**  
This block launches the workflow on a recurring schedule, retrieves the lead dataset from Google Sheets, removes rows already marked as sent, and caps the number of leads processed in one run.

**Nodes Involved:**  
- Schedule Trigger
- Get Leads
- Filter
- Limit Connection Request

### Node: Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry point that starts the workflow automatically on a time-based schedule.
- **Configuration choices:**  
  Uses interval-based scheduling with a default interval object. In practice, this should be adjusted to a real cadence such as every 4–8 hours, as suggested in the sticky note.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Output → `Get Leads`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Misconfigured schedule causes too frequent or too infrequent sends
  - Excessive cadence may consume LinkedIn InMail credits too quickly
- **Sub-workflow reference:**  
  None

### Node: Get Leads
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads prospect rows from a Google Sheet.
- **Configuration choices:**  
  Targets spreadsheet ID `1IM0qnZlhYK7Pbj__xwyPgEprW-5-P3RBnen1xGZjSSk` and worksheet `gid=0`, labeled `leads`.
- **Key expressions or variables used:**  
  None in parameters; pulls sheet row data as workflow items.
- **Input and output connections:**  
  - Input ← `Schedule Trigger`
  - Output → `Filter`
- **Version-specific requirements:**  
  Type version `4.7`  
  Requires valid Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - OAuth token expired or missing
  - Sheet ID or tab selection invalid
  - Missing expected columns such as `first_name`, `company_name`, `linkedin_url`, `inmail`, `chat_ID`, `row_number`
  - Empty sheet returns no actionable items
- **Sub-workflow reference:**  
  None

### Node: Filter
- **Type and technical role:** `n8n-nodes-base.filter`  
  Excludes rows where InMail has already been sent.
- **Configuration choices:**  
  Keeps only items where `inmail != "Sent"`.
- **Key expressions or variables used:**  
  - Left value: `{{ $json.inmail }}`
- **Input and output connections:**  
  - Input ← `Get Leads`
  - Output → `Limit Connection Request`
- **Version-specific requirements:**  
  Type version `2.3`
- **Edge cases or potential failure types:**  
  - Case sensitivity matters: `sent`, `SENT`, or trailing spaces will not match `Sent`
  - Blank `inmail` values pass through, which is intended
  - If column name differs, the expression fails logically rather than technically
- **Sub-workflow reference:**  
  None

### Node: Limit Connection Request
- **Type and technical role:** `n8n-nodes-base.limit`  
  Restricts the number of leads processed in a single run.
- **Configuration choices:**  
  `maxItems = 10`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input ← `Filter`
  - Output → `Get Linkedin Username`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - If fewer than 10 leads remain, all pass through
  - If the schedule runs too often and limit is too high, you may exceed operational or credit thresholds
- **Sub-workflow reference:**  
  None

---

## 2.2 Lead Preparation

**Overview:**  
This block converts each lead row into the format required for downstream API requests. It extracts the LinkedIn username from the profile URL and adds all fields needed for message sending and sheet updates.

**Nodes Involved:**  
- Get Linkedin Username
- Data Arrangement

### Node: Get Linkedin Username
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the LinkedIn username slug from the `linkedin_url`.
- **Configuration choices:**  
  Custom JavaScript:
  - Reads `linkedin_url`
  - Removes a trailing slash
  - Splits on `/in/`
  - Extracts the username segment
  - Returns `{ username }`
- **Key expressions or variables used:**  
  - `const linkedinUrl = $input.first().json.linkedin_url;`
  - `linkedinUrl.replace(/\/$/, '')`
  - `cleanUrl.split('/in/')[1]?.split('/')[0] || null`
- **Input and output connections:**  
  - Input ← `Limit Connection Request`
  - Output → `Data Arrangement`
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - If `linkedin_url` is missing, JavaScript throws when calling `.replace()`
  - If URL is not in `/in/username/` format, returns `null`
  - Sales Navigator or non-standard LinkedIn URLs may not parse correctly
  - URLs with query parameters may still work if username appears before them, but malformed URLs may not
- **Sub-workflow reference:**  
  None

### Node: Data Arrangement
- **Type and technical role:** `n8n-nodes-base.set`  
  Consolidates parsed and original lead data into a single item structure for the loop and API requests.
- **Configuration choices:**  
  Assigns:
  - `username` from current item
  - `row_number` from `Filter`
  - `first_name` from `Filter`
  - `company_name` from `Limit Connection Request`
  - `unipile_DSN` hardcoded placeholder
  - `unipile_api_key` hardcoded placeholder
  - `unipile_account_ID` hardcoded placeholder
- **Key expressions or variables used:**  
  - `{{ $json.username }}`
  - `{{ $('Filter').item.json.row_number }}`
  - `{{ $('Filter').item.json.first_name }}`
  - `{{ $('Limit Connection Request').item.json.company_name }}`
- **Input and output connections:**  
  - Input ← `Get Linkedin Username`
  - Output → `Loop Over Items`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Placeholder credentials must be replaced manually; otherwise downstream HTTP requests fail
  - Cross-node item linking can become fragile if workflow structure changes
  - Missing `row_number`, `first_name`, or `company_name` will propagate null/undefined values
- **Sub-workflow reference:**  
  None

---

## 2.3 InMail Delivery Loop

**Overview:**  
This block processes one lead at a time. It resolves the LinkedIn contact identifier from Unipile, sends a personalized InMail, updates the sheet, then waits before looping to the next lead.

**Nodes Involved:**  
- Loop Over Items
- Get Linkedin Provider ID
- Send Inmail
- Wait

### Node: Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through items sequentially.
- **Configuration choices:**  
  Uses default batching behavior. In this workflow it acts as the loop controller:
  - Main output 1 is unused
  - Main output 2 sends the current item to processing
  - After `Wait`, the flow returns to this node to continue
- **Key expressions or variables used:**  
  Downstream nodes reference `$('Loop Over Items').item.json...`
- **Input and output connections:**  
  - Input ← `Data Arrangement`
  - Output 2 → `Get Linkedin Provider ID`
  - Input back from `Wait`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - Misunderstanding the second output can lead to broken loop logic when rebuilding
  - If no items arrive, downstream path never runs
- **Sub-workflow reference:**  
  None

### Node: Get Linkedin Provider ID
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries Unipile for a user object based on LinkedIn username and account context.
- **Configuration choices:**  
  GET request to:
  `https://{{ $json.unipile_DSN }}/api/v1/users/{{ $json.username }}`
  
  Sends query parameter:
  - `account_id = {{ $json.unipile_account_ID }}`
  
  Sends headers:
  - `X-API-KEY = {{ $json.unipile_api_key }}`
  - `accept = application/json`
- **Key expressions or variables used:**  
  - `{{ $json.unipile_DSN }}`
  - `{{ $json.username }}`
  - `{{ $json.unipile_account_ID }}`
  - `{{ $json.unipile_api_key }}`
- **Input and output connections:**  
  - Input ← `Loop Over Items`
  - Output → `Send Inmail`
- **Version-specific requirements:**  
  Type version `4.2`
- **Edge cases or potential failure types:**  
  - Invalid DSN, API key, or account ID
  - Username not found in Unipile context
  - Rate limiting or network errors
  - Response may not contain `provider_id`, causing the next node to fail logically
- **Sub-workflow reference:**  
  None

### Node: Send Inmail
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the actual LinkedIn InMail via Unipile chat creation endpoint.
- **Configuration choices:**  
  POST request to:
  `https://{{ $('Loop Over Items').item.json.unipile_DSN }}/api/v1/chats`
  
  Multipart form-data body includes:
  - `attendees_ids = {{ $json.provider_id }}`
  - `account_id = {{ $('Loop Over Items').item.json.unipile_account_ID }}`
  - `api = sales_navigator`
  - `subject = Automating the Repetitive Work Behind {{ company_name }}`
  - `text = personalized multi-line outreach message`
  
  Headers:
  - `accept = application/json`
  - `X-API-KEY = {{ $('Loop Over Items').item.json.unipile_api_key }}`
- **Key expressions or variables used:**  
  - `{{ $json.provider_id }}`
  - `{{ $('Loop Over Items').item.json.first_name }}`
  - `{{ $('Loop Over Items').item.json.company_name }}`
  - `{{ $('Loop Over Items').item.json.unipile_account_ID }}`
  - `{{ $('Loop Over Items').item.json.unipile_api_key }}`
- **Input and output connections:**  
  - Input ← `Get Linkedin Provider ID`
  - Output → `Update Data`
- **Version-specific requirements:**  
  Type version `4.2`
- **Edge cases or potential failure types:**  
  - Missing `provider_id`
  - Invalid LinkedIn entitlement: InMail may require Sales Navigator or Recruiter Classic
  - InMail credits exhausted
  - Unipile API rejects body format or account context
  - Subject/text personalization may produce awkward output if `first_name` or `company_name` is blank
  - If using Recruiter Classic, `api` must be changed from `sales_navigator` to `recruiter`
- **Sub-workflow reference:**  
  None

### Node: Wait
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses between sends to throttle outreach activity.
- **Configuration choices:**  
  Wait unit is set to `minutes`. No explicit duration value is shown in the JSON, so the node should be reviewed in the editor and set to the desired delay, such as 3–5 minutes as indicated in the sticky note.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input ← `Update Data`
  - Output → `Loop Over Items`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - If duration is left empty or unintended, pacing may not match expectations
  - Long waits increase overall execution duration
  - Depending on instance settings, long-running executions may interact with retention or timeout policies
- **Sub-workflow reference:**  
  None

---

## 2.4 Status Tracking

**Overview:**  
This block records successful sends back into Google Sheets so the same lead is not contacted again and the created conversation can be referenced for later follow-up workflows.

**Nodes Involved:**  
- Update Data

### Node: Update Data
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates an existing row in Google Sheets using `row_number` as the matching key.
- **Configuration choices:**  
  Operation: `update`  
  Target sheet: same spreadsheet and tab as the intake node  
  Writes:
  - `inmail = Sent`
  - `chat_ID = {{ $json.chat_id }}`
  - `row_number = {{ $('Loop Over Items').item.json.row_number }}`
  
  Matching column:
  - `row_number`
- **Key expressions or variables used:**  
  - `{{ $json.chat_id }}`
  - `{{ $('Loop Over Items').item.json.row_number }}`
- **Input and output connections:**  
  - Input ← `Send Inmail`
  - Output → `Wait`
- **Version-specific requirements:**  
  Type version `4.7`  
  Requires the same Google Sheets OAuth2 credentials as `Get Leads`.
- **Edge cases or potential failure types:**  
  - If `chat_id` is missing in API response, the row is still marked `Sent` unless additional validation is added
  - If `row_number` does not match exactly, the row may not update
  - Column names in the sheet must match configuration exactly
  - OAuth or sheet write permissions can fail
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts the workflow on a recurring schedule |  | Get Leads | ## 📥 Lead Collection & Filtering<br>Reads leads from Google Sheets and skips already-messaged contacts.<br>**Flow:** Schedule Trigger → Get Leads → Filter (`inmail` ≠ `Sent`) → Limit |
| Get Leads | Google Sheets | Reads lead rows from the spreadsheet | Schedule Trigger | Filter | ## 📥 Lead Collection & Filtering<br>Reads leads from Google Sheets and skips already-messaged contacts.<br>**Flow:** Schedule Trigger → Get Leads → Filter (`inmail` ≠ `Sent`) → Limit |
| Filter | Filter | Excludes leads already marked as sent | Get Leads | Limit Connection Request | ## 📥 Lead Collection & Filtering<br>Reads leads from Google Sheets and skips already-messaged contacts.<br>**Flow:** Schedule Trigger → Get Leads → Filter (`inmail` ≠ `Sent`) → Limit |
| Limit Connection Request | Limit | Caps how many leads are processed per execution | Filter | Get Linkedin Username | ## 📥 Lead Collection & Filtering<br>Reads leads from Google Sheets and skips already-messaged contacts.<br>**Flow:** Schedule Trigger → Get Leads → Filter (`inmail` ≠ `Sent`) → Limit |
| Get Linkedin Username | Code | Extracts LinkedIn username from profile URL | Limit Connection Request | Data Arrangement | ## ⚙️ Data Preparation<br>Extracts the LinkedIn username from the profile URL and bundles all fields for the loop.<br>- **Get LinkedIn Username** — parses `/in/username/` from the URL<br>- **Data Arrangement** — sets `username`, `first_name`, `company_name`, `row_number`, and Unipile credentials<br>> Replace placeholder values in **Data Arrangement**: DSN, API key, Account ID |
| Data Arrangement | Set | Builds the consolidated payload with lead fields and Unipile settings | Get Linkedin Username | Loop Over Items | ## ⚙️ Data Preparation<br>Extracts the LinkedIn username from the profile URL and bundles all fields for the loop.<br>- **Get LinkedIn Username** — parses `/in/username/` from the URL<br>- **Data Arrangement** — sets `username`, `first_name`, `company_name`, `row_number`, and Unipile credentials<br>> Replace placeholder values in **Data Arrangement**: DSN, API key, Account ID |
| Loop Over Items | Split In Batches | Iterates over leads one at a time | Data Arrangement, Wait | Get Linkedin Provider ID | ## 📨 InMail Sending Loop<br>Sends one personalized InMail per lead via Sales Navigator.<br>**Flow:** Loop → Get Provider ID → Send InMail → Update Sheet → Wait → repeat<br>- Uses `POST /api/v1/chats` with `api: sales_navigator`<br>- For Recruiter Classic: change `api` value to `recruiter`<br>- Inmail Credits deducted from your LinkedIn plan (not Unipile) |
| Get Linkedin Provider ID | HTTP Request | Resolves the target LinkedIn provider ID via Unipile | Loop Over Items | Send Inmail | ## 📨 InMail Sending Loop<br>Sends one personalized InMail per lead via Sales Navigator.<br>**Flow:** Loop → Get Provider ID → Send InMail → Update Sheet → Wait → repeat<br>- Uses `POST /api/v1/chats` with `api: sales_navigator`<br>- For Recruiter Classic: change `api` value to `recruiter`<br>- Inmail Credits deducted from your LinkedIn plan (not Unipile) |
| Send Inmail | HTTP Request | Sends the personalized LinkedIn InMail | Get Linkedin Provider ID | Update Data | ## 📨 InMail Sending Loop<br>Sends one personalized InMail per lead via Sales Navigator.<br>**Flow:** Loop → Get Provider ID → Send InMail → Update Sheet → Wait → repeat<br>- Uses `POST /api/v1/chats` with `api: sales_navigator`<br>- For Recruiter Classic: change `api` value to `recruiter`<br>- Inmail Credits deducted from your LinkedIn plan (not Unipile) |
| Update Data | Google Sheets | Marks lead as contacted and stores chat ID | Send Inmail | Wait | ## ✅ Tracking & Status Update<br>Writes results back to Google Sheets after each send.<br>| Column | Value |<br>|---|---|<br>| `inmail: ` | `Sent` — prevents duplicate messages |<br>| `chat_ID: ` | Conversation ID for follow-up workflows |<br>> Use the saved `chat_id` to build a follow-up sequence (e.g. Day 3 reply if no response). |
| Wait | Wait | Delays between sends before continuing the loop | Update Data | Loop Over Items | ## ✅ Tracking & Status Update<br>Writes results back to Google Sheets after each send.<br>| Column | Value |<br>|---|---|<br>| `inmail: ` | `Sent` — prevents duplicate messages |<br>| `chat_ID: ` | Conversation ID for follow-up workflows |<br>> Use the saved `chat_id` to build a follow-up sequence (e.g. Day 3 reply if no response). |
| 📖 Setup Instructions | Sticky Note | Workspace documentation for setup and operational guidance |  |  | # 📩 LinkedIn InMail Automation<br>Send personalized LinkedIn InMails to CEOs & founders via n8n, Google Sheets, and Unipile.<br>### ⚠️ Requirements<br>- LinkedIn **Sales Navigator** or **Recruiter Classic** (InMail credits are consumed from your plan)<br>### 🛠️ Setup<br>1. **Google Sheet columns:** `first_name` \| `company_name` \| `linkedin_url` \| `inmail` \| `chat_ID` \| `row_number`<br>2. **Unipile:** Sign up at [unipile.com](https://unipile.com) → connect LinkedIn → copy your API Key, DSN & Account ID<br>3. Paste credentials into the **Data Arrangement** node<br>4. **Limit node** → 10–15/run \| **Wait node** → 3–5 min \| **Schedule** → every 4–8 hrs |
| 📥 Lead Collection & Filtering | Sticky Note | Workspace documentation for intake and filtering |  |  | ## 📥 Lead Collection & Filtering<br>Reads leads from Google Sheets and skips already-messaged contacts.<br>**Flow:** Schedule Trigger → Get Leads → Filter (`inmail` ≠ `Sent`) → Limit |
| ⚙️ Data Preparation | Sticky Note | Workspace documentation for username parsing and field assembly |  |  | ## ⚙️ Data Preparation<br>Extracts the LinkedIn username from the profile URL and bundles all fields for the loop.<br>- **Get LinkedIn Username** — parses `/in/username/` from the URL<br>- **Data Arrangement** — sets `username`, `first_name`, `company_name`, `row_number`, and Unipile credentials<br>> Replace placeholder values in **Data Arrangement**: DSN, API key, Account ID |
| 📨 InMail Sending Loop | Sticky Note | Workspace documentation for the looped send process |  |  | ## 📨 InMail Sending Loop<br>Sends one personalized InMail per lead via Sales Navigator.<br>**Flow:** Loop → Get Provider ID → Send InMail → Update Sheet → Wait → repeat<br>- Uses `POST /api/v1/chats` with `api: sales_navigator`<br>- For Recruiter Classic: change `api` value to `recruiter`<br>- Inmail Credits deducted from your LinkedIn plan (not Unipile) |
| ✅ Tracking & Status Update | Sticky Note | Workspace documentation for result logging |  |  | ## ✅ Tracking & Status Update<br>Writes results back to Google Sheets after each send.<br>| Column | Value |<br>|---|---|<br>| `inmail: ` | `Sent` — prevents duplicate messages |<br>| `chat_ID: ` | Conversation ID for follow-up workflows |<br>> Use the saved `chat_id` to build a follow-up sequence (e.g. Day 3 reply if no response). |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Send personalized LinkedIn InMails to CEOs and founders with Google Sheets and Unipile**

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure an interval-based schedule
   - Recommended cadence from the notes: every 4–8 hours
   - This will be the workflow entry point

3. **Add a Google Sheets node named `Get Leads`**
   - Node type: `Google Sheets`
   - Operation: read rows
   - Connect Google Sheets OAuth2 credentials
   - Select the spreadsheet containing your leads
   - Select the worksheet/tab containing the lead list
   - Ensure your sheet contains these columns:
     - `first_name`
     - `company_name`
     - `linkedin_url`
     - `inmail`
     - `chat_ID`
     - `row_number`
   - Connect: `Schedule Trigger` → `Get Leads`

4. **Add a Filter node named `Filter`**
   - Condition: keep rows where `inmail` is not equal to `Sent`
   - Use:
     - Left value: `{{ $json.inmail }}`
     - Operator: `notEquals`
     - Right value: `Sent`
   - Connect: `Get Leads` → `Filter`

5. **Add a Limit node named `Limit Connection Request`**
   - Node type: `Limit`
   - Set `maxItems` to `10`
   - You may change this to `10–15` depending on your campaign policy
   - Connect: `Filter` → `Limit Connection Request`

6. **Add a Code node named `Get Linkedin Username`**
   - Node type: `Code`
   - Language: JavaScript
   - Paste this logic:
     ```javascript
     const linkedinUrl = $input.first().json.linkedin_url;
     const cleanUrl = linkedinUrl.replace(/\/$/, '');
     const username = cleanUrl.split('/in/')[1]?.split('/')[0] || null;

     return [{
       json: {
         username: username
       }
     }];
     ```
   - Purpose: extract the LinkedIn username slug from a standard profile URL
   - Connect: `Limit Connection Request` → `Get Linkedin Username`

7. **Add a Set node named `Data Arrangement`**
   - Node type: `Set`
   - Create the following fields:
     - `username` → `{{ $json.username }}`
     - `row_number` → `{{ $('Filter').item.json.row_number }}`
     - `first_name` → `{{ $('Filter').item.json.first_name }}`
     - `company_name` → `{{ $('Limit Connection Request').item.json.company_name }}`
     - `unipile_DSN` → your Unipile DSN
     - `unipile_api_key` → your Unipile API key
     - `unipile_account_ID` → your Unipile LinkedIn account ID
   - Replace placeholder values with your real Unipile values
   - Connect: `Get Linkedin Username` → `Data Arrangement`

8. **Add a Split In Batches node named `Loop Over Items`**
   - Node type: `Split In Batches`
   - Default settings are sufficient
   - This controls sequential processing of one lead at a time
   - Connect: `Data Arrangement` → `Loop Over Items`

9. **Add an HTTP Request node named `Get Linkedin Provider ID`**
   - Node type: `HTTP Request`
   - Method: `GET`
   - URL:
     ```text
     https://{{ $json.unipile_DSN }}/api/v1/users/{{ $json.username }}
     ```
   - Add query parameter:
     - `account_id` → `{{ $json.unipile_account_ID }}`
   - Add headers:
     - `X-API-KEY` → `{{ $json.unipile_api_key }}`
     - `accept` → `application/json`
   - Connect this node from the **loop-processing output** of `Loop Over Items`
   - Connect: `Loop Over Items` → `Get Linkedin Provider ID`

10. **Add an HTTP Request node named `Send Inmail`**
    - Node type: `HTTP Request`
    - Method: `POST`
    - URL:
      ```text
      https://{{ $('Loop Over Items').item.json.unipile_DSN }}/api/v1/chats
      ```
    - Enable body sending
    - Content type: `multipart-form-data`
    - Add headers:
      - `accept` → `application/json`
      - `X-API-KEY` → `{{ $('Loop Over Items').item.json.unipile_api_key }}`
    - Add body parameters:
      - `attendees_ids` → `{{ $json.provider_id }}`
      - `account_id` → `{{ $('Loop Over Items').item.json.unipile_account_ID }}`
      - `api` → `sales_navigator`
      - `subject` → `Automating the Repetitive Work Behind {{ $('Loop Over Items').item.json.company_name }}`
      - `text` → personalized message body using `first_name` and `company_name`
    - Example body text:
      ```text
      Hi {{ $('Loop Over Items').item.json.first_name }},

      I noticed you are leading {{ $('Loop Over Items').item.json.company_name }} and wanted to reach out directly.

      Growing a company comes with a lot of moving parts — and more often than not, a significant amount of your team's time goes into manual, repetitive tasks that could be fully automated.

      I specialise in building n8n automations for founders and CEOs — helping businesses like {{ $('Loop Over Items').item.json.company_name }} eliminate manual work across areas such as:

      - Lead outreach and follow-up sequences
      - CRM data entry and pipeline updates
      - Client onboarding and contract workflows
      - Internal reporting and team notifications
      - Invoice generation and payment tracking

      The result is a leaner operation, a more focused team, and more of your time back where it belongs — on growing {{ $('Loop Over Items').item.json.company_name }}.

      I would love to understand where {{ $('Loop Over Items').item.json.company_name }} is currently losing the most time and show you exactly how automation could solve it.

      Would you be open to a quick 20-minute call this week?

      Best regards,
      [Your Name]
      [Your Website / Calendar Link]
      ```
    - If you use Recruiter Classic instead of Sales Navigator, change:
      - `api = recruiter`
    - Connect: `Get Linkedin Provider ID` → `Send Inmail`

11. **Add a Google Sheets node named `Update Data`**
    - Node type: `Google Sheets`
    - Operation: `Update`
    - Use the same spreadsheet and worksheet as `Get Leads`
    - Matching column:
      - `row_number`
    - Map these columns:
      - `inmail` → `Sent`
      - `chat_ID` → `{{ $json.chat_id }}`
      - `row_number` → `{{ $('Loop Over Items').item.json.row_number }}`
    - Reuse the same Google Sheets OAuth2 credential
    - Connect: `Send Inmail` → `Update Data`

12. **Add a Wait node named `Wait`**
    - Node type: `Wait`
    - Set the unit to `minutes`
    - Recommended delay: 3–5 minutes
    - This reduces send frequency between leads
    - Connect: `Update Data` → `Wait`

13. **Close the loop**
    - Connect: `Wait` → `Loop Over Items`
    - This returns execution to the batch node so the next lead can be processed

14. **Verify the loop output selection**
    - In n8n, `Split In Batches` has multiple outputs
    - Ensure the processing path (`Get Linkedin Provider ID`) is attached to the output used for iterating current items, matching the original workflow logic

15. **Add optional sticky notes for workspace clarity**
    - Create one note for setup requirements
    - One for lead collection and filtering
    - One for data preparation
    - One for InMail sending loop
    - One for tracking and follow-up context

16. **Configure credentials**
    - **Google Sheets OAuth2**
      - Must have read and update access to the spreadsheet
    - **Unipile**
      - No n8n credential object is used here; values are manually stored in `Data Arrangement`
      - Required values:
        - DSN
        - API Key
        - LinkedIn Account ID

17. **Test with one or two leads first**
    - Add a couple of rows with valid:
      - `first_name`
      - `company_name`
      - `linkedin_url`
      - blank `inmail`
      - unique `row_number`
    - Run manually before enabling the schedule

18. **Activate the workflow**
    - Once confirmed, activate the schedule
    - Monitor chat creation responses and the `chat_ID` written to Google Sheets

### Input expectations
Each sheet row should provide at minimum:
- `first_name`
- `company_name`
- `linkedin_url`
- `row_number`

### Output expectations
For each successful lead:
- A LinkedIn InMail is sent through Unipile
- Google Sheets row is updated:
  - `inmail = Sent`
  - `chat_ID = returned chat_id`

### No sub-workflow usage
This workflow does not call any sub-workflow and is not itself documented as a sub-workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| LinkedIn Sales Navigator or Recruiter Classic is required for InMail sending. InMail credits come from your LinkedIn plan, not from Unipile. | Operational requirement |
| Expected Google Sheet columns: `first_name`, `company_name`, `linkedin_url`, `inmail`, `chat_ID`, `row_number` | Data model |
| Replace placeholder values in the `Data Arrangement` node with your real Unipile DSN, API key, and account ID. | Configuration requirement |
| Recommended operating settings: Limit node `10–15` per run, Wait node `3–5` minutes, Schedule every `4–8` hours. | Throughput and pacing guidance |
| For Recruiter Classic, change the `api` field in `Send Inmail` from `sales_navigator` to `recruiter`. | Unipile / LinkedIn mode switch |
| The saved `chat_id` can be reused in a later workflow to implement follow-up sequences if the lead does not reply. | Extension idea |
| Unipile signup and account connection | [https://unipile.com](https://unipile.com) |