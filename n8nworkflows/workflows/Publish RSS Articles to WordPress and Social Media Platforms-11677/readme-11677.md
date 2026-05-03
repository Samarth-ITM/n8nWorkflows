Publish RSS Articles to WordPress and Social Media Platforms

https://n8nworkflows.xyz/workflows/publish-rss-articles-to-wordpress-and-social-media-platforms-11677


# Publish RSS Articles to WordPress and Social Media Platforms

### 1. Workflow Overview

This workflow automates the process of publishing articles from an RSS feed to a WordPress site and multiple social media platforms. Its primary use case is to streamline content distribution by extracting new RSS items, enhancing them with AI processing, publishing to WordPress (including media handling), and then broadcasting the posts across social channels such as Facebook, Telegram, LinkedIn, Discord, and WhatsApp notifications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Scheduling:** Periodic triggering and RSS feed reading.
- **1.2 AI Content Processing:** Using an AI agent to process or analyze RSS content.
- **1.3 Content Validation:** Conditional checks on the processed content.
- **1.4 WordPress Publishing:** Handling media downloads, image editing, media uploads, metadata updates, and post creation.
- **1.5 Social Media Broadcasting:** Posting the finalized content to various social networks.
- **1.6 Notifications:** Sending alerts about the publishing status through Telegram, Gmail, Discord, and WhatsApp.
- **1.7 No Operation Handling:** A placeholder node to maintain workflow structure when no action is needed.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scheduling

**Overview:**  
This block triggers the workflow on a schedule and reads the RSS feed from a specified URL, setting the stage for content processing.

**Nodes Involved:**  
- Schedule Trigger  
- Add RSS link  
- RSS Read Domani

**Node Details:**  

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow at defined intervals (time-based trigger).  
  - *Configuration:* Likely set to a periodic schedule (e.g., hourly/daily).  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connects to "Add RSS link".  
  - *Edge Cases:* Workflow may not trigger if scheduling is misconfigured; time zone considerations.  

- **Add RSS link**  
  - *Type:* Set  
  - *Role:* Sets or updates the RSS feed URL or related parameters before reading.  
  - *Configuration:* Contains the RSS feed URL and any necessary parameters for the RSS reader.  
  - *Inputs:* From Schedule Trigger.  
  - *Outputs:* Connects to "RSS Read Domani".  
  - *Edge Cases:* Incorrect URL or parameters will lead to empty or failed RSS fetch.

- **RSS Read Domani**  
  - *Type:* RSS Feed Read  
  - *Role:* Reads articles/items from the given RSS feed.  
  - *Configuration:* Uses the URL set in the previous node.  
  - *Inputs:* From Add RSS link.  
  - *Outputs:* Connects to "AI Agent".  
  - *Edge Cases:* RSS feed may be unreachable, malformed, or empty; network timeouts.

---

#### 1.2 AI Content Processing

**Overview:**  
Processes the RSS feed items using an AI agent, potentially for content enrichment, summarization, or validation.

**Nodes Involved:**  
- AI Agent  
- OpenAI

**Node Details:**  

- **AI Agent**  
  - *Type:* n8n Langchain Agent  
  - *Role:* Runs AI-based content processing or analysis on RSS items.  
  - *Configuration:* Connected to an AI language model, likely configured with prompts or chains to process article data.  
  - *Inputs:* From RSS Read Domani.  
  - *Outputs:* Connects to "If (check link)".  
  - *Version:* v2.2  
  - *Edge Cases:* AI API rate limits, malformed input data, or unexpected AI output structure.

- **OpenAI**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides the language model backend for AI Agent.  
  - *Configuration:* OpenAI credentials and parameters set for chat completions.  
  - *Inputs:* Connected internally to AI Agent as language model.  
  - *Outputs:* To AI Agent.  
  - *Edge Cases:* API key expiration, quota limits, or network issues.

---

#### 1.3 Content Validation

**Overview:**  
Decides if the AI-processed content contains valid or usable data to proceed with publishing.

**Nodes Involved:**  
- If (check link)

**Node Details:**  

- **If (check link)**  
  - *Type:* If  
  - *Role:* Conditional node that evaluates if the processed content meets criteria (e.g., valid URL or post data).  
  - *Configuration:* Uses expressions to check variables from AI Agent output, such as presence of a valid link.  
  - *Inputs:* From AI Agent.  
  - *Outputs:* Main true branch to "link to binary" node; false branch discards or halts.  
  - *Edge Cases:* Misconfigured expression can block valid posts or allow invalid data.

---

#### 1.4 WordPress Publishing

**Overview:**  
Manages downloading images, editing them, uploading media to WordPress, setting featured images, and creating the WordPress post with all enriched content.

**Nodes Involved:**  
- link to binary  
- Create WordPress Post  
- download image  
- Edit Image  
- upload media to wp  
- upload image to meta data  
- set featured image

**Node Details:**  

- **link to binary**  
  - *Type:* HTTP Request  
  - *Role:* Converts a link (likely to media or content) to binary data for further processing.  
  - *Inputs:* From If (check link) on true branch.  
  - *Outputs:* Fans out to Create WordPress Post and multiple social post nodes.  
  - *Edge Cases:* Invalid URLs, network failures.

- **Create WordPress Post**  
  - *Type:* WordPress  
  - *Role:* Creates a new post on the WordPress site with content.  
  - *Inputs:* From link to binary.  
  - *Outputs:* Connects to "download image" for media handling.  
  - *Edge Cases:* Authentication failures, invalid post data.

- **download image**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the post’s featured image or media.  
  - *Inputs:* From Create WordPress Post.  
  - *Outputs:* Connects to Edit Image.  
  - *Edge Cases:* Broken image links, large file sizes causing timeouts.

- **Edit Image**  
  - *Type:* Edit Image  
  - *Role:* Performs image modifications (resizing, formatting) before upload.  
  - *Inputs:* From download image.  
  - *Outputs:* Connects to upload media to wp.  
  - *Edge Cases:* Unsupported image formats, editing failures.

- **upload media to wp**  
  - *Type:* HTTP Request  
  - *Role:* Uploads the edited image binary to WordPress media library.  
  - *Inputs:* From Edit Image.  
  - *Outputs:* Connects to upload image to meta data.  
  - *Edge Cases:* Authentication errors, upload size limits.

- **upload image to meta data**  
  - *Type:* HTTP Request  
  - *Role:* Associates uploaded media with WordPress post metadata.  
  - *Inputs:* From upload media to wp.  
  - *Outputs:* Connects to set featured image.  
  - *Edge Cases:* API errors or data mismatches.

- **set featured image**  
  - *Type:* HTTP Request  
  - *Role:* Sets the uploaded media as the featured image of the WordPress post.  
  - *Inputs:* From upload image to meta data.  
  - *Outputs:* None (end of WordPress publishing chain).  
  - *Edge Cases:* API permission issues.

---

#### 1.5 Social Media Broadcasting

**Overview:**  
Posts the newly created content to multiple social platforms to maximize reach.

**Nodes Involved:**  
- Post on Facebook page  
- Post on  telegram Channel  
- Post on LinkkedIN Profile  
- Post on LinkedN page  
- Post on Discord Channel

**Node Details:**  

- **Post on Facebook page**  
  - *Type:* Facebook Graph API  
  - *Role:* Publishes the post content or link to a Facebook page.  
  - *Inputs:* From link to binary (WordPress content).  
  - *Outputs:* None (terminal node).  
  - *Edge Cases:* Token expiry, permission errors.

- **Post on  telegram Channel**  
  - *Type:* Telegram  
  - *Role:* Sends post to a Telegram channel.  
  - *Inputs:* From link to binary.  
  - *Outputs:* None.  
  - *Edge Cases:* Invalid channel ID or bot permissions.

- **Post on LinkkedIN Profile**  
  - *Type:* LinkedIn  
  - *Role:* Posts content on LinkedIn user profile.  
  - *Inputs:* From link to binary.  
  - *Outputs:* None.  
  - *Edge Cases:* API rate limits or invalid credentials.

- **Post on LinkedN page**  
  - *Type:* LinkedIn  
  - *Role:* Posts content on LinkedIn company page.  
  - *Inputs:* From link to binary.  
  - *Outputs:* None.  
  - *Edge Cases:* Page admin rights or API issues.

- **Post on Discord Channel**  
  - *Type:* Discord  
  - *Role:* Posts content to a Discord channel via webhook.  
  - *Inputs:* From link to binary.  
  - *Outputs:* None.  
  - *Edge Cases:* Invalid webhook URL or rate limits.

---

#### 1.6 Notifications

**Overview:**  
Sends notifications about workflow execution or posting results via multiple channels.

**Nodes Involved:**  
- Rapiwa (sent whatsapp Notification)  
- Send a Notification (Telegram)  
- Send a Notification1 (Gmail)  
- Sent a Notification (Discord)

**Node Details:**  

- **Rapiwa (sent whatsapp Notification)**  
  - *Type:* Rapiwa (WhatsApp)  
  - *Role:* Sends WhatsApp notifications on workflow events.  
  - *Inputs:* From Do nothing node (see below).  
  - *Outputs:* None.  
  - *Edge Cases:* API limits or authentication issues.

- **Send a Notification (Telegram)**  
  - *Type:* Telegram  
  - *Role:* Sends Telegram notification messages.  
  - *Inputs:* From Do nothing.  
  - *Outputs:* None.  
  - *Edge Cases:* Bot permissions or invalid chat IDs.

- **Send a Notification1 (Gmail)**  
  - *Type:* Gmail  
  - *Role:* Sends email notifications.  
  - *Inputs:* From Do nothing.  
  - *Outputs:* None.  
  - *Edge Cases:* OAuth credential expiry, quota limits.

- **Sent a Notification (Discord)**  
  - *Type:* Discord  
  - *Role:* Sends notifications to Discord channels.  
  - *Inputs:* From Do nothing.  
  - *Outputs:* None.  
  - *Edge Cases:* Webhook invalidation or rate limits.

---

#### 1.7 No Operation Handling

**Overview:**  
Acts as a placeholder or endpoint for conditional branches where no further action is required but notifications are sent.

**Nodes Involved:**  
- Do nothing

**Node Details:**  

- **Do nothing**  
  - *Type:* No Operation  
  - *Role:* Passes data without changes to trigger notification nodes.  
  - *Inputs:* From link to binary node (fan-out) branch.  
  - *Outputs:* Connects to all notification nodes.  
  - *Edge Cases:* None (minimal risk).

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                              | Input Node(s)           | Output Node(s)                                      | Sticky Note |
|-----------------------------|----------------------------|----------------------------------------------|-------------------------|-----------------------------------------------------|-------------|
| Schedule Trigger            | Schedule Trigger           | Triggers workflow periodically                | None                    | Add RSS link                                        |             |
| Add RSS link               | Set                        | Sets RSS feed URL or parameters                | Schedule Trigger        | RSS Read Domani                                     |             |
| RSS Read Domani            | RSS Feed Read              | Reads articles from RSS feed                    | Add RSS link            | AI Agent                                           |             |
| AI Agent                   | Langchain Agent            | Processes RSS items with AI                     | RSS Read Domani         | If (check link)                                    |             |
| OpenAI                     | Langchain OpenAI Chat Model| Provides AI language model to AI Agent          | AI Agent (internal)     | AI Agent                                           |             |
| If (check link)            | If                         | Validates AI output for content readiness       | AI Agent                | link to binary (true), none (false)                |             |
| link to binary             | HTTP Request               | Converts links to binary for media processing   | If (check link)         | Create WordPress Post, social posts, Do nothing    |             |
| Create WordPress Post      | WordPress                  | Creates WordPress post                           | link to binary          | download image                                     |             |
| download image             | HTTP Request               | Downloads post images                            | Create WordPress Post   | Edit Image                                         |             |
| Edit Image                 | Edit Image                 | Edits images (resize, format)                    | download image          | upload media to wp                                 |             |
| upload media to wp         | HTTP Request               | Uploads media to WordPress media library         | Edit Image              | upload image to meta data                          |             |
| upload image to meta data  | HTTP Request               | Updates WordPress post metadata with media info | upload media to wp      | set featured image                                 |             |
| set featured image         | HTTP Request               | Sets featured image for WordPress post           | upload image to meta data| None                                              |             |
| Post on Facebook page      | Facebook Graph API         | Posts content to Facebook page                   | link to binary          | None                                              |             |
| Post on  telegram Channel  | Telegram                   | Posts content to Telegram channel                 | link to binary          | None                                              |             |
| Post on LinkkedIN Profile  | LinkedIn                   | Posts content to LinkedIn user profile            | link to binary          | None                                              |             |
| Post on LinkedN page       | LinkedIn                   | Posts content to LinkedIn company page            | link to binary          | None                                              |             |
| Post on Discord Channel    | Discord                    | Posts content to Discord channel                   | link to binary          | None                                              |             |
| Do nothing                 | No Operation               | Placeholder to trigger notifications             | link to binary          | Rapiwa, Send a Notification, Send a Notification1, Sent a Notification |             |
| Rapiwa (sent whatsapp Notification) | Rapiwa (WhatsApp)  | Sends WhatsApp notifications                      | Do nothing              | None                                              |             |
| Send a Notification        | Telegram                   | Sends Telegram notifications                      | Do nothing              | None                                              |             |
| Send a Notification1       | Gmail                      | Sends email notifications                         | Do nothing              | None                                              |             |
| Sent a Notification        | Discord                    | Sends Discord notifications                        | Do nothing              | None                                              |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set it to run on your desired interval (e.g., hourly or daily) to check for new RSS articles.

2. **Add a Set node ("Add RSS link")**  
   - Configure a field to hold the RSS feed URL you want to monitor.

3. **Add an RSS Feed Read node ("RSS Read Domani")**  
   - Use the URL from the Set node to fetch the RSS feed items.

4. **Add an AI Agent node ("AI Agent")**  
   - Configure with your Langchain agent setup.  
   - Connect it to the OpenAI node as the language model backend.

5. **Add an OpenAI Chat Model node ("OpenAI")**  
   - Configure OpenAI credentials (API key) with suitable chat completion parameters (temperature, model).  
   - Connect it as the language model for "AI Agent".

6. **Add an If node ("If (check link)")**  
   - Configure to evaluate if AI Agent output contains a valid post link or required content.  
   - Use expression checks on AI output fields.

7. **Add an HTTP Request node ("link to binary")**  
   - Configure to fetch media or content URLs as binary data for WordPress.  
   - Connect the true output of the If node to this node.

8. **Add a WordPress node ("Create WordPress Post")**  
   - Configure WordPress credentials (OAuth2 or application password).  
   - Set parameters for post creation (post type, title, content).  
   - Connect input from "link to binary".

9. **Add an HTTP Request node ("download image")**  
   - Configure to download the featured image or media from WordPress post data.

10. **Add an Edit Image node ("Edit Image")**  
    - Set image processing parameters (resize, format conversions).

11. **Add an HTTP Request node ("upload media to wp")**  
    - Configure to upload the edited image binary to the WordPress media library.

12. **Add an HTTP Request node ("upload image to meta data")**  
    - Configure to associate the uploaded media with WordPress post metadata.

13. **Add an HTTP Request node ("set featured image")**  
    - Configure to set the uploaded media as the post’s featured image.

14. **Add social media posting nodes:**  
    - Facebook Graph API node ("Post on Facebook page")  
    - Telegram node ("Post on telegram Channel")  
    - LinkedIn nodes ("Post on LinkkedIN Profile" and "Post on LinkedN page")  
    - Discord node ("Post on Discord Channel")  
    - Configure each node with appropriate credentials, target pages/groups, and message formatting. Connect all to "link to binary".

15. **Add a No Operation node ("Do nothing")**  
    - Connect as a fan-out branch from "link to binary" to trigger notifications without modifying data.

16. **Add notification nodes:**  
    - Rapiwa node ("Rapiwa (sent whatsapp Notification)") for WhatsApp notifications — configure API credentials.  
    - Telegram node ("Send a Notification") — configure bot and chat IDs.  
    - Gmail node ("Send a Notification1") — configure OAuth2 credentials.  
    - Discord node ("Sent a Notification") — configure webhook URLs.  
    - Connect all notification nodes to the output of "Do nothing".

17. **Connect nodes as per the workflow logic:**  
    - Schedule Trigger → Add RSS link → RSS Read Domani → AI Agent → OpenAI (internal)  
    - AI Agent → If (check link) → True → link to binary  
    - link to binary → Create WordPress Post → download image → Edit Image → upload media to wp → upload image to meta data → set featured image  
    - link to binary → all social post nodes  
    - link to binary → Do nothing → notification nodes

18. **Test the workflow:**  
    - Verify API credentials for each platform.  
    - Confirm RSS feed accessibility and AI processing output correctness.  
    - Check media uploads and post creation on WordPress.  
    - Confirm posts appear on all social channels and notifications are sent.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                            |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------|
| Workflow automates multi-channel publishing from RSS feeds leveraging AI content processing and media handling. | Workflow purpose summary                   |
| For WordPress nodes, ensure REST API and authentication is properly configured for media uploads and posts.    | WordPress API docs: https://developer.wordpress.org/rest-api/ |
| OpenAI API key must have appropriate permissions and sufficient quota for Langchain usage.                     | OpenAI docs: https://platform.openai.com/docs/ |
| Social media posting nodes require valid OAuth2 credentials and permissions for posting on respective platforms.| Facebook, LinkedIn, Telegram API docs     |
| Rapiwa node requires a configured account for WhatsApp notifications.                                          | Rapiwa docs: https://docs.rapiwa.com       |
| Consider error handling for network timeouts, invalid credentials, and API rate limits at each external node.  | n8n error handling best practices          |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.