Currency Conversion Workflow

https://n8nworkflows.xyz/workflows/currency-conversion-workflow-2691


# Currency Conversion Workflow

### 1. Workflow Overview

This workflow automates currency conversion by intercepting user queries via a webhook, fetching real-time exchange rates from Google search results, extracting relevant data from the HTML response, formatting the conversion output, and returning the result to the requester. It is designed to avoid reliance on third-party currency conversion APIs by leveraging publicly available Google search data, making it cost-effective and adaptable.

The workflow is logically divided into the following blocks:

- **1.1 Capture Conversion Query:** Receives user input through a webhook GET request with a query parameter specifying the conversion (e.g., `5usd to mxn`).
- **1.2 Fetch Exchange Rate:** Performs an HTTP GET request to Google Search to retrieve the current exchange rate data embedded in the search results.
- **1.3 Extract Conversion Data:** Parses the HTML response from Google to extract the precise exchange rate and related information using CSS selectors.
- **1.4 Format Currency Response:** Processes and formats the extracted data into a clear, human-readable conversion string.
- **1.5 Send Conversion Response:** Sends the formatted conversion result back to the original requester via the webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Capture Conversion Query

- **Overview:**  
  This block initiates the workflow by capturing the user’s currency conversion query via a webhook configured to accept GET requests. It expects a query parameter `q` formatted as `<amount><source_currency> to <target_currency>` (e.g., `5usd to mxn`).

- **Nodes Involved:**  
  - Capture Conversion Query (Webhook)

- **Node Details:**

  - **Capture Conversion Query**  
    - *Type & Role:* Webhook node; entry point for receiving HTTP GET requests.  
    - *Configuration:*  
      - Webhook method set to GET.  
      - Expects query parameter `q` containing the conversion request string.  
    - *Expressions/Variables:*  
      - Accesses `{{$json["query"]["q"]}}` to retrieve the user input.  
    - *Input/Output:*  
      - No input nodes (start node).  
      - Outputs to "Fetch Exchange Rate" node.  
    - *Version:* 2  
    - *Edge Cases / Failures:*  
      - Missing or malformed `q` parameter leads to failure or incorrect processing.  
      - Non-conforming input syntax (e.g., missing "to" keyword) may cause downstream extraction errors.  
    - *Sticky Note:*  
      - Advises testing with tools like Postman or browser to verify GET request reception.  
      - Emphasizes strict adherence to input format for reliable processing.

#### 2.2 Fetch Exchange Rate

- **Overview:**  
  This block queries Google Search with the user’s conversion request to retrieve the latest exchange rate data embedded in the HTML response.

- **Nodes Involved:**  
  - Fetch Exchange Rate (HTTP Request)

- **Node Details:**

  - **Fetch Exchange Rate**  
    - *Type & Role:* HTTP Request node; performs GET request to Google Search.  
    - *Configuration:*  
      - URL dynamically constructed using the query parameter from the webhook, e.g., `https://www.google.com/search?q={{$json["query"]["q"]}}`.  
      - Method: GET.  
      - Response format: HTML (default).  
      - Headers may include User-Agent to mimic browser requests (not explicitly shown but recommended).  
    - *Expressions/Variables:*  
      - Uses expression to inject the query string from webhook input into the URL.  
    - *Input/Output:*  
      - Input from "Capture Conversion Query".  
      - Output to "Extract Conversion Data".  
    - *Version:* 4.2  
    - *Edge Cases / Failures:*  
      - Network errors or Google blocking automated requests (rate limiting, CAPTCHA).  
      - Changes in Google’s search URL structure or query parameters may require updates.  
      - Absence of expected data in the HTML response if Google modifies page layout.  
    - *Sticky Note:*  
      - Notes the economical advantage of avoiding third-party APIs.  
      - Suggests monitoring for changes in Google’s HTML that may affect extraction.

#### 2.3 Extract Conversion Data

- **Overview:**  
  Parses the HTML response from Google to extract the currency conversion rate and related information using CSS selectors.

- **Nodes Involved:**  
  - Extract Conversion Data (HTML Extract)

- **Node Details:**

  - **Extract Conversion Data**  
    - *Type & Role:* HTML Extract node; extracts specific data points from the HTML content.  
    - *Configuration:*  
      - Uses CSS selectors targeting Google’s currency conversion widget elements to extract:  
        - The converted amount.  
        - Source and target currency codes.  
        - Possibly the exchange rate or other metadata.  
      - Configured to execute once per workflow run.  
      - Always outputs data even if extraction partially fails.  
    - *Expressions/Variables:*  
      - CSS selectors configured to match Google’s current HTML structure (modifiable).  
    - *Input/Output:*  
      - Input from "Fetch Exchange Rate".  
      - Output to "Format Currency Response".  
    - *Version:* 1.2  
    - *Edge Cases / Failures:*  
      - Extraction failure if Google changes HTML structure or CSS classes.  
      - Incorrect or malformed input query causing no matching elements.  
      - Partial extraction leading to incomplete data.  
    - *Sticky Note:*  
      - Advises updating CSS selectors if extraction fails.  
      - Emphasizes verifying input format correctness.

#### 2.4 Format Currency Response

- **Overview:**  
  Formats the extracted currency conversion data into a clean, user-friendly string suitable for returning to the requester.

- **Nodes Involved:**  
  - Format Currency Response (Set)

- **Node Details:**

  - **Format Currency Response**  
    - *Type & Role:* Set node; constructs the final response string.  
    - *Configuration:*  
      - Uses expressions to combine extracted values into a string like:  
        `"5 USD = 95 MXN"`  
      - May include trimming, uppercasing currency codes, and formatting numbers for readability.  
    - *Expressions/Variables:*  
      - Accesses extracted data fields from previous node output.  
      - Example expression:  
        ```  
        {{$json["amount"]}} {{$json["sourceCurrency"].toUpperCase()}} = {{$json["convertedAmount"]}} {{$json["targetCurrency"].toUpperCase()}}  
        ```  
    - *Input/Output:*  
      - Input from "Extract Conversion Data".  
      - Output to "Send Conversion Response".  
    - *Version:* 3.4  
    - *Edge Cases / Failures:*  
      - Missing or incomplete extracted data leads to malformed output.  
      - Formatting errors if data types are unexpected.  
    - *Sticky Note:*  
      - Notes possibility to customize output format or add metadata.

#### 2.5 Send Conversion Response

- **Overview:**  
  Sends the formatted conversion result back to the original HTTP requester, completing the workflow cycle.

- **Nodes Involved:**  
  - Send Conversion Response (Respond to Webhook)

- **Node Details:**

  - **Send Conversion Response**  
    - *Type & Role:* Respond to Webhook node; returns HTTP response to the client.  
    - *Configuration:*  
      - Sends the formatted string from the previous node as the response body.  
      - HTTP status code defaults to 200 (OK).  
    - *Expressions/Variables:*  
      - Response body set to the output of "Format Currency Response".  
    - *Input/Output:*  
      - Input from "Format Currency Response".  
      - No output nodes (terminal node).  
    - *Version:* 1.1  
    - *Edge Cases / Failures:*  
      - Failure to send response if previous node output is empty or invalid.  
      - Network issues affecting response delivery.  
    - *Sticky Note:*  
      - Emphasizes efficient and reliable completion of the request-response cycle.

---

### 3. Summary Table

| Node Name                | Node Type             | Functional Role                      | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                          |
|--------------------------|-----------------------|------------------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Capture Conversion Query  | Webhook               | Receives user currency query       | —                        | Fetch Exchange Rate      | Use Postman or browser to test GET requests. Input must follow strict syntax like `5usd to mxn`.   |
| Fetch Exchange Rate       | HTTP Request          | Queries Google for exchange rate   | Capture Conversion Query | Extract Conversion Data  | Economical alternative to APIs. Monitor Google HTML changes affecting extraction.                   |
| Extract Conversion Data   | HTML Extract          | Parses HTML to extract conversion  | Fetch Exchange Rate      | Format Currency Response | Update CSS selectors if extraction fails. Verify input format correctness.                          |
| Format Currency Response  | Set                   | Formats extracted data into string | Extract Conversion Data  | Send Conversion Response | Customize output format or add metadata as needed.                                                 |
| Send Conversion Response  | Respond to Webhook    | Returns formatted response to user | Format Currency Response | —                        | Ensures efficient and reliable response delivery.                                                  |
| Note - Webhook           | Sticky Note           | Documentation note                 | —                        | —                        | See Capture Conversion Query node note.                                                            |
| Note - HTTP Request      | Sticky Note           | Documentation note                 | —                        | —                        | See Fetch Exchange Rate node note.                                                                  |
| Note - HTML Extract      | Sticky Note           | Documentation note                 | —                        | —                        | See Extract Conversion Data node note.                                                             |
| Note - Format Response   | Sticky Note           | Documentation note                 | —                        | —                        | See Format Currency Response node note.                                                            |
| Note - Send Response     | Sticky Note           | Documentation note                 | —                        | —                        | See Send Conversion Response node note.                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Capture Conversion Query"**  
   - Type: Webhook  
   - Method: GET  
   - Path: e.g., `currency-converter` (or your preferred endpoint)  
   - Configure to accept query parameter `q` (e.g., `https://your-webhook-url/currency-converter?q=5usd+to+mxn`)  
   - No authentication required  
   - Save and activate the webhook

2. **Create HTTP Request Node: "Fetch Exchange Rate"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.google.com/search?q={{$json["query"]["q"]}}`  
   - Headers: (optional but recommended)  
     - User-Agent: Set to a common browser user agent string to avoid blocking  
   - Response Format: Default (HTML)  
   - Connect input from "Capture Conversion Query"

3. **Create HTML Extract Node: "Extract Conversion Data"**  
   - Type: HTML Extract  
   - Execute Once: Enabled (to process single response)  
   - Always Output Data: Enabled  
   - Configure CSS selectors to extract:  
     - Converted amount (e.g., selector targeting the element showing the converted value)  
     - Source currency code  
     - Target currency code  
   - Connect input from "Fetch Exchange Rate"  
   - Adjust selectors as needed based on Google’s current HTML structure

4. **Create Set Node: "Format Currency Response"**  
   - Type: Set  
   - Disable "Keep Only Set" (optional, depending on desired output)  
   - Add a new field, e.g., `formattedResponse` (or use default output)  
   - Use expression to format string:  
     ```
     {{$json["amount"]}} {{$json["sourceCurrency"].toUpperCase()}} = {{$json["convertedAmount"]}} {{$json["targetCurrency"].toUpperCase()}}
     ```  
   - Connect input from "Extract Conversion Data"

5. **Create Respond to Webhook Node: "Send Conversion Response"**  
   - Type: Respond to Webhook  
   - Response Body: Set to the formatted string from "Format Currency Response" node  
   - Connect input from "Format Currency Response"

6. **Connect Nodes in Sequence:**  
   - Capture Conversion Query → Fetch Exchange Rate → Extract Conversion Data → Format Currency Response → Send Conversion Response

7. **Activate the Workflow**  
   - Ensure all nodes are properly configured and connected  
   - Activate the workflow to listen for incoming requests

8. **Testing**  
   - Use Postman or a browser to send GET requests to the webhook URL with query parameter `q` formatted as `<amount><source_currency> to <target_currency>`  
   - Verify the response matches expected format, e.g., `5 USD = 95 MXN`

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow avoids third-party API dependencies by scraping Google search results for exchange rates.    | Workflow Overview                                                                                  |
| Input queries must strictly follow the syntax: `<amount><source_currency> to <target_currency>`.           | Capture Conversion Query node sticky note                                                         |
| CSS selectors in the HTML Extract node may require updates if Google modifies their page structure.        | Extract Conversion Data node sticky note                                                          |
| Testing with Postman or browser GET requests is recommended to verify webhook reception and response.      | Capture Conversion Query node sticky note                                                         |
| The workflow can be extended to log conversions, monitor metrics, or trigger additional workflows.         | Workflow Overview - Advanced Features                                                             |
| Example output format: `5 USD = 95 MXN` optimized for readability and practical use.                        | Format Currency Response node sticky note                                                         |
| Tags: `currency conversion`, `automation`, `webhook`, `data extraction`, `AI integration`, `financial automation`, `e-commerce`, `real-time data`, `scalable workflows`. | Workflow metadata                                                                                  |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the Currency Conversion Workflow in n8n. It highlights critical configuration points, potential failure modes, and customization options to ensure robust and scalable operation.