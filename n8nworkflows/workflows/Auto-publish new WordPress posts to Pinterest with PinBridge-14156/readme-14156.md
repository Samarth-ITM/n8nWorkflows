Auto-publish new WordPress posts to Pinterest with PinBridge

https://n8nworkflows.xyz/workflows/auto-publish-new-wordpress-posts-to-pinterest-with-pinbridge-14156


# Auto-publish new WordPress posts to Pinterest with PinBridge

## 1. Workflow Overview

This workflow automatically takes newly published WordPress posts and submits them to Pinterest through PinBridge, while applying lightweight duplicate protection and basic content validation before publishing.

Its main use case is content distribution for sites that publish articles in WordPress and want to turn those posts into Pinterest pins with minimal manual work. The workflow is manually triggered in this version, but its logic is structured so it could later be adapted to scheduled execution.

### 1.1 Trigger and Existing Pin Reference Set
The workflow starts manually, loads existing pins from PinBridge, and aggregates their titles. This creates a simple reference list used to avoid submitting posts that appear to have already been published.

### 1.2 WordPress Post Retrieval and Early Filtering
It then fetches published WordPress posts from the WordPress REST API, including embedded featured media. Posts are allowed to continue only if they have usable featured media and their cleaned title is not already present in the existing PinBridge title list.

### 1.3 Payload Construction and Validation
For each post that passes the early filter, the workflow maps WordPress fields into a Pinterest-oriented payload. It then validates the minimum required fields before any external publishing actions occur.

### 1.4 Image Handling and Publishing
For valid posts, the workflow downloads the full-size featured image, uploads it as an asset to PinBridge, and submits a Pinterest publishing job using the mapped metadata.

### 1.5 Structured Output
The workflow returns a structured success object for submitted jobs, or a structured invalid object when required fields are missing after payload construction.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger and Existing Pin Reference Set

**Overview:**  
This block initializes the workflow and builds a lightweight duplicate-detection dataset from already existing PinBridge pins. The resulting aggregated title list is later used to skip WordPress posts that seem already published.

**Nodes Involved:**  
- Manual Trigger
- List pins
- Published Titles

#### Node: Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Entry point for manual execution.
- **Configuration choices:**  
  No custom parameters; this node simply starts the workflow when executed from the editor.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `List pins`
- **Version-specific requirements:**  
  Type version 1; standard behavior.
- **Edge cases or potential failure types:**  
  No runtime failure by itself; execution depends entirely on downstream nodes.
- **Sub-workflow reference:**  
  None.

#### Node: List pins
- **Type and technical role:** `n8n-nodes-pinbridge.pinBridge`  
  Retrieves existing pins from PinBridge.
- **Configuration choices:**  
  - Operation: `list`
  - Uses PinBridge API credentials
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Manual Trigger`
  - Output: `Published Titles`
- **Version-specific requirements:**  
  Type version 1; requires the PinBridge community node/package installed in the n8n environment.
- **Edge cases or potential failure types:**  
  - Missing or invalid PinBridge API key
  - API rate limiting
  - Empty result set if no pins exist
  - Unexpected schema changes from PinBridge
- **Sub-workflow reference:**  
  None.

#### Node: Published Titles
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Aggregates titles from existing pins into a single structure used for duplicate checking.
- **Configuration choices:**  
  - Aggregates the field `title`
- **Key expressions or variables used:**  
  Later referenced via:
  - `$('Published Titles').first().json.title`
- **Input and output connections:**  
  - Input: `List pins`
  - Output: `Get Latest WordPress Posts`
- **Version-specific requirements:**  
  Type version 1.
- **Edge cases or potential failure types:**  
  - If PinBridge returns no titles, the aggregate output may be empty or structurally different depending on execution mode
  - Duplicate protection relies on title text matching exactly after title cleanup; false negatives and false positives are possible
- **Sub-workflow reference:**  
  None.

---

### 2.2 WordPress Post Retrieval and Early Filtering

**Overview:**  
This block fetches published WordPress posts with embedded media and applies the first validation gate. Only posts with featured media and titles not already found in the aggregated PinBridge title list are allowed to continue.

**Nodes Involved:**  
- Get Latest WordPress Posts
- Skip Posts Invalid Posts

#### Node: Get Latest WordPress Posts
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the WordPress REST API to retrieve published posts.
- **Configuration choices:**  
  - URL: `https://www.mywordpresssite.com/wp-json/wp/v2/posts?status=publish&_embed=wp:featuredmedia`
  - Authentication: predefined credential type
  - Credential type: WordPress API
- **Key expressions or variables used:**  
  URL is expression-based but currently static except for the domain placeholder.
- **Input and output connections:**  
  - Input: `Published Titles`
  - Output: `Skip Posts Invalid Posts`
- **Version-specific requirements:**  
  Type version 4.2.
- **Edge cases or potential failure types:**  
  - Placeholder domain not replaced
  - Invalid WordPress credentials
  - WordPress REST API disabled or protected
  - `_embed=wp:featuredmedia` not returning expected structure
  - Large post lists causing longer execution times
- **Sub-workflow reference:**  
  None.

#### Node: Skip Posts Invalid Posts
- **Type and technical role:** `n8n-nodes-base.if`  
  Acts as the first filter for invalid or already-published posts.
- **Configuration choices:**  
  Main logical condition checks:
  - Embedded featured media exists
  - `source_url` exists on the embedded media
  - The cleaned WordPress title is not included in the aggregated published title list
- **Key expressions or variables used:**  
  Core expression:
  ```js
  {{ !!$json._embedded && !!$json._embedded['wp:featuredmedia'] && !!$json._embedded['wp:featuredmedia'][0] && !!$json._embedded['wp:featuredmedia'][0].source_url && !$('Published Titles').first().json.title.includes($json.title.rendered.replace(/<[^>]*>/g, '').trim()) }}
  ```
  Also includes a second empty string equality condition:
  - `leftValue: ""`
  - `rightValue: ""`
  This is always true and functionally redundant.
- **Input and output connections:**  
  - Input: `Get Latest WordPress Posts`
  - True output: `Build Pin Payload from Post`
  - False output: not connected
- **Version-specific requirements:**  
  Type version 2.2.
- **Edge cases or potential failure types:**  
  - If `Published Titles` output structure is unexpected, `.includes(...)` may fail
  - If `title.rendered` is missing, expression failure is possible
  - Duplicate detection is title-based only, so different titles pointing to the same URL are not caught
  - False branch is intentionally dropped, so skipped posts produce no explicit result at this stage
- **Sub-workflow reference:**  
  None.

---

### 2.3 Payload Construction and Validation

**Overview:**  
This block transforms a WordPress post into a normalized publishing payload and verifies that all minimum required values are present. It protects the publish path from incomplete or malformed content.

**Nodes Involved:**  
- Build Pin Payload from Post
- Validate Required Fields
- Build Invalid Result

#### Node: Build Pin Payload from Post
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps WordPress fields into a Pinterest-ready object.
- **Configuration choices:**  
  Assigns:
  - `post_id` from WordPress post ID
  - `title` from cleaned `title.rendered`
  - `description` from cleaned `excerpt.rendered`, truncated to 450 characters
  - `link_url` from post permalink
  - `image_url` from `_links['wp:featuredmedia'][0].href`
  - `alt_text` from linked media alt text or fallback title
- **Key expressions or variables used:**  
  - `{{ $json.id }}`
  - `{{ $json.title.rendered.replace(/<[^>]*>/g, '').trim() }}`
  - `{{ ($json.excerpt && $json.excerpt.rendered ? $json.excerpt.rendered.replace(/<[^>]*>/g, '').trim() : '').slice(0, 450) }}`
  - `{{ $json.link }}`
  - `{{ $json._links['wp:featuredmedia'][0].href }}`
  - `{{ $json._links['wp:featuredmedia'][0].alt_text || $json.title.rendered.replace(/<[^>]*>/g, '').trim() }}`
- **Input and output connections:**  
  - Input: `Skip Posts Invalid Posts`
  - Output: `Validate Required Fields`
- **Version-specific requirements:**  
  Type version 3.4.
- **Edge cases or potential failure types:**  
  - `image_url` is built from `_links` rather than `_embedded`; it becomes a media endpoint URL, not the actual image file URL
  - `alt_text` is read from `_links['wp:featuredmedia'][0].alt_text`, but `_links` usually does not contain alt text; fallback title likely handles most cases
  - Empty excerpts will produce an empty description and fail later validation
- **Sub-workflow reference:**  
  None.

#### Node: Validate Required Fields
- **Type and technical role:** `n8n-nodes-base.if`  
  Confirms that required payload fields exist before image download and publication.
- **Configuration choices:**  
  Requires all of the following to be truthy:
  - `title`
  - `description`
  - `link_url`
  - `image_url`
- **Key expressions or variables used:**  
  - `{{ !!$json.title }}`
  - `{{ !!$json.description }}`
  - `{{ !!$json.link_url }}`
  - `{{ !!$json.image_url }}`
- **Input and output connections:**  
  - Input: `Build Pin Payload from Post`
  - True output: `Download Featured Image`
  - False output: `Build Invalid Result`
- **Version-specific requirements:**  
  Type version 2.2.
- **Edge cases or potential failure types:**  
  - Truthiness check does not validate URL format or content quality
  - A technically non-empty but invalid `image_url` still passes
  - Long descriptions are already trimmed earlier, but title length is not constrained here
- **Sub-workflow reference:**  
  None.

#### Node: Build Invalid Result
- **Type and technical role:** `n8n-nodes-base.code`  
  Returns a structured invalid object for posts failing required-field validation.
- **Configuration choices:**  
  JavaScript code emits:
  - `status: 'invalid'`
  - fixed `error_message`
  - `post_id`
- **Key expressions or variables used:**  
  Code:
  ```js
  return [{ json: { status: 'invalid', error_message: 'Post is missing one or more required fields: title, description, link_url, image_url', post_id: $json.post_id || '' } }];
  ```
- **Input and output connections:**  
  - Input: `Validate Required Fields` false branch
  - Output: none
- **Version-specific requirements:**  
  Type version 2.
- **Edge cases or potential failure types:**  
  - Minimal result object; no title or link included for debugging
  - Posts filtered out earlier by `Skip Posts Invalid Posts` do not reach this node
- **Sub-workflow reference:**  
  None.

---

### 2.4 Image Handling and Publishing

**Overview:**  
This block performs the external publishing steps for valid posts: it downloads the full-size featured image from WordPress, uploads the file to PinBridge, and submits the Pinterest publication job.

**Nodes Involved:**  
- Download Featured Image
- Upload Image to PinBridge
- Publish to Pinterest

#### Node: Download Featured Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the full-size WordPress featured image as a binary file.
- **Configuration choices:**  
  - URL:
    `{{ $('Get Latest WordPress Posts').item.json._embedded['wp:featuredmedia'][0].media_details.sizes.full.source_url }}`
  - Response format: file
- **Key expressions or variables used:**  
  Uses item linkage back to `Get Latest WordPress Posts`.
- **Input and output connections:**  
  - Input: `Validate Required Fields` true branch
  - Output: `Upload Image to PinBridge`
- **Version-specific requirements:**  
  Type version 4.3.
- **Edge cases or potential failure types:**  
  - Missing `media_details.sizes.full.source_url`
  - Remote image URL inaccessible or blocked
  - Large image causing timeout or memory pressure
  - If item linking breaks due to changed execution pattern, expression resolution may fail
- **Sub-workflow reference:**  
  None.

#### Node: Upload Image to PinBridge
- **Type and technical role:** `n8n-nodes-pinbridge.pinBridge`  
  Uploads the downloaded image as an asset to PinBridge.
- **Configuration choices:**  
  - Resource: `assets`
  - Uses PinBridge API credentials
- **Key expressions or variables used:**  
  None explicitly in parameters; assumes the node consumes incoming binary data.
- **Input and output connections:**  
  - Input: `Download Featured Image`
  - Output: `Publish to Pinterest`
- **Version-specific requirements:**  
  Type version 1; depends on PinBridge node implementation and expected binary property naming.
- **Edge cases or potential failure types:**  
  - PinBridge credential issues
  - Binary data missing or in unexpected property
  - Unsupported image type/size
  - Upload success schema may differ from what the next node expects
- **Sub-workflow reference:**  
  None.

#### Node: Publish to Pinterest
- **Type and technical role:** `n8n-nodes-pinbridge.pinBridge`  
  Submits a Pinterest publish job through PinBridge.
- **Configuration choices:**  
  - Title from payload
  - Alt text from payload
  - Board ID fixed to `1124281563167437941`
  - Link URL with appended UTM parameters
  - Account ID fixed to `e3ebfe81-3854-4f56-a016-9e7fb4d7fcf7`
  - Description from payload
  - Dominant color from payload
- **Key expressions or variables used:**  
  - `{{ $('Build Pin Payload from Post').item.json.title }}`
  - `{{ $('Build Pin Payload from Post').item.json.alt_text }}`
  - `{{ $('Build Pin Payload from Post').item.json.link_url }}?utm_source=pinterest&utm_medium=social`
  - `{{ $('Build Pin Payload from Post').item.json.description }}`
  - `{{ $('Build Pin Payload from Post').item.json.dominant_color }}`
- **Input and output connections:**  
  - Input: `Upload Image to PinBridge`
  - Output: `Build Success Result`
- **Version-specific requirements:**  
  Type version 1; depends on installed PinBridge node capabilities.
- **Edge cases or potential failure types:**  
  - Hard-coded board/account IDs must be replaced with valid target values
  - `dominant_color` is referenced, but never set in the payload block; likely resolves undefined
  - Link URL always appends `?utm_source...`; if the original URL already has query parameters, the resulting URL may be malformed and should instead use `&`
  - If asset-upload output is required by the PinBridge node internally and not mapped, publication may fail depending on node behavior
- **Sub-workflow reference:**  
  None.

---

### 2.5 Structured Output

**Overview:**  
This block formats a normalized success result after a publish job is submitted. It is intended for downstream inspection, logging, or later extension into notifications and reporting.

**Nodes Involved:**  
- Build Success Result

#### Node: Build Success Result
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a structured success object.
- **Configuration choices:**  
  Assigns:
  - `post_id`
  - `title`
  - `link_url`
  - `job_id` from current node input
  - `status = submitted`
  - `submitted_at = $now`
  - `error_message = ''`
- **Key expressions or variables used:**  
  - `{{ $('Build Pin Payload from Post').item.json.post_id }}`
  - `{{ $('Build Pin Payload from Post').item.json.title }}`
  - `{{ $('Build Pin Payload from Post').item.json.link_url }}`
  - `{{ $json.id }}`
  - `{{ $now }}`
- **Input and output connections:**  
  - Input: `Publish to Pinterest`
  - Output: none
- **Version-specific requirements:**  
  Type version 3.4.
- **Edge cases or potential failure types:**  
  - Assumes PinBridge returns a job identifier in `$json.id`
  - Success here means “job submitted”, not necessarily “pin successfully published”
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Starts the workflow manually |  | List pins | # Publish WordPress posts to Pinterest with duplicate protection\n\nThis template uses **WordPress as the content source**, **n8n as the orchestration layer**, and **PinBridge as the publishing layer**.\n\n### How it works\n1. Load existing PinBridge pins\n2. Aggregate their titles\n3. Fetch published WordPress posts\n4. Skip posts that are **invalid**\n5. Build a Pinterest-ready payload\n6. Validate the required fields\n7. Download the featured image\n8. Upload the image to PinBridge\n9. Submit the Pinterest publish job\n10. Return a structured success or invalid result |
| List pins | PinBridge | Lists existing pins for duplicate protection | Manual Trigger | Published Titles | ## Build the reference set and fetch source posts\n\n`List pins` loads existing PinBridge pins.\n\n`Published Titles` aggregates existing titles into a single list that the workflow uses as a lightweight duplicate-protection layer.\n\n`Get Latest WordPress Posts` then fetches published posts with embedded featured media from the WordPress REST API. |
| Published Titles | Aggregate | Aggregates existing pin titles | List pins | Get Latest WordPress Posts | ## Build the reference set and fetch source posts\n\n`List pins` loads existing PinBridge pins.\n\n`Published Titles` aggregates existing titles into a single list that the workflow uses as a lightweight duplicate-protection layer.\n\n`Get Latest WordPress Posts` then fetches published posts with embedded featured media from the WordPress REST API. |
| Get Latest WordPress Posts | HTTP Request | Fetches published WordPress posts with embedded media | Published Titles | Skip Posts Invalid Posts | ## Build the reference set and fetch source posts\n\n`List pins` loads existing PinBridge pins.\n\n`Published Titles` aggregates existing titles into a single list that the workflow uses as a lightweight duplicate-protection layer.\n\n`Get Latest WordPress Posts` then fetches published posts with embedded featured media from the WordPress REST API. |
| Skip Posts Invalid Posts | If | Filters out posts without usable featured media or with duplicate titles | Get Latest WordPress Posts | Build Pin Payload from Post | ## Skip invalid posts early\n\n`Skip Posts Invalid Posts` is the first protection layer.\n\nA post continues only when:\n- featured media exists with a usable source URL\n- the cleaned WordPress title is **not** already published by this workflow\n\nPosts that fail this check are treated as **invalid** and skipped. |
| Build Pin Payload from Post | Set | Maps WordPress post fields into a Pinterest payload | Skip Posts Invalid Posts | Validate Required Fields | ## Build a publish-ready payload\n\n`Build Pin Payload from Post` maps WordPress data into:\n- `post_id` \| `title` \| `description` \| `link_url` \| `image_url` \| `alt_text`\n\n\n`Validate Required Fields` then checks that the minimum publish fields are present before the workflow touches any external publish steps. |
| Validate Required Fields | If | Confirms required payload fields exist | Build Pin Payload from Post | Download Featured Image; Build Invalid Result | ## Build a publish-ready payload\n\n`Build Pin Payload from Post` maps WordPress data into:\n- `post_id` \| `title` \| `description` \| `link_url` \| `image_url` \| `alt_text`\n\n\n`Validate Required Fields` then checks that the minimum publish fields are present before the workflow touches any external publish steps. |
| Download Featured Image | HTTP Request | Downloads the full-size featured image as binary | Validate Required Fields | Upload Image to PinBridge | ## Valid posts: download, upload, submit\n\nFor valid posts, the workflow:\n1. Downloads the full-size featured image\n2. Uploads the image asset to PinBridge\n3. Submits the Pinterest publish job\n4. Builds a structured success result |
| Upload Image to PinBridge | PinBridge | Uploads the image asset to PinBridge | Download Featured Image | Publish to Pinterest | ## Valid posts: download, upload, submit\n\nFor valid posts, the workflow:\n1. Downloads the full-size featured image\n2. Uploads the image asset to PinBridge\n3. Submits the Pinterest publish job\n4. Builds a structured success result |
| Publish to Pinterest | PinBridge | Submits the Pinterest publish job | Upload Image to PinBridge | Build Success Result | ## Valid posts: download, upload, submit\n\nFor valid posts, the workflow:\n1. Downloads the full-size featured image\n2. Uploads the image asset to PinBridge\n3. Submits the Pinterest publish job\n4. Builds a structured success result |
| Build Success Result | Set | Formats a structured success response | Publish to Pinterest |  | ## Valid posts: download, upload, submit\n\nFor valid posts, the workflow:\n1. Downloads the full-size featured image\n2. Uploads the image asset to PinBridge\n3. Submits the Pinterest publish job\n4. Builds a structured success result |
| Build Invalid Result | Code | Returns a structured invalid object when required fields are missing | Validate Required Fields |  | ## Invalid result branch\n\nIf required fields are missing after payload construction, `Build Invalid Result` returns a structured object instead of attempting submission.\n\nThis keeps incomplete content away from the publishing path and makes the workflow easier to review during testing. |
| Sticky Note - Overview | Sticky Note | Visual documentation block |  |  | # Publish WordPress posts to Pinterest with duplicate protection\n\nThis template uses **WordPress as the content source**, **n8n as the orchestration layer**, and **PinBridge as the publishing layer**.\n\n### How it works\n1. Load existing PinBridge pins\n2. Aggregate their titles\n3. Fetch published WordPress posts\n4. Skip posts that are **invalid**\n5. Build a Pinterest-ready payload\n6. Validate the required fields\n7. Download the featured image\n8. Upload the image to PinBridge\n9. Submit the Pinterest publish job\n10. Return a structured success or invalid result |
| Sticky Note - Setup | Sticky Note | Visual setup instructions |  |  | ## Setup checklist\n\nBefore running this workflow, replace placeholders and connect credentials:\n\n- Add a **WordPress** credential\n- Add a **PinBridge** credential\n- Replace `https://www.mywordpresssite.com`\n- Confirm the target **Pinterest account ID**\n- Confirm the target **board ID**\n- Confirm your WordPress REST API returns published posts with `_embed=wp:featuredmedia`\n\n\nThis workflow appends `?utm_source=pinterest&utm_medium=social` to the destination URL in the publish step for better analytics. |
| Sticky Note - Existing pins and source posts | Sticky Note | Visual documentation for reference set and source retrieval |  |  | ## Build the reference set and fetch source posts\n\n`List pins` loads existing PinBridge pins.\n\n`Published Titles` aggregates existing titles into a single list that the workflow uses as a lightweight duplicate-protection layer.\n\n`Get Latest WordPress Posts` then fetches published posts with embedded featured media from the WordPress REST API. |
| Sticky Note - Skip invalid posts | Sticky Note | Visual documentation for early filtering |  |  | ## Skip invalid posts early\n\n`Skip Posts Invalid Posts` is the first protection layer.\n\nA post continues only when:\n- featured media exists with a usable source URL\n- the cleaned WordPress title is **not** already published by this workflow\n\nPosts that fail this check are treated as **invalid** and skipped. |
| Sticky Note - Build and validate payload | Sticky Note | Visual documentation for payload mapping and validation |  |  | ## Build a publish-ready payload\n\n`Build Pin Payload from Post` maps WordPress data into:\n- `post_id` \| `title` \| `description` \| `link_url` \| `image_url` \| `alt_text`\n\n\n`Validate Required Fields` then checks that the minimum publish fields are present before the workflow touches any external publish steps. |
| Sticky Note - Publish path | Sticky Note | Visual documentation for publish steps |  |  | ## Valid posts: download, upload, submit\n\nFor valid posts, the workflow:\n1. Downloads the full-size featured image\n2. Uploads the image asset to PinBridge\n3. Submits the Pinterest publish job\n4. Builds a structured success result |
| Sticky Note - Invalid result branch | Sticky Note | Visual documentation for invalid result handling |  |  | ## Invalid result branch\n\nIf required fields are missing after payload construction, `Build Invalid Result` returns a structured object instead of attempting submission.\n\nThis keeps incomplete content away from the publishing path and makes the workflow easier to review during testing. |
| Sticky Note - Scope | Sticky Note | Visual note about workflow boundaries |  |  | ## What this template covers\n\nThis template is intentionally scoped to:\n- list existing pins\n- fetch WordPress posts\n- skip invalid or already-published posts\n- submit new publish jobs\n- return structured results\n\n\nA separate workflow should handle webhook callbacks, final publish confirmation, retries, notifications, and reporting. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Auto-publish new WordPress posts to Pinterest with PinBridge`.

2. **Add a Manual Trigger node**.
   - Node type: `Manual Trigger`
   - Keep default configuration.

3. **Create PinBridge credentials**.
   - Add a credential for the PinBridge node.
   - Use your production or test API key depending on environment.
   - Make sure the PinBridge community node is installed in your n8n instance if it is not already available.

4. **Create WordPress credentials**.
   - Add a `WordPress API` credential in n8n.
   - Use a WordPress user/application password or whichever authentication method your n8n setup supports.
   - Ensure the account can access the REST API endpoint for posts.

5. **Add a `PinBridge` node named `List pins`**.
   - Operation: `list`
   - Credential: your PinBridge credential
   - Connect `Manual Trigger -> List pins`.

6. **Add an `Aggregate` node named `Published Titles`**.
   - Configure it to aggregate the field `title`.
   - Connect `List pins -> Published Titles`.

7. **Add an `HTTP Request` node named `Get Latest WordPress Posts`**.
   - Method: `GET`
   - URL:  
     `https://YOUR_WORDPRESS_DOMAIN/wp-json/wp/v2/posts?status=publish&_embed=wp:featuredmedia`
   - Authentication: predefined credential type
   - Credential type: `WordPress API`
   - Credential: your WordPress credential
   - Connect `Published Titles -> Get Latest WordPress Posts`.

8. **Replace the placeholder domain**.
   - Change `https://www.mywordpresssite.com` to your actual WordPress site.
   - Verify that calling the URL returns published posts and that `_embedded['wp:featuredmedia']` is populated.

9. **Add an `If` node named `Skip Posts Invalid Posts`**.
   - Create a boolean condition using an expression.
   - Use this logic:
     - featured media exists
     - featured media has a source URL
     - current cleaned title is not already present in aggregated published titles
   - Equivalent expression:
     ```js
     {{ !!$json._embedded && !!$json._embedded['wp:featuredmedia'] && !!$json._embedded['wp:featuredmedia'][0] && !!$json._embedded['wp:featuredmedia'][0].source_url && !$('Published Titles').first().json.title.includes($json.title.rendered.replace(/<[^>]*>/g, '').trim()) }}
     ```
   - The original workflow also contains a second empty string equals condition. You may reproduce it exactly, but it is not necessary because it is redundant.
   - Connect `Get Latest WordPress Posts -> Skip Posts Invalid Posts`.

10. **Add a `Set` node named `Build Pin Payload from Post`**.
    - Create these fields:
      - `post_id` = `{{ $json.id }}`
      - `title` = `{{ $json.title.rendered.replace(/<[^>]*>/g, '').trim() }}`
      - `description` = `{{ ($json.excerpt && $json.excerpt.rendered ? $json.excerpt.rendered.replace(/<[^>]*>/g, '').trim() : '').slice(0, 450) }}`
      - `link_url` = `{{ $json.link }}`
      - `image_url` = `{{ $json._links['wp:featuredmedia'][0].href }}`
      - `alt_text` = `{{ $json._links['wp:featuredmedia'][0].alt_text || $json.title.rendered.replace(/<[^>]*>/g, '').trim() }}`
    - Connect the **true** output of `Skip Posts Invalid Posts -> Build Pin Payload from Post`.

11. **Add an `If` node named `Validate Required Fields`**.
    - Require all of these boolean expressions to equal `true`:
      - `{{ !!$json.title }}`
      - `{{ !!$json.description }}`
      - `{{ !!$json.link_url }}`
      - `{{ !!$json.image_url }}`
    - Connect `Build Pin Payload from Post -> Validate Required Fields`.

12. **Add an `HTTP Request` node named `Download Featured Image`**.
    - Method: `GET`
    - URL:
      ```js
      {{ $('Get Latest WordPress Posts').item.json._embedded['wp:featuredmedia'][0].media_details.sizes.full.source_url }}
      ```
    - Response format: `File`
    - Keep the downloaded file as binary output.
    - Connect the **true** output of `Validate Required Fields -> Download Featured Image`.

13. **Add a `PinBridge` node named `Upload Image to PinBridge`**.
    - Resource: `assets`
    - Credential: your PinBridge credential
    - Confirm how the node expects binary input; usually it consumes incoming file data from the previous node.
    - Connect `Download Featured Image -> Upload Image to PinBridge`.

14. **Add a `PinBridge` node named `Publish to Pinterest`**.
    - Credential: your PinBridge credential
    - Configure:
      - `title` = `{{ $('Build Pin Payload from Post').item.json.title }}`
      - `altText` = `{{ $('Build Pin Payload from Post').item.json.alt_text }}`
      - `description` = `{{ $('Build Pin Payload from Post').item.json.description }}`
      - `linkUrl` = `{{ $('Build Pin Payload from Post').item.json.link_url }}?utm_source=pinterest&utm_medium=social`
      - `boardId` = your Pinterest board ID
      - `accountId` = your Pinterest account ID
      - `dominantColor` = `{{ $('Build Pin Payload from Post').item.json.dominant_color }}`
    - Connect `Upload Image to PinBridge -> Publish to Pinterest`.

15. **Replace the hard-coded Pinterest identifiers**.
    - Replace:
      - `boardId = 1124281563167437941`
      - `accountId = e3ebfe81-3854-4f56-a016-9e7fb4d7fcf7`
    - Use valid values from your own Pinterest/PinBridge environment.

16. **Add a `Set` node named `Build Success Result`**.
    - Create these fields:
      - `post_id` = `{{ $('Build Pin Payload from Post').item.json.post_id }}`
      - `title` = `{{ $('Build Pin Payload from Post').item.json.title }}`
      - `link_url` = `{{ $('Build Pin Payload from Post').item.json.link_url }}`
      - `job_id` = `{{ $json.id }}`
      - `status` = `submitted`
      - `submitted_at` = `{{ $now }}`
      - `error_message` = empty string
    - Connect `Publish to Pinterest -> Build Success Result`.

17. **Add a `Code` node named `Build Invalid Result`**.
    - Paste this JavaScript:
      ```js
      return [{
        json: {
          status: 'invalid',
          error_message: 'Post is missing one or more required fields: title, description, link_url, image_url',
          post_id: $json.post_id || ''
        }
      }];
      ```
    - Connect the **false** output of `Validate Required Fields -> Build Invalid Result`.

18. **Optionally add the sticky notes** to preserve the same visual documentation structure:
    - Overview
    - Setup checklist
    - Existing pins and source posts
    - Skip invalid posts
    - Build and validate payload
    - Publish path
    - Invalid result branch
    - Scope

19. **Test the WordPress endpoint separately** before full execution.
    - Confirm post objects contain:
      - `title.rendered`
      - `excerpt.rendered`
      - `link`
      - `_embedded['wp:featuredmedia'][0].source_url`
      - `_embedded['wp:featuredmedia'][0].media_details.sizes.full.source_url`

20. **Run the workflow manually**.
    - Expected path for valid items:
      `Manual Trigger -> List pins -> Published Titles -> Get Latest WordPress Posts -> Skip Posts Invalid Posts -> Build Pin Payload from Post -> Validate Required Fields -> Download Featured Image -> Upload Image to PinBridge -> Publish to Pinterest -> Build Success Result`
    - Expected invalid path:
      `... -> Validate Required Fields -> Build Invalid Result`

21. **Recommended fixes before production use**.
    - Change the `linkUrl` append logic so it uses `&` when the post URL already contains query parameters.
    - Consider using the actual featured image URL directly in the payload instead of the media endpoint in `image_url`.
    - Consider extracting `alt_text` from `_embedded['wp:featuredmedia'][0].alt_text` rather than `_links`.
    - Add explicit handling for posts skipped by `Skip Posts Invalid Posts`, since they currently disappear without structured output.
    - If you need automated runs, replace `Manual Trigger` with a `Schedule Trigger`.

### Credential configuration summary
- **WordPress API credential**
  - Required by: `Get Latest WordPress Posts`
  - Must allow access to your site’s `/wp-json/wp/v2/posts` endpoint
- **PinBridge API credential**
  - Required by: `List pins`, `Upload Image to PinBridge`, `Publish to Pinterest`

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and is not itself documented as a callable sub-workflow. No `Execute Workflow` node is present.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This template uses WordPress as the content source, n8n as the orchestration layer, and PinBridge as the publishing layer. | Workflow overview note |
| Before running, replace placeholders and connect credentials: add WordPress credential, add PinBridge credential, replace the WordPress domain, confirm Pinterest account ID and board ID, and confirm `_embed=wp:featuredmedia` works in your REST response. | Setup guidance |
| The workflow appends `?utm_source=pinterest&utm_medium=social` to destination URLs for analytics. | Publish step behavior |
| The workflow scope is intentionally limited to listing pins, fetching posts, filtering invalid or already-published content, submitting new publish jobs, and returning structured results. | Scope note |
| A separate workflow should handle webhook callbacks, final publish confirmation, retries, notifications, and reporting. | Architecture recommendation |

**Important implementation note:** the provided workflow is operational in structure, but it contains a few design inconsistencies, especially around `image_url`, `alt_text`, `dominant_color`, and URL query-string handling. These should be reviewed before production deployment.