Transfer data from website to Google Sheets

https://n8nworkflows.xyz/workflows/transfer-data-from-website-to-google-sheets-1076


# Transfer data from website to Google Sheets

### 1. Workflow Overview

This workflow automates the process of capturing data submitted through a website form and saving it directly into a specified Google Sheets document. It is designed for use cases where form responses collected via a webhook endpoint need to be logged or archived systematically in a Google Sheets spreadsheet for further analysis or record-keeping.

Logical blocks:

- **1.1 Input Reception:** Receives data sent from a website form via a webhook.
- **1.2 Data Storage:** Appends the received data as new rows into a specified Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP POST requests from the website form and captures all submitted form data.
  
- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - **Type and Role:** HTTP Webhook node; serves as an entry point for external data into the workflow.  
    - **Configuration:**  
      - Path set to `/webhook` (i.e., the URL endpoint ends with `/webhook`).  
      - Response mode set to return all entries collected in this execution (`responseData: allEntries`).  
      - Response sent from the last node in the workflow.  
    - **Key Expressions/Variables:** None explicitly used; accepts raw incoming data.  
    - **Input/Output:**  
      - Input: External HTTP requests (e.g., from a website form submission).  
      - Output: Passes the received form data downstream to the Google Sheets node.  
    - **Version Requirements:** Compatible with n8n version 0.100.0+ (standard webhook functionality).  
    - **Edge Cases / Potential Failures:**  
      - Malformed or unexpected form data may cause downstream nodes to fail or append incorrect data.  
      - If the webhook URL is not publicly accessible, external form submissions will fail.  
      - High volume of requests may cause throttling or delays.  
    - **Sub-Workflow:** None.

#### 1.2 Data Storage

- **Overview:**  
  This block takes the data received from the webhook and appends it as new rows into a specific Google Sheets spreadsheet tab ("Problems!A:D").

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Google Sheets**  
    - **Type and Role:** Google Sheets node; appends or updates data in a Google Sheets spreadsheet.  
    - **Configuration:**  
      - Spreadsheet ID: `17fzSFl1BZ1njldTfp5lvh8HtS0-pNXH66b7qGZIiGRU` (unique identifier of the target Google Sheets document).  
      - Sheet tab and range: `Problems!A:D` (columns A to D on the "Problems" sheet).  
      - Options: Default (no special options configured).  
    - **Key Expressions/Variables:** None explicitly used; the node automatically maps incoming JSON data fields to sheet columns (A–D).  
    - **Input/Output:**  
      - Input: Receives form data from the Webhook node.  
      - Output: Sends data downstream (not connected further in this workflow).  
    - **Version Requirements:** Requires valid Google API credentials configured in n8n.  
    - **Edge Cases / Potential Failures:**  
      - Authentication errors if Google API credentials are missing, expired, or misconfigured.  
      - Range conflicts if data structure does not match the expected columns or exceeds the range.  
      - API quota limits or network issues could cause failures or delays.  
    - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name      | Node Type             | Functional Role          | Input Node(s) | Output Node(s) | Sticky Note                         |
|----------------|-----------------------|-------------------------|---------------|----------------|-----------------------------------|
| Webhook        | n8n-nodes-base.webhook | Input Reception         |               | Google Sheets  |                                   |
| Google Sheets  | n8n-nodes-base.googleSheets | Data Storage           | Webhook       |                |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Type: Webhook  
   - Name: `Webhook`  
   - Set the HTTP path to `webhook` (this defines the webhook URL endpoint).  
   - Response Mode: `Last Node` (to respond with the output of the last node).  
   - Response Data: `All Entries` (to return all data received during execution).  
   - Leave other options at defaults.  

2. **Create a Google Sheets node**  
   - Type: Google Sheets  
   - Name: `Google Sheets`  
   - Set the operation to append data (default behavior when range is specified).  
   - Enter the Sheet ID: `17fzSFl1BZ1njldTfp5lvh8HtS0-pNXH66b7qGZIiGRU` (this must be replaced with your target Google Sheets document ID).  
   - Set the Range to `Problems!A:D` (targeting columns A to D on the "Problems" sheet).  
   - Ensure authentication credentials are set:  
     - Configure Google API OAuth2 credentials in n8n (via Credentials section).  
   - Leave other options as default.  

3. **Connect nodes**  
   - Link the output of the `Webhook` node to the input of the `Google Sheets` node.  

4. **Activate the webhook workflow**  
   - Deploy the workflow and note the full webhook URL (e.g., `https://[your-n8n-domain]/webhook`).  
   - Configure the website form to send data via HTTP POST to this webhook URL.  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow requires valid Google API credentials with permission to edit the specified sheet. | n8n Credential setup documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.google-sheets/ |
| The webhook URL must be publicly accessible from the website to receive form data correctly.   | Ensure n8n is hosted with public access or use tunneling tools like ngrok during development.  |
| Google Sheets API quota limits may affect high-frequency data submissions.                      | Google API quotas: https://developers.google.com/sheets/api/limits                             |

---

This document fully describes the workflow's structure, node configurations, and step-by-step recreation instructions for advanced users or AI agents.