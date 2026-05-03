Create and post social media image captions with GPT-4.1-mini and Google Sheets

https://n8nworkflows.xyz/workflows/create-and-post-social-media-image-captions-with-gpt-4-1-mini-and-google-sheets-12596


# Create and post social media image captions with GPT-4.1-mini and Google Sheets

## 1. Workflow Overview

**Title:** Create and post social media image captions with GPT-4.1-mini and Google Sheets

**Purpose & Use Cases**  
This workflow automates (1) **AI caption generation** for an image-based social post and (2) **publishing approved posts** to selected social platforms using a Google Sheet as the review/approval queue. Typical use: a team generates captions with GPT-4.1-mini, reviews them in Google Sheets, sets platform flags (IG/FB/X/LinkedIn), then the workflow posts automatically and marks the row as posted.

### 1.1 Caption Creation & Sheet Ingestion (Manual)
- Manually run the workflow.
- Set a topic + image URL.
- GPT generates a caption.
- Append the new post draft to Google Sheets for review.

### 1.2 Scheduled Approval Queue Processing (Schedule Trigger)
- Runs on a schedule.
- Reads rows from Google Sheets filtered by **STATUS = "post"** (intended as “approved for posting”).
- Loops through items and routes each row to the correct platform posting branch.

### 1.3 Platform Posting (IG / X / FB / LinkedIn)
- Downloads the image (binary) when needed.
- Posts to selected platform(s) depending on sheet flags and “POST TYPE”.
- Updates Google Sheets row status to **posted**.

---

## 2. Block-by-Block Analysis

### Block A — Caption generation and draft storage (Manual path)

**Overview:**  
Creates a post draft from a manually provided topic and image URL, generates a caption with GPT-4.1-mini, then writes the resulting caption + image URL into Google Sheets.

**Nodes Involved:**  
- When clicking ‘Execute workflow’
- Post Topic
- Generate Caption
- Append row in sheet

#### Node: **When clicking ‘Execute workflow’**
- **Type / Role:** Manual Trigger (`manualTrigger`) — entry point for caption generation.
- **Configuration:** No parameters.
- **Outputs:** Sends one empty item downstream.
- **Failure modes:** None (except workflow execution permissions).

#### Node: **Post Topic**
- **Type / Role:** Set (`set`) — defines the input topic and asset URL for the AI model.
- **Key fields created:**
  - `title`: `"Ai Automation"`
  - `image_url`: Cloudinary URL
  - `phone`: `"7745930380"` (not used elsewhere in this workflow)
- **Connections:**
  - Input: Manual Trigger
  - Output: Generate Caption
- **Edge cases:**
  - Missing/invalid `image_url` will not break caption generation (caption doesn’t use it), but will affect later posting if you reuse it.
  - `phone` is unused—could be removed or integrated into prompt/CTA if desired.

#### Node: **Generate Caption**
- **Type / Role:** OpenAI via LangChain (`@n8n/n8n-nodes-langchain.openAi`) — generates a single social caption.
- **Model:** `gpt-4.1-mini`
- **Prompt design:**
  - System message defines goal, tone, structure, constraints (under 1000 chars, 4–6 emojis, 3–5 hashtags incl. `#n8n` and `#AI`, line breaks).
  - User message uses expression: `Topic for Today's Post: {{ $json.title }}`
- **Credentials:** OpenAI API credential (“OpenAi Chaaabi”).
- **Connections:**
  - Input: Post Topic
  - Output: Append row in sheet
- **Failure modes / edge cases:**
  - OpenAI auth errors (bad key), rate limits, model availability changes.
  - Output format: this node typically returns `message.content`; if the node response format changes, the sheet append mapping may fail.

#### Node: **Append row in sheet**
- **Type / Role:** Google Sheets (`googleSheets`, operation: append) — stores generated caption as a draft row.
- **Document:** “SocialPulse” spreadsheet (ID `1xCdabz...`), sheet “Video Content data” (gid=0).
- **Mapped columns:**
  - `URL` = `{{ $('Post Topic').item.json.image_url }}`
  - `POST TYPE` = `"IMAGE"`
  - `Description` = `{{ $json.message.content }}`
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account Neer_Sam”).
- **Connections:**
  - Input: Generate Caption
  - Output: (none)
- **Failure modes / edge cases:**
  - Sheet permissions / OAuth expiration.
  - Column headers must match exactly: `URL`, `POST TYPE`, `Description`.
  - STATUS/platform flags are not set here; they must be filled manually later in the sheet.

---

### Block B — Scheduled queue intake and item iteration

**Overview:**  
Polls the Google Sheet on a schedule, pulls rows with STATUS matching “post”, then iterates through each row for conditional routing to platform branches.

**Nodes Involved:**  
- Schedule social media posts
- get sheet data
- Loop Over Items
- Route

#### Node: **Schedule social media posts**
- **Type / Role:** Schedule Trigger (`scheduleTrigger`) — timed entry point for publishing.
- **Schedule config:** Interval on `seconds` (no value shown in JSON; must be configured in UI).
- **Connections:**
  - Output: get sheet data
- **Failure modes:**
  - Misconfigured schedule interval can cause overposting or rate-limit issues.

#### Node: **get sheet data**
- **Type / Role:** Google Sheets (`googleSheets`) — reads rows to post.
- **Filter:** `STATUS` equals **"post"**.
- **Document/Sheet:** same spreadsheet and gid=0.
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account Neer_Sam”).
- **Connections:**
  - Input: Schedule trigger
  - Output: Loop Over Items
- **Edge cases:**
  - If STATUS values differ (e.g., “Approved” vs “post”), nothing will be returned.
  - If boolean flags in the sheet are strings (“TRUE”/“FALSE”) rather than booleans, routing conditions may not match.

#### Node: **Loop Over Items**
- **Type / Role:** Split In Batches (`splitInBatches`) — iterates through sheet rows.
- **Behavior:** Outputs items batch-by-batch. In this workflow:
  - **Output 0** is unused (goes nowhere).
  - **Output 1** is connected to Route (this is unusual; typically output 0 is used for processing).
- **Connections:**
  - Input: get sheet data
  - Output(1): Route
- **Edge cases / likely issues:**
  - If output index is not intended, the workflow may not process items as expected.
  - Standard pattern is: Output 0 → processing → back into SplitInBatches to continue; here there is no “continue” loop, so it may only emit once depending on n8n behavior/config.

#### Node: **Route**
- **Type / Role:** Switch (`switch`) — routes each row to one or more platform branches.
- **Option:** `allMatchingOutputs: true` (can post to multiple platforms for the same row).
- **Rules (interpreted):**
  1. **INSTAGRAM_IMAGE:** if `POST TYPE == "IMAGE"` AND `IG == true`
  2. **X_IMAGE:** if `POST TYPE == "IMAGE"` AND `X == true`
  3. **FACEBOOK_IMAGE:** if `POST TYPE == "IMAGE"` AND `FB == true`
  4. **LINKEDIN_IMAGE:** condition appears misconfigured:
     - `leftValue = {{ $json['L-in'] }}` with boolean `true` operator, but also includes `rightValue: "IMAGE"` (not coherent)
- **Connections (by output):**
  - INSTAGRAM_IMAGE → Instagram IMG
  - X_IMAGE → Get Media2
  - FACEBOOK_IMAGE → Post Facebook Img
  - LINKEDIN_IMAGE → Get Media
- **Edge cases / failures:**
  - If sheet values are `"TRUE"`/`"FALSE"` strings, boolean checks fail.
  - LinkedIn rule likely never matches due to inconsistent condition setup; it should probably be: `POST TYPE == "IMAGE"` AND `L-in == true`.

---

### Block C — Instagram image publishing

**Overview:**  
Creates an Instagram media container with image URL + caption, waits, then publishes the container.

**Nodes Involved:**  
- Instagram IMG
- Wait
- Instagram Post

#### Node: **Instagram IMG**
- **Type / Role:** Facebook Graph API (`facebookGraphApi`) — creates IG media container.
- **Endpoint logic:**
  - `node` (IG user id): `17841460793685821`
  - `edge`: `media`
  - HTTP method: POST
  - Query params:
    - `image_url` = `{{ $json.URL }}`
    - `caption` = `"test"` (hardcoded placeholder; not using sheet Description)
- **Credentials:** Facebook Graph API (“test aman”).
- **Connections:**
  - Input: Route (Instagram output)
  - Output: Wait
- **Failure modes / edge cases:**
  - IG requires a valid Business/Creator account connected to a Facebook Page and proper permissions (`instagram_basic`, `instagram_content_publish`).
  - `caption` is not dynamic; intended value is likely `{{ $json.Description }}`.
  - Invalid/expired image URL causes container creation failure.

#### Node: **Wait**
- **Type / Role:** Wait (`wait`) — delays to allow IG media container readiness.
- **Configuration:** No explicit duration shown; uses Wait node mechanism with stored `webhookId`.
- **Connections:**
  - Input: Instagram IMG
  - Output: Instagram Post
- **Failure modes:**
  - If not configured for a time-based wait in UI, it may pause indefinitely until resumed.
  - Operationally, IG container readiness often needs a short delay; too short can fail publish.

#### Node: **Instagram Post**
- **Type / Role:** Facebook Graph API (`facebookGraphApi`) — publishes IG media container.
- **Endpoint logic:**
  - `node`: `17841460793685821`
  - `edge`: `media_publish`
  - Query param: `creation_id` = `{{ $json.id }}`
- **Connections:**
  - Input: Wait
  - Output: Update Sheet
- **Failure modes:**
  - Permissions missing, container not ready, invalid creation_id.

---

### Block D — X (Twitter) image publishing

**Overview:**  
Downloads the image as binary, uploads it to X as media, then creates a tweet with the uploaded media attached.

**Nodes Involved:**  
- Get Media2
- Post  X Img
- Create Tweet

#### Node: **Get Media2**
- **Type / Role:** HTTP Request (`httpRequest`) — fetches image from `URL`.
- **URL expression:** `{{ $json.URL }}`
- **Connections:**
  - Input: Route (X output)
  - Output: Post  X Img
- **Edge cases:**
  - Must return binary data for upload. If node is not set to “Download”/binary (depends on n8n settings), upload will fail.
  - 403/404 from hosting, large files, timeouts.

#### Node: **Post  X Img**
- **Type / Role:** HTTP Request (`httpRequest`) — uploads media to X API.
- **Request:**
  - POST `https://api.x.com/2/media/upload`
  - Content-Type: multipart/form-data
  - Auth: predefined credential type `twitterOAuth2Api`
  - Body:
    - `media` from form binary data field `data`
    - `media_category` = `tweet_image`
- **Retry:** `retryOnFail: true`
- **Connections:**
  - Input: Get Media2
  - Output: Create Tweet
- **Failure modes / edge cases:**
  - X media upload endpoint/versioning may differ by account tier/permissions; failures may return 403/404.
  - Binary property name must match (`data`). If Get Media2 outputs binary under another key, upload fails.
  - OAuth scopes must include tweet/media permissions.

#### Node: **Create Tweet**
- **Type / Role:** Twitter/X Node (`twitter`) — creates the tweet with text + media attachment.
- **Text expression:** `{{ $('get sheet data').item.json.Description }}`
  - This references the **original sheet item** rather than the current `$json` from upload chain.
- **Attachments:** `{{ $json.data.id }}` (expects upload response at `data.id`)
- **Connections:**
  - Input: Post  X Img
  - Output: Update Sheet
- **Edge cases / likely issues:**
  - If the upload response shape is not `$json.data.id`, attachment will be missing.
  - Using `$('get sheet data').item...` can be risky in loops if item linkage is lost; safer is `{{ $item(0).$json.Description }}` or passing Description forward explicitly.

---

### Block E — Facebook image publishing

**Overview:**  
Fetches an image (currently from a hardcoded URL), then uploads it to a Facebook page as a photo post.

**Nodes Involved:**  
- Post Facebook Img
- Facebook Post

#### Node: **Post Facebook Img**
- **Type / Role:** HTTP Request (`httpRequest`) — downloads the image.
- **URL:** Hardcoded Cloudinary image URL (not `{{ $json.URL }}`).
- **Connections:**
  - Input: Route (Facebook output)
  - Output: Facebook Post
- **Edge cases:**
  - This ignores the sheet row’s `URL`; all Facebook posts will use the same hardcoded image unless changed.
  - Must output binary as `data` for the Facebook Post node.

#### Node: **Facebook Post**
- **Type / Role:** Facebook Graph API (`facebookGraphApi`) — uploads photo to a page.
- **Endpoint logic:**
  - `node`: `/824769547395395/photos` (page photos edge)
  - `edge`: `photos`
  - Method: POST
  - `sendBinaryData: true`, `binaryPropertyName: data`
  - Query param: `message` = `"hey there"` (hardcoded placeholder; not using Description)
- **Connections:**
  - Input: Post Facebook Img
  - Output: Update Sheet
- **Failure modes / edge cases:**
  - Requires page access token with `pages_manage_posts` / `pages_read_engagement` etc.
  - Message and image are not dynamic; likely intended: message = `{{ $json.Description }}` and image from sheet URL.

---

### Block F — LinkedIn image publishing

**Overview:**  
Downloads the image from the sheet URL, then creates a LinkedIn post with shareMediaCategory set to IMAGE.

**Nodes Involved:**  
- Get Media
- Linkedin Post

#### Node: **Get Media**
- **Type / Role:** HTTP Request (`httpRequest`) — fetches the image from `URL`.
- **URL expression:** `{{ $json.URL }}`
- **Connections:**
  - Input: Route (LinkedIn output)
  - Output: Linkedin Post
- **Edge cases:**
  - LinkedIn image posting often requires a multi-step upload registration flow; the simple LinkedIn node may handle this internally, but binary handling must be correct.
  - If the routing condition never matches (likely), this branch won’t run.

#### Node: **Linkedin Post**
- **Type / Role:** LinkedIn (`linkedIn`) — creates a post.
- **Parameters:**
  - `text` = `{{ $json.Description }}`
  - `person` = `UsSnS-LAtb`
  - `shareMediaCategory` = `IMAGE`
- **Connections:**
  - Input: Get Media
  - Output: Update Sheet
- **Failure modes / edge cases:**
  - LinkedIn API permissions (w_member_social) and app review constraints.
  - If media binary is not provided/recognized, the post may fail or post without image depending on node behavior.

---

### Block G — Sheet status update after posting

**Overview:**  
Marks content as posted in Google Sheets after a platform action succeeds.

**Nodes Involved:**  
- Update Sheet

#### Node: **Update Sheet**
- **Type / Role:** Google Sheets (`googleSheets`, operation: update) — updates STATUS to “posted”.
- **Matching column:** `Description` (updates row(s) whose Description matches).
- **Values set:**
  - `STATUS` = `"posted"`
  - `Description` = `{{ $('get sheet data').item.json.Description }}`
- **Connections:**
  - Inputs: Create Tweet, Facebook Post, Linkedin Post, Instagram Post
  - Output: (none)
- **Edge cases / risks:**
  - **Matching on Description is unsafe**: captions may repeat or be edited; multiple rows could match.
  - Better: match using a stable unique key (row_number, an ID column, timestamp, etc.).
  - The expression uses `$('get sheet data').item...` which can break if item pairing isn’t preserved through routing branches.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point for caption generation | — | Post Topic | Click to run workflow – Trigger the workflow manually / Set topic in the Set node / LLM generates caption / Store in the sheet |
| Post Topic | Set | Define topic + image URL for caption generation | When clicking ‘Execute workflow’ | Generate Caption | Update the title with the topic you want to post and add the image URL in the Set node. |
| Generate Caption | OpenAI (LangChain) | Generate caption using GPT-4.1-mini | Post Topic | Append row in sheet | Click to run workflow – Trigger the workflow manually / Set topic in the Set node / LLM generates caption / Store in the sheet |
| Append row in sheet | Google Sheets | Append generated draft (URL, POST TYPE, Description) | Generate Caption | — | Click to run workflow – Trigger the workflow manually / Set topic in the Set node / LLM generates caption / Store in the sheet |
| Schedule social media posts | Schedule Trigger | Scheduled entry point to process approved posts | — | get sheet data | Once reviewed, update the status to ‘Approved’ in the sheet. Then the workflow will run and post on the respective platform(s) you selected. |
| get sheet data | Google Sheets | Read rows filtered by STATUS=post | Schedule social media posts | Loop Over Items | Once reviewed, update the status to ‘Approved’ in the sheet. Then the workflow will run and post on the respective platform(s) you selected. |
| Loop Over Items | Split In Batches | Iterate through sheet rows | get sheet data | Route (output index 1) | Once reviewed, update the status to ‘Approved’ in the sheet. Then the workflow will run and post on the respective platform(s) you selected. |
| Route | Switch | Route each row to platform branches | Loop Over Items | Instagram IMG; Get Media2; Post Facebook Img; Get Media | Once reviewed, update the status to ‘Approved’ in the sheet. Then the workflow will run and post on the respective platform(s) you selected. |
| Instagram IMG | Facebook Graph API | Create IG media container | Route | Wait |  |
| Wait | Wait | Delay before IG publish | Instagram IMG | Instagram Post |  |
| Instagram Post | Facebook Graph API | Publish IG media container | Wait | Update Sheet |  |
| Get Media2 | HTTP Request | Download image for X upload | Route | Post  X Img |  |
| Post  X Img | HTTP Request | Upload image to X media endpoint | Get Media2 | Create Tweet |  |
| Create Tweet | Twitter/X | Create tweet with caption + media | Post  X Img | Update Sheet |  |
| Post Facebook Img | HTTP Request | Download image for Facebook post (currently hardcoded URL) | Route | Facebook Post |  |
| Facebook Post | Facebook Graph API | Post photo to Facebook page (hardcoded message) | Post Facebook Img | Update Sheet |  |
| Get Media | HTTP Request | Download image for LinkedIn | Route | Linkedin Post |  |
| Linkedin Post | LinkedIn | Post to LinkedIn with image category | Get Media | Update Sheet |  |
| Update Sheet | Google Sheets | Mark row as posted | Create Tweet; Facebook Post; Linkedin Post; Instagram Post | — |  |
| Sticky Note8 | Sticky Note | Creator information / links | — | — | ## 👨‍💻 Creator Information / **Created by:** Samyotech / 🔗 **LinkedIn:** https://www.linkedin.com/company/samyotech/posts/?feedView=all / 🔗 **Youtube:** https://www.youtube.com/@samyotech / Need Help: https://samyotech.com/ |
| Sticky Note | Sticky Note | Manual run instructions | — | — | Click to run workflow – Trigger the workflow manually / Set topic in the Set node / LLM generates caption / Store in the sheet |
| Sticky Note1 | Sticky Note | Review/approve instructions | — | — | Review the generated caption in the sheet, update its status to ‘Approved’, and select the platform(s) where you want to post. |
| Sticky Note2 | Sticky Note | Set node guidance | — | — | Update the title with the topic you want to post and add the image URL in the Set node. |
| Sticky Note3 | Sticky Note | Approval-to-post explanation | — | — | Once reviewed, update the status to ‘Approved’ in the sheet. Then the workflow will run and post on the respective platform(s) you selected. |
| Sticky Note4 | Sticky Note | Workflow concept + setup steps | — | — | Social media posting automation with images and captions / Setup steps (OpenAI, Google, credentials, test run) |

> Note: Sticky notes are included as nodes in this workflow export; they do not connect to the execution graph.

---

## 4. Reproducing the Workflow from Scratch

### A) Create the caption generation (manual) path
1. **Add node:** *Manual Trigger*  
   - Name: `When clicking ‘Execute workflow’`

2. **Add node:** *Set*  
   - Name: `Post Topic`  
   - Add fields:
     - `title` (string) — e.g. “Ai Automation”
     - `image_url` (string) — public image URL (Cloudinary, etc.)
     - (optional) `phone` (string) — only if you plan to use it later

3. **Connect:** Manual Trigger → Post Topic

4. **Add node:** *OpenAI (LangChain)*  
   - Name: `Generate Caption`  
   - Model: `gpt-4.1-mini`  
   - Messages:
     - System message containing your caption rules/constraints
     - User message: `Topic for Today's Post: {{ $json.title }}`
   - **Credentials:** Create/OpenAI credential (API key) and select it.

5. **Connect:** Post Topic → Generate Caption

6. **Add node:** *Google Sheets* (operation: **Append**)  
   - Name: `Append row in sheet`  
   - Credentials: connect Google account (OAuth2)  
   - Document: select your spreadsheet  
   - Sheet: select the tab (gid=0 / “Video Content data”)  
   - Map columns:
     - `URL` = `{{ $('Post Topic').item.json.image_url }}`
     - `POST TYPE` = `IMAGE`
     - `Description` = `{{ $json.message.content }}`

7. **Connect:** Generate Caption → Append row in sheet

---

### B) Create the scheduled posting path (queue processor)
8. **Add node:** *Schedule Trigger*  
   - Name: `Schedule social media posts`  
   - Configure interval (e.g., every 60 seconds / every 10 minutes as desired).

9. **Add node:** *Google Sheets* (operation: **Read/Get Many**, depending on UI)  
   - Name: `get sheet data`  
   - Same document + sheet as above  
   - Add filter: `STATUS` equals `post` (or align to your intended “Approved” value).

10. **Connect:** Schedule social media posts → get sheet data

11. **Add node:** *Split In Batches*  
   - Name: `Loop Over Items`  
   - Batch size: set as needed (default is typically fine).

12. **Connect:** get sheet data → Loop Over Items  
   - In your own build, connect **Output 0** for processing (recommended), unless you intentionally replicate the export’s Output 1 usage.

13. **Add node:** *Switch*  
   - Name: `Route`  
   - Enable: “Send to all matching outputs” (allMatchingOutputs)  
   - Add rules (recommended corrected form):
     - Instagram: `POST TYPE == "IMAGE"` AND `IG == true`
     - X: `POST TYPE == "IMAGE"` AND `X == true`
     - Facebook: `POST TYPE == "IMAGE"` AND `FB == true`
     - LinkedIn: `POST TYPE == "IMAGE"` AND `L-in == true`

14. **Connect:** Loop Over Items → Route

---

### C) Instagram branch
15. **Add node:** *Facebook Graph API*  
   - Name: `Instagram IMG`  
   - Graph API version: v20.0  
   - Method: POST  
   - Node (IG User ID): your IG business account id  
   - Edge: `media`  
   - Query params:
     - `image_url` = `{{ $json.URL }}`
     - `caption` = `{{ $json.Description }}` (recommended; export uses `test`)
   - Credentials: Facebook Graph credential with IG publish permissions.

16. **Add node:** *Wait*  
   - Name: `Wait`  
   - Configure a time-based wait (e.g., 5–30 seconds).

17. **Add node:** *Facebook Graph API*  
   - Name: `Instagram Post`  
   - Method: POST  
   - Edge: `media_publish`  
   - Query param: `creation_id` = `{{ $json.id }}`

18. **Connect:** Route(Instagram) → Instagram IMG → Wait → Instagram Post

---

### D) X (Twitter) branch
19. **Add node:** *HTTP Request*  
   - Name: `Get Media2`  
   - URL: `{{ $json.URL }}`  
   - Configure to **download** the file as binary (ensure binary property name is `data` if you keep the upload mapping).

20. **Add node:** *HTTP Request*  
   - Name: `Post  X Img`  
   - Method: POST  
   - URL: `https://api.x.com/2/media/upload`  
   - Auth: Twitter OAuth2 credential  
   - Content-Type: multipart/form-data  
   - Body params:
     - `media` = formBinaryData from `data`
     - `media_category` = `tweet_image`

21. **Add node:** *Twitter*  
   - Name: `Create Tweet`  
   - Text: `{{ $json.Description }}` (recommended; export references `get sheet data`)  
   - Attachments: expression pointing to upload response media id (verify response path; export uses `{{ $json.data.id }}`)

22. **Connect:** Route(X) → Get Media2 → Post  X Img → Create Tweet

---

### E) Facebook branch
23. **Add node:** *HTTP Request*  
   - Name: `Post Facebook Img`  
   - URL: `{{ $json.URL }}` (recommended; export is hardcoded)  
   - Configure binary download to `data`.

24. **Add node:** *Facebook Graph API*  
   - Name: `Facebook Post`  
   - Graph API v20.0, method POST  
   - Node: `/{PAGE_ID}/photos`  
   - Edge: `photos`  
   - Enable `sendBinaryData`, binary property `data`  
   - Query param `message` = `{{ $json.Description }}` (recommended; export uses `hey there`)  
   - Credentials: Facebook Graph with page posting permissions.

25. **Connect:** Route(Facebook) → Post Facebook Img → Facebook Post

---

### F) LinkedIn branch
26. **Add node:** *HTTP Request*  
   - Name: `Get Media`  
   - URL: `{{ $json.URL }}`  
   - Configure binary download if required by your LinkedIn node usage.

27. **Add node:** *LinkedIn*  
   - Name: `Linkedin Post`  
   - Person URN/id: select your account (export uses `UsSnS-LAtb`)  
   - Text: `{{ $json.Description }}`  
   - Share media category: IMAGE  
   - Credentials: LinkedIn OAuth2 with posting permissions.

28. **Connect:** Route(LinkedIn) → Get Media → Linkedin Post

---

### G) Update Google Sheets after posting
29. **Add node:** *Google Sheets* (operation: **Update**)  
   - Name: `Update Sheet`  
   - Document/sheet: same as above  
   - Matching: **use a unique identifier** (recommended: `row_number` if you read it, or add your own `POST_ID`)  
   - Set:
     - `STATUS` = `posted`

30. **Connect:** Instagram Post → Update Sheet; Create Tweet → Update Sheet; Facebook Post → Update Sheet; Linkedin Post → Update Sheet

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Created by:** Samyotech | From sticky note |
| LinkedIn: Connect with me | https://www.linkedin.com/company/samyotech/posts/?feedView=all |
| YouTube: Subscribe to my Channel | https://www.youtube.com/@samyotech |
| Need Help / Visit | https://samyotech.com/ |
| Sheet review process: generate caption → review in sheet → set status + platform flags | Sticky notes describe a human-in-the-loop approval step |
| Setup guidance: connect OpenAI, Google Sheets, and social platform credentials | Sticky note “Setup steps” |

**Disclaimer (provided):** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.