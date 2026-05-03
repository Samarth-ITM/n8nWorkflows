Mine user complaints and generate insight reports with Olostep, Gemini and Google Docs

https://n8nworkflows.xyz/workflows/mine-user-complaints-and-generate-insight-reports-with-olostep--gemini-and-google-docs-13435


# Mine user complaints and generate insight reports with Olostep, Gemini and Google Docs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Complaint Mining*  
**Template title (given):** *Mine user complaints and generate insight reports with Olostep, Gemini and Google Docs*  

**Purpose:**  
This workflow collects real user complaints from the web based on a problem statement/keywords, classifies each complaint’s pain level, extracts recurring keywords and their mention counts, analyzes “demand signal” and frustration level, then generates a structured markdown report and writes it into a newly created Google Doc.

**Target use cases:**
- SaaS founders validating a problem space (demand signal)
- Product/UX teams mining recurring pain points
- Support/success teams prioritizing issues by severity
- Researchers collecting verbatim quotes + sources for evidence

### Logical Blocks
**1.1 Trigger & Input Capture**
- Receives a “Problem Statement or Keywords” via an n8n Form trigger.

**1.2 Complaint Mining (Verbatim Quotes)**
- Uses an AI Agent with the Olostep `/answers` endpoint as a tool to search the web and return **at least 10** verbatim complaint snippets, each with URL and simplified source title.
- Parses output into a structured JSON schema and splits into individual items.

**1.3 Pain Level Classification**
- Runs an LLM chain to label each quote as High/Medium/Low pain.
- Merges pain labels back with the corresponding quote items.

**1.4 Keyword Mining (Mentions + Totals)**
- Uses a second AI Agent (also with Olostep tool access) to extract recurring keywords and mention counts, plus a total mentions count.
- Splits keyword list into items and merges keyword items with quote items.

**1.5 Data Preparation & Aggregation**
- Aggregates fields across items into arrays for downstream analysis and report writing.

**1.6 Demand/Frustration Analysis & Structured Extraction**
- Analyzer LLM produces a narrative analysis and signal classification.
- Information Extractor pulls two structured attributes: frustration percentage and demand signal.

**1.7 Report Writing & Google Docs Publishing**
- Writing agent produces a markdown report (without changing inputs).
- Creates a Google Doc and inserts the report.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Input Capture

**Overview:**  
Starts the workflow when a user submits a form containing a problem statement or keywords. This single input drives both mining branches (verbatim quotes and keywords).

**Nodes involved:**
- On form submission

#### Node: **On form submission**
- **Type / role:** `n8n-nodes-base.formTrigger` — entry point; collects user input.
- **Configuration (interpreted):**
  - Form title: **“Mine Complaint”**
  - Field: **“Problem Statement or Keywords”** (required)
- **Key variables/expressions:**
  - Accessed later as: `$('On form submission').item.json['Problem Statement or Keywords']`
- **Outputs / connections:**
  - Outputs to:
    - **Verbatim Quotes Agent**
    - **Keyword Agent**
- **Edge cases / failures:**
  - Empty input prevented by required field.
  - Very long input may reduce search quality or exceed LLM/tool limits downstream.

**Sticky note covering this block:**
- “## Trigger\nStart the workflow with user feedback or complaint text.”

---

### 2.2 Complaint Mining (Verbatim Quotes)

**Overview:**  
An AI agent generates multiple search queries and uses Olostep to retrieve real user complaint quotes, returning structured items (quote, url, source_title). Output is parsed and split into individual findings.

**Nodes involved:**
- Verbatim Quotes Agent
- Structured Output Parser
- olostep (HTTP Request Tool)
- Wait
- Split Out

#### Node: **Verbatim Quotes Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — web research agent that must output structured findings.
- **Configuration (interpreted):**
  - Input text: `Problem Statement Or Keywords: {{ $json['Problem Statement or Keywords'] }}`
  - System message instructs:
    - generate 3–5+ queries
    - search forums (Reddit via `site:reddit.com`, review sites like G2/Capterra, communities)
    - extract **verbatim complaint snippets** + **URL** + **source title** (short label like “Reddit r/saas”)
    - output **at least 10 items**
    - do not analyze or summarize
  - Has Output Parser enabled.
  - `maxTries: 2`, `retryOnFail: true`
- **Model/tooling connections:**
  - Uses **Google Gemini Chat Model2** as `ai_languageModel`.
  - Uses **Structured Output Parser** as `ai_outputParser`.
  - Uses **olostep** node as an `ai_tool`.
- **Outputs / connections:**
  - Main output → **Wait**
- **Edge cases / failures:**
  - Tool failures (Olostep auth/quotas/network) can cause partial/no results.
  - If agent output does not match schema, parsing fails.
  - Search results may include non-verbatim content; agent is instructed to avoid it but can still drift.
  - URLs may be inaccessible, blocked, or rate-limited.

**Sticky note covering this block:**
- “## Verbatim Quotes Agent\nUses Olostep /answer endpoint as a tool to searches for complaints about the problem statement/keywords directly from the mouth of potential customers from forums like Reddit, Hacker news, and other specialized forums.”

#### Node: **Structured Output Parser**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON structure for agent output.
- **Configuration (interpreted):**
  - Expected schema example:
    - `findings[]` with `{ verbatim_quote, url, source_title }`
- **Connections:**
  - Connected to **Verbatim Quotes Agent** as its `ai_outputParser`.
- **Edge cases / failures:**
  - If LLM returns non-JSON, missing keys, or different structure → parser error.
  - Quotes containing unescaped characters can break JSON formatting if the model doesn’t escape properly.

#### Node: **olostep**
- **Type / role:** `n8n-nodes-base.httpRequestTool` — tool node callable by AI agents to fetch/search via Olostep.
- **Configuration (interpreted):**
  - POST `https://api.olostep.com/v1/answers`
  - Body includes `task` (tool input), mapped from AI tool call:
    - `task = $fromAI('parameters0_Value', 'this is the search query field', 'string')`
  - Authentication: predefined credential type `olostepScrapeApi`
- **Credentials required:**
  - **Olostep Scrape account**
- **Connections:**
  - Exposed as `ai_tool` to:
    - Verbatim Quotes Agent
    - Keyword Agent
- **Edge cases / failures:**
  - 401/403 if credential invalid.
  - 429 if rate limited / quota exceeded.
  - Response shape changes could break agent assumptions.
  - Tool timeouts for complex queries.

**Sticky note:**
- “## Olostep /answer enpoint”

#### Node: **Wait**
- **Type / role:** `n8n-nodes-base.wait` — pauses execution before downstream splitting.
- **Configuration (interpreted):**
  - Wait amount: **30** (seconds)
- **Connections:**
  - Input: Verbatim Quotes Agent
  - Output: Split Out
- **Edge cases / failures:**
  - Adds latency; not a reliability guarantee.
  - If upstream produced no `output.findings`, downstream split fails.

#### Node: **Split Out**
- **Type / role:** `n8n-nodes-base.splitOut` — converts an array of findings into one item per finding.
- **Configuration (interpreted):**
  - Field to split: `output.findings`
- **Connections:**
  - Output to:
    - Pain Level Identifier
    - Merge (input index 1)
- **Edge cases / failures:**
  - If `output.findings` is missing/not an array → node error.
  - Empty array → produces no items (downstream merges may misalign).

---

### 2.3 Pain Level Classification

**Overview:**  
Each verbatim quote is classified into High/Medium/Low pain using a rubric-based prompt. The resulting label is attached to the corresponding quote item and merged back.

**Nodes involved:**
- Pain Level Identifier
- Wait1
- pain level (Set)
- Merge

#### Node: **Pain Level Identifier**
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — single-step LLM chain for classification.
- **Configuration (interpreted):**
  - Input text: `list of verbatim quotes: {{ $json.verbatim_quote }}`
  - Prompt defines rubric and strict output format (exactly “High”, “Medium”, or “Low”).
  - `maxTries: 2`, `retryOnFail: true`
- **Model connection:**
  - Uses **Google Gemini Chat Model2**.
- **Connections:**
  - Output → Wait1
- **Edge cases / failures:**
  - Model may return extra whitespace or punctuation; downstream Set node uses `{{$json.text}}` so text must be clean.
  - If the chain returns anything other than the three expected tokens, report quality degrades (no validation node exists).

**Sticky note:**
- “## Pain Level Identifier\nTakes the raw verbatim quotes and identifies if this quote defines a high, medium, or low pain.”

#### Node: **Wait1**
- **Type / role:** `n8n-nodes-base.wait`
- **Configuration:** Wait **30** seconds
- **Connections:** Pain Level Identifier → Wait1 → pain level
- **Edge cases:** same as other Wait nodes (latency, not a guarantee).

#### Node: **pain level**
- **Type / role:** `n8n-nodes-base.set` — writes LLM result into a stable field.
- **Configuration (interpreted):**
  - Sets `pain-level` (string) = `{{ $json.text }}`
- **Connections:**
  - Output → Merge (input index 0)
- **Edge cases / failures:**
  - If upstream doesn’t provide `text` field → `pain-level` becomes empty.

#### Node: **Merge**
- **Type / role:** `n8n-nodes-base.merge` — recombines quote items with their pain levels.
- **Configuration (interpreted):**
  - Mode: **combine**
  - Combine by: **position**
- **Connections:**
  - Input 0: pain level
  - Input 1: Split Out (original quote item)
  - Output: Merge1 (input index 0)
- **Edge cases / failures:**
  - Combine-by-position assumes both inputs emit the **same number of items in the same order**.
  - If any quote failed classification (missing item) the merge will misalign pain levels to quotes.

---

### 2.4 Keyword Mining (Mentions + Totals)

**Overview:**  
A second agent searches for recurring keywords related to the problem and returns keyword+mention counts plus a total mentions number. Keywords are split into individual items and merged into the main dataset.

**Nodes involved:**
- Keyword Agent
- Structured Output Parser1
- Wait2
- Split Out1
- Merge1
- total mentions (Set)

#### Node: **Keyword Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — research agent returning structured keyword stats.
- **Configuration (interpreted):**
  - Input: `Problem Statement or Keywords: {{ $json['Problem Statement or Keywords'] }}`
  - System message instructs:
    - generate 3–5+ queries; do searches first
    - extract frequently mentioned keywords and simple counts
    - compute `total_mentions`
  - Has Output Parser enabled.
- **Model/tooling connections:**
  - Uses **Google Gemini Chat Model2**
  - Uses **Structured Output Parser1**
  - Uses **olostep** as an `ai_tool`
- **Connections:**
  - Main output → total mentions → Merge2 (input 2)
  - Main output → Wait2 → Split Out1
- **Edge cases / failures:**
  - Same tool/auth/rate-limit issues as verbatim agent.
  - Schema mismatch breaks parsing.
  - “Mentions” counts are heuristic and depend on search result interpretation.

**Sticky note (note: content label says Verbatim Quotes Agent but describes keywords):**
- “## Verbatim Quotes Agent\nUses Olostep /answer endpoint as a tool to searches for relevant keywords about the problem and defines how many times those keywords have been mentioned.”

#### Node: **Structured Output Parser1**
- **Type / role:** structured parser for Keyword Agent output.
- **Configuration (interpreted):**
  - Expected schema example:
    - `problem_statement` (string)
    - `keywords[]`: `{ keyword, mentions }`
    - `total_mentions` (number)
- **Connections:**
  - Connected to Keyword Agent as `ai_outputParser`.
- **Edge cases:** schema mismatch / invalid JSON.

#### Node: **Wait2**
- **Type / role:** Wait 30 seconds
- **Connections:** Keyword Agent → Wait2 → Split Out1

#### Node: **Split Out1**
- **Type / role:** split keywords array into items
- **Configuration:**
  - Field: `output.keywords`
- **Connections:**
  - Output → Merge1 (input index 1)
- **Edge cases:**
  - Missing `output.keywords` or not array → error.

#### Node: **total mentions**
- **Type / role:** `n8n-nodes-base.set` — extracts total mentions to a simplified field.
- **Configuration:**
  - Sets `total-mentions` = `{{ $json.output.total_mentions }}`
  - (Stored as string in this workflow.)
- **Connections:**
  - Output → Merge2 (input index 2)
- **Edge cases:**
  - If total_mentions missing/not numeric → blank or incorrect string.

---

### 2.5 Data Preparation & Aggregation

**Overview:**  
Merges quote+pain items with keyword items, then aggregates multiple fields into arrays to provide a single combined payload for analysis and report generation.

**Nodes involved:**
- Merge1
- Aggregate

#### Node: **Merge1**
- **Type / role:** `n8n-nodes-base.merge` — merges two streams into one combined dataset.
- **Configuration (interpreted):**
  - Default merge behavior (no explicit mode shown in parameters; node exists with empty parameters).
  - In n8n, Merge defaults typically to “append” unless configured; however connections indicate it is used to bring together:
    - Input 0: Merge (quotes + pain-level)
    - Input 1: Split Out1 (keywords)
- **Connections:**
  - Output → Aggregate
- **Edge cases / failures:**
  - If merge mode is not combine-by-position, outputs may be appended rather than paired.
  - This workflow later aggregates fields, so exact pairing may not be required, but mis-merge can change ordering.

#### Node: **Aggregate**
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates multiple fields into arrays.
- **Configuration (interpreted):**
  - Aggregates fields:
    - `verbatim_quote`
    - `source_title`
    - `keyword`
    - `mentions`
    - `pain-level`
- **Connections:**
  - Output to:
    - Analyzer Agent
    - Merge2 (input index 1)
- **Edge cases / failures:**
  - If some fields are missing in some items, aggregated arrays may contain null/empty entries.
  - Large arrays may exceed LLM context downstream.

**Sticky note:**
- “## Merge & Aggregate\nClean and prepare data for Analysis.”

---

### 2.6 Demand/Frustration Analysis & Structured Extraction

**Overview:**  
The aggregated data is analyzed by an LLM to determine demand signal and frustration percentage. An extractor node then pulls two required attributes into structured fields.

**Nodes involved:**
- Analyzer Agent
- Information Extractor
- Merge3

#### Node: **Analyzer Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — produces an analysis narrative and demand signal classification.
- **Configuration (interpreted):**
  - Input text combines:
    - Problem statement (from trigger)
    - Aggregated verbatim quotes + source titles
    - Aggregated keywords + mentions
    - Total mentions from Keyword Agent: `{{ $('Keyword Agent').item.json.output.total_mentions }}`
  - Prompt: determine if problem has “high/medium/low demand signal” and provide key insights like frustration percentage.
- **Model connection:**
  - Uses **Google Gemini Chat Model2**
- **Connections:**
  - Output to:
    - Information Extractor
    - Merge3 (input index 1)
- **Edge cases / failures:**
  - The prompt does not enforce a strict schema; extraction depends on the downstream extractor being able to interpret the text.
  - If `$('Keyword Agent').item...` is unavailable (branch failure), expression may error.

**Sticky note:**
- “## Analyzer Agent\nTakes the raw data and analyze it and determines if the problem has a high, medium, or low signal.\nAlso provides key insights like the express frustration percantage and other valuable key insights.”

#### Node: **Information Extractor**
- **Type / role:** `@n8n/n8n-nodes-langchain.informationExtractor` — pulls structured attributes from the analyzer’s text.
- **Configuration (interpreted):**
  - Text: `{{ $json.text }}`
  - Required attributes:
    - `express frustration` (percentage of frustration expression)
    - `demand signal` (demand signal classification)
- **Model connection:**
  - Uses **Google Gemini Chat Model2**
- **Connections:**
  - Output → Merge3 (input index 0)
- **Edge cases / failures:**
  - If analyzer output doesn’t clearly include these values, extraction can be wrong or empty.
  - Attributes are required; node may fail if it cannot extract them (depends on node behavior/version).

**Sticky note:**
- “## Information Extractor\nExtracts the frustration percentage and the demand signal from the analysis provided by the analyzer agent.”

#### Node: **Merge3**
- **Type / role:** `n8n-nodes-base.merge` — combines extraction output with analysis text by position.
- **Configuration:**
  - Mode: combine
  - Combine by: position
- **Connections:**
  - Output → Merge2 (input index 0)
- **Edge cases:**
  - Position-based combine assumes both branches produce aligned items (usually 1 item each here).

**Sticky note:**
- “## Merge\nMerge all previous data and prepare it for the final report writing agent.”

---

### 2.7 Report Writing & Google Docs Publishing

**Overview:**  
All computed fields are merged, a writing agent formats them into markdown (without altering content), then a Google Doc is created and the markdown is inserted.

**Nodes involved:**
- Merge2
- AI Agent
- Create a document
- Update a document

#### Node: **Merge2**
- **Type / role:** `n8n-nodes-base.merge` — final assembly of all data inputs for report writing.
- **Configuration:**
  - Mode: combine
  - Combine by: position
  - Number of inputs: 3
- **Inputs:**
  - Input 0: Merge3 (analysis + extracted attributes)
  - Input 1: Aggregate (arrays of quotes/keywords/pain)
  - Input 2: total mentions (Set)
- **Outputs:**
  - Output → AI Agent
- **Edge cases:**
  - If any input missing, merge may fail or produce incomplete dataset.
  - Combine-by-position relies on each input producing the same number of items (typically 1 each at this stage).

#### Node: **AI Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — report writer that outputs markdown.
- **Configuration (interpreted):**
  - Input text includes:
    - Express frustration: `{{ $json.output['express frustration'] }}`
    - Demand signal: `{{ $json.output['demand signal'] }}`
    - Analysis: `{{ $json.text }}`
    - Verbatim Quotes: `{{ $json.verbatim_quote }}`
    - Pain levels: `{{ $json['pain-level'] }}`
    - Source Titles: `{{ $json.source_title }}`
    - Keywords and mentions arrays
  - System message:
    - “Do not change any incoming information”
    - “Do not summarize”
    - Output must be markdown
- **Model connection:**
  - Uses **Google Gemini Chat Model2**
- **Connections:**
  - Output → Create a document
- **Edge cases / failures:**
  - Output size may exceed Google Docs insert limits (large arrays).
  - “Do not summarize” can cause extremely long output, increasing risk of truncation.

**Sticky note:**
- “## Writing agent\nWrites the final report gathering all the information and insights from previous steps, and structure it in a good format.”

#### Node: **Create a document**
- **Type / role:** `n8n-nodes-base.googleDocs` — creates a new Google Doc.
- **Configuration (interpreted):**
  - Title: `Complaint Mining Report For: {{ $('On form submission').item.json['Problem Statement or Keywords'] }}`
  - Folder: `default`
- **Credentials required:**
  - Google Docs OAuth2 (“Google Docs account”)
- **Connections:**
  - Output → Update a document
- **Edge cases / failures:**
  - OAuth permission issues (Docs scope).
  - Folder misconfiguration (if “default” is not available in some environments).

**Sticky note:**
- “## Document creation\nCreates a new document and write the content of the final report into it.”

#### Node: **Update a document**
- **Type / role:** `n8n-nodes-base.googleDocs` — inserts markdown content into the newly created doc.
- **Configuration (interpreted):**
  - Operation: update
  - Document URL/ID: `{{ $json.id }}` (from Create a document output)
  - Action: Insert text = `{{ $('AI Agent').item.json.output }}`
- **Credentials:**
  - Same Google Docs OAuth2
- **Edge cases / failures:**
  - If AI Agent output is very long, insert may fail or partially insert.
  - If Create node returns a different identifier field than `id` in some versions, expression breaks.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | formTrigger | Collect problem statement/keywords | — | Verbatim Quotes Agent; Keyword Agent | ## Trigger<br>Start the workflow with user feedback or complaint text. |
| Verbatim Quotes Agent | langchain.agent | Search web and collect verbatim complaints + sources | On form submission; (ai_tool: olostep); (ai_model: Google Gemini Chat Model2); (ai_outputParser: Structured Output Parser) | Wait | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for complaints about the problem statement/keywords directly from the mouth of potential customers from forums like Reddit, Hacker news, and other specialized forums. |
| Structured Output Parser | outputParserStructured | Enforce schema for verbatim findings | (connected as parser to Verbatim Quotes Agent) | (to agent only) | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for complaints about the problem statement/keywords directly from the mouth of potential customers from forums like Reddit, Hacker news, and other specialized forums. |
| Wait | wait | Pause after quote mining | Verbatim Quotes Agent | Split Out | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for complaints about the problem statement/keywords directly from the mouth of potential customers from forums like Reddit, Hacker news, and other specialized forums. |
| Split Out | splitOut | Split findings array into items | Wait | Pain Level Identifier; Merge | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for complaints about the problem statement/keywords directly from the mouth of potential customers from forums like Reddit, Hacker news, and other specialized forums. |
| Pain Level Identifier | chainLlm | Classify pain level High/Medium/Low | Split Out; (ai_model: Google Gemini Chat Model2) | Wait1 | ## Pain Level Identifier<br>Takes the raw verbatim quotes and identifies if this quote defines a high, medium, or low pain. |
| Wait1 | wait | Pause after pain classification | Pain Level Identifier | pain level | ## Pain Level Identifier<br>Takes the raw verbatim quotes and identifies if this quote defines a high, medium, or low pain. |
| pain level | set | Store pain level in `pain-level` field | Wait1 | Merge | ## Pain Level Identifier<br>Takes the raw verbatim quotes and identifies if this quote defines a high, medium, or low pain. |
| Merge | merge | Combine quote item + pain-level by position | pain level; Split Out | Merge1 | ## Pain Level Identifier<br>Takes the raw verbatim quotes and identifies if this quote defines a high, medium, or low pain. |
| Keyword Agent | langchain.agent | Search web and extract keywords + mention counts + total | On form submission; (ai_tool: olostep); (ai_model: Google Gemini Chat Model2); (ai_outputParser: Structured Output Parser1) | total mentions; Wait2 | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for relevant keywords about the problem and defines how many times those keywords have been mentioned. |
| Structured Output Parser1 | outputParserStructured | Enforce schema for keyword stats | (connected as parser to Keyword Agent) | (to agent only) | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for relevant keywords about the problem and defines how many times those keywords have been mentioned. |
| Wait2 | wait | Pause after keyword mining | Keyword Agent | Split Out1 | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for relevant keywords about the problem and defines how many times those keywords have been mentioned. |
| Split Out1 | splitOut | Split keywords array into items | Wait2 | Merge1 | ## Verbatim Quotes Agent<br>Uses Olostep /answer endpoint as a tool to searches for relevant keywords about the problem and defines how many times those keywords have been mentioned. |
| Merge1 | merge | Merge quotes stream with keywords stream | Merge; Split Out1 | Aggregate | ## Merge & Aggregate<br>Clean and prepare data for Analysis. |
| Aggregate | aggregate | Aggregate fields into arrays for analysis/report | Merge1 | Analyzer Agent; Merge2 | ## Merge & Aggregate<br>Clean and prepare data for Analysis. |
| Analyzer Agent | chainLlm | Analyze demand signal + frustration + insights | Aggregate; (ai_model: Google Gemini Chat Model2) | Information Extractor; Merge3 | ## Analyzer Agent<br>Takes the raw data and analyze it and determines if the problem has a high, medium, or low signal.<br>Also provides key insights like the express frustration percantage and other valuable key insights. |
| Information Extractor | informationExtractor | Extract “express frustration” % and “demand signal” | Analyzer Agent; (ai_model: Google Gemini Chat Model2) | Merge3 | ## Information Extractor<br>Extracts the frustration percentage and the demand signal from the analysis provided by the analyzer agent. |
| Merge3 | merge | Combine extracted attributes with analysis text | Information Extractor; Analyzer Agent | Merge2 | ## Merge<br>Merge all previous data and prepare it for the final report writing agent. |
| total mentions | set | Store `total-mentions` from keyword output | Keyword Agent | Merge2 |  |
| Merge2 | merge | Combine (analysis+attributes) + aggregated arrays + total | Merge3; Aggregate; total mentions | AI Agent | ## Merge<br>Merge all previous data and prepare it for the final report writing agent. |
| AI Agent | langchain.agent | Write final markdown report without altering data | Merge2; (ai_model: Google Gemini Chat Model2) | Create a document | ## Writing agent<br>Writes the final report gathering all the information and insights from previous steps, and structure it in a good format. |
| Create a document | googleDocs | Create Google Doc container | AI Agent | Update a document | ## Document creation<br>Creates a new document and write the content of the final report into it. |
| Update a document | googleDocs | Insert report text into created doc | Create a document | — | ## Document creation<br>Creates a new document and write the content of the final report into it. |
| Google Gemini Chat Model2 | lmChatGoogleGemini | Shared LLM for all AI nodes | — | (as ai_languageModel to multiple nodes) | ## LLM Model |
| olostep | httpRequestTool | Tool endpoint for agents to search/collect data | — | (as ai_tool to agents) | ## Olostep /answer enpoint |
| Sticky Note | stickyNote | Canvas documentation | — | — | # AI Complaint Mining & Insight Extraction… (full note content) |
| Sticky Note1 | stickyNote | Canvas documentation | — | — | ## Trigger… |
| Sticky Note2 | stickyNote | Canvas documentation | — | — | ## Verbatim Quotes Agent… |
| Sticky Note3 | stickyNote | Canvas documentation | — | — | ## Pain Level Identifier… |
| Sticky Note4 | stickyNote | Canvas documentation | — | — | ## Verbatim Quotes Agent… (keywords description) |
| Sticky Note5 | stickyNote | Canvas documentation | — | — | ## LLM Model |
| Sticky Note6 | stickyNote | Canvas documentation | — | — | ## Olostep /answer enpoint |
| Sticky Note7 | stickyNote | Canvas documentation | — | — | ## Merge & Aggregate… |
| Sticky Note8 | stickyNote | Canvas documentation | — | — | ## Analyzer Agent… |
| Sticky Note9 | stickyNote | Canvas documentation | — | — | ## Information Extractor… |
| Sticky Note10 | stickyNote | Canvas documentation | — | — | ## Merge… |
| Sticky Note11 | stickyNote | Canvas documentation | — | — | ## Writing agent… |
| Sticky Note12 | stickyNote | Canvas documentation | — | — | ## Document creation… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1. Add node **Form Trigger**.
   2. Set **Form title** to `Mine Complaint`.
   3. Add a required field labeled **Problem Statement or Keywords**.

2. **Add and configure the shared LLM**
   1. Add node **Google Gemini Chat Model**.
   2. Connect credentials: **Gemini(PaLM) Api** (Google PaLM/Gemini key).
   3. Keep default options unless you need temperature/token tuning.

3. **Add Olostep tool node**
   1. Add node **HTTP Request Tool**.
   2. Method: `POST`
   3. URL: `https://api.olostep.com/v1/answers`
   4. Authentication: **Predefined credential type** → select/create `olostepScrapeApi`.
   5. Enable **Send Body**.
   6. Add body parameter:
      - Name: `task`
      - Value: use `$fromAI(...)` mapping so the agent can pass the query (as in the workflow).
   7. This node will be used as an AI tool by both agents.

4. **Verbatim quote mining branch**
   1. Add **Structured Output Parser** with a schema example like:
      - `{ findings: [{ verbatim_quote, url, source_title }] }`
   2. Add **AI Agent** (LangChain Agent) named *Verbatim Quotes Agent*:
      - Input text: `Problem Statement Or Keywords: {{ $json['Problem Statement or Keywords'] }}`
      - System message: instruct query generation, real-user complaints, include URL/source_title, output ≥10 items, no analysis.
      - Enable Output Parser; attach the structured output parser node.
      - Attach **Google Gemini Chat Model** as its language model.
      - Attach **olostep** node as an available tool.
   3. Add **Wait** node (30 seconds) after the agent.
   4. Add **Split Out** node:
      - Field to split: `output.findings`

5. **Pain level classification**
   1. Add **Chain LLM** node named *Pain Level Identifier*:
      - Input: `list of verbatim quotes: {{ $json.verbatim_quote }}`
      - Prompt: rubric; output must be exactly High/Medium/Low.
      - Attach **Google Gemini Chat Model**.
      - Enable retry (optional) and set max tries (e.g., 2).
   2. Add **Wait** node (30 seconds).
   3. Add **Set** node named *pain level*:
      - Set field `pain-level` = `{{ $json.text }}`
   4. Add **Merge** node:
      - Mode: **Combine**
      - Combine by: **Position**
      - Input 0: *pain level*
      - Input 1: *Split Out* (original quote items)

6. **Keyword mining branch**
   1. Add **Structured Output Parser** for keywords:
      - `{ problem_statement, keywords:[{keyword,mentions}], total_mentions }`
   2. Add **AI Agent** named *Keyword Agent*:
      - Input: `Problem Statement or Keywords: {{ $json['Problem Statement or Keywords'] }}`
      - System message: generate 3–5 queries, search, extract keyword counts, compute total_mentions.
      - Attach **Google Gemini Chat Model**
      - Attach **olostep** tool
      - Enable output parser and attach the keyword parser.
   3. Add **Set** node named *total mentions*:
      - `total-mentions` = `{{ $json.output.total_mentions }}`
   4. Add **Wait** node (30 seconds).
   5. Add **Split Out** node:
      - Field: `output.keywords`

7. **Merge + aggregate for analysis**
   1. Add **Merge** node named *Merge1* to combine:
      - Input 0: quotes+pain stream (from earlier Merge)
      - Input 1: keywords split stream
      - (Keep merge settings consistent with your intended behavior; aggregation later expects all relevant fields to exist across items.)
   2. Add **Aggregate** node:
      - Aggregate fields into arrays: `verbatim_quote`, `source_title`, `keyword`, `mentions`, `pain-level`.

8. **Analysis + extraction**
   1. Add **Chain LLM** named *Analyzer Agent*:
      - Provide the aggregated arrays and include:
        - total mentions from Keyword Agent (expression referencing that node’s output)
      - Prompt: decide high/medium/low demand signal and provide frustration percentage.
      - Attach **Google Gemini Chat Model**.
   2. Add **Information Extractor**:
      - Text: `{{ $json.text }}`
      - Attributes (required):
        - `express frustration` (percentage)
        - `demand signal`
      - Attach **Google Gemini Chat Model**.

9. **Final merging for report**
   1. Add **Merge** node named *Merge3*:
      - Combine-by-position the extractor output with analyzer text.
   2. Add **Merge** node named *Merge2*:
      - Mode: **Combine**
      - Combine by: **Position**
      - Number of inputs: **3**
      - Inputs:
        1) Merge3  
        2) Aggregate  
        3) total mentions

10. **Report writing**
   1. Add **AI Agent** named *AI Agent* (writing agent):
      - Input text should include:
        - extracted attributes (`output['express frustration']`, `output['demand signal']`)
        - analyzer text
        - arrays of quotes/pain/source titles/keywords/mentions
      - System message: do not change data, do not summarize, output markdown.
      - Attach **Google Gemini Chat Model**.

11. **Google Docs publishing**
   1. Add **Google Docs** node *Create a document*:
      - Title: `Complaint Mining Report For: {{ $('On form submission').item.json['Problem Statement or Keywords'] }}`
      - Folder: choose default or a specific folder ID.
      - Connect Google Docs OAuth2 credentials.
   2. Add **Google Docs** node *Update a document*:
      - Operation: **Update**
      - Document URL/ID: `{{ $json.id }}` (from create step)
      - Action: **Insert** text = `{{ $('AI Agent').item.json.output }}`
   3. Connect: Writing agent → Create doc → Update doc.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Complaint Mining & Insight Extraction… (full description of purpose, who it’s for, how it works, requirements, customization ideas)” | Sticky note content at top-left of canvas (workflow documentation) |
| “👉 This template helps you turn complaints into insights — and insights into product improvements.” | Included in the main sticky note content |

