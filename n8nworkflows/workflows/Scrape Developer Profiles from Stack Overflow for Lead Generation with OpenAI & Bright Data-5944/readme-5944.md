Scrape Developer Profiles from Stack Overflow for Lead Generation with OpenAI & Bright Data

https://n8nworkflows.xyz/workflows/scrape-developer-profiles-from-stack-overflow-for-lead-generation-with-openai---bright-data-5944


# Scrape Developer Profiles from Stack Overflow for Lead Generation with OpenAI & Bright Data

---
### 1. Workflow Overview

This n8n workflow automates the process of scraping developer profiles from Stack Overflow for lead generation purposes. It leverages AI (OpenAI GPT-4o-mini model) to dynamically generate scraping instructions, uses Bright Data’s MCP client to perform the actual HTML scraping, parses the scraped data into structured JSON, formats it, and finally saves the results into Google Sheets.

The workflow is logically divided into three main blocks:

- **1.1 Trigger & Input Setup:** Manual initiation and configuration of scraping parameters such as target URL and output format.
- **1.2 AI-Powered Scraping Brain:** The core automation logic where AI generates scraping instructions, manages conversation memory, executes the scraping tool, and parses the output into structured data.
- **1.3 Format & Save to Google Sheets:** Cleans and formats the parsed data and appends it to a Google Sheet for persistent storage.

Each block includes nodes tightly coupled by data flow and functional dependencies, creating a clear modular architecture for ease of maintenance and extensibility.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Setup

**Overview:**  
This block initiates the workflow manually and sets up input parameters such as the Stack Overflow users URL and the desired scraping output format.

**Nodes Involved:**  
- Start Scraping  
- Input Setup  

**Node Details:**

- **Start Scraping**  
  - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
  - Role: Allows user to start the scraping process manually at any time.  
  - Configuration: No parameters; simple click-to-start node.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to Input Setup node.  
  - Edge Cases: No direct failure modes; workflow won’t start without user action.  

- **Input Setup**  
  - Type: Set (n8n-nodes-base.set)  
  - Role: Defines or overrides input variables for the scraping process.  
  - Configuration:  
    - `url` set to "https://stackoverflow.com/users" (target page to scrape)  
    - `format` set to "scrape_as_markdown" (output format expected from scraper)  
  - Inputs: From Start Scraping node  
  - Outputs: Feeds AI Agent node  
  - Edge Cases: If URL or format is changed to invalid values, downstream scraping may fail or produce no data. Validate URL correctness externally if needed.

---

#### 1.2 AI-Powered Scraping Brain

**Overview:**  
This block uses OpenAI’s GPT-powered AI agent to dynamically generate instructions for scraping, manages session memory to retain context, triggers Bright Data’s MCP client to perform the scrape, and parses the scraped HTML into structured JSON data.

**Nodes Involved:**  
- AI Agent: Generate Scraper Instructions  
- OpenAI Chat Model  
- Conversation Memory  
- MCP Client to Scrape as HTML  
- Structured Output Parser  
- Auto-fixing Output Parser  
- OpenAI Chat Model1 (used by Auto-fixing Output Parser)  

**Node Details:**

- **AI Agent: Generate Scraper Instructions**  
  - Type: Langchain Agent (AI agent node)  
  - Role: Core AI brain that receives input URL and prompt, generates scraping instructions, orchestrates tools and parsing.  
  - Configuration:  
    - Prompt text uses expression to insert URL dynamically: `=Scrape all users data as per the provided URL:  {{ $json.url }}`  
    - Output parser enabled for structured response.  
  - Inputs: Receives input from Input Setup node and context from Conversation Memory.  
  - Outputs: Passes structured data to Format Data node.  
  - Edge Cases: Possible failure if AI model quota exceeded, prompt malformed, or if AI returns unexpected output.  
  - Version: TypeVersion 2  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides GPT-4o-mini language model for AI Agent.  
  - Configuration: Model set to "gpt-4o-mini" with default options.  
  - Inputs: Connected to AI Agent node’s language model input.  
  - Outputs: Language model responses to AI Agent.  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API rate limits, invalid credentials, connectivity issues.  

- **Conversation Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains session context keyed by custom session key using URL.  
  - Configuration: Session key expression: `=Perform the web scraping for the below URL  {{ $json.url }}`  
  - Inputs: Connected to AI Agent’s memory input.  
  - Outputs: Provides memory context to AI Agent.  
  - Edge Cases: Memory overflow or corruption can cause AI agent to lose context.  

- **MCP Client to Scrape as HTML**  
  - Type: MCP Client Tool (Bright Data MCP)  
  - Role: Executes scraping tool “scrape_as_html” on Bright Data MCP platform to fetch actual webpage content.  
  - Configuration:  
    - Tool name: "scrape_as_html"  
    - Operation: "executeTool"  
    - Tool Parameters: dynamically set from AI Agent output override.  
    - Tool Description: Scrape a single webpage URL and return HTML.  
  - Inputs: Connected to AI Agent’s AI tool input.  
  - Outputs: Returns scraped HTML content to AI Agent.  
  - Credentials: Requires MCP Client API credentials.  
  - Edge Cases: Authentication errors, Bright Data quota limits, network timeouts, malformed tool parameters.  

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI-generated output into a predefined JSON schema representing user profiles.  
  - Configuration: Uses a detailed JSON schema example describing expected structure (users array with fields like name, location, reputation, tags).  
  - Inputs: Connected to AI Agent output parser input.  
  - Outputs: Parsed JSON data to Auto-fixing Output Parser.  
  - Edge Cases: Parsing failures if AI output deviates from schema, malformed JSON.  

- **Auto-fixing Output Parser**  
  - Type: Langchain Output Parser Autofixing  
  - Role: Enhances robustness by automatically correcting minor parsing errors or malformed JSON before passing data back to AI agent.  
  - Configuration: Default options enabled.  
  - Inputs: Receives from Structured Output Parser.  
  - Outputs: Returns corrected parsed data to AI Agent.  
  - Edge Cases: May fail on severe output corruption; fallback error handling needed.  

- **OpenAI Chat Model1**  
  - Type: Langchain OpenAI Chat Model (secondary)  
  - Role: Supports Auto-fixing Output Parser by providing GPT-4o-mini model for parsing correction.  
  - Configuration: Model "gpt-4o-mini" with default options.  
  - Inputs: Connected to Auto-fixing Output Parser.  
  - Outputs: Provides correction suggestions.  
  - Credentials: Uses same OpenAI API credentials.  
  - Edge Cases: Same as other OpenAI model node.  

---

#### 1.3 Format & Save to Google Sheets

**Overview:**  
This block takes the cleaned and structured user profile data, formats it into a shape suitable for Google Sheets, then appends each user’s data as a new row in the configured spreadsheet.

**Nodes Involved:**  
- Format Data for Google Sheets  
- Save Leads to Google Sheet  

**Node Details:**

- **Format Data for Google Sheets**  
  - Type: Code (JavaScript)  
  - Role: Maps and filters the parsed forums/users array to output one item per user with relevant fields extracted.  
  - Configuration:  
    - Code extracts fields: baseUrl, profileUrl, name, location, reputation, tags from incoming JSON structure.  
    - Returns an array of items, each representing a single user.  
  - Inputs: Receives structured JSON from AI Agent node.  
  - Outputs: Passes formatted items to Google Sheets node.  
  - Edge Cases: Failures if input data structure changes; empty arrays result in no rows appended.  

- **Save Leads to Google Sheet**  
  - Type: Google Sheets (n8n-nodes-base.googleSheets)  
  - Role: Appends data rows to a specified Google Sheets document and sheet.  
  - Configuration:  
    - Operation: Append  
    - Document ID: "121-9GYz9uIhC-cBQaJ2Rg26paivPVDAvTE9pwgZ7MCA" (example ID)  
    - Sheet Name: "gid=0" (Sheet1)  
    - Columns mapped explicitly to JSON fields (Name, Tags, baseUrl, Location, Reputation, Profile URL)  
    - No type conversion or matching columns enabled.  
  - Inputs: From Format Data node  
  - Outputs: None (end of workflow)  
  - Credentials: Google Sheets OAuth2 account required.  
  - Edge Cases: Google API quota limits, authentication errors, invalid document or sheet name, network issues.  

---

### 3. Summary Table

| Node Name                       | Node Type                            | Functional Role                         | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                          |
|--------------------------------|------------------------------------|---------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Start Scraping                 | Manual Trigger                     | Manual workflow initiation             | None                          | Input Setup                   | Section 1: Trigger & Input Setup - Manual trigger to start scraping with user control                                                  |
| Input Setup                   | Set                               | Define URL and output format inputs    | Start Scraping                | AI Agent: Generate Scraper Instructions | Section 1: Trigger & Input Setup - Setup URL and format parameters                                                                    |
| AI Agent: Generate Scraper Instructions | Langchain Agent                  | Generate AI-driven scraper instructions| Input Setup, Conversation Memory, OpenAI Chat Model, MCP Client, Auto-fixing Output Parser | Format Data for Google Sheets    | Section 2: AI-Powered Scraping Brain - Core AI logic managing scraping instructions and tool execution                                |
| OpenAI Chat Model             | Langchain OpenAI Chat Model       | Provides GPT-4o-mini model to AI Agent | Connected to AI Agent         | AI Agent                      | Section 2: AI-Powered Scraping Brain - Language model for AI agent                                                                    |
| Conversation Memory           | Langchain Memory Buffer Window    | Maintains session context for AI agent | Connected to AI Agent         | AI Agent                      | Section 2: AI-Powered Scraping Brain - Maintains context for consistent scraping instructions                                         |
| MCP Client to Scrape as HTML  | MCP Client Tool                   | Executes Bright Data MCP scraping tool | Connected to AI Agent         | AI Agent                      | Section 2: AI-Powered Scraping Brain - Executes HTML scraping via Bright Data MCP                                                     |
| Structured Output Parser      | Langchain Structured Output Parser| Parses AI output to JSON schema        | Connected to AI Agent         | Auto-fixing Output Parser      | Section 2: AI-Powered Scraping Brain - Parses scraped data into structured JSON                                                       |
| Auto-fixing Output Parser     | Langchain Output Parser Autofixing| Corrects parsing errors automatically  | Structured Output Parser      | AI Agent                      | Section 2: AI-Powered Scraping Brain - Auto-corrects parsing output for robustness                                                    |
| OpenAI Chat Model1            | Langchain OpenAI Chat Model       | Supports auto-fixing parser with GPT   | Connected to Auto-fixing Parser| Auto-fixing Output Parser      | Section 2: AI-Powered Scraping Brain - Language model for output parser autofixing                                                    |
| Format Data for Google Sheets | Code                             | Cleans and maps user profile data      | AI Agent                     | Save Leads to Google Sheet     | Section 3: Format & Save to Sheets - Formats data for Google Sheets storage                                                          |
| Save Leads to Google Sheet    | Google Sheets                    | Appends formatted leads to spreadsheet | Format Data for Google Sheets | None                         | Section 3: Format & Save to Sheets - Saves leads automatically to configured Google Sheet                                             |
| Sticky Note                   | Sticky Note                      | Documentation and workflow guidance    | None                         | None                         | Section 1: Trigger & Input Setup - Explains manual trigger and input setup usage                                                      |
| Sticky Note1                  | Sticky Note                      | Documentation on AI scraping brain     | None                         | None                         | Section 2: AI-Powered Scraping Brain - Explains AI agent, memory, scraper tool, and parsing                                           |
| Sticky Note2                  | Sticky Note                      | Documentation on formatting and saving | None                         | None                         | Section 3: Format & Save to Sheets - Explains formatting code and Google Sheets storage                                              |
| Sticky Note3                  | Sticky Note                      | Affiliate and external resource link   | None                         | None                         | Contains Bright Data affiliate link                                                                                                  |
| Sticky Note4                  | Sticky Note                      | Full workflow overview and benefits    | None                         | None                         | Comprehensive overview combining all sections and final benefits                                                                     |
| Sticky Note9                  | Sticky Note                      | Support contact and social links       | None                         | None                         | Workflow assistance contact info and external links                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add **Manual Trigger** node named "Start Scraping".  
   - No parameters needed. This starts the workflow manually.

2. **Create Set Node for Input Setup**  
   - Add **Set** node named "Input Setup".  
   - Add two fields:  
     - `url` (string) with value: `https://stackoverflow.com/users`  
     - `format` (string) with value: `scrape_as_markdown`  
   - Connect **Start Scraping** → **Input Setup**.

3. **Add Langchain OpenAI Chat Model Node**  
   - Add **OpenAI Chat Model** node named "OpenAI Chat Model".  
   - Set model to `"gpt-4o-mini"`.  
   - Configure OpenAI API credentials.  

4. **Add Conversation Memory Node**  
   - Add **Conversation Memory** node named "Conversation Memory".  
   - Set session key expression to: `=Perform the web scraping for the below URL  {{ $json.url }}`.  

5. **Add MCP Client Tool Node**  
   - Add **MCP Client to Scrape as HTML** node.  
   - Set "Tool Name" to `scrape_as_html`.  
   - Set "Operation" to `executeTool`.  
   - Set "Tool Parameters" to be dynamically overridden by AI Agent output (leave base empty).  
   - Configure Bright Data MCP client credentials.  

6. **Add Structured Output Parser Node**  
   - Add **Structured Output Parser** node.  
   - Paste the JSON schema example describing expected user profile output.  

7. **Add Auto-fixing Output Parser Node**  
   - Add **Auto-fixing Output Parser** node.  
   - Use default options.  

8. **Add Second OpenAI Chat Model Node**  
   - Add **OpenAI Chat Model1** node.  
   - Set model to `"gpt-4o-mini"`.  
   - Use same OpenAI credentials.  
   - Connect this node to **Auto-fixing Output Parser** for parsing correction support.  

9. **Add AI Agent Node**  
   - Add **AI Agent: Generate Scraper Instructions** node.  
   - Set prompt text to: `=Scrape all users data as per the provided URL:  {{ $json.url }}`.  
   - Enable output parser and specify structured output parser and memory inputs.  
   - Connect inputs:  
     - Language Model: **OpenAI Chat Model**  
     - Memory: **Conversation Memory**  
     - AI tool: **MCP Client to Scrape as HTML**  
     - Output Parser: **Auto-fixing Output Parser**  
   - Connect output to **Format Data for Google Sheets** node (to be created next).  
   - Connect input from **Input Setup** node.

10. **Add Code Node for Formatting Data**  
    - Add **Code** node named "Format Data for Google Sheets".  
    - Use JavaScript code:  

    ```javascript
    const forums = items[0].json.output.forums;

    const results = forums.map(user => {
      return {
        json: {
          baseUrl: user.baseUrl,
          profileUrl: user.exampleProfile,
          name: user.selectors.name,
          location: user.selectors.location,
          reputation: user.selectors.reputation,
          tags: user.selectors.tags
        }
      };
    });

    return results;
    ```  
    - Connect input from **AI Agent** node.  
    - Connect output to **Save Leads to Google Sheet** node (next step).

11. **Add Google Sheets Node**  
    - Add **Google Sheets** node named "Save Leads to Google Sheet".  
    - Set operation to `append`.  
    - Configure Google Sheets OAuth2 credentials.  
    - Set Document ID to your target Google Sheet's ID (e.g., `121-9GYz9uIhC-cBQaJ2Rg26paivPVDAvTE9pwgZ7MCA`).  
    - Set Sheet Name to `"gid=0"` or your sheet's ID/name.  
    - Define columns mapping as:  
      - Name = `{{$json.name}}`  
      - Tags = `{{$json.tags}}`  
      - baseUrl = `{{$json.baseUrl}}`  
      - Location = `{{$json.location}}`  
      - Raputation = `{{$json.reputation}}` (note: spelling as per original)  
      - Profile URL = `{{$json.profileUrl}}`  
    - Connect input from **Format Data for Google Sheets**.

12. **Connect all nodes as per above and save workflow.**

13. **(Optional) Add Sticky Notes**  
    - Add sticky notes for documentation and user guidance at appropriate positions to explain workflow sections.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow designed to scrape Stack Overflow user profiles for lead generation using AI and Bright Data MCP | Workflow purpose description                                                                    |
| Bright Data affiliate link: https://get.brightdata.com/1tndi4600b25                                        | Supports workflow creator; optional referral link                                              |
| Contact support: Yaron@nofluff.online                                                                     | For workflow assistance and questions                                                          |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos                         | Additional learning resources                                                                   |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                  | Professional contact and workflow updates                                                      |

---

This document fully describes the Stack Overflow lead scraping workflow with AI-driven scraping instructions, HTML scraping via Bright Data, and automated data persistence to Google Sheets. It covers node details, logic structure, potential failure points, and instructions for full reconstruction without raw JSON.