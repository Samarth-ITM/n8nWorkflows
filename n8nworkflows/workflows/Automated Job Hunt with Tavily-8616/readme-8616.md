Automated Job Hunt with Tavily

https://n8nworkflows.xyz/workflows/automated-job-hunt-with-tavily-8616


# Automated Job Hunt with Tavily

### 1. Workflow Overview

This workflow automates monitoring and processing new software engineering job postings from various trusted domains using the Tavily API. It periodically triggers a search for recent job listings, processes the raw data with AI-driven extraction and formatting, structures the job postings into detailed entries, aggregates the results, and finally emails a consolidated report. The workflow is logically divided into the following blocks:

- **1.1 Trigger and Search Execution:** Scheduled initiation and querying the Tavily API for recent job postings.
- **1.2 AI Processing and Extraction:** Using LangChain AI agents and OpenAI models to extract, clean, and structure job posting data.
- **1.3 Data Structuring and Aggregation:** Parsing the formatted data into individual job posting objects and combining all postings.
- **1.4 Email Notification:** Sending a formatted email with all job postings to a designated recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Search Execution

**Overview:**  
Starts the workflow on a weekly schedule and performs a customized search for software engineering roles posted within the last week via the Tavily API.

**Nodes Involved:**  
- Schedule Trigger  
- AI Agent  
- Search in Tavily  
- Simple Memory

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger (Base node)  
  - Role: Initiates the workflow weekly at 8 AM.  
  - Configuration: Interval set to trigger every week at hour 8.  
  - Inputs: None  
  - Outputs: Triggers AI Agent node.  
  - Edge Cases: Workflow won't start if the scheduler service is down or misconfigured.

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates the interaction between the workflow and the Tavily API tool. Acts as an autonomous research agent.  
  - Configuration:  
    - System message sets instructions for data extraction from job postings, requesting structured output with strict formatting and no hallucinations.  
    - Text prompt instructs the agent to act in three stages (not fully detailed in this node).  
  - Inputs: Trigger from Schedule Trigger; also receives API results from Search in Tavily via AI Tool connection.  
  - Outputs: Sends processed text to the Edit Fields node.  
  - Edge Cases:  
    - May fail or return incomplete data if Tavily API responses are malformed or missing expected fields.  
    - Requires correct API credentials; failure to authenticate will halt the flow.

- **Search in Tavily**  
  - Type: Tavily API Tool Node  
  - Role: Searches for job postings matching the query "Roles posted this week for Software Engineering."  
  - Configuration:  
    - Time range: last week  
    - Max results: 15  
    - Search depth: advanced  
    - Domains filtered to indeed.com, glassdoor.com, linkedin.com  
    - Includes raw content for downstream processing  
  - Inputs: Triggered via AI Agent’s ai_tool connection.  
  - Outputs: Provides raw job postings data back to AI Agent.  
  - Credentials: Tavily API credentials required.  
  - Edge Cases:  
    - API rate limits or authentication failures.  
    - No results returned if query is too restrictive or API issues.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains session state with a fixed session key, allowing the AI Agent to maintain context if needed.  
  - Inputs: Connected to AI Agent’s ai_memory input.  
  - Outputs: To AI Agent.  
  - Edge Cases:  
    - Memory size or session key mismatch could cause loss of context.

---

#### 2.2 AI Processing and Extraction

**Overview:**  
Transforms raw job posting data into a clean, structured summary using OpenAI language models with explicit formatting instructions.

**Nodes Involved:**  
- Edit Fields  
- Message a model  
- OpenAI Chat Model

**Node Details:**

- **Edit Fields**  
  - Type: Set Node  
  - Role: Reassigns the AI Agent’s output to a single field named `Response` for standardized input to the next model node.  
  - Configuration: Assigns `Response = {{$json.output}}`.  
  - Inputs: From AI Agent.  
  - Outputs: To Message a model node.  
  - Edge Cases: Empty or malformed AI Agent output can cause downstream errors.

- **Message a model**  
  - Type: LangChain OpenAI Node  
  - Role: Uses OpenAI GPT-4o-mini model to further format the job postings into a strict, numbered structure with detailed field extraction.  
  - Configuration:  
    - Message prompt includes explicit instructions not to paraphrase or hallucinate except for specified optional fields.  
    - Input is the `Response` field from Edit Fields.  
  - Inputs: Response field from Edit Fields.  
  - Outputs: Structured text blocks with job postings to Code node.  
  - Credentials: OpenAI API credentials required.  
  - Edge Cases: API timeout, invalid input formatting, or rate limiting.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model Node  
  - Role: (Configured but not directly connected in main flow) Possibly intended for alternative or backup AI interaction.  
  - Configuration: Model set to GPT-4.1-mini with text response format.  
  - Inputs: None directly connected in main flow.  
  - Outputs: None directly connected.  
  - Credentials: OpenAI API credentials.  
  - Edge Cases: Unused node; may cause confusion.

---

#### 2.3 Data Structuring and Aggregation

**Overview:**  
Parses the formatted text into individual job posting JSON objects and aggregates all postings into one array for sending.

**Nodes Involved:**  
- Code  
- Aggregate

**Node Details:**

- **Code**  
  - Type: Code Node (JavaScript)  
  - Role: Parses the multi-job posting text output into separate JSON objects, extracting each field by regex matching.  
  - Configuration:  
    - Joins all input messages’ content.  
    - Splits by '--- END JOB POSTING ---'.  
    - Extracts job title, URL, posting date, description, requirements, company name, company description, company website, location.  
    - Returns an array of structured job posting JSON objects with numbered posting_number.  
  - Inputs: From Message a model node.  
  - Outputs: Array of structured job postings to Aggregate node.  
  - Edge Cases: Parsing errors if formatting does not exactly match expected patterns; missing fields default to "Not specified".

- **Aggregate**  
  - Type: Aggregate Node  
  - Role: Combines all job posting JSON objects into a single array for easier processing and sending.  
  - Configuration: Aggregates all item data.  
  - Inputs: From Code node.  
  - Outputs: To Send a message node.  
  - Edge Cases: None significant; empty input results in empty array.

---

#### 2.4 Email Notification

**Overview:**  
Sends a consolidated email containing all structured job postings formatted for human readability.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - Type: Gmail Node  
  - Role: Sends a plaintext email to a specified recipient with all job postings summarized.  
  - Configuration:  
    - Recipient: may@tavily.com  
    - Subject: "New Jobs for this week!"  
    - Message body: Iterates over aggregated job postings, formatting each posting with clear labels and emojis for readability.  
  - Inputs: From Aggregate node.  
  - Outputs: None (terminal node).  
  - Credentials: Gmail OAuth2 credentials required.  
  - Edge Cases:  
    - Email sending failure due to auth errors or quota limits.  
    - Large payloads might cause timeout or truncation.

---

### 3. Summary Table

| Node Name         | Node Type                          | Functional Role                                  | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                              |
|-------------------|----------------------------------|-------------------------------------------------|------------------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger   | Schedule Trigger                  | Triggers workflow weekly at 8 AM                 | None                   | AI Agent              | Runs this workflow automatically at your chosen interval (daily or weekly)                             |
| AI Agent          | LangChain Agent                  | Orchestrates search and processing               | Schedule Trigger, Search in Tavily, Simple Memory | Edit Fields           |                                                                                                        |
| Search in Tavily   | Tavily API Tool                  | Performs job search with filters                  | AI Agent (ai_tool)      | AI Agent (ai_tool)     | Performs a web search for job postings based on your query. You can customize the search for specific job titles or keywords by changing the query, and limit results to a specific time range. Filters results by trusted domains and optional search depth |
| Simple Memory      | LangChain Memory Buffer Window   | Maintains AI session context                      | None                   | AI Agent (ai_memory)   |                                                                                                        |
| Edit Fields       | Set Node                        | Assigns AI Agent output to Response field         | AI Agent                | Message a model        | Collects all search results into a single structured bundle for processing                              |
| Message a model    | LangChain OpenAI Node           | Formats job postings into strict structured text  | Edit Fields             | Code                  | Restructures raw job postings into a clean, readable format. Extracts Job Title, Description, Requirements, Company info, and Location |
| OpenAI Chat Model  | LangChain Chat Model            | (Unused) AI language model node for chat          | None                   | None                  |                                                                                                        |
| Code              | Code Node (JavaScript)           | Parses formatted text into individual JSON objects| Message a model         | Aggregate              | Splits multiple job postings into individual objects and extracts each field for easy aggregation      |
| Aggregate         | Aggregate Node                   | Aggregates all job posting objects into one array | Code                    | Send a message         | Combines all processed postings into one array for sending in a single email                           |
| Send a message    | Gmail Node                      | Sends consolidated email with job postings         | Aggregate               | None                  | Sends one consolidated email with all job postings formatted for readability                           |
| Sticky Note       | Sticky Note                     | Informational note about Schedule Trigger          | None                   | None                  | Runs this workflow automatically at your chosen interval (daily or weekly)                             |
| Sticky Note1      | Sticky Note                     | Note on Search in Tavily configuration             | None                   | None                  | Performs a web search for job postings based on your query. You can customize the search for specific job titles or keywords by changing the query, and limit results to a specific time range. Filters results by trusted domains and optional search depth |
| Sticky Note2      | Sticky Note                     | Note on Edit Fields node                            | None                   | None                  | Collects all search results into a single structured bundle for processing                              |
| Sticky Note3      | Sticky Note                     | Note on Message a model node                        | None                   | None                  | Restructures raw job postings into a clean, readable format. Extracts Job Title, Description, Requirements, Company info, and Location |
| Sticky Note4      | Sticky Note                     | Note on Code node                                  | None                   | None                  | Splits multiple job postings into individual objects and extracts each field for easy aggregation      |
| Sticky Note5      | Sticky Note                     | Note on Aggregate node                             | None                   | None                  | Combines all processed postings into one array for sending in a single email                           |
| Sticky Note6      | Sticky Note                     | Note on Send a message node                        | None                   | None                  | Sends one consolidated email with all job postings formatted for readability                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger every week at 8 AM.

2. **Create AI Agent node**  
   - Type: LangChain Agent  
   - Configure system message to instruct the agent as an information extraction specialist for job postings.  
   - Input: Connect from Schedule Trigger node.  
   - Configure AI agent with:  
     - System message specifying the exact data fields to extract (job title, URL, posting date, description, requirements, company info, location).  
     - Text prompt specifying the workflow interaction stages and strict output formatting.  
   - Connect output to Edit Fields node.

3. **Create Search in Tavily node**  
   - Type: Tavily API Tool  
   - Query: "Roles posted this week for Software Engineering"  
   - Options:  
     - Time range: week  
     - Max results: 15  
     - Search depth: advanced  
     - Include domains: indeed.com, glassdoor.com, linkedin.com  
     - Include raw content: true  
   - Connect as ai_tool input to AI Agent node.  
   - Set Tavily API credentials.

4. **Create Simple Memory node**  
   - Type: LangChain Memory Buffer Window  
   - Session Key: "1234"  
   - Session ID Type: custom key  
   - Connect ai_memory output to AI Agent node.

5. **Create Edit Fields node**  
   - Type: Set Node  
   - Assign field "Response" with value `{{$json.output}}` (copy AI Agent output).  
   - Connect input from AI Agent output.  
   - Output to Message a model node.

6. **Create Message a model node**  
   - Type: LangChain OpenAI Node  
   - Model: GPT-4o-mini  
   - Message: Use provided prompt instructing the model to structure job postings strictly and not to paraphrase, except for allowed missing fields.  
   - Input: Use "Response" from Edit Fields node.  
   - Set OpenAI API credentials.  
   - Output to Code node.

7. **Create Code node**  
   - Type: Code Node (JavaScript)  
   - Paste the provided JS code that:  
     - Joins all input messages content.  
     - Splits by "--- END JOB POSTING ---".  
     - Extracts job fields with regex.  
     - Outputs array of structured job posting JSON objects.  
   - Input from Message a model node.  
   - Output to Aggregate node.

8. **Create Aggregate node**  
   - Type: Aggregate Node  
   - Aggregate all item data into one array.  
   - Input from Code node.  
   - Output to Send a message node.

9. **Create Send a message node**  
   - Type: Gmail Node  
   - Recipient email: may@tavily.com  
   - Subject: "New Jobs for this week!"  
   - Message: Use expression to iterate over aggregated postings and format each with labels and emojis, e.g.:  
     ```
     Job Posting <number> - <job_title>
     🔗 URL: <url>
     🗓️ Posting Date: <posting_date>
     📝 Job Description:
     <job_description>
     📌 Requirements:
     1. <requirement1>
     2. <requirement2>
     🏢 Company Name: <company_name>
     📝 Company Description: <company_description>
     🔗 Company Website: <company_website>
     📍 Location: <location>
     ---  
     ```
   - Set Gmail OAuth2 credentials.  
   - Connect input from Aggregate node.

10. **Add Sticky Notes** (optional but recommended)  
    - Add notes at key points describing the role of each block for documentation purposes.

11. **Activate the workflow and test**  
    - Ensure all credentials are valid.  
    - Run a manual test to validate the correct retrieval, processing, and emailing of job postings.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                       |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Workflow runs automatically on a weekly schedule to monitor software engineering job postings from trusted domains.          | Scheduling setup via Schedule Trigger node            |
| The Tavily API is used to source job postings filtered by domain and time range for reliability and relevance.               | Tavily API documentation: https://tavily.com/api     |
| AI-driven extraction ensures consistent formatting and eliminates manual data cleaning.                                       | Leveraging LangChain and OpenAI GPT models            |
| Email formatting includes emojis and clear labels for readability in plain text emails.                                       | Gmail node configuration                              |
| Potential expansions: add more domains, adjust search queries, or integrate follow-up actions (e.g., notifications).          | Consider workflow scalability and API rate limits    |

---

**Disclaimer:** The content provided is derived exclusively from an automated n8n workflow utilizing the Tavily API and OpenAI services. All data processed is public and lawful; no assumptions or unauthorized data collection occurs. The workflow adheres strictly to content and usage policies.