Post new product drops to Instagram using uploadtourl and Airtable

https://n8nworkflows.xyz/workflows/post-new-product-drops-to-instagram-using-uploadtourl-and-airtable-14595


# Post new product drops to Instagram using uploadtourl and Airtable

# 1. Workflow Overview

This workflow automates the publication of a new product announcement to Instagram from a webhook-triggered payload. It receives product data, normalizes it into a consistent structure, fetches and re-hosts the product image to a public URL, builds an Instagram-ready caption, creates and publishes an Instagram media container through the Graph API, logs the result to Airtable, and finally notifies a Slack channel.

Typical use cases include:
- Shopify `products/create` webhook integrations
- Airtable automations posting product launches
- Manual HTTP POST tests from custom systems or scripts

The workflow is organized into the following logical blocks:

## 1.1 Input Reception and Immediate Acknowledgment
Receives a POST request through a webhook and immediately returns a JSON acknowledgment to avoid sender-side timeout issues.

## 1.2 Payload Normalization
Transforms incoming payload variants into a unified schema with fallback defaults for product name, image URL, price, caption intro, hashtags, Instagram account ID, and trigger source.

## 1.3 Image Retrieval and Public Hosting
Downloads the product image as binary data, then uploads it to an external hosting service so Instagram can access it via a public URL.

## 1.4 Caption Assembly and Instagram Media Container Creation
Builds a caption that matches Instagram limits and sends the image URL plus caption to Instagram’s media creation endpoint.

## 1.5 Processing Delay and Final Publication
Waits 5 seconds to give Instagram time to process the media container, then publishes it to the Instagram feed.

## 1.6 Logging and Team Notification
Stores publication metadata in Airtable and sends a Slack notification after the post is live.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Immediate Acknowledgment

### Overview
This block starts the workflow with an HTTP POST webhook and returns an immediate JSON response. The response is decoupled from the longer Instagram publishing chain, which helps prevent upstream systems from timing out.

### Nodes Involved
- `Webhook — Product Trigger1`
- `Webhook Response1`

### Node Details

#### Webhook — Product Trigger1
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that listens for incoming HTTP POST requests.
- **Configuration choices:**
  - Path: `product-drop`
  - Method: `POST`
  - Response mode: `responseNode`, meaning a dedicated Respond to Webhook node must return the HTTP response
- **Key expressions or variables used:** None in node parameters
- **Input and output connections:**
  - No input node
  - Outputs to:
    - `Webhook Response1`
    - `Set — Normalize Fields1`
- **Version-specific requirements:**
  - Uses node type version `2`
  - Requires support for `responseNode` mode in the installed n8n version
- **Edge cases or potential failure types:**
  - Wrong HTTP method will not trigger
  - Incorrect webhook URL/path configuration will produce 404
  - If the response node is removed or misconfigured, callers may hang or receive no response
- **Sub-workflow reference:** None

#### Webhook Response1
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an immediate JSON acknowledgment back to the webhook caller.
- **Configuration choices:**
  - Response format: JSON
  - Response body:
    ```json
    { "status": "received", "message": "Product drop workflow triggered successfully" }
    ```
- **Key expressions or variables used:**
  - Static expression-based JSON body
- **Input and output connections:**
  - Input from `Webhook — Product Trigger1`
  - No output
- **Version-specific requirements:**
  - Uses node type version `1`
- **Edge cases or potential failure types:**
  - If this node errors, the caller may receive an HTTP error instead of acknowledgment
  - Invalid JSON expression would break response generation
- **Sub-workflow reference:** None

---

## 2.2 Payload Normalization

### Overview
This block converts heterogeneous webhook payloads into a stable internal schema. It supports both flat payloads and nested fields commonly seen in Shopify-style webhook bodies.

### Nodes Involved
- `Set — Normalize Fields1`

### Node Details

#### Set — Normalize Fields1
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps and sanitizes incoming data into normalized fields used throughout the rest of the workflow.
- **Configuration choices:**
  - Creates these fields:
    - `product_name`
    - `image_url`
    - `price`
    - `caption_intro`
    - `hashtags`
    - `ig_account_id`
    - `trigger_source`
  - Uses nullish fallback logic for resilience
- **Key expressions or variables used:**
  - `product_name`: `{{ $json.body.product_name ?? $json.body.title ?? 'New Product' }}`
  - `image_url`: `{{ $json.body.image_url ?? $json.body.images?.[0]?.src ?? '' }}`
  - `price`: `{{ $json.body.price ?? $json.body.variants?.[0]?.price ?? '' }}`
  - `caption_intro`: `{{ $json.body.caption_intro ?? 'Something new just dropped.' }}`
  - `hashtags`: `{{ $json.body.hashtags ?? '#newdrop #product #brand' }}`
  - `ig_account_id`: `{{ $json.body.ig_account_id ?? $env.IG_ACCOUNT_ID }}`
  - `trigger_source`: `{{ $json.body.source ?? 'webhook' }}`
- **Input and output connections:**
  - Input from `Webhook — Product Trigger1`
  - Output to `HTTP — Fetch Product Image1`
- **Version-specific requirements:**
  - Uses node type version `3.4`
  - Requires support for `assignments`-based Set node configuration
- **Edge cases or potential failure types:**
  - If incoming payload does not contain `body`, expressions may still work if `body` is undefined, but resulting values may be empty/defaulted
  - Empty `image_url` will cause downstream HTTP download failure
  - Missing `ig_account_id` and missing environment variable `IG_ACCOUNT_ID` will cause Instagram API requests to fail
  - Invalid payload shape may silently fall back to defaults, masking upstream data issues
- **Sub-workflow reference:** None

---

## 2.3 Image Retrieval and Public Hosting

### Overview
This block downloads the source product image and uploads it to a public hosting service. This is necessary because Instagram’s API expects a publicly reachable image URL when creating a media container.

### Nodes Involved
- `HTTP — Fetch Product Image1`
- `Upload a File`

### Node Details

#### HTTP — Fetch Product Image1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the product image from the normalized `image_url` and stores it as binary data.
- **Configuration choices:**
  - URL: `{{ $json.image_url }}`
  - Response format: file
  - Binary output property: `imageData`
- **Key expressions or variables used:**
  - Uses normalized `image_url`
- **Input and output connections:**
  - Input from `Set — Normalize Fields1`
  - Output to `Upload a File`
- **Version-specific requirements:**
  - Uses node type version `4.2`
  - Requires HTTP Request node support for file response output
- **Edge cases or potential failure types:**
  - Empty or malformed URL
  - 403/404 from source image host
  - Large image download may time out
  - Redirect handling depends on node defaults and source host behavior
  - Non-image content may still download but fail later with upload service or Instagram
- **Sub-workflow reference:** None

#### Upload a File
- **Type and technical role:** `n8n-nodes-uploadtourl.uploadToUrl`  
  Uploads binary file content to a media hosting service and returns a public URL.
- **Configuration choices:**
  - Default parameters only
  - Uses credential: `uploadtourl - Deepanshi`
- **Key expressions or variables used:** None visible in parameters
- **Input and output connections:**
  - Input from `HTTP — Fetch Product Image1`
  - Output to `Code — Build Caption1`
- **Version-specific requirements:**
  - Community/custom node: `n8n-nodes-uploadtourl.uploadToUrl`
  - Requires this package to be installed in the n8n instance
  - Uses node type version `1`
- **Edge cases or potential failure types:**
  - Community node not installed
  - Credential missing or invalid
  - Binary property expectations may differ depending on node implementation
  - Upload host may reject file size, MIME type, or unsupported formats
  - Returned URL field names may vary; this workflow compensates for that in the next Code node
- **Sub-workflow reference:** None

---

## 2.4 Caption Assembly and Instagram Media Container Creation

### Overview
This block generates a final Instagram caption and creates the Instagram media container. It also resolves the public image URL from the upload node output and prepares all fields expected by the Graph API.

### Nodes Involved
- `Code — Build Caption1`
- `IG — Create Media Container1`

### Node Details

#### Code — Build Caption1
- **Type and technical role:** `n8n-nodes-base.code`  
  Creates a formatted caption and normalizes the uploaded image URL field into `public_image_url`.
- **Configuration choices:**
  - JavaScript code reads:
    - current item via `$input.item.json`
    - normalized values via `$('Set — Normalize Fields1').item.json`
  - Builds caption with:
    - intro
    - product line
    - optional price
    - CTA
    - hashtags
  - Truncates caption at 2,200 characters
  - Maps public image URL using fallbacks:
    - `public_url`
    - `url`
    - `file_url`
    - `cdn_url`
- **Key expressions or variables used:**
  - `const norm = $('Set — Normalize Fields1').item.json;`
  - `final_caption`
  - `public_image_url`
- **Input and output connections:**
  - Input from `Upload a File`
  - Output to `IG — Create Media Container1`
- **Version-specific requirements:**
  - Uses node type version `2`
  - Requires Code node JavaScript runtime support
- **Edge cases or potential failure types:**
  - If the upload node returns none of `public_url`, `url`, `file_url`, or `cdn_url`, `public_image_url` will be undefined
  - If upstream item count changes unexpectedly, direct `.item` references may behave unpredictably
  - Caption truncation is character-based, not semantic; emoji or line breaks near the cutoff may look awkward
- **Sub-workflow reference:** None

#### IG — Create Media Container1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Instagram Graph API `/media` endpoint to create a media container for a single-image feed post.
- **Configuration choices:**
  - Method: `POST`
  - URL pattern:
    `https://graph.facebook.com/v19.0/{ig_account_id}/media`
  - Query parameters:
    - `image_url`: `{{ $json.public_image_url }}`
    - `caption`: `{{ $json.final_caption }}`
    - `access_token`: `{{ $credentials.instagramGraphApi.accessToken }}`
- **Key expressions or variables used:**
  - Instagram account ID from `Set — Normalize Fields1`
  - Access token from credentials object
- **Input and output connections:**
  - Input from `Code — Build Caption1`
  - Output to `Wait — 5s Processing Buffer1`
- **Version-specific requirements:**
  - Uses node type version `4.2`
  - Assumes an `instagramGraphApi` credential exists and exposes `accessToken`
  - Uses Graph API version `v19.0`
- **Edge cases or potential failure types:**
  - Missing or expired access token
  - Invalid or inaccessible image URL
  - Unsupported image format, image dimensions, or file constraints
  - Invalid Instagram business account ID
  - Graph API version changes may eventually require endpoint updates
  - Sending access token as query parameter is functional but may be less desirable from a security/logging perspective than header-based auth
- **Sub-workflow reference:** None

---

## 2.5 Processing Delay and Final Publication

### Overview
This block waits briefly for Instagram to complete asynchronous media processing, then publishes the created media container. It implements the standard two-step Instagram publishing sequence.

### Nodes Involved
- `Wait — 5s Processing Buffer1`
- `IG — Publish Container1`

### Node Details

#### Wait — 5s Processing Buffer1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Adds a fixed delay before attempting publication.
- **Configuration choices:**
  - Wait duration: 5 seconds
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `IG — Create Media Container1`
  - Output to `IG — Publish Container1`
- **Version-specific requirements:**
  - Uses node type version `1`
- **Edge cases or potential failure types:**
  - 5 seconds may be insufficient for larger or slower-to-process images
  - For high reliability, a polling loop against container status would be safer than a fixed wait
  - Wait nodes in some environments can depend on proper webhook/resume configuration
- **Sub-workflow reference:** None

#### IG — Publish Container1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Instagram Graph API `/media_publish` endpoint using the container ID returned by the previous creation step.
- **Configuration choices:**
  - Method: `POST`
  - URL pattern:
    `https://graph.facebook.com/v19.0/{ig_account_id}/media_publish`
  - Query parameters:
    - `creation_id`: `{{ $('IG — Create Media Container1').item.json.id }}`
    - `access_token`: `{{ $credentials.instagramGraphApi.accessToken }}`
- **Key expressions or variables used:**
  - Container ID from `IG — Create Media Container1`
  - Instagram account ID from `Set — Normalize Fields1`
  - Access token from credentials
- **Input and output connections:**
  - Input from `Wait — 5s Processing Buffer1`
  - Output to `Airtable — Log Post1`
- **Version-specific requirements:**
  - Uses node type version `4.2`
  - Same credential assumptions as the media creation step
- **Edge cases or potential failure types:**
  - Publishing too early may return an API error if media is not yet ready
  - Invalid or expired creation ID
  - Access token expiration or permission issues
  - Graph API throttling/rate limiting
- **Sub-workflow reference:** None

---

## 2.6 Logging and Team Notification

### Overview
This block records the published result in Airtable and notifies a Slack channel. It creates an audit trail and gives the team operational visibility when a post goes live.

### Nodes Involved
- `Airtable — Log Post1`
- `Slack — Notify Team1`

### Node Details

#### Airtable — Log Post1
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Creates a new record in Airtable with publication metadata.
- **Configuration choices:**
  - Operation: `create`
  - Base ID from environment variable: `AIRTABLE_BASE_ID`
  - Table name: `IG Post Log`
  - Column mapping:
    - `Status`: `Published`
    - `Caption`: from `Code — Build Caption1.final_caption`
    - `Post ID`: from `IG — Publish Container1.id`
    - `Image URL`: from `Code — Build Caption1.public_image_url`
    - `Product Name`: from `Set — Normalize Fields1.product_name`
    - `Published At`: `new Date().toISOString()`
    - `Trigger Source`: from `Set — Normalize Fields1.trigger_source`
- **Key expressions or variables used:**
  - `{{ $env.AIRTABLE_BASE_ID }}`
  - Several cross-node expressions
- **Input and output connections:**
  - Input from `IG — Publish Container1`
  - Output to `Slack — Notify Team1`
- **Version-specific requirements:**
  - Uses node type version `2.1`
  - Requires Airtable credentials configured in n8n
- **Edge cases or potential failure types:**
  - Missing `AIRTABLE_BASE_ID`
  - Table or field names do not match exactly
  - Airtable credential/authentication failure
  - Field type mismatch in Airtable schema
  - Rate limiting on Airtable API
- **Sub-workflow reference:** None

#### Slack — Notify Team1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack notification to a target channel after logging succeeds.
- **Configuration choices:**
  - Target type: channel
  - Channel ID from environment variable: `SLACK_CHANNEL_ID`
  - No message text is explicitly configured in the provided JSON
- **Key expressions or variables used:**
  - `{{ $env.SLACK_CHANNEL_ID }}`
- **Input and output connections:**
  - Input from `Airtable — Log Post1`
  - No output
- **Version-specific requirements:**
  - Uses node type version `2.3`
  - Requires Slack credentials configured in n8n
- **Edge cases or potential failure types:**
  - As configured, this node appears incomplete for a useful notification unless the Slack node defaults are providing a message elsewhere
  - Missing or invalid Slack credential
  - Missing `SLACK_CHANNEL_ID`
  - Bot not invited to the destination channel
  - Channel ID vs channel name mismatch
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky — Webhook + Normalize | n8n-nodes-base.stickyNote | Documentation note for webhook and normalization area |  |  | Webhook Trigger: Listens for POST payloads (Shopify, Airtable, or manual). It uses a Webhook Respond node to immediately return a 200 ACK, preventing timeout errors from the sender. |
| Sticky — Webhook + Normalize | n8n-nodes-base.stickyNote | Documentation note for webhook and normalization area |  |  | Set (Normalize Fields): Sanitizes raw data into a consistent schema. It maps nested fields (e.g., Shopify’s body.title) to flat variables, applies fallback defaults for missing captions/hashtags, and pulls the ig_account_id from environment variables if the payload is empty. |
| Sticky — Fetch + Upload | n8n-nodes-base.stickyNote | Documentation note for image fetch and upload area |  |  | HTTP — Fetch Product Image: Downloads the image from image_url as a binary file (imageData). It supports any public source, including Shopify CDNs or S3 buckets. |
| Sticky — Fetch + Upload | n8n-nodes-base.stickyNote | Documentation note for image fetch and upload area |  |  | Upload to URL: Uploads the binary to a media host and returns a public CDN URL. |
| Sticky — Caption + Container | n8n-nodes-base.stickyNote | Documentation note for caption and media container area |  |  | Code — Build Caption: Assembles the final text using the product name, price, and links. It automatically truncates to 2,200 characters to meet Instagram’s API limits and outputs a single final_caption string. |
| Sticky — Caption + Container | n8n-nodes-base.stickyNote | Documentation note for caption and media container area |  |  | IG — Create Media Container: Initiates the first half of Instagram’s two-step publishing flow. It sends the image_url and final_caption to the Graph API, which processes the image asynchronously and returns a container_id. |
| Sticky — Wait + Publish | n8n-nodes-base.stickyNote | Documentation note for wait and publish area |  |  | Wait (5s Buffer): Pauses the workflow to ensure Instagram completes its asynchronous image processing. For high-resolution files, this can be extended or replaced with a status-check polling loop. |
| Sticky — Wait + Publish | n8n-nodes-base.stickyNote | Documentation note for wait and publish area |  |  | IG — Publish Container: Executes the final media_publish call using the container_id. Once successful, Instagram returns a Live Post ID, confirming the content is officially on the feed. |
| Sticky — Log + Notify | n8n-nodes-base.stickyNote | Documentation note for logging and notifications area |  |  | Airtable — Log Post: Records the Post ID, product name, and timestamp into a tracking base for audits and analytics. Easily swappable with Google Sheets. |
| Sticky — Log + Notify | n8n-nodes-base.stickyNote | Documentation note for logging and notifications area |  |  | Slack — Notify Team: Sends a formatted alert confirming the post is live. Includes the live link and trigger source. Can be replaced with Email or Discord nodes as needed. |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | ## 📦 Brand Product Drop — Single Image Scheduler |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | **What this workflow does:** |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | Automates the full Instagram product announcement pipeline — from a new product trigger (Shopify, Airtable, or manual webhook) all the way to a published Instagram post with a structured caption and hashtag block. |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | **Flow Overview:** |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 1. 🔔 **Webhook Trigger** — Receives product data (name, image URL, price, hashtags, caption intro) |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 2. 🧹 **Set / Normalize Fields** — Cleans and structures the incoming payload into consistent fields |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 3. 🌐 **Fetch Product Image** — Downloads the raw image binary from the external product image URL |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 4. ☁️ **Upload to URL** — Uploads the image binary and returns a public CDN URL |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 5. ✍️ **Build Caption** — Constructs a brand-ready caption with emoji, price, CTA, and hashtag block |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 6. 📸 **IG: Create Media Container** — Calls Instagram Graph API to create a media container using the public image URL |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 7. ⏳ **Wait** — Pauses 5 seconds to allow Instagram to process the media container |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 8. ✅ **IG: Publish Container** — Publishes the approved container to the Instagram feed |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 9. 📊 **Log to Airtable** — Records post ID, timestamp, caption, and image URL for tracking |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | 10. 🔔 **Notify via Slack** — Sends a confirmation message with the published post details |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | **Prerequisites:** |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - Instagram Graph API access token (Business or Creator account) |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - Instagram Business Account ID |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - uploadtourl node credentials configured in n8n |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - (Optional) Airtable API key + Base ID for post logging |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - (Optional) Slack API credentials for team notifications |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | **Supported Trigger Sources:** |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - Shopify `products/create` webhook |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - Airtable automation webhook |
| 📋 Template Overview1 | n8n-nodes-base.stickyNote | Top-level workflow description |  |  | - Manual HTTP POST for testing |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | Webhook Trigger: Listens for POST payloads (Shopify, Airtable, or manual). It uses a Webhook Respond node to immediately return a 200 ACK, preventing timeout errors from the sender. |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | Set (Normalize Fields): Sanitizes raw data into a consistent schema. It maps nested fields (e.g., Shopify’s body.title) to flat variables, applies fallback defaults for missing captions/hashtags, and pulls the ig_account_id from environment variables if the payload is empty. |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | ## 📦 Brand Product Drop — Single Image Scheduler |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | **What this workflow does:** |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | Automates the full Instagram product announcement pipeline — from a new product trigger (Shopify, Airtable, or manual webhook) all the way to a published Instagram post with a structured caption and hashtag block. |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | **Flow Overview:** |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | 1. 🔔 **Webhook Trigger** — Receives product data (name, image URL, price, hashtags, caption intro) |
| Webhook — Product Trigger1 | n8n-nodes-base.webhook | Receives product-drop POST requests |  | Webhook Response1; Set — Normalize Fields1 | 2. 🧹 **Set / Normalize Fields** — Cleans and structures the incoming payload into consistent fields |
| Webhook Response1 | n8n-nodes-base.respondToWebhook | Immediately acknowledges webhook caller | Webhook — Product Trigger1 |  | Webhook Trigger: Listens for POST payloads (Shopify, Airtable, or manual). It uses a Webhook Respond node to immediately return a 200 ACK, preventing timeout errors from the sender. |
| Webhook Response1 | n8n-nodes-base.respondToWebhook | Immediately acknowledges webhook caller | Webhook — Product Trigger1 |  | Set (Normalize Fields): Sanitizes raw data into a consistent schema. It maps nested fields (e.g., Shopify’s body.title) to flat variables, applies fallback defaults for missing captions/hashtags, and pulls the ig_account_id from environment variables if the payload is empty. |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | Webhook Trigger: Listens for POST payloads (Shopify, Airtable, or manual). It uses a Webhook Respond node to immediately return a 200 ACK, preventing timeout errors from the sender. |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | Set (Normalize Fields): Sanitizes raw data into a consistent schema. It maps nested fields (e.g., Shopify’s body.title) to flat variables, applies fallback defaults for missing captions/hashtags, and pulls the ig_account_id from environment variables if the payload is empty. |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | ## 📦 Brand Product Drop — Single Image Scheduler |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | **What this workflow does:** |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | Automates the full Instagram product announcement pipeline — from a new product trigger (Shopify, Airtable, or manual webhook) all the way to a published Instagram post with a structured caption and hashtag block. |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | **Flow Overview:** |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | 1. 🔔 **Webhook Trigger** — Receives product data (name, image URL, price, hashtags, caption intro) |
| Set — Normalize Fields1 | n8n-nodes-base.set | Normalizes incoming product payload | Webhook — Product Trigger1 | HTTP — Fetch Product Image1 | 2. 🧹 **Set / Normalize Fields** — Cleans and structures the incoming payload into consistent fields |
| HTTP — Fetch Product Image1 | n8n-nodes-base.httpRequest | Downloads source image as binary | Set — Normalize Fields1 | Upload a File | HTTP — Fetch Product Image: Downloads the image from image_url as a binary file (imageData). It supports any public source, including Shopify CDNs or S3 buckets. |
| HTTP — Fetch Product Image1 | n8n-nodes-base.httpRequest | Downloads source image as binary | Set — Normalize Fields1 | Upload a File | Upload to URL: Uploads the binary to a media host and returns a public CDN URL. |
| Upload a File | n8n-nodes-uploadtourl.uploadToUrl | Uploads binary image to public hosting | HTTP — Fetch Product Image1 | Code — Build Caption1 | HTTP — Fetch Product Image: Downloads the image from image_url as a binary file (imageData). It supports any public source, including Shopify CDNs or S3 buckets. |
| Upload a File | n8n-nodes-uploadtourl.uploadToUrl | Uploads binary image to public hosting | HTTP — Fetch Product Image1 | Code — Build Caption1 | Upload to URL: Uploads the binary to a media host and returns a public CDN URL. |
| Code — Build Caption1 | n8n-nodes-base.code | Builds final caption and resolves public image URL | Upload a File | IG — Create Media Container1 | Code — Build Caption: Assembles the final text using the product name, price, and links. It automatically truncates to 2,200 characters to meet Instagram’s API limits and outputs a single final_caption string. |
| Code — Build Caption1 | n8n-nodes-base.code | Builds final caption and resolves public image URL | Upload a File | IG — Create Media Container1 | IG — Create Media Container: Initiates the first half of Instagram’s two-step publishing flow. It sends the image_url and final_caption to the Graph API, which processes the image asynchronously and returns a container_id. |
| IG — Create Media Container1 | n8n-nodes-base.httpRequest | Creates Instagram media container | Code — Build Caption1 | Wait — 5s Processing Buffer1 | Code — Build Caption: Assembles the final text using the product name, price, and links. It automatically truncates to 2,200 characters to meet Instagram’s API limits and outputs a single final_caption string. |
| IG — Create Media Container1 | n8n-nodes-base.httpRequest | Creates Instagram media container | Code — Build Caption1 | Wait — 5s Processing Buffer1 | IG — Create Media Container: Initiates the first half of Instagram’s two-step publishing flow. It sends the image_url and final_caption to the Graph API, which processes the image asynchronously and returns a container_id. |
| Wait — 5s Processing Buffer1 | n8n-nodes-base.wait | Delays publish attempt for media processing | IG — Create Media Container1 | IG — Publish Container1 | Wait (5s Buffer): Pauses the workflow to ensure Instagram completes its asynchronous image processing. For high-resolution files, this can be extended or replaced with a status-check polling loop. |
| Wait — 5s Processing Buffer1 | n8n-nodes-base.wait | Delays publish attempt for media processing | IG — Create Media Container1 | IG — Publish Container1 | IG — Publish Container: Executes the final media_publish call using the container_id. Once successful, Instagram returns a Live Post ID, confirming the content is officially on the feed. |
| IG — Publish Container1 | n8n-nodes-base.httpRequest | Publishes Instagram media container | Wait — 5s Processing Buffer1 | Airtable — Log Post1 | Wait (5s Buffer): Pauses the workflow to ensure Instagram completes its asynchronous image processing. For high-resolution files, this can be extended or replaced with a status-check polling loop. |
| IG — Publish Container1 | n8n-nodes-base.httpRequest | Publishes Instagram media container | Wait — 5s Processing Buffer1 | Airtable — Log Post1 | IG — Publish Container: Executes the final media_publish call using the container_id. Once successful, Instagram returns a Live Post ID, confirming the content is officially on the feed. |
| Airtable — Log Post1 | n8n-nodes-base.airtable | Logs published post metadata to Airtable | IG — Publish Container1 | Slack — Notify Team1 | Airtable — Log Post: Records the Post ID, product name, and timestamp into a tracking base for audits and analytics. Easily swappable with Google Sheets. |
| Airtable — Log Post1 | n8n-nodes-base.airtable | Logs published post metadata to Airtable | IG — Publish Container1 | Slack — Notify Team1 | Slack — Notify Team: Sends a formatted alert confirming the post is live. Includes the live link and trigger source. Can be replaced with Email or Discord nodes as needed. |
| Slack — Notify Team1 | n8n-nodes-base.slack | Notifies team in Slack channel | Airtable — Log Post1 |  | Airtable — Log Post: Records the Post ID, product name, and timestamp into a tracking base for audits and analytics. Easily swappable with Google Sheets. |
| Slack — Notify Team1 | n8n-nodes-base.slack | Notifies team in Slack channel | Airtable — Log Post1 |  | Slack — Notify Team: Sends a formatted alert confirming the post is live. Includes the live link and trigger source. Can be replaced with Email or Discord nodes as needed. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node** named `Webhook — Product Trigger1`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `product-drop`
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`

3. **Add a Respond to Webhook node** named `Webhook Response1`.
   - Type: `Respond to Webhook`
   - Respond With: `JSON`
   - Response Body:
     ```json
     { "status": "received", "message": "Product drop workflow triggered successfully" }
     ```

4. **Connect** `Webhook — Product Trigger1` to `Webhook Response1`.

5. **Add a Set node** named `Set — Normalize Fields1`.
   - Type: `Set`
   - Create the following fields as strings:
     - `product_name` → `{{ $json.body.product_name ?? $json.body.title ?? 'New Product' }}`
     - `image_url` → `{{ $json.body.image_url ?? $json.body.images?.[0]?.src ?? '' }}`
     - `price` → `{{ $json.body.price ?? $json.body.variants?.[0]?.price ?? '' }}`
     - `caption_intro` → `{{ $json.body.caption_intro ?? 'Something new just dropped.' }}`
     - `hashtags` → `{{ $json.body.hashtags ?? '#newdrop #product #brand' }}`
     - `ig_account_id` → `{{ $json.body.ig_account_id ?? $env.IG_ACCOUNT_ID }}`
     - `trigger_source` → `{{ $json.body.source ?? 'webhook' }}`
   - Leave other options at defaults.

6. **Connect** `Webhook — Product Trigger1` to `Set — Normalize Fields1`.

7. **Add an HTTP Request node** named `HTTP — Fetch Product Image1`.
   - Type: `HTTP Request`
   - Method: `GET` is sufficient unless your n8n version defaults appropriately
   - URL: `{{ $json.image_url }}`
   - Response Format: `File`
   - Output Binary Property: `imageData`

8. **Connect** `Set — Normalize Fields1` to `HTTP — Fetch Product Image1`.

9. **Install the community node package** for uploadtourl if not already present.
   - Node type required: `n8n-nodes-uploadtourl.uploadToUrl`
   - Confirm your n8n instance allows community nodes.

10. **Create uploadtourl credentials** in n8n.
    - Use the credential type required by the community node, shown here as `uploadToUrlApi`
    - Supply the API key or auth details required by that service

11. **Add the uploadtourl node** named `Upload a File`.
    - Type: `Upload to URL`
    - Keep default parameters unless your upload service needs a specific binary property
    - Attach the `uploadToUrlApi` credential

12. **Connect** `HTTP — Fetch Product Image1` to `Upload a File`.

13. **Add a Code node** named `Code — Build Caption1`.
   - Type: `Code`
   - Language: JavaScript
   - Paste this logic:
     ```javascript
     const item = $input.item.json;
     const norm = $('Set — Normalize Fields1').item.json;

     const productName = norm.product_name || 'New Drop';
     const price = norm.price ? `💰 ${norm.price}` : '';
     const captionIntro = norm.caption_intro || 'Something new just dropped.';
     const hashtags = norm.hashtags || '#newdrop';

     const lines = [
       captionIntro,
       '',
       `🛍️ ${productName}`,
       price,
       '',
       '👉 Link in bio to shop now.',
       '.',
       '.',
       '.',
       hashtags
     ].filter(l => l !== undefined && l !== null);

     let finalCaption = lines.join('\n');

     if (finalCaption.length > 2200) {
       finalCaption = finalCaption.substring(0, 2196) + '...';
     }

     return {
       ...item,
       final_caption: finalCaption,
       public_image_url: item.public_url || item.url || item.file_url || item.cdn_url
     };
     ```

14. **Connect** `Upload a File` to `Code — Build Caption1`.

15. **Prepare Instagram Graph API credentials**.
   - This workflow expects a credential object accessible as `instagramGraphApi` with an `accessToken`
   - Ensure the token belongs to a valid Instagram Business or Creator setup through Meta Graph API
   - Confirm the token has permissions required for content publishing

16. **Add an HTTP Request node** named `IG — Create Media Container1`.
   - Type: `HTTP Request`
   - Method: `POST`
   - URL:
     `https://graph.facebook.com/v19.0/{{ $('Set — Normalize Fields1').item.json.ig_account_id }}/media`
   - Enable query parameters
   - Add:
     - `image_url` → `{{ $json.public_image_url }}`
     - `caption` → `{{ $json.final_caption }}`
     - `access_token` → `{{ $credentials.instagramGraphApi.accessToken }}`
   - Attach the credential context that exposes `instagramGraphApi.accessToken`

17. **Connect** `Code — Build Caption1` to `IG — Create Media Container1`.

18. **Add a Wait node** named `Wait — 5s Processing Buffer1`.
   - Type: `Wait`
   - Unit: `Seconds`
   - Amount: `5`

19. **Connect** `IG — Create Media Container1` to `Wait — 5s Processing Buffer1`.

20. **Add another HTTP Request node** named `IG — Publish Container1`.
   - Type: `HTTP Request`
   - Method: `POST`
   - URL:
     `https://graph.facebook.com/v19.0/{{ $('Set — Normalize Fields1').item.json.ig_account_id }}/media_publish`
   - Enable query parameters
   - Add:
     - `creation_id` → `{{ $('IG — Create Media Container1').item.json.id }}`
     - `access_token` → `{{ $credentials.instagramGraphApi.accessToken }}`

21. **Connect** `Wait — 5s Processing Buffer1` to `IG — Publish Container1`.

22. **Create Airtable credentials** in n8n.
   - Use a valid Airtable personal access token or supported auth method
   - Ensure the token can create records in the target base and table

23. **Set the environment variable** `AIRTABLE_BASE_ID` on the n8n instance.
   - This workflow reads the Airtable base ID from environment settings
   - Create a table named `IG Post Log`

24. **Prepare the Airtable table schema** with fields matching these names:
   - `Status`
   - `Caption`
   - `Post ID`
   - `Image URL`
   - `Product Name`
   - `Published At`
   - `Trigger Source`

25. **Add an Airtable node** named `Airtable — Log Post1`.
   - Type: `Airtable`
   - Operation: `Create`
   - Base ID: `{{ $env.AIRTABLE_BASE_ID }}`
   - Table: `IG Post Log`
   - Map fields:
     - `Status` → `Published`
     - `Caption` → `{{ $('Code — Build Caption1').item.json.final_caption }}`
     - `Post ID` → `{{ $('IG — Publish Container1').item.json.id }}`
     - `Image URL` → `{{ $('Code — Build Caption1').item.json.public_image_url }}`
     - `Product Name` → `{{ $('Set — Normalize Fields1').item.json.product_name }}`
     - `Published At` → `{{ new Date().toISOString() }}`
     - `Trigger Source` → `{{ $('Set — Normalize Fields1').item.json.trigger_source }}`

26. **Connect** `IG — Publish Container1` to `Airtable — Log Post1`.

27. **Create Slack credentials** in n8n.
   - Use a Slack app/bot token with permission to post to the target channel

28. **Set the environment variable** `SLACK_CHANNEL_ID`.
   - Use the actual Slack channel ID, not the display name

29. **Add a Slack node** named `Slack — Notify Team1`.
   - Type: `Slack`
   - Target selection mode: `Channel`
   - Channel ID: `{{ $env.SLACK_CHANNEL_ID }}`
   - Important: the provided workflow JSON does not define a visible message body, so you should configure one manually for a working notification, for example:
     - “Instagram post published: {{ $('IG — Publish Container1').item.json.id }} for {{ $('Set — Normalize Fields1').item.json.product_name }}”
   - If your Slack node requires choosing an operation, select a message-posting action such as `Post Message`

30. **Connect** `Airtable — Log Post1` to `Slack — Notify Team1`.

31. **Set required environment variables** in your n8n deployment:
   - `IG_ACCOUNT_ID`
   - `AIRTABLE_BASE_ID`
   - `SLACK_CHANNEL_ID`

32. **Activate the workflow** after testing.

33. **Test with a sample POST payload** sent to the webhook URL. Example structure:
   ```json
   {
     "body": {
       "product_name": "New Sneaker",
       "image_url": "https://example.com/product.jpg",
       "price": "$129",
       "caption_intro": "Fresh arrival.",
       "hashtags": "#newdrop #sneakers #brand",
       "ig_account_id": "1784xxxxxxxxxx",
       "source": "manual-test"
     }
   }
   ```

34. **Verify each stage**:
   - Webhook returns immediate JSON acknowledgment
   - Image downloads successfully as binary
   - Upload service returns a public URL
   - Caption is generated
   - Instagram media container returns an `id`
   - Publish call returns a live post `id`
   - Airtable record is created
   - Slack message is posted

35. **Recommended hardening improvements** if rebuilding for production:
   - Add validation for empty `image_url`
   - Add an IF node to stop when `public_image_url` is missing
   - Replace fixed Wait with a polling loop checking Instagram container readiness
   - Add error handling branches for Slack/Airtable failures so publication success is not obscured by downstream logging issues

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow title: Post new product drops to Instagram using uploadtourl and Airtable | Project context |
| The workflow is designed for single-image Instagram feed posts, not carousel or video publishing | Instagram publishing scope |
| It assumes the incoming payload is nested under `body`, which matches many webhook payload formats but not all custom senders | Input payload design |
| The Slack notification node appears underconfigured in the provided JSON and should be reviewed before production use | Operational note |
| The upload step depends on a community node rather than a built-in n8n node | Dependency note |
| Graph API version used by both Instagram requests is `v19.0` | Meta API compatibility note |