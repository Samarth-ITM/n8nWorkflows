Manage Adobe Acrobat e-signatures with webhooks

https://n8nworkflows.xyz/workflows/manage-adobe-acrobat-e-signatures-with-webhooks-1588


# Manage Adobe Acrobat e-signatures with webhooks

### 1. Workflow Overview

This workflow automates the management of Adobe Acrobat e-signatures by handling incoming webhooks from Acrobat Sign. Its primary function is to listen for signature-related events (intents) sent via Acrobat Sign webhooks, process the incoming data, and respond appropriately with required headers and extracted information.

The workflow is logically divided into the following blocks:

- **1.1 Webhook Reception:** Listens for HTTP POST and GET requests from Adobe Acrobat Sign containing signature event data.
- **1.2 Data Processing & Response:** Extracts client information from the webhook headers and returns a custom response header.
- **1.3 Data Extraction & Preparation:** Parses the webhook payload to extract key details such as agreement ID, participants, and agreement status for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Reception

**Overview:**  
This block receives incoming HTTP requests (both POST and GET) at the specified webhook endpoint (`test1`). It triggers the workflow whenever Acrobat Sign sends signature event notifications.

**Nodes Involved:**  
- POST (Webhook node)  
- reg-GET (Webhook node)

**Node Details:**

- **POST**  
  - Type: Webhook  
  - Role: Accepts POST requests at `/test1` path.  
  - Configuration:  
    - HTTP Method: POST  
    - Response Mode: `responseNode` (response will be handled by subsequent node)  
  - Input: External HTTP POST request from Acrobat Sign webhook  
  - Output: Passes incoming request data and headers downstream  
  - Edge Cases:  
    - Invalid or missing webhook payloads  
    - Unauthorized requests (no built-in auth configured)  
    - Network timeouts or dropped connections  

- **reg-GET**  
  - Type: Webhook  
  - Role: Accepts GET requests at `/test1` path (likely used for webhook registration verification or testing).  
  - Configuration:  
    - HTTP Method: GET (default)  
    - Response Mode: `responseNode`  
  - Input: External HTTP GET request  
  - Output: Passes request data downstream  
  - Edge Cases:  
    - Unexpected GET requests or malformed queries  
    - Same network/connectivity concerns as POST  

---

#### 2.2 Data Processing & Response

**Overview:**  
This block processes incoming webhook requests to extract the client ID from headers and sends a custom response header back to Acrobat Sign to confirm receipt or for validation purposes.

**Nodes Involved:**  
- Function  
- webhook-response (Respond to Webhook)

**Node Details:**

- **Function**  
  - Type: Function (JavaScript execution)  
  - Role: Extracts `x-adobesign-clientid` header from the incoming webhook request and adds it as a new field `clientID` to the JSON data for downstream use.  
  - Configuration:  
    - Reads `x-adobesign-clientid` from `items[0].json.headers`  
    - Iterates over all input items (usually one) and adds `clientID` and a fixed field `myNewField` set to 1  
  - Key Expressions: Accesses `items[0].json.headers['x-adobesign-clientid']`  
  - Input: Webhook data from `POST` or `reg-GET` nodes  
  - Output: Modified JSON with added fields  
  - Edge Cases:  
    - Missing or malformed `x-adobesign-clientid` header (could cause undefined `clientID`)  
    - Multiple input items (unlikely but handled)  
    - JavaScript execution errors  
  - Version Requirements: Standard Function node, no special version needed  

- **webhook-response**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP response back to the webhook sender, setting custom headers.  
  - Configuration:  
    - Sets response header `x-adobesign-clientid` with the value extracted in the Function node via expression `{{$node["Function"].json["clientID"]}}`  
  - Input: From Function node  
  - Output: HTTP response to Acrobat Sign server  
  - Edge Cases:  
    - Missing `clientID` could result in empty header or malformed response  
    - Response delays or failures in sending response  
  - Documentation Reference: https://docs.n8n.io/integrations/core-nodes/n8n-nodes-base.respondToWebhook/  

---

#### 2.3 Data Extraction & Preparation

**Overview:**  
This block extracts detailed information from the webhook payload's body, such as agreement ID, participant details, and status, and stores them as separate JSON fields for further processing or logging.

**Nodes Involved:**  
- SetWebhookData (Set node)

**Node Details:**

- **SetWebhookData**  
  - Type: Set  
  - Role: Extracts and sets specific fields from the webhook response body into structured workflow variables.  
  - Configuration:  
    - Sets the following fields using expressions referencing the `webhook-response` node's JSON body:  
      - `webhookData` = full webhook body  
      - `agreement_ID` = `body.agreement.id`  
      - `all_participants` = `body.agreement.participantSetsInfo`  
      - `agreement_status` = `body.agreement.status`  
    - `keepOnlySet` enabled to discard other fields  
  - Input: Output of `webhook-response` node (which contains the webhook JSON body)  
  - Output: JSON object containing extracted fields  
  - Edge Cases:  
    - Missing `agreement` object or nested fields in webhook body  
    - Data type mismatches if webhook payload changes  
    - Expression evaluation failures if paths are invalid  

---

### 3. Summary Table

| Node Name       | Node Type         | Functional Role                              | Input Node(s) | Output Node(s) | Sticky Note                                                                                                  |
|-----------------|-------------------|----------------------------------------------|---------------|----------------|--------------------------------------------------------------------------------------------------------------|
| POST            | Webhook           | Receives webhook POST requests from Acrobat Sign | —             | Function       |                                                                                                              |
| reg-GET         | Webhook           | Receives webhook GET requests (verification) | —             | Function       |                                                                                                              |
| Function        | Function          | Extracts client ID from headers, adds fields  | POST, reg-GET | webhook-response |                                                                                                              |
| webhook-response| Respond to Webhook| Sends HTTP response with custom header         | Function      | SetWebhookData |                                                                                                              |
| SetWebhookData  | Set               | Extracts agreement data from webhook body      | webhook-response | —            |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes**  
   - Add a **Webhook** node named `POST`.  
     - Set HTTP Method to `POST`.  
     - Set Path to `test1`.  
     - Set Response Mode to `responseNode`.  
   - Add another **Webhook** node named `reg-GET`.  
     - Set HTTP Method to `GET` (default).  
     - Set Path to `test1`.  
     - Set Response Mode to `responseNode`.

2. **Create Function Node**  
   - Add a **Function** node named `Function`.  
   - Use the following JavaScript code:  
     ```javascript
     const c_id = items[0].json.headers['x-adobesign-clientid'];
     for (const item of items) {
       item.json.myNewField = 1;
       item.json.clientID = c_id;
     }
     return items;
     ```  
   - Connect both `POST` and `reg-GET` nodes to this `Function` node.

3. **Create Respond to Webhook Node**  
   - Add a **Respond to Webhook** node named `webhook-response`.  
   - Under Options → Response Headers, add a header entry:  
     - Name: `x-adobesign-clientid`  
     - Value: `={{$node["Function"].json["clientID"]}}`  
   - Connect the `Function` node output to `webhook-response` node.

4. **Create Set Node for Data Extraction**  
   - Add a **Set** node named `SetWebhookData`.  
   - Enable "Keep Only Set".  
   - Add the following fields with expressions:  
     - `webhookData` → `={{ $item("0").$node["webhook-response"].json["body"] }}`  
     - `agreement_ID` → `={{ $item("0").$node["webhook-response"].json["body"]["agreement"]["id"] }}`  
     - `all_participants` → `={{ $item("0").$node["webhook-response"].json["body"]["agreement"]["participantSetsInfo"] }}`  
     - `agreement_status` → `={{ $item("0").$node["webhook-response"].json["body"]["agreement"]["status"] }}`  
   - Connect the `webhook-response` node output to the `SetWebhookData` node.

5. **Save and Activate Workflow**  
   - Ensure the workflow is activated to start receiving webhooks at `/test1`.

6. **Credential Setup**  
   - No specific credentials are required for this workflow as it listens to public HTTP endpoints.  
   - Ensure that Acrobat Sign webhook is configured to send events to your n8n webhook URL at `/test1`.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| More info on Function node usage and JavaScript examples                                      | https://docs.n8n.io/nodes/n8n-nodes-base.function/                                                     |
| Webhook node documentation for configuring HTTP endpoints                                    | https://docs.n8n.io/integrations/core-nodes/n8n-nodes-base.webhook/                                   |
| Respond to Webhook node for controlling HTTP response headers                                | https://docs.n8n.io/integrations/core-nodes/n8n-nodes-base.respondToWebhook/                          |
| Adobe Acrobat Sign webhook API documentation                                                | https://helpx.adobe.com/sign/using/adobe-sign-webhooks-api.html                                      |
| Basic JavaScript knowledge recommended for customizing Function node                         |                                                                                                       |

---

This structured analysis and instructions enable users and AI agents to fully understand, reproduce, and maintain the "Manage Adobe Acrobat e-signatures with webhooks" workflow, handling Adobe Acrobat Sign webhook events efficiently within n8n.