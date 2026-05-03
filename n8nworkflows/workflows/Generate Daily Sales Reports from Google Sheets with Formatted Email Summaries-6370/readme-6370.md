Generate Daily Sales Reports from Google Sheets with Formatted Email Summaries

https://n8nworkflows.xyz/workflows/generate-daily-sales-reports-from-google-sheets-with-formatted-email-summaries-6370


# Generate Daily Sales Reports from Google Sheets with Formatted Email Summaries

### 1. Workflow Overview

This workflow automates the generation and emailing of daily sales reports by reading data from a Google Sheet, formatting the data into a report, and sending it as an email summary. It is designed for sales teams or managers who want automated, formatted daily insights without manual compilation.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow execution.
- **1.2 Data Retrieval:** Reading sales data from a specified Google Sheet.
- **1.3 Data Validation:** Checking if the retrieved data exists and branching accordingly.
- **1.4 Report Formatting:** Processing and formatting the sales data into a readable report.
- **1.5 No Data Handling:** Defining an alternate path if no data is found.
- **1.6 Email Delivery:** Sending the formatted report or notification via email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow execution manually through user interaction.
- **Nodes Involved:**  
  - *When clicking ‘Execute workflow’*

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts workflow on user command, no automatic scheduling or external trigger.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Triggers the next node (*Read Google Sheet*).  
    - Edge Cases: None inherent; workflow won't run unless manually started.  
    - Notes: Useful for testing or ad-hoc report generation.

#### 2.2 Data Retrieval

- **Overview:** Reads raw sales data from a configured Google Sheet to be processed further.
- **Nodes Involved:**  
  - *Read Google Sheet*

- **Node Details:**

  - **Read Google Sheet**  
    - Type: Google Sheets node  
    - Role: Fetches sales data from a specified Google Sheet document.  
    - Configuration:  
      - Sheet ID or URL configured within node (not shown in JSON, assumed set).  
      - Operation: Likely 'Read Rows' or equivalent.  
      - Range: Not specified but implied to cover sales data range.  
    - Inputs: Triggered by manual node.  
    - Outputs: Outputs rows of data to *Check Data Exists* node.  
    - Edge Cases:  
      - Authentication errors if OAuth credentials expire or are invalid.  
      - Empty sheet or incorrect range returns no data.  
      - API quota limits could lead to failures or delays.  
    - Requirements: OAuth2 credentials for Google Sheets API must be configured correctly.

#### 2.3 Data Validation

- **Overview:** Checks if the Google Sheets node returned any data and routes workflow accordingly.
- **Nodes Involved:**  
  - *Check Data Exists*

- **Node Details:**

  - **Check Data Exists**  
    - Type: If node  
    - Role: Conditional branching based on presence of data.  
    - Configuration:  
      - Condition likely checks if incoming data length > 0.  
      - True branch: Data exists → proceed to formatting.  
      - False branch: No data → handle with alternate path.  
    - Inputs: Receives data from *Read Google Sheet*.  
    - Outputs:  
      - True: Connects to *Format Report*.  
      - False: Connects to *No Data Handler*.  
    - Edge Cases:  
      - Expression failure if data format unexpected.  
      - False positives if data contains empty rows but no meaningful data.

#### 2.4 Report Formatting

- **Overview:** Converts raw sales data into a human-readable formatted report string or HTML.
- **Nodes Involved:**  
  - *Format Report*

- **Node Details:**

  - **Format Report**  
    - Type: Code node (JavaScript)  
    - Role: Processes raw data into a formatted message or report.  
    - Configuration:  
      - Custom JavaScript code (not included in JSON) likely loops over data rows, formats sales figures, dates, totals, etc.  
      - Outputs are structured text or HTML to embed in email body.  
    - Inputs: True branch from *Check Data Exists* node.  
    - Outputs: Passes formatted report to *Send email* node.  
    - Edge Cases:  
      - Code errors if data schema changes.  
      - Unexpected data values causing formatting exceptions.  
    - Version: Uses Code node version 2 — standard for n8n for custom JS scripting.

#### 2.5 No Data Handling

- **Overview:** Creates a fallback message or report if no sales data is present.
- **Nodes Involved:**  
  - *No Data Handler*

- **Node Details:**

  - **No Data Handler**  
    - Type: Code node (JavaScript)  
    - Role: Generates a polite message indicating no data was found for the report period.  
    - Configuration:  
      - Custom JS code, likely returns a string such as "No sales data available for today."  
    - Inputs: False branch from *Check Data Exists* node.  
    - Outputs: Passes message to *Send email* node.  
    - Edge Cases: Minimal risk; straightforward fallback message generation.

#### 2.6 Email Delivery

- **Overview:** Sends the final report or notification email to the configured recipients.
- **Nodes Involved:**  
  - *Send email*

- **Node Details:**

  - **Send email**  
    - Type: Email Send node  
    - Role: Dispatches the formatted report or no-data notification via email.  
    - Configuration:  
      - SMTP or email service credentials configured (not shown in JSON).  
      - Subject and recipient fields set appropriately (likely dynamic to include date or report title).  
      - Email body populated with output from either *Format Report* or *No Data Handler*.  
    - Inputs: Receives formatted message or fallback message.  
    - Outputs: None (terminal node).  
    - Edge Cases:  
      - SMTP authentication failures.  
      - Email server timeouts or connectivity issues.  
      - Invalid recipient addresses causing delivery failure.

---

### 3. Summary Table

| Node Name                | Node Type                | Functional Role              | Input Node(s)                | Output Node(s)          | Sticky Note |
|--------------------------|--------------------------|-----------------------------|-----------------------------|------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Workflow initiation          | None                        | Read Google Sheet       |             |
| Read Google Sheet        | Google Sheets            | Data retrieval from spreadsheet | When clicking ‘Execute workflow’ | Check Data Exists       |             |
| Check Data Exists        | If                       | Conditional data check       | Read Google Sheet            | Format Report, No Data Handler |             |
| Format Report            | Code                     | Format and compose report    | Check Data Exists (true)     | Send email              |             |
| No Data Handler          | Code                     | Generate fallback message    | Check Data Exists (false)    | Send email              |             |
| Send email               | Email Send               | Dispatch report email        | Format Report, No Data Handler | None                   |             |
| Sticky Note              | Sticky Note              | -                           | -                           | -                      |             |
| Sticky Note1             | Sticky Note              | -                           | -                           | -                      |             |
| Sticky Note2             | Sticky Note              | -                           | -                           | -                      |             |
| Sticky Note3             | Sticky Note              | -                           | -                           | -                      |             |
| Sticky Note4             | Sticky Note              | -                           | -                           | -                      |             |
| Sticky Note5             | Sticky Note              | -                           | -                           | -                      |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Add a *Manual Trigger* node.
   - Name it “When clicking ‘Execute workflow’”.
   - No parameters needed.

2. **Add Google Sheets Node**
   - Add a *Google Sheets* node.
   - Name it “Read Google Sheet”.
   - Configure credentials with valid Google OAuth2.
   - Set operation to ‘Read Rows’.
   - Specify the Google Sheet ID and the data range covering the sales data.
   - Connect output of *When clicking ‘Execute workflow’* to this node’s input.

3. **Add If Node for Data Check**
   - Add an *If* node.
   - Name it “Check Data Exists”.
   - Set condition: check if the length of the incoming data array > 0 (e.g., expression like `{{$json["length"] > 0}}` or use n8n’s data existence check).
   - Connect output of *Read Google Sheet* to this node.

4. **Add Code Node to Format Report**
   - Add a *Code* node.
   - Name it “Format Report”.
   - Insert JavaScript to iterate over input data and format it into a readable report string or HTML.
   - Connect the true output branch of *Check Data Exists* to this node.

5. **Add Code Node for No Data**
   - Add another *Code* node.
   - Name it “No Data Handler”.
   - Insert JavaScript that returns a simple message such as “No sales data available for today.”.
   - Connect the false output branch of *Check Data Exists* to this node.

6. **Add Email Send Node**
   - Add an *Email Send* node.
   - Name it “Send email”.
   - Configure SMTP or your preferred email credentials.
   - Set recipient(s), subject (e.g., “Daily Sales Report for {{date}}”).
   - Set the email body to use the output from the *Format Report* or *No Data Handler* nodes.
   - Connect outputs of both *Format Report* and *No Data Handler* nodes to this node.

7. **Validate Workflow**
   - Confirm all nodes are connected as per the flow:
     - Manual Trigger → Read Google Sheet → Check Data Exists → (True) Format Report → Send email
     - Check Data Exists → (False) No Data Handler → Send email
   - Test manually to verify data retrieval, formatting, and email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                            |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------|
| This workflow requires valid Google Sheets API OAuth2 credentials configured in n8n for data retrieval.  | Google Sheets API documentation            |
| Email sending requires SMTP or other email service credentials properly configured in n8n.               | n8n Email Node documentation               |
| Manual triggering is suitable for ad-hoc reports; for automation, consider adding a Cron node trigger.   | n8n Cron Trigger node                       |
| Ensure the Google Sheets data range covers all relevant daily sales data to avoid empty reports.         | Google Sheets data range setup              |
| Code nodes require JavaScript familiarity to customize formatting and error handling.                     | n8n Code node documentation                 |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, an integration and automation tool. The process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.