Calory Tracker & Meal Logger with Telegram, Gemini AI and Data Tables

https://n8nworkflows.xyz/workflows/calory-tracker---meal-logger-with-telegram--gemini-ai-and-data-tables-9294


# Calory Tracker & Meal Logger with Telegram, Gemini AI and Data Tables

---
### 1. Workflow Overview

This workflow implements a **Calorie Tracker & Meal Logger** that interacts with users via **Telegram**, processes natural language inputs using **Google Gemini AI models**, and manages structured data with **Data Tables**. The main purpose is to allow users to log, update, and report meals, track calories, and manage their profile through conversational AI. It supports multiple input types (text, voice, image) and maintains conversation state for context-aware interactions.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception and Preprocessing**  
  Handles incoming Telegram messages (text, voice, images), standardizes them, and routes based on message type.

- **1.2 User Registration and Conversation State Management**  
  Checks if the user is registered, manages user session state, and routes accordingly.

- **1.3 AI Processing & Routing**  
  Uses Google Gemini AI models and Langchain agents to interpret user intents, including meal logging, updates, reports, and profile management.

- **1.4 Meal Logging and Updating**  
  Agents and tools handle the creation, analysis, update, and storage of meal data, including multi-step interactions for meal updates.

- **1.5 Reporting and Profile Management**  
  Agents generate reports and manage user profile data, with updates persisted to Data Tables.

- **1.6 Output & Messaging**  
  Formats AI responses into markdown and sends messages back to Telegram users.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Preprocessing

**Overview:**  
This block captures incoming Telegram messages via webhook or Telegram Trigger nodes, standardizes inputs, and routes messages by type (text, voice, image, or error).

**Nodes Involved:**  
- Telegram Trigger  
- Webhook  
- Typing…  
- Standardize Input  
- Input Message Router  
- Analyze Text Message  
- Download Voice Message  
- Download IMAGE  
- Fix mime (voice)  
- Fix mime5 (image)  
- get_message (Audio/Video message)  
- get_message (Media message)  
- get_error_message  
- Format Text Analysis  
- Merge2

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger Node  
  - Role: Entry point for Telegram user messages (text, voice, image)  
  - Configuration: Default webhook for Telegram updates  
  - Inputs: Telegram messages  
  - Outputs: Passes to Typing… and Standardize Input nodes  
  - Edge cases: Telegram webhook errors, message format variations  

- **Webhook**  
  - Type: HTTP Webhook Node  
  - Role: Alternate entry for external triggers (could support other integrations)  
  - Configuration: HTTP webhook with unique ID  
  - Outputs: Feeds into Standardize Input  
  - Edge cases: HTTP errors, unauthorized calls  

- **Typing…**  
  - Type: Telegram node (send chat action)  
  - Role: Sends “typing” status to Telegram to indicate processing  
  - Inputs: From Telegram Trigger  
  - Outputs: None (fire-and-forget)  
  - Edge cases: Telegram API rate limits  

- **Standardize Input**  
  - Type: Set Node  
  - Role: Normalizes user input format for downstream processing  
  - Inputs: From Webhook or Telegram Trigger  
  - Outputs: To Is User Registered? node  
  - Edge cases: Missing or malformed messages  

- **Input Message Router**  
  - Type: Switch Node  
  - Role: Routes input based on message type (text, voice, image, error)  
  - Inputs: From Sequential Routing node  
  - Outputs: To Analyze Text Message, Download Voice Message, Download IMAGE, or get_error_message nodes  
  - Edge cases: Unknown message types  

- **Analyze Text Message**  
  - Type: Google Gemini AI node for text analysis  
  - Role: Processes text messages with AI to extract intent and content  
  - Inputs: From Input Message Router  
  - Outputs: To Format Text Analysis  
  - Edge cases: API failures, retry enabled with 5s delay  

- **Download Voice Message**  
  - Type: Telegram node (file download)  
  - Role: Downloads voice messages for further processing  
  - Inputs: From Input Message Router  
  - Outputs: To Fix mime (voice)  
  - Edge cases: File access errors, Telegram API limits  

- **Download IMAGE**  
  - Type: Telegram node (file download)  
  - Role: Downloads images sent by users  
  - Inputs: From Input Message Router  
  - Outputs: To Fix mime5 (image)  
  - Edge cases: File access errors  

- **Fix mime (voice) & Fix mime5 (image)**  
  - Type: Code Node (JavaScript)  
  - Role: Corrects or standardizes MIME types before AI analysis  
  - Inputs: Downloaded media files  
  - Outputs: To Analyze voice message or Analyze image nodes  
  - Edge cases: Unexpected file formats  

- **get_message (Audio/Video message) & get_message (Media message)**  
  - Type: Set Node  
  - Role: Extracts or formats message content post media analysis  
  - Inputs: From Analyze voice message / Analyze image nodes  
  - Outputs: To Merge2 node  
  - Edge cases: Missing or malformed content  

- **get_error_message**  
  - Type: Set Node  
  - Role: Handles unsupported or error message cases  
  - Inputs: From Input Message Router  
  - Outputs: None or user notification (not detailed)  

- **Format Text Analysis**  
  - Type: Set Node  
  - Role: Formats the AI text analysis output for merging  
  - Inputs: From Analyze Text Message  
  - Outputs: To Merge2  
  - Edge cases: Expression failures  

- **Merge2**  
  - Type: Merge Node  
  - Role: Consolidates different message input streams into a single processing stream  
  - Inputs: From get_message (Audio/Video message), get_message (Media message), and Format Text Analysis  
  - Outputs: The Log Meal Agent  
  - Edge cases: Data mismatch, empty inputs  

---

#### 1.2 User Registration and Conversation State Management

**Overview:**  
This block verifies if the Telegram user is registered, retrieves or sets conversation state, and routes the flow accordingly.

**Nodes Involved:**  
- Is User Registered? (Data Table)  
- If (Condition)  
- Get Conversation State (Data Table)  
- Conversation State Router (Switch)  
- Register User (Data Table Tool)  
- Register Agent (Langchain Agent)  
- get_message (register)  
- Simple Memory1  
- Simple Memory

**Node Details:**

- **Is User Registered?**  
  - Type: Data Table Node  
  - Role: Queries user registration status from the database  
  - Inputs: From Standardize Input  
  - Outputs: To If node  
  - Edge cases: DB connection errors, empty results  

- **If**  
  - Type: If Node  
  - Role: Branches flow based on user registration (Yes/No)  
  - Inputs: From Is User Registered?  
  - Outputs: Yes → Get Conversation State; No → get_message (register)  
  - Edge cases: Expression evaluation errors  

- **Get Conversation State**  
  - Type: Data Table Node  
  - Role: Retrieves current conversation context for the user from DB  
  - Inputs: From If node (Yes branch)  
  - Outputs: To Conversation State Router  
  - Edge cases: Missing state, DB errors  

- **Conversation State Router**  
  - Type: Switch Node  
  - Role: Routes based on conversation state to appropriate agent (update meal, profile, report, or main router)  
  - Inputs: From Get Conversation State  
  - Outputs: Update Meal Agent (Step 2), Update Meal Agent (Step 3), Profile Agent, Cal AI Router Agent  
  - Edge cases: Unknown state values  

- **Register User**  
  - Type: Data Table Tool Node  
  - Role: Creates new user record in DB  
  - Inputs: From Register Agent  
  - Outputs: To Register Agent (Langchain Agent) via ai_tool connection  
  - Edge cases: Duplicate registration, DB write errors  

- **Register Agent**  
  - Type: Langchain Agent Node  
  - Role: Handles registration dialogue and logic using AI  
  - Inputs: From get_message (register) and Simple Memory1 (conversation memory)  
  - Outputs: To MarkdownV (send confirmation message) and Register User via ai_tool  
  - Edge cases: AI model failures, conversation context loss  

- **get_message (register)**  
  - Type: Set Node  
  - Role: Prepares registration message content for agent  
  - Inputs: From If node (No branch)  
  - Outputs: To Register Agent  
  - Edge cases: Missing text  

- **Simple Memory1 & Simple Memory**  
  - Type: Langchain Memory Buffer Window Nodes  
  - Role: Store short-term conversation context for agents (registration and meal logging)  
  - Inputs/Outputs: Connected to respective agents  
  - Edge cases: Memory overflow, context loss  

---

#### 1.3 AI Processing & Routing

**Overview:**  
This block uses AI models to understand user intents and routes the conversation to the appropriate functional agent: meal logging, updating, reporting, or profile management.

**Nodes Involved:**  
- Cal AI Router Agent  
- Sequential Routing (Switch)  
- Google Gemini Chat Model2  
- Google Gemini Chat Model  
- Google Gemini Chat Model1  
- Google Gemini Chat Model3  
- Google Gemini Chat Model4  
- The Log Meal Agent  
- Report Agent  
- Profile Agent  
- Simple Memory2, Simple Memory3, Simple Memory4

**Node Details:**

- **Cal AI Router Agent**  
  - Type: Langchain Agent Node  
  - Role: Main router AI that classifies user intent into functional categories  
  - Inputs: From Google Gemini Chat Model2 and Simple Memory  
  - Outputs: Sequential Routing  
  - Edge cases: Misclassification, AI downtime  

- **Sequential Routing**  
  - Type: Switch Node  
  - Role: Routes classified intent to the corresponding agent (log meal, update meal step 1, report, profile)  
  - Inputs: From Cal AI Router Agent  
  - Outputs: Input Message Router, Update Meal Agent (Step 1), Report Agent, Profile Agent  
  - Edge cases: Undefined intent  

- **Google Gemini Chat Models (1-4)**  
  - Type: Langchain Language Model Nodes  
  - Role: Various AI language models for registration, routing, meal logging, reporting, and profile management  
  - Inputs/Outputs: Connected to respective agents  
  - Edge cases: API limits, model errors  

- **The Log Meal Agent**  
  - Type: Langchain Agent Node  
  - Role: Handles meal logging interactions and generates text responses  
  - Inputs: From Merge2, Simple Memory, Google Gemini Chat Model3  
  - Outputs: MarkdownV2 (format response)  
  - Edge cases: AI errors, memory issues  

- **Report Agent**  
  - Type: Langchain Agent Node  
  - Role: Generates user-specific meal and calorie reports  
  - Inputs: From getUserData (Data Table Tool), Simple Memory3, Google Gemini Chat Model4  
  - Outputs: MarkdownV2  
  - Edge cases: Data retrieval errors  

- **Profile Agent**  
  - Type: Langchain Agent Node  
  - Role: Manages user profile queries and updates  
  - Inputs: From getUserData, Simple Memory4, Google Gemini Chat Model4 and Set Update Profile State  
  - Outputs: MarkdownV2  
  - Edge cases: Profile DB errors  

- **Simple Memory2, Simple Memory3, Simple Memory4**  
  - Type: Langchain Memory Buffer Nodes  
  - Role: Conversation memory buffers for update meal, report, and profile agents respectively  
  - Edge cases: Context loss  

---

#### 1.4 Meal Logging and Updating

**Overview:**  
Handles detailed meal logging, updating existing meals with multi-step AI interactions, and saves meal data to Data Tables.

**Nodes Involved:**  
- Append Meal Data (Data Table Tool)  
- fetchmealdetails (Data Table Tool)  
- updateMeal (Data Table Tool)  
- Save Updated Meal (Data Table)  
- Clear the State (Data Table)  
- Parse AI Output (Code)  
- Re-Analyze Meal (Google Gemini AI)  
- Update Meal Agent (Step 1 - Ask for ID)  
- Update Meal Agent (Step 2 - Ask for Desc)  
- Update Meal Agent (Step 3 - Final Output)  
- State-Setting Node #1 (Data Table)  
- New Description State-Setting Node #2 (Data Table)  
- Send a text message2  
- Send a text message3  
- MarkdownV3, MarkdownV4  
- Create Success Message (Set Node)

**Node Details:**

- **Append Meal Data**  
  - Type: Data Table Tool  
  - Role: Adds new meal entries to the meal log table  
  - Inputs: From The Log Meal Agent (ai_tool)  
  - Edge cases: DB write failures  

- **fetchmealdetails**  
  - Type: Data Table Tool  
  - Role: Retrieves details of a meal for updating  
  - Inputs: From Update Meal Agent (Step 3) via ai_tool  
  - Outputs: To Update Meal Agent for final processing  
  - Edge cases: Missing meal ID  

- **updateMeal**  
  - Type: Data Table Tool  
  - Role: Updates existing meal entry in DB  
  - Inputs: From Update Meal Agent (Step 3)  
  - Outputs: To Update Meal Agent (Step 3)  
  - Edge cases: DB update conflicts  

- **Save Updated Meal**  
  - Type: Data Table  
  - Role: Persists updated meal data after parsing AI output  
  - Inputs: From Parse AI Output  
  - Outputs: Clear the State  
  - Edge cases: Save failures  

- **Clear the State**  
  - Type: Data Table  
  - Role: Resets conversation state after successful update  
  - Inputs: From Save Updated Meal  
  - Outputs: Create Success Message  
  - Edge cases: DB write errors  

- **Parse AI Output**  
  - Type: Code Node  
  - Role: Extracts structured data from AI response for DB update  
  - Inputs: From Re-Analyze Meal  
  - Outputs: Save Updated Meal  
  - Edge cases: Parsing errors  

- **Re-Analyze Meal**  
  - Type: Google Gemini AI Node  
  - Role: Validates and clarifies meal update data before saving  
  - Inputs: From Update Meal Agent (Step 3)  
  - Outputs: Parse AI Output  
  - Edge cases: AI failure  

- **Update Meal Agent (Steps 1-3)**  
  - Type: Langchain Agent Nodes  
  - Role: Handles multi-step meal update dialogue  
    - Step 1: Asks for meal ID  
    - Step 2: Asks for new description  
    - Step 3: Final confirmation and processing  
  - Inputs: From Conversation State Router and updateMeal Data Table Tool  
  - Outputs: MarkdownV3, MarkdownV4, Re-Analyze Meal  
  - Edge cases: User input errors, AI failures  

- **State-Setting Node #1 & New Description State-Setting Node #2**  
  - Type: Data Table Nodes  
  - Role: Save state information at steps 1 and 2 of update meal workflow  
  - Outputs: Send a text message2 and Send a text message3 respectively  
  - Edge cases: State save failures  

- **Send a text message2 and 3**  
  - Type: Telegram nodes  
  - Role: Send stepwise prompts and confirmations to users  
  - Inputs: From State-Setting Nodes  
  - Edge cases: Telegram API errors  

- **MarkdownV3 and MarkdownV4**  
  - Type: Code Nodes  
  - Role: Format messages for meal update steps  
  - Inputs: From Update Meal Agent (Steps 1 and 2)  
  - Outputs: To State-Setting Nodes  
  - Edge cases: Formatting errors  

- **Create Success Message**  
  - Type: Set Node  
  - Role: Generates final confirmation message after update  
  - Inputs: From Clear the State  
  - Outputs: MarkdownV2 (send message)  
  - Edge cases: Missing data  

---

#### 1.5 Reporting and Profile Management

**Overview:**  
Generates daily meal reports and manages user profile data based on AI analysis and Data Table queries.

**Nodes Involved:**  
- Report Agent  
- Profile Agent  
- getUserData (Data Table Tool)  
- getDailyMealReport (Data Table Tool)  
- Update Profile Data (Data Table Tool)  
- Set Update Profile State (Data Table Tool)  
- MarkdownV2  
- Send a text message (Telegram)

**Node Details:**

- **Report Agent**  
  - Type: Langchain Agent  
  - Role: Creates reports from meal data for the user  
  - Inputs: From getUserData and getDailyMealReport, and Simple Memory3  
  - Outputs: MarkdownV2 (formatted report)  
  - Edge cases: Data retrieval errors  

- **Profile Agent**  
  - Type: Langchain Agent  
  - Role: Handles user profile queries and updates through conversation  
  - Inputs: From getUserData, Update Profile Data, Set Update Profile State, and Simple Memory4  
  - Outputs: MarkdownV2  
  - Edge cases: Profile data conflicts  

- **getUserData, getDailyMealReport, Update Profile Data, Set Update Profile State**  
  - Type: Data Table Tool nodes  
  - Role: Fetch and update profile and report data in DB  
  - Inputs: From respective agents via ai_tool connections  
  - Edge cases: DB errors  

- **MarkdownV2**  
  - Type: Code Node  
  - Role: Formats AI-generated text for Telegram message output  
  - Inputs: From Report Agent, Profile Agent, Create Success Message  
  - Outputs: Send a text message (Telegram)  
  - Edge cases: Formatting errors  

- **Send a text message**  
  - Type: Telegram Node  
  - Role: Sends final report or profile messages to user  
  - Inputs: From MarkdownV2  
  - Edge cases: Telegram API errors  

---

#### 1.6 Output & Messaging

**Overview:**  
Final block responsible for sending formatted AI responses to users via Telegram.

**Nodes Involved:**  
- MarkdownV  
- MarkdownV2  
- MarkdownV3  
- MarkdownV4  
- Send a text message1  
- Send a text message  
- Send a text message2  
- Send a text message3  
- Typing…

**Node Details:**

- **Markdown Nodes (V, V2, V3, V4)**  
  - Type: Code Nodes  
  - Role: Format AI output into markdown for better Telegram presentation  
  - Inputs: From respective agents or process nodes  
  - Outputs: Corresponding Send a text message nodes  
  - Edge cases: Syntax errors in markdown  

- **Send a text message nodes**  
  - Type: Telegram Nodes  
  - Role: Deliver AI responses to users in Telegram conversations  
  - Inputs: From Markdown nodes or state-setting nodes  
  - Edge cases: Message delivery failures, rate limits  

- **Typing…**  
  - See above in 1.1  

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                        | Input Node(s)                            | Output Node(s)                                   | Sticky Note                         |
|-----------------------------------|----------------------------------|-------------------------------------|-----------------------------------------|-------------------------------------------------|-----------------------------------|
| Telegram Trigger                  | Telegram Trigger                 | Entry point for Telegram messages   | -                                       | Typing…, Standardize Input                      |                                   |
| Webhook                          | Webhook                         | Alternate HTTP entry point           | -                                       | Standardize Input                               |                                   |
| Typing…                         | Telegram                        | Sends typing action to Telegram      | Telegram Trigger                        | -                                               |                                   |
| Standardize Input                | Set                            | Normalize input data                  | Telegram Trigger, Webhook               | Is User Registered?                             |                                   |
| Is User Registered?              | Data Table                     | Checks registration status           | Standardize Input                      | If                                             |                                   |
| If                              | If                             | Branch based on registration         | Is User Registered?                    | Get Conversation State, get_message (register) |                                   |
| get_message (register)           | Set                            | Prepare registration message         | If                                    | Register Agent                                  |                                   |
| Register Agent                  | Langchain Agent                | Registration dialogue and logic      | get_message (register), Simple Memory1 | MarkdownV, Register User                         |                                   |
| Register User                   | Data Table Tool                | Store new user data                   | Register Agent                         | Register Agent (ai_tool)                         |                                   |
| Get Conversation State          | Data Table                     | Retrieve conversation context        | If (Yes branch)                       | Conversation State Router                        |                                   |
| Conversation State Router       | Switch                        | Route by conversation state          | Get Conversation State                 | Update Meal Agent (Step 2,3), Profile Agent, Cal AI Router Agent |                                   |
| Cal AI Router Agent             | Langchain Agent                | Route user intent                    | Google Gemini Chat Model2, Simple Memory | Sequential Routing                               |                                   |
| Sequential Routing              | Switch                        | Route by intent                      | Cal AI Router Agent                   | Input Message Router, Update Meal Agent (Step 1), Report Agent, Profile Agent |                                   |
| Input Message Router            | Switch                        | Route by input type (text/voice/img)| Sequential Routing                    | Analyze Text Message, Download Voice Message, Download IMAGE, get_error_message |                                   |
| Analyze Text Message            | Google Gemini AI               | NLP analysis of text                 | Input Message Router                  | Format Text Analysis                            |                                   |
| Format Text Analysis            | Set                           | Format AI text analysis              | Analyze Text Message                  | Merge2                                          |                                   |
| Download Voice Message          | Telegram                      | Download voice message               | Input Message Router                  | Fix mime                                         |                                   |
| Fix mime                       | Code                          | MIME correction for voice            | Download Voice Message                | Analyze voice message                           |                                   |
| Analyze voice message          | Google Gemini AI               | Voice message AI analysis            | Fix mime                             | get_message (Audio/Video message)               |                                   |
| get_message (Audio/Video message)| Set                          | Format voice message AI output       | Analyze voice message                 | Merge2                                          |                                   |
| Download IMAGE                 | Telegram                      | Download image                      | Input Message Router                  | Fix mime5                                        |                                   |
| Fix mime5                      | Code                          | MIME correction for image            | Download IMAGE                      | Analyze image                                   |                                   |
| Analyze image                 | Google Gemini AI               | Image AI analysis                    | Fix mime5                            | get_message (Media  message)                     |                                   |
| get_message (Media  message)    | Set                           | Format image AI output               | Analyze image                       | Merge2                                          |                                   |
| get_error_message              | Set                           | Handle unsupported message types    | Input Message Router                  | -                                               |                                   |
| Merge2                        | Merge                         | Merge all input message streams     | get_message (Audio/Video/Media message), Format Text Analysis | The Log Meal Agent                              |                                   |
| The Log Meal Agent             | Langchain Agent                | Meal logging AI processing          | Merge2, Simple Memory, Google Gemini Chat Model3 | MarkdownV2                                      |                                   |
| Append Meal Data               | Data Table Tool                | Append new meal entries             | The Log Meal Agent (ai_tool)          | The Log Meal Agent (ai_tool)                     |                                   |
| getUserData                   | Data Table Tool                | Fetch user data                     | Report Agent, Profile Agent (ai_tool) | Report Agent, Profile Agent (ai_tool)            |                                   |
| getDailyMealReport            | Data Table Tool                | Fetch daily meal report             | Report Agent (ai_tool)                 | Report Agent (ai_tool)                           |                                   |
| Report Agent                  | Langchain Agent                | Generate meal reports               | getUserData, getDailyMealReport, Simple Memory3, Google Gemini Chat Model4 | MarkdownV2                                      |                                   |
| Profile Agent                 | Langchain Agent                | Manage user profile                 | getUserData, Update Profile Data, Set Update Profile State, Simple Memory4, Google Gemini Chat Model4 | MarkdownV2                                      |                                   |
| Update Profile Data           | Data Table Tool                | Update user profile data            | Profile Agent (ai_tool)                | Profile Agent (ai_tool)                          |                                   |
| Set Update Profile State      | Data Table Tool                | Manage profile update state        | Profile Agent (ai_tool)                | Profile Agent (ai_tool)                          |                                   |
| MarkdownV                     | Code                          | Format messages                     | Register Agent                        | Send a text message1                             |                                   |
| Send a text message1          | Telegram                      | Send Telegram message               | MarkdownV                            | -                                               |                                   |
| MarkdownV2                    | Code                          | Format messages                     | The Log Meal Agent, Report Agent, Profile Agent, Create Success Message | Send a text message                             |                                   |
| Send a text message           | Telegram                      | Send Telegram message               | MarkdownV2                          | -                                               |                                   |
| Update Meal Agent (Step 1)    | Langchain Agent                | Ask for meal ID                    | Conversation State Router, Google Gemini Chat Model | MarkdownV3                                      |                                   |
| MarkdownV3                    | Code                          | Format message for step 1          | Update Meal Agent (Step 1)            | State-Setting Node #1                            |                                   |
| State-Setting Node #1         | Data Table                    | Save state for step 1              | MarkdownV3                          | Send a text message2                             |                                   |
| Send a text message2          | Telegram                      | Send Telegram message              | State-Setting Node #1                | -                                               |                                   |
| Update Meal Agent (Step 2)    | Langchain Agent                | Ask for new description            | Conversation State Router, Google Gemini Chat Model | MarkdownV4                                      |                                   |
| MarkdownV4                    | Code                          | Format message for step 2          | Update Meal Agent (Step 2)            | New Description State-Setting Node #2            |                                   |
| New Description State-Setting Node #2 | Data Table            | Save state for step 2              | MarkdownV4                         | Send a text message3                             |                                   |
| Send a text message3          | Telegram                      | Send Telegram message              | New Description State-Setting Node #2 | -                                               |                                   |
| Update Meal Agent (Step 3)    | Langchain Agent                | Finalize meal update               | Conversation State Router, Google Gemini Chat Model, updateMeal (ai_tool) | Re-Analyze Meal                                 |                                   |
| updateMeal                    | Data Table Tool               | Fetch/update meal data             | Update Meal Agent (Step 3)            | Update Meal Agent (Step 3)                       |                                   |
| Re-Analyze Meal              | Google Gemini AI              | Validate meal update               | Update Meal Agent (Step 3)            | Parse AI Output                                  |                                   |
| Parse AI Output              | Code                         | Extract structured data            | Re-Analyze Meal                     | Save Updated Meal                                |                                   |
| Save Updated Meal            | Data Table                   | Save updated meal data             | Parse AI Output                    | Clear the State                                  |                                   |
| Clear the State             | Data Table                   | Reset conversation state           | Save Updated Meal                  | Create Success Message                           |                                   |
| Create Success Message      | Set                          | Confirmation message               | Clear the State                   | MarkdownV2                                       |                                   |
| Send a text message2         | Telegram                     | Send Telegram message              | State-Setting Node #1             | -                                               |                                   |
| Send a text message3         | Telegram                     | Send Telegram message              | New Description State-Setting Node #2 | -                                               |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Purpose: Capture incoming Telegram messages (text, voice, images).  
   - Configure with your Telegram bot credentials and webhook setup.

2. **Add Typing… Node (Telegram)**  
   - Send "typing" action to Telegram when processing starts.  
   - Connect output of Telegram Trigger to Typing… node.

3. **Add Webhook Node**  
   - For alternate HTTP triggers if needed. Configure unique webhook URL.

4. **Create Set Node "Standardize Input"**  
   - Normalize incoming message data structure for downstream nodes.  
   - Connect outputs of Telegram Trigger and Webhook to this node.

5. **Create Data Table Node "Is User Registered?"**  
   - Query user registration status using Telegram user ID.  
   - Connect output of Standardize Input to this node.

6. **Add If Node**  
   - Condition: check if user is registered.  
   - Connect output of Is User Registered? to If node.

7. **For Registered Users (If true branch):**  
   - Add Data Table Node "Get Conversation State" to fetch user session context.  
   - Connect If node (true) to Get Conversation State.

8. **Add Switch Node "Conversation State Router"**  
   - Route based on conversation state values (e.g., update meal step 2, step 3, profile, or normal flow).  
   - Connect Get Conversation State to this router.

9. **Add Langchain Agents:**  
   - "Update Meal Agent (Step 1 - Ask for ID)"  
   - "Update Meal Agent (Step 2 - Ask for Desc)"  
   - "Update Meal Agent (Step 3 - Final Output)"  
   - "Profile Agent"  
   - "Cal AI Router Agent"  
   - "Report Agent"  
   - Connect Conversation State Router outputs to respective agents.

10. **Create Switch Node "Sequential Routing"**  
    - Route intents classified by Cal AI Router Agent to appropriate agent nodes.  
    - Connect Cal AI Router Agent output to Sequential Routing.

11. **Create Switch Node "Input Message Router"**  
    - Route incoming messages by type: text, voice, image, or error.  
    - Connect Sequential Routing output to Input Message Router.

12. **For Text Messages:**  
    - Add Google Gemini AI node "Analyze Text Message".  
    - Connect Input Message Router text branch to this node.

13. **Add Set Node "Format Text Analysis" and Merge Node "Merge2"**  
    - Format AI output and merge with other media inputs.  
    - Connect Analyze Text Message to Format Text Analysis, then to Merge2.

14. **For Voice Messages:**  
    - Add Telegram node "Download Voice Message" to fetch audio file.  
    - Connect Input Message Router voice branch to this node.  
    - Add Code Node "Fix mime" to fix audio MIME type.  
    - Connect Download Voice Message to Fix mime.  
    - Add Google Gemini AI node "Analyze voice message".  
    - Connect Fix mime to Analyze voice message.  
    - Add Set node "get_message (Audio/Video message)".  
    - Connect Analyze voice message to this node, then to Merge2.

15. **For Image Messages:**  
    - Add Telegram node "Download IMAGE" to fetch image file.  
    - Connect Input Message Router image branch to this node.  
    - Add Code Node "Fix mime5" to fix image MIME type.  
    - Connect Download IMAGE to Fix mime5.  
    - Add Google Gemini AI node "Analyze image".  
    - Connect Fix mime5 to Analyze image.  
    - Add Set node "get_message (Media message)".  
    - Connect Analyze image to this node, then to Merge2.

16. **For Errors:**  
    - Add Set node "get_error_message" to handle unsupported messages.  
    - Connect Input Message Router error branch to this node.

17. **Connect Merge2 output to "The Log Meal Agent"**  
    - Handles meal logging AI interaction.

18. **Create Data Table Tool nodes:**  
    - "Append Meal Data" to store new meals (input: The Log Meal Agent ai_tool output).  
    - "fetchmealdetails" and "updateMeal" for meal updates.  
    - "Save Updated Meal" to persist updates.  
    - "Clear the State" to reset conversation state.  
    - "State-Setting Node #1" and "New Description State-Setting Node #2" to manage step states.

19. **Create Code Nodes:**  
    - "Parse AI Output" to extract structured data from AI responses.  
    - "MarkdownV", "MarkdownV2", "MarkdownV3", "MarkdownV4" to format responses.

20. **Add Telegram Send Message nodes:**  
    - "Send a text message1", "Send a text message", "Send a text message2", "Send a text message3" for user communications.

21. **Set up Langchain Memory Buffer nodes:**  
    - "Simple Memory", "Simple Memory1", "Simple Memory2", "Simple Memory3", "Simple Memory4" for respective agents to maintain conversation context.

22. **Add Data Table Tool nodes:**  
    - "getUserData", "getDailyMealReport", "Update Profile Data", "Set Update Profile State" to manage user data and reports.

23. **Link Google Gemini Chat Models:**  
    - "Google Gemini Chat Model", "Google Gemini Chat Model1", "Google Gemini Chat Model2", "Google Gemini Chat Model3", "Google Gemini Chat Model4" to respective AI agents for language processing.

24. **Configure all nodes with appropriate credentials:**  
    - Telegram Bot API credentials for Telegram nodes.  
    - OpenAI or Google Gemini credentials for AI nodes.  
    - Database credentials for Data Table nodes.

25. **Test the workflow end-to-end:**  
    - Send test messages via Telegram for registration, meal logging, updating, and reporting.  
    - Monitor logs for errors or edge case handling.

---

### 5. General Notes & Resources

| Note Content                                        | Context or Link                                                                                 |
|----------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow leverages Google Gemini AI for natural language understanding and generation. | Refer to Google Gemini API documentation for credential setup and usage.                      |
| Telegram Bot integration requires proper webhook configuration and bot token from BotFather. | Telegram Bot API docs: https://core.telegram.org/bots/api                                      |
| Data Tables are used as a lightweight database for user and meal data management within n8n. | n8n Data Table docs: https://docs.n8n.io/nodes/n8n-nodes-base.datatable/                      |
| Langchain agents and memory nodes enable conversational context and multi-turn dialogues. | Langchain integration docs: https://docs.n8n.io/integrations/ai/langchain/                    |
| The workflow includes retry strategies on AI nodes to handle transient failures.          | Retry configured with 5 seconds wait between tries on critical AI nodes.                       |
| MIME fix nodes ensure media files are correctly processed by AI nodes.                     | Important for voice and image message reliability.                                            |

---

**Disclaimer:**  
The provided text originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.