create single new masked email address with fastmail

https://n8nworkflows.xyz/workflows/create-single-new-masked-email-address-with-fastmail-2471


# create single new masked email address with fastmail

### 1. Workflow Overview

This workflow automates the creation of a masked (disposable) email address using the Fastmail API, triggered via an HTTP webhook. It is intended for users who want to generate privacy-preserving email aliases or conduct testing without exposing their primary email.

The workflow consists of the following logical blocks:

- **1.1 Webhook Input Reception**: Captures incoming POST requests containing optional parameters for the new masked email.
- **1.2 Session Retrieval**: Authenticates and retrieves session details from Fastmail to obtain necessary account information.
- **1.3 Parameter Preparation**: Extracts and defaults the input fields (`state` and `description`) from the webhook payload, preparing them for email creation.
- **1.4 Masked Email Creation**: Sends a structured POST request to Fastmail’s JMAP API to create the masked email using the prepared parameters.
- **1.5 Output Preparation and Response**: Extracts the resulting masked email and description, then responds to the original webhook call with these details.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Input Reception

- **Overview:**  
  Captures incoming HTTP POST requests at a fixed endpoint. Accepts JSON payload with optional fields `state` and `description` that control the masked email's attributes.

- **Nodes Involved:**  
  - `Webhook`

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `createMaskedEmail` (webhookId: `87f9abd1-2c9b-4d1f-8c7f-2261f4698c3c`)  
      - Response Mode: Uses a separate node for response (`Respond to Webhook`)  
    - Inputs: External HTTP POST request  
    - Outputs: Passes received JSON body downstream  
    - Edge Cases / Failures:  
      - Missing or malformed JSON body may cause downstream nodes to fail  
      - Unauthorized access if webhook is public and no auth is configured (recommend securing webhook)  
    - Version: n8n v2 webhook node

---

#### 2.2 Session Retrieval

- **Overview:**  
  Authenticates with Fastmail API using stored HTTP Header credentials to retrieve session information, specifically the primary account ID for masked emails.

- **Nodes Involved:**  
  - `Session`

- **Node Details:**

  - **Session**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://api.fastmail.com/jmap/session`  
      - Method: GET (default)  
      - Authentication: HTTP Header Authentication using stored Fastmail API key credential  
    - Inputs: Triggered by the webhook node  
    - Outputs: JSON containing session info with account IDs  
    - Key Expression: None directly; account ID extracted in downstream node  
    - Edge Cases / Failures:  
      - Authentication failure if API key invalid or expired  
      - Network or timeout errors  
      - API rate limits or unavailability  
    - Version: HTTP Request node v4.2

---

#### 2.3 Parameter Preparation

- **Overview:**  
  Extracts `state` and `description` from the webhook JSON payload, applying default values if missing. This prepares the parameters for masked email creation.

- **Nodes Involved:**  
  - `get fields for creation`

- **Node Details:**

  - **get fields for creation**  
    - Type: Set Node  
    - Configuration:  
      - Assigns two fields:  
        - `state`: Set to `Webhook` JSON body field `state` if present, else `"pending"`  
        - `description`: Set to `Webhook` JSON body field `description` if present, else `"Test via N8n"`  
    - Inputs: Takes session data output but only references webhook JSON body for defaults  
    - Outputs: JSON with `state` and `description` for masked email creation  
    - Edge Cases / Failures:  
      - If webhook JSON structure changes, expressions may fail  
      - Missing `state` or `description` handled by fallback defaults  
    - Version: Set node v3.4

---

#### 2.4 Masked Email Creation

- **Overview:**  
  Sends a POST request to Fastmail’s JMAP API to create a new masked email address, using the session’s account ID and the prepared parameters.

- **Nodes Involved:**  
  - `create random masked email`

- **Node Details:**

  - **create random masked email**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://api.fastmail.com/jmap/api/`  
      - Method: POST  
      - Headers: Content-Type: application/json  
      - Authentication: HTTP Header Authentication (same Fastmail API key credential)  
      - JSON Body: Includes:  
        - `"using"`: JMAP core and Fastmail masked email namespaces  
        - `"methodCalls"`: Calls `MaskedEmail/set` with:  
          - `accountId`: Dynamically extracted from `Session` node JSON: `primaryAccounts['https://www.fastmail.com/dev/maskedemail']`  
          - `create.maskedEmailId1`: object with `state` and `description` from previous node  
    - Inputs: Uses output from `get fields for creation` and `Session` nodes (via expression)  
    - Outputs: JMAP API response with details of created masked email  
    - Edge Cases / Failures:  
      - Invalid or missing accountId leads to API error  
      - Authentication failure  
      - Invalid JSON or unexpected API changes may cause failures  
      - Network issues, rate limits  
    - Version: HTTP Request node v4.2  
    - Notes:  
      - Refer to Fastmail JMAP API documentation:  
        https://api.fastmail.com/.well-known/jmap  
        https://api.fastmail.com/jmap/session

---

#### 2.5 Output Preparation and Response

- **Overview:**  
  Extracts the newly created masked email address and its description from the API response, then sends this data back as the webhook response.

- **Nodes Involved:**  
  - `prepare output`  
  - `Respond to Webhook`

- **Node Details:**

  - **prepare output**  
    - Type: Set Node  
    - Configuration:  
      - Assigns two fields:  
        - `email`: Extracted from `create random masked email` response at path:  
          `$json.methodResponses[0][1].created.maskedEmailId1.email`  
        - `desciption`: (note the typo in field name) copied from the `description` field in `get fields for creation` node  
    - Inputs: Takes response JSON from masked email creation node  
    - Outputs: JSON object with `email` and `desciption` (typo preserved)  
    - Edge Cases / Failures:  
      - Missing or malformed API response could cause extraction to fail  
      - Typo in field name may cause confusion downstream  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration:  
      - Response type: Text  
      - Response body: Passes entire JSON received from `prepare output`  
    - Inputs: Takes output from `prepare output` node  
    - Outputs: HTTP response to original POST request with created masked email info  
    - Edge Cases / Failures:  
      - If previous node outputs invalid JSON or is empty, response may be malformed  

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                     | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                                         |
|---------------------------|----------------------|-----------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook Trigger       | Receive HTTP POST to trigger flow | -                          | Session                     | ### Template Description<br>This n8n workflow template allows you to create a masked email address using the Fastmail API…         |
| Session                   | HTTP Request          | Retrieve Fastmail session info    | Webhook                    | get fields for creation      | ### Template Description (see above)                                                                                              |
| get fields for creation    | Set                   | Extract and default input fields  | Session                    | create random masked email   | ### Template Description (see above)                                                                                              |
| create random masked email | HTTP Request          | Create masked email via API       | get fields for creation     | prepare output               | https://api.fastmail.com/.well-known/jmap<br>https://api.fastmail.com/jmap/session<br>### Template Description (see above)         |
| prepare output            | Set                   | Extract masked email & description| create random masked email | Respond to Webhook           | ### Template Description (see above)                                                                                              |
| Respond to Webhook        | Respond to Webhook    | Send response back to caller      | prepare output             | -                           | ### Template Description (see above)                                                                                              |
| Sticky Note               | Sticky Note           | Documentation                     | -                          | -                           | ### Template Description<br>This n8n workflow template allows you to create a masked email address using the Fastmail API…         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook  
   - Name: `Webhook`  
   - HTTP Method: POST  
   - Path: `createMaskedEmail`  
   - Response Mode: `Response Node` (so response is handled separately)  

2. **Create HTTP Request Node for Session Retrieval**
   - Type: HTTP Request  
   - Name: `Session`  
   - HTTP Method: GET (default)  
   - URL: `https://api.fastmail.com/jmap/session`  
   - Authentication: HTTP Header Authentication  
     - Create/Use credential with Fastmail API token under HTTP Header Auth  
   - Connect `Webhook` output to `Session` input  

3. **Create Set Node to Extract Input Fields**
   - Type: Set  
   - Name: `get fields for creation`  
   - Add two fields:  
     - `state` (string): Expression: `{{$json["body"]["state"] ?? "pending"}}`  
     - `description` (string): Expression: `{{$json["body"]["description"] ?? "Test via N8n"}}`  
   - Connect `Session` output to this node  

4. **Create HTTP Request Node to Create Masked Email**
   - Type: HTTP Request  
   - Name: `create random masked email`  
   - HTTP Method: POST  
   - URL: `https://api.fastmail.com/jmap/api/`  
   - Headers: Add header `Content-Type: application/json`  
   - Authentication: Use same HTTP Header Auth credential as `Session` node  
   - Body Content Type: JSON  
   - JSON Body: Use the following template with expressions:

     ```json
     {
       "using": [
         "urn:ietf:params:jmap:core",
         "https://www.fastmail.com/dev/maskedemail"
       ],
       "methodCalls": [
         [
           "MaskedEmail/set",
           {
             "accountId": "{{ $('Session').item.json.primaryAccounts['https://www.fastmail.com/dev/maskedemail'] }}",
             "create": {
               "maskedEmailId1": {
                 "description": "{{ $json.description }}",
                 "state": "{{ $json.state }}"
               }
             }
           },
           "c1"
         ]
       ]
     }
     ```

   - Connect `get fields for creation` output to this node  

5. **Create Set Node to Prepare Output**
   - Type: Set  
   - Name: `prepare output`  
   - Add two fields:  
     - `email` (string): Expression: `{{$json.methodResponses[0][1].created.maskedEmailId1.email}}`  
     - `desciption` (string): Expression: `{{ $('get fields for creation').item.json.description }}`  
       *(Note: Keep the same typo in the field name for consistency)*  
   - Connect `create random masked email` output to this node  

6. **Create Respond to Webhook Node**
   - Type: Respond to Webhook  
   - Name: `Respond to Webhook`  
   - Response Mode: Default  
   - Respond with: Text  
   - Response Body: Expression: `{{ $json }}` (to send the JSON prepared)  
   - Connect `prepare output` node to this node  

7. **Connect All Nodes in Sequence:**

   ```
   Webhook -> Session -> get fields for creation -> create random masked email -> prepare output -> Respond to Webhook
   ```

8. **Credential Setup:**

   - Create a new HTTP Header Auth credential for Fastmail API with your API key.  
   - Assign this credential to both `Session` and `create random masked email` HTTP Request nodes.

9. **Optional: Secure Webhook**

   - Add authentication or IP restrictions on the webhook node if exposed publicly to prevent abuse.

10. **Testing**

    - Use a tool like `curl` or an HTTP client to POST JSON data to the webhook URL, e.g.:

      ```bash
      curl -X POST -H 'Content-Type: application/json' https://your-n8n-instance/webhook/createMaskedEmail -d '{"state": "pending", "description": "my mega fancy masked email"}'
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow template simplifies integrating Fastmail masked email functionality into n8n and can be extended for various privacy and testing use cases.                                                                  | Workflow description                                                                                |
| Fastmail JMAP API documentation: https://api.fastmail.com/.well-known/jmap and https://api.fastmail.com/jmap/session                                                                                                       | Referenced in `create random masked email` node notes                                              |
| Companion iOS/macOS Shortcut available to trigger this workflow easily: https://www.icloud.com/shortcuts/ac249b50eab34c04acd9fb522f9f7068                                                                                   | Useful for quick testing and usage                                                                 |
| Recommended securing of webhook endpoints exposed to the internet to prevent unauthorized use.                                                                                                                              | Security best practice                                                                               |
| Typo in `prepare output` node in field name: "desciption" instead of "description" – consider correcting if modifying workflow for clarity.                                                                                  | Minor correction that does not affect functionality                                                |

---

This document provides a comprehensive understanding of the workflow structure, node configurations, and reproduction instructions to facilitate customization, debugging, or extension.