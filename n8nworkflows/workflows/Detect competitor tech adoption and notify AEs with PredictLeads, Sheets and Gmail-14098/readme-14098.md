Detect competitor tech adoption and notify AEs with PredictLeads, Sheets and Gmail

https://n8nworkflows.xyz/workflows/detect-competitor-tech-adoption-and-notify-aes-with-predictleads--sheets-and-gmail-14098


# Detect competitor tech adoption and notify AEs with PredictLeads, Sheets and Gmail

## 1. Workflow Overview

This workflow monitors a list of target companies for adoption of a competitor’s technology using PredictLeads, then alerts the assigned Account Executive (AE) by email when a match is found.

Typical use case: a sales team tracks prospect domains in Google Sheets and wants proactive alerts when those prospects begin using a competing platform such as Salesforce. The workflow runs daily, checks each company individually, evaluates the detected technologies, and sends a Gmail notification to the AE responsible for that account.

### 1.1 Trigger & Watchlist Intake
The workflow starts on a daily schedule and reads a Google Sheets watchlist containing company domains and AE contact information.

### 1.2 Per-Company Enrichment
Each row from the watchlist is processed one at a time. For each company domain, the workflow calls the PredictLeads API to fetch technology detection data.

### 1.3 Competitor Match Evaluation
A code node compares the detected technologies against a hardcoded competitor technology name and flags any matches.

### 1.4 Conditional Notification
If a competitor match exists, the workflow validates the AE email address, formats a message, and sends an alert through Gmail. If no match exists, or if the email is invalid, the workflow simply moves on to the next company.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Company Source

**Overview:**  
This block initiates the workflow once per day and loads the company watchlist from Google Sheets. The sheet acts as the source of truth for which companies to monitor and who should be notified.

**Nodes Involved:**  
- ⏰ Daily Schedule Trigger  
- 📂 Read Company Watchlist

#### Node: ⏰ Daily Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a cron schedule.
- **Configuration choices:**  
  Configured with cron expression `0 8 * * *`, meaning it runs every day at 08:00 according to the n8n instance timezone.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Output → `📂 Read Company Watchlist`
- **Version-specific requirements:**  
  Uses node type version `1.2`.
- **Edge cases or potential failure types:**  
  - Unexpected run time if server timezone differs from business timezone
  - Workflow will not run unless activated
- **Sub-workflow reference:**  
  None.

#### Node: 📂 Read Company Watchlist
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheet containing monitored companies.
- **Configuration choices:**  
  - Uses a specific Google Sheets document ID placeholder: `YOUR_GOOGLE_SHEET_ID_02`
  - Reads from sheet `gid=0`, cached as `CompetitorWatchlist`
  - No custom options configured
- **Key expressions or variables used:**  
  None directly, but downstream logic expects columns such as:
  - `domain`
  - `company_name`
  - `ae_email`
- **Input and output connections:**  
  - Input ← `⏰ Daily Schedule Trigger`
  - Output → `🔄 Loop Over Companies`
- **Version-specific requirements:**  
  Uses Google Sheets node version `4.5`; requires valid Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - OAuth2 authentication failure
  - Spreadsheet not found or access denied
  - Wrong sheet selected
  - Missing expected columns (`domain`, `ae_email`)
  - Empty rows may propagate invalid items downstream
- **Sub-workflow reference:**  
  None.

---

### Block 2 — Company Processing

**Overview:**  
This block iterates over the watchlist and enriches each company with current technology detection data from PredictLeads. The batching structure ensures one company is processed at a time.

**Nodes Involved:**  
- 🔄 Loop Over Companies  
- 🔍 Fetch Tech Detections

#### Node: 🔄 Loop Over Companies
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the rows from the watchlist and feeds them into the rest of the workflow one item at a time.
- **Configuration choices:**  
  Default options only; no custom batch size shown.
- **Key expressions or variables used:**  
  Used later by the code node through cross-node access:
  - `$('🔄 Loop Over Companies').first().json`
- **Input and output connections:**  
  - Input ← `📂 Read Company Watchlist`
  - Output 1 (loop branch) → `🔍 Fetch Tech Detections`
  - Receives loop-back input from:
    - `📧 Gmail Alert to AE`
    - `❓ Competitor Tech Found?` false branch
- **Version-specific requirements:**  
  Uses node version `3`.
- **Edge cases or potential failure types:**  
  - If the input dataset is empty, nothing else runs
  - Cross-node access with `.first()` can be fragile if execution context changes or node behavior differs from expectations
- **Sub-workflow reference:**  
  None.

#### Node: 🔍 Fetch Tech Detections
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the PredictLeads API to retrieve technology detections for the current company domain.
- **Configuration choices:**  
  - URL is dynamically built:  
    `https://predictleads.com/api/v3/companies/{{ $json.domain }}/technology_detections`
  - Sends headers manually:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `{{ $json.domain }}` from the current company row
- **Input and output connections:**  
  - Input ← `🔄 Loop Over Companies`
  - Output → `⚙️ Check Competitor Tech`
- **Version-specific requirements:**  
  Uses HTTP Request node version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing or blank `domain` creates an invalid endpoint
  - API key/token errors
  - Rate limiting from PredictLeads
  - Network timeout or 4xx/5xx responses
  - Response schema may differ from expected `data` array
- **Sub-workflow reference:**  
  None.

---

### Block 3 — Competitor Technology Check

**Overview:**  
This block analyzes the PredictLeads response and checks whether the configured competitor technology appears in the detection list. It then routes execution based on whether a match was found.

**Nodes Involved:**  
- ⚙️ Check Competitor Tech  
- ❓ Competitor Tech Found?

#### Node: ⚙️ Check Competitor Tech
- **Type and technical role:** `n8n-nodes-base.code`  
  Processes the PredictLeads response and produces a normalized result for downstream notification logic.
- **Configuration choices:**  
  The node contains JavaScript with these main behaviors:
  - Hardcodes `COMPETITOR_TECH = 'Salesforce'`
  - Retrieves company metadata from `🔄 Loop Over Companies`
  - Reads PredictLeads response from `$input.first().json.data || []`
  - Performs case-insensitive matching on `technology_name` or `name`
  - Outputs:
    - `domain`
    - `companyName`
    - `aeEmail`
    - `competitorTechFound`
    - `matchedTechnologies`
    - `totalTechDetections`
    - `competitorTechName`
    - `checkDate`
- **Key expressions or variables used:**  
  - `$('🔄 Loop Over Companies').first().json`
  - `$input.first().json.data || []`
  - `new Date().toISOString().split('T')[0]`
- **Input and output connections:**  
  - Input ← `🔍 Fetch Tech Detections`
  - Output → `❓ Competitor Tech Found?`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - If PredictLeads does not return `data`, node falls back to empty array
  - If `domain`, `company_name`, or `ae_email` are missing from the watchlist, output may be incomplete
  - Matching is substring-based; may produce false positives if a technology name contains the competitor string incidentally
  - Hardcoded competitor value requires manual editing for reuse
  - Reliance on cross-node `.first()` may be brittle in more complex batch scenarios
- **Sub-workflow reference:**  
  None.

#### Node: ❓ Competitor Tech Found?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the previous node found a competitor technology match.
- **Configuration choices:**  
  Boolean equality check:
  - Left value: `{{ $json.competitorTechFound }}`
  - Right value: `true`
- **Key expressions or variables used:**  
  - `{{ $json.competitorTechFound }}`
- **Input and output connections:**  
  - Input ← `⚙️ Check Competitor Tech`
  - True output → `⚙️ Extract AE Email`
  - False output → `🔄 Loop Over Companies`
- **Version-specific requirements:**  
  Uses IF node version `2`.
- **Edge cases or potential failure types:**  
  - If `competitorTechFound` is undefined or not a strict boolean, the condition may fail unexpectedly
- **Sub-workflow reference:**  
  None.

---

### Block 4 — Notification

**Overview:**  
This block prepares and sends an email alert to the assigned AE when a competitor technology match exists. It also validates the email format at a basic level before sending.

**Nodes Involved:**  
- ⚙️ Extract AE Email  
- 📧 Gmail Alert to AE

#### Node: ⚙️ Extract AE Email
- **Type and technical role:** `n8n-nodes-base.code`  
  Validates the AE email and constructs email metadata for the Gmail node.
- **Configuration choices:**  
  The JavaScript:
  - Reads current item
  - Extracts and trims `aeEmail`
  - Returns no items if email is missing or does not contain `@`
  - Builds:
    - `toEmail`
    - `emailSubject`
    - `emailBody`
- **Key expressions or variables used:**  
  - `$input.first().json`
  - Subject template: `Competitor Alert: ${item.companyName} adopted ${item.competitorTechName}`
  - Body includes company name, domain, matched technologies, and date
- **Input and output connections:**  
  - Input ← `❓ Competitor Tech Found?` true branch
  - Output → `📧 Gmail Alert to AE`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - Very basic email validation; malformed addresses like `abc@` still pass
  - If `matchedTechnologies` is empty despite a true flag, email content may be incomplete
  - Returns zero items on invalid email, which silently skips notification
- **Sub-workflow reference:**  
  None.

#### Node: 📧 Gmail Alert to AE
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the notification email to the AE.
- **Configuration choices:**  
  - Recipient: `{{ $json.toEmail }}`
  - Subject: `{{ $json.emailSubject }}`
  - Message: `{{ $json.emailBody }}`
  - `emailType` is set to `text`
- **Key expressions or variables used:**  
  - `{{ $json.toEmail }}`
  - `{{ $json.emailSubject }}`
  - `{{ $json.emailBody }}`
- **Input and output connections:**  
  - Input ← `⚙️ Extract AE Email`
  - Output → `🔄 Loop Over Companies` to continue batch processing
- **Version-specific requirements:**  
  Uses Gmail node version `2.1`; requires valid Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Gmail OAuth failure
  - Sending quota exceeded
  - Recipient rejected
  - Plain text email body includes Markdown-like `**bold**` markers, which will not render as formatting in text mode
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| About This Workflow | Sticky Note | General workflow documentation on canvas |  |  | ABOUT THIS WORKFLOW  \nMonitors companies for competitor technology adoption and emails the assigned account executive when a match is found.  \nSetup: Google Sheet with domains and AE emails, Gmail OAuth2, PredictLeads API credentials.  \nUse case: You sell a CRM alternative. This detects when a prospect adopts Salesforce and notifies your AE so they can engage before the prospect commits long-term.  \nPredictLeads API: https://predictleads.com  \nQuestions: https://www.linkedin.com/in/yaronbeen |
| ⏰ Daily Schedule Trigger | Schedule Trigger | Starts the workflow daily at 08:00 |  | 📂 Read Company Watchlist | ## 1️⃣ Trigger & Company Source  \n**Nodes:** ⏰ Daily Schedule Trigger → 📂 Read Company Watchlist  \n**Description:** The workflow begins with a scheduled trigger that runs automatically each day. It loads the list of companies from a Google Sheets watchlist. This sheet contains the domains of companies to monitor along with the assigned Account Executive (AE) responsible for each account. The watchlist acts as the monitoring source, defining which companies should be checked for competitor technology adoption. |
| 📂 Read Company Watchlist | Google Sheets | Loads company rows from the watchlist spreadsheet | ⏰ Daily Schedule Trigger | 🔄 Loop Over Companies | ## 1️⃣ Trigger & Company Source  \n**Nodes:** ⏰ Daily Schedule Trigger → 📂 Read Company Watchlist  \n**Description:** The workflow begins with a scheduled trigger that runs automatically each day. It loads the list of companies from a Google Sheets watchlist. This sheet contains the domains of companies to monitor along with the assigned Account Executive (AE) responsible for each account. The watchlist acts as the monitoring source, defining which companies should be checked for competitor technology adoption. |
| 📋 Sticky: Trigger & Input | Sticky Note | Visual documentation for trigger and input block |  |  | ## 1️⃣ Trigger & Company Source  \n**Nodes:** ⏰ Daily Schedule Trigger → 📂 Read Company Watchlist  \n**Description:** The workflow begins with a scheduled trigger that runs automatically each day. It loads the list of companies from a Google Sheets watchlist. This sheet contains the domains of companies to monitor along with the assigned Account Executive (AE) responsible for each account. The watchlist acts as the monitoring source, defining which companies should be checked for competitor technology adoption. |
| 🔄 Loop Over Companies | Split In Batches | Iterates through companies one by one and manages loop continuation | 📂 Read Company Watchlist, 📧 Gmail Alert to AE, ❓ Competitor Tech Found? | 🔍 Fetch Tech Detections | ## 2️⃣ Company Processing  \n**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Tech Detections  \n**Description:** Each company from the watchlist is processed individually. For every company domain, the workflow calls the PredictLeads API to retrieve recent technology detection data. This data includes technologies currently detected on the company’s website or infrastructure. The retrieved technology signals are then passed to the next step for competitor comparison. |
| 🔍 Fetch Tech Detections | HTTP Request | Queries PredictLeads for technology detections by domain | 🔄 Loop Over Companies | ⚙️ Check Competitor Tech | ## 2️⃣ Company Processing  \n**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Tech Detections  \n**Description:** Each company from the watchlist is processed individually. For every company domain, the workflow calls the PredictLeads API to retrieve recent technology detection data. This data includes technologies currently detected on the company’s website or infrastructure. The retrieved technology signals are then passed to the next step for competitor comparison. |
| 📋 Sticky: Enrichment | Sticky Note | Visual documentation for enrichment block |  |  | ## 2️⃣ Company Processing  \n**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Tech Detections  \n**Description:** Each company from the watchlist is processed individually. For every company domain, the workflow calls the PredictLeads API to retrieve recent technology detection data. This data includes technologies currently detected on the company’s website or infrastructure. The retrieved technology signals are then passed to the next step for competitor comparison. |
| ⚙️ Check Competitor Tech | Code | Compares detected technologies against the target competitor technology | 🔍 Fetch Tech Detections | ❓ Competitor Tech Found? | ## 3️⃣ Competitor Technology Check  \n**Nodes:** ⚙️ Check Competitor Tech → ❓ Competitor Tech Found?  \n**Description:** The workflow evaluates the detected technologies to determine whether a configured competitor technology is present. The system compares all detected technologies against a predefined competitor tool (for example Salesforce or another competing platform). If a match is found, the company is flagged as having adopted the competitor’s technology, triggering a notification workflow. |
| ❓ Competitor Tech Found? | IF | Routes matched companies to notification and non-matches back to the loop | ⚙️ Check Competitor Tech | ⚙️ Extract AE Email, 🔄 Loop Over Companies | ## 3️⃣ Competitor Technology Check  \n**Nodes:** ⚙️ Check Competitor Tech → ❓ Competitor Tech Found?  \n**Description:** The workflow evaluates the detected technologies to determine whether a configured competitor technology is present. The system compares all detected technologies against a predefined competitor tool (for example Salesforce or another competing platform). If a match is found, the company is flagged as having adopted the competitor’s technology, triggering a notification workflow. |
| 📋 Sticky: Processing | Sticky Note | Visual documentation for competitor check block |  |  | ## 3️⃣ Competitor Technology Check  \n**Nodes:** ⚙️ Check Competitor Tech → ❓ Competitor Tech Found?  \n**Description:** The workflow evaluates the detected technologies to determine whether a configured competitor technology is present. The system compares all detected technologies against a predefined competitor tool (for example Salesforce or another competing platform). If a match is found, the company is flagged as having adopted the competitor’s technology, triggering a notification workflow. |
| ⚙️ Extract AE Email | Code | Validates recipient email and formats the email payload | ❓ Competitor Tech Found? | 📧 Gmail Alert to AE | ## 4️⃣ Notification  \n**Nodes:** ⚙️ Extract AE Email → 📧 Gmail Alert to AE  \n**Description:** When a competitor technology match is detected, the workflow extracts the assigned Account Executive’s email from the watchlist. A notification email is then sent via Gmail to the responsible AE. The alert contains the company name, domain, detected competitor technology, and detection date. This allows the sales team to quickly respond to potential competitive risks or opportunities. |
| 📧 Gmail Alert to AE | Gmail | Sends the alert email to the assigned AE | ⚙️ Extract AE Email | 🔄 Loop Over Companies | ## 4️⃣ Notification  \n**Nodes:** ⚙️ Extract AE Email → 📧 Gmail Alert to AE  \n**Description:** When a competitor technology match is detected, the workflow extracts the assigned Account Executive’s email from the watchlist. A notification email is then sent via Gmail to the responsible AE. The alert contains the company name, domain, detected competitor technology, and detection date. This allows the sales team to quickly respond to potential competitive risks or opportunities. |
| 📋 Sticky: Output | Sticky Note | Visual documentation for notification block |  |  | ## 4️⃣ Notification  \n**Nodes:** ⚙️ Extract AE Email → 📧 Gmail Alert to AE  \n**Description:** When a competitor technology match is detected, the workflow extracts the assigned Account Executive’s email from the watchlist. A notification email is then sent via Gmail to the responsible AE. The alert contains the company name, domain, detected competitor technology, and detection date. This allows the sales team to quickly respond to potential competitive risks or opportunities. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Detect competitor tech adoption and notify AEs with PredictLeads, Sheets and Gmail`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name it: `⏰ Daily Schedule Trigger`
   - Set it to run using a cron expression:
     - `0 8 * * *`
   - This runs daily at 08:00 based on your n8n server timezone.

3. **Prepare the Google Sheet**
   - Create a Google Sheet that will act as the watchlist.
   - Include at least these columns:
     - `domain`
     - `company_name`
     - `ae_email`
   - Example row:
     - `acme.com | Acme | ae@yourcompany.com`

4. **Add a Google Sheets node**
   - Node type: **Google Sheets**
   - Name it: `📂 Read Company Watchlist`
   - Connect `⏰ Daily Schedule Trigger` → `📂 Read Company Watchlist`
   - Configure:
     - Operation: read rows from the sheet
     - Spreadsheet: select your watchlist spreadsheet
     - Sheet/tab: select the first tab or the tab that contains the watchlist
   - Credentials:
     - Create or select **Google Sheets OAuth2** credentials
     - Ensure the connected Google account has access to the spreadsheet

5. **Add a Split In Batches node**
   - Node type: **Split In Batches**
   - Name it: `🔄 Loop Over Companies`
   - Connect `📂 Read Company Watchlist` → `🔄 Loop Over Companies`
   - Leave default settings unless you want to explicitly set batch size to 1
   - The intended behavior is one company processed per loop cycle.

6. **Add an HTTP Request node**
   - Node type: **HTTP Request**
   - Name it: `🔍 Fetch Tech Detections`
   - Connect `🔄 Loop Over Companies` main loop output → `🔍 Fetch Tech Detections`
   - Configure:
     - Method: `GET`
     - URL:
       - `https://predictleads.com/api/v3/companies/{{ $json.domain }}/technology_detections`
     - Send Headers: enabled
     - Headers:
       - `X-Api-Key` = your PredictLeads API key
       - `X-Api-Token` = your PredictLeads API token
       - `Content-Type` = `application/json`
   - Optional improvement:
     - Enable retry or response handling if you expect rate limits or occasional API issues.

7. **Add a Code node for competitor matching**
   - Node type: **Code**
   - Name it: `⚙️ Check Competitor Tech`
   - Connect `🔍 Fetch Tech Detections` → `⚙️ Check Competitor Tech`
   - Paste logic equivalent to this behavior:
     - Define a constant competitor technology, e.g. `Salesforce`
     - Read the current watchlist item from `🔄 Loop Over Companies`
     - Read PredictLeads detections from the HTTP response `data` array
     - Compare each detection name to the competitor technology using a case-insensitive match
     - Output normalized fields:
       - `domain`
       - `companyName`
       - `aeEmail`
       - `competitorTechFound`
       - `matchedTechnologies`
       - `totalTechDetections`
       - `competitorTechName`
       - `checkDate`
   - Important:
     - The original workflow hardcodes the competitor name as `Salesforce`
     - If you want another competitor, change that constant manually

8. **Add an IF node**
   - Node type: **IF**
   - Name it: `❓ Competitor Tech Found?`
   - Connect `⚙️ Check Competitor Tech` → `❓ Competitor Tech Found?`
   - Configure condition:
     - Value 1: `{{ $json.competitorTechFound }}`
     - Operation: equals
     - Value 2: `true`
   - This creates:
     - **True branch** for matched companies
     - **False branch** for non-matches

9. **Connect the false branch back to the loop**
   - Connect `❓ Competitor Tech Found?` false output → `🔄 Loop Over Companies`
   - This ensures non-matching companies are skipped and the workflow continues to the next row.

10. **Add a Code node for email preparation**
    - Node type: **Code**
    - Name it: `⚙️ Extract AE Email`
    - Connect `❓ Competitor Tech Found?` true output → `⚙️ Extract AE Email`
    - Implement logic equivalent to:
      - Read the incoming item
      - Trim `aeEmail`
      - If empty or missing `@`, return no items
      - Build:
        - `toEmail`
        - `emailSubject`
        - `emailBody`
    - Subject format:
      - `Competitor Alert: {companyName} adopted {competitorTechName}`
    - Body content should include:
      - Company
      - Domain
      - Competitor technology detected
      - Detection date
      - A short action-oriented message

11. **Add a Gmail node**
    - Node type: **Gmail**
    - Name it: `📧 Gmail Alert to AE`
    - Connect `⚙️ Extract AE Email` → `📧 Gmail Alert to AE`
    - Configure:
      - To: `{{ $json.toEmail }}`
      - Subject: `{{ $json.emailSubject }}`
      - Message: `{{ $json.emailBody }}`
      - Email type: `text`
    - Credentials:
      - Create or select **Gmail OAuth2** credentials
      - Ensure the account is authorized to send mail

12. **Connect the Gmail node back to the loop**
    - Connect `📧 Gmail Alert to AE` → `🔄 Loop Over Companies`
    - This is required so processing continues after a notification is sent.

13. **Optional: add sticky notes**
    - Add four sticky notes to document:
      - Trigger & Company Source
      - Company Processing
      - Competitor Technology Check
      - Notification
    - Add one general sticky note with setup and business context.

14. **Test the workflow**
    - Add sample rows to the sheet with valid domains and AE emails
    - Run manually
    - Verify:
      - Google Sheets returns rows
      - PredictLeads returns a `data` array
      - Match logic correctly identifies your chosen competitor
      - Gmail sends messages only when matches occur

15. **Activate the workflow**
    - Once credentials and test results are correct, activate the workflow so the schedule trigger runs daily.

### Required credentials
- **Google Sheets OAuth2**
  - Must have read access to the spreadsheet
- **Gmail OAuth2**
  - Must be allowed to send messages
- **PredictLeads API credentials**
  - API key
  - API token

### Expected input structure from Google Sheets
At minimum, each row should provide:
- `domain`: company domain used in the PredictLeads endpoint
- `company_name`: optional but recommended for email readability
- `ae_email`: recipient email address

### Expected output structure from PredictLeads
The workflow expects the HTTP response to contain:
- a top-level `data` array
- each array item ideally includes `technology_name` or `name`

### No sub-workflows
This workflow does not invoke or depend on any sub-workflows.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PredictLeads API | https://predictleads.com |
| Questions / creator contact | https://www.linkedin.com/in/yaronbeen |
| Setup prerequisites mentioned in the workflow | Google Sheet with domains and AE emails, Gmail OAuth2, PredictLeads API credentials |
| Business use case mentioned in the workflow | Sell against a CRM competitor and alert the assigned AE when a prospect adopts Salesforce |
| Workflow status | The workflow is currently inactive (`active: false`) in the provided export |
| Email formatting note | The email is sent as plain text, so Markdown-style `**bold**` markers will appear as literal characters rather than rich formatting |