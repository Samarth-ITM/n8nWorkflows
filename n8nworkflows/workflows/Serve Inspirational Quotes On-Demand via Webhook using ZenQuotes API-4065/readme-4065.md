Serve Inspirational Quotes On-Demand via Webhook using ZenQuotes API

https://n8nworkflows.xyz/workflows/serve-inspirational-quotes-on-demand-via-webhook-using-zenquotes-api-4065


# Serve Inspirational Quotes On-Demand via Webhook using ZenQuotes API

### 1. Workflow Overview

This workflow serves inspirational quotes on demand via an HTTP webhook by integrating with the free ZenQuotes API. It is designed to retrieve a batch of random motivational quotes and return them as a clean, formatted JSON response to the webhook caller. The workflow is ideal for developers, content creators, or educators who want to embed dynamic inspirational content into their applications or websites without building a custom backend.

The workflow’s logic is divided into the following functional blocks:

- **1.1 Input Reception:** Handles incoming HTTP requests via a webhook.
- **1.2 Fetching Quotes:** Calls the ZenQuotes API to retrieve multiple random quotes.
- **1.3 Data Formatting:** Processes and formats the raw quote data into a readable string combining quote text and author.
- **1.4 Response Delivery:** Returns the formatted quotes as a JSON array in the HTTP response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming HTTP requests on a specified webhook path and triggers the workflow execution.

**Nodes Involved:**  
- Webhook  
- Sticky Note1 (commentary node)

**Node Details:**

- **Webhook**  
  - *Type & Role:* HTTP Webhook node; entry point of the workflow.  
  - *Configuration:* Listens on a customizable path (default is a UUID path). Accepts any HTTP method (GET or POST).  
  - *Key Variables:* None specifically used here.  
  - *Connections:* Output triggers “Get Random Quote from ZenQuotes” node.  
  - *Version:* v2 (no special version requirements).  
  - *Potential Failures:* Invalid path configuration, webhook not activated, or network issues preventing incoming requests.  
  - *Sticky Note1:* “Receives incoming requests” — clarifies node role.

- **Sticky Note1**  
  - Provides user-friendly context; no technical configuration.

---

#### 1.2 Fetching Quotes

**Overview:**  
This block sends an HTTP GET request to the ZenQuotes API endpoint to fetch a specified number of random inspirational quotes.

**Nodes Involved:**  
- Get Random Quote from ZenQuotes  
- Sticky Note2 (commentary node)

**Node Details:**

- **Get Random Quote from ZenQuotes**  
  - *Type & Role:* HTTP Request node; API integration to fetch random quotes from ZenQuotes.  
  - *Configuration:*  
    - URL: `https://zenquotes.io/api/random`  
    - Query Parameter: `count=5` (requests 5 random quotes)  
    - Method: GET (default)  
    - No authentication required (free API).  
  - *Key Expressions/Variables:* Query parameter `count` is statically set to 5 but can be changed manually.  
  - *Connections:* Receives trigger from Webhook; outputs JSON response to “Format data”.  
  - *Version:* 4.2  
  - *Potential Failures:*  
    - API downtime or rate limiting.  
    - Network timeouts.  
    - Unexpected API response format changes.  
  - *Sticky Note2:* “Fetches random quotes” — clarifies node role.

- **Sticky Note2**  
  - Provides user guidance on the quote-fetching step.

---

#### 1.3 Data Formatting

**Overview:**  
Transforms each quote object from the API into a single string formatted as `"“quote” – author"`, preparing the data for returning to the caller.

**Nodes Involved:**  
- Format data (Set node)  
- Sticky Note3 (commentary node)

**Node Details:**

- **Format data**  
  - *Type & Role:* Set node; used here to restructure and format incoming JSON data.  
  - *Configuration:*  
    - Assigns a new field `quotes` using the expression:  
      ```  
      ="\"{{ $json.q }}\" - {{ $json.a }}"  
      ```  
      where `$json.q` is the quote text and `$json.a` is the author.  
    - This transforms each quote object into a formatted string.  
  - *Key Expressions:* Uses n8n expression syntax to interpolate quote and author fields into a formatted string.  
  - *Connections:* Inputs from “Get Random Quote from ZenQuotes”; outputs to “Send response”.  
  - *Version:* 3.4  
  - *Potential Failures:*  
    - Expression syntax errors if API response structure changes.  
    - Missing fields (`q` or `a`) in the API response could cause empty or malformed strings.  
  - *Sticky Note3:* “Formats into ‘quote – author’ strings” — clarifies node role.

- **Sticky Note3**  
  - Provides context on the formatting operation.

---

#### 1.4 Response Delivery

**Overview:**  
Sends the formatted quotes back to the original HTTP requestor as a JSON array.

**Nodes Involved:**  
- Send response  
- Sticky Note4 (commentary node)

**Node Details:**

- **Send response**  
  - *Type & Role:* Respond to Webhook node; sends HTTP response back to the webhook caller.  
  - *Configuration:*  
    - Response type: JSON  
    - Response body:  
      ```json
      {
        "quote": "{{ $json.quotes }}"
      }
      ```  
      This returns the formatted quote string(s) under the key `quote`.  
  - *Key Expressions:* Uses `{{ $json.quotes }}` to insert the formatted quote string(s) from the previous node.  
  - *Connections:* Receives input from “Format data”.  
  - *Version:* 1.1  
  - *Potential Failures:*  
    - If input data is empty or malformed, response may be invalid or empty.  
    - Network or permission errors sending HTTP response.  
  - *Sticky Note4:* “Sends back JSON array” — clarifies node role.

- **Sticky Note4**  
  - Provides clarity on the final output step.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                  | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                 |
|-------------------------------|---------------------------|---------------------------------|----------------------------------|--------------------------------|------------------------------------------------------------------------------|
| Webhook                       | Webhook                   | Receives incoming HTTP requests | –                                | Get Random Quote from ZenQuotes | Receives incoming requests                                                   |
| Get Random Quote from ZenQuotes | HTTP Request              | Fetches random quotes from API  | Webhook                          | Format data                    | Fetches random quotes                                                        |
| Format data                  | Set                       | Formats quotes into strings     | Get Random Quote from ZenQuotes  | Send response                 | Formats into ‘quote – author’ strings                                       |
| Send response                 | Respond to Webhook         | Sends JSON response             | Format data                     | –                              | Sends back JSON array                                                        |
| Sticky Note                  | Sticky Note               | Provides beginner-friendly note | –                                | –                              | ## Beginner friendly workflow on how to work with API and format the data   |
| Sticky Note1                 | Sticky Note               | Commentary                      | –                                | –                              | Receives incoming requests                                                   |
| Sticky Note2                 | Sticky Note               | Commentary                      | –                                | –                              | Fetches random quotes                                                        |
| Sticky Note3                 | Sticky Note               | Commentary                      | –                                | –                              | Formats into ‘quote – author’ strings                                       |
| Sticky Note4                 | Sticky Note               | Commentary                      | –                                | –                              | Sends back JSON array                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Add a **Webhook** node.  
   - Set the **HTTP Method** to accept any method, or specify GET/POST as needed.  
   - Set the **path** to a unique identifier or desired endpoint path (e.g., `/inspire`).  
   - Save the node.

2. **Create the HTTP Request Node**  
   - Add an **HTTP Request** node named “Get Random Quote from ZenQuotes”.  
   - Set the **HTTP Method** to GET.  
   - Set the **URL** to `https://zenquotes.io/api/random`.  
   - Under **Query Parameters**, add parameter:  
     - Name: `count`  
     - Value: `5` (or desired number of quotes).  
   - No authentication is required.  
   - Connect the Webhook node’s output to this node’s input.

3. **Create the Set Node for Formatting**  
   - Add a **Set** node named “Format data”.  
   - Under **Values to Set**, add a new field:  
     - Name: `quotes`  
     - Type: String  
     - Value:  
       ```
       ="\"{{ $json.q }}\" - {{ $json.a }}"
       ```  
     - This formats each quote as: `"quote" - author`.  
   - Connect the HTTP Request node output to this Set node input.

4. **Create the Respond to Webhook Node**  
   - Add a **Respond to Webhook** node named “Send response”.  
   - Set **Respond With** to `JSON`.  
   - Set the **Response Body** to:  
     ```json
     {
       "quote": "{{ $json.quotes }}"
     }
     ```  
   - Connect the Set node output to this Respond to Webhook node input.

5. **Add Sticky Notes for Documentation (Optional)**  
   - Add **Sticky Note** nodes to describe each block or node for clarity, matching the content from the original workflow.

6. **Activate the Workflow**  
   - Save and activate the workflow.  
   - Test by sending a GET or POST request to:  
     ```
     https://<your-n8n-domain>/webhook/<your-path>
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                          |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This beginner-friendly workflow demonstrates how to integrate with a public API and format data in n8n.        | Included as Sticky Note in the workflow                   |
| ZenQuotes API documentation: https://zenquotes.io/                                                             | Official API resource for reference                       |
| Ideal use cases include Slack/Discord bots, website widgets, and educational inspiration delivery.             | Workflow description section                              |
| Adjust the `count` query parameter in the HTTP Request node to change how many quotes you receive per request. | Setup instructions in the workflow description           |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.