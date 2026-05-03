Create a Secure MongoDB Data Retrieval API with Input Validation and HTTP Responses

https://n8nworkflows.xyz/workflows/create-a-secure-mongodb-data-retrieval-api-with-input-validation-and-http-responses-7674


# Create a Secure MongoDB Data Retrieval API with Input Validation and HTTP Responses

---

### 1. Workflow Overview

This n8n workflow exposes a secure, parameterized HTTP API endpoint to retrieve all documents from a specified MongoDB collection. It is designed for scenarios where clients need read-only access to MongoDB data via RESTful calls, with strict input validation to prevent unauthorized or malformed queries.

The logical blocks of the workflow are:

- **1.1 Input Reception and Parameter Extraction:** Accepts HTTP GET requests with a dynamic path parameter representing the MongoDB collection name.

- **1.2 Collection Name Validation:** Validates the collection name against a regex pattern to enforce security and naming conventions.

- **1.3 Conditional Branching on Validation Result:** Routes the workflow either to an error response or to the database query.

- **1.4 MongoDB Query Execution:** Queries the validated collection for all documents.

- **1.5 Data Transformation:** Renames MongoDB’s default `_id` field to `id` for cleaner output.

- **1.6 HTTP Response Delivery:** Sends the final JSON data back to the client, or a structured error message if validation fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Parameter Extraction

- **Overview:**  
  Defines an HTTP webhook endpoint that receives GET requests with a dynamic parameter `nameCollection` specifying the MongoDB collection to query.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: `Webhook` (HTTP entry point)  
    - Configuration:  
      - HTTP Method: GET (default for webhook node v2)  
      - Path: `:nameCollection` (captures collection name from URL)  
      - Response Mode: `responseNode` (response handled downstream)  
      - Bots ignored: false (accepts all requests)  
    - Input: HTTP request  
    - Output: JSON containing the parameter `nameCollection` under `params`  
    - Edge Cases: Missing or malformed `nameCollection` parameter; non-GET requests; network or routing failures.

---

#### 1.2 Collection Name Validation

- **Overview:**  
  Validates the received collection name against a regex pattern to ensure it does not begin with `system.` and contains only allowed characters, with a length between 1 and 120 characters.

- **Nodes Involved:**  
  - Validate Pattern (Code node)

- **Node Details:**  
  - **Validate Pattern**  
    - Type: `Code` (JavaScript code execution)  
    - Configuration:  
      - Extracts `nameCollection` from webhook parameters  
      - Regex used: `/^(?!system\.)[a-zA-Z0-9._]{1,120}$/`  
      - Returns an object with:  
        - `valid` (boolean) indicating validation success  
        - `collection` (string) the validated collection name if valid  
        - `message` (string) error description if invalid  
    - Inputs: Output from Webhook node  
    - Outputs: JSON object with validation result and message  
    - Edge Cases: Empty or missing `nameCollection`, input injection attempts, regex evaluation errors.

---

#### 1.3 Conditional Branching on Validation Result

- **Overview:**  
  Branches the workflow based on validation output: routes to MongoDB query if valid, or returns HTTP 400 error response if invalid.

- **Nodes Involved:**  
  - If  
  - Respond code 400

- **Node Details:**  
  - **If**  
    - Type: `If` node (conditional branching)  
    - Configuration:  
      - Condition: `$json.valid === true` (strict boolean check)  
      - True branch: proceed to MongoDB node  
      - False branch: proceed to Respond code 400  
    - Inputs: Output from Validate Pattern node  
    - Outputs: Two outputs, index 0 (true), index 1 (false)  
    - Edge Cases: Missing or malformed validation fields, expression evaluation errors.

  - **Respond code 400**  
    - Type: `Respond to Webhook`  
    - Configuration:  
      - HTTP Response Code: 400  
      - Response Body: JSON object `{ "code": 400, "message": "{{ $json.message }}" }`  
      - Responds with JSON content type  
    - Inputs: From If node false branch  
    - Outputs: Ends workflow response  
    - Edge Cases: Response delivery failure; missing message field.

---

#### 1.4 MongoDB Query Execution

- **Overview:**  
  Queries the specified MongoDB collection (validated) to retrieve all documents.

- **Nodes Involved:**  
  - MongoDB

- **Node Details:**  
  - **MongoDB**  
    - Type: `MongoDB`  
    - Configuration:  
      - Collection: dynamically set to `{{$json.collection}}` from validation node output  
      - No query filter specified (retrieves all documents)  
      - Credentials: MongoDB account configured externally (OAuth or connection string)  
    - Inputs: True branch from If node  
    - Outputs: MongoDB documents as JSON objects  
    - Edge Cases: Connection failures, authentication errors, empty collections, malformed collection names despite validation, query timeouts.

---

#### 1.5 Data Transformation

- **Overview:**  
  Transforms MongoDB documents by renaming the default `_id` field to `id` for API consumer convenience and consistency.

- **Nodes Involved:**  
  - IDS format (Code node)

- **Node Details:**  
  - **IDS format**  
    - Type: `Code`  
    - Configuration: JavaScript code that:  
      - Iterates over all input items (Mongo documents)  
      - For each object, if `_id` exists, creates a new `id` property with the same value and deletes `_id`  
      - Returns the transformed array of objects, each as a separate item  
    - Inputs: MongoDB query result  
    - Outputs: JSON array with `_id` replaced by `id`  
    - Edge Cases: Documents missing `_id`, large data volumes affecting performance, mutation side effects.

---

#### 1.6 HTTP Response Delivery

- **Overview:**  
  Sends the transformed MongoDB documents back to the client as a JSON HTTP response, completing the API request.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Respond to Webhook**  
    - Type: `Respond to Webhook`  
    - Configuration:  
      - Responds with all incoming items as JSON array  
      - Default HTTP status code 200  
    - Inputs: Output from IDS format node  
    - Outputs: HTTP response to original requester  
    - Edge Cases: Network timeouts, client disconnects, large payloads causing delays.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                          | Input Node(s)       | Output Node(s)       | Sticky Note                                      |
|-------------------|---------------------|----------------------------------------|---------------------|----------------------|-------------------------------------------------|
| Webhook           | Webhook             | HTTP Entry Point, receives collection name param | -                   | Validate Pattern     | ## 1 (Input Reception)                           |
| Validate Pattern   | Code                | Validates collection name format       | Webhook             | If                   | ## 2 (Validation Layer)                          |
| If                | If                  | Branches on validation result           | Validate Pattern     | MongoDB (true), Respond code 400 (false) | ## 3 (Validation Check)                          |
| Respond code 400   | Respond to Webhook  | Returns HTTP 400 error with message    | If (false branch)    | -                    | ## 3.A. Respond code 400 standard HTTP response |
| MongoDB           | MongoDB             | Queries MongoDB collection              | If (true branch)     | IDS format            | ## 4 (Data Retrieval)                            |
| IDS format        | Code                | Renames `_id` to `id` in Mongo documents | MongoDB             | Respond to Webhook    | ## 5 (Data Transformation)                       |
| Respond to Webhook | Respond to Webhook  | Sends final JSON response to client    | IDS format          | -                    | ## 6 (Final Response)                            |
| Sticky Note       | Sticky Note         | Documentation and explanations          | -                   | -                    | # Workflow Objective and steps                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook (v2)  
   - Name: `Webhook`  
   - HTTP Method: GET  
   - Path: `:nameCollection` (parameter capturing collection name)  
   - Response Mode: `responseNode` (response handled downstream)  
   - Save node.

2. **Create Code Node for Validation**  
   - Type: Code (JavaScript)  
   - Name: `Validate Pattern`  
   - Paste the following JS code:

   ```javascript
   const name = $input.first().json.params.nameCollection;
   const regex = /^(?!system\.)[a-zA-Z0-9._]{1,120}$/;

   if (!name) {
     return { valid: false, message: 'The name Collection cannot be empty.' };
   }

   if (!regex.test(name)) {
     let message = 'The name of the collection does not comply with the rules:';
     if (name.startsWith('system.')) {
       message += "You cannot start with 'system.'. ";
     }
     if (name.length < 1 || name.length > 120) {
       message += 'It must be between 1 and 120 characters long. ';
     }
     if (/[^a-zA-Z0-9._]/.test(name)) {
       message += 'Only letters, numbers, and underscores are allowed. (_) and point (.). ';
     }
     return { valid: false, message };
   }

   return { valid: true, collection: name };
   ```

   - Connect `Webhook` node output to this node input.

3. **Create If Node**  
   - Type: `If` (v2.2)  
   - Name: `If`  
   - Condition:  
     - Expression: `$json.valid === true`  
   - Connect output of `Validate Pattern` to `If` node.

4. **Create Respond to Webhook Node for 400 Error**  
   - Type: `Respond to Webhook` (v1.4)  
   - Name: `Respond code 400`  
   - Response Code: 400  
   - Respond With: JSON  
   - Response Body:

   ```json
   {
     "code": 400,
     "message": "{{ $json.message }}"
   }
   ```

   - Connect `If` node's **false** output to this node.

5. **Create MongoDB Node**  
   - Type: `MongoDB` (v1.2)  
   - Name: `MongoDB`  
   - Credentials: select or create your MongoDB credentials  
   - Collection: expression `{{$json.collection}}` (from validation node)  
   - Query: leave empty (fetch all documents)  
   - Connect `If` node's **true** output to this node.

6. **Create Code Node for ID Transformation**  
   - Type: `Code` (v2)  
   - Name: `IDS format`  
   - Paste this JavaScript code:

   ```javascript
   function replaceIdKey(arr) {
     return arr.map(item => {
       const newItem = { ...item };
       if (newItem._id !== undefined) {
         newItem.id = newItem._id;
         delete newItem._id;
       }
       return newItem;
     });
   }

   const inputArray = $input.all().map(i => i.json);
   const outputArray = replaceIdKey(inputArray);

   return outputArray.map(obj => ({ json: obj }));
   ```

   - Connect output of `MongoDB` node to this node.

7. **Create Final Respond to Webhook Node**  
   - Type: `Respond to Webhook` (v1.4)  
   - Name: `Respond to Webhook`  
   - Respond With: `allIncomingItems` (returns all items as JSON array)  
   - Connect output of `IDS format` node to this node.

8. **Verify Connections:**  
   - `Webhook` → `Validate Pattern` → `If`  
   - `If` true → `MongoDB` → `IDS format` → `Respond to Webhook` (final)  
   - `If` false → `Respond code 400`

9. **Configure Credentials:**  
   - MongoDB node requires valid connection credentials (URI, authentication) to your MongoDB instance.

10. **Activate and Test:**  
    - Activate the workflow.  
    - Test with HTTP GET requests to:  
      `https://<your-n8n-instance>/webhook/<your-webhook-id>/<collectionName>`  
    - Expect JSON array of documents with `_id` renamed to `id` if valid.  
    - Expect HTTP 400 error with message if invalid collection name.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Author: Samuel Heredia, LinkedIn profile: https://www.linkedin.com/in/samuel-heredia-2b5b7a98/ | Workflow originally authored by Samuel Heredia                                                           |
| Objective: Secure HTTP endpoint for MongoDB data retrieval with input validation and clean JSON responses | Workflow purpose and design rationale                                                                     |
| Validation regex explanation: Prevents querying system collections and enforces naming convention | Critical for security to avoid accidental or malicious access to reserved MongoDB collections             |
| MongoDB _id field renamed to id in output for API consistency and readability                   | Improves client-side usage and JSON clarity                                                               |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---