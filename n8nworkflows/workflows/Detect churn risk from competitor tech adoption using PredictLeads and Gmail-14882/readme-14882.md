Detect churn risk from competitor tech adoption using PredictLeads and Gmail

https://n8nworkflows.xyz/workflows/detect-churn-risk-from-competitor-tech-adoption-using-predictleads-and-gmail-14882


# Detect churn risk from competitor tech adoption using PredictLeads and Gmail

Now I have all the information I need. Let me write the comprehensive document.### 1. Workflow Overview

This workflow automates **churn risk detection** by monitoring whether existing customers have adopted competitor technologies. It runs on a weekly schedule, loads a customer list from Google Sheets, queries each customer's technology stack via the PredictLeads API, and cross-references detected tools against a hardcoded competitor list (HubSpot, Salesforce, Zoho CRM). When a competitor tool is found on a customer account whose contract renewal falls within 90 days, a high-priority alert email is sent to the assigned Customer Success Manager via Gmail. Accounts without competitor adoption receive an informational Slack notification. Accounts with a competitor detected but no imminent renewal are logged but not escalated.

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Trigger & Customer Source | Weekly schedule trigger fetches the customer roster from a Google Sheet. |
| 2 | Tech Stack Analysis with PredictLeads | For each customer domain, retrieves the full technology detection list, splits it into individual records, limits volume, and extracts the technology ID for detail lookup. |
| 3 | Competitor Detection | Resolves each technology ID to a human-readable name, checks it against the competitor list, and routes items into two branches (competitor detected vs. safe). |
| 4 | Renewal Risk & Alerting | For flagged accounts, calculates days until renewal; if ≤ 90 days, sends a churn-risk email to the CSM via Gmail. Otherwise, the item is silently ignored. Safe accounts receive a Slack notification. |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Trigger & Customer Source

**Overview:** A cron-like weekly trigger fires at 6:00 AM, then a Google Sheets node reads the customer roster. Each row is expected to contain a company domain, a renewal date, and the CSM's email address.

**Nodes Involved:**
- Weekly Workflow Trigger
- Fetch Customers from Google Sheet

---

**Node: Weekly Workflow Trigger**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.scheduleTrigger` (v1.3) |
| **Technical Role** | Time-based entry point; fires once per week |
| **Configuration** | Interval rule: `triggerAtHour = 6` (runs daily at 06:00; the "Weekly" intent should be reinforced by adding a `triggerAtDayOfWeek` constraint if true weekly execution is desired) |
| **Key Expressions** | None |
| **Input** | None (root node) |
| **Output** | → Fetch Customers from Google Sheet |
| **Edge Cases** | If the n8n instance is down at 06:00, the trigger is missed for that day. No retry mechanism is built in. If weekly execution is intended, the current config fires **daily** at 06:00 — the `triggerAtDayOfWeek` parameter should be set (e.g., Monday). |

---

**Node: Fetch Customers from Google Sheet**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.googleSheets` (v4.7) |
| **Technical Role** | Reads all rows from a specified Google Sheet and emits one item per row |
| **Configuration** | Operation: Read (default). Sheet name selected by `gid=0` (first sheet). Document ID set to an empty value — **must be populated** with a real spreadsheet ID before execution. Options: none. |
| **Key Expressions** | None |
| **Input** | ← Weekly Workflow Trigger |
| **Output** | → Get Customer Tech Stack (PredictLeads). Each output item contains the full row data (expected columns: `company_domain`, `renewal_date`, `csm_email`). |
| **Required Credentials** | Google Sheets OAuth2 |
| **Edge Cases** | If the sheet is empty, the downstream PredictLeads node receives no items. If `company_domain` is missing or malformed, the PredictLeads API call may fail or return no results. Column names must match exactly (case-sensitive). The `documentId` value is currently blank — the workflow will fail at runtime until a valid ID is provided. |

---

#### Block 2 — Tech Stack Analysis with PredictLeads

**Overview:** For each customer domain, this block calls the PredictLeads API to retrieve all technology detections, splits the array of detections into individual items, limits the item count per customer to control volume and cost, and extracts the nested technology ID for subsequent detail lookups.

**Nodes Involved:**
- Get Customer Tech Stack (PredictLeads)
- Split Technologies Per Customer
- Limit Technologies for Processing
- Extract Technology ID

---

**Node: Get Customer Tech Stack (PredictLeads)**

| Property | Detail |
|----------|--------|
| **Type** | `@predictleads/n8n-nodes-predictleads.predictLeads` (v1) |
| **Technical Role** | API call — retrieves all technologies detected for a given company domain |
| **Configuration** | Resource: `technologyDetections`. Operation: `retrieveTechnologiesUsedByCompany`. Domain: `={{ $json.company_domain }}` (dynamically reads from the Google Sheet row). RequestOptions: empty. |
| **Key Expressions** | `={{ $json.company_domain }}` |
| **Input** | ← Fetch Customers from Google Sheet |
| **Output** | → Split Technologies Per Customer. The response includes a `data` array of technology detection objects (JSON:API format). |
| **Required Credentials** | PredictLeads API credentials (X-Api-Key and X-Api-Token headers) |
| **Edge Cases** | Rate limiting on the PredictLeads API (not handled — no retry/delay logic). If `company_domain` is invalid or the company has no detections, `data` may be empty or missing, causing the downstream Split node to produce zero items. The `credentials` object on this node is currently empty — credentials must be attached before execution. |

---

**Node: Split Technologies Per Customer**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.splitOut` (v1) |
| **Technical Role** | Flattens the `data` array from the PredictLeads response into one item per technology detection |
| **Configuration** | Field to split out: `data`. Options: none. |
| **Key Expressions** | None |
| **Input** | ← Get Customer Tech Stack (PredictLeads) |
| **Output** | → Limit Technologies for Processing. Each output item contains a single technology detection object with nested `relationships.technology.data.id`. |
| **Edge Cases** | If `data` is not an array or is absent, the node will error. If `data` is empty, no items are emitted downstream. |

---

**Node: Limit Technologies for Processing**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.limit` (v1) |
| **Technical Role** | Caps the number of technology items passing through to control downstream API calls and processing time |
| **Configuration** | Max items: `2` |
| **Key Expressions** | None |
| **Input** | ← Split Technologies Per Customer |
| **Output** | → Extract Technology ID. Only the first 2 technology detections per customer pass through. |
| **Edge Cases** | Setting `maxItems = 2` means at most 2 technologies per customer are analyzed. If a competitor tool is the 3rd or later detection, it will be **silently missed**. Increase this value for broader coverage. If fewer than 2 items arrive, all pass through without error. |

---

**Node: Extract Technology ID**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Extracts the nested technology ID from the JSON:API relationships structure and promotes it to a top-level field |
| **Configuration** | Assignments: `tech_Id` (string) = `={{ $json.relationships.technology.data.id }}`. Options: none. |
| **Key Expressions** | `={{ $json.relationships.technology.data.id }}` |
| **Input** | ← Limit Technologies for Processing |
| **Output** | → Get Technology Details by ID. Each item now has a `tech_Id` field alongside the original data. |
| **Edge Cases** | If the `relationships.technology.data.id` path does not exist (e.g., malformed API response or different resource shape), the expression will resolve to `undefined`, causing the downstream PredictLeads lookup to fail or return an error. |

---

#### Block 3 — Competitor Detection

**Overview:** This block resolves each technology ID to its human-readable name, checks whether that name matches any entry in a hardcoded competitor list, and then branches the flow: accounts with a detected competitor proceed to renewal risk calculation, while safe accounts receive an informational Slack message.

**Nodes Involved:**
- Get Technology Details by ID
- Detect Competitor Usage
- Is Competitor Detected?
- Notify Team (No Competitor - Safe Account)

---

**Node: Get Technology Details by ID**

| Property | Detail |
|----------|--------|
| **Type** | `@predictleads/n8n-nodes-predictleads.predictLeads` (v1) |
| **Technical Role** | API call — retrieves the full technology record for a given ID |
| **Configuration** | Resource: `technologies`. Operation: `retrieveSingleTechnologyByIdOrFuzzyName`. ID or fuzzy name: `={{ $json.tech_Id }}`. RequestOptions: empty. |
| **Key Expressions** | `={{ $json.tech_Id }}` |
| **Input** | ← Extract Technology ID |
| **Output** | → Detect Competitor Usage. The response contains `data[0].attributes.name` (the technology's display name) and `data[0].attributes.domain`. |
| **Required Credentials** | PredictLeads API credentials (same as earlier node; `credentials` object currently empty — must be attached) |
| **Edge Cases** | If `tech_Id` is invalid, the API may return a 404 or an empty `data` array. The downstream Code node accesses `data[0]` which will throw on an empty array. Rate limiting risk again — each technology requires a separate API call. |

---

**Node: Detect Competitor Usage**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Custom JavaScript logic — identifies whether the detected technology belongs to the competitor list and excludes self-references (e.g., a company whose domain contains the technology name) |
| **Configuration** | Language: JavaScript. Code (summarized logic): |
| | 1. Defines competitor list: `["HubSpot", "Salesforce", "Zoho CRM"]` |
| | 2. Iterates over all input items via `$input.all()` |
| | 3. Extracts `techName` from `item.json.data[0].attributes.name` |
| | 4. Extracts `domain` from `item.json.company_domain` |
| | 5. Performs case-insensitive substring match of each competitor name against `techName` |
| | 6. Checks if the company domain contains the technology name (self-reference guard) |
| | 7. Sets `competitorDetected = isCompetitor && !isSameCompany` |
| | 8. Returns items with added fields: `techName` and `competitorDetected` |
| **Key Expressions** | `$input.all()`, `$json.data?.[0]?.attributes?.name`, `$json.company_domain` |
| **Input** | ← Get Technology Details by ID |
| **Output** | → Is Competitor Detected? |
| **Edge Cases** | If `data[0]` does not exist, the optional chaining `?.` returns `""`, so `techName` becomes empty and no competitor will match — a false negative. The substring matching is broad: a technology named "Zoho CRM Plus" will match "Zoho CRM", but a technology named "SalesforceIQ" would also match "Salesforce". The self-reference check (`domain.includes(techLower)`) could produce false positives if the domain coincidentally contains a competitor substring. The competitor list is hardcoded — any changes require editing this node's code. |

---

**Node: Is Competitor Detected?**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.if` (v2.3) |
| **Technical Role** | Conditional router — splits the flow based on whether a competitor was detected |
| **Configuration** | Combinator: AND. Condition: `competitorDetected` (boolean) **equals** `false`. Case sensitive: true. Type validation: strict. |
| **Key Expressions** | `={{$json["competitorDetected"]}}` |
| **Input** | ← Detect Competitor Usage |
| **Output (true branch, index 0)** | → Calculate Renewal Risk (90 Days) — fires when `competitorDetected === false` |
| **Output (false branch, index 1)** | → Notify Team (No Competitor - Safe Account) — fires when `competitorDetected === true` |
| **Edge Cases** | ⚠️ **Potential Logic Inversion** — The condition checks for `competitorDetected === false`, meaning the TRUE output (index 0) activates when **no** competitor is found, routing to renewal risk calculation. The FALSE output (index 1) activates when a competitor **is** found, routing to the "No Competitor - Safe Account" Slack notification. This appears inverted: the Slack notification labeled "No Competitor" fires when a competitor IS detected, and the renewal risk path fires when no competitor is detected. The likely intended logic is the reverse — condition should be `competitorDetected equals true`, or the output connections should be swapped. This is a critical bug that would cause competitor detections to trigger "safe account" messages and non-competitor accounts to receive unnecessary renewal risk evaluation. |

---

**Node: Notify Team (No Competitor - Safe Account)**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.slack` (v2.4) |
| **Technical Role** | Sends an informational Slack message confirming no competitor technology was detected |
| **Configuration** | Authentication: OAuth2. Channel selection: list mode (channel ID must be configured — currently empty). Message template (interpolated): |
| | `📊 Weekly Customer Tech Check Update` |
| | `Company: {{ $json.data[0].attributes.domain }}` |
| | `Detected Tool: {{ $json["techName"] }}` |
| | `Status: ✅ No competitor detected` |
| | `Insight: Customer is currently not using any tracked competitor tools…` |
| | `Next Step: Continue regular engagement…` |
| | `— Automated Churn Monitoring System` |
| **Key Expressions** | `$json.data[0].attributes.domain`, `$json["techName"]` |
| **Input** | ← Is Competitor Detected? (false branch — index 1) |
| **Output** | None (terminal node) |
| **Required Credentials** | Slack OAuth2 |
| **Edge Cases** | The `channelId` value is empty — execution will fail at runtime until a valid channel is selected. If `$json.data[0]` is undefined (malformed or missing PredictLeads response), the expression will error. ⚠️ Due to the logic inversion in the If node, this notification may fire for accounts where a competitor IS detected, sending a misleading "safe" message. |

---

#### Block 4 — Renewal Risk & Alerting

**Overview:** For accounts flagged as having competitor technology, this block calculates the number of days until contract renewal. If the renewal falls within 90 days, a high-priority alert email is dispatched to the assigned CSM via Gmail. If the renewal is more than 90 days away, the item is silently discarded.

**Nodes Involved:**
- Calculate Renewal Risk (90 Days)
- Is Renewal Within 90 Days?
- Send Churn Risk Alert Email to CSM
- Ignore (No Immediate Risk)

---

**Node: Calculate Renewal Risk (90 Days)**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Computes days-to-renewal and flags whether the renewal falls within a 90-day window |
| **Configuration** | Language: JavaScript. Code logic (summarized): |
| | 1. Iterates over all input items via `$input.all()` |
| | 2. For each item, retrieves the corresponding Google Sheet row by index: `$items("Fetch Customers from Google Sheet")[index]` |
| | 3. Parses `renewal_date` from that row into a Date object |
| | 4. Computes `diffDays = (renewalDate - today) / (1000 * 60 * 60 * 24)` |
| | 5. Adds `daysToRenewal` (rounded) and `within90Days` (boolean: `diffDays <= 90`) to each item |
| **Key Expressions** | `$items("Fetch Customers from Google Sheet")[index]`, `$input.all()` |
| **Input** | ← Is Competitor Detected? (true branch — index 0) |
| **Output** | → Is Renewal Within 90 Days? |
| **Edge Cases** | The index-based correlation between the current item (which has been split/limited from the tech stack) and the original Google Sheet row is fragile. If any items were dropped by the Limit node or if the Split node changed the item order, the index mapping will be misaligned, associating the wrong `renewal_date` with the wrong customer. If `renewal_date` is not a valid date string, `new Date(sheetItem.json.renewal_date)` returns `Invalid Date` and `diffDays` becomes `NaN`. Negative `diffDays` (past renewal dates) will correctly set `within90Days = true`, which may or may not be desired. ⚠️ Due to the If node logic inversion, this node currently receives items where **no competitor was detected**, making the renewal risk calculation irrelevant for those items. |

---

**Node: Is Renewal Within 90 Days?**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.if` (v2.3) |
| **Technical Role** | Routes items based on whether the renewal is within 90 days |
| **Configuration** | Combinator: AND. Condition: `within90Days` (boolean) **equals** `true`. Case sensitive: true. Type validation: strict. |
| **Key Expressions** | `={{$json["within90Days"]}}` |
| **Input** | ← Calculate Renewal Risk (90 Days) |
| **Output (true branch, index 0)** | → Send Churn Risk Alert Email to CSM — fires when `within90Days === true` |
| **Output (false branch, index 1)** | → Ignore (No Immediate Risk) — fires when `within90Days === false` |
| **Edge Cases** | This conditional logic is correct as described. However, if the upstream mapping is broken (see Calculate Renewal Risk), the boolean may be based on the wrong customer's renewal date. |

---

**Node: Send Churn Risk Alert Email to CSM**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.gmail` (v2.2) |
| **Technical Role** | Sends a churn risk alert email to the Customer Success Manager assigned to the at-risk account |
| **Configuration** | Email type: text. Send to: `={{ $('Fetch Customers from Google Sheet').item.json.csm_email }}` (references the Google Sheet node's current item for the CSM email). Subject: `🚨 Churn Risk Alert - {{ $json.data[0].attributes.domain }}`. Message body (interpolated): |
| | `Hi,` |
| | `Churn Risk Alert for your assigned account:` |
| | `Company: {{ $json.data[0].attributes.domain }}` |
| | `Detected Tool: {{ $json["techName"] }}` |
| | `Days to Renewal: {{ $json["daysToRenewal"] }}` |
| | `Renewal Window: Within 90 days` |
| | `Recommended Action: Please engage with the customer proactively…` |
| | `Automated Churn Monitoring System` |
| | Options: none. |
| **Key Expressions** | `$('Fetch Customers from Google Sheet').item.json.csm_email`, `$json.data[0].attributes.domain`, `$json["techName"]`, `$json["daysToRenewal"]` |
| **Input** | ← Is Renewal Within 90 Days? (true branch) |
| **Output** | None (terminal node) |
| **Required Credentials** | Gmail OAuth2 |
| **Edge Cases** | The expression `$('Fetch Customers from Google Sheet').item.json.csm_email` uses the `.item` accessor which references the item at the **same index** in the referenced node. If items have been split or limited, index alignment may be off (same fragility as in Calculate Renewal Risk). If `csm_email` is empty or invalid, the Gmail send will fail. If `$json.data[0]` is undefined, the subject expression errors. Gmail API daily send quotas may be hit for large customer lists. |

---

**Node: Ignore (No Immediate Risk)**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.noOp` (v1) |
| **Technical Role** | Silent discard — consumes items that do not meet the alert threshold without any action |
| **Configuration** | None |
| **Key Expressions** | None |
| **Input** | ← Is Renewal Within 90 Days? (false branch) |
| **Output** | None (terminal node) |
| **Edge Cases** | No errors possible. Items are simply consumed and not passed further. No audit trail is created for these "ignored" accounts — consider adding a logging node if traceability is needed. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|---------------|-----------------|-------------|
| Weekly Workflow Trigger | scheduleTrigger | Weekly scheduled entry point | — | Fetch Customers from Google Sheet | ## Trigger & Customer Source — Weekly trigger loads customer data from Google Sheets. |
| Fetch Customers from Google Sheet | googleSheets | Read customer roster (domain, renewal date, CSM email) | Weekly Workflow Trigger | Get Customer Tech Stack (PredictLeads) | ## Trigger & Customer Source — Weekly trigger loads customer data from Google Sheets. |
| Get Customer Tech Stack (PredictLeads) | predictLeads | Retrieve all technology detections for a company domain | Fetch Customers from Google Sheet | Split Technologies Per Customer | ## Tech Stack Analysis with Predict Leads — Fetches tech stack, splits records, limits processing, extracts IDs. |
| Split Technologies Per Customer | splitOut | Flatten the `data` array into individual technology detection items | Get Customer Tech Stack (PredictLeads) | Limit Technologies for Processing | ## Tech Stack Analysis with Predict Leads — Fetches tech stack, splits records, limits processing, extracts IDs. |
| Limit Technologies for Processing | limit | Cap technology items to 2 per customer | Split Technologies Per Customer | Extract Technology ID | ## Tech Stack Analysis with Predict Leads — Fetches tech stack, splits records, limits processing, extracts IDs. |
| Extract Technology ID | set | Extract nested technology ID to a top-level field | Limit Technologies for Processing | Get Technology Details by ID | ## Tech Stack Analysis with Predict Leads — Fetches tech stack, splits records, limits processing, extracts IDs. |
| Get Technology Details by ID | predictLeads | Resolve technology ID to its display name and details | Extract Technology ID | Detect Competitor Usage | ## Competitor Detection — Looks up tech details, detects competitor tools, routes accounts. |
| Detect Competitor Usage | code | Check technology name against competitor list; flag matches | Get Technology Details by ID | Is Competitor Detected? | ## Competitor Detection — Looks up tech details, detects competitor tools, routes accounts. |
| Is Competitor Detected? | if | Branch based on competitor detection flag | Detect Competitor Usage | Calculate Renewal Risk (90 Days) (true), Notify Team (No Competitor - Safe Account) (false) | ## Competitor Detection — Looks up tech details, detects competitor tools, routes accounts. |
| Notify Team (No Competitor - Safe Account) | slack | Send informational Slack message for safe accounts | Is Competitor Detected? (false branch) | — | ## Churn Risk Detection via Competitor Tech Adoption Using Predict leads — ### How it works: 1. Scheduled trigger runs weekly and loads customer domains from Google Sheets; 2. Fetches each customer's tech stack from PredictLeads API; 3. Detects if customer adopted competitor tools (HubSpot, Salesforce, Zoho CRM); 4. Flags accounts where competitors are detected; 5. Calculates renewal timing - alerts CSM for high-risk accounts (competitor + within 90 days). ### Setup: 1. Create Google Sheet with columns: company_domain, renewal_date, csm_email; 2. Connect Gmail OAuth2 for CSM alerts; 3. Connect Slack OAuth2 for team notifications (optional); 4. Add PredictLeads API credentials (X-Api-Key, X-Api-Token). ### Customization: - Adjust competitor list in "Detect Competitor Usage" Code node; - Modify 90-day threshold in "Calculate Renewal Risk" node. |
| Calculate Renewal Risk (90 Days) | code | Compute days to renewal and flag if within 90-day window | Is Competitor Detected? (true branch) | Is Renewal Within 90 Days? | ## Renewal Risk & Alerting — Calculates days to renewal, alerts CSM for high-risk accounts. |
| Is Renewal Within 90 Days? | if | Branch based on 90-day renewal window | Calculate Renewal Risk (90 Days) | Send Churn Risk Alert Email to CSM (true), Ignore (No Immediate Risk) (false) | ## Renewal Risk & Alerting — Calculates days to renewal, alerts CSM for high-risk accounts. |
| Send Churn Risk Alert Email to CSM | gmail | Send churn risk alert email to the CSM | Is Renewal Within 90 Days? (true branch) | — | ## Renewal Risk & Alerting — Calculates days to renewal, alerts CSM for high-risk accounts. |
| Ignore (No Immediate Risk) | noOp | Silently discard items not meeting alert criteria | Is Renewal Within 90 Days? (false branch) | — | ## Renewal Risk & Alerting — Calculates days to renewal, alerts CSM for high-risk accounts. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance.

2. **Add the Schedule Trigger node.**
   - Type: `Schedule Trigger` (v1.3).
   - Configuration: Set `triggerAtHour` to `6`. If weekly execution is intended, also set `triggerAtDayOfWeek` (e.g., Monday).
   - This node has no input; it is the workflow entry point.

3. **Add the Google Sheets node.**
   - Type: `Google Sheets` (v4.7).
   - Operation: `Read` (default).
   - Document ID: Select the target Google Sheet from your connected Drive (the value is currently blank in the template).
   - Sheet Name: Select the first sheet (`gid=0`).
   - Options: default (none).
   - Connect: Output of **Schedule Trigger** → Input of this node.
   - **Credential:** Create and assign a Google Sheets OAuth2 credential.

4. **Prepare the Google Sheet** (prerequisite, not a node):
   - Create a Google Sheet with at minimum these columns: `company_domain` (e.g., `acme.com`), `renewal_date` (e.g., `2025-09-15`), `csm_email` (e.g., `jane@example.com`).
   - Populate it with one row per customer.

5. **Add the PredictLeads — Tech Stack node.**
   - Type: `PredictLeads` (v1, community node `@predictleads/n8n-nodes-predictleads`).
   - Resource: `technologyDetections`.
   - Operation: `retrieveTechnologiesUsedByCompany`.
   - Domain: `={{ $json.company_domain }}`.
   - RequestOptions: empty.
   - Connect: Output of **Google Sheets** → Input of this node.
   - **Credential:** Create and assign PredictLeads API credentials (requires `X-Api-Key` and `X-Api-Token` headers).

6. **Add the Split Out node.**
   - Type: `Split Out` (v1).
   - Field to split out: `data`.
   - Options: default.
   - Connect: Output of **PredictLeads (tech stack)** → Input of this node.

7. **Add the Limit node.**
   - Type: `Limit` (v1).
   - Max items: `2` (adjust as needed for broader coverage).
   - Connect: Output of **Split Out** → Input of this node.

8. **Add the Set node (Extract Technology ID).**
   - Type: `Set` (v3.4).
   - Assignment 1: Name = `tech_Id`, Type = `String`, Value = `={{ $json.relationships.technology.data.id }}`.
   - Options: default.
   - Connect: Output of **Limit** → Input of this node.

9. **Add the PredictLeads — Technology Details node.**
   - Type: `PredictLeads` (v1).
   - Resource: `technologies`.
   - Operation: `retrieveSingleTechnologyByIdOrFuzzyName`.
   - ID or fuzzy name: `={{ $json.tech_Id }}`.
   - RequestOptions: empty.
   - Connect: Output of **Set (Extract Technology ID)** → Input of this node.
   - **Credential:** Same PredictLeads credential as step 5.

10. **Add the Code node (Detect Competitor Usage).**
    - Type: `Code` (v2).
    - Language: JavaScript.
    - Paste the following logic:
      - Define competitor list: `["HubSpot", "Salesforce", "Zoho CRM"]`.
      - Iterate over `$input.all()`.
      - Extract `techName` from `item.json.data[0].attributes.name`.
      - Extract `domain` from `item.json.company_domain`.
      - Case-insensitive substring match each competitor against `techName`.
      - Self-reference guard: skip if `domain` includes `techName` (lowercased).
      - Set `competitorDetected = isCompetitor && !isSameCompany`.
      - Return items with added `techName` and `competitorDetected` fields.
    - Connect: Output of **PredictLeads (technology details)** → Input of this node.

11. **Add the If node (Is Competitor Detected?).**
    - Type: `If` (v2.3).
    - Condition: `competitorDetected` (boolean) **equals** `true`. ⚠️ **Critical fix:** The original workflow checks for `equals false`, which inverts the routing. Set to `equals true` so that the true branch handles competitor detections correctly.
    - Combinator: AND. Type validation: strict. Case sensitive: true.
    - Connect: Output of **Code (Detect Competitor Usage)** → Input of this node.
    - **True branch (output 0):** → **Calculate Renewal Risk (90 Days)**.
    - **False branch (output 1):** → **Notify Team (No Competitor - Safe Account)**.

12. **Add the Slack node (Notify Team — Safe Account).**
    - Type: `Slack` (v2.4).
    - Authentication: OAuth2.
    - Channel: Select the target Slack channel (currently empty — must be configured).
    - Message: Template as described in Block 3 analysis (includes `{{ $json.data[0].attributes.domain }}`, `{{ $json["techName"] }}`).
    - Connect: **If (Is Competitor Detected?) false branch** → Input of this node.
    - **Credential:** Create and assign Slack OAuth2 credential.

13. **Add the Code node (Calculate Renewal Risk — 90 Days).**
    - Type: `Code` (v2).
    - Language: JavaScript.
    - Paste the following logic:
      - Iterate over `$input.all()`.
      - For each item at index `i`, retrieve the matching Google Sheet row: `$items("Fetch Customers from Google Sheet")[i]`.
      - Parse `renewal_date` from that row.
      - Compute `diffDays = (renewalDate - today) / milliseconds-per-day`.
      - Add `daysToRenewal` (rounded) and `within90Days` (boolean: `diffDays <= 90`).
    - Connect: **If (Is Competitor Detected?) true branch** → Input of this node.

14. **Add the If node (Is Renewal Within 90 Days?).**
    - Type: `If` (v2.3).
    - Condition: `within90Days` (boolean) **equals** `true`.
    - Combinator: AND. Type validation: strict. Case sensitive: true.
    - Connect: Output of **Code (Calculate Renewal Risk)** → Input of this node.
    - **True branch (output 0):** → **Send Churn Risk Alert Email to CSM**.
    - **False branch (output 1):** → **Ignore (No Immediate Risk)**.

15. **Add the Gmail node (Send Churn Risk Alert Email to CSM).**
    - Type: `Gmail` (v2.2).
    - Email type: `text`.
    - Send to: `={{ $('Fetch Customers from Google Sheet').item.json.csm_email }}`.
    - Subject: `🚨 Churn Risk Alert - {{ $json.data[0].attributes.domain }}`.
    - Message: Template as described in Block 4 analysis (includes company domain, detected tool, days to renewal, recommended action).
    - Options: default.
    - Connect: **If (Is Renewal Within 90 Days?) true branch** → Input of this node.
    - **Credential:** Create and assign Gmail OAuth2 credential.

16. **Add the NoOp node (Ignore — No Immediate Risk).**
    - Type: `No Operation` (v1).
    - No configuration required.
    - Connect: **If (Is Renewal Within 90 Days?) false branch** → Input of this node.

17. **Install the PredictLeads community node** (prerequisite, if not already installed):
    - In n8n, go to Settings → Community Nodes → Install `@predictleads/n8n-nodes-predictleads`.

18. **Verify all credentials:**
    - Google Sheets OAuth2 (read access to the target spreadsheet).
    - PredictLeads API (X-Api-Key and X-Api-Token).
    - Slack OAuth2 (post messages to the selected channel).
    - Gmail OAuth2 (send emails on behalf of the authenticated user).

19. **Set the Google Sheet Document ID** in step 3 — the template ships with an empty value; the workflow will not run until this is populated.

20. **Set the Slack Channel ID** in step 12 — the template ships with an empty value; the Slack node will error at runtime without it.

21. **Test the workflow** with a small dataset (1–2 customers) and verify:
    - Competitor detection correctly identifies matching tools.
    - The If node routing is correct (competitor detected → renewal path; no competitor → Slack notification).
    - Days-to-renewal calculation aligns with the correct customer row.
    - Gmail sends to the correct CSM email.
    - Slack posts to the correct channel.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ⚠️ **Critical Logic Bug:** The "Is Competitor Detected?" If node checks for `competitorDetected equals false`, which causes the true branch (no competitor) to route to renewal risk calculation and the false branch (competitor detected) to route to the "No Competitor - Safe Account" Slack notification. This is inverted. The condition should be `competitorDetected equals true`, or the two output connections should be swapped. | Node: Is Competitor Detected? |
| The Schedule Trigger is configured with only `triggerAtHour: 6`, which fires **daily** at 06:00. For true weekly execution, add a `triggerAtDayOfWeek` parameter (e.g., `"triggerAtDayOfWeek": ["1"]` for Monday). | Node: Weekly Workflow Trigger |
| The `Limit` node is set to `maxItems: 2`, meaning only the first 2 technologies per customer are analyzed. If a competitor tool appears as the 3rd or later detection, it will be silently missed. Increase this value for production use. | Node: Limit Technologies for Processing |
| The index-based correlation in "Calculate Renewal Risk (90 Days)" between split technology items and original Google Sheet rows is fragile. If items are dropped or reordered upstream, the wrong `renewal_date` may be associated with the wrong customer. A safer approach would be to carry `company_domain`, `renewal_date`, and `csm_email` forward through every node (via Set nodes) and reference them directly rather than by index. | Node: Calculate Renewal Risk (90 Days) |
| PredictLeads API credentials are declared on two nodes but the `credentials` object is empty in the template. Both PredictLeads nodes will fail until credentials are attached. | Nodes: Get Customer Tech Stack, Get Technology Details by ID |
| The Google Sheet Document ID and the Slack Channel ID are blank in the template. Both must be populated before the workflow can execute successfully. | Nodes: Fetch Customers from Google Sheet, Notify Team (No Competitor - Safe Account) |
| No error handling or retry logic is present. If the PredictLeads API rate-limits, the Gmail API hits send quotas, or Slack fails, the workflow will error out on that item without fallback. Consider adding Error Trigger or error branches for production resilience. | Global |
| The "Ignore (No Immediate Risk)" node silently discards items. If auditability is required, replace it with a logging node (e.g., append to a Google Sheet or send to a logging channel). | Node: Ignore (No Immediate Risk) |
| The competitor list is hardcoded in the Code node. For easier maintenance, consider moving it to a workflow variable, a Google Sheet, or an environment variable. | Node: Detect Competitor Usage |
| The Slack notification for safe accounts references `$json.data[0].attributes.domain` for the company domain, but this value comes from the PredictLeads response, not the original customer record. If the PredictLeads domain differs from the customer's canonical domain, the notification may display inconsistent data. | Node: Notify Team (No Competitor - Safe Account) |
| PredictLeads community node package: `@predictleads/n8n-nodes-predictleads` must be installed before the workflow can be activated. | Prerequisite |