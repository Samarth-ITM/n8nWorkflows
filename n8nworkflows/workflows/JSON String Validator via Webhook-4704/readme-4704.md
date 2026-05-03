JSON String Validator via Webhook

https://n8nworkflows.xyz/workflows/json-string-validator-via-webhook-4704


# JSON String Validator via Webhook

### 1. Workflow Overview

This workflow provides a simple JSON string validation service accessible via an HTTP webhook. Its main purpose is to receive a JSON string from an external client, validate whether the string is properly formatted JSON, and return the validation result immediately through the webhook response.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Listens for incoming HTTP POST requests containing a JSON payload with a `jsonString` property.
- **1.2 JSON Validation Logic:** Executes custom JavaScript code to parse and validate the string as JSON.
- **1.3 Response Delivery:** Returns the validation result (validity status and error message if any) back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives incoming HTTP POST requests on a specific webhook endpoint. It expects the request body to include a property named `jsonString` which contains the string to be validated.

**Nodes Involved:**  
- Webhook: Receive JSON String  
- Note: Webhook Input (documentation node)

**Node Details:**

- **Webhook: Receive JSON String**  
  - *Type:* Webhook node  
  - *Role:* Entry point of the workflow, listens for HTTP POST requests at the path `/validate-json-string`.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Response Mode: `responseNode` (defers response until later node)  
    - Webhook Path: `validate-json-string`  
  - *Expressions/Variables:* Accesses incoming request body at `json.body`  
  - *Connections:* Output to "Code: Validate JSON String" node  
  - *Edge Cases/Potential Failures:*  
    - Missing or malformed HTTP request body  
    - Non-POST HTTP methods ignored  
  - *Sticky Notes:*  
    - Explains expected input format: a JSON body with property `jsonString` (string to validate)

- **Note: Webhook Input**  
  - *Type:* Sticky Note (documentation node)  
  - *Role:* Documents the expected input format and usage of the webhook node  

---

#### 1.2 JSON Validation Logic

**Overview:**  
This block evaluates whether the provided string is valid JSON by attempting to parse it. It returns an object indicating success (`valid: true`) or failure (`valid: false`) along with an error message if invalid.

**Nodes Involved:**  
- Code: Validate JSON String  
- Note: JSON Validation Logic (documentation node)

**Node Details:**

- **Code: Validate JSON String**  
  - *Type:* Code node (JavaScript execution)  
  - *Role:* Performs the core validation logic by parsing the `jsonString` field from the webhook payload.  
  - *Configuration:*  
    - Custom JavaScript code:  
      - Iterates over all incoming items (usually one)  
      - Checks if `jsonString` exists and is a string  
      - Attempts to parse `jsonString` with `JSON.parse()`  
      - On success: outputs `{ valid: true }`  
      - On failure or missing input: outputs `{ valid: false, error: <message> }`  
  - *Key Expressions/Variables:*  
    - Reads `item.json.body.jsonString`  
    - Returns an array of JSON objects with validation results  
  - *Connections:* Output to "Respond to Webhook with Result" node  
  - *Edge Cases/Potential Failures:*  
    - Input missing `jsonString` property  
    - `jsonString` not a string type  
    - Malformed JSON string causing parsing exceptions  
    - Unexpected input structure  
  - *Sticky Notes:*  
    - Describes the validation logic and output format

- **Note: JSON Validation Logic**  
  - *Type:* Sticky Note  
  - *Role:* Documents the validation functionality and expected outputs  

---

#### 1.3 Response Delivery

**Overview:**  
This block sends the validation result back to the caller of the webhook, closing the HTTP request with the validation outcome.

**Nodes Involved:**  
- Respond to Webhook with Result  
- Note: Webhook Response (documentation node)

**Node Details:**

- **Respond to Webhook with Result**  
  - *Type:* Respond to Webhook node  
  - *Role:* Sends the output from the validation code node back as the HTTP response to the webhook caller.  
  - *Configuration:*  
    - Respond With: `allIncomingItems` (returns full JSON output from previous node)  
  - *Connections:* No outputs (terminal node)  
  - *Edge Cases/Potential Failures:*  
    - If no previous node output available, response may be empty or cause error  
    - Network or client disconnects before response is sent  
  - *Sticky Notes:*  
    - Explains this node returns the validation result to the system invoking the webhook

- **Note: Webhook Response**  
  - *Type:* Sticky Note  
  - *Role:* Documents the purpose and behavior of the response node  

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role            | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                     |
|-----------------------------|-------------------------|----------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Webhook: Receive JSON String | Webhook                 | Input Reception            |                              | Code: Validate JSON String      | This node listens for incoming POST requests. It expects a JSON body containing a single property: `jsonString` (the string you wish to validate as JSON). |
| Note: Webhook Input         | Sticky Note             | Documentation              |                              |                               | This node listens for incoming POST requests. It expects a JSON body containing a single property: `jsonString` (the string you wish to validate as JSON).   |
| Code: Validate JSON String  | Code                    | JSON Validation Logic     | Webhook: Receive JSON String  | Respond to Webhook with Result  | This node contains custom JavaScript code to parse the `jsonString` from the webhook input. It returns `valid: true` if successful, or `valid: false` with an `error` message if parsing fails. |
| Note: JSON Validation Logic | Sticky Note             | Documentation              |                              |                               | This node contains custom JavaScript code to parse the `jsonString` from the webhook input. It returns `valid: true` if successful, or `valid: false` with an `error` message if parsing fails. |
| Respond to Webhook with Result | Respond to Webhook    | Response Delivery          | Code: Validate JSON String    |                               | This node sends the validation result (whether the `jsonString` was valid JSON or not, including an error if applicable) back to the system that triggered the webhook. |
| Note: Webhook Response      | Sticky Note             | Documentation              |                              |                               | This node sends the validation result (whether the `jsonString` was valid JSON or not, including an error if applicable) back to the system that triggered the webhook. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a new **Webhook** node named `Webhook: Receive JSON String`.  
   - Set HTTP Method to `POST`.  
   - Set Webhook Path to `validate-json-string`.  
   - Set Response Mode to `responseNode`.  
   - Leave other options as default.

2. **Create Code Node**  
   - Add a **Code** node named `Code: Validate JSON String`.  
   - Insert the following JavaScript code:

     ```javascript
     const results = [];

     for (const item of $input.all()) {
       try {
         if (item.json.body && typeof item.json.body.jsonString === 'string') {
           JSON.parse(item.json.body.jsonString);
           results.push({ json: { valid: true } });
         } else {
           results.push({ json: { valid: false, error: "Input 'jsonString' is missing or not a string." } });
         }
       } catch (e) {
         results.push({ json: { valid: false, error: e.message } });
       }
     }

     return results;
     ```
   - Ensure **Type Version** is set to 2.

3. **Create Respond to Webhook Node**  
   - Add a **Respond to Webhook** node named `Respond to Webhook with Result`.  
   - Set `Respond With` option to `allIncomingItems`.  
   - Leave other options default.

4. **Connect Nodes**  
   - Connect the output of `Webhook: Receive JSON String` node to the input of `Code: Validate JSON String` node.  
   - Connect the output of `Code: Validate JSON String` node to the input of `Respond to Webhook with Result` node.

5. **Add Sticky Notes (Optional but Recommended for Documentation)**  
   - Add a sticky note near the webhook node describing expected input: `This node listens for incoming POST requests. It expects a JSON body containing a single property: jsonString (the string you wish to validate as JSON).`  
   - Add a sticky note near the code node describing the validation logic and output.  
   - Add a sticky note near the respond node explaining that it returns the validation result back to the webhook caller.

6. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test by sending a POST request to the webhook URL appended with `/validate-json-string` with JSON body:  
     ```json
     { "jsonString": "{\"key\": \"value\"}" }
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates a pattern for validating JSON strings received via HTTP requests and returning immediate validation results. | Useful pattern for API endpoints requiring JSON format validation.                                             |
| The webhook node uses `responseNode` mode to enable synchronous HTTP responses after workflow processing completes.   | n8n Webhook documentation: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                                  |
| Custom code node provides flexibility but must handle potential missing or malformed inputs gracefully.                | JavaScript `JSON.parse()` exceptions are caught and returned as error messages.                                |
| Sticky notes are used to document the workflow inline, improving maintainability and clarity for future users.         | Best practice for n8n workflows with external integrations.                                                    |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and does not contain any illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.