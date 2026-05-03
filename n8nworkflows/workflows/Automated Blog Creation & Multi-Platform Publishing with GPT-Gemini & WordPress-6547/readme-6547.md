Automated Blog Creation & Multi-Platform Publishing with GPT/Gemini & WordPress

https://n8nworkflows.xyz/workflows/automated-blog-creation---multi-platform-publishing-with-gpt-gemini---wordpress-6547


# Automated Blog Creation & Multi-Platform Publishing with GPT/Gemini & WordPress

### 1. Workflow Overview

This workflow automates the creation of blog articles using AI models (OpenAI GPT, Google Gemini), optimizes content, generates featured images, and publishes posts to WordPress and multiple social media platforms (Twitter/X, LinkedIn, Discord, Threads, Bluesky). It supports both scheduled runs every three hours and ad-hoc triggers via Telegram commands.

The workflow is logically divided into these major blocks:

- **1.1 Input Reception & Triggering**: Accepts triggers either via Telegram commands or scheduled intervals to initiate content generation.
- **1.2 Topic Selection & Title Generation**: Uses AI to decide blog topics and generate titles.
- **1.3 Article Content Generation**: Produces blog post bodies using AI models.
- **1.4 WordPress Publishing**: Creates draft posts in WordPress, generates featured images, uploads images, and attaches them to posts.
- **1.5 Content Optimization & Parsing**: Optimizes AI output, parses metadata, and splits content tailored for different social platforms.
- **1.6 Multi-Platform Social Media Publishing**: Routes platform-specific content to Twitter/X, LinkedIn, Discord, Threads, and Bluesky.
- **1.7 Authentication & Token Management**: Manages OAuth tokens and requests for API authentication (e.g., Bluesky).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Triggering

**Overview:**  
This block handles workflow initiation either by receiving Telegram commands or by scheduled automatic triggers every 3 hours.

**Nodes Involved:**  
- Telegram Trigger  
- Scheduled Auto Trigger (Every 3 Hours)1  
- Check command from telegram

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger Node  
  - Role: Listens for incoming Telegram messages to trigger the workflow.  
  - Configuration: Uses a webhook (ID: b211bd9f-8eee-4882-b960-14b6499fee30) to receive Telegram updates.  
  - Inputs: Incoming Telegram messages.  
  - Outputs: Passes data to "Check command from telegram".  
  - Failure modes: Network issues, invalid webhook setup, Telegram API limits.

- **Scheduled Auto Trigger (Every 3 Hours)1**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow every 3 hours without manual input.  
  - Configuration: Cron or fixed interval (3-hour frequency).  
  - Inputs: None; self-triggered.  
  - Outputs: Passes control to "Topic Chooser and Title Maker".  
  - Failure modes: Scheduler service downtime.

- **Check command from telegram**  
  - Type: If Node  
  - Role: Evaluates Telegram commands to determine if they should trigger content generation.  
  - Configuration: Conditions likely check the content of incoming messages.  
  - Inputs: From Telegram Trigger.  
  - Outputs: Routes to "Topic Chooser and Title Maker" if command matches.  
  - Failure modes: Expression errors if message format unexpected; command mismatch.

---

#### 1.2 Topic Selection & Title Generation

**Overview:**  
AI-driven selection of blog topics and generation of article titles.

**Nodes Involved:**  
- Topic Chooser and Title Maker  
- Parse Blog Metadata JSON  
- Google Gemini Chat Model

**Node Details:**  

- **Topic Chooser and Title Maker**  
  - Type: ChainLlm (LangChain LLM Chain)  
  - Role: Uses AI to generate blog topics and titles based on input triggers or commands.  
  - Configuration: Connected downstream to "Generate Article Body".  
  - Inputs: From "Check command from telegram" or "Scheduled Auto Trigger".  
  - Outputs: Provides structured data including proposed topics and titles.  
  - Failure modes: AI model downtime, rate limits, malformed input data.

- **Parse Blog Metadata JSON**  
  - Type: OutputParserStructured (LangChain)  
  - Role: Parses AI-generated JSON metadata for topic and title extraction.  
  - Inputs: From "Topic Chooser and Title Maker".  
  - Outputs: Parsed structured data to "Topic Chooser and Title Maker" for chaining.  
  - Failure modes: Parsing errors if AI output JSON is malformed.

- **Google Gemini Chat Model**  
  - Type: LM Chat Google Gemini  
  - Role: Provides AI chat capability to assist in topic/title generation.  
  - Inputs: Possibly used within the "Topic Chooser and Title Maker" chain.  
  - Outputs: AI-generated content for topic/title.  
  - Failure modes: API authentication, rate limits.

---

#### 1.3 Article Content Generation

**Overview:**  
Generates the main article body text using AI, including alternate models.

**Nodes Involved:**  
- Generate Article Body  
- Article Generator (alt)  
- Article Generator (alt2)  
- Azure OpenAI Chat Model  
- OpenAI Chat Model  
- Generate Featured Image (OpenAI)

**Node Details:**  

- **Generate Article Body**  
  - Type: ChainLlm  
  - Role: Main node generating article content from titles/topics.  
  - Inputs: Output from "Topic Chooser and Title Maker".  
  - Outputs: Sends content to "Create WP Draft Post".  
  - Failure modes: AI latency, malformed prompts.

- **Article Generator (alt)**  
  - Type: LM Chat OpenAI  
  - Role: Alternative AI model to generate article content optionally.  
  - Inputs: Possibly used in parallel or fallback to main article generator.  
  - Outputs: Feeds into "Generate Article Body".  
  - Failure modes: API limits or credentials issues.

- **Article Generator (alt2)**  
  - Type: LM Chat Google Gemini  
  - Role: Another alternative AI content generator.  
  - Inputs/Outputs: No connected outputs in current config; may be placeholder or for manual use.  
  - Failure modes: Same as other AI nodes.

- **Azure OpenAI Chat Model**  
  - Type: LM Chat Azure OpenAI  
  - Role: Alternative or supplementary AI model for content generation.  
  - Inputs/Outputs: No downstream output connected, likely for testing or fallback.  
  - Failure modes: Azure API auth/configuration issues.

- **OpenAI Chat Model**  
  - Type: LM Chat OpenAI  
  - Role: Used primarily for category mapping downstream.  
  - Inputs: From "Aggregate" node.  
  - Outputs: To "Category Mapping".  
  - Failure modes: Same as other OpenAI nodes.

- **Generate Featured Image (OpenAI)**  
  - Type: OpenAI Node  
  - Role: Generates an image prompt or image content for featured blog images.  
  - Inputs: From "Create WP Draft Post".  
  - Outputs: Sends image data to "Upload Image to Wordpress".  
  - Failure modes: Image generation failures, API limits.

---

#### 1.4 WordPress Publishing

**Overview:**  
Creates draft posts in WordPress, uploads and attaches featured images.

**Nodes Involved:**  
- Create WP Draft Post  
- Generate Featured Image (OpenAI)  
- Upload Image to Wordpress  
- Attach Feature Image to Post  
- If (conditional node)

**Node Details:**  

- **Create WP Draft Post**  
  - Type: WordPress Node  
  - Role: Creates a draft post in WordPress with generated content.  
  - Inputs: From "Generate Article Body".  
  - Outputs: Passes control to image generation node.  
  - Failure modes: WordPress API authentication, permissions errors.

- **Generate Featured Image (OpenAI)**  
  - See above.

- **Upload Image to Wordpress**  
  - Type: HTTP Request  
  - Role: Uploads generated image to WordPress media library.  
  - Inputs: From "Generate Featured Image (OpenAI)".  
  - Outputs: Forwards image upload response to "Attach Feature Image to Post".  
  - Failure modes: HTTP errors, media library quota.

- **Attach Feature Image to Post**  
  - Type: HTTP Request  
  - Role: Links uploaded image as featured image to the post.  
  - Inputs: From "Upload Image to Wordpress".  
  - Outputs: To "If" node to verify success.  
  - Failure modes: API errors, incorrect post or media IDs.

- **If**  
  - Type: If Node  
  - Role: Conditional check after image attachment to proceed with next steps or error handling.  
  - Inputs: From "Attach Feature Image to Post".  
  - Outputs: To "AI Content Optimizer" if true.  
  - Failure modes: Expression failure if response data missing.

---

#### 1.5 Content Optimization & Parsing

**Overview:**  
Optimizes the generated article content, parses it, splits it for platform adaptation, then routes content to social platforms.

**Nodes Involved:**  
- AI Content Optimizer  
- Generate Content  
- Output Parser  
- Split Platform Content  
- Route to Platform  
- Split Out

**Node Details:**  

- **AI Content Optimizer**  
  - Type: ChainLlm  
  - Role: Refines and improves generated content using AI.  
  - Inputs: From "If" node after WordPress image attachment.  
  - Outputs: To "Split Platform Content".  
  - Failure modes: AI API issues, content generation errors.

- **Generate Content**  
  - Type: LM Chat OpenRouter  
  - Role: Generates content tailored for social media platforms.  
  - Inputs: AI Content Optimizer output.  
  - Outputs: To "Output Parser".  
  - Failure modes: Connectivity, invalid prompts.

- **Output Parser**  
  - Type: OutputParserStructured  
  - Role: Parses AI output into structured data for social platform posts.  
  - Inputs: From "Generate Content".  
  - Outputs: To "AI Content Optimizer".  
  - Failure modes: Malformed AI output.

- **Split Platform Content**  
  - Type: SplitOut  
  - Role: Splits content into distinct pieces for each social media platform.  
  - Inputs: From "AI Content Optimizer".  
  - Outputs: To "Route to Platform".  
  - Failure modes: Data structure inconsistencies.

- **Route to Platform**  
  - Type: Switch  
  - Role: Routes each piece of content to its target social media platform node.  
  - Inputs: From "Split Platform Content".  
  - Outputs: To various social posting nodes (Twitter/X, Discord, LinkedIn, Threads, Bluesky).  
  - Failure modes: Incorrect routing if data keys mismatch.

---

#### 1.6 Multi-Platform Social Media Publishing

**Overview:**  
Posts content to multiple social platforms using dedicated nodes or HTTP requests.

**Nodes Involved:**  
- Post to X (Twitter)  
- Post to LinkedIn  
- Post to Discord  
- Post to Threads  
- Post to Bluesky  
- Get Access Token  
- Edit Fields  
- Generate text container

**Node Details:**  

- **Post to X**  
  - Type: Twitter Node  
  - Role: Publishes content to Twitter/X account.  
  - Inputs: Routed content from "Route to Platform".  
  - Failure modes: Twitter API rate limits, auth errors.

- **Post to LinkedIn**  
  - Type: LinkedIn Node  
  - Role: Posts updates to LinkedIn.  
  - Inputs: Routed content.  
  - Failure modes: OAuth token expiration, LinkedIn API errors.

- **Post to Discord**  
  - Type: Discord Node  
  - Role: Sends messages to Discord channels using webhook.  
  - Inputs: Routed content.  
  - Failure modes: Webhook URL invalid, message size limits.

- **Post to Threads**  
  - Type: HTTP Request  
  - Role: Publishes to Threads (Meta’s platform) via API.  
  - Inputs: From "Generate text container".  
  - Failure modes: API auth, rate limits.

- **Post to Bluesky**  
  - Type: HTTP Request  
  - Role: Posts content to Bluesky platform.  
  - Inputs: From "Get Access Token".  
  - Failure modes: OAuth token expiry, API errors.

- **Get Access Token**  
  - Type: HTTP Request  
  - Role: Obtains OAuth access token for Bluesky API.  
  - Inputs: From "Edit Fields".  
  - Failure modes: Authentication failure, network issues.

- **Edit Fields**  
  - Type: Set Node  
  - Role: Prepares or modifies data fields required for access token request.  
  - Inputs: From "Generate text container".  
  - Failure modes: Data mapping errors.

- **Generate text container**  
  - Type: HTTP Request  
  - Role: Prepares the textual content container for Threads posting.  
  - Inputs: From "Route to Platform".  
  - Failure modes: HTTP errors, data formatting.

---

#### 1.7 Authentication & Token Management

**Overview:**  
Handles OAuth and token generation specifically for Bluesky API integration.

**Nodes Involved:**  
- Get Access Token  
- Edit Fields

**Node Details:**  

- See above in 1.6.  
- Critical for Bluesky posting success.  
- Failure modes include invalid credentials, token expiration, and network errors.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                           | Input Node(s)                  | Output Node(s)                               | Sticky Note                              |
|-------------------------------|----------------------------------|-----------------------------------------|-------------------------------|----------------------------------------------|-----------------------------------------|
| Telegram Trigger               | Telegram Trigger                  | Trigger on Telegram commands             | -                             | Check command from telegram                   |                                         |
| Scheduled Auto Trigger (Every 3 Hours)1 | Schedule Trigger                 | Scheduled trigger every 3 hours          | -                             | Topic Chooser and Title Maker                 |                                         |
| Check command from telegram    | If                               | Filters Telegram commands                 | Telegram Trigger               | Topic Chooser and Title Maker                  |                                         |
| Topic Chooser and Title Maker  | ChainLlm (LangChain)              | Generate blog topics and titles           | Scheduled Auto Trigger, Check command | Generate Article Body                        |                                         |
| Parse Blog Metadata JSON       | OutputParserStructured (LangChain) | Parse AI metadata JSON                    | Topic Chooser and Title Maker | Topic Chooser and Title Maker                  |                                         |
| Google Gemini Chat Model       | LM Chat Google Gemini             | AI chat for topic/title assistance       | -                             | Topic Chooser and Title Maker                  |                                         |
| Generate Article Body          | ChainLlm                         | Generate article body                      | Topic Chooser and Title Maker | Create WP Draft Post                          |                                         |
| Article Generator (alt)        | LM Chat OpenAI                   | Alternative article generation            | -                             | Generate Article Body                          |                                         |
| Article Generator (alt2)       | LM Chat Google Gemini            | Alternative article generation            | -                             | -                                            |                                         |
| Azure OpenAI Chat Model        | LM Chat Azure OpenAI             | Alternative AI model                      | -                             | -                                            |                                         |
| OpenAI Chat Model              | LM Chat OpenAI                   | Category mapping AI                       | Aggregate                     | Category Mapping                              |                                         |
| Create WP Draft Post           | WordPress                       | Create draft post in WordPress            | Generate Article Body          | Generate Featured Image (OpenAI)              |                                         |
| Generate Featured Image (OpenAI) | OpenAI Node                     | Generate featured image                    | Create WP Draft Post           | Upload Image to Wordpress                      |                                         |
| Upload Image to Wordpress      | HTTP Request                    | Upload image to WordPress media library   | Generate Featured Image (OpenAI) | Attach Feature Image to Post                  |                                         |
| Attach Feature Image to Post   | HTTP Request                    | Attach uploaded image as featured image   | Upload Image to Wordpress      | If                                            |                                         |
| If                           | If                               | Conditional check after image attachment  | Attach Feature Image to Post   | AI Content Optimizer                          |                                         |
| AI Content Optimizer           | ChainLlm                        | Optimize AI-generated content             | If                           | Split Platform Content                         |                                         |
| Generate Content               | LM Chat OpenRouter              | Generate social media content              | AI Content Optimizer           | Output Parser                                 |                                         |
| Output Parser                 | OutputParserStructured (LangChain) | Parse AI social content output             | Generate Content              | AI Content Optimizer                          |                                         |
| Split Platform Content         | SplitOut                       | Split content per social platform          | AI Content Optimizer           | Route to Platform                             |                                         |
| Route to Platform              | Switch                        | Routes content to social media nodes       | Split Platform Content         | Post to X, Post to Discord, Post to LinkedIn, Generate text container |                                         |
| Post to X                    | Twitter Node                   | Post to Twitter/X                         | Route to Platform             | -                                            |                                         |
| Post to LinkedIn              | LinkedIn Node                  | Post to LinkedIn                          | Route to Platform             | -                                            |                                         |
| Post to Discord              | Discord Node                   | Post to Discord                           | Route to Platform             | -                                            |                                         |
| Generate text container        | HTTP Request                   | Prepare content for Threads                 | Route to Platform             | Edit Fields                                   |                                         |
| Edit Fields                   | Set Node                      | Prepare data for Bluesky token request      | Generate text container       | Get Access Token                              |                                         |
| Get Access Token              | HTTP Request                  | Obtain Bluesky OAuth token                 | Edit Fields                  | Post to Bluesky                               |                                         |
| Post to Threads              | HTTP Request                   | Post content to Threads                    | Generate text container       | -                                            |                                         |
| Post to Bluesky               | HTTP Request                   | Post content to Bluesky                    | Get Access Token             | -                                            |                                         |
| Get Category                 | HTTP Request                   | Retrieve blog categories                    | Start                       | Aggregate                                     |                                         |
| Aggregate                   | Aggregate                      | Aggregate category data                      | Get Category                | OpenAI Chat Model                             |                                         |
| Category Mapping            | ChainLlm                      | Map categories using AI                       | Aggregate                    | OpenAI Chat Model                             |                                         |
| Start                       | Manual Trigger                | Manual workflow start                        | -                           | Get Category                                  |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Telegram Trigger** node configured with your Telegram bot credentials and webhook URL.
   - Add a **Schedule Trigger** node set to trigger every 3 hours.

2. **Add Command Check Node:**
   - Add an **If** node ("Check command from telegram") to filter commands from Telegram messages matching the blog creation trigger.

3. **Topic and Title Generation:**
   - Add a **ChainLlm** node named "Topic Chooser and Title Maker" configured with an AI model (e.g., Google Gemini or OpenAI) to generate blog topics and titles.
   - Connect both the Schedule Trigger (directly) and the If node (for Telegram) to this node.

4. **Parse Metadata:**
   - Add an **OutputParserStructured** node ("Parse Blog Metadata JSON") connected to "Topic Chooser and Title Maker" to parse the AI-generated JSON metadata.

5. **Article Body Generation:**
   - Add a **ChainLlm** node ("Generate Article Body") configured with your preferred AI model to generate the article body text.
   - Connect the output of "Topic Chooser and Title Maker" to this node.

6. **Create WordPress Draft Post:**
   - Add a **WordPress** node ("Create WP Draft Post") configured with WordPress credentials (OAuth or Basic Auth).
   - Connect from "Generate Article Body" output.

7. **Generate Featured Image:**
   - Add an **OpenAI** node ("Generate Featured Image (OpenAI)") configured for image generation (DALL·E or similar).
   - Connect from "Create WP Draft Post".

8. **Upload Image to WordPress:**
   - Add an **HTTP Request** node ("Upload Image to Wordpress") that uploads the generated image to WordPress media library.
   - Connect from "Generate Featured Image (OpenAI)".

9. **Attach Feature Image:**
   - Add an **HTTP Request** node ("Attach Feature Image to Post") that attaches uploaded image as featured image to the post.
   - Connect from "Upload Image to Wordpress".

10. **Conditional Check:**
    - Add an **If** node to verify successful image attachment.
    - Connect from "Attach Feature Image to Post".
    - On true, proceed to content optimization.

11. **Content Optimization:**
    - Add a **ChainLlm** node ("AI Content Optimizer") to refine the article content.
    - Connect from the If node.

12. **Social Media Content Generation:**
    - Add a **LM Chat OpenRouter** node ("Generate Content") to generate platform-specific social media posts.
    - Connect from "AI Content Optimizer".

13. **Parse Social Content:**
    - Add an **OutputParserStructured** node ("Output Parser") connected from "Generate Content".

14. **Split Content Per Platform:**
    - Add a **SplitOut** node ("Split Platform Content") connected from "AI Content Optimizer" output.
    - Connect output to a **Switch** node ("Route to Platform").

15. **Route Content to Social Platforms:**
    - Configure the **Switch** node with cases for Twitter/X, LinkedIn, Discord, Threads, and Bluesky.
    - Connect to respective posting nodes:
      - **Twitter Node** ("Post to X") with Twitter OAuth.
      - **LinkedIn Node** ("Post to LinkedIn") with LinkedIn OAuth.
      - **Discord Node** ("Post to Discord") with webhook URL.
      - **HTTP Request Node** ("Post to Threads") configured to Threads API.
      - **HTTP Request Node** ("Post to Bluesky") configured to Bluesky API.

16. **Bluesky Token Management:**
    - Add a **Set** node ("Edit Fields") to prepare token request payload.
    - Add an **HTTP Request** node ("Get Access Token") to retrieve OAuth token.
    - Connect "Edit Fields" to "Get Access Token" and "Get Access Token" to "Post to Bluesky".

17. **Category Mapping and Aggregation (Optional):**
    - Add **HTTP Request** node ("Get Category") to fetch categories.
    - Add **Aggregate** node to process categories.
    - Add **ChainLlm** node ("Category Mapping") for AI-based category mapping.
    - Connect "Start" manual trigger to "Get Category" to begin this process.

18. **Manual Trigger:**
    - Add a **Manual Trigger** node ("Start") to allow manual workflow execution for testing.

19. **Connect all nodes according to the logical flow described in section 2.**

20. **Credentials Setup:**
    - Configure API credentials for Telegram, OpenAI (including Azure if used), Google Gemini, WordPress, Twitter, LinkedIn, Discord, and APIs for Threads and Bluesky.

21. **Test Each Block:**
    - Validate triggers.
    - Confirm AI content generation.
    - Verify post creation and image upload.
    - Test social media posts.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow leverages multiple AI models (OpenAI GPT, Google Gemini) via LangChain integration for flexible content generation. | AI Models Integration                                                                               |
| Scheduling allows automatic blog generation every 3 hours, but can also be triggered manually or via Telegram commands. | Workflow Triggering                                                                                  |
| WordPress media uploads and post creation use REST API calls via HTTP Request nodes for flexibility.                    | WordPress Publishing Details                                                                        |
| Social media posting uses both dedicated nodes (Twitter, LinkedIn, Discord) and HTTP Requests (Threads, Bluesky) due to API availability. | Social Media Integrations                                                                           |
| OAuth token management is essential for Bluesky posting; ensure credentials are correctly configured and refreshed.    | Bluesky API Authentication                                                                          |
| The workflow includes sticky notes for internal documentation and future enhancement hints.                            | Internal Documentation                                                                                |
| For more details on n8n LangChain nodes and AI integration, visit: https://docs.n8n.io/integrations/nodes/n8n-nodes-langchain/ | n8n LangChain Node Documentation                                                                     |
| Telegram bot setup guide: https://core.telegram.org/bots/api                                                               | Telegram API Documentation                                                                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow built with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.