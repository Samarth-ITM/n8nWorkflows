Automated Request-to-Quote with OpenAI, Google Sheets & CraftMyPDF

https://n8nworkflows.xyz/workflows/automated-request-to-quote-with-openai--google-sheets---craftmypdf-8239


# Automated Request-to-Quote with OpenAI, Google Sheets & CraftMyPDF

### 1. Workflow Overview

This workflow automates the entire Request-to-Quote (RTQ) process by integrating a user input form, product data lookup, AI-driven quote generation, PDF creation, and email delivery. It is designed for businesses or services that want to streamline quote generation and delivery with minimal manual intervention. The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Captures user quote requests via a web form trigger node.
- **1.2 Product Data Retrieval:** Fetches relevant product information from Google Sheets based on the request.
- **1.3 AI Quote Generation:** Uses OpenAI (via LangChain integration) to select products and generate a quote in JSON format.
- **1.4 PDF Preparation:** Maps AI output into the format required by CraftMyPDF and generates a PDF document.
- **1.5 PDF Retrieval and Emailing:** Downloads the generated PDF file and sends it to the requester via email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives quote requests submitted by users through a web form. This is the entry point of the workflow.

- **Nodes Involved:**  
  - Form: Request a Quote

- **Node Details:**  
  - **Form: Request a Quote**  
    - Type: Form Trigger  
    - Role: Listens for incoming form submissions with quote request data.  
    - Configuration: Uses a webhook with ID "FORM-QUOTE-ID" that receives HTTP POST requests carrying form data.  
    - Inputs: External web form submissions.  
    - Outputs: Passes form data to the next node (Google Sheets lookup).  
    - Edge Cases:  
      - Missing or malformed form data could cause downstream errors.  
      - Webhook connectivity issues or incorrect webhook ID would prevent triggering.  
      - Rate limiting or spam submissions may require mitigation outside the workflow.

---

#### 2.2 Product Data Retrieval

- **Overview:**  
Queries Google Sheets to retrieve product information needed for the quote based on the form data.

- **Nodes Involved:**  
  - Google Sheets: Products

- **Node Details:**  
  - **Google Sheets: Products**  
    - Type: Google Sheets node  
    - Role: Reads product data from a specified spreadsheet to support quote generation.  
    - Configuration: Uses credentials linked to a Google account with access to the product sheet.  
    - Inputs: Receives form data from the Form node to filter or identify relevant products.  
    - Outputs: Passes product data to the AI node.  
    - Edge Cases:  
      - Authentication failures with Google API.  
      - Missing or outdated product data in the sheet.  
      - API call limits or timeouts.  
      - Data format mismatches between sheet and AI expectations.

---

#### 2.3 AI Quote Generation

- **Overview:**  
Invokes an OpenAI model through LangChain to select products and generate a detailed quote in JSON format.

- **Nodes Involved:**  
  - LLM: Select & Quote (JSON only)

- **Node Details:**  
  - **LLM: Select & Quote (JSON only)**  
    - Type: LangChain OpenAI node  
    - Role: Processes product data and request details, returning a structured JSON quote.  
    - Configuration: Uses OpenAI credentials (API key), with model and prompt set to produce JSON output suitable for quoting.  
    - Inputs: Product data from Google Sheets and request context.  
    - Outputs: JSON quote passed to the Code node for further processing.  
    - Edge Cases:  
      - OpenAI API key missing, invalid, or rate limited.  
      - Unexpected AI responses or malformed JSON outputs.  
      - Timeout or connectivity issues with OpenAI APIs.  
      - Prompt misconfiguration causing irrelevant or incomplete quotes.

---

#### 2.4 PDF Preparation

- **Overview:**  
Transforms the AI-generated JSON quote into a format compatible with CraftMyPDF and creates the PDF document.

- **Nodes Involved:**  
  - Code: Map for CraftMyPDF  
  - Create a PDF

- **Node Details:**  
  - **Code: Map for CraftMyPDF**  
    - Type: Code (JavaScript) node  
    - Role: Maps and formats the JSON quote output from the AI node into the input format required by CraftMyPDF.  
    - Configuration: Custom JavaScript code parses the JSON and restructures it according to the PDF template schema.  
    - Inputs: JSON quote from the LLM node.  
    - Outputs: Formatted JSON data for PDF generation.  
    - Edge Cases:  
      - Parsing errors if the AI output is malformed.  
      - Schema mismatches with CraftMyPDF expectations.  
      - Runtime errors in custom code.  

  - **Create a PDF**  
    - Type: CraftMyPDF node  
    - Role: Generates a PDF document from the mapped data using a predefined PDF template.  
    - Configuration: Uses credentials and a configured PDF template ID in CraftMyPDF.  
    - Inputs: Mapped JSON data from the Code node.  
    - Outputs: A request to get PDF file URL.  
    - Edge Cases:  
      - Invalid or missing template ID or credentials.  
      - PDF generation failures due to malformed input data.  
      - API rate limits or timeouts.

---

#### 2.5 PDF Retrieval and Emailing

- **Overview:**  
Downloads the generated PDF file and emails it to the requester.

- **Nodes Involved:**  
  - Get PDF file  
  - Email: Send Quote

- **Node Details:**  
  - **Get PDF file**  
    - Type: HTTP Request node  
    - Role: Retrieves the generated PDF file from CraftMyPDF via HTTP GET.  
    - Configuration: Uses URL received from the previous node to download the binary PDF content.  
    - Inputs: PDF generation output from Create a PDF node.  
    - Outputs: Binary PDF file to Email node.  
    - Edge Cases:  
      - HTTP errors (404, 403, 500) if URL is invalid or expired.  
      - Network timeouts or connectivity issues.  

  - **Email: Send Quote**  
    - Type: Email Send node  
    - Role: Sends the PDF quote as an email attachment to the requester.  
    - Configuration: Uses SMTP or other email credentials configured in n8n. Email parameters (recipient, subject, body) are dynamically set based on form input.  
    - Inputs: Binary PDF file and requester email address.  
    - Outputs: None (end of workflow).  
    - Edge Cases:  
      - Email server authentication failures.  
      - Invalid email addresses from form input.  
      - Attachment size limits or email sending errors.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                      | Input Node(s)              | Output Node(s)             | Sticky Note                                |
|---------------------------|--------------------------------|------------------------------------|----------------------------|----------------------------|--------------------------------------------|
| Form: Request a Quote      | Form Trigger                   | Captures quote requests             | -                          | Google Sheets: Products     |                                            |
| Google Sheets: Products    | Google Sheets                  | Retrieves product data              | Form: Request a Quote       | LLM: Select & Quote (JSON only) |                                            |
| LLM: Select & Quote (JSON only) | LangChain OpenAI Node          | Generates quote JSON via AI         | Google Sheets: Products     | Code: Map for CraftMyPDF    |                                            |
| Code: Map for CraftMyPDF   | Code (JavaScript)              | Maps AI output for PDF generation   | LLM: Select & Quote (JSON only) | Create a PDF               |                                            |
| Create a PDF              | CraftMyPDF                    | Generates PDF document              | Code: Map for CraftMyPDF    | Get PDF file                |                                            |
| Get PDF file              | HTTP Request                  | Downloads generated PDF             | Create a PDF               | Email: Send Quote           |                                            |
| Email: Send Quote          | Email Send                    | Emails PDF quote to requester       | Get PDF file               | -                          |                                            |
| Sticky Note               | Sticky Note                   | -                                  | -                          | -                          |                                            |
| Sticky Note1              | Sticky Note                   | -                                  | -                          | -                          |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named "Form: Request a Quote".  
   - Configure webhook with a unique ID (e.g., "FORM-QUOTE-ID").  
   - No additional parameters needed.  
   - This node starts the workflow upon form submission.

2. **Create the Google Sheets Node**  
   - Add a **Google Sheets** node named "Google Sheets: Products".  
   - Set operation to **Read Rows** or appropriate operation to fetch product data.  
   - Configure Google OAuth2 credentials with access to the products spreadsheet.  
   - Link input from "Form: Request a Quote".  
   - Configure sheet name and range to target product data.

3. **Create the OpenAI LangChain Node**  
   - Add a **LangChain OpenAI** node named "LLM: Select & Quote (JSON only)".  
   - Configure with your OpenAI API credentials.  
   - Set the model (e.g., GPT-4 or GPT-3.5) and prompt to generate a JSON structured quote based on product data and request input.  
   - Connect input from "Google Sheets: Products".

4. **Create the Code Node for Mapping**  
   - Add a **Code** node named "Code: Map for CraftMyPDF".  
   - Paste JavaScript code to transform the AI’s JSON output into the CraftMyPDF template format.  
   - Connect input from "LLM: Select & Quote (JSON only)".

5. **Create the CraftMyPDF Node**  
   - Add a **CraftMyPDF** node named "Create a PDF".  
   - Configure with CraftMyPDF API credentials and specify the PDF template ID.  
   - Connect input from "Code: Map for CraftMyPDF".

6. **Create the HTTP Request Node to Get PDF File**  
   - Add an **HTTP Request** node named "Get PDF file".  
   - Set method to GET.  
   - Use the PDF URL output from "Create a PDF" as the request URL.  
   - Connect input from "Create a PDF".

7. **Create the Email Send Node**  
   - Add an **Email Send** node named "Email: Send Quote".  
   - Configure SMTP or email service credentials (e.g., Outlook, Gmail).  
   - Set recipient email dynamically from form data.  
   - Attach the binary PDF file from "Get PDF file".  
   - Configure subject and body text for the quote email.  
   - Connect input from "Get PDF file".

8. **Connect Nodes in Sequence**  
   - Connect in this order:  
     Form Trigger → Google Sheets → LLM (OpenAI) → Code → CraftMyPDF → HTTP Request (Get PDF) → Email Send.

9. **Test the Workflow**  
   - Deploy and submit test quote requests via the form.  
   - Validate product data retrieval, AI quote generation, PDF formatting, and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                   |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow automates Request-to-Quote using OpenAI, Google Sheets, and CraftMyPDF integrations.      | General workflow purpose                          |
| CraftMyPDF integration requires a valid template ID and API credentials.                           | CraftMyPDF official documentation                 |
| OpenAI API usage requires API key and proper prompt engineering to ensure JSON output format.      | OpenAI API documentation                          |
| Ensure Google Sheets access is granted with OAuth2 credentials configured in n8n.                  | Google Sheets API setup guidelines                |
| Email node requires SMTP or OAuth2 credentials (e.g., Gmail, Outlook) for sending emails reliably. | n8n email node documentation                       |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.