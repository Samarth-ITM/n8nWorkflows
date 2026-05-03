Automated Competitor Pricing Monitor with Bright Data MCP & OpenAI

https://n8nworkflows.xyz/workflows/automated-competitor-pricing-monitor-with-bright-data-mcp---openai-5948


# Automated Competitor Pricing Monitor with Bright Data MCP & OpenAI

---
### 1. Workflow Overview

This workflow automates the monitoring of competitor pricing updates on ClickUp’s pricing page. It uses scheduled triggers, AI-powered scraping through Bright Data’s MCP proxy, and Google Sheets to track and update pricing data only when changes occur. The workflow is designed for competitive pricing analysis, enabling businesses to keep up-to-date pricing intelligence for strategic decisions.

Logical blocks included:

- **1.1 Setup & Historical Data Fetch**: Scheduled trigger initiates the workflow; static parameters set the target URL; existing pricing data is fetched from Google Sheets.
- **1.2 AI Agent Scrapes Pricing**: Uses Bright Data MCP proxy and OpenAI models to scrape and parse the competitor’s pricing page into structured pricing plans.
- **1.3 Compare, Decide & Update**: Compares newly scraped prices to historical data, decides if updates are needed, and updates the Google Sheet accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Historical Data Fetch

- **Overview:**  
  This block triggers the workflow on a schedule, sets the target URL for scraping, and retrieves the last known pricing from Google Sheets to serve as a baseline for comparison.

- **Nodes Involved:**  
  - ⏰ Trigger: Check Job Listings  
  - 🛠️ Set Search Parameters  
  - Retrieve Pricing data

- **Node Details:**

  1. **⏰ Trigger: Check Job Listings**  
     - *Type:* Schedule Trigger  
     - *Role:* Initiates the workflow daily at 09:00 AM.  
     - *Configuration:* Schedule set to trigger once daily at hour 9.  
     - *Inputs:* None (trigger node).  
     - *Outputs:* Starts chain to "Set Search Parameters".  
     - *Edge Cases:* Missed trigger if n8n is offline; time zone considerations may apply.

  2. **🛠️ Set Search Parameters**  
     - *Type:* Set Node  
     - *Role:* Defines static input parameters for the workflow, notably the URL to scrape.  
     - *Configuration:* Sets `url` variable to "https://clickup.com/pricing".  
     - *Inputs:* Receives trigger output.  
     - *Outputs:* Passes parameters to "Retrieve Pricing data".  
     - *Edge Cases:* Hardcoded URL limits flexibility unless changed manually.

  3. **Retrieve Pricing data**  
     - *Type:* Google Sheets (Read)  
     - *Role:* Reads existing pricing data from a Google Sheet to compare with fresh data.  
     - *Configuration:*  
       - Document ID: Google Sheet tracking ClickUp pricing.  
       - Sheet name: "Sheet1" (gid=0).  
     - *Inputs:* Receives search parameters input.  
     - *Outputs:* Passes retrieved data to "AI agent".  
     - *Credentials:* Google Sheets OAuth2 account required.  
     - *Edge Cases:* Authentication failures; spreadsheet access issues; empty or malformed data.

---

#### 1.2 AI Agent Scrapes Pricing

- **Overview:**  
  This block orchestrates the AI-driven scraping and processing of the competitor’s pricing page. It uses the Bright Data MCP proxy to fetch the page content, then applies OpenAI large language models (LLMs) to interpret and extract structured pricing information.

- **Nodes Involved:**  
  - AI agent (Langchain Agent Node)  
  - MCP Client to Scrape as markdown  
  - 🧠 OpenAI: LLM Brain  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Auto-fixing Output Parser

- **Node Details:**

  1. **AI agent**  
     - *Type:* Langchain Agent Node  
     - *Role:* Central AI orchestrator managing scraping and data extraction.  
     - *Configuration:*  
       - Text prompt uses the URL from "Set Search Parameters" to instruct scraping for plan names and pricing.  
       - Uses MCP proxy and OpenAI LLM to perform scraping and parsing.  
       - Has output parser enabled for structured JSON output.  
     - *Inputs:* Receives historical pricing data from "Retrieve Pricing data".  
     - *Outputs:* Sends data to "If price changes" node.  
     - *Edge Cases:* API rate limits; malformed input URL; failure in scraping or parsing; proxy errors.

  2. **MCP Client to Scrape as markdown**  
     - *Type:* Bright Data MCP Client Tool Node  
     - *Role:* Fetches competitor webpage content through Bright Data’s MCP proxy as markdown.  
     - *Configuration:*  
       - Tool: scrape_as_markdown  
       - Uses manual description explaining advanced content extraction.  
     - *Inputs:* Invoked by AI agent as a tool.  
     - *Outputs:* Supplies page markdown content to AI agent for interpretation.  
     - *Credentials:* Bright Data MCP API credentials required.  
     - *Edge Cases:* Proxy connection failures; scraping blocked; malformed or incomplete markdown output.

  3. **🧠 OpenAI: LLM Brain**  
     - *Type:* Langchain OpenAI Chat Model Node  
     - *Role:* Runs GPT-4o-mini model to interpret scraped markdown content.  
     - *Configuration:* Uses GPT-4o-mini model with default options.  
     - *Inputs:* Receives markdown content from MCP Client.  
     - *Outputs:* Feeds interpreted text to AI agent.  
     - *Credentials:* OpenAI API key required.  
     - *Edge Cases:* API failures; token limits exceeded; unexpected content structure.

  4. **OpenAI Chat Model**  
     - *Type:* Langchain OpenAI Chat Model Node  
     - *Role:* Secondary chat model node used in chained parsing steps.  
     - *Configuration:* GPT-4o-mini model.  
     - *Inputs:* Used internally in parsing chain.  
     - *Outputs:* Feeds into Auto-fixing Output Parser.  
     - *Credentials:* OpenAI API key required.  
     - *Edge Cases:* Same as LLM Brain node.

  5. **Structured Output Parser**  
     - *Type:* Langchain Structured Output Parser  
     - *Role:* Parses AI output into structured JSON format with pricing plans.  
     - *Configuration:* Uses a detailed JSON schema example defining plans, pricing, and key features.  
     - *Inputs:* Accepts raw AI text output.  
     - *Outputs:* Passes structured JSON to Auto-fixing Output Parser.  
     - *Edge Cases:* Parsing errors if AI output deviates from schema.

  6. **Auto-fixing Output Parser**  
     - *Type:* Langchain Output Parser with autofixing  
     - *Role:* Attempts to correct and finalize the structured JSON output for consistency.  
     - *Inputs:* Receives data from Structured Output Parser and OpenAI Chat Model.  
     - *Outputs:* Feeds cleaned structured data back to AI agent node.  
     - *Edge Cases:* Failures in auto-correction; infinite loops if output is malformed.

---

#### 1.3 Compare, Decide & Update

- **Overview:**  
  This block compares the freshly scraped pricing data with the historical records, determines if updates are necessary, and either updates the Google Sheet or does nothing.

- **Nodes Involved:**  
  - If price changes  
  - No Operation, do nothing  
  - Update google sheet

- **Node Details:**

  1. **If price changes**  
     - *Type:* If Node  
     - *Role:* Compares the scraped prices for four plans against the saved prices in the sheet.  
     - *Configuration:*  
       - Four string equality conditions comparing each plan’s price from "Retrieve Pricing data" and from "AI agent" output.  
       - Only if all four prices match, the condition is true (meaning no change).  
     - *Inputs:* Receives AI agent output and historical data.  
     - *Outputs:*  
       - TRUE branch: No Operation, do nothing (prices match)  
       - FALSE branch: Update google sheet (prices differ)  
     - *Edge Cases:* Inconsistent data formats; partial data missing; string comparison may fail if formatting differs.

  2. **No Operation, do nothing**  
     - *Type:* NoOp Node  
     - *Role:* Placeholder to confirm no action when prices are unchanged.  
     - *Inputs:* TRUE branch from IF node.  
     - *Outputs:* Terminal node; no further action.  
     - *Edge Cases:* None.

  3. **Update google sheet**  
     - *Type:* Google Sheets (Write)  
     - *Role:* Updates the pricing data in the Google Sheet with new values.  
     - *Configuration:*  
       - Document and sheet same as "Retrieve Pricing data".  
       - Updates row 2 with plan names, prices, and key features for four plans from AI output.  
       - Uses matching on `row_number` column for update targeting.  
     - *Credentials:* Google Sheets OAuth2 account required.  
     - *Inputs:* FALSE branch from IF node.  
     - *Outputs:* Terminal node.  
     - *Edge Cases:* Write failures; concurrent edits; data format mismatches.

---

### 3. Summary Table

| Node Name                    | Node Type                       | Functional Role                             | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                     |
|------------------------------|--------------------------------|---------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------|
| ⏰ Trigger: Check Job Listings | Schedule Trigger               | Initiate workflow daily                      | None                         | 🛠️ Set Search Parameters       | Starts the automation on a regular daily interval to monitor latest pricing updates.                            |
| 🛠️ Set Search Parameters       | Set Node                      | Define static URL parameter                   | ⏰ Trigger                    | Retrieve Pricing data          | Defines the URL to scrape; reusable for different tools by changing this value.                                 |
| Retrieve Pricing data         | Google Sheets (Read)           | Fetch historical pricing data                 | 🛠️ Set Search Parameters       | AI agent                      | Retrieves last known pricing to compare against fresh scraped data.                                            |
| AI agent                     | Langchain Agent Node           | Orchestrate scraping and AI extraction       | Retrieve Pricing data         | If price changes              | AI-driven scraping with Bright Data MCP; parses pricing into structured JSON using OpenAI.                      |
| MCP Client to Scrape as markdown | Bright Data MCP Client Tool  | Fetch page content via proxy                   | AI agent (tool call)          | AI agent                      | Uses Bright Data MCP proxy to scrape page as markdown.                                                         |
| 🧠 OpenAI: LLM Brain          | Langchain OpenAI Chat Model    | Interpret markdown content                     | MCP Client                   | AI agent                      | Runs GPT-4o-mini to extract pricing info from scraped markdown.                                                |
| OpenAI Chat Model             | Langchain OpenAI Chat Model    | Support model in parsing chain                 | Structured Output Parser      | Auto-fixing Output Parser     | Secondary LLM node used for output parsing.                                                                     |
| Structured Output Parser      | Langchain Output Parser        | Parse AI output into structured JSON           | OpenAI Chat Model            | Auto-fixing Output Parser     | Converts text to JSON with detailed pricing schema.                                                             |
| Auto-fixing Output Parser     | Langchain Output Parser (Auto) | Correct and finalize parsed JSON output        | Structured Output Parser      | AI agent                      | Attempts to autofix parser output for consistency before feeding back to AI agent.                              |
| If price changes              | If Node                       | Compare scraped vs saved prices                | AI agent                     | No Operation, do nothing; Update google sheet | Checks if prices changed to avoid unnecessary updates.                                                         |
| No Operation, do nothing     | NoOp Node                     | Confirm no update needed                        | If price changes (true)       | None                         | Path taken if prices match; no operation performed.                                                             |
| Update google sheet          | Google Sheets (Write)          | Update sheet with new pricing data             | If price changes (false)      | None                         | Writes new pricing and features to Google Sheet when changes are detected.                                      |
| Sticky Note                  | Sticky Note                   | Documentation and explanations                  | None                         | None                         | Provides detailed section-wise explanations of workflow logic.                                                |
| Sticky Note1                 | Sticky Note                   | Documentation for AI Agent scraping section     | None                         | None                         | Explains AI agent's role and internal nodes.                                                                    |
| Sticky Note2                 | Sticky Note                   | Documentation for Compare, Decide & Update      | None                         | None                         | Explains IF node logic and update paths.                                                                        |
| Sticky Note3                 | Sticky Note                   | Affiliate link for Bright Data                  | None                         | None                         | Affiliate commission note with Bright Data link.                                                                |
| Sticky Note4                 | Sticky Note                   | Full detailed workflow documentation            | None                         | None                         | Extensive overview and visual summary, plus extension ideas.                                                    |
| Sticky Note9                 | Sticky Note                   | Workflow assistance contact and resource links  | None                         | None                         | Contact info and useful external links for support and tutorials.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 09:00 AM.

2. **Create Set node named "🛠️ Set Search Parameters"**  
   - Add a string parameter named `url`.  
   - Set value to `https://clickup.com/pricing`.  
   - Connect output of Schedule Trigger to this node.

3. **Create Google Sheets Read node named "Retrieve Pricing data"**  
   - Connect input from "🛠️ Set Search Parameters".  
   - Set Document ID to the Google Sheet containing pricing data (e.g., "1olDNB5lFN8NGfVK8otJF2A8aYiR5jG6ve7cqwPyPXrY").  
   - Set Sheet Name to "Sheet1" (gid=0).  
   - Use Google Sheets OAuth2 credentials.

4. **Create Langchain Agent node named "AI agent"**  
   - Connect input from "Retrieve Pricing data".  
   - Configure prompt text:  
     ```
     Scrape Plan name and pricing from the url below
     url: {{ $('🛠️ Set Search Parameters').item.json.url }}
     ```  
   - Enable output parser.  
   - Use OpenAI credentials and MCP Client tool integration.

5. **Create MCP Client Tool node named "MCP Client to Scrape as markdown"**  
   - Configure tool name as "scrape_as_markdown".  
   - Operation: executeTool.  
   - Provide manual description explaining markdown scraping.  
   - Use Bright Data MCP API credentials.  
   - Connect as a tool invoked by "AI agent".

6. **Create Langchain OpenAI Chat Model node named "🧠 OpenAI: LLM Brain"**  
   - Use model "gpt-4o-mini".  
   - Connect input from MCP Client node.  
   - Use OpenAI API credentials.

7. **Create Langchain OpenAI Chat Model node named "OpenAI Chat Model"**  
   - Use model "gpt-4o-mini".  
   - Connect input from Structured Output Parser node (to be created).  
   - Use OpenAI API credentials.

8. **Create Langchain Structured Output Parser node named "Structured Output Parser"**  
   - Provide JSON schema example for pricing plans, including plan_name, price, and key_features array.  
   - Connect input from "OpenAI Chat Model".

9. **Create Langchain Auto-fixing Output Parser node named "Auto-fixing Output Parser"**  
   - Default options.  
   - Connect input from "Structured Output Parser" and "OpenAI Chat Model".  
   - Connect output back to "AI agent" node's parser input.

10. **Create If node named "If price changes"**  
    - Connect input from "AI agent".  
    - Set conditions: compare four pricing fields between "Retrieve Pricing data" and "AI agent" output using string equals operation.  
    - TRUE branch: connect to "No Operation, do nothing".  
    - FALSE branch: connect to "Update google sheet".

11. **Create No Operation node named "No Operation, do nothing"**  
    - Connect input from TRUE branch of IF node.

12. **Create Google Sheets Write node named "Update google sheet"**  
    - Connect input from FALSE branch of IF node.  
    - Use same Google Sheet document and sheet as "Retrieve Pricing data".  
    - Configure to update row 2 with plan_name, price, and key_features for 4 plans from AI output JSON.  
    - Match on `row_number` column.  
    - Use Google Sheets OAuth2 credentials.

13. **Add Sticky Notes for documentation**  
    - Add notes describing each section as per the original workflow for clarity and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| I’ll receive a tiny commission if you join Bright Data through this link — thanks for support!  | Affiliate link: https://get.brightdata.com/1tndi4600b25                                                  |
| For any questions or support, contact Yaron at Yaron@nofluff.online                              | Support contact and workflow assistance                                                                 |
| Explore more tips and tutorials on YouTube and LinkedIn                                         | YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/    |
| Visual summary of workflow logic demonstrates clear flow from trigger to update or no action    | Included in Sticky Note4                                                                                  |
| Extension ideas: Slack alerts, dashboards, auto-save snapshots, multi-competitor tracking        | Suggested enhancements for broader application                                                           |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.