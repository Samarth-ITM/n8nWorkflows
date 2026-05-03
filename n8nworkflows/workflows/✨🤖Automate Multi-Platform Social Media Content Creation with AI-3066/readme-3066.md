✨🤖Automate Multi-Platform Social Media Content Creation with AI

https://n8nworkflows.xyz/workflows/---automate-multi-platform-social-media-content-creation-with-ai-3066


# ✨🤖Automate Multi-Platform Social Media Content Creation with AI

### 1. Workflow Overview

This workflow automates multi-platform social media content creation using AI, targeting Social Media Managers and Digital Marketers who want to streamline content production across 7+ platforms (X/Twitter, Instagram, LinkedIn, Facebook, TikTok, Threads, YouTube Shorts). It reduces manual work by generating platform-optimized posts, managing approvals, and publishing automatically.

The workflow is logically divided into these blocks:

- **1.1 User Input Reception:** Collects user input for social media post topics, keywords, and optional links via a form trigger.
- **1.2 AI Content Generation:** Uses AI language models (OpenAI GPT-4o-mini primarily) and SERP API to generate platform-specific social media content, including hashtags, CTAs, and media suggestions.
- **1.3 Image Creation & Upload:** Optionally generates or uploads images for posts, hosting them via ImgBB.
- **1.4 Content Approval Workflow:** Sends formatted HTML emails for human review and implements a double-approval system using Gmail.
- **1.5 Conditional Publishing:** Upon approval, publishes posts to multiple social media platforms (Instagram, Facebook, X/Twitter, LinkedIn).
- **1.6 Results Aggregation & Notification:** Aggregates publishing results, formats status emails, and sends notifications via Gmail and Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Reception

- **Overview:** Captures user inputs for social media post creation via a web form.
- **Nodes Involved:**  
  - Submit Social Post Details (Form Trigger)
- **Node Details:**

  - **Submit Social Post Details**  
    - Type: Form Trigger  
    - Role: Entry point for user input; collects Topic, optional Keywords/Hashtags, and optional Link.  
    - Configuration: Form fields with placeholders and descriptions to guide user input.  
    - Input: User-submitted form data.  
    - Output: JSON with user inputs for downstream AI content generation.  
    - Edge Cases: Missing required fields (Topic) will block submission; optional fields can be empty.

#### 2.2 AI Content Generation

- **Overview:** Generates platform-specific social media posts using AI models and SERP API for content research.
- **Nodes Involved:**  
  - SerpAPI  
  - gpt-4o LLM (OpenAI GPT-4o)  
  - gpt-4o-mini (OpenAI GPT-4o-mini)  
  - Social Media Content Factory (LangChain Agent)  
  - Social Media Content (Output Parser Structured)  
  - Prepare Content Review Email (LangChain Agent)
- **Node Details:**

  - **SerpAPI**  
    - Type: LangChain Tool (SERP API)  
    - Role: Provides relevant search results to enrich AI content generation.  
    - Configuration: Uses SERP API credentials; no special parameters.  
    - Input: Triggered by AI agent to research topic.  
    - Output: Search results for AI context.  
    - Edge Cases: API quota limits, network errors.

  - **gpt-4o LLM**  
    - Type: LangChain OpenAI LLM (GPT-4o)  
    - Role: Provides advanced language model capabilities for content generation.  
    - Configuration: Temperature 0.4, response format text, OpenAI API credentials.  
    - Input: Prompt from Social Media Content Factory agent.  
    - Output: Text content for posts.  
    - Disabled in this workflow; fallback to gpt-4o-mini.  
    - Edge Cases: API rate limits, timeouts.

  - **gpt-4o-mini**  
    - Type: LangChain OpenAI LLM (GPT-4o-mini)  
    - Role: Primary AI model for content generation and email formatting.  
    - Configuration: Temperature 0.4, response format text, OpenAI API credentials.  
    - Input: Prompts from Social Media Content Factory and Prepare Content Review Email agents.  
    - Output: Generated text content or HTML email content.  
    - Edge Cases: API errors, malformed prompts.

  - **Social Media Content Factory**  
    - Type: LangChain Agent  
    - Role: Core AI agent that crafts platform-specific social media content based on user input and SERP data.  
    - Configuration: Detailed prompt defining style, tone, hashtags, CTAs for 7 platforms; uses system message with current date.  
    - Input: User form data and SERP API results.  
    - Output: JSON structured content for each platform (posts, hashtags, CTAs, media suggestions).  
    - Edge Cases: Incomplete or inconsistent AI output; fallback or validation needed.

  - **Social Media Content**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses AI JSON output into structured data for downstream nodes.  
    - Configuration: JSON schema defining expected properties per platform (post text, hashtags, media suggestions, CTAs).  
    - Input: Raw AI JSON text from Social Media Content Factory.  
    - Output: Parsed JSON object with platform-specific content.  
    - Edge Cases: Parsing errors if AI output deviates from schema.

  - **Prepare Content Review Email**  
    - Type: LangChain Agent  
    - Role: Converts structured social media content JSON into a clean, modern HTML email for approval.  
    - Configuration: Detailed prompt specifying email layout, inline CSS, platform cards, hashtag formatting, emoji preservation.  
    - Input: Parsed social media content JSON.  
    - Output: Raw HTML email content.  
    - Edge Cases: HTML formatting errors, email client compatibility issues.

#### 2.3 Image Creation & Upload

- **Overview:** Handles optional image generation or upload for social media posts, hosting images on ImgBB.
- **Nodes Involved:**  
  - Image Choice (Form) [disabled]  
  - Rename Binary File (Code)  
  - OpenAI (Image generation)  
  - pollinations.ai (Image generation) [disabled]  
  - Save Image to imgbb.com3 (HTTP Request)  
  - Merge (Merge node)
- **Node Details:**

  - **Image Choice**  
    - Type: Form  
    - Role: Allows user to upload an image or request AI-generated image.  
    - Disabled: Not active in current workflow.  
    - Edge Cases: File upload errors, unsupported formats.

  - **Rename Binary File**  
    - Type: Code  
    - Role: Renames uploaded image binary data for consistent downstream processing.  
    - Input: Binary data from Image Choice form.  
    - Output: Renamed binary data.  
    - Edge Cases: Missing binary data.

  - **OpenAI (Image generation)**  
    - Type: LangChain OpenAI Image resource  
    - Role: Generates images based on Instagram caption text.  
    - Input: Instagram caption from Social Media Content Factory output.  
    - Output: Generated image binary data.  
    - Edge Cases: API limits, generation failures.

  - **pollinations.ai**  
    - Type: HTTP Request  
    - Role: Alternative AI image generation service using prompt from content description.  
    - Disabled: Not active currently.  
    - Edge Cases: API downtime, malformed URL.

  - **Save Image to imgbb.com3**  
    - Type: HTTP Request  
    - Role: Uploads image binary data to ImgBB for hosting and returns hosted URL.  
    - Configuration: Multipart form-data POST with API key from environment variable.  
    - Input: Image binary data from OpenAI or user upload.  
    - Output: JSON with hosted image URL.  
    - Edge Cases: API key invalid, upload failures.

  - **Merge**  
    - Type: Merge  
    - Role: Combines image upload results with AI content data for downstream processing.  
    - Configuration: Combine mode, include unpaired items.  
    - Edge Cases: Data mismatch or missing inputs.

#### 2.4 Content Approval Workflow

- **Overview:** Sends formatted HTML emails for human review and implements a double-approval system before publishing.
- **Nodes Involved:**  
  - Prepare Content Review Email (LangChain Agent)  
  - Gmail User for Approval (Gmail node)  
  - Is Content Approved? (If)  
  - Set Default True 2 (Set node)  
  - Approve Final Post Content (Gmail node) [disabled]  
  - Is Approved? (If)
- **Node Details:**

  - **Gmail User for Approval**  
    - Type: Gmail node (sendAndWait)  
    - Role: Sends approval request email with HTML content and waits for double approval.  
    - Configuration: Uses Gmail OAuth2 credentials; recipient email from environment variable; subject includes post name and description; approval type double.  
    - Input: HTML email content from Prepare Content Review Email.  
    - Output: Approval response JSON with approved boolean.  
    - Edge Cases: Email delivery failures, timeout waiting for approval.

  - **Is Content Approved?**  
    - Type: If node  
    - Role: Checks if approval response contains approved=true.  
    - Input: Approval response JSON.  
    - Output: Branches workflow based on approval.  
    - Edge Cases: Missing approval field, malformed response.

  - **Set Default True 2**  
    - Type: Set node  
    - Role: Defaults approval to true if no explicit approval (used to bypass approval in some cases).  
    - Output: JSON with approved=true.  
    - Edge Cases: Logic conflicts if used improperly.

  - **Is Approved?**  
    - Type: If node  
    - Role: Final gate before publishing; checks approved flag.  
    - Output: Proceeds to publishing or stops.  
    - Edge Cases: False negatives blocking publishing.

  - **Approve Final Post Content**  
    - Type: Gmail node (disabled)  
    - Role: Optional additional approval email for final post content.  
    - Disabled: Not active.  
    - Edge Cases: Same as Gmail User for Approval.

#### 2.5 Conditional Publishing

- **Overview:** Publishes approved posts to multiple social media platforms using their APIs.
- **Nodes Involved:**  
  - Instagram Image (HTTP Request to Facebook Graph API)  
  - Instragram Post (Facebook Graph API)  
  - X Post (Twitter API)  
  - Facebook Post (Facebook Graph API)  
  - LinkedIn Post (LinkedIn API)  
  - Merge1, Merge2, Merge Results (Merge nodes)  
  - Instagram Result, X Result, Facebook Result, LinkedIn Result (Set nodes)
- **Node Details:**

  - **Instagram Image**  
    - Type: HTTP Request  
    - Role: Uploads image to Instagram via Facebook Graph API media endpoint.  
    - Configuration: Uses Facebook Graph API credentials; sends image URL and caption from AI output.  
    - Input: Hosted image URL from ImgBB and Instagram caption.  
    - Output: Media object ID for publishing.  
    - Edge Cases: API permission errors, invalid image URL.

  - **Instragram Post**  
    - Type: Facebook Graph API node  
    - Role: Publishes Instagram post using media ID.  
    - Configuration: Uses Facebook Graph API credentials; posts to media_publish edge.  
    - Input: Media ID from Instagram Image node.  
    - Output: Post confirmation JSON.  
    - Edge Cases: API errors, permission issues.

  - **X Post**  
    - Type: Twitter node (OAuth2)  
    - Role: Posts text content to X/Twitter.  
    - Configuration: Uses Twitter OAuth2 credentials; text from AI output.  
    - Output: Post response JSON.  
    - Edge Cases: Token expiration, rate limits.

  - **Facebook Post**  
    - Type: Facebook Graph API node  
    - Role: Posts photo with message to Facebook page.  
    - Configuration: Uses Facebook Graph API credentials; sends binary image data and message with CTA.  
    - Output: Post response JSON.  
    - Edge Cases: Permissions, invalid image data.

  - **LinkedIn Post**  
    - Type: LinkedIn API node  
    - Role: Posts content as organization on LinkedIn.  
    - Configuration: Uses LinkedIn OAuth2 credentials; text includes post, CTA, hashtags; shares image media.  
    - Output: Post response JSON.  
    - Edge Cases: Access token issues, invalid organization ID.

  - **Merge1, Merge2, Merge Results**  
    - Type: Merge nodes  
    - Role: Combine outputs from publishing nodes for result aggregation.  
    - Edge Cases: Missing or partial outputs.

  - **Instagram Result, X Result, Facebook Result, LinkedIn Result**  
    - Type: Set nodes  
    - Role: Store and label individual platform post results for reporting.  
    - Output: JSON with platform-specific post results.

#### 2.6 Results Aggregation & Notification

- **Overview:** Aggregates publishing results, formats summary emails, and sends notifications via Gmail and Telegram.
- **Nodes Involved:**  
  - Merge Results (Merge)  
  - Aggregate (Aggregate)  
  - Prepare Results Email (LangChain Agent)  
  - Prepare Results Message (LangChain Agent)  
  - Gmail Results (Gmail)  
  - Telegram Results (Telegram)
- **Node Details:**

  - **Merge Results**  
    - Type: Merge  
    - Role: Combines all platform post results into a single dataset.  
    - Edge Cases: Partial data from some platforms.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates all merged results into one item for processing.  
    - Edge Cases: Empty input.

  - **Prepare Results Email**  
    - Type: LangChain Agent  
    - Role: Parses post results and generates a professional HTML table summarizing platform statuses and errors.  
    - Input: Aggregated post results JSON.  
    - Output: HTML email content.  
    - Edge Cases: Malformed JSON, missing error details.

  - **Prepare Results Message**  
    - Type: LangChain Agent  
    - Role: Generates a brief textual summary of post results for notifications.  
    - Input: Aggregated post results JSON.  
    - Output: Text summary.  
    - Edge Cases: Same as above.

  - **Gmail Results**  
    - Type: Gmail node  
    - Role: Sends results summary email to user.  
    - Configuration: Uses Gmail OAuth2 credentials; recipient from environment variable.  
    - Edge Cases: Email delivery failures.

  - **Telegram Results**  
    - Type: Telegram node  
    - Role: Sends results summary message to Telegram chat.  
    - Configuration: Uses Telegram API credentials; chat ID from environment variable.  
    - Edge Cases: Bot permissions, network errors.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                     |
|----------------------------|--------------------------------------|----------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Submit Social Post Details  | Form Trigger                         | User input collection                   | -                                | Social Media Content Factory       | # 🧑‍🦱 User Input for Social Media Posts 💡 Unpin default data to get started                   |
| SerpAPI                    | LangChain Tool (SERP API)             | Content research for AI                 | Social Media Content Factory (ai_tool) | Social Media Content Factory       |                                                                                                |
| gpt-4o LLM                 | LangChain OpenAI LLM (GPT-4o)        | Advanced AI content generation (disabled) | Social Media Content Factory (ai_languageModel) | Social Media Content Factory       |                                                                                                |
| gpt-4o-mini                | LangChain OpenAI LLM (GPT-4o-mini)   | Primary AI content generation           | Social Media Content Factory (ai_languageModel), Prepare Content Review Email (ai_languageModel) | Prepare Content Review Email, Prepare Results Email, Prepare Results Message |                                                                                                |
| Social Media Content Factory| LangChain Agent                      | Core AI content creation agent          | Submit Social Post Details, SerpAPI, gpt-4o LLM/mini | Prepare Content Review Email       | # 🛠️ Social Media Content Factory - LinkedIn - Instagram - Facebook - X - TikTok - Threads - YouTube Shorts |
| Social Media Content        | LangChain Output Parser Structured    | Parses AI JSON output                    | Social Media Content Factory      | Social Media Content Factory       |                                                                                                |
| Prepare Content Review Email| LangChain Agent                      | Formats approval email HTML             | Social Media Content Factory      | Gmail User for Approval            | # ✉️ Prepare & Format Approval Email                                                           |
| Gmail User for Approval     | Gmail node (sendAndWait)              | Sends approval email and waits          | Prepare Content Review Email      | Is Content Approved?               | # 👍 Approve Content Before Proceeding                                                         |
| Is Content Approved?        | If node                             | Checks approval status                   | Gmail User for Approval           | OpenAI (if approved)               |                                                                                                |
| OpenAI                     | LangChain OpenAI Image resource       | Generates post image                     | Is Content Approved?              | Save Image to imgbb.com3           | ## Create Post Image                                                                           |
| pollinations.ai            | HTTP Request (disabled)               | Alternative AI image generation          | -                                | -                                 | ## Create Post Image                                                                           |
| Save Image to imgbb.com3   | HTTP Request                        | Uploads image to ImgBB                   | OpenAI                          | Merge, Merge1                     |                                                                                                |
| Merge                     | Merge node                          | Combines image upload and AI content    | Save Image to imgbb.com3, OpenAI | Merge2                           |                                                                                                |
| Merge1                    | Merge node                          | Combines X Post and Instagram Image     | Is Approved?, Instagram Image     | Instagram Image                   |                                                                                                |
| Merge2                    | Merge node                          | Combines Facebook and LinkedIn posts    | Is Approved?, Facebook Post, LinkedIn Post | Facebook Post, LinkedIn Post       |                                                                                                |
| Instagram Image            | HTTP Request (Facebook Graph API)    | Uploads image to Instagram               | Merge1                          | Instragram Post                   |                                                                                                |
| Instragram Post            | Facebook Graph API                   | Publishes Instagram post                 | Instagram Image                  | Instagram Result                  |                                                                                                |
| X Post                    | Twitter node (OAuth2)                 | Posts to X/Twitter                       | Is Approved?                    | X Result                        |                                                                                                |
| Facebook Post              | Facebook Graph API                   | Posts photo and message to Facebook      | Merge2                          | Facebook Result                  |                                                                                                |
| LinkedIn Post              | LinkedIn API                        | Posts content to LinkedIn organization   | Merge2                          | LinkedIn Result                  |                                                                                                |
| Instagram Result           | Set node                           | Stores Instagram post result             | Instragram Post                 | Merge Results                   |                                                                                                |
| X Result                   | Set node                           | Stores X post result                     | X Post                         | Merge Results                   |                                                                                                |
| Facebook Result            | Set node                           | Stores Facebook post result              | Facebook Post                  | Merge Results                   |                                                                                                |
| LinkedIn Result            | Set node                           | Stores LinkedIn post result              | LinkedIn Post                  | Merge Results                   |                                                                                                |
| Merge Results              | Merge node                        | Combines all platform post results       | Instagram Result, X Result, Facebook Result, LinkedIn Result | Aggregate                      |                                                                                                |
| Aggregate                  | Aggregate node                    | Aggregates merged results into one item  | Merge Results                  | Prepare Results Email, Prepare Results Message | # Step 4️⃣: Send Final Results of Social Media Factory                                         |
| Prepare Results Email      | LangChain Agent                  | Generates HTML summary email of results  | Aggregate                     | Gmail Results                   |                                                                                                |
| Prepare Results Message    | LangChain Agent                  | Generates brief text summary of results  | Aggregate                     | Telegram Results               |                                                                                                |
| Gmail Results              | Gmail node                      | Sends results summary email               | Prepare Results Email          | -                             |                                                                                                |
| Telegram Results           | Telegram node                   | Sends results summary message             | Prepare Results Message        | -                             |                                                                                                |
| Set Default True 2         | Set node                       | Defaults approval to true                  | Merge                         | Is Approved?                  |                                                                                                |
| Is Approved?               | If node                       | Final approval check before publishing     | Set Default True 2             | X Post, Merge1, Merge2          |                                                                                                |
| Rename Binary File         | Code node                     | Renames uploaded image binary data         | Image Choice                  | Save Image to imgbb.com3         | ## Alternative User Uploaded Image (optional)                                                 |
| Image Choice               | Form node (disabled)           | Optional user image upload or AI image gen | -                            | Rename Binary File             | ## Alternative User Uploaded Image (optional)                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Name: Submit Social Post Details  
   - Type: Form Trigger  
   - Configure form fields:  
     - Topic (required, text)  
     - Keywords or Hashtags (optional, text)  
     - Link (optional, text)  
   - Set webhook ID (auto-generated).  

2. **Add SERP API Node:**  
   - Type: LangChain Tool (SERP API)  
   - Connect input from Submit Social Post Details (ai_tool).  
   - Configure with valid SERP API credentials.  

3. **Add AI Language Model Node (Primary):**  
   - Type: LangChain OpenAI LLM (gpt-4o-mini)  
   - Connect input from Social Media Content Factory (ai_languageModel).  
   - Configure with OpenAI API credentials, model gpt-4o-mini, temperature 0.4.  

4. **Add Social Media Content Factory Agent Node:**  
   - Type: LangChain Agent  
   - Connect main input from Submit Social Post Details.  
   - Connect ai_tool input from SERP API node.  
   - Connect ai_languageModel input from gpt-4o LLM or gpt-4o-mini node.  
   - Configure prompt with detailed platform-specific instructions (as per overview).  
   - Enable output parser with JSON schema for platform posts.  

5. **Add Social Media Content Output Parser Node:**  
   - Type: LangChain Output Parser Structured  
   - Connect input from Social Media Content Factory output.  
   - Configure JSON schema matching expected AI output structure.  

6. **Add Prepare Content Review Email Agent Node:**  
   - Type: LangChain Agent  
   - Connect input from Social Media Content Factory output.  
   - Configure prompt to generate HTML email with inline CSS, platform cards, hashtags, emojis, CTAs.  

7. **Add Gmail Node for Approval:**  
   - Type: Gmail (sendAndWait)  
   - Connect input from Prepare Content Review Email output.  
   - Configure with Gmail OAuth2 credentials.  
   - Set recipient email from environment variable.  
   - Set subject with post name and description.  
   - Enable double approval.  

8. **Add If Node to Check Approval:**  
   - Name: Is Content Approved?  
   - Connect input from Gmail User for Approval output.  
   - Condition: Check if `data.approved` is true.  

9. **Add OpenAI Image Generation Node:**  
   - Type: LangChain OpenAI Image resource  
   - Connect input from Is Content Approved? (true branch).  
   - Use Instagram caption as prompt.  

10. **Add HTTP Request Node to Upload Image to ImgBB:**  
    - Type: HTTP Request  
    - Connect input from OpenAI image generation output.  
    - Configure POST to https://api.imgbb.com/1/upload with multipart form-data.  
    - Use API key from environment variable.  

11. **Add Merge Node:**  
    - Combine ImgBB upload output with AI content data.  

12. **Add Social Media Platform Publishing Nodes:**  
    - Instagram Image (HTTP Request to Facebook Graph API media endpoint)  
    - Instragram Post (Facebook Graph API media_publish)  
    - X Post (Twitter OAuth2)  
    - Facebook Post (Facebook Graph API photos)  
    - LinkedIn Post (LinkedIn API)  
    - Connect inputs from Merge node and Is Approved? node.  
    - Configure each with respective platform credentials and parameters from AI output.  

13. **Add Set Nodes for Each Platform Result:**  
    - Store and label each platform’s post response.  

14. **Add Merge Results Node:**  
    - Combine all platform results.  

15. **Add Aggregate Node:**  
    - Aggregate merged results into a single item.  

16. **Add Prepare Results Email Agent Node:**  
    - Generate HTML summary table of post statuses and errors.  

17. **Add Prepare Results Message Agent Node:**  
    - Generate brief textual summary of post results.  

18. **Add Gmail Node to Send Results Email:**  
    - Configure with Gmail OAuth2 credentials and recipient email.  

19. **Add Telegram Node to Send Results Message:**  
    - Configure with Telegram API credentials and chat ID.  

20. **Add Set Default True Node:**  
    - Default approval to true for bypass scenarios.  

21. **Add Final Approval If Node:**  
    - Check approved flag before publishing.  

22. **Optional:**  
    - Add Image Choice form node for user-uploaded images.  
    - Add Rename Binary File code node to handle uploaded images.  
    - Add additional approval steps or notifications as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Adjust Social Media Content Factory prompts to suit personal or business requirements.                               | Sticky Note16                                                                                       |
| Update credentials for OpenAI API, Google Studio API, SERP API, Gmail OAuth, Telegram Bot, ImgBB.                    | Sticky Note16                                                                                       |
| Update all social media platform credentials as required.                                                           | Sticky Note17                                                                                       |
| Customize character limits, approval thresholds, and platform-specific parameters as needed.                         | Workflow description                                                                                |
| Use Pollinations.ai or OpenAI for image generation; Pollinations.ai URL format: https://image.pollinations.ai/prompt/ | Sticky Note6                                                                                       |
| Workflow designed for 7+ platforms: X/Twitter, Instagram, LinkedIn, Facebook, TikTok, Threads, YouTube Shorts.       | Sticky Note1                                                                                       |
| Approval workflow uses Gmail with double-approval option and formatted HTML emails for review.                        | Workflow description, Gmail User for Approval node                                                 |
| For detailed API setup and OAuth instructions, refer to n8n docs: https://docs.n8n.io/integrations/builtin/credentials/ | Workflow description                                                                                |

---

This document fully describes the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.