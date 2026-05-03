Scrape industry growth signals with BrowserAct, OpenRouter, and Slack reports

https://n8nworkflows.xyz/workflows/scrape-industry-growth-signals-with-browseract--openrouter--and-slack-reports-13374


# Scrape industry growth signals with BrowserAct, OpenRouter, and Slack reports

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Run a monthly “growth signals” monitor for a chosen industry (default: **Property Management**). It scrapes a growth/funding source via **BrowserAct**, uses an **OpenRouter (GPT-4o)** agent to filter results to the **current month** and the **target industry**, then posts a formatted, potentially multi-part report to a **Slack channel**.

**Primary use cases**
- Monthly market monitoring (funding rounds, growth events) for an industry vertical
- Automated lead discovery and Slack reporting
- Reusable pattern: scrape → LLM filter/format → Slack publishing

### 1.1 Scheduling & Target Selection
Starts on a monthly schedule and defines the industry keyword used downstream.

### 1.2 Web Extraction (BrowserAct)
Invokes a BrowserAct workflow (template-based) to collect structured company/activity data.

### 1.3 AI Filtering, Formatting & Output Structuring
Uses OpenRouter chat model + an agent with strict JSON output requirements to:
- Filter by industry match
- Filter by date (current month/year as specified)
- Format Slack-ready messages and split them if too long

### 1.4 Slack Delivery
Splits the AI-produced message array into individual items and posts each to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Targeting

**Overview:** Triggers monthly and sets a single variable (`Target_Industry`) used later by the AI agent to filter scraped companies.

**Nodes Involved**
- Monthly Trigger
- Set Target Industry

#### Node: **Monthly Trigger**
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — workflow entry point.
- **Configuration (interpreted):** Runs every **month** (`interval: months`). No specific day/time is shown, so n8n will use the default schedule time behavior for the instance/workflow.
- **Inputs / outputs:** No inputs. Output goes to **Set Target Industry** (main).
- **Edge cases / failures:**
  - If instance timezone differs from expectations, “monthly” execution timing can drift relative to business timezone.
  - If workflow is inactive (it is `active: false`), it will not run.

#### Node: **Set Target Industry**
- **Type / role:** `Set` (`n8n-nodes-base.set`) — defines configuration data for downstream nodes.
- **Configuration (interpreted):**
  - Creates field `Target_Industry` (string) with value **“Property Management”**.
- **Key variables/expressions:** Downstream references use:
  - `$('Set Target Industry').first().json.Target_Industry`
- **Inputs / outputs:** Input from **Monthly Trigger**. Output to **Perform web data extraction**.
- **Edge cases / failures:**
  - If renamed or removed, downstream expression lookups can break.
  - Industry matching is performed by AI prompt logic (string containment), so spelling/case/wording affects results.

**Sticky note coverage**
- “Step 1 Explanation” applies conceptually to this block: Scheduling & targeting.

---

### Block 2 — Web Data Extraction (BrowserAct)

**Overview:** Calls a BrowserAct “WORKFLOW” automation (likely the “ABM Signal Monitor” template) to scrape growth/funding signals into structured output.

**Nodes Involved**
- Perform web data extraction

#### Node: **Perform web data extraction**
- **Type / role:** `BrowserAct` (`n8n-nodes-browseract.browserAct`) — runs a BrowserAct workflow and returns extracted data.
- **Configuration (interpreted):**
  - **Mode:** `WORKFLOW`
  - **Timeout:** 7200 seconds (2 hours)
  - **BrowserAct workflowId:** `76594689776452912`
  - **Workflow config schema:** contains an input called `growthlist` (shown as removed=true in schema). If blank, BrowserAct defaults apply.
- **Credentials:** BrowserAct API credential “BrowserAct account”.
- **Inputs / outputs:** Input from **Set Target Industry** (but the industry is not mapped into BrowserAct in this JSON). Output goes to **Analyze the leads and generate Slack report**.
- **Data expectations (important):**
  - Downstream expects the BrowserAct result to include something like: `output.string` (see agent prompt: `{{ $json.output.string }}`).
  - The scraped “string” is expected to represent a JSON list of companies (per the agent’s system instructions).
- **Edge cases / failures:**
  - **Auth errors** (invalid/expired BrowserAct API key).
  - **Workflow ID mismatch** or template not present in BrowserAct account.
  - **Timeout** if the BrowserAct run exceeds 2 hours.
  - **Output shape changes**: if BrowserAct returns different fields, the agent prompt referencing `$json.output.string` can become empty/invalid.
  - If the BrowserAct workflow returns non-JSON/unparseable content, the AI may hallucinate structure unless constrained adequately.

**Sticky note coverage**
- “Step 2 Explanation” applies to this block (data extraction) and the AI/reporting that follows.

---

### Block 3 — AI Filtering, Formatting & Structured Output

**Overview:** Uses OpenRouter GPT-4o with an n8n LangChain Agent to filter scraped leads (industry + date), then produce Slack-formatted report text in a strict JSON object containing a `messages` array.

**Nodes Involved**
- OpenRouter Chat Model
- Structured Output Parser
- Analyze the leads and generate Slack report

#### Node: **OpenRouter Chat Model**
- **Type / role:** LangChain Chat Model for OpenRouter (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) — provides the LLM backend.
- **Configuration (interpreted):**
  - Model: `openai/gpt-4o` via OpenRouter.
  - No special model options configured.
- **Credentials:** OpenRouter API credential “OpenRouter account”.
- **Connections:**
  - Its **AI language model** output is connected to:
    - **Structured Output Parser** (as language model provider)
    - **Analyze the leads and generate Slack report** (as language model provider)
- **Edge cases / failures:**
  - OpenRouter auth/quota/rate limits.
  - Model availability changes or provider-side errors.
  - Output variability if prompts depend on a hard-coded date context (see agent node).

#### Node: **Structured Output Parser**
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces valid JSON output and can auto-fix.
- **Configuration (interpreted):**
  - **autoFix: true** — attempts to repair near-valid JSON.
  - Provides a **JSON schema example** expecting:
    - `{ "messages": ["..."] }`
- **Connections:**
  - Receives the LLM provider connection from **OpenRouter Chat Model**.
  - Provides **AI output parser** connection to **Analyze the leads and generate Slack report**.
- **Edge cases / failures:**
  - If the agent’s output is too malformed, auto-fix may fail.
  - If the agent outputs a different structure, downstream split will break (expects `output.messages`).

#### Node: **Analyze the leads and generate Slack report**
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — performs reasoning, filtering, formatting, and returns structured JSON.
- **Configuration (interpreted):**
  - **Prompt (user text):**
    - `Scraped Data: {{ $json.output.string }},`
    - `Target Industry: {{ $('Set Target Industry').first().json.Target_Industry }}`
  - **System message:** Defines strict logic:
    1. Filter companies where `"Industries"` contains the target industry.
    2. Keep only companies with `"Funding_date"` in **current month/year (January 2026)** (note: hard-coded).
    3. Aggregate into Slack report.
    4. Split into multiple messages if 6+ companies or to avoid Slack length limits.
    5. Output **strict JSON**: `{ "messages": [ "...", ... ] }`
  - **Output parser:** enabled (`hasOutputParser: true`) and connected to Structured Output Parser.
- **Inputs / outputs:**
  - Main input from **Perform web data extraction** (expects scraped content at `$json.output.string`).
  - AI language model input from **OpenRouter Chat Model**.
  - AI output parser input from **Structured Output Parser**.
  - Main output to **Split Out**.
- **Version-specific notes:**
  - This node uses `typeVersion: 3` and depends on n8n’s LangChain integration behavior (AI connectors: `ai_languageModel`, `ai_outputParser`).
- **Edge cases / failures:**
  - **Hard-coded date constraint:** It explicitly checks for **January 2026**. If run in any other month, it will likely filter out everything unless dates match January 2026. This is the biggest logic risk.
  - If scraped data doesn’t include `"Industries"` or `"Funding_date"`, filtering may produce empty output or incorrect reasoning.
  - If `$('Set Target Industry')...` fails (node renamed/removed or multiple items behavior changes), target industry may be blank.
  - Slack formatting expectations: websites/links may be missing; prompt asks for `> 🔗 [Website]` and “Why” context—if unavailable, the agent may invent context unless you enforce “use only provided data”.

**Sticky note coverage**
- “Step 2 Explanation” applies to this entire block (AI filtering & reporting).

---

### Block 4 — Message Splitting & Slack Delivery

**Overview:** Takes the agent’s JSON output (`messages` array), splits it into individual Slack messages, then posts each to a target channel.

**Nodes Involved**
- Split Out
- Send the report to the channel

#### Node: **Split Out**
- **Type / role:** `Split Out` (`n8n-nodes-base.splitOut`) — converts an array into multiple items.
- **Configuration (interpreted):**
  - Splits the field: `output.messages`
- **Inputs / outputs:**
  - Input from **Analyze the leads and generate Slack report**.
  - Output to **Send the report to the channel** (one item per message).
- **Edge cases / failures:**
  - If the agent output does not contain `output.messages` (or it is not an array), the node will error or produce no items.
  - If messages are empty, Slack node won’t post anything.

#### Node: **Send the report to the channel**
- **Type / role:** `Slack` (`n8n-nodes-base.slack`) — posts messages to Slack.
- **Configuration (interpreted):**
  - Operation: “send message” (implied by `text` + channel selection).
  - Channel: `C09KLV9DJSX` (named in cached result as `all-browseract-workflow-test`)
  - Text expression: `{{ $json["output.messages"] }}`
    - Because of Split Out, each item should contain a single message value; however, note that the expression references `output.messages` again.
- **Important potential issue:**
  - After **Split Out** on `output.messages`, each item typically contains the split value at a simpler path (often `output.messages` becomes a scalar, or the split value may be placed at `output.messages` depending on node behavior). If Split Out outputs the message as a top-level field instead, this Slack expression may not match.
  - If the value is already the message string, a safer expression is often `{{ $json.value }}` or `{{ $json["output"]["messages"] }}` depending on Split Out’s exact output structure.
- **Credentials:** Slack API credential “Slack account 2”.
- **Edge cases / failures:**
  - Slack auth errors / missing scopes (e.g., `chat:write`).
  - Posting blocked by channel permissions.
  - Slack message length limits: the agent attempts to split, but if it fails, Slack may reject or truncate.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Trigger | Schedule Trigger | Monthly workflow entry point | — | Set Target Industry | ### 🎯 Step 1: Scheduling & Targeting — The workflow triggers automatically on a monthly schedule... |
| Set Target Industry | Set | Defines `Target_Industry` used for filtering | Monthly Trigger | Perform web data extraction | ### 🎯 Step 1: Scheduling & Targeting — The workflow triggers automatically on a monthly schedule... |
| Perform web data extraction | BrowserAct | Runs BrowserAct workflow to scrape growth signals | Set Target Industry | Analyze the leads and generate Slack report | ### 🌐 🧠 Step 2: Data Extraction, AI Filtering & Reporting — BrowserAct navigates... |
| OpenRouter Chat Model | LangChain OpenRouter Chat Model | Provides GPT-4o LLM via OpenRouter | — (AI connector) | Structured Output Parser; Analyze the leads and generate Slack report | ### 🌐 🧠 Step 2: Data Extraction, AI Filtering & Reporting — BrowserAct navigates... |
| Structured Output Parser | LangChain Structured Output Parser | Enforces `{messages: []}` JSON output; auto-fix | OpenRouter Chat Model (AI connector) | Analyze the leads and generate Slack report (AI connector) | ### 🌐 🧠 Step 2: Data Extraction, AI Filtering & Reporting — BrowserAct navigates... |
| Analyze the leads and generate Slack report | LangChain Agent | Filters scraped data; formats Slack report; outputs messages array | Perform web data extraction; OpenRouter Chat Model (AI); Structured Output Parser (AI) | Split Out | ### 🌐 🧠 Step 2: Data Extraction, AI Filtering & Reporting — BrowserAct navigates... |
| Split Out | Split Out | Splits `output.messages` array into separate items | Analyze the leads and generate Slack report | Send the report to the channel | ### 🌐 🧠 Step 2: Data Extraction, AI Filtering & Reporting — BrowserAct navigates... |
| Send the report to the channel | Slack | Posts each message to a Slack channel | Split Out | — | ### 🌐 🧠 Step 2: Data Extraction, AI Filtering & Reporting — BrowserAct navigates... |
| Documentation | Sticky Note | Embedded operational notes and links | — | — | ## ⚡ Workflow Overview & Setup — Summary, requirements, and links to BrowserAct docs |
| Step 1 Explanation | Sticky Note | Visual annotation for Step 1 | — | — | ### 🎯 Step 1: Scheduling & Targeting (annotation) |
| Step 2 Explanation | Sticky Note | Visual annotation for Step 2 | — | — | ### 🌐 🧠 Step 2: Data Extraction, AI Filtering & Reporting (annotation) |
| Sticky Note | Sticky Note | Embedded media reference | — | — | @[youtube](6v3BZ7fw0sI) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Schedule Trigger**
   - Type: **Schedule Trigger**
   - Set schedule to **every 1 month**.
3. **Add node: Set**
   - Name it **Set Target Industry**
   - Add a string field:
     - `Target_Industry` = `Property Management` (or your preferred industry keyword).
   - Connect: **Monthly Trigger → Set Target Industry**
4. **Add node: BrowserAct**
   - Name it **Perform web data extraction**
   - Credentials: create/select **BrowserAct API** credentials.
   - Operation/mode: **WORKFLOW**
   - Workflow ID: `76594689776452912`
   - Timeout: `7200` seconds
   - (Optional) Configure inputs if your BrowserAct workflow expects them. In this JSON, `growthlist` exists but is marked removed; rely on BrowserAct defaults unless needed.
   - Connect: **Set Target Industry → Perform web data extraction**
5. **Add node: OpenRouter Chat Model**
   - Type: **OpenRouter Chat Model** (LangChain)
   - Credentials: create/select **OpenRouter API** credentials.
   - Model: `openai/gpt-4o`
6. **Add node: Structured Output Parser**
   - Type: **Structured Output Parser** (LangChain)
   - Enable **Auto-fix**
   - Provide an example schema like:
     - `{ "messages": ["..."] }`
   - Connect AI: **OpenRouter Chat Model (AI Language Model) → Structured Output Parser**
7. **Add node: AI Agent**
   - Type: **Agent** (LangChain)
   - Name: **Analyze the leads and generate Slack report**
   - Set **Prompt / Text** to include:
     - Scraped data from BrowserAct output (example from workflow):
       - `Scraped Data: {{ $json.output.string }}`
     - Target industry:
       - `Target Industry: {{ $('Set Target Industry').first().json.Target_Industry }}`
   - Set **System message** to:
     - Filter by industry field (`Industries` contains target)
     - Filter by funding date in the current month/year
     - Output strict JSON `{ "messages": [...] }`
     - Split into multiple messages if needed for Slack limits
   - Enable **Use Output Parser**
   - Connect AI:
     - **OpenRouter Chat Model (AI Language Model) → Agent**
     - **Structured Output Parser (AI Output Parser) → Agent**
   - Connect main:
     - **Perform web data extraction → Agent**
8. **Add node: Split Out**
   - Field to split: `output.messages`
   - Connect: **Agent → Split Out**
9. **Add node: Slack**
   - Name: **Send the report to the channel**
   - Credentials: create/select **Slack API** credentials (OAuth2).
   - Select posting **Channel** (e.g., `all-browseract-workflow-test` / channel id `C09KLV9DJSX`).
   - Message text: map the split message field.
     - Start with the workflow’s expression: `{{ $json["output.messages"] }}`
     - If it doesn’t post correctly, adjust to whatever field Split Out produces per item (commonly the split value becomes a scalar).
   - Connect: **Split Out → Slack**
10. **Activate workflow** (optional) to enable monthly runs.

**Credential requirements**
- **BrowserAct API**: must allow running the referenced BrowserAct workflow ID.
- **OpenRouter API**: must have access to `openai/gpt-4o`.
- **Slack OAuth**: must include permissions to post messages to the chosen channel (typically `chat:write`, plus channel access).

**Sub-workflow setup**
- The BrowserAct node invokes a BrowserAct workflow (not an n8n sub-workflow). Ensure the workflow/template exists in BrowserAct and returns structured data compatible with the agent prompt (ideally a JSON list containing at least `Name`, `Industries`, `Funding_date`, `Amount`, `Website/Link`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow overview, requirements, and setup notes (BrowserAct, OpenRouter, Slack; BrowserAct template “ABM Signal Monitor”; update target industry in Set node). | Included in the “Documentation” sticky note |
| How to find BrowserAct API key & workflow ID | https://docs.browseract.com |
| How to connect n8n to BrowserAct | https://docs.browseract.com |
| How to use & customize BrowserAct templates | https://docs.browseract.com |
| YouTube reference | `@[youtube](6v3BZ7fw0sI)` |

