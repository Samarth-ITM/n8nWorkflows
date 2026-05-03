Track Customer NPS Scores Across Platforms with Bright Data and OpenAI

https://n8nworkflows.xyz/workflows/track-customer-nps-scores-across-platforms-with-bright-data-and-openai-5967


# Track Customer NPS Scores Across Platforms with Bright Data and OpenAI

### 1. Workflow Overview

This workflow automates the process of tracking customer satisfaction by scraping review data from a specified survey page (e.g., Trustpilot for Shopify), calculating the Net Promoter Score (NPS) from these reviews, and logging the results into a Google Sheet for leadership insights.

It consists of three main logical blocks:

- **1.1 Schedule & Input Setup**: Defines when to run and which survey page URL to scrape.  
- **1.2 AI-Powered Scraping and Data Parsing**: Uses an AI agent combined with Bright Data's Mobile Carrier Proxy to scrape live customer reviews, parse them into structured JSON data.  
- **1.3 NPS Calculation and Logging**: Processes the scraped reviews to compute NPS metrics and appends the results into Google Sheets for tracking over time.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Input Setup

**Overview:**  
This block triggers the workflow automatically on a weekly schedule and sets the target survey page URL from which to scrape customer reviews.

**Nodes Involved:**  
- `⏰ Run Weekly NPS Tracker` (Schedule Trigger)  
- `✏️ Set Survey Page URL` (Set node)

**Node Details:**

- **⏰ Run Weekly NPS Tracker**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow every week on Monday at 9 AM.  
  - *Configuration:* Interval set to weekly with trigger day = Monday, hour = 9.  
  - *Inputs:* None (start node).  
  - *Outputs:* Connects to `✏️ Set Survey Page URL`.  
  - *Edge cases:* Workflow won’t run off schedule if n8n server is down; no retries built-in.

- **✏️ Set Survey Page URL**  
  - *Type:* Set (Edit Fields)  
  - *Role:* Defines the URL to the customer reviews page (default: Shopify’s Trustpilot page).  
  - *Configuration:* Sets a string variable `url` with value `"https://www.trustpilot.com/review/shopify.com"`.  
  - *Inputs:* Receives trigger from Schedule node.  
  - *Outputs:* Passes the URL to the scraping agent node.  
  - *Edge cases:* URL needs to be valid and publicly accessible; invalid URLs will cause scraping failures.

---

#### 1.2 AI-Powered Scraping and Data Parsing

**Overview:**  
This block uses an OpenAI-powered AI agent to orchestrate a web scrape via Bright Data’s Mobile Carrier Proxy, extracting customer review data in a structured format.

**Nodes Involved:**  
- `🧠 Scrape Reviews with Agent (MCP)` (Langchain AI Agent)  
- `🎯 Prompt & Guide Agent` (OpenAI Chat Model)  
- `🌐 Execute Web Scrape (Bright Data)` (MCP Client Tool)  
- `Auto-fixing Output Parser` (Langchain Output Parser Autofixing)  
- `OpenAI Chat Model` (support for parsing)  
- `Structured Output Parser` (Langchain Structured Output Parser)

**Node Details:**

- **🧠 Scrape Reviews with Agent (MCP)**  
  - *Type:* Langchain AI Agent  
  - *Role:* Coordinates the scraping task by sending a natural language prompt and invoking the scraping tool.  
  - *Configuration:* Prompt instructs extraction of customer reviews, star ratings (1-5), optional comments, and review dates from the URL passed as `{{ $json.url }}`.  
  - *Input:* Receives URL from the previous node and AI model/tool responses.  
  - *Output:* Provides parsed JSON array of reviews.  
  - *Connections:* Receives AI language model input from `🎯 Prompt & Guide Agent`, and scraping tool execution from `🌐 Execute Web Scrape (Bright Data)`. Outputs to `📊 Calculate NPS from Ratings`.  
  - *Edge cases:* Possible failures include API rate limits, invalid URL scraping, or parsing errors if the output format changes.

- **🎯 Prompt & Guide Agent**  
  - *Type:* OpenAI Chat Model  
  - *Role:* Generates the AI prompt guiding what data to extract and how.  
  - *Configuration:* Uses the GPT-4o-mini model.  
  - *Input:* Triggered to provide prompt text for `🧠 Scrape Reviews with Agent (MCP)`.  
  - *Edge cases:* API auth errors, rate limits.

- **🌐 Execute Web Scrape (Bright Data)**  
  - *Type:* MCP Client Tool  
  - *Role:* Executes the actual web scraping using Bright Data’s proxy infrastructure.  
  - *Configuration:* Uses `scrape_as_markdown` tool with parameters dynamically generated or overridden from AI.  
  - *Credentials:* Bright Data MCP API credentials required.  
  - *Edge cases:* Proxy failures, site blocking, or network timeouts.

- **Auto-fixing Output Parser**  
  - *Type:* Output Parser Autofixing  
  - *Role:* Attempts to auto-correct malformed AI outputs for reliable parsing.  
  - *Input:* Connects from AI language model outputs.  
  - *Output:* Feeds clean data into the AI Agent node.

- **OpenAI Chat Model & Structured Output Parser**  
  - *Role:* Assist in transforming unstructured scraped content into structured JSON arrays with fields like rating, comment, date, user.  
  - *Configuration:* The JSON schema example illustrates expected review structure.

---

#### 1.3 NPS Calculation and Logging

**Overview:**  
This block processes the structured review data to compute the Net Promoter Score and logs the results into a Google Sheet for trend tracking.

**Nodes Involved:**  
- `📊 Calculate NPS from Ratings` (Code Function)  
- `📄 Log NPS to Google Sheet` (Google Sheets node)

**Node Details:**

- **📊 Calculate NPS from Ratings**  
  - *Type:* Code (JavaScript)  
  - *Role:* Calculates NPS by converting 1–5 star ratings to a 0–10 scale, classifying reviews as promoters, passives, or detractors, and computing the NPS formula: (Promoters% - Detractors%) × 100.  
  - *Input:* Receives parsed JSON reviews from the AI Agent node.  
  - *Output:* JSON object containing total responses, counts of promoters, passives, detractors, and rounded NPS score with a summary message.  
  - *Edge cases:* Handles zero responses gracefully by returning NPS=0; potential errors if input data format deviates.

- **📄 Log NPS to Google Sheet**  
  - *Type:* Google Sheets Append  
  - *Role:* Appends the calculated NPS data to a specific Google Sheet tab for record-keeping.  
  - *Configuration:* Maps fields like NPS, Promoters, Passives, Detractors, Total Responses, and a summary message to corresponding sheet columns.  
  - *Credentials:* Google OAuth2 required with edit permissions on the target spreadsheet.  
  - *Edge cases:* API quota limits, permission errors, or connectivity issues may cause logging failure.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                            | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                     |
|-------------------------------|----------------------------------|--------------------------------------------|------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| ⏰ Run Weekly NPS Tracker       | Schedule Trigger                 | Triggers workflow weekly at Monday 9 AM    | None                         | ✏️ Set Survey Page URL           | ## 🔶 **Section 1: Set the Target Survey Page** Controls when and where to begin; automates run |
| ✏️ Set Survey Page URL          | Set                              | Sets the URL for scraping                   | ⏰ Run Weekly NPS Tracker      | 🧠 Scrape Reviews with Agent (MCP) | See above                                                                                      |
| 🧠 Scrape Reviews with Agent (MCP) | Langchain AI Agent              | Coordinates AI-driven scraping              | ✏️ Set Survey Page URL, Auto-fixing Output Parser, 🎯 Prompt & Guide Agent, 🌐 Execute Web Scrape (Bright Data) | 📊 Calculate NPS from Ratings        | ## 🤖 **Section 2: Scrape Reviews Using AI Agent** Uses AI + Bright Data MCP for scraping       |
| 🎯 Prompt & Guide Agent         | OpenAI Chat Model                | Generates natural language prompt           | None (indirect trigger)       | 🧠 Scrape Reviews with Agent (MCP) | See above                                                                                      |
| 🌐 Execute Web Scrape (Bright Data) | MCP Client Tool                 | Runs scraping via Bright Data proxy         | None (indirect trigger)       | 🧠 Scrape Reviews with Agent (MCP) | See above                                                                                      |
| Auto-fixing Output Parser       | Output Parser Autofixing         | Cleans and fixes AI output for parsing      | OpenAI Chat Model, Structured Output Parser | 🧠 Scrape Reviews with Agent (MCP) | See above                                                                                      |
| OpenAI Chat Model              | OpenAI Chat Model                | Assists in parsing AI output                 | None                         | Auto-fixing Output Parser       | See above                                                                                      |
| Structured Output Parser       | Structured Output Parser         | Parses scraped data into structured JSON    | None                         | Auto-fixing Output Parser       | See above                                                                                      |
| 📊 Calculate NPS from Ratings   | Code Function                   | Calculates NPS from parsed reviews          | 🧠 Scrape Reviews with Agent (MCP) | 📄 Log NPS to Google Sheet       | ## 📈 **Section 3: Analyze & Log NPS Results** Calculates and logs NPS                          |
| 📄 Log NPS to Google Sheet      | Google Sheets                   | Logs NPS data into spreadsheet               | 📊 Calculate NPS from Ratings  | None                           | See above                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Monday at 9:00 AM.

2. **Create a Set node for the survey URL**  
   - Type: Set  
   - Add a field named `url` (string).  
   - Set default value to `"https://www.trustpilot.com/review/shopify.com"`.  
   - Connect the Schedule Trigger output to this node.

3. **Set up Langchain AI Agent node for scraping**  
   - Type: Langchain Agent  
   - Prompt: Use a template to instruct extraction of customer reviews, star ratings (1-5), comments, and review dates from the URL passed as `{{ $json.url }}`.  
   - Connect output of the Set node to this AI Agent.

4. **Add OpenAI Chat Model node to generate prompts**  
   - Type: Langchain LM Chat OpenAI  
   - Model: Select `gpt-4o-mini` or similar.  
   - Connect this node as the AI language model input for the AI Agent node.

5. **Add MCP Client Tool node to perform web scraping**  
   - Type: MCP Client Tool  
   - Tool name: `scrape_as_markdown`  
   - Operation: `executeTool`  
   - Parameters: Accept dynamic JSON or use AI override to enable flexible scraping parameters.  
   - Connect this as the tool input for the AI Agent node.  
   - Set up Bright Data MCP credentials accordingly.

6. **Add Output Parser nodes**  
   - Add a Structured Output Parser node with a JSON schema example describing the expected review data structure (`rating`, `comment`, `date`, `user`).  
   - Add an Output Parser Autofixing node to clean AI output.  
   - Connect OpenAI Chat Model and Structured Output Parser to Autofixing Parser, then connect Autofixing Parser output to the AI Agent node.

7. **Create a Code node to calculate NPS**  
   - Use JavaScript code to:  
     - Receive reviews JSON array.  
     - Convert 1–5 star ratings into 0–10 scale.  
     - Categorize into Promoters (9-10), Passives (7-8), Detractors (0-6).  
     - Calculate NPS = ((Promoters - Detractors) / total) * 100.  
     - Output totals and NPS score with a summary message.  
   - Connect AI Agent output to this node.

8. **Create Google Sheets node to log NPS**  
   - Operation: Append  
   - Map fields: `NPS`, `Passive`, `Detractor`, `Promoters`, `Total Responses`, and `summary` to corresponding columns.  
   - Specify target spreadsheet ID and sheet name (e.g., Sheet1).  
   - Connect Code node output to this node.  
   - Configure Google OAuth2 credentials with edit access.

9. **Connect nodes in this order:**  
   Schedule Trigger → Set URL → AI Agent (with linked AI and tool nodes) → NPS Calculation → Google Sheets Append.

10. **Test the workflow:**  
    - Run manually first to verify correct scraping, parsing, calculation, and logging.  
    - Adjust URLs, prompt texts, or sheet mappings as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow automatically runs weekly, scraping customer reviews and calculating NPS for leadership insights.           | Workflow description and purpose                             |
| Bright Data proxy used to bypass anti-bot protections on JavaScript-heavy review sites like Trustpilot.              | Bright Data MCP integration                                   |
| OpenAI GPT-4o-mini model powers natural language understanding and output parsing for scraping instructions.        | OpenAI model choice                                          |
| NPS calculation converts 1–5 star ratings to 0–10 scale and categorizes users as promoters, passives, detractors.    | NPS calculation logic                                        |
| Google Sheets logging allows tracking trends over time, enabling proactive customer satisfaction management.         | Reporting and data persistence                               |
| Contact for support: Yaron@nofluff.online; YouTube and LinkedIn channels for workflow tips and tutorials available.  | Workflow author contact and resources                         |
| Affiliate note: Referral link to Bright Data for supporting free content creation: https://get.brightdata.com/1tndi4600b25 | Affiliate program information                                |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.