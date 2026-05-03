Create a Google Analytics Data Report with AI and sent it to E-Mail and Telegram

https://n8nworkflows.xyz/workflows/create-a-google-analytics-data-report-with-ai-and-sent-it-to-e-mail-and-telegram-2673


# Create a Google Analytics Data Report with AI and sent it to E-Mail and Telegram

### 1. Workflow Overview

This workflow automates the generation and distribution of a weekly Google Analytics report enriched by AI analysis and delivered via email and Telegram. It targets users who want a comparative summary of website metrics for the last 7 days versus the same period in the previous year.

The workflow consists of these logical blocks:

- **1.1 Time Trigger:** Initiates the workflow weekly (e.g., Monday 7 a.m.).
- **1.2 Current Period Data Retrieval:** Fetches Google Analytics data for the last 7 days.
- **1.3 Current Period Data Assignment & Summarization:** Normalizes and summarizes the recent data.
- **1.4 Previous Year Period Calculation:** Computes date range for the same 7-day period last year.
- **1.5 Previous Year Data Retrieval and Processing:** Fetches, assigns, and summarizes last year’s data.
- **1.6 AI Processing for Email Report:** Formats, analyzes, and summarizes data into an HTML email report.
- **1.7 Email Dispatch:** Sends the AI-generated report via SMTP.
- **1.8 AI Processing for Telegram Summary:** Converts the email report into a concise plaintext summary suitable for Telegram.
- **1.9 Telegram Message Dispatch:** Sends the summarized report to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Time Trigger

- **Overview:** Starts the workflow automatically every week on Monday at 7 a.m.
- **Nodes Involved:** Schedule Trigger
- **Node Details:**
  - Type: Schedule Trigger
  - Configuration: Weekly interval, triggers every Monday at 7:00 AM
  - Inputs: None (trigger node)
  - Outputs: Triggers Google Analytics data retrieval for the last 7 days
  - Edge Cases: Workflow will not run if n8n instance is down at scheduled time; no retry logic here.
  - Version: 1.2

#### 1.2 Current Period Data Retrieval

- **Overview:** Retrieves Google Analytics 4 metrics for the last 7 days for a specific GA property.
- **Nodes Involved:** Google Analytics Letzte 7 Tage
- **Node Details:**
  - Type: Google Analytics (GA4)
  - Configuration:
    - Property ID: 345060083 (example property)
    - Metrics: screenPageViews, sessions, sessionsPerUser, averageSessionDuration, ecommercePurchases, averagePurchaseRevenue, purchaseRevenue
    - Dimensions: none specified (empty)
  - Credentials: Google Analytics OAuth2
  - Input: Trigger from Schedule Trigger
  - Output: Raw GA data for last 7 days
  - Edge Cases: API auth errors, quota limits, no data returned, network timeouts
  - Version: 2

#### 1.3 Current Period Data Assignment & Summarization

- **Overview:** Maps raw GA metrics to normalized field names and summarizes them (sums, averages).
- **Nodes Involved:** Assign data, Summarize Data
- **Node Details:**
  - **Assign data:**
    - Type: Set node
    - Function: Extracts and renames GA metrics into standard field names in German (e.g., Aufrufe = screenPageViews)
    - Key Expressions: Uses expressions to map JSON fields from GA response
    - Input: GA last 7 days node
    - Output: Structured JSON with fields like Aufrufe, Nutzer, Sitzungen, etc.
    - Edge Cases: Missing or malformed GA data could cause expression failures
  - **Summarize Data:**
    - Type: Summarize node
    - Function: Aggregates numeric fields (sum or average as appropriate)
    - Fields Summarized: Aufrufe, Nutzer, Sitzungen, Sitzungen pro Nutzer, Sitzungsdauer, Käufe, Revenue pro Kauf, Revenue, date
    - Input: Output from Assign data
    - Output: Aggregated summary statistics
    - Edge Cases: Empty input data leads to empty summary; aggregation on missing fields

#### 1.4 Previous Year Period Calculation

- **Overview:** Calculates start and end dates for the same 7-day period one year ago.
- **Nodes Involved:** Calculation same period previous year
- **Node Details:**
  - Type: Code node (JavaScript)
  - Function: Calculates ISO date strings for startDate and endDate by subtracting one year and adjusting 7 days back for start
  - Input: Output of Summarize Data (used for date reference)
  - Output: JSON object with startDate and endDate for last year's period
  - Edge Cases: Date calculation errors if system clock misconfigured; no input validation
  - Version: 2

#### 1.5 Previous Year Data Retrieval and Processing

- **Overview:** Fetches Google Analytics data for the previous year’s same period, assigns and summarizes it similarly to current period data.
- **Nodes Involved:** Google Analytics: Past 7 days of the previous year, Assign data1, Summarize Data1
- **Node Details:**
  - **Google Analytics: Past 7 days of the previous year:**
    - Type: Google Analytics (GA4)
    - Configuration:
      - Property ID: 345060083
      - Date range: Custom, with startDate and endDate from previous node
      - Metrics and dimensions same as current period node
    - Credentials: Google Analytics OAuth2
    - Input: Calculation same period previous year
    - Output: GA data for previous year’s period
    - Edge Cases: Same as current period GA node
  - **Assign data1:**
    - Type: Set node
    - Function: Maps GA metrics to normalized fields (same as Assign data)
    - Input: GA previous year node
    - Output: Structured JSON for previous year data
  - **Summarize Data1:**
    - Type: Summarize node
    - Function: Aggregates numeric fields (sum/average)
    - Input: Output from Assign data1
    - Output: Aggregated summary statistics for previous year
    - Edge Cases: Same as Summarize Data

#### 1.6 AI Processing for Email Report

- **Overview:** Uses AI (OpenAI GPT-4O) to analyze summarized data, compute percentage changes, and generate a polished HTML report including a short summary and a metrics table.
- **Nodes Involved:** Processing for email
- **Node Details:**
  - Type: OpenAI node (Langchain integration)
  - Model: GPT-4O
  - Prompt: Takes summarized numeric data from Summarize Data and Summarize Data1 nodes, formats a markdown table with metrics, last 7 days, previous year, and percentage change columns, plus a short textual summary.
  - Output: HTML-formatted report suitable for email body
  - Input: Summarize Data1 (previous year summary) and Summarize Data (current summary)
  - Output: JSON with `message.content` containing HTML report
  - Edge Cases: AI API failures, rate limits, prompt formatting errors, unexpected AI output
  - Version: 1.7

#### 1.7 Email Dispatch

- **Overview:** Sends the AI-generated HTML report as an email.
- **Nodes Involved:** Send Email
- **Node Details:**
  - Type: Email Send node
  - Configuration:
    - To: friedemann.schuetz@ep-reisen.de (example)
    - From: friedemann.schuetz@posteo.de
    - Subject: Weekly Report: Google Analytics: Last 7 days
    - Body: HTML content from AI node (`message.content`)
  - Credentials: SMTP account
  - Input: Processing for email node
  - Output: None (terminal node for email dispatch)
  - Edge Cases: SMTP authentication failure, connectivity issues, invalid email addresses
  - Version: 2.1

#### 1.8 AI Processing for Telegram Summary

- **Overview:** Converts the detailed HTML email report into a simplified plaintext summary formatted for Telegram, with one metric per paragraph.
- **Nodes Involved:** Processing for Telegram
- **Node Details:**
  - Type: OpenAI node (Langchain integration)
  - Model: GPT-4O-MINI
  - Prompt: Converts HTML to plaintext, reformats table metrics as separate paragraphs, e.g. "Total views: xx", with differences and percentages.
  - Input: Processing for email node output (`message.content`)
  - Output: Plaintext message content for Telegram
  - Credentials: OpenAI API
  - Edge Cases: AI conversion errors, malformed HTML input, API limits
  - Version: 1.7

#### 1.9 Telegram Message Dispatch

- **Overview:** Sends the plaintext summary to a specified Telegram chat.
- **Nodes Involved:** Telegram
- **Node Details:**
  - Type: Telegram node
  - Configuration:
    - Chat ID: 1810565648
    - Message text: From Processing for Telegram node (`message.content`)
  - Credentials: Telegram API
  - Input: Processing for Telegram node
  - Output: None (terminal node)
  - Edge Cases: Invalid chat ID, Telegram API rate limits, connectivity issues
  - Version: 1.2

---

### 3. Summary Table

| Node Name                            | Node Type                         | Functional Role                          | Input Node(s)                     | Output Node(s)                              | Sticky Note                                                                                                                              |
|------------------------------------|----------------------------------|----------------------------------------|----------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                   | Schedule Trigger                 | Initiates weekly workflow trigger      | None                             | Google Analytics Letzte 7 Tage               | Welcome to my Google Analytics Weekly Report Workflow! This workflow has the following sequence: (See full note in workflow)              |
| Google Analytics Letzte 7 Tage     | Google Analytics (GA4)           | Fetches current 7 days GA data         | Schedule Trigger                 | Assign data                                 | See above                                                                                                                               |
| Assign data                       | Set                             | Normalizes current GA data fields      | Google Analytics Letzte 7 Tage   | Summarize Data                              | See above                                                                                                                               |
| Summarize Data                   | Summarize                       | Aggregates current period data         | Assign data                     | Calculation same period previous year       | See above                                                                                                                               |
| Calculation same period previous year | Code                           | Computes last year’s date range        | Summarize Data                 | Google Analytics: Past 7 days of the previous year | See above                                                                                                                               |
| Google Analytics: Past 7 days of the previous year | Google Analytics (GA4)           | Fetches previous year GA data          | Calculation same period previous year | Assign data1                                | See above                                                                                                                               |
| Assign data1                      | Set                             | Normalizes previous year GA data       | Google Analytics: Past 7 days of the previous year | Summarize Data1                             | See above                                                                                                                               |
| Summarize Data1                  | Summarize                       | Aggregates previous year data          | Assign data1                   | Processing for email                        | See above                                                                                                                               |
| Processing for email             | OpenAI (Langchain)              | AI analysis and HTML report generation | Summarize Data1, Summarize Data | Send Email, Processing for Telegram         | See above                                                                                                                               |
| Send Email                      | Email Send                     | Sends the HTML email report            | Processing for email            | None                                        | See above                                                                                                                               |
| Processing for Telegram          | OpenAI (Langchain)              | Converts HTML report to Telegram text  | Processing for email            | Telegram                                    | See above                                                                                                                               |
| Telegram                       | Telegram                       | Sends report summary to Telegram chat | Processing for Telegram         | None                                        | See above                                                                                                                               |
| Sticky Note                    | Sticky Note                   | Documentation and workflow overview    | None                          | None                                        | Welcome to my Google Analytics Weekly Report Workflow! This workflow has the following sequence: (See full note)                       |
| Calculator                    | AI Tool (Calculator)           | Available AI calculator tool (not actively used) | Processing for email (ai_tool connection) | None                                        | None                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Type: Schedule Trigger
   - Configure to run weekly on Monday at 7 a.m.

2. **Add Google Analytics node for current period:**
   - Type: Google Analytics (GA4)
   - Credentials: Google Analytics OAuth2 (configure with appropriate account)
   - Set Property ID to your GA4 property (e.g., 345060083)
   - Metrics: screenPageViews, sessions, sessionsPerUser, averageSessionDuration, ecommercePurchases, averagePurchaseRevenue, purchaseRevenue
   - Dimensions: none
   - Connect Schedule Trigger output to this node

3. **Add Set node to assign current period data:**
   - Type: Set
   - Map GA output fields to normalized fields:
     - Aufrufe = screenPageViews
     - Nutzer = totalUsers
     - Sitzungen = sessions
     - Sitzungen pro Nutzer = sessionsPerUser
     - Sitzungsdauer = averageSessionDuration
     - Käufe = ecommercePurchases
     - Revenue pro Kauf = averagePurchaseRevenue
     - Revenue = purchaseRevenue
     - date = date (string)
   - Connect Google Analytics current period node output to this node

4. **Add Summarize node to aggregate current data:**
   - Type: Summarize
   - Configure fields to summarize:
     - Sum: Aufrufe, Nutzer, Sitzungen, Käufe, Revenue
     - Average: Sitzungen pro Nutzer, Sitzungsdauer, Revenue pro Kauf
     - Include date field as is
   - Connect Set node output to this node

5. **Add Code node to calculate previous year date range:**
   - Type: Code (JavaScript)
   - Use JavaScript to calculate startDate and endDate as ISO strings for same 7-day period last year:
     ```js
     return {
       startDate: (() => {
         const date = new Date();
         date.setFullYear(date.getFullYear() - 1);
         date.setDate(date.getDate() - 7);
         return date.toISOString().split('T')[0];
       })(),
       endDate: (() => {
         const date = new Date();
         date.setFullYear(date.getFullYear() - 1);
         return date.toISOString().split('T')[0];
       })(),
     };
     ```
   - Connect Summarize current data node output to this node

6. **Add Google Analytics node for previous year data:**
   - Type: Google Analytics (GA4)
   - Credentials: Google Analytics OAuth2
   - Property ID: same as current period
   - Date Range: Custom (use startDate and endDate from previous node output)
   - Metrics and dimensions same as current period node
   - Connect Code node output to this node

7. **Add Set node to assign previous year data:**
   - Same mapping as current period Set node
   - Connect previous year GA node output to this node

8. **Add Summarize node to aggregate previous year data:**
   - Same configuration as current period Summarize node
   - Connect previous year Set node output to this node

9. **Add OpenAI (Langchain) node for email report generation:**
   - Type: OpenAI (Langchain)
   - Credentials: OpenAI API
   - Model: GPT-4O
   - Prompt: Provide a prompt to analyze summarized data from both periods and output an HTML formatted report with:
     - Short summary (max 3 sentences)
     - Table comparing metrics for last 7 days and previous year with percentage changes
     - Format numbers with dot as thousands separator, comma for decimals, convert session duration to minutes, currency for revenue fields
   - Connect Summarize previous year and current data nodes output to this node

10. **Add Email Send node:**
    - Type: Email Send
    - Credentials: SMTP account
    - To: desired recipient email
    - From: configured sender email
    - Subject: Weekly Report: Google Analytics: Last 7 days
    - Body: use HTML content from OpenAI node (`message.content`)
    - Connect OpenAI email report node output to this node

11. **Add OpenAI (Langchain) node for Telegram summary:**
    - Type: OpenAI (Langchain)
    - Credentials: OpenAI API
    - Model: GPT-4O-MINI
    - Prompt: Convert the HTML email report to plaintext, formatting each metric as a separate paragraph, e.g.:
      ```
      Total views: xx.xxx
      Total views previous year: xx.xxx
      Difference: x.xx%
      ```
    - Connect OpenAI email report node output to this node

12. **Add Telegram node:**
    - Type: Telegram
    - Credentials: Telegram API
    - Chat ID: your Telegram chat ID
    - Message text: use output text from OpenAI Telegram summary node
    - Connect OpenAI Telegram summary node output to this node

13. **Optional: Add Sticky Note node:**
    - To document workflow overview, sequence, and required credentials for clarity inside n8n editor.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Google Analytics API credentials are required; see https://docs.n8n.io/integrations/builtin/credentials/google/                                                        | Google Analytics credential setup                                                                   |
| AI API access needed (OpenAI, Anthropic, Google, or Ollama)                                                                                                             | OpenAI API credentials setup                                                                         |
| SMTP credentials needed for email sending                                                                                                                               | SMTP credential setup                                                                                |
| Telegram credentials optional, for sending Telegram messages                                                                                                            | Telegram credential setup, see https://docs.n8n.io/integrations/builtin/credentials/telegram/        |
| Contact workflow author via LinkedIn for questions: https://www.linkedin.com/in/friedemann-schuetz                                                                       | Author contact                                                                                       |
| The workflow uses German metric field names (Aufrufe, Nutzer, etc.) and outputs the report in German language context                                                  | Localization note                                                                                   |
| AI nodes use Langchain integration with specific models (GPT-4O, GPT-4O-MINI) requiring valid API keys                                                                 | Ensure API keys are valid and have quota                                                           |
| The workflow assumes GA4 API returns all metrics requested; mismatches or missing fields may cause errors in Set nodes                                                  | Validate GA API responses during initial runs                                                      |
| Date calculations rely on system clock correctness for accurate previous year date ranges                                                                               | System time zone and clock should be accurate                                                     |
| Email content is generated as HTML and may need styling adjustment depending on client email clients                                                                    | Optional customization may be needed                                                              |

---

This detailed reference enables understanding, modification, and full manual reconstruction of the workflow, while anticipating error points in integrations with Google Analytics, AI APIs, SMTP, and Telegram.