Generate Branded Word Documents with Claude AI and Json2Doc (up to 20 Pages)

https://n8nworkflows.xyz/workflows/generate-branded-word-documents-with-claude-ai-and-json2doc--up-to-20-pages--11370


# Generate Branded Word Documents with Claude AI and Json2Doc (up to 20 Pages)

### 1. Workflow Overview

This workflow automates the generation of fully branded Word documents (up to 20 pages) using AI-powered content creation combined with structured document assembly via the Json2Doc MCP tool. It is designed to take a user prompt and a company logo URL from a form, apply consistent styling, generate document sections dynamically through AI, submit the document configuration to an external document generation API, wait for completion, and finally download the completed Word document.

**Target Use Cases:**  
- Automated report generation from user prompts  
- Branded document creation for companies with predefined style guides  
- Large document assembly using AI-generated structured JSON configurations  

**Logical Blocks:**

- **1.1 Input Reception:** Captures user input via a form including document prompt and logo URL.  
- **1.2 Styling Configuration:** Applies company style guide settings (fonts, colors, header/footer).  
- **1.3 AI Document Section Generation:** Uses Anthropic Claude AI and LangChain agent to generate structured JSON for document sections conforming to Json2Doc schema.  
- **1.4 Document Job Submission & Validation:** Submits JSON config to Json2Doc MCP API, validates, and creates a job ID.  
- **1.5 Document Generation Monitoring:** Waits and polls the Json2Doc service until the document generation is completed.  
- **1.6 Document Download:** Downloads the final branded Word document using the job ID.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block collects user inputs for the document generation: a textual prompt describing the desired document and a company logo URL.

**Nodes Involved:**  
- On form submission  
- Sticky Note (Input Form)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Captures user input asynchronously via a webhook form titled "Generate docx" with two fields: "Prompt" (textarea, required) and "Logo URL" (optional URL).  
  - Key expressions: `$('On form submission').item.json.Prompt` to access prompt text, and `Logo URL` used in styling.  
  - Input: External HTTP form POST  
  - Output: JSON object with the prompt and logo URL for downstream processing  
  - Failure cases: Missing required prompt, invalid URLs, webhook errors

- **Sticky Note (Input Form)**  
  - Type: Sticky Note  
  - Role: Documentation comment explaining form inputs  
  - Content: "Takes a Prompt and a logo URL as input"

---

#### 2.2 Styling Configuration

**Overview:**  
Applies a consistent company style guide including fonts, colors, spacing, and header/footer with dynamic logo insertion.

**Nodes Involved:**  
- Add Company styleguide  
- Sticky Note1

**Node Details:**

- **Add Company styleguide**  
  - Type: Set Node  
  - Role: Defines style settings as JSON output for use in document generation:  
    - Font family: Arial  
    - Font size: 11  
    - Color: #333333  
    - Line height: 1.5  
    - Spacing before/after various elements (h1, h2, h3, text, tables, lists, images)  
    - Styles for h1 and h2 headings with font sizes, weights, colors, and alignment  
    - Header: left text ("Example LLC"), right image (company logo URL from input), width 30px, alt text  
    - Footer: center aligned page numbering with dynamic page numbers  
  - Key expressions: Uses `{{ $json['Logo URL'] }}` to dynamically insert logo URL from form input  
  - Input: JSON from form submission node  
  - Output: JSON style configuration object  
  - Failure cases: Missing or invalid logo URL could cause image loading issues in header

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Documentation on styling rationale and references Json2Doc docs link for defaults system  
  - Content includes link: https://docs.json2doc.com/docs/document-generation/overview#defaults-system

---

#### 2.3 AI Document Section Generation

**Overview:**  
Generates the structured JSON configuration representing the document’s sections, validating and creating a job on Json2Doc MCP. Uses Claude AI (Anthropic) and LangChain agent tooling to generate document JSON based on a schema. This is the core AI-driven content generation block.

**Nodes Involved:**  
- Anthropic Chat Model  
- OpenRouter Chat Model  
- Structured Output Parser  
- Json2Doc MCP  
- Generate Sections  
- Sticky Note2

**Node Details:**

- **Anthropic Chat Model**  
  - Type: LangChain Anthropic Chat Model Node  
  - Role: Processes the user prompt with Claude Sonnet 4.5 model, high max tokens (8000) to generate content  
  - Parameters: Claude Sonnet 4.5, thinking mode off, max tokens to sample 8000  
  - Credentials: Anthropic API key  
  - Input: Prompt text from form and style settings  
  - Output: AI-generated text response for document content  
  - Failure cases: API key invalidation, rate limits, timeout, text generation errors

- **OpenRouter Chat Model**  
  - Type: LangChain OpenRouter Chat Model Node  
  - Role: Alternative AI model for output parsing or fallback processing  
  - Credentials: OpenRouter API key  
  - Input/Output: Connected to Structured Output Parser for further processing

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI output into structured JSON adhering to Json2Doc schema, with autoFix enabled to correct minor format issues  
  - Parameters: Uses an example JSON schema with "JobId" field  
  - Input: AI text output  
  - Output: Parsed JSON document config  
  - Failure cases: Parsing errors, invalid JSON schema, incomplete AI output

- **Json2Doc MCP**  
  - Type: LangChain MCP Client Tool  
  - Role: Interfaces with Json2Doc MCP API to validate and submit document configuration as a job  
  - Parameters: Includes tools "Create_a_new_job" and "Get_section_schema", endpoint URL https://mcp.json2doc.com/mcp, header authentication  
  - Credentials: HTTP Header Auth with x-api-key for Json2Doc API  
  - Input: Structured JSON from output parser combined with style guide  
  - Output: Job ID for the created document generation job  
  - Failure cases: Authentication errors, invalid JSON config, network errors

- **Generate Sections**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates the process:  
    1. Obtains schema for sections from Json2Doc tool  
    2. Generates JSON document definition based on schema and user prompt  
    3. Validates JSON config by creating a job, retries if invalid  
    4. Returns Job ID  
  - Parameters: System message instructs to use only Json2Doc MCP tool, no builder mode, JSON config template provided in prompt  
  - Input: Prompt, style guide JSON  
  - Output: Job ID and document config  
  - Failure cases: Schema retrieval failure, invalid JSON generation, repeated validation failures

- **Sticky Note2**  
  - Type: Sticky Note  
  - Content: Detailed instructions about generating Json2Doc config, warning about large context, authentication setup steps with API key, and link to https://app.json2docs.com  

---

#### 2.4 Document Generation Monitoring

**Overview:**  
Waits for the asynchronous document generation to complete by polling the Json2Doc API using the job ID.

**Nodes Involved:**  
- Wait for Document Generation  
- HTTP Request  
- is Completed? (If Node)  
- Sticky Note3

**Node Details:**

- **Wait for Document Generation**  
  - Type: Wait Node  
  - Role: Introduces a delay (3 seconds) to allow document generation before status check  
  - Input: Job ID from Generate Sections  
  - Output: Triggers status check HTTP Request after wait  
  - Failure cases: Timeout if document generation takes longer than expected, webhook issues

- **HTTP Request**  
  - Type: HTTP Request Node  
  - Role: GET request to Json2Doc API endpoint `https://api.json2doc.com/api/v1/jobs/{{ JobId }}` to fetch job status  
  - Authentication: Generic Credential Type with HTTP Header Auth and x-api-key  
  - Input: Job ID from previous node  
  - Output: JSON job status information  
  - Failure cases: Network errors, authentication failures, malformed job ID

- **is Completed?**  
  - Type: If Node  
  - Role: Checks if the job status equals "COMPLETED"  
  - Conditions: `$json.data.status === 'COMPLETED'`  
  - Outputs: If true, proceeds to download; if false, loops back to wait again  
  - Failure cases: Unexpected status values, JSON path errors

- **Sticky Note3**  
  - Type: Sticky Note  
  - Content: Explanation about waiting and downloading, authentication setup instructions repeated from Sticky Note2 for HTTP requests

---

#### 2.5 Document Download

**Overview:**  
Downloads the fully generated Word document file from Json2Doc API once the job is completed.

**Nodes Involved:**  
- Download Docx

**Node Details:**

- **Download Docx**  
  - Type: HTTP Request Node  
  - Role: Downloads the generated file from `https://api.json2doc.com/api/v1/files/{{ outputFileId }}/download` as an octet-stream  
  - Authentication: Same HTTP Header Auth with x-api-key as previous HTTP request nodes  
  - Input: Job output containing `outputFileId` (file identifier)  
  - Output: Binary file stream for the docx document  
  - Failure cases: File not found (invalid outputFileId), authentication errors, network failure

---

### 3. Summary Table

| Node Name                | Node Type                                  | Functional Role                         | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                      |
|--------------------------|--------------------------------------------|---------------------------------------|------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger                              | Receives user prompt and logo URL     | (external HTTP)               | Add Company styleguide          | Takes a Prompt and a logo URL as input                                                                           |
| Add Company styleguide   | Set                                       | Defines company style guide JSON      | On form submission            | Generate Sections              | Consistent Styling with font, colors, spacing, header/footer, see https://docs.json2doc.com/docs/document-generation/overview#defaults-system |
| Anthropic Chat Model     | LangChain Anthropic Chat Model             | AI text generation with Claude AI     | Generate Sections (ai_languageModel) | Generate Sections (ai_languageModel) |                                                                                                                  |
| OpenRouter Chat Model    | LangChain OpenRouter Chat Model            | Alternative AI model                   |                              | Structured Output Parser       |                                                                                                                  |
| Structured Output Parser | LangChain Structured Output Parser         | Parses AI output into structured JSON | OpenRouter Chat Model         | Generate Sections              |                                                                                                                  |
| Json2Doc MCP             | LangChain MCP Client Tool                   | Validates & submits JSON doc config   | Generate Sections (ai_tool)   | Generate Sections              | Generate Json2Doc Config, Authentication with x-api-key at https://app.json2docs.com                              |
| Generate Sections        | LangChain Agent                            | Orchestrates JSON config creation     | Add Company styleguide, Anthropic Chat Model, Structured Output Parser, Json2Doc MCP | Wait for Document Generation     | Warning about large context, validation retries, API key authentication                                         |
| Wait for Document Generation | Wait                                  | Pauses workflow before status check   | Generate Sections            | HTTP Request                  | Waits for document generation completion                                                                        |
| HTTP Request            | HTTP Request                              | Polls Json2Doc job status             | Wait for Document Generation  | is Completed?                 | Authentication setup for Json2Doc API with x-api-key                                                             |
| is Completed?            | If Node                                   | Checks if document generation finished| HTTP Request                 | Download Docx / Wait for Document Generation |                                                                                                                  |
| Download Docx            | HTTP Request                              | Downloads final docx file              | is Completed? (true)          | (ends)                       |                                                                                                                  |
| Sticky Note              | Sticky Note                               | Notes on Input Form                    |                              |                                | Takes a Prompt and a logo URL as input                                                                           |
| Sticky Note1             | Sticky Note                               | Notes on Styling Configuration        |                              |                                | Consistent Styling, see https://docs.json2doc.com/docs/document-generation/overview#defaults-system               |
| Sticky Note2             | Sticky Note                               | Notes on Json2Doc Config Generation   |                              |                                | Generate Json2Doc Config, API Key setup, https://app.json2docs.com                                              |
| Sticky Note3             | Sticky Note                               | Notes on Waiting and Download          |                              |                                | Wait for Document & Download, API Key setup                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Configure form:  
     - Title: "Generate docx"  
     - Fields:  
       - "Prompt" (Textarea, required)  
       - "Logo URL" (Text, optional)  
   - Save, note webhook URL for external usage

2. **Create Set Node for Styling**  
   - Type: Set  
   - Name: "Add Company styleguide"  
   - Mode: Raw JSON  
   - Paste styling JSON:  
     - Font family Arial, font size 11, color #333333, line height 1.5  
     - Spacing before/after for h1,h2,h3,text,table,list,image  
     - Styles for h1 (28pt, bold, #1a365d, center) and h2 (22pt, bold, #2d3748)  
     - Header left text "Example LLC"  
     - Header right image using expression `{{ $json["Logo URL"] }}`, width 30, alt "Company Logo"  
     - Footer center with page numbering dynamic expression  
   - Connect "On form submission" → "Add Company styleguide"

3. **Add LangChain Anthropic Chat Model Node**  
   - Type: Anthropic Chat Model  
   - Name: "Anthropic Chat Model"  
   - Select model: Claude Sonnet 4.5  
   - Max tokens: 8000  
   - Thinking mode: Off  
   - Attach Anthropic API credentials (API key)  
   - Connect to "Generate Sections" node (see step 5)

4. **Add LangChain OpenRouter Chat Model Node**  
   - Type: OpenRouter Chat Model  
   - Name: "OpenRouter Chat Model"  
   - Use default options  
   - Attach OpenRouter API credentials  
   - Connect output to "Structured Output Parser"

5. **Add Structured Output Parser Node**  
   - Type: LangChain Structured Output Parser  
   - Name: "Structured Output Parser"  
   - Enable AutoFix for JSON errors  
   - Provide JSON schema example with "JobId" field  
   - Connect input from "OpenRouter Chat Model"  
   - Connect output to "Generate Sections"

6. **Add LangChain MCP Client Tool Node**  
   - Type: MCP Client Tool  
   - Name: "Json2Doc MCP"  
   - Endpoint URL: https://mcp.json2doc.com/mcp  
   - Include tools: "Create_a_new_job", "Get_section_schema"  
   - Authentication: HTTP Header Auth with header name "x-api-key" and your Json2Doc API key  
   - Connect input from "Generate Sections" (ai_tool input)

7. **Add LangChain Agent Node**  
   - Type: LangChain Agent  
   - Name: "Generate Sections"  
   - Parameters:  
     - Text prompt referencing form prompt: `=The user wants to create a document with the following prompt:\n{{ $('On form submission').item.json.Prompt }}`  
     - System message with instructions to:  
       1. Get schema from MCP tool  
       2. Generate JSON document config with sections  
       3. Validate config by creating job, retry if invalid  
       4. Return Job ID  
   - Max iterations: 10  
   - Connect inputs from "Add Company styleguide", "Anthropic Chat Model", "Structured Output Parser", "Json2Doc MCP"

8. **Add Wait Node**  
   - Type: Wait  
   - Name: "Wait for Document Generation"  
   - Wait time: 3 seconds  
   - Connect input from "Generate Sections" output (Job ID)  
   - Output to "HTTP Request"

9. **Add HTTP Request Node to Poll Job Status**  
   - Type: HTTP Request  
   - Name: "HTTP Request"  
   - Method: GET  
   - URL: `https://api.json2doc.com/api/v1/jobs/{{ $json.output.JobId }}`  
   - Authentication: HTTP Header Auth with same Json2Doc API key  
   - Connect input from "Wait for Document Generation"  
   - Output to "is Completed?"

10. **Add If Node to Check Completion**  
    - Type: If  
    - Name: "is Completed?"  
    - Condition: Check if `{{ $json.data.status }}` equals "COMPLETED" (case-sensitive, strict)  
    - If true: connect to "Download Docx"  
    - If false: connect back to "Wait for Document Generation" (loop polling)

11. **Add HTTP Request Node for File Download**  
    - Type: HTTP Request  
    - Name: "Download Docx"  
    - Method: GET  
    - URL: `https://api.json2doc.com/api/v1/files/{{ $json.data.outputFileId }}/download`  
    - Headers: Accept = application/octet-stream  
    - Authentication: HTTP Header Auth with same API key  
    - Connect input from "is Completed?" (true branch)  
    - Output: Binary docx file ready for download or further action

12. **Add Sticky Notes** (optional but recommended for maintainability)  
    - Input Form explanation near "On form submission"  
    - Styling info near "Add Company styleguide" with link https://docs.json2doc.com/docs/document-generation/overview#defaults-system  
    - Json2Doc config generation notes near "Generate Sections" with API key setup instructions and link https://app.json2docs.com  
    - Document generation wait and download notes near "Wait for Document Generation" and "Download Docx" with repeated API key instructions

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses Claude Sonnet 4.5 via Anthropic for advanced AI text generation, supporting up to 8000 tokens per request.   | Anthropic Claude API                                                                            |
| Json2Doc MCP API requires an API key header authentication (`x-api-key`) which can be obtained at https://app.json2docs.com   | Json2Doc MCP API documentation and dashboard                                                   |
| Styling defaults and document layout are based on Json2Doc's defaults system; see detailed docs for customization options.    | https://docs.json2doc.com/docs/document-generation/overview#defaults-system                     |
| The workflow handles asynchronous document generation with polling and retry logic to ensure the job completes before download| Wait node with polling loop and status check via HTTP Request                                  |
| Document pagination and page numbering are dynamically configured with template expressions within the footer configuration  | Uses Mustache-like expressions for page numbers                                                |

---

*Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.*