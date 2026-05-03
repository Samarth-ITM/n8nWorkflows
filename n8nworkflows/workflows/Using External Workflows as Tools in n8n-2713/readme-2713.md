Using External Workflows as Tools in n8n

https://n8nworkflows.xyz/workflows/using-external-workflows-as-tools-in-n8n-2713


# Using External Workflows as Tools in n8n

### 1. Workflow Overview

This workflow, titled **"get_a_web_page"**, demonstrates how to create a reusable external workflow in n8n that fetches and processes the content of a web page. It is designed as a tool that can be invoked by other workflows or AI agents to scrape web content in a consistent, structured manner. The workflow is particularly useful for integrating web scraping capabilities into larger automation projects without duplicating scraping logic.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the URL input via the "Execute Workflow Trigger" node, allowing external workflows or agents to trigger this workflow with a URL to scrape.
- **1.2 Web Scraping Request:** Uses the "FireCrawl" HTTP Request node to send the URL to an external scraping API and retrieve the page content in markdown format.
- **1.3 Output Formatting:** Uses the "Edit Fields" node to standardize the output format, ensuring the response contains a predictable JSON structure with the scraped markdown content.
- **1.4 Documentation and Usage Notes:** A sticky note node provides usage instructions and example input format for users or AI agents.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving external input. It expects a JSON object containing a URL to scrape, enabling the workflow to be triggered programmatically with dynamic URLs.

- **Nodes Involved:**  
  - Execute Workflow Trigger

- **Node Details:**  
  - **Execute Workflow Trigger**  
    - *Type:* `executeWorkflowTrigger`  
    - *Role:* Entry point for external workflows or AI agents to trigger this workflow with input data.  
    - *Configuration:* No special parameters; it simply accepts incoming JSON input.  
    - *Key Expressions:* Input JSON expected in the form:  
      ```json
      [
        {
          "query": {
            "url": "https://en.wikipedia.org/wiki/some_info"
          }
        }
      ]
      ```  
    - *Input Connections:* None (trigger node)  
    - *Output Connections:* Connects to "FireCrawl" node  
    - *Version Requirements:* Compatible with n8n version supporting `executeWorkflowTrigger` node (v1 or later)  
    - *Potential Failures:* Missing or malformed input JSON; no URL provided; invalid URL format.

#### 2.2 Web Scraping Request

- **Overview:**  
  This block sends the provided URL to the FireCrawl API, which scrapes the web page and returns its content in markdown format.

- **Nodes Involved:**  
  - FireCrawl (HTTP Request)

- **Node Details:**  
  - **FireCrawl**  
    - *Type:* `httpRequest`  
    - *Role:* Sends a POST request to the FireCrawl API to scrape the specified URL.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://api.firecrawl.dev/v1/scrape`  
      - Body Type: JSON  
      - Body Content:  
        ```json
        {
          "url": "{{ $json.query.url }}",
          "formats": ["markdown"]
        }
        ```  
      - Authentication: HTTP Header Auth with Firecrawl API credentials  
      - Headers: Set via credentials (no manual headers specified)  
    - *Key Expressions:* Uses expression to dynamically insert the URL from the trigger node input (`{{ $json.query.url }}`)  
    - *Input Connections:* From "Execute Workflow Trigger"  
    - *Output Connections:* To "Edit Fields" node  
    - *Version Requirements:* HTTP Request node version 4.2 or later for advanced JSON body handling  
    - *Potential Failures:*  
      - Authentication errors (invalid or expired API key)  
      - Network timeouts or connectivity issues  
      - API rate limiting or quota exceeded  
      - Invalid URL causing API to fail scraping  
      - Unexpected API response format changes

#### 2.3 Output Formatting

- **Overview:**  
  This block standardizes the output by extracting the markdown content from the API response and assigning it to a consistent field named `response`. This ensures that any workflow or agent consuming this tool receives a predictable output format.

- **Nodes Involved:**  
  - Edit Fields (Set)

- **Node Details:**  
  - **Edit Fields**  
    - *Type:* `set`  
    - *Role:* Transforms the raw API response to a simplified JSON output containing only the markdown content.  
    - *Configuration:*  
      - Assigns a new field `response` with the value extracted from `data.markdown` in the previous node's JSON output:  
        ```javascript
        {{$json.data.markdown}}
        ```  
    - *Input Connections:* From "FireCrawl"  
    - *Output Connections:* None (end of workflow)  
    - *Version Requirements:* Set node version 3.4 or later for expression support  
    - *Potential Failures:*  
      - If the API response does not contain `data.markdown`, the expression will fail or return undefined.  
      - Malformed JSON input from previous node.

#### 2.4 Documentation and Usage Notes

- **Overview:**  
  Provides inline documentation within the workflow canvas to guide users on how to use this workflow as a reusable tool.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - *Type:* `stickyNote`  
    - *Role:* Contains usage instructions and example input JSON format for invoking this workflow.  
    - *Content Highlights:*  
      - Explains that the workflow can be reused by AI agents or other workspaces by sending a JSON request with a URL field.  
      - Provides example input JSON:  
        ```json
        {
          "url": "Some URL to Get"
        }
        ```  
    - *Input/Output Connections:* None (informational only)  
    - *Version Requirements:* None  
    - *Potential Failures:* None

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                  | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                   |
|-------------------------|----------------------------|--------------------------------|--------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Execute Workflow Trigger | executeWorkflowTrigger     | Entry point; receives URL input | None                     | FireCrawl               | This can be reused by AI Agents and any Workspace to crawl a site. Input JSON example provided. |
| FireCrawl               | httpRequest                | Sends URL to FireCrawl API to scrape page | Execute Workflow Trigger | Edit Fields             |                                                                                              |
| Edit Fields             | set                        | Formats output to return markdown content | FireCrawl                | None                    |                                                                                              |
| Sticky Note             | stickyNote                 | Provides usage instructions and example input | None                     | None                    | ## Send URL got Crawl This can be reused by Ai Agents and any Workspace to crawl a site. All that Workspace has to do is send a request: ```json { "url": "Some URL to Get" } ``` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it `get_a_web_page`.

2. **Add the "Execute Workflow Trigger" node:**  
   - Type: `Execute Workflow Trigger`  
   - No special parameters needed.  
   - This node will serve as the entry point for external workflows or AI agents to trigger this workflow with input data.

3. **Add the "FireCrawl" HTTP Request node:**  
   - Type: `HTTP Request`  
   - Connect the output of "Execute Workflow Trigger" to this node's input.  
   - Configure as follows:  
     - HTTP Method: POST  
     - URL: `https://api.firecrawl.dev/v1/scrape`  
     - Authentication: HTTP Header Auth (create or select credentials for Firecrawl API key)  
     - Body Content Type: JSON  
     - Body Parameters:  
       ```json
       {
         "url": "{{ $json.query.url }}",
         "formats": ["markdown"]
       }
       ```  
     - Ensure "Send Body" and "Send Headers" options are enabled.

4. **Add the "Edit Fields" node:**  
   - Type: `Set`  
   - Connect the output of "FireCrawl" to this node's input.  
   - Configure to assign a new field named `response` with the value:  
     ```javascript
     {{$json.data.markdown}}
     ```  
   - This node ensures the output is a simple JSON object with a `response` field containing the scraped markdown content.

5. **Add a "Sticky Note" node (optional but recommended):**  
   - Type: `Sticky Note`  
   - Place it near the "Execute Workflow Trigger" node.  
   - Add the following content to guide users:  
     ```
     ## Send URL got Crawl
     This can be reused by AI Agents and any Workspace to crawl a site. All that Workspace has to do is send a request:

     ```json
     {
       "url": "Some URL to Get"
     }
     ```
     ```

6. **Set the workflow to inactive or active as needed.**

7. **Test the workflow:**  
   - Use the "Execute Workflow Trigger" node's test feature or trigger externally with JSON input:  
     ```json
     [
       {
         "query": {
           "url": "https://en.wikipedia.org/wiki/Linux"
         }
       }
     ]
     ```  
   - Confirm that the output from the "Edit Fields" node contains the markdown content of the requested page.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow exemplifies how to create reusable tools in n8n by using the "Execute Workflow Trigger" node.   | n8n official documentation on reusable workflows and triggers                                     |
| The FireCrawl API is used here to scrape web pages and return markdown content.                               | FireCrawl API documentation: https://firecrawl.dev/docs                                         |
| Consistent output formatting via the "Set" node is critical for predictable integration with AI agents.      | n8n expressions documentation: https://docs.n8n.io/nodes/expressions/                             |
| Example input JSON schema for AI Agent integration: `{ "url": "URL_TO_GET" }`                                 | Shown in workflow sticky note and description                                                    |
| Using external workflows as tools improves modularity, scalability, and maintainability of complex automations.| Best practice recommended by n8n community and automation architects                              |

---

This documentation provides a complete, detailed reference to understand, reproduce, and extend the "get_a_web_page" workflow as a reusable scraping tool in n8n.