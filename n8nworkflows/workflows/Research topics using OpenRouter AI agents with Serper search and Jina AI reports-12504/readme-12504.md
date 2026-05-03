Research topics using OpenRouter AI agents with Serper search and Jina AI reports

https://n8nworkflows.xyz/workflows/research-topics-using-openrouter-ai-agents-with-serper-search-and-jina-ai-reports-12504


# Research topics using OpenRouter AI agents with Serper search and Jina AI reports

## 1. Workflow Overview

**Workflow name (internal):** *Research topics using AI agents with web search, verification, and report generation*  
**User-provided title:** *Research topics using OpenRouter AI agents with Serper search and Jina AI reports*  
**Primary purpose:** Accept a research prompt (via webhook or from another workflow), run an AI “research agent” that must call multiple external tools (web search + academic search + URL reading + social/profile scrapers), then generate a long Markdown report, verify it for material hallucinations using a second LLM pass with a structured JSON verdict, optionally rewrite to fix hallucinations, and finally return the report (streaming via webhook or as normal workflow output).

**Target use cases**
- Rapid OSINT-style topic research requiring citations and cross-checking
- Research synthesis combining web search snippets, full-page extraction, and academic sources
- Generating a comprehensive “intelligence report” with a verification/fix loop to reduce hallucinations

### 1.1 Entry & Prompt Normalization
- Two entry points: **Webhook** and **Execute Workflow Trigger**
- Normalizes input into `prompt` and marks `source` (“webhook” vs “workflow”)

### 1.2 Research Agent (Tool-Using)
- A LangChain Agent receives the prompt and is instructed to run **all tools** (mandatory parallel execution) and return intermediate steps (tool calls + observations).

### 1.3 Report Drafting (Writer Agent)
- A second Agent transforms the intermediate tool results into a comprehensive Markdown report.
- Includes retry logic if the agent output is empty.

### 1.4 Verification (LLM Chain + Structured Parser)
- Takes the drafted report + retrieved documents (intermediateSteps) and produces a verification verdict.
- Output is parsed against a JSON schema (auto-fix enabled).
- If hallucinations are detected, a dedicated fixing agent rewrites the report conservatively.

### 1.5 Delivery (Webhook response or workflow output)
- If request came from webhook, stream JSON response through **Respond to Webhook**.
- Otherwise set a `report` field as the workflow output.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggers & Prompt Setup
**Overview:** Receives requests from either an HTTP webhook or another workflow execution, then normalizes the prompt and sets a source flag used later for routing the response.

**Nodes involved:**
- Trigger research request (Webhook)
- Trigger research from another workflow
- Set Prompt

#### Node: Trigger research request (Webhook)
- **Type / role:** `n8n-nodes-base.webhook` — HTTP entry point.
- **Key configuration:**
  - **POST** `/general-research`
  - **Response mode:** `streaming` (pairs with Respond to Webhook streaming)
- **Inputs/outputs:**
  - No input.
  - Outputs to **Set Prompt**.
- **Failure/edge cases:**
  - Missing body/prompt → later `Set Prompt` yields empty string, which can lead to weak agent output.
  - Streaming requires n8n instance + client to support streaming responses properly.

#### Node: Trigger research from another workflow
- **Type / role:** `n8n-nodes-base.executeWorkflowTrigger` — internal trigger when called by Execute Workflow.
- **Key configuration:** Defines one input: `prompt`
- **Inputs/outputs:**
  - No input (trigger).
  - Outputs to **Set Prompt**.
- **Failure/edge cases:**
  - If caller does not pass `prompt`, downstream prompt becomes empty.

#### Node: Set Prompt
- **Type / role:** `n8n-nodes-base.set` — normalizes inbound data.
- **Key configuration choices:**
  - Sets `prompt` to:
    - `$json.prompt` OR `$json.body.prompt` OR `''`
  - Sets `source` to:
    - `'webhook'` if `$json.webhookUrl` exists, else `'workflow'`
- **Key expressions:**
  - `={{ $json.prompt || ($json.body && $json.body.prompt) || '' }}`
  - `={{ $json.webhookUrl ? 'webhook' : 'workflow' }}`
- **Inputs/outputs:**
  - Input from either trigger.
  - Output to **Searching For Information**.
- **Failure/edge cases:**
  - If inbound payload differs (e.g., `body.text`), prompt becomes empty unless updated.

---

### Block 2.2 — Tooling Layer (Agent Tools & LLM Models)
**Overview:** Provides the research agent with a set of callable tools (search, academic search, social scrapers, URL reader) and multiple OpenRouter chat models.

**Nodes involved:**
- Serper API 2
- Semantic Scholar 2
- Read URL content in Jina AI
- LinkedinScraper 2
- InstagramScraper 2
- XPostScraper 2
- XProfileScraper 2
- Think1
- Qwen3
- OpenRouter Chat Model

#### Node: Serper API 2
- **Type / role:** `n8n-nodes-base.httpRequestTool` — tool for the agent to call Google search via Serper.
- **Key configuration choices:**
  - POST `https://google.serper.dev/search`
  - Body parameter: `q` from `$fromAI(...)`
  - Response: `fullResponse: true`, `neverError: true` (prevents hard failures but can hide non-200 issues unless inspected)
  - Batching enabled with batchSize=1
  - Auth: HTTP Header auth credential (“Serper API”)
- **Key variables:**
  - `q = {{$fromAI('parameters0_Value', 'Plain text search query', 'string')}}`
- **Inputs/outputs:**
  - Connected as `ai_tool` into **Searching For Information**.
- **Failure/edge cases:**
  - Bad/expired API key → response may still “not error” at node level due to `neverError`, but content will indicate failure.
  - Rate limits / quota errors.
  - If the agent fails to populate `q`, tool call fails logically (but node tries to run).

#### Node: Semantic Scholar 2
- **Type / role:** `n8n-nodes-base.httpRequestTool` — academic paper search tool.
- **Key configuration choices:**
  - GET `https://api.semanticscholar.org/graph/v1/paper/search`
  - Query params:
    - `query` from `$fromAI(...)` (**mandatory** per tool description)
    - `limit` from `$fromAI(...)` (must be <= 100)
  - Response: `fullResponse: true`, `neverError: true`
- **Failure/edge cases:**
  - `limit` not numeric or >100 may be rejected by API.
  - Semantic Scholar API throttling.
  - Missing `query` from tool call causes failure.

#### Node: Read URL content in Jina AI
- **Type / role:** `n8n-nodes-base.jinaAiTool` — extracts cleaned page text from a URL.
- **Key configuration choices:**
  - `url` from `$fromAI('URL', '', 'string')`
  - `retryOnFail: true`
  - Credential: Jina AI account
- **Failure/edge cases:**
  - Invalid URL / blocked sites / paywalls → empty or partial extraction.
  - Large pages might time out depending on Jina limits.
  - If agent doesn’t pass URLs systematically, verification quality drops.

#### Node: LinkedinScraper 2
- **Type / role:** `n8n-nodes-base.httpRequestTool` — ScrapingDog LinkedIn profile API.
- **Key configuration choices:**
  - URL must be fully formed by agent via `$fromAI('URL', ...)`
  - Auth: query auth via ScrapingDog Key credential
  - Response: fullResponse + neverError
- **Failure/edge cases:**
  - Wrong URL formatting (missing `linkId`, wrong type) → API error.
  - LinkedIn anti-scraping changes; ScrapingDog availability/limits.

#### Node: InstagramScraper 2
- **Type / role:** `n8n-nodes-base.httpRequestTool` — ScrapingDog Instagram profile API.
- **Key configuration:** agent supplies complete URL, query-auth credential, fullResponse/neverError.
- **Failure/edge cases:**
  - Private accounts / rate limits / API errors returned as “successful” node execution due to `neverError`.

#### Node: XPostScraper 2
- **Type / role:** `n8n-nodes-base.httpRequestTool` — ScrapingDog X/Twitter post API.
- **Key configuration:** URL from agent including `tweetId` and `parsed=true`.
- **Failure/edge cases:** invalid tweetId, deleted tweets, API throttling.

#### Node: XProfileScraper 2
- **Type / role:** `n8n-nodes-base.httpRequestTool` — ScrapingDog X/Twitter profile API.
- **Key configuration:** URL from agent including `profileId` and `parsed=true`.
- **Failure/edge cases:** suspended/private accounts; API limits.

#### Node: Think1
- **Type / role:** `@n8n/n8n-nodes-langchain.toolThink` — “thinking” tool exposed to the research agent.
- **Configuration:** descriptive guidance only; used by agent for structured reasoning.
- **Inputs/outputs:** connected as `ai_tool` to **Searching For Information**.
- **Edge cases:** none (but can consume tokens if overused by the agent).

#### Node: Qwen3
- **Type / role:** `lmChatOpenRouter` — primary LLM for research agent.
- **Model:** `qwen/qwen3-235b-a22b`
- **Inputs/outputs:** connected as `ai_languageModel` to **Searching For Information**.
- **Failure/edge cases:** OpenRouter auth/quota; model availability.

#### Node: OpenRouter Chat Model
- **Type / role:** `lmChatOpenRouter` — alternate/fallback LLM path for research agent.
- **Model:** `openrouter/auto`
- **Inputs/outputs:** connected as an additional `ai_languageModel` to **Searching For Information** (index 1).
- **Edge cases:** same as above; auto-routing may choose different providers with varied latency.

---

### Block 2.3 — Research Agent Execution + “Tools not used” retry loop
**Overview:** Runs the research agent with strict instructions to call all tools; if it returns no intermediate tool steps, it retries up to 4 times (based on `$runIndex`).

**Nodes involved:**
- Searching For Information
- Retry if Tools Not Used

#### Node: Searching For Information
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — tool-using research agent.
- **Key configuration choices:**
  - Input text: `={{ $json.prompt }}`
  - Options:
    - `systemMessage`: long instruction set enforcing “Mandatory Parallel Tool Execution” of all listed tools
    - `enableStreaming: true`
    - `returnIntermediateSteps: true` (critical: later writer consumes `intermediateSteps`)
  - `onError: continueRegularOutput` (workflow continues even if node errors)
  - `needsFallback: true`
  - `retryOnFail: true`
- **Tools connected (ai_tool):**
  - Serper API 2, Semantic Scholar 2, Read URL content in Jina AI, LinkedinScraper 2, InstagramScraper 2, XPostScraper 2, XProfileScraper 2, Think1
- **Models connected (ai_languageModel):**
  - Qwen3 (index 0) and OpenRouter Chat Model (index 1)
- **Outputs:**
  - Main output to **Retry if Tools Not Used**
- **Failure/edge cases:**
  - Agent may not actually call all tools despite instruction (common with LLM agents). The workflow attempts to detect this by checking `intermediateSteps[0]`.
  - Tool nodes have `neverError` in HTTP tools, masking failures unless report explicitly inspects responses.
  - Streaming enabled: if downstream doesn’t stream, still okay but can increase memory usage.

#### Node: Retry if Tools Not Used
- **Type / role:** `n8n-nodes-base.if` — detects “no tool calls” and retries.
- **Condition logic:**
  - `{{$json.intermediateSteps[0]}}` is empty **AND**
  - `{{$runIndex}} < 4`
- **Outputs:**
  - **True** → loops back to **Searching For Information**
  - **False** → proceeds to **Writing Report**
- **Failure/edge cases:**
  - If intermediateSteps exists but tools failed silently (neverError), condition won’t trigger.
  - `$runIndex` counts node executions within current workflow run; after 4 attempts it stops retrying.

---

### Block 2.4 — Report Writing + Empty-response retry
**Overview:** Converts tool observations into a large report, with a retry loop if the agent returns an empty `output`.

**Nodes involved:**
- Writing Report
- Opus
- Auto Fallback3
- Think Tool Analysis
- Retry if Response Empty1

#### Node: Opus
- **Type / role:** `lmChatOpenRouter` — primary LLM for writing agent.
- **Model:** `anthropic/claude-opus-4.5`
- **Options:** temperature 0.6
- **Outputs:** feeds **Writing Report** as `ai_languageModel` (index 0)

#### Node: Auto Fallback3
- **Type / role:** `lmChatOpenRouter` — fallback model for writing agent.
- **Model:** `openrouter/auto`
- **Options:** temperature 0.6
- **Outputs:** feeds **Writing Report** as `ai_languageModel` (index 1)

#### Node: Think Tool Analysis
- **Type / role:** `toolThink` — additional analysis tool available to the writing agent.
- **Usage:** prompts the model to synthesize and find patterns/gaps/recommendations.
- **Outputs:** connected to **Writing Report** as `ai_tool`.

#### Node: Writing Report
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — report drafting agent.
- **Key configuration:**
  - Input text is a large expression that:
    - Reads `$json.intermediateSteps`
    - Converts each step to `{ step, tool, input, response }`
    - Truncates responses dynamically under a ~2.4M character total budget
    - Emits the resulting JSON string into the prompt under “RETRIEVED DATA…”
  - Options:
    - `maxIterations: 10`
    - `systemMessage`: extensive requirements (no follow-ups; exhaustive; citations with URLs; executive summary first; etc.)
    - `enableStreaming: true`
    - `returnIntermediateSteps: true`
  - `needsFallback: true`, `retryOnFail: true`
- **Outputs:** to **Retry if Response Empty1**
- **Failure/edge cases:**
  - Very large intermediateSteps can still exceed provider context length; truncation logic reduces risk but not guaranteed.
  - If intermediateSteps contains non-serializable structures, `JSON.stringify` could fail (rare in n8n, but possible if circular data were introduced).
  - Agent may output non-string or empty output.

#### Node: Retry if Response Empty1
- **Type / role:** `n8n-nodes-base.if` — retries the writing agent if `output` is empty.
- **Condition logic:**
  - `{{$json.output}}` is empty **AND**
  - `{{$runIndex}} < 4`
- **Outputs:**
  - **True** → loop back to **Writing Report**
  - **False** → proceed to **Set Report**
- **Edge cases:**
  - If writing agent outputs whitespace, “empty” check may not catch it (depends on n8n empty operator behavior).

---

### Block 2.5 — Packaging Draft + Verification + Structured Parsing
**Overview:** Stores the drafted report and retrieved docs, then runs a verification LLM chain whose output must match a strict JSON schema. Includes fallback logic if verification output is empty.

**Nodes involved:**
- Set Report
- Verifying Report
- Sonnet
- Auto Fallback4
- Structured Output Parser
- If Empty Output1

#### Node: Set Report
- **Type / role:** `n8n-nodes-base.set` — prepares fields for verification.
- **Key configuration:**
  - `Final Report = {{$json.output}}`
  - `Retrieved Documents = {{$json.intermediateSteps.slice(-500000)}}`
    - Note: `slice(-500000)` here keeps up to 500k *items* (not characters). For typical runs it effectively keeps all steps.
- **Outputs:** to **Verifying Report**
- **Edge cases:**
  - If `intermediateSteps` is missing, `slice` could throw; in practice intermediateSteps exists because writer returns it, but if not, this is a risk.

#### Node: Sonnet
- **Type / role:** `lmChatOpenRouter` — primary model for verification chain.
- **Model:** `anthropic/claude-sonnet-4.5`
- **Options:** temperature 0.2
- **Outputs:** `ai_languageModel` to **Verifying Report** (index 0)

#### Node: Auto Fallback4
- **Type / role:** `lmChatOpenRouter` — fallback model for verification + parser.
- **Model:** `openrouter/auto`
- **Options:** temperature 0.2
- **Outputs:**
  - `ai_languageModel` to **Verifying Report** (index 1)
  - `ai_languageModel` to **Structured Output Parser** (index 0)

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces JSON schema on verification output.
- **Key configuration:**
  - `schemaType: manual`
  - `autoFix: true` (attempts to correct slightly malformed JSON)
  - Schema requires:
    - `contains_hallucinations` (boolean)
    - `hallucinations` (array of objects with `exact_text`, `issue_type`, `severity`, optional `searched_in`)
    - `summary` (string)
- **Outputs:** connected as `ai_outputParser` into **Verifying Report**
- **Edge cases:**
  - If model produces non-repairable output, parser may still fail.
  - If the LLM omits required fields repeatedly, verification step can become unreliable.

#### Node: Verifying Report
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — verification pass that compares report to retrieved documents.
- **Key configuration:**
  - Prompt includes:
    - Truncation logic for Retrieved Documents to ~2.4M chars
    - Truncation logic for Final Report to ~800k chars
  - `hasOutputParser: true` and uses Structured Output Parser
  - `needsFallback: true`, `retryOnFail: true`
- **Outputs:** to **If Empty Output1**
- **Failure/edge cases:**
  - Context too large even after truncation (provider limits differ).
  - Verification is “permissive” by design; may miss subtle citation issues intentionally.

#### Node: If Empty Output1
- **Type / role:** `n8n-nodes-base.if` — handles empty verification output.
- **Condition:** `{{$json.output}}` is empty (object empty check).
- **Outputs:**
  - **True** → back to **Verifying Report**
  - **False** → to **If hallucinations present**
- **Edge cases:**
  - If parser returns something but missing key fields, this may pass “non-empty” but later logic could fail.

---

### Block 2.6 — Hallucination Fix Loop + Empty-output handling
**Overview:** If verification flags hallucinations, rewrite the report with a specialized “surgical correction” agent. Also includes an “empty output” guard.

**Nodes involved:**
- If hallucinations present
- Fixing Hallucinations
- Opus1
- Auto Fallback5
- Think Tool Analysis2
- If Empty Output

#### Node: If hallucinations present
- **Type / role:** `n8n-nodes-base.if` — branching on verification verdict with a retry ceiling.
- **Conditions:**
  - `{{$json.output.contains_hallucinations}}` is true
  - `{{$runIndex}} < 4`
- **Outputs:**
  - **True** → **Fixing Hallucinations**
  - **False** → **Set Report1** (accept current draft even if not flagged, or after retries exhausted)
- **Edge cases:**
  - If contains_hallucinations is missing or not boolean, condition may fail unexpectedly.

#### Node: Opus1
- **Type / role:** `lmChatOpenRouter` — primary model for fixing agent.
- **Model:** `anthropic/claude-opus-4.5`
- **Options:** temperature 0.1

#### Node: Auto Fallback5
- **Type / role:** `lmChatOpenRouter` — fallback for fixing agent.
- **Model:** `openrouter/auto`
- **Options:** temperature 0.1

#### Node: Think Tool Analysis2
- **Type / role:** `toolThink` — analysis tool for fixing agent.

#### Node: Fixing Hallucinations
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — rewrites report to fix flagged items only.
- **Key configuration:**
  - Prompt includes:
    - Original user query
    - Retrieved documents
    - Failed report (truncated at 800k chars)
    - Hallucinations list (truncated at 80k chars)
  - Options:
    - `maxIterations: 10`
    - Strict system message: minimal intervention, preserve structure, fix only flagged claims, citations required
    - `enableStreaming: true`
    - `returnIntermediateSteps: true`
  - `needsFallback: true`, `retryOnFail: true`
- **Outputs:** to **If Empty Output**
- **Failure/edge cases:**
  - If hallucination list is large/complex, agent may still miss some items.
  - If retrieved documents are too long, context truncation may omit relevant supporting evidence.

#### Node: If Empty Output
- **Type / role:** `n8n-nodes-base.if` — guard if fixing agent output is empty.
- **Condition:** `{{$json.output}}` is empty (string empty check).
- **Outputs:**
  - **True** → **Set Report** and **Fixing Hallucinations** (parallel connections exist; effectively can re-enter fix path)
  - **False** → **Set Report** (store corrected report)
- **Edge cases:**
  - The wiring here can be confusing: the “true” path also connects back into Fixing Hallucinations, which can create repeated attempts if the fixing agent keeps returning empty.

---

### Block 2.7 — Final Routing & Response
**Overview:** Copies the final report into a consistent `report` field, then either responds via webhook (streaming JSON) or outputs data for workflow-to-workflow use.

**Nodes involved:**
- Set Report1
- If Source is Webhook
- Respond to Webhook
- Set Output

#### Node: Set Report1
- **Type / role:** `n8n-nodes-base.set` — prepares `report` for downstream routing.
- **Key configuration:** `report = {{ $('Set Report').item.json['Final Report'] }}`
- **Inputs/outputs:** from **If hallucinations present**, to **If Source is Webhook**
- **Edge cases:** If Set Report never ran (upstream failure), this reference fails.

#### Node: If Source is Webhook
- **Type / role:** `n8n-nodes-base.if` — routes delivery mechanism.
- **Condition:** `{{ $('Set Prompt').item.json.source }}` equals `"webhook"`
- **Outputs:**
  - **True** → **Respond to Webhook**
  - **False** → **Set Output**

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — returns JSON response to the HTTP caller.
- **Key configuration:**
  - `enableStreaming: true`
  - Respond with JSON string:
    - `{ type: "research_report", data: { report, metadata: { generated_at }}}`
- **Expression used:**
  - `={{ JSON.stringify({ type: "research_report", data: { report: $json['report'], metadata: { generated_at: new Date().toISOString() } } }) }}`
- **Edge cases:**
  - If webhook trigger expects a different schema, client integration may break.
  - Streaming in combination with large payload can stress reverse proxies.

#### Node: Set Output
- **Type / role:** `n8n-nodes-base.set` — final output for non-webhook invocation.
- **Key configuration:** `report = {{ $('Set Report').item.json['Final Report'] }}`
- **Edge cases:** Same dependency on Set Report.

---

### Block 2.8 — Unused/Standalone Node (Model not wired)
**Overview:** Present in workflow but not connected to any agent/model/tool input, so it has no effect at runtime.

**Node involved:**
- e701b64e-9a56-466e-abc4-28fedb9cd02d (Auto Fallback3) is used;  
- **e701b64e… is used.**  
- **ca7ae9a4… Auto Fallback4 used.**  
- **43b1afd7… Auto Fallback5 used.**  
- **60319d4… Opus used.**  
- **baa7257c… Opus1 used.**
- **e701b64e…** already covered.
- **However:** node `e701b64e-9a56-466e-abc4-28fedb9cd02d` *Auto Fallback3* is connected; no extra unused LLM node is apparent from connections.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger research request (Webhook) | n8n-nodes-base.webhook | External HTTP entry point (streaming) | — | Set Prompt | # Research topics using AI agents with web search, verification, and report generation (setup + how it works) |
| Trigger research from another workflow | n8n-nodes-base.executeWorkflowTrigger | Internal entry point from other workflows | — | Set Prompt | # Research topics using AI agents with web search, verification, and report generation (setup + how it works) |
| Set Prompt | n8n-nodes-base.set | Normalize prompt + detect source | Trigger research request (Webhook); Trigger research from another workflow | Searching For Information | # Research topics using AI agents with web search, verification, and report generation (setup + how it works) |
| Searching For Information | @n8n/n8n-nodes-langchain.agent | Research agent calling mandatory tools | Set Prompt; Retry if Tools Not Used (loop) | Retry if Tools Not Used | ## 🧰 The Agent’s Toolbox (Superpowers) |
| Qwen3 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Primary research LLM | — | Searching For Information (ai_languageModel) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Alternate research LLM | — | Searching For Information (ai_languageModel) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| Think1 | @n8n/n8n-nodes-langchain.toolThink | Reasoning tool for research agent | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| Serper API 2 | n8n-nodes-base.httpRequestTool | Live web search tool | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| Semantic Scholar 2 | n8n-nodes-base.httpRequestTool | Academic paper search tool | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| Read URL content in Jina AI | n8n-nodes-base.jinaAiTool | Full text extraction from URLs | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| LinkedinScraper 2 | n8n-nodes-base.httpRequestTool | LinkedIn profile scraping tool | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| InstagramScraper 2 | n8n-nodes-base.httpRequestTool | Instagram profile scraping tool | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| XPostScraper 2 | n8n-nodes-base.httpRequestTool | X/Twitter post scraping tool | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| XProfileScraper 2 | n8n-nodes-base.httpRequestTool | X/Twitter profile scraping tool | — | Searching For Information (ai_tool) | ## 🧰 The Agent’s Toolbox (Superpowers) |
| Retry if Tools Not Used | n8n-nodes-base.if | Retry loop if agent didn’t use tools | Searching For Information | Searching For Information (true); Writing Report (false) |  |
| Opus | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Primary report-writer LLM | — | Writing Report (ai_languageModel) | ## ✍️ Writing Report Agent |
| Auto Fallback3 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback writer LLM | — | Writing Report (ai_languageModel) | ## ✍️ Writing Report Agent |
| Think Tool Analysis | @n8n/n8n-nodes-langchain.toolThink | Analysis tool for report writer | — | Writing Report (ai_tool) | ## ✍️ Writing Report Agent |
| Writing Report | @n8n/n8n-nodes-langchain.agent | Draft report generation | Retry if Tools Not Used; Retry if Response Empty1 (loop) | Retry if Response Empty1 | ## ✍️ Writing Report Agent |
| Retry if Response Empty1 | n8n-nodes-base.if | Retry loop if writer output empty | Writing Report | Writing Report (true); Set Report (false) |  |
| Set Report | n8n-nodes-base.set | Store Final Report + Retrieved Documents | Retry if Response Empty1; If Empty Output (both branches) | Verifying Report |  |
| Sonnet | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Primary verifier LLM | — | Verifying Report (ai_languageModel) | ## 🔍 Verifying Report Agent |
| Auto Fallback4 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback verifier LLM | — | Verifying Report (ai_languageModel); Structured Output Parser (ai_languageModel) | ## 🔍 Verifying Report Agent |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce verification JSON schema | — | Verifying Report (ai_outputParser) | ## 🔍 Verifying Report Agent |
| Verifying Report | @n8n/n8n-nodes-langchain.chainLlm | Fact-check report vs retrieved docs | Set Report; If Empty Output1 (loop) | If Empty Output1 | ## 🔍 Verifying Report Agent |
| If Empty Output1 | n8n-nodes-base.if | Retry verification if output empty | Verifying Report | Verifying Report (true); If hallucinations present (false) |  |
| If hallucinations present | n8n-nodes-base.if | Branch to fix hallucinations or accept report | If Empty Output1 | Fixing Hallucinations (true); Set Report1 (false) |  |
| Opus1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Primary hallucination-fixer LLM | — | Fixing Hallucinations (ai_languageModel) | ## 🧯 Fixing Hallucinations Agent |
| Auto Fallback5 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Fallback fixer LLM | — | Fixing Hallucinations (ai_languageModel) | ## 🧯 Fixing Hallucinations Agent |
| Think Tool Analysis2 | @n8n/n8n-nodes-langchain.toolThink | Analysis tool for fixing agent | — | Fixing Hallucinations (ai_tool) | ## 🧯 Fixing Hallucinations Agent |
| Fixing Hallucinations | @n8n/n8n-nodes-langchain.agent | Rewrite report to correct flagged hallucinations | If hallucinations present; If Empty Output (loop) | If Empty Output | ## 🧯 Fixing Hallucinations Agent |
| If Empty Output | n8n-nodes-base.if | Guard if fixer output empty | Fixing Hallucinations | Set Report + Fixing Hallucinations (true); Set Report (false) |  |
| Set Report1 | n8n-nodes-base.set | Prepare `report` field for final routing | If hallucinations present | If Source is Webhook |  |
| If Source is Webhook | n8n-nodes-base.if | Route to webhook response vs workflow output | Set Report1 | Respond to Webhook (true); Set Output (false) |  |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return streaming JSON response | If Source is Webhook | — |  |
| Set Output | n8n-nodes-base.set | Output `report` for workflow callers | If Source is Webhook | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | (content is the note itself) |
| Sticky Note6 | n8n-nodes-base.stickyNote | Credential requirements note | — | — | (content is the note itself) |
| Sticky Note12 | n8n-nodes-base.stickyNote | Toolbox explanation note | — | — | (content is the note itself) |
| Sticky Note13 | n8n-nodes-base.stickyNote | Writer agent explanation note | — | — | (content is the note itself) |
| Sticky Note14 | n8n-nodes-base.stickyNote | Verifier agent explanation note | — | — | (content is the note itself) |
| Sticky Note15 | n8n-nodes-base.stickyNote | Fixer agent explanation note | — | — | (content is the note itself) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *Research topics using AI agents with web search, verification, and report generation*

2) **Add Trigger 1: Webhook**
   - Node: **Webhook**
   - Method: **POST**
   - Path: `general-research`
   - Response mode: **Streaming**

3) **Add Trigger 2: Execute Workflow Trigger**
   - Node: **Execute Workflow Trigger**
   - Add Workflow Input: `prompt` (string)

4) **Add “Set Prompt” node**
   - Node: **Set**
   - Add fields:
     - `prompt` (string): `{{$json.prompt || ($json.body && $json.body.prompt) || ''}}`
     - `source` (string): `{{$json.webhookUrl ? 'webhook' : 'workflow'}}`
   - Connect **both triggers → Set Prompt**

5) **Add the Research Agent**
   - Node: **AI Agent** (LangChain Agent)
   - Text: `{{$json.prompt}}`
   - Enable:
     - **Return Intermediate Steps** = true
     - **Streaming** = true
   - System message: paste the “Intelligence & Research Agent” system instructions (mandatory tool execution, citations, etc.)
   - Connect: **Set Prompt → Research Agent**

6) **Add tools for the Research Agent (connect each as “AI Tool” into the agent)**
   - **HTTP Request Tool: Serper**
     - POST `https://google.serper.dev/search`
     - Body param: `q` (from AI): use `$fromAI` field for query text
     - Credentials: **HTTP Header Auth** with Serper API key in headers (per your credential setup)
     - Response options: “Full response” and “Never error” (optional but matches template)
   - **HTTP Request Tool: Semantic Scholar**
     - GET `https://api.semanticscholar.org/graph/v1/paper/search`
     - Query params: `query` (from AI), `limit` (from AI; must be <=100)
   - **Jina AI Tool**
     - URL from AI input
     - Credentials: Jina AI API key/account
   - **HTTP Request Tool: ScrapingDog LinkedIn**
     - URL from AI input (complete ScrapingDog endpoint)
     - Credentials: **HTTP Query Auth** with ScrapingDog key
   - **HTTP Request Tool: ScrapingDog Instagram**
     - URL from AI input
     - Credentials: ScrapingDog key
   - **HTTP Request Tool: ScrapingDog X Post**
     - URL from AI input
     - Credentials: ScrapingDog key
   - **HTTP Request Tool: ScrapingDog X Profile**
     - URL from AI input
     - Credentials: ScrapingDog key
   - **Think Tool**
     - Node: “Think Tool” (LangChain toolThink)
     - Connect as AI tool

7) **Add LLMs for the Research Agent (connect as “AI Language Model”)**
   - **OpenRouter Chat Model** with model `qwen/qwen3-235b-a22b`
   - Optional additional model: `openrouter/auto` as fallback
   - Credentials: OpenRouter API credential

8) **Add “Retry if Tools Not Used” IF node**
   - Condition group (AND):
     - `{{$json.intermediateSteps[0]}}` is **empty**
     - `{{$runIndex}} < 4`
   - True output → back to **Research Agent**
   - False output → proceed to **Writing Report**

9) **Add the Writing Report Agent**
   - Node: **AI Agent**
   - Text: build a prompt that:
     - embeds `$json.intermediateSteps`
     - truncates content to stay within context limits
     - includes the original `prompt` as “REPORT GOAL”
   - Enable streaming and returnIntermediateSteps
   - Add writer system message (the long “COMPREHENSIVE REPORT WRITING AGENT” instructions)
   - Connect: **Retry if Tools Not Used (false) → Writing Report**

10) **Add Writer LLMs + tools**
   - LLM: OpenRouter model `anthropic/claude-opus-4.5` (temperature ~0.6)
   - Fallback LLM: `openrouter/auto`
   - Tool: Think Tool (analysis synthesis) connected as AI tool to Writing Report

11) **Add “Retry if Response Empty” IF node**
   - Condition (AND):
     - `{{$json.output}}` is **empty**
     - `{{$runIndex}} < 4`
   - True → loop back to **Writing Report**
   - False → **Set Report**

12) **Add “Set Report” node**
   - Node: **Set**
   - Fields:
     - `Final Report` = `{{$json.output}}`
     - `Retrieved Documents` = `{{$json.intermediateSteps.slice(-500000)}}`
   - Connect: **Retry if Response Empty (false) → Set Report**

13) **Add Verification chain**
   - Node: **LLM Chain** (chainLlm)
   - Prompt: include:
     - Retrieved Documents (with truncation)
     - Final Report (with truncation)
     - permissive verification instructions
   - Add **Structured Output Parser**
     - Manual JSON schema (contains_hallucinations, hallucinations[], summary)
     - Auto-fix enabled
   - Connect parser to chain as output parser
   - Add LLM: `anthropic/claude-sonnet-4.5` (temp ~0.2)
   - Fallback LLM: `openrouter/auto`
   - Connect: **Set Report → Verifying Report**

14) **Add “If Empty Output1” IF node**
   - Condition: `{{$json.output}}` is empty (object empty)
   - True → loop back to **Verifying Report**
   - False → **If hallucinations present**

15) **Add “If hallucinations present” IF node**
   - Conditions (AND):
     - `{{$json.output.contains_hallucinations}}` is **true**
     - `{{$runIndex}} < 4`
   - True → **Fixing Hallucinations**
   - False → **Set Report1** (accept current report)

16) **Add Fixing Hallucinations Agent**
   - Node: **AI Agent**
   - Prompt includes:
     - original query (`$('Set Prompt').item.json.prompt`)
     - retrieved docs (`$('Set Report').item.json['Retrieved Documents']`)
     - failed report (`$('Set Report').item.json['Final Report']`)
     - hallucinations list (`$json.output.hallucinations`)
   - LLM: `anthropic/claude-opus-4.5` (temp ~0.1)
   - Fallback: `openrouter/auto`
   - Tool: Think Tool for analysis
   - Connect: **If hallucinations present (true) → Fixing Hallucinations**

17) **Add “If Empty Output” guard**
   - Condition: `{{$json.output}}` is empty (string empty)
   - False → **Set Report** (store corrected output)
   - (Optional, to match template) True → retry path(s) if desired, but ensure you don’t create an infinite loop.

18) **Add “Set Report1” node**
   - Node: **Set**
   - Field: `report = {{ $('Set Report').item.json['Final Report'] }}`
   - Connect: **If hallucinations present (false) → Set Report1**

19) **Add “If Source is Webhook” router**
   - IF condition: `{{ $('Set Prompt').item.json.source }}` equals `webhook`
   - True → **Respond to Webhook**
   - False → **Set Output**

20) **Add “Respond to Webhook” node**
   - Node: **Respond to Webhook**
   - Respond with: **JSON**
   - Enable streaming
   - Body expression:
     - `JSON.stringify({ type: "research_report", data: { report: $json.report, metadata: { generated_at: new Date().toISOString() } } })`

21) **Add “Set Output” node**
   - Node: **Set**
   - Field: `report = {{ $('Set Report').item.json['Final Report'] }}`
   - This is the terminal output for workflow-to-workflow invocation.

### Credentials to configure
- **OpenRouter API** credential: used by all `lmChatOpenRouter` nodes (Qwen3, Opus, Sonnet, fallbacks).
- **Jina AI API** credential: used by Jina AI tool node.
- **Serper API**: HTTP Header Auth (typically header like `X-API-KEY: <key>`).
- **ScrapingDog Key**: HTTP Query Auth for LinkedIn/Instagram/X scraper endpoints.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “No credentials are included in this template. You must add your own credentials after importing the workflow.” | From the workflow’s credential sticky note |
| The workflow is designed as an automated, one-shot pipeline: no follow-up questions; exhaustive report; citations required; verification + optional correction loop. | Embedded in the agents’ system messages |
| Disclaimer (FR): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Provided in the request context (compliance/attribution) |