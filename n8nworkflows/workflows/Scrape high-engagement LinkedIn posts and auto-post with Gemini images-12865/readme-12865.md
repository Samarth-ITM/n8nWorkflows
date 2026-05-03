Scrape high-engagement LinkedIn posts and auto-post with Gemini images

https://n8nworkflows.xyz/workflows/scrape-high-engagement-linkedin-posts-and-auto-post-with-gemini-images-12865


# Scrape high-engagement LinkedIn posts and auto-post with Gemini images

## 1. Workflow Overview

**Purpose:**  
This workflow discovers high-engagement (“viral”) LinkedIn posts from a list of profiles, stores them in Google Sheets, then uses Google Gemini to analyze recent trends, generate a new LinkedIn post + an image prompt, generates an image with Imagen, and publishes the post to a LinkedIn organization page. It also marks processed rows to avoid reprocessing.

**Target use cases:**
- Continuous trend mining from competitors/creators on LinkedIn
- Semi-automated content ideation and publishing for a company page
- Building a content “inbox” in Google Sheets and letting AI turn it into original posts

### 1.1 Scheduled scraping & storage (Phase 1)
Runs every 12 hours, reads profile URLs from Google Sheets, scrapes the latest post per profile via Apify, filters for engagement thresholds, then upserts results into a “scrape data” sheet.

### 1.2 Sheet monitoring & trend aggregation (Phase 2)
Polls Google Sheets for new rows. Filters for (a) posts published within 3 days and (b) not yet processed. Aggregates the selected rows into a single payload for the AI agent.

### 1.3 AI content generation + image generation + publishing
Gemini agent generates structured JSON (title, post, image prompt), Imagen generates an image from the prompt, then the workflow publishes an image post to a LinkedIn organization page.

### 1.4 Status tracking
After publishing, extracts the LinkedIn post URLs that were analyzed and marks corresponding rows as “is done = yes” in the sheet to prevent duplicates.

---

## 2. Block-by-Block Analysis

### Block A — Scheduled Profile Scraping
**Overview:** Kicks off every 12 hours, loads LinkedIn profile URLs from Google Sheets, and iterates through them in small batches to control throughput.  
**Nodes involved:**  
- LinkedIn Content Automation Scheduler  
- Fetch LinkedIn Profile URLs  
- Process Profiles in Batches

#### Node: LinkedIn Content Automation Scheduler
- **Type / role:** Schedule Trigger; periodic entry point.
- **Configuration:** Runs every **12 hours**.
- **Inputs/Outputs:** No inputs. Output triggers **Fetch LinkedIn Profile URLs**.
- **Potential failures:** None typical; ensure workflow is active. Timezone considerations depend on n8n instance settings.
- **Version notes:** typeVersion 1.2.

#### Node: Fetch LinkedIn Profile URLs
- **Type / role:** Google Sheets; reads the list of target profiles.
- **Configuration choices:**
  - Connects to a Google Spreadsheet (“LinkedIn Usernames” document).
  - Reads from sheet/tab named **“usernames & links”** (via gid placeholder).
  - Operation is not explicitly shown in JSON, but by placement and usage it functions as a read/list step.
- **Key fields expected in each row:** `Linkedin url` (used later as `{{ $json['Linkedin url'] }}`).
- **Outputs:** Items passed into **Process Profiles in Batches**.
- **Potential failures / edge cases:**
  - OAuth/permissions issues accessing the spreadsheet.
  - Column name mismatch (must be exactly `Linkedin url`).
  - Empty sheet returns 0 items (downstream will do nothing).
- **Version notes:** typeVersion 4.5.

#### Node: Process Profiles in Batches
- **Type / role:** Split In Batches; throttles processing volume.
- **Configuration:** `batchSize: 3`.
- **Connections:**
  - **Output 1 (loop)** is used: connected to **Scrape LinkedIn Posts API**.
  - The “continue” output loops back from **Save Viral Posts to Sheets** into this node to process the next batch.
- **Potential failures / edge cases:**
  - If downstream errors occur, the loop may stop mid-batch.
  - Batch size too high can increase API rate limit risk.
- **Version notes:** typeVersion 3.

**Sticky note (applies to this block):**  
“## Profile Scraping … Scheduled trigger runs every 12 hours … process them in batches of 3 to avoid rate limits.”

---

### Block B — Post Extraction via Apify + Rate Limiting
**Overview:** For each profile, calls Apify’s LinkedIn scraping actor, fetches dataset items, and waits 3 seconds between calls.  
**Nodes involved:**  
- Scrape LinkedIn Posts API  
- Rate Limiting Delay

#### Node: Scrape LinkedIn Posts API
- **Type / role:** HTTP Request; calls Apify Actor endpoint.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://api.apify.com/v2/acts/LQQIXN9Othf8f7R5n/run-sync-get-dataset-items`
  - **Timeout:** 60s
  - **Headers:**  
    - `Accept: application/json`  
    - `Authorization: Bearer {{APIFY_API_TOKEN_PLACEHOLDER}}`  
    - `Content-Type: application/json`
  - **JSON body (expression):**
    - `username`: `{{ $json['Linkedin url'] }}`
    - `page_number`: 1
    - `limit`: 1 (only the most recent post)
- **Inputs:** One item per LinkedIn profile row.
- **Outputs:** Response items forwarded to **Rate Limiting Delay**.
- **Potential failures / edge cases:**
  - 401/403 if Apify token is invalid.
  - Actor ID may be private/changed; endpoint could break if actor is removed.
  - Response shape differences: downstream expects fields like `stats.like`, `stats.reposts`, `stats.comments`, `author.first_name`, `posted_at.date`, `text`, `url`.
  - `limit: 1` means you may miss other viral posts not in the most recent slot.
- **Version notes:** typeVersion 4.2.

#### Node: Rate Limiting Delay
- **Type / role:** Wait; throttling between API calls.
- **Configuration:** `amount: 3` (seconds by default in Wait node).
- **Connections:** Outputs to **Filter High-Engagement Posts**.
- **Potential failures:** Minimal; excessive workflow concurrency could still trigger rate limits.
- **Version notes:** typeVersion 1.1.

**Sticky note (applies to this block):**  
“## Post Extraction … Calls Apify API … Includes 3-second delay between requests …”

---

### Block C — Engagement Filtering + Persisting Viral Posts
**Overview:** Filters scraped posts for engagement thresholds, then upserts matching “viral” posts into a Google Sheet (“scrape data”).  
**Nodes involved:**  
- Filter High-Engagement Posts  
- Save Viral Posts to Sheets

#### Node: Filter High-Engagement Posts
- **Type / role:** Filter; selects items meeting engagement criteria.
- **Configuration choices:**
  - OR-combinator across three conditions (any one passes):
    - `stats.reposts > 20`
    - `stats.like > 20`
    - `stats.comments > 20`
  - Loose type validation enabled (reduces strict failures but can hide data issues).
- **Inputs:** Apify result items.
- **Outputs:** Passing items go to **Save Viral Posts to Sheets**.
- **Potential failures / edge cases:**
  - If `stats` or nested fields are missing/null, comparisons may behave unexpectedly.
  - Engagement thresholds are low for many niches; may over-capture.
- **Version notes:** typeVersion 2.2.

#### Node: Save Viral Posts to Sheets
- **Type / role:** Google Sheets; append or update rows in “scrape data”.
- **Configuration choices:**
  - **Operation:** Append or Update (`appendOrUpdate`)
  - **Sheet:** `scrape data` (gid=0)
  - **Matching column:** `Caption/Text/Copy` (used to decide update vs append)
  - **Mapped fields written:**
    - Likes = `{{ $json.stats.like }}`
    - Reposts = `{{ $json.stats.reposts }}`
    - Comments = `{{ $json.stats.comments }}`
    - Creator Name = `{{ $json.author.first_name }} {{ $json.author.last_name }}`
    - Date Scraped = `{{ $now.format('yyyy-MM-dd').toJsonString() }}`
    - Date Published = `{{ $json.posted_at.date.toDate().format('yyyy-MM-dd').toJsonString() }}`
    - Caption/Text/Copy = `{{ $json.text }}`
    - Linkedin Post URL = `{{ $json.url }}`
  - Column `is done` exists but is marked removed in the schema here; later nodes still rely on it existing in the sheet.
- **Connections:**
  - After write, it connects back to **Process Profiles in Batches** to continue the batch loop.
- **Potential failures / edge cases:**
  - If `Caption/Text/Copy` is not unique, updates may overwrite unintended rows.
  - `posted_at.date` conversion can fail if the API returns unexpected date formats.
  - If the sheet column names differ even slightly, mapping will fail or write blanks.
- **Version notes:** typeVersion 4.5.

**Sticky note (applies to this block):**  
“## Engagement Filtering … Filters posts with high engagement … saves viral content to Google Sheets …”

---

### Block D — Monitoring Sheet for New Data + Filtering Recent & Unprocessed
**Overview:** Watches the “scrape data” sheet for new rows, then keeps only posts published within the last 3 days and not yet marked as done.  
**Nodes involved:**  
- New Post Data Trigger  
- Filter Recent Posts (3 Days)  
- Aggregate Trending Content

#### Node: New Post Data Trigger
- **Type / role:** Google Sheets Trigger; second entry point.
- **Configuration choices:**
  - Polling mode: **every minute**
  - Watches the `scrape data` sheet (gid=0).
- **Credentials:** Google Sheets Trigger OAuth2.
- **Outputs:** New/updated row items to **Filter Recent Posts (3 Days)**.
- **Potential failures / edge cases:**
  - Polling triggers can miss rapid inserts if API limits or if multiple rows are inserted at once depending on trigger behavior.
  - Requires stable Google auth; token refresh failures will halt.
- **Version notes:** typeVersion 1.

#### Node: Filter Recent Posts (3 Days)
- **Type / role:** Filter; enforces recency and “not processed”.
- **Configuration choices (AND-combinator):**
  1) Days difference <= 3, computed as:  
     `Math.floor((todayMidnight - publishedMidnight) / (msPerDay)) <= 3`  
     Uses `$json['Date Published']` from the sheet.
  2) `is done` is empty (string empty check): `{{ $json['is done'] }}`
- **Outputs:** Passing items go to **Aggregate Trending Content**.
- **Potential failures / edge cases:**
  - If `Date Published` is empty or not parseable by `new Date(...)`, result becomes `NaN` and the condition may fail unexpectedly.
  - If the sheet uses localized date formats (DD/MM/YYYY), JS parsing may misinterpret.
  - If “is done” column is missing, expression resolves to undefined; “empty” check may pass or fail depending on node behavior.
- **Version notes:** typeVersion 2.2.

#### Node: Aggregate Trending Content
- **Type / role:** Aggregate; merges multiple filtered rows into one combined object for AI.
- **Configuration:** `aggregateAllItemData` (collects all item JSONs).
- **Outputs:** Single aggregated payload to **LinkedIn Content Strategy AI**.
- **Potential failures / edge cases:**
  - Large numbers of rows could create very large prompts; may exceed model limits or slow execution.
- **Version notes:** typeVersion 1.

**Sticky note (applies to this block):**  
“## Data Monitoring … Triggers when new rows are added … Filters unprocessed posts from the last 3 days and aggregates data …”

---

### Block E — AI Trend Analysis + Structured Output Parsing
**Overview:** Gemini-based agent analyzes aggregated posts, generates a new post (plain text), plus a short image prompt. Output is expected to be valid JSON according to the schema parser.  
**Nodes involved:**  
- Google Gemini Chat Model  
- LinkedIn Content Strategy AI  
- JSON Output Parser

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain chat model wrapper for Google Gemini (PaLM API credential).
- **Configuration:** Default options (none set).
- **Connections:** Provides the **ai_languageModel** input to **LinkedIn Content Strategy AI**.
- **Potential failures:**
  - Invalid/expired API key.
  - Model availability or quota limits.
- **Version notes:** typeVersion 1.

#### Node: LinkedIn Content Strategy AI
- **Type / role:** LangChain Agent; prompt-driven content strategist.
- **Configuration choices:**
  - **Input text:** `{{ $json.data }}` (from Aggregate node)
  - **System message:** Very detailed content strategy framework, formatting rules (plain text, no markdown), and strict “three outputs” requirement in JSON: `title`, `post`, `image_prompt`.
  - **Output parser enabled:** `hasOutputParser: true`
- **Connections:**
  - Receives **ai_languageModel** from Google Gemini Chat Model.
  - Receives **ai_outputParser** from JSON Output Parser node.
  - Main output goes to **Generate an image**.
- **Potential failures / edge cases:**
  - If the agent outputs non-JSON or violates schema, the output parser step will fail the run (or the agent may fail to validate).
  - Prompt size can exceed limits if too many posts are aggregated.
  - The system message asks for emojis (1–3) but also requires plain text; that’s fine, but ensure LinkedIn node supports the characters.
- **Version notes:** typeVersion 2.2.

#### Node: JSON Output Parser
- **Type / role:** Structured output parser; enforces JSON shape.
- **Configuration:** Schema example requires keys: `title`, `post`, `image_prompt`.
- **Connections:** Connected to the agent via `ai_outputParser`.
- **Potential failures:**
  - The model returns extra text around JSON or invalid escaping.
  - Missing required keys.
- **Version notes:** typeVersion 1.3.

**Sticky note (applies to this block):**  
“## AI Content Strategy … Outputs structured JSON with title, post text, and image prompt.”

---

### Block F — Visual Generation (Imagen) + Publish to LinkedIn
**Overview:** Generates an image from the AI’s prompt and publishes a LinkedIn organization image post using the AI-generated title and post content.  
**Nodes involved:**  
- Generate an image  
- Publish to LinkedIn

#### Node: Generate an image
- **Type / role:** Google Gemini (Imagen) image generation node.
- **Configuration choices:**
  - **Resource:** image
  - **Model:** `models/imagen-4.0-generate-preview-06-06`
  - **Prompt:** `{{ $json.output.image_prompt }}`
    - Note: this assumes the incoming item contains `output.image_prompt` (consistent with agent output structure).
- **Outputs:** Image binary/metadata forwarded to **Publish to LinkedIn**.
- **Potential failures / edge cases:**
  - Quota/model access restrictions on Imagen preview model.
  - Prompt safety filters may block some content.
  - If the incoming JSON path differs (e.g., `$('LinkedIn Content Strategy AI').item.json.output.image_prompt`), this node could get an empty prompt.
- **Version notes:** typeVersion 1.1.

#### Node: Publish to LinkedIn
- **Type / role:** LinkedIn node; posts to an organization page with an image.
- **Configuration choices:**
  - Post as: **organization**
  - Organization ID: placeholder `{{LINKEDIN_ORGANIZATION_ID_PLACEHOLDER}}`
  - Share media category: **IMAGE**
  - Text: `{{ $('LinkedIn Content Strategy AI').item.json.output.post }}`
  - Title: `{{ $('LinkedIn Content Strategy AI').item.json.output.title }}`
- **Inputs:** Output from **Generate an image** (so it can attach media) plus it references the AI node via expressions.
- **Outputs:** Goes to **Extract Analyzed Post URLs**.
- **Potential failures / edge cases:**
  - LinkedIn API permissions: organization posting requires proper app scopes and admin rights.
  - Media upload failures (file size, format).
  - Expression references a different node item than the incoming one; in multi-item contexts, mismatches can occur.
- **Version notes:** typeVersion 1.

**Sticky note (applies to this block):**  
“## Visual Generation … Generates professional LinkedIn-optimized images using Google Imagen …”

---

### Block G — Status Tracking (“is done”)
**Overview:** Extracts LinkedIn Post URLs of analyzed items and upserts “is done = yes” back to the sheet to prevent reprocessing.  
**Nodes involved:**  
- Extract Analyzed Post URLs  
- Mark Posts as Analyzed

#### Node: Extract Analyzed Post URLs
- **Type / role:** Code; creates a list of URLs to mark as processed.
- **Configuration (JS code):**
  - Uses `$items(sourceNode)` where `sourceNode = 'Filter Posts Within 3 Days'`.
  - Maps each item to `{ postUrl: item.json["Linkedin Post URL"] }`.
- **Connections:** Output goes to **Mark Posts as Analyzed**.
- **Critical issue / edge case (likely bug):**
  - The node name referenced **does not exist** in this workflow. The actual filter node is named **“Filter Recent Posts (3 Days)”**.
  - As written, `$items('Filter Posts Within 3 Days')` will throw an error at runtime.
  - Fix: set `sourceNode` to `'Filter Recent Posts (3 Days)'`.
- **Version notes:** typeVersion 2.

#### Node: Mark Posts as Analyzed
- **Type / role:** Google Sheets; append or update to set status.
- **Configuration choices:**
  - **Operation:** Append or Update (`appendOrUpdate`)
  - **Matching column:** `Linkedin Post URL`
  - Writes:
    - `Linkedin Post URL = {{ $json.postUrl }}`
    - `is done = yes`
- **Inputs:** Items from Extract Analyzed Post URLs.
- **Potential failures / edge cases:**
  - If URL is blank/null, upsert may create bad rows or fail to match.
  - If the sheet column is named differently (e.g., “LinkedIn Post URL” capitalization), matching will fail.
- **Version notes:** typeVersion 4.7.

**Sticky note (applies to this block):**  
“## Status Tracking … marks them as analyzed in Google Sheets to prevent duplicate processing.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini chat model to agent | (none) | LinkedIn Content Strategy AI (ai_languageModel) | ## AI Content Strategy  Google Gemini analyzes trending patterns from aggregated posts and generates original LinkedIn content following platform best practices. Outputs structured JSON with title, post text, and image prompt. |
| LinkedIn Content Automation Scheduler | n8n-nodes-base.scheduleTrigger | Entry point; runs scraping every 12h | (none) | Fetch LinkedIn Profile URLs | ## Profile Scraping  Scheduled trigger runs every 12 hours to fetch LinkedIn profile URLs from Google Sheets and process them in batches of 3 to avoid rate limits. |
| Fetch LinkedIn Profile URLs | n8n-nodes-base.googleSheets | Reads profile URLs from sheet | LinkedIn Content Automation Scheduler | Process Profiles in Batches | ## Profile Scraping  Scheduled trigger runs every 12 hours to fetch LinkedIn profile URLs from Google Sheets and process them in batches of 3 to avoid rate limits. |
| Process Profiles in Batches | n8n-nodes-base.splitInBatches | Batch iteration (size 3) | Fetch LinkedIn Profile URLs; Save Viral Posts to Sheets | Scrape LinkedIn Posts API | ## Profile Scraping  Scheduled trigger runs every 12 hours to fetch LinkedIn profile URLs from Google Sheets and process them in batches of 3 to avoid rate limits. |
| Scrape LinkedIn Posts API | n8n-nodes-base.httpRequest | Calls Apify to scrape posts | Process Profiles in Batches | Rate Limiting Delay | ## Post Extraction  Calls Apify API to scrape recent LinkedIn posts from each profile. Includes 3-second delay between requests to respect API rate limits. |
| Rate Limiting Delay | n8n-nodes-base.wait | Throttle requests | Scrape LinkedIn Posts API | Filter High-Engagement Posts | ## Post Extraction  Calls Apify API to scrape recent LinkedIn posts from each profile. Includes 3-second delay between requests to respect API rate limits. |
| Filter High-Engagement Posts | n8n-nodes-base.filter | Keeps posts with likes/comments/reposts > 20 | Rate Limiting Delay | Save Viral Posts to Sheets | ## Engagement Filtering  Filters posts with high engagement (20+ likes, comments, or reposts) and saves viral content to Google Sheets for analysis. |
| Save Viral Posts to Sheets | n8n-nodes-base.googleSheets | Upserts viral posts into “scrape data” | Filter High-Engagement Posts | Process Profiles in Batches | ## Engagement Filtering  Filters posts with high engagement (20+ likes, comments, or reposts) and saves viral content to Google Sheets for analysis. |
| New Post Data Trigger | n8n-nodes-base.googleSheetsTrigger | Entry point; detects new sheet rows | (none) | Filter Recent Posts (3 Days) | ## Data Monitoring  Triggers when new rows are added to Google Sheets. Filters unprocessed posts from the last 3 days and aggregates data for AI analysis. |
| Filter Recent Posts (3 Days) | n8n-nodes-base.filter | Keeps posts within 3 days and not done | New Post Data Trigger | Aggregate Trending Content | ## Data Monitoring  Triggers when new rows are added to Google Sheets. Filters unprocessed posts from the last 3 days and aggregates data for AI analysis. |
| Aggregate Trending Content | n8n-nodes-base.aggregate | Aggregates items into single AI input | Filter Recent Posts (3 Days) | LinkedIn Content Strategy AI | ## Data Monitoring  Triggers when new rows are added to Google Sheets. Filters unprocessed posts from the last 3 days and aggregates data for AI analysis. |
| LinkedIn Content Strategy AI | @n8n/n8n-nodes-langchain.agent | Trend analysis + generates title/post/image_prompt | Aggregate Trending Content; Google Gemini Chat Model (ai); JSON Output Parser (ai) | Generate an image | ## AI Content Strategy  Google Gemini analyzes trending patterns from aggregated posts and generates original LinkedIn content following platform best practices. Outputs structured JSON with title, post text, and image prompt. |
| JSON Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output | (none) | LinkedIn Content Strategy AI (ai_outputParser) | ## AI Content Strategy  Google Gemini analyzes trending patterns from aggregated posts and generates original LinkedIn content following platform best practices. Outputs structured JSON with title, post text, and image prompt. |
| Generate an image | @n8n/n8n-nodes-langchain.googleGemini | Generates Imagen image from prompt | LinkedIn Content Strategy AI | Publish to LinkedIn | ## Visual Generation  Generates professional LinkedIn-optimized images using Google Imagen based on AI-created prompts. |
| Publish to LinkedIn | n8n-nodes-base.linkedIn | Posts an organization image post | Generate an image | Extract Analyzed Post URLs | ## Visual Generation  Generates professional LinkedIn-optimized images using Google Imagen based on AI-created prompts. |
| Extract Analyzed Post URLs | n8n-nodes-base.code | Extracts URLs to mark as done | Publish to LinkedIn | Mark Posts as Analyzed | ## Status Tracking  Extracts processed post URLs and marks them as analyzed in Google Sheets to prevent duplicate processing. |
| Mark Posts as Analyzed | n8n-nodes-base.googleSheets | Upserts “is done=yes” matched by URL | Extract Analyzed Post URLs | (none) | ## Status Tracking  Extracts processed post URLs and marks them as analyzed in Google Sheets to prevent duplicate processing. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## LinkedIn Viral Content Automation  This workflow automates LinkedIn content discovery and creation by scraping high-engagement posts, analyzing trends with AI, and publishing optimized content.  ## How it works  The workflow operates in two phases. First, it scrapes LinkedIn profiles every 12 hours via Apify API, filters posts with 20+ likes/comments/reposts, and saves viral content to Google Sheets. Second, when new data arrives, it analyzes recent posts using Google Gemini AI to identify trends, generates original LinkedIn-optimized content with images, and auto-publishes to your organization page.  ## Setup steps  1. Create a Google Sheet with two sheets: one for LinkedIn profile URLs and one for scraped post data 2. Add Apify API credentials for LinkedIn scraping 3. Configure Google Sheets credentials for data storage 4. Add Google Gemini API key for AI content generation 5. Set up LinkedIn organization credentials for publishing 6. Update all placeholder IDs in nodes with your actual credentials 7. Adjust engagement thresholds and timing based on your needs |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## Profile Scraping  Scheduled trigger runs every 12 hours to fetch LinkedIn profile URLs from Google Sheets and process them in batches of 3 to avoid rate limits. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## Post Extraction  Calls Apify API to scrape recent LinkedIn posts from each profile. Includes 3-second delay between requests to respect API rate limits. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## Engagement Filtering  Filters posts with high engagement (20+ likes, comments, or reposts) and saves viral content to Google Sheets for analysis. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## Data Monitoring  Triggers when new rows are added to Google Sheets. Filters unprocessed posts from the last 3 days and aggregates data for AI analysis. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## AI Content Strategy  Google Gemini analyzes trending patterns from aggregated posts and generates original LinkedIn content following platform best practices. Outputs structured JSON with title, post text, and image prompt. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## Visual Generation  Generates professional LinkedIn-optimized images using Google Imagen based on AI-created prompts. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Comment / documentation | (none) | (none) | ## Status Tracking  Extracts processed post URLs and marks them as analyzed in Google Sheets to prevent duplicate processing. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Google Sheet**
   1. Spreadsheet: “LinkedIn Usernames” (name optional).
   2. Tab 1: “usernames & links” with a column named exactly: `Linkedin url`
   3. Tab 2: “scrape data” with columns (at minimum):  
      - `Linkedin Post URL`, `Creator Name`, `Caption/Text/Copy`, `Likes`, `Comments`, `Reposts`, `Date Published`, `Date Scraped`, `is done`

2) **Phase 1 — Scheduled scraping**
   1. Add **Schedule Trigger** node named “LinkedIn Content Automation Scheduler”
      - Interval: every **12 hours**
   2. Add **Google Sheets** node named “Fetch LinkedIn Profile URLs”
      - Credential: Google Sheets OAuth2
      - Document: your spreadsheet
      - Sheet: “usernames & links”
      - Operation: read/get all rows (the node should output items with `Linkedin url`)
   3. Add **Split In Batches** node named “Process Profiles in Batches”
      - Batch size: **3**
   4. Add **HTTP Request** node named “Scrape LinkedIn Posts API”
      - Method: POST
      - URL: `https://api.apify.com/v2/acts/LQQIXN9Othf8f7R5n/run-sync-get-dataset-items`
      - Send JSON body with:
        - `username`: expression `{{ $json['Linkedin url'] }}`
        - `page_number`: 1
        - `limit`: 1
      - Headers:
        - Accept: application/json
        - Content-Type: application/json
        - Authorization: `Bearer <your_apify_token>`
      - Timeout: 60000 ms
   5. Add **Wait** node named “Rate Limiting Delay”
      - Wait: **3 seconds**
   6. Add **Filter** node named “Filter High-Engagement Posts”
      - OR conditions:
        - `{{$json.stats.reposts}} > 20`
        - `{{$json.stats.like}} > 20`
        - `{{$json.stats.comments}} > 20`
   7. Add **Google Sheets** node named “Save Viral Posts to Sheets”
      - Operation: **Append or Update**
      - Sheet: “scrape data”
      - Matching column: `Caption/Text/Copy`
      - Map fields to the columns listed in section 2 (Likes/Reposts/Comments/etc.)
   8. **Connect nodes (Phase 1):**
      - Scheduler → Fetch URLs → Split In Batches
      - Split In Batches (batch output) → HTTP Request → Wait → Filter → Save Viral
      - Save Viral → Split In Batches (to continue processing next batch)

3) **Phase 2 — Monitoring + AI creation**
   1. Add **Google Sheets Trigger** named “New Post Data Trigger”
      - Poll: every minute
      - Document: same spreadsheet
      - Sheet: “scrape data”
   2. Add **Filter** named “Filter Recent Posts (3 Days)”
      - Condition 1 (number):  
        `{{ Math.floor((new Date().setHours(0,0,0,0) - new Date($json['Date Published']).setHours(0,0,0,0)) / (1000*60*60*24)) }} <= 3`
      - Condition 2 (string): `is done` is empty
   3. Add **Aggregate** node named “Aggregate Trending Content”
      - Mode: aggregate all item data
   4. Add **Google Gemini Chat Model** node named “Google Gemini Chat Model”
      - Credential: Google Gemini(PaLM) / Google Palm API key
   5. Add **JSON Output Parser** node named “JSON Output Parser”
      - Schema example includes `title`, `post`, `image_prompt`
   6. Add **LangChain Agent** node named “LinkedIn Content Strategy AI”
      - Input text: `{{ $json.data }}`
      - System message: paste the provided strategist framework (ensure it requests strict JSON)
      - Enable output parser
      - Connect:
        - Gemini Chat Model → Agent (ai_languageModel)
        - JSON Output Parser → Agent (ai_outputParser)
        - Aggregate → Agent (main)
   7. Add **Google Gemini (Imagen) Image** node named “Generate an image”
      - Resource: image
      - Model: `models/imagen-4.0-generate-preview-06-06`
      - Prompt: use `{{ $json.output.image_prompt }}` (or reference the AI node output directly if needed)
   8. Add **LinkedIn** node named “Publish to LinkedIn”
      - Credential: LinkedIn OAuth2 with organization posting permissions
      - Post as: organization
      - Organization ID: your org ID
      - Share category: IMAGE
      - Title: `{{ $('LinkedIn Content Strategy AI').item.json.output.title }}`
      - Text: `{{ $('LinkedIn Content Strategy AI').item.json.output.post }}`
   9. **Connect nodes (Phase 2):**
      - Sheets Trigger → Filter Recent → Aggregate → Agent → Generate image → Publish to LinkedIn

4) **Status tracking (prevent duplicates)**
   1. Add **Code** node named “Extract Analyzed Post URLs”
      - Use `$items('Filter Recent Posts (3 Days)')` and map `Linkedin Post URL` into `postUrl`
      - Important: ensure the referenced node name matches exactly.
   2. Add **Google Sheets** node named “Mark Posts as Analyzed”
      - Operation: Append or Update
      - Sheet: “scrape data”
      - Matching column: `Linkedin Post URL`
      - Set `is done` = `yes`
   3. Connect: Publish to LinkedIn → Extract Analyzed Post URLs → Mark Posts as Analyzed

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| LinkedIn Viral Content Automation — two-phase workflow: scrape → store → monitor new rows → AI generate post+image → publish → mark done | Sticky note “LinkedIn Viral Content Automation” |
| Setup requires: Google Sheets (2 tabs), Apify token, Google Gemini API key, LinkedIn organization credentials; update all placeholders | Sticky note “LinkedIn Viral Content Automation” |
| Important fix: Code node references non-existent node name `'Filter Posts Within 3 Days'`; should reference `'Filter Recent Posts (3 Days)'` | Status tracking block |
| Engagement thresholds and timing are adjustable; current filter is 20+ likes/comments/reposts and scrape every 12 hours | Sticky notes + Filter node configuration |