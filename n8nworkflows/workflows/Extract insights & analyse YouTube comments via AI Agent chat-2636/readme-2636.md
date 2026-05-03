Extract insights & analyse YouTube comments via AI Agent chat

https://n8nworkflows.xyz/workflows/extract-insights---analyse-youtube-comments-via-ai-agent-chat-2636


# Extract insights & analyse YouTube comments via AI Agent chat

### 1. Workflow Overview

This workflow automates the extraction and analysis of insights from YouTube videos and comments using AI agents. It is designed primarily for content creators, marketers, and analysts who want to understand audience preferences and optimize video content through data-driven insights. The workflow integrates YouTube Data API calls, AI-powered transcription, comment processing, and thumbnail analysis.

Logical blocks:

- **1.1 Input Reception and AI Agent Activation:** Handles chat message reception and routes user queries to the AI agent.
- **1.2 Command Processing and Routing:** Switch node that directs commands to appropriate API or AI tool workflows.
- **1.3 YouTube Data Retrieval:** Nodes that fetch channel details, video descriptions, video lists, search results, and comments from YouTube API.
- **1.4 Data Transformation and Response Preparation:** Processes raw API data (e.g., formats comments) and prepares responses.
- **1.5 AI Processing Blocks:** Includes OpenAI language model nodes and AI agent handling for transcription, thumbnail analysis, and multi-step reasoning.
- **1.6 External API Integration:** Uses Apify for video transcription and OpenAI for thumbnail image analysis.
- **1.7 Memory Management:** Postgres-based chat memory storage for session persistence.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and AI Agent Activation

- **Overview:** This block receives chat messages from users and passes them to the AI agent for processing.
- **Nodes Involved:** When chat message received, AI Agent, Postgres Chat Memory, OpenAI Chat Model
- **Node Details:**

  - **When chat message received**  
    - Type: Chat trigger node  
    - Role: Receives user chat input via webhook (Webhook ID configured)  
    - Input/Output: Incoming chat message → AI Agent  
    - Failure types: Webhook misconfiguration, network issues  
    - Version: 1.1

  - **AI Agent**  
    - Type: Langchain agent node  
    - Role: Processes user input, plans tool usage, and manages execution order  
    - Configuration: Uses "openAiFunctionsAgent" with system message guiding YouTube assistant behavior (e.g., filtering shorts, querying tools)  
    - Key expressions: Text input from chat trigger `={{ $('When chat message received').item.json.chatInput }}`  
    - Input/Output: Input from chat trigger → executes relevant tools → outputs to Postgres Chat Memory and OpenAI Chat Model  
    - Failure types: API auth errors, expression errors, tool execution failures  
    - Version: 1.6

  - **Postgres Chat Memory**  
    - Type: Postgres memory node for chat  
    - Role: Stores chat session memory keyed by session ID from chat input  
    - Configuration: Uses custom session key `={{ $('When chat message received').item.json.sessionId }}`  
    - Credential: Postgres database connection  
    - Input/Output: Memory interface for AI Agent  
    - Failure types: DB connectivity, session key issues  
    - Version: 1.3

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI chat model node  
    - Role: Language model for generating responses within AI Agent workflow  
    - Credential: OpenAI API key  
    - Input/Output: Receives prompts from AI Agent, outputs text responses  
    - Failure types: API quota, network errors  
    - Version: 1

---

#### 1.2 Command Processing and Routing

- **Overview:** This block routes incoming commands from the workflow trigger to the appropriate YouTube or AI processing nodes.
- **Nodes Involved:** Execute Workflow Trigger, Switch
- **Node Details:**

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger node  
    - Role: Receives API trigger with commands and parameters (e.g., search query, channel ID)  
    - Input/Output: Input trigger → outputs JSON with command and query parameters → Switch node  
    - Failure types: Trigger misconfiguration, invalid input format  
    - Version: 1

  - **Switch**  
    - Type: Switch node  
    - Role: Routes based on `command` field in trigger JSON to nodes such as get_channel_details, video_details, comments, search, videos, analyze_thumbnail, video_transcription  
    - Conditions: String equals checks on `command` property  
    - Input/Output: Input from Execute Workflow Trigger → routes to respective nodes  
    - Failure types: Missing/unknown commands, expression evaluation errors  
    - Version: 3.2

---

#### 1.3 YouTube Data Retrieval

- **Overview:** This block fetches YouTube data such as channel details, video descriptions, video lists, comments, and search results through HTTP requests using YouTube Data API.
- **Nodes Involved:** Get Channel Details, Get Video Description, Get Videos by Channel, Get Comments, Run Query
- **Node Details:**

  - **Get Channel Details**  
    - Type: HTTP Request  
    - Role: Fetches channel snippet details by handle/username via YouTube API `channels` endpoint  
    - Query params: `part=snippet`, `forHandle={{handle}}`  
    - Credential: Google API key via HTTP Query Auth  
    - Input/Output: Input handle from Execute Workflow Trigger → outputs channel info JSON → Response node  
    - Failure types: API quota, invalid handle, auth errors  
    - Version: 4.2

  - **Get Video Description**  
    - Type: HTTP Request  
    - Role: Retrieves video snippet, contentDetails, and statistics by video ID  
    - Query params: `part=snippet,contentDetails,statistics`, `id={{video_id}}`  
    - Credential: Google API key  
    - Input/Output: Input video_id → outputs video details JSON → Response node  
    - Failure types: API quota, invalid video ID  
    - Version: 4.2

  - **Get Videos by Channel**  
    - Type: HTTP Request  
    - Role: Lists videos for a given channel ID sorted by order (date, viewCount, relevance)  
    - Query params: `part=snippet`, `channelId={{channel_id}}`, `order`, `maxResults`, `type=video`, `publishedAfter`  
    - Credential: Google API key  
    - Input/Output: Input channel_id and params → outputs list of videos → Response node  
    - Failure types: API quota, invalid channel ID  
    - Version: 4.2

  - **Get Comments**  
    - Type: HTTP Request  
    - Role: Retrieves up to 100 comments for a video using the `commentThreads` endpoint  
    - Query params: `part=id,snippet,replies`, `videoId={{video_id}}`, `maxResults=100`  
    - Credential: Google API key  
    - Input/Output: Input video_id → outputs comments JSON → Edit Fields node  
    - Failure types: API quota, disabled comments, invalid video ID  
    - Version: 4.2

  - **Run Query**  
    - Type: HTTP Request  
    - Role: Executes YouTube search queries for videos or channels with specified filters and sorting  
    - Query params: `part=snippet`, `q`, `order`, `type`, `maxResults`, `publishedAfter`  
    - Credential: Google API key  
    - Input/Output: Input search parameters → outputs search results → Response node  
    - Failure types: API quota, invalid query parameters  
    - Version: 4.2

---

#### 1.4 Data Transformation and Response Preparation

- **Overview:** This block formats raw data (e.g., comments) into readable text and prepares the final response object.
- **Nodes Involved:** Edit Fields, Response
- **Node Details:**

  - **Edit Fields**  
    - Type: Set node  
    - Role: Converts raw comment thread JSON into a text string listing top-level comments and replies formatted by author and content  
    - Expression example: Maps over `$json.items` to create concatenated string of comments and replies  
    - Input/Output: Input raw comments JSON → outputs stringified, human-readable comment list → Response node  
    - Failure types: Expression errors if data shape unexpected  
    - Version: 3.4

  - **Response**  
    - Type: Set node  
    - Role: Passes the final response object (e.g., comment string, video details) downstream or to webhook response  
    - Input/Output: Input JSON or string → output JSON response  
    - Failure types: None specific  
    - Version: 3.4

---

#### 1.5 AI Processing Blocks

- **Overview:** These nodes use AI models and agents to analyze video transcriptions, thumbnails, and manage intelligent multi-step workflows.
- **Nodes Involved:** OpenAI, OpenAI Chat Model, AI Agent (also overlaps with 1.1), get_channel_details, get_video_description, get_list_of_videos, get_list_of_comments, search, analyze_thumbnail, video_transcription
- **Node Details:**

  - **OpenAI**  
    - Type: Langchain OpenAI node  
    - Role: Analyzes images (thumbnails) using AI based on prompt and image URL  
    - Parameters: Text prompt and image URLs from input  
    - Credential: OpenAI API key  
    - Input/Output: Input prompt and URL → outputs analysis text  
    - Failure types: API limits, invalid URLs, network errors  
    - Version: 1.7

  - **AI Agent**  
    - Previously described in 1.1; here it also acts as orchestrator running tool workflows on demand.

  - **Tool Workflow Invocations (get_channel_details, get_video_description, get_list_of_videos, get_list_of_comments, search, analyze_thumbnail, video_transcription)**  
    - Type: Langchain toolWorkflow nodes  
    - Role: Encapsulate reusable workflows for specific YouTube or AI tasks  
    - Configuration: Each defines a command, input schema, and description for use by AI agent  
    - Input/Output: Receive specific inputs (e.g., channel handle, video ID) and output structured data  
    - Failure types: Workflow-specific errors, invalid input, API errors  
    - Version: 1.2+

---

#### 1.6 External API Integration

- **Overview:** Manages calls to third-party services for transcription and thumbnail analysis.
- **Nodes Involved:** Get Video Transcription, OpenAI
- **Node Details:**

  - **Get Video Transcription**  
    - Type: HTTP Request  
    - Role: Calls Apify actor to transcribe video content from URL  
    - Method: POST with JSON body containing `startUrls` array with video URL  
    - Credential: Apify API key via HTTP Query Auth  
    - Input/Output: Input video URL → outputs transcription text → Response node  
    - Failure types: API quota, transcription delays, invalid URL  
    - Version: 4.2

  - **OpenAI**  
    - See above in 1.5 for thumbnail image analysis

---

#### 1.7 Memory Management

- **Overview:** Stores chat session data persistently for context-aware AI agent interactions.
- **Nodes Involved:** Postgres Chat Memory
- **Node Details:** See 1.1

---

### 3. Summary Table

| Node Name                | Node Type                                 | Functional Role                              | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                         |
|--------------------------|-------------------------------------------|----------------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received| chatTrigger                               | Receives user chat input                      |                             | AI Agent                   |                                                                                                   |
| AI Agent                 | langchain.agent                           | Processes chat input, orchestrates tools     | When chat message received   | Postgres Chat Memory, OpenAI Chat Model |                                                                                                   |
| Postgres Chat Memory      | langchain.memoryPostgresChat               | Stores chat session memory                     | AI Agent                    | AI Agent                   |                                                                                                   |
| OpenAI Chat Model        | langchain.lmChatOpenAi                    | AI language model for chat responses          | AI Agent                    | AI Agent                   |                                                                                                   |
| Execute Workflow Trigger | executeWorkflowTrigger                    | Receives external API trigger commands        |                             | Switch                     |                                                                                                   |
| Switch                   | switch                                   | Routes commands to corresponding workflow nodes | Execute Workflow Trigger     | Multiple API or AI nodes    |                                                                                                   |
| Get Channel Details       | httpRequest                              | Fetches YouTube channel details by handle     | Switch                      | Response                   |                                                                                                   |
| Get Video Description     | httpRequest                              | Fetches video details by video ID              | Switch                      | Response                   |                                                                                                   |
| Get Videos by Channel     | httpRequest                              | Lists videos from a channel                     | Switch                      | Response                   |                                                                                                   |
| Get Comments             | httpRequest                              | Retrieves comments for a video                  | Switch                      | Edit Fields                |                                                                                                   |
| Run Query                | httpRequest                              | Executes YouTube search queries                 | Switch                      | Response                   |                                                                                                   |
| Edit Fields              | set                                      | Formats raw comments JSON to readable string  | Get Comments                | Response                   |                                                                                                   |
| Response                 | set                                      | Prepares final response JSON                    | Multiple nodes              |                            |                                                                                                   |
| OpenAI                   | langchain.openAi                         | AI analysis of thumbnails                       | Switch                      | Response                   |                                                                                                   |
| Get Video Transcription   | httpRequest                              | Calls Apify actor to transcribe video          | Switch                      | Response                   |                                                                                                   |
| get_channel_details       | langchain.toolWorkflow                   | Tool workflow: get channel details             | AI Agent                    | AI Agent                   |                                                                                                   |
| get_video_description     | langchain.toolWorkflow                   | Tool workflow: get video details                | AI Agent                    | AI Agent                   |                                                                                                   |
| get_list_of_videos        | langchain.toolWorkflow                   | Tool workflow: list videos from channel        | AI Agent                    | AI Agent                   |                                                                                                   |
| get_list_of_comments      | langchain.toolWorkflow                   | Tool workflow: list comments for video          | AI Agent                    | AI Agent                   |                                                                                                   |
| search                   | langchain.toolWorkflow                   | Tool workflow: search videos or channels        | AI Agent                    | AI Agent                   |                                                                                                   |
| analyze_thumbnail        | langchain.toolWorkflow                   | Tool workflow: analyze thumbnail images         | AI Agent                    | AI Agent                   |                                                                                                   |
| video_transcription       | langchain.toolWorkflow                   | Tool workflow: transcribe video                  | AI Agent                    | AI Agent                   |                                                                                                   |
| Sticky Note7             | stickyNote                              | Project branding, overview, and purpose         |                             |                            | Contains workflow purpose and use case details                                                    |
| Sticky Note6             | stickyNote                              | Detailed setup instructions                      |                             |                            | Contains instructions for API setup, video selection, comment analysis, transcription, thumbnail analysis |
| Sticky Note9             | stickyNote                              | Link to video guide                              |                             |                            | Contains YouTube video guide and clickable thumbnail image                                       |
| Sticky Note              | stickyNote                              | Label for Scenario 1 AI agent                    |                             |                            |                                                                                                   |
| Sticky Note1             | stickyNote                              | Label for Scenario 2 agent tools                  |                             |                            |                                                                                                   |
| Sticky Note2             | stickyNote                              | Reminder to replace credentials                   |                             |                            |                                                                                                   |
| Sticky Note3             | stickyNote                              | Reminder to replace credentials in all nodes     |                             |                            |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Google Cloud API Key with YouTube Data API enabled.
   - OpenAI API key.
   - Apify API key.
   - Postgres credentials for chat memory.

2. **Add Webhook Trigger Node:**
   - Create **When chat message received** node (chatTrigger) with webhook ID.
   - This node receives chat input with fields: `chatInput`, `sessionId`.

3. **Add Postgres Chat Memory Node:**
   - Create **Postgres Chat Memory** node.
   - Configure with Postgres credentials.
   - Set session key: `={{ $('When chat message received').item.json.sessionId }}`.

4. **Add OpenAI Chat Model Node:**
   - Create **OpenAI Chat Model** node.
   - Configure with OpenAI API credentials.

5. **Add AI Agent Node:**
   - Create **AI Agent** node.
   - Set `text` input to `={{ $('When chat message received').item.json.chatInput }}`.
   - Select agent type `openAiFunctionsAgent`.
   - Add system message:
     ```
     You are Youtube assistant. 
     You need to process user's requests and run relevant tools for that. 
     Plan and execute in right order runs of tools to get data for user's request.
     IMPORTANT Search query and list of videos for channel tools returns all videos including shorts - use Get Video description tool to identify shorts (less than minute) and filter them out if needed.
     Feel free to ask questions before do actions - especially if you noticed some inconsistency in user requests that might be error/misspelling.
     ```
   - Connect AI Agent input from When chat message received node.
   - Connect AI Agent memory input from Postgres Chat Memory.
   - Connect AI Agent language model input from OpenAI Chat Model.

6. **Add Execute Workflow Trigger Node:**
   - Create **Execute Workflow Trigger** node.
   - This node accepts external API commands with JSON structure including fields such as `command`, `query`.

7. **Add Switch Node:**
   - Add **Switch** node after Execute Workflow Trigger.
   - Configure multiple conditions based on `.json.command`:
     - `get_channel_details`
     - `video_details`
     - `comments`
     - `search`
     - `videos`
     - `analyze_thumbnail`
     - `video_transcription`

8. **Add YouTube API HTTP Request Nodes:**
   - For **Get Channel Details**:
     - HTTP GET to `https://www.googleapis.com/youtube/v3/channels`
     - Query: `part=snippet`, `forHandle={{handle}}`
     - Use Google API key in HTTP Query Auth.
   - For **Get Video Description**:
     - HTTP GET to `https://www.googleapis.com/youtube/v3/videos`
     - Query: `part=snippet,contentDetails,statistics`, `id={{video_id}}`
   - For **Get Videos by Channel**:
     - HTTP GET to `https://www.googleapis.com/youtube/v3/search`
     - Query: `part=snippet`, `channelId={{channel_id}}`, `order`, `maxResults`, `type=video`, `publishedAfter`
   - For **Get Comments**:
     - HTTP GET to `https://www.googleapis.com/youtube/v3/commentThreads`
     - Query: `part=id,snippet,replies`, `videoId={{video_id}}`, `maxResults=100`
   - For **Run Query**:
     - HTTP GET to `https://www.googleapis.com/youtube/v3/search`
     - Query: `part=snippet`, `q`, `order`, `type`, `maxResults`, `publishedAfter`

9. **Add Data Processing Nodes:**
   - Add **Edit Fields** (Set node) to transform comments JSON into readable text using expressions mapping over `items`.
   - Add **Response** (Set node) to output final responses.

10. **Add External API Nodes:**
    - **Get Video Transcription** (HTTP Request):
      - POST to Apify API `https://api.apify.com/v2/acts/dB9f4B02ocpTICIEY/run-sync-get-dataset-items`
      - JSON body: `{ "startUrls": ["{{video_url}}"] }`
      - Use Apify API key.
    - **OpenAI** (Langchain openAi node):
      - Configure for image analysis.
      - Input text prompt and image URL for thumbnail analysis.
      - Use OpenAI API key.

11. **Add Tool Workflow Nodes:**
    - Import or create the following tool workflows:
      - `get_channel_details`
      - `get_video_description`
      - `get_list_of_videos`
      - `get_list_of_comments`
      - `search`
      - `analyze_thumbnail`
      - `video_transcription`
    - Configure input schemas as described.
    - These are invoked by the AI Agent node as tools.

12. **Connect Switch Outputs:**
    - Connect each Switch output to the corresponding HTTP Request or tool workflow node.
    - Connect their outputs to the Response node.

13. **Final Connections:**
    - Connect Response nodes back to the initial API response or chat interface as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                            | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Detailed video guide available for workflow setup and usage: [YouTube Walkthrough (13 min)](https://youtu.be/6RmLZS8Yl4E) with visual thumbnail link.                                                                                                                                                 | Workflow setup video                                                                             |
| Workflow designed and maintained by Mark Shcherbakov from the 5minAI community. Visit [Mark Shcherbakov LinkedIn](https://www.linkedin.com/in/marklowcoding/) and [5minAI community](https://www.skool.com/5minai).                                                                                     | Project credits and branding                                                                    |
| API setup requires Google Cloud project with YouTube Data API enabled, plus API keys for OpenAI and Apify. Credentials must be created in n8n for each service.                                                                                                                                          | Setup instructions                                                                              |
| Workflow handles pagination and filtering of YouTube shorts (videos under 1 minute) by using video description metadata to improve data quality.                                                                                                                                                       | Important data filtering note                                                                   |
| Use of Postgres chat memory enables persistent and contextual AI agent chat sessions, improving user experience in multi-turn conversations.                                                                                                                                                            | Chat memory management                                                                          |

---

This documentation enables advanced users and AI agents to understand, reproduce, and extend the "Extract insights & analyse YouTube comments via AI Agent chat" workflow fully, anticipating potential failure points and integration challenges.