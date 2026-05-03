Summarize SERPBear data with AI (via Openrouter) and save it to Baserow

https://n8nworkflows.xyz/workflows/summarize-serpbear-data-with-ai--via-openrouter--and-save-it-to-baserow-2565


# Summarize SERPBear data with AI (via Openrouter) and save it to Baserow

---

### 1. Workflow Overview

This workflow automates the process of analyzing keyword ranking data from SerpBear, an open-source SEO analytics tool, by summarizing it using AI via Openrouter and storing the results in a Baserow database. It is designed primarily for website owners and SEO practitioners who want to monitor and improve their keyword rankings without hiring an SEO expert.

The workflow logic is divided into the following functional blocks:

- **1.1 Input Triggers**: Initiates the workflow manually or on a weekly schedule.
- **1.2 SerpBear Data Retrieval**: Requests keyword ranking data from the SerpBear API using user credentials.
- **1.3 Data Parsing & Prompt Generation**: Processes the raw keyword data to create a structured summary and generates an AI prompt.
- **1.4 AI Analysis via Openrouter**: Sends the prompt to Openrouter’s AI endpoint for expert SEO analysis and recommendations.
- **1.5 Save Analysis Results**: Stores the AI-generated report in a Baserow database for historical tracking and review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Triggers

**Overview:**  
This block provides two entry points to start the workflow: manual execution by the user or an automated weekly schedule.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger (Interval-based Trigger)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Enables manual start for testing or on-demand runs.  
  - Config: No parameters; triggers on user interaction.  
  - Inputs: None  
  - Outputs: Connected to “Get data from SerpBear” node.  
  - Edge Cases: None significant; user must manually trigger.  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow on a weekly interval.  
  - Config: Interval set to 1 week.  
  - Inputs: None  
  - Outputs: Connected to “Get data from SerpBear” node.  
  - Edge Cases: If the workflow is inactive, scheduled trigger won't run.  

---

#### 2.2 SerpBear Data Retrieval

**Overview:**  
Fetches keyword ranking data for a specified domain from the SerpBear API using HTTP Header Authentication.

**Nodes Involved:**  
- Get data from SerpBear

**Node Details:**

- **Get data from SerpBear**  
  - Type: HTTP Request  
  - Role: Retrieves JSON keyword ranking data from SerpBear API.  
  - Config:  
    - HTTP method: GET (default)  
    - URL: `https://myserpbearinstance.com/api/keyword?id=22` (example placeholder; user should replace with their instance and keyword list ID)  
    - Query Parameter: `domain` set to the target domain (e.g., "rumjahn.com").  
    - Authentication: HTTP Header Auth with custom header credentials (token-based).  
  - Inputs: Trigger nodes (“When clicking ‘Test workflow’” and “Schedule Trigger”)  
  - Outputs: Passes raw keyword data JSON to “Parse data from SerpBear” node.  
  - Edge Cases:  
    - Authentication failure if API token is invalid or expired.  
    - Network or timeout errors if SerpBear instance is unreachable.  
    - Unexpected API response format or missing data.  
  - Notes: User must obtain correct API token and domain ID from SerpBear.  

---

#### 2.3 Data Parsing & Prompt Generation

**Overview:**  
Transforms raw keyword data into a structured summary and dynamically generates a prompt string for AI analysis.

**Nodes Involved:**  
- Parse data from SerpBear

**Node Details:**

- **Parse data from SerpBear**  
  - Type: Code (JavaScript)  
  - Role: Processes keyword ranking info, calculates trends, averages, and formats a detailed prompt for AI.  
  - Config: Custom JS code that:  
    - Extracts keywords array from API response.  
    - Calculates current position, 7-day average position, and trend (improving, declining, stable).  
    - Constructs a multi-line prompt string listing keyword stats.  
    - Includes instructions requesting AI to analyze and provide actionable SEO insights.  
  - Key Expressions:  
    - Uses current date via `new Date().toISOString().split('T')[0]`  
    - Computes average position from keyword history.  
    - Conditional logic for trend evaluation.  
  - Inputs: Raw JSON from “Get data from SerpBear”  
  - Outputs: JSON object with a single field `prompt` containing the AI input text.  
  - Edge Cases:  
    - Missing or empty keyword list in API response.  
    - Unexpected or malformed history data causing calculation failures.  
    - Empty or null fields in keyword entries.  
  - Version Requirements: n8n version supporting Code Node v2 with JS ES6 features.  

---

#### 2.4 AI Analysis via Openrouter

**Overview:**  
Sends the prepared prompt to Openrouter’s chat completion API for expert SEO analysis and recommendation generation.

**Nodes Involved:**  
- Send data to A.I. for analysis

**Node Details:**

- **Send data to A.I. for analysis**  
  - Type: HTTP Request  
  - Role: Calls Openrouter API endpoint to generate AI text based on SEO prompt.  
  - Config:  
    - URL: `https://openrouter.ai/api/v1/chat/completions`  
    - Method: POST  
    - Body (JSON):  
      - Model: `meta-llama/llama-3.1-70b-instruct:free`  
      - Messages payload with role “user” and content using the encoded prompt from previous node.  
    - Authentication: HTTP Header Auth with bearer token (Openrouter API key).  
  - Key Expressions: Uses `encodeURIComponent($json.prompt)` to safely pass prompt text.  
  - Inputs: Prompt JSON from “Parse data from SerpBear”  
  - Outputs: AI response JSON containing generated text under `choices[0].message.content`.  
  - Edge Cases:  
    - Authentication failure if Openrouter API key invalid or expired.  
    - Rate limiting or quota exceeded on Openrouter side.  
    - Timeout or network failures.  
    - Possible incomplete or unexpected AI response structure.  
  - Notes: User must configure HTTP Header Auth credentials with header name `Authorization` and value `Bearer {API key}` including space after “Bearer”.  

---

#### 2.5 Save Analysis Results

**Overview:**  
Stores the AI-generated SEO analysis report along with the current date and blog name into a Baserow database table for record-keeping.

**Nodes Involved:**  
- Save data to Baserow

**Node Details:**

- **Save data to Baserow**  
  - Type: Baserow node (API integration)  
  - Role: Creates a new row in a specified Baserow table with the analysis data.  
  - Config:  
    - Database ID: 121 (example; user-specific)  
    - Table ID: 644 (example; user-specific)  
    - Fields:  
      - Date: dynamically set to today’s date in `yyyy-MM-dd` format using `DateTime.now().toFormat('yyyy-MM-dd')`  
      - Note: AI analysis text from `choices[0].message.content`  
      - Blog: Static string, e.g. “Rumjahn” (user’s blog/site name)  
    - Operation: Create row  
  - Inputs: AI response from “Send data to A.I. for analysis”  
  - Outputs: None connected (terminal node)  
  - Edge Cases:  
    - Credential errors if Baserow API token invalid.  
    - Table or field ID mismatch errors if Baserow database schema changed or missing.  
    - API rate limiting or network errors.  
  - Notes: Requires prior setup of Baserow database with columns: Date, Note, Blog.  

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                  | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                    |
|---------------------------|-------------------------|--------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Manual workflow start           | None                         | Get data from SerpBear         |                                                                                                               |
| Schedule Trigger           | Schedule Trigger        | Weekly automated start          | None                         | Get data from SerpBear         |                                                                                                               |
| Get data from SerpBear     | HTTP Request            | Fetch keyword ranking data      | When clicking ‘Test workflow’, Schedule Trigger | Parse data from SerpBear       | ## Get SERPBear Data 1. Enter your SerpBear API keys and URL. You need to find your website ID which is probably 1. 2. Navigate to Administration > Personal > Security > Auth tokens within your Matomo dashboard. Click on Create new token and provide a purpose for reference. |
| Parse data from SerpBear   | Code                    | Parse and summarize keyword data| Get data from SerpBear       | Send data to A.I. for analysis |                                                                                                               |
| Send data to A.I. for analysis | HTTP Request            | Send prompt to Openrouter AI    | Parse data from SerpBear     | Save data to Baserow           | ## Send data to A.I. Fill in your Openrouter A.I. credentials. Use Header Auth. - Username: Authorization - Password: Bearer {insert your API key} Remember to add a space after bearer. Also, feel free to modify the prompt to A.I. |
| Save data to Baserow       | Baserow                 | Save AI analysis to database    | Send data to A.I. for analysis| None                         | ## Send data to Baserow Create a table first with the following columns: - Date - Note - Blog Enter the name of your website under "Blog" field. |
| Sticky Note               | Sticky Note             | Informational note              | None                         | None                          | ## Send Matomo analytics to A.I. and save results to baserow This workflow will check the Google keywords for your site and it's rank. [💡 You can read more about this workflow here](https://rumjahn.com/how-to-create-an-a-i-agent-to-analyze-serpbear-keyword-rankings-using-n8n-for-free-without-any-coding-skills-required/) |
| Sticky Note1              | Sticky Note             | Informational note              | None                         | None                          | ## Get SERPBear Data 1. Enter your SerpBear API keys and URL. You need to find your website ID which is probably 1. 2. Navigate to Administration > Personal > Security > Auth tokens within your Matomo dashboard. Click on Create new token and provide a purpose for reference. |
| Sticky Note2              | Sticky Note             | Informational note              | None                         | None                          | ## Send data to A.I. Fill in your Openrouter A.I. credentials. Use Header Auth. - Username: Authorization - Password: Bearer {insert your API key} Remember to add a space after bearer. Also, feel free to modify the prompt to A.I. |
| Sticky Note3              | Sticky Note             | Informational note              | None                         | None                          | ## Send data to Baserow Create a table first with the following columns: - Date - Note - Blog Enter the name of your website under "Blog" field. |


---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: “When clicking ‘Test workflow’”  
   - Purpose: Manual start for testing. No configuration needed.

2. **Create a Schedule Trigger node**  
   - Name: “Schedule Trigger”  
   - Set interval to 1 week (every 1 week).  
   - Purpose: Automatic weekly execution.

3. **Create an HTTP Request node**  
   - Name: “Get data from SerpBear”  
   - Method: GET  
   - URL: `https://myserpbearinstance.com/api/keyword?id=22` *(replace with your SerpBear URL and keyword list ID)*  
   - Query Parameter: Add `domain` with your website domain (e.g., `rumjahn.com`).  
   - Authentication: HTTP Header Auth  
     - Header name: as required by SerpBear (usually `Authorization` or custom)  
     - Header value: your API token/key  
   - Connect inputs from both “When clicking ‘Test workflow’” and “Schedule Trigger”.

4. **Create a Code node**  
   - Name: “Parse data from SerpBear”  
   - Language: JavaScript  
   - Paste the provided JS code that:  
     - Extracts `keywords` array from the API response JSON.  
     - Calculates current position, 7-day average, and trend per keyword.  
     - Constructs a prompt string summarizing data for AI input.  
   - Connect input from “Get data from SerpBear”.

5. **Create an HTTP Request node**  
   - Name: “Send data to A.I. for analysis”  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Authentication: HTTP Header Auth  
     - Header name: `Authorization`  
     - Header value: `Bearer {Your Openrouter API Key}` (note the space after “Bearer”)  
   - Body (JSON):  
     ```json
     {
       "model": "meta-llama/llama-3.1-70b-instruct:free",
       "messages": [
         {
           "role": "user",
           "content": "You are an SEO expert. This is keyword data for my site. Can you summarize the data into a table and then give me some suggestions:{{ encodeURIComponent($json.prompt) }}"
         }
       ]
     }
     ```  
   - Use expression for the prompt content as shown, encoding the prompt from the previous node.  
   - Connect input from “Parse data from SerpBear”.

6. **Create a Baserow node**  
   - Name: “Save data to Baserow”  
   - Configure credentials with your Baserow API token.  
   - Set operation: Create row  
   - Database ID: your Baserow database ID (e.g., 121)  
   - Table ID: your Baserow table ID (e.g., 644)  
   - Fields to map:  
     - Date: Use expression `={{ DateTime.now().toFormat('yyyy-MM-dd') }}`  
     - Note: Map from AI response: `{{$json.choices[0].message.content}}`  
     - Blog: Static string of your blog/site name (e.g., “Rumjahn”)  
   - Connect input from “Send data to A.I. for analysis”.

7. **Connect the nodes** as follows:  
   - “When clicking ‘Test workflow’” → “Get data from SerpBear”  
   - “Schedule Trigger” → “Get data from SerpBear”  
   - “Get data from SerpBear” → “Parse data from SerpBear”  
   - “Parse data from SerpBear” → “Send data to A.I. for analysis”  
   - “Send data to A.I. for analysis” → “Save data to Baserow”

8. **Set credentials for:**  
   - SerpBear API (HTTP Header Auth) with your token.  
   - Openrouter API (HTTP Header Auth) with your bearer token.  
   - Baserow API with your API key.

9. **Create the Baserow database and table** beforehand with columns:  
   - Date (date type)  
   - Note (text or long text)  
   - Blog (single line text)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow checks weekly Google keyword rankings and generates AI-powered SEO insights without coding expertise.                         | See detailed usage and explanation at: [https://rumjahn.com/how-to-create-an-a-i-agent-to-analyze-serpbear-keyword-rankings-using-n8n-for-free-without-any-coding-skills-required/](https://rumjahn.com/how-to-create-an-a-i-agent-to-analyze-serpbear-keyword-rankings-using-n8n-for-free-without-any-coding-skills-required/) |
| User must create Baserow table with columns Date, Note, and Blog before saving results.                                                     | Workflow relies on existing Baserow database schema.                                                                                         |
| Openrouter API key requires HTTP Header Auth with "Authorization: Bearer {API key}" format.                                                  | Important to include space after "Bearer" in the header value.                                                                              |
| SerpBear API token and domain ID must be acquired from your SerpBear instance; token is set in HTTP Header Auth for API requests.           | See SerpBear documentation or admin dashboard for token management.                                                                         |

---

This document provides a complete and precise understanding of the SERPBear analytics workflow, enabling reproduction, modification, and troubleshooting by both advanced users and AI agents.