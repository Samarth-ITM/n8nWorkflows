Create and schedule LinkedIn posts from Google Sheets with Gemini and DALL·E

https://n8nworkflows.xyz/workflows/create-and-schedule-linkedin-posts-from-google-sheets-with-gemini-and-dall-e-12663


# Create and schedule LinkedIn posts from Google Sheets with Gemini and DALL·E

## 1. Workflow Overview

This n8n workflow implements a “LinkedIn Content Engine” backed by Google Sheets and AI. It has two independent entry points (two workflows in one canvas):

### 1.1 Creator (Draft Generator)
Triggered when a new row (topic idea) is added to a Google Sheet. It validates the row, processes items one-by-one, retrieves brand voice guidelines, generates a LinkedIn post via Gemini (OpenRouter), generates a matching image via DALL·E, uploads the image to ImgBB, then writes the generated content + image URL back into the sheet with **Status = Draft**.

### 1.2 Publisher (Scheduled Poster)
Triggered daily by a schedule. It reads the same sheet, filters rows where **Status = Approved** and **Date Scheduled = today**, then posts to LinkedIn via an “Upload Post” integration node (or an HTTP alternative) and updates the row to **Status = Posted**. If criteria don’t match, it does nothing.

Logical blocks (by direct dependencies):
- Block A: Topic intake + validation + batching (Creator)
- Block B: Brand voice retrieval (Creator)
- Block C: AI writing chain (Creator)
- Block D: Image generation + hosting + save draft (Creator)
- Block E: Daily scan + eligibility check (Publisher)
- Block F: Publish + mark posted (Publisher)

---

## 2. Block-by-Block Analysis

### Block A — Topic intake, filtering, and one-at-a-time processing (Creator)
**Overview:** Detects new topic rows in the content calendar sheet, filters only rows that should be processed, and enforces sequential processing to avoid rate limits and duplicate writes.

**Nodes involved:** `GetTopic`, `Filter`, `Loop Over Items`

#### Node: GetTopic
- **Type / role:** Google Sheets Trigger (`googleSheetsTrigger`) — entry point for Creator.
- **Configuration (interpreted):**
  - Event: **Row Added**
  - Poll time: configured at **08:00** (polling trigger; not instant push)
  - Spreadsheet: “Content Calendar” tab (gid `1652769038`) via URL-based selection.
- **Key data produced:** a row JSON containing at least `Topic`, `Status`, `id` (and other columns if present).
- **Connections:**
  - Output → `Filter`
- **Edge cases / failures:**
  - Trigger may re-read or miss rows depending on polling and sheet changes.
  - Missing expected columns (e.g., `Status`, `Topic`, `id`) leads to downstream expression failures.
  - Google Sheets auth / permission issues.

#### Node: Filter
- **Type / role:** Filter node — gatekeeper.
- **Configuration (interpreted):**
  - Condition 1: `GetTopic.Topic` **is not empty**
  - Condition 2: current row `Status` **equals** `"Not started"`
- **Key expressions:**
  - `{{ $('GetTopic').item.json.Topic }}`
  - `{{ $json.Status }}`
- **Connections:**
  - Output (pass) → `Loop Over Items`
  - Output (fail) → (not connected; items are dropped)
- **Edge cases / failures:**
  - If sheet uses different status labels (case/spacing), nothing processes.
  - If `Status` is blank, it fails the equals check and is dropped.

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`splitInBatches`) — sequential processing loop.
- **Configuration (interpreted):**
  - Uses default batch settings (no explicit batch size shown).
  - The workflow is wired so that after saving a draft it loops back to fetch the next batch.
- **Connections:**
  - Output 1 (current batch) → *(not used)*
  - Output 2 (“No items left” path in this workflow’s wiring) → `GetBrandGuide`
  - From `Append or update row in sheet` → back to `Loop Over Items` (forms the iteration loop)
- **Important note (wiring nuance):**
  - In many n8n designs, the “items” output is used for processing and the second output is used to end. Here, processing is effectively controlled by the loop-back from the final write node; ensure batch behavior matches your expectations when reproducing.
- **Edge cases / failures:**
  - Misconfigured batch size can cause rate limiting or long runs.
  - If the loop-back is removed, only the first batch may process.

---

### Block B — Brand voice retrieval (Creator)
**Overview:** Pulls brand guidelines from a separate tab so the writing model stays consistent with your voice.

**Nodes involved:** `GetBrandGuide`

#### Node: GetBrandGuide
- **Type / role:** Google Sheets node (`googleSheets`) — read brand guidelines.
- **Configuration (interpreted):**
  - Reads from the same spreadsheet but different tab (gid `0`, typically the first sheet), presumably containing a column like `Guidelines`.
  - Operation is not explicitly shown in parameters; with default settings, this commonly means “Read” / “Get Many”. (When recreating, choose a read operation that returns the row(s) containing `Guidelines`.)
- **Connections:**
  - Output → `Basic LLM Chain`
- **Key dependency:** `Basic LLM Chain` expects `{{ $json.Guidelines }}` to exist in the incoming item.
- **Edge cases / failures:**
  - If the guidelines sheet returns multiple rows, you may unintentionally generate multiple posts per topic unless you pick a single row.
  - If the guidelines column is named differently, the prompt expression breaks.

---

### Block C — AI writing chain (Creator)
**Overview:** Uses a LangChain LLM chain node connected to OpenRouter (Gemini Flash 1.5) to generate LinkedIn post text in a strict format, returning structured JSON (`{ "content": "..." }`) via an output parser, with a fallback model configured.

**Nodes involved:** `OpenRouter Chat Model`, `Fallback model`, `Structured Output Parser`, `Basic LLM Chain`

#### Node: OpenRouter Chat Model
- **Type / role:** LangChain Chat Model (`lmChatOpenRouter`) — primary LLM provider.
- **Configuration (interpreted):**
  - Model: `google/gemini-flash-1.5`
  - Requires OpenRouter credentials (API key) in n8n.
- **Connections:**
  - AI languageModel output → `Basic LLM Chain` (input index 0)
- **Edge cases / failures:**
  - OpenRouter rate limits, model availability changes, or invalid key.
  - Output style drift if the model ignores constraints (mitigated by parser + strict prompt).

#### Node: Fallback model
- **Type / role:** LangChain Chat Model (`lmChatOpenRouter`) — fallback LLM.
- **Configuration (interpreted):**
  - Model not specified in the JSON snippet (defaults or UI selection required). When recreating, choose a reliable fallback model (e.g., another Gemini variant or OpenAI via OpenRouter).
- **Connections:**
  - AI languageModel output → `Basic LLM Chain` (fallback / secondary index 1)
- **Edge cases / failures:**
  - If no model is selected, fallback may be ineffective.
  - Ensure fallback credentials match OpenRouter.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured parser (`outputParserStructured`) — forces JSON output.
- **Configuration (interpreted):**
  - Schema example:
    ```json
    { "content": "linkedin post" }
    ```
  - This nudges the chain to produce parseable JSON with a `content` field.
- **Connections:**
  - AI outputParser → `Basic LLM Chain`
- **Edge cases / failures:**
  - If model returns non-JSON or invalid JSON, parsing fails and the chain errors.
  - If the model returns `content` under a different key, downstream expressions break.

#### Node: Basic LLM Chain
- **Type / role:** LangChain LLM Chain (`chainLlm`) — prompt + parsing + fallback orchestration.
- **Configuration (interpreted):**
  - Prompt is “define” mode with a long instruction set:
    - Applies brand guidelines: `{{ $json.Guidelines }}`
    - Injects topic: `{{ $('GetTopic').item.json.Topic }}`
    - Enforces: third-grade reading level, hook, short sentences, line breaks, no hashtags inside content, etc.
  - **Needs fallback:** enabled.
  - **Has output parser:** enabled (Structured Output Parser).
- **Connections:**
  - Main output → `Generate an image`
  - Receives AI model connections from `OpenRouter Chat Model` and `Fallback model`, and parser from `Structured Output Parser`.
- **Edge cases / failures:**
  - If `GetTopic` isn’t in scope (e.g., changed wiring), `$('GetTopic').item...` can fail.
  - Character limit compliance is best-effort; you may still want a post-length validation step.

---

### Block D — Image generation, image hosting, and draft persistence (Creator)
**Overview:** Generates an image prompt from the produced post, creates an image using OpenAI image generation (DALL·E), uploads it to ImgBB, and writes the draft content + image URL back to the Content Calendar sheet.

**Nodes involved:** `Generate an image`, `UploadImg`, `Append or update row in sheet`

#### Node: Generate an image
- **Type / role:** OpenAI node via LangChain (`openAi`) — image generation.
- **Configuration (interpreted):**
  - Resource: **image**
  - Prompt:
    - “Generate a linkedin post image in black and white, professional, for the following post: {{ $json.output.content }}”
  - Requires OpenAI (or compatible) credentials configured in n8n for this node.
- **Connections:**
  - Output → `UploadImg`
- **Edge cases / failures:**
  - Image generation may return different payload structures depending on node version/provider.
  - Cost/rate limits.
  - Safety policy blocks for certain prompts/topics.

#### Node: UploadImg
- **Type / role:** HTTP Request — upload binary image to ImgBB.
- **Configuration (interpreted):**
  - POST `https://api.imgbb.com/1/upload`
  - Multipart form-data
  - Body field: `image` from binary input field named `data`
  - Auth: “httpQueryAuth” via generic credentials (typically `key=<IMGBB_API_KEY>` in query parameters).
  - Sets `Content-type: multipart/form-data` header explicitly.
- **Connections:**
  - Output → `Append or update row in sheet`
- **Key dependency:** Expects image binary to be present in `binary.data`.
- **Edge cases / failures:**
  - If the image node outputs base64 instead of binary `data`, upload fails.
  - ImgBB API key invalid/expired; 400/401 errors.
  - Large images can exceed limits.

#### Node: Append or update row in sheet
- **Type / role:** Google Sheets — persist generated draft (upsert by id).
- **Configuration (interpreted):**
  - Operation: **Append or Update**
  - Match column: `id`
  - Writes:
    - `id` = `{{ $('GetTopic').item.json.id }}`
    - `Image` = `{{ $json.data.url }}` (ImgBB hosted URL)
    - `Status` = `"Draft"`
    - `Content` = `{{ $('Basic LLM Chain').item.json.output.content }}`
    - `Date Created` = `{{ $now.format('yyyy-MM-dd') }}`
  - Sheet: Content Calendar (gid `1652769038`)
- **Connections:**
  - Output → `Loop Over Items` (loop continuation)
- **Edge cases / failures:**
  - If `id` is missing or not unique, you may overwrite the wrong row or append duplicates.
  - If `data.url` is not present (ImgBB error response), expression fails.
  - Date formats may not match your sheet locale.

---

### Block E — Daily scan and eligibility check (Publisher)
**Overview:** Every morning, loads posts from the sheet and checks which ones are both approved and scheduled for today.

**Nodes involved:** `Schedule Trigger`, `GetPosts`, `IfApprovedandToday`, `No Operation, do nothing`

#### Node: Schedule Trigger
- **Type / role:** Schedule (`scheduleTrigger`) — entry point for Publisher.
- **Configuration (interpreted):**
  - Runs daily at **09:00** (triggerAtHour: 9).
- **Connections:**
  - Output → `GetPosts`
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings.
  - Missed runs if instance is offline at trigger time.

#### Node: GetPosts
- **Type / role:** Google Sheets — fetch content calendar rows for evaluation.
- **Configuration (interpreted):**
  - Reads from Content Calendar (gid `1652769038`).
  - Operation not explicit; recreate as “Read / Get Many” returning all rows (or a filtered subset).
- **Connections:**
  - Output → `IfApprovedandToday`
- **Edge cases / failures:**
  - Large sheets can cause slow runs; consider server-side filtering (if available) or limit rows.

#### Node: IfApprovedandToday
- **Type / role:** IF node — branch decision.
- **Configuration (interpreted):**
  - Condition A: `Status` equals `"Approved"`
  - Condition B: `Date Scheduled` equals `{{ $now.format('dd/MM/yyyy') }}`
- **Connections:**
  - True → `Upload Post`
  - False → `No Operation, do nothing`
- **Edge cases / failures:**
  - Date format mismatch is common (sheet may store as `yyyy-MM-dd` or as a real date value).
  - If Google Sheets returns a Date object/serial formatted differently, string compare fails.
  - Status values must match exactly.

#### Node: No Operation, do nothing
- **Type / role:** NoOp — explicit end for non-eligible rows.
- **Configuration:** none.
- **Connections:** none.
- **Edge cases:** none (acts as sink).

---

### Block F — Publish to LinkedIn and mark as posted (Publisher)
**Overview:** Posts the selected content and image to LinkedIn using the Upload Post integration, then updates the row status to “Posted”.

**Nodes involved:** `Upload Post`, `Upload Post - Upload Photo` (alternative), `Append or update row in sheet1`

#### Node: Upload Post
- **Type / role:** Community/preview node (`n8n-nodes-preview-upload-post.uploadPost`) — publishes to LinkedIn via Upload Post service.
- **Configuration (interpreted):**
  - Resource: `uploads`
  - Operation: `uploadPhotos`
  - Credentials: not included in JSON export (`credentials`: `{}`), must be set in your environment.
- **Connections:**
  - Output → `Append or update row in sheet1`
- **Edge cases / failures:**
  - Missing credentials is the most common issue.
  - Service/API outages or LinkedIn restrictions.
  - Payload expectations: typically needs post text + image URL; verify the node’s required fields in UI.

#### Node: Upload Post - Upload Photo (alternative)
- **Type / role:** HTTP Request — manual alternative to the community node (not connected in current wiring).
- **Configuration (interpreted):**
  - POST `https://api.upload-post.com/api/upload_photos`
  - Form URL-encoded body:
    - `user = simon`
    - `platform[] = linkedin`
    - `photos[] = {{ $json.Image }}`
    - `title = {{ $json.Content }}`
    - `async_upload = true`
    - `visibility = PUBLIC`
  - Auth: predefined credential type `uploadPostApi`
- **Connections:** none (provided as fallback replacement path).
- **Edge cases / failures:**
  - Requires correct Upload-Post API credentials and expected field names.
  - If `Image` is not a publicly accessible URL, upload may fail.
  - If `Content` exceeds platform limits, API may reject.

#### Node: Append or update row in sheet1
- **Type / role:** Google Sheets — mark row as posted (upsert).
- **Configuration (interpreted):**
  - Operation: **Append or Update**
  - Match column: `id`
  - Writes:
    - `id` = `{{ $('GetPosts').item.json.id }}`
    - `Status` = `"Posted"`
  - Sheet: Content Calendar (gid `1652769038`)
- **Connections:** none (end).
- **Edge cases / failures:**
  - If `id` missing, may append new rows instead of updating.
  - If multiple rows share same id, wrong row updated.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| GetTopic | Google Sheets Trigger | Creator entry point: detect new topic rows | — | Filter | ## Topic input<br>GSheet |
| Filter | Filter | Validate Topic and Status before generating | GetTopic | Loop Over Items | ## Filters and process one at a time |
| Loop Over Items | Split In Batches | Sequential processing loop | Filter; Append or update row in sheet | GetBrandGuide | ## Filters and process one at a time |
| GetBrandGuide | Google Sheets | Load brand voice guidelines | Loop Over Items | Basic LLM Chain | ## Brand input<br>GSheet |
| OpenRouter Chat Model | OpenRouter Chat Model (LangChain) | Primary LLM (Gemini Flash 1.5) | — (AI connection) | Basic LLM Chain (AI) | ## AI stage<br>LLM chain that writes our content |
| Fallback model | OpenRouter Chat Model (LangChain) | Fallback LLM for chain | — (AI connection) | Basic LLM Chain (AI fallback) | ## AI stage<br>LLM chain that writes our content |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON `{content}` output | — (AI connection) | Basic LLM Chain (AI) | ## AI stage<br>LLM chain that writes our content |
| Basic LLM Chain | LLM Chain (LangChain) | Generate LinkedIn post text using brand guidelines | GetBrandGuide (+ AI model + parser) | Generate an image | ## AI stage<br>LLM chain that writes our content |
| Generate an image | OpenAI (LangChain) | Create an image from post content | Basic LLM Chain | UploadImg | ## Image generation<br>Generate an image that matches the content |
| UploadImg | HTTP Request | Upload generated image to ImgBB, get public URL | Generate an image | Append or update row in sheet | ## Image generation<br>Generate an image that matches the content |
| Append or update row in sheet | Google Sheets | Save draft content + image URL, set Draft | UploadImg | Loop Over Items | ## Topic output<br>GSheet |
| Schedule Trigger | Schedule Trigger | Publisher entry point: daily run | — | GetPosts | ## Time trigger<br>GSheet |
| GetPosts | Google Sheets | Load all candidate posts from calendar | Schedule Trigger | IfApprovedandToday | ## Get post data<br>GSheet |
| IfApprovedandToday | IF | Check scheduled date == today and status == Approved | GetPosts | Upload Post (true); No Operation, do nothing (false) | ## Is it scheduled today and is it approved? |
| Upload Post | Upload Post (community/preview) | Publish post to LinkedIn | IfApprovedandToday (true) | Append or update row in sheet1 | ## If yes - post to Linkedin<br>## Having trouble posting the content?<br>If you have any trouble with the Upload Post community node, just replace it for this HTTP Request with your own API credentials |
| Append or update row in sheet1 | Google Sheets | Mark published row as Posted | Upload Post | — | ## If yes - post to Linkedin |
| No Operation, do nothing | NoOp | Sink node for non-eligible rows | IfApprovedandToday (false) | — | ## If no - no further action |
| Upload Post - Upload Photo | HTTP Request | Alternative method to publish via Upload-Post API (not wired) | — | — | ## Having trouble posting the content?<br>If you have any trouble with the Upload Post community node, just replace it for this HTTP Request with your own API credentials |
| Sticky Note | Sticky Note | Comment | — | — | **Workflow 1** - Generator (8am): Pull brand voice → Get today's topics → AI generates posts → Create images → Save for review<br>- Google Sheets setup: Create content calendar with: topic, date, status, generated_post, edited_post, approved checkbox. Create brand voice guidelines.<br>- Manual review step: Edit posts in sheet between workflows, check "approved" when ready<br><br>**Workflow 2** - Publisher (9am): Get approved posts → Post to LinkedIn → Update status to "posted"<br>- Key connections: Google Sheets for storage, AI for writing, Hugging Face for images, Upload-Post for LinkedIn<br>- Testing: Run with one test post through entire flow - generate, review, edit, approve, publish |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Topic input<br>GSheet |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Topic output<br>GSheet |
| Sticky Note3 | Sticky Note | Comment | — | — | ## AI stage<br>LLM chain that writes our content |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Image generation<br>Generate an image that matches the content |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Brand input<br>GSheet |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Time trigger<br>GSheet |
| Sticky Note7 | Sticky Note | Comment | — | — | ## Is it scheduled today and is it approved? |
| Sticky Note8 | Sticky Note | Comment | — | — | ## If no - no further action |
| Sticky Note9 | Sticky Note | Comment | — | — | ## If yes - post to Linkedin |
| Sticky Note10 | Sticky Note | Comment | — | — | ## Get post data<br>GSheet |
| Sticky Note11 | Sticky Note | Comment | — | — | ## Filters and process one at a time |
| Sticky Note12 | Sticky Note | Comment | — | — | ## Having trouble posting the content?<br>If you have any trouble with the Upload Post community node, just replace it for this HTTP Request with your own API credentials |
| Sticky Note13 | Sticky Note | Comment | — | — | ## Welcome! This template is a production-ready system that generates a week's worth of content in minutes (in your brand voice)<br><br>The template gives you full editorial control, and posts automatically - turning 5 hours of work into 15 minutes of review.<br><br>You can follow the step-by-step build and setup here:<br><br>Start at [1hr 11mins for step-by-step build](https://youtu.be/eiIRSUhPgOI?si=GUdIjMxA11o4jfm5&t=4275)<br>@[youtube](eiIRSUhPgOI)<br><br>### Check out my YouTube [@simonscrapes](https://www.youtube.com/@simonscrapes) for more tutorials<br>### Master AI & Automation in our [main community](https://skool.com/scrapes/about) |

---

## 4. Reproducing the Workflow from Scratch

### A) Google Sheets preparation (required structure)
1. Create a spreadsheet with:
   - **Content Calendar** sheet (tab) containing columns at least:  
     `id`, `Topic`, `Date Scheduled`, `Date Created`, `Content`, `Image`, `Status`
   - A **Brand Voice** sheet/tab containing a `Guidelines` column (ideally one row with your current brand guidelines).
2. Ensure each Content Calendar row has a unique `id` (you can use a formula, manual UUID, or an n8n-generated id).

### B) Creator workflow (Draft Generator)
1. Add node: **Google Sheets Trigger** named `GetTopic`
   - Event: **Row Added**
   - Select the Content Calendar sheet
   - Set polling time to **08:00**
   - Configure Google Sheets OAuth2 credentials.
2. Add node: **Filter** named `Filter`
   - Conditions:
     - `Topic` is not empty (use expression from trigger item)
     - `Status` equals `Not started`
   - Connect: `GetTopic` → `Filter`
3. Add node: **Split In Batches** named `Loop Over Items`
   - Keep default options (or set batch size = 1 explicitly to be safe)
   - Connect: `Filter` → `Loop Over Items`
4. Add node: **Google Sheets** named `GetBrandGuide`
   - Operation: read your Brand Voice tab (return the row with `Guidelines`)
   - Connect: `Loop Over Items` → `GetBrandGuide`
5. Add nodes for LangChain:
   - **OpenRouter Chat Model** named `OpenRouter Chat Model`
     - Model: `google/gemini-flash-1.5`
     - Configure OpenRouter API credential (API key).
   - **OpenRouter Chat Model** named `Fallback model`
     - Pick a fallback model in OpenRouter (any stable alternative).
   - **Structured Output Parser** named `Structured Output Parser`
     - Schema example with `content` string.
6. Add node: **Basic LLM Chain** (`chainLlm`) named `Basic LLM Chain`
   - Prompt: paste your full constraints prompt
   - Include expressions:
     - Brand guidelines: `{{ $json.Guidelines }}` (from `GetBrandGuide`)
     - Topic: `{{ $('GetTopic').item.json.Topic }}`
   - Enable fallback and structured output parsing.
   - Connect AI ports:
     - `OpenRouter Chat Model` → `Basic LLM Chain` (AI languageModel)
     - `Fallback model` → `Basic LLM Chain` (AI languageModel fallback)
     - `Structured Output Parser` → `Basic LLM Chain` (AI outputParser)
   - Connect main flow: `GetBrandGuide` → `Basic LLM Chain`
7. Add node: **OpenAI (Image)** via LangChain named `Generate an image`
   - Resource: Image generation (DALL·E)
   - Prompt referencing the chain output: `{{ $json.output.content }}`
   - Configure OpenAI credentials.
   - Connect: `Basic LLM Chain` → `Generate an image`
8. Add node: **HTTP Request** named `UploadImg` (ImgBB)
   - Method: POST
   - URL: `https://api.imgbb.com/1/upload`
   - Authentication: query auth with parameter `key=<IMGBB_API_KEY>` (store in credential)
   - Send Body: multipart-form-data
   - Body: `image` from binary field `data`
   - Connect: `Generate an image` → `UploadImg`
9. Add node: **Google Sheets** named `Append or update row in sheet`
   - Operation: **Append or Update**
   - Matching column: `id`
   - Values to set:
     - `id` = `{{ $('GetTopic').item.json.id }}`
     - `Content` = `{{ $('Basic LLM Chain').item.json.output.content }}`
     - `Image` = `{{ $json.data.url }}`
     - `Status` = `Draft`
     - `Date Created` = `{{ $now.format('yyyy-MM-dd') }}`
   - Connect: `UploadImg` → `Append or update row in sheet`
10. Close the loop:
   - Connect: `Append or update row in sheet` → `Loop Over Items` (to continue batches)

### C) Publisher workflow (Scheduled Poster)
1. Add node: **Schedule Trigger** named `Schedule Trigger`
   - Daily at **09:00** (confirm timezone in n8n settings).
2. Add node: **Google Sheets** named `GetPosts`
   - Read rows from Content Calendar.
   - Connect: `Schedule Trigger` → `GetPosts`
3. Add node: **IF** named `IfApprovedandToday`
   - Conditions:
     - `Status` equals `Approved`
     - `Date Scheduled` equals `{{ $now.format('dd/MM/yyyy') }}`
   - Connect: `GetPosts` → `IfApprovedandToday`
4. Add node: **Upload Post** named `Upload Post`
   - Use the Upload-Post integration node (community/preview) *or* replace with HTTP Request to Upload-Post API.
   - Configure credentials for Upload-Post (API key/token as required by the node).
   - Connect: `IfApprovedandToday` (true) → `Upload Post`
5. Add node: **Google Sheets** named `Append or update row in sheet1`
   - Operation: **Append or Update**
   - Matching column: `id`
   - Set `Status` = `Posted`
   - `id` = `{{ $('GetPosts').item.json.id }}`
   - Connect: `Upload Post` → `Append or update row in sheet1`
6. Add node: **NoOp** named `No Operation, do nothing`
   - Connect: `IfApprovedandToday` (false) → `No Operation, do nothing`

**Credential checklist**
- Google Sheets OAuth2 (read/write + trigger)
- OpenRouter API key (Gemini model)
- OpenAI API key (image generation)
- ImgBB API key (query auth)
- Upload-Post credentials (for community node) or API key for HTTP alternative

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| YouTube Channel: Simon Scrapes | https://www.youtube.com/@simonscrapes |
| Community: Skool “Scrapes” | https://www.skool.com/scrapes/about |
| Full build video link (timestamp) | https://youtu.be/eiIRSUhPgOI?si=lgZTrPZPMqWF4uqz&t=4276 |
| Step-by-step build starts around 1hr 11mins | https://youtu.be/eiIRSUhPgOI?si=GUdIjMxA11o4jfm5&t=4275 |
| Note in workflow mentions: “Having trouble posting the content? Replace Upload Post node with HTTP Request using your own API credentials.” | Applies to Publisher publishing step (Upload-Post integration). |
| The canvas note references “Hugging Face for images”, but the actual implementation uses OpenAI image generation + ImgBB hosting. | Ensure docs/config reflect the real nodes used. |