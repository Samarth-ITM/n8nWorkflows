Answer research questions using OpenAI GPT-4.1 and arXiv papers

https://n8nworkflows.xyz/workflows/answer-research-questions-using-openai-gpt-4-1-and-arxiv-papers-12938


# Answer research questions using OpenAI GPT-4.1 and arXiv papers

## 1. Workflow Overview

**Purpose:** This workflow acts as an autonomous research assistant for answering user questions either (a) from **general knowledge** or (b) using **freshly retrieved arXiv papers** (titles/abstracts/links) and then producing a grounded, cited response.

**Target use cases:**
- Literature scouting for a topic (recent papers, trends)
- Quick paper-by-paper summaries based on abstracts
- Producing answers that are explicitly grounded in retrieved arXiv metadata (with citations)

### 1.1 Input Reception
A user question is received via an n8n Chat Trigger.

### 1.2 Intent Planning (General vs Research)
A “Planning Agent” (LLM) decides whether arXiv needs to be queried. Its output is parsed into JSON and routed through an IF node.

### 1.3 Research Path: arXiv Retrieval + Parsing
If research is needed, the workflow calls the arXiv API, normalizes the response, parses XML into paper objects, and forwards them to a grounded summarization/answer agent.

### 1.4 General Knowledge Path
If research is not needed, the workflow answers directly using a general reasoning agent that must explicitly state it is using general knowledge.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Captures the user’s message and starts the workflow.  
**Nodes involved:** `User asks a question`

#### Node: User asks a question
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point for chat-based interactions.
- **Key configuration:** Uses the Chat Trigger default options; identified by a webhookId (internal to n8n).
- **Outputs:** Sends the incoming chat message payload to **Planning Agent**.
- **Edge cases / failures:**
  - Chat trigger not reachable (n8n URL/webhook configuration).
  - Missing/empty user message leads to downstream expression issues (agents expect text).

---

### Block 2 — Intent Planning (General vs Research)
**Overview:** Uses an LLM agent to decide whether to search arXiv. Then parses the agent output into JSON and routes execution.  
**Nodes involved:** `Planning Agent`, `OpenAI Chat Model`, `Json Parser 1`, `Check research q`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — shared language model used by multiple agent nodes.
- **Key configuration choices:**
  - **Model:** `gpt-4.1-mini`
  - **Temperature:** 0.7, **TopP:** 0.9
  - **Max tokens:** 5000
  - **Max retries:** 10 (helps transient API failures)
- **Connections:** Feeds the `ai_languageModel` input of:
  - `Planning Agent`
  - `General Reasoning Agent`
  - `Arxiv Grounded Agent`
- **Version requirements:** Node typeVersion `1.2` (model list UI and options may differ across n8n versions).
- **Edge cases / failures:**
  - OpenAI credential missing/invalid, quota exceeded, model not available.
  - Large prompts (many abstracts) can still exceed context limits despite maxTokens setting.

#### Node: Planning Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — classification/planning agent.
- **Configuration (interpreted):**
  - **System message** strictly instructs the agent to output **only valid JSON** with keys:
    - `need_search` (boolean)
    - `topic` (string)
    - `search_query` (array of strings)
    - `max_papers` (number, default suggested 15)
    - `iterate` (boolean)
    - `initial user question` (array with the question)
  - No user-facing answer should be generated here.
- **Inputs:** From `User asks a question` (chat payload).
- **Outputs:** To `Json Parser 1`.
- **Edge cases / failures:**
  - The schema in the system message has a subtle risk: missing comma between `"iterate"` and `"initial user question"` in the displayed schema block could bias the model into emitting invalid JSON. (It often still works, but it’s a real fragility.)
  - If the agent outputs markdown fences or commentary, parsing will fail downstream.

#### Node: Json Parser 1
- **Type / role:** `n8n-nodes-base.code` — converts Planning Agent output string into structured JSON.
- **Configuration (interpreted):**
  - Reads `const raw = $json.output;`
  - Strips ```json / ``` fences and `trim()`
  - `JSON.parse(cleaned)`
- **Inputs:** Output of `Planning Agent` (expects a field named `output` containing JSON text).
- **Outputs:** To `Check research q`.
- **Edge cases / failures:**
  - If `Planning Agent` returns invalid JSON, `JSON.parse` throws and the workflow errors.
  - If the agent output key isn’t `output` (varies by node version/settings), this code breaks.

#### Node: Check research q
- **Type / role:** `n8n-nodes-base.if` — routes to research pipeline vs general answering.
- **Condition:** `{{$json.need_search}}` is boolean `true`.
- **Outputs:**
  - **True branch →** `arXiv Search`
  - **False branch →** `General Reasoning Agent`
- **Edge cases / failures:**
  - If `need_search` is missing or not boolean, strict validation may route incorrectly or fail depending on n8n version/typeValidation behavior.

---

### Block 3 — Research Path: arXiv Retrieval + Parsing
**Overview:** Builds an arXiv API query from Planning Agent output, fetches XML, and extracts title/summary/link for each paper.  
**Nodes involved:** `arXiv Search`, `Normalize arXiv`, `Json Parser 2`

#### Node: arXiv Search
- **Type / role:** `n8n-nodes-base.httpRequest` — calls arXiv API.
- **Configuration (interpreted):**
  - URL is dynamically built:
    - Base: `https://export.arxiv.org/api/query`
    - `search_query=` is built from `$json.search_query` (array of phrases).
      - Each phrase is split into words, each word is wrapped as `all:<term>` and joined with `+AND+`.
      - Multiple phrases are joined with `+OR+`.
      - Hyphens in words are replaced with spaces before encoding.
    - Sorting: `sortBy=submittedDate&sortOrder=descending`
    - `max_results={{ $json.max_papers || 15 }}`
- **Inputs:** The JSON from `Check research q` true path (i.e., Planning output).
- **Outputs:** To `Normalize arXiv`.
- **Edge cases / failures:**
  - If `$json.search_query` is not an array, `.map` will throw (expression evaluation failure).
  - arXiv API rate limiting or intermittent failures (no retry configured here unless inherited globally).
  - The query builder uses `encodeURIComponent` per “word”; arXiv query syntax can be sensitive—multi-word phrases may not behave as expected.

#### Node: Normalize arXiv
- **Type / role:** `n8n-nodes-base.function` — wraps raw HTTP response into a predictable structure for parsing.
- **Configuration (interpreted):**
  - `const xml = $json; return [{ json: { raw: xml } }];`
  - This assumes the HTTP response is stored in `$json` (common in n8n).
- **Inputs:** HTTP response from `arXiv Search`.
- **Outputs:** To `Json Parser 2`.
- **Edge cases / failures:**
  - If HTTP Request returns binary or a different response shape, downstream code expecting `raw.data` may fail.

#### Node: Json Parser 2
- **Type / role:** `n8n-nodes-base.code` — parses arXiv Atom XML (string) via regex and builds paper objects.
- **Configuration (interpreted):**
  - Expects XML text at: `$json.raw.data`
  - Splits entries by `<entry>` tags.
  - Extracts:
    - `title` from `<title>...</title>`
    - `summary` from `<summary>...</summary>`
    - `arxiv_link` by scanning `<link .../>` tags where `rel="alternate"`
  - Also tries to capture the original user prompt:
    - `const originalPrompt = $node["Check research q"]?.json?.["initial user question"] ?? null;`
  - Returns a single item shaped like:
    - `{ papers: [ {json:{title, summary, arxiv_link}}, ... ], original_prompt: originalPrompt }`
- **Inputs:** Output of `Normalize arXiv` (raw HTTP response wrapper).
- **Outputs:** To `Arxiv Grounded Agent`.
- **Edge cases / failures:**
  - **Fragile XML parsing:** regex/splitting will break on unexpected formatting, namespaces, multiline attributes, or if `<entry>` appears in unexpected ways.
  - `raw.data` may not exist (depending on HTTP Request “Response Format” / n8n version).
  - `extractAllTags('category', ...)` is implemented but categories aren’t used in the returned object; also the regex won’t correctly read `<category term="..."/>` as a normal open/close tag.
  - `original_prompt` is taken from the IF node’s JSON; if the planning schema changes or key differs, it becomes `null` and later expressions can fail.

---

### Block 4 — Answer Generation (Grounded vs General)
**Overview:** Produces the final user-facing response: either grounded in arXiv paper metadata, or general knowledge only.  
**Nodes involved:** `Arxiv Grounded Agent`, `General Reasoning Agent`, `OpenAI Chat Model`

#### Node: Arxiv Grounded Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — summarization and grounded answering based strictly on supplied documents.
- **Key configuration choices:**
  - **Text prompt** is constructed from:
    - `{{$json.original_prompt[0]}}`
    - Then a concatenation of all papers:
      - title, summary, arxiv_link per paper (each paper separated by blank lines).
  - **System message** enforces:
    - No external knowledge, no invention, no restating question
    - Paper-by-paper summaries, include references only if present
    - Empty-input rule: output exactly `No relevant stored papers found.`
  - `hasOutputParser: true` (indicates structured output handling may be enabled internally, though output is requested as readable text).
  - `maxIterations: 10`
- **Inputs:** `Json Parser 2` output containing `papers` and `original_prompt`.
- **Outputs:** Final answer (no further nodes).
- **Edge cases / failures:**
  - If `original_prompt` is `null` or not an array, `original_prompt[0]` becomes `undefined` and the agent receives a malformed prompt.
  - Large number of papers/abstracts can exceed model context window.
  - The agent instructions contain a mild contradiction: it says “Do NOT answer the user’s question with your own knowledge” but later says “Answer the original user question using only the information explicitly provided”. This is workable, but can confuse the model if papers are thin.

#### Node: General Reasoning Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — general knowledge response when search is not needed.
- **Key configuration choices:**
  - **Text:** `{{ $json['initial user question'][0] }}`
  - **System message** requires:
    - Explicitly state the question is general and answer is based on general knowledge
    - No external sources
    - If cannot answer: output exactly `"No relevant stored papers found."`
  - `maxIterations: 5`
- **Inputs:** From `Check research q` false branch (planning JSON).
- **Outputs:** Final answer (no further nodes).
- **Edge cases / failures:**
  - If the planning JSON does not include `initial user question` as an array, the expression errors.
  - The fallback phrase mentions “stored papers,” which is semantically odd for general knowledge mode (but it’s enforced by the prompt).

---

### Block 5 — Annotations / Notes (Sticky Notes)
**Overview:** Documentation-only nodes that explain the workflow sections and link to resources.  
**Nodes involved:** `Sticky Note1`, `Sticky Note`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`

#### Sticky Note1
- Contains detailed usage notes, requirements, and links:
  - GitHub: https://github.com/msilaev/n8n-arxiv-search-agent
  - n8n Forum: https://community.n8n.io/

#### Sticky Note (General or research question?)
- Labels the planning/branching area.

#### Sticky Note2 / Sticky Note3 / Sticky Note4
- Label the arXiv request+parsing area, grounded response area, and general response area respectively.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User asks a question | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point (user question) | — | Planning Agent | ## Try It Out!… (full note includes links: https://github.com/msilaev/n8n-arxiv-search-agent ; https://community.n8n.io/) |
| Planning Agent | @n8n/n8n-nodes-langchain.agent | Decide whether arXiv search is required; emit JSON plan | User asks a question; OpenAI Chat Model (ai) | Json Parser 1 | ## General or research question? |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Shared LLM backend for agents | — | Planning Agent; General Reasoning Agent; Arxiv Grounded Agent (ai) |  |
| Json Parser 1 | n8n-nodes-base.code | Parse Planning Agent JSON text into object | Planning Agent | Check research q | ## General or research question? |
| Check research q | n8n-nodes-base.if | Branch: research vs general | Json Parser 1 | arXiv Search (true); General Reasoning Agent (false) | ## General or research question? |
| arXiv Search | n8n-nodes-base.httpRequest | Query arXiv API (Atom XML) | Check research q (true) | Normalize arXiv | ## API request to arXiv and response parsing |
| Normalize arXiv | n8n-nodes-base.function | Wrap raw HTTP response into `{ raw: ... }` | arXiv Search | Json Parser 2 | ## API request to arXiv and response parsing |
| Json Parser 2 | n8n-nodes-base.code | Extract papers (title/summary/link) from XML | Normalize arXiv | Arxiv Grounded Agent | ## API request to arXiv and response parsing |
| Arxiv Grounded Agent | @n8n/n8n-nodes-langchain.agent | Summarize papers and answer strictly from provided metadata | Json Parser 2; OpenAI Chat Model (ai) | — | ## Reply to Question Based on Documents Retrieved from arXiv |
| General Reasoning Agent | @n8n/n8n-nodes-langchain.agent | General-knowledge answer when search not required | Check research q (false); OpenAI Chat Model (ai) | — | ## Reply to Question Based on general knowlege |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Try It Out!… (includes links above) |
| Sticky Note | n8n-nodes-base.stickyNote | Section label | — | — | ## General or research question? |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section label | — | — | ## API request to arXiv and response parsing |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section label | — | — | ## Reply to Question Based on Documents Retrieved from arXiv |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section label | — | — | ## Reply to Question Based on general knowlege |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **“arXiv Autonomous Research Agent”**
   - (Optional) Keep execution order setting as default (`v1`).

2. **Add the Chat Trigger**
   - Node: **Chat Trigger** (`@n8n/n8n-nodes-langchain.chatTrigger`)
   - Name: **User asks a question**
   - Leave default options.
   - This is your entry point.

3. **Add the OpenAI chat model**
   - Node: **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
   - Name: **OpenAI Chat Model**
   - Configure:
     - Model: `gpt-4.1-mini`
     - Temperature: `0.7`
     - TopP: `0.9`
     - Max Tokens: `5000`
     - Max Retries: `10`
   - **Credentials:** Add/choose your OpenAI credentials (API key) in n8n.

4. **Add the Planning Agent**
   - Node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Name: **Planning Agent**
   - In **Options → System Message**, paste the planning instructions (the strict JSON-only policy and schema).
   - Connect:
     - **User asks a question → Planning Agent** (main)
     - **OpenAI Chat Model → Planning Agent** (ai_languageModel)

5. **Parse the planner JSON**
   - Node: **Code** (`n8n-nodes-base.code`)
   - Name: **Json Parser 1**
   - JavaScript:
     - Read `$json.output`, remove markdown fences, `JSON.parse(...)`.
   - Connect:
     - **Planning Agent → Json Parser 1**

6. **Branch on `need_search`**
   - Node: **IF** (`n8n-nodes-base.if`)
   - Name: **Check research q**
   - Condition:
     - Left value: `={{$json.need_search}}`
     - Operator: boolean is `true`
   - Connect:
     - **Json Parser 1 → Check research q**

7. **Research branch: call arXiv**
   - Node: **HTTP Request** (`n8n-nodes-base.httpRequest`)
   - Name: **arXiv Search**
   - Method: GET
   - URL: build using an expression that:
     - Converts `$json.search_query` array into arXiv query syntax with `all:` terms
     - Applies `max_results={{ $json.max_papers || 15 }}`
     - Sorts by submittedDate desc
   - Connect:
     - **Check research q (true) → arXiv Search**

8. **Normalize HTTP response**
   - Node: **Function** (`n8n-nodes-base.function`)
   - Name: **Normalize arXiv**
   - Code: wrap incoming `$json` into `{ raw: $json }`.
   - Connect:
     - **arXiv Search → Normalize arXiv**

9. **Parse arXiv XML into paper objects**
   - Node: **Code** (`n8n-nodes-base.code`)
   - Name: **Json Parser 2**
   - Code logic:
     - Read XML string from `$json.raw.data`
     - Split by `<entry>`
     - Extract `<title>`, `<summary>`, and the alternate link (`rel="alternate"`) into `arxiv_link`
     - Carry forward original question from `$node["Check research q"].json["initial user question"]`
     - Output: `{ papers: [...], original_prompt: ... }`
   - Connect:
     - **Normalize arXiv → Json Parser 2**

10. **Grounded arXiv answering agent**
   - Node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Name: **Arxiv Grounded Agent**
   - Prompt/Text: expression that concatenates:
     - The original prompt: `{{$json.original_prompt[0]}}`
     - Followed by each paper’s title, summary, link.
   - System message: enforce “no external knowledge”, paper-by-paper summaries, include references if provided, and empty-input rule.
   - Connect:
     - **Json Parser 2 → Arxiv Grounded Agent**
     - **OpenAI Chat Model → Arxiv Grounded Agent** (ai_languageModel)

11. **General branch: general reasoning agent**
   - Node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Name: **General Reasoning Agent**
   - Text: `={{ $json['initial user question'][0] }}`
   - System message: must state it’s general knowledge; no external sources; fallback exact string if cannot answer.
   - Connect:
     - **Check research q (false) → General Reasoning Agent**
     - **OpenAI Chat Model → General Reasoning Agent** (ai_languageModel)

12. **Add sticky notes (optional but recommended for maintainability)**
   - Add sticky notes for:
     - Overall description/links
     - Planning/branching section
     - arXiv API + parsing section
     - Grounded response section
     - General response section

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Try It Out!” note describing purpose, use cases, and requirements (OpenAI account, n8n install, optional .env). | Canvas documentation (Sticky Note1) |
| Self-hosting instructions on GitHub | https://github.com/msilaev/n8n-arxiv-search-agent |
| Community help | https://community.n8n.io/ |
| The workflow is demonstrated with a manual/chat-style entry but can be replaced with webhooks/forms/triggers. | Sticky Note1 content |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.