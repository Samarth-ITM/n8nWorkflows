🎥 Analyze YouTube Video for Summaries, Transcripts & Content + Google Gemini AI

https://n8nworkflows.xyz/workflows/---analyze-youtube-video-for-summaries--transcripts---content---google-gemini-ai-3289


# 🎥 Analyze YouTube Video for Summaries, Transcripts & Content + Google Gemini AI

### 1. Workflow Overview

This workflow automates the extraction and AI-powered analysis of YouTube video content, targeting content creators, marketers, and researchers who want to generate actionable insights, transcripts, summaries, and metadata efficiently. It leverages Google Gemini AI to analyze video content and dynamically generates outputs tailored to different prompt types.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Configuration:** Receives user input (YouTube Video ID and prompt type), sets configuration variables including API keys and URLs.
- **1.2 YouTube Video Data Retrieval:** Constructs YouTube Data API URL and fetches detailed video metadata.
- **1.3 Audience Metadata Extraction:** Uses Google Gemini AI to analyze the video and extract structured audience-specific metadata.
- **1.4 Prompt Composition and Selection:** Composes multiple prompt templates for different output types and selects the appropriate prompt based on user input.
- **1.5 AI Content Generation:** Sends the selected prompt and video URL to Google Gemini AI to generate the requested content (summary, transcription, timestamps, etc.).
- **1.6 Output Processing and Delivery:** Converts AI output from markdown to HTML, saves results to Google Drive, sends via Gmail, and/or provides output in a completion form.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Configuration

**Overview:**  
This block captures user inputs from a form trigger and sets essential workflow variables including API keys, YouTube URLs, and prompt types.

**Nodes Involved:**  
- Start Workflow (Form Trigger)  
- Config (Set)  
- Sticky Note1 (Informational)

**Node Details:**  

- **Start Workflow**  
  - Type: Form Trigger  
  - Role: Entry point; collects YouTube Video ID and prompt type from user via dropdown and text input.  
  - Configuration: Form fields for "Prompt Type" (dropdown with options: default, transcribe, timestamps, summary, scene, clips) and "YouTube Video Id" (text).  
  - Outputs: JSON with user inputs.  
  - Edge Cases: Missing or invalid inputs; no validation beyond required fields.

- **Config**  
  - Type: Set  
  - Role: Defines workflow variables for downstream use.  
  - Configuration:  
    - `google_api_key` from environment variable `GOOGLE_API_KEY`  
    - `youtube_url` constructed as `https://www.youtube.com/watch?v={{ YouTube Video Id }}`  
    - `prompt_type` from user input  
    - `video_id` from user input  
  - Inputs: From Start Workflow  
  - Outputs: JSON with config variables for other nodes.  
  - Edge Cases: Missing environment variable or malformed video ID.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Provides user instruction and example input.  
  - Content: "👍Try Me! YouTube Video Id: wBuULAoJxok"

---

#### 2.2 YouTube Video Data Retrieval

**Overview:**  
Constructs the YouTube Data API URL dynamically and fetches detailed video metadata including snippet, statistics, and topic details.

**Nodes Involved:**  
- Create YouTube API URL (Code)  
- Get YouTube Video Details (HTTP Request)  
- Sticky Note16, Sticky Note17 (Informational)

**Node Details:**  

- **Create YouTube API URL**  
  - Type: Code  
  - Role: Builds YouTube API URL using video ID and Google API key.  
  - Configuration:  
    - Validates presence of video ID and API key.  
    - Constructs URL with parts: snippet, contentDetails, status, statistics, player, topicDetails.  
  - Inputs: From Config node JSON.  
  - Outputs: JSON with `youtubeUrl` property.  
  - Edge Cases: Throws error if video ID or API key missing.

- **Get YouTube Video Details**  
  - Type: HTTP Request  
  - Role: Calls YouTube Data API to retrieve video details.  
  - Configuration: Uses URL from previous node.  
  - Inputs: From Create YouTube API URL.  
  - Outputs: JSON with video metadata.  
  - Edge Cases: API quota exceeded, invalid video ID, network errors.

- **Sticky Note16 & Sticky Note17**  
  - Type: Sticky Note  
  - Role: Document the purpose of this block.

---

#### 2.3 Audience Metadata Extraction

**Overview:**  
Uses Google Gemini AI to analyze the YouTube video and extract structured audience-specific metadata as a JSON object.

**Nodes Involved:**  
- Define Audience Meta Prompt (Set)  
- Get Video Audience MetaData (HTTP Request)  
- Extract MetaData Object (Set)  
- Sticky Note4, Sticky Note7, Sticky Note8 (Informational)

**Node Details:**  

- **Define Audience Meta Prompt**  
  - Type: Set  
  - Role: Defines a meta prompt instructing the AI to analyze the video and return a JSON object with fields like video_type, primary_audience, key_topics, etc.  
  - Configuration: Detailed prompt text emphasizing objective analysis and JSON-only response without markdown or explanation.  
  - Inputs: From Config node.  
  - Outputs: JSON with `meta_prompt`.  
  - Edge Cases: Prompt formatting errors.

- **Get Video Audience MetaData**  
  - Type: HTTP Request  
  - Role: Sends meta prompt and YouTube URL to Google Gemini API to generate audience metadata.  
  - Configuration:  
    - POST request to Gemini model `gemini-1.5-flash` endpoint with API key.  
    - JSON body includes `meta_prompt` and `file_uri` (YouTube URL).  
    - Temperature 0.2, topP 0.8, topK 40, maxOutputTokens 2048.  
  - Inputs: From Define Audience Meta Prompt.  
  - Outputs: AI response with metadata candidates.  
  - Edge Cases: API errors, invalid API key, malformed response.

- **Extract MetaData Object**  
  - Type: Set  
  - Role: Cleans AI response text by removing markdown code blocks and extracts JSON string into an object under `text`.  
  - Inputs: From Get Video Audience MetaData.  
  - Outputs: JSON object with parsed metadata.  
  - Edge Cases: Parsing errors if AI response is malformed.

- **Sticky Notes**  
  - Provide context and documentation for this block.

---

#### 2.4 Prompt Composition and Selection

**Overview:**  
Composes multiple prompt templates for different output types and selects the appropriate prompt and model based on user-selected prompt type.

**Nodes Involved:**  
- Compose Prompts (Set)  
- Get Prompt by Prompt Type (Code)  
- Sticky Note2, Sticky Note9, Sticky Note10 (Informational)

**Node Details:**  

- **Compose Prompts**  
  - Type: Set  
  - Role: Defines XML-like structured prompts for six prompt types: default, transcribe, timestamps, summary, scene, clips.  
  - Configuration: Each prompt includes a `<prompt>` and `<model>` tag with instructions tailored to the prompt type, referencing metadata fields dynamically.  
  - Inputs: From Extract MetaData Object (metadata fields).  
  - Outputs: Single string with all prompt templates.  
  - Edge Cases: Expression errors if metadata fields missing.

- **Get Prompt by Prompt Type**  
  - Type: Code  
  - Role: Parses the XML-like prompt string to extract the prompt and model corresponding to the selected prompt type from Config.  
  - Configuration: Uses regex to extract `<prompt>` and `<model>` content for the tag matching `prompt_type`.  
  - Inputs: From Compose Prompts and Config.  
  - Outputs: JSON with `prompt` and `model` strings.  
  - Edge Cases: Regex failures if prompt_type invalid or missing tags.

- **Sticky Notes**  
  - Explain prompt options and block purpose.

---

#### 2.5 AI Content Generation

**Overview:**  
Sends the selected prompt and YouTube URL to Google Gemini AI to generate the requested content such as summaries, transcripts, or clips.

**Nodes Involved:**  
- Get YouTube Information by Prompt Type (HTTP Request)  
- Sticky Note5, Sticky Note6, Sticky Note3 (Informational)

**Node Details:**  

- **Get YouTube Information by Prompt Type**  
  - Type: HTTP Request  
  - Role: Calls Google Gemini API with the selected prompt and model to generate content.  
  - Configuration:  
    - POST request to Gemini endpoint for the selected model.  
    - JSON body includes prompt text and YouTube URL as file data.  
    - Content-Type: application/json.  
  - Inputs: From Get Prompt by Prompt Type.  
  - Outputs: AI-generated content in markdown format.  
  - Edge Cases: API errors, invalid model name, rate limits.

- **Sticky Notes**  
  - Document Google Generative Language API usage.

---

#### 2.6 Output Processing and Delivery

**Overview:**  
Processes AI output by converting markdown to HTML, saving results to Google Drive, sending via Gmail, and/or providing output in a completion form.

**Nodes Involved:**  
- Convert Markdown to HTML (Markdown)  
- Save to Google Drive as Text File (Google Drive)  
- Send to Gmail as HTML (Gmail)  
- Provide YouTube Information to User as HTML (Form)  
- Merge (Merge)  
- Sticky Note11, Sticky Note12, Sticky Note13, Sticky Note14 (Informational)

**Node Details:**  

- **Convert Markdown to HTML**  
  - Type: Markdown  
  - Role: Converts AI-generated markdown content to HTML for email and form display.  
  - Inputs: From Get YouTube Information by Prompt Type.  
  - Outputs: HTML content.  
  - Edge Cases: Markdown syntax errors.

- **Save to Google Drive as Text File**  
  - Type: Google Drive  
  - Role: Saves a text file named with video ID and timestamp containing metadata and AI output.  
  - Configuration:  
    - Folder: Root of "My Drive"  
    - Content includes video ID, timestamp, key topics, content purpose, primary audience, AI output text, and video details JSON.  
  - Inputs: From Get YouTube Information by Prompt Type and Merge node.  
  - Credentials: Google Drive OAuth2.  
  - Edge Cases: Permission errors, quota limits.

- **Send to Gmail as HTML**  
  - Type: Gmail  
  - Role: Sends an email with video title, ID, thumbnail, and AI-generated content in HTML format.  
  - Configuration:  
    - Recipient: Environment variable `EMAIL_ADDRESS_JOE`  
    - Subject: Video ID and key topic  
    - Credentials: Gmail OAuth2.  
  - Inputs: From Convert Markdown to HTML and Merge node.  
  - Edge Cases: Authentication errors, email delivery failures.

- **Provide YouTube Information to User as HTML**  
  - Type: Form (Completion)  
  - Role: Provides AI output and video thumbnail as a completion response to the user form.  
  - Inputs: From Convert Markdown to HTML and Merge node.  
  - Edge Cases: Webhook failures.

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from video details and metadata extraction for use in saving and emailing.  
  - Inputs: From Get YouTube Video Details and Extract MetaData Object.  
  - Outputs: Combined JSON.

- **Sticky Notes**  
  - Document each output node’s purpose.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                  | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                                                                      |
|--------------------------------|---------------------|-------------------------------------------------|----------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Start Workflow                 | Form Trigger        | Entry point: collects YouTube ID and prompt type | None                             | Config                                      | ## 👍Try Me! YouTube Video Id: wBuULAoJxok                                                                      |
| Config                        | Set                 | Sets workflow variables including API keys and URLs | Start Workflow                   | Create YouTube API URL, Define Audience Meta Prompt | ## Set Workflow Config Variables                                                                                 |
| Create YouTube API URL         | Code                | Constructs YouTube Data API URL                   | Config                           | Get YouTube Video Details                    | ## Create YouTube API URL                                                                                        |
| Get YouTube Video Details      | HTTP Request        | Fetches detailed video metadata from YouTube API | Create YouTube API URL           | Merge                                        | ## Get YouTube Video Details                                                                                     |
| Define Audience Meta Prompt    | Set                 | Defines AI prompt to extract audience metadata    | Config                           | Get Video Audience MetaData                   | ## Define Audience Meta Prompt                                                                                   |
| Get Video Audience MetaData    | HTTP Request        | Calls Google Gemini AI to extract audience metadata | Define Audience Meta Prompt      | Extract MetaData Object                       | ## Analyze YouTube Video for Audience MetaData                                                                   |
| Extract MetaData Object        | Set                 | Cleans and parses AI metadata response             | Get Video Audience MetaData      | Merge                                        | ## Extract MetaData Object                                                                                        |
| Compose Prompts               | Set                 | Defines prompt templates for all prompt types      | Extract MetaData Object          | Get Prompt by Prompt Type                     | ## Compose the Prompts with Audience MetaData                                                                    |
| Get Prompt by Prompt Type      | Code                | Selects prompt and model based on user prompt type | Compose Prompts, Config          | Get YouTube Information by Prompt Type        | ## Get Prompt by Prompt Type                                                                                      |
| Get YouTube Information by Prompt Type | HTTP Request | Calls Google Gemini AI with selected prompt to generate content | Get Prompt by Prompt Type        | Convert Markdown to HTML, Save to Google Drive as Text File | ## Google Generative Language API                                                                                 |
| Convert Markdown to HTML       | Markdown            | Converts AI markdown output to HTML                 | Get YouTube Information by Prompt Type | Send to Gmail as HTML, Provide YouTube Information to User as HTML | ## Convert Markdown to HTML                                                                                        |
| Save to Google Drive as Text File | Google Drive     | Saves AI output and metadata as text file           | Get YouTube Information by Prompt Type, Merge | None                                         | ## Save YouTube Information to Google Drive                                                                       |
| Send to Gmail as HTML          | Gmail               | Sends AI output and video info via email            | Convert Markdown to HTML, Merge  | None                                         | ## Email YouTube Information                                                                                      |
| Provide YouTube Information to User as HTML | Form          | Provides AI output and thumbnail as form completion | Convert Markdown to HTML, Merge  | None                                         | ## Provide YouTube Information in Completion Form                                                                 |
| Merge                         | Merge               | Combines video details and metadata for output nodes | Get YouTube Video Details, Extract MetaData Object | Compose Prompts, Save to Google Drive as Text File, Send to Gmail as HTML, Provide YouTube Information to User as HTML |                                                                                                                  |
| Sticky Note(s)                | Sticky Note         | Documentation and instructions                      | Various                         | Various                                      | See individual notes above                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Start Workflow")**  
   - Type: Form Trigger  
   - Configure form fields:  
     - Dropdown "Prompt Type" with options: default, transcribe, timestamps, summary, scene, clips (required)  
     - Text input "YouTube Video Id" (required)  
   - Set response mode to "lastNode" with description explaining the workflow purpose.

2. **Create Set Node ("Config")**  
   - Type: Set  
   - Assign variables:  
     - `google_api_key` = `{{$env.GOOGLE_API_KEY}}` (environment variable)  
     - `youtube_url` = `"https://www.youtube.com/watch?v=" + $json["YouTube Video Id"]`  
     - `prompt_type` = `$json["Prompt Type"]`  
     - `video_id` = `$json["YouTube Video Id"]`  
   - Connect input from "Start Workflow".

3. **Create Code Node ("Create YouTube API URL")**  
   - Type: Code  
   - JavaScript code:  
     - Validate `video_id` and `google_api_key` presence.  
     - Construct YouTube Data API URL with parts: snippet, contentDetails, status, statistics, player, topicDetails.  
     - Output JSON with `youtubeUrl`.  
   - Connect input from "Config".

4. **Create HTTP Request Node ("Get YouTube Video Details")**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `{{$json.youtubeUrl}}`  
   - Connect input from "Create YouTube API URL".

5. **Create Set Node ("Define Audience Meta Prompt")**  
   - Type: Set  
   - Assign `meta_prompt` string with detailed instructions to extract audience metadata as JSON (see overview for prompt content).  
   - Connect input from "Config".

6. **Create HTTP Request Node ("Get Video Audience MetaData")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key={{$json.google_api_key}}`  
   - Headers: Content-Type: application/json  
   - Body (JSON): includes `meta_prompt` and `file_uri` with YouTube URL, generationConfig with temperature 0.2, topP 0.8, topK 40, maxOutputTokens 2048, model "gemini-1.5-flash".  
   - Connect input from "Define Audience Meta Prompt".

7. **Create Set Node ("Extract MetaData Object")**  
   - Type: Set  
   - Assign `text` by cleaning AI response text from `candidates[0].content.parts[0].text` removing markdown code blocks.  
   - Connect input from "Get Video Audience MetaData".

8. **Create Merge Node ("Merge")**  
   - Type: Merge  
   - Mode: Combine by position  
   - Connect inputs from "Get YouTube Video Details" (main input 0) and "Extract MetaData Object" (main input 1).

9. **Create Set Node ("Compose Prompts")**  
   - Type: Set  
   - Assign a single string containing XML-like prompt templates for all six prompt types (default, transcribe, timestamps, summary, scene, clips) referencing metadata fields dynamically.  
   - Connect input from "Extract MetaData Object".

10. **Create Code Node ("Get Prompt by Prompt Type")**  
    - Type: Code  
    - JavaScript code to parse the XML-like prompt string and extract the `<prompt>` and `<model>` tags matching the `prompt_type` from "Config".  
    - Connect input from "Compose Prompts".

11. **Create HTTP Request Node ("Get YouTube Information by Prompt Type")**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/{{$json.model}}:generateContent?key={{$node["Config"].json.google_api_key}}`  
    - Headers: Content-Type: application/json  
    - Body (JSON): includes prompt text and YouTube URL as file data.  
    - Connect input from "Get Prompt by Prompt Type".

12. **Create Markdown Node ("Convert Markdown to HTML")**  
    - Type: Markdown  
    - Mode: markdownToHtml  
    - Input: AI-generated markdown content from previous node.  
    - Connect input from "Get YouTube Information by Prompt Type".

13. **Create Google Drive Node ("Save to Google Drive as Text File")**  
    - Type: Google Drive  
    - Operation: createFromText  
    - Name: `{{$node["Start Workflow"].json["YouTube Video Id"]}} - {{$now}}`  
    - Content: includes video ID, timestamp, key topics, content purpose, primary audience, AI output text, and video details JSON from "Merge".  
    - Folder: Root of "My Drive"  
    - Credentials: Google Drive OAuth2  
    - Connect input from "Get YouTube Information by Prompt Type" and "Merge".

14. **Create Gmail Node ("Send to Gmail as HTML")**  
    - Type: Gmail  
    - Send To: environment variable `EMAIL_ADDRESS_JOE`  
    - Subject: video ID and key topic from metadata  
    - Message: HTML including video title, ID, thumbnail, and AI output HTML  
    - Credentials: Gmail OAuth2  
    - Connect input from "Convert Markdown to HTML" and "Merge".

15. **Create Form Node ("Provide YouTube Information to User as HTML")**  
    - Type: Form (Completion)  
    - Respond With: showText  
    - Response Text: HTML including video thumbnail and AI output HTML  
    - Connect input from "Convert Markdown to HTML" and "Merge".

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow title: 🎥 Analyze YouTube Video for Summaries, Transcripts & Content + Google Gemini AI | Workflow purpose and branding                                                                    |
| Prompt options explained in Sticky Note2                                                        | Details on six prompt types: default, transcribe, timestamps, summary, scene, clips              |
| Google Generative Language API usage notes in Sticky Note3, Sticky Note6                         | Reference to Google Gemini AI API integration                                                   |
| Example YouTube Video ID for testing: wBuULAoJxok                                               | Provided in Sticky Note1                                                                         |
| Instructions for API key setup: Add Google API key as environment variable `GOOGLE_API_KEY`      | Setup requirement                                                                                |
| Gmail recipient email address set via environment variable `EMAIL_ADDRESS_JOE`                   | Email sending configuration                                                                     |
| Google Drive and Gmail OAuth2 credentials required                                              | Integration credentials setup                                                                    |
| Workflow designed for content creators, marketers, and researchers                              | Target audience                                                                                  |
| Workflow automates metadata extraction, AI content generation, and multi-channel output delivery | Use case summary                                                                                |

---

This documentation provides a complete, structured reference for understanding, reproducing, and modifying the workflow. It anticipates potential errors such as missing API keys, invalid inputs, API rate limits, and parsing failures. Credentials and environment variables must be configured correctly for successful execution. The modular design allows easy extension to other platforms or prompt types.