Generate & Distribute Market Reports with Google Docs, Sheets, and Gmail

https://n8nworkflows.xyz/workflows/generate---distribute-market-reports-with-google-docs--sheets--and-gmail-6429


# Generate & Distribute Market Reports with Google Docs, Sheets, and Gmail

---

### 1. Workflow Overview

This workflow automates the monthly generation and distribution of market reports using Google Docs, Google Sheets, and Gmail. Designed for marketing or sales teams, it fetches market data, processes it into a structured report, and emails the finalized report to a client mailing list. The workflow consists of these logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow monthly.
- **1.2 Market Data Retrieval:** Obtains raw market data from an external API.
- **1.3 Data Processing:** Transforms raw data into a formatted structure suitable for reporting.
- **1.4 Report Generation:** Creates a Google Docs report based on a template populated with processed data.
- **1.5 Client List Acquisition:** Retrieves client email addresses from a Google Sheets mailing list.
- **1.6 Report Distribution:** Sends the generated report as an attachment to each client individually.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically on a monthly schedule, specifically on the first day of every month at 8 AM.

- **Nodes Involved:**  
  - 0. Cron (Monthly Schedule)

- **Node Details:**  
  - **Node:** 0. Cron (Monthly Schedule)  
    - Type: Cron Trigger  
    - Configuration: Set to execute at 8:00 AM on day 1 of every month.  
    - Expressions: None required; fixed schedule.  
    - Input: None (trigger node)  
    - Output: Triggers the HTTP Request node to start data retrieval.  
    - Version: v1  
    - Edge Cases: Workflow will not run if n8n server is offline at scheduled time. No retry mechanism within this node.  
    - Notes: Reliable scheduling depends on server uptime.

#### 1.2 Market Data Retrieval

- **Overview:**  
  Fetches raw market data from an external API for a specific region, providing the foundational data for the report.

- **Nodes Involved:**  
  - 1. HTTP Request (Get Market Data)

- **Node Details:**  
  - **Node:** 1. HTTP Request (Get Market Data)  
    - Type: HTTP Request  
    - Configuration: Configured to call an external market data API endpoint (URL and authentication details are assumed set in parameters).  
    - Expressions: May use expressions to dynamically set API parameters such as region or date, though none explicitly shown.  
    - Input: Triggered by Cron node.  
    - Output: JSON or other structured data representing raw market data.  
    - Version: v1  
    - Edge Cases:  
      - API auth failure (invalid token, expired credentials)  
      - Network timeouts or connectivity issues  
      - Unexpected API response format or errors  
    - Error Handling: Should be monitored or wrapped with error workflows for retry or alert.

#### 1.3 Data Processing

- **Overview:**  
  Processes and formats the raw data into a structured format suitable for filling into the report template.

- **Nodes Involved:**  
  - 2. Function (Process Data)

- **Node Details:**  
  - **Node:** 2. Function (Process Data)  
    - Type: Function (JavaScript code execution)  
    - Configuration: Custom JavaScript code manipulates the raw API data, e.g., filtering, summarizing, or reformatting.  
    - Expressions: Uses input data from the HTTP Request node and outputs formatted data objects.  
    - Input: JSON data from the HTTP Request node.  
    - Output: Structured data ready for report injection.  
    - Version: v1  
    - Edge Cases:  
      - Code errors or unexpected data structures causing runtime exceptions  
      - Null or malformed input data  
    - Error Handling: Add try/catch in code to handle errors gracefully.

#### 1.4 Report Generation

- **Overview:**  
  Creates a Google Docs document from a predefined template, populating it with the processed market data to generate the monthly report.

- **Nodes Involved:**  
  - 3. Google Docs (Create Report)

- **Node Details:**  
  - **Node:** 3. Google Docs (Create Report)  
    - Type: Google Docs node  
    - Configuration:  
      - Uses a Google Docs template document ID.  
      - Inserts processed data into placeholders or specific fields in the document.  
      - Creates a new report document for the current month.  
    - Expressions: Likely uses data variables from the Function node output for replacement.  
    - Input: Structured data from the Function node.  
    - Output: Document metadata including document ID and URL.  
    - Version: v1  
    - Credentials: Requires Google Docs OAuth2 credentials with edit permissions.  
    - Edge Cases:  
      - Template document missing or inaccessible  
      - Insufficient permissions  
      - API quota limits or rate limiting  
    - Error Handling: Verify credentials before execution; consider retries on failures.

#### 1.5 Client List Acquisition

- **Overview:**  
  Retrieves the list of client email addresses from a Google Sheets spreadsheet used as a mailing list.

- **Nodes Involved:**  
  - 4. Google Sheets (Get Client List)

- **Node Details:**  
  - **Node:** 4. Google Sheets (Get Client List)  
    - Type: Google Sheets node  
    - Configuration:  
      - Reads a specific sheet and range containing client emails.  
      - Fetches all rows to obtain a complete client list.  
    - Expressions: None explicitly; static sheet and range configured.  
    - Input: Output from Google Docs node (after report creation).  
    - Output: Array of client email addresses.  
    - Version: v3  
    - Credentials: Google Sheets OAuth2 credentials with read permissions.  
    - Edge Cases:  
      - Sheet or range not found  
      - Empty or malformed data  
      - Permission issues  
    - Error Handling: Validate sheet access and data format before sending emails.

#### 1.6 Report Distribution

- **Overview:**  
  Sends the generated report as an email attachment to each client individually by processing the client list one at a time.

- **Nodes Involved:**  
  - 5. Split In Batches  
  - 6. Gmail (Send Report)

- **Node Details:**  
  - **Node:** 5. Split In Batches  
    - Type: Split In Batches  
    - Configuration: Splits the array of client emails to process one email per batch (batch size = 1).  
    - Input: Client list array from Google Sheets node.  
    - Output: Single client email per execution to Gmail node.  
    - Version: v1  
    - Edge Cases:  
      - Empty list results in no emails sent  
      - Large lists might require execution time considerations  
    - Notes: Facilitates controlled sending and throttling.

  - **Node:** 6. Gmail (Send Report)  
    - Type: Gmail Node  
    - Configuration:  
      - Sends an email with the generated Google Docs report attached (possibly converted to PDF).  
      - Recipient email is set dynamically from the current batch item.  
      - Email subject and body can include expressions referencing report metadata or date.  
    - Input: Single client email from Split In Batches node; report document info from previous nodes.  
    - Output: Email send status.  
    - Version: v1  
    - Credentials: Gmail OAuth2 with send email permission.  
    - Edge Cases:  
      - Gmail API quota exceeded  
      - Invalid email addresses causing send failures  
      - Attachment size limits  
    - Error Handling: Monitor send failures and handle retries or logging accordingly.

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                  | Input Node(s)               | Output Node(s)             | Sticky Note                                      |
|-------------------------------|--------------------------|--------------------------------|-----------------------------|-----------------------------|-------------------------------------------------|
| 0. Cron (Monthly Schedule)     | Cron Trigger             | Scheduled monthly trigger       | None                        | 1. HTTP Request (Get Market Data) |                                                 |
| 1. HTTP Request (Get Market Data) | HTTP Request           | Fetch market data from API      | 0. Cron                     | 2. Function (Process Data)  |                                                 |
| 2. Function (Process Data)     | Function                 | Process and format raw data     | 1. HTTP Request             | 3. Google Docs (Create Report) |                                                 |
| 3. Google Docs (Create Report) | Google Docs              | Generate report from template   | 2. Function                 | 4. Google Sheets (Get Client List) |                                                 |
| 4. Google Sheets (Get Client List) | Google Sheets          | Retrieve client emails          | 3. Google Docs              | 5. Split In Batches         |                                                 |
| 5. Split In Batches            | Split In Batches          | Process emails one by one       | 4. Google Sheets            | 6. Gmail (Send Report)      |                                                 |
| 6. Gmail (Send Report)         | Gmail                    | Send report email with attachment | 5. Split In Batches          | None                        |                                                 |
| Sticky Note                   | Sticky Note              | -                              | -                           | -                           |                                                 |
| Sticky Note1                  | Sticky Note              | -                              | -                           | -                           |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Type: Cron  
   - Name: "0. Cron (Monthly Schedule)"  
   - Schedule: Set to trigger at 8:00 AM on the 1st day of every month  
   - No additional parameters needed.  

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "1. HTTP Request (Get Market Data)"  
   - Configure method (likely GET) and URL to the market data API endpoint.  
   - Set authentication as required (e.g., API key in headers or query).  
   - Connect input from Cron node.  

3. **Create Function Node**  
   - Type: Function  
   - Name: "2. Function (Process Data)"  
   - Write JavaScript code to:  
     - Parse raw API data  
     - Format or summarize data as needed for report insertion  
   - Connect input from HTTP Request node.  

4. **Create Google Docs Node**  
   - Type: Google Docs  
   - Name: "3. Google Docs (Create Report)"  
   - Configure:  
     - Provide template Document ID  
     - Map processed data fields to template placeholders  
   - Ensure Google OAuth2 credentials with edit access are configured.  
   - Connect input from Function node.  

5. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Name: "4. Google Sheets (Get Client List)"  
   - Configure spreadsheet ID and sheet name or range containing client emails  
   - Set read operation to fetch all relevant rows  
   - Ensure Google OAuth2 credentials with read access are set.  
   - Connect input from Google Docs node.  

6. **Create Split In Batches Node**  
   - Type: Split In Batches  
   - Name: "5. Split In Batches"  
   - Set batch size to 1 (to process one client email at a time)  
   - Connect input from Google Sheets node.  

7. **Create Gmail Node**  
   - Type: Gmail  
   - Name: "6. Gmail (Send Report)"  
   - Configure:  
     - Recipient email: Use the current batch item email address (expression)  
     - Subject and body: Include dynamic content if desired (e.g., report month)  
     - Attachment: Include the generated Google Docs report (converted to PDF if needed)  
   - Configure Gmail OAuth2 credentials with send permissions.  
   - Connect input from Split In Batches node.  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                              |
|-------------------------------------------------------------------------------------------------|---------------------------------------------|
| The workflow triggers monthly to generate and send timely market reports automatically.         | Workflow purpose                             |
| Requires proper OAuth2 credentials setup for Google Docs, Google Sheets, and Gmail nodes.       | Credential configuration                     |
| Ensure API keys or authentication for the external Market Data API are securely stored.         | API integration security                      |
| Handling errors for API calls, document generation, and email sending is critical for reliability.| Suggested best practice                      |
| To attach Google Docs documents via Gmail, conversion to PDF format is recommended for compatibility. | Gmail attachment handling                     |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---