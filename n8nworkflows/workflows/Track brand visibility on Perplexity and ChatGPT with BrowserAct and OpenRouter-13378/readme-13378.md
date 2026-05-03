Track brand visibility on Perplexity and ChatGPT with BrowserAct and OpenRouter

https://n8nworkflows.xyz/workflows/track-brand-visibility-on-perplexity-and-chatgpt-with-browseract-and-openrouter-13378


# Track brand visibility on Perplexity and ChatGPT with BrowserAct and OpenRouter

## 1. Workflow Overview

**Purpose:** This workflow acts as a GEO (Generative Engine Optimization) visibility tracker. On a weekly schedule, it generates two high-intent queries for your brand, runs them in parallel on **Perplexity** and **ChatGPT** via **BrowserAct** (browser automation + scraping), then uses an **OpenRouter LLM** to analyze whether your brand is recommended/mentioned and posts a structured report to **Slack**.

**Target use cases**
- Track brand/product visibility in AI answers over time
- Detect “invisible” (not mentioned) or “hallucinated” (mentioned incorrectly) appearances
- Provide a team-ready Slack update on visibility + sentiment

### 1.1 Logical blocks
1. **Scheduling & Brand Input**: Weekly trigger + define Brand and Description.
2. **AI Query Generation (OpenRouter)**: LLM produces two platform-specific search queries as strict JSON.
3. **Parallel AI Search Execution (BrowserAct)**: Run BrowserAct sub-workflow twice (Perplexity + ChatGPT) to scrape full answers.
4. **Merge / Synchronize**: Wait until both platform results are available.
5. **AI Analysis & Report Formatting (OpenRouter)**: LLM judges visibility + sentiment and outputs JSON containing database fields and a Slack-formatted message.
6. **Notification (Slack)**: Post the generated Slack message to a selected channel.
7. **Documentation (Sticky Notes)**: Embedded usage notes + links.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Brand Input
**Overview:** Starts weekly, then defines the brand identity (name + description) that will be tested for visibility.

**Nodes involved**
- **Weekly Trigger**
- **Add Brand & Description**

#### Node: Weekly Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration (interpreted):** Runs on an interval of **every 1 week** (no specific day/time shown in JSON; n8n uses the schedule trigger defaults unless configured further).
- **Connections:**
  - **Output →** Add Brand & Description
- **Edge cases / failures:**
  - If the workflow is inactive (`active: false`), it will not run.
  - Timezone/clock differences depend on n8n instance settings.

#### Node: Add Brand & Description
- **Type / role:** `n8n-nodes-base.set` — sets constants for downstream use.
- **Configuration (interpreted):**
  - Sets:
    - `Brand` = `"BrowserAct Automation"`
    - `Description` = `"A headless browser automation tool for AI agents that handles captchas and deep scraping."`
- **Connections:**
  - **Input ←** Weekly Trigger
  - **Output →** Generate search queries
- **Edge cases / failures:**
  - If you leave these empty or rename fields, downstream expressions referencing `Brand`/`Description` will break.

---

### Block 2 — AI Query Generation (OpenRouter)
**Overview:** Uses an OpenRouter-backed chat model and a structured output parser to generate exactly two queries (Perplexity-focused and ChatGPT-focused) as strict JSON.

**Nodes involved**
- **OpenRouter Chat Model**
- **Structured Output**
- **Generate search queries**

#### Node: OpenRouter Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides the LLM to LangChain Agent + parser.
- **Configuration (interpreted):**
  - Uses **OpenRouter API credentials**: “OpenRouter account”.
  - No extra options set (defaults apply: model selection likely configured in credential or node defaults depending on your n8n/OpenRouter setup).
- **Connections:**
  - **AI Language Model →** Generate search queries
  - **AI Language Model →** Structured Output
- **Version-specific:** node `typeVersion: 1`.
- **Edge cases / failures:**
  - Invalid/expired OpenRouter key, rate limits, model unavailability.
  - Output quality depends on chosen model; strict JSON requirement may still fail without parser autofix.

#### Node: Structured Output
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — forces/repairs JSON output to match schema.
- **Configuration (interpreted):**
  - `autoFix: true` (attempts to fix malformed JSON from the LLM)
  - Schema example requires:
    - `perplexity_query` (string)
    - `chatgpt_query` (string)
- **Connections:**
  - **AI Output Parser →** Generate search queries
- **Version-specific:** `typeVersion: 1.3`
- **Edge cases / failures:**
  - If the model outputs content that cannot be coerced into valid JSON with required keys, parsing fails.

#### Node: Generate search queries
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompt-driven agent that generates the two queries.
- **Configuration (interpreted):**
  - **Input text** passed to agent:
    - `Brand : {{ $json.Brand }}, Description : {{ $json.Description }}`
  - **System message** instructs:
    - Produce exactly two high-intent queries: one “research” for Perplexity, one “conversational” for ChatGPT
    - Output **ONLY raw JSON** (no markdown)
  - `hasOutputParser: true` (paired with “Structured Output”)
- **Connections:**
  - **Input ←** Add Brand & Description
  - **Main outputs →**
    - Run a Perplexity Search
    - Run ChatGPT Search
- **Version-specific:** `typeVersion: 3`
- **Edge cases / failures:**
  - If Brand/Description contain problematic characters, the model may produce malformed JSON (parser attempts autofix).
  - If OpenRouter model is not attached correctly via the AI connection, the agent will fail.

---

### Block 3 — Parallel AI Search Execution (BrowserAct)
**Overview:** Executes a BrowserAct workflow template twice, once targeting Perplexity and once targeting ChatGPT, feeding each the generated query and scraping the resulting answer text.

**Nodes involved**
- **Run a Perplexity Search**
- **Run ChatGPT Search**

#### Node: Run a Perplexity Search
- **Type / role:** `n8n-nodes-browseract.browserAct` — calls a BrowserAct workflow (sub-automation) to perform browser steps and scrape results.
- **Configuration (interpreted):**
  - Mode: **WORKFLOW**
  - BrowserAct `workflowId`: `76059068857656105`
  - Inputs passed to BrowserAct workflow:
    - `Target_Ai_Link` = `https://www.perplexity.ai/`
    - `Search_Querry` = `{{ $json.output.perplexity_query }}`
  - Uses BrowserAct credentials: “BrowserAct account”
  - Mapping mode “define below”; matching columns includes `input-Search_Querry`
- **Connections:**
  - **Input ←** Generate search queries
  - **Output →** Wait for both paths (merge) on input index 0
- **Edge cases / failures:**
  - BrowserAct auth errors, workflowId not found, template missing, quota limits.
  - Perplexity UI changes, bot detection, captchas (BrowserAct claims captcha handling but still can fail).
  - If `output.perplexity_query` is missing (parser failure), BrowserAct gets an empty query.

#### Node: Run ChatGPT Search
- **Type / role:** `n8n-nodes-browseract.browserAct` — same BrowserAct workflow template, different target URL and query.
- **Configuration (interpreted):**
  - Mode: **WORKFLOW**
  - BrowserAct `workflowId`: `76059068857656105`
  - Inputs:
    - `Target_Ai_Link` = `https://chatgpt.com`
    - `Search_Querry` = `{{ $json.output.chatgpt_query }}`
  - Uses BrowserAct credentials: “BrowserAct account”
- **Connections:**
  - **Input ←** Generate search queries
  - **Output →** Wait for both paths (merge) on input index 1
- **Edge cases / failures:**
  - ChatGPT may require login, present region/consent flows, or change DOM selectors, causing BrowserAct scraping failure.
  - Rate limiting / temporary blocks.
  - Empty/missing `output.chatgpt_query`.

**Sub-workflow reference (BrowserAct):**
- Both BrowserAct nodes invoke the **same BrowserAct workflow/template** (ID `76059068857656105`), described in sticky note as: **“AI Search Visibility Tracker (Perplexity & ChatGPT)”**.  
- Expected outputs (in this n8n workflow) are read from:
  - `Run ChatGPT Search`.json.output.string
  - `Run a Perplexity Search`.json.output.string  
  So the BrowserAct workflow must return an object containing `output.string` with the scraped answer text.

---

### Block 4 — Merge / Synchronize
**Overview:** Ensures both parallel BrowserAct executions have completed before analysis starts.

**Nodes involved**
- **Wait for both paths**

#### Node: Wait for both paths
- **Type / role:** `n8n-nodes-base.merge` — synchronization gate.
- **Configuration (interpreted):**
  - Mode: `chooseBranch`  
  In practice here it’s used to wait for both branches to arrive; the downstream node references results directly from the two BrowserAct nodes by name.
- **Connections:**
  - **Input 0 ←** Run a Perplexity Search
  - **Input 1 ←** Run ChatGPT Search
  - **Output →** Analyze both results & generate report
- **Version-specific:** `typeVersion: 3.2`
- **Edge cases / failures:**
  - If one BrowserAct branch errors, the merge may never get both inputs, preventing analysis (depending on n8n error settings).
  - If one branch returns no items, merge behavior can be confusing; consider “Always Output Data” or error handling if needed.

---

### Block 5 — AI Analysis & Report Formatting (OpenRouter)
**Overview:** Combines the generated queries, brand info, and both scraped results into a single prompt; an LLM then outputs strict JSON with (a) structured data fields and (b) a Slack-ready message.

**Nodes involved**
- **OpenRouter Chat Model1**
- **Structured Output Parser**
- **Analyze both results & generate report**

#### Node: OpenRouter Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider for the analysis agent and its parser.
- **Configuration (interpreted):**
  - Uses the same OpenRouter credentials as earlier.
- **Connections:**
  - **AI Language Model →** Analyze both results & generate report
  - **AI Language Model →** Structured Output Parser
- **Edge cases / failures:** Same as OpenRouter Chat Model (auth/rate limits/model issues).

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates and repairs final JSON response.
- **Configuration (interpreted):**
  - `autoFix: true`
  - Schema example expects:
    - `data` object (e.g., `perplexity_visible`, `chatgpt_visible`, `overall_sentiment`)
    - `slack_message` string
- **Connections:**
  - **AI Output Parser →** Analyze both results & generate report
- **Edge cases / failures:**
  - If the LLM fails to produce valid JSON with required keys, parsing fails.

#### Node: Analyze both results & generate report
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — performs visibility + sentiment analysis and produces final JSON output.
- **Configuration (interpreted):**
  - **Prompt text** manually constructs a combined input using node lookups:
    - Queries from `Generate search queries`:
      - `{{ $('Generate search queries').first().json.output.perplexity_query }}`
      - `{{ $('Generate search queries').first().json.output.chatgpt_query }}`
    - Brand profile from `Add Brand & Description`:
      - `{{ $('Add Brand & Description').first().json.Brand }}`
      - `{{ $('Add Brand & Description').first().json.Description }}`
    - Scraped results:
      - `ChatGPt Search Result : {{ $('Run ChatGPT Search').first().json.output.string }}`
      - `Perplexity Search Result : {{ $('Run a Perplexity Search').first().json.output.string }}`
  - **System message** defines “GEO Analyst” logic:
    - Determine VISIBLE / INVISIBLE / HALLUCINATED
    - Add sentiment
    - Output **ONLY raw JSON** with `data` + `slack_message`
  - `hasOutputParser: true`
  - `executeOnce: true` (agent executes once for the run)
- **Connections:**
  - **Input ←** Wait for both paths
  - **Output →** Send team update
- **Version-specific:** `typeVersion: 3`
- **Edge cases / failures:**
  - If either BrowserAct node doesn’t return `output.string`, the expression will evaluate to `undefined` and analysis quality will degrade or fail.
  - The system message references placeholders like `{{ $json.brand_name }}` / `brand_description` / etc., but the actual prompt text is supplying different labels. Because the agent mainly receives the *text* block, this mismatch can reduce reliability unless the agent is robust. (The structured parser helps only with output shape, not analysis correctness.)
  - Long scraped responses may exceed model context limits; consider truncation or summarization if needed.

---

### Block 6 — Notification (Slack)
**Overview:** Posts the LLM-generated Slack message to a chosen channel.

**Nodes involved**
- **Send team update**

#### Node: Send team update
- **Type / role:** `n8n-nodes-base.slack` — sends a Slack message.
- **Configuration (interpreted):**
  - Operation: send message to a **channel**
  - Channel: `C09KLV9DJSX` (cached name: `all-browseract-workflow-test`)
  - Text: `{{ $json.output.slack_message }}`
  - Credentials: “Slack account 2”
- **Connections:**
  - **Input ←** Analyze both results & generate report
  - No outputs used downstream.
- **Version-specific:** `typeVersion: 2.4`
- **Edge cases / failures:**
  - Slack token revoked/expired, missing scopes (e.g., `chat:write`), posting restricted to channel.
  - If `output.slack_message` is missing (parser/agent failure), Slack posts blank or errors depending on Slack node behavior.

---

### Block 7 — Documentation (Sticky Notes)
**Overview:** Provides embedded operational notes and links for credentials and BrowserAct setup.

**Nodes involved (sticky notes)**
- Documentation
- Step 1 Explanation
- Step 2 Explanation
- Step 3 Explanation
- Sticky Note (YouTube)

**Failure considerations:** none (non-executing nodes), but they contain critical setup links.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly entry point | — | Add Brand & Description | ### 🎯 Step 1: Strategy & Query Generation<br><br>The workflow starts by defining your brand profile.<br><br>An AI agent analyzes your brand description and generates two distinct high-intent search queries:<br>1. **Research Query** (for Perplexity)<br>2. **Conversational Query** (for ChatGPT) |
| Add Brand & Description | Set | Define Brand/Description constants | Weekly Trigger | Generate search queries | ### 🎯 Step 1: Strategy & Query Generation<br><br>The workflow starts by defining your brand profile.<br><br>An AI agent analyzes your brand description and generates two distinct high-intent search queries:<br>1. **Research Query** (for Perplexity)<br>2. **Conversational Query** (for ChatGPT) |
| OpenRouter Chat Model | OpenRouter Chat Model (LangChain) | LLM for query generation | — (AI connection) | Generate search queries (AI), Structured Output (AI) |  |
| Structured Output | Structured Output Parser (LangChain) | Enforce JSON schema for queries | OpenRouter Chat Model (AI) | Generate search queries (AI parser) |  |
| Generate search queries | LangChain Agent | Produce perplexity/chatgpt queries | Add Brand & Description; OpenRouter Chat Model (AI); Structured Output (AI parser) | Run a Perplexity Search; Run ChatGPT Search | ### 🎯 Step 1: Strategy & Query Generation<br><br>The workflow starts by defining your brand profile.<br><br>An AI agent analyzes your brand description and generates two distinct high-intent search queries:<br>1. **Research Query** (for Perplexity)<br>2. **Conversational Query** (for ChatGPT) |
| Run a Perplexity Search | BrowserAct | Execute BrowserAct workflow against Perplexity | Generate search queries | Wait for both paths | ### 🕵️ Step 2: Parallel Search Execution<br><br>BrowserAct executes the generated queries on both platforms simultaneously.<br><br>It navigates to Perplexity and ChatGPT, inputs the questions, and scrapes the full AI-generated answers to capture exactly what potential customers are seeing. |
| Run ChatGPT Search | BrowserAct | Execute BrowserAct workflow against ChatGPT | Generate search queries | Wait for both paths | ### 🕵️ Step 2: Parallel Search Execution<br><br>BrowserAct executes the generated queries on both platforms simultaneously.<br><br>It navigates to Perplexity and ChatGPT, inputs the questions, and scrapes the full AI-generated answers to capture exactly what potential customers are seeing. |
| Wait for both paths | Merge | Synchronize both search branches | Run a Perplexity Search; Run ChatGPT Search | Analyze both results & generate report |  |
| OpenRouter Chat Model1 | OpenRouter Chat Model (LangChain) | LLM for analysis/report generation | — (AI connection) | Analyze both results & generate report (AI), Structured Output Parser (AI) | ### 🧠 Step 3: Visibility & Sentiment Analysis<br><br>A "GEO Analyst" AI reviews the scraped results.<br><br>It determines if your brand was **Visible** (recommended), **Invisible** (ignored), or **Hallucinated** (incorrect facts). It also gauges the sentiment of the mention, then the findings are formatted into a clean Slack report. |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for report | OpenRouter Chat Model1 (AI) | Analyze both results & generate report (AI parser) | ### 🧠 Step 3: Visibility & Sentiment Analysis<br><br>A "GEO Analyst" AI reviews the scraped results.<br><br>It determines if your brand was **Visible** (recommended), **Invisible** (ignored), or **Hallucinated** (incorrect facts). It also gauges the sentiment of the mention, then the findings are formatted into a clean Slack report. |
| Analyze both results & generate report | LangChain Agent | Judge visibility/sentiment and format Slack message | Wait for both paths; OpenRouter Chat Model1 (AI); Structured Output Parser (AI parser) | Send team update | ### 🧠 Step 3: Visibility & Sentiment Analysis<br><br>A "GEO Analyst" AI reviews the scraped results.<br><br>It determines if your brand was **Visible** (recommended), **Invisible** (ignored), or **Hallucinated** (incorrect facts). It also gauges the sentiment of the mention, then the findings are formatted into a clean Slack report. |
| Send team update | Slack | Post Slack report | Analyze both results & generate report | — |  |
| Documentation | Sticky Note | Setup notes + links | — | — | ## ⚡ Workflow Overview & Setup<br><br>**Summary:** This automation acts as a "GEO" (Generative Engine Optimization) Tracker. It monitors how your brand appears in AI search engines (Perplexity & ChatGPT) by simulating real user queries and analyzing the visibility of your product.<br><br>### Requirements<br>* **Credentials:** BrowserAct, OpenRouter (GPT-4), Slack.<br>* **Mandatory:** BrowserAct API (Template: **AI Search Visibility Tracker (Perplexity & ChatGPT)**)<br><br>### How to Use<br>1. **Credentials:** Set up API keys for BrowserAct, OpenRouter, and Slack.<br>2. **BrowserAct Template:** Ensure you have the **AI Search Visibility Tracker (Perplexity & ChatGPT)** template saved in your BrowserAct account.<br>3. **Configuration:** Update the **Add Brand & Description** node with your specific Company Name and Value Proposition.<br><br>### Need Help?<br>[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)<br>[How to Connect n8n to BrowserAct](https://docs.browseract.com)<br>[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | Explains query generation step | — | — | ### 🎯 Step 1: Strategy & Query Generation<br><br>The workflow starts by defining your brand profile.<br><br>An AI agent analyzes your brand description and generates two distinct high-intent search queries:<br>1. **Research Query** (for Perplexity)<br>2. **Conversational Query** (for ChatGPT) |
| Step 2 Explanation | Sticky Note | Explains parallel BrowserAct step | — | — | ### 🕵️ Step 2: Parallel Search Execution<br><br>BrowserAct executes the generated queries on both platforms simultaneously.<br><br>It navigates to Perplexity and ChatGPT, inputs the questions, and scrapes the full AI-generated answers to capture exactly what potential customers are seeing. |
| Step 3 Explanation | Sticky Note | Explains analysis + reporting step | — | — | ### 🧠 Step 3: Visibility & Sentiment Analysis<br><br>A "GEO Analyst" AI reviews the scraped results.<br><br>It determines if your brand was **Visible** (recommended), **Invisible** (ignored), or **Hallucinated** (incorrect facts). It also gauges the sentiment of the mention, then the findings are formatted into a clean Slack report. |
| Sticky Note | Sticky Note | Video link | — | — | @[youtube](lH-uMJYQIJ4) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name: “Track brand visibility on Perplexity and ChatGPT with BrowserAct & OpenRouter”

2. **Add trigger**
   - Add node: **Schedule Trigger**
   - Configure: **Interval = Weeks (every 1 week)**

3. **Add brand definition**
   - Add node: **Set**
   - Name it: **Add Brand & Description**
   - Add fields:
     - `Brand` (string) = your product/company name
     - `Description` (string) = short value proposition

4. **Add OpenRouter chat model (for query generation)**
   - Add node: **OpenRouter Chat Model** (LangChain)
   - Create/Open credentials: **OpenRouter API**
     - Provide your OpenRouter API key
     - Choose the model if required by your node/instance setup

5. **Add structured output parser (for query JSON)**
   - Add node: **Structured Output** (LangChain Structured Output Parser)
   - Enable **Auto-fix**
   - Provide a schema example with keys:
     - `perplexity_query`
     - `chatgpt_query`

6. **Add agent to generate queries**
   - Add node: **AI Agent** (LangChain Agent)
   - Name: **Generate search queries**
   - Prompt text (use expressions):
     - `Brand : {{ $json.Brand }}, Description : {{ $json.Description }}`
   - System message: instruct it to output ONLY JSON with the two keys (Perplexity + ChatGPT query).
   - Connect AI wiring:
     - **OpenRouter Chat Model** → Agent (AI language model connection)
     - **Structured Output** → Agent (AI output parser connection)
   - Connect main flow:
     - Schedule Trigger → Add Brand & Description → Generate search queries

7. **Set up BrowserAct prerequisites**
   - In **BrowserAct**, ensure you have the template/workflow saved: **“AI Search Visibility Tracker (Perplexity & ChatGPT)”**
   - Note the **BrowserAct Workflow ID** (you will paste it in both BrowserAct nodes).
   - In n8n, create BrowserAct credentials: **BrowserAct API key**.

8. **Add BrowserAct node for Perplexity**
   - Add node: **BrowserAct**
   - Name: **Run a Perplexity Search**
   - Mode: **WORKFLOW**
   - Workflow ID: your BrowserAct workflow id (in the provided workflow it is `76059068857656105`)
   - Workflow inputs:
     - `Target_Ai_Link` = `https://www.perplexity.ai/`
     - `Search_Querry` = `{{ $json.output.perplexity_query }}`
   - Connect: Generate search queries → Run a Perplexity Search

9. **Add BrowserAct node for ChatGPT**
   - Add node: **BrowserAct**
   - Name: **Run ChatGPT Search**
   - Mode: **WORKFLOW**
   - Workflow ID: same BrowserAct workflow id
   - Inputs:
     - `Target_Ai_Link` = `https://chatgpt.com`
     - `Search_Querry` = `{{ $json.output.chatgpt_query }}`
   - Connect: Generate search queries → Run ChatGPT Search

10. **Add merge/synchronization**
   - Add node: **Merge**
   - Name: **Wait for both paths**
   - Mode: **chooseBranch**
   - Connect:
     - Run a Perplexity Search → Merge (Input 1 / index 0)
     - Run ChatGPT Search → Merge (Input 2 / index 1)

11. **Add OpenRouter chat model (for analysis)**
   - Add node: **OpenRouter Chat Model** again
   - Name: **OpenRouter Chat Model1**
   - Reuse the same OpenRouter credentials

12. **Add structured output parser (for final report JSON)**
   - Add node: **Structured Output Parser**
   - Enable **Auto-fix**
   - Schema example must include:
     - `data` (object)
     - `slack_message` (string)

13. **Add analysis agent**
   - Add node: **AI Agent**
   - Name: **Analyze both results & generate report**
   - Prompt text: combine brand info, both queries, and both scraped results using expressions referencing the earlier nodes, e.g.:
     - queries from **Generate search queries**
     - brand fields from **Add Brand & Description**
     - results from each BrowserAct node (ensure your BrowserAct output includes `output.string`)
   - System message: GEO analyst rules + strict “ONLY raw JSON”.
   - Connect AI wiring:
     - OpenRouter Chat Model1 → Analysis Agent (AI language model)
     - Structured Output Parser → Analysis Agent (AI output parser)
   - Connect main flow:
     - Wait for both paths → Analyze both results & generate report

14. **Add Slack posting**
   - Add node: **Slack**
   - Name: **Send team update**
   - Credentials: Slack OAuth2 / Slack API token with `chat:write`
   - Select: **Channel**
   - Channel: pick target channel
   - Message text: `{{ $json.output.slack_message }}`
   - Connect: Analyze both results & generate report → Send team update

15. **(Optional) Add sticky notes**
   - Add sticky notes for internal documentation and include links:
     - https://docs.browseract.com
     - YouTube embed: `@[youtube](lH-uMJYQIJ4)`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| BrowserAct, OpenRouter, Slack credentials are required | From “Documentation” sticky note |
| Mandatory BrowserAct template: **AI Search Visibility Tracker (Perplexity & ChatGPT)** | From “Documentation” sticky note |
| How to find BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to connect n8n to BrowserAct | https://docs.browseract.com |
| How to use & customize BrowserAct templates | https://docs.browseract.com |
| Video link | @[youtube](lH-uMJYQIJ4) |