🧾 Automated Invoice Processing with Mistral OCR + GPT-4o-mini

https://n8nworkflows.xyz/workflows/---automated-invoice-processing-with-mistral-ocr---gpt-4o-mini-4331


# 🧾 Automated Invoice Processing with Mistral OCR + GPT-4o-mini

### 1. Workflow Overview

This workflow automates the processing of newly uploaded invoice files in Google Drive by extracting their textual content using Mistral OCR, then structuring the extracted data into a well-defined invoice format leveraging GPT-4o-mini via an AI agent. The logical flow is divided into three main blocks:

- **1.1 Input Reception & File Preparation:** Detect new invoices in Google Drive and download the invoice file, preparing it for OCR processing.
- **1.2 OCR Processing & Text Aggregation:** Convert the downloaded invoice file into Base64, send it to Mistral OCR API to extract text, split the OCR results by page, extract markdown text per page, and recombine all pages into a single aggregated text.
- **1.3 AI-driven Data Structuring:** Use an AI Agent powered by GPT-4o-mini to parse the aggregated text and output a structured JSON representation of the invoice data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Preparation

- **Overview:**  
  This block monitors Google Drive for new invoice files and downloads them for further processing.

- **Nodes Involved:**  
  - Google Drive Trigger: New Invoice Detection  
  - Google Drive: Download Invoice  
  - Convert invoice File to Base64  

- **Node Details:**

  1. **Google Drive Trigger: New Invoice Detection**  
     - *Type & Role:* Trigger node that activates when a new file appears in a specified Google Drive folder.  
     - *Configuration:* Uses Google Drive credentials to monitor a folder (details are implicit). No filters or file type constraints are explicitly configured.  
     - *Connections:* Outputs to "Google Drive: Download Invoice".  
     - *Potential Failures:* Authentication errors with Google Drive OAuth2; missed triggers if folder monitoring is misconfigured.  
     - *Version Notes:* Compatible with n8n Google Drive Trigger v1.

  2. **Google Drive: Download Invoice**  
     - *Type & Role:* Downloads the detected invoice file from Google Drive to be used in workflow.  
     - *Configuration:* Uses credentials linked to Google Drive; no explicit parameters shown, but typically downloads the file from the trigger event.  
     - *Connections:* Outputs to "Convert invoice File to Base64".  
     - *Potential Failures:* File not found, permission errors, download timeouts.  
     - *Version Notes:* Uses Google Drive node v3.

  3. **Convert invoice File to Base64**  
     - *Type & Role:* Converts the binary invoice file into a Base64 encoded string for API transmission.  
     - *Configuration:* Extracts raw file data and encodes it.  
     - *Connections:* Outputs to "Mistral OCR API: Extract Text".  
     - *Potential Failures:* Corrupt or unsupported file formats; large file size may cause performance issues.  
     - *Version Notes:* ExtractFromFile node v1.

---

#### 2.2 OCR Processing & Text Aggregation

- **Overview:**  
  This block sends the invoice file (encoded as Base64) to the Mistral OCR API for text extraction, processes multi-page invoices by splitting and extracting markdown per page, then aggregates all pages into a single text blob.

- **Nodes Involved:**  
  - Mistral OCR API: Extract Text  
  - Data Splitter: OCR Pages  
  - Field Extractor: Page Markdown  
  - Data Aggregator: Combine Pages  

- **Node Details:**

  1. **Mistral OCR API: Extract Text**  
     - *Type & Role:* HTTP Request node calling Mistral OCR API to perform text extraction on the Base64 invoice file.  
     - *Configuration:* Sends Base64 data in the request body, expects JSON response with OCR results.  
     - *Connections:* Outputs to "Data Splitter: OCR Pages".  
     - *Potential Failures:* API authentication or quota errors; network timeouts; malformed requests; OCR inaccuracies.  
     - *Version Notes:* HTTP Request node v4.2.

  2. **Data Splitter: OCR Pages**  
     - *Type & Role:* Splits the OCR JSON response into separate items per page for individual processing.  
     - *Configuration:* Uses splitting parameters targeting the pages array in the OCR output.  
     - *Connections:* Outputs to "Field Extractor: Page Markdown".  
     - *Potential Failures:* Unexpected OCR response format; empty or missing pages array.

  3. **Field Extractor: Page Markdown**  
     - *Type & Role:* Set node that extracts markdown text per page from the OCR output for consistent formatting.  
     - *Configuration:* Extracts and formats the textual content into markdown field for each page.  
     - *Connections:* Outputs to "Data Aggregator: Combine Pages".  
     - *Potential Failures:* Missing or malformed page text fields; expression errors if keys are missing.

  4. **Data Aggregator: Combine Pages**  
     - *Type & Role:* Summarizes and recombines all pages’ markdown text into a single string for AI processing.  
     - *Configuration:* Concatenates page markdown texts into one aggregated document.  
     - *Connections:* Outputs to "AI Agent: Structure Invoice Data".  
     - *Potential Failures:* Data loss if pages are missing; incorrect concatenation order.

---

#### 2.3 AI-driven Data Structuring

- **Overview:**  
  This block uses an AI Agent leveraging GPT-4o-mini to convert the aggregated invoice text into a structured JSON invoice format, followed by parsing the AI output into structured data nodes.

- **Nodes Involved:**  
  - AI Agent: Structure Invoice Data  
  - AI Engine: GPT-4o-mini  
  - JSON Parser: Invoice Structure  

- **Node Details:**

  1. **AI Agent: Structure Invoice Data**  
     - *Type & Role:* LangChain Agent node orchestrating the interaction between the workflow and the GPT-4o-mini language model. It sends the aggregated invoice text and manages response parsing.  
     - *Configuration:* Configured to use GPT-4o-mini as the underlying language model; prompt engineering to instruct the AI to output structured invoice data.  
     - *Connections:*  
       - Input: Receives aggregated invoice text from "Data Aggregator: Combine Pages".  
       - Output: Sends prompt to "AI Engine: GPT-4o-mini" and receives response.  
       - AI output parsed by "JSON Parser: Invoice Structure".  
     - *Potential Failures:* AI API quota or authentication errors; malformed AI responses; prompt interpretation errors.  
     - *Version Notes:* LangChain Agent node v1.9.

  2. **AI Engine: GPT-4o-mini**  
     - *Type & Role:* Language Model node providing GPT-4o-mini model inference for the Agent.  
     - *Configuration:* Uses OpenAI credentials or equivalent; model set to GPT-4o-mini; parameters tuned for structured output.  
     - *Connections:* Input from "AI Agent: Structure Invoice Data"; output back to that agent.  
     - *Potential Failures:* API rate limits; network issues; model unavailability.  
     - *Version Notes:* LangChain LM Chat OpenAI node v1.2.

  3. **JSON Parser: Invoice Structure**  
     - *Type & Role:* Parses the AI-generated raw JSON text output into structured data objects for downstream use.  
     - *Configuration:* Uses LangChain structured output parser to ensure JSON validity and schema adherence.  
     - *Connections:* Input from "AI Agent: Structure Invoice Data".  
     - *Potential Failures:* Invalid JSON output from AI; parser exceptions.  
     - *Version Notes:* LangChain Output Parser Structured node v1.2.

---

### 3. Summary Table

| Node Name                          | Node Type                                     | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note                          |
|-----------------------------------|-----------------------------------------------|----------------------------------------|-----------------------------------|-----------------------------------|------------------------------------|
| Google Drive Trigger: New Invoice Detection | Google Drive Trigger                          | Detect new invoice files in Google Drive | —                                 | Google Drive: Download Invoice     |                                    |
| Google Drive: Download Invoice     | Google Drive                                  | Download invoice file from Google Drive | Google Drive Trigger: New Invoice Detection | Convert invoice File to Base64       |                                    |
| Convert invoice File to Base64     | Extract From File                             | Convert invoice file to Base64 string  | Google Drive: Download Invoice     | Mistral OCR API: Extract Text       |                                    |
| Mistral OCR API: Extract Text      | HTTP Request                                 | Send Base64 invoice to OCR API for text extraction | Convert invoice File to Base64       | Data Splitter: OCR Pages            |                                    |
| Data Splitter: OCR Pages           | Data Splitter                                | Split OCR result into pages             | Mistral OCR API: Extract Text      | Field Extractor: Page Markdown      |                                    |
| Field Extractor: Page Markdown     | Set Node                                     | Extract markdown text from each page   | Data Splitter: OCR Pages           | Data Aggregator: Combine Pages      |                                    |
| Data Aggregator: Combine Pages     | Summarize                                    | Combine page markdown texts into one   | Field Extractor: Page Markdown     | AI Agent: Structure Invoice Data    |                                    |
| AI Agent: Structure Invoice Data   | LangChain Agent                              | Orchestrate AI text-to-structure parsing | Data Aggregator: Combine Pages     | AI Engine: GPT-4o-mini, JSON Parser: Invoice Structure |                                    |
| AI Engine: GPT-4o-mini             | LangChain LM Chat OpenAI                      | Provide GPT-4o-mini model inference    | AI Agent: Structure Invoice Data   | AI Agent: Structure Invoice Data    |                                    |
| JSON Parser: Invoice Structure     | LangChain Output Parser Structured            | Parse AI output JSON into structured data | AI Agent: Structure Invoice Data   | —                                 |                                    |
| Workflow Documentation             | Sticky Note                                  | Documentation placeholder              | —                                 | —                                 |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Google Drive Trigger: New Invoice Detection**  
   - Node Type: Google Drive Trigger  
   - Purpose: Monitor a specific Google Drive folder for new invoice file uploads.  
   - Setup: Authenticate with Google Drive OAuth2 credentials; select folder to watch.  
   - Connect output to next node.

2. **Create Node: Google Drive: Download Invoice**  
   - Node Type: Google Drive  
   - Purpose: Download the new invoice file detected by trigger.  
   - Setup: Use same Google Drive credentials; configure to download file from trigger event data.  
   - Connect input from trigger; output to next node.

3. **Create Node: Convert invoice File to Base64**  
   - Node Type: Extract From File  
   - Purpose: Convert downloaded binary invoice file into Base64 string for API usage.  
   - Configuration: Use default extraction settings to output Base64 encoding.  
   - Connect input from Download Invoice; output to next node.

4. **Create Node: Mistral OCR API: Extract Text**  
   - Node Type: HTTP Request  
   - Purpose: Send the Base64-encoded invoice file to Mistral OCR API for text extraction.  
   - Configuration:  
     - Method: POST  
     - URL: [Mistral OCR API endpoint] (enter your actual endpoint)  
     - Headers: Add required API key/authorization headers  
     - Body: JSON containing Base64 file data  
     - Response: Expect JSON OCR results  
   - Connect input from Convert invoice File to Base64; output to next node.

5. **Create Node: Data Splitter: OCR Pages**  
   - Node Type: SplitOut  
   - Purpose: Split OCR API response into individual pages for per-page processing.  
   - Configuration: Set splitting path to the array of pages in OCR response JSON.  
   - Connect input from Mistral OCR API; output to next node.

6. **Create Node: Field Extractor: Page Markdown**  
   - Node Type: Set  
   - Purpose: Extract the markdown text content from each OCR page for formatting.  
   - Configuration: Use expressions to set a field (e.g., `pageMarkdown`) with the text content from the page.  
   - Connect input from Data Splitter; output to next node.

7. **Create Node: Data Aggregator: Combine Pages**  
   - Node Type: Summarize  
   - Purpose: Concatenate all page markdown text into a single aggregated string.  
   - Configuration: Use concatenation or summarization settings to merge texts.  
   - Connect input from Field Extractor; output to next node.

8. **Create Node: AI Agent: Structure Invoice Data**  
   - Node Type: LangChain Agent  
   - Purpose: Coordinate prompt and response handling for AI model to structure invoice data.  
   - Configuration:  
     - Select GPT-4o-mini as underlying LM.  
     - Setup prompt template to instruct AI to parse raw invoice text into structured JSON.  
   - Connect input from Data Aggregator; outputs to AI Engine and JSON Parser nodes.

9. **Create Node: AI Engine: GPT-4o-mini**  
   - Node Type: LangChain LM Chat OpenAI  
   - Purpose: Provide GPT-4o-mini inference.  
   - Configuration:  
     - Authenticate using OpenAI API or appropriate credentials.  
     - Set model to GPT-4o-mini.  
     - Tune parameters for JSON output if needed.  
   - Connect input from AI Agent; output back to AI Agent.

10. **Create Node: JSON Parser: Invoice Structure**  
    - Node Type: LangChain Output Parser Structured  
    - Purpose: Parse the AI agent’s JSON output into valid structured data.  
    - Configuration: Define expected JSON schema or format for invoices.  
    - Connect input from AI Agent; output to downstream systems or storage as needed.

11. **(Optional) Create Sticky Note: Workflow Documentation**  
    - Node Type: Sticky Note  
    - Purpose: Add documentation or notes about the workflow.  
    - Position near the start for visibility.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                     |
|-------------------------------------------------------------------------------------------------|-----------------------------------|
| The workflow uses Mistral OCR API for document text extraction, which requires an active API key. | Mistral OCR API documentation     |
| GPT-4o-mini is the language model used via LangChain integration; ensure OpenAI credentials are configured properly. | OpenAI API and LangChain docs      |
| Consider rate limits and file size constraints when processing large invoices to avoid API timeouts. | n8n and API rate limit best practices |
| For improved accuracy, prompt engineering within the AI Agent should clearly specify invoice data fields expected in JSON output. | LangChain prompt design guidelines |
| Workflow is designed for PDF or image invoice files stored in Google Drive.                      | Google Drive file format requirements |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.