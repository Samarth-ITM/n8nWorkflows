Simple Bluesky multi-image post using native Bluesky API

https://n8nworkflows.xyz/workflows/simple-bluesky-multi-image-post-using-native-bluesky-api-2562


# Simple Bluesky multi-image post using native Bluesky API

### 1. Workflow Overview

This workflow automates posting multi-image content (1 to 4 images) with a caption to Bluesky using the native Bluesky API. It targets content creators, businesses, and social media managers who want to streamline image post publishing on Bluesky, saving time and ensuring consistent delivery.

The workflow is logically divided into these blocks:

- **1.1 Trigger Initialization**: Manual start node to kick off the workflow.
- **1.2 Credentials and Session Setup**: Defining Bluesky credentials and creating an authenticated session.
- **1.3 Define Post Content**: Setting the post caption and image URLs to be posted.
- **1.4 Image Download and Upload**: Downloading each image from provided URLs, uploading them individually as blobs to Bluesky.
- **1.5 Post Creation**: Aggregating uploaded image references and posting the text caption with embedded images to Bluesky.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Initialization

- **Overview:**  
  Starts the workflow execution on manual trigger, adaptable to other triggers (e.g., webhook, schedule).

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution manually.  
    - Config: No parameters; default manual trigger.  
    - Input/Output: No inputs; output connects to Define Credentials.  
    - Edge Cases: None.  
    - Notes: Adaptable to other triggers as needed.

---

#### 2.2 Credentials and Session Setup

- **Overview:**  
  Sets Bluesky user credentials (username and app password) and uses them to create an authenticated session for API calls.

- **Nodes Involved:**  
  - Define Credentials  
  - Create Bluesky Session  
  - Sticky Note1 (Instructions for credentials)

- **Node Details:**  
  - **Define Credentials**  
    - Type: Set node  
    - Role: Stores Bluesky identifier and app password in JSON format.  
    - Config: Raw JSON with keys `identifier` and `password`.  
    - Expressions: Not dynamic here; static placeholder values.  
    - Connections: Output to Create Bluesky Session.  
    - Edge Cases: Wrong credentials cause session creation failure.  
    - Notes: Must replace placeholder with real username and app password from [Bluesky app passwords](https://bsky.app/settings/app-passwords).  
  - **Create Bluesky Session**  
    - Type: HTTP Request  
    - Role: Performs POST to Bluesky API to create authenticated session using credentials.  
    - Config:  
      - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`  
      - Method: POST  
      - Body: JSON from Define Credentials node’s credentials object.  
      - Sends JSON body, expects JSON response containing session tokens.  
    - Input: From Define Credentials output.  
    - Output: JSON with session tokens including `accessJwt` and `did`.  
    - Edge Cases: Authentication errors, API downtime, malformed credentials.  
    - Version: HTTP Request v4.2.  
  - **Sticky Note1**  
    - Explains credential setup requirements and links to Bluesky app password creation page.

---

#### 2.3 Define Post Content

- **Overview:**  
  Sets the text caption for the post and specifies 1-4 image URLs to be posted.

- **Nodes Involved:**  
  - Set Caption  
  - Set Images  
  - Sticky Note (caption and images instructions)

- **Node Details:**  
  - **Set Caption**  
    - Type: Set node  
    - Role: Defines the text content of the post, max 300 chars to comply with Bluesky limits.  
    - Config: Assigns property `Caption Text` with static string text.  
    - Input: From Create Bluesky Session output.  
    - Output: To Set Images.  
    - Edge Cases: Exceeding 300 chars causes API error.  
  - **Set Images**  
    - Type: Set node  
    - Role: Defines the array `photos` with image URLs (up to 4).  
    - Config: Raw JSON array of objects with `url` properties.  
    - Input: From Set Caption output.  
    - Output: To Split Out.  
    - Edge Cases: URLs must be accessible; images >1MB may cause issues.  
  - **Sticky Note**  
    - Provides instructions on setting caption and image URLs, including limitations.

---

#### 2.4 Image Download and Upload

- **Overview:**  
  Splits the array of image URLs, downloads each image, uploads each as a blob to Bluesky, then prepares data for embedding.

- **Nodes Involved:**  
  - Split Out  
  - Download Images  
  - Post Image to Bluesky  
  - Code  
  - Aggregate  
  - Sticky Note2 (image upload explanation)

- **Node Details:**  
  - **Split Out**  
    - Type: SplitOut  
    - Role: Splits the `photos` array into individual items for sequential processing.  
    - Config: Splits on field `photos`.  
    - Input: From Set Images output.  
    - Output: To Download Images.  
  - **Download Images**  
    - Type: HTTP Request  
    - Role: Downloads image binary data from each URL.  
    - Config: URL set dynamically from current item’s `url`.  
    - Input: From Split Out.  
    - Output: To Post Image to Bluesky.  
    - Edge Cases: Invalid URLs, network errors, timeouts.  
  - **Post Image to Bluesky**  
    - Type: HTTP Request  
    - Role: Uploads downloaded image binary data as blob to Bluesky.  
    - Config:  
      - URL: `https://bsky.social/xrpc/com.atproto.repo.uploadBlob`  
      - Method: POST  
      - Content type: binary data with image in `data` field.  
      - Header Authorization: Bearer token from Create Bluesky Session node’s `accessJwt`.  
    - Input: Binary data from Download Images.  
    - Output: To Code node.  
    - Edge Cases: Authorization errors, upload failures, large image size.  
  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Processes each uploaded blob response, constructs image embed objects with alt text and blob info.  
    - Config: Returns an array of objects with `alt` (fixed to "-") and `image` properties from blob response.  
    - Input: From Post Image to Bluesky output for each image.  
    - Output: To Aggregate node.  
    - Edge Cases: Code errors, unexpected response formats.  
  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects all individual image embed objects into a single array for the post payload.  
    - Config: Aggregates all items into one array under `data`.  
    - Input: From Code node.  
    - Output: To Post to Bluesky.  
  - **Sticky Note2**  
    - Explains that Bluesky requires images to be uploaded as blobs before embedding them in posts, linking official documentation.

---

#### 2.5 Post Creation

- **Overview:**  
  Posts the composed Bluesky feed record containing the caption and embedded images using the authenticated session.

- **Nodes Involved:**  
  - Post to Bluesky

- **Node Details:**  
  - **Post to Bluesky**  
    - Type: HTTP Request  
    - Role: Creates a Bluesky post record with text caption and embedded images.  
    - Config:  
      - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
      - Method: POST  
      - Headers: Authorization Bearer token from session’s `accessJwt`.  
      - Body: JSON including:  
        - `repo`: user DID from session (`did`)  
        - `collection`: fixed string `app.bsky.feed.post`  
        - `record`: object with  
          - `$type`: `app.bsky.feed.post`  
          - `text`: trimmed caption from Set Caption node  
          - `createdAt`: current timestamp (`$now`)  
          - `embed`: object with `$type` `app.bsky.embed.images` and array of images aggregated earlier.  
    - Input: From Aggregate.  
    - Output: Final workflow output.  
    - Edge Cases: Exceeding 300 char limit, invalid blobs, authorization expiration, API errors.  
    - Notes: To post text-only (no images), must remove `embed` section and related image nodes.

---

### 3. Summary Table

| Node Name                 | Node Type        | Functional Role                   | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                        |
|---------------------------|------------------|---------------------------------|-----------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger   | Starts workflow manually         |                             | Define Credentials          |                                                                                                                   |
| Define Credentials         | Set              | Stores Bluesky username & app password | When clicking ‘Test workflow’ | Create Bluesky Session      |                                                                                                                   |
| Create Bluesky Session     | HTTP Request     | Creates authenticated Bluesky session | Define Credentials           | Set Caption                | Explains credential setup & links to Bluesky app password setup page                                              |
| Set Caption               | Set              | Defines post text caption         | Create Bluesky Session       | Set Images                 | Instructions on setting caption and image URLs, mentions 300 char limit                                            |
| Set Images                | Set              | Defines array of image URLs       | Set Caption                  | Split Out                  | Instructions on setting caption and image URLs                                                                    |
| Split Out                 | SplitOut         | Splits photos array into items    | Set Images                   | Download Images            | Explains image attachment process                                                                                  |
| Download Images           | HTTP Request     | Downloads images from URLs        | Split Out                   | Post Image to Bluesky      | Explains image attachment process                                                                                  |
| Post Image to Bluesky     | HTTP Request     | Uploads images as blobs to Bluesky | Download Images              | Code                      | Explains image attachment process                                                                                  |
| Code                      | Code             | Prepares image embed objects      | Post Image to Bluesky        | Aggregate                  | Explains image attachment process                                                                                  |
| Aggregate                 | Aggregate        | Aggregates embed objects into array | Code                        | Post to Bluesky            | Explains image attachment process                                                                                  |
| Post to Bluesky           | HTTP Request     | Posts final feed record to Bluesky | Aggregate                   |                            | Explains image attachment process                                                                                  |
| Sticky Note1              | Sticky Note      | Credential setup instructions     |                             |                            | Explains Bluesky username and App Password requirements, links to https://bsky.app/settings/app-passwords         |
| Sticky Note               | Sticky Note      | Caption and image URL instructions |                             |                            | Explains caption and image URL settings, limits on counts and size                                                |
| Sticky Note2              | Sticky Note      | Image upload process explanation  |                             |                            | Links to https://docs.bsky.app/docs/tutorials/creating-a-post#images-embeds explaining image uploading and embedding |
| Sticky Note3              | Sticky Note      | Workflow purpose summary          |                             |                            | Summarizes workflow steps: image retrieval, upload, caption setting, posting                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - No configuration needed. This node will start the workflow.

2. **Create Set Node: Define Credentials**  
   - Node Type: Set  
   - Configure JSON output mode with static JSON object:  
     ```json
     {
       "credentials": {
         "identifier": "username.bsky.social",
         "password": "XXXX-YYYY-ZZZZ-XXXX"
       }
     }
     ```  
   - Replace with your actual Bluesky username and app password from [Bluesky App Passwords](https://bsky.app/settings/app-passwords).  
   - Connect Manual Trigger output to this node.

3. **Create HTTP Request Node: Create Bluesky Session**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`  
   - Body Content Type: JSON  
   - Body Parameters: Use expression to inject JSON from Define Credentials node:  
     ```
     {{$node["Define Credentials"].json["credentials"]}}
     ```  
   - Connect Define Credentials output to this node.

4. **Create Set Node: Set Caption**  
   - Node Type: Set  
   - Add a string field, e.g., `Caption Text`  
   - Static value: your desired post caption (max 300 characters).  
   - Connect Create Bluesky Session output to this node.

5. **Create Set Node: Set Images**  
   - Node Type: Set  
   - Configure raw JSON output with `photos` array of objects (1-4), each having a `url` string:  
     ```json
     {
       "photos": [
         { "url": "https://example.com/image1.jpg" },
         { "url": "https://example.com/image2.jpg" }
       ]
     }
     ```  
   - Replace URLs with your image sources.  
   - Connect Set Caption output to this node.

6. **Create Split Out Node**  
   - Node Type: SplitOut  
   - Field to split out: `photos`  
   - Connect Set Images output to Split Out node.

7. **Create HTTP Request Node: Download Images**  
   - Node Type: HTTP Request  
   - Method: GET (default)  
   - URL parameter: expression extracting current item URL:  
     ```
     {{$json["url"]}}
     ```  
   - Enable binary data download (default).  
   - Connect Split Out output to this node.

8. **Create HTTP Request Node: Post Image to Bluesky**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.repo.uploadBlob`  
   - Content Type: binary data  
   - Input Data Field Name: `data` (or binary property containing image)  
   - Add header parameter:  
     - Name: Authorization  
     - Value: `=Bearer {{$node["Create Bluesky Session"].json["accessJwt"]}}`  
   - Connect Download Images output to this node.

9. **Create Code Node: Prepare Image Embed Objects**  
   - Node Type: Code  
   - Language: JavaScript  
   - Code snippet:  
     ```javascript
     return $input.all().map(item => ({
       alt: "-",
       image: {
         ...item.json.blob
       }
     }));
     ```  
   - Connect Post Image to Bluesky output to this node.

10. **Create Aggregate Node**  
    - Node Type: Aggregate  
    - Configuration: Aggregate all item data into one array.  
    - Connect Code output to this node.

11. **Create HTTP Request Node: Post to Bluesky**  
    - Node Type: HTTP Request  
    - Method: POST  
    - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
    - Headers:  
      - Authorization: `=Bearer {{$node["Create Bluesky Session"].json["accessJwt"]}}`  
    - Body Content Type: JSON  
    - Body Parameters (use expressions):  
      ```json
      {
        "repo": "{{$node["Create Bluesky Session"].json["did"]}}",
        "collection": "app.bsky.feed.post",
        "record": {
          "$type": "app.bsky.feed.post",
          "text": "{{$node["Set Caption"].json["Caption Text"].trim()}}",
          "createdAt": "{{$now}}",
          "embed": {
            "$type": "app.bsky.embed.images",
            "images": {{$node["Aggregate"].json["data"]}}
          }
        }
      }
      ```  
    - Connect Aggregate output to this node.

12. **Optional: Add Sticky Notes**  
    - Add sticky notes to document credential setup, image upload explanation, and workflow summary for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Bluesky session creation requires an app password generated from your account settings.                        | https://bsky.app/settings/app-passwords                        |
| Image upload on Bluesky requires uploading images as blobs first, then embedding them in the post record.     | https://docs.bsky.app/docs/tutorials/creating-a-post#images-embeds |
| The 300-character limit includes caption, hashtags, and image alt texts. Exceeding it causes API errors.      | Workflow limitation note                                       |
| To create text-only posts, remove the `embed` section and related image upload nodes from the workflow.       | Workflow limitation note                                       |

---

This structured documentation provides a clear understanding of the workflow, node-by-node details, reproduction steps, and useful references for troubleshooting and enhancement.