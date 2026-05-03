Score and route leads with Clearbit, Mattermost and Trello

https://n8nworkflows.xyz/workflows/score-and-route-leads-with-clearbit--mattermost-and-trello-12751


# Score and route leads with Clearbit, Mattermost and Trello

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow periodically retrieves new leads from a CRM/form API, enriches each lead using Clearbit (person/company data), calculates a lead score, and routes the lead into Trello lists (Qualified / Unqualified / Needs Research). It also posts alerts to Mattermost for Sales (qualified) and Ops (enrichment failures).

**Primary use cases:**
- Automated lead enrichment and scoring from inbound form submissions
- Sales-ready lead routing with minimal manual triage
- Operational visibility into enrichment failures (research queue)

### 1.1 Scheduled Intake (Trigger → Fetch → Parse)
Runs every hour, fetches new leads from an external API, and converts the returned array into individual n8n items.

### 1.2 Per-Lead Processing & Enrichment (Batching → Clearbit → Response Gate)
Processes leads one-by-one (rate-limit friendly), calls Clearbit enrichment, and branches based on HTTP status.

### 1.3 Merge & Score (Merge → Code Score → Qualification Decision)
Combines the original lead item with enrichment data, computes a numeric score, then checks whether the lead meets the qualification threshold.

### 1.4 Storage & Notifications (Trello + Mattermost by outcome)
Creates Trello cards in outcome-specific lists and sends Mattermost notifications to the appropriate channel/team.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Intake
**Overview:**  
Triggers hourly, calls the lead source API, then transforms the returned lead list into individual items for downstream processing.

**Nodes involved:**
- Daily Lead Check
- Fetch New Leads
- Parse Leads

#### Node: Daily Lead Check
- **Type / role:** Schedule Trigger — entry point that starts the workflow on a time interval.
- **Configuration:** Runs every `hours` interval (effectively every hour).
- **Inputs / outputs:** No inputs (trigger). Output → **Fetch New Leads**.
- **Version notes:** `typeVersion 1.1` (standard schedule trigger behavior).
- **Edge cases / failures:**
  - If n8n instance is paused/down, schedules can be missed depending on n8n settings and hosting.

#### Node: Fetch New Leads
- **Type / role:** HTTP Request — retrieves new leads from an external CRM/form endpoint.
- **Configuration choices:**
  - `GET https://api.yourcrm.com/leads/new`
  - Timeout: 30s
  - Auth: predefined credential type `httpHeaderAuth` (expects an HTTP Header Auth credential such as API key header).
- **Key data expectations:** Response should contain `items[0].json.leads` as an array.
- **Inputs / outputs:** Input from **Daily Lead Check**. Output → **Parse Leads**.
- **Version notes:** `typeVersion 4.2` (HTTP Request node behavior/fields match newer n8n versions).
- **Edge cases / failures:**
  - 401/403 if header auth is missing/invalid.
  - Non-JSON responses or changed payload shape (e.g., `leads` not present) will cause downstream parsing to produce zero items.
  - Timeouts for slow CRM endpoint.

#### Node: Parse Leads
- **Type / role:** Code — converts an array of leads into one item per lead.
- **Configuration choices (interpreted):**
  - Reads `items[0].json.leads || []`
  - Returns `leads.map(lead => ({ json: lead }))`
- **Inputs / outputs:** Input from **Fetch New Leads**. Output → **Split Leads**.
- **Version notes:** `typeVersion 2` (Code node with `jsCode`).
- **Edge cases / failures:**
  - If `items[0].json.leads` is not an array, the node will behave unexpectedly (could map over non-array only if coerced; here it defaults to `[]` if falsy, but won’t guard against an object).
  - If the CRM returns leads nested elsewhere, no leads will be processed.

---

### Block 2 — Per-Lead Processing & Enrichment
**Overview:**  
Ensures leads are processed in controlled batches (one at a time), enriches each email via Clearbit, then branches based on whether Clearbit responded successfully.

**Nodes involved:**
- Split Leads
- Enrich with Clearbit
- Check Clearbit Response
- Fallback Score

#### Node: Split Leads
- **Type / role:** Split In Batches — iterates through lead items in batches.
- **Configuration choices:**
  - Default options (batch size not explicitly set in parameters; n8n default is typically 1 unless configured otherwise in UI—verify in your instance UI).
- **Inputs / outputs (important):**
  - Input from **Parse Leads**.
  - Output **0** → **Enrich with Clearbit** (the “current batch” item(s)).
  - Output **1** → **Merge Lead & Enrichment** (this is a non-standard usage: output 1 is typically “No items left” / continuation, not a parallel pass-through of original items).
- **Version notes:** `typeVersion 3`.
- **Edge cases / failures:**
  - **Potential design issue:** The connection from Split Leads output index 1 to **Merge Lead & Enrichment** suggests the workflow is trying to supply the “original lead” to the merge node. In most n8n patterns, SplitInBatches does *not* emit the original item on a second output for merging; it emits a “done” signal. This can cause the merge-by-index to misalign or not receive expected input.
  - If processing many items, you typically need a loop back from downstream nodes to SplitInBatches to request the next batch. This workflow does not show a loop-back, so it may only process the first batch and stop.

#### Node: Enrich with Clearbit
- **Type / role:** HTTP Request — calls Clearbit Person API for enrichment.
- **Configuration choices:**
  - URL: `https://person.clearbit.com/v2/people/find?email={{ $json.email }}`
  - Timeout: 20s
  - TLS: do not allow unauthorized certs
  - Auth: predefined credential type `httpHeaderAuth` (typically `Authorization: Bearer <clearbit_key>`).
- **Key expressions / variables:**
  - `{{ $json.email }}` must exist in the lead item.
- **Inputs / outputs:** Input from **Split Leads**. Output → **Check Clearbit Response**.
- **Version notes:** `typeVersion 4.2`.
- **Edge cases / failures:**
  - Missing/invalid email → Clearbit may return 400/422.
  - 401 if Clearbit key missing/invalid.
  - 402/429 rate limiting depending on Clearbit plan and volume.
  - **Continue On Fail:** The sticky note claims it is enabled, but the node JSON does not explicitly show `continueOnFail`. Confirm in node settings; without it, an enrichment error can stop the workflow.

#### Node: Check Clearbit Response
- **Type / role:** IF — routes based on whether Clearbit returned a “successful” status code.
- **Configuration choices:**
  - Condition: `Number: value1 = {{$json.statusCode}}` is `< 400`
  - True branch: success path
  - False branch: failure path
- **Inputs / outputs:**
  - Input from **Enrich with Clearbit**
  - **True** → **Merge Lead & Enrichment**
  - **False** → **Fallback Score**
- **Version notes:** `typeVersion 2`.
- **Edge cases / failures:**
  - The IF expects `$json.statusCode` to exist. By default, HTTP Request nodes may not include a `statusCode` field unless “Full Response” (or similar option) is enabled. If absent, the expression may evaluate to `undefined`, which can cause routing errors or always-false behavior depending on n8n evaluation.
  - If “Continue On Fail” is enabled, the error output JSON shape differs; ensure status code is accessible consistently.

#### Node: Fallback Score
- **Type / role:** Code — assigns a minimal score when enrichment fails.
- **Configuration choices:**
  - Sets `lead.score = 10` and returns the modified item.
- **Inputs / outputs:** Input from **Check Clearbit Response** (false). Output → **Create Trello Card - Needs Research**.
- **Version notes:** `typeVersion 2`.
- **Edge cases / failures:**
  - Assumes `items[0].json` contains at least `name`/`email` for downstream Trello description; if not, cards may be incomplete.

---

### Block 3 — Merge & Score
**Overview:**  
Merges original lead data with Clearbit enrichment, computes a numeric score, then branches to qualified/unqualified.

**Nodes involved:**
- Merge Lead & Enrichment
- Calculate Lead Score
- IF Qualified?

#### Node: Merge Lead & Enrichment
- **Type / role:** Merge — combines two inputs into one item.
- **Configuration choices:**
  - Mode: `mergeByIndex` (pairs item 0 from input 1 with item 0 from input 2, etc.).
- **Inputs / outputs:**
  - Input 1: from **Check Clearbit Response** (true) (Clearbit payload)
  - Input 2: from **Split Leads** output index 1 (intended to be original lead; see concern above)
  - Output → **Calculate Lead Score**
- **Version notes:** `typeVersion 2.1`.
- **Edge cases / failures:**
  - If the second input is not the original lead stream, merge results will be empty, mispaired, or will stall waiting for input.
  - `mergeByIndex` is brittle if one side has missing items; `mergeByKey` (e.g., email) is often safer.

#### Node: Calculate Lead Score
- **Type / role:** Code — applies a simple scoring model and appends `score`.
- **Configuration choices (interpreted scoring):**
  - Base score: `50`
  - Company size (employees): +20 (>1000), +10 (>200), else +5
  - Penalize gmail domain: -15 if email ends with `@gmail.com`
  - Industry: +10 if `company.category.industry === 'Software'`
  - Funding: +10 if `company.metrics.raised` exists/truthy
  - Output: `{ ...item, score }`
- **Inputs / outputs:** Input from **Merge Lead & Enrichment**. Output → **IF Qualified?**
- **Version notes:** `typeVersion 2`.
- **Edge cases / failures:**
  - Assumes merged structure includes `company.metrics.employees`, `company.category.industry`, etc. If Clearbit fields differ, score may remain near base value.
  - If merge failed and only lead data exists without `company`, scoring still works (guards exist), but may be less meaningful.

#### Node: IF Qualified?
- **Type / role:** IF — routes leads based on final score threshold.
- **Configuration choices:**
  - Condition: `$json.score >= 70`
  - True branch: qualified
  - False branch: unqualified
- **Inputs / outputs:**
  - Input from **Calculate Lead Score**
  - True → **Create Trello Card - Qualified**
  - False → **Create Trello Card - Unqualified**
- **Version notes:** `typeVersion 2`.
- **Edge cases / failures:**
  - If `score` is missing or non-numeric, qualification logic may misroute.

---

### Block 4 — Storage & Notifications
**Overview:**  
Stores leads in Trello lists depending on outcome and notifies the correct Mattermost channel/team.

**Nodes involved:**
- Create Trello Card - Qualified
- Notify Sales Team
- Create Trello Card - Unqualified
- Create Trello Card - Needs Research
- Notify Ops

#### Node: Create Trello Card - Qualified
- **Type / role:** Trello — creates a card in the Qualified list.
- **Configuration choices:**
  - Card name: `{{$json.name}} (Score: {{$json.score}})`
  - List ID: placeholder `QUALIFIED_LIST_ID`
  - Description includes email, company name fallback, score
- **Inputs / outputs:** Input from **IF Qualified?** (true). Output → **Notify Sales Team**.
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:**
  - Invalid Trello credentials, listId, or missing board permissions.
  - `$json.name` missing → card name may be “ (Score: 85)”.

#### Node: Notify Sales Team
- **Type / role:** Mattermost — posts a message (operation: create).
- **Configuration choices:**
  - Operation set to `create` (in Mattermost node this typically means “create post”).
  - Channel/message fields are not shown in JSON; must be configured in UI (likely placeholders for sales channel).
- **Inputs / outputs:** Input from **Create Trello Card - Qualified**. No downstream connections.
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:**
  - Missing OAuth2 credentials or channel ID.
  - Message template not configured → post may fail validation.

#### Node: Create Trello Card - Unqualified
- **Type / role:** Trello — creates a card in the Unqualified list.
- **Configuration choices:**
  - Card name includes score
  - List ID: placeholder `UNQUALIFIED_LIST_ID`
  - Description includes email and score
- **Inputs / outputs:** Input from **IF Qualified?** (false). No downstream connections.
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:** same as other Trello node (credentials/listId).

#### Node: Create Trello Card - Needs Research
- **Type / role:** Trello — stores leads whose enrichment failed.
- **Configuration choices:**
  - List ID: placeholder `RESEARCH_LIST_ID`
  - Description indicates Clearbit enrichment failed and includes original email.
- **Inputs / outputs:** Input from **Fallback Score**. Output → **Notify Ops**.
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:** same as other Trello node (credentials/listId).

#### Node: Notify Ops
- **Type / role:** Mattermost — posts a message to Ops channel about enrichment failure.
- **Configuration choices:**
  - Operation: `create`
  - Channel/message fields not present in JSON; configure in UI.
- **Inputs / outputs:** Input from **Create Trello Card - Needs Research**. No downstream connections.
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:** same as sales notification (credentials/channelId, missing required fields).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / setup guidance |  |  | ## How it works… (hourly fetch, Clearbit enrichment, scoring, Trello + Mattermost routing) + Setup steps 1–6 |
| Intake & Enrichment Note | Sticky Note | Documentation for intake/enrichment block |  |  | ## Lead Intake & Enrichment… (SplitInBatches, Continue On Fail, IF routing) |
| Scoring Note | Sticky Note | Documentation for scoring block |  |  | ## Scoring & Qualification… (score logic, threshold 70) |
| Storage & Alerts Note | Sticky Note | Documentation for storage/alerts block |  |  | ## Storage & Notifications… (Trello as source of truth, Sales/Ops notifications) |
| Daily Lead Check | Schedule Trigger | Hourly workflow trigger |  | Fetch New Leads |  |
| Fetch New Leads | HTTP Request | Pull new leads from CRM/form API | Daily Lead Check | Parse Leads |  |
| Parse Leads | Code | Convert leads array into individual items | Fetch New Leads | Split Leads |  |
| Split Leads | Split In Batches | Throttle per-lead processing | Parse Leads | Enrich with Clearbit; Merge Lead & Enrichment |  |
| Enrich with Clearbit | HTTP Request | Clearbit enrichment by email | Split Leads | Check Clearbit Response |  |
| Check Clearbit Response | IF | Route by Clearbit HTTP status | Enrich with Clearbit | Merge Lead & Enrichment; Fallback Score |  |
| Merge Lead & Enrichment | Merge | Combine original lead + enrichment | Check Clearbit Response; Split Leads | Calculate Lead Score |  |
| Calculate Lead Score | Code | Compute `score` from enrichment signals | Merge Lead & Enrichment | IF Qualified? |  |
| IF Qualified? | IF | Threshold routing (`score >= 70`) | Calculate Lead Score | Create Trello Card - Qualified; Create Trello Card - Unqualified |  |
| Create Trello Card - Qualified | Trello | Store qualified lead as Trello card | IF Qualified? | Notify Sales Team |  |
| Notify Sales Team | Mattermost | Alert Sales channel | Create Trello Card - Qualified |  |  |
| Create Trello Card - Unqualified | Trello | Store low-score lead in backlog list | IF Qualified? |  |  |
| Fallback Score | Code | Assign minimal score on enrichment failure | Check Clearbit Response | Create Trello Card - Needs Research |  |
| Create Trello Card - Needs Research | Trello | Store failed-enrichment lead for manual research | Fallback Score | Notify Ops |  |
| Notify Ops | Mattermost | Alert Ops channel about enrichment failure | Create Trello Card - Needs Research |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (name as desired, e.g., “Score and route leads with Clearbit, Mattermost and Trello”).  
2. **Add Trigger: “Schedule Trigger”**  
   - Set interval to **every 1 hour**.

3. **Add “HTTP Request” node: Fetch New Leads**  
   - Method: GET  
   - URL: `https://api.yourcrm.com/leads/new`  
   - Timeout: 30000 ms  
   - Authentication: **Predefined Credential Type → HTTP Header Auth**  
   - Create credential (e.g., `crmApi`) with the required header(s) for your CRM (commonly `Authorization: Bearer …` or `X-API-Key: …`).  
   - Connect: **Schedule Trigger → Fetch New Leads**

4. **Add “Code” node: Parse Leads**  
   - Paste logic that maps `items[0].json.leads` into individual items:  
     - Read `const leads = items[0].json.leads || [];`  
     - Return `leads.map(lead => ({ json: lead }));`  
   - Connect: **Fetch New Leads → Parse Leads**

5. **Add “Split In Batches” node: Split Leads**  
   - Set **Batch Size = 1** (recommended to match the intent in notes).  
   - Connect: **Parse Leads → Split Leads**  
   - (Recommended for correct looping) After downstream processing of one lead, add a connection back to **Split Leads** to fetch the next batch until done. The provided JSON does not include this loop-back; add it if you want to process all leads each run.

6. **Add “HTTP Request” node: Enrich with Clearbit**  
   - Method: GET  
   - URL: `https://person.clearbit.com/v2/people/find?email={{ $json.email }}`  
   - Timeout: 20000 ms  
   - Authentication: **Predefined Credential Type → HTTP Header Auth**  
   - Create credential (e.g., `clearbitApi`): set `Authorization` header to `Bearer <CLEARBIT_API_KEY>` (or whatever Clearbit requires for your plan).  
   - Enable **Continue On Fail** (recommended to prevent workflow halting on 404/422/etc.).  
   - (Optional but strongly recommended) Enable **Full Response** so you can reliably check `statusCode`.  
   - Connect: **Split Leads (main output 0) → Enrich with Clearbit**

7. **Add “IF” node: Check Clearbit Response**  
   - Condition: Number  
   - Value1: `{{ $json.statusCode }}`  
   - Operation: “smaller”  
   - Value2: `400`  
   - Connect: **Enrich with Clearbit → Check Clearbit Response**

8. **Add “Merge” node: Merge Lead & Enrichment**  
   - Mode: **Merge by Index**  
   - Connect:
     - **Check Clearbit Response (true) → Merge (Input 1)** (enrichment)
     - Provide the **original lead item** to **Merge (Input 2)**.  
       - Recommended approach: use an additional wiring/structure so the original lead is available (commonly by keeping lead data on the item and attaching enrichment into a field, or using Merge by Key on `email`).  
       - The JSON connects Split Leads output 1 to Merge; verify in your n8n version whether this actually carries the original item. If not, adjust: e.g., use a **Set** node to store original lead in a field, or use HTTP Request “Response” merge strategy.
   - Connect: **Merge Lead & Enrichment → Calculate Lead Score**

9. **Add “Code” node: Calculate Lead Score**  
   - Implement scoring (base 50, modify by employees, gmail penalty, industry, funding).  
   - Output a single item with `score` appended.  
   - Connect: **Merge Lead & Enrichment → Calculate Lead Score**

10. **Add “IF” node: IF Qualified?**  
   - Condition: Number  
   - Value1: `{{ $json.score }}`  
   - Operation: “largerEqual”  
   - Value2: `70`  
   - Connect: **Calculate Lead Score → IF Qualified?**

11. **Add “Trello” node: Create Trello Card - Qualified** (True branch)  
   - Operation: Create Card  
   - Credentials: Trello (API key + token)  
   - List ID: your real qualified list id  
   - Name: `{{ $json.name }} (Score: {{ $json.score }})`  
   - Description: include email/company/score as desired  
   - Connect: **IF Qualified? (true) → Create Trello Card - Qualified**

12. **Add “Mattermost” node: Notify Sales Team**  
   - Credentials: Mattermost OAuth2  
   - Operation: Create Post  
   - Configure **Channel ID** (sales channel) and a message template (include lead name, score, Trello link if available).  
   - Connect: **Create Trello Card - Qualified → Notify Sales Team**

13. **Add “Trello” node: Create Trello Card - Unqualified** (False branch)  
   - List ID: your unqualified list id  
   - Name/description include score/email  
   - Connect: **IF Qualified? (false) → Create Trello Card - Unqualified**

14. **Failure path (Clearbit failed): Add “Code” node: Fallback Score**  
   - Set `score = 10` (or your chosen default).  
   - Connect: **Check Clearbit Response (false) → Fallback Score**

15. **Add “Trello” node: Create Trello Card - Needs Research**  
   - List ID: your research list id  
   - Description indicates enrichment failure  
   - Connect: **Fallback Score → Create Trello Card - Needs Research**

16. **Add “Mattermost” node: Notify Ops**  
   - Configure Ops channel ID and message template (include email + reason).  
   - Connect: **Create Trello Card - Needs Research → Notify Ops**

17. **Replace placeholders**  
   - Trello list IDs: `QUALIFIED_LIST_ID`, `UNQUALIFIED_LIST_ID`, `RESEARCH_LIST_ID`  
   - Mattermost channel IDs in both notification nodes.

18. **Test and activate**
   - Execute workflow manually with sample leads.
   - Verify Clearbit status handling (`statusCode` availability).
   - Confirm batch looping processes all items (add loop-back if required).
   - Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Every hour: fetch leads → enrich via Clearbit → score → route to Trello lists + notify Sales/Ops in Mattermost. | From sticky note “Workflow Overview” |
| Setup prerequisites: CRM API header credentials, Clearbit header credentials, Trello credentials + board/list IDs, Mattermost OAuth2 + channel IDs; replace placeholders. | From sticky note “Workflow Overview” |
| Split-in-batches is intended to prevent Clearbit rate-limit spikes; Clearbit request should “Continue On Fail” to avoid halting. | From sticky note “Lead Intake & Enrichment” |
| Scoring threshold defaults to 70; scoring weights are intended to be easily editable in the code node. | From sticky note “Scoring & Qualification” |
| Trello is used as a single source of truth across all branches; Ops channel is alerted on enrichment failures. | From sticky note “Storage & Notifications” |