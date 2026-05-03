Generate Academic Assignments with Google Gemini & Deliver via Telegram/PDF

https://n8nworkflows.xyz/workflows/generate-academic-assignments-with-google-gemini---deliver-via-telegram-pdf-7369


# Generate Academic Assignments with Google Gemini & Deliver via Telegram/PDF

---

## 1. Workflow Overview

This workflow, titled **"AI-Powered Academic Assignment Generator"**, automates the entire process of generating academic assignments based on student queries submitted via Telegram. It targets educational institutions, academic support services, and AI-powered content generation platforms that require a seamless, end-to-end pipeline for assignment creation, formatting, storage, and delivery.

The workflow is logically divided into the following functional blocks:

### 1.1 Input Reception and Parsing  
Receives structured student assignment requests via Telegram and extracts relevant student and assignment data into a clean JSON format for processing.

### 1.2 AI-Powered Assignment Generation  
Uses Google Gemini AI models with LangChain agents to generate detailed, plagiarism-free academic assignments in a structured JSON format, enforcing academic standards and formatting rules.

### 1.3 Output Validation and Error Handling  
Validates AI-generated JSON output against a strict schema; applies fallback AI models and error handlers to ensure robust processing and data integrity.

### 1.4 Document Generation and Conversion  
Transforms the validated structured data into professionally formatted HTML documents styled for academic submissions, then converts these HTML documents into PDF files.

### 1.5 Storage and Delivery  
Uploads the generated PDFs to Google Drive for organized storage and shares access links directly with students via Telegram messages. Additionally, records all assignment data in a Google Sheets database for tracking and archival.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Parsing

**Overview:**  
This block captures student assignment requests submitted via Telegram messages, listens on specific authorized chat IDs, and extracts structured data fields using regular expressions to prepare for AI processing.

**Nodes Involved:**  
- Student Query Intake Bot  
- Structured Data Parser

**Node Details:**  

- **Student Query Intake Bot**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for incoming Telegram messages from specific chat IDs (configured to accept messages from authorized users only).  
  - *Configuration:* Monitors message updates with chat ID filtering (`1026132948` used in example).  
  - *Inputs:* Telegram messages containing student assignment details.  
  - *Outputs:* Raw message JSON forwarded downstream.  
  - *Edge Cases:* Unauthorized chat messages are ignored; malformed messages may lead to missing data downstream.  

- **Structured Data Parser**  
  - *Type:* Code Node  
  - *Role:* Parses incoming Telegram text messages using regex to extract fields: Name, Faculty, Department, Level, Course, Registration Number, Question, and sets current date automatically.  
  - *Configuration:* JavaScript code extracts fields, defaults missing fields to empty strings, and formats the date as `YYYY-MM-DD`.  
  - *Inputs:* Raw Telegram message JSON from the trigger.  
  - *Outputs:* Clean JSON object with parsed student and assignment data.  
  - *Edge Cases:* Missing or malformed fields result in empty strings; questions spanning multiple lines are captured via regex with the `s` flag.  
  - *Error Potential:* Regex may fail if message format deviates significantly; missing date field is replaced with the current date.

---

### 2.2 AI-Powered Assignment Generation

**Overview:**  
This block uses LangChain agents and Google Gemini AI models to generate comprehensive academic assignments in JSON format based on parsed student inputs. It includes fallback mechanisms and output parsing for reliability.

**Nodes Involved:**  
- Student Assignment Auto-Composer  
- Generator Model (Google Gemini)  
- Fallback Model Generator (Google Gemini - Gemma)  
- Structured Output Parser  
- Handler For Structured Output Parser

**Node Details:**  

- **Student Assignment Auto-Composer**  
  - *Type:* LangChain Agent  
  - *Role:* Main orchestrator node that sends a detailed prompt embedding student input data to AI models to generate a structured, plagiarism-free academic assignment JSON.  
  - *Configuration:* Complex multi-rule prompt specifying JSON schema, academic formatting, writing style, and citation requirements. Uses dynamic expressions to inject student data into prompt.  
  - *Inputs:* Parsed JSON student data from the Structured Data Parser node; AI model outputs.  
  - *Outputs:* Raw AI-generated JSON string requiring validation.  
  - *Edge Cases:* AI may output invalid JSON or incomplete fields; handled by Structured Output Parser and fallback logic.  
  - *Version Requirements:* Requires LangChain agent with output parsing support.  

- **Generator Model (Google Gemini Chat Model6)**  
  - *Type:* LangChain Google Gemini Chat Model  
  - *Role:* Primary AI model providing initial assignment text generation.  
  - *Configuration:* Uses Google Gemini API with credentials for high-quality academic text generation.  
  - *Inputs:* Prompt from Student Assignment Auto-Composer.  
  - *Outputs:* AI-generated text passed back to the composer.  
  - *Edge Cases:* API limits, network errors, or rate limiting may cause failures.  

- **Fallback Model Generator**  
  - *Type:* LangChain Google Gemini Chat Model (Gemma-3-27b-it)  
  - *Role:* Backup AI model invoked if the primary model fails or produces invalid output.  
  - *Configuration:* Uses an alternative Google Gemini model variant with fallback credentials.  
  - *Inputs:* Same prompt as primary model.  
  - *Outputs:* AI-generated fallback text.  
  - *Edge Cases:* Same as primary model; fallback ensures higher availability.  

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Validates AI-generated JSON against a defined JSON schema (fields: Name, Faculty, Department, Level, Course, Date, Matric_No, Questions, Answers, References).  
  - *Configuration:* Enforces required fields, auto-fixes common formatting errors, and triggers retries if output is invalid.  
  - *Inputs:* AI-generated JSON string from Student Assignment Auto-Composer.  
  - *Outputs:* Validated and parsed JSON object.  
  - *Edge Cases:* Malformed JSON or missing fields cause retries or fallback.  

- **Handler For Structured Output Parser**  
  - *Type:* LangChain Google Gemini Chat Model  
  - *Role:* AI node to assist in auto-fixing or reformatting output when Structured Output Parser flags errors.  
  - *Configuration:* Uses fallback Google Gemini API account to improve output validity.  
  - *Inputs:* Error messages and invalid output from Structured Output Parser.  
  - *Outputs:* Attempted corrected output for re-validation.  
  - *Edge Cases:* May loop if output remains invalid; workflow limits should prevent infinite retries.

---

### 2.3 Output Validation and Error Handling

**Overview:**  
Ensures the AI output is consistent, fixes common data issues, and manages timing for stable processing.

**Nodes Involved:**  
- Error Handler  
- Wait Node

**Node Details:**  

- **Error Handler**  
  - *Type:* Code Node  
  - *Role:* Fixes common errors such as "text.trim is not a function" by checking and converting non-string values before trimming.  
  - *Configuration:* JavaScript code inspects the problematic field, converts it to string if needed, trims it, and logs the correction.  
  - *Inputs:* AI-generated raw or partially processed data.  
  - *Outputs:* Corrected JSON ensuring string fields are properly handled.  
  - *Edge Cases:* Unexpected data types or missing fields may still cause failures; robust error logging is present.  

- **Wait Node**  
  - *Type:* Wait  
  - *Role:* Introduces a 2-second delay to ensure AI processing completes and system stability before downstream processing.  
  - *Configuration:* Fixed 2-second wait.  
  - *Inputs:* Triggered after error handling.  
  - *Outputs:* Pauses workflow flow to next node.  

---

### 2.4 Document Generation and Conversion

**Overview:**  
Converts validated academic assignment JSON data into a professionally formatted HTML document and then into a PDF file suitable for academic submission.

**Nodes Involved:**  
- Long Essay Record Sheet (Google Sheets)  
- Static Html Builder  
- HTTP Request (PDFCrowd API)

**Node Details:**  

- **Long Essay Record Sheet**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends or updates the assignment record in a Google Sheets document for data persistence and record-keeping.  
  - *Configuration:* Maps output JSON fields (Faculty, Department, Name, Matric_No, Level, Course, Date, Questions, Answers, References) to corresponding sheet columns. Uses "Name" as the matching column to update existing records.  
  - *Inputs:* Validated JSON output fields mapped from Edit Fields node.  
  - *Outputs:* Confirms record storage; triggers next node for document generation.  
  - *Edge Cases:* Google Sheets API rate limits or permission errors may cause failures.  

- **Static Html Builder**  
  - *Type:* LangChain Agent  
  - *Role:* Generates a full HTML document using a detailed prompt with strict academic formatting rules, embedded CSS, and structured academic content from the JSON data.  
  - *Configuration:* Uses Google Gemini Chat model with academic styling instructions such as Times New Roman font, 12pt size, double spacing, and structured sections (header, student info, questions, answers, references). Dynamic template variables populate content.  
  - *Inputs:* JSON assignment data from Google Sheets node.  
  - *Outputs:* Complete academic HTML document string, ready for PDF conversion.  
  - *Edge Cases:* Large HTML content may hit API size limits; AI must correctly apply formatting rules.  

- **HTTP Request (PDFCrowd API)**  
  - *Type:* HTTP POST Request  
  - *Role:* Sends the generated HTML content to PDFCrowd API for conversion into a PDF file.  
  - *Configuration:* POSTs form-urlencoded data including HTML text, output filename (student name), and viewport settings for balanced layout. Authorization header uses Basic Auth with API key.  
  - *Inputs:* HTML content from Static Html Builder.  
  - *Outputs:* PDF file binary stream.  
  - *Edge Cases:* API quota limits, network errors, or malformed HTML may cause failures.

---

### 2.5 Storage and Delivery

**Overview:**  
Uploads the generated PDF to Google Drive and sends a Telegram message to the student with the download link.

**Nodes Involved:**  
- Upload file (Google Drive)  
- Send a text message (Telegram)

**Node Details:**  

- **Upload file**  
  - *Type:* Google Drive Node  
  - *Role:* Uploads the PDF file to a predefined Google Drive folder ("Long Essay") for organized file storage.  
  - *Configuration:* Uploads with filename matching the student's name; uses OAuth2 Google Drive credentials. Folder ID is preset to target the correct Drive folder.  
  - *Inputs:* PDF binary file from HTTP Request node.  
  - *Outputs:* Provides Google Drive file metadata including a shareable webViewLink.  
  - *Edge Cases:* Permission issues, API limits, or drive quota may cause failures.  

- **Send a text message**  
  - *Type:* Telegram Node  
  - *Role:* Sends the Google Drive file link to the student's Telegram chat ID, completing the delivery process.  
  - *Configuration:* Uses Telegram API credentials; message text is the Google Drive file webViewLink; chat ID is fixed or dynamic per user.  
  - *Inputs:* Google Drive file metadata link.  
  - *Outputs:* Confirmation of message delivery.  
  - *Edge Cases:* Chat ID mismatch, Telegram API rate limits, or network issues may cause message failures.  

---

## 3. Summary Table

| Node Name                     | Node Type                                    | Functional Role                              | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                      |
|-------------------------------|----------------------------------------------|----------------------------------------------|------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Student Query Intake Bot       | Telegram Trigger                             | Receives Telegram assignment requests        | None                         | Structured Data Parser         | Listens for incoming student messages with assignment details; monitors specific chat ID for authorized users. |
| Structured Data Parser         | Code Node                                   | Parses Telegram message text into JSON       | Student Query Intake Bot      | Student Assignment Auto-Composer | Extracts student and assignment fields using regex; sets current date automatically.                             |
| Student Assignment Auto-Composer| LangChain Agent                            | Generates academic assignment JSON            | Structured Data Parser, Generator Model, Fallback Model Generator, Structured Output Parser | Error Handler                  | Main AI orchestrator node using detailed prompts for plagiarism-free academic content generation.               |
| Generator Model               | LangChain Google Gemini Chat Model          | Primary AI model for content generation       | Student Assignment Auto-Composer | Student Assignment Auto-Composer | Uses Google Gemini API for academic text generation.                                                             |
| Fallback Model Generator      | LangChain Google Gemini Chat Model (Gemma)  | Backup AI model for fallback                   | Student Assignment Auto-Composer | Student Assignment Auto-Composer | Activated if primary model fails or outputs invalid content.                                                    |
| Structured Output Parser      | LangChain Structured Output Parser           | Validates AI output JSON                       | Student Assignment Auto-Composer, Handler For Structured Output Parser | Student Assignment Auto-Composer | Enforces JSON schema compliance; auto-fixes common errors.                                                      |
| Handler For Structured Output Parser | LangChain Google Gemini Chat Model      | Helps fix AI output errors                      | Structured Output Parser      | Structured Output Parser       | Attempts to correct output JSON formatting errors for re-validation.                                           |
| Error Handler                | Code Node                                     | Fixes text processing errors                   | Student Assignment Auto-Composer | Wait                          | Converts non-string values to strings and trims them to prevent errors like "text.trim is not a function".      |
| Wait                        | Wait Node                                     | Introduces delay for processing stability      | Error Handler                | Edit Fields                   | Adds a 2-second delay to allow AI processing to complete.                                                       |
| Edit Fields                 | Set Node                                      | Maps AI output fields to Google Sheets schema | Wait                        | Long Essay Record Sheet        | Prepares data structure for Google Sheets insertion.                                                           |
| Long Essay Record Sheet      | Google Sheets Node                            | Stores assignment records                       | Edit Fields                 | Static Html Builder            | Appends or updates records in Google Sheets; uses "Name" as unique identifier.                                  |
| Static Html Builder          | LangChain Agent                              | Generates formatted academic HTML document     | Long Essay Record Sheet      | HTTP Request                  | Converts JSON assignment data into university-standard HTML with specified styling and formatting rules.        |
| HTTP Request                | HTTP Request Node                            | Converts HTML to PDF via PDFCrowd API          | Static Html Builder          | Upload file                  | Sends HTML content to PDFCrowd API to generate PDF; uses Basic Auth for authorization.                          |
| Upload file                 | Google Drive Node                            | Stores PDF in Google Drive                      | HTTP Request                | Send a text message           | Uploads PDF to specified Drive folder; provides shareable download link.                                        |
| Send a text message         | Telegram Node                                | Sends Google Drive download link to student    | Upload file                 | None                         | Completes delivery by sending PDF download link via Telegram.                                                  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("Student Query Intake Bot")**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: message  
     - Filter by chat ID: Enter authorized user chat ID(s) (e.g., `1026132948`)  
   - Credentials: Telegram Bot API token  
   - Position: Start of workflow  

2. **Add Code Node ("Structured Data Parser")**  
   - Parses Telegram message text with regex to extract fields: Name, Faculty, Department, Level, Course, Reg number, Question.  
   - Auto-sets current date in `YYYY-MM-DD` format.  
   - Output: JSON with cleaned fields.  
   - Connect from Telegram Trigger node.  

3. **Add LangChain Agent Node ("Student Assignment Auto-Composer")**  
   - Type: LangChain Agent  
   - Prompt: Detailed academic assignment generation instructions with placeholders for fields from previous node.  
   - Output: AI-generated assignment JSON string.  
   - Connect from Structured Data Parser node.  
   - Use Google Gemini Chat model credentials.  

4. **Add LangChain Google Gemini Chat Model Node ("Generator Model")**  
   - Primary AI model for generating content.  
   - Connect as ai_languageModel input to "Student Assignment Auto-Composer".  

5. **Add LangChain Google Gemini Chat Model Node ("Fallback Model Generator")**  
   - Secondary AI model for fallback.  
   - Connect as fallback ai_languageModel input to "Student Assignment Auto-Composer".  

6. **Add LangChain Structured Output Parser Node ("Structured Output Parser")**  
   - Configure JSON schema to validate fields: Name, Faculty, Department, Level, Course, Date, Matric_No, Questions, Answers, References.  
   - Enable auto-fix and retry prompts.  
   - Connect ai_outputParser input from "Student Assignment Auto-Composer".  

7. **Add LangChain Google Gemini Chat Model Node ("Handler For Structured Output Parser")**  
   - Fixes errors flagged by output parser.  
   - Connect ai_languageModel input from Structured Output Parser error output.  

8. **Add Code Node ("Error Handler")**  
   - JavaScript code to convert and trim non-string fields (e.g., `text`) to prevent runtime errors.  
   - Connect main output from "Student Assignment Auto-Composer" (or after output parser).  

9. **Add Wait Node ("Wait")**  
   - Set fixed delay: 2 seconds.  
   - Connect from Error Handler node.  

10. **Add Set Node ("Edit Fields")**  
    - Map validated AI JSON fields into output fields matching Google Sheets column names: Faculty, Department, Name, Matric_No, Level, Course, Date, Questions, Answers, References.  
    - Connect from Wait node.  

11. **Add Google Sheets Node ("Long Essay Record Sheet")**  
    - Operation: Append or Update  
    - Document ID: Link to your Google Sheet document  
    - Sheet Name: Use appropriate sheet (e.g., "Sheet1")  
    - Matching column: Name  
    - Map fields as set in Edit Fields node.  
    - Connect from Edit Fields node.  
    - Credentials: Google Sheets OAuth2  

12. **Add LangChain Agent Node ("Static Html Builder")**  
    - Prompt: Generates full academic HTML document with CSS styling based on assignment JSON data.  
    - Connect from Long Essay Record Sheet node.  
    - Use Google Gemini Chat model credentials.  

13. **Add HTTP Request Node ("HTTP Request")**  
    - Method: POST  
    - URL: `https://api.pdfcrowd.com/convert/24.04`  
    - Content-Type: form-urlencoded  
    - Body Parameters:  
      - content_viewport_width: balanced  
      - text: HTML content from Static Html Builder node output  
      - output_name: Student's name  
    - Headers: Authorization Basic with PDFCrowd API key  
    - Response: File output (PDF)  
    - Connect from Static Html Builder node.  

14. **Add Google Drive Node ("Upload file")**  
    - Upload PDF file to specific Google Drive folder (e.g., "Long Essay").  
    - Filename: Student's name from previous node.  
    - Connect from HTTP Request node.  
    - Credentials: Google Drive OAuth2  

15. **Add Telegram Node ("Send a text message")**  
    - Send message containing Google Drive file webViewLink to student's Telegram chat ID.  
    - Chat ID: Use fixed or dynamic chat ID matching student.  
    - Connect from Upload file node.  
    - Credentials: Telegram API token.  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow template automates academic assignment creation from Telegram input to PDF delivery using Google Gemini AI and integrates with Google Drive and Sheets for storage and record keeping. It supports multi-question assignments and enforces strict academic formatting and citation styles.                                                                                           | See Sticky Note1 content in workflow.                                                                           |
| Input format requires structured Telegram messages with fields: Name, Faculty, Department, Level, Course, Reg number, Date (optional), and Questions.                                                                                                                                                                                                                                                                                       | Example input format explained in Sticky Note1.                                                                 |
| Google Gemini API keys and credentials must be properly configured for LangChain nodes to function. Fallback models improve reliability.                                                                                                                                                                                                                                                                                                    | Requires Google Gemini (PaLM) API credentials setup.                                                            |
| PDFCrowd API is used for HTML to PDF conversion; ensure API keys and quota are managed.                                                                                                                                                                                                                                                                                                                                                     | PDFCrowd: https://pdfcrowd.com/                                                                                  |
| Google Drive folder ID and Google Sheets document ID must be replaced with user-specific values during setup.                                                                                                                                                                                                                                                                                                                               | Google Drive and Sheets API credentials and folder/document IDs must be configured.                              |
| The workflow enforces JSON schema validation with auto-fix and retry mechanisms to ensure AI output integrity.                                                                                                                                                                                                                                                                                                                              | Structured Output Parser node configuration is critical for data integrity.                                      |
| The workflow includes error handling for common data issues such as non-string values to maintain flow stability.                                                                                                                                                                                                                                                                                                                            | See Error Handler node's code for details.                                                                       |
| Academic formatting in HTML follows strict university standards (Times New Roman, 12pt, double spacing, justified paragraphs, hierarchical headings, bullet points, APA citations).                                                                                                                                                                                                                                                        | See Static Html Builder prompt for detailed formatting rules and CSS styles.                                     |
| Telegram message delivery uses fixed chat IDs; this can be extended with dynamic user identification for broader deployment.                                                                                                                                                                                                                                                                                                               | Chat ID is currently hardcoded; modify for multi-user support.                                                  |

---

# Disclaimer

The provided text and workflow content derive exclusively from an automated n8n workflow built with ethical and legal compliance. All data handled is public and lawful. No illegal, offensive, or protected content is involved.

---