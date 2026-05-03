Extract Structured Data from D&B Company Reports with GPT-4o

https://n8nworkflows.xyz/workflows/extract-structured-data-from-d-b-company-reports-with-gpt-4o-8868


# Extract Structured Data from D&B Company Reports with GPT-4o

---

### 1. Workflow Overview

This workflow, titled **Extract Structured Data from D&B Company Reports with GPT-4o**, automates the process of retrieving company data and reports from Dun & Bradstreet (D&B), converting report PDFs to text, and extracting structured business intelligence using advanced AI language models (GPT-4o). The primary use case is to turn complex, often unstructured PDF reports from D&B into clean, flat JSON objects with key company financial and credit information for downstream systems like CRMs, databases, or analytics platforms.

The workflow is logically grouped into the following blocks:

- **1.1 Authentication and Data Retrieval:** Obtains an OAuth2 token via Basic Auth to access D&B APIs, then fetches company reports or data blocks using the DUNS number.
- **1.2 PDF Processing:** Converts the retrieved D&B report PDF into a text-extractable format and extracts raw text content.
- **1.3 AI-Powered Data Extraction:** Uses GPT-4o models to analyze the extracted text and produce a single flat JSON object with structured data fields.
- **1.4 Output Structuring and Validation:** Applies a structured output parser with a predefined JSON schema to ensure data cleanliness and completeness.
- **1.5 Documentation and Setup Guidance:** Includes multiple sticky notes with detailed setup instructions for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Authentication and Data Retrieval

**Overview:**  
This block authenticates with the D&B API by obtaining an OAuth2 bearer token using Basic Authentication and client credentials. It then retrieves a PDF report or JSON data blocks for a specified company identified by its DUNS number.

**Nodes Involved:**  
- `Get Token` (HTTP Request)  
- `D&B Report` (HTTP Request)  
- Sticky Notes: `Sticky Note10`, `Sticky Note65`, `Sticky Note66`

**Node Details:**

- **Get Token**  
  - Type: HTTP Request  
  - Role: Obtains OAuth2 token from D&B API using Basic Auth with username/password.  
  - Config:  
    - Method: POST  
    - URL: `https://plus.dnb.com/v3/token`  
    - Authentication: Basic Auth (D&B credentials)  
    - Body: `grant_type=client_credentials` (form-urlencoded)  
    - Headers: `Content-Type: application/x-www-form-urlencoded`  
  - Inputs: None (triggered manually or upstream trigger)  
  - Outputs: JSON containing `access_token`  
  - Edge cases: Invalid credentials causing 401 Unauthorized; network/timeouts; token expiry (not handled here)  
  - Sticky Note: `Sticky Note66` provides detailed setup steps.

- **D&B Report**  
  - Type: HTTP Request  
  - Role: Fetches a company report PDF from D&B using DUNS number and access token.  
  - Config:  
    - Method: GET  
    - URL Template:  
      `https://plus.dnb.com/v1/reports/duns/804735132?productId=birstd&inLanguage=en-US&reportFormat=PDF&orderReason=6332&tradeUp=hq&customerReference=customer%20reference%20text`  
      *Note:* Hardcoded DUNS number here; can be parameterized for dynamic use.  
    - Authentication: Header Auth with bearer token from `Get Token` node (`Authorization: Bearer {{$json["access_token"]}}`)  
    - Headers: `Accept: application/json`  
  - Inputs: Token from `Get Token`  
  - Outputs: Binary PDF content inside JSON response  
  - Edge cases: Rate limits, 403 Forbidden if token invalid, 404 if DUNS invalid, malformed PDF data.

- **Sticky Notes** (`Sticky Note10`, `Sticky Note65`, `Sticky Note66`)  
  - Role: Provide detailed human-readable setup instructions for configuring the HTTP Request nodes for authentication and data retrieval.  
  - Content includes examples of URL parameters, header setup, and token usage.  
  - Sticky Note10 also contains contact info for support.

---

#### 2.2 PDF Processing

**Overview:**  
This block converts the downloaded PDF report from binary response into a file, then extracts raw text content from the PDF for AI processing.

**Nodes Involved:**  
- `Convert to PDF File` (Convert To File)  
- `Extract Binary` (Extract From File)

**Node Details:**

- **Convert to PDF File**  
  - Type: Convert To File  
  - Role: Converts the binary content of the D&B report into a PDF file stored in workflow memory (`binary` property).  
  - Config:  
    - Operation: toBinary  
    - Source property: `contents[0].contentObject` (from previous HTTP response)  
  - Inputs: Binary data from `D&B Report`  
  - Outputs: PDF file binary data  
  - Edge cases: Source property missing, invalid binary data corrupting PDF.

- **Extract Binary**  
  - Type: Extract From File  
  - Role: Extracts plain text content from the PDF file binary for further text-based processing.  
  - Config:  
    - Operation: pdf (extract text)  
  - Inputs: PDF binary from `Convert to PDF File`  
  - Outputs: Extracted text in JSON format (`text` property)  
  - Edge cases: Complex PDFs with poor OCR; extraction failures; empty output if PDF is image-only without OCR.

---

#### 2.3 AI-Powered Data Extraction

**Overview:**  
This block uses the Langchain GPT-4o model agents to analyze the extracted PDF text and generate a minimal, flat JSON object containing key business report data points.

**Nodes Involved:**  
- `Analyze PDF` (Langchain Agent)  
- `OpenAI Chat Model6` (OpenAI GPT-4o language model node)  
- `OpenAI Chat Model7` (OpenAI GPT-4o language model node)  

**Node Details:**

- **Analyze PDF**  
  - Type: Langchain Agent (AI agent node)  
  - Role: Processes extracted text with GPT-4o to generate a JSON object with required fields.  
  - Config:  
    - Input text: Expression `={{ $json.text }}` (extracted from previous node)  
    - System message prompt: Instructs model to output a single flat JSON with specified fields (e.g., report_date, company_name, duns, ratings, scores).  
    - Rules: No arrays/lists, no prose, null for missing values, date/number formatting.  
    - Has output parser enabled to validate JSON output.  
    - Passthrough binary images enabled (though not used downstream here).  
  - Inputs: Text from `Extract Binary`  
  - Outputs: Raw JSON response candidate  
  - Edge cases: Model hallucination; incomplete extraction; parsing errors; token limits.

- **OpenAI Chat Model6 & OpenAI Chat Model7**  
  - Type: OpenAI GPT-4o nodes (Langchain integration)  
  - Role: Serve as language model backends for the `Analyze PDF` and `Structured Output` nodes respectively.  
  - Config:  
    - Model: GPT-4o  
    - Credentials: OpenAI API key  
  - Inputs/Outputs: Connected as AI language model nodes.  
  - Edge cases: API rate limits; token limits; network errors.

---

#### 2.4 Output Structuring and Validation

**Overview:**  
This block parses and validates the AI-generated JSON output against a defined schema to auto-correct and standardize the extracted data fields.

**Nodes Involved:**  
- `Structured Output` (Langchain Output Parser)  

**Node Details:**

- **Structured Output**  
  - Type: Langchain Output Parser (structured)  
  - Role: Validates and auto-fixes the JSON output from `Analyze PDF` to ensure it matches the expected schema.  
  - Config:  
    - AutoFix: true (attempts to fix errors in output)  
    - JSON Schema Example: Predefined example specifying fields like `report_date`, `company_name`, `duns`, various credit scores, etc.  
  - Inputs: AI JSON output from `Analyze PDF`  
  - Outputs: Cleaned and validated JSON object ready for use or export  
  - Edge cases: Unfixable parsing errors; missing required fields; unexpected data types.

---

### 3. Summary Table

| Node Name          | Node Type                              | Functional Role                          | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                                                                                                                                                              |
|--------------------|--------------------------------------|----------------------------------------|---------------------|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note10      | Sticky Note                          | Setup instructions for D&B API Auth and Data Blocks | None                | None                  | Setup Instructions for fetching company data and setting up D&B Auth HTTP Request node; contact info included                                                                                                                          |
| Sticky Note65      | Sticky Note                          | Instructions specific to fetching company data (Data Blocks) | None                | None                  | Details for configuring HTTP Request node to fetch company data blocks with dynamic DUNS and token usage                                                                                                                               |
| Sticky Note66      | Sticky Note                          | Instructions for setting up D&B Auth HTTP Request node | None                | None                  | Step-by-step for creating HTTP Request node to get OAuth2 token via Basic Auth                                                                                                                                                          |
| Get Token          | HTTP Request                        | Obtain OAuth2 Bearer Token from D&B API | None                | D&B Report            |                                                                                                                                                                                                                                         |
| D&B Report         | HTTP Request                        | Download company report PDF from D&B using token | Get Token           | Convert to PDF File    |                                                                                                                                                                                                                                         |
| Convert to PDF File | Convert To File                     | Convert downloaded content to PDF binary | D&B Report          | Extract Binary        |                                                                                                                                                                                                                                         |
| Extract Binary      | Extract From File                   | Extract text content from PDF binary    | Convert to PDF File  | Analyze PDF           |                                                                                                                                                                                                                                         |
| Analyze PDF        | Langchain Agent                    | Use GPT-4o to extract structured JSON from text | Extract Binary, OpenAI Chat Model6 | Structured Output     |                                                                                                                                                                                                                                         |
| OpenAI Chat Model6  | OpenAI GPT-4o Model Node           | Language model backend for Analyze PDF | None (credentialed)  | Analyze PDF           |                                                                                                                                                                                                                                         |
| Structured Output   | Langchain Output Parser (structured) | Validate and auto-fix AI JSON output    | Analyze PDF, OpenAI Chat Model7 | None                  |                                                                                                                                                                                                                                         |
| OpenAI Chat Model7  | OpenAI GPT-4o Model Node           | Language model backend for Structured Output parser | None (credentialed)  | Structured Output     |                                                                                                                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to build the workflow manually in n8n:

**1. Create HTTP Request Node: `Get Token`**  
- Type: HTTP Request  
- Name: Get Token  
- Method: POST  
- URL: `https://plus.dnb.com/v3/token`  
- Authentication: Basic Auth (enter your D&B username and password credentials)  
- Body Parameters (Form URL-encoded):  
  - `grant_type` = `client_credentials`  
- Headers:  
  - `Content-Type` = `application/x-www-form-urlencoded`  
- Save credentials with your D&B API username/password.

**2. Create HTTP Request Node: `D&B Report`**  
- Type: HTTP Request  
- Name: D&B Report  
- Method: GET  
- URL:  
  ```
  https://plus.dnb.com/v1/reports/duns/804735132?productId=birstd&inLanguage=en-US&reportFormat=PDF&orderReason=6332&tradeUp=hq&customerReference=customer%20reference%20text
  ```  
  *Note:* Replace hardcoded DUNS `804735132` with dynamic expression if needed: `{{ $json.duns }}`  
- Authentication: Header Auth  
- Headers:  
  - `Accept` = `application/json`  
  - `Authorization` = `Bearer {{$json["access_token"]}}` (taken from `Get Token` output)  
- Connect `Get Token` node output to this node input.

**3. Create Convert To File Node: `Convert to PDF File`**  
- Type: Convert To File  
- Name: Convert to PDF File  
- Operation: toBinary  
- Source Property: `contents[0].contentObject` (adjust based on actual HTTP response property containing PDF binary)  
- Connect output of `D&B Report` here.

**4. Create Extract From File Node: `Extract Binary`**  
- Type: Extract From File  
- Name: Extract Binary  
- Operation: pdf (extract text)  
- Connect from `Convert to PDF File`.

**5. Create Langchain Agent Node: `Analyze PDF`**  
- Type: Langchain Agent  
- Name: Analyze PDF  
- Text input: Use expression `={{ $json.text }}` to get extracted text from previous node  
- System Message Prompt:  
  ```
  You are a precision extractor. Read the provided business report PDF and return only a single flat JSON object with the fields below. Keep it minimal and focused on overall scores.

  No arrays/lists.

  No prose.

  If a value is missing, output null.

  Dates must be YYYY-MM-DD.

  Numbers must be plain numerics (no commas or $).

  Output Format

  Return only in JSON object:


  Rules

  Prefer the most recent or highest-level “overall” values if multiple are shown.

  Never include arrays, nested structures, or text outside of the JSON object.
  ```  
- Enable output parser  
- Passthrough binary images enabled (optional)  
- Connect input from `Extract Binary` and set AI language model node (see next step).

**6. Create OpenAI GPT-4o Node: `OpenAI Chat Model6`**  
- Type: OpenAI Chat Model (Langchain integration)  
- Name: OpenAI Chat Model6  
- Model: GPT-4o  
- Credentials: Your OpenAI API key configured in n8n  
- Connect this node as the AI language model for `Analyze PDF`.

**7. Create Langchain Output Parser Node: `Structured Output`**  
- Type: Output Parser (Structured)  
- Name: Structured Output  
- Enable AutoFix: true  
- JSON Schema Example (paste the following example):  
  ```json
  {
    "report_date": "",
    "company_name": "",
    "duns": "",
    "dnb_rating_overall": "",
    "composite_credit_appraisal": "",
    "viability_score": "",
    "portfolio_comparison_score": "",
    "paydex_3mo": "",
    "paydex_24mo": "",
    "credit_limit_conservative": ""
  }
  ```  
- Connect input from `Analyze PDF` output  
- Assign AI language model node (see next step).

**8. Create OpenAI GPT-4o Node: `OpenAI Chat Model7`**  
- Type: OpenAI Chat Model (Langchain integration)  
- Name: OpenAI Chat Model7  
- Model: GPT-4o  
- Credentials: Your OpenAI API key  
- Connect as AI language model for `Structured Output`.

**9. Connect all nodes in order:**  
`Get Token` → `D&B Report` → `Convert to PDF File` → `Extract Binary` → `Analyze PDF` → `Structured Output`

**10. (Optional) Add Sticky Notes for Documentation**  
- Add sticky notes with setup instructions, referencing headers, URLs, tokens as in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Setup instructions include detailed HTTP Request configuration for D&B token acquisition and data block retrieval with dynamic tokens. | See Sticky Notes `Sticky Note10`, `Sticky Note65`, and `Sticky Note66` in the workflow.           |
| Contact for customization help: Robert Breen, email: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Provided in Sticky Note10 for user support and consulting.                                        |
| The workflow uses Langchain’s n8n nodes integration for GPT-4o, requiring valid OpenAI API credentials configured in n8n.          | Ensure OpenAI API keys are properly set up under credentials for AI nodes `OpenAI Chat Model6/7`. |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---