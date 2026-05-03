Automated Sales Leaderboard with HighLevel CRM, GPT-4o, Notion & Slack

https://n8nworkflows.xyz/workflows/automated-sales-leaderboard-with-highlevel-crm--gpt-4o--notion---slack-10148


# Automated Sales Leaderboard with HighLevel CRM, GPT-4o, Notion & Slack

---
### 1. Workflow Overview

This workflow, titled **"Automated Sales Leaderboard with HighLevel CRM, GPT-4o, Notion & Slack"**, automates the tracking, summarization, and motivational communication of sales representatives’ performance. It is designed to fetch all deals from the HighLevel CRM, clean and aggregate the data by sales rep, generate personalized performance dashboards in Notion, create motivational Slack messages using GPT-4o via LangChain, and notify the sales team in Slack. This enables transparent, up-to-date leaderboard insights and encourages healthy competition through daily automated updates.

The workflow logic is grouped into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 CRM Data Retrieval:** Fetch all deal records from HighLevel CRM.
- **1.3 Data Validation & Error Handling:** Validate fetched data and log errors to Google Sheets.
- **1.4 Data Cleaning & Structuring:** Normalize raw deal data into a consistent schema.
- **1.5 Data Aggregation:** Summarize deal data per sales representative.
- **1.6 Performance Dashboard Generation:** Create/update personalized Notion pages per rep.
- **1.7 AI Message Preparation:** Transform summary data for AI input.
- **1.8 AI-Driven Motivation Message Generation:** Use GPT-4o to craft motivational Slack messages via LangChain.
- **1.9 Slack Notification:** Send generated messages to selected Slack users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Initiates the workflow manually on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - Type: Manual Trigger  
  - Configuration: Default manual trigger, no parameters  
  - Inputs: None  
  - Outputs: Triggers the next node (Fetch All Deals from HighLevel CRM)  
  - Edge Cases: None (manual start)  
  - Version: 1  

---

#### 1.2 CRM Data Retrieval

- **Overview:**  
  Retrieves all deal (opportunity) records from HighLevel CRM without limits, capturing key fields for tracking.

- **Nodes Involved:**  
  - Fetch All Deals from HighLevel CRM  
  - Sticky Note (explaining this block)

- **Node Details:**  
  - **Fetch All Deals from HighLevel CRM**  
    - Type: HighLevel CRM node (resource: opportunity, operation: getAll)  
    - Credentials: OAuth2 with HighLevel account  
    - Configuration: No filters, pulls unlimited records  
    - Input: Trigger from manual start  
    - Output: Array of deal objects, including name, stage, assigned rep, value, client/company name, updated date  
    - Edge Cases: API rate limits, authentication failure, empty dataset  
    - Version: 2

---

#### 1.3 Data Validation & Error Handling

- **Overview:**  
  Validates that each deal has a valid ID to ensure data integrity; logs any failed or invalid deals to Google Sheets for auditing.

- **Nodes Involved:**  
  - Validate Deal Fetch Success (IF node)  
  - Log Fetch or Validation Errors (Google Sheets Append)  
  - Sticky Notes explaining validation and error logging

- **Node Details:**  
  - **Validate Deal Fetch Success**  
    - Type: IF node  
    - Configuration: Checks if `$json.id` is not empty for each deal  
    - Inputs: Deals from HighLevel fetch  
    - Outputs:  
      - True branch: Valid deals → proceed to cleaning  
      - False branch: Invalid deals → logged to error sheet  
    - Edge Cases: Missing or empty deal IDs, malformed data  
    - Version: 2.2

  - **Log Fetch or Validation Errors**  
    - Type: Google Sheets Append  
    - Configuration: Appends records with fields `error_id` and `error` into a dedicated error log sheet  
    - Credentials: Google Sheets OAuth2  
    - Input: Deals failing validation  
    - Output: None (logs only)  
    - Edge Cases: Google Sheets API limits, auth failures  
    - Version: 4.6

---

#### 1.4 Data Cleaning & Structuring

- **Overview:**  
  Normalizes and structures raw CRM deal data into a consistent and simplified JSON schema for downstream processing.

- **Nodes Involved:**  
  - Clean & Structure Deal Data (Code node)  
  - Sticky Note explaining cleaning steps

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Maps all input deals into objects with normalized fields: deal_id, deal_name, rep_id (or "Unassigned"), status, stage_id, value (default 0), client_name (default "Unknown"), company_name, updated_at  
    - Handles null or missing values gracefully  
  - Inputs: Valid deal data from IF node  
  - Outputs: Array of cleaned deal JSON objects  
  - Edge Cases: Deals missing optional fields, unexpected data types  
  - Version: 2

---

#### 1.5 Data Aggregation

- **Overview:**  
  Aggregates deal data by sales representative to generate performance summaries including total deals, total value, deals won, and average deal value.

- **Nodes Involved:**  
  - Summarize Sales by Representative (Code node)  
  - Sticky Note describing aggregation logic

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Iterates over cleaned deals, grouping by rep_id  
    - Totals deals, sums monetary values, counts deals with status including "won" (case-insensitive)  
    - Calculates average deal value per rep  
  - Inputs: Cleaned deal data  
  - Outputs: Array of summarized rep performance objects  
  - Edge Cases: Reps with zero deals, division by zero handled via logic  
  - Version: 2

---

#### 1.6 Performance Dashboard Generation

- **Overview:**  
  Generates or updates a personalized Notion page per sales rep with their daily performance summary and motivational context.

- **Nodes Involved:**  
  - Generate Notion Performance Dashboard  
  - Sticky Note describing Notion page contents and business value

- **Node Details:**  
  - Type: Notion node  
  - Configuration:  
    - Creates a page titled `{rep_id} - Sales Rep Performance Tracker` under a specific parent page ID  
    - Populates page with:  
      - Heading including rep_id  
      - Summary stats (total deals, total value, deals won, average deal value)  
      - Motivational and business value text emphasizing transparency and competition  
    - Runs for each rep summary record  
  - Inputs: Aggregated sales rep summaries  
  - Outputs: Updated Notion pages  
  - Credentials: Notion API account  
  - Edge Cases: Notion API rate limits, permission issues, invalid page IDs  
  - Version: 2.2

---

#### 1.7 AI Message Preparation

- **Overview:**  
  Formats and flattens sales rep summary data into JSON items tailored for AI text generation input.

- **Nodes Involved:**  
  - Transform Data for AI Input (Code node)  
  - Sticky Note explaining the transformation

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Maps summary data to simplified JSON objects containing rep_id, total_deals, total_value, won_deals, avg_value  
    - Emits each rep as an individual item to facilitate AI processing  
  - Inputs: Aggregated sales rep summaries  
  - Outputs: Array of AI-friendly JSON items  
  - Edge Cases: Missing or incomplete summary fields handled by direct mapping  
  - Version: 2

---

#### 1.8 AI-Driven Motivation Message Generation

- **Overview:**  
  Uses GPT-4o (Azure OpenAI) via LangChain agent to generate short, friendly, motivational Slack messages per sales rep based on performance data.

- **Nodes Involved:**  
  - GPT-4o Model Configuration (Language Model node)  
  - AI-Generated Motivational Slack Messages (LangChain Agent node)  
  - Sticky Notes detailing GPT-4o setup and message style

- **Node Details:**  
  - **GPT-4o Model Configuration**  
    - Type: LangChain Azure OpenAI Chat Model  
    - Configuration: Model set to "gpt-4o", system message defines role to generate motivational, friendly messages with optimized token usage  
    - Credentials: Azure OpenAI API  
  - **AI-Generated Motivational Slack Messages**  
    - Type: LangChain Agent  
    - Configuration:  
      - Prompt instructs to create 2-3 line motivational blurbs per rep  
      - Includes emojis 🎯🔥💪 for energy  
      - Mentions rep_id, highlights achievements positively  
      - Keeps messages concise for Slack readability  
    - Inputs: Flattened summary data from previous node  
    - Outputs: Generated motivational text per rep  
  - Edge Cases: OpenAI API rate limits, prompt failures, malformed input data  
  - Version: 1 (GPT-4o), 2.1 (LangChain Agent)

---

#### 1.9 Slack Notification

- **Overview:**  
  Sends the AI-generated motivational messages to the sales team via Slack, targeting specific user(s) for transparency and morale building.

- **Nodes Involved:**  
  - Notify Sales Team in Slack  
  - Sticky Note describing Slack notification purpose

- **Node Details:**  
  - Type: Slack node  
  - Configuration:  
    - Posts message text from AI output (`$json.output`)  
    - Targets a specific Slack user ID (hardcoded user list)  
    - Uses Slack API credentials  
  - Inputs: Motivational message text from AI node  
  - Outputs: Slack message sent confirmation  
  - Edge Cases: Slack API authorization errors, user ID not found, message formatting issues  
  - Version: 2.3

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                                | Input Node(s)                     | Output Node(s)                                | Sticky Note                                                       |
|----------------------------------|--------------------------------|-----------------------------------------------|----------------------------------|-----------------------------------------------|------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Workflow start trigger                         | None                             | Fetch All Deals from HighLevel CRM            |                                                                  |
| Fetch All Deals from HighLevel CRM | HighLevel CRM                  | Fetches all deal records from CRM              | When clicking ‘Execute workflow’ | Validate Deal Fetch Success                    | 📦 Fetch All Deals from HighLevel CRM - retrieves all opportunities |
| Validate Deal Fetch Success       | IF                            | Checks if fetched deals have valid IDs         | Fetch All Deals from HighLevel CRM | Clean & Structure Deal Data / Log Fetch or Validation Errors | 🔍 Validate Deal Fetch Success (IF Node) - validation gate         |
| Log Fetch or Validation Errors    | Google Sheets Append          | Logs errors for failed or invalid deals        | Validate Deal Fetch Success (false branch) | None                                    | 🚨 Log Fetch or Validation Errors - centralized error logging     |
| Clean & Structure Deal Data       | Code                          | Normalizes raw deal data into clean schema     | Validate Deal Fetch Success (true branch) | Summarize Sales by Representative           | 🧹 Clean & Structure Deal Data - prepares dataset for analysis     |
| Summarize Sales by Representative| Code                          | Aggregates deals per rep into performance stats| Clean & Structure Deal Data       | Generate Notion Performance Dashboard, Transform Data for AI Input | 📊 Summarize Sales by Representative - performance summary        |
| Generate Notion Performance Dashboard | Notion                      | Creates personalized Notion pages per rep      | Summarize Sales by Representative | None                                         | 🧾 Generate Notion Performance Dashboard - personalized sales pages|
| Transform Data for AI Input       | Code                          | Formats summary data for AI consumption        | Summarize Sales by Representative | GPT-4o Model Configuration                   | ⚙️ Transform Data for AI Input - prepares AI-friendly JSON         |
| GPT-4o Model Configuration        | LangChain Azure OpenAI LM Chat| Configures GPT-4o model for message generation | Transform Data for AI Input       | AI-Generated Motivational Slack Messages       | 🧠 GPT-4o Model Configuration - sets AI tone and role              |
| AI-Generated Motivational Slack Messages | LangChain Agent           | Generates motivational Slack messages           | GPT-4o Model Configuration        | Notify Sales Team in Slack                     | 🤖 AI-Generated Motivational Slack Messages - creates motivational blurbs |
| Notify Sales Team in Slack        | Slack                         | Sends messages to sales reps on Slack          | AI-Generated Motivational Slack Messages | None                                         | 💬 Notify Sales Team in Slack - posts motivational updates         |
| Sticky Note                      | Sticky Note                   | Documentation notes                             | N/A                              | N/A                                          | See respective notes in documentation                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: *When clicking ‘Execute workflow’*  
   - Type: Manual Trigger (default)  
   - No parameters needed.

2. **Create HighLevel CRM Node:**  
   - Name: *Fetch All Deals from HighLevel CRM*  
   - Type: HighLevel CRM  
   - Resource: `opportunity`  
   - Operation: `getAll`  
   - Filters: None (fetch all)  
   - Credentials: Configure OAuth2 with HighLevel account  
   - Connect input from Manual Trigger node output.

3. **Create IF Node for Validation:**  
   - Name: *Validate Deal Fetch Success*  
   - Type: IF  
   - Condition: Check if `$json.id` is not empty (operator: string, notEmpty)  
   - Connect input from HighLevel CRM node output.

4. **Create Google Sheets Append Node for Errors:**  
   - Name: *Log Fetch or Validation Errors*  
   - Type: Google Sheets (Append)  
   - Document ID: Use your Google Sheet ID for error logging  
   - Sheet Name: Specify error log sheet/tab  
   - Columns: `error_id` (string), `error` (string)  
   - Credentials: Google Sheets OAuth2 with appropriate account  
   - Connect input from IF node false branch (invalid deals).

5. **Create Code Node to Clean Data:**  
   - Name: *Clean & Structure Deal Data*  
   - Type: Code (JavaScript)  
   - Code: Map input deals to normalized structure with fallback defaults for missing fields.  
   - Connect input from IF node true branch (valid deals).

6. **Create Code Node to Summarize by Rep:**  
   - Name: *Summarize Sales by Representative*  
   - Type: Code (JavaScript)  
   - Code: Aggregate deals by rep_id; calculate total deals, total value, won deals, average deal value; handle missing rep_id as "Unassigned".  
   - Connect input from Clean & Structure Deal Data node.

7. **Create Notion Node for Dashboard:**  
   - Name: *Generate Notion Performance Dashboard*  
   - Type: Notion  
   - Parent Page ID: Configure to your Notion workspace parent page  
   - Title: `{{ $json["rep_id"] }} - Sales Rep Performance Tracker`  
   - Add blocks to display performance stats and motivational text as per description.  
   - Credentials: Notion API with configured access  
   - Connect input from Summarize Sales by Representative node output.

8. **Create Code Node for AI Input Formatting:**  
   - Name: *Transform Data for AI Input*  
   - Type: Code (JavaScript)  
   - Code: Flatten summary data to simplified JSON with rep_id, total_deals, total_value, won_deals, avg_value; emit each as separate item.  
   - Connect input from Summarize Sales by Representative node output.

9. **Create LangChain Azure OpenAI Model Node:**  
   - Name: *GPT-4o Model Configuration*  
   - Type: LangChain LM Chat Azure OpenAI  
   - Model: `gpt-4o`  
   - System Message: Define role as motivational message generator for sales team, keeping messages friendly, concise, and positive.  
   - Credentials: Azure OpenAI API account  
   - Connect input from Transform Data for AI Input node.

10. **Create LangChain Agent Node for Message Generation:**  
    - Name: *AI-Generated Motivational Slack Messages*  
    - Type: LangChain Agent  
    - Prompt: Provide JSON performance data and instruct to create 2-3 line motivational Slack messages with emojis and rep mentions.  
    - Connect input from GPT-4o Model Configuration node output.

11. **Create Slack Node to Notify:**  
    - Name: *Notify Sales Team in Slack*  
    - Type: Slack  
    - Text: Use `{{$json.output}}` from AI node  
    - Target: Specific Slack user(s) or channel ID(s)  
    - Credentials: Slack OAuth with relevant permissions  
    - Connect input from AI-Generated Motivational Slack Messages node output.

12. **Connect All Nodes in Execution Order:**  
    Manual Trigger → HighLevel CRM → IF Validation →  
    - True branch → Clean Data → Summarize → Notion & AI Input Transform → GPT-4o → AI Message Agent → Slack Notify  
    - False branch → Log Errors in Google Sheets

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Transparent performance insights foster healthy competition and better sales accountability.                           | Workflow business value description                                                                 |
| Use emojis 🎯🔥💪 in motivational messages for energy and tone consistency.                                             | AI message style guidance                                                                            |
| GPT-4o via Azure OpenAI enables scalable and optimized language generation integrated with LangChain AI agent.          | AI node configuration details                                                                        |
| Notion pages refresh daily to keep sales reps updated with current leaderboard and stats.                               | Notion API usage for personalized dashboards                                                        |
| Error logging centralizes data quality issues for manual review via a Google Sheet accessible to admins.                | Google Sheets error log setup                                                                         |
| Slack messages promote team morale and transparency by direct user mentions and timely updates.                        | Slack notification best practices                                                                    |
| For HighLevel CRM OAuth2, ensure token refresh and proper scopes to avoid API failures.                                 | HighLevel API authentication note                                                                    |
| LangChain nodes require specific n8n community nodes installed and compatible n8n version supporting `@n8n/n8n-nodes-langchain` | Node installation and version requirements                                                          |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.