✨🩷Automated Social Media Content Publishing Factory + System Prompt Composition

https://n8nworkflows.xyz/workflows/---automated-social-media-content-publishing-factory---system-prompt-composition-3135


# ✨🩷Automated Social Media Content Publishing Factory + System Prompt Composition

### 1. Workflow Overview

This workflow is an **Automated Social Media Content Publishing Factory** designed to streamline content creation and publishing across multiple social media platforms including X (Twitter), Instagram, Facebook, LinkedIn, Threads, and YouTube Shorts. It targets content creators, social media managers, and marketing teams aiming to maintain a consistent and optimized social media presence without manual platform-specific content crafting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Chat Memory:** Receives user content ideas via chat and maintains conversation context.
- **1.2 System Prompt and Schema Composition:** Retrieves and parses external system prompts and JSON schemas from Google Docs to standardize content generation.
- **1.3 Social Media Content Generation:** Uses AI agents to generate platform-optimized content based on user input, system prompts, and schemas.
- **1.4 AI Image Generation and Storage:** Creates matching images via AI image generation services and stores them in cloud storage.
- **1.5 Content Approval Process:** Sends generated content for approval via email and waits for confirmation before publishing.
- **1.6 Social Media Publishing Router:** Routes approved content to the appropriate social media platform nodes for publishing.
- **1.7 Post-Publishing Archiving and Notifications:** Archives published content and images, and optionally sends notifications via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Chat Memory

**Overview:**  
This block handles incoming user content ideas via a chat interface and maintains chat memory to provide context for AI content generation.

**Nodes Involved:**  
- When chat message received  
- Window Buffer Memory  
- 🤖Social Media Router Agent  

**Node Details:**  
- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Entry point for user content ideas via chat  
  - Configuration: Default chat trigger with webhook enabled  
  - Input: Incoming chat message  
  - Output: User prompt text  
  - Edge cases: Webhook failures, malformed chat input  

- **Window Buffer Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains chat memory context for AI agents  
  - Configuration: Default window buffer to store recent conversation  
  - Input: Chat messages  
  - Output: Contextual memory for AI agents  
  - Edge cases: Memory overflow or loss, context mismatch  

- **🤖Social Media Router Agent**  
  - Type: Langchain Agent  
  - Role: Determines which social media platform tool to invoke based on user prompt  
  - Configuration: System message restricts direct answers; only calls platform-specific tools  
  - Input: User chat input  
  - Output: JSON directing which platform tool to use  
  - Edge cases: Tool invocation errors, invalid platform routing  

---

#### 2.2 System Prompt and Schema Composition

**Overview:**  
This block retrieves system prompts and JSON schemas from Google Docs, parses them, and prepares structured data for AI content generation.

**Nodes Involved:**  
- Social Media System Prompt (Google Docs)  
- Social Media Schema (Google Docs)  
- System Prompt (Set)  
- Schema (Set)  
- Parse System Prompt (Code)  
- Parse Schema (Code)  
- Merge Prompts and Schema (Merge)  
- Compose Prompt & Schema (Set)  

**Node Details:**  
- **Social Media System Prompt**  
  - Type: Google Docs  
  - Role: Fetches system prompt XML content from Google Docs  
  - Configuration: Document URL set to external prompt doc  
  - Input: Trigger from sub-workflow or manual execution  
  - Output: Raw XML system prompt content  
  - Edge cases: Google Docs API errors, invalid document ID  

- **Social Media Schema**  
  - Type: Google Docs  
  - Role: Fetches JSON schema XML content from Google Docs  
  - Configuration: Document URL set to external schema doc  
  - Input: Trigger from sub-workflow or manual execution  
  - Output: Raw XML schema content  
  - Edge cases: Google Docs API errors, invalid document ID  

- **System Prompt (Set)**  
  - Type: Set  
  - Role: Extracts system prompt content and document ID for downstream use  
  - Configuration: Assigns `system_prompt` and `system_prompt_doc_id` from input  

- **Schema (Set)**  
  - Type: Set  
  - Role: Extracts schema content and platform route from input  
  - Configuration: Assigns `schema` and `platform` variables  

- **Parse System Prompt (Code)**  
  - Type: Code  
  - Role: Parses XML system prompt into JSON object keyed by tags (e.g., system, rules, linkedin)  
  - Configuration: Uses regex to extract all XML tags and their content  
  - Output: JSON object with system configuration for AI  

- **Parse Schema (Code)**  
  - Type: Code  
  - Role: Parses XML schema into JSON objects for platform-specific schema, root schema, and common schema  
  - Configuration: Extracts platform-specific schema based on route  
  - Output: JSON with `schema`, `root_schema`, and `common_schema`  

- **Merge Prompts and Schema (Merge)**  
  - Type: Merge  
  - Role: Combines outputs from system prompt parsing, schema parsing, and initial input  
  - Configuration: Combine by position from three inputs  

- **Compose Prompt & Schema (Set)**  
  - Type: Set  
  - Role: Prepares final variables including route, user prompt, system config, and schemas for AI agent  
  - Configuration: Assigns all necessary fields for content generation  

**Edge Cases:**  
- Parsing errors due to malformed XML or JSON  
- Missing or incorrect platform keys  
- Google Docs API rate limits or failures  

---

#### 2.3 Social Media Content Generation

**Overview:**  
This block uses an AI agent to generate platform-optimized social media content based on the composed prompts and schemas.

**Nodes Involved:**  
- Social Media Content Creator (Langchain Agent)  
- Social Content (Set)  
- pollinations.ai1 (HTTP Request for image generation)  

**Node Details:**  
- **Social Media Content Creator**  
  - Type: Langchain Agent  
  - Role: Generates social media content JSON conforming to platform schema  
  - Configuration: Uses system message with rules and schemas, user prompt, and web search tool (SerpAPI) for real-time info  
  - Input: Composed prompt and schema data  
  - Output: JSON content for social media post  
  - Edge cases: API rate limits, malformed output, schema validation failures  

- **Social Content (Set)**  
  - Type: Set  
  - Role: Stores AI-generated content output for downstream use  

- **pollinations.ai1**  
  - Type: HTTP Request  
  - Role: Generates AI images based on image suggestion from content  
  - Configuration: Calls Pollinations.ai with sanitized prompt, retries on failure  
  - Output: Image URL and metadata  
  - Edge cases: API failures, image generation delays, invalid prompts  

---

#### 2.4 AI Image Generation and Storage

**Overview:**  
This block downloads generated images, uploads them to image hosting and cloud storage, and prepares metadata for publishing.

**Nodes Involved:**  
- Get Social Post Image (HTTP Request)  
- Save Image to imgbb.com (HTTP Request)  
- Save Image to Google Drive (Google Drive)  
- Google Drive Image Meta (Set)  
- Merge (Merge)  
- Merge Image and Post Contents (Merge)  

**Node Details:**  
- **Get Social Post Image**  
  - Type: HTTP Request  
  - Role: Downloads the image from Pollinations.ai URL  
  - Configuration: Retries on failure  
  - Edge cases: Network errors, invalid URLs  

- **Save Image to imgbb.com**  
  - Type: HTTP Request  
  - Role: Uploads image binary to imgbb.com for hosting  
  - Configuration: Uses API key from environment variables  
  - Edge cases: API key invalid, upload failures  

- **Save Image to Google Drive**  
  - Type: Google Drive  
  - Role: Saves image file to Google Drive folder for archival  
  - Configuration: Uses configured Drive and folder IDs  
  - Edge cases: Google Drive API errors, permission issues  

- **Google Drive Image Meta**  
  - Type: Set  
  - Role: Extracts and stores metadata from Google Drive upload response  

- **Merge**  
  - Type: Merge  
  - Role: Combines image data from multiple sources for unified metadata  

- **Merge Image and Post Contents**  
  - Type: Merge  
  - Role: Combines social media content JSON with image metadata for publishing  

---

#### 2.5 Content Approval Process

**Overview:**  
This block sends generated content and images for approval via Gmail and waits for approval before proceeding to publishing.

**Nodes Involved:**  
- Prepare Email Contents (Langchain Agent)  
- Gmail User for Approval (Gmail)  
- Is Approved? (If)  

**Node Details:**  
- **Prepare Email Contents**  
  - Type: Langchain Agent  
  - Role: Generates HTML email content using a template populated with social content and image metadata  
  - Configuration: Uses system prompt to output only HTML without extra text  
  - Edge cases: Template parsing errors, malformed HTML  

- **Gmail User for Approval**  
  - Type: Gmail node  
  - Role: Sends email for approval and waits for user response (double approval)  
  - Configuration: Sends to configured email, subject includes content title  
  - Edge cases: Email delivery failures, timeout waiting for approval  

- **Is Approved?**  
  - Type: If  
  - Role: Checks if approval flag is true to continue publishing  
  - Edge cases: Missing approval flag, false negatives  

---

#### 2.6 Social Media Publishing Router

**Overview:**  
This block routes approved content to the appropriate social media platform publishing nodes.

**Nodes Involved:**  
- Social Media Publishing Router (Switch)  
- X Post (Twitter)  
- Instagram Image (Facebook Graph API)  
- Instragram Post (Facebook Graph API)  
- Facebook Post (Facebook Graph API)  
- LinkedIn Post (LinkedIn)  
- Implement Threads Here (NoOp placeholder)  
- Implement YouTube Shorts Here (NoOp placeholder)  

**Node Details:**  
- **Social Media Publishing Router**  
  - Type: Switch  
  - Role: Routes content based on platform route field (xtwitter, instagram, facebook, linkedin, threads, youtube_short)  
  - Edge cases: Unknown platform route, routing errors  

- **X Post**  
  - Type: Twitter node  
  - Role: Publishes post text to X (Twitter)  
  - Configuration: Uses OAuth2 credentials, posts text from content JSON  
  - Edge cases: API rate limits, auth errors  

- **Instagram Image**  
  - Type: Facebook Graph API  
  - Role: Uploads image media for Instagram via Facebook Graph API  
  - Configuration: Uses Facebook Graph credentials, posts image URL and caption  
  - Edge cases: API errors, invalid media IDs  

- **Instragram Post**  
  - Type: Facebook Graph API  
  - Role: Publishes Instagram post using media creation ID  
  - Edge cases: Timing issues between media upload and publish  

- **Facebook Post**  
  - Type: Facebook Graph API  
  - Role: Posts photo with message to Facebook page  
  - Edge cases: API errors, permission issues  

- **LinkedIn Post**  
  - Type: LinkedIn node  
  - Role: Posts text and image to LinkedIn organization page  
  - Edge cases: API errors, invalid organization ID  

- **Implement Threads Here**  
  - Type: NoOp  
  - Role: Placeholder for future Threads publishing implementation  

- **Implement YouTube Shorts Here**  
  - Type: NoOp  
  - Role: Placeholder for future YouTube Shorts publishing implementation  

---

#### 2.7 Post-Publishing Archiving and Notifications

**Overview:**  
This block archives published content and images to Google Drive and optionally sends Telegram notifications about success or errors.

**Nodes Involved:**  
- Save Social Post to Google Drive (Google Drive)  
- Respond with Google Drive Id (Set)  
- Telegram Success Message (Telegram)  
- Telegram Error Message (Telegram)  

**Node Details:**  
- **Save Social Post to Google Drive**  
  - Type: Google Drive  
  - Role: Saves the full social post JSON response as a text file for archival  
  - Edge cases: API errors, permission issues  

- **Respond with Google Drive Id**  
  - Type: Set  
  - Role: Outputs Google Drive file ID after saving  

- **Telegram Success Message**  
  - Type: Telegram  
  - Role: Sends success notification to configured Telegram chat ID  
  - Edge cases: Telegram API errors  

- **Telegram Error Message**  
  - Type: Telegram  
  - Role: Sends error notification for debugging image creation failures  
  - Edge cases: Telegram API errors  

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                   | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                     |
|--------------------------------|----------------------------------|-------------------------------------------------|----------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received      | Langchain Chat Trigger            | Entry point for user content input               | -                                | 🤖Social Media Router Agent          | ## 👍Start Here                                                                                |
| Window Buffer Memory            | Langchain Memory Buffer Window    | Maintains chat memory context                     | When chat message received        | 🤖Social Media Router Agent          | ## Chat Memory                                                                                |
| 🤖Social Media Router Agent      | Langchain Agent                  | Routes user prompt to platform-specific tools    | When chat message received, Window Buffer Memory | File Id                            | # Social Media Router Agent                                                                   |
| File Id                        | Set                              | Sets output for file ID                           | 🤖Social Media Router Agent       | Get Social Post from Google Drive    |                                                                                                |
| Get Social Post from Google Drive | Google Drive                    | Downloads social post JSON file                   | File Id                          | Extract as JSON                      |                                                                                                |
| Extract as JSON                | Extract From File                 | Parses downloaded JSON file                        | Get Social Post from Google Drive | Merge Image and Post Contents        |                                                                                                |
| Merge Image and Post Contents  | Merge                            | Combines image and post content                    | Extract as JSON, Get Social Post Image | Social Media Publishing Router      | ## Social Media Publishing Router                                                             |
| Social Media Publishing Router | Switch                          | Routes content to platform publishing nodes       | Merge Image and Post Contents     | X Post, Instagram Image, Facebook Post, LinkedIn Post, Implement Threads Here, Implement YouTube Shorts Here | ## Social Media Publishing Router                                                             |
| X Post                        | Twitter                          | Publishes post to X (Twitter)                      | Social Media Publishing Router    | X Response                         |                                                                                                |
| Instagram Image               | Facebook Graph API               | Uploads Instagram image media                       | Social Media Publishing Router    | Instragram Post                    |                                                                                                |
| Instragram Post              | Facebook Graph API               | Publishes Instagram post                            | Instagram Image                  | Instagram Response                 |                                                                                                |
| Facebook Post                | Facebook Graph API               | Publishes Facebook post                             | Social Media Publishing Router    | Facebook Response                 |                                                                                                |
| LinkedIn Post                | LinkedIn                        | Publishes LinkedIn post                            | Social Media Publishing Router    | LinkedIn Response                 |                                                                                                |
| Implement Threads Here       | NoOp                            | Placeholder for Threads publishing                 | Social Media Publishing Router    | -                                 |                                                                                                |
| Implement YouTube Shorts Here | NoOp                            | Placeholder for YouTube Shorts publishing          | Social Media Publishing Router    | -                                 |                                                                                                |
| X Response                   | Set                             | Formats X post response for output                  | X Post                          | -                                 |                                                                                                |
| Instagram Response           | Set                             | Formats Instagram post response                      | Instragram Post                 | -                                 |                                                                                                |
| Facebook Response            | Set                             | Formats Facebook post response                       | Facebook Post                  | -                                 |                                                                                                |
| LinkedIn Response            | Set                             | Formats LinkedIn post response                       | LinkedIn Post                  | -                                 |                                                                                                |
| Get Social Post Image        | HTTP Request                   | Downloads generated image                            | Is Approved?                   | Merge Image and Post Contents       |                                                                                                |
| Save Image to imgbb.com      | HTTP Request                   | Uploads image to imgbb.com                            | pollinations.ai1               | Merge                              |                                                                                                |
| Save Image to Google Drive   | Google Drive                   | Saves image to Google Drive                           | pollinations.ai1               | Merge                              |                                                                                                |
| Google Drive Image Meta      | Set                             | Extracts Google Drive image metadata                  | Save Image to Google Drive      | Social Post JSON                   |                                                                                                |
| Social Post JSON             | Set                             | Prepares final social post JSON for archiving         | Google Drive Image Meta        | Save Social Post to Google Drive    |                                                                                                |
| Save Social Post to Google Drive | Google Drive                   | Archives social post JSON                              | Social Post JSON               | Respond with Google Drive Id        | ## 9️⃣ Social Post Archiving to Google Drive for Future Use                                   |
| Respond with Google Drive Id | Set                             | Outputs Google Drive file ID                           | Save Social Post to Google Drive | -                                 |                                                                                                |
| pollinations.ai1             | HTTP Request                   | Generates AI image from prompt                         | Social Content                | Save Image to imgbb.com, Save Image to Google Drive, Merge | ## Create Post Image https://pollinations.ai/                                                  |
| Prepare Email Contents       | Langchain Agent                | Generates HTML email content for approval              | Extract as JSON, Merge Image and Post Contents | Gmail User for Approval            | ## Prepare Email Approval Contents as HTML                                                    |
| Gmail User for Approval      | Gmail                         | Sends email for content approval and waits for response | Prepare Email Contents         | Is Approved?                      | # 👍 Approve Content Before Proceeding                                                       |
| Is Approved?                | If                            | Checks if content is approved                           | Gmail User for Approval        | Get Social Post Image             |                                                                                                |
| Social Media Content Creator | Langchain Agent                | Generates platform-optimized social media content      | Compose Prompt & Schema        | Social Content                   | ## Social Media Content Creator                                                              |
| Compose Prompt & Schema     | Set                             | Prepares prompt, schema, and system config for AI       | Merge Prompts and Schema       | Social Media Content Creator       |                                                                                                |
| Merge Prompts and Schema    | Merge                          | Combines system prompt, schema, and input data          | Parse System Prompt, Parse Schema, When Executed by Another Workflow | Compose Prompt & Schema           |                                                                                                |
| Parse System Prompt         | Code                           | Parses XML system prompt into JSON                       | System Prompt                 | Merge Prompts and Schema           |                                                                                                |
| Parse Schema               | Code                           | Parses XML schema into JSON                               | Schema                       | Merge Prompts and Schema           |                                                                                                |
| System Prompt              | Set                            | Extracts system prompt content                            | Social Media System Prompt    | Parse System Prompt               |                                                                                                |
| Schema                    | Set                            | Extracts schema content and platform route                | Social Media Schema           | Parse Schema                    |                                                                                                |
| Social Media System Prompt | Google Docs                   | Fetches system prompt document                             | When Executed by Another Workflow | System Prompt                   |                                                                                                |
| Social Media Schema        | Google Docs                   | Fetches social media schema document                       | When Executed by Another Workflow | Schema                        |                                                                                                |
| When Executed by Another Workflow | Execute Workflow Trigger       | Sub-workflow entry point with user prompt and route         | -                            | Social Media System Prompt, Social Media Schema, Merge Prompts and Schema |                                                                                                |
| Prepare Social Media Email Contents | Langchain Agent                | Generates HTML email content for Gmail sending             | pollinations.ai1             | Gmail                           |                                                                                                |
| Gmail                      | Gmail                         | Sends approval email                                      | Prepare Social Media Email Contents | -                             |                                                                                                |
| Telegram Success Message (Optional) | Telegram                      | Sends success notification                                | pollinations.ai1             | -                               | ## 7️⃣ Telegram Messaging for Workflow Status                                                |
| Telegram Error Message (Optional) | Telegram                      | Sends error notification                                  | pollinations.ai1             | -                               | ## 7️⃣ Telegram Messaging for Workflow Status                                                |
| Social Content             | Set                            | Stores AI-generated social media content                   | Social Media Content Creator | pollinations.ai1                |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: Langchain Chat Trigger  
   - Purpose: Receive user content ideas via chat interface  
   - Configure webhook and default options  

2. **Add Memory Buffer Node**  
   - Type: Langchain Memory Buffer Window  
   - Purpose: Maintain chat context for AI agents  
   - Connect output of chat trigger to this node  

3. **Add Social Media Router Agent**  
   - Type: Langchain Agent  
   - Purpose: Route user prompt to platform-specific content generation tools  
   - Configure system message to restrict direct answers and list available tools  
   - Connect chat trigger and memory buffer outputs to this agent  

4. **Add Sub-Workflow Trigger Node**  
   - Type: Execute Workflow Trigger  
   - Purpose: Entry point for sub-workflows with inputs `user_prompt` and `route`  

5. **Add Google Docs Nodes for System Prompt and Schema**  
   - Type: Google Docs  
   - Purpose: Fetch system prompt and JSON schema documents  
   - Configure with document URLs for system prompt and schema  
   - Connect sub-workflow trigger node outputs to these nodes  

6. **Add Set Nodes to Extract Content**  
   - Extract system prompt content and document ID  
   - Extract schema content and platform route  

7. **Add Code Nodes to Parse XML Content**  
   - Parse system prompt XML into JSON object keyed by tags  
   - Parse schema XML into JSON objects for platform schema, root schema, and common schema  

8. **Add Merge Node to Combine Parsed Data**  
   - Combine outputs of system prompt parsing, schema parsing, and initial input  

9. **Add Set Node to Compose Final Prompt and Schema**  
   - Assign variables: route, user_prompt, system config, root schema, common schema, platform schema  

10. **Add Langchain Agent Node for Content Creation**  
    - Configure with system message including rules and schemas  
    - Use web search tool (SerpAPI) for real-time info  
    - Connect composed prompt & schema node to this agent  

11. **Add Set Node to Store AI-generated Content**  

12. **Add HTTP Request Node for AI Image Generation**  
    - Use Pollinations.ai or alternative image generation service  
    - Configure URL with sanitized image suggestion from content  
    - Enable retry on failure  

13. **Add HTTP Request Node to Download Generated Image**  
    - Download image from AI service URL  
    - Enable retry on failure  

14. **Add HTTP Request Node to Upload Image to imgbb.com**  
    - Configure with API key from environment variables  
    - Upload image binary data  

15. **Add Google Drive Node to Save Image**  
    - Configure with Drive and folder IDs  
    - Upload image file for archival  

16. **Add Set Node to Extract Google Drive Image Metadata**  

17. **Add Merge Node to Combine Image Metadata**  

18. **Add Merge Node to Combine Image and Post Content**  

19. **Add Langchain Agent Node to Prepare Email Contents**  
    - Use HTML template populated with social content and image metadata  
    - Configure to output only HTML  

20. **Add Gmail Node for Approval Email**  
    - Configure with OAuth2 credentials  
    - Send to approval email address  
    - Enable send and wait with double approval option  

21. **Add If Node to Check Approval Status**  

22. **Add Switch Node for Social Media Publishing Router**  
    - Configure rules to route based on platform route field (xtwitter, instagram, facebook, linkedin, threads, youtube_short)  

23. **Add Platform Publishing Nodes:**  
    - Twitter node for X Post (OAuth2 credentials)  
    - Facebook Graph API nodes for Instagram Image upload and Instagram Post publish  
    - Facebook Graph API node for Facebook Post  
    - LinkedIn node for LinkedIn Post (OAuth2 credentials)  
    - NoOp nodes as placeholders for Threads and YouTube Shorts publishing  

24. **Add Set Nodes to Format Platform Responses**  

25. **Add Google Drive Node to Save Social Post JSON**  
    - Archive full social post JSON response  
    - Configure with Drive and folder IDs  

26. **Add Set Node to Respond with Google Drive File ID**  

27. **Add Telegram Nodes for Optional Notifications**  
    - Success message on image creation  
    - Error message on image creation failure  

28. **Credential Setup:**  
    - OpenAI API key for Langchain AI nodes  
    - Google Docs OAuth2 for fetching prompts and schemas  
    - Google Drive OAuth2 for image and post archiving  
    - Gmail OAuth2 for sending approval emails  
    - Twitter OAuth2 for posting to X  
    - Facebook Graph API credentials for Instagram and Facebook posting  
    - LinkedIn OAuth2 for LinkedIn posting  
    - Telegram API for notifications  
    - imgbb.com API key for image hosting (optional)  
    - SerpAPI key for web search tool integration  

29. **Customize External Documents:**  
    - Create and maintain Google Docs for system prompts and JSON schemas with platform-specific content requirements  
    - Update document URLs in Google Docs nodes accordingly  

30. **Test and Validate:**  
    - Test chat input reception and routing  
    - Validate AI content generation against schemas  
    - Confirm image generation and storage  
    - Verify email approval workflow  
    - Confirm publishing on all connected social media platforms  
    - Monitor Telegram notifications for workflow status  

---

This structured documentation enables developers and AI agents to fully understand, reproduce, and customize the Automated Social Media Content Publishing Factory workflow with clear guidance on each functional block, node configuration, and integration points.