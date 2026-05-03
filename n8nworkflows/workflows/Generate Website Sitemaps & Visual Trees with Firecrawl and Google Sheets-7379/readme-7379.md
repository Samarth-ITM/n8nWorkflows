Generate Website Sitemaps & Visual Trees with Firecrawl and Google Sheets

https://n8nworkflows.xyz/workflows/generate-website-sitemaps---visual-trees-with-firecrawl-and-google-sheets-7379


# Generate Website Sitemaps & Visual Trees with Firecrawl and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of website sitemaps and visual hierarchical trees (site arborescences) by crawling URLs using the Firecrawl API and then organizing the data into a structured Google Sheets document. It is designed for users who want to analyze website structures, visualize the URL hierarchy, and maintain the output in a shareable spreadsheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives a chat message containing a website URL to crawl.
- **1.2 Firecrawl Website Mapping:** Uses Firecrawl to map the website and retrieve URLs via sitemap.
- **1.3 Validation and Control Flow:** Checks if Firecrawl returned successful data; routes accordingly.
- **1.4 Template Copying:** Copies a Google Sheets template to create a new document for each request.
- **1.5 URL Processing and Hierarchy Creation:** Runs a custom JavaScript code node to parse, normalize, and organize URLs into a hierarchically structured format.
- **1.6 Data Mapping to Google Sheets:** Appends the processed hierarchical data into the copied Google Sheet.
- **1.7 Response Generation:** Sends a final response with a link to the generated Google Sheets document or an error message if URL is invalid.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Starts the workflow by receiving a chat message containing the target website URL via a webhook designed for LangChain chat triggers.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **Type:** Chat Trigger (LangChain integration)  
  - **Role:** Entry point webhook that listens for incoming chat messages.  
  - **Configuration:**  
    - Mode: Webhook with public access  
    - Response Mode: Immediate node response  
  - **Expressions:** Retrieves the chat input URL from the incoming JSON payload (`$json.chatInput`).  
  - **Input:** External webhook call with chat message JSON.  
  - **Output:** Sends chat input URL to "Map a website and get urls" node.  
  - **Failure modes:**  
    - No chat message received or malformed input.  
    - Network or webhook access issues.  
  - **Version:** 1.1

#### 1.2 Firecrawl Website Mapping

- **Overview:**  
  Calls the Firecrawl API to map the website using the provided URL and fetch all sitemap URLs.

- **Nodes Involved:**  
  - Map a website and get urls

- **Node Details:**  
  - **Type:** Firecrawl node (custom n8n node for Firecrawl API)  
  - **Role:** Performs the actual crawling and sitemap extraction.  
  - **Configuration:**  
    - URL: Dynamically set from chat input (`={{ $json.chatInput }}`)  
    - Operation: "map" (to crawl and map the website)  
    - Sitemap Only: true (limits crawl to sitemap URLs)  
    - Ignore Sitemap: false (respects sitemap)  
  - **Credentials:** Firecrawl API credentials required.  
  - **Input:** Receives URL from previous node.  
  - **Output:** JSON with success status and array of URLs (links).  
  - **Failure modes:**  
    - Invalid URL or unsupported domain.  
    - API errors or quota exceeded.  
    - Network timeouts.  
  - **Version:** 1

#### 1.3 Validation and Control Flow

- **Overview:**  
  Checks if Firecrawl successfully returned data. Routes the workflow either to continue processing or respond with an error.

- **Nodes Involved:**  
  - Firecrawl OK  
  - Bad URL

- **Node Details:**  
  - **Firecrawl OK**  
    - Type: If node  
    - Role: Evaluates if `success` field from Firecrawl output is true  
    - Condition: Boolean check `{{$json.success}} === true`  
    - Output:  
      - True branch to "Copy template" node  
      - False branch to "Bad URL" node  
    - Failure modes: Expression evaluation errors or missing `success` field.  
    - Version: 2.2  

  - **Bad URL**  
    - Type: Respond to Webhook  
    - Role: Sends JSON error message if URL is invalid or not processed.  
    - Configuration: Responds with message referencing original chat input URL.  
    - Input: False branch from "Firecrawl OK" node.  
    - Output: JSON response to webhook caller.  
    - Failure modes: Response transmission failure.  
    - Version: 1.1

#### 1.4 Template Copying

- **Overview:**  
  Copies a predefined Google Sheets template to create a new spreadsheet file where data will be appended.

- **Nodes Involved:**  
  - Copy template

- **Node Details:**  
  - **Type:** Google Drive node  
  - **Role:** Copies a Google Sheets file template to create a new file named after the chat input URL appended with " - n8n - Arborescence"  
  - **Configuration:**  
    - Operation: copy  
    - File ID: Fixed ID of the template file  
    - Name: Dynamic naming using chat input from "When chat message received" node  
  - **Credentials:** Google Drive OAuth2 credentials required.  
  - **Input:** Success branch from "Firecrawl OK" node.  
  - **Output:** New file ID for the copied spreadsheet.  
  - **Failure modes:**  
    - Permission errors or expired credentials.  
    - Template file not found or access denied.  
    - Google Drive API quota limits.  
  - **Version:** 3

#### 1.5 URL Processing and Hierarchy Creation

- **Overview:**  
  Processes the URLs retrieved by Firecrawl into a hierarchical structure representing the website’s sitemap tree, ready for spreadsheet mapping.

- **Nodes Involved:**  
  - Sorting URL into table

- **Node Details:**  
  - **Type:** Code (JavaScript) node  
  - **Role:**  
    - Validates input data structure.  
    - Normalizes URLs (enforces HTTPS).  
    - Extracts domains and subdomains.  
    - Builds a tree-like representation of URLs with up to 5 levels of depth.  
    - Formats display text with URL decoding and capitalization.  
    - Returns an array of JSON objects formatted for Google Sheets columns "Niv 0" to "Niv 5".  
    - Includes error handling and returns error details if processing fails.  
  - **Key expressions:** Uses `$()` selector to access data from "Map a website and get urls" node.  
  - **Input:** Receives JSON data from "Copy template" (indirectly via node connection).  
  - **Output:** Array of hierarchical rows for Google Sheets.  
  - **Failure modes:**  
    - Input data missing or malformed.  
    - URL decoding errors.  
    - Unexpected data structure or empty links array.  
    - Runtime errors in JS code.  
  - **Version:** 2

#### 1.6 Data Mapping to Google Sheets

- **Overview:**  
  Appends the generated hierarchical sitemap data rows into the copied Google Sheet under the "FR" sheet tab.

- **Nodes Involved:**  
  - Data mapping

- **Node Details:**  
  - **Type:** Google Sheets node  
  - **Role:** Appends rows to Google Sheets document  
  - **Configuration:**  
    - Operation: append  
    - Document ID: dynamically set from "Copy template" node output  
    - Sheet Name: "FR" (fixed)  
    - Columns: "Niv 0" through "Niv 5", plus "error" and "message" for error reporting  
    - Mapping Mode: Auto-map input data fields to columns  
    - No type conversion attempted (all strings)  
  - **Credentials:** Google Sheets OAuth2 credentials required.  
  - **Input:** Rows from "Sorting URL into table" node.  
  - **Output:** Data appended confirmation to next node.  
  - **Failure modes:**  
    - Invalid document or sheet name.  
    - Permission issues or token expiration.  
    - API quota exceeded.  
  - **Version:** 4.5

#### 1.7 Response Generation

- **Overview:**  
  Sends a final JSON response with a clickable link to the generated Google Sheets document for user access.

- **Nodes Involved:**  
  - Final answer

- **Node Details:**  
  - **Type:** Respond to Webhook  
  - **Role:** Returns a JSON message including a clickable URL to the new spreadsheet document.  
  - **Configuration:**  
    - Respond with JSON  
    - Response body includes a Markdown link using the copied file ID dynamically retrieved from "Copy template" node output.  
  - **Input:** From "Data mapping" node after successful append operation.  
  - **Output:** Final webhook response to the chat message origin.  
  - **Failure modes:**  
    - Failure to format or send response.  
    - Missing or invalid file ID.  
  - **Version:** 1.1

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                              | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                   |
|----------------------------|----------------------------|----------------------------------------------|-----------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (LangChain)   | Receive chat input URL via webhook            | (webhook external)          | Map a website and get urls      |                                                                                                |
| Map a website and get urls | Firecrawl API node         | Crawl website sitemap URLs                     | When chat message received  | Firecrawl OK                   |                                                                                                |
| Firecrawl OK               | If node                   | Check success status from Firecrawl           | Map a website and get urls  | Copy template (true), Bad URL (false) |                                                                                                |
| Bad URL                   | Respond to Webhook         | Return error if URL invalid                     | Firecrawl OK (false)        | (end)                         |                                                                                                |
| Copy template              | Google Drive               | Copy Google Sheets template for output         | Firecrawl OK (true)         | Sorting URL into table         |                                                                                                |
| Sorting URL into table     | Code (JavaScript)          | Process URLs into hierarchical sitemap array  | Copy template               | Data mapping                  |                                                                                                |
| Data mapping              | Google Sheets              | Append hierarchy data rows to copied sheet     | Sorting URL into table       | Final answer                  |                                                                                                |
| Final answer              | Respond to Webhook         | Send final response with Google Sheets link    | Data mapping                | (end)                         |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Node type: Chat Trigger (LangChain)  
   - Configure mode as "Webhook" with public access.  
   - Set response mode to "responseNode" for immediate response handling.  
   - Position at workflow start.

2. **Add Firecrawl Node**  
   - Node type: Firecrawl (custom n8n node)  
   - Set operation to "map".  
   - Enable sitemapOnly = true and ignoreSitemap = false.  
   - Under URL parameter, set expression to `{{$json.chatInput}}` to use input URL.  
   - Add Firecrawl API credentials (requires Firebase API key and related account).  
   - Connect from Chat Trigger output.

3. **Add If Node for Firecrawl Success Check**  
   - Node type: If  
   - Condition: Check if `{{$json.success}}` equals true (boolean strict)  
   - Connect Firecrawl node output here.

4. **Add Respond to Webhook Node for Bad URL**  
   - Node type: Respond to Webhook  
   - Response body JSON:  
     `{"text": "L'url {{$json.chatInput}} n'est pas une url correcte ou elle n'est pas prise en compte par ce service"}`  
   - Connect from If node false branch.

5. **Add Google Drive Copy Node**  
   - Node type: Google Drive  
   - Operation: copy  
   - File ID: Set to your Google Sheets template file ID.  
   - Name: Set expression to `{{$node["When chat message received"].json.chatInput}} - n8n - Arborescence`  
   - Add Google Drive OAuth2 credentials.  
   - Connect from If node true branch.

6. **Add Code Node to Process URLs**  
   - Node type: Code (JavaScript)  
   - Paste the provided JavaScript code that:  
     - Validates Firecrawl data from node "Map a website and get urls".  
     - Normalizes URLs and builds hierarchical sitemap.  
     - Outputs array of objects with columns Niv 0 to Niv 5 and URL.  
   - Connect from Google Drive copy node.

7. **Add Google Sheets Append Node**  
   - Node type: Google Sheets  
   - Operation: append  
   - Document ID: Use expression from Copy template node output (`{{$node["Copy template"].json.id}}`)  
   - Sheet Name: "FR" (static)  
   - Columns: Define columns "Niv 0" through "Niv 5", "error", and "message" matching the code output fields.  
   - Add Google Sheets OAuth2 credentials.  
   - Connect from Code node.

8. **Add Respond to Webhook Node for Final Answer**  
   - Node type: Respond to Webhook  
   - Response body JSON:  
     `{"text": "Cliquez [ici](https://docs.google.com/spreadsheets/d/{{$node[\"Copy template\"].json.id}}) afin d'accéder à votre arborescence"}`  
   - Connect from Google Sheets append node.

9. **Validate Credentials and Permissions**  
   - Ensure Google Drive and Google Sheets OAuth2 credentials have permissions to copy and edit files.  
   - Ensure Firecrawl API credentials are valid and have quota.  
   - Test webhook availability and public access.

10. **Save and Activate Workflow**  
    - Test with valid URL chat messages.  
    - Verify spreadsheet creation and data population.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses Firecrawl API for website crawling and sitemap extraction.                      | Firecrawl API documentation (register for API keys)                                            |
| Google Sheets template must have predefined columns: Niv 0 to Niv 5, error, and message columns. | Template file ID is hardcoded and copied per request to avoid overwriting data.                 |
| The JavaScript code node normalizes http to https URLs and correctly handles www subdomains.    | Important for URL consistency and hierarchical grouping.                                        |
| The final webhook response provides a clickable link to the generated Google Sheets document.   | Users can directly access their sitemap visualization through this link.                        |
| Workflow designed for French-speaking users (column names and messages in French).               | Adapt column names/messages if used in other languages.                                         |
| Ensure Google Drive and Sheets OAuth2 credentials do not expire to maintain workflow reliability. | OAuth2 tokens may require periodic refresh or reauthorization.                                  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.