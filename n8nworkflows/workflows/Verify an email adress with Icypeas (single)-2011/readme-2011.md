Verify an email adress with Icypeas (single)

https://n8nworkflows.xyz/workflows/verify-an-email-adress-with-icypeas--single--2011


# Verify an email adress with Icypeas (single)

### 1. Workflow Overview

This workflow automates the verification of a single email address using the Icypeas email verification API. It targets users who want to validate an email's existence and status by leveraging their Icypeas account credentials within n8n.

The workflow is logically divided into three main blocks:

- **1.1 Trigger Input:** Manual initiation of the workflow by the user.
- **1.2 Authentication Preparation:** A custom code node that prepares the API request by generating the necessary authentication headers and parameters (API key, secret, user ID, and signature).
- **1.3 Email Verification Execution:** An HTTP POST request node that sends the email to Icypeas and receives verification results.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Input

**Overview:**  
This block starts the workflow manually whenever the user clicks the execute button in n8n. It provides a simple and controlled entry point.

**Nodes Involved:**  
- When clicking "Execute Workflow"

**Node Details:**

- **When clicking "Execute Workflow"**  
  - **Type:** Manual Trigger  
  - **Technical Role:** Initiates workflow execution on user interaction.  
  - **Configuration:** Default manual trigger; no parameters required.  
  - **Expressions/Variables:** None.  
  - **Input/Output:** No input; outputs a single trigger event to the next node.  
  - **Version Requirements:** Compatible with all n8n versions supporting manual trigger nodes.  
  - **Potential Failures:** None expected under normal usage.  
  - **Sub-workflow:** None.

---

#### 2.2 Authentication Preparation

**Overview:**  
This code node generates the authentication signature and prepares all necessary API parameters required to authenticate requests to the Icypeas email verification API. It uses the API key, API secret, and user ID provided by the user.

**Nodes Involved:**  
- Authenticates to your Icypeas account

**Node Details:**

- **Authenticates to your Icypeas account**  
  - **Type:** Code Node (JavaScript)  
  - **Technical Role:** Creates the authorization signature and prepares API request metadata.  
  - **Configuration:**  
    - User must replace placeholders in the code for `API_KEY`, `API_SECRET`, and `USER_ID` with their Icypeas credentials.  
    - Uses Node.js crypto module to generate an HMAC-SHA1 signature required by Icypeas.  
    - Constructs a timestamp in ISO 8601 format.  
    - Outputs an enriched JSON object with API details (`url`, `timestamp`, `key`, `secret`, `userId`, `signature`).  
  - **Key Expressions/Variables:**  
    - `API_KEY`, `API_SECRET`, `USER_ID` (must be manually set)  
    - `genSignature()` function: generates HMAC signature using method, path, secret, timestamp  
    - Outputs under `$json.api` object for downstream nodes.  
  - **Input/Output:**  
    - Input: Trigger event from manual trigger node (no payload used).  
    - Output: JSON object with authentication info merged into data.  
  - **Version Requirements:**  
    - Requires n8n instance with Node.js crypto module enabled.  
    - For self-hosted users, crypto module activation instructions are provided in sticky notes.  
  - **Edge Cases / Failures:**  
    - Failure to replace API credentials will cause authentication errors.  
    - Missing or disabled crypto module will cause runtime errors.  
    - Timestamp or signature generation issues due to system time misconfiguration.  
  - **Sub-workflow:** None.

---

#### 2.3 Email Verification Execution

**Overview:**  
Sends the prepared HTTP POST request to the Icypeas API endpoint to verify the specified email address. The request includes required headers and body parameters for authentication and email verification.

**Nodes Involved:**  
- Run email verification (single)

**Node Details:**

- **Run email verification (single)**  
  - **Type:** HTTP Request Node  
  - **Technical Role:** Executes POST request to Icypeas API endpoint to verify the email.  
  - **Configuration:**  
    - URL: Dynamically set to `$json.api.url` from previous node (full API endpoint).  
    - HTTP Method: POST.  
    - Authentication: Generic header authentication using credentials created with the header `Authorization`. The header value is an expression combining API key and signature: `{{ $json.api.key + ':' + $json.api.signature }}`.  
    - Headers: Includes `X-ROCK-TIMESTAMP` header with timestamp.  
    - Body Parameters: Includes a single parameter named "email" with a hardcoded example email address (`uyqsdqkudhfiqudhfiqduhfiqdhfqif@gmail.com`) — this should be replaced to validate different emails.  
    - Sends both headers and body in the request.  
  - **Key Expressions/Variables:**  
    - URL: `={{ $json.api.url }}`  
    - Header `X-ROCK-TIMESTAMP`: `={{ $json.api.timestamp }}`  
    - Authorization Header Credential: expression `{{ $json.api.key + ':' + $json.api.signature }}`  
    - Body parameter "email": hardcoded string value (replaceable).  
  - **Input/Output:**  
    - Input: JSON object enriched with authentication details.  
    - Output: Icypeas API response containing verification results about the email.  
  - **Version Requirements:**  
    - Requires HTTP Request node version 4.1 or later for generic header authentication support.  
  - **Edge Cases / Failures:**  
    - Invalid or expired credentials cause 401 Unauthorized errors.  
    - Network errors or API downtime.  
    - Incorrect email format or invalid email leads to negative verification results.  
    - Hardcoded email parameter limits reusability unless modified or parameterized.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                        | Node Type           | Functional Role                      | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|---------------------------------|---------------------|------------------------------------|-------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger      | Workflow initiation trigger         | —                             | Authenticates to your Icypeas account |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Authenticates to your Icypeas account | Code Node           | Prepare API authentication details | When clicking "Execute Workflow" | Run email verification (single)   | Authenticates to your Icypeas account: Insert API Key, Secret, User ID inside the code node. Instructions for enabling Node.js crypto module on self-hosted n8n instances. Credentials are found at https://app.icypeas.com/bo/profile.                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Run email verification (single) | HTTP Request        | Performs the email verification POST request | Authenticates to your Icypeas account  | —                               | Performs an email verification on your Icypeas account. Requires creating HTTP Header Auth credential with Authorization header set to `{{ $json.api.key + ':' + $json.api.signature }}`. Set email parameter in body to the email to verify. Results can be seen at https://app.icypeas.com/bo/singlesearch?task=email-verification.                                                                                                                                                                                                                                                                                                                                        |
| Sticky Note                     | Sticky Note         | Informational overview              | —                             | —                               | ## Email verification with Icypeas (single)\n\nThis workflow demonstrates how to perform an email verification using Icypeas. Visit https://icypeas.com to create your account.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Sticky Note1                    | Sticky Note         | Authentication instructions        | —                             | —                               | ## Authenticates to your Icypeas account\n\nThis code node utilizes your API key, API secret, and User ID to establish a connection with your Icypeas account.\n\nOpen this node and insert your API Key, API secret, and User ID within the quotation marks. You can locate these credentials on your Icypeas profile at https://app.icypeas.com/bo/profile.\n\nDo not change any other line of the code.\n\nInstructions to enable crypto module for self-hosted n8n instances.                                                                                                                                                                                                                                            |
| Sticky Note2                    | Sticky Note         | HTTP request execution instructions | —                             | —                               | ## Performs an email verification on your Icypeas account\n\nThis node executes an HTTP POST request to verify the email provided in the body using Icypeas.\n\nYou need to create credentials for Header Auth with the Authorization header set to `{{ $json.api.key + ':' + $json.api.signature }}`.\n\nTo verify the email, set the body parameter `email` to the email address.\n\nVerification results are accessible at https://app.icypeas.com/bo/singlesearch?task=email-verification.                                                                                                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `"When clicking \"Execute Workflow\""` with default settings.  
   - This node will start the workflow manually.

2. **Create Code Node for Authentication:**  
   - Add a **Code** node named `"Authenticates to your Icypeas account"`.  
   - Use JavaScript as language.  
   - Insert the following code snippet (replace credentials accordingly):  
     ```javascript
     const BASE_URL = "https://app.icypeas.com";
     const PATH = "/api/email-verification";
     const METHOD = "POST";

     // Replace these with your Icypeas credentials
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
   - Connect output of manual trigger node to this code node.  
   - **Important:** Replace `"PUT_API_KEY_HERE"`, `"PUT_API_SECRET_HERE"`, and `"PUT_USER_ID_HERE"` with your actual Icypeas credentials.  
   - **Ensure** your n8n instance has the `crypto` module enabled (instructions provided below).

3. **Create HTTP Request Node for Verification:**  
   - Add an **HTTP Request** node named `"Run email verification (single)"`.  
   - Configure as follows:  
     - **HTTP Method:** POST  
     - **URL:** Use an expression: `{{$json.api.url}}` (comes from previous node)  
     - **Authentication:** Choose "Header Auth" (generic header authentication).  
       - Create new credentials:  
         - Name: `"Authorization"`  
         - Value: Use expression: `{{$json.api.key + ':' + $json.api.signature}}`  
     - **Headers:** Add header parameter:  
       - Name: `X-ROCK-TIMESTAMP`  
       - Value: Expression: `{{$json.api.timestamp}}`  
     - **Body Parameters:**  
       - Add parameter:  
         - Name: `email`  
         - Value: Enter the email address you want to verify (replace the example email).  
     - Enable options to send body and headers.  
   - Connect output of the code node to this HTTP Request node.

4. **Save and Activate Workflow:**  
   - Save your workflow and set it to active if desired.

5. **Run Workflow:**  
   - Click the execute button on the manual trigger node to test.  
   - The HTTP Request node will return the email verification result from Icypeas.

**Additional Setup Notes:**

- **Enabling Crypto Module on Self-Hosted n8n:**  
  - Log into your n8n instance UI.  
  - Navigate to your user settings → General → Additional Node Packages.  
  - Check the box for `crypto` to enable the module.  
  - Save changes and restart n8n if necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow demonstrates single email verification using the Icypeas API. To use it, you need an active Icypeas account with API credentials. Visit https://www.icypeas.com/ to create an account.                                                                                                                                                                                                                                                                                                         | Icypeas main website                                         |
| Credentials (API Key, API Secret, User ID) required for this workflow can be found at https://app.icypeas.com/bo/profile.                                                                                                                                                                                                                                                                                                                                                                                      | Icypeas credentials page                                    |
| Results of email verifications performed by this workflow can be viewed at https://app.icypeas.com/bo/singlesearch?task=email-verification.                                                                                                                                                                                                                                                                                                                                                                  | Icypeas verification results UI                              |
| The workflow requires the Node.js `crypto` module for generating HMAC signatures. For self-hosted n8n users, enable this module in settings under Additional Node Packages.                                                                                                                                                                                                                                                                                                                                  | Crypto module enable instructions                            |

---

This completes the detailed reference documentation of the "Verify an email address with Icypeas (single)" n8n workflow, enabling users and developers to understand, reproduce, and maintain the workflow confidently.