Create AI Dream Videos with Analysis using Veo3 and Telegram

https://n8nworkflows.xyz/workflows/create-ai-dream-videos-with-analysis-using-veo3-and-telegram-11575


# Create AI Dream Videos with Analysis using Veo3 and Telegram

### 1. Workflow Overview

This workflow, titled **Create AI Dream Videos with Analysis using Veo3 and Telegram**, enables users to interact with a Telegram bot to submit dream descriptions and receive AI-generated cinematic videos alongside psychological dream analyses. Its target use case is for individuals interested in visualizing dreams through AI-generated video content enriched with thematic and symbolic interpretation, also optionally logging the data for journaling purposes.

The workflow is logically structured into the following blocks:

- **1.1 Telegram Input**: Receives user messages via Telegram bot.
- **1.2 Command Parsing**: Parses commands to extract requested video style and dream description.
- **1.3 Style Help Routing**: Provides style options on user request.
- **1.4 Dream Content Validation**: Checks if valid dream description is present.
- **1.5 AI Dream Analysis**: Uses a language model to analyze the dream and generate a video prompt and psychological interpretation.
- **1.6 Video Generation with Veo3**: Invokes Google Veo3 API to generate the dream video with native audio based on AI prompt.
- **1.7 Video Generation Polling & Completion Handling**: Waits and polls for video generation completion.
- **1.8 Optional Dream Journal Logging**: Logs dream details and video information to Google Sheets.
- **1.9 User Notification**: Sends video and analysis back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input

- **Overview:**  
  Listens for incoming Telegram messages from users to trigger the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Entry point, listens for new messages with the webhook ID `dream-bot-webhook`.  
    - Configuration: Listens specifically to "message" update types.  
    - Input: Incoming Telegram message webhook.  
    - Output: Message JSON passed downstream.  
    - Potential Failures: Invalid bot token, webhook misconfiguration, Telegram API downtime.

#### 1.2 Command Parsing

- **Overview:**  
  Parses user messages to extract the requested dream style and the dream description.

- **Nodes Involved:**  
  - Parse Dream Command (Code node)

- **Node Details:**  
  - **Parse Dream Command**  
    - Type: Code Node (JavaScript)  
    - Role: Extracts style keyword and dream text from the Telegram message.  
    - Configuration: Uses a predefined styles dictionary mapping style names to descriptions. Defaults to "cinematic" if none specified. Parses `/dream` commands and `/styles` requests.  
    - Expressions/Variables: `$input.first().json.message.text` for message text.  
    - Outputs: JSON with parsed style, styleDescription, dreamContent, flags for styles request and content presence, chat and user info, and timestamp.  
    - Failure Modes: Unexpected message format, missing text property, parsing errors.

#### 1.3 Style Help Routing

- **Overview:**  
  Routes requests for style information and sends a style list message back to the user.

- **Nodes Involved:**  
  - Route: Style Help Request (If node)  
  - Send Available Styles (Telegram node)

- **Node Details:**  
  - **Route: Style Help Request**  
    - Type: If Node  
    - Role: Checks if incoming message is a `/styles` request (boolean flag from parsing).  
    - Conditions: `$json.isStylesRequest === true`.  
    - Outputs: True branch for style help, false passes to valid dream input routing.  
  - **Send Available Styles**  
    - Type: Telegram Node  
    - Role: Sends a formatted message listing all available styles and usage examples.  
    - Configuration: Markdown formatting, uses `$json.chatId` to send to correct chat.  
    - Potential Failures: Telegram API errors, invalid chat IDs.

#### 1.4 Dream Content Validation

- **Overview:**  
  Checks if the parsed dream content is valid (non-empty and sufficiently long).

- **Nodes Involved:**  
  - Route: Valid Dream Input (If node)  
  - Send Processing Status (Telegram node)  
  - Send Usage Instructions (Telegram node)

- **Node Details:**  
  - **Route: Valid Dream Input**  
    - Type: If Node  
    - Role: Evaluates if dream content length > 5 characters to confirm valid input.  
    - Outputs: True branch sends processing status, false branch sends usage instructions.  
  - **Send Processing Status**  
    - Type: Telegram Node  
    - Role: Notifies user that dream is being analyzed in the selected style.  
    - Configuration: Uses `$json.style` for dynamic message.  
  - **Send Usage Instructions**  
    - Type: Telegram Node  
    - Role: Provides instructions on how to use the bot properly when input is invalid or missing.

#### 1.5 AI Dream Analysis

- **Overview:**  
  Uses an LLM (via LangChain Agent) to analyze the dream, generate a cinematic video prompt, and produce a psychological interpretation.

- **Nodes Involved:**  
  - Set API Configuration (Set node)  
  - AI Dream Analyzer Agent (LangChain Agent node)  
  - Extract Video Prompt & Analysis (Code node)  
  - OpenRouter LLM (LangChain LLM node, auxiliary)

- **Node Details:**  
  - **Set API Configuration**  
    - Type: Set Node  
    - Role: Sets Veo3 API URL and passes existing data downstream.  
  - **AI Dream Analyzer Agent**  
    - Type: LangChain Agent Node  
    - Role: Sends combined style and dream content text to the LLM agent with detailed system instructions to return JSON with video prompt and analysis fields.  
    - Configuration: Includes system message defining styles, output format restrictions (JSON only), and rules for prompt generation.  
    - Expressions: Input text composed of dream content and style description.  
  - **OpenRouter LLM**  
    - Type: LangChain LLM Chat Node  
    - Role: Secondary LLM node configured but chained into agent node for language model backend.  
  - **Extract Video Prompt & Analysis**  
    - Type: Code Node  
    - Role: Parses the JSON output from the AI Dream Analyzer Agent, with fallback defaults if parsing fails.  
    - Input: Raw text output from AI agent.  
    - Output: Parsed structured data including video prompt and detailed dream analysis (theme, mood, symbols, meaning, type).  
    - Edge Cases: Malformed AI output JSON, empty or unexpected responses.

#### 1.6 Video Generation with Veo3

- **Overview:**  
  Sends the generated video prompt to Google Veo3 API to create an 8-second cinematic video with native audio.

- **Nodes Involved:**  
  - Generate Dream Video (Veo3) (HTTP Request node)

- **Node Details:**  
  - **Generate Dream Video (Veo3)**  
    - Type: HTTP Request Node  
    - Role: POSTs the video prompt to Veo3 API endpoint with parameters for aspect ratio and duration.  
    - Configuration: Uses HTTP Header Authentication with fal.ai API key via credential named `Authorization` header with format `Key YOUR_API_KEY`.  
    - Body: JSON including `prompt`, `aspect_ratio` ("16:9"), and `duration` ("8s").  
    - Outputs: JSON response containing `status_url` for polling.  
    - Failure Modes: API key issues, network errors, invalid prompt format.

#### 1.7 Video Generation Polling & Completion Handling

- **Overview:**  
  Waits a fixed time, polls the Veo3 API for video generation status, and routes based on completion.

- **Nodes Involved:**  
  - Wait for Video Generation (Wait node)  
  - Poll Generation Status (HTTP Request node)  
  - Route: Generation Complete (If node)

- **Node Details:**  
  - **Wait for Video Generation**  
    - Type: Wait Node  
    - Role: Pauses workflow for 2 minutes before status polling.  
  - **Poll Generation Status**  
    - Type: HTTP Request Node  
    - Role: GETs the `status_url` received from video generation response to check current status.  
    - Authentication: Same HTTP Header Auth.  
  - **Route: Generation Complete**  
    - Type: If Node  
    - Role: Checks if status equals `"COMPLETED"`.  
    - True branch proceeds to logging and user notification.  
    - False branch triggers another wait cycle (loop) for continued polling.

#### 1.8 Optional Dream Journal Logging

- **Overview:**  
  Logs all dream data including timestamp, username, style, analysis, and video URL to a Google Sheets document.

- **Nodes Involved:**  
  - Log to Google Sheets (Google Sheets node)

- **Node Details:**  
  - **Log to Google Sheets**  
    - Type: Google Sheets Node  
    - Role: Appends a new row with all relevant dream data.  
    - Configuration: User must supply Google Sheets credential, target spreadsheet document ID, and sheet name. Columns expected: Timestamp, Username, Style, Dream, Theme, Emotion, Type, Meaning, Video URL.  
    - Failure Modes: Credential expiry, missing document or sheet, quota exceeded.

#### 1.9 User Notification

- **Overview:**  
  Sends the generated video file to the user on Telegram with accompanying analysis caption.

- **Nodes Involved:**  
  - Send Dream Video to User (Telegram node)

- **Node Details:**  
  - **Send Dream Video to User**  
    - Type: Telegram Node  
    - Role: Sends the video file URL received from Veo3 API to the chat.  
    - Configuration: Uses `sendVideo` operation, dynamic caption includes style, theme, mood, dream type, and meaning from analysis JSON. Markdown parse mode enabled.  
    - Input: Video URL from polling node response, chat ID from parsed command node.  
    - Failure Modes: File URL invalid or expired, Telegram API errors.

---

### 3. Summary Table

| Node Name                   | Node Type                     | Functional Role                        | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                                      |
|-----------------------------|-------------------------------|-------------------------------------|--------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note                   | Informational overview               | None                           | None                              | ## 🌙 Dream Journal Suite - AI Video Generator ... Setup Time: ~15 minutes                                                       |
| Sticky Note1                | Sticky Note                   | Telegram input explanation           | None                           | None                              | ### 1️⃣ Telegram Input ... Add to Telegram credentials                                                                           |
| Sticky Note2                | Sticky Note                   | Command parser explanation           | None                           | None                              | ### 2️⃣ Command Parser ... Supported Styles: cinematic, ghibli, surreal etc.                                                     |
| Sticky Note3                | Sticky Note                   | AI dream analysis explanation        | None                           | None                              | ### 3️⃣ AI Dream Analysis ... Customize: Edit system prompt                                                                      |
| Sticky Note4                | Sticky Note                   | Veo3 video generation explanation    | None                           | None                              | ### 4️⃣ Veo3 Video Generation ... Create Header Auth credential `Authorization`                                                   |
| Sticky Note5                | Sticky Note                   | Dream journal logging explanation    | None                           | None                              | ### 5️⃣ Dream Journal (Optional) ... Connect Google Sheets credential                                                            |
| Telegram Trigger           | Telegram Trigger              | Receives Telegram messages           | None                           | Parse Dream Command               | ### 1️⃣ Telegram Input                                                                                                           |
| Parse Dream Command         | Code                         | Parses style and dream content       | Telegram Trigger               | Route: Style Help Request         | ### 2️⃣ Command Parser                                                                                                           |
| Route: Style Help Request   | If                           | Routes style help requests           | Parse Dream Command            | Send Available Styles, Route: Valid Dream Input | ### 2️⃣ Command Parser                                                                                                           |
| Send Available Styles       | Telegram                     | Sends style list to user             | Route: Style Help Request (true) | None                            | ### 2️⃣ Command Parser                                                                                                           |
| Route: Valid Dream Input    | If                           | Checks for valid dream content       | Route: Style Help Request (false) | Send Processing Status, Send Usage Instructions | ### 2️⃣ Command Parser                                                                                                           |
| Send Processing Status      | Telegram                     | Notifies user dream is being processed | Route: Valid Dream Input (true) | Set API Configuration            | ### 2️⃣ Command Parser                                                                                                           |
| Send Usage Instructions     | Telegram                     | Sends usage info if dream missing    | Route: Valid Dream Input (false) | None                            | ### 2️⃣ Command Parser                                                                                                           |
| Set API Configuration       | Set                          | Sets Veo3 API URL                    | Send Processing Status         | AI Dream Analyzer Agent           | ### 3️⃣ AI Dream Analysis                                                                                                        |
| AI Dream Analyzer Agent     | LangChain Agent              | Generates video prompt and analysis  | Set API Configuration          | Extract Video Prompt & Analysis   | ### 3️⃣ AI Dream Analysis                                                                                                        |
| OpenRouter LLM              | LangChain LLM Chat           | Backend LLM for AI agent             | AI Dream Analyzer Agent (ai_languageModel) | None                         |                                                                                                                                 |
| Extract Video Prompt & Analysis | Code                     | Parses AI output JSON                | AI Dream Analyzer Agent        | Generate Dream Video (Veo3)       | ### 3️⃣ AI Dream Analysis                                                                                                        |
| Generate Dream Video (Veo3) | HTTP Request                 | Sends prompt to Veo3 video API       | Extract Video Prompt & Analysis | Wait for Video Generation         | ### 4️⃣ Veo3 Video Generation                                                                                                    |
| Wait for Video Generation   | Wait                         | Pauses to wait for video creation    | Generate Dream Video (Veo3), Route: Generation Complete (false) | Poll Generation Status   | ### 4️⃣ Veo3 Video Generation                                                                                                    |
| Poll Generation Status      | HTTP Request                 | Polls Veo3 API for generation status | Wait for Video Generation       | Route: Generation Complete        | ### 4️⃣ Veo3 Video Generation                                                                                                    |
| Route: Generation Complete  | If                           | Checks if video generation complete  | Poll Generation Status          | Log to Google Sheets, Wait for Video Generation (loop) | ### 4️⃣ Veo3 Video Generation                                                                                                    |
| Log to Google Sheets        | Google Sheets                | Logs dream and video data             | Route: Generation Complete     | Send Dream Video to User          | ### 5️⃣ Dream Journal (Optional)                                                                                                |
| Send Dream Video to User    | Telegram                     | Sends generated dream video and analysis | Log to Google Sheets           | None                              | ### 5️⃣ Dream Journal (Optional)                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials:**
   - Use @BotFather to create a Telegram bot.
   - Obtain bot API token.
   - Add Telegram OAuth2 credentials in n8n with this token.

2. **Add Telegram Trigger Node:**
   - Type: Telegram Trigger
   - Parameters: Listen to "message" updates.
   - Configure webhook ID as `dream-bot-webhook`.

3. **Add Code Node "Parse Dream Command":**
   - Purpose: Parse user message for `/dream` commands or `/styles`.
   - JavaScript logic: Extract style keyword and dream content, set defaults.
   - Output: JSON with parsed properties including style, dreamContent, flags.

4. **Add If Node "Route: Style Help Request":**
   - Condition: `$json.isStylesRequest === true`.
   - True branch: Connect to a Telegram node "Send Available Styles".
   - False branch: Connect to If node "Route: Valid Dream Input".

5. **Add Telegram Node "Send Available Styles":**
   - Text: List the 8 supported visual styles with usage example.
   - Chat ID: Use `$json.chatId`.
   - Markdown formatting enabled.

6. **Add If Node "Route: Valid Dream Input":**
   - Condition: `$json.hasDreamContent === true`.
   - True branch: Connect to "Send Processing Status" Telegram node.
   - False branch: Connect to "Send Usage Instructions" Telegram node.

7. **Add Telegram Node "Send Processing Status":**
   - Text: Inform user that dream is being analyzed in chosen style.
   - Use expression to insert `$json.style`.
   - Chat ID from parsed data.

8. **Add Telegram Node "Send Usage Instructions":**
   - Text: Provide usage help for `/dream` command and `/styles`.
   - Chat ID from parsed data.

9. **Add Set Node "Set API Configuration":**
   - Assign Veo3 API URL: `https://queue.fal.run/fal-ai/veo3`.
   - Pass all other fields through.

10. **Add LangChain Agent Node "AI Dream Analyzer Agent":**
    - Input text template: Include dream content and style description.
    - System prompt: Detailed instructions for generating JSON output with video prompt and dream analysis.
    - Connect output to "Extract Video Prompt & Analysis".

11. **Add LangChain LLM Chat Node "OpenRouter LLM":**
    - Set as language model backend for the agent node.
    - Configure OpenRouter API credentials.

12. **Add Code Node "Extract Video Prompt & Analysis":**
    - Parse JSON output from AI agent.
    - If parsing fails, provide fallback default JSON.
    - Output parsed video prompt and analysis fields.

13. **Add HTTP Request Node "Generate Dream Video (Veo3)":**
    - Method: POST
    - URL: Use `$json.veo3ApiUrl`.
    - Body JSON: `{ prompt, aspect_ratio: "16:9", duration: "8s" }`
    - Authentication: HTTP Header Auth with fal.ai API key credential (`Authorization: Key YOUR_API_KEY`).

14. **Add Wait Node "Wait for Video Generation":**
    - Wait duration: 2 minutes.

15. **Add HTTP Request Node "Poll Generation Status":**
    - Method: GET
    - URL: Use `status_url` from previous node response.
    - Authentication: Same as video generation node.

16. **Add If Node "Route: Generation Complete":**
    - Condition: `$json.status === "COMPLETED"`.
    - True branch: Connect to Google Sheets logging.
    - False branch: Loop back to "Wait for Video Generation" for continued polling.

17. **Add Google Sheets Node "Log to Google Sheets":** *(Optional)*
    - Configure Google Sheets credentials.
    - Select document and sheet.
    - Map columns: Timestamp, Username, Style, Dream, Theme, Emotion, Type, Meaning, Video URL.

18. **Add Telegram Node "Send Dream Video to User":**
    - Operation: sendVideo.
    - File: Use video URL from polling node response.
    - Chat ID: From parsed command node.
    - Caption: Include style, analysis theme, mood, type, and meaning using Markdown.
    - Enable Markdown parse mode.

19. **Connect nodes according to the logical flow:**
    - Telegram Trigger → Parse Dream Command → Route: Style Help Request
    - Style help true → Send Available Styles
    - Style help false → Route: Valid Dream Input
    - Valid dream true → Send Processing Status → Set API Configuration → AI Dream Analyzer Agent → Extract Video Prompt & Analysis → Generate Dream Video (Veo3) → Wait for Video Generation → Poll Generation Status → Route: Generation Complete
    - Generation complete true → Log to Google Sheets → Send Dream Video to User
    - Generation complete false → Wait for Video Generation (loop)
    - Valid dream false → Send Usage Instructions

20. **Test the full workflow:**
    - Send `/styles` command in Telegram to verify style list.
    - Send `/dream cinematic I was flying` to verify video generation and analysis.
    - Check Google Sheets logs if enabled.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| 🌙 Dream Journal Suite - AI Video Generator transforms dream descriptions into cinematic AI videos with psychological analysis. Setup time approx. 15 minutes. Commands: `/dream`, `/styles`. Requires Telegram bot, fal.ai API Key, OpenRouter API Key, Google Sheets optional. | Sticky Note overview at workflow start.                                                          |
| For Veo3 API use, create HTTP Header Auth credential named `Authorization` with value `Key YOUR_API_KEY` (fal.ai API key).                                                                                                          | Sticky Note4                                                                                        |
| Supported styles: cinematic, ghibli, surreal, vintage, horror, abstract, watercolor, cyberpunk. Each style has a detailed visual and atmosphere description used by LLM to generate video prompts.                                   | Sticky Note2 and Sticky Note3                                                                     |
| AI Dream Analysis outputs strict JSON with fields: videoPrompt (150-200 word cinematic description including sound), analysis with theme, emotionalTone, symbols, possibleMeaning, dreamType.                                        | Sticky Note3                                                                                      |
| Google Sheets logging is optional; create a sheet with columns Timestamp, Username, Style, Dream, Theme, Emotion, Type, Meaning, Video URL.                                                                                         | Sticky Note5                                                                                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. This process strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.