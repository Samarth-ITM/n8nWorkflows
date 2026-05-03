Tutorial - Creating a Secure Webhook

https://n8nworkflows.xyz/workflows/tutorial---creating-a-secure-webhook-5174


# Tutorial - Creating a Secure Webhook

### 1. Workflow Overview

This workflow demonstrates how to create a **secure webhook** endpoint that authenticates incoming requests using an API key mechanism. It simulates a small, in-memory "database" of registered API keys and verifies requests against this list before responding appropriately.

The workflow is logically divided into the following blocks:

- **1.1 Public Endpoint & Request Reception**: Listens for incoming HTTP POST requests with an `x-api-key` header.
- **1.2 API Key Verification (Mock Database)**: Simulates a database lookup by splitting and filtering a static list of registered API keys.
- **1.3 Key Validation and Response**: Uses conditional logic to check if the API key is valid and sends success or unauthorized HTTP responses accordingly.
- **1.4 Testing Setup**: Contains an HTTP Request node to test the secured webhook with different API keys.

---

### 2. Block-by-Block Analysis

#### 1.1 Public Endpoint & Request Reception

- **Overview:**  
  This block exposes a public webhook endpoint that listens for POST requests. It expects an `x-api-key` header for authentication. Incoming requests are forwarded to the API key verification block.

- **Nodes Involved:**  
  - Secured Webhook  
  - Check API Key  
  - Sticky Note (Public Endpoint & Tester)  
  - Test Secure Webhook  

- **Node Details:**

  - **Secured Webhook**  
    - Type: Webhook  
    - Role: Receives incoming POST HTTP requests at path `/tutorial/secure-webhook`  
    - Configuration:  
      - HTTP Method: POST  
      - Authentication: Header-based (expects an `x-api-key` header)  
      - Response Mode: Controlled by response nodes downstream  
      - Credentials: Uses HTTP Header Auth credential for protection of private webhook access  
    - Inputs: External HTTP requests from clients  
    - Outputs: Passes request data to "Check API Key" node  
    - Potential Failures: Missing or malformed headers, webhook not reachable  
    - Version: 2  

  - **Check API Key**  
    - Type: HTTP Request  
    - Role: Sends the received API key to the private webhook acting as the mock database for validation  
    - Configuration:  
      - URL: Dynamically constructed from environment variables, pointing to `/tutorial/secure-webhook/api-keys` webhook  
      - Sends body containing the extracted `x-api-key` header value  
      - Authenticated using HTTP Header Auth credentials  
      - On error: continues regular output without failing the workflow  
    - Inputs: Receives data from "Secured Webhook" node  
    - Outputs: Forwards response to "API Key Identified" node  
    - Edge Cases: Network errors, authentication failures, invalid responses from API key webhook  
    - Version: 4.2  

  - **Test Secure Webhook**  
    - Type: HTTP Request  
    - Role: Used to test the secured webhook endpoint by sending POST requests with configurable `x-api-key` headers  
    - Configuration:  
      - URL: Points to the public webhook path `/tutorial/secure-webhook`  
      - HTTP Method: POST  
      - Sends header `x-api-key` with test values (default: "test")  
      - Authenticated with HTTP Header Auth credentials  
    - Inputs: Manual trigger (not connected)  
    - Outputs: None  
    - Edge Cases: Invalid URL, incorrect header setup  

  - **Sticky Note1** (Public Endpoint & Tester)  
    - Explains the purpose of the public webhook and test setup  

#### 1.2 API Key Verification (Mock Database)

- **Overview:**  
  Simulates a database storing registered API keys. It receives a key from the public webhook, splits the internal list of keys, and filters to find a matching key.

- **Nodes Involved:**  
  - Get API Key  
  - Registered API Keys  
  - Split Out Users  
  - Find API Key  
  - Sticky Note2  

- **Node Details:**

  - **Get API Key**  
    - Type: Webhook  
    - Role: Private webhook that receives API key POST requests for verification at `/tutorial/secure-webhook/api-keys`  
    - Configuration:  
      - HTTP Method: POST  
      - Authentication: Header-based (same HTTP Header Auth credential)  
      - Response Mode: Last node output (returns filtered user data)  
    - Inputs: Receives API key from "Check API Key" node  
    - Outputs: Sends data to "Registered API Keys" node  
    - Edge Cases: Missing or invalid API key, unauthorized access  
    - Version: 2  

  - **Registered API Keys**  
    - Type: Set  
    - Role: Stores a static array of user IDs and their corresponding API keys (mock database)  
    - Configuration:  
      - Sets variable `registered_api_keys` as an array of objects with `user_id` and `api_key`  
      - Editable to add/remove API keys  
    - Inputs: Receives execution trigger from "Get API Key" node  
    - Outputs: Passes data to "Split Out Users" node  
    - Edge Cases: Incorrect data format or duplicates  
    - Version: 3.4  

  - **Split Out Users**  
    - Type: Split Out  
    - Role: Splits the array `registered_api_keys` into individual items to process each user separately  
    - Configuration:  
      - Field to split: `registered_api_keys`  
    - Inputs: Receives array from "Registered API Keys" node  
    - Outputs: Passes individual user objects to "Find API Key" node  
    - Edge Cases: Empty array, null values  
    - Version: 1  

  - **Find API Key**  
    - Type: Filter  
    - Role: Filters the split user records to find one where the `api_key` matches the incoming key  
    - Configuration:  
      - Condition: `api_key` equals the incoming API key (from the last "Get API Key" webhook call)  
    - Inputs: Receives split user items from "Split Out Users" node  
    - Outputs: Outputs matched user or empty if no match  
    - Edge Cases: No matches found, case sensitivity issues  
    - Version: 2.2  

  - **Sticky Note2**  
    - Describes the mock database simulation and editing instructions for API keys  

#### 1.3 Key Validation and Response

- **Overview:**  
  Validates if the API key corresponds to a registered user and sends either a success or unauthorized response.

- **Nodes Involved:**  
  - API Key Identified  
  - Respond to Webhook (success)  
  - Respond to Webhook (unauthorized)  
  - Sticky Note3  

- **Node Details:**

  - **API Key Identified**  
    - Type: IF (Conditional)  
    - Role: Checks if the API key from the verified user exists and matches the header value  
    - Configuration:  
      - Condition: Checks if `user_id` exists and matches the `x-api-key` header from the original "Secured Webhook" request  
      - Case sensitive and strict type validation enabled  
    - Inputs: Receives response from "Check API Key" node (which has the user info)  
    - Outputs:  
      - True branch leads to "Respond to Webhook (success)"  
      - False branch leads to "Respond to Webhook (unauthorized)"  
    - Edge Cases: Missing user ID, mismatch, header absence  
    - Version: 2.2  

  - **Respond to Webhook (success)**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 response with JSON body indicating success and the `user_id`  
    - Configuration:  
      - Responds with JSON  
      - Response body includes `"status": "success"` and `"user_id"` from matched user  
    - Inputs: True branch from "API Key Identified" node  
    - Outputs: Ends workflow response  
    - Edge Cases: Missing user info leading to incomplete response  
    - Version: 1.4  

  - **Respond to Webhook (unauthorized)**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 401 Unauthorized response with JSON error message  
    - Configuration:  
      - Response code: 401  
      - Responds with JSON error message `"Please provide a valid x-api-key header."`  
    - Inputs: False branch from "API Key Identified" node  
    - Outputs: Ends workflow response  
    - Edge Cases: None significant; fallback for invalid keys  
    - Version: 1.4  

  - **Sticky Note3**  
    - Explains the gatekeeping logic of the IF node  

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                                      | Input Node(s)        | Output Node(s)               | Sticky Note                                                                                                                                |
|----------------------------|------------------------|-----------------------------------------------------|----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Secured Webhook            | Webhook                | Public webhook endpoint receiving POST with API key | (External HTTP)       | Check API Key               | ### ▶️ Public Endpoint & Tester\n\n*   **`Secured Webhook`**: This is your public-facing endpoint. It listens for requests containing an `x-api-key` header.\n*   **`Test Secure Webhook`**: Use this node to test the endpoint. Change the `x-api-key` header value to test valid and invalid keys. |
| Check API Key              | HTTP Request           | Sends API key to private webhook for validation      | Secured Webhook       | API Key Identified          |                                                                                                                                            |
| API Key Identified         | IF                     | Validates presence and correctness of user for key  | Check API Key         | Respond to Webhook (success), Respond to Webhook (unauthorized) | #### ✅ Gatekeeper\n\nThis IF node checks the result from our "database".\n\n*   If a user was found for the given API key, it proceeds to the **success** response.\n*   If not, it sends a **401 Unauthorized** error. |
| Respond to Webhook (success) | Respond to Webhook      | Sends success HTTP response with user ID             | API Key Identified (true) | (Ends workflow)             |                                                                                                                                            |
| Respond to Webhook (unauthorized) | Respond to Webhook      | Sends 401 Unauthorized HTTP response                  | API Key Identified (false) | (Ends workflow)             |                                                                                                                                            |
| Get API Key                | Webhook                | Private webhook to receive API key for validation    | (Check API Key)       | Registered API Keys         | ### 📦 Mock Database\n\nThese nodes simulate a database of users and their API keys.\n\n*   **`Get API Key`**: A private webhook that receives a key and checks it against the list.\n*   **`Registered API Keys`**: **EDIT THIS NODE** to add or remove the API keys you want to be considered valid. Each key should be unique to a user. |
| Registered API Keys        | Set                    | Stores static array of registered API keys            | Get API Key           | Split Out Users             | ### 📦 Mock Database\n\nThese nodes simulate a database of users and their API keys.\n\n*   **`Get API Key`**: A private webhook that receives a key and checks it against the list.\n*   **`Registered API Keys`**: **EDIT THIS NODE** to add or remove the API keys you want to be considered valid. Each key should be unique to a user. |
| Split Out Users            | Split Out              | Splits API keys array to individual items             | Registered API Keys   | Find API Key                | ### 📦 Mock Database\n\nThese nodes simulate a database of users and their API keys.\n\n*   **`Get API Key`**: A private webhook that receives a key and checks it against the list.\n*   **`Registered API Keys`**: **EDIT THIS NODE** to add or remove the API keys you want to be considered valid. Each key should be unique to a user. |
| Find API Key               | Filter                 | Filters users to find matching API key                 | Split Out Users       | (To Check API Key via HTTP) | ### 📦 Mock Database\n\nThese nodes simulate a database of users and their API keys.\n\n*   **`Get API Key`**: A private webhook that receives a key and checks it against the list.\n*   **`Registered API Keys`**: **EDIT THIS NODE** to add or remove the API keys you want to be considered valid. Each key should be unique to a user. |
| Test Secure Webhook        | HTTP Request           | Sends test requests to public webhook                  | (Manual trigger)      |                             | ### ▶️ Public Endpoint & Tester\n\n*   **`Secured Webhook`**: This is your public-facing endpoint. It listens for requests containing an `x-api-key` header.\n*   **`Test Secure Webhook`**: Use this node to test the endpoint. Change the `x-api-key` header value to test valid and invalid keys. |
| Sticky Note1               | Sticky Note            | Explains the key verification logic                    |                      |                             | ### ⚙️ Key Verification Logic\n\nThis node takes the API key from the incoming request and asks our "database" (the second webhook) if it's valid.\n\n**In a real-world scenario, you would replace this and the nodes below with a single Database node (e.g., Supabase, Postgres) to perform the lookup.** |
| Sticky Note2               | Sticky Note            | Describes mock database nodes                           |                      |                             | ### 📦 Mock Database\n\nThese nodes simulate a database of users and their API keys.\n\n*   **`Get API Key`**: A private webhook that receives a key and checks it against the list.\n*   **`Registered API Keys`**: **EDIT THIS NODE** to add or remove the API keys you want to be considered valid. Each key should be unique to a user. |
| Sticky Note3               | Sticky Note            | Describes gatekeeper IF node logic                      |                      |                             | #### ✅ Gatekeeper\n\nThis IF node checks the result from our "database".\n\n*   If a user was found for the given API key, it proceeds to the **success** response.\n*   If not, it sends a **401 Unauthorized** error. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Public Webhook Node**  
   - Node Type: Webhook  
   - Name: `Secured Webhook`  
   - HTTP Method: POST  
   - Path: `tutorial/secure-webhook`  
   - Authentication: HTTP Header Authentication  
   - Configure HTTP Header Auth credential (must have valid credentials set up in n8n)  
   - Response Mode: `Response Node`  

2. **Create the HTTP Request Node to Validate API Key**  
   - Node Type: HTTP Request  
   - Name: `Check API Key`  
   - HTTP Method: POST  
   - URL: Construct dynamically as `{{$env.WEBHOOK_URL + ($env.N8N_ENDPOINT_WEBHOOK ?? "webhook")}}/tutorial/secure-webhook/api-keys`  
   - Authentication: HTTP Header Authentication (same credential as webhook)  
   - Body Parameters: Include `"api_key"` with value from expression: `{{$json.headers['x-api-key']}}`  
   - On Error: Continue Regular Output (to prevent workflow failure on errors)  

3. **Connect**: `Secured Webhook` → `Check API Key`  

4. **Create Private API Key Webhook**  
   - Node Type: Webhook  
   - Name: `Get API Key`  
   - HTTP Method: POST  
   - Path: `tutorial/secure-webhook/api-keys`  
   - Authentication: HTTP Header Authentication (same credential)  
   - Response Mode: `Last Node`  

5. **Create Set Node with Registered API Keys**  
   - Node Type: Set  
   - Name: `Registered API Keys`  
   - Add a new field:  
     - Name: `registered_api_keys`  
     - Type: Array  
     - Value: JSON array of objects, each with `user_id` and `api_key`. Example:  
       ```json
       [
         {"user_id":"user_1","api_key":"test"},
         {"user_id":"user_2","api_key":"sk-lihefoihz12121ZFzk124zehfAZJAOJZ14joEKe1h"}
       ]
       ```  
   - Connect: `Get API Key` → `Registered API Keys`  

6. **Create Split Out Node**  
   - Node Type: Split Out  
   - Name: `Split Out Users`  
   - Field to Split Out: `registered_api_keys`  
   - Connect: `Registered API Keys` → `Split Out Users`  

7. **Create Filter Node to Find Matching API Key**  
   - Node Type: Filter  
   - Name: `Find API Key`  
   - Condition:  
     - Field: `api_key`  
     - Operator: Equals  
     - Value: Expression: `{{$node["Get API Key"].json["body"]["api_key"]}}`  
   - Connect: `Split Out Users` → `Find API Key`  

8. **Create IF Node to Validate User**  
   - Node Type: IF  
   - Name: `API Key Identified`  
   - Condition:  
     - Check if `user_id` exists (operation: exists)  
     - And `user_id` equals the header `x-api-key` from original request  
       - Left value: `{{$json.user_id}}`  
       - Right value: `{{$node["Secured Webhook"].item.json.headers["x-api-key"]}}`  
   - Connect: `Check API Key` → `API Key Identified`  

9. **Create Success Response Node**  
   - Node Type: Respond to Webhook  
   - Name: `Respond to Webhook (success)`  
   - Response Code: Default 200  
   - Respond With: JSON  
   - Response Body:  
     ```json
     {
       "status": "success",
       "user_id": "={{ $json.user_id }}"
     }
     ```  
   - Connect: `API Key Identified` (true) → `Respond to Webhook (success)`  

10. **Create Unauthorized Response Node**  
    - Node Type: Respond to Webhook  
    - Name: `Respond to Webhook (unauthorized)`  
    - Response Code: 401  
    - Respond With: JSON  
    - Response Body:  
      ```json
      {
        "error": "Please provide a valid x-api-key header."
      }
      ```  
    - Connect: `API Key Identified` (false) → `Respond to Webhook (unauthorized)`  

11. **Create Test HTTP Request Node**  
    - Node Type: HTTP Request  
    - Name: `Test Secure Webhook`  
    - Method: POST  
    - URL: `{{$env.WEBHOOK_URL + ($env.N8N_ENDPOINT_WEBHOOK ?? "webhook")}}/tutorial/secure-webhook`  
    - Send Headers: true  
    - Header: `x-api-key` with test value (e.g., `test`)  
    - Authentication: HTTP Header Auth (same credential)  

12. **Set up Credentials**  
    - Create or reuse HTTP Header Auth credential with a secret to protect private webhooks and test requests  
    - Assign this credential to all relevant nodes requiring authentication (webhooks, HTTP requests)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| This workflow simulates API key validation using static data. In production, replace the mock database with an actual database (e.g., Supabase, Postgres) or an external auth service for better security and scalability. | Sticky Note1 and Sticky Note2 describing the mock database and key verification logic                                                    |
| The `Test Secure Webhook` node allows quick verification of the webhook endpoint by sending test requests with configurable API keys.                                                                                       | Sticky Note1 and Sticky Note (Public Endpoint & Tester)                                                                                  |
| The IF node acts as a gatekeeper, ensuring only valid keys receive successful responses, otherwise returning 401 Unauthorized.                                                                                                | Sticky Note3                                                                                                                            |
| Keep API keys confidential and rotate them periodically. Use HTTPS for all webhook endpoints to secure data in transit.                                                                                                      | General best practice                                                                                                                    |
| For further advanced implementations, consider integrating n8n credentials with external vaults or secret managers to secure API keys and secrets.                                                                           | Recommended security best practices                                                                                                      |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow and fully complies with current content policies. It contains no illegal, offensive, or protected content. All data processed are legal and publicly accessible.