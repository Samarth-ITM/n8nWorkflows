Find & Qualify Funded Leads with BrowserAct & Gemini

https://n8nworkflows.xyz/workflows/find---qualify-funded-leads-with-browseract---gemini-9887


# Find & Qualify Funded Leads with BrowserAct & Gemini

### 1. Workflow Overview

This workflow automates the process of finding and qualifying newly funded companies by scraping web articles, analyzing them with AI, and updating a Google Sheet with the results. It is targeted at investment teams, lead generation specialists, or business intelligence users who want to automate the discovery of companies that have recently announced funding rounds.

The workflow is logically divided into four main blocks:

- **1.1 Input Acquisition**: Fetches keywords and geographic data from a Google Sheet to drive the scraping tasks.
- **1.2 Web Scraping and Data Collection**: Uses BrowserAct nodes to run two parallel scraping workflows for different funding series keywords, waits for their completion, and merges the results.
- **1.3 AI Processing and Data Transformation**: Applies Google Gemini-powered AI to analyze scraped articles, extracting companies and funding details; then processes and filters the AI output.
- **1.4 Output and Notification**: Appends or updates the qualified leads into a Google Sheet and sends a Slack notification confirming the update.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Acquisition

**Overview:**  
This block retrieves keywords (e.g., funding Series A and B) and location data from a Google Sheet, preparing input parameters to run the scraping workflows in batches.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet  
- Loop Over Items  
- Sticky Note5 (annotation)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually.  
  - Configuration: Default, no parameters.  
  - Input: None  
  - Output: Triggers "Get row(s) in sheet"  
  - Edge Cases: No trigger means no run; replaceable by Cron for automation.

- **Get row(s) in sheet**  
  - Type: Google Sheets node  
  - Role: Reads rows from a specific sheet containing keywords and geographic info.  
  - Configuration:  
    - Document ID: Points to a specific Google Sheet (Test For BrowserAct)  
    - Sheet Name: "Keywords For Funding Announcement to Lead List (TechCrunch)"  
  - Input: Trigger node  
  - Output: Passes data to "Loop Over Items"  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Google API failures, empty data, rate-limiting.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each row individually to run scraping tasks per keyword/location pair.  
  - Configuration: Default batch size (usually 1)  
  - Input: Rows from Google Sheets  
  - Output: Splits data for parallel scraping workflows  
  - Edge Cases: Large datasets may slow execution; handle empty inputs gracefully.

- **Sticky Note5**  
  - Content: Explains this block’s purpose: "Get input keyword and location for the loop from the list of keywords and geo inside the google sheet."

#### 1.2 Web Scraping and Data Collection

**Overview:**  
This block runs two parallel BrowserAct scraping workflows for two different funding series (Series A and Series B), waits for their completion, and merges their results.

**Nodes Involved:**  
- Run a workflow Series 1  
- Get workflow Series1  
- Run a workflow Series 2  
- Get workflow Series 2  
- Merge  
- Sticky Note - Scraping Stage (annotation)

**Node Details:**  

- **Run a workflow Series 1**  
  - Type: BrowserAct workflow node  
  - Role: Starts a BrowserAct scraping job with Series A keyword and location from current batch item.  
  - Configuration:  
    - workflowId: 56655803760865399 (BrowserAct scraping workflow)  
    - Input Parameters: KeyWord from "keyword Series A" column, Location from Geo column  
    - Additional: saveBrowserData disabled  
  - Input: Batch item from Loop Over Items  
  - Output: Task ID for monitoring  
  - Credentials: BrowserAct API  
  - Edge Cases: Invalid workflow ID, API auth failure, input parameter errors.

- **Get workflow Series1**  
  - Type: BrowserAct workflow node  
  - Role: Polls the BrowserAct job started by "Run a workflow Series 1" until completion.  
  - Configuration:  
    - taskId from previous node’s output  
    - waitForFinish: true (blocks until finished)  
    - maxWaitTime: 900 seconds (15 minutes)  
    - pollingInterval: 30 seconds  
  - Input: Task ID from "Run a workflow Series 1"  
  - Output: Scraped data  
  - Credentials: BrowserAct API  
  - Edge Cases: Timeout, job failure, network issues.

- **Run a workflow Series 2**  
  - Similar to Series 1 node but uses "keyword Series B" for input parameters.  
  - Input: Batch item from Loop Over Items.

- **Get workflow Series 2**  
  - Similar to Get workflow Series1, waits on Series 2 job completion.

- **Merge**  
  - Type: Merge node  
  - Role: Combines outputs from Series 1 and Series 2 scraping results into a single data stream.  
  - Configuration: Default (merge inputs)  
  - Input: Outputs from "Get workflow Series 2" and "Get workflow Series1"  
  - Output: Merged data passed to AI Agent  
  - Edge Cases: Empty inputs, data format mismatches.

- **Sticky Note - Scraping Stage**  
  - Content: Details about the scraping step, emphasizing the dual BrowserAct nodes working together to reduce false positives by merging Series A and B results.

#### 1.3 AI Processing and Data Transformation

**Overview:**  
This block uses an AI Agent powered by Google Gemini to analyze the merged articles, extracting company names and funding details. The output is structured, filtered for valid entries, and transformed for sheet insertion.

**Nodes Involved:**  
- Gemini l (Google Gemini LM Chat node)  
- AI Agent (LangChain agent node)  
- Structured Output (Output parser for structured JSON)  
- Code in JavaScript (transforms AI output for Google Sheets)  
- If (filter to exclude "No Company" entries)  
- Sticky Note - AI Stage  
- Sticky Note - Processing Stage

**Node Details:**  

- **Gemini l**  
  - Type: LangChain LM Chat Google Gemini node  
  - Role: Provides AI large language model backend for the AI Agent node.  
  - Configuration: No parameters; credentials attached.  
  - Credentials: Google Palm API (Google Gemini)  
  - Input: Connected via AI Agent node interface  
  - Output: AI-generated responses  
  - Edge Cases: API quota exceeded, latency, malformed responses.

- **AI Agent**  
  - Type: LangChain AI Agent node  
  - Role: Receives merged articles, prompts the LLM to identify companies that announced funding rounds, returning structured data.  
  - Configuration:  
    - Custom prompt instructing to parse the article text for company name, invested object, and URL.  
    - Returns default "No Company" map if none found.  
    - Uses Structured Output Parser node for consistent JSON format.  
  - Input: Merged data from scraping  
  - Output: AI-processed structured data  
  - Edge Cases: AI hallucinations, incomplete data, prompt failure.

- **Structured Output**  
  - Type: LangChain Output Parser Structured node  
  - Role: Enforces AI output JSON schema containing Company, InvestedOn, and Url fields.  
  - Configuration: Example JSON schema provided for validation.  
  - Input: AI Agent output  
  - Output: Clean structured JSON entries  
  - Edge Cases: Schema mismatch, parsing errors.

- **Code in JavaScript**  
  - Type: Code node  
  - Role: Transforms the AI output list of maps into an array of individual n8n items with a "text" key wrapping each map, enabling proper downstream processing.  
  - Key logic: Loops original AI output array, wraps each item in `{ json: { text: item } }`.  
  - Input: AI Agent output  
  - Output: Items formatted for Sheet insertion  
  - Edge Cases: Unexpected input format, empty arrays.

- **If**  
  - Type: Conditional filter node  
  - Role: Filters out entries where any of Company, Url, or InvestedOn equals "No Company".  
  - Configuration: OR condition on these fields equal to "No Company".  
  - Input: Code node output  
  - Output: Valid entries proceed; invalid filtered out.  
  - Edge Cases: Case sensitivity issues, missing fields.

- **Sticky Note - AI Stage**  
  - Content: Emphasizes the AI Agent as the brain, importance of clear prompting, and structured output for reliability.

- **Sticky Note - Processing Stage**  
  - Content: Highlights the Code node’s role in adapting AI lists for n8n and the If node’s guard logic to permit only qualified data forward.

#### 1.4 Output and Notification

**Overview:**  
This block appends or updates qualified funding leads into a Google Sheet and sends a Slack notification about the update.

**Nodes Involved:**  
- Append or update row in sheet (Google Sheets)  
- Send a message (Slack)  
- Sticky Note - Output Stage

**Node Details:**  

- **Append or update row in sheet**  
  - Type: Google Sheets node  
  - Role: Inserts or updates rows in the "Funding Announcement to Lead List (TechCrunch)" sheet with company, investedOn, and URL details.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Matching Column: Company (to prevent duplicates)  
    - Columns mapped from the `text` object of filtered AI results.  
  - Input: Validated AI output from If node  
  - Output: Passes control to Slack notification  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Sheet API rate limits, concurrent writes.

- **Send a message**  
  - Type: Slack node  
  - Role: Sends a notification message to a Slack channel indicating that the lead data has been updated.  
  - Configuration:  
    - Channel: Specified by channel ID (e.g., all-browseract-workflow-test)  
    - Text: "The data for the lead announcement has been updated"  
  - Input: After Google Sheets update  
  - Credentials: Slack OAuth2  
  - Edge Cases: Slack API downtime, invalid channel.

- **Sticky Note - Output Stage**  
  - Content: Describes this final stage and recommends appendOrUpdate operation with matching on Company to avoid duplicates.

---

### 3. Summary Table

| Node Name                   | Node Type                       | Functional Role                               | Input Node(s)                    | Output Node(s)                 | Sticky Note                                                                                                      |
|-----------------------------|--------------------------------|-----------------------------------------------|---------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Workflow manual start trigger                   | None                            | Get row(s) in sheet            |                                                                                                                  |
| Get row(s) in sheet          | Google Sheets                  | Fetches keywords and location data              | When clicking ‘Execute workflow’ | Loop Over Items                |                                                                                                                  |
| Loop Over Items              | SplitInBatches                 | Iterates over each keyword/location pair       | Get row(s) in sheet             | Run a workflow Series 1, Run a workflow Series 2 | Sticky Note5: "Get input keyword and location from Google Sheet"                                                |
| Run a workflow Series 1      | BrowserAct workflow runner     | Starts scraping job for Series A keyword       | Loop Over Items                 | Get workflow Series1           | Sticky Note - Scraping Stage: "Dual BrowserAct nodes fetch data for keywords..."                                 |
| Get workflow Series1         | BrowserAct workflow status     | Waits for completion of Series 1 scraping job  | Run a workflow Series 1         | Merge                        | Sticky Note - Scraping Stage                                                                                        |
| Run a workflow Series 2      | BrowserAct workflow runner     | Starts scraping job for Series B keyword       | Loop Over Items                 | Get workflow Series 2          | Sticky Note - Scraping Stage                                                                                        |
| Get workflow Series 2        | BrowserAct workflow status     | Waits for completion of Series 2 scraping job  | Run a workflow Series 2         | Merge                        | Sticky Note - Scraping Stage                                                                                        |
| Merge                       | Merge                         | Combines Series 1 & 2 scraping results          | Get workflow Series1, Get workflow Series 2 | Gemini l                      |                                                                                                                  |
| Gemini l                    | LangChain LM Chat Google Gemini | Provides Google Gemini LLM for AI Agent         | Merge                          | AI Agent                      | Sticky Note - AI Stage: "AI Agent is the brain, uses Google Gemini..."                                           |
| AI Agent                   | LangChain AI Agent             | Processes articles to extract funding leads     | Gemini l                      | Structured Output             | Sticky Note - AI Stage                                                                                             |
| Structured Output           | LangChain Output Parser        | Enforces structured JSON output from AI Agent   | AI Agent                      | Code in JavaScript            | Sticky Note - AI Stage                                                                                             |
| Code in JavaScript          | Code                          | Transforms AI output list to n8n item format    | Structured Output              | If                           | Sticky Note - Processing Stage: "Code node processes AI lists for downstream nodes."                             |
| If                         | If                            | Filters out entries with 'No Company'           | Code in JavaScript             | Append or update row in sheet | Sticky Note - Processing Stage: "If node filters invalid AI results."                                            |
| Append or update row in sheet | Google Sheets                  | Appends/updates qualified leads in Google Sheet | If                            | Send a message               | Sticky Note - Output Stage: "Final output stage, prevent duplicates with matching on Company."                   |
| Send a message             | Slack                         | Sends Slack notification about data update      | Append or update row in sheet  | Loop Over Items (for next batch run) |                                                                                                                  |
| Sticky Note - Intro        | Sticky Note                   | Overview and instructions for entire workflow   | None                          | None                         | Full workflow overview, requirements, and links to help resources                                                |
| Sticky Note - How to Use   | Sticky Note                   | Usage instructions                               | None                          | None                         | Steps for credential setup and workflow activation                                                               |
| Sticky Note - Need Help    | Sticky Note                   | Help and tutorial video links                     | None                          | None                         | Contains multiple YouTube tutorial links                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: To manually start the workflow.

2. **Add Google Sheets Node to Read Input Data**  
   - Name: "Get row(s) in sheet"  
   - Operation: Read rows  
   - Document ID: Your Google Sheet ID with keywords and Geo data  
   - Sheet Name: The sheet containing keyword Series A, B, and Geo columns  
   - Credentials: Setup Google Sheets OAuth2 credentials.

3. **Add SplitInBatches Node**  
   - Name: "Loop Over Items"  
   - Default batch size (usually 1)  
   - Connect input from "Get row(s) in sheet"  
   - Purpose: To process each keyword/location row individually.

4. **Add Two BrowserAct Workflow Nodes**  
   - Name: "Run a workflow Series 1" and "Run a workflow Series 2"  
   - Operation: Run a BrowserAct scraping workflow  
   - Workflow ID: Your BrowserAct scraping workflow ID (e.g., "56655803760865399")  
   - Input Parameters:  
     - Series 1: KeyWord from current item’s "keyword Series A", Location from "Geo"  
     - Series 2: KeyWord from current item’s "keyword Series B", Location from "Geo"  
   - Disable "Save Browser Data" for performance  
   - Credentials: Setup BrowserAct API credentials.

5. **Add Two BrowserAct Workflow Status Nodes**  
   - Name: "Get workflow Series1" and "Get workflow Series 2"  
   - Operation: Get task status  
   - Parameters:  
     - taskId from respective "Run a workflow" nodes  
     - waitForFinish: true  
     - maxWaitTime: 900 seconds (15 minutes)  
     - pollingInterval: 30 seconds  
   - Credentials: BrowserAct API credentials.

6. **Add a Merge Node**  
   - Name: "Merge"  
   - Operation: Merge inputs from both "Get workflow" nodes  
   - Purpose: Combine Series 1 and 2 scraped data into one stream.

7. **Add LangChain LM Chat Node (Google Gemini)**  
   - Name: "Gemini l"  
   - Credentials: Google Palm API (Gemini)  
   - No specific parameters needed.

8. **Add LangChain AI Agent Node**  
   - Name: "AI Agent"  
   - Set prompt to instruct analyzing articles to find funding announcements.  
   - Input: Connect LM Chat node ("Gemini l") as language model  
   - Configure output parser to the Structured Output node (next step).

9. **Add LangChain Structured Output Parser Node**  
   - Name: "Structured Output"  
   - Schema example:  
     ```json
     [
       {
         "Company": "<String>",
         "InvestedOn": "<String>",
         "Url": "<String>"
       }
     ]
     ```
   - Connect as AI Agent output parser.

10. **Add Code Node (JavaScript)**  
    - Name: "Code in JavaScript"  
    - Purpose: Transform AI output list into n8n items format.  
    - JavaScript code:  
      ```javascript
      const originalList = $input.first().json.output;
      const outputItems = [];
      for (const item of originalList) {
        outputItems.push({ json: { text: item } });
      }
      return outputItems;
      ```
    - Input: AI Agent output.

11. **Add If Node**  
    - Name: "If"  
    - Condition: OR any of these fields equal "No Company":  
      - `{{$json.text.Company}} == "No Company"`  
      - `{{$json.text.Url}} == "No Company"`  
      - `{{$json.text.InvestedOn}} == "No Company"`  
    - True branch: Discard invalid entries  
    - False branch: Pass valid entries forward.

12. **Add Google Sheets Node to Append or Update Rows**  
    - Name: "Append or update row in sheet"  
    - Operation: appendOrUpdate  
    - Document ID: Your target Google Sheet ID for leads  
    - Sheet Name: "Funding Announcement to Lead List (TechCrunch)" or equivalent  
    - Columns: Map from `{{$json.text.Url}}`, `{{$json.text.Company}}`, `{{$json.text.InvestedOn}}`  
    - Matching Columns: "Company" to avoid duplicates  
    - Credentials: Google Sheets OAuth2.

13. **Add Slack Node to Send Notification**  
    - Name: "Send a message"  
    - Channel: Slack channel ID to notify  
    - Text: "The data for the lead announcement has been updated"  
    - Credentials: Slack OAuth2.

14. **Connect Nodes Sequentially:**  
    - Manual Trigger → Get row(s) in sheet → Loop Over Items →  
      (in parallel) Run a workflow Series 1 → Get workflow Series1 and Run a workflow Series 2 → Get workflow Series 2 → Merge → Gemini l → AI Agent → Structured Output → Code in JavaScript → If → Append or update row in sheet → Send a message

15. **Add Sticky Notes** (optional but recommended)  
    - Add descriptive sticky notes near each block for clarity, using the content from the workflow.

16. **Credential Setup:**  
    - BrowserAct API credentials for BrowserAct nodes  
    - Google Gemini (Google Palm API) credentials for AI nodes  
    - Google Sheets OAuth2 for Sheets nodes  
    - Slack OAuth2 for Slack notification node

17. **Test and Activate:**  
    - Manually trigger the workflow to verify correct operation.  
    - Optionally, replace the manual trigger with a Cron node for scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Try It Out! This n8n template helps you find new investment leads by scraping and analyzing articles for funding news. | Sticky Note - Intro (workflow overview and instructions)                                           |
| Join the Discord community for help or visit the BrowserAct blog for tutorials and updates.                            | [Discord](https://discord.com/invite/UpnCKd7GaU), [Blog](https://www.browseract.com/blog)          |
| How to Find Your BrowserAct API Key & Workflow ID (YouTube tutorial).                                                  | https://www.youtube.com/watch?v=pDjoZWEsZlE                                                        |
| How to Connect n8n to BrowserAct (YouTube tutorial).                                                                   | https://www.youtube.com/watch?v=RoYMdJaRdcQ                                                        |
| How to Use & Customize BrowserAct Templates (YouTube tutorial).                                                        | https://www.youtube.com/watch?v=CPZHFUASncY                                                        |
| How to Use the BrowserAct n8n Community Node (YouTube tutorial).                                                       | https://youtu.be/j0Nlba2pRLU                                                                      |
| How to Automatically Find Leads from Funding News (n8n Workflow example video).                                        | https://youtu.be/zMO_1EC1RVM                                                                      |
| The `appendOrUpdate` operation in Google Sheets node helps avoid duplicate entries by matching on the Company column. | Sticky Note - Output Stage                                                                          |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected information. All processed data is legal and public.