Generate Images with Pollinations & Blog Articles with Gemini 2.5 from Telegram

https://n8nworkflows.xyz/workflows/generate-images-with-pollinations---blog-articles-with-gemini-2-5-from-telegram-5844


# Generate Images with Pollinations & Blog Articles with Gemini 2.5 from Telegram

### 1. Workflow Overview

This n8n workflow implements a multifunctional Telegram bot designed for content creators. It enables users to generate AI images using Pollinations API and write blog articles with Google Gemini 2.5 via LangChain integration. The bot interacts with users through Telegram messages and inline buttons, processes commands to distinguish between image or blog generation requests, and manages user preferences such as article style. It logs all activities into Google Sheets for tracking and archives generated images to Google Drive for easy access.

**Target Use Cases:**
- Telegram users who want to generate AI images from text prompts.
- Users seeking automatic blog article creation from titles with customizable writing styles.
- Content creators needing a simple chat interface to interact with AI content generation services.
- Administrators requiring audit logs and content archives via Google Sheets and Drive.

**Logical Blocks:**

- **1.1 Telegram Input Reception and Classification:** Receives Telegram messages and callback queries; classifies the input type (message, callback, text, help).
- **1.2 User Interaction and Command Routing:** Presents main menu, interprets user commands, switches between image generation, blog generation, help instructions, and style selection.
- **1.3 AI Image Generation Flow:** Constructs Pollinations image URLs, downloads images, sends them to Telegram, uploads images to Google Drive, logs prompts to Google Sheets.
- **1.4 AI Blog Article Generation Flow:** Validates blog commands, prompts for article styles, fetches style preferences, generates articles with Gemini 2.5, parses AI response, sends articles to Telegram, logs prompts and final articles to Google Sheets.
- **1.5 Auxiliary Nodes:** Handle typing indicators, error notifications, and user input validations.
- **1.6 Documentation and Setup Notes:** Sticky notes provide detailed instructions, setup guidance, and best practices.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception and Classification

**Overview:**  
This block listens for Telegram updates (messages and callback queries) and classifies the input type to guide the workflow routing.

**Nodes Involved:**  
- Trigger Telegram Message  
- Classify Telegram Input  
- Switch Input Type  

**Node Details:**

- **Trigger Telegram Message**  
  - Type: Telegram Trigger  
  - Role: Entry point, listens for messages and callback queries on Telegram bot webhook.  
  - Config: Listens for `callback_query` and `message` updates.  
  - Credentials: Telegram API credential configured.  
  - Inputs: Telegram webhook data.  
  - Outputs: Raw Telegram JSON.  
  - Edge Cases: Webhook connectivity issues, Telegram API rate limits.

- **Classify Telegram Input**  
  - Type: Code (JavaScript)  
  - Role: Determines input type — distinguishes between callback query, /start command, /help command, generic text message, or other.  
  - Key Logic: Checks presence of `callback_query`, message text content and prefixes like `/start`, `/help`, or free text.  
  - Inputs: Output from Telegram trigger.  
  - Outputs: Adds `inputType` property in JSON (values: 'style', 'callback_query', 'message', 'help', 'text').  
  - Edge Cases: Unexpected message formats, missing fields.

- **Switch Input Type**  
  - Type: Switch  
  - Role: Routes flow based on `inputType` property.  
  - Routes:  
    - `message` → main menu display  
    - `callback_query` → callback handling  
    - `text` → command validation  
    - `style` → style selection storage  
  - Inputs: Output from Classify Telegram Input.  
  - Outputs: Branches to corresponding next steps.  
  - Edge Cases: Unknown or unexpected `inputType` values.

---

#### 1.2 User Interaction and Command Routing

**Overview:**  
Manages the main menu display, processes user callback selections, asks for prompts or titles, and sends help instructions.

**Nodes Involved:**  
- Send Main Menu to User  
- Switch Callback Selection  
- Prompt User for Image Description  
- Prompt User for Blog Title  
- Send Help Instructions  

**Node Details:**

- **Send Main Menu to User**  
  - Type: Telegram  
  - Role: Sends an inline keyboard with menu buttons for "Generate Image", "Blog Article", and "Help".  
  - Configuration: Uses chat ID from incoming message, inline keyboard with callback data `image`, `blog`, `help`.  
  - Credentials: Telegram API.  
  - Inputs: Routed from Switch Input Type on `message` type.  
  - Outputs: None (final Telegram send).  
  - Edge Cases: Failure sending message, invalid chat IDs.

- **Switch Callback Selection**  
  - Type: Switch  
  - Role: Routes based on `callback_query.data` values `image`, `blog`, or `help`.  
  - Inputs: Output of Switch Input Type on `callback_query` type.  
  - Outputs:  
    - `image` → Prompt User for Image Description  
    - `blog` → Prompt User for Blog Title  
    - `help` → Send Help Instructions  
  - Edge Cases: Unknown callback data, malformed callback.

- **Prompt User for Image Description**  
  - Type: Telegram  
  - Role: Asks user to input an image prompt starting with "image".  
  - Configuration: Uses chat ID from callback query message.  
  - Credentials: Telegram API.  
  - Inputs: From Switch Callback Selection on `image`.  
  - Outputs: None.  
  - Edge Cases: User ignores or inputs invalid format.

- **Prompt User for Blog Title**  
  - Type: Telegram  
  - Role: Requests blog article title input starting with "blog".  
  - Configuration: Uses chat ID from callback query message.  
  - Credentials: Telegram API.  
  - Inputs: From Switch Callback Selection on `blog`.  
  - Outputs: None.  
  - Edge Cases: User inputs invalid titles or wrong format.

- **Send Help Instructions**  
  - Type: HTTP Request (Telegram Bot API)  
  - Role: Sends formatted help text with usage instructions in Markdown.  
  - Configuration: Uses Telegram API sendMessage endpoint, dynamic chat ID from callback query message, Markdown parse mode.  
  - Inputs: From Switch Callback Selection on `help`.  
  - Outputs: None.  
  - Edge Cases: Telegram API token must be replaced correctly, network errors.

---

#### 1.3 AI Image Generation Flow

**Overview:**  
Processes validated image commands, builds Pollinations URL, downloads AI-generated image, sends image to Telegram, uploads image to Google Drive, and logs prompt and image URL to Google Sheets.

**Nodes Involved:**  
- Validate Command Format  
- Detect Text-Based Input Type  
- Switch Text Command Type  
- Show Typing for Image Generation  
- Build Image Generation URL  
- Download AI Image  
- Send Image Result to Telegram  
- Upload Image to Google Drive  
- Log Image Prompt to Google Sheets  

**Node Details:**

- **Validate Command Format**  
  - Type: If  
  - Role: Checks if message text starts with "image" or "blog".  
  - Inputs: From Switch Input Type on `text`.  
  - Outputs: True → proceed; False → Notify Invalid Input Format.  
  - Edge Cases: Case sensitivity, partial matching.

- **Detect Text-Based Input Type**  
  - Type: Set  
  - Role: Determines if text command is "blog" or "image" based on prefix.  
  - Inputs: From Validate Command Format true branch.  
  - Outputs: Sets property `typeText` to "blog" or "image".  
  - Edge Cases: Ambiguous commands.

- **Switch Text Command Type**  
  - Type: Switch  
  - Role: Routes workflow based on `typeText` value.  
  - Outputs:  
    - `image` → Show Typing for Image Generation  
    - `blog` → Show Typing for Article Response  
  - Inputs: Detect Text-Based Input Type output.  
  - Edge Cases: Unexpected values.

- **Show Typing for Image Generation**  
  - Type: HTTP Request (Telegram API)  
  - Role: Sends Telegram "upload_photo" action to indicate image generation in progress.  
  - Inputs: Switch Text Command Type on `image` branch.  
  - Outputs: Build Image Generation URL.  
  - Edge Cases: Telegram bot token must be replaced, network errors.

- **Build Image Generation URL**  
  - Type: Code  
  - Role: Constructs Pollinations API URL with prompt, image size (1024x1024), random seed, and model parameters.  
  - Inputs: From Show Typing for Image Generation.  
  - Outputs: JSON with final prompt and image URL.  
  - Edge Cases: Prompt encoding issues.

- **Download AI Image**  
  - Type: HTTP Request (binary download)  
  - Role: Downloads image binary from Pollinations API URL.  
  - Inputs: Build Image Generation URL output.  
  - Outputs: Binary image data for sending.  
  - Edge Cases: Download failures, URL invalidation.

- **Send Image Result to Telegram**  
  - Type: Telegram (sendPhoto)  
  - Role: Sends the downloaded image to the user chat.  
  - Inputs: Download AI Image output.  
  - Outputs: Log Image Prompt to Google Sheets.  
  - Edge Cases: Telegram API errors, chat ID issues.

- **Upload Image to Google Drive**  
  - Type: Google Drive  
  - Role: Uploads image binary to specified Drive folder with prompt as filename.  
  - Inputs: Download AI Image output.  
  - Outputs: None.  
  - Edge Cases: OAuth credential issues, folder permissions.

- **Log Image Prompt to Google Sheets**  
  - Type: Google Sheets (append)  
  - Role: Logs date, type ("image"), prompt text, image URL, and user id to a Google Sheet.  
  - Inputs: After Send Image Result to Telegram node.  
  - Outputs: None.  
  - Edge Cases: Sheet ID or permissions errors.

---

#### 1.4 AI Blog Article Generation Flow

**Overview:**  
Handles blog title inputs, allows users to select article style, generates blog articles with Gemini 2.5 AI, parses AI JSON responses, sends articles back to Telegram, and logs all interactions in Google Sheets.

**Nodes Involved:**  
- Send Article Style Options  
- Store Selected Article Style  
- Fetch Last User Prompt  
- Extract Last Blog Prompt  
- Generate Article with Gemini  
- Parse Gemini Response  
- Send Article to Telegram  
- Log Blog Prompt to Google Sheets  
- Store Blog Prompt  
- Log Final Article to Google Sheets  
- Show Typing for Article Response  

**Node Details:**

- **Send Article Style Options**  
  - Type: Telegram  
  - Role: Presents inline keyboard for article style choices: Formal, Relax, News.  
  - Inputs: After logging blog prompt to Google Sheets.  
  - Outputs: User selects style (callback).  
  - Edge Cases: Missing chat ID, malformed callback.

- **Store Selected Article Style**  
  - Type: Google Sheets (appendOrUpdate)  
  - Role: Stores user-selected style in Google Sheet keyed by user id.  
  - Inputs: From Switch Input Type `style` branch.  
  - Outputs: Fetch Last User Prompt.  
  - Edge Cases: Sheet update conflicts, permission issues.

- **Fetch Last User Prompt**  
  - Type: Google Sheets (lookup)  
  - Role: Retrieves last prompt row for the user to associate with style.  
  - Inputs: From Store Selected Article Style output.  
  - Outputs: Extract Last Blog Prompt.  
  - Edge Cases: No prior prompt found.

- **Extract Last Blog Prompt**  
  - Type: Code  
  - Role: Filters Google Sheets rows by user id and returns last prompt entry.  
  - Inputs: Fetch Last User Prompt output.  
  - Outputs: Generate Article with Gemini input.  
  - Edge Cases: Empty data, filtering errors.

- **Generate Article with Gemini**  
  - Type: LangChain chainLlm (Google Gemini 2.5)  
  - Role: Sends prompt to Gemini 2.5 AI to generate blog article per selected style.  
  - Prompt: Structured with instructions to output JSON with `title` and `content`.  
  - Credentials: Google Gemini (PaLM) API via LangChain.  
  - Inputs: Blog prompt and style from previous node.  
  - Outputs: Raw AI text response.  
  - Edge Cases: API errors, malformed JSON response.

- **Parse Gemini Response**  
  - Type: Code  
  - Role: Cleans code block markdown from AI response and parses JSON to extract article content and title.  
  - Inputs: Generate Article with Gemini output.  
  - Outputs: Parsed JSON with title and content.  
  - Edge Cases: JSON parse errors, incomplete data.

- **Send Article to Telegram**  
  - Type: HTTP Request (Telegram API)  
  - Role: Sends article title and content in a single Telegram message.  
  - Inputs: Parsed Gemini response and user chat id.  
  - Outputs: Log Final Article to Google Sheets.  
  - Edge Cases: Telegram API token placeholders must be replaced, network errors.

- **Log Blog Prompt to Google Sheets**  
  - Type: Google Sheets (append)  
  - Role: Logs initial blog prompt, user id, and timestamp.  
  - Inputs: After Store Blog Prompt node.  
  - Outputs: Sends article style options to user.  
  - Edge Cases: Sheet permissions.

- **Store Blog Prompt**  
  - Type: Set  
  - Role: Saves blog title from user message for logging and later use in AI generation.  
  - Inputs: From Show Typing for Article Response node.  
  - Outputs: Log Blog Prompt to Google Sheets.  
  - Edge Cases: Missing or malformed text.

- **Log Final Article to Google Sheets**  
  - Type: Google Sheets (update)  
  - Role: Updates the previously logged blog prompt record with the generated article content, style, and timestamp.  
  - Inputs: After Send Article to Telegram.  
  - Edge Cases: Row number matching must be accurate.

- **Show Typing for Article Response**  
  - Type: HTTP Request (Telegram API)  
  - Role: Sends "typing" action to Telegram to indicate article generation in progress.  
  - Inputs: Switch Text Command Type on `blog` branch.  
  - Outputs: Store Blog Prompt.  
  - Edge Cases: Must configure Telegram bot token.

---

#### 1.5 Auxiliary Nodes

**Overview:**  
Nodes for error handling, typing indicators, and invalid input notifications.

**Nodes Involved:**  
- Notify Invalid Input Format  
- Show Typing for Article Response  
- Show Typing for Image Generation  

**Node Details:**

- **Notify Invalid Input Format**  
  - Type: Telegram  
  - Role: Sends error message if user command format is invalid.  
  - Inputs: Validate Command Format `false` branch.  
  - Outputs: None.  
  - Edge Cases: Telegram API errors.

- **Show Typing for Article Response** and **Show Typing for Image Generation**  
  - Both use Telegram API `sendChatAction` to indicate bot activity.

---

#### 1.6 Documentation and Setup Notes (Sticky Notes)

**Overview:**  
Several sticky notes provide extensive documentation, setup instructions, customization tips, security best practices, and project credits.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  

**Content Highlights:**  
- Workflow purpose and features  
- Example commands and usage  
- Setup instructions including credential management  
- Google Sheets and Drive integration details  
- Security best practices (no hardcoded keys, credential manager use)  
- External links to Pollinations and Google API resources

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                                   | Input Node(s)                    | Output Node(s)                                   | Sticky Note                                                                                               |
|-------------------------------|----------------------------------|--------------------------------------------------|---------------------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Trigger Telegram Message       | Telegram Trigger                 | Entry point listening for Telegram updates       | -                               | Classify Telegram Input                         |                                                                                                          |
| Classify Telegram Input        | Code                            | Classifies input type from Telegram payload       | Trigger Telegram Message         | Switch Input Type                              |                                                                                                          |
| Switch Input Type             | Switch                         | Routes flow based on input type                    | Classify Telegram Input          | Send Main Menu to User, Switch Callback Selection, Validate Command Format, Store Selected Article Style |                                                                                                          |
| Send Main Menu to User         | Telegram                        | Sends main menu keyboard to user                   | Switch Input Type (message)      | -                                               |                                                                                                          |
| Switch Callback Selection      | Switch                         | Routes based on callback data                      | Switch Input Type (callback)     | Prompt User for Image Description, Prompt User for Blog Title, Send Help Instructions                   |                                                                                                          |
| Prompt User for Image Description | Telegram                        | Prompts user to send image description             | Switch Callback Selection (image) | -                                               |                                                                                                          |
| Prompt User for Blog Title     | Telegram                        | Prompts user to send blog title                     | Switch Callback Selection (blog) | -                                               |                                                                                                          |
| Send Help Instructions         | HTTP Request (Telegram API)     | Sends help message with usage instructions         | Switch Callback Selection (help) | -                                               |                                                                                                          |
| Validate Command Format        | If                             | Validates if text command starts with image/blog  | Switch Input Type (text)         | Detect Text-Based Input Type, Notify Invalid Input Format |                                                                                                          |
| Notify Invalid Input Format    | Telegram                        | Notifies user about invalid input format           | Validate Command Format (false)  | -                                               |                                                                                                          |
| Detect Text-Based Input Type   | Set                            | Determines if command is image or blog             | Validate Command Format (true)   | Switch Text Command Type                        |                                                                                                          |
| Switch Text Command Type       | Switch                         | Routes based on detected text command type         | Detect Text-Based Input Type     | Show Typing for Image Generation, Show Typing for Article Response |                                                                                                          |
| Show Typing for Image Generation | HTTP Request (Telegram API)     | Sends typing indicator for image generation        | Switch Text Command Type (image) | Build Image Generation URL                      |                                                                                                          |
| Build Image Generation URL     | Code                           | Constructs Pollinations image URL                   | Show Typing for Image Generation | Download AI Image                               |                                                                                                          |
| Download AI Image              | HTTP Request                   | Downloads image binary from Pollinations API       | Build Image Generation URL       | Send Image Result to Telegram, Upload Image to Google Drive |                                                                                                          |
| Send Image Result to Telegram  | Telegram (sendPhoto)            | Sends image to Telegram chat                        | Download AI Image                | Log Image Prompt to Google Sheets               |                                                                                                          |
| Upload Image to Google Drive   | Google Drive                   | Uploads image to Drive folder                        | Download AI Image                | -                                               |                                                                                                          |
| Log Image Prompt to Google Sheets | Google Sheets                 | Logs image prompt and URL                           | Send Image Result to Telegram    | -                                               | Sticky Note1 covers Google Sheets integration details                                                   |
| Send Article Style Options     | Telegram                       | Sends article style choices inline keyboard        | Log Blog Prompt to Google Sheets | -                                               |                                                                                                          |
| Store Selected Article Style   | Google Sheets                 | Saves selected article style                        | Switch Input Type (style)         | Fetch Last User Prompt                          |                                                                                                          |
| Fetch Last User Prompt         | Google Sheets                 | Retrieves last prompt for user                       | Store Selected Article Style     | Extract Last Blog Prompt                        |                                                                                                          |
| Extract Last Blog Prompt       | Code                          | Filters last blog prompt row for user               | Fetch Last User Prompt           | Generate Article with Gemini                    |                                                                                                          |
| Generate Article with Gemini   | LangChain Gemini 2.5 AI         | Generates blog article from prompt and style       | Extract Last Blog Prompt         | Parse Gemini Response                           |                                                                                                          |
| Parse Gemini Response          | Code                          | Parses AI JSON response                              | Generate Article with Gemini     | Send Article to Telegram                        |                                                                                                          |
| Send Article to Telegram       | HTTP Request (Telegram API)     | Sends generated blog article to Telegram           | Parse Gemini Response            | Log Final Article to Google Sheets              |                                                                                                          |
| Log Blog Prompt to Google Sheets | Google Sheets                 | Logs initial blog prompt                            | Store Blog Prompt                | Send Article Style Options                      |                                                                                                          |
| Store Blog Prompt              | Set                           | Stores blog title from user input                   | Show Typing for Article Response | Log Blog Prompt to Google Sheets                |                                                                                                          |
| Log Final Article to Google Sheets | Google Sheets                 | Updates blog prompt record with final article       | Send Article to Telegram         | -                                               |                                                                                                          |
| Show Typing for Article Response | HTTP Request (Telegram API)     | Sends typing indicator for blog generation          | Switch Text Command Type (blog)  | Store Blog Prompt                               |                                                                                                          |
| Sticky Note                   | Sticky Note                   | Documentation and overview                           | -                               | -                                               | Comprehensive overview, features, and usage instructions                                               |
| Sticky Note1                  | Sticky Note                   | Google Sheets logging system details                 | -                               | -                                               | Detailed Sheet schema and logging explanations                                                         |
| Sticky Note2                  | Sticky Note                   | Setup instructions and customization tips            | -                               | -                                               | Credential setup, variable updates, testing tips                                                       |
| Sticky Note3                  | Sticky Note                   | Security best practices and project credits           | -                               | -                                               | Security advice and external resource links                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for `callback_query` and `message` updates.  
   - Add Telegram API credentials.  
   - Position as workflow entry.

2. **Add a Code node to classify Telegram input**  
   - Name: Classify Telegram Input  
   - Use JavaScript logic to set `inputType` property based on presence of callback_query, message text starting with `/start`, `/help`, or free text.  
   - Connect Telegram Trigger output to this node.

3. **Add Switch node for input routing**  
   - Name: Switch Input Type  
   - Route on `$json.inputType` with branches: `message`, `callback_query`, `text`, `style`.  
   - Connect Classify Telegram Input output here.

4. **Add Telegram node to send main menu**  
   - Name: Send Main Menu to User  
   - Text: "Please choose a menu:"  
   - Use inline keyboard with buttons: "Generate Image" (callback_data: `image`), "Blog Article" (`blog`), "❓ Help" (`help`).  
   - Use dynamic chat ID from `$json.message.chat.id`.  
   - Connect `message` branch from Switch Input Type to this node.

5. **Add Switch node for callback query selection**  
   - Name: Switch Callback Selection  
   - Route on `$json.callback_query.data` with `image`, `blog`, `help` branches.  
   - Connect `callback_query` branch from Switch Input Type.

6. **Add Telegram nodes to prompt for user input**  
   - Prompt User for Image Description: ask to "Please send an image prompt, start with image" using chat ID from `$json.callback_query.message.chat.id`.  
   - Prompt User for Blog Title: ask to "Please submit article title, start with blog" using chat ID similarly.  
   - Connect Switch Callback Selection branches `image` and `blog` to respective prompt nodes.

7. **Add HTTP Request node to send help instructions**  
   - Use Telegram Bot API `sendMessage` endpoint.  
   - Text includes usage instructions with Markdown formatting.  
   - Use dynamic chat ID from callback query message.  
   - Connect `help` branch from Switch Callback Selection.

8. **Add If node to validate command format**  
   - Validate if message text starts with "image" or "blog" (case insensitive).  
   - Connect `text` branch from Switch Input Type here.

9. **Add Telegram node to notify invalid input**  
   - Sends error message "❌ Invalid format. Please use image your prompt or blog your title."  
   - Connect `false` branch of validation to this node.

10. **Add Set node to detect text command type**  
    - Set `typeText` to `blog` if text starts with "blog", else `image`.  
    - Connect `true` branch of validation here.

11. **Add Switch node to route text commands**  
    - Route on `typeText`: `image` or `blog`.  
    - Connect Set node output here.

12. **Image generation branch:**

    - Add HTTP Request node to send "upload_photo" chat action to Telegram (typing indicator).  
    - Add Code node to build Pollinations image URL with prompt text, fixed width/height (1024x1024), random seed, model parameters.  
    - Add HTTP Request node to download image binary from constructed URL.  
    - Add Telegram node (sendPhoto) to send the downloaded image to user.  
    - Add Google Drive node to upload image binary to a specific folder, named by prompt text.  
    - Add Google Sheets node to append image prompt, user ID, date, type, and output URL to a sheet.  
    - Connect all nodes sequentially starting from Switch Text Command Type `image` branch.

13. **Blog article generation branch:**

    - Add HTTP Request node to send "typing" chat action for Telegram.  
    - Add Set node to store blog title from user message text.  
    - Add Google Sheets node to append the blog prompt (title, user id, date).  
    - Add Telegram node to send inline keyboard for article style selection (buttons: Formal, Relax, News).  
    - Add Google Sheets node to appendOrUpdate style selection associated with user id.  
    - Add Google Sheets node to fetch last prompt entry for the user.  
    - Add Code node to extract last blog prompt data from fetched rows.  
    - Add LangChain Gemini 2.5 AI node to generate article based on title and style, requesting JSON output with title and content.  
    - Add Code node to parse AI JSON response, removing markdown artifacts.  
    - Add HTTP Request node to send parsed article text back to Telegram.  
    - Add Google Sheets node to update the logged prompt record with final article content and style.  
    - Connect nodes sequentially starting from Switch Text Command Type `blog` branch and style selection callback.

14. **Add auxiliary nodes:**

    - Notify Invalid Input Format node (Telegram) for invalid commands.  
    - Typing indicators for image and article generation via HTTP Request nodes.  
    - Ensure all Telegram API tokens and credentials are properly configured and replaced dynamically in HTTP requests.

15. **Add Sticky Note nodes for documentation:**

    - Include detailed workflow description, setup instructions, security best practices, and external resource links for user reference.

16. **Configure credentials:**

    - Telegram API credential with bot token.  
    - Google Sheets OAuth2 credential with access to the specific spreadsheet.  
    - Google Drive OAuth2 credential with access to designated folder.  
    - Google Gemini API credentials integrated via LangChain nodes.

17. **Test the workflow:**

    - Send `/start` to bot on Telegram → expect main menu.  
    - Test image generation command: `image a sunset over mountains` → receives AI image.  
    - Test blog generation command: `blog Benefits of Meditation` → follows style selection and receives article.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow enables Telegram users to generate AI image assets and blog articles seamlessly via chat interface.                      | Workflow overview in Sticky Note.                                                                  |
| Google Sheets used as centralized logging system for prompts, outputs, and user IDs.                                                    | Sticky Note1 details Google Sheets schema and logging strategy.                                    |
| Google Drive stores AI-generated images for archiving and future retrieval.                                                             | Sticky Note1 and Upload Image to Google Drive node.                                               |
| Setup requires Telegram Bot token, Google Sheets OAuth2, Google Drive OAuth2, and Google Gemini API credentials.                       | Sticky Note2 setup instructions and credential info.                                             |
| Security best practices: do not hardcode API keys in HTTP nodes, always use n8n credential manager.                                     | Sticky Note3 highlights security guidelines.                                                     |
| Commands start with keywords: `image` or `blog`. Blog requests prompt for style selection before generation.                            | Command format and interaction flow explained in Sticky Notes and workflow logic.                 |
| External resources for further learning: https://github.com/pollinations/pollinations, https://aistudio.google.com/apikey               | Provided in Sticky Note3.                                                                          |
| Ensure all Telegram API URLs in HTTP Request nodes replace placeholder tokens with actual bot token securely.                          | Critical for HTTP Request nodes sending chat actions and messages.                               |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.