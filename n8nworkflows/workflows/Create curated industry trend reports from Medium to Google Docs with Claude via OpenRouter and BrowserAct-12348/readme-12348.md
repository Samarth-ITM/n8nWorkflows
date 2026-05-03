Create curated industry trend reports from Medium to Google Docs with Claude via OpenRouter and BrowserAct

https://n8nworkflows.xyz/workflows/create-curated-industry-trend-reports-from-medium-to-google-docs-with-claude-via-openrouter-and-browseract-12348


# Create curated industry trend reports from Medium to Google Docs with Claude via OpenRouter and BrowserAct

## 1. Workflow Overview

**Purpose:** This workflow runs on a daily schedule, scrapes recent Medium articles from a chosen Medium tag archive using **BrowserAct**, uses **Claude (via OpenRouter)** to clean/deduplicate/filter spam and categorize articles, then writes a formatted digest into a **Google Doc** and notifies a **Slack** channel upon completion.

**Target use cases:**
- Automated “industry trends” or “daily digest” generation from Medium tag pages (e.g., AI, startups, product, engineering).
- Converting noisy scraped feeds into a structured, readable report in Google Docs.

### 1.1 Schedule & Target Selection
Daily trigger → set the Medium archive/tag URL to scrape.

### 1.2 Web Scraping (BrowserAct)
Invoke a BrowserAct “WORKFLOW” template to extract article data from the target Medium page.

### 1.3 AI Analysis & Structured Digest Generation
Send scraped JSON to a LangChain Agent backed by OpenRouter Claude; enforce strict JSON array output via a structured output parser.

### 1.4 Google Docs Report Assembly
Create a Google Doc titled from the digest cover item, split digest items, then append each item with spacing and conditional formatting (header+body vs body-only).

### 1.5 Completion Notification
After looping through all digest items, post a Slack notification.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule & Target Selection
**Overview:** Triggers the workflow daily and defines the Medium tag/archive URL to scrape.  
**Nodes involved:** `Every Day`, `Target Page Link`

#### Node: Every Day
- **Type / Role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration (interpreted):** Runs on a regular interval (“Every Day” intent; the rule is defined as an interval with default UI settings).
- **Outputs:** To `Target Page Link`.
- **Potential failures / edge cases:** None typical; misconfigured interval/timezone can cause unexpected run times.
- **Sticky note:** “Step 1: Schedule & Extract” (applies to this block).

#### Node: Target Page Link
- **Type / Role:** Set node (`n8n-nodes-base.set`) — defines scrape target.
- **Configuration:** Sets `Target_Medium_Link` to:  
  `https://medium.com/tag/artificial-intelligence/archive`
- **Outputs:** To `Scrape Headlines`.
- **Edge cases:** If Medium changes URL patterns or blocks scraping, downstream BrowserAct may fail.
- **Sticky note:** “Step 1: Schedule & Extract”.

---

### Block 2 — Scrape Medium with BrowserAct
**Overview:** Calls a BrowserAct workflow template to scrape article data from the Medium archive/tag page.  
**Nodes involved:** `Scrape Headlines`

#### Node: Scrape Headlines
- **Type / Role:** BrowserAct (`n8n-nodes-browseract.browserAct`) — remote scraping automation.
- **Configuration:**
  - Mode: `WORKFLOW`
  - `workflowId`: `70169279773867251`
  - Passes BrowserAct input `input-Medium` from n8n: `{{$json.Target_Medium_Link}}`
  - Incognito mode: disabled (`open_incognito_mode: false`)
- **Credentials:** `BrowserAct account`
- **Outputs:** To `Analyzer & Script writer`.
- **Edge cases / failures:**
  - BrowserAct auth/key issues.
  - Template ID missing or not accessible in the BrowserAct account.
  - Medium anti-bot measures, layout changes, or timeouts leading to partial/empty outputs.
- **Sticky note:** “Step 1: Schedule & Extract”.  
- **External references from sticky note:**  
  - https://docs.browseract.com (API key/workflow ID, n8n connection, templates)

---

### Block 3 — AI Intelligence (Clean, Deduplicate, Categorize, Format)
**Overview:** Sends scraped results to Claude via OpenRouter, instructs an agent to filter spam/duplicates and categorize articles, then forces output to match a strict schema (JSON array of `{header, body}` objects) without newline characters in values.  
**Nodes involved:** `Analyzer & Script writer`, `OpenRouter`, `Structured Output`

#### Node: OpenRouter
- **Type / Role:** LangChain Chat Model via OpenRouter (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`)
- **Configuration:**
  - Model: `anthropic/claude-sonnet-4.5`
  - Options: default/empty
- **Credentials:** `OpenRouter account`
- **Connections:**
  - Provides **AI language model** connection to both:
    - `Analyzer & Script writer` (agent)
    - `Structured Output` (output parser’s model channel)
- **Edge cases / failures:**
  - OpenRouter API key/limits, billing issues, model availability changes.
  - Response truncation if scraped payload is too large.
- **Version notes:** Node typeVersion `1`.

#### Node: Structured Output
- **Type / Role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)
- **Configuration:**
  - Auto-fix enabled (`autoFix: true`) to repair slight JSON/schema issues.
  - Schema example indicates an array of objects with keys: `header`, `body`.
- **Connections:**
  - `Structured Output` feeds the parser output into the Agent via `ai_outputParser` connection.
  - Receives model connection from `OpenRouter`.
- **Edge cases / failures:**
  - If the agent outputs newlines inside string values, the schema might still validate but violate the agent’s “no newline” rule (a semantic constraint).
  - AutoFix can mask systematic prompt issues by “fixing” output; validate quality if formatting drifts.
- **Version notes:** typeVersion `1.3`.

#### Node: Analyzer & Script writer
- **Type / Role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`)
- **Configuration:**
  - Input text: `={{ $json.output.string }}` (expects a `output.string` field from the BrowserAct result)
  - Prompt uses a **system message** defining:
    - Deduplication (near-identical titles)
    - Spam detection keywords (Invitation Code, Discount, VPN, Cheap, Save Money, repetitive promotion)
    - Categorization buckets (Must Reads, AI, Engineering, Wealth, Design/Culture)
    - **Strict output rules:** JSON array only; objects only `{header, body}`; no newline characters in values; one article per object
    - First object must be a “digest cover” (title + executive summary)
- **Connections (main outputs):**
  - Output 0 → `Create a Document for Outlines`
  - Output 1 → `Waiting for Inputs`
- **AI connections:**
  - Uses `OpenRouter` as the language model.
  - Uses `Structured Output` as output parser.
- **Edge cases / failures:**
  - If BrowserAct returns a different shape than expected (missing `output.string`), the agent receives empty input.
  - Large scraped JSON may exceed model context.
  - Category labeling/ordering may vary; downstream doc formatting assumes `header` may be empty for article items.
- **Version notes:** typeVersion `3`.
- **Sticky note:** “Step 2: AI Intelligence”.

---

### Block 4 — Create Google Doc & Prepare Items
**Overview:** Creates a new Google Doc titled from the digest’s first item, then merges control flow so the doc creation and the AI digest data are available before splitting into individual digest entries.  
**Nodes involved:** `Create a Document for Outlines`, `Waiting for Inputs`, `Split the AI-generated data`

#### Node: Create a Document for Outlines
- **Type / Role:** Google Docs (`n8n-nodes-base.googleDocs`) — creates a new document.
- **Configuration:**
  - Title: `={{ $json.output[0].header }}` (uses the first digest item’s header as doc title)
  - Folder: `default`
  - `alwaysOutputData: true` to ensure output continues even if creation returns minimal data.
- **Credentials:** `Google Docs account` (OAuth2)
- **Outputs:** To `Waiting for Inputs` (input 0).
- **Edge cases / failures:**
  - If AI output is not an array or missing `[0].header`, title expression fails or becomes blank.
  - Google auth/token expiration; permission issues.
- **Sticky note:** “Step 3: Digest Generation” (conceptually).

#### Node: Waiting for Inputs
- **Type / Role:** Merge (`n8n-nodes-base.merge`) — synchronization gate.
- **Configuration:**
  - Mode: `chooseBranch`
  - `useDataOfInput: 2` (uses data from input 2 / second input)
- **Connections:**
  - Input 0 from `Create a Document for Outlines`
  - Input 1 from `Analyzer & Script writer` (second output)
  - Output → `Split the AI-generated data`
- **Behavior:** Ensures both the document exists and the digest data is ready; passes forward the selected input’s data (the AI digest stream).
- **Edge cases:**
  - If either branch fails, merge may not execute.
  - “chooseBranch” logic can be confusing: you must ensure the desired payload (digest array) is what continues.
- **Sticky note:** “Step 3: Digest Generation”.

#### Node: Split the AI-generated data
- **Type / Role:** Split Out (`n8n-nodes-base.splitOut`) — turns an array into individual items.
- **Configuration:** Splits field `output` (expects the agent result to contain an `output` array).
- **Outputs:** To `Loop Over Items`.
- **Edge cases:**
  - If `output` is not an array, split produces no items or errors.
- **Sticky note:** “Step 3: Digest Generation”.

---

### Block 5 — Loop, Rate-Limit, Conditional Formatting, Append to Doc
**Overview:** Iterates over each digest object, inserts spacing and throttles requests, then conditionally inserts either header+body or body-only into the Google Doc.  
**Nodes involved:** `Loop Over Items`, `Rate Limit Mitigation`, `Add Spacer`, `Check for Header`, `Add Header & Body`, `Add Body`

#### Node: Loop Over Items
- **Type / Role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iteration controller.
- **Configuration:** Default batch behavior (no explicit batch size shown).
- **Connections:**
  - Receives items from `Split the AI-generated data`
  - Output 0 → `Rate Limit Mitigation` (process each item)
  - Output 1 → `Slack Team Notification` (runs when loop completes)
- **Edge cases:**
  - If batch size defaults unexpectedly, you may process more/less per cycle than intended.
- **Sticky note:** “Step 3: Digest Generation”.

#### Node: Rate Limit Mitigation
- **Type / Role:** Wait (`n8n-nodes-base.wait`) — throttling.
- **Configuration:** Waits `amount: 10` (seconds).
- **Outputs:** To both `Check for Header` and `Add Spacer` (in parallel).
- **Purpose:** Reduce likelihood of Google Docs API rate-limits and improve document readability by inserting spacing.
- **Edge cases:**
  - Excessive waits slow runs; insufficient waits may still hit Google API rate limits.
- **Sticky note:** “Step 3: Digest Generation”.

#### Node: Add Spacer
- **Type / Role:** Google Docs update (`n8n-nodes-base.googleDocs`) — inserts newline spacer.
- **Configuration:**
  - Operation: `update`
  - Action: insert text `=\n` (a newline)
  - Document URL/ID: `={{ $('Create a Document for Outlines').first().json.id }}`
- **Credentials:** Google Docs OAuth2
- **Input:** From `Rate Limit Mitigation`
- **Edge cases:**
  - Expression depends on `Create a Document for Outlines` output being available; `.first().json.id` must exist.
  - Note: Agent is told “no newlines in values”, but this node intentionally inserts a newline into the doc for spacing (that’s fine; it’s not part of the JSON schema).
- **Sticky note:** “Step 3: Digest Generation”.

#### Node: Check for Header
- **Type / Role:** IF (`n8n-nodes-base.if`) — choose formatting path.
- **Condition:** `{{$json.header}}` **is not empty**
- **Outputs:**
  - True → `Add Header & Body`
  - False → `Add Body`
- **Edge cases:**
  - Header containing whitespace may count as “not empty” depending on n8n string evaluation; consider trimming if needed.
- **Sticky note:** “Step 3: Digest Generation”.

#### Node: Add Header & Body
- **Type / Role:** Google Docs update — inserts header line then body text.
- **Configuration:**
  - Operation: `update`
  - Inserts:
    - `{{$json.header}}\n`
    - `{{$json.body}}`
  - Document URL/ID: `={{ $('Create a Document for Outlines').first().json.id }}`
- **Outputs:** To `Loop Over Items` (to continue iteration)
- **Edge cases:**
  - If header is long, formatting may be plain text only (no styling); this workflow does not apply headings.
  - Document ID expression must resolve.
- **Sticky note:** “Step 3: Digest Generation”.

#### Node: Add Body
- **Type / Role:** Google Docs update — inserts only body text.
- **Configuration:**
  - Operation: `update`
  - Inserts: `{{$json.body}}`
  - Document URL/ID: `={{ $('Create a Document for Outlines').first().json.id }}`
- **Outputs:** To `Loop Over Items`
- **Edge cases:** Same Google/auth/rate-limit considerations as above.
- **Sticky note:** “Step 3: Digest Generation”.

---

### Block 6 — Slack Notification
**Overview:** Sends a Slack message once all digest items have been appended to the document.  
**Nodes involved:** `Slack Team Notification`

#### Node: Slack Team Notification
- **Type / Role:** Slack (`n8n-nodes-base.slack`) — channel message.
- **Configuration:**
  - Sends text: `The 'Automated Industry Trend Scraper & Outline Creator' workflow has been completed.`
  - Channel: `#all-browseract-workflow-test` (ID `C09KLV9DJSX`)
  - `executeOnce: true` (prevents repeated posting across retries in some scenarios)
- **Credentials:** `Slack account 2`
- **Inputs:** From `Loop Over Items` “done” output.
- **Edge cases:**
  - Slack token scope issues; channel access issues.
  - If the loop never reaches completion (error mid-way), Slack may never fire.
- **Sticky note:** “Step 3: Digest Generation”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Day | Schedule Trigger | Daily entry point | — | Target Page Link | ### ⏰ Step 1: Schedule & Extract — The workflow triggers on a set schedule and targets a specific Medium tag URL. It then initiates a BrowserAct automation to scrape article data from the target page for analysis. |
| Target Page Link | Set | Define Medium archive/tag URL | Every Day | Scrape Headlines | ### ⏰ Step 1: Schedule & Extract — The workflow triggers on a set schedule and targets a specific Medium tag URL. It then initiates a BrowserAct automation to scrape article data from the target page for analysis. |
| Scrape Headlines | BrowserAct | Scrape Medium articles via BrowserAct workflow template | Target Page Link | Analyzer & Script writer | ### ⏰ Step 1: Schedule & Extract — The workflow triggers on a set schedule and targets a specific Medium tag URL. It then initiates a BrowserAct automation to scrape article data from the target page for analysis. |
| OpenRouter | LangChain Chat Model (OpenRouter) | Claude model provider | — | Analyzer & Script writer (ai), Structured Output (ai) | ### 🧠 Step 2: AI Intelligence — An AI agent analyzes the raw scraped data to remove duplicates and spam. It then categorizes valid articles into specific industry segments and formats them into a structured digest. |
| Structured Output | LangChain Structured Output Parser | Enforce `{header, body}` array schema | OpenRouter (ai) | Analyzer & Script writer (ai_outputParser) | ### 🧠 Step 2: AI Intelligence — An AI agent analyzes the raw scraped data to remove duplicates and spam. It then categorizes valid articles into specific industry segments and formats them into a structured digest. |
| Analyzer & Script writer | LangChain Agent | Clean/dedupe/filter/categorize and emit structured digest | Scrape Headlines | Create a Document for Outlines; Waiting for Inputs | ### 🧠 Step 2: AI Intelligence — An AI agent analyzes the raw scraped data to remove duplicates and spam. It then categorizes valid articles into specific industry segments and formats them into a structured digest. |
| Create a Document for Outlines | Google Docs | Create new doc using digest title | Analyzer & Script writer | Waiting for Inputs | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Waiting for Inputs | Merge | Synchronize doc creation + digest output | Create a Document for Outlines; Analyzer & Script writer | Split the AI-generated data | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Split the AI-generated data | Split Out | Split digest array into items | Waiting for Inputs | Loop Over Items | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Loop Over Items | Split In Batches | Iterate through digest items; completion branch | Split the AI-generated data; Add Header & Body; Add Body | Rate Limit Mitigation; Slack Team Notification | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Rate Limit Mitigation | Wait | Throttle Google Docs updates | Loop Over Items | Check for Header; Add Spacer | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Add Spacer | Google Docs | Insert newline spacer between entries | Rate Limit Mitigation | — | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Check for Header | IF | Choose “header+body” vs “body-only” | Rate Limit Mitigation | Add Header & Body; Add Body | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Add Header & Body | Google Docs | Append header line + body | Check for Header | Loop Over Items | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Add Body | Google Docs | Append body only (no header) | Check for Header | Loop Over Items | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Slack Team Notification | Slack | Post completion message | Loop Over Items (done output) | — | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Documentation | Sticky Note | Project notes / requirements / links | — | — | ## ⚡ Workflow Overview & Setup — Credentials: BrowserAct, OpenRouter (Claude), Google Docs, Slack. Mandatory: BrowserAct template “Automated Industry Trend Scraper & Outline Creator”. Links: https://docs.browseract.com |
| Step 1 Explanation | Sticky Note | Explains schedule + scrape stage | — | — | ### ⏰ Step 1: Schedule & Extract — The workflow triggers on a set schedule and targets a specific Medium tag URL. It then initiates a BrowserAct automation to scrape article data from the target page for analysis. |
| Step 2 Explanation | Sticky Note | Explains AI stage | — | — | ### 🧠 Step 2: AI Intelligence — An AI agent analyzes the raw scraped data to remove duplicates and spam. It then categorizes valid articles into specific industry segments and formats them into a structured digest. |
| Step 3 Explanation | Sticky Note | Explains doc generation stage | — | — | ### 📄 Step 3: Digest Generation — The system creates a new Google Doc and iterates through the structured AI output. It dynamically appends headers and article summaries to the document, inserting spacers between sections for readability. |
| Sticky Note | Sticky Note | Video link | — | — | https://www.youtube.com/watch?v=XUpmdpucNzg |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Automated Industry Trend Scraper & Outline Creator** (or your preferred name).

2. **Add the trigger**
   - Add node: **Schedule Trigger**
   - Name: **Every Day**
   - Configure it to run daily at your desired time/timezone.

3. **Add the target URL setter**
   - Add node: **Set**
   - Name: **Target Page Link**
   - Add a string field: `Target_Medium_Link`
   - Value: `https://medium.com/tag/artificial-intelligence/archive`  
   - Connect: **Every Day → Target Page Link**

4. **Add BrowserAct scraping node**
   - Add node: **BrowserAct**
   - Name: **Scrape Headlines**
   - Type: run a **WORKFLOW**
   - Select/enter your BrowserAct `workflowId` (must exist in BrowserAct): `70169279773867251`
   - Map BrowserAct input `input-Medium` to expression: `{{$json.Target_Medium_Link}}`
   - Credentials: configure **BrowserAct API** credential in n8n and select it.
   - Connect: **Target Page Link → Scrape Headlines**

5. **Add OpenRouter model node**
   - Add node: **OpenRouter Chat Model** (LangChain)
   - Name: **OpenRouter**
   - Model: `anthropic/claude-sonnet-4.5`
   - Credentials: create/select **OpenRouter API** credential.

6. **Add Structured Output Parser**
   - Add node: **Structured Output Parser** (LangChain)
   - Name: **Structured Output**
   - Enable **Auto-fix**
   - Provide a schema example indicating an array of objects with:
     - `header` (string)
     - `body` (string)

7. **Add the AI Agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Name: **Analyzer & Script writer**
   - Set its input text to expression (matching your BrowserAct output):  
     `{{$json.output.string}}`
   - Set prompt/system message with the workflow rules:
     - dedupe by title similarity
     - spam filtering keywords
     - categorization buckets
     - strict JSON array output, keys only `header` and `body`
     - no newline characters in values
     - first object is digest cover (title + executive summary)
   - Connect AI inputs:
     - Connect **OpenRouter** to the Agent as its **language model**
     - Connect **Structured Output** to the Agent as its **output parser**
   - Connect main flow:
     - **Scrape Headlines → Analyzer & Script writer**

8. **Create the Google Doc**
   - Add node: **Google Docs**
   - Name: **Create a Document for Outlines**
   - Operation: **Create**
   - Title expression: `{{$json.output[0].header}}`
   - Folder: `default` (or choose a folder)
   - Credentials: configure **Google Docs OAuth2** and select it.
   - Connect: **Analyzer & Script writer (main output 0) → Create a Document for Outlines**

9. **Add a Merge node to synchronize doc + data**
   - Add node: **Merge**
   - Name: **Waiting for Inputs**
   - Mode: **Choose Branch**
   - Set “use data of input” to the **second input** (so digest data continues)
   - Connect:
     - **Create a Document for Outlines → Waiting for Inputs (input 0)**
     - **Analyzer & Script writer (main output 1) → Waiting for Inputs (input 1)**

10. **Split digest array into items**
   - Add node: **Split Out**
   - Name: **Split the AI-generated data**
   - Field to split out: `output`
   - Connect: **Waiting for Inputs → Split the AI-generated data**

11. **Iterate over items**
   - Add node: **Split In Batches**
   - Name: **Loop Over Items**
   - Use default batch settings (or set batch size = 1 for clarity).
   - Connect: **Split the AI-generated data → Loop Over Items**

12. **Add throttling**
   - Add node: **Wait**
   - Name: **Rate Limit Mitigation**
   - Wait time: `10` seconds
   - Connect: **Loop Over Items → Rate Limit Mitigation**

13. **Insert spacer into the doc**
   - Add node: **Google Docs** (Update)
   - Name: **Add Spacer**
   - Operation: **Update**
   - Add an action: **Insert text** with value being a newline (e.g. `\n`)
   - Document ID/URL expression:  
     `{{$('Create a Document for Outlines').first().json.id}}`
   - Connect: **Rate Limit Mitigation → Add Spacer**

14. **Conditionally write header+body or body-only**
   - Add node: **IF**
   - Name: **Check for Header**
   - Condition: `header` **is not empty** (`{{$json.header}} notEmpty`)
   - Connect: **Rate Limit Mitigation → Check for Header**

15. **Write header + body path**
   - Add node: **Google Docs** (Update)
   - Name: **Add Header & Body**
   - Insert text actions in order:
     1) `{{$json.header}}\n`
     2) `{{$json.body}}`
   - Document ID/URL: `{{$('Create a Document for Outlines').first().json.id}}`
   - Connect: **Check for Header (true) → Add Header & Body**
   - Then connect: **Add Header & Body → Loop Over Items** (to continue loop)

16. **Write body-only path**
   - Add node: **Google Docs** (Update)
   - Name: **Add Body**
   - Insert text: `{{$json.body}}`
   - Document ID/URL: `{{$('Create a Document for Outlines').first().json.id}}`
   - Connect: **Check for Header (false) → Add Body**
   - Then connect: **Add Body → Loop Over Items**

17. **Slack notification upon completion**
   - Add node: **Slack**
   - Name: **Slack Team Notification**
   - Operation: send message to channel
   - Text: `The 'Automated Industry Trend Scraper & Outline Creator' workflow has been completed.`
   - Choose channel (e.g. `#all-browseract-workflow-test`)
   - Credentials: configure Slack API credential with chat:write permissions.
   - Connect: **Loop Over Items (done output) → Slack Team Notification**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow requirements: BrowserAct, OpenRouter (Claude), Google Docs, Slack. Mandatory BrowserAct template: **Automated Industry Trend Scraper & Outline Creator**. | From “Documentation” sticky note |
| How to find BrowserAct API key & workflow ID | https://docs.browseract.com |
| How to connect n8n to BrowserAct | https://docs.browseract.com |
| How to use & customize BrowserAct templates | https://docs.browseract.com |
| Video reference | https://www.youtube.com/watch?v=XUpmdpucNzg |