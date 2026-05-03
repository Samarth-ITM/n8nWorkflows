Subsidy hunter: GPT-4o analysis & scoring with Google News & Chatwork alerts

https://n8nworkflows.xyz/workflows/subsidy-hunter--gpt-4o-analysis---scoring-with-google-news---chatwork-alerts-11155


# Subsidy hunter: GPT-4o analysis & scoring with Google News & Chatwork alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow monitors Japanese subsidy/grant information (補助金・助成金) daily, aggregates items from **Google News** and **two RSS feeds**, fetches each article’s content, uses **GPT-4o** to extract structured subsidy details and compute an **importance score (1–10)** plus **urgency**, prevents duplicates via **Google Sheets**, then sends formatted alerts to **Chatwork** and logs results to Sheets.

**Target use cases:**
- SME-focused teams tracking upcoming fiscal-year subsidy programs (e.g., 2026/令和8年度)
- Grant consultants, corporate planning, back-office teams wanting daily scoring + alerting
- Automated intake pipeline feeding a Sheet as a canonical “known items” database

### 1.1 Setup & Trigger
Runs on a daily schedule (9 AM) and defines configuration variables (queries, RSS URLs, Chatwork token/room, Sheets doc ID).

### 1.2 Data Aggregation
Pulls latest items from:
- Google News search results page
- J-Net21 RSS
- Mirasapo Plus RSS  
Then merges into a single item stream.

### 1.3 AI Analysis
Normalizes each item to a URL + metadata, fetches full page HTML/text, then uses an AI Agent (GPT-4o + structured parser) to produce consistent JSON fields including `importanceScore` and `urgency`.

### 1.4 Logic, Deduplication, Persistence & Notification
Checks Google Sheets for duplicates, filters new items, tags as High/Normal priority by score threshold, appends/updates in Google Sheets, and sends a Chatwork message.

---

## 2. Block-by-Block Analysis

### Block 1 — Setup & Trigger

**Overview:**  
Starts the workflow daily at 9 AM and centralizes all runtime configuration parameters in a single Set node.

**Nodes involved:**
- Sticky Note Main
- Sticky Note Setup
- Daily Schedule (9AM)
- Workflow Configuration

#### Sticky Note Main (sticky note)
- **Type/role:** Sticky Note for documentation.
- **Configuration choices:** Provides overview, setup steps, required headers for Google Sheets, credential requirements.
- **Connections:** None (documentation only).
- **Failure/edge cases:** None.

#### Sticky Note Setup (sticky note)
- **Type/role:** Sticky Note describing block 1 purpose.
- **Connections:** None.
- **Failure/edge cases:** None.

#### Daily Schedule (9AM)
- **Type/role:** `scheduleTrigger` entry point.
- **Configuration choices:** Runs daily at **09:00** (server time / workflow timezone depending on n8n instance settings).
- **Outputs:** Triggers **Workflow Configuration**.
- **Edge cases / failures:**
  - Timezone mismatch (expected local 9 AM vs server UTC).
  - Missed runs if n8n is down at trigger time (depends on n8n scheduling behavior).

#### Workflow Configuration
- **Type/role:** `set` node used as a configuration registry.
- **Key fields set:**
  - `googleNewsSearchQuery`: `2026年度 補助金 OR 助成金 OR 令和8年度`
  - `jnet21RssUrl`: `https://j-net21.smrj.go.jp/snavi/support/support.xml`
  - `mirasapoPlusRssUrl`: `https://mirasapo-plus.go.jp/subsidy/feed/`
  - `chatworkRoomId`: placeholder
  - `chatworkApiToken`: placeholder
  - `googleSheetsId`: placeholder
- **Configuration choices:**
  - “Include other fields” enabled, so the trigger fields (if any) are preserved.
- **Outputs:** Fans out to:
  - Search Google News
  - Read J-Net21 RSS
  - Read Mirasapo RSS
- **Edge cases / failures:**
  - Placeholders not replaced → downstream auth/URL failures.
  - Mis-typed Sheet ID → Google Sheets node errors.

---

### Block 2 — Data Aggregation

**Overview:**  
Collects subsidy-related items from three sources and merges them into one list for downstream processing.

**Nodes involved:**
- Sticky Note Aggregation
- Search Google News
- Read J-Net21 RSS
- Read Mirasapo RSS
- Merge All Sources

#### Sticky Note Aggregation (sticky note)
- **Role:** Documents aggregation logic.
- **Connections:** None.

#### Search Google News
- **Type/role:** `httpRequest` to fetch Google News search results HTML.
- **Configuration choices:**
  - URL built dynamically:
    - `https://www.google.com/search?q=<encoded query>&tbm=nws&num=20`
  - Sends a desktop browser `User-Agent` header to reduce bot blocking.
  - `onError: continueRegularOutput` to avoid stopping workflow if Google blocks/rate-limits.
- **Key expressions:**
  - Uses `encodeURIComponent($('Workflow Configuration').first().json.googleNewsSearchQuery)`
- **Outputs:** To **Merge All Sources** (input 0).
- **Edge cases / failures:**
  - Google may return CAPTCHA/blocked page (HTML not parseable later).
  - Response format defaults (not explicitly set here) can vary; typically n8n tries to parse JSON if possible, otherwise returns string/binary depending on node settings.
  - Legal/ToS considerations for scraping Google HTML; consider using a news API if needed.

#### Read J-Net21 RSS
- **Type/role:** `rssFeedRead` fetches and parses RSS into items.
- **Configuration choices:** URL from configuration.
- **Error handling:** `onError: continueRegularOutput` (workflow continues if feed fails).
- **Outputs:** To **Merge All Sources** (input 1).
- **Edge cases / failures:**
  - Feed downtime, invalid XML, network timeouts.

#### Read Mirasapo RSS
- **Type/role:** `rssFeedRead` for the Mirasapo Plus feed.
- **Configuration choices:** URL from configuration.
- **Error handling:** `onError: continueErrorOutput` (different from others).
  - This can route failures to an error output internally; however, in this workflow wiring, only the main output is merged, so errors may effectively drop items.
- **Outputs:** To **Merge All Sources** (input 2).
- **Edge cases / failures:**
  - Same as RSS above.
  - Inconsistent error strategy vs other sources may cause silent loss of items.

#### Merge All Sources
- **Type/role:** `merge` node combining three inputs into one stream.
- **Configuration choices:** `numberInputs: 3`
- **Inputs:**
  - Input 0: Google News HTTP output
  - Input 1: J-Net21 RSS items
  - Input 2: Mirasapo RSS items
- **Outputs:** To **Extract URLs**.
- **Edge cases / failures:**
  - Data shape mismatch is expected (HTML from Google vs structured RSS JSON). The next node must normalize.

---

### Block 3 — AI Analysis

**Overview:**  
Normalizes aggregated items to URLs, fetches article content, then uses GPT-4o with a structured output parser to extract subsidy attributes and scoring.

**Nodes involved:**
- Sticky Note Analysis
- Extract URLs
- Fetch Article Content
- AI Scoring Agent
- OpenAI GPT-4o
- Structured Output Parser

#### Sticky Note Analysis (sticky note)
- **Role:** Documents AI steps: URL extraction, content fetch, GPT-4o analysis/scoring.
- **Connections:** None.

#### Extract URLs
- **Type/role:** `code` node that sanitizes and standardizes incoming items.
- **What it does:**
  - Iterates all incoming items (`$input.all()`).
  - Attempts to find a usable link from: `link`, `url`, `guid`, `originallink`.
  - Normalizes arrays to first element, trims strings, drops empty.
  - Outputs items with JSON:
    - `url`, `title`, `source`, `pubDate`
- **Outputs:** To **Fetch Article Content**.
- **Edge cases / failures:**
  - Google News HTML results are not RSS-like; they likely won’t contain `link/url/guid` fields. This means Google items may be dropped unless the HTTP node returns structured fields (it typically won’t). In practice, the workflow may mainly rely on RSS sources unless Google parsing is added.
  - Some RSS feeds use `link` but may also embed tracking URLs.
  - Missing/invalid URLs cause item to be skipped (silently).

#### Fetch Article Content
- **Type/role:** `httpRequest` to fetch each `url` and return page content.
- **Configuration choices:**
  - URL: `{{$json.url}}`
  - Response format explicitly set to **text**.
  - Browser-like `User-Agent`.
  - `onError: continueErrorOutput` so a single failed fetch won’t halt the entire run.
- **Outputs:** To **AI Scoring Agent**.
- **Important data shape expectation:**
  - Downstream AI node uses `{{$json.data}}` as input text. In many n8n HTTP Request configurations, the returned text may be under a field such as `body` (or similar), depending on node settings/version.
  - If the response text is not actually in `$json.data`, the AI agent will receive `undefined` and produce poor/empty outputs.
- **Edge cases / failures:**
  - 403/406/429 blocks, redirects, large HTML pages, encoding issues.
  - Some sites require JS rendering; raw HTML may not contain readable content.
  - Timeout or very large pages may exceed limits.

#### AI Scoring Agent
- **Type/role:** LangChain **Agent** node that orchestrates prompt + model + structured parsing.
- **Configuration choices:**
  - Input text: `={{ $json.data }}`
  - System message: “expert grant analyst” with instructions:
    - Extract key subsidy info (focus on upcoming fiscal years)
    - Compute:
      - `importanceScore` 1–10 (budget size, ease, coverage)
      - `urgency` High/Medium/Low (deadlines)
    - Return structured JSON with specified fields.
  - `hasOutputParser: true` and connected parser node.
- **Connections:**
  - **Language model input:** from **OpenAI GPT-4o** (AI connection).
  - **Output parser:** from **Structured Output Parser** (AI output parser connection).
  - **Main output:** to **Check Duplicate**.
- **Version-specific notes:**
  - Requires n8n LangChain nodes package (`@n8n/n8n-nodes-langchain`) compatible with the given node versions.
- **Edge cases / failures:**
  - If fetched content is empty/HTML-noisy, extraction quality degrades.
  - Model may return data that doesn’t match schema → parser failures.
  - Rate limits / invalid OpenAI key cause runtime errors.

#### OpenAI GPT-4o
- **Type/role:** OpenAI chat model node providing the LLM for the agent.
- **Configuration choices:** Model set to `gpt-4o`.
- **Credentials:** `openAiApi` credential required.
- **Connections:** Feeds the AI Scoring Agent via `ai_languageModel`.
- **Edge cases / failures:**
  - Invalid/expired API key, quota exceeded, regional restrictions.
  - Model name availability depends on OpenAI account/region.

#### Structured Output Parser
- **Type/role:** Structured JSON schema parser for the agent’s output.
- **Configuration choices:**
  - Provides an example schema including:
    - `subsidyName`, `fiscalYear`, `targetRecipients`, `applicationDeadline`, `budgetAmount`, `requiredDocuments` (array), `administeringAgency`, `eligibilityCriteria`, `importanceScore` (number), `urgency` (string).
- **Connections:** Supplies parser to AI Scoring Agent via `ai_outputParser`.
- **Edge cases / failures:**
  - If the model returns non-JSON, partial JSON, or wrong types, parsing fails (behavior depends on node’s error strategy and agent settings).

---

### Block 4 — Logic, Deduplication, Persistence & Notification

**Overview:**  
Checks whether the analyzed subsidy is already stored in Google Sheets, filters new ones, tags priority by score threshold, saves to Sheets, then notifies Chatwork.

**Nodes involved:**
- Sticky Note Logic
- Check Duplicate
- Is New Subsidy?
- High Priority?
- Set High Priority
- Set Normal Priority
- Save to Google Sheets
- Send Chatwork

#### Sticky Note Logic (sticky note)
- **Role:** Documents dedupe + filter + notify logic.
- **Connections:** None.

#### Check Duplicate
- **Type/role:** `googleSheets` node (spreadsheet-level operation in this configuration).
- **Configuration choices:**
  - Resource: `spreadsheet`
  - Title: `補助金・助成金`
  - No explicit “operation” shown → this is a critical ambiguity:
    - Many dedupe patterns use “Read”/“Get All” rows from a specific sheet, then compare URLs.
    - Here, only the spreadsheet title is set, so the node may not actually return rows/columns unless further configured in the UI.
- **Credentials:** Google Sheets OAuth2 credential required.
- **Outputs:** To **Is New Subsidy?**
- **Edge cases / failures:**
  - Spreadsheet title mismatch or multiple sheets with same name.
  - Missing permissions for service account / OAuth user.
  - If it doesn’t return `sourceUrl` values, downstream dedupe expression breaks.

#### Is New Subsidy?
- **Type/role:** `if` node to filter out duplicates.
- **Condition logic (as configured):**
  - It checks that this expression is **false**:
    ```js
    $('Check Duplicate').all().map(item => item.json.sourceUrl)
      .includes($('AI Scoring Agent').item.json.sourceUrl)
    ```
  - Meaning: proceed only if the current item’s `sourceUrl` is **not** found in Sheets.
- **Important integration note:**
  - The AI agent schema does **not** explicitly include `sourceUrl`, and earlier nodes output `url`, not `sourceUrl`.
  - Unless the agent adds `sourceUrl` or another node maps `url -> sourceUrl`, this will evaluate unexpectedly (often `includes(undefined)` issues).
- **Outputs:**
  - True branch → **High Priority?**
  - False branch → (not connected) ends processing for that item.
- **Edge cases / failures:**
  - If `Check Duplicate` returns no items, `.all()` is empty → includes(...) is false → treated as “new” (may cause duplicates).
  - If `$('AI Scoring Agent').item.json.sourceUrl` is undefined, includes(undefined) may be true if any sheet row has undefined/missing; or always false; behavior depends on actual data.

#### High Priority?
- **Type/role:** `if` node to classify by importance score.
- **Condition:** `{{$json.importanceScore}} >= 7`
- **Outputs:**
  - True → **Set High Priority**
  - False → **Set Normal Priority**
- **Edge cases / failures:**
  - `importanceScore` missing or non-numeric string → type coercion issues (node uses loose type validation settings).
  - Parser failures upstream may prevent reaching this node.

#### Set High Priority
- **Type/role:** `set` node to annotate alert fields.
- **Adds/overwrites:**
  - `alertEmoji`: `🚨【緊急・重要】`
  - `priorityTag`: `High`
- **Includes other fields:** yes (keeps AI extracted fields).
- **Output:** To **Save to Google Sheets**
- **Edge cases:** None major (simple assignment).

#### Set Normal Priority
- **Type/role:** `set` node for normal priority.
- **Adds/overwrites:**
  - `alertEmoji`: `ℹ️`
  - `priorityTag`: `Normal`
- **Output:** To **Save to Google Sheets**

#### Save to Google Sheets
- **Type/role:** `googleSheets` node to persist the record.
- **Operation:** `appendOrUpdate`
- **Document:** ID from config: `={{ $('Workflow Configuration').first().json.googleSheetsId }}`
- **Sheet name:** `Subsidies`
- **Column mapping:** auto-map from input JSON.
- **Output:** To **Send Chatwork**
- **Edge cases / failures:**
  - If the target sheet headers don’t exist or mismatch, auto-mapping can fail or create sparse rows.
  - `appendOrUpdate` typically requires a matching key column configuration; if not configured, behavior may effectively append (depends on node UI settings not visible here).
  - Rate limits and permission errors.

#### Send Chatwork
- **Type/role:** `httpRequest` to Chatwork API to post a message.
- **Configuration choices:**
  - POST to: `={{ $('Workflow Configuration').first().json.chatworkRoomId }}/messages`
    - **Note:** This appears to be missing the Chatwork API base URL (typically `https://api.chatwork.com/v2/rooms/{room_id}/messages`). As written, it will try to POST to a relative/invalid URL unless `chatworkRoomId` already contains the full base path.
  - Header: `X-ChatWorkToken` from config.
  - Body (form-urlencoded): formatted `[info]...[title]...` message including:
    - `alertEmoji`, `importanceScore`, `subsidyName`, `priorityTag`, `urgency`, `targetRecipients`, `applicationDeadline`, `sourceUrl`.
- **Edge cases / failures:**
  - Incorrect endpoint URL (most likely issue).
  - Invalid token / room ID (401/404).
  - Message references `sourceUrl` but workflow upstream primarily has `url`; unless mapped, the URL field may be blank.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note Main | stickyNote | Global documentation and setup requirements |  |  | # AI Subsidy Hunter: Intelligent Scoring & Alert… (includes Google Sheets headers, credentials, configuration steps) |
| Sticky Note Setup | stickyNote | Documents Setup & Trigger block |  |  | ## 1. Setup & Trigger Defines the schedule (9 AM daily) and holds all configuration variables like API keys and room IDs. |
| Daily Schedule (9AM) | scheduleTrigger | Daily workflow trigger |  | Workflow Configuration | ## 1. Setup & Trigger Defines the schedule (9 AM daily) and holds all configuration variables like API keys and room IDs. |
| Workflow Configuration | set | Central config variables (query, RSS URLs, tokens, IDs) | Daily Schedule (9AM) | Search Google News; Read J-Net21 RSS; Read Mirasapo RSS | ## 1. Setup & Trigger Defines the schedule (9 AM daily) and holds all configuration variables like API keys and room IDs. |
| Sticky Note Aggregation | stickyNote | Documents aggregation sources and merge |  |  | ## 2. Data Aggregation Collects subsidy news… then merges them into a single list. |
| Search Google News | httpRequest | Fetch Google News search HTML | Workflow Configuration | Merge All Sources | ## 2. Data Aggregation Collects subsidy news… then merges them into a single list. |
| Read J-Net21 RSS | rssFeedRead | Read/parse J-Net21 RSS feed | Workflow Configuration | Merge All Sources | ## 2. Data Aggregation Collects subsidy news… then merges them into a single list. |
| Read Mirasapo RSS | rssFeedRead | Read/parse Mirasapo Plus RSS feed | Workflow Configuration | Merge All Sources | ## 2. Data Aggregation Collects subsidy news… then merges them into a single list. |
| Merge All Sources | merge | Merge 3 inbound source streams | Search Google News; Read J-Net21 RSS; Read Mirasapo RSS | Extract URLs | ## 2. Data Aggregation Collects subsidy news… then merges them into a single list. |
| Sticky Note Analysis | stickyNote | Documents URL extraction, content fetch, GPT-4o scoring |  |  | ## 3. AI Analysis Extracts the article URL… uses **GPT-4o** to analyze… |
| Extract URLs | code | Normalize items into `{url,title,source,pubDate}` | Merge All Sources | Fetch Article Content | ## 3. AI Analysis Extracts the article URL… uses **GPT-4o** to analyze… |
| Fetch Article Content | httpRequest | Fetch full article HTML/text | Extract URLs | AI Scoring Agent | ## 3. AI Analysis Extracts the article URL… uses **GPT-4o** to analyze… |
| AI Scoring Agent | langchain agent | LLM extraction + scoring + structured JSON output | Fetch Article Content | Check Duplicate | ## 3. AI Analysis Extracts the article URL… uses **GPT-4o** to analyze… |
| OpenAI GPT-4o | langchain lmChatOpenAi | LLM provider for agent |  | AI Scoring Agent (ai_languageModel) | ## 3. AI Analysis Extracts the article URL… uses **GPT-4o** to analyze… |
| Structured Output Parser | langchain outputParserStructured | Enforces JSON schema for agent output |  | AI Scoring Agent (ai_outputParser) | ## 3. AI Analysis Extracts the article URL… uses **GPT-4o** to analyze… |
| Sticky Note Logic | stickyNote | Documents dedupe, priority filter, notifications |  |  | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |
| Check Duplicate | googleSheets | Intended: read existing items for dedupe | AI Scoring Agent | Is New Subsidy? | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |
| Is New Subsidy? | if | Filter out items already in Sheets | Check Duplicate | High Priority? | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |
| High Priority? | if | Classify by `importanceScore >= 7` | Is New Subsidy? | Set High Priority; Set Normal Priority | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |
| Set High Priority | set | Add `alertEmoji` + `priorityTag=High` | High Priority? | Save to Google Sheets | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |
| Set Normal Priority | set | Add `alertEmoji` + `priorityTag=Normal` | High Priority? | Save to Google Sheets | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |
| Save to Google Sheets | googleSheets | Append/update the item in sheet `Subsidies` | Set High Priority; Set Normal Priority | Send Chatwork | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |
| Send Chatwork | httpRequest | Post formatted alert message to Chatwork | Save to Google Sheets |  | ## 4. Logic, Filter & Notify Checks for duplicates… filters… sends… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n with the name:  
   `AI Subsidy Hunter: Intelligent Scoring & Alert System (Fixed Layout)` (or your preferred name).

2. **Add Trigger**
   1) Add node: **Schedule Trigger**  
   2) Set schedule to run **daily at 09:00**.

3. **Add configuration registry**
   1) Add node: **Set** named `Workflow Configuration`  
   2) Add string fields:
      - `googleNewsSearchQuery` = `2026年度 補助金 OR 助成金 OR 令和8年度`
      - `jnet21RssUrl` = `https://j-net21.smrj.go.jp/snavi/support/support.xml`
      - `mirasapoPlusRssUrl` = `https://mirasapo-plus.go.jp/subsidy/feed/`
      - `chatworkRoomId` = your room id (or full endpoint path if you keep the existing URL pattern)
      - `chatworkApiToken` = your Chatwork token
      - `googleSheetsId` = your spreadsheet document ID
   3) Connect: **Schedule Trigger → Workflow Configuration**.

4. **Add aggregation sources**
   1) Add **HTTP Request** named `Search Google News`
      - Method: GET
      - URL expression:
        - `https://www.google.com/search?q={{encodeURIComponent($('Workflow Configuration').first().json.googleNewsSearchQuery)}}&tbm=nws&num=20`
      - Add header `User-Agent` with a browser string.
      - Set “On Error” to **Continue (regular output)**.
   2) Add **RSS Feed Read** named `Read J-Net21 RSS`
      - URL expression: `{{$('Workflow Configuration').first().json.jnet21RssUrl}}`
      - On Error: **Continue (regular output)**.
   3) Add **RSS Feed Read** named `Read Mirasapo RSS`
      - URL expression: `{{$('Workflow Configuration').first().json.mirasapoPlusRssUrl}}`
      - On Error: choose your desired behavior (match workflow: continue to error output).

5. **Merge all sources**
   1) Add **Merge** node named `Merge All Sources`
   2) Set number of inputs to **3**
   3) Connect:
      - Workflow Configuration → Search Google News → Merge input 0
      - Workflow Configuration → Read J-Net21 RSS → Merge input 1
      - Workflow Configuration → Read Mirasapo RSS → Merge input 2

6. **Normalize items to URLs**
   1) Add **Code** node named `Extract URLs`
   2) Paste logic that:
      - Reads all incoming items
      - Extracts `url` from `link|url|guid|originallink`
      - Outputs `{url,title,source,pubDate}`
   3) Connect: **Merge All Sources → Extract URLs**

7. **Fetch each article**
   1) Add **HTTP Request** node named `Fetch Article Content`
      - URL: `{{$json.url}}`
      - Response format: **Text**
      - Header `User-Agent` as above
      - On Error: **Continue to error output** (or continue regular output)
   2) Connect: **Extract URLs → Fetch Article Content**

8. **Add AI components (LangChain)**
   1) Add **OpenAI Chat Model** node named `OpenAI GPT-4o`
      - Select model: `gpt-4o`
      - Configure **OpenAI API credential** (create an OpenAI credential with your API key).
   2) Add **Structured Output Parser** node named `Structured Output Parser`
      - Provide a JSON schema example with fields:
        `subsidyName, fiscalYear, targetRecipients, applicationDeadline, budgetAmount, requiredDocuments, administeringAgency, eligibilityCriteria, importanceScore, urgency`
   3) Add **AI Agent** node named `AI Scoring Agent`
      - Text input: map to the fetched page text (ensure you reference the correct field from the HTTP node output; in the provided workflow it is `{{$json.data}}`)
      - System message: include extraction instructions and scoring requirements
      - Enable structured output parsing
   4) Connect:
      - **Fetch Article Content → AI Scoring Agent**
      - **OpenAI GPT-4o → AI Scoring Agent** via the Agent’s *Language Model* connection
      - **Structured Output Parser → AI Scoring Agent** via the Agent’s *Output Parser* connection

9. **Prepare Google Sheets (data store)**
   1) In Google Sheets, create a spreadsheet (ID used in config).
   2) Create a tab named **`Subsidies`** with headers (at minimum):
      - `subsidyName`, `targetRecipients`, `applicationDeadline`, `budgetAmount`, `urgency`, `importanceScore`, `priorityTag`, `sourceUrl`
   3) Create/configure **Google Sheets OAuth2 credentials** in n8n.

10. **Deduplication check**
   1) Add **Google Sheets** node named `Check Duplicate`
      - Configure it to **read existing rows** from the `Subsidies` sheet (or whichever sheet contains `sourceUrl`)
      - Ensure output includes `sourceUrl` column values for comparison
   2) Connect: **AI Scoring Agent → Check Duplicate**

11. **Filter new items**
   1) Add **IF** node named `Is New Subsidy?`
      - Condition: “current URL not in existing `sourceUrl` list”
      - Ensure you compare consistent fields (e.g., current item `url` vs sheet `sourceUrl`)
   2) Connect: **Check Duplicate → Is New Subsidy?**

12. **Priority classification**
   1) Add **IF** node named `High Priority?`
      - Condition: `importanceScore >= 7`
   2) Connect: **Is New Subsidy? (true) → High Priority?**

13. **Tag priority**
   1) Add **Set** node `Set High Priority`:
      - `alertEmoji` = `🚨【緊急・重要】`
      - `priorityTag` = `High`
   2) Add **Set** node `Set Normal Priority`:
      - `alertEmoji` = `ℹ️`
      - `priorityTag` = `Normal`
   3) Connect:
      - High Priority? (true) → Set High Priority
      - High Priority? (false) → Set Normal Priority

14. **Save to Google Sheets**
   1) Add **Google Sheets** node named `Save to Google Sheets`
      - Operation: `appendOrUpdate`
      - Document ID: `{{$('Workflow Configuration').first().json.googleSheetsId}}`
      - Sheet name: `Subsidies`
      - Mapping: auto-map (or explicit mapping if you want strict control)
   2) Connect:
      - Set High Priority → Save to Google Sheets
      - Set Normal Priority → Save to Google Sheets

15. **Send Chatwork alert**
   1) Add **HTTP Request** node named `Send Chatwork`
      - Method: POST
      - URL: set to the correct Chatwork endpoint, e.g.  
        `https://api.chatwork.com/v2/rooms/{{$('Workflow Configuration').first().json.chatworkRoomId}}/messages`
      - Headers: `X-ChatWorkToken = {{$('Workflow Configuration').first().json.chatworkApiToken}}`
      - Body type: `application/x-www-form-urlencoded`
      - Body parameter `body`: your formatted message including score, urgency, deadline, and URL
   2) Connect: **Save to Google Sheets → Send Chatwork**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheets should contain a tab named `Subsidies` with headers: `subsidyName`, `targetRecipients`, `applicationDeadline`, `budgetAmount`, `urgency`, `importanceScore`, `priorityTag`, `sourceUrl` | From “Sticky Note Main” |
| Requirements: OpenAI API Key, Chatwork Account, Google Sheets | From “Sticky Note Main” |
| Aggregates sources: Google News + J-Net21 + Mirasapo Plus | From sticky notes |
| AI analysis uses GPT-4o and returns JSON including `importanceScore` (1–10) and `urgency` | From sticky notes + agent system message |
| RSS sources: `https://j-net21.smrj.go.jp/snavi/support/support.xml` and `https://mirasapo-plus.go.jp/subsidy/feed/` | From configuration |

