Source Top GitHub Talent by Language & Location with Gemini AI and BrowserAct

https://n8nworkflows.xyz/workflows/source-top-github-talent-by-language---location-with-gemini-ai-and-browseract-9891


# Source Top GitHub Talent by Language & Location with Gemini AI and BrowserAct

### 1. Workflow Overview

This workflow, titled **"AI-Powered GitHub Talent Sourcing (by Language & Location) to Google Sheet"**, automates the process of sourcing, analyzing, scoring, and storing top GitHub contributors based on specified programming languages and locations. It is designed for talent acquisition teams seeking to identify and rank potential developer candidates efficiently.

The workflow is logically divided into three main blocks:

- **1.1 Automated Sourcing:** Periodically triggers and initiates a web scraping task to collect GitHub contributor data filtered by language, location, and repository/publication criteria using BrowserAct.

- **1.2 AI Processing and Scoring:** Parses the raw scraped data into individual candidate profiles, then leverages a custom AI Agent with Google Gemini to analyze each profile’s resume details and calculate a composite score based on followers, repositories, and resume quality.

- **1.3 Data Persistence and Alerts:** Saves the scored and structured candidate data into a Google Sheet for record-keeping and sends Slack notifications about workflow success or failure to keep the team informed.

---

### 2. Block-by-Block Analysis

#### 2.1 Automated Sourcing

**Overview:**  
This block initiates the workflow on a schedule and performs the GitHub scraping task using BrowserAct based on user-defined parameters: programming language, location, number of pages, and minimum public repositories.

**Nodes Involved:**  
- Schedule Trigger  
- Run a workflow task (BrowserAct)  
- Get details of a workflow task (BrowserAct)  
- Send a message (Slack - error notification)

**Node Details:**  

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Starts the workflow automatically every hour to keep sourcing up-to-date.  
  - Configuration: Interval set to hourly (every 1 hour).  
  - Input: None (trigger node)  
  - Output: Runs the "Run a workflow task" node.  
  - Failure modes: None expected; if disabled, sourcing won't run automatically.

- **Run a workflow task (BrowserAct)**  
  - Type: Community integration node for BrowserAct  
  - Role: Executes a pre-configured BrowserAct scraping template workflow to extract GitHub contributor data.  
  - Configuration:  
    - Workflow ID: references a BrowserAct template named “Source Top GitHub Contributors by Language & Location”.  
    - Parameters: Language = Python, Location = Berlin, Total_Page = 2, Public_Repositories = 5.  
    - saveBrowserData: false (browser session data not saved).  
  - Credentials: BrowserAct API credentials required.  
  - Input: Trigger from Schedule Trigger.  
  - Output: Task ID passed to “Get details of a workflow task”.  
  - Failure modes: API authentication errors, invalid workflow ID, network timeouts.

- **Get details of a workflow task (BrowserAct)**  
  - Type: BrowserAct node to poll task status and retrieve results.  
  - Role: Waits for the scraping task to complete, then fetches the output data.  
  - Configuration:  
    - Operation: getTask  
    - Task ID: dynamically set from previous node output.  
    - maxWaitTime: 600 seconds  
    - waitForFinish: true  
    - pollingInterval: 20 seconds  
  - Credentials: BrowserAct API credentials.  
  - Input: Task ID from "Run a workflow task".  
  - Output: Scraped data (JSON string) or error.  
  - On error: workflow set to continue on error (prevents workflow stop).  
  - Failure modes: Task timeout, API errors, malformed data, network issues.

- **Send a message (Slack - error notification)**  
  - Type: Slack node  
  - Role: Sends an alert message to Slack channel if scraping task fails.  
  - Configuration:  
    - Channel: preselected Slack channel “all-browseract-workflow-test”  
    - Text: "BrowserAct Workflow Faces Problem"  
  - Credentials: Slack OAuth2 credentials.  
  - Input: Error output from "Get details of a workflow task".  
  - Failure modes: Slack API issues, invalid token, channel not found.

---

#### 2.2 AI Processing and Scoring

**Overview:**  
This block converts the raw scraped JSON into individual candidate items, then uses a custom AI Agent powered by Google Gemini to analyze and score each candidate’s profile based on a proprietary formula considering followers, total repositories, and resume quality.

**Nodes Involved:**  
- Code in JavaScript  
- AI Agent (LangChain agent with Google Gemini)  
- Structured Output Parser  
- Gemini Chat (Google Gemini LM Chat, invoked by AI Agent)

**Node Details:**  

- **Code in JavaScript**  
  - Type: Code execution node  
  - Role: Parses the JSON string output from BrowserAct into an array of individual candidate objects as separate n8n items.  
  - Configuration logic:  
    - Extracts JSON string from `$input.first().json.output.string`.  
    - Throws error if string is missing or empty.  
    - Parses string into array; throws error if parsing fails or data is not an array.  
    - Maps each array element to `{ json: item }` for downstream nodes.  
  - Input: Output JSON string from "Get details of a workflow task".  
  - Output: Split candidate profiles as separate items.  
  - Failure modes: Missing or malformed JSON string, parsing errors, unexpected data structure.

- **AI Agent (LangChain agent)**  
  - Type: LangChain agent node with Google Gemini integration  
  - Role: Processes each candidate item by analyzing their summary/resume, assigns a ResumeScore (0-1000), calculates FinalScore based on formula:  
    `FinalScore = (Followers × 0.5) + (TotalRepo × 3) + ResumeScore`  
  - Handles insufficient or missing data by assigning zero scores and continuing without error.  
  - Sends structured JSON output for each candidate including Name, Location, TotalRepo, Followers, URL, and Score.  
  - Configuration: Custom prompt defining the scoring logic and output structure.  
  - Input: Candidate items from the JavaScript node.  
  - Output: AI-analyzed and scored candidate data passed to Structured Output Parser.  
  - Failure modes: AI API limits, malformed input data, unexpected API responses, credential expiration.

- **Structured Output Parser**  
  - Type: LangChain output parser node  
  - Role: Ensures AI Agent’s output matches the expected JSON schema and extracts parsed structured data.  
  - Configuration: JSON schema example with fields: Name, Location, TotalRepo, Followers, URL, Score (all strings).  
  - Input: AI Agent output.  
  - Output: Parsed structured candidate profiles.  
  - Failure modes: Output does not conform to schema, parser errors.

- **Gemini Chat (Google Gemini LM Chat)**  
  - Type: Language Model Chat node  
  - Role: Invoked internally by AI Agent as the language model backend for natural language understanding and scoring.  
  - Configuration: Uses Google Gemini (PaLM) API credentials.  
  - Input: AI Agent prompts.  
  - Output: AI Agent receives processed responses.  
  - Failure modes: API authentication, request timeouts, quota limits.

---

#### 2.3 Data Persistence and Alerts

**Overview:**  
This block appends or updates the scored candidate data into a Google Sheet and sends Slack notifications to confirm successful updates.

**Nodes Involved:**  
- Append or update row in sheet (Google Sheets)  
- Send a message1 (Slack - success notification)

**Node Details:**  

- **Append or update row in sheet (Google Sheets)**  
  - Type: Google Sheets node  
  - Role: Saves structured candidate data into a designated Google Sheet, matching rows by candidate Name to prevent duplicates.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Matching column: Name  
    - Sheet Name & Document ID: Points to "Source Top GitHub Contributors by Language & Location" sheet in a specific Google Sheet document.  
    - Fields mapped: URL, Name, Score, Followers, Location, TotalRepo (all string type) from AI output.  
  - Credentials: Google Sheets OAuth2 credentials.  
  - Input: Parsed structured candidate data from Structured Output Parser via AI Agent.  
  - Output: Triggers Slack success message.  
  - Failure modes: Credential expiration, API rate limits, sheet access errors, mismatched data types.

- **Send a message1 (Slack - success notification)**  
  - Type: Slack node  
  - Role: Sends confirmation message to Slack channel after candidates are successfully saved.  
  - Configuration:  
    - Channel: same as error Slack node ("all-browseract-workflow-test")  
    - Text: "GitHub Contributors Updated"  
  - Credentials: Slack OAuth2 credentials.  
  - Input: Success output from Google Sheets node.  
  - Failure modes: Slack API errors, invalid channel or token.

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                         | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                                         |
|----------------------------|-------------------------------------|---------------------------------------|-----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | n8n-nodes-base.scheduleTrigger       | Initiates workflow hourly             | None                        | Run a workflow task             | 🔎 1. Automated Sourcing: Starts workflow automatically every hour                                                 |
| Run a workflow task        | n8n-nodes-browseract-workflows.browserAct | Executes GitHub scraping task          | Schedule Trigger            | Get details of a workflow task  | 🔎 1. Automated Sourcing: Executes BrowserAct GitHub search; error output connected for alerts                      |
| Get details of a workflow task | n8n-nodes-browseract-workflows.browserAct | Polls BrowserAct task for results     | Run a workflow task         | Code in JavaScript, Send a message | 🔎 1. Automated Sourcing: Polls for scraping completion; continue on error; recommended to connect alert node       |
| Send a message             | n8n-nodes-base.slack                 | Sends Slack error alert                | Get details of a workflow task (error) | None                        | 🔎 1. Automated Sourcing: Alerts team on scraping failure                                                          |
| Code in JavaScript         | n8n-nodes-base.code                  | Parses raw JSON string to individual items | Get details of a workflow task | AI Agent                      | 🧠 2. AI Scoring Engine: Parses scraped data into items for AI processing                                          |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent       | Analyzes candidate, calculates score | Code in JavaScript          | Append or update row in sheet   | 🧠 2. AI Scoring Engine: Custom AI scoring of candidates using Google Gemini                                       |
| Structured Output Parser   | @n8n/n8n-nodes-langchain.outputParserStructured | Validates and structures AI output   | AI Agent                   | AI Agent (loopback outputParser) | 🧠 2. AI Scoring Engine: Ensures AI output matches expected schema                                                 |
| Gemini Chat               | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Language model backend for AI Agent  | AI Agent                   | AI Agent                       | 🧠 2. AI Scoring Engine: Provides natural language processing via Google Gemini                                    |
| Append or update row in sheet | n8n-nodes-base.googleSheets          | Saves candidate data to Google Sheet  | AI Agent                   | Send a message1                | 💾 3. Save Ranked Candidates: Saves scored candidates; uses appendOrUpdate to avoid duplicates                      |
| Send a message1            | n8n-nodes-base.slack                 | Sends Slack success notification      | Append or update row in sheet | None                        | 💾 3. Save Ranked Candidates: Notifies team when GitHub contributors list is updated                               |
| Sticky Note - Intro       | n8n-nodes-base.stickyNote             | Documentation - workflow intro        | None                        | None                          | Introductory explanation and requirements                                                                        |
| Sticky Note - How to Use  | n8n-nodes-base.stickyNote             | Documentation - usage instructions    | None                        | None                          | Step-by-step usage guide                                                                                           |
| Sticky Note - Need Help   | n8n-nodes-base.stickyNote             | Documentation - help resources        | None                        | None                          | Helpful video links and community resources                                                                        |
| Sticky Note - Sourcing Stage | n8n-nodes-base.stickyNote             | Documentation - sourcing block info   | None                        | None                          | Explains the scheduling and BrowserAct scraping nodes                                                             |
| Sticky Note - AI Scoring Engine | n8n-nodes-base.stickyNote             | Documentation - AI scoring block info | None                        | None                          | Describes the data parsing and AI scoring process                                                                  |
| Sticky Note - Saving Candidates | n8n-nodes-base.stickyNote             | Documentation - saving & alert block  | None                        | None                          | Details Google Sheets usage and Slack notifications                                                                |
| Sticky Note                | n8n-nodes-base.stickyNote             | Empty placeholder                     | None                        | None                          |                                                                                                                    |
| Sticky Note1               | n8n-nodes-base.stickyNote             | Empty placeholder                     | None                        | None                          |                                                                                                                    |
| Sticky Note2               | n8n-nodes-base.stickyNote             | Empty placeholder                     | None                        | None                          |                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run every 1 hour.

2. **Add a Run a workflow task node (BrowserAct)**  
   - Type: BrowserAct workflow node  
   - Configure with your BrowserAct API credentials.  
   - Set the workflow ID to your BrowserAct template “Source Top GitHub Contributors by Language & Location.”  
   - Input parameters:  
     - Language: e.g., "Python"  
     - Location: e.g., "Berlin"  
     - Total_Page: "2"  
     - Public_Repositories: "5"  
   - Disable saveBrowserData.

3. **Connect Schedule Trigger output to Run a workflow task input.**

4. **Add Get details of a workflow task node (BrowserAct)**  
   - Type: BrowserAct node  
   - Credentials: BrowserAct API credentials.  
   - Operation: getTask  
   - Task ID: use expression `{{$json["id"]}}` from Run a workflow task output.  
   - maxWaitTime: 600 seconds  
   - waitForFinish: true  
   - pollingInterval: 20 seconds  
   - Set error handling to continue on error (so workflow continues even if this node fails).

5. **Connect Run a workflow task main output to Get details of a workflow task input.**

6. **Add a Code node (JavaScript)**  
   - Type: Code  
   - Paste JavaScript code to parse the JSON string from the BrowserAct output:  
     - Extract JSON string from `$input.first().json.output.string`  
     - Validate presence and parse  
     - Throw errors if missing or malformed  
     - Split array elements into separate items for downstream processing

7. **Connect Get details of a workflow task main output to Code node input.**

8. **Add AI Agent node (LangChain agent)**  
   - Type: LangChain agent node  
   - Set credentials for Google Gemini (PaLM) API.  
   - Prompt configuration:  
     - Instruct to analyze candidate summary/resume (field `Summary`) and score between 0-1000 ResumeScore.  
     - Calculate FinalScore: `(Followers * 0.5) + (TotalRepo * 3) + ResumeScore`.  
     - If data insufficient, assign 0 to ResumeScore and FinalScore, continue without error.  
     - Output all candidate fields plus FinalScore in structured JSON format: Name, Location, TotalRepo, Followers, URL, Score.  
   - Enable output parser option.

9. **Connect Code node main output to AI Agent input.**

10. **Add Structured Output Parser node**  
    - Type: LangChain structured output parser  
    - JSON schema example:  
      ```json
      {
        "Name": "<String>",
        "Location": "<String>",
        "TotalRepo": "<String>",
        "Folowers": "<String>",
        "URL": "<String>",
        "Score": "<String>"
      }
      ```  
    - Connect AI Agent’s output parser slot to this node’s input.

11. **Connect Structured Output Parser output back to AI Agent’s output parser input slot (for structured parsing).**

12. **Add Google Sheets node (Append or update row)**  
    - Type: Google Sheets node  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Operation: appendOrUpdate  
    - Sheet name: select your target sheet (e.g., “Source Top GitHub Contributors by Language & Location”).  
    - Document ID: your Google Sheet document ID.  
    - Matching columns: ["Name"] to avoid duplicates.  
    - Map columns: URL, Name, Score, Followers, Location, TotalRepo from AI Agent output fields.  
    - Convert all data to strings.

13. **Connect AI Agent main output to Google Sheets node input.**

14. **Add Slack node (Send a message1) for success notification**  
    - Type: Slack node  
    - Credentials: Slack OAuth2 credentials.  
    - Channel: pre-select your team Slack channel.  
    - Text: "GitHub Contributors Updated".  
    - Connect Google Sheets node main output to this Slack node input.

15. **Add Slack node (Send a message) for error notification**  
    - Type: Slack node  
    - Credentials: Slack OAuth2 credentials.  
    - Channel: same Slack channel as above.  
    - Text: "BrowserAct Workflow Faces Problem".  
    - Connect Get details of a workflow task node’s error output to this Slack node input.

16. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This n8n template is a powerful talent sourcing engine that finds, analyzes, and scores GitHub contributors using a custom AI formula.                             | Sticky Note - Intro                                                                                     |
| Requirements include BrowserAct API account, BrowserAct n8n Community Node, Google Gemini API, and Google Sheets credentials.                                      | Sticky Note - Intro                                                                                     |
| For help, join the [Discord](https://discord.com/invite/UpnCKd7GaU) or visit the [BrowserAct Blog](https://www.browseract.com/blog).                                | Sticky Note - Intro                                                                                     |
| Setup instructions: configure credentials, use the correct BrowserAct template, customize search parameters, activate workflow, and adjust schedule trigger as needed.| Sticky Note - How to Use                                                                                |
| Helpful instructional videos on BrowserAct API key retrieval, n8n connection, template customization, and workflow automation are available on YouTube.             | Sticky Note - Need Help                                                                                  |
| Scheduling and BrowserAct scraping nodes form the automated sourcing stage; recommended to add alert nodes for failure notifications.                              | Sticky Note - Sourcing Stage                                                                             |
| The AI scoring engine block includes data parsing and AI ranking with custom scoring logic for candidate prioritization.                                           | Sticky Note - AI Scoring Engine                                                                          |
| Google Sheets node uses appendOrUpdate operation matched by candidate name to maintain clean, non-duplicated candidate records; Slack nodes provide workflow alerts. | Sticky Note - Saving Candidates                                                                          |

---

**Disclaimer:**  
The provided text exclusively originates from an automated workflow created with n8n, adhering strictly to current content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.