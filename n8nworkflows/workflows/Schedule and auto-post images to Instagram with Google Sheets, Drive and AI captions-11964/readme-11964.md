Schedule and auto-post images to Instagram with Google Sheets, Drive and AI captions

https://n8nworkflows.xyz/workflows/schedule-and-auto-post-images-to-instagram-with-google-sheets--drive-and-ai-captions-11964


# Schedule and auto-post images to Instagram with Google Sheets, Drive and AI captions

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically pick the next “Ready to Post” row from a Google Sheet, find the matching image in Google Drive (by date), generate an Instagram caption + hashtags using an OpenAI-powered agent, then publish the image to Instagram via the Meta Graph API. Finally, it updates the Google Sheet with **Posted** or **Failed** status and stores the generated content.

**Typical use cases**
- Scheduled/queued Instagram posting driven by a spreadsheet content calendar.
- Centralized asset management in Google Drive, with AI-generated captions.
- Hands-off hourly polling (instead of webhooks) for “what should be posted next”.

### Logical blocks
1. **1.1 Trigger & configuration**
   - Runs every hour and defines reusable IDs (Sheet/Drive).
2. **1.2 Fetch next post from Google Sheets**
   - Finds the first row with `status = "Ready to Post"` and stops gracefully if none.
3. **1.3 Meta authentication**
   - Retrieves a Page access token required for subsequent Graph calls.
4. **1.4 Image discovery and preparation**
   - Searches Drive for an image matching the post date; downloads it; uploads it to Facebook (unpublished) to obtain a public image URL usable by Instagram container creation.
5. **1.5 AI caption/hashtag generation**
   - Uses a LangChain Agent connected to an OpenAI chat model; returns strict JSON.
6. **1.6 Instagram publish pipeline**
   - Creates an Instagram media container, waits, then publishes it.
7. **1.7 Result handling & sheet update**
   - Detects success and updates the Google Sheet accordingly.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & configuration

**Overview:** Starts the workflow on a fixed schedule and stores key IDs (Sheet/Drive) in a single place for reuse in expressions.

**Nodes involved:**
- Daily Trigger
- Workflow Configuration

#### Node: Daily Trigger
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Config choices:** Runs every **1 hour** (`rule.interval` with `field: hours`).
- **Inputs/outputs:** No inputs; outputs one execution item to **Workflow Configuration**.
- **Edge cases / failures:**
  - Timezone is governed by n8n instance/workflow settings; “hourly” may drift if instance is paused.
- **Version notes:** typeVersion `1.3`.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralized configuration variables.
- **Config choices:**
  - Sets:
    - `googleDriveFolderId = "<YOUR-DRIVE-ID>"` (note: currently **not used** later)
    - `googleSheetId = "<YOUR-SHEET-ID>"`
  - `includeOtherFields: true` keeps incoming fields (though trigger has minimal payload).
- **Key variables used later:**
  - `$('Workflow Configuration').first().json.googleSheetId`
- **Outputs:** Feeds into **Get Next Scheduled Post**.
- **Edge cases:**
  - If IDs remain placeholders, downstream Google nodes fail (auth ok but resource not found).
- **Version notes:** typeVersion `3.4`.

---

### 2.2 Fetch next post from Google Sheets (and stop if none)

**Overview:** Reads the content calendar, filters to rows ready to post, and branches based on whether a row exists.

**Nodes involved:**
- Get Next Scheduled Post
- Check If Post Found
- No More Posts

#### Node: Get Next Scheduled Post
- **Type / role:** `Google Sheets` — lookup the next candidate row.
- **Config choices (interpreted):**
  - Operation: read with filter where `status == "Ready to Post"`.
  - Range: `A:G` (A1 notation), returning the **first match only** (`returnFirstMatch: true`).
  - Sheet: `Sheet1` (via gid=0 in UI).
  - Document ID pulled from configuration: `={{ $('Workflow Configuration').first().json.googleSheetId }}`
  - `onError: continueErrorOutput` lets execution continue even if the node errors (but the output may be empty/partial).
- **Inputs/outputs:** Receives config item; outputs row item(s) to **Check If Post Found**.
- **Edge cases / failures:**
  - Auth/consent issues with Google OAuth.
  - Range/schema mismatch (missing columns like `status`, `Topic`, `Date`, etc.).
  - If the node errors but “continue” is enabled, later nodes may behave as if “no post found”.
- **Version notes:** typeVersion `4.7`.

#### Node: Check If Post Found
- **Type / role:** `IF` — guards the rest of the workflow.
- **Config choices:**
  - Condition: `{{ $input.all().length }} > 0`
- **Branches:**
  - **True** → continues to Meta token retrieval (**Get Access Token from Facebook**).
  - **False** → **No More Posts**.
- **Edge cases:**
  - If Google Sheets node returned an error with continue enabled, `$input.all().length` may be 0 even though the issue is not “no posts”.
- **Version notes:** typeVersion `2.2`.

#### Node: No More Posts
- **Type / role:** `Code` — graceful termination message.
- **Config choices:** Returns a single JSON object:
  - `message: "No posts scheduled for today or all posts completed"`
- **Outputs:** Terminal in this workflow (no outgoing connections shown).
- **Edge cases:** None significant.
- **Version notes:** typeVersion `2`.

---

### 2.3 Meta authentication (Page access token retrieval)

**Overview:** Fetches the Facebook Pages accessible to a user token, and uses the Page access token for later upload/container operations.

**Nodes involved:**
- Get Access Token from Facebook

#### Node: Get Access Token from Facebook
- **Type / role:** `HTTP Request` — calls Meta Graph API to list accounts/pages and retrieve page tokens.
- **Config choices:**
  - GET `https://graph.facebook.com/me/accounts?access_token=<YOUR-ACCESS-TOKEN>`
  - Uses a **long-lived user access token** placeholder directly in URL (not a credential).
- **Outputs:** Feeds token data into **Search files and folders** and later HTTP nodes via expressions like:
  - `$('Get Access Token from Facebook').item.json.data[0].access_token`
- **Edge cases / failures:**
  - `<YOUR-ACCESS-TOKEN>` expiring or lacking permissions (common failure).
  - Choosing `data[0]` assumes the needed Page is always the first entry; wrong page = wrong token.
  - Rate limits / Graph API errors.
- **Version notes:** typeVersion `4.3`.
- **Recommendation:** Prefer storing tokens in n8n credentials or environment variables; select the correct Page by ID instead of `data[0]`.

---

### 2.4 Image discovery and preparation (Drive → binary → Facebook-hosted public URL)

**Overview:** Finds the image file in Google Drive using the scheduled date, downloads it, uploads it to Facebook as unpublished media to obtain a reliable public image URL for Instagram container creation.

**Nodes involved:**
- Search files and folders
- Download Image from Drive
- Upload Image to Facebook
- Get Public URL From Facebook
- Prepare AI Context

#### Node: Search files and folders
- **Type / role:** `Google Drive` search — locate the image asset.
- **Config choices:**
  - Resource: `fileFolder` (search)
  - Folder filter: currently fixed to **root** (not using `googleDriveFolderId`)
  - Query string: `={{ $('Get Next Scheduled Post').item.json.Date + '.*' }}`
  - Limit: `1`
  - Fields: `id`, `name`
- **Inputs/outputs:** Uses Sheet row fields; outputs file metadata to **Download Image from Drive**.
- **Edge cases / failures:**
  - If `Date` formatting in the sheet does not match filenames, search returns nothing.
  - `limit:1` can select the wrong file if multiple matches exist.
  - Searching only root can miss files stored in a dedicated folder.
- **Version notes:** typeVersion `3`.

#### Node: Download Image from Drive
- **Type / role:** `Google Drive` download — fetches binary image data.
- **Config choices:**
  - Operation: `download`
  - File ID: `={{ $json.id }}` (from search result)
  - Binary property: `data`
- **Inputs/outputs:** Outputs binary `data` to **Upload Image to Facebook**.
- **Edge cases / failures:**
  - File not found / permission denied.
  - Non-image file downloaded, later upload may fail.
- **Version notes:** typeVersion `3`.

#### Node: Upload Image to Facebook
- **Type / role:** `HTTP Request` multipart upload to Facebook Photos endpoint to host the image.
- **Config choices:**
  - POST `https://graph.facebook.com/v24.0/<YOUR-ACCOUNT-D>/photos`
    - `<YOUR-ACCOUNT-D>` is a placeholder (likely a Page ID). Must be correct.
  - Multipart form-data body:
    - `data` = binary from input field `data`
    - `published=false` (unpublished upload used as asset)
    - `access_token={{ $('Get Access Token from Facebook').item.json.data[0].access_token }}`
  - Also sets a header `Content-Type: application/json` (this is **inconsistent** with multipart form-data and may cause issues depending on n8n/http library behavior).
- **Inputs/outputs:** Consumes binary; outputs uploaded object with an `id` to **Get Public URL From Facebook**.
- **Edge cases / failures:**
  - Wrong token type/permissions (needs appropriate Graph permissions).
  - Header mismatch (`multipart-form-data` + forcing `application/json`) can break uploads.
  - If binary property name mismatches (not `data`), upload fails.
- **Version notes:** typeVersion `4.3`.

#### Node: Get Public URL From Facebook
- **Type / role:** `HTTP Request` — fetches image renditions/URLs for the uploaded photo ID.
- **Config choices:**
  - GET `https://graph.facebook.com/v19.0/{{ $json.id }}?fields=images&access_token={{ ...page_access_token... }}`
  - Uses `images` field; expects `images[0].source`.
- **Inputs/outputs:** Outputs `images` array to **Prepare AI Context**.
- **Edge cases / failures:**
  - Graph API version mismatch (upload v24.0, fetch v19.0) can introduce subtle changes; standardize if possible.
  - `images[0]` may not be the desired size ordering; could select a very large rendition.
- **Version notes:** typeVersion `4.3`.

#### Node: Prepare AI Context
- **Type / role:** `Set` — normalizes fields needed later.
- **Config choices:**
  - `image_public_url = {{ $json.images[0].source }}`
  - `image_facebook_id = {{ $('Upload Image to Facebook').item.json.id }}`
  - Keeps other fields.
- **Inputs/outputs:** Outputs to **Generate Caption and Hashtags** (main input).
- **Edge cases:**
  - If `images` is missing/empty, expression fails.
- **Version notes:** typeVersion `3.4`.

---

### 2.5 AI caption/hashtag generation

**Overview:** Uses a LangChain Agent with an OpenAI chat model to generate Instagram caption and hashtags as strict JSON, based on Sheet fields and brand context.

**Nodes involved:**
- OpenAI Chat Model
- Generate Caption and Hashtags

#### Node: OpenAI Chat Model
- **Type / role:** LangChain chat model connector (`@n8n/n8n-nodes-langchain.lmChatOpenAi`).
- **Config choices:**
  - Model: `gpt-5-mini`
  - Built-in tool: `webSearch` enabled (`searchContextSize: medium`) (agent may use it if allowed by agent config).
- **Connections:** Connected to **Generate Caption and Hashtags** via the `ai_languageModel` port.
- **Edge cases / failures:**
  - OpenAI credential missing/invalid.
  - Tool usage may increase latency/cost; could cause timeouts.
- **Version notes:** typeVersion `1.3`.

#### Node: Generate Caption and Hashtags
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — produces structured output.
- **Config choices:**
  - Prompt instructs: return **JSON only**, with structure:
    ```json
    { "instagram": { "post_content": "...", "hashtags": ["#..."] } }
    ```
  - Uses sheet fields:
    - `Topic: {{ $('Get Next Scheduled Post').item.json.Topic }}`
    - `Angle/Question: {{ $('Get Next Scheduled Post').item.json['Question / Angle'] }}`
  - System message enforces length, separation of hashtags, no URLs, etc.
  - `onError: continueErrorOutput` allows continuing even if the agent fails (dangerous if downstream expects valid JSON).
- **Outputs:** Sends to **Post to Instagram**.
- **Key downstream expressions:**
  - `JSON.parse($node["Generate Caption and Hashtags"].json["output"]).instagram.post_content`
  - `JSON.parse(...).instagram.hashtags`
- **Edge cases / failures:**
  - Agent returns invalid JSON → `JSON.parse(...)` fails later.
  - Agent returns `hashtags` array; workflow later injects it directly into caption. In JavaScript string context, an array becomes comma-separated; Instagram expects hashtags separated by spaces/newlines.
- **Version notes:** typeVersion `3`.

---

### 2.6 Instagram publish pipeline (container → wait → publish)

**Overview:** Creates an Instagram media container using the public image URL and generated caption, waits for processing, then publishes the container.

**Nodes involved:**
- Post to Instagram
- 6b. Wait for Container Processing
- Publish Post to Instagram
- Check Post Success

#### Node: Post to Instagram
- **Type / role:** `Facebook Graph API` — creates an IG media container (`/{ig-user-id}/media`).
- **Config choices:**
  - Edge: `media`
  - Node: `<PAGE-ID>` (placeholder; in IG API this should be the **Instagram User ID (ig-user-id)** connected to the Page, not the Page ID—naming here is ambiguous).
  - Method: POST, Graph version `v23.0`
  - Query parameters:
    - `caption` combines caption + hashtags using:
      - `post_content` + newline + `hashtags`
    - `image_url = {{ $('Prepare AI Context').item.json.image_public_url }}`
  - `retryOnFail: true`
- **Outputs:** If successful, returns a `creation_id`-like `id` used by publish step; connected to **Wait**.
- **Edge cases / failures:**
  - Wrong “node” ID (Page ID vs IG user ID) → Graph error.
  - Caption formatting: hashtags array stringifies with commas unless joined.
  - If agent failed and output is missing, JSON.parse will error.
- **Version notes:** typeVersion `1`.

#### Node: 6b. Wait for Container Processing
- **Type / role:** `Wait` — delays to let Meta process the container.
- **Config choices:** Default wait node (no explicit duration shown in parameters). Despite the sticky note mentioning 5 seconds, the node parameters are empty; actual behavior depends on node configuration in UI (may be “wait indefinitely for webhook” vs “wait amount of time”). Here it appears as a standard wait step but **duration is not explicit**.
- **Outputs:** Continues to **Publish Post to Instagram**.
- **Edge cases / failures:**
  - If configured as “wait for webhook” accidentally, workflow will hang.
  - If wait time is too short, `media_publish` may fail with “container not ready”.
- **Version notes:** typeVersion `1.1`.

#### Node: Publish Post to Instagram
- **Type / role:** `Facebook Graph API` — publishes the container (`/{ig-user-id}/media_publish`).
- **Config choices:**
  - Edge: `media_publish`
  - Node: `=<PAGE-ID>` (again placeholder/ambiguous; should match IG user id)
  - Query param: `creation_id = {{ $json.id }}`
  - Graph version `v22.0`
  - `retryOnFail: true`
- **Outputs:** Feeds to **Check Post Success**.
- **Edge cases / failures:**
  - Container not ready (timing).
  - Wrong node id.
- **Version notes:** typeVersion `1`.

#### Node: Check Post Success
- **Type / role:** `IF` — routes to Posted vs Failed updates.
- **Config choices:** Checks:
  - `{{ $node["Post to Instagram"].json.id ? true : false }} == true`
  - Note: This checks the **container creation** response, not the publish response. A publish failure could still mark as success if container creation succeeded.
- **Branches:**
  - True → **Update Sheet - Posted**
  - False → **Update Sheet - Failed**
- **Edge cases:**
  - False negatives/positives due to checking the wrong node’s output.
- **Version notes:** typeVersion `2.2`.

---

### 2.7 Result handling & Google Sheet status updates

**Overview:** Writes back posting status and generated content to the same row using `row_number` as the matching key.

**Nodes involved:**
- Update Sheet - Posted
- Update Sheet - Failed

#### Node: Update Sheet - Posted
- **Type / role:** `Google Sheets` update — records success + content.
- **Config choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Values set:
    - `row_number = {{ $node["Get Next Scheduled Post"].json.row_number }}`
    - `Instagram_Status = "Posted"`
    - `Instagram Post Content + hashtags` = caption + hashtags (same JSON.parse pattern)
  - Document ID from configuration.
- **Edge cases / failures:**
  - If `row_number` is missing, update won’t match.
  - JSON.parse failures if AI output invalid.
- **Version notes:** typeVersion `4.7`.

#### Node: Update Sheet - Failed
- **Type / role:** `Google Sheets` update — records failure for review.
- **Config choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Values set:
    - `Instagram_Status = "Failed"`
- **Edge cases / failures:**
  - Same row_number concerns.
- **Version notes:** typeVersion `4.7`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Trigger | scheduleTrigger | Hourly workflow entrypoint | — | Workflow Configuration | 📅 WORKFLOW START … This trigger works every 1 hour |
| Workflow Configuration | set | Store Sheet/Drive IDs | Daily Trigger | Get Next Scheduled Post | 🔐 AUTHENTICATION … (full block note applies) |
| Get Next Scheduled Post | googleSheets | Fetch first row with status “Ready to Post” | Workflow Configuration | Check If Post Found | 🔐 AUTHENTICATION … (full block note applies) |
| Check If Post Found | if | Branch if a row exists | Get Next Scheduled Post | Get Access Token from Facebook; No More Posts | 🔐 AUTHENTICATION … (full block note applies) |
| No More Posts | code | Graceful termination message | Check If Post Found (false) | — | 🔐 AUTHENTICATION … (full block note applies) |
| Get Access Token from Facebook | httpRequest | Retrieve Page access token | Check If Post Found (true) | Search files and folders | 🔐 AUTHENTICATION … (full block note applies) |
| Search files and folders | googleDrive | Find image file by date pattern | Get Access Token from Facebook | Download Image from Drive | 🔐 AUTHENTICATION … (full block note applies) |
| Download Image from Drive | googleDrive | Download image binary | Search files and folders | Upload Image to Facebook | 🔐 AUTHENTICATION … (full block note applies) |
| Upload Image to Facebook | httpRequest | Upload image as unpublished to get hosted asset | Download Image from Drive | Get Public URL From Facebook | 🔐 AUTHENTICATION … (full block note applies) |
| Get Public URL From Facebook | httpRequest | Fetch public image URL from uploaded asset | Upload Image to Facebook | Prepare AI Context | 🔐 AUTHENTICATION … (full block note applies) |
| Prepare AI Context | set | Store image URL + FB image ID | Get Public URL From Facebook | Generate Caption and Hashtags | 🔐 AUTHENTICATION … (full block note applies) |
| OpenAI Chat Model | lmChatOpenAi | LLM model backing the agent | — | Generate Caption and Hashtags (ai_languageModel) | Optional Add your Facebook Credentials… (note near AI area) |
| Generate Caption and Hashtags | langchain agent | Generate IG caption + hashtags JSON | Prepare AI Context (+ model connection) | Post to Instagram | 🔐 AUTHENTICATION … (full block note applies) |
| Post to Instagram | facebookGraphApi | Create IG media container | Generate Caption and Hashtags | 6b. Wait for Container Processing | 🔐 AUTHENTICATION … (full block note applies) |
| 6b. Wait for Container Processing | wait | Delay until container is ready | Post to Instagram | Publish Post to Instagram | 🔐 AUTHENTICATION … (full block note applies) |
| Publish Post to Instagram | facebookGraphApi | Publish the IG container | 6b. Wait for Container Processing | Check Post Success | 🔐 AUTHENTICATION … (full block note applies) |
| Check Post Success | if | Route to Posted vs Failed | Publish Post to Instagram | Update Sheet - Posted; Update Sheet - Failed | 🔐 AUTHENTICATION … (full block note applies) |
| Update Sheet - Posted | googleSheets | Mark as posted + save content | Check Post Success (true) | — | 🔐 AUTHENTICATION … (full block note applies) |
| Update Sheet - Failed | googleSheets | Mark as failed | Check Post Success (false) | — | 🔐 AUTHENTICATION … (full block note applies) |

Additional sticky note (applies to relevant nodes in the image hosting section):
- **“Public URL… from AWS S3, imgbb, etc.. I have used Facebook…”** (contextual to Upload Image to Facebook / Get Public URL From Facebook / Prepare AI Context)

---

## 4. Reproducing the Workflow from Scratch

1. **Create trigger**
   1. Add **Schedule Trigger** node.
   2. Set interval to **every 1 hour**.

2. **Add configuration variables**
   1. Add a **Set** node named **Workflow Configuration**.
   2. Enable “Keep Only Set” = **false** (i.e., include other fields).
   3. Add fields:
      - `googleSheetId` (String) = your Google Sheet ID
      - `googleDriveFolderId` (String) = your Drive folder ID (optional; if you’ll use it in the Drive search)

3. **Fetch the next ready post**
   1. Add **Google Sheets** node named **Get Next Scheduled Post**.
   2. Authenticate with **Google Sheets OAuth2** credential.
   3. Document ID: expression `{{ $('Workflow Configuration').first().json.googleSheetId }}`
   4. Sheet: `Sheet1`
   5. Range: `A:G`
   6. Filter: column `status` equals `Ready to Post`
   7. Return: enable **Return First Match**.
   8. (Optional but matches workflow) set **On Error** to “Continue”.

4. **Branch if a post exists**
   1. Add **IF** node named **Check If Post Found**.
   2. Condition: `{{ $input.all().length }}` **greater than** `0`.
   3. False branch → add **Code** node named **No More Posts** returning a message object.

5. **Meta token retrieval**
   1. Add **HTTP Request** node named **Get Access Token from Facebook**.
   2. Method: GET
   3. URL: `https://graph.facebook.com/me/accounts?access_token=...`
      - Use a valid long-lived **User Access Token** with permission to list pages.
   4. (Recommended) Instead of `data[0]`, plan to pick the right page by ID in later expressions.

6. **Find and download the image from Drive**
   1. Add **Google Drive** node named **Search files and folders**.
      - Credential: Google Drive OAuth2
      - Resource: search file/folder
      - Folder: choose your folder (or root)
      - Query: `{{ $('Get Next Scheduled Post').item.json.Date + '.*' }}`
      - Limit: 1
      - Fields: `id,name`
   2. Add **Google Drive** node named **Download Image from Drive**.
      - Operation: Download
      - File ID: `{{ $json.id }}`
      - Binary property name: `data`

7. **Upload to Facebook to obtain a public image URL**
   1. Add **HTTP Request** node named **Upload Image to Facebook**.
      - Method: POST
      - URL: `https://graph.facebook.com/v24.0/<PAGE_OR_ACCOUNT_ID>/photos`
      - Body content type: **multipart-form-data**
      - Form fields:
        - `data` (binary) from input binary field `data`
        - `published` = `false`
        - `access_token` = `{{ $('Get Access Token from Facebook').item.json.data[0].access_token }}`
      - Avoid forcing `Content-Type: application/json` header when using multipart.
   2. Add **HTTP Request** node named **Get Public URL From Facebook**.
      - Method: GET
      - URL: `https://graph.facebook.com/v19.0/{{ $json.id }}?fields=images&access_token={{ ...page_access_token... }}`

8. **Prepare fields for posting + AI**
   1. Add **Set** node named **Prepare AI Context**.
   2. Add fields:
      - `image_public_url = {{ $json.images[0].source }}`
      - `image_facebook_id = {{ $('Upload Image to Facebook').item.json.id }}`

9. **Configure AI generation**
   1. Add **OpenAI Chat Model** (LangChain) node.
      - Credential: OpenAI API key
      - Model: `gpt-5-mini` (or your choice)
   2. Add **LangChain Agent** node named **Generate Caption and Hashtags**.
      - Set prompt to demand **JSON only** with `instagram.post_content` and `instagram.hashtags`.
      - Reference sheet fields via expressions.
   3. Connect **OpenAI Chat Model** → Agent via the **AI Language Model** connection.

10. **Create Instagram container**
   1. Add **Facebook Graph API** node named **Post to Instagram**.
   2. Edge: `media`
   3. Node: your **IG User ID** (ensure correct for Instagram Content Publishing API).
   4. Query parameters:
      - `image_url = {{ $('Prepare AI Context').item.json.image_public_url }}`
      - `caption = {{ JSON.parse($node["Generate Caption and Hashtags"].json["output"]).instagram.post_content }} ...`
        - Include hashtags as desired (ideally join array with spaces/newlines).
   5. Enable retry on fail.

11. **Wait for processing**
   1. Add **Wait** node named **6b. Wait for Container Processing**.
   2. Configure it explicitly as a **time-based wait** (e.g., 5–10 seconds).

12. **Publish to Instagram**
   1. Add **Facebook Graph API** node named **Publish Post to Instagram**.
   2. Edge: `media_publish`
   3. Node: same IG User ID as above.
   4. Query parameter:
      - `creation_id = {{ $json.id }}`
   5. Enable retry on fail.

13. **Success check and sheet updates**
   1. Add **IF** node named **Check Post Success**.
      - Prefer checking **Publish Post to Instagram** response ID (not the container creation), but to match current workflow: check `Post to Instagram` output.
   2. True branch: add **Google Sheets** node **Update Sheet - Posted**:
      - Operation: Update
      - Match column: `row_number`
      - Set:
        - `Instagram_Status` = `Posted`
        - `Instagram Post Content + hashtags` = parsed agent output
   3. False branch: add **Google Sheets** node **Update Sheet - Failed**:
      - Operation: Update
      - Match column: `row_number`
      - Set `Instagram_Status` = `Failed`

14. **Connect nodes in order**
   - Schedule Trigger → Workflow Configuration → Get Next Scheduled Post → Check If Post Found
   - If True: Get Access Token from Facebook → Search files and folders → Download Image from Drive → Upload Image to Facebook → Get Public URL From Facebook → Prepare AI Context → Generate Caption and Hashtags → Post to Instagram → Wait → Publish Post to Instagram → Check Post Success → (Posted/Failed updates)
   - If False: No More Posts

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Retrieves Facebook Page access token needed for Instagram API calls… used throughout the workflow…” | Sticky note: 🔐 AUTHENTICATION (overall workflow explanation) |
| “Public URL… It can be from any public URL from AWS S3, imgbb, etc… I have used Facebook… to avoid errors from external services.” | Sticky note near image URL step (asset hosting strategy) |
| “Optional: Add your Facebook Credentials in another loop to make agent post in both Instagram and Facebook” | Sticky note near AI section (extension idea) |

