Generate Contextual YouTube Comments Automatically with GPT-4o

https://n8nworkflows.xyz/workflows/generate-contextual-youtube-comments-automatically-with-gpt-4o-4580


# Generate Contextual YouTube Comments Automatically with GPT-4o

### 1. Workflow Overview

This workflow automates the generation of contextual YouTube comments using GPT-4o by processing meeting transcripts and related metadata from Microsoft 365 and OneDrive. Its main purpose is to extract relevant meeting transcripts, analyze and summarize their content with AI, and generate intelligent and contextual comments or follow-ups for YouTube videos or related platforms. The workflow is designed for scenarios involving meeting recordings, transcript management, and automated content generation to enhance engagement or follow-up actions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Triggering:** Handling external events from webhooks and scheduled triggers to initiate processing.
- **1.2 Data Retrieval & Preprocessing:** Fetching meeting records, filtering relevant data, and retrieving transcript files.
- **1.3 Transcript Processing & Deduplication:** Splitting, sorting, and filtering transcript items; checking for already processed entries.
- **1.4 AI Analysis & Comment Generation:** Using OpenAI GPT-4o models and LangChain agents for summarization and drafting comments.
- **1.5 Output Handling & Notifications:** Formatting AI-generated content into HTML, saving data to Postgres, and sending email notifications.
- **1.6 Follow-up Actions & Webhook Responses:** Managing follow-up email dispatch and responding to webhook calls.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Triggering

- **Overview:** This block manages the workflow's entry points, triggering the process either on scheduled intervals or via form/webhook submissions.
- **Nodes Involved:** `Schedule Trigger`, `On form submission`, `Web Page`, `Next Action`
  
**Node Details:**

- **Schedule Trigger**
  - Type: Scheduled trigger node
  - Role: Polls every 5 minutes to check for new transcripts to process
  - Config: Default schedule with 5-minute interval
  - Input: None
  - Output: Starts flow into `Merge1`
  - Edge cases: Missed triggers if workflow down; rate limits if API calls are high
  
- **On form submission**
  - Type: Form Trigger (Webhook)
  - Role: Starts workflow on user form submission from web app
  - Config: Webhook ID assigned; expects form data payload
  - Input: External HTTP form submission
  - Output: Passes data to `Code2`
  - Edge cases: Invalid or missing form data, webhook failures
  
- **Web Page**
  - Type: Webhook node
  - Role: Receives initial web requests to start processing or fetch meeting rows
  - Config: Webhook ID assigned
  - Input: Incoming HTTP request
  - Output: Connected to `Get Meeting Row1`
  - Edge cases: Unauthorized access, invalid parameters
  
- **Next Action**
  - Type: Webhook node
  - Role: Accepts subsequent webhook calls for follow-up or next steps
  - Config: Webhook ID assigned
  - Input: Incoming HTTP requests
  - Output: Connects to `Get Meeting Row`
  - Edge cases: Same as above

---

#### 1.2 Data Retrieval & Preprocessing

- **Overview:** Fetches meeting data from Postgres and Microsoft 365 APIs, filters and sorts meeting records, and retrieves transcript files from OneDrive.
- **Nodes Involved:** `Get Meeting Row1`, `Get Meeting Row`, `Get Call Records`, `Code1`, `Filter for Join Url1`, `Get Online Meeting Details1`, `Split Out Meetings1`, `Remove Duplicates`, `Sort1`, `Check if Processed`, `Filter No Items`, `Get Items`, `Keep Matches`, `Search OneDrive for Meeting`, `Split Out3`, `List Transcripts2`, `Get Transcript1`

**Node Details:**

- **Get Meeting Row1**
  - Type: Postgres node
  - Role: Queries database for initial meeting data on web request
  - Config: SQL query to fetch meeting rows based on webhook input
  - Input: Triggered by `Web Page`
  - Output: Data passed to `Respond With WebApp HTML`
  - Edge cases: DB connection errors, empty results
  
- **Get Meeting Row**
  - Type: Postgres node
  - Role: Fetches meeting rows for follow-up actions
  - Config: SQL query with parameters from `Next Action`
  - Input: From `Next Action`
  - Output: To `Markdown`
  
- **Get Call Records**
  - Type: HTTP Request
  - Role: Retrieves call records metadata from Microsoft Graph or similar API
  - Config: Authenticated request with OAuth2 credentials
  - Input: From `Set Time Range`
  - Output: To `Code1`
  - Edge cases: API rate limits, auth token expiration
  
- **Code1**
  - Type: Code node (JavaScript)
  - Role: Processes call records data, potentially formatting or filtering
  - Input: From `Get Call Records`
  - Output: To `Filter for Join Url1`
  - Edge cases: Script errors, null data
  
- **Filter for Join Url1**
  - Type: Filter node
  - Role: Filters call records to keep only those with valid join URLs
  - Input: From `Code1`
  - Output: To `Get Online Meeting Details1`
  
- **Get Online Meeting Details1**
  - Type: HTTP Request
  - Role: Fetches detailed meeting information from Microsoft Graph or API for filtered calls
  - Config: Authenticated API call
  - Input: From `Filter for Join Url1`
  - Output: To `Split Out Meetings1`
  
- **Split Out Meetings1**
  - Type: SplitOut
  - Role: Splits array of meetings into individual items for further processing
  - Input: From `Get Online Meeting Details1`
  - Output: To `Remove Duplicates`
  
- **Remove Duplicates**
  - Type: Remove Duplicates node
  - Role: Ensures unique meeting entries, preventing duplicate processing
  - Input: From `Split Out Meetings1`
  - Output: To `Sort1`
  
- **Sort1**
  - Type: Sort node
  - Role: Sorts meetings, e.g., by date or relevance
  - Input: From `Remove Duplicates`
  - Output: To `Check if Processed`
  
- **Check if Processed**
  - Type: Postgres node
  - Role: Checks database to avoid re-processing meetings/transcripts already handled
  - Input: From `Sort1`
  - Output: Either `Filter No Items` (no new items) or `Keep Matches` (new items found)
  
- **Filter No Items**
  - Type: Filter node
  - Role: Filters out cases where no new transcripts are detected
  - Input: From `Check if Processed`
  - Output: To `Get Items`
  
- **Get Items**
  - Type: Code node
  - Role: Extracts relevant items (meetings/transcripts) from filtered data
  - Input: From `Filter No Items`
  - Output: To `Keep Matches`
  
- **Keep Matches**
  - Type: Merge node
  - Role: Combines new items from `Get Items` with matches from `Check if Processed`
  - Input: From `Get Items` and `Check if Processed`
  - Output: To `Get Transcript Data`
  
- **Search OneDrive for Meeting**
  - Type: HTTP Request
  - Role: Searches OneDrive for transcript files related to meetings
  - Input: From `Get Transcript Data`
  - Output: To `Split Out3`
  
- **Split Out3**
  - Type: SplitOut
  - Role: Splits array of transcript files into individual items
  - Input: From `Search OneDrive for Meeting`
  - Output: To `List Transcripts2`
  
- **List Transcripts2**
  - Type: HTTP Request
  - Role: Lists transcript metadata or contents from storage
  - Input: From `Split Out3`
  - Output: To `Get Transcript1`
  
- **Get Transcript1**
  - Type: HTTP Request
  - Role: Downloads or retrieves transcript content files
  - Input: From `List Transcripts2`
  - Output: To `Set Transcription & Subject1`

---

#### 1.3 Transcript Processing & Deduplication

- **Overview:** Prepares and structures transcript content for AI processing, including splitting, setting variables, and batching.
- **Nodes Involved:** `Get Transcript Data`, `Split Out Transcriptions`, `Get Transcript`, `Set Transcription & Subject`, `Set Transcription & Subject1`, `Set Transcription & Subject2`, `Merge`, `Loop Over Items`

**Node Details:**

- **Get Transcript Data**
  - Type: HTTP Request
  - Role: Fetches transcript metadata and content links
  - Input: From `Keep Matches`
  - Output: Splits to `Split Out Transcriptions` and `Search OneDrive for Meeting`
  
- **Split Out Transcriptions**
  - Type: SplitOut
  - Role: Splits transcript data array into individual transcript items
  - Input: From `Get Transcript Data`
  - Output: To `Get Transcript`
  
- **Get Transcript**
  - Type: HTTP Request
  - Role: Retrieves individual transcript content
  - Input: From `Split Out Transcriptions`
  - Output: To `Set Transcription & Subject`
  
- **Set Transcription & Subject / Set Transcription & Subject1 / Set Transcription & Subject2**
  - Type: Set node
  - Role: Assigns transcription text and subject metadata for AI input
  - Input: From `Get Transcript` / `Get Transcript1` / `If2`
  - Output: To `Merge`
  
- **Merge**
  - Type: Merge node
  - Role: Combines multiple transcription and subject sets into a single data stream
  - Input: Receives data from three Set nodes
  - Output: To `Loop Over Items`
  
- **Loop Over Items**
  - Type: SplitInBatches
  - Role: Processes transcripts in batches for AI summarization, with one branch iterating and the other triggering AI agent
  - Input: From `Merge`
  - Output: To `AI Agent Create Summary` (on second output)

---

#### 1.4 AI Analysis & Comment Generation

- **Overview:** Uses GPT-4o powered LangChain agents and OpenAI chat models to analyze transcripts, generate summaries, and create contextual YouTube comments or follow-ups.
- **Nodes Involved:** `OpenAI Chat Model`, `OpenAI Chat Model1`, `AI Agent Create Summary`, `AI Agent Draft Follow Up`, `Set JSON`, `Set Email JSON`

**Node Details:**

- **OpenAI Chat Model**
  - Type: LangChain OpenAI Chat Model node
  - Role: Performs initial AI summarization of transcript batches
  - Input: From `Loop Over Items`
  - Output: To `AI Agent Create Summary`
  - Config: Uses GPT-4o or similar model with prompt templates
  - Edge cases: API key limits, prompt formatting errors
  
- **AI Agent Create Summary**
  - Type: LangChain Agent
  - Role: Further processes AI summaries, possibly refining or structuring
  - Input: From `OpenAI Chat Model`
  - Output: To `Set JSON`
  
- **OpenAI Chat Model1**
  - Type: LangChain OpenAI Chat Model node
  - Role: Creates draft follow-up comments or emails using AI
  - Input: From `AI Agent Draft Follow Up`
  - Output: To `AI Agent Draft Follow Up`
  
- **AI Agent Draft Follow Up**
  - Type: LangChain Agent
  - Role: Drafts follow-up content based on AI suggestions
  - Input: From `OpenAI Chat Model1`
  - Output: To `Set Email JSON`
  
- **Set JSON**
  - Type: Set node
  - Role: Prepares JSON structure for email and summary HTML content
  - Input: From `AI Agent Create Summary`
  - Output: To `Email HTML` and `Summary HTML`
  
- **Set Email JSON**
  - Type: Set node
  - Role: Prepares JSON for follow-up email content
  - Input: From `AI Agent Draft Follow Up`
  - Output: To `Send Follow Up`

---

#### 1.5 Output Handling & Notifications

- **Overview:** Formats AI-generated content to HTML, saves summarized data to the database, and sends email notifications.
- **Nodes Involved:** `Email HTML`, `Summary HTML`, `Save Data`, `End Analysis Notification`, `Send Follow Up`

**Node Details:**

- **Email HTML**
  - Type: HTML node
  - Role: Converts JSON content to HTML email format for notifications
  - Input: From `Set JSON`
  - Output: To `End Analysis Notification`
  
- **Summary HTML**
  - Type: HTML node
  - Role: Converts summary JSON to HTML for display or storage
  - Input: From `Set JSON`
  - Output: To `Save Data`
  
- **Save Data**
  - Type: Postgres node
  - Role: Saves summarized and processed transcript data back to the database
  - Input: From `Summary HTML`
  - Output: To `Loop Over Items` (possibly for iterative saving)
  
- **End Analysis Notification**
  - Type: Microsoft Outlook node
  - Role: Sends final email notification that analysis is complete
  - Input: From `Email HTML`
  - Output: None
  - Config: Requires Outlook OAuth2 credentials
  - Edge cases: Email send failures, auth errors
  
- **Send Follow Up**
  - Type: Microsoft Outlook node
  - Role: Sends AI-generated follow-up emails or comments
  - Input: From `Set Email JSON`
  - Output: To `Respond With Code 200`
  - Config: Requires Outlook OAuth2 credentials
  - Edge cases: Same as above

---

#### 1.6 Follow-up Actions & Webhook Responses

- **Overview:** Handles responses to external webhook calls and manages user interactions or form submissions.
- **Nodes Involved:** `Respond to Webhook1`, `Respond With WebApp HTML`, `Respond With Code 200`

**Node Details:**

- **Respond to Webhook1**
  - Type: Respond to Webhook node
  - Role: Sends HTTP response after memory note or processing completion
  - Input: From `Mem Note`
  - Output: None
  
- **Respond With WebApp HTML**
  - Type: Respond to Webhook node
  - Role: Returns HTML content to web app clients after meeting row retrieval
  - Input: From `Get Meeting Row1`
  
- **Respond With Code 200**
  - Type: Respond to Webhook node
  - Role: Sends HTTP 200 OK response after follow-up email send
  - Input: From `Send Follow Up`

---

### 3. Summary Table

| Node Name                | Node Type                                   | Functional Role                      | Input Node(s)                | Output Node(s)                  | Sticky Note                                |
|--------------------------|---------------------------------------------|------------------------------------|-----------------------------|---------------------------------|--------------------------------------------|
| Schedule Trigger         | scheduleTrigger                             | Trigger processing every 5 minutes | None                        | Merge1                         | Poll for new transcripts every 5 minutes  |
| On form submission       | formTrigger                                | Trigger on form submission          | None                        | Code2                         |                                            |
| Web Page                 | webhook                                    | Receive web requests                | None                        | Get Meeting Row1               |                                            |
| Next Action              | webhook                                    | Receive next action requests        | None                        | Get Meeting Row                |                                            |
| Get Meeting Row1         | postgres                                   | Fetch meeting data                  | Web Page                    | Respond With WebApp HTML       |                                            |
| Get Meeting Row          | postgres                                   | Fetch meeting data for next action | Next Action                 | Markdown                      |                                            |
| Get Call Records         | httpRequest                                | Retrieve call records               | Set Time Range              | Code1                        |                                            |
| Code1                    | code                                       | Process call records                | Get Call Records            | Filter for Join Url1           |                                            |
| Filter for Join Url1     | filter                                     | Filter calls with join URLs         | Code1                      | Get Online Meeting Details1    |                                            |
| Get Online Meeting Details1 | httpRequest                              | Get detailed meeting info           | Filter for Join Url1        | Split Out Meetings1            |                                            |
| Split Out Meetings1      | splitOut                                   | Split meetings into items           | Get Online Meeting Details1 | Remove Duplicates              |                                            |
| Remove Duplicates        | removeDuplicates                           | Remove duplicate meetings           | Split Out Meetings1         | Sort1                        |                                            |
| Sort1                    | sort                                       | Sort meetings                      | Remove Duplicates           | Check if Processed             |                                            |
| Check if Processed       | postgres                                   | Check if meeting already processed  | Sort1                      | Filter No Items, Keep Matches  |                                            |
| Filter No Items          | filter                                     | Filter out empty results            | Check if Processed          | Get Items                    |                                            |
| Get Items                | code                                       | Extract relevant items              | Filter No Items             | Keep Matches                 |                                            |
| Keep Matches             | merge                                      | Combine new and matched items       | Get Items, Check if Processed | Get Transcript Data          |                                            |
| Get Transcript Data      | httpRequest                                | Fetch transcript metadata           | Keep Matches                | Split Out Transcriptions, Search OneDrive for Meeting |                                            |
| Split Out Transcriptions | splitOut                                   | Split transcripts into items        | Get Transcript Data         | Get Transcript               |                                            |
| Get Transcript           | httpRequest                                | Retrieve individual transcript      | Split Out Transcriptions    | Set Transcription & Subject   |                                            |
| Set Transcription & Subject | set                                     | Assign transcription and subject    | Get Transcript              | Merge                        |                                            |
| Set Transcription & Subject1 | set                                     | Assign transcription and subject    | Get Transcript1             | Merge                        |                                            |
| Set Transcription & Subject2 | set                                     | Assign transcription and subject    | If2                        | Merge                        |                                            |
| Merge                    | merge                                      | Combine transcription data          | Multiple Set nodes          | Loop Over Items              |                                            |
| Loop Over Items          | splitInBatches                            | Batch processing of transcripts     | Merge                      | AI Agent Create Summary       |                                            |
| OpenAI Chat Model        | lmChatOpenAi                               | AI summarization                    | Loop Over Items             | AI Agent Create Summary       |                                            |
| AI Agent Create Summary  | agent                                      | Process AI summaries                | OpenAI Chat Model           | Set JSON                     |                                            |
| Set JSON                 | set                                        | Prepare JSON for email and summary | AI Agent Create Summary     | Email HTML, Summary HTML      |                                            |
| Email HTML               | html                                       | Format email content as HTML        | Set JSON                   | End Analysis Notification    |                                            |
| Summary HTML             | html                                       | Format summary content as HTML      | Set JSON                   | Save Data                   |                                            |
| Save Data                | postgres                                   | Save processed data                 | Summary HTML               | Loop Over Items              |                                            |
| End Analysis Notification | microsoftOutlook                         | Send final analysis email           | Email HTML                 | None                        |                                            |
| AI Agent Draft Follow Up | agent                                      | Draft follow-up content             | OpenAI Chat Model1          | Set Email JSON               |                                            |
| OpenAI Chat Model1       | lmChatOpenAi                               | Generate draft follow-up            | AI Agent Draft Follow Up    | AI Agent Draft Follow Up      |                                            |
| Set Email JSON           | set                                        | Prepare email JSON                  | AI Agent Draft Follow Up    | Send Follow Up              |                                            |
| Send Follow Up           | microsoftOutlook                           | Send follow-up email                | Set Email JSON             | Respond With Code 200        |                                            |
| Respond to Webhook1      | respondToWebhook                           | Respond to webhook call             | Mem Note                   | None                        |                                            |
| Respond With WebApp HTML | respondToWebhook                           | Return HTML response to web client  | Get Meeting Row1           | None                        |                                            |
| Respond With Code 200    | respondToWebhook                           | Send HTTP 200 OK response           | Send Follow Up             | None                        |                                            |
| Set Time Range           | set                                        | Set time range parameters           | None                       | Get Call Records             | Set time range in minutes                   |
| Mem Note                 | httpRequest                                | Possibly a logging or memo call     | Switch                     | Respond to Webhook1          |                                            |
| Markdown                 | markdown                                   | Format markdown text                | Get Meeting Row            | Switch                      |                                            |
| Switch                   | switch                                     | Decision branching based on conditions | Markdown                 | Mem Note, AI Agent Draft Follow Up |                                            |
| Merge1                   | merge                                      | Merge trigger inputs                | Schedule Trigger, Code2     | Get Microsoft 365 Profile    |                                            |
| Get Microsoft 365 Profile | httpRequest                              | Fetch user profile from Microsoft 365 | Merge1                  | If2                         |                                            |
| If2                      | if                                         | Conditional branching               | Get Microsoft 365 Profile  | Set Transcription & Subject2, Set Time Range |                                            |
| Code2                    | code                                       | Processes form submission data      | On form submission         | Merge1                      |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger Node:**
   - Name: `Schedule Trigger`
   - Type: Schedule Trigger
   - Set to trigger every 5 minutes.

2. **Create Webhook Trigger Nodes:**
   - `On form submission` (Form Trigger) with a webhook ID for form data reception.
   - `Web Page` (Webhook) to receive initial web requests.
   - `Next Action` (Webhook) for subsequent action triggers.

3. **Create Postgres Nodes to Query Meetings:**
   - `Get Meeting Row1` to fetch meeting data on initial web request.
   - `Get Meeting Row` to fetch meeting data for follow-ups.
   - `Check if Processed` to verify if meetings/transcripts have been processed.
   - `Save Data` to store processed summaries.

4. **Create HTTP Request Nodes for API Calls:**
   - `Get Call Records` to fetch call metadata from Microsoft Graph API.
   - `Get Online Meeting Details1` to fetch detailed info of filtered calls.
   - `Search OneDrive for Meeting` to search for transcript files.
   - `List Transcripts2` and `Get Transcript1` to list and retrieve transcripts.

5. **Create Data Processing Nodes:**
   - `Code1` and `Code2` for JavaScript data manipulation.
   - `Filter for Join Url1` to filter for calls with valid join URLs.
   - `Remove Duplicates` to remove duplicate meetings.
   - `Sort1` to sort meeting records.
   - `Filter No Items` to filter out empty results.
   - `Get Items` to extract relevant items.
   - `Keep Matches` (Merge node) to combine data streams.
   - `Merge1` to combine trigger inputs.

6. **Add Split and Merge Nodes:**
   - `Split Out Meetings1` and `Split Out3` to split arrays into individual items.
   - `Split Out Transcriptions` to split transcript arrays.
   - `Merge` to combine transcription & subject sets.

7. **Add Set Nodes to Assign Variables:**
   - `Set Transcription & Subject`, `Set Transcription & Subject1`, and `Set Transcription & Subject2` to prepare AI inputs.
   - `Set JSON` and `Set Email JSON` to prepare content for output nodes.
   - `Set Time Range` to define time range parameters for calls retrieval.

8. **Add AI Nodes:**
   - `OpenAI Chat Model` and `OpenAI Chat Model1` configured with GPT-4o or appropriate OpenAI credentials.
   - `AI Agent Create Summary` and `AI Agent Draft Follow Up` LangChain agents for AI-driven summarization and drafting.

9. **Add Output Formatting and Notification Nodes:**
   - `Email HTML` and `Summary HTML` to convert JSON content to HTML.
   - `End Analysis Notification` and `Send Follow Up` Microsoft Outlook nodes configured with OAuth2 credentials.
   - `Respond to Webhook1`, `Respond With WebApp HTML`, and `Respond With Code 200` to send HTTP responses.

10. **Add Utility Nodes:**
    - `Markdown` to format text.
    - `Switch` and `If2` for conditional logic.
    - `Mem Note` HTTP Request node (purpose likely memo or logging).
    - Sticky notes for documentation (optional but recommended).

11. **Connect Nodes per the Logical Flow:**
    - Triggers to data retrieval nodes.
    - Data retrieval to filtering and splitting nodes.
    - Splitting to AI processing nodes.
    - AI outputs to formatting and saving nodes.
    - Final output nodes to email/send responses.

12. **Configure Credentials:**
    - Set up Postgres credentials for database access.
    - Set up Microsoft 365 OAuth2 credentials for Graph API and Outlook email.
    - Set up OpenAI API credentials for GPT-4o usage.

13. **Validate and Test:**
    - Test each trigger independently.
    - Simulate API responses for meeting and transcript data.
    - Confirm AI nodes generate expected summaries and drafts.
    - Ensure email notifications and webhook responses are correctly sent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                         | Context or Link                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow requires valid Microsoft 365 API credentials for fetching meeting and transcript data, as well as sending emails via Outlook.           | Microsoft Graph API and Outlook OAuth2 documentation |
| OpenAI GPT-4o or compatible models must be configured with API keys and appropriate prompt engineering for best results.                            | OpenAI API docs: https://platform.openai.com/docs |
| Sticky notes in the workflow provide inline comments and reminders but do not affect execution logic.                                                | Workflow visualization in n8n editor             |
| The workflow respects data privacy by only processing transcripts and meetings authorized by the connected Microsoft 365 account.                  | User data and privacy policy considerations      |
| For troubleshooting, monitor API rate limits and errors in HTTP Request and AI nodes for potential timeouts or auth issues.                        | n8n logs and node error handling                  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.