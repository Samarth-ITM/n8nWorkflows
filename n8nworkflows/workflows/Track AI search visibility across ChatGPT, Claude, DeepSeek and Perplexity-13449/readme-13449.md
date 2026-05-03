Track AI search visibility across ChatGPT, Claude, DeepSeek and Perplexity

https://n8nworkflows.xyz/workflows/track-ai-search-visibility-across-chatgpt--claude--deepseek-and-perplexity-13449


# Track AI search visibility across ChatGPT, Claude, DeepSeek and Perplexity

## 1. Workflow Overview

**Workflow name:** `AI_Ranking_Checker`  
**Title provided:** Track AI search visibility across ChatGPT, Claude, DeepSeek and Perplexity

This workflow measures a client website’s **AI-search discoverability** (“ASEO”) by generating a **brand-neutral search query** from a website summary, running that same query across **four AI platforms** (ChatGPT/OpenAI, Claude/Anthropic, DeepSeek, Perplexity), and then using an LLM to **score visibility** (ranking presence + strength %) and **extract competitors**, returning a **structured 27-field output** for downstream reporting.

### 1.1 Input Reception (Parent → Child workflow)
Receives the website URL and a short summary from a parent workflow via an Execute Workflow Trigger.

### 1.2 Prompt Generation + Parsing
Uses GPT-4.1-mini to generate a single brand-neutral search prompt, then parses it into structured JSON.

### 1.3 Multi-platform Visibility Tests (4 platforms)
Runs the generated prompt through:
- OpenAI (as a proxy for “ChatGPT” style result)
- Claude (Anthropic)
- DeepSeek
- Perplexity  
Some nodes are configured to continue on errors to avoid failing the whole run.

### 1.4 Cross-platform Analysis + Structured Output
Aggregates all platform outputs into one analysis prompt, uses DeepSeek to generate a structured assessment, parses it, then flattens to 27 fields and ends.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Receives runtime inputs from a parent workflow (website URL + summary). This workflow is designed to be called as a sub-workflow rather than run on a schedule/webhook.

**Nodes involved:**
- Receive Website and Summary from Parent

#### Node: Receive Website and Summary from Parent
- **Type / role:** `Execute Workflow Trigger` (`n8n-nodes-base.executeWorkflowTrigger`) — entry point when invoked by a parent “Execute Workflow” node.
- **Key configuration:**
  - Defines two expected inputs:
    - `Website`
    - `Website Summary`
- **Inputs/outputs:**
  - **Input:** Provided by parent workflow execution.
  - **Output:** JSON object containing `Website` and `Website Summary`.
- **Edge cases / failure types:**
  - Missing/empty inputs → downstream prompt generation may produce low-quality or invalid structured JSON.
  - Website summary too short/vague → prompt may be generic, reducing signal in rankings.
- **Version notes:** TypeVersion `1.1`.

---

### Block 2 — Prompt Generation + Structured Parsing
**Overview:** Generates a single “brand-neutral” search query based on the client’s website and summary, then parses it into JSON (`output.Prompts`) for reliable reuse.

**Nodes involved:**
- Generate Brand-Neutral Search Prompts
- GPT Model for Prompt Generation
- Parse Prompt as JSON
- GPT Model for Parser Support

#### Node: Generate Brand-Neutral Search Prompts
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — LLM-driven text generation.
- **Key configuration choices:**
  - Prompt instructs: generate a relevant prompt users might use to find companies like the website **without using client name**, focusing on core services.
  - Uses expressions:
    - `{{ $json.Website }}`
    - `{{ $json["Website Summary"] }}`
  - `hasOutputParser: true` meaning it expects structured parsing downstream.
- **Inputs/outputs:**
  - **Input:** From “Receive Website and Summary from Parent”.
  - **Output:** Agent output expected to be parsed into an object with `Prompts`.
- **Connections:**
  - Uses **GPT Model for Prompt Generation** as its `ai_languageModel`.
  - Uses **Parse Prompt as JSON** as its `ai_outputParser`.
  - Main output goes to “Test Visibility on ChatGPT”.
- **Edge cases:**
  - Model returns multiple prompts or includes the brand name despite instruction → parsing may still succeed but violates intent.
  - If the agent output isn’t parseable, the structured parser may attempt auto-fix (enabled), but can still fail.
- **Version notes:** TypeVersion `2`.

#### Node: GPT Model for Prompt Generation
- **Type / role:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the model for the prompt-generation agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - Uses OpenAI credentials: `testing`
- **Connections:** Feeds as `ai_languageModel` into “Generate Brand-Neutral Search Prompts”.
- **Failure types:**
  - Auth/credit issues on OpenAI key
  - Rate limits/timeouts
- **Version notes:** TypeVersion `1.2`.

#### Node: Parse Prompt as JSON
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces a JSON shape for the generated prompt.
- **Key configuration:**
  - `jsonSchemaExample`:
    ```json
    { "Prompts": "Prompt" }
    ```
  - `autoFix: true` + `customizeRetryPrompt: true` (parser will attempt to repair malformed outputs)
- **Connections:**
  - Takes `ai_languageModel` from “GPT Model for Parser Support”.
  - Feeds `ai_outputParser` into “Generate Brand-Neutral Search Prompts”.
- **Failure types:**
  - Output too malformed to repair → agent node can error.
  - Schema mismatch (e.g., key name differs) → downstream expressions may break.
- **Version notes:** TypeVersion `1.3`.

#### Node: GPT Model for Parser Support
- **Type / role:** OpenAI Chat Model — used specifically by the output parser to fix/normalize JSON.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - Credentials: `testing`
- **Connections:** Supplies `ai_languageModel` to “Parse Prompt as JSON”.
- **Failure types:** Same as other OpenAI nodes (auth, rate limits).
- **Version notes:** TypeVersion `1.2`.

---

### Block 3 — Platform Tests (ChatGPT/OpenAI, Claude, DeepSeek, Perplexity)
**Overview:** Uses the generated prompt to query four platforms sequentially (in the workflow graph) while some nodes continue on error so analysis can still run.

**Nodes involved:**
- Test Visibility on ChatGPT
- GPT-4o-mini for ChatGPT Test
- Test Visibility on Claude
- Claude Sonnet 3.7 Model
- Test Visibility on DeepSeek
- DeepSeek Model for Testing
- Test Visibility on Perplexity

#### Node: Test Visibility on ChatGPT
- **Type / role:** LangChain Agent — runs the generated prompt to produce a “ChatGPT output”.
- **Key configuration:**
  - Text: `={{ $json.output.Prompts }}`
    - Assumes the upstream agent+parser produced `output.Prompts`.
- **Connections:**
  - `ai_languageModel` from “GPT-4o-mini for ChatGPT Test”.
  - Main output → “Test Visibility on Claude”.
- **Failure types / edge cases:**
  - If `output.Prompts` missing → expression resolves to empty/undefined, causing weak output or failure.
  - Output format is unconstrained (no structured parser), so downstream analysis must handle variability.
- **Version notes:** TypeVersion `2`.

#### Node: GPT-4o-mini for ChatGPT Test
- **Type / role:** OpenAI Chat Model — powers the ChatGPT visibility test.
- **Key configuration:**
  - Model: `gpt-4o-mini`
  - Credentials: `Open AI - Misc for testing`
- **Connections:** `ai_languageModel` → “Test Visibility on ChatGPT”.
- **Failure types:** OpenAI auth/rate limits/timeouts.
- **Version notes:** TypeVersion `1.2`.

#### Node: Test Visibility on Claude
- **Type / role:** LangChain Agent — queries Claude using the same prompt.
- **Key configuration:**
  - Text: `={{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
    - Uses a *node reference* rather than current `$json`.
  - `onError: continueRegularOutput` (important): workflow continues even if Claude fails.
  - `alwaysOutputData: false` (so if it errors, output may be missing/empty depending on n8n behavior and the specific failure path).
- **Connections:**
  - `ai_languageModel` from “Claude Sonnet 3.7 Model”.
  - Main output → “Test Visibility on DeepSeek”.
- **Edge cases:**
  - If Claude errors, downstream analysis expression `$('Test Visibility on Claude').item.json.output` may be missing → analysis agent may fail unless it tolerates missing values.
- **Version notes:** TypeVersion `2`.

#### Node: Claude Sonnet 3.7 Model
- **Type / role:** Anthropic chat model node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`).
- **Key configuration:**
  - Model: `claude-3-7-sonnet-20250219`
  - Credentials: `testing_account`
- **Connections:** `ai_languageModel` → “Test Visibility on Claude”.
- **Failure types:** Anthropic key issues, rate limits, model availability.
- **Version notes:** TypeVersion `1.3`.

#### Node: Test Visibility on DeepSeek
- **Type / role:** LangChain Agent — queries DeepSeek with the same prompt.
- **Key configuration:**
  - Text: `={{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
  - `onError: continueRegularOutput`
  - `alwaysOutputData: false`
- **Connections:**
  - `ai_languageModel` from “DeepSeek Model for Testing”.
  - Main output → “Test Visibility on Perplexity”.
- **Edge cases:**
  - Same missing-output risk as Claude when error occurs.
- **Version notes:** TypeVersion `2`.

#### Node: DeepSeek Model for Testing
- **Type / role:** DeepSeek chat model (`@n8n/n8n-nodes-langchain.lmChatDeepSeek`).
- **Key configuration:**
  - Uses DeepSeek credentials: `testing_account`
- **Connections:** `ai_languageModel` → “Test Visibility on DeepSeek”.
- **Failure types:** Credential issues, endpoint downtime.
- **Version notes:** TypeVersion `1`.

#### Node: Test Visibility on Perplexity
- **Type / role:** Perplexity node (`n8n-nodes-base.perplexity`) — sends a chat-style request to Perplexity.
- **Key configuration:**
  - `onError: continueRegularOutput`
  - Message content: `={{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
- **Inputs/outputs:**
  - Output is used later via: `{{ $json.choices[0].message.content }}`
- **Connections:** Main output → “Analyze All Platform Results”.
- **Edge cases:**
  - If Perplexity errors or returns a different response shape, `choices[0].message.content` may not exist.
- **Version notes:** TypeVersion `1`.

---

### Block 4 — Cross-Platform Analysis + JSON Parsing + Flattening
**Overview:** Combines the website URL, the generated prompt, and each platform’s output into a single analysis request. DeepSeek generates a structured scorecard; a structured parser enforces schema; the result is flattened into 27 fields for easy reporting/export.

**Nodes involved:**
- Analyze All Platform Results
- DeepSeek Model for Analysis
- Parse Analysis as Structured JSON
- Flatten JSON to 27 Data Fields
- Output Data Complete

#### Node: Analyze All Platform Results
- **Type / role:** LangChain Agent — synthesizes and scores visibility across platforms.
- **Key configuration:**
  - Prompt includes:
    - Client website: `{{ $('Receive Website and Summary from Parent').item.json.Website }}`
    - Prompt used: `{{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
    - ChatGPT output: `{{ $('Test Visibility on ChatGPT').item.json.output }}`
    - Claude output: `{{ $('Test Visibility on Claude').item.json.output }}`
    - DeepSeek output: `{{ $('Test Visibility on DeepSeek').item.json.output }}`
    - Perplexity output: `{{ $json.choices[0].message.content }}`
  - Requests: visibility yes/no, position, and a **0–100% presence strength**, plus competitors and recommendations.
  - `hasOutputParser: true` (expects structured JSON).
- **Connections:**
  - `ai_languageModel` from “DeepSeek Model for Analysis”.
  - `ai_outputParser` from “Parse Analysis as Structured JSON”.
  - Main output → “Flatten JSON to 27 Data Fields”.
- **Critical edge cases:**
  - If Claude/DeepSeek/Perplexity steps fail and produce no usable output, the expressions referencing `.item.json.output` or `.choices[0]...` can throw expression errors or produce `undefined`. Consider adding fallback text like “(no result due to error)” to keep analysis stable.
- **Version notes:** TypeVersion `2`.

#### Node: DeepSeek Model for Analysis
- **Type / role:** DeepSeek chat model node — powers the cross-platform analysis.
- **Key configuration:**
  - Credentials: `Deepseek - Upwork Covers`
- **Connections:** `ai_languageModel` → “Analyze All Platform Results”.
- **Failure types:** Credential/rate limits/outages.
- **Version notes:** TypeVersion `1`.

#### Node: Parse Analysis as Structured JSON
- **Type / role:** Structured Output Parser — enforces a detailed schema for the analysis output.
- **Key configuration:**
  - Large `jsonSchemaExample` containing:
    - `client_website`, `search_query`
    - `chatgpt_analysis`, `claude_analysis`, `deepseek_analysis`, `perplexity_analysis`
    - `overall_summary` (platform counts, averages, strongest/weakest, competitors, recommendations)
- **Connections:** Provides `ai_outputParser` to “Analyze All Platform Results”.
- **Failure types:**
  - Model returns non-conforming fields; parser may fail unless auto-fix is enabled (note: autoFix is **not** shown enabled here, unlike prompt parser).
- **Version notes:** TypeVersion `1.2`.

#### Node: Flatten JSON to 27 Data Fields
- **Type / role:** Set node (`n8n-nodes-base.set`) — maps nested structured analysis into flat reporting fields.
- **Key configuration:**
  - Creates 27 output keys, including:
    - ChatGPT/Claude/DeepSeek/Perplexity: Ranking, Position, Presence Strength, Key Mentions, Competitors
    - Overall: Ranking count, average strength, strongest/weakest platform, main competitors, recommendations
  - Uses expressions like:
    - `={{ $json.output.chatgpt_analysis.is_ranking }}`
    - `={{ $json.output.overall_summary.recommendations }}`
  - **Potential bug:** Field `Prompt` is set as:
    - `={{ $('Receive Website and Summary from Parent').item.json.Prompt }}`
    - But the trigger inputs are `Website` and `Website Summary` (no `Prompt`). Likely intended to reference:
      - `$('Generate Brand-Neutral Search Prompts').item.json.output.Prompts`
- **Connections:** Main output → “Output Data Complete”.
- **Failure types:**
  - If parser output isn’t in `$json.output...` shape, many fields will be null/undefined.
  - Typos in field names:
    - `Calude: Presence Strength` (spelling)
    - `Weekest Platform` (spelling)
    - `Overall: Main Comeptitors` (spelling)
    These won’t break execution but may confuse downstream consumers relying on exact column names.
- **Version notes:** TypeVersion `3.4`.

#### Node: Output Data Complete
- **Type / role:** NoOp (`n8n-nodes-base.noOp`) — marks the end; useful as a terminal output anchor.
- **Connections:** Receives from “Flatten JSON to 27 Data Fields”.
- **Version notes:** TypeVersion `1`.

---

### Block 5 — Workflow Documentation Note (Sticky Note)
**Overview:** A sticky note describes purpose, steps, and credential setup expectations. It does not affect execution.

**Nodes involved:**
- Sticky Note

#### Node: Sticky Note
- **Type / role:** Sticky note (`n8n-nodes-base.stickyNote`) — in-canvas documentation.
- **Content:** (see Summary Table; duplicated there per node as required)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Website and Summary from Parent | Execute Workflow Trigger | Receives `Website` and `Website Summary` from parent workflow | — | Generate Brand-Neutral Search Prompts | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Generate Brand-Neutral Search Prompts | LangChain Agent | Creates one brand-neutral search prompt from website + summary | Receive Website and Summary from Parent | Test Visibility on ChatGPT | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| GPT Model for Prompt Generation | OpenAI Chat Model | LLM backend for prompt generation | — | Generate Brand-Neutral Search Prompts (ai_languageModel) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Parse Prompt as JSON | Structured Output Parser (LangChain) | Enforces `{ "Prompts": "..." }` output for generated prompt | GPT Model for Parser Support (ai_languageModel) | Generate Brand-Neutral Search Prompts (ai_outputParser) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| GPT Model for Parser Support | OpenAI Chat Model | LLM used by parser to fix/normalize JSON | — | Parse Prompt as JSON (ai_languageModel) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Test Visibility on ChatGPT | LangChain Agent | Runs the prompt via OpenAI to simulate ChatGPT visibility | Generate Brand-Neutral Search Prompts | Test Visibility on Claude | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| GPT-4o-mini for ChatGPT Test | OpenAI Chat Model | LLM backend for ChatGPT test | — | Test Visibility on ChatGPT (ai_languageModel) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Test Visibility on Claude | LangChain Agent | Runs the prompt on Claude; continues on error | Test Visibility on ChatGPT | Test Visibility on DeepSeek | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Claude Sonnet 3.7 Model | Anthropic Chat Model | LLM backend for Claude test | — | Test Visibility on Claude (ai_languageModel) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Test Visibility on DeepSeek | LangChain Agent | Runs the prompt on DeepSeek; continues on error | Test Visibility on Claude | Test Visibility on Perplexity | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| DeepSeek Model for Testing | DeepSeek Chat Model | LLM backend for DeepSeek test | — | Test Visibility on DeepSeek (ai_languageModel) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Test Visibility on Perplexity | Perplexity | Queries Perplexity with the prompt; continues on error | Test Visibility on DeepSeek | Analyze All Platform Results | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Analyze All Platform Results | LangChain Agent | Combines all outputs; scores ranking presence & strength; outputs structured JSON | Test Visibility on Perplexity | Flatten JSON to 27 Data Fields | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| DeepSeek Model for Analysis | DeepSeek Chat Model | LLM backend for cross-platform analysis | — | Analyze All Platform Results (ai_languageModel) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Parse Analysis as Structured JSON | Structured Output Parser (LangChain) | Enforces analysis schema (ranking, strength %, competitors, summary) | — | Analyze All Platform Results (ai_outputParser) | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Flatten JSON to 27 Data Fields | Set | Flattens structured analysis to fixed reporting columns | Analyze All Platform Results | Output Data Complete | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Output Data Complete | NoOp | Terminal node (end of workflow) | Flatten JSON to 27 Data Fields | — | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |
| Sticky Note | Sticky Note | In-canvas documentation | — | — | ## AI Search Ranking Analyzer Across 4 Platforms  / Automates AI Search Engine Optimization (ASEO) tracking… / Setup steps 1–8 (full note content) |

**Sticky note full content (verbatim):**  
## AI Search Ranking Analyzer Across 4 Platforms  
Automates AI Search Engine Optimization (ASEO) tracking for  
digital marketing agencies. Receives client website + summary,  
generates brand-neutral search prompts using GPT-4.1-mini,  
tests visibility across ChatGPT, Claude, DeepSeek, and Perplexity  
simultaneously with error handling, analyzes each platform's  
ranking position and presence strength (0-100%), identifies top  
competitors across all platforms, and produces structured 27-field  
output with overall summary, strongest/weakest platforms, and  
actionable recommendations. Tracks the new frontier of AI  
discoverability beyond traditional SEO.  

## How it works  
1. Receives Website URL and Summary from parent workflow.  
2. AI generates search prompts users would ask (no brand names).  
3. Tests same prompt on ChatGPT (GPT-4o-mini).  
4. Tests on Claude (Sonnet 3.7) with error handling.  
5. Tests on DeepSeek with error handling.  
6. Tests on Perplexity with error handling.  
7. AI analyzes all 4 outputs for ranking, position, strength %.  
8. Identifies competitors mentioned across platforms.  
9. Flattens JSON to 27 fields for reporting.  
10. Returns comprehensive visibility scorecard.  

## Setup steps  
1. Create parent workflow with Execute Workflow node.  
2. Pass Website URL and Website Summary parameters.  
3. Configure OpenAI API credentials (GPT-4.1-mini, GPT-4o-mini).  
4. Configure Anthropic API credentials (Claude Sonnet 3.7).  
5. Configure DeepSeek API credentials.  
6. Configure Perplexity API credentials.  
7. Test with sample website data.  
8. Enable error handling (continueRegularOutput on agents).  

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named `AI_Ranking_Checker` (inactive by default if desired).

2. **Add Trigger node:**  
   - Node type: **Execute Workflow Trigger**  
   - Name: `Receive Website and Summary from Parent`  
   - Define **Workflow Inputs**:
     - `Website`
     - `Website Summary`

3. **Add OpenAI model node (prompt generation):**  
   - Node type: **OpenAI Chat Model (LangChain)**  
   - Name: `GPT Model for Prompt Generation`  
   - Model: `gpt-4.1-mini`  
   - Credentials: configure an **OpenAI API** credential in n8n and select it.

4. **Add Structured Output Parser (prompt):**  
   - Node type: **Structured Output Parser (LangChain)**  
   - Name: `Parse Prompt as JSON`  
   - JSON schema example:
     - `{ "Prompts": "Prompt" }`
   - Enable:
     - **Auto-fix** = on  
     - **Customize retry prompt** = on

5. **Add OpenAI model node (parser support):**  
   - Node type: **OpenAI Chat Model (LangChain)**  
   - Name: `GPT Model for Parser Support`  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API credential

6. **Wire parser support to parser:**  
   - Connect `GPT Model for Parser Support` → `Parse Prompt as JSON` via **ai_languageModel** connection (LangChain connection type).

7. **Add Agent node (prompt generation):**  
   - Node type: **Agent (LangChain)**  
   - Name: `Generate Brand-Neutral Search Prompts`  
   - Prompt text (define mode), using expressions:
     - Include `{{$json.Website}}` and `{{$json["Website Summary"]}}`
     - Explicitly instruct: “Don’t use client’s name directly… GOAL: Create one prompt”
   - Set **Has Output Parser** = on

8. **Wire model + parser into the prompt generator agent:**
   - `GPT Model for Prompt Generation` → `Generate Brand-Neutral Search Prompts` (**ai_languageModel**)  
   - `Parse Prompt as JSON` → `Generate Brand-Neutral Search Prompts` (**ai_outputParser**)

9. **Connect main flow:**  
   - `Receive Website and Summary from Parent` → `Generate Brand-Neutral Search Prompts` (main)

10. **Add OpenAI model node (ChatGPT test):**
    - Node type: **OpenAI Chat Model (LangChain)**
    - Name: `GPT-4o-mini for ChatGPT Test`
    - Model: `gpt-4o-mini`
    - Credentials: OpenAI API credential

11. **Add Agent node (ChatGPT test):**
    - Node type: **Agent (LangChain)**
    - Name: `Test Visibility on ChatGPT`
    - Text: `={{ $json.output.Prompts }}`
    - Wire `GPT-4o-mini for ChatGPT Test` → `Test Visibility on ChatGPT` (**ai_languageModel**)
    - Connect `Generate Brand-Neutral Search Prompts` → `Test Visibility on ChatGPT` (main)

12. **Add Anthropic model node (Claude):**
    - Node type: **Anthropic Chat Model (LangChain)**
    - Name: `Claude Sonnet 3.7 Model`
    - Model: `claude-3-7-sonnet-20250219`
    - Credentials: configure **Anthropic API** credential

13. **Add Agent node (Claude test) with error tolerance:**
    - Node type: **Agent (LangChain)**
    - Name: `Test Visibility on Claude`
    - Text: `={{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
    - Set node **On Error**: `Continue (continueRegularOutput)`
    - Wire `Claude Sonnet 3.7 Model` → `Test Visibility on Claude` (**ai_languageModel**)
    - Connect `Test Visibility on ChatGPT` → `Test Visibility on Claude` (main)

14. **Add DeepSeek model node (testing):**
    - Node type: **DeepSeek Chat Model (LangChain)**
    - Name: `DeepSeek Model for Testing`
    - Credentials: configure **DeepSeek API** credential

15. **Add Agent node (DeepSeek test) with error tolerance:**
    - Node type: **Agent (LangChain)**
    - Name: `Test Visibility on DeepSeek`
    - Text: `={{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
    - **On Error**: `Continue (continueRegularOutput)`
    - Wire `DeepSeek Model for Testing` → `Test Visibility on DeepSeek` (**ai_languageModel**)
    - Connect `Test Visibility on Claude` → `Test Visibility on DeepSeek` (main)

16. **Add Perplexity node with error tolerance:**
    - Node type: **Perplexity**
    - Name: `Test Visibility on Perplexity`
    - Messages → add one message content:
      - `={{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
    - **On Error**: `Continue (continueRegularOutput)`
    - Credentials: configure/select **Perplexity API** credential
    - Connect `Test Visibility on DeepSeek` → `Test Visibility on Perplexity` (main)

17. **Add DeepSeek model node (analysis):**
    - Node type: **DeepSeek Chat Model (LangChain)**
    - Name: `DeepSeek Model for Analysis`
    - Credentials: DeepSeek API credential (can be different from testing)

18. **Add Structured Output Parser (analysis schema):**
    - Node type: **Structured Output Parser (LangChain)**
    - Name: `Parse Analysis as Structured JSON`
    - Paste the detailed schema example (chatgpt/claude/deepseek/perplexity analysis + overall_summary).

19. **Add Agent node (cross-platform analysis):**
    - Node type: **Agent (LangChain)**
    - Name: `Analyze All Platform Results`
    - **Has Output Parser** = on
    - Prompt text should include:
      - Website from trigger
      - Generated prompt
      - Outputs from each platform
      - Perplexity content via: `{{ $json.choices[0].message.content }}`
    - Wire:
      - `DeepSeek Model for Analysis` → `Analyze All Platform Results` (**ai_languageModel**)
      - `Parse Analysis as Structured JSON` → `Analyze All Platform Results` (**ai_outputParser**)
    - Connect `Test Visibility on Perplexity` → `Analyze All Platform Results` (main)

20. **Add Set node (flatten):**
    - Node type: **Set**
    - Name: `Flatten JSON to 27 Data Fields`
    - Add fields mapping from `{{$json.output...}}` into flat keys (ChatGPT/Claude/DeepSeek/Perplexity + overall).
    - **Important fix to apply:** set `Prompt` to:
      - `={{ $('Generate Brand-Neutral Search Prompts').item.json.output.Prompts }}`
      - (instead of reading a non-existent `Receive...json.Prompt`)
    - Connect `Analyze All Platform Results` → `Flatten JSON to 27 Data Fields` (main)

21. **Add final node:**
    - Node type: **NoOp**
    - Name: `Output Data Complete`
    - Connect `Flatten JSON to 27 Data Fields` → `Output Data Complete`

22. **Credentials checklist (minimum):**
    - OpenAI API (for `gpt-4.1-mini` and `gpt-4o-mini`)
    - Anthropic API (Claude Sonnet 3.7)
    - DeepSeek API (testing + analysis, can reuse)
    - Perplexity API

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Search Ranking Analyzer Across 4 Platforms” sticky note describes intent, steps, and credential setup; emphasizes ASEO beyond traditional SEO and the 27-field output structure. | Sticky note embedded in workflow canvas (see full content in section 3). |
| Error handling is explicitly enabled for Claude/DeepSeek/Perplexity tests (`continueRegularOutput`), but downstream expressions still assume outputs exist. Consider adding fallbacks to avoid expression failures when a platform call errors. | Applies to cross-platform analysis reliability. |
| Field naming inconsistencies in the flattened output (e.g., “Calude”, “Weekest”, “Comeptitors”) may affect downstream automations expecting exact column headers. | Applies to “Flatten JSON to 27 Data Fields”. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.