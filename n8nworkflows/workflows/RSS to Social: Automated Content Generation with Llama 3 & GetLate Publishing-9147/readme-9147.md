RSS to Social: Automated Content Generation with Llama 3 & GetLate Publishing

https://n8nworkflows.xyz/workflows/rss-to-social--automated-content-generation-with-llama-3---getlate-publishing-9147


# RSS to Social: Automated Content Generation with Llama 3 & GetLate Publishing

### 1. Workflow Overview

This workflow automates the process of transforming RSS feed content into optimized social media posts specifically tailored for LinkedIn and Bluesky platforms. It integrates AI-driven content generation using the Llama 3 model (via Groq API) and orchestrates content publishing through the GetLate API. The workflow includes logical blocks that:

- **1.1 RSS Content Ingestion and Storage:** Fetch tagged RSS feed articles from Wallabag feeds, convert HTML to Markdown, and store raw article content in Airtable as a centralized content repository.
  
- **1.2 Scheduled Article Processing & AI Content Generation:** On a scheduled basis, pull unpublished articles from Airtable, apply AI prompt templates customized for each platform, generate polished post text with hashtags, CTAs, tone, and goals using Llama 3, and parse structured JSON outputs.

- **1.3 Conditional Image Generation & Publishing:** Based on user-defined flags, optionally generate AI images via Hugging Face's Stable Diffusion API, convert images to Base64, upload to Imgbb, and attach media URLs to posts.

- **1.4 Posting & Content Upsert:** Publish or draft posts on GetLate for LinkedIn and Bluesky platforms with or without images, then update Airtable records marking content as posted and storing generated metadata.

The workflow is designed for multi-platform social media automation, leveraging AI to customize content per platform with flexibility for images and draft settings.

---

### 2. Block-by-Block Analysis

#### 2.1 RSS Content Ingestion and Storage

**Overview:**  
This block fetches new articles tagged for social sharing from Wallabag RSS feeds for LinkedIn and Bluesky, converts article HTML content to Markdown, and stores the article data in Airtable for later processing.

**Nodes Involved:**  
- RSS Feed Trigger: '#to-share-linkedin'  
- Set Platform (LinkedIn)  
- Markdown: Convert Article Content  
- Store Article Content (LinkedIn)  
- RSS Feed Trigger: '#to-share-bluesky'  
- Set Platform (Bluesky)  
- Markdown: Convert Article Content1  
- Store Article Content  

**Node Details:**

- **RSS Feed Trigger: '#to-share-linkedin' & '#to-share-bluesky'**  
  - Type: Trigger node that polls RSS feeds every minute.  
  - Configured with Wallabag RSS feed URLs filtered by tags `to-share-linkedin` and `to-share-bluesky`.  
  - Poll interval: every minute for latest content.  
  - Possible failures: connectivity issues, invalid feed URLs, rate limiting.

- **Set Platform (LinkedIn) & Set Platform (Bluesky)**  
  - Type: Set node to label incoming feed items with platform strings ('linkedin' or 'bluesky').  
  - Adds a `platform` attribute to the item to guide downstream conditional logic.

- **Markdown: Convert Article Content & Markdown: Convert Article Content1**  
  - Type: Markdown node converting HTML content from RSS feed to Markdown format for cleaner text processing by AI.  
  - Uses expression to extract `content` from RSS feed item JSON.  
  - Outputs Markdown content keyed by `content_markdown`.  
  - Failure cases: malformed HTML content or empty content.

- **Store Article Content (LinkedIn) & Store Article Content (Bluesky)**  
  - Type: Airtable node that creates new records in a content store base.  
  - Stores fields including title, author, feed ID, platform, article URL, Markdown content, and an `is_posted` flag set to 0 (unpublished).  
  - Failure cases: Airtable API errors, invalid API key, schema mismatches.

---

#### 2.2 Scheduled Article Processing & AI Content Generation

**Overview:**  
This block runs periodically to pull unpublished articles from Airtable, set AI prompt parameters, determine platform-specific processing, generate optimized social media posts with Llama 3 via Groq API, and parse AI responses into structured JSON.

**Nodes Involved:**  
- Set Post Schedule (Schedule Trigger)  
- Pull New Article (Airtable search for unpublished content)  
- Set SMCG Custom Prompt Parameters (Set node for niche, tone, goals, etc.)  
- Check Platform (Switch node)  
- LinkedIn SMCG Prompt (Set node for LinkedIn prompt)  
- Bluesky SMCG Prompt (Set node for Bluesky prompt)  
- Groq Chat Model (AI LLM node for LinkedIn)  
- Groq Chat Model1 (AI LLM node for Bluesky)  
- Structured Output Parser (LinkedIn)  
- Structured Output Parser1 (Bluesky)  

**Node Details:**

- **Set Post Schedule**  
  - Type: Schedule trigger firing every minute (configurable).  
  - Starts the workflow to process pending articles.

- **Pull New Article**  
  - Airtable node searching for articles with `is_posted=0` (unpublished).  
  - Retrieves full article metadata for AI processing.  
  - Edge cases: empty result sets, Airtable API failures.

- **Set SMCG Custom Prompt Parameters**  
  - Set node defining fixed values for niche, tone, goal, target audience, custom hashtags, draft flag, image generation flags, and platform account IDs.  
  - These parameters are merged with the article content for AI prompt inputs.

- **Check Platform (Switch)**  
  - Routes flow to either LinkedIn or Bluesky content generation based on the article’s platform field.  
  - Ensures platform-specific prompt and generation logic.

- **LinkedIn SMCG Prompt & Bluesky SMCG Prompt**  
  - Set nodes constructing detailed prompt text for the Llama 3 model tailored to each platform’s content style and constraints.  
  - Embed article metadata and user-defined parameters with instructions for length, tone, hashtags, CTAs, and image prompt generation.  
  - Output a JSON string as input to the AI model.

- **Groq Chat Model & Groq Chat Model1**  
  - Langchain LLM Chat nodes using Groq API with Llama 3.3 70b model.  
  - Receive prompt text, generate raw AI responses.  
  - Failures: API request limits, auth errors, model timeouts.

- **Structured Output Parser & Structured Output Parser1**  
  - Nodes parse the AI raw response into a strict JSON structure containing keys: platform, post_text, suggested_hashtags (array), cta_used, tone_applied, goal_applied, audience_targeted, character_count, image_prompt.  
  - Validates AI output format.  
  - Failures if AI output is malformed or parsing fails.

---

#### 2.3 Conditional Image Generation & Publishing

**Overview:**  
Depending on the `generate_image_*` boolean flags set in prompt parameters, this block decides whether to generate an AI image, convert it, upload it, and include it in the post. If image generation is disabled, it proceeds directly to posting without media.

**Nodes Involved:**  
- Check For Image Generation Prompt (LinkedIn) & Check For Image Generation Prompt (Bluesky) (If nodes)  
- Generate Post Image (LinkedIn) & Generate Post Image (Bluesky) (HTTP Request to Hugging Face)  
- Convert Image to Base64 (LinkedIn) & Convert Image to Base64 (Bluesky) (Code nodes)  
- Store Image on Imgbb (LinkedIn) & Store Image on Imgbb (Bluesky) (HTTP Request)  

**Node Details:**

- **Check For Image Generation Prompt (If nodes)**  
  - Evaluate if `generate_image_linkedin` or `generate_image_bluesky` is true.  
  - Routes flow to image generation if true, else skips to post without image.

- **Generate Post Image (HTTP Request)**  
  - Calls Hugging Face’s Stable Diffusion XL model endpoint with the AI-generated image prompt.  
  - Receives PNG image response (binary).  
  - Failures: API limits, bad prompt input, network issues.

- **Convert Image to Base64 (Code node)**  
  - Converts binary image response into Base64 encoded string suitable for uploading.  
  - Handles raw HTTP response body.  
  - Edge cases: corrupt image data, conversion errors.

- **Store Image on Imgbb (HTTP Request)**  
  - Uploads Base64 image to Imgbb service via multipart form-data POST.  
  - Sets image name dynamically using article created time, ID, and platform.  
  - Returns image URL and metadata used for publishing.  
  - Failures: authentication errors, upload limits, invalid image data.

---

#### 2.4 Posting & Content Upsert

**Overview:**  
This block publishes or drafts the generated post content on GetLate platform for LinkedIn and Bluesky, with or without images, and updates the Airtable content store marking articles as posted including all generated metadata.

**Nodes Involved:**  
- Post/Draft via GetLate API (LinkedIn) & (Bluesky), with and without image variants  
- Store Generated Post Content (LinkedIn) & (Bluesky), with and without image variants  

**Node Details:**

- **Post/Draft via GetLate API**  
  - HTTP Request nodes calling https://getlate.dev/api/v1/posts to create or draft posts.  
  - JSON body includes content text, platform, account IDs, publish/draft flag, visibility, hashtags, and optionally media items with image URLs.  
  - Uses Bearer token authentication.  
  - Failures: API auth errors, invalid payloads, network timeouts.

- **Store Generated Post Content (Airtable Upsert)**  
  - Updates Airtable records with post_text, hashtags, CTA, tone, goal, audience, character count, image metadata (prompt, URL, filename, ID), and sets `is_posted=1`.  
  - Uses upsert operation keyed on article ID to avoid duplicates.  
  - Failures: Airtable API errors, schema mismatches.

---

### 3. Summary Table

| Node Name                              | Node Type                          | Functional Role                                  | Input Node(s)                         | Output Node(s)                                | Sticky Note                                                  |
|--------------------------------------|----------------------------------|-------------------------------------------------|-------------------------------------|-----------------------------------------------|--------------------------------------------------------------|
| RSS Feed Trigger: '#to-share-linkedin' | rssFeedReadTrigger                | Fetch LinkedIn-tagged articles from Wallabag RSS | -                                   | Set Platform (LinkedIn)                        | # Step 1️⃣: RSS Aggregator & Content Store                   |
| RSS Feed Trigger: '#to-share-bluesky' | rssFeedReadTrigger                | Fetch Bluesky-tagged articles from Wallabag RSS  | -                                   | Set Platform (Bluesky)                         | # Step 1️⃣: RSS Aggregator & Content Store                   |
| Set Platform (LinkedIn)              | set                              | Mark items with platform = linkedin              | RSS Feed Trigger: '#to-share-linkedin' | Markdown: Convert Article Content              | # Step 1️⃣: RSS Aggregator & Content Store                   |
| Set Platform (Bluesky)               | set                              | Mark items with platform = bluesky                | RSS Feed Trigger: '#to-share-bluesky' | Markdown: Convert Article Content1             | # Step 1️⃣: RSS Aggregator & Content Store                   |
| Markdown: Convert Article Content    | markdown                         | Convert LinkedIn article HTML to Markdown         | Set Platform (LinkedIn)              | Store Article Content (LinkedIn)               | # Step 1️⃣: RSS Aggregator & Content Store                   |
| Markdown: Convert Article Content1   | markdown                         | Convert Bluesky article HTML to Markdown          | Set Platform (Bluesky)               | Store Article Content                           | # Step 1️⃣: RSS Aggregator & Content Store                   |
| Store Article Content (LinkedIn)    | airtable                         | Store LinkedIn article content raw data           | Markdown: Convert Article Content    | -                                             | # Step 1️⃣: RSS Aggregator & Content Store                   |
| Store Article Content                | airtable                         | Store Bluesky article content raw data            | Markdown: Convert Article Content1   | -                                             | # Step 1️⃣: RSS Aggregator & Content Store                   |
| Set Post Schedule                   | scheduleTrigger                  | Periodic trigger to process unpublished articles  | -                                   | Pull New Article                               | # ⚙️ Define Posting Schedule                                  |
| Pull New Article                   | airtable                         | Search Airtable for unpublished articles          | Set Post Schedule                   | Set SMCG Custom Prompt Parameters              | # ⚙️ Configure Airtable Credentials                           |
| Set SMCG Custom Prompt Parameters  | set                              | Set AI content generation parameters               | Pull New Article                   | Check Platform                                 | # ⚙️ Custom AI Generator Parameters & Other Settings          |
| Check Platform                     | switch                          | Route by article platform (linkedin or bluesky)   | Set SMCG Custom Prompt Parameters  | LinkedIn SMCG Prompt, Bluesky SMCG Prompt      | # ⚙️ Configure Feed Credentials & Social Media Platform      |
| LinkedIn SMCG Prompt               | set                              | Compose LinkedIn AI prompt with article & params  | Check Platform (linkedin branch)   | Groq Chat Model                                | # ⚙️ Custom AI Generator Parameters & Other Settings          |
| Bluesky SMCG Prompt                | set                              | Compose Bluesky AI prompt with article & params   | Check Platform (bluesky branch)    | Groq Chat Model1                               | # ⚙️ Custom AI Generator Parameters & Other Settings          |
| Groq Chat Model                   | ai_languageModel                 | Generate LinkedIn post text with Llama 3 model    | LinkedIn SMCG Prompt               | Structured Output Parser                        | # ⚙️ Configure LLM Credentials                                |
| Groq Chat Model1                  | ai_languageModel                 | Generate Bluesky post text with Llama 3 model     | Bluesky SMCG Prompt                | Structured Output Parser1                       | # ⚙️ Configure LLM Credentials                                |
| Structured Output Parser          | ai_outputParser                 | Parse LinkedIn AI raw response into structured JSON | Groq Chat Model                   | LinkedIn SMCG                                   | -                                                            |
| Structured Output Parser1         | ai_outputParser                 | Parse Bluesky AI raw response into structured JSON | Groq Chat Model1                  | Bluesky SMCG                                    | -                                                            |
| LinkedIn SMCG                    | chainLlm                        | Final LinkedIn AI output container                 | Structured Output Parser           | Check For Image Generation Prompt (LinkedIn)  | -                                                            |
| Bluesky SMCG                     | chainLlm                        | Final Bluesky AI output container                   | Structured Output Parser1          | Check For Image Generation Prompt (Bluesky)   | -                                                            |
| Check For Image Generation Prompt (LinkedIn) | if                             | Decide if image generation is required for LinkedIn | LinkedIn SMCG                    | Generate Post Image (LinkedIn) or Post/Draft via GetLate API (No Image, LinkedIn) | -                                                            |
| Check For Image Generation Prompt (Bluesky)  | if                             | Decide if image generation is required for Bluesky  | Bluesky SMCG                     | Generate Post Image (Bluesky) or Post/Draft via GetLate API (No Image, Bluesky) | -                                                            |
| Generate Post Image (LinkedIn)   | httpRequest                    | Generate post image via Hugging Face API            | Check For Image Generation Prompt (LinkedIn) | Convert Image to Base64 (LinkedIn)              | ## ⚙️ Configure Imgbb & Image Generation Service Credentials |
| Generate Post Image (Bluesky)    | httpRequest                    | Generate post image via Hugging Face API            | Check For Image Generation Prompt (Bluesky) | Convert Image to Base64 (Bluesky)               | ## ⚙️ Configure Imgbb & Image Generation Service Credentials |
| Convert Image to Base64 (LinkedIn) | code                           | Convert binary image response to Base64             | Generate Post Image (LinkedIn)    | Store Image on Imgbb (LinkedIn)                 | ## ⚙️ Configure Imgbb & Image Generation Service Credentials |
| Convert Image to Base64 (Bluesky)  | code                           | Convert binary image response to Base64             | Generate Post Image (Bluesky)     | Store Image on Imgbb (Bluesky)                  | ## ⚙️ Configure Imgbb & Image Generation Service Credentials |
| Store Image on Imgbb (LinkedIn)  | httpRequest                    | Upload Base64 image to Imgbb for LinkedIn           | Convert Image to Base64 (LinkedIn) | Post/Draft via GetLate API (LinkedIn)           | ## ⚙️ Configure Imgbb & Image Generation Service Credentials |
| Store Image on Imgbb (Bluesky)   | httpRequest                    | Upload Base64 image to Imgbb for Bluesky            | Convert Image to Base64 (Bluesky)  | Post/Draft via GetLate API (Bluesky)            | ## ⚙️ Configure Imgbb & Image Generation Service Credentials |
| Post/Draft via GetLate API (LinkedIn) | httpRequest                    | Publish or draft LinkedIn post with image           | Store Image on Imgbb (LinkedIn)   | Store Generated Post Content (LinkedIn)         | -                                                            |
| Post/Draft via GetLate API (Bluesky) | httpRequest                    | Publish or draft Bluesky post with image            | Store Image on Imgbb (Bluesky)    | Store Generated Post Content (Bluesky)          | -                                                            |
| Post/Draft via GetLate API (No Image, LinkedIn) | httpRequest                    | Publish or draft LinkedIn post without image        | Check For Image Generation Prompt (LinkedIn) | Store Generated Post Content (No Image, LinkedIn) | -                                                            |
| Post/Draft via GetLate API (No Image, Bluesky) | httpRequest                    | Publish or draft Bluesky post without image         | Check For Image Generation Prompt (Bluesky) | Store Generated Post Content (No Image, Bluesky) | -                                                            |
| Store Generated Post Content (LinkedIn) | airtable                       | Upsert LinkedIn post details and metadata           | Post/Draft via GetLate API (LinkedIn) | -                                             | -                                                            |
| Store Generated Post Content (No Image, LinkedIn) | airtable                       | Upsert LinkedIn post details (no image)              | Post/Draft via GetLate API (No Image, LinkedIn) | -                                          | -                                                            |
| Store Generated Post Content (Bluesky) | airtable                       | Upsert Bluesky post details and metadata            | Post/Draft via GetLate API (Bluesky) | -                                             | -                                                            |
| Store Generated Post Content (No Image, Bluesky) | airtable                       | Upsert Bluesky post details (no image)               | Post/Draft via GetLate API (No Image, Bluesky) | -                                          | -                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Triggers**  
   - Add two `RSS Feed Read Trigger` nodes.  
   - Configure one with Wallabag RSS feed URL filtered by tag `to-share-linkedin`.  
   - Configure the other with Wallabag RSS feed URL filtered by tag `to-share-bluesky`.  
   - Set polling interval to every minute.

2. **Set Platform Tags**  
   - Create two `Set` nodes, named `Set Platform (LinkedIn)` and `Set Platform (Bluesky)`.  
   - Set `platform` attribute to `"linkedin"` and `"bluesky"` respectively.  
   - Connect each RSS Feed Trigger node to the corresponding Set node.

3. **Convert HTML Content to Markdown**  
   - Add two `Markdown` nodes named `Markdown: Convert Article Content` and `Markdown: Convert Article Content1`.  
   - For LinkedIn branch, set input HTML to `$('RSS Feed Trigger: '#to-share-linkedin'').item.json.content`.  
   - For Bluesky branch, set input HTML to `$('RSS Feed Trigger: '#to-share-bluesky'').item.json.content`.

4. **Store Raw Articles in Airtable**  
   - Create two `Airtable` nodes named `Store Article Content (LinkedIn)` and `Store Article Content`.  
   - Configure with Airtable base and table for content storage.  
   - Map article fields: title, author, feed ID, platform, article URL, `content_markdown`, `is_posted=0`.  
   - Connect each Markdown node to corresponding Airtable node.

5. **Create Scheduled Trigger**  
   - Add a `Schedule Trigger` node named `Set Post Schedule`.  
   - Configure to run every minute or preferred interval.

6. **Pull Unpublished Articles from Airtable**  
   - Add an `Airtable` node named `Pull New Article`.  
   - Configure to search for records where `is_posted=0`.  
   - Connect `Set Post Schedule` to `Pull New Article`.

7. **Set AI Content Generation Parameters**  
   - Add a `Set` node `Set SMCG Custom Prompt Parameters`.  
   - Define variables: niche, tone, goal, target audience, custom tags, post_as_draft (boolean), generate_image_bluesky, generate_image_linkedin (booleans), bluesky_account_id, linkedin_account_id.  
   - Connect `Pull New Article` to this node.

8. **Platform Switch Routing**  
   - Add a `Switch` node `Check Platform`.  
   - Configure rules to route based on `platform` field equal to `linkedin` or `bluesky`.  
   - Connect `Set SMCG Custom Prompt Parameters` output to this switch.

9. **Create Platform-Specific AI Prompts**  
   - Add two `Set` nodes: `LinkedIn SMCG Prompt` and `Bluesky SMCG Prompt`.  
   - Configure prompt templates with embedded article content, parameters, and detailed instructions as per the workflow (see node details in 2.2).  
   - Connect `Check Platform` outputs to each respective prompt node.

10. **Configure AI Language Model Nodes**  
    - Add two Langchain `lmChatGroq` nodes named `Groq Chat Model` (LinkedIn) and `Groq Chat Model1` (Bluesky).  
    - Select model `llama-3.3-70b-versatile`.  
    - Add Groq API credentials.  
    - Connect LinkedIn prompt to `Groq Chat Model`, Bluesky prompt to `Groq Chat Model1`.

11. **Add Structured Output Parsers**  
    - Add two `outputParserStructured` nodes named `Structured Output Parser` and `Structured Output Parser1`.  
    - Use provided JSON schema example for parsing AI output.  
    - Connect each Groq Chat Model node to corresponding parser.

12. **Chain LLM Output Nodes**  
    - Add two `chainLlm` nodes named `LinkedIn SMCG` and `Bluesky SMCG`.  
    - Enable output parser usage.  
    - Connect structured output parsers to these nodes.

13. **Add Conditional Image Generation Checks**  
    - Add two `If` nodes named `Check For Image Generation Prompt (LinkedIn)` and `(Bluesky)`.  
    - Check boolean flags `generate_image_linkedin` and `generate_image_bluesky` from prompt parameters.  
    - Connect LinkedIn SMCG to LinkedIn Check node, Bluesky SMCG to Bluesky Check node.

14. **Configure Image Generation HTTP Requests**  
    - Add two `HTTP Request` nodes named `Generate Post Image (LinkedIn)` and `(Bluesky)`.  
    - Set URL to Hugging Face Stable Diffusion API endpoint.  
    - Use POST with JSON body parameter `inputs` set to AI-generated image prompt.  
    - Add Bearer Auth credentials for Hugging Face.  
    - Connect true branch of each Check node to respective image generation node.

15. **Add Code Nodes to Convert Image to Base64**  
    - Add two `Code` nodes `Convert Image to Base64 (LinkedIn)` and `(Bluesky)`.  
    - JavaScript code returns raw response body as `image_data`.  
    - Connect each image generation node to respective code node.

16. **Upload Image to Imgbb**  
    - Add two `HTTP Request` nodes `Store Image on Imgbb (LinkedIn)` and `(Bluesky)`.  
    - Configure multipart form-data POST to Imgbb API with Base64 image.  
    - Include query parameter `name` dynamically set from article metadata.  
    - Use API key via Query Auth credentials.  
    - Connect each code node to respective Imgbb node.

17. **Post or Draft via GetLate API (With Image)**  
    - Add two `HTTP Request` nodes `Post/Draft via GetLate API (LinkedIn)` and `(Bluesky)`.  
    - POST to GetLate API with JSON body including post text, platform, account ID, draft flag, hashtags, and media items referencing Imgbb URL and filename.  
    - Use Bearer Auth for GetLate API key.  
    - Connect Imgbb upload nodes to these posting nodes.

18. **Post or Draft via GetLate API (No Image)**  
    - Add two `HTTP Request` nodes `Post/Draft via GetLate API (No Image, LinkedIn)` and `(No Image, Bluesky)`.  
    - Similar to above but without media items.  
    - Connect false branch of each Check For Image Generation node to these.

19. **Store Generated Post Content in Airtable**  
    - Add four `Airtable` nodes for upserting post data with or without images for each platform.  
    - Map fields including post text, hashtags, CTA, tone, goal, audience, character count, image metadata, `is_posted=1`.  
    - Connect each Post/Draft via GetLate API node to corresponding Airtable node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| ⚙️ **Required Credentials & Services Setup**: Configure Wallabag Access Token & RSS Feed URLs, Airtable API Key, LLM Provider API Key (Groq or compatible), GetLate API Key, Imgbb API Key, and Image Generation Service API Key (Hugging Face or similar). Ensure proper permission scopes and test each connection before executing the workflow. Consider rate limits and billing. | Sticky Note at top-left of workflow canvas.                                                                      |
| # Step 1️⃣: RSS Aggregator & Content Store - Centralized repository for fresh article content tagged for social sharing.                                                                                                                                                                                                                                                                            | Sticky Note near RSS Feed Trigger and Airtable content storage nodes.                                           |
| # Step 2️⃣: AI Content Generation & Publishing - Scheduled processing pulls unpublished articles, generates AI-optimized posts, optionally generates images, and publishes or drafts posts to social media via GetLate API.                                                                                                                                                                          | Sticky Note near scheduling and AI generation nodes.                                                             |
| ⚙️ Posting Schedule Configuration - Control frequency of content publishing and drafting, enabling consistent social media activity with minimal manual input.                                                                                                                                                                                                                                    | Sticky Note near Schedule Trigger node.                                                                          |
| ⚙️ Custom AI Generator Parameters & Other Settings - Define niche, tone, goal, audience, custom hashtags, draft mode, image generation flags, and platform account IDs.                                                                                                                                                                                                                           | Sticky Note near Set SMCG Custom Prompt Parameters node.                                                         |
| ⚙️ Configure Imgbb & Image Generation Service Credentials - Necessary for uploading and generating images used in social posts.                                                                                                                                                                                                                                                                    | Sticky Note near image generation and upload nodes.                                                              |
| ⚙️ Configure Feed Credentials & Social Media Platform - Setup of feed URLs and platform routing logic.                                                                                                                                                                                                                                                                                              | Sticky Note near Switch node and feed-related nodes.                                                             |
| ⚙️ Configure Airtable Credentials - Required for storing and updating article and post data.                                                                                                                                                                                                                                                                                                        | Sticky Note near Airtable nodes.                                                                                   |

---

**Disclaimer:**  
The text and content described originate solely from an automated n8n workflow designed for lawful and public data. All API keys and credentials must be secured and used according to terms of service of the respective platforms. No illegal or offensive content is present or processed.