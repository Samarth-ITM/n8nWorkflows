Build person OSINT profiles using Humantic AI, Hunter, CourtListener and GPT-5

https://n8nworkflows.xyz/workflows/build-person-osint-profiles-using-humantic-ai--hunter--courtlistener-and-gpt-5-12507


# Build person OSINT profiles using Humantic AI, Hunter, CourtListener and GPT-5

## 1. Workflow Overview

**Workflow name (in JSON):** *Analyze publicly available information about individuals using AI*  
**User-provided title:** *Build person OSINT profiles using Humantic AI, Hunter, CourtListener and GPT-5*

**Purpose:**  
This workflow builds an OSINT-style profile about a person using public sources and structured enrichment. It combines (1) personality/work-history enrichment (Humantic AI), (2) email discovery (Hunter), (3) broad discovery via multiple public data/search tools (Serper/Google, CourtListener, LegiScan, DocumentCloud, OpenStates, internal Open Paws vector DB), (4) strict identity verification + prioritization, (5) targeted retrieval of full texts, (6) report writing, and (7) hallucination verification + optional correction loop.

**Primary use cases:**
- Generate a detailed intelligence-style report tailored to a user goal (e.g., influence strategy, partnership due diligence, background research).
- Reduce false positives via a separate identity verification agent and a final verification agent.

### 1.1 Input Reception & Normalization
Receives person identifiers and goal either from a Webhook or from another workflow, then normalizes fields into a single item.

### 1.2 Enrichment (Humantic + Hunter)
Creates/refreshes a Humantic profile from LinkedIn URL, waits, fetches profile data, and runs Hunter email finder using company domain + name.

### 1.3 Discovery (Agent)
An LLM agent runs “broad net” discovery using multiple tools (web search, legal/legislative databases, document archives, OpenStates, internal vector DB).

### 1.4 Prioritization & Identity Verification (Agent)
A second LLM agent conservatively selects only items that are **100% confirmed** to match the target person; uses a required Think tool checklist.

### 1.5 Retrieval (Agent)
A third LLM agent fetches full text/content only for the prioritized items (opinions, bills, DocumentCloud text, URLs, social scrapes, OpenStates person detail).

### 1.6 Report Writing (Agent)
A report writer agent synthesizes Humantic + Hunter + retrieved texts into a comprehensive, citation-heavy report.

### 1.7 Verification & Hallucination Repair Loop
A verification LLM checks the report against retrieved documents (very permissive toward analysis/predictions, flags only material fabrications), outputs structured JSON, and conditionally triggers a rewrite agent to fix hallucinations. Includes retry/empty-output guards.

### 1.8 Output Delivery
Depending on input source, either responds to the webhook (streaming) or outputs a final JSON field usable by downstream workflows.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggers & Input Preparation
**Overview:** Accepts inputs via two entry points and normalizes them into consistent fields for downstream nodes.  
**Nodes involved:**  
- Trigger individual research (Webhook)  
- Trigger individual research from another workflow  
- Prepare research input fields

#### Node: **Trigger individual research (Webhook)**
- **Type / role:** `n8n-nodes-base.webhook` — external HTTP entry point.
- **Key configuration:**
  - POST `path: osint-personal-profile`
  - `responseMode: streaming` (workflow can stream response)
- **Outputs:** To **Prepare research input fields**
- **Edge cases / failures:**
  - Missing required body fields → downstream searches degrade.
  - Streaming responses require Respond to Webhook configured correctly later.
- **Version:** 2.1

#### Node: **Trigger individual research from another workflow**
- **Type / role:** `n8n-nodes-base.executeWorkflowTrigger` — internal entry point from parent workflow.
- **Key configuration:**
  - Declares expected inputs: `firstName, lastName, companyName, linkedinURL, reportGoal, companyDomain`
- **Outputs:** To **Prepare research input fields**
- **Edge cases:**
  - Parent workflow passes incomplete fields → profiling accuracy decreases.
- **Version:** 1.1

#### Node: **Prepare research input fields**
- **Type / role:** `n8n-nodes-base.set` — normalize/derive canonical fields.
- **Key configuration (expressions):**
  - `firstName`: `{{$json.firstName || ($json.body && $json.body.firstName) || ''}}`
  - `lastName`: similar pattern
  - `linkedinURL`, `reportGoal`, `companyDomain`: similar pattern
  - **Potential bug:** `companyName` uses `($json.body && $json.body.companyDomain)` as fallback, not `companyName`.
  - `source`: `{{$json.webhookUrl ? 'webhook' : 'workflow'}}`
- **Outputs:** To **Create a profile**
- **Edge cases:**
  - If invoked from execute-workflow trigger, `$json.webhookUrl` typically absent → `source=workflow`.
  - The companyName fallback can silently set companyName to a domain string.
- **Version:** 3.4

---

### Block 2.2 — Humantic Profile Creation + Wait + Retrieval
**Overview:** Initiates a Humantic AI profile build from LinkedIn URL, waits for processing, then retrieves the finished profile.  
**Nodes involved:**  
- Create a profile  
- Wait before next research step  
- Get a profile

#### Node: **Create a profile**
- **Type / role:** `n8n-nodes-base.humanticAi` — create/queue a Humantic user profile.
- **Key configuration:**
  - `userId = {{$json.linkedinURL}}`
  - `onError: continueRegularOutput` (workflow continues even if Humantic fails)
- **Input:** From **Prepare research input fields**
- **Output:** To **Wait before next research step**
- **Credentials:** Humantic AI account
- **Edge cases:**
  - Invalid LinkedIn URL or Humantic API errors → output may be empty; later “Get” may fail too.
- **Version:** 1

#### Node: **Wait before next research step**
- **Type / role:** `n8n-nodes-base.wait` — pauses to allow Humantic profile generation.
- **Key configuration:** 45 seconds
- **Error handling:** `onError: continueRegularOutput`
- **Output:** To **Get a profile**
- **Edge cases:**
  - Humantic profile may still be processing after 45s → subsequent get may return incomplete/empty.
- **Version:** 1.1

#### Node: **Get a profile**
- **Type / role:** `n8n-nodes-base.humanticAi` — fetch Humantic profile.
- **Key configuration:**
  - `operation: get`
  - `userId = {{ $('Prepare research input fields').item.json.linkedinURL }}`
- **Output:** To **Hunter**
- **Credentials:** Humantic AI account
- **Edge cases:**
  - If LinkedIn URL missing → userId empty → request fails; but workflow continues.
- **Version:** 1

---

### Block 2.3 — Hunter Email Finder + Consolidation
**Overview:** Uses company domain + name to find likely email(s) and merges Humantic + Hunter outputs into a unified payload for agents.  
**Nodes involved:**  
- Hunter  
- Set Fields

#### Node: **Hunter**
- **Type / role:** `n8n-nodes-base.hunter` — email finder enrichment.
- **Key configuration:**
  - `operation: emailFinder`
  - `domain, firstname, lastname` from **Prepare research input fields**
- **Input:** From **Get a profile**
- **Output:** To **Set Fields**
- **Credentials:** Hunter account
- **Edge cases:**
  - companyDomain missing/incorrect → no results.
  - Rate limits / quota exceeded → node continues but outputs error payload.
- **Version:** 1

#### Node: **Set Fields**
- **Type / role:** `n8n-nodes-base.set` — attaches enrichment objects.
- **Key configuration:**
  - `Humantic Findings` = full JSON from **Get a profile**
  - `Hunter Findings` = current `$json` (Hunter response)
  - `ignoreConversionErrors: true`
- **Output:** To **Step 1: Discovery**
- **Edge cases:**
  - If Humantic failed, Humantic Findings may be error/empty object.
- **Version:** 3.4

---

### Block 2.4 — Step 1: Discovery (Agent + Tools)
**Overview:** An LLM agent performs comprehensive discovery using multiple tools (web search, CourtListener, LegiScan, DocumentCloud, OpenStates, internal vector DB). Includes retry if tools weren’t used.  
**Nodes involved:**  
- Step 1: Discovery  
- Gemini 2.5 Flash  
- Auto Fallback  
- Google Search Discovery  
- Court Listener Discovery  
- LegiScan Discovery  
- DocumentCloud Discovery  
- Plural Discovery1 (OpenStates people search)  
- Search Open Paws Database2 (Weaviate tool)  
- Embeddings OpenAI2  
- Retry if Tools Not Used

#### Node: **Step 1: Discovery**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — multi-tool discovery agent.
- **Key configuration:**
  - `maxIterations: 10`
  - `returnIntermediateSteps: true` (captures tool calls and results)
  - Prompt includes:
    - normalized identity fields
    - embedded Humantic + Hunter data (truncated to 100k/50k chars)
    - instruction to run all 7 phases and “EVERY search” listed
  - `needsFallback: true`
- **Language models:**
  - Primary: **Gemini 2.5 Flash**
  - Fallback: **OpenRouter auto**
- **Tools connected (AI tool port):**
  - Google Search Discovery (Serper)
  - Court Listener Discovery
  - LegiScan Discovery
  - DocumentCloud Discovery
  - Plural Discovery1 (OpenStates people discovery)
  - Search Open Paws Database2 (Weaviate retrieval tool)
- **Output:** To **Retry if Tools Not Used**
- **Edge cases:**
  - Tool misconfiguration (auth, wrong base URL) → discovery incomplete.
  - The system prompt demands exhaustive searching, which can hit rate limits/quotas.
- **Version:** 2.2

#### Node: **Gemini 2.5 Flash**
- **Type / role:** `lmChatOpenRouter` — model provider for Step 1.
- **Model:** `google/gemini-2.5-flash`, temperature 0.8
- **Edge cases:** model refusal / tool-use limitations handled by fallback.
- **Version:** 1

#### Node: **Auto Fallback**
- **Type / role:** `lmChatOpenRouter` fallback model
- **Model:** `openrouter/auto`, temperature 0.8
- **Version:** 1

#### Node: **Google Search Discovery**
- **Type / role:** `toolHttpRequest` — Serper Google search tool for the agent.
- **Key configuration:**
  - URL: `https://google.serper.dev/{endpoint}`
  - Uses header auth credential (“Serper API”)
  - Content-Type: application/json
  - Placeholder `{endpoint}` must be set by the agent (search/news/scholar/etc.)
- **Edge cases:**
  - Wrong endpoint value → 404.
  - Serper quota/rate limits.
- **Version:** 1.1

#### Node: **Court Listener Discovery**
- **Type / role:** `n8n-nodes-base.httpRequestTool` — CourtListener search tool.
- **Key configuration:**
  - Header auth credential (“CourtListener Key”)
  - `Accept: application/json`
  - Batching enabled (size 1)
  - Tool description includes correct search URL patterns.
- **Edge cases:**
  - Agent may pass partial URLs unless it follows instructions (“ALWAYS include full base URL”).
- **Version:** 4.2

#### Node: **LegiScan Discovery**
- **Type / role:** `httpRequestTool` — LegiScan search tool.
- **Key configuration:**
  - Query auth credential (“Legiscan Key”)
  - Batching size 1
  - Tool description strongly enforces full URL starting with `https://api.legiscan.com/?op=...`
- **Edge cases:** missing base URL is a common failure; description attempts to prevent it.
- **Version:** 4.2

#### Node: **DocumentCloud Discovery**
- **Type / role:** `httpRequestTool` — DocumentCloud search tool.
- **Key configuration:** expects agent to build full search URL under `https://api.www.documentcloud.org/api/documents/search/`
- **Edge cases:** DocumentCloud API may require auth for some endpoints; here none is configured (public searches can still work, but may be limited).
- **Version:** 4.2

#### Node: **Plural Discovery1 (OpenStates People Discovery)**
- **Type / role:** `httpRequestTool` — OpenStates people search tool.
- **Key configuration issues:**
  - Uses **httpHeaderAuth credential named “CourtListener Key”** (likely incorrect for OpenStates).
  - Tool description: base `https://v3.openstates.org/people`
  - Enforces `per_page` 1–20.
- **Edge cases:**
  - Auth mismatch will cause 401/403 if OpenStates key not provided correctly.
- **Version:** 4.2

#### Node: **Search Open Paws Database2**
- **Type / role:** `vectorStoreWeaviate` (retrieve-as-tool) — internal knowledge base lookup.
- **Key configuration:**
  - `mode: retrieve-as-tool`, `topK: 10`
  - Collection: `Content`
  - Requires embeddings via **Embeddings OpenAI2**
- **Edge cases:** Weaviate connectivity/auth issues, missing embedding credentials.
- **Version:** 1.3

#### Node: **Embeddings OpenAI2**
- **Type / role:** `embeddingsOpenAi` — embedding model for Weaviate queries.
- **Credentials:** OpenAI account
- **Edge cases:** model availability/quotas.
- **Version:** 1.2

#### Node: **Retry if Tools Not Used**
- **Type / role:** `n8n-nodes-base.if` — retry guard.
- **Condition logic:**
  - If `$json.intermediateSteps[0]` is empty **AND** `$runIndex < 4`
- **Outputs:**
  - **True path:** loops back to **Step 1: Discovery**
  - **False path:** proceeds to **Step 2: Prioritization**
- **Edge cases:**
  - If agent returns intermediateSteps but tool calls failed silently, this check may not catch quality issues.
- **Version:** 2.2

---

### Block 2.5 — Step 2: Prioritization & Identity Verification
**Overview:** A conservative agent verifies identity match for every discovered item and selects only 100% confirmed items for retrieval. Includes retry if response empty.  
**Nodes involved:**  
- Step 2: Prioritization  
- GPT-5a  
- Auto Fallback1  
- Think Tool Prioritization  
- Retry if Response Empty

#### Node: **Step 2: Prioritization**
- **Type / role:** `langchain.agent` — filters/prioritizes discovery outputs.
- **Key configuration:**
  - `maxIterations: 10`
  - `returnIntermediateSteps: true`
  - Prompt:
    - Includes cleaned/truncated `intermediateSteps` from discovery (dynamic truncation up to 2.4M chars)
    - Includes “Humantic profile” subset (location, work_history, education, skills) — note: it references `$('Prepare research input fields').item.json['Humantic Findings']` which is likely **not present** there; Humantic Findings actually exists on **Set Fields** output. This can reduce identity context.
  - System message enforces: **ONLY 100% CONFIRMED**
  - Requires **Think Tool** for each item.
- **Models:**
  - Primary: **GPT-5a** (openai/gpt-5, temp 0.2)
  - Fallback: **Auto Fallback1** (openrouter/auto)
- **Tool:** Think Tool Prioritization
- **Output:** To **Retry if Response Empty**
- **Edge cases:**
  - Wrong reference to Humantic Findings can weaken verification.
- **Version:** 2.2

#### Node: **Think Tool Prioritization**
- **Type / role:** `toolThink` — forces checklist-style reasoning for identity confirmation.
- **Used by:** Step 2 agent
- **Version:** 1

#### Node: **Retry if Response Empty**
- **Type / role:** `if` — retry guard.
- **Condition:** `$json.output` empty AND `$runIndex < 4`
- **Outputs:**
  - True: loops back to **Step 2**
  - False: proceeds to **Step 3: Retrieval**
- **Version:** 2.2

#### Node: **GPT-5a**
- **Type / role:** model provider for Step 2
- **Model:** `openai/gpt-5`, temperature 0.2
- **Version:** 1

#### Node: **Auto Fallback1**
- **Type / role:** fallback model provider for Step 2
- **Model:** `openrouter/auto`, temperature 0.2
- **Version:** 1

---

### Block 2.6 — Step 3: Retrieval (Agent + Tools)
**Overview:** Retrieves full content only for prioritized items using specialized retrieval tools. Includes retry if no tools used.  
**Nodes involved:**  
- Step 3: Retrieval  
- Gemini 2.5 Flash2  
- Auto Fallback2  
- DocumentCloud Retrieval  
- LegiScan Retrieval  
- Court Listener Retrieveal  
- Jina URL Text Extraction  
- Linkedin Person and Company Scraper1  
- Instagram Profile Scraper1  
- Twitter Profile Scraper1  
- Plural Retrieval  
- Retry if Tools Not Used1

#### Node: **Step 3: Retrieval**
- **Type / role:** `langchain.agent` — performs targeted retrieval.
- **Key configuration:**
  - `maxIterations: 10`
  - Prompt includes prioritization agent output (truncated to 500k chars)
  - System message: “NO NEW SEARCHES”, retrieve selected items only.
  - `returnIntermediateSteps: true`
- **Models:**
  - Primary: Gemini 2.5 Flash2 (temp 0.2)
  - Fallback: Auto Fallback2
- **Tools connected:** all retrieval tools listed below
- **Output:** To **Retry if Tools Not Used1**
- **Version:** 2.2

#### Node: **DocumentCloud Retrieval**
- **Type / role:** `httpRequestTool` — fetches `.../documents/{id}/{slug}.txt.json` from DocumentCloud S3.
- **Key configuration:** URL is AI-supplied, batching size 1.
- **Edge cases:** slug/id mismatch → 404; documents may not have S3 text JSON available.
- **Version:** 4.2

#### Node: **LegiScan Retrieval**
- **Type / role:** `httpRequestTool` — fetches full bill metadata/text.
- **Key configuration:** query auth; AI provides URL.
- **Edge cases:** bill text requires subsequent `getBillText` doc_id calls; agent must chain them.
- **Version:** 4.2

#### Node: **Court Listener Retrieveal**
- **Type / role:** `httpRequestTool` — fetches full opinion text by ID with correct trailing slash.
- **Key configuration:** header auth; AI provides URL; emphasizes `absolute_url` inclusion.
- **Edge cases:** common URL formatting mistakes; tool description attempts to prevent them.
- **Version:** 4.2

#### Node: **Jina URL Text Extraction**
- **Type / role:** `n8n-nodes-base.jinaAiTool` — extracts readable article content from a URL.
- **Credentials:** Jina AI account
- **Edge cases:** paywalled sites, bot protections, very large pages.
- **Version:** 1

#### Node: **Linkedin Person and Company Scraper1**
- **Type / role:** `toolHttpRequest` — ScrapingDog LinkedIn scraper.
- **Key configuration:**
  - URL: `https://api.scrapingdog.com/linkedin`
  - Query params: `linkId`, `type` (profile/company)
  - Auth: query auth credential (“ScrapingDog Key”)
- **Edge cases:** LinkedIn anti-bot changes; scraper failures; rate limiting.
- **Version:** 1.1

#### Node: **Instagram Profile Scraper1**
- **Type / role:** `toolHttpRequest` — ScrapingDog Instagram profile.
- **URL:** `https://api.scrapingdog.com/instagram/profile?username={username}`
- **Edge cases:** private accounts, rate limiting.
- **Version:** 1.1

#### Node: **Twitter Profile Scraper1**
- **Type / role:** `toolHttpRequest` — ScrapingDog X/Twitter profile.
- **URL:** `http://api.scrapingdog.com/x/profile?profileId={profileId}&parsed=true`
- **Edge cases:** HTTP (not HTTPS) endpoint may be blocked in some environments; X restrictions.
- **Version:** 1.1

#### Node: **Plural Retrieval**
- **Type / role:** `httpRequestTool` — OpenStates person detail retrieval.
- **Key configuration issue:**
  - Uses httpHeaderAuth credential named “CourtListener Key” (likely wrong).
- **Edge cases:** 401/403 if OpenStates key not correctly supplied.
- **Version:** 4.2

#### Node: **Retry if Tools Not Used1**
- **Type / role:** `if` retry guard
- **Condition:** `$json.intermediateSteps[0]` empty AND `$runIndex < 4`
- **Outputs:**
  - True: loops to Step 3
  - False: proceeds to Step 4
- **Version:** 2.2

#### Node: **Gemini 2.5 Flash2 / Auto Fallback2**
- **Type / role:** model providers for Step 3
- **Models:** `google/gemini-2.5-flash` (temp 0.2), `openrouter/auto` (temp 0.2)
- **Version:** 1

---

### Block 2.7 — Step 4: Report Writing (Agent)
**Overview:** Produces the final long-form OSINT report with citations, using Humantic + Hunter + retrieved tool results. Includes retry on empty output.  
**Nodes involved:**  
- Step 4: Report Writing  
- GPT-5b  
- Auto Fallback3  
- Think Tool Analysis  
- Retry if Response Empty1

#### Node: **Step 4: Report Writing**
- **Type / role:** `langchain.agent` — report generation.
- **Key configuration:**
  - `maxIterations: 10`
  - Uses Humantic Findings and Hunter Findings from **Set Fields**
  - Injects retrieved intermediateSteps from Step 3 with generous 1.8M char budget and dynamic truncation.
  - System message heavily enforces:
    - no follow-up questions
    - exhaustive detail
    - inline citations with full URLs
  - `returnIntermediateSteps: true`
- **Models:**
  - Primary: GPT-5b (openai/gpt-5, temp 0.6)
  - Fallback: Auto Fallback3 (openrouter/auto, temp 0.6)
- **Tool:** Think Tool Analysis (provided but not strictly required by system message)
- **Output:** To **Retry if Response Empty1**
- **Version:** 2.2

#### Node: **Retry if Response Empty1**
- **Type / role:** `if` — retry guard
- **Condition:** `$json.output` empty AND `$runIndex < 4`
- **Outputs:**
  - True: loops back to Step 4
  - False: goes to Set Report
- **Version:** 2.2

#### Node: **Think Tool Analysis**
- **Type / role:** `toolThink` — analysis guidance tool (synthesis checklist).
- **Used by:** Step 4 agent (tool port)
- **Version:** 1

#### Node: **GPT-5b / Auto Fallback3**
- **Type / role:** model providers
- **Models:** `openai/gpt-5` (temp 0.6), `openrouter/auto` (temp 0.6)
- **Version:** 1

---

### Block 2.8 — Packaging for Verification
**Overview:** Converts the generated report and all supporting documents into fields used by the verification agent.  
**Nodes involved:**  
- Set Report

#### Node: **Set Report**
- **Type / role:** `set` — builds `Final Report` and `Retrieved Documents` strings.
- **Key configuration:**
  - `Final Report` = `{{$json.output}}` (from Step 4)
  - `Retrieved Documents` is a large concatenated text blob containing:
    - HUMANTIC FINDINGS (truncated at 200k chars)
    - HUNTER FINDINGS (truncated at 30k chars)
    - “RETRIEVED TEXTS FROM PREVIOUS AGENT” from `$('Step 3: Retrieval').item.json.intermediateSteps` (truncated at 320k chars)
- **Output:** To **Step 5: Verification**
- **Edge cases:**
  - If Step 3 is missing/failed, the retrieval section becomes “No retrieved texts available”.
  - Truncation warnings indicate possible verification blind spots.
- **Version:** 3.4

---

### Block 2.9 — Step 5: Verification (LLM Chain + Structured Parser)
**Overview:** Checks the final report against retrieved documents and returns a structured JSON list of material fabrications (ideally 0–2). Includes retry guards and conditional hallucination repair.  
**Nodes involved:**  
- Step 5: Verification  
- GPT-5c  
- Auto Fallback4  
- Structured Output Parser  
- If Empty Output1  
- If hallucinations present

#### Node: **Step 5: Verification**
- **Type / role:** `chainLlm` — verification agent with strict output expectations.
- **Key configuration:**
  - Large prompt includes:
    - permissive guidance: do not flag analysis/predictions; only material false factual claims
    - embedded “Retrieved Documents” (up to 2.4M chars with truncation logic)
    - embedded “Final Report” (up to 800k chars with truncation logic)
  - `needsFallback: true`, `hasOutputParser: true`
- **Models:**
  - Primary: GPT-5c (openai/gpt-5, temp 0.2)
  - Fallback: Auto Fallback4 (openrouter/auto, temp 0.2) also connected to the output parser model input
- **Output:** to **If Empty Output1**
- **Version:** 1.7

#### Node: **Structured Output Parser**
- **Type / role:** `outputParserStructured` — enforces JSON schema and auto-fixes.
- **Schema:** `{ contains_hallucinations: boolean, hallucinations: [...], summary: string }` with typed enums.
- **AutoFix:** true (attempts to repair slightly invalid JSON)
- **Input:** from Step 5 (ai_outputParser connection)
- **Edge cases:** If model returns non-JSON or refuses, autofix may fail; downstream conditions can break.
- **Version:** 1.3

#### Node: **If Empty Output1**
- **Type / role:** `if` — guard for empty verification output.
- **Condition (loose validation):** `$json.output` empty (object empty)  
- **Outputs:**
  - True → loops back to **Step 5: Verification** and also to **If hallucinations present** (based on wiring)
  - False → proceeds to **If hallucinations present**
- **Edge cases:** Because typeValidation is loose, malformed outputs may pass/loop unexpectedly.
- **Version:** 2.2

#### Node: **If hallucinations present**
- **Type / role:** `if` — decides whether to invoke hallucination-fixing agent.
- **Condition:** `{{$json.output.contains_hallucinations}}` is true AND `$runIndex < 4`
- **Outputs:**
  - True → **Step 6: Fixing Hallucinations**
  - False → **Set Output**
- **Version:** 2.2

---

### Block 2.10 — Step 6: Fixing Hallucinations (Agent + Memory + Guards)
**Overview:** If verification found hallucinations, rewrites the report *surgically* to fix only flagged issues, preserving analysis and Humantic-based insights. Includes memory and an empty-output guard that can route back to verification.  
**Nodes involved:**  
- Step 6: Fixing Hallucinations  
- GPT-5d  
- Auto Fallback5  
- Simple Memory6  
- Think Tool Analysis2  
- If Empty Output  
- Think Tool Analysis2 (tool)  
- If Empty Output (guard)

#### Node: **Step 6: Fixing Hallucinations**
- **Type / role:** `langchain.agent` — rewrite agent.
- **Key configuration:**
  - Prompt includes:
    - user original query from `Trigger individual research from another workflow` (`prompt` field) — may be empty if webhook path used (webhook payload uses `reportGoal`).
    - retrieved docs: JSON of Step 3 intermediateSteps (up to 1.6M chars)
    - failed report: from Set Report Final Report (up to 800k chars)
    - hallucinations: structured list (up to 80k chars)
  - System message: minimal intervention; preserve structure; fix only flagged issues; maintain citations.
  - `returnIntermediateSteps: true`
- **Models:**
  - Primary: GPT-5d (openai/gpt-5, temp 0.1)
  - Fallback: Auto Fallback5 (openrouter/auto, temp 0.1)
- **Memory:** Simple Memory6 attached (session-based)
- **Tool:** Think Tool Analysis2 attached
- **Output:** To **If Empty Output**
- **Edge cases:**
  - If running from webhook, “User’s Original Query” reference may be missing.
- **Version:** 2.2

#### Node: **Simple Memory6**
- **Type / role:** `memoryBufferWindow` — conversation memory buffer.
- **Key configuration:**
  - `sessionKey = {{ $('Trigger individual research from another workflow').item.json.sessionId }}d`
  - This may be undefined if no sessionId is provided by parent workflow.
- **Used by:** Step 6 agent
- **Edge cases:** missing sessionId → memory key becomes `"undefinedd"` causing cross-run contamination.
- **Version:** 1.3

#### Node: **Think Tool Analysis2**
- **Type / role:** `toolThink` — synthesis checklist (same content as Think Tool Analysis).
- **Used by:** Step 6 agent
- **Version:** 1

#### Node: **If Empty Output**
- **Type / role:** `if` — guard after hallucination fixing.
- **Condition:** `$json.output` empty (string empty)
- **Outputs:**
  - True → routes to **Step 6: Fixing Hallucinations** and also to **Step 5: Verification** (per wiring)
  - False → routes to **Step 5: Verification** (to re-check)  
- **Edge cases:** can create loops if model repeatedly outputs empty.
- **Version:** 2.2

---

### Block 2.11 — Final Output Routing (Webhook vs Workflow)
**Overview:** Outputs results either via a streaming webhook response or as a normal workflow output.  
**Nodes involved:**  
- Set Output  
- If Source is Webhook  
- Respond to Webhook  
- Final Output

#### Node: **Set Output**
- **Type / role:** `set` — finalizes output field.
- **Key configuration:**
  - Sets `Final Report` = `{{ $('Set Report').item.json['Final Report'] }}`
  - Note: this uses Set Report’s field (not necessarily Step 6 corrected output unless Set Report is updated upstream; in the current wiring, Step 6 loops back through verification and Set Report, so it can be updated indirectly).
- **Output:** To **If Source is Webhook**
- **Version:** 3.4

#### Node: **If Source is Webhook**
- **Type / role:** `if` — route based on input source.
- **Condition:** `{{ $('Prepare research input fields').item.json.source }} == 'webhook'`
- **Outputs:**
  - True → **Respond to Webhook**
  - False → **Final Output**
- **Version:** 2.3

#### Node: **Respond to Webhook**
- **Type / role:** `respondToWebhook` — sends HTTP response.
- **Key configuration:**
  - `respondWith: json`
  - `enableStreaming: true`
  - Response body: `={{ JSON.stringify($json) }}`
- **Edge cases:** streaming + large payload may hit gateway limits.
- **Version:** 1.5

#### Node: **Final Output**
- **Type / role:** `set` — creates a simplified output object for non-webhook callers.
- **Key configuration:** sets `report = {{ $('Set Report').item.json['Final Report'] }}`
- **Version:** 3.4

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger individual research (Webhook) | Webhook | External entry point | — | Prepare research input fields | # Analyze publicly available information about individuals using AI / How it works / Setup steps |
| Trigger individual research from another workflow | Execute Workflow Trigger | Internal entry point | — | Prepare research input fields | # Analyze publicly available information about individuals using AI / How it works / Setup steps |
| Prepare research input fields | Set | Normalize input schema | Trigger individual research (Webhook); Trigger individual research from another workflow | Create a profile | ## 📥 Input Schema (Required fields list) |
| Create a profile | Humantic AI | Create/queue Humantic profile by LinkedIn URL | Prepare research input fields | Wait before next research step |  |
| Wait before next research step | Wait | Delay for Humantic processing | Create a profile | Get a profile |  |
| Get a profile | Humantic AI | Fetch Humantic profile | Wait before next research step | Hunter |  |
| Hunter | Hunter | Email finder enrichment | Get a profile | Set Fields |  |
| Set Fields | Set | Merge Humantic + Hunter findings | Hunter | Step 1: Discovery |  |
| Step 1: Discovery | LangChain Agent | Broad OSINT discovery via tools | Set Fields | Retry if Tools Not Used | ## 🟤 Step 1: Discovery |
| Gemini 2.5 Flash | OpenRouter Chat Model | Primary model for Step 1 | — (AI) | Step 1: Discovery (ai_languageModel) | ## 🟤 Step 1: Discovery |
| Auto Fallback | OpenRouter Chat Model | Fallback model for Step 1 | — (AI) | Step 1: Discovery (ai_languageModel) | ## 🟤 Step 1: Discovery |
| Google Search Discovery | LangChain Tool (HTTP) | Serper Google search tool | — (AI tool) | Step 1: Discovery | ## 🟤 Step 1: Discovery |
| Court Listener Discovery | HTTP Request Tool | CourtListener discovery search | — (AI tool) | Step 1: Discovery | ## 🟤 Step 1: Discovery |
| LegiScan Discovery | HTTP Request Tool | Legislation discovery | — (AI tool) | Step 1: Discovery | ## 🟤 Step 1: Discovery |
| DocumentCloud Discovery | HTTP Request Tool | DocumentCloud search discovery | — (AI tool) | Step 1: Discovery | ## 🟤 Step 1: Discovery |
| Plural Discovery1 | HTTP Request Tool | OpenStates people discovery | — (AI tool) | Step 1: Discovery | ## 🟤 Step 1: Discovery |
| Embeddings OpenAI2 | OpenAI Embeddings | Embeddings for Weaviate retrieval | — | Search Open Paws Database2 (ai_embedding) | ### Please refer [Open Paws Guide](https://github.com/Open-Paws/documentation/tree/main/Knowledge) to know how to use our open-source vector database |
| Search Open Paws Database2 | Weaviate Vector Store Tool | Internal KB retrieval | — (AI tool) | Step 1: Discovery | ### Please refer [Open Paws Guide](https://github.com/Open-Paws/documentation/tree/main/Knowledge) to know how to use our open-source vector database |
| Retry if Tools Not Used | If | Retry Step 1 if no tool steps | Step 1: Discovery | Step 1: Discovery; Step 2: Prioritization | ## 🟤 Step 1: Discovery |
| Step 2: Prioritization | LangChain Agent | Strict identity verification + selection | Retry if Tools Not Used | Retry if Response Empty | ## 🟤 Step 2: Prioritization |
| GPT-5a | OpenRouter Chat Model | Primary model for Step 2 | — (AI) | Step 2: Prioritization | ## 🟤 Step 2: Prioritization |
| Auto Fallback1 | OpenRouter Chat Model | Fallback model for Step 2 | — (AI) | Step 2: Prioritization | ## 🟤 Step 2: Prioritization |
| Think Tool Prioritization | LangChain Think Tool | Checklist reasoning for identity match | — (AI tool) | Step 2: Prioritization | ## 🟤 Step 2: Prioritization |
| Retry if Response Empty | If | Retry Step 2 if output empty | Step 2: Prioritization | Step 2: Prioritization; Step 3: Retrieval | ## 🟤 Step 2: Prioritization |
| Step 3: Retrieval | LangChain Agent | Retrieve full texts for selected items | Retry if Response Empty | Retry if Tools Not Used1 | ## 🟤 Step 3: Retrieval |
| Gemini 2.5 Flash2 | OpenRouter Chat Model | Primary model for Step 3 | — (AI) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Auto Fallback2 | OpenRouter Chat Model | Fallback model for Step 3 | — (AI) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| DocumentCloud Retrieval | HTTP Request Tool | Fetch DocumentCloud S3 text JSON | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| LegiScan Retrieval | HTTP Request Tool | Fetch bill + bill text | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Court Listener Retrieveal | HTTP Request Tool | Fetch full opinion text | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Jina URL Text Extraction | Jina AI Tool | Extract article content | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Linkedin Person and Company Scraper1 | LangChain Tool (HTTP) | Scrape LinkedIn profile/company | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Instagram Profile Scraper1 | LangChain Tool (HTTP) | Scrape Instagram profile | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Twitter Profile Scraper1 | LangChain Tool (HTTP) | Scrape X/Twitter profile | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Plural Retrieval | HTTP Request Tool | OpenStates person detail retrieval | — (AI tool) | Step 3: Retrieval | ## 🟤 Step 3: Retrieval |
| Retry if Tools Not Used1 | If | Retry Step 3 if no tool steps | Step 3: Retrieval | Step 3: Retrieval; Step 4: Report Writing | ## 🟤 Step 3: Retrieval |
| Step 4: Report Writing | LangChain Agent | Generate final OSINT report | Retry if Tools Not Used1 | Retry if Response Empty1 | ## ✍️ Writing Report Agent |
| GPT-5b | OpenRouter Chat Model | Primary model for Step 4 | — (AI) | Step 4: Report Writing | ## ✍️ Writing Report Agent |
| Auto Fallback3 | OpenRouter Chat Model | Fallback model for Step 4 | — (AI) | Step 4: Report Writing | ## ✍️ Writing Report Agent |
| Think Tool Analysis | LangChain Think Tool | Synthesis guidance tool | — (AI tool) | Step 4: Report Writing | ## ✍️ Writing Report Agent |
| Retry if Response Empty1 | If | Retry Step 4 if output empty | Step 4: Report Writing | Step 4: Report Writing; Set Report | ## ✍️ Writing Report Agent |
| Set Report | Set | Package report + retrieved docs for verification | Retry if Response Empty1 | Step 5: Verification | ## 🔍 Verifying Report Agent |
| Step 5: Verification | LangChain LLM Chain | Detect material fabrications | Set Report | If Empty Output1 | ## 🔍 Verifying Report Agent |
| GPT-5c | OpenRouter Chat Model | Primary model for Step 5 | — (AI) | Step 5: Verification | ## 🔍 Verifying Report Agent |
| Auto Fallback4 | OpenRouter Chat Model | Fallback model for Step 5 / parser | — (AI) | Step 5: Verification; Structured Output Parser | ## 🔍 Verifying Report Agent |
| Structured Output Parser | Structured Output Parser | Enforce verification JSON schema | — (AI) | Step 5: Verification (ai_outputParser) | ## 🔍 Verifying Report Agent |
| If Empty Output1 | If | Retry verification on empty object | Step 5: Verification | Step 5: Verification; If hallucinations present | ## 🔍 Verifying Report Agent |
| If hallucinations present | If | Route to fix agent if hallucinations | If Empty Output1 | Step 6: Fixing Hallucinations; Set Output | ## 🧯 Fixing Hallucinations Agent |
| Step 6: Fixing Hallucinations | LangChain Agent | Rewrite report to fix flagged issues | If hallucinations present; If Empty Output | If Empty Output | ## 🧯 Fixing Hallucinations Agent |
| GPT-5d | OpenRouter Chat Model | Primary model for Step 6 | — (AI) | Step 6: Fixing Hallucinations | ## 🧯 Fixing Hallucinations Agent |
| Auto Fallback5 | OpenRouter Chat Model | Fallback model for Step 6 | — (AI) | Step 6: Fixing Hallucinations | ## 🧯 Fixing Hallucinations Agent |
| Simple Memory6 | Memory Buffer Window | Session memory for Step 6 | — | Step 6: Fixing Hallucinations (ai_memory) | ## 🧯 Fixing Hallucinations Agent |
| Think Tool Analysis2 | LangChain Think Tool | Synthesis guidance for Step 6 | — (AI tool) | Step 6: Fixing Hallucinations | ## 🧯 Fixing Hallucinations Agent |
| If Empty Output | If | Guard if Step 6 output empty | Step 6: Fixing Hallucinations | Step 6: Fixing Hallucinations; Step 5: Verification | ## 🧯 Fixing Hallucinations Agent |
| Set Output | Set | Prepare final response payload | If hallucinations present | If Source is Webhook |  |
| If Source is Webhook | If | Route webhook vs workflow output | Set Output | Respond to Webhook; Final Output |  |
| Respond to Webhook | Respond to Webhook | Send streaming HTTP JSON response | If Source is Webhook | — |  |
| Final Output | Set | Final output field for workflow callers | If Source is Webhook | — |  |
| Sticky Note3 | Sticky Note | Documentation | — | — | # Analyze publicly available information about individuals using AI (content) |
| Sticky Note5 | Sticky Note | Input schema documentation | — | — | ## 📥 Input Schema (content) |
| Sticky Note7 | Sticky Note | External resource link | — | — | ### Please refer [Open Paws Guide](https://github.com/Open-Paws/documentation/tree/main/Knowledge) to know how to use our open-source vector database |
| Sticky Note8 | Sticky Note | Block annotation | — | — | ## 🟤 Step 1: Discovery (content) |
| Sticky Note9 | Sticky Note | Block annotation | — | — | ## 🟤 Step 2: Prioritization (content) |
| Sticky Note10 | Sticky Note | Block annotation | — | — | ## 🟤 Step 3: Retrieval (content) |
| Sticky Note11 | Sticky Note | Credentials note | — | — | ## 🔐 Credentials Required (content) |
| Sticky Note13 | Sticky Note | Block annotation | — | — | ## ✍️ Writing Report Agent (content) |
| Sticky Note14 | Sticky Note | Block annotation | — | — | ## 🔍 Verifying Report Agent (content) |
| Sticky Note15 | Sticky Note | Block annotation | — | — | ## 🧯 Fixing Hallucinations Agent (content) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named appropriately (e.g., “Analyze publicly available information about individuals using AI”).  
2. **Add Trigger A (Webhook)**  
   - Node: *Webhook*  
   - Method: POST  
   - Path: `osint-personal-profile`  
   - Response mode: *Streaming*  
3. **Add Trigger B (Execute Workflow Trigger)**  
   - Node: *Execute Workflow Trigger*  
   - Add workflow inputs: `firstName,lastName,companyName,linkedinURL,reportGoal,companyDomain`  
4. **Add “Prepare research input fields” (Set node)**  
   - Create fields using expressions:
     - `firstName`, `lastName`, `linkedinURL`, `reportGoal`, `companyDomain` from either `$json` or `$json.body`
     - `source = {{$json.webhookUrl ? 'webhook' : 'workflow'}}`
     - Fix recommended: set `companyName` fallback to `($json.body && $json.body.companyName)` rather than companyDomain.
   - Connect both triggers → this Set node.

5. **Add Humantic enrichment**
   - Node: *Humantic AI* “Create a profile”
     - userId: `={{ $json.linkedinURL }}`
     - On Error: *Continue*
     - Credential: Humantic AI API
   - Node: *Wait*
     - 45 seconds
     - On Error: *Continue*
   - Node: *Humantic AI* “Get a profile”
     - Operation: *Get*
     - userId: `={{ $('Prepare research input fields').item.json.linkedinURL }}`
     - On Error: *Continue*
   - Connect: Prepare → Create → Wait → Get

6. **Add Hunter enrichment**
   - Node: *Hunter* “Hunter”
     - Operation: Email Finder
     - domain: `={{ $('Prepare research input fields').item.json.companyDomain }}`
     - firstname/lastname from Prepare
     - On Error: *Continue*
     - Credential: Hunter API key
   - Connect: Get a profile → Hunter

7. **Add “Set Fields” (Set node)**
   - Add object fields:
     - `Humantic Findings = {{ $('Get a profile').item.json }}`
     - `Hunter Findings = {{ $json }}`
   - Connect: Hunter → Set Fields

8. **Add Step 1: Discovery (LangChain Agent)**
   - Node: *AI Agent* (LangChain Agent)
   - Enable: `Return Intermediate Steps = true`
   - Max iterations: 10
   - Prompt: include identity fields + embedded Humantic/Hunter blobs (with truncation as desired)
   - System message: the 7-phase exhaustive discovery instruction
   - **Attach models:**
     - Add OpenRouter Chat Model node “Gemini 2.5 Flash” (model `google/gemini-2.5-flash`, temp 0.8) → connect to agent language model port (index 0).
     - Add OpenRouter Chat Model node “Auto Fallback” (model `openrouter/auto`, temp 0.8) → connect as fallback (index 1).
   - **Attach tools to agent (AI tool port):**
     - Serper toolHttpRequest (`https://google.serper.dev/{endpoint}`) with header auth credential
     - CourtListener discovery httpRequestTool with header auth credential + Accept header
     - LegiScan discovery httpRequestTool with query auth credential
     - DocumentCloud discovery httpRequestTool (public search)
     - OpenStates people discovery httpRequestTool (fix credential to OpenStates key)
     - Weaviate vector tool (retrieve-as-tool) + OpenAI embeddings node
   - Connect: Set Fields → Step 1

9. **Add “Retry if Tools Not Used” (IF)**
   - Condition: `$json.intermediateSteps[0]` is empty AND `$runIndex < 4`
   - True → loop back to Step 1
   - False → Step 2

10. **Add Step 2: Prioritization (LangChain Agent)**
   - Configure strict identity-verification system message
   - Return intermediate steps = true
   - Attach model nodes:
     - “GPT-5a” OpenRouter model `openai/gpt-5` temp 0.2 (primary)
     - “Auto Fallback1” `openrouter/auto` temp 0.2 (fallback)
   - Attach Think tool node “Think Tool Prioritization”
   - Connect: Retry node (false) → Step 2

11. **Add “Retry if Response Empty” (IF)**
   - Condition: `$json.output` is empty AND `$runIndex < 4`
   - True → Step 2
   - False → Step 3

12. **Add Step 3: Retrieval (LangChain Agent)**
   - System message: retrieve only selected items
   - Return intermediate steps = true
   - Attach models:
     - “Gemini 2.5 Flash2” (temp 0.2)
     - “Auto Fallback2” (temp 0.2)
   - Attach retrieval tools:
     - DocumentCloud Retrieval (S3 txt.json)
     - LegiScan Retrieval
     - CourtListener Retrieval
     - Jina URL extraction tool (Jina credential)
     - ScrapingDog LinkedIn/Instagram/X scrapers (ScrapingDog query auth)
     - OpenStates person retrieval (fix credential)
   - Connect: Retry-if-empty (false) → Step 3

13. **Add “Retry if Tools Not Used1” (IF)**
   - Condition: `$json.intermediateSteps[0]` empty AND `$runIndex < 4`
   - True → Step 3
   - False → Step 4

14. **Add Step 4: Report Writing (LangChain Agent)**
   - Prompt: includes report goal, Humantic Findings, Hunter Findings, retrieved texts
   - System message: citation rules + “no follow-up”
   - Return intermediate steps = true
   - Attach models:
     - “GPT-5b” (`openai/gpt-5`, temp 0.6)
     - “Auto Fallback3” (`openrouter/auto`, temp 0.6)
   - Attach Think tool “Think Tool Analysis”
   - Connect: Retry-if-tools-not-used1 (false) → Step 4

15. **Add “Retry if Response Empty1” (IF)**
   - Condition: `$json.output` empty AND `$runIndex < 4`
   - True → Step 4
   - False → Set Report

16. **Add “Set Report” (Set node)**
   - Create:
     - `Final Report = {{$json.output}}`
     - `Retrieved Documents` = concatenation of Humantic/Hunter + Step 3 intermediateSteps (with truncation)
   - Connect: Retry-if-empty1 (false) → Set Report

17. **Add Step 5: Verification (Chain LLM) + Structured Parser**
   - Node: Chain LLM “Step 5: Verification”
   - Prompt: permissive hallucination detection + includes Retrieved Documents + Final Report with truncation logic
   - Attach model nodes:
     - “GPT-5c” (`openai/gpt-5`, temp 0.2)
     - “Auto Fallback4” (`openrouter/auto`, temp 0.2)
   - Add “Structured Output Parser” with the provided JSON schema, AutoFix enabled
   - Connect parser to chain’s output parser port.

18. **Add “If Empty Output1” (IF)**
   - If `$json.output` empty → loop to Step 5 (optional) else continue.

19. **Add “If hallucinations present” (IF)**
   - If `output.contains_hallucinations == true` AND `$runIndex < 4` → Step 6  
   - Else → Set Output

20. **Add Step 6: Fixing Hallucinations (LangChain Agent)**
   - Prompt: original query + retrieved docs + failed report + hallucinations list
   - Attach:
     - “GPT-5d” (`openai/gpt-5`, temp 0.1)
     - “Auto Fallback5” (`openrouter/auto`, temp 0.1)
     - Memory node (ensure sessionId is provided, or remove memory)
     - Think tool “Think Tool Analysis2”
   - Add “If Empty Output” guard and route back to Step 6 and/or Step 5 for re-verification.

21. **Add output routing**
   - “Set Output” (Set node): set `Final Report` from Set Report (or from the corrected report path if you prefer explicit wiring)
   - “If Source is Webhook” checks Prepare.source
   - True → Respond to Webhook (streaming JSON)
   - False → “Final Output” Set node (sets `report` field)

### Credential setup checklist
- **Humantic AI**: `humanticAiApi`
- **Hunter**: `hunterApi`
- **OpenRouter**: `openRouterApi` (used by all GPT-5/Gemini/OpenRouter model nodes)
- **Serper**: `httpHeaderAuth` with Serper key in header (commonly `X-API-KEY`)
- **CourtListener**: `httpHeaderAuth` (CourtListener token header as required by their API)
- **LegiScan**: `httpQueryAuth` (key in querystring)
- **Weaviate**: `weaviateApi`
- **OpenAI embeddings**: `openAiApi`
- **Jina**: `jinaAiApi`
- **ScrapingDog**: `httpQueryAuth`
- **OpenStates**: should have its own header auth credential (do not reuse CourtListener key)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow researches an individual using publicly available information… includes validation steps… result is a clear Markdown report…” | Sticky note: workflow description block |
| Input fields required: `firstName,lastName,companyName,companyDomain,linkedinURL,reportGoal` | Sticky note: input schema block |
| Open Paws vector database usage guide | https://github.com/Open-Paws/documentation/tree/main/Knowledge |
| Credentials list note (OpenRouter, Serper, LegiScan, CourtListener, OpenCorporates, Jina, ScrapingDog, BuiltWith) | Sticky note: credentials required (note: OpenCorporates and BuiltWith are mentioned but not present as nodes in this workflow) |

---

**Disclaimer (provided by developer):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.