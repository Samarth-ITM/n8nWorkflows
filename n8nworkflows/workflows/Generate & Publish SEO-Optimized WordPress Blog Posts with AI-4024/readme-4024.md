Generate & Publish SEO-Optimized WordPress Blog Posts with AI

https://n8nworkflows.xyz/workflows/generate---publish-seo-optimized-wordpress-blog-posts-with-ai-4024


# Generate & Publish SEO-Optimized WordPress Blog Posts with AI

### 1. Workflow Overview

**Purpose:**  
This workflow, named **BlogBlitz**, automates the generation and publishing of SEO-optimized blog posts to a WordPress site using AI. It significantly reduces manual effort in content creation by leveraging AI for topic selection, article writing, image generation, and publishing.

**Target Use Cases:**  
- Bloggers and content marketers who want to automate content creation.  
- WordPress site owners aiming to maintain fresh, SEO-friendly blog posts without manual writing.  
- Marketing teams seeking efficient content workflows integrated with Telegram and Discord notifications.  

**Logical Blocks:**

- **1.1 Input Triggers:** Receives commands either on schedule (every 3 hours) or via Telegram message to start content generation.  
- **1.2 Topic & Metadata Generation:** Uses OpenRouter and LangChain AI nodes to select a category and generate blog post metadata (title, slug, focus keyphrase, meta description).  
- **1.3 Article Writing:** Utilizes OpenAI GPT-4.1-mini via LangChain to write a long-form (1,500–2,500 words), SEO-optimized article based on metadata.  
- **1.4 Post Drafting on WordPress:** Creates and publishes the blog post draft on WordPress, assigning correct categories and author.  
- **1.5 Featured Image Creation & Upload:** Generates a realistic featured image using OpenAI image API, then uploads it to WordPress media library.  
- **1.6 Set Featured Image:** Updates the WordPress post to assign the uploaded image as the featured image.  
- **1.7 Notifications:** Sends publish alerts with post links to Discord (via webhook) and Telegram (to the triggering user).  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Triggers

- **Overview:**  
  This block listens for workflow triggers, either automatically on a schedule or via an on-demand Telegram command. It ensures the workflow runs regularly or on user request.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Telegram Trigger  
  - If (conditional filter node)

- **Node Details:**  

  - **Schedule Trigger**  
    - *Type & Role:* Triggers the workflow every 3 hours.  
    - *Configuration:* Interval set to 3 hours.  
    - *Input/Output:* No input; outputs trigger event to Topic Chooser node.  
    - *Edge Cases:* Potential timing drifts or missed executions if n8n instance is down.

  - **Telegram Trigger**  
    - *Type & Role:* Listens to Telegram bot messages for commands.  
    - *Configuration:* Listens for "message" updates.  
    - *Input/Output:* No input; outputs Telegram message JSON.  
    - *Edge Cases:* Telegram API downtime, invalid bot token, or user abuse.

  - **If Node**  
    - *Type & Role:* Filters Telegram messages, only allowing "generate" command to proceed.  
    - *Configuration:* Checks if message text equals "generate" (case-sensitive).  
    - *Input:* Telegram Trigger output.  
    - *Output:* Passes on matching messages; blocks others.  
    - *Edge Cases:* Case sensitivity may exclude commands like "Generate" or "GENERATE".

---

#### 2.2 Topic & Metadata Generation

- **Overview:**  
  This block generates the category, article title, URL slug, focus keyphrase, and meta description to seed the article creation process.

- **Nodes Involved:**  
  - Title, category, meta, keyphrase generator (OpenRouter LLM Chat node)  
  - Topic Chooser and Title Maker (LangChain chain LLM node)  
  - Topic Chooser and Title Maker Parser (output parser node)

- **Node Details:**  

  - **Title, category, meta, keyphrase generator**  
    - *Type & Role:* Uses OpenRouter's Gemini-2.5-flash-preview model to generate blog metadata.  
    - *Configuration:* Model preset without additional prompt customization.  
    - *Input:* Triggered by Schedule Trigger or If node.  
    - *Output:* Raw AI response passed to parser.  
    - *Edge Cases:* API limits, unexpected content formats, or latency.

  - **Topic Chooser and Title Maker**  
    - *Type & Role:* LangChain chain LLM node using a strict prompt to generate category, title, slug, focus phrase, and meta description in structured format.  
    - *Configuration:* Prompt instructs selection of one category from five options, generates SEO-friendly metadata, uses current year placeholder if needed.  
    - *Input:* Output from Title generator parser.  
    - *Output:* Structured JSON with fields: category, title, slug, focus_phrase, meta_description.  
    - *Edge Cases:* Prompt misinterpretation, malformed JSON output.

  - **Topic Chooser and Title Maker Parser**  
    - *Type & Role:* Parses AI output text into structured JSON following a fixed schema.  
    - *Configuration:* JSON schema example defines expected fields.  
    - *Input:* AI raw output string.  
    - *Output:* Parsed JSON object with metadata fields.  
    - *Edge Cases:* Parsing failures if AI output deviates from expected format.

---

#### 2.3 Article Writing

- **Overview:**  
  Using the generated metadata, this block produces a long-form, SEO-optimized article in WordPress-compatible plain text format.

- **Nodes Involved:**  
  - Article Generator (OpenAI GPT-4.1-mini chat node)  
  - Basic LLM Chain (LangChain chain LLM node)

- **Node Details:**  

  - **Article Generator**  
    - *Type & Role:* OpenAI GPT-4.1-mini chat node generating the article body.  
    - *Configuration:* Model: gpt-4.1-mini, no extra prompt options.  
    - *Input:* Metadata from Topic Chooser and Title Maker node (title, category, focus phrase).  
    - *Output:* Article text content.  
    - *Edge Cases:* API rate limits, incomplete generation, or hallucinated content.

  - **Basic LLM Chain**  
    - *Type & Role:* LangChain node with detailed prompt instructions for article structure, tone, SEO rules, and WordPress formatting.  
    - *Configuration:* Prompt mandates 1,500–2,500 words, H2/H3 headings, keyword density, no Markdown, no extra commentary, and call-to-action inclusion.  
    - *Input:* Output from Article Generator node.  
    - *Output:* Final article content text for WordPress post.  
    - *Edge Cases:* Formatting errors, prompt misinterpretation.

---

#### 2.4 Post Drafting on WordPress

- **Overview:**  
  This block creates a new WordPress post with the generated article content, correctly categorizes it, and sets author and slug.

- **Nodes Involved:**  
  - Wordpress Post Draft (WordPress API node)

- **Node Details:**  

  - **Wordpress Post Draft**  
    - *Type & Role:* n8n’s WordPress node creating or updating a post via REST API.  
    - *Configuration:*  
      - Title, slug, category assigned dynamically from AI output.  
      - Status set to "publish" for immediate publication.  
      - Author ID hardcoded to 1 (admin).  
      - Content field populated with the article text.  
      - Category IDs mapped as:  
        - Technology → 3  
        - Artificial Intelligence (AI) → 4  
        - Tech Fact → 7  
        - Tech History → 8  
        - Tech Tips → 9  
    - *Credentials:* WordPress API credentials with write access.  
    - *Input:* Article content and metadata.  
    - *Output:* WordPress post JSON including post ID for further processing.  
    - *Edge Cases:* API authentication errors, invalid category IDs, content length limits.

---

#### 2.5 Featured Image Creation & Upload

- **Overview:**  
  Generates a realistic featured image for the blog post and uploads it to the WordPress media library.

- **Nodes Involved:**  
  - OpenAI - Generate Image (OpenAI image generation node)  
  - Upload Image to WP (HTTP Request node)

- **Node Details:**  

  - **OpenAI - Generate Image**  
    - *Type & Role:* Generates a 1024x1024 realistic image based on the article title.  
    - *Configuration:*  
      - Prompt emphasizes lifelike, natural details without fantasy or text in image.  
      - Style set to "vivid".  
      - Output format is binary image data.  
    - *Credentials:* OpenAI API key.  
    - *Input:* Article title from WordPress post draft JSON.  
    - *Output:* Binary image data.  
    - *Edge Cases:* API limits, generation failures, or unsuitable images generated.

  - **Upload Image to WP**  
    - *Type & Role:* HTTP POST request to WordPress media endpoint to upload the image.  
    - *Configuration:*  
      - URL: `/wp-json/wp/v2/media` endpoint.  
      - Binary data sent as PNG, with dynamic filename based on current timestamp.  
      - Headers include content-type and content-disposition.  
      - Uses WordPress API credentials for authentication.  
    - *Input:* Binary image data from OpenAI node.  
    - *Output:* JSON response with uploaded media ID.  
    - *Edge Cases:* Upload failures, auth errors, or file size limits.

---

#### 2.6 Set Featured Image on Post

- **Overview:**  
  Associates the uploaded media as the featured image of the published WordPress post.

- **Nodes Involved:**  
  - Wordpress - Set Featured Image (HTTP Request node)

- **Node Details:**  

  - **Wordpress - Set Featured Image**  
    - *Type & Role:* HTTP POST request to update the post’s featured_media field.  
    - *Configuration:*  
      - URL dynamically includes the WordPress post ID.  
      - Body contains `featured_media` field set to uploaded media ID.  
      - Authenticated with WordPress API credentials.  
    - *Input:* Post ID from Wordpress Post Draft node and media ID from Upload Image node.  
    - *Output:* Updated post JSON.  
    - *Edge Cases:* API update errors, invalid media or post ID, auth failure.

---

#### 2.7 Notifications

- **Overview:**  
  Sends notifications of the published post to Discord via webhook and Telegram to the user who triggered the workflow.

- **Nodes Involved:**  
  - Send to Discord Using Webhook (Discord node)  
  - Telegram (Telegram node)

- **Node Details:**  

  - **Send to Discord Using Webhook**  
    - *Type & Role:* Posts a simple message with the published post link to a Discord channel.  
    - *Configuration:*  
      - Message content includes “Published!” plus post URL.  
      - Uses Discord webhook authentication.  
    - *Input:* Post link from Wordpress - Set Featured Image node.  
    - *Output:* Confirmation of webhook post.  
    - *Edge Cases:* Webhook URL invalid or revoked, Discord downtime.

  - **Telegram**  
    - *Type & Role:* Sends a Telegram message back to the user who requested post generation.  
    - *Configuration:*  
      - Message content mirrors Discord notification.  
      - Chat ID dynamically extracted from Telegram Trigger node.  
    - *Input:* Post link from Wordpress - Set Featured Image node and chat ID from initial Telegram Trigger.  
    - *Output:* Send message confirmation.  
    - *Edge Cases:* Invalid chat ID, Telegram API errors.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                      | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                   |
|--------------------------------|----------------------------------|------------------------------------|----------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                 | Initiates workflow every 3 hours   | None                             | Topic Chooser and Title Maker       | Workflow runs every 3 hours or on Telegram command.                                           |
| Telegram Trigger               | Telegram Trigger                 | Listens for Telegram “generate”    | None                             | If                                 | Workflow triggered on-demand via Telegram “generate” command.                                |
| If                            | If                              | Filters Telegram command for “generate” | Telegram Trigger                | Topic Chooser and Title Maker       | Only allows “generate” command to proceed.                                                   |
| Title, category, meta, keyphrase generator | OpenRouter LM Chat node         | Generates blog metadata             | Topic Chooser and Title Maker Parser | Topic Chooser and Title Maker        |                                                                                               |
| Topic Chooser and Title Maker  | LangChain Chain LLM             | Creates category, title, slug, SEO data | If, Schedule Trigger             | Basic LLM Chain                     |                                                                                               |
| Topic Chooser and Title Maker Parser | LangChain Output Parser         | Parses AI output into structured JSON | Title, category, meta, keyphrase generator | Topic Chooser and Title Maker       |                                                                                               |
| Article Generator              | OpenAI LM Chat                  | Generates long-form article body   | Topic Chooser and Title Maker    | Basic LLM Chain                     |                                                                                               |
| Basic LLM Chain               | LangChain Chain LLM             | Formats article text for WordPress | Article Generator                | Wordpress Post Draft                |                                                                                               |
| Wordpress Post Draft          | WordPress API Node              | Creates and publishes WordPress post | Basic LLM Chain                 | OpenAI - Generate Image            | Assigns categories based on AI-generated category.                                           |
| OpenAI - Generate Image       | OpenAI Image Generation Node    | Generates featured image           | Wordpress Post Draft             | Upload Image to WP                 | Image prompt focuses on realistic, camera-like visuals without text.                         |
| Upload Image to WP            | HTTP Request                   | Uploads generated image to WP media library | OpenAI - Generate Image         | Wordpress - Set Featured Image     |                                                                                               |
| Wordpress - Set Featured Image | HTTP Request                   | Sets media as post’s featured image | Upload Image to WP               | Send to Discord Using Webhook, Telegram |                                                                                               |
| Send to Discord Using Webhook | Discord Node                  | Sends publish notification to Discord | Wordpress - Set Featured Image  | None                             |                                                                                               |
| Telegram                     | Telegram Node                  | Sends publish notification to Telegram user | Wordpress - Set Featured Image  | None                             |                                                                                               |
| Sticky Note                   | Sticky Note                   | Workflow description               | None                             | None                             | This workflow generates and publishes SEO-optimized WordPress blog posts using AI.          |
| Sticky Note1                  | Sticky Note                   | Setup instructions                 | None                             | None                             | Setup credentials for WordPress, OpenAI, OpenRouter, Discord, Telegram APIs.                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger every 3 hours.

2. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot API credentials.  
   - Listen for "message" updates.

3. **Create If node**  
   - Type: If  
   - Condition: `$json.message.text` equals `"generate"` (case-sensitive).  
   - Connect Telegram Trigger → If node.

4. **Create Title, category, meta, keyphrase generator node**  
   - Type: LangChain LM Chat OpenRouter node  
   - Model: `google/gemini-2.5-flash-preview`  
   - Set credentials to OpenRouter API.  
   - Connect If node (true output) and Schedule Trigger output to this node.

5. **Create Topic Chooser and Title Maker Parser node**  
   - Type: LangChain Output Parser Structured  
   - Configure JSON schema with keys: category, title, slug, focus_phrase, meta_description.  
   - Connect output of Title, category, meta, keyphrase generator node to this parser.

6. **Create Topic Chooser and Title Maker node**  
   - Type: LangChain Chain LLM  
   - Use prompt that instructs the AI to:  
     - Randomly select category from five options.  
     - Generate title, slug, focus phrase, meta description.  
   - Connect Topic Chooser and Title Maker Parser output → this node.

7. **Create Article Generator node**  
   - Type: OpenAI LM Chat node  
   - Model: `gpt-4.1-mini`  
   - Connect Topic Chooser and Title Maker node output → Article Generator node.  
   - Credentials: OpenAI API key.

8. **Create Basic LLM Chain node**  
   - Type: LangChain Chain LLM  
   - Use detailed prompt for writing a 1,500–2,500-word SEO-optimized article with WordPress-compatible plain text formatting.  
   - Connect Article Generator → Basic LLM Chain node.

9. **Create Wordpress Post Draft node**  
   - Type: WordPress node  
   - Configure credentials with WordPress API (username, password, URL).  
   - Set parameters:  
     - Title from Topic Chooser and Title Maker output.  
     - Slug from same output.  
     - Status: publish.  
     - Author ID: 1 (admin).  
     - Categories: map AI category to WordPress category IDs (3,4,7,8,9).  
     - Content: from Basic LLM Chain output.  
   - Connect Basic LLM Chain → Wordpress Post Draft.

10. **Create OpenAI - Generate Image node**  
    - Type: LangChain OpenAI Image node  
    - Prompt: Realistic image based on article title, no text in image, style vivid, size 1024x1024.  
    - Credentials: OpenAI API key.  
    - Connect Wordpress Post Draft → OpenAI - Generate Image.

11. **Create Upload Image to WP node**  
    - Type: HTTP Request node  
    - Method: POST  
    - URL: `https://<your-wordpress-site>/wp-json/wp/v2/media`  
    - Send binary data as PNG image.  
    - Headers: Content-Type: image/png, Content-Disposition: attachment with dynamic filename.  
    - Authentication: WordPress API credentials.  
    - Connect OpenAI - Generate Image → Upload Image to WP.

12. **Create Wordpress - Set Featured Image node**  
    - Type: HTTP Request node  
    - Method: POST  
    - URL: `https://<your-wordpress-site>/wp-json/wp/v2/posts/{{postId}}` (use post ID from Wordpress Post Draft)  
    - Body parameters: featured_media set to media ID from Upload Image to WP.  
    - Authentication: WordPress API credentials.  
    - Connect Upload Image to WP → Wordpress - Set Featured Image.

13. **Create Send to Discord Using Webhook node**  
    - Type: Discord node  
    - Configure with your Discord webhook credentials.  
    - Content: “Published!\n{{post link}}”  
    - Connect Wordpress - Set Featured Image → Discord node.

14. **Create Telegram node**  
    - Type: Telegram node  
    - Use Telegram bot credentials.  
    - Send message: “Published!\n\n{{post link}}”  
    - Chat ID: from Telegram Trigger node message.  
    - Connect Wordpress - Set Featured Image → Telegram node.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow requires the `@n8n/n8n-nodes-langchain` package for LangChain nodes.                             | n8n community docs for LangChain nodes                       |
| WordPress categories must exist with IDs matching those in the workflow or update the node’s category mapping. | WordPress Admin > Posts > Categories                          |
| Telegram bot token and Discord webhook URL should be securely stored in n8n credentials.                        | Telegram and Discord official docs                           |
| The workflow produces fully WordPress-compatible plain text articles with hierarchical headings and SEO rules. | See prompt details in Basic LLM Chain node                   |
| To test, send “generate” command message to your Telegram bot or wait for the schedule trigger.                 | Telegram app                                                  |
| Adjust the OpenAI image node parameters to change image style or size as needed.                                | OpenAI image API documentation                                |

---

**Disclaimer:**  
The text provided is exclusively from an n8n automated workflow. It strictly complies with content policies and contains no illegal or offensive material. All data handled is legal and public.