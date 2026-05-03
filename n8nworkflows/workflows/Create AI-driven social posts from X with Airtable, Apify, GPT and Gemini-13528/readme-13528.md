Create AI-driven social posts from X with Airtable, Apify, GPT and Gemini

https://n8nworkflows.xyz/workflows/create-ai-driven-social-posts-from-x-with-airtable--apify--gpt-and-gemini-13528


# Create AI-driven social posts from X with Airtable, Apify, GPT and Gemini

## 1. Workflow Overview

**Title:** Create AI-driven social posts from X with Airtable, Apify, GPT and Gemini

**Purpose & use cases:**  
This workflow is a two-part social media engine that (1) **researches and recreates** high-performing X posts into new original short posts + long captions, then (2) **publishes** approved drafts across multiple platforms (X, Threads, LinkedIn, Facebook, Instagram) with an auto-generated text image.

### 1.1 Research & scraping block (Flow 1)
Triggered on a schedule, it fetches an “inspiration username” from Airtable, scrapes recent X posts via Apify, filters and ranks them by view count, and selects the top items.

### 1.2 AI analysis & rewriting block (Flow 2)
For each selected post, Gemini analyzes the structure and psychology; GPT writes a fresh short post; GPT writes a long-form caption.

### 1.3 Draft persistence & cycling block (Flow 2 continuation)
The workflow stores the inspired post, analysis, rewritten post, and caption into an Airtable “Content” table, and marks the inspiration username as “Scraped” to avoid reuse.

### 1.4 Approval-driven publishing block (Flow 3+4)
When a draft is set to **Status = Approved** in Airtable, it generates an image (OpenAI image model), hosts it on ImgBB, posts text to X and Threads, posts image+caption to LinkedIn/Facebook/Instagram (Graph API), then marks the Airtable record as **Posted**.

---

## 2. Block-by-Block Analysis

### Block A — Research trigger & get target username (Flow 1 entry)

**Overview:** Runs daily, pulls the next inspiration username from Airtable that has not been scraped yet.

**Nodes involved:**
- Schedule Trigger
- Get Username

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — entry point for the research pipeline.
- **Configuration:** Triggers daily at **12:00** (server/instance timezone).
- **Outputs:** Sends one empty item to **Get Username**.
- **Edge cases / failures:**
  - Timezone mismatch may run at unexpected hour.
  - If workflow is inactive, nothing runs.

#### Node: Get Username
- **Type / role:** `Airtable` (search) — fetch one record to scrape.
- **Configuration choices (interpreted):**
  - Base: **“Lv1: Text Based Posts”** (`app1OpqBHenznP8od`)
  - Table: **“Inspiration”** (`tblEovZulFXLEjOFv`)
  - Operation: **Search**
  - `filterByFormula`: `{Scraped?} = FALSE()`
  - Limit: `1` record
- **Key fields referenced later:** `Username` (note: output shape depends on Airtable node response; the workflow expects a top-level `Username` in the item JSON).
- **Connections:** Output → **Scrape Tweets**
- **Edge cases / failures:**
  - No matching records → downstream nodes run with empty input (effectively no-op) or may error if expressions assume data exists.
  - Airtable auth/permission errors (PAT scope, base access).
  - If Airtable returns fields nested (e.g., `fields.Username`) but the workflow expects `Username`, expressions may break.

---

### Block B — Scrape X posts & select top-performing posts

**Overview:** Uses Apify’s X scraper actor to pull latest tweets from the target username, removes undesirable posts, sorts by view count, then selects up to 5.

**Nodes involved:**
- Scrape Tweets
- Filter
- Sort by Views
- Limit

#### Node: Scrape Tweets
- **Type / role:** `Apify` — runs an actor and returns dataset items.
- **Configuration choices:**
  - Actor: **“Tweet Scraper|$0.25/1K Tweets | Pay-Per Result | No Rate Limits …”** (`CJdippxWmn9uRfooo`)
  - Operation: **Run actor and get dataset**
  - Custom input body (notable parameters):
    - `from`: `{{ $('Get Username').item.json.Username }}`
    - `lang`: `en`
    - `queryType`: `Latest`
    - `maxItems`: `15`
    - Many `filter:*` flags set to `false` (minimal filtering at source)
- **Connections:** Output → **Filter**
- **Edge cases / failures:**
  - Actor quota/cost limits; dataset may be empty.
  - Username invalid/suspended/private → no results.
  - Apify timeouts or actor changes (breaking output schema: `viewCount`, `text`, etc.).
  - If `Get Username` returns no item, `from` becomes empty and actor may return irrelevant results or fail.

#### Node: Filter
- **Type / role:** `Filter` — remove posts not suitable for reuse.
- **Configuration choices:**
  - Condition group (AND):
    1. `viewCount` **exists** (number)
    2. `text` **does not contain** `"@"` (avoid mentions/replies style)
    3. `text` **does not contain** `"https"` (avoid link posts)
- **Connections:** Output → **Sort by Views**
- **Edge cases / failures:**
  - If Apify uses different field names (e.g., `views` vs `viewCount`), filter drops everything.
  - Posts with “@” in legitimate contexts get removed.
  - Posts containing “https” but still valuable get removed.

#### Node: Sort by Views
- **Type / role:** `Sort` — rank by performance.
- **Configuration:** Sort descending by `viewCount`.
- **Connections:** Output → **Limit**
- **Edge cases / failures:**
  - Missing `viewCount` on some items can affect ordering.

#### Node: Limit
- **Type / role:** `Limit` — keep only top N items.
- **Configuration:** `maxItems = 5`
- **Connections:** Output → **Analysis Agent**
- **Edge cases / failures:**
  - If fewer than 5 items pass filter, fewer are processed (expected).

---

### Block C — AI analysis + rewriting + caption generation (Flow 2)

**Overview:** Gemini analyzes each inspired X post; GPT rewrites a new short post; GPT expands it into a long-form caption suitable for IG/FB/LinkedIn.

**Nodes involved:**
- Gemini 3 Flash (LLM)
- Analysis Agent
- GPT 5.2 (LLM)
- Writing Agent
- Caption Agent

#### Node: Gemini 3 Flash
- **Type / role:** `lmChatGoogleGemini` — language model provider for the Analysis Agent.
- **Configuration:**
  - Model: `models/gemini-3-flash-preview`
- **Connections:** Provides **AI languageModel** input to **Analysis Agent**.
- **Version-specific notes:** Preview models can change behavior and availability.
- **Edge cases / failures:**
  - Gemini API quota/auth issues.
  - Model name deprecation.

#### Node: Analysis Agent
- **Type / role:** `LangChain Agent` — analyzes the post and outputs a structured breakdown + reusable template.
- **Configuration choices:**
  - Input text: `Social media post: {{ $json.text }}`
  - System message: detailed instruction to:
    - break down structure, explain psychological principles,
    - produce fill-in-the-blank “recreation template”
    - avoid rewriting original post
    - no bold syntax; no preamble/conclusion
- **Connections:**
  - Main output → **Writing Agent**
  - AI model supplied by **Gemini 3 Flash**
- **Key variables used downstream:** `$('Analysis Agent').item.json.output`
- **Edge cases / failures:**
  - If `$json.text` is missing (schema mismatch), agent input becomes empty.
  - Output may exceed expected length; downstream storage fields in Airtable could truncate depending on field type.
  - Agent tool failures if misconfigured model connection.

#### Node: GPT 5.2
- **Type / role:** `lmChatOpenAi` — language model provider for Writing Agent and Caption Agent.
- **Configuration:**
  - Model: `gpt-5.2`
  - Built-in tools: none enabled
- **Connections:** Provides **AI languageModel** input to:
  - **Writing Agent**
  - **Caption Agent**
- **Edge cases / failures:**
  - OpenAI auth/quota.
  - Model availability / org permission to access `gpt-5.2`.

#### Node: Writing Agent
- **Type / role:** `LangChain Agent` — rewrites inspired tweet into a new original tweet for the user’s niche.
- **Configuration choices:**
  - Input text:
    - Inspired Post: `{{ $('Limit').item.json.text }}`
    - Analysis: `{{ $('Analysis Agent').item.json.output }}`
  - System message constraints:
    - Tone: timeless, motivational, punchy, casual
    - Target topics: AI/marketing/growth/lead gen/funnels
    - Audience: entrepreneurs struggling to grow/monetize
    - Must be < 280 chars, no emojis, no jargon, no em-dash, output tweet only
- **Connections:** Main output → **Caption Agent**
- **Key output used:** `$('Writing Agent').item.json.output`
- **Edge cases / failures:**
  - Character limit not enforced by system reliably → may exceed 280.
  - If `Limit` node outputs multiple items, this runs per item; later Airtable “Scraped?” update only marks one username, but creates up to 5 drafts (intended).

#### Node: Caption Agent
- **Type / role:** `LangChain Agent` — creates long-form caption + hashtags.
- **Configuration choices:**
  - Input text: `Post: {{ $json.output }}` (uses Writing Agent output)
  - System message constraints:
    - Hook + message + takeaway + CTA + hashtags (3–5)
    - No emojis, no jargon, no em-dash
- **Connections:** Main output → **Create records**
- **Key output used:** `$('Caption Agent').item.json.output`
- **Edge cases / failures:**
  - Hashtags might be too generic or wrong count.
  - If `$json.output` is missing (unexpected agent output format), caption becomes weak/empty.

---

### Block D — Save drafts in Airtable + mark inspiration as scraped

**Overview:** Stores the generated content into Airtable “Content” and marks the inspiration username as scraped to rotate sources.

**Nodes involved:**
- Create records
- Scraped?

#### Node: Create records
- **Type / role:** `Airtable` (create) — persists drafts for review/approval.
- **Configuration choices:**
  - Base: **Lv1: Text Based Posts** (`app1OpqBHenznP8od`)
  - Table: **Content** (`tblbVyuvTdlvoRk4U`)
  - Operation: **Create**
  - Mapped fields (high-signal):
    - `Inspired Post` = `$('Limit').item.json.text`
    - `Analysis` = `$('Analysis Agent').item.json.output`
    - `New Post` = `$('Writing Agent').item.json.output`
    - `New Caption` = `$('Caption Agent').item.json.output`
    - Metrics: Views/Likes/Comments/Reposts/Bookmarks from Apify item
    - Metadata: URL, Date, Name (author)
- **Connections:** Output → **Scraped?**
- **Edge cases / failures:**
  - Airtable field type mismatches (numbers vs strings).
  - Missing Apify fields causes empty inserts.
  - Duplicate content creation: runs for each of up to 5 items daily.

#### Node: Scraped?
- **Type / role:** `Airtable` (update) — sets `Scraped?` checkbox to true for the username record.
- **Configuration choices:**
  - Base/Table: same Inspiration table as Get Username
  - Operation: **Update**
  - Matching column: `Username`
  - Values:
    - `Scraped? = true`
    - `Username = $('Get Username').item.json.Username`
- **Connections:** terminal
- **Edge cases / failures:**
  - If `Username` is not unique in Airtable, multiple matches may update unpredictably (Airtable update-by-matching behavior depends on node implementation).
  - If `Get Username` output is missing, update fails or marks wrong record.

---

### Block E — Approval trigger, image generation & image hosting (Flow 3+4 entry)

**Overview:** When a draft is manually approved in Airtable, generate a text-image from the short post and upload it to ImgBB to obtain a public URL.

**Nodes involved:**
- Approved Trigger
- Generate an image
- Upload Image

#### Node: Approved Trigger
- **Type / role:** `Airtable Trigger` — entry point for publishing.
- **Configuration choices:**
  - Base: `app1OpqBHenznP8od`
  - Table: `tblbVyuvTdlvoRk4U` (Content)
  - Polling: every minute
  - Trigger field: `Last Modified`
  - Formula: `{Status} = "Approved"`
- **Connections:** Fan-out to:
  - **X**
  - **Mark as Posted**
  - **Generate an image**
- **Critical behavior note:** Because it triggers on “Last Modified” with a formula, edits to approved records can retrigger posting unless additional safeguards exist.
- **Edge cases / failures:**
  - Polling delay and repeated triggers (idempotency risk).
  - If “Status” field name/value differs, trigger never fires.

#### Node: Generate an image
- **Type / role:** `OpenAI Image` — creates a vertical text graphic.
- **Configuration choices:**
  - Model: `gpt-image-1`
  - Prompt includes the approved record field: `{{ $json.fields['New Post'] }}`
  - Size: `1024x1536`, quality: `medium`
  - Output binary is expected on property `data` (default behavior for this node/resource)
- **Connections:** Fan-out to:
  - **Facebook** (expects binary `data`)
  - **LinkedIn** (expects image media category; but no explicit binary mapping shown—see edge cases)
  - **Upload Image**
  - **Threads Container**
- **Edge cases / failures:**
  - Text rendering: prompt says “12 pixels text” which may produce unreadably small text at 1024x1536.
  - If the node outputs binary under a different property name, Facebook/ImgBB upload will fail.
  - Rate limits / content policy refusals by image model.

#### Node: Upload Image
- **Type / role:** `HTTP Request` — uploads binary image to ImgBB to get a hosted URL.
- **Configuration choices:**
  - URL: `https://api.imgbb.com/1/upload`
  - Method: POST
  - Content-Type: `multipart-form-data`
  - Auth: `HTTP Query Auth` (ImgBB API key typically passed as query string via credential)
  - Body: form binary `image` from input field `data`
- **Connections:** Output → **Wait 5sec** (Instagram container sequence)
- **Edge cases / failures:**
  - If binary field name is not `data`, upload fails.
  - ImgBB limits/quotas; API key invalid.
  - Response mapping: workflow expects `$('Upload Image').item.json.data.image.url`.

---

### Block F — Publish to text-first platforms (X + Threads)

**Overview:** Posts the short text to X directly, and to Threads via two Graph API calls (create container, then publish after a delay).

**Nodes involved:**
- X
- Threads Container
- Wait 5s
- Threads

#### Node: X
- **Type / role:** `Twitter/X` — creates a post.
- **Configuration:** `text = {{ $json.fields['New Post'] }}`
- **Input source:** From **Approved Trigger**.
- **Connections:** terminal
- **Edge cases / failures:**
  - X API access level limitations (write permissions, app type).
  - Duplicate posting if Approved Trigger retriggers.
  - Character limit issues (agent might exceed 280).

#### Node: Threads Container
- **Type / role:** `HTTP Request` — creates a Threads post container.
- **Configuration:**
  - POST `https://graph.threads.net/v1.0/25302735036057876/threads`
  - Query params:
    - `text = {{ $json.fields['New Post'] }}`
    - `media_type = TEXT`
  - Auth: `HTTP Header Auth` credential (“Threads account”)
- **Connections:** Output → **Wait 5s**
- **Edge cases / failures:**
  - Threads Graph API token expiration/permissions.
  - Requires correct Threads user ID in URL path.
  - API may return errors if posting too frequently.

#### Node: Wait 5s
- **Type / role:** `Wait` — delay to allow container processing.
- **Configuration:** default 5 seconds (node name indicates 5s; no explicit duration shown in parameters, so it relies on node default UI-configured value).
- **Connections:** Output → **Threads**
- **Edge cases / failures:**
  - Too short: publish may fail if container not ready.
  - Too long: increases workflow runtime.

#### Node: Threads
- **Type / role:** `HTTP Request` — publishes the Threads container.
- **Configuration:**
  - POST `https://graph.threads.net/v1.0/25302735036057876/threads_publish`
  - Query param: `creation_id = {{ $json.id }}` (uses container response)
  - Retry on fail enabled, 5s between tries
  - Auth: HTTP Header Auth
- **Connections:** terminal
- **Edge cases / failures:**
  - If container response `id` is missing or renamed, publish fails.
  - Retries may still fail if the underlying error is permission-related.

---

### Block G — Publish visual posts to LinkedIn, Facebook, Instagram (image + caption)

**Overview:** Uses the generated image for visual platforms. Facebook posts a photo with message, LinkedIn posts an image share with text, Instagram uses a “create media” container then “publish” sequence, after delays.

**Nodes involved:**
- LinkedIn
- Facebook
- Instagram Container
- Wait 5 sec
- Instagram
- Wait 5sec
- Wait 5 sec (note: there are multiple wait nodes with similar names)

#### Node: LinkedIn
- **Type / role:** `LinkedIn` — publish a post with image category.
- **Configuration:**
  - Text: `{{ $('Approved Trigger').item.json.fields['New Caption'] }}`
  - Person URN/id: `nY1EvjMSIJ`
  - `shareMediaCategory = IMAGE`
- **Input source:** From **Generate an image** (fan-out).
- **Potential integration issue:** The node is configured for image posting, but the workflow JSON does not show explicit mapping of binary image or asset upload step. LinkedIn image posting typically requires uploading the image asset first or providing media content via node-specific fields.
- **Edge cases / failures:**
  - Missing media binary/asset → LinkedIn API error.
  - Permissions: requires correct LinkedIn scopes (w_member_social or equivalent).

#### Node: Facebook
- **Type / role:** `Facebook Graph API` — publish a photo to a page.
- **Configuration:**
  - Edge: `photos`
  - Node (Page ID): `884325181427396`
  - Method: POST, Graph API version `v23.0`
  - `sendBinaryData = true`, `binaryPropertyName = data`
  - Message query parameter: `{{ $('Approved Trigger').item.json.fields['New Caption'] }}`
- **Input source:** From **Generate an image** (must provide binary `data`).
- **Edge cases / failures:**
  - Page permissions (pages_manage_posts, pages_read_engagement, etc.).
  - If binary is missing, Graph API returns “No file was uploaded”.
  - Caption length limits or forbidden content.

#### Node: Wait 5sec
- **Type / role:** `Wait` — delay before creating Instagram container.
- **Configuration:** default wait; named “Wait 5sec”.
- **Connections:** Output → **Instagram Container**
- **Input source:** From **Upload Image**
- **Edge cases / failures:** Same as other waits; also if ImgBB response lacks URL, Instagram container fails next.

#### Node: Instagram Container
- **Type / role:** `Facebook Graph API` — create Instagram media container.
- **Configuration:**
  - Edge: `media`
  - Node (Instagram Business Account ID): `17841462646273569`
  - Query params:
    - `image_url = {{ $('Upload Image').item.json.data.image.url }}`
    - `caption = {{ $('Approved Trigger').item.json.fields['New Caption'] }}`
  - Method: POST, Graph API version `v23.0`
- **Connections:** Output → **Wait 5 sec**
- **Edge cases / failures:**
  - Instagram requires the image URL to be publicly accessible and stable.
  - If ImgBB URL is slow/blocked, container creation fails.
  - Caption may exceed IG limits (rare but possible).

#### Node: Wait 5 sec
- **Type / role:** `Wait` — allow IG container processing.
- **Connections:** Output → **Instagram**

#### Node: Instagram
- **Type / role:** `Facebook Graph API` — publish Instagram media container.
- **Configuration:**
  - Edge: `media_publish`
  - Node: `17841462646273569`
  - Query param: `creation_id = {{ $json.id }}`
  - Retry on fail enabled, 5s between tries
- **Connections:** terminal
- **Edge cases / failures:**
  - Container not ready yet (timing).
  - Token permissions: instagram_content_publish, pages/ig linkage requirements.

---

### Block H — Mark Airtable record as posted

**Overview:** Updates the Airtable content record after triggering publishing by setting Status to “Posted”.

**Nodes involved:**
- Mark as Posted

#### Node: Mark as Posted
- **Type / role:** `Airtable` (update) — changes status.
- **Configuration choices:**
  - Base: shown cached as “Social Media Posts” but ID is the same base (`app1OpqBHenznP8od`)
  - Table: `tblbVyuvTdlvoRk4U`
  - Operation: Update
  - Matching column: `Title`
  - Sets:
    - `Title = {{ $json.fields.Title }}`
    - `Status = Posted`
- **Connections:** Triggered directly from **Approved Trigger** (in parallel with posting).
- **Important behavior note:** This marks as posted immediately, not “after all publishing nodes succeed”, because it is not chained after the publish nodes.
- **Edge cases / failures:**
  - If `Title` is empty or non-unique, wrong record may be updated or update fails.
  - Race condition: it can set Posted even if posting later fails.
  - Better practice would be matching by Airtable record ID.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | Sticky Note | Workspace description / requirements |  |  | # Social Media Post Automation for Content Creators … Ask in the [n8n Forum](https://community.n8n.io/) or shoot me a DM on [LinkedIn](https://www.linkedin.com/in/vincentthenguyen/) … |
| Sticky Note5 | Sticky Note | Section header |  |  | # Flow 1+2: Research & Recreating |
| Sticky Note2 | Sticky Note | Explains research block |  |  | ### 1. Research & Scrape Top Content … |
| Schedule Trigger | Schedule Trigger | Start daily research run |  | Get Username | ### 1. Research & Scrape Top Content … |
| Get Username | Airtable | Fetch next username to scrape | Schedule Trigger | Scrape Tweets | ### 1. Research & Scrape Top Content … |
| Scrape Tweets | Apify | Scrape latest X posts for username | Get Username | Filter | ### 1. Research & Scrape Top Content … |
| Filter | Filter | Remove posts with missing views/mentions/links | Scrape Tweets | Sort by Views | ### 1. Research & Scrape Top Content … |
| Sort by Views | Sort | Rank posts by viewCount | Filter | Limit | ### 1. Research & Scrape Top Content … |
| Limit | Limit | Keep top 5 posts | Sort by Views | Analysis Agent | ### 1. Research & Scrape Top Content … |
| Sticky Note3 | Sticky Note | Explains AI block |  |  | ### 2. AI Analysis & Rewriting … |
| Gemini 3 Flash | Google Gemini Chat Model | LLM provider for analysis |  | Analysis Agent (ai_languageModel) | ### 2. AI Analysis & Rewriting … |
| Analysis Agent | LangChain Agent | Analyze structure/psychology & produce template | Limit + Gemini 3 Flash | Writing Agent | ### 2. AI Analysis & Rewriting … |
| GPT 5.2 | OpenAI Chat Model | LLM provider for writing/caption |  | Writing Agent, Caption Agent (ai_languageModel) | ### 2. AI Analysis & Rewriting … |
| Writing Agent | LangChain Agent | Write fresh short post from analysis | Analysis Agent + GPT 5.2 | Caption Agent | ### 2. AI Analysis & Rewriting … |
| Caption Agent | LangChain Agent | Generate long-form caption + hashtags | Writing Agent + GPT 5.2 | Create records | ### 2. AI Analysis & Rewriting … |
| Sticky Note4 | Sticky Note | Explains saving drafts |  |  | ### 3. Save Drafts for Approval … |
| Create records | Airtable | Create draft rows in Content table | Caption Agent | Scraped? | ### 3. Save Drafts for Approval … |
| Scraped? | Airtable | Mark inspiration username scraped | Create records |  | ### 3. Save Drafts for Approval … |
| Sticky Note | Sticky Note | Section header |  |  | # Flow 3+4: Auto-Posting & Repurposing |
| Sticky Note6 | Sticky Note | Explains approval + image generation |  |  | ### 4. Approval Trigger & Image Generation … |
| Approved Trigger | Airtable Trigger | Start publishing when Status=Approved |  | X, Mark as Posted, Generate an image | ### 4. Approval Trigger & Image Generation … |
| Generate an image | OpenAI Image | Create text-based image graphic | Approved Trigger | Facebook, LinkedIn, Upload Image, Threads Container | ### 4. Approval Trigger & Image Generation … |
| Upload Image | HTTP Request | Upload image to ImgBB and get public URL | Generate an image | Wait 5sec | ### 4. Approval Trigger & Image Generation … |
| Sticky Note7 | Sticky Note | Explains X + Threads posting |  |  | ### 5. Publish Text to X and Threads … |
| X | Twitter/X | Publish short text post | Approved Trigger |  | ### 5. Publish Text to X and Threads … |
| Threads Container | HTTP Request | Create Threads post container | Generate an image | Wait 5s | ### 5. Publish Text to X and Threads … |
| Wait 5s | Wait | Delay before Threads publish | Threads Container | Threads | ### 5. Publish Text to X and Threads … |
| Threads | HTTP Request | Publish Threads container | Wait 5s |  | ### 5. Publish Text to X and Threads … |
| Sticky Note8 | Sticky Note | Explains visual posting |  |  | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| LinkedIn | LinkedIn | Publish image-category post with caption | Generate an image |  | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| Facebook | Facebook Graph API | Publish photo + caption to Page | Generate an image |  | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| Wait 5sec | Wait | Delay before IG container | Upload Image | Instagram Container | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| Instagram Container | Facebook Graph API | Create IG media container | Wait 5sec | Wait 5 sec | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| Wait 5 sec | Wait | Delay before IG publish | Instagram Container | Instagram | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| Instagram | Facebook Graph API | Publish IG container | Wait 5 sec |  | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| Mark as Posted | Airtable | Update Airtable status to Posted | Approved Trigger |  | ### 6. Publish Visuals to LinkedIn, Facebook & Instagram … |
| ebd354d7… Wait 5 sec (duplicate name already listed) | Wait | Delay node used in IG publish sequence | Instagram Container | Instagram | (covered above; same sticky note content applies) |
| 284453f7… Wait 5sec | Wait | Unused wait node (not connected) |  |  |  |
| Sticky Note5/6/7/8 etc. | Sticky Note | Documentation only |  |  | (as above) |

Note: The node named **“Wait 5sec”** with id `284453f7-...` is present but has **no connections** in this workflow.

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials in n8n (Credentials UI)**
   1. Airtable Personal Access Token credential with access to base `app1OpqBHenznP8od`.
   2. Apify API token credential.
   3. Google Gemini (PaLM) credential for `models/gemini-3-flash-preview`.
   4. OpenAI API credential with access to:
      - `gpt-5.2` (chat model)
      - `gpt-image-1` (image generation)
   5. X (Twitter) OAuth2 credential with write permissions.
   6. LinkedIn OAuth2 credential with permission to post (member social).
   7. Facebook Graph API credential with:
      - Facebook Page posting permissions
      - Instagram Business publishing permissions
   8. HTTP Header Auth credential for Threads Graph API (Bearer token in header).
   9. HTTP Query Auth credential for ImgBB (API key as query parameter).

2. **Build Flow 1: Schedule → Airtable username → Apify scrape**
   1. Add **Schedule Trigger**
      - Set daily trigger at **12:00**.
   2. Add **Airtable** node named **Get Username**
      - Operation: **Search**
      - Base: `app1OpqBHenznP8od`
      - Table: `tblEovZulFXLEjOFv` (“Inspiration”)
      - `Filter by formula`: `{Scraped?} = FALSE()`
      - Limit: `1`
   3. Connect **Schedule Trigger → Get Username**.
   4. Add **Apify** node named **Scrape Tweets**
      - Operation: “Run actor and get dataset”
      - Actor ID: `CJdippxWmn9uRfooo`
      - Custom input JSON includes:
        - `"from": "{{ $('Get Username').item.json.Username }}"`
        - `"maxItems": 15`, `"lang": "en"`, `"queryType": "Latest"`
        - other filters set false
   5. Connect **Get Username → Scrape Tweets**.

3. **Filter/sort/limit**
   1. Add **Filter** node named **Filter**
      - Conditions (AND):
        - `{{$json.viewCount}}` exists (number exists)
        - `{{$json.text}}` does not contain `@`
        - `{{$json.text}}` does not contain `https`
   2. Add **Sort** node named **Sort by Views**
      - Sort field: `viewCount` descending.
   3. Add **Limit** node named **Limit**
      - Max items: `5`
   4. Connect: **Scrape Tweets → Filter → Sort by Views → Limit**.

4. **AI analysis and writing (LangChain Agents + model nodes)**
   1. Add **Google Gemini Chat Model** node named **Gemini 3 Flash**
      - Model: `models/gemini-3-flash-preview`
   2. Add **LangChain Agent** node named **Analysis Agent**
      - Prompt type: “Define”
      - Text: `Social media post: {{ $json.text }}`
      - Paste the system message from the workflow (analysis + template requirements).
      - Connect **Gemini 3 Flash** to **Analysis Agent** via the **ai_languageModel** connection.
   3. Add **OpenAI Chat Model** node named **GPT 5.2**
      - Model: `gpt-5.2`
   4. Add **LangChain Agent** node named **Writing Agent**
      - Text:
        - `Inspired Post: {{ $('Limit').item.json.text }}`
        - `Analysis: {{ $('Analysis Agent').item.json.output }}`
      - System message: paste constraints (<280 chars, no emojis, etc.).
      - Connect **GPT 5.2** to **Writing Agent** via **ai_languageModel**.
   5. Add **LangChain Agent** node named **Caption Agent**
      - Text: `Post: {{ $json.output }}`
      - System message: caption structure + hashtags + constraints.
      - Connect **GPT 5.2** to **Caption Agent** via **ai_languageModel**.
   6. Wire main flow: **Limit → Analysis Agent → Writing Agent → Caption Agent**.

5. **Save draft to Airtable + mark inspiration scraped**
   1. Add **Airtable** node named **Create records**
      - Operation: **Create**
      - Base: `app1OpqBHenznP8od`
      - Table: `tblbVyuvTdlvoRk4U` (“Content”)
      - Map fields:
        - URL, Date, Name, Likes, Views, Reposts, Comments, Bookmarks from the scraped item
        - Analysis from Analysis Agent output
        - New Post from Writing Agent output
        - New Caption from Caption Agent output
        - Inspired Post from scraped item text
   2. Connect **Caption Agent → Create records**.
   3. Add **Airtable** node named **Scraped?**
      - Operation: **Update**
      - Base: `app1OpqBHenznP8od`
      - Table: `tblEovZulFXLEjOFv` (“Inspiration”)
      - Matching column: `Username`
      - Set `Scraped? = true` and `Username = {{ $('Get Username').item.json.Username }}`
   4. Connect **Create records → Scraped?**.

6. **Build Flow 3+4: Approval trigger**
   1. Add **Airtable Trigger** node named **Approved Trigger**
      - Base: `app1OpqBHenznP8od`
      - Table: `tblbVyuvTdlvoRk4U`
      - Trigger field: `Last Modified`
      - Poll: every minute
      - Formula: `{Status} = "Approved"`

7. **Image generation + ImgBB hosting**
   1. Add **OpenAI Image** node named **Generate an image**
      - Resource: Image
      - Model: `gpt-image-1`
      - Prompt includes: `{{ $json.fields['New Post'] }}`
      - Size: `1024x1536`, Quality: `medium`
   2. Connect **Approved Trigger → Generate an image**.
   3. Add **HTTP Request** node named **Upload Image**
      - POST `https://api.imgbb.com/1/upload`
      - Auth: HTTP Query Auth (ImgBB key)
      - Content-Type: multipart/form-data
      - Body form param `image` = binary from `data`
   4. Connect **Generate an image → Upload Image**.

8. **Post to X**
   1. Add **X (Twitter)** node named **X**
      - Text: `{{ $json.fields['New Post'] }}`
   2. Connect **Approved Trigger → X**.

9. **Post to Threads (container + publish)**
   1. Add **HTTP Request** node named **Threads Container**
      - POST `https://graph.threads.net/v1.0/<THREADS_USER_ID>/threads`
      - Query params: `text={{ $json.fields['New Post'] }}`, `media_type=TEXT`
      - Auth: HTTP Header Auth (Bearer token)
   2. Add **Wait** node named **Wait 5s** (5 seconds)
   3. Add **HTTP Request** node named **Threads**
      - POST `https://graph.threads.net/v1.0/<THREADS_USER_ID>/threads_publish`
      - Query param: `creation_id={{ $json.id }}`
      - Retry on fail enabled, wait between tries 5000ms
   4. Connect: **Generate an image → Threads Container → Wait 5s → Threads**  
      (Note: the original workflow starts Threads from Generate an image; you could also start from Approved Trigger if preferred.)

10. **Post to Facebook Page**
   1. Add **Facebook Graph API** node named **Facebook**
      - Edge: `photos`
      - Node: your Page ID
      - Send binary data: true; Binary property: `data`
      - Query param `message = {{ $('Approved Trigger').item.json.fields['New Caption'] }}`
   2. Connect **Generate an image → Facebook**.

11. **Post to LinkedIn**
   1. Add **LinkedIn** node named **LinkedIn**
      - Person: your LinkedIn identifier
      - Text: `{{ $('Approved Trigger').item.json.fields['New Caption'] }}`
      - Share media category: IMAGE
   2. Connect **Generate an image → LinkedIn**.
   3. If LinkedIn requires an explicit upload step in your n8n version, add the appropriate LinkedIn “upload image/register upload” configuration or intermediary node so the IMAGE share has an actual asset.

12. **Post to Instagram (container + publish)**
   1. Add **Wait** node named **Wait 5sec** (5 seconds) after ImgBB upload.
   2. Add **Facebook Graph API** node named **Instagram Container**
      - Edge: `media`
      - Node: Instagram Business Account ID
      - Query params:
        - `image_url = {{ $('Upload Image').item.json.data.image.url }}`
        - `caption = {{ $('Approved Trigger').item.json.fields['New Caption'] }}`
   3. Add **Wait** node named **Wait 5 sec** (5 seconds).
   4. Add **Facebook Graph API** node named **Instagram**
      - Edge: `media_publish`
      - Query param `creation_id = {{ $json.id }}`
      - Retry on fail enabled
   5. Connect: **Upload Image → Wait 5sec → Instagram Container → Wait 5 sec → Instagram**.

13. **Mark Airtable record as posted**
   1. Add **Airtable** node named **Mark as Posted**
      - Operation: **Update**
      - Base: `app1OpqBHenznP8od`
      - Table: `tblbVyuvTdlvoRk4U`
      - Matching column: ideally use **Record ID**; if replicating exactly, match on `Title`
      - Set `Status = Posted`
   2. Connect **Approved Trigger → Mark as Posted** (as in the workflow), or better: connect it after all publish nodes succeed to avoid false “Posted”.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Social Media Post Automation for Content Creators — two-part engine (research/rewrite + approval-based multi-platform publishing). | Sticky note description covering the whole workflow. |
| Support/community: “Ask in the n8n Forum” | https://community.n8n.io/ |
| Author contact: LinkedIn | https://www.linkedin.com/in/vincentthenguyen/ |
| Key operational concept: you approve drafts in Airtable; posting triggers only when Status becomes “Approved”. | Workflow design principle (sticky note content). |
| Risk to consider: “Mark as Posted” runs in parallel with publishing; it can mark Posted even if publishing fails. | Architectural note from current connections. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.