Unstructured Resume Parser with Thordata Universal API + OpenAI GPT-4.1-mini

https://n8nworkflows.xyz/workflows/unstructured-resume-parser-with-thordata-universal-api---openai-gpt-4-1-mini-10289


# Unstructured Resume Parser with Thordata Universal API + OpenAI GPT-4.1-mini

### 1. Workflow Overview

This workflow automates the parsing of unstructured resumes available via a web URL into a structured JSON Resume format compliant with the [JSON Resume Schema](https://jsonresume.org/schema/). It is designed for HR, recruitment, and talent management use cases requiring automated extraction of candidate information from diverse resume formats.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**  
  Captures the input URL of a candidate’s resume hosted online and prepares it for processing.

- **1.2 Document Processing and Content Extraction**  
  Sends the resume URL to the Thordata Universal API for web content scraping, cleaning, and format normalization, yielding HTML content.

- **1.3 Content Transformation and AI-Powered Parsing**  
  Converts the extracted HTML into Markdown, then uses the OpenAI GPT-4.1-mini language model to parse the Markdown text into a structured JSON Resume format using a detailed JSON schema.

- **1.4 Data Output and Storage**  
  Converts the parsed JSON into a binary format for file writing, saves the structured JSON resume to disk, and appends or updates a Google Sheets document for visualization and further processing.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception and Initialization

**Overview:**  
This block initiates the workflow via manual trigger and sets the input parameter, which is the URL of the resume to be parsed.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Set the Input Fields

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual execution by the user.  
  - Configuration: No parameters; simply triggers the next node.  
  - Inputs: None (trigger only)  
  - Outputs: One output to "Set the Input Fields" node.  
  - Edge Cases: None, but workflow depends on manual execution.  

- **Set the Input Fields**  
  - Type: Set  
  - Role: Defines input data; specifically sets the URL of the resume to be parsed.  
  - Configuration: Sets a string variable `resume_url` to `"https://registry.jsonresume.org/thomasdavis?theme=elegant"`. This URL can be customized to any accessible resume URL.  
  - Inputs: From manual trigger  
  - Outputs: Passes JSON with `resume_url` to the next node.  
  - Edge Cases: If the URL is invalid or unreachable, downstream nodes may fail or produce empty results.

---

#### Block 1.2: Document Processing and Content Extraction

**Overview:**  
This block sends the resume URL to Thordata Universal API to scrape and normalize the resume content into HTML, with cleaning options to remove CSS and JavaScript.

**Nodes Involved:**  
- Perform Thordata Universal API Request  
- Convert HTML to Markdown format

**Node Details:**

- **Perform Thordata Universal API Request**  
  - Type: HTTP Request  
  - Role: Calls Thordata Universal API endpoint to extract document content from the provided URL.  
  - Configuration:  
    - Method: POST  
    - URL: `https://universalapi.thordata.com/request`  
    - Content-Type: `application/x-www-form-urlencoded`  
    - Body Parameters:  
      - `url`: set dynamically from `resume_url` input (`={{ $json.resume_url }}`)  
      - `type`: `"html"` (requesting HTML output)  
      - `js_render`: `"False"` (disables JavaScript rendering)  
      - `country`: `"in"` (indicates country context; India in this case)  
      - `clean_content`: `"css,js"` (removes CSS and JavaScript from the content)  
    - Authentication: HTTP Bearer token using stored credential `Thordata Universal API Token`  
  - Inputs: Receives JSON with `resume_url`  
  - Outputs: JSON response containing `html` content of the resume webpage.  
  - Edge Cases:  
    - Authentication failure if token invalid or expired  
    - URL unreachable or returns error from API  
    - Returned HTML may be empty or malformed if page structure unexpected  

- **Convert HTML to Markdown format**  
  - Type: Markdown  
  - Role: Converts the HTML content from Thordata API into Markdown format to simplify text processing and parsing.  
  - Configuration:  
    - Input HTML: taken from `html` field of previous node output (`={{ $json.html }}`)  
    - Output field: `markdown` (stores Markdown content)  
  - Inputs: HTML content JSON  
  - Outputs: JSON with new `markdown` field containing Markdown text  
  - Edge Cases: HTML content may be empty or malformed, resulting in empty or incomplete Markdown.

---

#### Block 1.3: Content Transformation and AI-Powered Parsing

**Overview:**  
Transforms the Markdown resume content into a structured JSON Resume object by invoking OpenAI GPT-4.1-mini with a detailed JSON schema prompt.

**Nodes Involved:**  
- JSON Resume Builder  
- OpenAI Chat Model for Resume Builder

**Node Details:**

- **JSON Resume Builder**  
  - Type: Information Extractor (LangChain node)  
  - Role: Uses a JSON schema prompt to direct the AI model to parse resume content into JSON Resume schema.  
  - Configuration:  
    - Input text: `"Analyze and Parse the provided resume in JSON Resume format.\n\n {{ $json.markdown }}"` — injects Markdown resume text from previous node.  
    - Schema Type: Manual  
    - Input Schema: Detailed JSON schema covering all fields defined by JSON Resume standard, including basics, work experience, education, skills, projects, certifications, languages, interests, and meta information.  
  - Inputs: Receives `markdown` text and AI model output from OpenAI Chat Model node.  
  - Outputs: JSON structured resume under `output` key.  
  - Edge Cases:  
    - AI parsing errors or incomplete parsing if resume content ambiguous  
    - Schema validation failures if AI output does not fully conform  
    - Large resumes might result in API token limits or timeouts  

- **OpenAI Chat Model for Resume Builder**  
  - Type: AI Language Model (OpenAI GPT)  
  - Role: Provides the GPT-4.1-mini model to process the input prompt and generate structured data.  
  - Configuration:  
    - Model: `gpt-4.1-mini` (lightweight GPT-4 variant)  
    - Credentials: OpenAI API key stored in `OpenAi account` credential  
  - Inputs: Receives prompt text from previous node, returns AI-generated content to JSON Resume Builder node.  
  - Outputs: AI-generated JSON string representing the parsed resume fields.  
  - Edge Cases:  
    - API rate limits or quota exhaustion  
    - Network timeouts or failures connecting to OpenAI  
    - Model hallucination or inaccurate parsing if prompt ambiguous

---

#### Block 1.4: Data Output and Storage

**Overview:**  
This block formats the parsed JSON resume for storage, saves it as a JSON file on disk, and appends or updates the resume data in a Google Sheet for visualization or downstream use.

**Nodes Involved:**  
- Create a Binary Response  
- Write the Structured JSON resume to Disk  
- Append or update row in sheet

**Node Details:**

- **Create a Binary Response**  
  - Type: Function  
  - Role: Converts the JSON resume data into a base64-encoded binary buffer for file writing.  
  - Configuration:  
    - JavaScript code creates a binary `data` property containing the stringified JSON resume encoded in base64.  
  - Inputs: JSON resume output from JSON Resume Builder node.  
  - Outputs: Item with binary data ready for file writing.  
  - Edge Cases: Malformed JSON could cause encoding errors.  

- **Write the Structured JSON resume to Disk**  
  - Type: Read/Write File  
  - Role: Saves the binary-encoded JSON resume to a local disk file.  
  - Configuration:  
    - Operation: `write`  
    - Filename: Dynamic, constructed as `C:\{{ $json.output.basics.name }}.json` to save file named after candidate.  
  - Inputs: Binary data from previous function node.  
  - Outputs: Confirmation of file write.  
  - Edge Cases:  
    - File write permissions or disk full errors  
    - Invalid characters in filename (from candidate name) causing file system errors  

- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates a row in a Google Sheets document with the JSON resume string for record-keeping and visualization.  
  - Configuration:  
    - Operation: `appendOrUpdate`  
    - Document ID: hardcoded to a specific Google Sheets document `1CCj7PRT93hmTfkrDXSw6vwmz5UGZED4vA5l7LCNIAhk`  
    - Sheet name: `gid=0` (first sheet)  
    - Column mapping: Single column named `json_resume` set to the JSON resume stringified via expression `={{ $json.output.toJsonString() }}`  
  - Credentials: OAuth2 for Google Sheets access (`Google Sheets account`)  
  - Inputs: JSON resume output from JSON Resume Builder node  
  - Outputs: Confirmation of append or update operation  
  - Edge Cases:  
    - Authentication failure with Google OAuth2  
    - Sheet access or quota limits  
    - Large JSON strings exceeding cell size limits

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                         | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                                                 |
|-----------------------------------|----------------------------------|---------------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                   | Workflow start trigger                 | None                            | Set the Input Fields              | ![Logo](https://consumersiteimages.trustpilot.net/business-units/67b212598525b99cf90a59cc-198x149-1x.jpg)                 |
| Set the Input Fields              | Set                              | Sets the resume URL input              | When clicking ‘Execute workflow’ | Perform Thordata Universal API Request | ## Purpose  This workflow automates the process of parsing unstructured resume data into structured JSON objects compatible with the [JSON Resume Schema](https://jsonresume.org/schema/). It leverages **Thordata Universal API** for intelligent document ingestion and **OpenAI GPT-4.1-mini** for contextual field extraction, ensuring high-quality structured resume output for ATS, CRM, or recruitment analytics systems. |
| Perform Thordata Universal API Request | HTTP Request                   | Scrapes and normalizes resume content | Set the Input Fields            | Convert HTML to Markdown format   | ## Thordata Universal API  Thordata Universal API is being used for web scraping the URL content                            |
| Convert HTML to Markdown format   | Markdown                        | Converts HTML resume content to markdown | Perform Thordata Universal API Request | JSON Resume Builder              |                                                                                                                             |
| OpenAI Chat Model for Resume Builder | AI Language Model (OpenAI GPT) | Provides AI model for resume parsing  | (Implicit in chain with JSON Resume Builder) | JSON Resume Builder              | ## Structured Resume Parser  Uses the OpenAI gtp-4.1.mini for parsing the resume content in Markdown format to JSON resume  |
| JSON Resume Builder               | Information Extractor (LangChain) | Parses Markdown text to JSON Resume   | Convert HTML to Markdown format, OpenAI Chat Model for Resume Builder | Create a Binary Response, Append or update row in sheet |                                                                                                                             |
| Create a Binary Response          | Function                       | Encodes JSON resume as base64 binary  | JSON Resume Builder             | Write the Structured JSON resume to Disk | ## Export Data Handling  Export the JSON Resume to Disk and Google Speadsheet                                               |
| Write the Structured JSON resume to Disk | Read/Write File              | Saves JSON resume file on disk         | Create a Binary Response        | None                            |                                                                                                                             |
| Append or update row in sheet     | Google Sheets                  | Saves JSON resume to Google Sheets     | JSON Resume Builder             | None                            |                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - Default settings; no parameters.

2. **Create Set Node to Define Input URL**  
   - Name: `Set the Input Fields`  
   - Type: Set  
   - Add field assignment:  
     - Field Name: `resume_url` (string)  
     - Value: `"https://registry.jsonresume.org/thomasdavis?theme=elegant"` (or any accessible resume URL)  
   - Connect trigger output to this node.

3. **Create HTTP Request Node for Thordata API**  
   - Name: `Perform Thordata Universal API Request`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://universalapi.thordata.com/request`  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Authentication: HTTP Bearer Auth using credential "Thordata Universal API Token" (set up with your token)  
   - Body Parameters (form-urlencoded):  
     - `url`: `={{ $json.resume_url }}`  
     - `type`: `"html"`  
     - `js_render`: `"False"`  
     - `country`: `"in"`  
     - `clean_content`: `"css,js"`  
   - Connect Set Node output to this node.

4. **Create Markdown Node to convert HTML to Markdown**  
   - Name: `Convert HTML to Markdown format`  
   - Type: Markdown  
   - HTML input: `={{ $json.html }}` (from HTTP Request output)  
   - Destination key: `markdown`  
   - Connect HTTP Request node output to this node.

5. **Create OpenAI Chat Model Node for GPT-4.1-mini**  
   - Name: `OpenAI Chat Model for Resume Builder`  
   - Type: AI Language Model (OpenAI)  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API key (set up OAuth/Key in credentials)  
   - This node is linked as AI input for the next node (JSON Resume Builder).

6. **Create Information Extractor Node (LangChain)**  
   - Name: `JSON Resume Builder`  
   - Type: Information Extractor  
   - Input text: `"Analyze and Parse the provided resume in JSON Resume format.\n\n {{ $json.markdown }}"`  
   - Schema Type: Manual  
   - Input Schema: Use the detailed JSON Resume schema covering basics, work, education, skills, projects, certifications, languages, interests, and meta fields.  
   - Connect Markdown node output and OpenAI Chat Model node output as inputs (languageModel input).  

7. **Create Function Node to Generate Binary Data**  
   - Name: `Create a Binary Response`  
   - Type: Function  
   - Code:  
     ```javascript
     items[0].binary = {
       data: {
         data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
       }
     };
     return items;
     ```  
   - Connect output of JSON Resume Builder to this node.

8. **Create Read/Write File Node to Save JSON Resume to Disk**  
   - Name: `Write the Structured JSON resume to Disk`  
   - Type: Read/Write File  
   - Operation: Write  
   - File Name: `C:\\{{ $json.output.basics.name }}.json` (dynamic filename based on candidate name)  
   - Input: binary data from Function node  
   - Connect Function node output to this node.

9. **Create Google Sheets Node to Append or Update Row**  
   - Name: `Append or update row in sheet`  
   - Type: Google Sheets  
   - Operation: appendOrUpdate  
   - Document ID: Use your Google Sheet ID (e.g., `1CCj7PRT93hmTfkrDXSw6vwmz5UGZED4vA5l7LCNIAhk`)  
   - Sheet Name: `gid=0` or your target sheet name  
   - Column Mapping: Single column named `json_resume` set to `={{ $json.output.toJsonString() }}`  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect JSON Resume Builder output to this node.

10. **Connect Workflow Nodes in Order:**  
    - When clicking ‘Execute workflow’ → Set the Input Fields → Perform Thordata Universal API Request → Convert HTML to Markdown format → JSON Resume Builder  
    - OpenAI Chat Model for Resume Builder → JSON Resume Builder (as AI model input)  
    - JSON Resume Builder → Create a Binary Response → Write the Structured JSON resume to Disk  
    - JSON Resume Builder → Append or update row in sheet  

11. **Credentials Setup:**  
    - Thordata Universal API Token: Setup HTTP Bearer Auth with valid token.  
    - OpenAI API: Setup with valid OpenAI API key for GPT-4.1-mini access.  
    - Google Sheets OAuth2: Setup OAuth2 credentials with write access to target Google Sheet.

12. **Validation:**  
    - Test workflow by running manual trigger, verify resume URL is reachable.  
    - Check that JSON resume file is created on disk with candidate’s name.  
    - Confirm entry is appended or updated in Google Sheets.  
    - Review logs for any API or parsing errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates the conversion of unstructured resume data from web-hosted resumes into structured JSON Resume format for ATS and recruitment analytics integration.                                                       | Workflow summary and purpose description sticky note                                          |
| The workflow uses Thordata Universal API for document ingestion, which supports cleaning and normalization of web content including removal of CSS and JavaScript to isolate resume text.                                         | Thordata Universal API sticky note                                                            |
| OpenAI GPT-4.1-mini is utilized for accurate semantic parsing of Markdown resume content into JSON Resume schema, enabling extraction of detailed fields such as work history, education, skills, and certifications.               | Structured Resume Parser sticky note                                                          |
| Exported JSON resumes are saved locally and appended to a Google Sheet for record-keeping and easy visualization in spreadsheets.                                                                                               | Export Data Handling sticky note                                                              |
| JSON Resume Schema reference used for defining the structured output: https://jsonresume.org/schema/                                                                                                                             | External schema reference                                                                       |
| Example resume URL used for testing: https://registry.jsonresume.org/thomasdavis?theme=elegant                                                                                                                                  | Input example URL                                                                              |
| Logo image embedded in sticky note for branding: ![Logo](https://consumersiteimages.trustpilot.net/business-units/67b212598525b99cf90a59cc-198x149-1x.jpg)                                                                    | Branding                                                                                      |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.