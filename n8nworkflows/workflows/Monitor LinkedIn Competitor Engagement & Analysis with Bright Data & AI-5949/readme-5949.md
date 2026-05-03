Monitor LinkedIn Competitor Engagement & Analysis with Bright Data & AI

https://n8nworkflows.xyz/workflows/monitor-linkedin-competitor-engagement---analysis-with-bright-data---ai-5949


# Monitor LinkedIn Competitor Engagement & Analysis with Bright Data & AI

### 1. Workflow Overview

This workflow automates the monitoring and analysis of LinkedIn competitor engagement by scraping recent posts from a specified LinkedIn company profile, analyzing engagement metrics (likes, comments), and storing both summarized and detailed post data into Google Sheets. It uses a combination of AI-powered scraping (via Bright Data MCP and OpenAI) and JavaScript processing to deliver actionable insights without requiring manual data extraction or coding.

The workflow is logically divided into four main blocks:

- **1.1 Start & Input:** User triggers the workflow and defines the LinkedIn company profile URL.
- **1.2 Smart Scraper Agent:** An AI-powered agent fetches the latest 5 LinkedIn posts using Bright Data proxies to bypass anti-bot measures.
- **1.3 Analyze & Save Metrics:** Processes the scraped posts to calculate engagement averages and saves the summary to Google Sheets.
- **1.4 Format & Store Full Posts:** Formats detailed post data and saves each post individually to a second Google Sheet for deeper insights.

---

### 2. Block-by-Block Analysis

#### 1.1 Start & Input

**Overview:**  
Initial trigger and input setup block. It starts the workflow manually and sets the LinkedIn company URL to scrape.

**Nodes Involved:**  
- `🔘 Trigger: Manual Start`  
- `🔗 Set LinkedIn Company URL`

**Node Details:**

- **`🔘 Trigger: Manual Start`**  
  - Type: Manual Trigger (Initiates workflow execution manually)  
  - Configuration: No parameters, just a manual start button.  
  - Inputs: None  
  - Outputs: Connects to `🔗 Set LinkedIn Company URL`  
  - Potential Failures: None (manual start)  

- **`🔗 Set LinkedIn Company URL`**  
  - Type: Set (Assigns fixed values to variables)  
  - Configuration: Sets a string variable `URL` with the LinkedIn company page URL (default example: `https://www.linkedin.com/company/hubspot/`).  
  - Inputs: From manual trigger  
  - Outputs: To `🤖 Agent: Fetch LinkedIn Posts (via MCP Tool)`  
  - Edge Cases: User must input a valid public LinkedIn company URL; invalid or private URLs may cause scraping failures.  

---

#### 1.2 Smart Scraper Agent

**Overview:**  
Uses an AI-powered agent to scrape the latest 5 posts from the LinkedIn company profile. The agent combines natural language understanding with the Bright Data MCP proxy client to bypass LinkedIn’s anti-bot protections and extract structured post data.

**Nodes Involved:**  
- `🤖 Agent: Fetch LinkedIn Posts (via MCP Tool)`  
- `OpenAI Chat Model` (sub-node of agent)  
- `🌐 Bright Data MCP Client` (sub-node of agent)  
- `Auto-fixing Output Parser` (sub-node of agent)  
- `OpenAI Chat Model1` (sub-node for parsing)  
- `Structured Output Parser` (sub-node for JSON structuring)

**Node Details:**

- **`🤖 Agent: Fetch LinkedIn Posts (via MCP Tool)`**  
  - Type: LangChain Agent (AI agent to perform complex tasks)  
  - Configuration: Prompt instructs the AI to scrape the specified LinkedIn URL for the latest 5 posts. Uses natural language input with embedded URL variable.  
  - Inputs: Receives `URL` from previous Set node  
  - Outputs: Provides raw scraped post data array  
  - Credentials: Uses MCP Client API account (Bright Data) to execute proxy scraping tool.  
  - Edge Cases:  
    - Potential proxy or network errors from Bright Data MCP.  
    - LinkedIn rate limiting or changes in page structure may cause scraping issues.  
    - AI parsing failures if response format changes.  

- **`OpenAI Chat Model` and `OpenAI Chat Model1`**  
  - Type: OpenAI GPT-4o-mini (Chat Language Models)  
  - Configuration: Used inside the agent to interpret scraping instructions and parse results.  
  - Credentials: OpenAI API account  
  - Edge Cases: API quota limits, network timeouts, or model response errors.  

- **`🌐 Bright Data MCP Client`**  
  - Type: MCP Client Tool (Executes scraping through Bright Data Mobile Proxies)  
  - Configuration: Executes LinkedIn company profile scraping tool with parameters from AI.  
  - Credentials: MCP Client API credentials  
  - Edge Cases: Authentication failures, proxy exhaustion, or LinkedIn blocking.  

- **`Auto-fixing Output Parser` & `Structured Output Parser`**  
  - Type: LangChain Output Parsers  
  - Configuration: Parses AI-generated output into structured JSON arrays representing posts with fields like date, likes, comments, content, post links, and videos.  
  - Edge Cases: Parsing errors if output deviates from expected schema.  

---

#### 1.3 Analyze & Save Metrics

**Overview:**  
Processes the scraped post data to calculate total and average engagement metrics such as likes and comments, then saves the summarized numbers to a designated Google Sheet.

**Nodes Involved:**  
- `📈 Analyze Engagement Metrics`  
- `📥 Save Averages to Google Sheets`

**Node Details:**

- **`📈 Analyze Engagement Metrics`**  
  - Type: Code (JavaScript)  
  - Configuration: Uses JavaScript to iterate over the posts array, summing likes and comments, then calculating averages. Returns a JSON object with total posts, total likes, total comments, average likes, and average comments.  
  - Inputs: Output from `🤖 Agent: Fetch LinkedIn Posts (via MCP Tool)`  
  - Outputs: Metrics summary JSON  
  - Edge Cases: No posts returned (division by zero), missing likes/comments fields in posts.  

- **`📥 Save Averages to Google Sheets`**  
  - Type: Google Sheets (Append Operation)  
  - Configuration: Appends the calculated metrics to a Google Sheet with columns for total posts, likes, comments, and averages. Uses OAuth2 credentials.  
  - Inputs: From metrics code node  
  - Outputs: To `🧾 Format Post Content`  
  - Edge Cases: Authentication failure, API quota exceeded, sheet access denied, or schema mismatch.  

---

#### 1.4 Format & Store Full Posts

**Overview:**  
Formats each individual post from the scraped data for clarity and appends each as a separate row in a second Google Sheet, providing detailed historical records of competitor posts.

**Nodes Involved:**  
- `🧾 Format Post Content`  
- `📥 Save Posts to Google Sheets`

**Node Details:**

- **`🧾 Format Post Content`**  
  - Type: Code (JavaScript)  
  - Configuration: Extracts the array of post objects from the agent output and maps each post to a separate item to conform with n8n’s data structure for batch processing.  
  - Inputs: From `📥 Save Averages to Google Sheets` (via connection chain)  
  - Outputs: An array of post items  
  - Edge Cases: Empty or malformed post arrays, missing fields.  

- **`📥 Save Posts to Google Sheets`**  
  - Type: Google Sheets (Append Operation)  
  - Configuration: Appends detailed post data fields (date, likes, content, comments, post link, videos, competitor name, post title) to a separate Google Sheet dedicated to full posts. Uses OAuth2 credentials.  
  - Inputs: From formatting code node  
  - Edge Cases: Authentication issues, API limits, incorrect data types, or sheet access problems.  

---

### 3. Summary Table

| Node Name                                 | Node Type                              | Functional Role                        | Input Node(s)                           | Output Node(s)                           | Sticky Note                                   |
|-------------------------------------------|--------------------------------------|--------------------------------------|----------------------------------------|------------------------------------------|-----------------------------------------------|
| 🔘 Trigger: Manual Start                   | Manual Trigger                       | Starts workflow manually              | None                                   | 🔗 Set LinkedIn Company URL               | Section 1: Start & Input                       |
| 🔗 Set LinkedIn Company URL                | Set                                  | Sets LinkedIn company URL             | 🔘 Trigger: Manual Start                | 🤖 Agent: Fetch LinkedIn Posts (via MCP) | Section 1: Start & Input                       |
| 🤖 Agent: Fetch LinkedIn Posts (via MCP)  | LangChain Agent                      | Scrapes LinkedIn posts via AI & MCP  | 🔗 Set LinkedIn Company URL              | 📈 Analyze Engagement Metrics             | Section 2: Smart Scraper Agent                 |
| OpenAI Chat Model                          | OpenAI Chat Model                    | AI language understanding             | — (internal to Agent)                   | 🤖 Agent: Fetch LinkedIn Posts (via MCP)  | Section 2: Smart Scraper Agent                 |
| 🌐 Bright Data MCP Client                  | MCP Client Tool                     | Executes proxy scraping tool          | — (internal to Agent)                   | 🤖 Agent: Fetch LinkedIn Posts (via MCP)  | Section 2: Smart Scraper Agent                 |
| Auto-fixing Output Parser                  | Output Parser                       | Parses AI output                      | OpenAI Chat Model1                      | 🤖 Agent: Fetch LinkedIn Posts (via MCP)  | Section 2: Smart Scraper Agent                 |
| OpenAI Chat Model1                         | OpenAI Chat Model                   | Parses output to structured data      | Structured Output Parser                | Auto-fixing Output Parser                 | Section 2: Smart Scraper Agent                 |
| Structured Output Parser                   | Output Parser                       | Structures JSON output                | Auto-fixing Output Parser               | OpenAI Chat Model1                        | Section 2: Smart Scraper Agent                 |
| 📈 Analyze Engagement Metrics              | Code                                | Calculates engagement averages        | 🤖 Agent: Fetch LinkedIn Posts (via MCP) | 📥 Save Averages to Google Sheets         | Section 3: Analyze & Save Metrics              |
| 📥 Save Averages to Google Sheets          | Google Sheets                      | Saves engagement summary to sheet    | 📈 Analyze Engagement Metrics           | 🧾 Format Post Content                    | Section 3: Analyze & Save Metrics              |
| 🧾 Format Post Content                      | Code                                | Formats full post details             | 📥 Save Averages to Google Sheets       | 📥 Save Posts to Google Sheets             | Section 4: Format & Store Full Posts           |
| 📥 Save Posts to Google Sheets              | Google Sheets                      | Stores detailed posts in sheet       | 🧾 Format Post Content                   | None                                     | Section 4: Format & Store Full Posts           |
| Sticky Note (multiple notes across flow)   | Sticky Note                        | Documentation and tips                | N/A                                    | N/A                                      | See section 5 for details                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Manual Trigger** node named `🔘 Trigger: Manual Start`. No parameters needed.

2. **Set LinkedIn Company URL:**  
   - Add a **Set** node named `🔗 Set LinkedIn Company URL`.  
   - Add a string field named `URL` with a default value (e.g., `https://www.linkedin.com/company/hubspot/`).  
   - Connect the manual trigger output to this node.

3. **Configure the Smart Scraper Agent:**  
   - Add a **LangChain Agent** node named `🤖 Agent: Fetch LinkedIn Posts (via MCP Tool)`.  
   - Set the prompt text to: `"Scrape the below user profile on LinkedIn and get the latest 5 post data:\n{{ $json.URL }}"`.  
   - Use the `define` prompt type with output parser enabled.  
   - Connect the output of the Set URL node to this agent node.  
   - Configure credentials for **Bright Data MCP Client** (MCP Client API credentials).  
   - Inside the agent, configure sub-nodes:  
     - **OpenAI Chat Model** (model: `gpt-4o-mini`) with OpenAI API credentials.  
     - **Bright Data MCP Client** node configured to run the `web_data_linkedin_company_profile` tool.  
     - **Auto-fixing Output Parser** and **Structured Output Parser** nodes to parse the AI output into structured post data.

4. **Add Code Node to Analyze Engagement Metrics:**  
   - Add a **Code** node named `📈 Analyze Engagement Metrics`.  
   - Paste the JavaScript code that sums likes and comments, calculates averages, and outputs JSON with totals and averages.  
   - Connect the output of the Agent node to this node.

5. **Save Averages to Google Sheets:**  
   - Add a **Google Sheets** node named `📥 Save Averages to Google Sheets`.  
   - Set operation to `append`.  
   - Configure the mapping with columns: Total posts, Total likes, Total comments, Average likes, Average comments, linked to the respective fields from the previous node’s JSON.  
   - Connect the output of the Analysis node to this node.  
   - Set up OAuth2 credentials for Google Sheets account.  
   - Specify the target spreadsheet ID and sheet name (e.g., Spreadsheet with ID `1FzzslRBkdEgz14zuyy1J5IAMAHgkmdmiYiEqYG9AnOI` and `gid=0`).

6. **Format Post Content:**  
   - Add a **Code** node named `🧾 Format Post Content`.  
   - Use JavaScript code that extracts the posts array from the Agent output and maps each post to a separate item.  
   - Connect output from the Google Sheets averages save node here.

7. **Save Full Posts to Google Sheets:**  
   - Add a **Google Sheets** node named `📥 Save Posts to Google Sheets`.  
   - Operation: `append`.  
   - Map fields: date, likes, video (videos array), content, comments, post link, Competitor, Post title.  
   - Connect output from the formatting code node here.  
   - Use the same or another Google Sheets OAuth2 credential.  
   - Specify spreadsheet ID (e.g., `1R-rAkvVh1lhbPrFsxuBUXNMFFFgsyIWzS4OritRiHLU`) and appropriate sheet.

8. **Final Connections:**  
   - Connect the nodes in sequence: Trigger → Set URL → Agent → Analyze Metrics → Save Averages → Format Posts → Save Posts.

9. **Credentials Setup:**  
   - Configure OpenAI API credentials for all OpenAI nodes (`lmChatOpenAi`).  
   - Configure Bright Data MCP Client credentials for MCP Client node.  
   - Configure Google Sheets OAuth2 credentials for both Google Sheets nodes.

10. **Test the Workflow:**  
    - Manually trigger the workflow.  
    - Verify that the LinkedIn URL is correct and publicly accessible.  
    - Check Google Sheets for appended data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow assistance contact: Yaron@nofluff.online                                                                                                            | Support contact email                                                                           |
| YouTube tutorials and tips: https://www.youtube.com/@YaronBeen/videos                                                                                        | Video resources                                                                                |
| LinkedIn profile of creator: https://www.linkedin.com/in/yaronbeen/                                                                                          | Creator’s professional profile                                                                |
| Bright Data affiliate link for proxy service with commission: https://get.brightdata.com/1tndi4600b25                                                        | Proxy service used for LinkedIn scraping                                                      |
| Workflow branding and detailed description including beginner tips provided in multiple sticky notes in the workflow                                        | See sticky notes content in the workflow JSON                                                  |
| Use case highlights: Competitive monitoring, marketing analytics, client reports, content strategy planning                                                  | Business applications                                                                          |

---

**Disclaimer:**  
This document is based exclusively on an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected content. All data processed is legal and publicly available.