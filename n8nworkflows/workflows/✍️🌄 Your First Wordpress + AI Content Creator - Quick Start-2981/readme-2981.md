✍️🌄 Your First Wordpress + AI Content Creator - Quick Start

https://n8nworkflows.xyz/workflows/-----your-first-wordpress---ai-content-creator---quick-start-2981


# ✍️🌄 Your First Wordpress + AI Content Creator - Quick Start

### 1. Workflow Overview

This workflow automates the creation, rewriting, and publishing of WordPress blog posts optimized for multiple reading levels using AI. It is designed for content creators who want to generate SEO-friendly, structured articles with appropriate HTML formatting, create featured images, and publish drafts on WordPress while maintaining backups and sending notifications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Topic Setup:** Manual trigger and setting the blog topic.
- **1.2 AI Content Generation:** Using AI to create a structured blog post with title and content in JSON format.
- **1.3 Content Validation & Processing:** Separating and validating the title and content, converting HTML to Markdown.
- **1.4 Multi-Reading Level Rewriting:** Rewriting the original content into Grade 9, Grade 5, and Grade 2 reading levels using AI agents.
- **1.5 WordPress Draft Creation & Image Handling:** Creating a draft post on WordPress with the Grade 9 content, generating a featured image via Pollinations.ai, uploading it to WordPress, and setting it as the featured image.
- **1.6 Notifications & Error Handling:** Sending success or error messages via Telegram based on workflow outcomes.
- **1.7 Backup:** Saving the original draft content to Google Drive as a backup.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Topic Setup

- **Overview:** Starts the workflow manually and sets the blog topic for content generation.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Blog Topic (Set Node)  
  - Sticky Note4 (Instructional)  
  - Sticky Note3 (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; triggers workflow on manual test.  
    - Connections: Outputs to "Set Blog Topic".  
    - Edge Cases: None; manual trigger.

  - **Set Blog Topic**  
    - Type: Set  
    - Role: Defines the blog topic string used for AI content generation.  
    - Configuration: Sets a string variable `topic` with a sample value ("Why Nostr is the and coming decentralized network.").  
    - Connections: Outputs to "Create Structured Blog Post".  
    - Edge Cases: Topic must be meaningful; empty or malformed topics may cause poor AI output.

---

#### 2.2 AI Content Generation

- **Overview:** Uses OpenAI GPT-4o-mini model to generate a structured blog post in JSON format containing a title and HTML-formatted content.
- **Nodes Involved:**  
  - Create Structured Blog Post (Langchain Agent)  
  - gpt-4o-mini (OpenAI LM Chat)  
  - Structured Output - JSON (Langchain Output Parser)  
  - Sticky Note7 (Instructional)

- **Node Details:**

  - **gpt-4o-mini**  
    - Type: OpenAI LM Chat Node (Langchain)  
    - Role: Calls OpenAI GPT-4o-mini model to generate text.  
    - Configuration: Response format set to JSON object.  
    - Credentials: OpenAI API credentials required.  
    - Connections: Outputs to "Create Structured Blog Post".  
    - Edge Cases: API rate limits, timeouts, malformed responses.

  - **Create Structured Blog Post**  
    - Type: Langchain Agent  
    - Role: Defines prompt and system instructions to generate a blog post with title and content in JSON format.  
    - Configuration:  
      - Input: Blog topic from "Set Blog Topic".  
      - System message: Detailed instructions for title (SEO-friendly, <10 words, no colon) and content structure (intro, chapters, conclusion) with HTML formatting guidelines.  
      - Output parser enabled to parse JSON with fields "title" and "content".  
    - Connections: Outputs to "Separate Title & Content".  
    - Edge Cases: AI may generate invalid JSON or missing fields.

  - **Structured Output - JSON**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses AI output to enforce JSON schema with "title" and "content".  
    - Configuration: JSON schema example provided.  
    - Connections: Outputs to "Create Structured Blog Post" (feedback loop).  
    - Edge Cases: Parsing failures if AI output deviates from schema.

---

#### 2.3 Content Validation & Processing

- **Overview:** Separates the title and content from AI output, validates their presence, removes redundant HTML tags, converts content to Markdown, and checks for errors.
- **Nodes Involved:**  
  - Separate Title & Content (Code Node)  
  - Tiltle & Content Exist? (If Node)  
  - HTML to Markdown (Markdown Node)  
  - Send Error Message (Telegram Node)  
  - Sticky Note8 (Instructional)  
  - Sticky Note (Instructional)

- **Node Details:**

  - **Separate Title & Content**  
    - Type: Code (JavaScript)  
    - Role: Extracts `title` and `content` from AI output JSON, removes any `<h1>` tags from content, trims whitespace, and validates presence.  
    - Configuration:  
      - Throws errors if input missing or malformed.  
      - Returns error object if validation fails.  
    - Connections: Outputs to "Tiltle & Content Exist?".  
    - Edge Cases: Missing title/content, empty content after processing, malformed input.

  - **Tiltle & Content Exist?**  
    - Type: If  
    - Role: Checks if both title and content are non-empty strings.  
    - Configuration: Conditions check for non-empty `title` and `content`.  
    - Connections:  
      - True branch: to "HTML to Markdown".  
      - False branch: to "Send Error Message".  
    - Edge Cases: Empty or missing fields trigger error path.

  - **HTML to Markdown**  
    - Type: Markdown  
    - Role: Converts HTML content to Markdown format for rewriting.  
    - Configuration: Input is the `content` field.  
    - Connections: Outputs to "Rewrite for Grade 9 Reading Level" and "Google Drive".  
    - Edge Cases: Conversion errors if HTML malformed.

  - **Send Error Message**  
    - Type: Telegram  
    - Role: Sends error notification if title or content missing.  
    - Configuration: Sends message with timestamp to configured Telegram chat ID.  
    - Credentials: Telegram bot credentials required.  
    - Connections: Terminal node on error path.  
    - Edge Cases: Telegram API failures.

---

#### 2.4 Multi-Reading Level Rewriting

- **Overview:** Uses AI agents to rewrite the original content into three reading levels (Grade 9, 5, and 2), each with tailored prompts and formatting.
- **Nodes Involved:**  
  - Rewrite for Grade 9 Reading Level (Langchain Agent)  
  - Rewrite for Grade 5 Reading Level (Langchain Agent)  
  - Rewrite for Grade 2 Reading Level (Langchain Agent)  
  - gpt-4o-mini1, gpt-4o-mini2, gpt-4o-mini3 (OpenAI LM Chat Nodes)  
  - If1, If2, If3 (If Nodes)  
  - Send Error Message1, Send Error Message2, Send Error Message3 (Telegram Nodes)  
  - Sticky Note9, Sticky Note10, Sticky Note11 (Instructional)  
  - Sticky Note2 (Instructional)

- **Node Details:**

  - **Rewrite for Grade 9 Reading Level**  
    - Type: Langchain Agent  
    - Role: Rewrites content at Grade 9 level with metaphors, no title, HTML formatting.  
    - Configuration: Uses prompt with detailed formatting guidelines and original content injected.  
    - Connections: Outputs to "If1".  
    - Edge Cases: AI rewriting failures, empty output.

  - **If1**  
    - Type: If  
    - Role: Checks if rewritten Grade 9 title and content exist.  
    - Connections:  
      - True: to "Create Wordpress Post".  
      - False: to "Send Error Message1".  
    - Edge Cases: Missing rewritten content triggers error.

  - **Rewrite for Grade 5 Reading Level**  
    - Type: Langchain Agent  
    - Role: Rewrites content at Grade 5 level with light humor and metaphors, HTML formatting.  
    - Configuration: Similar prompt structure as Grade 9 but adjusted for Grade 5.  
    - Connections: Outputs to "If2".  
    - Edge Cases: Same as above.

  - **If2**  
    - Type: If  
    - Role: Validates Grade 5 rewritten content presence.  
    - Connections:  
      - True: no further nodes (end of branch).  
      - False: to "Send Error Message2".  

  - **Rewrite for Grade 2 Reading Level**  
    - Type: Langchain Agent  
    - Role: Rewrites content at Grade 2 level with simple metaphors and child-friendly explanations, HTML formatting.  
    - Configuration: Similar prompt structure as above.  
    - Connections: Outputs to "If3".  
    - Edge Cases: Same as above.

  - **If3**  
    - Type: If  
    - Role: Validates Grade 2 rewritten content presence.  
    - Connections:  
      - True: no further nodes (end of branch).  
      - False: to "Send Error Message3".  

  - **Send Error Message1, Send Error Message2, Send Error Message3**  
    - Type: Telegram  
    - Role: Send error notifications for missing rewritten content at respective reading levels.  
    - Configuration: Sends error message with timestamp to Telegram chat.  
    - Credentials: Telegram API credentials required.

---

#### 2.5 WordPress Draft Creation & Image Handling

- **Overview:** Creates a draft WordPress post using the Grade 9 rewritten content, generates a featured image via Pollinations.ai, uploads the image to WordPress, and sets it as the post’s featured image.
- **Nodes Involved:**  
  - Create Wordpress Post (WordPress Node)  
  - pollinations.ai (HTTP Request)  
  - Upload Image to Wordpress (HTTP Request)  
  - Set Image on Wordpress Post (HTTP Request)  
  - Send Success Message (Telegram Node)  
  - Sticky Note (Instructional)  
  - Sticky Note1 (Instructional)  
  - Sticky Note4 (Instructional)

- **Node Details:**

  - **Create Wordpress Post**  
    - Type: WordPress Node  
    - Role: Creates a draft post on WordPress with title and Grade 9 content.  
    - Configuration:  
      - Title from separated title node.  
      - Content from Grade 9 rewritten output.  
      - Status set to "draft".  
    - Credentials: WordPress API credentials required.  
    - Connections: Outputs to "pollinations.ai".  
    - Edge Cases: WordPress API errors, authentication failures.

  - **pollinations.ai**  
    - Type: HTTP Request  
    - Role: Generates a featured image based on the blog post title.  
    - Configuration:  
      - URL dynamically constructed with title for image prompt.  
      - No text added to image; vibrant style requested.  
    - Connections: Outputs binary image data to "Upload Image to Wordpress".  
    - Edge Cases: HTTP errors, image generation failures.

  - **Upload Image to Wordpress**  
    - Type: HTTP Request  
    - Role: Uploads the generated image as media to WordPress.  
    - Configuration:  
      - POST to WordPress media endpoint.  
      - Sends binary image data with appropriate headers including filename.  
      - Uses WordPress credentials for authentication.  
    - Connections: Outputs media ID to "Set Image on Wordpress Post".  
    - Edge Cases: Upload failures, authentication errors.

  - **Set Image on Wordpress Post**  
    - Type: HTTP Request  
    - Role: Sets the uploaded image as the featured image of the WordPress post.  
    - Configuration:  
      - POST to WordPress post endpoint with query parameter `featured_media` set to uploaded media ID.  
      - Uses WordPress credentials.  
    - Connections: Outputs to "Send Success Message".  
    - Edge Cases: API errors, invalid media ID.

  - **Send Success Message**  
    - Type: Telegram  
    - Role: Sends success notification with timestamp to Telegram chat.  
    - Credentials: Telegram API credentials required.  
    - Edge Cases: Telegram API failures.

---

#### 2.6 Notifications & Error Handling

- **Overview:** Sends Telegram messages to notify success or failure at various stages.
- **Nodes Involved:**  
  - Send Error Message (multiple instances)  
  - Send Success Message

- **Node Details:**  
  - All Telegram nodes send messages to a configured chat ID using bot credentials.  
  - Error messages include timestamp and context of failure (missing title/content or rewriting failure).  
  - Success message confirms blog post creation with timestamp.

---

#### 2.7 Backup

- **Overview:** Saves the original AI-generated draft content to Google Drive as a text file for backup.
- **Nodes Involved:**  
  - Google Drive (Google Drive Node)  
  - Sticky Note8 (Instructional)

- **Node Details:**

  - **Google Drive**  
    - Type: Google Drive Node  
    - Role: Creates a text file in Google Drive with the blog post title as filename and content as file content.  
    - Configuration:  
      - Drive: "My Drive"  
      - Folder: Root  
      - Operation: createFromText  
    - Credentials: Google Drive OAuth2 credentials required.  
    - Connections: Triggered after "HTML to Markdown".  
    - Edge Cases: API quota limits, authentication errors.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                   |
|-----------------------------|----------------------------------|----------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                   | Workflow manual start                        | -                             | Set Blog Topic                  | ## 👍 Start Here                                                                             |
| Set Blog Topic              | Set                              | Defines blog topic string                     | When clicking ‘Test workflow’  | Create Structured Blog Post     | ## 🌟 Set Blog Topic                                                                         |
| gpt-4o-mini                 | OpenAI LM Chat (Langchain)        | AI call for content generation                | -                             | Create Structured Blog Post     | ## Create Blog Post Refer to this workflow for help getting setup with DeepSeek https://n8n.io/workflows/2777-deepseek-v3-chat-and-r1-reasoning-quick-start/ |
| Create Structured Blog Post | Langchain Agent                  | Generates structured blog post JSON           | Set Blog Topic, gpt-4o-mini    | Separate Title & Content        |                                                                                              |
| Structured Output - JSON    | Langchain Output Parser Structured| Parses AI output JSON                          | Create Structured Blog Post    | Create Structured Blog Post     |                                                                                              |
| Separate Title & Content    | Code                             | Extracts and validates title and content      | Create Structured Blog Post    | Tiltle & Content Exist?         |                                                                                              |
| Tiltle & Content Exist?     | If                               | Checks presence of title and content          | Separate Title & Content       | HTML to Markdown, Send Error Message |                                                                                              |
| HTML to Markdown            | Markdown                         | Converts HTML content to Markdown              | Tiltle & Content Exist?        | Rewrite for Grade 9 Reading Level, Google Drive | ## Save Draft Blog Post to Google Drive                                                     |
| Google Drive                | Google Drive                     | Saves draft content as backup                  | HTML to Markdown              | -                              |                                                                                              |
| Rewrite for Grade 9 Reading Level | Langchain Agent              | Rewrites content at Grade 9 reading level      | HTML to Markdown, gpt-4o-mini1 | If1                           | ## Rewrite for Grade 9 Reading Level Update Agent prompt as required                         |
| If1                        | If                               | Validates Grade 9 rewritten content            | Rewrite for Grade 9 Reading Level | Create Wordpress Post, Send Error Message1 |                                                                                              |
| Create Wordpress Post       | WordPress                        | Creates draft WordPress post                    | If1                           | pollinations.ai                | ## Create Wordpress Post and Add New Image https://docs.n8n.io/integrations/builtin/credentials/wordpress/ |
| pollinations.ai             | HTTP Request                    | Generates featured image based on title        | Create Wordpress Post          | Upload Image to Wordpress       | ## Create Post Image https://pollinations.ai/ https://image.pollinations.ai/prompt/[your image description] |
| Upload Image to Wordpress   | HTTP Request                    | Uploads image to WordPress media library        | pollinations.ai                | Set Image on Wordpress Post     |                                                                                              |
| Set Image on Wordpress Post | HTTP Request                    | Sets uploaded image as featured image on post  | Upload Image to Wordpress      | Send Success Message            |                                                                                              |
| Send Success Message        | Telegram                        | Sends success notification                      | Set Image on Wordpress Post    | -                              |                                                                                              |
| Rewrite for Grade 5 Reading Level | Langchain Agent              | Rewrites content at Grade 5 reading level       | gpt-4o-mini2                  | If2                           | ## Rewrite for Grade 5 Reading Level Update Agent prompt as required                         |
| If2                        | If                               | Validates Grade 5 rewritten content            | Rewrite for Grade 5 Reading Level | Send Error Message2           |                                                                                              |
| Rewrite for Grade 2 Reading Level | Langchain Agent              | Rewrites content at Grade 2 reading level       | gpt-4o-mini3                  | If3                           | ## Rewrite for Grade 2 Reading Level Update Agent prompt as required                         |
| If3                        | If                               | Validates Grade 2 rewritten content            | Rewrite for Grade 2 Reading Level | Send Error Message3           |                                                                                              |
| Send Error Message          | Telegram                        | Sends error notification for missing title/content | Tiltle & Content Exist? (false branch) | -                        |                                                                                              |
| Send Error Message1         | Telegram                        | Sends error notification for Grade 9 rewrite failure | If1 (false branch)           | -                              |                                                                                              |
| Send Error Message2         | Telegram                        | Sends error notification for Grade 5 rewrite failure | If2 (false branch)           | -                              |                                                                                              |
| Send Error Message3         | Telegram                        | Sends error notification for Grade 2 rewrite failure | If3 (false branch)           | -                              |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create Set Node "Set Blog Topic"**  
   - Assign a string variable `topic` with a sample blog topic (e.g., "Why Nostr is the and coming decentralized network.")  
   - Connect Manual Trigger output to this node.

3. **Create OpenAI LM Chat Node "gpt-4o-mini"**  
   - Model: GPT-4o-mini  
   - Response Format: JSON object  
   - Credentials: Configure OpenAI API credentials.  
   - No direct input; will be connected later.

4. **Create Langchain Agent Node "Create Structured Blog Post"**  
   - Input: Use expression `={{ $json.topic }}` from "Set Blog Topic".  
   - System Message: Provide detailed instructions for generating a blog post with title and content in JSON format, including HTML formatting guidelines as described in the overview.  
   - Enable output parser with JSON schema for fields "title" and "content".  
   - Connect "Set Blog Topic" and "gpt-4o-mini" outputs to this node.

5. **Create Langchain Output Parser Node "Structured Output - JSON"**  
   - JSON schema example: `{ "title": "title", "content": "content" }`  
   - Connect output of "Create Structured Blog Post" to this parser.

6. **Create Code Node "Separate Title & Content"**  
   - JavaScript code to:  
     - Extract `title` and `content` from AI output.  
     - Remove `<h1>` tags from content.  
     - Validate presence of title and content.  
     - Return error object if validation fails.  
   - Connect output of "Structured Output - JSON" to this node.

7. **Create If Node "Tiltle & Content Exist?"**  
   - Condition: Check if `title` and `content` are non-empty strings.  
   - True branch: Connect to "HTML to Markdown".  
   - False branch: Connect to "Send Error Message".

8. **Create Markdown Node "HTML to Markdown"**  
   - Input: `content` from "Separate Title & Content".  
   - Converts HTML content to Markdown.  
   - Connect true branch of "Tiltle & Content Exist?" to this node.

9. **Create Google Drive Node "Google Drive"**  
   - Operation: createFromText  
   - Filename: Use `title` from "Separate Title & Content".  
   - Content: Use Markdown content from "HTML to Markdown".  
   - Drive: "My Drive", Folder: Root  
   - Credentials: Configure Google Drive OAuth2 credentials.  
   - Connect output of "HTML to Markdown" to this node.

10. **Create Langchain Agent Nodes for Rewriting:**  
    - "Rewrite for Grade 9 Reading Level"  
      - Prompt: Rewrite content at Grade 9 level with metaphors, no title, HTML formatting.  
      - Input: Markdown content from "HTML to Markdown".  
      - Credentials: OpenAI API.  
      - Connect output of "HTML to Markdown" to this node.

    - "Rewrite for Grade 5 Reading Level"  
      - Prompt: Rewrite content at Grade 5 level with light humor and metaphors, HTML formatting.  
      - Input: Markdown content.  
      - Credentials: OpenAI API.

    - "Rewrite for Grade 2 Reading Level"  
      - Prompt: Rewrite content at Grade 2 level with simple metaphors, child-friendly, HTML formatting.  
      - Input: Markdown content.  
      - Credentials: OpenAI API.

11. **Create If Nodes for Validation of Rewrites:**  
    - "If1" for Grade 9 rewrite: Check non-empty title and content.  
      - True: Connect to "Create Wordpress Post".  
      - False: Connect to "Send Error Message1".

    - "If2" for Grade 5 rewrite: Check non-empty title and content.  
      - True: No further action needed.  
      - False: Connect to "Send Error Message2".

    - "If3" for Grade 2 rewrite: Check non-empty title and content.  
      - True: No further action needed.  
      - False: Connect to "Send Error Message3".

12. **Create WordPress Node "Create Wordpress Post"**  
    - Title: Use `title` from "Separate Title & Content".  
    - Content: Use Grade 9 rewritten content from "Rewrite for Grade 9 Reading Level".  
    - Status: draft  
    - Credentials: Configure WordPress API credentials.  
    - Connect true branch of "If1" to this node.

13. **Create HTTP Request Node "pollinations.ai"**  
    - Method: GET  
    - URL: `https://image.pollinations.ai/prompt/{{ $('Separate Title & Content').item.json.title }} and Avoid adding text and keep the image vibrant.`  
    - Connect output of "Create Wordpress Post" to this node.

14. **Create HTTP Request Node "Upload Image to Wordpress"**  
    - Method: POST  
    - URL: `https://[YOUR-WORDPRESS-SITE.com]/wp-json/wp/v2/media`  
    - Authentication: Use WordPress credentials.  
    - Headers: `Content-Disposition` with filename `cover-image-{{ $('Create Wordpress Post').item.json.id }}.jpeg`  
    - Content-Type: binaryData  
    - Input Data Field Name: data (binary image from pollinations.ai)  
    - Connect output of "pollinations.ai" to this node.

15. **Create HTTP Request Node "Set Image on Wordpress Post"**  
    - Method: POST  
    - URL: `https://[YOUR-WORDPRESS-SITE.com]/wp-json/wp/v2/posts/{{ $('Create Wordpress Post').item.json.id }}`  
    - Query Parameter: `featured_media` set to uploaded media ID from previous node.  
    - Authentication: WordPress credentials.  
    - Connect output of "Upload Image to Wordpress" to this node.

16. **Create Telegram Node "Send Success Message"**  
    - Text: `Success! Your blog post was created at {{ $now }}`  
    - Chat ID: Use environment variable for Telegram chat ID.  
    - Credentials: Telegram bot credentials.  
    - Connect output of "Set Image on Wordpress Post" to this node.

17. **Create Telegram Nodes for Error Messages:**  
    - "Send Error Message" for missing title/content after separation.  
    - "Send Error Message1" for Grade 9 rewrite failure.  
    - "Send Error Message2" for Grade 5 rewrite failure.  
    - "Send Error Message3" for Grade 2 rewrite failure.  
    - Configure messages with timestamps and connect respective error branches.

18. **Add Sticky Notes** at appropriate positions with instructional content as per original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow automates multi-reading-level WordPress blog content creation with AI, image generation, and notifications.                      | Workflow description embedded in Sticky Note5.                                                             |
| Refer to DeepSeek workflow for advanced AI content generation setup: https://n8n.io/workflows/2777-deepseek-v3-chat-and-r1-reasoning-quick-start/ | Sticky Note7                                                                                                 |
| WordPress integration documentation: https://docs.n8n.io/integrations/builtin/credentials/wordpress/                                        | Sticky Note (near WordPress nodes)                                                                           |
| Pollinations.ai image generation service: https://pollinations.ai/ and https://image.pollinations.ai/prompt/[your image description]       | Sticky Note1                                                                                                 |
| Telegram bot setup required for notifications; use environment variable `TELEGRAM_CHAT_ID` for chat ID configuration.                      | Telegram nodes configuration                                                                                 |
| OpenAI API credentials required for all AI nodes; ensure rate limits and usage quotas are managed.                                          | All Langchain and OpenAI LM Chat nodes                                                                       |
| Google Drive OAuth2 credentials required for backup node.                                                                                   | Google Drive node                                                                                             |

---

This document provides a complete, structured reference for understanding, reproducing, and modifying the "✍️🌄 Your First Wordpress + AI Content Creator - Quick Start" workflow. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and AI agents to work effectively with this workflow.