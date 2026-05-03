Secure API Endpoint with Bearer Token Authentication and Field Validation

https://n8nworkflows.xyz/workflows/secure-api-endpoint-with-bearer-token-authentication-and-field-validation-3970


# Secure API Endpoint with Bearer Token Authentication and Field Validation

### 1. Workflow Overview

This n8n workflow provides a **secure API endpoint** by protecting a public webhook with **Bearer Token authentication** and **dynamic request body validation**. It is designed to be reusable and production-ready for developers and builders who need to expose workflows as authenticated APIs with basic field validation.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception**  
  The entry point where an external `POST` request is received via a webhook.

- **1.2 Configuration Setup**  
  Defines the secret Bearer token and the required fields expected in the request body.

- **1.3 Authentication**  
  Verifies that the incoming request includes a valid Bearer token in the Authorization header.

- **1.4 Validation**  
  Checks that all required fields configured are present in the request body.

- **1.5 Business Logic Placeholder**  
  A placeholder node where users can insert their own workflow logic after validation passes.

- **1.6 Response Handling**  
  Returns standardized JSON responses depending on the outcome:  
  - `401 Unauthorized` if token is missing or invalid  
  - `400 Bad Request` if required fields are missing  
  - `200 OK` with a customizable success message

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Accepts incoming HTTP POST requests at a fixed webhook path with response mode set to delegate response handling to downstream nodes.

- **Nodes Involved:**  
  - `Webhook`

- **Node Details:**  
  - **Type:** Webhook (HTTP entry point)  
  - **Configuration:**  
    - Path: `secure-webhook`  
    - HTTP Method: `POST`  
    - Response mode: `responseNode` (response is handled by a subsequent Respond to Webhook node)  
  - **Inputs:** None  
  - **Outputs:** Passes incoming request data including headers and body to next node (`Configuration`)  
  - **Edge Cases:**  
    - No request or unsupported method → n8n default webhook error  
    - Large payloads may require adjusting webhook settings externally  
  - **Sub-Workflow:** None

---

#### 2.2 Configuration Setup

- **Overview:**  
  Sets static configuration values for the workflow: the secret Bearer token and the list of required fields to validate.

- **Nodes Involved:**  
  - `Configuration`

- **Node Details:**  
  - **Type:** Set node  
  - **Configuration:**  
    - Assigns `config.bearerToken` (e.g., `"123"`) as the secret token to verify  
    - Assigns `config.requiredFields` object with keys representing required fields (e.g., `{ message: true }`)  
  - **Inputs:** Receives data from `Webhook` node (includes webhook request json)  
  - **Outputs:** Forwards enriched data including config to next node (`Check Authorization Header`)  
  - **Edge Cases:**  
    - Misconfiguration of token or required fields can cause unintended failures  
    - Values assigned to required fields keys are ignored; only keys matter  
  - **Sub-Workflow:** None

---

#### 2.3 Authentication

- **Overview:**  
  Validates the presence and correctness of the Bearer token in the `Authorization` HTTP header.

- **Nodes Involved:**  
  - `Check Authorization Header` (If node)  
  - `401 Unauthorized` (Respond to Webhook node)

- **Node Details:**  

  **Check Authorization Header**  
  - **Type:** If node  
  - **Configuration:**  
    - Compares the incoming header string from `Webhook` (`headers.authorization`) to the expected string: `"Bearer <config.bearerToken>"`  
    - Expression used: `={{ $('Webhook').item.json.headers.authorization }}` equals `"Bearer {{ $json.config.bearerToken }}"`  
  - **Inputs:** Receives data with config and webhook request  
  - **Outputs:**  
    - True branch: proceeds to `Has required fields?` node (validation)  
    - False branch: sends to `401 Unauthorized` node  
  - **Edge Cases:**  
    - Missing `Authorization` header → false branch  
    - Token mismatch → false branch  
    - Case sensitivity enforced; header must exactly match `"Bearer <token>"`  
    - Header formatting errors may cause false negatives  
  - **Sub-Workflow:** None

  **401 Unauthorized**  
  - **Type:** Respond to Webhook node  
  - **Configuration:**  
    - HTTP Response Code: `401`  
    - Response format: JSON with keys: `code`, `message`, and `hint` explaining the error  
  - **Inputs:** Receives from false branch of `Check Authorization Header`  
  - **Outputs:** None (ends workflow with response)  
  - **Edge Cases:** None  
  - **Sub-Workflow:** None

---

#### 2.4 Validation

- **Overview:**  
  Checks if the incoming request body contains all fields specified as required in the configuration.

- **Nodes Involved:**  
  - `Has required fields?` (Code node)  
  - `Check Valid Request` (If node)  
  - `400 Bad Request` (Respond to Webhook node)

- **Node Details:**  

  **Has required fields?**  
  - **Type:** Code node (JavaScript)  
  - **Configuration:**  
    - Reads `config.requiredFields` from current item JSON  
    - Reads request body from `Webhook` node's first item json body  
    - Iterates over keys of `config.requiredFields` and checks presence in body  
    - Returns `{ valid: true }` if all keys are present, else `{ valid: false }`  
  - **Inputs:** Receives from true branch of `Check Authorization Header`  
  - **Outputs:** Passes `{ valid: boolean }` to `Check Valid Request` node  
  - **Edge Cases:**  
    - Empty or missing `config.requiredFields` results in `valid: true` (no fields required)  
    - Body missing or improperly formatted may cause errors or false negatives  
  - **Sub-Workflow:** None

  **Check Valid Request**  
  - **Type:** If node  
  - **Configuration:**  
    - Checks if `$json.valid === true`  
  - **Inputs:** Receives validation result from `Has required fields?`  
  - **Outputs:**  
    - True branch: proceeds to business logic placeholder node  
    - False branch: sends to `400 Bad Request` node  
  - **Edge Cases:** None  
  - **Sub-Workflow:** None

  **400 Bad Request**  
  - **Type:** Respond to Webhook node  
  - **Configuration:**  
    - HTTP Response Code: `400` (Note: the node is incorrectly configured with 401 in JSON but comment and naming indicate 400)  
    - Responds with JSON error message indicating missing required fields and a hint  
  - **Inputs:** Receives from false branch of `Check Valid Request`  
  - **Outputs:** None (ends workflow with response)  
  - **Edge Cases:** None  
  - **Sub-Workflow:** None

---

#### 2.5 Business Logic Placeholder

- **Overview:**  
  Placeholder node where users can insert custom workflow logic once authentication and validation pass.

- **Nodes Involved:**  
  - `Add workflow nodes here`

- **Node Details:**  
  - **Type:** NoOp node (does nothing)  
  - **Configuration:** Empty; serves as a placeholder for user nodes  
  - **Inputs:** Receives from true branch of `Check Valid Request`  
  - **Outputs:** Passes data unchanged to `Create Response` node  
  - **Edge Cases:** None  
  - **Sub-Workflow:** None

---

#### 2.6 Response Handling

- **Overview:**  
  Constructs a success response and sends it back to the requester.

- **Nodes Involved:**  
  - `Create Response` (Set node)  
  - `200 OK` (Respond to Webhook node)

- **Node Details:**  

  **Create Response**  
  - **Type:** Set node  
  - **Configuration:**  
    - Sets a JSON field `message` with a success string: `"Success! Workflow completed."`  
    - User can customize this node to include other response data from the request or workflow results  
  - **Inputs:** Receives from `Add workflow nodes here`  
  - **Outputs:** Passes data to `200 OK` node  
  - **Edge Cases:** None  
  - **Sub-Workflow:** None

  **200 OK**  
  - **Type:** Respond to Webhook node  
  - **Configuration:**  
    - Default HTTP 200 response code (no override specified)  
    - Sends JSON response constructed in the previous node  
  - **Inputs:** Receives from `Create Response`  
  - **Outputs:** Ends workflow with success response  
  - **Edge Cases:** None  
  - **Sub-Workflow:** None

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                           | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                   |
|----------------------------|---------------------------|-----------------------------------------|--------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook                    | Webhook                   | Entry point for incoming HTTP POST      | None                     | Configuration              |                                                                                                               |
| Configuration              | Set                       | Defines secret token and required fields| Webhook                  | Check Authorization Header | 🛠️ Config Node Setup: defines config.bearerToken and config.requiredFields keys (values ignored)               |
| Check Authorization Header | If                        | Validates Authorization Bearer token   | Configuration            | Has required fields?, 401 Unauthorized |                                                                                                               |
| 401 Unauthorized           | Respond to Webhook        | Returns 401 Unauthorized JSON response | Check Authorization Header (false branch) | None              | 🚫 Error Handling Nodes: returns standard 401 if token missing or invalid                                     |
| Has required fields?       | Code                      | Validates required fields in request   | Check Authorization Header (true branch) | Check Valid Request         | 🔍 Required Fields Validator: checks presence of each key in config.requiredFields in request body            |
| Check Valid Request        | If                        | Checks if request fields are valid      | Has required fields?      | Add workflow nodes here, 400 Bad Request |                                                                                                               |
| 400 Bad Request            | Respond to Webhook        | Returns 400 Bad Request JSON response   | Check Valid Request (false branch) | None                      | 🚫 Error Handling Nodes: returns standard 400 if required fields missing                                     |
| Add workflow nodes here    | NoOp                      | Placeholder for user business logic     | Check Valid Request (true branch) | Create Response            |                                                                                                               |
| Create Response            | Set                       | Builds success JSON response             | Add workflow nodes here   | 200 OK                     | ✅ Set & 200 Response Nodes: customize success message and response payload                                  |
| 200 OK                    | Respond to Webhook        | Returns 200 OK success response          | Create Response           | None                       | ✅ Set & 200 Response Nodes: sends success response                                                         |
| Sticky Note                | Sticky Note               | Configuration node explanation           | None                     | None                       | 🛠️ Config Node Setup content                                                                                   |
| Sticky Note1               | Sticky Note               | Explains error handling nodes            | None                     | None                       | 🚫 Error Handling Nodes content                                                                                 |
| Sticky Note2               | Sticky Note               | Explains response nodes                   | None                     | None                       | ✅ Set & 200 Response Nodes content                                                                             |
| Sticky Note3               | Sticky Note               | Explains required fields validator       | None                     | None                       | 🔍 Required Fields Validator content                                                                             |
| Sticky Note4               | Sticky Note               | Workflow summary and usage explanation   | None                     | None                       | 🔐 Secure Webhook – Summary content                                                                              |
| Sticky Note6               | Sticky Note               | Author support and credits                | None                     | None                       | Support My Work! ❤️ content                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Set HTTP Method: `POST`  
   - Set Path: `secure-webhook`  
   - Set Response Mode: `responseNode` (response handled downstream)  
   - This node receives incoming requests.

2. **Create Set Node named `Configuration`:**  
   - Assign variable `config.bearerToken` with your secret token string (e.g., `"123"`)  
   - Assign variable `config.requiredFields` with keys for required fields, e.g., `message` set to any value (value ignored)  
   - Connect output of `Webhook` → input of `Configuration`

3. **Create If Node named `Check Authorization Header`:**  
   - Condition Type: String equals  
   - Left Value: Expression: `={{ $('Webhook').item.json.headers.authorization }}`  
   - Right Value: Expression: `"Bearer {{ $json.config.bearerToken }}"`  
   - Connect output of `Configuration` → input of `Check Authorization Header`

4. **Create Respond to Webhook Node named `401 Unauthorized`:**  
   - Set Response Code: `401`  
   - Respond With: JSON  
   - Response Body:  
     ```json
     {
       "code": 401,
       "message": "Unauthorized: Missing or invalid authorization token.",
       "hint": "Ensure the request includes a valid 'Authorization' header (e.g., 'Bearer YOUR_SECRET_TOKEN')."
     }
     ```  
   - Connect **False** output of `Check Authorization Header` → input of `401 Unauthorized`

5. **Create Code Node named `Has required fields?`:**  
   - Mode: Run Once For Each Item  
   - JavaScript code:  
     ```javascript
     if(! $json.config.requiredFields) {
       return { json: { valid: true } };
     }
     
     const body = $('Webhook').first().json.body;
     let requiredFields = $json.config.requiredFields;
     
     for (let [key, value] of Object.entries(requiredFields)) {
       if (!(key in body)) {
         return { json: { valid: false } };
       }
     }
     
     return { json: { valid: true } };
     ```  
   - Connect **True** output of `Check Authorization Header` → input of `Has required fields?`

6. **Create If Node named `Check Valid Request`:**  
   - Condition Type: Boolean equals  
   - Left Value: Expression: `={{ $json.valid }}`  
   - Right Value: `true`  
   - Connect output of `Has required fields?` → input of `Check Valid Request`

7. **Create Respond to Webhook Node named `400 Bad Request`:**  
   - Set Response Code: `400`  
   - Respond With: JSON  
   - Response Body:  
     ```json
     {
       "code": 400,
       "message": "Bad Request: Missing required fields",
       "hint": "Make sure all required fields are included in the request body."
     }
     ```  
   - Connect **False** output of `Check Valid Request` → input of `400 Bad Request`

8. **Create NoOp Node named `Add workflow nodes here`:**  
   - This is a placeholder node where you can insert your own workflow logic after validation.  
   - Connect **True** output of `Check Valid Request` → input of this node

9. **Create Set Node named `Create Response`:**  
   - Assign a field named `message` with the string `"Success! Workflow completed."` (customize as needed)  
   - Connect output of `Add workflow nodes here` → input of `Create Response`

10. **Create Respond to Webhook Node named `200 OK`:**  
    - Use default Response Code `200`  
    - Respond With: JSON (inherits data from `Create Response`)  
    - Connect output of `Create Response` → input of `200 OK`

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow is designed as a **secure, reusable webhook base** to protect public API endpoints.   | Workflow purpose                                                |
| Configure your secret token and required fields centrally in the `Configuration` node.              | Setup instructions                                              |
| Customize the `Create Response` node to tailor success responses, including forwarding request data.| Success response customization                                  |
| Useful for scenarios like public form submissions, internal API endpoints, and validating external service payloads. | Use cases                                                      |
| Author: Audun / xqus — My work: [xqus.com](https://xqus.com)                                        | Author credit                                                  |
| Support my work: [https://donate.stripe.com/9AQ6ps6Kna3t8Vi28b](https://donate.stripe.com/9AQ6ps6Kna3t8Vi28b) | Support link                                                  |

---

This documentation fully describes the workflow’s structure, logic, configuration, and reproduction steps, enabling advanced users and AI agents to understand, reproduce, and extend this secure webhook pattern efficiently.