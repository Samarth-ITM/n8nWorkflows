Public Holiday Lookup with Nager.Date API via Webhook

https://n8nworkflows.xyz/workflows/public-holiday-lookup-with-nager-date-api-via-webhook-4601


# Public Holiday Lookup with Nager.Date API via Webhook

### 1. Workflow Overview

This workflow provides a public holiday lookup service via a webhook using the Nager.Date API. It is designed to receive a POST request containing a year and a country code, query the Nager.Date API for public holidays corresponding to those parameters, and return the holiday data to the caller.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives and validates incoming HTTP POST requests containing the year and country code.
- **1.2 External API Interaction:** Queries the Nager.Date public holiday API using the provided parameters.
- **1.3 Response Delivery:** Sends the retrieved holiday data back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming POST requests on a specified webhook path. It expects a JSON payload with two properties: `year` (e.g., 2025) and `countryCode` (e.g., US, PH, DE). This block serves as the entry point to the workflow.

- **Nodes Involved:**  
  - Receive Holiday Request Webhook  
  - Note for Webhook Trigger

- **Node Details:**

  - **Receive Holiday Request Webhook**  
    - Type: Webhook Trigger  
    - Role: Listens for HTTP POST requests on path `/public-holidays`  
    - Configurations:  
      - HTTP Method: POST  
      - Path: `public-holidays`  
      - Response Mode: `responseNode` (defers response to a later node)  
      - Webhook ID: unique internal identifier  
    - Input: External HTTP POST request with JSON body  
    - Output: JSON data accessible as `$json.body.year` and `$json.body.countryCode`  
    - Edge Cases:  
      - Missing or malformed JSON  
      - Missing required parameters (`year`, `countryCode`)  
      - Invalid HTTP method or path  
      - Webhook listener downtime or misconfiguration  
    - Version: 2

  - **Note for Webhook Trigger**  
    - Type: Sticky Note  
    - Purpose: Documentation for users explaining expected input format and valid country codes  
    - Content: Points to `https://www.nager.at/Country` for valid country codes  
    - No inputs or outputs

#### 1.2 External API Interaction

- **Overview:**  
  This block makes an HTTP GET request to the Nager.Date API, dynamically assembling the URL using the received year and country code. It fetches detailed public holiday data including date, name, and type.

- **Nodes Involved:**  
  - Get Public Holidays  
  - Note for Holiday API Call

- **Node Details:**

  - **Get Public Holidays**  
    - Type: HTTP Request  
    - Role: Queries Nager.Date API endpoint to get public holidays  
    - Configuration:  
      - Method: GET  
      - URL template: `https://date.nager.at/api/v3/PublicHolidays/{{ $json.body.year }}/{{ $json.body.countryCode }}`  
      - No additional headers or authentication required  
    - Inputs: JSON body from webhook node containing `year` and `countryCode`  
    - Outputs: JSON array of public holidays with fields such as date, local name, English name, and types  
    - Edge Cases:  
      - Invalid or unsupported country code or year leading to 404 or empty results  
      - Network errors or timeouts with the external API  
      - Rate limiting or API downtime  
    - Version: 4.2

  - **Note for Holiday API Call**  
    - Type: Sticky Note  
    - Purpose: Describes the API call and expected response format  
    - Content: Explains what the API returns (date, name, type)  
    - No inputs or outputs

#### 1.3 Response Delivery

- **Overview:**  
  This block sends the retrieved list of public holidays back to the original webhook caller as the HTTP response. It allows for optional additional processing before response if needed.

- **Nodes Involved:**  
  - Respond with Holiday Data  
  - Note for Webhook Response

- **Node Details:**

  - **Respond with Holiday Data**  
    - Type: Respond to Webhook  
    - Role: Sends the final JSON data back to the HTTP client that called the webhook  
    - Configuration:  
      - Respond With: `allIncomingItems` (returns all data from previous node)  
      - No additional headers or formatting applied by default  
    - Inputs: Output from "Get Public Holidays" node  
    - Outputs: HTTP response with JSON body containing public holiday data  
    - Edge Cases:  
      - Large payload size impacting response time  
      - Possible missing or malformed data if upstream node failed silently  
    - Version: 1.2

  - **Note for Webhook Response**  
    - Type: Sticky Note  
    - Purpose: Explains that this node sends data back to the caller and can be extended for filtering or formatting  
    - No inputs or outputs

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                  |
|---------------------------|-----------------------|----------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Receive Holiday Request Webhook | Webhook Trigger       | Input reception of POST requests        |                             | Get Public Holidays         | This node listens for incoming POST requests. It expects a JSON body with 'year' (e.g., 2025) and 'countryCode' (e.g., US, PH, DE) properties to fetch public holidays. Find country codes at: https://www.nager.at/Country |
| Note for Webhook Trigger  | Sticky Note           | Documentation on webhook input          |                             |                            | This node listens for incoming POST requests. It expects a JSON body with 'year' (e.g., 2025) and 'countryCode' (e.g., US, PH, DE) properties to fetch public holidays. Find country codes at: https://www.nager.at/Country |
| Get Public Holidays        | HTTP Request          | Fetches public holidays from Nager.Date API | Receive Holiday Request Webhook | Respond with Holiday Data   | This node makes an HTTP GET request to the Nager.Date API to retrieve a list of public holidays for the specified year and country. The API returns details like the date, name, and type of each holiday.             |
| Note for Holiday API Call | Sticky Note           | Documentation on API call and response  |                             |                            | This node makes an HTTP GET request to the Nager.Date API to retrieve a list of public holidays for the specified year and country. The API returns details like the date, name, and type of each holiday.             |
| Respond with Holiday Data  | Respond to Webhook    | Sends holiday data back to webhook caller | Get Public Holidays          |                            | This node sends the list of public holidays received from Nager.Date back to the original caller of the webhook. You can insert other nodes before this to filter, format, store, or further process the holiday data. |
| Note for Webhook Response  | Sticky Note           | Documentation on webhook response       |                             |                            | This node sends the list of public holidays received from Nager.Date back to the original caller of the webhook. You can insert other nodes before this to filter, format, store, or further process the holiday data. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Node Type: Webhook  
   - Name: `Receive Holiday Request Webhook`  
   - HTTP Method: `POST`  
   - Path: `public-holidays`  
   - Response Mode: Set to `Response Node` (defers response to a later node)  
   - No authentication required  
   - Accept JSON body with two properties: `year` (integer), `countryCode` (string)  

2. **Add Sticky Note for Webhook Trigger:**  
   - Content: Explain that this node expects a JSON body with `year` and `countryCode` to fetch public holidays  
   - Include link: `https://www.nager.at/Country` for valid country codes  
   - Position near the webhook node for clarity  

3. **Create HTTP Request Node:**  
   - Node Type: HTTP Request  
   - Name: `Get Public Holidays`  
   - Method: GET  
   - URL: `https://date.nager.at/api/v3/PublicHolidays/{{ $json.body.year }}/{{ $json.body.countryCode }}` (use expression to dynamically insert year and country code)  
   - No authentication or additional headers needed  
   - Connect output of `Receive Holiday Request Webhook` to this node’s input  

4. **Add Sticky Note for API Call:**  
   - Content: Describe that this node calls the Nager.Date API and returns public holiday data including date, name, and type  
   - Position near the HTTP Request node  

5. **Create Respond to Webhook Node:**  
   - Node Type: Respond to Webhook  
   - Name: `Respond with Holiday Data`  
   - Set to respond with `allIncomingItems` to return the full JSON response from the previous node  
   - Connect output of `Get Public Holidays` node to this node’s input  

6. **Add Sticky Note for Webhook Response:**  
   - Content: Explain that this node sends the holiday data back to the webhook caller and can be extended to filter or format the data  
   - Position near the Respond node  

7. **Connect Nodes:**  
   - Connect `Receive Holiday Request Webhook` → `Get Public Holidays` → `Respond with Holiday Data`  

8. **Activate Workflow:**  
   - Ensure all nodes are configured correctly and test by sending a POST request to the webhook URL with a JSON body containing `year` and `countryCode`.  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                       |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------|
| Valid country codes can be found at the official Nager.Date country list: https://www.nager.at/Country          | Documentation for country codes     |
| The Nager.Date API is free and public, no authentication needed for holiday queries                             | API usage note                      |
| Webhook response mode `responseNode` defers sending the HTTP response until the Respond to Webhook node runs | n8n webhook configuration concept   |
| This workflow can be extended to filter, format, or store holiday data before responding                       | Potential workflow customization     |

---

**Disclaimer:**  
The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.