Generate Document Summaries & Q&As from PDF/TXT using GPT-4o with Slack Alerts

https://n8nworkflows.xyz/workflows/generate-document-summaries---q-as-from-pdf-txt-using-gpt-4o-with-slack-alerts-11736


# Generate Document Summaries & Q&As from PDF/TXT using GPT-4o with Slack Alerts

### 1. Workflow Overview

This workflow automates the process of generating structured document summaries and Q&A pairs from uploaded PDF or TXT files using GPT-4o, with Slack notifications and error logging. It targets use cases where automated document understanding, rapid insight extraction, and structured knowledge delivery are needed, such as internal knowledge bases, customer support documentation, or research summaries.

The workflow is logically divided into the following blocks:

- **1.1 Document Intake & File Type Validation:** Receives documents via webhook, determines file type (PDF or TXT), and routes accordingly.
- **1.2 Text Extraction:** Extracts plain text from the uploaded PDF or TXT files.
- **1.3 AI Summary & Q&A Generation:** Uses an Azure GPT-4o LLM to generate a concise summary and structured Q&A in JSON format.
- **1.4 AI Output Validation & Error Logging:** Validates the AI output format, logs errors to Google Sheets if any, and unwraps the JSON for downstream use.
- **1.5 Final Response Delivery & Slack Preview:** Prepares and sends the final structured JSON response back to the requester via webhook and posts a summary preview to Slack.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Document Intake & File Type Validation

**Overview:**  
This block receives an uploaded document through a webhook, inspects the file extension, and routes the data to appropriate text extraction nodes based on whether the file is a PDF or TXT.

**Nodes Involved:**  
- Receive Document Upload via Webhook  
- Check If Uploaded File Is PDF  
- Check If Uploaded File Is TXT

**Node Details:**

- **Receive Document Upload via Webhook**  
  - Type: Webhook  
  - Role: Entry point capturing HTTP POST requests carrying document files.  
  - Config: HTTP POST on a unique path; responds directly to the caller.  
  - Input: External HTTP POST request with binary file data.  
  - Output: Binary data forwarded downstream.  
  - Edge cases: Missing or malformed file uploads; webhook authorization/configuration errors.

- **Check If Uploaded File Is PDF**  
  - Type: If node  
  - Role: Checks if uploaded file extension equals "pdf" (case-sensitive strict check).  
  - Config: Condition on binary property `$binary.file.fileExtension`.  
  - Input: Webhook output.  
  - Output: True branch routes to PDF text extraction; false branch ignored here.  
  - Edge cases: Files with incorrect extensions, missing extension, or case variations.

- **Check If Uploaded File Is TXT**  
  - Type: If node  
  - Role: Checks if uploaded file extension equals "txt" (case-sensitive strict check).  
  - Config: Condition similar to PDF check.  
  - Input: Webhook output.  
  - Output: True branch routes to TXT text extraction; false branch ignored here.  
  - Edge cases: Same as PDF check.

---

#### 1.2 Text Extraction

**Overview:**  
Extracts readable text content from the uploaded document, converting PDFs via a dedicated extractor and reading raw text from TXT files.

**Nodes Involved:**  
- Extract Text from PDF File  
- Extract Text from TXT File

**Node Details:**

- **Extract Text from PDF File**  
  - Type: ExtractFromFile  
  - Role: Converts PDF binary content into plain text suitable for AI processing.  
  - Config: Operation set to "pdf", input binary property is "file".  
  - Input: Binary PDF file from the PDF check node.  
  - Output: JSON with extracted text under `text` property.  
  - Edge cases: PDFs with scanned images only (no text layer), corrupted PDFs, large file timeout.

- **Extract Text from TXT File**  
  - Type: ExtractFromFile  
  - Role: Reads raw text directly from the TXT binary file.  
  - Config: Operation set to "text", input binary property "file".  
  - Input: Binary TXT file from the TXT check node.  
  - Output: JSON with extracted text under `text` property.  
  - Edge cases: TXT files with unusual encoding, binary disguised as text.

---

#### 1.3 AI Summary & Q&A Generation

**Overview:**  
This block processes extracted text using GPT-4o via Azure OpenAI to generate a structured JSON containing a summary (150–200 words) and 5 Q&A pairs, enforcing strict JSON output formatting.

**Nodes Involved:**  
- Provide LLM Engine for Document Analysis  
- Store AI Memory for Document Analysis Session  
- Generate Summary & Q&A Using AI  
- Parse AI Summary and Q&A into Structured JSON

**Node Details:**

- **Provide LLM Engine for Document Analysis**  
  - Type: LangChain LM Chat Azure OpenAI  
  - Role: Supplies the GPT-4o model configured for chat-based document analysis.  
  - Config: Model set to "gpt-4o", uses Azure OpenAI credentials.  
  - Input: Text prompt and system message from downstream node.  
  - Output: Raw LLM response.  
  - Credentials: Azure OpenAI API account required.  
  - Edge cases: API quota limits, network timeouts, invalid API key.

- **Store AI Memory for Document Analysis Session**  
  - Type: LangChain memory buffer window  
  - Role: Maintains a short-term context memory window (7 entries) keyed by a custom session ID "json_review" to improve AI consistency.  
  - Config: Session key "json_review", context window length 7.  
  - Input: Text prompt and context.  
  - Output: Context-enhanced prompt for AI.  
  - Edge cases: Memory overflow or loss, session key collision.

- **Generate Summary & Q&A Using AI**  
  - Type: LangChain agent node  
  - Role: Core AI node that sends the prompt with document text, requesting a JSON with summary and Q&A.  
  - Config:  
    - Text prompt instructs generating a 150–200 word summary, 5 questions, and answers.  
    - System message enforces accuracy, conciseness, and strict JSON output.  
    - Output parser enabled to enforce JSON parsing.  
  - Input: Extracted text from previous block.  
  - Output: JSON string with summary and Q&A array.  
  - Edge cases: AI hallucination, incomplete JSON output, timeout, prompt injection.

- **Parse AI Summary and Q&A into Structured JSON**  
  - Type: LangChain output parser structured  
  - Role: Parses and validates the LLM output to ensure it conforms to the expected JSON schema (summary + array of Q&A).  
  - Config: JSON schema example provided for validation.  
  - Input: Raw AI output string.  
  - Output: Structured JSON object with summary and Q&A fields.  
  - Edge cases: Parsing failures due to malformed AI output.

---

#### 1.4 AI Output Validation & Error Logging

**Overview:**  
Validates that the AI output is non-empty and properly parsed; logs invalid or empty outputs to a Google Sheets error log; unwraps the JSON object for downstream use.

**Nodes Involved:**  
- Validate AI Output Before Processing  
- Log Invalid AI Output to Google Sheet  
- Unwrap AI Output Object

**Node Details:**

- **Validate AI Output Before Processing**  
  - Type: If node  
  - Role: Checks if the AI output object is non-empty using an object non-empty condition on `$json.output`.  
  - Input: Parsed AI JSON output.  
  - Output: If true, proceeds to unwrap; if false, routes to error logging.  
  - Edge cases: Missing or empty AI output, JSON property naming errors.

- **Log Invalid AI Output to Google Sheet**  
  - Type: Google Sheets node  
  - Role: Appends error details about invalid AI output to a Google Sheet for debugging.  
  - Config: Appends to a specific sheet ("error log sheet") in a defined spreadsheet ("Interviewer Brief Pack").  
  - Credentials: Google Sheets OAuth2 credentials required.  
  - Input: Error context from validation node.  
  - Output: None (logging only).  
  - Edge cases: Google Sheets API quota, permission errors.

- **Unwrap AI Output Object**  
  - Type: Code node  
  - Role: Cleans the AI output by unwrapping arrays to ensure a single JSON object is passed forward.  
  - Code logic: If output is an array, take the first element; else pass as is.  
  - Input: Validated AI JSON output.  
  - Output: Clean JSON object with summary and Q&A.  
  - Edge cases: Unexpected output types, empty arrays.

---

#### 1.5 Final Response Delivery & Slack Preview

**Overview:**  
Prepares a clean JSON response payload and sends it to the original webhook caller; simultaneously posts a short summary preview to Slack for internal team visibility.

**Nodes Involved:**  
- Prepare Final Response Payload  
- Send Final Summary & Q&A Response to Webhook  
- Send Summary Preview to Slack

**Node Details:**

- **Prepare Final Response Payload**  
  - Type: Code node  
  - Role: Ensures the final item is a clean JSON object (unwrapped from any arrays) for n8n to properly respond.  
  - Code logic: Unwraps arrays if present and returns an array wrapping the single object (n8n requirement).  
  - Input: Unwrapped AI JSON object.  
  - Output: Clean JSON ready for webhook response and Slack message.  
  - Edge cases: Unexpected input formats.

- **Send Final Summary & Q&A Response to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends the final structured JSON response back to the original HTTP requestor.  
  - Config: Default options, sends what is passed.  
  - Input: Clean JSON from prepare payload node.  
  - Output: HTTP response with summary + Q&A JSON.  
  - Edge cases: Client disconnect, large payload.

- **Send Summary Preview to Slack**  
  - Type: Slack node  
  - Role: Sends a short preview (first 300 characters of summary) as a direct message to a specified Slack user for quick review.  
  - Config: Text expression extracts summary substring; user specified by Slack user ID.  
  - Credentials: Slack API token with chat permissions.  
  - Input: Final JSON summary object.  
  - Output: Slack message.  
  - Edge cases: Slack API rate limits, invalid user ID.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                               | Input Node(s)                         | Output Node(s)                                      | Sticky Note                                                                                                                        |
|-----------------------------------|----------------------------------|-----------------------------------------------|-------------------------------------|----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Receive Document Upload via Webhook | Webhook                         | Receives uploaded document via HTTP POST      | -                                   | Check If Uploaded File Is PDF, Check If Uploaded File Is TXT | ## 📥 Document Intake & File Type Validation: Receives uploaded docs and routes by file extension.                                |
| Check If Uploaded File Is PDF     | If                               | Checks if file extension is "pdf"              | Receive Document Upload via Webhook | Extract Text from PDF File                           | ## 📥 Document Intake & File Type Validation                                                                                       |
| Check If Uploaded File Is TXT     | If                               | Checks if file extension is "txt"              | Receive Document Upload via Webhook | Extract Text from TXT File                           | ## 📥 Document Intake & File Type Validation                                                                                       |
| Extract Text from PDF File        | ExtractFromFile                  | Extracts plain text from PDF binary            | Check If Uploaded File Is PDF        | Generate Summary & Q&A Using AI                      | ## 📝 File Text Extraction: Converts binary PDF to text.                                                                           |
| Extract Text from TXT File        | ExtractFromFile                  | Extracts text content from TXT binary          | Check If Uploaded File Is TXT        | Generate Summary & Q&A Using AI                      | ## 📝 File Text Extraction: Reads raw text from TXT files.                                                                         |
| Provide LLM Engine for Document Analysis | LangChain LM Chat AzureOpenAI | Provides GPT-4o LLM for summarization          | -                                   | Generate Summary & Q&A Using AI                      | ## 🤖 AI Summary + Q&A Generation: Supplies GPT-4o model.                                                                           |
| Store AI Memory for Document Analysis Session | LangChain memoryBufferWindow    | Maintains short-term AI memory window           | -                                   | Generate Summary & Q&A Using AI                      | ## 🤖 AI Summary + Q&A Generation                                                                                                  |
| Generate Summary & Q&A Using AI   | LangChain agent                  | Generates summary and structured Q&A JSON      | Extract Text from PDF/TXT File       | Validate AI Output Before Processing                 | ## 🤖 AI Summary + Q&A Generation                                                                                                  |
| Parse AI Summary and Q&A into Structured JSON | LangChain outputParserStructured | Parses AI output string into JSON schema         | Generate Summary & Q&A Using AI      | Generate Summary & Q&A Using AI (outputParser link) | ## 🤖 AI Summary + Q&A Generation                                                                                                  |
| Validate AI Output Before Processing | If                              | Checks if AI output is non-empty                 | Generate Summary & Q&A Using AI      | Unwrap AI Output Object (true), Log Invalid AI Output (false) | ## ⚠️ AI Output Validation & Error Logging: Validates AI output presence.                                                        |
| Log Invalid AI Output to Google Sheet | Google Sheets                   | Logs invalid AI output for debugging            | Validate AI Output Before Processing | -                                                  | ## ⚠️ AI Output Validation & Error Logging: Logs malformed AI responses.                                                         |
| Unwrap AI Output Object           | Code                            | Unwraps AI output array to single JSON object  | Validate AI Output Before Processing | Prepare Final Response Payload                       | ## ⚠️ AI Output Validation & Error Logging                                                                                         |
| Prepare Final Response Payload    | Code                            | Cleans final JSON for response                   | Unwrap AI Output Object             | Send Final Summary & Q&A Response to Webhook, Send Summary Preview to Slack | ## 📤 Final Response Delivery & Slack Preview: Prepares clean JSON response.                                                      |
| Send Final Summary & Q&A Response to Webhook | RespondToWebhook                | Sends JSON response back to the HTTP caller     | Prepare Final Response Payload       | -                                                  | ## 📤 Final Response Delivery & Slack Preview                                                                                      |
| Send Summary Preview to Slack     | Slack                           | Sends summary preview snippet via Slack message | Prepare Final Response Payload       | -                                                  | ## 📤 Final Response Delivery & Slack Preview: Sends preview to Slack user for visibility.                                        |
| Sticky Note                      | Sticky Note                     | Describes workflow purpose and high-level steps | -                                   | -                                                  | ## 📄Generate Structured Summary & Q&A from Documents Automatically: Overview and capabilities.                                   |
| Sticky Note1                     | Sticky Note                     | Describes Document Intake & Validation block    | -                                   | -                                                  | ## 📥 Document Intake & File Type Validation                                                                                       |
| Sticky Note2                     | Sticky Note                     | Describes Text Extraction block                  | -                                   | -                                                  | ## 📝 File Text Extraction                                                                                                         |
| Sticky Note3                     | Sticky Note                     | Describes AI Summary + Q&A Generation block     | -                                   | -                                                  | ## 🤖 AI Summary + Q&A Generation                                                                                                  |
| Sticky Note4                     | Sticky Note                     | Describes AI Output Validation & Error Logging  | -                                   | -                                                  | ## ⚠️ AI Output Validation & Error Logging                                                                                         |
| Sticky Note5                     | Sticky Note                     | Describes Final Response Delivery & Slack Preview | -                                   | -                                                  | ## 📤 Final Response Delivery & Slack Preview                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: Receive Document Upload via Webhook  
   - Type: Webhook (HTTP POST)  
   - Path: Unique identifier path (e.g., "65df1798-c390-404d-827c-be1bf6fbe411")  
   - Response Mode: Response Node  
   - Save node.

2. **Add If Node to Check PDF File**  
   - Name: Check If Uploaded File Is PDF  
   - Type: If  
   - Condition: `$binary.file.fileExtension` equals "pdf" (case-sensitive)  
   - Connect input from the webhook node.

3. **Add If Node to Check TXT File**  
   - Name: Check If Uploaded File Is TXT  
   - Type: If  
   - Condition: `$binary.file.fileExtension` equals "txt" (case-sensitive)  
   - Connect input from the webhook node.

4. **Add ExtractFromFile Node for PDF**  
   - Name: Extract Text from PDF File  
   - Type: ExtractFromFile  
   - Operation: pdf  
   - Binary Property Name: file  
   - Connect input from true output of PDF check node.

5. **Add ExtractFromFile Node for TXT**  
   - Name: Extract Text from TXT File  
   - Type: ExtractFromFile  
   - Operation: text  
   - Binary Property Name: file  
   - Connect input from true output of TXT check node.

6. **Add LangChain LM Chat AzureOpenAI Node**  
   - Name: Provide LLM Engine for Document Analysis  
   - Type: LangChain LM Chat AzureOpenAI  
   - Model: gpt-4o  
   - Credentials: Configure with valid Azure OpenAI API account.

7. **Add LangChain Memory Buffer Window Node**  
   - Name: Store AI Memory for Document Analysis Session  
   - Type: LangChain memoryBufferWindow  
   - Session Key: "json_review"  
   - Session ID Type: customKey  
   - Context Window Length: 7  
   - Connect to LLM node’s AI memory input.

8. **Add LangChain Agent Node for AI Processing**  
   - Name: Generate Summary & Q&A Using AI  
   - Type: LangChain agent  
   - Text prompt:  
     ```
     Read the following document text and generate:

     1. A clear summary (150–200 words)
     2. 5 important questions about the document
     3. Answers for each question

     Document Text:
     {{ $json["text"] }}{{ $json.data }}

     Return the output STRICTLY in this JSON format (no additional text):

     {
       "summary": "...",
       "qa": [
         { "q": "Question 1", "a": "Answer 1" },
         { "q": "Question 2", "a": "Answer 2" },
         { "q": "Question 3", "a": "Answer 3" },
         { "q": "Question 4", "a": "Answer 4" },
         { "q": "Question 5", "a": "Answer 5" }
       ]
     }
     ```
   - System message:  
     ```
     You are an expert AI document analyst. 
     Your job is to read long documents and produce a clean summary and structured Q&A.

     Rules:
     - Be accurate.
     - Be concise.
     - Do NOT add any content that is not inside the document.
     - Always return the result in JSON format.
     ```
   - Enable output parser.  
   - Connect extracted text nodes (from PDF and TXT) as input.  
   - Connect LLM engine, output parser, and memory nodes accordingly.

9. **Add LangChain Output Parser Structured Node**  
   - Name: Parse AI Summary and Q&A into Structured JSON  
   - Type: LangChain outputParserStructured  
   - JSON Schema Example:  
     ```
     {
       "summary": "Write a 150–200 word summary here.",
       "qa": [
         {
           "q": "Question text here",
           "a": "Answer text here"
         }
       ]
     }
     ```
   - Connect to AI agent node’s outputParser input.

10. **Add If Node for AI Output Validation**  
    - Name: Validate AI Output Before Processing  
    - Type: If  
    - Condition: `$json.output` is not empty (object non-empty check)  
    - Connect input from AI agent node.

11. **Add Google Sheets Node for Error Logging**  
    - Name: Log Invalid AI Output to Google Sheet  
    - Type: Google Sheets  
    - Operation: Append  
    - Sheet Name: "error log sheet" (sheet ID 1338537721)  
    - Document ID: Spreadsheet "Interviewer Brief Pack"  
    - Columns: error_id, error (mapping manually defined)  
    - Credentials: Google Sheets OAuth2 account  
    - Connect from false output of AI output validation node.

12. **Add Code Node to Unwrap AI Output Object**  
    - Name: Unwrap AI Output Object  
    - Type: Code  
    - JavaScript:  
      ```js
      let out = $json.output;

      if (Array.isArray(out)) {
        out = out[0];
      }

      return out;
      ```
    - Connect from true output of AI output validation node.

13. **Add Code Node to Prepare Final Response Payload**  
    - Name: Prepare Final Response Payload  
    - Type: Code  
    - JavaScript:  
      ```js
      let item = $input.item;

      if (Array.isArray(item)) {
        item = item[0];
      }

      return [
        item
      ];
      ```
    - Connect from Unwrap AI Output Object node.

14. **Add Respond to Webhook Node**  
    - Name: Send Final Summary & Q&A Response to Webhook  
    - Type: RespondToWebhook  
    - Connect from Prepare Final Response Payload node.

15. **Add Slack Node for Summary Preview**  
    - Name: Send Summary Preview to Slack  
    - Type: Slack  
    - Text: `=Short Summary:\n{{ $json.summary.substring(0, 300) }}...`  
    - User: Slack user ID (e.g., "U09HMPVD466")  
    - Credentials: Slack API token with chat permissions  
    - Connect from Prepare Final Response Payload node.

16. **Connect all nodes to reflect the flow:**  
    - Webhook → File Type Checks → Respective Extract Text nodes → AI Processing → AI Output Validation → (Valid) Unwrap → Prepare Response → Respond to Webhook & Slack  
    - (Invalid) AI Output Validation → Log to Google Sheets

17. **Test with sample PDF and TXT files via webhook POST.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow fully automates document understanding and delivers clean, structured, human-readable insights instantly.                                                    | Sticky Note content at workflow start                                                                                                      |
| Slack preview message user must exist and have chat permissions in Slack workspace.                                                                                        | Slack node configuration                                                                                                                    |
| Azure OpenAI credentials must be configured with access to GPT-4o model and appropriate quota.                                                                              | Provide LLM Engine for Document Analysis node                                                                                              |
| Google Sheets API credentials require write permissions to the specified spreadsheet for error logging.                                                                    | Log Invalid AI Output to Google Sheet node                                                                                                 |
| For large documents or scanned PDFs without text layers, consider OCR preprocessing before feeding into this workflow.                                                    | General limitation of Extract Text from PDF node                                                                                            |
| Strict JSON output formatting and parsing prevents corrupt or unexpected AI responses from breaking downstream logic.                                                    | AI Summary & Q&A Generation block                                                                                                          |
| The session memory buffer window is set to 7; adjust based on expected conversation or retry depth to optimize AI context.                                                | Store AI Memory for Document Analysis Session                                                                                              |
| Slack message sends only the first 300 characters of the summary for quick internal reviews to avoid message flooding.                                                    | Send Summary Preview to Slack node                                                                                                         |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow. The content adheres strictly to current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.