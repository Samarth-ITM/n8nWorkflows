Monitor Competitor Financial Performance with Bright Data MCP, OpenAI & Google Sheets

https://n8nworkflows.xyz/workflows/monitor-competitor-financial-performance-with-bright-data-mcp--openai---google-sheets-5978


# Monitor Competitor Financial Performance with Bright Data MCP, OpenAI & Google Sheets

### 1. Workflow Overview

This workflow automates monitoring Tesla, Inc.'s financial performance by scraping its latest financial data from Yahoo Finance, comparing it against your company's financial metrics stored in Google Sheets, and then emailing the comparison results to your team. It targets use cases such as competitive financial benchmarking, automated reporting, and decision support for business strategy.

The workflow is logically divided into four main blocks:

- **1.1 Start & Input Reception:** Manual trigger and input of Tesla's Yahoo Finance URL.
- **1.2 AI-Powered Data Scraping:** AI agent invoking Bright Data MCP for scraping and OpenAI for data processing.
- **1.3 Financial Data Comparison:** Retrieval of your company's financial data from Google Sheets and comparison against Tesla's scraped data.
- **1.4 Email Notification:** Sending an automated email with the comparison results to the team.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Start & Input Reception

**Overview:**  
This block initiates the workflow manually and sets the URL for Tesla’s financial data on Yahoo Finance. It serves as the entry point and input provider for the rest of the workflow.

**Nodes Involved:**  
- `🚦 Start Workflow (Manual Trigger)`  
- `🔗 Enter Yahoo Finance URL for Tesla`

**Node Details:**

- **🚦 Start Workflow (Manual Trigger)**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual execution  
  - *Configuration:* No parameters; manual execution by user click  
  - *Inputs:* None  
  - *Outputs:* Triggers next node  
  - *Edge Cases:* Workflow will not start without manual trigger  

- **🔗 Enter Yahoo Finance URL for Tesla**  
  - *Type:* Set  
  - *Role:* Defines Tesla's Yahoo Finance URL as a workflow variable  
  - *Configuration:* Sets variable `teslaURL` to `https://finance.yahoo.com/quote/TSLA/financials`  
  - *Inputs:* Trigger from manual start  
  - *Outputs:* Passes URL to AI agent node  
  - *Edge Cases:* Hardcoded URL assumes Tesla’s page URL structure remains unchanged; requires update if Yahoo Finance changes URL schema  

---

#### 1.2 AI-Powered Data Scraping

**Overview:**  
This block uses an AI agent to scrape Tesla’s latest financial data from Yahoo Finance. It combines Langchain AI prompting with Bright Data MCP’s scraping tool and OpenAI language models for data extraction and structuring.

**Nodes Involved:**  
- `🤖 AI Agent: Scrape Tesla Financial Data`  
- `🧠 OpenAI Chat Model` (used inside agent)  
- `🌐 MCP Client Tool ` (used inside agent)  
- `📦 Format Financial data as Json Output`  
- `Auto-fixing Output Parser`

**Node Details:**

- **🤖 AI Agent: Scrape Tesla Financial Data**  
  - *Type:* Langchain AI Agent  
  - *Role:* Coordinates scraping process by issuing instructions to tools and parsing output  
  - *Configuration:*  
    - Prompt requests specific financial metrics (revenue, income, expenses, EPS, net margin, profit) from the Tesla Yahoo Finance URL (`{{$json.teslaURL}}`).  
    - Uses “define” prompt type to specify exact output format.  
    - Has output parser enabled to structure data.  
  - *Inputs:* Tesla Yahoo Finance URL from previous node  
  - *Outputs:* Parsed financial data JSON  
  - *Version Requirements:* Langchain agent v2 or higher  
  - *Edge Cases:* Potential failures if Yahoo Finance page layout changes; AI parsing errors if output format deviates; network or MCP tool failures.

- **🧠 OpenAI Chat Model**  
  - *Type:* Language Model (OpenAI GPT-4.1-mini)  
  - *Role:* Processes AI agent’s prompt, interprets scraped data text  
  - *Configuration:* Uses OpenAI GPT-4.1-mini model via authorized OpenAI API credentials  
  - *Inputs:* Prompt and URL from agent  
  - *Outputs:* Unstructured text response  
  - *Edge Cases:* API rate limits, authentication failures, unexpected response format  

- **🌐 MCP Client Tool**  
  - *Type:* Bright Data MCP Client  
  - *Role:* Executes web scraping tool “web_data_yahoo_finance_business” to retrieve raw financial data from Yahoo Finance  
  - *Configuration:* Tool parameters dynamically passed from AI agent’s instruction  
  - *Inputs:* Scraping command from AI agent  
  - *Outputs:* Raw scraped data for AI processing  
  - *Edge Cases:* Requires valid Bright Data MCP credentials; possible anti-bot measures or data changes on Yahoo Finance; tool execution errors  

- **📦 Format Financial data as Json Output**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Converts raw textual AI output into structured JSON matching financial data schema  
  - *Configuration:* Uses JSON schema example defining keys like `company`, `quarter_end_date`, `revenue`, `total_income`, `earnings_per_share`, etc.  
  - *Inputs:* AI chat model output  
  - *Outputs:* Structured JSON financial data  
  - *Edge Cases:* Parsing errors if output deviates from schema; invalid JSON  

- **Auto-fixing Output Parser**  
  - *Type:* Langchain Output Parser Autofixing  
  - *Role:* Automatically detects and corrects minor output format errors to ensure valid JSON  
  - *Inputs:* Output from structured parser  
  - *Outputs:* Cleaned and fixed JSON output for downstream nodes  
  - *Edge Cases:* Complex errors beyond autofix capability may cause failures  

---

#### 1.3 Financial Data Comparison

**Overview:**  
This block retrieves your company’s financial data from a Google Sheet and compares each metric against Tesla’s scraped data to determine whether Tesla is outperforming or underperforming.

**Nodes Involved:**  
- `📊 Get Company Data from Google Sheets`  
- `⚖️ Compare Data`

**Node Details:**

- **📊 Get Company Data from Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Reads your company’s financial data from a specified Google Sheet document and sheet (gid=0)  
  - *Configuration:*  
    - Document ID set to `1WvSWlWjowhz0amszpgv7CUBXjJHYZE9vuiVafgqG6tU`  
    - Sheet name: `Sheet1` (gid=0)  
    - OAuth2 credentials for Google Sheets API access  
  - *Inputs:* Output from AI agent (Tesla data)  
  - *Outputs:* Your company’s financial data as JSON  
  - *Edge Cases:* Google API authentication errors; empty or malformed sheet data; network timeouts  

- **⚖️ Compare Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Compares Tesla’s financial metrics against your company’s data field-by-field  
  - *Configuration:*  
    - Reads Tesla data from AI agent output JSON path `output.financial_data`  
    - Reads your company’s data from Google Sheets JSON  
    - Compares metrics: Total Revenue, Quarter Estimate, Pretax Income, Basic EPS, Diluted EPS, Net Income, Net Margin, Total Expenses  
    - Outputs comparison results with labels “Outperforming” or “Underperforming” based on numeric comparisons  
  - *Inputs:* Tesla financial data and your company’s data  
  - *Outputs:* JSON object summarizing comparison results for each metric  
  - *Edge Cases:* Parsing errors if any data fields are missing or malformed; string parsing issues on monetary values; type coercion errors  

---

#### 1.4 Email Notification

**Overview:**  
This block automatically sends an email to your team with the financial comparison results so they can review Tesla’s performance relative to your company.

**Nodes Involved:**  
- `📧 Send Financial Comparison to Team `

**Node Details:**

- **📧 Send Financial Comparison to Team**  
  - *Type:* Gmail  
  - *Role:* Sends an email with the comparison summary to a specified recipient email  
  - *Configuration:*  
    - Recipient: `shahkar.genai@gmail.com`  
    - Subject: `Financial Comparison: Your Company vs Tesla, Inc. (TSLA)`  
    - Email Body: Template with placeholders for each metric’s comparison result, e.g., `Total Revenue: {{ $json.comparison[0].result }}`  
    - Email type: Plain text  
    - OAuth2 Gmail credentials configured  
  - *Inputs:* Comparison JSON from the code node  
  - *Outputs:* Email send status  
  - *Edge Cases:* Gmail API limits or authentication errors; malformed email body if comparison JSON is missing fields  

---

### 3. Summary Table

| Node Name                           | Node Type                              | Functional Role                              | Input Node(s)                     | Output Node(s)                           | Sticky Note                                                                                                                       |
|-----------------------------------|--------------------------------------|----------------------------------------------|----------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| 🚦 Start Workflow (Manual Trigger) | Manual Trigger                       | Manual workflow start                         | None                             | 🔗 Enter Yahoo Finance URL for Tesla    | ### 🔹 **SECTION 1: Start & Input Financial Data URL**<br>Begin manual trigger and input Tesla URL.                              |
| 🔗 Enter Yahoo Finance URL for Tesla | Set                                 | Sets Tesla Yahoo Finance URL                   | 🚦 Start Workflow                 | 🤖 AI Agent: Scrape Tesla Financial Data |                                                                                                                                |
| 🤖 AI Agent: Scrape Tesla Financial Data | Langchain AI Agent                  | Scrapes and structures Tesla financial data  | 🔗 Enter Yahoo Finance URL        | 📊 Get Company Data from Google Sheets   | ### 🤖 **SECTION 2: AI Agent Fetching Financial Data**<br>AI agent uses Bright Data MCP to scrape and OpenAI to parse data.      |
| 🧠 OpenAI Chat Model               | OpenAI Language Model                | Processes AI scraping prompt                   | Inside AI Agent                  | Inside AI Agent                         |                                                                                                                                |
| 🌐 MCP Client Tool                | Bright Data MCP Client               | Executes protected web scraping                | Inside AI Agent                  | Inside AI Agent                         |                                                                                                                                |
| 📦 Format Financial data as Json Output | Langchain Structured Output Parser | Structures AI output into JSON                  | Inside AI Agent                  | Auto-fixing Output Parser                |                                                                                                                                |
| Auto-fixing Output Parser         | Langchain Autofixing Output Parser  | Fixes JSON formatting issues                    | 📦 Format Financial data as Json Output | 🤖 AI Agent: Scrape Tesla Financial Data |                                                                                                                                |
| 📊 Get Company Data from Google Sheets | Google Sheets                      | Fetches your company’s financial data          | 🤖 AI Agent: Scrape Tesla Financial Data | ⚖️ Compare Data                          | ### 📊 **SECTION 3: Compare Tesla vs Your Company’s Financial Data**<br>Fetch your data and compare metrics with Tesla.          |
| ⚖️ Compare Data                   | Code (JavaScript)                   | Compares Tesla and your company’s financials  | 📊 Get Company Data from Google Sheets | 📧 Send Financial Comparison to Team    |                                                                                                                                |
| 📧 Send Financial Comparison to Team  | Gmail                              | Sends comparison email to team                  | ⚖️ Compare Data                  | None                                    | ### 📧 **SECTION 4: Send Comparison Email to Team**<br>Automatically emails the team with the financial comparison results.      |
| Sticky Note                      | Sticky Note                        | Informational notes on workflow sections       | None                             | None                                    | Contains detailed descriptions for each workflow section and tips for users.                                                   |
| Sticky Note1                     | Sticky Note                        | Detailed explanation of AI Agent block         | None                             | None                                    |                                                                                                                                |
| Sticky Note2                     | Sticky Note                        | Detailed explanation of comparison block       | None                             | None                                    |                                                                                                                                |
| Sticky Note3                     | Sticky Note                        | Explanation of email notification block        | None                             | None                                    |                                                                                                                                |
| Sticky Note4                     | Sticky Note                        | Full workflow summary and use cases            | None                             | None                                    |                                                                                                                                |
| Sticky Note5                     | Sticky Note                        | Affiliate link for Bright Data                  | None                             | None                                    | Contains affiliate Bright Data link: https://get.brightdata.com/1tndi4600b25                                                      |
| Sticky Note9                     | Sticky Note                        | Support contact and resource links              | None                             | None                                    | Support contact: Yaron@nofluff.online; YouTube and LinkedIn links included                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Entry point for manual execution  
   - No parameters needed.

2. **Add Set Node to Define Tesla Yahoo Finance URL**  
   - Type: Set  
   - Parameter: Add string variable named `teslaURL` with value `https://finance.yahoo.com/quote/TSLA/financials`  
   - Connect: Manual Trigger → Set Node

3. **Add Langchain AI Agent Node for Scraping**  
   - Type: Langchain Agent (Langchain agent v2+)  
   - Prompt:  
     ```
     Please scrape the latest financial data for Tesla, Inc. (TSLA) from Yahoo Finance for the latest quarter available.

     Search URL : {{ $json.teslaURL }}

     Include the following information:
     - Revenue for the latest quarter.
     - Total Income
     - Total Expenses
     - Earnings per Share (EPS) for the latest quarter.
     - Net margin for the latest quarter.
     - Profit (Net Income) for the latest quarter.

     Please provide a detailed breakdown of only these above financial metrics nothing else.
     ```
   - Enable output parser  
   - Connect: Set Node → AI Agent Node

4. **Configure AI Agent’s Sub-nodes:**

   a. **OpenAI Chat Model Node**  
      - Use OpenAI GPT-4.1-mini model  
      - Provide OpenAI API credentials  
      - Connect as AI language model inside AI Agent  

   b. **Bright Data MCP Client Tool Node**  
      - Tool Name: `web_data_yahoo_finance_business`  
      - Operation: `executeTool`  
      - Parameters: Dynamic from AI agent prompt  
      - Provide Bright Data MCP API credentials  
      - Connect as AI tool inside AI Agent  

   c. **Structured Output Parser Node**  
      - Configure JSON schema example for financial data:  
        ```json
        {
          "financial_data": {
            "company": "Tesla, Inc.",
            "quarter_end_date": "2024-09-30",
            "revenue": {
              "total_revenue": "$97.69 billion (annualized)",
              "quarter_value_estimate": "$24.42 billion (approx.)"
            },
            "total_income": {
              "pretax_income": "$8.99 billion"
            },
            "total_expenses": {
              "total_expenses": "$89.93 billion (annualized)"
            },
            "earnings_per_share": {
              "basic_eps": "$0.68",
              "diluted_eps": "$0.62"
            },
            "profit": {
              "net_income": "$7.13 billion"
            },
            "margins": {
              "net_margin": "6.38% (annualized)"
            }
          }
        }
        ```
      - Connect AI Chat Model → Structured Output Parser  

   d. **Autofixing Output Parser Node**  
      - Enable autofixing to correct minor JSON formatting errors  
      - Connect Structured Output Parser → Autofixing Output Parser  

   e. Connect Autofixing Output Parser → AI Agent main output

5. **Add Google Sheets Node to Retrieve Your Company’s Financial Data**  
   - Type: Google Sheets  
   - Parameters:  
     - Document ID: Your Google Sheet ID containing your company’s financial data  
     - Sheet Name: `Sheet1` or relevant sheet  
   - Configure Google Sheets OAuth2 credentials  
   - Connect AI Agent output → Google Sheets Node

6. **Add Code Node to Compare Tesla vs Your Company**  
   - Type: Code (JavaScript)  
   - Paste comparison script that:  
     - Reads Tesla financial data from AI Agent output JSON path  
     - Reads your company’s data from Google Sheets output  
     - Compares eight financial metrics  
     - Outputs JSON with “Outperforming” or “Underperforming” results  
   - Connect Google Sheets Node → Code Node

7. **Add Gmail Node to Send Email**  
   - Type: Gmail  
   - Parameters:  
     - Recipient: your team email (example: `shahkar.genai@gmail.com`)  
     - Subject: `Financial Comparison: Your Company vs Tesla, Inc. (TSLA)`  
     - Message Body: Use template with placeholders for each comparison result  
     - Email Type: Plain text  
   - Configure Gmail OAuth2 credentials  
   - Connect Code Node → Gmail Node

8. **Final Checks:**  
   - Ensure all credentials are valid and authorized  
   - Test manual trigger to start workflow  
   - Validate that Tesla data scrapes correctly and is parsed  
   - Verify Google Sheets data format matches expected fields  
   - Confirm email is sent with correct comparison results

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow automates Tesla financial data scraping, comparison, and reporting for competitive analysis. | Workflow description and use cases.                                                                  |
| For support or questions, contact Yaron at Yaron@nofluff.online                                   | Support contact email.                                                                                |
| Explore video tutorials and tips on YouTube: https://www.youtube.com/@YaronBeen/videos            | YouTube resource link.                                                                                |
| LinkedIn profile for additional insights: https://www.linkedin.com/in/yaronbeen/                  | LinkedIn professional profile.                                                                       |
| Bright Data affiliate link for MCP service: https://get.brightdata.com/1tndi4600b25                | Affiliate referral link for Bright Data MCP client service.                                          |
| The workflow includes detailed sticky notes for beginner tips and section explanations.           | Embedded sticky notes within workflow for user guidance and context.                                 |

---

**Disclaimer:** The provided text is exclusively from an n8n automated workflow. It complies with all current content policies and does not contain illegal or protected content. All data handled is legal and publicly available.