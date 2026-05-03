Extract Structured Candidate Data from Resumes with GPT AI

https://n8nworkflows.xyz/workflows/extract-structured-candidate-data-from-resumes-with-gpt-ai-5752


# Extract Structured Candidate Data from Resumes with GPT AI

### 1. Workflow Overview

This workflow automates the extraction of structured candidate information from resume files uploaded via a chat interface. It is designed for HR teams, recruiters, or any system requiring automated parsing of resumes in multiple file formats. Upon receiving a resume file, the workflow determines the file type, extracts the raw text content, and then leverages an AI agent powered by GPT-based models to extract structured data fields such as name, email, skills, and education. The extracted structured data is optionally stored in Google Sheets for record-keeping.

**Logical Blocks:**

- **1.1 Input Reception:** Handles incoming chat messages with file uploads.
- **1.2 File Type Detection and Routing:** Determines the file format and routes to the appropriate extraction method.
- **1.3 File Content Extraction:** Extracts raw text content from various supported file formats.
- **1.4 AI Processing:** Sends extracted text to an AI agent to parse structured candidate data.
- **1.5 Output Validation and Storage:** Validates AI output and appends data to Google Sheets or handles parsing errors.
- **1.6 Error Handling:** Provides fallback messages when file parsing or AI extraction fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Waits for incoming chat messages that may contain resume files uploaded by users. Supports file uploads as part of the chat message.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type:* `chatTrigger` (LangChain integration)  
    - *Role:* Entry point; triggers workflow on chat message reception with file upload support enabled.  
    - *Configuration:* File uploads allowed (`allowFileUploads: true`).  
    - *Input:* External chat interface webhook.  
    - *Output:* JSON containing uploaded file metadata and binary data.  
    - *Edge Cases:* Missing file upload, unsupported file types, webhook misconfiguration.  
    - *Version:* 1.1  

---

#### 2.2 File Type Detection and Routing

- **Overview:**  
  Uses the filename extension of the first uploaded file to determine its type and routes the workflow to the corresponding extraction node.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - *Type:* `switch` (base node)  
    - *Role:* Routes execution based on file extension detected in filename (case-sensitive).  
    - *Configuration:* Multiple outputs defined for extensions: `.csv`, `.html`, `.ods`, `.pdf`, `.rtf`, `.txt`, `.xml`, `.xls`. Fallback output named `extra` handles unsupported file types.  
    - *Expressions:* Extracts `fileName` from incoming JSON at `$json.files[0].fileName.toLowerCase()`.  
    - *Input:* Output of "When chat message received".  
    - *Output:* Directs flow to appropriate file extraction node or error node.  
    - *Edge Cases:* Missing or malformed filename, unsupported extensions, case sensitivity issues.  
    - *Version:* 3.2  

---

#### 2.3 File Content Extraction

- **Overview:**  
  Extracts raw text or data content from the uploaded file, using format-specific extraction nodes.

- **Nodes Involved:**  
  - Extract from CSV  
  - Extract from HTML  
  - Extract from ODS  
  - Extract from PDF  
  - Extract from RTF  
  - Extract from TXT  
  - Extract from XML  
  - Extract from XLS  
  - Aggregate

- **Node Details:**

  - **Extract from CSV**  
    - *Type:* `extractFromFile`  
    - *Role:* Parses CSV file from binary.  
    - *Config:* Uses binary property `data0`.  
    - *Output:* Multiple items representing rows; connected to "Aggregate" node to consolidate text.  
    - *Edge Cases:* Malformed CSV, encoding issues, large files.  
    - *Version:* 1  

  - **Aggregate**  
    - *Type:* `aggregate`  
    - *Role:* Aggregates all CSV rows into a single `text` field for AI processing.  
    - *Configuration:* Aggregates all item data into the field `text`.  
    - *Input:* Output of "Extract from CSV".  
    - *Output:* Single JSON with combined text.  
    - *Edge Cases:* Very large CSVs, data truncation.  
    - *Version:* 1  

  - **Extract from HTML**  
    - *Type:* `extractFromFile`  
    - *Role:* Extracts text from HTML files.  
    - *Config:* Operation set to `html`, binary property `data0`.  
    - *Output:* Parsed text content.  
    - *Edge Cases:* Complex HTML, embedded scripts.  
    - *Version:* 1  

  - **Extract from ODS**  
    - *Type:* `extractFromFile`  
    - *Role:* Extracts content from OpenDocument Spreadsheet files.  
    - *Config:* Operation: `ods`, binary property `data0`.  
    - *Output:* Parsed text.  
    - *Edge Cases:* Multi-sheet documents, embedded objects.  
    - *Version:* 1  

  - **Extract from PDF**  
    - *Type:* `extractFromFile`  
    - *Role:* Extracts text from PDF documents.  
    - *Config:* Operation: `pdf`, binary property `data0`.  
    - *Output:* Extracted text.  
    - *Edge Cases:* Scanned PDFs, encrypted or password-protected PDFs.  
    - *Version:* 1  

  - **Extract from RTF**  
    - *Type:* `extractFromFile`  
    - *Role:* Extracts text from RTF files.  
    - *Config:* Operation: `rtf`, binary property `data0`.  
    - *Output:* Plain text.  
    - *Edge Cases:* Complex RTF formatting, embedded objects.  
    - *Version:* 1  

  - **Extract from TXT**  
    - *Type:* `extractFromFile`  
    - *Role:* Reads plain text from TXT files.  
    - *Config:* Operation: `text`, binary property `data0`, output to `text` field.  
    - *Output:* Raw text content.  
    - *Edge Cases:* Encoding issues, very large files.  
    - *Version:* 1  

  - **Extract from XML**  
    - *Type:* `extractFromFile`  
    - *Role:* Extracts text content from XML files.  
    - *Config:* Operation: `xml`, binary property `data0`, text output to `text` field.  
    - *Output:* Parsed XML text.  
    - *Edge Cases:* Complex nested XML, namespaces.  
    - *Version:* 1  

  - **Extract from XLS**  
    - *Type:* `extractFromFile`  
    - *Role:* Extracts data from Excel XLS files.  
    - *Config:* Operation: `xls`, binary property `data0`.  
    - *Output:* Extracted text or data.  
    - *Edge Cases:* Multiple sheets, formulas, macros.  
    - *Version:* 1  

  - For all extraction nodes except CSV, the output connects directly to "Edit Fields" node; CSV extraction output flows through "Aggregate" to "Edit Fields".

---

#### 2.4 AI Processing

- **Overview:**  
  Prepares text for AI input, invokes the AI agent to extract structured resume data, and parses AI response into defined fields.

- **Nodes Involved:**  
  - Edit Fields  
  - AI Agent1  
  - OpenAI Chat Model1  
  - Structured Output Parser

- **Node Details:**

  - **Edit Fields**  
    - *Type:* `set`  
    - *Role:* Formats input JSON to include a `text` property containing the extracted resume text for the AI agent.  
    - *Config:* Sets `text` = `{{$json.text}}`.  
    - *Input:* Output from extraction nodes.  
    - *Output:* JSON with `text` field.  
    - *Version:* 3.4  

  - **AI Agent1**  
    - *Type:* `agent` (LangChain)  
    - *Role:* Sends resume text to an AI agent that extracts email, name, skills, and education fields.  
    - *Config:*  
      - Input text: `{{$json.text}}`  
      - System message: "You are a helpful assistant that extracts email, name, skills, and education from resume text."  
      - Prompt type: define  
      - Output parser enabled (uses Structured Output Parser).  
    - *Input:* Text from "Edit Fields".  
    - *Output:* AI response JSON containing structured data under `output`.  
    - *Edge Cases:* AI response latency, API quota limits, unexpected AI outputs or formatting errors.  
    - *Version:* 1.9  

  - **OpenAI Chat Model1**  
    - *Type:* `lmChatOpenAi`  
    - *Role:* Provides GPT-4o-mini model backend for AI Agent.  
    - *Config:* Model: `gpt-4o-mini`  
    - *Credentials:* Uses OpenAI API key named "Marketing OpenAI".  
    - *Input:* AI Agent1 language model integration.  
    - *Output:* AI-generated chat completions.  
    - *Edge Cases:* Authentication failures, model unavailability.  
    - *Version:* 1.2  

  - **Structured Output Parser**  
    - *Type:* `outputParserStructured`  
    - *Role:* Parses AI output into a JSON schema capturing `name`, `email`, `skills`, and `education`.  
    - *Config:* Example JSON schema supplied to guide parsing.  
    - *Input:* AI Agent1 output parser integration.  
    - *Output:* Structured JSON fields.  
    - *Edge Cases:* Parsing failures due to unexpected AI output format.  
    - *Version:* 1.2  

---

#### 2.5 Output Validation and Storage

- **Overview:**  
  Validates parsed AI output fields, stores structured candidate data in Google Sheets, or handles errors if parsing fails.

- **Nodes Involved:**  
  - Validate Output  
  - Google Sheets (disabled)  
  - Edit Fields1  
  - Edit Fields3

- **Node Details:**

  - **Validate Output**  
    - *Type:* `set`  
    - *Role:* Copies extracted structured fields (`name`, `email`, `skills`, `education`) into `output` namespace for further use.  
    - *Config:* Creates `output.name`, `output.email`, `output.skills`, and `output.education` from AI agent's output.  
    - *On Error:* Continues workflow through error output path.  
    - *Input:* AI Agent1 output.  
    - *Output:* Validated output or error path.  
    - *Edge Cases:* Missing fields, null or empty values from AI.  
    - *Version:* 3.4  

  - **Google Sheets** *(disabled)*  
    - *Type:* `googleSheets`  
    - *Role:* Intended to append or update candidate data in a Google Sheets document.  
    - *Config:* Operation `appendOrUpdate` with placeholder for `sheetName` and `documentId`.  
    - *Credentials:* Uses OAuth2 credentials named "Google Sheets Angel Access".  
    - *Input:* Validated output fields.  
    - *Output:* On success, flows to "Edit Fields3".  
    - *Edge Cases:* Disabled node, missing spreadsheet info, auth errors.  
    - *Version:* 4.6  

  - **Edit Fields1**  
    - *Type:* `set`  
    - *Role:* Sets output message to "Unable to parse the agent outputs" when AI output validation fails.  
    - *Input:* Error output path from "Validate Output".  
    - *Output:* Error message JSON.  
    - *Version:* 3.4  

  - **Edit Fields3**  
    - *Type:* `set`  
    - *Role:* Formats a human-readable summary of the extracted candidate info (name, email, skills, education).  
    - *Config:* Template string combining output fields for display or further use.  
    - *Input:* Success path from Google Sheets (currently disabled).  
    - *Output:* Summary string.  
    - *Version:* 3.4  

---

#### 2.6 Error Handling

- **Overview:**  
  Provides fallback messages when file data extraction fails or unsupported file types are uploaded.

- **Nodes Involved:**  
  - Edit Fields2

- **Node Details:**

  - **Edit Fields2**  
    - *Type:* `set`  
    - *Role:* Returns an error message "Unable to parse the file data" when file type is unsupported or file extraction fails.  
    - *Input:* Fallback output from "Switch" node (`extra` output).  
    - *Output:* Error message JSON for user feedback.  
    - *Version:* 3.4  

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                               | Input Node(s)                    | Output Node(s)                   | Sticky Note                                      |
|---------------------------|--------------------------------|-----------------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------|
| When chat message received | chatTrigger (LangChain)         | Entry point, receives chat messages with files | (Webhook trigger)                | Switch                          |                                                 |
| Switch                    | switch                         | Routes based on file extension                  | When chat message received       | Extract from CSV, HTML, ODS, etc. |                                                 |
| Extract from CSV           | extractFromFile                | Parses CSV file content                         | Switch                         | Aggregate                       |                                                 |
| Aggregate                 | aggregate                      | Aggregates CSV rows into single text            | Extract from CSV                | Edit Fields                    |                                                 |
| Extract from HTML          | extractFromFile                | Extracts text from HTML files                    | Switch                         | Edit Fields                    |                                                 |
| Extract from ODS           | extractFromFile                | Extracts text from ODS files                     | Switch                         | Edit Fields                    |                                                 |
| Extract from PDF           | extractFromFile                | Extracts text from PDF files                     | Switch                         | Edit Fields                    |                                                 |
| Extract from RTF           | extractFromFile                | Extracts text from RTF files                     | Switch                         | Edit Fields                    |                                                 |
| Extract from TXT           | extractFromFile                | Extracts text from TXT files                     | Switch                         | Edit Fields                    |                                                 |
| Extract from XML           | extractFromFile                | Extracts text from XML files                     | Switch                         | Edit Fields                    |                                                 |
| Extract from XLS           | extractFromFile                | Extracts text from XLS files                     | Switch                         | Edit Fields                    |                                                 |
| Edit Fields               | set                            | Prepares text field for AI input                 | Aggregate / Extraction nodes   | AI Agent1                      |                                                 |
| AI Agent1                 | agent (LangChain)              | AI extraction of structured candidate data      | Edit Fields                   | Validate Output                |                                                 |
| OpenAI Chat Model1         | lmChatOpenAi                   | GPT-4o-mini model backend for AI agent          | AI Agent1 (languageModel)       | AI Agent1                      |                                                 |
| Structured Output Parser   | outputParserStructured         | Parses AI output into structured JSON            | AI Agent1 (outputParser)        | AI Agent1                     |                                                 |
| Validate Output           | set                            | Validates and maps AI output fields              | AI Agent1                     | Google Sheets, Edit Fields1     |                                                 |
| Google Sheets             | googleSheets                   | Stores structured data in Google Sheets          | Validate Output                | Edit Fields3                   | Disabled node; placeholder for sheet/document ID |
| Edit Fields1              | set                            | Error message when AI parsing fails              | Validate Output (error output) | -                             |                                                 |
| Edit Fields3              | set                            | Formats final candidate summary message          | Google Sheets                  | -                             |                                                 |
| Edit Fields2              | set                            | Error message for unsupported/failed file parse | Switch (fallback 'extra' output) | -                            |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `When chat message received` node (LangChain chatTrigger).  
   - Set `allowFileUploads` to `true`. This will receive chat messages with file uploads.

2. **Add Switch Node:**  
   - Add `Switch` (base node).  
   - Configure multiple outputs for file extensions `.csv`, `.html`, `.ods`, `.pdf`, `.rtf`, `.txt`, `.xml`, `.xls`.  
   - Use expression `{{$json.files[0].fileName.toLowerCase()}}` and check if it contains the extension string.  
   - Set fallback output to `extra`.

3. **Add File Extraction Nodes For Each Format:**

   - For CSV:  
     - Add `Extract from CSV` node (`extractFromFile` type).  
     - Use binary property `data0`.  
     - Connect from `Switch` CSV output.

   - Add `Aggregate` node to combine CSV rows:  
     - Operation: aggregate all item data into field `text`.  
     - Connect from `Extract from CSV`.

   - For other formats (`html`, `ods`, `pdf`, `rtf`, `txt`, `xml`, `xls`):  
     - Add corresponding `Extract from <format>` nodes of type `extractFromFile`.  
     - Set operation accordingly (e.g., `html`, `ods`, `pdf`, etc.).  
     - Use binary property `data0`.  
     - Connect each from the respective `Switch` output.

4. **Add `Edit Fields` Node:**  
   - Add `Set` node named `Edit Fields`.  
   - Assign field `text` = `{{$json.text}}`.  
   - Connect from `Aggregate` (for CSV) and all other extraction nodes.

5. **Add AI Agent Node:**  
   - Add `AI Agent1` (LangChain agent).  
   - Set the input text to `{{$json.text}}`.  
   - System message: "You are a helpful assistant that extracts email, name, skills, and education from resume text."  
   - Prompt type: `define`.  
   - Enable output parser.

6. **Add OpenAI Chat Model Node:**  
   - Add `OpenAI Chat Model1` node.  
   - Select model `gpt-4o-mini`.  
   - Configure OpenAI credentials (e.g., "Marketing OpenAI" API key).  
   - Connect to `AI Agent1` language model input.

7. **Add Structured Output Parser:**  
   - Add `Structured Output Parser` node.  
   - Provide JSON schema example matching extracted fields:  
     ```json
     {
       "name": "Angel Menendez",
       "email": "angel.menendez@example.com",
       "skills": "JavaScript, Node.js, n8n, Docker, REST APIs",
       "education": "B.S. in Computer Science, University of Arizona, 2010"
     }
     ```  
   - Connect as output parser to `AI Agent1`.

8. **Add Validate Output Node:**  
   - Add `Set` node named `Validate Output`.  
   - Copy AI output fields:  
     - `output.name` = `{{$json.output.name}}`  
     - `output.email` = `{{$json.output.email}}`  
     - `output.skills` = `{{$json.output.skills}}`  
     - `output.education` = `{{$json.output.education}}`  
   - Set error workflow to continue on error (to handle parsing failures).  
   - Connect from `AI Agent1`.

9. **Add Google Sheets Node (Optional):**  
   - Add `Google Sheets` node.  
   - Operation: `appendOrUpdate`.  
   - Provide `sheetName` and `documentId`.  
   - Configure Google Sheets OAuth2 credentials.  
   - Connect from `Validate Output`.  
   - (Note: This node is disabled in the original workflow; enable and configure as needed.)

10. **Add Error Handling Nodes:**  
    - Add `Edit Fields1` (Set node) after `Validate Output` error path.  
    - Set field `output` = `"Unable to parse the agent outputs"`.  
    - Add `Edit Fields2` (Set node) connected from `Switch` fallback output (`extra`).  
    - Set field `output` = `"Unable to parse the file data"`.

11. **Add Final Output Formatting Node:**  
    - Add `Edit Fields3` (Set node) connected from Google Sheets success path.  
    - Set field `output` to a formatted string:  
      ```
      =Name: {{ $json.output.name }}
      Email: {{ $json.output.email }}
      Skills: {{ $json.output.skills }}
      Education: {{ $json.output.education }}
      ```

12. **Connect the Nodes According to the Flow:**  
    - `When chat message received` → `Switch`  
    - `Switch` → respective `Extract from <format>` nodes or `Edit Fields2` (fallback)  
    - `Extract from CSV` → `Aggregate` → `Edit Fields`  
    - Other extract nodes → `Edit Fields`  
    - `Edit Fields` → `AI Agent1`  
    - `AI Agent1` → `Validate Output`  
    - `Validate Output` → `Google Sheets` (success) or `Edit Fields1` (error)  
    - `Google Sheets` → `Edit Fields3`  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow relies on LangChain nodes integrated with n8n for AI interaction using OpenAI models. | [n8n LangChain Integration Documentation](https://docs.n8n.io/integrations/ai/langchain/)                        |
| Supported resume file types: CSV, HTML, ODS, PDF, RTF, TXT, XML, XLS.                              | Ensure files are not password-protected or corrupted for proper extraction.                                     |
| The "Google Sheets" node is disabled by default; configure credentials and spreadsheet details.    | Google Sheets OAuth2 setup: https://developers.google.com/sheets/api/quickstart/python                           |
| AI model used is GPT-4o-mini for cost-effective parsing with reasonable accuracy.                   | Adjust model and parameters based on required accuracy and quota limits.                                        |
| Ensure webhook endpoint for "When chat message received" is properly configured in chat platform.  | Failure to receive chat messages or files will stop the workflow trigger.                                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow designed for legal and public data processing. It complies strictly with content policies and contains no illegal, offensive, or protected materials.