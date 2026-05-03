Scrape Event Data from 10times with Bright Data MCP, AI & Google Sheets

https://n8nworkflows.xyz/workflows/scrape-event-data-from-10times-with-bright-data-mcp--ai---google-sheets-5947


# Scrape Event Data from 10times with Bright Data MCP, AI & Google Sheets

### 1. Workflow Overview

This workflow automates the scraping of event attendee data from the website 10times.com using Bright Data’s Mobile Carrier Proxy (MCP) and AI-driven extraction, then saves the structured data into Google Sheets for analysis or lead generation. Its core purpose is to bypass anti-bot protections by leveraging AI agents combined with proxy-based scraping, parse the scraped data into structured JSON, and store it for business intelligence or marketing outreach.

**Target Use Cases:**  
- Event marketing and lead generation  
- Competitor and market research on event participation  
- Automating data collection from dynamically loaded web pages with anti-scraping measures  

**Logical Blocks:**  
- **1.1 Trigger & Input Configuration:** Defines when the workflow runs and sets the input URL and scraping parameters.  
- **1.2 Smart Scraping via AI Agent:** Coordinates scraping using AI instructions, Bright Data MCP scraping, and memory context to handle dynamic content and format responses.  
- **1.3 Data Parsing & Cleaning:** Transforms raw markdown scraping results into clean structured JSON for reliable downstream use.  
- **1.4 Save & Reuse Leads:** Appends the parsed data into a Google Sheets document for further business use.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Configuration  
**Overview:**  
Starts the workflow on a scheduled trigger and sets the URL and scraping format parameters for the AI agent.

**Nodes Involved:**  
- 🕒 Schedule Scraper  
- ⚙️ Input URL & Params  

**Node Details:**  

- **🕒 Schedule Scraper**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow execution daily at 9:00 AM.  
  - Configuration: Runs once daily at hour 9.  
  - Inputs: None (trigger node).  
  - Outputs: Passes control to "⚙️ Input URL & Params".  
  - Edge Cases: Workflow won’t run if system time or n8n scheduler is misconfigured.

- **⚙️ Input URL & Params**  
  - Type: Set Node  
  - Role: Defines static input parameters for scraping: URL = "https://10times.com" and format = "scrape_as_markdown".  
  - Configuration: Hardcoded URL and output format as markdown for the scraper.  
  - Inputs: From Schedule Scraper.  
  - Outputs: Passes parameters to the AI Agent node.  
  - Edge Cases: URL must be valid and reachable; incorrect URL leads to scraping failure.

---

#### 1.2 Smart Scraping via AI Agent  
**Overview:**  
This block handles the AI-driven scraping orchestration using Bright Data’s MCP proxy service and OpenAI models to handle dynamic data extraction and session memory.

**Nodes Involved:**  
- 🤖 Bright Data AI Agent  
- 🧠 Agent Memory Model  
- 🕷️ MCP Scraper: 10times  

**Node Details:**  

- **🤖 Bright Data AI Agent**  
  - Type: Langchain Agent Node  
  - Role: Coordinates scraping instructions, specifying URL and extraction targets (categories, events, feedback, venues) using the MCP scraper tool.  
  - Configuration: Uses a prompt that instructs scraping "all category, featured_event, attendee_feedback and venue data" from the URL in markdown format.  
  - Inputs: Receives parameters from "⚙️ Input URL & Params" and memory context from "🧠 Agent Memory Model".  
  - Outputs: Passes scraped results to Google Sheets node.  
  - Edge Cases: Requires valid API credentials for MCP and OpenAI; errors if AI prompt fails or MCP tool is unreachable.

- **🧠 Agent Memory Model**  
  - Type: OpenAI Chat Model (gpt-4o-mini)  
  - Role: Maintains conversational memory for the AI agent to persist context (e.g., pagination, scroll commands).  
  - Configuration: Uses GPT-4o-mini model with OpenAI credentials.  
  - Inputs: Feeds memory context to Bright Data AI Agent.  
  - Outputs: Provides context to AI Agent node.  
  - Edge Cases: API rate limits or auth failures can disrupt memory context.

- **🕷️ MCP Scraper: 10times**  
  - Type: MCP Client Tool Node  
  - Role: Executes the actual web scraping on Bright Data’s Mobile Carrier Proxy, simulating a real mobile device to bypass bot detection.  
  - Configuration: Tool "scrape_as_markdown" with dynamic parameters injected from AI agent.  
  - Inputs: Invoked by AI Agent node as a tool action.  
  - Outputs: Returns markdown scraped content to AI Agent node.  
  - Credentials: Requires MCP Client API credentials.  
  - Edge Cases: Proxy failures, network timeouts, or anti-bot blocks may cause scraping failure.

---

#### 1.3 Data Parsing & Cleaning  
**Overview:**  
Converts the raw markdown scraped data into structured JSON objects with defined schema fields like categories, events, feedback, and venues for easy consumption.

**Nodes Involved:**  
- Structured Output Parser  
- Auto-fixing Output Parser  
- OpenAI Chat Model  

**Node Details:**  

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser Node  
  - Role: Parses markdown input into structured JSON according to a provided JSON schema example.  
  - Configuration: Uses a detailed JSON schema example defining categories, featured_events, attendee_feedback, and venues with example fields.  
  - Inputs: Receives raw markdown from AI Agent’s output.  
  - Outputs: Parsed structured JSON sent to Auto-fixing Output Parser.  
  - Edge Cases: Parsing fails if markdown format deviates significantly from expected pattern.

- **Auto-fixing Output Parser**  
  - Type: Langchain Auto-fixing Output Parser Node  
  - Role: Automatically detects and corrects minor parsing errors to ensure JSON validity.  
  - Configuration: Default auto-fix enabled.  
  - Inputs: From Structured Output Parser.  
  - Outputs: Fixed JSON passed back to AI Agent node for final output.  
  - Edge Cases: Complex errors might not be auto-fixable, causing failure.

- **OpenAI Chat Model**  
  - Type: Langchain Chat Model Node (gpt-4o-mini)  
  - Role: Supports parsing with AI assistance for error correction and interpretation.  
  - Configuration: Uses GPT-4o-mini model with OpenAI credentials.  
  - Inputs: Supports Auto-fixing Output Parser as language model.  
  - Outputs: Helps improve parsing accuracy.  
  - Edge Cases: API limits and latency affect performance.

---

#### 1.4 Save & Reuse Leads  
**Overview:**  
Appends the cleaned, structured attendee and event data into a Google Sheets document for storage, analysis, and reuse.

**Nodes Involved:**  
- 📥 Save to Google Sheets  

**Node Details:**  

- **📥 Save to Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Inserts new rows into a specified Google Sheets spreadsheet with parsed event data fields.  
  - Configuration:  
    - Document ID set to a specific Google Sheet.  
    - Sheet name (gid=0) targeted for data insertion.  
    - Columns mapped for venues, categories, featured_events, attendee_feedback.  
    - Operation: Append new rows.  
  - Inputs: Receives parsed JSON data from AI Agent node.  
  - Credentials: Requires Google Sheets OAuth2 credentials.  
  - Edge Cases: Google API quota limits, permissions, or malformed data can cause write failure.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                        | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                           |
|-------------------------|----------------------------------|-------------------------------------|----------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------|
| 🕒 Schedule Scraper       | Schedule Trigger                  | Starts workflow on schedule          | -                          | ⚙️ Input URL & Params           | This is the starting point; set schedule to run daily or manually.                                  |
| ⚙️ Input URL & Params      | Set                              | Defines URL and scraping parameters  | 🕒 Schedule Scraper          | 🤖 Bright Data AI Agent          | Set the target URL (default https://10times.com) and output format for scraping.                    |
| 🤖 Bright Data AI Agent    | Langchain Agent                  | Coordinates scraping instructions    | ⚙️ Input URL & Params, 🧠 Agent Memory Model | 📥 Save to Google Sheets          | Uses AI to direct MCP scraper, extract event and attendee data in markdown.                         |
| 🧠 Agent Memory Model      | OpenAI Chat Model                | Maintains AI session memory          | -                          | 🤖 Bright Data AI Agent          | Helps AI agent persist context across scraping steps (e.g., scrolling).                            |
| 🕷️ MCP Scraper: 10times    | MCP Client Tool                  | Executes scraping via Bright Data MCP | Called by 🤖 Bright Data AI Agent | 🤖 Bright Data AI Agent          | Runs scraper on mobile proxy, bypassing bot protections.                                           |
| Structured Output Parser  | Langchain Structured Parser      | Parses markdown to structured JSON  | Output from 🤖 Bright Data AI Agent | Auto-fixing Output Parser         | Converts scraped markdown into clean JSON schema.                                                  |
| Auto-fixing Output Parser | Langchain Auto-fixing Parser     | Auto-corrects parsing errors         | Structured Output Parser    | 🤖 Bright Data AI Agent          | Fixes minor JSON parse errors to ensure valid output.                                              |
| OpenAI Chat Model         | Langchain Chat Model             | Supports parsing with AI             | -                          | Auto-fixing Output Parser       | Provides language model support for auto-fixing parser.                                           |
| 📥 Save to Google Sheets    | Google Sheets                   | Stores structured data into Sheets  | 🤖 Bright Data AI Agent     | -                              | Appends attendee and event data into specified Google Sheet for analysis and reuse.                |
| Sticky Note               | Sticky Note                      | Documentation and instructions      | -                          | -                              | Section explanations for users, including scheduling, AI scraping details, and saving data notes.  |
| Sticky Note1              | Sticky Note                      | Section 2 explanation                | -                          | -                              | Details about AI agent, MCP scraper, and parsing advantages.                                      |
| Sticky Note2              | Sticky Note                      | Section 3 explanation                | -                          | -                              | Explains use of Google Sheets node for storing leads.                                             |
| Sticky Note3              | Sticky Note                      | Affiliate link notice                | -                          | -                              | Bright Data affiliate link.                                                                        |
| Sticky Note4              | Sticky Note                      | Full workflow summary and beginner guide | -                      | -                              | Comprehensive overview including tips and business use cases.                                     |
| Sticky Note9              | Sticky Note                      | Workflow assistance contact          | -                          | -                              | Contact and social links for workflow support by author.                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: 🕒 Schedule Scraper  
   - Set to trigger daily at 9:00 AM (select "At Hour" → 9).  
   - No credentials needed.

3. **Add a Set node:**  
   - Name: ⚙️ Input URL & Params  
   - Define two fields:  
     - `url`: string, value "https://10times.com" (or your target event page)  
     - `format`: string, value "scrape_as_markdown"  
   - Connect output of 🕒 Schedule Scraper to input of this node.

4. **Add a Langchain Agent node:**  
   - Name: 🤖 Bright Data AI Agent  
   - Parameters:  
     - Text prompt: "Scrape all category, featured_event, attendee_feedback and venue data using \"MCP Client to Scrape as markdown\" tool:\nurl: {{ $json.url }}"  
     - Prompt type: Define  
     - Enable output parser  
   - Connect output of ⚙️ Input URL & Params to input here.  
   - Set credentials for OpenAI API (see step 6) and MCP Client API (see step 7).  

5. **Add OpenAI Chat Model node:**  
   - Name: 🧠 Agent Memory Model  
   - Model: gpt-4o-mini  
   - Connect output as ai_languageModel input to 🤖 Bright Data AI Agent.

6. **Set OpenAI API credentials:**  
   - Create or configure an OpenAI API credential with your API key.  
   - Assign it to nodes using OpenAI (Agent Memory Model, Bright Data AI Agent, OpenAI Chat Model).

7. **Add MCP Client Tool node:**  
   - Name: 🕷️ MCP Scraper: 10times  
   - Tool Name: scrape_as_markdown  
   - Operation: executeTool  
   - Tool Parameters: set to be dynamically passed from AI Agent node (ensure expression `{{$fromAI(...)}}` or equivalent).  
   - Assign your Bright Data MCP Client API credentials here.

8. **Add Structured Output Parser node:**  
   - Name: Structured Output Parser  
   - Provide JSON Schema Example to define expected output fields: categories, featured_events, attendee_feedback, venues (copy from provided schema).  
   - Connect output from AI Agent node to this parser.

9. **Add Auto-fixing Output Parser node:**  
   - Name: Auto-fixing Output Parser  
   - Leave default settings.  
   - Connect Structured Output Parser output to this node.  
   - Connect this node’s output back to AI Agent node’s output parser input.

10. **Add OpenAI Chat Model node for auto-fixing:**  
    - Name: OpenAI Chat Model  
    - Model: gpt-4o-mini  
    - Connect as ai_languageModel input to Auto-fixing Output Parser.

11. **Add Google Sheets node:**  
    - Name: 📥 Save to Google Sheets  
    - Operation: Append  
    - Document ID: Set to your target Google Sheets document ID.  
    - Sheet Name: gid=0 (or your target sheet)  
    - Define columns:  
      - "vanues" mapped to `{{$json.output.venues}}`  
      - "Categories" mapped to `{{$json.output.categories}}`  
      - "Featured events" mapped to `{{$json.output.featured_events}}`  
      - "Attendee feedback" mapped to `{{$json.output.attendee_feedback}}`  
    - Connect output from 🤖 Bright Data AI Agent to this node.  
    - Assign Google Sheets OAuth2 credentials.

12. **Verify all connections:**  
    - Schedule Trigger → Set Input URL & Params → Bright Data AI Agent → Google Sheets  
    - AI Agent uses Agent Memory Model and MCP Scraper as tools for scraping.  
    - Structured Output Parser and Auto-fixing Parser are integrated into AI Agent’s output parsing pipeline.

13. **Activate the workflow** and test by running manually or waiting for scheduled execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow requires valid API credentials for OpenAI and Bright Data MCP Client.                               | Credentials setup in n8n for OpenAI and MCP Client API.                                                |
| Bright Data MCP proxy simulates mobile devices to bypass bot detection, enabling scraping of dynamic content.    | https://brightdata.com/products/mobile-carrier-proxy                                                    |
| The AI Agent uses human-like prompts to control scraping logic and formatting, reducing need for manual selectors.| Sticky Note1 content on AI-driven scraping advantages.                                                |
| Google Sheets output allows easy lead management and marketing workflows without manual data handling.            | Spreadsheet example linked in node parameters.                                                         |
| For workflow support and updates contact Yaron at Yaron@nofluff.online or visit https://www.youtube.com/@YaronBeen/videos | Workflow assistance and tutorials by the author.                                                        |
| Affiliate link for Bright Data to support free content creation: https://get.brightdata.com/1tndi4600b25          | Sticky Note3 content on affiliate program.                                                             |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation platform. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.