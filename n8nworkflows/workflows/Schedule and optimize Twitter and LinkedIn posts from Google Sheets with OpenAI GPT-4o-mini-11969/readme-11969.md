Schedule and optimize Twitter and LinkedIn posts from Google Sheets with OpenAI GPT-4o-mini

https://n8nworkflows.xyz/workflows/schedule-and-optimize-twitter-and-linkedin-posts-from-google-sheets-with-openai-gpt-4o-mini-11969


# Schedule and optimize Twitter and LinkedIn posts from Google Sheets with OpenAI GPT-4o-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Schedule and optimize Twitter and LinkedIn posts from Google Sheets with OpenAI GPT-4o-mini  
**Workflow name (in JSON):** Schedule and optimize social media posts to Twitter and LinkedIn using AI  
**Primary purpose:**  
Automatically fetch scheduled social posts from Google Sheets, optimize them with OpenAI (GPT‑4o‑mini) for Twitter and LinkedIn, publish to both platforms, write results back to Google Sheets, and notify via Slack. Supports both hourly automation and manual triggering via webhook.

### 1.1 Entry & Triggering
- Two entry points:
  - Hourly schedule trigger for continuous operation.
  - Webhook trigger for manual testing / external invocation.
- These are merged into a single path.

### 1.2 Data Retrieval (Google Sheets Queue)
- Pulls rows from a Google Sheet representing a content queue.
- Normalizes fields and filters to only posts that are due and marked scheduled/ready.

### 1.3 AI Optimization (OpenAI Agent)
- Uses an OpenAI chat model (gpt-4o-mini) via a LangChain Agent node.
- Generates platform-specific text + hashtags in a strict JSON structure.
- Parses the AI output robustly with fallback behavior.

### 1.4 Social Publishing (Twitter + LinkedIn)
- Publishes the optimized content to Twitter and LinkedIn in parallel.
- Aggregates the results into a single item.

### 1.5 Reporting, Logging & Response
- Formats a publishing summary (including URLs).
- Appends a status/result row back into Google Sheets.
- Posts a summary message to Slack.
- If no content is due, returns a “no content” response.
- Always responds to webhook calls with a JSON payload.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Data Retrieval

**Overview:**  
This block triggers the workflow (hourly or manually), then fetches the content queue from Google Sheets and filters for posts that are ready to publish.

**Nodes involved:**
- Main Overview (Sticky Note)
- Section Sticky 1 (Sticky Note)
- Hourly Content Check (Schedule Trigger)
- Manual Post Trigger (Webhook)
- Merge Triggers (Merge)
- Fetch Content Queue (Google Sheets)
- Filter Ready Posts (Code)
- Has Content to Post? (IF)

#### Node: Main Overview (Sticky Note)
- **Type / role:** Sticky Note (documentation)
- **Configuration (interpreted):** Contains global explanation + setup checklist.
- **Connections:** None
- **Failure modes:** None

**Sticky content:**
> ## How it works  
> This workflow automates your social media presence. It monitors a Google Sheet for scheduled posts, uses AI to optimize captions and hashtags for specific platforms (Twitter and LinkedIn), and publishes them automatically. Finally, it updates the post status and notifies you via Slack.  
>
> ## Setup steps  
> 1. **Spreadsheet**: Create a Google Sheet with columns: `status`, `content`, `platforms`, `scheduled_time`, and `hashtags`.  
> 2. **Credentials**: Connect your Google Sheets, OpenAI, Twitter, LinkedIn, and Slack accounts.  
> 3. **Node Config**: Select your specific spreadsheet in both the 'Fetch' and 'Update' Google Sheets nodes.  
> 4. **Test**: Use the 'Manual Post Trigger' to verify the flow before enabling the 'Hourly' schedule.

#### Node: Section Sticky 1 (Sticky Note)
- **Type / role:** Sticky Note (documentation)
- **Connections:** None

**Sticky content:**
> ## 1. Data Retrieval  
> Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets.

#### Node: Hourly Content Check
- **Type / role:** `Schedule Trigger` — time-based entry point
- **Configuration:**
  - Runs every hour (`interval: hours`)
- **Outputs:** → Merge Triggers (input 0)
- **Version notes:** typeVersion `1.2`
- **Failure modes / edge cases:**
  - n8n instance time zone affects “hourly” timing.
  - If the workflow is inactive (`active:false`), it will not run.

#### Node: Manual Post Trigger
- **Type / role:** `Webhook` — manual/external entry point
- **Configuration:**
  - Method: `POST`
  - Path: `social-post`
  - `responseMode: responseNode` (requires Respond to Webhook node to send HTTP response)
  - `onError: continueRegularOutput` (workflow continues even if webhook errors occur)
- **Outputs:** → Merge Triggers (input 1)
- **Version notes:** typeVersion `2`
- **Failure modes / edge cases:**
  - If Respond to Webhook is not reached, the HTTP request may time out.
  - Security/auth for webhook is not configured here (depends on n8n instance settings).

#### Node: Merge Triggers
- **Type / role:** `Merge` — unify two trigger branches
- **Configuration:**
  - Mode: `chooseBranch` (passes through whichever trigger fired)
- **Inputs:** Hourly Content Check, Manual Post Trigger
- **Outputs:** → Fetch Content Queue
- **Version notes:** typeVersion `3`
- **Failure modes / edge cases:**
  - If both triggers somehow execute simultaneously, behavior depends on execution instances (each run is isolated in n8n).

#### Node: Fetch Content Queue
- **Type / role:** `Google Sheets` — retrieves queued posts
- **Configuration (important):**
  - Document ID: **not set in JSON** (must be selected in UI)
  - Sheet name: **not set in JSON** (must be selected in UI)
  - Operation is not explicitly shown; by default this node typically reads rows (you must configure “Read/Get Many” depending on node UI defaults).
- **Inputs:** Merge Triggers
- **Outputs:** → Filter Ready Posts
- **Version notes:** typeVersion `4.5`
- **Failure modes / edge cases:**
  - Missing/invalid OAuth credentials for Google.
  - Document/sheet not selected (node will fail).
  - Column name mismatches (e.g., `scheduled_time` vs `scheduledTime`) are handled later in code, but missing `status/content` may lead to “noContent”.

#### Node: Filter Ready Posts
- **Type / role:** `Code` — transforms rows into publish-ready items
- **Configuration (logic summary):**
  - Reads all incoming items (sheet rows).
  - Keeps rows where:
    - `status` is `scheduled` or `ready` (case-insensitive).
    - `scheduled_time` (or `scheduledTime`) is <= now (if present).
    - Has content in `content` or `text` or `message`.
  - Normalizes fields into:
    - `id` (row id/row_number fallback to Date.now())
    - `content`, `platforms[]` (default: `twitter,linkedin`)
    - `imageUrl`, `scheduledTime`, `category`, `campaign`, `hashtags`, `linkUrl`, `tone`
  - If nothing ready: returns one item `{ noContent: true, message: ... }`
- **Inputs:** Fetch Content Queue
- **Outputs:** → Has Content to Post?
- **Version notes:** typeVersion `2`
- **Failure modes / edge cases:**
  - Date parsing: `new Date(scheduledTime)` may yield “Invalid Date” if sheet uses unexpected format; such a value won’t compare as intended.
  - `platforms` parsing assumes comma-separated string; if sheet stores JSON array, it will fail unless pre-normalized.
  - `id` fallback `Date.now()` can collide if multiple items are processed in the same millisecond (rare but possible).

#### Node: Has Content to Post?
- **Type / role:** `IF` — route depending on `noContent`
- **Condition:**
  - `{{ $json.noContent }}` **notEquals** `true`
- **True output:** → AI Content Optimizer  
- **False output:** → No Content Response
- **Version notes:** typeVersion `2`
- **Failure modes / edge cases:**
  - If items are actual posts, they won’t have `noContent`, so condition passes.
  - If a real row includes `noContent: true` accidentally, it will be treated as empty queue.

---

### Block 2 — AI Optimization

**Overview:**  
Uses GPT‑4o‑mini through a LangChain agent to rewrite content per platform constraints and output structured JSON; then parses and prepares final posting text.

**Nodes involved:**
- Section Sticky 2 (Sticky Note)
- OpenAI Chat Model
- AI Content Optimizer
- Parse AI Content

#### Node: Section Sticky 2 (Sticky Note)
- **Type / role:** Sticky Note (documentation)

**Sticky content:**
> ## 2. AI Optimization  
> Filters posts ready for publishing. The AI Agent then rewrites content to fit platform constraints (e.g., character limits) and generates hashtags.

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` (LangChain OpenAI chat model provider)
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.7`
- **Connections:**
  - Provides **AI languageModel** connection → AI Content Optimizer
- **Version notes:** typeVersion `1.2`
- **Failure modes / edge cases:**
  - Missing OpenAI credentials / invalid API key.
  - Model availability constraints depending on your OpenAI account/region.
  - Rate limits/timeouts.

#### Node: AI Content Optimizer
- **Type / role:** `agent` (LangChain agent) — calls model and returns generated text
- **Configuration (interpreted):**
  - Prompt includes:
    - Original content, target platforms, tone/category, existing hashtags, link.
  - Requires output in a strict JSON structure with:
    - `twitter.text` (<=280 chars) + `twitter.hashtags` (max 5)
    - `linkedin.text` + `linkedin.hashtags`
    - `engagementTips`, `bestPostTime`, `contentScore`
  - System message positions the model as a social media marketing expert.
- **Inputs:** Has Content to Post? (true)
- **Outputs:** → Parse AI Content
- **Model binding:** Receives language model from OpenAI Chat Model via `ai_languageModel` connection.
- **Version notes:** typeVersion `1.7`
- **Failure modes / edge cases:**
  - Model may return non-JSON or malformed JSON (handled by parsing node with fallback).
  - If AI omits required keys (`twitter/linkedin`), parser fallback triggers.

#### Node: Parse AI Content
- **Type / role:** `Code` — parse AI output and build final per-platform post text
- **Configuration (logic summary):**
  - Takes agent output (`input.output` or `input.text`).
  - Extracts first `{ ... }` block via regex and attempts `JSON.parse`.
  - If parsing fails: creates a fallback structure using original content + hashtags.
  - Builds:
    - Twitter final text: `optimized.twitter.text + hashtags` (ensures `#` prefix), truncates to 280 chars.
    - LinkedIn final text: `optimized.linkedin.text + "\n\n" + hashtags`
  - Outputs enriched object:
    - `optimized` object, `posts.twitter`, `posts.linkedin`, `contentScore`, `processedAt`
  - Uses original row data from `$('Filter Ready Posts').item.json`
- **Inputs:** AI Content Optimizer
- **Outputs:** → Post to Twitter and Post to LinkedIn (parallel)
- **Version notes:** typeVersion `2`
- **Failure modes / edge cases:**
  - Regex extraction `/\{[\s\S]*\}/` is greedy; if the AI returns multiple JSON-like blocks, it may parse the wrong combined block.
  - `$('Filter Ready Posts').item.json` assumes node context aligns to the correct item; with multiple items, cross-item referencing can be fragile unless n8n maintains item pairing (often OK, but a common scaling pitfall).
  - Hashtag splitting fallback uses `split(' ')`, but sheet may store hashtags comma-separated.

---

### Block 3 — Social Publishing

**Overview:**  
Posts prepared content to Twitter and LinkedIn concurrently, then aggregates both results into one consolidated item for downstream reporting.

**Nodes involved:**
- Section Sticky 3 (Sticky Note)
- Post to Twitter
- Post to LinkedIn
- Aggregate Post Results

#### Node: Section Sticky 3 (Sticky Note)
- **Type / role:** Sticky Note (documentation)

**Sticky content:**
> ## 3. Social Publishing  
> Distributes the optimized content to Twitter and LinkedIn simultaneously.

#### Node: Post to Twitter
- **Type / role:** `Twitter` — create a tweet
- **Configuration:**
  - Text: `{{ $json.posts.twitter.text }}`
  - No additional fields set (no media upload configured)
- **Inputs:** Parse AI Content
- **Outputs:** → Aggregate Post Results
- **Version notes:** typeVersion `2`
- **Failure modes / edge cases:**
  - Twitter/X API credential issues, app permissions, revoked tokens.
  - Twitter character limit: text is truncated upstream, but URL shortening / special characters can still cause API rejections in some cases.
  - If `posts.twitter.ready` is false, the node still runs because there is no IF gate; it will attempt to post anyway unless you add routing logic.

#### Node: Post to LinkedIn
- **Type / role:** `LinkedIn` — create a LinkedIn share/post
- **Configuration:**
  - Text: `{{ $json.posts.linkedin.text }}`
  - Person ID: `{{ $json.linkedinPersonId || '' }}`
    - **Important:** `linkedinPersonId` is not created earlier in the workflow; you must provide it in the sheet, set it via a prior node, or hardcode it.
- **Inputs:** Parse AI Content
- **Outputs:** → Aggregate Post Results
- **Version notes:** typeVersion `1`
- **Failure modes / edge cases:**
  - Missing `person` id causes failures (empty string).
  - LinkedIn API permissions and approved app requirements.
  - Like Twitter, there is no gate for `posts.linkedin.ready`; it may post even when platforms exclude LinkedIn.

#### Node: Aggregate Post Results
- **Type / role:** `Aggregate` — combine outputs of parallel posting nodes
- **Configuration:**
  - Mode: `aggregateAllItemData` (collect all items into one)
- **Inputs:** Post to Twitter, Post to LinkedIn
- **Outputs:** → Format Results
- **Version notes:** typeVersion `1`
- **Failure modes / edge cases:**
  - If one platform node errors and stops execution, aggregation may not receive both results unless error handling is configured on those nodes (not shown here).

---

### Block 4 — Reporting, Logging & Webhook Response

**Overview:**  
Builds a summary (including post URLs), appends it to Google Sheets, notifies Slack, handles the “no content” case, merges all final paths, and returns an HTTP response for webhook-triggered runs.

**Nodes involved:**
- Section Sticky 4 (Sticky Note)
- Format Results
- Update Content Status
- Post Summary to Slack
- No Content Response
- Merge Final Paths
- Respond to Webhook

#### Node: Section Sticky 4 (Sticky Note)
- **Type / role:** Sticky Note (documentation)

**Sticky content:**
> ## 4. Reporting & Response  
> Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel.

#### Node: Format Results
- **Type / role:** `Code` — interpret API responses and build summary
- **Configuration (logic summary):**
  - Reads aggregated array: `$input.first().json.data || []`
  - Retrieves original data from `$('Parse AI Content').first().json`
  - Detects Twitter success if result has `id_str` or `result.data.id`
    - Builds URL: `https://twitter.com/i/status/<id>`
  - Detects LinkedIn success if result has `id` containing `urn:li:share`
    - Builds URL: `https://linkedin.com/feed/update/<urn>`
  - Produces summary object:
    - `contentId`, short preview, platform list, `postResults`, `summary`, `contentScore`, `publishedAt`
- **Inputs:** Aggregate Post Results
- **Outputs:** → Update Content Status and Post Summary to Slack
- **Version notes:** typeVersion `2`
- **Failure modes / edge cases:**
  - Response shape variability: Twitter/LinkedIn nodes can change output formats; detection logic may miss success.
  - LinkedIn URL construction with a URN may not match LinkedIn’s canonical URL formats in all cases.
  - If only one platform ran successfully, `allSuccessful` becomes false.

#### Node: Update Content Status
- **Type / role:** `Google Sheets` — writes publishing results back
- **Configuration:**
  - Operation: `append`
  - Document ID / Sheet name: **not set in JSON** (must be selected)
  - Appending implies it adds a new row (it does **not** update the original queue row).
- **Inputs:** Format Results
- **Outputs:** → Merge Final Paths
- **Version notes:** typeVersion `4.5`
- **Failure modes / edge cases:**
  - If you intended to mark the original row as “published”, append won’t do it; you’d need an “Update” operation with row identification.
  - Missing sheet/document configuration or credentials.

#### Node: Post Summary to Slack
- **Type / role:** `Slack` — send a channel message
- **Configuration:**
  - Channel: `#social-media` (selected by name)
  - Message text template includes platforms, success count, content score, preview.
- **Inputs:** Format Results
- **Outputs:** → Merge Final Paths
- **Version notes:** typeVersion `2.2`
- **Failure modes / edge cases:**
  - Slack OAuth token missing/expired; channel not found or bot not in channel.
  - If `platforms` missing, expression `join(', ')` fails (in this workflow it should exist).

#### Node: No Content Response
- **Type / role:** `Set` — constructs a simple output when nothing is due
- **Configuration:**
  - `message`: “No content scheduled for posting”
  - `timestamp`: `{{ $now.toISO() }}`
- **Inputs:** Has Content to Post? (false)
- **Outputs:** → Merge Final Paths
- **Version notes:** typeVersion `3.4`
- **Failure modes / edge cases:**
  - None significant; `$now` requires modern n8n expression support (standard).

#### Node: Merge Final Paths
- **Type / role:** `Merge` — unify success path and no-content path
- **Configuration:**
  - Mode: `chooseBranch`
- **Inputs:** Update Content Status, Post Summary to Slack, No Content Response
- **Outputs:** → Respond to Webhook
- **Version notes:** typeVersion `3`
- **Failure modes / edge cases:**
  - Because both “Update Content Status” and “Post Summary to Slack” feed into this merge, they can create multiple items/races; `chooseBranch` will pass through data from whichever input arrives per item. If you need a single consolidated “final” item, consider chaining Slack after Sheets (or using Merge mode that waits for both, depending on desired behavior).

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook` — returns HTTP response to webhook caller
- **Configuration:**
  - Respond with JSON
  - Body:
    ```js
    {
      success: true,
      published: $json.summary ? $json.summary.successCount : 0,
      message: $json.message || 'Posts published successfully'
    }
    ```
- **Inputs:** Merge Final Paths
- **Outputs:** none (terminal)
- **Version notes:** typeVersion `1.1`
- **Failure modes / edge cases:**
  - If workflow started from Schedule Trigger, this node still runs but there is no HTTP caller; it simply completes (normal in n8n).
  - If Merge Final Paths passes an item without `summary` and without `message`, message defaults correctly.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | Sticky Note | Documentation / setup guidance | — | — | ## How it works … (includes setup steps and sheet columns + credential notes; see full text in section 2) |
| Section Sticky 1 | Sticky Note | Documentation for retrieval block | — | — | ## 1. Data Retrieval Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets. |
| Hourly Content Check | Schedule Trigger | Hourly automation entry point | — | Merge Triggers | ## 1. Data Retrieval Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets. |
| Manual Post Trigger | Webhook | Manual/external entry point (POST /social-post) | — | Merge Triggers | ## 1. Data Retrieval Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets. |
| Merge Triggers | Merge | Unifies trigger branches | Hourly Content Check; Manual Post Trigger | Fetch Content Queue | ## 1. Data Retrieval Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets. |
| Fetch Content Queue | Google Sheets | Reads queued posts from sheet | Merge Triggers | Filter Ready Posts | ## 1. Data Retrieval Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets. |
| Filter Ready Posts | Code | Filters/normalizes ready posts | Fetch Content Queue | Has Content to Post? | ## 1. Data Retrieval Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets. |
| Has Content to Post? | IF | Routes to AI or no-content | Filter Ready Posts | AI Content Optimizer; No Content Response | ## 1. Data Retrieval Triggers the workflow hourly or via webhook and pulls the latest content queue from Google Sheets. |
| Section Sticky 2 | Sticky Note | Documentation for AI block | — | — | ## 2. AI Optimization Filters posts ready for publishing. The AI Agent then rewrites content to fit platform constraints (e.g., character limits) and generates hashtags. |
| OpenAI Chat Model | LangChain OpenAI Chat Model | Provides GPT-4o-mini model | — | AI Content Optimizer (ai_languageModel) | ## 2. AI Optimization Filters posts ready for publishing. The AI Agent then rewrites content to fit platform constraints (e.g., character limits) and generates hashtags. |
| AI Content Optimizer | LangChain Agent | Generates optimized platform JSON | Has Content to Post? | Parse AI Content | ## 2. AI Optimization Filters posts ready for publishing. The AI Agent then rewrites content to fit platform constraints (e.g., character limits) and generates hashtags. |
| Parse AI Content | Code | Parses AI JSON, builds final post texts | AI Content Optimizer | Post to Twitter; Post to LinkedIn | ## 2. AI Optimization Filters posts ready for publishing. The AI Agent then rewrites content to fit platform constraints (e.g., character limits) and generates hashtags. |
| Section Sticky 3 | Sticky Note | Documentation for publishing block | — | — | ## 3. Social Publishing Distributes the optimized content to Twitter and LinkedIn simultaneously. |
| Post to Twitter | Twitter | Publishes tweet | Parse AI Content | Aggregate Post Results | ## 3. Social Publishing Distributes the optimized content to Twitter and LinkedIn simultaneously. |
| Post to LinkedIn | LinkedIn | Publishes LinkedIn post | Parse AI Content | Aggregate Post Results | ## 3. Social Publishing Distributes the optimized content to Twitter and LinkedIn simultaneously. |
| Aggregate Post Results | Aggregate | Aggregates publish results | Post to Twitter; Post to LinkedIn | Format Results | ## 3. Social Publishing Distributes the optimized content to Twitter and LinkedIn simultaneously. |
| Section Sticky 4 | Sticky Note | Documentation for reporting block | — | — | ## 4. Reporting & Response Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel. |
| Format Results | Code | Builds summary + URLs | Aggregate Post Results | Update Content Status; Post Summary to Slack | ## 4. Reporting & Response Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel. |
| Update Content Status | Google Sheets | Appends results/status row | Format Results | Merge Final Paths | ## 4. Reporting & Response Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel. |
| Post Summary to Slack | Slack | Sends summary message | Format Results | Merge Final Paths | ## 4. Reporting & Response Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel. |
| No Content Response | Set | Output when no posts are due | Has Content to Post? (false) | Merge Final Paths | ## 4. Reporting & Response Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel. |
| Merge Final Paths | Merge | Unifies final outcomes | Update Content Status; Post Summary to Slack; No Content Response | Respond to Webhook | ## 4. Reporting & Response Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel. |
| Respond to Webhook | Respond to Webhook | Returns HTTP JSON response | Merge Final Paths | — | ## 4. Reporting & Response Aggregates results, logs post URLs back to the spreadsheet, and sends a summary report to your Slack channel. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Trigger: “Schedule Trigger”**
   - Name: `Hourly Content Check`
   - Configure: run every **1 hour**.
3. **Add Trigger: “Webhook”**
   - Name: `Manual Post Trigger`
   - Method: `POST`
   - Path: `social-post`
   - Response mode: **Using “Respond to Webhook” node** (responseNode)
4. **Add “Merge” node**
   - Name: `Merge Triggers`
   - Mode: `Choose Branch`
   - Connect:
     - `Hourly Content Check` → `Merge Triggers` (Input 1)
     - `Manual Post Trigger` → `Merge Triggers` (Input 2)

5. **Add “Google Sheets” node** (read queue)
   - Name: `Fetch Content Queue`
   - Credentials: connect Google account (OAuth2) with access to the spreadsheet.
   - Select:
     - **Document**: your content queue spreadsheet
     - **Sheet**: the queue tab
   - Operation: configure to **read rows** (e.g., “Read / Get Many” depending on your n8n version).
   - Connect: `Merge Triggers` → `Fetch Content Queue`

6. **Add “Code” node**
   - Name: `Filter Ready Posts`
   - Paste the logic that:
     - selects rows where `status` in (`scheduled`, `ready`)
     - scheduled time is due
     - normalizes fields (`content`, `platforms`, etc.)
     - returns `{noContent:true}` when none are ready
   - Connect: `Fetch Content Queue` → `Filter Ready Posts`

7. **Add “IF” node**
   - Name: `Has Content to Post?`
   - Condition (Boolean): `{{$json.noContent}}` **not equals** `true`
   - Connect: `Filter Ready Posts` → `Has Content to Post?`

8. **Add OpenAI model node (LangChain)**
   - Node: `OpenAI Chat Model` (`lmChatOpenAi`)
   - Credentials: OpenAI API key
   - Model: `gpt-4o-mini`
   - Temperature: `0.7`

9. **Add “AI Agent” node (LangChain Agent)**
   - Name: `AI Content Optimizer`
   - System message: social media marketing expert guidance (as in workflow)
   - User prompt: include original content, platforms, tone, category, hashtags, link, and demand **exact JSON** with twitter/linkedin + metadata.
   - Connect:
     - `Has Content to Post?` (true) → `AI Content Optimizer`
     - `OpenAI Chat Model` (AI languageModel output) → `AI Content Optimizer` (ai_languageModel input)

10. **Add “Code” node**
    - Name: `Parse AI Content`
    - Implement:
      - extract JSON from AI text
      - fallback to original content if parsing fails
      - create `posts.twitter.text` (<=280) and `posts.linkedin.text`
    - Connect: `AI Content Optimizer` → `Parse AI Content`

11. **Add “Twitter” node**
    - Name: `Post to Twitter`
    - Operation: create tweet/post
    - Text: `{{$json.posts.twitter.text}}`
    - Credentials: Twitter/X (as supported by your n8n node; ensure write permissions)
    - Connect: `Parse AI Content` → `Post to Twitter`

12. **Add “LinkedIn” node**
    - Name: `Post to LinkedIn`
    - Operation: create share/post
    - Text: `{{$json.posts.linkedin.text}}`
    - Person/User URN or ID:
      - Set from `{{$json.linkedinPersonId}}` **or** hardcode your LinkedIn person id.
    - Credentials: LinkedIn OAuth2 with posting permissions
    - Connect: `Parse AI Content` → `Post to LinkedIn`

13. **Add “Aggregate” node**
    - Name: `Aggregate Post Results`
    - Mode: aggregate all item data (collect both platform responses)
    - Connect:
      - `Post to Twitter` → `Aggregate Post Results`
      - `Post to LinkedIn` → `Aggregate Post Results`

14. **Add “Code” node**
    - Name: `Format Results`
    - Build:
      - detect success for each platform from responses
      - construct URLs
      - compute success counts and timestamps
    - Connect: `Aggregate Post Results` → `Format Results`

15. **Add “Google Sheets” node** (write-back)
    - Name: `Update Content Status`
    - Credentials: same Google account
    - Select document + sheet for logging results
    - Operation: `Append` (adds a new row with summary)
    - Connect: `Format Results` → `Update Content Status`

16. **Add “Slack” node**
    - Name: `Post Summary to Slack`
    - Credentials: Slack OAuth2
    - Channel: `#social-media` (ensure the bot is in the channel)
    - Message: include platforms, success ratio, content score, preview
    - Connect: `Format Results` → `Post Summary to Slack`

17. **Add “Set” node** (no-content branch)
    - Name: `No Content Response`
    - Set fields:
      - `message` = “No content scheduled for posting”
      - `timestamp` = `{{$now.toISO()}}`
    - Connect: `Has Content to Post?` (false) → `No Content Response`

18. **Add final “Merge” node**
    - Name: `Merge Final Paths`
    - Mode: `Choose Branch`
    - Connect:
      - `Update Content Status` → `Merge Final Paths`
      - `Post Summary to Slack` → `Merge Final Paths`
      - `No Content Response` → `Merge Final Paths`

19. **Add “Respond to Webhook” node**
    - Name: `Respond to Webhook`
    - Response: JSON
    - Body expression:
      - `success: true`
      - `published: $json.summary ? $json.summary.successCount : 0`
      - `message: $json.message || 'Posts published successfully'`
    - Connect: `Merge Final Paths` → `Respond to Webhook`

20. **Credentials checklist**
    - Google Sheets OAuth2: read + append access to target spreadsheet
    - OpenAI API key: access to `gpt-4o-mini`
    - Twitter/X: posting permissions
    - LinkedIn: posting permissions + valid `person id`
    - Slack: chat:write permissions to the target channel

21. **Test procedure**
    - Execute via `Manual Post Trigger` (POST `/social-post`) after adding one due “scheduled/ready” row in Sheets.
    - Verify:
      - Posts created on both platforms
      - Slack message received
      - Sheet append row created
      - Webhook response contains `published` count

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Google Sheet with columns: `status`, `content`, `platforms`, `scheduled_time`, `hashtags`. | From “Main Overview” sticky note |
| Configure spreadsheet selection in both Google Sheets nodes (“Fetch” and “Update”). | From “Main Overview” sticky note |
| Test with “Manual Post Trigger” before enabling “Hourly” schedule. | From “Main Overview” sticky note |
| Workflow includes AI optimization for platform constraints and auto-publishing to Twitter + LinkedIn, plus Slack notification. | From “Main Overview” sticky note |

