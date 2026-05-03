⚡AI-Powered YouTube Video Summarization & Analysis

https://n8nworkflows.xyz/workflows/-ai-powered-youtube-video-summarization---analysis-2679


# ⚡AI-Powered YouTube Video Summarization & Analysis

### 1. Workflow Overview

This workflow automates the transformation of public YouTube videos into structured, AI-generated summaries and analyses. It is designed for users who want quick, clear insights from video content without watching the entire video, enabling efficient research, learning, and content review.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives a YouTube URL via webhook and extracts the URL from the incoming request.
- **1.2 Video Identification & Metadata Retrieval:** Extracts the video ID from the URL, fetches detailed video metadata using the YouTube API.
- **1.3 Transcript Extraction & Processing:** Retrieves the video transcript, splits it into manageable segments, then concatenates these segments into a single text block.
- **1.4 AI-Powered Summarization & Analysis:** Sends the concatenated transcript to an OpenAI GPT-4o-mini-powered node for structured, hierarchical summarization with markdown formatting.
- **1.5 Response & Notifications:** Packages the summary and video metadata into a response object, returns it to the webhook caller, and optionally sends a Telegram notification with video title and URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives an HTTP POST request containing a YouTube URL and extracts it for downstream processing.

**Nodes Involved:**  
- Webhook  
- Get YouTube URL

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook (n8n-nodes-base.webhook)  
  - Role: Entry point for external POST requests with YouTube URL payload.  
  - Configuration: POST method at path `/ytube`; response mode set to `responseNode` to wait for workflow completion before responding.  
  - Inputs: Incoming HTTP request containing JSON with a `youtubeUrl` field.  
  - Outputs: JSON payload forwarded to the next node.  
  - Edge Cases: Invalid or missing URL in POST body; improper HTTP method; webhook path conflicts.  

- **Get YouTube URL**  
  - Type: Set node (n8n-nodes-base.set)  
  - Role: Extracts and assigns `youtubeUrl` from incoming JSON body to a named variable for clarity.  
  - Configuration: Sets `youtubeUrl` string from `{{$json.body.youtubeUrl}}`.  
  - Inputs: JSON object from Webhook node.  
  - Outputs: JSON with explicit `youtubeUrl` property.  
  - Edge Cases: Missing or malformed `youtubeUrl` field in request body.  

---

#### 1.2 Video Identification & Metadata Retrieval

**Overview:**  
Extracts YouTube video ID from the URL, then retrieves video metadata like title and description using the YouTube API.

**Nodes Involved:**  
- YouTube Video ID  
- Get YouTube Video

**Node Details:**

- **YouTube Video ID**  
  - Type: Code node (n8n-nodes-base.code)  
  - Role: Parses the YouTube video ID from multiple URL formats using a regex pattern.  
  - Configuration: JavaScript function extracts video ID from `youtubeUrl`.  
  - Key Expression: Regex matching both `youtube.com` and `youtu.be` URLs for 11-character video ID.  
  - Inputs: JSON with `youtubeUrl`.  
  - Outputs: JSON with `videoId` field.  
  - Edge Cases: Invalid or unsupported URL formats, missing video ID, regex failure resulting in null.  

- **Get YouTube Video**  
  - Type: YouTube node (n8n-nodes-base.youTube)  
  - Role: Queries YouTube API for video details using the extracted `videoId`.  
  - Configuration: Operation `get` on resource `video` with dynamic `videoId`.  
  - Inputs: JSON with `videoId`.  
  - Outputs: Video metadata JSON, including snippet with title, description, and ID.  
  - Requirements: Valid YouTube API credentials configured in n8n.  
  - Edge Cases: Invalid video ID, private or deleted videos, API rate limits, network errors.  

---

#### 1.3 Transcript Extraction & Processing

**Overview:**  
Fetches the full transcript of the YouTube video, splits it into discrete segments, then concatenates these segments into a single text block for summarization.

**Nodes Involved:**  
- YouTube Transcript  
- Split Out  
- Concatenate

**Node Details:**

- **YouTube Transcript**  
  - Type: YouTube Transcription node (n8n-nodes-youtube-transcription.youtubeTranscripter)  
  - Role: Automatically extracts the full transcript text from the public YouTube video.  
  - Configuration: Default; no parameters specified, assumes video ID is used internally.  
  - Inputs: Triggered after video metadata retrieval (implicit connection).  
  - Outputs: Array of transcript segments with a `transcript` field.  
  - Edge Cases: Videos without transcripts, disabled captions, private videos, API failures.  

- **Split Out**  
  - Type: Split Out node (n8n-nodes-base.splitOut)  
  - Role: Separates each transcript segment into individual items based on the `transcript` field for further processing.  
  - Configuration: Splits on `transcript` field.  
  - Inputs: Array of transcript segments.  
  - Outputs: One item per transcript segment.  
  - Edge Cases: Empty or missing transcript field, no segments to split.  

- **Concatenate**  
  - Type: Summarize node (n8n-nodes-base.summarize)  
  - Role: Concatenates all split transcript segments into a unified text string, separated by spaces.  
  - Configuration: Aggregates all `text` fields by concatenation with space separator.  
  - Inputs: Transcript segments from Split Out node.  
  - Outputs: Single item with `concatenated_text` field.  
  - Edge Cases: Excessively long transcripts exceeding API limits, missing text fields.  

---

#### 1.4 AI-Powered Summarization & Analysis

**Overview:**  
Uses an OpenAI language model (GPT-4o-mini) to analyze the concatenated transcript and generate a structured summary with markdown formatting, highlighting main topics and key points.

**Nodes Involved:**  
- gpt-4o-mini  
- Summarize & Analyze Transcript

**Node Details:**

- **gpt-4o-mini**  
  - Type: OpenAI Chat LLM node (lmChatOpenAi)  
  - Role: Provides AI language model capabilities, serving as the backend for the summarization chain.  
  - Configuration: Default OpenAI parameters with GPT-4o-mini selected.  
  - Inputs: Prompt text from Summarize & Analyze Transcript node.  
  - Outputs: AI-generated text response.  
  - Credential: Requires valid OpenAI API credentials configured in n8n.  
  - Edge Cases: API rate limits, timeout, invalid prompt formatting, credential issues.  

- **Summarize & Analyze Transcript**  
  - Type: Chain LLM node (chainLlm)  
  - Role: Constructs a detailed prompt instructing GPT-4o-mini to produce a hierarchical, markdown-formatted summary of the transcript.  
  - Configuration:  
    - Uses a custom prompt specifying:  
      - Breaking content into main topics with level 2 headers (##)  
      - Bullet points for essential concepts  
      - Concise explanations preserving technical accuracy  
      - Markdown formatting rules including bold terms and tables  
    - Injects `concatenated_text` into the prompt dynamically.  
  - Inputs: Concatenated transcript text.  
  - Outputs: Structured AI summary.  
  - Edge Cases: Prompt injection errors, incomplete text input, AI hallucinations or inaccuracies.  

---

#### 1.5 Response & Notifications

**Overview:**  
Packages the AI summary along with video metadata into a response object sent back to the webhook caller and optionally sends a Telegram notification with video details.

**Nodes Involved:**  
- Response Object  
- Respond to Webhook  
- Telegram

**Node Details:**

- **Response Object**  
  - Type: Set node (n8n-nodes-base.set)  
  - Role: Constructs the final JSON response with fields: summary, topics (empty array placeholder), title, description, video id, and original YouTube URL.  
  - Configuration:  
    - `summary` assigned from AI-generated summary text  
    - `topics` initialized as an empty array (could be used for future expansion)  
    - `title`, `description`, `id` dynamically extracted from YouTube video metadata node  
    - `youtubeUrl` from original webhook input  
  - Inputs: Output of Summarize & Analyze Transcript and Get YouTube Video nodes.  
  - Outputs: JSON ready for webhook response and notifications.  
  - Edge Cases: Missing or null summary or metadata fields.  

- **Respond to Webhook**  
  - Type: Respond node (n8n-nodes-base.respondToWebhook)  
  - Role: Sends the constructed response object back to the original HTTP requestor, completing the webhook cycle.  
  - Configuration: Default, sending previous node’s JSON output as response.  
  - Inputs: Response Object node output.  
  - Outputs: HTTP response to webhook caller.  
  - Edge Cases: Timeout if workflow exceeds webhook response window, malformed response data.  

- **Telegram**  
  - Type: Telegram node (n8n-nodes-base.telegram)  
  - Role: Sends an optional notification message to a configured Telegram chat with video title and URL.  
  - Configuration:  
    - Message text formatted as:  
      ```  
      {{ $json.title }}  
      {{ $json.youtubeUrl }}  
      ```  
    - Parse mode set to HTML  
    - No attribution appended  
  - Inputs: Response Object node output.  
  - Outputs: Message sent to Telegram chat.  
  - Credential: Requires valid Telegram Bot API credentials.  
  - Edge Cases: Invalid bot token, chat ID misconfiguration, Telegram API failures.  

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                            | Input Node(s)           | Output Node(s)                 | Sticky Note                              |
|-----------------------------|-----------------------------------------|--------------------------------------------|-------------------------|-------------------------------|-----------------------------------------|
| Webhook                     | HTTP Webhook (n8n-nodes-base.webhook)   | Receives YouTube URL HTTP POST requests   | —                       | Get YouTube URL                |                                         |
| Get YouTube URL             | Set (n8n-nodes-base.set)                 | Extracts `youtubeUrl` from webhook payload | Webhook                 | YouTube Video ID              |                                         |
| YouTube Video ID            | Code (n8n-nodes-base.code)               | Extracts video ID from YouTube URL         | Get YouTube URL          | Get YouTube Video             |                                         |
| Get YouTube Video           | YouTube API node (n8n-nodes-base.youTube) | Retrieves video metadata                    | YouTube Video ID         | YouTube Transcript            | Requires valid YouTube API credentials  |
| YouTube Transcript          | YouTube Transcription node               | Extracts transcript from video              | Get YouTube Video       | Split Out                     | Supports multiple YouTube URL formats   |
| Split Out                   | Split Out (n8n-nodes-base.splitOut)     | Splits transcript array into individual items | YouTube Transcript       | Concatenate                   |                                         |
| Concatenate                 | Summarize (n8n-nodes-base.summarize)    | Concatenates transcript segments into text | Split Out                | Summarize & Analyze Transcript |                                         |
| gpt-4o-mini                 | OpenAI Chat LLM (lmChatOpenAi)           | AI language model for summarization         | Summarize & Analyze Transcript | Summarize & Analyze Transcript | Requires OpenAI API credentials          |
| Summarize & Analyze Transcript | Chain LLM (chainLlm)                   | Constructs prompt and analyzes transcript   | Concatenate, gpt-4o-mini | Response Object              | Structured markdown summary with technical accuracy |
| Response Object             | Set (n8n-nodes-base.set)                 | Packages summary and video metadata         | Summarize & Analyze Transcript, Get YouTube Video | Respond to Webhook, Telegram |                                         |
| Respond to Webhook          | Respond to Webhook (n8n-nodes-base.respondToWebhook) | Returns final response to webhook caller    | Response Object          | —                             |                                         |
| Telegram                    | Telegram node (n8n-nodes-base.telegram) | Sends Telegram notification with video info | Response Object          | —                             | Optional notification; requires Telegram credentials |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - HTTP Method: POST  
   - Path: `ytube`  
   - Response Mode: `responseNode` (to send response after workflow completes)  
   - Save credentials if needed (none required)  

2. **Add Set Node 'Get YouTube URL'**  
   - Type: Set  
   - Add assignment:  
     - Name: `youtubeUrl`  
     - Type: String  
     - Value: `={{ $json.body.youtubeUrl }}`  
   - Connect Webhook node main output to this node input  

3. **Add Code Node 'YouTube Video ID'**  
   - Type: Code  
   - Language: JavaScript  
   - Paste JS code to extract YouTube video ID from URL using regex:  
     ```js
     const extractYoutubeId = (url) => {
       const pattern = /(?:youtube\.com\/(?:[^\/]+\/.+\/|(?:v|e(?:mbed)?)\/|.*[?&]v=)|youtu\.be\/)([^"&?\/\s]{11})/;
       const match = url.match(pattern);
       return match ? match[1] : null;
     };
     const youtubeUrl = items[0].json.youtubeUrl;
     return [{ json: { videoId: extractYoutubeId(youtubeUrl) } }];
     ```  
   - Connect output of Get YouTube URL node to this node  

4. **Add YouTube Node 'Get YouTube Video'**  
   - Type: YouTube (API) node  
   - Resource: Video  
   - Operation: Get  
   - Video ID: `={{ $json.videoId }}`  
   - Configure YouTube API credentials (API key or OAuth2)  
   - Connect YouTube Video ID node output here  

5. **Add YouTube Transcript Node**  
   - Type: YouTube Transcription (custom community or official node)  
   - No parameters needed  
   - Connect output of Get YouTube Video node to this node  

6. **Add Split Out Node 'Split Out'**  
   - Type: Split Out  
   - Field to split out: `transcript`  
   - Connect YouTube Transcript node output here  

7. **Add Summarize Node 'Concatenate'**  
   - Type: Summarize  
   - Fields to summarize:  
     - Field: `text`  
     - Separate By: space `" "`  
     - Aggregation: concatenate  
   - Connect Split Out node output here  

8. **Add Chain LLM Node 'Summarize & Analyze Transcript'**  
   - Type: Chain LLM (Langchain)  
   - Text prompt:  
     ```
     Please analyze the given text and create a structured summary following these guidelines:

     1. Break down the content into main topics using Level 2 headers (##)
     2. Under each header:
        - List only the most essential concepts and key points
        - Use bullet points for clarity
        - Keep explanations concise
        - Preserve technical accuracy
        - Highlight key terms in bold
     3. Organize the information in this sequence:
        - Definition/Background
        - Main characteristics
        - Implementation details
        - Advantages/Disadvantages
     4. Format requirements:
        - Use markdown formatting
        - Keep bullet points simple (no nesting)
        - Bold important terms using **term**
        - Use tables for comparisons
        - Include relevant technical details

     Please provide a clear, structured summary that captures the core concepts while maintaining technical accuracy.

     Here is the text: {{ $json.concatenated_text }}
     ```  
   - Prompt type: Define  
   - Connect Concatenate node output here  

9. **Add OpenAI Chat LLM Node 'gpt-4o-mini'**  
   - Type: OpenAI Chat (lmChatOpenAi)  
   - Model: GPT-4o-mini  
   - Configure OpenAI credentials (API key)  
   - Connect this node as AI language model inside Summarize & Analyze Transcript node (ai_languageModel input)  

10. **Add Set Node 'Response Object'**  
    - Type: Set  
    - Assignments:  
      - `summary`: `={{ $json.text }}` (output of Summarize & Analyze Transcript)  
      - `topics`: `=[]` (empty array placeholder)  
      - `title`: `={{ $('Get YouTube Video').item.json.snippet.title }}`  
      - `description`: `={{ $('Get YouTube Video').item.json.snippet.description }}`  
      - `id`: `={{ $('Get YouTube Video').item.json.id }}`  
      - `youtubeUrl`: `={{ $('Webhook').item.json.body.youtubeUrl }}`  
    - Connect Summarize & Analyze Transcript node output to this node  

11. **Add Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Default configuration to respond with the previous node’s JSON output  
    - Connect Response Object node output here  

12. **(Optional) Add Telegram Node**  
    - Type: Telegram  
    - Text field:  
      ```
      {{ $json.title }}
      {{ $json.youtubeUrl }}
      ```  
    - Additional Fields:  
      - Parse Mode: HTML  
      - Append Attribution: false  
    - Configure Telegram Bot API credentials (bot token and chat ID)  
    - Connect Response Object node output here  

13. **Verify all connections as follows:**  
    - Webhook → Get YouTube URL → YouTube Video ID → Get YouTube Video → YouTube Transcript → Split Out → Concatenate → Summarize & Analyze Transcript → Response Object → Respond to Webhook  
    - Summarize & Analyze Transcript → gpt-4o-mini (ai_languageModel input)  
    - Response Object → Telegram (optional)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow enables instant, AI-driven summarization of YouTube videos, saving hours of manual viewing time.                                                     | Workflow overview and purpose                                                                    |
| Uses GPT-4o-mini, a capable and cost-effective OpenAI language model variant, for analysis and summarization.                                                  | AI model choice                                                                                  |
| Transcript extraction supports multiple YouTube URL formats and only works for public videos with available transcripts.                                       | YouTube Transcript node capability                                                              |
| Telegram notifications are optional and require proper bot setup with valid credentials.                                                                       | Telegram node integration                                                                        |
| Regex for video ID extraction covers standard YouTube watch URLs and shortened youtu.be links.                                                                | Code node logic                                                                                  |
| Prompt used for summarization emphasizes markdown formatting, bullet points, and technical accuracy to produce clear, useful output for professional users.    | Summarization prompt details                                                                     |
| For troubleshooting, verify API credentials (YouTube, OpenAI, Telegram) and ensure videos have transcripts available.                                         | General operational advice                                                                       |
| For more details on n8n nodes and APIs, visit official docs: https://docs.n8n.io/                                                                              | n8n official documentation                                                                       |
| For YouTube API info and quota management: https://developers.google.com/youtube/v3                                                                         | YouTube API documentation                                                                        |
| OpenAI API docs and best practices: https://platform.openai.com/docs                                                                                          | OpenAI API reference                                                                             |
| Telegram Bot API docs: https://core.telegram.org/bots/api                                                                                                    | Telegram API reference                                                                           |

---

This reference fully describes the structure, logic, and configuration of the "⚡AI-Powered YouTube Video Summarization & Analysis" workflow, enabling reproduction, modification, and operational understanding by both human experts and automation agents.