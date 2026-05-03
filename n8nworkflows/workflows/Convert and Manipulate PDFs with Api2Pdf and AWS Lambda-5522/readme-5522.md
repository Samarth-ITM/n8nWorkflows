Convert and Manipulate PDFs with Api2Pdf and AWS Lambda

https://n8nworkflows.xyz/workflows/convert-and-manipulate-pdfs-with-api2pdf-and-aws-lambda-5522


# Convert and Manipulate PDFs with Api2Pdf and AWS Lambda

### 1. Workflow Overview

This workflow provides an interface to the Api2Pdf service, a robust PDF generation and manipulation API powered by AWS Lambda. It is designed as an MCP (Multi-Channel Pipeline) server that AI agents can interact with via a single webhook endpoint. The workflow exposes nine distinct API operations for PDF conversion and barcode generation, simplifying integration with AI workflows for document processing tasks.

Logical blocks included:

- **1.1 Input Reception & MCP Trigger:** The entry point for AI agents via an MCP trigger node.
- **1.2 Headless Chrome PDF Conversion:** Operations converting HTML or URLs to PDF using Headless Chrome.
- **1.3 LibreOffice PDF Conversion:** Converts office documents or images to PDFs.
- **1.4 PDF Merge:** Merges multiple PDFs into one file.
- **1.5 WKHtmlToPdf Conversion:** Converts HTML or URLs to PDF using wkhtmltopdf engine.
- **1.6 ZXing Barcode Generation:** Generates barcodes and QR codes using the ZXing library.
- **1.7 Setup and Documentation Notes:** Sticky notes providing setup instructions, usage notes, and an overview of the API capabilities.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & MCP Trigger

**Overview:**  
This block serves as the single entry point for AI agents, exposing a webhook URL that triggers the workflow. It handles incoming requests and routes them to the appropriate API endpoint nodes.

**Nodes Involved:**  
- Api2Pdf - PDF Generation, Powered by AWS Lambda MCP Server

**Node Details:**

- **Name:** Api2Pdf - PDF Generation, Powered by AWS Lambda MCP Server  
- **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
- **Configuration:**  
  - Webhook path: `api2pdf---pdf-generation,-powered-by-aws-lambda-mcp`  
  - Acts as an MCP trigger, receiving calls from AI agents and distributing them to connected HTTP request nodes.  
- **Inputs:** None (trigger node)  
- **Outputs:** Connects to all HTTP Request Tool nodes  
- **Version-specific requirements:** Requires n8n version supporting MCP trigger nodes (v1.95+ recommended)  
- **Potential failures:**  
  - Webhook not activated or URL misconfigured  
  - Authentication failures if API key is missing or invalid  
- **Sub-workflow:** None  

---

#### 1.2 Headless Chrome PDF Conversion

**Overview:**  
This block contains three operations that convert raw HTML or web URLs into PDFs using the Headless Chrome engine.

**Nodes Involved:**  
- Convert raw HTML to PDF  
- Convert URL to PDF  
- Convert URL to PDF 1  
- Sticky Note (Headless Chrome)

**Node Details:**

- **Convert raw HTML to PDF**  
  - Type: HTTP Request Tool node  
  - Role: POST request to `https://v2018.api2pdf.com/chrome/html`  
  - Auth: API key in HTTP header `Authorization`  
  - Parameters: Body contains raw HTML (populated via AI expressions)  
  - Input: From MCP trigger  
  - Output: Response sent back to MCP trigger  
  - Failure modes:  
    - Invalid HTML input  
    - Network or API errors  
    - Authentication errors  

- **Convert URL to PDF**  
  - Type: HTTP Request Tool node  
  - Role: GET request to `https://v2018.api2pdf.com/chrome/url` with query params  
  - Query Parameters:  
    - `url` (required): URL to convert, provided via `$fromAI('url')` expression, must start with http:// or https://  
    - `output` (optional): Defaults to PDF file, can be set to JSON for structured response  
  - Auth: API key in HTTP header  
  - Input: From MCP trigger  
  - Output: To MCP trigger  
  - Edge cases:  
    - Invalid or unreachable URL  
    - Missing or malformed query parameters  

- **Convert URL to PDF 1**  
  - Type: HTTP Request Tool node  
  - Role: POST request to `https://v2018.api2pdf.com/chrome/url`  
  - Auth: API key in header  
  - Parameters: POST body expected to contain URL and other options (populated by AI)  
  - Input: From MCP trigger  
  - Output: To MCP trigger  

- **Sticky Note (Headless Chrome)**  
  - Provides section label and context for the above nodes  

---

#### 1.3 LibreOffice PDF Conversion

**Overview:**  
This block converts office documents or images into PDFs using the LibreOffice engine.

**Nodes Involved:**  
- Convert office document or image to PDF  
- Sticky Note2 (Libre Office)

**Node Details:**

- **Convert office document or image to PDF**  
  - Type: HTTP Request Tool node  
  - Role: POST request to `https://v2018.api2pdf.com/libreoffice/convert`  
  - Auth: API key in header  
  - Parameters: Document or image file data expected in the request body (populated by AI)  
  - Input: From MCP trigger  
  - Output: To MCP trigger  
  - Edge cases:  
    - Unsupported file types  
    - Large file uploads causing timeouts  
    - Authentication issues  

- **Sticky Note2 (Libre Office)**  
  - Section label for LibreOffice conversion nodes  

---

#### 1.4 PDF Merge

**Overview:**  
Merges multiple PDF files into a single PDF document.

**Nodes Involved:**  
- Merge multiple PDFs together  
- Sticky Note3 (Merge / Combine PDFs)

**Node Details:**

- **Merge multiple PDFs together**  
  - Type: HTTP Request Tool node  
  - Role: POST request to `https://v2018.api2pdf.com/merge`  
  - Auth: API key in header  
  - Parameters: List of URLs or base64 PDFs to merge (populated by AI)  
  - Input: From MCP trigger  
  - Output: To MCP trigger  
  - Edge cases:  
    - Providing non-PDF files  
    - Large number of PDFs causing performance issues  
    - API limits or errors  

- **Sticky Note3**  
  - Section label for PDF merge functionality  

---

#### 1.5 WKHtmlToPdf Conversion

**Overview:**  
Converts raw HTML or URLs into PDFs using the wkhtmltopdf engine.

**Nodes Involved:**  
- Convert raw HTML to PDF 1  
- Convert URL to PDF 2  
- Convert URL to PDF 3  
- Sticky Note4 (W K Html To Pdf)

**Node Details:**

- **Convert raw HTML to PDF 1**  
  - Type: HTTP Request Tool node  
  - Role: POST request to `https://v2018.api2pdf.com/wkhtmltopdf/html`  
  - Auth: API key in header  
  - Parameters: Raw HTML content in POST body (populated via AI)  
  - Input: From MCP trigger  
  - Output: To MCP trigger  

- **Convert URL to PDF 2**  
  - Type: HTTP Request Tool node  
  - Role: GET request to `https://v2018.api2pdf.com/wkhtmltopdf/url` with query params  
  - Query Parameters:  
    - `url` (required)  
    - `output` (optional)  
  - Auth: API key in header  
  - Input: From MCP trigger  
  - Output: To MCP trigger  

- **Convert URL to PDF 3**  
  - Type: HTTP Request Tool node  
  - Role: POST request to `https://v2018.api2pdf.com/wkhtmltopdf/url`  
  - Auth: API key in header  
  - Parameters: POST body with URL and options  
  - Input: From MCP trigger  
  - Output: To MCP trigger  

- **Sticky Note4**  
  - Section label for WKHtmlToPdf operations  

---

#### 1.6 ZXing Barcode Generation

**Overview:**  
Generates barcodes and QR codes using the ZXing library.

**Nodes Involved:**  
- Generate bar codes and QR codes with ZXING.  
- Sticky Note5 (Zxing (zebra Crossing) Bar Codes)

**Node Details:**

- **Generate bar codes and QR codes with ZXING.**  
  - Type: HTTP Request Tool node  
  - Role: GET request to `https://v2018.api2pdf.com/zebra` with query parameters  
  - Query Parameters (all populated via AI expressions):  
    - `format` (required): Barcode format, e.g., CODE_39 or QR_CODE  
    - `value` (required): Text to encode  
    - `showlabel` (optional, boolean): Show label below barcode  
    - `height` (optional, number): Image height  
    - `width` (optional, number): Image width  
  - Auth: API key in header  
  - Input: From MCP trigger  
  - Output: To MCP trigger  
  - Edge cases:  
    - Unsupported barcode format  
    - Missing required parameters  
    - Large image sizes causing issues  

- **Sticky Note5**  
  - Section label for ZXing barcode generation  

---

#### 1.7 Setup and Documentation Notes

**Overview:**  
Sticky notes providing detailed setup instructions, usage guidance, API overview, and customization tips.

**Nodes Involved:**  
- Setup Instructions  
- Workflow Overview

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Content summarizes:  
    - How to import and configure the workflow  
    - API key setup for header authentication with key name `Authorization`  
    - Steps to activate and use the MCP webhook URL  
    - Notes on AI parameter auto-population and API endpoints  
    - Suggestions for customization and error handling  
    - Support Discord link: [https://discord.me/cfomodz](https://discord.me/cfomodz)  
    - n8n documentation link: [https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Provides detailed description of Api2Pdf service, supported engines, authentication methods, quickstart example, and the nine API operations exposed by the workflow.  
  - Contains useful links to Api2Pdf SDKs and registration portal.

---

### 3. Summary Table

| Node Name                                           | Node Type                              | Functional Role                                 | Input Node(s)                                     | Output Node(s)                                   | Sticky Note                                                                                                          |
|----------------------------------------------------|--------------------------------------|------------------------------------------------|--------------------------------------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                                 | Sticky Note                         | Setup and configuration guidance               | None                                             | None                                            | Detailed setup instructions, usage notes, customization tips, Discord and documentation links                         |
| Workflow Overview                                  | Sticky Note                         | API service description and workflow overview  | None                                             | None                                            | Detailed API overview, authentication, supported operations, SDK links                                                |
| Api2Pdf - PDF Generation, Powered by AWS Lambda MCP Server | MCP Trigger                        | Entry point for AI agents, triggers workflow   | None                                             | All HTTP Request nodes                           |                                                                                                                      |
| Sticky Note                                        | Sticky Note                         | Section label: Headless Chrome                  | None                                             | None                                            | Headless Chrome conversion operations section label                                                                  |
| Convert raw HTML to PDF                            | HTTP Request Tool                   | Convert raw HTML to PDF via Headless Chrome    | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Convert URL to PDF                                 | HTTP Request Tool                   | Convert URL to PDF via Headless Chrome (GET)   | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Convert URL to PDF 1                               | HTTP Request Tool                   | Convert URL to PDF via Headless Chrome (POST)  | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Sticky Note2                                       | Sticky Note                         | Section label: Libre Office                      | None                                             | None                                            | Libre Office conversion section label                                                                                 |
| Convert office document or image to PDF           | HTTP Request Tool                   | Convert office docs or images to PDF via LibreOffice | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Sticky Note3                                       | Sticky Note                         | Section label: Merge / Combine PDFs             | None                                             | None                                            | Merge PDF files section                                                                                               |
| Merge multiple PDFs together                       | HTTP Request Tool                   | Merge multiple PDFs into one                     | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Sticky Note4                                       | Sticky Note                         | Section label: WKHtmlToPdf                       | None                                             | None                                            | WKHtmlToPdf conversion section                                                                                        |
| Convert raw HTML to PDF 1                          | HTTP Request Tool                   | Convert raw HTML to PDF via wkhtmltopdf         | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Convert URL to PDF 2                               | HTTP Request Tool                   | Convert URL to PDF via wkhtmltopdf (GET)        | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Convert URL to PDF 3                               | HTTP Request Tool                   | Convert URL to PDF via wkhtmltopdf (POST)       | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |
| Sticky Note5                                       | Sticky Note                         | Section label: ZXing (Zebra Crossing) Bar Codes | None                                             | None                                            | ZXing barcode and QR code generation section                                                                         |
| Generate bar codes and QR codes with ZXING.       | HTTP Request Tool                   | Generate barcodes and QR codes                   | MCP Trigger                                       | MCP Trigger                                      |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Name: `Api2Pdf - PDF Generation, Powered by AWS Lambda MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set webhook path: `api2pdf---pdf-generation,-powered-by-aws-lambda-mcp`  
   - This node will serve as the main entry point for AI requests.

2. **Set up Credentials**  
   - Create new credentials of type `Generic Credential` for API key in HTTP header.  
   - Name the header key as `Authorization`.  
   - Provide your Api2Pdf API key as the credential value.

3. **Add HTTP Request Tool Nodes for Each Operation**

   - For **Headless Chrome**:  
     - **Convert raw HTML to PDF**  
       - Method: POST  
       - URL: `https://v2018.api2pdf.com/chrome/html`  
       - Authentication: Use the API key credentials in header  
       - Body: Raw HTML content (set to be populated dynamically via AI)  
       - Connect input from MCP trigger.

     - **Convert URL to PDF (GET)**  
       - Method: GET  
       - URL: `https://v2018.api2pdf.com/chrome/url`  
       - Query Parameters:  
         - `url` (required): use expression `$fromAI('url')`  
         - `output` (optional): `$fromAI('output')`  
       - Authentication: API key header  
       - Connect input from MCP trigger.

     - **Convert URL to PDF (POST)**  
       - Method: POST  
       - URL: `https://v2018.api2pdf.com/chrome/url`  
       - Authentication: API key header  
       - Body: Set to receive URL and options dynamically  
       - Connect input from MCP trigger.

   - For **LibreOffice**:  
     - **Convert office document or image to PDF**  
       - Method: POST  
       - URL: `https://v2018.api2pdf.com/libreoffice/convert`  
       - Authentication: API key header  
       - Body: Office document or image data (dynamic)  
       - Connect input from MCP trigger.

   - For **Merge PDFs**:  
     - **Merge multiple PDFs together**  
       - Method: POST  
       - URL: `https://v2018.api2pdf.com/merge`  
       - Authentication: API key header  
       - Body: List of PDF URLs or base64 PDFs (dynamic)  
       - Connect input from MCP trigger.

   - For **WKHtmlToPdf**:  
     - **Convert raw HTML to PDF 1**  
       - Method: POST  
       - URL: `https://v2018.api2pdf.com/wkhtmltopdf/html`  
       - Authentication: API key header  
       - Body: Raw HTML content (dynamic)  
       - Connect input from MCP trigger.

     - **Convert URL to PDF 2 (GET)**  
       - Method: GET  
       - URL: `https://v2018.api2pdf.com/wkhtmltopdf/url`  
       - Query Parameters:  
         - `url` (required) `$fromAI('url')`  
         - `output` (optional) `$fromAI('output')`  
       - Authentication: API key header  
       - Connect input from MCP trigger.

     - **Convert URL to PDF 3 (POST)**  
       - Method: POST  
       - URL: `https://v2018.api2pdf.com/wkhtmltopdf/url`  
       - Authentication: API key header  
       - Body: URL and options (dynamic)  
       - Connect input from MCP trigger.

   - For **ZXing Barcode Generation**:  
     - **Generate bar codes and QR codes with ZXING.**  
       - Method: GET  
       - URL: `https://v2018.api2pdf.com/zebra`  
       - Query Parameters:  
         - `format` (string, required) `$fromAI('format')` (e.g., CODE_39, QR_CODE)  
         - `value` (string, required) `$fromAI('value')`  
         - `showlabel` (boolean, optional) `$fromAI('showlabel')`  
         - `height` (number, optional) `$fromAI('height')`  
         - `width` (number, optional) `$fromAI('width')`  
       - Authentication: API key header  
       - Connect input from MCP trigger.

4. **Link all HTTP Request Tool nodes’ outputs back to the MCP trigger node**  
   - This allows responses from external API calls to be returned to the calling AI agent.

5. **Add Sticky Notes for Documentation and Section Labels**  
   - Create sticky notes with the content describing setup instructions, API overview, and grouping labels for each block (Headless Chrome, LibreOffice, Merge PDFs, WKHtmlToPdf, ZXing).

6. **Activate the Workflow**  
   - Ensure credentials are set with a valid Api2Pdf API key.  
   - Enable the workflow to listen on the MCP webhook URL.

7. **Testing**  
   - Use the MCP webhook URL in your AI agent or HTTP client.  
   - Send requests with appropriate parameters for desired operations, leveraging `$fromAI()` expressions for dynamic input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                        |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Setup instructions emphasize API key authentication using the header key `Authorization`.                                          | Setup Instructions sticky note                                                                                         |
| The workflow exposes 9 API operations covering PDF conversion via different engines and barcode generation.                        | Workflow Overview sticky note                                                                                          |
| The Api2Pdf API is serverless, running on AWS Lambda, allowing scalable, cost-effective PDF generation without rate limits.        | Workflow Overview sticky note                                                                                          |
| Official Api2Pdf SDKs available for Python, .NET, Node.js, PHP; Ruby SDK coming soon.                                               | Workflow Overview sticky note (links included)                                                                         |
| Support for AI-driven parameter auto-population via `$fromAI()` expressions.                                                       | Setup Instructions sticky note                                                                                         |
| For integration help and custom automation support, join the Discord community at: https://discord.me/cfomodz                     | Setup Instructions sticky note                                                                                         |
| n8n MCP Trigger documentation: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/        | Setup Instructions sticky note                                                                                         |
| Recommended n8n version should support MCP Trigger nodes and HTTP Request Tool with generic API key authentication (v1.95+).       | General operational note                                                                                               |

---

**End of Workflow Documentation**