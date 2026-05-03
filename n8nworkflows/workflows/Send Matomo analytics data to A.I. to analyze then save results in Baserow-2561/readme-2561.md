Send Matomo analytics data to A.I. to analyze then save results in Baserow

https://n8nworkflows.xyz/workflows/send-matomo-analytics-data-to-a-i--to-analyze-then-save-results-in-baserow-2561


# Send Matomo analytics data to A.I. to analyze then save results in Baserow

---

### 1. Workflow Overview

This workflow automates the process of analyzing website visitor data from Matomo analytics using an AI language model, then storing the analyzed insights into a Baserow database. It targets website owners and SEO practitioners who want to generate actionable SEO reports without hiring experts, focusing on visitors with more than 3 visits over the past week.

The workflow can be logically divided into these blocks:  
- **1.1 Triggering Input** — Manual or scheduled initiation of the workflow.  
- **1.2 Data Acquisition from Matomo** — Retrieval of visitor analytics data for visitors with more than 3 visits.  
- **1.3 Data Parsing & Prompt Preparation** — Formatting the raw Matomo data into a structured prompt for AI analysis.  
- **1.4 AI Processing** — Sending the prompt to Openrouter’s free LLM (Meta LLaMA 3.1) for SEO insights generation.  
- **1.5 Data Storage** — Writing the AI-generated SEO analysis results into a Baserow table for record keeping and future reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering Input

- **Overview:**  
  This block allows the workflow to be activated either manually by a user or automatically on a weekly schedule.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Schedule Trigger

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Enables manual start for testing or ad-hoc runs.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: To "Get data from Matomo" node  
    - Edge Cases: None significant; manual start means user controls execution.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically initiates the workflow once per week.  
    - Configuration: Interval set to 1 week.  
    - Inputs: None  
    - Outputs: To "Get data from Matomo" node  
    - Edge Cases: Scheduling misconfiguration could cause missed runs; time zone considerations may affect exact timing.

---

#### 2.2 Data Acquisition from Matomo

- **Overview:**  
  This block requests visitor analytics data from the Matomo API, specifically targeting visitors with more than 3 visits in the last 30 days, limited to 100 records.

- **Nodes Involved:**  
  - Get data from Matomo

- **Node Details:**

  - **Get data from Matomo**  
    - Type: HTTP Request  
    - Role: Fetches detailed visitor data from the Matomo analytics API.  
    - Configuration:  
      - Method: POST  
      - URL: `https://shrewd-lyrebird.pikapod.net/index.php` (Matomo API endpoint)  
      - Body: multipart-form-data with parameters:  
        - module=API  
        - method=Live.getLastVisitsDetails  
        - idSite=3 (Matomo site ID — user must replace as needed)  
        - period=range  
        - date=last30 (last 30 days)  
        - format=JSON  
        - segment=visitCount>3 (filter visitors with more than 3 visits)  
        - filter_limit=100  
        - showColumns=actionDetails,visitIp,visitorId,visitCount (data fields)  
        - token_auth=User’s Matomo API token (must be provided by user)  
    - Inputs: From manual or schedule trigger  
    - Outputs: To "Parse data from Matomo" node  
    - Edge Cases:  
      - Auth token missing or invalid → API error.  
      - API downtime or network issues → request failure or timeout.  
      - Large data sets may exceed filter_limit.  
      - User must replace placeholder token_auth with a valid token.

---

#### 2.3 Data Parsing & Prompt Preparation

- **Overview:**  
  This block transforms raw Matomo JSON data into a formatted textual prompt suitable for AI analysis, summarizing visitor actions and engagement.

- **Nodes Involved:**  
  - Parse data from Matomo

- **Node Details:**

  - **Parse data from Matomo**  
    - Type: Code (JavaScript)  
    - Role: Converts API JSON into a human-readable prompt for AI.  
    - Configuration: Custom JS code that:  
      - Iterates over each visitor record.  
      - Extracts visitor ID, visit count, and detailed page actions (page title, URL, time spent).  
      - Formats these details into a bullet point list.  
      - Composes a prompt requesting AI to analyze common visitor paths, popular pages, engagement patterns, and provide improvement recommendations.  
    - Key Expressions:  
      - Maps over `$input.all()` to access all items.  
      - Uses template literals to compose prompt content.  
    - Inputs: From "Get data from Matomo"  
    - Outputs: JSON with a "messages" array containing the formatted prompt for AI consumption.  
    - Edge Cases:  
      - Empty or malformed input data → prompt may be incomplete or cause downstream errors.  
      - Unexpected JSON structure → code may throw exceptions.  
      - Large inputs could exceed prompt size limits for AI.

---

#### 2.4 AI Processing

- **Overview:**  
  Sends the formatted visitor data prompt to Openrouter’s API using a free LLM model specialized for instructive SEO analysis.

- **Nodes Involved:**  
  - Send data to A.I. for analysis

- **Node Details:**

  - **Send data to A.I. for analysis**  
    - Type: HTTP Request  
    - Role: Calls Openrouter AI endpoint to generate SEO insights based on visitor data.  
    - Configuration:  
      - Method: POST  
      - URL: `https://openrouter.ai/api/v1/chat/completions`  
      - Body (JSON): Includes:  
        - model: `"meta-llama/llama-3.1-70b-instruct:free"`  
        - messages: an array with a user role message embedding the encoded visitor data prompt.  
      - Authentication: HTTP Header Authentication with a custom header:  
        - Header Name: `Authorization`  
        - Header Value: `Bearer {your API key}` (space after "Bearer" important)  
    - Credentials: User must provide Openrouter API key via n8n credentials.  
    - Inputs: From "Parse data from Matomo"  
    - Outputs: AI analysis result JSON to "Store results in Baserow"  
    - Edge Cases:  
      - Invalid or expired API key → HTTP 401 Unauthorized.  
      - Rate limiting or quota exceeded → 429 Too Many Requests.  
      - Network issues causing timeouts.  
      - Prompt size limits exceeded → API error.  
      - Encoding or interpolation errors in prompt string could cause malformed requests.  

---

#### 2.5 Data Storage

- **Overview:**  
  Saves the AI-generated SEO insights along with the current date and site name into a Baserow database table for tracking and reporting.

- **Nodes Involved:**  
  - Store results in Baserow

- **Node Details:**

  - **Store results in Baserow**  
    - Type: Baserow Node (Database Create Operation)  
    - Role: Inserts a new row into a specified Baserow table with the AI output.  
    - Configuration:  
      - Operation: Create  
      - Database ID: 121 (user’s Baserow database)  
      - Table ID: 643 (user’s table)  
      - Fields:  
        - Date (fieldId 6261): current date in `yyyy-MM-dd` format (generated dynamically)  
        - Note (fieldId 6262): AI analysis text from Openrouter response  
        - Blog (fieldId 6263): user-defined blog/site name (static string)  
    - Credentials: User must configure Baserow API credentials in n8n.  
    - Inputs: From "Send data to A.I. for analysis"  
    - Outputs: None (terminal node)  
    - Edge Cases:  
      - Incorrect database or table IDs → API error.  
      - Missing or invalid API credentials → authentication failure.  
      - API rate limits or network errors.  
      - Data type mismatches in fields could cause insertion failures.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                   | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                            |
|----------------------------|------------------------|---------------------------------|-------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger         | Manual workflow initiation       | None                    | Get data from Matomo         |                                                                                                                        |
| Schedule Trigger           | Schedule Trigger       | Scheduled weekly initiation       | None                    | Get data from Matomo         |                                                                                                                        |
| Get data from Matomo       | HTTP Request           | Retrieve Matomo visitor data      | When clicking ‘Test workflow’, Schedule Trigger | Parse data from Matomo           | "Get Matomo Data": Instructions to input Matomo API key and generate token_auth in Matomo dashboard                     |
| Parse data from Matomo     | Code (JavaScript)      | Format raw data into AI prompt    | Get data from Matomo     | Send data to A.I. for analysis |                                                                                                                        |
| Send data to A.I. for analysis | HTTP Request           | Send prompt to Openrouter AI      | Parse data from Matomo   | Store results in Baserow     | "Send data to A.I.": Fill Openrouter credentials with Header Auth; username: Authorization; password: Bearer {API key}  |
| Store results in Baserow   | Baserow                | Save AI analysis results          | Send data to A.I. for analysis | None                      | "Send data to Baserow": Create table with columns Date, Note, Blog; enter website name under Blog field                 |
| Sticky Note                | Sticky Note            | Documentation and instructions    | None                    | None                        | Workflow overview, links to tutorials and SEO AI agent                                                               |
| Sticky Note1               | Sticky Note            | Instructions for Matomo API Setup | None                    | None                        | How to obtain Matomo API key and create auth token                                                                     |
| Sticky Note2               | Sticky Note            | Instructions for AI node setup    | None                    | None                        | Details on Openrouter credentials setup and prompt customization                                                       |
| Sticky Note3               | Sticky Note            | Instructions for Baserow setup    | None                    | None                        | Baserow table schema and data entry instructions                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Allows manual execution for testing.

2. **Create a Schedule Trigger node**  
   - Name: "Schedule Trigger"  
   - Set interval to 1 week for automatic weekly runs.

3. **Create an HTTP Request node**  
   - Name: "Get data from Matomo"  
   - Set method to POST  
   - URL: Your Matomo API endpoint (e.g., `https://your-matomo-domain/index.php`)  
   - Content Type: multipart-form-data  
   - Body Parameters:  
     - module: API  
     - method: Live.getLastVisitsDetails  
     - idSite: your Matomo site ID (e.g., 3)  
     - period: range  
     - date: last30 (or last7 to match weekly data)  
     - format: JSON  
     - segment: visitCount>3  
     - filter_limit: 100  
     - showColumns: actionDetails,visitIp,visitorId,visitCount  
     - token_auth: Your Matomo API token (replace placeholder)  
   - Connect outputs of both trigger nodes to this node.

4. **Create a Code node**  
   - Name: "Parse data from Matomo"  
   - Language: JavaScript  
   - Paste the provided JavaScript code that formats visitor data into a prompt.  
   - Input: Output from "Get data from Matomo" node.  
   - Output: JSON containing a "messages" array for AI.

5. **Create an HTTP Request node**  
   - Name: "Send data to A.I. for analysis"  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Set "Content-Type" to application/json.  
   - Body (raw JSON):  
     ```json
     {
       "model": "meta-llama/llama-3.1-70b-instruct:free",
       "messages": [
         {
           "role": "user",
           "content": "You are an SEO expert. This is data of visitors who have visited my site more than 3 times and the pages they have visited. Can you give me insights into this data:{{ encodeURIComponent($json.messages[0].content)}}"
         }
       ]
     }
     ```  
   - Authentication: HTTP Header Authentication (via n8n credentials)  
     - Header Name: Authorization  
     - Header Value: Bearer {Your Openrouter API Key} (note the space after "Bearer")  
   - Connect input from "Parse data from Matomo" node.

6. **Create a Baserow node**  
   - Name: "Store results in Baserow"  
   - Operation: Create  
   - Database ID: Your Baserow database ID  
   - Table ID: Your Baserow table ID  
   - Set fields:  
     - Date: Expression `={{ DateTime.now().toFormat('yyyy-MM-dd') }}`  
     - Note: Map to AI response content: `={{ $json.choices[0].message.content }}`  
     - Blog: Static string with your blog/site name (e.g., "Your blog name")  
   - Connect input from "Send data to A.I. for analysis" node.

7. **Connections:**  
   - Connect manual trigger and schedule trigger outputs to "Get data from Matomo".  
   - Connect "Get data from Matomo" to "Parse data from Matomo".  
   - Connect "Parse data from Matomo" to "Send data to A.I. for analysis".  
   - Connect "Send data to A.I. for analysis" to "Store results in Baserow".

8. **Credential Setup:**  
   - Matomo API token: Replace placeholder in "Get data from Matomo" node.  
   - Openrouter API key: Create HTTP Header Auth credential in n8n with header name "Authorization" and value "Bearer {API Key}".  
   - Baserow API key: Configure Baserow credentials in n8n.

9. **Baserow Table Setup:**  
   - Create a table with columns:  
     - Date (Date type)  
     - Note (Long text)  
     - Blog (Short text)  
   - Note the database and table IDs for node configuration.

10. **Testing:**  
    - Run the manual trigger to verify data flow and outputs.  
    - Check Baserow for inserted records.  
    - Inspect AI responses for relevance and adjust prompt as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow analyzes Matomo visitors with >3 visits over last week and compares to last week’s data to give SEO suggestions. | Workflow purpose and use case description                                                                        |
| [Watch youtube tutorial here](https://www.youtube.com/watch?v=hGzdhXyU-o8)                                                | Video guide for usage                                                                                            |
| [Get my SEO A.I. agent system here](https://2828633406999.gumroad.com/l/rumjahn)                                          | Commercial product link for an SEO AI agent system                                                              |
| [How to create an A.I. Agent to analyze Matomo analytics using n8n for free](https://rumjahn.com/how-to-create-an-a-i-agent-to-analyze-matomo-analytics-using-n8n-for-free/) | Detailed blog post explaining the workflow and setup                                                            |
| Notes on Openrouter Header Auth: Use header name "Authorization" and value "Bearer {API key}", with a space after "Bearer" | Critical for correct API authentication                                                                          |
| Baserow table must have columns: Date, Note, Blog                                                                       | Required schema for data insertion                                                                               |
| Matomo API token can be generated from: Administration > Personal > Security > Auth tokens in Matomo dashboard            | Instructions for obtaining necessary API credentials                                                            |

---

This structured documentation should enable advanced users and automation agents to fully understand, reproduce, and maintain the workflow in n8n, as well as to anticipate common error conditions and setup requirements.