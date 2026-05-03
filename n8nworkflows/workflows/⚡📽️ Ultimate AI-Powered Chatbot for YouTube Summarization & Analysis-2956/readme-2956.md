⚡📽️ Ultimate AI-Powered Chatbot for YouTube Summarization & Analysis

https://n8nworkflows.xyz/workflows/-----ultimate-ai-powered-chatbot-for-youtube-summarization---analysis-2956


# ⚡📽️ Ultimate AI-Powered Chatbot for YouTube Summarization & Analysis

### 1. Workflow Overview

This workflow, titled **"Ultimate AI-Powered Chatbot for YouTube Summarization & Analysis"**, enables users to interact with an AI agent that extracts detailed metadata and transcripts from a YouTube video by its video ID. The AI agent then facilitates conversational exploration and analysis of the video content, providing summaries, key points, and clarifications.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Initialization**: Accepts user input (YouTube video ID) either via webhook chat message or from another workflow trigger, and sets up necessary variables including API keys.

- **1.2 YouTube Data Retrieval**: Constructs API requests to fetch video metadata and retrieves the video transcript using an external transcript extraction library.

- **1.3 Transcript Processing & Data Aggregation**: Processes the transcript by splitting and recombining segments, merges transcript data with video details, and aggregates all information into a single JSON object.

- **1.4 AI Agent Interaction & Response Generation**: Uses an AI agent powered by OpenAI models and LangChain tools to analyze the combined video data and respond to user queries conversationally.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

- **Overview:**  
  This block receives the YouTube video ID input either from an external workflow trigger or a chat message webhook. It initializes workflow variables including the Google API key and video ID for downstream nodes.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - When chat message received  
  - Workflow Variables

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point for external workflows to invoke this workflow with a JSON input containing a video ID.  
    - *Configuration:* Uses a JSON example input with a `query.videoId` field.  
    - *Inputs:* Triggered externally.  
    - *Outputs:* Passes input JSON downstream.  
    - *Edge Cases:* Missing or malformed input JSON may cause failures.

  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Webhook entry point for chat messages from users interacting with the AI agent.  
    - *Configuration:* Uses a webhook ID for receiving chat messages.  
    - *Inputs:* Incoming chat messages.  
    - *Outputs:* Passes chat input to AI agent node.  
    - *Edge Cases:* Webhook connectivity issues or malformed chat payloads.

  - **Workflow Variables**  
    - *Type:* Set  
    - *Role:* Sets key variables `GOOGLE_API_KEY` (user must replace placeholder) and `VIDEO_ID` extracted from input JSON query.  
    - *Configuration:*  
      - `GOOGLE_API_KEY` is a string placeholder `[Your-Google-API-Key]` to be replaced by user.  
      - `VIDEO_ID` is dynamically set from incoming JSON query.  
    - *Inputs:* From "When Executed by Another Workflow" node.  
    - *Outputs:* Passes variables downstream.  
    - *Edge Cases:* Missing API key or video ID will cause errors downstream.

---

#### 2.2 YouTube Data Retrieval

- **Overview:**  
  This block constructs the YouTube Data API URL dynamically using the video ID and API key, fetches video metadata via HTTP request, and retrieves the video transcript using a custom code node leveraging the `youtube-transcript` npm package.

- **Nodes Involved:**  
  - Create YouTube API URL  
  - Get YouTube Video Details  
  - Get YouTube Transcript

- **Node Details:**

  - **Create YouTube API URL**  
    - *Type:* Code  
    - *Role:* Dynamically builds the YouTube Data API URL for fetching video details.  
    - *Configuration:*  
      - Reads `VIDEO_ID` and `GOOGLE_API_KEY` from input JSON.  
      - Constructs URL with parts: snippet, contentDetails, status, statistics, player, topicDetails.  
      - Throws error if parameters missing.  
    - *Inputs:* From "Workflow Variables" node.  
    - *Outputs:* JSON with `youtubeUrl` string.  
    - *Edge Cases:* Missing parameters cause errors; malformed URL unlikely but possible.

  - **Get YouTube Video Details**  
    - *Type:* HTTP Request  
    - *Role:* Calls YouTube Data API using constructed URL to fetch video metadata.  
    - *Configuration:* URL set dynamically from previous node output.  
    - *Inputs:* From "Create YouTube API URL".  
    - *Outputs:* JSON with video metadata.  
    - *Edge Cases:* API key invalid or quota exceeded; network errors; video ID invalid or private.

  - **Get YouTube Transcript**  
    - *Type:* Code  
    - *Role:* Fetches the transcript of the YouTube video using the `youtube-transcript` npm package.  
    - *Configuration:*  
      - Extracts `VIDEO_ID` from input JSON.  
      - Uses `YoutubeTranscript.fetchTranscript(VIDEO_ID)` asynchronously.  
      - Returns transcript or error message.  
    - *Inputs:* From "Workflow Variables" node (shares same input as "Create YouTube API URL").  
    - *Outputs:* JSON with `youtubeId` and `transcript` array or error.  
    - *Edge Cases:* Transcript unavailable for some videos; network or package errors; invalid video ID.

---

#### 2.3 Transcript Processing & Data Aggregation

- **Overview:**  
  This block processes the transcript by splitting it into segments, concatenates text segments, merges transcript data with video metadata, and aggregates all into a single JSON object for the AI agent.

- **Nodes Involved:**  
  - Split Out Transcript Segments  
  - Combine Transcript Segments  
  - Merge YouTube Details & Transcript  
  - Create One JSON Object

- **Node Details:**

  - **Split Out Transcript Segments**  
    - *Type:* Split Out  
    - *Role:* Splits the transcript array into individual items for processing.  
    - *Configuration:* Splits on field `transcript`.  
    - *Inputs:* From "Get YouTube Transcript".  
    - *Outputs:* Multiple items, each representing a transcript segment.  
    - *Edge Cases:* Empty transcript array results in no output items.

  - **Combine Transcript Segments**  
    - *Type:* Summarize  
    - *Role:* Concatenates all transcript text segments into a single string.  
    - *Configuration:* Summarizes field `text` by concatenation separated by space.  
    - *Inputs:* From "Split Out Transcript Segments".  
    - *Outputs:* Single item with combined transcript text.  
    - *Edge Cases:* Large transcripts may hit size limits; empty input results in empty output.

  - **Merge YouTube Details & Transcript**  
    - *Type:* Merge  
    - *Role:* Combines video metadata and combined transcript text into one data stream.  
    - *Configuration:* Mode set to "combine" by position (pairwise).  
    - *Inputs:* From "Get YouTube Video Details" and "Combine Transcript Segments".  
    - *Outputs:* Merged JSON objects containing both metadata and transcript.  
    - *Edge Cases:* Mismatched input counts may cause incomplete merges.

  - **Create One JSON Object**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all merged data into a single JSON object for easier consumption.  
    - *Configuration:* Aggregates all item data into one object.  
    - *Inputs:* From "Merge YouTube Details & Transcript".  
    - *Outputs:* Single JSON object with all video details and transcript.  
    - *Edge Cases:* Large data size may impact performance.

---

#### 2.4 AI Agent Interaction & Response Generation

- **Overview:**  
  This block hosts the AI agent that interacts with the user chat input, uses a LangChain tool to analyze YouTube video data, maintains chat history memory, and generates contextual responses using OpenAI models.

- **Nodes Involved:**  
  - When chat message received  
  - YouTube Processing Tool (LangChain Tool Workflow)  
  - Window Buffer Memory  
  - gpt-4o-mini1 (OpenAI Chat Model)  
  - YouTube Video Agent (LangChain Agent)  
  - Respond with YouTube Details & Transcript

- **Node Details:**

  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Entry point for user chat messages to the AI agent.  
    - *Inputs:* Incoming chat messages.  
    - *Outputs:* Passes chat input to "YouTube Video Agent".  
    - *Edge Cases:* Webhook failures or malformed chat input.

  - **YouTube Processing Tool**  
    - *Type:* LangChain Tool Workflow  
    - *Role:* Defines a callable tool named `youtube_video_analyzer` that can be invoked by the AI agent to fetch video details and transcript by video ID.  
    - *Configuration:*  
      - Points to this same workflow ID (recursive call).  
      - Input schema requires JSON with `videoId`.  
      - Description instructs to extract video ID from user prompt and analyze video.  
    - *Inputs:* Invoked internally by "YouTube Video Agent".  
    - *Outputs:* Returns video data for AI processing.  
    - *Edge Cases:* Recursive calls must be carefully managed to avoid infinite loops.

  - **Window Buffer Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains chat history memory window for context retention in conversations.  
    - *Inputs:* Connected to "YouTube Video Agent" as memory.  
    - *Outputs:* Provides memory context to AI agent.  
    - *Edge Cases:* Memory size limits; loss of context if window too small.

  - **gpt-4o-mini1**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Role:* Provides the language model for the AI agent's natural language processing.  
    - *Configuration:*  
      - Model: `gpt-4o-mini` (a smaller GPT-4 variant).  
      - Temperature: 0.1 (low randomness for focused responses).  
      - Credentials: OpenAI API key configured.  
    - *Inputs:* Receives prompts from "YouTube Video Agent".  
    - *Outputs:* Generates AI responses.  
    - *Edge Cases:* API rate limits, invalid credentials, or timeouts.

  - **YouTube Video Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Core AI agent that processes user chat input, extracts YouTube video ID, calls the `youtube_video_analyzer` tool, and generates structured summaries and answers.  
    - *Configuration:*  
      - System message instructs detailed summarization and analysis with markdown formatting.  
      - Uses the `youtube_video_analyzer` tool to fetch video data.  
      - Connected to memory and language model nodes.  
    - *Inputs:* Chat input from "When chat message received".  
    - *Outputs:* AI-generated responses to user queries.  
    - *Edge Cases:* Failure to extract video ID, tool invocation errors, or incomplete data.

  - **Respond with YouTube Details & Transcript**  
    - *Type:* Set  
    - *Role:* Prepares the final response JSON with aggregated video data for output or further processing.  
    - *Configuration:* Assigns the aggregated data to a `response` field.  
    - *Inputs:* From "Create One JSON Object".  
    - *Outputs:* Final structured JSON response.  
    - *Edge Cases:* Large data payloads may affect downstream consumption.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                  | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                   |
|-----------------------------|----------------------------------|-------------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point for external workflow invocation     | -                                | Workflow Variables                |                                                                                              |
| Workflow Variables          | Set                              | Sets GOOGLE_API_KEY and VIDEO_ID variables       | When Executed by Another Workflow | Create YouTube API URL, Get YouTube Transcript | See note on workflow variables and Google API key setup                                     |
| Create YouTube API URL      | Code                             | Constructs YouTube Data API URL                   | Workflow Variables               | Get YouTube Video Details         |                                                                                              |
| Get YouTube Video Details   | HTTP Request                     | Fetches video metadata from YouTube API          | Create YouTube API URL           | Merge YouTube Details & Transcript | See YouTube Video Details API docs link                                                      |
| Get YouTube Transcript      | Code                             | Retrieves video transcript using youtube-transcript package | Workflow Variables               | Split Out Transcript Segments     | See YouTube Video Transcript notes and npm package link                                     |
| Split Out Transcript Segments | Split Out                       | Splits transcript array into individual segments | Get YouTube Transcript           | Combine Transcript Segments       |                                                                                              |
| Combine Transcript Segments | Summarize                       | Concatenates transcript text segments             | Split Out Transcript Segments    | Merge YouTube Details & Transcript |                                                                                              |
| Merge YouTube Details & Transcript | Merge                         | Combines video metadata and transcript data       | Get YouTube Video Details, Combine Transcript Segments | Create One JSON Object            |                                                                                              |
| Create One JSON Object      | Aggregate                       | Aggregates merged data into single JSON object    | Merge YouTube Details & Transcript | Respond with YouTube Details & Transcript |                                                                                              |
| Respond with YouTube Details & Transcript | Set                      | Prepares final response JSON                       | Create One JSON Object           | -                                |                                                                                              |
| When chat message received  | LangChain Chat Trigger           | Entry point for user chat messages                 | -                                | YouTube Video Agent              |                                                                                              |
| YouTube Video Agent         | LangChain Agent                 | AI agent for analyzing video transcript and metadata | When chat message received, Window Buffer Memory, gpt-4o-mini1, YouTube Processing Tool | -                                | See detailed system message instructions for summarization and analysis                     |
| YouTube Processing Tool     | LangChain Tool Workflow          | Tool to fetch video details and transcript by video ID | YouTube Video Agent             | YouTube Video Agent              | Recursive call to this workflow to fetch video data                                         |
| Window Buffer Memory        | LangChain Memory Buffer Window   | Maintains chat history context                     | -                                | YouTube Video Agent              |                                                                                              |
| gpt-4o-mini1                | LangChain OpenAI Chat Model      | Language model for AI agent responses              | YouTube Video Agent             | YouTube Video Agent              | Requires valid OpenAI API credentials                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes for Input:**

   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
     - Configure JSON example input with structure:  
       ```json
       {
         "query": {
           "videoId": "YouTube video id"
         }
       }
       ```
   
   - Add **LangChain Chat Trigger** node named `When chat message received`.  
     - Configure webhook ID as needed for chat integration.

2. **Set Workflow Variables:**

   - Add a **Set** node named `Workflow Variables`.  
     - Add two fields:  
       - `GOOGLE_API_KEY` (string) — set to placeholder `[Your-Google-API-Key]` (replace with actual key).  
       - `VIDEO_ID` (string) — set to expression: `={{ $json.query.videoId }}` to extract from input.  
     - Connect `When Executed by Another Workflow` → `Workflow Variables`.

3. **Construct YouTube API URL:**

   - Add a **Code** node named `Create YouTube API URL`.  
     - JavaScript code:  
       - Read `VIDEO_ID` and `GOOGLE_API_KEY` from input JSON.  
       - Validate presence, throw errors if missing.  
       - Construct URL:  
         ```
         https://www.googleapis.com/youtube/v3/videos?part=snippet,contentDetails,status,statistics,player,topicDetails&id=VIDEO_ID&key=GOOGLE_API_KEY
         ```  
       - Return JSON with `youtubeUrl`.  
     - Connect `Workflow Variables` → `Create YouTube API URL`.

4. **Fetch YouTube Video Details:**

   - Add an **HTTP Request** node named `Get YouTube Video Details`.  
     - Set URL to expression: `={{ $json.youtubeUrl }}`.  
     - Method: GET.  
     - Connect `Create YouTube API URL` → `Get YouTube Video Details`.

5. **Retrieve YouTube Transcript:**

   - Add a **Code** node named `Get YouTube Transcript`.  
     - Use JavaScript with `youtube-transcript` npm package:  
       - For each input item, extract `VIDEO_ID`.  
       - Call `YoutubeTranscript.fetchTranscript(VIDEO_ID)`.  
       - Return transcript or error message.  
     - Connect `Workflow Variables` → `Get YouTube Transcript`.

6. **Process Transcript Segments:**

   - Add a **Split Out** node named `Split Out Transcript Segments`.  
     - Configure to split on field `transcript`.  
     - Connect `Get YouTube Transcript` → `Split Out Transcript Segments`.

   - Add a **Summarize** node named `Combine Transcript Segments`.  
     - Configure to concatenate field `text` separated by space.  
     - Connect `Split Out Transcript Segments` → `Combine Transcript Segments`.

7. **Merge Video Details and Transcript:**

   - Add a **Merge** node named `Merge YouTube Details & Transcript`.  
     - Mode: Combine by position.  
     - Connect `Get YouTube Video Details` → input 1.  
     - Connect `Combine Transcript Segments` → input 2.

   - Add an **Aggregate** node named `Create One JSON Object`.  
     - Aggregate all item data into one JSON object.  
     - Connect `Merge YouTube Details & Transcript` → `Create One JSON Object`.

   - Add a **Set** node named `Respond with YouTube Details & Transcript`.  
     - Assign field `response` with value `={{ $json.data }}`.  
     - Connect `Create One JSON Object` → `Respond with YouTube Details & Transcript`.

8. **Configure AI Agent Interaction:**

   - Add a **LangChain Tool Workflow** node named `YouTube Processing Tool`.  
     - Set `name` to `youtube_video_analyzer`.  
     - Set `workflowId` to this workflow's ID (to enable recursive calls).  
     - Provide description explaining tool purpose and input schema requiring `videoId`.  
     - Enable input schema specification.  
     - Connect to AI agent node later.

   - Add a **LangChain Memory Buffer Window** node named `Window Buffer Memory`.  
     - Default configuration to maintain chat history context.

   - Add a **LangChain OpenAI Chat Model** node named `gpt-4o-mini1`.  
     - Model: `gpt-4o-mini`.  
     - Temperature: 0.1.  
     - Configure OpenAI API credentials.

   - Add a **LangChain Agent** node named `YouTube Video Agent`.  
     - Set `text` input to expression: `={{ $json.chatInput }}` (from chat trigger).  
     - Define system message with detailed instructions for extracting video ID, analyzing transcript, and formatting output in markdown with structured summaries.  
     - Connect AI model (`gpt-4o-mini1`), memory (`Window Buffer Memory`), and tool (`YouTube Processing Tool`) nodes as inputs.  
     - Connect `When chat message received` → `YouTube Video Agent`.

9. **Connect Outputs:**

   - The AI agent node outputs responses directly to chat clients or downstream systems as configured.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Google Cloud Console API Setup Guide                                                                          | https://cloud.google.com/docs/get-started/access-apis                                              |
| YouTube Data API Key Setup Guide                                                                               | https://developers.google.com/youtube/v3/docs                                                      |
| n8n Local Installation Guide                                                                                   | https://docs.n8n.io/integrations/creating-nodes/test/run-node-locally/                             |
| YouTube Transcript npm Package                                                                                 | https://www.npmjs.com/package/youtube-transcript                                                   |
| DeepSeek API Documentation                                                                                      | https://api-docs.deepseek.com/                                                                     |
| OpenAI API Key Management                                                                                        | https://platform.openai.com/api-keys                                                               |
| Workflow Purpose and Use Cases                                                                                  | Embedded in sticky notes within the workflow for user reference                                    |
| Sample Prompts for User Interaction                                                                             | "Tell me about this YouTube video with id: JWfNLF_g_V0" and "Can you provide a list of key takeaways from this video with id: [youtube-video-id]?" |

---

This documentation provides a comprehensive, structured reference for understanding, reproducing, and modifying the "Ultimate AI-Powered Chatbot for YouTube Summarization & Analysis" workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with this workflow.