Evaluate AI workflows using Google Sheets, Gemini, Claude, GPT, and Perplexity

https://n8nworkflows.xyz/workflows/evaluate-ai-workflows-using-google-sheets--gemini--claude--gpt--and-perplexity-12644


# Evaluate AI workflows using Google Sheets, Gemini, Claude, GPT, and Perplexity

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Evaluate AI workflows using Google Sheets, Gemini, Claude, GPT, and Perplexity  
**Workflow name (n8n):** Set up evaluations in n8n

This workflow demonstrates how to **run AI tasks against a test dataset** (from **Google Sheets or n8n Data Tables**) and then **score the AI outputs using n8n Evaluations**. It includes multiple example “evaluation patterns” (Categorization, Tools Used, String Similarity, Helpfulness, Correctness) and a small demo of “evaluations in action” using a Gmail-triggered document flow.

### Logical blocks (by dependency)
1. **1.1 Dataset-driven evaluation entry points (Evaluation Trigger nodes)**  
   Multiple triggers fetch dataset rows from Google Sheets (and one from Data Tables).
2. **1.2 AI task execution (Agents / Extractor / Summarization)**  
   Different AI nodes produce outputs (World Series answer, address cleaning, rewriting, summarization, information extraction).
3. **1.3 Evaluation gating (checkIfEvaluating)**  
   Splits flow into “evaluation mode” versus “normal execution” (one branch is often unused in this template).
4. **1.4 Persist outputs (Evaluation node writing outputs back to Sheets)**  
   Stores run results (and sometimes tool usage) in Google Sheets.
5. **1.5 Compute metrics (setMetrics)**  
   Applies evaluation metrics such as Categorization, String Similarity, Tools Used, Helpfulness, Correctness.
6. **1.6 Non-evaluation demo branch (Gmail Trigger → If → Send message)**  
   Shows a “regular workflow” path that can exist alongside evaluation instrumentation.

---

## 2. Block-by-Block Analysis

### 2.1 Example 1 — Categorization metric (World Series winners, Google Sheets)
**Overview:** Fetches a row containing a `year` and expected `team`, asks an agent to output the winner, then evaluates if the result exactly matches the expected team (categorization metric).

**Nodes involved:**
- When fetching a dataset row
- Google Gemini Chat Model
- World Series Searcher
- Evaluation3 (checkIfEvaluating)
- Evaluation (write outputs)
- Evaluation5 (setMetrics: categorization)

#### Node details

**When fetching a dataset row**
- **Type / role:** `evaluationTrigger` — dataset row fetcher (Google Sheets).
- **Config:** Source = Google Sheets; Spreadsheet “eval test” (`documentId=1absZ-...`), Sheet “Sheet1” (`gid=0`).
- **Output fields (expected):** at least `year`, `team` (based on downstream expressions).
- **Connections:** Outputs → World Series Searcher.
- **Edge cases:** missing columns (`year`/`team`), Sheets auth issues, sheet permissions, empty rows.

**Google Gemini Chat Model**
- **Type / role:** LangChain chat model connector for Gemini.
- **Config:** model `models/gemini-3-pro-preview`.
- **Credentials:** Google PaLM / Gemini credential “n8n gemini 3”.
- **Connections:** Provides `ai_languageModel` input to World Series Searcher.
- **Edge cases:** model availability/renames, quota/rate limits, invalid API key.

**World Series Searcher**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent that responds with team name only.
- **Config:** Prompt uses expression `Who won the world series for: {{ $json.year }} ... Only output the team name ... else Undetermined`.
- **Options:** `returnIntermediateSteps=false` (so no tool call trace).
- **Inputs:** Main input from trigger row; language model from Gemini node.
- **Output:** `output` containing team name or “Undetermined”.
- **Connections:** Output → Evaluation3.
- **Edge cases:** LLM may output extra text (breaks exact-match categorization), missing `year`.

**Evaluation3**
- **Type / role:** `evaluation` — operation `checkIfEvaluating` (gating).
- **Config:** `operation=checkIfEvaluating`.
- **Behavior:** Produces two outputs: evaluation path vs normal path (n8n’s evaluation runtime decides).
- **Connections:** Output 0 → Evaluation; output 1 unused in this example.
- **Edge cases:** if run outside evaluation context, “evaluation branch” may not execute as expected.

**Evaluation**
- **Type / role:** `evaluation` — “set outputs” to Google Sheets.
- **Config:** Source = Google Sheets; writes an output column mapping:
  - `Result = {{ $json.output }}`
  Sheet = “Sheet1” (`gid=0`) in “eval test”.
- **Connections:** → Evaluation5.
- **Edge cases:** write permissions, sheet structure mismatch, `output` missing.

**Evaluation5**
- **Type / role:** `evaluation` — `operation=setMetrics` with metric `categorization`.
- **Config:**
  - Metric: `categorization` (named “Categorization”)
  - `actualAnswer = {{ $('Evaluation3').item.json.output }}`
  - `expectedAnswer = {{ $('When fetching a dataset row').item.json.team }}`
- **Connections:** terminal (no outgoing).
- **Edge cases:** strict exact-match; whitespace/case mismatches; references to node items fail if item pairing differs.

---

### 2.2 Example 2 — Tools Used metric (Claude agent + Perplexity tool, Google Sheets)
**Overview:** Fetches a row describing the year and the *expected tool usage*, runs a Claude-based agent that can call Perplexity, records tool used, and scores whether the expected tool was used.

**Nodes involved:**
- When fetching a dataset row2
- Anthropic Chat Model
- Perplexity Tool
- World Series Searcher1
- Evaluation9 (checkIfEvaluating)
- Evaluation1 (write outputs: Tools Used)
- Evaluation10 (setMetrics: toolsUsed)

#### Node details

**When fetching a dataset row2**
- **Type / role:** `evaluationTrigger` (Google Sheets).
- **Config:** Spreadsheet “eval test”; Sheet2 (`gid=890240055`).
- **Expected columns:** `year`, `tool` (based on downstream expressions).
- **Connections:** → World Series Searcher1.

**Anthropic Chat Model**
- **Type / role:** Claude chat model connector.
- **Config:** Model `claude-sonnet-4-5-20250929` (display “Claude Sonnet 4.5”).
- **Credentials:** “Ryan Nolan Account”.
- **Connections:** `ai_languageModel` → World Series Searcher1.
- **Edge cases:** model naming/availability; Anthropic rate limits.

**Perplexity Tool**
- **Type / role:** `perplexityTool` — tool callable by agents.
- **Config:** Message content uses `$fromAI('message0_Text', '', 'string')` (LangChain tool input binding).
- **Credentials:** “Perplexity account 2”.
- **Connections:** `ai_tool` → World Series Searcher1 (tool is available to agent).
- **Edge cases:** tool call failures/timeouts; empty observations; tool not invoked if agent decides not to.

**World Series Searcher1**
- **Type / role:** LangChain agent that should use Perplexity before “Undetermined”.
- **Config:** Prompt: asks winner for `{{$json.year}}`, “Before writing Undetermined use the perplexity tool…”.
- **Options:** `returnIntermediateSteps=true` (critical for Tools Used evaluation).
- **Output:** `output` plus `intermediateSteps` containing tool calls (if any).
- **Connections:** → Evaluation9.
- **Edge cases:** `intermediateSteps` absent if no tool call; output may include punctuation; tool name mismatch.

**Evaluation9**
- **Type / role:** `evaluation` — `checkIfEvaluating`.
- **Connections:** output 0 → Evaluation1.
- **Edge cases:** running outside evaluation context.

**Evaluation1**
- **Type / role:** `evaluation` — writes “Tools Used” output back to Sheets.
- **Config:** Google Sheets Sheet2; output mapping:
  - `Tools Used = {{ $json.intermediateSteps[0].action.tool }}`
- **Connections:** → Evaluation10.
- **Edge cases:** indexing `[0]` fails if no tools were used; intermediateSteps structure differences between node versions.

**Evaluation10**
- **Type / role:** `evaluation` — `setMetrics` metric `toolsUsed`.
- **Config:**
  - `expectedTools = {{ $('When fetching a dataset row2').item.json.tool }}`
  - `intermediateSteps = {{ $('Evaluation9').item.json.intermediateSteps[0].action.tool }}`
- **Connections:** terminal.
- **Edge cases:** expects a comparable tool identifier; null intermediate steps; expression item mismatch.

---

### 2.3 Example 3 — String Similarity metric (Address cleaning, Google Sheets)
**Overview:** Fetches messy address text and an expected cleaned result, runs an agent to standardize formatting, then scores similarity via edit-distance-based metric.

**Nodes involved:**
- When fetching a dataset row3
- Anthropic Chat Model1
- US Postal Formatter
- Evaluation11 (checkIfEvaluating)
- Evaluation2 (write outputs)
- Evaluation12 (setMetrics: stringSimilarity)

#### Node details

**When fetching a dataset row3**
- **Type / role:** `evaluationTrigger` (Google Sheets).
- **Config:** Spreadsheet “eval test”; Sheet3 (`gid=1065824480`).
- **Expected columns:** `messydata`, `Expected clean data`.
- **Connections:** → US Postal Formatter.

**Anthropic Chat Model1**
- **Type / role:** Claude connector.
- **Config:** same model as above (`claude-sonnet-4-5-20250929`).
- **Connections:** `ai_languageModel` → US Postal Formatter.

**US Postal Formatter**
- **Type / role:** LangChain agent for deterministic formatting.
- **Config:** Prompt returns *only cleaned address* using `{{$json.messydata}}`.
- **Options:** `returnIntermediateSteps=false`.
- **Connections:** → Evaluation11.
- **Edge cases:** LLM may add commentary; inconsistent abbreviations; missing `messydata`.

**Evaluation11**
- **Type / role:** `checkIfEvaluating`.
- **Connections:** output 0 → Evaluation2.

**Evaluation2**
- **Type / role:** writes `Result={{$json.output}}` to Sheet1 (gid=0).
- **Connections:** → Evaluation12.

**Evaluation12**
- **Type / role:** `setMetrics` with metric `stringSimilarity`.
- **Config:**
  - `actualAnswer = {{ $('Evaluation11').item.json.output }}`
  - `expectedAnswer = {{ $('When fetching a dataset row3').item.json['Expected clean data'] }}`
- **Edge cases:** similarity penalizes minor punctuation/case; ensure consistent normalization.

---

### 2.4 Example 4 — Helpfulness metric (Product description rewriting, Google Sheets + GPT)
**Overview:** Rewrites product descriptions and scores whether the response answers the rewriting task (1–5 helpfulness scale), using an OpenAI chat model for evaluation.

**Nodes involved:**
- When fetching a dataset row5
- Google Gemini Chat Model4
- Product Description Rewriter
- Evaluation17 (checkIfEvaluating)
- Evaluation16 (write outputs)
- Evaluation18 (setMetrics: helpfulness)
- OpenAI Chat Model

#### Node details

**When fetching a dataset row5**
- **Type / role:** `evaluationTrigger` (Google Sheets Sheet1 gid=0).
- **Expected column:** `originalDescription`.
- **Connections:** → Product Description Rewriter.

**Google Gemini Chat Model4**
- **Type / role:** Gemini connector for the rewriting agent.
- **Config:** `models/gemini-3-pro-preview`.
- **Connections:** `ai_languageModel` → Product Description Rewriter.

**Product Description Rewriter**
- **Type / role:** LangChain agent rewriting text.
- **Config:** Multi-requirement prompt, 2–3 sentences, preserve specs; input `{{$json.originalDescription}}`.
- **Options:** `returnIntermediateSteps=false`.
- **Connections:** → Evaluation17.

**Evaluation17**
- **Type / role:** `checkIfEvaluating`.
- **Connections:** output 0 → Evaluation16.

**Evaluation16**
- **Type / role:** writes `Result` to Sheet1.
- **Connections:** → Evaluation18.

**OpenAI Chat Model**
- **Type / role:** OpenAI connector used by evaluation metric node.
- **Config:** model `gpt-5-mini`.
- **Connections:** `ai_languageModel` → Evaluation18.
- **Edge cases:** API quotas; model name availability.

**Evaluation18**
- **Type / role:** `setMetrics` metric `helpfulness` (AI-judged 1–5).
- **Config:** `actualAnswer = {{ $('Evaluation17').item.json.output }}`
- **Note:** No explicit `expectedAnswer` for helpfulness; it judges based on prompt/task context.
- **Edge cases:** evaluation scoring depends on evaluator model; may be inconsistent—control with stable rubric/prompting if needed.

---

### 2.5 Example 5 — Correctness metric (Summarization accuracy, Data Tables + Gemini + GPT evaluator)
**Overview:** Pulls a row from an n8n Data Table, runs a summarization chain, then scores correctness against a reference answer (meaning-consistency 1–5), using an OpenAI model as the evaluator.

**Nodes involved:**
- When fetching a dataset row4
- Google Gemini Chat Model3
- Article Reviewer (summarization chain)
- Evaluation14 (checkIfEvaluating)
- Evaluation13 (write outputs)
- OpenAI Chat Model1
- Evaluation15 (setMetrics: correctness-like config)

#### Node details

**When fetching a dataset row4**
- **Type / role:** `evaluationTrigger` (Data Tables).
- **Config:** `dataTableId=yF4PbotpSaEboTBu` (table name “eval_vid”).
- **Expected columns:** includes `team` (per downstream expectedAnswer), and whatever Article Reviewer uses.
- **Connections:** → Article Reviewer.
- **Edge cases:** Data Table not found, permissions, schema changes.

**Google Gemini Chat Model3**
- **Type / role:** Gemini connector for summarization.
- **Connections:** `ai_languageModel` → Article Reviewer.

**Article Reviewer**
- **Type / role:** `chainSummarization` — summarization chain (LangChain).
- **Config:** default options (not specified); relies on incoming text fields (not defined in JSON).
- **Connections:** → Evaluation14.
- **Edge cases:** if input text field isn’t provided, summarizer may output empty/irrelevant content.

**Evaluation14**
- **Type / role:** `checkIfEvaluating`.
- **Connections:** output 0 → Evaluation13.

**Evaluation13**
- **Type / role:** writes `Result={{$json.output}}` to Sheet1.
- **Connections:** → Evaluation15.

**OpenAI Chat Model1**
- **Type / role:** OpenAI evaluator connector.
- **Config:** model `gpt-5-mini`.
- **Connections:** `ai_languageModel` → Evaluation15.

**Evaluation15**
- **Type / role:** `setMetrics` (operation) with `actualAnswer` and `expectedAnswer`.
- **Config:**  
  - `actualAnswer = {{ $('Evaluation14').item.json.output }}`
  - `expectedAnswer = {{ $('When fetching a dataset row4').item.json.team }}`
- **Important:** In the JSON, `metric` is not explicitly set here (unlike other metric nodes). It appears intended for **Correctness** (per sticky note) and relies on evaluator model connection.
- **Edge cases:** if metric defaults don’t match intent, set `metric=correctness` explicitly; ensure expected reference field is correct (here it’s `team`, which may be placeholder).

---

### 2.6 “Evaluations in Action” demo — Gmail-triggered extractor + evaluation gate + email branching
**Overview:** Demonstrates how a real workflow (Gmail ingestion → file extraction → information extraction) can be instrumented with evaluation gating and reporting, while retaining a normal “If → Send message” operational branch.

**Nodes involved:**
- Gmail Trigger
- Extract from File
- Information Extractor
- Google Gemini Chat Model1
- Evaluation20 (checkIfEvaluating)
- Evaluation19 (write outputs)
- Evaluation21 (setMetrics: categorization)
- If
- Send a message
- Send a message1
- When fetching a dataset row6

#### Node details

**Gmail Trigger**
- **Type / role:** Polling trigger for Gmail.
- **Config:** poll every minute; no filters defined.
- **Credentials:** “Gmail account 3”.
- **Connections:** → Extract from File.
- **Edge cases:** Gmail API quota; missing permissions; message volume; attachment availability.

**Extract from File**
- **Type / role:** Extract text/content from an incoming file/attachment.
- **Config:** default options.
- **Connections:** → Information Extractor.
- **Edge cases:** unsupported file types; corrupted attachments; no binary data in input.

**Information Extractor**
- **Type / role:** LangChain information extractor.
- **Config:** `text="YOUR TEXT GOES HERE"` (placeholder); in practice should be set from extracted content.
- **Connections:** → Evaluation20.
- **Edge cases:** placeholder text not replaced; extraction schema not defined; empty input.

**Google Gemini Chat Model1**
- **Type / role:** Gemini connector.
- **Connections:** `ai_languageModel` → Information Extractor.

**Evaluation20**
- **Type / role:** `checkIfEvaluating` gate.
- **Connections:**
  - Output 0 → Evaluation19 (evaluation branch)
  - Output 1 → If (normal branch)

**Evaluation19**
- **Type / role:** write outputs to Google Sheets Sheet1.
- **Config:** `Result = {{ $json.output }}`.
- **Connections:** → Evaluation21.

**Evaluation21**
- **Type / role:** `setMetrics` with `categorization`.
- **Config:**
  - `actualAnswer = {{ $('Evaluation20').item.json.output }}`
  - `expectedAnswer = {{ $('When fetching a dataset row').item.json.team }}`
- **Edge cases:** expectedAnswer references **a different trigger** (“When fetching a dataset row”) which belongs to Example 1; in a standalone Gmail evaluation, you likely want to reference a Gmail/dataset-specific expected field instead.

**When fetching a dataset row6**
- **Type / role:** `evaluationTrigger` (Google Sheets Sheet1).
- **Connections:** → Information Extractor.
- **Purpose:** Provides dataset-driven entry for the extractor demo (separate from Gmail trigger).

**If**
- **Type / role:** conditional router for non-evaluation execution.
- **Config:** currently has empty `leftValue`/`rightValue` (placeholder), so it is not functional as-is.
- **Connections:** True → Send a message; False → Send a message1.
- **Edge cases:** must define real condition; otherwise behavior may be unpredictable/always false depending on n8n validation.

**Send a message / Send a message1**
- **Type / role:** Gmail send nodes.
- **Config:** default options (placeholders).
- **Credentials:** Gmail account 3.
- **Edge cases:** missing “To/Subject/Body”; Gmail send limits; draft vs send configuration.

---

### 2.7 Unconnected “setup skeleton” nodes (template scaffolding)
**Overview:** These nodes illustrate how to structure an evaluation workflow (trigger → gate → set outputs → set metrics), but are not connected in the execution graph.

**Nodes involved:**
- When fetching a dataset row1 (Data Table trigger placeholder)
- Evaluation4 (checkIfEvaluating)
- Evaluation6 (set outputs to a different spreadsheet)
- Evaluation7 (setMetrics, unspecified metric)
- Evaluation8 (setMetrics stringSimilarity)

#### Node details

**When fetching a dataset row1**
- **Type / role:** evaluationTrigger (Data Tables), but `dataTableId` is empty.
- **Edge cases:** cannot run until a table is selected.

**Evaluation4**
- **Type / role:** checkIfEvaluating skeleton gate.

**Evaluation6**
- **Type / role:** writes outputs to Google Sheets “Kenneth Spreadsheet” → sheet “Cases”.
- **Config:** outputs mapping is empty (`values:[{}]`), so nothing meaningful is written until configured.

**Evaluation7**
- **Type / role:** setMetrics placeholder; no metric configured.

**Evaluation8**
- **Type / role:** setMetrics stringSimilarity placeholder; missing `actualAnswer/expectedAnswer` in parameters, so incomplete.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note12 | stickyNote | Reference / media embed |  |  | `@[youtube](-4LXYOhQ-Z0)` |
| Sticky Note14 | stickyNote | Title banner |  |  | `# Set up evaluations in n8n` |
| Sticky Note13 | stickyNote | Setup notes + links |  |  | `## Setup steps ... 1. Free Skool AI/n8n Group: https://www.skool.com/data-and-ai` |
| Sticky Note13 | stickyNote | Setup notes + links |  |  | `2. LinkedIn: https://www.linkedin.com/in/ryan-p-nolan/` |
| Sticky Note13 | stickyNote | Setup notes + links |  |  | `3. Twitter/X:https://x.com/RyanMattDS` |
| Sticky Note13 | stickyNote | Setup notes + links |  |  | `4. Website: https://ryanandmattdatascience.com/` |
| Sticky Note11 | stickyNote | Intro to extractor demo |  |  | `## Evaluations in Action ... improve the prompt` |
| When fetching a dataset row6 | evaluationTrigger | Dataset entry for extractor demo (Sheets) |  | Information Extractor |  |
| Gmail Trigger | gmailTrigger | Live email entry point |  | Extract from File |  |
| Extract from File | extractFromFile | Attachment/text extraction | Gmail Trigger | Information Extractor |  |
| Google Gemini Chat Model1 | lmChatGoogleGemini | LLM for extractor |  | Information Extractor (ai_languageModel) |  |
| Information Extractor | informationExtractor | Extract structured info from text | Extract from File; When fetching a dataset row6 | Evaluation20 |  |
| Evaluation20 | evaluation | Gate: evaluation vs normal | Information Extractor | Evaluation19; If |  |
| Evaluation19 | evaluation | Persist output to Sheets | Evaluation20 | Evaluation21 |  |
| Evaluation21 | evaluation | Metric: categorization | Evaluation19 |  |  |
| If | if | Normal branch router | Evaluation20 | Send a message; Send a message1 |  |
| Send a message | gmail | Send email (branch A) | If |  |  |
| Send a message1 | gmail | Send email (branch B) | If |  |  |
| Sticky Note2 | stickyNote | Notes on triggers |  |  | `Evaluation Trigger\n\nData Table or Google Sheets -> Expand in the future?` |
| When fetching a dataset row1 | evaluationTrigger | Placeholder Data Table trigger |  |  | (Covered by “Evaluation Trigger...” note) |
| Sticky Note3 | stickyNote | Notes on checkIfEvaluating |  |  | `checkIfEvaluating\n\nFlow branch -> Like if\nWorkflow either goes evaluation route or normal` |
| Evaluation4 | evaluation | Placeholder gate |  |  | (Covered by “checkIfEvaluating...” note) |
| Sticky Note4 | stickyNote | Notes on set outputs |  |  | `set outputs\n\nPut the results from the run into Google Sheets or Data Table` |
| Evaluation6 | evaluation | Placeholder “set outputs” to Sheets |  |  | (Covered by “set outputs...” note) |
| Sticky Note5 | stickyNote | Notes on metrics |  |  | `set metrics\n\nEvaluates how your run performed. May use a Model` |
| Evaluation7 | evaluation | Placeholder setMetrics |  |  | (Covered by “set metrics...” note) |
| Sticky Note6 | stickyNote | Metric definitions (AI-judged) |  |  | `Correctness... 1 to 5 ... Helpfulness... 1 to 5` |
| Sticky Note7 | stickyNote | Metric definitions (non-AI) |  |  | `String Similarity... 0 to 1 ... Categorization... Tools Used...` |
| Sticky Note | stickyNote | Example 1 description |  |  | `## Example 1 Categorization ... World Series Winners - No results 2024 and 2025` |
| When fetching a dataset row | evaluationTrigger | Example 1 dataset row (Sheets) |  | World Series Searcher | (Covered by Example 1 note) |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM for Example 1 agent |  | World Series Searcher (ai_languageModel) | (Covered by Example 1 note) |
| World Series Searcher | agent | Example 1 agent (no tools) | When fetching a dataset row | Evaluation3 | (Covered by Example 1 note) |
| Evaluation3 | evaluation | Gate for Example 1 | World Series Searcher | Evaluation | (Covered by Example 1 note) |
| Evaluation | evaluation | Persist Example 1 result | Evaluation3 | Evaluation5 | (Covered by Example 1 note) |
| Evaluation5 | evaluation | Metric: categorization | Evaluation |  | (Covered by Example 1 note) |
| Sticky Note1 | stickyNote | Example 2 description |  |  | `## Example 2 Tools Used ... Use Perplexity to get new results` |
| When fetching a dataset row2 | evaluationTrigger | Example 2 dataset row (Sheets) |  | World Series Searcher1 | (Covered by Example 2 note) |
| Anthropic Chat Model | lmChatAnthropic | LLM for Example 2 agent |  | World Series Searcher1 (ai_languageModel) | (Covered by Example 2 note) |
| Perplexity Tool | perplexityTool | Tool for agent |  | World Series Searcher1 (ai_tool) | (Covered by Example 2 note) |
| World Series Searcher1 | agent | Example 2 agent (tools enabled) | When fetching a dataset row2 | Evaluation9 | (Covered by Example 2 note) |
| Evaluation9 | evaluation | Gate for Example 2 | World Series Searcher1 | Evaluation1 | (Covered by Example 2 note) |
| Evaluation1 | evaluation | Persist “Tools Used” | Evaluation9 | Evaluation10 | (Covered by Example 2 note) |
| Evaluation10 | evaluation | Metric: toolsUsed | Evaluation1 |  | (Covered by Example 2 note) |
| Sticky Note8 | stickyNote | Example 3 description |  |  | `## Example 3 String Similarity ... Data Cleaning Validation` |
| When fetching a dataset row3 | evaluationTrigger | Example 3 dataset row (Sheets) |  | US Postal Formatter | (Covered by Example 3 note) |
| Anthropic Chat Model1 | lmChatAnthropic | LLM for Example 3 agent |  | US Postal Formatter (ai_languageModel) | (Covered by Example 3 note) |
| US Postal Formatter | agent | Address cleaning agent | When fetching a dataset row3 | Evaluation11 | (Covered by Example 3 note) |
| Evaluation11 | evaluation | Gate for Example 3 | US Postal Formatter | Evaluation2 | (Covered by Example 3 note) |
| Evaluation2 | evaluation | Persist Example 3 result | Evaluation11 | Evaluation12 | (Covered by Example 3 note) |
| Evaluation12 | evaluation | Metric: stringSimilarity | Evaluation2 |  | (Covered by Example 3 note) |
| Sticky Note10 | stickyNote | Example 4 description |  |  | `## Example 4 Helpfulness ... Product Description Rewriting ...` |
| When fetching a dataset row5 | evaluationTrigger | Example 4 dataset row (Sheets) |  | Product Description Rewriter | (Covered by Example 4 note) |
| Google Gemini Chat Model4 | lmChatGoogleGemini | LLM for rewriting |  | Product Description Rewriter (ai_languageModel) | (Covered by Example 4 note) |
| Product Description Rewriter | agent | Rewrite agent | When fetching a dataset row5 | Evaluation17 | (Covered by Example 4 note) |
| Evaluation17 | evaluation | Gate for Example 4 | Product Description Rewriter | Evaluation16 | (Covered by Example 4 note) |
| Evaluation16 | evaluation | Persist Example 4 result | Evaluation17 | Evaluation18 | (Covered by Example 4 note) |
| OpenAI Chat Model | lmChatOpenAi | Evaluator model for helpfulness |  | Evaluation18 (ai_languageModel) | (Covered by Example 4 note) |
| Evaluation18 | evaluation | Metric: helpfulness | Evaluation16 |  | (Covered by Example 4 note) |
| Sticky Note9 | stickyNote | Example 5 description |  |  | `## Example 5 Correctness ... Summary Accuracy Testing ...` |
| When fetching a dataset row4 | evaluationTrigger | Example 5 dataset row (Data Table) |  | Article Reviewer | (Covered by Example 5 note) |
| Google Gemini Chat Model3 | lmChatGoogleGemini | LLM for summarization |  | Article Reviewer (ai_languageModel) | (Covered by Example 5 note) |
| Article Reviewer | chainSummarization | Summarize content | When fetching a dataset row4 | Evaluation14 | (Covered by Example 5 note) |
| Evaluation14 | evaluation | Gate for Example 5 | Article Reviewer | Evaluation13 | (Covered by Example 5 note) |
| Evaluation13 | evaluation | Persist Example 5 result | Evaluation14 | Evaluation15 | (Covered by Example 5 note) |
| OpenAI Chat Model1 | lmChatOpenAi | Evaluator model |  | Evaluation15 (ai_languageModel) | (Covered by Example 5 note) |
| Evaluation15 | evaluation | setMetrics (intended correctness) | Evaluation13 |  | (Covered by Example 5 note) |
| Evaluation8 | evaluation | Placeholder stringSimilarity metric |  |  | (Covered by metric definitions notes) |
| Evaluation6 | evaluation | (Already listed) |  |  | (Covered by set outputs note) |
| 0ad59ce6-... etc. |  |  |  |  |  |

> Note: All sticky note content is preserved above; some notes apply broadly to multiple nodes and are duplicated per row where applicable.

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1. Google Sheets OAuth2 credential (read/write access to target spreadsheets).
   2. Gmail OAuth2 credential (for Gmail Trigger + Gmail Send).
   3. Google Gemini (PaLM/Gemini) API credential.
   4. Anthropic API credential.
   5. OpenAI API credential.
   6. Perplexity API credential.

2. **Example 1 (Categorization)**
   1. Add **Evaluation Trigger** node:
      - Source: Google Sheets
      - Document: `eval test`
      - Sheet: `Sheet1 (gid=0)`
   2. Add **Google Gemini Chat Model** node:
      - Model: `models/gemini-3-pro-preview`
   3. Add **AI Agent** node (rename “World Series Searcher”):
      - Prompt: `Who won the world series for: {{ $json.year }}. Only output the team name...`
      - Return intermediate steps: **off**
      - Connect Gemini model to agent via **ai_languageModel**.
      - Connect Trigger → Agent (main).
   4. Add **Evaluation** node (operation **checkIfEvaluating**) (rename “Evaluation3”):
      - Connect Agent → Evaluation3.
   5. Add **Evaluation** node configured to **write outputs** to Google Sheets (rename “Evaluation”):
      - Source: Google Sheets, same doc/sheet
      - Outputs mapping: `Result = {{ $json.output }}`
      - Connect Evaluation3 (evaluation output) → this node.
   6. Add **Evaluation** node (operation **setMetrics**):
      - Metric: **categorization**
      - `actualAnswer = {{ $('Evaluation3').item.json.output }}`
      - `expectedAnswer = {{ $('When fetching a dataset row').item.json.team }}`
      - Connect outputs writer → setMetrics node.

3. **Example 2 (Tools Used)**
   1. Add **Evaluation Trigger** (Sheets) for `Sheet2 (gid=890240055)`.
   2. Add **Anthropic Chat Model** (Claude Sonnet 4.5).
   3. Add **Perplexity Tool** node; keep default options.
   4. Add **AI Agent** node (rename “World Series Searcher1”):
      - Prompt includes: “Before writing Undetermined use the perplexity tool…”
      - Return intermediate steps: **on**
      - Connect Claude model to agent via **ai_languageModel**.
      - Connect Perplexity Tool to agent via **ai_tool**.
      - Connect Trigger → Agent.
   5. Add **Evaluation checkIfEvaluating** (rename “Evaluation9”), connect Agent → it.
   6. Add **Evaluation write outputs** to Sheet2:
      - Output mapping: `Tools Used = {{ $json.intermediateSteps[0].action.tool }}`
   7. Add **Evaluation setMetrics**:
      - Metric: **toolsUsed**
      - `expectedTools = {{ $('When fetching a dataset row2').item.json.tool }}`
      - `intermediateSteps = {{ $('Evaluation9').item.json.intermediateSteps[0].action.tool }}`

4. **Example 3 (String Similarity)**
   1. Add **Evaluation Trigger** for `Sheet3 (gid=1065824480)`.
   2. Add **Anthropic Chat Model**.
   3. Add **AI Agent** “US Postal Formatter” with prompt referencing `{{$json.messydata}}`.
   4. Add **Evaluation checkIfEvaluating** → **Evaluation write outputs** to Sheet1 (`Result={{$json.output}}`) → **Evaluation setMetrics**:
      - Metric: **stringSimilarity**
      - `expectedAnswer = {{ $json['Expected clean data'] }}` (from trigger node item)
      - `actualAnswer = {{ $('Evaluation11').item.json.output }}`

5. **Example 4 (Helpfulness)**
   1. Add **Evaluation Trigger** for product descriptions (Sheet1 or separate sheet).
   2. Add **Gemini Chat Model** and **AI Agent** “Product Description Rewriter”.
   3. Add **Evaluation checkIfEvaluating** → **Evaluation write outputs**.
   4. Add **OpenAI Chat Model** (`gpt-5-mini`) connected to the metrics node.
   5. Add **Evaluation setMetrics**:
      - Metric: **helpfulness**
      - `actualAnswer` from the gated output.

6. **Example 5 (Correctness)**
   1. Create/select an n8n **Data Table** (e.g., `eval_vid`) with input and reference fields.
   2. Add **Evaluation Trigger** with source = Data Tables.
   3. Add **Gemini Chat Model** and **Chain Summarization** node.
   4. Add **Evaluation checkIfEvaluating** → **Evaluation write outputs**.
   5. Add **OpenAI Chat Model** (`gpt-5-mini`) for evaluation scoring.
   6. Add **Evaluation setMetrics**:
      - Set metric explicitly to **correctness** (recommended) and map `actualAnswer` and `expectedAnswer` from your table columns.

7. **Gmail “in action” path**
   1. Add **Gmail Trigger** (poll every minute).
   2. Add **Extract from File**; connect Trigger → Extract.
   3. Add **Information Extractor** and connect Extract → Extractor.
      - Replace placeholder text with expression pointing to extracted content.
   4. Add **Gemini Chat Model** connected to Extractor.
   5. Add **Evaluation checkIfEvaluating** after extractor:
      - Evaluation output → (optional) write outputs + metrics
      - Normal output → **If** → **Gmail Send** nodes
   6. Configure the **If** node with real conditions; configure Gmail send nodes with recipients/content.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| `@[youtube](-4LXYOhQ-Z0)` | Embedded YouTube video reference (sticky note) |
| Free Skool AI/n8n Group | https://www.skool.com/data-and-ai |
| LinkedIn | https://www.linkedin.com/in/ryan-p-nolan/ |
| Twitter/X | https://x.com/RyanMattDS |
| Website | https://ryanandmattdatascience.com/ |
| Contact for help/coaching | `ryannolandata@gmail.com` (from sticky note text) |