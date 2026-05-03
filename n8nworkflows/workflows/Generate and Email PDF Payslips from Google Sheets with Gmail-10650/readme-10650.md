Generate and Email PDF Payslips from Google Sheets with Gmail

https://n8nworkflows.xyz/workflows/generate-and-email-pdf-payslips-from-google-sheets-with-gmail-10650


# Generate and Email PDF Payslips from Google Sheets with Gmail

### 1. Workflow Overview

This workflow automates the generation and delivery of employee payslips as PDF documents using data stored in Google Sheets and sends them via Gmail. It is designed for HR or payroll departments that want to streamline payslip distribution without manual intervention.

The workflow consists of the following logical blocks:

- **1.1 Input Reception and Initialization**: Starts the process manually and sets company-specific configuration.
- **1.2 Payroll Data Retrieval and Filtering**: Fetches payroll data from Google Sheets and filters out rows where payslips have already been sent.
- **1.3 Iteration and Payslip Processing**: Iterates through each payslip record to prepare data, generate HTML payslip content, convert it to PDF, and prepare it for delivery.
- **1.4 Email Delivery and Post-Processing**: Sends the payslip PDF via Gmail and marks the record in Google Sheets as emailed to avoid duplicates.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block starts the workflow on manual trigger and sets any necessary company configurations before retrieving payroll data.

- **Nodes Involved:**  
  - Manual Trigger  
  - Company Configuration

- **Node Details:**  

  - **Manual Trigger**  
    - Type: Trigger  
    - Role: Starts the workflow manually when run by user.  
    - Configuration: Default, no parameters.  
    - Input: None  
    - Output: Triggers "Company Configuration" node.  
    - Edge Cases: None.  
    - Version: 1

  - **Company Configuration**  
    - Type: Set  
    - Role: Holds static company configuration data (e.g., company name, email templates, etc.).  
    - Configuration: Typically sets variables or constants used later in the workflow.  
    - Input: Receives from Manual Trigger.  
    - Output: Passes data to "Fetch Payroll Data".  
    - Edge Cases: Misconfiguration can lead to incorrect email content or failure downstream.  
    - Version: 3.4

#### 2.2 Payroll Data Retrieval and Filtering

- **Overview:**  
  Retrieves payroll data from Google Sheets and filters out employees whose payslips have already been sent.

- **Nodes Involved:**  
  - Fetch Payroll Data  
  - Check Email Not Sent

- **Node Details:**  

  - **Fetch Payroll Data**  
    - Type: Google Sheets  
    - Role: Reads payroll data from a configured Google Sheets document.  
    - Configuration: Connects to Google Sheets with proper credentials, reads the payroll sheet, retrieves all rows (likely including employee details, salary, email, and email sent flag).  
    - Input: From "Company Configuration".  
    - Output: Sends data to "Check Email Not Sent".  
    - Edge Cases: Authentication errors, sheet access issues, empty or malformed data.  
    - Version: 4.7

  - **Check Email Not Sent**  
    - Type: Filter  
    - Role: Filters out rows where the payslip email has already been sent (based on a flag/column in the sheet).  
    - Configuration: Condition checks if the "email sent" column is false or empty.  
    - Input: From "Fetch Payroll Data".  
    - Output: Passes only unsent rows to "Iterate Payslip Rows".  
    - Edge Cases: Incorrect field name, condition failure leading to no output or duplicates.  
    - Version: 2.2

#### 2.3 Iteration and Payslip Processing

- **Overview:**  
  Iterates over each unsent payslip entry, prepares data, generates an HTML payslip, converts it to PDF, and prepares for email delivery.

- **Nodes Involved:**  
  - Iterate Payslip Rows  
  - Prepare Payslip Data  
  - Generate Payslip HTML  
  - Generate Payslip PDF  
  - Create PDF File

- **Node Details:**  

  - **Iterate Payslip Rows**  
    - Type: Split In Batches  
    - Role: Processes each payslip row individually in batches (likely for memory and API limits).  
    - Configuration: Batch size configured to control iteration load.  
    - Input: From "Check Email Not Sent".  
    - Output: Main output empty array branch (for rows without email sent), secondary output connects to "Prepare Payslip Data".  
    - Edge Cases: Large datasets may slow processing; empty input leads to no iteration.  
    - Version: 3

  - **Prepare Payslip Data**  
    - Type: Code (JavaScript)  
    - Role: Prepares and formats payslip data for HTML generation (e.g., formatting numbers, dates, or calculating totals).  
    - Configuration: Custom JavaScript code to manipulate input JSON data.  
    - Input: From "Iterate Payslip Rows" (secondary output).  
    - Output: Passes prepared data to "Generate Payslip HTML".  
    - Edge Cases: Code errors, undefined fields, unexpected data types.  
    - Version: 2

  - **Generate Payslip HTML**  
    - Type: HTML  
    - Role: Creates the HTML content of the payslip using templates and prepared data.  
    - Configuration: Uses a template with placeholders replaced by prepared data.  
    - Input: From "Prepare Payslip Data".  
    - Output: Sends generated HTML to "Generate Payslip PDF".  
    - Edge Cases: Template errors, missing variables.  
    - Version: 1.2

  - **Generate Payslip PDF**  
    - Type: Puppeteer  
    - Role: Converts the generated HTML payslip into a PDF document using headless Chrome rendering.  
    - Configuration: Puppeteer node configured for PDF generation with options like page size, margins.  
    - Input: HTML from previous node.  
    - Output: PDF buffer to "Create PDF File".  
    - Edge Cases: Rendering errors, Puppeteer crashes, large HTML content causing timeout.  
    - Version: 1

  - **Create PDF File**  
    - Type: Convert To File  
    - Role: Converts raw PDF buffer into a file object suitable for email attachments.  
    - Configuration: Sets filename, mime-type (application/pdf).  
    - Input: PDF buffer from Puppeteer.  
    - Output: Passes file to "Send Payslip Email".  
    - Edge Cases: File format errors, missing buffer data.  
    - Version: 1.1

#### 2.4 Email Delivery and Post-Processing

- **Overview:**  
  Sends the generated PDF payslip by email via Gmail and marks the corresponding row in Google Sheets as emailed to avoid resending.

- **Nodes Involved:**  
  - Send Payslip Email  
  - Mark Email Sent in Sheet

- **Node Details:**  

  - **Send Payslip Email**  
    - Type: Gmail  
    - Role: Sends the payslip PDF as an email attachment to the employee.  
    - Configuration: Uses Gmail OAuth2 credentials, sets recipient, subject, body, and attaches the PDF file.  
    - Input: PDF file from "Create PDF File".  
    - Output: On success, triggers "Mark Email Sent in Sheet".  
    - Edge Cases: Authentication errors, invalid email addresses, Gmail API limits.  
    - Version: 2.1

  - **Mark Email Sent in Sheet**  
    - Type: Google Sheets  
    - Role: Updates the original payroll Google Sheet row to mark the payslip email as sent (e.g., sets a boolean or date).  
    - Configuration: Uses Google Sheets credentials, identifies row by unique key or row number, updates the "email sent" column.  
    - Input: Confirmation from "Send Payslip Email".  
    - Output: Loops back to "Iterate Payslip Rows" to continue processing remaining rows.  
    - Edge Cases: Sheet write permission errors, row conflicts, update failures.  
    - Version: 4.7

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)              | Output Node(s)                   | Sticky Note        |
|-------------------------|---------------------|----------------------------------------|----------------------------|---------------------------------|--------------------|
| Manual Trigger          | Trigger             | Starts the workflow manually            | —                          | Company Configuration           |                    |
| Company Configuration   | Set                 | Sets company-specific variables         | Manual Trigger             | Fetch Payroll Data              |                    |
| Fetch Payroll Data      | Google Sheets       | Retrieves payroll data from Google Sheets| Company Configuration      | Check Email Not Sent            |                    |
| Check Email Not Sent    | Filter              | Filters rows where email not sent       | Fetch Payroll Data         | Iterate Payslip Rows            |                    |
| Iterate Payslip Rows    | Split In Batches    | Processes each payslip row individually | Check Email Not Sent       | Prepare Payslip Data (secondary output), empty (main output) |                    |
| Prepare Payslip Data    | Code                | Formats and prepares data for HTML      | Iterate Payslip Rows       | Generate Payslip HTML            |                    |
| Generate Payslip HTML   | HTML                | Generates HTML payslip content           | Prepare Payslip Data       | Generate Payslip PDF             |                    |
| Generate Payslip PDF    | Puppeteer           | Converts HTML to PDF                     | Generate Payslip HTML      | Create PDF File                 |                    |
| Create PDF File         | Convert To File     | Converts PDF buffer to file object       | Generate Payslip PDF       | Send Payslip Email              |                    |
| Send Payslip Email      | Gmail               | Sends PDF payslip email                   | Create PDF File            | Mark Email Sent in Sheet        |                    |
| Mark Email Sent in Sheet| Google Sheets       | Marks email as sent in sheet             | Send Payslip Email         | Iterate Payslip Rows            |                    |
| Sticky Note             | Sticky Note         | —                                        | —                          | —                               |                    |
| Sticky Note1            | Sticky Note         | —                                        | —                          | —                               |                    |
| Sticky Note2            | Sticky Note         | —                                        | —                          | —                               |                    |
| Sticky Note3            | Sticky Note         | —                                        | —                          | —                               |                    |
| Sticky Note4            | Sticky Note         | —                                        | —                          | —                               |                    |
| Sticky Note5            | Sticky Note         | —                                        | —                          | —                               |                    |
| Sticky Note6            | Sticky Note         | —                                        | —                          | —                               |                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually without input.  
   - No parameters needed.

2. **Create Company Configuration node**  
   - Type: Set  
   - Connect: Manual Trigger → Company Configuration  
   - Configure static variables such as company name, email subject templates, sender info.  
   - No dynamic inputs.

3. **Create Fetch Payroll Data node**  
   - Type: Google Sheets  
   - Connect: Company Configuration → Fetch Payroll Data  
   - Credentials: Google Sheets OAuth2 with read access.  
   - Parameters: Select spreadsheet and worksheet containing payroll data.  
   - Read mode: Read all rows or defined range containing payslip data.

4. **Create Check Email Not Sent node**  
   - Type: Filter  
   - Connect: Fetch Payroll Data → Check Email Not Sent  
   - Condition: Check if the "email sent" column is empty or false.  
   - Output true: rows to process; Output false: ignore.

5. **Create Iterate Payslip Rows node**  
   - Type: Split In Batches  
   - Connect: Check Email Not Sent → Iterate Payslip Rows  
   - Configure batch size (e.g., 1 to process one row at a time).  
   - Outputs: Secondary output to "Prepare Payslip Data".

6. **Create Prepare Payslip Data node**  
   - Type: Code (JavaScript)  
   - Connect: Iterate Payslip Rows (secondary output) → Prepare Payslip Data  
   - Write JavaScript to format data fields, calculate totals, format currency and dates.  
   - Input: Current row data.

7. **Create Generate Payslip HTML node**  
   - Type: HTML  
   - Connect: Prepare Payslip Data → Generate Payslip HTML  
   - Use an HTML template for payslip layout with placeholders for data fields.  
   - Replace placeholders with prepared data.

8. **Create Generate Payslip PDF node**  
   - Type: Puppeteer  
   - Connect: Generate Payslip HTML → Generate Payslip PDF  
   - Configure PDF options: page size (A4), margins, print background.  
   - Input: HTML content.  
   - Output: PDF buffer.

9. **Create Create PDF File node**  
   - Type: Convert To File  
   - Connect: Generate Payslip PDF → Create PDF File  
   - Configure filename (e.g., "Payslip_<EmployeeName>.pdf").  
   - Set mime-type as "application/pdf".

10. **Create Send Payslip Email node**  
    - Type: Gmail  
    - Connect: Create PDF File → Send Payslip Email  
    - Credentials: Gmail OAuth2 with send email scope.  
    - Parameters:  
      - To: Employee email from current data.  
      - Subject: e.g., "Your Payslip for [Month]"  
      - Body: Optional text or HTML including greeting and instructions.  
      - Attachments: Attach created PDF file.  
    - Ensure proper error handling for invalid addresses.

11. **Create Mark Email Sent in Sheet node**  
    - Type: Google Sheets  
    - Connect: Send Payslip Email → Mark Email Sent in Sheet  
    - Credentials: Google Sheets OAuth2 with write access.  
    - Parameters: Identify row by unique ID or row number.  
    - Update the "email sent" column to true or current date.  
    - Output: Connect back to Iterate Payslip Rows (main output) to continue processing next batch.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                      |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow requires Google Sheets API and Gmail API credentials to be configured properly. | Credential setup in n8n for Google Sheets and Gmail |
| Puppeteer node relies on headless Chromium for PDF generation; ensure environment supports it. | n8n Puppeteer node documentation                     |
| Email sending may be subject to Gmail sending limits; consider quotas when processing many payslips. | Gmail sending limits documentation                   |
| Use batching in Split In Batches node to avoid hitting API limits or memory issues.           | n8n SplitInBatches node docs                         |

---

_Disclaimer: The provided text is exclusively derived from an automated workflow designed with n8n, an integration and automation tool. This processing strictly adheres to existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible._