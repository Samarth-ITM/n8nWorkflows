Score multi-signal buying intent and prioritize leads with PredictLeads, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/score-multi-signal-buying-intent-and-prioritize-leads-with-predictleads--google-sheets--and-slack-14348


# Score multi-signal buying intent and prioritize leads with PredictLeads, Google Sheets, and Slack

# 1. Workflow Overview

This workflow performs daily lead prioritization by combining multiple external buying-intent signals for a list of prospect companies. It reads prospect domains from Google Sheets, enriches each company with PredictLeads data, computes a weighted intent score, filters high-intent companies, ranks them, then sends Slack alerts and stores qualified leads in a second Google Sheets tab.

Typical use cases include:
- Sales development teams prioritizing outbound outreach
- RevOps teams building lightweight lead scoring automation
- Growth teams monitoring account activity without a full CRM enrichment stack

## 1.1 Trigger & Prospect Intake

The workflow starts on a daily schedule at 8 AM and retrieves a list of prospect domains from a Google Sheets tab. This sheet acts as the source of truth for which companies should be evaluated.

## 1.2 Signal Enrichment with PredictLeads

For each prospect domain, the workflow sequentially queries PredictLeads for:
- job openings
- technology detections / tech stack
- news events

These three signals are used as proxies for buying intent.

## 1.3 Scoring, Filtering, and Ranking

A Code node normalizes the three signal counts and combines them into a single intent score using a weighted formula:

- hiring × 5
- tech × 3
- news × 2

Only leads with a score greater than 20 are kept. The remaining leads are sorted in descending order of intent score.

## 1.4 Alerting and Persistence

Qualified leads are sent to Slack as formatted notifications and appended to a dedicated Google Sheets output tab for tracking and later follow-up.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger & Input Reception

### Overview

This block launches the workflow every day and loads the list of prospect companies from Google Sheets. It establishes the item stream used by all subsequent enrichment steps.

### Nodes Involved

- Daily Prospect Scan Trigger
- Get Prospect List

### Node Details

#### 1. Daily Prospect Scan Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a recurring schedule.
- **Configuration choices:**  
  Configured to run daily at hour 8. No additional interval complexity is defined.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none (entry point)
  - Output: `Get Prospect List`
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`. Standard Schedule Trigger behavior in recent n8n versions.
- **Edge cases or potential failure types:**  
  - Workflow will not run unless activated
  - Server timezone may affect the actual trigger time
  - If n8n instance is paused/down at trigger time, execution behavior depends on deployment/runtime configuration
- **Sub-workflow reference:**  
  None.

#### 2. Get Prospect List

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads the input list of prospect domains from Google Sheets.
- **Configuration choices:**  
  - Uses a specific spreadsheet document
  - Reads from the tab named **Prospects**
  - No explicit filters or column selection are defined, so the node is expected to return rows from the sheet
- **Key expressions or variables used:**  
  Downstream nodes rely on:
  - `{{$json.domain}}`
- **Input and output connections:**  
  - Input: `Daily Prospect Scan Trigger`
  - Output: `Fetch Job Openings`
- **Version-specific requirements:**  
  Uses `typeVersion 4.7`, which corresponds to newer Google Sheets node behavior and UI schema handling.
- **Edge cases or potential failure types:**  
  - Missing or invalid Google Sheets credentials
  - Spreadsheet/tab not found
  - Sheet lacks a `domain` column
  - Empty rows may produce blank domains
  - Rate limits or permission issues from Google
- **Sub-workflow reference:**  
  None.

---

## Block 2 — PredictLeads Signal Enrichment

### Overview

This block enriches each prospect domain with three external intent signals from PredictLeads: open jobs, technology detections, and news events. The nodes are chained sequentially, so each item carries forward through all three API lookups.

### Nodes Involved

- Fetch Job Openings
- Fetch Tech Stack
- Fetch News Events

### Node Details

#### 3. Fetch Job Openings

- **Type and technical role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Calls PredictLeads to retrieve company job opening data.
- **Configuration choices:**  
  - Resource: `jobOpenings`
  - Operation: `retrieveCompanyJobOpenings`
  - Domain is dynamically taken from the Google Sheets item
- **Key expressions or variables used:**  
  - `={{ $('Get Prospect List').item.json.domain }}`
- **Input and output connections:**  
  - Input: `Get Prospect List`
  - Output: `Fetch Tech Stack`
- **Version-specific requirements:**  
  Uses `typeVersion 1`. Requires the PredictLeads community node/package to be installed in the n8n instance.
- **Edge cases or potential failure types:**  
  - Missing PredictLeads credentials
  - Invalid domain values from the source sheet
  - API quota/rate limiting
  - Company not found or no job data available
  - Transient API/network failures
- **Sub-workflow reference:**  
  None.

#### 4. Fetch Tech Stack

- **Type and technical role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Calls PredictLeads to retrieve detected technologies used by the company.
- **Configuration choices:**  
  - Resource: `technologyDetections`
  - Operation: `retrieveTechnologiesUsedByCompany`
  - Domain comes from the original Google Sheets row, not from the previous PredictLeads response
- **Key expressions or variables used:**  
  - `={{ $('Get Prospect List').item.json.domain }}`
- **Input and output connections:**  
  - Input: `Fetch Job Openings`
  - Output: `Fetch News Events`
- **Version-specific requirements:**  
  Uses `typeVersion 1`. Requires PredictLeads node support in the n8n environment.
- **Edge cases or potential failure types:**  
  - Same credential/API issues as above
  - No technologies detected for a domain
  - Expression failure if `Get Prospect List` did not produce a matching item
- **Sub-workflow reference:**  
  None.

#### 5. Fetch News Events

- **Type and technical role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Retrieves company news events from PredictLeads.
- **Configuration choices:**  
  - Resource: `newsEvents`
  - Operation: `retrieveCompanyNewsEvents`
  - Domain again references the source prospect row
- **Key expressions or variables used:**  
  - `={{ $('Get Prospect List').item.json.domain }}`
- **Input and output connections:**  
  - Input: `Fetch Tech Stack`
  - Output: `Normalize Data and Score`
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - API authentication or quota failures
  - Empty news results
  - Domain expression mismatch if item ordering breaks
- **Sub-workflow reference:**  
  None.

---

## Block 3 — Scoring, Filtering, and Ranking

### Overview

This block converts raw enrichment responses into numeric signals, calculates a weighted intent score, filters leads above a fixed threshold, and sorts the survivors from highest to lowest score.

### Nodes Involved

- Normalize Data and Score
- Filter High Intent Leads
- Rank Leads by Intent Score

### Node Details

#### 6. Normalize Data and Score

- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates data across previous nodes and computes normalized buying-intent fields.
- **Configuration choices:**  
  The node uses JavaScript to map over incoming items and derive:
  - `hiring_signal`
  - `tech_signal`
  - `news_signal`
  - `intent_score`
  - `domain`
- **Key expressions or variables used:**  
  The script references cross-node item arrays:
  - `$items("Fetch Job Openings")[index]?.json?.data?.length || 0`
  - `$items("Fetch Tech Stack")[index]?.json?.data?.length || 0`
  - `item.json.data?.length || 0`
  - `$items("Get Prospect List")[index]?.json?.domain || "unknown"`
  
  Scoring formula:
  - `score = (jobs * 5) + (tech * 3) + (news * 2)`
- **Input and output connections:**  
  - Input: `Fetch News Events`
  - Output: `Filter High Intent Leads`
- **Version-specific requirements:**  
  Uses `typeVersion 2`, which supports JavaScript execution in the Code node.
- **Edge cases or potential failure types:**  
  - Assumes item index alignment across `Get Prospect List`, `Fetch Job Openings`, `Fetch Tech Stack`, and `Fetch News Events`
  - If one upstream node drops or reorders items unexpectedly, signal counts may be mismatched across companies
  - If PredictLeads returns a structure without `json.data`, counts become `0`
  - If the source row lacks `domain`, the fallback becomes `"unknown"`
- **Sub-workflow reference:**  
  None.

#### 7. Filter High Intent Leads

- **Type and technical role:** `n8n-nodes-base.if`  
  Keeps only leads whose computed intent score exceeds the configured threshold.
- **Configuration choices:**  
  - Condition type: number
  - Comparison: greater than
  - Left value: `{{$json.intent_score}}`
  - Right value: `20`
  - Logical combinator: `and` (though only one condition is present)
- **Key expressions or variables used:**  
  - `={{ $json.intent_score }}`
- **Input and output connections:**  
  - Input: `Normalize Data and Score`
  - True output: `Rank Leads by Intent Score`
  - False output: not connected
- **Version-specific requirements:**  
  Uses `typeVersion 2.3`.
- **Edge cases or potential failure types:**  
  - Items with missing or non-numeric `intent_score` may fail strict validation or evaluate unexpectedly
  - False branch is ignored, so low-intent leads are simply discarded
- **Sub-workflow reference:**  
  None.

#### 8. Rank Leads by Intent Score

- **Type and technical role:** `n8n-nodes-base.sort`  
  Orders the qualifying leads by descending intent score.
- **Configuration choices:**  
  - Sort field: `intent_score`
  - Order: descending
- **Key expressions or variables used:**  
  None beyond the configured field name.
- **Input and output connections:**  
  - Input: `Filter High Intent Leads`
  - Outputs:
    - `Send Slack Alert`
    - `Save Qualified Leads to Sheet`
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - Sorting may behave lexicographically if data types are inconsistent and score is stored as string upstream in altered versions
  - If no items pass the IF node, nothing reaches this node
- **Sub-workflow reference:**  
  None.

---

## Block 4 — Alerts & Storage

### Overview

This final block sends high-intent lead notifications to Slack and appends the same qualified lead data to a Google Sheets output tab. Both actions are executed in parallel from the sorted result set.

### Nodes Involved

- Send Slack Alert
- Save Qualified Leads to Sheet

### Node Details

#### 9. Send Slack Alert

- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a formatted alert message to a Slack channel.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Destination mode: channel
  - Fixed selected channel ID
  - Message body includes domain, score, and signal breakdown
- **Key expressions or variables used:**  
  Message interpolates:
  - `{{$json.domain}}`
  - `{{$json.intent_score}}`
  - `{{$json.hiring_signal}}`
  - `{{$json.tech_signal}}`
  - `{{$json.news_signal}}`
- **Input and output connections:**  
  - Input: `Rank Leads by Intent Score`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion 2.4`.
- **Edge cases or potential failure types:**  
  - Missing/expired Slack OAuth2 credentials
  - Bot not invited to target channel
  - Invalid or inaccessible channel ID
  - Slack rate limits if many leads qualify at once
- **Sub-workflow reference:**  
  None.

#### 10. Save Qualified Leads to Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends each qualified lead as a new row in a Google Sheets output tab.
- **Configuration choices:**  
  - Operation: `append`
  - Uses manual field mapping
  - Writes:
    - `domain`
    - `hiring_signal`
    - `tech_signal`
    - `news_signal`
    - `intent_score`
  - Output tab is a dedicated sheet named **Save Qualified Leads to Sheet**
- **Key expressions or variables used:**  
  - `={{ $json.domain }}`
  - `={{ $json.hiring_signal }}`
  - `={{ $json.tech_signal }}`
  - `={{ $json.news_signal }}`
  - `={{ $json.intent_score }}`
- **Input and output connections:**  
  - Input: `Rank Leads by Intent Score`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion 4.7`.
- **Edge cases or potential failure types:**  
  - Missing Google Sheets credentials
  - Target tab missing or schema changed
  - Column names in the target sheet do not match mapping
  - Duplicate leads are appended without deduplication
  - Numeric fields may be stored as text depending on sheet formatting
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Prospect Scan Trigger | Schedule Trigger | Starts the workflow daily at 8 AM |  | Get Prospect List | ## Trigger & Input<br>Runs daily at 8 AM and loads prospect domains from Google Sheets. |
| Get Prospect List | Google Sheets | Loads prospect domains from the input spreadsheet | Daily Prospect Scan Trigger | Fetch Job Openings | ## Trigger & Input<br>Runs daily at 8 AM and loads prospect domains from Google Sheets. |
| Fetch Job Openings | PredictLeads | Retrieves company hiring/job opening data | Get Prospect List | Fetch Tech Stack | ## Signal Enrichment<br>Queries PredictLeads for hiring activity, tech stack, and news events for each prospect. |
| Fetch Tech Stack | PredictLeads | Retrieves detected technologies for each company | Fetch Job Openings | Fetch News Events | ## Signal Enrichment<br>Queries PredictLeads for hiring activity, tech stack, and news events for each prospect. |
| Fetch News Events | PredictLeads | Retrieves recent company news events | Fetch Tech Stack | Normalize Data and Score | ## Signal Enrichment<br>Queries PredictLeads for hiring activity, tech stack, and news events for each prospect. |
| Normalize Data and Score | Code | Converts raw enrichment data into signals and computes intent score | Fetch News Events | Filter High Intent Leads | ## Scoring & Filtering<br>Combines signals into a weighted intent score, filters leads above threshold, and ranks by score. |
| Filter High Intent Leads | IF | Keeps only leads with intent score > 20 | Normalize Data and Score | Rank Leads by Intent Score | ## Scoring & Filtering<br>Combines signals into a weighted intent score, filters leads above threshold, and ranks by score. |
| Rank Leads by Intent Score | Sort | Sorts qualified leads by descending intent score | Filter High Intent Leads | Send Slack Alert; Save Qualified Leads to Sheet | ## Scoring & Filtering<br>Combines signals into a weighted intent score, filters leads above threshold, and ranks by score. |
| Send Slack Alert | Slack | Sends a high-intent notification to Slack | Rank Leads by Intent Score |  | ## Alerts & Storage<br>Notifies Slack with lead details and saves qualified leads to Google Sheets. |
| Save Qualified Leads to Sheet | Google Sheets | Appends qualified leads to the output spreadsheet tab | Rank Leads by Intent Score |  | ## Alerts & Storage<br>Notifies Slack with lead details and saves qualified leads to Google Sheets. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node** named **Daily Prospect Scan Trigger**.
   - Node type: `Schedule Trigger`
   - Configure it to run **daily at 8 AM**
   - Confirm your n8n instance timezone is correct

3. **Add a Google Sheets node** named **Get Prospect List**.
   - Node type: `Google Sheets`
   - Connect it after **Daily Prospect Scan Trigger**
   - Authenticate with a Google Sheets credential that has access to your spreadsheet
   - Select the spreadsheet that contains your prospect list
   - Select the input tab named **Prospects**
   - Ensure the tab includes a column named **domain**
   - Keep row output in standard item form so each row becomes one item

4. **Prepare the input Google Sheet**.
   - Create a spreadsheet with a tab named **Prospects**
   - Add at minimum a header row with:
     - `domain`
   - Populate it with company domains such as:
     - `example.com`
     - `acme.io`

5. **Install/configure the PredictLeads n8n node** if not already available.
   - Required node type: `@predictleads/n8n-nodes-predictleads.predictLeads`
   - Obtain a PredictLeads API credential from [https://predictleads.com](https://predictleads.com)
   - Add the credential in n8n before testing the workflow

6. **Add a PredictLeads node** named **Fetch Job Openings**.
   - Connect it after **Get Prospect List**
   - Configure:
     - Resource: `jobOpenings`
     - Operation: `retrieveCompanyJobOpenings`
     - Domain: `{{ $('Get Prospect List').item.json.domain }}`
   - Attach your PredictLeads credential

7. **Add a second PredictLeads node** named **Fetch Tech Stack**.
   - Connect it after **Fetch Job Openings**
   - Configure:
     - Resource: `technologyDetections`
     - Operation: `retrieveTechnologiesUsedByCompany`
     - Domain: `{{ $('Get Prospect List').item.json.domain }}`
   - Use the same PredictLeads credential

8. **Add a third PredictLeads node** named **Fetch News Events**.
   - Connect it after **Fetch Tech Stack**
   - Configure:
     - Resource: `newsEvents`
     - Operation: `retrieveCompanyNewsEvents`
     - Domain: `{{ $('Get Prospect List').item.json.domain }}`
   - Use the same PredictLeads credential

9. **Add a Code node** named **Normalize Data and Score**.
   - Connect it after **Fetch News Events**
   - Use JavaScript mode
   - Paste logic equivalent to the following behavior:
     - For each item index:
       - Count jobs from `Fetch Job Openings.json.data.length`
       - Count technologies from `Fetch Tech Stack.json.data.length`
       - Count news events from current item `json.data.length`
       - Read domain from `Get Prospect List`
       - Compute score as:
         - `(jobs * 5) + (tech * 3) + (news * 2)`
       - Return an object with:
         - `domain`
         - `hiring_signal`
         - `tech_signal`
         - `news_signal`
         - `intent_score`
   - Important: this implementation assumes item order stays aligned across nodes

10. **Add an IF node** named **Filter High Intent Leads**.
    - Connect it after **Normalize Data and Score**
    - Configure one numeric condition:
      - Left value: `{{ $json.intent_score }}`
      - Operator: `>`
      - Right value: `20`
    - Use the **true** output as the main continuation path

11. **Add a Sort node** named **Rank Leads by Intent Score**.
    - Connect it from the **true** output of **Filter High Intent Leads**
    - Sort by:
      - Field: `intent_score`
      - Order: `descending`

12. **Add a Slack node** named **Send Slack Alert**.
    - Connect it after **Rank Leads by Intent Score**
    - Authenticate with a Slack OAuth2 credential
    - Choose **channel** as the destination type
    - Select the Slack channel where alerts should be posted
    - Use a message template containing:
      - company domain
      - intent score
      - hiring signal
      - tech signal
      - news signal
      - recommended next action

13. **Use a Slack message body similar to this**:
    - `🚀 High Intent Lead Detected!`
    - `Company: {{$json.domain}}`
    - `Intent Score: {{$json.intent_score}}`
    - `Signals Breakdown:`
    - `Hiring Activity: {{$json.hiring_signal}}`
    - `Tech Adoption: {{$json.tech_signal}}`
    - `News Mentions: {{$json.news_signal}}`
    - Add outreach guidance text as desired

14. **Create the output sheet tab** in the same spreadsheet or another spreadsheet.
    - Recommended tab name: **Save Qualified Leads to Sheet**
    - Add columns:
      - `domain`
      - `hiring_signal`
      - `tech_signal`
      - `news_signal`
      - `intent_score`

15. **Add a second Google Sheets node** named **Save Qualified Leads to Sheet**.
    - Connect it in parallel from **Rank Leads by Intent Score**
    - Operation: `Append`
    - Authenticate with Google Sheets credentials
    - Select the output spreadsheet and the output tab
    - Use manual mapping for these fields:
      - `domain` → `{{ $json.domain }}`
      - `hiring_signal` → `{{ $json.hiring_signal }}`
      - `tech_signal` → `{{ $json.tech_signal }}`
      - `news_signal` → `{{ $json.news_signal }}`
      - `intent_score` → `{{ $json.intent_score }}`

16. **Test the input path first**.
    - Run from **Get Prospect List**
    - Confirm each row produces an item with a valid `domain`

17. **Test each PredictLeads node**.
    - Validate that each response contains a `data` array
    - If no results exist, confirm the Code node will safely default counts to `0`

18. **Test the Code node output**.
    - Check that each item contains:
      - `domain`
      - `hiring_signal`
      - `tech_signal`
      - `news_signal`
      - `intent_score`

19. **Test the filter and ranking logic**.
    - Confirm only leads with score greater than `20` continue
    - Verify sort order is highest score first

20. **Test Slack delivery**.
    - Make sure the Slack app/bot is authorized and invited to the selected channel
    - Confirm messages render correctly with interpolated values

21. **Test sheet append behavior**.
    - Confirm qualified leads are appended as new rows
    - Verify the column names exactly match the node mapping

22. **Activate the workflow** once all tests pass.

## Credential configuration summary

- **Google Sheets credential**
  - Must have read access to the prospect tab
  - Must have write access to the qualified-leads tab

- **PredictLeads credential**
  - Must be valid for all three enrichment operations
  - Watch API quotas if scanning many domains daily

- **Slack OAuth2 credential**
  - Must allow posting to the target channel
  - The Slack app/bot must be present in that channel

## Constraints and implementation notes

- The workflow depends on a `domain` field in the input sheet.
- The Code node assumes all upstream items remain in the same order and count.
- There is no deduplication before writing to the output sheet.
- There is no retry or error-handling branch in the current design.
- The IF node discards low-scoring leads silently because the false branch is unused.

## Sub-workflow setup

This workflow does **not** use any sub-workflow or Execute Workflow node.  
It has a single entry point: **Daily Prospect Scan Trigger**.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Multi-Signal Buying Intent Scoring & Sales Prioritization. A daily schedule pulls domains from Google Sheets, queries PredictLeads for job openings, technology stack changes, and news events, combines them into a weighted score, filters leads above threshold, ranks them, posts to Slack, and stores them in Google Sheets. | Workflow-level note |
| Connect your Google Sheets credential and point the input node to a sheet with a `domain` column listing prospect company domains. | Setup guidance |
| Connect your PredictLeads API credential. | [https://predictleads.com](https://predictleads.com) |
| Connect your Slack credential and choose the channel where you want high-intent alerts. | Setup guidance |
| Create a second sheet tab for qualified leads output with columns: `domain`, `hiring_signal`, `tech_signal`, `news_signal`, `intent_score`. | Setup guidance |
| Adjust the intent score threshold in the Filter node; default is 20. | Customization |
| Change signal weights in the Code node to match your sales priorities. | Customization |
| Add more output channels such as email or CRM actions alongside Slack. | Customization |