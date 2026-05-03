Social media auto-posting to Instagram, Facebook & LinkedIn from Google Sheets

https://n8nworkflows.xyz/workflows/social-media-auto-posting-to-instagram--facebook---linkedin-from-google-sheets-11996


# Social media auto-posting to Instagram, Facebook & LinkedIn from Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Social media auto-posting to Instagram, Facebook & LinkedIn from Google Sheets  
**Internal workflow name (JSON):** 4 Multi-Platform Social Media Content Distribution with Google Sheets Integration

**Purpose:**  
Automatically publishes text-based social posts to selected platforms (Instagram, Facebook, LinkedIn) when a new row is added to a Google Sheet, then updates the originating row with a status for tracking.

**Target use cases:**
- Lightweight content calendar in Google Sheets
- Multi-platform distribution with per-row platform toggles
- Basic publishing + post tracking in the same spreadsheet

### Logical blocks
1. **1.1 Trigger & Configuration**
   - Detect new row, inject required IDs (Instagram Business Account ID, Facebook Page ID).
2. **1.2 Platform Routing (Flags in Sheet)**
   - Read `Instagram`, `Facebook`, `LinkedIn` columns and route only to TRUE platforms.
3. **1.3 Publishing**
   - Post `Content` to each selected platform via API nodes.
4. **1.4 Status Update**
   - Update the same Google Sheets row after posting (intended to write to `Status`).

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Configuration

**Overview:**  
Watches a Google Sheet for new rows and enriches each incoming row with configuration constants (page/account IDs) used by downstream API calls.

**Nodes involved:**
- 📋 Workflow Overview (Sticky Note)
- New Row in Content Sheet (Google Sheets Trigger)
- Workflow Configuration (Set)

#### Node: 📋 Workflow Overview
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Displays a short description of the trigger/config block.
- **Connections:** None (non-executable)
- **Edge cases:** None

#### Node: New Row in Content Sheet
- **Type / role:** `googleSheetsTrigger` — polls for new rows (`rowAdded`)
- **Configuration (interpreted):**
  - Event: **Row added**
  - Polling: **Every minute**
  - **Document ID** and **Sheet name** are not filled in the JSON (placeholders / empty selection), must be configured.
- **Inputs / outputs:**
  - **Output:** Emits the newly added row as JSON (includes columns like `Content`, `Instagram`, etc., and a `row_number` field as referenced later).
  - **Output connection:** → `Workflow Configuration`
- **Credentials:** Google Sheets Trigger OAuth2 required
- **Common failures / edge cases:**
  - Missing/invalid document or sheet selection
  - OAuth token expiration / permission issues
  - Polling duplicates if the sheet is edited in ways the trigger interprets as new rows (depends on connector behavior)
  - Data typing inconsistencies (e.g., `TRUE` vs `true` vs `Yes`)

#### Node: Workflow Configuration
- **Type / role:** `set` — injects workflow constants into the item
- **Configuration (interpreted):**
  - Adds:
    - `instagramPageId` = placeholder for **Instagram Business Account ID**
    - `facebookPageId` = placeholder for **Facebook Page ID**
  - **Include other fields: true** (keeps the row data such as `Content`, platform flags, `row_number`)
- **Outputs / connections:**
  - → `Check Platform: Instagram`
- **TypeVersion:** Set node v3.4
- **Failures / edge cases:**
  - If IDs remain placeholders, downstream Graph API calls will fail (invalid node ID)
  - If multiple items were ever processed, using `$('Workflow Configuration').first()` downstream can cause mismatches (this workflow appears single-row-per-trigger, so usually OK)

---

### 2.2 Platform Routing (Flags in Sheet)

**Overview:**  
Evaluates platform flag columns from the row (`Instagram`, `Facebook`, `LinkedIn`). Only the platforms marked as the string `TRUE` will be posted to. The routing is implemented as a chain of IF nodes with “false” paths continuing to the next platform.

**Nodes involved:**
- 🔮 Future: Image Processing (Sticky Note)
- Check Platform: Instagram (IF)
- Check Platform: Facebook (IF)
- Check Platform: LinkedIn (IF)

#### Node: 🔮 Future: Image Processing
- **Type / role:** Sticky Note (documentation)
- **Content:** Describes platform routing based on TRUE flags.
- **Connections:** None

#### Node: Check Platform: Instagram
- **Type / role:** `if` — platform gate for Instagram
- **Condition logic:**
  - Checks: `={{ $json.Instagram }}` **equals** `"TRUE"` (case sensitive enabled in this node’s condition options)
- **Outputs / connections:**
  - **True path (index 0):** → `Post to Instagram`
  - **False path (index 1):** → `Check Platform: Facebook`
  - Additionally, after posting to Instagram, there is no explicit join back into the chain; the chain continues only via the IF false output. In this workflow, the Instagram IF true output does not proceed to Facebook check automatically.
- **Important behavior implication:**
  - As wired, if Instagram is TRUE, it will post to Instagram, but **Facebook/LinkedIn checks will not run** for that row unless separately connected. (Currently, the node is connected so that IF’s false path goes to Facebook; true path goes only to Instagram posting.)
- **Failures / edge cases:**
  - If Google Sheets provides boolean TRUE instead of string `"TRUE"`, strict equals may fail.
  - Column name mismatch (`Instagram` must exist exactly)
- **TypeVersion:** 2.3

#### Node: Check Platform: Facebook
- **Type / role:** `if` — platform gate for Facebook
- **Condition logic:**
  - Checks: `={{ $json.Facebook }}` equals `"TRUE"`
  - Case sensitivity false, type validation loose
- **Outputs / connections:**
  - **True path:** → `Post to Facebook`
  - **False path:** → `Check Platform: LinkedIn`
- **Edge cases:**
  - Same typing issues as above; values like `"TRUE "` or `"Yes"` will not match.

#### Node: Check Platform: LinkedIn
- **Type / role:** `if` — platform gate for LinkedIn
- **Condition logic:**
  - Checks: `={{ $json.LinkedIn }}` equals `"TRUE"`
- **Outputs / connections:**
  - **True path:** → `Post to LinkedIn`
  - **False path:** no connection (row effectively ends with no action)
- **Edge cases:**
  - If multiple platforms should be posted simultaneously, current chain design prevents “fan-out” when earlier IF is true (see note above).

---

### 2.3 Publishing

**Overview:**  
Publishes the `Content` field to the selected platforms using native n8n nodes (Facebook Graph API for Facebook/Instagram and LinkedIn node for LinkedIn).

**Nodes involved:**
- 🔮 Future: Analytics (Sticky Note)
- Post to Instagram (Facebook Graph API)
- Post to Facebook (Facebook Graph API)
- Post to LinkedIn (LinkedIn)

#### Node: 🔮 Future: Analytics
- **Type / role:** Sticky Note (documentation)
- **Content:** “Publish Posts” section label.

#### Node: Post to Instagram
- **Type / role:** `facebookGraphApi` — POST to Graph API edge `feed`
- **Configuration (interpreted):**
  - HTTP method: POST
  - Node (Graph object ID): `={{ $('Workflow Configuration').first().json.instagramPageId }}`
  - Edge: `feed`
  - Query parameter: `message = {{ $json.Content }}`
- **Inputs / outputs:**
  - Input: row item (must contain `Content`)
  - Output: Graph API response (post ID, etc., depending on API)
  - Output connection: → `Update Status: Instagram`
- **Credentials:** Not shown in JSON (Graph API node typically requires Facebook access token credentials in n8n). Must be configured in n8n UI.
- **Version notes:** Node typeVersion 1
- **Potential failures / edge cases:**
  - **Instagram posting via `/{ig-user-id}/feed` is not valid for Instagram publishing.** Instagram Graph API typically requires the **Content Publishing** flow (create media container + publish) and uses `/{ig-user-id}/media` and `/{ig-user-id}/media_publish` (for images/videos). Text-only “feed” posting is generally a Facebook Page concept.
  - Permission issues: missing `pages_manage_posts`, `pages_read_engagement`, or Instagram-specific permissions and required business linkage.
  - Invalid ID (placeholder not replaced)
  - Rate limits or API version changes

#### Node: Post to Facebook
- **Type / role:** `facebookGraphApi` — POST to Page feed
- **Configuration (interpreted):**
  - Node (Page ID): `={{ $('Workflow Configuration').first().json.facebookPageId }}`
  - Edge: `feed`
  - Query parameter: `message = {{ $json.Content }}`
- **Connections:** → `Update Status: Facebook`
- **Credentials:** Must be configured (Facebook token with page permissions)
- **Failures / edge cases:**
  - Missing permissions, page token issues
  - Posting restrictions (page limitations, spam detection)
  - Content empty or exceeds limits

#### Node: Post to LinkedIn
- **Type / role:** `linkedIn` — publish text
- **Configuration (interpreted):**
  - `text = {{ $json.Content }}`
  - Additional fields: none
- **Connections:** → `Update Status: LinkedIn`
- **Credentials:** LinkedIn OAuth2 required (present in JSON)
- **Failures / edge cases:**
  - LinkedIn API access limitations (requires appropriate app permissions and sometimes product access)
  - Posting identity ambiguity (member vs organization); this node configuration doesn’t show organization URN selection—may default to the authenticated member depending on node behavior/version
  - Rate limits, OAuth expiry

---

### 2.4 Status Update

**Overview:**  
After each successful post, updates the original Google Sheets row (matched by `Row Number`) to mark publishing status. However, the current configuration **does not actually set a Status value**—it only matches the row.

**Nodes involved:**
- 🔮 Future: Analytics1 (Sticky Note)
- Update Status: Instagram (Google Sheets)
- Update Status: Facebook (Google Sheets)
- Update Status: LinkedIn (Google Sheets)

#### Node: 🔮 Future: Analytics1
- **Type / role:** Sticky Note (documentation)
- **Content:** “Update Sheet Status” section label.

#### Node: Update Status: Instagram
- **Type / role:** `googleSheets` — update row
- **Configuration (interpreted):**
  - Operation: **Update**
  - Matching column: `Row Number`
  - Values provided:
    - `Row Number = {{ $json.row_number }}`
    - **No `Status` value is set** (schema includes Status, but it is not assigned)
  - Sheet name and document ID are placeholders and must match the trigger’s sheet/document.
- **Input / output:**
  - Input: Graph API response merged with original fields? (Depends on n8n item passing; typically it passes the API response item. If `row_number` isn’t preserved, update will fail.)
- **Critical edge case:**
  - If the posting node output does not include `row_number`, the expression `{{ $json.row_number }}` will be undefined, and the update may not match any row.
  - Common fix: use **Merge** / **Set** to carry forward original sheet fields, or reference the trigger node directly: `{{$('New Row in Content Sheet').first().json.row_number}}`
- **Failures:**
  - Sheet/document not configured
  - Auth errors
  - No matching row found due to row number mismatch or missing field

#### Node: Update Status: Facebook
- Same structure and issues as Instagram status update:
  - Matches on `Row Number = {{ $json.row_number }}`
  - Does not set `Status`
  - Placeholders for sheet/document

#### Node: Update Status: LinkedIn
- Same structure and issues as above.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Workflow Overview | Sticky Note | Documents trigger/config block | — | — | ## Trigger & Configuration; Detects new rows added to Google Sheets and sets required configuration values such as platform IDs before routing the post through the workflow. |
| New Row in Content Sheet | Google Sheets Trigger | Entry point: detect new rows | — | Workflow Configuration | ## How it works; This workflow automatically publishes social media posts using Google Sheets as a lightweight content calendar. Each new row added to the sheet represents one post. Platform-specific flags (Instagram, Facebook, LinkedIn) determine where the content should be published. … (includes setup steps 1–6) |
| Workflow Configuration | Set | Inject platform IDs into item | New Row in Content Sheet | Check Platform: Instagram | ## Trigger & Configuration; Detects new rows added to Google Sheets and sets required configuration values such as platform IDs before routing the post through the workflow. |
| Check Platform: Instagram | IF | Route to Instagram if enabled | Workflow Configuration | Post to Instagram; Check Platform: Facebook | ## Platform Routing; Checks platform flags from the sheet (Instagram, Facebook, LinkedIn) and routes the post only to the platforms marked as TRUE. |
| Check Platform: Facebook | IF | Route to Facebook if enabled | Check Platform: Instagram | Post to Facebook; Check Platform: LinkedIn | ## Platform Routing; Checks platform flags from the sheet (Instagram, Facebook, LinkedIn) and routes the post only to the platforms marked as TRUE. |
| Check Platform: LinkedIn | IF | Route to LinkedIn if enabled | Check Platform: Facebook | Post to LinkedIn | ## Platform Routing; Checks platform flags from the sheet (Instagram, Facebook, LinkedIn) and routes the post only to the platforms marked as TRUE. |
| Post to Instagram | Facebook Graph API | Publish content to Instagram (as configured) | Check Platform: Instagram | Update Status: Instagram | ## Publish Posts; Publishes the content text to the selected social media platforms using their respective APIs. |
| Post to Facebook | Facebook Graph API | Publish content to Facebook Page feed | Check Platform: Facebook | Update Status: Facebook | ## Publish Posts; Publishes the content text to the selected social media platforms using their respective APIs. |
| Post to LinkedIn | LinkedIn | Publish content to LinkedIn | Check Platform: LinkedIn | Update Status: LinkedIn | ## Publish Posts; Publishes the content text to the selected social media platforms using their respective APIs. |
| Update Status: Instagram | Google Sheets | Update originating row status (intended) | Post to Instagram | — | ## Update Sheet Status; Updates the original Google Sheets row after posting, allowing you to track which platforms were successfully published. |
| Update Status: Facebook | Google Sheets | Update originating row status (intended) | Post to Facebook | — | ## Update Sheet Status; Updates the original Google Sheets row after posting, allowing you to track which platforms were successfully published. |
| Update Status: LinkedIn | Google Sheets | Update originating row status (intended) | Post to LinkedIn | — | ## Update Sheet Status; Updates the original Google Sheets row after posting, allowing you to track which platforms were successfully published. |
| 🔮 Future: Image Processing | Sticky Note | Documents routing section | — | — | ## Platform Routing; Checks platform flags from the sheet (Instagram, Facebook, LinkedIn) and routes the post only to the platforms marked as TRUE. |
| 🔮 Future: Scheduling | Sticky Note | Full description + setup steps | — | — | ## How it works; (full block with sheet columns and setup steps 1–6) |
| 🔮 Future: Analytics | Sticky Note | Documents publish section | — | — | ## Publish Posts; Publishes the content text to the selected social media platforms using their respective APIs. |
| 🔮 Future: Analytics1 | Sticky Note | Documents status update section | — | — | ## Update Sheet Status; Updates the original Google Sheets row after posting, allowing you to track which platforms were successfully published. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet**
   1. Create a spreadsheet with columns (header row):  
      `Content`, `Instagram`, `Facebook`, `LinkedIn`, `Status`  
      (Optionally add `Row Number` if you plan to store it; the trigger often provides `row_number` automatically.)
   2. For each post row, set platform cells to `TRUE`/`FALSE` (as text), and fill `Content`.

2. **Add Trigger Node**
   1. Add node: **Google Sheets Trigger** → *New Row in Content Sheet*
   2. Set **Event**: `Row Added`
   3. Set **Poll Time**: Every minute
   4. Select **Document** (Spreadsheet) and **Sheet** (tab)
   5. Attach **Google Sheets Trigger OAuth2** credentials.

3. **Add Configuration Node**
   1. Add node: **Set** → *Workflow Configuration*
   2. Enable **Include Other Fields**
   3. Add fields:
      - `instagramPageId` = your Instagram Business Account ID (or the correct ID required by your chosen endpoint)
      - `facebookPageId` = your Facebook Page ID
   4. Connect: *New Row in Content Sheet* → *Workflow Configuration*

4. **Add Platform Routing IF nodes (chain)**
   1. Add **IF** → *Check Platform: Instagram*
      - Condition: `{{$json.Instagram}}` equals `"TRUE"` (be consistent with case)
   2. Add **IF** → *Check Platform: Facebook*
      - Condition: `{{$json.Facebook}}` equals `"TRUE"`
   3. Add **IF** → *Check Platform: LinkedIn*
      - Condition: `{{$json.LinkedIn}}` equals `"TRUE"`
   4. Connect:
      - *Workflow Configuration* → *Check Platform: Instagram*
      - *Check Platform: Instagram* **false** → *Check Platform: Facebook*
      - *Check Platform: Facebook* **false** → *Check Platform: LinkedIn*
   5. If you want to post to **multiple** platforms when several flags are TRUE, do **fan-out** routing instead of a true/false chain (e.g., connect the same input to multiple IF nodes, or connect true paths back into subsequent checks).

5. **Add Publishing Nodes**
   1. Add **Facebook Graph API** → *Post to Facebook*
      - Method: POST
      - Node: `{{$('Workflow Configuration').first().json.facebookPageId}}`
      - Edge: `feed`
      - Query param `message`: `{{$json.Content}}`
      - Configure Facebook Graph credentials/token with permissions to post to the page.
   2. Add **LinkedIn** → *Post to LinkedIn*
      - Text: `{{$json.Content}}`
      - Configure **LinkedIn OAuth2** credentials.
   3. Add **Facebook Graph API** → *Post to Instagram* (as in the provided workflow)
      - Same structure as Facebook node but using `instagramPageId`
      - Note: for real Instagram publishing, replace with the proper Instagram Content Publishing flow.
   4. Connect:
      - *Check Platform: Facebook* **true** → *Post to Facebook*
      - *Check Platform: LinkedIn* **true** → *Post to LinkedIn*
      - *Check Platform: Instagram* **true** → *Post to Instagram*

6. **Add Status Update Nodes (Google Sheets)**
   1. Add **Google Sheets** → *Update Status: Facebook*
      - Operation: `Update`
      - Document: same spreadsheet as trigger
      - Sheet: same tab as trigger
      - Matching column: `Row Number`
      - Set `Row Number = {{$('New Row in Content Sheet').first().json.row_number}}` (recommended for reliability)
      - Set `Status = "Posted to Facebook"` (or any desired text)
   2. Repeat for **Instagram** and **LinkedIn** with platform-specific status text.
   3. Connect:
      - *Post to Facebook* → *Update Status: Facebook*
      - *Post to Instagram* → *Update Status: Instagram*
      - *Post to LinkedIn* → *Update Status: LinkedIn*

7. **Credentials checklist**
   - Google Sheets Trigger OAuth2
   - Google Sheets OAuth2 (for update nodes)
   - Facebook Graph API credentials/token (pages posting permissions)
   - LinkedIn OAuth2 credentials (posting permissions)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Uses Google Sheets as a lightweight content calendar; each new row becomes one post; flags determine destinations; updates the same row with a status. | From sticky note “## How it works” |
| Setup steps: create columns Content/Instagram/Facebook/LinkedIn/Status; connect credentials; fill IDs in Workflow Configuration; set TRUE/FALSE; add row to publish. | From sticky note “## Setup steps” |

### Notable implementation gaps to address (important)
- **Multi-platform posting logic:** current true/false chaining means a TRUE on Instagram prevents Facebook/LinkedIn evaluation for that same row unless you redesign routing (fan-out).
- **Status update nodes don’t set `Status`:** they currently only provide Row Number; add a `Status` value (and/or separate columns per platform).
- **Row number propagation risk:** posting nodes may not carry `row_number`; reference the trigger node directly or merge original fields before updating.
- **Instagram posting endpoint:** `/{instagramPageId}/feed` is typically not a valid Instagram publishing method; implement Instagram Graph “content publishing” flow if you need real Instagram posts.