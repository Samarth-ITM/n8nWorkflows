Cross-Platform Content Publisher: Telegram to WordPress, Facebook, Twitter & LinkedIn

https://n8nworkflows.xyz/workflows/cross-platform-content-publisher--telegram-to-wordpress--facebook--twitter---linkedin-11127


# Cross-Platform Content Publisher: Telegram to WordPress, Facebook, Twitter & LinkedIn

### 1. Workflow Overview

This workflow, titled **"Cross-Platform Content Publisher: Telegram to WordPress, Facebook, Twitter & LinkedIn"**, automates the broadcasting of content received on Telegram to multiple social media and publishing platforms using n8n. Its primary purpose is to receive various types of Telegram messages—text, images, audio, video, documents, and voice messages—and publish or repost them appropriately on WordPress, Facebook, Twitter, LinkedIn, and back on Telegram if needed.

Logical blocks within the workflow are grouped as follows:

- **1.1 Input Reception & Message Type Detection:** Listens for incoming Telegram messages and determines the message type to route processing.
- **1.2 Media Download & Quality Detection:** Downloads media files from Telegram and assesses image quality to decide further processing.
- **1.3 WordPress Publishing:** Creates WordPress posts with appropriate media, including image editing and uploading.
- **1.4 Social Media Posting:** Generates and posts content to Facebook (image, video, document posts), Twitter, LinkedIn (profile and page posts).
- **1.5 Telegram Reposting:** Sends back media or text posts to Telegram channels or users as needed.
- **1.6 Merge & Branch Control:** Coordinates the flow of data from media downloads into the publishing nodes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Message Type Detection

**Overview:**  
This block triggers on incoming Telegram messages and uses a switch node to detect the type of message (text, audio, video, document, image, voice, file) to route processing accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- Detect Message Type (Switch)

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point, listens to Telegram updates (messages) via webhook.  
  - Config: Default parameters, listens for all message types.  
  - Inputs: None (Webhook trigger)  
  - Outputs: One output to Detect Message Type  
  - Edge cases: Webhook connectivity issues, Telegram API rate limits.

- **Detect Message Type**  
  - Type: Switch  
  - Role: Determines the type of Telegram message to route flow into appropriate branches.  
  - Config: Switch based on message properties to differentiate between document, video, audio, voice, images, files, and text.  
  - Inputs: From Telegram Trigger  
  - Outputs: Multiple outputs, each for a message type leading to download or sending nodes.  
  - Edge cases: Unexpected message types or missing message content might cause misrouting or errors.

---

#### 1.2 Media Download & Quality Detection

**Overview:**  
After identifying the message type, this block downloads the media content from Telegram and performs image quality detection to decide if further image editing is necessary.

**Nodes Involved:**  
- Download document  
- Telegram Send document  
- Telegram Send video  
- Telegram Sent Document Video  
- Download audio  
- Telegram Send audio  
- Download voice  
- Telegram Send voice  
- File image quality detect switch (Code node)  
- Image quality detect (Code node)  
- Download file  
- Telegram Send File  
- Download image  
- Merge (to combine downloaded media for downstream processing)

**Node Details:**  

- **Download document, audio, voice, file, image**  
  - Type: Telegram node (Telegram API file download)  
  - Role: Fetches media files from Telegram servers using file IDs.  
  - Config: Default, configured to pull file from the incoming message.  
  - Inputs: From Detect Message Type Switch outputs.  
  - Outputs: Media binary data forwarded downstream.  
  - Edge cases: Telegram file not found, download timeout, file size limits.

- **File image quality detect switch & Image quality detect**  
  - Type: Code (JavaScript)  
  - Role: Analyze downloaded images to detect quality, enabling conditional branching for image processing or direct use.  
  - Config: Custom JS code analyzes image metadata or pixel data.  
  - Inputs: From Download nodes  
  - Outputs: Branch to Download File or Download Image nodes.  
  - Edge cases: Corrupted images, unsupported formats, code execution errors.

- **Merge**  
  - Type: Merge  
  - Role: Combines different media streams into a unified output for WordPress and social media posting.  
  - Inputs: From Download file, Download image, Download audio nodes.  
  - Outputs: Forward to WordPress post creation and social media blocks.  
  - Edge cases: Input data mismatch or missing branches could cause incomplete merges.

- **Telegram Send nodes (document, video, audio, voice, file)**  
  - Type: Telegram node (send message/file)  
  - Role: Send media content back to Telegram channels or users.  
  - Inputs: From respective download or processing nodes.  
  - Edge cases: Message send failure due to permissions, network issues.

---

#### 1.3 WordPress Publishing

**Overview:**  
This block handles creating WordPress posts from the processed content, including media upload, image editing, and setting featured images.

**Nodes Involved:**  
- Create WordPress Post  
- Get a image  
- Edit Image  
- upload media to wp (HTTP Request)  
- upload image to meta data (HTTP Request)  
- set featured image (HTTP Request)

**Node Details:**  

- **Create WordPress Post**  
  - Type: WordPress node  
  - Role: Create a new post on a WordPress site using received Telegram content.  
  - Config: Post title, content, status are dynamically set (likely from Telegram message text or processed content).  
  - Inputs: From Merge node (media & text combined).  
  - Outputs: Post creation confirmation, post ID for further media association.  
  - Edge cases: Authentication failure, API limits, invalid post data.

- **Get a image**  
  - Type: Telegram node (download image)  
  - Role: Downloads images associated with WordPress posts.  
  - Inputs: From Create WordPress Post node.  
  - Outputs: Image binary data for editing.  
  - Edge cases: Missing images or download failure.

- **Edit Image**  
  - Type: Edit Image node  
  - Role: Performs image manipulation (resize, crop, optimize) prior to upload.  
  - Inputs: From Get a image  
  - Outputs: Edited image binary data.  
  - Edge cases: Unsupported image formats, image corruption, processing errors.

- **upload media to wp**  
  - Type: HTTP Request  
  - Role: Upload edited media to WordPress media library via REST API.  
  - Inputs: Edited image data.  
  - Outputs: Media upload confirmation and media ID.  
  - Edge cases: API authentication issues, upload size limits.

- **upload image to meta data**  
  - Type: HTTP Request  
  - Role: Attach uploaded media to post metadata (custom fields or featured image reference).  
  - Inputs: From media upload response.  
  - Outputs: Metadata update confirmation.  
  - Edge cases: API errors, incorrect post ID.

- **set featured image**  
  - Type: HTTP Request  
  - Role: Sets the uploaded image as the featured image of the WordPress post.  
  - Inputs: From metadata update.  
  - Outputs: Success confirmation.  
  - Edge cases: Permission errors, invalid media or post IDs.

---

#### 1.4 Social Media Posting

**Overview:**  
This block generates and posts content on Facebook, Twitter, and LinkedIn profiles and pages using the content received from Telegram.

**Nodes Involved:**  
- Facebook Image post  
- Facebook Video post  
- Facebook Document Video post  
- Facebook text post  
- Create Tweet text  
- Create profile text post (LinkedIn)  
- Create page text post (LinkedIn)  
- Create profile image post (LinkedIn)  
- Create page image post (LinkedIn)  
- Get facebook post image  
- Get facebook post video  
- Get facebook post document  
- Get linkedIn profile post image  
- Get linkedIn page post image

**Node Details:**  

- **Facebook posts (Image, Video, Document Video, Text)**  
  - Type: Facebook Graph API node  
  - Role: Posts respective content types to Facebook Page or Profile using Graph API.  
  - Config: Message, media, links dynamically mapped from Telegram content or WordPress posts.  
  - Inputs: From Get facebook post nodes or directly from Merge.  
  - Outputs: Post ID confirmation.  
  - Edge cases: Token expiration, permission issues, media upload failures.

- **Create Tweet text**  
  - Type: Twitter node  
  - Role: Posts text tweets based on Telegram text content.  
  - Config: Tweet text dynamically set.  
  - Inputs: From do nothing node (text posts branch).  
  - Outputs: Tweet ID confirmation.  
  - Edge cases: Twitter API limits, auth failures.

- **Create LinkedIn posts (profile/page, text/image)**  
  - Type: LinkedIn node  
  - Role: Posts text or images to LinkedIn profile or pages.  
  - Config: Message and media dynamically assigned from Telegram content.  
  - Inputs: From Get linkedIn post image nodes or do nothing node for text.  
  - Outputs: Post confirmation.  
  - Edge cases: API rate limits, permission errors.

- **Get facebook post image/video/document & LinkedIn post image nodes**  
  - Type: Telegram nodes downloading media for social media posting.  
  - Role: Fetch media files from Telegram to post on social platforms.  
  - Edge cases: Media missing or download failure.

---

#### 1.5 Telegram Reposting

**Overview:**  
Sends media or text posts back to Telegram channels or users, enabling content redistribution or confirmation.

**Nodes Involved:**  
- Telegram Send audio  
- Telegram Send voice  
- Telegram Send video  
- Telegram Sent Document Video  
- Telegram Send File  
- Telegram Send document  
- Telegram Image post  
- send text post

**Node Details:**  

- **Telegram Send nodes**  
  - Type: Telegram node (message send)  
  - Role: Posts media or text back to Telegram.  
  - Config: Target chat/channel ID, media payload dynamically set.  
  - Inputs: From respective download or processing nodes.  
  - Outputs: Confirmation of send.  
  - Edge cases: Chat permissions, rate limiting, invalid media.

---

#### 1.6 Merge & Branch Control

**Overview:**  
Coordinates the integration of different media streams and text content into a unified flow for subsequent posting.

**Nodes Involved:**  
- Merge  
- do nothing (No Operation)

**Node Details:**  

- **Merge**  
  - Type: Merge node  
  - Role: Combines multiple inputs (file, image, audio downloads) into one stream for posting.  
  - Inputs: From Download file, Download image, Download audio nodes  
  - Outputs: To Create WordPress Post and social media nodes.  
  - Edge cases: Missing inputs or data inconsistency.

- **do nothing**  
  - Type: No Operation node  
  - Role: Passes through text messages directly to social media posting nodes without transformation.  
  - Inputs: From Detect Message Type for text messages.  
  - Outputs: To social media text post nodes.  
  - Edge cases: None (pass-through).

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                                  | Input Node(s)                      | Output Node(s)                                   | Sticky Note                                             |
|------------------------------|-------------------------|-------------------------------------------------|----------------------------------|-------------------------------------------------|---------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger        | Entry point for Telegram messages                | None                             | Detect Message Type                              |                                                         |
| Detect Message Type          | Switch                  | Routes flow based on Telegram message type      | Telegram Trigger                 | Download document, Telegram Send video, ...      |                                                         |
| Download document            | Telegram                | Downloads document files from Telegram           | Detect Message Type              | Telegram Send document                           |                                                         |
| Telegram Send document       | Telegram                | Sends document back to Telegram                   | Download document                | None                                            |                                                         |
| Telegram Send video          | Telegram                | Sends video back to Telegram                       | Detect Message Type              | Get facebook post video                          |                                                         |
| Telegram Sent Document Video | Telegram                | Sends document video back to Telegram             | Detect Message Type              | Get facebook post document                       |                                                         |
| Download audio              | Telegram                | Downloads audio files from Telegram                | Detect Message Type              | Telegram Send audio                              |                                                         |
| Telegram Send audio         | Telegram                | Sends audio back to Telegram                        | Download audio                   | None                                            |                                                         |
| Download voice              | Telegram                | Downloads voice message from Telegram              | Detect Message Type              | Telegram Send voice                              |                                                         |
| Telegram Send voice         | Telegram                | Sends voice message back to Telegram                | Download voice                   | None                                            |                                                         |
| File image quality detect switch | Code               | Decides if file is image and needs quality check  | Detect Message Type              | Download file                                    |                                                         |
| Download file               | Telegram                | Downloads generic files from Telegram              | File image quality detect switch | Telegram Send File                               |                                                         |
| Telegram Send File          | Telegram                | Sends generic file back to Telegram                 | Download File                   | None                                            |                                                         |
| Image quality detect        | Code                    | Detects image quality to decide processing path   | Detect Message Type              | Download image                                   |                                                         |
| Download image              | Telegram                | Downloads image files from Telegram                 | Image quality detect             | Merge                                            |                                                         |
| Merge                      | Merge                    | Combines media downloads into one stream          | Download file, image, audio     | Create WordPress Post, Get facebook post image, LinkedIn posts, Telegram Image post |                                                         |
| Create WordPress Post       | WordPress                | Creates WordPress post with Telegram content      | Merge                           | Get a image                                      |                                                         |
| Get a image                 | Telegram                | Downloads image for WordPress post                 | Create WordPress Post            | Edit Image                                       |                                                         |
| Edit Image                 | Edit Image               | Edits images prior to upload                        | Get a image                     | upload media to wp                               |                                                         |
| upload media to wp          | HTTP Request             | Uploads media to WordPress media library           | Edit Image                     | upload image to meta data                        |                                                         |
| upload image to meta data   | HTTP Request             | Updates WordPress post metadata with media info   | upload media to wp              | set featured image                               |                                                         |
| set featured image          | HTTP Request             | Sets featured image for WordPress post             | upload image to meta data       | None                                            |                                                         |
| Get facebook post image     | Telegram                | Downloads image for Facebook post                   | Create WordPress Post            | Facebook Image post                              |                                                         |
| Facebook Image post         | Facebook Graph API       | Posts image content on Facebook                      | Get facebook post image         | None                                            |                                                         |
| Get facebook post video     | Telegram                | Downloads video for Facebook post                   | Telegram Send video             | Facebook Video post                              |                                                         |
| Facebook Video post         | Facebook Graph API       | Posts video content on Facebook                      | Get facebook post video         | None                                            |                                                         |
| Get facebook post document  | Telegram                | Downloads document for Facebook post                | Telegram Sent Document Video    | Facebook Document Video post                     |                                                         |
| Facebook Document Video post| Facebook Graph API       | Posts document or video content on Facebook         | Get facebook post document      | None                                            |                                                         |
| Create Tweet text           | Twitter                 | Posts text tweets                                  | do nothing                     | None                                            |                                                         |
| Create profile text post    | LinkedIn                | Posts text to LinkedIn profile                      | do nothing                     | None                                            |                                                         |
| Create page text post       | LinkedIn                | Posts text to LinkedIn page                          | do nothing                     | None                                            |                                                         |
| Facebook text post          | Facebook Graph API       | Posts text content on Facebook                       | do nothing                     | None                                            |                                                         |
| do nothing                 | NoOp                    | Passes text messages to social media posting nodes | Detect Message Type (text branch) | send text post, Create Tweet text, LinkedIn posts, Facebook text post |                                                         |
| Telegram Image post         | Telegram                | Sends image posts back to Telegram                   | Merge                           | None                                            |                                                         |
| Get linkedIn profile post image | Telegram            | Downloads image for LinkedIn profile post           | Create WordPress Post            | Create profile image post                        |                                                         |
| Create profile image post   | LinkedIn                | Posts image to LinkedIn profile                      | Get linkedIn profile post image | None                                            |                                                         |
| Get linkedIn page post image | Telegram                | Downloads image for LinkedIn page post              | Create WordPress Post            | Create page image post                           |                                                         |
| Create page image post      | LinkedIn                | Posts image to LinkedIn page                          | Get linkedIn page post image    | None                                            |                                                         |
| send text post             | Telegram                | Sends text posts back to Telegram                    | do nothing                     | None                                            |                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Set up Telegram credentials with bot token.  
   - Leave default parameters to listen to all message types.  
   - Connect output to Detect Message Type node.

2. **Create Detect Message Type Node (Switch)**  
   - Type: Switch  
   - Configure rules to detect message type by checking Telegram message properties:  
     - Document  
     - Video  
     - Document Video  
     - Audio  
     - Voice  
     - File (generic)  
     - Image  
     - Text (default)  
   - Connect outputs accordingly:
     - Document → Download document node  
     - Video → Telegram Send video node  
     - Document Video → Telegram Sent Document Video node  
     - Audio → Download audio node  
     - Voice → Download voice node  
     - File → File image quality detect switch (Code) node  
     - Image → Image quality detect (Code) node  
     - Text → do nothing node

3. **Download Media Nodes**  
   - For each media type (document, audio, voice, image, file), create Telegram nodes configured with credentials to download the corresponding file using the file ID from the message.  
   - Connect each download node to its respective Telegram Send node or further processing nodes.

4. **File Image Quality Detect Switch (Code Node)**  
   - Paste JavaScript code to check if the file is an image and decide to download file or image.  
   - Connect outputs to Download file or Download image nodes.

5. **Image Quality Detect (Code Node)**  
   - Insert JavaScript code to analyze image quality and decide if image editing is required.  
   - Connect output to Download image node.

6. **Download image Node**  
   - Telegram node to download image binary.  
   - Connect to Merge node.

7. **Merge Node**  
   - Type: Merge  
   - Mode: Wait for all inputs or pass-through depending on use case.  
   - Inputs: Download file, Download image, Download audio nodes.  
   - Output: Connect to Create WordPress Post and social media posting nodes.

8. **Create WordPress Post Node**  
   - Type: WordPress node  
   - Configure WordPress credentials (URL, username, password or application password).  
   - Map post title and content from Telegram message or processed data.  
   - Connect output to Get a image node.

9. **Get a image Node**  
   - Telegram node to download images for WordPress post.  
   - Connect output to Edit Image node.

10. **Edit Image Node**  
    - Configure image transformations (resize, crop, quality).  
    - Connect output to upload media to wp node.

11. **upload media to wp Node**  
    - HTTP Request node  
    - Set URL to WordPress REST API media endpoint (e.g., `/wp-json/wp/v2/media`)  
    - Use POST method, attach image binary data.  
    - Configure authentication using WordPress credentials.  
    - Connect output to upload image to meta data node.

12. **upload image to meta data Node**  
    - HTTP Request node  
    - Set URL to update post metadata with media ID.  
    - Use POST or PUT method as required.  
    - Connect output to set featured image node.

13. **set featured image Node**  
    - HTTP Request node  
    - Set URL to WordPress REST API to set featured image for post.  
    - Use POST or PUT method.  
    - Configure authentication.  
    - Final node in WordPress publishing branch.

14. **Social Media Posting Nodes**  
    - Create Facebook Graph API nodes for image, video, document, and text posts.  
    - Create Twitter node configured with Twitter app credentials for text posts.  
    - Create LinkedIn nodes for profile and page posts (text and image).  
    - Connect respective media download nodes (Get facebook post image/video/document, Get linkedIn profile/page post image) to these social media post nodes.

15. **Telegram Reposting Nodes**  
    - Create Telegram Send nodes for audio, voice, video, document, file, image, and text.  
    - Connect from respective media download or do nothing nodes.

16. **do nothing Node**  
    - Create No Operation node to pass text messages directly to social media text post nodes.  
    - Connect from Detect Message Type's text output to do nothing.  
    - Connect do nothing node outputs to Create Tweet text, Create profile text post, Create page text post, Facebook text post, and send text post nodes.

17. **Credentials Setup**  
    - Telegram: Bot token for Telegram Trigger and Telegram nodes.  
    - WordPress: WordPress site URL and authentication (application password or OAuth).  
    - Facebook: Facebook app token with appropriate permissions for Graph API calls.  
    - Twitter: Twitter API key, secret, access token, and access token secret.  
    - LinkedIn: OAuth credentials for posting to profile and pages.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                               |
|------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow is branded as "TeleCast AI" implying automated broadcasting of Telegram content via AI integrations. | Workflow title and description.                               |
| The workflow supports multiple media types and social platforms, ensuring wide content distribution.             | Cross-platform content publishing use case.                   |
| The workflow uses advanced media handling including image quality detection and editing before upload to WordPress. | Media processing best practices.                              |
| Facebook API nodes require valid app permissions and tokens, ensure Facebook page management access.             | Facebook Graph API documentation.                             |
| Twitter and LinkedIn nodes require proper OAuth credentials and API permissions.                                  | Twitter API and LinkedIn API documentation.                   |
| Telegram file downloads rely on Telegram bot API access and file IDs from incoming messages.                     | Telegram Bot API documentation.                               |
| Edge cases to monitor include API rate limits, media download failures, and authentication token expirations.    | General API integration considerations.                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.