Cross-post social content from Google Sheets to 5 platforms with OpenAI images

https://n8nworkflows.xyz/workflows/cross-post-social-content-from-google-sheets-to-5-platforms-with-openai-images-13323


# Cross-post social content from Google Sheets to 5 platforms with OpenAI images

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Cross-post social content from Google Sheets to 5 platforms with OpenAI images

**Purpose / use case:**  
This workflow automatically cross-posts social content managed in Google Sheets to **X (Twitter), Threads, LinkedIn, Facebook, and Instagram**. It also generates a **custom AI image** (OpenAI image model) based on the post text, uploads it to **ImgBB** to get a public URL, and uses that image for platforms that benefit from visuals (LinkedIn/Facebook/Instagram).

### 1.1 Logical Blocks
1. **Trigger + Content Intake (Sheets selection)**
   - Scheduled runs pull exactly one row where `Status = Ready To Post`.
2. **Text-only Publishing**
   - Post text is sent to **X** and **Threads** (two-step Threads API).
3. **AI Image Pipeline**
   - An AI agent turns the post into an *image prompt* (LLM via OpenRouter Gemini), then OpenAI generates an image, then ImgBB hosts it.
4. **Image-based Publishing**
   - Uses generated image (binary and/or hosted URL) to publish to **LinkedIn**, **Facebook**, and **Instagram**.
5. **Sheet Status Update (success vs error)**
   - Updates Google Sheets status to **Posted** or **Errored** (as designed by sticky note).  
   **Important:** In the provided workflow wiring, “Done” and “Error” are both connected directly from “Get Content”, not from success/failure branches—so it currently cannot accurately reflect success vs failure without modification (details below).

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger + Get Content
**Overview:** Runs on a schedule and fetches a single “Ready To Post” row from Google Sheets to drive all downstream publishing.

**Nodes involved:**
- Schedule Trigger
- Get Content

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point, timed execution.
- **Configuration (interpreted):** Runs daily at three times:
  - 07:03
  - 12:01
  - 19:02
- **Inputs/outputs:** No inputs; outputs to **Get Content**.
- **Potential failures / edge cases:**
  - Misconfigured timezone (n8n instance timezone affects actual run times).
  - Workflow inactive => no runs.

#### Node: Get Content
- **Type / role:** `n8n-nodes-base.googleSheets` — reads one sheet row to post.
- **Configuration (interpreted):**
  - Spreadsheet: “(Old) Auto Posting” (document ID `100OG7...sdA`)
  - Sheet tab: `Sheet1` (`gid=0`)
  - Filter: `Status` equals **Ready To Post**
  - Option: **Return First Match** enabled (prevents multiple posts per run)
- **Key fields used later:** `Title`, `Content`, `Status` (plus implicit sheet row metadata depending on node output).
- **Outputs / connections:**
  - To **Image Prompt Agent**
  - To **X**
  - To **Create Threads**
  - Also (currently) to **Done** and **Error** (see Block 5 issue)
- **Potential failures / edge cases:**
  - OAuth permission errors / expired Google credential.
  - No matching rows → node may output **0 items**; downstream nodes may not run or may error depending on node behavior.
  - Column naming mismatch (expects “Status”, “Content”, “Title”).
  - If multiple rows are “Ready To Post”, only the first match is processed each run.

---

### Block 2 — Publish Text to X and Threads
**Overview:** Posts the sheet’s `Content` as plain text to X and Threads. Threads uses a create-then-publish sequence.

**Nodes involved:**
- X
- Create Threads
- Threads

#### Node: X
- **Type / role:** `n8n-nodes-base.twitter` — publishes a tweet/post.
- **Configuration (interpreted):**
  - Text: `{{ $json.Content }}`
  - Uses X OAuth2 credentials.
- **Inputs/outputs:**
  - Input from **Get Content**
  - No downstream connections in this workflow.
- **Potential failures / edge cases:**
  - OAuth2 scope issues, revoked access, or app permissions.
  - Text length limits (X limits; if Content exceeds, request fails).
  - Rate limits and API errors.

#### Node: Create Threads
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Threads API to create a thread container.
- **Configuration (interpreted):**
  - POST `https://graph.threads.net/v1.0/{your_threads-id}/threads`
  - Query parameters:
    - `text = {{ $json.Content }}`
    - `media_type = TEXT`
  - Auth: Generic credential type **httpHeaderAuth** (expects an Authorization header token).
- **Inputs/outputs:**
  - Input from **Get Content**
  - Output to **Threads** node (expects response with an `id`).
- **Potential failures / edge cases:**
  - Placeholder `{your_threads-id}` must be replaced with a real Threads user ID.
  - Missing/invalid token in header auth.
  - API versioning/permissions changes on Threads Graph API.
  - If the response format changes and no `id` is returned, publishing will fail.

#### Node: Threads
- **Type / role:** `n8n-nodes-base.httpRequest` — publishes the created Threads post.
- **Configuration (interpreted):**
  - POST `https://graph.threads.net/v1.0/{your_threads-id}/threads_publish`
  - Query parameters:
    - `creation_id = {{ $json.id }}`
  - Auth: same header auth credential as Create Threads.
- **Inputs/outputs:**
  - Input from **Create Threads**
  - No downstream connections.
- **Potential failures / edge cases:**
  - Same `{your_threads-id}` placeholder issue.
  - If Create Threads fails or returns no `id`, this fails.
  - Rate limiting / permissions.

---

### Block 3 — AI Image Generation (Prompt → Image → Hosting)
**Overview:** Converts the social post into a marketing-style image prompt using an LLM agent, generates an image using OpenAI’s image model, then uploads it to ImgBB to get a shareable URL.

**Nodes involved:**
- OpenRouter Gemini 2.5 Flash (LLM)
- Image Prompt Agent (LangChain Agent)
- Generate an image (OpenAI image)
- ImgBB (HTTP upload)

#### Node: OpenRouter Gemini 2.5 Flash
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides an LLM backend to the agent.
- **Configuration (interpreted):**
  - Model: `google/gemini-2.5-flash`
  - Credentials: OpenRouter API key
- **Connections:**
  - Connected via **ai_languageModel** output to **Image Prompt Agent** (agent uses this model).
- **Potential failures / edge cases:**
  - OpenRouter API key invalid, quota exceeded, model unavailable.
  - Latency/timeouts for long posts.

#### Node: Image Prompt Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — transforms post text into a single image prompt string.
- **Configuration (interpreted):**
  - Input text: `Social Media Post: {{ $json.Content }}`
  - Prompt type: “define”
  - **System message:** strict instruction set to output **only** the final image prompt, marketing-style, LinkedIn-appropriate, no quotes, no extra commentary.
  - Uses OpenRouter Gemini via ai_languageModel connection.
- **Outputs / connections:**
  - Output to **Generate an image**
  - Produces something like `{ output: "…image prompt…" }` (the workflow later references `$json.output`).
- **Potential failures / edge cases:**
  - If the agent returns a different field name than `output`, downstream prompt expression breaks.
  - Content with sensitive terms could trigger model refusal (depends on provider policy).
  - Very short/empty content → weak prompt.

#### Node: Generate an image
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — generates an image using OpenAI.
- **Configuration (interpreted):**
  - Resource: `image`
  - Model: `gpt-image-1`
  - Prompt: `{{ $json.output }}`
- **Outputs / connections:**
  - Outputs to **ImgBB**, **LinkedIn**, **Facebook** (all directly from this node).
  - Expected output typically includes **binary image data** (how it appears depends on node version/config). Facebook node expects binary property `data` (see below).
- **Potential failures / edge cases:**
  - OpenAI credential invalid / billing/quota issues.
  - Prompt policy refusal → no image returned.
  - Output format mismatch: downstream nodes assume binary field `data` exists.

#### Node: ImgBB
- **Type / role:** `n8n-nodes-base.httpRequest` — uploads image binary to ImgBB for a public URL.
- **Configuration (interpreted):**
  - POST `https://api.imgbb.com/1/upload`
  - Content-Type: multipart/form-data
  - Body parameter:
    - `image` = binary file from property **`data`**
  - Auth: `httpQueryAuth` (API key passed as query parameter via credential)
- **Outputs / connections:**
  - Output to **Instagram Image**
  - The workflow later uses `{{ $json.data.url }}` from ImgBB response.
- **Potential failures / edge cases:**
  - ImgBB key invalid / quota exceeded.
  - If OpenAI node doesn’t output binary under `data`, upload fails.
  - Response structure changes could break `data.url` reference.

---

### Block 4 — Publish Image Posts (LinkedIn, Facebook, Instagram)
**Overview:** Publishes the caption plus AI image to LinkedIn, Facebook, and Instagram. Instagram requires two Graph calls: create media object with `image_url`, then publish it.

**Nodes involved:**
- LinkedIn
- Facebook
- Instagram Image
- Instagram

#### Node: LinkedIn
- **Type / role:** `n8n-nodes-base.linkedIn` — creates a LinkedIn post with image category.
- **Configuration (interpreted):**
  - Text: `{{ $('Get Content').item.json.Content }}`
  - Person URN/ID: `B5TC_VlQdn`
  - Share media category: `IMAGE`
- **Inputs/outputs:**
  - Input from **Generate an image**
  - No downstream connections.
- **Potential failures / edge cases:**
  - LinkedIn API often requires a multi-step upload/register process; this node abstracts it, but may fail if:
    - OAuth scopes insufficient
    - Account type restrictions
    - Image format/size unsupported
  - If the incoming item from Generate an image doesn’t contain the expected binary/image field, post fails.

#### Node: Facebook
- **Type / role:** `n8n-nodes-base.facebookGraphApi` — posts a photo to a Facebook Page with message.
- **Configuration (interpreted):**
  - Graph edge: `/{page-id}/photos` with page node `884325181427396`
  - Method: POST
  - Sends binary data: **enabled**
  - Binary property name: `data`
  - Query params:
    - `message = {{ $('Get Content').item.json.Content }}`
- **Inputs/outputs:**
  - Input from **Generate an image**
- **Potential failures / edge cases:**
  - Page permissions missing (`pages_manage_posts`, `pages_read_engagement`, etc.).
  - Token expiration.
  - If binary property `data` is missing (image generation output mismatch), upload fails.

#### Node: Instagram Image
- **Type / role:** `n8n-nodes-base.facebookGraphApi` — creates an Instagram media container.
- **Configuration (interpreted):**
  - Node (IG user): `17841462646273569`
  - Edge: `media`
  - Method: POST
  - Query params:
    - `image_url = {{ $json.data.url }}` (from ImgBB response)
    - `caption = {{ $('Get Content').item.json.Content }}`
- **Inputs/outputs:**
  - Input from **ImgBB**
  - Output to **Instagram** (expects response with `id`).
- **Potential failures / edge cases:**
  - IG account must be a professional account linked to a Facebook Page.
  - The `image_url` must be publicly accessible and valid; ImgBB URL must not require auth.
  - If ImgBB response path differs, `$json.data.url` breaks.

#### Node: Instagram
- **Type / role:** `n8n-nodes-base.facebookGraphApi` — publishes the Instagram media container.
- **Configuration (interpreted):**
  - Node: `17841462646273569`
  - Edge: `media_publish`
  - Method: POST
  - Query params:
    - `creation_id = {{ $json.id }}`
    - `caption = {{ $('Get Content').item.json.Content }}` (note: caption is typically set during container creation; keeping it here may be ignored by API)
- **Inputs/outputs:**
  - Input from **Instagram Image**
- **Potential failures / edge cases:**
  - Container not ready yet (sometimes needs polling); immediate publish may error for larger images.
  - Permissions/scopes and app review restrictions.

---

### Block 5 — Update Google Sheet (Posted vs Errored)
**Overview:** Intended to mark the processed row as Posted if successful or Errored if something fails.

**Nodes involved:**
- Done
- Error

#### Node: Done
- **Type / role:** `n8n-nodes-base.googleSheets` — updates sheet row status to Posted.
- **Configuration (interpreted):**
  - Operation: Update
  - Match row where `Title = {{ $('Get Content').item.json.Title }}`
  - Set `Status = Posted`
  - Uses “define below” mapping; only Title and Status are written.
- **Inputs/outputs:**
  - **Input currently from Get Content** (not from completion of publishing).
- **Potential failures / edge cases:**
  - Matching by **Title** is risky: duplicates overwrite the wrong row.
  - If Title is empty, update may fail or update unintended rows.
  - If multiple rows share Title, the “update” behavior may update first match or error (depends on node behavior).

#### Node: Error
- **Type / role:** `n8n-nodes-base.googleSheets` — updates sheet row status to Errored.
- **Configuration (interpreted):**
  - Operation: Update
  - Match row where `Title = {{ $('Get Content').item.json.Title }}`
  - Set `Status = Errored`
- **Inputs/outputs:**
  - **Input currently from Get Content**.
- **Major wiring issue (logic):**
  - Because both **Done** and **Error** run directly after **Get Content**, the workflow does **not** currently implement “if success then Posted, else Errored”.
  - As-is, both nodes will execute in parallel on every run (unless n8n execution stops on error before reaching one branch), and whichever runs last could “win”.
- **Recommended fix (design intent):**
  - Use an **Error Trigger** workflow or node-level error handling, or route to Done only after all publish nodes succeed (e.g., merge branches), and route failures to Error via an error workflow or “Continue On Fail” + IF logic.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Timed entry point | — | Get Content | ## 1. Trigger + Get Content / A Schedule Trigger runs the workflow at set times. / Each run: / * Looks at your Google Sheet / * Finds one row where **Status = Ready To Post** / * Sends that post into the rest of the workflow / This keeps everything automatic and prevents duplicates. |
| Get Content | Google Sheets | Fetch first row with Status=Ready To Post | Schedule Trigger | Image Prompt Agent; Error; Done; X; Create Threads | ## 1. Trigger + Get Content / A Schedule Trigger runs the workflow at set times. / Each run: / * Looks at your Google Sheet / * Finds one row where **Status = Ready To Post** / * Sends that post into the rest of the workflow / This keeps everything automatic and prevents duplicates. |
| OpenRouter Gemini 2.5 Flash | OpenRouter LLM (LangChain) | LLM provider for the agent | — (model connection) | Image Prompt Agent (ai_languageModel) |  |
| Image Prompt Agent | LangChain Agent | Turn post text into an image prompt | Get Content; OpenRouter Gemini 2.5 Flash (model) | Generate an image | ## 3. AI Image Generation / Your post is analyzed by an AI agent that creates a visual prompt. / Then: / * GPT-Image generates a matching graphic / * The image is uploaded to ImgBB / * A shareable image URL is created for reuse |
| Generate an image | OpenAI Image (LangChain) | Generate AI image from prompt | Image Prompt Agent | ImgBB; LinkedIn; Facebook | ## 3. AI Image Generation / Your post is analyzed by an AI agent that creates a visual prompt. / Then: / * GPT-Image generates a matching graphic / * The image is uploaded to ImgBB / * A shareable image URL is created for reuse |
| ImgBB | HTTP Request | Host image and return public URL | Generate an image | Instagram Image | ## 3. AI Image Generation / Your post is analyzed by an AI agent that creates a visual prompt. / Then: / * GPT-Image generates a matching graphic / * The image is uploaded to ImgBB / * A shareable image URL is created for reuse |
| Instagram Image | Facebook Graph API | Create Instagram media container using image URL | ImgBB | Instagram | ## 4. Publish Image Posts / The hosted AI image + your caption are posted to: / * LinkedIn / * Facebook / * Instagram / Each platform receives the same message with the same custom visual. |
| Instagram | Facebook Graph API | Publish Instagram media container | Instagram Image | — | ## 4. Publish Image Posts / The hosted AI image + your caption are posted to: / * LinkedIn / * Facebook / * Instagram / Each platform receives the same message with the same custom visual. |
| LinkedIn | LinkedIn | Post image + caption to LinkedIn | Generate an image | — | ## 4. Publish Image Posts / The hosted AI image + your caption are posted to: / * LinkedIn / * Facebook / * Instagram / Each platform receives the same message with the same custom visual. |
| Facebook | Facebook Graph API | Post photo + message to Facebook Page | Generate an image | — | ## 4. Publish Image Posts / The hosted AI image + your caption are posted to: / * LinkedIn / * Facebook / * Instagram / Each platform receives the same message with the same custom visual. |
| X | Twitter/X | Post text to X | Get Content | — | ## 2. Publish Text to X and Threads / Your post text is sent directly to: / * X (Twitter) / * Threads / No images are required for this part — it’s pure text posting. |
| Create Threads | HTTP Request | Create Threads post container | Get Content | Threads | ## 2. Publish Text to X and Threads / Your post text is sent directly to: / * X (Twitter) / * Threads / No images are required for this part — it’s pure text posting. |
| Threads | HTTP Request | Publish Threads container | Create Threads | — | ## 2. Publish Text to X and Threads / Your post text is sent directly to: / * X (Twitter) / * Threads / No images are required for this part — it’s pure text posting. |
| Done | Google Sheets | Update row Status=Posted | Get Content | — | ## 5. Update Google Sheet / If everything works: / Status → Posted / If anything fails: / Status → Errored / This makes it easy to track what went live and what needs fixing. |
| Error | Google Sheets | Update row Status=Errored | Get Content | — | ## 5. Update Google Sheet / If everything works: / Status → Posted / If anything fails: / Status → Errored / This makes it easy to track what went live and what needs fixing. |
| Sticky Note7 | Sticky Note | Workspace documentation block | — | — | ## Auto Cross-Posting with AI Images / … Need help? Ask in the [n8n Forum](https://community.n8n.io/) or shoot me a DM on [LinkedIn](https://www.linkedin.com/in/vincentthenguyen/) |
| Sticky Note5 | Sticky Note | Block label: Trigger + Get Content | — | — | (content already reflected above) |
| Sticky Note3 | Sticky Note | Block label: X + Threads | — | — | (content already reflected above) |
| Sticky Note1 | Sticky Note | Block label: AI Image Generation | — | — | (content already reflected above) |
| Sticky Note6 | Sticky Note | Block label: Publish Image Posts | — | — | (content already reflected above) |
| Sticky Note4 | Sticky Note | Block label: Update Google Sheet | — | — | (content already reflected above) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add “Schedule Trigger”**
   - Node: *Schedule Trigger*
   - Configure three daily times: **07:03**, **12:01**, **19:02** (set timezone appropriately).
   - Connect to **Get Content**.

3. **Add “Get Content” (Google Sheets)**
   - Node: *Google Sheets* → operation to **Read / Get Many** (as per node UI), configured to return one matching row.
   - Select your Spreadsheet and Sheet tab.
   - Add filter: `Status` equals `Ready To Post`.
   - Enable option **Return First Match**.
   - Credential: Google Sheets OAuth2 (authorized to the spreadsheet).
   - Ensure sheet columns exist: **Title**, **Content**, **Status**.

4. **Add X posting**
   - Node: *X (Twitter)*.
   - Text field: `{{ $json.Content }}`.
   - Credential: X OAuth2.
   - Connect **Get Content → X**.

5. **Add Threads posting (2-step)**
   - Node: *HTTP Request* named **Create Threads**
     - Method: POST
     - URL: `https://graph.threads.net/v1.0/<THREADS_USER_ID>/threads`
     - Send query parameters:
       - `text` = `{{ $json.Content }}`
       - `media_type` = `TEXT`
     - Auth: Header auth (Bearer token), store in an **HTTP Header Auth** credential.
   - Node: *HTTP Request* named **Threads**
     - Method: POST
     - URL: `https://graph.threads.net/v1.0/<THREADS_USER_ID>/threads_publish`
     - Query parameter:
       - `creation_id` = `{{ $json.id }}`
     - Same header auth credential.
   - Connect **Get Content → Create Threads → Threads**.

6. **Add LLM provider (OpenRouter)**
   - Node: *OpenRouter Chat Model* named **OpenRouter Gemini 2.5 Flash**
   - Model: `google/gemini-2.5-flash`
   - Credential: OpenRouter API key.

7. **Add “Image Prompt Agent”**
   - Node: *AI Agent* (LangChain Agent) named **Image Prompt Agent**
   - Text input: `Social Media Post: {{ $json.Content }}`
   - System message: paste the provided system message (marketing graphic prompt rules).
   - Connect the model:
     - Connect **OpenRouter Gemini 2.5 Flash → Image Prompt Agent** via the **ai_languageModel** connection.
   - Connect **Get Content → Image Prompt Agent**.

8. **Add OpenAI image generation**
   - Node: *OpenAI* (LangChain OpenAI node) named **Generate an image**
   - Resource: **image**
   - Model: `gpt-image-1`
   - Prompt: `{{ $json.output }}`
   - Credential: OpenAI API key (with billing enabled).
   - Connect **Image Prompt Agent → Generate an image**.

9. **Add ImgBB upload**
   - Node: *HTTP Request* named **ImgBB**
   - Method: POST
   - URL: `https://api.imgbb.com/1/upload`
   - Content-Type: `multipart/form-data`
   - Body (multipart):
     - `image` from binary property **data**
   - Auth: Query auth (ImgBB key), store in **HTTP Query Auth** credential.
   - Connect **Generate an image → ImgBB**.

10. **Add Facebook Page photo post**
   - Node: *Facebook Graph API* named **Facebook**
   - Node/Page ID: your Page ID
   - Edge: `photos`
   - Method: POST
   - Enable **Send Binary Data**
   - Binary property: `data`
   - Query param: `message = {{ $('Get Content').item.json.Content }}`
   - Credential: Facebook Graph API (Page access).
   - Connect **Generate an image → Facebook**.

11. **Add LinkedIn image post**
   - Node: *LinkedIn* named **LinkedIn**
   - Person ID/URN: your LinkedIn identifier
   - Share media category: `IMAGE`
   - Text: `{{ $('Get Content').item.json.Content }}`
   - Credential: LinkedIn OAuth2.
   - Connect **Generate an image → LinkedIn**.

12. **Add Instagram (2-step publish via Graph)**
   - Node: *Facebook Graph API* named **Instagram Image**
     - Node (IG user id): your Instagram Business/Creator IG user ID
     - Edge: `media`
     - Method: POST
     - Query:
       - `image_url = {{ $json.data.url }}` (from ImgBB response)
       - `caption = {{ $('Get Content').item.json.Content }}`
   - Node: *Facebook Graph API* named **Instagram**
     - Node: same IG user ID
     - Edge: `media_publish`
     - Method: POST
     - Query:
       - `creation_id = {{ $json.id }}`
   - Connect **ImgBB → Instagram Image → Instagram**.

13. **Add Google Sheets update nodes**
   - Node: *Google Sheets* named **Done**
     - Operation: Update
     - Match on column: `Title`
     - Title value: `{{ $('Get Content').item.json.Title }}`
     - Set `Status = Posted`
   - Node: *Google Sheets* named **Error**
     - Same match
     - Set `Status = Errored`
   - **Important (recommended wiring):**
     - As provided, both are connected directly from Get Content (not truly success/failure).
     - To reproduce the JSON exactly, connect **Get Content → Done** and **Get Content → Error**.
     - To make it work as intended, implement proper success/error routing (see notes in Section 2 Block 5).

14. **Credentials checklist**
   - Google Sheets OAuth2: access to the spreadsheet
   - OpenRouter API key: access to Gemini model
   - OpenAI API key: image generation
   - ImgBB API key (query auth)
   - X OAuth2
   - Facebook Graph: Page + IG permissions and tokens
   - LinkedIn OAuth2
   - Threads token in HTTP Header Auth + replace `<THREADS_USER_ID>` in both URLs

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Ask in the n8n Forum | https://community.n8n.io/ |
| DM on LinkedIn (workflow author reference in sticky note) | https://www.linkedin.com/in/vincentthenguyen/ |
| The workflow description emphasizes “no duplicate posts” via sheet Status changes, but the current graph wiring updates Posted/Errored immediately after fetching content, not after publish results. | Applies to the intended design vs current implementation |
| Matching Google Sheet rows by `Title` can be unsafe if titles are duplicated; using a unique ID or row number is more reliable. | Applies to Done/Error update strategy |