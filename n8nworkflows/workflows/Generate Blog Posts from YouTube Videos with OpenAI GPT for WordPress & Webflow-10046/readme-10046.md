Generate Blog Posts from YouTube Videos with OpenAI GPT for WordPress & Webflow

https://n8nworkflows.xyz/workflows/generate-blog-posts-from-youtube-videos-with-openai-gpt-for-wordpress---webflow-10046


# Generate Blog Posts from YouTube Videos with OpenAI GPT for WordPress & Webflow

### 1. Workflow Overview

This n8n workflow automates the creation and publication of blog posts based on newly uploaded YouTube videos. It is designed for content creators, marketers, and businesses who want to repurpose their YouTube video content into SEO-optimized blog posts, publishing them automatically to WordPress and Webflow platforms.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Monitoring:** Polls a specified YouTube channel RSS feed on a weekly schedule to detect new videos.
- **1.2 Video Metadata Extraction:** Retrieves detailed information about each new video, including title, description, URL, and thumbnail.
- **1.3 AI Blog Post Generation:** Uses OpenAI GPT (GPT-4.1-mini) to generate a comprehensive, structured blog post in markdown format based on the video data.
- **1.4 Blog Post Formatting:** Cleans and formats the AI-generated markdown content, generates blog post metadata like slug and featured image URL.
- **1.5 Content Conversion:** Converts markdown blog content to HTML suitable for CMS platforms.
- **1.6 Publishing:** Publishes the formatted blog post to WordPress and Webflow, with separate error handling and notifications.
- **1.7 Error Notification:** Sends Telegram alerts if publishing fails on either platform.
- **1.8 Flow Control:** Manages batch processing and rate limiting to ensure smooth execution and API compliance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Monitoring

- **Overview:**  
  Triggers the workflow on a weekly basis and reads the YouTube channel’s RSS feed to detect new video uploads.

- **Nodes Involved:**  
  - Weekly RSS Check (Schedule Trigger)  
  - Monitor YouTube Feed (RSS Feed Read)  
  - Process Each Video (Split In Batches)  

- **Node Details:**  

  - **Weekly RSS Check**  
    - Type: Schedule Trigger  
    - Configuration: Runs every Monday at 21:00 (once per week)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to Monitor YouTube Feed  
    - Edge cases: Workflow won’t trigger off-schedule; time zone differences might affect triggering time.

  - **Monitor YouTube Feed**  
    - Type: RSS Feed Read  
    - Configuration: RSS URL set to YouTube channel’s videos feed (replace `UXXXXXXXXX` with channel ID)  
    - Inputs: Trigger from Weekly RSS Check  
    - Outputs: Connects to Process Each Video  
    - Edge cases: RSS feed unavailability or incorrect channel ID will cause no new videos to process.

  - **Process Each Video**  
    - Type: Split In Batches  
    - Configuration: Default batch size and options (processes one video at a time)  
    - Inputs: List of new videos from RSS feed  
    - Outputs: Two branches:  
      - Main (empty) for batch completion  
      - Secondary branch to Get video details node for video metadata enrichment  
    - Edge cases: Large batches could cause rate limiting downstream, but default is safe.

#### 2.2 Video Metadata Extraction

- **Overview:**  
  Retrieves detailed video metadata from YouTube API for each video identified in the RSS feed.

- **Nodes Involved:**  
  - Get video details (YouTube)  
  - Extract Video Data (Set)  

- **Node Details:**  

  - **Get video details**  
    - Type: YouTube node  
    - Configuration: Extracts video ID from RSS feed link parameter (splits URL by '=' and takes second part)  
    - Credentials: YouTube OAuth2 API credentials required  
    - Inputs: From Process Each Video (batch output)  
    - Outputs: Connects to Extract Video Data  
    - Edge cases: Invalid video ID or expired OAuth token may cause API failure.

  - **Extract Video Data**  
    - Type: Set  
    - Configuration: Maps YouTube API response fields to internal variables:  
      - videoTitle: snippet.title  
      - videoUrl: link from RSS feed  
      - videoId: id  
      - thumbnailUrl: snippet.thumbnails.maxres.url  
      - videoDescription: snippet.description  
    - Inputs: From Get video details  
    - Outputs: Connects to AI Blog Generator  
    - Edge cases: Missing thumbnail at maxres resolution may cause undefined values.

#### 2.3 AI Blog Post Generation

- **Overview:**  
  Sends structured prompt with video metadata to OpenAI GPT-4.1-mini model to generate a detailed blog post in markdown format.

- **Nodes Involved:**  
  - AI Blog Generator (OpenAI LangChain node)  
  - AI Note (Sticky Note)

- **Node Details:**  

  - **AI Blog Generator**  
    - Type: OpenAI (via LangChain)  
    - Configuration:  
      - Model: GPT-4.1-mini  
      - Temperature: 0.7 (balances creativity and coherence)  
      - Messages: System prompt defines expert content writer persona; user prompt requests blog post based on video metadata with specific structure and tone (conversational, engaging, 600-800 words, markdown formatting)  
    - Credentials: OpenAI API key required  
    - Inputs: Video metadata from Extract Video Data  
    - Outputs: Connects to Format Blog Post  
    - Edge cases: API rate limits, token length limits, or network issues may cause failures or incomplete generation.

#### 2.4 Blog Post Formatting

- **Overview:**  
  Cleans the AI-generated markdown, extracts blog title and slug, and prepares the blog content and metadata for CMS publishing.

- **Nodes Involved:**  
  - Format Blog Post (Set)  
  - Format Note (Sticky Note)  

- **Node Details:**  

  - **Format Blog Post**  
    - Type: Set  
    - Configuration:  
      - blogTitle: Extracts the first markdown header (line starting with '#') from AI response content  
      - blogContent: Removes markdown code block markers from AI content for clean content  
      - blogSlug: Generates URL-friendly slug from video title (lowercase, hyphens, no special chars)  
      - featuredImage: Uses thumbnail URL from video data  
    - Inputs: From AI Blog Generator  
    - Outputs: Connects to Convert to HTML  
    - Edge cases: If AI response lacks expected header or formatting, extraction may fail or yield incorrect titles/slugs.

#### 2.5 Content Conversion

- **Overview:**  
  Converts markdown blog content to HTML for compatibility with WordPress and Webflow publishing.

- **Nodes Involved:**  
  - Convert to HTML (Markdown node)  
  - Format Note1 (Sticky Note)  

- **Node Details:**  

  - **Convert to HTML**  
    - Type: Markdown converter  
    - Configuration: Converts markdown (blogContent) to HTML  
    - Inputs: From Format Blog Post  
    - Outputs: Feeds both Publish to WordPress and Publish to Webflow nodes  
    - Edge cases: Complex markdown elements may not convert perfectly; ensure AI output markdown is standard.

#### 2.6 Publishing

- **Overview:**  
  Publishes the HTML blog post to WordPress and Webflow CMS platforms with error handling and rate limiting.

- **Nodes Involved:**  
  - Publish to WordPress (WordPress node)  
  - Publish to Webflow (Webflow node)  
  - Rate Limit Delay (Wait node)  
  - Publish Note & Publish Note1 (Sticky Notes)  

- **Node Details:**  

  - **Publish to WordPress**  
    - Type: WordPress node  
    - Configuration:  
      - Title, slug, content, tags, and categories set dynamically from the workflow data  
      - Tags and categories hardcoded as IDs `[6]` and `[5]` respectively (customize as needed)  
      - On error: continue workflow and trigger error notification  
    - Credentials: WordPress API credentials required (OAuth or API key)  
    - Inputs: From Convert to HTML  
    - Outputs: On success to Rate Limit Delay; on error to Send Error Notification  
    - Edge cases: Auth failure, API downtime, content validation errors.

  - **Publish to Webflow**  
    - Type: Webflow node  
    - Configuration:  
      - Site ID and collection ID specified  
      - Fields set: name, page-content (HTML), slug, featured-image URL  
      - Operation: create item in CMS collection  
      - On error: continue workflow and trigger error notification  
    - Credentials: Webflow OAuth2 credentials required  
    - Inputs: From Convert to HTML  
    - Outputs: On success to Rate Limit Delay; on error to Send Error Alert  
    - Edge cases: Auth failure, API limits, missing collection fields.

  - **Rate Limit Delay**  
    - Type: Wait node  
    - Configuration: Default (no custom delay set explicitly)  
    - Purpose: Prevents hitting API rate limits by pacing processing of videos  
    - Inputs: From both Publish nodes  
    - Outputs: Loops back to Process Each Video to continue batch processing  
    - Edge cases: If delay is too short, may still hit rate limits; too long delays overall workflow.

#### 2.7 Error Notification

- **Overview:**  
  Sends Telegram alerts if publishing fails on either WordPress or Webflow.

- **Nodes Involved:**  
  - Send Error Notification (Telegram node) — WordPress errors  
  - Send Error Alert (Telegram node) — Webflow errors  
  - Publish Note1 (Sticky Note)  

- **Node Details:**  

  - **Send Error Notification**  
    - Type: Telegram node  
    - Configuration: Sends message including video title, error detail, and timestamp  
    - Credentials: Telegram Bot API credentials required  
    - Inputs: From Publish to WordPress error output  
    - Outputs: None (terminal node)  
    - Edge cases: Telegram API failures, invalid chat ID.

  - **Send Error Alert**  
    - Type: Telegram node  
    - Configuration: Similar to above, but for Webflow publishing errors  
    - Inputs: From Publish to Webflow error output  
    - Outputs: None  
    - Edge cases: Same as above.

---

### 3. Summary Table

| Node Name             | Node Type                   | Functional Role                       | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                   |
|-----------------------|-----------------------------|------------------------------------|------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Weekly RSS Check       | Schedule Trigger            | Triggers workflow weekly           | None                   | Monitor YouTube Feed           |                                                                                                |
| Monitor YouTube Feed   | RSS Feed Read               | Reads YouTube channel RSS feed     | Weekly RSS Check       | Process Each Video             |                                                                                                |
| Process Each Video     | Split In Batches            | Processes videos one by one        | Monitor YouTube Feed   | Get video details (secondary), batch output empty |                                                                                                |
| Get video details      | YouTube                    | Fetches full video metadata        | Process Each Video     | Extract Video Data             |                                                                                                |
| Extract Video Data     | Set                         | Maps video metadata fields         | Get video details      | AI Blog Generator             | Extracts video title, description, URL, thumbnail. Creates clean slug for blog post.           |
| AI Blog Generator      | OpenAI (LangChain)           | Generates blog post from video data| Extract Video Data     | Format Blog Post              | Uses OpenAI to create comprehensive blog post with proper structure and engaging content.      |
| Format Blog Post       | Set                         | Cleans AI output and sets metadata | AI Blog Generator      | Convert to HTML               | Cleans up AI response and prepares final blog post structure.                                 |
| Convert to HTML        | Markdown                    | Converts markdown to HTML          | Format Blog Post       | Publish to WordPress, Publish to Webflow | Formats Markdown to HTML for Webflow and WordPress                                            |
| Publish to WordPress   | WordPress                   | Publishes blog post to WordPress  | Convert to HTML        | Rate Limit Delay (success), Send Error Notification (error) | Publishes to WordPress and/or Webflow. Error handling ensures workflow continues if one platform fails. |
| Publish to Webflow     | Webflow                     | Publishes blog post to Webflow    | Convert to HTML        | Rate Limit Delay (success), Send Error Alert (error) | Publishes to WordPress and/or Webflow. Error handling ensures workflow continues if one platform fails. |
| Rate Limit Delay       | Wait                        | Controls execution pace for rate limits | Publish to WordPress, Publish to Webflow | Process Each Video               |                                                                                                |
| Send Error Notification| Telegram                    | Sends Telegram alert on WP error  | Publish to WordPress   | None                         | Optional: You can just leave one notification node                                             |
| Send Error Alert       | Telegram                    | Sends Telegram alert on Webflow error | Publish to Webflow     | None                         | Optional: You can just leave one notification node                                             |
| Workflow Info          | Sticky Note                 | Describes full workflow context   | None                   | None                         | # Information ... (full workflow explanation and setup instructions)                          |
| Extract Note           | Sticky Note                 | Notes on Extract Video Data node  | None                   | None                         | Extracts video title, description, URL, thumbnail. Creates clean slug for blog post.           |
| AI Note                | Sticky Note                 | Notes on AI Blog Generator node   | None                   | None                         | Uses OpenAI to create comprehensive blog post with proper structure and engaging content.      |
| Format Note            | Sticky Note                 | Notes on Format Blog Post node    | None                   | None                         | Cleans up AI response and prepares final blog post structure.                                 |
| Publish Note           | Sticky Note                 | Notes on publishing nodes         | None                   | None                         | Publishes to WordPress and/or Webflow. Error handling ensures workflow continues if one platform fails. |
| Extract Note1          | Sticky Note                 | Notes on Get video details node   | None                   | None                         | Uses Video ID to extract full description for extra context                                  |
| Format Note1           | Sticky Note                 | Notes on Convert to HTML node     | None                   | None                         | Formats Markdown to HTML for Webflow and WordPress                                            |
| Publish Note1          | Sticky Note                 | Notes on error notification nodes | None                   | None                         | Optional: You can just leave one notification node                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Name: `Weekly RSS Check`  
   - Set to trigger weekly on Monday at 21:00.

2. **Create RSS Feed Read node**  
   - Name: `Monitor YouTube Feed`  
   - Set URL to YouTube channel RSS feed: `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID`  
   - Connect output of `Weekly RSS Check` to this node.

3. **Create Split In Batches node**  
   - Name: `Process Each Video`  
   - Default options (batch size 1)  
   - Connect output of `Monitor YouTube Feed` to this node.

4. **Create YouTube node**  
   - Name: `Get video details`  
   - Resource: Video  
   - Operation: Get  
   - Video ID expression: `{{$json.link.split('=')[1]}}` (extracts video ID from RSS feed link)  
   - Configure YouTube OAuth2 credentials.

5. **Connect secondary output of `Process Each Video` (batch) to `Get video details`.**

6. **Create Set node**  
   - Name: `Extract Video Data`  
   - Assign variables:  
     - `videoTitle`: `{{$json.snippet.title}}`  
     - `videoUrl`: `{{$('Monitor YouTube Feed').item.json.link}}`  
     - `videoId`: `{{$json.id}}`  
     - `thumbnailUrl`: `{{$json.snippet.thumbnails.maxres.url}}`  
     - `videoDescription`: `{{$json.snippet.description}}`  
   - Connect `Get video details` output to this node.

7. **Create OpenAI LangChain node**  
   - Name: `AI Blog Generator`  
   - Model: GPT-4.1-mini  
   - Temperature: 0.7  
   - Messages:  
     - System role: Expert content writer persona with instructions to structure content with headings and actionable takeaways.  
     - User role: Prompt with video metadata: title, description, URL; request a 600-800 word markdown blog post targeting no-code developers and Bubble.io users, engaging tone, proper markdown formatting.  
   - Configure OpenAI API credentials.  
   - Connect `Extract Video Data` to this node.

8. **Create Set node**  
   - Name: `Format Blog Post`  
   - Assign variables:  
     - `blogTitle`: Extract first markdown header from AI content: `{{$json.message.content.match(/#\s*(.+)/)[1].trim()}}`  
     - `blogContent`: Remove markdown code block markers: `{{$json.message.content.replace(/```(?:\w+)?\n?/, '').replace(/```$/, '').trim()}}`  
     - `blogSlug`: Generate slug from video title: `{{$('Extract Video Data').item.json.videoTitle.toLowerCase().replace(/[^a-z0-9]/g, '-').replace(/-+/g, '-').replace(/^-|-$/g, '')}}`  
     - `featuredImage`: `{{$('Extract Video Data').item.json.thumbnailUrl}}`  
   - Connect `AI Blog Generator` to this node.

9. **Create Markdown node**  
   - Name: `Convert to HTML`  
   - Mode: Markdown to HTML  
   - Input markdown field: `blogContent`  
   - Connect `Format Blog Post` to this node.

10. **Create WordPress node**  
    - Name: `Publish to WordPress`  
    - Operation: Create post  
    - Title: `{{$json.blogTitle}}`  
    - Slug: `{{$json.blogSlug}}`  
    - Content: `{{$json.data}}` (HTML from previous node)  
    - Tags: `[6]` (customize as needed)  
    - Categories: `[5]` (customize)  
    - Credentials: WordPress API OAuth2 or API key  
    - On error: Continue workflow, connect error output to Telegram error notification node.  
    - Connect success output to `Rate Limit Delay`.

11. **Create Webflow node**  
    - Name: `Publish to Webflow`  
    - Site ID: Your Webflow site ID  
    - Collection ID: Your CMS collection ID  
    - Fields to set:  
      - `name`: `{{$json.blogTitle}}`  
      - `page-content`: `{{$json.data}}` (HTML)  
      - `slug`: `{{$json.blogSlug}}`  
      - `featured-image`: `{{$json.featuredImage}}`  
    - Credentials: Webflow OAuth2  
    - On error: Continue workflow, connect error output to Telegram error alert node.  
    - Connect success output to `Rate Limit Delay`.

12. **Create Wait node**  
    - Name: `Rate Limit Delay`  
    - Default delay (optional: configure delay duration based on API limits)  
    - Connect outputs of both publishing nodes to this node.  
    - Connect output of this node back to `Process Each Video` to continue processing the next batch.

13. **Create Telegram nodes for error notifications**  
    - Name: `Send Error Notification` (for WordPress errors)  
    - Configure message text with video title, error message, and timestamp.  
    - Connect error output of `Publish to WordPress` to this node.  
    - Name: `Send Error Alert` (for Webflow errors)  
    - Similar configuration and connection from `Publish to Webflow` error output.  
    - Both require Telegram Bot API credentials and valid chat ID.

14. **Add Sticky Notes for documentation** at appropriate positions to describe each block and node function as per your preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow is designed for content creators, marketers, and businesses managing YouTube and blog platforms simultaneously.                                                                                                                                                                                                                                                                                                                                      | Workflow Info sticky note                                                                             |
| Replace `YOUR_CHANNEL_ID` in RSS feed URL with your actual YouTube channel ID for monitoring.                                                                                                                                                                                                                                                                                                                                                                | Monitor YouTube Feed node configuration                                                             |
| OpenAI API key and credentials must be set up in n8n for the AI Blog Generator node.                                                                                                                                                                                                                                                                                                                                                                          | AI Blog Generator node                                                                                |
| WordPress and Webflow credentials require OAuth2 or API key setup before publishing nodes can operate.                                                                                                                                                                                                                                                                                                                                                         | Publish to WordPress and Publish to Webflow nodes                                                    |
| Telegram bot setup is optional but recommended for error notifications to maintain workflow health.                                                                                                                                                                                                                                                                                                                                                            | Send Error Notification and Send Error Alert nodes                                                  |
| Customize AI prompt to adjust blog post style, length, tone, or target audience.                                                                                                                                                                                                                                                                                                                                                                               | AI Blog Generator node                                                                                |
| Adjust polling frequency in the Schedule Trigger to suit your content update cadence (30-60 minutes recommended for higher frequency).                                                                                                                                                                                                                                                                                                                        | Weekly RSS Check node                                                                                |
| Ensure video thumbnails exist in max resolution or fallback to lower thumbnail resolutions to avoid empty featured image URLs.                                                                                                                                                                                                                                                                                                                               | Extract Video Data node                                                                               |
| Error handling on publishing nodes ensures the entire workflow continues even if one platform fails, with notifications sent accordingly.                                                                                                                                                                                                                                                                                                                    | Publish to WordPress and Publish to Webflow nodes                                                   |
| Rate limiting via Wait node helps prevent API throttling especially with multiple videos in batch processing. Adjust delay as needed depending on API usage.                                                                                                                                                                                                                                                                                                  | Rate Limit Delay node                                                                                |
| Useful external resource on n8n WordPress integration: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/                                                                                                                                                                                                                                                                                                                          | Official n8n documentation                                                                           |
| Useful external resource on n8n Webflow integration: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.webflow/                                                                                                                                                                                                                                                                                                                               | Official n8n documentation                                                                           |
| For detailed OpenAI prompt engineering best practices, see https://platform.openai.com/docs/guides/prompting                                                                                                                                                                                                                                                                                                                                                   | OpenAI documentation                                                                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.