Capture, qualify, and route real estate leads with WhatsApp, Typeform, Airtable, Slack, Gmail, and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/capture--qualify--and-route-real-estate-leads-with-whatsapp--typeform--airtable--slack--gmail--and-gpt-4-1-mini-12820


# Capture, qualify, and route real estate leads with WhatsApp, Typeform, Airtable, Slack, Gmail, and GPT-4.1-mini

## 1. Workflow Overview

**Purpose:** Automate end-to-end real estate lead intake, validation, deduplication, qualification, routing/assignment, CRM storage, agent notifications, and weekly reporting.

**Primary use cases**
- Capture leads from **WhatsApp conversations** and **Typeform website forms**
- Use **GPT-4.1-mini** to (a) guide WhatsApp Q&A collection, (b) validate completeness, (c) extract structured fields, and (d) qualify leads
- Prevent double-processing via **in-run deduplication** + **CRM deduplication**
- Assign leads to agents via **round-robin by property category**
- Store new leads + duplicates in separate **Airtable** tables
- Notify assigned agents via **Gmail + Slack**
- Generate a **weekly report** (leads + duplicates in last 7 days) via Gmail

### 1.1 WhatsApp Intake & AI Conversation
Incoming WhatsApp messages (texts only) are handled by an AI agent with memory to collect lead details step-by-step, then the bot replies back on WhatsApp.

### 1.2 AI Completeness Check & Structured Extraction
A second AI step judges if all required fields are complete. If complete, a structured extraction chain produces a strict JSON object validated by a structured output parser.

### 1.3 Lead Normalization & Multi-Source Merge
WhatsApp and Typeform leads are normalized to common field names and tagged with a `Source`. Both sources are merged into a unified pipeline.

### 1.4 Deduplication (In-Run + CRM)
Leads are deduplicated within the same execution by email/phone, then compared against existing Airtable CRM records by email. Non-duplicates continue; duplicates are stored separately.

### 1.5 Routing, Assignment, Qualification, Storage, Notifications
Leads are routed by property type, assigned using a round-robin algorithm per category, qualified by GPT, written to Airtable CRM, then agents are notified via Gmail and Slack.

### 1.6 Weekly Reporting
A weekly schedule fetches CRM records and duplicate-table records, filters to last 7 days, aggregates metrics, formats the report, and emails it via Gmail.

---

## 2. Block-by-Block Analysis

### Block 2.1 — WhatsApp Lead Intake (Trigger → Filter)
**Overview:** Listens for WhatsApp inbound updates and only continues if the message type is text.  
**Nodes involved:** `WhatsApp Trigger`, `WhatsApp Texts Only`, `No Operation, do nothing`  
**Sticky note (applies):** “## WhatsApp Lead intake”

#### Node: WhatsApp Trigger
- **Type / role:** WhatsApp Trigger; entry point for inbound WhatsApp events.
- **Config choices:**
  - Subscribes to `updates: ["messages"]`.
  - Uses WhatsApp OAuth credential (“WhatsApp OAuth account”).
- **Key data used:** `messages[0].from`, `messages[0].type`, `messages[0].text.body`.
- **Outputs:** To `WhatsApp Texts Only`.
- **Failure/edge cases:**
  - Webhook verification/auth issues with Meta/WhatsApp.
  - Non-message updates or unexpected payload shape.

#### Node: WhatsApp Texts Only
- **Type / role:** Switch; filters only `messages[0].type === "text"`.
- **Outputs:**
  - **Text** → `Lead Collection Agent`
  - **fallback (“extra”)** → `No Operation, do nothing`
- **Failure/edge cases:** If `messages[0]` missing, expression can fail unless n8n tolerates undefined; consider guarding.

#### Node: No Operation, do nothing
- **Type / role:** NoOp; sink for non-text WhatsApp content (images, audio, etc.).
- **Outputs:** None (ends that branch).

---

### Block 2.2 — WhatsApp AI Agent: Collection + Reply
**Overview:** An AI agent converses with the lead on WhatsApp, asking questions in order, storing context in memory, and sending the agent’s output back to the lead.  
**Nodes involved:** `Lead Collection Agent`, `Simple Memory`, `OpenAI Chat Model`, `Send message`  
**Sticky notes (apply):**
- “### WhatsApp AI Agent - Lead Collection…”
- “### WhatsApp AI Agent - Info Completeness Check…” (context overlaps because the conversation output feeds the checker next)

#### Node: Simple Memory
- **Type / role:** LangChain memory buffer (windowed); preserves conversation context per WhatsApp sender.
- **Config choices:**
  - `sessionKey` = WhatsApp sender: `{{ $("WhatsApp Trigger").item.json.messages[0].from }}`
  - `contextWindowLength: 20` (keeps last 20 interactions)
- **Connections:** Memory is linked into `Lead Collection Agent` via `ai_memory`.
- **Edge cases:** If sender ID format changes or missing, memory sessions may collide or fail.

#### Node: OpenAI Chat Model
- **Type / role:** LLM provider node (OpenAI chat).
- **Config choices:** Model `gpt-4.1-mini`.
- **Connections:** Feeds as `ai_languageModel` into:
  - `Lead Collection Agent`
  - `Info Completeness Check`
  - `Extract Lead Info`
- **Failure/edge cases:** OpenAI auth, quota, model availability, rate limits, or latency.

#### Node: Lead Collection Agent
- **Type / role:** LangChain Agent; orchestrates question-by-question collection.
- **Config choices (interpreted):**
  - Uses the WhatsApp inbound text: `{{ $json.messages[0].text.body }}`
  - Strong system message defines:
    - Exact question order
    - Conditional mortgage question if financing
    - A `collectionComplete` flag logic requirement
    - Required final summary message format when complete
- **Inputs:** From `WhatsApp Texts Only (Text)`; memory + OpenAI model attached.
- **Outputs:** To `Send message` (WhatsApp response).
- **Edge cases:**
  - User provides partial/ambiguous answers (“Other”) or refuses info.
  - Agent may not reliably maintain `collectionComplete` as a machine-readable field (it’s in natural language output). This is mitigated by the next “Info Completeness Check” step.

#### Node: Send message
- **Type / role:** WhatsApp Cloud “send” action; sends agent output back to lead.
- **Config choices:**
  - Recipient: `{{ $('WhatsApp Trigger').item.json.messages[0].from }}`
  - Text: `{{ $json.output }}` (agent output)
  - `phoneNumberId`: set to a specific WhatsApp Business Phone Number ID
- **Outputs:** To `Info Completeness Check`.
- **Failure/edge cases:** WhatsApp token expiration, wrong phoneNumberId, recipient format issues, rate limiting.

---

### Block 2.3 — AI Completeness Gate + Structured Extraction
**Overview:** Uses an LLM to decide if all required fields are present. If yes, extracts structured fields into validated JSON.  
**Nodes involved:** `Info Completeness Check`, `Are All Info Complete?`, `Extract Lead Info`, `Structured Output Parser`, `No Operation, do nothing1`  
**Sticky notes (apply):**
- “### WhatsApp AI Agent - Info Completeness Check…”
- “### WhatsApp AI Agent - Extract Lead Info…”

#### Node: Info Completeness Check
- **Type / role:** LLM chain; outputs **only** “Yes” or “No”.
- **Config choices:**
  - Input: `{{ $('Lead Collection Agent').item.json.output }}`
  - Prompt defines required fields + conditional mortgage requirement.
- **Outputs:** To `Are All Info Complete?`.
- **Edge cases:** LLM may output unexpected text; downstream IF expects exact “Yes”.

#### Node: Are All Info Complete?
- **Type / role:** IF gate on LLM result.
- **Condition:** `{{ $json.text }} == "Yes"`
- **Outputs:**
  - **true** → `Extract Lead Info`
  - **false** → `No Operation, do nothing1`
- **Edge cases:** If completeness check returns lowercase or whitespace; consider normalizing.

#### Node: Extract Lead Info
- **Type / role:** LLM chain with structured output parsing; converts conversation into consistent CRM fields.
- **Config choices:**
  - Input: `{{ $('Lead Collection Agent').item.json.output }}`
  - Has output parser enabled and strict instructions: return valid JSON only.
- **Connections:** Uses `OpenAI Chat Model` as language model; `Structured Output Parser` as `ai_outputParser`.
- **Edge cases:** Parser failures if JSON invalid or schema mismatch (common when LLM adds commentary).

#### Node: Structured Output Parser
- **Type / role:** Enforces schema (manual JSON schema).
- **Schema highlights:**
  - Required: First name, Phone number, Email, budget range, purchase method, property type, location, timeline, agent status, and `collectionComplete` boolean.
  - Mortgage question is present but not “required” in schema (conditional is in prompts, not schema).
- **Edge cases:** If mortgage field required conditionally, schema won’t enforce it; completeness check does.

#### Node: No Operation, do nothing1
- **Type / role:** Ends flow when info is incomplete (no follow-up automation beyond WhatsApp conversation loop).

---

### Block 2.4 — Data Normalization & Multi-Source Unification
**Overview:** Converts WhatsApp-extracted JSON into flat fields matching the Typeform lead shape, tags each source, then merges into one lead stream.  
**Nodes involved:** `Normalize WhatsApp Leads`, `Typeform Trigger`, `Normalize Form Leads`, `Combine All Lead Sources`  
**Sticky notes (apply):**
- “## Website forms Lead intake”
- “## Data Normalization…”

#### Node: Typeform Trigger
- **Type / role:** Typeform trigger; entry point for website form submissions.
- **Config choices:** `formId: QPCD9YG4`
- **Outputs:** To `Normalize Form Leads`.
- **Edge cases:** Typeform webhook disabled, credential revoked, form changes causing field name drift.

#### Node: Normalize Form Leads
- **Type / role:** Set; ensures `Source = "Website Form"` and keeps other fields.
- **Config choices:** `includeOtherFields: true`.
- **Outputs:** To `Combine All Lead Sources` (input index 1).

#### Node: Normalize WhatsApp Leads
- **Type / role:** Set; maps `output` object fields from the extractor into top-level fields matching CRM schema, and sets `Source = "WhatsApp Form"`.
- **Key mappings:** First name, Last name, Phone number, Email, budget range, purchase method, mortgage status, property type, location, timeline, agent status.
- **Outputs:** To `Combine All Lead Sources` (input index 0).
- **Edge cases:** If extractor output keys differ (capitalization / punctuation), mappings will produce empty fields.

#### Node: Combine All Lead Sources
- **Type / role:** Merge; combines both sources into one stream.
- **Config:** Default merge behavior (no explicit mode set in parameters).
- **Inputs:** From WhatsApp normalization and Typeform normalization.
- **Outputs:** To `Remove Duplicate Leads`.
- **Edge cases:** Merge node behavior depends on arrivals; ensure it’s configured as intended for “multiple sources” (often “Append” is desired—verify in UI).

---

### Block 2.5 — Deduplication (In-Run + CRM)
**Overview:** Removes duplicates within the execution and against Airtable CRM records; duplicates are stored in a dedicated Airtable table.  
**Nodes involved:** `Remove Duplicate Leads`, `Search records`, `Check for Duplicates in CRM`, `Create Duplicate record`  
**Sticky notes (apply):**
- “## Deduplication Logic…”
- “## Store Duplicates in CRM”

#### Node: Remove Duplicate Leads
- **Type / role:** Remove Duplicates; prevents re-processing duplicates within the same run.
- **Config choices:** Compare selected fields: `Email, Phone number`.
- **Outputs:** 
  - Main output → `Check for Duplicates in CRM` (index 0) and `Search records` (index 0)
- **Edge cases:** If Email or Phone missing, dedupe can behave unexpectedly; consider fallback keys.

#### Node: Search records
- **Type / role:** Airtable search; fetches existing CRM leads.
- **Config choices:**
  - Base: “N8n” (appBnug5XIRAWl5sK)
  - Table: “Real Estate Qualifier” (tblaHgYnpXzZl0zRH)
  - Operation: `search` (but no filter formula shown—this may return many records depending on Airtable node defaults/UI settings)
- **Outputs:** To `Check for Duplicates in CRM` (index 1).
- **Edge cases:**
  - If unfiltered, may pull large datasets (performance/timeouts).
  - Airtable API pagination limits.
  - Token permissions.

#### Node: Check for Duplicates in CRM
- **Type / role:** Compare Datasets; compares incoming leads (input 0) against Airtable records (input 1) by Email.
- **Config choices:** Merge by `Email` (field1 Email, field2 Email).
- **Outputs (as wired):**
  - Output 0 → `Route by Location/Property Type1` (non-duplicates/new leads in intended design)
  - Output 1 and Output 2 → `Create Duplicate record` (duplicates/overlaps; both connected)
- **Edge cases:**
  - Email casing/whitespace differences cause false negatives; normalize emails before compare.
  - Multiple matches in CRM can create ambiguous results.
  - Dual connections to `Create Duplicate record` can create double inserts if both outputs emit for the same item (verify node semantics).

#### Node: Create Duplicate record
- **Type / role:** Airtable create; stores duplicates in a separate table.
- **Config choices:**
  - Table: “Real Estate Qualifier Duplicate” (tblrmNx8MIvJvUHNz)
  - Writes mapped fields from `Combine All Lead Sources` item.
- **Edge cases:** Missing fields, Airtable column type mismatches, duplicate insertion risk if called twice.

---

### Block 2.6 — Routing & Round-Robin Agent Assignment
**Overview:** Routes by property type and assigns an agent using category-based round-robin stored in node static data.  
**Nodes involved:** `Route by Location/Property Type1`, `Round-Robin Agent Assignment1`, `Aggregate`  
**Sticky note (applies):** “## Agent Assignment (Round-Robin)…”

#### Node: Route by Location/Property Type1
- **Type / role:** Switch; routes by normalized property type string.
- **Rules:** Compares `{{ $json['What type of property are you interested in?'].toLowerCase() }}` to:
  - `single-family`
  - `multi-family`
  - `condo`
  - `investment property`
- **Outputs:** Each route output goes to `Round-Robin Agent Assignment1`.
- **Fallback:** `extra` (not connected).
- **Edge cases:** If value differs (e.g., “Investment Property” vs “investment property” is handled by lowercase; but “Investment-Property” not).

#### Node: Round-Robin Agent Assignment1
- **Type / role:** Code; assigns agent email based on property type category and rotates evenly.
- **Config choices:**
  - `agentPools` keyed by category, e.g. `single-family`, `multi-family`, etc.
  - Category derivation:
    - raw: `item["What type of property are you interested in?"]`
    - normalized: lowercased and whitespace → hyphens
  - Uses `$getWorkflowStaticData('node')` to store `agentIndexes[category]`.
  - Adds metadata: `lead_status = "New"`, `assigned_agent`, `assignment_category`, `assignment_timestamp`.
- **Outputs:** To `Aggregate`.
- **Edge cases:**
  - **Key mismatch bug risk:** pool keys use both `single-family` and `investment property` (with space), while normalization converts spaces to `-` producing `investment-property`. That will miss the intended pool and fall back to `default-unassigned`. Fix by aligning keys (e.g., `investment-property`).
  - Static data is per workflow and persists; cloning workflow resets it; running on multiple instances can diverge.

#### Node: Aggregate
- **Type / role:** Aggregate node; set to “aggregateAllItemData”.
- **Role here:** Collapses/collects before qualification step (likely to ensure a single payload with a `data` array for the LLM chain).
- **Outputs:** To `Basic LLM Chain`.
- **Edge cases:** Unexpected output shape; confirm that the next LLM expects `$json.data`.

---

### Block 2.7 — AI Qualification → CRM Storage → Notifications
**Overview:** GPT qualifies leads (“Yes/No”), writes them to Airtable CRM with assignment metadata, then notifies the assigned agent via email and Slack.  
**Nodes involved:** `Basic LLM Chain`, `OpenAI Chat Model2`, `Create A Record`, `Send a message`, `Notify Agent via Slack`  
**Sticky notes (apply):**
- “## Lead Qualification…”
- “## CRM Storage (Airtable)…”
- “## Notifications & Alerts…”

#### Node: OpenAI Chat Model2
- **Type / role:** OpenAI chat model for qualification chain.
- **Config:** `gpt-4.1-mini`.
- **Connections:** `ai_languageModel` → `Basic LLM Chain`.

#### Node: Basic LLM Chain
- **Type / role:** LLM chain to qualify the lead.
- **Config choices:**
  - System prompt defines qualification criteria (budget, TX/FL/AZ, property type, timeline < 6 months, financing status, not working with agent).
  - Strict output: “Yes / No”.
  - Input: `The responses from the form: {{ JSON.stringify($json.data) }}`
- **Outputs:** To `Create A Record`.
- **Edge cases:** If `Aggregate` doesn’t produce `data`, qualification input will be incorrect and output unreliable.

#### Node: Create A Record
- **Type / role:** Airtable create; stores new lead in main CRM table with qualification + assignment.
- **Config choices:**
  - Table: “Real Estate Qualifier”
  - Writes `Qualified` = `{{ $json.text }}` (from LLM chain output)
  - Writes assignment fields from `Round-Robin Agent Assignment1` (Assigned Agent, Assigned Time, Lead Status)
- **Outputs:** To `Send a message` (Gmail) and `Notify Agent via Slack`.
- **Edge cases:** Airtable field names must match exactly; option fields require allowed values (“Yes/No”, statuses).

#### Node: Send a message (Gmail)
- **Type / role:** Gmail send; emails the assigned agent a formatted HTML lead summary.
- **Config choices:**
  - `sendTo`: `{{ $('Round-Robin Agent Assignment1').item.json.assigned_agent }}`
  - HTML includes recommended action based on `Qualified`.
- **Edge cases:** Gmail sending limits, OAuth token expiration, missing assigned_agent.

#### Node: Notify Agent via Slack
- **Type / role:** Slack post message to a channel.
- **Config choices:**
  - Posts a formatted lead summary to selected channel ID `C09MZKZS5QE`.
  - “includeLinkToWorkflow” disabled.
- **Edge cases:** Slack token scopes, channel permissions, rate limits.

---

### Block 2.8 — Weekly Reporting & Analytics
**Overview:** Every week, fetch last-7-days CRM leads and duplicates, aggregate metrics, format them, and email a weekly HTML report.  
**Nodes involved:** `Weekly Report Schedule`, `Get Records`, `Weekly Fetch`, `Aggregate Report Data`, `Get Duplicates Records`, `7-Day Duplicate Count`, `Combine both duplicate and non-duplicate records`, `Format Weekly Report`, `Send Weekly Report`  
**Sticky note (applies):** “## Reporting & Analytics…”

#### Node: Weekly Report Schedule
- **Type / role:** Schedule Trigger; weekly at day 1 (Monday) 09:00.
- **Outputs:** Starts two parallel fetches: `Get Records` and `Get Duplicates Records`.
- **Edge cases:** Timezone depends on n8n instance settings.

#### Node: Get Records
- **Type / role:** Airtable search; fetches CRM table records.
- **Output:** To `Weekly Fetch`.
- **Edge cases:** Like earlier Airtable search—ensure it’s not unintentionally unfiltered/heavy.

#### Node: Weekly Fetch
- **Type / role:** Code; filters fetched CRM records to those created within last 7 days using `createdTime`.
- **Output:** To `Aggregate Report Data`.
- **Edge cases:** Missing/invalid `createdTime` in items.

#### Node: Aggregate Report Data
- **Type / role:** Aggregate; builds arrays for counts and breakdowns.
- **Config choices:**
  - Aggregates:
    - `id` → `total_leads`
    - `Source` → `leads_by_source`
    - `Lead Status` → `leads_by_status`
    - `Qualified` → `leads_qualified?`
- **Output:** To `Combine both duplicate and non-duplicate records` (index 0).
- **Edge cases:** Aggregated fields become arrays; downstream uses `.length` and `.filter()` accordingly.

#### Node: Get Duplicates Records
- **Type / role:** Airtable search; fetches duplicates table records.
- **Config:** Has `filterByFormula: "="` (likely invalid/incomplete; may return 0 or error depending on Airtable API handling).
- **Output:** To `7-Day Duplicate Count`.
- **Edge cases:** Invalid formula can break reporting; should be blank or a valid formula.

#### Node: 7-Day Duplicate Count
- **Type / role:** Code; counts duplicate records created in last 7 days.
- **Output:** To `Combine both duplicate and non-duplicate records` (index 1).

#### Node: Combine both duplicate and non-duplicate records
- **Type / role:** Merge (combine by position).
- **Config choices:** `mode: combine`, `combineBy: combineByPosition`.
- **Role:** Combines the aggregated lead metrics item with the duplicate count item into one object for formatting.
- **Edge cases:** If one branch returns no items, combine-by-position can fail or output empty.

#### Node: Format Weekly Report
- **Type / role:** Set; computes final report fields.
- **Key expressions:**
  - `report_period`: `{{ $now.minus({days: 7}).toFormat('MMM dd') + ' - ' + $now.toFormat('MMM dd, yyyy') }}`
  - `total_leads`: `{{ $json.total_leads.length }}`
  - “Website Forms” count: filters `leads_by_source` equals `Website Form`
  - “Facebook Ads” field actually counts `WhatsApp Form` (naming mismatch)
  - Contact metrics based on `Lead Status`
  - `contact_rate`: guarded division; returns empty string if non-finite
- **Output:** To `Send Weekly Report`.
- **Edge cases:** If arrays missing, `.filter` fails; consider defaulting arrays to `[]`.

#### Node: Send Weekly Report
- **Type / role:** Gmail send; emails HTML report.
- **Config choices:**
  - To: `user@example.com` (placeholder)
  - Subject: `Weekly Lead Report: {period}`
  - HTML report includes totals, duplicates, contact rate, and source/status tables.
- **Edge cases:** Gmail send limits; placeholders must be updated.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Canvas documentation / setup guidance |  |  | # Automate real estate lead capture, smart routing, and reporting using n8n… Contact: buzanalytics@gmail.com |
| WhatsApp Trigger | WhatsApp Trigger | Entry: receive WhatsApp messages |  | WhatsApp Texts Only | ## WhatsApp Lead intake |
| WhatsApp Texts Only | Switch | Filter non-text WhatsApp messages | WhatsApp Trigger | Lead Collection Agent; No Operation, do nothing | ## WhatsApp Lead intake |
| No Operation, do nothing | NoOp | Drop non-text WhatsApp messages | WhatsApp Texts Only |  | ## WhatsApp Lead intake |
| Simple Memory | LangChain Memory (Buffer Window) | Persist WhatsApp chat context per sender |  | Lead Collection Agent (ai_memory) | ### WhatsApp AI Agent - Lead Collection… |
| OpenAI Chat Model | OpenAI Chat Model | LLM for WhatsApp collection, completeness, extraction |  | Lead Collection Agent; Info Completeness Check; Extract Lead Info | ### WhatsApp AI Agent - Lead Collection… |
| Lead Collection Agent | LangChain Agent | Ask questions and collect lead info | WhatsApp Texts Only | Send message | ### WhatsApp AI Agent - Lead Collection… |
| Send message | WhatsApp | Reply to lead on WhatsApp | Lead Collection Agent | Info Completeness Check | ### WhatsApp AI Agent - Lead Collection… |
| Info Completeness Check | LangChain LLM Chain | Determine if required info is complete (Yes/No) | Send message | Are All Info Complete? | ### WhatsApp AI Agent - Info Completeness Check… |
| Are All Info Complete? | IF | Gate: proceed only if complete | Info Completeness Check | Extract Lead Info; No Operation, do nothing1 | ### WhatsApp AI Agent - Info Completeness Check… |
| No Operation, do nothing1 | NoOp | End branch if incomplete | Are All Info Complete? |  | ### WhatsApp AI Agent - Info Completeness Check… |
| Structured Output Parser | Structured Output Parser | Enforce JSON schema for extracted fields |  | Extract Lead Info (ai_outputParser) | ### WhatsApp AI Agent - Extract Lead Info… |
| Extract Lead Info | LangChain LLM Chain | Extract structured lead fields to JSON | Are All Info Complete? | Normalize WhatsApp Leads | ### WhatsApp AI Agent - Extract Lead Info… |
| Normalize WhatsApp Leads | Set | Flatten extracted JSON + tag Source | Extract Lead Info | Combine All Lead Sources | ## Data Normalization… |
| Typeform Trigger | Typeform Trigger | Entry: website form submission |  | Normalize Form Leads | ## Website forms Lead intake |
| Normalize Form Leads | Set | Tag Source=Website Form | Typeform Trigger | Combine All Lead Sources | ## Data Normalization… |
| Combine All Lead Sources | Merge | Unify WhatsApp + Typeform lead stream | Normalize WhatsApp Leads; Normalize Form Leads | Remove Duplicate Leads |  |
| Remove Duplicate Leads | Remove Duplicates | In-run dedup by Email/Phone | Combine All Lead Sources | Check for Duplicates in CRM; Search records | ## Deduplication Logic… |
| Search records | Airtable (search) | Pull CRM records for dedupe compare | Remove Duplicate Leads | Check for Duplicates in CRM | ## Deduplication Logic… |
| Check for Duplicates in CRM | Compare Datasets | Compare incoming leads vs CRM by Email | Remove Duplicate Leads; Search records | Route by Location/Property Type1; Create Duplicate record | ## Deduplication Logic… |
| Create Duplicate record | Airtable (create) | Store duplicate leads | Check for Duplicates in CRM |  | ## Store Duplicates in CRM |
| Route by Location/Property Type1 | Switch | Route by property type | Check for Duplicates in CRM | Round-Robin Agent Assignment1 | ## Agent Assignment (Round-Robin)… |
| Round-Robin Agent Assignment1 | Code | Assign agent via category round-robin | Route by Location/Property Type1 | Aggregate | ## Agent Assignment (Round-Robin)… |
| Aggregate | Aggregate | Prepare payload for qualification | Round-Robin Agent Assignment1 | Basic LLM Chain | ## Lead Qualification… |
| OpenAI Chat Model2 | OpenAI Chat Model | LLM for qualification |  | Basic LLM Chain | ## Lead Qualification… |
| Basic LLM Chain | LangChain LLM Chain | Qualify lead (Yes/No) | Aggregate | Create A Record | ## Lead Qualification… |
| Create A Record | Airtable (create) | Store new lead in CRM | Basic LLM Chain | Send a message; Notify Agent via Slack | ## CRM Storage (Airtable)… |
| Send a message | Gmail | Email assigned agent lead details | Create A Record |  | ## Notifications & Alerts… |
| Notify Agent via Slack | Slack | Post lead alert to Slack channel | Create A Record |  | ## Notifications & Alerts… |
| Weekly Report Schedule | Schedule Trigger | Weekly reporting trigger |  | Get Records; Get Duplicates Records | ## Reporting & Analytics… |
| Get Records | Airtable (search) | Fetch CRM leads for reporting | Weekly Report Schedule | Weekly Fetch | ## Reporting & Analytics… |
| Weekly Fetch | Code | Filter CRM leads to last 7 days | Get Records | Aggregate Report Data | ## Reporting & Analytics… |
| Aggregate Report Data | Aggregate | Build arrays for counts/breakdowns | Weekly Fetch | Combine both duplicate and non-duplicate records | ## Reporting & Analytics… |
| Get Duplicates Records | Airtable (search) | Fetch duplicate records for reporting | Weekly Report Schedule | 7-Day Duplicate Count | ## Reporting & Analytics… |
| 7-Day Duplicate Count | Code | Count duplicates in last 7 days | Get Duplicates Records | Combine both duplicate and non-duplicate records | ## Reporting & Analytics… |
| Combine both duplicate and non-duplicate records | Merge (combine) | Merge lead metrics + duplicate count | Aggregate Report Data; 7-Day Duplicate Count | Format Weekly Report | ## Reporting & Analytics… |
| Format Weekly Report | Set | Compute final report fields | Combine both duplicate and non-duplicate records | Send Weekly Report | ## Reporting & Analytics… |
| Send Weekly Report | Gmail | Email weekly HTML report | Format Weekly Report |  | ## Reporting & Analytics… |
| Sticky Note2 | Sticky Note | Documentation: dedupe logic |  |  | ## Deduplication Logic… |
| Sticky Note4 | Sticky Note | Documentation: duplicate storage |  |  | ## Store Duplicates in CRM |
| Sticky Note5 | Sticky Note | Documentation: agent assignment |  |  | ## Agent Assignment (Round-Robin)… |
| Sticky Note6 | Sticky Note | Documentation: qualification |  |  | ## Lead Qualification… |
| Sticky Note7 | Sticky Note | Documentation: CRM storage |  |  | ## CRM Storage (Airtable)… |
| Sticky Note8 | Sticky Note | Documentation: notifications |  |  | ## Notifications & Alerts… |
| Sticky Note9 | Sticky Note | Documentation: reporting |  |  | ## Reporting & Analytics… |
| Sticky Note10 | Sticky Note | Documentation: normalization |  |  | ## Data Normalization… |
| Sticky Note11 | Sticky Note | Documentation: Typeform intake |  |  | ## Website forms Lead intake |
| Sticky Note12 | Sticky Note | Documentation: WhatsApp intake |  |  | ## WhatsApp Lead intake |
| Sticky Note13 | Sticky Note | Documentation: WA AI collection |  |  | ### WhatsApp AI Agent - Lead Collection… |
| Sticky Note14 | Sticky Note | Documentation: WA completeness |  |  | ### WhatsApp AI Agent - Info Completeness Check… |
| Sticky Note15 | Sticky Note | Documentation: WA extraction |  |  | ### WhatsApp AI Agent - Extract Lead Info… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name: “Automate real estate lead capture, smart routing, and reporting using n8n”

2) **Add credentials (required)**
   - WhatsApp Trigger OAuth credential (Meta webhook/trigger integration)
   - WhatsApp Cloud API credential (Access Token) for sending messages
   - OpenAI API credential (for `gpt-4.1-mini`)
   - Airtable Personal Access Token credential (read/write to both tables)
   - Gmail OAuth2 credential (send emails)
   - Slack API credential (post messages)

3) **WhatsApp intake nodes**
   1. Add **WhatsApp Trigger**
      - Updates: `messages`
   2. Add **Switch** “WhatsApp Texts Only”
      - Rule: `{{ $json.messages[0].type }}` equals `text`
      - Text output → next step; fallback → NoOp
   3. Add **NoOp** “No Operation, do nothing” and connect fallback to it

4) **WhatsApp AI collection**
   1. Add **OpenAI Chat Model** node
      - Model: `gpt-4.1-mini`
   2. Add **Simple Memory** (Buffer Window)
      - Session ID type: custom
      - Session key: `{{ $("WhatsApp Trigger").item.json.messages[0].from }}`
      - Window length: 20
   3. Add **AI Agent** “Lead Collection Agent”
      - Input text: `{{ $json.messages[0].text.body }}`
      - System message: paste the question-order + conditional logic + summary format
      - Connect:
        - Switch(Text) → Agent
        - OpenAI Chat Model → Agent (ai_languageModel)
        - Simple Memory → Agent (ai_memory)
   4. Add **WhatsApp** node “Send message”
      - Operation: Send
      - Phone Number ID: your WhatsApp Business phone number id
      - Recipient: `{{ $('WhatsApp Trigger').item.json.messages[0].from }}`
      - Text body: `{{ $json.output }}`
      - Connect Agent → WhatsApp Send

5) **Completeness gate**
   1. Add **LLM Chain** “Info Completeness Check”
      - Input text: `{{ $('Lead Collection Agent').item.json.output }}`
      - Prompt: outputs only Yes/No, with conditional mortgage rule
      - Connect OpenAI Chat Model → this chain (ai_languageModel)
      - Connect WhatsApp Send → this chain
   2. Add **IF** “Are All Info Complete?”
      - Condition: `{{ $json.text }}` equals `Yes`
      - Connect completeness chain → IF
   3. Add **NoOp** “No Operation, do nothing1”
      - Connect IF(false) → NoOp

6) **Structured extraction**
   1. Add **Structured Output Parser**
      - Schema type: manual
      - Paste the provided JSON schema (fields like “First name”, “Phone number”, etc.)
   2. Add **LLM Chain** “Extract Lead Info”
      - Input: `{{ $('Lead Collection Agent').item.json.output }}`
      - Enable output parser
      - Use strict JSON-only instructions
      - Connect:
        - IF(true) → Extract Lead Info
        - OpenAI Chat Model → Extract Lead Info (ai_languageModel)
        - Structured Output Parser → Extract Lead Info (ai_outputParser)

7) **Normalization for WhatsApp leads**
   - Add **Set** “Normalize WhatsApp Leads”
     - Map `$json.output.<field>` to top-level fields:
       - First name, Last name, Phone number, Email, budget, purchase method, mortgage status, property type, location, timeline, agent status
     - Set `Source = "WhatsApp Form"`
   - Connect Extract Lead Info → Normalize WhatsApp Leads

8) **Typeform intake + normalization**
   1. Add **Typeform Trigger**
      - Form ID: `QPCD9YG4` (replace with yours)
   2. Add **Set** “Normalize Form Leads”
      - `includeOtherFields = true`
      - Set `Source = "Website Form"`
   3. Connect Typeform Trigger → Normalize Form Leads

9) **Combine lead sources**
   - Add **Merge** “Combine All Lead Sources”
   - Connect:
     - Normalize WhatsApp Leads → Merge input 0
     - Normalize Form Leads → Merge input 1
   - Ensure merge mode in UI matches your intention (commonly “Append” for multi-source streams).

10) **In-run deduplication**
   - Add **Remove Duplicates** “Remove Duplicate Leads”
     - Compare: selected fields
     - Fields: `Email, Phone number`
   - Connect Merge → Remove Duplicates

11) **CRM dedupe compare**
   1. Add **Airtable** “Search records”
      - Operation: Search
      - Base: your base
      - Table: main CRM table
      - Configure a filter if possible to avoid fetching entire table (recommended).
   2. Add **Compare Datasets** “Check for Duplicates in CRM”
      - Merge by fields: Email (incoming vs CRM)
   3. Connect:
      - Remove Duplicates → Compare (input 0)
      - Search records → Compare (input 1)
      - Remove Duplicates → Search records (so both run per batch as designed)

12) **Store duplicates**
   - Add **Airtable Create** “Create Duplicate record”
     - Table: duplicates table
     - Map fields from the unified lead item (Email, names, phone, property type, etc.)
   - Connect compare “duplicate” output(s) → Create Duplicate record  
   - Verify outputs to avoid double-create.

13) **Routing + round-robin assignment**
   1. Add **Switch** “Route by Location/Property Type1”
      - Compare lowercased property type to `single-family`, `multi-family`, `condo`, `investment property`
   2. Add **Code** “Round-Robin Agent Assignment1”
      - Paste the round-robin code
      - Update agent email pools
      - Important: make pool keys match your normalization (`investment-property` vs `investment property`)
   3. Connect:
      - Compare “new leads” output → Switch
      - Each Switch route → Code node

14) **Qualification**
   1. Add **Aggregate** (aggregate all item data)
      - Operation: aggregateAllItemData
   2. Add **OpenAI Chat Model** (second instance) or reuse; set model `gpt-4.1-mini`
   3. Add **LLM Chain** “Basic LLM Chain”
      - Prompt with qualification criteria
      - Input includes `{{ JSON.stringify($json.data) }}`
   4. Connect: Code → Aggregate → LLM Chain, and attach OpenAI model to LLM chain.

15) **CRM storage**
   - Add **Airtable Create** “Create A Record”
     - Table: main CRM
     - Map:
       - Lead fields from assignment node
       - `Qualified = {{ $json.text }}` from qualification chain
       - Assignment metadata fields
   - Connect LLM Chain → Create A Record

16) **Notifications**
   1. Add **Gmail** “Send a message”
      - To: `{{ $('Round-Robin Agent Assignment1').item.json.assigned_agent }}`
      - Subject/body: use the provided HTML template; adjust field references as needed
   2. Add **Slack** “Notify Agent via Slack”
      - Channel: choose your channel
      - Message template: provided
   3. Connect Create A Record → Gmail and Create A Record → Slack

17) **Weekly reporting**
   1. Add **Schedule Trigger** “Weekly Report Schedule”
      - Weekly: Monday 09:00 (or desired)
   2. Add **Airtable Search** “Get Records” (CRM)
   3. Add **Code** “Weekly Fetch” (filter last 7 days by `createdTime`)
   4. Add **Aggregate** “Aggregate Report Data” (arrays for id/source/status/qualified)
   5. Add **Airtable Search** “Get Duplicates Records” (duplicates table)
      - Use a valid `filterByFormula` (or none); remove the literal `"="` unless intended and valid.
   6. Add **Code** “7-Day Duplicate Count”
   7. Add **Merge** “Combine both duplicate and non-duplicate records”
      - Mode: Combine by position
   8. Add **Set** “Format Weekly Report” (expressions for totals and rates)
   9. Add **Gmail** “Send Weekly Report”
      - Update recipient email(s)
   10. Connect schedule to both Airtable searches; then follow the chain as in the JSON.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Import the .json file into your n8n instance… Connect credentials… Update Airtable base/table… Customize agent pools/routing… Click Publish…” | Sticky note: “Automate real estate lead capture, smart routing, and reporting using n8n” |
| “Optional: extend to Facebook and Google Ads; weekly reporting can be extracted into a standalone workflow.” | Same sticky note |
| “For Enquiries: buzanalytics@gmail.com” | Same sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.