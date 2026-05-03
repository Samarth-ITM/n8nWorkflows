Nutrition Tracker & Meal Logger with Telegram, Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/nutrition-tracker---meal-logger-with-telegram--gemini-ai-and-google-sheets-7756


# Nutrition Tracker & Meal Logger with Telegram, Gemini AI and Google Sheets

---

## 1. Workflow Overview

This workflow implements a **Nutrition Tracker & Meal Logger** that allows users to interact via Telegram to log meals, receive nutritional analysis, and track their nutrition goals. It integrates **Telegram**, **Google Sheets** for data storage, and **Google Gemini AI** for processing and understanding text, voice, and images.

The workflow is logically divided into the following blocks:

- **1.1 User Input Capture and Routing**  
  Handles incoming Telegram messages (text, voice, images), routes them based on content type for further processing.

- **1.2 User Registration Validation and Management**  
  Checks if users are registered, and if not, runs a registration agent workflow to collect user profile and nutrition targets.

- **1.3 Message Processing and AI Analysis**  
  Processes text messages directly; downloads and transcribes voice messages using AI; downloads and analyzes food images with AI vision for nutritional content.

- **1.4 Meal Data Storage and Profile Updates**  
  Stores parsed meal information into Google Sheets, updates user profiles if requested, and manages user data retrieval.

- **1.5 Report Generation and Delivery**  
  Generates daily nutrition summaries with progress bars and sends personalized reports to users.

- **1.6 Message Formatting and Telegram Response**  
  Formats messages safely in Telegram MarkdownV2, chunks long messages, and sends responses back to Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 User Input Capture and Routing

**Overview:**  
This block captures all incoming messages from Telegram users and routes them based on whether the input is text, voice, or image.

**Nodes Involved:**  
- Telegram Trigger  
- Input Message Router1  
- Download Voice Message  
- Fix mime (for audio)  
- Analyze voice message  
- get_message (Audio/Video message)  
- Download IMAGE  
- Fix mime5 (for images)  
- Analyze image  
- get_message (Media message)  
- get_message (text)  
- get_error_message1  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Entry point capturing all user messages (text, voice, images)  
  - Configuration: Listens for "message" updates from Telegram API  
  - Output: Sends messages to "Registered?" node and "Typing..." action  
  - Possible failures: Telegram API connectivity, webhook misconfiguration  

- **Input Message Router1**  
  - Type: Switch node  
  - Role: Routes messages based on content type: text, voice message, image, or others  
  - Configuration:  
    - Routes to "Text" if `message.text` exists  
    - Routes to "Voice Message" if `message.voice` exists  
    - Routes to "Image" if `message.photo[0].file_id` exists  
    - Fallback route to error message node  
  - Inputs: Telegram Trigger  
  - Outputs: Multiple branches based on message type  

- **Download Voice Message**  
  - Type: Telegram node  
  - Role: Downloads voice message binary file from Telegram API  
  - Configuration: Uses `file_id` from Telegram voice message  
  - Possible failures: File missing, Telegram API errors, network issues  

- **Fix mime**  
  - Type: Code node  
  - Role: Corrects MIME types of downloaded audio files based on file extension  
  - Configuration: Contains a detailed mapping of common MIME types  
  - Input: Binary data from downloaded voice message  
  - Output: Corrected MIME type binary data  

- **Analyze voice message**  
  - Type: LangChain Google Gemini AI node (Audio analysis)  
  - Role: Transcribes and analyzes the content of the voice message  
  - Configuration: Model "gemini-2.5-pro" with prompt "What's in this audio message from telegram user?"  
  - Input: Binary audio data  
  - Output: Textual transcription of voice message  

- **get_message (Audio/Video message)**  
  - Type: Set node  
  - Role: Extracts text transcription and chat ID, formats message for AI agent  
  - Output: JSON with `message` and `chat_id` fields  

- **Download IMAGE**  
  - Type: Telegram node  
  - Role: Downloads the highest resolution photo from Telegram message (up to 4 photos)  
  - Configuration: Uses file IDs from photo array, picks the highest available  
  - Possible failures: Telegram API errors, file not found  

- **Fix mime5**  
  - Type: Code node  
  - Role: Corrects MIME types of downloaded image files similarly to "Fix mime" node  
  - Input: Binary image data  
  - Output: Corrected MIME type binary data  

- **Analyze image**  
  - Type: LangChain Google Gemini AI node (Image analysis)  
  - Role: Performs detailed nutritional analysis on food image using custom prompt with scientific reasoning and macros estimation  
  - Configuration: Uses model "gemini-2.5-pro", input type "binary", operation "analyze"  
  - Output: Text with meal description and macros (Calories, Proteins, Carbs, Fat) in a strict format  

- **get_message (Media message)**  
  - Type: Set node  
  - Role: Extracts analyzed text and chat ID from image analysis for AI agent  
  - Output: JSON with `message` and `chat_id`  

- **get_message (text)**  
  - Type: Set node  
  - Role: Extracts user text message and chat ID for AI agent  
  - Output: JSON with `message` and `chat_id`  

- **get_error_message1**  
  - Type: Set node  
  - Role: Sets an error message when unsupported file types or errors occur  
  - Output: JSON with error `message` and `chat_id`  

**Edge Cases:**  
- Missing or unsupported media files fail gracefully with error message  
- Telegram API rate limits or downtime  
- Voice or image analysis failures due to AI service unavailability or malformed inputs  

---

### 2.2 User Registration Validation and Management

**Overview:**  
Validates if the Telegram user is registered in the system; if not, triggers an AI-driven registration process to collect user profile and nutrition targets.

**Nodes Involved:**  
- Registered? (Google Sheets Lookup)  
- If (Check user ID existence)  
- get_message (register)  
- Register Agent (AI LangChain agent)  
- Simple Memory1 (AI memory for registration)  
- Register User (Google Sheets append)  
- MarkdownV (MarkdownV2 formatter)  
- Send a text message1 (Telegram message sender)  

**Node Details:**

- **Registered?**  
  - Type: Google Sheets node  
  - Role: Looks up the user's `User_ID` in the Profile sheet to check registration  
  - Configuration: Filters rows where User_ID equals Telegram chat ID  
  - Output: Data about registration or empty if not registered  
  - Possible failure: Google Sheets API errors, permission issues  

- **If**  
  - Type: If node  
  - Role: Checks if `User_ID` exists in lookup result to branch logic  
  - Output: Yes (registered) or No (not registered)  

- **get_message (register)**  
  - Type: Set node  
  - Role: Prepares user text and chat ID for registration agent  
  - Output: JSON with user input message and chat_id  

- **Register Agent**  
  - Type: LangChain AI Agent node  
  - Role: Conversational agent guiding new users through registration, collecting Name, Calories_target, Protein_target  
  - System prompt: Friendly, motivational tone with emoji support; calculates targets if needed; ensures only numeric data is stored; confirms registration  
  - Inputs: User message  
  - Outputs: Commands to append user to Profile sheet or continue conversation  

- **Simple Memory1**  
  - Type: AI memory buffer  
  - Role: Maintains conversation context during registration process keyed by chat_id  

- **Register User**  
  - Type: Google Sheets Tool node  
  - Role: Appends new user data (Name, User_ID, Calories_target, Protein_target) to Profile sheet  
  - Configuration: Defines columns explicitly; uses append operation  
  - Possible issues: Duplicates if user registers twice; data validation  

- **MarkdownV**  
  - Type: Code node  
  - Role: Formats AI agent responses safely for Telegram MarkdownV2 with chunking  
  - Output: One or multiple message chunks  

- **Send a text message1**  
  - Type: Telegram node  
  - Role: Sends formatted messages back to Telegram user  
  - Configuration: Parses MarkdownV2, disables attribution  
  - Possible failures: Telegram API errors  

**Edge Cases:**  
- User abandons registration midway  
- Invalid or missing numeric targets  
- Google Sheets failures on append  
- AI agent generating unexpected responses  

---

### 2.3 Message Processing and AI Analysis

**Overview:**  
Processes registered user input messages. Text messages are sent directly to the AI agent. Voice messages are transcribed and analyzed. Images are analyzed for food content and nutritional values.

**Nodes Involved:**  
- Cal IA Agent (AI LangChain agent)  
- Simple Memory (AI memory buffer)  
- Google Gemini Chat Model (AI language model)  
- Append Meal Data (Google Sheets append)  
- Update Profile Data (Google Sheets appendOrUpdate)  
- Get Profile Data (Google Sheets Tool)  
- Get Report (Tool Workflow node invoking report subworkflow)  
- MarkdownV2 (MarkdownV2 formatter)  
- Send a text message (Telegram message sender)  

**Node Details:**

- **Cal IA Agent**  
  - Type: LangChain AI agent  
  - Role: Main conversational engine for logged-in users, managing meal logging, profile updates, and report generation  
  - System prompt: Friendly fitness coach tone with emoji use; uses four tools: appendMealData, updateProfileData, getUserData, getReport  
  - Inputs: User message (text or AI-processed content)  
  - Outputs: Commands to tools or conversational responses  

- **Simple Memory**  
  - Type: AI memory buffer  
  - Role: Maintains conversation context by chat_id for natural interaction continuity  

- **Google Gemini Chat Model**  
  - Type: AI language model node  
  - Role: Executes the language model calls used by Cal IA Agent  

- **Append Meal Data**  
  - Type: Google Sheets Tool node  
  - Role: Stores meal data (date, description, calories, proteins, carbs, fats, user_id) in Meals sheet  
  - Configuration: Defines columns explicitly; appends rows  
  - Possible failures: Google Sheets write errors  

- **Update Profile Data**  
  - Type: Google Sheets Tool node  
  - Role: Updates user profile data in Profile sheet, overwriting existing values for Name, Calories_target, Protein_target  
  - Configuration: appendOrUpdate operation matching by User_ID  
  - Validation: Only updates changed fields to avoid unnecessary writes  

- **Get Profile Data**  
  - Type: Google Sheets Tool node  
  - Role: Retrieves user profile information based on User_ID  
  - Used by AI agent to fetch current profile targets  

- **Get Report**  
  - Type: Tool Workflow node  
  - Role: Invokes the Report Subworkflow passing User_ID and Date to generate daily nutrition summary  

- **MarkdownV2**  
  - Type: Code node  
  - Role: Formats AI responses safely in Telegram MarkdownV2 with chunking for long messages  

- **Send a text message**  
  - Type: Telegram node  
  - Role: Sends the AI agent's responses back to the user on Telegram  

**Edge Cases:**  
- AI model errors or rate limits  
- Google Sheets API failures during read/write  
- Data format mismatches from AI outputs  
- User requests unsupported by defined tools  

---

### 2.4 Meal Data Storage and Profile Updates

**Overview:**  
Handles storing meal entries, updating user profiles, and fetching user data from Google Sheets.

**Nodes Involved:**  
- Append Meal Data  
- Update Profile Data  
- Get Profile Data  

**Node Details:**  
These nodes are Google Sheets Tool nodes configured to read or write user data and meals using explicit column mappings. Append Meal Data stores meals, Update Profile Data updates profile targets, and Get Profile Data fetches current profile info for AI decision making.

---

### 2.5 Report Generation and Delivery

**Overview:**  
Generates personalized daily nutrition reports by fetching meals and profile targets, calculating totals, formatting progress bars, and delivering the summary.

**Nodes Involved:**  
- Get report (Execute Workflow Trigger node)  
- Get Meals Info (Google Sheets lookup for daily meals)  
- Get User Info (Google Sheets lookup for user profile)  
- Get Data (Set node to extract macros fields)  
- Unify data (Code node to sum totals)  
- Get chart message (Code node to produce MarkdownV2 formatted report with progress bars)  
- Send back message (Set node)  
- Send a text message (Telegram node)  

**Node Details:**

- **Get report**  
  - Triggers subworkflow for daily report generation, passing User_ID and Date  

- **Get Meals Info**  
  - Retrieves all meals logged for user on given date from Meals sheet  

- **Get User Info**  
  - Retrieves user profile data (calorie/protein targets)  

- **Get Data**  
  - Extracts Calories, Proteins, Carbs, Fats from meals for aggregation  

- **Unify data**  
  - Sums Calories, Proteins, Carbs, Fats from all meals to get daily totals  

- **Get chart message**  
  - Produces a Telegram MarkdownV2 safe message containing:  
    - Greeting with user name  
    - Calories and protein progress bars with percentage  
    - Carbs and fats summary  
  - Uses custom functions to escape Markdown, chunk message, and create progress bars  

- **Send back message**  
  - Prepares outgoing message for Telegram node  

- **Send a text message**  
  - Sends the formatted report message back to the user on Telegram  

**Edge Cases:**  
- No meals logged for the date → report may show zeros  
- Missing user profile data → fallback behavior  
- Google Sheets API errors during data retrieval  

---

### 2.6 Message Formatting and Telegram Response

**Overview:**  
Ensures all AI-generated messages are safely formatted in Telegram MarkdownV2, chunked if too long, and sent back to the user.

**Nodes Involved:**  
- MarkdownV2  
- MarkdownV  
- Send a text message  
- Send a text message1  

**Node Details:**

- **MarkdownV2 / MarkdownV**  
  - Code nodes with advanced escaping for Telegram MarkdownV2 including safe handling of links, bold, italics, spoilers, and splitting messages into chunks <= 4096 chars  

- **Send a text message / Send a text message1**  
  - Telegram nodes sending the prepared messages to user chats  
  - Uses MarkdownV2 parse mode and disables attribution for clean messages  

---

## 3. Summary Table

| Node Name               | Node Type                                | Functional Role                              | Input Node(s)                | Output Node(s)                 | Sticky Note                                                  |
|-------------------------|-----------------------------------------|----------------------------------------------|-----------------------------|-------------------------------|--------------------------------------------------------------|
| Telegram Trigger         | Telegram Trigger                        | Entry point, captures incoming Telegram messages | -                           | Typing…, Registered?           | See Sticky Note7: Telegram Trigger & User Check              |
| Typing…                 | Telegram                               | Sends Telegram "Typing..." status             | Telegram Trigger            | -                             |                                                              |
| Registered?              | Google Sheets Lookup                   | Checks if user is registered                    | Telegram Trigger            | If                            | See Sticky Note7: Telegram Trigger & User Check              |
| If                      | If                                    | Branches based on user registration status    | Registered?                 | Input Message Router1, get_message(register) |                                                              |
| Input Message Router1    | Switch                                | Routes messages by content type (text, voice, image) | If                      | get_message(text), Download Voice Message, Download IMAGE, get_error_message1 | See Sticky Note11: Message Processing                         |
| get_message (text)       | Set                                   | Extracts text message & chat_id for AI agent  | Input Message Router1       | Cal IA Agent                  |                                                              |
| Download Voice Message   | Telegram                              | Downloads voice message binary                  | Input Message Router1       | Fix mime                      | See Sticky Note11: Message Processing                         |
| Fix mime                 | Code                                  | Corrects MIME type for audio files              | Download Voice Message      | Analyze voice message         |                                                              |
| Analyze voice message    | LangChain Google Gemini AI (Audio)    | Transcribes & analyzes voice message            | Fix mime                   | get_message (Audio/Video message) |                                                              |
| get_message (Audio/Video message) | Set                           | Extracts transcribed text & chat_id              | Analyze voice message       | Cal IA Agent                  |                                                              |
| Download IMAGE           | Telegram                              | Downloads food image                             | Input Message Router1       | Fix mime5                    | See Sticky Note11: Message Processing                         |
| Fix mime5                | Code                                  | Corrects MIME type for image files               | Download IMAGE             | Analyze image                |                                                              |
| Analyze image            | LangChain Google Gemini AI (Image)    | Analyzes food image, estimates nutrition        | Fix mime5                  | get_message (Media message)  | See Sticky Note11: Message Processing                         |
| get_message (Media message) | Set                                | Extracts analyzed food info & chat_id            | Analyze image              | Cal IA Agent                  |                                                              |
| get_error_message1       | Set                                   | Sends error message for unsupported files       | Input Message Router1       | Cal IA Agent                  | See Sticky Note11: Message Processing                         |
| get_message (register)   | Set                                   | Prepares message for registration agent         | If                         | Register Agent                | See Sticky Note10: Register Agent                            |
| Register Agent           | LangChain AI Agent                    | Guides new user registration & collects data    | get_message (register)      | MarkdownV                    | See Sticky Note10: Register Agent                            |
| Simple Memory1           | AI Memory Buffer                     | Maintains registration conversation context     | Register Agent              | Register Agent                |                                                              |
| Register User            | Google Sheets Tool                   | Appends new user profile data                    | Register Agent              | Register Agent                | See Sticky Note10: Register Agent                            |
| MarkdownV                | Code                                  | Formats AI registration responses for Telegram  | Register Agent              | Send a text message1          |                                                              |
| Send a text message1     | Telegram                              | Sends registration message to user               | MarkdownV                  | -                            |                                                              |
| Cal IA Agent             | LangChain AI Agent                   | Main AI agent for meal logging, profile update  | get_message (text), get_message (Audio/Video message), get_message (Media message), get_error_message1 | MarkdownV2                   | See Sticky Note13: Main AI Agent                             |
| Simple Memory            | AI Memory Buffer                     | Maintains conversation context for main AI      | Cal IA Agent               | Cal IA Agent                  |                                                              |
| Google Gemini Chat Model | LangChain AI Language Model          | Runs AI language model for Cal IA Agent          | Cal IA Agent               | Cal IA Agent                  |                                                              |
| Append Meal Data         | Google Sheets Tool                   | Stores meal data into Meals sheet                 | Cal IA Agent               | Cal IA Agent                  | See Sticky Note13: Main AI Agent                             |
| Update Profile Data      | Google Sheets Tool                   | Updates user profile targets                       | Cal IA Agent               | Cal IA Agent                  | See Sticky Note13: Main AI Agent                             |
| Get Profile Data         | Google Sheets Tool                   | Retrieves user profile info                         | Cal IA Agent               | Cal IA Agent                  | See Sticky Note13: Main AI Agent                             |
| Get Report               | Tool Workflow Trigger               | Invokes report subworkflow                          | Cal IA Agent               | Cal IA Agent                  | See Sticky Note12: Report Subworkflow                        |
| MarkdownV2               | Code                                  | Formats AI messages safely for Telegram            | Cal IA Agent               | Send a text message           |                                                              |
| Send a text message      | Telegram                              | Sends AI agent messages back to Telegram user      | MarkdownV2                 | -                            |                                                              |
| Get report               | Execute Workflow Trigger            | Starts report generation workflow                   | Cal IA Agent               | Get Meals Info, Get User Info | See Sticky Note12: Report Subworkflow                        |
| Get Meals Info           | Google Sheets Lookup                | Fetches meals logged on a date                       | Get report                 | Get Data                     |                                                              |
| Get User Info            | Google Sheets Lookup                | Fetches user profile info                            | Get report                 | Merge                        |                                                              |
| Get Data                 | Set                                   | Extracts macros fields from meals                    | Get Meals Info             | Unify data                   |                                                              |
| Unify data               | Code                                  | Aggregates daily totals of calories, macros          | Get Data                   | Merge                        |                                                              |
| Merge                   | Merge                                 | Combines user info and daily totals                   | Unify data, Get User Info  | Get chart message            |                                                              |
| Get chart message        | Code                                  | Builds MarkdownV2 formatted daily nutrition summary  | Merge                      | Send back message            | See Sticky Note12: Report Subworkflow                        |
| Send back message        | Set                                   | Prepares final message for Telegram                   | Get chart message          | Send a text message          |                                                              |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram API credentials  
   - Listen for "message" updates  

2. **Add "Typing…" Telegram Node**  
   - Type: Telegram  
   - Operation: sendChatAction  
   - Action: "typing"  
   - Chat ID: `={{ $json.message.chat.id }}`  
   - Connect Telegram Trigger → Typing…  

3. **Add "Registered?" Google Sheets Node**  
   - Type: Google Sheets (Lookup)  
   - Configure credentials for Google Sheets OAuth2  
   - Spreadsheet ID: Profile spreadsheet ID  
   - Sheet: "Profile" (gid=0)  
   - Filter: User_ID = `={{ $json.message.chat.id }}`  
   - Connect Telegram Trigger → Registered?  

4. **Add "If" Node**  
   - Check if registered user exists by verifying User_ID in Registered? output  
   - If yes → proceed to message processing  
   - If no → proceed to registration  

5. **Add "Input Message Router1" Switch Node**  
   - Routes messages based on presence of text, voice, or photo in Telegram message  
   - Conditions:  
     - Text exists → route to get_message (text)  
     - Voice exists → route to Download Voice Message  
     - Photo exists → route to Download IMAGE  
     - Else → route to error message  

6. **Set up Text Message Handling**  
   - Create "get_message (text)" Set Node to extract text and chat_id  
   - Connect Input Message Router1 → get_message (text)  
   - Connect get_message (text) → Cal IA Agent  

7. **Set up Voice Message Handling**  
   - Create "Download Voice Message" Telegram Node to download voice file  
   - Connect Input Message Router1 → Download Voice Message  
   - Create "Fix mime" Code Node to fix MIME types  
   - Connect Download Voice Message → Fix mime  
   - Create "Analyze voice message" AI node (Google Gemini Audio)  
   - Connect Fix mime → Analyze voice message  
   - Create "get_message (Audio/Video message)" Set Node to format transcription and chat_id  
   - Connect Analyze voice message → get_message (Audio/Video message)  
   - Connect get_message (Audio/Video message) → Cal IA Agent  

8. **Set up Image Message Handling**  
   - Create "Download IMAGE" Telegram Node to get highest resolution photo  
   - Connect Input Message Router1 → Download IMAGE  
   - Create "Fix mime5" Code Node for image MIME correction  
   - Connect Download IMAGE → Fix mime5  
   - Create "Analyze image" AI node (Google Gemini Image) with custom nutrition prompt  
   - Connect Fix mime5 → Analyze image  
   - Create "get_message (Media message)" Set Node to format output and chat_id  
   - Connect Analyze image → get_message (Media message)  
   - Connect get_message (Media message) → Cal IA Agent  

9. **Set up Error Handling for Unsupported Files**  
   - Create "get_error_message1" Set Node with error text and chat_id  
   - Connect Input Message Router1 fallback → get_error_message1  
   - Connect get_error_message1 → Cal IA Agent  

10. **Set up Registration Flow**  
    - Create "get_message (register)" Set Node to extract message and chat_id for registration  
    - Connect If (not registered) → get_message (register)  
    - Create "Register Agent" LangChain AI Agent Node with registration system prompt (collect Name, Calories_target, Protein_target, help with targets, append to Google Sheets)  
    - Connect get_message (register) → Register Agent  
    - Create "Simple Memory1" AI memory node keyed by chat_id  
    - Connect Register Agent → Simple Memory1 → Register Agent  
    - Create "Register User" Google Sheets Tool Node to append user data to Profile sheet  
    - Connect Register Agent (AI tool output) → Register User  
    - Create "MarkdownV" Code Node for MarkdownV2 safe formatting  
    - Connect Register Agent → MarkdownV  
    - Create "Send a text message1" Telegram Node to send registration messages  
    - Connect MarkdownV → Send a text message1  

11. **Set up Main AI Agent for Registered Users**  
    - Create "Cal IA Agent" LangChain AI Agent Node with system prompt for nutrition assistant, tools: appendMealData, updateProfileData, getUserData, getReport  
    - Connect get_message (text), get_message (Audio/Video message), get_message (Media message), get_error_message1 → Cal IA Agent  
    - Create "Simple Memory" AI memory node keyed by chat_id  
    - Connect Cal IA Agent → Simple Memory → Cal IA Agent  
    - Create "Google Gemini Chat Model" node for AI language model calls  
    - Connect Cal IA Agent → Google Gemini Chat Model → Cal IA Agent  

12. **Set up Google Sheets Tools for AI Agent**  
    - "Append Meal Data" Google Sheets Tool to append meals to Meals sheet  
    - "Update Profile Data" Google Sheets Tool for updating profile targets (appendOrUpdate by User_ID)  
    - "Get Profile Data" Google Sheets Tool to retrieve profile data by User_ID  
    - Connect these tools as AI tools inside Cal IA Agent  

13. **Set up Report Generation Subworkflow**  
    - Create a separate workflow for report generation (Report Subworkflow) with inputs User_ID and Date  
    - In main workflow, add "Get Report" Tool Workflow node invoking subworkflow with inputs User_ID and Date from AI agent  
    - Connect Cal IA Agent → Get Report → Cal IA Agent  

14. **Build Report Subworkflow**  
    - Nodes:  
      - "Get Meals Info" Google Sheets Lookup filtering meals by date and user  
      - "Get User Info" Google Sheets Lookup for profile data  
      - "Get Data" Set Node extracting Calories, Proteins, Carbs, Fats from meals  
      - "Unify data" Code Node summing daily totals  
      - "Merge" node to join profile and totals  
      - "Get chart message" Code Node to generate a MarkdownV2 formatted summary with progress bars  
      - "Send back message" Set Node preparing output  
      - "Send a text message" Telegram Node sending report  

15. **Set up Message Formatting and Sending**  
    - Create "MarkdownV2" and "MarkdownV" Code nodes for formatting all AI outputs safely for Telegram MarkdownV2  
    - Connect these to "Send a text message" or "Send a text message1" Telegram nodes accordingly  

16. **Configure Credentials**  
    - Telegram API credentials with Bot Token and webhook setup  
    - Google Sheets OAuth2 credentials with access to Profile and Meals spreadsheets  
    - Google Gemini (PaLM) API credentials for AI nodes  

---

## 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| 📘 Cal AI Alternative – Nutrition Assistant: Integrates Telegram, Google Sheets, and Gemini AI. | See Sticky Note at workflow start; explains features and intended user benefits                                |
| Telegram Trigger & User Check: Handles message capture and registration validation               | See Sticky Note7                                                                                               |
| Register Agent: AI-driven user registration and nutrition target setup                           | See Sticky Note10                                                                                              |
| Message Processing: Classifies and analyzes messages (text, voice, image)                        | See Sticky Note11                                                                                              |
| Main AI Agent: Central conversational agent managing meals, profile, and reports                 | See Sticky Note13                                                                                              |
| Report Subworkflow: Generates personalized daily nutrition reports with progress bars           | See Sticky Note12                                                                                              |
| MarkdownV2 Safe Formatter: Code node to ensure Telegram Markdown compliance and chunking         | Used in multiple places for message formatting                                                                |
| Contact for customization or assistance: Email johnsilva11031@gmail.com, LinkedIn profile       | See Sticky Note15                                                                                              |
| Workflow includes detailed MIME type correction for various media files                          | Ensures compatibility with Telegram media downloads                                                           |
| AI model used: Google Gemini 2.5 Pro for text, audio transcription, and image analysis           | Requires Google PaLM API credentials                                                                           |
| Google Sheets Structure: Two main sheets - Profile (User info and targets) and Meals (logged meals) | Document links specified in node parameters                                                                    |

---

**Disclaimer:** The text and data processed in this workflow are legal and public. The workflow respects content policies and contains no illegal or offensive content.

---