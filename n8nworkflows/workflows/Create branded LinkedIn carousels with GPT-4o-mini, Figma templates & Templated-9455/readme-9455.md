Create branded LinkedIn carousels with GPT-4o-mini, Figma templates & Templated

https://n8nworkflows.xyz/workflows/create-branded-linkedin-carousels-with-gpt-4o-mini--figma-templates---templated-9455


# Create branded LinkedIn carousels with GPT-4o-mini, Figma templates & Templated

---

### 1. Workflow Overview

This workflow automates the creation and publishing of branded LinkedIn carousel posts using AI-generated content and structured templates. It orchestrates a daily content generation pipeline that leverages GPT-4o-mini for writing posts, integrates external visual template services (Templated MCP), prepares media assets, uploads them to LinkedIn, and notifies via Telegram.

**Target Use Cases:**  
- Automated daily LinkedIn content marketing for business owners and AI automation experts.  
- Conversion of AI-written LinkedIn posts into visually compelling carousel slideposts.  
- Seamless integration with LinkedIn APIs for media upload and post creation.  
- Real-time notification of success or failure of post publication.

**Logical Blocks:**

- **1.1 Daily Trigger & Context Initialization:** Triggers workflow daily and sets up memory/context for AI tools.  
- **1.2 LinkedIn Post Writing:** Uses GPT-4o-mini and Perplexity research tool to generate a compliant LinkedIn post.  
- **1.3 Carousel Ideation & Template Selection:** Transforms the LinkedIn post into a structured slidepost request using Templated MCP client.  
- **1.4 Carousel Generation & File Preparation:** Sends the slidepost request to Templated API, extracts binary content, and prepares for LinkedIn upload.  
- **1.5 LinkedIn Media Upload & Post Creation:** Initializes LinkedIn document upload, uploads binary carousel files, retrieves file URN, and publishes the LinkedIn carousel post.  
- **1.6 Status Handling & Notification:** Monitors processing status, handles errors or waits for completion, and sends Telegram notifications about success or failure.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger & Context Initialization

**Overview:**  
Initiates the workflow once per day, establishing session keys for memory nodes that maintain conversational context for AI tools.

**Nodes Involved:**  
- Schedule Trigger  
- Simple Memory (two instances)  
- Date & Time  

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger  
  - Configuration: Interval trigger scheduled to run daily (default interval)  
  - Inputs: None (start node)  
  - Outputs: Initiates the LinkedIn Writer node  
  - Failures: Misconfiguration could prevent triggering; ensure server time zone aligns with intended schedule.

- **Simple Memory (2 nodes)**  
  - Type: LangChain memoryBufferWindow  
  - Configuration: Session key based on current date string (`dd-LL-yyyy`), custom sessionIdType  
  - Purpose: Maintains AI chat context per day to enable coherent multi-turn interactions  
  - Inputs: Feed data from upstream AI nodes  
  - Outputs: Passes memory buffer to AI nodes  
  - Failures: Session key misformatting or memory overflow possible if memory grows too large.

- **Date & Time**  
  - Type: DateTime Tool  
  - Role: Provides current date/time information to AI nodes  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Passes date/time info to LinkedIn Writer node  
  - Failures: None expected unless system clock is incorrect.

---

#### 2.2 LinkedIn Post Writing

**Overview:**  
Generates an engaging, properly formatted LinkedIn post on AI automation based on latest news and research, enforcing strict formatting rules and tone.

**Nodes Involved:**  
- Linkedin Writer (LangChain Agent)  
- OpenAI Chat Model (gpt-4o-mini)  
- Search latest news (Perplexity Tool)  
- Think (LangChain Tool Think)  
- Simple Memory (memory input for LinkedIn Writer)  
- Date & Time (date input for LinkedIn Writer)  
- Sticky Note2 (comment explaining block)

**Node Details:**

- **Linkedin Writer**  
  - Type: LangChain Agent node  
  - Configuration:  
    - System message sets expert content marketing persona  
    - Detailed instructions on post structure, style, formatting, and banned markdown syntax  
    - References Perplexity tool for research and enforces output validation  
  - Inputs: Date & Time, Perplexity Tool, Think tool, Simple Memory  
  - Outputs: Post content passed to Carousel Ideator  
  - Failures:  
    - AI generation could fail due to OpenAI API errors or rate limits  
    - Formatting rules might be violated if AI outputs unexpected text  
    - Memory or session key errors could disrupt context

- **OpenAI Chat Model**  
  - Type: Language Model node using GPT-4o-mini  
  - Configuration: Model set to "gpt-4o-mini" for chat completion  
  - Inputs: Connected from LinkedIn Writer for language generation tasks  
  - Outputs: Chat completions feeding back to LinkedIn Writer and Carousel Ideator  
  - Failures: API authentication errors, timeouts, or quota exceeded

- **Search latest news (Perplexity Tool)**  
  - Type: AI external tool for researching latest news  
  - Configuration: Uses Perplexity API with provided credentials  
  - Inputs: Query generated by LinkedIn Writer agent  
  - Outputs: Research results for LinkedIn Writer context  
  - Failures: API errors, network issues, or invalid query format

- **Think (LangChain Tool Think)**  
  - Type: AI tool for internal reasoning or intermediate thoughts  
  - Inputs/Outputs: Helps LinkedIn Writer with reasoning steps  
  - Failures: Rare, mostly tied to AI service availability

---

#### 2.3 Carousel Ideation & Template Selection

**Overview:**  
Transforms the AI-generated LinkedIn post text into a structured JSON request that defines the slidepost carousel using available visual templates via Templated MCP server.

**Nodes Involved:**  
- Carousel Ideator (LangChain Agent)  
- MCP Client (Templated MCP tool)  
- Structured Output Parser (LangChain output parser)  
- Simple Memory1 (memory buffer for Carousel Ideator)  
- Sticky Note (comment about carousel ideator)  
- Sticky Note3 (comment about carousel generation)

**Node Details:**

- **Carousel Ideator**  
  - Type: LangChain Agent node  
  - Configuration:  
    - System message instructs it to analyze LinkedIn post content and produce structured slidepost JSON  
    - Enforces rules on template selection, slide flow, concise content, and valid JSON output  
  - Inputs: Output from LinkedIn Writer, memory buffer, MCP Client templates data  
  - Outputs: JSON request for carousel generation  
  - Failures:  
    - JSON generation errors (invalid syntax)  
    - Missing required fields for templates  
    - API rate limits on MCP server

- **MCP Client**  
  - Type: LangChain MCP Client Tool  
  - Configuration: Connects to external Templated MCP API with bearer token authentication  
  - Purpose: Fetches available post templates and assists Carousel Ideator with template info  
  - Inputs: Requests from Carousel Ideator  
  - Outputs: Template metadata for ideation  
  - Failures: Authentication errors, network issues, or invalid responses

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Configuration: Uses JSON schema example to validate and parse Carousel Ideator’s JSON output  
  - Inputs: Raw output from Carousel Ideator  
  - Outputs: Parsed structured JSON for downstream usage  
  - Failures: Parsing errors if output is malformed or schema mismatched

---

#### 2.4 Carousel Generation & File Preparation

**Overview:**  
Sends the structured carousel JSON request to the Templated API to generate the actual media files, extracts the binary content, and prepares it for upload to LinkedIn.

**Nodes Involved:**  
- Generate posts (HTTP Request)  
- Extract from File (Extract from binary)  
- Edit Fields (Set attachment property)  
- Convert to binary (Convert attachment to binary)  
- Sticky Note1 (comment about preparing LinkedIn post resource)

**Node Details:**

- **Generate posts**  
  - Type: HTTP Request  
  - Configuration:  
    - POST to Templated API endpoint `/api/batches/4/generate`  
    - Sends JSON body with carousel JSON output under `output` property  
    - Authenticated with bearer token  
  - Inputs: JSON from Carousel Ideator  
  - Outputs: Binary file data of generated carousel  
  - Failures: HTTP errors, invalid request payload, auth failures

- **Extract from File**  
  - Type: ExtractFromFile node  
  - Configuration: Converts binary data from HTTP response into a string property  
  - Inputs: Binary data from Generate posts  
  - Outputs: JSON containing file data string  
  - Failures: Malformed binary data or conversion errors

- **Edit Fields**  
  - Type: Set node  
  - Configuration: Assigns the extracted file data string to a property named "attachment" for further processing  
  - Inputs: Extract from File output  
  - Outputs: Modified JSON with attachment  
  - Failures: Incorrect property naming causing downstream failures

- **Convert to binary**  
  - Type: ConvertToFile  
  - Configuration: Converts the "attachment" string property back into binary file data suitable for upload  
  - Inputs: JSON with attachment string  
  - Outputs: Binary data for upload  
  - Failures: Conversion failure if attachment is malformed

---

#### 2.5 LinkedIn Media Upload & Post Creation

**Overview:**  
Handles LinkedIn API interactions to upload the carousel media, create a post referencing the uploaded media, and retrieve necessary URNs for these operations.

**Nodes Involved:**  
- Get Linkedin user info (HTTP Request)  
- Initialize upload URN (HTTP Request)  
- Upload Posts as binary (HTTP Request)  
- Get uploaded file URN (HTTP Request)  
- Create Linkedin post (HTTP Request)  
- Switch (status check and routing)  
- Wait (delayed retry)  
- Sticky Note5 (comment about LinkedIn post creation)

**Node Details:**

- **Get Linkedin user info**  
  - Type: HTTP Request  
  - Configuration: GET `https://api.linkedin.com/v2/userinfo` with OAuth2 LinkedIn credentials  
  - Purpose: Retrieve LinkedIn user URN (person ID) to set ownership of uploads  
  - Outputs: User info with `sub` (person urn suffix)  
  - Failures: Auth errors, expired tokens

- **Initialize upload URN**  
  - Type: HTTP Request  
  - Configuration: POST to LinkedIn `/rest/documents?action=initializeUpload` to start upload session  
  - JSON body sets owner as `urn:li:person:<sub>` from user info  
  - Outputs: Upload URL and document URN for subsequent upload  
  - Failures: Auth errors, invalid URN, API limits

- **Upload Posts as binary**  
  - Type: HTTP Request  
  - Configuration: POST binary data to upload URL from previous step, sets LinkedIn headers  
  - Inputs: Binary file from Convert to binary  
  - Outputs: Upload confirmation  
  - Failures: Upload failures, network errors, incorrect content type

- **Get uploaded file URN**  
  - Type: HTTP Request  
  - Configuration: GET document info from LinkedIn API using document URN  
  - Outputs: Media URN needed for post creation  
  - Failures: Document not found, permissions

- **Create Linkedin post**  
  - Type: HTTP Request  
  - Configuration: POST to LinkedIn `/rest/posts`  
  - JSON body constructs post referencing uploaded media URN, author, commentary (post text), and visibility  
  - Inputs: Post content from LinkedIn Writer, media URN  
  - Outputs: Post creation confirmation  
  - Failures: API errors, invalid JSON, permission denied

- **Switch**  
  - Type: Switch node  
  - Configuration: Routes workflow based on LinkedIn document status (`AVAILABLE`, `PROCESSING`, `PROCESSING_FAILED`)  
  - Outputs:  
    - AVAILABLE: Proceed to Create Linkedin post  
    - PROCESSING_FAILED: Send error notification  
    - PROCESSING: Wait and retry  
  - Failures: Unexpected status values, infinite loops if retry logic not capped

- **Wait**  
  - Type: Wait node  
  - Configuration: Waits 3 seconds before rechecking status to avoid API rate limit or excessive polling  
  - Inputs: From Switch for PROCESSING status  
  - Outputs: Loops back to Get uploaded file URN  
  - Failures: Delays may accumulate if processing takes too long

---

#### 2.6 Status Handling & Notification

**Overview:**  
Sends Telegram notifications based on the success or failure of the LinkedIn post creation process.

**Nodes Involved:**  
- Success (Telegram)  
- Error (Telegram)  
- Sticky Note6 (comment about Telegram notification)

**Node Details:**

- **Success**  
  - Type: Telegram node  
  - Configuration: Sends "Created new post" message to configured Telegram bot/channel  
  - Inputs: Triggered on successful post creation  
  - Failures: Telegram API errors, invalid chat ID

- **Error**  
  - Type: Telegram node  
  - Configuration: Sends "Failed to create new post" message to Telegram bot/channel  
  - Inputs: Triggered on processing failure status from Switch node  
  - Failures: Telegram API errors, invalid chat ID

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                               | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                      |
|-------------------------|---------------------------------|----------------------------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                 | Daily trigger to start workflow               | None                             | Linkedin Writer                   | ### Daily trigger Triggers this workflow daily for a daily LinkedIn post                       |
| Simple Memory           | LangChain Memory Buffer          | Maintains context for LinkedIn Writer         | Linkedin Writer                  | Linkedin Writer                   |                                                                                                |
| Date & Time             | DateTime Tool                   | Provides current date/time                      | Schedule Trigger                 | Linkedin Writer                   |                                                                                                |
| Search latest news      | Perplexity Tool                 | Research latest news for post                   | Linkedin Writer                  | Linkedin Writer                   |                                                                                                |
| Think                   | LangChain Tool Think             | AI reasoning helper                            | Linkedin Writer                  | Linkedin Writer                   |                                                                                                |
| OpenAI Chat Model       | LangChain LM Chat OpenAI         | GPT-4o-mini model for language generation      | Linkedin Writer, Carousel Ideator| Linkedin Writer, Carousel Ideator |                                                                                                |
| Linkedin Writer         | LangChain Agent                  | Generates LinkedIn post text                    | Schedule Trigger, Date & Time, Perplexity, Think, Simple Memory | Carousel Ideator  | ### LinkedIn post writer Uses the **OpenAI** model to write the post, optionally triggers **Perplexity** tool to research the topic it wants to make the post about |
| Simple Memory1          | LangChain Memory Buffer          | Maintains context for Carousel Ideator         | Carousel Ideator                | Carousel Ideator                 |                                                                                                |
| MCP Client              | LangChain MCP Client Tool        | Fetches templates metadata from Templated MCP | Carousel Ideator                | Carousel Ideator                 |                                                                                                |
| Carousel Ideator        | LangChain Agent                  | Generates structured slidepost JSON request    | Linkedin Writer, Simple Memory1, MCP Client, Structured Output Parser | Generate posts | ### Carousel ideator (using [Templated](https://templated.cometai.eu)) Connects to the Templated MCP server to list all available post templates and builds a logical carousel flow using available templates. |
| Structured Output Parser| LangChain Output Parser Structured| Parses Carousel Ideator's JSON output          | Carousel Ideator                | Carousel Ideator                 |                                                                                                |
| Generate posts          | HTTP Request                    | Sends slidepost JSON to Templated API          | Carousel Ideator                | Extract from File                | ### Generate the Carousel (https://templated.cometai.eu) Uses the request generated by Carousel Ideator |
| Extract from File       | ExtractFromFile                 | Extracts binary response content                | Generate posts                 | Edit Fields                    |                                                                                                |
| Edit Fields             | Set                            | Assigns extracted file data to 'attachment'    | Extract from File              | Convert to binary              | ## Prepare LinkedIn post resource Takes the user information, prepares files and uploads them to LinkedIn as a carousel. |
| Convert to binary       | ConvertToFile                  | Converts attachment string to binary            | Edit Fields                   | Upload Posts as binary         |                                                                                                |
| Get Linkedin user info  | HTTP Request                   | Retrieves LinkedIn user URN                      | Extract from File              | Initialize upload URN          |                                                                                                |
| Initialize upload URN   | HTTP Request                   | Initializes LinkedIn document upload session    | Get Linkedin user info         | Edit Fields                   |                                                                                                |
| Upload Posts as binary  | HTTP Request                   | Uploads carousel binary data to LinkedIn        | Convert to binary              | Get uploaded file URN          |                                                                                                |
| Get uploaded file URN   | HTTP Request                   | Retrieves uploaded file document URN            | Upload Posts as binary         | Switch                       |                                                                                                |
| Switch                  | Switch                        | Routes based on LinkedIn document status        | Get uploaded file URN          | Create Linkedin post, Error, Wait |                                                                                                |
| Wait                    | Wait                         | Waits 3 seconds before retrying status          | Switch                       | Get uploaded file URN          |                                                                                                |
| Create Linkedin post    | HTTP Request                   | Creates LinkedIn post with uploaded carousel    | Switch                       | Success                      | ## Create LinkedIn post Takes the carousel resource from the next steps and creates a LinkedIn post with the generated content. |
| Success                 | Telegram                      | Sends success notification                       | Create Linkedin post          | None                        | ## Telegram notification Sends a message whether the post was successful or not.               |
| Error                   | Telegram                      | Sends failure notification                       | Switch                       | None                        |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger once daily (default interval)  
   - Connect output to LinkedIn Writer node.

2. **Create Date & Time node**  
   - Type: DateTime Tool  
   - Connect input from Schedule Trigger  
   - Connect output to LinkedIn Writer node.

3. **Create Simple Memory node (for LinkedIn Writer)**  
   - Type: LangChain memoryBufferWindow  
   - Set Session Key: `={{ $now.format('dd-LL-yyyy') }}`  
   - Set SessionIdType: CustomKey  
   - Connect input from LinkedIn Writer  
   - Connect output back to LinkedIn Writer for memory context.

4. **Create Search latest news node**  
   - Type: Perplexity Tool  
   - Configure with Perplexity API credentials  
   - Input message: from LinkedIn Writer as prompt  
   - Connect output to LinkedIn Writer node.

5. **Create Think node**  
   - Type: LangChain Tool Think  
   - Connect input and output to LinkedIn Writer for reasoning steps.

6. **Create OpenAI Chat Model node**  
   - Type: LangChain lmChatOpenAi  
   - Model: `gpt-4o-mini`  
   - Configure OpenAI API credentials  
   - Connect as AI language model to LinkedIn Writer and Carousel Ideator.

7. **Create LinkedIn Writer node**  
   - Type: LangChain Agent  
   - Paste provided system message with detailed instructions on writing style, format, and structure.  
   - Set prompt text to "Create a Linkedin post."  
   - Connect inputs from Schedule Trigger, Date & Time, Perplexity Tool, Think, Simple Memory, OpenAI Chat Model.  
   - Connect output to Carousel Ideator node.

8. **Create Simple Memory1 node (for Carousel Ideator)**  
   - Same configuration as Simple Memory but connected to Carousel Ideator for session context.

9. **Create MCP Client node**  
   - Type: LangChain MCP Client Tool  
   - Endpoint URL: `https://templated.cometai.eu/mcp`  
   - Authentication: Bearer Auth with Templated Bearer token  
   - Connect input from Carousel Ideator.

10. **Create Structured Output Parser node**  
    - Type: LangChain Output Parser Structured  
    - Paste JSON schema example for carousel items  
    - Connect input from Carousel Ideator.

11. **Create Carousel Ideator node**  
    - Type: LangChain Agent  
    - Paste provided system message instructing slidepost JSON generation rules.  
    - Input from LinkedIn Writer, Simple Memory1, MCP Client, and Structured Output Parser.  
    - Output connects to Generate posts node.

12. **Create Generate posts node**  
    - Type: HTTP Request  
    - URL: `https://templated.cometai.eu/api/batches/4/generate`  
    - Method: POST  
    - Body: JSON, send `={{ $json.output }}` from Carousel Ideator  
    - Auth: HTTP Bearer with Templated Bearer token  
    - Output connects to Extract from File node.

13. **Create Extract from File node**  
    - Type: Extract From File (binaryToProperty operation)  
    - Input from Generate posts  
    - Output connects to Get Linkedin user info and Edit Fields nodes.

14. **Create Get Linkedin user info node**  
    - Type: HTTP Request  
    - URL: `https://api.linkedin.com/v2/userinfo`  
    - Auth: LinkedIn OAuth2 credentials  
    - Headers: `LinkedIn-Version: 202509`, `X-Restli-Protocol-Version: 2.0.0`  
    - Output connects to Initialize upload URN node.

15. **Create Initialize upload URN node**  
    - Type: HTTP Request  
    - URL: `https://api.linkedin.com/rest/documents?action=initializeUpload`  
    - Method: POST  
    - Body: JSON with owner urn from user info (`urn:li:person:{{ $json.sub }}`)  
    - Auth: LinkedIn OAuth2 credentials  
    - Output connects to Edit Fields node.

16. **Create Edit Fields node**  
    - Type: Set node  
    - Set property `attachment` to extracted file data string  
    - Input from Initialize upload URN and Extract from File  
    - Output connects to Convert to binary node.

17. **Create Convert to binary node**  
    - Type: Convert To File  
    - Operation: toBinary from `attachment` property  
    - Output connects to Upload Posts as binary node.

18. **Create Upload Posts as binary node**  
    - Type: HTTP Request  
    - URL: Dynamic from Initialize upload URN uploadUrl  
    - Method: POST  
    - Content-Type: binaryData  
    - Input binary data from Convert to binary node  
    - Auth: LinkedIn OAuth2 credentials  
    - Output connects to Get uploaded file URN node.

19. **Create Get uploaded file URN node**  
    - Type: HTTP Request  
    - URL: `https://api.linkedin.com/rest/documents/{{ upload document URN }}`  
    - Auth: LinkedIn OAuth2 credentials  
    - Output connects to Switch node.

20. **Create Switch node**  
    - Conditions on `$json.status` with values:  
      - "AVAILABLE" → Create Linkedin post  
      - "PROCESSING_FAILED" → Error node  
      - "PROCESSING" → Wait node  
    - Output accordingly.

21. **Create Wait node**  
    - Type: Wait  
    - Duration: 3 seconds  
    - Output loops back to Get uploaded file URN node.

22. **Create Create Linkedin post node**  
    - Type: HTTP Request  
    - URL: `https://api.linkedin.com/rest/posts`  
    - Method: POST  
    - Body: JSON referencing uploaded media URN, author, commentary from LinkedIn Writer output, visibility set to PUBLIC  
    - Auth: LinkedIn OAuth2 credentials  
    - Output connects to Success node.

23. **Create Success node**  
    - Type: Telegram node  
    - Sends "Created new post" message  
    - Auth: Telegram Bot API credentials

24. **Create Error node**  
    - Type: Telegram node  
    - Sends "Failed to create new post" message  
    - Auth: Telegram Bot API credentials

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                            | Context or Link                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Workflow branding and details: This workflow automates LinkedIn carousel posts leveraging GPT-4o-mini, Perplexity research, and Templated visual templates.                                                                            | Workflow title and description                       |
| Sticky Note on Carousel Ideator references [Templated](https://templated.cometai.eu) — official site for carousel template generation.                                                                                                | https://templated.cometai.eu                        |
| LinkedIn API version used: 202509, with REST APIs for documents and posts, requires OAuth2 credentials configured with proper scopes.                                                                                                | LinkedIn API documentation                          |
| Telegram notifications provide real-time feedback on success or failure of post creation, useful for monitoring automated publishing status.                                                                                        | Telegram Bot API                                     |
| Strict formatting rules for LinkedIn posts prevent markdown usage and enforce a conversational, story-driven style with Unicode formatting for emphasis.                                                                              | Inside Linkedin Writer system message                |
| The workflow uses LangChain nodes for AI orchestration, requiring n8n version supporting LangChain nodes and external AI tools integration.                                                                                         | n8n documentation on LangChain nodes                |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---