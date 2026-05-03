Detect human vs AI text using stylometric metrics and multi‑agent LLM debate

https://n8nworkflows.xyz/workflows/detect-human-vs-ai-text-using-stylometric-metrics-and-multi-agent-llm-debate-14551


# Detect human vs AI text using stylometric metrics and multi‑agent LLM debate

# 1. Workflow Overview

This workflow, titled **AI Lie Detector: Forensic Stylometry Engine**, analyzes a user-submitted text and estimates whether it is **human-written**, **AI-generated**, or **AI-augmented/hybrid**. It combines two approaches:

- **Deterministic stylometric analysis** via a Code node that computes text metrics
- **Sequential multi-agent LLM debate** where three specialized agents review the text from different perspectives

It also includes a **secondary maintenance branch** that updates a data table of “AI fingerprint” words on a monthly schedule.

## 1.1 Chat Input & Fingerprint Loading

A public chat trigger receives free-form text from the user. The workflow then loads a stored fingerprint word list from an n8n Data Table.

## 1.2 Stylometric Metric Extraction

A Code node computes structural and lexical metrics from the input text, including sentence burstiness, type-token ratio, hapax rate, repetition, paragraph variance, and density of tracked AI-fingerprint words.

## 1.3 Sequential Forensic Debate

A short chat message informs the user that analysis has started. Then three LLM agents operate in sequence:

1. **Scanner**: judges by instinct only
2. **Forensic Analyst**: evaluates text using the computed metrics and Scanner output
3. **Devil’s Advocate**: argues the opposite of the Analyst’s conclusion

Between agents, Code nodes parse and package their JSON outputs.

## 1.4 Final Decision & User Response

A final Code node weighs agent outputs and stylometric signals to produce a verdict and confidence score. Another Code node formats the result into a readable report, which is then returned in chat.

## 1.5 Monthly Fingerprint Generator

A separate scheduled branch periodically asks an LLM to suggest new overused AI-marker words not already in the Data Table, splits them into rows, and saves them back into the table.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Chat Entry and Fingerprint Retrieval

### Overview
This block starts the interactive analysis flow. It receives text via chat and retrieves the current AI-fingerprint vocabulary list from the Data Table, which is later used for metric extraction.

### Nodes Involved
- **When chat message received**
- **Load Fingerprint List**
- **Sticky Note**
- **Sticky Note12**

### Node Details

#### When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — public chat entry point
- **Configuration choices:**
  - Public chat endpoint enabled
  - Response mode set to **responseNodes**
  - Initial greeting prompts users to paste text for classification
- **Key expressions / variables used:**
  - Output field later referenced as `{{$json.chatInput}}`
- **Input / output connections:**
  - Entry point node
  - Outputs to **Load Fingerprint List**
- **Version-specific requirements:**
  - Type version `1.4`
  - Requires chat-enabled workflow behavior in n8n
- **Edge cases / failures:**
  - Empty or extremely short user input may still pass through and degrade metric quality
  - Public endpoint may invite noisy traffic if exposed broadly
- **Sub-workflow reference:** None

#### Load Fingerprint List
- **Type / role:** `n8n-nodes-base.dataTable` — fetches rows from Data Table `AIFingerprints`
- **Configuration choices:**
  - Operation: **get**
  - Reads from Data Table named **AIFingerprints**
- **Key expressions / variables used:**
  - Output rows are expected to contain a `word` column
- **Input / output connections:**
  - Input from **When chat message received**
  - Output to **Extract Stylometric Metrics**
- **Version-specific requirements:**
  - Type version `1.1`
  - Requires Data Table support in the n8n instance
- **Edge cases / failures:**
  - Missing Data Table or wrong table ID causes retrieval failure
  - Empty table is technically valid, but weakens fingerprint detection
  - Permissions/project scoping issues can block access
- **Sub-workflow reference:** None

#### Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — documentation only
- **Configuration choices:**
  - Large explanatory note describing workflow purpose, setup, and customization
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note12
- **Type / role:** `n8n-nodes-base.stickyNote` — documentation only
- **Configuration choices:**
  - Labels the block as Input & Metrics Extraction
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

## 2.2 Block: Stylometric Metric Extraction

### Overview
This block turns the raw user text and fingerprint word list into quantitative stylometric signals. These metrics become the evidence base for the later debate and final verdict.

### Nodes Involved
- **Extract Stylometric Metrics**
- **Sticky Note2**

### Node Details

#### Extract Stylometric Metrics
- **Type / role:** `n8n-nodes-base.code` — computes lexical, syntactic, and fingerprint metrics
- **Configuration choices:**
  - Reads original text directly from the chat trigger:
    - `$('When chat message received').first().json.chatInput`
  - Reads all Data Table rows via `$input.all()`
  - Extracts lowercased fingerprint words from the `word` field
  - Computes:
    - `totalWords`
    - `totalSentences`
    - `totalParagraphs`
    - `avgSentenceLength`
    - `sentenceLengthVariance`
    - `burstiness`
    - `typeTokenRatio`
    - `hapaxRate`
    - `repetitionScore`
    - `avgParagraphLength`
    - `paragraphVariance`
    - `transitionDensity`
    - `aiFingerprintsFound`
    - `aiFingerprintCount`
- **Key expressions / variables used:**
  - `rawText`
  - `$input.all()`
  - `word`
  - regex-based sentence and paragraph splitting
- **Input / output connections:**
  - Input from **Load Fingerprint List**
  - Output to **Agent Orchestrator**
- **Version-specific requirements:**
  - Type version `2`
  - Uses n8n Code node helpers like `$input.all()` and node references by name
- **Edge cases / failures:**
  - Empty text produces valid but low-information metrics
  - Word tokenization is whitespace-based; punctuation remains attached to some words
  - Sentence splitting by `/[.!?]+/` can mis-handle abbreviations
  - Fingerprint matching is exact token-based lowercase matching; punctuation variants may be missed
  - If the chat trigger node name changes, direct node reference breaks
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Notes that the block computes stylometric metrics from raw text
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

## 2.3 Block: User Progress Message and Scanner Agent

### Overview
This block acknowledges the analysis start to the user, then launches the first LLM agent. The Scanner evaluates the text without seeing the metrics, simulating a “cold read.”

### Nodes Involved
- **Agent Orchestrator**
- **LLM Scanner**
- **Agent 1 - The Scanner**
- **Package Scanner Output**
- **Sticky Note3**

### Node Details

#### Agent Orchestrator
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — sends interim status message to chat
- **Configuration choices:**
  - Message: informs the user that three agents will debate the text and analysis takes 30–60 seconds
- **Key expressions / variables used:** None
- **Input / output connections:**
  - Input from **Extract Stylometric Metrics**
  - Output to **Agent 1 - The Scanner**
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - If chat response behavior is misconfigured, user may not receive progress updates
- **Sub-workflow reference:** None

#### LLM Scanner
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — language model backend for Scanner
- **Configuration choices:**
  - Uses model **Claude Sonnet 4.5**
  - Anthropic credentials configured
- **Key expressions / variables used:** None in this node itself
- **Input / output connections:**
  - Connected to **Agent 1 - The Scanner** via `ai_languageModel`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires valid Anthropic API credentials
- **Edge cases / failures:**
  - Authentication error
  - Model unavailability or quota limits
  - Timeout on long input
- **Sub-workflow reference:** None

#### Agent 1 - The Scanner
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — first debate agent
- **Configuration choices:**
  - Prompt instructs the model to inspect only text between START/END markers
  - Reads text via:
    - `{{ $('Extract Stylometric Metrics').first().json.originalText }}`
  - Required output: single-line JSON with:
    - `impression`
    - `confidence`
    - `gut_reasoning`
  - System prompt frames the agent as a veteran editor judging by feel alone
  - `onError: continueRegularOutput`
- **Key expressions / variables used:**
  - Direct node reference to **Extract Stylometric Metrics**
- **Input / output connections:**
  - Main input from **Agent Orchestrator**
  - LLM input from **LLM Scanner**
  - Main output to **Package Scanner Output**
- **Version-specific requirements:**
  - Type version `3.1`
  - Requires compatible LangChain agent support
- **Edge cases / failures:**
  - Model may emit malformed JSON despite prompt constraints
  - Long text may reduce formatting compliance
  - If upstream node is renamed, expression fails
  - “continueRegularOutput” allows downstream handling even when the agent fails
- **Sub-workflow reference:** None

#### Package Scanner Output
- **Type / role:** `n8n-nodes-base.code` — parses Scanner JSON and packages context
- **Configuration choices:**
  - Reads metrics from `$('Extract Stylometric Metrics').first().json`
  - Reads Scanner raw output from `.json.output` or `.json.text`
  - Strips optional Markdown code fences
  - Parses JSON; on failure, injects fallback:
    - `impression: "unknown"`
    - `confidence: 0`
    - diagnostic message in `gut_reasoning`
- **Key expressions / variables used:**
  - `$('Extract Stylometric Metrics').first().json`
  - `$input.first().json.output`
  - `$input.first().json.text`
- **Input / output connections:**
  - Input from **Agent 1 - The Scanner**
  - Output to **Agent 2 - Forensic Analyst**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases / failures:**
  - Invalid JSON from Scanner
  - Empty agent response
  - Unexpected property naming from model
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Describes the sequential forensic debate
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

## 2.4 Block: Forensic Analyst Agent

### Overview
This block gives the second agent both the Scanner’s opinion and the stylometric evidence. The goal is to produce a data-grounded forensic assessment.

### Nodes Involved
- **LLM - Analyst**
- **Agent 2 - Forensic Analyst**
- **Package Analyst Output**

### Node Details

#### LLM - Analyst
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — model backend for Analyst
- **Configuration choices:**
  - Uses Google Gemini credentials
  - No advanced options configured
- **Key expressions / variables used:** None
- **Input / output connections:**
  - Connected to **Agent 2 - Forensic Analyst** via `ai_languageModel`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires valid Google Gemini / PaLM API credential setup
- **Edge cases / failures:**
  - Authentication or billing issues
  - Token/context limits for long texts and rich prompts
- **Sub-workflow reference:** None

#### Agent 2 - Forensic Analyst
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — second debate agent
- **Configuration choices:**
  - Receives:
    - original text
    - scanner impression/confidence/reasoning
    - full stylometric metrics
  - Must return JSON with:
    - `classification`
    - `confidence`
    - `forensic_report`
    - `agrees_with_scanner`
  - System prompt emphasizes data-backed reasoning and explicit metric references
  - `onError: continueRegularOutput`
- **Key expressions / variables used:**
  - `{{ $json.originalText }}`
  - `{{ $json.scannerVerdict.* }}`
  - `{{ $json.metrics.* }}`
- **Input / output connections:**
  - Main input from **Package Scanner Output**
  - LLM input from **LLM - Analyst**
  - Main output to **Package Analyst Output**
- **Version-specific requirements:**
  - Type version `3.1`
- **Edge cases / failures:**
  - Malformed JSON output
  - Hallucinated metric references
  - Agent may ignore “return JSON only” instruction
- **Sub-workflow reference:** None

#### Package Analyst Output
- **Type / role:** `n8n-nodes-base.code` — parses Analyst JSON and preserves debate history
- **Configuration choices:**
  - Reads packaged Scanner context from `$('Package Scanner Output').first().json`
  - Reads Analyst raw output from `.json.output` or `.json.text`
  - Removes code fences if present
  - Parses JSON; on failure, returns fallback object:
    - `classification: "unknown"`
    - `confidence: 0`
    - parse error in `forensic_report`
    - `agrees_with_scanner: false`
- **Key expressions / variables used:**
  - `$('Package Scanner Output').first().json`
  - `$input.first().json.output`
  - `$input.first().json.text`
- **Input / output connections:**
  - Input from **Agent 2 - Forensic Analyst**
  - Output to **Agent 3 - Devil's Advocate**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases / failures:**
  - Invalid Analyst JSON
  - Missing fields
  - Upstream node rename breaks node lookup
- **Sub-workflow reference:** None

---

## 2.5 Block: Devil’s Advocate Agent

### Overview
This block forces adversarial review. The third agent must argue against the Analyst’s conclusion, exposing weaknesses in reasoning and interpretation.

### Nodes Involved
- **LLM - Devil's Advocate**
- **Agent 3 - Devil's Advocate**

### Node Details

#### LLM - Devil's Advocate
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — model backend for the adversarial agent
- **Configuration choices:**
  - Model set to **gpt-5-mini**
  - No built-in tools enabled
  - `onError: continueRegularOutput`
- **Key expressions / variables used:** None in this node itself
- **Input / output connections:**
  - Connected to **Agent 3 - Devil's Advocate** via `ai_languageModel`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires valid OpenAI credentials, though credentials are not embedded in the JSON excerpt
- **Edge cases / failures:**
  - Missing credentials
  - Unsupported model name depending on account availability
  - Rate limits / quota exhaustion
- **Sub-workflow reference:** None

#### Agent 3 - Devil's Advocate
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — third debate agent
- **Configuration choices:**
  - Receives:
    - original text
    - Scanner verdict
    - Analyst verdict
    - key metrics
  - Explicitly instructed to argue the opposite of the Analyst
  - Must return one-line JSON with:
    - `counter_classification`
    - `confidence`
    - `counter_argument`
    - `strongest_weakness`
  - System prompt makes the agent adversarial
  - `onError: continueRegularOutput`
- **Key expressions / variables used:**
  - `{{ $json.originalText }}`
  - `{{ $json.scannerVerdict.* }}`
  - `{{ $json.analystVerdict.* }}`
  - `{{ $json.metrics.* }}`
- **Input / output connections:**
  - Main input from **Package Analyst Output**
  - LLM input from **LLM - Devil's Advocate**
  - Main output to **Final Verdict**
- **Version-specific requirements:**
  - Type version `3.1`
- **Edge cases / failures:**
  - May still agree with Analyst despite prompt
  - May return invalid JSON or multiline content
  - Missing OpenAI credential would prevent execution
- **Sub-workflow reference:** None

---

## 2.6 Block: Final Verdict and Chat Formatting

### Overview
This block combines deterministic stylometric evidence with the three-agent debate to compute a final weighted classification and then formats the result for the user.

### Nodes Involved
- **Final Verdict**
- **Format Chat Message**
- **Send Final Report**
- **Sticky Note4**
- **Sticky Note5** (disabled)

### Node Details

#### Final Verdict
- **Type / role:** `n8n-nodes-base.code` — scoring and arbitration engine
- **Configuration choices:**
  - Reads Analyst package from `$('Package Analyst Output').first().json`
  - Reads Devil’s Advocate raw output from current input
  - Parses possible fenced JSON
  - Derives AI vs human signal counts using threshold logic:
    - burstiness `< 0.3`
    - TTR `< 0.4`
    - hapax `< 0.4`
    - repetition `> 0.15`
    - transition density `> 0.015`
  - Adds bonus AI weighting for:
    - `aiFingerprintCount`
    - high `transitionDensity`
    - short texts under 150 words
  - Default weights:
    - Analyst `0.35`
    - Scanner `0.15`
    - Devil’s Advocate `0.15`
    - Metrics `0.35`
  - Rebalances weights if Scanner is invalid
  - Strengthens metrics weight when metric AI ratio is extreme
  - Converts class labels into scores:
    - human = `0`
    - ai-augmented = `0.5`
    - ai = `1`
  - Outputs:
    - `finalVerdict`
    - `debate`
    - `metrics`
    - `metricSignals`
    - `voteCounts`
    - `originalText`
    - `failedAgents`
- **Key expressions / variables used:**
  - `$('Package Analyst Output').first().json`
  - `$input.first().json.output`
  - `$input.first().json.text`
- **Input / output connections:**
  - Input from **Agent 3 - Devil's Advocate**
  - Output to **Format Chat Message**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases / failures:**
  - Bad Devil’s Advocate output triggers fallback “unknown”
  - Thresholds are heuristic and may need tuning for domain-specific text
  - Very short or highly edited human text may be over-flagged as AI
  - Hybrid texts are reduced to a midpoint score, which may flatten nuance
- **Sub-workflow reference:** None

#### Format Chat Message
- **Type / role:** `n8n-nodes-base.code` — converts verdict object into readable chat response
- **Configuration choices:**
  - Builds verdict badge:
    - Human
    - AI
    - AI-Augmented
  - Creates 10-block confidence bar
  - Displays selected metrics and labels each as AI/Human leaning
  - Summarizes outputs from:
    - Agent 1
    - Agent 2
    - Agent 3
  - Truncates long reasoning/report fields to 200 characters
  - Returns final string as `output`
- **Key expressions / variables used:**
  - `$input.first().json`
  - `finalVerdict.classification`
  - `metrics.*`
  - `debate.*`
- **Input / output connections:**
  - Input from **Final Verdict**
  - Output to **Send Final Report**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases / failures:**
  - Missing expected keys from prior node can break formatting
  - Uses direct fixed field names; schema changes upstream require updates
- **Sub-workflow reference:** None

#### Send Final Report
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — sends final formatted analysis back to the chat session
- **Configuration choices:**
  - Message expression: `={{ $json.output }}`
- **Key expressions / variables used:**
  - `$json.output`
- **Input / output connections:**
  - Input from **Format Chat Message**
  - Terminal node for chat branch
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - If `output` is missing, response may be blank
  - Chat channel misconfiguration can suppress visible reply
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Notes that the block weighs debate chain and metrics
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels the final chat response area
  - Disabled in this workflow
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

## 2.7 Block: Monthly Fingerprint Generator

### Overview
This independent branch maintains the fingerprint word list used in stylometric analysis. It runs on a schedule, loads the current list, asks an LLM for five new overused LLM words, splits them, and saves them to the Data Table.

### Nodes Involved
- **Schedule Trigger**
- **Check Existing Words**
- **Format Existing List**
- **Find New Words**
- **LLM - Generator**
- **Split into Rows**
- **Save New Fingerprints**
- **Sticky Note1**
- **Sticky Note11**

### Node Details

#### Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — scheduled entry point for maintenance branch
- **Configuration choices:**
  - Rule interval on months
  - The sticky notes describe intended monthly execution on the 1st at midnight, but the actual node JSON shows only a month interval, so exact calendar timing is not fully encoded here
- **Key expressions / variables used:** None
- **Input / output connections:**
  - Entry point for maintenance branch
  - Outputs to **Check Existing Words**
- **Version-specific requirements:**
  - Type version `1.2`
- **Edge cases / failures:**
  - Schedule may not match the note’s described cron behavior unless manually adjusted
  - Inactive workflows do not run scheduled nodes
- **Sub-workflow reference:** None

#### Check Existing Words
- **Type / role:** `n8n-nodes-base.dataTable` — fetches current fingerprint list
- **Configuration choices:**
  - Operation: **get**
  - Reads from same **AIFingerprints** Data Table
- **Key expressions / variables used:** Expects `word` column
- **Input / output connections:**
  - Input from **Schedule Trigger**
  - Output to **Format Existing List**
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases / failures:**
  - Missing table, permission issues, empty table
- **Sub-workflow reference:** None

#### Format Existing List
- **Type / role:** `n8n-nodes-base.code` — prepares existing words for prompt injection
- **Configuration choices:**
  - Collects all `word` values from input items
  - Produces a comma-separated string in `existingWords`
- **Key expressions / variables used:**
  - `$input.all().map(item => item.json.word)`
- **Input / output connections:**
  - Input from **Check Existing Words**
  - Output to **Find New Words**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases / failures:**
  - Empty input yields empty exclusion list
  - Duplicate words are not deduplicated before prompt construction
- **Sub-workflow reference:** None

#### Find New Words
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — prompts an LLM to suggest new AI-marker words
- **Configuration choices:**
  - Prompt asks for 5 new lowercase vocabulary words overused by modern LLMs
  - Explicitly excludes current tracked words via `{{ $json.existingWords }}`
  - Requires output as comma-separated list only
- **Key expressions / variables used:**
  - `{{ $json.existingWords }}`
- **Input / output connections:**
  - Main input from **Format Existing List**
  - LLM input from **LLM - Generator**
  - Main output to **Split into Rows**
- **Version-specific requirements:**
  - Type version `1.4`
- **Edge cases / failures:**
  - LLM may return phrases, commentary, or duplicates
  - “5 words” constraint is not programmatically enforced
- **Sub-workflow reference:** None

#### LLM - Generator
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGroq` — model backend for fingerprint generation
- **Configuration choices:**
  - Model: `openai/gpt-oss-safeguard-20b`
- **Key expressions / variables used:** None
- **Input / output connections:**
  - Connected to **Find New Words** via `ai_languageModel`
- **Version-specific requirements:**
  - Type version `1`
  - Requires Groq credentials, not included in the JSON excerpt
- **Edge cases / failures:**
  - Missing credentials
  - Model availability issues
  - Prompt noncompliance
- **Sub-workflow reference:** None

#### Split into Rows
- **Type / role:** `n8n-nodes-base.code` — normalizes comma-separated LLM output into Data Table rows
- **Configuration choices:**
  - Reads response from `.json.text` or `.json.output`
  - Splits by comma
  - Trims, lowercases, filters empties
  - Emits one item per word as `{ word }`
- **Key expressions / variables used:**
  - `$input.first().json.text`
  - `$input.first().json.output`
- **Input / output connections:**
  - Input from **Find New Words**
  - Output to **Save New Fingerprints**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases / failures:**
  - Comma-separated phrases may create poor entries
  - Duplicate words are not removed
  - Explanatory text from LLM may pollute rows
- **Sub-workflow reference:** None

#### Save New Fingerprints
- **Type / role:** `n8n-nodes-base.dataTable` — saves generated words back into the Data Table
- **Configuration choices:**
  - Writes to **AIFingerprints**
  - Auto-maps input data
  - Matching column: `word`
  - Single schema field:
    - `word` as string
- **Key expressions / variables used:** Uses incoming `word` field
- **Input / output connections:**
  - Input from **Split into Rows**
  - Terminal node for maintenance branch
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases / failures:**
  - Exact behavior on duplicates depends on Data Table semantics with matching column
  - Invalid table schema causes write errors
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Detailed note documenting the generator section, schedule intent, customization, and first-time setup
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note11
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Labels monthly auto-update generator branch
- **Input / output connections:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger` | Public chat entry point for text analysis |  | Load Fingerprint List | ## Input & Metrics Extraction<br>Chat trigger receives text → Loads fingerprints from data table → Extracts stylometric signals |
| Load Fingerprint List | `n8n-nodes-base.dataTable` | Loads AI fingerprint words from Data Table | When chat message received | Extract Stylometric Metrics | ## Input & Metrics Extraction<br>Chat trigger receives text → Loads fingerprints from data table → Extracts stylometric signals |
| Extract Stylometric Metrics | `n8n-nodes-base.code` | Computes stylometric and fingerprint metrics | Load Fingerprint List | Agent Orchestrator | ## Metrics Extraction<br>Computes burstiness, vocabulary density, repetition, sentence variance from raw text<br>## Input & Metrics Extraction<br>Chat trigger receives text → Loads fingerprints from data table → Extracts stylometric signals |
| Agent Orchestrator | `@n8n/n8n-nodes-langchain.chat` | Sends progress update to chat before debate begins | Extract Stylometric Metrics | Agent 1 - The Scanner | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| LLM Scanner | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Anthropic model backend for Scanner agent |  | Agent 1 - The Scanner | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| Agent 1 - The Scanner | `@n8n/n8n-nodes-langchain.agent` | Gut-check classifier based on reading instinct | Agent Orchestrator | Package Scanner Output | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| Package Scanner Output | `n8n-nodes-base.code` | Parses Scanner JSON and packages context | Agent 1 - The Scanner | Agent 2 - Forensic Analyst | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| LLM - Analyst | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Gemini model backend for Analyst agent |  | Agent 2 - Forensic Analyst | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| Agent 2 - Forensic Analyst | `@n8n/n8n-nodes-langchain.agent` | Data-driven forensic classification agent | Package Scanner Output | Package Analyst Output | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| Package Analyst Output | `n8n-nodes-base.code` | Parses Analyst JSON and preserves debate state | Agent 2 - Forensic Analyst | Agent 3 - Devil's Advocate | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| LLM - Devil's Advocate | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | OpenAI model backend for adversarial agent |  | Agent 3 - Devil's Advocate | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| Agent 3 - Devil's Advocate | `@n8n/n8n-nodes-langchain.agent` | Oppositional agent challenging Analyst’s conclusion | Package Analyst Output | Final Verdict | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| Final Verdict | `n8n-nodes-base.code` | Weighs metrics and agent outputs into final classification | Agent 3 - Devil's Advocate | Format Chat Message | ## Final Verdict<br>Weighs the full debate chain and metrics to produce the classification |
| Format Chat Message | `n8n-nodes-base.code` | Formats verdict and debate summary into chat-ready text | Final Verdict | Send Final Report | ## Final Verdict<br>Weighs the full debate chain and metrics to produce the classification<br>## Chat Response<br>Formats the forensic report and sends it to the user |
| Send Final Report | `@n8n/n8n-nodes-langchain.chat` | Returns final formatted report to user chat | Format Chat Message |  | ## Chat Response<br>Formats the forensic report and sends it to the user |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Monthly trigger for fingerprint auto-update branch |  | Check Existing Words | ##  Fingerprint Generator Section<br>**Runs:** Monthly (1st at midnight) OR manual trigger<br><br>**Purpose:** Keeps detection current by asking an LLM to generate fresh AI fingerprint words based on latest model patterns (GPT-4o, Claude 3.5+, Gemini 1.5+, Llama 3.3+).<br><br>**Flow:**<br>1. Schedule Trigger → Get current date<br>2. Load existing fingerprints from data table<br>3. Ask LLM: "What new AI markers emerged this month?"<br>4. Compare with existing words (avoid duplicates)<br>5. Insert new words into data table with categories + weights<br><br>**Customization:**<br>- Change schedule: Edit cron expression (default: `0 0 1 * *`)<br>- Adjust prompt: Edit "Find New Words" node<br>- Different LLM: Swap "LLM - Generator" node<br><br>**First-Time Setup:**<br>Run this section ONCE manually to populate initial data table with ~80-100 fingerprint words.<br>## Generator (Monthly Auto-Update)<br>Runs 1st of each month: Loads existing words → Asks LLM for new AI fingerprints → Saves to data table<br><br>**First run:** Populates initial 80-100 fingerprint words |
| Check Existing Words | `n8n-nodes-base.dataTable` | Loads current fingerprint words for exclusion prompt | Schedule Trigger | Format Existing List | ## Generator (Monthly Auto-Update)<br>Runs 1st of each month: Loads existing words → Asks LLM for new AI fingerprints → Saves to data table<br><br>**First run:** Populates initial 80-100 fingerprint words |
| Format Existing List | `n8n-nodes-base.code` | Joins existing words into comma-separated string | Check Existing Words | Find New Words | ## Generator (Monthly Auto-Update)<br>Runs 1st of each month: Loads existing words → Asks LLM for new AI fingerprints → Saves to data table<br><br>**First run:** Populates initial 80-100 fingerprint words |
| Find New Words | `@n8n/n8n-nodes-langchain.chainLlm` | Asks LLM for five new fingerprint words | Format Existing List | Split into Rows | ## Generator (Monthly Auto-Update)<br>Runs 1st of each month: Loads existing words → Asks LLM for new AI fingerprints → Saves to data table<br><br>**First run:** Populates initial 80-100 fingerprint words |
| LLM - Generator | `@n8n/n8n-nodes-langchain.lmChatGroq` | Groq model backend for word generation |  | Find New Words | ## Generator (Monthly Auto-Update)<br>Runs 1st of each month: Loads existing words → Asks LLM for new AI fingerprints → Saves to data table<br><br>**First run:** Populates initial 80-100 fingerprint words |
| Split into Rows | `n8n-nodes-base.code` | Splits generated comma-separated words into rows | Find New Words | Save New Fingerprints | ## Generator (Monthly Auto-Update)<br>Runs 1st of each month: Loads existing words → Asks LLM for new AI fingerprints → Saves to data table<br><br>**First run:** Populates initial 80-100 fingerprint words |
| Save New Fingerprints | `n8n-nodes-base.dataTable` | Stores newly generated fingerprint words | Split into Rows |  | ## Generator (Monthly Auto-Update)<br>Runs 1st of each month: Loads existing words → Asks LLM for new AI fingerprints → Saves to data table<br><br>**First run:** Populates initial 80-100 fingerprint words |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation / overview note |  |  |  |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation for generator branch |  |  |  |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation for metrics block |  |  |  |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation for debate block |  |  |  |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation for final verdict block |  |  |  |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Disabled documentation note for chat response block |  |  |  |
| Sticky Note11 | `n8n-nodes-base.stickyNote` | Documentation for auto-update generator branch |  |  |  |
| Sticky Note12 | `n8n-nodes-base.stickyNote` | Documentation for input/metrics block |  |  |  |
| 92bec431-ff76-4ecb-a1a1-3c979e56f3cd / displayed as LLM - Analyst | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Included above under visible node name |  | Agent 2 - Forensic Analyst | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| c65987f4-612d-4699-a982-99cde99d1158 / displayed as LLM Scanner | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Included above under visible node name |  | Agent 1 - The Scanner | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |
| 87687ccf-8cbc-4377-8a40-1070d8697dc4 / displayed as LLM - Devil's Advocate | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Included above under visible node name |  | Agent 3 - Devil's Advocate | ## Sequential Forensic Debate<br>Three specialists analyze the text in sequence, each building on the previous agent's output |

> Note: The last three rows are logically duplicates of already listed visible nodes. In practical use, rely on the visible node names above.

---

# 4. Reproducing the Workflow from Scratch

## A. Prepare prerequisites

1. **Create a new workflow** in n8n.
2. **Create a Data Table** named something like **AIFingerprints** with at least one column:
   - `word` — string
3. **Set up credentials** for the LLM providers you want to use:
   - **Anthropic API** for the Scanner model
   - **Google Gemini / PaLM API** for the Analyst model
   - **OpenAI API** for the Devil’s Advocate model
   - **Groq API** for the fingerprint generator branch
4. Keep the workflow **inactive** until all credentials and the Data Table are ready.

---

## B. Build the interactive text-analysis branch

### 1. Add the chat trigger
1. Create a **Chat Trigger** node.
2. Name it **When chat message received**.
3. Set it to **public**.
4. In options, set **response mode** to **responseNodes**.
5. Add an initial message such as:
   - “Hi there! 👋 Paste any text and I'll analyze whether it was written by a human, AI, or a mix of both.”

### 2. Add the fingerprint loader
6. Create a **Data Table** node.
7. Name it **Load Fingerprint List**.
8. Set **Operation** to **Get**.
9. Select the **AIFingerprints** table.
10. Connect:
   - **When chat message received → Load Fingerprint List**

### 3. Add the stylometric extractor
11. Create a **Code** node.
12. Name it **Extract Stylometric Metrics**.
13. Connect:
   - **Load Fingerprint List → Extract Stylometric Metrics**
14. Paste logic that:
   - reads chat text from `$('When chat message received').first().json.chatInput`
   - reads all Data Table rows from `$input.all()`
   - extracts `word` values
   - computes stylometric metrics
15. Ensure the node returns:
   - `originalText`
   - `metrics`
16. Include at minimum these metric fields:
   - `totalWords`
   - `totalSentences`
   - `totalParagraphs`
   - `avgSentenceLength`
   - `sentenceLengthVariance`
   - `burstiness`
   - `typeTokenRatio`
   - `hapaxRate`
   - `repetitionScore`
   - `avgParagraphLength`
   - `paragraphVariance`
   - `transitionDensity`
   - `aiFingerprintsFound`
   - `aiFingerprintCount`

### 4. Add the progress chat message
17. Create a **Chat** node.
18. Name it **Agent Orchestrator**.
19. Set message to something like:
   - “🔍 Analyzing your text... Three specialist agents are about to debate it. This takes 30-60 seconds.”
20. Connect:
   - **Extract Stylometric Metrics → Agent Orchestrator**

---

## C. Build Agent 1: Scanner

### 5. Add the Anthropic model node
21. Create an **Anthropic Chat Model** node.
22. Name it **LLM Scanner**.
23. Select a model similar to **Claude Sonnet 4.5**.
24. Attach valid **Anthropic credentials**.

### 6. Add the Scanner agent
25. Create a **LangChain Agent** node.
26. Name it **Agent 1 - The Scanner**.
27. Set **Prompt Type** to **Define**.
28. Enable **On Error → Continue Regular Output**.
29. Use a prompt that:
   - injects only the original text
   - asks for instinct-based classification
   - requires JSON-only output
30. Use a system message similar to:
   - “You are a veteran editor with 20 years of experience reading manuscripts. You can spot AI-generated text by feel alone. Trust your instincts. Return valid JSON only.”
31. Connect:
   - **Agent Orchestrator → Agent 1 - The Scanner**
   - **LLM Scanner → Agent 1 - The Scanner** using the `ai_languageModel` connection
32. The agent should return:
   - `impression`
   - `confidence`
   - `gut_reasoning`

### 7. Package Scanner output
33. Create a **Code** node.
34. Name it **Package Scanner Output**.
35. Connect:
   - **Agent 1 - The Scanner → Package Scanner Output**
36. In code:
   - get metrics from **Extract Stylometric Metrics**
   - read raw agent output from `output` or `text`
   - remove optional ```json fences
   - try `JSON.parse`
   - on parse failure, set fallback values
37. Return:
   - `originalText`
   - `metrics`
   - `scannerVerdict`

---

## D. Build Agent 2: Forensic Analyst

### 8. Add the Gemini model node
38. Create a **Google Gemini Chat Model** node.
39. Name it **LLM - Analyst**.
40. Add valid **Google Gemini / PaLM credentials**.

### 9. Add the Analyst agent
41. Create another **LangChain Agent** node.
42. Name it **Agent 2 - Forensic Analyst**.
43. Set **Prompt Type** to **Define**.
44. Enable **On Error → Continue Regular Output**.
45. Use a prompt that includes:
   - original text
   - Scanner classification/confidence/reasoning
   - all stylometric metrics
46. Require JSON-only output with:
   - `classification`
   - `confidence`
   - `forensic_report`
   - `agrees_with_scanner`
47. Use a system message emphasizing metric-based forensic reasoning.
48. Connect:
   - **Package Scanner Output → Agent 2 - Forensic Analyst**
   - **LLM - Analyst → Agent 2 - Forensic Analyst** via `ai_languageModel`

### 10. Package Analyst output
49. Create a **Code** node.
50. Name it **Package Analyst Output**.
51. Connect:
   - **Agent 2 - Forensic Analyst → Package Analyst Output**
52. In code:
   - read packaged Scanner context from **Package Scanner Output**
   - parse Analyst JSON from raw output/text
   - apply fallback values if parsing fails
53. Return:
   - `originalText`
   - `metrics`
   - `scannerVerdict`
   - `analystVerdict`

---

## E. Build Agent 3: Devil’s Advocate

### 11. Add the OpenAI model node
54. Create an **OpenAI Chat Model** node.
55. Name it **LLM - Devil's Advocate**.
56. Select a model such as **gpt-5-mini** if available in your account.
57. Add valid **OpenAI credentials**.
58. Optionally set node-level error handling to continue if supported.

### 12. Add the Devil’s Advocate agent
59. Create another **LangChain Agent** node.
60. Name it **Agent 3 - Devil's Advocate**.
61. Set **Prompt Type** to **Define**.
62. Enable **On Error → Continue Regular Output**.
63. Create a prompt that includes:
   - original text
   - Scanner output
   - Analyst output
   - metrics
64. Explicitly instruct it to argue the **opposite** of the Analyst.
65. Require JSON-only output with:
   - `counter_classification`
   - `confidence`
   - `counter_argument`
   - `strongest_weakness`
66. Use a system message like:
   - “You are a hostile cross-examiner. Your job is to destroy the previous analysis. Do not agree with the Forensic Analyst under any circumstances. Find every possible flaw. Return valid JSON only.”
67. Connect:
   - **Package Analyst Output → Agent 3 - Devil's Advocate**
   - **LLM - Devil's Advocate → Agent 3 - Devil's Advocate** via `ai_languageModel`

---

## F. Build the verdict and response block

### 13. Add the final scoring engine
68. Create a **Code** node.
69. Name it **Final Verdict**.
70. Connect:
   - **Agent 3 - Devil's Advocate → Final Verdict**
71. Implement logic that:
   - parses Devil’s Advocate output
   - loads Scanner and Analyst packaged results
   - counts AI vs human metric signals
   - boosts AI score based on fingerprint count and transition density
   - applies short-text adjustments under 150 words
   - converts classifications to numeric scores
   - combines:
     - Scanner
     - Analyst
     - Devil’s Advocate
     - metrics
   - returns final classification and confidence
72. Use thresholds matching the original design if you want identical behavior:
   - burstiness `< 0.3` = AI signal
   - type-token ratio `< 0.4` = AI signal
   - hapax rate `< 0.4` = AI signal
   - repetition score `> 0.15` = AI signal
   - transition density `> 0.015` = AI signal
73. Output at least:
   - `finalVerdict`
   - `debate`
   - `metrics`
   - `metricSignals`
   - `voteCounts`
   - `originalText`
   - `failedAgents`

### 14. Add the formatter
74. Create a **Code** node.
75. Name it **Format Chat Message**.
76. Connect:
   - **Final Verdict → Format Chat Message**
77. Build a markdown-friendly output string that includes:
   - verdict badge
   - confidence bar
   - metric summary
   - brief excerpts from all three agent outputs
78. Return:
   - `output`

### 15. Add the final chat response
79. Create a **Chat** node.
80. Name it **Send Final Report**.
81. Set message to:
   - `={{ $json.output }}`
82. Connect:
   - **Format Chat Message → Send Final Report**

---

## G. Build the monthly fingerprint-generator branch

### 16. Add the schedule trigger
83. Create a **Schedule Trigger** node.
84. Name it **Schedule Trigger**.
85. Configure it for monthly execution.
86. If you want to match the note’s intention precisely, use a cron or schedule equivalent for:
   - **1st day of each month at 00:00**
87. This branch is independent from the chat branch.

### 17. Load existing tracked words
88. Create a **Data Table** node.
89. Name it **Check Existing Words**.
90. Set **Operation** to **Get**.
91. Select the same **AIFingerprints** table.
92. Connect:
   - **Schedule Trigger → Check Existing Words**

### 18. Format the exclusion list
93. Create a **Code** node.
94. Name it **Format Existing List**.
95. Connect:
   - **Check Existing Words → Format Existing List**
96. In code:
   - collect all `word` values
   - join them with commas
97. Return:
   - `existingWords`

### 19. Add the Groq model node
98. Create a **Groq Chat Model** node.
99. Name it **LLM - Generator**.
100. Select model:
   - `openai/gpt-oss-safeguard-20b`
101. Add valid **Groq credentials**.

### 20. Ask for new fingerprint words
102. Create a **Chain LLM** node.
103. Name it **Find New Words**.
104. Set **Prompt Type** to **Define**.
105. Use a prompt that:
   - asks for 5 new words overused by modern LLMs
   - excludes words from `{{ $json.existingWords }}`
   - requires a lowercase comma-separated list only
106. Connect:
   - **Format Existing List → Find New Words**
   - **LLM - Generator → Find New Words** via `ai_languageModel`

### 21. Split the generated list into rows
107. Create a **Code** node.
108. Name it **Split into Rows**.
109. Connect:
   - **Find New Words → Split into Rows**
110. In code:
   - read response from `text` or `output`
   - split by commas
   - trim and lowercase words
   - emit one item per word as `{ word }`

### 22. Save new words
111. Create a **Data Table** node.
112. Name it **Save New Fingerprints**.
113. Connect:
   - **Split into Rows → Save New Fingerprints**
114. Select the **AIFingerprints** table.
115. Configure the schema mapping so input field `word` maps to Data Table column `word`.
116. If available, set matching column to:
   - `word`
117. This helps avoid duplicate insert behavior depending on Data Table semantics.

---

## H. Add documentation notes

118. Add optional **Sticky Note** nodes to document:
   - overall workflow purpose
   - input & extraction block
   - sequential debate block
   - final verdict block
   - generator branch
119. These notes are not required for execution but improve maintainability.

---

## I. Final validation checklist

120. Confirm both entry points exist:
   - **When chat message received**
   - **Schedule Trigger**
121. Confirm every LLM agent has an attached model node.
122. Confirm the Data Table exists and has a `word` column.
123. Test the analysis branch with:
   - clearly human prose
   - clearly AI-formatted text
   - short ambiguous text
124. Test the generator branch manually once to seed the table.
125. Activate the workflow only after all credentials and Data Table references are valid.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Lie Detector: Forensic Stylometry Engine — paste any text and three specialist agents debate whether it was written by a human, AI, or a mix of both. | Overall workflow purpose |
| Setup note: add API credentials for the LLM providers you want to use, activate the workflow, open the chat window, paste text, and wait for forensic analysis. | General setup guidance |
| Customization note: swap any LLM for another, adjust metric thresholds in the Extract Stylometric Metrics node, and modify agent personas in their system prompts. | General customization guidance |
| Fingerprint generator note: intended to run monthly and keep detection current by refreshing tracked AI-marker words. | Maintenance branch |
| First-time setup note: manually run the generator branch once to populate the fingerprint table with an initial vocabulary list. | Maintenance branch initialization |
| Important implementation note: the sticky note describes a cron-like monthly schedule (`0 0 1 * *`), but the actual Schedule Trigger configuration in the JSON is only a generic monthly interval. | Schedule consistency check |
| Important implementation note: the workflow is currently inactive (`active: false`). | Deployment state |
| Execution timeout is set to 180 seconds. | Workflow settings |
| Caller policy is `workflowsFromSameOwner`; no sub-workflows are used in this design. | Workflow settings / architecture |

## Additional implementation cautions
- The workflow relies heavily on **node-name-based references** inside Code and Agent nodes. Renaming nodes without updating expressions will break execution.
- LLM outputs are expected to be **strict JSON**, but the workflow already includes defensive parsing and fallback handling.
- The “AI fingerprint” mechanism depends on exact lowercase token matching and is relatively simple; adding punctuation normalization or stemming would improve robustness.
- The final verdict logic is **heuristic**, not statistically calibrated. Threshold tuning may be necessary for legal, academic, or domain-specific writing.