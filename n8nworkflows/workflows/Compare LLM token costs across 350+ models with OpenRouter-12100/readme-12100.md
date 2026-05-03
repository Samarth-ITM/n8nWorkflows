Compare LLM token costs across 350+ models with OpenRouter

https://n8nworkflows.xyz/workflows/compare-llm-token-costs-across-350--models-with-openrouter-12100


# Compare LLM token costs across 350+ models with OpenRouter

## 1. Workflow Overview

**Purpose:** This workflow lets you run a prompt against any OpenRouter-hosted LLM (350+ models) and **compute the exact token costs** using **live OpenRouter model pricing** plus **measured token usage** returned by the model call. It supports two user entry points:
- **Chat interface** (quick test; fixed model default)
- **Form interface** (choose model + prompt; shows a styled HTML result page)

### 1.1 Chat (Simple Testing via n8n Chat)
Receives a chat message, executes the “backend” logic inside the same workflow (as a self-call), then returns the model answer plus token/cost breakdown back to chat.

### 1.2 UI (Interactive Form)
Displays a form that first loads the full OpenRouter model list, builds a dropdown dynamically, then on submission calls the backend logic and finally shows a formatted HTML result page (answer + cost + details).

### 1.3 Backend (Model list → model call → token & cost computation)
Acts as an internal callable sub-flow (invoked via **Execute Workflow** nodes). It:
1) fetches OpenRouter model metadata (including pricing),
2) selects the requested model record,
3) runs the prompt via OpenRouter chat model (LangChain),
4) extracts token usage,
5) calculates costs,
6) returns structured results to the caller (chat or form completion).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Chat (simple testing via chat)
**Overview:** Handles ad-hoc testing from the n8n Chat panel. Uses a fixed default model and returns a markdown-formatted answer with token and cost details.

**Nodes involved:**
- When chat message received
- Backend1
- Respond to Chat

#### Node: **When chat message received**
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — chat entry trigger.
- **Key configuration:**
  - Response mode: **responseNodes** (the workflow returns content via a response node, here “Respond to Chat”).
- **Inputs / outputs:**
  - Output → **Backend1**
- **Edge cases / failures:**
  - If chat input is empty/unexpected, downstream nodes may compute costs with `null` token usage.
  - Response mode requires a proper response node to complete the chat request.

#### Node: **Backend1**
- **Type / role:** `n8n-nodes-base.executeWorkflow` — calls a workflow (here: itself) to run the backend.
- **Key configuration:**
  - Workflow ID: `={{ $workflow.id }}` (self-invocation pattern).
  - Inputs:
    - `chatInput: {{ $json.chatInput }}`
    - `model: "openai/gpt-4.1-mini"` (hardcoded default)
  - Converts fields to string: enabled.
- **Inputs / outputs:**
  - Input from: **When chat message received**
  - Output → **Respond to Chat**
- **Version-specific notes:**
  - Uses Execute Workflow node v1.3 behavior for “workflowInputs” mapping.
- **Edge cases / failures:**
  - Self-call recursion risk is mitigated because the self-called branch starts at **ExecuteWorkflowTrigger**, not at the chat trigger.
  - If the backend branch errors (HTTP auth, model failure), chat will fail to respond unless error handling is added.

#### Node: **Respond to Chat**
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — sends the final message to chat.
- **Key configuration:**
  - Message is templated from backend output:
    - `$json.output`
    - `$json.model.id`, `$json.model.name`
    - `$json.tokenUsage.*`
    - `$json.costs.*` including `toFixed()`
- **Inputs / outputs:**
  - Input from: **Backend1**
  - No outgoing connections.
- **Edge cases / failures:**
  - If `tokenUsage` is `null`, expressions like `{{ $json.tokenUsage.input_tokens }}` will error. (This can happen if the model/provider does not populate `usage_metadata`.)

---

### Block 2.2 — UI (form start → model dropdown)
**Overview:** On form trigger, fetches the OpenRouter model catalog, turns it into dropdown options, and renders a custom styled form.

**Nodes involved:**
- On form submission
- Get Openrouter models
- Split Out
- Code in JavaScript1
- Form

#### Node: **On form submission**
- **Type / role:** `n8n-nodes-base.formTrigger` — starts the UI flow.
- **Key configuration:**
  - Title: **LLM Token Cost Tracker**
  - Description: explains features and usage.
  - Button label: **Start**
  - Custom CSS: dark theme styling for inputs/buttons.
- **Inputs / outputs:**
  - Output → **Get Openrouter models**
- **Edge cases / failures:**
  - If n8n instance cannot expose webhook URLs, the form won’t load externally.
  - If user refreshes, may re-trigger.

#### Node: **Get Openrouter models**
- **Type / role:** `n8n-nodes-base.httpRequest` — calls OpenRouter models endpoint.
- **Key configuration:**
  - URL: `https://openrouter.ai/api/v1/models`
  - Auth: **HTTP Bearer Auth** via generic credential type.
  - Sends headers: enabled.
- **Inputs / outputs:**
  - Input from: **On form submission**
  - Output → **Split Out**
- **Edge cases / failures:**
  - 401/403 if API key missing/invalid.
  - Rate limits / transient 5xx from OpenRouter.
  - Response shape changes could break downstream split (expects a top-level `data` array).

#### Node: **Split Out**
- **Type / role:** `n8n-nodes-base.splitOut` — converts `data[]` into one item per model.
- **Key configuration:**
  - Field to split: `data`
- **Inputs / outputs:**
  - Input from: **Get Openrouter models**
  - Output → **Code in JavaScript1**
- **Edge cases / failures:**
  - If `data` missing/not an array, node errors.

#### Node: **Code in JavaScript1**
- **Type / role:** `n8n-nodes-base.code` — builds the form JSON schema dynamically.
- **Key configuration (interpreted):**
  - Creates dropdown values as `{ option: item.json.id }` (important: Form node expects `option`, not `name/value`).
  - Sorts options alphabetically.
  - Builds `formJson` with:
    1) dropdown “Select Model”
    2) textarea “Prompt”
  - Returns one item with:
    - `formJson`
    - `templates: items.map(i => i.json)` (passed through but not used later in this workflow)
- **Inputs / outputs:**
  - Input from: **Split Out**
  - Output → **Form**
- **Edge cases / failures:**
  - If some items lack `id`, dropdown entries may be blank.
  - Large model list may create a very large form JSON; could impact load time.

#### Node: **Form**
- **Type / role:** `n8n-nodes-base.form` — renders the model selection form.
- **Key configuration:**
  - Define form: **json**
  - JSON output: `={{ $json.formJson }}`
  - Custom CSS: dark theme styling.
- **Inputs / outputs:**
  - Input from: **Code in JavaScript1**
  - Output → **Backend**
- **Edge cases / failures:**
  - If the incoming JSON is not exactly the expected structure, the form may not render.
  - Dropdown with 350+ entries may be unwieldy; consider search UI (not included here).

---

### Block 2.3 — Backend (callable): select model → run LLM → compute costs
**Overview:** This is the core logic, invoked by both chat and form branches via a self-call. It fetches model pricing, runs the selected model with LangChain, extracts token usage, and calculates USD costs.

**Nodes involved:**
- When Executed by Another Workflow
- Get Openrouter models1
- Split Out1
- Filter
- OpenRouter
- LangChain Code
- Code in JavaScript

#### Node: **When Executed by Another Workflow**
- **Type / role:** `n8n-nodes-base.executeWorkflowTrigger` — entry point for sub-workflow execution.
- **Key configuration:**
  - Declares expected inputs:
    - `chatInput`
    - `model`
- **Inputs / outputs:**
  - Output → **Get Openrouter models1**
- **Edge cases / failures:**
  - If caller doesn’t provide `model`, downstream filter will fail to match and may produce no items.
  - If caller doesn’t provide `chatInput`, model call may produce irrelevant output or error depending on provider.

#### Node: **Get Openrouter models1**
- **Type / role:** `n8n-nodes-base.httpRequest` — fetch model list again (backend uses live pricing at runtime).
- **Key configuration:** same as “Get Openrouter models”:
  - URL: `https://openrouter.ai/api/v1/models`
  - Bearer auth credential
- **Inputs / outputs:**
  - Input from: **When Executed by Another Workflow**
  - Output → **Split Out1**
- **Edge cases / failures:** same as the UI fetch node (auth, rate limit, shape changes).

#### Node: **Split Out1**
- **Type / role:** `n8n-nodes-base.splitOut` — one item per model from `data[]`.
- **Key configuration:**
  - Field to split: `data`
- **Inputs / outputs:**
  - Output → **Filter**
- **Edge cases:** `data` not present or non-array.

#### Node: **Filter**
- **Type / role:** `n8n-nodes-base.filter` — selects the model record matching the requested model id.
- **Key configuration:**
  - Condition: `$json.id` **equals** `$('When Executed by Another Workflow').item.json.model`
  - Strict type validation enabled.
- **Inputs / outputs:**
  - Input from: **Split Out1**
  - Output → **LangChain Code** (only matching model item continues)
- **Edge cases / failures:**
  - If no model matches, **no items** go forward → later nodes may not execute (or callers receive empty results).
  - If multiple items match (unlikely), downstream `$().first()` usage will pick the first.
  - Any mismatch in model string (case, whitespace) causes drop.

#### Node: **OpenRouter**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides the chat LLM connection to LangChain.
- **Key configuration:**
  - Model: `={{ $('When Executed by Another Workflow').item.json.model }}`
  - Credential: OpenRouter API key.
- **Inputs / outputs:**
  - Output (ai_languageModel) → **LangChain Code** (LLM input)
- **Edge cases / failures:**
  - Invalid model id results in OpenRouter error.
  - Auth errors if credential misconfigured.
  - Provider may not return token usage metadata consistently.

#### Node: **LangChain Code**
- **Type / role:** `@n8n/n8n-nodes-langchain.code` — executes a LangChain chain and attempts to capture token usage.
- **Key configuration (interpreted):**
  - Pulls LLM from `ai_languageModel` input.
  - Reads `chatInput` from ExecuteWorkflowTrigger:  
    `$('When Executed by Another Workflow').first().json.chatInput`
  - Uses `PromptTemplate.fromTemplate('{input}')` then `prompt.pipe(llm)` and `chain.invoke(...)`.
  - Adds a callback `handleLLMEnd` to read `message.usage_metadata` and store:
    - input_tokens, output_tokens, total_tokens
  - Returns:
    - `output: response.content`
    - `tokenUsage: tokenUsage` (may remain `null` if not captured)
- **Inputs / outputs:**
  - Main input: from **Filter** (model record flows through the “main” line, though the code mostly ignores it)
  - ai_languageModel input: from **OpenRouter**
  - Output → **Code in JavaScript**
- **Version-specific notes:**
  - Requires n8n LangChain Code node support for `this.getInputConnectionData('ai_languageModel', 0)`.
- **Edge cases / failures:**
  - Token usage capture depends on OpenRouter/LangChain returning `usage_metadata`. If absent, `tokenUsage` stays `null`.
  - If OpenRouter returns multiple generations or different structure, callback parsing could break.
  - Timeout on long prompts/completions.

#### Node: **Code in JavaScript**
- **Type / role:** `n8n-nodes-base.code` — computes USD costs from usage + pricing and shapes final output.
- **Key configuration (interpreted):**
  - Reads selected model data via: `const modelData = $('Filter').first().json;`
  - Pricing expected at: `modelData.pricing.prompt` and `modelData.pricing.completion` (strings representing **USD per token**).
  - Reads token usage from incoming json: `$json.tokenUsage`.
  - Computes:
    - `inputCost = input_tokens * promptPrice`
    - `outputCost = output_tokens * completionPrice`
    - `totalCost = inputCost + outputCost`
  - Returns a structured object:
    - `output`
    - `tokenUsage`
    - `costs` (raw + formatted)
    - `model` (id, name, prompt/completion per-token prices)
- **Inputs / outputs:**
  - Input from: **LangChain Code**
  - Output goes to whichever caller invoked the backend (chat or form). In this workflow’s graph:
    - Backend branch output is consumed by **Form1**
    - Chat branch output is consumed by **Respond to Chat**
- **Edge cases / failures:**
  - If `tokenUsage` is `null`, `tokenUsage.input_tokens` will throw.
  - If pricing fields are missing or not parseable, `parseFloat()` yields `NaN` and costs become `NaN`.
  - Assumes OpenRouter pricing is already “per token” (not per 1K tokens). If OpenRouter ever changes units, results would be wrong.

---

### Block 2.4 — UI Completion (display results)
**Overview:** After the user submits the model+prompt form, the workflow calls the backend and renders an HTML page showing the answer, total cost, and expandable details.

**Nodes involved:**
- Backend
- Form1

#### Node: **Backend**
- **Type / role:** `n8n-nodes-base.executeWorkflow` — self-call to backend branch.
- **Key configuration:**
  - Workflow ID: `={{ $workflow.id }}`
  - Inputs mapped from the form submission:
    - `model: {{ $json["Select Model"] }}`
    - `chatInput: {{ $json.Prompt }}`
- **Inputs / outputs:**
  - Input from: **Form**
  - Output → **Form1**
- **Edge cases / failures:**
  - If form field labels change, expressions break (they are label-based keys).
  - Same backend risks as above (auth, model errors, token metadata missing).

#### Node: **Form1**
- **Type / role:** `n8n-nodes-base.form` (operation: completion) — returns a final HTML response page.
- **Key configuration:**
  - Operation: **completion**
  - Respond with: **showText**
  - ResponseText: full HTML template interpolating:
    - `{{ $json.output }}`
    - `{{ $json.model.id }}`, `{{ $json.model.name }}`
    - `{{ $json.costs.total_cost_formatted }}` and raw cost fields
    - token usage fields
    - model pricing per token fields
- **Inputs / outputs:**
  - Input from: **Backend**
  - No outgoing connections.
- **Edge cases / failures:**
  - If output contains HTML, it will be injected (no escaping). Consider sanitization if untrusted input is possible.
  - If token usage/cost fields are missing, the page may render blank placeholders or error depending on n8n templating evaluation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point | — | Backend1 | ## Chat (simple testing via chat) |
| Backend1 | n8n-nodes-base.executeWorkflow | Self-call to backend (chat default model) | When chat message received | Respond to Chat | ## Chat (simple testing via chat) |
| Respond to Chat | @n8n/n8n-nodes-langchain.chat | Send response to chat | Backend1 | — | ## Chat (simple testing via chat) |
| On form submission | n8n-nodes-base.formTrigger | UI entry point | — | Get Openrouter models | ## UI |
| Get Openrouter models | n8n-nodes-base.httpRequest | Fetch models list for dropdown | On form submission | Split Out | ## UI |
| Split Out | n8n-nodes-base.splitOut | Split models array into items | Get Openrouter models | Code in JavaScript1 | ## UI |
| Code in JavaScript1 | n8n-nodes-base.code | Build dynamic form JSON | Split Out | Form | ## UI |
| Form | n8n-nodes-base.form | Render selection form | Code in JavaScript1 | Backend | ## UI |
| Backend | n8n-nodes-base.executeWorkflow | Self-call to backend (form-selected model/prompt) | Form | Form1 | ## UI |
| Form1 | n8n-nodes-base.form | Render HTML completion page | Backend | — | ## UI |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Backend entry point (callable) | — | Get Openrouter models1 | ## Backend |
| Get Openrouter models1 | n8n-nodes-base.httpRequest | Fetch models list (for live pricing) | When Executed by Another Workflow | Split Out1 | ## Backend |
| Split Out1 | n8n-nodes-base.splitOut | Split models array into items | Get Openrouter models1 | Filter | ## Backend |
| Filter | n8n-nodes-base.filter | Select requested model item | Split Out1 | LangChain Code | ## Backend |
| OpenRouter | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider connection | — | LangChain Code (ai_languageModel) | ## Backend |
| LangChain Code | @n8n/n8n-nodes-langchain.code | Run LLM + capture token usage | Filter; OpenRouter (ai_languageModel) | Code in JavaScript | ## Backend |
| Code in JavaScript | n8n-nodes-base.code | Compute costs and format output | LangChain Code | (returned to caller) | ## Backend |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block | — | — | # LLM Token Cost Tracker… (see Notes) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Chat (simple testing via chat) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block | — | — | ## UI |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block | — | — | ## Backend |

> Note: Sticky notes are not executable nodes; they are listed here only to ensure nothing is skipped.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named: *Compare LLM token costs across 350+ models with OpenRouter*.

2) **Create OpenRouter credentials**
   - Create **HTTP Bearer Auth** credential (for HTTP Request nodes):
     - Token: your OpenRouter API key.
   - Create **OpenRouter API** credential (for the OpenRouter LangChain node):
     - API key: same OpenRouter key.

### A) Backend (callable sub-flow inside same workflow)
3) Add **Execute Workflow Trigger** node:
   - Name: *When Executed by Another Workflow*
   - Define inputs: `chatInput` (string), `model` (string)

4) Add **HTTP Request** node:
   - Name: *Get Openrouter models1*
   - Method: GET
   - URL: `https://openrouter.ai/api/v1/models`
   - Authentication: *Generic credential type* → *HTTP Bearer Auth*
   - Select your bearer credential

5) Add **Split Out** node:
   - Name: *Split Out1*
   - Field to split out: `data`

6) Add **Filter** node:
   - Name: *Filter*
   - Condition (String → equals):
     - Left: `{{ $json.id }}`
     - Right: `{{ $('When Executed by Another Workflow').item.json.model }}`

7) Add **OpenRouter Chat Model** node (LangChain):
   - Name: *OpenRouter*
   - Model: `{{ $('When Executed by Another Workflow').item.json.model }}`
   - Credentials: OpenRouter API credential

8) Add **LangChain Code** node:
   - Name: *LangChain Code*
   - Ensure it has **two inputs**: `main` and `ai_languageModel`
   - Paste code (adapt if needed) that:
     - pulls `llm` from `ai_languageModel`
     - reads `chatInput` from the trigger
     - invokes the chain
     - captures `message.usage_metadata` into `tokenUsage`
     - returns `{ output, tokenUsage }`

9) Add **Code (JavaScript)** node:
   - Name: *Code in JavaScript*
   - Implement:
     - `modelData = $('Filter').first().json`
     - `pricing = modelData.pricing`
     - compute costs from `$json.tokenUsage`
     - return `output`, `tokenUsage`, `costs`, `model`

10) **Connect backend nodes**
   - When Executed by Another Workflow → Get Openrouter models1 → Split Out1 → Filter → LangChain Code (main)
   - OpenRouter (ai_languageModel output) → LangChain Code (ai_languageModel input)
   - LangChain Code → Code in JavaScript

### B) Chat entry path
11) Add **Chat Trigger** node:
   - Name: *When chat message received*
   - Options: Response mode = **responseNodes**

12) Add **Execute Workflow** node:
   - Name: *Backend1*
   - Workflow: select by ID expression `{{ $workflow.id }}` (self)
   - Inputs:
     - `chatInput: {{ $json.chatInput }}`
     - `model: openai/gpt-4.1-mini`

13) Add **Respond to Chat** node:
   - Name: *Respond to Chat*
   - Message: build a markdown message using fields from backend output:
     - `$json.output`, `$json.costs.total_cost_formatted`, token fields, pricing fields

14) Connect:
   - When chat message received → Backend1 → Respond to Chat

### C) Form UI path
15) Add **Form Trigger** node:
   - Name: *On form submission*
   - Title: “LLM Token Cost Tracker”
   - Description: (your text)
   - Button label: “Start”
   - Optional: paste the provided custom CSS (dark theme)

16) Add **HTTP Request** node:
   - Name: *Get Openrouter models*
   - GET `https://openrouter.ai/api/v1/models`
   - Auth: HTTP Bearer Auth credential

17) Add **Split Out** node:
   - Name: *Split Out*
   - Field to split out: `data`

18) Add **Code (JavaScript)** node:
   - Name: *Code in JavaScript1*
   - Build `formJson` with:
     - dropdown labeled **Select Model** with `values: [{ option: "<model id>"}, ...]`
     - textarea labeled **Prompt**
   - Return a single item: `{ formJson }`

19) Add **Form** node:
   - Name: *Form*
   - Define form: **json**
   - JSON output: `{{ $json.formJson }}`
   - Optional: paste the custom CSS

20) Add **Execute Workflow** node:
   - Name: *Backend*
   - Workflow: `{{ $workflow.id }}` (self)
   - Inputs:
     - `model: {{ $json["Select Model"] }}`
     - `chatInput: {{ $json.Prompt }}`

21) Add **Form** node (completion page):
   - Name: *Form1*
   - Operation: **completion**
   - Respond with: **showText**
   - ResponseText: HTML template interpolating backend result fields

22) Connect:
   - On form submission → Get Openrouter models → Split Out → Code in JavaScript1 → Form → Backend → Form1

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get your OpenRouter API Key: create at OpenRouter settings | https://openrouter.ai/settings/keys |
| OpenRouter model catalog & pricing | https://openrouter.ai/models |
| OpenRouter homepage | https://openrouter.ai/ |
| Workflow note: “Made with ❤️ by philflow for the n8n community” | From sticky note content |
| Important setup note: configure both HTTP Bearer Auth (HTTP nodes) and OpenRouter API credential (LLM node) using the same API key | Applies to Get Openrouter models / Get Openrouter models1 and OpenRouter node |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.