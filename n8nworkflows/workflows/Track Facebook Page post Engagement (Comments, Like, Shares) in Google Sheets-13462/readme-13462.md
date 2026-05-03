Track Facebook Page post Engagement (Comments, Like, Shares) in Google Sheets

https://n8nworkflows.xyz/workflows/track-facebook-page-post-engagement--comments--like--shares--in-google-sheets-13462


# Track Facebook Page post Engagement (Comments, Like, Shares) in Google Sheets

## 1. Workflow Overview

**Workflow name:** Facebook Page Posts Engagement Tracker  
**Purpose:** Collect engagement metrics (**comments count, likes/reactions count, shares count**) for a configurable number of recent posts from a Facebook Page and store/update those metrics in **Google Sheets**, keyed by **POST ID**.

**Target use cases:** social media reporting, lightweight analytics, recurring client reporting, tracking engagement trends without external analytics suites.

### Logical blocks
**1.1 Manual start + configuration**  
Manual trigger and a parameter that controls how many posts to analyze.

**1.2 Facebook Page + feed retrieval**  
Fetches the Page context (`me`) and then pulls the Page feed with a `limit`.

**1.3 Feed normalization (split into per-post items)**  
Splits the feed `data[]` array into single items so each post can be processed.

**1.4 Parallel engagement collection + Google Sheets upsert (3 branches)**  
Three parallel branches loop through posts to fetch:
- comments (with summary count) → upsert COMMENTS (+ POST text)  
- reactions/likes (with summary count) → upsert LIKES  
- shares (field `shares.count`) → upsert SHARES  

Each branch includes a **Wait** step after writing to Sheets to reduce rate-limit risk and to pace the loop.

**1.5 Documentation / notes (sticky notes)**  
In-workflow guidance and external links (Facebook token, YouTube channel).

---

## 2. Block-by-Block Analysis

### 2.1 Manual start + configuration
**Overview:** Starts the workflow manually and defines `max_post`, which is later used as the Facebook feed `limit`.

**Nodes involved:**  
- When clicking ‘Execute workflow’  
- Set Max Posts

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`manualTrigger`) — entry point.
- **Configuration choices:** No parameters; runs only when executed manually.
- **Inputs/outputs:** No inputs. Output goes to **Set Max Posts**.
- **Edge cases:** None (except human execution dependency).

#### Node: Set Max Posts
- **Type / role:** Set node — defines a workflow variable.
- **Configuration choices:** Sets field `max_post` to string `"3"`.
  - Used downstream to limit feed results.
- **Key expressions/variables:** Produces `$json.max_post`.
- **Inputs/outputs:** Input from Manual Trigger; output to **Get Page Info**.
- **Edge cases / failures:**
  - `max_post` is a **string**. Facebook `limit` typically accepts numeric strings, but if changed to non-numeric the API can error or ignore the parameter.

**Sticky note coverage (context):**
- “STEP 1 - Set Max Posts: Set number max posts to analyze”

---

### 2.2 Facebook Page + feed retrieval
**Overview:** Uses the Facebook Graph API credential to fetch the current Page identity (`me`) and then pulls recent feed posts with a configurable `limit`.

**Nodes involved:**  
- Get Page Info  
- Get Page Feed

#### Node: Get Page Info
- **Type / role:** Facebook Graph API node — retrieves the authenticated entity info.
- **Configuration choices:**
  - **Node/endpoint:** `me`
  - **Graph API Version:** `v21.0`
  - Uses configured Facebook Graph API credential.
- **Key outputs:** Expects `id` in output (Page ID or user/page context depending on token).
- **Inputs/outputs:** Input from **Set Max Posts**; output to **Get Page Feed**.
- **Edge cases / failures:**
  - Token does not represent a Page or lacks permissions → may return user ID instead of Page, or fail.
  - Expired/temporary token → auth error.
  - Graph API permission issues for reading feed (e.g., missing `pages_read_engagement`, `pages_read_user_content` depending on API requirements).

#### Node: Get Page Feed
- **Type / role:** Facebook Graph API node — retrieves posts from the feed.
- **Configuration choices:**
  - **Node/endpoint:** `={{ $json.id }}/feed` (uses Page ID from previous node)
  - **Query parameter:** `limit = {{ $('Set Max Posts').item.json.max_post }}`
  - **Graph API Version:** `v21.0`
- **Inputs/outputs:** Input from **Get Page Info**; output to **Split Out**.
- **Edge cases / failures:**
  - If `Get Page Info` returns an ID that is not a Page accessible by token → feed call fails.
  - If feed returns no `data` array, downstream `Split Out` will produce no items.

**Sticky note coverage (context):**
- “STEP 2 - Obtain Access Token: Get a temporary access token for your Facebook Page”  
  Link: https://developers.facebook.com/tools/explorer/

---

### 2.3 Feed normalization (split into per-post items)
**Overview:** Converts the feed response containing `data[]` into a stream of individual post items for parallel processing.

**Nodes involved:**  
- Split Out

#### Node: Split Out
- **Type / role:** Split Out (`splitOut`) — item normalization.
- **Configuration choices:**
  - **Field to split:** `data`
- **Inputs/outputs:** Input from **Get Page Feed**; outputs to all three loops:
  - **Loop Over Items** (comments)
  - **Loop Over Items1** (likes)
  - **Loop Over Items2** (shares)
- **Edge cases / failures:**
  - If `data` is missing or not an array, the node may output zero items or error depending on the exact payload.

---

### 2.4 Engagement collection + Google Sheets upsert (Comments branch)
**Overview:** Iterates over each post, fetches comments (with a summary count), then writes/updates the Google Sheet row keyed by POST ID. Includes a wait step before continuing to the next item.

**Nodes involved:**  
- Loop Over Items  
- Get comments  
- Add n. comments  
- Wait

#### Node: Loop Over Items
- **Type / role:** Split in Batches (`splitInBatches`, v3) — loop controller.
- **Configuration choices:** Default options; batch size not explicitly set (n8n defaults apply).
- **Connections (important):**
  - **Second output (loop body)** → **Get comments**
  - **First output (done)** is unused (empty).
  - After **Wait**, it routes back into **Loop Over Items** to continue the loop.
- **Edge cases / failures:**
  - Large feeds + API calls can be slow; without explicit batch size control you may want to set it to `1` for predictable pacing.

#### Node: Get comments
- **Type / role:** Facebook Graph API — fetch post comments edge.
- **Configuration choices:**
  - **Node:** `={{ $json.id }}` (current post ID)
  - **Edge:** `comments`
  - **Fields:** `id,from,message,created_time,comments`
  - **Query parameters:**
    - `order=reverse_chronological`
    - `summary=true` (required to get `summary.total_count`)
    - `filter=stream`
  - **Graph API Version:** `v21.0`
- **Key outputs/variables:** Uses `$json.summary.total_count` downstream.
- **Inputs/outputs:** Input from loop; output to **Add n. comments**.
- **Edge cases / failures:**
  - Permission restrictions can hide comments or cause errors.
  - Facebook may paginate comments; summary count should still be present with `summary=true`, but structure can vary.
  - Some posts may have comments disabled → summary may exist with 0, or edge call may behave differently.

#### Node: Add n. comments
- **Type / role:** Google Sheets — **appendOrUpdate** (upsert) engagement data.
- **Configuration choices:**
  - **Operation:** Append or Update
  - **Document:** “Count Facebook Engagement” (Spreadsheet ID `1tZgsygFZ4xXNJ-CYWzib9zlaw6ZH2AHS8JoAjlvY65A`)
  - **Sheet:** “Foglio1” (`gid=0`)
  - **Matching column(s):** `POST ID` (unique key)
  - **Mapped values:**
    - `POST ID = {{ $('Loop Over Items').item.json.id }}`
    - `POST = {{ $('Loop Over Items').item.json.message }}`
    - `COMMENTS = {{ $json.summary.total_count }}`
- **Inputs/outputs:** Input from **Get comments**; output to **Wait**.
- **Edge cases / failures:**
  - If the post has no `message`, the `POST` cell may be blank (common for shared posts, media-only posts).
  - If sheet headers don’t exactly match (`POST ID`, `POST`, `LIKES`, `COMMENTS`, `SHARES`) the node may fail mapping.
  - OAuth token expired/revoked → auth error.
  - Concurrency: since three branches write to the same sheet keyed by `POST ID`, updates are generally safe, but timing can still cause transient write conflicts.

#### Node: Wait
- **Type / role:** Wait node — throttling / pacing.
- **Configuration choices:** Default wait behavior; has a stored webhookId (internal).
- **Inputs/outputs:** Input from **Add n. comments**; output loops back to **Loop Over Items**.
- **Edge cases / failures:**
  - Wait node configuration determines whether it actually pauses execution; ensure it’s configured to delay rather than “wait for webhook” if the intention is throttling. (In exported JSON, the delay specifics aren’t shown here.)

**Sticky note coverage (context):**
- “STEP 3- Get Facebook Comments”

---

### 2.5 Engagement collection + Google Sheets upsert (Likes branch)
**Overview:** Iterates posts, fetches reactions summary count, upserts LIKES for the corresponding POST ID, and waits before continuing.

**Nodes involved:**  
- Loop Over Items1  
- Get Likes  
- Add n. likes  
- Wait1

#### Node: Loop Over Items1
- **Type / role:** Split in Batches (loop controller).
- **Connections:** Loop body → **Get Likes**; after **Wait1** returns to **Loop Over Items1**.
- **Edge cases:** Same loop considerations as comments branch.

#### Node: Get Likes
- **Type / role:** Facebook Graph API — reactions edge.
- **Configuration choices:**
  - **Node:** `={{ $json.id }}`
  - **Edge:** `reactions`
  - **Fields:** `type`
  - **Query parameters:**
    - `order=reverse_chronological`
    - `summary=true` (to get `summary.total_count`)
  - **Graph API Version:** `v21.0`
- **Key outputs:** `summary.total_count` used as likes/reactions count.
- **Inputs/outputs:** Input from loop; output to **Add n. likes**.
- **Edge cases / failures:**
  - “Likes” here are actually **total reactions** (Like/Love/Wow/etc.) since the Graph API returns reactions.
  - Permissions required for reading engagement; may fail or return partial data.

#### Node: Add n. likes
- **Type / role:** Google Sheets — appendOrUpdate LIKES value.
- **Configuration choices:**
  - **Matching column:** `POST ID`
  - **Mapped values:**
    - `POST ID = {{ $('Loop Over Items1').item.json.id }}`
    - `LIKES = {{ $json.summary.total_count }}`
- **Inputs/outputs:** Input from **Get Likes**; output to **Wait1**.
- **Edge cases / failures:**
  - Same sheet header and OAuth issues as other Sheets nodes.

#### Node: Wait1
- **Type / role:** Wait node — throttling/pacing.
- **Inputs/outputs:** Input from **Add n. likes**; output back to **Loop Over Items1**.
- **Edge cases:** Same as Wait node above.

**Sticky note coverage (context):**
- “STEP 4- Get Facebook Likes”

---

### 2.6 Engagement collection + Google Sheets upsert (Shares branch)
**Overview:** Iterates posts, fetches the `shares` field, writes shares count for the POST ID, and waits before continuing.

**Nodes involved:**  
- Loop Over Items2  
- Get Shares  
- Add n. shares  
- Wait2

#### Node: Loop Over Items2
- **Type / role:** Split in Batches (loop controller).
- **Connections:** Loop body → **Get Shares**; after **Wait2** returns to **Loop Over Items2**.
- **Edge cases:** Same loop considerations as other branches.

#### Node: Get Shares
- **Type / role:** Facebook Graph API — fetches post fields including shares.
- **Configuration choices:**
  - **Node:** `={{ $json.id }}`
  - **Fields:** `id,message,created_time,shares`
  - **Query parameter:** `order=reverse_chronological`
  - **Graph API Version:** `v21.0`
- **Key outputs:** `shares.count` used downstream.
- **Inputs/outputs:** Input from loop; output to **Add n. shares**.
- **Edge cases / failures:**
  - Many posts will not include a `shares` object if count is zero; in that case `{{$json.shares.count}}` can evaluate to `undefined` and may write blank or error depending on node behavior. A safer expression would be `{{ $json.shares?.count ?? 0 }}`.
  - Permission restrictions similar to other Facebook calls.

#### Node: Add n. shares
- **Type / role:** Google Sheets — appendOrUpdate SHARES value.
- **Configuration choices:**
  - **Matching column:** `POST ID`
  - **Mapped values:**
    - `POST ID = {{ $('Loop Over Items2').item.json.id }}`
    - `SHARES = {{ $json.shares.count }}`
- **Inputs/outputs:** Input from **Get Shares**; output to **Wait2**.
- **Edge cases / failures:**
  - Missing `shares.count` (see above).
  - Same sheet header and OAuth issues as other Sheets nodes.

#### Node: Wait2
- **Type / role:** Wait node — throttling/pacing.
- **Inputs/outputs:** Input from **Add n. shares**; output back to **Loop Over Items2**.
- **Edge cases:** Same as other Wait nodes.

**Sticky note coverage (context):**
- “STEP 5 - Get Facebook Shares”

---

### 2.7 Sticky notes / embedded documentation
**Overview:** Non-executing nodes used for guidance, setup steps, and external resources.

**Nodes involved:**  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note  
- Sticky Note4  
- Sticky Note5  
- Sticky Note9

**Edge cases:** None (documentation only).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow start | — | Set Max Posts |  |
| Set Max Posts | Set | Defines `max_post` limit | When clicking ‘Execute workflow’ | Get Page Info | ## STEP 1 - Set Max Posts<br><br>Set number max posts to analyze |
| Get Page Info | Facebook Graph API | Fetches `me` info (expects Page context) | Set Max Posts | Get Page Feed | ## STEP 2 - Obtain Access Token<br><br>Get a temporary [access token](https://developers.facebook.com/tools/explorer/) for your Facebook Page |
| Get Page Feed | Facebook Graph API | Reads Page feed with `limit` | Get Page Info | Split Out | ## STEP 2 - Obtain Access Token<br><br>Get a temporary [access token](https://developers.facebook.com/tools/explorer/) for your Facebook Page |
| Split Out | Split Out | Splits `data[]` feed into per-post items | Get Page Feed | Loop Over Items; Loop Over Items1; Loop Over Items2 |  |
| Loop Over Items | Split in Batches | Loop controller (comments branch) | Split Out; Wait | Get comments | ## STEP 3- Get Facebook Comments |
| Get comments | Facebook Graph API | Fetch comments edge + summary count | Loop Over Items | Add n. comments | ## STEP 3- Get Facebook Comments |
| Add n. comments | Google Sheets | Upsert POST text + COMMENTS by POST ID | Get comments | Wait | ## STEP 3- Get Facebook Comments |
| Wait | Wait | Pacing between iterations (comments) | Add n. comments | Loop Over Items | ## STEP 3- Get Facebook Comments |
| Loop Over Items1 | Split in Batches | Loop controller (likes branch) | Split Out; Wait1 | Get Likes | ## STEP 4- Get Facebook Likes |
| Get Likes | Facebook Graph API | Fetch reactions edge + summary count | Loop Over Items1 | Add n. likes | ## STEP 4- Get Facebook Likes |
| Add n. likes | Google Sheets | Upsert LIKES by POST ID | Get Likes | Wait1 | ## STEP 4- Get Facebook Likes |
| Wait1 | Wait | Pacing between iterations (likes) | Add n. likes | Loop Over Items1 | ## STEP 4- Get Facebook Likes |
| Loop Over Items2 | Split in Batches | Loop controller (shares branch) | Split Out; Wait2 | Get Shares | ## STEP 5 - Get Facebook Shares |
| Get Shares | Facebook Graph API | Fetch post fields including `shares` | Loop Over Items2 | Add n. shares | ## STEP 5 - Get Facebook Shares |
| Add n. shares | Google Sheets | Upsert SHARES by POST ID | Get Shares | Wait2 | ## STEP 5 - Get Facebook Shares |
| Wait2 | Wait | Pacing between iterations (shares) | Add n. shares | Loop Over Items2 | ## STEP 5 - Get Facebook Shares |
| Sticky Note1 | Sticky Note | Documentation | — | — |  |
| Sticky Note2 | Sticky Note | Documentation | — | — |  |
| Sticky Note3 | Sticky Note | Documentation | — | — |  |
| Sticky Note | Sticky Note | Documentation | — | — |  |
| Sticky Note4 | Sticky Note | Documentation | — | — |  |
| Sticky Note5 | Sticky Note | Documentation | — | — |  |
| Sticky Note9 | Sticky Note | Documentation / external link | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Facebook Page Posts Engagement Tracker”** (inactive by default).

2. **Add Trigger**
   1. Add node: **Manual Trigger**
   2. Name it: **When clicking ‘Execute workflow’**

3. **Add configuration node**
   1. Add node: **Set**
   2. Name it: **Set Max Posts**
   3. Add field:
      - `max_post` (String) = `3` (or your preferred number)
   4. Connect: Manual Trigger → Set Max Posts

4. **Configure Facebook Graph API credential**
   1. In **Credentials**, create **Facebook Graph API** credential.
   2. Paste an access token (e.g., from Graph API Explorer): https://developers.facebook.com/tools/explorer/
   3. Ensure the token has the needed Page permissions to read feed and engagement (requirements vary by app mode and Graph API policies).

5. **Add Facebook Page identity node**
   1. Add node: **Facebook Graph API**
   2. Name: **Get Page Info**
   3. Set **Node** = `me`
   4. Set **Graph API Version** = `v21.0`
   5. Select your Facebook credential
   6. Connect: Set Max Posts → Get Page Info

6. **Add Facebook feed retrieval**
   1. Add node: **Facebook Graph API**
   2. Name: **Get Page Feed**
   3. Set **Node** to expression: `{{ $json.id }}/feed`
   4. Add **Query Parameter**:
      - `limit` = `{{ $('Set Max Posts').item.json.max_post }}`
   5. Set **Graph API Version** = `v21.0`
   6. Connect: Get Page Info → Get Page Feed

7. **Split feed into posts**
   1. Add node: **Split Out**
   2. Name: **Split Out**
   3. Set **Field to split out** = `data`
   4. Connect: Get Page Feed → Split Out

8. **Branch A (Comments)**
   1. Add node: **Split in Batches** (v3)
      - Name: **Loop Over Items**
      - (Optional) Set batch size to `1` for controlled pacing
   2. Connect: Split Out → Loop Over Items
   3. Add node: **Facebook Graph API**
      - Name: **Get comments**
      - Node (expression): `{{ $json.id }}`
      - Edge: `comments`
      - Fields: `id,from,message,created_time,comments`
      - Query parameters:
        - `order=reverse_chronological`
        - `summary=true`
        - `filter=stream`
      - Graph API Version: `v21.0`
   4. Connect: Loop Over Items (loop output) → Get comments
   5. Add node: **Google Sheets** (OAuth2)
      - Name: **Add n. comments**
      - Credential: **Google Sheets OAuth2**
      - Operation: **Append or Update**
      - Spreadsheet: choose/create your target sheet
      - Sheet tab: select the tab containing headers
      - Matching column: `POST ID`
      - Map fields:
        - `POST ID` = `{{ $('Loop Over Items').item.json.id }}`
        - `POST` = `{{ $('Loop Over Items').item.json.message }}`
        - `COMMENTS` = `{{ $json.summary.total_count }}`
      - Ensure the sheet has headers: `POST ID`, `POST`, `LIKES`, `COMMENTS`, `SHARES`
   6. Connect: Get comments → Add n. comments
   7. Add node: **Wait**
      - Name: **Wait**
      - Configure it as a delay/pause appropriate for pacing (per your n8n Wait mode)
   8. Connect: Add n. comments → Wait
   9. Connect: Wait → Loop Over Items (to continue looping)

9. **Branch B (Likes/Reactions)**
   1. Add node: **Split in Batches** (v3)
      - Name: **Loop Over Items1**
   2. Connect: Split Out → Loop Over Items1
   3. Add node: **Facebook Graph API**
      - Name: **Get Likes**
      - Node: `{{ $json.id }}`
      - Edge: `reactions`
      - Fields: `type`
      - Query parameters:
        - `order=reverse_chronological`
        - `summary=true`
      - Graph API Version: `v21.0`
   4. Connect: Loop Over Items1 (loop output) → Get Likes
   5. Add node: **Google Sheets**
      - Name: **Add n. likes**
      - Operation: **Append or Update**
      - Matching column: `POST ID`
      - Map fields:
        - `POST ID` = `{{ $('Loop Over Items1').item.json.id }}`
        - `LIKES` = `{{ $json.summary.total_count }}`
   6. Connect: Get Likes → Add n. likes
   7. Add node: **Wait**
      - Name: **Wait1**
   8. Connect: Add n. likes → Wait1
   9. Connect: Wait1 → Loop Over Items1

10. **Branch C (Shares)**
   1. Add node: **Split in Batches** (v3)
      - Name: **Loop Over Items2**
   2. Connect: Split Out → Loop Over Items2
   3. Add node: **Facebook Graph API**
      - Name: **Get Shares**
      - Node: `{{ $json.id }}`
      - Fields: `id,message,created_time,shares`
      - Query parameter: `order=reverse_chronological`
      - Graph API Version: `v21.0`
   4. Connect: Loop Over Items2 (loop output) → Get Shares
   5. Add node: **Google Sheets**
      - Name: **Add n. shares**
      - Operation: **Append or Update**
      - Matching column: `POST ID`
      - Map fields:
        - `POST ID` = `{{ $('Loop Over Items2').item.json.id }}`
        - `SHARES` = `{{ $json.shares.count }}` (recommended safer version: `{{ $json.shares?.count ?? 0 }}`)
   6. Connect: Get Shares → Add n. shares
   7. Add node: **Wait**
      - Name: **Wait2**
   8. Connect: Add n. shares → Wait2
   9. Connect: Wait2 → Loop Over Items2

11. **(Optional) Replace Manual Trigger with Schedule Trigger**
   - If you want periodic tracking, swap the Manual Trigger for a **Schedule Trigger** (e.g., daily).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically retrieves engagement data (likes, comments, shares) from a Facebook Page and stores it in Google Sheets; uses POST ID as unique key; Wait nodes reduce rate-limit risk; configure Facebook token + Google Sheets OAuth2; adjust `max_post`; optionally use Schedule Trigger. | Sticky Note1 |
| STEP 1 - Set Max Posts: Set number max posts to analyze | Sticky Note2 |
| STEP 2 - Obtain Access Token: Get a temporary access token for your Facebook Page | https://developers.facebook.com/tools/explorer/ |
| STEP 3- Get Facebook Comments | Sticky Note |
| STEP 4- Get Facebook Likes | Sticky Note4 |
| STEP 5 - Get Facebook Shares | Sticky Note5 |
| MY NEW YOUTUBE CHANNEL: Subscribe + templates | https://youtube.com/@n3witalia (image link: https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg) |