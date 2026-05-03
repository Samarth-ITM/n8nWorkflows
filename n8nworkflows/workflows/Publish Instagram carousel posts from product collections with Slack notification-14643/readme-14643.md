Publish Instagram carousel posts from product collections with Slack notification

https://n8nworkflows.xyz/workflows/publish-instagram-carousel-posts-from-product-collections-with-slack-notification-14643


# Publish Instagram carousel posts from product collections with Slack notification

## 1. Workflow Overview

This workflow publishes an Instagram carousel post from a product collection received through a webhook, then sends a Slack notification and returns a JSON response to the caller.

Its main purpose is to automate a “collection drop” social publishing flow where an external system sends a payload describing a set of product slides, and the workflow transforms that payload into a live Instagram carousel post.

### 1.1 Input Reception and Validation
The workflow starts with a webhook that accepts a POST payload. It validates the structure of the request, enforces Instagram carousel constraints, and ensures the caption input is present.

### 1.2 Caption Preparation
After validation, the workflow constructs a final Instagram caption using collection metadata, product lines, CTA text, and hashtags, while enforcing Instagram’s caption length limit.

### 1.3 Slide-by-Slide Media Processing
The slides are processed one by one using a batch loop. Each image is downloaded, uploaded to a public URL service, and then submitted to the Instagram Graph API as a child carousel media container.

### 1.4 Carousel Assembly
Once all child containers are created, their IDs are aggregated into a comma-separated list. The workflow then creates a parent Instagram carousel container containing the caption and all child media references.

### 1.5 Publishing, Notification, and Response
After a short wait to allow Instagram processing, the parent carousel is published. The workflow retrieves the published post metadata, posts a formatted Slack notification, and responds to the original webhook with a success payload.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

**Overview:**  
This block receives the incoming collection payload and performs basic structural validation before any external API call is made. It prevents invalid carousels from reaching Instagram and fails early with descriptive errors.

**Nodes Involved:**  
- Webhook Receive Payload1
- Code Validate Payload1

### Node: Webhook Receive Payload1
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for the workflow. Accepts incoming HTTP POST requests.
- **Configuration choices:**  
  - Path: `ig-carousel-drop`
  - Method: `POST`
  - Response mode: handled by a separate Respond to Webhook node
- **Key expressions or variables used:**  
  None in node parameters.
- **Input and output connections:**  
  - Input: none
  - Output: `Code Validate Payload1`
- **Version-specific requirements:**  
  Uses webhook node version 2.
- **Edge cases or potential failure types:**  
  - Wrong HTTP method
  - Caller sends malformed JSON
  - Caller expects immediate auto-response while workflow is configured for response node mode
- **Sub-workflow reference:**  
  None

### Node: Code Validate Payload1
- **Type and technical role:** `n8n-nodes-base.code`  
  Validates payload structure and applies Instagram slide-count rules.
- **Configuration choices:**  
  The JavaScript:
  - Ensures `slides` exists and is an array
  - Requires at least 2 slides
  - Truncates to 10 slides if more are provided
  - Ensures every slide has an `imageUrl`
  - Enforces HTTPS on each `imageUrl`
  - Requires `caption`
- **Key expressions or variables used:**  
  - `$input.item.json`
- **Input and output connections:**  
  - Input: `Webhook Receive Payload1`
  - Output: `Code Build Caption1`
- **Version-specific requirements:**  
  Uses Code node version 2.
- **Edge cases or potential failure types:**  
  - `slides` missing or not an array
  - Fewer than 2 slides
  - Missing `imageUrl`
  - Non-HTTPS image URLs
  - Missing `caption`
  - Silent truncation from more than 10 slides down to 10, which may surprise upstream systems
- **Sub-workflow reference:**  
  None

---

## 2.2 Caption Preparation

**Overview:**  
This block transforms the validated payload into a formatted Instagram caption and prepares metadata used later in the workflow. It also calculates the slide count and normalizes hashtags.

**Nodes Involved:**  
- Code Build Caption1

### Node: Code Build Caption1
- **Type and technical role:** `n8n-nodes-base.code`  
  Builds the final `fullCaption` and enriches the item with reusable metadata.
- **Configuration choices:**  
  The JavaScript:
  - Defaults `collectionName` to `New Collection`
  - Defaults `hook` to `Swipe to see the full collection`
  - Defaults `cta` to `Link in bio to shop`
  - Creates one line per product slide using title and optional price/currency
  - Normalizes hashtags so each starts with `#`
  - Composes a multi-part caption
  - Truncates the final caption to 2,200 characters
  - Returns:
    - `collectionName`
    - `fullCaption`
    - `slideCount`
- **Key expressions or variables used:**  
  - `$input.item.json`
- **Input and output connections:**  
  - Input: `Code Validate Payload1`
  - Output: `Split In Batches1`
- **Version-specific requirements:**  
  Uses Code node version 2.
- **Edge cases or potential failure types:**  
  - `slides` array exists but contains missing optional fields like `title` or `price`; handled with defaults
  - Caption truncation may cut marketing text unexpectedly
  - Currency formatting is simplistic and does not validate locale or numeric precision
- **Sub-workflow reference:**  
  None

---

## 2.3 Slide-by-Slide Media Processing

**Overview:**  
This block loops through each slide and prepares each image for Instagram carousel publishing. Instagram requires publicly accessible media URLs, so the workflow downloads each source image, re-uploads it to a public URL service, and creates one Instagram child container per slide.

**Nodes Involved:**  
- Split In Batches1
- HTTP Fetch Slide Image1
- Upload to URL1
- Code Create Child Container1

### Node: Split In Batches1
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through slides one at a time and then releases aggregated results when looping is complete.
- **Configuration choices:**  
  No custom batch options are set.
- **Key expressions or variables used:**  
  Its `context.currentRunIndex` is referenced downstream.
- **Input and output connections:**  
  - Input: `Code Build Caption1` and loopback from `Code Create Child Container1`
  - Output 1: `HTTP Fetch Slide Image1` for per-slide processing
  - Output 2: `Code Aggregate Child IDs1` after loop completion
- **Version-specific requirements:**  
  Uses Split In Batches version 3.
- **Edge cases or potential failure types:**  
  - Loop behavior depends on upstream item structure
  - Misuse of `currentRunIndex` can cause off-by-one or undefined access if the incoming item shape changes
- **Sub-workflow reference:**  
  None

### Node: HTTP Fetch Slide Image1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the current slide image as binary data.
- **Configuration choices:**  
  - URL is dynamically pulled from the slide at the current batch index
  - Response format is file/binary
- **Key expressions or variables used:**  
  - `$('Code Build Caption1').item.json.slides[$('Split In Batches1').context.currentRunIndex].imageUrl`
- **Input and output connections:**  
  - Input: `Split In Batches1`
  - Output: `Upload to URL1`
- **Version-specific requirements:**  
  Uses HTTP Request node version 4.2.
- **Edge cases or potential failure types:**  
  - Source image URL unavailable or returns non-200 response
  - Timeout on large images
  - Invalid content type
  - Expression failure if `currentRunIndex` is out of bounds
- **Sub-workflow reference:**  
  None

### Node: Upload to URL1
- **Type and technical role:** `n8n-nodes-uploadtourl.uploadToUrl`  
  Uploads the binary image to a hosting/CDN service to obtain a public URL.
- **Configuration choices:**  
  No explicit parameters are shown, so it relies on the node’s default binary handling behavior.
- **Key expressions or variables used:**  
  None directly shown.
- **Input and output connections:**  
  - Input: `HTTP Fetch Slide Image1`
  - Output: `Code Create Child Container1`
- **Version-specific requirements:**  
  Uses UploadToURL node version 1.  
  This node may require installation of a community/custom node package in the target n8n instance.
- **Edge cases or potential failure types:**  
  - Missing binary input
  - Upload provider outage or size limits
  - Returned schema may differ; downstream code tries multiple possible public URL fields
- **Sub-workflow reference:**  
  None

### Node: Code Create Child Container1
- **Type and technical role:** `n8n-nodes-base.code`  
  Calls the Instagram Graph API to create a carousel child media container for the uploaded image.
- **Configuration choices:**  
  The JavaScript:
  - Extracts a public URL from one of several possible keys:
    - `public_url`
    - `url`
    - `file_url`
    - `cdn_url`
  - Reads:
    - `IG_USER_ID`
    - `IG_ACCESS_TOKEN`
    from environment variables
  - Sends a POST request to `https://graph.facebook.com/v19.0/{igUserId}/media`
  - Uses:
    - `image_url`
    - `media_type: IMAGE`
    - `is_carousel_item: true`
    - `access_token`
  - Returns `childContainerId`
- **Key expressions or variables used:**  
  - `$input.item.json`
  - `$env.IG_USER_ID`
  - `$env.IG_ACCESS_TOKEN`
- **Input and output connections:**  
  - Input: `Upload to URL1`
  - Output: loops back to `Split In Batches1`
- **Version-specific requirements:**  
  Uses Code node version 2 with async `fetch`.
- **Edge cases or potential failure types:**  
  - No public URL returned by upload node
  - Missing environment variables
  - Instagram API authentication failure
  - Invalid image format, inaccessible URL, or unsupported dimensions
  - Rate limiting or transient Graph API errors
- **Sub-workflow reference:**  
  None

---

## 2.4 Carousel Assembly

**Overview:**  
This block collects all generated child container IDs and creates the parent carousel container in Instagram. The caption is attached only at this parent level.

**Nodes Involved:**  
- Code Aggregate Child IDs1
- IG Create Carousel Container1
- Wait IG Buffer1

### Node: Code Aggregate Child IDs1
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates all child media container IDs and reconstructs final metadata for the parent creation step.
- **Configuration choices:**  
  The JavaScript:
  - Uses `$input.all()` to collect all loop outputs
  - Reads prepared metadata from `Code Build Caption1`
  - Filters valid `childContainerId` values
  - Requires at least 2 child containers
  - Joins child IDs as a comma-separated string
  - Returns:
    - `childrenParam`
    - `fullCaption`
    - `slideCount`
    - `collectionName`
    - `igUserId`
    - `accessToken`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `$('Code Build Caption1').first().json`
  - `$env.IG_USER_ID`
  - `$env.IG_ACCESS_TOKEN`
- **Input and output connections:**  
  - Input: completion output of `Split In Batches1`
  - Output: `IG Create Carousel Container1`
- **Version-specific requirements:**  
  Uses Code node version 2.
- **Edge cases or potential failure types:**  
  - Fewer than 2 successful child containers
  - Cross-node reference failure if the earlier node is renamed
  - Missing environment variables
- **Sub-workflow reference:**  
  None

### Node: IG Create Carousel Container1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Creates the parent Instagram carousel container using the list of child container IDs.
- **Configuration choices:**  
  - POST to `https://graph.facebook.com/v19.0/{igUserId}/media`
  - Sends JSON body containing:
    - `media_type: CAROUSEL`
    - `children`
    - `caption`
    - `access_token`
- **Key expressions or variables used:**  
  - `{{ $json.igUserId }}`
  - `{{ JSON.stringify({ media_type: 'CAROUSEL', children: $json.childrenParam, caption: $json.fullCaption, access_token: $json.accessToken }) }}`
- **Input and output connections:**  
  - Input: `Code Aggregate Child IDs1`
  - Output: `Wait IG Buffer1`
- **Version-specific requirements:**  
  Uses HTTP Request node version 4.2.
- **Edge cases or potential failure types:**  
  - Invalid or expired Instagram token
  - Child container IDs not ready or invalid
  - Caption too long despite prior truncation if special character handling changes length unexpectedly
  - API version behavior differences in future Graph API versions
- **Sub-workflow reference:**  
  None

### Node: Wait IG Buffer1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays execution to give Instagram time to process carousel assets before publishing.
- **Configuration choices:**  
  - Waits 8 seconds
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `IG Create Carousel Container1`
  - Output: `IG Publish Carousel1`
- **Version-specific requirements:**  
  Uses Wait node version 1.
- **Edge cases or potential failure types:**  
  - 8 seconds may be insufficient under heavy API or media-processing load
  - Longer waits may be needed for large images or slower processing
- **Sub-workflow reference:**  
  None

---

## 2.5 Publishing, Notification, and Response

**Overview:**  
This block publishes the carousel, retrieves its metadata, alerts Slack, and returns a success response to the original webhook caller. It closes the automation loop with both internal and external confirmation.

**Nodes Involved:**  
- IG Publish Carousel1
- HTTP Fetch Post Metadata1
- Slack Notify Team1
- Respond to Webhook1

### Node: IG Publish Carousel1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls Instagram’s publish endpoint for the parent carousel container.
- **Configuration choices:**  
  - POST to `https://graph.facebook.com/v19.0/{igUserId}/media_publish`
  - Query params:
    - `creation_id` = parent container ID
    - `access_token` = Instagram token
- **Key expressions or variables used:**  
  - `$('Code Aggregate Child IDs1').item.json.igUserId`
  - `$('IG Create Carousel Container1').item.json.id`
  - `$env.IG_ACCESS_TOKEN`
- **Input and output connections:**  
  - Input: `Wait IG Buffer1`
  - Output: `HTTP Fetch Post Metadata1`
- **Version-specific requirements:**  
  Uses HTTP Request node version 4.2.
- **Edge cases or potential failure types:**  
  - Parent container not ready yet
  - Invalid `creation_id`
  - Token expiration or permission issues
- **Sub-workflow reference:**  
  None

### Node: HTTP Fetch Post Metadata1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves metadata for the newly published Instagram post.
- **Configuration choices:**  
  - GET request to `https://graph.facebook.com/v19.0/{publishedMediaId}`
  - Query params:
    - `fields=id,permalink,timestamp,media_type`
    - `access_token`
- **Key expressions or variables used:**  
  - `$('IG Publish Carousel1').item.json.id`
  - `$env.IG_ACCESS_TOKEN`
- **Input and output connections:**  
  - Input: `IG Publish Carousel1`
  - Output: `Slack Notify Team1`
- **Version-specific requirements:**  
  Uses HTTP Request node version 4.2.
- **Edge cases or potential failure types:**  
  - Metadata not immediately available
  - Missing permalink in edge cases
  - Token/permission errors
- **Sub-workflow reference:**  
  None

### Node: Slack Notify Team1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a notification message to a Slack channel.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Sends a text message to a channel ID from environment variables
- **Key expressions or variables used:**  
  - `{{ $env.SLACK_CHANNEL_ID }}`
  - Message body references:
    - `$('Code Build Caption').item.json.collectionName`
    - `$('Code Build Caption').item.json.slideCount`
    - `$('IG Publish Carousel').item.json.id`
    - `$('HTTP Fetch Post Metadata').item.json.permalink`
    - `{{ new Date().toLocaleString() }}`
- **Input and output connections:**  
  - Input: `HTTP Fetch Post Metadata1`
  - Output: `Respond to Webhook1`
- **Version-specific requirements:**  
  Uses Slack node version 2.3.
- **Edge cases or potential failure types:**  
  - OAuth2 credential issues
  - Invalid channel ID
  - Slack rate limits
  - Important: the expressions reference node names without the `1` suffix (`Code Build Caption`, `IG Publish Carousel`, `HTTP Fetch Post Metadata`), but the actual nodes are named `Code Build Caption1`, `IG Publish Carousel1`, and `HTTP Fetch Post Metadata1`. This will likely cause expression failures unless corrected.
- **Sub-workflow reference:**  
  None

### Node: Respond to Webhook1
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns the final JSON response to the original webhook request.
- **Configuration choices:**  
  - Response code: `200`
  - Respond with JSON
  - Response body includes:
    - `success`
    - `mediaId`
    - `permalink`
    - `slideCount`
    - `collectionName`
- **Key expressions or variables used:**  
  - `$('IG Publish Carousel1').item.json.id`
  - `$('HTTP Fetch Post Metadata1').item.json.permalink`
  - `$('Code Aggregate Child IDs1').item.json.slideCount`
  - `$('Code Aggregate Child IDs1').item.json.collectionName`
- **Input and output connections:**  
  - Input: `Slack Notify Team1`
  - Output: none
- **Version-specific requirements:**  
  Uses Respond to Webhook version 1.
- **Edge cases or potential failure types:**  
  - Upstream failure prevents response
  - JSON serialization issues are unlikely here but possible if referenced data is missing
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Template Overview1 | Sticky Note | Visual documentation |  |  | Instagram Image Carousel Post – Product Collection Drop<br>Receives a product collection via webhook, validates slides (2–10), and builds a carousel caption. Each image is fetched, uploaded via UploadToURL to get a public URL, then used to create Instagram child media containers. All slides are combined into a carousel, published, and a Slack notification is sent.<br>Flow:<br>Webhook → Validate → Build caption → Process slides → Upload images → Create IG containers → Publish carousel → Notify Slack<br>Required:<br>IG_USER_ID, IG_ACCESS_TOKEN, SLACK_CHANNEL_ID<br>Payload:<br>collectionName, caption, hook, cta, hashtags[],<br>slides[]: { imageUrl, title, price, currency } |
| Sticky Nodes 1- | Sticky Note | Visual documentation |  |  | Webhook – Receive Payload: Accepts POST requests to /ig-carousel and provides an inline response upon workflow completion.<br>Code – Validate: Verifies that the payload contains 2–10 valid HTTPS image URLs and required environment variables, failing fast with descriptive errors if criteria aren't met.<br>Code – Build Caption: Construct a structured Instagram caption including hooks, product lists, and CTAs, ensuring a safe limit of 2,200 characters. |
| Sticky Nodes 4- | Sticky Note | Visual documentation |  |  | Split In Batches: Iterates through the slides array one by one to trigger individual image processing, then routes to assembly once all items are processed.<br>HTTP – Fetch Slide Image: Downloads each slide's image as a binary file to prepare it for hosting.<br>Upload to URL: Uploads the binary to a CDN to generate a mandatory public HTTPS URL, which is required by Instagram's API for carousel items.<br>Code – Create Child Container: Communicates with the Instagram Graph API to generate a specific "child" media container marked as a carousel item, returning its unique ID. |
| Sticky Nodes 8- | Sticky Note | Visual documentation |  |  | Code – Aggregate Child IDs: Collects all individual child container IDs from the loop and joins them into a comma-separated string required for the parent API call. It also re-attaches the final caption and metadata via cross-node references.<br>HTTP – Create Carousel Container: Submits a POST request to the Instagram Graph API to create a parent container with the media_type set to CAROUSEL. The caption is applied exclusively to this parent container.<br>Wait 8s: Pauses the workflow for 8 seconds to allow Instagram sufficient time to validate and process the multiple assets within the carousel. |
| Sticky Nodes 11- | Sticky Note | Visual documentation |  |  | Nodes 11-14: Publish, Metadata, Notify, Respond<br>HTTP – Publish Carousel: Finalizes the process by calling the /media_publish endpoint with the parent container ID, returning the official live Instagram Post ID.<br>HTTP – Fetch Post Metadata: Automatically retrieves the live permalink, timestamp, and media type to provide a direct URL for the published content.<br>Slack – Notify Team: Dispatches a formatted alert to Slack containing the collection name, slide count, and a direct link to the new post.<br>Respond to Webhook: Concludes the workflow by returning a JSON success payload to the initial caller, including the media ID, permalink, and slide count. |
| Webhook Receive Payload1 | Webhook | Receives POST payload from caller |  | Code Validate Payload1 | Webhook – Receive Payload: Accepts POST requests to /ig-carousel and provides an inline response upon workflow completion.<br>Code – Validate: Verifies that the payload contains 2–10 valid HTTPS image URLs and required environment variables, failing fast with descriptive errors if criteria aren't met.<br>Code – Build Caption: Construct a structured Instagram caption including hooks, product lists, and CTAs, ensuring a safe limit of 2,200 characters. |
| Code Validate Payload1 | Code | Validates input payload and slide constraints | Webhook Receive Payload1 | Code Build Caption1 | Webhook – Receive Payload: Accepts POST requests to /ig-carousel and provides an inline response upon workflow completion.<br>Code – Validate: Verifies that the payload contains 2–10 valid HTTPS image URLs and required environment variables, failing fast with descriptive errors if criteria aren't met.<br>Code – Build Caption: Construct a structured Instagram caption including hooks, product lists, and CTAs, ensuring a safe limit of 2,200 characters. |
| Code Build Caption1 | Code | Builds final Instagram caption and metadata | Code Validate Payload1 | Split In Batches1 | Webhook – Receive Payload: Accepts POST requests to /ig-carousel and provides an inline response upon workflow completion.<br>Code – Validate: Verifies that the payload contains 2–10 valid HTTPS image URLs and required environment variables, failing fast with descriptive errors if criteria aren't met.<br>Code – Build Caption: Construct a structured Instagram caption including hooks, product lists, and CTAs, ensuring a safe limit of 2,200 characters. |
| Split In Batches1 | Split In Batches | Iterates through slides and then exits loop for aggregation | Code Build Caption1, Code Create Child Container1 | HTTP Fetch Slide Image1, Code Aggregate Child IDs1 | Split In Batches: Iterates through the slides array one by one to trigger individual image processing, then routes to assembly once all items are processed.<br>HTTP – Fetch Slide Image: Downloads each slide's image as a binary file to prepare it for hosting.<br>Upload to URL: Uploads the binary to a CDN to generate a mandatory public HTTPS URL, which is required by Instagram's API for carousel items.<br>Code – Create Child Container: Communicates with the Instagram Graph API to generate a specific "child" media container marked as a carousel item, returning its unique ID. |
| HTTP Fetch Slide Image1 | HTTP Request | Downloads each slide image as binary | Split In Batches1 | Upload to URL1 | Split In Batches: Iterates through the slides array one by one to trigger individual image processing, then routes to assembly once all items are processed.<br>HTTP – Fetch Slide Image: Downloads each slide's image as a binary file to prepare it for hosting.<br>Upload to URL: Uploads the binary to a CDN to generate a mandatory public HTTPS URL, which is required by Instagram's API for carousel items.<br>Code – Create Child Container: Communicates with the Instagram Graph API to generate a specific "child" media container marked as a carousel item, returning its unique ID. |
| Upload to URL1 | UploadToURL | Hosts binary image at a public URL | HTTP Fetch Slide Image1 | Code Create Child Container1 | Split In Batches: Iterates through the slides array one by one to trigger individual image processing, then routes to assembly once all items are processed.<br>HTTP – Fetch Slide Image: Downloads each slide's image as a binary file to prepare it for hosting.<br>Upload to URL: Uploads the binary to a CDN to generate a mandatory public HTTPS URL, which is required by Instagram's API for carousel items.<br>Code – Create Child Container: Communicates with the Instagram Graph API to generate a specific "child" media container marked as a carousel item, returning its unique ID. |
| Code Create Child Container1 | Code | Creates Instagram carousel child media container | Upload to URL1 | Split In Batches1 | Split In Batches: Iterates through the slides array one by one to trigger individual image processing, then routes to assembly once all items are processed.<br>HTTP – Fetch Slide Image: Downloads each slide's image as a binary file to prepare it for hosting.<br>Upload to URL: Uploads the binary to a CDN to generate a mandatory public HTTPS URL, which is required by Instagram's API for carousel items.<br>Code – Create Child Container: Communicates with the Instagram Graph API to generate a specific "child" media container marked as a carousel item, returning its unique ID. |
| Code Aggregate Child IDs1 | Code | Joins child container IDs and restores metadata | Split In Batches1 | IG Create Carousel Container1 | Code – Aggregate Child IDs: Collects all individual child container IDs from the loop and joins them into a comma-separated string required for the parent API call. It also re-attaches the final caption and metadata via cross-node references.<br>HTTP – Create Carousel Container: Submits a POST request to the Instagram Graph API to create a parent container with the media_type set to CAROUSEL. The caption is applied exclusively to this parent container.<br>Wait 8s: Pauses the workflow for 8 seconds to allow Instagram sufficient time to validate and process the multiple assets within the carousel. |
| IG Create Carousel Container1 | HTTP Request | Creates Instagram parent carousel container | Code Aggregate Child IDs1 | Wait IG Buffer1 | Code – Aggregate Child IDs: Collects all individual child container IDs from the loop and joins them into a comma-separated string required for the parent API call. It also re-attaches the final caption and metadata via cross-node references.<br>HTTP – Create Carousel Container: Submits a POST request to the Instagram Graph API to create a parent container with the media_type set to CAROUSEL. The caption is applied exclusively to this parent container.<br>Wait 8s: Pauses the workflow for 8 seconds to allow Instagram sufficient time to validate and process the multiple assets within the carousel. |
| Wait IG Buffer1 | Wait | Adds delay before publish | IG Create Carousel Container1 | IG Publish Carousel1 | Code – Aggregate Child IDs: Collects all individual child container IDs from the loop and joins them into a comma-separated string required for the parent API call. It also re-attaches the final caption and metadata via cross-node references.<br>HTTP – Create Carousel Container: Submits a POST request to the Instagram Graph API to create a parent container with the media_type set to CAROUSEL. The caption is applied exclusively to this parent container.<br>Wait 8s: Pauses the workflow for 8 seconds to allow Instagram sufficient time to validate and process the multiple assets within the carousel. |
| IG Publish Carousel1 | HTTP Request | Publishes the parent Instagram carousel container | Wait IG Buffer1 | HTTP Fetch Post Metadata1 | Nodes 11-14: Publish, Metadata, Notify, Respond<br>HTTP – Publish Carousel: Finalizes the process by calling the /media_publish endpoint with the parent container ID, returning the official live Instagram Post ID.<br>HTTP – Fetch Post Metadata: Automatically retrieves the live permalink, timestamp, and media type to provide a direct URL for the published content.<br>Slack – Notify Team: Dispatches a formatted alert to Slack containing the collection name, slide count, and a direct link to the new post.<br>Respond to Webhook: Concludes the workflow by returning a JSON success payload to the initial caller, including the media ID, permalink, and slide count. |
| HTTP Fetch Post Metadata1 | HTTP Request | Retrieves permalink and metadata of published post | IG Publish Carousel1 | Slack Notify Team1 | Nodes 11-14: Publish, Metadata, Notify, Respond<br>HTTP – Publish Carousel: Finalizes the process by calling the /media_publish endpoint with the parent container ID, returning the official live Instagram Post ID.<br>HTTP – Fetch Post Metadata: Automatically retrieves the live permalink, timestamp, and media type to provide a direct URL for the published content.<br>Slack – Notify Team: Dispatches a formatted alert to Slack containing the collection name, slide count, and a direct link to the new post.<br>Respond to Webhook: Concludes the workflow by returning a JSON success payload to the initial caller, including the media ID, permalink, and slide count. |
| Slack Notify Team1 | Slack | Sends publication alert to Slack | HTTP Fetch Post Metadata1 | Respond to Webhook1 | Nodes 11-14: Publish, Metadata, Notify, Respond<br>HTTP – Publish Carousel: Finalizes the process by calling the /media_publish endpoint with the parent container ID, returning the official live Instagram Post ID.<br>HTTP – Fetch Post Metadata: Automatically retrieves the live permalink, timestamp, and media type to provide a direct URL for the published content.<br>Slack – Notify Team: Dispatches a formatted alert to Slack containing the collection name, slide count, and a direct link to the new post.<br>Respond to Webhook: Concludes the workflow by returning a JSON success payload to the initial caller, including the media ID, permalink, and slide count. |
| Respond to Webhook1 | Respond to Webhook | Returns final success JSON to caller | Slack Notify Team1 |  | Nodes 11-14: Publish, Metadata, Notify, Respond<br>HTTP – Publish Carousel: Finalizes the process by calling the /media_publish endpoint with the parent container ID, returning the official live Instagram Post ID.<br>HTTP – Fetch Post Metadata: Automatically retrieves the live permalink, timestamp, and media type to provide a direct URL for the published content.<br>Slack – Notify Team: Dispatches a formatted alert to Slack containing the collection name, slide count, and a direct link to the new post.<br>Respond to Webhook: Concludes the workflow by returning a JSON success payload to the initial caller, including the media ID, permalink, and slide count. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node**
   - Type: `Webhook`
   - Name it `Webhook Receive Payload1`
   - Set:
     - HTTP Method: `POST`
     - Path: `ig-carousel-drop`
     - Response Mode: `Using Respond to Webhook Node`
   - This will be the entry point.

3. **Add a Code node for validation**
   - Type: `Code`
   - Name it `Code Validate Payload1`
   - Connect `Webhook Receive Payload1` → `Code Validate Payload1`
   - Paste logic that:
     - checks `slides` exists and is an array
     - requires at least 2 slides
     - trims slide count to 10
     - requires each `slides[i].imageUrl`
     - enforces `https://`
     - requires `caption`
   - Return the validated item.

4. **Add a Code node for caption building**
   - Type: `Code`
   - Name it `Code Build Caption1`
   - Connect `Code Validate Payload1` → `Code Build Caption1`
   - Configure logic to:
     - set default `collectionName`
     - set default `hook`
     - set default `cta`
     - generate one line per slide with title and optional price
     - normalize hashtags
     - assemble `fullCaption`
     - truncate to 2,200 characters
     - return `fullCaption`, `slideCount`, `collectionName`, and original payload fields

5. **Add a Split In Batches node**
   - Type: `Split In Batches`
   - Name it `Split In Batches1`
   - Connect `Code Build Caption1` → `Split In Batches1`
   - Leave default settings unless you want explicit batch size behavior.

6. **Add an HTTP Request node to fetch each image**
   - Type: `HTTP Request`
   - Name it `HTTP Fetch Slide Image1`
   - Connect `Split In Batches1` main output to this node
   - Set URL expression to read the current slide image URL using the batch index:
     - from `Code Build Caption1`
     - indexed by `Split In Batches1.context.currentRunIndex`
   - Set response format to binary/file

7. **Add the UploadToURL node**
   - Type: `Upload to URL`
   - Name it `Upload to URL1`
   - Connect `HTTP Fetch Slide Image1` → `Upload to URL1`
   - Keep default configuration unless your installed node requires target provider settings
   - Ensure it accepts the incoming binary and returns a public URL
   - If this node is not available, install the required community/custom package in your n8n instance

8. **Add a Code node to create Instagram child containers**
   - Type: `Code`
   - Name it `Code Create Child Container1`
   - Connect `Upload to URL1` → `Code Create Child Container1`
   - Configure code to:
     - extract a public URL from upload output
     - read environment variables:
       - `IG_USER_ID`
       - `IG_ACCESS_TOKEN`
     - call `https://graph.facebook.com/v19.0/{IG_USER_ID}/media`
     - send:
       - `image_url`
       - `media_type: IMAGE`
       - `is_carousel_item: true`
       - `access_token`
     - return `childContainerId`

9. **Create the loopback**
   - Connect `Code Create Child Container1` back to `Split In Batches1`
   - This is required so the next slide can be processed.

10. **Add a Code node to aggregate child container IDs**
    - Type: `Code`
    - Name it `Code Aggregate Child IDs1`
    - Connect the second/main completion output of `Split In Batches1` to `Code Aggregate Child IDs1`
    - Configure logic to:
      - collect all input items via `$input.all()`
      - extract all `childContainerId` values
      - require at least 2 valid child containers
      - join them with commas into `childrenParam`
      - re-read `fullCaption`, `slideCount`, and `collectionName` from `Code Build Caption1`
      - include `IG_USER_ID` and `IG_ACCESS_TOKEN`

11. **Add an HTTP Request node for parent carousel creation**
    - Type: `HTTP Request`
    - Name it `IG Create Carousel Container1`
    - Connect `Code Aggregate Child IDs1` → `IG Create Carousel Container1`
    - Configure:
      - Method: `POST`
      - URL: `https://graph.facebook.com/v19.0/{{ $json.igUserId }}/media`
      - Send Body: enabled
      - Body Type: JSON
      - JSON body should contain:
        - `media_type: "CAROUSEL"`
        - `children: $json.childrenParam`
        - `caption: $json.fullCaption`
        - `access_token: $json.accessToken`

12. **Add a Wait node**
    - Type: `Wait`
    - Name it `Wait IG Buffer1`
    - Connect `IG Create Carousel Container1` → `Wait IG Buffer1`
    - Set:
      - Amount: `8`
      - Unit: `seconds`

13. **Add an HTTP Request node to publish the carousel**
    - Type: `HTTP Request`
    - Name it `IG Publish Carousel1`
    - Connect `Wait IG Buffer1` → `IG Publish Carousel1`
    - Configure:
      - Method: `POST`
      - URL: `https://graph.facebook.com/v19.0/{{ $('Code Aggregate Child IDs1').item.json.igUserId }}/media_publish`
      - Send Query Parameters: enabled
      - Query parameters:
        - `creation_id` = `{{ $('IG Create Carousel Container1').item.json.id }}`
        - `access_token` = `{{ $env.IG_ACCESS_TOKEN }}`

14. **Add an HTTP Request node to fetch post metadata**
    - Type: `HTTP Request`
    - Name it `HTTP Fetch Post Metadata1`
    - Connect `IG Publish Carousel1` → `HTTP Fetch Post Metadata1`
    - Configure:
      - URL: `https://graph.facebook.com/v19.0/{{ $('IG Publish Carousel1').item.json.id }}`
      - Query parameters:
        - `fields` = `id,permalink,timestamp,media_type`
        - `access_token` = `{{ $env.IG_ACCESS_TOKEN }}`

15. **Add a Slack node**
    - Type: `Slack`
    - Name it `Slack Notify Team1`
    - Connect `HTTP Fetch Post Metadata1` → `Slack Notify Team1`
    - Configure:
      - Authentication: `OAuth2`
      - Operation: send message to channel
      - Channel ID: `{{ $env.SLACK_CHANNEL_ID }}`
      - Message text with collection name, slide count, post ID, permalink, and timestamp

16. **Important correction for Slack expressions**
    - The provided workflow contains broken node references in the Slack text.
    - Use the actual node names:
      - `Code Build Caption1`
      - `IG Publish Carousel1`
      - `HTTP Fetch Post Metadata1`
    - Do not use the unsuffixed names.

17. **Add a Respond to Webhook node**
    - Type: `Respond to Webhook`
    - Name it `Respond to Webhook1`
    - Connect `Slack Notify Team1` → `Respond to Webhook1`
    - Configure:
      - Response Code: `200`
      - Respond With: `JSON`
      - Build response body containing:
        - `success: true`
        - published `mediaId`
        - `permalink`
        - `slideCount`
        - `collectionName`

18. **Set environment variables in n8n**
    - Required:
      - `IG_USER_ID`
      - `IG_ACCESS_TOKEN`
      - `SLACK_CHANNEL_ID`
    - Ensure the n8n instance exposes these variables to workflow expressions and code nodes.

19. **Configure credentials**
    - **Slack OAuth2 credential**
      - Create/select a Slack OAuth2 credential in n8n
      - Ensure permission to post messages to the target channel
    - **Instagram/Facebook Graph**
      - This workflow uses raw HTTP/code requests with access tokens rather than a dedicated credential
      - Ensure the token in `IG_ACCESS_TOKEN` is valid and has required Instagram Graph API permissions

20. **Optional but strongly recommended hardening**
    - Add validation for:
      - presence of `IG_USER_ID` and `IG_ACCESS_TOKEN` before processing
      - image size and format
      - upstream payload schema
    - Add retries for:
      - image download
      - upload hosting
      - Graph API calls
    - Consider increasing wait time if Instagram processing is slow.

21. **Test with a sample payload**
   Example structure:
   - `collectionName`
   - `caption`
   - `hook`
   - `cta`
   - `hashtags`: array of strings
   - `slides`: array of 2 to 10 objects, each containing:
     - `imageUrl`
     - `title`
     - `price`
     - `currency`

22. **Activate the workflow**
   - Copy the production webhook URL
   - Send a POST request with a valid JSON body
   - Confirm:
     - child containers are created
     - parent carousel is published
     - Slack receives notification
     - webhook caller receives final JSON success payload

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Required environment variables: `IG_USER_ID`, `IG_ACCESS_TOKEN`, `SLACK_CHANNEL_ID` | Runtime configuration |
| Expected payload fields: `collectionName`, `caption`, `hook`, `cta`, `hashtags[]`, `slides[]: { imageUrl, title, price, currency }` | Input contract |
| The workflow intentionally trims slide arrays above 10 items instead of failing. | Instagram carousel constraint handling |
| Slack node expressions in the provided workflow reference incorrect node names without the `1` suffix and should be fixed before production use. | Workflow quality note |
| The UploadToURL node appears to be a non-core/community node and may need to be installed separately in the target n8n instance. | Deployment consideration |