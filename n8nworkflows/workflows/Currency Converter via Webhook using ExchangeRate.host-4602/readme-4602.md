Currency Converter via Webhook using ExchangeRate.host

https://n8nworkflows.xyz/workflows/currency-converter-via-webhook-using-exchangerate-host-4602


# Currency Converter via Webhook using ExchangeRate.host

### 1. Workflow Overview

This workflow provides a currency conversion service accessible via a webhook. It is designed to receive conversion requests with source and target currency codes plus an amount, query the ExchangeRate.host API for the conversion, and respond with the converted amount. The workflow is structured into three logical blocks:

- **1.1 Input Reception:** Receives POST requests on a webhook with JSON payload specifying currencies and amount.
- **1.2 Currency Conversion API Call:** Calls the ExchangeRate.host API using the input parameters and a securely stored API key.
- **1.3 Output Response:** Returns the API response to the original webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for incoming HTTP POST requests at a specific webhook path. Validates and extracts the input parameters needed for currency conversion.

- **Nodes Involved:**  
  - Receive Conversion Request Webhook  
  - Note: Webhook Trigger (sticky note for documentation)

- **Node Details:**

  - **Receive Conversion Request Webhook**  
    - Type: Webhook Trigger  
    - Role: Entry point for the workflow; listens on POST `/convert-currency` path.  
    - Configuration: HTTP Method = POST, Response Mode = `responseNode` (defers response until later node).  
    - Input: Incoming HTTP POST request with JSON body containing `from`, `to`, and `amount`.  
    - Output: Passes JSON body to next node for processing.  
    - Edge Cases: Missing or invalid fields in JSON may cause API call failures downstream; no explicit validation in this workflow.  
    - Version Requirements: n8n webhook node v2.

  - **Note: Webhook Trigger**  
    - Type: Sticky Note  
    - Role: Documents expected input format and security note about API key handling.

#### 1.2 Currency Conversion API Call

- **Overview:**  
  Makes an authenticated HTTP GET request to ExchangeRate.host’s `/convert` endpoint using input currencies and amount. Uses n8n’s credential system to securely provide the API key.

- **Nodes Involved:**  
  - Convert Currency  
  - Note: Currency Conversion API Call (sticky note)

- **Node Details:**

  - **Convert Currency**  
    - Type: HTTP Request  
    - Role: Queries the external currency conversion API.  
    - Configuration:  
      - Method: GET  
      - URL Template: `https://api.exchangerate.host/convert?from={{ $json.body.from }}&to={{ $json.body.to }}&amount={{ $json.body.amount }}`  
      - Authentication: `httpQueryAuth` type credential named “ExchangeRate” (injects `access_key` parameter).  
    - Input: JSON body from webhook node containing currencies and amount.  
    - Output: JSON response from ExchangeRate.host converted API result.  
    - Edge Cases:  
      - API key invalid or expired (auth error)  
      - Network timeout or unreachable API  
      - Input currencies invalid or unsupported (API error response)  
      - Amount not numeric or missing  
    - Version Requirements: HTTP Request node version 4.2.

  - **Note: Currency Conversion API Call**  
    - Type: Sticky Note  
    - Role: Documents the use of API key via credentials and the API call construction.

#### 1.3 Output Response

- **Overview:**  
  Sends back the result of the conversion API call to the original webhook caller, completing the request-response cycle.

- **Nodes Involved:**  
  - Respond with Converted Amount  
  - Note: Webhook Response (sticky note)

- **Node Details:**

  - **Respond with Converted Amount**  
    - Type: Respond to Webhook  
    - Role: Outputs the API response JSON directly to the webhook client.  
    - Configuration: Respond with all incoming items (full API response payload).  
    - Input: Output of the Convert Currency node.  
    - Output: HTTP response to original request.  
    - Edge Cases: If input is empty or malformed, response will reflect that (no explicit error handling).  
    - Version: 1.2.

  - **Note: Webhook Response**  
    - Type: Sticky Note  
    - Role: Explains that this node can be extended to format or log data before responding.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                  | Input Node(s)                    | Output Node(s)                 | Sticky Note                                                                                                                                                                                                                                      |
|-------------------------------|---------------------|--------------------------------|---------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Receive Conversion Request Webhook | Webhook Trigger     | Input Reception (webhook)       | —                               | Convert Currency               | This node listens for incoming POST requests. It expects a JSON body with `from`, `to`, and `amount`. API key handled securely by credentials.                                                                                                  |
| Note: Webhook Trigger         | Sticky Note         | Documentation                   | —                               | —                             | ## Webhook Input\n\nThis node listens for incoming POST requests. It expects a JSON body with the following properties:\n\n* `from` (source currency's 3-letter ISO 4217 code, e.g., `USD`)\n* `to` (target currency's 3-letter ISO 4217 code, e.g., `EUR`)\n* `amount` (numeric value to convert)\n\n**Important:** The ExchangeRate.host API key is handled securely by n8n's credential system and should **not** be included in the webhook body or headers. |
| Convert Currency              | HTTP Request        | Currency Conversion API Call   | Receive Conversion Request Webhook | Respond with Converted Amount | This node makes an HTTP GET request to the ExchangeRate.host API to perform the currency conversion. It uses the `from`, `to`, and `amount` from the webhook body to build the API request URL.\n\n**The API access key is securely retrieved from n8n's pre-configured credentials** (HTTP Query Auth type) and automatically added as a query parameter (`access_key`). This ensures your key is not exposed in the workflow or webhook requests. |
| Note: Currency Conversion API Call | Sticky Note         | Documentation                   | —                               | —                             | ## Currency Conversion API Call\n\nThis node makes an HTTP GET request to the ExchangeRate.host API to perform the currency conversion. It uses the `from`, `to`, and `amount` from the webhook body to build the API request URL.\n\n**The API access key is securely retrieved from n8n's pre-configured credentials** (HTTP Query Auth type) and automatically added as a query parameter (`access_key`). This ensures your key is not exposed in the workflow or webhook requests. |
| Respond with Converted Amount | Respond to Webhook  | Output Response                | Convert Currency                | —                             | This node sends the currency conversion result received from ExchangeRate.host back to the original caller of the webhook. You can insert other nodes before this to format the output, log the conversion, or perform further actions before responding. |
| Note: Webhook Response       | Sticky Note         | Documentation                   | —                               | —                             | ## Webhook Response\n\nThis node sends the currency conversion result received from ExchangeRate.host back to the original caller of the webhook. You can insert other nodes before this to format the output, log the conversion, or perform further actions before responding. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**  
   - Name: `Receive Conversion Request Webhook`  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `convert-currency`  
   - Response Mode: `Response Node` (to defer response until later)  
   - No authentication on webhook.  
   - Position appropriately (e.g., left side).

2. **Add a Sticky Note node near the webhook:**  
   - Name: `Note: Webhook Trigger`  
   - Content: Document expected JSON input keys (`from`, `to`, `amount`) and note that the API key is managed securely via credentials.

3. **Create an HTTP Request node:**  
   - Name: `Convert Currency`  
   - HTTP Method: GET  
   - URL: Use expression to build URL:  
     ```
     https://api.exchangerate.host/convert?from={{ $json.body.from }}&to={{ $json.body.to }}&amount={{ $json.body.amount }}
     ```  
   - Authentication: Select `Generic Credential` type -> `HTTP Query Auth`  
   - Attach credentials named `"ExchangeRate"` (to be created separately with your ExchangeRate.host API key)  
   - Position to the right of the webhook node.

4. **Add a Sticky Note node near the HTTP Request:**  
   - Name: `Note: Currency Conversion API Call`  
   - Content: Explain the API call construction and secure use of API key via credentials.

5. **Create a Respond to Webhook node:**  
   - Name: `Respond with Converted Amount`  
   - Configuration: Respond with `allIncomingItems` (full data from previous node)  
   - Position to the right of the HTTP Request node.

6. **Add a Sticky Note node near the respond node:**  
   - Name: `Note: Webhook Response`  
   - Content: Explain this node sends the response back and can be extended for formatting or logging.

7. **Connect nodes in sequence:**  
   - `Receive Conversion Request Webhook` → `Convert Currency` → `Respond with Converted Amount`

8. **Set up credentials:**  
   - Create HTTP Query Auth credential named `"ExchangeRate"`  
   - Provide your ExchangeRate.host API key as the `access_key` parameter.

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| ExchangeRate.host API key is securely managed via n8n credentials to avoid exposure in webhook requests.                                                                                                                                      | Workflow security best practice                                  |
| The webhook expects a JSON body with keys `from`, `to`, and `amount` representing ISO 4217 currency codes and numeric amount.                                                                                                                | Input data specification                                         |
| You can extend the workflow to add input validation, error handling, or logging before responding.                                                                                                                                             | Suggested enhancement                                            |
| ExchangeRate.host API documentation: https://exchangerate.host/#/#docs                                                                                                                                                                        | Official API docs                                                |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.