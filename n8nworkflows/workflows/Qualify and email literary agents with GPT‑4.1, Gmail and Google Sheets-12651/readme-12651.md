Qualify and email literary agents with GPT‑4.1, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/qualify-and-email-literary-agents-with-gpt-4-1--gmail-and-google-sheets-12651


# Qualify and email literary agents with GPT‑4.1, Gmail and Google Sheets

## 1. Workflow Overview

**Title:** Qualify and email literary agents with GPT‑4.1, Gmail and Google Sheets

**Purpose:**  
This workflow automates the collection of literary-agent leads from multiple data sources, uses GPT‑4.1-mini (via n8n LangChain nodes) to (1) decide whether an agent should be emailed, (2) research the agent with citations for safe personalization, (3) draft a personalized email opener + subject, then sends the email via Gmail and updates submission status in Google Sheets. It also contains an optional chat-driven analysis branch that generates a QuickChart.

**Target use cases:**
- Building an outreach pipeline for a nonfiction book pitch using public agent data
- Qualification of leads (email-eligible vs not) and tracking outreach state in a spreadsheet
- Agentic research + personalization while preserving verifiability and safety constraints

### 1.1 Scheduled ingestion (multi-source lead intake)
Runs daily on schedule; pulls lead candidates from BigQuery, Azure Blob CSV, AWS S3 CSV, and Google Sheets (and includes a disabled HTTP lead source).

### 1.2 AI-driven lead discovery (optional/parallel)
An AI “data collection” prompt targets a public directory page and instructs the LLM to extract a small set of email-eligible agents (strict rules).

### 1.3 Eligibility decision (“Marketing” qualification)
Given each agent record, an AI agent determines whether email should be sent (based on flags and email presence). Output is normalized and routed.

### 1.4 Research enrichment (“Research Team”)
For email-eligible leads, a research agent builds a source-backed brief: agency, interests, notable works, and safe personalization angles with citations.

### 1.5 Message personalization + send (“Sales Team”)
A sales agent drafts a short personalized opener and a subject line in strict JSON. Gmail sends an HTML email, then Google Sheets is updated to mark submission complete and timestamp it.

### 1.6 Chat-triggered analysis & charting (auxiliary)
A chat trigger starts a lightweight analysis branch that (via an agent) normalizes output and generates a QuickChart. (Some BI/GA nodes are present but disabled.)

---

## 2. Block-by-Block Analysis

### Block A — Scheduling & multi-source data ingestion
**Overview:** Triggers the workflow on a schedule and fetches lead data from several sources, then merges them for downstream processing.

**Nodes involved:**
- Schedule Trigger
- HTTP Request (disabled)
- Google BQ
- Msft Azure Blob
- Amzn AWS S3
- Extract from File
- Merge1
- Goog Sheets
- Merge2

#### 1) Schedule Trigger
- **Type / role:** `Schedule Trigger` — entry point; runs on time interval.
- **Config choices:** Runs daily at **12:02** (interval rule with hour/minute).
- **Outputs to:** HTTP Request, Google BQ, Msft Azure Blob, Amzn AWS S3, Goog Sheets (in parallel).
- **Failure modes:** n8n scheduling disabled/inactive workflow; timezone expectations (n8n instance timezone).

#### 2) HTTP Request (disabled)
- **Type / role:** `HTTP Request` — optional lead source via an external actor endpoint (Apify-like).
- **Config choices:** Disabled; marked “Not Recommended”. Uses JSON body with many fields referencing `$json.Location`, `$json['Lead Number']`, `$json['Business Type']`.
- **Connections:** Trigger would feed it, but it has no downstream connections.
- **Failure modes:** If enabled, expression failures (missing input fields), auth/endpoint errors, rate limits.

#### 3) Google BQ
- **Type / role:** `Google BigQuery` — pulls agent rows from a BigQuery table.
- **Config choices:** Executes SQL:
  - `SELECT * FROM fiverproject-01.DatasetBQ_SufferingDigsTheWell.TableSufferinDigsTheWell3Rows LIMIT 1000`
- **Credentials:** Google BigQuery OAuth2.
- **Outputs to:** Merge2 (input index 0).
- **Failure modes:** Dataset/table permissions, invalid SQL, OAuth token expiry, project mismatch.

#### 4) Msft Azure Blob
- **Type / role:** `Azure Storage` — downloads a CSV blob from Azure container.
- **Config choices:** Container `n8ncontainer`, blob `N8N - Suffering Digs The Well Automation - Azure.csv`, operation **get**.
- **Credentials:** Azure Storage Shared Key.
- **Outputs to:** Merge1 (input index 0).
- **Failure modes:** Missing blob/container, shared key revoked, large file memory constraints.

#### 5) Amzn AWS S3
- **Type / role:** `AWS S3` — retrieves a CSV from S3.
- **Config choices:** Bucket `n8n-data-bucketname`, file key `N8N - Suffering Digs The Well Automation - AWS.csv`.
- **Credentials:** AWS IAM.
- **Outputs to:** Merge1 (input index 1).
- **Failure modes:** Missing object, IAM policy denies, region misconfig, file too large.

#### 6) Merge1
- **Type / role:** `Merge` — merges the two binary/file inputs (Azure + S3) into one stream for file extraction.
- **Config choices:** Default merge behavior (n8n Merge v3.2 default). Practically used to combine two file inputs before extraction.
- **Inputs:** Msft Azure Blob, Amzn AWS S3.
- **Outputs to:** Extract from File.
- **Failure modes:** If one input empty, merge behavior can produce no items or partial items depending on merge mode; binary data naming mismatches.

#### 7) Extract from File
- **Type / role:** `Extract From File` — parses content from incoming file (likely CSV) into JSON items.
- **Config choices:** Defaults (options empty).
- **Inputs:** Merge1 output.
- **Outputs to:** Merge2 (input index 1).
- **Failure modes:** Unsupported encoding/delimiters, malformed CSV, missing binary property.

#### 8) Goog Sheets
- **Type / role:** `Google Sheets` — reads a sheet as another lead source.
- **Config choices:** References document **“N8N - Suffering Digs The Well Automation”** and sheet gid `1743918742` (named “Sheets” in cached result).
- **Credentials:** Google Sheets OAuth2.
- **Outputs to:** Merge2 (input index 2).
- **Failure modes:** Sheet permissions, wrong sheet ID, API quota.

#### 9) Merge2
- **Type / role:** `Merge` — combines **3 inputs** (BQ + extracted file + Sheets) into one stream.
- **Config choices:** `numberInputs: 3`.
- **Inputs:** Google BQ (0), Extract from File (1), Goog Sheets (2).
- **Outputs to:** Check (disabled), Data Collection Prompt.
- **Failure modes:** Mixed schemas; depending on merge behavior, you may get interleaved items or only matched pairs—ensure merge mode is what you intend for “union”.

---

### Block B — AI lead discovery (agent extraction from public webpage)
**Overview:** Creates a strict extraction prompt, uses an AI agent with a structured parser to return a small array of eligible agents, then normalizes into individual items.

**Nodes involved:**
- Data Collection Prompt
- AI Agent Determines Which Email to Email1
- OpenAI Chat Model2
- Structured Output Parser1
- Data Collection Normalizer
- Check2 (disabled)

#### 1) Data Collection Prompt
- **Type / role:** `Code` — builds `chatInput` instructions for lead extraction.
- **Config choices:**
  - Hard-coded source page: `https://manuscriptwishlist.com/find-agentseditors/agent-list/`
  - `itemCount = 3` maximum agents returned
  - Requires strict JSON *string* output representing an array of up to 3 objects with exact ordered keys.
  - Forces: `"Eligible for Submission" = "TRUE"`, `"Email Submission Enabled" = "TRUE"`, `"Submission Completed" = "FALSE"`.
- **Input:** Merge2 output (but it does not actually use the input items; it returns a single prompt item).
- **Output:** To AI Agent Determines Which Email to Email1.
- **Edge cases / issues:**
  - The first line contains a stray expression: `` `$input.all().length` `` which does nothing but is confusing and could be removed for clarity.
  - The prompt asks the model to “extract from webpages” but the agent has no explicit browsing tool configured here; without a web-capable tool or pre-fetched HTML, the model may hallucinate. (This is a critical reliability risk.)

#### 2) AI Agent Determines Which Email to Email1
- **Type / role:** `LangChain Agent` — executes the extraction task and returns parsed structured data.
- **Config choices:** `hasOutputParser: true` (so Structured Output Parser1 is attached).
- **Language model:** OpenAI Chat Model2 (`gpt-4.1-mini`).
- **Output parser:** Structured Output Parser1 (manual JSON schema enforcing the array of objects and TRUE/FALSE constraints).
- **Inputs:** Data Collection Prompt.
- **Outputs to:** Data Collection Normalizer.
- **Failure modes:** LLM returns invalid JSON; schema mismatch (additional keys); inability to access web content; timeouts.

#### 3) OpenAI Chat Model2
- **Type / role:** `OpenAI Chat Model` — provides LLM to the agent.
- **Config choices:** model = `gpt-4.1-mini`.
- **Credentials:** OpenAI API credential.
- **Failure modes:** API quota, invalid key, model availability.

#### 4) Structured Output Parser1
- **Type / role:** `Structured Output Parser` — validates and coerces agent output to a strict schema.
- **Config choices:** Manual JSON Schema:
  - Top-level is an array; each item must contain required keys.
  - Forces enums: `"Email Submission Enabled": ["TRUE"]`, `"Eligible for Submission": ["TRUE"]`, `"Submission Completed": ["FALSE"]`.
- **Failure modes:** Any deviation causes parsing failure, agent may retry or fail depending on n8n agent settings.

#### 5) Data Collection Normalizer
- **Type / role:** `Code` — converts an array output into individual n8n items.
- **Config choices:** Expects agent output in `$input.first().json.output` as an array; falls back to root if root is an array.
- **Outputs to:** Check2 (disabled), Mkt Prompt.
- **Failure modes:** If the agent output structure changes (e.g., `output` is string), it produces empty array.

#### 6) Check2 (disabled)
- **Type / role:** `Google Sheets` appendOrUpdate — debug sink to write collected leads to a “Dev Temp” sheet.
- **Config choices:** Matching column: `Agent Email to Email`. Maps fields from normalized data.
- **Failure modes:** Disabled; if enabled, schema mismatch or missing columns.

---

### Block C — Eligibility decision (email vs no email)
**Overview:** Uses an LLM agent to decide whether to email each lead and routes eligible leads to research.

**Nodes involved:**
- Mkt Prompt
- Eligibility Agent
- OpenAI Chat Model
- Structured Output Parser
- Simple Memory
- Mkt Normalizer
- Switch1

#### 1) Mkt Prompt
- **Type / role:** `Code` — builds a strict decision prompt for the eligibility agent.
- **Config choices:**
  - Uses incoming lead fields: `Literary Agent Name`, `Agent Email to Email`, `Accepted Genres`, `Email Submission Enabled`, `Eligible for Submission`.
  - Decision rule: send email only if both flags are TRUE and email is not UNKNOWN.
  - Requires strict JSON: `{"send_email":"TRUE|FALSE","selected_email":"string","reason":"string"}`
- **Inputs:** Data Collection Normalizer (collected agents) in the current graph.
- **Outputs to:** Eligibility Agent.
- **Failure modes:** Missing expected field names → prompt includes “undefined” values, possibly wrong decision.

#### 2) Eligibility Agent
- **Type / role:** `LangChain Agent` — returns eligibility decision.
- **Config choices:** `hasOutputParser: true`.
- **Language model:** OpenAI Chat Model (`gpt-4.1-mini`).
- **Memory:** Simple Memory attached.
- **Output parser:** Structured Output Parser (example-based schema).
- **Outputs to:** Mkt Normalizer.
- **Failure modes:** Bad JSON; schema mismatch; memory key issues (see below).

#### 3) OpenAI Chat Model
- **Type / role:** `OpenAI Chat Model` for Eligibility Agent.
- **Config:** `gpt-4.1-mini`.
- **Failure modes:** auth/quota.

#### 4) Structured Output Parser
- **Type / role:** `Structured Output Parser` — validates eligibility decision.
- **Config choices:** Uses a **JSON schema example** (not a strict JSON Schema):
  - expects keys `send_email`, `selected_email`, `reason`.
- **Failure modes:** Model returns extra keys or non-JSON → parse fails.

#### 5) Simple Memory
- **Type / role:** `Buffer Window Memory` — provides short-term context memory keyed per lead.
- **Config choices:** `sessionIdType: customKey`, `sessionKey = {{$json['Primary Information Source']}}`.
- **Connections:** Connected to Eligibility Agent as `ai_memory`.
- **Failure modes / edge cases:**
  - If `Primary Information Source` is missing/empty, all leads may share the same memory session, causing cross-contamination.

#### 6) Mkt Normalizer
- **Type / role:** `Code` — flattens agent decision from `item.json.output`.
- **Config choices:** Outputs:
  - `send_email` default `"FALSE"`
  - `selected_email` default `"UNKNOWN"`
  - `reason` default `""`
- **Outputs to:** Switch1.
- **Failure modes:** If decision is not in `output`, all defaults will be used.

#### 7) Switch1
- **Type / role:** `Switch` — routes eligible leads to research.
- **Config choices:**
  - Output 1 renamed: **Eligib2Mail** when `{{$json.send_email}} == "TRUE"`.
  - Output 2 renamed: **InEligib2Mail** when `{{$json.output.send_email}} != "TRUE"`.
- **Outputs:** Only output 1 is connected to Research Prompt.
- **Important issue:** The second rule references `{{$json.output.send_email}}` while the first uses `{{$json.send_email}}`. After **Mkt Normalizer**, the field is at top-level (`send_email`), not nested under `output`. This inconsistency can cause misrouting or unexpected behavior.

---

### Block D — Research enrichment (source-backed brief)
**Overview:** For eligible leads, builds a research prompt, executes a research agent with structured output, then flattens results for downstream personalization and optional sheet logging.

**Nodes involved:**
- Research Prompt
- Research Agent
- OpenAI Chat Model3
- Structured Output Parser3
- Research Normalizer
- check3

#### 1) Research Prompt
- **Type / role:** `Code` — creates `chatInput` for research with strict anti-hallucination and citation requirements.
- **Config choices:**
  - Pulls: `Primary Information Source`, agent name, agent email.
  - Also includes eligibility decision fields (supports nested `output.*` or top-level).
  - Requires output JSON object with keys:
    `agent_email, agent_name, agency, professional_background, genres_represented[], notable_clients_or_books[], public_statements_or_interviews[{context,summary,source}], personalization_angles[], sources[]`
  - Forces:
    - Unknown => `"UNKNOWN"` (or array containing `"UNKNOWN"`).
    - Every factual claim must have URLs in `sources`.
    - Each personalization angle must end with `(source: <URL>)`.
- **Input:** Switch1 eligible output.
- **Output:** Research Agent.
- **Failure modes:** If the agent cannot browse or retrieve sources, it may return UNKNOWNs or hallucinate (prompt tries to prevent this).

#### 2) Research Agent
- **Type / role:** `LangChain Agent` — generates the research brief.
- **Config choices:** `hasOutputParser: true`.
- **Language model:** OpenAI Chat Model3 (`gpt-4.1-mini`).
- **Output parser:** Structured Output Parser3 (strict JSON schema).
- **Output:** Research Normalizer.
- **Failure modes:** Schema mismatch; invalid URL strings; long responses/timeouts.

#### 3) Structured Output Parser3
- **Type / role:** Structured schema validator for research output.
- **Config:** Manual JSON schema; forbids additional properties; enforces array/object shapes for claims/interviews.
- **Failure modes:** Any missing keys or wrong types causes parsing failure.

#### 4) Research Normalizer
- **Type / role:** `Code` — flattens arrays/objects into strings for sheet/email usage.
- **Config choices:**
  - Converts arrays to comma-separated string
  - Converts objects to `key: value; ...`
  - If item contains `json.output` object, flattens that instead
- **Outputs to:** SalesAgentPrompt and check3 (parallel).
- **Failure modes:** Loss of structure (citations become plain text); if downstream needs arrays, they’re gone.

#### 5) check3
- **Type / role:** `Google Sheets` appendOrUpdate — persists enrichment + decision fields (appears to be the main tracking sheet write for research results).
- **Config choices:**
  - operation: `appendOrUpdate`
  - matchingColumns: `agent_email`
  - Populates many columns using cross-node references like:
    - `$('Mkt Normalizer').item.json.send_email`
    - `$('Data Collection Normalizer').item.json['Agent Email to Email']`
- **Edge cases:**
  - Cross-node item referencing is sensitive to item order and execution mode; if multiple items run, ensure the correct item pairing.
  - One mapping bug: `"Eligible for Submission"` is assigned from `Email Submission Enabled` instead of the `Eligible for Submission` field.

---

### Block E — Sales personalization, email send, and post-send update
**Overview:** Creates a sales prompt, uses an agent to draft email opener + subject in strict JSON, sends HTML email via Gmail, then updates Google Sheets submission status.

**Nodes involved:**
- SalesAgentPrompt
- Sales Team
- OpenAI Chat Model1
- Code Readability
- Send a message
- Update Submission Time
- Sticky Note (Agent Task for sales) [documented in table]

#### 1) SalesAgentPrompt
- **Type / role:** `Code` — constructs the outreach drafting prompt (`chatInput`) using research fields.
- **Config choices:**
  - Uses agent fields (supports both `agent_name` and `"Literary Agent Name"`; likewise for email)
  - Book context hard-coded:
    - Title: *Suffering Digs The Well*
    - Nonfiction theme: meaning-making through suffering (existential + psychology)
  - Output must be JSON exactly:
    `{"to_email":"string","agent_name":"string","subject":"string","personalized_opener_html":"string"}`
  - `personalized_opener_html` must be exactly one `<p>...</p>`
  - Subject: 6–10 words; include title or theme
- **Input:** Research Normalizer output.
- **Output:** Sales Team agent.
- **Failure modes:** If research normalizer flattened sources awkwardly, personalization may be weak/incorrect.

#### 2) Sales Team
- **Type / role:** `LangChain Agent` — produces the JSON needed for sending email.
- **Config choices:** Notes “Message personalization”.
- **Language model:** OpenAI Chat Model1 (`gpt-4.1-mini`).
- **Output:** Code Readability.
- **Failure modes:** Returns non-JSON or includes markdown; missing keys; HTML not compliant.

#### 3) Code Readability
- **Type / role:** `Code` — parses the agent’s raw JSON string from `item.json.output`.
- **Config choices:**
  - `JSON.parse(raw)`; on error outputs `{parse_error:true, error, raw}`.
- **Output:** Send a message.
- **Failure modes:** If agent returns already-parsed object (not string), `JSON.parse` will fail; if `output` is absent, `raw` becomes `""`.

#### 4) Send a message (Gmail)
- **Type / role:** `Gmail` — sends the outreach email.
- **Config choices:**
  - **sendTo is hard-coded** to `mex3woof@gmail.com` (not `{{$json.to_email}}`)
  - subject: `{{$json.subject}}`
  - message: large HTML template; inserts:
    - `{{$json.agent_name}}`
    - `{{$json.personalized_opener_html}}`
  - After send, goes to Update Submission Time.
- **Credentials:** Gmail OAuth2.
- **Failure modes / risks:**
  - Hard-coded recipient prevents real outreach; likely a testing placeholder.
  - HTML includes sensitive claims about author experience; ensure compliance with outreach policies and truthfulness.
  - Gmail rate limits, “from” restrictions, spam classification.

#### 5) Update Submission Time (Google Sheets)
- **Type / role:** `Google Sheets` appendOrUpdate — marks the agent as emailed and sets timestamps.
- **Config choices:**
  - matchingColumns: `agent_email`
  - sets `Submission Completed = "TRUE"`
  - sets `Submission Timestamp` using `new Date().toLocaleString('en-US', ...)` (not ISO 8601)
  - agent_email is pulled from `$('Research Normalizer').item.json.agent_email`
- **Failure modes / edge cases:**
  - Timestamp format does not match ISO 8601; earlier blocks required ISO (in data collection). This creates inconsistency.
  - If `agent_email` doesn’t match the sheet key exactly, it will append duplicates instead of updating.

---

### Block F — Chat-triggered analysis & visualization (auxiliary)
**Overview:** A chat message can trigger an analysis branch that runs an agent, normalizes output, and generates a QuickChart. Several analytics/BI nodes exist but are disabled.

**Nodes involved:**
- When chat message received
- Analysis Prompt
- Eligibility Agent1
- OpenAI Chat Model4
- Analysis Normalizer
- QuickChart1
- Google Analytics 4 (disabled)
- PowerBi Dashboard (disabled)

#### 1) When chat message received
- **Type / role:** `LangChain Chat Trigger` — secondary entry point for interactive runs.
- **Config choices:** `agentName: "jOHN"`, available in chat = true.
- **Output:** Analysis Prompt.
- **Failure modes:** Webhook not reachable; chat not enabled in instance.

#### 2) Analysis Prompt
- **Type / role:** `Code` — placeholder logic; sets `myNewField = 1` on each item.
- **Output:** Eligibility Agent1.
- **Failure modes:** None significant; it’s minimal.

#### 3) Eligibility Agent1
- **Type / role:** `LangChain Agent` — used here for analysis (not the same as marketing eligibility).
- **Language model:** OpenAI Chat Model4 (`gpt-4.1-mini`).
- **Output:** Analysis Normalizer.
- **Failure modes:** No attached structured output parser in this branch; output shape may vary.

#### 4) Analysis Normalizer
- **Type / role:** `Code` — extracts `send_email`, `selected_email`, `reason` from `item.json.output` (same logic as Mkt Normalizer).
- **Output:** QuickChart1.
- **Failure modes:** If the agent doesn’t output these keys, defaults are used.

#### 5) QuickChart1
- **Type / role:** `QuickChart` — generates a chart.
- **Config choices:** Hard-coded data `1\n2\n3\n4\n`; labels include “Completed” and “not completed”.
- **Failure modes:** Chart doesn’t reflect real data; needs wiring to actual counts.

#### 6) Google Analytics 4 (disabled) & PowerBi Dashboard (disabled)
- **Type / role:** Optional reporting integrations (currently disabled).
- **Connections:** GA4 → PowerBI (if enabled).
- **Failure modes:** OAuth, property configuration, report metrics/dimensions required.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Scheduled entry point | — | HTTP Request; Google BQ; Msft Azure Blob; Amzn AWS S3; Goog Sheets | # Data Engineering Team - Lead Generation - populate database |
| HTTP Request | HTTP Request | (Disabled) Optional external lead fetch | Schedule Trigger | — | # Notes: 1. I used afew sources in case you want to do a PLUG-N-PLAY 2. Do not use HTTP UNLESS NECESSARY cos Want to be fully Agentic and not Automation (Final Notes:#underline and bold) I designed and implemeted this whole pipeline from scratch in 3 days. If i can, so can you :] Go programmers! ❤️❤️ |
| Google BQ | Google BigQuery | Pull leads from BigQuery | Schedule Trigger | Merge2 | # Data Engineering Team - Lead Generation - populate database |
| Msft Azure Blob | Azure Storage | Download CSV from Azure Blob | Schedule Trigger | Merge1 | # Data Engineering Team - Lead Generation - populate database |
| Amzn AWS S3 | AWS S3 | Download CSV from S3 | Schedule Trigger | Merge1 | # Data Engineering Team - Lead Generation - populate database |
| Merge1 | Merge | Combine Azure+S3 file streams | Msft Azure Blob; Amzn AWS S3 | Extract from File | # Data Engineering Team - Lead Generation - populate database |
| Extract from File | Extract From File | Parse file contents into JSON | Merge1 | Merge2 | # Data Engineering Team - Lead Generation - populate database |
| Goog Sheets | Google Sheets | Pull leads from Google Sheets | Schedule Trigger | Merge2 | # Data Engineering Team - Lead Generation - populate database |
| Merge2 | Merge | Combine 3 lead streams | Google BQ; Extract from File; Goog Sheets | Check; Data Collection Prompt | # Data Engineering Team - Lead Generation - populate database |
| Check | Google Sheets | (Disabled) Debug append/update | Merge2 | — |  |
| Data Collection Prompt | Code | Build AI extraction prompt (agent discovery) | Merge2 | AI Agent Determines Which Email to Email1 | ## Agent Task - Find Compatible Literary agents that represents 1. **non-fiction** 2. **Genre:** Memoir, Spiritual, Self-help, Psychology, Relationships, Family. |
| AI Agent Determines Which Email to Email1 | LangChain Agent | Extract eligible agents (structured) | Data Collection Prompt | Data Collection Normalizer | ## Agent Task - Find Compatible Literary agents that represents 1. **non-fiction** 2. **Genre:** Memoir, Spiritual, Self-help, Psychology, Relationships, Family. |
| OpenAI Chat Model2 | OpenAI Chat Model | LLM for extraction agent | — | AI Agent Determines Which Email to Email1 (ai_languageModel) | ## Agent Task - Find Compatible Literary agents that represents 1. **non-fiction** 2. **Genre:** Memoir, Spiritual, Self-help, Psychology, Relationships, Family. |
| Structured Output Parser1 | Structured Output Parser | Validate extracted agents array | — | AI Agent Determines Which Email to Email1 (ai_outputParser) | ## Agent Task - Find Compatible Literary agents that represents 1. **non-fiction** 2. **Genre:** Memoir, Spiritual, Self-help, Psychology, Relationships, Family. |
| Data Collection Normalizer | Code | Split extracted array into items | AI Agent Determines Which Email to Email1 | Check2; Mkt Prompt | ## Agent Task - Find Compatible Literary agents that represents 1. **non-fiction** 2. **Genre:** Memoir, Spiritual, Self-help, Psychology, Relationships, Family. |
| Check2 | Google Sheets | (Disabled) Debug write of collected agents | Data Collection Normalizer | — |  |
| Mkt Prompt | Code | Build eligibility-decision prompt | Data Collection Normalizer | Eligibility Agent | # Marketing & Research Team |
| Eligibility Agent | LangChain Agent | Decide send_email TRUE/FALSE | Mkt Prompt | Mkt Normalizer | ### AI Agent Determines Which Email to Email |
| OpenAI Chat Model | OpenAI Chat Model | LLM for Eligibility Agent | — | Eligibility Agent (ai_languageModel) | ### AI Agent Determines Which Email to Email |
| Structured Output Parser | Structured Output Parser | Validate eligibility JSON | — | Eligibility Agent (ai_outputParser) | ### AI Agent Determines Which Email to Email |
| Simple Memory | Buffer Window Memory | Memory keyed by Primary Information Source | — | Eligibility Agent (ai_memory) | ### AI Agent Determines Which Email to Email |
| Mkt Normalizer | Code | Flatten eligibility output | Eligibility Agent | Switch1 | # Marketing & Research Team |
| Switch1 | Switch | Route eligible leads to research | Mkt Normalizer | Research Prompt | # Marketing & Research Team |
| Research Prompt | Code | Build research brief prompt (must cite sources) | Switch1 | Research Agent | ## Agent Task ### For Each Literary, Research on them to write tailored email |
| Research Agent | LangChain Agent | Generate structured research brief | Research Prompt | Research Normalizer | # Marketing & Research Team |
| OpenAI Chat Model3 | OpenAI Chat Model | LLM for Research Agent | — | Research Agent (ai_languageModel) | # Marketing & Research Team |
| Structured Output Parser3 | Structured Output Parser | Validate research schema | — | Research Agent (ai_outputParser) | # Marketing & Research Team |
| Research Normalizer | Code | Flatten research fields to strings | Research Agent | SalesAgentPrompt; check3 | # Marketing & Research Team |
| check3 | Google Sheets | Write combined qualification+research to sheet | Research Normalizer | — | ## Data Analysis Team |
| SalesAgentPrompt | Code | Build email-personalization prompt | Research Normalizer | Sales Team | # Sales Team |
| Sales Team | LangChain Agent | Draft opener+subject JSON | SalesAgentPrompt | Code Readability | ## Agent Task AI Writes Message personalized emails to improve sales % Hit-rate |
| OpenAI Chat Model1 | OpenAI Chat Model | LLM for Sales Team | — | Sales Team (ai_languageModel) | ## Agent Task AI Writes Message personalized emails to improve sales % Hit-rate |
| Code Readability | Code | Parse JSON string output | Sales Team | Send a message | # Sales Team |
| Send a message | Gmail | Send HTML email | Code Readability | Update Submission Time | # Sales Team |
| Update Submission Time | Google Sheets | Mark submission completed + timestamp | Send a message | — | # Sales Team |
| When chat message received | Chat Trigger | Chat entry point for analysis branch | — | Analysis Prompt | ## Agent Task ### Write in chat, send to Prompt and AI Generates Quick Chart. Alternaively, you can use BI or Google Analytics too |
| Analysis Prompt | Code | Placeholder analysis prep | When chat message received | Eligibility Agent1 | ## Agent Task ### Write in chat, send to Prompt and AI Generates Quick Chart. Alternaively, you can use BI or Google Analytics too |
| Eligibility Agent1 | LangChain Agent | Analysis agent (no parser attached) | Analysis Prompt | Analysis Normalizer | ## Agent Task ### Write in chat, send to Prompt and AI Generates Quick Chart. Alternaively, you can use BI or Google Analytics too |
| OpenAI Chat Model4 | OpenAI Chat Model | LLM for analysis agent | — | Eligibility Agent1 (ai_languageModel) | ## Agent Task ### Write in chat, send to Prompt and AI Generates Quick Chart. Alternaively, you can use BI or Google Analytics too |
| Analysis Normalizer | Code | Extract send_email/selected_email/reason | Eligibility Agent1 | QuickChart1 | ## Agent Task ### Write in chat, send to Prompt and AI Generates Quick Chart. Alternaively, you can use BI or Google Analytics too |
| QuickChart1 | QuickChart | Build chart image/url from data | Analysis Normalizer | — | ## Agent Task ### Write in chat, send to Prompt and AI Generates Quick Chart. Alternaively, you can use BI or Google Analytics too |
| Google Analytics 4 | Google Analytics | (Disabled) Fetch GA4 report | — | PowerBi Dashboard | ## Data Analysis Team |
| PowerBi Dashboard | PowerBI | (Disabled) Fetch PowerBI data | Google Analytics 4 | — | ## Data Analysis Team |
| Sticky Note1 | Sticky Note | Comment block label | — | — | # Data Engineering Team - Lead Generation - populate database |
| Sticky Note2 | Sticky Note | Notes / guidance | — | — | # Notes: 1. I used afew sources in case you want to do a PLUG-N-PLAY 2. Do not use HTTP UNLESS NECESSARY cos Want to be fully Agentic and not Automation (Final Notes:#underline and bold) I designed and implemeted this whole pipeline from scratch in 3 days. If i can, so can you :] Go programmers! ❤️❤️ |
| Sticky Note12 | Sticky Note | Section label | — | — | ## Data Analysis Team |
| Sticky Note16 | Sticky Note | Agent task requirements | — | — | ## Agent Task - Find Compatible Literary agents that represents 1. **non-fiction** 2. **Genre:** Memoir, Spiritual, Self-help, Psychology, Relationships, Family. |
| Sticky Note17 | Sticky Note | Section label | — | — | ### AI Agent Determines Which Email to Email |
| Sticky Note21 | Sticky Note | Main objective | — | — | ## Main Objective  ### Marketing 1. Audience discovery (agent discovery) 2. Market segmentation (genre filtering) 3. Channel qualification (email vs platform) 4. Message personalization (agent-specific emails) 5. Campaign readiness (HTML email templates) 6. Funnel top-of-pipeline automation  ### Sales 1. Lead qualification (eligible vs ineligible) 2. Lead deduplication 3. CRM-style tracking (submission status, timestamps) 4. Outreach execution 5. Post-send state updates 6. Pipeline hygiene |
| Sticky Note22 | Sticky Note | Section label | — | — | # Marketing & Research Team |
| Sticky Note23 | Sticky Note | Section label | — | — | # Sales Team |
| Sticky Note3 | Sticky Note | Research task note | — | — | ## Agent Task ### For Each Literary, Research on them to write tailored email |
| Sticky Note4 | Sticky Note | Analysis task note | — | — | ## Agent Task ### Write in chat, send to Prompt and AI Generates Quick Chart. Alternaively, you can use BI or Google Analytics too |
| Sticky Note | Sticky Note | Sales personalization note | — | — | ## Agent Task AI Writes Message personalized emails to improve sales % Hit-rate |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *Qualify and email literary agents with GPT‑4.1, Gmail and Google Sheets*.

2) **Add Trigger: “Schedule Trigger”**  
   - Set schedule to run daily at **12:02** (or your preferred time).

3) **Add optional lead-source nodes (parallel from trigger):**
   1. **Google BigQuery node**
      - Operation: run query  
      - SQL: select rows from your lead table (limit as desired)  
      - Configure **Google BigQuery OAuth2** credentials.
   2. **Azure Storage node**
      - Resource: Blob; Operation: Get  
      - Container: your container name  
      - Blob: your CSV filename  
      - Configure **Azure Storage Shared Key** credentials.
   3. **AWS S3 node**
      - Operation: Get (download)  
      - Bucket: your bucket  
      - File key: your CSV filename  
      - Configure **AWS IAM** credentials.
   4. **Google Sheets node** (read)
      - Select Spreadsheet + Sheet tab that contains lead rows  
      - Configure **Google Sheets OAuth2** credentials.
   5. *(Optional, disabled as in original)* **HTTP Request node**
      - Keep disabled unless you truly need it.

4) **Merge and parse file-based sources**
   1. Add **Merge** node (“Merge1”)
      - Connect: Azure Storage → Merge1 input 0; AWS S3 → Merge1 input 1
   2. Add **Extract From File**
      - Connect: Merge1 → Extract From File  
      - Configure parsing options for your CSV if required.
   3. Add **Merge** node (“Merge2”)
      - Set **Number of Inputs = 3**
      - Connect:
        - BigQuery → Merge2 input 0
        - Extract From File → Merge2 input 1
        - Google Sheets (read) → Merge2 input 2

5) **AI lead discovery branch (build prompt + extraction agent)**
   1. Add **Code** node (“Data Collection Prompt”)
      - Output a single item with `json.chatInput` containing your strict extraction instructions (similar to provided).
      - Decide a maximum count (e.g., 3).
   2. Add **LangChain Agent** node (“AI Agent Determines Which Email to Email1”)
      - Enable “Has Output Parser”.
   3. Add **OpenAI Chat Model** node (“OpenAI Chat Model2”)
      - Model: `gpt-4.1-mini`
      - Configure **OpenAI API** credentials.
      - Connect as the agent’s **ai_languageModel**.
   4. Add **Structured Output Parser** node (“Structured Output Parser1”)
      - Use **manual JSON schema** for the required array-of-objects.
      - Connect as the agent’s **ai_outputParser**.
   5. Add **Code** node (“Data Collection Normalizer”)
      - Convert agent output array into multiple items (one item per agent).
   6. Connect: Merge2 → Data Collection Prompt → AI Agent → Data Collection Normalizer.

6) **Eligibility decision (marketing qualification)**
   1. Add **Code** node (“Mkt Prompt”)
      - Build `chatInput` that instructs the model to output strict JSON: send_email, selected_email, reason.
   2. Add **LangChain Agent** node (“Eligibility Agent”)
      - Enable “Has Output Parser”.
   3. Add **OpenAI Chat Model** node (“OpenAI Chat Model”)
      - Model: `gpt-4.1-mini`
      - Connect to Eligibility Agent as **ai_languageModel**.
   4. Add **Structured Output Parser** node (“Structured Output Parser”)
      - Provide schema/example for the 3 keys.
      - Connect to Eligibility Agent as **ai_outputParser**.
   5. Add **Simple Memory** node (“Simple Memory”)
      - sessionIdType: customKey  
      - sessionKey: use a stable per-agent key (ideally agent email).  
      - Connect to Eligibility Agent as **ai_memory**.
   6. Add **Code** node (“Mkt Normalizer”)
      - Extract `send_email`, `selected_email`, `reason` from `item.json.output`.
   7. Add **Switch** node (“Switch1”)
      - Rule 1: `{{$json.send_email}} == "TRUE"` → eligible output
      - Rule 2: `{{$json.send_email}} != "TRUE"` → ineligible output (recommended; avoid `output.send_email` inconsistency)
   8. Connect: Data Collection Normalizer → Mkt Prompt → Eligibility Agent → Mkt Normalizer → Switch1.

7) **Research enrichment for eligible leads**
   1. Add **Code** node (“Research Prompt”)
      - Build a strict, citation-required `chatInput` and define the exact JSON output schema.
   2. Add **LangChain Agent** node (“Research Agent”)
      - Enable “Has Output Parser”.
   3. Add **OpenAI Chat Model** node (“OpenAI Chat Model3”) with `gpt-4.1-mini`
      - Connect as **ai_languageModel**.
   4. Add **Structured Output Parser** node (“Structured Output Parser3”)
      - Manual schema for the research object.
      - Connect as **ai_outputParser**.
   5. Add **Code** node (“Research Normalizer”)
      - Flatten arrays/objects to strings for sheet/email usage.
   6. Connect: Switch1 (eligible) → Research Prompt → Research Agent → Research Normalizer.

8) **Write research + qualification into Google Sheets (tracking)**
   - Add **Google Sheets** node (“check3”)
     - Operation: appendOrUpdate
     - Matching column: `agent_email` (or your chosen unique key)
     - Map columns from:
       - Research Normalizer output
       - Mkt Normalizer output
       - Original lead fields
     - Connect: Research Normalizer → check3.

9) **Sales personalization and Gmail send**
   1. Add **Code** node (“SalesAgentPrompt”)
      - Build `chatInput` instructing strict JSON with `to_email`, `agent_name`, `subject`, `personalized_opener_html`.
   2. Add **LangChain Agent** node (“Sales Team”)
      - Provide OpenAI Chat Model (`gpt-4.1-mini`) as **ai_languageModel**.
   3. Add **Code** node (“Code Readability”)
      - Parse the agent response into JSON fields (handle parse errors).
   4. Add **Gmail** node (“Send a message”)
      - **Recommended:** set `sendTo = {{$json.to_email}}` (instead of a hard-coded test email)
      - subject = `{{$json.subject}}`
      - message = your HTML template, inserting:
        - `{{$json.agent_name}}`
        - `{{$json.personalized_opener_html}}`
      - Configure **Gmail OAuth2** credentials.
   5. Connect: Research Normalizer → SalesAgentPrompt → Sales Team → Code Readability → Send a message.

10) **Update submission status in Sheets after send**
   - Add **Google Sheets** node (“Update Submission Time”)
     - Operation: appendOrUpdate
     - Matching: `agent_email`
     - Set:
       - `Submission Completed = "TRUE"`
       - `Submission Timestamp = {{new Date().toISOString()}}` (recommended ISO consistency)
   - Connect: Send a message → Update Submission Time.

11) **Chat-driven analysis branch (optional)**
   1. Add **When chat message received** (Chat Trigger)
   2. Add **Analysis Prompt** code node (or real logic)
   3. Add **Eligibility Agent1** with OpenAI model attached
   4. Add **Analysis Normalizer**
   5. Add **QuickChart**
   - Connect: Chat Trigger → Analysis Prompt → Agent → Normalizer → QuickChart.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Do not use HTTP UNLESS NECESSARY cos Want to be fully Agentic and not Automation” | Workflow sticky note guidance (HTTP Request node is disabled). |
| “I designed and implemeted this whole pipeline from scratch in 3 days. If i can, so can you :] Go programmers!” | Author note (Sticky Note2). |
| Main objective lists Marketing + Sales pipeline goals (audience discovery, segmentation, qualification, personalization, tracking, hygiene). | Sticky Note21. |
| Lead discovery target page | https://manuscriptwishlist.com/find-agentseditors/agent-list/ |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.