⚛️🐋🤖 Extract Data from YAPE Receipts via Telegram OCR and Store in Google Sheets

https://n8nworkflows.xyz/workflows/-------extract-data-from-yape-receipts-via-telegram-ocr-and-store-in-google-sheets-3073


# ⚛️🐋🤖 Extract Data from YAPE Receipts via Telegram OCR and Store in Google Sheets

### 1. Workflow Overview

This n8n workflow automates the extraction and structured storage of payment data from Yape receipts sent via Telegram. It is designed for users who receive Yape payment receipts as images through a Telegram bot and want to automatically process these images to extract transaction details and log them into Google Sheets without manual intervention.

The workflow is logically divided into four main blocks:

- **1.1 Image Reception & Classification:** Listens for incoming Telegram messages, classifies message types (commands, images), and handles the start command with a welcome message.
- **1.2 Image Retrieval & OCR Processing:** Selects the best quality image from the Telegram message, downloads it, and uses ChatGPT Vision Computing to perform OCR, extracting raw text from the receipt image.
- **1.3 AI Text Analysis & Structuring:** Sends the extracted text to a DeepSeek-powered AI agent to identify and structure key transaction details into a JSON format.
- **1.4 Data Storage & User Notification:** Finds or creates the appropriate Google Sheets file in Google Drive, prepares the structured data for insertion, inserts it into the spreadsheet, and sends a confirmation message back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Image Reception & Classification

- **Overview:**  
  This block listens for incoming Telegram messages, classifies them to determine if they are commands or images, and handles the start command by sending a welcome message.

- **Nodes Involved:**  
  - 🛎️Telegram Listener  
  - 🔀 Message Classifier  
  - 🔀 Start Command Handle  
  - ✉️ Send Welcome Message

- **Node Details:**

  - **🛎️Telegram Listener**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point that triggers the workflow on any Telegram message received by the bot.  
    - *Configuration:* Uses webhook to receive updates from Telegram bot. No parameters set, listens to all messages.  
    - *Input:* External Telegram messages.  
    - *Output:* Passes message data to the Message Classifier node.  
    - *Edge Cases:* Telegram API downtime, webhook misconfiguration, message format variations.

  - **🔀 Message Classifier**  
    - *Type:* Switch  
    - *Role:* Routes messages based on content type: commands, images, or other.  
    - *Configuration:* Conditions to detect start command, image attachments, or other message types.  
    - *Input:* Telegram message data.  
    - *Output:*  
      - Start command → Start Command Handle  
      - Image → Image Retrieval & OCR Processing block  
      - Other → Select Best Quality Image (fallback or alternative image processing)  
    - *Edge Cases:* Unrecognized commands, messages without images, malformed messages.

  - **🔀 Start Command Handle**  
    - *Type:* Switch  
    - *Role:* Specifically handles the "/start" command from Telegram users.  
    - *Configuration:* Checks if the message text equals "/start".  
    - *Input:* Routed from Message Classifier.  
    - *Output:* Sends to Send Welcome Message node.  
    - *Edge Cases:* Case sensitivity, command variants.

  - **✉️ Send Welcome Message**  
    - *Type:* Telegram node (Send Message)  
    - *Role:* Sends a welcome message to the user initiating the bot.  
    - *Configuration:* Predefined welcome text, sent to the chat ID from the trigger.  
    - *Input:* Routed from Start Command Handle.  
    - *Output:* None (end of this branch).  
    - *Edge Cases:* Telegram API errors, user blocked bot.

---

#### 2.2 Image Retrieval & OCR Processing

- **Overview:**  
  This block processes the Telegram message to select the best quality image, downloads it, and uses ChatGPT Vision Computing to extract text via OCR.

- **Nodes Involved:**  
  - 🗂️Select Best Quality Image  
  - 🖼️Download High-Quality Image  
  - 📄 Extract Text with OCR

- **Node Details:**

  - **🗂️Select Best Quality Image**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses Telegram message payload to select the highest resolution image file_id for download.  
    - *Configuration:* Custom code inspects message photo array, picks the largest image variant.  
    - *Input:* Telegram message data from Message Classifier.  
    - *Output:* Passes selected file_id to Download High-Quality Image node.  
    - *Edge Cases:* Messages without photos, multiple images, malformed photo arrays.

  - **🖼️Download High-Quality Image**  
    - *Type:* Telegram node (Get File)  
    - *Role:* Downloads the image file from Telegram servers using the file_id.  
    - *Configuration:* Uses file_id from previous node, downloads file content.  
    - *Input:* file_id from Select Best Quality Image.  
    - *Output:* Binary image data passed to OCR node.  
    - *Edge Cases:* File not found, Telegram API errors, download timeouts.

  - **📄 Extract Text with OCR**  
    - *Type:* OpenAI (ChatGPT Vision Computing)  
    - *Role:* Performs OCR on the downloaded image to extract all visible text.  
    - *Configuration:* Uses OpenAI API key with Vision capabilities, processes binary image input.  
    - *Input:* Binary image data from Download High-Quality Image.  
    - *Output:* Extracted raw text output.  
    - *Edge Cases:* OCR inaccuracies, API rate limits, image quality issues.

---

#### 2.3 AI Text Analysis & Structuring

- **Overview:**  
  This block sends the OCR-extracted text to a DeepSeek AI agent to identify, extract, and structure key transaction details into a JSON object.

- **Nodes Involved:**  
  - 🤖 AI Data Processor  
  - 🧠 AI Model for Processing

- **Node Details:**

  - **🧠 AI Model for Processing**  
    - *Type:* LangChain DeepSeek Language Model node  
    - *Role:* Provides the AI model interface to DeepSeek for text analysis.  
    - *Configuration:* Uses DeepSeek API key, configured for structured data extraction from text.  
    - *Input:* Text from OCR node.  
    - *Output:* Structured JSON data with transaction fields.  
    - *Edge Cases:* API errors, unexpected text formats, incomplete data extraction.

  - **🤖 AI Data Processor**  
    - *Type:* LangChain Agent node  
    - *Role:* Orchestrates the AI model call and processes the response to ensure correct data formatting.  
    - *Configuration:* Connects to AI Model node, may include prompt engineering or validation logic.  
    - *Input:* OCR text from Extract Text with OCR node.  
    - *Output:* Structured transaction data JSON.  
    - *Edge Cases:* AI response parsing errors, malformed JSON, missing fields.

---

#### 2.4 Data Storage & User Notification

- **Overview:**  
  This block manages Google Drive and Google Sheets integration to store the structured transaction data and notifies the user with the processing result.

- **Nodes Involved:**  
  - 🔍 Find Google Sheet in Drive  
  - 🔄 Prepare Data for Insertion  
  - 📑 Insert Data into Google Sheets  
  - ✉️ Send Analysis Result to User

- **Node Details:**

  - **🔍 Find Google Sheet in Drive**  
    - *Type:* Google Drive node  
    - *Role:* Searches for the Google Sheets file where data will be stored; creates it if not found (implicit or external).  
    - *Configuration:* Uses Google Drive API credentials, searches by file name or folder.  
    - *Input:* Structured JSON data from AI Data Processor.  
    - *Output:* File metadata passed to Prepare Data for Insertion node.  
    - *Edge Cases:* File not found, permission errors, API rate limits.

  - **🔄 Prepare Data for Insertion**  
    - *Type:* Code (JavaScript)  
    - *Role:* Transforms the structured JSON data into the format required by Google Sheets API (e.g., array of values matching columns).  
    - *Configuration:* Maps JSON fields to spreadsheet columns as per predefined schema.  
    - *Input:* File metadata and structured data.  
    - *Output:* Formatted data array for insertion.  
    - *Edge Cases:* Missing fields, data type mismatches, formatting errors.

  - **📑 Insert Data into Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Inserts the prepared data row into the specified Google Sheets spreadsheet.  
    - *Configuration:* Uses Google Sheets API credentials, targets specific spreadsheet and sheet, appends new row.  
    - *Input:* Formatted data array from Prepare Data for Insertion.  
    - *Output:* Confirmation of insertion passed to Send Analysis Result node.  
    - *Edge Cases:* API quota exceeded, permission denied, spreadsheet locked.

  - **✉️ Send Analysis Result to User**  
    - *Type:* Telegram node (Send Message)  
    - *Role:* Sends a confirmation message back to the Telegram user with the result of the data extraction and storage.  
    - *Configuration:* Uses chat ID from original message, sends success or error message.  
    - *Input:* Confirmation from Google Sheets insertion.  
    - *Output:* None (end of workflow).  
    - *Edge Cases:* Telegram API errors, user blocked bot.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                          | Input Node(s)                  | Output Node(s)                      | Sticky Note                      |
|-------------------------------|----------------------------------|----------------------------------------|-------------------------------|-----------------------------------|---------------------------------|
| 🛎️Telegram Listener           | Telegram Trigger                 | Entry point for Telegram messages      | External Telegram messages     | 🔀 Message Classifier              |                                 |
| 🔀 Message Classifier          | Switch                          | Routes messages by type                 | 🛎️Telegram Listener           | 🔀 Start Command Handle, 🖼️Retrieve Image Attachment, 🗂️Select Best Quality Image |                                 |
| 🔀 Start Command Handle        | Switch                          | Handles "/start" command                | 🔀 Message Classifier          | ✉️ Send Welcome Message           |                                 |
| ✉️ Send Welcome Message        | Telegram (Send Message)          | Sends welcome message to user           | 🔀 Start Command Handle        | None                              |                                 |
| 🗂️Select Best Quality Image    | Code                           | Selects highest quality image file_id  | 🔀 Message Classifier          | 🖼️Download High-Quality Image     |                                 |
| 🖼️Download High-Quality Image  | Telegram (Get File)              | Downloads image from Telegram servers  | 🗂️Select Best Quality Image    | 📄 Extract Text with OCR          |                                 |
| 📄 Extract Text with OCR        | OpenAI (ChatGPT Vision)          | Performs OCR on image                   | 🖼️Download High-Quality Image, 🖼️Retrieve Image Attachment | 🤖 AI Data Processor             |                                 |
| 🤖 AI Data Processor            | LangChain Agent                 | Processes OCR text into structured data | 📄 Extract Text with OCR       | 🔍 Find Google Sheet in Drive     |                                 |
| 🧠 AI Model for Processing      | LangChain DeepSeek LM           | AI model for text analysis              | 🤖 AI Data Processor (ai_languageModel) | 🤖 AI Data Processor (ai_languageModel) |                                 |
| 🔍 Find Google Sheet in Drive   | Google Drive                    | Finds or creates Google Sheets file    | 🤖 AI Data Processor           | 🔄 Prepare Data for Insertion     |                                 |
| 🔄 Prepare Data for Insertion   | Code                           | Formats data for Google Sheets          | 🔍 Find Google Sheet in Drive  | 📑 Insert Data into Google Sheets |                                 |
| 📑 Insert Data into Google Sheets | Google Sheets                  | Inserts structured data into spreadsheet | 🔄 Prepare Data for Insertion  | ✉️ Send Analysis Result to User   |                                 |
| ✉️ Send Analysis Result to User | Telegram (Send Message)          | Sends confirmation message to user     | 📑 Insert Data into Google Sheets | None                            |                                 |
| 🖼️Retrieve Image Attachment     | Telegram (Get File)              | Retrieves image attachment from message | 🔀 Message Classifier          | 📄 Extract Text with OCR          |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Obtain Credentials**  
   - Set up a Telegram bot via BotFather and obtain the bot token.  
   - Configure Telegram Trigger node with webhook to receive messages.

2. **Add Telegram Trigger Node (🛎️Telegram Listener)**  
   - Type: Telegram Trigger  
   - Configure webhook for your bot token.  
   - No filters; listen to all messages.

3. **Add Switch Node (🔀 Message Classifier)**  
   - Type: Switch  
   - Add conditions to classify messages:  
     - If message text equals "/start" → output 1  
     - If message contains photo attachments → output 2  
     - Else → output 3 (fallback or other handling)

4. **Add Switch Node (🔀 Start Command Handle)**  
   - Type: Switch  
   - Condition: message text equals "/start"  
   - Connect output to Send Welcome Message node.

5. **Add Telegram Node (✉️ Send Welcome Message)**  
   - Type: Telegram (Send Message)  
   - Configure with bot credentials.  
   - Message: Friendly welcome text.  
   - Send to chat ID from trigger.

6. **Add Code Node (🗂️Select Best Quality Image)**  
   - Type: Code (JavaScript)  
   - Input: Telegram message object with photos array.  
   - Logic: Select the photo with the highest resolution (largest file size or dimensions).  
   - Output: file_id of selected image.

7. **Add Telegram Node (🖼️Download High-Quality Image)**  
   - Type: Telegram (Get File)  
   - Use file_id from previous node to download image binary data.

8. **Add Telegram Node (🖼️Retrieve Image Attachment)**  
   - Type: Telegram (Get File)  
   - For messages with single image attachment, retrieve image binary.

9. **Add OpenAI Node (📄 Extract Text with OCR)**  
   - Type: OpenAI (ChatGPT Vision Computing)  
   - Configure with OpenAI API key.  
   - Input: Binary image data from download nodes.  
   - Output: Extracted raw text.

10. **Add LangChain DeepSeek LM Node (🧠 AI Model for Processing)**  
    - Type: LangChain DeepSeek Language Model  
    - Configure with DeepSeek API key.  
    - Input: OCR text.  
    - Output: Structured JSON data.

11. **Add LangChain Agent Node (🤖 AI Data Processor)**  
    - Type: LangChain Agent  
    - Connect to AI Model node.  
    - Process OCR text and produce structured transaction data.

12. **Add Google Drive Node (🔍 Find Google Sheet in Drive)**  
    - Type: Google Drive  
    - Configure with Google Drive credentials.  
    - Search for Google Sheets file by name or folder.  
    - If file does not exist, create it externally or add logic to create.

13. **Add Code Node (🔄 Prepare Data for Insertion)**  
    - Type: Code (JavaScript)  
    - Map structured JSON fields to an array matching Google Sheets columns:  
      id, beneficiaryName, amount, currency, company, date, hour, originalDate, dateToISO, operation, operationNumber, beneficiaryNumber, commission, account, channel, agentCode.

14. **Add Google Sheets Node (📑 Insert Data into Google Sheets)**  
    - Type: Google Sheets  
    - Configure with Google Sheets API credentials.  
    - Target the spreadsheet and sheet.  
    - Append the prepared data row.

15. **Add Telegram Node (✉️ Send Analysis Result to User)**  
    - Type: Telegram (Send Message)  
    - Send confirmation or error message to user chat ID.

16. **Connect Nodes According to Logic:**  
    - Telegram Listener → Message Classifier  
    - Message Classifier → Start Command Handle → Send Welcome Message  
    - Message Classifier → Select Best Quality Image → Download High-Quality Image → Extract Text with OCR  
    - Message Classifier → Retrieve Image Attachment → Extract Text with OCR  
    - Extract Text with OCR → AI Data Processor → Find Google Sheet in Drive → Prepare Data for Insertion → Insert Data into Google Sheets → Send Analysis Result to User  
    - AI Model for Processing connected as ai_languageModel to AI Data Processor.

17. **Set Credentials:**  
    - Telegram Bot Token for all Telegram nodes.  
    - OpenAI API Key for OCR node.  
    - DeepSeek API Key for AI Model node.  
    - Google Drive and Google Sheets API credentials.

18. **Test Workflow:**  
    - Send "/start" command to Telegram bot to verify welcome message.  
    - Send Yape receipt image to Telegram bot and verify data extraction and insertion into Google Sheets.  
    - Confirm user receives confirmation message.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Bot creation is required to receive images. Setup guide: https://core.telegram.org/bots/features#creating-a-new-bot | Telegram Bot setup instructions                                                                  |
| Google Sheets API key is required for data storage. Setup guide: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/ | Google Sheets integration documentation                                                         |
| ChatGPT API key is required for OCR and AI text extraction. Get it here: https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key | OpenAI API key retrieval                                                                         |
| DeepSeek API key is required for AI data structuring. Get it here: https://platform.deepseek.com/api_keys | DeepSeek API key retrieval                                                                       |
| The workflow stores extracted transaction data with columns: id, beneficiaryName, amount, currency, company, date, hour, originalDate, dateToISO, operation, operationNumber, beneficiaryNumber, commission, account, channel, agentCode | Data schema for Google Sheets                                                                   |
| This workflow is ideal for freelancers, businesses, and finance teams to automate Yape receipt processing with 100% accuracy | Use case and benefits overview                                                                  |
| ChatGPT Vision Computing provides high-accuracy OCR capabilities for image text extraction                   | AI technology used for OCR                                                                       |
| DeepSeek AI agent refines and formats extracted text into structured JSON                                   | AI technology used for data structuring                                                         |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.