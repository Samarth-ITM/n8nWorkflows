Google Autocomplete Keyword Scraper

https://n8nworkflows.xyz/workflows/google-autocomplete-keyword-scraper-3836


# Google Autocomplete Keyword Scraper

### 1. Workflow Overview

This workflow, titled **Google Autocomplete Keyword Scraper**, is designed primarily for **SEO specialists** and **digital marketers** who want to discover keyword opportunities by leveraging Google's autocomplete suggestions. The workflow automates querying Google autocomplete by appending each letter from A to Z to a user-provided keyword and collecting all suggested completions.

**Target Use Cases:**  
- Keyword research and discovery  
- Content ideation based on trending or related search terms  
- Competitive SEO analysis by uncovering long-tail keywords  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives the initial keyword from a chat interface or other input sources.  
- **1.2 Query Generation:** Generates 26 queries by appending each letter A-Z to the keyword.  
- **1.3 Batch Processing & Rate Limiting:** Processes queries in batches with wait times to avoid Google API blocking.  
- **1.4 Google Autocomplete Requests:** Sends HTTP requests to Google’s autocomplete API for each query.  
- **1.5 Response Parsing:** Extracts autocomplete suggestions from the API responses.  
- **1.6 Keyword Aggregation:** Merges all suggestions into a single list.  
- **1.7 Output Delivery:** Returns the aggregated list of keywords to the requester.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures the initial keyword input from the user or an external source to start the scraping process.

- **Nodes Involved:**  
  - Get Keyword

- **Node Details:**  

  - **Get Keyword**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry node that triggers the workflow upon receiving a chat input keyword.  
    - Configuration: Listens for chat input; no additional parameters configured.  
    - Expressions/Variables: Captures the keyword as `$input.first().json.chatInput` in downstream nodes.  
    - Connections: Outputs to "Generate A-Z Queries".  
    - Edge Cases: No keyword or empty input could cause downstream nodes to fail or return empty results. Input validation is recommended upstream.  
    - Notes: Alternative input sources suggested include Google Sheets, webhooks, or messaging apps.

#### 1.2 Query Generation

- **Overview:**  
  Generates 26 queries by appending each letter of the alphabet (a-z) to the input keyword, including a space separator.

- **Nodes Involved:**  
  - Generate A-Z Queries  
  - Loop Over Items

- **Node Details:**  

  - **Generate A-Z Queries**  
    - Type: `Code` node  
    - Role: Creates an array of query strings combining the keyword and each letter from a to z.  
    - Configuration: Uses JavaScript to split the alphabet and map each letter to a query string `${keyword} ${letter}`.  
    - Expressions/Variables: Reads keyword from `$input.first().json.chatInput`.  
    - Output: 26 JSON items with a `query` property.  
    - Connections: Outputs to "Loop Over Items".  
    - Edge Cases: If keyword is missing or empty, queries will be malformed (e.g., " undefined a").  

  - **Loop Over Items**  
    - Type: `SplitInBatches` node  
    - Role: Splits the 26 queries into batches of 10 to control request rate.  
    - Configuration: Batch size set to 10.  
    - Connections: Outputs two branches: one to "Extract Keywords" (for aggregated results) and one to "Google Autocomplete" (for processing each batch).  
    - Edge Cases: Batch processing helps avoid rate limits; improper batch size or missing wait times could cause API blocking.

#### 1.3 Batch Processing & Rate Limiting

- **Overview:**  
  Introduces a delay between batches to prevent Google from blocking requests due to rapid querying.

- **Nodes Involved:**  
  - Wait 1s

- **Node Details:**  

  - **Wait 1s**  
    - Type: `Wait` node  
    - Role: Pauses execution for 1 second between batches.  
    - Configuration: Wait time set to 1 second.  
    - Connections: Receives input from "Code" node (which parses API response), outputs to "Loop Over Items" to continue batch processing.  
    - Edge Cases: Insufficient wait time could cause Google to block requests; too long could slow workflow unnecessarily.

#### 1.4 Google Autocomplete Requests

- **Overview:**  
  Sends HTTP GET requests to Google’s autocomplete API for each query string generated.

- **Nodes Involved:**  
  - Google Autocomplete  
  - Code (response parsing)

- **Node Details:**  

  - **Google Autocomplete**  
    - Type: `HTTP Request` node  
    - Role: Queries Google autocomplete API with the current query string.  
    - Configuration:  
      - URL: `https://suggestqueries.google.com/complete/search?client=firefox&hl=en&oe=utf-8&q={{ $json.query }}`  
      - Method: GET  
      - No authentication required.  
    - Expressions: Uses `{{ $json.query }}` to inject the current query.  
    - Connections: Outputs raw JSON response to "Code" node.  
    - Edge Cases:  
      - Google may block requests if rate limits exceeded.  
      - Network timeouts or connectivity issues.  
      - Language code `hl=en` can be modified to change autocomplete language.  
    - Notes: Sticky note explains language adaptation by changing `hl` parameter.

  - **Code (response parsing)**  
    - Type: `Code` node  
    - Role: Parses the JSON response from Google to extract autocomplete keywords.  
    - Configuration: Parses the JSON string from `$json.data` and extracts the second element (index 1) which contains the suggestions array.  
    - Output: JSON object with a `keywords` property containing the array of suggestions.  
    - Connections: Outputs to "Wait 1s" node.  
    - Edge Cases: Malformed JSON or unexpected response structure could cause parsing errors.

#### 1.5 Keyword Aggregation

- **Overview:**  
  Collects all autocomplete suggestions from each batch and merges them into a single consolidated list.

- **Nodes Involved:**  
  - Extract Keywords

- **Node Details:**  

  - **Extract Keywords**  
    - Type: `Code` node  
    - Role: Iterates over all input items and merges their `keywords` arrays into one flat list.  
    - Configuration: Uses JavaScript to concatenate all keyword arrays from input.  
    - Output: Single JSON object with `keywords` property containing all aggregated keywords.  
    - Connections: Outputs to "Return Keywords".  
    - Edge Cases: Empty inputs or missing `keywords` fields could result in empty output.

#### 1.6 Output Delivery

- **Overview:**  
  Sends the final aggregated list of keywords back to the requester via webhook response.

- **Nodes Involved:**  
  - Return Keywords

- **Node Details:**  

  - **Return Keywords**  
    - Type: `Respond to Webhook` node  
    - Role: Returns the final keyword list as the HTTP response to the original trigger.  
    - Configuration: Default response options; returns the JSON payload from previous node.  
    - Connections: None (terminal node).  
    - Edge Cases: If upstream nodes fail or return empty data, response will reflect that.

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                          | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                              |
|---------------------|-------------------------------|----------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| Get Keyword         | Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) | Receives initial keyword input          | —                      | Generate A-Z Queries      | You could also get this initial keyword from: a Google Sheet, webhook/form, or messaging apps.          |
| Generate A-Z Queries | Code                          | Generates queries by appending letters | Get Keyword            | Loop Over Items           | This code adds a blank space and a letter (a-z) to the keyword, producing 26 queries.                   |
| Loop Over Items     | SplitInBatches                | Processes queries in batches of 10      | Generate A-Z Queries, Wait 1s | Extract Keywords, Google Autocomplete | The 26 items are processed in batches of 10 to avoid Google API blocking.                               |
| Google Autocomplete | HTTP Request                  | Sends autocomplete queries to Google   | Loop Over Items         | Code (response parsing)   | Change `&hl=en` to adapt language (e.g., `&hl=fr` for French).                                         |
| Code (response parsing) | Code                       | Parses Google autocomplete JSON response | Google Autocomplete     | Wait 1s                   | —                                                                                                      |
| Wait 1s             | Wait                         | Adds 1-second delay between batches     | Code                    | Loop Over Items           | Necessary wait to prevent Google API blocking.                                                         |
| Extract Keywords    | Code                         | Aggregates all keywords into one list   | Loop Over Items         | Return Keywords           | This code gathers all keywords from all batches into a single list.                                    |
| Return Keywords     | Respond to Webhook            | Returns final keyword list to requester | Extract Keywords        | —                        | Use this node to send the keyword list back via webhook response.                                     |
| Sticky Note         | Sticky Note                  | Informational note                      | —                      | —                        | ## Type a Keyword and Discover What People Search on Google. Explains the workflow purpose and example. |
| Sticky Note1        | Sticky Note                  | Informational note                      | —                      | —                        | ## Exporting the Keywords. Suggests ways to export results (webhook, email, file, website).             |
| Sticky Note2        | Sticky Note                  | Informational note                      | —                      | —                        | ## Adapt the Language. Explains how to change Google Autocomplete language parameter.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the entry node:**  
   - Add a **Chat Trigger** node (`@n8n/n8n-nodes-langchain.chatTrigger`) named **Get Keyword**.  
   - No special parameters needed; this node listens for chat input.  

2. **Generate A-Z Queries:**  
   - Add a **Code** node named **Generate A-Z Queries**.  
   - Paste the following JavaScript:  
     ```javascript
     const keyword = $input.first().json.chatInput;
     const alphabet = "abcdefghijklmnopqrstuvwxyz".split("");
     return alphabet.map(letter => ({
       json: { query: `${keyword} ${letter}` }
     }));
     ```  
   - Connect **Get Keyword** output to this node input.

3. **Batch processing:**  
   - Add a **SplitInBatches** node named **Loop Over Items**.  
   - Set **Batch Size** to 10.  
   - Connect **Generate A-Z Queries** output to this node input.

4. **Google Autocomplete request:**  
   - Add an **HTTP Request** node named **Google Autocomplete**.  
   - Configure:  
     - Method: GET  
     - URL:  
       ```
       https://suggestqueries.google.com/complete/search?client=firefox&hl=en&oe=utf-8&q={{ $json.query }}
       ```  
   - Connect **Loop Over Items** output to this node input.

5. **Parse autocomplete response:**  
   - Add a **Code** node named **Code**.  
   - Paste:  
     ```javascript
     const data = JSON.parse($json.data);
     return {
       json: {
         keywords: data[1]
       }
     };
     ```  
   - Connect **Google Autocomplete** output to this node input.

6. **Add wait between batches:**  
   - Add a **Wait** node named **Wait 1s**.  
   - Set **Unit** to seconds and **Time** to 1.  
   - Connect **Code** output to this node input.

7. **Loop back to batch processing:**  
   - Connect **Wait 1s** output back to **Loop Over Items** input (to process next batch).

8. **Aggregate keywords:**  
   - Connect the **Loop Over Items** node’s second output (the aggregated output) to a **Code** node named **Extract Keywords**.  
   - Paste:  
     ```javascript
     let mergedKeywords = [];
     for (const item of $input.all()) {
       mergedKeywords.push(...item.json.keywords);
     }
     return { json: { keywords: mergedKeywords } };
     ```

9. **Return keywords to requester:**  
   - Add a **Respond to Webhook** node named **Return Keywords**.  
   - Connect **Extract Keywords** output to this node input.  
   - Use default response settings.

10. **Optional sticky notes:**  
    - Add sticky notes with the following content for documentation inside the workflow:  
      - About the workflow purpose and example keyword usage.  
      - Export options for keywords (webhook, email, file, website).  
      - How to adapt the language by modifying the `hl` parameter in the HTTP Request node.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow scrapes Google autocomplete results by combining your keyword with every letter from A to Z.           | Workflow purpose and usage explanation (Sticky Note).                                           |
| You can export the keywords via webhook, email, file (e.g., Google Drive), or directly to a website.                  | Export options for keyword results (Sticky Note).                                               |
| Autocomplete results depend on the selected language; change the `&hl=en` parameter in the Google Autocomplete node. | Language adaptation instructions (Sticky Note).                                                |
| Example output image showing autocomplete results for keyword "n8n".                                                 | Provided in original workflow description (not included here).                                  |

---

This document fully captures the workflow’s structure, logic, and configuration to enable thorough understanding, modification, and reproduction.