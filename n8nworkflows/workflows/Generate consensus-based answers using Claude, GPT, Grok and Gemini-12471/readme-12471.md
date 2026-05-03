Generate consensus-based answers using Claude, GPT, Grok and Gemini

https://n8nworkflows.xyz/workflows/generate-consensus-based-answers-using-claude--gpt--grok-and-gemini-12471


# Generate consensus-based answers using Claude, GPT, Grok and Gemini

## 1. Workflow Overview

**Workflow name:** LLM Council  
**Title (provided):** Generate consensus-based answers using Claude, GPT, Grok and Gemini

**Purpose:**  
This workflow takes a single user question (via n8n Chat Trigger), sends it in parallel to multiple LLMs to generate independent answers, anonymizes those answers (A–D), has multiple LLMs *peer-evaluate and rank* the anonymized answers, aggregates the rankings into a consolidated scoring, and finally asks a “Chairman” model to synthesize a single consensus response using both the answers and the peer rankings.

**Target use cases:**
- Higher-quality answers via model diversity and “wisdom of the crowd”
- Reducing single-model hallucinations by cross-checking
- Creating an explainable final answer that reflects both content and peer assessment

### 1.1 Input Reception
- Entry point is a chat message. The user question becomes `chatInput`.

### 1.2 Stage 1 — Parallel Answer Generation (4 models)
- The same question is answered by 4 OpenRouter-backed chat models (Claude, GPT, Grok, Gemini) through LangChain “LLM Chain” nodes.

### 1.3 Stage 2 — Response Storage & Anonymization
- Each model’s answer is stored into standardized fields `text_A` … `text_D` to hide which model produced which answer.

### 1.4 Stage 3 — Peer Evaluation & Ranking (4 evaluators)
- Each evaluator LLM receives all anonymized responses and must output critique + a strictly formatted **FINAL RANKING** section.

### 1.5 Stage 4 — Ranking Aggregation
- Rankings from all evaluators are merged, parsed, and aggregated into average positions to identify the best response.

### 1.6 Stage 5 — Chairman Synthesis (Final Answer)
- A final LLM chain (Chairman) uses the original question, all responses, and all rankings to produce one comprehensive final answer.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Receives the user’s question via n8n chat and fans out to the 4 response-generation chains.  
**Nodes involved:** `When chat message received`

#### Node: When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — workflow entry point for chat-based interactions.
- **Configuration (interpreted):**
  - Uses default trigger options (no special options configured).
  - Exposes the incoming user text as `$('When chat message received').item.json.chatInput`.
- **Outputs / connections:**
  - Sends the same incoming item to:
    - `Basic LLM Chain`
    - `Basic LLM Chain1`
    - `Basic LLM Chain2`
    - `Basic LLM Chain3`
- **Version-specific:** typeVersion `1.4`.
- **Edge cases / failures:**
  - If `chatInput` is empty/null, downstream prompt templates will still render but produce low-quality or error-triggering evaluations (“If there is an error with the data…”).
  - Very long inputs may cause model context overflow depending on model limits.

---

### Block 2 — Stage 1: Parallel LLM Responses (Answer Generation)
**Overview:** Four different LLMs produce independent answers in parallel to the same question, each constrained to answer in English.  
**Nodes involved:** `Basic LLM Chain`, `Basic LLM Chain1`, `Basic LLM Chain2`, `Basic LLM Chain3`, plus their attached OpenRouter model nodes: `claude3`, `openAI2`, `groq1`, `gemini1`.

#### Node: Basic LLM Chain (Response A generator)
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — runs a simple LLM chain and outputs `text`.
- **Configuration:**
  - Adds an instruction message: “Give an answer in English”.
  - Uses an attached language model via **AI connection**.
- **Inputs:** item from `When chat message received`.
- **Outputs:** to `response a`.
- **Model attached:** `claude3` via `ai_languageModel`.
- **Version:** `1.7`.
- **Edge cases:**
  - If the attached model fails (auth/rate limit), chain produces an error and downstream stages won’t have `text_A`.

#### Node: claude3
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — OpenRouter chat model.
- **Configuration:**
  - Model: `anthropic/claude-sonnet-4.5`
  - `maxTokens: 4000`, `temperature: 0.7`
  - Credential: **OpenRouter account**
- **Connection:** Provides the AI model to `Basic LLM Chain`.
- **Version:** `1`.
- **Edge cases:**
  - OpenRouter 401/403 (bad key), 429 (rate limiting), 5xx, or model unavailability.

#### Node: Basic LLM Chain1 (Response B generator)
- **Type / role:** LangChain LLM Chain for a second model.
- **Configuration:** Same “Give an answer in English”.
- **Output:** `response b`
- **Model attached:** `openAI2`
- **Version:** `1.7`

#### Node: openAI2
- **Type / role:** OpenRouter chat model node.
- **Configuration:**
  - Model: `openai/gpt-5.1`
  - `maxTokens: 4000`, `temperature: 0.7`
  - Credential: OpenRouter account
- **Version:** `1`

#### Node: Basic LLM Chain2 (Response C generator)
- **Output:** `response c`
- **Model attached:** `groq1`

#### Node: groq1
- **Type / role:** OpenRouter chat model node.
- **Configuration:**
  - Model: `x-ai/grok-4`
  - `maxTokens: 4000`, `temperature: 0.7`
  - Credential: OpenRouter account
- **Version:** `1`

#### Node: Basic LLM Chain3 (Response D generator)
- **Output:** `response d`
- **Model attached:** `gemini1`

#### Node: gemini1
- **Type / role:** OpenRouter chat model node.
- **Configuration:**
  - Model: `google/gemini-3-flash-preview`
  - `maxTokens: 4000`, `temperature: 0.7`
  - Credential: OpenRouter account
- **Version:** `1`

---

### Block 3 — Stage 2: Response Storage & Anonymization
**Overview:** Each generated response is copied into standardized fields (`text_A`…`text_D`) so evaluators see anonymized answers only.  
**Nodes involved:** `response a`, `response b`, `response c`, `response d`

#### Node: response a
- **Type / role:** `n8n-nodes-base.set` — assigns anonymized response field.
- **Configuration:**
  - Sets `text_A` = `{{$json.text}}` (the chain output).
- **Inputs:** from `Basic LLM Chain`.
- **Outputs:** fans out to 4 merge nodes:
  - `Merge3` (index 0)
  - `Merge1` (index 0)
  - `Merge` (index 0)
  - `Merge5` (index 0)
- **Version:** `3.4`
- **Edge cases:**
  - If upstream chain output doesn’t contain `text`, `text_A` becomes empty and evaluators may trigger their “error with the data” instruction.

(Identical pattern for the other Set nodes:)

#### Node: response b
- Sets `text_B` = `{{$json.text}}`
- Outputs to `Merge3` (1), `Merge5` (1), `Merge1` (1), `Merge` (1)

#### Node: response c
- Sets `text_C` = `{{$json.text}}`
- Outputs to `Merge3` (2), `Merge5` (2), `Merge1` (2), `Merge` (2)

#### Node: response d
- Sets `text_D` = `{{$json.text}}`
- Outputs to `Merge3` (3), `Merge` (3), `Merge1` (3), `Merge5` (3)

---

### Block 4 — Stage 3: Peer Evaluation & Ranking (4 evaluators)
**Overview:** Each evaluator LLM receives all four anonymized responses plus the original question, critiques them, and must produce a strictly formatted `FINAL RANKING:` section to enable deterministic parsing later.  
**Nodes involved:** `Merge3`, `Merge5`, `Merge1`, `Merge`, evaluator chains (`evaluate claude`, `evaluate gpt`, `evaluate grok`, `evaluate gemini`), evaluator model nodes (`claude`, `openAI`, `groq`, `gemini2`), and storage nodes (`evaluate_a`, `evaluate_b`, `evaluate_c`, `evaluate_d`).

#### Node: Merge3
- **Type / role:** `n8n-nodes-base.merge` — combines 4 inputs into one item.
- **Configuration:**
  - Mode: **combine**
  - Combine by: **position** (`combineByPosition`)
  - `numberInputs: 4`
- **Inputs:** responses A–D (from `response a/b/c/d`)
- **Output:** to `evaluate claude`
- **Version:** `3.2`
- **Edge cases:**
  - If any response path produces 0 items or errors, combine-by-position can misalign or output incomplete merged data.

#### Node: evaluate claude
- **Type / role:** LangChain `chainLlm` — produces evaluation text and strict ranking.
- **Configuration highlights:**
  - Prompt injects:
    - Question: `{{ $('When chat message received').item.json.chatInput }}`
    - Responses: `{{ $json.text_A }}`, `{{ $json.text_B }}`, `{{ $json.text_C }}`, `{{ $json.text_D }}`
  - Enforces ranking format:
    - Must include line `FINAL RANKING:`
    - Must list `1. Response X` lines only.
  - Instruction: “give answer on english”
- **Model attached:** `claude` via `ai_languageModel`.
- **Output:** to `evaluate_a` (Set node).
- **Version:** `1.7`
- **Failure modes:**
  - Model may not follow strict formatting → parsing/aggregation fails later.
  - If any response field missing/empty, prompt asks to “report it immediately and do not generate any response” (but models may still produce ranking).

#### Node: claude (evaluator model)
- **Type / role:** OpenRouter chat model node used for evaluation.
- **Configuration:**
  - Model: `anthropic/claude-sonnet-4.5`
  - `maxTokens: 2000`, `temperature: 0.3` (lower temperature for consistency)
- **Version:** `1`

#### Node: evaluate_a
- **Type / role:** Set node to store evaluator output.
- **Configuration:** sets `Raiting_1` = `{{$json.text}}`
- **Output:** to `Merge4` as input index 0
- **Edge case:** Field name is misspelled “Raiting” (not “Rating”). Downstream code relies on the misspelled keys, so renaming will break aggregation unless code is updated.

---

#### Node: Merge5 → evaluate gpt → evaluate_b
- **Merge5:** combine-by-position of responses A–D → `evaluate gpt`
- **evaluate gpt:** same evaluation prompt and formatting rules
- **Model attached:** `openAI` (OpenRouter `openai/gpt-5.1`, `maxTokens: 2000`, `temperature: 0.3`)
- **evaluate_b:** stores `Raiting_2` = `text`, outputs to `Merge4` index 1

#### Node: Merge1 → evaluate grok → evaluate_c
- **Model attached:** `groq` (OpenRouter `x-ai/grok-4`, `maxTokens: 2000`, `temperature: 0.3`)
- **evaluate_c:** stores `Raiting_3`, outputs to `Merge4` index 2

#### Node: Merge → evaluate gemini → evaluate_d
- **Model attached:** `gemini2` (OpenRouter `google/gemini-3-flash-preview`, `maxTokens: 2000`, `temperature: 0.3`)
- **evaluate_d:** stores `Raiting_4`, outputs to `Merge4` index 3

---

### Block 5 — Stage 4: Ranking Aggregation (Merge + Code)
**Overview:** Collects all evaluator rankings into one item, parses the “FINAL RANKING” sections, aggregates average positions for each response, and outputs a best-response summary.  
**Nodes involved:** `Merge4`, `Code in JavaScript`

#### Node: Merge4
- **Type / role:** Merge (combine 4 evaluator outputs).
- **Configuration:** combine-by-position, `numberInputs: 4`.
- **Inputs:** `evaluate_a`, `evaluate_b`, `evaluate_c`, `evaluate_d`
- **Output:** `Code in JavaScript`
- **Version:** `3.2`
- **Edge cases:**
  - If any evaluator chain failed, Merge4 may not receive all inputs; the Code node will then have missing `Raiting_*` fields.

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — parses rankings and computes aggregate statistics.
- **Key logic (interpreted):**
  - Reads a single merged item: `const itemData = items[0].json;`
  - For each key `Raiting_1..Raiting_4`, parses only lines after “FINAL RANKING”.
  - Regex expects: `(\d+)\.\s*(Response [A-D])`
  - Aggregates positions per response; computes average rank position (lower is better).
  - Sorts by average ascending, outputs:
    - `raw_rankings` (parsed maps)
    - `aggregated_rankings` (sorted summary)
    - `best_response` (e.g., “Response B”)
    - `best_score` (average position)
    - `rankings_count`
- **Output:** to `Basic LLM Chain8`
- **Version:** `2`
- **Edge cases / failure types:**
  - **Formatting drift:** If an evaluator outputs `Final ranking` (wrong case) or adds extra text in ranking lines, parsing may return empty maps.
  - **Model label mismatch:** Regex requires “Response A/B/C/D” exactly.
  - If no rankings parse correctly, `sorted[Object.keys(sorted)[0]].average` can be `undefined`, risking runtime issues or misleading output.
  - If some responses never appear in parsed rankings, `average` remains undefined; sorting with undefined values can yield unstable results.

---

### Block 6 — Stage 5: Chairman Synthesis (Final consensus answer)
**Overview:** A final LLM (“Chairman”) receives the original question, the four individual responses, and all peer rankings, and synthesizes one final answer representing collective consensus.  
**Nodes involved:** `Basic LLM Chain8`, `claude4`

#### Node: Basic LLM Chain8
- **Type / role:** LangChain LLM chain with a long, explicitly defined prompt (“Chairman” role).
- **Configuration highlights:**
  - Prompt injects:
    - Original question from `When chat message received` (`chatInput`)
    - Responses from set nodes:
      - `$('response a').item.json.text_A`, etc.
    - Rankings from set nodes:
      - `$('evaluate_a').item.json.Raiting_1`, etc.
  - Note: It does **not** directly reference the aggregated output from the Code node in the prompt, even though it runs after it. Aggregation is computed but not used in the Chairman text unless you extend the prompt.
- **Inputs:** from `Code in JavaScript` (execution sequencing).
- **Model attached:** `claude4`
- **Version:** `1.7`
- **Edge cases:**
  - If any upstream node didn’t run or produced no item, expressions like `$('response a').item.json.text_A` can fail (“Referenced node did not return data”) depending on n8n expression evaluation and execution path.
  - Prompt contains “Model: anthropic/claud” (typo: “claud”)—harmless, but confusing.

#### Node: claude4 (Chairman model)
- **Type / role:** OpenRouter chat model.
- **Configuration:**
  - Model: `google/gemini-3-pro-preview` (despite name “claude4”)
  - `temperature: 0.6`
  - Credential: OpenRouter account
- **Version:** `1`
- **Edge cases:**
  - Model availability/preview instability; output variability with temperature 0.6.

---

### Block 7 — Unconnected “Possible options” nodes (not part of execution path)
**Overview:** The workflow includes additional nodes suggesting alternative input methods (WhatsApp/Telegram/Gmail/Slack) and alternative direct LLM nodes (Anthropic/OpenAI/Gemini/xAI native nodes). They are not connected, so they do not execute.  
**Nodes involved:** `Anthropic Chat Model`, `OpenAI Chat Model`, `Google Gemini Chat Model`, `xAI Grok Chat Model`, `Send message`, `Send a text message`, `Send a message1`, `Send a message`

#### LLM nodes (native providers, unconnected)
- `Anthropic Chat Model` (`lmChatAnthropic`, Anthropic credential “Test Claud”)
- `OpenAI Chat Model` (`lmChatOpenAi`, OpenAI credential “Test”)
- `Google Gemini Chat Model` (`lmChatGoogleGemini`, Google Palm credential “Test”)
- `xAI Grok Chat Model` (`lmChatXAiGrok`, xAI credential “Test Grok”)

**Potential issues if you connect them later:**
- Different parameter naming and response formats vs OpenRouter nodes.
- Different model IDs and token limits.
- Credential scopes and billing differences per provider.

#### Messaging nodes (unconnected)
- `Send message` (WhatsApp node)
- `Send a text message` (Telegram node with credential “Test”)
- `Send a message1` (Gmail node with credential “Test GMail”)
- `Send a message` (Slack node; no credential shown in JSON snippet)

**Potential issues if connected later:**
- Need to map the final Chairman output into the appropriate message field.
- Provider-specific rate limits and formatting constraints.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point (chat input) | — | Basic LLM Chain; Basic LLM Chain1; Basic LLM Chain2; Basic LLM Chain3 |  |
| Basic LLM Chain | @n8n/n8n-nodes-langchain.chainLlm | Generate Response A | When chat message received | response a | ## Stage 1\nGet answers from different models |
| claude3 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for Response A | — (AI-only) | Basic LLM Chain (AI) | ## Stage 1\nGet answers from different models |
| Basic LLM Chain1 | @n8n/n8n-nodes-langchain.chainLlm | Generate Response B | When chat message received | response b | ## Stage 1\nGet answers from different models |
| openAI2 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for Response B | — (AI-only) | Basic LLM Chain1 (AI) | ## Stage 1\nGet answers from different models |
| Basic LLM Chain2 | @n8n/n8n-nodes-langchain.chainLlm | Generate Response C | When chat message received | response c | ## Stage 1\nGet answers from different models |
| groq1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for Response C | — (AI-only) | Basic LLM Chain2 (AI) | ## Stage 1\nGet answers from different models |
| Basic LLM Chain3 | @n8n/n8n-nodes-langchain.chainLlm | Generate Response D | When chat message received | response d | ## Stage 1\nGet answers from different models |
| gemini1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for Response D | — (AI-only) | Basic LLM Chain3 (AI) | ## Stage 1\nGet answers from different models |
| response a | n8n-nodes-base.set | Store/anonymize Response A (`text_A`) | Basic LLM Chain | Merge3; Merge1; Merge; Merge5 | ## Stage 2\nFor convenience and impartiality, we store and anonymize responses. |
| response b | n8n-nodes-base.set | Store/anonymize Response B (`text_B`) | Basic LLM Chain1 | Merge3; Merge5; Merge1; Merge | ## Stage 2\nFor convenience and impartiality, we store and anonymize responses. |
| response c | n8n-nodes-base.set | Store/anonymize Response C (`text_C`) | Basic LLM Chain2 | Merge3; Merge5; Merge1; Merge | ## Stage 2\nFor convenience and impartiality, we store and anonymize responses. |
| response d | n8n-nodes-base.set | Store/anonymize Response D (`text_D`) | Basic LLM Chain3 | Merge3; Merge; Merge1; Merge5 | ## Stage 2\nFor convenience and impartiality, we store and anonymize responses. |
| Merge3 | n8n-nodes-base.merge | Combine responses for Claude evaluation | response a; response b; response c; response d | evaluate claude | ## For convenience and impartiality, we store and anonymize responses. |
| Merge5 | n8n-nodes-base.merge | Combine responses for GPT evaluation | response a; response b; response c; response d | evaluate gpt | ## For convenience and impartiality, we store and anonymize responses. |
| Merge1 | n8n-nodes-base.merge | Combine responses for Grok evaluation | response a; response b; response c; response d | evaluate grok | ## For convenience and impartiality, we store and anonymize responses. |
| Merge | n8n-nodes-base.merge | Combine responses for Gemini evaluation | response a; response b; response c; response d | evaluate gemini | ## For convenience and impartiality, we store and anonymize responses. |
| evaluate claude | @n8n/n8n-nodes-langchain.chainLlm | Evaluate + rank responses (Claude) | Merge3 | evaluate_a | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| claude | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for Claude evaluation | — (AI-only) | evaluate claude (AI) | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| evaluate gpt | @n8n/n8n-nodes-langchain.chainLlm | Evaluate + rank responses (GPT) | Merge5 | evaluate_b | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| openAI | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for GPT evaluation | — (AI-only) | evaluate gpt (AI) | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| evaluate grok | @n8n/n8n-nodes-langchain.chainLlm | Evaluate + rank responses (Grok) | Merge1 | evaluate_c | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| groq | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for Grok evaluation | — (AI-only) | evaluate grok (AI) | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| evaluate gemini | @n8n/n8n-nodes-langchain.chainLlm | Evaluate + rank responses (Gemini) | Merge | evaluate_d | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| gemini2 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for Gemini evaluation | — (AI-only) | evaluate gemini (AI) | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| evaluate_a | n8n-nodes-base.set | Store evaluator output `Raiting_1` | evaluate claude | Merge4 | ## we get an analysis of all the answers |
| evaluate_b | n8n-nodes-base.set | Store evaluator output `Raiting_2` | evaluate gpt | Merge4 | ## we get an analysis of all the answers |
| evaluate_c | n8n-nodes-base.set | Store evaluator output `Raiting_3` | evaluate grok | Merge4 | ## we get an analysis of all the answers |
| evaluate_d | n8n-nodes-base.set | Store evaluator output `Raiting_4` | evaluate gemini | Merge4 | ## we get an analysis of all the answers |
| Merge4 | n8n-nodes-base.merge | Combine all rankings for aggregation | evaluate_a; evaluate_b; evaluate_c; evaluate_d | Code in JavaScript | ## Stage 4\nThis is where the magic happens and all the scores are ranked. |
| Code in JavaScript | n8n-nodes-base.code | Parse rankings + compute averages/best | Merge4 | Basic LLM Chain8 | ## Stage 4\nThis is where the magic happens and all the scores are ranked. |
| Basic LLM Chain8 | @n8n/n8n-nodes-langchain.chainLlm | Chairman synthesis (final answer) | Code in JavaScript | (end) | ## Stage 5\nAll responses are evaluated by one model and a generalized and summarized response is provided. |
| claude4 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Chairman model (actually Gemini Pro Preview) | — (AI-only) | Basic LLM Chain8 (AI) | ## Stage 5\nAll responses are evaluated by one model and a generalized and summarized response is provided. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Unconnected alternative model option | — | — | ## possible options for data analysis directly |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Unconnected alternative model option | — | — | ## possible options for data analysis directly |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Unconnected alternative model option | — | — | ## possible options for data analysis directly |
| xAI Grok Chat Model | @n8n/n8n-nodes-langchain.lmChatXAiGrok | Unconnected alternative model option | — | — | ## possible options for data analysis directly |
| Send a message1 | n8n-nodes-base.gmail | Unconnected alternative output channel | — | — | ## possible data entry options |
| Send a message | n8n-nodes-base.slack | Unconnected alternative output channel | — | — | ## possible data entry options |
| Send message | n8n-nodes-base.whatsApp | Unconnected alternative output channel | — | — | ## possible data entry options |
| Send a text message | n8n-nodes-base.telegram | Unconnected alternative output channel | — | — | ## possible data entry options |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | ## possible data entry options |
| Sticky Note9 | n8n-nodes-base.stickyNote | Comment | — | — | ## possible options for data analysis directly |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | ## Stage 1\nGet answers from different models |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | ## Stage 2\nFor convenience and impartiality, we store and anonymize responses. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — | ## For convenience and impartiality, we store and anonymize responses. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment | — | — | ## Stage 3\nFor convenience and impartiality, we store and anonymize responses. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment | — | — | ## we get an analysis of all the answers |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment | — | — | ## Stage 4\nThis is where the magic happens and all the scores are ranked. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Comment | — | — | ## Stage 5\nAll responses are evaluated by one model and a generalized and summarized response is provided. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Comment | — | — | ## How It Works\n\n1️⃣ User Input\nA user sends a single question via chat. This message becomes the shared input for the entire workflow.\n\n2️⃣ Parallel LLM Responses\nThe same question is sent simultaneously to multiple large language models.\nEach model generates its own answer independently, using the same prompt but its own reasoning.\n\n3️⃣ Response Anonymization\nAll model outputs are stored as anonymous responses (Response A–D).\nThis prevents evaluator models from knowing which LLM produced which answer.\n\n4️⃣ Peer Evaluation & Ranking\nEach LLM reviews all anonymized responses, analyzes their strengths and weaknesses, and produces a strict ranking from best to worst.\n\n5️⃣ Ranking Aggregation\nAll rankings are collected and aggregated.\nThe workflow calculates average positions and identifies the strongest response based on collective evaluation.\n\n6️⃣ Final Consensus Answer\nA dedicated “Chairman” model reviews all responses and rankings, detects agreement and disagreement patterns, and synthesizes one clear, high-quality final answer.\n\n➡️ Result: a consensus-driven response built from multiple independent LLM perspectives, not a single-model output. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | ## Setup Steps\n1️⃣ Import the Workflow\nImport the provided n8n workflow template into your instance.\n\n2️⃣ Configure OpenRouter Credentials\nAdd your OpenRouter API key in n8n credentials and connect it to all LLM nodes in the workflow.\n\n3️⃣ Select LLM Models\nChoose which models will participate in the council (e.g. GPT, Claude, Gemini, Grok).\nYou can add or remove models without changing the core logic.\n\n4️⃣ Review Prompt Nodes\nCheck the system and evaluation prompts.\nAdjust tone, language, or strictness if your use case requires it.\n\n5️⃣ Verify the Aggregation Code Node\nEnsure the JavaScript aggregation node matches the number of active LLMs and expected ranking format.\n\n6️⃣ Test with a Sample Question\nRun the workflow with a simple input to confirm that:\n\nAll models respond\n\nRankings are generated\n\nA final consensus answer is produced\n\n7️⃣ Activate the Workflow\nOnce verified, activate the workflow and use it in production or connect it to other automations. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: **LLM Council**
- (Optional) Tag: **Technical**

2) **Add trigger**
- Add node: **Chat Trigger** (`When chat message received`)
- Keep default options.

3) **Create Stage 1: 4 parallel answer chains**
For each of the four branches, do:

3.1 **Add a LangChain “LLM Chain” node**
- Node type: **LLM Chain** (`chainLlm`)
- Message/instruction: `Give an answer in English`
- Connect `When chat message received` → this chain.

3.2 **Add an OpenRouter Chat Model node and attach**
- Node type: **OpenRouter Chat Model** (`lmChatOpenRouter`)
- Create credential: **OpenRouter API** (store your API key)
- Select model + options:
  - Branch A model: `anthropic/claude-sonnet-4.5`, maxTokens 4000, temperature 0.7
  - Branch B model: `openai/gpt-5.1`, maxTokens 4000, temperature 0.7
  - Branch C model: `x-ai/grok-4`, maxTokens 4000, temperature 0.7
  - Branch D model: `google/gemini-3-flash-preview`, maxTokens 4000, temperature 0.7
- Connect model node to the chain using the **AI Language Model** connector.

4) **Create Stage 2: store/anonymize responses**
For each branch output, add a **Set** node:
- `response a`: set field `text_A` = `{{$json.text}}`
- `response b`: set field `text_B` = `{{$json.text}}`
- `response c`: set field `text_C` = `{{$json.text}}`
- `response d`: set field `text_D` = `{{$json.text}}`
Connect each chain → corresponding Set node.

5) **Create Stage 3: 4 evaluation merges + evaluator chains**
You will create 4 separate **Merge (combine-by-position, 4 inputs)** nodes, each feeding one evaluator chain.

5.1 **Create merge nodes**
- `Merge3`, `Merge5`, `Merge1`, `Merge`
- Each:
  - Mode: **Combine**
  - Combine by: **Position**
  - Number of inputs: **4**
Connect:
- `response a/b/c/d` to each merge node inputs 0/1/2/3 respectively.

5.2 **Create evaluator LLM Chains**
Create four `chainLlm` nodes:
- `evaluate claude`
- `evaluate gpt`
- `evaluate grok`
- `evaluate gemini`

Each uses the same evaluation prompt structure:
- Inject original question: `{{ $('When chat message received').item.json.chatInput }}`
- Inject anonymized responses: `{{ $json.text_A }}` etc.
- Enforce strict format section:
  - `FINAL RANKING:`
  - `1. Response X` … `4. Response Y`

Connect:
- `Merge3` → `evaluate claude`
- `Merge5` → `evaluate gpt`
- `Merge1` → `evaluate grok`
- `Merge` → `evaluate gemini`

5.3 **Attach evaluator OpenRouter model nodes**
Add 4 OpenRouter model nodes (or reuse, but keep settings consistent):
- For evaluation: maxTokens **2000**, temperature **0.3**
- Models:
  - Claude evaluator: `anthropic/claude-sonnet-4.5`
  - GPT evaluator: `openai/gpt-5.1`
  - Grok evaluator: `x-ai/grok-4`
  - Gemini evaluator: `google/gemini-3-flash-preview`
Attach each to its evaluator chain via the **AI** connector.

5.4 **Store evaluator outputs**
Add 4 Set nodes:
- `evaluate_a`: set `Raiting_1` = `{{$json.text}}`
- `evaluate_b`: set `Raiting_2` = `{{$json.text}}`
- `evaluate_c`: set `Raiting_3` = `{{$json.text}}`
- `evaluate_d`: set `Raiting_4` = `{{$json.text}}`
Connect each evaluator chain → corresponding Set node.

6) **Create Stage 4: merge rankings + aggregate in code**
6.1 Add `Merge4`
- Merge mode: Combine
- Combine by position
- Inputs: 4
Connect:
- `evaluate_a` → Merge4 input 0
- `evaluate_b` → Merge4 input 1
- `evaluate_c` → Merge4 input 2
- `evaluate_d` → Merge4 input 3

6.2 Add `Code in JavaScript`
- Paste logic that:
  - Reads `items[0].json`
  - Parses each `Raiting_1..4` after “FINAL RANKING”
  - Regex matches `1. Response A` etc.
  - Computes averages and best response
Connect `Merge4` → `Code in JavaScript`.

7) **Create Stage 5: Chairman synthesis**
7.1 Add `Basic LLM Chain8` (LLM Chain)
- PromptType: “Define” (custom text)
- Prompt includes:
  - Original question via `$('When chat message received').item.json.chatInput`
  - Responses via `$('response a').item.json.text_A` etc.
  - Rankings via `$('evaluate_a').item.json.Raiting_1` etc.
Connect `Code in JavaScript` → `Basic LLM Chain8`.

7.2 Add Chairman OpenRouter model node
- Node: `lmChatOpenRouter`
- Model: `google/gemini-3-pro-preview`
- Temperature: `0.6`
- Attach to `Basic LLM Chain8` via AI connector.

8) **(Optional) Add output channels**
- If you want to send the Chairman result to Slack/Telegram/Gmail/WhatsApp:
  - Connect from `Basic LLM Chain8` to the messaging node
  - Map the outgoing message text from `{{$json.text}}` (typical LLM Chain output)

9) **Credentials setup**
- Required for core workflow: **OpenRouter API credential** and applied to all `lmChatOpenRouter` nodes.
- Optional (only if you connect them): Gmail OAuth2, Telegram bot token, Slack OAuth/token, WhatsApp provider credentials.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer (provided) |
| ## How It Works … (User Input → Parallel LLM Responses → Anonymization → Peer Evaluation → Ranking Aggregation → Final Consensus Answer) | Sticky Note “How It Works” content (in-workflow documentation) |
| ## Setup Steps … (Import → Configure OpenRouter → Select Models → Review Prompts → Verify Code Node → Test → Activate) | Sticky Note “Setup Steps” content (in-workflow checklist) |
| “Raiting_1..4” is intentionally misspelled and referenced in the Code node | Important for maintenance: keep naming consistent unless you update code + prompts |

