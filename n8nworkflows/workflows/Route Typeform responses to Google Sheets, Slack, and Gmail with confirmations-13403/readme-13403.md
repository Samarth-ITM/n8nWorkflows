Route Typeform responses to Google Sheets, Slack, and Gmail with confirmations

https://n8nworkflows.xyz/workflows/route-typeform-responses-to-google-sheets--slack--and-gmail-with-confirmations-13403


# Route Typeform responses to Google Sheets, Slack, and Gmail with confirmations

## 1. Workflow Overview

**Purpose:**  
This workflow receives new **Typeform** submissions, normalizes key fields (name, email, interest, submitted timestamp), logs each submission to **Google Sheets**, routes the submission to the appropriate **Slack** channel based on the “interest” answer, and sends a **Gmail confirmation email** to the respondent.

**Target use cases:**
- Lead triage (e.g., pricing/sales inquiries → Sales Slack channel)
- Support intake (support/help → Support Slack channel)
- Centralized logging of form submissions in Google Sheets
- Automatic respondent acknowledgement via email

### 1.1 Input Reception & Normalization
Triggered by a Typeform webhook; extracts/normalizes key fields into consistent JSON properties.

### 1.2 Persistence / Logging
Appends a row to a Google Sheets tab (“Responses”) for every submission.

### 1.3 Routing & Notifications
Uses a Switch to route submissions to Slack channels based on `interest`, and also sends a confirmation email (note: in the current wiring, email is only sent on fallback output).

---

## 2. Block-by-Block Analysis

### Block 1 — Receive Typeform Submission & Normalize Fields
**Overview:**  
Listens for new Typeform responses, then maps multiple possible question labels into a stable internal schema (`name`, `email`, `interest`, `submitted_at`) to avoid downstream breakage when Typeform labels vary.

**Nodes involved:**
- Typeform Trigger
- Set Response Fields

#### Node: Typeform Trigger
- **Type / role:** `typeformTrigger` — webhook entry point for new Typeform submissions.
- **Configuration (interpreted):**
  - Uses a **Form ID** placeholder: `Your Typeform form ID` (must be replaced).
  - On each submission, emits the submission payload to the workflow.
- **Key variables/expressions:** none.
- **Connections:**
  - **Output →** Set Response Fields
- **Version requirements:** node typeVersion **1**.
- **Edge cases / failures:**
  - Missing/incorrect form ID → trigger won’t receive events.
  - Typeform credential not connected/invalid → webhook setup fails.
  - Payload structure changes depending on Typeform configuration and n8n’s Typeform node mapping; field names may not match what the Set node expects.

#### Node: Set Response Fields
- **Type / role:** `set` — normalizes fields and creates a predictable schema for downstream nodes.
- **Configuration (interpreted):**
  - Creates/sets:
    - `name` from `What is your name?` OR `Name` OR `''`
    - `email` from `What is your email?` OR `Email` OR `''`
    - `interest` from `What are you interested in?` OR `Interest` OR `''`
    - `submitted_at` from `submittedAt` OR current timestamp (`$now.toISO()`)
- **Key expressions:**
  - `{{ $json.What is your name? || $json.Name || '' }}`
  - `{{ $json.What is your email? || $json.Email || '' }}`
  - `{{ $json.What are you interested in? || $json.Interest || '' }}`
  - `{{ $json.submittedAt || $now.toISO() }}`
- **Connections:**
  - **Input ←** Typeform Trigger
  - **Output →** Log to Google Sheets
- **Version requirements:** typeVersion **3.4**.
- **Edge cases / failures:**
  - If Typeform fields don’t map to these keys, the result may be empty strings, causing:
    - Google Sheets rows with blanks
    - Slack messages missing identifiers
    - Gmail send failures if `email` is empty/invalid

---

### Block 2 — Log Every Submission to Google Sheets
**Overview:**  
Persists every normalized response as a new row in a spreadsheet tab named “Responses”.

**Nodes involved:**
- Log to Google Sheets

#### Node: Log to Google Sheets
- **Type / role:** `googleSheets` — appends a row to a Google Sheet.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Document:** placeholder “Google Sheet ID for responses”
  - **Sheet name:** `Responses`
  - **Mapping mode:** “define below” with explicit columns:
    - `name` = `{{$json.name}}`
    - `email` = `{{$json.email}}`
    - `interest` = `{{$json.interest}}`
    - `submitted_at` = `{{$json.submitted_at}}`
  - **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
- **Connections:**
  - **Input ←** Set Response Fields
  - **Output →** Route by Interest
- **Version requirements:** typeVersion **4.7**.
- **Edge cases / failures:**
  - Spreadsheet ID invalid/not shared with the OAuth account → 403/404 errors.
  - Sheet name “Responses” missing → append fails.
  - Column headers mismatch (e.g., `submitted_at` not present) → mapping/appending may fail or insert in unexpected columns depending on node behavior.
  - Google API quota/rate limits.

---

### Block 3 — Route by Interest & Send Notifications
**Overview:**  
Routes the entry based on `interest` keywords to Slack channels; also sends a confirmation email. In the current configuration, the email node is connected to the **fallback (“extra”) output**, meaning it triggers only when the interest is neither sales/pricing nor support/help.

**Nodes involved:**
- Route by Interest
- Notify Sales Channel
- Notify Support Channel
- Send Confirmation Email

#### Node: Route by Interest
- **Type / role:** `switch` — branching logic by string matching on `interest`.
- **Configuration (interpreted):**
  - **Ignore case:** enabled (`ignoreCase: true`)
  - **Fallback output:** enabled and labeled as **“extra”**
  - **Rule set (2 rules + fallback):**
    1) **Sales/Pricing route** if `interest`:
       - contains `"pricing"` OR contains `"sales"` OR equals `"Pricing"`
    2) **Support route** if `interest`:
       - contains `"support"` OR contains `"help"` OR equals `"Support"`
    3) **Fallback (“extra”)** if neither rule matches
- **Key expressions:**
  - Left value in conditions: `{{ $json.interest }}`
- **Connections:**
  - **Input ←** Log to Google Sheets
  - **Output 0 (Rule 1) →** Notify Sales Channel
  - **Output 1 (Rule 2) →** Notify Support Channel
  - **Fallback/extra →** Send Confirmation Email
- **Version requirements:** typeVersion **3.3**.
- **Edge cases / failures:**
  - If `interest` is empty, it will go to fallback.
  - If Typeform answer values don’t contain the expected keywords, routing won’t behave as intended.
  - Because **ignoreCase is true**, case differences are handled; however, extra whitespace or different phrasing may still miss.

#### Node: Notify Sales Channel
- **Type / role:** `slack` — posts a message to a Slack channel (sales/pricing).
- **Configuration (interpreted):**
  - Posts text:  
    `New pricing inquiry from {name} ({email}). Interest: {interest}`
  - Channel selected by **Channel ID placeholder**: “Sales Slack channel ID”
- **Key expressions:**
  - Message uses `{{ $json.name }}`, `{{ $json.email }}`, `{{ $json.interest }}`
- **Connections:**
  - **Input ←** Route by Interest (Rule 1 output)
  - **Output:** none (end of branch)
- **Version requirements:** typeVersion **2.3**.
- **Edge cases / failures:**
  - Missing/invalid Slack credentials or missing permissions to post to the channel.
  - Wrong channel ID.
  - Empty `name/email/interest` reduces message usefulness.

#### Node: Notify Support Channel
- **Type / role:** `slack` — posts a message to a Slack channel (support/help).
- **Configuration (interpreted):**
  - Posts text:  
    `New support request from {name} ({email}). Interest: {interest}`
  - Channel selected by **Channel ID placeholder**: “Support Slack channel ID”
- **Key expressions:** same variables as above.
- **Connections:**
  - **Input ←** Route by Interest (Rule 2 output)
  - **Output:** none (end of branch)
- **Version requirements:** typeVersion **2.3**.
- **Edge cases / failures:** same as Sales node.

#### Node: Send Confirmation Email
- **Type / role:** `gmail` — sends an email acknowledgement to the respondent.
- **Configuration (interpreted):**
  - **To:** `{{ $json.email }}`
  - **Subject:** `Thanks for reaching out!`
  - **Body:** templated with `{{ $json.name }}`
  - **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Connections:**
  - **Input ←** Route by Interest **fallback (“extra”) output**
  - **Output:** none (end of branch)
- **Version requirements:** typeVersion **2.1**.
- **Edge cases / failures:**
  - If `email` is blank/invalid → send fails.
  - Gmail OAuth consent/refresh token issues.
  - Sending limits/quota.
- **Important behavior note:**  
  The sticky note says *“Every respondent also receives a confirmation email”*, but the current wiring sends confirmations **only for fallback cases** (i.e., when interest doesn’t match sales/support rules). If you truly want “every respondent”, this node should be connected in a way that runs for all branches (see Section 4).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Typeform Trigger | Typeform Trigger | Receive Typeform submissions via webhook | — | Set Response Fields | ## Receive & Log<br>Typeform webhook receives submissions, extracts fields, and logs them to Google Sheets. |
| Set Response Fields | Set | Normalize/rename fields into stable schema | Typeform Trigger | Log to Google Sheets | ## Receive & Log<br>Typeform webhook receives submissions, extracts fields, and logs them to Google Sheets. |
| Log to Google Sheets | Google Sheets | Append submission to “Responses” sheet | Set Response Fields | Route by Interest | ## Receive & Log<br>Typeform webhook receives submissions, extracts fields, and logs them to Google Sheets. |
| Route by Interest | Switch | Branch to Slack channels or fallback | Log to Google Sheets | Notify Sales Channel; Notify Support Channel; Send Confirmation Email | ## Route & Notify<br>Routes responses to the right Slack channel based on interest and sends a confirmation email to every respondent. |
| Notify Sales Channel | Slack | Post pricing/sales inquiries to Sales channel | Route by Interest | — | ## Route & Notify<br>Routes responses to the right Slack channel based on interest and sends a confirmation email to every respondent. |
| Notify Support Channel | Slack | Post support/help inquiries to Support channel | Route by Interest | — | ## Route & Notify<br>Routes responses to the right Slack channel based on interest and sends a confirmation email to every respondent. |
| Send Confirmation Email | Gmail | Email confirmation to respondent | Route by Interest (fallback/extra) | — | ## Route & Notify<br>Routes responses to the right Slack channel based on interest and sends a confirmation email to every respondent. |
| Sticky Note | Sticky Note | Documentation | — | — | ## How it works<br>This workflow listens for new Typeform submissions via a webhook. When a response comes in, it extracts the key fields (name, email, interest), logs every submission to a Google Sheets spreadsheet, then routes the response based on what the person is interested in. Pricing inquiries go to your sales Slack channel, support questions go to the support channel, and all other submissions get a default fallback. Every respondent also receives a confirmation email via Gmail.<br><br>## Setup steps<br>1. Connect your Typeform account and set the form ID in the trigger node<br>2. Adjust the field mappings in the Set Response Fields node to match your form's question labels<br>3. Connect your Google Sheets account and set the spreadsheet ID for logging<br>4. Create a sheet called "Responses" with columns: name, email, interest, submitted_at<br>5. Connect your Slack workspace and set the channel IDs for sales and support<br>6. Adjust the Switch node conditions to match your form's answer values<br>7. Connect your Gmail account for sending confirmation emails<br>8. Activate the workflow |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Receive & Log<br>Typeform webhook receives submissions, extracts fields, and logs them to Google Sheets. |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Route & Notify<br>Routes responses to the right Slack channel based on interest and sends a confirmation email to every respondent. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Ensure workflow setting **Execution Order** is `v1` (default in many instances).

2) **Add node: “Typeform Trigger”**  
   - Node type: **Typeform Trigger**  
   - Configure:
     - **Form ID:** paste your Typeform form ID
   - Credentials:
     - Connect your **Typeform** account credentials in n8n
   - This node will create/manage a webhook subscription in Typeform when the workflow is activated.

3) **Add node: “Set Response Fields”** (Set node)  
   - Node type: **Set**
   - Add fields (as String):
     - `name` = `{{ $json["What is your name?"] || $json["Name"] || "" }}`
     - `email` = `{{ $json["What is your email?"] || $json["Email"] || "" }}`
     - `interest` = `{{ $json["What are you interested in?"] || $json["Interest"] || "" }}`
     - `submitted_at` = `{{ $json.submittedAt || $now.toISO() }}`
   - Connect: **Typeform Trigger → Set Response Fields**

4) **Add node: “Log to Google Sheets”**  
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Document: select by **Spreadsheet ID** (paste your Google Sheet ID)
   - Sheet name: `Responses`
   - Columns mapping (define manually):
     - `name` → `{{ $json.name }}`
     - `email` → `{{ $json.email }}`
     - `interest` → `{{ $json.interest }}`
     - `submitted_at` → `{{ $json.submitted_at }}`
   - Credentials:
     - Connect **Google Sheets OAuth2**
   - In Google Sheets, create a tab named **Responses** with headers:  
     `name | email | interest | submitted_at`
   - Connect: **Set Response Fields → Log to Google Sheets**

5) **Add node: “Route by Interest”** (Switch node)  
   - Node type: **Switch**
   - Set **Ignore Case** = enabled
   - Set **Fallback Output** = enabled (extra)
   - Create Rule 1 (Sales/Pricing), on value `{{ $json.interest }}`:
     - contains `pricing`
     - OR contains `sales`
     - OR equals `Pricing`
   - Create Rule 2 (Support), on value `{{ $json.interest }}`:
     - contains `support`
     - OR contains `help`
     - OR equals `Support`
   - Connect: **Log to Google Sheets → Route by Interest**

6) **Add node: “Notify Sales Channel”** (Slack)  
   - Node type: **Slack**
   - Operation: post message to channel (message/text)
   - Channel: select by **Channel ID** (paste Sales channel ID)
   - Text:
     - `New pricing inquiry from {{ $json.name }} ({{ $json.email }}). Interest: {{ $json.interest }}`
   - Credentials:
     - Connect **Slack OAuth2** (or Slack credentials appropriate to your n8n setup)
   - Connect: **Route by Interest (Rule 1 output) → Notify Sales Channel**

7) **Add node: “Notify Support Channel”** (Slack)  
   - Node type: **Slack**
   - Channel ID: paste Support channel ID
   - Text:
     - `New support request from {{ $json.name }} ({{ $json.email }}). Interest: {{ $json.interest }}`
   - Connect: **Route by Interest (Rule 2 output) → Notify Support Channel**

8) **Add node: “Send Confirmation Email”** (Gmail)  
   - Node type: **Gmail**
   - To: `{{ $json.email }}`
   - Subject: `Thanks for reaching out!`
   - Message body:
     - `Hi {{ $json.name }},`  
       `Thank you for your submission...`
   - Credentials:
     - Connect **Gmail OAuth2**
   - Connect (to match the provided workflow exactly):
     - **Route by Interest (fallback/extra output) → Send Confirmation Email**

9) **(Recommended adjustment if you truly want email to go to everyone)**  
   The current design sends email only on fallback. To send confirmation to *all* respondents, you can:
   - Add a **Merge** node (e.g., “Merge by Pass-through”) after Slack branches and fallback, then connect Merge → Gmail, or
   - Place Gmail earlier (right after Google Sheets) and keep Slack routing separate, or
   - Duplicate Gmail node on each Switch output (Sales, Support, Fallback).

10) **Activate the workflow**  
   - Activation will register the Typeform webhook so new submissions trigger the flow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer (provided) |
| “Every respondent also receives a confirmation email via Gmail.” | Sticky note description; note that current wiring sends email only via Switch fallback (“extra”). |
| Setup guidance (Typeform form ID, field mapping, spreadsheet columns, Slack channel IDs, Switch conditions, Gmail OAuth, activate workflow) | Sticky note “Setup steps” embedded in the workflow |

