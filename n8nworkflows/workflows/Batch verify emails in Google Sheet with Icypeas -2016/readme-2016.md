Batch verify emails in Google Sheet with Icypeas 

https://n8nworkflows.xyz/workflows/batch-verify-emails-in-google-sheet-with-icypeas--2016


# Batch verify emails in Google Sheet with Icypeas 

### 1. Workflow Overview

This workflow enables batch verification of email addresses stored in a Google Sheet using the Icypeas email verification service. It is designed for users who want to validate multiple email addresses in bulk to improve data quality or for marketing/email campaign preparation.

**Target Use Cases:**  
- Bulk email validation from Google Sheets  
- Integration with Icypeas API for email verification  
- Automating email verification workflows without manual API calls  

**Logical Blocks:**  
- **1.1 Manual Trigger:** Starts the workflow manually on user action.  
- **1.2 Google Sheets Data Retrieval:** Reads the email addresses from a specified Google Sheet.  
- **1.3 Data Preparation & Authentication:** Converts the sheet data into the required format and generates authentication signatures for Icypeas API.  
- **1.4 Email Verification Request:** Sends a POST request to Icypeas to perform the bulk email verification.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

**Overview:**  
This block initiates the workflow operation when the user manually clicks 'Execute Workflow' in n8n.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)

**Node Details:**  
- **Node Type:** Manual Trigger  
- **Configuration:** Default manual trigger, no parameters.  
- **Expressions/Variables:** None  
- **Inputs:** None (entry point)  
- **Outputs:** Connected to Google Sheets reading node  
- **Version Requirements:** Compatible with n8n v1.x and above  
- **Potential Failures:** None (manual trigger)  
- **Sub-workflows:** None  

---

#### 1.2 Google Sheets Data Retrieval

**Overview:**  
Reads the user’s Google Sheet containing email addresses to be verified. The sheet must have a header column named "email".

**Nodes Involved:**  
- Reads lastname,firstname and company from your sheet (Google Sheets node)

**Node Details:**  
- **Node Type:** Google Sheets  
- **Configuration:**  
  - Spreadsheet Document ID and Sheet Name are set via expressions or static values.  
  - Reads the entire sheet or specific range of rows.  
  - Expects the first column header as "email".  
- **Expressions/Variables:** Uses documentId and sheetName configured via list mode or expressions.  
- **Inputs:** Connected from Manual Trigger  
- **Outputs:** Passes data to Icypeas authentication node  
- **Version Requirements:** Google Sheets node v4.1+ recommended for latest API support  
- **Potential Failures:**  
  - Authentication issues with Google API credentials  
  - Incorrect document ID or sheet name  
  - Malformed or missing "email" column  
- **Sub-workflows:** None  

---

#### 1.3 Data Preparation & Authentication

**Overview:**  
Transforms the sheet data into an array of emails for Icypeas API and generates a required HMAC SHA1 signature for authentication.

**Nodes Involved:**  
- Authenticates to your Icypeas account (Code node)

**Node Details:**  
- **Node Type:** Code (JavaScript)  
- **Configuration:**  
  - API credentials placeholders for API_KEY, API_SECRET, and USER_ID must be replaced by the user.  
  - Constructs the API URL and HTTP method for the bulk search endpoint.  
  - Extracts emails from the input data and formats them as an array of arrays `[ [email1], [email2], ... ]`.  
  - Generates a timestamp and HMAC SHA1 signature for authentication using Node.js crypto module.  
  - Injects authentication info and formatted data into the JSON output for the next node.  
- **Expressions/Variables:**  
  - Uses `$input.all()` and `$input.first()` to access incoming data.  
  - Variables: API_KEY, API_SECRET, USER_ID, timestamp, signature  
- **Inputs:** Connected from Google Sheets node  
- **Outputs:** Passes authentication info and data to HTTP Request node  
- **Version Requirements:**  
  - Requires Node.js crypto module enabled in n8n instance (self-hosted users must enable via settings).  
- **Potential Failures:**  
  - Missing or incorrect API credentials  
  - Crypto module not available or disabled  
  - Incorrect data formatting causing signature errors  
- **Sub-workflows:** None  

---

#### 1.4 Email Verification Request

**Overview:**  
Sends an authenticated POST HTTP request to Icypeas API to perform bulk email verification with the prepared data.

**Nodes Involved:**  
- Run bulk search (email-verif) (HTTP Request node)

**Node Details:**  
- **Node Type:** HTTP Request  
- **Configuration:**  
  - URL is dynamically set using expression from previous node `$json.api.url`.  
  - Method: POST  
  - Body parameters include:  
    - task = "email-verification"  
    - name = a static string (e.g., "dernierTsfg") - can be customized  
    - user = user ID from auth node  
    - data = array of emails  
  - Header Authentication using custom header "Authorization" with value as an expression combining the API key and signature `${key}:${signature}`.  
  - Additional header "X-ROCK-TIMESTAMP" set with the generated timestamp.  
  - Requires creation of HTTP Header Auth credentials in n8n with name "Authorization" and value set by expression.  
- **Expressions/Variables:**  
  - `{{ $json.api.key + ':' + $json.api.signature }}` for Authorization header value  
  - `{{ $json.api.timestamp }}` for timestamp header  
  - `{{ $json.api.userId }}` and `{{ $json.data }}` for body parameters  
- **Inputs:** Connected from Code node (auth/data preparation)  
- **Outputs:** Final output of the workflow; response contains status, but actual verification results are downloadable later from Icypeas dashboard or emailed.  
- **Version Requirements:** HTTP Request node v4.1+ recommended  
- **Potential Failures:**  
  - Network errors/timeouts  
  - Invalid API credentials or malformed signature leading to 401 Unauthorized  
  - API rate limiting or quota exceeded  
  - Incorrect body parameter formatting  
- **Sub-workflows:** None  

---

### 3. Summary Table

| Node Name                               | Node Type       | Functional Role                          | Input Node(s)                         | Output Node(s)                       | Sticky Note                                                                                               |
|----------------------------------------|-----------------|----------------------------------------|-------------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow"       | Manual Trigger  | Starts the workflow manually           | None                                | Reads lastname,firstname and company from your sheet |                                                                                                          |
| Reads lastname,firstname and company from your sheet | Google Sheets   | Reads email addresses from Google Sheet | When clicking "Execute Workflow"    | Authenticates to your Icypeas account | Sticky Note1: Describes sheet setup - must have "email" header; specify file path and credentials.       |
| Authenticates to your Icypeas account  | Code            | Prepares data array and generates auth signature | Reads lastname,firstname and company from your sheet | Run bulk search (email-verif)       | Sticky Note3: Instructions to insert API credentials, enable crypto module for self-hosted n8n.          |
| Run bulk search (email-verif)           | HTTP Request    | Sends POST request to Icypeas API      | Authenticates to your Icypeas account | None                               | Sticky Note4: Explains HTTP auth setup, how to create credentials, and where to find results in Icypeas.  |
| Sticky Note                            | Sticky Note     | Informational note about batch email verification with Icypeas | None                                | None                               | Sticky Note: Overview and link to Icypeas website                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a "Manual Trigger" node named `When clicking "Execute Workflow"`.  
   - No configuration required.

2. **Create Google Sheets Node:**  
   - Add "Google Sheets" node named `Reads lastname,firstname and company from your sheet`.  
   - Configure Google Sheets credentials with access to your Google Drive.  
   - Set "Document ID" to your Google Sheet ID containing emails.  
   - Set "Sheet Name" to the specific sheet/tab name.  
   - Ensure your sheet has a first column header named "email".  
   - Connect this node's input to the Manual Trigger node output.

3. **Create Code Node for Authentication and Data Preparation:**  
   - Add a "Code" node named `Authenticates to your Icypeas account`.  
   - Paste the provided JavaScript code that:  
     - Defines API base URL, path, method.  
     - Requires user to replace placeholders `PUT_API_KEY_HERE`, `PUT_API_SECRET_HERE`, and `PUT_USER_ID_HERE` with actual Icypeas credentials.  
     - Extracts emails from input data and formats for API.  
     - Generates HMAC SHA1 signature using the Node.js crypto module.  
   - Enable the crypto module in your n8n instance if self-hosted (via Settings > General > Additional Node Packages).  
   - Connect this node output to be the next step after Google Sheets node.

4. **Create HTTP Request Node for Bulk Email Verification:**  
   - Add an "HTTP Request" node named `Run bulk search (email-verif)`.  
   - Set method to "POST".  
   - Set URL to expression: `{{ $json.api.url }}` to use the dynamically created API URL.  
   - Under "Body Parameters", add parameters:  
     - task = "email-verification"  
     - name = any identifier string (e.g., "dernierTsfg")  
     - user = `{{ $json.api.userId }}`  
     - data = `{{ $json.data }}` (the array of emails)  
   - Under "Authentication", choose "Generic Credential Type".  
   - Create new HTTP Header Auth credential:  
     - Name: "Authorization"  
     - Value: Expression: `{{ $json.api.key + ':' + $json.api.signature }}`  
     - Save credential and assign it to this node.  
   - Add custom header "X-ROCK-TIMESTAMP" with value expression: `{{ $json.api.timestamp }}`.  
   - Connect this node input from the Code node output.

5. **Connect Workflow:**  
   - Manual Trigger → Google Sheets → Code Node (Authentication & Data Prep) → HTTP Request Node.

6. **Test Workflow:**  
   - Manually execute the workflow.  
   - Confirm emails are read and request is sent.  
   - Check Icypeas dashboard for results or email notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow demonstrates how to perform batch email verifications using Icypeas. You must have an active Icypeas account for API Key, Secret, and User ID.                                                                                                                                                                                                                                                                                                           | https://www.icypeas.com/                           |
| Google Sheet must be formatted with a header named "email" in the first column containing all email addresses to verify.                                                                                                                                                                                                                                                                                                                                               | Setup instructions in Sticky Note1                |
| Code node requires the Node.js crypto module enabled in n8n. For self-hosted instances, enable it via Settings > General > Additional Node Packages and restart n8n if necessary.                                                                                                                                                                                                                                                                                       | n8n self-hosted settings                           |
| HTTP Request node requires creating a custom HTTP Header Auth credential for Authorization header using dynamic expressions for key and signature.                                                                                                                                                                                                                                                                                                                      | Sticky Note4 instructions                          |
| Verification results are not returned immediately in the workflow response. They are available for download in the Icypeas web dashboard at https://app.icypeas.com/bo/bulksearch?task=email-verification and also sent via email from no-reply@icypeas.com.                                                                                                                                                                                                             | Icypeas bulk verification results location         |

---

This document fully describes the workflow structure, node configuration, and provides all necessary details to recreate and troubleshoot the batch email verification workflow using Icypeas and n8n.