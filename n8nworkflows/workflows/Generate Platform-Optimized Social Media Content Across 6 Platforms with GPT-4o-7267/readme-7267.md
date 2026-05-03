Generate Platform-Optimized Social Media Content Across 6 Platforms with GPT-4o

https://n8nworkflows.xyz/workflows/generate-platform-optimized-social-media-content-across-6-platforms-with-gpt-4o-7267


# Generate Platform-Optimized Social Media Content Across 6 Platforms with GPT-4o

### 1. Workflow Overview

This workflow automates the generation and publishing of platform-optimized social media content across six major platforms (X/Twitter, Instagram, Facebook, LinkedIn, Threads, YouTube Shorts) using GPT-4o and supporting AI tools. It is designed for content creators and marketing teams to streamline multi-channel social media campaigns by generating tailored posts, images, and metadata, then routing and optionally publishing or archiving the results.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception and Routing:** Capture user prompts from chat messages or sub-workflow triggers, and direct requests to the appropriate platform-specific content generation tools.
- **1.2 External Prompt and Schema Composition:** Retrieve system prompts and platform-specific JSON schemas from Google Docs and parse them to enforce content standards and tone.
- **1.3 AI Content Generation:** Use LangChain AI agents with GPT-4o models to generate social media content matching the platform schema and rules, incorporating web search for up-to-date relevance.
- **1.4 Dynamic Image Creation and Storage:** Generate image suggestions via Pollinations AI, upload images to third-party hosts (imgbb.com, Google Drive), and manage metadata.
- **1.5 Content Approval and Email Preparation:** Send generated content for approval via Gmail, prepare HTML email summaries for stakeholders, and handle approval logic.
- **1.6 Social Media Publishing Router:** Based on platform, route content to appropriate API nodes for posting on social media platforms.
- **1.7 Archiving and Notifications:** Save posts and images to Google Drive for archival purposes and send Telegram notifications on success or failure.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception and Routing

- **Overview:** This block receives user input via chat messages or external workflow triggers and routes the requests to platform-specific content generation tools.
- **Nodes Involved:**
  - When chat message received
  - When Executed by Another Workflow
  - 🤖Social Media Router Agent
  - X-Twiter, Instagram, Facebook, LinkedIn, Short (Threads), YouTube Short (toolWorkflow nodes)
  - Sticky Notes related to platforms and routing
- **Node Details:**

1. **When chat message received**
   - Type: LangChain chatTrigger node
   - Role: Entry point for chat-based user prompts.
   - Config: No special options, webhook-based trigger.
   - Outputs to: 🤖Social Media Router Agent.
   - Potential Failures: Webhook errors, invalid chat input.

2. **When Executed by Another Workflow**
   - Type: ExecuteWorkflowTrigger
   - Role: Triggered by other workflows; accepts `user_prompt` and `route` inputs.
   - Outputs to: Social Media System Prompt, Social Media Schema.

3. **🤖Social Media Router Agent**
   - Type: LangChain Agent
   - Role: Determines which platform-specific tool to invoke based on user prompt.
   - Config: Has system message defining rules to only call tools; tools list includes platform-specific content generators.
   - Inputs: User prompt from "When chat message received".
   - Outputs to: Platform toolWorkflow nodes (X-Twiter, Instagram, etc.).
   - Edge cases: Tool invocation errors, invalid platform names.

4. **Platform toolWorkflow nodes (e.g., X-Twiter, Instagram, etc.)**
   - Type: LangChain toolWorkflow
   - Role: Each generates social media posts for a specific platform using the user prompt.
   - Config: Receives user prompt and route as parameters, calls same workflow with route parameter.
   - Output: JSON with platform-optimized post content.
   - Failure modes: AI generation errors, invalid prompt data.

---

#### Block 1.2: External Prompt and Schema Composition

- **Overview:** Retrieves system prompts and JSON schema definitions from Google Docs and parses them for use in AI content generation to ensure consistent tone, style, and output format.
- **Nodes Involved:**
  - Social Media System Prompt (Google Docs)
  - Social Media Schema (Google Docs)
  - Parse System Prompt (Code)
  - Parse Schema (Code)
  - Merge Prompts and Schema (Merge)
  - Compose Prompt & Schema (Set)
  - Sticky Notes describing prompt & schema concepts
- **Node Details:**

1. **Social Media System Prompt**
   - Type: Google Docs
   - Role: Fetches system prompt content from a Google Doc.
   - Config: Document URL/ID to specific prompt doc.
   - Failure: Google Docs API auth, doc missing.

2. **Social Media Schema**
   - Type: Google Docs
   - Role: Fetches platform-specific JSON schema definitions.
   - Config: Document URL/ID to schema doc.
   - Failure: Google Docs API auth, doc missing.

3. **Parse System Prompt**
   - Type: Code node (JavaScript)
   - Role: Extracts multiple XML-tagged sections (e.g., system, rules, platform-specific prompts) from the system prompt document.
   - Key logic: Regex to parse XML tags and create JSON object.
   - Inputs: Raw system prompt text.
   - Outputs: Parsed system_config JSON.
   - Possible errors: Malformed XML content, missing tags.

4. **Parse Schema**
   - Type: Code node (JavaScript)
   - Role: Extracts platform-specific JSON schemas from XML content.
   - Key logic: Extracts `common`, `root`, and platform tags and parses JSON.
   - Inputs: Raw schema string and platform name.
   - Outputs: Parsed JSON schemas for validation.
   - Edge cases: Schema parse errors, invalid JSON.

5. **Merge Prompts and Schema**
   - Type: Merge node
   - Role: Combines outputs of system prompt parsing, schema parsing, and external inputs.
   - Config: Combine by position.
   - Outputs: Aggregated data for prompt composition.

6. **Compose Prompt & Schema**
   - Type: Set node
   - Role: Structures data for AI agent, including user prompt, platform route, system config, and schema.
   - Outputs: Final input for AI content generation.

---

#### Block 1.3: AI Content Generation

- **Overview:** Uses LangChain AI agents with GPT-4o-mini models to generate platform-optimized social media content, obeying the schema and system prompt rules; includes web search capability.
- **Nodes Involved:**
  - Social Media Content Creator (LangChain agent)
  - SerpAPI (LangChain tool for web search)
  - Social Content (Set)
  - Sticky Notes describing features of the content creator
- **Node Details:**

1. **Social Media Content Creator**
   - Type: LangChain Agent
   - Role: Generates social posts using system prompt, user prompt, and schema.
   - Config: Uses system message with embedded rules and tools (including web search).
   - Inputs: Composed prompt & schema.
   - Outputs: JSON object containing platform-specific social content.
   - Edge cases: API limits, incomplete or invalid AI responses, schema validation errors.

2. **SerpAPI**
   - Type: LangChain toolSerpApi node
   - Role: Provides real-time web search data to agent for context.
   - Config: Requires SerpAPI credentials.
   - Failure: API key issues, rate limits.

3. **Social Content**
   - Type: Set node
   - Role: Stores the final output from the content creator for downstream usage.

---

#### Block 1.4: Dynamic Image Creation and Storage

- **Overview:** Generates social media images based on AI suggestions, uploads images to imgbb.com and Google Drive, and merges metadata for later use.
- **Nodes Involved:**
  - pollinations.ai1 (HTTP Request to image generation API)
  - Save Image to imgbb.com (HTTP Request)
  - Save Image to Google Drive (Google Drive node)
  - Merge (combine image metadata)
  - Telegram Success Message (Optional)
  - Telegram Error Message (Optional)
  - Sticky Notes describing image creation and storage
- **Node Details:**

1. **pollinations.ai1**
   - Type: HTTP Request
   - Role: Calls pollinations.ai API to generate an image based on prompt.
   - Config: Constructs URL from AI output image suggestion string.
   - Retries 5 times on failure.
   - Output: Binary image data.

2. **Save Image to imgbb.com**
   - Type: HTTP Request (multipart-form-data)
   - Role: Uploads image binary data to imgbb.com for hosting.
   - Uses API key from environment variable.
   - Potential failures: Network issues, API key limits.

3. **Save Image to Google Drive**
   - Type: Google Drive node
   - Role: Uploads image to Google Drive folder for archival.
   - Requires Google Drive OAuth2 credentials.
   - Config: Uses sanitized post name as filename.

4. **Merge**
   - Type: Merge node
   - Role: Combines image metadata from imgbb and Google Drive for further use.

5. **Telegram Success / Error Messages**
   - Type: Telegram node
   - Role: Optional notifications to Telegram chat for success or failure of image creation.
   - Config: Uses Telegram API credentials and chat ID from environment variables.

---

#### Block 1.5: Content Approval and Email Preparation

- **Overview:** Prepares HTML email content for approval, sends to approver via Gmail with wait-for-approval logic, and checks approval status before proceeding.
- **Nodes Involved:**
  - Gmail User for Approval
  - Is Approved? (If node)
  - Prepare Email Contents (LangChain agent)
  - Sticky Notes on email preparation
- **Node Details:**

1. **Gmail User for Approval**
   - Type: Gmail node with sendAndWait operation
   - Role: Sends generated content to approver email and waits for double approval.
   - Config: Uses OAuth2 Gmail credentials.
   - Inputs: HTML email content prepared upstream.
   - Possible failures: Email send errors, timeout waiting for approval.

2. **Is Approved?**
   - Type: If node
   - Role: Checks boolean flag `approved` from Gmail response to continue workflow.
   - Outputs: True branch proceeds to image retrieval for publishing.

3. **Prepare Email Contents**
   - Type: LangChain agent
   - Role: Creates HTML email content from social content data using a provided HTML template.
   - Inputs: JSON social content.
   - Outputs: HTML string for email body.

---

#### Block 1.6: Social Media Publishing Router

- **Overview:** Routes the approved social media content to the appropriate platform API node for publishing.
- **Nodes Involved:**
  - Social Media Publishing Router (Switch)
  - X Post (Twitter node)
  - Instagram Image and Instragram Post (Facebook Graph API nodes)
  - Facebook Post (Facebook Graph API node with image upload)
  - LinkedIn Post (LinkedIn API node)
  - Implement Threads Here (NoOp placeholder)
  - Implement YouTube Shorts Here (NoOp placeholder)
  - Response Set nodes for each platform (formatting output message)
- **Node Details:**

1. **Social Media Publishing Router**
   - Type: Switch node
   - Role: Routes workflow data based on platform route string.
   - Outputs: 6 outputs for the supported platforms.
   - Edge case: Routing fails if platform string mismatches.

2. **X Post**
   - Type: Twitter node (OAuth2)
   - Role: Posts text content to X (formerly Twitter).
   - Uses OAuth2 credentials.
   - Potential failures: API limits, auth errors.

3. **Instagram Image & Instragram Post**
   - Type: HTTP Request + Facebook Graph API node
   - Role: Uploads media and publishes Instagram post.
   - Uses Facebook Graph API credentials.
   - Handles image URL from previous image generation.
   - Failure modes: API rate limits, invalid media ID.

4. **Facebook Post**
   - Type: Facebook Graph API node
   - Role: Uploads photo and posts to Facebook.
   - Sends binary image data.
   - Edge cases: API errors, permissions.

5. **LinkedIn Post**
   - Type: LinkedIn node (OAuth2)
   - Role: Posts content to LinkedIn organization page including image.
   - Configured with organization ID.
   - Failures: Auth issues, content limit exceeded.

6. **Implement Threads Here / YouTube Shorts Here**
   - Type: NoOp placeholders
   - Role: Placeholders for future implementation.

7. **Response Set nodes (X Response, Instagram Response, etc.)**
   - Type: Set nodes
   - Role: Format textual confirmation messages with post content and image thumbs.
   - Output: Used downstream for notifications or archiving.

---

#### Block 1.7: Archiving and Notifications

- **Overview:** Archives social media posts and images to Google Drive and optionally notifies via Telegram.
- **Nodes Involved:**
  - Social Post JSON (Set)
  - Save Social Post to Google Drive (Google Drive node)
  - Respond with Google Drive Id (Set)
  - Telegram Success/Failure Messages
- **Node Details:**

1. **Social Post JSON**
   - Type: Set node
   - Role: Constructs a JSON object with route, social image metadata, content, and Google Drive info.
   - Inputs: Results from merges and content nodes.

2. **Save Social Post to Google Drive**
   - Type: Google Drive node
   - Role: Saves JSON post data as text file in Google Drive folder.
   - Config: Uses OAuth2 credentials.

3. **Respond with Google Drive Id**
   - Type: Set node
   - Role: Returns Google Drive file ID as response.

4. **Telegram messages**
   - Already detailed in Block 1.4.

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                                  | Input Node(s)                             | Output Node(s)                              | Sticky Note                                                                                                                                |
|-------------------------------|-----------------------------------|-------------------------------------------------|------------------------------------------|---------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received     | LangChain chatTrigger              | Entry point for chat user prompt                 | -                                        | 🤖Social Media Router Agent                  | ## 👍Start Here                                                                                                                             |
| When Executed by Another Workflow | ExecuteWorkflowTrigger           | Sub-workflow entry point with user prompt input | -                                        | Social Media System Prompt, Social Media Schema |                                                                                                                                           |
| 🤖Social Media Router Agent      | LangChain Agent                   | Routes prompt to platform-specific tools         | When chat message received, gpt-4o       | Platform toolWorkflow nodes (X-Twiter, Instagram, etc.) | ## Social Media Router Agent                                                                                                               |
| X-Twiter                      | LangChain toolWorkflow             | Generate X/Twitter post content                   | 🤖Social Media Router Agent                | Social Media Content Creator                 | ## 1️⃣ X - Twitter                                                                                                                         |
| Instagram                     | LangChain toolWorkflow             | Generate Instagram post content                   | 🤖Social Media Router Agent                | Social Media Content Creator                 | ## 2️⃣ Instagram                                                                                                                           |
| Facebook                      | LangChain toolWorkflow             | Generate Facebook post content                    | 🤖Social Media Router Agent                | Social Media Content Creator                 | ## 3️⃣ Facebook                                                                                                                            |
| LinkedIn                      | LangChain toolWorkflow             | Generate LinkedIn post content                    | 🤖Social Media Router Agent                | Social Media Content Creator                 | ## 4️⃣ LinkedIn                                                                                                                            |
| Short                        | LangChain toolWorkflow             | Generate Threads post content                      | 🤖Social Media Router Agent                | Social Media Content Creator                 | ## 5️⃣Threads                                                                                                                              |
| YouTube Short                | LangChain toolWorkflow             | Generate YouTube Shorts content                    | 🤖Social Media Router Agent                | Social Media Content Creator                 | ## 6️⃣YouTube Shorts                                                                                                                       |
| Social Media System Prompt    | Google Docs                       | Fetch system prompt document                       | When Executed by Another Workflow          | System Prompt node                           |                                                                                                                                           |
| Social Media Schema           | Google Docs                       | Fetch platform JSON schema document                | When Executed by Another Workflow          | Schema node                                 |                                                                                                                                           |
| Parse System Prompt            | Code                             | Parse system prompt XML content                    | Social Media System Prompt                  | Merge Prompts and Schema                     |                                                                                                                                           |
| Parse Schema                  | Code                             | Parse platform schema XML content                  | Social Media Schema                          | Merge Prompts and Schema                     |                                                                                                                                           |
| Merge Prompts and Schema      | Merge                            | Combine prompt and schema parsed data              | Parse System Prompt, Parse Schema, When Executed by Another Workflow | Compose Prompt & Schema                     |                                                                                                                                           |
| Compose Prompt & Schema       | Set                              | Prepare structured prompt and schema for AI       | Merge Prompts and Schema                     | Social Media Content Creator                 |                                                                                                                                           |
| Social Media Content Creator  | LangChain Agent                  | Generate platform-specific social content          | Compose Prompt & Schema                      | Social Content                               | ## Social Media Content Creator                                                                                                           |
| SerpAPI                      | LangChain toolSerpApi            | Provide web search tool for content generation     | Social Media Content Creator                 | Social Media Content Creator                 |                                                                                                                                           |
| Social Content               | Set                              | Store AI-generated social content                   | Social Media Content Creator                 | pollinations.ai1                             |                                                                                                                                           |
| pollinations.ai1              | HTTP Request                    | Generate image based on AI image suggestion         | Social Content                               | Save Image to imgbb.com, Save Image to Google Drive | ## Create Post Image https://pollinations.ai/                                                                                              |
| Save Image to imgbb.com       | HTTP Request                    | Upload generated image to imgbb.com                 | pollinations.ai1                             | Merge                                        |                                                                                                                                           |
| Save Image to Google Drive    | Google Drive                    | Upload generated image to Google Drive              | pollinations.ai1                             | Merge                                        |                                                                                                                                           |
| Merge                        | Merge                            | Combine image metadata from multiple sources        | Save Image to imgbb.com, Save Image to Google Drive | Social Media Publishing Router, Google Drive Image Meta |                                                                                                                                           |
| Telegram Success Message (Optional) | Telegram                      | Notify success of image creation                     | pollinations.ai1                             | -                                            |                                                                                                                                           |
| Telegram Error Message (Optional) | Telegram                      | Notify failure of image creation                     | pollinations.ai1                             | -                                            |                                                                                                                                           |
| Social Media Publishing Router | Switch                         | Route content to correct platform publishing node   | Merge Image and Post Contents                | X Post, Instagram Image, Facebook Post, LinkedIn Post, Implement Threads Here, Implement YouTube Shorts Here | ## Social Media Publishing Router                                                                                                         |
| X Post                       | Twitter                         | Publish post to X/Twitter                            | Social Media Publishing Router                | X Response                                    |                                                                                                                                           |
| Instagram Image              | HTTP Request                   | Upload image to Instagram via Facebook Graph API    | Social Media Publishing Router                | Instragram Post                               |                                                                                                                                           |
| Instragram Post             | Facebook Graph API             | Publish Instagram post                               | Instagram Image                              | Instagram Response                            |                                                                                                                                           |
| Facebook Post               | Facebook Graph API             | Publish Facebook post with image                     | Social Media Publishing Router                | Facebook Response                             |                                                                                                                                           |
| LinkedIn Post               | LinkedIn API                  | Publish LinkedIn post                                | Social Media Publishing Router                | LinkedIn Response                             |                                                                                                                                           |
| Implement Threads Here       | NoOp                           | Placeholder for Threads publishing                   | Social Media Publishing Router                | -                                            |                                                                                                                                           |
| Implement YouTube Shorts Here | NoOp                           | Placeholder for YouTube Shorts publishing            | Social Media Publishing Router                | -                                            |                                                                                                                                           |
| X Response                   | Set                            | Format X post confirmation message                   | X Post                                        | -                                            |                                                                                                                                           |
| Instagram Response          | Set                            | Format Instagram post confirmation message           | Instragram Post                               | -                                            |                                                                                                                                           |
| Facebook Response           | Set                            | Format Facebook post confirmation message            | Facebook Post                                 | -                                            |                                                                                                                                           |
| LinkedIn Response           | Set                            | Format LinkedIn post confirmation message            | LinkedIn Post                                 | -                                            |                                                                                                                                           |
| Gmail User for Approval      | Gmail                         | Sends content for approval and waits for reply       | Prepare Email Contents                         | Is Approved?                                  | ## 👍 Approve Content Before Proceeding                                                                                                  |
| Is Approved?                | If                             | Conditional check on approval status                  | Gmail User for Approval                        | Get Social Post Image (if approved)           |                                                                                                                                           |
| Prepare Email Contents       | LangChain Agent               | Generates HTML email content for approval             | Merge (merged post & image data)               | Gmail User for Approval                        | ## Prepare Email Approval Contents as HTML                                                                                                |
| Get Social Post Image        | HTTP Request                  | Downloads social media post image                      | Is Approved?                                   | Merge Image and Post Contents                  |                                                                                                                                           |
| Merge Image and Post Contents | Merge                         | Combines image binary and post content data           | Extract as JSON, Get Social Post Image          | Social Media Publishing Router                 |                                                                                                                                           |
| Social Post JSON             | Set                            | Prepare JSON object for archiving                      | Google Drive Image Meta, Social Content        | Save Social Post to Google Drive                |                                                                                                                                           |
| Save Social Post to Google Drive | Google Drive                 | Saves JSON post to Google Drive folder                 | Social Post JSON                               | Respond with Google Drive Id                    | ## 9️⃣ Social Post Archiving to Google Drive                                                                                             |
| Respond with Google Drive Id | Set                            | Returns Google Drive file ID                            | Save Social Post to Google Drive                | 🤖Social Media Router Agent (loop)              |                                                                                                                                           |
| Get Social Post from Google Drive | Google Drive                 | Downloads social post JSON from Google Drive            | File Id                                         | Extract as JSON                                 |                                                                                                                                           |
| Extract as JSON             | Extract from File              | Extracts JSON content from downloaded file             | Get Social Post from Google Drive                | Merge Image and Post Contents                    |                                                                                                                                           |
| File Id                     | Set                            | Stores file ID for retrieval                            | 🤖Social Media Router Agent                      | Get Social Post from Google Drive                |                                                                                                                                           |
| Prepare Social Media Email Contents | LangChain Agent               | Creates HTML email content for sending                    | pollinations.ai1                                | Gmail                                          |                                                                                                                                           |
| Social Media Content Creator | LangChain Agent               | Generates social media content                           | Compose Prompt & Schema                         | Social Content                                  |                                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Nodes:**
   - Add a **LangChain Chat Trigger** node named `When chat message received` to accept chat input.
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow` with inputs `user_prompt` and `route`.

2. **Setup External Documents Nodes:**
   - Add two **Google Docs** nodes named `Social Media System Prompt` and `Social Media Schema`.
   - Configure each with the appropriate Google Docs document ID containing the system prompt and JSON schema respectively.
   - Connect `When Executed by Another Workflow` to both.

3. **Parse External Content:**
   - Add a **Code** node `Parse System Prompt` to extract XML-tagged sections from the system prompt document.
   - Add a **Code** node `Parse Schema` to extract platform-specific JSON schemas from the schema document.
   - Connect `Social Media System Prompt` to `Parse System Prompt`.
   - Connect `Social Media Schema` to `Parse Schema`.

4. **Merge Parsed Data:**
   - Add a **Merge** node `Merge Prompts and Schema` to combine outputs of `Parse System Prompt`, `Parse Schema`, and `When Executed by Another Workflow`.
   - Connect all three outputs accordingly.

5. **Compose Prompt for AI:**
   - Add a **Set** node `Compose Prompt & Schema` to prepare a structured object with user prompt, route, system prompts, rules, and JSON schemas.
   - Connect output of `Merge Prompts and Schema` here.

6. **Create AI Content Generation Agent:**
   - Add a **LangChain Agent** node `Social Media Content Creator`.
   - Configure it with a system message referencing system prompt, rules, and schema.
   - Connect `Compose Prompt & Schema` to this node.

7. **Add Web Search Tool:**
   - Add **LangChain SerpAPI tool** node connected as a tool for `Social Media Content Creator` to provide real-time search.

8. **Store AI Output:**
   - Add a **Set** node `Social Content` to store the generated social content JSON.
   - Connect `Social Media Content Creator` to `Social Content`.

9. **Image Generation:**
   - Add an **HTTP Request** node `pollinations.ai1` calling Pollinations AI API using the image suggestion from social content.
   - Connect `Social Content` to `pollinations.ai1`.

10. **Save Image to Hosting:**
    - Add an **HTTP Request** node `Save Image to imgbb.com` to upload the image binary.
    - Connect `pollinations.ai1` to this node.

11. **Save Image to Google Drive:**
    - Add a **Google Drive** node `Save Image to Google Drive` configured with desired folder.
    - Connect `pollinations.ai1` to this node.

12. **Merge Image Metadata:**
    - Add a **Merge** node to combine outputs from `Save Image to imgbb.com` and `Save Image to Google Drive`.
    - Connect both image saving nodes to this merge.

13. **Optional Notifications:**
    - Add **Telegram** nodes for success and error notifications connected from `pollinations.ai1`.

14. **Prepare Email Content:**
    - Add a **LangChain Agent** node `Prepare Email Contents` with an HTML template prompt.
    - Connect the merged social content and image metadata to this node.

15. **Send Email for Approval:**
    - Add a **Gmail** node `Gmail User for Approval` configured with OAuth2 credentials.
    - Set to send and wait for approval with double confirmation enabled.
    - Connect `Prepare Email Contents` to this node.

16. **Check Approval:**
    - Add an **If** node `Is Approved?` checking the boolean approval flag from Gmail node.
    - On true, continue; on false, stop or notify.

17. **Download Post Image:**
    - Add an **HTTP Request** node `Get Social Post Image` to download image for publishing.
    - Connect from `Is Approved?` true path.

18. **Merge Image and Post Content:**
    - Add a **Merge** node to combine downloaded image and social content JSON.
    - Connect `Extract as JSON` (see next step) and `Get Social Post Image`.

19. **Extract Social Post JSON:**
    - Add **Google Drive** node `Get Social Post from Google Drive` to fetch saved post JSON.
    - Connect `File Id` (sets file ID) to this node.
    - Add **Extract from File** node to parse JSON content.
    - Connect Google Drive output to extraction.

20. **Social Media Publishing Router:**
    - Add a **Switch** node `Social Media Publishing Router`.
    - Configure rules to route based on platform route string to:
      - Twitter node (X Post)
      - Instagram nodes (Instagram Image + Instragram Post)
      - Facebook node
      - LinkedIn node
      - NoOp nodes for Threads and YouTube Shorts placeholders.

21. **Configure Social Media API Nodes:**
    - Set up OAuth2 credentials for Twitter, Facebook Graph API, LinkedIn.
    - Configure each node to post using content and image URLs or binaries.
    - Connect switch outputs accordingly.

22. **Set Response Messages:**
    - Add **Set** nodes to format confirmation messages for each platform after posting.

23. **Archive Social Post:**
    - Add **Set** node `Social Post JSON` to prepare JSON for archiving.
    - Add **Google Drive** node `Save Social Post to Google Drive` to save JSON object.
    - Add **Set** node `Respond with Google Drive Id` to output saved file ID.

24. **Final Steps:**
    - Connect `Save Social Post to Google Drive` output to `Respond with Google Drive Id`.
    - Connect this ID back to `🤖Social Media Router Agent` for potential reuse or chaining.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Update all Social Media Platform Credentials as required. Adjust parameters and content for each platform to suit your needs.                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note41                                 |
| Replace pollinations.ai with any online image generation service that produces an image file you can download.                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note44                                 |
| Replace Chat model with other LLMs and test out the results. Add more tools or try other web search tools to suit your use case.                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note45                                 |
| Create Google Doc for the Social Media Schema and copy the provided JSON schema. Update the Google Doc ID in the Social Media Schema node. Create Google Doc for the Social Media System Prompt and copy the provided system prompt. Update the Google Doc ID in the Social Media System Prompt node. Adjust system prompt and platform specific prompts to suit your needs.                                                                                                                                                | Sticky Note43                                 |
| Social Media System Prompt and Schema documents are key to managing tone, style, and output formats centrally, enabling easy updates and collaboration without workflow code changes.                                                                                                                                                                                                                                                                                                                         | Sticky Notes 5, 12, 16, 17                     |
| Workflow uses Gmail OAuth2 credentials for approval email sending and waiting for approval with double confirmation, ensuring human oversight before publishing.                                                                                                                                                                                                                                                                                                                                             | Gmail User for Approval node                   |
| Telegram API node is integrated optionally for real-time workflow success and error notifications.                                                                                                                                                                                                                                                                                                                                                                                                             | Telegram Success and Error Message nodes       |
| Supported platforms: X/Twitter, Instagram, Facebook, LinkedIn, Threads (placeholder), YouTube Shorts (placeholder). Add implementations for Threads and YouTube Shorts as needed.                                                                                                                                                                                                                                                                                                                           | Social Media Publishing Router and placeholders |
| Pollinations AI image generation API is used to create visual content matching post themes, improving engagement with custom visuals.                                                                                                                                                                                                                                                                                                                                                                         | pollinations.ai1 HTTP Request node             |
| The workflow enforces strict JSON schema compliance for all generated content to minimize errors and maintain platform standards.                                                                                                                                                                                                                                                                                                                                                                           | Schema parsing and system prompt rules         |
| System prompt includes detailed style, tone, and content guidelines per platform to maintain consistent branding and audience engagement.                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note1 and System Prompt node             |

---

**Disclaimer:** The content provided originates exclusively from an n8n automated workflow. All processing adheres strictly to content policies, containing no illegal, offensive, or protected material. All data handled is legal and public.