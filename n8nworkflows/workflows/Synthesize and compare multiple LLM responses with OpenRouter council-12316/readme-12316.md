Synthesize and compare multiple LLM responses with OpenRouter council

https://n8nworkflows.xyz/workflows/synthesize-and-compare-multiple-llm-responses-with-openrouter-council-12316


# Synthesize and compare multiple LLM responses with OpenRouter council

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow (“LLM Council”) takes a user’s chat question, queries multiple LLMs via **OpenRouter**, anonymizes their answers, has the same set of LLMs **evaluate and rank** the anonymized answers, computes an **aggregate ranking**, and then asks a designated “Chairman” model to **synthesize a final verdict**. Finally, it emails a complete report (question, verdict, rankings, and raw answers).

**Target use cases:**
- Reducing single-model bias by comparing multiple models.
- Producing higher-quality final answers via peer review + synthesis.
- Auditing model responses by keeping a transparent ranking trail.

### Logical blocks
1. **1.1 Input Reception & Initialization**: receive chat input, define member models + chairman.
2. **1.2 Stage 1 – Collect Council Answers**: query each model, extract text, aggregate.
3. **1.3 Stage 2 – Anonymize & Peer-Rank**: label answers (A/B/C…), create ranking prompt, query each model to rank, parse + aggregate results.
4. **1.4 Stage 3 – Aggregate Ranking + Chairman Verdict**: compute average ranks per model; build chairman prompt and query chairman model.
5. **1.5 Stage 4 – Reporting & Email Output**: format report text and send via SMTP email.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Initialization
**Overview:**  
Triggers on a chat message, stores the user query, defines the council member model list, and selects the chairman model.

**Nodes involved:**
- When chat message received
- Initialize Variables
- Split by Model

#### Node: **When chat message received**
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Entry point; receives chat input as `$json.chatInput`.
- **Configuration choices:** default options; exposes a webhook endpoint (internal to n8n chat trigger).
- **Key variables:** `$json.chatInput`
- **Connections:** outputs to **Initialize Variables**.
- **Failure modes / edge cases:**
  - Empty or missing `chatInput` leads to empty prompts downstream.
  - If used outside an n8n chat environment, it may never fire.

#### Node: **Initialize Variables**
- **Type / Role:** `Set` — Creates canonical fields used everywhere.
- **Configuration choices (interpreted):**
  - `userQuery` = `{{$json.chatInput}}`
  - `councilModels` = array of OpenRouter model identifiers:
    - `openai/gpt-4o`
    - `google/gemini-2.5-flash`
    - `anthropic/claude-sonnet-4.5`
    - `perplexity/sonar-pro-search`
  - `chairmanModel` = `google/gemini-2.5-flash`
- **Connections:** outputs to **Split by Model**.
- **Edge cases / failure types:**
  - Invalid model IDs will cause OpenRouter HTTP 4xx errors later.
  - Very large model list can increase runtime and cost.

#### Node: **Split by Model**
- **Type / Role:** `Split Out` — Iterates council models: one item per model.
- **Configuration choices:**
  - Splits `councilModels` into separate items.
  - Includes `userQuery` explicitly via `include = {{$json.userQuery}}` (keeps it on each item).
- **Connections:** outputs to **Stage 1: Query Model Answers**.
- **Edge cases:**
  - If `councilModels` isn’t an array, split fails (expression/type issue).

---

### 2.2 Stage 1 – Collect Council Answers
**Overview:**  
Calls OpenRouter for each model with the same user question, extracts each response, then aggregates all answers into a single structure for later processing.

**Nodes involved:**
- Stage 1: Query Model Answers
- Stage 1: Extract Responses
- Stage 1: Aggregate Responses

#### Node: **Stage 1: Query Model Answers**
- **Type / Role:** `HTTP Request` — Calls OpenRouter `/chat/completions` for each council model.
- **Configuration choices:**
  - `POST https://openrouter.ai/api/v1/chat/completions`
  - JSON body (stringified) with:
    - `model: $json.councilModels` (after Split, this is the *current* model string)
    - `messages: [{ role: "user", content: $('Initialize Variables').item.json.userQuery }]`
  - Timeout: 120s
  - Auth: `openRouterApi` predefined credential
  - Header: `Content-Type: application/json`
- **Connections:** outputs to **Stage 1: Extract Responses**.
- **Failure modes / edge cases:**
  - Auth/credential misconfiguration (401/403).
  - Rate limiting (429), model downtime (5xx).
  - Response schema mismatch (if OpenRouter changes fields).
  - Large prompts/answers could hit token limits (OpenRouter/model-specific).

#### Node: **Stage 1: Extract Responses**
- **Type / Role:** `Set` — Normalizes OpenRouter response into `{ model, response }`.
- **Configuration choices:**
  - `model = {{$json.model}}`
  - `response = {{$json.choices[0].message.content}}`
- **Connections:** outputs to **Stage 1: Aggregate Responses**.
- **Edge cases:**
  - If `choices[0].message.content` is missing (tool calls, refusal formats, API changes), expression fails or produces `undefined`.

#### Node: **Stage 1: Aggregate Responses**
- **Type / Role:** `Aggregate` — Collects all model response items into one.
- **Configuration choices:** `aggregateAllItemData` (produces a `data` array containing each item).
- **Connections:** outputs to **Stage 2: Create Anonymized Responses**.
- **Edge cases:**
  - If one model call fails and workflow stops, aggregation never completes (unless error handling is added).

---

### 2.3 Stage 2 – Anonymize & Peer-Rank
**Overview:**  
Labels each model response with an anonymous letter (A, B, C…), builds a strict ranking prompt, asks each model to evaluate and rank responses, parses the ranking section, and aggregates all ranking outputs.

**Nodes involved:**
- Stage 2: Create Anonymized Responses
- Stage 2: Build Ranking Prompt
- Stage 2: Re-Add councilModels
- Stage 2: rankingPrompt per LLM
- Stage 2: Query Model Answer Rankings
- Stage 2: Extract Rankings
- Stage 2: Aggregate

#### Node: **Stage 2: Create Anonymized Responses**
- **Type / Role:** `Set` — Builds anonymized, labeled response list and a combined text block.
- **Configuration choices:**
  - `labeledResponses` (stored as a stringified JSON array):
    - Maps aggregated `data` to `{ label: "A"/"B"/..., model, response }`
    - Uses ASCII letters: `String.fromCharCode(65 + index)`
  - `anonymizedText`:
    - Produces:
      - `Response A:\n...`
      - `Response B:\n...`
      - etc.
- **Connections:** outputs to **Stage 2: Build Ranking Prompt**.
- **Edge cases / important note:**
  - `labeledResponses` is created with `JSON.stringify(...)` but declared as type `array`. In practice, this can become a **string instead of an array** depending on n8n handling. Later nodes assume it is an array (e.g., `.map(...)`). If it becomes a string, downstream `.map` will fail.
  - If there are >26 models, labels go beyond `Z` (non-alphabet characters).

#### Node: **Stage 2: Build Ranking Prompt**
- **Type / Role:** `Set` — Creates a detailed evaluation + ranking instruction prompt.
- **Configuration choices:**
  - Uses the original question: `$('Initialize Variables').first().json.userQuery`
  - Embeds `$json.anonymizedText`
  - Enforces an exact section header: `FINAL RANKING:` and numbered lines like `1. Response A`
- **Connections:** outputs to **Stage 2: Re-Add councilModels**.
- **Edge cases:**
  - Models may not comply with formatting; parsing relies on strict marker `FINAL RANKING:`.

#### Node: **Stage 2: Re-Add councilModels**
- **Type / Role:** `Set` — Ensures both `rankingPrompt` and `councilModels` exist before splitting.
- **Configuration choices:**
  - `rankingPrompt = {{$json.rankingPrompt}}`
  - `councilModels = {{ $('Initialize Variables').item.json.councilModels }}`
- **Connections:** outputs to **Stage 2: rankingPrompt per LLM**.
- **Edge cases:**
  - If initialization node isn’t accessible by item linkage (rare in certain execution contexts), expression resolution may fail.

#### Node: **Stage 2: rankingPrompt per LLM**
- **Type / Role:** `Split Out` — Creates one item per model for ranking step.
- **Configuration choices:** `fieldToSplitOut = councilModels`, includes all other fields.
- **Connections:** outputs to **Stage 2: Query Model Answer Rankings**.

#### Node: **Stage 2: Query Model Answer Rankings**
- **Type / Role:** `HTTP Request` — For each model, requests evaluation + ranking of anonymized responses.
- **Configuration choices:**
  - POST OpenRouter `/chat/completions`
  - `model: $json.councilModels` (current model)
  - prompt content: `$json.rankingPrompt`
  - Timeout 120s, OpenRouter credential, JSON content-type.
- **Connections:** outputs to **Stage 2: Extract Rankings**.
- **Failure modes:**
  - Same as Stage 1 HTTP failures.
  - Some models may refuse to rank, or may produce tool output rather than a plain message.

#### Node: **Stage 2: Extract Rankings**
- **Type / Role:** `Set` — Extracts ranking text and parses labels.
- **Configuration choices:**
  - `model = {{$json.model}}`
  - `ranking = {{$json.choices[0].message.content}}`
  - `parsedRanking = {{ $json.choices[0].message.content.split('FINAL RANKING:')[1]?.match(/Response [A-Z]/g) || [] }}`
    - Extracts occurrences like `Response A`, `Response B`, etc.
- **Connections:** outputs to **Stage 2: Aggregate**.
- **Edge cases:**
  - If model omits `FINAL RANKING:` exactly, `split()[1]` is undefined and parsedRanking becomes `[]`.
  - Regex only captures single-letter labels A–Z.

#### Node: **Stage 2: Aggregate**
- **Type / Role:** `Aggregate` — Collects all per-model ranking outputs into one.
- **Configuration choices:** `aggregateAllItemData` (creates `data` array).
- **Connections:** outputs to **Stage 3: Calculate Aggregate Rankings**.

---

### 2.4 Stage 3 – Aggregate Ranking + Chairman Verdict
**Overview:**  
Computes an average rank per original responding model (based on peer rankings), builds a prompt containing Stage 1 answers and Stage 2 ranking commentary, then calls the chairman model to synthesize a final answer.

**Nodes involved:**
- Stage 3: Calculate Aggregate Rankings
- Stage 3: Build ChairmanPrompt Part 1
- Stage 3: Build Chairman Prompt Part 2
- Stage 3: Chairman Synthesis

#### Node: **Stage 3: Calculate Aggregate Rankings**
- **Type / Role:** `Code` — Aggregates rankings into average positions per model.
- **Configuration choices / logic:**
  - Inputs: all items from **Stage 2: Aggregate** (via `$input.all()`).
  - Pulls `labeledResponses` from **Stage 2: Create Anonymized Responses**.
  - Builds `labelToModel` map: `"Response A" -> "openai/gpt-4o"` etc.
  - For each ranking list, records the position (1..n) assigned to each response label.
  - Computes average rank per model and sorts ascending (lower = better).
  - Outputs:
    - `aggregateRankings` (array)
    - `labelToModel` (object)
    - `stage2Results` (raw ranking JSONs)
- **Connections:** outputs to **Stage 3: Build ChairmanPrompt Part 1**.
- **Edge cases / failure types:**
  - If `labeledResponses` is not an array (see Stage 2 stringify issue), `.forEach` will fail.
  - If many models return empty `parsedRanking`, averages may be based on sparse data.
  - This code assumes a particular nested structure (`ranking.json.data`); changes in upstream aggregation structure break it.

#### Node: **Stage 3: Build ChairmanPrompt Part 1**
- **Type / Role:** `Set` — Builds text sections for chairman prompt.
- **Configuration choices:**
  - `stage1Text` is built from **Stage 1: Aggregate Responses**:
    - Defensive access: `item.json.data || item.json[0].data`
    - Format: `Model: X\nResponse: Y`
  - `stage2Text` uses `stage2Results[0].data` from prior code output:
    - Format: `Model: X\nRanking: <full text>`
  - `includeOtherFields: true` (keeps existing JSON fields too).
- **Connections:** outputs to **Stage 3: Build Chairman Prompt Part 2**.
- **Edge cases:**
  - If the aggregate node output shape differs, `...json[0].data` workaround may still fail.
  - Assumes `stage2Results[0].data` exists.

#### Node: **Stage 3: Build Chairman Prompt Part 2**
- **Type / Role:** `Set` — Produces the final `chairmanPrompt` and attaches `chairmanModel`.
- **Configuration choices:**
  - `chairmanPrompt` includes:
    - Original question
    - Stage 1 responses (non-anonymized, with model names)
    - Stage 2 peer rankings (per model)
    - Instruction to synthesize a final, well-reasoned answer
  - `chairmanModel = {{ $('Initialize Variables').item.json.chairmanModel }}`
- **Connections:** outputs to **Stage 3: Chairman Synthesis**.

#### Node: **Stage 3: Chairman Synthesis**
- **Type / Role:** `HTTP Request` — Calls OpenRouter with chairman model to generate final verdict.
- **Configuration choices:**
  - POST `/chat/completions`
  - `model: $json.chairmanModel`
  - message content: `$json.chairmanPrompt`
  - Timeout 120s
  - Auth: OpenRouter credential explicitly set here (`OpenRouter account`)
- **Connections:** outputs to **Stage 4: Format Output Part 1**.
- **Failure modes:**
  - Same OpenRouter issues; chairman prompt can be large (token/latency risk).

---

### 2.5 Stage 4 – Reporting & Email Output
**Overview:**  
Formats a comprehensive report body and emails it via SMTP.

**Nodes involved:**
- Stage 4: Format Output Part 1
- Stage 4: Format Output Part 2
- Stage 4: Send Result per Email

#### Node: **Stage 4: Format Output Part 1**
- **Type / Role:** `Set` — Extracts final response and references aggregate rankings.
- **Configuration choices:**
  - `chairmanModel` from initialization
  - `finalResponse = {{$json.choices[0].message.content}}`
  - `aggregateRankings = {{ $('Stage 3: Calculate Aggregate Rankings').item.json.aggregateRankings }}`
  - `stage1Responses = {{ $('Stage 1: Aggregate Responses').item.json }}` (note: stored but not directly used later)
- **Connections:** outputs to **Stage 4: Format Output Part 2**.
- **Edge cases:**
  - If chairman response content is missing, report will be empty or fail.

#### Node: **Stage 4: Format Output Part 2**
- **Type / Role:** `Set` — Builds `MailBody` text with question, verdict, rankings, and original answers.
- **Configuration choices (major embedded expressions):**
  - Prints original question: `$('Initialize Variables').first().json.userQuery`
  - Prints chairman model + final response
  - Prints aggregate rankings from `Stage 3: Calculate Aggregate Rankings`:
    - `model`, `averageRank`, `rankingsCount`
  - Prints per-model parsed rankings from `Stage 2: Aggregate`
  - Prints individual anonymized labeled answers from `Stage 2: Create Anonymized Responses`:
    - `labeledResponses.map(...)`
- **Connections:** outputs to **Stage 4: Send Result per Email**.
- **Edge cases:**
  - If `labeledResponses` is a string (due to JSON.stringify earlier), `.map` will fail here.
  - If some `parsedRanking` arrays are empty, the per-model “Ranked” section becomes blank.

#### Node: **Stage 4: Send Result per Email**
- **Type / Role:** `Email Send` — Sends the report via SMTP.
- **Configuration choices:**
  - `toEmail` / `fromEmail`: `user@example.com` (placeholders)
  - Subject: `LLM-Council-Resultat vom: <date>` (en-GB date)
  - Body: plain text from `{{$json.MailBody}}`
  - `appendAttribution: false`
  - Credentials: `SMTP account`
- **Connections:** terminal node.
- **Failure modes:**
  - SMTP auth failures, relay restrictions, invalid from/to, provider rate limits.
  - Very large email bodies may be rejected.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | LangChain Chat Trigger | Entry point: receive user chat query | — | Initialize Variables | ## INPUT and INITIALIZATION\n- Ask the question\n- Initialize the models |
| Initialize Variables | Set | Define userQuery, councilModels, chairmanModel | When chat message received | Split by Model | ## INPUT and INITIALIZATION\n- Ask the question\n- Initialize the models |
| Split by Model | Split Out | Iterate over council member models for Stage 1 | Initialize Variables | Stage 1: Query Model Answers | ## Stage 1: Individual Model Responses\n- Feed user query to **all** Council Member Models\n- Collect the Model responses  from **all** Council Member Models |
| Stage 1: Query Model Answers | HTTP Request | Query OpenRouter for each model’s answer | Split by Model | Stage 1: Extract Responses | ## Stage 1: Individual Model Responses\n- Feed user query to **all** Council Member Models\n- Collect the Model responses  from **all** Council Member Models |
| Stage 1: Extract Responses | Set | Extract `{model, response}` from OpenRouter output | Stage 1: Query Model Answers | Stage 1: Aggregate Responses | ## Stage 1: Individual Model Responses\n- Feed user query to **all** Council Member Models\n- Collect the Model responses  from **all** Council Member Models |
| Stage 1: Aggregate Responses | Aggregate | Combine all model responses into one `data` array | Stage 1: Extract Responses | Stage 2: Create Anonymized Responses | ## Stage 1: Individual Model Responses\n- Feed user query to **all** Council Member Models\n- Collect the Model responses  from **all** Council Member Models |
| Stage 2: Create Anonymized Responses | Set | Label answers A/B/C, build anonymized text block | Stage 1: Aggregate Responses | Stage 2: Build Ranking Prompt | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Stage 2: Build Ranking Prompt | Set | Build strict evaluation + ranking prompt | Stage 2: Create Anonymized Responses | Stage 2: Re-Add councilModels | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Stage 2: Re-Add councilModels | Set | Attach `councilModels` again for split | Stage 2: Build Ranking Prompt | Stage 2: rankingPrompt per LLM | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Stage 2: rankingPrompt per LLM | Split Out | Iterate models for ranking task | Stage 2: Re-Add councilModels | Stage 2: Query Model Answer Rankings | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Stage 2: Query Model Answer Rankings | HTTP Request | Ask each model to evaluate and rank anonymized responses | Stage 2: rankingPrompt per LLM | Stage 2: Extract Rankings | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Stage 2: Extract Rankings | Set | Extract ranking text + parse final ranking labels | Stage 2: Query Model Answer Rankings | Stage 2: Aggregate | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Stage 2: Aggregate | Aggregate | Combine all ranking outputs into one `data` array | Stage 2: Extract Rankings | Stage 3: Calculate Aggregate Rankings | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Stage 3: Calculate Aggregate Rankings | Code | Compute average rank per model from peer rankings | Stage 2: Aggregate | Stage 3: Build ChairmanPrompt Part 1 | ## Stage 3: The Chairman's verdict\n- The Chairman model synthesizes the Council's result |
| Stage 3: Build ChairmanPrompt Part 1 | Set | Build Stage 1 and Stage 2 text blocks | Stage 3: Calculate Aggregate Rankings | Stage 3: Build Chairman Prompt Part 2 | ## Stage 3: The Chairman's verdict\n- The Chairman model synthesizes the Council's result |
| Stage 3: Build Chairman Prompt Part 2 | Set | Build chairmanPrompt + attach chairmanModel | Stage 3: Build ChairmanPrompt Part 1 | Stage 3: Chairman Synthesis | ## Stage 3: The Chairman's verdict\n- The Chairman model synthesizes the Council's result |
| Stage 3: Chairman Synthesis | HTTP Request | Ask chairman model to synthesize final verdict | Stage 3: Build Chairman Prompt Part 2 | Stage 4: Format Output Part 1 | ## Stage 3: The Chairman's verdict\n- The Chairman model synthesizes the Council's result |
| Stage 4: Format Output Part 1 | Set | Extract final answer + collect ranking info | Stage 3: Chairman Synthesis | Stage 4: Format Output Part 2 | ## Output\n- Build a result report (as mail body) with:\n-- the initial query\n-- the Chairman verdict \n-- the Ranking\n-- the individual model answers |
| Stage 4: Format Output Part 2 | Set | Build MailBody report text | Stage 4: Format Output Part 1 | Stage 4: Send Result per Email | ## Output\n- Build a result report (as mail body) with:\n-- the initial query\n-- the Chairman verdict \n-- the Ranking\n-- the individual model answers |
| Stage 4: Send Result per Email | Email Send | Send report via SMTP | Stage 4: Format Output Part 2 | — | ## Output\n- Build a result report (as mail body) with:\n-- the initial query\n-- the Chairman verdict \n-- the Ranking\n-- the individual model answers |
| Sticky Note | Sticky Note | Comment | — | — | ## INPUT and INITIALIZATION\n- Ask the question\n- Initialize the models |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Stage 1: Individual Model Responses\n- Feed user query to **all** Council Member Models\n- Collect the Model responses  from **all** Council Member Models |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Stage 2: Response Evaluation and Ranking\n- **Anonymize model responses**\n- Ask models to evaluate and rank the answers |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Stage 3: The Chairman's verdict\n- The Chairman model synthesizes the Council's result |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Output\n- Build a result report (as mail body) with:\n-- the initial query\n-- the Chairman verdict \n-- the Ranking\n-- the individual model answers |
| Sticky Note5 | Sticky Note | Comment / meta instructions | — | — | ## How it works:\nThis workflow is inspired by a vibe coding project of Andrej Karpathy: https://github.com/karpathy/llm-council .\nThe LLM Council serves as a moderation board to synthesize different LLM responses to the same prompt.\n- Stage 1: The same question / user query is answered by different LLMs, the \"Council Members\".\n- Stage 2: The answers are anonymized and then the same models must judge and rank all responses.\n- Stage 3: All this (the individual answers and the judgments and rankings of the models on their answers) is forwarded to a \"Council Chairman\" who synthesizes an overall answer (the Verdict) out of this input.\n- Besides this \"Chairman Verdict\", all other essential information is integrated in the output:\n\t- The original query\n\t- The Chairman Verdict\n\t- The ranking of each model for all responses\n\t- The original query responses of each model\n\n\n\n## Setup steps:\n1. Set up openRouter Credential:\n**Configure it in each of the HTTP Request nodes of Stage 1, 2 and 3.**\n2. Initialize the model variables in the node of the same name:\n**Use any model from https://openrouter.ai/models.**\n2a) councilModels -> Council Member LLMs:\n**All models that are asked to respond to the input query and subsequently evaluate and rank all responses.**\n2b) chairmanModel -> the Council Chairman LLM:\n**The model to summarize and issue the Council's verdict.**\n3. Define your mail output node:\n**You must have access to an SMTP mailbox and know the mail address you want to send the output to.**\n4. Start the chatTrigger to send your query to the LLM council.\n5. Check the workflow result in the configured mailbox. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it **“LLM Council”**.
2. **Add node: “When chat message received”**
   - Node type: **Chat Trigger** (LangChain).
   - Keep defaults.
3. **Add node: “Initialize Variables” (Set)**
   - Add fields:
     - `userQuery` (String): `{{$json.chatInput}}`
     - `councilModels` (Array): set to your desired OpenRouter model IDs (e.g. `["openai/gpt-4o", ...]`)
     - `chairmanModel` (String): e.g. `google/gemini-2.5-flash`
   - Connect: **Chat Trigger → Initialize Variables**
4. **Add node: “Split by Model” (Split Out)**
   - Field to split out: `councilModels`
   - Include: set to expression `{{$json.userQuery}}` (so query persists per item)
   - Connect: **Initialize Variables → Split by Model**
5. **Add node: “Stage 1: Query Model Answers” (HTTP Request)**
   - Method: POST
   - URL: `https://openrouter.ai/api/v1/chat/completions`
   - Authentication: **Predefined Credential Type → OpenRouter API**
   - Headers: `Content-Type: application/json`
   - Body (RAW JSON) using an expression that stringifies:
     - `model`: current split item (`$json.councilModels`)
     - `messages`: one user message containing `$('Initialize Variables').item.json.userQuery`
   - Timeout: 120000 ms
   - Connect: **Split by Model → Stage 1: Query Model Answers**
6. **Add node: “Stage 1: Extract Responses” (Set)**
   - `model` (String): `{{$json.model}}`
   - `response` (String): `{{$json.choices[0].message.content}}`
   - Connect: **Stage 1 HTTP → Extract Responses**
7. **Add node: “Stage 1: Aggregate Responses” (Aggregate)**
   - Mode: **Aggregate All Item Data**
   - Connect: **Extract Responses → Aggregate Responses**
8. **Add node: “Stage 2: Create Anonymized Responses” (Set)**
   - Create:
     - `labeledResponses`: map the aggregated `data` to labels A/B/C…
     - `anonymizedText`: join as `Response A:\n...`
   - Connect: **Aggregate Responses → Create Anonymized Responses**
   - (Recommended for reliability: ensure `labeledResponses` is stored as a real array, not a JSON string.)
9. **Add node: “Stage 2: Build Ranking Prompt” (Set)**
   - Build a prompt containing:
     - Original question from Initialize Variables
     - The anonymizedText
     - Strict “FINAL RANKING:” formatting requirements
   - Connect: **Create Anonymized Responses → Build Ranking Prompt**
10. **Add node: “Stage 2: Re-Add councilModels” (Set)**
   - `rankingPrompt = {{$json.rankingPrompt}}`
   - `councilModels = {{ $('Initialize Variables').item.json.councilModels }}`
   - Connect: **Build Ranking Prompt → Re-Add councilModels**
11. **Add node: “Stage 2: rankingPrompt per LLM” (Split Out)**
   - Split out field: `councilModels`
   - Include: “all other fields”
   - Connect: **Re-Add councilModels → rankingPrompt per LLM**
12. **Add node: “Stage 2: Query Model Answer Rankings” (HTTP Request)**
   - Same OpenRouter setup as Stage 1 (POST `/chat/completions`)
   - Body uses:
     - `model: $json.councilModels`
     - `messages[0].content: $json.rankingPrompt`
   - Connect: **rankingPrompt per LLM → Query Model Answer Rankings**
13. **Add node: “Stage 2: Extract Rankings” (Set)**
   - `model = {{$json.model}}`
   - `ranking = {{$json.choices[0].message.content}}`
   - `parsedRanking` (Array): split on `FINAL RANKING:` then regex match `Response [A-Z]`
   - Connect: **Rankings HTTP → Extract Rankings**
14. **Add node: “Stage 2: Aggregate” (Aggregate)**
   - Aggregate all item data
   - Connect: **Extract Rankings → Stage 2: Aggregate**
15. **Add node: “Stage 3: Calculate Aggregate Rankings” (Code)**
   - Implement JS to:
     - Map response labels to original models
     - Collect positions from parsedRanking arrays
     - Compute average ranks and sort
   - Connect: **Stage 2: Aggregate → Calculate Aggregate Rankings**
16. **Add node: “Stage 3: Build ChairmanPrompt Part 1” (Set)**
   - Build:
     - `stage1Text` from Stage 1 aggregated responses
     - `stage2Text` from Stage 2 aggregated ranking texts
   - Connect: **Calculate Aggregate Rankings → Build ChairmanPrompt Part 1**
17. **Add node: “Stage 3: Build Chairman Prompt Part 2” (Set)**
   - `chairmanPrompt`: combine question + stage1Text + stage2Text + synthesis instructions
   - `chairmanModel`: from Initialize Variables
   - Connect: **Part 1 → Part 2**
18. **Add node: “Stage 3: Chairman Synthesis” (HTTP Request)**
   - POST OpenRouter `/chat/completions`
   - `model: $json.chairmanModel`
   - `messages[0].content: $json.chairmanPrompt`
   - Connect: **Part 2 → Chairman Synthesis**
19. **Add node: “Stage 4: Format Output Part 1” (Set)**
   - Extract:
     - `finalResponse = {{$json.choices[0].message.content}}`
     - `chairmanModel` from Initialize Variables
     - `aggregateRankings` from Stage 3 code node
   - Connect: **Chairman Synthesis → Format Output Part 1**
20. **Add node: “Stage 4: Format Output Part 2” (Set)**
   - Build `MailBody` text:
     - Question
     - Chairman verdict
     - Aggregate rankings + per-model rankings
     - Individual answers with labels and original model names
   - Connect: **Format Output Part 1 → Part 2**
21. **Add node: “Stage 4: Send Result per Email” (Email Send)**
   - Configure SMTP credentials (host, port, user/pass or app password).
   - Set:
     - To / From addresses
     - Subject with current date
     - Body: `{{$json.MailBody}}`
   - Connect: **Format Output Part 2 → Email Send**
22. **Credentials setup**
   - **OpenRouter credential**: create in n8n credentials (API key), select it in all 3 HTTP Request nodes (Stage 1, Stage 2, Stage 3).
   - **SMTP credential**: configure and test with your mail provider.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Inspired by Andrej Karpathy’s LLM council concept. | https://github.com/karpathy/llm-council |
| Choose models from OpenRouter model list; set them in “Initialize Variables”. | https://openrouter.ai/models |
| Workflow output is emailed; requires working SMTP access and valid from/to addresses. | Setup requirement from workflow notes |
| Process summary: Stage 1 answers → Stage 2 anonymized peer ranking → Stage 3 chairman synthesis → Stage 4 email report. | Workflow design notes (Sticky Note5) |