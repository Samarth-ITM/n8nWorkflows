Generating Keywords using Google Autosuggest

https://n8nworkflows.xyz/workflows/generating-keywords-using-google-autosuggest-2155


# Generating Keywords using Google Autosuggest

### 1. Workflow Overview

This workflow is designed to generate relevant keywords for SEO, articles, and social media marketing purposes by leveraging Google Autosuggest data. It accepts a keyword query through a webhook, fetches keyword suggestions from Google’s autosuggest service, processes the data, and returns a structured array of suggested keywords.  
The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the keyword query via a webhook URL with a query parameter.
- **1.2 Google Autosuggest Request:** Sends the query to Google’s autosuggest API and receives XML-formatted suggestions.
- **1.3 Data Parsing and Cleaning:** Parses the XML response, extracts keyword suggestions, and formats them into an array.
- **1.4 Aggregation and Response:** Aggregates all keyword suggestions into a single array and returns them via the webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block handles receiving the external request containing the user’s keyword query. It acts as the entry point of the workflow.

- **Nodes Involved:**  
  - Receive Keyword (Webhook)

- **Node Details:**

  - **Receive Keyword**  
    - Type: Webhook node  
    - Role: Listens for HTTP requests at a specific URL path; extracts query parameter `q` containing the keyword(s).  
    - Configuration:  
      - Path: A unique webhook path (UUID-based) to be called externally.  
      - Response Mode: Waits for the final node’s response to return data.  
    - Expressions: Accesses query parameter via `{{$json.query.q}}` in downstream nodes.  
    - Inputs: None (start node)  
    - Outputs: Passes the received data to the next node "Autogenerate Keywords".  
    - Edge cases:  
      - Missing or empty `q` parameter may cause empty or invalid downstream requests.  
      - Unauthorized or unwanted requests are not explicitly filtered here.  
      - Webhook downtime or invalid URL will cause failure to receive requests.

#### 2.2 Google Autosuggest Request

- **Overview:**  
  This block sends the user query to Google Autosuggest’s API and receives XML data with keyword suggestions.

- **Nodes Involved:**  
  - Autogenerate Keywords (HTTP Request)  
  - Format Keywords (XML Parser)

- **Node Details:**

  - **Autogenerate Keywords**  
    - Type: HTTP Request node  
    - Role: Sends a GET request to Google’s autocomplete API with the query parameter embedded.  
    - Configuration:  
      - URL: `https://google.com/complete/search?output=toolbar&gl=US&q={{ $json.query.q }}`  
      - HTTP Method: GET (default)  
      - No authentication required.  
    - Expressions: Injects the query parameter dynamically from webhook input.  
    - Inputs: Receives the webhook data.  
    - Outputs: Returns raw XML response containing keyword suggestions.  
    - Edge cases:  
      - Network errors or rate limiting by Google may cause failures.  
      - Invalid or empty queries may return empty or error responses.

  - **Format Keywords**  
    - Type: XML node  
    - Role: Converts the XML response into JSON format for easier processing.  
    - Configuration: Default XML parsing options.  
    - Inputs: Receives the XML from the HTTP Request node.  
    - Outputs: Outputs parsed JSON data.  
    - Edge cases:  
      - Malformed XML responses will cause parsing errors.  
      - Empty responses yield empty arrays.

#### 2.3 Data Parsing and Cleaning

- **Overview:**  
  This block extracts the keyword suggestions from the parsed JSON, formats them into an array, and prepares them for aggregation.

- **Nodes Involved:**  
  - Split Out (Split Out node)  
  - Clean Keywords (Set node)

- **Node Details:**

  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits the array of suggestions into individual items for downstream processing.  
    - Configuration:  
      - Field to Split Out: `toplevel.CompleteSuggestion` (path to the array in parsed JSON)  
    - Inputs: Receives parsed JSON from XML node.  
    - Outputs: Emits each suggestion as a separate item.  
    - Edge cases:  
      - If the field is missing or empty, no items are emitted.  
      - Unexpected JSON structure changes may cause failures.

  - **Clean Keywords**  
    - Type: Set node  
    - Role: Extracts the actual keyword string from each suggestion and assigns it to a clean array field.  
    - Configuration:  
      - Assignments: Sets field `Keywords` to `{{$json.suggestion.data}}` (the keyword text from each suggestion)  
      - Ignores conversion errors to prevent workflow crashes on malformed data.  
    - Inputs: Single suggestion items from Split Out node.  
    - Outputs: Passes cleaned keyword data to the Aggregate node.  
    - Edge cases:  
      - Missing `suggestion.data` fields may produce undefined or empty values.

#### 2.4 Aggregation and Response

- **Overview:**  
  Aggregates all individual keyword items into a single array and returns them as the webhook response.

- **Nodes Involved:**  
  - Aggregate (Aggregate node)  
  - return Keywords (Respond to Webhook node)

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Combines all incoming keyword items into a single aggregated array.  
    - Configuration:  
      - Fields to Aggregate: Aggregates the `Keywords` field into an array.  
    - Inputs: Receives cleaned keyword items from Clean Keywords node.  
    - Outputs: Passes the aggregated array downstream.  
    - Edge cases:  
      - If no keywords are received, output may be an empty array.

  - **return Keywords**  
    - Type: Respond to Webhook node  
    - Role: Sends the aggregated keyword array back as the HTTP response to the initial webhook call.  
    - Configuration:  
      - Respond With: All incoming items (here, the aggregated array).  
    - Inputs: Receives aggregated keywords.  
    - Outputs: Responds to caller, ending the workflow.  
    - Edge cases:  
      - Response delays or failures will cause webhook timeouts.

---

### 3. Summary Table

| Node Name          | Node Type             | Functional Role                           | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                     |
|--------------------|-----------------------|-----------------------------------------|-------------------------|------------------------|-----------------------------------------------------------------------------------------------------------------|
| Receive Keyword    | Webhook               | Receives keyword query via HTTP request | None                    | Autogenerate Keywords   | * If you are using this one, just copy the this webhook url http://localhost:5678/webhook/76a63718-b3cb-4141-bc55-efa614d13f1d?q=keyword%20research * All you need is to change the keyword to your desired keyword and you will be good to go * You can use the keyword with a space and the results will be the same |
| Autogenerate Keywords | HTTP Request         | Sends query to Google Autosuggest API   | Receive Keyword          | Format Keywords         |                                                                                                                 |
| Format Keywords    | XML                   | Parses XML response into JSON            | Autogenerate Keywords    | Split Out               |                                                                                                                 |
| Split Out          | Split Out             | Splits suggestions array into items      | Format Keywords          | Clean Keywords          |                                                                                                                 |
| Clean Keywords     | Set                   | Extracts keyword strings from suggestions | Split Out                | Aggregate               |                                                                                                                 |
| Aggregate          | Aggregate             | Aggregates individual keywords into array | Clean Keywords           | return Keywords         |                                                                                                                 |
| return Keywords    | Respond to Webhook    | Returns aggregated keywords as HTTP response | Aggregate                | None                   |                                                                                                                 |
| Sticky Note        | Sticky Note           | Provides workflow title and purpose      | None                    | None                   | * Generating keywords for your SEO                                                                             |
| Sticky Note1       | Sticky Note           | Usage instructions for the webhook URL   | None                    | None                   | * If you are using this one, just copy the this webhook url http://localhost:5678/webhook/76a63718-b3cb-4141-bc55-efa614d13f1d?q=keyword%20research * All you need is to change the keyword to your desired keyword and you will be good to go * You can use the keyword with a space and the results will be the same |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** node named "Receive Keyword".  
   - Set the HTTP Method to GET (default).  
   - Set the Path to a unique string (e.g., `76a63718-b3cb-4141-bc55-efa614d13f1d`).  
   - Ensure the node is set to wait for the last node to respond to enable returning data.  

2. **Create HTTP Request Node**  
   - Add an **HTTP Request** node named "Autogenerate Keywords".  
   - Connect "Receive Keyword" output to this node’s input.  
   - Configure the URL as:  
     `https://google.com/complete/search?output=toolbar&gl=US&q={{ $json.query.q }}`  
   - Use HTTP GET method. No auth needed.  
   - Enable "Keep Response Headers" if desired (optional).  

3. **Add XML Node**  
   - Add an **XML** node named "Format Keywords".  
   - Connect "Autogenerate Keywords" output to this node’s input.  
   - Use default XML parsing options to convert XML to JSON.  

4. **Add Split Out Node**  
   - Add a **Split Out** node named "Split Out".  
   - Connect "Format Keywords" output to this node.  
   - Set "Field To Split Out" to `toplevel.CompleteSuggestion`.  

5. **Add Set Node**  
   - Add a **Set** node named "Clean Keywords".  
   - Connect "Split Out" output to this node.  
   - Configure to assign a new field named `Keywords`.  
   - Set value to `{{$json.suggestion.data}}` (value extracted from each suggestion).  
   - Enable "Ignore Conversion Errors" to avoid crashes on unexpected data.  

6. **Add Aggregate Node**  
   - Add an **Aggregate** node named "Aggregate".  
   - Connect "Clean Keywords" output to this node.  
   - Configure to aggregate the field `Keywords` into an array.  

7. **Add Respond to Webhook Node**  
   - Add a **Respond to Webhook** node named "return Keywords".  
   - Connect "Aggregate" output to this node.  
   - Set "Respond With" option to "All Incoming Items" to return the aggregated keywords array as the HTTP response.  

8. **Connect Nodes**  
   - Final connection chain:  
     Receive Keyword → Autogenerate Keywords → Format Keywords → Split Out → Clean Keywords → Aggregate → return Keywords  

9. **Activate Workflow**  
   - Save and activate the workflow.  
   - Use the webhook URL to call it with a query parameter `q` set to your keyword, e.g.:  
     `http://<your-n8n-host>/webhook/76a63718-b3cb-4141-bc55-efa614d13f1d?q=keyword research`  

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is optimized for SEO pros, content marketers, and social media marketers to generate relevant keyword suggestions. | Workflow purpose and target audience.                                                           |
| Example webhook usage: `http://localhost:5678/webhook/76a63718-b3cb-4141-bc55-efa614d13f1d?q=keyword%20research`                 | Provided usage example in sticky notes for testing and deployment.                              |
| Google Autosuggest API endpoint used: `https://google.com/complete/search?output=toolbar&gl=US&q=YOUR_QUERY`                    | Reference for the autosuggest API format used in the HTTP Request node.                         |
| The workflow returns an array of keywords extracted from Google’s suggestions, making it easy to integrate with other systems. | Integration note for developers planning to use or extend the workflow.                         |

---

This structured documentation fully describes the "Generating Keywords using Google Autosuggest" workflow, enabling understanding, reproduction, and modification by both advanced users and automation tools.