Match Indeed Job Listings to Your Resume with Gemini AI and Telegram Alerts

https://n8nworkflows.xyz/workflows/match-indeed-job-listings-to-your-resume-with-gemini-ai-and-telegram-alerts-8864


# Match Indeed Job Listings to Your Resume with Gemini AI and Telegram Alerts

### 1. Workflow Overview

This workflow automates the process of scraping job listings from the web, matching them against a user’s resume using Google Gemini AI, and sending filtered job alerts via Telegram. The main use case is to keep job seekers informed of relevant opportunities without manual searching, leveraging AI to identify best matches based on their profile.

The workflow consists of four logical blocks:

- **1.1 Input Reception and Job Scraping Initiation:** Manual trigger starts a web scraping task through the BrowserAct API to collect job market data.
- **1.2 Scraping Status Monitoring:** Periodic checks ensure the scraping task is completed before proceeding.
- **1.3 AI Processing and Matching:** An AI Agent powered by Google Gemini processes scraped listings, compares them with the resume, removes duplicates, and formats the results.
- **1.4 Notification Delivery:** The filtered job matches are reformatted for readability and sent as Telegram messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Job Scraping Initiation

- **Overview:**  
  This block triggers the workflow manually and initiates a job scraping task via the BrowserAct API using a workflow ID configured in the node. It collects initial raw job listing data asynchronously.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - HTTP Request (Start BrowserAct Workflow)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow  
    - Configuration: Default manual trigger, no parameters  
    - Input: None  
    - Output: Triggers next node  

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Starts a job scraping task via BrowserAct API  
    - Configuration:  
      - POST to `https://api.browseract.com/v2/workflow/run-task`  
      - Sends JSON body with `workflow_id` (user must replace with their BrowserAct workflow)  
      - Authentication: HTTP Bearer with BrowserAct API token credential  
      - Retries on failure enabled  
      - Always outputs data even on errors  
    - Input: Trigger from manual node  
    - Output: Response JSON including task ID or error info  
    - Potential Failures: API authentication failure, invalid workflow ID, network timeout  

---

#### 2.2 Scraping Status Monitoring

- **Overview:**  
  This block monitors the scraping task status by polling BrowserAct API until the task is completed. It uses conditional logic with `If` nodes and `Wait` nodes to pause and retry checking the task status.

- **Nodes Involved:**  
  - If (Check for error and valid task ID)  
  - HTTP Request1 (Get task status)  
  - If1 (Check if task status is "finished")  
  - Wait1 (Pause before retry after starting task)  
  - Wait (Pause before retry after checking status)

- **Node Details:**

  - **If**  
    - Type: Conditional (If)  
    - Role: Checks if the response from starting the task has no error and valid task ID  
    - Condition:  
      - `error` field must not exist  
      - `id` field must not equal "null"  
    - Input: Output of HTTP Request (start task)  
    - Output:  
      - True: Proceed to get task status  
      - False: Retry starting task after Wait1  
    - Failures: Expression evaluation errors if JSON path missing  

  - **HTTP Request1**  
    - Type: HTTP Request  
    - Role: Queries BrowserAct API to get current task status by task ID  
    - Configuration:  
      - GET `https://api.browseract.com/v2/workflow/get-task`  
      - Query parameter `task_id` set from previous node's JSON `id` field  
      - Uses same HTTP Bearer authentication as start task  
    - Input: True output from If node  
    - Output: Task status JSON  
    - Failures: API errors, invalid task ID, timeout  

  - **If1**  
    - Type: Conditional (If)  
    - Role: Check if task status is `"finished"` and no error present  
    - Condition:  
      - `error` field must not exist  
      - `status` field equals `"finished"`  
    - Input: Output of HTTP Request1  
    - Output:  
      - True: Proceed to AI processing  
      - False: Wait and retry polling status  
    - Failures: Expression failures if fields missing  

  - **Wait1**  
    - Type: Wait  
    - Role: Pause 1 minute before retrying task start if initial start failed  
    - Input: False output from If node  
    - Output: Loops back to HTTP Request node  

  - **Wait**  
    - Type: Wait  
    - Role: Pause 2 minutes before retrying task status check if not finished  
    - Input: False output from If1 node  
    - Output: Loops back to HTTP Request1 node  

---

#### 2.3 AI Processing and Matching

- **Overview:**  
  After scraping is finished, this block uses a Google Gemini-powered AI Agent to analyze job offers against the user’s resume. The AI filters and formats matching jobs, removing duplicates. The output is then parsed into a structured JSON format.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent with Gemini)  
  - Structured Output Parser  
  - Code in JavaScript  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent node  
    - Role: Process scraped job listings and user resume to find matching jobs  
    - Configuration:  
      - Prompt includes user resume details (hardcoded in prompt text)  
      - Input data from scraping results passed via expression (`{{ $json.output.string }}`)  
      - Output expected in a strict JSON format listing job offers with keys: Title, Summary, Needs, Education, Link  
      - Removes duplicates by company name and job title  
      - Outputs default "No job Found" structured message if no matches  
      - Uses connected Google Gemini API credentials  
      - Has fallback enabled for output parser  
    - Input: Output of If1 node (task finished)  
    - Output: AI-generated JSON string with matched jobs  
    - Failures: API authentication errors, prompt parsing issues, rate limits, malformed AI output  

  - **Structured Output Parser**  
    - Type: LangChain Output Parser node  
    - Role: Enforces JSON schema on AI Agent output to ensure expected format  
    - Configuration: Example JSON schema provided matching expected job listing structure  
    - Input: AI Agent output  
    - Output: Parsed structured JSON for downstream processing  
    - Failures: Parser errors if AI output malformed  

  - **Code in JavaScript**  
    - Type: Code node  
    - Role: Transforms AI output JSON into an array of objects each with a `text` field containing job info  
    - Logic: Extracts the `Request` array from AI output and wraps each item in `{ json: { text: item } }` format for Telegram node compatibility  
    - Input: AI Agent structured output  
    - Output: Reformatted job listings ready for messaging  
    - Failures: Code runtime errors if input JSON missing or malformed  

---

#### 2.4 Notification Delivery

- **Overview:**  
  This final block sends the filtered job listings as formatted messages to a Telegram channel or chat. It uses the Telegram node to post each job offer individually.

- **Nodes Involved:**  
  - Send a text message (Telegram)

- **Node Details:**

  - **Send a text message**  
    - Type: Telegram node  
    - Role: Sends job offer messages to Telegram channel  
    - Configuration:  
      - Message text built from JSON fields Title, Summary, Needs, Education, and Link using expressions  
      - Chat ID set to a Telegram channel or group (e.g., `@Test`; user must replace with actual chat ID)  
      - Requires Telegram API credentials (OAuth2 token or bot token)  
    - Input: Reformatted job listing array from Code node  
    - Output: Telegram API message send response  
    - Failures: Invalid chat ID, Telegram API limits, credential errors, message formatting errors  

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                            | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                                                                            |
|---------------------------|----------------------------------|--------------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Entry point to start the workflow           | None                          | HTTP Request                | ## Try It Out! This n8n template helps you stay on top of the job market by matching scraped job offers with your resume using an AI Agent.            |
| HTTP Request              | HTTP Request                     | Start BrowserAct job scraping task          | When clicking ‘Execute workflow’ | If                         | ## 1. Trigger the Job Scraper. Uses BrowserAct API to collect job offers. Don't forget to add your BrowserAct Workflow ID.                              |
| If                       | If Condition                    | Check if scraping task started successfully | HTTP Request                  | HTTP Request1, Wait1         | ## 2. Wait for Scraping to Finish. These nodes check the status of the scraping task.                                                                   |
| HTTP Request1             | HTTP Request                    | Get current scraping task status             | If                           | If1                         | ## 2. Wait for Scraping to Finish                                                                                                                      |
| If1                      | If Condition                   | Check if scraping task is finished           | HTTP Request1                 | AI Agent, Wait               | ## 2. Wait for Scraping to Finish                                                                                                                      |
| Wait1                    | Wait                           | Pause 1 minute before retrying start task    | If (false)                   | HTTP Request                 | ## 2. Wait for Scraping to Finish                                                                                                                      |
| Wait                     | Wait                           | Pause 2 minutes before retrying status check | If1 (false)                  | HTTP Request1                | ## 2. Wait for Scraping to Finish                                                                                                                      |
| AI Agent                 | LangChain AI Agent              | Use AI to filter and match jobs to resume    | If1 (true)                   | Code in JavaScript           | ## 3. Use an AI Agent to Find Matching Jobs. Connect Gemini and update your resume in prompt.                                                          |
| Structured Output Parser | LangChain Output Parser         | Validate and structure AI output JSON        | AI Agent                     | AI Agent (output parser link) | ## 3. Use an AI Agent to Find Matching Jobs                                                                                                           |
| Code in JavaScript       | Code                           | Reformat AI output for Telegram messaging    | AI Agent                     | Send a text message          | ## 4. Send Notifications. Transform AI output into readable format, then send via Telegram.                                                             |
| Send a text message      | Telegram                       | Send job alert messages to Telegram channel | Code in JavaScript           | None                        | ## 4. Send Notifications. Connect Telegram and add your Channel ID.                                                                                     |
| Sticky Note-Intro        | Sticky Note                    | Introductory instructions                     | None                         | None                        | ## Try It Out! This n8n template helps you stay on top of the job market by matching scraped job offers with your resume using an AI Agent.            |
| Sticky Note-Wait         | Sticky Note                    | Explains waiting and status check nodes      | None                         | None                        | ## 2. Wait for Scraping to Finish                                                                                                                      |
| Sticky Note-AI           | Sticky Note                    | Explains AI Agent node                         | None                         | None                        | ## 3. Use an AI Agent to Find Matching Jobs. Connect Gemini and update your resume.                                                                     |
| Sticky Note-Notifications | Sticky Note                    | Explains notification nodes                    | None                         | None                        | ## 4. Send Notifications. Connect Telegram and add your Channel ID.                                                                                    |
| Sticky Note-How to Use   | Sticky Note                    | Setup and usage instructions                   | None                         | None                        | ## How to use: Set up credentials, BrowserAct workflow, update workflow_id, activate or manually trigger workflow.                                       |
| Sticky Note-How to Use1  | Sticky Note                    | Help links and video tutorials                  | None                         | None                        | ## Need Help? [BrowserAct API & Workflow ID](https://www.youtube.com/watch?v=pDjoZWEsZlE), [Connect n8n to BrowserAct](https://www.youtube.com/watch?v=RoYMdJaRdcQ), [Customize BrowserAct Templates](https://www.youtube.com/watch?v=CPZHFUASncY) |
| Sticky Note-How to Use2  | Sticky Note                    | Summary instructions for setup                  | None                         | None                        | Create BrowserAct Workflow -> Add Token -> Change workflow_id -> Connect Gemini & update resume prompt -> Configure Telegram token & channel ID        |
| Sticky Note-How to Use3  | Sticky Note                    | Link to workflow guidance and showcase video   | None                         | None                        | ## Watch Workflow Guidance and Showcase: [Never Manually Search for a Job Again (AI Automation Tutorial)](https://youtu.be/mRJw8Jyrizg)                   |
| Gemini                   | LangChain Google Gemini LM      | AI language model used by AI Agent             | None                         | AI Agent (languageModel)     | ## 3. Use an AI Agent to Find Matching Jobs                                                                                                           |
| Gemini Backup            | LangChain Google Gemini LM      | Backup AI language model (not in main flow)    | None                         | AI Agent (languageModel)     |                                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Default manual trigger, no parameters  

2. **Create HTTP Request node to start scraping task**  
   - Name: "HTTP Request"  
   - HTTP Method: POST  
   - URL: `https://api.browseract.com/v2/workflow/run-task`  
   - Authentication: HTTP Bearer with BrowserAct API credential (create or import credentials)  
   - Body Parameters (JSON): `{"workflow_id": "<Your BrowserAct workflow ID>"}`  
   - Retry On Fail: true  
   - Always Output Data: true  
   - Connect output of Manual Trigger to this node  

3. **Create If node to check start task response**  
   - Name: "If"  
   - Conditions (AND):  
     - `error` does not exist in JSON  
     - `id` field not equal to `"null"`  
   - Connect output of HTTP Request to this node  

4. **Create HTTP Request node to get task status**  
   - Name: "HTTP Request1"  
   - HTTP Method: GET  
   - URL: `https://api.browseract.com/v2/workflow/get-task`  
   - Query Parameter: `task_id` = Expression: `{{$json["id"]}}` from previous node  
   - Authentication: HTTP Bearer with BrowserAct API credential  
   - Connect True output of "If" node to this node  

5. **Create If node to check if task finished**  
   - Name: "If1"  
   - Conditions (AND):  
     - `error` does not exist  
     - `status` equals `"finished"`  
   - Connect output of HTTP Request1 to this node  

6. **Create Wait nodes**  
   - Name: "Wait1"  
     - Duration: 1 minute  
     - Connect False output of "If" node to this node  
     - Connect output of this node back to "HTTP Request" node to retry start task  

   - Name: "Wait"  
     - Duration: 2 minutes  
     - Connect False output of "If1" node to this node  
     - Connect output of this node back to "HTTP Request1" node to retry status check  

7. **Create AI Agent node**  
   - Name: "AI Agent"  
   - Use LangChain AI Agent node configured with Google Gemini credentials  
   - Prompt: Insert your resume text and instructions to filter job listings from the scraping output (pass scraped data as input)  
   - Enable Output Parser with a JSON schema matching expected job offer fields (Title, Summary, Needs, Education, Link)  
   - Connect True output of "If1" node to this node  

8. **Create Structured Output Parser node**  
   - Name: "Structured Output Parser"  
   - Provide JSON schema example matching AI Agent output structure  
   - Connect output of AI Agent node to this node's input parser  

9. **Create Code node to reformat AI output for Telegram**  
   - Name: "Code in JavaScript"  
   - JavaScript code to transform AI Agent JSON output into an array of JSON objects each with a `text` field containing job info (see code logic in original workflow)  
   - Connect output of AI Agent node to this node  

10. **Create Telegram node to send messages**  
    - Name: "Send a text message"  
    - Telegram API credentials: configure your Telegram Bot token or OAuth2 credentials  
    - Chat ID: set to your Telegram channel ID or username (e.g., `@YourChannel`)  
    - Message Text: use expressions to format job fields (Title, Summary, Needs, Education, Link) into a message string  
    - Connect output of Code node to this node  

11. **Activate and test manually**  
    - Set credentials for BrowserAct, Google Gemini, and Telegram  
    - Replace placeholders (BrowserAct workflow ID, Telegram chat ID, resume text in AI Agent prompt)  
    - Execute workflow manually and verify each step’s output  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This n8n template helps you stay on top of the job market by matching scraped job offers with your resume using an AI Agent.                                                                                                                  | Introductory sticky note                                                                                  |
| Join the [Discord](https://discord.com/invite/UpnCKd7GaU) or Visit Our [Blog](https://www.browseract.com/blog) for help and additional resources.                                                                                             | Support and community links                                                                                |
| Set up credentials for BrowserAct, Google Gemini, and Telegram to enable the respective nodes. Replace the BrowserAct workflow ID and Telegram chat ID with your own before activating the workflow.                                          | Setup instructions                                                                                         |
| [How to Find Your BrowserAct API Key & Workflow ID](https://www.youtube.com/watch?v=pDjoZWEsZlE), [How to Connect n8n to BrowserAct](https://www.youtube.com/watch?v=RoYMdJaRdcQ), [How to Use & Customize BrowserAct Templates](https://www.youtube.com/watch?v=CPZHFUASncY) | Video tutorials                                                                                           |
| Watch Workflow Guidance and Showcase: [Never Manually Search for a Job Again (AI Automation Tutorial)](https://youtu.be/mRJw8Jyrizg)                                                                                                           | Workflow demonstration video                                                                               |
| Create BrowserAct Workflow -> Add BrowserAct Token to Run Node -> Change BrowserAct workflow_id in Run Node -> Connect Gemini & Update Resume in Agent Node -> Configure Telegram Token & Channel ID in Send Node                           | Summary of setup steps                                                                                      |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.