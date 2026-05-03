Automated Local Event Monitor with Bright Data MCP and OpenAI Analysis

https://n8nworkflows.xyz/workflows/automated-local-event-monitor-with-bright-data-mcp-and-openai-analysis-5972


# Automated Local Event Monitor with Bright Data MCP and OpenAI Analysis

---
### 1. Workflow Overview

This workflow automates the monitoring and analysis of local events listed on the 10Times website for New York City. Its primary aim is to scrape event data, analyze sponsorship opportunities relevant to a tech company launching a project management product, and store the enriched data for further use. The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Trigger**: Manual initiation of the workflow and setting the target URL for scraping events.
- **1.2 AI-Powered Event Data Scraping**: Using an AI agent combined with Bright Data MCP Client to scrape event listings and parse them into structured JSON.
- **1.3 Event Data Analysis for Sponsorship**: Splitting the event dataset into individual entries and analyzing each for sponsorship potential using OpenAI models.
- **1.4 Data Storage**: Saving the analyzed events with sponsorship ratings into a Google Sheets document for record-keeping and reporting.

This modular structure ensures clear separation of concerns, robustness for potential data issues, and flexibility for future modifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Trigger

- **Overview:**  
  This block initiates the workflow manually and sets the URL for the 10Times New York events page, defining the data source for subsequent scraping.

- **Nodes Involved:**  
  - 🔘 Trigger: Execute Workflow  
  - 🌐 Set 10Times URL 

- **Node Details:**

  - **🔘 Trigger: Execute Workflow**  
    - *Type & Role:* Manual trigger node to start the workflow on demand.  
    - *Configuration:* No parameters; simply user-activated.  
    - *Inputs/Outputs:* No input; outputs trigger downstream nodes.  
    - *Edge Cases:* User forgetting to trigger; no issues expected.  
    - *Version:* Compatible with n8n v1.x.  

  - **🌐 Set 10Times URL**  
    - *Type & Role:* Set node to assign a static URL string.  
    - *Configuration:* Sets variable `URL` to `"https://10times.com/newyork-us"`.  
    - *Inputs/Outputs:* Receives trigger, outputs JSON with URL.  
    - *Edge Cases:* URL hardcoded, so changing event location requires manual adjustment.  
    - *Version:* Uses typeVersion 3.4 (latest stable).  

#### 1.2 AI-Powered Event Data Scraping

- **Overview:**  
  This block leverages a LangChain AI agent integrated with the Bright Data MCP Client to scrape event data from the 10Times page. It extracts structured information on events such as name, location, date, category, description, URL, and attendees.

- **Nodes Involved:**  
  - 🤖 Agent: Scrape Events Data  
    - Subnodes inside the agent:  
      - 💬 AI Model: Data Processing (lmChatOpenAi)  
      - 🌐 Bright Data MCP Client  
      - Auto-fixing Output Parser  
      - 📝 Parse Scraped Data into JSON1 (structured output parser)

- **Node Details:**

  - **🤖 Agent: Scrape Events Data**  
    - *Type & Role:* LangChain Agent node orchestrating scraping via AI and MCP.  
    - *Configuration:*  
      - Prompt instructs agent to scrape specified URL for event details.  
      - Extracts event fields: event_name, location, date, category, description, url, attendees.  
      - Uses `{{ $json.URL }}` from previous Set node.  
      - Output parser enabled to enforce structured response.  
    - *Inputs/Outputs:* Takes URL input, outputs parsed event list JSON.  
    - *Edge Cases:*  
      - Website layout changes could break scraping logic.  
      - Missing attendee counts handled by conditional return.  
      - MCP Client failures or rate limits possible.  
    - *Version:* LangChain agent typeVersion 2.  

  - **💬 AI Model: Data Processing**  
    - *Type & Role:* OpenAI GPT-4.1 mini model, processes raw scraped text.  
    - *Configuration:* Default parameters, GPT-4.1-mini chosen for balance of power and cost.  
    - *Inputs/Outputs:* Input raw scraped text, output refined structured data.  
    - *Edge Cases:* Model response variability, timeout or rate limits.  
    - *Credentials:* Uses configured OpenAI API credentials.  

  - **🌐 Bright Data MCP Client**  
    - *Type & Role:* Executes scraping tool via Bright Data's MCP interface.  
    - *Configuration:* Tool named `scrape_as_markdown` executed with parameters from AI output.  
    - *Inputs/Outputs:* Invoked by AI agent to fetch page content for parsing.  
    - *Edge Cases:* Network failures, authentication errors, quota exhaustion.  
    - *Credentials:* Uses MCP Client API credentials.  

  - **Auto-fixing Output Parser**  
    - *Type & Role:* Improves reliability by auto-correcting output format deviations from AI model.  
    - *Configuration:* Default options.  
    - *Inputs/Outputs:* Receives raw AI output, outputs corrected JSON.  
    - *Edge Cases:* Complex or ambiguous AI outputs may still fail parsing.  

  - **📝 Parse Scraped Data into JSON1**  
    - *Type & Role:* Structured output parser node enforcing JSON schema for events.  
    - *Configuration:* Example JSON schema provided with event fields and sample data.  
    - *Inputs/Outputs:* Input AI text output, output validated structured JSON array.  
    - *Edge Cases:* Schema mismatches or incomplete data may trigger parse errors.  
    - *Version:* Uses typeVersion 1.3 for JSON parsing.  

#### 1.3 Event Data Analysis for Sponsorship

- **Overview:**  
  This block takes the structured event data, splits the list into individual events, then uses an OpenAI GPT-4o-mini model to analyze each event for sponsorship opportunities, rating each out of 10 based on alignment with the company's project management product.

- **Nodes Involved:**  
  - 🔀 Split Events into Separate Items  
  - 💬 AI: Analyze Events for Sponsorship Opportunities

- **Node Details:**

  - **🔀 Split Events into Separate Items**  
    - *Type & Role:* Code node splitting array of events into individual items for parallel processing.  
    - *Configuration:* JavaScript code maps each event JSON object to a separate item.  
    - *Inputs/Outputs:* Input: array of events; Output: multiple items with one event each.  
    - *Edge Cases:* Empty event list results in no output items; malformed JSON may error.  

  - **💬 AI: Analyze Events for Sponsorship Opportunities**  
    - *Type & Role:* OpenAI node using GPT-4o-mini to evaluate sponsorship fit and rate events.  
    - *Configuration:*  
      - Message prompt embeds event details dynamically via expressions.  
      - Instructions specify company context: tech company launching project management product.  
      - Outputs AI message content with rating and analysis.  
    - *Inputs/Outputs:* Input: individual event JSON; Output: sponsorship analysis string.  
    - *Edge Cases:* Model API rate limits, inconsistent or incomplete event data, potential hallucinations in AI output.  
    - *Credentials:* OpenAI API credentials required.  

#### 1.4 Data Storage

- **Overview:**  
  Final block appends the enriched event data along with sponsorship analysis into a dedicated Google Sheets spreadsheet for tracking and reporting.

- **Nodes Involved:**  
  - 📥 Save Events & Sponsorship Ratings to Google Sheets

- **Node Details:**

  - **📥 Save Events & Sponsorship Ratings to Google Sheets**  
    - *Type & Role:* Google Sheets node appending rows of event data and AI analysis.  
    - *Configuration:*  
      - Maps multiple columns: Event name, Location, Date, Category, Description, URL, Attendees, Sponsorship Opportunities.  
      - References fields from split events and AI analysis nodes via expressions.  
      - Appends data to sheet with `gid=0` inside document `17iJ3Qr6GwZF8gGxx7xUEnLVPV7eMADef12IaBwe8qZQ`.  
    - *Inputs/Outputs:* Input: enriched event objects with AI commentary; Output: confirmation of row addition.  
    - *Edge Cases:* Authentication failures, quota limits on Google Sheets API, schema mismatch or invalid data types.  
    - *Credentials:* Uses Google Sheets OAuth2 credential.  
    - *Version:* typeVersion 4.6 supporting latest Google Sheets API features.  

---

### 3. Summary Table

| Node Name                                | Node Type                             | Functional Role                                        | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                                                                    |
|------------------------------------------|-------------------------------------|-------------------------------------------------------|-----------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------|
| 🔘 Trigger: Execute Workflow              | Manual Trigger                      | Manual start of workflow                               | None                              | 🌐 Set 10Times URL                    | Section 1: Input URL and Trigger Workflow                                                                     |
| 🌐 Set 10Times URL                        | Set                                 | Assign URL string for 10Times NYC events               | 🔘 Trigger                         | 🤖 Agent: Scrape Events Data          | Section 1: Input URL and Trigger Workflow                                                                     |
| 🤖 Agent: Scrape Events Data              | LangChain Agent                     | AI-powered scraping of event data from 10Times        | 🌐 Set 10Times URL                 | 🔀 Split Events into Separate Items   | Section 2: Scrape Event Data from 10Times                                                                     |
| 💬 AI Model: Data Processing              | LangChain OpenAI Chat Model         | Process raw scraped text into structured event data    | Part of Agent node                | Part of Agent node                    | Included inside Agent: Scrape Events Data                                                                     |
| 🌐 Bright Data MCP Client                 | MCP Client Tool                    | Web scraping tool used by AI Agent                      | Part of Agent node                | Part of Agent node                    | Included inside Agent: Scrape Events Data                                                                     |
| Auto-fixing Output Parser                 | LangChain Output Parser             | Auto-correct AI output formatting                        | Part of Agent node                | Part of Agent node                    | Included inside Agent: Scrape Events Data                                                                     |
| 📝 Parse Scraped Data into JSON1          | LangChain Structured Output Parser  | Enforce JSON schema on scraped event data               | Part of Agent node                | Part of Agent node                    | Included inside Agent: Scrape Events Data                                                                     |
| 🔀 Split Events into Separate Items       | Code                               | Split event list into individual event items            | 🤖 Agent: Scrape Events Data      | 💬 AI: Analyze Events for Sponsorship | Section 3: Analyze Events for Sponsorship Opportunities                                                       |
| 💬 AI: Analyze Events for Sponsorship Opportunities | OpenAI (LangChain)                 | Analyze each event for sponsorship potential and rate  | 🔀 Split Events into Separate Items | 📥 Save Events & Sponsorship Ratings | Section 3: Analyze Events for Sponsorship Opportunities                                                       |
| 📥 Save Events & Sponsorship Ratings to Google Sheets | Google Sheets                     | Append analyzed event data and ratings to Google Sheet | 💬 AI: Analyze Events for Sponsorship Opportunities | None                                | Section 4: Store Data in Google Sheets                                                                         |
| Sticky Note                              | Sticky Note                        | Documentation and explanations                          | None                              | None                                 | Covers sections 1 to 4 with detailed functional explanations                                                  |
| Sticky Note1                             | Sticky Note                        | Documentation of scraping section                       | None                              | None                                 | Section 2: Scrape Event Data from 10Times                                                                     |
| Sticky Note2                             | Sticky Note                        | Documentation of analysis section                       | None                              | None                                 | Section 3: Analyze Events for Sponsorship Opportunities                                                       |
| Sticky Note3                             | Sticky Note                        | Documentation of storage section                        | None                              | None                                 | Section 4: Store Data in Google Sheets                                                                         |
| Sticky Note4                             | Sticky Note                        | Full workflow summary and benefits                      | None                              | None                                 | Comprehensive overview covering all workflow sections                                                         |
| Sticky Note5                             | Sticky Note                        | Affiliate link for Bright Data                           | None                              | None                                 | Contains affiliate link: https://get.brightdata.com/1tndi4600b25                                              |
| Sticky Note9                             | Sticky Note                        | Workflow assistance and contact info                    | None                              | None                                 | Contact: Yaron@nofluff.online; YouTube & LinkedIn links provided                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: `🔘 Trigger: Execute Workflow`  
   - No configuration required. This node starts the workflow on user execution.

2. **Create Set Node to Define URL**  
   - Node Type: Set  
   - Name: `🌐 Set 10Times URL`  
   - Add an assignment:  
     - Variable Name: `URL`  
     - Type: String  
     - Value: `https://10times.com/newyork-us`  
   - Connect output of the manual trigger to this node.

3. **Create LangChain Agent Node for Scraping**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `🤖 Agent: Scrape Events Data`  
   - Parameters:  
     - Prompt Type: Define  
     - Prompt Text:  
       ```
       You are a data extraction agent.

       Scrape the following 10Times URL for events in New York, US:
       {{ $json.URL }}

       Extract data for the events listed on this page. For each event, provide the following details:

       - event_name: The name of the event.
       - location: The location of the event (venue name and address, if available).
       - date: The event date and time.
       - category: The event's category (e.g., Networking, Tech, Sports, etc.).
       - description: A brief description of the event.
       - url: The URL to the event page.
       - attendees: The number of attendees (if available, otherwise don't return).
       ```
     - Enable output parser.  
   - Credentials: Assign OpenAI API credentials.  
   - Connect output of the Set node to this agent node.

4. **Configure Subnodes within Agent (automatic in agent node but conceptually):**  
   - **AI Model: Data Processing Node**  
     - Use OpenAI GPT-4.1-mini model.  
   - **Bright Data MCP Client Node**  
     - Set tool name to `scrape_as_markdown`.  
     - Credentials: Bright Data MCP API credentials.  
   - **Auto-fixing Output Parser Node**  
     - Default options to auto-fix AI output.  
   - **Structured Output Parser Node**  
     - JSON schema example with event fields (event_name, location, date, category, description, url, attendees).  

5. **Create Code Node to Split Events**  
   - Node Type: Code  
   - Name: `🔀 Split Events into Separate Items`  
   - JavaScript Code:  
     ```javascript
     const events = items[0].json.output;
     return events.map(event => ({ json: event }));
     ```  
   - Connect output of the Agent node to this node.

6. **Create OpenAI Node for Sponsorship Analysis**  
   - Node Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Name: `💬 AI: Analyze Events for Sponsorship Opportunities`  
   - Parameters:  
     - Model ID: `gpt-4o-mini`  
     - Messages:  
       ```
       We have our own company related to tech and we launch a product about project management. Following are some events and I want you to analyze them and then find the sponsorship opportunities and rate it out of 10.
       The event is given below:

       event_name: {{ $json.event_name }}
       location: {{ $json.location }}
       date: {{ $json.date }}
       category: {{ $json.category }}
       description: {{ $json.description }}
       url: {{ $json.url }}
       attendees: {{ $json.attendees }}
       ```  
   - Credentials: OpenAI API credentials.  
   - Connect output of Split Events node to this node.

7. **Create Google Sheets Node to Save Data**  
   - Node Type: Google Sheets  
   - Name: `📥 Save Events & Sponsorship Ratings to Google Sheets`  
   - Operation: Append  
   - Document ID: `17iJ3Qr6GwZF8gGxx7xUEnLVPV7eMADef12IaBwe8qZQ`  
   - Sheet Name: `gid=0`  
   - Columns to map:  
     - Event name: `={{ $json.event_name }}`  
     - Location: `={{ $json.location }}`  
     - Date: `={{ $json.date }}`  
     - Category: `={{ $json.category }}`  
     - Description: `={{ $json.description }}`  
     - URL: `={{ $json.url }}`  
     - Attendees: `={{ $json.attendees }}`  
     - Sponsorship opportunities: `={{ $json.message.content }}` (from AI analysis)  
   - Credentials: Google Sheets OAuth2 credentials.  
   - Connect output of AI analysis node to this node.

8. **Final Workflow Connections:**  
   - Connect nodes in this order:  
     Manual Trigger → Set URL → Agent → Split Events → AI Sponsorship Analysis → Google Sheets Save.

9. **Credential Setup:**  
   - Configure OpenAI API credentials for the GPT models.  
   - Configure Bright Data MCP API credentials for scraping.  
   - Configure Google Sheets OAuth2 credentials with write access to target sheet.

10. **Defaults and Constraints:**  
    - Ensure that the URL in the Set node reflects the desired event location or listing.  
    - The AI models require stable internet and valid API keys with sufficient quota.  
    - Bright Data MCP usage depends on subscription and quotas; monitor usage.  
    - Google Sheets API quotas apply; avoid excessive writes in short intervals.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Affiliate link for Bright Data supporting free content creation: https://get.brightdata.com/1tndi4600b25   | Sticky Note5                                                                                   |
| Workflow assistance contact: Yaron@nofluff.online                                                         | Sticky Note9                                                                                   |
| Additional tips and resources available on YouTube: https://www.youtube.com/@YaronBeen/videos             | Sticky Note9                                                                                   |
| LinkedIn profile for support and updates: https://www.linkedin.com/in/yaronbeen/                           | Sticky Note9                                                                                   |
| The workflow is designed for scraping 10Times event listings specifically for New York but can be adapted   | Change URL in Set node to target other cities or event lists                                  |
| AI models used are GPT-4.1-mini and GPT-4o-mini for balanced cost and performance                           | Requires OpenAI API credentials                                                                |
| Bright Data MCP Client used to bypass anti-bot measures on scraping websites                                | Requires valid Bright Data subscription                                                        |
| Google Sheets is used as a centralized repository for event data and sponsorship analyses                   | Requires Google OAuth2 credentials with write access                                           |

---

**Disclaimer:** The provided text is exclusively sourced from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.