Webhook-enabled AI PDF analyzer

https://n8nworkflows.xyz/workflows/webhook-enabled-ai-pdf-analyzer-4336


# Webhook-enabled AI PDF analyzer

### 1. Workflow Overview

This workflow, titled **Webhook-enabled AI PDF analyzer**, is designed to receive a PDF document via a webhook, validate the PDF against size and page count constraints, extract text content from it, analyze and summarize the content using an AI language model (Google Gemini via LangChain), and finally respond with a structured markdown summary. It is targeted at use cases where automated AI-based insight extraction from user-uploaded PDFs is required, such as document summarization, content indexing, or knowledge management.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Receive PDF file from an external source via webhook.
- **1.2 File Reference and Hashing:** Store and optionally hash the file binary for metadata.
- **1.3 File Size and Page Validation:** Extract file size and page count, and validate against limits.
- **1.4 Document Text Extraction:** Extract text content from the PDF.
- **1.5 AI Processing:** Use LangChain’s integration with Google Gemini to parse and summarize the document.
- **1.6 Format and Respond:** Format the AI output into markdown and respond via webhook.
- **1.7 Error Handling:** Return appropriate error responses for validation or processing failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming HTTP POST requests with a PDF file at the endpoint `/ai_pdf_summariser`.
- **Nodes Involved:** `POST /ai_pdf_summariser`, `File Ref`
- **Node Details:**

  - **POST /ai_pdf_summariser**
    - Type: Webhook
    - Role: Entry point for the workflow to accept PDF uploads.
    - Configuration: Listens on path `ai_pdf_summariser` with HTTP POST method, response mode set to respond from a node downstream.
    - Inputs: HTTP request containing PDF file binary.
    - Outputs: Passes binary data to `File Ref`.
    - Edge cases: Handles bot requests (ignoreBots=false ensures bots are not ignored).
  
  - **File Ref**
    - Type: NoOp
    - Role: Acts as a placeholder to store the incoming file binary for downstream referencing.
    - Inputs: From webhook node.
    - Outputs: To `Get File Hash`.
    - Edge cases: None.

#### 2.2 File Reference and Hashing

- **Overview:** Optionally computes a hash of the uploaded file binary for metadata tracking (disabled by default).
- **Nodes Involved:** `Get File Hash` (disabled)
- **Node Details:**

  - **Get File Hash**
    - Type: Crypto
    - Role: Intended to compute hash of the binary data from `File Ref`.
    - Configuration: Uses the binary data of the file as input.
    - Inputs: From `File Ref`.
    - Outputs: To `Get Filesize`.
    - Disabled: This node is currently disabled, indicating hashing is optional.
    - Edge cases: Hashing errors if binary data is corrupted; node disabled here avoids processing.

#### 2.3 File Size and Page Validation

- **Overview:** Extracts the file size in bytes and the number of pages in the PDF, validating them against predefined service limits (max 10MB and 20 pages).
- **Nodes Involved:** `Get Filesize`, `Extract from File`, `Is Valid File?`
- **Node Details:**

  - **Get Filesize**
    - Type: Code
    - Role: Extracts file size from the binary metadata string (e.g., "4.3 MB") and converts it to bytes.
    - Configuration: Custom JavaScript code parses file size and normalizes to bytes.
    - Key expressions: Uses regex to parse size string; reads binary fileSize property.
    - Inputs: From `Get File Hash`.
    - Outputs: To `Extract from File`.
    - Edge cases: Parsing errors if file size string format is invalid; exceptions thrown if unsupported units.
  
  - **Extract from File**
    - Type: ExtractFromFile
    - Role: Extracts PDF text content and metadata such as page count.
    - Configuration: Operation set to "pdf".
    - Inputs: From `Get Filesize`.
    - Outputs: To `Is Valid File?`.
    - Edge cases: Extraction failure on corrupted PDFs.
  
  - **Is Valid File?**
    - Type: IF
    - Role: Checks if the file size ≤ 10MB and number of pages ≤ 20.
    - Conditions: Both conditions must be true to proceed.
    - Inputs: From `Extract from File`.
    - Outputs: True branch to `Information Extractor`, False branch to `400 Error`.
    - Edge cases: Incorrect or missing page count or file size fields could cause condition evaluation errors.

- **Sticky Note:**  
  > **2. Validate PDF File**  
  > We need to add validation here to meet service limits of our LLM. Here, we need to get for filesize and number of pages.

#### 2.4 Document Text Extraction

- **Overview:** Extracts the textual content from the PDF for AI processing.
- **Nodes Involved:** Same as above (`Extract from File`) feeds to next block.
- **Note:** Extraction and page count retrieval done in the same node.

#### 2.5 AI Processing

- **Overview:** Sends the extracted text to a LangChain Information Extractor node configured to use Google Gemini AI for summarization.
- **Nodes Involved:** `Information Extractor`, `OpenAI Chat Model1`
- **Node Details:**

  - **Information Extractor**
    - Type: LangChain Information Extractor
    - Role: Breaks down the document text into distinct topics and generates a 3-paragraph summary for each topic.
    - Configuration:
      - Input text: `=Document as follows:\n{{ $json.text }}`
      - System prompt instructs a note-taking agent to summarize PDF in note form.
      - Output schema: Array of objects, each with `topic` (string) and `insights` (array of `{title, body}` objects).
    - Inputs: From `Is Valid File?` true branch.
    - Outputs: Success to `Format Response`, failure to `500 Error`.
    - Edge cases: AI model errors, API timeouts, input text too large, or schema validation failures.
  
  - **OpenAI Chat Model1**
    - Type: LangChain LM Chat OpenAI
    - Role: Provides the AI language model capability behind the Information Extractor.
    - Configuration: Model set to "gpt-4o-mini".
    - Credentials: Uses OpenAI API credentials.
    - Inputs: AI language model input connection from `Information Extractor`.
    - Outputs: To `Information Extractor`.
    - Edge cases: API quota limits, authentication failures, rate limits.

- **Sticky Note:**  
  > **3. Document Parsing with Gemini API**  
  > Google Gemini is used to breakdown the PDF into topics and provide insights for each. The number of topics and insights are not set and can be variable.

#### 2.6 Format and Respond

- **Overview:** Formats the AI output into a markdown string with topics and insights, then responds to the original HTTP request.
- **Nodes Involved:** `Format Response`, `Success1`
- **Node Details:**

  - **Format Response**
    - Type: Set
    - Role: Transforms the array of topics and insights into markdown format.
    - Configuration: Uses JavaScript expression to map over output items, formatting each topic as a markdown header and insights as bullet points with bold titles.
    - Inputs: From `Information Extractor`.
    - Outputs: To `Success1`.
    - Edge cases: Output format errors if AI output schema is altered or empty.
  
  - **Success1**
    - Type: Respond To Webhook
    - Role: Sends HTTP 200 response with JSON containing:
      - Null error
      - Meta data including file hash (currently from disabled node, may be null)
      - Result: formatted markdown response string.
    - Configuration: Response code 200, content-type application/json.
    - Inputs: From `Format Response`.
    - Outputs: None (ends workflow).
    - Edge cases: Response sending failures, malformed JSON.

- **Sticky Note:**  
  > **5. Format Response**  
  > Finally, we'll format the output to return the summary as markdown.

#### 2.7 Error Handling

- **Overview:** Handles validation failure and unknown error cases by responding with appropriate HTTP error codes and messages.
- **Nodes Involved:** `400 Error`, `500 Error`
- **Node Details:**

  - **400 Error**
    - Type: Respond To Webhook
    - Role: Responds with HTTP 400 Bad Request when file size or page validation fails.
    - Configuration: JSON response with error message about size/page limits.
    - Inputs: From `Is Valid File?` false branch.
  
  - **500 Error**
    - Type: Respond To Webhook
    - Role: Responds with HTTP 500 Internal Server Error on AI processing failure.
    - Configuration: JSON response with generic unknown error message.
    - Inputs: From `Information Extractor` failure branch.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                 | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                               |
|---------------------|----------------------------------|--------------------------------|------------------------|-----------------------|----------------------------------------------------------------------------------------------------------|
| POST /ai_pdf_summariser | Webhook                         | Entry point to receive PDF      | —                      | File Ref              | **POST /ai_pdf_summariser** Call this endpoint to upload a document for summarising.                      |
| File Ref            | NoOp                             | Holds incoming file binary      | POST /ai_pdf_summariser | Get File Hash          |                                                                                                          |
| Get File Hash       | Crypto (disabled)                 | Optional file hash calculation  | File Ref                | Get Filesize           |                                                                                                          |
| Get Filesize        | Code                             | Parses file size to bytes       | Get File Hash            | Extract from File      | **2. Validate PDF File** We need to add validation here to meet service limits of our LLM.                |
| Extract from File   | ExtractFromFile                   | Extract PDF text & metadata     | Get Filesize             | Is Valid File?         |                                                                                                          |
| Is Valid File?      | IF                               | Checks size and page limits     | Extract from File        | Information Extractor (true), 400 Error (false) |                                                                                                          |
| 400 Error           | Respond To Webhook                | Responds with 400 on validation fail | Is Valid File? (false) | —                     |                                                                                                          |
| Information Extractor | LangChain Information Extractor | AI document parsing & summarization | Is Valid File? (true)    | Format Response (success), 500 Error (failure) | **3. Document Parsing with Gemini API** Google Gemini breaks down PDF into topics and insights.           |
| OpenAI Chat Model1  | LangChain LM Chat OpenAI          | Provides AI model for LangChain | — (invoked by Information Extractor) | Information Extractor  |                                                                                                          |
| Format Response     | Set                              | Formats AI output to markdown  | Information Extractor    | Success1               | **5. Format Response** Finally, we'll format the output to return the summary as markdown.                |
| Success1            | Respond To Webhook                | Sends HTTP 200 success response | Format Response          | —                      |                                                                                                          |
| 500 Error           | Respond To Webhook                | Responds with 500 on AI errors  | Information Extractor (failure) | —                      |                                                                                                          |
| Sticky Note1        | Sticky Note                      | Note about validation block    | —                       | —                      | **2. Validate PDF File** We need to add validation here to meet service limits of our LLM.                |
| Sticky Note2        | Sticky Note                      | Note about AI parsing block    | —                       | —                      | **3. Document Parsing with Gemini API** Google Gemini breaks down PDF into topics and insights.           |
| Sticky Note4        | Sticky Note                      | Note about response formatting | —                       | —                      | **5. Format Response** Finally, we'll format the output to return the summary as markdown.                |
| Sticky Note9        | Sticky Note                      | Note about webhook entry point | —                       | —                      | **POST /ai_pdf_summariser** Call this endpoint to upload a document for summarising.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `POST /ai_pdf_summariser`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `ai_pdf_summariser`  
   - Response Mode: Response Node  
   - Options: `ignoreBots` set to false  

2. **Create NoOp Node**  
   - Name: `File Ref`  
   - Type: NoOp  
   - Connect output of webhook to input of this node  

3. **(Optional) Create Crypto Node for File Hashing**  
   - Name: `Get File Hash`  
   - Type: Crypto  
   - Parameters: `value` set to binary data from `File Ref` (expression: `={{ $('File Ref').first().binary }}`)  
   - Disabled by default (optional, enable if hash needed)  
   - Connect output of `File Ref` to `Get File Hash`  

4. **Create Code Node to Extract File Size**  
   - Name: `Get Filesize`  
   - Type: Code  
   - Mode: Run Once For Each Item  
   - Code: Implement JavaScript function to parse fileSize string (e.g. "4.3 MB") to bytes using regex and unit multipliers (b, kb, mb, gb)  
   - Input: Binary data from `Get File Hash` node (or directly from `File Ref` if hash disabled)  
   - Output: JSON with `filesize_in_bytes` and binary data passed along  
   - Connect output of `Get File Hash` or `File Ref` to this node  

5. **Create ExtractFromFile Node**  
   - Name: `Extract from File`  
   - Type: ExtractFromFile  
   - Operation: PDF  
   - Input: Output of `Get Filesize`  
   - Outputs: Extracted text and metadata, including number of pages  

6. **Create IF Node for Validation**  
   - Name: `Is Valid File?`  
   - Type: IF  
   - Conditions (AND):  
     - `filesize_in_bytes` ≤ 10,000,000 (10MB)  
     - `numpages` ≤ 20  
   - Input: From `Extract from File`  
   - True output: Proceed to AI Processing  
   - False output: Connect to 400 Error response node  

7. **Create Respond To Webhook Node for 400 Error**  
   - Name: `400 Error`  
   - Type: Respond To Webhook  
   - Response Code: 400  
   - Response Body: JSON with error message about file size and page count limits  
   - Connect from false output of `Is Valid File?`  

8. **Create LangChain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model1`  
   - Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o-mini`  
   - Credentials: Set OpenAI API credentials (OAuth or API key)  
   - This node is configured to be used internally by the Information Extractor node  

9. **Create LangChain Information Extractor Node**  
   - Name: `Information Extractor`  
   - Type: LangChain Information Extractor  
   - Input Text: Expression `=Document as follows:\n{{ $json.text }}` where `text` is extracted from PDF  
   - System Prompt: "You are a note-taking agent helping the user to summarise this PDF in note form. Breakdown the document into distinct topics and provide a 3 paragraph summary of each topic discussed in the document."  
   - Schema Type: Manual  
   - Input Schema: JSON schema expecting array of objects with `topic` string and `insights` array of `{title, body}` objects  
   - Connect true output of `Is Valid File?` to this node  
   - Connect AI language model input to `OpenAI Chat Model1` node  
   - Success output connects to formatting node  
   - Error output connects to 500 Error node  

10. **Create Respond To Webhook Node for 500 Error**  
    - Name: `500 Error`  
    - Type: Respond To Webhook  
    - Response Code: 500  
    - Response Body: JSON with generic error message about unknown error  
    - Connect error output of `Information Extractor` here  

11. **Create Set Node for Formatting**  
    - Name: `Format Response`  
    - Type: Set  
    - Assignment: Create a single string field `response`  
      - Expression: Map over AI output array, format each topic as markdown header `## {topic}` and each insight as bullet point with bold title:  
      ```js
      $json.output.map(item => {
        return `## ${item.topic}\n${item.insights.map(insight => `- **${insight.title}**: ${insight.body}`).join('\n')}\n`;
      }).join('\n\n');
      ```  
    - Input: From `Information Extractor` success  
    - Output: To final success response node  

12. **Create Respond To Webhook Node for Success**  
    - Name: `Success1`  
    - Type: Respond To Webhook  
    - Response Code: 200  
    - Response Headers: Content-Type application/json  
    - Response Body: JSON object with `error: null`, `meta` containing file hash if available, and `result` with formatted markdown response  
    - Connect output of `Format Response`  

13. **Connect all nodes according to above logic**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                  |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow respects service limits of max 10MB file size and max 20 pages for PDF documents.      | Validation enforced in `Is Valid File?` node                    |
| Uses Google Gemini API (via LangChain) for document summarization and topic extraction.         | This is a powerful AI model designed for document understanding |
| The OpenAI model `gpt-4o-mini` is configured as the language model backend via LangChain.       | OpenAI API credentials are required for this node to work       |
| Recommended to enable file hashing (`Get File Hash`) if tracking of uploads is needed.           | Currently disabled to reduce processing time                     |
| Markdown formatting allows easy integration with downstream markdown renderers or viewers.      | Output is structured for readability                             |
| See n8n documentation for setting up webhook credentials and LangChain integrations.             | https://docs.n8n.io/nodes/                                         |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data are legal and publicly accessible.