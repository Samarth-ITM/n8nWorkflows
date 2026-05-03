Snap & Track Nutrition: Telegram Food Photos → Gemini Vision AI → Google Sheets

https://n8nworkflows.xyz/workflows/snap---track-nutrition--telegram-food-photos---gemini-vision-ai---google-sheets-10277


# Snap & Track Nutrition: Telegram Food Photos → Gemini Vision AI → Google Sheets

### 1. Workflow Overview

This workflow automates nutritional tracking by processing food photos sent via Telegram, leveraging Google Gemini Vision AI to analyze the meal visually, and logging the extracted structured nutrition data into a Google Sheet. It integrates several services to provide users with an easy way to capture meal photos, receive detailed nutrition estimates, and keep a historical record of their meals.

The workflow’s logic is split into the following blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages with photos.
- **1.2 Image Upload & Acknowledgement:** Uploads the meal photo to Google Drive and informs the user the photo is received.
- **1.3 Visual AI Nutrition Analysis:** Uses Google Gemini Vision AI to analyze the food photo and estimate nutritional content.
- **1.4 AI-Based Text Parsing & Structuring:** Processes the AI vision output with Langchain AI agents to parse and structure nutrition data in a strict format.
- **1.5 Data Logging:** Appends the structured nutrition data as a new row in a Google Sheet.
- **1.6 User Notification:** Sends a final message back to the Telegram user with the meal nutritional summary.
- **1.7 Supporting Nodes:** Includes memory buffering for conversation context, a Google Gemini chat model linked to the AI agent, and multiple sticky notes with setup instructions, troubleshooting, and pro tips.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new Telegram messages, specifically capturing those that contain photos. It downloads the photo for further processing.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node (n8n-nodes-base.telegramTrigger)  
    - Role: Entry point for incoming Telegram messages. It listens for “message” updates and downloads attached media automatically.  
    - Configuration:  
      - Triggers on message updates.  
      - Downloads files automatically from the message (photo).  
    - Inputs: None (trigger node)  
    - Outputs: Message JSON including photo binary data.  
    - Credentials: Uses Telegram API credentials linked to the bot token.  
    - Edge Cases:  
      - If a message without photo is received, no file is downloaded, potentially causing downstream errors.  
      - Telegram API rate limits or bot token invalidity will prevent triggering.  
    - Version: 1.2

#### 2.2 Image Upload & Acknowledgement

- **Overview:**  
  Uploads the received food photo to a designated Google Drive folder and sends an initial Telegram message confirming receipt and ongoing analysis.

- **Nodes Involved:**  
  - Upload file  
  - Send a text message1

- **Node Details:**  
  - **Upload file**  
    - Type: Google Drive node (n8n-nodes-base.googleDrive)  
    - Role: Uploads the meal photo binary data to Google Drive to archive images.  
    - Configuration:  
      - Sets the filename dynamically with current timestamp.  
      - Uploads to a specified Google Drive folder ID (configured folder for meal photos).  
    - Inputs: Binary data from Telegram Trigger (photo).  
    - Outputs: Metadata about uploaded file.  
    - Credentials: Google Drive OAuth2 credentials with permission to write files.  
    - Edge Cases:  
      - Upload failure due to OAuth token expiration or lack of permission.  
      - Folder ID misconfiguration causing upload to fail.  
    - Version: 3

  - **Send a text message1**  
    - Type: Telegram node (n8n-nodes-base.telegram)  
    - Role: Sends a Telegram message to the user confirming photo receipt and that analysis is in progress.  
    - Configuration:  
      - Static message text describing the upload and analysis status.  
      - Chat ID dynamically fetched from the Telegram Trigger node.  
      - Attribution disabled for clean messages.  
    - Inputs: None directly, triggered after Upload file node.  
    - Outputs: Confirmation of message sent.  
    - Credentials: Telegram API credentials.  
    - Edge Cases:  
      - Chat ID not found or invalid causes send failure.  
      - Telegram API errors or rate limits.  
    - Version: 1.2

#### 2.3 Visual AI Nutrition Analysis

- **Overview:**  
  This block uses the Google Gemini Vision AI model to analyze the uploaded food photo. It performs detailed vision-based food component identification, portion size estimation, and macro-nutrient calculations.

- **Nodes Involved:**  
  - Analyze an image

- **Node Details:**  
  - **Analyze an image**  
    - Type: Langchain Google Gemini Vision AI node (@n8n/n8n-nodes-langchain.googleGemini)  
    - Role: Accepts image binary and applies a custom nutrition-focused vision model to identify foods and estimate calories/macros.  
    - Configuration:  
      - Uses a detailed prompt instructing the AI to reason silently and output a strict text format listing meal description and nutrition values (calories, proteins, carbs, fat).  
      - Model: Gemini 2.5 Pro image analysis model selected.  
      - Input type: Binary image data from Telegram Trigger.  
    - Inputs: Image binary data from Telegram Trigger.  
    - Outputs: Plain text with nutrition summary in a fixed structure.  
    - Credentials: Google Gemini API (PaLM) with Vision enabled.  
    - Edge Cases:  
      - Image quality issues (blurry, dark) reduce accuracy.  
      - API rate limits or invalid key cause failures.  
      - Output format deviations require fail safes downstream.  
    - Version: 1

#### 2.4 AI-Based Text Parsing & Structuring

- **Overview:**  
  Parses the raw AI vision output text into a structured JSON object with consistent fields for meal name, date, description, and macros, ensuring correct formatting before storage.

- **Nodes Involved:**  
  - AI Agent  
  - Structured Output Parser  
  - Google Gemini Chat Model  
  - Simple Memory

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain AI Agent node (@n8n/n8n-nodes-langchain.agent)  
    - Role: Processes meal description text with an AI prompt that formats input into strict structured nutrition data.  
    - Configuration:  
      - Uses a prompt defining input fields and exact output format.  
      - Pulls meal description from AI vision output.  
      - Applies formatting rules (e.g., date format, no blanks, short meal name).  
      - Links to AI language model and output parser nodes for modularity.  
    - Inputs: Text from “Analyze an image” node (via Simple Memory and Gemini Chat Model).  
    - Outputs: Structured JSON with meal_name, date, meal_description, calories, protein, fats, carbs.  
    - Edge Cases:  
      - Parsing errors if AI output deviates.  
      - Empty or invalid meal descriptions trigger fallback output.  
    - Version: 2.2

  - **Structured Output Parser**  
    - Type: Langchain Output Parser Structured node  
    - Role: Validates and enforces JSON schema for AI Agent output.  
    - Configuration:  
      - JSON schema example specifying required fields (meal_name, date, meal_description, calories, protein, fats, carbs).  
    - Inputs: AI Agent output.  
    - Outputs: Strictly formatted JSON object for downstream use.  
    - Edge Cases: Schema validation errors if output malformed.

  - **Google Gemini Chat Model**  
    - Type: Langchain Language Model node (@n8n/n8n-nodes-langchain.lmChatGoogleGemini)  
    - Role: Provides language model backend for AI Agent’s text generation.  
    - Configuration: Uses the same Google Gemini API credentials.  
    - Inputs/Outputs: Connected to AI Agent node as language model resource.  
    - Edge Cases: API key or quota issues.

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window node  
    - Role: Maintains a contextual session memory keyed by the meal description text, allowing context-aware AI reasoning.  
    - Configuration:  
      - Session key derived dynamically from the analyzed text.  
      - Context window length set to 60 messages.  
    - Inputs: Output from “Analyze an image” node.  
    - Outputs: Memory context for AI Agent.  
    - Edge Cases: Memory overflow or session key mismanagement.

#### 2.5 Data Logging

- **Overview:**  
  Inserts the parsed nutrition data as a new row in a configured Google Sheet, mapping fields to specific columns.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**  
  - **Append row in sheet**  
    - Type: Google Sheets node (n8n-nodes-base.googleSheets)  
    - Role: Writes a new row with nutrition data to a Google Sheet tracking meals.  
    - Configuration:  
      - Maps fields from AI Agent output JSON to sheet columns: Meal_Name, Date, Meal_description, Calories, Proteins, Carbs, Fats.  
      - Uses “append” operation to add rows sequentially.  
      - Targets a specific sheet tab by ID within the Google Sheet.  
    - Inputs: Structured JSON output from AI Agent.  
    - Outputs: Confirmation of row appended.  
    - Credentials: Google Sheets OAuth2 credentials with write access.  
    - Edge Cases:  
      - Column mismatch or sheet permission errors cause failures.  
      - Data type or format mismatch may cause append errors.  
    - Version: 4.7

#### 2.6 User Notification

- **Overview:**  
  Sends the user a Telegram message containing the analyzed meal’s nutrition summary along with a confirmation that image and data were saved.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**  
  - **Send a text message**  
    - Type: Telegram node  
    - Role: Sends final nutritional summary text back to the Telegram user.  
    - Configuration:  
      - Message text composed dynamically from “Analyze an image” node’s output text.  
      - Includes a note that the image is saved in Google Drive and data logged.  
      - Chat ID derived from Telegram Trigger node.  
      - Attribution disabled.  
    - Inputs: Structured nutrition text from analysis node.  
    - Outputs: Confirmation of message sent.  
    - Credentials: Telegram API credentials.  
    - Edge Cases: Chat ID errors or Telegram API failures.  
    - Version: 1.2

#### 2.7 Supporting Nodes (Sticky Notes)

- **Overview:**  
  Provides user-facing documentation and troubleshooting guidance embedded in the workflow as sticky notes.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  
  - Sticky Note & Sticky Note1: Quick Setup Guide and Step-by-Step Setup instructions covering Telegram bot creation, Google Sheet and Drive folder setup, credentials configuration, and testing steps.  
  - Sticky Note2: Common first-time issues and checks before daily use, highlighting typical errors and recommended tests.  
  - Sticky Note3: Pro tips for better photo taking, AI accuracy, and data tracking best practices.  
  - These nodes do not impact workflow logic but serve as embedded documentation for maintainers.  
  - Positioning visually groups these notes near relevant nodes.

---

### 3. Summary Table

| Node Name               | Node Type                                      | Functional Role                        | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                          |
|-------------------------|------------------------------------------------|-------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger (n8n-nodes-base.telegramTrigger) | Input Reception - photo message capture | None                   | Analyze an image, Upload file | Sticky Note1 covers setup steps related to this node                                                                  |
| Analyze an image        | Langchain Google Gemini Vision AI (@n8n/n8n-nodes-langchain.googleGemini) | Visual AI Nutrition Analysis           | Telegram Trigger        | AI Agent                | Sticky Note2 covers common issues relevant to this node                                                               |
| Upload file             | Google Drive (n8n-nodes-base.googleDrive)      | Image Upload - save photo to Drive    | Telegram Trigger        | Send a text message1     | Sticky Note1 covers setup related to Drive folder                                                                     |
| Send a text message1    | Telegram (n8n-nodes-base.telegram)              | Acknowledge photo upload & analysis   | Upload file             | None                    | Sticky Note1                                                                                                          |
| AI Agent                | Langchain AI Agent (@n8n/n8n-nodes-langchain.agent) | AI Text Parsing & Structuring          | Analyze an image, Simple Memory | Append row in sheet      | Sticky Note3 covers accuracy tips tied to AI processing                                                               |
| Structured Output Parser| Langchain Output Parser Structured               | Validate & enforce structured output   | AI Agent                | AI Agent (output parser) |                                                                                                                      |
| Google Gemini Chat Model| Langchain Language Model (@n8n/n8n-nodes-langchain.lmChatGoogleGemini) | Language model backend for AI Agent    | AI Agent                | AI Agent                 |                                                                                                                      |
| Simple Memory           | Langchain Memory Buffer Window                    | Session context memory for AI          | Analyze an image        | AI Agent                 |                                                                                                                      |
| Append row in sheet     | Google Sheets (n8n-nodes-base.googleSheets)      | Data Logging - append nutrition row    | AI Agent                | Send a text message      | Sticky Note2 includes tips about sheet update issues                                                                  |
| Send a text message     | Telegram (n8n-nodes-base.telegram)                | Final user notification with nutrition | Append row in sheet     | None                    |                                                                                                                      |
| Sticky Note             | Sticky Note (n8n-nodes-base.stickyNote)          | Setup guide & instructions             | None                   | None                    | Contains Quick Setup Guide                                                                                            |
| Sticky Note1            | Sticky Note (n8n-nodes-base.stickyNote)          | Detailed step-by-step setup instructions | None                   | None                    | Detailed setup for Telegram bot, Google Sheet, Drive, and credentials                                                |
| Sticky Note2            | Sticky Note (n8n-nodes-base.stickyNote)          | Troubleshooting and testing tips        | None                   | None                    | Common first-time issues and test checklist                                                                           |
| Sticky Note3            | Sticky Note (n8n-nodes-base.stickyNote)          | Pro tips for photo taking and accuracy  | None                   | None                    | Best practices for photos, AI accuracy, and nutrition tracking                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use @BotFather on Telegram to create a bot and obtain the bot token.  
   - In n8n, create Telegram API credentials with this token.

2. **Set Up Google APIs Credentials**  
   - Create OAuth2 credentials for Google Drive and Google Sheets for the same Google account.  
   - Obtain Google Gemini (PaLM) API key with Vision enabled.

3. **Prepare Google Sheet**  
   - Create a Google Sheet with columns exactly:  
     `Meal_Name | Date | Meal_description | Calories | Proteins | Carbs | Fats`  
   - Note the Sheet ID from the URL.

4. **Prepare Google Drive Folder**  
   - Create a folder in Google Drive for meal photo uploads (e.g., "Cal AI Meals").  
   - Note the folder ID.

5. **Add Nodes in n8n**

   - **Telegram Trigger**  
     - Type: Telegram Trigger  
     - Parameters: Listen for "message" updates, enable "Download" to true to get photos.  
     - Credentials: Telegram API.  

   - **Analyze an image**  
     - Type: Langchain Google Gemini Vision AI  
     - Parameters:  
       - Input type: Binary  
       - Operation: Analyze  
       - Model: Select "models/gemini-2.5-pro" or equivalent.  
       - Prompt: Use detailed nutrition vision prompt as in the original workflow.  
     - Credentials: Google Gemini API.

   - **Upload file**  
     - Type: Google Drive  
     - Parameters:  
       - Name: Dynamic, e.g., `Meal_{{ $now.format('yyyy-MM-dd HH:mm') }}`  
       - Folder ID: Set to your Drive folder ID.  
     - Credentials: Google Drive OAuth2.  

   - **Send a text message1**  
     - Type: Telegram  
     - Parameters:  
       - Text: Static message "The image of the meal is uploaded, please wait till I analyze it and I will give you back the calories and nutriants of the meal."  
       - Chat ID: `={{ $json.message.from.id }}` from Telegram Trigger.  
     - Credentials: Telegram API.

   - **Simple Memory**  
     - Type: Langchain Memory Buffer Window  
     - Parameters:  
       - Session Key: `={{ $json.content.parts[0].text }}` (from Analyze an image output)  
       - Context Window Length: 60  
     - No credentials needed.

   - **Google Gemini Chat Model**  
     - Type: Langchain Language Model (Google Gemini)  
     - Parameters: Default or as per your API key.  
     - Credentials: Google Gemini API.

   - **Structured Output Parser**  
     - Type: Langchain Output Parser Structured  
     - Parameters: JSON schema example with fields: meal_name, date, meal_description, calories, protein, fats, carbs.

   - **AI Agent**  
     - Type: Langchain AI Agent  
     - Parameters:  
       - Text prompt with instructions and rules for structured nutrition data extraction from meal description (see original prompt).  
       - Link to Google Gemini Chat Model as languageModel.  
       - Link to Structured Output Parser as outputParser.  
       - Link to Simple Memory as ai_memory.  
     - Version: Use compatible node version supporting these features.

   - **Append row in sheet**  
     - Type: Google Sheets  
     - Parameters:  
       - Operation: Append  
       - Document ID: Your Google Sheet ID  
       - Sheet Name or ID: Specific tab ID in your sheet  
       - Columns mapping: Map AI Agent output fields to sheet columns exactly as:  
         `Meal_Name`, `Date`, `Meal_description`, `Calories`, `Proteins`, `Carbs`, `Fats`  
     - Credentials: Google Sheets OAuth2.

   - **Send a text message**  
     - Type: Telegram  
     - Parameters:  
       - Text: Dynamic from “Analyze an image” node’s output text plus note about image and data saved.  
       - Chat ID: From Telegram Trigger node.  
     - Credentials: Telegram API.

6. **Connect Nodes**

   - Telegram Trigger → Analyze an image  
   - Telegram Trigger → Upload file  
   - Upload file → Send a text message1  
   - Analyze an image → Simple Memory → AI Agent  
   - Analyze an image → AI Agent (main input)  
   - AI Agent (output) → Append row in sheet  
   - Append row in sheet → Send a text message  

7. **Final Steps**

   - Save and activate workflow.  
   - Test by sending a food photo to your Telegram bot.  
   - Verify photo appears in Google Drive folder.  
   - Confirm that nutrition data is appended to Google Sheet.  
   - Confirm Telegram messages are sent at each stage.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Quick Setup Guide: Telegram Bot creation, Google Gemini API key, Google Sheets and Drive setup, basic spreadsheet layout | Sticky Note node near start of workflow                                                                      |
| Step-by-Step Setup: Detailed instructions for Telegram, Google Sheet, Drive, credentials, and testing                   | Sticky Note1 providing comprehensive setup instructions                                                      |
| Common First-Time Issues: No bot response, incorrect nutrition, sheet not updating, photo not saving                    | Sticky Note2 outlining troubleshooting steps                                                                 |
| Pro Tips: Best photo practices, AI accuracy considerations, tracking consistency tips                                   | Sticky Note3 with recommendations for better results                                                          |
| Google Gemini API with Vision enabled is mandatory for image analysis                                                    | Credential requirement                                                                                        |
| Telegram bot token must be valid and workflow activated for message reception                                            | Credential and workflow activation requirement                                                               |
| Google Sheets and Drive OAuth credentials require correct scopes for reading/writing files                              | Credential setup notes                                                                                        |
| AI output format is strictly validated; deviations may cause failures downstream                                         | Important for maintaining data integrity                                                                     |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.