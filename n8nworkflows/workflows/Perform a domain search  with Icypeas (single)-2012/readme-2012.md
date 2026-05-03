Perform a domain search  with Icypeas (single)

https://n8nworkflows.xyz/workflows/perform-a-domain-search--with-icypeas--single--2012


# Perform a domain search  with Icypeas (single)

### 1. Workflow Overview

This workflow performs a domain or company scan using the Icypeas API. It is designed for users who want to programmatically query Icypeas to retrieve information about a single domain or company. The workflow is structured into three logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Authentication:** A code node that generates the necessary authentication signature and prepares API credentials.
- **1.3 Domain Scan Request:** An HTTP request node that performs the actual domain/company scan by calling the Icypeas API with the authenticated details.

The workflow expects the user to provide API credentials (API Key, API Secret, User ID) and the domain or company name to scan. It handles signature generation internally and uses header authentication for the API call.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow manually to initiate the domain scan process. It ensures that the user consciously starts the workflow.

- **Nodes Involved:**  
  - When clicking "Execute Workflow"

- **Node Details:**  
  - **When clicking "Execute Workflow"**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution when the user clicks “Execute Workflow” in the n8n editor or UI.  
    - Configuration: No parameters set; simple manual trigger.  
    - Inputs: None  
    - Outputs: Connected to the authentication code node.  
    - Edge cases: No inputs, so no failures expected here. The workflow runs only when manually triggered.

#### 2.2 Authentication

- **Overview:**  
  This block generates the authentication signature required by Icypeas API. It combines API Key, API Secret, User ID, and timestamp to produce a valid HMAC-SHA1 signature, which is mandatory for API authorization.

- **Nodes Involved:**  
  - Authenticates to your Icypeas account

- **Node Details:**  
  - **Authenticates to your Icypeas account**  
    - Type: Code (JavaScript) Node  
    - Role: Generates authentication signature and prepares API credential details for downstream nodes.  
    - Configuration:  
      - Hardcoded placeholders for API_KEY, API_SECRET, and USER_ID that the user must replace with actual Icypeas credentials.  
      - Uses Node.js crypto module to generate HMAC-SHA1 signature based on HTTP method, path, and timestamp.  
      - Sets output JSON fields: api.key, api.secret, api.userId, api.url (full endpoint), api.timestamp, and api.signature.  
    - Key expressions/variables:  
      - `API_KEY`, `API_SECRET`, `USER_ID` (user input required)  
      - `genSignature()` function computes the signature  
      - Outputs timestamp in ISO format  
    - Input: Receives trigger from manual node.  
    - Output: Provides a JSON object with all API authentication parameters and the full URL for the HTTP request node.  
    - Version-specific requirements: Requires Node.js crypto module enabled; if self-hosting n8n, must enable crypto module in settings as per instructions in sticky note.  
    - Edge cases:  
      - If credentials are missing or incorrect, API calls will fail downstream.  
      - Crypto module unavailability will cause signature generation failure.  
    - Sub-workflow: None

#### 2.3 Domain Scan Request

- **Overview:**  
  This block performs the actual HTTP POST request to Icypeas API to scan the specified domain or company name using the authenticated credentials.

- **Nodes Involved:**  
  - Run domain scan (single)

- **Node Details:**  
  - **Run domain scan (single)**  
    - Type: HTTP Request  
    - Role: Sends POST request to Icypeas domain search API endpoint with authentication headers and body parameters.  
    - Configuration:  
      - URL: Dynamically set from `{{$json.api.url}}` output by the authentication node.  
      - Method: POST  
      - Authentication: Generic HTTP Header Authentication using credentials stored in "Header Auth account".  
      - Headers: Includes `X-ROCK-TIMESTAMP` set from `{{$json.api.timestamp}}`.  
      - Body Parameters: Contains `domainOrCompany` parameter with a hardcoded value `"google"` (can be replaced or parameterized).  
    - Key expressions:  
      - Header value expression: `{{ $json.api.key + ':' + $json.api.signature }}` used in custom credential "Authorization" header.  
      - URL and timestamp dynamically from previous node outputs.  
    - Input: Receives API details and signature from authentication node.  
    - Output: Returns the response from Icypeas API (scan results).  
    - Credentials: Requires a header auth credential configured as per instructions (with Authorization header combining API key and signature).  
    - Edge cases:  
      - HTTP errors like 401 Unauthorized if signature or credentials are incorrect.  
      - Network timeout or unreachable API endpoint.  
      - Missing or malformed body parameters causing API rejection.  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name                     | Node Type       | Functional Role                        | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                         |
|-------------------------------|-----------------|-------------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger  | Initiates workflow execution         | None                          | Authenticates to your Icypeas account |                                                                                                   |
| Authenticates to your Icypeas account | Code            | Generates API authentication signature | When clicking "Execute Workflow" | Run domain scan (single)        | Authenticates to your Icypeas account: Instructions to input API Key, Secret, User ID; crypto module setup steps. |
| Run domain scan (single)       | HTTP Request    | Performs domain/company scan POST request | Authenticates to your Icypeas account | None                          | Performs a domain/company scan on your Icypeas account: Instructions to configure HTTP header auth credential and body parameters. |
| Sticky Note                   | Sticky Note     | Informational note about workflow    | None                          | None                          | ## Perform a domain search (single) with Icypeas. Visit https://icypeas.com to create your account. |
| Sticky Note1                  | Sticky Note     | Instructions for API credentials and crypto module | None                          | None                          | ## Authenticates to your Icypeas account. Instructions on where to input API Key, Secret, User ID, and how to enable crypto module if self-hosted. |
| Sticky Note2                  | Sticky Note     | Instructions for HTTP Request node setup | None                          | None                          | ## Performs a domain/company scan on your Icypeas account. How to configure header auth and body parameters. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Add a "Manual Trigger" node named `When clicking "Execute Workflow"`.  
   - No parameters needed. This node will start the workflow manually.

2. **Create Code node for Authentication:**  
   - Add a "Code" node named `Authenticates to your Icypeas account`.  
   - Paste the following JavaScript code (adjust credentials):  
   ```javascript
   const BASE_URL = "https://app.icypeas.com";
   const PATH = "/api/domain-search";
   const METHOD = "POST";

   // Replace these with your actual Icypeas credentials
   const API_KEY = "PUT_API_KEY_HERE";
   const API_SECRET = "PUT_API_SECRET_HERE";
   const USER_ID = "PUT_USER_ID_HERE";

   const genSignature = (
       path,
       method,
       secret,
       timestamp = new Date().toISOString()
   ) => {
       const Crypto = require('crypto');
       const payload = `${method}${path}${timestamp}`.toLowerCase();
       const sign = Crypto.createHmac("sha1", secret).update(payload).digest("hex");
       return sign;
   };

   const fullPath = `${BASE_URL}${PATH}`;
   $input.first().json.api = {
     timestamp: new Date().toISOString(),
     secret: API_SECRET,
     key: API_KEY,
     userId: USER_ID,
     url: fullPath,
   };
   $input.first().json.api.signature = genSignature(PATH, METHOD, API_SECRET, $input.first().json.api.timestamp);
   return $input.first();
   ```  
   - **Important:** Replace `"PUT_API_KEY_HERE"`, `"PUT_API_SECRET_HERE"`, and `"PUT_USER_ID_HERE"` with your actual credentials from your Icypeas profile at https://app.icypeas.com/bo/profile.  
   - **Note:** If self-hosting n8n, enable the crypto module in n8n settings under "Additional Node Packages" by checking the "crypto" box and saving settings. Restart n8n if necessary.

3. **Create HTTP Request node for domain scan:**  
   - Add an "HTTP Request" node named `Run domain scan (single)`.  
   - Set **HTTP Method** to `POST`.  
   - Set **URL** to expression: `{{$json.api.url}}` (use expression editor).  
   - Enable **Send Body** and **Send Headers**.  
   - Under **Authentication**, select `HTTP Header Auth`.  
   - Create new credentials named e.g. "Header Auth account":  
     - For the header:  
       - **Name:** `Authorization`  
       - **Value:** Expression `{{ $json.api.key + ':' + $json.api.signature }}` (concatenates API key and signature).  
     - Save the credential.  
   - Set header parameters:  
     - Add header `X-ROCK-TIMESTAMP` with value expression `{{$json.api.timestamp}}`.  
   - Set body parameters:  
     - Add parameter named `domainOrCompany`.  
     - Set value to the domain or company name you want to scan, e.g., `"google"`. This can be later parameterized if desired.  
   - Connect output from the Authentication node to this HTTP Request node.

4. **Connect nodes:**  
   - Connect `When clicking "Execute Workflow"` → `Authenticates to your Icypeas account`.  
   - Connect `Authenticates to your Icypeas account` → `Run domain scan (single)`.

5. **Execute and test:**  
   - Save the workflow.  
   - Click "Execute Workflow" manually.  
   - Check the HTTP Request node output for scan results returned by Icypeas.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow demonstrates how to perform a domain scan using Icypeas. Visit https://icypeas.com to create your account.                      | Workflow overview                                   |
| API credentials (API Key, API Secret, User ID) can be found in your Icypeas profile at https://app.icypeas.com/bo/profile.                      | Authentication node instructions                     |
| To use the crypto module in a self-hosted n8n instance, enable it in Settings → General → Additional Node Packages → check “crypto” checkbox. | Crypto module setup instructions                      |
| HTTP Request node requires a custom HTTP Header Auth credential named “Authorization” with value combining API key and signature.               | HTTP Request node instructions                        |
| Scan results are accessible in your Icypeas account at https://app.icypeas.com/bo/singlesearch?task=domain-search                              | For validation and manual result checking            |

---

This document fully describes the workflow “Perform a domain search (single) with Icypeas” to allow understanding, customization, and accurate rebuilding from scratch.