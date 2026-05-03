Create event recap Instagram carousels from photo dumps using Upload to URL

https://n8nworkflows.xyz/workflows/create-event-recap-instagram-carousels-from-photo-dumps-using-upload-to-url-14656


# Create event recap Instagram carousels from photo dumps using Upload to URL

# Technical Documentation: Instagram Event Recap Carousel Workflow

## 1. Workflow Overview

This workflow automates the creation and publication of Instagram Carousel posts from a set of event photos (a "photo dump"). It transforms a raw event payload received via webhook into a structured storytelling post, handling the complex requirements of the Instagram Graph API—specifically the need for publicly accessible image URLs and the multi-step process of creating child containers before the parent carousel.

The logic is divided into three primary functional blocks:
- **1.1 Input & Data Preparation:** Handles the webhook trigger, validates image URLs, and generates a dynamic storytelling caption based on event metadata.
- **1.2 Image Processing & Containerization:** Downloads images, uploads them to a public CDN, and registers each image as an Instagram child media container.
- **1.3 Carousel Assembly & Distribution:** Merges individual containers into a single carousel, publishes the post, logs the result in Airtable, and notifies the team via Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Input & Data Preparation
**Overview:** This block serves as the entry point. it ensures the incoming data is valid and converts a single request containing an array of photos into multiple individual items to enable parallel processing.

- **Nodes Involved:** `Webhook Receive Event1`, `Code Validate Caption and Split Photos1`.
- **Node Details:**
    - **Webhook Receive Event1**: 
        - *Role:* HTTP Listener.
        - *Configuration:* Listens for `POST` requests at `/ig-event-recap`.
        - *Input/Output:* Entry point $\rightarrow$ `Code Validate Caption and Split Photos1`.
    - **Code Validate Caption and Split Photos1**:
        - *Role:* Data Validation and Transformation.
        - *Logic:* Validates that at least 2 unique HTTPS photos exist (max 10). It constructs a multi-line caption using `eventName`, `location`, `eventDate`, `attendees`, `caption`, `highlights`, and `hashtags`.
        - *Key Expressions:* Uses `$env.IG_USER_ID` and `$env.IG_ACCESS_TOKEN` for downstream use.
        - *Output:* Returns an array of objects (one per photo), effectively "fanning out" the workflow.
        - *Failure Types:* Throws errors if `eventName` is missing or if fewer than 2 valid URLs are provided.

### 2.2 Image Processing & Containerization
**Overview:** This block processes each image individually. Since Instagram requires a public URL to "pull" the image, this block fetches the binary and re-uploads it to a dedicated CDN.

- **Nodes Involved:** `HTTP Fetch Photo Binary1`, `Upload to URL1`, `Code Create Child Container1`, `Merge All Child Items1`.
- **Node Details:**
    - **HTTP Fetch Photo Binary1**:
        - *Role:* Binary downloader.
        - *Configuration:* Set to `responseFormat: file`. Fetches the `photoUrl`.
    - **Upload to URL1**:
        - *Role:* CDN Upload.
        - *Configuration:* Uses UploadToURL credentials to generate a public HTTPS link from the binary.
    - **Code Create Child Container1**:
        - *Role:* Instagram API Integration.
        - *Logic:* Performs an asynchronous `fetch` to `graph.facebook.com` to create a media container with `is_carousel_item: true`.
        - *Edge Cases:* Fails if the CDN URL is missing or if the Instagram API returns an error.
    - **Merge All Child Items1**:
        - *Role:* Synchronization Point (Fan-in).
        - *Configuration:* Waits for all parallel image processes to complete before proceeding.

### 2.3 Carousel Assembly & Distribution
**Overview:** This block consolidates the individual child containers into one carousel post and handles the final publication and logging.

- **Nodes Involved:** `Code Build Children Param1`, `IG Create Carousel Container1`, `Wait IG Buffer1`, `IG Publish Carousel1`, `HTTP Fetch Post Metadata1`, `Airtable Log Published Post1`, `Slack Notify Team1`, `Respond to Webhook1`.
- **Node Details:**
    - **Code Build Children Param1**: 
        - *Role:* String Aggregator.
        - *Logic:* Sorts items by `slideIndex` and joins `childContainerId`s into a comma-separated string required by the Instagram API.
    - **IG Create Carousel Container1**:
        - *Role:* Parent Container Creation.
        - *Configuration:* `POST` to Instagram API using `media_type: CAROUSEL` and the combined `children` string.
    - **Wait IG Buffer1**:
        - *Role:* Latency Buffer.
        - *Configuration:* 8-second delay to ensure Instagram has processed the carousel container before publication.
    - **IG Publish Carousel1**:
        - *Role:* Final Publication.
        - *Configuration:* Calls `/media_publish` using the `creation_id` from the previous step.
    - **HTTP Fetch Post Metadata1**:
        - *Role:* Data Retrieval.
        - *Configuration:* Fetches the `permalink` and `timestamp` of the published post.
    - **Airtable Log Published Post1**:
        - *Role:* Database Logging.
        - *Configuration:* Creates a record in "Event Posts Log" table using `$env.AIRTABLE_BASE_ID`.
    - **Slack Notify Team1**:
        - *Role:* Team Notification.
        - *Configuration:* Sends a formatted message to `$env.SLACK_CHANNEL_ID` with the post link.
    - **Respond to Webhook1**:
        - *Role:* Final Response.
        - *Configuration:* Returns a JSON object containing the success status, permalink, and event name.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook Receive Event1 | Webhook | Entry Point | - | Code Validate... | Webhook – Receive Event Payload... |
| Code Validate Caption... | Code | Validator/Splitter | Webhook... | HTTP Fetch Photo... | Code – Validate & Caption... |
| HTTP Fetch Photo Binary1 | HTTP Request | Binary Downloader | Code Validate... | Upload to URL1 | HTTP – Fetch Photo Binary... |
| Upload to URL1 | UploadToURL | CDN Upload | HTTP Fetch... | Code Create Child... | Upload to URL... |
| Code Create Child... | Code | IG Child Creator | Upload to URL1 | Merge All Child... | Code – Create Child Container... |
| Merge All Child Items1 | Merge | Synchronization | Code Create Child... | Code Build Children... | Merge: Acts as the "fan-in" point... |
| Code Build Children... | Code | Param Assembler | Merge All Child... | IG Create Carousel... | Code – Build Children Param... |
| IG Create Carousel... | HTTP Request | Parent Creator | Code Build... | Wait IG Buffer1 | HTTP – Create Carousel Container... |
| Wait IG Buffer1 | Wait | API Buffer | IG Create Carousel... | IG Publish Carousel1 | Wait 8s & Publish... |
| IG Publish Carousel1 | HTTP Request | Publisher | Wait IG Buffer1 | HTTP Fetch Post... | Wait 8s & Publish... |
| HTTP Fetch Post Metadata1 | HTTP Request | Metadata Retriever | IG Publish... | Airtable Log... | Log & Notify... |
| Airtable Log Published... | Airtable | Database Log | HTTP Fetch Post... | Slack Notify Team1 | Log & Notify... |
| Slack Notify Team1 | Slack | Notification | Airtable Log... | Respond to Webhook1 | Log & Notify... |
| Respond to Webhook1 | Respond to Webhook | API Response | Slack Notify... | - | Log & Notify... |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Environment Configuration
Set the following Environment Variables in n8n:
- `IG_USER_ID`: Your Instagram Business Account ID.
- `IG_ACCESS_TOKEN`: Long-lived Page Access Token.
- `SLACK_CHANNEL_ID`: The ID of the notification channel.
- `AIRTABLE_BASE_ID`: The ID of your Airtable base.

### Step 2: Input and Validation
1. Create a **Webhook Node**: Path `ig-event-recap`, Method `POST`, Response Mode `Response Node`.
2. Create a **Code Node**: 
    - Implement validation for `eventName` and `photos` (min 2, max 10).
    - Implement caption logic to join metadata into a single string.
    - Use `.map()` to return one object per photo to trigger the "fan-out".

### Step 3: Media Processing Loop
3. Create an **HTTP Request Node**: URL set to `{{ $json.photoUrl }}`, Response Format `File`.
4. Create an **UploadToURL Node**: Configure credentials to upload the binary and get a public URL.
5. Create a **Code Node**: 
    - Perform a `fetch` POST to `https://graph.facebook.com/v19.0/{{$env.IG_USER_ID}}/media`.
    - Body: `{ "image_url": publicUrl, "media_type": "IMAGE", "is_carousel_item": true, "access_token": $env.IG_ACCESS_TOKEN }`.
6. Create a **Merge Node**: Set to "Wait for all inputs to arrive" (Combine mode).

### Step 4: Carousel Finalization
7. Create a **Code Node**: Sort all incoming items by `slideIndex` and join `childContainerId` into a comma-separated string.
8. Create an **HTTP Request Node**: 
    - `POST` to `https://graph.facebook.com/v19.0/{{$json.igUserId}}/media`.
    - Body: `{ "media_type": "CAROUSEL", "children": childrenParam, "caption": fullCaption, "access_token": accessToken }`.
9. Create a **Wait Node**: Set to 8 seconds.
10. Create an **HTTP Request Node**: 
    - `POST` to `.../media_publish`.
    - Query Params: `creation_id` (from step 8) and `access_token`.

### Step 5: Post-Processing & Response
11. Create an **HTTP Request Node**: GET request to the Post ID to retrieve the `permalink`.
12. Create an **Airtable Node**: Operation `Create` in the "Event Posts Log" table. Map the Post ID and Permalink.
13. Create a **Slack Node**: Use `oAuth2` to send a formatted summary to the channel ID.
14. Create a **Respond to Webhook Node**: Send a JSON response back to the original caller containing the `permalink` and `success: true`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| All Instagram API calls utilize version v19.0 of the Graph API. | [Facebook Graph API Docs](https://developers.facebook.com/docs/graph-api) |
| The `UploadToURL` node is critical because Instagram cannot access local files or non-public URLs. | Integration dependency |
| Ensure the Instagram Access Token has `instagram_basic` and `instagram_content_publish` permissions. | Credential Requirements |