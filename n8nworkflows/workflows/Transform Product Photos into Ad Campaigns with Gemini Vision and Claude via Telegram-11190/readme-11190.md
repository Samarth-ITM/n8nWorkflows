Transform Product Photos into Ad Campaigns with Gemini Vision and Claude via Telegram

https://n8nworkflows.xyz/workflows/transform-product-photos-into-ad-campaigns-with-gemini-vision-and-claude-via-telegram-11190


# Transform Product Photos into Ad Campaigns with Gemini Vision and Claude via Telegram

### 1. Workflow Overview

This workflow automates the transformation of product photos sent via Telegram into complete ad campaign concepts with visual mock-ups, leveraging Google Gemini Vision and Claude AI models. It targets marketers and creatives who want to rapidly generate AI-driven advertising ideas and images directly from mobile or desktop Telegram chats.

The workflow is logically divided into three main blocks:

- **1.1 Receive & Authenticate:** Listens for incoming Telegram messages, verifies authorized user, and extracts the product photo.
- **1.2 AI Analysis & Strategy:** Runs Gemini Vision to analyze the photo, then Claude (via OpenRouter) to generate creative ad concepts and image prompts.
- **1.3 Generate & Deliver:** Downloads the original photo, uses Gemini’s Image Generation API to create ad visuals, and sends the results back to Telegram.

This modular structure ensures secure usage, sophisticated AI-driven creative output, and seamless user experience within Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Block: Receive & Authenticate

- **Overview:**  
  This block triggers on new Telegram messages, verifies the sender’s identity, checks that a photo is included, and retrieves the photo's file path for processing.

- **Nodes Involved:**  
  - On New Message  
  - CONFIG  
  - Auth: Verify User ID  
  - Check: Has Photo?  
  - Get Photo File Path  

- **Node Details:**

  - **On New Message**  
    - Type: Telegram Trigger  
    - Role: Entry point listening for all new messages sent to the Telegram bot.  
    - Config: Listens to "message" updates; uses Telegram Bot Token credential.  
    - Inputs: Telegram webhook trigger  
    - Outputs: Message JSON including user info & media  
    - Edge cases: No message received, Telegram API downtime, webhook misconfiguration.  

  - **CONFIG**  
    - Type: Set  
    - Role: Stores workflow configuration, primarily `authorized_user_id` for user validation.  
    - Config: Single string variable `authorized_user_id` to be manually set.  
    - Inputs: Trigger output  
    - Outputs: Passes config data downstream  
    - Edge cases: Missing or incorrect user ID causes auth failure.  

  - **Auth: Verify User ID**  
    - Type: If  
    - Role: Compares message sender ID against `authorized_user_id` to ensure access control.  
    - Config: Expression comparing `{{$json.message.from.id}}` to config value.  
    - Inputs: CONFIG output  
    - Outputs: Passes only if user matches; otherwise workflow stops here.  
    - Edge cases: Unauthorized users, expression evaluation errors.  

  - **Check: Has Photo?**  
    - Type: If  
    - Role: Verifies that the incoming Telegram message contains at least one photo attachment.  
    - Config: Checks existence of `$json.message.photo` array.  
    - Inputs: Auth success path  
    - Outputs: Passes only if photo exists, else stops.  
    - Edge cases: Messages without photos, other media types.  

  - **Get Photo File Path**  
    - Type: Telegram Node  
    - Role: Retrieves the file path for the photo from Telegram servers using the photo file ID.  
    - Config: Uses second element of photo array (`photo[1].file_id`) for medium resolution photo.  
    - Inputs: Photo-validated message  
    - Outputs: JSON including Telegram file path for the photo  
    - Edge cases: Telegram API errors, invalid file ID, network issues.  

---

#### 2.2 Block: AI Analysis & Strategy

- **Overview:**  
  This block sends the photo to Gemini Vision for image analysis, then uses Claude AI to generate an ad campaign concept and detailed prompts based on that analysis plus any user caption.

- **Nodes Involved:**  
  - Gemini Vision: Analyze  
  - OpenRouter Chat Model  
  - think_tool  
  - Creative Agent  
  - Structured Output Parser  
  - Loop: Ad Variations  

- **Node Details:**

  - **Gemini Vision: Analyze**  
    - Type: Google Gemini Vision (LangChain Node)  
    - Role: Analyzes the product photo (binary image input) to extract product, context, and lighting details.  
    - Config: Model `models/gemini-pro-vision`, operation `analyze`, inputType `binary`.  
    - Inputs: Photo file path from Telegram (binary data)  
    - Outputs: Textual description of image content  
    - Credentials: Google Gemini API key  
    - Edge cases: API quota limits, invalid image data, network timeouts.  

  - **OpenRouter Chat Model**  
    - Type: AI Language Model (Claude via OpenRouter)  
    - Role: Provides AI language generation capabilities integrated with the Creative Agent node.  
    - Config: Model set to `anthropic/claude-sonnet-4.5`.  
    - Inputs: Text prompts from Creative Agent  
    - Outputs: AI-generated text responses  
    - Credentials: OpenRouter API key  
    - Edge cases: API limits, authentication errors, model downtime.  

  - **think_tool**  
    - Type: LangChain Think Tool  
    - Role: AI tool interface to invoke the Creative Agent with Gemini Vision analysis and user caption.  
    - Config: No additional parameters, invoked by Creative Agent.  
    - Inputs: Gemini Vision analysis text and Telegram message caption  
    - Outputs: Structured AI thoughts and campaign suggestions  
    - Edge cases: AI processing errors, malformed input.  

  - **Creative Agent**  
    - Type: LangChain Agent  
    - Role: Acts as an AI Art Director, generating multiple ad campaign concepts and image prompts.  
    - Config: System prompt defines agent identity and instructions; Input includes user caption and Gemini Vision analysis.  
    - Inputs: Gemini Vision output + Telegram message caption  
    - Outputs: JSON with `agent_thought_process` and array `ad_campaigns` each with fields like `concept_title`, `marketing_rationale`, `nbp_prompt`, and `aspect_ratio`.  
    - Edge cases: Parsing errors, incomplete AI output.  

  - **Structured Output Parser**  
    - Type: LangChain Output Parser (Structured JSON)  
    - Role: Parses AI output into a strict JSON schema for downstream processing.  
    - Config: Example JSON schema with `agent_thought_process` and `ad_campaigns` array.  
    - Inputs: Raw AI text output from Creative Agent  
    - Outputs: Validated structured JSON  
    - Edge cases: Parsing failures, schema mismatches.  

  - **Loop: Ad Variations**  
    - Type: Split Out  
    - Role: Iterates over each ad campaign variation in the parsed JSON to process each concept independently.  
    - Config: Splits on `output.ad_campaigns` array field.  
    - Inputs: Structured JSON parsed ad campaigns  
    - Outputs: Single ad campaign per iteration  
    - Edge cases: Empty campaigns array, iteration errors.  

---

#### 2.3 Block: Generate & Deliver

- **Overview:**  
  For each ad campaign variation, downloads the original photo, generates a new ad image using Gemini Image Generation API with the AI prompt, converts the response to a file, and sends it back to the user on Telegram.

- **Nodes Involved:**  
  - Config: Source URL  
  - HTTP: Download Source  
  - Data: Binary Prep  
  - HTTP: Generate Image  
  - Data: Convert Response  
  - Telegram: Send Result  

- **Node Details:**

  - **Config: Source URL**  
    - Type: Set  
    - Role: Constructs the direct URL to download the product photo from Telegram servers using the bot token and file path.  
    - Config: URL template: `https://api.telegram.org/file/bot{{ telegram_bot_token }}/{{ file_path }}`  
    - Inputs: Current campaign iteration + photo file path  
    - Outputs: Object with `input_url` string  
    - Edge cases: Incorrect token or file path leads to broken URL.  

  - **HTTP: Download Source**  
    - Type: HTTP Request  
    - Role: Downloads the original product photo binary data from Telegram.  
    - Config: GET request to `input_url`  
    - Inputs: Source URL from previous node  
    - Outputs: Binary image data  
    - Edge cases: Network errors, 403 Forbidden if token invalid, large file size/timeouts.  

  - **Data: Binary Prep**  
    - Type: Extract From File  
    - Role: Converts downloaded binary file into proper binary property (`data`) for API use.  
    - Config: Operation `binaryToProperty`  
    - Inputs: HTTP response binary data  
    - Outputs: Binary data in `$json.data` property  
    - Edge cases: Binary conversion errors, corrupted files.  

  - **HTTP: Generate Image**  
    - Type: HTTP Request  
    - Role: Sends the AI-generated prompt and original image as inline data to Gemini’s Image Generation API to create an ad mockup.  
    - Config:  
      - POST to Gemini image generation endpoint  
      - JSON body includes prompt text from current campaign variation’s `nbp_prompt`, inline image data, generation config (image size 1K), etc.  
      - Query parameter includes Google API key.  
    - Inputs: Binary image data, campaign prompt  
    - Outputs: API response with generated image inline data  
    - Edge cases: API quota limits, malformed JSON, network errors.  

  - **Data: Convert Response**  
    - Type: Convert To File  
    - Role: Converts generated image base64 inline data back into binary file format.  
    - Config: Source property `candidates[0].content.parts[0].inlineData.data`  
    - Inputs: HTTP generate image response  
    - Outputs: Binary image file for sending  
    - Edge cases: Missing or invalid inline data.  

  - **Telegram: Send Result**  
    - Type: Telegram Node  
    - Role: Sends the generated ad image back to the original user as a Telegram photo message.  
    - Config:  
      - Chat ID from original message sender  
      - Operation: sendPhoto with binary data  
      - Credential: Telegram Bot Token  
    - Inputs: Binary image file  
    - Outputs: Telegram message confirmation  
    - Edge cases: Telegram API errors, chat ID mismatch, message size limits.  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                              | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                          |
|-------------------------|--------------------------------|----------------------------------------------|--------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Main Sticky             | Sticky Note                    | Overview and instructions for the workflow   |                          |                             | 📸 AI Ad Creative Generator - Turn simple product photos into ad concepts using Gemini Vision & Claude. Setup steps included. |
| Section 1               | Sticky Note                    | Describes Receive & Authenticate block       |                          |                             | 1. Receive & Authenticate - Validates user and extracts image file path.                           |
| Section 2               | Sticky Note                    | Describes AI Analysis & Strategy block        |                          |                             | 2. AI Analysis & Strategy - Gemini Vision analyzes product, AI Agent plans campaign.               |
| Section 3               | Sticky Note                    | Describes Generate & Deliver block             |                          |                             | 3. Generate & Deliver - Create ad visual and send result back to Telegram.                         |
| CONFIG                  | Set                           | Stores authorized user ID                      | On New Message            | Auth: Verify User ID          |                                                                                                    |
| On New Message          | Telegram Trigger              | Entry point: triggers on new Telegram message |                          | CONFIG                      |                                                                                                    |
| Auth: Verify User ID    | If                            | Checks if sender is authorized user           | CONFIG                    | Check: Has Photo?            |                                                                                                    |
| Check: Has Photo?       | If                            | Verifies message contains a photo             | Auth: Verify User ID      | Get Photo File Path          |                                                                                                    |
| Get Photo File Path     | Telegram                      | Retrieves Telegram file path of photo          | Check: Has Photo?         | Gemini Vision: Analyze       |                                                                                                    |
| Gemini Vision: Analyze  | Google Gemini (LangChain)     | Analyzes image for content and context         | Get Photo File Path       | Creative Agent               |                                                                                                    |
| OpenRouter Chat Model   | AI Language Model             | Provides Claude AI model for text generation   | Creative Agent (ai model) | Creative Agent               |                                                                                                    |
| think_tool              | LangChain Tool                | AI helper tool used by Creative Agent          | Creative Agent (ai tool)  | Creative Agent               |                                                                                                    |
| Creative Agent          | LangChain Agent               | Generates ad campaign concepts and prompts     | Gemini Vision: Analyze, OpenRouter Chat Model, think_tool, Structured Output Parser | Loop: Ad Variations          |                                                                                                    |
| Structured Output Parser| LangChain Output Parser       | Parses AI output into structured JSON          | Creative Agent            | Loop: Ad Variations          |                                                                                                    |
| Loop: Ad Variations     | Split Out                    | Iterates over each ad campaign variation       | Structured Output Parser  | Config: Source URL           |                                                                                                    |
| Config: Source URL      | Set                           | Constructs URL to download original photo      | Loop: Ad Variations       | HTTP: Download Source        |                                                                                                    |
| HTTP: Download Source   | HTTP Request                 | Downloads photo from Telegram server            | Config: Source URL        | Data: Binary Prep            |                                                                                                    |
| Data: Binary Prep       | Extract From File             | Converts downloaded photo to binary property    | HTTP: Download Source     | HTTP: Generate Image         |                                                                                                    |
| HTTP: Generate Image    | HTTP Request                 | Calls Gemini Image Generation API               | Data: Binary Prep, Loop: Ad Variations | Data: Convert Response      |                                                                                                    |
| Data: Convert Response  | Convert To File               | Converts Gemini image generation response to binary file | HTTP: Generate Image      | Telegram: Send Result        |                                                                                                    |
| Telegram: Send Result   | Telegram                      | Sends generated ad image back to user           | Data: Convert Response    |                             |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Get Credentials:**  
   - Create a Telegram bot via BotFather, obtain Bot Token.  
   - Obtain your personal Telegram User ID (via @userinfobot).  

2. **Create n8n Credentials:**  
   - Add Telegram Bot Token as credential for Telegram nodes.  
   - Add Google Gemini API Key as credential for Gemini nodes.  
   - Add OpenRouter API Key for Claude AI model.  

3. **Node: On New Message**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates.  
   - Credentials: Telegram Bot Token.  

4. **Node: CONFIG**  
   - Type: Set  
   - Parameters: Create string variable `authorized_user_id` and set to your Telegram User ID.  
   - Connect: On New Message → CONFIG.  

5. **Node: Auth: Verify User ID**  
   - Type: If  
   - Condition: Check if `{{$json.message.from.id}}` equals `{{ $json.authorized_user_id }}`.  
   - Connect: CONFIG → Auth: Verify User ID.  

6. **Node: Check: Has Photo?**  
   - Type: If  
   - Condition: Check if `$json.message.photo` exists.  
   - Connect: Auth: Verify User ID (true) → Check: Has Photo?.  

7. **Node: Get Photo File Path**  
   - Type: Telegram  
   - Operation: Get file info using photo file ID at index 1 (`photo[1].file_id`).  
   - Credentials: Telegram Bot Token.  
   - Connect: Check: Has Photo? (true) → Get Photo File Path.  

8. **Node: Gemini Vision: Analyze**  
   - Type: Google Gemini Vision (LangChain Node)  
   - Parameters: Model ID `models/gemini-pro-vision`, operation `analyze`, inputType `binary`.  
   - Credentials: Google Gemini API Key.  
   - Connect: Get Photo File Path → Gemini Vision: Analyze.  

9. **Node: OpenRouter Chat Model**  
   - Type: LangChain LM Chat OpenRouter  
   - Parameters: Model `anthropic/claude-sonnet-4.5`.  
   - Credentials: OpenRouter API Key.  

10. **Node: think_tool**  
    - Type: LangChain Tool (Think Tool)  
    - Connect: OpenRouter Chat Model → think_tool.  

11. **Node: Creative Agent**  
    - Type: LangChain Agent  
    - Parameters:  
      - System message defines the agent as Creative Art Director and AI Prompt Specialist.  
      - Input text includes user brief (Telegram message caption) and image description from Gemini Vision.  
    - Connect: Gemini Vision: Analyze → Creative Agent; think_tool → Creative Agent; OpenRouter Chat Model → Creative Agent.  

12. **Node: Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Parameters: JSON schema example with fields for agent thought process and ad campaigns array.  
    - Connect: Creative Agent → Structured Output Parser.  

13. **Node: Loop: Ad Variations**  
    - Type: Split Out  
    - Parameters: Split on `output.ad_campaigns`.  
    - Connect: Structured Output Parser → Loop: Ad Variations.  

14. **Node: Config: Source URL**  
    - Type: Set  
    - Parameters: Construct download URL using `telegram_bot_token` and file path from Get Photo File Path node.  
    - Connect: Loop: Ad Variations → Config: Source URL.  

15. **Node: HTTP: Download Source**  
    - Type: HTTP Request  
    - Parameters: GET request to `input_url` from previous node.  
    - Connect: Config: Source URL → HTTP: Download Source.  

16. **Node: Data: Binary Prep**  
    - Type: Extract From File  
    - Parameters: Operation `binaryToPropery` to prepare binary data for API.  
    - Connect: HTTP: Download Source → Data: Binary Prep.  

17. **Node: HTTP: Generate Image**  
    - Type: HTTP Request  
    - Parameters:  
      - POST to Gemini image generation API endpoint.  
      - JSON body includes prompt from current campaign variation `nbp_prompt`, inline image binary data, generation config (image size 1K).  
      - Query parameter: Google API key.  
    - Connect: Data: Binary Prep → HTTP: Generate Image.  
    - Also connect Loop: Ad Variations → HTTP: Generate Image (for prompt).  

18. **Node: Data: Convert Response**  
    - Type: Convert To File  
    - Parameters: Source property set to inline data in the image generation response (`candidates[0].content.parts[0].inlineData.data`).  
    - Connect: HTTP: Generate Image → Data: Convert Response.  

19. **Node: Telegram: Send Result**  
    - Type: Telegram  
    - Parameters:  
      - Chat ID from original message sender (`{{$json.message.from.id}}`).  
      - Operation: Send photo with binary data.  
    - Credentials: Telegram Bot Token.  
    - Connect: Data: Convert Response → Telegram: Send Result.  

20. **Activate the workflow** and test by sending a photo to your Telegram bot from the authorized user ID.  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow acts as an automated Art Director living in your Telegram. It requires a Telegram bot, Google Gemini API, and OpenRouter API credentials. | Main Sticky Note content in workflow |
| Setup steps: Create Telegram bot, get User ID, configure authorized user, add credentials, activate workflow. | Main Sticky Note content in workflow |
| Gemini Vision Pro model is used for image analysis; Gemini 3 Pro model for image generation. | Gemini Vision and Generate Image nodes |
| AI language model used is Claude Sonnet 4.5 via OpenRouter API. | OpenRouter Chat Model node |
| Telegram photo file retrieval uses the medium resolution photo (index 1 in photo array). | Get Photo File Path node |
| Workflow ensures only authorized users can use the bot, protecting usage credits. | Auth: Verify User ID node |
| For advanced prompt crafting or customization, adjust system messages in Creative Agent node. | Creative Agent node |
| Output parsing uses structured JSON schema to ensure reliable downstream processing. | Structured Output Parser node |
| Image generation response is streamed and binary converted before sending to Telegram. | HTTP Generate Image and Data Convert Response nodes |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.