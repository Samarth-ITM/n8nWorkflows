Auto-generate Instagram posts from Google Sheets reviews using UploadtoUrl

https://n8nworkflows.xyz/workflows/auto-generate-instagram-posts-from-google-sheets-reviews-using-uploadtourl-14648


# Auto-generate Instagram posts from Google Sheets reviews using UploadtoUrl

# 1. Workflow Overview

This workflow automates the creation and publication of Instagram social proof posts from customer reviews stored in Google Sheets.

Its intended use case is straightforward: on a schedule, the workflow retrieves a review, transforms it into branded marketing content, renders a square image, uploads that image to a public URL, publishes it via the Instagram Graph API, updates the source sheet to avoid reposting, and notifies the team in Slack.

## 1.1 Scheduled Review Intake
The workflow starts on a daily schedule and reads review data from Google Sheets. It then checks whether a usable review exists before continuing.

## 1.2 Review Normalization and Caption Preparation
The selected review is cleaned and transformed into reusable fields: reviewer name, text variants, rating, source, and Instagram caption.

## 1.3 Card Composition and Image Rendering
The workflow builds an HTML/CSS quote card using brand variables, then sends that HTML to hcti.io to generate an image.

## 1.4 Public Image Hosting and Instagram Media Creation
Because Instagram requires a public image URL, the rendered image is uploaded through Upload to URL to obtain a CDN link. The workflow then creates an Instagram media container using that link and the generated caption.

## 1.5 Publication, Record Update, and Team Notification
After a short wait to let Instagram validate the media, the workflow publishes the post, updates the Google Sheet to mark the review as posted, and sends a Slack notification with the publishing details.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Review Intake

### Overview
This block triggers the workflow once per day and attempts to fetch a review row from Google Sheets. It also prevents unnecessary downstream execution when no valid review is available.

### Nodes Involved
- Schedule Trigger
- Google Sheets Fetch Review
- IF Has Valid Review

### Node Details

#### 1. Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry-point node that runs the workflow on a time-based schedule.
- **Configuration choices:**
  - Configured to trigger daily at **9 AM**.
  - Uses the schedule rule interval format rather than a manual cron string.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No input; workflow entry point.
  - Outputs to **Google Sheets Fetch Review**.
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`.
- **Edge cases or potential failure types:**
  - Timezone mismatches if the n8n instance timezone is not aligned with business expectations.
  - Missed execution if the instance is offline at trigger time.
- **Sub-workflow reference:** None.

#### 2. Google Sheets Fetch Review
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from a Google Sheets document.
- **Configuration choices:**
  - Operation: **Read Rows**
  - Document ID is taken from environment variable `GSHEET_DOCUMENT_ID`.
  - The template description indicates this node should fetch the oldest unposted 5-star review, but the JSON shown only explicitly sets the document ID and operation. Any row filtering, sorting, sheet selection, or row limit is not visible in the provided configuration.
- **Key expressions or variables used:**
  - `={{ $env.GSHEET_DOCUMENT_ID }}`
- **Input and output connections:**
  - Input from **Schedule Trigger**
  - Output to **IF Has Valid Review**
- **Version-specific requirements:**
  - Uses `typeVersion: 4.4`.
  - Requires valid Google Sheets credentials in n8n.
- **Edge cases or potential failure types:**
  - Missing or invalid Google Sheets credentials.
  - `GSHEET_DOCUMENT_ID` missing or incorrect.
  - Wrong sheet/tab selection if not configured manually after import.
  - Unexpected column names causing downstream expressions to fail or return empty values.
  - If multiple rows are returned, the current workflow logic may not behave as intended unless only one row is actually emitted.
- **Sub-workflow reference:** None.

#### 3. IF Has Valid Review
- **Type and technical role:** `n8n-nodes-base.if`; gatekeeper node that checks whether a row with a reviewer name exists.
- **Configuration choices:**
  - Condition: `Reviewer Name` must be non-empty.
  - Strict type validation enabled.
  - Only the true branch is connected; the false branch exits silently.
- **Key expressions or variables used:**
  - `={{ $json['Reviewer Name'] }}`
- **Input and output connections:**
  - Input from **Google Sheets Fetch Review**
  - True output to **Code Prepare Fields**
  - False output is intentionally unused
- **Version-specific requirements:**
  - Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - If Google Sheets returns data under different field names, the condition may fail even when a row exists.
  - Empty reviewer names cause the workflow to stop without error.
- **Sub-workflow reference:** None.

---

## Block 2 — Review Normalization and Caption Preparation

### Overview
This block standardizes the review data and builds the text assets required for both the visual card and the Instagram caption.

### Nodes Involved
- Code Prepare Fields

### Node Details

#### 4. Code Prepare Fields
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript transformation node.
- **Configuration choices:**
  - Reads the first incoming item.
  - Supports multiple possible field naming conventions:
    - `Reviewer Name` or `reviewer_name`
    - `Review Text` or `review_text`
    - `Rating` or `rating`
    - `Source` or `source`
  - Fallback defaults:
    - Reviewer name: `Happy Customer`
    - Rating: `5`
    - Source: `Verified Review`
  - Throws an error if review text is empty.
  - Creates:
    - `card_text` truncated to 200 chars
    - `final_caption` truncated to 2200 chars
    - `star_count` clamped to range 1–5
    - `ig_user_id` from environment
  - Also attempts to preserve a source record ID via `item.id` or `item.record_id`.
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$env.IG_USER_ID`
- **Input and output connections:**
  - Input from **IF Has Valid Review**
  - Output to **Set Assemble HTML Card**
- **Version-specific requirements:**
  - Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Throws explicit error if `reviewText` is empty.
  - If the Google Sheets node does not emit `id` or `record_id`, later row update logic may lack a reliable row identifier.
  - `starsText` is computed but never used; not a failure, but a minor implementation artifact.
  - Non-numeric ratings are coerced with `Number(...)`; invalid values become `NaN`, then clamping logic may not behave as expected.
- **Sub-workflow reference:** None.

---

## Block 3 — Card Composition and Image Rendering

### Overview
This block creates a branded HTML/CSS quote card and renders it to an image using hcti.io.

### Nodes Involved
- Set Assemble HTML Card
- HTTP Render HTML to Image

### Node Details

#### 5. Set Assemble HTML Card
- **Type and technical role:** `n8n-nodes-base.set`; prepares a full HTML payload and passes through structured data.
- **Configuration choices:**
  - Creates a string field `html` containing a full 1080x1080 HTML document.
  - Uses environment variables for styling:
    - `CARD_BG_COLOR`
    - `CARD_TEXT_COLOR`
    - `CARD_ACCENT_COLOR`
    - `BRAND_NAME`
  - Includes:
    - Star line based on `star_count`
    - Review quote from `card_text`
    - Reviewer name
    - Source
    - Brand name watermark/footer
  - Also stores `passFields` containing the full incoming JSON object.
- **Key expressions or variables used:**
  - `{{$env.CARD_BG_COLOR || '#1a1a2e'}}`
  - `{{$env.CARD_TEXT_COLOR || '#ffffff'}}`
  - `{{$env.CARD_ACCENT_COLOR || '#e94560'}}`
  - `{{$env.BRAND_NAME || 'Our Brand'}}`
  - `$json.star_count`
  - `$json.card_text`
  - `$json.reviewer_name`
  - `$json.source`
- **Input and output connections:**
  - Input from **Code Prepare Fields**
  - Output to **HTTP Render HTML to Image**
- **Version-specific requirements:**
  - Uses `typeVersion: 3.3`.
- **Edge cases or potential failure types:**
  - Unescaped HTML-sensitive characters in review content may break the generated HTML or alter rendering.
  - Very long text may still overflow visually despite truncation.
  - Missing env vars fall back to defaults, which is safe but may not match branding.
- **Sub-workflow reference:** None.

#### 6. HTTP Render HTML to Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends the HTML to hcti.io for image rendering.
- **Configuration choices:**
  - URL: `https://hcti.io/v1/image`
  - Method: `POST`
  - Body parameter:
    - `html` = generated HTML string
  - Uses generic credential type with **HTTP Basic Auth**
- **Key expressions or variables used:**
  - `={{ $json.html }}`
- **Input and output connections:**
  - Input from **Set Assemble HTML Card**
  - Output to **Upload to URL**
- **Version-specific requirements:**
  - Uses `typeVersion: 4.1`.
  - Requires hcti.io credentials stored as HTTP Basic Auth in n8n.
- **Edge cases or potential failure types:**
  - Invalid hcti.io credentials.
  - Rate limiting or service outage.
  - HTML too large or malformed.
  - If hcti.io returns only a hosted image URL and the downstream Upload to URL node expects binary input, extra configuration may be required.
- **Sub-workflow reference:** None.

---

## Block 4 — Public Image Hosting and Instagram Media Creation

### Overview
This block ensures the rendered image is accessible at a public HTTPS URL, then submits it to Instagram as a media container.

### Nodes Involved
- Upload to URL
- Code Merge Fields
- IG Create Media Container

### Node Details

#### 7. Upload to URL
- **Type and technical role:** `n8n-nodes-uploadtourl.uploadToUrl`; uploads an image to a public hosting layer and returns a public URL.
- **Configuration choices:**
  - No visible parameters are set in the JSON.
  - The sticky note indicates its purpose is to convert the rendered PNG into a public CDN URL for Instagram.
- **Key expressions or variables used:** None shown.
- **Input and output connections:**
  - Input from **HTTP Render HTML to Image**
  - Output to **Code Merge Fields**
- **Version-specific requirements:**
  - Uses `typeVersion: 1`.
  - Requires the Upload to URL community node/package to be installed in the n8n instance.
- **Edge cases or potential failure types:**
  - Community node not installed.
  - Upload provider misconfiguration.
  - If the upstream node does not provide binary data or a supported source URL format, upload may fail.
  - Returned schema may vary, which is why the next node checks several possible URL keys.
- **Sub-workflow reference:** None.

#### 8. Code Merge Fields
- **Type and technical role:** `n8n-nodes-base.code`; consolidates the uploaded public image URL with metadata prepared earlier.
- **Configuration choices:**
  - Reads current item from **Upload to URL**.
  - Pulls reference data directly from **Code Prepare Fields** using node lookup.
  - Searches for a public image URL in several possible output properties:
    - `public_url`
    - `url`
    - `file_url`
    - `cdn_url`
    - `imageUrl`
  - Throws an error if no public URL is found.
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Code Prepare Fields').first().json`
- **Input and output connections:**
  - Input from **Upload to URL**
  - Output to **IG Create Media Container**
- **Version-specific requirements:**
  - Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Throws explicit error if Upload to URL returns an unexpected schema.
  - Cross-node references assume **Code Prepare Fields** has executed successfully.
- **Sub-workflow reference:** None.

#### 9. IG Create Media Container
- **Type and technical role:** `n8n-nodes-base.httpRequest`; creates an Instagram media container via Graph API.
- **Configuration choices:**
  - URL pattern: `https://graph.facebook.com/v19.0/{{ig_user_id}}/media`
  - Method: `POST`
  - Sends query parameters:
    - `image_url`
    - `caption`
    - `access_token`
  - Uses environment token rather than stored OAuth credentials.
- **Key expressions or variables used:**
  - `=https://graph.facebook.com/v19.0/{{ $json.ig_user_id }}/media`
  - `={{ $json.public_image_url }}`
  - `={{ $json.final_caption }}`
  - `={{ $env.IG_ACCESS_TOKEN }}`
- **Input and output connections:**
  - Input from **Code Merge Fields**
  - Output to **Wait IG Buffer**
- **Version-specific requirements:**
  - Uses `typeVersion: 4.1`.
  - Requires a valid Instagram Business/Creator setup tied to Meta Graph API permissions.
- **Edge cases or potential failure types:**
  - Invalid or expired access token.
  - Wrong `IG_USER_ID`.
  - Public image URL not reachable by Meta servers.
  - Caption length or content rejected by API.
  - Graph API version changes may require endpoint updates.
- **Sub-workflow reference:** None.

---

## Block 5 — Publication, Record Update, and Team Notification

### Overview
This block waits for Instagram media processing, publishes the media container, records the result in Google Sheets, and notifies Slack.

### Nodes Involved
- Wait IG Buffer
- IG Publish Container
- Google Sheets Mark as Posted
- Slack Notify Team

### Node Details

#### 10. Wait IG Buffer
- **Type and technical role:** `n8n-nodes-base.wait`; inserts a fixed delay before publishing.
- **Configuration choices:**
  - Wait duration: **6 seconds**
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from **IG Create Media Container**
  - Output to **IG Publish Container**
- **Version-specific requirements:**
  - Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - Six seconds may be insufficient during API latency or media validation delays.
  - Longer waits may be needed in production for reliability.
- **Sub-workflow reference:** None.

#### 11. IG Publish Container
- **Type and technical role:** `n8n-nodes-base.httpRequest`; final Instagram publish call.
- **Configuration choices:**
  - URL pattern: `https://graph.facebook.com/v19.0/{{ig_user_id}}/media_publish`
  - Method: `POST`
  - Sends query parameters:
    - `creation_id` from the prior container creation response
    - `access_token`
  - Uses cross-node lookups rather than only current item values.
- **Key expressions or variables used:**
  - `=https://graph.facebook.com/v19.0/{{ $("Code Merge Fields").first().json.ig_user_id }}/media_publish`
  - `={{ $("IG Create Media Container").first().json.id }}`
  - `={{ $env.IG_ACCESS_TOKEN }}`
- **Input and output connections:**
  - Input from **Wait IG Buffer**
  - Output to **Google Sheets Mark as Posted**
- **Version-specific requirements:**
  - Uses `typeVersion: 4.1`.
- **Edge cases or potential failure types:**
  - Publish may fail if container is not yet ready.
  - Token, permission, or account-linking issues.
  - Missing `id` in container response breaks `creation_id`.
- **Sub-workflow reference:** None.

#### 12. Google Sheets Mark as Posted
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates the source sheet after successful publication.
- **Configuration choices:**
  - Operation: **Update Row**
  - Document ID from environment variable `GSHEET_DOCUMENT_ID`
  - The sticky note indicates it should set:
    - `Posted = TRUE`
    - `Instagram Post ID`
    - `Posted At`
    - likely `Card Image URL`
  - However, the exact row matching and updated fields are not visible in the provided JSON and must be configured manually.
- **Key expressions or variables used:**
  - `={{ $env.GSHEET_DOCUMENT_ID }}`
- **Input and output connections:**
  - Input from **IG Publish Container**
  - Output to **Slack Notify Team**
- **Version-specific requirements:**
  - Uses `typeVersion: 4.4`.
- **Edge cases or potential failure types:**
  - If no row key/row number is configured, the update will fail or affect the wrong row.
  - If the earlier fetch step does not preserve row identity, the workflow may not know which row to update.
  - Credential or document access failures.
- **Sub-workflow reference:** None.

#### 13. Slack Notify Team
- **Type and technical role:** `n8n-nodes-base.slack`; sends a channel message after successful publication.
- **Configuration choices:**
  - Uses Slack OAuth2 authentication.
  - Sends a formatted plaintext message to the channel from `SLACK_CHANNEL_ID`.
  - Includes:
    - Reviewer name
    - Source
    - Published Instagram post ID
    - First 100 characters of review text
    - Public card image URL
- **Key expressions or variables used:**
  - `={{ $env.SLACK_CHANNEL_ID }}`
  - `{{ $("Code Merge Fields").first().json.reviewer_name }}`
  - `{{ $("Code Merge Fields").first().json.source }}`
  - `{{ $("IG Publish Container").first().json.id }}`
  - `{{ $("Code Merge Fields").first().json.review_text.substring(0, 100) }}`
  - `{{ $("Code Merge Fields").first().json.public_image_url }}`
- **Input and output connections:**
  - Input from **Google Sheets Mark as Posted**
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 2.2`.
  - Requires Slack OAuth2 credentials with permission to post to the target channel.
- **Edge cases or potential failure types:**
  - Invalid Slack OAuth token.
  - Bot not present in the selected channel.
  - Missing `SLACK_CHANNEL_ID`.
  - If `review_text` is undefined, `.substring(0, 100)` would fail, though upstream code normally guarantees it.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Template Overview | Sticky Note | Documentation note describing the full workflow |  |  | Customer Review to Social Proof – Instagram Post<br>Automatically fetches the oldest unposted 5-star review from Google Sheets, generates a branded quote card using hcti.io, uploads it via UploadToURL to get a public CDN link, and publishes it to Instagram with a caption. Marks the review as posted and notifies the team on Slack.<br>Flow: Schedule → Fetch review → Validate → Prepare caption → Generate image → Upload → Create IG post → Publish → Update sheet → Notify Slack<br>Required: IG_USER_ID, IG_ACCESS_TOKEN, GSHEET_DOCUMENT_ID, SLACK_CHANNEL_ID, BRAND_NAME, CARD_BG_COLOR, CARD_TEXT_COLOR, CARD_ACCENT_COLOR<br>Sheet Columns: Reviewer Name, Review Text, Rating, Source, Posted, Submitted At, Instagram Post ID, Posted At, Card Image URL |
| Sticky Nodes 1-3 | Sticky Note | Documentation note for intake and gating |  |  | Nodes 1-3: Schedule, Fetch and Gate<br>Schedule Trigger fires daily at 9 AM.<br>Adjust the cron expression for different cadences.<br>Google Sheets - Fetch Review reads the Reviews sheet, filtering for Rating = 5 and Posted = FALSE, sorted by Submitted At ascending (FIFO queue). Fetches 1 row per run.<br>IF - Has Valid Review exits cleanly on the false branch when no unposted reviews remain. No errors, no noise. |
| Sticky Nodes 4-6 | Sticky Note | Documentation note for field prep and rendering |  |  | Field Preparation & Card Rendering<br>Code – Prepare Fields: Cleans raw row data by truncating review text, formatting star ratings, and generating a structured Instagram caption.<br>Set – Assemble HTML Card: Constructs a 1080x1080px HTML/CSS template referencing brand colors and text fields; using a Set node helps bypass Web Application Firewall (WAF) issues.<br>HTTP – hcti.io: Sends the HTML string to hcti.io to render it into a PNG image, returning a URL for the final graphic. |
| Sticky Nodes 7-9 | Sticky Note | Documentation note for upload and IG container creation |  |  | Upload to URL: Downloads the PNG from hcti.io and uploads it to a CDN to generate a mandatory public HTTPS URL, as Instagram's API rejects binary or base64 data.<br>Code – Merge Fields: Unifies the new CDN image URL with the pre-built caption and associated metadata.<br>HTTP – IG Create Media Container: Submits the CDN URL and caption to the Instagram Graph API, returning a container_id required for final publishing. |
| Sticky Nodes 10-13 | Sticky Note | Documentation note for finalization |  |  | 🚀 Finalization & Team Notification<br>Wait 6s: Provides a processing buffer to ensure Instagram finishes validating the media container before the final call.<br>HTTP – IG Publish Container: Executes the /media_publish call using the container_id and retrieves the official live Post ID.<br>Google Sheets – Mark as Posted: Updates the specific row by setting Posted to TRUE and logging the Post ID and timestamp to prevent duplicate entries.<br>Slack – Notify Team: Dispatches an alert containing the reviewer's name, a snippet of the text, and the final card URL for internal verification. |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily workflow trigger |  | Google Sheets Fetch Review | Nodes 1-3: Schedule, Fetch and Gate<br>Schedule Trigger fires daily at 9 AM.<br>Adjust the cron expression for different cadences.<br>Google Sheets - Fetch Review reads the Reviews sheet, filtering for Rating = 5 and Posted = FALSE, sorted by Submitted At ascending (FIFO queue). Fetches 1 row per run.<br>IF - Has Valid Review exits cleanly on the false branch when no unposted reviews remain. No errors, no noise. |
| Google Sheets Fetch Review | n8n-nodes-base.googleSheets | Read review data from Google Sheets | Schedule Trigger | IF Has Valid Review | Nodes 1-3: Schedule, Fetch and Gate<br>Schedule Trigger fires daily at 9 AM.<br>Adjust the cron expression for different cadences.<br>Google Sheets - Fetch Review reads the Reviews sheet, filtering for Rating = 5 and Posted = FALSE, sorted by Submitted At ascending (FIFO queue). Fetches 1 row per run.<br>IF - Has Valid Review exits cleanly on the false branch when no unposted reviews remain. No errors, no noise. |
| IF Has Valid Review | n8n-nodes-base.if | Stop flow when no valid review exists | Google Sheets Fetch Review | Code Prepare Fields | Nodes 1-3: Schedule, Fetch and Gate<br>Schedule Trigger fires daily at 9 AM.<br>Adjust the cron expression for different cadences.<br>Google Sheets - Fetch Review reads the Reviews sheet, filtering for Rating = 5 and Posted = FALSE, sorted by Submitted At ascending (FIFO queue). Fetches 1 row per run.<br>IF - Has Valid Review exits cleanly on the false branch when no unposted reviews remain. No errors, no noise. |
| Code Prepare Fields | n8n-nodes-base.code | Normalize review fields and build caption | IF Has Valid Review | Set Assemble HTML Card | Field Preparation & Card Rendering<br>Code – Prepare Fields: Cleans raw row data by truncating review text, formatting star ratings, and generating a structured Instagram caption.<br>Set – Assemble HTML Card: Constructs a 1080x1080px HTML/CSS template referencing brand colors and text fields; using a Set node helps bypass Web Application Firewall (WAF) issues.<br>HTTP – hcti.io: Sends the HTML string to hcti.io to render it into a PNG image, returning a URL for the final graphic. |
| Set Assemble HTML Card | n8n-nodes-base.set | Build HTML/CSS quote card payload | Code Prepare Fields | HTTP Render HTML to Image | Field Preparation & Card Rendering<br>Code – Prepare Fields: Cleans raw row data by truncating review text, formatting star ratings, and generating a structured Instagram caption.<br>Set – Assemble HTML Card: Constructs a 1080x1080px HTML/CSS template referencing brand colors and text fields; using a Set node helps bypass Web Application Firewall (WAF) issues.<br>HTTP – hcti.io: Sends the HTML string to hcti.io to render it into a PNG image, returning a URL for the final graphic. |
| HTTP Render HTML to Image | n8n-nodes-base.httpRequest | Render HTML card to image via hcti.io | Set Assemble HTML Card | Upload to URL | Field Preparation & Card Rendering<br>Code – Prepare Fields: Cleans raw row data by truncating review text, formatting star ratings, and generating a structured Instagram caption.<br>Set – Assemble HTML Card: Constructs a 1080x1080px HTML/CSS template referencing brand colors and text fields; using a Set node helps bypass Web Application Firewall (WAF) issues.<br>HTTP – hcti.io: Sends the HTML string to hcti.io to render it into a PNG image, returning a URL for the final graphic. |
| Upload to URL | n8n-nodes-uploadtourl.uploadToUrl | Upload rendered image to public CDN | HTTP Render HTML to Image | Code Merge Fields | Upload to URL: Downloads the PNG from hcti.io and uploads it to a CDN to generate a mandatory public HTTPS URL, as Instagram's API rejects binary or base64 data.<br>Code – Merge Fields: Unifies the new CDN image URL with the pre-built caption and associated metadata.<br>HTTP – IG Create Media Container: Submits the CDN URL and caption to the Instagram Graph API, returning a container_id required for final publishing. |
| Code Merge Fields | n8n-nodes-base.code | Combine hosted image URL with prepared metadata | Upload to URL | IG Create Media Container | Upload to URL: Downloads the PNG from hcti.io and uploads it to a CDN to generate a mandatory public HTTPS URL, as Instagram's API rejects binary or base64 data.<br>Code – Merge Fields: Unifies the new CDN image URL with the pre-built caption and associated metadata.<br>HTTP – IG Create Media Container: Submits the CDN URL and caption to the Instagram Graph API, returning a container_id required for final publishing. |
| IG Create Media Container | n8n-nodes-base.httpRequest | Create Instagram media container | Code Merge Fields | Wait IG Buffer | Upload to URL: Downloads the PNG from hcti.io and uploads it to a CDN to generate a mandatory public HTTPS URL, as Instagram's API rejects binary or base64 data.<br>Code – Merge Fields: Unifies the new CDN image URL with the pre-built caption and associated metadata.<br>HTTP – IG Create Media Container: Submits the CDN URL and caption to the Instagram Graph API, returning a container_id required for final publishing. |
| Wait IG Buffer | n8n-nodes-base.wait | Delay before publishing Instagram media | IG Create Media Container | IG Publish Container | 🚀 Finalization & Team Notification<br>Wait 6s: Provides a processing buffer to ensure Instagram finishes validating the media container before the final call.<br>HTTP – IG Publish Container: Executes the /media_publish call using the container_id and retrieves the official live Post ID.<br>Google Sheets – Mark as Posted: Updates the specific row by setting Posted to TRUE and logging the Post ID and timestamp to prevent duplicate entries.<br>Slack – Notify Team: Dispatches an alert containing the reviewer's name, a snippet of the text, and the final card URL for internal verification. |
| IG Publish Container | n8n-nodes-base.httpRequest | Publish Instagram media container | Wait IG Buffer | Google Sheets Mark as Posted | 🚀 Finalization & Team Notification<br>Wait 6s: Provides a processing buffer to ensure Instagram finishes validating the media container before the final call.<br>HTTP – IG Publish Container: Executes the /media_publish call using the container_id and retrieves the official live Post ID.<br>Google Sheets – Mark as Posted: Updates the specific row by setting Posted to TRUE and logging the Post ID and timestamp to prevent duplicate entries.<br>Slack – Notify Team: Dispatches an alert containing the reviewer's name, a snippet of the text, and the final card URL for internal verification. |
| Google Sheets Mark as Posted | n8n-nodes-base.googleSheets | Update source row after successful publish | IG Publish Container | Slack Notify Team | 🚀 Finalization & Team Notification<br>Wait 6s: Provides a processing buffer to ensure Instagram finishes validating the media container before the final call.<br>HTTP – IG Publish Container: Executes the /media_publish call using the container_id and retrieves the official live Post ID.<br>Google Sheets – Mark as Posted: Updates the specific row by setting Posted to TRUE and logging the Post ID and timestamp to prevent duplicate entries.<br>Slack – Notify Team: Dispatches an alert containing the reviewer's name, a snippet of the text, and the final card URL for internal verification. |
| Slack Notify Team | n8n-nodes-base.slack | Notify team of successful publication | Google Sheets Mark as Posted |  | 🚀 Finalization & Team Notification<br>Wait 6s: Provides a processing buffer to ensure Instagram finishes validating the media container before the final call.<br>HTTP – IG Publish Container: Executes the /media_publish call using the container_id and retrieves the official live Post ID.<br>Google Sheets – Mark as Posted: Updates the specific row by setting Posted to TRUE and logging the Post ID and timestamp to prevent duplicate entries.<br>Slack – Notify Team: Dispatches an alert containing the reviewer's name, a snippet of the text, and the final card URL for internal verification. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence for n8n.

## Prerequisites
Before building the workflow, prepare:

1. **Environment variables**
   - `IG_USER_ID`
   - `IG_ACCESS_TOKEN`
   - `GSHEET_DOCUMENT_ID`
   - `SLACK_CHANNEL_ID`
   - `BRAND_NAME`
   - `CARD_BG_COLOR`
   - `CARD_TEXT_COLOR`
   - `CARD_ACCENT_COLOR`

2. **Accounts / credentials**
   - Google Sheets credentials in n8n
   - Slack OAuth2 credentials in n8n
   - hcti.io credentials as HTTP Basic Auth
   - Instagram Graph API access token with publishing permissions
   - Upload to URL community node installed and configured if required by your n8n instance

3. **Google Sheet structure**
   Create a sheet with at least these columns:
   - `Reviewer Name`
   - `Review Text`
   - `Rating`
   - `Source`
   - `Posted`
   - `Submitted At`
   - `Instagram Post ID`
   - `Posted At`
   - `Card Image URL`

---

## Step-by-step build

1. **Create a new workflow**
   - Name it: **Auto-generate Instagram posts from Google Sheets reviews using UploadtoUrl**

2. **Add a Schedule Trigger node**
   - Type: **Schedule Trigger**
   - Configure it to run daily at **9 AM**
   - Save the node

3. **Add a Google Sheets node named `Google Sheets Fetch Review`**
   - Type: **Google Sheets**
   - Operation: **Read Rows**
   - Credentials: your Google Sheets credential
   - Document ID: `{{$env.GSHEET_DOCUMENT_ID}}`
   - Select the correct sheet/tab containing reviews
   - Configure filtering so it returns only:
     - `Rating = 5`
     - `Posted = FALSE`
   - Configure sorting:
     - by `Submitted At`
     - ascending
   - Limit results to **1 row**
   - If possible, preserve a row number or unique row identifier for later update

4. **Connect `Schedule Trigger` → `Google Sheets Fetch Review`**

5. **Add an IF node named `IF Has Valid Review`**
   - Condition type: string
   - Check that `{{$json['Reviewer Name']}}` is **not empty**
   - Leave the false branch unconnected

6. **Connect `Google Sheets Fetch Review` → `IF Has Valid Review`**

7. **Add a Code node named `Code Prepare Fields`**
   - Paste this logic equivalent:
     - Read the first input item
     - Extract reviewer, review text, rating, source, and row identifier
     - Default missing values where appropriate
     - Throw an error if review text is empty
     - Truncate review text to:
       - 200 chars for card
       - 280 chars for caption quote segment
     - Clamp rating to 1–5
     - Build a caption with:
       - rating text
       - review quote
       - reviewer attribution
       - CTA
       - hashtags
     - Return:
       - `record_id`
       - `reviewer_name`
       - `review_text`
       - `card_text`
       - `rating`
       - `star_count`
       - `source`
       - `final_caption`
       - `ig_user_id: $env.IG_USER_ID`
   - Use the same field naming flexibility as the original if desired

8. **Connect `IF Has Valid Review` true output → `Code Prepare Fields`**

9. **Add a Set node named `Set Assemble HTML Card`**
   - Create a field `html` of type string
   - Build a full HTML document sized to **1080x1080**
   - Include CSS for:
     - background color from `CARD_BG_COLOR`
     - text color from `CARD_TEXT_COLOR`
     - accent color from `CARD_ACCENT_COLOR`
   - Include dynamic content:
     - stars using `&#9733;`.repeat(star count)
     - quote text from `card_text`
     - reviewer name
     - source
     - brand name from `BRAND_NAME`
   - Optionally create a second field `passFields` containing the entire incoming JSON

10. **Connect `Code Prepare Fields` → `Set Assemble HTML Card`**

11. **Add an HTTP Request node named `HTTP Render HTML to Image`**
   - Method: `POST`
   - URL: `https://hcti.io/v1/image`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Basic Auth**
   - Choose your hcti.io credential
   - Send body: enabled
   - Body parameter:
     - `html` = `{{$json.html}}`

12. **Connect `Set Assemble HTML Card` → `HTTP Render HTML to Image`**

13. **Add an `Upload to URL` node**
   - Type: **Upload to URL** community node
   - Name it `Upload to URL`
   - Configure it according to your installed node’s expectations
   - If it accepts a source URL, map the image URL returned by hcti.io
   - If it expects binary input, insert an intermediate download step before this node
   - Ensure the output contains a public HTTPS URL

14. **Connect `HTTP Render HTML to Image` → `Upload to URL`**

15. **Add a Code node named `Code Merge Fields`**
   - Logic:
     - Read output of the Upload to URL node
     - Read prepared metadata from `Code Prepare Fields`
     - Detect the public image URL from one of:
       - `public_url`
       - `url`
       - `file_url`
       - `cdn_url`
       - `imageUrl`
     - Throw an error if none exists
     - Return:
       - `public_image_url`
       - `final_caption`
       - `ig_user_id`
       - `record_id`
       - `reviewer_name`
       - `review_text`
       - `source`

16. **Connect `Upload to URL` → `Code Merge Fields`**

17. **Add an HTTP Request node named `IG Create Media Container`**
   - Method: `POST`
   - URL:
     - `https://graph.facebook.com/v19.0/{{$json.ig_user_id}}/media`
   - Send query parameters:
     - `image_url` = `{{$json.public_image_url}}`
     - `caption` = `{{$json.final_caption}}`
     - `access_token` = `{{$env.IG_ACCESS_TOKEN}}`

18. **Connect `Code Merge Fields` → `IG Create Media Container`**

19. **Add a Wait node named `Wait IG Buffer`**
   - Wait amount: `6`
   - Unit: `seconds`

20. **Connect `IG Create Media Container` → `Wait IG Buffer`**

21. **Add an HTTP Request node named `IG Publish Container`**
   - Method: `POST`
   - URL:
     - `https://graph.facebook.com/v19.0/{{$("Code Merge Fields").first().json.ig_user_id}}/media_publish`
   - Send query parameters:
     - `creation_id` = `{{$("IG Create Media Container").first().json.id}}`
     - `access_token` = `{{$env.IG_ACCESS_TOKEN}}`

22. **Connect `Wait IG Buffer` → `IG Publish Container`**

23. **Add a Google Sheets node named `Google Sheets Mark as Posted`**
   - Type: **Google Sheets**
   - Operation: **Update Row**
   - Credentials: same Google Sheets credential
   - Document ID: `{{$env.GSHEET_DOCUMENT_ID}}`
   - Select the same review sheet
   - Configure row matching using a reliable identifier:
     - preferred: row number or unique key preserved from the fetch step
   - Update at least:
     - `Posted` = `TRUE`
     - `Instagram Post ID` = published Instagram ID
     - `Posted At` = current timestamp
     - `Card Image URL` = `public_image_url`
   - If your fetch node does not preserve row identity automatically, add a row number column in the sheet and carry it through the workflow

24. **Connect `IG Publish Container` → `Google Sheets Mark as Posted`**

25. **Add a Slack node named `Slack Notify Team`**
   - Type: **Slack**
   - Authentication: **OAuth2**
   - Select operation to send a channel message
   - Channel ID: `{{$env.SLACK_CHANNEL_ID}}`
   - Message text:
     - Reviewer name
     - Source
     - Instagram post ID
     - Review snippet
     - Card URL
   - Equivalent message format:
     - `Social Proof Post Published!`
     - `Reviewer: {{$("Code Merge Fields").first().json.reviewer_name}}`
     - `Source: {{$("Code Merge Fields").first().json.source}}`
     - `Post ID: {{$("IG Publish Container").first().json.id}}`
     - `Review: {{$("Code Merge Fields").first().json.review_text.substring(0, 100)}}`
     - `Card URL: {{$("Code Merge Fields").first().json.public_image_url}}`

26. **Connect `Google Sheets Mark as Posted` → `Slack Notify Team`**

27. **Optionally add sticky notes**
   - Add the overview and grouped block notes for maintainability

28. **Test the workflow manually**
   - Use a single known valid row in Google Sheets
   - Confirm:
     - fetch works
     - card renders
     - upload returns a public HTTPS URL
     - Instagram container is created
     - publish succeeds
     - sheet is updated correctly
     - Slack message is sent

29. **Activate the workflow**
   - Once all credentials and row update logic are verified, enable the workflow

---

## Important implementation notes while rebuilding

### Google Sheets fetch/update gap
The JSON does not include visible filter, sort, limit, tab selection, or row update mapping parameters, even though the sticky notes describe them. Those settings must be explicitly configured during rebuild.

### Row identity is critical
The workflow tries to preserve `record_id`, but Google Sheets commonly needs either:
- row number,
- unique column value,
- or dedicated matching settings.

Without that, `Google Sheets Mark as Posted` cannot safely update the original review row.

### Upload to URL node behavior may differ
Because this is a community node, its expected input and returned fields may vary by version. Validate:
- whether it accepts a remote image URL from hcti.io directly,
- whether it needs binary data,
- and what property contains the final public URL.

### Instagram API prerequisites
The workflow assumes:
- an Instagram Business or Creator account,
- proper Facebook Page linking,
- Graph API permissions for content publishing,
- valid long-lived access token.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Customer Review to Social Proof – Instagram Post. Automatically fetches the oldest unposted 5-star review from Google Sheets, generates a branded quote card using hcti.io, uploads it via UploadToURL to get a public CDN link, and publishes it to Instagram with a caption. Marks the review as posted and notifies the team on Slack. | Workflow purpose |
| Flow: Schedule → Fetch review → Validate → Prepare caption → Generate image → Upload → Create IG post → Publish → Update sheet → Notify Slack | Process summary |
| Required environment variables: IG_USER_ID, IG_ACCESS_TOKEN, GSHEET_DOCUMENT_ID, SLACK_CHANNEL_ID, BRAND_NAME, CARD_BG_COLOR, CARD_TEXT_COLOR, CARD_ACCENT_COLOR | Deployment/configuration |
| Sheet columns expected: Reviewer Name, Review Text, Rating, Source, Posted, Submitted At, Instagram Post ID, Posted At, Card Image URL | Source data contract |