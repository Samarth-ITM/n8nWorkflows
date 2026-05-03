Bitrix24 Chatbot Application Workflow example with Webhook Integration

https://n8nworkflows.xyz/workflows/bitrix24-chatbot-application-workflow-example-with-webhook-integration-2923


# Bitrix24 Chatbot Application Workflow example with Webhook Integration

### 1. Workflow Overview

This workflow automates chatbot interactions within Bitrix24 by processing incoming webhook events and responding appropriately. It is designed to handle multiple event types such as messages sent to the bot, the bot joining chats, app installation events, and bot deletion notifications. The workflow ensures secure communication by validating tokens and managing authentication credentials. It also supports bot registration and sending automated responses back to Bitrix24.

Logical blocks in the workflow:

- **1.1 Input Reception:** Receives webhook POST requests from Bitrix24.
- **1.2 Authentication & Validation:** Extracts and validates authentication tokens from incoming requests.
- **1.3 Event Routing:** Routes the workflow logic based on the event type received.
- **1.4 Event Processing:** Contains separate processing logic for message events, join chat events, app installation, and bot deletion.
- **1.5 Outbound Communication:** Sends HTTP requests back to Bitrix24 to register the bot or send messages.
- **1.6 Response Handling:** Sends success or error responses back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming webhook POST requests from Bitrix24 at a specified endpoint and passes the raw data downstream for processing.

- **Nodes Involved:**  
  - Bitrix24 Handler

- **Node Details:**

  - **Bitrix24 Handler**  
    - Type: Webhook  
    - Role: Entry point for Bitrix24 webhook POST requests.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `bitrix24/handler.php`  
      - Response Mode: Uses a response node downstream to send replies.  
    - Input: External HTTP POST from Bitrix24.  
    - Output: JSON payload containing event data and authentication info.  
    - Edge cases:  
      - Invalid or malformed webhook requests.  
      - Missing required fields in the payload.  
    - Version: n8n webhook node v1.

#### 1.2 Authentication & Validation

- **Overview:**  
  Extracts authentication tokens and client credentials from the webhook payload and validates the application token to ensure secure communication.

- **Nodes Involved:**  
  - Credentials  
  - Validate Token  
  - Error Response

- **Node Details:**

  - **Credentials**  
    - Type: Set  
    - Role: Extracts and assigns authentication and client credentials from the incoming JSON payload to workflow variables.  
    - Configuration:  
      - Assigns CLIENT_ID and CLIENT_SECRET with static values.  
      - Extracts application_token, domain, and access_token dynamically from the webhook JSON body.  
      - Includes other fields from the input JSON.  
    - Input: Output from Bitrix24 Handler.  
    - Output: JSON enriched with credentials for validation.  
    - Edge cases:  
      - Missing or malformed auth fields in the webhook payload.

  - **Validate Token**  
    - Type: If  
    - Role: Validates that the application token from the webhook matches the expected CLIENT_ID or allows a default pass-through.  
    - Configuration:  
      - Condition: Checks if CLIENT_ID equals application_token OR always true (1==1) as fallback.  
      - If true: proceeds to event routing.  
      - If false: triggers error response.  
    - Input: Credentials node output.  
    - Output: Two branches — valid token or invalid token.  
    - Edge cases:  
      - Token mismatch leads to unauthorized error response.  
      - Expression evaluation failures.

  - **Error Response**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 401 Unauthorized response with JSON error message if token validation fails.  
    - Configuration:  
      - Response code: 401  
      - Response body: JSON with result=false and error message.  
    - Input: Validate Token (false branch).  
    - Output: Terminates workflow with error response.

#### 1.3 Event Routing

- **Overview:**  
  Routes the workflow execution path based on the event type specified in the webhook payload.

- **Nodes Involved:**  
  - Route Event

- **Node Details:**

  - **Route Event**  
    - Type: Switch  
    - Role: Directs workflow to different processing nodes depending on the event type in `$json.body.event`.  
    - Configuration:  
      - Routes for events:  
        - ONIMBOTMESSAGEADD → Process Message  
        - ONIMBOTJOINCHAT → Process Join  
        - ONAPPINSTALL → Process Install  
        - ONIMBOTDELETE → Process Delete  
      - Case sensitive, strict string matching.  
    - Input: Validate Token (true branch).  
    - Output: Multiple outputs, one per event type.  
    - Edge cases:  
      - Unknown or unsupported event types result in no output (no handling).  
      - Expression evaluation errors.

#### 1.4 Event Processing

- **Overview:**  
  Contains logic to handle each event type: processing messages, welcoming on join, registering the bot on install, and handling bot deletion.

- **Nodes Involved:**  
  - Process Message  
  - Process Join  
  - Process Install  
  - Process Delete

- **Node Details:**

  - **Process Message**  
    - Type: Function  
    - Role: Processes incoming chat messages and prepares a response.  
    - Configuration:  
      - Extracts message text and dialog ID from webhook data.  
      - Checks if message equals "what's hot" (case insensitive).  
      - If yes, replies with a fixed greeting message.  
      - Otherwise, echoes back the user's message prefixed with "You said:".  
      - Passes along auth tokens and domain for downstream use.  
    - Input: Route Event (ONIMBOTMESSAGEADD).  
    - Output: JSON with DIALOG_ID, MESSAGE, AUTH, DOMAIN.  
    - Edge cases:  
      - Missing message or dialog ID fields.  
      - Case sensitivity in message matching.

  - **Process Join**  
    - Type: Function  
    - Role: Handles bot joining a chat by sending a welcome message.  
    - Configuration:  
      - Extracts dialog ID from webhook data.  
      - Sends a fixed welcome message.  
      - Passes auth tokens and domain.  
    - Input: Route Event (ONIMBOTJOINCHAT).  
    - Output: JSON with DIALOG_ID, MESSAGE, AUTH, DOMAIN.  
    - Edge cases:  
      - Missing dialog ID.

  - **Process Install**  
    - Type: Function  
    - Role: Prepares bot registration data upon app installation event.  
    - Configuration:  
      - Extracts webhook URL and auth tokens from input.  
      - Constructs registration payload including bot properties (name, color, email, etc.) and event URLs.  
      - Passes auth tokens and domain.  
    - Input: Route Event (ONAPPINSTALL).  
    - Output: JSON with registration parameters.  
    - Edge cases:  
      - Missing webhook URL or auth tokens.

  - **Process Delete**  
    - Type: NoOp  
    - Role: Placeholder for handling bot deletion events; no processing done.  
    - Input: Route Event (ONIMBOTDELETE).  
    - Output: Passes through to success response.  
    - Edge cases: None.

#### 1.5 Outbound Communication

- **Overview:**  
  Sends HTTP requests back to Bitrix24 to register the bot or send messages based on processed data.

- **Nodes Involved:**  
  - Register Bot  
  - Send Message  
  - Send Join Message

- **Node Details:**

  - **Register Bot**  
    - Type: HTTP Request  
    - Role: Registers the bot with Bitrix24 using the REST API.  
    - Configuration:  
      - URL constructed dynamically using domain and auth token.  
      - POST method with body parameters including bot code, type, event URLs, properties, client ID, and client secret.  
      - Sends JSON body with bot registration details.  
    - Input: Process Install output.  
    - Output: Passes to success response.  
    - Edge cases:  
      - HTTP errors (network, auth failure).  
      - Invalid or missing parameters.

  - **Send Message**  
    - Type: HTTP Request  
    - Role: Sends chat messages back to Bitrix24 via REST API.  
    - Configuration:  
      - URL dynamically built with domain and auth token.  
      - POST method with dialog ID and message text in body.  
    - Input: Process Message output.  
    - Output: Passes to success response.  
    - Edge cases:  
      - HTTP request failures.  
      - Invalid dialog ID or message content.

  - **Send Join Message**  
    - Type: HTTP Request  
    - Role: Sends welcome messages when the bot joins a chat.  
    - Configuration:  
      - Same as Send Message node but triggered by join event.  
    - Input: Process Join output.  
    - Output: Passes to success response.  
    - Edge cases: Same as Send Message.

#### 1.6 Response Handling

- **Overview:**  
  Sends HTTP 200 success responses back to Bitrix24 after successful processing of events.

- **Nodes Involved:**  
  - Success Response

- **Node Details:**

  - **Success Response**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 OK with JSON `{ "result": true }` to acknowledge successful processing.  
    - Configuration:  
      - Response code: 200  
      - Response body: JSON with result true.  
    - Input: Outputs from Register Bot, Send Message, Send Join Message, and Process Delete.  
    - Output: Terminates workflow with success response.  
    - Edge cases: None.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                      |
|-------------------|---------------------|----------------------------------------|------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Bitrix24 Handler  | Webhook             | Receives webhook POST requests         | External HTTP POST      | Credentials              |                                                                                                |
| Credentials       | Set                 | Extracts and assigns auth credentials  | Bitrix24 Handler       | Validate Token           |                                                                                                |
| Validate Token    | If                  | Validates application token             | Credentials            | Route Event, Error Response |                                                                                                |
| Error Response    | Respond to Webhook   | Sends 401 error if token invalid        | Validate Token (false) | None                     |                                                                                                |
| Route Event       | Switch              | Routes workflow based on event type     | Validate Token (true)  | Process Message, Process Join, Process Install, Process Delete |                                                                                                |
| Process Message   | Function            | Processes chat messages                  | Route Event (ONIMBOTMESSAGEADD) | Send Message             |                                                                                                |
| Process Join      | Function            | Handles bot joining chat event           | Route Event (ONIMBOTJOINCHAT) | Send Join Message        |                                                                                                |
| Process Install   | Function            | Prepares bot registration data           | Route Event (ONAPPINSTALL) | Register Bot             |                                                                                                |
| Process Delete    | NoOp                | Handles bot deletion event (no action)   | Route Event (ONIMBOTDELETE) | Success Response         |                                                                                                |
| Register Bot      | HTTP Request        | Registers the bot via Bitrix24 API       | Process Install        | Success Response         |                                                                                                |
| Send Message      | HTTP Request        | Sends chat messages via Bitrix24 API     | Process Message        | Success Response         |                                                                                                |
| Send Join Message | HTTP Request        | Sends welcome messages on join event     | Process Join           | Success Response         |                                                                                                |
| Success Response  | Respond to Webhook   | Sends 200 OK success response            | Register Bot, Send Message, Send Join Message, Process Delete | None                     |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Bitrix24 Handler"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `bitrix24/handler.php`  
   - Response Mode: Use response node downstream  
   - Position: (0,0)

2. **Create Set Node: "Credentials"**  
   - Type: Set  
   - Assign static CLIENT_ID: `"local.6779636e712043.37129431"`  
   - Assign static CLIENT_SECRET: `"dTzUfBoTFLxNhuzc1zsnDbCeii98ZaE5By4aQPQEOxLJAS9y6i"`  
   - Dynamically assign:  
     - `application_token` from `$json.body['auth[application_token]']`  
     - `domain` from `$json.body['auth[domain]']`  
     - `access_token` from `$json.body['auth[access_token]']`  
   - Include other fields from input JSON  
   - Connect input from "Bitrix24 Handler"

3. **Create If Node: "Validate Token"**  
   - Type: If  
   - Condition:  
     - Check if `CLIENT_ID` equals `application_token` OR `1 == 1` (always true fallback)  
   - Connect input from "Credentials"  
   - True branch to "Route Event"  
   - False branch to "Error Response"

4. **Create Respond to Webhook Node: "Error Response"**  
   - HTTP Response Code: 401  
   - Response Body (JSON):  
     ```json
     {
       "result": false,
       "error": "Invalid application token"
     }
     ```  
   - Connect input from "Validate Token" (false branch)

5. **Create Switch Node: "Route Event"**  
   - Type: Switch  
   - Property to check: `$json.body.event`  
   - Cases:  
     - ONIMBOTMESSAGEADD → Output 1  
     - ONIMBOTJOINCHAT → Output 2  
     - ONAPPINSTALL → Output 3  
     - ONIMBOTDELETE → Output 4  
   - Connect input from "Validate Token" (true branch)

6. **Create Function Node: "Process Message"**  
   - Code:  
     ```javascript
     const items = $input.all();
     const item = items[0];
     const message = item.json.body['data[PARAMS][MESSAGE]'];
     const dialogId = item.json.body['data[PARAMS][DIALOG_ID]'];
     const auth = {
       access_token: item.json.access_token,
       domain: item.json.domain
     };
     if (message.toLowerCase() === "what's hot") {
       return {
         json: {
           DIALOG_ID: dialogId,
           MESSAGE: "Hi! I am an example-bot.\nI repeat what you say",
           AUTH: auth.access_token,
           DOMAIN: auth.domain
         }
       };
     } else {
       return {
         json: {
           DIALOG_ID: dialogId,
           MESSAGE: `You said:\n${message}`,
           AUTH: auth.access_token,
           DOMAIN: auth.domain
         }
       };
     }
     ```  
   - Connect input from "Route Event" output ONIMBOTMESSAGEADD

7. **Create HTTP Request Node: "Send Message"**  
   - Method: POST  
   - URL: `https://{{$json.DOMAIN}}/rest/imbot.message.add?auth={{$json.AUTH}}`  
   - Body Parameters (JSON):  
     - DIALOG_ID: `{{$json.DIALOG_ID}}`  
     - MESSAGE: `{{$json.MESSAGE}}`  
     - AUTH: `{{$json.AUTH}}` (included but not required by API)  
   - Connect input from "Process Message"

8. **Create Function Node: "Process Join"**  
   - Code:  
     ```javascript
     const items = $input.all();
     const item = items[0];
     const dialogId = item.json.body['data[PARAMS][DIALOG_ID]'];
     const auth = {
       access_token: item.json.access_token,
       domain: item.json.domain
     };
     return {
       json: {
         DIALOG_ID: dialogId,
         MESSAGE: 'Hi! I am an example-bot. I repeat what you say',
         AUTH: auth.access_token,
         DOMAIN: auth.domain
       }
     };
     ```  
   - Connect input from "Route Event" output ONIMBOTJOINCHAT

9. **Create HTTP Request Node: "Send Join Message"**  
   - Same configuration as "Send Message" node  
   - Connect input from "Process Join"

10. **Create Function Node: "Process Install"**  
    - Code:  
      ```javascript
      const items = $input.all();
      const item = items[0];
      const handlerBackUrl = item.json.webhookUrl;
      const auth = {
        access_token: item.json.access_token,
        application_token: item.json.application_token,
        domain: item.json.domain
      };
      return {
        json: {
          handler_back_url: handlerBackUrl,
          CODE: 'LocalExampleBot',
          TYPE: 'B',
          EVENT_MESSAGE_ADD: handlerBackUrl,
          EVENT_WELCOME_MESSAGE: handlerBackUrl,
          EVENT_BOT_DELETE: handlerBackUrl,
          PROPERTIES: {
            NAME: 'Bot',
            LAST_NAME: 'Example',
            COLOR: 'AQUA',
            EMAIL: 'no@example.com',
            PERSONAL_BIRTHDAY: '2020-07-18',
            WORK_POSITION: 'Report on affairs',
            PERSONAL_GENDER: 'M'
          },
          AUTH: auth.access_token,
          CLIENT_ID: auth.application_token,
          DOMAIN: auth.domain
        }
      };
      ```  
    - Connect input from "Route Event" output ONAPPINSTALL

11. **Create HTTP Request Node: "Register Bot"**  
    - Method: POST  
    - URL: `https://{{ $json.DOMAIN }}/rest/imbot.register?auth={{$json.AUTH}}`  
    - Body Parameters (JSON):  
      - CODE: `"LocalExampleBot"`  
      - TYPE: `"B"`  
      - EVENT_MESSAGE_ADD: `{{$json.handler_back_url}}`  
      - EVENT_WELCOME_MESSAGE: `{{$json.handler_back_url}}`  
      - EVENT_BOT_DELETE: `{{$json.handler_back_url}}`  
      - PROPERTIES: JSON object with bot properties (NAME, LAST_NAME, COLOR, etc.)  
      - CLIENT_ID: `{{$json.CLIENT_ID}}`  
      - CLIENT_SECRET: `{{$json.AUTH}}` (note: uses access token here)  
    - Connect input from "Process Install"

12. **Create NoOp Node: "Process Delete"**  
    - Type: No Operation (no processing)  
    - Connect input from "Route Event" output ONIMBOTDELETE

13. **Create Respond to Webhook Node: "Success Response"**  
    - HTTP Response Code: 200  
    - Response Body (JSON): `{ "result": true }`  
    - Connect inputs from:  
      - "Send Message"  
      - "Send Join Message"  
      - "Register Bot"  
      - "Process Delete"

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow demonstrates a Bitrix24 chatbot integration using webhook events and REST API calls.  | Workflow description and use case.                                                              |
| Bot registration requires valid client credentials and correct webhook URL configuration in Bitrix24. | Setup instructions in workflow description.                                                     |
| Bitrix24 webhook engine sends events such as ONIMBOTMESSAGEADD, ONIMBOTJOINCHAT, ONAPPINSTALL, etc. | Event types handled by the workflow.                                                            |
| For more information on Bitrix24 REST API and imbot methods, see: https://training.bitrix24.com/rest_help/imbot/ | Official Bitrix24 API documentation.                                                            |
| The workflow uses n8n nodes: Webhook, Set, If, Switch, Function, HTTP Request, Respond to Webhook.   | Node types and versions are compatible with n8n v1+ and HTTP Request v4+.                        |

---

This document fully describes the Bitrix24 Chatbot Application Workflow example with Webhook Integration, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.