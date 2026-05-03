Generate and Publish SEO-Optimized Blog Posts to WordPress

https://n8nworkflows.xyz/workflows/generate-and-publish-seo-optimized-blog-posts-to-wordpress-4362


# Generate and Publish SEO-Optimized Blog Posts to WordPress

### 1. Workflow Overview

The **"Generate and Publish SEO-Optimized Blog Posts to WordPress"** workflow automates the entire process of creating SEO-friendly blog posts and publishing them to a WordPress site. It targets content creators, marketers, and agencies aiming to streamline blog post generation with AI assistance and automate publishing, including image generation and notifications.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception**: Accepts triggers from Telegram commands or a scheduled timer to start post generation.
- **1.2 Topic & Title Generation**: Uses AI models to generate blog post topics and SEO-optimized titles.
- **1.3 Article Content Generation**: Generates the main article body using AI language models.
- **1.4 Featured Image Generation & Upload**: Creates a featured image using AI, uploads it to WordPress, and attaches it to the post.
- **1.5 WordPress Post Creation**: Creates a draft WordPress post with the generated content.
- **1.6 Notifications**: Sends notifications to Discord and Telegram channels upon successful post creation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for external triggers to start the workflow. It supports manual Telegram commands and a scheduled automatic trigger every 3 hours.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Scheduled Auto Trigger (Every 3 Hours)1  
  - Check command from telegram (If node)  

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messages.  
    - Config: Uses a webhook ID to listen for Telegram updates.  
    - Inputs: External via Telegram webhook.  
    - Outputs: Passes incoming messages to the conditional check.  
    - Edge Cases: Missing or malformed Telegram messages; Telegram API downtime.

  - **Scheduled Auto Trigger (Every 3 Hours)1**  
    - Type: Time-based trigger.  
    - Config: Fires every 3 hours to initiate automatic post generation.  
    - Inputs: None (time-based).  
    - Outputs: Starts topic generation automatically.  
    - Edge Cases: System downtime leading to missed triggers.

  - **Check command from telegram**  
    - Type: Conditional (If) node.  
    - Config: Checks if Telegram message contains a valid command to start post generation.  
    - Inputs: From Telegram Trigger.  
    - Outputs: Routes to topic generation if condition true.  
    - Edge Cases: Unsupported commands or empty messages.

---

#### 1.2 Topic & Title Generation

- **Overview:**  
  Generates blog post topics and SEO-friendly titles using AI language models based on the trigger input.

- **Nodes Involved:**  
  - Topic Chooser and Title Maker (Chain LLM)  
  - ALT: Metadata Generator (Gemini)1 (LM Chat OpenRouter)  
  - Parse Blog Metadata JSON1 (Output Parser Structured)  

- **Node Details:**  
  - **Topic Chooser and Title Maker**  
    - Type: Chain LLM (Large Language Model) node.  
    - Config: Uses chained prompts to generate blog topics and titles.  
    - Inputs: Triggered by input reception block.  
    - Outputs: Passes generated topics and titles to article content generation.  
    - Edge Cases: AI model response delays or failures; malformed outputs.

  - **ALT: Metadata Generator (Gemini)1**  
    - Type: LM Chat OpenRouter (alternative AI metadata generator).  
    - Config: AI model alternative for metadata generation.  
    - Inputs: Receives data for metadata creation.  
    - Outputs: Passes metadata to output parser.  
    - Edge Cases: Model unavailability or timeouts.

  - **Parse Blog Metadata JSON1**  
    - Type: Output Parser Structured.  
    - Config: Parses structured JSON metadata from AI output for further use.  
    - Inputs: From ALT Metadata Generator.  
    - Outputs: Sends parsed metadata back to Topic Chooser node (loop for refinement).  
    - Edge Cases: Parsing errors due to malformed AI output.

---

#### 1.3 Article Content Generation

- **Overview:**  
  Generates the main body of the blog article using AI language models, supporting multiple alternative AI nodes.

- **Nodes Involved:**  
  - Generate Article Body (Chain LLM)  
  - Article Generator (LM Chat OpenRouter)  
  - Article Generator (alt) (LM Chat OpenAI)  
  - Article Generator (alt2) (LM Chat Google Gemini)  

- **Node Details:**  
  - **Generate Article Body**  
    - Type: Chain LLM.  
    - Config: Primary AI model for article text generation.  
    - Inputs: Receives topics and titles from Topic Chooser.  
    - Outputs: Passes article content to WordPress draft creation.  
    - Edge Cases: API rate limits, incomplete generation, or timeouts.

  - **Article Generator**  
    - Type: LM Chat OpenRouter.  
    - Config: Alternative AI model for content generation.  
    - Inputs/Outputs: Passes output to Generate Article Body node.  
    - Edge Cases: Model availability.

  - **Article Generator (alt)** and **Article Generator (alt2)**  
    - Types: LM Chat OpenAI and LM Chat Google Gemini respectively.  
    - Config: Alternative AI models for robustness or fallback.  
    - Inputs/Outputs: Not directly connected in main flow; possibly for experimentation or backup.

---

#### 1.4 Featured Image Generation & Upload

- **Overview:**  
  Generates a featured image using AI, uploads it to WordPress, and attaches it to the blog post.

- **Nodes Involved:**  
  - Generate Featured Image (OpenAI)  
  - Upload Image to Wordpress (HTTP Request)  
  - Attach Feature Image to Post (HTTP Request)  

- **Node Details:**  
  - **Generate Featured Image (OpenAI)**  
    - Type: OpenAI API node.  
    - Config: Uses OpenAI image generation API to create images based on article metadata.  
    - Inputs: Triggered after WordPress draft post creation.  
    - Outputs: Passes image data to Upload Image node.  
    - Edge Cases: API limits, image generation failures.

  - **Upload Image to Wordpress**  
    - Type: HTTP Request.  
    - Config: Uploads generated image to WordPress media library using REST API.  
    - Inputs: Image data from AI node.  
    - Outputs: Passes media ID to image attachment node.  
    - Edge Cases: Authentication failures, upload errors, API rate limits.

  - **Attach Feature Image to Post**  
    - Type: HTTP Request.  
    - Config: Updates WordPress post with the uploaded image as the featured image.  
    - Inputs: Media ID and post ID.  
    - Outputs: Triggers notification nodes.  
    - Edge Cases: Post not found, permission issues.

---

#### 1.5 WordPress Post Creation

- **Overview:**  
  Creates a draft blog post in WordPress using the generated article content.

- **Nodes Involved:**  
  - Create WP Draft Post1 (WordPress node)  

- **Node Details:**  
  - **Create WP Draft Post1**  
    - Type: WordPress node (native).  
    - Config: Creates a new post in draft status with AI-generated title and content.  
    - Inputs: Receives article content and metadata.  
    - Outputs: Sends post ID to featured image generation node.  
    - Edge Cases: Authentication errors, post creation failures, API limits.

---

#### 1.6 Notifications

- **Overview:**  
  Sends notifications to Discord and Telegram channels about the new blog post publication.

- **Nodes Involved:**  
  - Notify Discord channel (Discord node)  
  - Notify Telegram (Telegram node)  

- **Node Details:**  
  - **Notify Discord channel**  
    - Type: Discord node.  
    - Config: Posts a notification message to a specified Discord channel via webhook.  
    - Inputs: Triggered after featured image attachment.  
    - Outputs: None.  
    - Edge Cases: Webhook URL invalid, rate limits.

  - **Notify Telegram**  
    - Type: Telegram node.  
    - Config: Sends a Telegram message to a configured chat or user.  
    - Inputs: Triggered alongside Discord notification.  
    - Outputs: None.  
    - Edge Cases: Messaging API errors, invalid chat ID.

---

### 3. Summary Table

| Node Name                       | Node Type                           | Functional Role                             | Input Node(s)                 | Output Node(s)                               | Sticky Note                                                                                                      |
|--------------------------------|-----------------------------------|---------------------------------------------|------------------------------|----------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Telegram Trigger                | telegramTrigger                   | Input reception via Telegram command        | External webhook             | Check command from telegram                   |                                                                                                                 |
| Scheduled Auto Trigger (Every 3 Hours)1 | scheduleTrigger                  | Scheduled trigger every 3 hours              | None                         | Topic Chooser and Title Maker                 |                                                                                                                 |
| Check command from telegram     | if                                | Checks if Telegram message is a valid command | Telegram Trigger             | Topic Chooser and Title Maker                 |                                                                                                                 |
| Topic Chooser and Title Maker   | chainLlm                         | Generates blog topics and titles             | Check command from telegram, Scheduled Auto Trigger | Generate Article Body                          |                                                                                                                 |
| ALT: Metadata Generator (Gemini)1 | lmChatOpenRouter                 | Alternative metadata generator                | Not directly used in main flow | Parse Blog Metadata JSON1                      |                                                                                                                 |
| Parse Blog Metadata JSON1       | outputParserStructured            | Parses AI metadata JSON                       | ALT: Metadata Generator (Gemini)1 | Topic Chooser and Title Maker                 |                                                                                                                 |
| Generate Article Body           | chainLlm                         | Generates main article content                | Topic Chooser and Title Maker | Create WP Draft Post1                          |                                                                                                                 |
| Article Generator              | lmChatOpenRouter                 | Alternative AI content generator              | None (starts chain)           | Generate Article Body                          |                                                                                                                 |
| Article Generator (alt)         | lmChatOpenAi                      | Alternative AI content generator              | None                         | None                                          |                                                                                                                 |
| Article Generator (alt2)        | lmChatGoogleGemini                | Alternative AI content generator              | None                         | None                                          |                                                                                                                 |
| Create WP Draft Post1           | wordpress                        | Creates draft WordPress post                   | Generate Article Body         | Generate Featured Image (OpenAI)               |                                                                                                                 |
| Generate Featured Image (OpenAI)| openAi                          | Generates featured image with AI               | Create WP Draft Post1         | Upload Image to Wordpress                      |                                                                                                                 |
| Upload Image to Wordpress       | httpRequest                     | Uploads generated image to WordPress           | Generate Featured Image (OpenAI) | Attach Feature Image to Post                   |                                                                                                                 |
| Attach Feature Image to Post    | httpRequest                     | Attaches uploaded image as featured image      | Upload Image to Wordpress     | Notify Discord channel, Notify Telegram        |                                                                                                                 |
| Notify Discord channel          | discord                         | Sends notification to Discord channel          | Attach Feature Image to Post  | None                                          |                                                                                                                 |
| Notify Telegram                | telegram                        | Sends notification message on Telegram         | Attach Feature Image to Post  | None                                          |                                                                                                                 |
| ⚙️ Setup Guide                 | stickyNote                      | Setup instructions                             | None                         | None                                          |                                                                                                                 |
| 🧠 Overview - What This Workflow Does | stickyNote                      | High-level workflow overview                   | None                         | None                                          |                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Telegram Trigger** node: Configure webhook ID to receive Telegram messages.  
   - Add a **Schedule Trigger** node: Set it to trigger every 3 hours.

2. **Add Conditional Node to Validate Telegram Commands**  
   - Add an **If** node connected to the Telegram Trigger.  
   - Configure to check if the incoming Telegram message contains a specific command to trigger blog post generation.

3. **Add Topic & Title Generation Node**  
   - Add a **Chain LLM** node named "Topic Chooser and Title Maker".  
   - Configure prompts to generate SEO-optimized blog topics and titles based on trigger input.  
   - Connect outputs from the conditional node (if true) and schedule trigger to this node.

4. **(Optional) Add Alternative Metadata Generation**  
   - Add **LM Chat OpenRouter** node for metadata generation (ALT: Metadata Generator).  
   - Connect its output to an **Output Parser Structured** node to parse JSON metadata.  
   - Loop parsed metadata back to the "Topic Chooser and Title Maker" if refinement is needed.

5. **Add Article Content Generation**  
   - Add a **Chain LLM** node named "Generate Article Body".  
   - Connect "Topic Chooser and Title Maker" to this node.  
   - Configure prompts to generate the full blog article body.

6. **Add Alternative Article Generators (optional)**  
   - Add alternative AI nodes such as **LM Chat OpenRouter**, **LM Chat OpenAI**, **LM Chat Google Gemini** for experimentation or fallback.  
   - Connect as needed, but main flow uses "Generate Article Body".

7. **Add WordPress Draft Post Creation**  
   - Add a **WordPress** node named "Create WP Draft Post1".  
   - Configure it to create a draft post with the title and content from the article body node.  
   - Ensure WordPress credentials are set with appropriate permissions.

8. **Add Featured Image Generation**  
   - Add an **OpenAI** node named "Generate Featured Image (OpenAI)".  
   - Configure it to generate images based on the blog post topic or metadata.  
   - Connect it to the WordPress draft post creation node.

9. **Add Image Upload to WordPress**  
   - Add an **HTTP Request** node named "Upload Image to Wordpress".  
   - Configure to upload the generated image to WordPress Media Library via REST API.  
   - Use WordPress credentials for authentication.

10. **Attach Featured Image to Post**  
    - Add an **HTTP Request** node named "Attach Feature Image to Post".  
    - Configure to update the WordPress post with the uploaded image as the featured image.  
    - Input the post ID and media ID from previous nodes.

11. **Add Notification Nodes**  
    - Add a **Discord** node named "Notify Discord channel".  
    - Configure webhook URL and message content to notify about new blog post.  
    - Add a **Telegram** node named "Notify Telegram".  
    - Configure chat ID and message to notify the Telegram channel or user.  
    - Connect both notification nodes to the output of the "Attach Feature Image to Post" node.

12. **Add Sticky Notes** (optional for clarity)  
    - Create sticky notes for Setup Guide and Workflow Overview for internal documentation purposes.

13. **Set Execution Order**  
    - Verify connections reflect the above sequence: triggers → topic/title → article → draft → image generation → image upload → image attach → notifications.

14. **Credentials Setup**  
    - Configure credentials for:  
      - Telegram Bot API  
      - OpenAI API (for text and image generation)  
      - WordPress REST API (with OAuth2 or Application Password)  
      - Discord Webhook URL  

15. **Test Workflow**  
    - Trigger via Telegram command or wait for scheduled trigger.  
    - Check WordPress for draft post and uploaded featured image.  
    - Confirm notifications appear in Discord and Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow leverages multiple AI providers (OpenAI, Google Gemini, OpenRouter) for resilience and experimentation. | Advanced users can swap or combine AI nodes for best results.                                   |
| Discord notification webhook setup is required to send alerts about new posts.                   | Discord developer portal: https://discord.com/developers/applications                           |
| Telegram bot token and chat ID needed for Telegram trigger and notification nodes.                | Telegram Bot API docs: https://core.telegram.org/bots/api                                       |
| WordPress REST API must be enabled and credentials provided with correct permissions.             | WordPress REST API docs: https://developer.wordpress.org/rest-api/                              |
| Image generation uses OpenAI’s DALL·E or similar endpoints, requiring proper API key and quota.  | OpenAI Image API: https://platform.openai.com/docs/api-reference/images                         |
| Sticky notes in the workflow provide setup guidance and high-level overview (empty in JSON but recommended to fill). | n8n sticky notes improve maintainability and onboarding.                                        |

---

**Disclaimer:**  
The provided description and analysis originate exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.