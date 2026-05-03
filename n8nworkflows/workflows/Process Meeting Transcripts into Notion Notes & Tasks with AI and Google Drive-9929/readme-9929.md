Process Meeting Transcripts into Notion Notes & Tasks with AI and Google Drive

https://n8nworkflows.xyz/workflows/process-meeting-transcripts-into-notion-notes---tasks-with-ai-and-google-drive-9929


# Process Meeting Transcripts into Notion Notes & Tasks with AI and Google Drive

### 1. Workflow Overview

This workflow automates the processing of meeting transcripts into structured notes and task items within Notion, leveraging AI for intelligent summarization and classification. It integrates meeting data from external sources (Google Drive and Notion), processes transcripts with AI models, and organizes the output into categorized notes and actionable tasks in Notion.

Logical blocks include:

- **1.1 Input Reception and Triggering**: Detects new meetings via webhook or scheduled checks.
- **1.2 Transcript Retrieval and Preparation**: Fetches and ensures availability of meeting transcripts, flattens data structures for processing.
- **1.3 AI-Powered Meeting Note Generation**: Uses LangChain agents and AI models (Anthropic, OpenAI) to generate structured meeting notes.
- **1.4 Notion Integration for Notes and Tasks**: Adds curated notes and extracted tasks into Notion databases.
- **1.5 Task Filtering and Assignment**: Filters tasks assigned to the user and adds them to a specific Notion tasks database.
- **1.6 Google Drive File Creation and Linking**: Creates files for transcripts in Google Drive and links them back into Notion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Triggering

- **Overview:** Captures new meeting events either through webhook or scheduled polling, initiating the workflow.
- **Nodes Involved:** `New Meeting Webhook`, `Schedule Trigger`, `Get Meetings from Notion`, `List Meetings`, `Get New Meetings`
  
- **Node Details:**

  - **New Meeting Webhook**
    - Type: Webhook (Entry point)
    - Role: Receives incoming meeting data to start processing immediately.
    - Configuration: Uses a unique webhook ID; no additional parameters.
    - Inputs: External HTTP calls.
    - Outputs: Flattens transcript data next.
    - Potential failures: Webhook timeout or malformed requests.

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Periodically triggers workflow to check for new meetings.
    - Configuration: Disabled by default; can be enabled with custom schedule.
    - Outputs: Fetches meetings from Notion.
    - Failures: Scheduler misconfiguration or service downtime.

  - **Get Meetings from Notion**
    - Type: Notion node
    - Role: Retrieves existing meeting records to compare and find new ones.
    - Disabled by default.
    - Outputs: Passed to `List Meetings`.

  - **List Meetings**
    - Type: HTTP Request (disabled)
    - Role: Intended for listing meetings from an external API.
    - Disabled.

  - **Get New Meetings**
    - Type: Code node (disabled)
    - Role: Compares meetings from previous node to identify new ones.
    - Disabled.

---

#### 1.2 Transcript Retrieval and Preparation

- **Overview:** Ensures the meeting transcript is ready for processing; flattens nested data structures for easier AI consumption.
- **Nodes Involved:** `Get Transcript`, `Transcript Ready?`, `Flatten`, `Wait`, `Flatten Transcript`, `Set Title + Transcript + URL`
  
- **Node Details:**

  - **Get Transcript**
    - Type: HTTP Request (disabled)
    - Role: Fetches transcript data from an external source.
    - Config: Disabled; may include API endpoint and authentication.
    - Continues on failure to avoid blocking workflow.

  - **Transcript Ready?**
    - Type: If node (disabled)
    - Role: Checks if the transcript is fully available.
    - Routes to either flatten data or wait/retry.

  - **Wait**
    - Type: Wait node (disabled)
    - Role: Pauses workflow before retrying transcript fetch.

  - **Flatten**
    - Type: Code node (disabled)
    - Role: Flattens complex transcript JSON into simpler format.

  - **Flatten Transcript**
    - Type: Code node
    - Role: Active flattening of transcript data received from webhook.
    - Outputs: Passes to setting title, transcript, and URL.

  - **Set Title + Transcript + URL**
    - Type: Set node
    - Role: Sets key fields (meeting title, full transcript text, and URL) for downstream processing.
    - Inputs: Flattened transcript data.
    - Outputs: Passes to AI categorization.

  - **Potential Failures:** Transcript not ready, malformed JSON, timeout on external API calls.

---

#### 1.3 AI-Powered Meeting Note Generation

- **Overview:** Uses AI models and LangChain agents to summarize and categorize meeting content, generating structured notes.
- **Nodes Involved:** `Categorize Meeting`, `Switch`, `Anthropic Chat Model`, `Misc Meeting Notetaker`, `Discovery Call Notetaker`, `Structured Output Parser`, `Structured Output Parser1`

- **Node Details:**

  - **Categorize Meeting**
    - Type: OpenAI node via LangChain integration
    - Role: Classifies meeting type for routing.
    - Configuration: OpenAI API credentials required.
    - Outputs: Routes to `Switch`.

  - **Switch**
    - Type: Switch node
    - Role: Routes flow based on meeting category to one of two LangChain agents.
    - Outputs: To either `Misc Meeting Notetaker` or `Discovery Call Notetaker`.

  - **Anthropic Chat Model**
    - Type: Anthropic chat model node via LangChain
    - Role: Provides AI-generated text for agents.
    - Shared resource feeding both agents.
    - Requires Anthropic API credentials.

  - **Misc Meeting Notetaker**
    - Type: LangChain agent node
    - Role: Processes general meetings, generates notes.
    - Input: AI language model and structured output parser.
    - Outputs: Adds notes to Notion.

  - **Discovery Call Notetaker**
    - Type: LangChain agent node
    - Role: Processes discovery call meetings with specialized prompts.
    - Input: AI model and structured output parser.
    - Outputs: Adds notes to Notion.

  - **Structured Output Parser & Structured Output Parser1**
    - Type: LangChain structured output parsers
    - Role: Parse AI responses into structured JSON for agents.
    - Each tied respectively to Discovery Call and Misc Meeting Notetaker.

  - **Potential Failures:** AI API rate limits, malformed AI responses, misclassification causing wrong routing.

---

#### 1.4 Notion Integration for Notes and Tasks

- **Overview:** Inserts processed notes and tasks into Notion databases, maintaining meeting records and task lists.
- **Nodes Involved:** `Add Meeting Notes to Notion`, `Add Discovery Meeting Notes to Notion`, `Set Notion Page ID`, `Link Transcript in Notion`, `Split out Tasks`, `Add Tasks`

- **Node Details:**

  - **Add Meeting Notes to Notion**
    - Type: Notion node
    - Role: Creates or updates meeting notes page.
    - Inputs: Output from `Misc Meeting Notetaker`.
    - Credentials: Notion integration required.

  - **Add Discovery Meeting Notes to Notion**
    - Type: Notion node
    - Role: Creates or updates discovery call notes page.
    - Inputs: Output from `Discovery Call Notetaker`.

  - **Set Notion Page ID**
    - Type: Set node
    - Role: Stores page ID for linking files/tasks.
    - Inputs: From notes nodes.
    - Outputs: Triggers Google Drive file creation.

  - **Link Transcript in Notion**
    - Type: Notion node
    - Role: Adds Google Drive transcript file link back into Notion page.
    - Inputs: File creation data.

  - **Split out Tasks**
    - Type: Code node
    - Role: Extracts individual tasks from AI-generated notes.
    - Outputs: Passes tasks for filtering and adding.

  - **Add Tasks**
    - Type: Notion node
    - Role: Inserts tasks into a Notion tasks database.
    - Inputs: Filtered tasks assigned to the user.

  - **Potential Failures:** Notion API rate limits, invalid page IDs, credential expiration.

---

#### 1.5 Task Filtering and Assignment

- **Overview:** Filters extracted tasks to those assigned to the current user before adding them to Notion.
- **Nodes Involved:** `If Assigned to Me`

- **Node Details:**

  - **If Assigned to Me**
    - Type: If node
    - Role: Checks if task assignee matches current user.
    - Inputs: Extracted tasks.
    - Outputs: Passes matching tasks to `Add Tasks`.
    - Failures: Incorrect user ID, empty assignee fields.

---

#### 1.6 Google Drive File Creation and Linking

- **Overview:** Creates transcript files in Google Drive and links them to Notion meeting pages.
- **Nodes Involved:** `Create File`

- **Node Details:**

  - **Create File**
    - Type: Google Drive node
    - Role: Creates a file with the transcript content.
    - Inputs: Page ID and transcript data.
    - Outputs: Passes file metadata to Notion link node.
    - Credentials: Google Drive OAuth2 required.
    - Failures: Drive quota exceeded, auth errors.

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                              | Input Node(s)                 | Output Node(s)                      | Sticky Note                                     |
|-------------------------------|---------------------------------------|----------------------------------------------|------------------------------|------------------------------------|------------------------------------------------|
| New Meeting Webhook            | Webhook                               | Entry point for new meetings                  | External HTTP                | Flatten Transcript                  |                                                |
| Flatten Transcript             | Code                                  | Flatten input transcript data                  | New Meeting Webhook          | Set Title + Transcript + URL       |                                                |
| Set Title + Transcript + URL  | Set                                   | Prepare key transcript metadata                | Flatten Transcript           | Categorize Meeting                 |                                                |
| Categorize Meeting            | OpenAI (LangChain)                    | Classify meeting type                           | Set Title + Transcript + URL | Switch                           |                                                |
| Switch                       | Switch                                | Route based on meeting category                | Categorize Meeting           | Misc Meeting Notetaker, Discovery Call Notetaker |                                                |
| Anthropic Chat Model          | LangChain AI Model                    | AI model providing chat completions            | -                           | Misc Meeting Notetaker, Discovery Call Notetaker |                                                |
| Misc Meeting Notetaker        | LangChain Agent                      | Generate notes for general meetings            | Anthropic Chat Model, Structured Output Parser1 | Add Meeting Notes to Notion          |                                                |
| Discovery Call Notetaker      | LangChain Agent                      | Generate notes for discovery calls             | Anthropic Chat Model, Structured Output Parser | Add Discovery Meeting Notes to Notion |                                                |
| Structured Output Parser      | LangChain Output Parser               | Parse AI response for discovery calls          | -                           | Discovery Call Notetaker          |                                                |
| Structured Output Parser1     | LangChain Output Parser               | Parse AI response for misc meetings             | -                           | Misc Meeting Notetaker            |                                                |
| Add Meeting Notes to Notion   | Notion                               | Insert meeting notes                            | Misc Meeting Notetaker       | Set Notion Page ID                |                                                |
| Add Discovery Meeting Notes to Notion | Notion                       | Insert discovery call notes                      | Discovery Call Notetaker     | Set Notion Page ID                |                                                |
| Set Notion Page ID            | Set                                   | Store Notion page ID for linking files/tasks   | Add Meeting Notes to Notion, Add Discovery Meeting Notes to Notion | Create File                      |                                                |
| Create File                  | Google Drive                         | Create transcript file in Google Drive          | Set Notion Page ID           | Link Transcript in Notion         |                                                |
| Link Transcript in Notion     | Notion                               | Link created file URL in Notion page            | Create File                  | Split out Tasks                   |                                                |
| Split out Tasks              | Code                                  | Extract tasks from notes                         | Link Transcript in Notion    | If Assigned to Me                 |                                                |
| If Assigned to Me             | If                                   | Filter tasks assigned to current user           | Split out Tasks              | Add Tasks                        |                                                |
| Add Tasks                   | Notion                               | Add filtered tasks to Notion tasks database     | If Assigned to Me            | -                                |                                                |
| Schedule Trigger             | Schedule Trigger                     | Periodic trigger to poll meetings                | -                           | Get Meetings from Notion          | Disabled by default                             |
| Get Meetings from Notion      | Notion                               | Retrieve meetings for comparison                  | Schedule Trigger             | List Meetings                    | Disabled by default                             |
| List Meetings                | HTTP Request                        | List meetings from external source              | Get Meetings from Notion     | Get New Meetings                 | Disabled by default                             |
| Get New Meetings             | Code                                  | Identify new meetings compared to existing       | List Meetings                | Get Transcript                  | Disabled by default                             |
| Get Transcript              | HTTP Request                        | Fetch transcript data                             | Get New Meetings             | Transcript Ready?                | Disabled by default                             |
| Transcript Ready?            | If                                   | Check transcript availability                     | Get Transcript               | Flatten, Wait                   | Disabled by default                             |
| Wait                         | Wait                                 | Pause for retry when transcript not ready        | Transcript Ready?            | Get Transcript                 | Disabled by default                             |
| Flatten                      | Code                                  | Flatten transcript structure                        | Transcript Ready?            | -                              | Disabled by default                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "New Meeting Webhook"**
   - Type: Webhook
   - Purpose: Entry point for new meeting data.
   - Configure unique webhook ID.
   - No authentication needed.
   - Connect output to next node.

2. **Create Code Node: "Flatten Transcript"**
   - Type: Code
   - Purpose: Flatten nested transcript JSON into simple fields.
   - Input: Webhook data.
   - Output: Passes to Set node.

3. **Create Set Node: "Set Title + Transcript + URL"**
   - Type: Set
   - Purpose: Extract and assign key metadata fields (title, transcript text, meeting URL).
   - Use expressions to map flattened fields.
   - Connect output to AI categorization.

4. **Create OpenAI LangChain Node: "Categorize Meeting"**
   - Type: LangChain OpenAI
   - Purpose: Classify meeting type (e.g., discovery call vs. general).
   - Configure with OpenAI API credentials.
   - Connect output to Switch node.

5. **Create Switch Node: "Switch"**
   - Type: Switch
   - Purpose: Route flow based on meeting category result.
   - Define cases for different meeting types (e.g., "Discovery Call", "Miscellaneous").
   - Connect outputs to respective LangChain agent nodes.

6. **Create LangChain AI Node: "Anthropic Chat Model"**
   - Type: LangChain AI Model (Anthropic)
   - Purpose: Provide chat completions to agents.
   - Configure Anthropic API credentials.
   - Connect as AI model input for both agent nodes.

7. **Create LangChain Agent Node: "Misc Meeting Notetaker"**
   - Type: LangChain Agent
   - Purpose: Generate meeting notes for general meetings.
   - Use Anthropic model and structured output parser.
   - Connect output to Notion notes node.

8. **Create LangChain Agent Node: "Discovery Call Notetaker"**
   - Type: LangChain Agent
   - Purpose: Generate notes specialized for discovery calls.
   - Use Anthropic model and structured output parser.
   - Connect output to Notion notes node.

9. **Create LangChain Structured Output Parser Nodes**
   - "Structured Output Parser" for Discovery Call Notetaker
   - "Structured Output Parser1" for Misc Meeting Notetaker
   - Configure parsers to convert AI text to structured JSON.

10. **Create Notion Node: "Add Meeting Notes to Notion"**
    - Type: Notion
    - Purpose: Add notes for general meetings.
    - Configure Notion credentials and target database/page.
    - Connect input from Misc Meeting Notetaker.

11. **Create Notion Node: "Add Discovery Meeting Notes to Notion"**
    - Type: Notion
    - Purpose: Add notes for discovery calls.
    - Configure Notion credentials.
    - Connect input from Discovery Call Notetaker.

12. **Create Set Node: "Set Notion Page ID"**
    - Type: Set
    - Purpose: Store the Notion page ID for linking files/tasks.
    - Connect input from both Add Meeting Notes nodes.
    - Connect output to Google Drive node.

13. **Create Google Drive Node: "Create File"**
    - Type: Google Drive
    - Purpose: Upload transcript as a file.
    - Configure with Google Drive OAuth2 credentials.
    - Use Notion page ID and transcript content.
    - Connect output to Notion linking node.

14. **Create Notion Node: "Link Transcript in Notion"**
    - Type: Notion
    - Purpose: Add link to transcript file inside Notion page.
    - Connect input from Google Drive Create File node.
    - Connect output to task extraction.

15. **Create Code Node: "Split out Tasks"**
    - Type: Code
    - Purpose: Extract individual tasks from notes JSON.
    - Connect input from Link Transcript in Notion.
    - Connect output to filtering node.

16. **Create If Node: "If Assigned to Me"**
    - Type: If
    - Purpose: Filter tasks by current user assignment.
    - Configure condition to match current user ID.
    - Connect output to Add Tasks node.

17. **Create Notion Node: "Add Tasks"**
    - Type: Notion
    - Purpose: Insert user-assigned tasks into Notion tasks database.
    - Configure credentials and database.
    - Connect input from If node.

18. *(Optional)* Enable and configure Schedule Trigger, Get Meetings from Notion, List Meetings, Get New Meetings, Get Transcript, Transcript Ready?, Wait, and Flatten nodes for periodic polling alternative to webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                             |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| Workflow designed to automate meeting transcript processing into Notion notes and actionable tasks | Workflow Description                                                                       |
| Uses LangChain AI integration with Anthropic and OpenAI models                                     | n8n LangChain nodes documentation                                                         |
| Requires valid API credentials for Notion, Google Drive, OpenAI, and Anthropic                      | Credential Setup in n8n                                                                    |
| Disabled nodes provide alternate or legacy methods for meeting retrieval and transcript polling    | Nodes disabled for webhook-based triggering                                               |
| For more on LangChain agents and structured output parsing, see: https://docs.n8n.io/nodes/ai/    | Official n8n AI nodes documentation                                                       |

---

This structured document covers all aspects of the workflow "Transcript -> Notion -> Tasks," enabling understanding, modification, and reimplementation without requiring the original JSON.