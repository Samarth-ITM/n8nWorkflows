Post curated remote job listings to Slack with BrowserAct and OpenRouter

https://n8nworkflows.xyz/workflows/post-curated-remote-job-listings-to-slack-with-browseract-and-openrouter-13382


# Post curated remote job listings to Slack with BrowserAct and OpenRouter

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs weekly to scrape remote job listings from selected job boards (via BrowserAct), uses an LLM (via OpenRouter) to analyze and rank jobs against a user “resume/config” profile, formats a curated Slack digest (mrkdwn), and posts the digest to a Slack channel.

**Target use cases:**
- Automate weekly job prospecting for a specific role/location/salary preference.
- Produce an AI-curated shortlist (2–3 best matches) instead of raw scraped dumps.
- Deliver digest messages safely chunked for Slack message limits.

### 1.1 Scheduling & User Profile Configuration
Defines when the workflow runs and what the user is looking for (location, skill, salary expectation, personal preferences, target job sites).

### 1.2 Site Iteration (Batch Loop)
Splits the list of target job board URLs and iterates per site.

### 1.3 Scraping (BrowserAct)
Calls a BrowserAct “Job Board Aggregator” style workflow/template to fetch job postings from each target site.

### 1.4 AI Analysis + Structured Slack Message Generation
Uses an OpenRouter chat model inside an n8n LangChain Agent, with a structured output parser enforcing strict JSON output containing a `slack_message` array.

### 1.5 Slack Posting + Continue Loop
Splits the `slack_message` array into multiple Slack messages (if needed), posts each to Slack, then continues to the next site.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & User Profile Configuration

**Overview:**  
Triggers weekly and prepares a single “profile/config” item containing job preferences and the list of sites to scrape.

**Nodes involved:**
- **Weekly Trigger**
- **Add a Resume**
- **Split out target sites**

#### Node: Weekly Trigger
- **Type / role:** `Schedule Trigger` — entry point; runs the workflow on a weekly schedule.
- **Configuration (interpreted):** Runs every week at **12:00** (server/workspace timezone).
- **Outputs:** 1 execution item to **Add a Resume**.
- **Edge cases / failures:**
  - Timezone mismatch can cause unexpected trigger times.
  - If n8n instance is down at trigger time, executions may be missed depending on n8n scheduling behavior/settings.

#### Node: Add a Resume
- **Type / role:** `Set` — acts as a configuration “resume/profile” object.
- **Configuration (interpreted):** Sets these fields:
  - `Location`: `"New York"`
  - `Skill`: `"Software Engineer"`
  - `Income`: `"higher than 100$/M"` (free text; AI is expected to normalize)
  - `Details`: preference text (remote preferred, open to on-site, strengths, FE/BE)
  - `Target_Sites`: array of URLs `["https://www.dice.com/", "https://indeed.com/"]`
- **Outputs:** One item to **Split out target sites**.
- **Key variables used downstream:** `$json.Location`, `$json.Skill`, `$json.Income`, `$json.Details`, `$json.Target_Sites`.
- **Edge cases / failures:**
  - If `Target_Sites` is not a valid array, downstream split will fail.
  - Inconsistent salary text may reduce matching quality.

#### Node: Split out target sites
- **Type / role:** `Split Out` — turns `Target_Sites` array into one item per site.
- **Configuration (interpreted):**
  - `fieldToSplitOut`: `Target_Sites`
  - `include`: `allOtherFields` (keeps Location/Skill/Income/Details on each split item)
- **Inputs:** The single profile item from **Add a Resume**.
- **Outputs:** Multiple items (one per site) to **Loop Over Items**.
- **Edge cases / failures:**
  - Missing `Target_Sites` or non-array type causes runtime error.
  - Empty array results in no downstream scraping.

---

### Block 2 — Site Iteration (Batch Loop)

**Overview:**  
Processes sites one by one using batch looping. The workflow is structured so Slack posting completion triggers the next batch.

**Nodes involved:**
- **Loop Over Items**
- **Send a message to the Slack channel** (feeds back to continue loop)

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — controls iterative processing.
- **Configuration (interpreted):**
  - Uses default options (batch size not explicitly set in parameters; n8n default applies, typically 1 for iteration patterns—verify in your UI).
- **Connections:**
  - Receives items from **Split out target sites**.
  - **Output 1** is not used here.
  - **Output 2** connects to **Scrape Suitable Jobs** (meaning it proceeds with the next batch/item as controlled by the loop).
  - Receives “continue” signal from **Send a message to the Slack channel** back into **Loop Over Items** (to advance the loop).
- **Edge cases / failures:**
  - If the loop is misconfigured (batch size too large), you may scrape multiple sites at once and mix contexts.
  - If Slack posting fails and the loop continuation never happens, the workflow may stop after first site.

---

### Block 3 — Scraping (BrowserAct)

**Overview:**  
Invokes a BrowserAct workflow to scrape jobs for the current target site, using the user’s skill and location.

**Nodes involved:**
- **Scrape Suitable Jobs**

#### Node: Scrape Suitable Jobs
- **Type / role:** `BrowserAct` (`n8n-nodes-browseract.browserAct`) — runs an external BrowserAct workflow/template.
- **Configuration (interpreted):**
  - `type`: `WORKFLOW` (invokes a BrowserAct workflow by ID)
  - `workflowId`: `766193817+1234567890` (must correspond to your BrowserAct workflow)
  - `workflowConfig` mapping:
    - `input-Skill` = `{{$json.Skill}}`
    - `input-Job_Site` = `{{$json.Target_Sites}}` (note: after split, this is a single URL string)
    - `input-Location` = `{{$json.Location}}`
  - Mapping mode: “define below”; fields are optional and fall back to BrowserAct defaults if blank.
- **Credentials:** BrowserAct API credential required.
- **Inputs:** One item representing one target site + user profile fields (from **Loop Over Items**).
- **Outputs:** Scraped result is expected to include `output.string` (used later by the AI agent).
- **Edge cases / failures:**
  - BrowserAct auth/API key invalid.
  - BrowserAct workflowId wrong or not accessible to the credential.
  - Target site blocks scraping (captcha, bot protection), resulting in empty/partial output.
  - Output shape mismatch (e.g., missing `output.string`) breaks downstream AI prompt.

---

### Block 4 — AI Analysis + Structured Slack Message Generation

**Overview:**  
Combines user profile + scraped job text and asks an LLM to produce a strictly structured JSON output containing Slack-ready message chunks.

**Nodes involved:**
- **Analyze the jobs and generate a Slack message** (LangChain Agent)
- **OpenRouter Chat Model**
- **Structured Output Parser**

#### Node: OpenRouter Chat Model
- **Type / role:** `LangChain Chat Model (OpenRouter)` — provides the LLM used by the agent and parser.
- **Configuration (interpreted):**
  - Model: `openai/gpt-4o` (served via OpenRouter)
- **Credentials:** OpenRouter API credential required.
- **Connections:**
  - Connected to the agent as `ai_languageModel`.
  - Also connected to the structured parser as `ai_languageModel` (so the parser can use the same model for auto-fixing).
- **Edge cases / failures:**
  - OpenRouter key invalid / quota exceeded.
  - Model not available or renamed.
  - Latency/timeouts for large scraped payloads.

#### Node: Structured Output Parser
- **Type / role:** `LangChain Structured Output Parser` — enforces JSON schema and can auto-fix malformed model output.
- **Configuration (interpreted):**
  - `autoFix`: enabled (attempts to repair invalid JSON)
  - Schema example expects:
    ```json
    { "slack_message": ["...","..."] }
    ```
- **Connections:**
  - Receives `ai_languageModel` from **OpenRouter Chat Model**.
  - Provides `ai_outputParser` into the agent.
- **Edge cases / failures:**
  - If the model output is too malformed, auto-fix may still fail.
  - If the agent returns non-JSON or violates required key `slack_message`, parsing fails.

#### Node: Analyze the jobs and generate a Slack message
- **Type / role:** `LangChain Agent` — orchestrates the prompt + model and returns structured JSON.
- **Configuration (interpreted):**
  - Prompt is manually defined (`promptType: define`) with:
    - **User Data** pulled from `$('Loop Over Items').first().json` (Location, Skill, Income, Details)
    - **Scraped Data** including:
      - `$('Loop Over Items').first().json.Target_Sites` (the current site URL)
      - `{{$json.output.string}}` (scraped job data from BrowserAct node)
  - System message instructs:
    - Normalize salary formats and compare to threshold
    - Prioritize location matches, include remote logic and disclaimers on mismatch
    - Score by skill overlap
    - Select top 2–3 matches
    - Format for Slack mrkdwn
    - Chunk into an array of strings `slack_message`
  - `hasOutputParser`: true (the structured parser is attached)
- **Inputs:** Output from **Scrape Suitable Jobs**.
- **Outputs:** A JSON object with `output.slack_message` (as used later).
- **Key expressions / variables:**
  - `$('Loop Over Items').first().json.Location` etc. (important: relies on loop context existing)
  - `$json.output.string` from BrowserAct result
- **Edge cases / failures:**
  - If `Loop Over Items` has no items (or context lost), `.first()` may be undefined.
  - Very large `output.string` may exceed model/context limits; consider truncation or pre-filtering.
  - Salary normalization is heuristic; ambiguous salary strings can lead to poor ranking.

---

### Block 5 — Slack Posting + Continue Loop

**Overview:**  
Splits the AI-produced message array into separate items and posts each as a Slack message, then signals the loop to continue to the next target site.

**Nodes involved:**
- **Split Out**
- **Send a message to the Slack channel**
- **Loop Over Items** (loop continuation)

#### Node: Split Out
- **Type / role:** `Split Out` — splits the array `output.slack_message` into one item per Slack post.
- **Configuration (interpreted):**
  - `fieldToSplitOut`: `output.slack_message`
- **Inputs:** Output of the agent, expected to contain `output.slack_message` as an array.
- **Outputs:** Each item contains one element from `output.slack_message`.
- **Edge cases / failures:**
  - If `output.slack_message` is missing or not an array, this node fails.
  - Empty array results in no Slack message (and thus loop may not continue).

#### Node: Send a message to the Slack channel
- **Type / role:** `Slack` — posts a message to a specific Slack channel.
- **Configuration (interpreted):**
  - Operation: send message to **channel**
  - Channel: `C09KLV9DJSX` (cached name: `all-browseract-workflow-test`)
  - Message text: `={{ $json["output.slack_message"] }}`
    - After the previous Split Out, each item should provide a single string at `output.slack_message`.
- **Credentials:** Slack API credential required.
- **Outputs / connections:**
  - On success, connects back to **Loop Over Items** to continue the batch loop.
- **Edge cases / failures:**
  - Slack token missing scopes (typically `chat:write`).
  - Channel ID invalid or bot not invited to channel.
  - If the message string is too long, Slack may reject it.
  - If no Slack message is sent (because split produced 0 items), the loop continuation won’t occur.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly entry point | — | Add a Resume | ### 🎯 Step 1: User Profile & Targeting \nThe workflow begins by defining your search parameters. \nThe **Add a Resume** node acts as a configuration file, storing your bio, salary expectations, and the list of job boards to monitor. |
| Add a Resume | Set | Stores user criteria and target job boards | Weekly Trigger | Split out target sites | ## ⚡ Workflow Overview & Setup \nSummary: This automation acts as your personal AI headhunter... \nRequirements: Credentials: BrowserAct, OpenRouter (GPT-4), Slack. Mandatory: BrowserAct API (Template: **Job Board Aggregator**) \nHow to Use: update Location/Skill/Income/Details and Target_Sites. \nLinks: https://docs.browseract.com |
| Split out target sites | Split Out | Creates one item per target site URL | Add a Resume | Loop Over Items | ### 🎯 Step 1: User Profile & Targeting \nThe workflow begins by defining your search parameters... |
| Loop Over Items | Split In Batches | Iterates over sites/batches; loop controller | Split out target sites; Send a message to the Slack channel | Scrape Suitable Jobs | ### 🕵️ Step 2: Intelligent Scraping & AI Matching & Filtering \nBrowserAct iterates through your target sites... |
| Scrape Suitable Jobs | BrowserAct | Scrapes job postings from a site via BrowserAct workflow | Loop Over Items | Analyze the jobs and generate a Slack message | ### 🕵️ Step 2: Intelligent Scraping & AI Matching & Filtering \nBrowserAct iterates through your target sites... |
| OpenRouter Chat Model | LangChain Chat Model (OpenRouter) | LLM backend for agent and parser | — | Analyze the jobs and generate a Slack message; Structured Output Parser | ### 🕵️ Step 2: Intelligent Scraping & AI Matching & Filtering \nBrowserAct iterates through your target sites... |
| Structured Output Parser | LangChain Structured Output Parser | Enforces JSON output structure; auto-fixes | OpenRouter Chat Model (as ai_languageModel) | Analyze the jobs and generate a Slack message (as ai_outputParser) | ### 🕵️ Step 2: Intelligent Scraping & AI Matching & Filtering \nBrowserAct iterates through your target sites... |
| Analyze the jobs and generate a Slack message | LangChain Agent | Ranks jobs vs profile; generates Slack mrkdwn chunks | Scrape Suitable Jobs; OpenRouter Chat Model; Structured Output Parser | Split Out | ### 🕵️ Step 2: Intelligent Scraping & AI Matching & Filtering \nBrowserAct iterates through your target sites... |
| Split Out | Split Out | Splits `output.slack_message` array into items | Analyze the jobs and generate a Slack message | Send a message to the Slack channel |  |
| Send a message to the Slack channel | Slack | Posts each chunk to Slack channel | Split Out | Loop Over Items |  |
| Documentation | Sticky Note | Informational note | — | — | ## ⚡ Workflow Overview & Setup \nLinks: https://docs.browseract.com |
| Step 1 Explanation | Sticky Note | Informational note | — | — | ### 🎯 Step 1: User Profile & Targeting ... |
| Step 2 Explanation | Sticky Note | Informational note | — | — | ### 🕵️ Step 2: Intelligent Scraping & AI Matching & Filtering ... |
| Sticky Note | Sticky Note | Video link | — | — | https://www.youtube.com/watch?v=IvgBQfmvDNI |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1. Add node: **Schedule Trigger**
   2. Set interval to **Weekly**, trigger hour **12** (adjust timezone as needed).

2. **Create User Profile Config**
   1. Add node: **Set** named **Add a Resume**
   2. Add fields:
      - `Location` (String)
      - `Skill` (String)
      - `Income` (String)
      - `Details` (String)
      - `Target_Sites` (Array of Strings, e.g. `https://www.dice.com/`, `https://indeed.com/`)
   3. Connect: **Weekly Trigger → Add a Resume**

3. **Split Target Sites**
   1. Add node: **Split Out** named **Split out target sites**
   2. `Field to Split Out`: `Target_Sites`
   3. Enable keeping other fields (Include: **All Other Fields**)
   4. Connect: **Add a Resume → Split out target sites**

4. **Add Loop Controller**
   1. Add node: **Split In Batches** named **Loop Over Items**
   2. Keep default options (or set batch size to **1** explicitly if you want strict one-site-at-a-time processing)
   3. Connect: **Split out target sites → Loop Over Items**

5. **Configure BrowserAct Scraping**
   1. Add node: **BrowserAct** named **Scrape Suitable Jobs**
   2. Set to run a BrowserAct **WORKFLOW** by ID (your BrowserAct workflow/template ID).
   3. Map inputs (names must match your BrowserAct workflow inputs):
      - `Job_Site` = `{{$json.Target_Sites}}`
      - `Location` = `{{$json.Location}}`
      - `Skill` = `{{$json.Skill}}`
   4. Credentials:
      - Create **BrowserAct API** credentials (API key).
   5. Connect: **Loop Over Items (loop output) → Scrape Suitable Jobs**
      - Ensure you connect the looping output that advances batches (in your JSON it’s output #2).

6. **Add LLM Model (OpenRouter)**
   1. Add node: **OpenRouter Chat Model**
   2. Choose model: `openai/gpt-4o` (or your preferred OpenRouter model)
   3. Credentials:
      - Create **OpenRouter API** credentials (API key)

7. **Add Structured Output Parser**
   1. Add node: **Structured Output Parser**
   2. Enable **Auto-fix**
   3. Provide a schema/example requiring:
      - `slack_message`: Array of Strings

8. **Add LangChain Agent**
   1. Add node: **AI Agent** named **Analyze the jobs and generate a Slack message**
   2. Set prompt to combine:
      - User criteria from the loop item (Location/Skill/Income/Details)
      - Scraped output from BrowserAct (ensure you reference the field your BrowserAct node returns; in this workflow it’s `output.string`)
   3. Set the **system message** to:
      - Normalize salary, match location (remote logic), score skills, pick top 2–3
      - Format Slack mrkdwn
      - Output strict JSON `{ "slack_message": ["..."] }`
   4. Connect AI ports:
      - **OpenRouter Chat Model → Agent** (as language model)
      - **OpenRouter Chat Model → Structured Output Parser** (so parser can auto-fix using the same model)
      - **Structured Output Parser → Agent** (as output parser)
   5. Connect main flow:
      - **Scrape Suitable Jobs → Analyze the jobs and generate a Slack message**

9. **Split Slack Message Chunks**
   1. Add node: **Split Out** named **Split Out**
   2. `Field to Split Out`: `output.slack_message`
   3. Connect: **Agent → Split Out**

10. **Post to Slack**
    1. Add node: **Slack** named **Send a message to the Slack channel**
    2. Operation: **Send message to channel**
    3. Select channel by ID (e.g., `C09KLV9DJSX`) and ensure the bot is in the channel
    4. Text expression: `{{$json["output.slack_message"]}}`
    5. Credentials:
       - Create Slack API credentials (OAuth or token) with `chat:write`
    6. Connect: **Split Out → Send a message to the Slack channel**

11. **Close the Loop**
    1. Connect: **Send a message to the Slack channel → Loop Over Items**
       - This “continue” connection is required so the loop advances to the next target site after posting.

12. **(Optional) Add Notes**
    - Add sticky notes for documentation and links, including:
      - https://docs.browseract.com
      - https://www.youtube.com/watch?v=IvgBQfmvDNI

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | https://www.youtube.com/watch?v=IvgBQfmvDNI |
| Workflow intent note: weekly scrape → AI scoring → Slack digest | Sticky notes “Documentation”, “Step 1 Explanation”, “Step 2 Explanation” within the workflow canvas |