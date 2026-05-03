TinyURL Shortener via Webhook

https://n8nworkflows.xyz/workflows/tinyurl-shortener-via-webhook-4595


# TinyURL Shortener via Webhook

### 1. Workflow Overview

This workflow provides an automated service to shorten URLs using the TinyURL API. It listens for incoming HTTP POST requests via a webhook, extracts the necessary parameters from the request body, calls the TinyURL API to generate a shortened URL, and responds back to the caller with the resulting shortened link and related data.

Logical blocks included:

- **1.1 Input Reception:** Receives and processes incoming HTTP POST requests containing URL shortening parameters.
- **1.2 TinyURL API Request:** Sends a POST request to the TinyURL API endpoint with extracted parameters to create a shortened URL.
- **1.3 Response Handling:** Sends the API response back to the original webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming HTTP POST requests on a specific webhook path (`shorten-link`). It acts as the entry point to the workflow, accepting JSON payloads containing the URL to shorten along with optional parameters.

**Nodes Involved:**  
- Receive Link Webhook

**Node Details:**

- **Receive Link Webhook**
  - Type: Webhook Trigger  
  - Role: Listens for and receives incoming HTTP POST requests at the path `/shorten-link`.  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `shorten-link`  
    - Response Mode: Uses the "Respond to Webhook" node to send the response (deferred response).  
  - Inputs: None (entry node)  
  - Outputs: Forwards the incoming request data (including JSON body) to the next node.  
  - Edge Cases / Potential Failures:  
    - Invalid or missing POST body parameters (e.g., missing `url` or `api_key`) may cause downstream errors.  
    - Unauthorized or malformed requests are not explicitly handled here.  
  - Version: n8n Webhook node v2

---

#### 1.2 TinyURL API Request

**Overview:**  
This block constructs and sends a POST request to the TinyURL API to create a shortened URL using provided parameters from the webhook. It passes authentication via an API token and includes optional fields like alias and description.

**Nodes Involved:**  
- Create TinyURL  
- Note for TinyURL API Request (informative)

**Node Details:**

- **Create TinyURL**  
  - Type: HTTP Request  
  - Role: Sends POST request to `https://api.tinyurl.com/create` to create a shortened URL.  
  - Configuration:  
    - URL: `https://api.tinyurl.com/create`  
    - Method: POST  
    - Body: JSON containing:  
      - `url`: Required. Taken from incoming webhook JSON body `body.url`.  
      - `domain`: Fixed as `"tinyurl.com"`.  
      - `alias`: Optional. Defaults to empty string if not provided (`body.alias || ''`).  
      - `description`: Optional. Defaults to empty string if not provided (`body.description || ''`).  
    - Query Parameters:  
      - `api_token`: Sent as query parameter, value extracted from `body.api_key` in webhook JSON.  
    - Sends body as JSON.  
  - Inputs: Receives JSON from the webhook node containing parameters.  
  - Outputs: The response from TinyURL API including the shortened URL and metadata.  
  - Edge Cases / Potential Failures:  
    - Missing or invalid `api_key` will cause authentication failure from TinyURL API.  
    - Invalid or empty `url` parameter will cause API to reject request.  
    - Network issues or API rate limits may cause request failures or timeouts.  
    - If alias conflicts with existing aliases, API may reject request.  
  - Version: n8n HTTP Request node v4.2  

- **Note for TinyURL API Request**  
  - Type: Sticky Note  
  - Role: Documentation note explaining the purpose and expectations of the Create TinyURL node.  
  - Content: Explains required and optional parameters and API call role.

---

#### 1.3 Response Handling

**Overview:**  
This block sends the result from the TinyURL API back to the original webhook caller as the HTTP response, effectively completing the request-response cycle.

**Nodes Involved:**  
- Respond with Shortened URL  
- Note for Respond to Webhook (informative)

**Node Details:**

- **Respond with Shortened URL**  
  - Type: Respond to Webhook  
  - Role: Sends the output from the TinyURL API node back to the initial HTTP request sender.  
  - Configuration:  
    - Responds with all incoming items (the full JSON response from the TinyURL API).  
  - Inputs: Takes input from the Create TinyURL HTTP Request node.  
  - Outputs: None (terminal node).  
  - Edge Cases / Potential Failures:  
    - If upstream node failed or returned no data, response may be empty or cause errors.  
  - Version: v1.2  

- **Note for Respond to Webhook**  
  - Type: Sticky Note  
  - Role: Documentation note explaining how this node completes the workflow and can be extended for additional processing before response.  
  - Content: Encourages adding nodes before responding if further processing is needed.

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                            | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                                                    |
|---------------------------|----------------------|-------------------------------------------|------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Receive Link Webhook       | Webhook              | Receives incoming HTTP POST requests      | None                   | Create TinyURL            |                                                                                                                                                |
| Create TinyURL            | HTTP Request         | Sends request to TinyURL API to shorten URL | Receive Link Webhook    | Respond with Shortened URL | This node sends a POST request to the TinyURL API to create a shortened URL. It expects 'api_token' and 'url' from the incoming webhook body. The 'domain', 'alias', and 'description' are optional. |
| Respond with Shortened URL | Respond to Webhook   | Sends TinyURL API response back to caller | Create TinyURL          | None                      | This node sends the response from the TinyURL API (the shortened URL and any other relevant data) back to the original caller of the webhook. You can insert other nodes before this one to process the shortened URL (e.g., save to a database, send in a message) before responding. |
| Note for TinyURL API Request | Sticky Note         | Documentation for Create TinyURL node     | None                   | None                      | This node sends a POST request to the TinyURL API to create a shortened URL. It expects 'api_token' and 'url' from the incoming webhook body. The 'domain', 'alias', and 'description' are optional. |
| Note for Respond to Webhook | Sticky Note          | Documentation for Respond with Shortened URL node | None                   | None                      | This node sends the response from the TinyURL API (the shortened URL and any other relevant data) back to the original caller of the webhook. You can insert other nodes before this one to process the shortened URL (e.g., save to a database, send in a message) before responding. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Link Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `shorten-link`  
   - Response Mode: Set to "Respond with 'Respond to Webhook' node" (deferred response).  
   - No authentication configured (open endpoint).  

2. **Create HTTP Request Node: "Create TinyURL"**  
   - Type: HTTP Request  
   - HTTP Method: POST  
   - URL: `https://api.tinyurl.com/create`  
   - Authentication: None (API token sent as query parameter).  
   - Query Parameters:  
     - Name: `api_token`  
     - Value: Expression `{{$json.body.api_key}}` (extract from webhook JSON body)  
   - Body Parameters (JSON):  
     ```json
     {
       "url": "={{ $json.body.url }}",
       "domain": "tinyurl.com",
       "alias": "={{ $json.body.alias || '' }}",
       "description": "={{ $json.body.description || '' }}"
     }
     ```  
   - Send Body: Yes, as JSON.  
   - Connect input from "Receive Link Webhook" node.  

3. **Create Respond to Webhook Node: "Respond with Shortened URL"**  
   - Type: Respond to Webhook  
   - Respond With: All incoming items (pass full API response as JSON).  
   - Connect input from "Create TinyURL" node.  

4. **Connect the Nodes Sequentially:**  
   - `Receive Link Webhook` → `Create TinyURL` → `Respond with Shortened URL`  

5. **Optional: Add Sticky Notes for Documentation**  
   - Add a sticky note near "Create TinyURL" node explaining API call requirements (api_token, url, optional parameters).  
   - Add a sticky note near "Respond with Shortened URL" node explaining response role and extension suggestions.  

6. **Activate the Workflow**  
   - Save and activate the workflow to start listening on the webhook endpoint.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow can be extended to include additional processing before responding, such as saving links or notifying users. | See sticky note near "Respond with Shortened URL" node.                                               |
| TinyURL API requires a valid API token; ensure correct token is passed in `api_key` field of webhook JSON. | https://tinyurl.com/app/dev/api                                                                     |
| Use of the "Respond to Webhook" node allows asynchronous processing and proper HTTP response handling in n8n. | n8n Documentation on Webhook and Respond to Webhook nodes.                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and publicly available.