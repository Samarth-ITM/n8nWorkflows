Personal Finance Tracker with Telegram Bot, Google Gemini Vision, and Sheets

https://n8nworkflows.xyz/workflows/personal-finance-tracker-with-telegram-bot--google-gemini-vision--and-sheets-10871


# Personal Finance Tracker with Telegram Bot, Google Gemini Vision, and Sheets

---

## 1. Workflow Overview

This workflow implements a **Personal Finance Tracker** integrated with a **Telegram bot**, **Google Gemini AI**, **Google Sheets**, and **Google Drive**. Its primary purpose is to enable users to track expenses effortlessly by sending receipts as images, PDFs, or text via Telegram, and to query financial data through natural language. The workflow processes incoming Telegram messages, extracts receipt data using AI and OCR, stores the data in a Google Sheet with associated receipt images uploaded to Google Drive, and answers user queries about expenses and financial advice using AI-powered analysis.

The logic is organized into three main blocks:

- **1.1 Input Reception & Routing**: Captures Telegram messages and determines their type (image, document, or text query).  
- **1.2 Receipt Processing and Logging**: Extracts receipt data from images/PDFs or text, uploads files to Google Drive, parses the data with AI, and appends structured expense data to Google Sheets.  
- **1.3 Financial Query Handling**: Processes user questions about spending and financial advice, leveraging AI with access to historical expense data in Google Sheets, calculator tools, and a memory buffer for conversational context.  

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Routing

**Overview:**  
This block listens for incoming Telegram messages, then uses a Switch node to route the input based on content type (image, document, or text). It enables the workflow to handle diverse input formats appropriately.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- Get a file1 (for images)  
- Get a file2 (for documents)  
- AI Agent1 (for text queries)  
- Sticky Note (Trigger explanation)

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point, listens for all message updates from Telegram bot  
  - *Config:* Triggers on new messages; uses Telegram API credentials  
  - *Input:* Incoming Telegram message  
  - *Output:* JSON with message data  
  - *Edge Cases:* Telegram API connection errors, missing message content  
  - *Sticky Note:* Explains this as the workflow start and content routing node  

- **Switch**  
  - *Type:* Switch  
  - *Role:* Routes messages by content type: image (photo), document (PDF), or text  
  - *Config:*  
    - Condition 1: Checks if message contains photo (exists photo height) → routes to image processing  
    - Condition 2: Checks if document MIME type exists → routes to PDF processing  
    - Default: Assumes text query → routes to AI Agent1  
  - *Input:* Telegram Trigger output  
  - *Output:* To Get a file1 (image), Get a file2 (document), or AI Agent1 (text)  
  - *Edge Cases:* Messages with unsupported file types or missing fields  

- **Get a file1**  
  - *Type:* Telegram node (Get File)  
  - *Role:* Retrieves image file from Telegram servers based on photo file_id  
  - *Config:* Uses photo file_id from message; Telegram API credentials  
  - *Input:* From Switch (image path)  
  - *Output:* Binary file data (image)  
  - *Edge Cases:* Failed file retrieval, invalid file_id  

- **Get a file2**  
  - *Type:* Telegram node (Get File)  
  - *Role:* Retrieves document file (e.g., PDF) from Telegram servers using file_id  
  - *Config:* Uses document file_id; Telegram API credentials  
  - *Input:* From Switch (document path)  
  - *Output:* Binary file data (document)  
  - *Edge Cases:* File retrieval errors, unsupported document type  

- **AI Agent1**  
  - *Type:* LangChain AI Agent  
  - *Role:* Handles financial text queries from users  
  - *Config:* System prompt tailored for financial questions, advice, and spend queries; has access to Google Sheets data and calculator tool  
  - *Input:* Text messages from Switch  
  - *Output:* AI-generated response text  
  - *Edge Cases:* AI parsing errors, lack of data to answer queries  

- **Sticky Note (Trigger/Start Workflow)**  
  - *Content:*  
    ```
    # Trigger/ start workflow

    This is where the workflow begins with a telegram node

    then a switch node that would determine if the input form telegram is an image, text or pdf 
    ```
  - *Context:* Explains the initial routing logic  

---

### 2.2 Receipt Processing and Logging

**Overview:**  
This block processes receipts sent as images or PDFs, extracts relevant data using Google Gemini AI and OCR, uploads files to Google Drive, aggregates data, parses structured JSON outputs, and appends expense records to Google Sheets. It also sends confirmation messages back to users.

**Nodes Involved:**  
- Code in JavaScript  
- Upload file  
- Analyze an image (Google Gemini AI)  
- Extract from File (PDF extractor)  
- Aggregate1  
- AI Agent (LangChain AI Agent for receipts)  
- Append row in sheet (Google Sheets)  
- Send a text message (Telegram)  
- Upload file1 (Google Drive upload for documents)  
- Merge, Merge1  
- Sticky Note (Sorting, extracting and OCR)  
- Sticky Note (Invoice Processing)  

**Node Details:**

- **Code in JavaScript**  
  - *Type:* Code node  
  - *Role:* Converts Telegram image binary data to base64 string for further processing  
  - *Config:* Reads binary data from Get a file1, outputs base64Image and retains binary data  
  - *Input:* Binary image file from Get a file1  
  - *Output:* JSON with base64Image and binary data  
  - *Edge Cases:* No binary data found error if upstream fails  

- **Upload file**  
  - *Type:* Google Drive node  
  - *Role:* Uploads the processed image file to a specific Google Drive folder  
  - *Config:* Uses file_unique_id as file name, uploads to "Monthly receipts" folder  
  - *Input:* Output from Code in JavaScript  
  - *Output:* Google Drive file metadata including webViewLink  
  - *Edge Cases:* Google Drive permission errors, upload failures  

- **Analyze an image**  
  - *Type:* Google Gemini AI (image analysis)  
  - *Role:* OCR and interpret the uploaded receipt image to extract text and data  
  - *Config:* Uses Gemini model "models/gemini-2.5-flash-lite-preview-06-17"  
  - *Input:* Binary image file from Code in JavaScript via Upload file node  
  - *Output:* Text analysis result for receipt parsing  
  - *Edge Cases:* OCR failure, model errors, unsupported image formats  

- **Extract from File**  
  - *Type:* PDF extract node  
  - *Role:* Extracts text content from PDF documents  
  - *Config:* Operation set to PDF extraction  
  - *Input:* Binary PDF file from Get a file2  
  - *Output:* Extracted text content for AI processing  
  - *Edge Cases:* Corrupted PDF, extraction errors  

- **Aggregate1**  
  - *Type:* Aggregate node  
  - *Role:* Combines extracted text and data from different sources into one JSON array for AI Agent  
  - *Config:* Aggregate all item data  
  - *Input:* From Merge nodes that combine text and image analysis  
  - *Output:* Aggregated JSON data passed to AI Agent  
  - *Edge Cases:* Empty aggregation if no valid input  

- **AI Agent**  
  - *Type:* LangChain AI Agent  
  - *Role:* Parses receipt data from aggregated input, extracts date, amount, description, category; outputs structured JSON with confirmation message  
  - *Config:*  
    - Custom system prompt designed for receipt tracking (detailed instructions)  
    - Supports binary and text receipt processing  
    - Outputs JSON with fields: type, data (date, amount, description, category), message  
  - *Input:* Aggregated data from Aggregate1  
  - *Output:* Structured JSON receipt data  
  - *Edge Cases:* Unreadable receipts, invalid files, parsing errors, amount=0 handled as empty string  

- **Append row in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends parsed receipt data as a new row to Google Sheets spreadsheet  
  - *Config:*  
    - Maps fields: Date, Category, Description, Amount in Naira, Google Drive image URL  
    - Uses spreadsheet and sheet IDs pre-configured  
  - *Input:* Output from AI Agent and Google Drive file link from Aggregate1  
  - *Output:* Confirmation of row appended  
  - *Edge Cases:* Google Sheets permission errors, mapping issues  

- **Send a text message**  
  - *Type:* Telegram node  
  - *Role:* Sends confirmation message back to user with receipt processing result  
  - *Config:* Uses chat ID from Telegram Trigger, message from AI Agent output  
  - *Input:* AI Agent output JSON message  
  - *Output:* Telegram message sent  
  - *Edge Cases:* Telegram API errors  

- **Upload file1**  
  - *Type:* Google Drive node  
  - *Role:* Uploads document files (PDFs) to Google Drive folder "Monthly receipts"  
  - *Config:* Uses file_id from Get a file2  
  - *Input:* Binary document file  
  - *Output:* Google Drive metadata  
  - *Edge Cases:* Upload failures  

- **Merge / Merge1**  
  - *Type:* Merge nodes  
  - *Role:* Synchronize branches for combined processing of file upload, extraction, and analysis results  
  - *Config:* Default merge mode, combining multiple inputs  
  - *Input:* Inputs from Upload file, Analyze an image, Extract from File, Upload file1  
  - *Output:* Combined data for aggregation and AI Agent consumption  
  - *Edge Cases:* Mismatched data causing incomplete merge  

- **Sticky Note (Sorting, extracting and OCR)**  
  - *Content:*  
    ```
    # Sorting, extracting and OCR

    _Here images are OCRed with Gemini model or any model of your choice 
    _ PDF are extracted with the extract with the native extract form PDF node
    _ Everything is aggregated with an aggregator node which is then passed to an LLM 
    ```
  - *Context:* Describes the receipt processing logic  

- **Sticky Note (Invoice Processing)**  
  - *Content:*  
    ```
    # Invoice Processing
       The agent processes the information form the invoice and then appends to your excel sheet
    ```
  - *Context:* Explains AI agent role in parsing and logging invoice data  

---

### 2.3 Financial Query Handling

**Overview:**  
This block manages text-based user queries related to spending, financial advice, and expense summaries. It uses an AI agent with access to expense data in Google Sheets, a calculator tool, and a memory buffer to maintain conversational context and provide detailed answers.

**Nodes Involved:**  
- AI Agent1 (LangChain AI Agent for queries)  
- Google Gemini Chat Model1 (AI language model)  
- Simple Memory1 (context memory buffer)  
- Calculator1 (calculator tool)  
- Get row(s) in sheet in Google Sheets1 (fetch expense data)  
- Send a text message1 (Telegram)  
- Sticky Note (Process text Enquires)  

**Node Details:**

- **AI Agent1**  
  - *Type:* LangChain AI Agent  
  - *Role:* Processes natural language financial queries, uses spreadsheet data and calculator for answers or advice  
  - *Config:*  
    - System prompt focused on spending questions, financial advice, and general responses  
    - Supports time period and category filtering, spending trend analysis  
  - *Input:* Text messages from Switch node for queries  
  - *Output:* AI-generated response string  
  - *Edge Cases:* Ambiguous queries, lack of data, AI errors  

- **Google Gemini Chat Model1**  
  - *Type:* LangChain Google Gemini Chat Model  
  - *Role:* Provides conversational AI language model capability for AI Agent1  
  - *Config:* Uses Google Gemini API credentials  
  - *Input:* Text prompt from AI Agent1  
  - *Output:* AI language model text response  
  - *Edge Cases:* API rate limits, connectivity issues  

- **Simple Memory1**  
  - *Type:* LangChain Memory Buffer (windowed)  
  - *Role:* Maintains a conversation context window of 10 recent messages keyed by chat ID  
  - *Config:* Custom session key from chatid, context window length 10  
  - *Input:* Conversation messages from AI Agent1  
  - *Output:* Context for AI Agent1 to use in dialog  
  - *Edge Cases:* Memory overflow, key mismatch  

- **Calculator1**  
  - *Type:* LangChain Tool Calculator  
  - *Role:* Performs calculations as requested by AI Agent1 during query processing  
  - *Input:* Calculation requests from AI Agent1  
  - *Output:* Numerical results to AI Agent1  
  - *Edge Cases:* Invalid math expressions  

- **Get row(s) in sheet in Google Sheets1**  
  - *Type:* Google Sheets Tool  
  - *Role:* Fetches expense data rows from Google Sheets to supply AI Agent1 with historical financial data for analysis  
  - *Config:* Uses spreadsheet and sheet IDs for "Monthly expensese"  
  - *Input:* Data requests from AI Agent1  
  - *Output:* Expense data rows as JSON  
  - *Edge Cases:* Permission errors, empty sheets  

- **Send a text message1**  
  - *Type:* Telegram node  
  - *Role:* Sends AI-generated textual responses back to user via Telegram chat  
  - *Config:* Uses chat ID from Telegram Trigger, text from AI Agent1 output  
  - *Input:* AI Agent1 output text  
  - *Output:* Telegram message sent  
  - *Edge Cases:* Telegram API errors  

- **Sticky Note (Process text Enquires)**  
  - *Content:*  
    ```
    # Process text Enquires 

    This section processes financial questions, advice and information about financial spend, the Gemini model has access to your excel sheet, a simple memory and calculator to aid with all the enquires it gets.
    ```
  - *Context:* Describes the natural language query processing logic  

---

## 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                                   | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                             |
|-------------------------------|----------------------------------|-------------------------------------------------|-----------------------|--------------------------|-------------------------------------------------------------------------------------------------------|
| Telegram Trigger              | Telegram Trigger                 | Entry point, listens for Telegram messages      | -                     | Switch                   | # Trigger/ start workflow This is where the workflow begins with a telegram node then a switch node that would determine if the input form telegram is an image, text or pdf  |
| Switch                       | Switch                          | Routes input by type: image, document, or text  | Telegram Trigger      | Get a file1, Get a file2, AI Agent1 |                                                                                                       |
| Get a file1                  | Telegram (Get File)             | Retrieves photo file from Telegram servers       | Switch                | Code in JavaScript        |                                                                                                       |
| Code in JavaScript           | Code                           | Converts image binary to base64 string           | Get a file1            | Upload file, Analyze an image |                                                                                                       |
| Upload file                  | Google Drive                   | Uploads image file to Google Drive folder        | Code in JavaScript     | Merge                    |                                                                                                       |
| Analyze an image             | Google Gemini AI (image analysis) | OCR & analyze receipt image to extract text      | Code in JavaScript     | Merge                    | # Sorting, extracting and OCR _Here images are OCRed with Gemini model or any model of your choice _ PDF are extracted with the extract with the native extract form PDF node _ Everything is aggregated with an aggregator node which is then passed to an LLM  |
| Get a file2                  | Telegram (Get File)             | Retrieves document (PDF) file from Telegram      | Switch                | Upload file1, Extract from File |                                                                                                       |
| Upload file1                 | Google Drive                   | Uploads document file to Google Drive folder     | Get a file2            | Merge1                   |                                                                                                       |
| Extract from File            | PDF Extract                    | Extracts text from PDF document                   | Get a file2            | Merge1                   |                                                                                                       |
| Merge                       | Merge                         | Combines outputs from Upload file and Analyze an image | Upload file, Analyze an image | If                      |                                                                                                       |
| Merge1                      | Merge                         | Combines outputs from Upload file1 and Extract from File | Upload file1, Extract from File | If                      |                                                                                                       |
| If                         | If                            | Conditional processing after merges               | Merge, Merge1           | Aggregate1, Aggregate1    |                                                                                                       |
| Aggregate1                  | Aggregate                     | Aggregates data from merged inputs                | If                     | AI Agent                 |                                                                                                       |
| AI Agent                    | LangChain AI Agent            | Parses receipt data and outputs structured JSON   | Aggregate1              | Append row in sheet       | # Invoice Processing The agent processes the information form the invoice and then appends to your excel sheet |
| Append row in sheet          | Google Sheets                 | Appends parsed receipt data to Google Sheets     | AI Agent                | Send a text message       |                                                                                                       |
| Send a text message          | Telegram                      | Sends confirmation back to Telegram user         | Append row in sheet     | -                        |                                                                                                       |
| AI Agent1                   | LangChain AI Agent            | Processes financial queries and advice            | Switch                  | Send a text message1      | # Process text Enquires This section processes financial questions, advice and information about financial spend, the Gemini model has access to your excel sheet, a simple memory and calculator to aid with all the enquires it gets. |
| Google Gemini Chat Model1    | Google Gemini Chat Model      | Provides conversational AI for AI Agent1         | AI Agent1               | AI Agent1                 |                                                                                                       |
| Simple Memory1              | LangChain Memory Buffer       | Maintains chat context for AI Agent1              | AI Agent1               | AI Agent1                 |                                                                                                       |
| Calculator1                | LangChain Calculator          | Performs calculations for AI Agent1                | AI Agent1               | AI Agent1                 |                                                                                                       |
| Get row(s) in sheet in Google Sheets1 | Google Sheets Tool           | Fetches expense data for AI Agent1                 | AI Agent1               | AI Agent1                 |                                                                                                       |
| Send a text message1         | Telegram                      | Sends AI query response back to Telegram user     | AI Agent1               | -                        |                                                                                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Trigger node**  
   - Set to trigger on "message" updates  
   - Configure with Telegram API credentials  
   - Position as workflow entry point  

2. **Add a Switch node connected to Telegram Trigger**  
   - Create three outputs:  
     - Output 1 "image": Condition checks if `message.photo[0].height` exists  
     - Output 2 "document": Condition checks if `message.document.mime_type` exists  
     - Output 3 default: For text messages (no image or document)  

3. **For image input (Output 1):**  
   - Add "Get a file1" (Telegram node) configured to get file by `message.photo[0].file_id`  
   - Connect to a "Code in JavaScript" node to extract base64 from binary data:  
     - JS code extracts base64 from the binary "data" property  
   - Connect "Code in JavaScript" to "Upload file" (Google Drive node):  
     - Set file name from `result.file_unique_id`  
     - Upload to folder "Monthly receipts" (set folder ID)  
     - Use Google Drive OAuth2 credentials  
   - Connect "Upload file" to "Analyze an image" node (LangChain Google Gemini):  
     - Model ID: `models/gemini-2.5-flash-lite-preview-06-17`  
     - Set resource to "image" and operation to "analyze"  
     - Use Google Gemini (Google PaLM) API credentials  

4. **For document input (Output 2):**  
   - Add "Get a file2" (Telegram node) to retrieve document by `message.document.file_id`  
   - Connect to "Upload file1" (Google Drive node):  
     - Upload to same folder as images  
   - Connect also to "Extract from File" node:  
     - Configure operation for PDF extraction  
   - Connect "Upload file1" and "Extract from File" to a Merge node ("Merge1")  

5. **Merge paths:**  
   - Add Merge nodes to combine outputs:  
     - Merge "Upload file" and "Analyze an image" → output to "Merge" node  
     - Merge "Upload file1" and "Extract from File" → output to "Merge1" node  
   - Connect both Merge and Merge1 outputs to an "If" node for conditional processing  

6. **Aggregate data:**  
   - Connect "If" node outputs to "Aggregate1" node to combine all extracted data for AI processing  

7. **AI Agent for receipt parsing:**  
   - Add LangChain AI Agent node  
   - Configure with:  
     - Input text from aggregated data's `data` property  
     - System message prompt that instructs parsing receipts from text or binary, extracting date, amount, description, category, with error handling and JSON output format as specified  
     - Enable passthrough of binary images  
     - Output parser should be active for structured JSON output  

8. **Append row to Google Sheets:**  
   - Add Google Sheets node configured to append a row  
   - Map these columns:  
     - Date, Category, Description, Amount, Google Drive image link (from aggregated data)  
   - Use your spreadsheet ID and sheet name (e.g., "Monthly expensese")  
   - Use Google Sheets OAuth2 credentials  

9. **Send Telegram confirmation message:**  
   - Add Telegram node to send text message  
   - Use chat ID from Telegram Trigger message chat  
   - Message text from AI Agent output message field  

10. **For text queries (Output 3 from Switch):**  
    - Add LangChain AI Agent node (AI Agent1) configured with a financial assistant prompt for handling spend queries, category breakdowns, comparisons, and advice  
    - Connect AI Agent1 to:  
      - Google Gemini Chat Model1 for conversational AI  
      - Simple Memory1 for context buffer (session key from chatid, window length 10)  
      - Calculator1 for math computations  
      - Google Sheets Tool node (Get row(s) in sheet in Google Sheets1) to fetch expense data  
    - Connect AI Agent1 output to a Telegram node (Send a text message1) to reply to user queries  

11. **Credential setup:**  
    - Telegram API credential with bot token  
    - Google Sheets OAuth2 API credential with access to your spreadsheet  
    - Google Drive OAuth2 API credential with access to your Drive folder  
    - Google Gemini (PaLM) API credential for AI and OCR functions  

12. **Test the workflow:**  
    - Activate workflow  
    - Send receipt images, PDFs, or text queries to Telegram bot  
    - Verify data extraction, upload, logging, and reply messages  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| # Personal Finance Telegram Bot: Automated Receipt Tracking & Expense Queries This workflow turns your Telegram bot into a smart personal finance assistant. Send receipt photos/PDFs or text, and AI (Google Gemini) extracts date, amount (in NGN), description, and category — then logs it to Google Sheets with the image uploaded to Google Drive. Ask natural language questions like "How much did I spend last month?" or "Total on food this month?" and get AI-powered summaries, breakdowns, and insights using your logged data. Perfect for effortless expense tracking without manual entry! ## How it works - Telegram Trigger detects incoming messages/photos/documents. - Switch routes: images/PDFs → OCR/analysis/upload → AI parsing → log to Sheets. - Text queries → AI agent queries Sheets data + calculator for answers. - Confirmation/summary replies sent back to Telegram. ## Setup steps 1. Connect credentials: Telegram API, Google Sheets (OAuth2), Google Drive, Google Gemini API. 2. Update Google Sheets node with your spreadsheet ID (pre-filled, but verify). 3. Update Google Drive upload nodes with your folder ID for receipts. 4. Activate the workflow and send a test message/receipt to your bot. ## Customization tips - Edit AI Agent prompts to add/remove categories or refine parsing. - Adjust memory window or add more tools for advanced queries. | Sticky Note near workflow start                                                                                      |

---

# Disclaimer

The text provided is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.

---

This document fully describes the workflow structure, logic, node configurations, and provides instructions to reproduce or modify it, enabling advanced users or automation agents to work effectively with this Personal Finance Tracker workflow.