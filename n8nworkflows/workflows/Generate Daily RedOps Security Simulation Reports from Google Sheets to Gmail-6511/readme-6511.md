Generate Daily RedOps Security Simulation Reports from Google Sheets to Gmail

https://n8nworkflows.xyz/workflows/generate-daily-redops-security-simulation-reports-from-google-sheets-to-gmail-6511


# Generate Daily RedOps Security Simulation Reports from Google Sheets to Gmail

### 1. Workflow Overview

This workflow automates the generation and sending of daily RedOps security simulation reports by extracting data from a Google Sheets document and emailing a formatted HTML report via Gmail. It is tailored for security teams or analysts who run RedOps (Red Team Operations) simulations and need a daily digest of trap logs for review and action.

The workflow is organized into these logical blocks:

- **1.1 Trigger and Data Retrieval**: Initiates the workflow manually and reads trap log data from a Google Sheets spreadsheet.
- **1.2 Data Formatting**: Processes and formats the raw spreadsheet data into structured report data.
- **1.3 Report Construction**: Builds an HTML email summary from the formatted data.
- **1.4 Email Dispatch**: Sends the constructed report to recipients using Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Data Retrieval

- **Overview:**  
  This block starts the workflow manually and fetches raw trap log data from Google Sheets.  
  It serves as the data input point and ensures the workflow runs only when triggered.

- **Nodes Involved:**  
  - 🕒 Trigger AutoReport  
  - 📄 Read Trap Log Sheet

- **Node Details:**

  - **🕒 Trigger AutoReport**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on-demand. No scheduled or webhook triggers configured.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None (entry point)  
    - Outputs: Connects to "📄 Read Trap Log Sheet"  
    - Edge Cases: No failure expected unless manual intervention fails or node disabled.  
    - Version: v1 (standard manual trigger)

  - **📄 Read Trap Log Sheet**  
    - Type: Google Sheets node  
    - Role: Reads trap log data from a configured Google Sheets spreadsheet.  
    - Configuration:  
      - Reads rows from a specific sheet (exact sheet name or range not disclosed here).  
      - Authentication via Google Sheets credentials (OAuth2).  
    - Inputs: From manual trigger  
    - Outputs: Raw spreadsheet rows to "📊 Format Report Data"  
    - Edge Cases:  
      - Authentication failure if credentials expire or revoked.  
      - Empty or malformed sheet data could lead to errors downstream.  
      - API rate limits or connectivity issues with Google services.  
    - Version: 4.6

#### 2.2 Data Formatting

- **Overview:**  
  This block transforms the raw spreadsheet data into a structured format suitable for reporting. It likely cleans, filters, and aggregates data points.

- **Nodes Involved:**  
  - 📊 Format Report Data

- **Node Details:**

  - **📊 Format Report Data**  
    - Type: Code node (JavaScript)  
    - Role: Processes input data to create an organized dataset for the report.  
    - Configuration: Contains custom JavaScript logic for data manipulation.  
    - Expressions/Variables: Reads input from "📄 Read Trap Log Sheet", outputs formatted JSON data.  
    - Inputs: Raw data rows from Google Sheets  
    - Outputs: Structured data to "📑 Build HTML Report Summary"  
    - Edge Cases:  
      - Script errors if input data schema changes unexpectedly.  
      - Null or missing fields in input data potentially causing runtime exceptions.  
    - Version: 2

#### 2.3 Report Construction

- **Overview:**  
  Builds an HTML formatted report summary from the structured data, preparing it for email delivery.

- **Nodes Involved:**  
  - 📑 Build HTML Report Summary

- **Node Details:**

  - **📑 Build HTML Report Summary**  
    - Type: Code node (JavaScript)  
    - Role: Generates an HTML email body summarizing the RedOps trap log report.  
    - Configuration: Custom JavaScript to create HTML, possibly including tables, headings, and styles.  
    - Expressions/Variables: Uses formatted data from "📊 Format Report Data" as input.  
    - Inputs: Formatted report data  
    - Outputs: HTML content to "📧 Send Summary Email"  
    - Edge Cases:  
      - If input data is empty or malformed, HTML output may be incomplete or broken.  
      - Script errors or encoding issues could affect email rendering.  
    - Version: 2

#### 2.4 Email Dispatch

- **Overview:**  
  Sends the constructed HTML report email using Gmail integration.

- **Nodes Involved:**  
  - 📧 Send Summary Email

- **Node Details:**

  - **📧 Send Summary Email**  
    - Type: Gmail node  
    - Role: Sends the daily RedOps report email with the generated HTML content.  
    - Configuration:  
      - Uses OAuth2 credentials for Gmail account access.  
      - Email parameters include recipients, subject line, and HTML body content from previous node.  
    - Inputs: HTML email content from "📑 Build HTML Report Summary"  
    - Outputs: None (end node)  
    - Edge Cases:  
      - Authentication failure due to expired or revoked Gmail credentials.  
      - Sending limits or quota exceeded on Gmail account.  
      - Email content formatting issues leading to display errors in recipients’ mail clients.  
    - Version: 2.1

---

### 3. Summary Table

| Node Name                | Node Type              | Functional Role                 | Input Node(s)         | Output Node(s)              | Sticky Note |
|--------------------------|------------------------|--------------------------------|-----------------------|----------------------------|-------------|
| Sticky Note              | Sticky Note            | (Visual annotation)             |                       |                            |             |
| 🕒 Trigger AutoReport    | Manual Trigger         | Manual start of workflow        |                       | 📄 Read Trap Log Sheet      |             |
| 📄 Read Trap Log Sheet   | Google Sheets          | Reads raw trap log data         | 🕒 Trigger AutoReport  | 📊 Format Report Data       |             |
| 📊 Format Report Data    | Code                   | Formats raw sheet data          | 📄 Read Trap Log Sheet | 📑 Build HTML Report Summary|             |
| 📑 Build HTML Report Summary | Code               | Builds HTML email report        | 📊 Format Report Data  | 📧 Send Summary Email       |             |
| 📧 Send Summary Email    | Gmail                  | Sends report email              | 📑 Build HTML Report Summary |                        |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named "🕒 Trigger AutoReport".  
   - Leave default settings. This node starts the workflow manually.

2. **Add Google Sheets Node**  
   - Add a Google Sheets node named "📄 Read Trap Log Sheet".  
   - Configure credentials with OAuth2 Google Sheets access.  
   - Set operation to "Read Rows".  
   - Select the spreadsheet and sheet containing the trap log data.  
   - Connect output from the manual trigger node to this node.

3. **Add Code Node to Format Data**  
   - Add a Code node named "📊 Format Report Data".  
   - Set language to JavaScript.  
   - Write or paste a script to process the input data from Google Sheets:  
     - Parse rows, clean or filter data as needed.  
     - Prepare structured JSON output suitable for report generation.  
   - Connect output from the Google Sheets node to this code node.

4. **Add Code Node to Build HTML Report**  
   - Add a Code node named "📑 Build HTML Report Summary".  
   - Set language to JavaScript.  
   - Use a script to generate an HTML string that formats the report data into an email-friendly layout (tables, styles, headers).  
   - Connect output from the data formatting node to this node.

5. **Add Gmail Node to Send Email**  
   - Add a Gmail node named "📧 Send Summary Email".  
   - Configure Gmail OAuth2 credentials.  
   - Set the "To" field with recipient email addresses.  
   - Set the "Subject" field (e.g., "Daily RedOps Security Simulation Report").  
   - Map the HTML content output from the previous node to the email body with HTML enabled.  
   - Connect output from the HTML report node to this Gmail node.

6. **Verify Flow**  
   - Ensure connection chain is: Manual Trigger → Google Sheets → Format Data → Build HTML → Send Email.  
   - Save and activate workflow.  
   - Test manually triggering the workflow to verify email receipt and content correctness.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                         |
|----------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow is part of the “Pro” tagged RedOps automation modules for daily security reports. | Workflow tags metadata                                  |
| Gmail integration requires OAuth2 credentials with permissions to send emails on behalf of user. | Gmail OAuth2 setup in n8n credentials                   |
| Google Sheets node needs valid OAuth2 credentials and correct spreadsheet permissions.       | Google Sheets API and OAuth2 setup                      |
| No scheduling trigger included; manual trigger used for on-demand report generation.         | Could be enhanced by adding Cron or Schedule trigger.  |

---

Disclaimer: The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.