Generate SSL/TLS Certificate Expiry Reports with AWS ACM and AI for Slack & Email

https://n8nworkflows.xyz/workflows/generate-ssl-tls-certificate-expiry-reports-with-aws-acm-and-ai-for-slack---email-7689


# Generate SSL/TLS Certificate Expiry Reports with AWS ACM and AI for Slack & Email

---

### 1. Workflow Overview

This workflow automates the generation and distribution of weekly SSL/TLS certificate expiry reports for AWS ACM (AWS Certificate Manager). It targets DevOps engineers, cloud administrators, and compliance teams responsible for managing AWS infrastructure certificates, providing them with timely insights to prevent certificate expiry risks and maintain compliance.

The workflow is logically divided into five main blocks:

- **1.1 Schedule Trigger:** Initiates the workflow automatically on a weekly basis.
- **1.2 AWS Certificate Retrieval & Data Parsing:** Fetches all AWS ACM certificates and processes raw data into a structured JSON format.
- **1.3 AI-Powered Report Generation:** Uses OpenAI agents to create two report formats:
  - Markdown report for Slack distribution.
  - (Optional/disabled) HTML report for email summaries.
- **1.4 Document Conversion & Delivery:** Converts Markdown reports to PDF and uploads them to Slack channels; optionally sends HTML reports via email.
- **1.5 Metadata Configuration & Credential Management:** Handles file metadata, Google Drive integration, and credential setup for external services (AWS, OpenAI, Slack, Google Drive, SendGrid).

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Automatically triggers the workflow once per week (e.g., every Monday at 08:00 UTC) to maintain consistent monitoring.

- **Nodes Involved:**  
  - Weekly schedule trigger

- **Node Details:**  
  - **Weekly schedule trigger**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger every week on Monday (day 1).  
    - Inputs: None (start node)  
    - Outputs: Connected to "Get many certificates" node.  
    - Edge Cases:  
      - Timezone misconfiguration may cause unexpected trigger times.  
      - Workflow disabled or credentials missing would prevent execution.  
    - Version: 1.2  

---

#### 2.2 AWS Certificate Retrieval & Data Parsing

- **Overview:**  
  Retrieves all ACM certificates from AWS and reformats the data into a clean, enriched JSON object for reporting.

- **Nodes Involved:**  
  - Get many certificates  
  - Parse ACM Data

- **Node Details:**  
  - **Get many certificates**  
    - Type: AWS Certificate Manager node  
    - Operation: "getMany" to fetch all certificates.  
    - AWS Region: ap-southeast-1  
    - Inputs: From schedule trigger  
    - Outputs: Certificate data array to "Parse ACM Data"  
    - Credentials: AWS credentials with ACM access.  
    - Edge Cases:  
      - AWS API throttling or permission errors.  
      - Network timeout or invalid credentials.  
    - Version: 1  

  - **Parse ACM Data**  
    - Type: Function (Code) node  
    - Role: Transforms raw ACM data fields: converts UNIX timestamps to ISO strings, flattens arrays of SANs and key usages, adds human-readable booleans, and computes summary statistics (total, expired count, in-use count).  
    - Key Expressions:  
      - Uses JavaScript Date conversion from UNIX timestamps (seconds to milliseconds).  
      - Filters for status "EXPIRED" and boolean InUse.  
    - Inputs: Raw JSON from AWS node  
    - Outputs: Structured JSON with array `certificates` and summary counts.  
    - Edge Cases:  
      - Empty or malformed AWS responses.  
      - Unexpected null or missing fields.  
    - Version: 2  

---

#### 2.3 AI-Powered Report Generation

- **Overview:**  
  Generates human-readable certificate expiry reports in Markdown (active) and optionally HTML (disabled) formats using OpenAI language models.

- **Nodes Involved:**  
  - OpenAI Chat Model1 (Markdown Agent input)  
  - Certificate Summary Markdown Agent  
  - OpenAI Chat Model (HTML Agent input) [disabled]  
  - Certificate Summary HTML Agent [disabled]

- **Node Details:**  
  - **Certificate Summary Markdown Agent**  
    - Type: LangChain Agent (OpenAI)  
    - Function: Converts structured JSON certificate data into a formatted Markdown report.  
    - Prompts:  
      - System prompt instructs to include summary stats and a Markdown table sorted by expiry date, highlighting expired certificates with ⚠️.  
      - Input data injected as JSON string.  
    - Inputs: From "OpenAI Chat Model1" node output.  
    - Outputs: Markdown text for report content.  
    - Edge Cases:  
      - OpenAI API rate limits, invalid API key.  
      - Model outputs incomplete or malformed markdown.  
    - Version: 2.1  

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model  
    - Model: gpt-4.1-mini  
    - Role: Provides input to Markdown Agent.  
    - Inputs: Parsed certificate data from "Parse ACM Data".  
    - Credentials: OpenAI API key.  
    - Outputs: To "Certificate Summary Markdown Agent".  
    - Edge Cases: API errors or latency.  
    - Version: 1.2  

  - **Certificate Summary HTML Agent** (disabled)  
    - Same role as Markdown agent but outputs HTML formatted report.  
    - Disabled in this workflow but can be enabled for email.  
    - Uses system prompt to generate an HTML table with color-coded statuses.  
    - Inputs: From "OpenAI Chat Model".  
    - Outputs: HTML string.  
    - Edge Cases: Same as Markdown agent.  

  - **OpenAI Chat Model** (HTML input)  
    - Model: gpt-5-mini  
    - Provides input to HTML agent.  
    - Disabled path in current workflow.  

---

#### 2.4 Document Conversion & Delivery

- **Overview:**  
  Converts Markdown report to Google Docs format, then exports as PDF and uploads to Slack. Optionally sends HTML report by email.

- **Nodes Involved:**  
  - Configure metadata  
  - Create document file (Google Drive HTTP Request)  
  - Convert to PDF (Google Drive)  
  - Send Weekly ACM Report PDF (Slack)  
  - Send Weekly ACM Report Email (SendGrid) [disabled]

- **Node Details:**  
  - **Configure metadata**  
    - Type: Set node  
    - Role: Defines metadata for Google Docs upload including folder ID and document content (Markdown output).  
    - Inputs: Markdown output from "Certificate Summary Markdown Agent".  
    - Outputs: Passes metadata to "Create document file".  
    - Edge Cases: Missing or incorrect folder ID may cause upload failure.  
    - Version: 3.4  

  - **Create document file**  
    - Type: HTTP Request  
    - Role: Uploads Markdown report as a Google Docs file via Google Drive API multipart upload.  
    - Parameters:  
      - Uses multipart/related content type with boundary.  
      - Document name includes date/time stamp.  
      - Folder ID set from metadata.  
    - Credentials: Google Drive OAuth2.  
    - Inputs: Metadata from Set node.  
    - Outputs: Google Docs file metadata (id) for next step.  
    - Edge Cases: OAuth token expiry, API errors, quota limits.  
    - Version: 4.2  

  - **Convert to PDF**  
    - Type: Google Drive node  
    - Role: Downloads the created Google Docs file as PDF.  
    - Parameters: File ID from previous node, conversion to PDF.  
    - Credentials: Google Drive OAuth2.  
    - Inputs: Google Docs file ID.  
    - Outputs: PDF file binary data.  
    - Edge Cases: Conversion errors, API failures.  
    - Version: 3  

  - **Send Weekly ACM Report PDF**  
    - Type: Slack node  
    - Role: Uploads the generated PDF file to a specific Slack channel with a comment.  
    - Parameters:  
      - Channel ID specified (`C097VAKKPUP`).  
      - Initial comment included.  
    - Credentials: Slack OAuth2.  
    - Inputs: PDF binary data from "Convert to PDF".  
    - Edge Cases: Slack API failure, permission issues, file size limits.  
    - Version: 2.3  

  - **Send Weekly ACM Report Email** (disabled)  
    - Type: SendGrid node  
    - Role: Sends the HTML report as an email to configured recipients.  
    - Parameters: Subject, From and To email addresses as workflow variables.  
    - Credentials: SendGrid API key.  
    - Inputs: HTML report from "Certificate Summary HTML Agent".  
    - Edge Cases: Email delivery failures, invalid addresses, API limits.  
    - Version: 1  

---

#### 2.5 Metadata Configuration & Credential Management

- **Overview:**  
  Handles setting workflow data such as sender/recipient email addresses and Google Drive folder IDs. Manages credentials for external services.

- **Nodes Involved:**  
  - Set Workflow Data (disabled)  
  - Credentials used by multiple nodes (AWS, OpenAI, Google Drive, Slack, SendGrid)

- **Node Details:**  
  - **Set Workflow Data**  
    - Type: Set node  
    - Role: Defines email sender name, sender email, and recipient email as workflow variables.  
    - Disabled in current workflow (used only if email sending enabled).  
    - Version: 3.4  

  - **Credentials**  
    - AWS: Access to AWS ACM in region ap-southeast-1.  
    - OpenAI: API key for ChatGPT models (gpt-4.1-mini, gpt-5-mini).  
    - Google Drive OAuth2: For document creation and conversion.  
    - Slack OAuth2: For file upload to Slack channel.  
    - SendGrid API: For email sending (disabled).  
    - Edge Cases: Expired or missing credentials cause node errors or workflow failure.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                          | Input Node(s)                 | Output Node(s)                              | Sticky Note                                                                                                                        |
|--------------------------------|----------------------------------|----------------------------------------|------------------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Weekly schedule trigger         | Schedule Trigger                 | Trigger workflow weekly                 | None                         | Get many certificates                       | ### 1. ⏰ Weekly Schedule Trigger<br>The workflow is automatically triggered every week using a scheduled event (e.g., every Monday at 08:00 UTC) |
| Get many certificates           | AWS Certificate Manager          | Fetch all ACM certificates              | Weekly schedule trigger       | Parse ACM Data                             | ### 2. 📥 Retrieve ACM Certificates<br>Uses the `Get many certificates` action from AWS Certificate Manager to fetch all issued, expired, or in-use SSL/TLS certificates from your AWS account. |
| Parse ACM Data                  | Function (Code)                  | Parse & format raw AWS certificate data | Get many certificates         | Certificate Summary HTML Agent, Certificate Summary Markdown Agent | ### 3. 🧮 Parse and Format Certificate Data<br>A function node processes the raw certificate data, converting UNIX timestamps to readable dates, flattening arrays, and calculating useful stats like total, expired, and in-use certificates. |
| OpenAI Chat Model1              | LangChain OpenAI Chat Model     | Provide AI input for Markdown report   | Parse ACM Data                | Certificate Summary Markdown Agent         |                                                                                                                                   |
| Certificate Summary Markdown Agent | LangChain Agent (OpenAI)        | Generate Markdown certificate report   | OpenAI Chat Model1            | Configure metadata                         | ### 4. 📝 Generate Markdown Report<br>The `Certificate Summary Markdown Agent` (powered by OpenAI) takes the structured JSON data and converts it into a clean, professional Markdown report, ready for export. |
| Configure metadata             | Set                             | Define document metadata for Google Docs | Certificate Summary Markdown Agent | Create document file                       |                                                                                                                                   |
| Create document file           | HTTP Request (Google Drive API) | Upload Markdown as Google Docs          | Configure metadata            | Convert to PDF                             |                                                                                                                                   |
| Convert to PDF                 | Google Drive                    | Convert Google Docs to PDF              | Create document file          | Send Weekly ACM Report PDF                  | ### 5. 🗂️ Convert Markdown to PDF<br>The Markdown output is transformed into a `.md` file and then converted to a PDF file format for easier distribution and archiving. |
| Send Weekly ACM Report PDF     | Slack                          | Upload PDF report to Slack channel      | Convert to PDF                | None                                        | ### 6. 💬 Send PDF Report to Slack<br>The finalized PDF report is uploaded to a specific Slack channel (e.g., `#it-security` or `#cloud-ops`) so stakeholders are instantly notified of certificate status. |
| Certificate Summary HTML Agent | LangChain Agent (OpenAI)        | Generate HTML certificate report (disabled) | Parse ACM Data                | Set Workflow Data                         |                                                                                                                                   |
| OpenAI Chat Model              | LangChain OpenAI Chat Model     | Provide AI input for HTML report (disabled) | Parse ACM Data                | Certificate Summary HTML Agent             |                                                                                                                                   |
| Set Workflow Data              | Set                             | Define email metadata (disabled)        | Certificate Summary HTML Agent | Send Weekly ACM Report Email (disabled)    |                                                                                                                                   |
| Send Weekly ACM Report Email   | SendGrid                       | Send HTML report via email (disabled)   | Set Workflow Data             | None                                        | ### 7. 📧 Send Weekly Email Summary<br>The HTML report is sent via email to IT, DevOps, or compliance teams, providing a clear and accessible summary of certificate health directly in their inbox. |
| Sticky Note                    | Sticky Note                    | Documentation and explanation notes     | None                         | None                                        | See all sticky notes in detailed sections; includes workflow purpose, screenshots, and setup instructions.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Schedule Trigger` node:**  
   - Set to trigger weekly on Monday (day 1) at 08:00 UTC.  
   - Leave default timezone unless specific adjustment required.

3. **Add `AWS Certificate Manager` node:**  
   - Operation: `getMany` to fetch all ACM certificates.  
   - Configure AWS credentials with access to ACM in region `ap-southeast-1`.  
   - Connect output of `Schedule Trigger` to this node.

4. **Add `Function` node named "Parse ACM Data":**  
   - Paste JavaScript code to process certificates:  
     - Convert UNIX timestamps (seconds) to ISO strings.  
     - Flatten arrays for SANs, key usages.  
     - Convert booleans to "Yes"/"No" strings.  
     - Calculate totals and counts of expired/in-use certs.  
   - Connect output of AWS node to this function.

5. **Add `LangChain OpenAI Chat Model` node ("OpenAI Chat Model1"):**  
   - Select model `gpt-4.1-mini` or equivalent.  
   - Set OpenAI API credentials.  
   - Connect output of "Parse ACM Data" node here.

6. **Add `LangChain Agent` node named "Certificate Summary Markdown Agent":**  
   - Set agent prompt to receive JSON certificate data and produce Markdown report:  
     - Include summary and table with specified columns.  
     - Highlight expired certs with ⚠️.  
     - Sort by expiry date descending.  
   - Connect output of OpenAI Chat Model1 node here.

7. **Add `Set` node named "Configure metadata":**  
   - Define two string fields:  
     - `Drive Folder ID`: Set to your Google Drive folder ID string (e.g., `"1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI"`).  
     - `Document Content`: Set to output Markdown from previous node using expression `={{ $json.output }}`.  
   - Connect output of Markdown Agent node here.

8. **Add `HTTP Request` node named "Create document file":**  
   - Method: POST  
   - URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`  
   - Headers:  
     - `Content-Type`: `multipart/related; boundary=foo_bar_baz`  
   - Body (raw): multipart with boundary `foo_bar_baz` including JSON metadata and Markdown content as text/markdown.  
   - Use expressions to insert current date/time in file name and folder ID from previous node.  
   - Credentials: Google Drive OAuth2.  
   - Connect output of "Configure metadata" node here.

9. **Add `Google Drive` node named "Convert to PDF":**  
   - Operation: Download file.  
   - File ID: Use ID from "Create document file" output.  
   - Conversion: Docs to PDF (`application/pdf`).  
   - Credentials: Google Drive OAuth2.  
   - Connect output of HTTP Request node here.

10. **Add `Slack` node named "Send Weekly ACM Report PDF":**  
    - Resource: File  
    - Authentication: OAuth2 with Slack credentials.  
    - Channel ID: Set to target Slack channel ID (e.g., `C097VAKKPUP`).  
    - Initial Comment: Provide message like "📄 The ACM Certificate Weekly Report is ready! Please find the generated PDF file attached for review and next steps."  
    - Connect output of "Convert to PDF" node here.

11. **(Optional) For HTML email report:**  
    - Add another LangChain OpenAI Chat Model node and Agent node configured for HTML output similar to Markdown but with HTML formatting.  
    - Use a `Set` node to configure sender and recipient email addresses.  
    - Add SendGrid node configured with API credentials to send the email with HTML content.  
    - Connect nodes accordingly. (Currently disabled in workflow.)

12. **Set up credentials:**  
    - AWS: Provide access with ACM permissions in required region.  
    - Google Drive OAuth2: For file creation and conversion.  
    - OpenAI API Key: For ChatGPT API calls.  
    - Slack OAuth2: For uploading files to Slack channel.  
    - SendGrid API Key: For sending emails (if used).

13. **Test the workflow:**  
    - Manually trigger to verify retrieval, parsing, report generation, document creation, PDF conversion, and Slack upload.  
    - Check Slack for PDF file appearance and formatting.  
    - Validate error handling (e.g., AWS API errors, OpenAI limits).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Automated SSL/TLS Certificate Expiry Report for AWS ACM generates weekly reports in Markdown (for Slack) and HTML (for email), providing comprehensive visibility into certificate status, expiry, and renewal eligibility.                                         | See sticky note content at workflow start with detailed explanation and screenshots.                                                                       |
| Workflow screenshot demonstrating node arrangement and flow is available: ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-21+at+2.35.17%E2%80%AFPM.png)                                                                                | Workflow design visualization.                                                                                                                            |
| Another screenshot showing PDF report upload to Slack: ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-21+at+2.37.24%E2%80%AFPM.png)                                                                                                | Slack integration confirmation.                                                                                                                           |
| For detailed setup, including AWS credential creation, Google Drive API setup, Slack OAuth app creation, and OpenAI key generation, refer to official n8n documentation and respective service provider guides.                                                     | n8n docs and external provider docs.                                                                                                                      |
| To customize frequency, modify the Schedule Trigger node interval. To filter certificates (e.g., only expired), update the parsing function node accordingly.                                                                                                      | Customization tips from sticky notes.                                                                                                                     |
| Replace Slack with other messaging apps by swapping the Slack node with corresponding integrations (e.g., Microsoft Teams, Telegram). Similarly, email can be enhanced or replaced with other SMTP or transactional email services.                                  | Expandability and integration notes.                                                                                                                      |
| The workflow respects all applicable content policies and handles only legal, public data.                                                                                                                                                                         | Disclaimer from workflow metadata.                                                                                                                        |

---

**Disclaimer:**  
The provided information is extracted exclusively from an automated workflow created with n8n, respecting all content policies. No illegal, offensive, or protected data is included. All handled data is legal and public.

---