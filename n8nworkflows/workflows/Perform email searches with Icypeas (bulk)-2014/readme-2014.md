Perform email searches with Icypeas (bulk)

https://n8nworkflows.xyz/workflows/perform-email-searches-with-icypeas--bulk--2014


# Perform email searches with Icypeas (bulk)

### 1. Workflow Overview

This workflow is designed to perform bulk email address verification searches using the Icypeas API. It targets users who have lists of persons with first names, last names, and company or domain names stored in a Google Sheet, and want to verify or find related email addresses efficiently through Icypeas.

The workflow’s logic is divided into four main blocks:

- **1.1 Manual Trigger:** Starts the workflow manually on demand.
- **1.2 Google Sheets Data Retrieval:** Reads structured contact data (firstname, lastname, company) from a specified Google Sheet.
- **1.3 Icypeas Authentication Preparation:** Prepares the API request payload and generates the required signature for authenticating with Icypeas.
- **1.4 Bulk Email Search Request:** Sends a POST request to Icypeas to perform the bulk email search using the authenticated parameters.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger

- **Overview:**  
  Initiates the workflow when the user manually clicks ‘Execute Workflow’.

- **Nodes Involved:**  
  - When clicking "Execute Workflow"

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Entry point to start the workflow manually  
  - **Configuration:** No parameters; triggered by user interaction  
  - **Input connections:** None (start node)  
  - **Output connections:** Passes trigger to Google Sheets node  
  - **Version requirements:** Compatible with all recent n8n versions  
  - **Edge cases:** No input data to validate; failure unlikely except UI issues  
  - **Sub-workflow:** None

---

#### 2.2 Google Sheets Data Retrieval

- **Overview:**  
  Reads contact data from a Google Sheet file which must have columns: firstname, lastname, and company.

- **Nodes Involved:**  
  - Reads lastname, firstname and company from your sheet

- **Node Details:**  
  - **Type:** Google Sheets node  
  - **Role:** Fetches rows from a configured Google Sheet document and sheet tab  
  - **Configuration:**  
    - Must specify Google Sheets credentials  
    - Document ID and Sheet Name set as expressions or static values  
  - **Expressions/Variables:** None explicit; outputs rows containing firstname, lastname, company fields  
  - **Input connections:** From manual trigger  
  - **Output connections:** To Icypeas authentication code node  
  - **Version requirements:** 4.1+ for Google Sheets node version compatibility  
  - **Edge cases:**  
    - Missing or malformed document ID or sheet name  
    - Google API auth errors (expired tokens, permission denied)  
    - Missing required columns in sheet causing data issues downstream  
  - **Sub-workflow:** None

---

#### 2.3 Icypeas Authentication Preparation

- **Overview:**  
  Prepares the API call by formatting the data and generating a HMAC-SHA1 signature required by Icypeas for authentication.

- **Nodes Involved:**  
  - Authenticates to your Icypeas account

- **Node Details:**  
  - **Type:** Code node (JavaScript)  
  - **Role:** Generates authentication signature and formats data array for bulk search  
  - **Configuration:**  
    - User must input API_KEY, API_SECRET, and USER_ID inside the script at designated placeholders  
    - Uses built-in Crypto module for HMAC signature generation  
    - Processes input data array of firstname, lastname, company to compose API payload  
  - **Expressions/Variables:**  
    - `data` array constructed from input JSON rows: `[firstname, lastname, company]`  
    - `api` object containing timestamp, signature, keys, userId, and URL  
  - **Input connections:** From Google Sheets node (contact rows)  
  - **Output connections:** To HTTP Request node for bulk search  
  - **Version requirements:**  
    - Crypto module must be enabled in self-hosted n8n instances (instructions provided in sticky note)  
  - **Edge cases:**  
    - Missing or incorrect API credentials leads to authentication failure  
    - Incorrect data format may cause invalid API payload  
    - Crypto module missing or disabled causes runtime errors  
  - **Sub-workflow:** None

---

#### 2.4 Bulk Email Search Request

- **Overview:**  
  Performs the HTTP POST request to Icypeas API endpoint `/bulk-search` to execute the email verification searches in bulk.

- **Nodes Involved:**  
  - Run bulk search (email-search)

- **Node Details:**  
  - **Type:** HTTP Request node  
  - **Role:** Sends authenticated POST request with the prepared payload to Icypeas  
  - **Configuration:**  
    - URL dynamically set from previous node output (`$json.api.url`)  
    - Method: POST  
    - Body parameters: task=email-search, name=Test, user=USER_ID, data=contact data array  
    - Authentication: HTTP Header Auth with custom header `Authorization` set via expression: `{{ $json.api.key + ':' + $json.api.signature }}`  
    - Additional header: `X-ROCK-TIMESTAMP` with timestamp value  
  - **Input connections:** From Code node (authentication prep)  
  - **Output connections:** None (end of workflow)  
  - **Version requirements:** 4.1+ recommended for HTTP Request node features  
  - **Edge cases:**  
    - Credential misconfiguration leads to 401 Unauthorized or 403 Forbidden  
    - Network errors or timeouts may cause request failure  
    - API limit or quota exhaustion may lead to error responses  
    - Response handling is minimal; user must check results manually in Icypeas dashboard or email  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                   | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|-------------------------------------|---------------------|---------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow"    | Manual Trigger      | Starts workflow manually         | None                             | Reads lastname, firstname and company from your sheet |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Reads lastname, firstname and company from your sheet | Google Sheets       | Reads contact data from Google Sheet | When clicking "Execute Workflow" | Authenticates to your Icypeas account | ## Read your Google Sheet file  This node reads a Google Sheet. You need to create a sheet with :   **The first column** : Header : lastname  **The first column** : Header : firstname  **The first column** : Header : company  Don't forget to specify the path of your file in the node and your credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Authenticates to your Icypeas account | Code                | Prepares API authentication and payload | Reads lastname, firstname and company from your sheet | Run bulk search (email-search)     | ## Authenticates to your Icypeas account  This code node utilizes your API key, API secret, and User ID to establish a connection with your Icypeas account.  Open this node and insert your API Key, API secret, and User ID within the quotation marks. You can locate these credentials on your Icypeas profile at https://app.icypeas.com/bo/profile. Do not change any other line of the code.  If you are a self-hosted user, follow these steps to activate the crypto module :  1. Access your n8n instance  2. Go to Settings  3. Select General Settings  4. Enable the Crypto module  5. Save the changes  Once done, restart n8n if needed.                                                                                                                                                                                                                                                                                                                       |
| Run bulk search (email-search)      | HTTP Request        | Executes bulk email search POST request | Authenticates to your Icypeas account | None                             | ## Performs email searches (bulk).  This node executes an HTTP request (POST) to search for the email addresses.  You need to create credentials in the HTTP Request node:  ➔ Create new Credential named “Authorization”  ➔ Set value to expression: {{ $json.api.key + ':' + $json.api.signature }}  ➔ Save credentials  To retrieve results, check https://app.icypeas.com/bo/bulksearch?task=email-search or wait for email from no-reply@icypeas.com.                                                                                                                                                                                                                                                                                                                                                                                                                |
| Sticky Note                         | Sticky Note         | Informational note              | None                             | None                             | ## Email search with Icypeas (bulk search)  This workflow demonstrates how to perform email searches (bulk search) using Icypeas. Visit https://icypeas.com to create your account.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note1                        | Sticky Note         | Informational note about Google Sheets node | None                             | None                             | ## Read your Google Sheet file  This node reads a Google Sheet. You need to create a sheet with :  **The first column** : Header : lastname  **The first column** : Header : firstname  **The first column** : Header : company  Don't forget to specify the path of your file in the node and your credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Sticky Note3                        | Sticky Note         | Instructions for Icypeas API authentication | None                             | None                             | ## Authenticates to your Icypeas account  This code node utilizes your API key, API secret, and User ID to establish a connection with your Icypeas account.  Open this node and insert your API Key, API secret, and User ID within the quotation marks. Found at https://app.icypeas.com/bo/profile.  For self-hosted n8n users, a detailed process to enable the 'crypto' module is provided.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note4                        | Sticky Note         | Instructions for HTTP Request node and retrieving results | None                             | None                             | ## Performs email searches (bulk).  This node executes an HTTP request (POST) to search for the email addresses.  You need to create credentials in the HTTP Request node:  ➔ Create new Credential named “Authorization”  ➔ Set value to expression: {{ $json.api.key + ':' + $json.api.signature }}  ➔ Save credentials  To retrieve results, check https://app.icypeas.com/bo/bulksearch?task=email-search or wait for email from no-reply@icypeas.com.                                                                                                                                                                                                                                                                                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `When clicking "Execute Workflow"` with default settings.

2. **Create Google Sheets Node:**  
   - Add a **Google Sheets** node named `Reads lastname, firstname and company from your sheet`.  
   - Configure with Google Sheets credentials.  
   - Set the **Document ID** to your Google Sheet containing contact data.  
   - Set the **Sheet Name** to the tab that contains data.  
   - This sheet must have columns with exact headers: `lastname`, `firstname`, `company`.  
   - Connect output of Manual Trigger node to this node.

3. **Create Code Node for Icypeas Authentication:**  
   - Add a **Code** node named `Authenticates to your Icypeas account`.  
   - Paste the following logic (interpreted, not raw JSON):  
     - Define API endpoint URL: `https://app.icypeas.com/api/bulk-search`.  
     - Define HTTP method as `POST`.  
     - Insert your **API Key**, **API Secret**, and **User ID** into the respective variables inside the code.  
     - Generate a timestamp (ISO string).  
     - Create a HMAC-SHA1 signature using the secret and concatenation of method, URL, and timestamp (all lowercase).  
     - Extract the firstname, lastname, and company from input items and prepare a data array.  
     - Output an object containing data array and an `api` object with timestamp, key, signature, userId, and URL.  
   - Connect output of Google Sheets node to the Code node.  
   - For self-hosted n8n users, ensure the `crypto` module is enabled in n8n settings.

4. **Create HTTP Request Node for Bulk Search:**  
   - Add an **HTTP Request** node named `Run bulk search (email-search)`.  
   - Configure:  
     - Method: POST  
     - URL: set as expression `{{$json.api.url}}`  
     - Body parameters (form or JSON):  
       - `task`: "email-search"  
       - `name`: "Test" (can be customized)  
       - `user`: set as expression `{{$json.api.userId}}`  
       - `data`: set as expression `{{$json.data}}` (the array of contacts)  
     - Headers:  
       - Add header `X-ROCK-TIMESTAMP` with value `{{$json.api.timestamp}}`  
     - Authentication:  
       - Use HTTP Header Auth  
       - Create new Credential named “Authorization”  
       - In the **Value** field, use an expression: `{{ $json.api.key + ':' + $json.api.signature }}`  
   - Connect output of Code node to this HTTP Request node.

5. **Finalize Workflow:**  
   - Confirm all nodes are connected in order:  
     `Manual Trigger` → `Google Sheets` → `Code (Authentication)` → `HTTP Request`  
   - Save the workflow.

6. **Test the Workflow:**  
   - Click ‘Execute Workflow’.  
   - Verify that the Google Sheets node reads data correctly.  
   - Check that the Code node outputs the signed API data.  
   - Confirm that the HTTP Request node sends POST request without errors.  
   - Monitor Icypeas dashboard and your email for bulk search results.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow demonstrates how to perform bulk email address searches using Icypeas API. Create an Icypeas account at https://icypeas.com to obtain your API credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | https://icypeas.com                                                                                |
| To use the Google Sheets node, ensure your sheet has columns named exactly `lastname`, `firstname`, and `company`. Also, configure Google API credentials correctly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Google Sheets API documentation                                                                    |
| The Code node requires you to input your Icypeas API Key, API Secret, and User ID directly inside the script. For self-hosted n8n instances, enable the `crypto` module in General Settings under “Additional Node Packages” to allow HMAC signature creation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Icypeas Profile: https://app.icypeas.com/bo/profile, n8n Settings UI                               |
| For the HTTP Request node, create an HTTP Header Auth credential named “Authorization” with the value expression `{{ $json.api.key + ':' + $json.api.signature }}` to authenticate API calls. Results of the bulk search will be available after some time on the Icypeas dashboard at https://app.icypeas.com/bo/bulksearch?task=email-search and will also be sent via email from no-reply@icypeas.com.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Icypeas Bulk Search Results: https://app.icypeas.com/bo/bulksearch?task=email-search               |

---

This document provides a complete structured reference for understanding, modifying, and reproducing the "Perform email searches with Icypeas (bulk)" n8n workflow.