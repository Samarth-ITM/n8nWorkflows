Generate SEO articles from search queries to WordPress with BrowserAct and OpenRouter

https://n8nworkflows.xyz/workflows/generate-seo-articles-from-search-queries-to-wordpress-with-browseract-and-openrouter-13383


# Generate SEO articles from search queries to WordPress with BrowserAct and OpenRouter

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate SEO articles from search queries to WordPress with BrowserAct and OpenRouter  
**Purpose:** Turn a predefined list of high-intent search queries into published WordPress posts by (1) scraping SERP/community-style insights via BrowserAct, (2) generating an SEO-focused HTML article with an LLM through OpenRouter, (3) parsing/validating structured JSON output, and (4) publishing the post to WordPress (optionally notifying Slack).

**Target use cases**
- Programmatic SEO content generation from query lists
- Rapid publishing of “industry consensus/market research” style long-form articles
- Automated research → drafting → CMS publishing pipeline

### Logical blocks
1. **1.1 Trigger & Query Input Preparation**: manual start, define an array of queries, split into individual items.
2. **1.2 Batch Iteration Control**: iterate item-by-item (batch loop) so each query is processed and published before moving to the next.
3. **1.3 Research Capture (BrowserAct)**: run a BrowserAct workflow per query to extract relevant search results text.
4. **1.4 AI Writing + Structured Parsing**: LLM agent produces a strict JSON object (`title`, `article` HTML), validated/auto-fixed by a structured output parser.
5. **1.5 Publish & Optional Notification**: create a WordPress post; loop continues; optional Slack notification (currently disabled).

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Query Input Preparation

**Overview:** Starts the workflow manually, defines the list of target search queries, then converts the array into one item per query for downstream processing.

**Nodes involved:**
- Execute manually
- Set queries (Search Input)
- Split Out

#### Node: **Execute manually**
- **Type / role:** `Manual Trigger` — entry point for ad-hoc runs.
- **Configuration (interpreted):** No parameters; run is started by a user in the n8n UI.
- **Inputs/outputs:** No inputs. Output goes to **Set queries (Search Input)**.
- **Failure/edge cases:** None (except workflow not executed).
- **Version notes:** Standard node v1.

#### Node: **Set queries (Search Input)**
- **Type / role:** `Set` — defines the query list payload.
- **Configuration (interpreted):**
  - Creates field **`Queries`** as an **array**:
    - `"What are the best tools for automation"`
    - `"Looking for a cheaper alternative to Zapier"`
- **Key variables/expressions:** None; static array value.
- **Inputs/outputs:** Receives from **Execute manually**, outputs to **Split Out**.
- **Failure/edge cases:**
  - Empty array → downstream nodes may execute zero times.
  - Non-array type in `Queries` → `Split Out` will fail.
- **Version notes:** Set node v3.4.

#### Node: **Split Out**
- **Type / role:** `Split Out` — converts an array into multiple items.
- **Configuration (interpreted):**
  - `fieldToSplitOut`: **`Queries`**
  - Produces one execution item per array element.
- **Inputs/outputs:** Input from **Set queries (Search Input)**; output to **Loop Over Items**.
- **Failure/edge cases:**
  - Missing `Queries` field or not an array → split error.
  - Array elements that are not strings still pass through; may reduce BrowserAct/LLM quality.
- **Version notes:** Split Out v1.

---

### 2.2 Batch Iteration Control

**Overview:** Processes queries sequentially (or in batches) to control throughput and ensure each query completes research → writing → publishing.

**Nodes involved:**
- Loop Over Items

#### Node: **Loop Over Items**
- **Type / role:** `Split In Batches` — loop controller for iterative processing.
- **Configuration (interpreted):**
  - Default batch settings (batch size not explicitly set in parameters).
  - Has two outgoing connections:
    - To **Extract search engine result** (main processing path)
    - To **Send completion notification** (notification path; node disabled)
- **Inputs/outputs:**
  - Input from **Split Out**
  - Output 1: **Extract search engine result**
  - Output 2: **Send completion notification** (disabled)
  - Receives loop-back input from **Create a post** to continue to next item.
- **Failure/edge cases:**
  - If downstream fails and no error handling is configured, the loop stops.
  - If WordPress creation returns an error, subsequent queries won’t run.
- **Version notes:** Split In Batches v3.

---

### 2.3 Research Capture (BrowserAct)

**Overview:** For each query item, calls a BrowserAct workflow (“Programmatic SEO Data Pipeline” per notes) to scrape/collect relevant SERP/community discussion data.

**Nodes involved:**
- Extract search engine result

#### Node: **Extract search engine result**
- **Type / role:** `BrowserAct` (`n8n-nodes-browseract.browserAct`) — executes an external BrowserAct workflow/template.
- **Configuration (interpreted):**
  - `type`: WORKFLOW
  - `workflowId`: `76966914945590330`
  - Passes one input field to BrowserAct workflow config:
    - `input-Querry` = `{{ $json.Queries }}`
  - BrowserAct schema indicates additional possible inputs (`Reddit_Answer`, `Reddit_Search`) but they are marked removed/not used here.
- **Credentials:** BrowserAct API credential **“BrowserAct account”**
- **Inputs/outputs:**
  - Input: current item containing `Queries` (string) from **Loop Over Items**
  - Output: BrowserAct response; downstream uses **`$json.output.string`** in the agent prompt.
- **Key expressions/variables:**
  - `={{ $json.Queries }}` (note spelling “Querry” is used as the BrowserAct input key)
- **Failure/edge cases:**
  - Invalid API key / missing BrowserAct credentials → auth failure.
  - Wrong workflowId or template not present → run failure.
  - Output shape mismatch (e.g., `output.string` missing) → downstream prompt will be empty or expression error.
  - Browser automation timeouts, CAPTCHA, blocked SERP pages → partial/empty data.
- **Version notes:** BrowserAct node v1.

---

### 2.4 AI Writing + Structured Parsing

**Overview:** Uses an LLM agent (via OpenRouter) to convert scraped research into an authoritative HTML article and a title, enforcing strict JSON output through a structured parser that can auto-fix format issues.

**Nodes involved:**
- OpenRouter Chat Model
- Structured Output Parser
- Analyze the results and generate a post

#### Node: **OpenRouter Chat Model**
- **Type / role:** LangChain Chat Model for OpenRouter (`lmChatOpenRouter`) — provides the LLM backend.
- **Configuration (interpreted):**
  - Model: **`openai/gpt-5`**
  - Options: default
- **Credentials:** OpenRouter API credential **“OpenRouter account”**
- **Inputs/outputs:**
  - Connected via `ai_languageModel` to:
    - **Analyze the results and generate a post** (agent)
    - **Structured Output Parser** (as model support for auto-fix)
- **Failure/edge cases:**
  - Invalid OpenRouter key, insufficient quota, model not available → request failure.
  - Model output that violates JSON requirements → relies on parser auto-fix, may still fail if too malformed.
- **Version notes:** Node v1.

#### Node: **Structured Output Parser**
- **Type / role:** LangChain structured parser (`outputParserStructured`) — enforces JSON schema and optionally repairs output.
- **Configuration (interpreted):**
  - `autoFix`: **true** (attempts to repair invalid JSON using the connected model)
  - Example schema provided (not strict JSON Schema, but an example object):
    - `{ "title": "...", "article": "..." }`
- **Inputs/outputs:**
  - Receives model connection from **OpenRouter Chat Model** (`ai_languageModel`)
  - Feeds parser connection into the agent: **Analyze the results and generate a post** (`ai_outputParser`)
- **Failure/edge cases:**
  - If model returns non-repairable output → parser throws an error.
  - If the agent returns extra keys or missing keys → parser may fail or auto-fix depending on output.
- **Version notes:** Node v1.3.

#### Node: **Analyze the results and generate a post**
- **Type / role:** LangChain Agent (`agent`) — prompts the model with scraped data + system instructions to draft the article.
- **Configuration (interpreted):**
  - Prompt text (user content):
    - `Scrapped Data : {{ $json.output.string }}`
  - System message defines:
    - Role: “Senior Technical Editor…”
    - Sanitization: forbids “Reddit/Redditor/thread/upvotes/comment section/community”; mandates framing as “market research/industry consensus”
    - Output: single JSON object with exactly two keys: `title`, `article`
    - Article must be **HTML** with semantic tags and required sections (hook, comparison table, paradox section with blockquote, stack section, decision matrix, conclusion, CTA).
  - `promptType`: define
  - `hasOutputParser`: true (uses the Structured Output Parser)
- **Inputs/outputs:**
  - Main input from **Extract search engine result**
  - Uses `ai_languageModel` from **OpenRouter Chat Model**
  - Uses `ai_outputParser` from **Structured Output Parser**
  - Main output to **Create a post**
- **Key expressions/variables:**
  - `{{ $json.output.string }}` must exist from BrowserAct output.
- **Failure/edge cases:**
  - If scraped text is too long → token/size limits; content may truncate or fail.
  - Empty scraped data → generic/low-quality article, or agent may hallucinate unless constrained.
  - If the agent includes forbidden words (from sanitization rule) → content policy breach for your editorial goals (workflow won’t automatically detect unless you add checks).
- **Version notes:** Agent node v3.

---

### 2.5 Publish & Optional Notification

**Overview:** Publishes the generated content to WordPress as a new post, then loops back to process the next query. A Slack completion message node exists but is disabled.

**Nodes involved:**
- Create a post
- Send completion notification (disabled)

#### Node: **Create a post**
- **Type / role:** `WordPress` — creates a WordPress post via WordPress API credentials.
- **Configuration (interpreted):**
  - Operation: create post
  - Title: `{{ $json.output.title }}`
  - Content: `{{ $json.output.article }}`
- **Credentials:** WordPress API credential **“Wordpress account”**
- **Inputs/outputs:**
  - Input from **Analyze the results and generate a post**
  - Output loops back to **Loop Over Items** to continue the batch.
- **Failure/edge cases:**
  - Auth errors (wrong application password/user, revoked token, wrong site URL in credential).
  - WordPress rejecting content (invalid HTML is usually accepted, but security plugins may block).
  - Missing `$json.output.*` if parsing failed upstream → expression error or empty post.
- **Version notes:** WordPress node v1.

#### Node: **Send completion notification** (disabled)
- **Type / role:** `Slack` — sends a completion message to a channel.
- **Configuration (interpreted):**
  - Channel: `all-browseract-workflow-test` (ID: `C09KLV9DJSX`)
  - Text: “The Programmatic SEO Data Pipeline job has completed.”
  - Node is **disabled**, so it will not run.
- **Credentials:** Slack API credential **“Slack account 2”**
- **Inputs/outputs:**
  - Connected from **Loop Over Items**, but disabled.
- **Failure/edge cases (if enabled):**
  - Slack auth/permissions issues (missing `chat:write` scope; not in channel).
- **Version notes:** Slack node v2.4.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Execute manually | manualTrigger | Manual entry point | — | Set queries (Search Input) |  |
| Set queries (Search Input) | set | Define query array input | Execute manually | Split Out |  |
| Split Out | splitOut | Expand query array into items | Set queries (Search Input) | Loop Over Items |  |
| Loop Over Items | splitInBatches | Iterate over queries + loop control | Split Out; Create a post | Extract search engine result; Send completion notification (disabled) | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / The workflow iterates through a predefined list of high-intent search queries... published to WordPress as a new post. |
| Extract search engine result | browserAct | Run BrowserAct scraping workflow per query | Loop Over Items | Analyze the results and generate a post | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / The workflow iterates through a predefined list of high-intent search queries... published to WordPress as a new post. |
| OpenRouter Chat Model | lmChatOpenRouter | LLM provider (OpenRouter GPT-5) | — (AI connection) | Analyze the results and generate a post (AI); Structured Output Parser (AI) | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / The workflow iterates through a predefined list of high-intent search queries... published to WordPress as a new post. |
| Structured Output Parser | outputParserStructured | Enforce/repair JSON output | OpenRouter Chat Model (AI) | Analyze the results and generate a post (AI parser) | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / The workflow iterates through a predefined list of high-intent search queries... published to WordPress as a new post. |
| Analyze the results and generate a post | agent | Draft SEO HTML article from scraped data | Extract search engine result; (AI model/parser) | Create a post | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / The workflow iterates through a predefined list of high-intent search queries... published to WordPress as a new post. |
| Create a post | wordpress | Publish generated title+HTML to WP | Analyze the results and generate a post | Loop Over Items | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / The workflow iterates through a predefined list of high-intent search queries... published to WordPress as a new post. |
| Send completion notification | slack | Post-run Slack message | Loop Over Items | — | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / The workflow iterates through a predefined list of high-intent search queries... published to WordPress as a new post. |
| Documentation | stickyNote | Workspace documentation | — | — | ## ⚡ Workflow Overview & Setup / Summary, requirements, usage, help links (https://docs.browseract.com) |
| Step 1 Explanation | stickyNote | Explains the main processing block | — | — | ### 🔍 Step 1: Batch Research, AI Editorial & Publishing / (same content as note) |
| Sticky Note | stickyNote | Video link | — | — | https://www.youtube.com/watch?v=5_bUlnRBre0 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Generate SEO articles from search queries to WordPress with BrowserAct** (or your preferred name).

2. **Add Trigger**
   - Add node **Manual Trigger** named **Execute manually**.

3. **Add query input node**
   - Add node **Set** named **Set queries (Search Input)**.
   - Add an assignment:
     - Field: `Queries`
     - Type: **Array**
     - Value (example): `["What are the best tools for automation", "Looking for a cheaper alternative to Zapier"]`
   - Connect: **Execute manually → Set queries (Search Input)**.

4. **Split the query array into items**
   - Add node **Split Out** named **Split Out**.
   - Set **Field to split out** = `Queries`.
   - Connect: **Set queries (Search Input) → Split Out**.

5. **Add loop controller**
   - Add node **Split In Batches** named **Loop Over Items**.
   - Keep default options (or set batch size to 1 if you want strict sequential processing).
   - Connect: **Split Out → Loop Over Items**.

6. **Add BrowserAct research node**
   - Add node **BrowserAct** named **Extract search engine result**.
   - Set it to run a **Workflow** (not a simple action).
   - Set **Workflow ID** to your BrowserAct workflow ID (in this workflow: `76966914945590330`).
   - In the workflow input mapping, define:
     - `input-Querry` = expression `{{ $json.Queries }}`
   - Create/attach **BrowserAct API credentials** in n8n.
   - Connect: **Loop Over Items → Extract search engine result**.

7. **Add OpenRouter model**
   - Add node **OpenRouter Chat Model** (LangChain) named **OpenRouter Chat Model**.
   - Choose model: **`openai/gpt-5`** (or another OpenRouter-available model).
   - Create/attach **OpenRouter API credentials**.

8. **Add structured output parser**
   - Add node **Structured Output Parser** (LangChain) named **Structured Output Parser**.
   - Enable **Auto-fix**.
   - Provide an example structure with keys:
     - `title` (string)
     - `article` (string)
   - Connect AI: **OpenRouter Chat Model (ai_languageModel) → Structured Output Parser (ai_languageModel)**.

9. **Add the AI agent**
   - Add node **AI Agent** (LangChain) named **Analyze the results and generate a post**.
   - Set prompt type to **Define**.
   - Set the **Text** (user prompt) to:  
     `Scrapped Data : {{ $json.output.string }}`
   - Paste the **System Message** requirements (role, sanitization, strict JSON output, HTML structure).
   - Connect AI:
     - **OpenRouter Chat Model (ai_languageModel) → Agent (ai_languageModel)**
     - **Structured Output Parser (ai_outputParser) → Agent (ai_outputParser)**
   - Connect main: **Extract search engine result → Agent**.

10. **Add WordPress publish node**
   - Add node **WordPress** named **Create a post**.
   - Operation: **Create Post**.
   - Title: expression `{{ $json.output.title }}`
   - Content/body: expression `{{ $json.output.article }}`
   - Create/attach **WordPress credentials** (commonly site URL + application password/user, depending on your n8n setup).
   - Connect: **Agent → Create a post**.

11. **Close the loop**
   - Connect: **Create a post → Loop Over Items** (to process the next query).

12. **(Optional) Slack notification**
   - Add node **Slack** named **Send completion notification**.
   - Configure channel + message text.
   - Connect from **Loop Over Items → Send completion notification**.
   - Leave it disabled unless you want notifications.
   - Create/attach **Slack credentials** with permission to post in the chosen channel.

13. **Add sticky notes (optional, for maintainability)**
   - Add a documentation note with requirements and BrowserAct links.
   - Add a note containing the video link: `https://www.youtube.com/watch?v=5_bUlnRBre0`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Workflow Overview & Setup… Requirements: BrowserAct, OpenRouter (GPT-5), WordPress, Slack. Mandatory: BrowserAct API (Template: Programmatic SEO Data Pipeline).” | Internal documentation sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | https://www.youtube.com/watch?v=5_bUlnRBre0 |