Track leadership hires and send AI outreach using PredictLeads, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/track-leadership-hires-and-send-ai-outreach-using-predictleads--openai-and-gmail-14104


# Track leadership hires and send AI outreach using PredictLeads, OpenAI and Gmail

# 1. Workflow Overview

This workflow monitors a list of target companies, detects hiring signals from PredictLeads, and automatically generates and sends outreach emails when leadership hiring activity is found. It also sends Slack alerts for sales hiring signals and unusually high hiring volume.

Primary use cases:
- Prospecting companies showing growth or organizational change
- Triggering outbound outreach from hiring intent data
- Logging hiring signals to Google Sheets for later review
- Alerting sales/ops teams in Slack about relevant hiring patterns

## 1.1 Trigger & Target Account Intake
The workflow starts on a daily schedule at 8:00 AM, loads company records from Google Sheets, and iterates through them one at a time using batch processing.

## 1.2 PredictLeads Job Retrieval & Core Signal Extraction
For each company domain, the workflow retrieves active, non-closed job openings from PredictLeads. It then analyzes those openings in parallel to:
- detect leadership hiring,
- detect sales hiring,
- count total open jobs.

## 1.3 Leadership Decision Routing
If at least one leadership role is found, the workflow continues into enrichment and outreach. If not, the current batch item is skipped and the workflow loops back to process the next company.

## 1.4 Parallel Alerting for Sales Hiring and Hiring Volume
Independently from the leadership path, the workflow checks whether the company is hiring for sales roles and whether total open roles meet a high-volume threshold. Matching cases trigger Slack alerts.

## 1.5 Company Enrichment, Signal Logging, AI Prompting, and Email Delivery
When leadership hiring is detected, the workflow enriches the company with PredictLeads company data, appends a record to Google Sheets, builds an OpenAI prompt, generates a personalized outreach email through the OpenAI Chat Completions API, sends it via Gmail, and then resumes batch processing.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Input Reception

### Overview
This block launches the workflow every day and retrieves the list of target companies from Google Sheets. It then feeds the list into a batch loop so each domain can be processed individually.

### Nodes Involved
- Daily Schedule
- Load Target Accounts
- Split Into Batches

### Node Details

#### Daily Schedule
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a cron schedule.
- **Configuration choices:**  
  Configured with cron expression `0 8 * * *`, meaning daily at 8:00 AM server time.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input. Outputs to **Load Target Accounts**.
- **Version-specific requirements:**  
  Uses `typeVersion 1.2`; schedule behavior depends on the n8n instance timezone/settings.
- **Edge cases or potential failure types:**  
  - Timezone mismatch between expected local time and server time
  - Workflow inactive, so trigger never runs
  - Cron misinterpretation if instance timezone changes
- **Sub-workflow reference:**  
  None.

#### Load Target Accounts
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Reads the spreadsheet containing target company records.
- **Configuration choices:**  
  Uses a specific Google Sheet document and sheet tab (`gid=0`, cached as `Sheet1`). The sticky note indicates expected columns are `domain` and `company_name`.
- **Key expressions or variables used:**  
  No dynamic expressions in the configured fields shown, but downstream nodes expect:
  - `$json.domain`
  - `$json.company_name`
- **Input and output connections:**  
  Input from **Daily Schedule**. Output to **Split Into Batches**.
- **Version-specific requirements:**  
  Uses Google Sheets node `typeVersion 4.5`.
- **Edge cases or potential failure types:**  
  - Google OAuth credential missing or expired
  - Spreadsheet/tab access denied
  - Missing `domain` column causing downstream PredictLeads failures
  - Empty sheet causing no processing
- **Sub-workflow reference:**  
  None.

#### Split Into Batches
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the loaded company rows one item at a time and also acts as the loop control point.
- **Configuration choices:**  
  Uses default options; no explicit batch size is defined in the JSON, so n8n defaults apply.
- **Key expressions or variables used:**  
  Referenced heavily by downstream Code nodes using:
  - `$('Split Into Batches').first().json.domain`
  - `$('Split Into Batches').first().json.company_name`
- **Input and output connections:**  
  Input from **Load Target Accounts**, and loop-back input from **Send Outreach Email** and false branch of **Leadership Role Found?**.  
  Output index 1 goes to **Retrieve Company Job Openings**. Output index 0 is the completion output and is unused.
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases or potential failure types:**  
  - Confusion around `.first()` in downstream nodes if multiple items are ever processed differently than expected
  - If looping is broken, workflow may stop after first item or skip remaining items
- **Sub-workflow reference:**  
  None.

---

## 2.2 Hiring Data Retrieval & Leadership Detection

### Overview
This block retrieves open job data for each domain from PredictLeads and determines whether leadership roles are present. It is the main decision point for whether enrichment and outreach should happen.

### Nodes Involved
- Retrieve Company Job Openings
- Filter Leadership Roles
- Leadership Role Found?

### Node Details

#### Retrieve Company Job Openings
- **Type and role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Fetches job opening data for the current company domain.
- **Configuration choices:**  
  - Resource: `jobOpenings`
  - Operation: `retrieveCompanyJobOpenings`
  - Domain: `={{ $json.domain }}`
  - `notClosed: true`
  - `activeOnly: true`
- **Key expressions or variables used:**  
  - `$json.domain`
- **Input and output connections:**  
  Input from **Split Into Batches**. Outputs in parallel to:
  - **Filter Leadership Roles**
  - **Calculate Job Count**
  - **Detect Sales Hiring Signals**
- **Version-specific requirements:**  
  PredictLeads community node `typeVersion 1`; requires installed package and valid PredictLeads credential.
- **Edge cases or potential failure types:**  
  - Missing or invalid PredictLeads API credential
  - Invalid domain format
  - PredictLeads rate limiting
  - API returns empty `data`
  - Unexpected schema changes in returned job payload
- **Sub-workflow reference:**  
  None.

#### Filter Leadership Roles
- **Type and role:** `n8n-nodes-base.code`  
  Inspects the returned job openings and filters for leadership titles.
- **Configuration choices:**  
  JavaScript code uses regex:
  ```js
  /\b(CRO|CMO|CTO|VP|Vice President|Head of|Chief)\b/i
  ```
  It maps matching jobs into a simplified structure with `title`, `location`, and `url`, and emits:
  - `domain`
  - `company_name`
  - `leadership_roles`
  - `leadership_found`
- **Key expressions or variables used:**  
  - `item.json.data || []`
  - `$('Split Into Batches').first().json.domain`
  - `$('Split Into Batches').first().json.company_name`
- **Input and output connections:**  
  Input from **Retrieve Company Job Openings**. Output to **Leadership Role Found?**
- **Version-specific requirements:**  
  Code node `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - If PredictLeads returns no `data`, code falls back to `[]`
  - Regex may miss relevant titles like “Director”, “GM”, “SVP”, “President”
  - Regex may include false positives for titles containing generic “Head of”
  - Using `$('Split Into Batches').first()` can be fragile if execution context changes
- **Sub-workflow reference:**  
  None.

#### Leadership Role Found?
- **Type and role:** `n8n-nodes-base.if`  
  Routes execution based on whether leadership hiring was found.
- **Configuration choices:**  
  Checks `={{ $json.leadership_found }}` is boolean true.
- **Key expressions or variables used:**  
  - `$json.leadership_found`
- **Input and output connections:**  
  Input from **Filter Leadership Roles**.  
  True output goes to **Retrieve Company**.  
  False output loops to **Split Into Batches** to process the next item.
- **Version-specific requirements:**  
  IF node `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - If `leadership_found` is missing or a string instead of boolean, strict validation may fail routing as expected
- **Sub-workflow reference:**  
  None.

---

## 2.3 Sales Hiring Signal Detection

### Overview
This parallel branch scans the same job opening data for sales-related roles. If found, it sends a Slack alert indicating possible CRM or sales-tool buying intent.

### Nodes Involved
- Detect Sales Hiring Signals
- Sales Hiring Detected?
- Send Sales Hiring Alert

### Node Details

#### Detect Sales Hiring Signals
- **Type and role:** `n8n-nodes-base.code`  
  Filters jobs for sales-oriented roles.
- **Configuration choices:**  
  Uses regex:
  ```js
  /\b(SDR|Sales|Account Executive|Business Development)\b/i
  ```
  Emits:
  - `domain`
  - `company_name`
  - `sales_roles`
  - `sales_hiring`
- **Key expressions or variables used:**  
  - `item.json.data || []`
  - `$('Split Into Batches').first().json.domain`
  - `$('Split Into Batches').first().json.company_name`
- **Input and output connections:**  
  Input from **Retrieve Company Job Openings**. Output to **Sales Hiring Detected?**
- **Version-specific requirements:**  
  Code node `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - Broad `Sales` term may cause false positives
  - Some relevant roles like “BDR” are not explicitly in regex, though “Business Development” partly covers them
  - Missing `attributes.title` can produce empty strings
- **Sub-workflow reference:**  
  None.

#### Sales Hiring Detected?
- **Type and role:** `n8n-nodes-base.if`  
  Checks whether one or more sales roles were found.
- **Configuration choices:**  
  Boolean equals true on `={{ $json.sales_hiring }}`.
- **Key expressions or variables used:**  
  - `$json.sales_hiring`
- **Input and output connections:**  
  Input from **Detect Sales Hiring Signals**.  
  True output goes to **Send Sales Hiring Alert**. False output unused.
- **Version-specific requirements:**  
  IF node `typeVersion 2.3`.
- **Edge cases or potential failure types:**  
  - If the field is absent, false branch is taken silently
- **Sub-workflow reference:**  
  None.

#### Send Sales Hiring Alert
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a Slack message when sales hiring is detected.
- **Configuration choices:**  
  Sends to a selected Slack channel using OAuth2. Message includes:
  - company name
  - domain
  - joined list of role titles
- **Key expressions or variables used:**  
  - `{{$json.company_name}}`
  - `{{$json.domain}}`
  - `{{$json.sales_roles.join(', ')}}`
- **Input and output connections:**  
  Input from **Sales Hiring Detected?** true branch. No downstream connection.
- **Version-specific requirements:**  
  Slack node `typeVersion 2.4`.
- **Edge cases or potential failure types:**  
  - Expired Slack OAuth token
  - Invalid channel access
  - `sales_roles` empty array resulting in a weak alert message
- **Sub-workflow reference:**  
  None.

---

## 2.4 Hiring Volume Signal Detection

### Overview
This parallel branch counts open jobs and checks whether the company is hiring at or above a threshold of 10 active roles. If so, it sends a Slack alert indicating a hiring spike.

### Nodes Involved
- Calculate Job Count
- High Hiring Volume?
- Send Hiring Spike Alert

### Node Details

#### Calculate Job Count
- **Type and role:** `n8n-nodes-base.set`  
  Derives a `job_count` field from the length of the PredictLeads `data` array.
- **Configuration choices:**  
  Assigns:
  - `job_count = {{ $json.data.length }}`
  Notably, the field is stored as **string**.
- **Key expressions or variables used:**  
  - `$json.data.length`
- **Input and output connections:**  
  Input from **Retrieve Company Job Openings**. Output to **High Hiring Volume?**
- **Version-specific requirements:**  
  Set node `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  - If `data` is undefined, expression may fail
  - String typing could create comparison ambiguity, though the IF node enables loose validation
- **Sub-workflow reference:**  
  None.

#### High Hiring Volume?
- **Type and role:** `n8n-nodes-base.if`  
  Checks whether `job_count >= 10`.
- **Configuration choices:**  
  Uses number comparison with loose typing enabled.
- **Key expressions or variables used:**  
  - `={{ $json.job_count }}`
- **Input and output connections:**  
  Input from **Calculate Job Count**.  
  True output goes to **Send Hiring Spike Alert**. False output unused.
- **Version-specific requirements:**  
  IF node `typeVersion 2.3`, with `looseTypeValidation: true`.
- **Edge cases or potential failure types:**  
  - If `job_count` is nonnumeric text, loose conversion may still fail or compare unexpectedly
  - Threshold fixed at 10 unless manually changed
- **Sub-workflow reference:**  
  None.

#### Send Hiring Spike Alert
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a Slack alert for high hiring volume.
- **Configuration choices:**  
  Sends a formatted multi-line message to a specified Slack channel.
- **Key expressions or variables used:**  
  - `{{ $('Retrieve Company Job Openings').item.json.included[0].attributes.company_name }}`
  - `{{ $('Retrieve Company Job Openings').item.json.included[0].attributes.domain }}`
  - `{{$json.job_count}}`
- **Input and output