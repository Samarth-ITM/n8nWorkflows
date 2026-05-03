Real-Time Currency Conversion via Webhook & Google Search Parsing

https://n8nworkflows.xyz/workflows/real-time-currency-conversion-via-webhook---google-search-parsing-6123


# Real-Time Currency Conversion via Webhook & Google Search Parsing

### 1. Workflow Overview

This workflow performs **real-time currency conversion** by capturing incoming HTTP GET requests with a query parameter specifying the conversion (e.g., "5usd to mxn"). It then:

- Queries Google Search to fetch the current exchange rate for the specified currency pair.
- Parses the HTML response from Google to extract exchange rate and converted amount.
- Formats the extracted data into a user-friendly response.
- Returns the conversion result as a JSON response to the requester.

**Target Use Cases:**  
- Providing a simple API endpoint for currency conversion without relying on dedicated currency exchange APIs.  
- Quick currency rate checks integrated into other systems or chatbots via webhook calls.

**Logical Blocks:**  
- **1.1 Input Reception & Validation:** Receives and validates incoming webhook requests with the required query parameter.  
- **1.2 Exchange Rate Retrieval:** Makes an HTTP request to Google Search with the currency conversion query.  
- **1.3 Conversion Data Extraction:** Parses Google’s HTML search results to extract exchange rate and converted amount.  
- **1.4 Response Formatting:** Formats the conversion data into a readable message with metadata.  
- **1.5 Response Delivery:** Sends back the formatted conversion or error response to the HTTP client.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

**Overview:**  
This block captures incoming HTTP GET requests to a specific webhook path and checks if the required query parameter `q` exists. If missing, it sends an error response.

**Nodes Involved:**  
- Webhook  
- Check Query Parameter (If)  
- Error Response (Respond to Webhook)  
- Sticky Notes: "Captures GET requests with query parameter q", "Validates that the required query parameter exists", "Handles missing query parameter errors"

**Node Details:**  

- **Webhook**  
  - Type: Webhook (entry point)  
  - Configuration: Listens on path `/currency-converter` for GET requests; response mode set to respond from a downstream node.  
  - Key variables: Expects query parameter `q` like `?q=5usd to mxn`.  
  - Inputs: External HTTP requests  
  - Outputs: Connects to "Check Query Parameter"  
  - Edge cases: Missing query parameter or malformed URL requests.

- **Check Query Parameter**  
  - Type: If (conditional node)  
  - Configuration: Checks existence (strict string operation) of `{{$json.query.q}}` in the incoming JSON.  
  - Inputs: From Webhook  
  - Outputs: If true, proceeds to "Fetch Exchange Rate"; if false, proceeds to "Error Response".  
  - Edge cases: Empty or null `q` parameter.

- **Error Response**  
  - Type: Respond to Webhook  
  - Configuration:  
    - HTTP Status: 400 Bad Request  
    - Response body: JSON error message indicating missing `q` parameter with usage hint.  
  - Inputs: From "Check Query Parameter" false output  
  - Outputs: None (final response)  
  - Edge cases: None; handles client errors gracefully.

---

#### 1.2 Exchange Rate Retrieval

**Overview:**  
Performs an HTTP GET request to Google Search with the encoded currency conversion query to retrieve raw HTML data, which presumably contains the exchange rate.

**Nodes Involved:**  
- Fetch Exchange Rate (HTTP Request)  
- Sticky Note: "Makes HTTP request to Google search for exchange rates"

**Node Details:**  

- **Fetch Exchange Rate**  
  - Type: HTTP Request  
  - Configuration:  
    - Method: GET  
    - URL: Dynamically constructed as `https://www.google.com/search?q={{ encodeURIComponent($json.query.q) }}+exchange+rate`  
    - No authentication or headers configured (relies on public Google Search).  
  - Inputs: From "Check Query Parameter" true output  
  - Outputs: Raw HTML content from Google search results  
  - Edge cases:  
    - Google blocking automated requests or rate limiting  
    - Changes in Google search HTML structure could break downstream parsing  
    - Network errors or timeouts

---

#### 1.3 Conversion Data Extraction

**Overview:**  
Processes the HTML response from Google to extract the amount, source currency, target currency, exchange rate, and converted amount. It supports multiple regex patterns to handle different Google result formats and provides fallback error messages.

**Nodes Involved:**  
- Extract Conversion Data (Code)  
- Sticky Note: "Processes HTML response to extract conversion data"

**Node Details:**  

- **Extract Conversion Data**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Parses the query parameter `q` to extract amount, fromCurrency, and toCurrency using regex `/(\d+(?:\.\d+)?)\s*([a-zA-Z]{3})\s+to\s+([a-zA-Z]{3})/i`  
    - Applies multiple regex patterns against HTML content to extract exchange rate and converted amount:  
      1. Patterns like "100 USD = 85.5 MXN"  
      2. "1 USD = X MXN"  
      3. Direct conversion result matching the amount and currency.  
    - Calculates exchange rate and converted amount accordingly.  
    - Fallback: Attempts to find any suitable number if patterns fail.  
    - Returns error JSON if extraction fails or query format invalid.  
  - Inputs: HTML content from "Fetch Exchange Rate", original query from "Webhook"  
  - Outputs: JSON with extracted data or error  
  - Edge cases:  
    - Query format invalid or missing  
    - HTML content structure changes breaking regex patterns  
    - No matches found for exchange rate  
    - Parsing number failures due to formatting (commas, dots)  
    - Potential time zone or timestamp inconsistencies

---

#### 1.4 Response Formatting

**Overview:**  
Formats the extracted currency conversion data into a clear human-readable string and packages additional metadata for structured response.

**Nodes Involved:**  
- Format Currency Response (Code)  
- Sticky Note: "Formats the result into a user-friendly response"

**Node Details:**  

- **Format Currency Response**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Checks if input contains error; if so, returns error message with HTTP 400 status.  
    - Otherwise, formats string as: "`{amount} {fromCurrency} = {convertedAmount} {toCurrency}`" with converted amount rounded to two decimals.  
    - Adds metadata: original amount, currencies, exchange rate (6 decimals), converted amount, timestamp, and original query string.  
    - Packages response JSON with keys `response`, `data`, and `statusCode`.  
  - Inputs: From "Extract Conversion Data"  
  - Outputs: Formatted JSON response  
  - Edge cases:  
    - Handling missing or null data fields gracefully  
    - Avoiding exceptions on undefined values

---

#### 1.5 Response Delivery

**Overview:**  
Sends the final JSON response back to the webhook client with proper content-type headers.

**Nodes Involved:**  
- Send Conversion Response (Respond to Webhook)  
- Sticky Note: "Returns the formatted response"

**Node Details:**  

- **Send Conversion Response**  
  - Type: Respond to Webhook  
  - Configuration:  
    - Response Headers: `Content-Type: application/json`  
    - Responds with JSON body from input field `response` (formatted string)  
  - Inputs: From "Format Currency Response"  
  - Outputs: None (ends workflow with HTTP response)  
  - Edge cases:  
    - Network or connection issues delivering HTTP response

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                          | Input Node(s)          | Output Node(s)             | Sticky Note                                                     |
|-------------------------|------------------------|----------------------------------------|------------------------|----------------------------|----------------------------------------------------------------|
| Webhook                 | Webhook                | Entry point capturing GET requests     | External HTTP          | Check Query Parameter       | Captures GET requests with query parameter q                   |
| Check Query Parameter    | If                     | Validates presence of query parameter  | Webhook                | Fetch Exchange Rate, Error Response | Validates that the required query parameter exists             |
| Error Response          | Respond to Webhook      | Sends error if query parameter missing | Check Query Parameter   | None                       | Handles missing query parameter errors                          |
| Fetch Exchange Rate      | HTTP Request           | Fetches Google Search results           | Check Query Parameter  | Extract Conversion Data      | Makes HTTP request to Google search for exchange rates         |
| Extract Conversion Data  | Code                   | Parses HTML to extract conversion data | Fetch Exchange Rate    | Format Currency Response    | Processes HTML response to extract conversion data             |
| Format Currency Response | Code                   | Formats conversion result and metadata | Extract Conversion Data | Send Conversion Response    | Formats the result into a user-friendly response                |
| Send Conversion Response | Respond to Webhook      | Sends final JSON response back to client| Format Currency Response | None                      | Returns the formatted response                                  |
| Sticky Note             | Sticky Note            | Comments and annotations                | -                      | -                          | See sticky notes per nodes above                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Set HTTP Method: GET  
   - Set Path: `currency-converter`  
   - Response Mode: `Response Node`  
   - This node will capture incoming requests with query parameter `q`.

2. **Create If Node ("Check Query Parameter")**  
   - Type: If  
   - Condition: Check if expression `{{$json.query.q}}` exists (string operation, strict, case-sensitive).  
   - Connect Webhook output to this node’s input.

3. **Create Respond to Webhook Node ("Error Response")**  
   - Type: Respond to Webhook  
   - HTTP Status Code: 400  
   - Response Headers: Add `Content-Type: application/json`  
   - Response Body (raw JSON):  
     ```json
     { "error": "Missing query parameter 'q'. Please use format: ?q=5usd+to+mxn" }
     ```  
   - Connect If node’s **false** output to this node.

4. **Create HTTP Request Node ("Fetch Exchange Rate")**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: Use expression:  
     ```
     https://www.google.com/search?q={{ encodeURIComponent($json.query.q) }}+exchange+rate
     ```  
   - No authentication or extra headers needed.  
   - Connect If node’s **true** output to this node.

5. **Create Code Node ("Extract Conversion Data")**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the provided custom code that parses the HTML response to extract amount, currencies, exchange rate, and converted amount.  
   - Inputs: Takes Google search HTML content from HTTP Request node and original query from Webhook.  
   - Connect "Fetch Exchange Rate" output to this node.

6. **Create Code Node ("Format Currency Response")**  
   - Type: Code  
   - Language: JavaScript  
   - Insert code that formats the extraction output into a readable string and JSON with metadata.  
   - Connect "Extract Conversion Data" output to this node.

7. **Create Respond to Webhook Node ("Send Conversion Response")**  
   - Type: Respond to Webhook  
   - Response Headers: Add `Content-Type: application/json`  
   - Respond With: JSON  
   - Response Body: Use expression `{{$json.response}}` (formatted string from previous node)  
   - Connect "Format Currency Response" output to this node.

8. **Add Sticky Notes** (optional, for documentation inside n8n)  
   - Add sticky notes near each logical block describing their purpose as detailed in the Sticky Notes section above.

9. **Connect all nodes in the order:**  
   Webhook -> Check Query Parameter  
   - True -> Fetch Exchange Rate -> Extract Conversion Data -> Format Currency Response -> Send Conversion Response  
   - False -> Error Response

10. **Activate the workflow** and test by sending requests like:  
    ```
    GET /currency-converter?q=5usd to mxn
    ```

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                               |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| The workflow relies on parsing Google Search HTML, which is subject to change without notice.       | Consider maintaining and updating regex patterns regularly.                   |
| Google may block automated scraping or rate limit requests; for high volume, consider official APIs.| https://developers.google.com/custom-search/v1/                             |
| Query format must be: `<amount><space><3-letter fromCurrency> to <3-letter toCurrency>` (e.g., 5usd to mxn) | Important for correct parsing in code node "Extract Conversion Data".        |
| Response includes both a formatted string and detailed JSON metadata for flexibility in integrations.| Useful for downstream systems that may need raw numeric data or timestamps.  |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.