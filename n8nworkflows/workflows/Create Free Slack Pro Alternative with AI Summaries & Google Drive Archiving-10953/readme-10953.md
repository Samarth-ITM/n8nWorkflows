Create Free Slack Pro Alternative with AI Summaries & Google Drive Archiving

https://n8nworkflows.xyz/workflows/create-free-slack-pro-alternative-with-ai-summaries---google-drive-archiving-10953


# Create Free Slack Pro Alternative with AI Summaries & Google Drive Archiving

### 1. Workflow Overview

This workflow provides a free Slack Pro alternative by combining AI-powered channel message summarization with automated monthly archiving of Slack channel history to Google Drive. It serves two main use cases:

- **AI Summarization Assistant:** When a user @mentions the bot in any Slack channel with a command like “summarize last 3 days,” the workflow extracts the time range, fetches channel history and user info, uses an AI agent to generate a structured summary, and posts that summary back into the Slack channel.

- **Monthly Compliance Backup:** On a monthly schedule, the workflow loops through all Slack channels, exports the last 30 days of messages from each channel, converts the raw data into files, and uploads them to a designated Google Drive folder for archival and compliance.

The workflow is organized into the following logical blocks:

- **1.1 Event & Context Extraction:** Captures Slack app_mention events and extracts user command text.
- **1.2 AI Processing & Summarization:** Invokes an AI agent (using Google Gemini) to parse the command, retrieve message history, resolve user IDs to names, maintain conversation memory, and generate summaries.
- **1.3 Summary Reply:** Sends the AI-generated summary back to the relevant Slack channel.
- **2.1 Scheduled Channel History Export:** On a monthly trigger, fetches all Slack channels and iterates over them.
- **2.2 Channel History Retrieval & Upload:** Retrieves past 30 days of history per channel, converts to file, and uploads to Google Drive.

---

### 2. Block-by-Block Analysis

#### 1.1 Event & Context Extraction

**Overview:**  
This block listens for Slack app_mention events (users tagging the bot) and extracts the clean user prompt text from the complex Slack event payload to prepare input for AI processing.

**Nodes Involved:**  
- On Slack App Mention Trigger  
- Extract User Prompt Text from Slack Event

**Node Details:**

- **On Slack App Mention Trigger**  
  - Type: Slack Trigger  
  - Role: Listens for app_mention events across the workspace  
  - Configuration: Watches entire workspace; triggers when bot is @mentioned  
  - Inputs: Slack workspace webhook  
  - Outputs: Raw Slack event JSON with full message data  
  - Failures: Possible auth errors if Slack credentials expire; webhook connectivity issues

- **Extract User Prompt Text from Slack Event**  
  - Type: Set node  
  - Role: Extracts the user’s text command from the Slack event JSON payload  
  - Configuration: Uses expression to retrieve the text from `$json.blocks[0].elements[0].elements[1].text` which corresponds to user input after the bot mention  
  - Inputs: Output of Slack trigger  
  - Outputs: JSON with a clean `input` field containing user prompt text  
  - Failures: May fail if Slack message format changes or user message structure differs (missing blocks array)

---

#### 1.2 AI Processing & Summarization

**Overview:**  
The core logic uses an AI agent (Google Gemini) configured with a detailed system prompt to interpret user commands, parse required time ranges, call Slack tools to fetch channel history and user profiles, and maintain conversation context with memory. It outputs a structured summary for Slack.

**Nodes Involved:**  
- Slack Channel Summary AI Agent  
- LLM for Message Summaries  
- Channel Conversation Memory Buffer  
- Get Slack Channel History for Summaries  
- Get Slack User Profile by ID

**Node Details:**

- **Slack Channel Summary AI Agent**  
  - Type: Langchain Agent (AI Orchestrator)  
  - Role: Central AI agent interpreting user commands, orchestrating data fetch calls, and generating summaries  
  - Configuration:  
    - Receives user input text  
    - System message instructs the agent to parse relative time expressions (e.g., last 3 days, yesterday) into timestamps  
    - Calls Slack tools to get channel message history and user info  
    - Uses conversation memory to maintain context in the same channel  
    - Outputs summary text formatted without markdown but with structured topic blocks  
  - Inputs:  
    - User prompt text from Set node  
    - Slack channel ID from trigger  
  - Outputs: Summary text to be posted back to Slack  
  - Failures:  
    - AI model failures or timeouts  
    - Slack API limit errors on history or user info calls  
    - Parsing errors if user input is ambiguous or malformed  
  - Requires: Google Gemini credentials for LLM, Slack API credentials for tools

- **LLM for Message Summaries**  
  - Type: Langchain Chat Model (Google Gemini)  
  - Role: Provides the language model backend for the AI agent’s processing  
  - Configuration: Uses Google Gemini API with configured credentials  
  - Inputs: Text prompts from AI Agent  
  - Outputs: Text completions for summarization  
  - Failures: API quota exceeded, network errors

- **Channel Conversation Memory Buffer**  
  - Type: Langchain Memory Buffer  
  - Role: Maintains recent conversation context keyed by Slack channel to avoid repetitive questions and keep session state  
  - Configuration: Context window length 10 messages, session keyed by Slack channel ID  
  - Inputs/Outputs: Passes memory data in AI agent context  
  - Failures: None expected unless memory store fails

- **Get Slack Channel History for Summaries**  
  - Type: Slack Tool (channel history)  
  - Role: Fetches all messages from the current Slack channel within defined time range  
  - Configuration:  
    - Uses time range parameters (`oldest`, `latest`) parsed by AI agent  
    - Returns all messages in range  
    - Channel ID preset to the channel where bot was mentioned  
  - Inputs: Channel ID, time range from AI agent  
  - Outputs: List of messages for summarization  
  - Failures: Slack API limits, auth failures

- **Get Slack User Profile by ID**  
  - Type: Slack Tool (user lookup)  
  - Role: Resolves Slack user IDs from messages into display names or fallbacks for summary readability  
  - Configuration:  
    - Input is Slack user ID extracted by AI agent  
    - Returns profile info including `display_name` or `real_name`  
    - Fallbacks to mention syntax `<@UserID>` if no name available  
  - Inputs: Slack user ID string  
  - Outputs: User profile JSON  
  - Failures: Slack API errors, missing profiles

---

#### 1.3 Summary Reply

**Overview:**  
This block sends the AI-generated summary text back into the originating Slack channel, simulating Slack Pro’s summary message feature.

**Nodes Involved:**  
- Send Summary Message to Slack Channel

**Node Details:**

- **Send Summary Message to Slack Channel**  
  - Type: Slack node (post message)  
  - Role: Posts the summary text into the Slack channel where the request originated  
  - Configuration:  
    - Text content bound to AI agent’s output field  
    - Channel ID bound to Slack trigger’s channel ID  
    - Markdown enabled for Slack formatting (though AI agent outputs plain text format)  
  - Inputs: Summary text from AI Agent  
  - Outputs: Confirmation of message sent  
  - Failures: Slack API permissions, channel not found, rate limits

---

#### 2.1 Scheduled Channel History Export

**Overview:**  
This scheduled block initiates the monthly archival process by triggering once a month, fetching all Slack channels in the workspace, and preparing for iterative processing.

**Nodes Involved:**  
- Monthly Export Schedule Trigger  
- Get All Slack Channels in Workspace  
- Loop Over Slack Channels

**Node Details:**

- **Monthly Export Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Fires workflow monthly at 1 AM server time  
  - Configuration: Monthly interval, trigger hour 1  
  - Inputs: None (time-based trigger)  
  - Outputs: Timestamp event to start export  
  - Failures: None expected

- **Get All Slack Channels in Workspace**  
  - Type: Slack node (getAll channels)  
  - Role: Retrieves complete list of Slack channels in workspace for export processing  
  - Configuration: No filters, returns all channels with full metadata  
  - Inputs: Trigger output  
  - Outputs: Array of channel objects  
  - Failures: Slack API limits, auth errors

- **Loop Over Slack Channels**  
  - Type: SplitInBatches node  
  - Role: Processes channels one by one for export steps  
  - Configuration: Default batch size, sequential processing  
  - Inputs: List of channels  
  - Outputs: Single channel per iteration  
  - Failures: None expected; designed to isolate failures per channel

---

#### 2.2 Channel History Retrieval & Upload

**Overview:**  
Within the channel loop, this block retrieves the last 30 days of messages for the current channel, converts the JSON data into a file, and uploads the file to a designated Google Drive folder for archival.

**Nodes Involved:**  
- Get Slack Channel History for Export  
- Convert Channel History JSON to (file) [currently disabled]  
- Upload Channel History File to Google Drive

**Node Details:**

- **Get Slack Channel History for Export**  
  - Type: Slack node (channel history)  
  - Role: Fetches all messages from the current channel for the last 30 days  
  - Configuration:  
    - `oldest` set to now minus 1 month  
    - `latest` set to now  
    - Channel ID from current loop item  
    - Returns all messages  
  - Inputs: Current channel from loop  
  - Outputs: Raw messages JSON  
  - Failures: Slack API rate limits, channel permissions  
  - On error: “continueErrorOutput” — allows workflow to continue if one channel fails

- **Convert Channel History JSON to (file)**  
  - Type: ConvertToFile node  
  - Role: Intended to convert JSON message history into a file for upload  
  - Configuration: Default (disabled currently)  
  - Inputs: JSON messages from previous node  
  - Outputs: File object for upload  
  - Failures: None (disabled)

- **Upload Channel History File to Google Drive**  
  - Type: Google Drive node (upload file)  
  - Role: Uploads the JSON file or raw data as a file to Google Drive folder “Slack history”  
  - Configuration:  
    - File name composed of channel name + export date  
    - Folder ID preset to Slack History folder on Google Drive  
    - Drive set to “My Drive”  
  - Inputs: File object or JSON data (if ConvertToFile disabled, uploads raw JSON)  
  - Outputs: Confirmation of upload  
  - Failures: Google Drive API errors, auth token expiry, folder access issues

---

### 3. Summary Table

| Node Name                             | Node Type                        | Functional Role                      | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                              |
|-------------------------------------|---------------------------------|------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------|
| Sticky Note                         | Sticky Note                     | Workflow overview and guide        |                                  |                                  | # 🚀 Slack AI Summary & History Archiver — Quick Start Guide ...                                        |
| On Slack App Mention Trigger         | Slack Trigger                  | Listen for bot mentions            |                                  | Extract User Prompt Text from Slack Event | ## 👂 Event & Context Extraction Captures the `app_mention` event from Slack and extracts user prompt text |
| Extract User Prompt Text from Slack Event | Set                           | Extract user command text          | On Slack App Mention Trigger     | Slack Channel Summary AI Agent    | ## 👂 Event & Context Extraction Captures the `app_mention` event from Slack and extracts user prompt text |
| Slack Channel Summary AI Agent       | Langchain Agent                | AI command interpretation & summary orchestration | Extract User Prompt Text from Slack Event, LLM for Message Summaries, Channel Conversation Memory Buffer, Get Slack Channel History for Summaries, Get Slack User Profile by ID | Send Summary Message to Slack Channel | ## 🧠 AI Agent & Tooling The core logic center. The Agent uses Gemini to interpret the request ...       |
| LLM for Message Summaries            | Langchain LM Chat Model (Gemini) | Language model backend             | Slack Channel Summary AI Agent   | Slack Channel Summary AI Agent    | ## 🧠 AI Agent & Tooling The core logic center. The Agent uses Gemini to interpret the request ...       |
| Channel Conversation Memory Buffer   | Langchain Memory Buffer        | Maintains conversation context    |                                  | Slack Channel Summary AI Agent    | ## 🧠 AI Agent & Tooling The core logic center. The Agent uses Gemini to interpret the request ...       |
| Get Slack Channel History for Summaries | Slack Tool (channel history)    | Fetch messages for summarization  | Slack Channel Summary AI Agent   | Slack Channel Summary AI Agent    | ## 🧠 AI Agent & Tooling The core logic center. The Agent uses Gemini to interpret the request ...       |
| Get Slack User Profile by ID         | Slack Tool (user lookup)       | Resolve user IDs to names          | Slack Channel Summary AI Agent   | Slack Channel Summary AI Agent    | ## 🧠 AI Agent & Tooling The core logic center. The Agent uses Gemini to interpret the request ...       |
| Send Summary Message to Slack Channel | Slack                         | Post AI summary back to Slack     | Slack Channel Summary AI Agent   |                                  | ## 💬 Replay Result Replay AI summary as like slack pro.                                                |
| Monthly Export Schedule Trigger      | Schedule Trigger              | Monthly workflow start             |                                  | Get All Slack Channels in Workspace | ## 🔄 Scheduling & Iteration Initiates the workflow on a monthly schedule ...                          |
| Get All Slack Channels in Workspace  | Slack                         | Retrieve all Slack channels        | Monthly Export Schedule Trigger  | Loop Over Slack Channels          | ## 🔄 Scheduling & Iteration Initiates the workflow on a monthly schedule ...                          |
| Loop Over Slack Channels             | SplitInBatches                | Iterate over channels              | Get All Slack Channels in Workspace | Get Slack Channel History for Export | ## 🔄 Scheduling & Iteration Initiates the workflow on a monthly schedule ...                          |
| Get Slack Channel History for Export | Slack                        | Fetch last 30 days messages per channel | Loop Over Slack Channels         | Convert Channel History JSON to , Loop Over Slack Channels | ## 💾 Fetch & Archive Inside the loop, this section retrieves the last month's raw message data ...      |
| Convert Channel History JSON to      | ConvertToFile                | Convert JSON to file (disabled)   | Get Slack Channel History for Export | Upload Channel History File to Google Drive | ## 💾 Fetch & Archive Inside the loop, this section retrieves the last month's raw message data ...      |
| Upload Channel History File to Google Drive | Google Drive                | Upload channel history archive    | Convert Channel History JSON to  | Loop Over Slack Channels          | ## 💾 Fetch & Archive Inside the loop, this section retrieves the last month's raw message data ...      |
| Sticky Note1                        | Sticky Note                   | Event & Context Extraction summary |                                  |                                  | ## 👂 Event & Context Extraction Captures the `app_mention` event from Slack and extracts the user's clean text prompt |
| Sticky Note2                        | Sticky Note                   | Scheduling & Iteration summary    |                                  |                                  | ## 🔄 Scheduling & Iteration Initiates the workflow on a monthly schedule, retrieves channels, loops them |
| Sticky Note3                        | Sticky Note                   | AI Agent & Tooling summary        |                                  |                                  | ## 🧠 AI Agent & Tooling The core logic center. The Agent uses Gemini to interpret the request, calls Slack tools, and maintains memory. |
| Sticky Note4                        | Sticky Note                   | Fetch & Archive summary            |                                  |                                  | ## 💾 Fetch & Archive Inside the loop, this section retrieves last month's raw message data for current channel, converts and uploads it |
| Sticky Note5                        | Sticky Note                   | AI summary replay                  |                                  |                                  | ## 💬 Replay Result Replay AI summary as like slack pro.                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Credentials**  
   - Setup Slack API credentials with permissions: read channels, read channel history, post messages, user info access.  
   - Ensure the bot is invited to target channels.

2. **Create Google Drive Credentials**  
   - Setup OAuth2 credentials for Google Drive.  
   - Prepare or create a folder for Slack archives and note its folder ID.

3. **Create AI Credentials**  
   - Setup Google Gemini API credentials (or other LLM provider).  
   - Ensure API key has sufficient quota.

4. **Create Nodes for AI Summarization Flow**

   4.1. **Slack Trigger Node**  
   - Type: Slack Trigger  
   - Trigger on event: `app_mention`  
   - Watch entire workspace  
   - Attach Slack credentials

   4.2. **Set Node: Extract User Prompt Text**  
   - Extract user message text from Slack event JSON: expression `{{$json.blocks[0].elements[0].elements[1].text}}`  
   - Output field: `input`

   4.3. **Langchain Agent Node: Slack Channel Summary AI Agent**  
   - Input text bound to `input` field from previous node  
   - System prompt provided to guide AI on parsing time ranges, calling Slack tools for history and user info, formatting summaries  
   - Configure AI tools:  
     - Slack channel history tool (see 4.4)  
     - Slack user info lookup tool (see 4.5)  
   - Attach Google Gemini credentials for LLM calls  
   - Configure conversation memory: window length 10, session key = Slack channel ID

   4.4. **Slack Tool Node: Get Slack Channel History for Summaries**  
   - Operation: channel history  
   - Channel ID: dynamic from Slack trigger’s channel  
   - Filters: `oldest` and `latest` set dynamically per AI agent’s parsing  
   - Return all messages

   4.5. **Slack Tool Node: Get Slack User Profile by ID**  
   - Operation: user info  
   - User ID: passed dynamically from AI agent to resolve user mentions

   4.6. **Langchain LM Chat Model Node: LLM for Message Summaries**  
   - Use Google Gemini node  
   - Connect as AI language model to Slack Channel Summary AI Agent

   4.7. **Langchain Memory Buffer Node: Channel Conversation Memory Buffer**  
   - Session key type: customKey  
   - Session key: Slack channel ID  
   - Context window length: 10  
   - Connect as AI memory for Slack Channel Summary AI Agent

   4.8. **Slack Node: Send Summary Message to Slack Channel**  
   - Operation: post message  
   - Text: output from Slack Channel Summary AI Agent  
   - Channel ID: Slack trigger channel ID  
   - Markdown enabled  
   - Attach Slack credentials

5. **Create Nodes for Monthly Archiving Flow**

   5.1. **Schedule Trigger Node: Monthly Export Schedule Trigger**  
   - Interval: monthly  
   - Trigger hour: 1 AM

   5.2. **Slack Node: Get All Slack Channels in Workspace**  
   - Operation: get all channels  
   - No filters  
   - Attach Slack credentials

   5.3. **SplitInBatches Node: Loop Over Slack Channels**  
   - Use default batch size (process one channel at a time)

   5.4. **Slack Node: Get Slack Channel History for Export**  
   - Operation: channel history  
   - Channel ID: from current batch item  
   - Filters:  
     - `oldest`: now minus 1 month (format `yyyy-MM-dd HH:mm:ss`)  
     - `latest`: now (format `yyyy-MM-dd HH:mm:ss`)  
   - Return all messages  
   - On error: continue workflow to next channel

   5.5. **ConvertToFile Node: Convert Channel History JSON to File**  
   - Convert JSON message list to a file (e.g., JSON or text file)  
   - (Optional: this node is disabled; if used enable and configure file format)

   5.6. **Google Drive Node: Upload Channel History File to Google Drive**  
   - File name: `${channel_name}_${export_date}`  
   - Folder: Slack History folder ID  
   - Drive: My Drive  
   - Input: file object from Convert node or raw JSON if convert node disabled  
   - Attach Google Drive credentials

   5.7. **Connect Upload node output to Loop Over Slack Channels**  
   - Enables processing next channel batch

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| This workflow simulates Slack Pro features using open AI summarization and Google Drive archiving.                                                                           | Workflow purpose                                                |
| System prompt example in AI Agent node is detailed and carefully crafted to parse relative times and summarize channel messages by topic blocks with emojis for readability | See Slack Channel Summary AI Agent systemMessage parameter     |
| Google Drive folder ID for archives must be pre-created and accessible by configured credentials.                                                                            | Google Drive node folderId parameter                            |
| Slack bot must be invited to all channels to be summarized or archived for API calls to succeed.                                                                             | Slack app installation requirement                              |
| Slack triggers use webhook with ID; ensure the webhook configuration in Slack app matches n8n webhook URL.                                                                  | Slack webhook setup                                            |
| For best results, the AI model used is Google Gemini; can be replaced with another LLM node with similar API.                                                                | LLM for Message Summaries node                                 |
| The monthly export schedule runs once a month at 1 AM server time; adjust as needed in Schedule Trigger node.                                                                | Monthly Export Schedule Trigger node                           |
| If exporting large workspaces, consider API rate limits and batch sizes; SplitInBatches node helps mitigate timeouts.                                                        | Loop Over Slack Channels node                                  |
| The ConvertToFile node is currently disabled; enabling it allows better file handling for uploads instead of raw JSON.                                                      | Convert Channel History JSON to node                           |
| Slack user ID resolution prefers display_name or real_name; falls back to Slack mention syntax if unavailable.                                                              | Get Slack User Profile by ID node                             |

---

This reference document fully describes the workflow, enabling expert users or AI agents to understand, reproduce, and maintain the Slack AI summarization and archiving solution.