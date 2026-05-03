Aggregate research answers from GPT 5.2, Claude Opus 4.6 and Gemini 3 Pro

https://n8nworkflows.xyz/workflows/aggregate-research-answers-from-gpt-5-2--claude-opus-4-6-and-gemini-3-pro-13261


# Aggregate research answers from GPT 5.2, Claude Opus 4.6 and Gemini 3 Pro

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Multi-AI Council Research  
**Purpose:** Receive a chat message, decide whether it is a *research/search* request, then (if yes) run the optimized query in parallel through **GPT‑5.2**, **Claude Opus 4.6**, and **Gemini 3 Pro**, finally **aggregating** their three responses into a single synthesized answer. If not a research query, it returns a fallback message.

**Target use cases**
- Research questions where multi-model triangulation reduces single-model bias
- Comparative synthesis (consensus + disagreement handling)
- Controlled routing to avoid unnecessary model usage on casual messages

### 1.1 Input Reception
- Entry point receives a chat message via n8n’s LangChain chat trigger.

### 1.2 Query Classification & Optimization (Structured)
- A Gemini model classifies the user message (“research”: yes/no) and produces an optimized query in structured JSON via an output parser.

### 1.3 Conditional Routing
- IF node routes to:
  - Multi-model execution if `output.search === "yes"`
  - Fallback chat response otherwise

### 1.4 Parallel Multi-Model Research Execution
- Three LLM chains run in parallel using the optimized query:
  - OpenAI GPT‑5.2 Pro (+ built-in web search)
  - Anthropic Claude Opus 4.6 (thinking enabled)
  - Google Gemini 3 Pro Preview

### 1.5 Response Normalization & Aggregation
- Each model output is mapped into a dedicated field (`chatgpt`, `claude`, `gemini`)
- Aggregator LLM synthesizes one final response from the three inputs

---

## 2. Block-by-Block Analysis

### Block A — Input Reception
**Overview:** Accepts chat messages and starts the workflow execution.

**Nodes involved**
- **When chat message received**

**Node details**
- **When chat message received**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — chat-based trigger/webhook entry point.
  - **Key config:** `responseMode: "responseNodes"` meaning the workflow will return a response via dedicated response nodes (here, the Chat node is configured to reply in the non-research path; the aggregator result is typically returned as the chat response in the research path depending on your n8n chat setup).
  - **Connections:**
    - **Main output →** Search Query Optimizer
  - **Potential failures / edge cases:**
    - Trigger URL misconfiguration, chat integration mismatch, or missing chat UI binding.
    - If downstream nodes do not send a response compatible with `responseNodes`, the chat client may not receive an answer.

**Sticky note(s) applying**
- “Multi-AI Council Research: GPT 5.2, Claude Opus 4.6 & Gemini 3 Pro Aggregation” (general workflow note)
- “STEP 1 - Input Processing …”

---

### Block B — Query Classification & Optimization (Gemini + Structured Parser)
**Overview:** Uses Gemini to determine if input is a research query and outputs structured JSON containing the decision and the optimized query string.

**Nodes involved**
- **Search Query Optimizer**
- **Google Gemini Chat Model** (for the optimizer)
- **Structured Output Parser**

**Node details**
- **Google Gemini Chat Model**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM backend for the optimizer chain.
  - **Key config:** Default options.
  - **Credentials:** `Google Gemini(PaLM) (Eure)` (googlePalmApi).
  - **Connections:**
    - **ai_languageModel →** Search Query Optimizer
  - **Failures / edge cases:**
    - Invalid/expired Google PaLM/Gemini key, quota limits, model availability changes.

- **Structured Output Parser**
  - **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces that the optimizer returns JSON matching a schema.
  - **Schema (manual):**
    - object with properties:
      - `search` (string)
      - `query` (string)
    - Note: the prompt also asks for `original_intent`, but the parser schema does **not** include it; depending on parser strictness/version, this can cause:
      - extra field being dropped, or
      - parsing failure if strict validation is enforced.
  - **Connections:**
    - **ai_outputParser →** Search Query Optimizer
  - **Failures / edge cases:**
    - Model returns non-JSON or invalid JSON → parsing fails.
    - Mismatch between prompt-required fields and schema fields.

- **Search Query Optimizer**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — LLM chain that:
    1) classifies input as research or not  
    2) returns optimized query as structured JSON
  - **Key config:**
    - Uses an explicit system-style instruction (classification criteria + required JSON format).
    - `hasOutputParser: true` → output is expected under `$json.output` (as used later).
  - **Key variables/expressions used downstream:**
    - `{{$json.output.search}}`
    - `{{$json.output.query}}`
  - **Connections:**
    - **Main input ←** When chat message received
    - **Main output →** IF node “Search?”
    - **AI language model input ←** Google Gemini Chat Model
    - **AI output parser input ←** Structured Output Parser
  - **Failures / edge cases:**
    - If `output.search` is not exactly `"yes"` (e.g., “Yes”, “true”, “y”), the IF condition fails and routes to fallback.
    - If `output.query` is missing/empty, later LLM chains will run with blank prompts.

**Sticky note(s) applying**
- “STEP 1 - Input Processing …”
- “Multi-AI Council Research …” (general)

---

### Block C — Conditional Routing (Research vs Fallback)
**Overview:** Routes execution based on the optimizer output field `output.search`.

**Nodes involved**
- **Search?**
- **Chat** (fallback response)

**Node details**
- **Search?**
  - **Type / role:** `n8n-nodes-base.if` — branching.
  - **Condition:** `{{$json.output.search}} equals "yes"` (strict).
  - **Connections:**
    - **True →** ChatGPT 5.2, Claude Opus 4.6, Gemini 3 Pro (in parallel)
    - **False →** Chat (fallback)
  - **Failures / edge cases:**
    - Strict string match can misroute if the optimizer returns variants (“Yes”, “yes.”).
    - If optimizer output is missing (`$json.output` undefined), the IF evaluation may error (type validation is strict).

- **Chat** (fallback)
  - **Type / role:** `@n8n/n8n-nodes-langchain.chat` — returns a chat message to the user.
  - **Message:** “This is not a research text. Please enter one.”
  - **Options:** `memoryConnection: false` (no conversational memory).
  - **Connections:**
    - Receives from **Search? → false branch**
    - No explicit outgoing connections (acts as response node).
  - **Failures / edge cases:**
    - If chat trigger expects a response node but routing doesn’t end in one, the user may see no response. Here fallback is covered.

**Sticky note(s) applying**
- “STEP 3 - Fallback Handling …”
- “Multi-AI Council Research …” (general)

---

### Block D — Parallel Multi-Model Query Execution
**Overview:** Runs the optimized query through three different model providers to obtain three independent answers.

**Nodes involved**
- **ChatGPT 5.2** + **OpenAI Chat Model**
- **Claude Opus 4.6** + **Anthropic Chat Model**
- **Gemini 3 Pro** + **Google Gemini Chat Model1**

**Node details (OpenAI path)**
- **OpenAI Chat Model**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI backend for the chain.
  - **Model:** `gpt-5.2-pro`
  - **Built-in tools:** Web search enabled (`webSearch.searchContextSize: "medium"`)
  - **Credentials:** `OpenAi account (Eure)`
  - **Connections:** **ai_languageModel →** ChatGPT 5.2
  - **Failures / edge cases:**
    - Tool availability depends on account/model support; web search can fail due to policy, quota, or network.
    - Higher latency due to web search.

- **ChatGPT 5.2**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — executes prompt text.
  - **Text:** `={{ $json.output.query }}`
  - **Connections:**
    - **Main input ←** Search? (true)
    - **AI language model input ←** OpenAI Chat Model
    - **Main output →** ChatGPT Result
  - **Failures / edge cases:**
    - If `$json.output.query` is undefined, the model may answer poorly or error depending on node validation.

**Node details (Anthropic path)**
- **Anthropic Chat Model**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`
  - **Model:** `claude-opus-4-6`
  - **Options:** `thinking: true`, `thinkingBudget: 1024`
  - **Credentials:** `Anthropic account`
  - **Connections:** **ai_languageModel →** Claude Opus 4.6
  - **Failures / edge cases:**
    - “Thinking” increases token usage/cost; can hit token limits or timeouts.
    - Credential/quota issues.

- **Claude Opus 4.6**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm`
  - **Text:** `={{ $json.output.query }}`
  - **Connections:**
    - **Main input ←** Search? (true)
    - **AI language model input ←** Anthropic Chat Model
    - **Main output →** Claude Result

**Node details (Gemini path)**
- **Google Gemini Chat Model1**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`
  - **Model name:** `models/gemini-3-pro-preview`
  - **Options:** `maxOutputTokens: 2048`
  - **Credentials:** `Google Gemini(PaLM) (Eure)`
  - **Connections:** **ai_languageModel →** Gemini 3 Pro
  - **Failures / edge cases:**
    - Preview model may be renamed/retired; causes runtime model-not-found errors.
    - Token cap may truncate long answers.

- **Gemini 3 Pro**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm`
  - **Text:** `={{ $json.output.query }}`
  - **Connections:**
    - **Main input ←** Search? (true)
    - **AI language model input ←** Google Gemini Chat Model1
    - **Main output →** Gemini Result

**Sticky note(s) applying**
- “STEP2 - Multi-Model Query Execution …”
- “Multi-AI Council Research …” (general)

---

### Block E — Response Normalization & Aggregation
**Overview:** Standardizes each model response into named fields, then passes the three responses to a dedicated aggregator model that synthesizes a single answer.

**Nodes involved**
- **ChatGPT Result**
- **Claude Result**
- **Gemini Result**
- **Multi-Response Aggregator**
- **Google Gemini Chat Model2** (aggregator model)

**Node details**
- **ChatGPT Result**
  - **Type / role:** `n8n-nodes-base.set` — maps output to a consistent property.
  - **Assignment:** `chatgpt = {{$json.text}}`
  - **Connections:** **→** Multi-Response Aggregator
  - **Edge cases:**
    - If upstream chain outputs a different field than `text`, mapping becomes empty.

- **Claude Result**
  - **Type / role:** `n8n-nodes-base.set`
  - **Assignment intended:** `claude = {{$json.text}}`
  - **Important issue:** In the provided configuration the value is `={{ $json.text }` (missing closing `}` and `}}`). This will cause an **expression parsing error** at runtime.
  - **Connections:** **→** Multi-Response Aggregator
  - **Edge cases / failures:**
    - This malformed expression likely prevents the workflow from completing the research path.

- **Gemini Result**
  - **Type / role:** `n8n-nodes-base.set`
  - **Assignment:** `gemini = {{$json.text}}`
  - **Connections:** **→** Multi-Response Aggregator

- **Google Gemini Chat Model2**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — LLM backend for aggregation.
  - **Credentials:** `Google Gemini(PaLM) Api account` (note: different credential than earlier Gemini nodes).
  - **Connections:** **ai_languageModel →** Multi-Response Aggregator
  - **Failures / edge cases:**
    - Credential mismatch across environments; ensure this second Gemini credential exists in target n8n instance.

- **Multi-Response Aggregator**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — synthesis step.
  - **Input text composed as:**
    - `Response 1: {{ $json.chatgpt }}`
    - `Response 2: {{ $json.claude }}`
    - `Response 3: {{ $json.gemini }}`
  - **System message:** Detailed aggregation guidelines (avoid concatenation, handle contradictions, no new info).
  - **Connections:**
    - **Main inputs ←** ChatGPT Result, Claude Result, Gemini Result
    - **AI language model input ←** Google Gemini Chat Model2
  - **Edge cases / integration concerns:**
    - Because three separate Set nodes feed into the aggregator independently, the aggregator will execute **once per incoming item** unless items are merged. As-is, it may run three times, each time with only one of `chatgpt/claude/gemini` present (depending on n8n item flow).
    - To truly aggregate all three responses in one pass, you typically need a **Merge** strategy (e.g., Merge node by position, or a Code node to combine into one item) before the aggregator.
    - If any of the three upstream paths fails, the aggregator may get partial data; prompts should handle missing responses.

**Sticky note(s) applying**
- “STEP 4 - Response Aggregation …”
- “Multi-AI Council Research …” (general)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | LangChain Chat Trigger | Entry point (chat webhook trigger) | — | Search Query Optimizer | Multi-AI Council Research: GPT 5.2, Claude Opus 4.6 & Gemini 3 Pro Aggregation |
| When chat message received | LangChain Chat Trigger | Entry point (chat webhook trigger) | — | Search Query Optimizer | STEP 1 - Input Processing: When a chat message is received… |
| Search Query Optimizer | LangChain LLM Chain | Classify + optimize query; emits structured output | When chat message received | Search? | STEP 1 - Input Processing: When a chat message is received… |
| Search Query Optimizer | LangChain LLM Chain | Classify + optimize query; emits structured output | When chat message received | Search? | Multi-AI Council Research: GPT 5.2, Claude Opus 4.6 & Gemini 3 Pro Aggregation |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | LLM backend for query optimizer | — | Search Query Optimizer (ai_languageModel) | STEP 1 - Input Processing: When a chat message is received… |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for optimizer output | — | Search Query Optimizer (ai_outputParser) | STEP 1 - Input Processing: When a chat message is received… |
| Search? | IF | Route research vs fallback | Search Query Optimizer | (true) ChatGPT 5.2, Claude Opus 4.6, Gemini 3 Pro; (false) Chat | Multi-AI Council Research: GPT 5.2, Claude Opus 4.6 & Gemini 3 Pro Aggregation |
| Chat | LangChain Chat | Fallback response to user | Search? (false) | — | STEP 3 - Fallback Handling: If the input is not a research query… |
| ChatGPT 5.2 | LangChain LLM Chain | Run optimized query on GPT‑5.2 | Search? (true) | ChatGPT Result | STEP2 - Multi-Model Query Execution: …sends the optimized query to three different AI models… |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | GPT‑5.2 model config + web search tool | — | ChatGPT 5.2 (ai_languageModel) | STEP2 - Multi-Model Query Execution: …sends the optimized query to three different AI models… |
| Claude Opus 4.6 | LangChain LLM Chain | Run optimized query on Claude | Search? (true) | Claude Result | STEP2 - Multi-Model Query Execution: …sends the optimized query to three different AI models… |
| Anthropic Chat Model | Anthropic Chat Model (LangChain) | Claude model config (thinking enabled) | — | Claude Opus 4.6 (ai_languageModel) | STEP2 - Multi-Model Query Execution: …sends the optimized query to three different AI models… |
| Gemini 3 Pro | LangChain LLM Chain | Run optimized query on Gemini 3 Pro | Search? (true) | Gemini Result | STEP2 - Multi-Model Query Execution: …sends the optimized query to three different AI models… |
| Google Gemini Chat Model1 | Gemini Chat Model (LangChain) | Gemini 3 Pro Preview config | — | Gemini 3 Pro (ai_languageModel) | STEP2 - Multi-Model Query Execution: …sends the optimized query to three different AI models… |
| ChatGPT Result | Set | Normalize GPT response to `chatgpt` field | ChatGPT 5.2 | Multi-Response Aggregator | STEP 4 - Response Aggregation: Each model's response is collected… |
| Claude Result | Set | Normalize Claude response to `claude` field | Claude Opus 4.6 | Multi-Response Aggregator | STEP 4 - Response Aggregation: Each model's response is collected… |
| Gemini Result | Set | Normalize Gemini response to `gemini` field | Gemini 3 Pro | Multi-Response Aggregator | STEP 4 - Response Aggregation: Each model's response is collected… |
| Multi-Response Aggregator | LangChain LLM Chain | Synthesize 3 responses into 1 | ChatGPT Result, Claude Result, Gemini Result | — | STEP 4 - Response Aggregation: Each model's response is collected… |
| Google Gemini Chat Model2 | Gemini Chat Model (LangChain) | LLM backend for aggregation | — | Multi-Response Aggregator (ai_languageModel) | STEP 4 - Response Aggregation: Each model's response is collected… |
| Sticky Note | Sticky Note | Documentation | — | — | ## Multi-AI Council Research: GPT 5.2, Claude Opus 4.6 & Gemini 3 Pro Aggregation … |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## STEP 1 - Input Processing … |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## STEP2 - Multi-Model Query Execution … |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## STEP 3 - Fallback Handling … |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## STEP 4 - Response Aggregation … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Multi-AI Council Research”**
   - (Optional) Add sticky notes with the provided contents for maintainability.

2. **Add the trigger**
   - Node: **LangChain → Chat Trigger**
   - Set **Response Mode** to **“Response Nodes”**
   - Connect: **Chat Trigger → Search Query Optimizer**

3. **Add query optimizer LLM chain**
   - Node: **LangChain → LLM Chain**
   - Name: **Search Query Optimizer**
   - In Messages (system instruction), paste the optimizer prompt (classification + JSON format).
   - Enable **Output Parser** on this chain.

4. **Attach Gemini model to optimizer**
   - Node: **LangChain → Google Gemini Chat Model**
   - Add/choose **Google Palm/Gemini credentials**
   - Connect: **Gemini Chat Model (ai_languageModel) → Search Query Optimizer**

5. **Attach structured output parser**
   - Node: **LangChain → Structured Output Parser**
   - Schema type: **Manual**
   - Use schema with at least:
     - `search` (string)
     - `query` (string)
   - Connect: **Structured Output Parser (ai_outputParser) → Search Query Optimizer**
   - Recommended: either (a) add `original_intent` to the schema, or (b) remove it from the optimizer prompt to prevent parse issues.

6. **Add conditional routing**
   - Node: **IF**
   - Condition: **String equals**
     - Left: `{{$json.output.search}}`
     - Right: `yes`
   - Connect: **Search Query Optimizer → IF**

7. **Add fallback response**
   - Node: **LangChain → Chat**
   - Message: **“This is not a research text. Please enter one.”**
   - Memory connection: **off**
   - Connect: **IF (false) → Chat**

8. **Create the three research LLM chains (parallel)**
   - Add three nodes: **LangChain → LLM Chain**
     - **ChatGPT 5.2**
     - **Claude Opus 4.6**
     - **Gemini 3 Pro**
   - For each, set **Text** to: `{{$json.output.query}}`
   - Connect: **IF (true) →** each of the three chains (parallel).

9. **Configure OpenAI GPT chain**
   - Node: **LangChain → OpenAI Chat Model**
   - Model: **gpt-5.2-pro**
   - Enable built-in tool: **Web Search** (context size medium)
   - Set OpenAI credentials (API key/OAuth depending on your setup)
   - Connect: **OpenAI Chat Model (ai_languageModel) → ChatGPT 5.2**

10. **Configure Anthropic chain**
   - Node: **LangChain → Anthropic Chat Model**
   - Model: **claude-opus-4-6**
   - Options: **thinking enabled**, budget **1024**
   - Set Anthropic API credentials
   - Connect: **Anthropic Chat Model (ai_languageModel) → Claude Opus 4.6**

11. **Configure Gemini 3 Pro chain**
   - Node: **LangChain → Google Gemini Chat Model**
   - Model name: **models/gemini-3-pro-preview**
   - maxOutputTokens: **2048**
   - Set Google Gemini credentials
   - Connect: **Gemini Chat Model (ai_languageModel) → Gemini 3 Pro**

12. **Normalize each model output**
   - Add three **Set** nodes:
     - **ChatGPT Result**: set `chatgpt = {{$json.text}}`
     - **Claude Result**: set `claude = {{$json.text}}` (**ensure the expression is valid**)
     - **Gemini Result**: set `gemini = {{$json.text}}`
   - Connect:
     - ChatGPT 5.2 → ChatGPT Result
     - Claude Opus 4.6 → Claude Result
     - Gemini 3 Pro → Gemini Result

13. **(Important) Combine the three results into one item**
   - As designed, you should add a **Merge** step (or Code node) so the aggregator receives a single item containing **all three fields**.
   - Typical approach:
     - Merge ChatGPT Result + Claude Result (by position) → intermediate
     - Merge intermediate + Gemini Result (by position) → combined item
   - Without this, the aggregator may run multiple times with partial inputs.

14. **Add the aggregation chain**
   - Node: **LangChain → LLM Chain**
   - Name: **Multi-Response Aggregator**
   - Text template:
     - `Response 1: {{ $json.chatgpt }}`
     - `Response 2: {{ $json.claude }}`
     - `Response 3: {{ $json.gemini }}`
   - Add the aggregation system message (guidelines prompt).

15. **Attach aggregation model**
   - Node: **LangChain → Google Gemini Chat Model**
   - Credentials: your “Api account” credential (or reuse the same one consistently)
   - Connect: **Gemini model (ai_languageModel) → Multi-Response Aggregator**

16. **Connect merged output to aggregator**
   - Connect the **merged combined item** node → **Multi-Response Aggregator**
   - Ensure the final output is returned to the chat client (depending on your n8n chat configuration, the chain output can be used as the response node; otherwise add a Chat response node to send `{{$json.text}}` from the aggregator).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Multi-AI Council Research: GPT 5.2, Claude Opus 4.6 & Gemini 3 Pro Aggregation… Setup steps…” | Sticky note describing the overall architecture and setup checklist |
| STEP notes (1–4): Input Processing, Multi-Model Execution, Fallback Handling, Response Aggregation | Sticky notes embedded in the canvas for maintainability |

--- 

### Key Implementation Warnings (from the provided configuration)
- **Claude Result Set node has a broken expression** (`={{ $json.text }`) and must be fixed to `={{ $json.text }}`.
- The **Structured Output Parser schema** does not include `original_intent` while the optimizer prompt demands it; align them to avoid parsing failures.
- Add a **Merge/Combine** step before the aggregator; otherwise aggregation likely runs with partial data (and/or runs multiple times).