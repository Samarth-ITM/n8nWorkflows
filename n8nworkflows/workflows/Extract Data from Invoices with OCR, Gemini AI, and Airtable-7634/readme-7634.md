Extract Data from Invoices with OCR, Gemini AI, and Airtable

https://n8nworkflows.xyz/workflows/extract-data-from-invoices-with-ocr--gemini-ai--and-airtable-7634


# Extract Data from Invoices with OCR, Gemini AI, and Airtable

### 1. Workflow Overview

This workflow automates the extraction of invoice data from image and PDF files dropped into a monitored folder, processes the extracted text using OCR and AI, and saves the structured invoice data to Airtable. It also optionally sends a notification via Telegram upon successful processing.

The workflow logically divides into these key blocks:

- **1.1 Input Reception and File Reading:** Watches a local folder for new invoice files and reads them.
- **1.2 File Type Inspection and Preprocessing:** Determines file type, extracts text via OCR (Tesseract for images, PDF Page Extract for PDFs), and normalizes data.
- **1.3 Data Aggregation:** Aggregates OCR results into a single dataset.
- **1.4 AI-based Invoice Data Extraction:** Uses Google Gemini AI and LangChain agent to parse the aggregated text to extract structured invoice fields.
- **1.5 Data Storage and Notification:** Saves extracted data into Airtable and sends a Telegram notification message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Reading

- **Overview:**  
  This block monitors a specific local folder for new invoice files (JPG, PNG, PDF) and reads the file data for further processing.

- **Nodes Involved:**  
  - Local File Trigger  
  - Read File

- **Node Details:**  

  - **Local File Trigger**  
    - Type: Local file system watcher  
    - Configuration: Monitors `/image-output/ocr` folder using polling, triggers on file addition after writes complete.  
    - Inputs: None (trigger node)  
    - Outputs: Passes file metadata (including file path) to Read File node  
    - Edge cases: Folder path must be correctly mounted and accessible; missing or permissions issues can cause trigger failure.  
    - Sticky Note: Advises to ensure the folder location matches the mounted folder in n8n.

  - **Read File**  
    - Type: File reader  
    - Configuration: Reads file content from the path provided by the trigger, stores data in property `data`.  
    - Inputs: From Local File Trigger (file path)  
    - Outputs: File data and metadata to Check File Type node  
    - Edge cases: File read errors if file is locked or missing; large files might impact performance.

---

#### 2.2 File Type Inspection and Preprocessing

- **Overview:**  
  This block determines the file type of the incoming invoice files, applies the appropriate OCR or text extraction method depending on file type, and aggregates the extracted data.

- **Nodes Involved:**  
  - Check File Type  
  - PDF Page Extract  
  - Tesseract  
  - Switch  
  - Aggregate

- **Node Details:**  

  - **Check File Type**  
    - Type: Conditional check (If node)  
    - Configuration: Checks if file type contains "pdf" (case-insensitive).  
    - Inputs: From Read File (file metadata)  
    - Outputs: Routes PDF files to PDF Page Extract; others to Tesseract.  
    - Edge cases: File extension missing or incorrect may cause misrouting.

  - **PDF Page Extract**  
    - Type: PDF text extractor (community node)  
    - Configuration: Extracts raw text from PDF pages.  
    - Inputs: From Check File Type (PDF files)  
    - Outputs: Passes text to Switch node  
    - Edge cases: PDFs with complex layouts or scanned images may have extraction errors.  
    - Sticky Note: Mentions community node `n8n-nodes-pdf-page-extract`.

  - **Tesseract**  
    - Type: OCR engine (community node)  
    - Configuration: OCR with Page Segmentation Mode (PSM) set to SINGLE_COLUMN and language set to English.  
    - Inputs: From Check File Type (non-PDF files)  
    - Outputs: Passes OCR text to Switch node  
    - Edge cases: Poor image quality or unsupported formats may cause OCR failure or inaccurate OCR results.  
    - Sticky Note: Mentions community node `n8n-nodes-tesseractjs`.

  - **Switch**  
    - Type: Routing node  
    - Configuration: Routes based on file extension ("jpg", "pdf", "png"), all routed to Aggregate node.  
    - Inputs: From PDF Page Extract and Tesseract nodes  
    - Outputs: Routes all to Aggregate node  
    - Edge cases: Unknown extensions are not handled explicitly.

  - **Aggregate**  
    - Type: Data aggregator  
    - Configuration: Aggregates all item data except specified fields (confidence, filename, totalPages, pages, metadata, info).  
    - Inputs: From Switch node  
    - Outputs: Passes aggregated text data to AI extraction node  
    - Edge cases: Large data sets may slow processing.

---

#### 2.3 AI-based Invoice Data Extraction

- **Overview:**  
  This block uses Google Gemini AI via LangChain agent to extract structured invoice fields from the aggregated OCR text.

- **Nodes Involved:**  
  - Extract Invoice Data (Gemini)  
  - Google Gemini Chat Model

- **Node Details:**  

  - **Extract Invoice Data (Gemini)**  
    - Type: LangChain agent node for AI data extraction  
    - Configuration:  
      - Input text is the aggregated OCR data (`$json.data`).  
      - System message instructs to extract key invoice fields (Invoice Number, date, subtotal, tax, total, currency, vendor info, email, phone, ship-to address).  
      - Date formatting required to `YYYY-MM-DD`.  
      - Output format is a structured Telegram response message.  
    - Inputs: Aggregated OCR data from Aggregate node  
    - Outputs: Extracted structured data to Save to Airtable and Telegram nodes  
    - Edge cases: AI response may vary; incorrect or missing fields possible.  
    - Credentials: Uses Google PaLM API via linked credential.

  - **Google Gemini Chat Model**  
    - Type: AI language model node for Google Gemini  
    - Configuration: Model set to `models/gemini-2.5-flash-preview-05-20`  
    - Inputs: Feeds into Extract Invoice Data node as AI language model source  
    - Outputs: AI-generated chat completions to Extract Invoice Data node  
    - Credentials: Uses Google PaLM API credential

---

#### 2.4 Data Storage and Notification

- **Overview:**  
  This block saves the extracted invoice data into an Airtable base and optionally sends a Telegram notification confirming the successful processing.

- **Nodes Involved:**  
  - Save to Airtable  
  - Telegram

- **Node Details:**  

  - **Save to Airtable**  
    - Type: Airtable integration tool  
    - Configuration:  
      - Connects to specific Airtable base and table named "OCR".  
      - Maps AI-extracted fields (Invoice Number, date, subtotal, tax, total, currency, vendor, email, phone, ship-to address) into Airtable columns via expressions.  
      - Mapping mode is defined explicitly with type hints (string, number).  
    - Inputs: Extracted invoice data from AI extraction node  
    - Outputs: Passes data to Telegram node (via main output)  
    - Edge cases: Airtable API errors, credential expiration, or field mismatch may cause failure.

  - **Telegram**  
    - Type: Telegram messaging node  
    - Configuration:  
      - Sends a message to specified chat ID (must be updated by user).  
      - Message content is the AI-generated Telegram response text.  
      - Attribution disabled to avoid appended bot info.  
    - Inputs: From AI extraction node (main output)  
    - Credentials: Requires Telegram Bot API token credential  
    - Edge cases: Invalid chat ID or bot permissions will prevent message delivery  
    - Sticky Note: Reminds users to update the chat ID for message delivery.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                        | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                         |
|-------------------------|----------------------------------|-------------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------------|
| Local File Trigger       | localFileTrigger                 | Watch folder for new files           | —                      | Read File               | Ensure you are pointing the folder location in the trigger to the folder you have mounted in n8n. |
| Read File               | readWriteFile                   | Read file content                    | Local File Trigger     | Check File Type          |                                                                                                   |
| Check File Type          | if                             | Check if file is PDF or image        | Read File              | PDF Page Extract, Tesseract |                                                                                                   |
| PDF Page Extract         | pdfPageExtract                 | Extract text from PDF                | Check File Type (PDF)  | Switch                  | Community node: n8n-nodes-pdf-page-extract                                                        |
| Tesseract                | tesseractNode                  | OCR on image files                   | Check File Type (Image)| Switch                  | Community node: n8n-nodes-tesseractjs                                                             |
| Switch                   | switch                        | Route by file extension              | Tesseract, PDF Page Extract | Aggregate            |                                                                                                   |
| Aggregate                | aggregate                     | Aggregate OCR text data              | Switch                 | Extract Invoice Data (Gemini) |                                                                                                   |
| Extract Invoice Data (Gemini) | langchain.agent            | AI extraction of invoice fields     | Aggregate              | Save to Airtable, Telegram |                                                                                                   |
| Google Gemini Chat Model | lmChatGoogleGemini             | AI language model for Gemini         | —                      | Extract Invoice Data (Gemini) | If not using Airtable, Google Sheets can be used as alternative.                                  |
| Save to Airtable         | airtableTool                  | Store extracted invoice data         | Extract Invoice Data    | Telegram                 |                                                                                                   |
| Telegram                 | telegram                      | Send notification message            | Extract Invoice Data    | —                        | Please ensure to update your Chat ID so that the bot can send the message your DM.                 |
| Sticky Note              | stickyNote                    | Instructional comments               | —                      | —                        | Multiple sticky notes with setup instructions and feature list distributed throughout workflow.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Local File Trigger node**  
   - Type: Local File Trigger  
   - Configure path to folder `/image-output/ocr` (adjust to your mounted path)  
   - Set event to `add` with polling enabled and await write finish enabled  

2. **Add a Read File node**  
   - Type: Read/Write File  
   - Connect input from Local File Trigger  
   - Set `fileSelector` to `{{$json.path}}` to read the newly added file  
   - Data property name: `data` (default)  

3. **Add an If node named "Check File Type"**  
   - Connect input from Read File node  
   - Condition: Check if `fileType` contains "pdf" (case-insensitive)  
   - Output true: PDF files  
   - Output false: Image files  

4. **Add a PDF Page Extract node**  
   - Connect input from Check File Type (true branch)  
   - Configure to include raw text extraction  

5. **Add a Tesseract OCR node**  
   - Connect input from Check File Type (false branch)  
   - Configure OCR options:  
     - PSM: SINGLE_COLUMN  
     - Language: English (eng)  

6. **Add a Switch node**  
   - Connect inputs from both PDF Page Extract and Tesseract nodes  
   - Configure rules to route by file extension: "jpg", "png", "pdf" all go to same output  

7. **Add an Aggregate node**  
   - Connect input from Switch node  
   - Configure to aggregate all item data except `confidence`, `filename`, `totalPages`, `pages`, `metadata`, `info`  

8. **Add the Google Gemini Chat Model node**  
   - Select `models/gemini-2.5-flash-preview-05-20`  
   - Authenticate with Google PaLM API credentials  

9. **Add the LangChain Agent node ("Extract Invoice Data (Gemini)")**  
   - Connect input from Aggregate node  
   - Configure parameters:  
     - Text input: `{{$json.data}}` (aggregated OCR text)  
     - System message: instruct extraction of invoice fields (Invoice Number, date in "YYYY-MM-DD", subtotal, tax, total, currency, vendor name, email, phone (ignore fax), ship-to address)  
     - Prompt output format: Telegram-style message with extracted fields  
   - Set AI language model input connection from Google Gemini Chat Model node  
   - Authenticate with Google PaLM API credentials  

10. **Add Airtable Tool node ("Save to Airtable")**  
    - Connect input from Extract Invoice Data node (main output)  
    - Configure Airtable credentials (Personal Access Token)  
    - Select Airtable base and table (e.g., base "Personal Project Tracker", table "OCR")  
    - Map extracted invoice fields from AI output to Airtable columns explicitly:  
      - Invoice Number, Invoice date, Invoice SubTotal, Invoice tax, Invoice Total, Currency, Vendor name, Email address, Phone number, Ship to address  
    - Use define mapping mode with type hints (string or number)  

11. **Add Telegram node**  
    - Connect input from Extract Invoice Data node (main output)  
    - Configure Telegram API credentials (bot token)  
    - Set chat ID to your Telegram chat (update accordingly)  
    - Set message text to `{{$json.output}}` (AI formatted response)  
    - Disable appended attribution  

12. **Add Sticky Notes as needed**  
    - Include instructions to update chat IDs, folder paths, and note community nodes used  

13. **Connect nodes accordingly**:  
    - Local File Trigger → Read File → Check File Type  
    - Check File Type (true) → PDF Page Extract → Switch  
    - Check File Type (false) → Tesseract → Switch  
    - Switch → Aggregate → Extract Invoice Data (Gemini)  
    - Google Gemini Chat Model → Extract Invoice Data (Gemini) (AI LM input)  
    - Extract Invoice Data (Gemini) → Save to Airtable  
    - Extract Invoice Data (Gemini) → Telegram  

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Supports JPG, PNG, and PDF invoices with automatic processing and field extraction.                   | Workflow feature summary (Sticky Note)                        |
| Community nodes used: `n8n-nodes-pdf-page-extract` and `n8n-nodes-tesseractjs`.                       | Sticky notes reference community nodes                        |
| If Airtable is not preferred, Google Sheets can be used as an alternative data destination.           | Sticky Note4                                                   |
| Telegram chat ID must be updated for notification messages to be delivered correctly.                 | Sticky Note3                                                  |
| For setup help, contact vinaysingh.b@outlook.in                                                      | Sticky Note6                                                  |
| Workflow triggers automatically on file drop; ensure folder path is correctly configured in n8n.     | Sticky Note near Local File Trigger                            |

---

**Disclaimer:** The provided content is extracted exclusively from an automated n8n workflow. It complies with content policies and handles only legal and public data.