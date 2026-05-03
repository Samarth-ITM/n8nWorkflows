Extract, Transform LinkedIn Data with Bright Data MCP Server & Google Gemini

https://n8nworkflows.xyz/workflows/extract--transform-linkedin-data-with-bright-data-mcp-server---google-gemini-3777


# Extract, Transform LinkedIn Data with Bright Data MCP Server & Google Gemini

### 1. Workflow Overview

This workflow automates the extraction and transformation of LinkedIn data using Bright Data MCP Server tools combined with Google Gemini LLM for data enrichment. It targets professionals who need structured LinkedIn datasets, such as data analysts, marketing/sales teams, recruiters, AI developers, and business intelligence teams.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Initialization**: Manual start and setting input URLs for LinkedIn person and company profiles.
- **1.2 Bright Data MCP Server Scraping**: Uses MCP Client nodes to scrape LinkedIn person and company profiles via Bright Data tools.
- **1.3 Data Processing & Transformation**: Parses raw scraped data, enriches and formats it using Google Gemini LLM and LangChain extractor.
- **1.4 Data Persistence & Notification**: Saves the extracted data to disk files and sends webhook notifications with the final structured data.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

**Overview:**  
Starts the workflow manually and sets the LinkedIn URLs and webhook endpoint for person and company data extraction.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- List all available tools for Bright Data (MCP Client)  
- List all tools for Bright Data (MCP Client)  
- Set the URLs (Set)  
- Set the LinkedIn Company URL (Set)  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters.  
  - Connections: Triggers two MCP Client nodes listing tools.  
  - Edge Cases: None typical; user must manually trigger.

- **List all available tools for Bright Data**  
  - Type: MCP Client  
  - Role: Lists all MCP tools available; used here likely for validation or debugging.  
  - Credentials: MCP Client (STDIO) with Bright Data API token.  
  - Connections: Outputs to "Set the URLs" node.  
  - Edge Cases: API errors, authentication failures.

- **List all tools for Bright Data**  
  - Type: MCP Client  
  - Role: Similar to above, lists MCP tools.  
  - Credentials: Same MCP Client credentials.  
  - Connections: Outputs to "Set the LinkedIn Company URL" node.  
  - Edge Cases: Same as above.

- **Set the URLs**  
  - Type: Set  
  - Role: Defines static LinkedIn person URL and webhook URL for person data.  
  - Configuration:  
    - url: "https://www.linkedin.com/in/ranjan-dailata/"  
    - webhook_url: webhook.site endpoint for person data.  
  - Connections: Outputs to "Bright Data MCP Client For LinkedIn Person".  
  - Edge Cases: Hardcoded URLs require manual update for other profiles.

- **Set the LinkedIn Company URL**  
  - Type: Set  
  - Role: Defines static LinkedIn company URL and webhook URL for company data.  
  - Configuration:  
    - url: "https://www.linkedin.com/company/bright-data/"  
    - webhook_url: webhook.site endpoint for company data.  
  - Connections: Outputs to "Bright Data MCP Client For LinkedIn Company".  
  - Edge Cases: Same as above.

---

#### 1.2 Bright Data MCP Server Scraping

**Overview:**  
Scrapes LinkedIn person and company profiles using Bright Data MCP Server LinkedIn tools.

**Nodes Involved:**  
- Bright Data MCP Client For LinkedIn Person (MCP Client)  
- Bright Data MCP Client For LinkedIn Company (MCP Client)  

**Node Details:**

- **Bright Data MCP Client For LinkedIn Person**  
  - Type: MCP Client  
  - Role: Executes Bright Data MCP tool `web_data_linkedin_person_profile` to scrape person profile.  
  - Configuration:  
    - toolName: "web_data_linkedin_person_profile"  
    - toolParameters: JSON with URL from "Set the URLs" node.  
  - Credentials: MCP Client (STDIO) with Bright Data API token.  
  - Connections: Outputs to "Webhook for LinkedIn Person Web Scraper" and "Create a binary data for LinkedIn person info extract".  
  - Edge Cases: Network failures, invalid URL, MCP tool errors, rate limiting.

- **Bright Data MCP Client For LinkedIn Company**  
  - Type: MCP Client  
  - Role: Executes Bright Data MCP tool `web_data_linkedin_company_profile` to scrape company profile.  
  - Configuration:  
    - toolName: "web_data_linkedin_company_profile"  
    - toolParameters: JSON with URL from "Set the LinkedIn Company URL" node.  
  - Credentials: MCP Client (STDIO) with Bright Data API token.  
  - Connections: Outputs to "Code" node and "LinkedIn Data Extractor".  
  - Edge Cases: Same as above.

---

#### 1.3 Data Processing & Transformation

**Overview:**  
Parses raw scraped data, extracts structured information, and enriches it using Google Gemini LLM and LangChain extractor.

**Nodes Involved:**  
- Code (Function)  
- LinkedIn Data Extractor (LangChain Information Extractor)  
- Google Gemini Chat Model (LLM)  
- Merge (Merge)  
- Aggregate (Aggregate)  

**Node Details:**

- **Code**  
  - Type: Function  
  - Role: Parses the raw MCP Client JSON response content string into JSON object.  
  - Configuration: Parses `$input.first().json.result.content[0].text` as JSON.  
  - Connections: Outputs to "Merge" node.  
  - Edge Cases: JSON parse errors if MCP response malformed.

- **LinkedIn Data Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Uses a prompt to generate a detailed company story from the scraped company JSON info.  
  - Configuration:  
    - Text prompt instructs to write a complete story from company info JSON.  
    - System prompt: "You are an expert data formatter".  
    - Extracts attribute "company_story".  
  - Connections: Outputs to "Merge" node.  
  - Edge Cases: LLM API errors, prompt failures, incomplete input data.

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model  
  - Role: Provides LLM processing for the LinkedIn Data Extractor node.  
  - Configuration:  
    - Model: "models/gemini-2.0-flash-exp"  
  - Credentials: Google Gemini (PaLM) API account.  
  - Connections: Outputs to "LinkedIn Data Extractor".  
  - Edge Cases: API quota limits, auth failures, latency.

- **Merge**  
  - Type: Merge  
  - Role: Combines parsed company data and extracted company story into one dataset.  
  - Connections: Outputs to "Aggregate".  
  - Edge Cases: Data mismatch or missing inputs.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates fields "about" and "output.company_story" into a single object for output.  
  - Connections: Outputs to "Webhook for LinkedIn Company Web Scraper" and "Create a binary data for LinkedIn company info extract".  
  - Edge Cases: Missing fields, aggregation errors.

---

#### 1.4 Data Persistence & Notification

**Overview:**  
Saves extracted LinkedIn person and company data to disk files and sends webhook notifications with the structured data.

**Nodes Involved:**  
- Webhook for LinkedIn Person Web Scraper (HTTP Request)  
- Create a binary data for LinkedIn person info extract (Function)  
- Write the LinkedIn person info to disk (Read/Write File)  
- Webhook for LinkedIn Company Web Scraper (HTTP Request)  
- Create a binary data for LinkedIn company info extract (Function)  
- Write the LinkedIn company info to disk (Read/Write File)  

**Node Details:**

- **Webhook for LinkedIn Person Web Scraper**  
  - Type: HTTP Request  
  - Role: Sends scraped LinkedIn person profile data to specified webhook URL.  
  - Configuration:  
    - URL from "Set the URLs" webhook_url field.  
    - Sends JSON body with "response" containing raw scraped text.  
  - Connections: Terminal node for person data branch.  
  - Edge Cases: Webhook endpoint failures, network errors.

- **Create a binary data for LinkedIn person info extract**  
  - Type: Function  
  - Role: Converts JSON person data into base64-encoded binary format for file writing.  
  - Configuration: Uses Buffer to encode JSON string.  
  - Connections: Outputs to "Write the LinkedIn person info to disk".  
  - Edge Cases: Encoding errors.

- **Write the LinkedIn person info to disk**  
  - Type: Read/Write File  
  - Role: Writes person data binary to file "d:\LinkedIn-Person.json".  
  - Configuration: Write operation, file path hardcoded.  
  - Edge Cases: File system permissions, path errors.

- **Webhook for LinkedIn Company Web Scraper**  
  - Type: HTTP Request  
  - Role: Sends aggregated and transformed company data to specified webhook URL.  
  - Configuration:  
    - URL from "Set the LinkedIn Company URL" webhook_url field.  
    - Sends JSON body with "about" and "story" fields.  
  - Connections: Terminal node for company data branch.  
  - Edge Cases: Same as person webhook.

- **Create a binary data for LinkedIn company info extract**  
  - Type: Function  
  - Role: Converts JSON company data into base64-encoded binary format for file writing.  
  - Configuration: Similar to person binary creator.  
  - Connections: Outputs to "Write the LinkedIn company info to disk".  
  - Edge Cases: Same as person binary creator.

- **Write the LinkedIn company info to disk**  
  - Type: Read/Write File  
  - Role: Writes company data binary to file "d:\LinkedIn-Company.json".  
  - Configuration: Write operation, file path hardcoded.  
  - Edge Cases: Same as person file writer.

---

### 3. Summary Table

| Node Name                              | Node Type                          | Functional Role                                | Input Node(s)                          | Output Node(s)                                         | Sticky Note                                                                                  |
|--------------------------------------|----------------------------------|-----------------------------------------------|--------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’         | Manual Trigger                   | Manual start trigger                           | None                                 | List all available tools for Bright Data, List all tools for Bright Data |                                                                                              |
| List all available tools for Bright Data | MCP Client                      | Lists MCP tools (for validation/debug)        | When clicking ‘Test workflow’         | Set the URLs                                          |                                                                                              |
| List all tools for Bright Data        | MCP Client                      | Lists MCP tools (for validation/debug)        | When clicking ‘Test workflow’         | Set the LinkedIn Company URL                          |                                                                                              |
| Set the URLs                         | Set                             | Sets LinkedIn person URL and webhook URL      | List all available tools for Bright Data | Bright Data MCP Client For LinkedIn Person             |                                                                                              |
| Bright Data MCP Client For LinkedIn Person | MCP Client                      | Scrapes LinkedIn person profile                | Set the URLs                         | Webhook for LinkedIn Person Web Scraper, Create a binary data for LinkedIn person info extract | ## Bright Data LinkedIn Person Scraper                                                       |
| Webhook for LinkedIn Person Web Scraper | HTTP Request                   | Sends person data to webhook endpoint          | Bright Data MCP Client For LinkedIn Person | None                                                |                                                                                              |
| Create a binary data for LinkedIn person info extract | Function                       | Converts person JSON to base64 binary          | Bright Data MCP Client For LinkedIn Person | Write the LinkedIn person info to disk                 |                                                                                              |
| Write the LinkedIn person info to disk | Read/Write File                | Saves person data to disk file                  | Create a binary data for LinkedIn person info extract | None                                                |                                                                                              |
| Set the LinkedIn Company URL         | Set                             | Sets LinkedIn company URL and webhook URL     | List all tools for Bright Data       | Bright Data MCP Client For LinkedIn Company             | ## Bright Data LinkedIn Company Scraper                                                     |
| Bright Data MCP Client For LinkedIn Company | MCP Client                      | Scrapes LinkedIn company profile               | Set the LinkedIn Company URL          | Code, LinkedIn Data Extractor                          | ## Bright Data LinkedIn Company Scraper                                                     |
| Code                                | Function                       | Parses MCP Client raw JSON response             | Bright Data MCP Client For LinkedIn Company | Merge                                                |                                                                                              |
| LinkedIn Data Extractor              | LangChain Information Extractor | Extracts and formats company story from JSON   | Google Gemini Chat Model              | Merge                                                |                                                                                              |
| Google Gemini Chat Model             | LangChain Google Gemini LLM     | Provides LLM processing for data extraction    | None (triggered by LinkedIn Data Extractor) | LinkedIn Data Extractor                               |                                                                                              |
| Merge                               | Merge                          | Combines parsed company data and extracted story | Code, LinkedIn Data Extractor         | Aggregate                                            |                                                                                              |
| Aggregate                          | Aggregate                     | Aggregates fields into final company data object | Merge                              | Webhook for LinkedIn Company Web Scraper, Create a binary data for LinkedIn company info extract |                                                                                              |
| Webhook for LinkedIn Company Web Scraper | HTTP Request                   | Sends company data to webhook endpoint          | Aggregate                           | None                                                |                                                                                              |
| Create a binary data for LinkedIn company info extract | Function                       | Converts company JSON to base64 binary          | Aggregate                           | Write the LinkedIn company info to disk                 |                                                                                              |
| Write the LinkedIn company info to disk | Read/Write File                | Saves company data to disk file                  | Create a binary data for LinkedIn company info extract | None                                                |                                                                                              |
| Sticky Note1                        | Sticky Note                   | Label for Bright Data LinkedIn Person Scraper   | None                                 | None                                                | ## Bright Data LinkedIn Person Scraper                                                       |
| Sticky Note                        | Sticky Note                   | Label for Bright Data LinkedIn Company Scraper  | None                                 | None                                                | ## Bright Data LinkedIn Company Scraper                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create MCP Client Node to List Available Tools (Person Branch)**  
   - Type: MCP Client  
   - Credentials: Configure MCP Client (STDIO) with Bright Data API token.  
   - No parameters needed.  
   - Connect Manual Trigger → this node.

3. **Create MCP Client Node to List Available Tools (Company Branch)**  
   - Same as above, separate node.  
   - Connect Manual Trigger → this node.

4. **Create Set Node for Person URLs**  
   - Type: Set  
   - Assign:  
     - `url` = "https://www.linkedin.com/in/ranjan-dailata/"  
     - `webhook_url` = your webhook URL (e.g., https://webhook.site/...)  
   - Connect output of "List all available tools for Bright Data" (person branch) → this node.

5. **Create Set Node for Company URLs**  
   - Type: Set  
   - Assign:  
     - `url` = "https://www.linkedin.com/company/bright-data/"  
     - `webhook_url` = your webhook URL for company data  
   - Connect output of "List all tools for Bright Data" (company branch) → this node.

6. **Create MCP Client Node for LinkedIn Person Scraper**  
   - Type: MCP Client  
   - Credentials: MCP Client (STDIO) with Bright Data API token.  
   - Parameters:  
     - toolName: "web_data_linkedin_person_profile"  
     - operation: "executeTool"  
     - toolParameters: JSON with `"url": "{{ $json.url }}"`  
   - Connect output of "Set the URLs" → this node.

7. **Create MCP Client Node for LinkedIn Company Scraper**  
   - Type: MCP Client  
   - Credentials: Same MCP Client credentials.  
   - Parameters:  
     - toolName: "web_data_linkedin_company_profile"  
     - operation: "executeTool"  
     - toolParameters: JSON with `"url": "{{ $json.url }}"`  
   - Connect output of "Set the LinkedIn Company URL" → this node.

8. **Create HTTP Request Node for Person Webhook**  
   - Type: HTTP Request  
   - URL: `={{ $('Set the URLs').item.json.webhook_url }}`  
   - Method: POST  
   - Body: JSON with parameter `response` set to scraped text: `={{ $json.result.content[0].text }}`  
   - Connect output of MCP Client for Person Scraper → this node.

9. **Create Function Node to Convert Person JSON to Binary**  
   - Type: Function  
   - Code:  
     ```js
     items[0].binary = {
       data: {
         data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
       }
     };
     return items;
     ```  
   - Connect output of MCP Client for Person Scraper → this node.

10. **Create Read/Write File Node to Save Person Data**  
    - Type: Read/Write File  
    - Operation: Write  
    - File Name: `d:\LinkedIn-Person.json` (adjust path as needed)  
    - Connect output of previous Function node → this node.

11. **Create Function Node to Parse Company MCP Response**  
    - Type: Function  
    - Code:  
      ```js
      jsonContent = JSON.parse($input.first().json.result.content[0].text);
      return jsonContent;
      ```  
    - Connect output of MCP Client for Company Scraper → this node.

12. **Create Google Gemini Chat Model Node**  
    - Type: LangChain Google Gemini Chat Model  
    - Credentials: Google Gemini (PaLM) API key configured.  
    - Model Name: "models/gemini-2.0-flash-exp"  
    - Connect output to LinkedIn Data Extractor node.

13. **Create LinkedIn Data Extractor Node**  
    - Type: LangChain Information Extractor  
    - Parameters:  
      - Text: Prompt instructing to write a company story from JSON input:  
        ```
        Write a complete story of the provided company information in JSON. Use the following Company info to produce a story or a blog post. Make sure to incorporate all the provided company context.

        Here's the Company Info in JSON - {{ $json.input }}
        ```  
      - System Prompt: "You are an expert data formatter"  
      - Attributes: Extract "company_story" (required)  
    - Connect Google Gemini Chat Model → this node.  
    - Connect output of Function node parsing company MCP response → this node as input JSON.

14. **Create Merge Node**  
    - Type: Merge  
    - Purpose: Combine parsed company JSON and extracted company story.  
    - Connect outputs of Function node (parsed JSON) and LinkedIn Data Extractor → this node.

15. **Create Aggregate Node**  
    - Type: Aggregate  
    - Fields to aggregate: "about" and "output.company_story"  
    - Connect output of Merge → this node.

16. **Create HTTP Request Node for Company Webhook**  
    - Type: HTTP Request  
    - URL: `={{ $('Set the LinkedIn Company URL').item.json.webhook_url }}`  
    - Method: POST  
    - Body: JSON with fields:  
      - "about": `{{ JSON.stringify($json.about[0]) }}`  
      - "story": `{{ JSON.stringify($json.company_story[0]) }}`  
    - Connect output of Aggregate → this node.

17. **Create Function Node to Convert Company JSON to Binary**  
    - Type: Function  
    - Code:  
      ```js
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```  
    - Connect output of Aggregate → this node.

18. **Create Read/Write File Node to Save Company Data**  
    - Type: Read/Write File  
    - Operation: Write  
    - File Name: `d:\LinkedIn-Company.json` (adjust path as needed)  
    - Connect output of previous Function node → this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow requires n8n self-hosted due to use of community MCP Client node.                                                                                                                                                        | Workflow description                                                                                     |
| Setup instructions and tutorial video for MCP Servers in n8n: [n8n-nodes-mcp Setup Video](https://www.youtube.com/watch?v=NUb73ErUCsA)                                                                                                | Setup section                                                                                           |
| Bright Data MCP Server Node package: [@brightdata/mcp on npm](https://www.npmjs.com/package/@brightdata/mcp)                                                                                                                          | Setup section                                                                                           |
| Bright Data signup and Web Unlocker proxy setup instructions: [Bright Data](https://brightdata.com/)                                                                                                                                  | Setup section                                                                                           |
| Google Gemini (PaLM) API key or Vertex AI access required for LLM nodes.                                                                                                                                                              | Setup section                                                                                           |
| Customize input URLs dynamically by replacing Set nodes with webhook or form input nodes.                                                                                                                                             | Customization suggestions                                                                               |
| Modify LinkedIn Data Extractor prompt to tailor data extraction and formatting.                                                                                                                                                        | Customization suggestions                                                                               |
| Webhook endpoints can be changed to integrate with Slack, Airtable, Notion, CRM, etc.                                                                                                                                                  | Customization suggestions                                                                               |
| Hardcoded file paths for saving JSON data require write permissions and may be adjusted for different OS or storage locations.                                                                                                       | General caution                                                                                         |

---

This document provides a detailed, structured reference for the LinkedIn data extraction workflow using Bright Data MCP Server and Google Gemini LLM, enabling advanced users and automation agents to understand, reproduce, and customize the workflow effectively.