News Collection & Multi-Platform Publishing with GPT, DALL-E, and Social Media APIs

https://n8nworkflows.xyz/workflows/news-collection---multi-platform-publishing-with-gpt--dall-e--and-social-media-apis-11652


# News Collection & Multi-Platform Publishing with GPT, DALL-E, and Social Media APIs

---

### 1. Workflow Overview

This workflow automates the collection of news content, creation of AI-generated text and images, and multi-platform publishing across WordPress, Facebook, Telegram, LinkedIn, Discord, and email. It leverages AI tools including OpenAI's GPT models and DALL·E for content generation, combined with social media APIs and HTTP requests to publish and distribute the content.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Topic Initialization**: Periodic activation and setting the news topic.
- **1.2 AI-Based Content Generation**: Using search APIs and OpenAI GPT models to generate tailored text content and prompts for image generation.
- **1.3 AI Image Generation and Editing**: Creating and watermarking images via AI and image editing nodes.
- **1.4 WordPress Content Publishing**: Uploading images and publishing posts with proper metadata on WordPress.
- **1.5 Multi-Platform Social Media Posting**: Publishing the created content (image and text) on Facebook, Telegram, LinkedIn (profile and page), and Discord.
- **1.6 Notifications and Logging**: Sending notifications through Telegram, Gmail, and Rapiwa for status updates or logs.
- **1.7 Conditional Flow Control**: Using conditional logic to manage workflow progress steps.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Topic Initialization

**Overview:**  
Starts the workflow on a defined schedule and sets the topic for news collection to be used downstream.

**Nodes Involved:**  
- Schedule  
- Add tropic

**Node Details:**  

- **Schedule**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow based on a time schedule (default unspecified; likely periodic).  
  - Configuration: Default cron or interval trigger with no parameters configured explicitly.  
  - Inputs: None  
  - Outputs: Connected to "Add tropic" node.  
  - Edge Cases: Misconfiguration may cause no trigger; time zone issues may affect scheduling.

- **Add tropic**  
  - Type: Set  
  - Role: Adds or sets a workflow variable containing the topic keyword or phrase for news collection.  
  - Configuration: Sets parameter(s) with the topic string, which is used by subsequent AI content generation.  
  - Inputs: From Schedule node  
  - Outputs: Feeds to "Specific Content Creation" node.  
  - Edge Cases: Empty or invalid topic may cause downstream failures in content generation.

---

#### 2.2 AI-Based Content Generation

**Overview:**  
Uses external search (SerpAPI) and OpenAI language models to create a specific news content draft and generate prompts for image creation.

**Nodes Involved:**  
- Specific Content Creation  
- SerpAPI  
- OpenAI Chat Model  
- OpenAI Chat Model1  
- Generate Prompt  
- Generate an image1  

**Node Details:**  

- **Specific Content Creation**  
  - Type: Langchain Agent  
  - Role: Orchestrates AI tools to create specific news content based on the topic.  
  - Configuration: Likely configured with chain of AI tools and prompt templates.  
  - Inputs: Receives topic from "Add tropic" and search results from "SerpAPI".  
  - Outputs: Sends to "If" node for conditional handling.  
  - Edge Cases: API quota exceeded, malformed input data, or AI response errors.

- **SerpAPI**  
  - Type: Langchain Tool (SerpAPI)  
  - Role: Performs a web search for relevant news content on the topic.  
  - Configuration: Configured with API key and query parameters matching the topic.  
  - Inputs: Triggered before Specific Content Creation via ai_tool connection.  
  - Outputs: Search results passed as ai_tool input to "Specific Content Creation".  
  - Edge Cases: API limits, network errors, or empty search results.

- **OpenAI Chat Model**  
  - Type: Langchain Language Model (OpenAI Chat)  
  - Role: Processes AI text generation requests within the content creation agent.  
  - Configuration: Uses OpenAI GPT model with set temperature and max tokens.  
  - Inputs: Used by "Specific Content Creation" as ai_languageModel.  
  - Outputs: AI-generated text data.  
  - Edge Cases: API key or quota issues, rate limiting, malformed prompts.

- **OpenAI Chat Model1**  
  - Type: Langchain Language Model (OpenAI Chat)  
  - Role: Used for generating prompts for image creation.  
  - Configuration: Similar configuration to OpenAI Chat Model but focused on prompt generation.  
  - Inputs: Feeds "Generate Prompt" node as ai_languageModel.  
  - Outputs: Generates text prompt for image generation.  
  - Edge Cases: Same as above.

- **Generate Prompt**  
  - Type: Langchain Agent  
  - Role: Produces an AI-generated prompt string for image creation based on the content.  
  - Configuration: Likely includes prompt engineering for descriptive image generation.  
  - Inputs: Receives output from "Wait" node and OpenAI Chat Model1.  
  - Outputs: Sends prompt to "Generate an image1".  
  - Edge Cases: Empty or ambiguous prompts causing poor image results.

- **Generate an image1**  
  - Type: Langchain OpenAI (Image Generation)  
  - Role: Uses DALL·E or similar model to generate an image from the prompt.  
  - Configuration: API key configured for OpenAI image generation, parameters for image size and style.  
  - Inputs: From "Generate Prompt".  
  - Outputs: Passes image to "Add Watermark".  
  - Edge Cases: API errors, generation timeout, invalid prompt issues.

---

#### 2.3 AI Image Generation and Editing

**Overview:**  
Adds watermark to the generated image and prepares it for publishing.

**Nodes Involved:**  
- Add Watermark  
- Create WordPress Post  
- HTTP Request1  
- Edit Image  

**Node Details:**  

- **Add Watermark**  
  - Type: Edit Image  
  - Role: Adds a watermark (branding or copyright) to the AI-generated image.  
  - Configuration: Watermark text/image, position, opacity configured.  
  - Inputs: Receives image from "Generate an image1".  
  - Outputs: Feeds to multiple publishing nodes including "Create WordPress Post" and social media posting nodes.  
  - Edge Cases: Image format incompatibility, watermark overlay errors.

- **Create WordPress Post**  
  - Type: WordPress  
  - Role: Creates a new post on WordPress with generated content and image references.  
  - Configuration: Site URL, credentials, post type, status, title, and content fields mapped.  
  - Inputs: Receives watermarked image and possibly text content.  
  - Outputs: Sends to "HTTP Request1" for further media processing.  
  - Edge Cases: Authentication failures, invalid post data, API restrictions.

- **HTTP Request1**  
  - Type: HTTP Request  
  - Role: Makes HTTP calls to upload or manipulate post-related media or metadata.  
  - Configuration: Custom HTTP methods and endpoints for media upload.  
  - Inputs: From "Create WordPress Post".  
  - Outputs: Sends to "Edit Image" for further processing.  
  - Edge Cases: Network timeouts, endpoint errors.

- **Edit Image**  
  - Type: Edit Image  
  - Role: Additional image editing post-processing before final upload.  
  - Configuration: Crop, resize, or optimize image parameters.  
  - Inputs: From "HTTP Request1".  
  - Outputs: Sends to "upload media to wp".  
  - Edge Cases: Image corruption, format errors.

---

#### 2.4 WordPress Content Publishing

**Overview:**  
Uploads media files and sets featured images for the WordPress post.

**Nodes Involved:**  
- upload media to wp  
- upload image to meta data  
- set featured image

**Node Details:**  

- **upload media to wp**  
  - Type: HTTP Request  
  - Role: Uploads media files (images) to WordPress media library via REST API.  
  - Configuration: POST method, authentication headers, media file payload.  
  - Inputs: From "Edit Image".  
  - Outputs: To "upload image to meta data".  
  - Edge Cases: Upload size limits, authentication failure.

- **upload image to meta data**  
  - Type: HTTP Request  
  - Role: Associates uploaded media with post metadata (custom fields or media metadata).  
  - Configuration: PATCH or POST request to update post metadata.  
  - Inputs: From "upload media to wp".  
  - Outputs: To "set featured image".  
  - Edge Cases: Metadata schema mismatch, API errors.

- **set featured image**  
  - Type: HTTP Request  
  - Role: Sets the featured image of the WordPress post to the uploaded media ID.  
  - Configuration: POST or PUT request to WordPress REST API endpoint for featured media.  
  - Inputs: From "upload image to meta data".  
  - Outputs: None (end of WordPress publishing chain).  
  - Edge Cases: Post not found, media ID invalid.

---

#### 2.5 Multi-Platform Social Media Posting

**Overview:**  
Posts the AI-generated content and images to various social media channels.

**Nodes Involved:**  
- Facebook Image post  
- Telegram Image post  
- Create profile image post (LinkedIn)  
- Create page image post (LinkedIn)  
- Post on Discord Channel  

**Node Details:**  

- **Facebook Image post**  
  - Type: Facebook Graph API  
  - Role: Publishes an image post on Facebook page or profile.  
  - Configuration: Access tokens, page ID, image URL, and message text.  
  - Inputs: From "Add Watermark".  
  - Outputs: None.  
  - Edge Cases: Token expiration, permission errors, API rate limits.

- **Telegram Image post**  
  - Type: Telegram  
  - Role: Sends the image and text as a post/message to a Telegram channel or chat.  
  - Configuration: Bot token, chat ID, message format.  
  - Inputs: From "Add Watermark".  
  - Outputs: None.  
  - Edge Cases: Bot blocked, network issues.

- **Create profile image post (LinkedIn)**  
  - Type: LinkedIn  
  - Role: Posts image content to LinkedIn user profile feed.  
  - Configuration: OAuth2 credentials, post text, image upload handling.  
  - Inputs: From "Add Watermark".  
  - Outputs: None.  
  - Edge Cases: OAuth token expiration, permission scopes.

- **Create page image post (LinkedIn)**  
  - Type: LinkedIn  
  - Role: Posts image content to a LinkedIn company page.  
  - Configuration: OAuth2 credentials with page admin rights, post content.  
  - Inputs: From "Add Watermark".  
  - Outputs: None.  
  - Edge Cases: Admin authorization failure.

- **Post on Discord Channel**  
  - Type: Discord  
  - Role: Posts the image and message to a configured Discord channel.  
  - Configuration: Webhook URL, message content, image attachment.  
  - Inputs: From "Add Watermark".  
  - Outputs: To "Do nothing" node for further processing.  
  - Edge Cases: Invalid webhook, Discord rate limiting.

---

#### 2.6 Notifications and Logging

**Overview:**  
Sends status messages and logs through various communication channels for monitoring.

**Nodes Involved:**  
- Do nothing  
- Rapiwa  
- Send a text message (Telegram)  
- Send a message (Gmail)  

**Node Details:**  

- **Do nothing**  
  - Type: NoOp  
  - Role: Placeholder node to branch into multiple notification nodes.  
  - Configuration: None.  
  - Inputs: From "Post on Discord Channel".  
  - Outputs: To "Rapiwa", "Send a text message", and "Send a message".  
  - Edge Cases: None.

- **Rapiwa**  
  - Type: Rapiwa (3rd party notification service)  
  - Role: Sends notifications or logs via Rapiwa API.  
  - Configuration: API key, message content.  
  - Inputs: From "Do nothing".  
  - Outputs: None.  
  - Edge Cases: API failures.

- **Send a text message (Telegram)**  
  - Type: Telegram  
  - Role: Sends status text messages via Telegram bot.  
  - Configuration: Bot token, recipient chat ID, message content.  
  - Inputs: From "Do nothing".  
  - Outputs: None.  
  - Edge Cases: Bot permission issues.

- **Send a message (Gmail)**  
  - Type: Gmail  
  - Role: Sends emails with status or logs.  
  - Configuration: OAuth2 credentials for Gmail, recipient address, subject, and body.  
  - Inputs: From "Do nothing".  
  - Outputs: None.  
  - Edge Cases: OAuth token expiration, sending limits.

---

#### 2.7 Conditional Flow Control

**Overview:**  
Manages workflow branching based on conditions evaluated after content creation.

**Nodes Involved:**  
- If  
- Wait  

**Node Details:**  

- **If**  
  - Type: If Node (Conditional)  
  - Role: Evaluates conditions (likely content presence or validation) to decide workflow continuation.  
  - Configuration: Conditions based on content or success flags.  
  - Inputs: From "Specific Content Creation".  
  - Outputs: If true, proceeds to "Wait"; else, other branches (not detailed).  
  - Edge Cases: Expression evaluation errors, undefined variables.

- **Wait**  
  - Type: Wait Node  
  - Role: Pauses execution for a configured time or until a webhook call triggers continuation.  
  - Configuration: Default or webhook-based wait, with webhook ID.  
  - Inputs: From "If".  
  - Outputs: Proceeds to "Generate Prompt".  
  - Edge Cases: Timeout or webhook not received may stall workflow.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                 | Input Node(s)                         | Output Node(s)                               | Sticky Note                          |
|-------------------------|--------------------------------|------------------------------------------------|-------------------------------------|----------------------------------------------|------------------------------------|
| Schedule                | Schedule Trigger               | Starts the workflow periodically                | None                                | Add tropic                                   |                                    |
| Add tropic              | Set                           | Sets the topic for content generation            | Schedule                           | Specific Content Creation                     |                                    |
| Specific Content Creation| Langchain Agent               | Generates news content using AI and search      | Add tropic, SerpAPI, OpenAI Chat Model | If                                         |                                    |
| SerpAPI                 | Langchain Tool (SerpAPI)      | Performs web search for topic-relevant content  | None (triggered by Specific Content Creation) | Specific Content Creation                   |                                    |
| OpenAI Chat Model       | Langchain LM (OpenAI Chat)    | AI text generation for content                   | Specific Content Creation           | Specific Content Creation                     |                                    |
| OpenAI Chat Model1      | Langchain LM (OpenAI Chat)    | AI text generation for prompt creation           | Generate Prompt                    | Generate Prompt                               |                                    |
| Generate Prompt         | Langchain Agent               | Generates image prompt text                       | Wait, OpenAI Chat Model1            | Generate an image1                            |                                    |
| Generate an image1      | Langchain OpenAI (Image Gen) | Creates AI-generated image                        | Generate Prompt                   | Add Watermark                                 |                                    |
| Add Watermark           | Edit Image                    | Adds watermark to image                           | Generate an image1                 | Create WordPress Post, Facebook Image post, Telegram Image post, Create profile image post (LinkedIn), Create page image post (LinkedIn), Post on Discord Channel |                                    |
| Create WordPress Post   | WordPress                     | Creates a WordPress post                          | Add Watermark                      | HTTP Request1                                |                                    |
| HTTP Request1           | HTTP Request                  | Uploads media or processes post content          | Create WordPress Post              | Edit Image                                   |                                    |
| Edit Image              | Edit Image                    | Additional image processing                       | HTTP Request1                     | upload media to wp                           |                                    |
| upload media to wp      | HTTP Request                  | Uploads media to WordPress library                | Edit Image                       | upload image to meta data                     |                                    |
| upload image to meta data| HTTP Request                  | Updates post metadata with media info             | upload media to wp                | set featured image                           |                                    |
| set featured image      | HTTP Request                  | Sets featured image of the post                    | upload image to meta data         | None                                         |                                    |
| Facebook Image post     | Facebook Graph API            | Posts image and text on Facebook                   | Add Watermark                    | None                                         |                                    |
| Telegram Image post     | Telegram                      | Posts image and text on Telegram                   | Add Watermark                    | None                                         |                                    |
| Create profile image post| LinkedIn                     | Posts image to LinkedIn user profile feed          | Add Watermark                    | None                                         |                                    |
| Create page image post  | LinkedIn                     | Posts image to LinkedIn company page                | Add Watermark                    | None                                         |                                    |
| Post on Discord Channel | Discord                      | Posts image and text on Discord channel             | Add Watermark                    | Do nothing                                   |                                    |
| Do nothing              | NoOp                         | Branching node for notifications                   | Post on Discord Channel          | Rapiwa, Send a text message, Send a message |                                    |
| Rapiwa                  | Rapiwa (3rd party)           | Sends status or log notifications                   | Do nothing                     | None                                         |                                    |
| Send a text message     | Telegram                     | Sends Telegram status message                       | Do nothing                     | None                                         |                                    |
| Send a message          | Gmail                        | Sends email notifications                           | Do nothing                     | None                                         |                                    |
| If                      | If Node                     | Conditional branching based on content validity    | Specific Content Creation          | Wait                                         |                                    |
| Wait                    | Wait                        | Pauses execution until triggered or timeout        | If                              | Generate Prompt                              |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule node:**  
   - Type: Schedule Trigger  
   - Configure to run periodically (e.g., daily at a set time).  
   - No inputs; output connects to "Add tropic".

2. **Add tropic node:**  
   - Type: Set  
   - Configure to set a key-value pair, e.g., `topic: "news topic or keyword"`.  
   - Connect input from Schedule, output to "Specific Content Creation".

3. **SerpAPI node:**  
   - Type: Langchain Tool (SerpAPI)  
   - Configure with SerpAPI API key and query parameter referencing the topic from "Add tropic".  
   - Connect output to "Specific Content Creation" on ai_tool input.

4. **OpenAI Chat Model node:**  
   - Type: Langchain Language Model (OpenAI Chat)  
   - Configure with OpenAI API key, model (e.g., GPT-4), temperature, max tokens.  
   - Connect output to "Specific Content Creation" on ai_languageModel input.

5. **Specific Content Creation node:**  
   - Type: Langchain Agent  
   - Configure to use "SerpAPI" as ai_tool and "OpenAI Chat Model" as ai_languageModel.  
   - Use prompt templates that generate news content based on the topic and search results.  
   - Connect output to "If" node.

6. **If node:**  
   - Type: If  
   - Configure conditions to check if the content generated is valid (e.g., non-empty).  
   - If true, connect output to "Wait" node; else, handle error or terminate.

7. **Wait node:**  
   - Type: Wait  
   - Configure timeout or webhook to pause workflow before proceeding.  
   - Connect output to "Generate Prompt" node.

8. **OpenAI Chat Model1 node:**  
   - Type: Langchain Language Model (OpenAI Chat)  
   - Configure similarly to the first OpenAI Chat Model but for prompt generation.  
   - Connect output to "Generate Prompt" on ai_languageModel input.

9. **Generate Prompt node:**  
   - Type: Langchain Agent  
   - Configure to produce descriptive image prompts from the content.  
   - Connect output to "Generate an image1".

10. **Generate an image1 node:**  
    - Type: Langchain OpenAI (Image generation)  
    - Configure with OpenAI API key for DALL·E or similar.  
    - Configure image parameters (size, style).  
    - Connect output to "Add Watermark".

11. **Add Watermark node:**  
    - Type: Edit Image  
    - Configure watermark text/image, position, opacity.  
    - Connect outputs to:  
      - "Create WordPress Post"  
      - "Facebook Image post"  
      - "Telegram Image post"  
      - "Create profile image post" (LinkedIn)  
      - "Create page image post" (LinkedIn)  
      - "Post on Discord Channel"

12. **Create WordPress Post node:**  
    - Type: WordPress  
    - Configure WordPress site URL, OAuth2 or basic auth credentials.  
    - Map post title and content fields from AI-generated content.  
    - Connect output to "HTTP Request1".

13. **HTTP Request1 node:**  
    - Type: HTTP Request  
    - Configure for WordPress media upload or post processing endpoints.  
    - Connect output to "Edit Image".

14. **Edit Image node:**  
    - Type: Edit Image  
    - Configure any image manipulation needed (resize, crop).  
    - Connect output to "upload media to wp".

15. **upload media to wp node:**  
    - Type: HTTP Request  
    - Configure to upload media files to WordPress media library REST API.  
    - Connect output to "upload image to meta data".

16. **upload image to meta data node:**  
    - Type: HTTP Request  
    - Configure to update post meta with media references.  
    - Connect output to "set featured image".

17. **set featured image node:**  
    - Type: HTTP Request  
    - Configure to set the featured image for the WordPress post.  
    - No further output.

18. **Facebook Image post node:**  
    - Type: Facebook Graph API  
    - Configure Facebook Page access token, page ID, message, and image URL.  
    - Input from "Add Watermark".

19. **Telegram Image post node:**  
    - Type: Telegram  
    - Configure Telegram bot token, target chat ID.  
    - Input from "Add Watermark".

20. **Create profile image post node:**  
    - Type: LinkedIn  
    - Configure OAuth2 credentials for user profile posting.  
    - Input from "Add Watermark".

21. **Create page image post node:**  
    - Type: LinkedIn  
    - Configure OAuth2 credentials for company page posting.  
    - Input from "Add Watermark".

22. **Post on Discord Channel node:**  
    - Type: Discord  
    - Configure Discord webhook URL for channel posting.  
    - Input from "Add Watermark".  
    - Connect output to "Do nothing".

23. **Do nothing node:**  
    - Type: NoOp  
    - Acts as branching point.  
    - Output to "Rapiwa", "Send a text message", and "Send a message".

24. **Rapiwa node:**  
    - Type: Rapiwa Notification  
    - Configure with API key and notification content.  
    - Input from "Do nothing".

25. **Send a text message node:**  
    - Type: Telegram  
    - Configure bot token and chat ID.  
    - Input from "Do nothing".

26. **Send a message node:**  
    - Type: Gmail  
    - Configure OAuth2 credentials, recipient email, subject, and body.  
    - Input from "Do nothing".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| Workflow automates AI content creation and multi-channel publishing including WordPress, Facebook, Telegram, LinkedIn, Discord, and email.             | Workflow general description              |
| Uses Langchain nodes for AI orchestration integrating OpenAI GPT and image generation models with SerpAPI for web search.                             | AI integration details                    |
| WordPress media handling uses HTTP Request nodes for fine-grained API interaction.                                                                     | WordPress API reference                   |
| Multi-platform posting requires proper API credentials and permissions for each social media platform.                                                | Platform API docs for Facebook, LinkedIn, Telegram, Discord |
| Notification nodes send logs and status updates to Telegram, Gmail, and Rapiwa.                                                                         | External notification services            |
| Be aware of API limits and authentication token lifetimes for all integrated services to avoid workflow failure.                                       | Operational considerations                |

---

**Disclaimer:** The provided text is exclusively derived from a workflow automated with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.

---