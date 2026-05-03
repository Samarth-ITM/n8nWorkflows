Transform Voice Memos into Daily Journals & Tasks with OMI.ME, Gemini AI & Google Drive

https://n8nworkflows.xyz/workflows/transform-voice-memos-into-daily-journals---tasks-with-omi-me--gemini-ai---google-drive-5662


# Transform Voice Memos into Daily Journals & Tasks with OMI.ME, Gemini AI & Google Drive

---
### 1. Workflow Overview

This n8n workflow automates the transformation of voice memos captured by the OMI.ME AI pendant into structured daily journals and actionable tasks. It targets users who want to seamlessly archive their personal memories and productivity-related action items using Google Drive for storage, Google Tasks for task management, and AI models (Google Gemini or OpenAI) for text summarization and task extraction.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception and Preprocessing:** Receives incoming voice memo data via webhook, filters for meaningful content, and extracts relevant transcript and structured metadata.

- **1.2 Daily Journal Construction:** Builds markdown-formatted journal entries from raw transcripts and metadata, stores these entries in Google Drive, and aggregates daily summaries.

- **1.3 AI Processing and Summarization:** Uses AI models to convert aggregated raw transcripts into polished, first-person journal entries reflecting emotional nuance and personal reflection.

- **1.4 Task Extraction and Management:** Extracts action items from the AI-processed data, formats them, and creates tasks in Google Tasks for follow-up.

- **1.5 Scheduled Daily Aggregation:** A scheduled trigger runs nightly to compile daily memos, generate journal entries, and clean up processed files.

- **1.6 Auxiliary and Maintenance:** Includes nodes for handling no-op flows, waiting periods, and error filtering.

This modular design supports extensibility for alternative AI models, task management platforms, and storage backends.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

**Overview:**  
This block captures incoming webhook POST requests from the OMI device containing voice memo data. It filters out short or incomplete memos, extracts key transcript segments, and relevant metadata such as title, overview, category, emoji, and app-generated summaries.

**Nodes Involved:**  
- Webhook  
- Check if app ran on memory (If)  
- Fix Transcript and extract relevant values (Set)  
- Check for Tasks (If)  
- Do Nothing (NoOp)

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook (POST)  
  - Configured to receive JSON payloads on a unique path.  
  - Input: External POST requests from OMI device webhook.  
  - Output: Raw JSON body with voice memo data.  
  - Edge Cases: Missing or malformed requests, auth not handled at this node.  

- **Check if app ran on memory (If)**  
  - Type: Conditional filter  
  - Checks if the transcript content exists to avoid processing empty or very short memos.  
  - Expression: Checks existence of `$json.body.apps_results[0].content` string field.  
  - Input: Webhook node output.  
  - Output: Branches into processing or no-op.  
  - Edge Cases: Empty or missing transcript content causes flow to Do Nothing.  

- **Fix Transcript and extract relevant values (Set)**  
  - Type: Data transformation  
  - Extracts and formats transcript segments as a concatenated string with speaker labels.  
  - Extracts structured metadata: title, overview, emoji, category, app summary content, and app name.  
  - Uses expressions to map raw JSON fields to new properties.  
  - Input: Passed only if transcript exists.  
  - Output: Structured JSON with cleaned transcript and metadata.  
  - Edge Cases: Missing expected fields may cause empty values or errors; expression failures possible if input structure changes.  

- **Check for Tasks (If)**  
  - Type: Conditional filter  
  - Checks if extracted structured action items exist and have length ≥ 1.  
  - Input: From Fix Transcript node, referencing `$json.body.structured.action_items`.  
  - Output: Branches to task extraction or skips task processing.  
  - Edge Cases: Empty or invalid action items array.  

- **Do Nothing (NoOp)**  
  - Type: No operation  
  - Used to terminate flow gracefully when no processing is needed.  

---

#### 2.2 Daily Journal Construction

**Overview:**  
This block constructs markdown-formatted journal entries from the extracted transcript and metadata, converts them into files, and uploads them to Google Drive in dedicated folders for daily summaries and long-term transcription storage.

**Nodes Involved:**  
- Build Markdown Transcription (Set)  
- Convert Transcription to MD file (ConvertToFile)  
- Upload Transcription to Gdrive for Long Term (Google Drive)  
- Upload Daily Summary to GDrive (Google Drive)

**Node Details:**

- **Build Markdown Transcription (Set)**  
  - Type: Data transformation  
  - Constructs a markdown text string combining emoji, category, current date/time, title, overview, and formatted transcript segments.  
  - Input: Cleaned data from Fix Transcript node.  
  - Output: JSON with `text` property containing markdown string.  
  - Edge Cases: Missing fields result in incomplete markdown content.  

- **Convert Transcription to MD file (ConvertToFile)**  
  - Type: File conversion  
  - Converts the markdown text into a UTF-8 encoded `.md` file named with date and title.  
  - Input: `text` property from previous Set node.  
  - Output: Binary file data ready for upload.  
  - Edge Cases: File naming collisions or invalid characters may cause failures.  

- **Upload Transcription to Gdrive for Long Term (Google Drive)**  
  - Type: Google Drive file upload  
  - Uploads the markdown file to a specified "OMI Transcriptions" folder on Google Drive for archival.  
  - Uses OAuth2 credentials for authentication.  
  - Input: File data from ConvertToFile.  
  - Output: Metadata about uploaded file.  
  - Edge Cases: Auth token expiration, API quota limits, folder permission issues.  

- **Upload Daily Summary to GDrive (Google Drive)**  
  - Type: Google Drive file upload  
  - Uploads the same markdown file to a "Daily Summaries" Google Drive folder, organized for daily journal review.  
  - Input/Output similar to above.  

---

#### 2.3 AI Processing and Summarization

**Overview:**  
Aggregates multiple daily markdown files, sorts them chronologically, merges their content, and sends the combined text to an AI agent for generating emotionally rich, first-person journal entries. The AI uses detailed system prompts to guide style and tone.

**Nodes Involved:**  
- Google Drive2 (Google Drive)  
- Loop Over Items (SplitInBatches)  
- Sort (Sort)  
- Aggregate (Aggregate)  
- AI Agent1 (Langchain AI Agent)  
- Edit Fields4 (Set)  
- Convert to File2 (ConvertToFile)  
- Google Drive4 (Google Drive)  
- Google Drive5 (Google Drive)  
- Google Drive6 (Google Drive)  
- Edit Fields5 (Set)  
- Wait (Wait)  
- Replace Me (NoOp)

**Node Details:**

- **Google Drive2 (Google Drive)**  
  - Lists all markdown files in the "Daily Summaries" folder for the current day.  
  - Input: Scheduled trigger.  
  - Output: List of files for processing.  

- **Loop Over Items (SplitInBatches)**  
  - Processes files sequentially in batches (default batch size).  
  - Input: Files from Google Drive2.  
  - Output: Individual file metadata for further processing.  

- **Sort (Sort)**  
  - Sorts files by name (date-based) to maintain chronological order.  
  - Input: From Loop Over Items.  
  - Output: Sorted file list.  

- **Aggregate (Aggregate)**  
  - Aggregates the text content of all files into a single array.  
  - Input: Extracted text fields from files.  
  - Output: Combined array of text segments.  

- **AI Agent1 (Langchain AI Agent)**  
  - Sends the aggregated text to a LangChain-powered AI agent (Google Gemini or OpenAI) with a detailed system prompt.  
  - The prompt guides the AI to write introspective, first-person journal entries reflecting personal growth, emotional nuance, and fatherhood.  
  - Input: Joined text array as prompt.  
  - Output: Polished journal entry text.  
  - Edge Cases: API rate limits, malformed prompt data, model errors.  

- **Edit Fields4 (Set)**  
  - Stores the AI output under `journal` field for further processing.  

- **Convert to File2 (ConvertToFile)**  
  - Converts the journal entry string into a `.md` file named by date and time.  

- **Google Drive4 (Google Drive)**  
  - Uploads the final journal entry file to the "AI Journal" folder on Google Drive.  

- **Google Drive5 (Google Drive)**  
  - Lists files in "Daily Summaries" folder for cleanup.  

- **Google Drive6 (Google Drive)**  
  - Deletes processed markdown files from "Daily Summaries" folder to prepare for the next day.  

- **Edit Fields5 (Set)**  
  - Prepares data for the Wait node.  

- **Wait (Wait)**  
  - Waits 3 seconds before continuing, likely to avoid API rate limiting or ensure ordered execution.  

- **Replace Me (NoOp)**  
  - Placeholder for possible future extensions or manual intervention points.  

---

#### 2.4 Task Extraction and Management

**Overview:**  
Extracts actionable tasks from the structured action items in the incoming webhook data, splits them into individual tasks, and creates corresponding Google Tasks in the user's task list.

**Nodes Involved:**  
- Check for Tasks (If)  
- extract tasks (Set)  
- Split Out tasks (SplitOut)  
- Create Google Tasks (Google Tasks)

**Node Details:**

- **Check for Tasks (If)**  
  - Checks for presence of action items array with at least one item.  
  - Input: From webhook processed data.  
  - Output: Branches to task extraction or skips.  

- **extract tasks (Set)**  
  - Extracts the action items array into a `tasks` field.  

- **Split Out tasks (SplitOut)**  
  - Splits the `tasks` array into individual task items for iterative processing.  

- **Create Google Tasks (Google Tasks)**  
  - Creates a task in Google Tasks for each individual task.  
  - Uses OAuth2 credentials for authentication.  
  - Maps `description` field from the task JSON to the task title in Google Tasks.  
  - Edge Cases: API quota, auth failures, malformed task data.  

---

#### 2.5 Scheduled Daily Aggregation

**Overview:**  
A scheduled trigger fires daily at 23:00 to initiate the aggregation and AI summarization of all daily markdown summaries, enabling end-of-day journal creation and cleanup.

**Nodes Involved:**  
- Schedule Trigger  
- Google Drive2  
- Loop Over Items  
- ... (continues into Daily Journal Construction and AI Processing blocks)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule trigger  
  - Configured to trigger daily at 23:00 (11 PM).  
  - Input: Time-based trigger.  
  - Output: Starts flow to process all daily summaries.  

---

#### 2.6 Auxiliary and Maintenance Nodes

**Overview:**  
Includes nodes for no-operation placeholders, waiting periods, and sorting that support flow control and error management.

**Nodes Involved:**  
- Do Nothing (NoOp)  
- Replace Me (NoOp)  
- Wait (Wait)  
- Sort (Sort)

**Node Details:**

- **Do Nothing (NoOp)**  
  - Stops flow execution gracefully when no further processing is needed.  

- **Replace Me (NoOp)**  
  - Placeholder for future nodes or manual intervention.  

- **Wait (Wait)**  
  - Delays execution by 3 seconds to manage timing issues or API limits.  

- **Sort (Sort)**  
  - Sorts inputs by specific field (e.g., filename) for ordered processing.  

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                               | Input Node(s)                    | Output Node(s)                | Sticky Note                                                                                                    |
|--------------------------------|----------------------------------|-----------------------------------------------|---------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook                        | n8n-nodes-base.webhook           | Receives OMI voice memo webhook POST data     | -                               | Check if app ran on memory    |                                                                                                               |
| Check if app ran on memory     | n8n-nodes-base.if                | Filters out very short or empty memos          | Webhook                         | Fix Transcript and extract relevant values, Do Nothing | This does not capture memories that are too short.                                                             |
| Fix Transcript and extract relevant values | n8n-nodes-base.set       | Extracts transcript segments and metadata      | Check if app ran on memory      | Build Markdown Transcription, Check for Tasks  |                                                                                                               |
| Check for Tasks                | n8n-nodes-base.if                | Checks if action items array exists and non-empty | Fix Transcript and extract relevant values | extract tasks, Do Nothing    |                                                                                                               |
| extract tasks                 | n8n-nodes-base.set               | Extracts tasks array for splitting              | Check for Tasks                 | Split Out tasks               |                                                                                                               |
| Split Out tasks               | n8n-nodes-base.splitOut          | Splits tasks array into individual task items  | extract tasks                  | Create Google Tasks           |                                                                                                               |
| Create Google Tasks           | n8n-nodes-base.googleTasks       | Creates tasks in Google Tasks from action items | Split Out tasks                | -                            |                                                                                                               |
| Build Markdown Transcription  | n8n-nodes-base.set               | Builds markdown string from transcript and metadata | Fix Transcript and extract relevant values | Convert Transcription to MD file |                                                                                                               |
| Convert Transcription to MD file | n8n-nodes-base.convertToFile    | Converts markdown text to .md file               | Build Markdown Transcription   | Upload Transcription to Gdrive for Long Term, Upload Daily Summary to GDrive |                                                                                                               |
| Upload Transcription to Gdrive for Long Term | n8n-nodes-base.googleDrive | Uploads markdown file to long-term transcription storage | Convert Transcription to MD file | -                            |                                                                                                               |
| Upload Daily Summary to GDrive | n8n-nodes-base.googleDrive       | Uploads markdown file to daily summaries folder | Convert Transcription to MD file | -                            |                                                                                                               |
| Schedule Trigger              | n8n-nodes-base.scheduleTrigger   | Triggers nightly aggregation and journaling     | -                               | Google Drive2                |                                                                                                               |
| Google Drive2                | n8n-nodes-base.googleDrive       | Lists daily markdown summary files               | Schedule Trigger               | Loop Over Items              |                                                                                                               |
| Loop Over Items              | n8n-nodes-base.splitInBatches    | Iterates over daily summary files                | Google Drive2                  | Sort, Google Drive3          |                                                                                                               |
| Sort                        | n8n-nodes-base.sort              | Sorts files chronologically                       | Loop Over Items                | Aggregate                   |                                                                                                               |
| Aggregate                   | n8n-nodes-base.aggregate         | Aggregates text content of all daily files       | Sort                          | AI Agent1                   |                                                                                                               |
| AI Agent1                   | @n8n/n8n-nodes-langchain.agent   | Generates polished journal entry from aggregated text | Aggregate                   | Edit Fields4                |                                                                                                               |
| Edit Fields4               | n8n-nodes-base.set               | Stores AI output as journal text                  | AI Agent1                     | Convert to File2            |                                                                                                               |
| Convert to File2           | n8n-nodes-base.convertToFile      | Converts journal text to markdown file             | Edit Fields4                  | Google Drive4               |                                                                                                               |
| Google Drive4              | n8n-nodes-base.googleDrive        | Uploads final journal entry to AI Journal folder  | Convert to File2              | Google Drive5               |                                                                                                               |
| Google Drive5              | n8n-nodes-base.googleDrive        | Lists files in Daily Summaries folder for cleanup | Google Drive4                | Google Drive6               |                                                                                                               |
| Google Drive6              | n8n-nodes-base.googleDrive        | Deletes processed daily summary files              | Google Drive5                | -                          |                                                                                                               |
| Edit Fields5               | n8n-nodes-base.set                | Prepares flow data for wait node                   | Extract from File             | Wait                       |                                                                                                               |
| Wait                      | n8n-nodes-base.wait               | Delays flow execution by 3 seconds                 | Edit Fields5                   | Replace Me                 |                                                                                                               |
| Replace Me                | n8n-nodes-base.noOp              | Placeholder node                                   | Wait                          | Loop Over Items            |                                                                                                               |
| Do Nothing                | n8n-nodes-base.noOp              | No-operation node to end flow when no processing needed | Check if app ran on memory | -                          |                                                                                                               |

Sticky Note covering initial nodes:  
"## Capture Memories to Google Drive and Google Tasks  
This extracts memories for daily journal and long term storage, as well as actions to Google tasks. Swap out the google tasks node for any todo list app or the HTTP request node for any task api."

Sticky Note covering workflow overview:  
"## Daily Journal  
This workflow captures daily memory summaries from the OMI AI pendant and turns them into structured journal entries and actionable tasks. Memories are stored in Google Drive as Markdown files, while tasks are parsed and pushed to Google Tasks for follow-up. Powered by Gemini AI or OpenAI, it's perfect for automating self-reflection, planning, and memory-keeping."

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create a Webhook Node**  
- Type: Webhook  
- HTTP Method: POST  
- Path: Unique path (e.g., `f83dfcc2-b83a-40ea-bd99-3c622fa1bc0e`)  
- Purpose: Receive incoming voice memo JSON from OMI device.  

**Step 2: Add an If Node "Check if app ran on memory"**  
- Checks if `$json.body.apps_results[0].content` exists and is non-empty.  
- True: Continue processing  
- False: Connect to NoOp node (Do Nothing) to halt flow.  

**Step 3: Add a Set Node "Fix Transcript and extract relevant values"**  
- Extract and set fields:  
  - `transcript_segments`: Concatenate each segment as `User/Speaker: text`  
  - `title`, `overview`, `emoji`, `category`, `app_summary`, `app_name` from structured JSON paths.  

**Step 4: Add an If Node "Check for Tasks"**  
- Check if `body.structured.action_items` array length ≥ 1.  
- True: Connect to task extraction steps  
- False: Continue without task creation.  

**Step 5: Task Extraction Workflow**  
- Add Set Node "extract tasks": Assign `tasks` field from `$json.body.structured.action_items`.  
- Add SplitOut Node "Split Out tasks": Splits `tasks` array into single task items.  
- Add Google Tasks Node "Create Google Tasks":  
  - Credential: OAuth2 with Google Tasks API  
  - Task list: Select or configure your task list  
  - Title: Map to task description from split item.  

**Step 6: Daily Journal Construction**  
- Add Set Node "Build Markdown Transcription": Construct markdown combining emoji, category, date/time, title, overview, and transcript.  
- Add ConvertToFile Node "Convert Transcription to MD file":  
  - Source Property: markdown text from previous node  
  - File Name: Use current date/time and title, e.g., `{{ $now.format('yyyy-MM-dd t') }} - {{ $json.title }}.md`  
- Add Google Drive Node "Upload Transcription to Gdrive for Long Term":  
  - Upload file to "OMI Transcriptions" folder  
  - Credentials: Google Drive OAuth2  
- Add Google Drive Node "Upload Daily Summary to GDrive":  
  - Upload same file to "Daily Summaries" folder  

**Step 7: Scheduled Aggregation and AI Processing**  
- Add Schedule Trigger Node:  
  - Time: Every day at 23:00  
- Add Google Drive Node "Google Drive2": List all `.md` files in "Daily Summaries" folder  
- Add SplitInBatches Node "Loop Over Items": Iterate over each file  
- Add Sort Node: Sort files by name (date) ascending  
- Add Google Drive Node "Google Drive3": Download each file in iteration  
- Add ExtractFromFile Node "Extract from File": Extract text content from `.md` files  
- Add Set Node "Edit Fields5": Assign extracted text to a field for aggregation  
- Add Aggregate Node "Aggregate": Combine all extracted texts into an array  
- Add LangChain AI Agent Node "AI Agent1":  
  - Pass aggregated text joined by separator as input  
  - Provide detailed system prompt to instruct AI to write personalized journal entries in first person tone  
  - Use Google Gemini or OpenAI credentials as desired  
- Add Set Node "Edit Fields4": Store AI output under `journal` property  
- Add ConvertToFile Node "Convert to File2": Convert journal text to `.md` file named by date/time  
- Add Google Drive Node "Google Drive4": Upload final journal entry file to "AI Journal" folder  
- Add Google Drive Node "Google Drive5": List all files in "Daily Summaries" folder for cleanup  
- Add Google Drive Node "Google Drive6": Delete processed summary files  
- Add Wait Node: Wait 3 seconds to throttle  
- Add NoOp Node "Replace Me": Placeholder or end of flow  

**Step 8: Auxiliary Nodes**  
- Add NoOp Node "Do Nothing" to gracefully end processing for empty inputs.  
- Add Sort Node for ordering files in aggregation.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow extracts memories for journaling and tasks, storing in Google Drive and Google Tasks. Replace Google Tasks node for other todo apps or task APIs as needed.                     | Sticky note on input and task extraction section                                                  |
| Workflow powered by Gemini AI (Google PaLM) or OpenAI models for text summarization and task extraction with detailed system prompts for personalized, emotionally intelligent journals. | Sticky note on AI Processing section                                                              |
| Workflow designed for OMI.ME AI Pendant voice memos, targeting users interested in automating self-reflection, planning, and memory-keeping.                                           | Workflow description and sticky notes                                                             |
| Uses OAuth2 credentials for Google Drive and Google Tasks integration; ensure these are properly configured in n8n instance before running workflow.                                     | Credential requirements for Google API nodes                                                      |
| Includes scheduled nightly trigger to aggregate and process daily journal entries for end-of-day summary creation.                                                                       | Schedule Trigger node configuration                                                                |
| Possible edge cases include API rate limits, auth token expiration, malformed input data, and network errors during Google API calls.                                                   | General considerations for all external API nodes                                                 |
| AI prompt includes detailed persona and style instructions to generate thoughtful, first-person journal entries reflecting fatherhood and personal growth.                             | AI Agent system message content                                                                    |
| Task extraction relies on action items provided by OMI device structured payload; may require prompt tuning for accuracy and completeness.                                              | Task extraction and AI prompt details                                                             |
| The workflow can be extended or adapted for other AI models, task managers, or storage platforms by swapping relevant nodes and credentials.                                           | Extensibility note                                                                                |

---

**Disclaimer:**  
The text and content described are derived exclusively from an automated workflow created using n8n, respecting all applicable content policies. The workflow processes only legal and public data. No illegal, offensive, or protected content is included.