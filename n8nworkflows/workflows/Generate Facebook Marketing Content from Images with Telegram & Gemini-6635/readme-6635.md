Generate Facebook Marketing Content from Images with Telegram & Gemini

https://n8nworkflows.xyz/workflows/generate-facebook-marketing-content-from-images-with-telegram---gemini-6635


# Generate Facebook Marketing Content from Images with Telegram & Gemini

### 1. Workflow Overview

This workflow automates the creation and approval of Facebook marketing content based on images received via Telegram. It targets marketing teams or social media managers who want to generate tailored ad copy and visuals efficiently by leveraging AI (Gemini language model) for content generation and Telegram as the user interface for input and approval.

The workflow logically divides into these main blocks:

- **1.1 Input Reception & Validation**: Listens to Telegram messages, detects if the message contains a photo, and extracts relevant metadata.
- **1.2 Image Retrieval**: Fetches the actual image file from Telegram servers using Telegram's API.
- **1.3 AI Content Generation**: Sends the image and caption data to an AI agent powered by Gemini to generate marketing content in Arabic following brand tone guidelines.
- **1.4 Content Parsing & Routing**: Parses the structured AI output JSON and decides if the content is new or an edited version from the user.
- **1.5 User Approval Workflow**: Sends the generated marketing content back to the user on Telegram for approval or rejection.
- **1.6 Publishing & Notification**: If approved, posts the content with image to Facebook and notifies the user of successful publication.
- **1.7 Feedback Loop**: If edits are requested, prompts the user to specify changes and re-triggers the workflow with updated text input for content regeneration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block triggers the workflow upon receiving any Telegram update, checks if the message contains a photo, and extracts key metadata required for downstream processing.

- **Nodes Involved:**  
  - Trigger from Telegram  
  - Check if Message has Photo  
  - Extract Telegram Metadata

- **Node Details:**

  1. **Trigger from Telegram**  
     - Type: `telegramTrigger`  
     - Role: Entry point triggered by any Telegram update (messages, photos, texts).  
     - Config: Listens for all update types (`updates: ["*"]`). Uses Telegram credentials via OAuth.  
     - Inputs: None (trigger)  
     - Outputs: Raw Telegram update data.  
     - Edge cases: Missing or malformed update data; Telegram API downtime; webhook setup errors.  
   
  2. **Check if Message has Photo**  
     - Type: `if`  
     - Role: Determines if the incoming Telegram message contains a photo to branch logic.  
     - Config: Checks if `message.photo` field is empty.  
     - Inputs: Telegram update JSON.  
     - Outputs:  
       - True branch: No photo → fallback route for text-only messages.  
       - False branch: Photo present → proceed with image processing.  
     - Edge cases: Photo array length less than expected; photo field missing; message from bots.  

  3. **Extract Telegram Metadata**  
     - Type: `set`  
     - Role: Extracts key fields from the Telegram message: the largest photo file ID (index 3), caption, chat ID, message ID, and current timestamp.  
     - Config: Uses expressions to access nested JSON fields safely; defaults caption to "No caption provided" if missing.  
     - Inputs: Telegram message JSON with photo info.  
     - Outputs: JSON containing extracted metadata for next API call.  
     - Edge cases: Photo array length less than 4 (index 3 might be undefined); missing caption; timestamp formatting.

---

#### 2.2 Image Retrieval

- **Overview:**  
  Using the extracted file ID, this block calls Telegram API to get the file path, builds the file download URL, and downloads the image for AI processing.

- **Nodes Involved:**  
  - Get Telegram File Info  
  - Build Telegram Image URL  
  - Download Telegram Image

- **Node Details:**

  1. **Get Telegram File Info**  
     - Type: `httpRequest`  
     - Role: Calls Telegram's `getFile` endpoint with the file ID to retrieve file path.  
     - Config: URL constructed dynamically using Telegram Bot Token and extracted file ID.  
     - Inputs: Metadata containing photo_file_id.  
     - Outputs: JSON with file path (`result.file_path`).  
     - Edge cases: Invalid file ID; expired or revoked bot token; Telegram API errors; network timeouts.

  2. **Build Telegram Image URL**  
     - Type: `set`  
     - Role: Constructs the full URL to download the image from Telegram servers using the file path.  
     - Config: Concatenates `https://api.telegram.org/file/bot[YOUR_BOT_TOKEN]/` with the file path.  
     - Inputs: API response JSON with file_path.  
     - Outputs: JSON with `download_url` for the image.  
     - Edge cases: Missing or malformed file_path; incorrect token substitution; URL encoding issues.

  3. **Download Telegram Image**  
     - Type: `httpRequest`  
     - Role: Downloads the actual image file as binary data for AI consumption.  
     - Config: Uses the constructed download URL; expects response as file attached to property `photo_data`.  
     - Inputs: JSON with download_url.  
     - Outputs: Binary image data stored in `photo_data`.  
     - Edge cases: Download failures, 404 errors, large file size limits, network interruptions.

---

#### 2.3 AI Content Generation

- **Overview:**  
  Sends the downloaded image and caption to an AI agent node configured with a Gemini-based language model to generate marketing content in Arabic following brand guidelines.

- **Nodes Involved:**  
  - Generate Marketing Content  
  - Language Model (OpenRouter)

- **Node Details:**

  1. **Generate Marketing Content**  
     - Type: `langchain.agent`  
     - Role: AI agent that generates structured marketing content from the image and caption input.  
     - Config: Uses a detailed prompt specifying brand voice, style, structure (headline, content, hashtags, CTA), and language (clear Arabic).  
     - Inputs: Binary image data (`photo_data`) and extracted caption.  
     - Outputs: AI-generated JSON string with headline, content, hashtags array, CTA.  
     - Edge cases: AI timeouts; prompt misinterpretation; output formatting errors; language model API failures.

  2. **Language Model (OpenRouter)**  
     - Type: `langchain.lmChatOpenRouter`  
     - Role: Underlying LLM provider node (Google Gemini 2.5 flash lite model) powering the AI agent.  
     - Config: Model name `google/gemini-2.5-flash-lite`; uses OpenRouter API credentials.  
     - Inputs: Prompt from `Generate Marketing Content` node.  
     - Outputs: AI text completion for agent.  
     - Edge cases: API key expiry; rate limits; network errors; version compatibility.

---

#### 2.4 Content Parsing & Routing

- **Overview:**  
  Parses the JSON response string from the AI to extract structured fields. Then determines if this is a new content generation or a user-requested edit to route content for appropriate approval flow.

- **Nodes Involved:**  
  - Parse AI Output  
  - Decide Input is New or Old

- **Node Details:**

  1. **Parse AI Output**  
     - Type: `set`  
     - Role: Parses the AI output string into JSON fields headline, content, hashtags (joined as string), CTA, and generates a random approval ID.  
     - Config: Uses JSON.parse with regex to clean triple backticks and whitespace from AI output.  
     - Inputs: raw AI JSON string from `Generate Marketing Content`.  
     - Outputs: Parsed JSON object with named fields for approval and posting.  
     - Edge cases: Malformed AI output; JSON parse errors; missing fields; expression evaluation errors.

  2. **Decide Input is New or Old**  
     - Type: `switch`  
     - Role: Branches the workflow based on whether the message sender is a bot or user, indicating new content or user edits.  
     - Config: Checks `message.from.is_bot` boolean field.  
     - Inputs: Telegram message JSON.  
     - Outputs:  
       - New content path (user message, not bot).  
       - Old/edit path (bot message).  
     - Edge cases: Missing or inconsistent `is_bot` field; unexpected message sources.

---

#### 2.5 User Approval Workflow

- **Overview:**  
  Sends the generated content back to the user via Telegram with instructions to approve or reject. Waits for the reply and routes based on approval decision.

- **Nodes Involved:**  
  - Send Content for Approval (New)  
  - Send Content for Approval (Old)  
  - Approval Decision

- **Node Details:**

  1. **Send Content for Approval (New)**  
     - Type: `telegram` (sendAndWait)  
     - Role: Sends the generated content to the user for review and waits for their approval or rejection response.  
     - Config: Message formatted with headline, content, hashtags, CTA; approval buttons "APPROVE" or "REJECT".  
     - Inputs: Parsed AI content fields for new user messages.  
     - Outputs: User response data.  
     - Edge cases: Telegram API send failures; user non-response; message formatting errors.

  2. **Send Content for Approval (Old)**  
     - Type: `telegram` (sendAndWait)  
     - Role: Same as above but for edited content requests (bot messages).  
     - Config: Similar message and buttons.  
     - Inputs: Parsed AI content fields for edits.  
     - Outputs: User response data.  
     - Edge cases: Same as above.

  3. **Approval Decision**  
     - Type: `if`  
     - Role: Checks if user approved or rejected the content based on Telegram reply.  
     - Config: Checks boolean `data.approved` field from user response.  
     - Inputs: User reply JSON.  
     - Outputs:  
       - Approved → proceed to Facebook posting.  
       - Rejected → loop back to request changes or fallback.  
     - Edge cases: User replies other than expected; missing approval data; expression evaluation errors.

---

#### 2.6 Publishing & Notification

- **Overview:**  
  Posts the approved marketing content with image to Facebook, then notifies the user of success via Telegram.

- **Nodes Involved:**  
  - Publish Post to Facebook  
  - Notify Facebook Success

- **Node Details:**

  1. **Publish Post to Facebook**  
     - Type: `httpRequest`  
     - Role: Posts photo and message content to Facebook page using Facebook Graph API.  
     - Config: URL uses Facebook Page ID; multipart form data with image file path and message body (headline + content + hashtags); OAuth2 authentication for Facebook.  
     - Inputs: Approved marketing content and file path.  
     - Outputs: Facebook API response.  
     - Edge cases: OAuth token expiry; API rate limits; invalid page ID; file upload failures; malformed request body.

  2. **Notify Facebook Success**  
     - Type: `telegram`  
     - Role: Sends confirmation message to user that post was scheduled/published on Facebook.  
     - Config: Static success text message.  
     - Inputs: Chat ID from Telegram metadata.  
     - Outputs: None.  
     - Edge cases: Telegram API failures; incorrect chat ID.

---

#### 2.7 Feedback Loop

- **Overview:**  
  If user requests changes or sends text-only input, prompts the user to specify desired edits and re-triggers the workflow with new text to regenerate content.

- **Nodes Involved:**  
  - Check If Text or Photo  
  - Asking User for Changes  
  - Prompt for User Input (Manual)  
  - Restart Workflow Again  
  - Restart Workflow with User Text

- **Node Details:**

  1. **Check If Text or Photo**  
     - Type: `switch`  
     - Role: Determines if the message came from the photo route or text route (bot or user).  
     - Config: Checks `message.from.is_bot` field.  
     - Inputs: Telegram message.  
     - Outputs:  
       - Text input path: prompt user for manual input.  
       - Photo input path: prompt user for change description.  
     - Edge cases: Missing `is_bot`; unexpected message types.

  2. **Asking User for Changes**  
     - Type: `telegram`  
     - Role: Sends a Telegram message asking the user "What changes would you like?" when a photo was sent.  
     - Inputs: Chat ID from metadata.  
     - Outputs: User reply awaited.  
     - Edge cases: User ignores prompt; Telegram failures.

  3. **Prompt for User Input (Manual)**  
     - Type: `telegram`  
     - Role: Sends a Telegram message asking for text input "What changes would you like?" when no photo was sent.  
     - Inputs: Chat ID.  
     - Outputs: User reply awaited.  
     - Edge cases: Same as above.

  4. **Restart Workflow Again**  
     - Type: `executeWorkflow`  
     - Role: Re-triggers the same workflow with the user’s text input data to regenerate content.  
     - Config: Requires setting workflow ID and input mapping.  
     - Inputs: User text responses.  
     - Outputs: Workflow restart.  
     - Edge cases: Missing workflow ID configuration; infinite loops if improperly handled.

  5. **Restart Workflow with User Text**  
     - Type: `executeWorkflow`  
     - Role: Similar to above, specifically handles manual text input re-trigger.  
     - Inputs/Outputs: Same as above.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                            | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                   |
|------------------------------|----------------------------------|--------------------------------------------|-------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Trigger from Telegram         | telegramTrigger                  | Entry point: receives Telegram updates     | None                          | Check if Message has Photo         | Starts the workflow when the Telegram bot receives a new message from a user, including photos and text.     |
| Check if Message has Photo    | if                              | Checks if message contains a photo          | Trigger from Telegram          | Extract Telegram Metadata, Generate Marketing Content | Determines whether the incoming Telegram message contains a photo. Routes to image or text path.              |
| Extract Telegram Metadata     | set                             | Extracts file_id, caption, chat_id, etc.    | Check if Message has Photo     | Get Telegram File Info             | Extracts key fields like chat_id, file_id, caption, and timestamp from the incoming message.                  |
| Get Telegram File Info        | httpRequest                     | Gets file path from Telegram API            | Extract Telegram Metadata      | Build Telegram Image URL           | Calls Telegram's API to retrieve file path information using the photo file ID.                              |
| Build Telegram Image URL      | set                             | Constructs URL to download the image        | Get Telegram File Info         | Download Telegram Image            | Builds a downloadable file URL using the file path from Telegram's response.                                |
| Download Telegram Image       | httpRequest                     | Downloads image file from Telegram server   | Build Telegram Image URL       | Generate Marketing Content         | Downloads the photo file from Telegram and stores it as photo_data for use by the AI.                        |
| Generate Marketing Content    | langchain.agent                 | Generates marketing content using AI agent  | Download Telegram Image        | Parse AI Output                   | Uses an AI agent (powered by Gemini) to generate marketing content based on the image and brand prompt.      |
| Language Model (OpenRouter)   | langchain.lmChatOpenRouter      | Provides underlying LLM for AI agent        | Generate Marketing Content (AI input) | Generate Marketing Content (AI output) | Provides the underlying LLM (Gemini) that powers the AI Agent node to generate the post content.             |
| Parse AI Output               | set                             | Parses AI JSON output into fields            | Generate Marketing Content     | Decide Input is New or Old         | Parses the structured JSON response from the AI to extract headline, content, hashtags, CTA, and approval ID.|
| Decide Input is New or Old    | switch                          | Routes content based on message source       | Parse AI Output               | Send Content for Approval (New), Send Content for Approval (Old) | Checks if message sender is new or user requested edit, routes content to appropriate approval flow.         |
| Send Content for Approval (New) | telegram                      | Sends generated content to user for approval| Decide Input is New or Old     | Approval Decision                 | Sends AI content to user for review and waits for approval or rejection reply.                              |
| Send Content for Approval (Old) | telegram                      | Sends generated content for edits approval   | Decide Input is New or Old     | Approval Decision                 | Sends AI content to user for review and waits for approval or rejection reply.                              |
| Approval Decision            | if                              | Routes based on user's approval decision     | Send Content for Approval (New), Send Content for Approval (Old) | Publish Post to Facebook, Check If Text or Photo | Checks if user approved or rejected the content and routes accordingly.                                   |
| Publish Post to Facebook     | httpRequest                     | Posts approved content and photo to Facebook | Approval Decision             | Notify Facebook Success           | Posts approved photo and content to a Facebook page via multipart HTTP request.                             |
| Notify Facebook Success      | telegram                        | Notifies user of Facebook post success       | Publish Post to Facebook       | None                            | Notifies user via Telegram that the Facebook post was successfully scheduled or published.                  |
| Check If Text or Photo       | switch                          | Determines if message is from text or photo path | Approval Decision             | Asking User for Changes, Prompt for User Input (Manual) | Checks if message came from text or photo route.                                                          |
| Asking User for Changes      | telegram                        | Asks user what changes they want (photo path) | Check If Text or Photo         | Restart Workflow Again            | Sends Telegram message asking "What changes would you like?" for photo messages.                            |
| Prompt for User Input (Manual) | telegram                      | Asks user for input text (text-only path)    | Check If Text or Photo         | Restart Workflow with User Text   | Sends Telegram message asking "What changes would you like?" for text-only messages.                        |
| Restart Workflow Again       | executeWorkflow                 | Re-triggers workflow with user's change text | Asking User for Changes        | None                            | Re-triggers workflow with user text input for content regeneration.                                        |
| Restart Workflow with User Text | executeWorkflow              | Re-triggers workflow with manual user text   | Prompt for User Input (Manual) | None                            | Re-triggers workflow with user text input for content regeneration.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Configure to listen for all update types (`updates: ["*"]`).  
   - Set Telegram API credentials (OAuth2) with Bot Token.  
   - Set webhook URL as required.

2. **Add If Node: Check if Message has Photo**  
   - Check if `message.photo` field is empty.  
   - True branch: no photo.  
   - False branch: photo present.

3. **Add Set Node: Extract Telegram Metadata** (connected from False branch)  
   - Extract largest photo file ID (`message.photo[3].file_id`).  
   - Extract photo caption or default "No caption provided".  
   - Extract chat ID and message ID.  
   - Add current timestamp in ISO format.

4. **Add HTTP Request Node: Get Telegram File Info**  
   - URL: `https://api.telegram.org/{{TELEGRAM_BOT_TOKEN}}/getFile?file_id={{photo_file_id}}`  
   - Method: GET.

5. **Add Set Node: Build Telegram Image URL**  
   - Extract `file_path` from previous response.  
   - Construct URL: `https://api.telegram.org/file/bot[YOUR_BOT_TOKEN]/{{file_path}}`.

6. **Add HTTP Request Node: Download Telegram Image**  
   - URL: download_url from previous node.  
   - Response type: file, output property name `photo_data`.

7. **Add LangChain Agent Node: Generate Marketing Content**  
   - Configure prompt with brand style instructions, output JSON format.  
   - Pass photo_data and caption as context.  
   - Use AI agent node type.

8. **Add LangChain LLM Node: Language Model (OpenRouter)**  
   - Model: `google/gemini-2.5-flash-lite`.  
   - Set OpenRouter API credentials.  
   - Connect to AI agent node as language model provider.

9. **Add Set Node: Parse AI Output**  
   - Parse AI output string removing backticks and whitespace.  
   - Extract fields: headline, content, hashtags (join array), CTA.  
   - Generate random approval ID.

10. **Add Switch Node: Decide Input is New or Old**  
    - Check `message.from.is_bot`.  
    - If false: new content path.  
    - If true: edit path.

11. **Add Telegram Node: Send Content for Approval (New)**  
    - Chat ID from metadata.  
    - Message with headline, content, hashtags, CTA.  
    - Use "sendAndWait" operation with approval buttons.

12. **Add Telegram Node: Send Content for Approval (Old)**  
    - Same as above but for edit path.

13. **Add If Node: Approval Decision**  
    - Check if user approved (boolean flag).  
    - If approved: proceed to Facebook posting.  
    - If rejected: proceed to feedback loop.

14. **Add HTTP Request Node: Publish Post to Facebook**  
    - URL: `https://graph.facebook.com/v12.0/{{PAGE_ID}}/photos`.  
    - Method: POST.  
    - Authentication: OAuth2 Facebook.  
    - Multipart form-data with photo file and composed message.

15. **Add Telegram Node: Notify Facebook Success**  
    - Send confirmation message to user chat ID.

16. **Add Switch Node: Check If Text or Photo**  
    - Check `message.from.is_bot` to route between photo or text input feedback.

17. **Add Telegram Node: Asking User for Changes**  
    - Message: "What changes would you like?" for photo input.  
    - Chat ID from metadata.

18. **Add Telegram Node: Prompt for User Input (Manual)**  
    - Message: "What changes would you like?" for text input.  
    - Chat ID from metadata.

19. **Add Execute Workflow Nodes: Restart Workflow Again & Restart Workflow with User Text**  
    - Re-trigger the main workflow with user input text to regenerate content.  
    - Configure workflow ID and input mappings accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses Telegram as both input and user approval interface, leveraging Telegram's API.    | Telegram Bot API documentation: https://core.telegram.org/bots/api                                |
| AI content generation uses LangChain agent and Google Gemini model via OpenRouter API.             | OpenRouter platform: https://openrouter.ai/                                                     |
| Facebook posting requires Facebook Graph API OAuth2 credentials with "pages_manage_posts" permission.| Facebook Graph API docs: https://developers.facebook.com/docs/graph-api                          |
| The AI prompt is carefully crafted to generate authentic Arabic marketing copy aligned with brand voice. | Prompt includes explicit style, tone, and output JSON formatting instructions.                   |
| Workflow includes user feedback loop to refine AI-generated content iteratively without leaving Telegram. | Improves user experience by minimizing context switching.                                       |

---

**Disclaimer:**  
The provided text and workflow are exclusively generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected elements. All handled data is legal and public.