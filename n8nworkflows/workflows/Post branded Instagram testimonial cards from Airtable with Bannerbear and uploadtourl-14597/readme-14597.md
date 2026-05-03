Post branded Instagram testimonial cards from Airtable with Bannerbear and uploadtourl

https://n8nworkflows.xyz/workflows/post-branded-instagram-testimonial-cards-from-airtable-with-bannerbear-and-uploadtourl-14597


# Post branded Instagram testimonial cards from Airtable with Bannerbear and uploadtourl

# 1. Workflow Overview

This workflow automates the creation and publication of branded Instagram testimonial cards from customer reviews stored in Airtable.

Its main purpose is to:
- run on a daily schedule,
- select one eligible 5-star review that has not yet been posted,
- transform that review into a branded visual using Bannerbear,
- make the generated image publicly accessible through Upload to URL,
- publish the result to Instagram via the Graph API,
- update Airtable to prevent reposting,
- and notify the team in Slack.

Typical use cases:
- social proof automation,
- testimonial repurposing for social media,
- review-to-content pipelines,
- low-touch marketing operations for brands collecting reviews in Airtable.

## 1.1 Trigger and Review Selection

The workflow begins with a daily schedule trigger, then queries Airtable for reviews meeting the publishing criteria: 5-star rating and not yet posted.

## 1.2 Review Validation and Payload Preparation

If a valid review exists, the workflow formats the text for both:
- a Bannerbear image-generation payload, and
- an Instagram caption.

This block also normalizes field names and prepares identifiers used later in the workflow.

## 1.3 Bannerbear Image Generation

The workflow submits an image-generation request to Bannerbear using a preconfigured template and dynamic text layers.

## 1.4 Render Polling and Readiness Check

Because Bannerbear image generation is asynchronous, the workflow waits, polls the job status, and checks whether the generated image is complete.

## 1.5 Public URL Bridging and Instagram Media Container Creation

Once the image is ready, the workflow downloads the rendered file, uploads it to a public hosting service using Upload to URL, and then creates an Instagram media container using the returned public URL.

## 1.6 Instagram Publication, Logging, and Notification

After a short buffer, the workflow publishes the Instagram media container, updates the originating Airtable record as posted, and sends a Slack notification.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Review Selection

### Overview
This block starts the automation on a daily cadence and searches Airtable for an eligible review. It ensures only unposted 5-star reviews enter the content generation pipeline.

### Nodes Involved
- Schedule — Daily 10AM
- Airtable — Fetch 5-Star Review
- IF — Has Valid Review?
- No Review — Exit Gracefully

### Node Details

#### Schedule — Daily 10AM
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger node.
- **Configuration choices:** Configured to run daily at 10:00.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Airtable — Fetch 5-Star Review**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Timezone mismatches depending on instance settings.
  - Trigger timing may differ if server timezone is not aligned with business timezone.
- **Sub-workflow reference:** None.

#### Airtable — Fetch 5-Star Review
- **Type and role:** `n8n-nodes-base.airtable`; searches Airtable for candidate records.
- **Configuration choices:**
  - Base ID comes from environment variable `AIRTABLE_BASE_ID`.
  - Table name is `Reviews`.
  - Operation is `search`.
  - Filter formula is `AND({Rating}=5, {Posted}=FALSE())`.
  - Results are sorted by `Submitted At`.
- **Key expressions or variables used:**
  - `{{ $env.AIRTABLE_BASE_ID }}`
- **Input and output connections:** Input from **Schedule — Daily 10AM**; output to **IF — Has Valid Review?**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Invalid Airtable credentials.
  - Missing base, missing table, or missing fields.
  - If `Posted` is not a checkbox/boolean-like field, the formula may fail or return unexpected results.
  - Sorting by `Submitted At` assumes that field exists and is consistently populated.
  - The node description says “latest,” but the sort is only by `Submitted At`; actual result order depends on Airtable search behavior and sort direction.
- **Sub-workflow reference:** None.

#### IF — Has Valid Review?
- **Type and role:** `n8n-nodes-base.if`; checks whether a fetched Airtable item contains a record ID.
- **Configuration choices:**
  - Condition checks that `{{$json.id}}` is not empty.
- **Key expressions or variables used:**
  - `={{ $json.id }}`
- **Input and output connections:**
  - Input from **Airtable — Fetch 5-Star Review**
  - True output to **Code — Prepare Bannerbear Payload**
  - False output to **No Review — Exit Gracefully**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If Airtable returns zero items, downstream execution simply stops unless n8n emits no item to the IF node.
  - If the Airtable node shape changes, the `id` assumption may no longer hold.
  - This node does not re-check the rating; it relies on Airtable filtering.
- **Sub-workflow reference:** None.

#### No Review — Exit Gracefully
- **Type and role:** `n8n-nodes-base.noOp`; terminal branch for runs where no eligible review exists.
- **Configuration choices:** No parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from false branch of **IF — Has Valid Review?**; no output.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; it acts as a clean stop.
- **Sub-workflow reference:** None.

---

## 2.2 Review Validation and Payload Preparation

### Overview
This block transforms Airtable review data into two publishable assets: Bannerbear text layer modifications and a finalized Instagram caption. It also carries forward identifiers needed for record updates and Instagram publishing.

### Nodes Involved
- Code — Prepare Bannerbear Payload

### Node Details

#### Code — Prepare Bannerbear Payload
- **Type and role:** `n8n-nodes-base.code`; data transformation and normalization node.
- **Configuration choices:**
  - Reads the current item as `review`.
  - Uses `review.fields || review` to support either Airtable field nesting or a flatter input structure.
  - Extracts:
    - reviewer name,
    - review text,
    - rating,
    - Airtable record ID.
  - Applies defaults:
    - reviewer fallback: `A Happy Customer`
    - rating fallback: `5`
  - Truncates review text to 180 characters for the image card.
  - Creates a star label string using `⭐`.
  - Builds Bannerbear `modifications` for layers:
    - `reviewer_name`
    - `review_text`
    - `star_label`
  - Builds an Instagram caption with review text, CTA, and hashtags.
  - Enforces Instagram caption limit by truncating at 2200 characters.
  - Injects `ig_account_id` from environment variables.
- **Key expressions or variables used:**
  - `fields['Reviewer Name']`
  - `fields['Review Text']`
  - `fields['Rating']`
  - `$env.IG_ACCOUNT_ID`
- **Input and output connections:** Input from **IF — Has Valid Review?**; output to **HTTP — Bannerbear: Create Image Job**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `review_text` is empty, the image and caption can still be generated but may be poor quality or rejected by downstream services.
  - If `rating` is not numeric, `Math.min(rating, 5)` may behave unexpectedly.
  - Hashtag contains a typo: `#socialprrof`.
  - If the Bannerbear template does not contain layers named exactly `reviewer_name`, `review_text`, and `star_label`, rendering may fail or omit content.
  - If `IG_ACCOUNT_ID` is missing from the environment, Instagram calls later will fail.
- **Sub-workflow reference:** None.

---

## 2.3 Bannerbear Image Generation

### Overview
This block sends the prepared payload to Bannerbear to create a render job. The response provides the asynchronous job identifier used for polling.

### Nodes Involved
- HTTP — Bannerbear: Create Image Job

### Node Details

#### HTTP — Bannerbear: Create Image Job
- **Type and role:** `n8n-nodes-base.httpRequest`; POST request to Bannerbear image-generation API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.bannerbear.com/v2/images`
  - Body format: JSON
  - JSON body includes:
    - `template`: from environment variable `BANNERBEAR_TEMPLATE_ID`
    - `modifications`: from previous code node
  - Sets `Content-Type: application/json`
  - Uses generic header authentication.
- **Key expressions or variables used:**
  - `{{ $env.BANNERBEAR_TEMPLATE_ID }}`
  - `{{ JSON.stringify($json.modifications) }}`
- **Input and output connections:** Input from **Code — Prepare Bannerbear Payload**; output to **Wait — 5s Before Poll**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid or missing Bannerbear API key.
  - Wrong template ID.
  - Template layer mismatch with provided modifications.
  - API quota issues or rate limiting.
  - Credential naming is misleading: it uses a credential called **Stripe Header Auth**, which may work technically if the header is correct, but is confusing operationally.
  - The poll node does not explicitly declare the same credential in the JSON, which may require manual correction after import.
- **Sub-workflow reference:** None.

---

## 2.4 Render Polling and Readiness Check

### Overview
This block waits briefly, polls Bannerbear for job completion, and branches based on whether the rendered image is ready. In the current JSON, the branch exits if the image is not yet ready; the note suggests retries, but no retry loop is actually implemented.

### Nodes Involved
- Wait — 5s Before Poll
- HTTP — Bannerbear: Poll Status
- IF — Image Ready?
- Image Not Ready — Exit with Alert

### Node Details

#### Wait — 5s Before Poll
- **Type and role:** `n8n-nodes-base.wait`; delays execution before first status poll.
- **Configuration choices:** Waits `5` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **HTTP — Bannerbear: Create Image Job**; output to **HTTP — Bannerbear: Poll Status**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Long-running executions may accumulate if many runs overlap.
- **Sub-workflow reference:** None.

#### HTTP — Bannerbear: Poll Status
- **Type and role:** `n8n-nodes-base.httpRequest`; fetches render job status from Bannerbear.
- **Configuration choices:**
  - URL is built using the `uid` returned by **HTTP — Bannerbear: Create Image Job**.
  - Uses generic header authentication.
- **Key expressions or variables used:**
  - `=https://api.bannerbear.com/v2/images/{{ $('HTTP — Bannerbear: Create Image Job').item.json.uid }}`
- **Input and output connections:** Input from **Wait — 5s Before Poll**; output to **IF — Image Ready?**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - If the create-job node did not return `uid`, this expression fails.
  - Missing authentication will return unauthorized errors.
  - Network/API transient errors.
- **Sub-workflow reference:** None.

#### IF — Image Ready?
- **Type and role:** `n8n-nodes-base.if`; checks Bannerbear status.
- **Configuration choices:**
  - Condition: `{{$json.status}} === "completed"`
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input from **HTTP — Bannerbear: Poll Status**
  - True output to **HTTP — Fetch Rendered Card**
  - False output to **Image Not Ready — Exit with Alert**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Incomplete handling of intermediate statuses like `pending`, `processing`, `failed`.
  - Sticky note mentions “Wait 3s + re-poll” and “up to 5 retries,” but this logic is not present in the actual workflow.
- **Sub-workflow reference:** None.

#### Image Not Ready — Exit with Alert
- **Type and role:** `n8n-nodes-base.noOp`; terminal branch when image generation is not complete.
- **Configuration choices:** No parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from false branch of **IF — Image Ready?**; no output.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Despite the node name, there is no actual alerting mechanism.
  - Render jobs that need more than 5 seconds will simply stop here.
- **Sub-workflow reference:** None.

---

## 2.5 Public URL Bridging and Instagram Media Container Creation

### Overview
This block retrieves the rendered image file, uploads it to a public URL service, merges the upload result with prepared caption data, and creates the Instagram media container.

### Nodes Involved
- HTTP — Fetch Rendered Card
- Upload a File
- Code — Merge Upload + Caption Data
- IG — Create Media Container

### Node Details

#### HTTP — Fetch Rendered Card
- **Type and role:** `n8n-nodes-base.httpRequest`; downloads the Bannerbear-rendered image as binary.
- **Configuration choices:**
  - URL is `{{$json.image_url}}`
  - Response format is file
  - Output binary property name is `cardImage`
- **Key expressions or variables used:**
  - `={{ $json.image_url }}`
- **Input and output connections:** Input from true branch of **IF — Image Ready?**; output to **Upload a File**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid `image_url`.
  - Temporary Bannerbear CDN access issues.
  - Very large image size could affect memory or timeout behavior.
- **Sub-workflow reference:** None.

#### Upload a File
- **Type and role:** `n8n-nodes-uploadtourl.uploadToUrl`; uploads the binary file to a hosting/CDN service and returns a public URL.
- **Configuration choices:**
  - Uses Upload to URL API credentials.
  - No explicit parameters are defined in the JSON.
- **Key expressions or variables used:** None visible in the exported parameters.
- **Input and output connections:** Input from **HTTP — Fetch Rendered Card**; output to **Code — Merge Upload + Caption Data**.
- **Version-specific requirements:** Type version `1`; requires the custom/community node package `n8n-nodes-uploadtourl`.
- **Edge cases or potential failure types:**
  - Community node must be installed on the n8n instance.
  - Credential must be valid.
  - Since no visible parameter mapping is included, behavior may depend on node defaults or binary auto-detection.
  - If the node expects a specific binary property name and does not auto-detect `cardImage`, it may fail.
- **Sub-workflow reference:** None.

#### Code — Merge Upload + Caption Data
- **Type and role:** `n8n-nodes-base.code`; combines upload result with previously prepared publishing metadata.
- **Configuration choices:**
  - Reads current upload result.
  - Reads prepared data from **Code — Prepare Bannerbear Payload**.
  - Derives `public_image_url` by checking several possible field names:
    - `public_url`
    - `url`
    - `file_url`
    - `cdn_url`
  - Carries forward:
    - `final_caption`
    - `ig_account_id`
    - `record_id`
    - `reviewer_name`
- **Key expressions or variables used:**
  - `$('Code — Prepare Bannerbear Payload').item.json`
- **Input and output connections:** Input from **Upload a File**; output to **IG — Create Media Container**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If Upload to URL returns a different property name, `public_image_url` may be undefined.
  - No validation is performed before calling Instagram.
- **Sub-workflow reference:** None.

#### IG — Create Media Container
- **Type and role:** `n8n-nodes-base.httpRequest`; creates an Instagram media container as the first step of the publish flow.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://graph.facebook.com/v19.0/{ig_account_id}/media`
  - Sends query parameters:
    - `image_url`
    - `caption`
    - `access_token`
  - Access token comes from credentials object `instagramGraphApi`.
- **Key expressions or variables used:**
  - `={{ $json.public_image_url }}`
  - `={{ $json.final_caption }}`
  - `={{ $credentials.instagramGraphApi.accessToken }}`
- **Input and output connections:** Input from **Code — Merge Upload + Caption Data**; output to **Wait — 6s IG Buffer**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - The credential reference exists in expressions, but no credential block is explicitly attached in the exported node JSON; manual setup may be required.
  - Invalid Instagram Business Account ID.
  - Expired access token.
  - Public image URL not reachable by Meta servers.
  - Caption too long or image format not accepted.
- **Sub-workflow reference:** None.

---

## 2.6 Instagram Publication, Logging, and Notification

### Overview
This block finalizes publication to Instagram, writes publication metadata back to Airtable, and notifies the team in Slack.

### Nodes Involved
- Wait — 6s IG Buffer
- IG — Publish Container
- Airtable — Mark as Posted
- Slack — Notify Team

### Node Details

#### Wait — 6s IG Buffer
- **Type and role:** `n8n-nodes-base.wait`; introduces a delay before publishing the Instagram container.
- **Configuration choices:** Waits `6` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **IG — Create Media Container**; output to **IG — Publish Container**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If Instagram processing takes longer than 6 seconds, publish may fail.
- **Sub-workflow reference:** None.

#### IG — Publish Container
- **Type and role:** `n8n-nodes-base.httpRequest`; publishes the prepared Instagram media container.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://graph.facebook.com/v19.0/{ig_account_id}/media_publish`
  - Sends query parameters:
    - `creation_id` from **IG — Create Media Container**
    - `access_token`
- **Key expressions or variables used:**
  - `={{ $('IG — Create Media Container').item.json.id }}`
  - `={{ $credentials.instagramGraphApi.accessToken }}`
  - `{{ $('Code — Merge Upload + Caption Data').item.json.ig_account_id }}`
- **Input and output connections:** Input from **Wait — 6s IG Buffer**; output to **Airtable — Mark as Posted**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Container not ready yet.
  - Invalid or expired access token.
  - Media container creation may succeed while publish fails.
- **Sub-workflow reference:** None.

#### Airtable — Mark as Posted
- **Type and role:** `n8n-nodes-base.airtable`; updates the source review record to prevent reposting.
- **Configuration choices:**
  - Base ID from `AIRTABLE_BASE_ID`
  - Table `Reviews`
  - Operation `update`
  - Sets fields:
    - `Posted = true`
    - `Posted At = {{ new Date().toISOString() }}`
    - `Card Image URL = public_image_url`
    - `Instagram Post ID = published post id`
- **Key expressions or variables used:**
  - `={{ $env.AIRTABLE_BASE_ID }}`
  - `={{ new Date().toISOString() }}`
  - `={{ $('Code — Merge Upload + Caption Data').item.json.public_image_url }}`
  - `={{ $('IG — Publish Container').item.json.id }}`
- **Input and output connections:** Input from **IG — Publish Container**; output to **Slack — Notify Team**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - The exported JSON does not visibly specify the Airtable record ID to update. In practice, an Airtable update operation usually needs a record identifier or matching strategy. This node may require manual completion in n8n.
  - If target fields do not exist in Airtable, update fails.
  - Date format may not match Airtable field type expectations.
- **Sub-workflow reference:** None.

#### Slack — Notify Team
- **Type and role:** `n8n-nodes-base.slack`; team notification node.
- **Configuration choices:**
  - Sends to a channel whose ID comes from `SLACK_CHANNEL_ID`.
  - Minimal visible parameters in export; message body/content is not defined in the provided JSON.
- **Key expressions or variables used:**
  - `={{ $env.SLACK_CHANNEL_ID }}`
- **Input and output connections:** Input from **Airtable — Mark as Posted**; no output.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Missing Slack credentials.
  - Invalid channel ID.
  - Because no message body is configured in the visible JSON, this node may be incomplete and require manual setup.
- **Sub-workflow reference:** None.

---

## 2.7 Documentation and Annotation Nodes

### Overview
These nodes do not affect execution. They document the workflow, explain each block, and capture prerequisites.

### Nodes Involved
- 📋 Template Overview
- Sticky — Schedule + Fetch + Filter
- Sticky — Build Data + Generate Image
- Sticky — Poll + Status Check
- Sticky — Upload to URL + IG Container
- Sticky — Publish + Log + Notify

### Node Details

#### 📋 Template Overview
- **Type and role:** Sticky note; top-level documentation and prerequisites.
- **Configuration choices:** Large note containing workflow summary, node list, and dependencies.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky — Schedule + Fetch + Filter
- **Type and role:** Sticky note; documents nodes 1–3.
- **Configuration choices:** Explains scheduling, Airtable retrieval, and validity check.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky — Build Data + Generate Image
- **Type and role:** Sticky note; documents payload building and Bannerbear job creation.
- **Configuration choices:** Explains transformation logic and asynchronous job UID return.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky — Poll + Status Check
- **Type and role:** Sticky note; documents polling logic.
- **Configuration choices:** Describes retry behavior that is not actually implemented in the workflow.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky — Upload to URL + IG Container
- **Type and role:** Sticky note; explains why Upload to URL is required before Instagram.
- **Configuration choices:** Documents CDN bridge rationale and naming convention.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky — Publish + Log + Notify
- **Type and role:** Sticky note; documents publishing, Airtable update, and Slack notification.
- **Configuration choices:** Describes expected final behaviors, some of which are more complete than the actual node configurations shown in JSON.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Template Overview | stickyNote | Global documentation and prerequisites |  |  | ## ⭐ Customer Review → Social Proof Instagram Post<br>**What this workflow does:** On a daily schedule, fetches the latest unposted 5-star review from an Airtable reviews base (populated via Typeform, Google Reviews sync, or Trustpilot webhook). Filters for quality, generates a branded quote card image via the Bannerbear API, uploads it via **Upload to URL** to get a public CDN link, then publishes to Instagram with a formatted caption. Marks the review as posted in Airtable and notifies the team on Slack.<br>**Node Summary:** 1. ⏰ Schedule Trigger — runs daily at 10 AM 2. 📋 Airtable — fetch latest unposted 5-star review 3. 🔀 IF — filter: skip if no review found or rating < 5 4. ✍️ Code — build Bannerbear template data + caption 5. 🎨 HTTP — Bannerbear: create image generation job 6. 🔁 HTTP — Bannerbear: poll until image is ready 7. 🔀 IF — check image status = completed 8. ☁️ Upload to URL — upload rendered card, get public CDN URL 9. 📸 HTTP — IG: create media container 10. ⏳ Wait — 6s IG processing buffer 11. ✅ HTTP — IG: publish container 12. 📋 Airtable — mark review as posted 13. 💬 Slack — notify team<br>**Prerequisites:** - Airtable base with a Reviews table (fields: Reviewer Name, Review Text, Rating, Star Label, Posted) - Bannerbear API key + Template ID with quote card design - Instagram Graph API token + Business Account ID - Upload to URL node credentials - Slack API token (optional) |
| Sticky — Schedule + Fetch + Filter | stickyNote | Block annotation |  |  | ### ⏰📋 Nodes 1–3 — Schedule, Fetch Review & Quality Filter<br>Schedule Trigger: Automatically fires daily at 10:00 AM; cadence can be adjusted via cron expression.<br>Airtable — Fetch Review: Retrieves the oldest 5-star, unposted record using a specific filter formula to prevent duplicates.<br>IF — Has Valid Review?: Validates the data; the workflow exits gracefully if no new reviews are found and only proceeds when a 5-star review is ready. |
| Sticky — Build Data + Generate Image | stickyNote | Block annotation |  |  | ### ✍️🎨 Nodes 4–5 — Build Template Data + Generate Image<br>Code — Prepare Payload: Formats review data into a JSON body, mapping fields like name and truncated text to Bannerbear layers while generating the final Instagram caption.<br>HTTP — Create Image Job: Submits the request to the Bannerbear API and retrieves a unique job uid for asynchronous processing. |
| Sticky — Poll + Status Check | stickyNote | Block annotation |  |  | ### 🔁🔀 Nodes 6–7 — Poll Image Status + Branch on Completion<br>HTTP — Poll Status: Regularly checks the job status via the Bannerbear API to see if the rendering is finished.<br>IF — Image Ready?: Confirms completion; if still processing, it triggers a "Wait 3s + re-poll" loop for up to 5 retries before passing the image_url forward. |
| Sticky — Upload to URL + IG Container | stickyNote | Block annotation |  |  | ### ☁️📸 Nodes 8–9 — Upload to URL + Create IG Container<br>**Upload to URL** is the mandatory CDN bridge. It fetches the Bannerbear-rendered image binary and uploads it to the configured hosting endpoint, returning a stable **public URL**. This step is essential — Instagram's Graph API rejects base64 or binary payloads and requires a direct publicly accessible image URL.<br>Filename is auto-set to `review_{reviewer_name}_{timestamp}.jpg` for clean asset tracking.<br>**IG — Create Media Container** is Step 1 of the Instagram 2-step publish flow. POSTs to `/v19.0/{ig_account_id}/media` with the CDN URL and assembled caption. Returns a `container_id` which acts as a staging slot — Instagram validates and pre-processes the image before it goes live. |
| Sticky — Publish + Log + Notify | stickyNote | Block annotation |  |  | ### ⏳✅📋💬 Nodes 10–13 — Publish, Log & Notify<br>**Wait — 6s** gives Instagram time to finalise the media container before the publish call. Slightly longer than typical to account for image card complexity.<br>**IG — Publish Container** calls `/media_publish` with the `container_id`. Returns the live Instagram Post ID confirming the post is on the feed.<br>**Airtable — Mark as Posted** updates the original review record: sets `Posted = true` and writes back the Instagram Post ID and publish timestamp. This prevents the same review from being picked up on the next scheduled run.<br>**Slack — Notify Team** sends a block message with reviewer name, a snippet of the review, the post ID, and the CDN card image URL so the team can preview exactly what went live. |
| Schedule — Daily 10AM | scheduleTrigger | Daily workflow trigger |  | Airtable — Fetch 5-Star Review | ### ⏰📋 Nodes 1–3 — Schedule, Fetch Review & Quality Filter<br>Schedule Trigger: Automatically fires daily at 10:00 AM; cadence can be adjusted via cron expression.<br>Airtable — Fetch Review: Retrieves the oldest 5-star, unposted record using a specific filter formula to prevent duplicates.<br>IF — Has Valid Review?: Validates the data; the workflow exits gracefully if no new reviews are found and only proceeds when a 5-star review is ready. |
| Airtable — Fetch 5-Star Review | airtable | Search eligible Airtable reviews | Schedule — Daily 10AM | IF — Has Valid Review? | ### ⏰📋 Nodes 1–3 — Schedule, Fetch Review & Quality Filter<br>Schedule Trigger: Automatically fires daily at 10:00 AM; cadence can be adjusted via cron expression.<br>Airtable — Fetch Review: Retrieves the oldest 5-star, unposted record using a specific filter formula to prevent duplicates.<br>IF — Has Valid Review?: Validates the data; the workflow exits gracefully if no new reviews are found and only proceeds when a 5-star review is ready. |
| IF — Has Valid Review? | if | Branch on review existence | Airtable — Fetch 5-Star Review | Code — Prepare Bannerbear Payload; No Review — Exit Gracefully | ### ⏰📋 Nodes 1–3 — Schedule, Fetch Review & Quality Filter<br>Schedule Trigger: Automatically fires daily at 10:00 AM; cadence can be adjusted via cron expression.<br>Airtable — Fetch Review: Retrieves the oldest 5-star, unposted record using a specific filter formula to prevent duplicates.<br>IF — Has Valid Review?: Validates the data; the workflow exits gracefully if no new reviews are found and only proceeds when a 5-star review is ready. |
| No Review — Exit Gracefully | noOp | End branch when no candidate review exists | IF — Has Valid Review? |  |  |
| Code — Prepare Bannerbear Payload | code | Build Bannerbear modifications and Instagram caption | IF — Has Valid Review? | HTTP — Bannerbear: Create Image Job | ### ✍️🎨 Nodes 4–5 — Build Template Data + Generate Image<br>Code — Prepare Payload: Formats review data into a JSON body, mapping fields like name and truncated text to Bannerbear layers while generating the final Instagram caption.<br>HTTP — Create Image Job: Submits the request to the Bannerbear API and retrieves a unique job uid for asynchronous processing. |
| HTTP — Bannerbear: Create Image Job | httpRequest | Submit Bannerbear render job | Code — Prepare Bannerbear Payload | Wait — 5s Before Poll | ### ✍️🎨 Nodes 4–5 — Build Template Data + Generate Image<br>Code — Prepare Payload: Formats review data into a JSON body, mapping fields like name and truncated text to Bannerbear layers while generating the final Instagram caption.<br>HTTP — Create Image Job: Submits the request to the Bannerbear API and retrieves a unique job uid for asynchronous processing. |
| Wait — 5s Before Poll | wait | Delay before checking Bannerbear job | HTTP — Bannerbear: Create Image Job | HTTP — Bannerbear: Poll Status | ### 🔁🔀 Nodes 6–7 — Poll Image Status + Branch on Completion<br>HTTP — Poll Status: Regularly checks the job status via the Bannerbear API to see if the rendering is finished.<br>IF — Image Ready?: Confirms completion; if still processing, it triggers a "Wait 3s + re-poll" loop for up to 5 retries before passing the image_url forward. |
| HTTP — Bannerbear: Poll Status | httpRequest | Query Bannerbear render status | Wait — 5s Before Poll | IF — Image Ready? | ### 🔁🔀 Nodes 6–7 — Poll Image Status + Branch on Completion<br>HTTP — Poll Status: Regularly checks the job status via the Bannerbear API to see if the rendering is finished.<br>IF — Image Ready?: Confirms completion; if still processing, it triggers a "Wait 3s + re-poll" loop for up to 5 retries before passing the image_url forward. |
| IF — Image Ready? | if | Branch on Bannerbear completion state | HTTP — Bannerbear: Poll Status | HTTP — Fetch Rendered Card; Image Not Ready — Exit with Alert | ### 🔁🔀 Nodes 6–7 — Poll Image Status + Branch on Completion<br>HTTP — Poll Status: Regularly checks the job status via the Bannerbear API to see if the rendering is finished.<br>IF — Image Ready?: Confirms completion; if still processing, it triggers a "Wait 3s + re-poll" loop for up to 5 retries before passing the image_url forward. |
| Image Not Ready — Exit with Alert | noOp | End branch when Bannerbear output is not ready | IF — Image Ready? |  |  |
| HTTP — Fetch Rendered Card | httpRequest | Download generated image as binary | IF — Image Ready? | Upload a File | ### ☁️📸 Nodes 8–9 — Upload to URL + Create IG Container<br>**Upload to URL** is the mandatory CDN bridge. It fetches the Bannerbear-rendered image binary and uploads it to the configured hosting endpoint, returning a stable **public URL**. This step is essential — Instagram's Graph API rejects base64 or binary payloads and requires a direct publicly accessible image URL.<br>Filename is auto-set to `review_{reviewer_name}_{timestamp}.jpg` for clean asset tracking.<br>**IG — Create Media Container** is Step 1 of the Instagram 2-step publish flow. POSTs to `/v19.0/{ig_account_id}/media` with the CDN URL and assembled caption. Returns a `container_id` which acts as a staging slot — Instagram validates and pre-processes the image before it goes live. |
| Upload a File | n8n-nodes-uploadtourl.uploadToUrl | Upload binary image to public hosting/CDN | HTTP — Fetch Rendered Card | Code — Merge Upload + Caption Data | ### ☁️📸 Nodes 8–9 — Upload to URL + Create IG Container<br>**Upload to URL** is the mandatory CDN bridge. It fetches the Bannerbear-rendered image binary and uploads it to the configured hosting endpoint, returning a stable **public URL**. This step is essential — Instagram's Graph API rejects base64 or binary payloads and requires a direct publicly accessible image URL.<br>Filename is auto-set to `review_{reviewer_name}_{timestamp}.jpg` for clean asset tracking.<br>**IG — Create Media Container** is Step 1 of the Instagram 2-step publish flow. POSTs to `/v19.0/{ig_account_id}/media` with the CDN URL and assembled caption. Returns a `container_id` which acts as a staging slot — Instagram validates and pre-processes the image before it goes live. |
| Code — Merge Upload + Caption Data | code | Combine public image URL with caption and IDs | Upload a File | IG — Create Media Container | ### ☁️📸 Nodes 8–9 — Upload to URL + Create IG Container<br>**Upload to URL** is the mandatory CDN bridge. It fetches the Bannerbear-rendered image binary and uploads it to the configured hosting endpoint, returning a stable **public URL**. This step is essential — Instagram's Graph API rejects base64 or binary payloads and requires a direct publicly accessible image URL.<br>Filename is auto-set to `review_{reviewer_name}_{timestamp}.jpg` for clean asset tracking.<br>**IG — Create Media Container** is Step 1 of the Instagram 2-step publish flow. POSTs to `/v19.0/{ig_account_id}/media` with the CDN URL and assembled caption. Returns a `container_id` which acts as a staging slot — Instagram validates and pre-processes the image before it goes live. |
| IG — Create Media Container | httpRequest | Create Instagram media container | Code — Merge Upload + Caption Data | Wait — 6s IG Buffer | ### ☁️📸 Nodes 8–9 — Upload to URL + Create IG Container<br>**Upload to URL** is the mandatory CDN bridge. It fetches the Bannerbear-rendered image binary and uploads it to the configured hosting endpoint, returning a stable **public URL**. This step is essential — Instagram's Graph API rejects base64 or binary payloads and requires a direct publicly accessible image URL.<br>Filename is auto-set to `review_{reviewer_name}_{timestamp}.jpg` for clean asset tracking.<br>**IG — Create Media Container** is Step 1 of the Instagram 2-step publish flow. POSTs to `/v19.0/{ig_account_id}/media` with the CDN URL and assembled caption. Returns a `container_id` which acts as a staging slot — Instagram validates and pre-processes the image before it goes live. |
| Wait — 6s IG Buffer | wait | Pause before Instagram publish call | IG — Create Media Container | IG — Publish Container | ### ⏳✅📋💬 Nodes 10–13 — Publish, Log & Notify<br>**Wait — 6s** gives Instagram time to finalise the media container before the publish call. Slightly longer than typical to account for image card complexity.<br>**IG — Publish Container** calls `/media_publish` with the `container_id`. Returns the live Instagram Post ID confirming the post is on the feed.<br>**Airtable — Mark as Posted** updates the original review record: sets `Posted = true` and writes back the Instagram Post ID and publish timestamp. This prevents the same review from being picked up on the next scheduled run.<br>**Slack — Notify Team** sends a block message with reviewer name, a snippet of the review, the post ID, and the CDN card image URL so the team can preview exactly what went live. |
| IG — Publish Container | httpRequest | Publish Instagram media container | Wait — 6s IG Buffer | Airtable — Mark as Posted | ### ⏳✅📋💬 Nodes 10–13 — Publish, Log & Notify<br>**Wait — 6s** gives Instagram time to finalise the media container before the publish call. Slightly longer than typical to account for image card complexity.<br>**IG — Publish Container** calls `/media_publish` with the `container_id`. Returns the live Instagram Post ID confirming the post is on the feed.<br>**Airtable — Mark as Posted** updates the original review record: sets `Posted = true` and writes back the Instagram Post ID and publish timestamp. This prevents the same review from being picked up on the next scheduled run.<br>**Slack — Notify Team** sends a block message with reviewer name, a snippet of the review, the post ID, and the CDN card image URL so the team can preview exactly what went live. |
| Airtable — Mark as Posted | airtable | Update source record after successful publish | IG — Publish Container | Slack — Notify Team | ### ⏳✅📋💬 Nodes 10–13 — Publish, Log & Notify<br>**Wait — 6s** gives Instagram time to finalise the media container before the publish call. Slightly longer than typical to account for image card complexity.<br>**IG — Publish Container** calls `/media_publish` with the `container_id`. Returns the live Instagram Post ID confirming the post is on the feed.<br>**Airtable — Mark as Posted** updates the original review record: sets `Posted = true` and writes back the Instagram Post ID and publish timestamp. This prevents the same review from being picked up on the next scheduled run.<br>**Slack — Notify Team** sends a block message with reviewer name, a snippet of the review, the post ID, and the CDN card image URL so the team can preview exactly what went live. |
| Slack — Notify Team | slack | Notify team after successful publication | Airtable — Mark as Posted |  | ### ⏳✅📋💬 Nodes 10–13 — Publish, Log & Notify<br>**Wait — 6s** gives Instagram time to finalise the media container before the publish call. Slightly longer than typical to account for image card complexity.<br>**IG — Publish Container** calls `/media_publish` with the `container_id`. Returns the live Instagram Post ID confirming the post is on the feed.<br>**Airtable — Mark as Posted** updates the original review record: sets `Posted = true` and writes back the Instagram Post ID and publish timestamp. This prevents the same review from being picked up on the next scheduled run.<br>**Slack — Notify Team** sends a block message with reviewer name, a snippet of the review, the post ID, and the CDN card image URL so the team can preview exactly what went live. |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## Prerequisites

1. **Prepare environment variables** in n8n:
   - `AIRTABLE_BASE_ID`
   - `BANNERBEAR_TEMPLATE_ID`
   - `IG_ACCOUNT_ID`
   - `SLACK_CHANNEL_ID`

2. **Create credentials**:
   - Airtable credential with access to the target base.
   - HTTP Header Auth credential for Bannerbear.
     - Header name typically: `Authorization`
     - Header value typically: `Bearer YOUR_BANNERBEAR_API_KEY` or the exact format Bannerbear expects in your account setup.
   - Instagram Graph API credential or store token in a secure credential pattern usable by HTTP Request nodes.
   - Slack credential.
   - Upload to URL credential for the community node.

3. **Install the Upload to URL community node** if it is not already available:
   - Package: `n8n-nodes-uploadtourl`  
   - Restart n8n if required.

4. **Prepare the Airtable table** `Reviews` with at least:
   - `Reviewer Name`
   - `Review Text`
   - `Rating`
   - `Posted`
   - `Submitted At`
   - recommended additional fields:
     - `Posted At`
     - `Card Image URL`
     - `Instagram Post ID`

5. **Prepare a Bannerbear template** with text layers named exactly:
   - `reviewer_name`
   - `review_text`
   - `star_label`

6. **Prepare Instagram publishing prerequisites**:
   - Instagram Business or Creator account connected to a Facebook Page.
   - Graph API permissions for media publishing.
   - Valid `IG_ACCOUNT_ID`.

---

## Build Steps

1. **Create a Schedule Trigger node**
   - Name: `Schedule — Daily 10AM`
   - Type: `Schedule Trigger`
   - Configure:
     - interval trigger
     - run daily at hour `10`
   - This is the workflow entry point.

2. **Create an Airtable node**
   - Name: `Airtable — Fetch 5-Star Review`
   - Type: `Airtable`
   - Operation: `Search`
   - Base: `{{ $env.AIRTABLE_BASE_ID }}`
   - Table: `Reviews`
   - Filter formula:
     ```text
     AND({Rating}=5, {Posted}=FALSE())
     ```
   - Sort by:
     - field: `Submitted At`
   - Connect:
     - `Schedule — Daily 10AM` → `Airtable — Fetch 5-Star Review`

3. **Create an IF node**
   - Name: `IF — Has Valid Review?`
   - Type: `IF`
   - Condition:
     - left value: `{{ $json.id }}`
     - operator: `is not empty`
   - Connect:
     - `Airtable — Fetch 5-Star Review` → `IF — Has Valid Review?`

4. **Create a No Operation node for the empty branch**
   - Name: `No Review — Exit Gracefully`
   - Type: `No Op`
   - Connect:
     - false branch of `IF — Has Valid Review?` → `No Review — Exit Gracefully`

5. **Create a Code node to prepare content**
   - Name: `Code — Prepare Bannerbear Payload`
   - Type: `Code`
   - Paste this logic:
     ```javascript
     const review = $input.item.json;
     const fields = review.fields || review;

     const reviewerName = fields['Reviewer Name'] || fields.reviewer_name || 'A Happy Customer';
     const reviewText = fields['Review Text'] || fields.review_text || '';
     const rating = fields['Rating'] || 5;
     const recordId = review.id || review.record_id;

     const cardText = reviewText.length > 180
       ? reviewText.substring(0, 177) + '...'
       : reviewText;

     const stars = '⭐'.repeat(Math.min(rating, 5));

     const modifications = [
       { name: 'reviewer_name', text: `— ${reviewerName}` },
       { name: 'review_text', text: `"${cardText}"` },
       { name: 'star_label', text: stars }
     ];

     const hashtagBlock = '#customerreview #5stars #testimonial #socialprrof #happycustomer #review';
     const caption = [
       `${stars} Real words from a real customer.`,
       '',
       `"${reviewText.substring(0, 300)}"`,
       `— ${reviewerName}`,
       '',
       '💬 See why hundreds of customers trust us.',
       '👉 Link in bio.',
       '.',
       '.',
       '.',
       hashtagBlock
     ].join('\n');

     const finalCaption = caption.length > 2200 ? caption.substring(0, 2196) + '...' : caption;

     return {
       record_id: recordId,
       reviewer_name: reviewerName,
       review_text: reviewText,
       card_text: cardText,
       stars,
       modifications,
       final_caption: finalCaption,
       ig_account_id: $env.IG_ACCOUNT_ID
     };
     ```
   - Connect:
     - true branch of `IF — Has Valid Review?` → `Code — Prepare Bannerbear Payload`

6. **Create an HTTP Request node for Bannerbear job creation**
   - Name: `HTTP — Bannerbear: Create Image Job`
   - Type: `HTTP Request`
   - Method: `POST`
   - URL:
     ```text
     https://api.bannerbear.com/v2/images
     ```
   - Authentication: `Generic Credential Type` → `HTTP Header Auth`
   - Select your Bannerbear header credential.
   - Send headers:
     - `Content-Type: application/json`
   - Body type: JSON
   - JSON body:
     ```json
     {
       "template": "{{ $env.BANNERBEAR_TEMPLATE_ID }}",
       "modifications": {{ JSON.stringify($json.modifications) }}
     }
     ```
   - Connect:
     - `Code — Prepare Bannerbear Payload` → `HTTP — Bannerbear: Create Image Job`

7. **Create a Wait node**
   - Name: `Wait — 5s Before Poll`
   - Type: `Wait`
   - Wait time: `5 seconds`
   - Connect:
     - `HTTP — Bannerbear: Create Image Job` → `Wait — 5s Before Poll`

8. **Create an HTTP Request node for Bannerbear polling**
   - Name: `HTTP — Bannerbear: Poll Status`
   - Type: `HTTP Request`
   - Method: `GET`
   - URL:
     ```text
     https://api.bannerbear.com/v2/images/{{ $('HTTP — Bannerbear: Create Image Job').item.json.uid }}
     ```
   - Authentication: same Bannerbear HTTP Header Auth credential
   - Connect:
     - `Wait — 5s Before Poll` → `HTTP — Bannerbear: Poll Status`

9. **Create an IF node to check render completion**
   - Name: `IF — Image Ready?`
   - Type: `IF`
   - Condition:
     - left value: `{{ $json.status }}`
     - operator: `equals`
     - right value: `completed`
   - Connect:
     - `HTTP — Bannerbear: Poll Status` → `IF — Image Ready?`

10. **Create a No Operation node for the not-ready branch**
    - Name: `Image Not Ready — Exit with Alert`
    - Type: `No Op`
    - Connect:
      - false branch of `IF — Image Ready?` → `Image Not Ready — Exit with Alert`

11. **Create an HTTP Request node to download the image**
    - Name: `HTTP — Fetch Rendered Card`
    - Type: `HTTP Request`
    - Method: `GET`
    - URL:
      ```text
      {{ $json.image_url }}
      ```
    - Response format: `File`
    - Binary output property: `cardImage`
    - Connect:
      - true branch of `IF — Image Ready?` → `HTTP — Fetch Rendered Card`

12. **Create the Upload to URL node**
    - Name: `Upload a File`
    - Type: `Upload to URL`
    - Select Upload to URL credentials.
    - If the node requires binary property selection, set it to `cardImage`.
    - If it supports custom filename, use something like:
      ```text
      review_{{ $('Code — Prepare Bannerbear Payload').item.json.reviewer_name }}_{{ Date.now() }}.jpg
      ```
    - Connect:
      - `HTTP — Fetch Rendered Card` → `Upload a File`

13. **Create a Code node to merge upload result with previous metadata**
    - Name: `Code — Merge Upload + Caption Data`
    - Type: `Code`
    - Paste:
      ```javascript
      const uploadResult = $input.item.json;
      const prepData = $('Code — Prepare Bannerbear Payload').item.json;

      return {
        ...uploadResult,
        public_image_url: uploadResult.public_url || uploadResult.url || uploadResult.file_url || uploadResult.cdn_url,
        final_caption: prepData.final_caption,
        ig_account_id: prepData.ig_account_id,
        record_id: prepData.record_id,
        reviewer_name: prepData.reviewer_name
      };
      ```
    - Connect:
      - `Upload a File` → `Code — Merge Upload + Caption Data`

14. **Create an HTTP Request node to create the Instagram media container**
    - Name: `IG — Create Media Container`
    - Type: `HTTP Request`
    - Method: `POST`
    - URL:
      ```text
      https://graph.facebook.com/v19.0/{{ $json.ig_account_id }}/media
      ```
    - Send query parameters:
      - `image_url` = `{{ $json.public_image_url }}`
      - `caption` = `{{ $json.final_caption }}`
      - `access_token` = your Instagram Graph API token
    - If you prefer credentials, store the token securely and reference it in expressions.
    - Connect:
      - `Code — Merge Upload + Caption Data` → `IG — Create Media Container`

15. **Create another Wait node**
    - Name: `Wait — 6s IG Buffer`
    - Type: `Wait`
    - Wait time: `6 seconds`
    - Connect:
      - `IG — Create Media Container` → `Wait — 6s IG Buffer`

16. **Create an HTTP Request node to publish the media container**
    - Name: `IG — Publish Container`
    - Type: `HTTP Request`
    - Method: `POST`
    - URL:
      ```text
      https://graph.facebook.com/v19.0/{{ $('Code — Merge Upload + Caption Data').item.json.ig_account_id }}/media_publish
      ```
    - Send query parameters:
      - `creation_id` = `{{ $('IG — Create Media Container').item.json.id }}`
      - `access_token` = your Instagram Graph API token
    - Connect:
      - `Wait — 6s IG Buffer` → `IG — Publish Container`

17. **Create an Airtable update node**
    - Name: `Airtable — Mark as Posted`
    - Type: `Airtable`
    - Operation: `Update`
    - Base: `{{ $env.AIRTABLE_BASE_ID }}`
    - Table: `Reviews`
    - Important: configure the node to update the specific record from the original Airtable search.
      - Use record ID: `{{ $('Code — Merge Upload + Caption Data').item.json.record_id }}`
      - If your Airtable node version requires a dedicated Record ID field, populate that field explicitly.
    - Fields to update:
      - `Posted` = `true`
      - `Posted At` = `{{ new Date().toISOString() }}`
      - `Card Image URL` = `{{ $('Code — Merge Upload + Caption Data').item.json.public_image_url }}`
      - `Instagram Post ID` = `{{ $('IG — Publish Container').item.json.id }}`
    - Connect:
      - `IG — Publish Container` → `Airtable — Mark as Posted`

18. **Create a Slack node**
    - Name: `Slack — Notify Team`
    - Type: `Slack`
    - Select Slack credentials.
    - Channel ID:
      ```text
      {{ $env.SLACK_CHANNEL_ID }}
      ```
    - Configure a message manually, because the provided JSON does not include the message body. A practical message example:
      - Reviewer name
      - short review snippet
      - Instagram post ID
      - public image URL
    - Example content:
      ```text
      New testimonial post published 🎉
      Reviewer: {{ $('Code — Merge Upload + Caption Data').item.json.reviewer_name }}
      Post ID: {{ $('IG — Publish Container').item.json.id }}
      Card URL: {{ $('Code — Merge Upload + Caption Data').item.json.public_image_url }}
      ```
    - Connect:
      - `Airtable — Mark as Posted` → `Slack — Notify Team`

19. **Optional: add sticky notes**
    - Recreate the documentation notes if you want the same annotated layout.
    - These are not required for execution.

20. **Test the workflow**
    - Start from the Airtable node or trigger manually.
    - Verify:
      - an eligible review is returned,
      - Bannerbear job returns a `uid`,
      - poll result returns `completed`,
      - image binary is present in `cardImage`,
      - Upload to URL returns a public URL,
      - Instagram media container returns an `id`,
      - publish returns a live Instagram post ID,
      - Airtable record updates correctly,
      - Slack message is delivered.

---

## Recommended Corrections While Rebuilding

To make the workflow production-safe, apply these improvements:

1. **Implement actual retry logic for Bannerbear polling**
   - The sticky note says there are retries, but the JSON does not implement them.
   - Add:
     - a retry counter,
     - a 3-second wait,
     - a loop back to poll,
     - a stop condition after 5 attempts.

2. **Explicitly attach Bannerbear auth to both Bannerbear HTTP nodes**
   - The poll node should clearly use the same credential as the create-job node.

3. **Fix the Airtable update node**
   - Ensure the record ID to update is explicitly set to the original Airtable record ID.

4. **Complete the Slack message configuration**
   - The node currently appears underconfigured in the exported JSON.

5. **Validate `public_image_url` before Instagram**
   - Add an IF node to stop if upload output does not contain a usable URL.

6. **Fix the typo in hashtags**
   - Replace `#socialprrof` with `#socialproof` if desired.

7. **Add error reporting**
   - Consider Slack alerts or error workflows for:
     - Bannerbear failure,
     - Instagram publish rejection,
     - Airtable update failure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow relies heavily on environment variables for portability across environments. | `AIRTABLE_BASE_ID`, `BANNERBEAR_TEMPLATE_ID`, `IG_ACCOUNT_ID`, `SLACK_CHANNEL_ID` |
| Upload to URL is used as a public CDN bridge because Instagram Graph API requires a directly reachable image URL rather than binary payload upload in this flow. | Workflow design note |
| The sticky documentation claims a retry loop for Bannerbear polling, but the provided workflow JSON only performs one poll after a 5-second delay. | Implementation mismatch to review before production use |
| The Bannerbear credential name shown in the exported JSON is “Stripe Header Auth,” which is operationally confusing even if technically functional. | Rename credential for maintainability |
| The Slack node appears incomplete in the JSON because no visible message body is defined. | Manual completion recommended |
| The Airtable update node appears incomplete because the target record identifier is not explicitly shown in the export. | Manual record ID mapping required |