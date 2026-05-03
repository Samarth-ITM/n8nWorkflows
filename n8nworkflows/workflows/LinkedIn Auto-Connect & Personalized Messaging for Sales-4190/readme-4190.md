LinkedIn Auto-Connect & Personalized Messaging for Sales

https://n8nworkflows.xyz/workflows/linkedin-auto-connect---personalized-messaging-for-sales-4190


# LinkedIn Auto-Connect & Personalized Messaging for Sales

### 1. Workflow Overview

This workflow automates LinkedIn outreach for sales purposes by managing connections and sending personalized messages. It orchestrates data intake, profile scraping, connection verification, personalized message generation using AI, and follow-up messaging, all integrated with Google Sheets for data management and Gmail for email outreach.

**Logical Blocks:**

- **1.1 Input Reception & Initialization**  
  Handles incoming data through form and sheet triggers, initiating the workflow and preparing batches of profiles for processing.

- **1.2 Profile Scraping & Data Enrichment**  
  Uses browser automation to scrape LinkedIn profiles and extract relevant information to enrich contact data.

- **1.3 Connection Verification & Status Update**  
  Checks if contacts are already LinkedIn connections, updates connection statuses and statistics in Google Sheets.

- **1.4 Invitation & Messaging Automation**  
  Automates sending connection invites and personalized messages via LinkedIn using AI-generated content.

- **1.5 AI-Powered Personalization**  
  Generates tailored messages using the OpenAI Chat model integrated with LangChain agent for natural language processing.

- **1.6 Control Flow & Throttling**  
  Implements batching, wait nodes, and loop controls to manage API limits, timing constraints, and error handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

- **Overview:**  
  This block captures initial input from users or Google Sheets updates and splits data into manageable batches for sequential processing.

- **Nodes Involved:**  
  - Fill Out Keywords (Form Trigger)  
  - Run when profile is update (Google Sheets Trigger)  
  - Loop Over Items2 (SplitInBatches)  
  - Split Out (SplitOut)  
  - Add Profiles (Google Sheets)  

- **Node Details:**  
  - **Fill Out Keywords**  
    - Type: Form Trigger  
    - Role: Starts workflow when keywords are submitted via a form  
    - Config: Uses webhook to receive form data  
    - Inputs: External form submission  
    - Outputs: Triggers Profile Scraper node  
    - Potential Failures: Missing/invalid form data, webhook timeout  

  - **Run when profile is update**  
    - Type: Google Sheets Trigger  
    - Role: Triggers workflow on profile sheet updates  
    - Config: Watches a specific Google Sheet and worksheet  
    - Inputs: Sheet update events  
    - Outputs: Check if a person is a connection node  
    - Potential Failures: Credential errors, sheet access issues  

  - **Loop Over Items2**  
    - Type: SplitInBatches  
    - Role: Processes profiles in batches for controlled flow  
    - Config: Batch size unspecified (default or dynamic)  
    - Inputs: Data array from previous nodes  
    - Outputs: Two outputs: main batch and error fallback  
    - Potential Failures: Batch size too large, empty input arrays  

  - **Split Out**  
    - Type: SplitOut  
    - Role: Splits incoming data into individual items for parallel processing  
    - Inputs: Array of profiles  
    - Outputs: Individual profile data to Add Profiles node  
    - Potential Failures: Null or malformed data arrays  

  - **Add Profiles**  
    - Type: Google Sheets  
    - Role: Writes new profile data into Google Sheets for record keeping  
    - Config: Writes to a specific sheet and range  
    - Inputs: Individual profile data  
    - Outputs: Continues workflow for connection checking  
    - Potential Failures: Write permission errors, API rate limits  

#### 2.2 Profile Scraping & Data Enrichment

- **Overview:**  
  Uses browser automation flows to scrape LinkedIn profiles, extracting detailed contact information for further processing.

- **Nodes Involved:**  
  - Profile Scraper (Browserflow)  
  - Scrap New Connection Profile (Browserflow)  
  - Organise Infomation (Set)  

- **Node Details:**  
  - **Profile Scraper**  
    - Type: Browserflow  
    - Role: Automates LinkedIn profile scraping based on keywords or URLs  
    - Config: Configured with browser flow script to navigate and scrape data  
    - Inputs: Incoming form data or batch items  
    - Outputs: Splits data for Add Profiles node  
    - Potential Failures: LinkedIn UI changes, login timeouts, network errors  

  - **Scrap New Connection Profile**  
    - Type: Browserflow  
    - Role: Scrapes profiles of newly connected contacts for detailed info  
    - Inputs: Triggered after AI checks and connection status confirms new contact  
    - Outputs: Organise Infomation node  
    - Potential Failures: Same as above, plus blocked profiles or private info  

  - **Organise Infomation**  
    - Type: Set  
    - Role: Formats and organizes scraped data into structured output compatible with AI nodes  
    - Inputs: Raw scraped data  
    - Outputs: AI Agent node for message generation  
    - Potential Failures: Expression errors, missing fields  

#### 2.3 Connection Verification & Status Update

- **Overview:**  
  Checks each profile’s connection status on LinkedIn and updates Google Sheets records and statistics accordingly.

- **Nodes Involved:**  
  - Check if a person is a connection. (If)  
  - Loop Over Items (SplitInBatches)  
  - Connection Checker (Browserflow)  
  - Check Connection (Browserflow)  
  - If connection (If)  
  - If connection1 (If)  
  - Stat Update (Google Sheets)  
  - Wait (Wait)  

- **Node Details:**  
  - **Check if a person is a connection.**  
    - Type: If  
    - Role: Determines if a profile is already a connection by evaluating sheet data or prior state  
    - Inputs: Google Sheets Trigger data  
    - Outputs: Branches to Loop Over Items for connected or not connected profiles  
    - Potential Failures: Expression logic errors  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes profiles batch-wise for connection checking  
    - Inputs: Profiles from previous nodes  
    - Outputs: Branches to If connection or Connection Checker nodes  
    - Potential Failures: Batch size issues  

  - **Connection Checker**  
    - Type: Browserflow  
    - Role: Automates LinkedIn UI to verify connection status for each profile  
    - Outputs: Loop Over Items for next processing step  
    - Potential Failures: UI changes, timeouts, captchas  

  - **Check Connection**  
    - Type: Browserflow  
    - Role: Secondary or confirmatory check of connection status  
    - Outputs: If connection1 node  
    - Potential Failures: Same as above  

  - **If connection**  
    - Type: If  
    - Role: Branches workflow based on connection status (connected vs not connected)  
    - Outputs: Stat Update (connected) or Wait (not connected)  
    - Potential Failures: Expression evaluation errors  

  - **If connection1**  
    - Type: If  
    - Role: Additional branching for nuanced connection states  
    - Outputs: Wait2 node for further scraping or processing  
    - Potential Failures: Same as above  

  - **Stat Update**  
    - Type: Google Sheets  
    - Role: Updates connection statistics and statuses in Google Sheets  
    - Inputs: Data from If connection branch  
    - Outputs: Continues to Wait node  
    - Potential Failures: Sheets API limits, write errors  

  - **Wait**  
    - Type: Wait  
    - Role: Introduces delay to throttle requests and avoid LinkedIn rate limits  
    - Outputs: Loop Over Items1 for next batch processing  
    - Potential Failures: Timeout misconfiguration  

#### 2.4 Invitation & Messaging Automation

- **Overview:**  
  Automates sending LinkedIn connection invites and personalized follow-up messages, integrating AI-generated content and Gmail for communication.

- **Nodes Involved:**  
  - Loop Over Items1 (SplitInBatches)  
  - Invite Sender (Browserflow)  
  - Update Invite (Google Sheets)  
  - Set Connection Limit (Set)  
  - Wait1 (Wait)  
  - Gmail (Gmail)  
  - Send Personalised Message (Browserflow)  

- **Node Details:**  
  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Processes invites and messages in manageable chunks  
    - Outputs: Gmail and Invite Sender nodes (parallel outputs)  
    - Potential Failures: Batch size and concurrency limitations  

  - **Invite Sender**  
    - Type: Browserflow  
    - Role: Automates sending LinkedIn connection requests  
    - Inputs: Profile data batches  
    - Outputs: Update Invite node  
    - Potential Failures: LinkedIn restrictions, UI changes  

  - **Update Invite**  
    - Type: Google Sheets  
    - Role: Updates invite status and timestamps in Google Sheets  
    - Outputs: Set Connection Limit node  
    - Potential Failures: API limits, write errors  

  - **Set Connection Limit**  
    - Type: Set  
    - Role: Sets or updates connection limits or counters for throttling  
    - Outputs: Wait1 node  
    - Potential Failures: Incorrect data setting  

  - **Wait1**  
    - Type: Wait  
    - Role: Adds delay between invite batches to comply with LinkedIn limits  
    - Outputs: Loop Over Items1 (creates loop)  
    - Potential Failures: Misconfigured delays causing queue blocks  

  - **Gmail**  
    - Type: Gmail  
    - Role: Sends email notifications or follow-ups related to invites or messages  
    - Inputs: Profile data and invite status batches  
    - Potential Failures: Authentication errors, quota exceeded  

  - **Send Personalised Message**  
    - Type: Browserflow  
    - Role: Sends personalized LinkedIn messages using AI-generated content  
    - Inputs: AI Agent output  
    - Potential Failures: LinkedIn UI failures, message formatting errors  

#### 2.5 AI-Powered Personalization

- **Overview:**  
  Utilizes OpenAI Chat models via LangChain agent to generate natural language personalized messages, improving engagement quality.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - OpenAI Chat Model (LangChain LM Chat OpenAI)  
  - Organise Infomation (Set)  

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI prompt handling, chaining, and response generation  
    - Inputs: Structured profile data from Organise Infomation  
    - Outputs: Send Personalised Message node  
    - Potential Failures: API key issues, prompt errors, rate limits  

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Executes the chat completions with OpenAI API  
    - Inputs: Prompts and context from AI Agent  
    - Outputs: AI Agent node as language model response  
    - Potential Failures: API connection failures, invalid parameters  

  - **Organise Infomation** (also part of block 2.2)  
    - Prepares input data for AI processing  

#### 2.6 Control Flow & Throttling

- **Overview:**  
  Controls execution pacing, error fallback, batch loops, and conditional branching to ensure robustness and compliance with platform limits.

- **Nodes Involved:**  
  - Wait, Wait1, Wait2 (Wait)  
  - Loop Over Items, Loop Over Items1, Loop Over Items2 (SplitInBatches)  
  - add browser flow if it gives error (NoOp)  
  - Split Out (SplitOut)  

- **Node Details:**  
  - **Wait, Wait1, Wait2**  
    - Type: Wait  
    - Role: Add delays to avoid rate limits and pacing issues  
    - Config: Configured with appropriate durations (not specified)  
    - Potential Failures: Excessive wait causing delays, insufficient wait causing throttling  

  - **Loop Over Items**, **Loop Over Items1**, **Loop Over Items2**  
    - Type: SplitInBatches  
    - Role: Manage batch processing loops  
    - Config: Batch sizes control throughput  
    - Potential Failures: Infinite loops if not properly closed, batch size misconfiguration  

  - **add browser flow if it gives error**  
    - Type: NoOp  
    - Role: Placeholder node to add alternative browser flow for error handling or recovery  
    - Inputs: Failure output from Loop Over Items2  
    - Outputs: Loop Over Items2 (retry or alternate path)  

  - **Split Out**  
    - Role: Splits arrays for parallel processing  

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                             | Input Node(s)                  | Output Node(s)                   | Sticky Note                                   |
|-------------------------------|---------------------------|---------------------------------------------|--------------------------------|---------------------------------|----------------------------------------------|
| Fill Out Keywords              | Form Trigger              | Entry point: receives keywords input       | -                              | Profile Scraper                  |                                              |
| Run when profile is update     | Google Sheets Trigger     | Trigger on profile sheet update             | -                              | Check if a person is a connection.|                                              |
| Loop Over Items2               | SplitInBatches            | Batch processing for initial profile data  | -                              | add browser flow if it gives error (fallback) |                                              |
| add browser flow if it gives error | NoOp                     | Error handling placeholder                   | Loop Over Items2               | Loop Over Items2                 |                                              |
| Profile Scraper               | Browserflow               | Scrapes LinkedIn profiles                    | Fill Out Keywords              | Split Out                      |                                              |
| Split Out                     | SplitOut                  | Splits data array into individual items     | Profile Scraper                | Add Profiles                   |                                              |
| Add Profiles                  | Google Sheets             | Adds new profile data to Google Sheets      | Split Out                     | Check if a person is a connection.|                                              |
| Run when profile is update    | Google Sheets Trigger     | See above (duplicate)                         | -                              | Check if a person is a connection.|                                              |
| Check if a person is a connection. | If                       | Checks if profile is a LinkedIn connection  | Run when profile is update     | Loop Over Items                 |                                              |
| Loop Over Items               | SplitInBatches            | Batch processing for connection checking    | Check if a person is a connection. | If connection, Connection Checker |                                              |
| Connection Checker            | Browserflow               | Verifies LinkedIn connection status         | Loop Over Items                | Loop Over Items                |                                              |
| If connection                 | If                       | Branches based on connection status         | Loop Over Items                | Stat Update, Wait              |                                              |
| Stat Update                  | Google Sheets             | Updates connection stats                      | If connection                 | Wait                         |                                              |
| Wait                         | Wait                      | Delay for throttling                          | Stat Update                   | Loop Over Items1               |                                              |
| Loop Over Items1             | SplitInBatches            | Batch processing for invites & messages      | Wait                         | Gmail, Invite Sender           |                                              |
| Gmail                        | Gmail                     | Sends email notifications                     | Loop Over Items1              | -                            |                                              |
| Invite Sender                | Browserflow               | Sends LinkedIn connection invites             | Loop Over Items1              | Update Invite                 |                                              |
| Update Invite                | Google Sheets             | Updates invite status in sheets                | Invite Sender                | Set Connection Limit          |                                              |
| Set Connection Limit         | Set                       | Sets limits/counters for connections          | Update Invite                | Wait1                        |                                              |
| Wait1                        | Wait                      | Delay between invite batches                    | Set Connection Limit          | Loop Over Items1              |                                              |
| Check Connection             | Browserflow               | Secondary connection check                      | Get Profiles                 | If connection1                |                                              |
| If connection1               | If                        | Branches based on refined connection status   | Check Connection             | Wait2                       |                                              |
| Wait2                        | Wait                      | Delay before scraping new connection profile   | If connection1               | Scrap New Connection Profile  |                                              |
| Scrap New Connection Profile | Browserflow               | Scrapes newly connected profile details        | Wait2                       | Organise Infomation           |                                              |
| Organise Infomation          | Set                       | Formats scraped data for AI processing          | Scrap New Connection Profile | AI Agent                     |                                              |
| AI Agent                    | LangChain Agent           | Generates AI personalized messages              | Organise Infomation           | Send Personalised Message     |                                              |
| OpenAI Chat Model           | LangChain LM Chat OpenAI  | OpenAI Chat completion model                     | AI Agent (ai_languageModel)   | AI Agent                     |                                              |
| Send Personalised Message   | Browserflow               | Sends AI-generated personalized LinkedIn message | AI Agent                   | -                            |                                              |
| Get Profiles                | Google Sheets             | Retrieves profile data from sheets               | Schedule Trigger             | Check Connection             |                                              |
| Schedule Trigger            | Schedule Trigger          | Scheduled start to fetch profiles                | -                            | Get Profiles                 |                                              |
| Connection Checker          | Browserflow               | See above (duplicate)                             | Loop Over Items              | Loop Over Items               |                                              |
| add browser flow if it gives error | NoOp                     | See above (duplicate)                             | Loop Over Items2             | Loop Over Items2              |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node "Fill Out Keywords"**  
   - Type: Form Trigger  
   - Configure webhook to receive keywords for LinkedIn search  
   - No authentication needed  

2. **Create Google Sheets Trigger node "Run when profile is update"**  
   - Connect to your Google Sheets account  
   - Set to trigger on updates to the profile data sheet  

3. **Create SplitInBatches node "Loop Over Items2"**  
   - Set batch size according to workflow needs (e.g., 10)  
   - Connect input to form trigger output or other data source  

4. **Create NoOp node "add browser flow if it gives error"**  
   - Used for error fallback; connect error output of Loop Over Items2 back to this node  

5. **Create Browserflow node "Profile Scraper"**  
   - Configure browser automation flow to scrape LinkedIn profiles based on keywords  
   - Ensure LinkedIn login is handled in the flow  

6. **Create SplitOut node "Split Out"**  
   - Configure to split array of profile data into individual items  

7. **Create Google Sheets node "Add Profiles"**  
   - Connect to Google Sheets credential  
   - Set to append new profile rows  

8. **Create If node "Check if a person is a connection."**  
   - Use expression to check connection status from sheet data or other indicators  

9. **Create SplitInBatches node "Loop Over Items"**  
   - Batch processing for connection verification  

10. **Create Browserflow node "Connection Checker"**  
    - Automate LinkedIn UI to verify connection status for each profile  

11. **Create If node "If connection"**  
    - Branch workflow based on connection result  

12. **Create Google Sheets node "Stat Update"**  
    - Update connection statistics and status  

13. **Create Wait node "Wait"**  
    - Add delay to avoid rate limits; configure appropriate timing (e.g., 1 minute)  

14. **Create SplitInBatches node "Loop Over Items1"**  
    - Batch processing for sending invites and messages  

15. **Create Browserflow node "Invite Sender"**  
    - Automate sending LinkedIn connection invites  

16. **Create Google Sheets node "Update Invite"**  
    - Update invite status and timestamps  

17. **Create Set node "Set Connection Limit"**  
    - Manage connection counters or limits  

18. **Create Wait node "Wait1"**  
    - Delay between invite batches to comply with limits  

19. **Create Gmail node "Gmail"**  
    - Configure Gmail OAuth2 credentials  
    - Set to send out notification or follow-up emails  

20. **Create Browserflow node "Send Personalised Message"**  
    - Automate sending personalized LinkedIn messages  

21. **Create Wait node "Wait2"**  
    - Delay before scraping newly connected profiles  

22. **Create Browserflow node "Scrap New Connection Profile"**  
    - Scrape detailed profiles of new connections  

23. **Create Set node "Organise Infomation"**  
    - Format scraped data for AI processing  

24. **Create LangChain LM Chat OpenAI node "OpenAI Chat Model"**  
    - Configure OpenAI API credentials  
    - Set model parameters (e.g., GPT-4, temperature)  

25. **Create LangChain Agent node "AI Agent"**  
    - Connect OpenAI Chat Model as language model  
    - Configure prompt templates for personalized messaging  

26. **Create Schedule Trigger node "Schedule Trigger"**  
    - Set schedule for workflow to start periodically  

27. **Create Google Sheets node "Get Profiles"**  
    - Retrieve profiles for processing on schedule trigger  

28. **Create Browserflow node "Check Connection"**  
    - Secondary confirmation of connection status  

29. **Create If node "If connection1"**  
    - Conditional branch based on refined connection checks  

30. **Connect all nodes according to the input/output described in the overview, ensuring loops and wait nodes are properly configured to prevent rate limits and infinite loops.**

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Use OAuth2 credentials for Gmail and Google Sheets to ensure seamless API access and avoid auth errors. | Credential setup within n8n integrations.                                                              |
| LinkedIn UI changes may break browser automation flows; maintain and update Browserflow scripts regularly. | Browserflow node maintenance best practices.                                                           |
| OpenAI API usage requires an API key with sufficient quota; monitor usage to avoid throttling.           | https://platform.openai.com/account/rate-limits                                                         |
| Consider adding error handling and retry logic in NoOp node for robustness in case of transient failures. | Workflow resilience and error management.                                                              |
| For LinkedIn automation, respect LinkedIn usage policies to avoid account bans.                          | LinkedIn User Agreement and Automation Guidelines.                                                      |
| Video demos and project blog available at: https://example.com/linkedin-auto-connect                    | Project documentation and demonstration resources.                                                     |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.