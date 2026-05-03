Research US legal regulations with CourtListener, LegiScan, OpenRouter and web search

https://n8nworkflows.xyz/workflows/research-us-legal-regulations-with-courtlistener--legiscan--openrouter-and-web-search-12508


# Research US legal regulations with CourtListener, LegiScan, OpenRouter and web search

## 1. Workflow Overview

**Title (given):** Research US legal regulations with CourtListener, LegiScan, OpenRouter and web search  
**Workflow name (in n8n):** Research US regulations using AI agents with web search and report generation

This workflow accepts a legal research prompt via a **POST Webhook**, runs a **multi-agent AI pipeline** to (1) discover relevant legal sources, (2) prioritize the best items, (3) retrieve full texts, (4) write a detailed legal intelligence report with citations, (5) verify the report for material legal fabrications, and (6) optionally rewrite the report to fix detected hallucinations before returning a final output.

### 1.1 Input Reception
- Receives the user query (`body.prompt`) via webhook in streaming response mode.

### 1.2 Discovery (Wide-net search across sources)
- AI agent must use all discovery tools: **LegiScan**, **CourtListener**, **DocumentCloud**, **Serper/Google via Serper**, plus **OpenStates/Plural**.

### 1.3 Prioritization (Scoring and selection)
- AI agent uses a Think tool to systematically score and select 3–10 best items for retrieval.

### 1.4 Retrieval (Fetch full texts for selected items)
- AI agent retrieves full texts using dedicated tools:
  - CourtListener opinion text
  - LegiScan bill details/text
  - DocumentCloud S3 text JSON
  - Jina AI Reader extraction for web URLs
  - OpenStates/Plural bill detail API

### 1.5 Report Writing (Synthesis)
- AI agent writes a long, strategic legal report with strict citation rules.

### 1.6 Verification (Hallucination detection)
- LLM chain checks the report against retrieved documents and produces a **structured JSON** verdict.

### 1.7 Fixing (Conditional rewrite if hallucinations found)
- If verification flags hallucinations, a rewrite agent produces a corrected final report; otherwise the original report is returned.

### 1.8 Resilience / Retry loops
- Several **IF nodes** use `$runIndex < 4` to retry earlier steps when:
  - tools were not used (no intermediate steps),
  - agent output is empty,
  - verification output is empty,
  - hallucinations persist (looping behavior capped by run index checks).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception (Webhook entry)
**Overview:** Receives a research prompt and starts the discovery agent.  
**Nodes involved:**
- Trigger legal research request (Webhook)

#### Node: Trigger legal research request (Webhook)
- **Type / role:** `n8n-nodes-base.webhook` — workflow entry point.
- **Configuration (interpreted):**
  - **Path:** `legal-research`
  - **Method:** `POST`
  - **Response mode:** `streaming` (client gets streaming output as workflow runs).
- **Key data used later:**
  - User query: `$('Trigger legal research request (Webhook)').item.json.body.prompt`
- **Connections:**
  - **Output →** Step 1: Discovery
- **Edge cases / failures:**
  - Missing `body.prompt` → downstream agents still run (system messages prohibit asking questions); output quality may degrade.
  - Streaming response + long LLM work → potential reverse proxy timeouts depending on hosting.

---

### Block 2.2 — Discovery (AI agent + discovery tools)
**Overview:** An AI agent runs broad searches across multiple sources; the pipeline expects tool calls and captures intermediate steps.  
**Nodes involved:**
- Step 1: Discovery
- Discovery tools:
  - LegiScan Discovery
  - Court Listener Discovery
  - Document Cloud Discovery
  - Google Search Discovery (Serper)
  - Plural Discovery1 (OpenStates v3)
- LLM models:
  - Qwen3
  - Auto Fallback
- Retry control:
  - Retry if Tools Not Used

#### Node: Step 1: Discovery
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LangChain agent performing discovery.
- **Configuration (interpreted):**
  - **User prompt template:** includes current time and webhook `body.prompt`.
  - **System message:** “DISCOVERY AGENT” requiring **ALL tools** (LegiScan, CourtListener, DocumentCloud Search, Serper).
  - **Max iterations:** 10
  - **Streaming enabled:** true
  - **Return intermediate steps:** true (critical: used to verify tools were called and to pass data to later agents).
- **Connections:**
  - **Main output →** Retry if Tools Not Used
  - **AI language model input:** Qwen3 (index 0) + Auto Fallback (index 1)
  - **AI tools available:** LegiScan Discovery, Court Listener Discovery, Document Cloud Discovery, Google Search Discovery, Plural Discovery1
- **Edge cases / failures:**
  - Agent may fail to call tools due to model behavior; handled by “Retry if Tools Not Used”.
  - Tool auth errors (LegiScan/CourtListener/Serper/Plural) → intermediateSteps may contain errors but still non-empty; prioritization must cope.
  - High iteration count + multiple HTTP calls → runtime and API quota risks.

#### Node: Retry if Tools Not Used
- **Type / role:** `n8n-nodes-base.if` — retry gate if no tools were invoked.
- **Configuration:**
  - Condition 1: `={{ $json.intermediateSteps[0] }}` **is empty** (object empty)
  - Condition 2: `={{ $runIndex }} < 4`
- **Connections:**
  - **True →** Step 1: Discovery (re-run) AND Step 2: Prioritization (parallel path)
    - In practice, this acts like a loop control for discovery; it also allows progress even if discovery is thin.
  - **False →** (not explicitly connected; workflow continues from the existing chain via other nodes’ connections)
- **Edge cases:**
  - `$json.intermediateSteps` missing entirely may evaluate as empty and cause retries until runIndex cap.
  - Because this IF is wired to both Step 1 and Step 2, it can produce concurrent downstream runs in some scenarios.

#### Node: LegiScan Discovery
- **Type / role:** `n8n-nodes-base.httpRequestTool` — tool for the discovery agent to call LegiScan search.
- **Auth:** `httpQueryAuth` credential “Legiscan Key” (API key in query string).
- **Configuration choices:**
  - URL is provided by the agent using `$fromAI('URL')`.
  - Batch size 1 (tool calls one URL at a time).
  - Tool description strongly enforces **full base URL**: `https://api.legiscan.com/?op=...`
- **Connections:** available as `ai_tool` to Step 1.
- **Failure modes:**
  - Missing base URL → likely request failure; description tries to prevent this.
  - Rate limiting or invalid key → 401/429.
  - LegiScan response structure varies; prioritization must not assume fixed keys beyond `status/searchresult`.

#### Node: Court Listener Discovery
- **Type / role:** `n8n-nodes-base.httpRequestTool` — tool to query CourtListener search/dockets.
- **Auth:** `httpHeaderAuth` credential “CourtListener Key” (header-based).
- **Configuration:**
  - Agent supplies full URL via `$fromAI('URL')`.
  - Sends `Accept: application/json`.
  - Description includes corrected endpoint: `/api/rest/v4/search/?type=o&q=...`
- **Connections:** `ai_tool` to Step 1.
- **Failure modes:**
  - Wrong endpoint or missing base URL.
  - Pagination via `next` link not followed unless agent chooses to.
  - Some opinions may lack `plain_text` later; retrieval must handle missing fields.

#### Node: Document Cloud Discovery
- **Type / role:** `n8n-nodes-base.httpRequestTool` — search DocumentCloud API.
- **Auth:** none configured in node (DocumentCloud search can be public; some queries may require auth depending on settings).
- **Configuration:**
  - Agent provides URL with `$fromAI('URL')`.
  - Description specifies pattern: `https://api.www.documentcloud.org/api/documents/search/?q={query}&per_page=25`
  - Critical: save both `id` and `slug` for later retrieval.
- **Connections:** `ai_tool` to Step 1.
- **Failure modes:**
  - Some documents not accessible publicly or missing S3 text export.
  - API pagination (`page`) may be needed for completeness.

#### Node: Google Search Discovery
- **Type / role:** `@n8n/n8n-nodes-langchain.toolHttpRequest` — Serper.dev wrapper tool for web/news/scholar search.
- **Auth:** `httpHeaderAuth` credential “Serper API”.
- **Configuration:**
  - URL template: `https://google.serper.dev/{endpoint}`
  - JSON body query is expected (Content-Type: application/json).
  - Placeholder `{endpoint}` must be chosen by agent: `search`, `news`, `scholar`, etc.
- **Connections:** `ai_tool` to Step 1.
- **Failure modes:**
  - Wrong endpoint string → 404.
  - Serper quota/429; or blocked query patterns.

#### Node: Plural Discovery1
- **Type / role:** `n8n-nodes-base.httpRequestTool` — OpenStates v3 (Plural) discovery.
- **Auth:** `httpHeaderAuth` credential “Plural API Key” (expects `X-API-KEY` or similar; tool description recommends header).
- **Configuration:**
  - Agent provides URL via `$fromAI('URL')`.
  - Tool description explains: `GET https://v3.openstates.org/bills` with `q=` or `jurisdiction=`.
  - Per-page must be **≤ 20**.
- **Connections:** `ai_tool` to Step 1.
- **Failure modes:**
  - Invalid jurisdiction/session strings; 400 responses.
  - Per-page > 20; request rejection.

#### Node: Qwen3
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — primary model for Step 1.
- **Model:** `qwen/qwen3-235b-a22b`
- **Temperature:** 0.8 (creative/broader exploration).
- **Connections:** provides `ai_languageModel` to Step 1.
- **Failure modes:** OpenRouter outages, model routing changes.

#### Node: Auto Fallback
- **Type / role:** OpenRouter “auto” fallback model for Step 1.
- **Model:** `openrouter/auto`
- **Temperature:** 0.8
- **Connections:** `ai_languageModel` second input to Step 1.
- **Failure modes:** auto-selection may choose smaller models with weaker tool-following.

---

### Block 2.3 — Prioritization (AI agent + Think tool)
**Overview:** Uses discovery intermediate steps to score and choose the best items for retrieval.  
**Nodes involved:**
- Step 2: Prioritization
- Think Tool Prioritization
- Sonnet 4.5
- Auto Fallback1
- Retry if Response Empty

#### Node: Step 2: Prioritization
- **Type / role:** LangChain agent.
- **Configuration:**
  - Input text includes:
    - Current time
    - A **cleaned and size-controlled JSON** of discovery `intermediateSteps` (with truncation logic up to 2,400,000 chars).
    - The original user query.
  - System message requires using **Think Tool** to score and rank; outputs a “SELECTED FOR RETRIEVAL” report listing IDs/URLs.
  - Max iterations: 10
  - Streaming: true
  - Return intermediate steps: true
- **Connections:**
  - **Main output →** Retry if Response Empty
  - **AI tool →** Think Tool Prioritization
  - **AI language model:** Sonnet 4.5 + Auto Fallback1
- **Edge cases:**
  - Discovery responses may be truncated; prioritization might miss items beyond truncation boundary.
  - Output formatting must be parseable by retrieval agent, but there is no strict parser here; retrieval relies on the LLM reading its own previous output.

#### Node: Think Tool Prioritization
- **Type / role:** `@n8n/n8n-nodes-langchain.toolThink` — structured reasoning tool.
- **Configuration:** Strong rubric:
  - Count totals found
  - Score items 1–10
  - Select 3–10 with reasons
  - Document what was skipped
- **Connections:** `ai_tool` to Step 2.
- **Failure modes:** None at HTTP level; failure is “agent doesn’t use it” (mitigated by system message + retry logic earlier).

#### Node: Retry if Response Empty
- **Type / role:** IF gate for empty intermediate steps (same pattern as discovery).
- **Configuration:**
  - Checks `={{ $json.intermediateSteps[0] }}` empty AND `$runIndex < 4`
- **Connections:**
  - **True →** Step 2: Prioritization (retry) AND Step 3: Retrieval (also triggered)
- **Risk:** Can cause parallel downstream starts even when prioritization is incomplete.

#### Node: Sonnet 4.5
- **Type / role:** OpenRouter model for Step 2.
- **Model:** `anthropic/claude-sonnet-4.5`
- **Temperature:** 0.3 (more deterministic, better tool-following).

#### Node: Auto Fallback1
- **Type / role:** OpenRouter auto model fallback for Step 2.
- **Temperature:** 0.3

---

### Block 2.4 — Retrieval (AI agent + retrieval tools)
**Overview:** Retrieves full text/details for the prioritized items only (no new searching).  
**Nodes involved:**
- Step 3: Retrieval
- Retrieval tools:
  - LegiScan Retrieval
  - Court Listener Retrieveal (typo in name)
  - DocumentCloud Retrieval
  - Jina URL Text Extraction
  - Plural Retrieval
- LLM models:
  - Qwen4
  - Auto Fallback2
- Retry control:
  - Retry if Tools Not Used1

#### Node: Step 3: Retrieval
- **Type / role:** LangChain agent.
- **Configuration:**
  - Input includes `JSON.stringify($json.output).slice(0, 1000000)` (takes prioritization output; hard cut at 1,000,000 chars).
  - System message: “RETRIEVAL AGENT” requires using all tools and retrieving full texts.
  - Max iterations: 10, streaming, return intermediate steps: true
- **Connections:**
  - **Main output →** Retry if Tools Not Used1
  - **AI tools:** all retrieval tools listed above
  - **AI language model:** Qwen4 + Auto Fallback2
- **Edge cases / failures:**
  - If prioritization output exceeds 1,000,000 chars, selection list might truncate.
  - Tools may return huge texts; later report writing truncates aggressively.
  - Some resources return PDFs or links rather than plain text; the workflow expects text where possible.

#### Node: Retry if Tools Not Used1
- **Type / role:** IF gate checking tool usage.
- **Configuration:** `intermediateSteps[0]` empty AND `$runIndex < 4`
- **Connections:**
  - **True →** Step 3: Retrieval (retry) AND Step 4: Report Writing (can start even on partial retrieval).
- **Risk:** Can cascade partial retrieval into report writing.

#### Node: LegiScan Retrieval
- **Type / role:** HTTP Request Tool (LegiScan getBill/getBillText).
- **Auth:** `httpQueryAuth` “Legiscan Key”.
- **Tool behavior guidance:**
  - Retrieve only selected bills: `https://api.legiscan.com/?op=getBill&id={bill_id}`
  - To get text: `?op=getBillText&id={doc_id}`
- **Failure modes:**
  - Bill text sometimes only via documents/links; doc_id usage must match LegiScan response.
  - Large responses or encoded content (PDF base64 in some APIs) may not be handled.

#### Node: Court Listener Retrieveal
- **Type / role:** HTTP Request Tool for opinions/clusters.
- **Auth:** `httpHeaderAuth` “CourtListener Key”.
- **Guidance:** Use fields query to limit to `plain_text, html_with_citations, case_name`.
- **Failure modes:**
  - Some opinions may not have `plain_text`.
  - Cluster vs opinion ID confusion.

#### Node: DocumentCloud Retrieval
- **Type / role:** HTTP Request Tool to fetch S3-hosted extracted text JSON.
- **Guidance:** `https://s3.documentcloud.org/documents/{id}/{slug}.txt.json`
- **Failure modes:**
  - 404 if slug mismatch or text export unavailable.
  - Multi-page content in `pages[]` can be large.

#### Node: Jina URL Text Extraction
- **Type / role:** `n8n-nodes-base.jinaAiTool` — article/content extraction.
- **Auth:** `jinaAiApi` credential “Jina AI account”.
- **Failure modes:**
  - Some sites block scrapers; extraction may return partial content.
  - Non-HTML or paywalled pages.

#### Node: Plural Retrieval
- **Type / role:** OpenStates v3 bill detail retrieval.
- **Auth:** `httpHeaderAuth` “Plural API Key”.
- **Guidance:**
  - Bill detail by UUID: `GET https://v3.openstates.org/bills/ocd-bill/{uuid}?include=...`
  - Full bill text is not returned directly; follow `versions[].url`.
- **Failure modes:**
  - The workflow does **not** include a dedicated tool to download `versions[].url` PDFs/HTML; the agent would need to fetch those via another tool (not provided) unless the URL is compatible with Jina.
  - Include parameter must be repeated, not comma-separated.

#### Node: Qwen4
- **Type / role:** OpenRouter model for Step 3.
- **Model:** `qwen/qwen3-235b-a22b`
- **Temperature:** 0.2 (tool-following / precision)

#### Node: Auto Fallback2
- **Type / role:** OpenRouter auto fallback for Step 3.
- **Temperature:** 0.2

---

### Block 2.5 — Report Writing (AI agent + analysis Think tool)
**Overview:** Synthesizes retrieved texts into a comprehensive strategic legal intelligence report with strict citation requirements.  
**Nodes involved:**
- Step 4: Report Writing
- Think Tool Analysis
- Opus
- Auto Fallback3
- Retry if Response Empty1

#### Node: Step 4: Report Writing
- **Type / role:** LangChain agent.
- **Configuration:**
  - Input is a cleaned representation of retrieval `intermediateSteps` with a **500,000 char** budget and per-response truncation.
  - System message enforces:
    - One-shot final report
    - No asking questions
    - Extensive inline citations with full URLs
    - Very long, detailed output encouraged
- **Connections:**
  - **Main output →** Retry if Response Empty1
  - **AI tool →** Think Tool Analysis (intended to help synthesis)
  - **AI language model:** Opus + Auto Fallback3
- **Edge cases:**
  - Citations: retrieved tools may not provide clean URLs (e.g., CourtListener `absolute_url` is partial). Agent must construct full URLs; not guaranteed.
  - Truncation of retrieved texts can cause missing support → increases hallucination risk.

#### Node: Think Tool Analysis
- **Type / role:** Think tool for deep synthesis.
- **Connections:** `ai_tool` to Step 4.

#### Node: Retry if Response Empty1
- **Type / role:** IF gate on empty report output.
- **Configuration:** `={{ $json.output }}` is empty string AND `$runIndex < 4`
- **Connections:**
  - **True →** Step 4: Report Writing (retry) AND Set Report
- **Note:** Sending empty output into Set Report can break verification semantics (verification expects a report).

#### Node: Opus
- **Type / role:** OpenRouter model for Step 4.
- **Model:** `anthropic/claude-opus-4.5`
- **Temperature:** 0.6

#### Node: Auto Fallback3
- **Type / role:** OpenRouter auto fallback for Step 4.
- **Temperature:** 0.6

---

### Block 2.6 — Packaging for Verification
**Overview:** Converts report output and retrieved documents into two fields for the verification chain.  
**Nodes involved:**
- Set Report

#### Node: Set Report
- **Type / role:** `n8n-nodes-base.set` — builds normalized fields.
- **Configuration (key assignments):**
  - **Final Report** = `={{ $json.output }}`
  - **Retrieved Documents** = custom JS expression that:
    - Reads `$('Step 3: Retrieval').item.json.intermediateSteps`
    - Normalizes each step into `{step, tool, input, response}`
    - Enforces total 400,000 char budget; per-response dynamic truncation
- **Connections:**
  - **Main output →** Step 5: Verification
  - Also referenced later by Set Output and Fixing agent.
- **Failure modes:**
  - If Step 3 has no `intermediateSteps`, returns “No retrieval data available”.
  - If there are many large tool outputs, truncation may remove key evidence needed for verification.

---

### Block 2.7 — Verification (LLM chain + structured parser + routing)
**Overview:** Verifies only material legal fabrications by comparing report content to retrieved documents; returns a structured JSON verdict; routes to fix step if hallucinations exist.  
**Nodes involved:**
- Step 5: Verification
- Structured Output Parser
- If Empty Output1
- If hallucinations present
- Sonnet 4.
- Auto Fallback4

#### Node: Step 5: Verification
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — single LLM chain call.
- **Configuration:**
  - Large prompt with strong policy: be permissive about legal analysis and strategy; only flag **clear factual legal fabrications**.
  - Includes two large injected variables:
    - `Retrieved Documents` from Set Report (up to ~2.4M chars)
    - `Final Report` from Set Report (up to 800k chars)
  - Uses truncation at “natural boundaries” to avoid mid-structure breaks.
  - `needsFallback: true`, output parser enabled.
- **Connections:**
  - **Main →** If Empty Output1
  - **AI output parser →** Structured Output Parser
  - **AI language model:** Sonnet 4. (primary) + Auto Fallback4 (secondary)
- **Failure modes:**
  - If report is extremely long, truncation can hide hallucinations in the omitted portion.
  - If retrieved docs are truncated too much, verifier may flag valid claims as unsupported.

#### Node: Structured Output Parser
- **Type / role:** Structured JSON schema enforcement (`outputParserStructured`).
- **Schema:** `contains_hallucinations` (boolean), `hallucinations` (array), `summary` (string), with severity enum.
- **autoFix:** true (attempts to repair model output into valid schema).
- **Connections:**
  - Feeds parsed output back into verification chain context.
- **Failure modes:**
  - If the LLM output is too malformed, auto-fix may still fail; downstream `$json.output.contains_hallucinations` will break.

#### Node: If Empty Output1
- **Type / role:** IF guard when verification returns empty output object.
- **Configuration:** checks `={{ $json.output }}` object empty (loose validation).
- **Connections:**
  - **True →** Step 5: Verification (retry) AND If hallucinations present
- **Risk:** “If hallucinations present” expects parsed structured output; running it when output is empty can cause expression errors.

#### Node: If hallucinations present
- **Type / role:** IF router for hallucination handling.
- **Configuration:**
  - Condition A: `={{ $json.output.contains_hallucinations }}` is true
  - Condition B: `$runIndex < 4`
- **Connections:**
  - **True →** Step 6: Fixing Hallucinations AND Set Output (both)
- **Edge cases:**
  - `contains_hallucinations` missing → expression evaluation error or false depending on n8n behavior; likely failure.

#### Node: Sonnet 4.
- **Type / role:** OpenRouter model for Step 5.
- **Model:** `anthropic/claude-sonnet-4.5`
- **Temperature:** 0.2

#### Node: Auto Fallback4
- **Type / role:** OpenRouter auto fallback for verification + parser.
- **Temperature:** 0.2

---

### Block 2.8 — Fixing Hallucinations (rewrite loop + output selection)
**Overview:** If hallucinations were found, an agent rewrites the report using retrieved documents and the hallucination list, then returns the fixed report.  
**Nodes involved:**
- Step 6: Fixing Hallucinations
- Think Tool Analysis1
- If Empty Output
- Set Output
- Opus1
- Auto Fallback5
- (Optional extra model nodes are connected as fallbacks)

#### Node: Step 6: Fixing Hallucinations
- **Type / role:** LangChain agent.
- **Configuration:**
  - Inputs:
    - Original user query
    - Retrieved documents (Step 3 intermediateSteps, truncated to 1.6M chars)
    - Failed report (`Set Report.Final Report`, up to 800k chars)
    - Hallucination list from verifier (up to 80k chars)
  - System message: rewrite agent must fix/remove all hallucinations, preserve valid strategy, enforce citations, one-shot output.
- **Connections:**
  - **Main output →** If Empty Output
  - **AI tool →** Think Tool Analysis1
  - **AI language model:** Opus1 + Auto Fallback5
- **Failure modes:**
  - If hallucination list references text not present due to truncation, the agent may not be able to “find and remove” precisely.
  - If retrieved docs are truncated, fixing may require removal rather than correction.

#### Node: If Empty Output
- **Type / role:** IF guard if rewrite output is empty.
- **Configuration:** `={{ $json.output }}` empty string
- **Connections:**
  - **True →** Step 6: Fixing Hallucinations (retry) AND Set Report
- **Risk:** can overwrite Set Report with empty/partial report depending on execution ordering.

#### Node: Set Output
- **Type / role:** final packaging set node for returning the final report.
- **Configuration:**
  - **Final Report** = `={{ $('Set Report').item.json['Final Report'] }}`
  - Note: This takes the report from **Set Report**, not directly from Step 6 output.
- **Connections:** none shown after Set Output (in this JSON, there is no explicit “Respond to Webhook” node; streaming webhook will stream execution logs/outputs depending on n8n behavior).
- **Edge case:**
  - If Step 6 creates a fixed report but Set Report is not updated with it, Set Output may return the old report. In the current wiring, Step 6 does not directly feed Set Report unless “If Empty Output” triggers Set Report. This is a potential logic bug.

#### Node: Think Tool Analysis1
- **Type / role:** Think tool for synthesis during rewrite (same text as Think Tool Analysis).
- **Connections:** `ai_tool` to Step 6.

#### Node: Opus1
- **Type / role:** OpenRouter model for Step 6.
- **Model:** `anthropic/claude-opus-4.5`
- **Temperature:** 0.1

#### Node: Auto Fallback5
- **Type / role:** OpenRouter auto fallback for Step 6.
- **Temperature:** 0.1

---

### Block 2.9 — Documentation / UI notes (Sticky Notes)
**Overview:** Visual grouping and explanation only; no execution impact.  
**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger legal research request (Webhook) | n8n-nodes-base.webhook | Entry point: receives prompt via POST | — | Step 1: Discovery | ## Research US regulations using AI agents with web search and report generation / How it works / Setup steps (full note content) |
| Step 1: Discovery | @n8n/n8n-nodes-langchain.agent | AI agent: broad discovery across sources | Trigger legal research request (Webhook); (Retry if Tools Not Used loop) | Retry if Tools Not Used | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| LegiScan Discovery | n8n-nodes-base.httpRequestTool | Discovery tool: search legislation via LegiScan | Step 1: Discovery (ai_tool) | Step 1: Discovery (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Court Listener Discovery | n8n-nodes-base.httpRequestTool | Discovery tool: search opinions/dockets via CourtListener | Step 1: Discovery (ai_tool) | Step 1: Discovery (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Document Cloud Discovery | n8n-nodes-base.httpRequestTool | Discovery tool: search DocumentCloud documents | Step 1: Discovery (ai_tool) | Step 1: Discovery (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Google Search Discovery | @n8n/n8n-nodes-langchain.toolHttpRequest | Discovery tool: Serper web/news/scholar search | Step 1: Discovery (ai_tool) | Step 1: Discovery (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Plural Discovery1 | n8n-nodes-base.httpRequestTool | Discovery tool: OpenStates v3 bills search | Step 1: Discovery (ai_tool) | Step 1: Discovery (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Qwen3 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for discovery agent | — | Step 1: Discovery (ai_languageModel) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Auto Fallback | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback LLM for discovery agent | — | Step 1: Discovery (ai_languageModel) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Retry if Tools Not Used | n8n-nodes-base.if | Retry discovery if no tools invoked | Step 1: Discovery | Step 1: Discovery; Step 2: Prioritization | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Step 2: Prioritization | @n8n/n8n-nodes-langchain.agent | AI agent: score/select items for retrieval | Retry if Tools Not Used; (Retry if Response Empty loop) | Retry if Response Empty | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Think Tool Prioritization | @n8n/n8n-nodes-langchain.toolThink | Tool: structured scoring & selection | Step 2: Prioritization (ai_tool) | Step 2: Prioritization (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Sonnet 4.5 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for prioritization agent | — | Step 2: Prioritization (ai_languageModel) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Auto Fallback1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback LLM for prioritization agent | — | Step 2: Prioritization (ai_languageModel) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Retry if Response Empty | n8n-nodes-base.if | Retry prioritization if no intermediate steps | Step 2: Prioritization | Step 2: Prioritization; Step 3: Retrieval | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Step 3: Retrieval | @n8n/n8n-nodes-langchain.agent | AI agent: retrieve full texts for selected items | Retry if Response Empty; (Retry if Tools Not Used1 loop) | Retry if Tools Not Used1 | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| LegiScan Retrieval | n8n-nodes-base.httpRequestTool | Retrieval tool: get bill details/text | Step 3: Retrieval (ai_tool) | Step 3: Retrieval (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Court Listener Retrieveal | n8n-nodes-base.httpRequestTool | Retrieval tool: get opinion/clusters full text | Step 3: Retrieval (ai_tool) | Step 3: Retrieval (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| DocumentCloud Retrieval | n8n-nodes-base.httpRequestTool | Retrieval tool: get DocumentCloud S3 text JSON | Step 3: Retrieval (ai_tool) | Step 3: Retrieval (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Jina URL Text Extraction | n8n-nodes-base.jinaAiTool | Retrieval tool: extract main text from URLs | Step 3: Retrieval (ai_tool) | Step 3: Retrieval (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Plural Retrieval | n8n-nodes-base.httpRequestTool | Retrieval tool: OpenStates v3 bill detail | Step 3: Retrieval (ai_tool) | Step 3: Retrieval (tool return) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Qwen4 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for retrieval agent | — | Step 3: Retrieval (ai_languageModel) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Auto Fallback2 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback LLM for retrieval agent | — | Step 3: Retrieval (ai_languageModel) | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Retry if Tools Not Used1 | n8n-nodes-base.if | Retry retrieval if no tools invoked | Step 3: Retrieval | Step 3: Retrieval; Step 4: Report Writing | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Step 4: Report Writing | @n8n/n8n-nodes-langchain.agent | AI agent: write cited legal intelligence report | Retry if Tools Not Used1; (Retry if Response Empty1 loop) | Retry if Response Empty1 | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Think Tool Analysis | @n8n/n8n-nodes-langchain.toolThink | Tool: synthesis guidance | Step 4: Report Writing (ai_tool) | Step 4: Report Writing (tool return) | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Opus | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for report writing agent | — | Step 4: Report Writing (ai_languageModel) | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Auto Fallback3 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback LLM for report writing agent | — | Step 4: Report Writing (ai_languageModel) | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Retry if Response Empty1 | n8n-nodes-base.if | Retry report writing if output empty | Step 4: Report Writing | Step 4: Report Writing; Set Report | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Set Report | n8n-nodes-base.set | Package “Final Report” + “Retrieved Documents” for verification | Retry if Response Empty1; If Empty Output (fix loop) | Step 5: Verification | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Step 5: Verification | @n8n/n8n-nodes-langchain.chainLlm | Verify report vs retrieved docs (material legal errors only) | Set Report; (If Empty Output1 loop) | If Empty Output1 | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce verification output schema | Auto Fallback4 (ai_languageModel via parser); Step 5: Verification (ai_outputParser) | Step 5: Verification | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Sonnet 4. | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for verification | — | Step 5: Verification (ai_languageModel) | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Auto Fallback4 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback LLM for verification + parser | — | Step 5: Verification; Structured Output Parser | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| If Empty Output1 | n8n-nodes-base.if | Retry verification if output empty | Step 5: Verification | Step 5: Verification; If hallucinations present | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| If hallucinations present | n8n-nodes-base.if | Branch to rewrite if hallucinations detected | If Empty Output1 | Step 6: Fixing Hallucinations; Set Output | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Step 6: Fixing Hallucinations | @n8n/n8n-nodes-langchain.agent | Rewrite report to remove/correct hallucinations | If hallucinations present; (If Empty Output loop) | If Empty Output | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Think Tool Analysis1 | @n8n/n8n-nodes-langchain.toolThink | Tool: synthesis guidance during rewrite | Step 6: Fixing Hallucinations (ai_tool) | Step 6: Fixing Hallucinations (tool return) | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Opus1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for rewrite agent | — | Step 6: Fixing Hallucinations (ai_languageModel) | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Auto Fallback5 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback LLM for rewrite agent | — | Step 6: Fixing Hallucinations (ai_languageModel) | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| If Empty Output | n8n-nodes-base.if | Retry rewrite if output empty | Step 6: Fixing Hallucinations | Step 6: Fixing Hallucinations; Set Report | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Set Output | n8n-nodes-base.set | Output packaging (Final Report) | If hallucinations present | — | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas annotation | — | — | ## Research & Tool Use / This section of the workflow focuses on finding the information that the next section will use to write and verify its report |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas annotation | — | — | ## Report Writing & Verification / This section of the workflow focuses on writing the user-facing report and ensuring it is accurate and free from hallucinations |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas annotation | — | — | ## Research US regulations using AI agents with web search and report generation (full note content with setup steps) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (keep it inactive until credentials are set).

2. **Add Webhook trigger**
   - Node: **Webhook**
   - Method: **POST**
   - Path: `legal-research`
   - Response mode: **Streaming**
   - Expect request body JSON like: `{ "prompt": "..." }`

3. **Add OpenRouter LLM nodes (models)**
   - Add the following nodes (type: **OpenRouter Chat Model** / `lmChatOpenRouter`):
     - `Qwen3` → model `qwen/qwen3-235b-a22b`, temp `0.8`
     - `Sonnet 4.5` → model `anthropic/claude-sonnet-4.5`, temp `0.3`
     - `Qwen4` → model `qwen/qwen3-235b-a22b`, temp `0.2`
     - `Opus` → model `anthropic/claude-opus-4.5`, temp `0.6`
     - `Sonnet 4.` → model `anthropic/claude-sonnet-4.5`, temp `0.2`
     - `Opus1` → model `anthropic/claude-opus-4.5`, temp `0.1`
     - `Auto Fallback`, `Auto Fallback1`, `Auto Fallback2`, `Auto Fallback3`, `Auto Fallback4`, `Auto Fallback5` → model `openrouter/auto` with temperatures as in the workflow (0.8/0.3/0.2/0.6/0.2/0.1)
   - **Credential:** OpenRouter API key credential (one shared credential is fine).

4. **Create Discovery tools**
   - Add **HTTP Request Tool** nodes:
     - `LegiScan Discovery` (auth: **Query Auth** API key)
     - `Court Listener Discovery` (auth: **Header Auth**)
     - `Document Cloud Discovery` (no auth unless needed)
     - `Plural Discovery1` (auth: **Header Auth** API key)
   - Add **Serper tool** node:
     - `Google Search Discovery` (LangChain tool HTTP request)
       - URL: `https://google.serper.dev/{endpoint}`
       - Set `Content-Type: application/json`
       - Header auth for Serper key
   - Ensure each tool uses agent-provided URL input (AI tool pattern), and include strong tool descriptions to enforce correct endpoints/base URLs.

5. **Add “Step 1: Discovery” agent**
   - Node type: **LangChain Agent**
   - Text: include `{{ $now }}` and the webhook prompt:  
     `{{ $('Trigger legal research request (Webhook)').item.json.body.prompt }}`
   - System message: discovery rules (must use all tools).
   - Enable:
     - **returnIntermediateSteps = true**
     - **streaming = true**
     - **maxIterations = 10**
   - Connect **AI Language Model**: `Qwen3` (primary) and `Auto Fallback` (secondary).
   - Attach **AI Tools**: LegiScan Discovery, Court Listener Discovery, Document Cloud Discovery, Google Search Discovery, Plural Discovery1.
   - Connect Webhook → Step 1.

6. **Add Retry gate for discovery**
   - Node type: **IF**
   - Condition: `{{$json.intermediateSteps[0]}}` is empty AND `{{$runIndex}} < 4`
   - Wire Step 1 → IF.
   - IF true outputs back to Step 1 (loop).

7. **Create Prioritization Think tool**
   - Node type: **LangChain Think Tool**
   - Name: `Think Tool Prioritization`
   - Description: scoring rubric and required selection output format.

8. **Add “Step 2: Prioritization” agent**
   - Node type: **LangChain Agent**
   - Text: inject discovery intermediateSteps (use a truncation/normalization expression similar to workflow) + original query.
   - System message: must use Think Tool.
   - Enable intermediate steps + streaming, maxIterations 10.
   - Connect AI Language Model: `Sonnet 4.5` + `Auto Fallback1`.
   - Attach AI Tool: `Think Tool Prioritization`.
   - Connect Discovery retry IF (false path or direct continuation) → Step 2.

9. **Add Retry gate for prioritization**
   - IF node: checks `intermediateSteps[0]` empty and `$runIndex < 4`.
   - Connect Step 2 → this IF.
   - Loop true back to Step 2.

10. **Create Retrieval tools**
    - Add HTTP Request Tool nodes:
      - `LegiScan Retrieval` (Query Auth)
      - `Court Listener Retrieveal` (Header Auth)
      - `DocumentCloud Retrieval` (no auth)
      - `Plural Retrieval` (Header Auth)
    - Add `Jina URL Text Extraction` (Jina AI tool node) with Jina API credentials.

11. **Add “Step 3: Retrieval” agent**
    - Text: feed prioritization output (cap/truncate if desired).
    - System message: retrieval only, must use all tools, no new searching.
    - Enable intermediate steps + streaming.
    - Connect AI Language Model: `Qwen4` + `Auto Fallback2`.
    - Attach AI tools: all retrieval tools above.
    - Connect prioritization retry IF (false path) → Step 3.

12. **Add Retry gate for retrieval**
    - IF node: `intermediateSteps[0]` empty and `$runIndex < 4`.
    - Connect Step 3 → IF; loop true to Step 3.

13. **Create Report Writing Think tool**
    - Node type: **Think Tool**
    - Name: `Think Tool Analysis`
    - Description: synthesis/patterns/gaps/recommendations rubric.

14. **Add “Step 4: Report Writing” agent**
    - Text: feed retrieval intermediateSteps, normalized and truncated.
    - System message: strict citations with full URLs; long report; one-shot.
    - Enable intermediate steps + streaming.
    - Connect AI Language Model: `Opus` + `Auto Fallback3`.
    - Attach AI tool: `Think Tool Analysis`.
    - Connect retrieval retry IF (false path) → Step 4.

15. **Add Retry gate for empty report**
    - IF node: `{{$json.output}}` empty and `$runIndex < 4`.
    - Connect Step 4 → IF; loop true to Step 4.

16. **Add “Set Report” node**
    - Node type: **Set**
    - Add fields:
      - `Final Report` = `{{$json.output}}`
      - `Retrieved Documents` = expression that extracts and truncates Step 3’s `intermediateSteps` into a JSON string.
    - Connect Step 4 (or its retry IF false path) → Set Report.

17. **Add Verification chain**
    - Node type: **Chain LLM** (`chainLlm`)
    - Prompt: verification instructions + inject `Retrieved Documents` and `Final Report` from Set Report with truncation.
    - Connect AI Language Model: `Sonnet 4.` + `Auto Fallback4`.

18. **Add Structured Output Parser**
    - Node type: **Structured Output Parser**
    - Schema: the boolean + hallucinations array + summary fields.
    - Connect parser as `ai_outputParser` into the verification chain.

19. **Add verification empty-output IF**
    - IF node checks `{{$json.output}}` object empty.
    - Connect Step 5 → IF; loop true to Step 5.

20. **Add hallucination router**
    - IF node checks:
      - `{{$json.output.contains_hallucinations}}` is true
      - `$runIndex < 4`
    - True path → Step 6: Fixing Hallucinations; also to Set Output (as in workflow).

21. **Add “Step 6: Fixing Hallucinations” agent**
    - Text: include user query, retrieved docs, failed report, and hallucination list.
    - System message: rewrite/fix all hallucinations, preserve valid strategy, strict citations.
    - Connect AI Language Model: `Opus1` + `Auto Fallback5`.
    - Attach tool: `Think Tool Analysis1` (a duplicate Think tool is acceptable).

22. **Add IF gate for empty rewrite output**
    - IF checks `{{$json.output}}` empty string.
    - Loop true back to Step 6.
    - (If you want the fixed report to actually be returned, ensure Step 6 output is written back into the final output—see note below.)

23. **Add Set Output node**
    - Node type: **Set**
    - Field: `Final Report`
      - If you want **verified-or-fixed** behavior, set this to:
        - If hallucinations: use Step 6 output
        - Else: use Step 4 output
      - The provided workflow instead pulls from `Set Report.Final Report`, which may not incorporate Step 6 changes unless you explicitly re-run Set Report with Step 6 output.

24. **Credentials to configure**
    - **OpenRouter API** for all LLM nodes.
    - **LegiScan** API key (Query Auth).
    - **CourtListener** API key (Header Auth).
    - **Serper.dev** API key (Header Auth).
    - **OpenStates/Plural** API key (Header Auth).
    - **Jina AI** API key.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer (applies to the whole workflow) |
| “This workflow accepts a research request via webhook… verifies the content… fixing step is triggered before returning the final output.” | Sticky Note2: overview & setup guidance (canvas note) |
| “This section of the workflow focuses on finding the information…” | Sticky Note: groups discovery/prioritization/retrieval area |
| “This section of the workflow focuses on writing the user-facing report…” | Sticky Note1: groups report writing/verification/fixing area |

--- 

### Important implementation observation (logic risk)
In the current wiring, **Step 6 (fixed report) is not clearly persisted into `Set Report`**, while `Set Output` reads from `Set Report.Final Report`. Unless execution ordering happens to re-run Set Report with Step 6 output (it only does so on “If Empty Output” true), the workflow may return the **unfixed** report even after a rewrite. To make the fix deterministic, connect Step 6 output into a dedicated “Set Fixed Report” and have Set Output choose between original vs fixed based on `contains_hallucinations`.