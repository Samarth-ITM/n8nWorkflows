Automated YouTube Video to Blog Post Conversion with Gemini AI Transcription

https://n8nworkflows.xyz/workflows/automated-youtube-video-to-blog-post-conversion-with-gemini-ai-transcription-8037


# Automated YouTube Video to Blog Post Conversion with Gemini AI Transcription

### 1. Workflow Overview

This workflow automates the conversion of a YouTube video into a polished blog post using AI transcription and content generation. It is designed to:

- Accept a YouTube video URL via webhook input.
- Download the video’s audio track in MP3 format.
- Transcribe the audio into text using Google Gemini AI.
- Use an AI agent (LangChain with Google Gemini Chat Model) to transform the raw transcript into a structured blog post including title, category, tags, a short description, and formatted HTML content.
- Return the final blog post JSON as a webhook response.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Audio Download:** Receives the YouTube URL, timestamps the request, downloads the audio, and reads it as binary.
- **1.2 AI Transcription:** Transcribes the downloaded audio file into text using Google Gemini’s audio transcription model.
- **1.3 AI Content Generation and Output:** Processes the transcript with an AI Agent to produce a structured blog post in JSON format and responds via webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Input Reception and Audio Download

**Overview:**  
This block handles receiving the YouTube URL input, generating a unique timestamp for file naming, downloading the audio from the video URL, and reading the downloaded MP3 file as binary data for further processing.

**Nodes Involved:**  
- Get Youtube Url  
- Get Time  
- Download the Youtube Audio  
- Read Output MP3 Binary  
- Sticky Note (explaining purpose)

**Node Details:**

- **Get Youtube Url**  
  - *Type:* Webhook  
  - *Role:* Entry point, receives POST request containing YouTube video URL JSON payload.  
  - *Configuration:* Webhook path set to a UUID. Responds via "Respond The Blog Post" node downstream.  
  - *Inputs:* External HTTP POST request.  
  - *Outputs:* Passes JSON payload with video URL to next node.  
  - *Edge cases:* Invalid or missing URL, webhook timeout, malformed POST body.

- **Get Time**  
  - *Type:* Set  
  - *Role:* Adds a numeric timestamp (`now_unix`) representing the current time in milliseconds, used to uniquely name the audio file.  
  - *Configuration:* Expression `={{ new Date().getTime() }}` to generate timestamp dynamically.  
  - *Inputs:* From webhook node.  
  - *Outputs:* Passes timestamp to next node for filename generation.

- **Download the Youtube Audio**  
  - *Type:* Execute Command  
  - *Role:* Runs `yt-dlp` command-line tool to download the YouTube video audio only as MP3, saving it to a uniquely named file based on timestamp.  
  - *Configuration:* Command dynamically constructed with expression referencing the video URL and timestamp, e.g., `yt-dlp -x --audio-format mp3 -o output-{{timestamp}}.mp3 '{{videoUrl}}'`.  
  - *Inputs:* Timestamp and video URL from previous nodes.  
  - *Outputs:* Triggers download and creates MP3 file on local disk.  
  - *Edge cases:* `yt-dlp` failures (network, invalid URL), file system permissions, command timeout.

- **Read Output MP3 Binary**  
  - *Type:* Read/Write File  
  - *Role:* Reads the downloaded MP3 file from disk into binary format for AI transcription input.  
  - *Configuration:* File selector expression matches filename using the timestamp, e.g., `output-{{timestamp}}.mp3`.  
  - *Inputs:* Filename from previous node.  
  - *Outputs:* Passes binary audio data to transcription node.  
  - *Edge cases:* File not found, read permissions, incomplete download.

- **Sticky Note**  
  - *Role:* Documentation explaining this block’s purpose: "Get YouTube Video Audio: Download the YouTube video from the provided URL and prepare it as binary data."

---

#### 2.2 Block 2: AI Transcription

**Overview:**  
This block transcribes the binary audio into text using Google Gemini’s audio transcription model.

**Nodes Involved:**  
- Transcribe a recording

**Node Details:**

- **Transcribe a recording**  
  - *Type:* LangChain Google Gemini (audio transcription)  
  - *Role:* Uses Google Gemini "gemini-1.5-flash" model to convert binary audio into text transcript.  
  - *Configuration:* Model ID set to `"models/gemini-1.5-flash"`, input type set to binary, resource set to audio.  
  - *Credentials:* Google Palm API credentials configured.  
  - *Inputs:* Binary audio from "Read Output MP3 Binary".  
  - *Outputs:* JSON containing transcript text in `content.parts[0].text`.  
  - *Edge cases:* API authentication errors, audio format incompatibility, transcription latency, rate limits.

---

#### 2.3 Block 3: AI Content Generation and Output

**Overview:**  
This block processes the transcript text through an AI Agent that structures the content into a blog post JSON, then attaches video URL and full transcript text to the output, and finally returns the blog post via webhook response.

**Nodes Involved:**  
- AI Agent  
- Structured Output Parser  
- Google Gemini Chat Model  
- Add Video Url to Output  
- Respond The Blog Post  
- Sticky Note (explaining purpose)

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Core node that sends prompt and transcript text to language model, receiving structured blog post content as JSON.  
  - *Configuration:*  
    - Prompt includes instructions to create a blog post with title, category, tags, short description, and HTML body from the transcript.  
    - System message enforces output strictly as JSON in markdown code block, tone, language consistency, and output structure.  
    - Input text is the transcript extracted from `"Transcribe a recording"` node output.  
    - Output parser enabled to parse JSON response.  
  - *Input:* Transcript text.  
  - *Output:* Parsed JSON blog post.  
  - *Edge cases:* AI model interpretation errors, invalid JSON output, empty transcript input, API quota limits.

- **Google Gemini Chat Model**  
  - *Type:* LangChain LLM Chat (Google Gemini)  
  - *Role:* Language model used by AI Agent for content generation.  
  - *Credentials:* Uses Google Palm API.  
  - *Input/Output:* Connected as the AI language model for the AI Agent node.  
  - *Edge cases:* API downtime, authentication errors.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI Agent output ensuring valid JSON matching example schema including category, tags, title, short_description, and body.  
  - *Configuration:* Auto-fix enabled, with example JSON schema provided.  
  - *Edge cases:* Malformed AI output needing correction, parser failures.

- **Add Video Url to Output**  
  - *Type:* Set  
  - *Role:* Adds the original YouTube URL and full transcript text as additional fields to the blog post JSON for completeness.  
  - *Configuration:* Sets `videoUrl` from original webhook input and `fulltext` from transcription output.  
  - *Input:* AI Agent JSON output and original URL.  
  - *Output:* Combined enriched JSON.  
  - *Edge cases:* Missing fields in inputs.

- **Respond The Blog Post**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends final JSON blog post as HTTP response to the initial webhook requestor.  
  - *Input:* Enriched JSON object.  
  - *Edge cases:* Response timeout, client disconnect.

- **Sticky Note1**  
  - *Role:* Documentation explaining this block: "Preparing the Blog Post: The AI Agent processes the full YouTube video transcript and transforms it into a blog post with content, a short description, a category, and tags. The output is provided as a structured JSON."

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role                              | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                      |
|-------------------------|---------------------------------------|----------------------------------------------|--------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Get Youtube Url         | Webhook                               | Entry point, receive video URL                | External HTTP POST       | Get Time                   |                                                                                                 |
| Get Time                | Set                                   | Generate timestamp for filename                | Get Youtube Url          | Download the Youtube Audio |                                                                                                 |
| Download the Youtube Audio | Execute Command                      | Download YouTube audio as MP3                   | Get Time                 | Read Output MP3 Binary     | Get YouTube Video Audio: Download the YouTube video from the provided URL and prepare it as binary data. |
| Read Output MP3 Binary  | Read/Write File                       | Read downloaded MP3 as binary                    | Download the Youtube Audio | Transcribe a recording     |                                                                                                 |
| Transcribe a recording  | LangChain Google Gemini (audio)       | Transcribe audio to text                         | Read Output MP3 Binary   | AI Agent                   |                                                                                                 |
| AI Agent                | LangChain Agent                      | Convert transcript to structured blog post JSON | Transcribe a recording   | Add Video Url to Output    | Preparing the Blog Post: The AI Agent processes the full YouTube video transcript and transforms it into a blog post with content, a short description, a category, and tags. The output is provided as a structured JSON. |
| Google Gemini Chat Model | LangChain LLM Chat (Google Gemini)   | Language model used by AI Agent                  |                          | AI Agent                   |                                                                                                 |
| Structured Output Parser | LangChain Structured Output Parser    | Parse AI Agent JSON output                       |                          | AI Agent                   |                                                                                                 |
| Add Video Url to Output | Set                                   | Append original video URL and full transcript   | AI Agent                 | Respond The Blog Post      |                                                                                                 |
| Respond The Blog Post   | Respond to Webhook                    | Send final blog post JSON response               | Add Video Url to Output  |                          |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: Get Youtube Url  
   - Type: Webhook (POST)  
   - Configure path to unique identifier (e.g., UUID).  
   - No authentication required for testing.  
   - This node receives JSON input containing the YouTube video URL (expected field: `body.videoUrl`).

2. **Create Set Node for Timestamp:**  
   - Name: Get Time  
   - Type: Set  
   - Add a field: `now_unix` (Number) with expression: `={{ new Date().getTime() }}`  
   - Connect from "Get Youtube Url".

3. **Create Execute Command Node to Download Audio:**  
   - Name: Download the Youtube Audio  
   - Type: Execute Command  
   - Command:  
     ```
     yt-dlp -x --audio-format mp3 -o output-{{ $json.now_unix }}.mp3 '{{ $('Get Youtube Url').item.json.body.videoUrl }}'
     ```  
   - Connect from "Get Time".  
   - Ensure `yt-dlp` is installed and accessible on the n8n host environment.  
   - Permissions needed for file writing.

4. **Create Read/Write File Node to Read Audio:**  
   - Name: Read Output MP3 Binary  
   - Type: Read/Write File  
   - Set "File Selector" to `output-{{ $('Get Time').item.json.now_unix }}.mp3`  
   - Connect from "Download the Youtube Audio".

5. **Create Google Gemini Audio Transcription Node:**  
   - Name: Transcribe a recording  
   - Type: LangChain Google Gemini (Audio)  
   - Model ID: `models/gemini-1.5-flash`  
   - Resource: Audio  
   - Input Type: Binary  
   - Credentials: Configure Google Palm API credentials with valid API key.  
   - Connect from "Read Output MP3 Binary".

6. **Create LangChain Agent Node:**  
   - Name: AI Agent  
   - Type: LangChain Agent  
   - Prompt Text:  
     ```
     Please create a blog post from the following video transcript. Identify the key themes, structure the content logically, and generate a title, category, and tags.

     Transcript:
     {{ $json.content.parts[0].text }}
     ```  
   - System Message (Options):  
     ```
     You are an expert content strategist and editor. Your task is to transform a raw video transcript into a well-structured and engaging blog post.

     You must adhere to the following rules:
     1. Your entire response must be a single JSON object enclosed in a markdown code block (```json ... ```).
     2. Do NOT include any introductory phrases, explanations, apologies, or any text whatsoever outside of the JSON markdown block.
     3. The tone should be informative, clear, and engaging, transforming the conversational transcript into polished written content.
     4. The language of all generated text in the output JSON must match the primary language of the input transcript.
     5. The JSON object must have this exact structure:
     {
       "category": "A single, top-level category based on the content.",
       "tags": [
         "An array of 3 to 5 relevant, lowercase string tags."
       ],
       "title": "A creative, compelling title for the blog post.",
       "short_description": "A concise, one-to-two sentence summary of the blog post, optimized for search engines (around 150-160 characters).",
       "body": "The full blog post content formatted in HTML, including <h1>, <h2>, and <p> tags."
     }
     ```  
   - Enable Output Parser with structured JSON parsing.  
   - Connect from "Transcribe a recording".

7. **Configure Google Gemini Chat Model Node:**  
   - Name: Google Gemini Chat Model  
   - Type: LangChain LLM Chat  
   - Credentials: Google Palm API  
   - Model: Default or specify as needed.  
   - Connect as AI language model for "AI Agent".

8. **Configure Structured Output Parser Node:**  
   - Name: Structured Output Parser  
   - Type: LangChain Structured Output Parser  
   - Provide JSON schema example matching expected blog post structure (category, tags, title, short_description, body).  
   - Enable autoFix for minor JSON issues.  
   - Connect as output parser for "AI Agent".

9. **Create Set Node to Add Video URL and Full Transcript:**  
   - Name: Add Video Url to Output  
   - Type: Set  
   - Fields to set:  
     - `videoUrl` = `={{ $('Get Youtube Url').item.json.body.url }}`  
     - `fulltext` = `={{ $('Transcribe a recording').item.json.content.parts[0].text }}`  
   - Include other fields from AI Agent output.  
   - Connect from "AI Agent".

10. **Create Respond to Webhook Node:**  
    - Name: Respond The Blog Post  
    - Type: Respond to Webhook  
    - Connect from "Add Video Url to Output".  
    - This node sends the final blog post JSON as HTTP response to the original webhook POST request.

11. **Connect nodes in order:**  
    Get Youtube Url → Get Time → Download the Youtube Audio → Read Output MP3 Binary → Transcribe a recording → AI Agent → Add Video Url to Output → Respond The Blog Post

12. **Add sticky notes (optional) to document:**  
    - Near initial nodes: "Get YouTube Video Audio: Download the YouTube video from the provided URL and prepare it as binary data."  
    - Near AI Agent node: "Preparing the Blog Post: The AI Agent processes the full YouTube video transcript and transforms it into a blog post with content, a short description, a category, and tags. The output is provided as a structured JSON."

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow requires `yt-dlp` installed on the n8n host machine for audio extraction.                                | https://github.com/yt-dlp/yt-dlp                                                                   |
| Google Palm API credentials must be configured with access to Gemini models for transcription and chat completion.   | https://developers.generativeai.google/api/gemini                                                   |
| The AI Agent enforces output as JSON in a markdown code block for robust parsing and downstream processing.           | Custom prompt and system message in AI Agent node                                                  |
| The transcription model used is `gemini-1.5-flash`, optimized for audio-to-text transcription.                        | Google Gemini documentation                                                                        |
| The output blog post JSON includes SEO-friendly short description and HTML formatting for easy integration into CMS. | Structured Output Parser example JSON schema                                                       |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.