Publish WordPress posts from Google Sheets with images and tags

https://n8nworkflows.xyz/workflows/publish-wordpress-posts-from-google-sheets-with-images-and-tags-13127


# Publish WordPress posts from Google Sheets with images and tags

## 1. Workflow Overview

**Title:** Publish WordPress posts from Google Sheets with images and tags  
**Workflow name (n8n):** Google Sheets to Worpress  
**Purpose:** Read rows from a Google Sheet where `status = READY`, publish each row as a WordPress post via the WP REST API, optionally upload a featured image from `image_url`, parse hashtags into WP tags (get or create), then update the original sheet row to `POSTED` with the new `post_url` and `posted_at`.

### 1.1 Inputs, configuration, and credentials
Sets base URLs and relies on Google Sheets OAuth2 + WordPress Basic Auth credentials.

### 1.2 Fetch READY rows from Google Sheets
Pulls only the rows eligible for publishing (`status=READY`).

### 1.3 Per-row processing loop
Iterates rows one-by-one to avoid mixing data across posts.

### 1.4 Optional featured image pipeline
If `image_url` exists: download image to binary → upload to WordPress Media → later attach as `featured_media`.

### 1.5 Tags pipeline (hashtags → WP tag IDs)
If `hashtags` exists: parse into tag slugs → for each tag: try fetch ID → if missing create → store IDs.

### 1.6 Publish post and update sheet
Builds the final WP post payload (title/excerpt/content/status/tags/featured_media), publishes to WP, then updates the same Google Sheets row.

---

## 2. Block-by-Block Analysis

### Block 1 — Inputs & credentials
**Overview:** Initializes reusable variables (site URL and sheet URL). All subsequent nodes reference these values.  
**Nodes involved:**  
- When clicking ‘Execute workflow’
- Variables

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`manualTrigger`) — starts the workflow on demand.
- **Config choices:** No parameters; purely an entry point.
- **Connections:** Outputs to **Variables**.
- **Failure types:** None (only user execution).

#### Node: Variables
- **Type / role:** Set (`set`) — stores constants used across the workflow.
- **Configuration (interpreted):**
  - `website_URL`: `https://ozwebexpert.com`
  - `google_sheet_URL`: URL of the spreadsheet (used by Google Sheets node via URL mode)
- **Key expressions/variables used by others:**
  - `$node["Variables"].json.website_URL`
  - `$json.google_sheet_URL` (passed forward)
- **Connections:** Outputs to **Get row(s) in sheet**.
- **Edge cases / failures:** If URL is wrong, downstream HTTP requests (WP) or sheet operations fail.

---

### Block 2 — Fetch READY rows
**Overview:** Loads only rows whose `status` column equals `READY`.  
**Nodes involved:**  
- Get row(s) in sheet

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets (`googleSheets`) — reads matching rows.
- **Configuration (interpreted):**
  - **Filter:** `status` equals `READY`
  - **Sheet selection:** `sheetName` is set by URL expression `={{ $json.google_sheet_URL }}` (expects a Sheets URL containing `#gid=...`)
  - **Document ID:** fixed to the same spreadsheet ID (also compatible with URL mode)
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Connections:** Outputs to **Code in JavaScript**.
- **Potential failures / edge cases:**
  - OAuth token expired/insufficient permissions
  - Sheet columns missing (`status`, etc.) → filter may return nothing or node may error depending on schema inference
  - If no rows match, downstream batch loop will have zero items (effectively no posts published)

---

### Block 3 — Initialize per-run tag storage + loop rows
**Overview:** Resets tag ID accumulator in workflow static data, then processes each sheet row in a controlled batch loop.  
**Nodes involved:**  
- Code in JavaScript
- Loop Over Items

#### Node: Code in JavaScript
- **Type / role:** Code (`code`) — initializes global static data for tag IDs.
- **Configuration (interpreted):**
  - Uses `$getWorkflowStaticData('global')`
  - Sets `data.wp_tag_ids = '0'` (a string accumulator, later appended as `0,12,34,...`)
- **Connections:** Outputs to **Loop Over Items**.
- **Edge cases / failures:**
  - Static data is **global**: if multiple executions overlap (or if you later add concurrency), tag IDs can leak between runs. Resetting here helps, but parallel executions can still collide.

#### Node: Loop Over Items
- **Type / role:** Split in Batches (`splitInBatches`) — iterates posts one by one.
- **Configuration (interpreted):** Default batch behavior (batch size not explicitly set; n8n defaults typically to 1 for this node unless changed).
- **Connections:**
  - **Input:** from Code in JavaScript
  - **Main output (index 1 in connections):** goes to **If ($image)** to process each row
  - **After Update row in sheet:** workflow returns to **Loop Over Items** to fetch the next batch item
- **Edge cases / failures:**
  - If a later node errors and the workflow stops, remaining READY rows will not be processed.
  - If batch size > 1, ensure downstream nodes don’t assume a single row.

---

### Block 4 — Optional featured image
**Overview:** If `image_url` is present, downloads it and uploads to WordPress Media using Basic Auth; the uploaded media ID can be used as featured image.  
**Nodes involved:**  
- If ($image)
- Load $image as binary
- Upload $image to WP

#### Node: If ($image)
- **Type / role:** IF (`if`) — checks presence of `image_url`.
- **Condition:** `{{$json.image_url}}` **not empty**
- **Connections:**
  - **True:** → Load $image as binary
  - **False:** → If ($tags)
- **Edge cases / failures:**
  - Non-empty but invalid URL will pass condition and fail in download step.

#### Node: Load $image as binary
- **Type / role:** HTTP Request (`httpRequest`) — downloads the image file.
- **Configuration (interpreted):**
  - URL: `={{ $json.image_url }}`
  - Uses defaults; intended to return binary data.
  - `alwaysOutputData: true` so even on certain failures it may still output something (depends on n8n behavior and HTTP status handling).
- **Connections:** → Upload $image to WP
- **Potential failures / edge cases:**
  - 404/403, hotlink protection, non-image content
  - Large file/timeouts
  - If response is not treated as binary, upload node may fail due to missing binary field

#### Node: Upload $image to WP
- **Type / role:** HTTP Request (`httpRequest`) — uploads binary to WordPress Media endpoint.
- **Configuration (interpreted):**
  - POST to: `{{ $node["Variables"].json.website_URL + "/wp-json/wp/v2/media" }}`
  - Auth: HTTP Basic Auth credential (`n8n-njmarketing`)
  - Body: **binaryData** from input field `data`
  - Headers:
    - `Content-Disposition: attachment; filename="post-{{ $json.id }}.png"`
    - `Content-Type: image/png`
- **Connections:** → If ($tags)
- **Potential failures / edge cases:**
  - WordPress may reject Basic Auth unless Application Passwords are enabled and REST API accessible
  - If actual image is not PNG, forcing `image/png` can lead to incorrect metadata; safer would be detecting MIME type
  - If binary property name isn’t `data`, upload fails

---

### Block 5 — Tags from hashtags (parse → get/create → collect IDs)
**Overview:** If `hashtags` exists, parse it into tag objects (`tag_name`, `tag_slug`), loop over each tag, fetch WP tag ID by slug; if not found create it; store IDs in workflow static data for later post creation.  
**Nodes involved:**  
- If ($tags)
- Parse $tags
- Split Out
- Loop Over Items1
- Get tag_id from WP
- If
- Create new tag
- Store tag_id
- Store tag_id1

#### Node: If ($tags)
- **Type / role:** IF (`if`) — checks if current row has hashtags.
- **Condition:** `={{ $('Loop Over Items').item.json.hashtags }}` not empty
- **Connections:**
  - **True:** → Parse $tags
  - **False:** → Prepare data
- **Edge cases:**
  - Uses `$('Loop Over Items').item.json.hashtags` (tied to the batching node context). If batch size changes or item linking changes, this reference may behave unexpectedly.

#### Node: Parse $tags
- **Type / role:** Code (`code`) — converts hashtags string into structured tag list.
- **Key logic:**
  - `extractTags()`:
    - Prefer regex `#[^\s#,;]+` matches
    - Fallback split on whitespace/`,`/`;`
    - Cleans trailing punctuation
    - Deduplicates by slug (slugify)
  - Output per item:
    - `tags_to_process: [{tag_name, tag_slug}, ...]`
    - `tags: []` (prepared but not used later directly)
- **Important expression detail / risk:**
  - It parses hashtags using `$('Loop Over Items').first().json.hashtags` (explicitly **first item of Loop Over Items**, not the current item). In most “1 item at a time” setups it works, but it is fragile if batch size changes or if multiple items are in play.
- **Connections:** → Split Out
- **Edge cases:**
  - Non-Latin tags may be slugified into empty slugs due to regex allowing only `[a-z0-9\-_]`
  - Duplicate tags are removed by slug

#### Node: Split Out
- **Type / role:** Split Out (`splitOut`) — turns `tags_to_process` array into separate items.
- **Configuration:** `fieldToSplitOut = tags_to_process`, include all other fields.
- **Connections:** → Loop Over Items1
- **Edge cases:** If `tags_to_process` is empty, output may be empty and tag loop won’t run.

#### Node: Loop Over Items1
- **Type / role:** Split in Batches (`splitInBatches`) — iterates tags one by one.
- **Connections:**
  - **First output (loop “done”):** → Prepare data (publishing continues after tag loop)
  - **Second output (each tag item):** → Get tag_id from WP
- **Edge cases:** Similar batch caveats; intended to process one tag at a time.

#### Node: Get tag_id from WP
- **Type / role:** HTTP Request — searches WP tags by slug.
- **Configuration (interpreted):**
  - GET: `.../wp-json/wp/v2/tags?slug={{tag_slug}}&per_page=100`
  - Auth: Basic Auth
  - `alwaysOutputData: true` (so the workflow can continue even if empty)
- **Connections:** → If
- **Edge cases / failures:**
  - WP returns an array; later node checks `$json.id` which may not exist if response is an array (this is a major integration mismatch unless n8n is automatically selecting an element)
  - If WP returns `[]`, there is no `id`

#### Node: If
- **Type / role:** IF — decides whether tag exists.
- **Condition:** `={{ $json.id }}` > 0
- **Connections:**
  - **True:** → Store tag_id1 (tag exists)
  - **False:** → Create new tag (tag missing)
- **Edge cases:**
  - If prior response is an array, `$json.id` is undefined → condition fails → tries to create tag every time.

#### Node: Create new tag
- **Type / role:** HTTP Request — creates WP tag.
- **Configuration:**
  - POST: `.../wp-json/wp/v2/tags`
  - JSON body uses current split-out tag fields:
    - `name`: `{{ $('Split Out').item.json.tags_to_process.tag_name }}`
    - `slug`: `{{ $('Split Out').item.json.tags_to_process.tag_slug }}`
  - Auth: Basic Auth
- **Connections:** → Store tag_id
- **Failures / edge cases:**
  - 409 / term_exists-like behavior if slug already exists (WP may respond with an error and include an existing term ID depending on config)
  - Permissions: user must have capability to manage terms
  - The use of `$('Split Out').item...` assumes the correct item context; safer is using `$json.tags_to_process...` at this point (since each item already has that object).

#### Node: Store tag_id
- **Type / role:** Code — appends created tag ID to static accumulator.
- **Logic:** Reads `$json.id`, validates digits, appends to `data.wp_tag_ids` if not already present.
- **Connections:** → Loop Over Items1 (continue next tag)
- **Edge cases:**
  - If Create new tag returns an error object without `id`, nothing is stored and post will miss tags.

#### Node: Store tag_id1
- **Type / role:** Code — appends existing tag ID to static accumulator.
- **Logic:** Same as Store tag_id (reads `$json.id`).
- **Connections:** → Loop Over Items1
- **Edge cases:** If Get tag_id from WP returned an array, `$json.id` won’t exist → nothing stored.

---

### Block 6 — Prepare payload → publish → update Google Sheet
**Overview:** Builds WP post payload including tag IDs and optional featured media ID, publishes post, then updates the row status and metadata in Google Sheets.  
**Nodes involved:**  
- Prepare data
- HTTP Request
- Update row in sheet

#### Node: Prepare data
- **Type / role:** Code — creates `wp_post_payload` for WP REST API.
- **Key variables and behavior:**
  - Reads `data.wp_tag_ids` from workflow static data, converts to integer array `tags`.
  - Uses current row from `$node["Loop Over Items"].json`
  - Featured image resolution:
    1) If row already has `media_id`, uses it
    2) Else tries to read from `$items("Upload $image to WP")[0].json.id` inside try/catch (safe if upload path wasn’t executed)
  - Payload fields:
    - `title` ← `row.title`
    - `excerpt` ← `row.excerpt`
    - `content` ← `row.body`
    - `status` ← `row.post_status` (note: sheet must contain `post_status`, otherwise status becomes empty string)
    - `tags` ← parsed IDs
    - Adds `featured_media` only when numeric > 0
- **Connections:** → HTTP Request (publish post)
- **Edge cases / failures:**
  - If `post_status` is missing, WordPress may default to `draft` or reject depending on API expectations; empty status is risky
  - Tag IDs may be incomplete due to issues noted in tag lookup logic
  - Static data tag accumulator is not reset per post (only once per run). This means tags from the first post can carry over to the next post unless you reset `wp_tag_ids` **inside the per-row loop**.

#### Node: HTTP Request (publish post)
- **Type / role:** HTTP Request — creates a WP post.
- **Configuration:**
  - POST: `.../wp-json/wp/v2/posts`
  - Body: `={{ $json.wp_post_payload }}`
  - Header: `Content-Type: application/json`
  - Auth: Basic Auth
- **Connections:** → Update row in sheet
- **Failures / edge cases:**
  - 401/403 auth/capabilities issues
  - Validation errors (empty title/content, invalid status, invalid tag IDs)
  - If WP returns an error JSON, Update row in sheet would attempt to read `$json.link`/`$json.date` and may write blanks or error.

#### Node: Update row in sheet
- **Type / role:** Google Sheets — updates the same row (by `id`) after publishing.
- **Configuration (interpreted):**
  - Operation: Update
  - Matching column: `id`
  - Values written:
    - `id`: `={{ $('Prepare data').item.json.id }}`
    - `status`: `POSTED`
    - `post_url`: `={{ $json.link }}`
    - `posted_at`: `={{ $json.date }}`
- **Credentials:** Google Sheets OAuth2
- **Connections:** → Loop Over Items (to continue next row)
- **Edge cases / failures:**
  - If WP response lacks `link` or `date` (error responses), sheet update may write incorrect values
  - If `id` in sheet is not unique, multiple rows may be updated unexpectedly (depends on n8n Sheets node behavior)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start entry point | — | Variables |  |
| Variables | Set | Store WP site URL and Google Sheet URL | When clicking ‘Execute workflow’ | Get row(s) in sheet | ## Inputs & credentials<br>Set WordPress site URL + Google Sheet URL. Connect WP + Google credentials. |
| Get row(s) in sheet | Google Sheets | Read rows where `status=READY` | Variables | Code in JavaScript | ## Fetch READY rows<br>Loads rows where status is READY from the Google Sheet. |
| Code in JavaScript | Code | Reset global tag accumulator (`wp_tag_ids`) | Get row(s) in sheet | Loop Over Items |  |
| Loop Over Items | Split in Batches | Process sheet rows one-by-one | Code in JavaScript | If ($image) (loop items) | ## Process rows<br>Loops through selected rows and publishes them one by one. |
| If ($image) | IF | Decide whether to fetch/upload featured image | Loop Over Items | Load $image as binary (true), If ($tags) (false) | ## Optional featured image<br>If image_url exists: download → upload to WP Media → use as featured image. |
| Load $image as binary | HTTP Request | Download image file to binary | If ($image) | Upload $image to WP | ## Optional featured image<br>If image_url exists: download → upload to WP Media → use as featured image. |
| Upload $image to WP | HTTP Request | Upload image to WP Media endpoint | Load $image as binary | If ($tags) | ## Optional featured image<br>If image_url exists: download → upload to WP Media → use as featured image. |
| If ($tags) | IF | Decide whether to parse/process hashtags | If ($image) / Upload $image to WP | Parse $tags (true), Prepare data (false) | ## Tags from hashtags<br>Parse hashtags → get/create WP tags → collect tag IDs for the post. |
| Parse $tags | Code | Parse hashtags into `{tag_name, tag_slug}` array | If ($tags) | Split Out | ## Tags from hashtags<br>Parse hashtags → get/create WP tags → collect tag IDs for the post. |
| Split Out | Split Out | Split `tags_to_process[]` into individual items | Parse $tags | Loop Over Items1 | ## Tags from hashtags<br>Parse hashtags → get/create WP tags → collect tag IDs for the post. |
| Loop Over Items1 | Split in Batches | Loop through tags one-by-one; then continue | Split Out | Get tag_id from WP (per tag), Prepare data (after loop) | ## Tags from hashtags<br>Parse hashtags → get/create WP tags → collect tag IDs for the post. |
| Get tag_id from WP | HTTP Request | Query WP tags by slug | Loop Over Items1 | If | ## Get tag ids<br>select tag ids from WP |
| If | IF | If tag exists then store ID else create tag | Get tag_id from WP | Store tag_id1 (true), Create new tag (false) | ## Create new ones<br>create new tags & get their ids from WP if necessary |
| Create new tag | HTTP Request | Create missing WP tag | If | Store tag_id | ## Create new ones<br>create new tags & get their ids from WP if necessary |
| Store tag_id | Code | Append created tag ID to global accumulator | Create new tag | Loop Over Items1 | ## Tags from hashtags<br>Parse hashtags → get/create WP tags → collect tag IDs for the post. |
| Store tag_id1 | Code | Append existing tag ID to global accumulator | If | Loop Over Items1 | ## Tags from hashtags<br>Parse hashtags → get/create WP tags → collect tag IDs for the post. |
| Prepare data | Code | Build `wp_post_payload` incl. tags + featured_media | If ($tags) / Loop Over Items1 | HTTP Request | ## Publish + update sheet<br>Create WP post → write POSTED + URL back to the same row. |
| HTTP Request | HTTP Request | Publish WP post (`/wp/v2/posts`) | Prepare data | Update row in sheet | ## Publish + update sheet<br>Create WP post → write POSTED + URL back to the same row. |
| Update row in sheet | Google Sheets | Mark row POSTED + write `post_url`, `posted_at` | HTTP Request | Loop Over Items | ## Publish + update sheet<br>Create WP post → write POSTED + URL back to the same row. |
| Sticky Note | Sticky Note | Comment | — | — | ## Fetch READY rows<br>Loads rows where status is READY from the Google Sheet. |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Process rows<br>Loops through selected rows and publishes them one by one. |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Optional featured image<br>If image_url exists: download → upload to WP Media → use as featured image. |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Tags from hashtags<br>Parse hashtags → get/create WP tags → collect tag IDs for the post. |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Publish + update sheet<br>Create WP post → write POSTED + URL back to the same row. |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Inputs & credentials<br>Set WordPress site URL + Google Sheet URL. Connect WP + Google credentials. |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Video Walkthrough<br>@[youtube](W3xQ1t4irqc) |
| Sticky Note7 | Sticky Note | Comment | — | — | ## Create new ones<br>create new tags & get their ids from WP if necessary |
| Sticky Note8 | Sticky Note | Comment | — | — | ## Get tag ids<br>select tag ids from WP |
| Sticky Note10 | Sticky Note | Comment | — | — | ## Google Sheets → WordPress Publisher<br><br>**How it works**<br>...<br>Use this template to organise your future posts https://docs.google.com/spreadsheets/d/1lJDeaN75c1hBk0gddsdvbeXDuxDr5Q-NablHRSjvLjQ/edit?gid=0#gid=0 |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n. Keep execution order default (`v1`).

2) **Add Manual Trigger**
   - Node: *Manual Trigger*
   - Name: `When clicking ‘Execute workflow’`

3) **Add Variables (Set)**
   - Node: *Set*
   - Name: `Variables`
   - Add fields:
     - `website_URL` (string) → your WP site base URL, e.g. `https://example.com`
     - `google_sheet_URL` (string) → full Google Sheets URL including `#gid=...`
   - Connect: Manual Trigger → Variables

4) **Add Google Sheets: Get rows**
   - Node: *Google Sheets*
   - Name: `Get row(s) in sheet`
   - Operation: Read / Get Many (the node UI will show the “get rows” style operation)
   - Document: select your spreadsheet (OAuth2 credential required)
   - Sheet: set **by URL** using expression: `={{ $json.google_sheet_URL }}`
   - Filter:
     - Column: `status`
     - Value: `READY`
   - Credentials: **Google Sheets OAuth2**
   - Connect: Variables → Get row(s) in sheet

5) **Add Code node to reset tag accumulator**
   - Node: *Code* (JavaScript)
   - Name: `Code in JavaScript`
   - Code: set workflow static data global value `wp_tag_ids` to `'0'`
   - Connect: Get row(s) in sheet → Code in JavaScript

6) **Add Split In Batches for rows**
   - Node: *Split in Batches*
   - Name: `Loop Over Items`
   - Batch size: 1 (recommended)
   - Connect: Code in JavaScript → Loop Over Items

7) **Add IF for image_url**
   - Node: *IF*
   - Name: `If ($image)`
   - Condition: String → `notEmpty` with left value `={{ $json.image_url }}`
   - Connect: Loop Over Items → If ($image)

8) **Add HTTP Request to download the image**
   - Node: *HTTP Request*
   - Name: `Load $image as binary`
   - URL: `={{ $json.image_url }}`
   - Configure response to be binary (in node options, enable “Download” / “Response: File” depending on your n8n version), ensure it outputs binary property (commonly `data`)
   - Connect: If ($image) (true) → Load $image as binary

9) **Add HTTP Request to upload media to WordPress**
   - Node: *HTTP Request*
   - Name: `Upload $image to WP`
   - Method: POST
   - URL: `={{ $node["Variables"].json.website_URL + "/wp-json/wp/v2/media" }}`
   - Authentication: **Basic Auth** (generic credential type → HTTP Basic Auth)
     - Use a WP user + **Application Password**
   - Body content type: Binary Data
   - Binary field name: `data`
   - Headers:
     - `Content-Disposition: attachment; filename="post-{{ $json.id }}.png"`
     - `Content-Type: image/png`
   - Connect: Load $image as binary → Upload $image to WP
   - Connect: Upload $image to WP → If ($tags) (next step)

10) **Add IF for hashtags**
   - Node: *IF*
   - Name: `If ($tags)`
   - Condition: String → `notEmpty`
   - Left value: `={{ $('Loop Over Items').item.json.hashtags }}`
   - Connect: If ($image) (false) → If ($tags)
   - Connect: Upload $image to WP → If ($tags)

11) **Add Code node to parse hashtags**
   - Node: *Code*
   - Name: `Parse $tags`
   - Implement:
     - `slugify`
     - `extractTags`
     - Output `tags_to_process` array and `tags: []`
   - Connect: If ($tags) (true) → Parse $tags

12) **Add Split Out for tags_to_process**
   - Node: *Split Out*
   - Name: `Split Out`
   - Field to split out: `tags_to_process`
   - Include: all other fields
   - Connect: Parse $tags → Split Out

13) **Add Split In Batches for tags**
   - Node: *Split in Batches*
   - Name: `Loop Over Items1`
   - Batch size: 1
   - Connect: Split Out → Loop Over Items1

14) **Add HTTP Request: get tag by slug**
   - Node: *HTTP Request*
   - Name: `Get tag_id from WP`
   - Method: GET
   - URL: `={{ $node["Variables"].json.website_URL + "/wp-json/wp/v2/tags?slug=" + $json.tags_to_process.tag_slug + "&per_page=100" }}`
   - Auth: Basic Auth (same WP credential)
   - Connect: Loop Over Items1 (item output) → Get tag_id from WP

15) **Add IF to decide create vs store**
   - Node: *IF*
   - Name: `If`
   - Condition: Number → `gt`
   - Left: `={{ $json.id }}`
   - Right: `0`
   - Connect: Get tag_id from WP → If

16) **Add HTTP Request: create tag**
   - Node: *HTTP Request*
   - Name: `Create new tag`
   - Method: POST
   - URL: `={{ $node["Variables"].json.website_URL + "/wp-json/wp/v2/tags" }}`
   - JSON body: `{ "name": ..., "slug": ... }` (from current tag item)
   - Auth: Basic Auth
   - Connect: If (false) → Create new tag

17) **Add Code nodes to store tag IDs**
   - Node: *Code*
   - Name: `Store tag_id1` (existing tag path)
   - Append `$json.id` to static `data.wp_tag_ids` with deduplication
   - Connect: If (true) → Store tag_id1 → Loop Over Items1

   - Node: *Code*
   - Name: `Store tag_id` (created tag path)
   - Same logic
   - Connect: Create new tag → Store tag_id → Loop Over Items1

18) **Add Code node: prepare WP payload**
   - Node: *Code*
   - Name: `Prepare data`
   - Build `wp_post_payload` from current row:
     - `title`, `excerpt`, `content`, `status` (from `post_status`)
     - `tags` array from `data.wp_tag_ids`
     - optionally `featured_media` from upload response id
   - Connect:
     - If ($tags) (false) → Prepare data
     - Loop Over Items1 (done output) → Prepare data

19) **Add HTTP Request: publish post**
   - Node: *HTTP Request*
   - Name: `HTTP Request`
   - Method: POST
   - URL: `={{ $node["Variables"].json.website_URL + "/wp-json/wp/v2/posts" }}`
   - Body: JSON = `={{ $json.wp_post_payload }}`
   - Header: `Content-Type: application/json`
   - Auth: Basic Auth
   - Connect: Prepare data → HTTP Request

20) **Add Google Sheets: update row**
   - Node: *Google Sheets*
   - Name: `Update row in sheet`
   - Operation: Update
   - Match column: `id`
   - Values:
     - `id`: `={{ $('Prepare data').item.json.id }}`
     - `status`: `POSTED`
     - `post_url`: `={{ $json.link }}`
     - `posted_at`: `={{ $json.date }}`
   - Credentials: Google Sheets OAuth2
   - Connect: HTTP Request → Update row in sheet

21) **Close the main loop**
   - Connect: Update row in sheet → Loop Over Items (so it fetches the next row)

22) **Credentials**
   - **Google Sheets OAuth2:** must have access to the spreadsheet.
   - **WordPress Basic Auth:** typically WordPress Application Passwords:
     - Username: WP user
     - Password: application password (not the normal login password)
   - Ensure WP REST API is reachable from n8n (no firewall/WAF blocks).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video Walkthrough: @[youtube](W3xQ1t4irqc) | Embedded in workflow sticky note |
| Spreadsheet template for organizing posts | https://docs.google.com/spreadsheets/d/1lJDeaN75c1hBk0gddsdvbeXDuxDr5Q-NablHRSjvLjQ/edit?gid=0#gid=0 |
| Workflow description (from sticky note): reads READY rows → publishes to WP; optional image upload; parses hashtags into tags; updates row to POSTED with URL/date | Documented in “Google Sheets → WordPress Publisher” note |

--- 

**Important implementation risks to anticipate (based on the current node logic):**
- **Tag lookup response shape:** WP `GET /tags?slug=...` usually returns an **array**; the workflow checks `$json.id` as if it were an object. This can cause unnecessary “create tag” attempts and missing tag IDs.
- **Tag ID accumulator is global per run, not per post:** `wp_tag_ids` is reset once before processing all rows, so tags from earlier posts can carry over to later posts unless you reset it inside the per-row loop.
- **`post_status` field dependency:** the payload uses `row.post_status`; if the sheet doesn’t contain it, WP may receive an empty status.