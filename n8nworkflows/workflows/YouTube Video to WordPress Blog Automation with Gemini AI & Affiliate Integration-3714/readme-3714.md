YouTube Video to WordPress Blog Automation with Gemini AI & Affiliate Integration

https://n8nworkflows.xyz/workflows/youtube-video-to-wordpress-blog-automation-with-gemini-ai---affiliate-integration-3714


# YouTube Video to WordPress Blog Automation with Gemini AI & Affiliate Integration

### 1. Workflow Overview

This n8n workflow automates the transformation of newly uploaded YouTube videos into SEO-optimized WordPress blog posts enriched with affiliate marketing links. It is designed for content creators and marketers who want to repurpose video content into written articles automatically, saving time and boosting monetization.

The workflow is logically divided into the following blocks:

- **1.1 YouTube Video Detection:** Monitors a YouTube channel RSS feed for new videos and extracts video IDs.
- **1.2 Existing WordPress Content Retrieval:** Fetches existing WordPress posts via sitemap to avoid duplicate publications and to provide internal linking data.
- **1.3 Video Metadata Extraction:** Retrieves detailed metadata for each new video from YouTube.
- **1.4 AI Content Generation:** Uses Google Gemini AI to generate structured, SEO-friendly blog content, including titles, excerpts, tags, and affiliate links.
- **1.5 Content Formatting and Image Handling:** Processes AI output, formats blog post content, downloads and uploads featured images to WordPress.
- **1.6 WordPress Publishing:** Publishes the generated blog post to WordPress with all metadata and featured images.
- **1.7 Affiliate Link Integration:** Integrates affiliate links dynamically into the blog content using PartnerStack API and Airtable product database.
- **1.8 Notification and Logging:** (Implied in description, not explicitly shown in nodes) For alerts and execution tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 YouTube Video Detection

- **Overview:** Detects new YouTube videos by reading the channel's RSS feed and extracts video IDs for further processing.
- **Nodes Involved:**  
  - RSS Feed Trigger  
  - EGet Video ID

- **Node Details:**

  - **RSS Feed Trigger**  
    - Type: Trigger node that monitors an RSS feed URL.  
    - Configuration: Set to the YouTube channel's RSS feed URL to detect new video uploads.  
    - Inputs: None (trigger).  
    - Outputs: Emits new video entries with metadata such as video link, title, and publication date.  
    - Edge Cases: RSS feed downtime, malformed feed, no new videos.  
    - Version: v1.

  - **EGet Video ID**  
    - Type: Set node used to extract and set the video ID from the RSS feed item.  
    - Configuration: Extracts video ID from the RSS feed link or identifier for use in API calls.  
    - Inputs: Output from RSS Feed Trigger.  
    - Outputs: Passes video ID to the next node.  
    - Edge Cases: Incorrect parsing if URL format changes.

#### 1.2 Existing WordPress Content Retrieval

- **Overview:** Retrieves existing WordPress posts via sitemap to prevent duplicate blog posts and to supply internal linking data for AI content generation.
- **Nodes Involved:**  
  - Get Post SiteMap  
  - Conver to JSON  
  - Extract URLs  
  - URL Lists  
  - Get all Posts for AI

- **Node Details:**

  - **Get Post SiteMap**  
    - Type: HTTP Request node.  
    - Configuration: Fetches the WordPress sitemap XML from the website.  
    - Inputs: From Edit Fields node (later in flow).  
    - Outputs: Raw XML sitemap data.  
    - Edge Cases: Sitemap URL changes, HTTP errors, large sitemap size.  
    - Version: v4.2.

  - **Conver to JSON**  
    - Type: XML node.  
    - Configuration: Converts XML sitemap to JSON format for easier processing.  
    - Inputs: Output from Get Post SiteMap.  
    - Outputs: JSON representation of sitemap URLs.  
    - Edge Cases: XML parsing errors if sitemap malformed.  
    - Version: v1.

  - **Extract URLs**  
    - Type: SplitOut node.  
    - Configuration: Extracts individual URL entries from sitemap JSON.  
    - Inputs: JSON from Conver to JSON.  
    - Outputs: Individual URLs for aggregation.  
    - Edge Cases: Empty or missing URL entries.  
    - Version: v1.

  - **URL Lists**  
    - Type: Aggregate node.  
    - Configuration: Aggregates extracted URLs into a list for AI processing.  
    - Inputs: Output from Extract URLs.  
    - Outputs: Aggregated list of existing post URLs.  
    - Edge Cases: Large number of URLs may affect performance.  
    - Version: v1.

  - **Get all Posts for AI**  
    - Type: Set node.  
    - Configuration: Prepares the aggregated URL list for input into AI content generation.  
    - Inputs: Output from URL Lists.  
    - Outputs: Passes existing posts data to AI nodes.  
    - Edge Cases: Data formatting errors.

#### 1.3 Video Metadata Extraction

- **Overview:** Retrieves detailed metadata for each new YouTube video to provide context for AI content generation.
- **Nodes Involved:**  
  - Get Details of Video from Youtube  
  - Edit Fields

- **Node Details:**

  - **Get Details of Video from Youtube**  
    - Type: YouTube node.  
    - Configuration: Uses YouTube Data API to fetch video details such as title, description, tags, thumbnail URL, and embed HTML.  
    - Inputs: Video ID from EGet Video ID.  
    - Outputs: Detailed video metadata.  
    - Edge Cases: API quota limits, invalid video ID, network errors.  
    - Version: v1.

  - **Edit Fields**  
    - Type: Set node.  
    - Configuration: Cleans or modifies video metadata fields for downstream processing.  
    - Inputs: Output from YouTube node.  
    - Outputs: Sanitized video metadata.  
    - Edge Cases: Missing fields, unexpected data formats.

#### 1.4 AI Content Generation

- **Overview:** Uses Google Gemini AI to generate structured, SEO-optimized blog content including titles, excerpts, tags, and affiliate links based on video metadata and existing WordPress content.
- **Nodes Involved:**  
  - Technical Blog Writer  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - affiliate_links  
  - wp_category  
  - wp_tag  
  - lms

- **Node Details:**

  - **Technical Blog Writer**  
    - Type: LangChain Agent node.  
    - Configuration: Orchestrates AI content generation using inputs from video metadata, existing posts, and affiliate data.  
    - Inputs: Existing posts (Get all Posts for AI), AI language model (Google Gemini Chat Model), affiliate links, categories, tags, and LMS data.  
    - Outputs: AI-generated blog post content with metadata.  
    - Edge Cases: AI timeout, incomplete or irrelevant content, API errors.  
    - Version: v1.8.

  - **Google Gemini Chat Model**  
    - Type: LangChain AI Language Model node.  
    - Configuration: Connects to Google Gemini API for chat-based AI content generation.  
    - Inputs: Prompts from Technical Blog Writer.  
    - Outputs: AI-generated text responses.  
    - Edge Cases: API quota, network issues, prompt formatting errors.  
    - Version: v1.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser node.  
    - Configuration: Parses AI output into structured JSON format for easier handling.  
    - Inputs: AI text output from Google Gemini Chat Model.  
    - Outputs: Structured blog content data.  
    - Edge Cases: Parsing errors if AI output deviates from expected format.  
    - Version: v1.2.

  - **affiliate_links**  
    - Type: LangChain HTTP Request Tool node.  
    - Configuration: Fetches affiliate link data dynamically, likely from PartnerStack or Airtable.  
    - Inputs: Technical Blog Writer node for integration into content.  
    - Outputs: Affiliate link data for insertion.  
    - Edge Cases: API authentication errors, data unavailability.  
    - Version: v1.1.

  - **wp_category**  
    - Type: LangChain HTTP Request Tool node.  
    - Configuration: Retrieves or suggests WordPress categories for the post.  
    - Inputs: Technical Blog Writer.  
    - Outputs: Category data.  
    - Edge Cases: API errors, missing categories.  
    - Version: v1.1.

  - **wp_tag**  
    - Type: LangChain HTTP Request Tool node.  
    - Configuration: Retrieves or suggests WordPress tags for the post.  
    - Inputs: Technical Blog Writer.  
    - Outputs: Tag data.  
    - Edge Cases: API errors, missing tags.  
    - Version: v1.1.

  - **lms**  
    - Type: Airtable Tool node.  
    - Configuration: Accesses an Airtable base, possibly for affiliate product database or learning management system data.  
    - Inputs: Technical Blog Writer.  
    - Outputs: Data for AI content enrichment.  
    - Edge Cases: Airtable API limits, authentication errors.  
    - Version: v2.1.

#### 1.5 Content Formatting and Image Handling

- **Overview:** Processes AI-generated content, formats it for WordPress, downloads video thumbnails, renames and uploads images, and updates image metadata.
- **Nodes Involved:**  
  - BlogPost  
  - Get Image  
  - rename image  
  - Upload Image  
  - Update Image Details

- **Node Details:**

  - **BlogPost**  
    - Type: Code node.  
    - Configuration: Custom JavaScript to format AI-generated content into WordPress-compatible HTML, embed video player, and prepare post content.  
    - Inputs: AI-generated content from Technical Blog Writer.  
    - Outputs: Formatted blog post content.  
    - Edge Cases: Code errors, malformed HTML, missing content fields.  
    - Version: v2.

  - **Get Image**  
    - Type: HTTP Request node.  
    - Configuration: Downloads the video thumbnail image from YouTube or other sources.  
    - Inputs: Formatted blog post data with image URL.  
    - Outputs: Binary image data for upload.  
    - Edge Cases: Image URL invalid, network errors, large image size.  
    - Version: v4.2.

  - **rename image**  
    - Type: Code node.  
    - Configuration: Renames the downloaded image file to a suitable filename for WordPress media library.  
    - Inputs: Binary image data from Get Image.  
    - Outputs: Renamed image binary data.  
    - Edge Cases: Filename conflicts, invalid characters.  
    - Version: v2.

  - **Upload Image**  
    - Type: HTTP Request node.  
    - Configuration: Uploads the renamed image to WordPress media library via REST API.  
    - Inputs: Renamed image data.  
    - Outputs: WordPress media item response.  
    - Edge Cases: Authentication errors, upload size limits, API errors.  
    - Version: v4.2.

  - **Update Image Details**  
    - Type: HTTP Request node.  
    - Configuration: Updates image metadata (title, alt text, description) in WordPress after upload.  
    - Inputs: Upload Image response.  
    - Outputs: Confirmation of update.  
    - Edge Cases: API errors, missing media ID.  
    - Version: v4.2.

#### 1.6 WordPress Publishing

- **Overview:** Publishes the fully prepared blog post to WordPress, setting all relevant post parameters including title, content, excerpt, categories, tags, featured image, and post status.
- **Nodes Involved:**  
  - Publish Post WP

- **Node Details:**

  - **Publish Post WP**  
    - Type: HTTP Request node.  
    - Configuration: Sends a POST request to WordPress REST API to create or update a blog post. Includes authentication via application password or OAuth2.  
    - Inputs: Formatted post content and metadata, image details from Update Image Details.  
    - Outputs: WordPress post creation response.  
    - Edge Cases: Authentication failure, permission issues, API rate limits, malformed post data.  
    - Version: v4.2.

#### 1.7 Affiliate Link Integration

- **Overview:** Dynamically fetches and inserts affiliate links into the blog content based on video topics and product relevance, using PartnerStack API and Airtable database.
- **Nodes Involved:**  
  - affiliate_links  
  - lms (Airtable Tool)

- **Node Details:**

  - **affiliate_links**  
    - Type: LangChain HTTP Request Tool node.  
    - Configuration: Queries PartnerStack API or other affiliate sources to retrieve relevant affiliate links.  
    - Inputs: AI content context from Technical Blog Writer.  
    - Outputs: Affiliate link data for insertion.  
    - Edge Cases: API authentication failure, no relevant products found.  
    - Version: v1.1.

  - **lms**  
    - Type: Airtable Tool node.  
    - Configuration: Accesses affiliate product keyword database for matching and rotation.  
    - Inputs: Technical Blog Writer.  
    - Outputs: Affiliate product data.  
    - Edge Cases: Airtable API limits, data inconsistencies.  
    - Version: v2.1.

#### 1.8 Notification & Logging

- **Overview:** While not explicitly shown in the nodes, the description mentions notifications (email or Slack) and logging for workflow executions and publication tracking.
- **Nodes Involved:** None explicitly in provided workflow JSON.
- **Implementation Notes:**  
  - Could be implemented via additional nodes such as Email, Slack, or Function nodes for logging.  
  - Important for monitoring success and troubleshooting failures.

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                                  | Input Node(s)                     | Output Node(s)                  | Sticky Note |
|----------------------------|---------------------------------------------|-------------------------------------------------|---------------------------------|--------------------------------|-------------|
| RSS Feed Trigger            | rssFeedReadTrigger (Trigger)                | Detect new YouTube videos via RSS feed          | None                            | EGet Video ID                  |             |
| EGet Video ID              | Set                                         | Extract video ID from RSS feed                   | RSS Feed Trigger                | Get Details of Video from Youtube |             |
| Get Details of Video from Youtube | YouTube                                  | Fetch detailed video metadata from YouTube API  | EGet Video ID                   | Edit Fields                   |             |
| Edit Fields                | Set                                         | Clean and prepare video metadata                 | Get Details of Video from Youtube | Get Post SiteMap              |             |
| Get Post SiteMap           | HTTP Request                                | Retrieve WordPress sitemap XML                    | Edit Fields                    | Conver to JSON                |             |
| Conver to JSON             | XML                                         | Convert sitemap XML to JSON                        | Get Post SiteMap               | Extract URLs                  |             |
| Extract URLs               | SplitOut                                    | Extract individual URLs from sitemap JSON        | Conver to JSON                 | URL Lists                    |             |
| URL Lists                  | Aggregate                                   | Aggregate URLs into list for AI                   | Extract URLs                   | Get all Posts for AI          |             |
| Get all Posts for AI       | Set                                         | Prepare existing posts data for AI                | URL Lists                     | Technical Blog Writer         |             |
| Technical Blog Writer      | LangChain Agent                             | Generate AI blog content with affiliate integration | Get all Posts for AI, Google Gemini Chat Model, affiliate_links, wp_category, wp_tag, lms | BlogPost                    |             |
| Google Gemini Chat Model   | LangChain LM Chat Model                     | AI language model for content generation          | Technical Blog Writer (AI input) | Technical Blog Writer (AI output) |             |
| Structured Output Parser   | LangChain Output Parser                      | Parse AI output into structured JSON              | Google Gemini Chat Model       | Technical Blog Writer         |             |
| affiliate_links            | LangChain HTTP Request Tool                  | Fetch affiliate links dynamically                  | Technical Blog Writer          | Technical Blog Writer         |             |
| wp_category               | LangChain HTTP Request Tool                  | Retrieve or suggest WordPress categories           | Technical Blog Writer          | Technical Blog Writer         |             |
| wp_tag                    | LangChain HTTP Request Tool                  | Retrieve or suggest WordPress tags                  | Technical Blog Writer          | Technical Blog Writer         |             |
| lms                       | Airtable Tool                               | Access affiliate product database                   | Technical Blog Writer          | Technical Blog Writer         |             |
| BlogPost                  | Code                                        | Format AI content into WordPress post HTML         | Technical Blog Writer          | Get Image                    |             |
| Get Image                 | HTTP Request                                | Download video thumbnail image                      | BlogPost                      | rename image                 |             |
| rename image              | Code                                        | Rename downloaded image for WordPress upload       | Get Image                    | Upload Image                 |             |
| Upload Image              | HTTP Request                                | Upload image to WordPress media library             | rename image                 | Update Image Details         |             |
| Update Image Details      | HTTP Request                                | Update image metadata in WordPress                   | Upload Image                 | Publish Post WP              |             |
| Publish Post WP           | HTTP Request                                | Publish the blog post to WordPress                   | Update Image Details          | None                        |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger Node**  
   - Type: `rssFeedReadTrigger`  
   - Configure with your YouTube channel's RSS feed URL to detect new videos.

2. **Add Set Node "EGet Video ID"**  
   - Extract the video ID from the RSS feed item URL for API use.

3. **Add YouTube Node "Get Details of Video from Youtube"**  
   - Configure with YouTube Data API credentials.  
   - Use video ID from previous node to fetch detailed video metadata.

4. **Add Set Node "Edit Fields"**  
   - Clean or adjust video metadata fields as needed for downstream use.

5. **Add HTTP Request Node "Get Post SiteMap"**  
   - Configure to fetch your WordPress sitemap XML URL (e.g., `https://yourwordpresssite.com/sitemap.xml`).  
   - Use authentication if required.

6. **Add XML Node "Conver to JSON"**  
   - Convert the sitemap XML response to JSON format.

7. **Add SplitOut Node "Extract URLs"**  
   - Extract individual URLs from the sitemap JSON.

8. **Add Aggregate Node "URL Lists"**  
   - Aggregate extracted URLs into a list for AI input.

9. **Add Set Node "Get all Posts for AI"**  
   - Prepare the aggregated URL list for AI processing.

10. **Add LangChain Agent Node "Technical Blog Writer"**  
    - Configure with your AI provider credentials (Google Gemini).  
    - Connect inputs: existing posts list, affiliate links, categories, tags, and LMS data.  
    - Set detailed prompts to generate SEO-optimized blog content with affiliate integration.

11. **Add LangChain Chat Model Node "Google Gemini Chat Model"**  
    - Connect to Google Gemini API with credentials.  
    - Link as AI language model input for Technical Blog Writer.

12. **Add LangChain Output Parser Node "Structured Output Parser"**  
    - Parse AI-generated text into structured JSON for easier handling.

13. **Add LangChain HTTP Request Tool Nodes:**  
    - "affiliate_links": Connect to PartnerStack API or affiliate source.  
    - "wp_category": Retrieve or suggest WordPress categories.  
    - "wp_tag": Retrieve or suggest WordPress tags.

14. **Add Airtable Tool Node "lms"**  
    - Configure with Airtable API credentials.  
    - Connect to affiliate product keyword database.

15. **Add Code Node "BlogPost"**  
    - Write JavaScript to format AI content into WordPress-compatible HTML, embed video player, and prepare post content.

16. **Add HTTP Request Node "Get Image"**  
    - Download video thumbnail image using URL from blog post data.

17. **Add Code Node "rename image"**  
    - Rename the downloaded image file to a WordPress-friendly filename.

18. **Add HTTP Request Node "Upload Image"**  
    - Upload renamed image to WordPress media library via REST API.

19. **Add HTTP Request Node "Update Image Details"**  
    - Update image metadata (title, alt text) in WordPress.

20. **Add HTTP Request Node "Publish Post WP"**  
    - Publish the blog post with all metadata and featured image.  
    - Authenticate using WordPress application password or OAuth2.

21. **Connect nodes in the order described above**, ensuring outputs feed into the correct inputs.

22. **Configure credentials:**  
    - YouTube Data API key for YouTube node.  
    - Google Gemini API key for AI nodes.  
    - WordPress REST API credentials (application password or OAuth2) for HTTP Request nodes.  
    - PartnerStack API credentials for affiliate_links node.  
    - Airtable API key and base for lms node.

23. **Set default values and constraints:**  
    - Set appropriate timeouts for AI generation nodes.  
    - Implement error handling or retries for HTTP requests.  
    - Filter duplicate videos by comparing video URLs with existing WordPress posts.

24. **(Optional) Add notification nodes** such as Email or Slack to alert on successful post publication or errors.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow overview image illustrating the YouTube to WordPress automation process.                         | https://n8n.io/images/workflows/youtube-wordpress-automation.png                                |
| Step-by-step video tutorial for implementing this workflow.                                               | https://youtu.be/IkFgllPNkCo                                                                    |
| YouTube Data API documentation for video metadata retrieval.                                             | https://developers.google.com/youtube/v3                                                        |
| WordPress REST API handbook for post and media management.                                               | https://developer.wordpress.org/rest-api/                                                      |
| OpenAI API documentation (for alternative AI providers).                                                 | https://platform.openai.com/docs/api-reference                                                  |
| PartnerStack affiliate marketing platform integration details.                                           | https://try.partnerstack.com/syncbricks                                                        |
| n8n Cloud platform for easy workflow deployment.                                                         | https://n8n.io/cloud/                                                                           |
| Self-hosting guide for n8n on Ubuntu 24.04.                                                              | https://syncbricks.com/self-hosting-n8n-on-ubuntu-24-04-a-step-by-step-guide/                    |
| Affiliate marketing best practices including FTC compliance and geo-targeting.                            | See workflow description under "Automatic Affiliate Marketing Integration"                      |
| AI content generation tips: maintain brand voice, SEO optimization, internal/external linking, monetization. | See workflow description under "AI Content Generation"                                         |
| Troubleshooting tips: AI timeouts, WordPress API errors, duplicate post prevention, content quality.      | See workflow description under "Troubleshooting"                                               |
| Pro Tip: Use separate affiliate tracking IDs for automated posts to measure channel-specific performance. | See workflow description under "Automatic Affiliate Marketing Integration"                      |

---

This documentation provides a detailed, structured understanding of the YouTube Video to WordPress Blog Automation workflow, enabling users and AI agents to comprehend, reproduce, and customize the process effectively.