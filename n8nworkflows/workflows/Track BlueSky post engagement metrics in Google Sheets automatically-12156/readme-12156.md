Track BlueSky post engagement metrics in Google Sheets automatically

https://n8nworkflows.xyz/workflows/track-bluesky-post-engagement-metrics-in-google-sheets-automatically-12156


# Track BlueSky post engagement metrics in Google Sheets automatically

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically refresh engagement metrics (Likes, Reposts, Replies) for recently published BlueSky posts and write them back into the same Google Sheet used for scheduling/content tracking.

**Target use cases:**
- Social/content teams maintaining a posting calendar in Google Sheets and wanting daily analytics updates.
- Solo creators tracking post performance without manual BlueSky checks.
- Lightweight “active window” analytics: only posts from the last 14 days are updated to reduce API calls.

**Logical blocks**
1.1 **Scheduled Start & Configuration** → runs daily and loads BlueSky credentials + timezone.  
1.2 **BlueSky Authentication** → creates a session and retrieves an access token + DID.  
1.3 **Fetch Posted Rows from Google Sheets** → pulls only rows where `Status = "Posted"`.  
1.4 **Gatekeeping / Active Window Filter** → ensures required fields exist and `Posted At` is within last 14 days.  
1.5 **Batch Processing (one row at a time)** → iterates through eligible rows safely.  
1.6 **Fetch BlueSky Post Metrics** → calls BlueSky API for stats (continue on error).  
1.7 **Update Google Sheet with Safe Defaults** → writes counts back, using `|| 0` fallbacks to avoid failures.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Start & Configuration

**Overview:** Starts the workflow on a daily schedule and sets required runtime variables (handle, app password, timezone).

**Nodes involved:**  
- Schedule Trigger  
- Configuration

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — workflow entrypoint.
- **Config choices:** Runs daily at **09:00** (server/instance schedule context; the node is configured with `triggerAtHour: 9`).
- **Connections:** Outputs to **Configuration**.
- **Edge cases / failures:**
  - If n8n instance timezone differs from desired timezone, the trigger fires at instance time; the *filtering logic* later uses the configured timezone, but the trigger time itself is instance-based.

#### Node: Configuration
- **Type / role:** `Set` — centralizes user-provided settings.
- **Config choices (interpreted):**
  - `bluesky_handle` (string): must be set (e.g., `steve.bsky.social`).
  - `app_password` (string): BlueSky App Password (not the main account password).
  - `timezone` (string): defaults to `Asia/Kolkata`. Used later for date parsing and the “last 14 days” window.
- **Connections:** Outputs to **BlueSky Auth**.
- **Edge cases / failures:**
  - Empty handle/password → authentication fails downstream (401/400).
  - Invalid timezone name → date parsing/filtering may error or filter everything out depending on Luxon behavior.

**Sticky note coverage (context):**
- “START HERE… handle, App Password and timezone… timezone list link.”

---

### 2.2 BlueSky Authentication

**Overview:** Authenticates against BlueSky to obtain an `accessJwt` (Bearer token) and the user’s `did`, used to build post URIs and authorize subsequent requests.

**Nodes involved:**  
- BlueSky Auth

#### Node: BlueSky Auth
- **Type / role:** `HTTP Request` — calls BlueSky ATProto session endpoint.
- **Config choices:**
  - `POST https://bsky.social/xrpc/com.atproto.server.createSession`
  - JSON body:
    - `identifier`: `$('Configuration').first().json.bluesky_handle`
    - `password`: `$('Configuration').first().json.app_password`
- **Key variables/expressions:**
  - `{{$('Configuration').first().json.bluesky_handle}}`
  - `{{ $('Configuration').first().json.app_password }}`
- **Outputs:** Session payload (typically includes `accessJwt`, `did`, etc.).
- **Connections:** Outputs to **Get row(s) in sheet**.
- **Edge cases / failures:**
  - 401/403 if app password invalid, revoked, or handle incorrect.
  - Rate limiting/network errors: would stop the workflow (no “continue on error” set here).
- **Version notes:** Uses `HTTP Request` node v4.3 features (body as JSON, expression support).

**Sticky note coverage (context):**
- “Get access token… used for all subsequent API calls.”

---

### 2.3 Fetch Posted Rows from Google Sheets

**Overview:** Pulls the set of rows that represent published posts (`Status = Posted`) from a Google Sheet.

**Nodes involved:**  
- Get row(s) in sheet

#### Node: Get row(s) in sheet
- **Type / role:** `Google Sheets` — read operation with filter.
- **Config choices:**
  - Operation: “Get row(s)” (read rows)
  - Filter: `Status` column equals `Posted`
  - `documentId` and `sheetName`: selected from the credential-linked list (left blank in the JSON template; must be chosen during setup).
- **Credentials:** Google Sheets OAuth2 credential (`googleSheetsOAuth2Api`).
- **Connections:** Outputs to **Filter1**.
- **Edge cases / failures:**
  - Wrong sheet/document selected → no rows returned or schema mismatch.
  - Column name mismatch (`Status`) → filter may return none or error depending on node behavior.
  - OAuth token expired/insufficient permissions → auth failure.
- **Version notes:** Node v4.7 (Google Sheets node behavior differs significantly from earlier versions; matching columns and schema mapping are v4-style concepts).

**Sticky note coverage (context):**
- “Fetches all rows where Status is 'Posted'…”

---

### 2.4 Gatekeeping / Active Window Filter (14 days)

**Overview:** Protects the workflow from blank/invalid rows and reduces API usage by only allowing posts from the last 14 days (based on `Posted At`).

**Nodes involved:**  
- Filter1

#### Node: Filter1
- **Type / role:** `Filter` — row eligibility gate (“The Gatekeeper”).
- **Config choices (AND conditions):**
  1. `Post Link` is **not empty**
  2. `Posted At` is **not empty**
  3. `Posted At` is **after** (now minus 14 days), using configured timezone
- **Key expressions:**
  - Post Link check: `={{ $json['Post Link'] }}`
  - Posted At check: `={{ $json['Posted At'] }}`
  - Date parsing:
    - `DateTime.fromFormat($json['Posted At'], 'yyyy-MM-dd HH:mm', { zone: $('Configuration').first().json.timezone })`
  - Window start:
    - `{{ $now.setZone($('Configuration').first().json.timezone || 'UTC').minus({days: 14}) }}`
- **Connections:** Outputs passed items to **Loop Over Items**.
- **Edge cases / failures:**
  - If `Posted At` format isn’t exactly `yyyy-MM-dd HH:mm`, parsing can fail → condition may evaluate unexpectedly or error.
  - If timezone invalid, Luxon may default or mark invalid; could filter everything out.
  - Rows older than 14 days are silently dropped (expected behavior).
- **Version notes:** Filter node v2.2, condition builder “version: 2” semantics.

**Sticky note coverage (context):**
- “Checks that 'Posted At' and 'Post Link' are not empty… only allows posts within the last 14 days.”

---

### 2.5 Batch Processing (one row at a time)

**Overview:** Iterates through filtered rows sequentially so one failing post does not prevent processing the remaining posts.

**Nodes involved:**  
- Loop Over Items (Split in Batches)

#### Node: Loop Over Items
- **Type / role:** `Split in Batches` — controls iteration/batching.
- **Config choices:** Batch size not explicitly set (defaults apply). Used in “loop mode” by feeding the end of the chain back into the node.
- **Connections:**
  - **Input:** from **Filter1**
  - **Output (index 1):** to **Get Post Stats**
  - **Output (index 0):** not used (empty in connections)
  - **Loop-back:** **Update row in sheet** → **Loop Over Items** (continues next item)
- **Key variables used by other nodes:**
  - `$('Loop Over Items').item.json[...]` is referenced downstream to access the current row.
- **Edge cases / failures:**
  - If no items pass Filter1, nothing runs after this node (normal).
  - If batch settings are changed incorrectly, you may skip items or loop improperly.
- **Version notes:** SplitInBatches v3.

**Sticky note coverage (context):**
- “Batch Processor… processes them one by one… if one post fails it doesn't stop the entire workflow…”

---

### 2.6 Fetch BlueSky Post Metrics (resilient)

**Overview:** For each row, calls BlueSky `getPosts` to retrieve engagement counts. Configured to continue even if the API errors (e.g., deleted post).

**Nodes involved:**  
- Get Post Stats

#### Node: Get Post Stats
- **Type / role:** `HTTP Request` — fetch post analytics.
- **Config choices:**
  - `GET https://bsky.social/xrpc/app.bsky.feed.getPosts`
  - Query parameter `uris` is dynamically constructed as an AT URI.
  - Header `Authorization: Bearer <accessJwt>`
  - **onError:** `continueRegularOutput` (critical resilience feature)
- **Key expressions:**
  - Authorization header:
    - `=Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
  - URI construction:
    - `=at://{{ $('BlueSky Auth').first().json.did }}/app.bsky.feed.post/{{ $('Loop Over Items').item.json["Post Link"].split("/post/")[1] }}`
    - Assumes the Sheet’s `Post Link` contains `/post/<rkey>` and extracts the `<rkey>` part.
- **Connections:** Outputs to **Update row in sheet**.
- **Edge cases / failures:**
  - If `Post Link` does not contain `/post/`, `.split("/post/")[1]` becomes `undefined` → malformed URI → API error.
  - Deleted post or invalid rkey → likely 400/404. Because of “continue on error”, node outputs an error-like JSON, and the workflow continues.
  - Token expired (accessJwt) → 401; would also be continued, leading to zeros being written (see updater) unless you add explicit detection.
- **Version notes:** HTTP Request node v4.3.

**Sticky note coverage (context):**
- “Resilient Fetcher… Continue on Error… deleted post returns error JSON instead of crashing.”

---

### 2.7 Update Google Sheet with Safe Defaults

**Overview:** Writes Like/Repost/Reply counts into the matching row using `row_number` as the key. Uses safe fallback logic to write `0` when metrics are missing (e.g., deleted post or API error).

**Nodes involved:**  
- Update row in sheet

#### Node: Update row in sheet
- **Type / role:** `Google Sheets` — update operation.
- **Config choices:**
  - Operation: `update`
  - Matching column: `row_number` (read-only column provided by the Google Sheets node result set)
  - Writes to columns:
    - `Like Count`
    - `Repost Count`
    - `Reply Count`
  - `documentId` and `sheetName`: must be selected.
  - **onError:** `continueRegularOutput` (ensures loop continues even if an update fails)
- **Key expressions:**
  - `row_number`: `={{ $('Loop Over Items').item.json.row_number }}`
  - Likes: `={{ $json?.posts[0]?.likeCount || 0 }}`
  - Reposts: `={{ $json?.posts[0]?.repostCount || 0 }}`
  - Replies: `={{ $json?.posts[0]?.replyCount || 0 }}`
- **Connections:**
  - **Input:** from **Get Post Stats**
  - **Output:** to **Loop Over Items** (loop continuation)
- **Edge cases / failures:**
  - If your sheet doesn’t have columns named exactly `Like Count`, `Repost Count`, `Reply Count`, update may fail or silently not map as expected.
  - If `row_number` isn’t present (e.g., you used a different read method), matching fails.
  - API quotas / Google permissions issues cause update failures; with continue-on-error, it will proceed to next items (but you may miss updates).
- **Version notes:** Google Sheets node v4.7; relies on v4 `row_number` matching behavior.

**Sticky note coverage (context):**
- “Safe Updater… uses (|| 0)… writes 0 into stats columns instead of breaking.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Daily start signal | — | Configuration | # 📈 Analytics Bot - How To Use… Sample Google Sheet link… Active Window 14 days… runs daily 9 AM |
| Configuration | Set | Stores handle/app password/timezone | Schedule Trigger | BlueSky Auth | ### 1- START HERE … timezone link: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones |
| BlueSky Auth | HTTP Request | Creates BlueSky session, returns token + DID | Configuration | Get row(s) in sheet | ### 2- Get access token… used for subsequent API calls. |
| Get row(s) in sheet | Google Sheets | Fetch rows where Status=Posted | BlueSky Auth | Filter1 | ### 3- Google sheets rows… Status is 'Posted'… |
| Filter1 | Filter | Validates fields + keeps last-14-days posts | Get row(s) in sheet | Loop Over Items | ### 4- The Gatekeeper… checks Posted At/Post Link not empty + within 14 days. |
| Loop Over Items | Split in Batches | Iterates eligible rows sequentially | Filter1; Update row in sheet (loop-back) | Get Post Stats (output index 1) | ### 5- The Batch Processor… processes one by one… failures don’t stop the run. |
| Get Post Stats | HTTP Request | Fetch metrics for one post (resilient) | Loop Over Items | Update row in sheet | ### 6- Resilient Fetcher… Continue on Error… deleted post returns error JSON. |
| Update row in sheet | Google Sheets | Writes metrics back using row_number match | Get Post Stats | Loop Over Items | ### 7- Safe Updater… (|| 0) fallback… writes 0 on 404. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Track BlueSky post engagement metrics in Google Sheets automatically* (or your preferred name).

2) **Add “Schedule Trigger”**
- Node: **Schedule Trigger**
- Set it to run **daily at 09:00** (triggerAtHour = 9).
- Connect **Schedule Trigger → Configuration**.

3) **Add “Configuration” (Set node)**
- Node: **Set**
- Add fields:
  - `bluesky_handle` (string) → your handle (e.g., `steve.bsky.social`)
  - `app_password` (string) → your BlueSky App Password
  - `timezone` (string) → e.g., `America/Los_Angeles` (tz database name)
- Connect **Configuration → BlueSky Auth**.

4) **Add “BlueSky Auth” (HTTP Request)**
- Node: **HTTP Request**
- Method: **POST**
- URL: `https://bsky.social/xrpc/com.atproto.server.createSession`
- Body type: **JSON**
- JSON body (with expressions):
  - `identifier`: `{{$('Configuration').first().json.bluesky_handle}}`
  - `password`: `{{ $('Configuration').first().json.app_password }}`
- Connect **BlueSky Auth → Get row(s) in sheet**.

5) **Add Google Sheets credential**
- Create/choose **Google Sheets OAuth2** credential with access to the target spreadsheet.

6) **Add “Get row(s) in sheet” (Google Sheets)**
- Node: **Google Sheets**
- Operation: **Get row(s)** (read)
- Select:
  - **Document** (Spreadsheet)
  - **Sheet**
- Add filter:
  - Lookup column: `Status`
  - Lookup value: `Posted`
- Connect **Get row(s) in sheet → Filter1**.

7) **Add “Filter1” (Filter node)**
- Node: **Filter**
- Use **AND** conditions:
  1. `Post Link` → **not empty**
     - Left value: `={{ $json['Post Link'] }}`
  2. `Posted At` → **not empty**
     - Left value: `={{ $json['Posted At'] }}`
  3. Date condition: Parsed `Posted At` **after** `now - 14 days`
     - Left value:
       - `={{ DateTime.fromFormat($json['Posted At'], 'yyyy-MM-dd HH:mm', { zone: $('Configuration').first().json.timezone }) }}`
     - Right value:
       - `={{ $now.setZone($('Configuration').first().json.timezone || 'UTC').minus({days: 14}) }}`
- Connect **Filter1 → Loop Over Items**.

8) **Add “Loop Over Items” (Split in Batches)**
- Node: **Split in Batches**
- Keep defaults (or set a batch size if desired).
- Connect **Loop Over Items (output 1 / “Next batch/item”) → Get Post Stats**.
- Later you will connect **Update row in sheet → Loop Over Items** to complete the loop.

9) **Add “Get Post Stats” (HTTP Request)**
- Node: **HTTP Request**
- Method: **GET**
- URL: `https://bsky.social/xrpc/app.bsky.feed.getPosts`
- Enable **Send Query Parameters**:
  - `uris` value:
    - `=at://{{ $('BlueSky Auth').first().json.did }}/app.bsky.feed.post/{{ $('Loop Over Items').item.json["Post Link"].split("/post/")[1] }}`
- Enable **Send Headers**:
  - `Authorization`:
    - `=Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
- Set **Error Handling**: **Continue on Error**.
- Connect **Get Post Stats → Update row in sheet**.

10) **Add “Update row in sheet” (Google Sheets update)**
- Node: **Google Sheets**
- Operation: **Update**
- Select the same **Document** and **Sheet** as step 6.
- Set **Matching column**: `row_number`
- Map fields to update:
  - `row_number`: `={{ $('Loop Over Items').item.json.row_number }}`
  - `Like Count`: `={{ $json?.posts[0]?.likeCount || 0 }}`
  - `Repost Count`: `={{ $json?.posts[0]?.repostCount || 0 }}`
  - `Reply Count`: `={{ $json?.posts[0]?.replyCount || 0 }}`
- Set **Error Handling**: **Continue on Error**.
- Connect **Update row in sheet → Loop Over Items** (this enables processing the next item).

11) **Validate your Google Sheet columns**
- Must include at least:
  - `Status`, `Posted At`, `Post Link`, plus `Like Count`, `Repost Count`, `Reply Count`.
- Ensure `Posted At` matches format: `yyyy-MM-dd HH:mm`.

12) **Activate the workflow**
- Turn it on. It will run daily and update metrics for eligible rows (last 14 days only).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample Google Sheet (required columns reference) | https://docs.google.com/spreadsheets/d/1Mg04gK1K5DBtJHrWw3ePRFc_JjkxwAp0deGjapVl2q0/edit?usp=sharing |
| Timezone name list (tz database) | https://en.wikipedia.org/wiki/List_of_tz_database_time_zones |
| “Active Window” strategy: only last 14 days updated to reduce API calls | Mentioned in workflow notes (posts older than 14 days effectively treated as archived) |
| Resilience design: HTTP metrics fetch and Google update both continue on error | Prevents one deleted/broken post from stopping the whole run; may write zeros if API result is missing |

If you share the exact Google Sheet column set (including whether you have “Posted At” vs “PostedAt”, etc.), I can point out any naming/format mismatches that would prevent updates.