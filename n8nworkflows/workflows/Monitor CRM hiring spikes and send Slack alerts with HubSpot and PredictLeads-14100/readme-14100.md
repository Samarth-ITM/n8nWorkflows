Monitor CRM hiring spikes and send Slack alerts with HubSpot and PredictLeads

https://n8nworkflows.xyz/workflows/monitor-crm-hiring-spikes-and-send-slack-alerts-with-hubspot-and-predictleads-14100


# Monitor CRM hiring spikes and send Slack alerts with HubSpot and PredictLeads

# 1. Workflow Overview

This workflow monitors HubSpot companies for hiring activity using PredictLeads, compares current job openings against previously stored counts in Google Sheets, and sends Slack alerts when hiring increases by more than 50%.

Its main business purpose is sales signal detection: if a target account suddenly starts hiring in strategic roles such as engineering, sales, product, marketing, or data, that can indicate growth, budget availability, or upcoming projects worth pursuing.

## 1.1 Scheduled Input Reception

The workflow starts every day at 9:00 AM and retrieves all companies from HubSpot CRM, including their domain, company name, and HubSpot object ID.

## 1.2 Company-by-Company Hiring Enrichment

Each company is processed individually in a loop. For each one, the workflow:
- calls PredictLeads for current job openings,
- filters only strategic job categories,
- reads the company’s historical hiring record from Google Sheets.

## 1.3 Hiring Spike Detection Logic

The workflow combines current hiring data with historical counts, calculates percentage change, and determines whether the company has a hiring spike. A spike is defined here as a rise greater than 50%.

## 1.4 Output Actions and Historical Persistence

If a spike is detected, the workflow:
- updates the HubSpot company record,
- sends a Slack alert,
- updates Google Sheets with the new hiring baseline.

If no spike is detected, it still updates Google Sheets so future comparisons stay accurate.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

### Overview
This block launches the workflow automatically on a daily schedule and pulls the list of companies to monitor from HubSpot. These records become the master input set for the rest of the workflow.

### Nodes Involved
- ⏰ Daily 9AM Trigger
- 🏢 HubSpot Get Companies

### Node Details

#### ⏰ Daily 9AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:**
  - Uses a cron expression: `0 9 * * *`
  - Runs daily at 9:00 AM server time.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No input; entry node.
  - Outputs to `🏢 HubSpot Get Companies`.
- **Version-specific requirements:**
  - Type version `1.2`.
  - Execution time depends on the n8n instance timezone/server timezone.
- **Edge cases or potential failure types:**
  - Timezone mismatch causing execution at an unexpected hour.
  - Workflow inactive, so trigger never fires.
- **Sub-workflow reference:** None.

#### 🏢 HubSpot Get Companies
- **Type and technical role:** `n8n-nodes-base.hubspot`; retrieves HubSpot company records.
- **Configuration choices:**
  - Resource: `company`
  - Operation: `getAll`
  - Authentication: OAuth2
  - Return all records enabled
  - Explicit properties fetched:
    - `domain`
    - `name`
    - `hs_object_id`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from `⏰ Daily 9AM Trigger`
  - Output to `🔄 Loop Over Companies`
- **Version-specific requirements:**
  - Type version `2`
  - Requires HubSpot OAuth2 credentials with permission to read companies.
- **Edge cases or potential failure types:**
  - OAuth token expired or missing scopes.
  - Large HubSpot datasets may increase runtime.
  - Some companies may not have a domain, which will later break the PredictLeads request.
  - Returned property structure uses HubSpot property wrappers like `.properties.domain.value`, which later nodes rely on.
- **Sub-workflow reference:** None.

---

## 2.2 Company-by-Company Hiring Enrichment

### Overview
This block iterates through each HubSpot company, fetches live job openings from PredictLeads, narrows them to target roles, and reads matching historical counts from Google Sheets for later comparison.

### Nodes Involved
- 🔄 Loop Over Companies
- 🔍 Fetch Job Openings
- ⚙️ Filter Target Roles
- 📂 Read Historical Counts
- Merge

### Node Details

#### 🔄 Loop Over Companies
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates over company records one at a time.
- **Configuration choices:**
  - No custom batch size shown; default behavior is used.
  - Acts as loop controller for processing each company.
- **Key expressions or variables used:** Referenced by downstream Code node using:
  - `$('🔄 Loop Over Companies').first().json.properties.domain`
  - `$('🔄 Loop Over Companies').first().json.properties.name`
  - `$('🔄 Loop Over Companies').first().json.properties.hs_object_id`
- **Input and output connections:**
  - Input from `🏢 HubSpot Get Companies`
  - Loop branch outputs to:
    - `🔍 Fetch Job Openings`
    - `📂 Read Historical Counts`
  - Completion/continue branch receives input back from:
    - `📝 Update Google Sheets`
    - `📝 Update Sheets (No Spike)`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases or potential failure types:**
  - If no companies are returned, downstream logic does not run.
  - The use of `.first()` in downstream code assumes the current loop item is the first accessible item from this node, which can be fragile depending on execution/data access semantics.
- **Sub-workflow reference:** None.

#### 🔍 Fetch Job Openings
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls PredictLeads API.
- **Configuration choices:**
  - URL built dynamically:
    - `https://predictleads.com/api/v3/companies/{{ $json.properties.domain.value }}/job_openings`
  - Sends headers:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ $json.properties.domain.value }}`
- **Input and output connections:**
  - Input from `🔄 Loop Over Companies`
  - Output to `⚙️ Filter Target Roles`
- **Version-specific requirements:**
  - Type version `4.2`
- **Edge cases or potential failure types:**
  - Missing or invalid company domain.
  - Invalid PredictLeads credentials.
  - API rate limits or network timeouts.
  - Non-200 responses if PredictLeads has no record for the company.
  - Companies with malformed domains could generate invalid URLs.
- **Sub-workflow reference:** None.

#### ⚙️ Filter Target Roles
- **Type and technical role:** `n8n-nodes-base.code`; transforms PredictLeads response into a normalized hiring summary.
- **Configuration choices:**
  - Defines target roles:
    - `sales`
    - `engineering`
    - `marketing`
    - `product`
    - `data`
  - Reads all jobs from `$input.first().json.data || []`
  - Uses job title matching with case-insensitive substring checks.
  - Outputs:
    - domain
    - companyName
    - companyId
    - totalJobCount
    - matchingJobCount
    - matchingJobs
    - targetRoles
- **Key expressions or variables used:**
  - `$('🔄 Loop Over Companies').first().json.properties.domain`
  - `$('🔄 Loop Over Companies').first().json.properties.name`
  - `$('🔄 Loop Over Companies').first().json.properties.hs_object_id`
  - `$input.first().json.data || []`
- **Input and output connections:**
  - Input from `🔍 Fetch Job Openings`
  - Output to `Merge`
- **Version-specific requirements:**
  - Type version `2`
  - Requires Code node JavaScript support.
- **Edge cases or potential failure types:**
  - If PredictLeads response structure changes and `json.data` is absent, matching count becomes 0.
  - Use of `.first()` from the loop node may incorrectly reference data in some execution patterns.
  - Simple substring matching can miss synonyms like “software developer”, “revops”, “growth”, or “ML engineer”.
  - False positives are possible if unrelated titles contain target keywords.
- **Sub-workflow reference:** None.

#### 📂 Read Historical Counts
- **Type and technical role:** `n8n-nodes-base.googleSheets`; retrieves historical company hiring data.
- **Configuration choices:**
  - Document: specified Google Sheet
  - Sheet/tab: `HistoricalCounts` (`gid=0`)
  - Filter on `domain` using current company domain:
    - `={{ $json.properties.domain.value }}`
- **Key expressions or variables used:**
  - `{{ $json.properties.domain.value }}`
- **Input and output connections:**
  - Input from `🔄 Loop Over Companies`
  - Output to `Merge`
- **Version-specific requirements:**
  - Type version `4.5`
  - Requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Credential or spreadsheet access errors.
  - Sheet/tab mismatch.
  - If the sheet returns rows directly rather than under a `rows` property, the later Code node assumption may not match actual output.
  - Domain format mismatches between HubSpot and Google Sheets may prevent historical matches.
- **Sub-workflow reference:** None.

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`; synchronizes current hiring data and historical lookup path.
- **Configuration choices:**
  - Default merge behavior; no explicit mode set in the exported parameters.
  - Receives:
    - Input 1 from `⚙️ Filter Target Roles`
    - Input 2 from `📂 Read Historical Counts`
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Input 0 from `⚙️ Filter Target Roles`
  - Input 1 from `📂 Read Historical Counts`
  - Output to `📊 Compare vs Historical`
- **Version-specific requirements:**
  - Type version `3.2`
- **Edge cases or potential failure types:**
  - Because the later code node independently reads from `📂 Read Historical Counts`, this merge may be used mainly as synchronization rather than data combination.
  - Default merge mode can behave differently than expected if item counts differ.
  - If one branch fails or returns no item, the merge may block downstream execution.
- **Sub-workflow reference:** None.

---

## 2.3 Hiring Spike Detection Logic

### Overview
This block computes the hiring change percentage relative to historical values and determines whether the company qualifies as having a meaningful hiring spike.

### Nodes Involved
- 📊 Compare vs Historical
- ❓ Spike Detected?

### Node Details

#### 📊 Compare vs Historical
- **Type and technical role:** `n8n-nodes-base.code`; calculates hiring trend metrics.
- **Configuration choices:**
  - Reads current summary from `$input.first().json`
  - Reads historical rows with:
    - `$('📂 Read Historical Counts').first().json.rows || []`
  - Finds historical row by matching `domain`
  - Computes:
    - `previousCount`
    - `currentCount`
    - `percentChange`
    - `spikeDetected`
    - `checkDate`
  - Rules:
    - If previous count > 0: standard percentage change formula
    - If previous count = 0 and current count > 0: treated as `100%`
    - Spike threshold: `percentChange > 50`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('📂 Read Historical Counts').first().json.rows || []`
  - `new Date().toISOString().split('T')[0]`
- **Input and output connections:**
  - Input from `Merge`
  - Output to `❓ Spike Detected?`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - The node assumes Google Sheets output is wrapped in `.json.rows`; this may not match actual node output shape.
  - If `current.domain` is a HubSpot property object rather than a string, equality with sheet `domain` values may fail.
  - New companies with no history automatically get 100%, which may create intentional but noisy alerts.
  - Negative changes are allowed and will simply not trigger the spike branch.
- **Sub-workflow reference:** None.

#### ❓ Spike Detected?
- **Type and technical role:** `n8n-nodes-base.if`; branches execution based on boolean spike detection.
- **Configuration choices:**
  - Boolean equals condition:
    - `={{ $json.spikeDetected }} === true`
  - True branch:
    - `🏢 HubSpot Update Company`
    - `💬 Slack Spike Alert`
  - False branch:
    - `📝 Update Sheets (No Spike)`
- **Key expressions or variables used:**
  - `={{ $json.spikeDetected }}`
- **Input and output connections:**
  - Input from `📊 Compare vs Historical`
  - True outputs to:
    - `🏢 HubSpot Update Company`
    - `💬 Slack Spike Alert`
  - False output to:
    - `📝 Update Sheets (No Spike)`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - If `spikeDetected` is missing or not boolean, strict validation may cause unexpected branch behavior.
  - True branch fans out in parallel; Slack can fail independently of HubSpot update.
- **Sub-workflow reference:** None.

---

## 2.4 Output Actions and Historical Persistence

### Overview
This block performs downstream business actions. Spike cases update HubSpot and notify Slack, while all outcomes write the latest hiring snapshot back to Google Sheets for future comparisons.

### Nodes Involved
- 🏢 HubSpot Update Company
- 💬 Slack Spike Alert
- 📝 Update Google Sheets
- 📝 Update Sheets (No Spike)

### Node Details

#### 🏢 HubSpot Update Company
- **Type and technical role:** `n8n-nodes-base.hubspot`; updates the HubSpot company with a custom signal property.
- **Configuration choices:**
  - Resource: `company`
  - Operation: `update`
  - Company ID expression:
    - `={{ $json.companyId.value }}`
  - Sets custom property:
    - `hiring_signal = true`
- **Key expressions or variables used:**
  - `{{ $json.companyId.value }}`
- **Input and output connections:**
  - Input from `❓ Spike Detected?` true branch
  - Output to `📝 Update Google Sheets`
- **Version-specific requirements:**
  - Type version `2`
  - Requires HubSpot OAuth2 credentials with update permissions.
  - The custom HubSpot property `hiring_signal` must already exist in HubSpot.
- **Edge cases or potential failure types:**
  - Property `hiring_signal` missing in HubSpot schema.
  - `companyId.value` absent if upstream object shape changes.
  - Permission errors on company updates.
- **Sub-workflow reference:** None.

#### 💬 Slack Spike Alert
- **Type and technical role:** `n8n-nodes-base.slack`; posts a message to a Slack channel.
- **Configuration choices:**
  - Authentication: OAuth2
  - Sends text message to a configured channel ID
  - Message includes:
    - company name
    - domain
    - current role count
    - previous count
    - percentage change
    - list of matching jobs
    - date
- **Key expressions or variables used:**
  - `{{ $json.companyName.value }}`
  - `{{ $json.domain.value }}`
  - `{{ $json.currentCount }}`
  - `{{ $json.previousCount }}`
  - `{{ $json.percentChange }}`
  - `{{ $json.matchingJobs.join(', ') }}`
  - `{{ $json.checkDate }}`
- **Input and output connections:**
  - Input from `❓ Spike Detected?` true branch
  - No downstream connection
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Slack OAuth2 credentials and channel access.
- **Edge cases or potential failure types:**
  - Slack bot not invited to the target channel.
  - Invalid channel ID.
  - If `matchingJobs` is empty, formatting may look odd.
  - If upstream values are objects without `.value`, expressions may fail or print `[object Object]`.
- **Sub-workflow reference:** None.

#### 📝 Update Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`; upserts the historical record after a spike.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `domain`
  - Writes:
    - `domain`
    - `company_name`
    - `job_count`
    - `previous_count`
    - `percent_change`
    - `check_date`
  - Uses user-entered cell formatting
- **Key expressions or variables used:**
  - `{{ $('📊 Compare vs Historical').item.json.domain.value }}`
  - `{{ $('📊 Compare vs Historical').item.json.currentCount }}`
  - `{{ $('📊 Compare vs Historical').item.json.checkDate }}`
  - `{{ $('📊 Compare vs Historical').item.json.companyName.value }}`
  - `{{ $('📊 Compare vs Historical').item.json.percentChange }}`
  - `{{ $('📊 Compare vs Historical').item.json.previousCount }}`
- **Input and output connections:**
  - Input from `🏢 HubSpot Update Company`
  - Output back to `🔄 Loop Over Companies` to continue loop execution
- **Version-specific requirements:**
  - Type version `4.5`
  - Requires write access to the configured sheet.
- **Edge cases or potential failure types:**
  - If `domain` or `companyName` is not stored as an object with `.value`, expressions will fail.
  - Matching column inconsistency may create duplicate rows instead of updating.
  - If HubSpot update fails, this node will not run even though Slack may already have been sent.
- **Sub-workflow reference:** None.

#### 📝 Update Sheets (No Spike)
- **Type and technical role:** `n8n-nodes-base.googleSheets`; upserts the historical record when no spike is found.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `domain`
  - Same destination sheet as spike path
  - Writes the same fields as the spike branch
- **Key expressions or variables used:**
  - `{{ $('📊 Compare vs Historical').item.json.domain.value }}`
  - `{{ $('📊 Compare vs Historical').item.json.currentCount }}`
  - `{{ $json.checkDate }}`
  - `{{ $('📊 Compare vs Historical').item.json.companyName.value }}`
  - `{{ $json.percentChange }}`
  - `{{ $json.previousCount }}`
- **Input and output connections:**
  - Input from `❓ Spike Detected?` false branch
  - Output back to `🔄 Loop Over Companies`
- **Version-specific requirements:**
  - Type version `4.5`
- **Edge cases or potential failure types:**
  - Same mapping fragility as the spike-path Sheets node.
  - Duplicates can occur if domains differ in case or format.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| About This Workflow | Sticky Note | Documentation/comment node |  |  | ABOUT THIS WORKFLOW<br>Pulls HubSpot companies, checks PredictLeads for new job openings, and sends Slack alerts when hiring spikes above 50%.<br>Setup: HubSpot API, Google Sheets (for historical tracking), Slack bot, PredictLeads API credentials.<br>Use case: Your sales team wants to know when target accounts ramp up hiring. A spike in engineering roles often means new budget and projects you can sell into.<br>PredictLeads API: https://predictleads.com<br>Questions: https://www.linkedin.com/in/yaronbeen |
| 📋 Sticky: Trigger & Input | Sticky Note | Documentation/comment node |  |  | ## ⏰ Trigger & Company Source<br>**Nodes:** ⏰ Daily 9AM Trigger → 🏢 HubSpot Get Companies<br>**Description:** The workflow begins with a scheduled trigger that runs every day at 9 AM. It retrieves all companies from HubSpot CRM along with key identifiers such as the company domain, company name, and HubSpot company ID. These companies represent the target accounts that will be monitored for hiring activity. The retrieved company data becomes the input dataset used for job monitoring and hiring signal detection. |
| 📋 Sticky: Enrichment | Sticky Note | Documentation/comment node |  |  | ## 🔍 Hiring Data Collection<br>**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Job Openings → ⚙️ Filter Target Roles → 📂 Read Historical Counts<br>**Description:** Each company is processed individually using a loop to evaluate its hiring activity. The workflow queries the PredictLeads API to retrieve the company’s current job openings. It then filters the results to focus on specific strategic roles such as sales, engineering, marketing, product, and data. At the same time, the workflow retrieves previously stored hiring counts from Google Sheets. These historical values will be used to compare current hiring activity with past records. |
| 📋 Sticky: Processing | Sticky Note | Documentation/comment node |  |  | ## 📊 Hiring Spike Detection<br>**Nodes:** Merge → 📊 Compare vs Historical → ❓ Spike Detected?<br>**Description:** Current hiring data is merged with the historical job counts retrieved from Google Sheets. The workflow calculates the percentage change between the current number of job openings and the previously recorded count. If the increase exceeds the defined threshold (greater than 50%), the workflow flags the company as experiencing a **hiring spike**, which may indicate new projects, budget expansion, or increased operational activity. |
| 📋 Sticky: Output | Sticky Note | Documentation/comment node |  |  | ## 📤 Actions & Alerts<br>**Nodes:** 🏢 HubSpot Update Company → 💬 Slack Spike Alert → 📝 Update Google Sheets → 📝 Update Sheets (No Spike)<br>**Description:** When a hiring spike is detected, the workflow updates the company record in HubSpot to mark the hiring signal and sends a Slack notification to alert the team. This allows sales or business development teams to quickly identify companies that may be expanding and potentially require new services or solutions. Regardless of whether a spike occurs or not, the workflow updates the historical hiring data in Google Sheets so future comparisons remain accurate. |
| ⏰ Daily 9AM Trigger | Schedule Trigger | Daily workflow start |  | 🏢 HubSpot Get Companies | ## ⏰ Trigger & Company Source<br>**Nodes:** ⏰ Daily 9AM Trigger → 🏢 HubSpot Get Companies<br>**Description:** The workflow begins with a scheduled trigger that runs every day at 9 AM. It retrieves all companies from HubSpot CRM along with key identifiers such as the company domain, company name, and HubSpot company ID. These companies represent the target accounts that will be monitored for hiring activity. The retrieved company data becomes the input dataset used for job monitoring and hiring signal detection. |
| 🏢 HubSpot Get Companies | HubSpot | Fetch all monitored companies from CRM | ⏰ Daily 9AM Trigger | 🔄 Loop Over Companies | ## ⏰ Trigger & Company Source<br>**Nodes:** ⏰ Daily 9AM Trigger → 🏢 HubSpot Get Companies<br>**Description:** The workflow begins with a scheduled trigger that runs every day at 9 AM. It retrieves all companies from HubSpot CRM along with key identifiers such as the company domain, company name, and HubSpot company ID. These companies represent the target accounts that will be monitored for hiring activity. The retrieved company data becomes the input dataset used for job monitoring and hiring signal detection. |
| 🔄 Loop Over Companies | Split In Batches | Iterate through companies one by one | 🏢 HubSpot Get Companies, 📝 Update Google Sheets, 📝 Update Sheets (No Spike) | 🔍 Fetch Job Openings, 📂 Read Historical Counts | ## 🔍 Hiring Data Collection<br>**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Job Openings → ⚙️ Filter Target Roles → 📂 Read Historical Counts<br>**Description:** Each company is processed individually using a loop to evaluate its hiring activity. The workflow queries the PredictLeads API to retrieve the company’s current job openings. It then filters the results to focus on specific strategic roles such as sales, engineering, marketing, product, and data. At the same time, the workflow retrieves previously stored hiring counts from Google Sheets. These historical values will be used to compare current hiring activity with past records. |
| 🔍 Fetch Job Openings | HTTP Request | Retrieve current job openings from PredictLeads | 🔄 Loop Over Companies | ⚙️ Filter Target Roles | ## 🔍 Hiring Data Collection<br>**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Job Openings → ⚙️ Filter Target Roles → 📂 Read Historical Counts<br>**Description:** Each company is processed individually using a loop to evaluate its hiring activity. The workflow queries the PredictLeads API to retrieve the company’s current job openings. It then filters the results to focus on specific strategic roles such as sales, engineering, marketing, product, and data. At the same time, the workflow retrieves previously stored hiring counts from Google Sheets. These historical values will be used to compare current hiring activity with past records. |
| ⚙️ Filter Target Roles | Code | Filter openings to monitored job categories | 🔍 Fetch Job Openings | Merge | ## 🔍 Hiring Data Collection<br>**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Job Openings → ⚙️ Filter Target Roles → 📂 Read Historical Counts<br>**Description:** Each company is processed individually using a loop to evaluate its hiring activity. The workflow queries the PredictLeads API to retrieve the company’s current job openings. It then filters the results to focus on specific strategic roles such as sales, engineering, marketing, product, and data. At the same time, the workflow retrieves previously stored hiring counts from Google Sheets. These historical values will be used to compare current hiring activity with past records. |
| 📂 Read Historical Counts | Google Sheets | Read previous hiring counts for the current domain | 🔄 Loop Over Companies | Merge | ## 🔍 Hiring Data Collection<br>**Nodes:** 🔄 Loop Over Companies → 🔍 Fetch Job Openings → ⚙️ Filter Target Roles → 📂 Read Historical Counts<br>**Description:** Each company is processed individually using a loop to evaluate its hiring activity. The workflow queries the PredictLeads API to retrieve the company’s current job openings. It then filters the results to focus on specific strategic roles such as sales, engineering, marketing, product, and data. At the same time, the workflow retrieves previously stored hiring counts from Google Sheets. These historical values will be used to compare current hiring activity with past records. |
| Merge | Merge | Synchronize enrichment and historical lookup branches | ⚙️ Filter Target Roles, 📂 Read Historical Counts | 📊 Compare vs Historical | ## 📊 Hiring Spike Detection<br>**Nodes:** Merge → 📊 Compare vs Historical → ❓ Spike Detected?<br>**Description:** Current hiring data is merged with the historical job counts retrieved from Google Sheets. The workflow calculates the percentage change between the current number of job openings and the previously recorded count. If the increase exceeds the defined threshold (greater than 50%), the workflow flags the company as experiencing a **hiring spike**, which may indicate new projects, budget expansion, or increased operational activity. |
| 📊 Compare vs Historical | Code | Compute current vs previous hiring change | Merge | ❓ Spike Detected? | ## 📊 Hiring Spike Detection<br>**Nodes:** Merge → 📊 Compare vs Historical → ❓ Spike Detected?<br>**Description:** Current hiring data is merged with the historical job counts retrieved from Google Sheets. The workflow calculates the percentage change between the current number of job openings and the previously recorded count. If the increase exceeds the defined threshold (greater than 50%), the workflow flags the company as experiencing a **hiring spike**, which may indicate new projects, budget expansion, or increased operational activity. |
| ❓ Spike Detected? | IF | Branch on whether hiring increase exceeds threshold | 📊 Compare vs Historical | 🏢 HubSpot Update Company, 💬 Slack Spike Alert, 📝 Update Sheets (No Spike) | ## 📊 Hiring Spike Detection<br>**Nodes:** Merge → 📊 Compare vs Historical → ❓ Spike Detected?<br>**Description:** Current hiring data is merged with the historical job counts retrieved from Google Sheets. The workflow calculates the percentage change between the current number of job openings and the previously recorded count. If the increase exceeds the defined threshold (greater than 50%), the workflow flags the company as experiencing a **hiring spike**, which may indicate new projects, budget expansion, or increased operational activity. |
| 🏢 HubSpot Update Company | HubSpot | Mark company with hiring signal in HubSpot | ❓ Spike Detected? | 📝 Update Google Sheets | ## 📤 Actions & Alerts<br>**Nodes:** 🏢 HubSpot Update Company → 💬 Slack Spike Alert → 📝 Update Google Sheets → 📝 Update Sheets (No Spike)<br>**Description:** When a hiring spike is detected, the workflow updates the company record in HubSpot to mark the hiring signal and sends a Slack notification to alert the team. This allows sales or business development teams to quickly identify companies that may be expanding and potentially require new services or solutions. Regardless of whether a spike occurs or not, the workflow updates the historical hiring data in Google Sheets so future comparisons remain accurate. |
| 💬 Slack Spike Alert | Slack | Notify team about hiring spike | ❓ Spike Detected? |  | ## 📤 Actions & Alerts<br>**Nodes:** 🏢 HubSpot Update Company → 💬 Slack Spike Alert → 📝 Update Google Sheets → 📝 Update Sheets (No Spike)<br>**Description:** When a hiring spike is detected, the workflow updates the company record in HubSpot to mark the hiring signal and sends a Slack notification to alert the team. This allows sales or business development teams to quickly identify companies that may be expanding and potentially require new services or solutions. Regardless of whether a spike occurs or not, the workflow updates the historical hiring data in Google Sheets so future comparisons remain accurate. |
| 📝 Update Google Sheets | Google Sheets | Upsert new hiring baseline after spike branch | 🏢 HubSpot Update Company | 🔄 Loop Over Companies | ## 📤 Actions & Alerts<br>**Nodes:** 🏢 HubSpot Update Company → 💬 Slack Spike Alert → 📝 Update Google Sheets → 📝 Update Sheets (No Spike)<br>**Description:** When a hiring spike is detected, the workflow updates the company record in HubSpot to mark the hiring signal and sends a Slack notification to alert the team. This allows sales or business development teams to quickly identify companies that may be expanding and potentially require new services or solutions. Regardless of whether a spike occurs or not, the workflow updates the historical hiring data in Google Sheets so future comparisons remain accurate. |
| 📝 Update Sheets (No Spike) | Google Sheets | Upsert new hiring baseline when no spike is found | ❓ Spike Detected? | 🔄 Loop Over Companies | ## 📤 Actions & Alerts<br>**Nodes:** 🏢 HubSpot Update Company → 💬 Slack Spike Alert → 📝 Update Google Sheets → 📝 Update Sheets (No Spike)<br>**Description:** When a hiring spike is detected, the workflow updates the company record in HubSpot to mark the hiring signal and sends a Slack notification to alert the team. This allows sales or business development teams to quickly identify companies that may be expanding and potentially require new services or solutions. Regardless of whether a spike occurs or not, the workflow updates the historical hiring data in Google Sheets so future comparisons remain accurate. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `CRM Hiring Enrichment & Slack Alerts with PredictLeads`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name: `⏰ Daily 9AM Trigger`
   - Set a cron schedule to `0 9 * * *`
   - Confirm the timezone used by your n8n instance.

3. **Add a HubSpot node to fetch companies**
   - Node type: `HubSpot`
   - Name: `🏢 HubSpot Get Companies`
   - Resource: `Company`
   - Operation: `Get Many` / `Get All`
   - Enable return all records.
   - Under properties to retrieve, include:
     - `domain`
     - `name`
     - `hs_object_id`
   - Authentication: `OAuth2`
   - Connect HubSpot OAuth2 credentials.
   - Connect `⏰ Daily 9AM Trigger` → `🏢 HubSpot Get Companies`.

4. **Add a Split In Batches node**
   - Node type: `Split In Batches`
   - Name: `🔄 Loop Over Companies`
   - Keep default batch behavior unless you want a specific batch size.
   - Connect `🏢 HubSpot Get Companies` → `🔄 Loop Over Companies`.

5. **Add an HTTP Request node for PredictLeads**
   - Node type: `HTTP Request`
   - Name: `🔍 Fetch Job Openings`
   - Method: `GET`
   - URL:
     - `https://predictleads.com/api/v3/companies/{{ $json.properties.domain.value }}/job_openings`
   - Enable headers.
   - Add headers:
     - `X-Api-Key`: your PredictLeads API key
     - `X-Api-Token`: your PredictLeads API token
     - `Content-Type`: `application/json`
   - Connect `🔄 Loop Over Companies` main loop output → `🔍 Fetch Job Openings`.

6. **Add a Google Sheets node to read historical counts**
   - Node type: `Google Sheets`
   - Name: `📂 Read Historical Counts`
   - Authentication: `OAuth2`
   - Select your spreadsheet.
   - Select the sheet/tab `HistoricalCounts` (or equivalent).
   - Configure a lookup/filter where:
     - Column: `domain`
     - Value: `{{ $json.properties.domain.value }}`
   - Connect Google Sheets OAuth2 credentials.
   - Connect `🔄 Loop Over Companies` main loop output → `📂 Read Historical Counts`.

7. **Prepare the Google Sheet structure**
   - In the spreadsheet, create columns:
     - `domain`
     - `company_name`
     - `job_count`
     - `previous_count`
     - `percent_change`
     - `check_date`
   - Ensure `domain` is the unique match key used for updates.

8. **Add a Code node to filter strategic roles**
   - Node type: `Code`
   - Name: `⚙️ Filter Target Roles`
   - Paste logic equivalent to:
     - read current company metadata from the loop node,
     - read PredictLeads job list,
     - filter titles containing `sales`, `engineering`, `marketing`, `product`, or `data`,
     - return domain, company name, company ID, total count, matching count, and matching titles.
   - Connect `🔍 Fetch Job Openings` → `⚙️ Filter Target Roles`.

9. **Use this filtering logic in the Code node**
   - Core behavior to implement:
     - `targetRoles = ['sales', 'engineering', 'marketing', 'product', 'data']`
     - `allJobs = $input.first().json.data || []`
     - `matchingJobs = allJobs.filter(job => target keyword appears in lowercased title)`
   - Output fields:
     - `domain`
     - `companyName`
     - `companyId`
     - `totalJobCount`
     - `matchingJobCount`
     - `matchingJobs`
     - `targetRoles`

10. **Add a Merge node**
    - Node type: `Merge`
    - Name: `Merge`
    - Keep default settings unless you explicitly want a specific merge mode.
    - Connect:
      - `⚙️ Filter Target Roles` → `Merge` input 1
      - `📂 Read Historical Counts` → `Merge` input 2

11. **Add a Code node to compare current vs historical**
    - Node type: `Code`
    - Name: `📊 Compare vs Historical`
    - This node should:
      - read current hiring data,
      - read historical rows from the Sheets branch,
      - find the row with the same domain,
      - compute `previousCount`,
      - compute `percentChange`,
      - set `spikeDetected = percentChange > 50`,
      - add `checkDate` as today’s date in `YYYY-MM-DD`.
    - Connect `Merge` → `📊 Compare vs Historical`.

12. **Implement the comparison logic**
    - Core rules:
      - If `previousCount > 0`, use:
        - `((currentCount - previousCount) / previousCount) * 100`
      - If no previous count exists and current count is greater than 0:
        - treat as `100`
      - Round percent change.
      - Set:
        - `spikeDetected`
        - `currentCount`
        - `previousCount`
        - `percentChange`
        - `checkDate`

13. **Add an IF node for branch control**
    - Node type: `IF`
    - Name: `❓ Spike Detected?`
    - Condition:
      - boolean field `spikeDetected`
      - equals `true`
    - Connect `📊 Compare vs Historical` → `❓ Spike Detected?`

14. **Add a HubSpot node for spike cases**
    - Node type: `HubSpot`
    - Name: `🏢 HubSpot Update Company`
    - Resource: `Company`
    - Operation: `Update`
    - Company ID:
      - `{{ $json.companyId.value }}`
    - Add custom property update:
      - `hiring_signal = true`
    - Important: create the custom HubSpot company property `hiring_signal` in HubSpot before running the workflow.
    - Connect `❓ Spike Detected?` true output → `🏢 HubSpot Update Company`.

15. **Add a Slack node for spike notifications**
    - Node type: `Slack`
    - Name: `💬 Slack Spike Alert`
    - Authentication: `OAuth2`
    - Operation: send message to channel
    - Choose target channel ID.
    - Message body should include:
      - company name
      - domain
      - current count
      - previous count
      - percent change
      - matching job titles
      - check date
    - Example format:
      - `:chart_with_upwards_trend: Hiring Spike Detected`
      - followed by the company details
    - Connect `❓ Spike Detected?` true output → `💬 Slack Spike Alert`.

16. **Add a Google Sheets node for spike-path persistence**
    - Node type: `Google Sheets`
    - Name: `📝 Update Google Sheets`
    - Operation: `Append or Update`
    - Matching column: `domain`
    - Map columns:
      - `domain`
      - `company_name`
      - `job_count`
      - `previous_count`
      - `percent_change`
      - `check_date`
    - Use values from `📊 Compare vs Historical`
    - Connect `🏢 HubSpot Update Company` → `📝 Update Google Sheets`.

17. **Add a Google Sheets node for no-spike persistence**
    - Node type: `Google Sheets`
    - Name: `📝 Update Sheets (No Spike)`
    - Operation: `Append or Update`
    - Same spreadsheet and matching column
    - Map the same six fields
    - Connect `❓ Spike Detected?` false output → `📝 Update Sheets (No Spike)`.

18. **Close the processing loop**
    - Connect `📝 Update Google Sheets` → `🔄 Loop Over Companies`
    - Connect `📝 Update Sheets (No Spike)` → `🔄 Loop Over Companies`
   - This allows the workflow to move on to the next company after each processed record.

19. **Configure credentials**
    - **HubSpot OAuth2**
      - Must allow reading and updating companies.
    - **Google Sheets OAuth2**
      - Must allow reading and writing the selected spreadsheet.
    - **Slack OAuth2**
      - Bot must have permission to post into the selected channel.
    - **PredictLeads**
      - Enter API key and token in the HTTP headers or, preferably, move them into reusable credentials/environment variables.

20. **Add optional sticky notes/documentation nodes**
    - Add sticky notes for:
      - trigger and input
      - enrichment
      - processing
      - output
      - overall workflow description
    - These are informational only and do not affect execution.

21. **Test with a limited dataset first**
    - Temporarily limit HubSpot companies to a few records.
    - Verify:
      - domains resolve correctly,
      - PredictLeads returns job data,
      - historical rows match by domain,
      - Slack alerts only trigger above the threshold,
      - Google Sheets updates instead of duplicating rows.

22. **Activate the workflow**
    - Once validated, activate it so the schedule trigger runs automatically each day.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PredictLeads API | https://predictleads.com |
| Questions / author contact | https://www.linkedin.com/in/yaronbeen |
| Workflow purpose | Pulls HubSpot companies, checks PredictLeads for new job openings, and sends Slack alerts when hiring spikes above 50%. |
| Required setup | HubSpot API, Google Sheets for historical tracking, Slack bot, PredictLeads API credentials. |
| Business context | Sales teams can use engineering or other hiring spikes as signals of budget growth, operational expansion, or upcoming projects. |

## Additional implementation notes
- There are **no sub-workflows** in this workflow.
- There is **one entry point**: `⏰ Daily 9AM Trigger`.
- The workflow is currently **inactive** in the provided export (`active: false`).
- Several expressions assume HubSpot fields remain wrapped as objects with a `.value` property. If you normalize the data earlier, you should update all downstream expressions accordingly.
- The biggest technical risk in this design is the assumed shape of the Google Sheets output and the repeated use of `.first()` to access loop data. If you rebuild this flow, consider normalizing both branches explicitly before comparison to make the workflow more robust.