Instant Competitor Product Launch Monitor with Bright Data & OpenAI to Email/Sheets

https://n8nworkflows.xyz/workflows/instant-competitor-product-launch-monitor-with-bright-data---openai-to-email-sheets-5969


# Instant Competitor Product Launch Monitor with Bright Data & OpenAI to Email/Sheets

### 1. Workflow Overview

This n8n workflow, titled **"Instant Competitor Product Launch Monitor with Bright Data & OpenAI to Email/Sheets"**, automates the process of monitoring competitor product launches by scraping product reviews from a target website (default: The Verge Reviews), structuring the data using AI, and then sending notifications and logging results. The workflow is designed to provide timely alerts to an R&D team and maintain a historical record in Google Sheets.

The workflow is logically divided into four main blocks:

- **1.1 Schedule & Scrape Target Configuration:** Defines when the workflow runs and sets the URL to scrape.
- **1.2 Agent-Powered Scraping with Bright Data:** Uses an AI agent integrated with Bright Data MCP to scrape and parse product review data.
- **1.3 Split & Format Each Review:** Processes the scraped data array into individual structured review items.
- **1.4 Notify & Log the Insights:** Sends email alerts per product review and appends the data to a Google Sheet for historical tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Scrape Target Configuration

- **Overview:**  
  This block sets the workflow trigger time and defines the target webpage URL for scraping. It controls the timing and target source of data extraction.

- **Nodes Involved:**  
  - ⏰ Daily Check (7AM)  
  - 🛠 Set Scrape Target (Verge Reviews)

- **Node Details:**  

  - **⏰ Daily Check (7AM)**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow automatically every day at 7 AM.  
    - Configuration: Interval trigger set to hour 7 (7:00 AM daily).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Set Scrape Target (Verge Reviews)".  
    - Edge Cases: Missed triggers if n8n server downtime occurs; time zone considerations must align with desired local time.  

  - **🛠 Set Scrape Target (Verge Reviews)**  
    - Type: Set Node  
    - Role: Defines the URL to scrape by assigning a string value.  
    - Configuration: Sets a single field `url` with value `https://www.theverge.com/reviews`.  
    - Inputs: From schedule trigger node.  
    - Outputs: Connects to the "🤖 Bright Data Scraper Agent".  
    - Edge Cases: If URL is invalid or unreachable, scraping will fail downstream; easy to customize for other competitor sites.  

---

#### 1.2 Agent-Powered Scraping with Bright Data

- **Overview:**  
  This is the core scraping block, using an AI-powered agent combined with Bright Data's scraping infrastructure to extract product launch data in a structured format.

- **Nodes Involved:**  
  - 🤖 Bright Data Scraper Agent  
  - OpenAI Chat Model  
  - MCP Client  
  - Auto-fixing Output Parser  
  - OpenAI Chat Model1  
  - Structured Output Parser1

- **Node Details:**  

  - **🤖 Bright Data Scraper Agent**  
    - Type: Langchain AI Agent Node  
    - Role: Coordinates the scraping process using instructions and sub-tools.  
    - Configuration:  
      - Text prompt instructs the agent to "Use Bright Data Web Unlocker to scrape the latest product titles, release dates, and brief summaries from URL."  
      - Uses prompt type "define" with output parser enabled.  
    - Inputs: Receives URL from "Set Scrape Target".  
    - Outputs: Array of scraped product review data passed to "Split & Format Each Review".  
    - Dependencies: Uses OpenAI Chat Model and MCP Client as sub-tools for scraping and data extraction.  
    - Edge Cases: Possible API rate limits, scraping failures if site structure changes, or anti-bot bypass issues.  

  - **OpenAI Chat Model (first instance)**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides language model capabilities to interpret instructions for scraping.  
    - Configuration: Uses GPT-4o-mini model variant.  
    - Inputs: Connected as language model to the AI Agent node.  
    - Outputs: Language model responses to the agent.  
    - Credentials: Requires valid OpenAI API key.  
    - Edge Cases: API quota limits, latency, or model errors.  

  - **MCP Client**  
    - Type: MCP Client Tool Node  
    - Role: Executes Bright Data's scraping tool ("scrape_as_markdown") to bypass anti-bot protections and fetch raw data.  
    - Configuration: Uses parameters dynamically overridden by AI.  
    - Inputs: Connected as tool in AI Agent.  
    - Outputs: Raw scraped markdown data.  
    - Credentials: Requires Bright Data MCP credentials.  
    - Edge Cases: Scraper tool failures, invalid credentials, or network issues.  

  - **Auto-fixing Output Parser**  
    - Type: Langchain Output Parser with Autofix  
    - Role: Parses and cleans AI-generated output, correcting minor errors to ensure valid JSON structure.  
    - Inputs: Connected to output of "OpenAI Chat Model1" and feeds parsed output to "Bright Data Scraper Agent".  
    - Outputs: Clean structured data for downstream use.  
    - Edge Cases: Parsing failures if output deviates significantly from expected structure.  

  - **OpenAI Chat Model1 (second instance)**  
    - Same type and role as the first OpenAI Chat Model node but used downstream for parsing assistance.  
    - Inputs: Receives data from MCP Client through the agent.  
    - Outputs: Passes processed data to "Structured Output Parser1".  
    - Edge Cases: Same as above.  

  - **Structured Output Parser1**  
    - Type: Structured Output Parser  
    - Role: Converts example-based JSON schema into parsed structured data for automation.  
    - Configuration: Contains a sample JSON schema illustrating expected output fields: title, url, release_date, summary.  
    - Inputs: Connected from OpenAI Chat Model1.  
    - Outputs: Passes clean JSON array to the Auto-fixing Output Parser.  
    - Edge Cases: Schema mismatches or incomplete data may cause parsing errors.  

---

#### 1.3 Split & Format Each Review

- **Overview:**  
  This Code node takes the array of product reviews from the scraper agent and splits it into individual review items, formatting URLs to be absolute.

- **Nodes Involved:**  
  - 🧾 Split & Format Each Review (Code Node)

- **Node Details:**  

  - **🧾 Split & Format Each Review**  
    - Type: Code Node (JavaScript)  
    - Role: Processes the array of scraped reviews and emits one item per review with formatted fields.  
    - Configuration:  
      - Maps each review object to a new object with:  
        - `title` (string)  
        - `url` (absolute URL by prepending `https://www.theverge.com` to relative URLs)  
        - `release_date` (string)  
        - `summary` (string)  
      - Returns an array of separate JSON objects for downstream nodes.  
    - Inputs: Receives JSON array from "🤖 Bright Data Scraper Agent".  
    - Outputs: Individual review items to email and sheet nodes.  
    - Edge Cases: Assumes input contains valid fields; missing or malformed fields could cause errors.  

---

#### 1.4 Notify & Log the Insights

- **Overview:**  
  This block handles notification and data persistence. Each product review triggers an email alert and is appended to a Google Sheets document for record-keeping.

- **Nodes Involved:**  
  - 📤 Email R&D: Product Alerts (Gmail Node)  
  - 📊 Log to Google Sheet (Review History)

- **Node Details:**  

  - **📤 Email R&D: Product Alerts**  
    - Type: Gmail Node  
    - Role: Sends an email alert for each new product launch review.  
    - Configuration:  
      - Recipient: `shahkar.genai@gmail.com`  
      - Subject: Includes product title dynamically.  
      - Message body: Text email with product details (title, release date, summary, and a link).  
      - Email Type: Plain text.  
    - Inputs: Receives individual review items from code node.  
    - Outputs: Passes items to Google Sheets node.  
    - Credentials: Requires Gmail OAuth2 credentials.  
    - Edge Cases: Email sending limits, invalid recipient address, OAuth token expiration.  

  - **📊 Log to Google Sheet (Review History)**  
    - Type: Google Sheets Node  
    - Role: Appends each review to a specific Google Sheet as a record.  
    - Configuration:  
      - Document ID and Sheet name specified (linked to "Product Launches" sheet).  
      - Columns mapped: URL, Title, Summary, Release date.  
      - Append operation mode.  
    - Inputs: From email node.  
    - Outputs: None (terminal node).  
    - Credentials: Requires Google Sheets OAuth2 credentials.  
    - Edge Cases: Sheet access permission issues, API quota limits, malformed data.  

---

### 3. Summary Table

| Node Name                         | Node Type                                    | Functional Role                          | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                                    |
|----------------------------------|----------------------------------------------|----------------------------------------|-----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------|
| ⏰ Daily Check (7AM)              | Schedule Trigger                             | Triggers workflow daily at 7 AM        | None                              | 🛠 Set Scrape Target (Verge Reviews) | Controls workflow timing and automation start time.                                                           |
| 🛠 Set Scrape Target (Verge Reviews) | Set Node                                   | Sets target URL for scraping            | ⏰ Daily Check (7AM)               | 🤖 Bright Data Scraper Agent       | Defines the scraping target URL; easy to customize for other competitor sites.                                |
| 🤖 Bright Data Scraper Agent      | Langchain AI Agent                          | Coordinates scraping using AI + Bright Data | 🛠 Set Scrape Target (Verge Reviews) | 🧾 Split & Format Each Review     | Core AI-powered scraper using Bright Data MCP and OpenAI to extract structured product review data.           |
| OpenAI Chat Model                | Langchain OpenAI Chat Model                  | Language model for AI agent             | Connected internally to Agent     | Connected internally to Agent    | Provides GPT-4o-mini model for AI agent instruction parsing.                                                  |
| MCP Client                      | MCP Client Tool                              | Executes Bright Data scraping tool      | Connected internally to Agent     | Connected internally to Agent    | Runs Bright Data scraper tool to bypass anti-bot for data extraction.                                         |
| Auto-fixing Output Parser        | Langchain Output Parser with Autofix         | Cleans and fixes AI output JSON         | Structured Output Parser1, OpenAI Chat Model1 | 🤖 Bright Data Scraper Agent     | Ensures output is valid JSON for downstream processing.                                                        |
| OpenAI Chat Model1               | Langchain OpenAI Chat Model                  | Secondary language model for parsing    | MCP Client                      | Structured Output Parser1         | Assists in parsing scraped data into structured format.                                                       |
| Structured Output Parser1        | Langchain Structured Output Parser           | Parses example schema to structured JSON | OpenAI Chat Model1               | Auto-fixing Output Parser         | Converts scraped data into a structured JSON array according to schema example.                               |
| 🧾 Split & Format Each Review    | Code Node (JavaScript)                       | Splits array into individual review items | 🤖 Bright Data Scraper Agent      | 📤 Email R&D: Product Alerts      | Splits and formats reviews; converts relative URLs to absolute.                                               |
| 📤 Email R&D: Product Alerts     | Gmail Node                                  | Sends email alerts per product review   | 🧾 Split & Format Each Review     | 📊 Log to Google Sheet (Review History) | Alerts R&D team with product launch details via email.                                                        |
| 📊 Log to Google Sheet (Review History) | Google Sheets Node                         | Appends product reviews to Google Sheet | 📤 Email R&D: Product Alerts      | None                             | Maintains historical log of product launches for tracking and analysis.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it accordingly.**

2. **Add a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger every day at 7:00 AM (hour 7)  
   - Connect its output to the next node.

3. **Add a Set node to specify the scraping target:**  
   - Type: Set  
   - Add a field `url` of type string with value: `https://www.theverge.com/reviews`  
   - Connect input from schedule trigger node.  
   - Connect output to the AI agent node.

4. **Add a Langchain AI Agent node:**  
   - Type: Langchain Agent  
   - Prompt:  
     ```
     Use Bright Data Web Unlocker to scrape the latest product titles, release dates, and brief summaries from the following url.

     URL: {{ $json.url }}
     ```  
   - Enable output parser.  
   - Connect input from Set node.

5. **Configure sub-nodes inside the AI Agent:**  
   - Add a Langchain OpenAI Chat Model node:  
     - Model: GPT-4o-mini  
     - Connect as language model for the AI Agent node.  
     - Provide OpenAI API credentials.

   - Add MCP Client node:  
     - Tool name: `scrape_as_markdown`  
     - Operation: `executeTool`  
     - Tool parameters: pass dynamically from AI agent.  
     - Provide Bright Data MCP credentials.  
     - Connect as tool for the AI Agent.

   - Add another Langchain OpenAI Chat Model node (for parsing):  
     - Model: GPT-4o-mini  
     - Connect output to Structured Output Parser.

   - Add Structured Output Parser node:  
     - Provide example JSON schema with fields title, url, release_date, summary.  
     - Connect output to Auto-fixing Output Parser.

   - Add Auto-fixing Output Parser node:  
     - Connect its output back to the AI Agent node output.

6. **Add a Code node to split and format each review:**  
   - Write JavaScript code that maps each review in the input array to a new item, converting relative URLs to absolute by prefixing `https://www.theverge.com`.  
   - Connect input from AI Agent node output.

7. **Add Gmail node for email alerts:**  
   - Configure to send emails using OAuth2 Gmail credentials.  
   - Recipient: `shahkar.genai@gmail.com`  
   - Subject: Dynamic with product title.  
   - Message body: includes product title, release date, summary, and full review URL.  
   - Connect input from Code node output.

8. **Add Google Sheets node to log data:**  
   - Configure with proper Google Sheets OAuth2 credentials.  
   - Set document ID and sheet name targeting "Product Launches" sheet.  
   - Append mode with columns mapped: URL, Title, Summary, Release date.  
   - Connect input from Gmail node output.

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| I’ll receive a tiny commission if you join Bright Data through this link—thanks for fueling more free content!                         | https://get.brightdata.com/1tndi4600b25                                                                     |
| Workflow Assistance and Contact: For questions or support, contact Yaron at Yaron@nofluff.online. Explore video tutorials and tips.    | YouTube: https://www.youtube.com/@YaronBeen/videos LinkedIn: https://www.linkedin.com/in/yaronbeen/          |
| This workflow is a no-code/low-code solution combining AI and Bright Data to scrape dynamic competitor product launches legally.       | N/A                                                                                                         |
| The workflow is easily customizable by changing the scrape target URL or tweaking the AI instructions within the agent node prompt.  | N/A                                                                                                         |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated n8n workflow created with the n8n integration tool. This process fully complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.