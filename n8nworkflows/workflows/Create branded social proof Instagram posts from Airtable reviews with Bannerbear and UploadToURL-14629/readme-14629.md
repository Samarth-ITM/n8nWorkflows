Create branded social proof Instagram posts from Airtable reviews with Bannerbear and UploadToURL

https://n8nworkflows.xyz/workflows/create-branded-social-proof-instagram-posts-from-airtable-reviews-with-bannerbear-and-uploadtourl-14629


# Create branded social proof Instagram posts from Airtable reviews with Bannerbear and UploadToURL

# 1. Workflow Overview

This workflow automates the creation and publication of branded Instagram social proof posts from Airtable reviews.

Its main purpose is to:
- run on a daily schedule,
- retrieve the oldest eligible 5-star review from Airtable,
- generate a branded quote card image with Bannerbear,
- convert that image into a public direct URL using Upload to URL,
- publish the image post to Instagram through the Facebook Graph API,
- mark the Airtable record as posted,
- notify the team in Slack.

Typical use cases:
- customer testimonial publishing,
- automated social proof distribution,
- review repurposing for Instagram marketing,
- low-touch content operations based on an Airtable queue.

## 1.1 Scheduled Intake and Review Selection
The workflow starts every day at 10:00 and queries the `Reviews` table in Airtable. It is intended to retrieve one review record that is eligible for posting.

## 1.2 Review Validation and Payload Preparation
If a valid Airtable record exists, a Code node normalizes the review fields, truncates text where needed, builds Bannerbear layer modifications, and prepares an Instagram caption.

## 1.3 Bannerbear Image Generation
The workflow submits an async image-generation request to Bannerbear, waits briefly, and polls the job status once.

## 1.4 Image Retrieval and Public URL Conversion
Once Bannerbear reports completion, the rendered image is downloaded as binary and uploaded through the Upload to URL node to obtain a public HTTPS asset URL that Instagram accepts.

## 1.5 Instagram Publishing
The workflow creates an Instagram media container using the public image URL and caption, waits for Instagram processing, then publishes the container.

## 1.6 Record Update and Team Notification
After publication, the Airtable review is marked as posted, and a Slack message is sent with the post details.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Intake and Review Selection

**Overview:**  
This block triggers the workflow once per day and fetches candidate review data from Airtable. It acts as the ingestion gate for the whole process.

**Nodes Involved:**  
- `Schedule Trigger1`
- `Airtable Fetch Review1`
- `IF Has Valid Review1`

### Node Details

#### Schedule Trigger1
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node that starts the workflow on a schedule.
- **Configuration choices:**  
  Configured to trigger daily at 10 AM.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input; outputs to `Airtable Fetch Review1`.
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`.
- **Edge cases or potential failure types:**  
  - Workflow will not run unless activated.
  - Timezone behavior depends on workflow/server timezone settings.
- **Sub-workflow reference:**  
  None.

#### Airtable Fetch Review1
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Reads records from Airtable.
- **Configuration choices:**  
  - Operation: `list`
  - Base ID taken from environment variable `AIRTABLE_BASE_ID`
  - Table name: `Reviews`
- **Key expressions or variables used:**  
  - `{{ $env.AIRTABLE_BASE_ID }}`
- **Input and output connections:**  
  Input from `Schedule Trigger1`; output to `IF Has Valid Review1`.
- **Version-specific requirements:**  
  Uses Airtable node `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Missing Airtable credentials
  - Invalid base ID
  - Table name mismatch
  - Rate limits
  - Important design issue: the sticky note says filtering should be applied (`Rating = 5`, `Posted = FALSE()`, sort ascending by `Submitted At`, `maxRecords: 1`), but the actual node configuration shown does **not** include those filters. As configured, it lists records from the table without enforcement of the intended FIFO 5-star unposted logic.
- **Sub-workflow reference:**  
  None.

#### IF Has Valid Review1
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether a usable Airtable record exists.
- **Configuration choices:**  
  Tests whether `{{$json.id}}` is not empty.
- **Key expressions or variables used:**  
  - `{{ $json.id }}`
- **Input and output connections:**  
  Input from `Airtable Fetch Review1`; true output goes to `Code Prepare Bannerbear Payload1`. No false-branch node is connected.
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - If Airtable returns no items, the node simply produces no downstream execution.
  - Since no false branch is connected, the workflow stops silently for invalid or empty input.
  - If Airtable returns multiple records, each matching item would continue independently.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Review Validation and Payload Preparation

**Overview:**  
This block converts Airtable review data into structured content for both Bannerbear and Instagram. It handles field extraction, text truncation, and caption assembly.

**Nodes Involved:**  
- `Code Prepare Bannerbear Payload1`

### Node Details

#### Code Prepare Bannerbear Payload1
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node used to map Airtable review data into downstream payloads.
- **Configuration choices:**  
  The script:
  - reads the first input item,
  - extracts Airtable fields from `review.fields` or direct JSON,
  - sets defaults,
  - truncates review text for image and caption usage,
  - builds Bannerbear `modifications`,
  - prepares an Instagram caption,
  - returns metadata for later nodes.
- **Key expressions or variables used:**  
  Fields accessed:
  - `Reviewer Name`
  - `Review Text`
  - `Rating`
  - Airtable record `id`

  Environment variables:
  - `$env.IG_USER_ID`
  - `$env.BANNERBEAR_TEMPLATE_ID`

  Output fields include:
  - `record_id`
  - `reviewer_name`
  - `review_text`
  - `card_text`
  - `stars`
  - `modifications`
  - `final_caption`
  - `ig_user_id`
  - `bannerbear_template_id`
- **Important implementation note:**  
  The code currently contains:
  ```js
  const stars = ''.repeat(Math.min(rating, 5));
  ```
  This produces an empty string, regardless of rating. The intended behavior was almost certainly something like:
  ```js
  const stars = '⭐'.repeat(Math.min(rating, 5));
  ```
  As written, the star layer and caption prefix will be blank.
- **Input and output connections:**  
  Input from `IF Has Valid Review1`; output to `HTTP Bannerbear Create Job1`.
- **Version-specific requirements:**  
  Uses Code node `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Missing review fields can lead to empty text content, though defaults are partially provided.
  - Missing environment variables will propagate invalid downstream requests.
  - Long captions are truncated to stay under Instagram’s 2200-character limit.
  - If `review.id` is absent, Airtable update later may fail.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Bannerbear Image Generation

**Overview:**  
This block submits the image generation request to Bannerbear, waits briefly, and checks the render status once. It is the asynchronous image-production segment of the workflow.

**Nodes Involved:**  
- `HTTP Bannerbear Create Job1`
- `Wait Before Poll1`
- `HTTP Bannerbear Poll Status1`
- `IF Image Ready1`

### Node Details

#### HTTP Bannerbear Create Job1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to Bannerbear to create an image-generation job.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.bannerbear.com/v2/images`
  - Raw JSON body sent manually
  - Headers include Bearer auth and JSON content type
- **Key expressions or variables used:**  
  - Body:
    `{{ JSON.stringify({ template: $json.bannerbear_template_id, modifications: $json.modifications }) }}`
  - Header:
    `Bearer {{ $env.BANNERBEAR_API_KEY }}`
- **Input and output connections:**  
  Input from `Code Prepare Bannerbear Payload1`; output to `Wait Before Poll1`.
- **Version-specific requirements:**  
  Uses HTTP Request `typeVersion: 4.1`.
- **Edge cases or potential failure types:**  
  - Missing or invalid Bannerbear API key
  - Invalid template ID
  - Template layer names not matching modification names
  - Malformed JSON body if upstream values are invalid
- **Sub-workflow reference:**  
  None.

#### Wait Before Poll1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays execution for a fixed time before polling Bannerbear.
- **Configuration choices:**  
  Waits `5` seconds.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from `HTTP Bannerbear Create Job1`; output to `HTTP Bannerbear Poll Status1`.
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Five seconds may be insufficient for slower renders.
  - Wait nodes depend on n8n’s execution persistence and resume support.
- **Sub-workflow reference:**  
  None.

#### HTTP Bannerbear Poll Status1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the status of the Bannerbear image job.
- **Configuration choices:**  
  - Method: GET
  - URL built from the `uid` returned by the create-job node
  - Authorization header uses Bannerbear API key
- **Key expressions or variables used:**  
  - URL:
    `https://api.bannerbear.com/v2/images/{{ $('HTTP Bannerbear Create Job1').first().json.uid }}`
  - Header:
    `Bearer {{ $env.BANNERBEAR_API_KEY }}`
- **Input and output connections:**  
  Input from `Wait Before Poll1`; output to `IF Image Ready1`.
- **Version-specific requirements:**  
  Uses HTTP Request `typeVersion: 4.1`.
- **Edge cases or potential failure types:**  
  - If the create-job node failed or returned no `uid`, expression resolution fails.
  - Bannerbear may still return `pending`.
  - Network/API timeouts.
- **Sub-workflow reference:**  
  None.

#### IF Image Ready1
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the Bannerbear render has completed.
- **Configuration choices:**  
  Condition: `{{$json.status}} == "completed"` (case-insensitive compare configured by node options).
- **Key expressions or variables used:**  
  - `{{ $json.status }}`
- **Input and output connections:**  
  Input from `HTTP Bannerbear Poll Status1`; true output goes to `HTTP Fetch Rendered Card1`. No false branch is connected.
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - If status remains `pending`, execution stops because no retry loop is implemented.
  - If Bannerbear returns `failed` or another unexpected status, execution also stops silently.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Image Retrieval and Public URL Conversion

**Overview:**  
This block downloads the rendered Bannerbear image as binary and uploads it to a service that exposes a public URL. It also merges that URL with the prepared caption and metadata.

**Nodes Involved:**  
- `HTTP Fetch Rendered Card1`
- `Upload to URL`
- `Code Merge Upload and Caption1`

### Node Details

#### HTTP Fetch Rendered Card1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed Bannerbear image file.
- **Configuration choices:**  
  - URL comes from `{{$json.image_url}}`
  - Response format set to `file`
  - Binary output property name: `cardImage`
- **Key expressions or variables used:**  
  - `{{ $json.image_url }}`
- **Input and output connections:**  
  Input from `IF Image Ready1`; output to `Upload to URL`.
- **Version-specific requirements:**  
  Uses HTTP Request `typeVersion: 4.1`.
- **Edge cases or potential failure types:**  
  - Missing `image_url`
  - CDN/image download failure
  - Large file or HTTP timeout issues
- **Sub-workflow reference:**  
  None.

#### Upload to URL
- **Type and technical role:** `n8n-nodes-uploadtourl.uploadToUrl`  
  Third-party/community node used to upload binary content and return a public URL.
- **Configuration choices:**  
  - Binary property name: `cardImage`
- **Key expressions or variables used:**  
  None directly in parameters.
- **Input and output connections:**  
  Input from `HTTP Fetch Rendered Card1`; output to `Code Merge Upload and Caption1`.
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.  
  This node is not part of core n8n and must be available in the instance.
- **Edge cases or potential failure types:**  
  - Node package not installed
  - Binary property mismatch
  - Service-side upload failure
  - Return schema may vary by version/provider
- **Sub-workflow reference:**  
  None.

#### Code Merge Upload and Caption1
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes Upload to URL output and combines it with data from the earlier preparation step.
- **Configuration choices:**  
  The script:
  - reads the upload result,
  - fetches prior payload data from `Code Prepare Bannerbear Payload1`,
  - tries several possible output field names for the public image URL,
  - throws an explicit error if none is found.
- **Key expressions or variables used:**  
  It checks for:
  - `public_url`
  - `url`
  - `file_url`
  - `cdn_url`
  - `imageUrl`

  It also reads from:
  - `$('Code Prepare Bannerbear Payload1').first().json`
- **Input and output connections:**  
  Input from `Upload to URL`; output to `IG Create Media Container1`.
- **Version-specific requirements:**  
  Uses Code node `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Upload node returns an unexpected schema
  - Expression lookup to prior node fails if upstream execution path breaks
  - Throws a hard error if no public URL is found
- **Sub-workflow reference:**  
  None.

---

## 2.5 Instagram Publishing

**Overview:**  
This block creates an Instagram media container and publishes it after a short wait. It uses the Facebook Graph API for Instagram Business/Creator posting.

**Nodes Involved:**  
- `IG Create Media Container1`
- `Wait IG Buffer1`
- `IG Publish Container1`

### Node Details

#### IG Create Media Container1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Graph API endpoint to create an Instagram media container from a public image URL.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://graph.facebook.com/v19.0/{{ $json.ig_user_id }}/media`
  - Parameters sent as query string:
    - `image_url`
    - `caption`
    - `access_token`
- **Key expressions or variables used:**  
  - `{{ $json.ig_user_id }}`
  - `{{ $json.public_image_url }}`
  - `{{ $json.final_caption }}`
  - `{{ $env.IG_ACCESS_TOKEN }}`
- **Input and output connections:**  
  Input from `Code Merge Upload and Caption1`; output to `Wait IG Buffer1`.
- **Version-specific requirements:**  
  Uses HTTP Request `typeVersion: 4.1`.
- **Edge cases or potential failure types:**  
  - Invalid or expired IG access token
  - Wrong Instagram user ID
  - Public URL not directly accessible by Meta crawlers
  - Caption too long or unsupported formatting
  - Instagram account not properly linked for API publishing
- **Sub-workflow reference:**  
  None.

#### Wait IG Buffer1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Adds a short delay to allow Instagram media processing before publish.
- **Configuration choices:**  
  Waits `6` seconds.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from `IG Create Media Container1`; output to `IG Publish Container1`.
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Six seconds may not always be enough for processing.
- **Sub-workflow reference:**  
  None.

#### IG Publish Container1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Publishes the previously created Instagram media container.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://graph.facebook.com/v19.0/{ig_user_id}/media_publish`
  - Query parameters:
    - `creation_id`
    - `access_token`
- **Key expressions or variables used:**  
  - `{{ $('Code Merge Upload and Caption1').first().json.ig_user_id }}`
  - `{{ $('IG Create Media Container1').first().json.id }}`
  - `{{ $env.IG_ACCESS_TOKEN }}`
- **Input and output connections:**  
  Input from `Wait IG Buffer1`; output to `Airtable Mark as Posted1`.
- **Version-specific requirements:**  
  Uses HTTP Request `typeVersion: 4.1`.
- **Edge cases or potential failure types:**  
  - Missing or invalid `creation_id`
  - Container not ready yet
  - Graph API permission issues
  - API version differences over time
- **Sub-workflow reference:**  
  None.

---

## 2.6 Record Update and Team Notification

**Overview:**  
This final block writes posting results back to Airtable and alerts the team in Slack. It closes the loop to avoid reposting the same testimonial.

**Nodes Involved:**  
- `Airtable Mark as Posted1`
- `Slack Notify Team1`

### Node Details

#### Airtable Mark as Posted1
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Updates the review record after successful publication.
- **Configuration choices:**  
  - Operation: `update`
  - Base ID from `AIRTABLE_BASE_ID`
  - Table: `Reviews`
  - Updates fields:
    - `Posted = true`
    - `Posted At = {{ new Date().toISOString() }}`
    - `Card Image URL = {{ $('Code Merge Upload and Caption1').first().json.public_image_url }}`
    - `Instagram Post ID = {{ $('IG Publish Container1').first().json.id }}`
- **Key expressions or variables used:**  
  - `{{ $env.AIRTABLE_BASE_ID }}`
  - `{{ new Date().toISOString() }}`
  - `{{ $('Code Merge Upload and Caption1').first().json.public_image_url }}`
  - `{{ $('IG Publish Container1').first().json.id }}`
- **Important implementation note:**  
  The node is configured with `matchingColumns: ["id"]`, but the provided configuration does not clearly show the Airtable record ID being passed in a standard update-record-ID parameter. In practice, this node may need explicit record identification depending on the Airtable node mode/version. Validate carefully during implementation.
- **Input and output connections:**  
  Input from `IG Publish Container1`; output to `Slack Notify Team1`.
- **Version-specific requirements:**  
  Uses Airtable node `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Record ID not supplied correctly
  - Field names differ from Airtable schema
  - Permission or rate-limit errors
- **Sub-workflow reference:**  
  None.

#### Slack Notify Team1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted Slack notification about the published post.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Sends a text message to the Slack channel ID from env var
  - Includes reviewer name, post ID, timestamp, review snippet, and card URL
- **Key expressions or variables used:**  
  - `{{ $('Code Merge Upload and Caption1').first().json.reviewer_name }}`
  - `{{ $('IG Publish Container1').first().json.id }}`
  - `{{ new Date().toLocaleString() }}`
  - `{{ $('Code Merge Upload and Caption1').first().json.review_text.substring(0, 120) }}`
  - `{{ $('Code Merge Upload and Caption1').first().json.public_image_url }}`
  - `{{ $env.SLACK_CHANNEL_ID }}`
- **Input and output connections:**  
  Input from `Airtable Mark as Posted1`; no downstream output.
- **Version-specific requirements:**  
  Uses Slack node `typeVersion: 2.2`.
- **Edge cases or potential failure types:**  
  - Missing Slack OAuth2 credentials
  - Invalid channel ID
  - If `review_text` is shorter than expected this is still safe, but the message always appends `...`
  - Slack permission scopes may be insufficient
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger1 | Schedule Trigger | Starts workflow daily at 10 AM |  | Airtable Fetch Review1 | ### Nodes 1-3: Schedule, Fetch & Gate<br>**Schedule Trigger** fires daily at 10 AM.<br>**Airtable -- Fetch Review** uses `filterByFormula` to retrieve only the oldest unposted 5-star record. `AND({Rating}=5, {Posted}=FALSE())` + sort by `Submitted At asc` + `maxRecords: 1` ensures FIFO queue dispatch.<br>**IF -- Has Valid Review?** exits cleanly on false branch if no records are found. |
| Airtable Fetch Review1 | Airtable | Retrieves review records from Airtable | Schedule Trigger1 | IF Has Valid Review1 | ### Nodes 1-3: Schedule, Fetch & Gate<br>**Schedule Trigger** fires daily at 10 AM.<br>**Airtable -- Fetch Review** uses `filterByFormula` to retrieve only the oldest unposted 5-star record. `AND({Rating}=5, {Posted}=FALSE())` + sort by `Submitted At asc` + `maxRecords: 1` ensures FIFO queue dispatch.<br>**IF -- Has Valid Review?** exits cleanly on false branch if no records are found. |
| IF Has Valid Review1 | IF | Validates that a review record exists | Airtable Fetch Review1 | Code Prepare Bannerbear Payload1 | ### Nodes 1-3: Schedule, Fetch & Gate<br>**Schedule Trigger** fires daily at 10 AM.<br>**Airtable -- Fetch Review** uses `filterByFormula` to retrieve only the oldest unposted 5-star record. `AND({Rating}=5, {Posted}=FALSE())` + sort by `Submitted At asc` + `maxRecords: 1` ensures FIFO queue dispatch.<br>**IF -- Has Valid Review?** exits cleanly on false branch if no records are found. |
| Code Prepare Bannerbear Payload1 | Code | Builds Bannerbear layer payload and Instagram caption | IF Has Valid Review1 | HTTP Bannerbear Create Job1 | ### Nodes 4-5: Build Payload & Generate Image<br>**Code -- Prepare Payload** formats review data into Bannerbear modifications array, truncates card text to 180 chars, and builds the full Instagram caption (max 2200 chars).<br>**HTTP -- Bannerbear Create Job** POSTs to `/v2/images`. Returns a `uid` for polling. Generation is async -- initial response is `pending`. |
| HTTP Bannerbear Create Job1 | HTTP Request | Creates async Bannerbear image job | Code Prepare Bannerbear Payload1 | Wait Before Poll1 | ### Nodes 4-5: Build Payload & Generate Image<br>**Code -- Prepare Payload** formats review data into Bannerbear modifications array, truncates card text to 180 chars, and builds the full Instagram caption (max 2200 chars).<br>**HTTP -- Bannerbear Create Job** POSTs to `/v2/images`. Returns a `uid` for polling. Generation is async -- initial response is `pending`. |
| Wait Before Poll1 | Wait | Adds initial Bannerbear rendering delay | HTTP Bannerbear Create Job1 | HTTP Bannerbear Poll Status1 | ### Nodes 6-8: Poll Loop<br>**Wait 5s** -- initial render buffer.<br>**HTTP -- Poll Status** calls `GET /v2/images/{uid}`. Returns `status: pending` or `status: completed`.<br>**IF -- Image Ready?** branches on `completed`. False branch exits -- extend with a loop back to Wait for full retry logic.<br>When completed, `image_url` is the rendered PNG CDN link. |
| HTTP Bannerbear Poll Status1 | HTTP Request | Polls Bannerbear render status | Wait Before Poll1 | IF Image Ready1 | ### Nodes 6-8: Poll Loop<br>**Wait 5s** -- initial render buffer.<br>**HTTP -- Poll Status** calls `GET /v2/images/{uid}`. Returns `status: pending` or `status: completed`.<br>**IF -- Image Ready?** branches on `completed`. False branch exits -- extend with a loop back to Wait for full retry logic.<br>When completed, `image_url` is the rendered PNG CDN link. |
| IF Image Ready1 | IF | Checks whether Bannerbear render is completed | HTTP Bannerbear Poll Status1 | HTTP Fetch Rendered Card1 | ### Nodes 6-8: Poll Loop<br>**Wait 5s** -- initial render buffer.<br>**HTTP -- Poll Status** calls `GET /v2/images/{uid}`. Returns `status: pending` or `status: completed`.<br>**IF -- Image Ready?** branches on `completed`. False branch exits -- extend with a loop back to Wait for full retry logic.<br>When completed, `image_url` is the rendered PNG CDN link. |
| HTTP Fetch Rendered Card1 | HTTP Request | Downloads completed Bannerbear image as binary | IF Image Ready1 | Upload to URL | ### Nodes 9-11: Fetch Binary  Upload to URL  Merge<br>**HTTP Fetch Rendered Card** downloads the Bannerbear image as a binary (`responseFormat: file`, stored in `cardImage`).<br>**Upload to URL** -- the mandatory CDN bridge. Instagram's API requires a direct public HTTPS image URL; it rejects base64 and binary. This node uploads the binary and returns a `public_url`.<br>**Code -- Merge** combines the CDN URL with caption and metadata from the earlier Code node. |
| Upload to URL | UploadToURL | Uploads binary image and returns a public URL | HTTP Fetch Rendered Card1 | Code Merge Upload and Caption1 | ### Nodes 9-11: Fetch Binary  Upload to URL  Merge<br>**HTTP Fetch Rendered Card** downloads the Bannerbear image as a binary (`responseFormat: file`, stored in `cardImage`).<br>**Upload to URL** -- the mandatory CDN bridge. Instagram's API requires a direct public HTTPS image URL; it rejects base64 and binary. This node uploads the binary and returns a `public_url`.<br>**Code -- Merge** combines the CDN URL with caption and metadata from the earlier Code node. |
| Code Merge Upload and Caption1 | Code | Merges uploaded public image URL with prepared caption data | Upload to URL | IG Create Media Container1 | ### Nodes 9-11: Fetch Binary  Upload to URL  Merge<br>**HTTP Fetch Rendered Card** downloads the Bannerbear image as a binary (`responseFormat: file`, stored in `cardImage`).<br>**Upload to URL** -- the mandatory CDN bridge. Instagram's API requires a direct public HTTPS image URL; it rejects base64 and binary. This node uploads the binary and returns a `public_url`.<br>**Code -- Merge** combines the CDN URL with caption and metadata from the earlier Code node. |
| IG Create Media Container1 | HTTP Request | Creates Instagram media container | Code Merge Upload and Caption1 | Wait IG Buffer1 | ### Nodes 12-16: Publish, Log & Notify<br>**IG Create Media Container** POSTs to `/media` with CDN URL + caption. Returns `container_id`.<br>**Wait 6s** -- IG processing buffer.<br>**IG Publish Container** calls `/media_publish` with `creation_id`. Returns live Post ID.<br>**Airtable -- Mark as Posted** updates record: `Posted=true`, Post ID, timestamp, card URL. Prevents re-posting.<br>**Slack -- Notify Team** sends reviewer name, review snippet, Post ID, and card URL. |
| Wait IG Buffer1 | Wait | Delays before Instagram publish call | IG Create Media Container1 | IG Publish Container1 | ### Nodes 12-16: Publish, Log & Notify<br>**IG Create Media Container** POSTs to `/media` with CDN URL + caption. Returns `container_id`.<br>**Wait 6s** -- IG processing buffer.<br>**IG Publish Container** calls `/media_publish` with `creation_id`. Returns live Post ID.<br>**Airtable -- Mark as Posted** updates record: `Posted=true`, Post ID, timestamp, card URL. Prevents re-posting.<br>**Slack -- Notify Team** sends reviewer name, review snippet, Post ID, and card URL. |
| IG Publish Container1 | HTTP Request | Publishes Instagram media container | Wait IG Buffer1 | Airtable Mark as Posted1 | ### Nodes 12-16: Publish, Log & Notify<br>**IG Create Media Container** POSTs to `/media` with CDN URL + caption. Returns `container_id`.<br>**Wait 6s** -- IG processing buffer.<br>**IG Publish Container** calls `/media_publish` with `creation_id`. Returns live Post ID.<br>**Airtable -- Mark as Posted** updates record: `Posted=true`, Post ID, timestamp, card URL. Prevents re-posting.<br>**Slack -- Notify Team** sends reviewer name, review snippet, Post ID, and card URL. |
| Airtable Mark as Posted1 | Airtable | Updates review record to prevent reposting | IG Publish Container1 | Slack Notify Team1 | ### Nodes 12-16: Publish, Log & Notify<br>**IG Create Media Container** POSTs to `/media` with CDN URL + caption. Returns `container_id`.<br>**Wait 6s** -- IG processing buffer.<br>**IG Publish Container** calls `/media_publish` with `creation_id`. Returns live Post ID.<br>**Airtable -- Mark as Posted** updates record: `Posted=true`, Post ID, timestamp, card URL. Prevents re-posting.<br>**Slack -- Notify Team** sends reviewer name, review snippet, Post ID, and card URL. |
| Slack Notify Team1 | Slack | Sends publication notification to Slack | Airtable Mark as Posted1 |  | ### Nodes 12-16: Publish, Log & Notify<br>**IG Create Media Container** POSTs to `/media` with CDN URL + caption. Returns `container_id`.<br>**Wait 6s** -- IG processing buffer.<br>**IG Publish Container** calls `/media_publish` with `creation_id`. Returns live Post ID.<br>**Airtable -- Mark as Posted** updates record: `Posted=true`, Post ID, timestamp, card URL. Prevents re-posting.<br>**Slack -- Notify Team** sends reviewer name, review snippet, Post ID, and card URL. |
| Template Overview1 | Sticky Note | Global workflow description and requirements |  |  | ## Branded Social Proof Automation<br>### Bannerbear + Upload to URL Instagram<br><br>**What this does:**<br>Daily schedule fetches the oldest unposted 5-star review from Airtable, generates a branded quote card via Bannerbear, uploads it via Upload to URL to get a public CDN link, then publishes to Instagram. Marks the review as posted and notifies Slack.<br><br>**Flow:**<br>1. Schedule -- daily at 10 AM<br>2. Airtable -- fetch oldest unposted 5 review<br>3. IF -- skip gracefully if none found<br>4. Code -- build Bannerbear payload + IG caption<br>5. HTTP -- Bannerbear: submit image generation job<br>6. Wait 5s -- initial render buffer<br>7. HTTP -- Bannerbear: poll for completed status<br>8. IF -- completed vs still pending<br>9. HTTP -- fetch rendered image as binary<br>10. Upload to URL -- upload binary, get public CDN URL<br>11. Code -- merge CDN URL + caption fields<br>12. HTTP -- IG: create media container<br>13. Wait 6s -- IG processing buffer<br>14. HTTP -- IG: publish container<br>15. Airtable -- mark review as posted<br>16. Slack -- notify team<br><br>**Required env vars:**<br>`IG_USER_ID` `IG_ACCESS_TOKEN`<br>`BANNERBEAR_API_KEY` `BANNERBEAR_TEMPLATE_ID`<br>`AIRTABLE_BASE_ID` `SLACK_CHANNEL_ID`<br><br>**Airtable Reviews table fields:**<br>`Reviewer Name` . `Review Text` . `Rating`<br>`Posted` (checkbox) . `Submitted At` (date)<br>`Instagram Post ID` . `Posted At` . `Card Image URL`<br><br>**Bannerbear template layer names:**<br>`reviewer_name` . `review_text` . `star_label` |
| Sticky Nodes 1- | Sticky Note | Notes for schedule/fetch/gate block |  |  | ### Nodes 1-3: Schedule, Fetch & Gate<br>**Schedule Trigger** fires daily at 10 AM.<br>**Airtable -- Fetch Review** uses `filterByFormula` to retrieve only the oldest unposted 5-star record. `AND({Rating}=5, {Posted}=FALSE())` + sort by `Submitted At asc` + `maxRecords: 1` ensures FIFO queue dispatch.<br>**IF -- Has Valid Review?** exits cleanly on false branch if no records are found. |
| Sticky Nodes 4- | Sticky Note | Notes for payload/image job block |  |  | ### Nodes 4-5: Build Payload & Generate Image<br>**Code -- Prepare Payload** formats review data into Bannerbear modifications array, truncates card text to 180 chars, and builds the full Instagram caption (max 2200 chars).<br>**HTTP -- Bannerbear Create Job** POSTs to `/v2/images`. Returns a `uid` for polling. Generation is async -- initial response is `pending`. |
| Sticky Nodes 6- | Sticky Note | Notes for Bannerbear polling block |  |  | ### Nodes 6-8: Poll Loop<br>**Wait 5s** -- initial render buffer.<br>**HTTP -- Poll Status** calls `GET /v2/images/{uid}`. Returns `status: pending` or `status: completed`.<br>**IF -- Image Ready?** branches on `completed`. False branch exits -- extend with a loop back to Wait for full retry logic.<br>When completed, `image_url` is the rendered PNG CDN link. |
| Sticky Nodes 9- | Sticky Note | Notes for file retrieval/upload/merge block |  |  | ### Nodes 9-11: Fetch Binary  Upload to URL  Merge<br>**HTTP Fetch Rendered Card** downloads the Bannerbear image as a binary (`responseFormat: file`, stored in `cardImage`).<br>**Upload to URL** -- the mandatory CDN bridge. Instagram's API requires a direct public HTTPS image URL; it rejects base64 and binary. This node uploads the binary and returns a `public_url`.<br>**Code -- Merge** combines the CDN URL with caption and metadata from the earlier Code node. |
| Sticky Nodes 12- | Sticky Note | Notes for publish/log/notify block |  |  | ### Nodes 12-16: Publish, Log & Notify<br>**IG Create Media Container** POSTs to `/media` with CDN URL + caption. Returns `container_id`.<br>**Wait 6s** -- IG processing buffer.<br>**IG Publish Container** calls `/media_publish` with `creation_id`. Returns live Post ID.<br>**Airtable -- Mark as Posted** updates record: `Posted=true`, Post ID, timestamp, card URL. Prevents re-posting.<br>**Slack -- Notify Team** sends reviewer name, review snippet, Post ID, and card URL. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Create branded social proof Instagram posts from Airtable reviews with Bannerbear and UploadToURL`.

2. **Create a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run once per day at `10:00`.
   - Confirm the workflow timezone is correct for your business schedule.

3. **Create an Airtable node for fetching reviews**
   - Node type: `Airtable`
   - Operation: `List`
   - Base: use expression `{{ $env.AIRTABLE_BASE_ID }}`
   - Table: `Reviews`
   - Configure Airtable credentials.
   - To match the intended behavior described in the notes, also configure:
     - `filterByFormula`: `AND({Rating}=5, {Posted}=FALSE())`
     - sort by `Submitted At` ascending
     - `maxRecords = 1`
   - Connect `Schedule Trigger` → `Airtable Fetch Review`.

4. **Create an IF node to validate that a record was returned**
   - Node type: `IF`
   - Condition:
     - left value: `{{ $json.id }}`
     - operator: `is not empty`
   - Connect `Airtable Fetch Review` → `IF Has Valid Review`.
   - Leave false branch unconnected if you want silent termination, or connect it to a logging/notification node if you want visibility when no review is available.

5. **Create a Code node to prepare the Bannerbear payload and caption**
   - Node type: `Code`
   - Connect from the true output of `IF Has Valid Review`.
   - Use JavaScript to:
     - access Airtable fields,
     - extract:
       - `Reviewer Name`
       - `Review Text`
       - `Rating`
       - record `id`
     - build a shortened `card_text` capped around 180 characters,
     - build Bannerbear `modifications`,
     - build an Instagram caption capped under 2200 chars,
     - pass through:
       - `record_id`
       - `reviewer_name`
       - `review_text`
       - `final_caption`
       - `ig_user_id`
       - `bannerbear_template_id`
   - Required env vars in this node:
     - `IG_USER_ID`
     - `BANNERBEAR_TEMPLATE_ID`
   - Important fix recommended:
     - use `'⭐'.repeat(Math.min(rating, 5))` for the star string, not `''.repeat(...)`.

6. **Create an HTTP Request node to submit a Bannerbear image job**
   - Node type: `HTTP Request`
   - Method: `POST`
   - URL: `https://api.bannerbear.com/v2/images`
   - Send headers:
     - `Authorization: Bearer {{ $env.BANNERBEAR_API_KEY }}`
     - `Content-Type: application/json`
   - Send raw JSON body:
     - include `template` from `bannerbear_template_id`
     - include `modifications` from the Code node
   - Connect `Code Prepare Bannerbear Payload` → `HTTP Bannerbear Create Job`.

7. **Create a Wait node for initial Bannerbear render delay**
   - Node type: `Wait`
   - Delay: `5 seconds`
   - Connect `HTTP Bannerbear Create Job` → `Wait Before Poll`.

8. **Create an HTTP Request node to poll Bannerbear job status**
   - Node type: `HTTP Request`
   - Method: `GET`
   - URL expression:  
     `https://api.bannerbear.com/v2/images/{{ $('HTTP Bannerbear Create Job').first().json.uid }}`
   - Header:
     - `Authorization: Bearer {{ $env.BANNERBEAR_API_KEY }}`
   - Connect `Wait Before Poll` → `HTTP Bannerbear Poll Status`.

9. **Create an IF node to verify image completion**
   - Node type: `IF`
   - Condition:
     - left value: `{{ $json.status }}`
     - equals: `completed`
   - Connect `HTTP Bannerbear Poll Status` → `IF Image Ready`.
   - True branch continues.
   - Optional enhancement: connect the false branch back into another wait/poll cycle for robust retry behavior.

10. **Create an HTTP Request node to download the rendered image**
    - Node type: `HTTP Request`
    - URL: `{{ $json.image_url }}`
    - Response format: `File`
    - Output binary property: `cardImage`
    - Connect true output of `IF Image Ready` → `HTTP Fetch Rendered Card`.

11. **Install and create the Upload to URL node**
    - Node type: `Upload to URL`
    - This is a non-core/community node; install it in your n8n instance first if unavailable.
    - Set binary property name to `cardImage`.
    - Connect `HTTP Fetch Rendered Card` → `Upload to URL`.

12. **Create a Code node to merge the uploaded public URL with the prepared caption**
    - Node type: `Code`
    - Connect `Upload to URL` → `Code Merge Upload and Caption`.
    - In the script:
      - read the current upload result,
      - read previous payload data from `Code Prepare Bannerbear Payload`,
      - detect the public URL from fields like:
        - `public_url`
        - `url`
        - `file_url`
        - `cdn_url`
        - `imageUrl`
      - throw an error if no usable URL exists.
    - Return at least:
      - `public_image_url`
      - `final_caption`
      - `ig_user_id`
      - `record_id`
      - `reviewer_name`
      - `review_text`

13. **Create an HTTP Request node to create the Instagram media container**
    - Node type: `HTTP Request`
    - Method: `POST`
    - URL expression:  
      `https://graph.facebook.com/v19.0/{{ $json.ig_user_id }}/media`
    - Send query parameters:
      - `image_url = {{ $json.public_image_url }}`
      - `caption = {{ $json.final_caption }}`
      - `access_token = {{ $env.IG_ACCESS_TOKEN }}`
    - Connect `Code Merge Upload and Caption` → `IG Create Media Container`.

14. **Create a Wait node for Instagram processing**
    - Node type: `Wait`
    - Delay: `6 seconds`
    - Connect `IG Create Media Container` → `Wait IG Buffer`.

15. **Create an HTTP Request node to publish the Instagram media container**
    - Node type: `HTTP Request`
    - Method: `POST`
    - URL expression:  
      `https://graph.facebook.com/v19.0/{{ $('Code Merge Upload and Caption').first().json.ig_user_id }}/media_publish`
    - Query parameters:
      - `creation_id = {{ $('IG Create Media Container').first().json.id }}`
      - `access_token = {{ $env.IG_ACCESS_TOKEN }}`
    - Connect `Wait IG Buffer` → `IG Publish Container`.

16. **Create an Airtable node to mark the review as posted**
    - Node type: `Airtable`
    - Operation: `Update`
    - Base: `{{ $env.AIRTABLE_BASE_ID }}`
    - Table: `Reviews`
    - Configure Airtable credentials.
    - Update the following fields:
      - `Posted = true`
      - `Posted At = {{ new Date().toISOString() }}`
      - `Card Image URL = {{ $('Code Merge Upload and Caption').first().json.public_image_url }}`
      - `Instagram Post ID = {{ $('IG Publish Container').first().json.id }}`
    - Make sure the node correctly identifies the Airtable record to update.
      - Best practice: use the Airtable `record_id` returned from the preparation node as the record identifier.
    - Connect `IG Publish Container` → `Airtable Mark as Posted`.

17. **Create a Slack node for team notification**
    - Node type: `Slack`
    - Authentication: `OAuth2`
    - Configure Slack credentials with permission to post messages.
    - Send message to channel ID: `{{ $env.SLACK_CHANNEL_ID }}`
    - Message should include:
      - reviewer name,
      - Instagram post ID,
      - timestamp,
      - truncated review snippet,
      - card URL.
    - Connect `Airtable Mark as Posted` → `Slack Notify Team`.

18. **Configure required environment variables**
    - `IG_USER_ID`
    - `IG_ACCESS_TOKEN`
    - `BANNERBEAR_API_KEY`
    - `BANNERBEAR_TEMPLATE_ID`
    - `AIRTABLE_BASE_ID`
    - `SLACK_CHANNEL_ID`

19. **Prepare external systems**
    - **Airtable**
      - Base contains a `Reviews` table
      - Expected fields:
        - `Reviewer Name`
        - `Review Text`
        - `Rating`
        - `Posted`
        - `Submitted At`
        - `Instagram Post ID`
        - `Posted At`
        - `Card Image URL`
    - **Bannerbear**
      - Template exists and layer names exactly match:
        - `reviewer_name`
        - `review_text`
        - `star_label`
    - **Instagram / Meta**
      - Instagram Business or Creator account connected through Meta
      - Valid access token with publishing permissions
    - **Slack**
      - OAuth2 app installed and allowed to post to the chosen channel

20. **Add optional robustness improvements**
    - Add retry loop for Bannerbear `pending` status.
    - Add retry loop or status check for Instagram media readiness.
    - Add error notifications to Slack.
    - Add explicit logging if no Airtable review is available.
    - Validate Airtable fetch filtering because the provided JSON does not currently enforce the filtering described in the notes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Branded Social Proof Automation — Bannerbear + Upload to URL Instagram | Global workflow purpose |
| Daily schedule fetches the oldest unposted 5-star review from Airtable, generates a branded quote card via Bannerbear, uploads it to get a public CDN link, publishes to Instagram, marks the review as posted, and notifies Slack. | Workflow summary |
| Required environment variables: `IG_USER_ID`, `IG_ACCESS_TOKEN`, `BANNERBEAR_API_KEY`, `BANNERBEAR_TEMPLATE_ID`, `AIRTABLE_BASE_ID`, `SLACK_CHANNEL_ID` | Deployment/setup |
| Airtable `Reviews` table fields expected: `Reviewer Name`, `Review Text`, `Rating`, `Posted`, `Submitted At`, `Instagram Post ID`, `Posted At`, `Card Image URL` | Data model |
| Bannerbear template layer names must match: `reviewer_name`, `review_text`, `star_label` | Bannerbear template setup |
| The sticky-note documentation says Airtable fetch should filter to the oldest unposted 5-star review, but the actual Airtable node configuration in the JSON does not implement that filter. | Important implementation discrepancy |
| The `stars` string in the Code node is currently built from an empty string, so no stars will appear unless corrected. | Important code issue |
| The workflow contains a single entry point: `Schedule Trigger1`. | Execution model |
| No sub-workflows are used or called by this workflow. | Architecture |