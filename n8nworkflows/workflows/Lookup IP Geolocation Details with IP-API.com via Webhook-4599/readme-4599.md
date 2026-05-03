Lookup IP Geolocation Details with IP-API.com via Webhook

https://n8nworkflows.xyz/workflows/lookup-ip-geolocation-details-with-ip-api-com-via-webhook-4599


# Lookup IP Geolocation Details with IP-API.com via Webhook

### 1. Workflow Overview

This workflow provides an IP geolocation lookup service accessible via a webhook. Its primary purpose is to receive an IP address through an HTTP POST request, query the IP-API.com geolocation API for detailed location information of that IP, and respond with the full geolocation data. It is designed for use cases such as enriching logs, security analysis, or any automation requiring geographic context of IP addresses.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Listens for incoming HTTP POST requests containing the IP address.
- **1.2 IP Geolocation Query:** Calls the external IP-API.com service to retrieve geolocation details.
- **1.3 Response Delivery:** Sends the retrieved geolocation data back to the requester.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming IP lookup requests via a webhook listening for POST requests with a JSON payload containing an `ip` property.

- **Nodes Involved:**  
  - Receive IP Webhook  
  - Note for Webhook Trigger (Sticky Note)

- **Node Details:**

  - **Receive IP Webhook**  
    - *Type & Role:* Webhook node; entry point of the workflow receiving HTTP POST requests.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Path: `ip-lookup` (customizable webhook path)  
      - Response Mode: `responseNode` (deferred response via a dedicated respond node)  
    - *Expressions / Variables:* None directly, but expects `ip` in JSON body.  
    - *Input Connections:* None (trigger node).  
    - *Output Connections:* Connected to "Get IP Geolocation" node.  
    - *Edge Cases / Failures:*  
      - Missing or malformed JSON body.  
      - Missing `ip` property in the request body.  
      - Invalid HTTP method or path.  
    - *Notes:* The sticky note describes the expected input format and flexibility of the webhook path.

  - **Note for Webhook Trigger**  
    - *Type & Role:* Sticky Note node providing explanatory context.  
    - *Content:* Explains the webhook expects a JSON body with an `ip` field (e.g., `{"ip": "8.8.8.8"}`) and mentions webhook path can be changed.  
    - *Input/Output:* None.

#### 1.2 IP Geolocation Query

- **Overview:**  
  This block performs an HTTP GET request to IP-API.com using the IP address received to fetch detailed geolocation data.

- **Nodes Involved:**  
  - Get IP Geolocation  
  - Note for IP Lookup (Sticky Note)

- **Node Details:**

  - **Get IP Geolocation**  
    - *Type & Role:* HTTP Request node; makes an external API call to fetch geolocation details.  
    - *Configuration:*  
      - Method: GET  
      - URL: `http://ip-api.com/json/{{ $json.body.ip }}` (dynamic URL using the `ip` from webhook body)  
      - No additional options or authentication required.  
    - *Expressions / Variables:* Uses an expression to insert the IP address dynamically from incoming webhook data.  
    - *Input Connections:* Receives from "Receive IP Webhook".  
    - *Output Connections:* Connected to "Respond with Geolocation Data" node.  
    - *Edge Cases / Failures:*  
      - Invalid or malformed IP address causing API errors.  
      - IP-API.com service downtime or rate limiting.  
      - Network timeouts or connectivity issues.  
      - API returning error responses (e.g., private IPs, reserved IPs).  
    - *Notes:* The sticky note explains that this node calls IP-API.com and details the kind of information returned (country, city, ISP, etc.).

  - **Note for IP Lookup**  
    - *Type & Role:* Sticky Note node providing context.  
    - *Content:* Describes the purpose of the HTTP request and the nature of the data returned by IP-API.com.  
    - *Input/Output:* None.

#### 1.3 Response Delivery

- **Overview:**  
  This block sends the full geolocation data obtained from IP-API.com back to the original caller of the webhook.

- **Nodes Involved:**  
  - Respond with Geolocation Data  
  - Note for Webhook Response (Sticky Note)

- **Node Details:**

  - **Respond with Geolocation Data**  
    - *Type & Role:* Respond to Webhook node; sends the workflow output back to the webhook client.  
    - *Configuration:*  
      - Respond With: `allIncomingItems` (returns all data from previous node)  
      - No additional headers or transformations applied.  
    - *Expressions / Variables:* None.  
    - *Input Connections:* Receives from "Get IP Geolocation".  
    - *Output Connections:* None (terminal node).  
    - *Edge Cases / Failures:*  
      - If prior node fails or returns empty data, response will be incomplete or empty.  
      - Network errors during response transmission.  
    - *Notes:* Sticky note highlights that the response contains full geolocation data and can be used as-is or further processed.

  - **Note for Webhook Response**  
    - *Type & Role:* Sticky Note node with explanatory content.  
    - *Content:* Describes that the node returns all geolocation data to the webhook caller and mentions possible use cases.  
    - *Input/Output:* None.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                  | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                  |
|-----------------------------|-------------------------|---------------------------------|------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Note for Webhook Trigger     | Sticky Note             | Explains webhook input format   |                        |                             | This node listens for incoming POST requests. It expects a JSON body with an 'ip' property containing the IP address you want to look up (e.g., {"ip": "8.8.8.8"}). You can easily adjust the webhook path to suit your needs. |
| Receive IP Webhook           | Webhook                 | Receives IP lookup requests     |                        | Get IP Geolocation          | This node listens for incoming POST requests. It expects a JSON body with an 'ip' property containing the IP address you want to look up (e.g., {"ip": "8.8.8.8"}). You can easily adjust the webhook path to suit your needs. |
| Note for IP Lookup           | Sticky Note             | Explains IP-API.com call        |                        |                             | This node makes an HTTP GET request to the IP-API.com service to get detailed geolocation information for the IP address provided by the webhook. The API returns data like country, city, region, ISP, and more. |
| Get IP Geolocation           | HTTP Request            | Calls IP-API.com for geolocation| Receive IP Webhook      | Respond with Geolocation Data| This node makes an HTTP GET request to the IP-API.com service to get detailed geolocation information for the IP address provided by the webhook. The API returns data like country, city, region, ISP, and more. |
| Note for Webhook Response    | Sticky Note             | Explains response behavior      |                        |                             | This node sends the full geolocation data received from IP-API.com back to the original caller of the webhook. This data can be directly consumed or further processed (e.g., logged, filtered, used in conditional logic) before being returned. |
| Respond with Geolocation Data| Respond to Webhook      | Sends geolocation data to caller| Get IP Geolocation      |                             | This node sends the full geolocation data received from IP-API.com back to the original caller of the webhook. This data can be directly consumed or further processed (e.g., logged, filtered, used in conditional logic) before being returned. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Name: `Receive IP Webhook`  
   - Type: `Webhook`  
   - HTTP Method: `POST`  
   - Path: `ip-lookup` (or your preferred path)  
   - Response Mode: `Response Node` (allows responding later via a dedicated node)  
   - No authentication or additional headers required.

2. **Add a Sticky Note** (optional)  
   - Name: `Note for Webhook Trigger`  
   - Content: Explain that this webhook expects a JSON body with an `ip` field, e.g., `{"ip":"8.8.8.8"}`.

3. **Add an HTTP Request Node**  
   - Name: `Get IP Geolocation`  
   - Type: `HTTP Request`  
   - Method: `GET`  
   - URL: `http://ip-api.com/json/{{ $json.body.ip }}` (use expression editor to insert: `{{$json.body.ip}}`)  
   - No authentication needed.  
   - No additional query parameters or headers required.

4. **Add a Sticky Note** (optional)  
   - Name: `Note for IP Lookup`  
   - Content: Describe that this node calls IP-API.com and retrieves geolocation data such as city, country, ISP, etc.

5. **Connect Nodes**  
   - Connect `Receive IP Webhook` → `Get IP Geolocation`.

6. **Add a Respond to Webhook Node**  
   - Name: `Respond with Geolocation Data`  
   - Type: `Respond to Webhook`  
   - Respond With: `All Incoming Items` (returns the full output from the previous node)  
   - No additional configuration needed.

7. **Add a Sticky Note** (optional)  
   - Name: `Note for Webhook Response`  
   - Content: Explain this node returns the full geolocation data back to the webhook caller.

8. **Connect Nodes**  
   - Connect `Get IP Geolocation` → `Respond with Geolocation Data`.

9. **Activate the Workflow**  
   - Save and activate the workflow.  
   - Test by sending a POST request with JSON body `{"ip": "8.8.8.8"}` to your webhook URL (e.g., `https://<your-n8n-domain>/webhook/ip-lookup`).

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| The IP-API.com service used is a free IP geolocation service with limitations on requests per minute; consider API limits.      | https://ip-api.com/docs/                                  |
| This workflow can be adapted for other IP geolocation APIs by adjusting the HTTP Request node's URL and response handling.     |                                                          |
| The webhook path `ip-lookup` is customizable to suit your deployment environment or naming conventions.                        |                                                          |
| Consider adding error handling nodes or conditional checks to handle invalid IPs or API failures gracefully.                   |                                                          |
| Example request for testing: `curl -X POST https://<your-n8n-domain>/webhook/ip-lookup -H "Content-Type: application/json" -d '{"ip":"8.8.8.8"}'` |                                                          |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.