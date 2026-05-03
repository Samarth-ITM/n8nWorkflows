Combine answers from OpenAI, Anthropic, Gemini and Groq into one consensus

https://n8nworkflows.xyz/workflows/combine-answers-from-openai--anthropic--gemini-and-groq-into-one-consensus-14497


# Combine answers from OpenAI, Anthropic, Gemini and Groq into one consensus

# 1. Workflow Overview

This workflow implements a multi-model AI consensus engine in n8n. A user asks a question through the n8n chat interface, and the workflow sends the same prompt to four different LLM providers in parallel: Google Gemini, Anthropic, OpenAI, and Groq. Each model is instructed to return a JSON object containing an answer, a confidence score, and brief reasoning.

The workflow then parses and validates the model outputs, measures how similar the answers are, calibrates confidence scores based on agreement patterns, and chooses between two response modes:

- **Weighted consensus mode:** when the answers are sufficiently aligned
- **Peer review fallback mode:** when the answers diverge too strongly

Finally, it formats the result into a human-readable chat response and sends it back to the user.

## 1.1 Input Reception and Prompt Preparation

The workflow starts from a public chat trigger. It extracts the incoming chat text and stores it as a normalized `userPrompt` field for reuse across all downstream LLM branches.

## 1.2 Parallel LLM Generation

The same prompt is routed to four agent nodes, each backed by a different chat model provider. Each agent must return JSON with:

- `answer`
- `confidence`
- `reasoning`

This block isolates model generation so each provider answers independently.

## 1.3 Response Aggregation and Validation

The four model outputs are merged, then parsed by a Code node. This block validates structure, strips optional Markdown code fences, clamps confidence values into the `[0,1]` range, and records failures as invalid parse results.

## 1.4 Similarity and Calibration Engine

The workflow computes pairwise similarity between valid responses using a combined Jaccard/Cosine/first-sentence metric. Based on those agreement scores, it recalibrates confidence values to penalize overconfident outliers and boost underconfident responses that align with consensus.

## 1.5 Consensus Decision

An If node checks whether the average similarity indicates extreme divergence. If yes, the workflow switches to peer review fallback mode. Otherwise, it produces a weighted consensus answer based on calibrated confidence.

## 1.6 Final Formatting and Chat Delivery

The selected output is transformed into a structured JSON report, then into a readable chat message. The final message is returned to the user through the chat response node.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Prompt Preparation

### Overview
This block receives the user’s question from the n8n chat interface and prepares a single normalized prompt field for the four parallel LLM branches. It is the workflow’s only entry point.

### Nodes Involved
- When chat message received
- Set User Prompt

### Node Details

#### 1. When chat message received
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`; public chat entry point
- **Configuration choices:**
  - Public chat endpoint enabled
  - Response mode set to **responseNodes**
  - Initial welcome message configured as:
    - “Hi there! 👋 Ask me any question and I'll analyze it using 4 AI models!”
- **Key expressions or variables used:**
  - Produces `chatInput`, later used downstream
- **Input and output connections:**
  - Entry node, no input
  - Output → `Set User Prompt`
- **Version-specific requirements:**
  - Type version `1.4`
  - Requires n8n chat functionality compatible with LangChain chat trigger nodes
- **Edge cases or potential failure types:**
  - Chat UI unavailable if workflow is inactive
  - Public exposure may invite abusive or excessive usage
  - If response nodes fail downstream, the chat session may receive no meaningful answer
- **Sub-workflow reference:** None

#### 2. Set User Prompt
- **Type and technical role:** `n8n-nodes-base.set`; normalizes incoming input into a dedicated field
- **Configuration choices:**
  - Creates one string field: `userPrompt`
  - Value is taken from the incoming chat payload
- **Key expressions or variables used:**
  - `={{ $json.chatInput }}`
- **Input and output connections:**
  - Input ← `When chat message received`
  - Output → `LLM1 `, `LLM2 `, `LLM3`, `LLM4 `
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - Empty `chatInput` produces an empty question
  - If the trigger payload format changes, the expression may fail or return undefined
- **Sub-workflow reference:** None

---

## Block 2 — Parallel LLM Generation

### Overview
This block sends the same prepared user prompt to four AI agent nodes in parallel. Each agent uses a different model provider and is instructed to return only valid JSON containing an answer and self-reported confidence.

### Nodes Involved
- Google Gemini Chat Model
- LLM1 
- Anthropic Chat Model
- LLM2 
- OpenAI Chat Model
- LLM3
- Groq Chat Model3
- LLM4 

### Node Details

#### 3. Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; provides the Gemini language model backend
- **Configuration choices:**
  - Default options left empty
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model output → `LLM1 `
- **Version-specific requirements:**
  - Type version `1`
  - Requires valid Google Gemini / PaLM API credentials
- **Edge cases or potential failure types:**
  - Invalid API key or revoked credential
  - Provider quota exhaustion
  - Model-side safety refusals
  - Slow responses or provider outages
- **Sub-workflow reference:** None

#### 4. LLM1 
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; prompts Gemini to answer and self-score confidence
- **Configuration choices:**
  - Prompt type: define
  - User text asks model to:
    - answer the question
    - rate confidence as float between `0.0` and `1.0`
    - return only valid JSON
  - System message: force valid JSON output
  - `onError` = `continueRegularOutput`
- **Key expressions or variables used:**
  - `{{ $json.userPrompt }}`
- **Input and output connections:**
  - Main input ← `Set User Prompt`
  - AI language model input ← `Google Gemini Chat Model`
  - Main output → `Merge All 4 LLMs` (input 0)
- **Version-specific requirements:**
  - Type version `3.1`
- **Edge cases or potential failure types:**
  - Model may still return invalid JSON despite instructions
  - Empty answers or omitted fields
  - Confidence not numeric
  - Because `continueRegularOutput` is enabled, failures may propagate as incomplete outputs instead of halting the workflow
- **Sub-workflow reference:** None

#### 5. Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; provides Claude model backend
- **Configuration choices:**
  - Model selected: `claude-sonnet-4-5-20250929`
  - Default options otherwise
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model output → `LLM2 `
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires Anthropic API credentials
  - The exact model name may depend on account access and current Anthropic catalog
- **Edge cases or potential failure types:**
  - Selected model may be unavailable in another environment
  - Credential issues, rate limiting, or account restrictions
- **Sub-workflow reference:** None

#### 6. LLM2 
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; prompts Anthropic to produce a structured answer
- **Configuration choices:**
  - Same prompt template and system message pattern as LLM1
  - `onError` = `continueRegularOutput`
- **Key expressions or variables used:**
  - `{{ $json.userPrompt }}`
- **Input and output connections:**
  - Main input ← `Set User Prompt`
  - AI language model input ← `Anthropic Chat Model`
  - Main output → `Merge All 4 LLMs` (input 1)
- **Version-specific requirements:**
  - Type version `3.1`
- **Edge cases or potential failure types:**
  - Same JSON-formatting and provider failure issues as other agent nodes
- **Sub-workflow reference:** None

#### 7. OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides OpenAI chat model backend
- **Configuration choices:**
  - Model selected: `gpt-5-mini`
  - Options left empty
  - Built-in tools empty
  - `retryOnFail` disabled
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model output → `LLM3`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI API credentials
  - Availability of `gpt-5-mini` depends on account and n8n/OpenAI integration compatibility
- **Edge cases or potential failure types:**
  - Model may not exist in another workspace or future environment
  - Rate limits, timeout, malformed provider response
  - No automatic retry because `retryOnFail` is false
- **Sub-workflow reference:** None

#### 8. LLM3
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; prompts OpenAI to produce answer JSON
- **Configuration choices:**
  - Same structured prompt pattern as the other agent nodes
  - No explicit `onError` flag shown, so standard failure behavior applies
- **Key expressions or variables used:**
  - `{{ $json.userPrompt }}`
- **Input and output connections:**
  - Main input ← `Set User Prompt`
  - AI language model input ← `OpenAI Chat Model`
  - Main output → `Merge All 4 LLMs` (input 2)
- **Version-specific requirements:**
  - Type version `3.1`
- **Edge cases or potential failure types:**
  - Unlike some sibling nodes, this agent may halt on failure depending on runtime behavior
  - Invalid JSON output remains possible
- **Sub-workflow reference:** None

#### 9. Groq Chat Model3
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; provides Groq-hosted model backend
- **Configuration choices:**
  - Model selected: `llama-3.3-70b-versatile`
  - Default options otherwise
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model output → `LLM4 `
- **Version-specific requirements:**
  - Type version `1`
  - Requires Groq API credentials
- **Edge cases or potential failure types:**
  - Invalid credential, quota/rate issues, model unavailability
- **Sub-workflow reference:** None

#### 10. LLM4 
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; prompts Groq-backed model to return answer JSON
- **Configuration choices:**
  - Same prompt and JSON-only instruction pattern as the other LLM agents
  - No explicit `onError` in parameters, but the node itself is shown without halt override in JSON
- **Key expressions or variables used:**
  - `{{ $json.userPrompt }}`
- **Input and output connections:**
  - Main input ← `Set User Prompt`
  - AI language model input ← `Groq Chat Model3`
  - Main output → `Merge All 4 LLMs` (input 3)
- **Version-specific requirements:**
  - Type version `3.1`
- **Edge cases or potential failure types:**
  - Invalid JSON, provider-side failure, malformed structure
- **Sub-workflow reference:** None

---

## Block 3 — Response Aggregation and Validation

### Overview
This block merges the four parallel model results into one stream and converts them into a validated internal structure. It also records parse failures instead of stopping the workflow.

### Nodes Involved
- Merge All 4 LLMs
- Parse & Validate Responses

### Node Details

#### 11. Merge All 4 LLMs
- **Type and technical role:** `n8n-nodes-base.merge`; combines outputs from four branches
- **Configuration choices:**
  - Number of inputs set to `4`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs ← `LLM1 `, `LLM2 `, `LLM3`, `LLM4 `
  - Output → `Parse & Validate Responses`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases or potential failure types:**
  - If one upstream branch fully fails and emits nothing, merge behavior may vary depending on execution state
  - Because several LLM nodes continue on error, the merge may still receive partial or malformed data
- **Sub-workflow reference:** None

#### 12. Parse & Validate Responses
- **Type and technical role:** `n8n-nodes-base.code`; parses LLM JSON and builds normalized response objects
- **Configuration choices:**
  - Uses JavaScript to:
    - iterate over all merged items
    - derive a model name from node metadata
    - read `output` or `text`
    - strip Markdown code fences like ```json
    - parse JSON
    - validate presence and types of `answer` and `confidence`
    - clamp confidence to `[0,1]`
    - flag invalid parses as `valid: false`
- **Key expressions or variables used:**
  - `$input.all()`
  - `item.json.output`
  - `item.json.text`
  - `item.json?.$node?.name`
- **Input and output connections:**
  - Input ← `Merge All 4 LLMs`
  - Output → `Similarity Analysis`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Node-name extraction may not work consistently if `$node` metadata is absent
  - Provider outputs may be wrapped in unexpected structures
  - If all outputs are invalid, downstream logic will have very limited data
  - A non-string `answer` or non-numeric `confidence` is treated as invalid
- **Sub-workflow reference:** None

---

## Block 4 — Similarity and Calibration Engine

### Overview
This block measures how much the valid answers agree semantically and adjusts confidence scores accordingly. It is the decision-making core of the workflow.

### Nodes Involved
- Similarity Analysis
- Confidence Calibration

### Node Details

#### 13. Similarity Analysis
- **Type and technical role:** `n8n-nodes-base.code`; computes pairwise agreement scores
- **Configuration choices:**
  - Filters valid responses only
  - If fewer than two valid responses exist, returns an error and empty similarity matrix
  - Normalizes text by lowercasing and stripping punctuation
  - Uses:
    - Jaccard similarity
    - Cosine similarity on full text
    - Cosine similarity on first sentence
  - Includes a shortcut rule:
    - if one normalized answer fully contains the shorter answer, similarity is forced to `0.95`
  - Weighted similarity formula:
    - Jaccard `0.25`
    - Cosine `0.45`
    - First sentence `0.30`
  - Adds `avgSimilarityToOthers` to each response
- **Key expressions or variables used:**
  - `$input.first().json.responses`
- **Input and output connections:**
  - Input ← `Parse & Validate Responses`
  - Output → `Confidence Calibration`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Lexical similarity is not true semantic equivalence
  - Different wording with same meaning may score lower than expected
  - Very short answers can distort similarity
  - If only one valid response remains, workflow cannot meaningfully compare answers
- **Sub-workflow reference:** None

#### 14. Confidence Calibration
- **Type and technical role:** `n8n-nodes-base.code`; recalibrates confidence using consensus signals
- **Configuration choices:**
  - Thresholds:
    - high confidence: `>= 0.80`
    - low agreement: `< 0.30`
    - strong agreement: `>= 0.50`
    - moderate disagreement: `< 0.30`
    - penalty multiplier: `0.70`
  - Rules:
    - high-confidence + low-agreement → cap at `0.40`
    - low-confidence + strong-agreement → boost to at least `0.65`
    - mid-confidence + moderate-disagreement → multiply by `0.70`
  - Computes workflow-level metrics:
    - `avgSimilarity`
    - `hasStrongConsensus`
    - `hasExtremeDivergence` when average similarity `< 0.10`
- **Key expressions or variables used:**
  - `$input.first().json.responses`
- **Input and output connections:**
  - Input ← `Similarity Analysis`
  - Output → `Check for Extreme Divergence`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Calibration is heuristic, not probabilistically grounded
  - Strong but wrong consensus is still possible
  - If responses array is empty, average calculations may become unsafe in other environments
- **Sub-workflow reference:** None

---

## Block 5 — Consensus Decision

### Overview
This block determines whether the workflow should return a single weighted answer or present all model perspectives because disagreement is too high.

### Nodes Involved
- Check for Extreme Divergence
- Peer Review Fallback
- Weighted Consensus

### Node Details

#### 15. Check for Extreme Divergence
- **Type and technical role:** `n8n-nodes-base.if`; routes execution based on consensus strength
- **Configuration choices:**
  - Checks whether `consensusMetrics.hasExtremeDivergence` is `true`
  - Boolean strict validation enabled
- **Key expressions or variables used:**
  - `={{ $json.consensusMetrics.hasExtremeDivergence }}`
- **Input and output connections:**
  - Input ← `Confidence Calibration`
  - True output → `Peer Review Fallback`
  - False output → `Weighted Consensus`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Missing `consensusMetrics` would break branching logic
  - Extreme divergence threshold is rigid and may need tuning
- **Sub-workflow reference:** None

#### 16. Peer Review Fallback
- **Type and technical role:** `n8n-nodes-base.code`; formats all responses as separate perspectives
- **Configuration choices:**
  - Sorts responses by original confidence descending
  - Builds `peerReviewSummary` with:
    - status
    - explanation
    - response list
  - Sets `mode` to `peer_review_fallback`
- **Key expressions or variables used:**
  - `$input.first().json.responses`
- **Input and output connections:**
  - Input ← `Check for Extreme Divergence` (true branch)
  - Output → `Format Final Output`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Original confidence may be misleading if a model is confidently wrong
  - Invalid responses are already filtered earlier, so failures may be invisible here
- **Sub-workflow reference:** None

#### 17. Weighted Consensus
- **Type and technical role:** `n8n-nodes-base.code`; chooses a primary answer weighted by calibrated confidence
- **Configuration choices:**
  - Sums all calibrated confidences
  - Converts each response into a normalized weight
  - Sorts by weight descending
  - Uses top-weight answer as primary consensus
  - Collects minority views with weight `> 15%`
  - Counts how many models were calibrated
  - Sets `mode` to `weighted_consensus`
- **Key expressions or variables used:**
  - `$input.first().json.responses`
  - `$input.first().json.consensusMetrics`
- **Input and output connections:**
  - Input ← `Check for Extreme Divergence` (false branch)
  - Output → `Format Final Output`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - This is not true answer synthesis; it selects the highest-weight response
  - If total calibrated confidence is zero, weights are evenly distributed
  - Similar but slightly better answers may be ignored in favor of one winner
- **Sub-workflow reference:** None

---

## Block 6 — Final Formatting and Chat Delivery

### Overview
This block converts the selected result mode into a structured final report and then into a user-facing chat message. It is responsible for display wording, agreement meter, and failure notes.

### Nodes Involved
- Format Final Output
- Format Output (chat message)
- Chat

### Node Details

#### 18. Format Final Output
- **Type and technical role:** `n8n-nodes-base.code`; builds a unified final JSON object for either branch
- **Configuration choices:**
  - Adds:
    - `mode`
    - ISO `timestamp`
  - In weighted consensus mode:
    - creates `result`
    - exposes `minorityOpinions`
    - creates detailed `calibrationReport`
    - creates `fullBreakdown`
  - In peer review mode:
    - creates generic result metadata
    - exposes `allPerspectives`
    - includes explanation
  - Attempts to derive `failedModels` from `allResponses`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Inputs ← `Weighted Consensus` or `Peer Review Fallback`
  - Output → `Format Output (chat message)`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - `failedModels` detection works only in weighted mode because peer-review output does not include `allResponses`
  - Parse-error models are filtered earlier by `Similarity Analysis`, so failure reporting may undercount invalid providers
- **Sub-workflow reference:** None

#### 19. Format Output (chat message)
- **Type and technical role:** `n8n-nodes-base.code`; converts structured output to chat-ready markdown text
- **Configuration choices:**
  - In weighted consensus mode:
    - shows agreement label
    - shows primary answer
    - builds a 10-block agreement bar
    - optionally mentions calibration count
    - lists minority opinions
  - In peer review mode:
    - explains disagreement causes
    - lists each model perspective with emoji based on confidence
    - adds recommendation to review all perspectives
  - Appends failed-model note if present
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Input ← `Format Final Output`
  - Output → `Chat`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Expects weighted mode fields under `result`, `calibrationReport`, `minorityOpinions`
  - Some field names are tightly coupled to upstream formatting and may break if schema changes
- **Sub-workflow reference:** None

#### 20. Chat
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chat`; returns final response to the active chat session
- **Configuration choices:**
  - Sends `chatResponse` as outgoing message
- **Key expressions or variables used:**
  - `={{ $json.chatResponse }}`
- **Input and output connections:**
  - Input ← `Format Output (chat message)`
  - Final response node
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - If `chatResponse` is empty or undefined, user gets a blank or broken response
  - Depends on chat trigger response-mode compatibility
- **Sub-workflow reference:** None

---

## Block 7 — Documentation / Annotation Nodes

### Overview
These sticky notes are non-executable documentation elements embedded in the canvas. They explain the workflow concept, setup steps, and major processing areas.

### Nodes Involved
- Sticky Note
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8

### Node Details

#### 21. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation
- **Configuration choices:**
  - Large introductory note with purpose, setup, and customization guidance
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 22. Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas section label
- **Configuration choices:**
  - Labels the parallel LLM generation area
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 23. Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas section label
- **Configuration choices:**
  - Labels the calibration engine area
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 24. Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas section label
- **Configuration choices:**
  - Labels the consensus/fallback branching area
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 25. Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas section label
- **Configuration choices:**
  - Labels the chat response area
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for overall workflow |  |  | ## AI Consensus Engine: 4 Models, 1 Trusted Answer<br>Think of this as a panel of experts that actually checks each other's work. Instead of trusting one AI's answer blindly, the system cross-examines multiple models and calls out the ones that are bluffing.<br><br>### How it works<br><br>Ask: Your question goes to four LLMs in parallel, each self-reporting its confidence.<br>Compare: Dual similarity analysis (Jaccard + Cosine) measures how much the answers actually agree.<br>Calibrate: Overconfident outliers get penalized. Underconfident models matching consensus get boosted.<br>Deliver: Strong agreement returns a single weighted answer. True disagreement switches to peer review mode showing every perspective.<br><br>### Setup<br><br>- [ ] Add API credentials for the four LLM providers ( OpenAI, Anthropic, Google Gemini, Groq)<br>- [ ] Activate the workflow and open the chat window<br>- [ ] Type any question and wait for the consensus analysis to come back<br><br>### Customization<br>Swap any LLM for another or add more parallel branches. Adjust similarity weights in the Similarity Analysis node. Change agreement tier thresholds in the Format Chat Message node. Replace the chat trigger with a webhook, Slack command, or any other entry point. |
| Groq Chat Model3 | @n8n/n8n-nodes-langchain.lmChatGroq | Groq model backend for one LLM branch |  | LLM4  | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas section label |  |  | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas section label |  |  | ## Calibration Engine<br><br>Detects overconfident outliers and underconfident consensus |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas section label |  |  | ## Consensus or Fallback<br><br>Weighted average if consensus exists, peer review if extreme divergence |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point |  | Set User Prompt |  |
| Chat | @n8n/n8n-nodes-langchain.chat | Returns final chat response | Format Output (chat message) |  | ## Chat Response<br><br>Formats the report into a readable message and sends it to the user |
| Parse & Validate Responses | n8n-nodes-base.code | Parses and validates LLM JSON outputs | Merge All 4 LLMs | Similarity Analysis |  |
| Similarity Analysis | n8n-nodes-base.code | Computes pairwise answer similarity | Parse & Validate Responses | Confidence Calibration | ## Calibration Engine<br><br>Detects overconfident outliers and underconfident consensus |
| Confidence Calibration | n8n-nodes-base.code | Recalibrates confidence from agreement patterns | Similarity Analysis | Check for Extreme Divergence | ## Calibration Engine<br><br>Detects overconfident outliers and underconfident consensus |
| Weighted Consensus | n8n-nodes-base.code | Selects highest-weight consensus answer | Check for Extreme Divergence | Format Final Output | ## Consensus or Fallback<br><br>Weighted average if consensus exists, peer review if extreme divergence |
| Peer Review Fallback | n8n-nodes-base.code | Returns separate model perspectives when divergence is high | Check for Extreme Divergence | Format Final Output | ## Consensus or Fallback<br><br>Weighted average if consensus exists, peer review if extreme divergence |
| Format Output (chat message) | n8n-nodes-base.code | Converts structured result into readable chat text | Format Final Output | Chat | ## Chat Response<br><br>Formats the report into a readable message and sends it to the user |
| Format Final Output | n8n-nodes-base.code | Normalizes final result structure for either branch | Weighted Consensus, Peer Review Fallback | Format Output (chat message) | ## Consensus or Fallback<br><br>Weighted average if consensus exists, peer review if extreme divergence |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas section label |  |  | ## Chat Response<br><br>Formats the report into a readable message and sends it to the user |
| Merge All 4 LLMs | n8n-nodes-base.merge | Combines four LLM branches | LLM1 , LLM2 , LLM3, LLM4  | Parse & Validate Responses |  |
| Set User Prompt | n8n-nodes-base.set | Stores incoming question as `userPrompt` | When chat message received | LLM1 , LLM2 , LLM3, LLM4  |  |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Gemini backend for one LLM branch |  | LLM1  | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Anthropic backend for one LLM branch |  | LLM2  | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI backend for one LLM branch |  | LLM3 | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| LLM1  | @n8n/n8n-nodes-langchain.agent | Gemini-powered answering agent | Set User Prompt, Google Gemini Chat Model | Merge All 4 LLMs | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| LLM2  | @n8n/n8n-nodes-langchain.agent | Anthropic-powered answering agent | Set User Prompt, Anthropic Chat Model | Merge All 4 LLMs | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| LLM3 | @n8n/n8n-nodes-langchain.agent | OpenAI-powered answering agent | Set User Prompt, OpenAI Chat Model | Merge All 4 LLMs | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| LLM4  | @n8n/n8n-nodes-langchain.agent | Groq-powered answering agent | Set User Prompt, Groq Chat Model3 | Merge All 4 LLMs | ## Parallel LLM Generation<br><br>Each model answers independently with confidence rating |
| Check for Extreme Divergence | n8n-nodes-base.if | Branches between consensus and peer review modes | Confidence Calibration | Peer Review Fallback, Weighted Consensus | ## Consensus or Fallback<br><br>Weighted average if consensus exists, peer review if extreme divergence |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI Consensus Engine: 4 Models, 1 Trusted Answer`.
   - Ensure the workflow uses n8n versions supporting:
     - LangChain chat trigger
     - LangChain agent nodes
     - Gemini, Anthropic, OpenAI, and Groq chat model nodes
     - Code nodes version 2
     - Merge node version 3

2. **Add the chat trigger node**
   - Create node: **When chat message received**
   - Type: `Chat Trigger`
   - Set:
     - **Public** = enabled
     - **Response Mode** = `responseNodes`
     - **Initial Messages** =
       `Hi there! 👋`
       `Ask me any question and I'll analyze it using 4 AI models!`

3. **Add a Set node to normalize the input**
   - Create node: **Set User Prompt**
   - Type: `Set`
   - Add one field:
     - `userPrompt` as string
     - Value: `{{ $json.chatInput }}`
   - Connect:
     - `When chat message received` → `Set User Prompt`

4. **Create the Gemini model backend**
   - Create node: **Google Gemini Chat Model**
   - Type: `Google Gemini Chat Model`
   - Leave options default unless you want custom temperature or safety behavior
   - Attach valid Google Gemini credentials

5. **Create the first agent branch**
   - Create node: **LLM1**
   - Type: `AI Agent`
   - Prompt type: `Define`
   - Text prompt:
     - Ask the model to answer `{{ $json.userPrompt }}`
     - Require JSON only with keys `answer`, `confidence`, `reasoning`
     - Specify confidence as float between `0.0` and `1.0`
   - System message:
     - “You are a helpful AI assistant. Always return responses in valid JSON format as requested.”
   - Set node error handling to continue regular output if available
   - Connect:
     - `Set User Prompt` → `LLM1`
     - `Google Gemini Chat Model` → `LLM1` through AI language model connection

6. **Create the Anthropic backend and second agent**
   - Create node: **Anthropic Chat Model**
   - Select an available Claude model
   - Attach Anthropic credentials
   - Create node: **LLM2**
   - Use the same agent prompt and system message as `LLM1`
   - Connect:
     - `Set User Prompt` → `LLM2`
     - `Anthropic Chat Model` → `LLM2` through AI language model connection

7. **Create the OpenAI backend and third agent**
   - Create node: **OpenAI Chat Model**
   - Select a model equivalent to the original, such as `gpt-5-mini` if available
   - Attach OpenAI credentials
   - Leave built-in tools empty
   - Create node: **LLM3**
   - Use the same agent prompt and system message
   - Connect:
     - `Set User Prompt` → `LLM3`
     - `OpenAI Chat Model` → `LLM3` through AI language model connection

8. **Create the Groq backend and fourth agent**
   - Create node: **Groq Chat Model3**
   - Select model `llama-3.3-70b-versatile` or equivalent available Groq model
   - Attach Groq credentials
   - Create node: **LLM4**
   - Use the same agent prompt and system message
   - Connect:
     - `Set User Prompt` → `LLM4`
     - `Groq Chat Model3` → `LLM4` through AI language model connection

9. **Add the merge node**
   - Create node: **Merge All 4 LLMs**
   - Type: `Merge`
   - Set **Number of Inputs** = `4`
   - Connect:
     - `LLM1` → input 1
     - `LLM2` → input 2
     - `LLM3` → input 3
     - `LLM4` → input 4

10. **Add the response parsing code node**
   - Create node: **Parse & Validate Responses**
   - Type: `Code`
   - Use JavaScript
   - Implement logic that:
     - loops through all merged items
     - extracts raw output from `output` or `text`
     - removes triple-backtick code fences if present
     - parses JSON
     - validates `answer` as string and `confidence` as number
     - clamps confidence into `[0,1]`
     - stores parse failures as invalid entries with confidence `0`
     - returns:
       - `responses`
       - `validCount`
   - Connect:
     - `Merge All 4 LLMs` → `Parse & Validate Responses`

11. **Add the similarity analysis node**
   - Create node: **Similarity Analysis**
   - Type: `Code`
   - Implement:
     - valid-response filtering
     - normalization to lowercase and punctuation removal
     - word-frequency cosine similarity
     - Jaccard similarity
     - first-sentence cosine comparison
     - weighted combined similarity:
       - Jaccard `0.25`
       - Cosine `0.45`
       - First sentence `0.30`
     - assign `avgSimilarityToOthers` to each valid response
   - If fewer than 2 valid responses exist, return an error payload
   - Connect:
     - `Parse & Validate Responses` → `Similarity Analysis`

12. **Add the confidence calibration node**
   - Create node: **Confidence Calibration**
   - Type: `Code`
   - Implement thresholds:
     - high confidence `0.80`
     - low agreement `0.30`
     - strong agreement `0.50`
     - extreme divergence average similarity `< 0.10`
   - Add rules:
     - overconfident outlier capped at `0.40`
     - underconfident consensus response boosted to at least `0.65`
     - moderate disagreement penalized by `0.70`
   - Return:
     - calibrated `responses`
     - `consensusMetrics` with:
       - `avgSimilarity`
       - `hasStrongConsensus`
       - `hasExtremeDivergence`
   - Connect:
     - `Similarity Analysis` → `Confidence Calibration`

13. **Add the branch decision node**
   - Create node: **Check for Extreme Divergence**
   - Type: `If`
   - Condition:
     - boolean true
     - left value: `{{ $json.consensusMetrics.hasExtremeDivergence }}`
   - Connect:
     - `Confidence Calibration` → `Check for Extreme Divergence`

14. **Add the fallback branch**
   - Create node: **Peer Review Fallback**
   - Type: `Code`
   - Logic:
     - sort responses by original confidence descending
     - create an object containing:
       - `peerReviewSummary.status`
       - `peerReviewSummary.explanation`
       - `peerReviewSummary.responses`
       - `mode = "peer_review_fallback"`
   - Connect:
     - `Check for Extreme Divergence` true branch → `Peer Review Fallback`

15. **Add the weighted consensus branch**
   - Create node: **Weighted Consensus**
   - Type: `Code`
   - Logic:
     - compute total calibrated confidence
     - derive per-response weights
     - sort descending
     - top answer becomes primary answer
     - collect minority views where weight `> 0.15`
     - set `mode = "weighted_consensus"`
   - Connect:
     - `Check for Extreme Divergence` false branch → `Weighted Consensus`

16. **Add the structured final output node**
   - Create node: **Format Final Output**
   - Type: `Code`
   - Logic for weighted mode:
     - create `result`
     - add `minorityOpinions`
     - add `calibrationReport`
     - add `fullBreakdown`
   - Logic for peer-review mode:
     - create generic result message
     - add `allPerspectives`
     - add `explanation`
   - Also add `timestamp`
   - Connect:
     - `Weighted Consensus` → `Format Final Output`
     - `Peer Review Fallback` → `Format Final Output`

17. **Add the chat message formatter**
   - Create node: **Format Output (chat message)**
   - Type: `Code`
   - Logic:
     - if weighted consensus:
       - show agreement label
       - include final answer
       - build agreement bar from consensus percentage
       - mention calibration adjustments
       - include minority opinions
     - else:
       - explain disagreement
       - list each perspective and confidence
       - add recommendation note
     - return `{ chatResponse }`
   - Connect:
     - `Format Final Output` → `Format Output (chat message)`

18. **Add the final chat response node**
   - Create node: **Chat**
   - Type: `Chat`
   - Message:
     - `{{ $json.chatResponse }}`
   - Connect:
     - `Format Output (chat message)` → `Chat`

19. **Add optional sticky notes for maintainability**
   - Add a large overview note with:
     - purpose
     - setup checklist
     - customization suggestions
   - Add section notes for:
     - Parallel LLM Generation
     - Calibration Engine
     - Consensus or Fallback
     - Chat Response

20. **Configure credentials**
   - Add and test:
     - OpenAI API credential
     - Anthropic API credential
     - Google Gemini API credential
     - Groq API credential
   - Verify each selected model exists in your own account and region

21. **Activate and test**
   - Activate the workflow
   - Open the chat interface
   - Submit:
     - a factual question with likely agreement
     - an ambiguous or controversial question to trigger fallback
   - Confirm both branches produce valid results

22. **Recommended hardening improvements**
   - Add explicit handling for fewer than 2 valid responses
   - Preserve invalid/failed responses through all later stages so failed-model reporting remains accurate
   - Normalize model names explicitly instead of relying on node metadata
   - Add execution timeout protections for slow model providers
   - Add rate-limit protection if the public chat is exposed widely

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Consensus Engine: 4 Models, 1 Trusted Answer | Workflow concept shown in canvas notes |
| The workflow acts like a panel of experts that cross-check one another instead of trusting one model blindly. | Overall workflow positioning |
| Setup checklist: add API credentials for OpenAI, Anthropic, Google Gemini, and Groq; activate the workflow; open the chat window; ask a question. | Operational setup |
| Customization guidance: swap models, add more branches, adjust similarity weights in the Similarity Analysis node, change agreement thresholds in the chat formatting node, or replace the chat trigger with webhook/Slack/other entry points. | Extension and maintenance guidance |

## Additional implementation observations

- The workflow title in metadata is **AI Consensus Engine: 4 Models, 1 Trusted Answer**, while your supplied title is **Combine answers from OpenAI, Anthropic, Gemini and Groq into one consensus**. They describe the same workflow.
- There are **no sub-workflows** in this JSON.
- There is **one entry point only**: the chat trigger.
- The workflow is currently **inactive** (`active: false`) in the provided JSON.
- A practical issue exists in the current logic: invalid parse results are filtered out before final output, so the displayed `failedModels` list may be incomplete or empty even when one or more providers fail.
- Another practical issue: the workflow does not truly synthesize multiple answers into a new combined answer; in weighted mode it selects the highest-weight response as the primary answer.