Scrape LinkedIn comments and score lead intent using ConnectSafely, Azure OpenAI, and Google Sheets

https://n8nworkflows.xyz/workflows/scrape-linkedin-comments-and-score-lead-intent-using-connectsafely--azure-openai--and-google-sheets-13185


# Scrape LinkedIn comments and score lead intent using ConnectSafely, Azure OpenAI, and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow discovers LinkedIn posts that mention lead-generation pain points, scrapes all comments from those posts, scores each commenter’s “buying intent” with an Azure OpenAI model, and appends the ranked results to a Google Sheet for follow-up.

**Target use cases:**
- B2B lead sourcing from public LinkedIn conversations
- Identifying problem-aware or high-intent commenters at scale
- Building a lightweight lead list in Google Sheets for outreach sequencing

**Logical blocks (by dependency):**
1.1 **Trigger / Start** → manual execution  
1.2 **Search LinkedIn posts (SerpAPI)** → Google search restricted to `linkedin.com/posts`  
1.3 **Parse & filter search results** → normalize results and keep valid post URLs  
1.4 **Scrape comments (ConnectSafely)** → fetch all comments per post  
1.5 **Flatten comment arrays** → one n8n item per comment  
1.6 **AI intent scoring (Azure OpenAI via LangChain Agent)** → strict JSON output  
1.7 **Parse AI output & persist** → JSON parse and append to Google Sheets

---

## 2. Block-by-Block Analysis

### 2.1 Trigger / Start
**Overview:** Starts the workflow manually from the n8n UI. Useful for testing and ad-hoc runs.

**Nodes involved:**
- **When clicking ‘Execute workflow’**

**Node details:**
- **When clicking ‘Execute workflow’**
  - **Type / role:** `Manual Trigger` — entry point
  - **Configuration:** No parameters
  - **Input / output:** No input; outputs a single empty item into the workflow
  - **Connections:** → `Search LinkedIn Posts via SerpAPI`
  - **Edge cases:** None (only runs when manually executed)

---

### 2.2 Search LinkedIn posts (SerpAPI)
**Overview:** Queries Google (via SerpAPI) for LinkedIn post URLs that match pain-point keywords.

**Nodes involved:**
- **Search LinkedIn Posts via SerpAPI**

**Node details:**
- **Search LinkedIn Posts via SerpAPI**
  - **Type / role:** `SerpAPI` node — external search API call
  - **Configuration choices:**
    - Query (`q`):  
      `site:linkedin.com/posts ("Missed leads" OR "Losing leads" OR "Low conversion rate" OR "Leads not converting")`
    - Uses default request options and no extra fields
  - **Credentials:** SerpAPI account required
  - **Input / output:** Receives trigger item; outputs SerpAPI search response JSON (includes arrays like `organic_results`, etc.)
  - **Connections:** → `Parse & Filter Search Results`
  - **Failure modes / edge cases:**
    - Invalid/missing API key → immediate node failure (401/403)
    - Quota exceeded / rate limiting
    - SerpAPI response shape changes (downstream parsing assumes certain fields exist)
    - Query may return non-LinkedIn results despite `site:` filter

---

### 2.3 Parse & filter search results
**Overview:** Consolidates multiple result arrays from SerpAPI, detects platform/link patterns, flags “problem posts” via keyword scanning, and filters out items without URLs.

**Nodes involved:**
- **Parse & Filter Search Results**

**Node details:**
- **Parse & Filter Search Results**
  - **Type / role:** `Code` node — transforms SerpAPI response into normalized items
  - **Configuration choices (interpreted):**
    - Merges results from: `organic_results`, `inline_results`, `discussions`, `forums`
    - Extracts:
      - `post_url` from `post.link`
      - `author_name` from `post.source` (strips `"LinkedIn · "` / `"Reddit · "`)
      - LinkedIn detection if URL matches `linkedin.com/posts/`
      - Also includes Reddit detection (even though the SerpAPI query is LinkedIn-focused)
    - “Problem detection”:
      - Builds `text = (title + snippet).toLowerCase()`
      - Checks for presence of keywords (e.g., `problem`, `struggling`, `low`, `miss`, `failing`, etc.)
      - Outputs:
        - `problem_keywords_detected` (array)
        - `is_problem_post` (boolean)
    - Filters out outputs with missing `post_url`
  - **Key variables / fields produced:**
    - `platform`, `author_name`, `username`, `post_id`, `post_url`, `post_title`, `post_snippet`, `engagement_hint`, `problem_keywords_detected`, `is_problem_post`
  - **Input / output:**
    - Input: single SerpAPI response item
    - Output: multiple items (one per result)
  - **Connections:** → `Fetch Post Comments via ConnectSafely`
  - **Failure modes / edge cases:**
    - If SerpAPI returns empty arrays, output becomes empty (workflow ends without error but produces no leads)
    - The LinkedIn URL regex for extracting `username` and `postId` may fail on some LinkedIn URL variants
    - `is_problem_post` is computed but not used downstream; irrelevant items still proceed unless you add a filter node

---

### 2.4 Scrape comments (ConnectSafely)
**Overview:** Fetches *all* comments for each LinkedIn post URL.

**Nodes involved:**
- **Fetch Post Comments via ConnectSafely**

**Node details:**
- **Fetch Post Comments via ConnectSafely**
  - **Type / role:** `ConnectSafely LinkedIn` node — LinkedIn data retrieval via ConnectSafely API
  - **Configuration choices:**
    - Operation: `getAllPostComments`
    - `postUrl`: expression `={{ $json.post_url }}`
    - `accountId`: hard-coded `695ce64a09c18d6bbbe90ed0`
  - **Credentials:** ConnectSafely API credential required
  - **Input / output:**
    - Input: normalized post items (must contain `post_url`)
    - Output: per post, a JSON payload including:
      - `postUrl`, `postDetails` (e.g., `content`, `activityUrn`), and `comments` array
  - **Connections:** → `Flatten Comments into Rows`
  - **Failure modes / edge cases:**
    - Invalid/expired API key or wrong accountId
    - LinkedIn URL unsupported/private/deleted → API may return empty comments or error
    - Large comment threads can cause long runtimes/timeouts
    - Inconsistent fields in `comments` objects (handled partially downstream)

---

### 2.5 Flatten comment arrays
**Overview:** Converts each post’s comments array into individual items so each comment can be scored independently by AI.

**Nodes involved:**
- **Flatten Comments into Rows**

**Node details:**
- **Flatten Comments into Rows**
  - **Type / role:** `Code` node — array flattening / item expansion
  - **Configuration choices (interpreted):**
    - Iterates over incoming items (posts)
    - For each comment, emits one output item with:
      - Post: `postUrl`, `postContent` (`post.postDetails?.content`), `activityUrn`
      - Comment: `commentId`, `commentText`, `commenterName` (`comment.authorName`), `commenterProfileUrl`, `commenterUrn`
      - Metadata: `createdAt`, `likeCount`, `replyCount`, `isProfile` (`comment.hasProfile`)
  - **Input / output:**
    - Input: items containing `comments` arrays
    - Output: many items (one per comment); if `comments` is empty, emits nothing for that post
  - **Connections:** → `AI Intent Detection Agent`
  - **Failure modes / edge cases:**
    - If ConnectSafely returns no `comments` field, code defaults to `[]` (safe)
    - If comments contain unexpected nulls, may emit items with missing `commentText`/`commenterName` (AI scoring quality drops)

---

### 2.6 AI intent scoring (LangChain Agent + Azure OpenAI)
**Overview:** An LLM agent scores each comment’s lead intent (0–100) and assigns an intent label. Output is required to be strict JSON.

**Nodes involved:**
- **AI Intent Detection Agent**
- **Azure OpenAI GPT-4o-mini**

**Node details:**
- **Azure OpenAI GPT-4o-mini**
  - **Type / role:** `Azure OpenAI Chat Model` (LangChain) — provides the LLM to the agent
  - **Configuration choices:**
    - Model: `gpt-4o-mini` (Azure deployment must map to this model name/deployment)
  - **Credentials:** Azure OpenAI credential required (endpoint, key, deployment config)
  - **Connections:** This node is connected via the agent’s **AI languageModel** input:
    - `Azure OpenAI GPT-4o-mini` → (ai_languageModel) → `AI Intent Detection Agent`
  - **Failure modes / edge cases:**
    - Wrong deployment/model name, invalid key, denied quota
    - Content filtering or policy blocks (less likely here, but possible)
    - Latency/timeouts with many comments

- **AI Intent Detection Agent**
  - **Type / role:** `LangChain Agent` — orchestrates prompt + model and returns the model output
  - **Configuration choices:**
    - Prompt includes:
      - `postUrl`, `postContent`, `commenterName`, `commentText`
    - Output contract: **JSON only** with schema:
      - `postUrl`, `leadName`, `comment`, `intentScore` (number), `intentLabel` (one of specified categories)
    - System message defines:
      - Conservative scoring rules and ranges
      - Label taxonomy: `no-intent | passive-interest | problem-aware | solution-aware | high-intent`
      - Note: system message also says: “If intent is below 30… mark it as `low-intent`.”
        - This conflicts with the declared allowed labels (does not include `low-intent`)
  - **Key expressions:**
    - Uses n8n expressions like `{{ $json.commentText }}` inside prompt
  - **Input / output:**
    - Input: one item per comment from flattening step
    - Output: agent response placed into `item.json.output` (expected by downstream parser)
  - **Connections:** → `Parse AI Intent Output`
  - **Failure modes / edge cases:**
    - Model returns non-JSON or wrapped JSON (parser will drop the record)
    - JSON schema mismatch (missing fields, wrong types)
    - Conflicting label instruction can lead to unexpected `intentLabel` values and downstream sheet inconsistency
  - **Version-specific notes:**
    - Node type `@n8n/n8n-nodes-langchain.agent` v3 requires n8n with LangChain nodes installed/enabled (n8n v1.x with the LangChain package available)

---

### 2.7 Parse AI output & persist to Google Sheets
**Overview:** Parses the AI JSON output safely (skipping invalid records) and appends structured lead rows to a target Google Sheet.

**Nodes involved:**
- **Parse AI Intent Output**
- **Save Leads to Google Sheet**

**Node details:**
- **Parse AI Intent Output**
  - **Type / role:** `Code` node — validates and parses model output
  - **Configuration choices (interpreted):**
    - Skips items where `item.json.output` is missing
    - Tries `JSON.parse(item.json.output)`
    - If parsing fails: silently skips that record
    - Emits normalized fields:
      - `postUrl`, `leadName`, `comment`, `intentScore`, `intentLabel`
  - **Input / output:**
    - Input: agent outputs
    - Output: parsed items suitable for Google Sheets append
  - **Connections:** → `Save Leads to Google Sheet`
  - **Failure modes / edge cases:**
    - Silent skipping can hide systemic prompt/model issues (e.g., model consistently returning non-JSON)
    - If AI returns `intentScore` as string, it will be passed through; sheet column type handling may vary

- **Save Leads to Google Sheet**
  - **Type / role:** `Google Sheets` node — persistence layer
  - **Operation:** `append`
  - **Target:**
    - Document: Spreadsheet “Post Scraping” (ID `1hXDNE90IFbXLgyLt2XwSHgI_9sASzTSM9LAA4cUDFj4`)
    - Sheet tab: “Sheet1” (`gid=0`)
  - **Column mapping (define below):**
    - `post_url` ← `{{$json.postUrl}}`
    - `lead name` ← `{{$json.leadName}}`
    - `comment` ← `{{$json.comment}}`
    - `intent score` ← `{{$json.intentScore}}`
    - `intent label` ← `{{$json.intentLabel}}`
  - **Credentials:** Google Sheets OAuth2 required
  - **Failure modes / edge cases:**
    - Missing permissions to spreadsheet
    - Sheet/tab renamed; column headers must match exactly (including spaces: `lead name`, `intent score`, `intent label`)
    - Appending many rows may hit Google API rate limits

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual entry point | — | Search LinkedIn Posts via SerpAPI |  |
| Search LinkedIn Posts via SerpAPI | n8n-nodes-serpapi.serpApi | Google search for LinkedIn posts | When clicking ‘Execute workflow’ | Parse & Filter Search Results | ⚠️ **SerpAPI Key Required**<br><br>This node calls the Google Search API. A valid SerpAPI credential must be configured, or every execution will fail at this step. |
| Parse & Filter Search Results | n8n-nodes-base.code | Normalize/filter search results into post URLs | Search LinkedIn Posts via SerpAPI | Fetch Post Comments via ConnectSafely | ## Search & Scrape<br><br>Triggers a Google search for pain-point posts on LinkedIn, parses the results, and fetches comments from each matched post via ConnectSafely. |
| Fetch Post Comments via ConnectSafely | n8n-nodes-connectsafely-ai.connectSafelyLinkedIn | Scrape all comments from each LinkedIn post | Parse & Filter Search Results | Flatten Comments into Rows | ⚠️ **ConnectSafely Credential Required**<br><br>Fetching post comments depends on an active ConnectSafely account. An invalid or expired API key will block the entire scrape. |
| Flatten Comments into Rows | n8n-nodes-base.code | Expand comment arrays into one item per comment | Fetch Post Comments via ConnectSafely | AI Intent Detection Agent | ## Comment Processing<br><br>Receives raw comment arrays from each post and flattens them into individual rows, one per commenter, ready for AI scoring. |
| Azure OpenAI GPT-4o-mini | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | LLM provider for the agent | — | AI Intent Detection Agent (ai_languageModel) | ⚠️ **Azure OpenAI Credential Required**<br><br>The AI intent agent uses gpt-4o-mini. Ensure the Azure deployment name matches and the credential is active; otherwise scoring will fail. |
| AI Intent Detection Agent | @n8n/n8n-nodes-langchain.agent | Prompt + scoring logic; returns JSON | Flatten Comments into Rows | Parse AI Intent Output | ## AI Intent Scoring<br><br>Each comment is analysed by an AI agent that returns a 0–100 intent score and a label (no-intent → high-intent). Output is parsed into clean rows. |
| Parse AI Intent Output | n8n-nodes-base.code | Parse JSON-only AI output into structured fields | AI Intent Detection Agent | Save Leads to Google Sheet |  |
| Save Leads to Google Sheet | n8n-nodes-base.googleSheets | Append scored leads to a spreadsheet | Parse AI Intent Output | — |  |

Additional sticky note present (applies globally, not tied to a single node):  
- **Main Overview** (explains end-to-end flow and setup requirements)

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **LinkedIn Post Scraping & Lead Intent Scorer** (or your preferred name)

2) **Add Trigger**
- Add node: **Manual Trigger**
- Keep defaults
- Connect: Manual Trigger → SerpAPI node

3) **Add SerpAPI search**
- Add node: **SerpAPI** (Google Search via SerpAPI)
- Set **Query (q)** to:  
  `site:linkedin.com/posts ("Missed leads" OR "Losing leads" OR "Low conversion rate" OR "Leads not converting")`
- Credentials:
  - Create/select **SerpAPI credential**
  - Paste API key
- Connect: SerpAPI → “Parse & Filter Search Results”

4) **Add “Parse & Filter Search Results” (Code node)**
- Add node: **Code**
- Paste logic that:
  - Reads `items[0].json`
  - Merges `organic_results`, `inline_results`, `discussions`, `forums`
  - Maps each result to a normalized item with `post_url` and metadata
  - Filters out items without `post_url`
- Connect: Code → “Fetch Post Comments via ConnectSafely”

5) **Add ConnectSafely LinkedIn comments fetch**
- Add node: **ConnectSafely LinkedIn**
- Operation: **Get All Post Comments** (`getAllPostComments`)
- Parameters:
  - **postUrl**: `={{ $json.post_url }}`
  - **accountId**: set to your ConnectSafely account ID
- Credentials:
  - Create/select **ConnectSafely API** credential
- Connect: ConnectSafely → “Flatten Comments into Rows”

6) **Add “Flatten Comments into Rows” (Code node)**
- Add node: **Code**
- Configure to:
  - Loop over `items`
  - For each post item, iterate `post.comments || []`
  - Emit one output item per comment with fields:
    - `postUrl`, `postContent`, `activityUrn`
    - `commentText`, `commenterName`, etc.
- Connect: Code → “AI Intent Detection Agent”

7) **Add Azure OpenAI chat model**
- Add node: **Azure OpenAI Chat Model** (LangChain)
- Model/deployment: **gpt-4o-mini** (ensure your Azure deployment name/settings match what n8n expects)
- Credentials:
  - Create/select **Azure OpenAI** credential (endpoint, API key, resource, deployment as required by your n8n version)
- Do **not** connect via main; this connects to the agent’s **AI languageModel** input.

8) **Add the AI Agent**
- Add node: **AI Agent** (LangChain Agent)
- Prompt (text) should include:
  - Post URL, post content, commenter name, comment text (using n8n expressions)
  - Require **JSON-only** output with keys: `postUrl`, `leadName`, `comment`, `intentScore`, `intentLabel`
- System message:
  - Define scoring rubric (0–100)
  - Define allowed labels
- Connect:
  - Main input: Flatten Comments into Rows → AI Agent
  - AI language model input: Azure OpenAI Chat Model → AI Agent (ai_languageModel output into the agent)

9) **Add “Parse AI Intent Output” (Code node)**
- Add node: **Code**
- Implement:
  - If `item.json.output` missing → skip
  - `JSON.parse(item.json.output)` in try/catch
  - On parse fail → skip (or optionally route to an error branch)
  - Emit fields: `postUrl`, `leadName`, `comment`, `intentScore`, `intentLabel`
- Connect: AI Agent → Parse AI Intent Output

10) **Add Google Sheets append**
- Add node: **Google Sheets**
- Operation: **Append**
- Select your spreadsheet (Document ID) and sheet/tab
- Ensure the sheet has headers exactly matching:
  - `post_url`, `lead name`, `comment`, `intent score`, `intent label`
- Map columns:
  - `post_url` → `={{ $json.postUrl }}`
  - `lead name` → `={{ $json.leadName }}`
  - `comment` → `={{ $json.comment }}`
  - `intent score` → `={{ $json.intentScore }}`
  - `intent label` → `={{ $json.intentLabel }}`
- Credentials:
  - Create/select **Google Sheets OAuth2** credential with access to that spreadsheet
- Connect: Parse AI Intent Output → Google Sheets

11) **(Recommended) Adjust the label inconsistency**
- In the agent system message, either:
  - Add `low-intent` to the allowed labels list, **or**
  - Remove the instruction to output `low-intent` and keep labels strictly to the 5-category taxonomy  
This prevents downstream analytics and filtering issues in Sheets.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow scrapes LinkedIn posts found via SerpAPI, pulls comments via ConnectSafely, scores intent via Azure OpenAI, saves to Google Sheets. | From sticky note “Main Overview” |
| Google Sheets must exist and include columns: `post_url`, `lead name`, `comment`, `intent score`, `intent label`. | Setup requirement in “Main Overview” |
| Customization ideas: adjust SerpAPI keywords; tweak intent prompt/thresholds; change spreadsheet ID/sheet name. | From “Main Overview” |
| Spreadsheet used in the provided configuration: “Post Scraping” | https://docs.google.com/spreadsheets/d/1hXDNE90IFbXLgyLt2XwSHgI_9sASzTSM9LAA4cUDFj4/edit?usp=drivesdk |