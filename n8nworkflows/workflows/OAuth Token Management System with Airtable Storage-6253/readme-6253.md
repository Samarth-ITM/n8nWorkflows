OAuth Token Management System with Airtable Storage

https://n8nworkflows.xyz/workflows/oauth-token-management-system-with-airtable-storage-6253


# OAuth Token Management System with Airtable Storage

---

### 1. Workflow Overview

This workflow implements an **OAuth Token Management System** using n8n and Airtable. It is designed to securely generate, validate, and store OAuth-like Bearer tokens for clients authenticated by `client_id` and `client_secret`. The workflow targets API providers or SaaS platforms that want to handle token issuance and validation without building a complex backend.

**Key logical blocks:**

- **1.1 Input Reception and Validation:** Receives HTTP POST requests with client credentials, validates input structure.
- **1.2 Client Lookup in Airtable:** Searches for the client in Airtable using the provided `client_id`.
- **1.3 Credential Verification:** Confirms the provided `client_secret` matches the stored secret.
- **1.4 Token Generation and Storage:** Generates a secure token, stores it in Airtable with metadata.
- **1.5 Response Handling:** Returns the generated token or error messages via webhook responses.
- **1.6 HTTP Method Handling:** Manages other HTTP methods with an error response.
- **1.7 Testing Utilities:** Manual trigger and HTTP request nodes to facilitate local testing.
- **1.8 Documentation and Metadata:** Sticky notes provide user guidance, references, and instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  Receives incoming POST requests with client credentials at a webhook node and validates the request body structure to ensure it contains only `client_id` and `client_secret`.

- **Nodes Involved:**  
  - `client receiver` (Webhook)  
  - `validator` (Code)  
  - `success` (If)  
  - `validation failed` (Respond to Webhook)

- **Node Details:**

  - **client receiver**  
    - Type: Webhook  
    - Role: Entry point receiving POST requests on path `/token-refresher`  
    - Config: POST method, responds via response nodes  
    - Inputs: External HTTP client  
    - Outputs: Passes JSON body downstream  
    - Edge cases: Non-POST requests handled separately in another block  
    - Credentials: None required here  

  - **validator**  
    - Type: Code  
    - Role: Validates that request body contains only `client_id` and `client_secret`, no missing or extra fields  
    - Config: JavaScript code checks keys and returns success or failure JSON  
    - Expressions: Uses `$json.body` to access incoming payload  
    - Inputs: From `client receiver`  
    - Outputs: JSON with success boolean and error reason if invalid  
    - Failure cases: Missing keys, extra keys trigger failure  
    - Version: Code node v2  

  - **success**  
    - Type: If  
    - Role: Routes flow based on validation success flag  
    - Config: Checks if `$json.success === true`  
    - Inputs: From `validator`  
    - Outputs: True path continues, false path triggers failure response  
    - Version: If node v2.2  

  - **validation failed**  
    - Type: Respond to Webhook  
    - Role: Returns 401 response with error reason from validator  
    - Config: Response code 401, JSON body with success false and error message  
    - Inputs: From `success` (false path)  
    - Outputs: Terminates flow here on failure  

---

#### 2.2 Client Lookup in Airtable

- **Overview:**  
  Searches the Airtable base for a record matching the provided `client_id` to verify client existence.

- **Nodes Involved:**  
  - `get client id` (Airtable)  
  - `client exists` (If)  
  - `invalid client` (Respond to Webhook)

- **Node Details:**

  - **get client id**  
    - Type: Airtable  
    - Role: Searches the "Client IDs" table for a record where `client` field matches incoming `client_id`  
    - Config: Uses Airtable API token credential, formula filter `={client} = "{{client_id}}"`  
    - Expressions: References `$('client receiver').item.json.body.client_id`  
    - Inputs: From `success` (true path)  
    - Outputs: Search results (array of matching records)  
    - Failures: Airtable API errors, no matching records found  
    - Credentials: Airtable API Token  

  - **client exists**  
    - Type: If  
    - Role: Checks if Airtable search returned at least one record with field `id` existing  
    - Config: Checks existence of `$json.id` in Airtable response  
    - Inputs: From `get client id`  
    - Outputs: True path continues, false path triggers invalid client response  

  - **invalid client**  
    - Type: Respond to Webhook  
    - Role: Returns 400 error with "Invalid client id" message  
    - Config: Response code 400, JSON error message  
    - Inputs: From `client exists` (false path)  
    - Outputs: Terminates flow  

---

#### 2.3 Credential Verification

- **Overview:**  
  Validates that the `client_secret` provided in the request matches the secret stored in the Airtable record.

- **Nodes Involved:**  
  - `secret validation` (If)  
  - `generate token` (Code)  
  - `invalid secret` (Respond to Webhook)

- **Node Details:**

  - **secret validation**  
    - Type: If  
    - Role: Compares incoming `secret` from request body with `client_secret` from retrieved Airtable record  
    - Config: Checks string equality: `$json.secret === $('client receiver').item.json.body.client_secret`  
    - Inputs: From `client exists` (true path)  
    - Outputs: True path continues to token generation, false path triggers invalid secret response  
    - Edge cases: Case sensitivity enforced, missing secrets cause failure  

  - **generate token**  
    - Type: Code  
    - Role: Generates a secure random 32-character token (alphanumeric)  
    - Config: JavaScript code uses randomized character selection  
    - Outputs: JSON object with generated token string  
    - Inputs: From `secret validation` (true path)  
    - Edge cases: Ensure randomness and sufficient entropy; potential runtime errors in code node  

  - **invalid secret**  
    - Type: Respond to Webhook  
    - Role: Returns 401 error with "Invalid client secret"  
    - Config: Response code 401, JSON error message  
    - Inputs: From `secret validation` (false path)  
    - Outputs: Terminates flow  

---

#### 2.4 Token Generation and Storage

- **Overview:**  
  Stores the newly generated token in Airtable, associating it with the client and metadata such as creation date and token type.

- **Nodes Involved:**  
  - `create token` (Airtable)  
  - `respond ` (Respond to Webhook)

- **Node Details:**

  - **create token**  
    - Type: Airtable  
    - Role: Creates a new record in the "Tokens" table with the generated token and client info  
    - Config: Fields mapped to token ID, client IDs (array), token type ("Bearer"), creation date (current time)  
    - Inputs: From `generate token` output, references `client receiver` node for client ID  
    - Outputs: Newly created Airtable record data  
    - Credentials: Airtable API token  
    - Edge cases: Airtable API write errors, typecasting issues  

  - **respond **  
    - Type: Respond to Webhook  
    - Role: Sends JSON response with `access_token`, expiration time (3600 seconds), and token type back to client  
    - Config: Response body builds from generated token and Airtable fields  
    - Inputs: From `create token`  
    - Outputs: Final webhook response to client  

---

#### 2.5 HTTP Method Handling

- **Overview:**  
  Handles unsupported HTTP methods on the `/token-refresher` endpoint by responding with a 405 Method Not Allowed error.

- **Nodes Involved:**  
  - `Other methods` (Webhook)  
  - `405 Error` (Respond to Webhook)

- **Node Details:**

  - **Other methods**  
    - Type: Webhook  
    - Role: Listens on `/test-jobs` path for HTTP methods other than POST (`DELETE`, `HEAD`, `PATCH`, `PUT`, `GET`)  
    - Config: Multiple HTTP methods configured, responseMode set to response node  
    - Outputs: Passes execution to error response node  
    - Inputs: External calls using unsupported HTTP methods  

  - **405 Error**  
    - Type: Respond to Webhook  
    - Role: Returns HTTP 405 with JSON error message instructing to use POST only  
    - Config: Response code 405, JSON body with error message  
    - Inputs: From `Other methods`  

---

#### 2.6 Testing Utilities

- **Overview:**  
  Provides manual trigger and HTTP request nodes for local testing of the token generation workflow.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Make a request` (HTTP Request)  
  - `Sticky Note` (Test the request)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow from the n8n editor  
    - Outputs: Triggers the HTTP request node  

  - **Make a request**  
    - Type: HTTP Request  
    - Role: Sends a POST request to the local `/token-refresher` webhook with example client credentials in JSON body  
    - Config: URL points to localhost webhook, method POST, JSON body with sample `client_id` and `client_secret`  
    - Inputs: From manual trigger  
    - Outputs: Response received can be inspected in n8n execution  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides label "Test the request" for clarity in editor  

---

#### 2.7 Documentation and Metadata

- **Overview:**  
  Sticky notes provide important documentation, example Airtable base link, instructions, and project description within the workflow editor.

- **Nodes Involved:**  
  - `Sticky Note3` (OAuth Token Generator and Validator description)  
  - `Sticky Note2` (Database Example with Airtable base clone link)  
  - `Sticky Note1` (HTTP Method Handler description)  
  - `Sticky Note` (Test the request label)

- **Node Details:**

  - **Sticky Note3**  
    - Contains detailed workflow description, purpose, how it works, project credits, and related workflows  
    - Includes link to Airtable base: https://airtable.com/appbw5TEhn8xIxxXR/shrN8ve4dfJIXjcAm  
    - Credits builder: Nazmy (https://n8n.io/creators/islamnazmi/)  
    - Serves as comprehensive in-editor documentation  

  - **Sticky Note2**  
    - Provides direct link to Airtable base example for clients and tokens  
    - Helps users clone and set up Airtable schema  

  - **Sticky Note1**  
    - Labels HTTP Method handler section for clarity  

  - **Sticky Note**  
    - Labels Test the request manual trigger and HTTP request nodes  

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                          | Input Node(s)           | Output Node(s)          | Sticky Note                                              |
|---------------------------|----------------------|----------------------------------------|------------------------|-------------------------|----------------------------------------------------------|
| client receiver           | Webhook              | Receives POST request with credentials | -                      | validator               |                                                          |
| validator                 | Code                 | Validates request body keys             | client receiver        | success                 |                                                          |
| success                   | If                   | Routes on validation success            | validator              | get client id, validation failed |                                                          |
| validation failed         | Respond to Webhook   | Responds with 401 if validation fails   | success                | -                       |                                                          |
| get client id             | Airtable             | Searches Airtable for client record     | success                | client exists           |                                                          |
| client exists             | If                   | Checks if client found                   | get client id          | secret validation, invalid client |                                                          |
| invalid client            | Respond to Webhook   | Responds with 400 if client not found    | client exists (false)  | -                       |                                                          |
| secret validation         | If                   | Validates client_secret matches          | client exists (true)   | generate token, invalid secret |                                                          |
| generate token            | Code                 | Generates secure token                    | secret validation (true) | create token           |                                                          |
| invalid secret            | Respond to Webhook   | Responds with 401 if secret invalid      | secret validation (false) | -                     |                                                          |
| create token              | Airtable             | Stores token and metadata                 | generate token         | respond                 |                                                          |
| respond                   | Respond to Webhook   | Responds with token JSON                  | create token           | -                       |                                                          |
| Other methods             | Webhook              | Handles unsupported HTTP methods         | -                      | 405 Error               |                                                          |
| 405 Error                 | Respond to Webhook   | Returns 405 Method Not Allowed error     | Other methods          | -                       |                                                          |
| When clicking ‘Execute workflow’ | Manual Trigger | Triggers local test HTTP request         | -                      | Make a request          |                                                          |
| Make a request            | HTTP Request         | Sends test POST to token refresher webhook | When clicking ‘Execute workflow’ | -                |                                                          |
| Sticky Note3              | Sticky Note          | Workflow description and credits         | -                      | -                       | Contains detailed workflow overview and Airtable link    |
| Sticky Note2              | Sticky Note          | Airtable base example link                | -                      | -                       | Contains Airtable Base clone link                         |
| Sticky Note1              | Sticky Note          | Labels HTTP method handler                | -                      | -                       |                                                          |
| Sticky Note               | Sticky Note          | Labels test request block                  | -                      | -                       |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "client receiver":**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `token-refresher`  
   - Response Mode: Respond with response nodes  
   - No credentials needed  

2. **Create Code Node "validator":**  
   - Input: Connect from `client receiver`  
   - JavaScript: Validate that body contains exactly `client_id` and `client_secret` keys. Return JSON with `success` boolean and `reason` if failed.  
   - Use `$json.body` to access the request data.

3. **Create If Node "success":**  
   - Input: Connect from `validator`  
   - Condition: `$json.success === true` (Boolean equals true)  
   - True output continues, False output leads to failure response.

4. **Create Respond to Webhook Node "validation failed":**  
   - Input: Connect from `success` (False path)  
   - Response Code: 401  
   - Response Body: JSON with `{ success: false, error: reason_from_validator }`  
   - Use expression to pull validator's reason: `{{ $('validator').item.json.reason }}`

5. **Create Airtable Node "get client id":**  
   - Input: Connect from `success` (True path)  
   - Operation: Search in base `appbw5TEhn8xIxxXR`, table `Client IDs` (`tblK2jv4hLOsaKM3m`)  
   - Filter Formula: `{client} = "{{ $('client receiver').item.json.body.client_id }}"`  
   - Credential: Airtable API Token (create or reuse)  
   - Always output data: true  

6. **Create If Node "client exists":**  
   - Input: Connect from `get client id`  
   - Condition: Check if `$json.id` exists (string exists)  
   - True leads to secret validation, False leads to invalid client response.

7. **Create Respond to Webhook Node "invalid client":**  
   - Input: Connect from `client exists` (False path)  
   - Response Code: 400  
   - Response Body: `{ "success": false, "error": "Invalid client id" }`

8. **Create If Node "secret validation":**  
   - Input: Connect from `client exists` (True path)  
   - Condition: Check if `{{$json.secret}} === {{ $('client receiver').item.json.body.client_secret }}`  
   - True proceeds to token generation, False to invalid secret response.

9. **Create Respond to Webhook Node "invalid secret":**  
   - Input: Connect from `secret validation` (False path)  
   - Response Code: 401  
   - Response Body: `{ "success": false, "error": "Invalid client secret" }`

10. **Create Code Node "generate token":**  
    - Input: Connect from `secret validation` (True path)  
    - JS Code: Generate a random 32-character alphanumeric string as token.  
    - Output JSON: `{ token: generatedToken }`

11. **Create Airtable Node "create token":**  
    - Input: Connect from `generate token`  
    - Operation: Create record in base `appbw5TEhn8xIxxXR`, table `Tokens` (`tblnqvjl4U2t9OMQD`)  
    - Fields to set:  
      - Token ID: `={{ $json.token }}`  
      - Client IDs: `={{ [$('client receiver').item.json.body.client_id] }}` (as array)  
      - Token Type: `"Bearer"`  
      - Creation Date: `={{ $now }}`  
    - Credential: Airtable API Token  
    - Enable Typecast: true  
    - Always output data: true

12. **Create Respond to Webhook Node "respond ":**  
    - Input: Connect from `create token`  
    - Response Body: JSON format:  
      ```json
      {
        "access_token": "{{ $('generate token').item.json.token }}",
        "expires_in": 3600,
        "token_type": "{{ $json.fields['Token Type'] }}"
      }
      ```

13. **Create Webhook Node "Other methods":**  
    - HTTP Methods: `DELETE`, `HEAD`, `PATCH`, `PUT`, `GET`  
    - Path: `test-jobs` (or `/token-refresher` if preferred)  
    - Response Mode: Response node  

14. **Create Respond to Webhook Node "405 Error":**  
    - Input: Connect from `Other methods`  
    - Response Code: 405  
    - Response Body: `{ "error": "Use POST request instead" }`

15. **Create Manual Trigger Node "When clicking ‘Execute workflow’":**  
    - For manual execution and testing  

16. **Create HTTP Request Node "Make a request":**  
    - Input: Connect from manual trigger  
    - Method: POST  
    - URL: `https://localhost:8080/webhook/token-refresher` (adjust to your environment)  
    - Body (JSON):  
      ```json
      {
        "client_id": "client_a_1234567890abcdef",
        "client_secret": "secret_a_abcdef1234567890"
      }
      ```  
    - Send body as JSON

17. **Add Sticky Notes:**  
    - Add descriptive sticky notes in the editor for clarity, including links to Airtable base and detailed documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Clone this Airtable base for client and token management: [Airtable Base](https://airtable.com/appbw5TEhn8xIxxXR/shrN8ve4dfJIXjcAm) | Airtable base schema example for clients and tokens                                                     |
| This n8n template implements OAuth-like token generation and validation using n8n and Airtable to manage secure API tokens.     | Workflow description and usage notes included in Sticky Note3                                           |
| Related workflow for token validation: [Validate API Requests with Bearer Token Authentication and Airtable](https://n8n.io/workflows/6184-validate-api-requests-with-bearer-token-authentication-and-airtable) | For secure token validation complementing this token generator workflow                                  |
| Built by Nazmy, see creator profile: [Islam Nazmi](https://n8n.io/creators/islamnazmi/)                                          | Project credit and additional resources                                                                |

---

**Disclaimer:**  
The text and workflow are generated from an automated n8n workflow. All data handled is legal and public. No illegal or offensive content is included.

---