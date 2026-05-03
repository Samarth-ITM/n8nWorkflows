WhatsApp-to-Social-Media Content Generation with GPT-4 & Approval Workflow

https://n8nworkflows.xyz/workflows/whatsapp-to-social-media-content-generation-with-gpt-4---approval-workflow-6769


# WhatsApp-to-Social-Media Content Generation with GPT-4 & Approval Workflow

### 1. Workflow Overview

This workflow automates the generation, review, and publication of social media content from WhatsApp user input, leveraging GPT-4 AI models and an approval process via email and WhatsApp. It is designed for businesses or content creators who want to streamline multi-platform social media content creation with AI assistance and human review before publishing.

**Target Use Cases:**  
- Automating content creation for multiple social media platforms (LinkedIn, Instagram, Facebook, X/Twitter, TikTok, Threads, YouTube Shorts) from WhatsApp messages.  
- Ensuring content approval through WhatsApp and email before publishing.  
- Managing image generation and upload for posts.  
- Publishing approved content automatically on various social media accounts.  
- Providing comprehensive post-publication reporting and error handling.

**Logical Blocks:**  
- **1.1 Input Reception & Initial Processing**: Receive user input via WhatsApp, call the main AI agent to generate social media posts.  
- **1.2 AI Content Generation**: Use GPT-4 and LangChain agents to generate platform-specific posts with structured output.  
- **1.3 Image Creation & Upload**: Generate or upload images for posts using OpenAI image generation and imgbb.com upload.  
- **1.4 Content Review & Approval**: Prepare and send approval emails via Gmail, receive approval status, and decide whether to proceed.  
- **1.5 Social Media Publishing**: Post approved content to LinkedIn, Instagram, Facebook, X (Twitter).  
- **1.6 Results Aggregation & Notification**: Collect results from posting nodes, prepare user-friendly summaries and send final notifications by email and WhatsApp.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Initial Processing

**Overview:**  
Receives user input from WhatsApp (or an HTTP request node simulating WhatsApp input), then triggers the main content generation agent.

**Nodes Involved:**  
- WhapAround.pro HTTP Request  
- Social Media Content Factory

**Node Details:**

- **WhapAround.pro HTTP Request**  
  - Type: HTTP Request  
  - Role: Entry point simulating WhatsApp user input reception.  
  - Config: Standard HTTP GET/POST handling with no special options shown.  
  - Connections: Output to Social Media Content Factory node.  
  - Edge cases: Input format errors, HTTP request failures, timeouts.

- **Social Media Content Factory**  
  - Type: LangChain Agent Node (AI Agent)  
  - Role: Core AI agent generating structured social media post content across platforms based on input.  
  - Configuration:  
    - Uses GPT-4o model (OpenAI) as language model.  
    - Detailed prompt instructing it to produce platform-specific, engaging content with hashtags, CTAs, and tone adapted per platform.  
    - Schema defined for structured JSON output specifying fields for posts and media suggestions for LinkedIn, Instagram, Facebook, X, TikTok, Threads, YouTube Shorts, plus additional notes.  
  - Inputs: User input from WhapAround.pro HTTP Request node.  
  - Outputs: Structured JSON content to Prepare Content Review message and Social Media Content nodes.  
  - Edge Cases: API errors, prompt parsing errors, incomplete output.  
  - Version requirements: LangChain agent node v1.7; requires valid OpenAI API credentials.

---

#### 1.2 AI Content Parsing & Review Preparation

**Overview:**  
Parses the structured JSON output from the AI agent and prepares a formatted HTML email for content review and approval.

**Nodes Involved:**  
- Social Media Content (OutputParserStructured)  
- Prepare Content Review message

**Node Details:**

- **Social Media Content**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates and parses the AI content JSON output according to the defined schema, ensuring structured data integrity.  
  - Config: Manual JSON schema describing all platform post fields and expected data types.  
  - Inputs: From Social Media Content Factory.  
  - Outputs: Parsed structured JSON to Prepare Content Review message.  
  - Edge Cases: Schema validation failures, missing fields, malformed JSON.

- **Prepare Content Review message**  
  - Type: LangChain Agent  
  - Role: Generates a clean, modern HTML email summarizing the AI-generated post content for all platforms, including images/videos suggestions, hashtags formatted as clickable links, CTAs, and emojis preserved.  
  - Config: Detailed prompt with instructions for email formatting, inline CSS for compatibility, platform-specific card sections, mobile responsiveness.  
  - Inputs: Parsed JSON from Social Media Content node.  
  - Outputs: HTML email content to WhapAround.pro HTTP Request node (sending approval message via WhatsApp).  
  - Edge Cases: HTML formatting errors, incomplete data, generation failures.

---

#### 1.3 Image Creation & Upload

**Overview:**  
Generates images for posts using OpenAI image generation or alternative image services, then uploads images to imgbb.com for hosting.

**Nodes Involved:**  
- OpenAI (Image Generation)  
- Save Image to imgbb.com3  
- Merge (for combining image data with content)  
- Merge1 (related to Instagram image upload)

**Node Details:**

- **OpenAI**  
  - Type: LangChain OpenAI Image Generation  
  - Role: Creates images based on Instagram caption or other prompts.  
  - Config: Uses prompt from Instagram caption, no additional options.  
  - Inputs: Triggered after content approval.  
  - Outputs: Binary image data to Save Image to imgbb.com3 and Merge nodes.  
  - Edge Cases: API quota limits, generation failure, inappropriate content filtering.

- **Save Image to imgbb.com3**  
  - Type: HTTP Request  
  - Role: Uploads generated image binary data to imgbb.com, returning hosted image URL.  
  - Config: POST multipart-form upload with API key from environment variable.  
  - Inputs: Binary image data from OpenAI.  
  - Outputs: JSON with image URL for use in social media posts.  
  - Edge Cases: API key invalid, upload failures, network errors.

- **Merge**  
  - Type: Merge (Combine mode)  
  - Role: Combines image upload response with other data streams for unified processing.  
  - Inputs: From Save Image to imgbb.com3 and OpenAI.  
  - Outputs: Data forwarded to Instagram Image and Merge2 nodes.  
  - Edge Cases: Mismatched data items, timing issues.

- **Merge1**  
  - Type: Merge (Combine mode)  
  - Role: Combines data for Instagram image posting.  
  - Inputs: From Save Image to imgbb.com3 and other nodes.  
  - Outputs: To Instagram Image node.  
  - Edge Cases: Missing image data, timing issues.

---

#### 1.4 Content Review & Approval

**Overview:**  
Sends an approval email to a designated recipient and waits for approval status, proceeding only if approved.

**Nodes Involved:**  
- Approve Final Post Content (Gmail)  
- Is Approved? (If)  
- Set Default True 2 (Set)  
- Is Content Approved? (If)

**Node Details:**

- **Approve Final Post Content**  
  - Type: Gmail node  
  - Role: Sends a formatted email containing LinkedIn post preview with image and content for approval.  
  - Config: Uses HTML email content with embedded image URL, double approval option, 45-minute timeout.  
  - Inputs: Triggered after content generation and image upload.  
  - Outputs: Approval response (boolean) to Is Approved? node.  
  - Edge Cases: Email delivery issues, no response, approval timeout.

- **Is Approved?**  
  - Type: If node  
  - Role: Checks if approval status is true to continue publishing.  
  - Config: Boolean check on $json.data.approved.  
  - Inputs: From Approve Final Post Content or Set Default True 2.  
  - Outputs: Yes branch proceeds to publishing nodes; No branch halts or alternative flow.  
  - Edge Cases: Missing approval data, incorrect format.

- **Set Default True 2**  
  - Type: Set node  
  - Role: Sets default approval to true (bypass approval if needed).  
  - Inputs: From Merge node (fallback).  
  - Outputs: To Is Approved? node.  
  - Edge Cases: Unintended auto-approval.

- **Is Content Approved?**  
  - Type: If node  
  - Role: Checks approval status after WhatsApp approval, triggers OpenAI image generation if approved.  
  - Inputs: From WhappAround.pro HTTP Request node.  
  - Outputs: Yes branch to OpenAI image generation; No branch halts or alternative flow.

---

#### 1.5 Social Media Publishing

**Overview:**  
Publishes approved posts to various social media platforms using their respective API nodes.

**Nodes Involved:**  
- Instagram Image (HTTP Request)  
- Instragram Post (Facebook Graph API)  
- X Post (Twitter node)  
- Facebook Post (Facebook Graph API)  
- LinkedIn Post (LinkedIn node)  
- Merge1, Merge2, Merge Results, and Result Set nodes (Instagram Result, X Result, Facebook Result, LinkedIn Result)

**Node Details:**

- **Instagram Image**  
  - Type: HTTP Request  
  - Role: Uploads image to Instagram media endpoint with caption.  
  - Config: Uses Facebook Graph API with predefined Facebook credentials.  
  - Inputs: Merged image URL and post data.  
  - Outputs: Media ID for publishing.  
  - Edge Cases: Permission errors, invalid media ID.

- **Instragram Post**  
  - Type: Facebook Graph API  
  - Role: Publishes Instagram post using media ID and caption.  
  - Config: Uses Facebook Graph API with page or user ID.  
  - Inputs: Media ID from Instagram Image node.  
  - Outputs: Post result to Instagram Result node.  
  - Edge Cases: Access token expiration, API rate limits.

- **X Post**  
  - Type: Twitter node  
  - Role: Posts text content to X/Twitter.  
  - Config: Uses OAuth credentials for Twitter API.  
  - Inputs: Post text from AI content.  
  - Outputs: Post result to X Result node.  
  - Edge Cases: OAuth token expiration, rate limits.

- **Facebook Post**  
  - Type: Facebook Graph API  
  - Role: Posts photo with message and link.  
  - Config: Uses Facebook Graph API with page ID and image binary data.  
  - Inputs: Image and post message.  
  - Outputs: Post result to Facebook Result node.  
  - Edge Cases: Permission issues, invalid image data.

- **LinkedIn Post**  
  - Type: LinkedIn node  
  - Role: Posts text content as organization.  
  - Config: Uses LinkedIn OAuth with organization ID.  
  - Inputs: Post text, hashtags, image media data.  
  - Outputs: Post result to LinkedIn Result node.  
  - Edge Cases: OAuth token issues, organization permissions.

- **Merge1, Merge2, Merge Results**  
  - Type: Merge nodes  
  - Role: Combine post results from different platforms for summary and reporting.  
  - Inputs: Post results from social media nodes.  
  - Outputs: To aggregation and result preparation nodes.  
  - Edge Cases: Missing or partial results.

- **Instagram Result, X Result, Facebook Result, LinkedIn Result**  
  - Type: Set nodes  
  - Role: Format and store individual platform post results for reporting.  
  - Inputs: From respective social media post nodes.  
  - Outputs: To Merge Results node.  
  - Edge Cases: Missing post data.

---

#### 1.6 Results Aggregation & Notification

**Overview:**  
Aggregates posting results, generates summary messages and HTML email reports, then notifies stakeholders by email and WhatsApp.

**Nodes Involved:**  
- Merge Results  
- Aggregate  
- Prepare Results Email  
- Prepare Results Message  
- Gmail Results  
- whappAround HTTP Request

**Node Details:**

- **Merge Results**  
  - Type: Merge node  
  - Role: Consolidates all platform post results into a single data stream.  
  - Inputs: Instagram, X, Facebook, LinkedIn Result nodes.  
  - Outputs: To Aggregate node.  
  - Edge Cases: Partial data or missing inputs.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates all items into a single JSON array for processing.  
  - Inputs: From Merge Results.  
  - Outputs: To Prepare Results Email and Prepare Results Message.  
  - Edge Cases: Large data volume handling.

- **Prepare Results Email**  
  - Type: LangChain Agent  
  - Role: Generates a professional HTML email table summarizing success/error status per platform with error details if any.  
  - Inputs: Aggregated post results.  
  - Outputs: HTML email to Gmail Results node.  
  - Edge Cases: Malformed JSON, incomplete data.

- **Prepare Results Message**  
  - Type: LangChain Agent  
  - Role: Creates a brief plain-text summary of posting results, including error descriptions.  
  - Inputs: Aggregated post results.  
  - Outputs: Message to whappAround HTTP Request node.  
  - Edge Cases: Parsing errors.

- **Gmail Results**  
  - Type: Gmail node  
  - Role: Sends final detailed results email to stakeholder (Joe).  
  - Inputs: HTML email content from Prepare Results Email.  
  - Edge Cases: Email delivery failure.

- **whappAround HTTP Request**  
  - Type: HTTP Request  
  - Role: Sends brief text results notification back to WhatsApp or related system.  
  - Inputs: Summary message from Prepare Results Message.  
  - Edge Cases: HTTP failures.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                          | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                               |
|---------------------------|--------------------------------------|----------------------------------------|---------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| WhapAround.pro HTTP Request| HTTP Request                         | Receive WhatsApp user input             | -                               | Social Media Content Factory     | # 🧑‍🦱 User Input on whatsapp for Social Media Posts                                                                                                                                                                                                        |
| Social Media Content Factory| LangChain Agent                    | Generate platform-specific content      | WhapAround.pro HTTP Request      | Prepare Content Review message, Social Media Content | # 🛠️ Social Media Content Factory - LinkedIn - Instagram - Facebook - X - TikTok - Threads - YouTube Shorts                                                                                                                                                  |
| Social Media Content       | LangChain Output Parser Structured   | Parse AI content JSON                    | Social Media Content Factory     | Social Media Content Factory (AI) |                                                                                                                                                                                                                                                            |
| Prepare Content Review message| LangChain Agent                  | Format content review email              | Social Media Content             | WhapAround.pro HTTP Request      |                                                                                                                                                                                                                                                            |
| Approve Final Post Content | Gmail                                | Send approval email and wait for response| Prepare Content Review message   | Is Approved?                    | # 👍 Approve Content on whatsapp Before Proceeding                                                                                                                                                                                                          |
| Is Approved?               | If                                   | Check approval status                    | Approve Final Post Content, Set Default True 2 | X Post, Merge1, Merge2           |                                                                                                                                                                                                                                                            |
| Set Default True 2         | Set                                  | Default approval true                    | Merge                          | Is Approved?                    |                                                                                                                                                                                                                                                            |
| Is Content Approved?       | If                                   | Check WhatsApp approval status           | WhappAround.pro HTTP Request    | OpenAI                         |                                                                                                                                                                                                                                                            |
| OpenAI                    | LangChain OpenAI Image Generation    | Generate post images                      | Is Content Approved?            | Save Image to imgbb.com3, Merge | ## Create Post Image https://pollinations.ai/ https://image.pollinations.ai/prompt/[your image description] (alternative)                                                                                                                                   |
| Save Image to imgbb.com3   | HTTP Request                        | Upload image to hosting                   | OpenAI                        | Merge, Merge1                   |                                                                                                                                                                                                                                                            |
| Merge                     | Merge (Combine)                      | Combine image upload and content data    | Save Image to imgbb.com3, OpenAI | Instagram Image, Merge2          |                                                                                                                                                                                                                                                            |
| Merge1                    | Merge (Combine)                      | Combine data streams for Instagram post  | Save Image to imgbb.com3, others | Instagram Image                 |                                                                                                                                                                                                                                                            |
| Instagram Image           | HTTP Request                        | Upload image to Instagram media endpoint | Merge1                        | Instragram Post                |                                                                                                                                                                                                                                                            |
| Instragram Post           | Facebook Graph API                  | Publish Instagram post                    | Instagram Image                | Instagram Result               |                                                                                                                                                                                                                                                            |
| X Post                    | Twitter node                        | Publish post to X/Twitter                 | Is Approved?                  | X Result                      |                                                                                                                                                                                                                                                            |
| Facebook Post             | Facebook Graph API                  | Publish Facebook photo post               | Merge2                        | Facebook Result               |                                                                                                                                                                                                                                                            |
| LinkedIn Post             | LinkedIn node                      | Publish LinkedIn organization post       | Merge2                        | LinkedIn Result               |                                                                                                                                                                                                                                                            |
| Merge2                    | Merge (Combine)                    | Combine Instagram and Facebook/LinkedIn  | Merge, others                 | Facebook Post, LinkedIn Post   |                                                                                                                                                                                                                                                            |
| Instagram Result          | Set node                          | Store Instagram post result                | Instragram Post               | Merge Results                 |                                                                                                                                                                                                                                                            |
| X Result                  | Set node                          | Store X post result                        | X Post                       | Merge Results                 |                                                                                                                                                                                                                                                            |
| Facebook Result           | Set node                          | Store Facebook post result                 | Facebook Post                 | Merge Results                 |                                                                                                                                                                                                                                                            |
| LinkedIn Result           | Set node                          | Store LinkedIn post result                 | LinkedIn Post                 | Merge Results                 |                                                                                                                                                                                                                                                            |
| Merge Results             | Merge (Combine)                   | Aggregate all platform post results       | Instagram Result, X Result, Facebook Result, LinkedIn Result | Aggregate                     |                                                                                                                                                                                                                                                            |
| Aggregate                 | Aggregate node                   | Aggregate multiple results into one       | Merge Results                | Prepare Results Email, Prepare Results Message |                                                                                                                                                                                                                                                            |
| Prepare Results Email     | LangChain Agent                  | Generate HTML email summary of results    | Aggregate                    | Gmail Results                | # Step 4️⃣: Send Final Results of Social Media Factory                                                                                                                                                                                                     |
| Prepare Results Message   | LangChain Agent                  | Generate brief summary message             | Aggregate                    | whappAround HTTP Request      |                                                                                                                                                                                                                                                            |
| Gmail Results             | Gmail                           | Send final results email                    | Prepare Results Email        | -                            |                                                                                                                                                                                                                                                            |
| whappAround HTTP Request  | HTTP Request                   | Send brief results notification to WhatsApp| Prepare Results Message      | -                            |                                                                                                                                                                                                                                                            |
| Sticky Note (multiple)    | Sticky Note                     | Descriptive notes and instructions         | -                           | -                            | Multiple contextual notes for user guidance, API keys, process steps, platform credentials, and publishing instructions.                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create HTTP Request Node: "WhapAround.pro HTTP Request"**  
   - Purpose: Receive WhatsApp user input via HTTP webhook or API call.  
   - Config: Accept POST requests with expected input JSON containing topic, keywords, optional link.

2. **Create LangChain Agent Node: "Social Media Content Factory"**  
   - Purpose: Generate social media posts adapted per platform using GPT-4o.  
   - Config:  
     - Language Model: OpenAI GPT-4o with responseFormat "text".  
     - Prompt: Detailed content creation instructions per platform including tone, hashtags, CTAs, and JSON schema.  
     - Schema: Define structured JSON output for multiple platforms with properties like post text, hashtags, image/video suggestions, and CTAs.  
   - Connect input from "WhapAround.pro HTTP Request".

3. **Create LangChain Output Parser Node: "Social Media Content"**  
   - Purpose: Parse AI output with JSON schema validation.  
   - Config: Manual schema matching the AI agent output structure.  
   - Connect input from "Social Media Content Factory".

4. **Create LangChain Agent Node: "Prepare Content Review message"**  
   - Purpose: Format AI content into a professional HTML email for approval.  
   - Config: Prompt instructing email formatting with tables, inline CSS, and platform cards.  
   - Connect input from "Social Media Content".

5. **Create HTTP Request Node: "WhapAround.pro HTTP Request" (Approval message send)**  
   - Purpose: Send the prepared HTML approval message via WhatsApp or related system.  
   - Connect input from "Prepare Content Review message".

6. **Create Gmail Node: "Approve Final Post Content"**  
   - Purpose: Send approval email to designated user with double approval option and timeout.  
   - Config:  
     - Recipient email from environment variable.  
     - HTML message with embedded post preview and image.  
     - Operation: sendAndWait with approval options.  
   - Connect input from "Prepare Content Review message".

7. **Create If Node: "Is Approved?"**  
   - Purpose: Check for approval boolean in response.  
   - Condition: $json.data.approved == true.  
   - Connect input from "Approve Final Post Content".  
   - Yes output to publishing nodes, No output to stop or alternate path.

8. **Create Set Node: "Set Default True 2"**  
   - Purpose: Optional node to force approval true (for testing or bypass).  
   - Set field: data.approved = true.  
   - Connect input from "Merge" (or appropriate fallback).  
   - Output to "Is Approved?".

9. **Create If Node: "Is Content Approved?"**  
   - Purpose: Check approval from WhatsApp response before generating images.  
   - Condition: $json.data.approved == true.  
   - Connect input from "WhapAround.pro HTTP Request" (approval response).  
   - Yes output to OpenAI image generation.

10. **Create LangChain OpenAI Node: "OpenAI" (Image Generation)**  
    - Purpose: Generate images for Instagram posts based on captions.  
    - Config: Use image resource, prompt from Instagram caption field.  
    - Connect input from "Is Content Approved?".

11. **Create HTTP Request Node: "Save Image to imgbb.com3"**  
    - Purpose: Upload generated image binary to imgbb.com for hosting.  
    - Config: POST multipart form with API key from environment variable.  
    - Connect input from "OpenAI" image generation.

12. **Create Merge Nodes ("Merge" and "Merge1")**  
    - Purpose: Combine image upload response with other post content data streams.  
    - Connect inputs from "Save Image to imgbb.com3" and "OpenAI", or other nodes as needed.

13. **Create HTTP Request Node: "Instagram Image"**  
    - Purpose: Upload image to Instagram media endpoint.  
    - Config: Facebook Graph API endpoint with OAuth credentials.  
    - Connect input from "Merge1".

14. **Create Facebook Graph API Node: "Instragram Post"**  
    - Purpose: Publish Instagram post using media ID.  
    - Connect input from "Instagram Image".

15. **Create Twitter Node: "X Post"**  
    - Purpose: Publish text post to X/Twitter.  
    - Connect input from "Is Approved?" yes output.

16. **Create Facebook Graph API Node: "Facebook Post"**  
    - Purpose: Publish photo post on Facebook.  
    - Connect input from "Merge2".

17. **Create LinkedIn Node: "LinkedIn Post"**  
    - Purpose: Publish organization post on LinkedIn.  
    - Connect input from "Merge2".

18. **Create Merge Nodes: "Merge2" and "Merge Results"**  
    - Purpose: Combine all post result data for reporting.  
    - Connect outputs from social media post nodes as inputs to these merges.

19. **Create Set Nodes: "Instagram Result", "X Result", "Facebook Result", "LinkedIn Result"**  
    - Purpose: Format and store individual platform results.  
    - Connect input from respective social media post nodes.  
    - Output to "Merge Results".

20. **Create Aggregate Node: "Aggregate"**  
    - Purpose: Aggregate all results into a single JSON array.  
    - Connect input from "Merge Results".

21. **Create LangChain Agent Nodes: "Prepare Results Email" and "Prepare Results Message"**  
    - Purpose: Generate detailed HTML email and brief text summary of post results.  
    - Connect inputs from "Aggregate".

22. **Create Gmail Node: "Gmail Results"**  
    - Purpose: Send final HTML results email to stakeholder.  
    - Connect input from "Prepare Results Email".

23. **Create HTTP Request Node: "whappAround HTTP Request"**  
    - Purpose: Send brief results message back to WhatsApp or related system.  
    - Connect input from "Prepare Results Message".

24. **Create Sticky Notes**  
    - Add contextual notes at various workflow stages for user guidance, credentials, API keys, and instructions.

25. **Credentials Setup**  
    - OpenAI API Key for GPT-4o and image generation.  
    - Facebook Graph API OAuth credentials for Instagram and Facebook posting.  
    - Twitter OAuth credentials for X posting.  
    - LinkedIn OAuth credentials and organization ID.  
    - Gmail OAuth for sending approval and results emails.  
    - imgbb API Key for image hosting.  
    - SERP API Key (optional, for search integration).  

26. **Test and Validate**  
    - Test end-to-end flow with sample WhatsApp input.  
    - Verify AI content generation, image creation, approval email delivery, approval response handling, and social media publishing.  
    - Validate error handling for API failures and approvals.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                            | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Adjust Social Media Content Factory prompts to suit your personal or business requirements.                                                                                                                                                                             | Prompt customization guidance                                   |
| Update Credentials for: OpenAI API (https://auth.openai.com/log-in), Google Studio API (https://aistudio.google.com/app/apikey), SERP API Key (https://serpapi.com/), Gmail OAuth (https://docs.n8n.io/integrations/builtin/credentials/google/), Telegram Bot (optional), imgbb (https://imgbb.com/) | Credential setup instructions                                  |
| Create post images using https://pollinations.ai/ or OpenAI image generation.                                                                                                                                                                                           | Alternative image generation resources                           |
| Social Media Content Factory covers multiple platforms: LinkedIn, Instagram, Facebook, X (Twitter), TikTok, Threads, YouTube Shorts.                                                                                                                                     | Platform coverage overview                                      |
| Approval workflow uses Gmail with double approval and 45-minute timeout for confirmation.                                                                                                                                                                                | Email approval process details                                  |
| Final results include detailed HTML email with platform-specific success/error statuses and WhatsApp notification with summary.                                                                                                                                           | Post-publication reporting and notifications                    |
| Workflow requires API permissions and valid OAuth tokens for all social media APIs for successful publishing.                                                                                                                                                             | API access prerequisites                                       |
| Content is generated with a focus on professional tone, platform-specific optimization, SEO/hashtags, engagement, and consistency to reflect workflows.diy branding and expertise.                                                                                       | Content creation strategy                                       |

---

**Disclaimer:**  
The text and workflow described derive exclusively from an automated n8n workflow. All data manipulated is legal and public. This documentation strictly respects content policies and contains no illegal or offensive elements.