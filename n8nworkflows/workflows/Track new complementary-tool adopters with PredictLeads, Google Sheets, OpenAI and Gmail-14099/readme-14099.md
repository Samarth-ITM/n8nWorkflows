Track new complementary-tool adopters with PredictLeads, Google Sheets, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/track-new-complementary-tool-adopters-with-predictleads--google-sheets--openai-and-gmail-14099


# Track new complementary-tool adopters with PredictLeads, Google Sheets, OpenAI and Gmail

# 1. Workflow Overview

This workflow monitors companies that recently adopted technologies complementary to your product, enriches those companies with PredictLeads data, generates a co-marketing outreach email with OpenAI, sends the email via Gmail, and logs the outreach in Google Sheets to prevent duplicate processing later.

Typical use case: you maintain a list of partner-relevant tools such as HubSpot, Shopify, Segment, or other platforms that work well with your product. Each day, the workflow checks for newly detected adopters of those tools and sends a personalized partnership or co-marketing message.

## 1.1 Trigger & Tool Source

The workflow starts on a daily schedule and reads a Google Sheet containing the list of complementary tools to monitor. Each row is expected to provide at least a human-readable tool name and a PredictLeads technology ID.

## 1.2 Discovery & Change Detection

The workflow iterates through the tools one by one, requests recent technology detection data from PredictLeads, loads historical scan data from another Google Sheet tab, and compares current detections with previously seen domains. Only domains not already present in the historical log are treated as new adopters.

## 1.3 Company Enrichment & AI Email Drafting

For each newly detected adopter, the workflow optionally limits throughput, fetches company details from PredictLeads, constructs a prompt with company and adoption context, and sends that prompt to OpenAI to generate a concise co-marketing outreach email.

## 1.4 Email Sending & Historical Logging

The AI-generated email is sent through Gmail. After sending, the workflow appends the processed domain, associated tool, timestamp, and send status to the historical Google Sheet so the same adopter will not be contacted again in future executions.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Tool Source

### Overview

This block defines when the workflow runs and where the monitored technology list comes from. It establishes the source dataset for the rest of the process.

### Nodes Involved

- ⏰ Daily Schedule
- 📑 Read Complementary Tools List

### Node Details

#### ⏰ Daily Schedule
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node that launches the workflow automatically.
- **Configuration choices:**  
  Configured to run on a recurring schedule at hour 8 each day.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input; outputs to `📑 Read Complementary Tools List`.
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Workflow must be active for scheduled execution.
  - Server timezone may affect actual trigger time.
  - If n8n instance is offline at trigger time, execution may be missed depending on deployment setup.
- **Sub-workflow reference:**  
  None.

#### 📑 Read Complementary Tools List
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheet tab containing tools to monitor.
- **Configuration choices:**  
  - Uses Google Sheets OAuth2 credentials.
  - Reads from spreadsheet `YOUR_GOOGLE_SHEET_ID_05`.
  - Reads tab `gid=0`, labeled `Complementary Tools`.
  - No extra filtering or options configured.
- **Key expressions or variables used:**  
  No custom expressions in visible parameters, but downstream logic expects fields such as:
  - `tool_name`
  - `tech_id`
- **Input and output connections:**  
  Input from `⏰ Daily Schedule`; output to `🔀 Split by Tool`.
- **Version-specific requirements:**  
  Type version `4.5`.
- **Edge cases or potential failure types:**  
  - OAuth permission issues.
  - Sheet not found or renamed.
  - Missing required columns like `tool_name` or `tech_id`.
  - Empty sheet results in no downstream processing.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Discovery & Change Detection

### Overview

This block loops through each monitored tool, calls PredictLeads to detect adopters, loads prior scan history from Google Sheets, and compares the two datasets. It outputs only genuinely new adopter domains.

### Nodes Involved

- 🔀 Split by Tool
- 🔍 Discover Technology Adopters
- 📑 Read Previous Scan Data
- ⚙️ Compare & Detect New Adoptions
- ✅ New Adoption Detected?

### Node Details

#### 🔀 Split by Tool
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the complementary tool list item by item.
- **Configuration choices:**  
  Default batching behavior; no custom batch size specified.
- **Key expressions or variables used:**  
  Downstream nodes reference current item fields with:
  - `$('🔀 Split by Tool').first().json.tool_name`
  - `$('🔀 Split by Tool').first().json.tech_id`
- **Input and output connections:**  
  Input from:
  - `📑 Read Complementary Tools List`
  - Loop-back from `✅ New Adoption Detected?` false branch
  - Loop-back from `📑 Update Previous Scan Data`
  
  Output:
  - Second/main batch output to `🔍 Discover Technology Adopters`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If the input sheet has malformed rows, the current batch item may not include `tech_id`.
  - Using `.first()` downstream works only if execution context matches expectations; in some redesigns `.item` may be safer.
- **Sub-workflow reference:**  
  None.

#### 🔍 Discover Technology Adopters
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the PredictLeads Technology Detections endpoint for the current technology ID.
- **Configuration choices:**  
  - GET request to:
    `https://predictleads.com/api/v3/discover/technologies/{{ $json.tech_id }}/technology_detections`
  - Sends headers:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - URL expression uses `{{$json.tech_id}}`
- **Input and output connections:**  
  Input from `🔀 Split by Tool`; output to `📑 Read Previous Scan Data`.
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid or expired PredictLeads credentials.
  - `tech_id` missing or invalid.
  - Rate limits or API quota exhaustion.
  - Non-200 responses.
  - Response shape changes; the code node expects `.json.data`.
- **Sub-workflow reference:**  
  None.

#### 📑 Read Previous Scan Data
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads the historical log of previously processed domains.
- **Configuration choices:**  
  - Uses same Google Sheets OAuth2 credential.
  - Reads from tab `Previous Scan` (`gid=423252497`) in the same spreadsheet.
  - `alwaysOutputData: true` is enabled.
- **Key expressions or variables used:**  
  Downstream logic expects a `domain` column in this sheet.
- **Input and output connections:**  
  Input from `🔍 Discover Technology Adopters`; output to `⚙️ Compare & Detect New Adoptions`.
- **Version-specific requirements:**  
  Type version `4.5`.
- **Edge cases or potential failure types:**  
  - Missing or empty `Previous Scan` tab.
  - Domain column absent or inconsistently named.
  - Large sheets may increase execution time.
  - `alwaysOutputData` helps when there are no rows, but code must still handle empty arrays.
- **Sub-workflow reference:**  
  None.

#### ⚙️ Compare & Detect New Adoptions
- **Type and technical role:** `n8n-nodes-base.code`  
  Compares current PredictLeads detections against historical domains and emits only unseen domains.
- **Configuration choices:**  
  JavaScript code performs:
  1. Read current detections from `🔍 Discover Technology Adopters`
  2. Read all previous scan rows from `📑 Read Previous Scan Data`
  3. Build a `Set` of historical domains
  4. Filter out already-seen domains
  5. Return normalized items with:
     - `domain`
     - `tool_name`
     - `tech_id`
     - `detected_at`
     - `is_new: true`
- **Key expressions or variables used:**  
  - `$('🔍 Discover Technology Adopters').first().json.data || []`
  - `$('📑 Read Previous Scan Data').all()`
  - `$('🔀 Split by Tool').first().json.tool_name`
  - `$('🔀 Split by Tool').first().json.tech_id`
  - `detection.attributes?.domain`
  - `detection.attributes?.first_detected_at`
- **Input and output connections:**  
  Input from `📑 Read Previous Scan Data`; output to `✅ New Adoption Detected?`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If PredictLeads response does not contain `data`, output becomes empty.
  - If domain values differ only in casing or formatting, duplicates may slip through.
  - If `Previous Scan` contains blank rows, they are filtered out.
  - If `tool_name` or `tech_id` is missing on the source row, returned items will be incomplete.
- **Sub-workflow reference:**  
  None.

#### ✅ New Adoption Detected?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches execution based on whether the current item was marked as a new adoption.
- **Configuration choices:**  
  Checks boolean condition:
  - `{{$json.is_new}}` is `true`
- **Key expressions or variables used:**  
  - `={{ $json.is_new }}`
- **Input and output connections:**  
  Input from `⚙️ Compare & Detect New Adoptions`
  
  Outputs:
  - **True branch:** `Limit`
  - **False branch:** `🔀 Split by Tool`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If code node returns no items, this node is skipped.
  - Strict boolean validation means a string like `"true"` would not match; the code correctly emits real boolean `true`.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Company Enrichment & AI Outreach Preparation

### Overview

This block constrains throughput, enriches each detected company, constructs the AI prompt, and requests an email draft from OpenAI. It transforms raw adoption data into personalized outreach copy.

### Nodes Involved

- Limit
- 🔍 Enrich Company
- ⚙️ Build AI Prompt
- 🤖 Draft Co-Marketing Email

### Node Details

#### Limit
- **Type and technical role:** `n8n-nodes-base.limit`  
  Restricts how many new adopters are processed in a run.
- **Configuration choices:**  
  `maxItems: 2`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from `✅ New Adoption Detected?` true branch; output to `🔍 Enrich Company`.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Additional new adopters beyond the first two are ignored for that execution.
  - This may delay outreach if there are many new detections.
- **Sub-workflow reference:**  
  None.

#### 🔍 Enrich Company
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the PredictLeads company endpoint to retrieve company-level metadata.
- **Configuration choices:**  
  - GET request to:
    `https://predictleads.com/api/v3/companies/{{ $('📑 Read Previous Scan Data').first().json.domain }}`
  - Sends same PredictLeads headers as earlier.
- **Key expressions or variables used:**  
  - URL expression references `$('📑 Read Previous Scan Data').first().json.domain`
- **Input and output connections:**  
  Input from `Limit`; output to `⚙️ Build AI Prompt`.
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - **Important logic issue:** this node appears to reference the first row from `📑 Read Previous Scan Data`, not the current new adopter domain from upstream. This likely causes incorrect enrichment and should probably use the current item domain instead, e.g. `{{$json.domain}}`.
  - API auth failures or quota issues.
  - Company may not exist in PredictLeads.
  - Response schema may differ from expectations in the prompt-building node.
- **Sub-workflow reference:**  
  None.

#### ⚙️ Build AI Prompt
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts enriched company data into an OpenAI-ready prompt.
- **Configuration choices:**  
  JavaScript code:
  - Reads `tool_name` from `⚙️ Compare & Detect New Adoptions`
  - Iterates over input items
  - Pulls company attributes from `item.json.data?.[0]?.attributes`
  - Builds a prompt instructing the AI to write a concise co-marketing email
  - Emits structured fields:
    - `prompt`
    - `company_name`
    - `domain`
    - `tool_name`
    - `industry`
    - `employee_count`
- **Key expressions or variables used:**  
  - `$('⚙️ Compare & Detect New Adoptions').first().json.tool_name`
  - `item.json.data?.[0]?.attributes`
  - Fallbacks like:
    - `company.company_name || company.friendly_company_name || domain`
    - `company.industry || 'technology'`
    - `company.employee_count || 'unknown'`
- **Input and output connections:**  
  Input from `🔍 Enrich Company`; output to `🤖 Draft Co-Marketing Email`.
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If the enrichment response is empty, the prompt still builds but with poor or blank company context.
  - Using `.first()` for `tool_name` may mismatch in multi-item scenarios.
  - If `data[0].attributes` does not exist, many fields default to empty strings or fallback values.
- **Sub-workflow reference:**  
  None.

#### 🤖 Draft Co-Marketing Email
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the generated prompt to OpenAI Chat Completions to produce email copy.
- **Configuration choices:**  
  - POST to `https://api.openai.com/v1/chat/completions`
  - JSON body includes:
    - `model: gpt-4o-mini`
    - system instruction defining role
    - user message containing prompt
    - `temperature: 0.7`
    - `max_tokens: 500`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.prompt) }}`
- **Input and output connections:**  
  Input from `⚙️ Build AI Prompt`; output to `📧 Send Co-Marketing Email`.
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid OpenAI API token.
  - Model availability or billing restrictions.
  - Rate limits.
  - Prompt formatting issues if JSON body interpolation is malformed.
  - Response parser downstream expects `choices[0].message.content`.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Outreach & Data Logging

### Overview

This block delivers the generated email and appends a historical record to Google Sheets. It also closes the loop by feeding the batch iterator so the next tool can be processed.

### Nodes Involved

- 📧 Send Co-Marketing Email
- 📑 Update Previous Scan Data

### Node Details

#### 📧 Send Co-Marketing Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the AI-generated outreach email through a Gmail account.
- **Configuration choices:**  
  - Uses Gmail OAuth2 credentials.
  - Recipient is dynamically built as:
    `{{ $('⚙️ Build AI Prompt').item.json.domain.split('.')[0] + '@example.com' }}`
  - Subject:
    `Co-Marketing Partnership Opportunity - {{ $('⚙️ Build AI Prompt').item.json.company_name }}`
  - Message body:
    `{{$json.choices[0].message.content}}`
  - Email type: text
- **Key expressions or variables used:**  
  - `$('⚙️ Build AI Prompt').item.json.domain.split('.')[0] + '@example.com'`
  - `$json.choices[0].message.content`
  - `$('⚙️ Build AI Prompt').item.json.company_name`
- **Input and output connections:**  
  Input from `🤖 Draft Co-Marketing Email`; output to `📑 Update Previous Scan Data`.
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - **Important logic issue:** recipient generation is placeholder logic, not a real contact discovery mechanism. `acme.com` becomes `acme@example.com`, which is not a valid production target in most cases.
  - Gmail OAuth token expiration or missing send permissions.
  - If OpenAI output is missing, message body may fail.
  - Domain parsing breaks on empty or malformed domains.
- **Sub-workflow reference:**  
  None.

#### 📑 Update Previous Scan Data
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the processed outreach result to the historical scan sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Writes to `Previous Scan` tab in the same spreadsheet.
  - Maps columns:
    - `domain` from `⚙️ Build AI Prompt`
    - `tool_name` from `⚙️ Build AI Prompt`
    - `email_sent` fixed to `"true"`
    - `detected_at` from `$now.toISO()`
- **Key expressions or variables used:**  
  - `={{ $('⚙️ Build AI Prompt').item.json.domain }}`
  - `={{ $('⚙️ Build AI Prompt').item.json.tool_name }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  Input from `📧 Send Co-Marketing Email`; output loops back to `🔀 Split by Tool`.
- **Version-specific requirements:**  
  Type version `4.5`.
- **Edge cases or potential failure types:**  
  - OAuth access issues.
  - Column schema mismatch in Google Sheets.
  - If email sending fails before this node, no history is written.
  - Logging uses current timestamp, not the original PredictLeads detection timestamp.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| About This Workflow | Sticky Note | Context and setup notes |  |  | ABOUT THIS WORKFLOW<br>Tracks companies adopting tools that complement yours and sends AI-drafted co-marketing outreach emails to new adopters.<br>Setup: Google Sheet with tool names and PredictLeads tech IDs, Gmail OAuth2, OpenAI API key, PredictLeads API.<br>Use case: You have an analytics tool that pairs well with HubSpot. This detects new HubSpot adopters and sends a partnership pitch offering a joint webinar or integration guide.<br>PredictLeads API: https://predictleads.com<br>Questions: https://www.linkedin.com/in/yaronbeen |
| 📋 Trigger & Input | Sticky Note | Visual documentation for trigger block |  |  | ## 1️⃣ Trigger & Tool Source<br>**Nodes:** ⏰ Daily Schedule → 📑 Read Complementary Tools List<br>**Description:** The workflow starts with a scheduled trigger that runs every day. It reads a list of complementary tools from Google Sheets. Each row contains the **tool name** and **PredictLeads technology ID**.<br>This sheet acts as the source dataset that defines which technologies should be monitored. The goal is to detect companies that start using these tools because they may be strong **co-marketing or partnership prospects**. |
| 🔄 Discovery & Change Detection | Sticky Note | Visual documentation for detection block |  |  | ## 2️⃣ Technology Discovery & Change Detection<br>**Nodes:** 🔀 Split by Tool → 🔍 Discover Technology Adopters → 📑 Read Previous Scan Data → ⚙️ Compare & Detect New Adoptions → ✅ New Adoption Detected?<br>**Description:** Each tool from the sheet is processed individually using **Split in Batches**.<br>The workflow calls the **PredictLeads Technology Detections API** to retrieve companies that recently adopted the specified technology. It then loads previously scanned domains from another Google Sheet tab to avoid duplicate processing.<br>A code node compares the newly detected companies with the previously stored domains and identifies **new technology adopters**. Only companies that appear for the first time are allowed to continue through the workflow. |
| 🤖 AI Outreach | Sticky Note | Visual documentation for AI block |  |  | ## 3️⃣ Company Enrichment & AI Outreach Preparation<br>**Nodes:** Limit → 🔍 Enrich Company → ⚙️ Build AI Prompt → 🤖 Draft Co-Marketing Email<br>**Description:** The workflow optionally limits the number of companies processed per run to control outreach volume.<br>For each new adopter, additional company details are fetched from the **PredictLeads company API**. This enriched data is used to generate a structured prompt describing the company, the detected tool adoption, and the potential partnership context.<br>The prompt is then sent to **OpenAI**, which generates a personalized co-marketing outreach email tailored to the company. |
| 📤 Output | Sticky Note | Visual documentation for output block |  |  | ## 4️⃣ Outreach & Data Logging<br>**Nodes:** 📧 Send Co-Marketing Email → 📑 Update Previous Scan Data<br>**Description:** The AI-generated email is sent through Gmail to the target company.<br>After sending the email, the workflow logs the domain, tool name, detection timestamp, and email status into Google Sheets. This log acts as a **historical database of detected adopters**, ensuring the workflow does not send duplicate outreach emails in future runs. |
| ⏰ Daily Schedule | Schedule Trigger | Daily workflow trigger |  | 📑 Read Complementary Tools List | ## 1️⃣ Trigger & Tool Source<br>**Nodes:** ⏰ Daily Schedule → 📑 Read Complementary Tools List<br>**Description:** The workflow starts with a scheduled trigger that runs every day. It reads a list of complementary tools from Google Sheets. Each row contains the **tool name** and **PredictLeads technology ID**.<br>This sheet acts as the source dataset that defines which technologies should be monitored. The goal is to detect companies that start using these tools because they may be strong **co-marketing or partnership prospects**. |
| 📑 Read Complementary Tools List | Google Sheets | Read monitored technology list | ⏰ Daily Schedule | 🔀 Split by Tool | ## 1️⃣ Trigger & Tool Source<br>**Nodes:** ⏰ Daily Schedule → 📑 Read Complementary Tools List<br>**Description:** The workflow starts with a scheduled trigger that runs every day. It reads a list of complementary tools from Google Sheets. Each row contains the **tool name** and **PredictLeads technology ID**.<br>This sheet acts as the source dataset that defines which technologies should be monitored. The goal is to detect companies that start using these tools because they may be strong **co-marketing or partnership prospects**. |
| 🔀 Split by Tool | Split In Batches | Iterate through each tool row | 📑 Read Complementary Tools List; ✅ New Adoption Detected?; 📑 Update Previous Scan Data | 🔍 Discover Technology Adopters | ## 2️⃣ Technology Discovery & Change Detection<br>**Nodes:** 🔀 Split by Tool → 🔍 Discover Technology Adopters → 📑 Read Previous Scan Data → ⚙️ Compare & Detect New Adoptions → ✅ New Adoption Detected?<br>**Description:** Each tool from the sheet is processed individually using **Split in Batches**.<br>The workflow calls the **PredictLeads Technology Detections API** to retrieve companies that recently adopted the specified technology. It then loads previously scanned domains from another Google Sheet tab to avoid duplicate processing.<br>A code node compares the newly detected companies with the previously stored domains and identifies **new technology adopters**. Only companies that appear for the first time are allowed to continue through the workflow. |
| 🔍 Discover Technology Adopters | HTTP Request | Query PredictLeads technology detections | 🔀 Split by Tool | 📑 Read Previous Scan Data | ## 2️⃣ Technology Discovery & Change Detection<br>**Nodes:** 🔀 Split by Tool → 🔍 Discover Technology Adopters → 📑 Read Previous Scan Data → ⚙️ Compare & Detect New Adoptions → ✅ New Adoption Detected?<br>**Description:** Each tool from the sheet is processed individually using **Split in Batches**.<br>The workflow calls the **PredictLeads Technology Detections API** to retrieve companies that recently adopted the specified technology. It then loads previously scanned domains from another Google Sheet tab to avoid duplicate processing.<br>A code node compares the newly detected companies with the previously stored domains and identifies **new technology adopters**. Only companies that appear for the first time are allowed to continue through the workflow. |
| 📑 Read Previous Scan Data | Google Sheets | Load historical processed domains | 🔍 Discover Technology Adopters | ⚙️ Compare & Detect New Adoptions | ## 2️⃣ Technology Discovery & Change Detection<br>**Nodes:** 🔀 Split by Tool → 🔍 Discover Technology Adopters → 📑 Read Previous Scan Data → ⚙️ Compare & Detect New Adoptions → ✅ New Adoption Detected?<br>**Description:** Each tool from the sheet is processed individually using **Split in Batches**.<br>The workflow calls the **PredictLeads Technology Detections API** to retrieve companies that recently adopted the specified technology. It then loads previously scanned domains from another Google Sheet tab to avoid duplicate processing.<br>A code node compares the newly detected companies with the previously stored domains and identifies **new technology adopters**. Only companies that appear for the first time are allowed to continue through the workflow. |
| ⚙️ Compare & Detect New Adoptions | Code | Filter out previously seen domains | 📑 Read Previous Scan Data | ✅ New Adoption Detected? | ## 2️⃣ Technology Discovery & Change Detection<br>**Nodes:** 🔀 Split by Tool → 🔍 Discover Technology Adopters → 📑 Read Previous Scan Data → ⚙️ Compare & Detect New Adoptions → ✅ New Adoption Detected?<br>**Description:** Each tool from the sheet is processed individually using **Split in Batches**.<br>The workflow calls the **PredictLeads Technology Detections API** to retrieve companies that recently adopted the specified technology. It then loads previously scanned domains from another Google Sheet tab to avoid duplicate processing.<br>A code node compares the newly detected companies with the previously stored domains and identifies **new technology adopters**. Only companies that appear for the first time are allowed to continue through the workflow. |
| ✅ New Adoption Detected? | IF | Allow only new detections forward | ⚙️ Compare & Detect New Adoptions | Limit; 🔀 Split by Tool | ## 2️⃣ Technology Discovery & Change Detection<br>**Nodes:** 🔀 Split by Tool → 🔍 Discover Technology Adopters → 📑 Read Previous Scan Data → ⚙️ Compare & Detect New Adoptions → ✅ New Adoption Detected?<br>**Description:** Each tool from the sheet is processed individually using **Split in Batches**.<br>The workflow calls the **PredictLeads Technology Detections API** to retrieve companies that recently adopted the specified technology. It then loads previously scanned domains from another Google Sheet tab to avoid duplicate processing.<br>A code node compares the newly detected companies with the previously stored domains and identifies **new technology adopters**. Only companies that appear for the first time are allowed to continue through the workflow. |
| Limit | Limit | Cap processing volume per run | ✅ New Adoption Detected? | 🔍 Enrich Company | ## 3️⃣ Company Enrichment & AI Outreach Preparation<br>**Nodes:** Limit → 🔍 Enrich Company → ⚙️ Build AI Prompt → 🤖 Draft Co-Marketing Email<br>**Description:** The workflow optionally limits the number of companies processed per run to control outreach volume.<br>For each new adopter, additional company details are fetched from the **PredictLeads company API**. This enriched data is used to generate a structured prompt describing the company, the detected tool adoption, and the potential partnership context.<br>The prompt is then sent to **OpenAI**, which generates a personalized co-marketing outreach email tailored to the company. |
| 🔍 Enrich Company | HTTP Request | Retrieve company profile from PredictLeads | Limit | ⚙️ Build AI Prompt | ## 3️⃣ Company Enrichment & AI Outreach Preparation<br>**Nodes:** Limit → 🔍 Enrich Company → ⚙️ Build AI Prompt → 🤖 Draft Co-Marketing Email<br>**Description:** The workflow optionally limits the number of companies processed per run to control outreach volume.<br>For each new adopter, additional company details are fetched from the **PredictLeads company API**. This enriched data is used to generate a structured prompt describing the company, the detected tool adoption, and the potential partnership context.<br>The prompt is then sent to **OpenAI**, which generates a personalized co-marketing outreach email tailored to the company. |
| ⚙️ Build AI Prompt | Code | Build personalized OpenAI prompt | 🔍 Enrich Company | 🤖 Draft Co-Marketing Email | ## 3️⃣ Company Enrichment & AI Outreach Preparation<br>**Nodes:** Limit → 🔍 Enrich Company → ⚙️ Build AI Prompt → 🤖 Draft Co-Marketing Email<br>**Description:** The workflow optionally limits the number of companies processed per run to control outreach volume.<br>For each new adopter, additional company details are fetched from the **PredictLeads company API**. This enriched data is used to generate a structured prompt describing the company, the detected tool adoption, and the potential partnership context.<br>The prompt is then sent to **OpenAI**, which generates a personalized co-marketing outreach email tailored to the company. |
| 🤖 Draft Co-Marketing Email | HTTP Request | Generate AI email draft with OpenAI | ⚙️ Build AI Prompt | 📧 Send Co-Marketing Email | ## 3️⃣ Company Enrichment & AI Outreach Preparation<br>**Nodes:** Limit → 🔍 Enrich Company → ⚙️ Build AI Prompt → 🤖 Draft Co-Marketing Email<br>**Description:** The workflow optionally limits the number of companies processed per run to control outreach volume.<br>For each new adopter, additional company details are fetched from the **PredictLeads company API**. This enriched data is used to generate a structured prompt describing the company, the detected tool adoption, and the potential partnership context.<br>The prompt is then sent to **OpenAI**, which generates a personalized co-marketing outreach email tailored to the company. |
| 📧 Send Co-Marketing Email | Gmail | Send drafted outreach email | 🤖 Draft Co-Marketing Email | 📑 Update Previous Scan Data | ## 4️⃣ Outreach & Data Logging<br>**Nodes:** 📧 Send Co-Marketing Email → 📑 Update Previous Scan Data<br>**Description:** The AI-generated email is sent through Gmail to the target company.<br>After sending the email, the workflow logs the domain, tool name, detection timestamp, and email status into Google Sheets. This log acts as a **historical database of detected adopters**, ensuring the workflow does not send duplicate outreach emails in future runs. |
| 📑 Update Previous Scan Data | Google Sheets | Append processed outreach log | 📧 Send Co-Marketing Email | 🔀 Split by Tool | ## 4️⃣ Outreach & Data Logging<br>**Nodes:** 📧 Send Co-Marketing Email → 📑 Update Previous Scan Data<br>**Description:** The AI-generated email is sent through Gmail to the target company.<br>After sending the email, the workflow logs the domain, tool name, detection timestamp, and email status into Google Sheets. This log acts as a **historical database of detected adopters**, ensuring the workflow does not send duplicate outreach emails in future runs. |

---

# 4. Reproducing the Workflow from Scratch

## Prerequisites

1. Create a Google Sheet with at least two tabs:
   - **Complementary Tools**
   - **Previous Scan**

2. In **Complementary Tools**, create at least these columns:
   - `tool_name`
   - `tech_id`

3. In **Previous Scan**, create at least these columns:
   - `domain`
   - `tool_name`
   - `detected_at`
   - `email_sent`

4. Prepare credentials:
   - **Google Sheets OAuth2**
   - **Gmail OAuth2**
   - **PredictLeads API key and token**
   - **OpenAI API key**

---

## Build Steps

1. **Create a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name it: `⏰ Daily Schedule`
   - Set it to run daily at hour `8`
   - This is the workflow entry point.

2. **Create a Google Sheets node to read tools**
   - Node type: `Google Sheets`
   - Name it: `📑 Read Complementary Tools List`
   - Connect `⏰ Daily Schedule` → `📑 Read Complementary Tools List`
   - Authenticate with Google Sheets OAuth2
   - Select your spreadsheet
   - Select the `Complementary Tools` tab
   - Use default read behavior

3. **Create a Split In Batches node**
   - Node type: `Split In Batches`
   - Name it: `🔀 Split by Tool`
   - Connect `📑 Read Complementary Tools List` → `🔀 Split by Tool`
   - Leave default batch settings unless you want custom batching

4. **Create an HTTP Request node for PredictLeads technology detections**
   - Node type: `HTTP Request`
   - Name it: `🔍 Discover Technology Adopters`
   - Connect `🔀 Split by Tool` main batch output → `🔍 Discover Technology Adopters`
   - Method: `GET`
   - URL:
     `https://predictleads.com/api/v3/discover/technologies/{{ $json.tech_id }}/technology_detections`
   - Enable headers
   - Add headers:
     - `X-Api-Key` = your PredictLeads API key
     - `X-Api-Token` = your PredictLeads API token
     - `Content-Type` = `application/json`

5. **Create a Google Sheets node to read historical scan data**
   - Node type: `Google Sheets`
   - Name it: `📑 Read Previous Scan Data`
   - Connect `🔍 Discover Technology Adopters` → `📑 Read Previous Scan Data`
   - Use the same Google Sheets credential
   - Select the same spreadsheet
   - Select the `Previous Scan` tab
   - Enable setting equivalent to always returning data if available in your n8n version

6. **Create a Code node to compare current and previous domains**
   - Node type: `Code`
   - Name it: `⚙️ Compare & Detect New Adoptions`
   - Connect `📑 Read Previous Scan Data` → `⚙️ Compare & Detect New Adoptions`
   - Paste logic equivalent to:
     - Read current detections from `🔍 Discover Technology Adopters`
     - Read all rows from `📑 Read Previous Scan Data`
     - Build a set of previously seen `domain` values
     - Return only detections whose domains are not yet in the set
     - Include these fields in output:
       - `domain`
       - `tool_name`
       - `tech_id`
       - `detected_at`
       - `is_new: true`

   Suggested implementation pattern:
   - Current detections from response `.json.data`
   - Previous domains from Google Sheet rows `.json.domain`
   - Source tool metadata from current `🔀 Split by Tool` item

7. **Create an IF node**
   - Node type: `If`
   - Name it: `✅ New Adoption Detected?`
   - Connect `⚙️ Compare & Detect New Adoptions` → `✅ New Adoption Detected?`
   - Add boolean condition:
     - Left value: `{{ $json.is_new }}`
     - Operation: `is true`

8. **Loop the false branch back to the batch node**
   - Connect `✅ New Adoption Detected?` false output → `🔀 Split by Tool`
   - This allows iteration to continue when nothing new was found

9. **Create a Limit node**
   - Node type: `Limit`
   - Name it: `Limit`
   - Connect `✅ New Adoption Detected?` true output → `Limit`
   - Set `Max Items` to `2`
   - This caps the number of adopters processed per run

10. **Create an HTTP Request node for company enrichment**
    - Node type: `HTTP Request`
    - Name it: `🔍 Enrich Company`
    - Connect `Limit` → `🔍 Enrich Company`
    - Method: `GET`
    - Add PredictLeads headers same as before
    - Recommended URL for a correct rebuild:
      `https://predictleads.com/api/v3/companies/{{ $json.domain }}`
    - Note: the provided workflow uses `$('📑 Read Previous Scan Data').first().json.domain`, which is likely a mistake. Use the current item domain instead.

11. **Create a Code node to build the AI prompt**
    - Node type: `Code`
    - Name it: `⚙️ Build AI Prompt`
    - Connect `🔍 Enrich Company` → `⚙️ Build AI Prompt`
    - Build output fields:
      - `prompt`
      - `company_name`
      - `domain`
      - `tool_name`
      - `industry`
      - `employee_count`
    - Prompt should instruct OpenAI to:
      - produce a concise co-marketing outreach email
      - include a subject line
      - mention the complementary product relationship
      - propose a specific collaboration idea
      - stay under 200 words
      - end with a CTA

12. **Create an HTTP Request node for OpenAI**
    - Node type: `HTTP Request`
    - Name it: `🤖 Draft Co-Marketing Email`
    - Connect `⚙️ Build AI Prompt` → `🤖 Draft Co-Marketing Email`
    - Method: `POST`
    - URL: `https://api.openai.com/v1/chat/completions`
    - Specify body as JSON
    - Add headers:
      - `Authorization` = `Bearer YOUR_OPENAI_API_KEY`
      - `Content-Type` = `application/json`
    - Use body fields:
      - `model`: `gpt-4o-mini`
      - `messages`:
        - system role describing B2B co-marketing email behavior
        - user role containing the prompt
      - `temperature`: `0.7`
      - `max_tokens`: `500`

13. **Create a Gmail node**
    - Node type: `Gmail`
    - Name it: `📧 Send Co-Marketing Email`
    - Connect `🤖 Draft Co-Marketing Email` → `📧 Send Co-Marketing Email`
    - Authenticate with Gmail OAuth2
    - Set email type to `text`
    - Subject:
      `Co-Marketing Partnership Opportunity - {{ $('⚙️ Build AI Prompt').item.json.company_name }}`
    - Message:
      `{{ $json.choices[0].message.content }}`
    - Recipient:
      - The supplied workflow uses placeholder logic based on domain, ending with `@example.com`
      - For a practical rebuild, replace this with a real contact lookup source or a verified outreach address field

14. **Create a Google Sheets node to append history**
    - Node type: `Google Sheets`
    - Name it: `📑 Update Previous Scan Data`
    - Connect `📧 Send Co-Marketing Email` → `📑 Update Previous Scan Data`
    - Operation: `Append`
    - Select spreadsheet and `Previous Scan` tab
    - Map fields:
      - `domain` = `{{ $('⚙️ Build AI Prompt').item.json.domain }}`
      - `tool_name` = `{{ $('⚙️ Build AI Prompt').item.json.tool_name }}`
      - `detected_at` = `{{ $now.toISO() }}`
      - `email_sent` = `true`

15. **Close the batch loop**
    - Connect `📑 Update Previous Scan Data` → `🔀 Split by Tool`
    - This advances batch processing to the next tool after logging

16. **Optionally add sticky notes**
    - Add sticky notes for:
      - Trigger & input
      - Discovery & change detection
      - AI outreach
      - Output/logging
      - General workflow explanation

17. **Test with a small dataset**
    - Add 1–2 rows to `Complementary Tools`
    - Put one known domain into `Previous Scan` to confirm duplicate filtering
    - Execute manually before activation

18. **Activate the workflow**
    - Once the credentials and sheet mappings are validated, activate the workflow so the daily trigger runs automatically

---

## Credential Configuration Notes

### Google Sheets OAuth2
- Required for both read and append operations
- The connected Google account must have edit access to the spreadsheet

### Gmail OAuth2
- Required for sending emails
- The Gmail account must permit send operations from n8n

### PredictLeads
- The workflow uses raw API headers, not a dedicated credential type
- You must supply:
  - API key
  - API token

### OpenAI
- The workflow uses raw HTTP authentication header
- Provide a valid bearer token with access to the chosen model

---

## Input/Output Expectations

### Complementary Tools tab
Expected columns:
- `tool_name`
- `tech_id`

### Previous Scan tab
Expected columns:
- `domain`
- `tool_name`
- `detected_at`
- `email_sent`

### PredictLeads technology detections response
Expected shape includes:
- `data[]`
- each entry containing `attributes.domain`
- optionally `attributes.first_detected_at`

### PredictLeads company response
Expected shape includes:
- `data[0].attributes`
- fields such as:
  - `domain`
  - `company_name`
  - `friendly_company_name`
  - `industry`
  - `employee_count`
  - `description`

### OpenAI response
Expected shape includes:
- `choices[0].message.content`

---

## Important Corrections Recommended During Rebuild

1. **Fix the enrichment URL**
   - Current workflow likely enriches the wrong domain because it references `📑 Read Previous Scan Data`.
   - Prefer:
     `https://predictleads.com/api/v3/companies/{{ $json.domain }}`

2. **Replace fake recipient generation**
   - Current Gmail `sendTo` value is placeholder logic and not a reliable contact strategy.
   - Use a true company contact field, prospecting source, CRM, or enrichment provider.

3. **Normalize domains before comparison**
   - Consider lowercasing and trimming domains in the comparison code to avoid duplicates caused by formatting differences.

4. **Consider error handling**
   - Add dedicated IF or Error Trigger paths for:
     - PredictLeads failures
     - OpenAI failures
     - Gmail send failures
     - Google Sheets append failures

5. **Preserve actual detection timestamp**
   - If needed, write the original PredictLeads `detected_at` rather than `$now`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Tracks companies adopting tools that complement yours and sends AI-drafted co-marketing outreach emails to new adopters. | Workflow purpose |
| Setup requires Google Sheet with tool names and PredictLeads tech IDs, Gmail OAuth2, OpenAI API key, and PredictLeads API access. | Setup requirements |
| Example use case: if you have an analytics tool that pairs well with HubSpot, this detects new HubSpot adopters and sends a partnership pitch offering a joint webinar or integration guide. | Use case |
| PredictLeads API | https://predictleads.com |
| Questions / author contact | https://www.linkedin.com/in/yaronbeen |