Research organizations with GPT‑5, Gemini, CourtListener, LegiScan and OSINT web sources

https://n8nworkflows.xyz/workflows/research-organizations-with-gpt-5--gemini--courtlistener--legiscan-and-osint-web-sources-12506


# Research organizations with GPT‑5, Gemini, CourtListener, LegiScan and OSINT web sources

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Research organizations with GPT‑5, Gemini, CourtListener, LegiScan and OSINT web sources  
Workflow reference for: **“Research organizations using AI with public data sources and report generation”**

---

## 1. Workflow Overview

This workflow performs **automated OSINT research on an organization** (company name + primary domain + report goal), using a **multi-agent LLM pipeline** with **public data sources** (web search, corporate registries, court opinions, legislation, document repositories, and web page text extraction). It outputs a **Markdown intelligence report**, runs a **verification pass** to detect unsupported factual claims, and (if needed) triggers a **hallucination-fixing rewrite** before returning a final result.

### 1.1 Input Reception & Normalization
Accepts inputs either from:
- a **Webhook** endpoint (`POST /org-osint`), or
- an **Execute Workflow Trigger** (called by another workflow)

Then normalizes input fields into a consistent internal format (`companyName`, `companyDomain`, `reportGoal`, `source`).

### 1.2 Discovery (Broad OSINT Collection)
An LLM “Discovery Agent” uses multiple tools to discover relevant assets and sources:
- Google/Serper web search
- CourtListener search
- LegiScan search
- DocumentCloud search
- OpenCorporates search
- Internal vector DB (Open Paws / Weaviate)

### 1.3 Prioritization & Verification (Select Only Confirmed Items)
A second LLM agent verifies which discovered items are **100% confirmed** to belong to the target organization and produces a structured list of **items selected for retrieval** with the exact IDs/URLs needed downstream.

### 1.4 Retrieval (Deep Pull of Selected Evidence)
A third LLM agent retrieves **full text and structured data** only for the selected items using:
- CourtListener opinion retrieval
- LegiScan bill retrieval (and optionally bill text via doc_id)
- DocumentCloud S3 text JSON retrieval
- Jina AI text extraction for web articles
- ScrapingDog for LinkedIn/Twitter/Instagram
- BuiltWith tech profile/relationships/social endpoints
- OpenCorporates company profile/filings/statements/data

### 1.5 Report Writing (Synthesis + Citations)
A report-writing agent synthesizes the retrieved material into a detailed Markdown report aimed at the user’s `reportGoal`, with **inline citations**.

### 1.6 Verification + Structured Hallucination Detection
A verification chain evaluates the report against retrieved sources and emits a **structured JSON** verdict (hallucinations present, list, summary).

### 1.7 Automated Remediation (Fix Hallucinations) + Output
If hallucinations are present, a “Fixing Hallucinations” agent rewrites only the flagged parts. Final output is returned either:
- as webhook response (streaming enabled), or
- as workflow output set node.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggers & Input Normalization
**Overview:** Provides two entry points and standardizes incoming fields into a consistent schema for all downstream agents.  
**Nodes involved:**  
- Trigger organization research (Webhook)  
- Trigger organization research from another workflow  
- Set Prompt

#### Node: Trigger organization research (Webhook)
- **Type / role:** `Execute Workflow Trigger` — entry point when this workflow is called as a sub-workflow by another workflow.
- **Configuration choices:** Defines expected inputs: `companyName`, `companyDomain`, `reportGoal`.
- **Outputs:** Connects to **Set Prompt**.
- **Edge cases / failures:** Missing inputs will propagate as `null` unless upstream ensures values.

#### Node: Trigger organization research from another workflow
- **Type / role:** `Webhook` — HTTP trigger endpoint.
- **Key config:** `POST` on path `org-osint`.
- **Outputs:** Connects to **Set Prompt**.
- **Edge cases:** Request body structure may vary; this is handled in **Set Prompt** via fallback expressions.

#### Node: Set Prompt
- **Type / role:** `Set` — normalizes and stores input parameters and source type.
- **Key expressions:**
  - `companyName`: `{{ $json.body?.[0]?.companyName ?? $json.companyName }}`
  - `companyDomain`: `{{ $json.body?.[0]?.companyDomain ?? $json.companyDomain }}`
  - `reportGoal`: `{{ $json.body?.[0]?.reportGoal ?? $json.reportGoal }}`
  - `source`: `{{ $json.webhookUrl ? 'webhook' : 'workflow' }}`
- **Inputs:** From either trigger.
- **Outputs:** To **Looking for Sources**.
- **Failure modes:** If both body path and top-level fields are absent, the values become `undefined` → downstream agents may produce weak/no results.

**Sticky note context (applies to this block):**  
- “Research organizations using AI with public data sources and report generation” (overview & setup guidance)  
- “🔐 Credentials Required” (credentials list; see section 5)

---

### Block 2.2 — Discovery Agent (Broad Collection)
**Overview:** Uses an LLM agent to search widely across public sources and an internal vector DB without filtering.  
**Nodes involved:**  
- Looking for Sources  
- Gemini 2.5 Flash  
- Auto Fallback  
- Google Search Discovery  
- Court Listener Discovery  
- LegiScan Discovery  
- DocumentCloud Dsicovery  
- Open Corporates1  
- Search Open Paws Database2  
- Embeddings OpenAI2  
- Retry if Tools Not Used

#### Node: Looking for Sources
- **Type / role:** `LangChain Agent` — “Organization Discovery Agent”.
- **Config choices:**
  - `maxIterations: 10`
  - `returnIntermediateSteps: true` (critical: later nodes inspect tool usage/results)
  - Strong system message enforcing discovery-only behavior (no verification/filtering).
- **Inputs:** From **Set Prompt** (`companyName`, `companyDomain`, `reportGoal`).
- **Outputs:** To **Retry if Tools Not Used**.
- **Tools available (via ai_tool connections):**
  - Google Search Discovery, Court Listener Discovery, LegiScan Discovery, DocumentCloud Dsicovery, Open Corporates1, Search Open Paws Database2.
- **Language models available (via ai_languageModel connections):**
  - Primary: **Gemini 2.5 Flash**
  - Fallback: **openrouter/auto**
- **Failure modes / edge cases:**
  - Tool calls may not happen (agent answers without using tools) → retry logic triggers.
  - Some tools require strict URL construction; discovery tools include “base URL must be included” guidance.

#### Node: Gemini 2.5 Flash
- **Type / role:** `OpenRouter Chat Model` — primary LLM for Discovery.
- **Model:** `google/gemini-2.5-flash`, temperature `0.7`.
- **Failure modes:** Provider outages/rate limits → fallback model used if agent supports it.

#### Node: Auto Fallback
- **Type / role:** `OpenRouter Chat Model` — fallback LLM for Discovery.
- **Model:** `openrouter/auto`, temperature `0.7`.

#### Node: Google Search Discovery
- **Type / role:** `LangChain HTTP Request Tool` (Serper API wrapper).
- **Endpoint:** `https://google.serper.dev/{endpoint}` with placeholder `{endpoint}` (`search`, `news`, `scholar`, etc.).
- **Headers:** Content-Type `application/json`.
- **Auth:** HTTP header auth credential “Serper API”.
- **Edge cases:** wrong endpoint placeholder; quota/rate limits; request body formatting errors.

#### Node: Court Listener Discovery
- **Type / role:** `HTTP Request Tool` for CourtListener search discovery.
- **Auth:** HTTP header auth “CourtListener Key”.
- **Headers:** `Accept: application/json`.
- **Key guidance:** Use `/api/rest/v4/search/?type=o&q=...&page_size=20` and save `id`/`cluster_id`.
- **Edge cases:** invalid query encoding; pagination; 401 if key invalid.

#### Node: LegiScan Discovery
- **Type / role:** `HTTP Request Tool` for LegiScan search.
- **Auth:** Query auth “Legiscan Key”.
- **Key guidance:** Must use full URL `https://api.legiscan.com/?op=getSearch&...`
- **Edge cases:** missing base URL (explicitly warned); rate limits; state/year parameter errors.

#### Node: DocumentCloud Dsicovery
- **Type / role:** `HTTP Request Tool` for DocumentCloud search.
- **Key guidance:** `https://api.www.documentcloud.org/api/documents/search/?q={query}&per_page=25` and save both `id` and `slug`.
- **Edge cases:** DocumentCloud API changes; large result sets; missing slug.

#### Node: Open Corporates1
- **Type / role:** `HTTP Request Tool` for OpenCorporates discovery/search.
- **Auth:** Query auth “OpenCorporates API Key”.
- **Key guidance:** base `https://api.opencorporates.com/v0.4/`.
- **Edge cases:** pagination; matching wrong entity names; rate limits.

#### Node: Search Open Paws Database2
- **Type / role:** `Weaviate Vector Store Tool` (retrieve-as-tool).
- **Config:** collection `Content`, `topK: 10`.
- **Purpose:** internal advocacy knowledge retrieval to complement public web data.
- **Edge cases:** Weaviate credentials/collection mismatch; embeddings mismatch.

#### Node: Embeddings OpenAI2
- **Type / role:** `OpenAI Embeddings` used by Weaviate retrieval.
- **Credential:** “OpenAi account”.
- **Edge cases:** embedding model default changes; quota/rate limits; mismatch with Weaviate vector dimensions if altered.

#### Node: Retry if Tools Not Used
- **Type / role:** `IF` — retries Discovery if no tools were invoked.
- **Condition logic:**
  - `$json.intermediateSteps[0]` is empty (no tool calls recorded)
  - `$runIndex < 4` (max ~3 retries beyond first run)
- **Outputs:**
  - **true** → back to **Looking for Sources**
  - **false** → to **Prioritizing Sources**
- **Edge cases:** If intermediateSteps exists but tool calls failed silently, this may not trigger.

**Sticky note context:**  
- Credentials note applies to all external tools in discovery.  
- Open Paws guide note applies to the Weaviate tool.

---

### Block 2.3 — Prioritization & Verification Agent (Select Confirmed Items)
**Overview:** Verifies identity match and relevance, selecting only 100% confirmed items for retrieval. Includes retry if output is empty.  
**Nodes involved:**  
- Prioritizing Sources  
- GPT-5a  
- Auto Fallback1  
- Think Tool Prioritization  
- Retry if Response Empty

#### Node: Prioritizing Sources
- **Type / role:** `LangChain Agent` — “Organization Prioritization & Verification Agent”.
- **Config:**
  - `maxIterations: 10`
  - `returnIntermediateSteps: true`
  - System message enforces “ONLY 100% CONFIRMED = INCLUDE” and outputs a strict structure including “CRITICAL RETRIEVAL IDS SUMMARY”.
- **Inputs:** Discovery agent `intermediateSteps` (stringified and truncated to 800k chars with per-tool truncation logic inside prompt).
- **Outputs:** To **Retry if Response Empty**.
- **Tools available:** **Think Tool Prioritization** (reasoning checklist).
- **Models:** Primary **GPT‑5a**, fallback **Auto Fallback1**.
- **Edge cases:**
  - Over-truncation may remove critical verification evidence → agent may exclude too much.
  - “100% confirmed only” can lead to empty selection if discovery evidence is weak.

#### Node: GPT-5a
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `openai/gpt-5`, temperature `0.3` (more deterministic for verification).
- **Edge cases:** model availability via OpenRouter.

#### Node: Auto Fallback1
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `openrouter/auto`, temperature `0.3`.

#### Node: Think Tool Prioritization
- **Type / role:** `Think Tool`
- **Purpose:** Provides an explicit verification rubric (name match, domain match, jurisdiction, industry, red flags; verdict categories).
- **Used by:** **Prioritizing Sources** agent as a tool.
- **Failure modes:** None (internal), but agent may ignore tool.

#### Node: Retry if Response Empty
- **Type / role:** `IF`
- **Condition logic:**
  - `$json.output` is empty
  - `$runIndex < 4`
- **Outputs:**
  - **true** → rerun **Prioritizing Sources**
  - **false** → proceed to **Investigating Sources Further**
- **Edge cases:** If the agent outputs non-empty but malformed structure, downstream retrieval may fail even though retry doesn’t trigger.

**Sticky note context:** “🟤 Prioritizing Sources” description applies to this block.

---

### Block 2.4 — Retrieval Agent (Deep Evidence Pull)
**Overview:** Retrieves detailed content only for prioritized items, using specialized tools for each source type; includes “tools not used” retry path.  
**Nodes involved:**  
- Investigating Sources Further  
- Gemini 2.5 Flash2  
- Auto Fallback2  
- BuiltWith1  
- Open Corporates2  
- Court Listener Retrieveal  
- LegiScan Retrieval  
- DocumentCloud Retrieval  
- Jina URL Text Extraction  
- Linkedin Person and Company Scraper1  
- Twitter Profile Scraper1  
- Instagram Profile Scraper1  
- Retry if Tools Not Used1

#### Node: Investigating Sources Further
- **Type / role:** `LangChain Agent` — “Organization Retrieval Agent”.
- **Config:**
  - `maxIterations: 10`
  - Input prompt passes prioritization output (JSON stringified, truncated to 500k).
  - System message prohibits new searches; retrieve only “SELECTED FOR RETRIEVAL”.
  - `returnIntermediateSteps: true`
- **Outputs:** To **Retry if Tools Not Used1**.
- **Tools available:** BuiltWith1, Open Corporates2, Court Listener Retrieveal, LegiScan Retrieval, DocumentCloud Retrieval, Jina URL Text Extraction, LinkedIn/Twitter/Instagram scrapers.
- **Models:** Primary **Gemini 2.5 Flash2**, fallback **Auto Fallback2**.
- **Edge cases:**
  - Prioritization output must include correct IDs/slugs; otherwise retrieval tools cannot construct URLs.
  - Large retrieval results may exceed prompt budgets later; workflow uses aggressive truncation in later nodes.

#### Node: Gemini 2.5 Flash2
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `google/gemini-2.5-flash`, temperature `0.2` (more factual, less creative).

#### Node: Auto Fallback2
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `openrouter/auto`, temperature `0.2`.

#### Node: BuiltWith1
- **Type / role:** `HTTP Request Tool` for BuiltWith tech profiling.
- **Auth:** query auth “BuiltWith Key”.
- **Critical configuration guidance:** Must use full API URL; key endpoints:
  - `/v21/api.json?LOOKUP=domain`
  - `/rv3/api.json?LOOKUP=domain`
  - `/social1/api.json?LOOKUP=name`
- **Headers:** `Accept: application/json`.
- **Edge cases:** rate limits; invalid LOOKUP; BuiltWith endpoint changes.

#### Node: Open Corporates2
- **Type / role:** `HTTP Request Tool` for OpenCorporates retrieval.
- **Auth:** query auth “OpenCorporates API Key”.
- **Use:** Retrieve company profile, filings, statements, data, ownership statements.
- **Edge cases:** pagination; sparse vs full; endpoint selection errors.

#### Node: Court Listener Retrieveal
- **Type / role:** `HTTP Request Tool` for CourtListener full opinion retrieval.
- **Auth:** header auth “CourtListener Key”; `Accept: application/json`.
- **Critical guidance:**
  - Use `/opinions/{id}/?fields=plain_text,html_with_citations,case_name,absolute_url`
  - Trailing slash before query params is required.
- **Edge cases:** missing opinion IDs; 404 for removed opinions; large texts.

#### Node: LegiScan Retrieval
- **Type / role:** `HTTP Request Tool` to fetch a bill by `bill_id`.
- **Auth:** query auth “Legiscan Key”.
- **Critical guidance:** Must call `https://api.legiscan.com/?op=getBill&id={bill_id}`; to get text: `getBillText&id={doc_id}`.
- **Edge cases:** bill texts sometimes behind additional URLs; doc_id retrieval required.

#### Node: DocumentCloud Retrieval
- **Type / role:** `HTTP Request Tool` retrieving text JSON from DocumentCloud S3.
- **Critical URL pattern:** `https://s3.documentcloud.org/documents/{id}/{slug}.txt.json`
- **Edge cases:** slug mismatch; missing txt.json for some documents.

#### Node: Jina URL Text Extraction
- **Type / role:** `Jina AI Tool` to extract clean article text from a URL.
- **Auth:** Jina AI API account.
- **Edge cases:** blocked pages/paywalls; dynamic sites; rate limits.

#### Node: Linkedin Person and Company Scraper1
- **Type / role:** `LangChain HTTP Request Tool` to ScrapingDog LinkedIn endpoint.
- **Endpoint:** `https://api.scrapingdog.com/linkedin` with query params `linkId`, `type` (`profile` or `company`).
- **Edge cases:** LinkedIn blocking; ScrapingDog quota; incorrect linkId/type.

#### Node: Twitter Profile Scraper1
- **Type / role:** `LangChain HTTP Request Tool` to ScrapingDog X/Twitter endpoint.
- **Endpoint:** `http://api.scrapingdog.com/x/profile?profileId={profileId}&parsed=true`
- **Edge cases:** username changes; X restrictions; non-HTTPS endpoint may be blocked in some environments.

#### Node: Instagram Profile Scraper1
- **Type / role:** `LangChain HTTP Request Tool` to ScrapingDog Instagram endpoint.
- **Endpoint:** `https://api.scrapingdog.com/instagram/profile?username={username}`
- **Edge cases:** private profiles; IG blocks; quota.

#### Node: Retry if Tools Not Used1
- **Type / role:** `IF`
- **Condition logic:**
  - `$json.intermediateSteps[0]` is empty
  - `$runIndex < 4`
- **Outputs:**
  - **true** → rerun **Investigating Sources Further**
  - **false** → proceed to **Writing Report**
- **Edge cases:** If tools were invoked but returned errors, this node won’t retry; you may want additional error-based conditions.

**Sticky note context:** “🟤 Investigating Sources Further” applies to this block.

---

### Block 2.5 — Report Writing (Synthesis)
**Overview:** Produces a comprehensive, goal-focused Markdown report from retrieved sources; retries on empty output.  
**Nodes involved:**  
- Writing Report  
- GPT-5b  
- Auto Fallback3  
- Think Tool Analysis  
- Retry if Response Empty1

#### Node: Writing Report
- **Type / role:** `LangChain Agent` — report writer.
- **Config:**
  - `maxIterations: 10`
  - Takes retrieval `intermediateSteps`, normalizes into `{step, tool, input, response}` objects, then applies dynamic truncation (2.4M char budget).
  - System message enforces: no follow-ups, exhaustive report, inline citations, adaptive structure.
  - `returnIntermediateSteps: true`
- **Tools:** **Think Tool Analysis** (strategic synthesis guidance).
- **Models:** Primary **GPT‑5b** (temp 0.6), fallback **Auto Fallback3**.
- **Outputs:** To **Retry if Response Empty1**.
- **Edge cases:** If retrieval results are huge, truncation may cut key sources; citations may be missing if URLs are absent from tool responses.

#### Node: GPT-5b
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `openai/gpt-5`, temperature `0.6`.

#### Node: Auto Fallback3
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `openrouter/auto`, temperature `0.6`.

#### Node: Think Tool Analysis
- **Type / role:** `Think Tool`
- **Purpose:** Forces structured synthesis: map to goal, patterns, contradictions, gaps, insights, recommendations.
- **Failure modes:** none.

#### Node: Retry if Response Empty1
- **Type / role:** `IF`
- **Condition logic:**
  - `$json.output` is empty
  - `$runIndex < 4`
- **Outputs:**
  - **true** → rerun **Writing Report**
  - **false** → proceed to **Set Report**
- **Edge cases:** Doesn’t validate Markdown quality; only emptiness.

**Sticky note context:** “✍️ Writing Report Agent” applies to this block.

---

### Block 2.6 — Verification + Structured Output Parsing
**Overview:** Converts the report + retrieved corpus into a structured hallucination verdict, with retry-on-empty gating.  
**Nodes involved:**  
- Set Report  
- Verifying Report  
- GPT-5c  
- Auto Fallback4  
- Structured Output Parser  
- If Empty Output1  
- If hallucinations present

#### Node: Set Report
- **Type / role:** `Set` — packages report and retrieved documents for verification.
- **Key fields:**
  - `Final Report`: `{{ $json.output }}` (from report writer)
  - `Retrieved Documents`: complex expression that extracts and truncates the retrieval agent’s `intermediateSteps` into a JSON string (400k char budget) with per-response truncation and a final size guard.
- **Inputs:** from **Retry if Response Empty1** false branch.
- **Outputs:** to **Verifying Report**.
- **Edge cases:** References `$('Investigating Sources Further').item.json.intermediateSteps`; if retrieval didn’t run or node name changes, expression fails.

#### Node: Verifying Report
- **Type / role:** `LangChain LLM Chain` — verification prompt (fact-check layer).
- **Key config:**
  - `promptType: define`
  - `hasOutputParser: true` (paired with Structured Output Parser)
  - Feeds “Retrieved Documents” and “Final Report” with large truncation logic.
- **Models:** primary **GPT‑5c**, fallback **Auto Fallback4** (also connected as LM for the parser).
- **Outputs:** to **If Empty Output1**.
- **Failure modes:** Output must match the parser schema; otherwise parser auto-fix attempts to repair.

#### Node: Structured Output Parser
- **Type / role:** `Structured Output Parser`
- **Schema:** JSON schema with:
  - `contains_hallucinations` (boolean)
  - `hallucinations` array of objects: `exact_text`, `issue_type`, `severity`, optional `searched_in`
  - `summary` string
- **autoFix:** enabled (attempts to fix invalid JSON outputs).
- **Edge cases:** LLM may output too-long strings or invalid enums; autoFix may still fail.

#### Node: If Empty Output1
- **Type / role:** `IF` — ensures verification output exists.
- **Condition:** `$json.output` object is empty.
- **Outputs:**
  - **true** → rerun **Verifying Report**
  - **false** → to **If hallucinations present**

#### Node: If hallucinations present
- **Type / role:** `IF` — checks whether remediation is required and caps retries.
- **Condition:**
  - `$json.output.contains_hallucinations === true`
  - `$runIndex < 4`
- **Outputs:**
  - **true** → **Step 6: Fixing Hallucinations**
  - **false** → **Set Report1** (pass-through final report)

**Sticky note context:** “🔍 Verifying Report Agent” applies to this block.

---

### Block 2.7 — Fixing Hallucinations + Final Output Routing
**Overview:** If hallucinations are detected, rewrites the report conservatively using only retrieved evidence; ensures non-empty output; routes response depending on trigger source.  
**Nodes involved:**  
- Step 6: Fixing Hallucinations  
- GPT-5d  
- Auto Fallback5  
- Think Tool Analysis2  
- If Empty Output  
- Set Report1  
- If Source is Webhook  
- Respond to Webhook  
- Set Output1

#### Node: Step 6: Fixing Hallucinations
- **Type / role:** `LangChain Agent` — “Report Rewrite Agent” focused on surgical fixes.
- **Inputs:**
  - User query fields from **Set Prompt**
  - Full retrieved `intermediateSteps` from **Investigating Sources Further** (truncated to 1.6M chars)
  - Failed report from **Set Report** (truncated to 800k)
  - Hallucinations list from verifier output (truncated to 80k)
- **Config:**
  - `maxIterations: 10`
  - `returnIntermediateSteps: true`
  - System message emphasizes minimal edits, preserve structure/analysis, fix only flagged claims, keep citations.
- **Models:** primary **GPT‑5d** (temp 0.1), fallback **Auto Fallback5**.
- **Tools:** **Think Tool Analysis2**.
- **Outputs:** to **If Empty Output**.
- **Edge cases:** If verifier flags many items, rewrite may become large; truncation could omit some hallucination entries.

#### Node: GPT-5d
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `openai/gpt-5`, temperature `0.1` (highly conservative).

#### Node: Auto Fallback5
- **Type / role:** `OpenRouter Chat Model`
- **Model:** `openrouter/auto`, temperature `0.1`.

#### Node: Think Tool Analysis2
- **Type / role:** `Think Tool` (same synthesis framework as Think Tool Analysis).
- **Used by:** Fixing agent.

#### Node: If Empty Output
- **Type / role:** `IF`
- **Condition:** `$json.output` is empty (string) — loose validation.
- **Outputs:**
  - **true** → **Set Report** and **Step 6: Fixing Hallucinations** (re-attempt path)
  - **false** → **Set Report** (finalize)
- **Edge cases:** Branching sends to Set Report in both cases; ensure connections align with desired retry behavior (here it still updates Set Report).

#### Node: Set Report1
- **Type / role:** `Set`
- **Field:** `report = {{ $('Set Report').item.json['Final Report'] }}`
- **Purpose:** Normalizes the final report into a single `report` field before output routing.
- **Inputs:** from “If hallucinations present” false branch.
- **Outputs:** to **If Source is Webhook**.

#### Node: If Source is Webhook
- **Type / role:** `IF`
- **Condition:** `{{ $('Set Prompt').item.json.source }} == 'webhook'`
- **Outputs:**
  - **true** → **Respond to Webhook**
  - **false** → **Set Output1**
- **Edge cases:** If `source` is missing/altered, it will default to workflow path.

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook`
- **Config:** streaming enabled; respond with JSON; body is `{{ JSON.stringify($json) }}`
- **Failure modes:** If upstream returns huge JSON, response may be large; some reverse proxies limit response size.

#### Node: Set Output1
- **Type / role:** `Set`
- **Field:** `report = {{ $('Set Report').item.json['Final Report'] }}`
- **Purpose:** For non-webhook execution, provides `report` in the workflow output.
- **Edge cases:** Similar dependency on **Set Report** naming.

**Sticky note context:** “🧯 Fixing Hallucinations Agent” applies to this block.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger organization research (Webhook) | executeWorkflowTrigger | Entry point when invoked as sub-workflow | — | Set Prompt | # Research organizations using AI with public data sources and report generation / ## 🔐 Credentials Required |
| Trigger organization research from another workflow | webhook | HTTP entry point (`POST /org-osint`) | — | Set Prompt | # Research organizations using AI with public data sources and report generation / ## 🔐 Credentials Required |
| Set Prompt | set | Normalize inputs and tag source type | Trigger organization research (Webhook); Trigger organization research from another workflow | Looking for Sources | # Research organizations using AI with public data sources and report generation / ## 🔐 Credentials Required |
| Looking for Sources | langchain.agent | Discovery agent (broad collection) | Set Prompt | Retry if Tools Not Used | # Research organizations using AI with public data sources and report generation / ## 🔐 Credentials Required |
| Gemini 2.5 Flash | lmChatOpenRouter | Primary LLM for discovery | — (model connection) | Looking for Sources (ai_languageModel) | ## 🔐 Credentials Required |
| Auto Fallback | lmChatOpenRouter | Fallback LLM for discovery | — (model connection) | Looking for Sources (ai_languageModel) | ## 🔐 Credentials Required |
| Google Search Discovery | toolHttpRequest | Web/news/scholar discovery (Serper) | — (tool connection) | Looking for Sources (ai_tool) | ## 🔐 Credentials Required |
| Court Listener Discovery | httpRequestTool | CourtListener search discovery | — (tool connection) | Looking for Sources (ai_tool) | ## 🔐 Credentials Required |
| LegiScan Discovery | httpRequestTool | Legislation discovery search | — (tool connection) | Looking for Sources (ai_tool) | ## 🔐 Credentials Required |
| DocumentCloud Dsicovery | httpRequestTool | DocumentCloud discovery search | — (tool connection) | Looking for Sources (ai_tool) | ## 🔐 Credentials Required |
| Open Corporates1 | httpRequestTool | OpenCorporates discovery search | — (tool connection) | Looking for Sources (ai_tool) | ## 🔐 Credentials Required |
| Search Open Paws Database2 | vectorStoreWeaviate | Internal KB retrieval tool | — (tool connection) | Looking for Sources (ai_tool) | ### Please refer [Open Paws Guide](https://github.com/Open-Paws/documentation/tree/main/Knowledge) to know how to use our open-source vector database |
| Embeddings OpenAI2 | embeddingsOpenAi | Embeddings for Weaviate search | — | Search Open Paws Database2 (ai_embedding) | ### Please refer [Open Paws Guide](https://github.com/Open-Paws/documentation/tree/main/Knowledge) to know how to use our open-source vector database |
| Retry if Tools Not Used | if | Retry discovery when no tools used | Looking for Sources | Looking for Sources; Prioritizing Sources | ## 🔐 Credentials Required |
| Prioritizing Sources | langchain.agent | Verify identity + prioritize, select IDs | Retry if Tools Not Used | Retry if Response Empty | ## 🟤 Prioritizing Sources |
| GPT-5a | lmChatOpenRouter | Primary LLM for prioritization | — (model connection) | Prioritizing Sources (ai_languageModel) | ## 🟤 Prioritizing Sources |
| Auto Fallback1 | lmChatOpenRouter | Fallback LLM for prioritization | — (model connection) | Prioritizing Sources (ai_languageModel) | ## 🟤 Prioritizing Sources |
| Think Tool Prioritization | toolThink | Verification rubric tool | — (tool connection) | Prioritizing Sources (ai_tool) | ## 🟤 Prioritizing Sources |
| Retry if Response Empty | if | Retry prioritization if output empty | Prioritizing Sources | Prioritizing Sources; Investigating Sources Further | ## 🟤 Prioritizing Sources |
| Investigating Sources Further | langchain.agent | Retrieve full evidence for selected items | Retry if Response Empty | Retry if Tools Not Used1 | ## 🟤 Investigating Sources Further |
| Gemini 2.5 Flash2 | lmChatOpenRouter | Primary LLM for retrieval | — (model connection) | Investigating Sources Further (ai_languageModel) | ## 🟤 Investigating Sources Further |
| Auto Fallback2 | lmChatOpenRouter | Fallback LLM for retrieval | — (model connection) | Investigating Sources Further (ai_languageModel) | ## 🟤 Investigating Sources Further |
| BuiltWith1 | httpRequestTool | Tech/financials/relationships retrieval | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| Open Corporates2 | httpRequestTool | Corporate registry retrieval | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| Court Listener Retrieveal | httpRequestTool | Court opinion full text retrieval | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| LegiScan Retrieval | httpRequestTool | Bill retrieval (and doc_id for text) | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| DocumentCloud Retrieval | httpRequestTool | DocumentCloud S3 text retrieval | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| Jina URL Text Extraction | jinaAiTool | Web page article extraction | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| Linkedin Person and Company Scraper1 | toolHttpRequest | LinkedIn scraping | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| Twitter Profile Scraper1 | toolHttpRequest | X/Twitter scraping | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| Instagram Profile Scraper1 | toolHttpRequest | Instagram scraping | — (tool connection) | Investigating Sources Further (ai_tool) | ## 🔐 Credentials Required |
| Retry if Tools Not Used1 | if | Retry retrieval when no tools used | Investigating Sources Further | Investigating Sources Further; Writing Report | ## 🟤 Investigating Sources Further |
| Writing Report | langchain.agent | Synthesize retrieved data into Markdown report | Retry if Tools Not Used1 | Retry if Response Empty1 | ## ✍️ Writing Report Agent |
| GPT-5b | lmChatOpenRouter | Primary LLM for report writing | — (model connection) | Writing Report (ai_languageModel) | ## ✍️ Writing Report Agent |
| Auto Fallback3 | lmChatOpenRouter | Fallback LLM for report writing | — (model connection) | Writing Report (ai_languageModel) | ## ✍️ Writing Report Agent |
| Think Tool Analysis | toolThink | Analysis framework tool | — (tool connection) | Writing Report (ai_tool) | ## ✍️ Writing Report Agent |
| Retry if Response Empty1 | if | Retry report writing if output empty | Writing Report | Writing Report; Set Report | ## ✍️ Writing Report Agent |
| Set Report | set | Package report + retrieved docs for verification | Retry if Response Empty1 | Verifying Report | ## 🔍 Verifying Report Agent |
| Verifying Report | chainLlm | Fact-check report vs retrieved sources | Set Report | If Empty Output1 | ## 🔍 Verifying Report Agent |
| GPT-5c | lmChatOpenRouter | Primary LLM for verification | — (model connection) | Verifying Report (ai_languageModel) | ## 🔍 Verifying Report Agent |
| Auto Fallback4 | lmChatOpenRouter | Fallback LLM for verification + parser | — (model connection) | Verifying Report; Structured Output Parser (ai_languageModel) | ## 🔍 Verifying Report Agent |
| Structured Output Parser | outputParserStructured | Parse hallucination verdict into strict schema | — (parser connection) | Verifying Report (ai_outputParser) | ## 🔍 Verifying Report Agent |
| If Empty Output1 | if | Retry verification if parsed output empty | Verifying Report | Verifying Report; If hallucinations present | ## 🔍 Verifying Report Agent |
| If hallucinations present | if | Gate remediation when hallucinations found | If Empty Output1 | Step 6: Fixing Hallucinations; Set Report1 | ## 🧯 Fixing Hallucinations Agent |
| Step 6: Fixing Hallucinations | langchain.agent | Surgical rewrite to remove hallucinations | If hallucinations present | If Empty Output | ## 🧯 Fixing Hallucinations Agent |
| GPT-5d | lmChatOpenRouter | Primary LLM for fixing hallucinations | — (model connection) | Step 6: Fixing Hallucinations (ai_languageModel) | ## 🧯 Fixing Hallucinations Agent |
| Auto Fallback5 | lmChatOpenRouter | Fallback LLM for fixing hallucinations | — (model connection) | Step 6: Fixing Hallucinations (ai_languageModel) | ## 🧯 Fixing Hallucinations Agent |
| Think Tool Analysis2 | toolThink | Synthesis guidance tool (for rewrite agent) | — (tool connection) | Step 6: Fixing Hallucinations (ai_tool) | ## 🧯 Fixing Hallucinations Agent |
| If Empty Output | if | Retry hallucination-fix if empty output | Step 6: Fixing Hallucinations | Set Report; Step 6: Fixing Hallucinations | ## 🧯 Fixing Hallucinations Agent |
| Set Report1 | set | Normalize final report into `report` field | If hallucinations present | If Source is Webhook | ## 🧯 Fixing Hallucinations Agent |
| If Source is Webhook | if | Route output based on trigger source | Set Report1 | Respond to Webhook; Set Output1 | # Research organizations using AI with public data sources and report generation |
| Respond to Webhook | respondToWebhook | Return JSON response (streaming) | If Source is Webhook | — | # Research organizations using AI with public data sources and report generation |
| Set Output1 | set | Provide final output for non-webhook runs | If Source is Webhook | — | # Research organizations using AI with public data sources and report generation |
| (Unused in active path) Set Report (duplicate name in JSON: “0755334c…”) | set | Alternative output setter (present but not connected) | — | — |  |

> Note: The workflow JSON contains **two Set nodes that set `report` from Set Report** (“Set Output1” and “Set Report1”), and also a separate node named **“Set Report1”** plus an additional **“Set Report1”**-like node (`0755334c...`) that is **not connected** in `connections`. It is still listed above as present.

---

## 4. Reproducing the Workflow from Scratch

1) **Create two triggers**
   1. Add **Execute Workflow Trigger** node named **“Trigger organization research (Webhook)”**  
      - Define workflow inputs: `companyName`, `companyDomain`, `reportGoal`.
   2. Add **Webhook** node named **“Trigger organization research from another workflow”**  
      - Method: `POST`  
      - Path: `org-osint`

2) **Add “Set Prompt” (Set node)**
   - Add fields:
     - `companyName` = `{{ $json.body?.[0]?.companyName ?? $json.companyName }}`
     - `companyDomain` = `{{ $json.body?.[0]?.companyDomain ?? $json.companyDomain }}`
     - `reportGoal` = `{{ $json.body?.[0]?.reportGoal ?? $json.reportGoal }}`
     - `source` = `{{ $json.webhookUrl ? 'webhook' : 'workflow' }}`
   - Connect both triggers → **Set Prompt**.

3) **Discovery agent**
   - Add **LangChain Agent** named **“Looking for Sources”**
     - `maxIterations: 10`, `returnIntermediateSteps: true`
     - Prompt includes `companyName`, `companyDomain`, `reportGoal` from input.
     - System message: discovery-only instructions (broad OSINT, no filtering).
   - Add LLMs:
     - **OpenRouter Chat Model** “Gemini 2.5 Flash” (`google/gemini-2.5-flash`, temp 0.7)
     - **OpenRouter Chat Model** “Auto Fallback” (`openrouter/auto`, temp 0.7)
   - Connect both models to the agent’s **ai_languageModel** ports (primary + fallback).

4) **Add discovery tools (as LangChain tools) and connect to discovery agent**
   - **Google Search Discovery** (`@n8n/n8n-nodes-langchain.toolHttpRequest`)
     - URL: `https://google.serper.dev/{endpoint}`
     - Header `Content-Type: application/json`
     - Credential: **HTTP Header Auth** (Serper API key)
   - **Court Listener Discovery** (`httpRequestTool`)
     - Credential: **HTTP Header Auth** (CourtListener token)
     - Header `Accept: application/json`
   - **LegiScan Discovery** (`httpRequestTool`)
     - Credential: **HTTP Query Auth** (LegiScan key)
   - **DocumentCloud Dsicovery** (`httpRequestTool`) (no auth required in this template)
   - **Open Corporates1** (`httpRequestTool`)
     - Credential: **HTTP Query Auth** (OpenCorporates key)
   - **Weaviate Vector Store** “Search Open Paws Database2”
     - Mode: retrieve-as-tool, `topK: 10`, collection `Content`
     - Add **OpenAI Embeddings** node and connect as embedding provider
     - Credential: OpenAI API key
   - Connect each tool node to **Looking for Sources** as **ai_tool**.

5) **Discovery retry gate**
   - Add **IF** “Retry if Tools Not Used”
     - Condition 1: object empty → `{{ $json.intermediateSteps[0] }}`
     - Condition 2: number `<` → `{{ $runIndex }}` `< 4`
   - Connect **Looking for Sources** → this IF
   - IF true → back to **Looking for Sources**
   - IF false → onward to Prioritization.

6) **Prioritization agent**
   - Add **LangChain Agent** “Prioritizing Sources”
     - `maxIterations: 10`, `returnIntermediateSteps: true`
     - Prompt: pass discovery `intermediateSteps` (stringify + truncation), plus confirmed `companyName/companyDomain/reportGoal`.
     - System message: strict “ONLY 100% CONFIRMED” plus required output sections and ID summary.
   - Add models:
     - “GPT-5a” (`openai/gpt-5`, temp 0.3)
     - “Auto Fallback1” (`openrouter/auto`, temp 0.3)
   - Add **Think Tool** “Think Tool Prioritization” and connect as **ai_tool**.
   - Connect models to agent’s ai_languageModel.

7) **Prioritization retry gate**
   - Add **IF** “Retry if Response Empty”
     - Condition: string empty `{{ $json.output }}` and `$runIndex < 4`
   - true → rerun **Prioritizing Sources**
   - false → proceed to retrieval agent.

8) **Retrieval agent**
   - Add **LangChain Agent** “Investigating Sources Further”
     - `maxIterations: 10`, `returnIntermediateSteps: true`
     - System message: retrieve only selected items; no new search.
   - Add models:
     - “Gemini 2.5 Flash2” (`google/gemini-2.5-flash`, temp 0.2)
     - “Auto Fallback2” (`openrouter/auto`, temp 0.2)
   - Add retrieval tools and credentials:
     - CourtListener retrieval tool (header auth)
     - LegiScan retrieval tool (query auth)
     - DocumentCloud retrieval tool (no auth)
     - Jina AI tool (Jina API key)
     - ScrapingDog tools (LinkedIn/Twitter/Instagram) (query auth)
     - BuiltWith tool (query auth)
     - OpenCorporates tool (query auth)
   - Connect tools to the agent as ai_tool.

9) **Retrieval retry gate**
   - Add **IF** “Retry if Tools Not Used1”
     - Condition: `{{ $json.intermediateSteps[0] }}` empty AND `{{ $runIndex }} < 4`
   - true → rerun retrieval agent
   - false → proceed to report writing.

10) **Report writing agent**
   - Add **LangChain Agent** “Writing Report”
     - `maxIterations: 10`, `returnIntermediateSteps: true`
     - Prompt: stringify retrieval intermediateSteps with truncation.
     - System message: exhaustive Markdown report, inline citations with full URLs.
   - Add models:
     - “GPT-5b” (`openai/gpt-5`, temp 0.6)
     - “Auto Fallback3” (`openrouter/auto`, temp 0.6)
   - Add **Think Tool** “Think Tool Analysis” and connect as ai_tool.

11) **Report retry gate**
   - Add **IF** “Retry if Response Empty1”
     - Condition: `{{ $json.output }}` empty AND `{{ $runIndex }} < 4`
   - true → rerun **Writing Report**
   - false → proceed.

12) **Package for verification**
   - Add **Set** node “Set Report”
     - `Final Report` = `{{ $json.output }}`
     - `Retrieved Documents` = expression that extracts `$('Investigating Sources Further').item.json.intermediateSteps` and truncates into JSON string (as in workflow).

13) **Verification chain + structured parser**
   - Add **Chain LLM** “Verifying Report”
     - Provide large prompt including retrieved docs + report
     - Enable output parser usage
   - Add **Structured Output Parser** node with the provided JSON schema
     - `autoFix: true`
   - Add models:
     - “GPT-5c” (`openai/gpt-5`, temp 0.2)
     - “Auto Fallback4” (`openrouter/auto`, temp 0.2)
   - Connect:
     - Model → Verifying Report (ai_languageModel)
     - Parser → Verifying Report (ai_outputParser)
     - Auto Fallback4 also connected to parser LM input.

14) **Verification empty-output retry**
   - Add **IF** “If Empty Output1”
     - Condition: object empty `{{ $json.output }}`
   - true → rerun Verifying Report
   - false → proceed.

15) **Hallucination gate**
   - Add **IF** “If hallucinations present”
     - `{{ $json.output.contains_hallucinations }}` is true
     - `{{ $runIndex }} < 4`
   - true → fixing agent
   - false → finalize without fixing.

16) **Fixing hallucinations agent**
   - Add **LangChain Agent** “Step 6: Fixing Hallucinations”
     - `maxIterations: 10`, `returnIntermediateSteps: true`
     - Prompt includes:
       - inputs from Set Prompt
       - retrieved intermediateSteps (truncated)
       - failed report
       - hallucinations list
     - System message emphasizes minimal intervention + citation discipline.
   - Add models:
     - “GPT-5d” (`openai/gpt-5`, temp 0.1)
     - “Auto Fallback5” (`openrouter/auto`, temp 0.1)
   - Add **Think Tool Analysis2** and connect as ai_tool.

17) **Fix output empty gate**
   - Add **IF** “If Empty Output”
     - Condition: string empty `{{ $json.output }}`
   - true → loop back to fixing (and/or Set Report) per your chosen wiring
   - false → proceed to Set Report.

18) **Final report field normalization**
   - Add **Set** node “Set Report1”:
     - `report = {{ $('Set Report').item.json['Final Report'] }}`
   - Connect “If hallucinations present” false branch → Set Report1 (and after fixing, ensure Set Report has updated Final Report).

19) **Output routing**
   - Add **IF** “If Source is Webhook”
     - Condition: `{{ $('Set Prompt').item.json.source }}` equals `webhook`
   - true → **Respond to Webhook**
     - Enable streaming; respond JSON with `{{ JSON.stringify($json) }}`
   - false → **Set Output1**
     - `report = {{ $('Set Report').item.json['Final Report'] }}`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Research organizations using AI with public data sources and report generation (how it works + setup steps) | (Sticky note content embedded in workflow canvas) |
| Credentials required: OpenRouter, Serper, LegiScan, CourtListener, OpenCorporates, Jina, ScrapingDog, BuiltWith | (Sticky note content embedded in workflow canvas) |
| Open Paws vector database usage guide | https://github.com/Open-Paws/documentation/tree/main/Knowledge |

