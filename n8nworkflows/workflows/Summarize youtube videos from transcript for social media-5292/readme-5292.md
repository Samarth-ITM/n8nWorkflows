Summarize youtube videos from transcript for social media

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-from-transcript-for-social-media-5292


# Summarize youtube videos from transcript for social media

### 1. Workflow Overview

This workflow automates the summarization of YouTube video transcripts for social media use. It is designed to accept a YouTube video ID via a form submission, retrieve the full transcript using an external transcription API, process and format the transcript, generate a concise summary with an AI language model, optimize the output format, and finally update a Google Docs document with the summary. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Receives YouTube video ID via a user form.
- **1.2 Transcript Retrieval:** Calls an external API to fetch the transcript of the specified video.
- **1.3 Transcript Formatting:** Processes and validates the raw transcript data.
- **1.4 AI Processing:** Uses a Google Gemini language model to summarize the transcript text.
- **1.5 Output Optimization:** Extracts and cleans the AI-generated summary from the raw response.
- **1.6 Document Update:** Updates a Google Docs document with the formatted summary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures the YouTube video ID from a submitted form to trigger the workflow.
- **Nodes Involved:**  
  - On form submission  
  - Mapper

##### Node Details

**On form submission**  
- Type: Form Trigger  
- Role: Entry point that triggers the workflow when the user submits the form with a YouTube video ID.  
- Configuration: Form titled "summarize youtube videos from transcript for social media" with one required field labeled "Yt Video Id".  
- Input/Output: No input, outputs form data including "Yt Video Id".  
- Edge cases: Missing or invalid video ID will cause downstream errors or empty transcript.  
- Version: 2.2

**Mapper**  
- Type: Set  
- Role: Maps form field "Yt Video Id" to a workflow variable `ytVideoId` for easier access downstream.  
- Configuration: Assigns `ytVideoId` = value from form field "Yt Video Id".  
- Input: Output from form submission node.  
- Output: JSON with `ytVideoId`.  
- Edge cases: None significant; relies on prior validation from form.  
- Version: 3.4

---

#### 2.2 Transcript Retrieval

- **Overview:** Fetches the transcript of the YouTube video by sending the video ID to a third-party API.  
- **Nodes Involved:**  
  - Youtube Transcriptor

##### Node Details

**Youtube Transcriptor**  
- Type: HTTP Request  
- Role: POST request to "https://youtube-transcriptor-ai.p.rapidapi.com/yt/index.php" with `yt_video_id` to retrieve the transcript.  
- Configuration:  
  - Method: POST  
  - Content-Type: multipart-form-data  
  - Body parameter: `yt_video_id` = `ytVideoId` from Mapper  
  - Headers: Requires `x-rapidapi-host` and `x-rapidapi-key` for authentication (RapidAPI).  
- Input: JSON with `ytVideoId`  
- Output: JSON containing transcript data in `tracks[0].transcript` array.  
- Edge cases: API failure, invalid video ID, no transcript available, network errors, or authentication errors.  
- Version: 4.2

---

#### 2.3 Transcript Formatting

- **Overview:** Validates and concatenates transcript lines into a single text string for AI processing.  
- **Nodes Involved:**  
  - Formator

##### Node Details

**Formator**  
- Type: Code (JavaScript)  
- Role: Extracts transcript array from API response, validates it, and concatenates all text lines into one string.  
- Configuration (logic):  
  - Checks if `tracks[0].transcript` exists and is an array with content.  
  - If invalid or empty, outputs a message: "Transcript not available or invalid video ID."  
  - Otherwise, joins all transcript segments into a single string under `chatInput`.  
- Input: Raw transcript JSON from Youtube Transcriptor  
- Output: JSON with `chatInput` text for AI input.  
- Edge cases: Missing transcript, empty transcript, malformed API response.  
- Version: 2

---

#### 2.4 AI Processing

- **Overview:** Summarizes the full transcript text using a Google Gemini language model through an AI Agent node.  
- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model

##### Node Details

**Google Gemini Chat Model**  
- Type: LangChain LM Chat Google Gemini  
- Role: Provides the AI language model instance used by the AI agent to generate the summary.  
- Configuration:  
  - Model: "models/gemini-2.0-flash"  
  - Credential: Google PaLM API account with OAuth2 credentials.  
- Input: Receives prompt from AI Agent’s system message and transcript text.  
- Output: AI-generated raw text output containing the summary.  
- Edge cases: API quota limits, authentication failure, model unavailability, timeouts.  
- Version: 1

**AI Agent**  
- Type: LangChain Agent  
- Role: Orchestrates the interaction with the AI language model.  
- Configuration:  
  - System message instructs to summarize the transcript in natural language, presenting main points and tone.  
  - Output format includes a "Summary" section with bullet points and optional tone/style.  
- Input: `chatInput` from Formator  
- Output: Raw AI text containing the summarization.  
- Edge cases: AI model returns incomplete or malformed output.  
- Version: 2

---

#### 2.5 Output Optimization

- **Overview:** Parses the AI raw output to extract only the formatted summary for clean downstream use.  
- **Nodes Involved:**  
  - Optimizer

##### Node Details

**Optimizer**  
- Type: Code (JavaScript)  
- Role: Uses regex to find the section labeled "🎬 **Summary**:" and extracts that content up to the next "---" delimiter or end of string.  
- Configuration:  
  - Extracts summary bullet points and optional tone/style from AI output text.  
  - Returns a JSON with a clean `summary` field.  
- Input: AI Agent raw text output  
- Output: JSON with clean `summary` string  
- Edge cases: AI output missing "Summary" section, malformed text, regex failures.  
- Version: 2

---

#### 2.6 Document Update

- **Overview:** Inserts the cleaned summary text into a Google Docs document to store or share the results.  
- **Nodes Involved:**  
  - Google Docs

##### Node Details

**Google Docs**  
- Type: Google Docs node  
- Role: Updates an existing Google Docs document with the summary text.  
- Configuration:  
  - Operation: Update document  
  - Action: Insert the `summary` text into the document body.  
  - Authentication: Uses a Google Service Account credential.  
  - Document URL: (empty in JSON, must be set by user)  
- Input: Cleaned summary JSON from Optimizer  
- Output: Confirmation of document update operation  
- Edge cases: Incorrect or missing document URL, authentication failures, API rate limits.  
- Version: 2

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                    | Input Node(s)         | Output Node(s)         | Sticky Note                     |
|---------------------|----------------------------------|----------------------------------|-----------------------|------------------------|--------------------------------|
| On form submission   | Form Trigger                     | Entry point - receives video ID  | -                     | Mapper                 |                                |
| Mapper              | Set                              | Maps form input to variable      | On form submission     | Youtube Transcriptor    |                                |
| Youtube Transcriptor | HTTP Request                     | Retrieves transcript from API    | Mapper                | Formator               |                                |
| Formator            | Code                             | Formats transcript text          | Youtube Transcriptor   | AI Agent               |                                |
| AI Agent            | LangChain Agent                  | Generates summary via AI         | Formator              | Optimizer              |                                |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | Language model for AI Agent      | AI Agent (ai_languageModel) | AI Agent           |                                |
| Optimizer           | Code                             | Extracts summary from AI output  | AI Agent              | Google Docs            |                                |
| Google Docs         | Google Docs                      | Updates document with summary    | Optimizer             | -                      | Must set target Document URL   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `On form submission`  
   - Set Form Title: "summarize youtube videos from transcript for social media"  
   - Add one required field: Label "Yt Video Id", placeholder "abc123"  
   - This node triggers when the form is submitted.

2. **Add a Set Node**  
   - Name: `Mapper`  
   - Map the form field "Yt Video Id" to a variable `ytVideoId` for use downstream.

3. **Add an HTTP Request Node**  
   - Name: `Youtube Transcriptor`  
   - Method: POST  
   - URL: `https://youtube-transcriptor-ai.p.rapidapi.com/yt/index.php`  
   - Content-Type: multipart/form-data  
   - Body Parameter: `yt_video_id` = `={{ $json.ytVideoId }}`  
   - Add headers for `x-rapidapi-host` and `x-rapidapi-key` (set with RapidAPI credentials)  
   - This node fetches the transcript JSON for the video.

4. **Add a Code Node**  
   - Name: `Formator`  
   - JavaScript code to:  
     - Extract `tracks[0].transcript` array from input  
     - If missing or empty, output message "Transcript not available or invalid video ID."  
     - Else join `text` fields of transcript segments into one string as `chatInput`.

5. **Add a LangChain Agent Node**  
   - Name: `AI Agent`  
   - Configure the system message to instruct summarization with the format:  
     ```
     You are a helpful assistant that summarizes YouTube video transcripts.

     Here is the full transcript of a video:
     Please provide a concise summary of the video in natural language, covering the main points, topics, and tone.

     Format the output as:

     ---
     🎬 **Summary**:
     - [Main idea 1]
     - [Main idea 2]
     - [Optional tone, style, or genre]
     ---
     ```
   - Connect to a Google Gemini Chat Model node for AI completion.

6. **Add a LangChain LM Chat Google Gemini Node**  
   - Name: `Google Gemini Chat Model`  
   - Model Name: `models/gemini-2.0-flash`  
   - Set credentials with Google PaLM API OAuth2 account.

7. **Add a Code Node**  
   - Name: `Optimizer`  
   - JavaScript code to:  
     - Extract the "🎬 **Summary**:" section from AI Agent output using regex (stop at `---`)  
     - Return a JSON field `summary` with the extracted clean text.

8. **Add a Google Docs Node**  
   - Name: `Google Docs`  
   - Operation: Update Document  
   - Action: Insert text  
   - Text to insert: `={{ $json.summary }}`  
   - Authentication: Google Service Account credential  
   - Set the Document URL to the target Google Docs file URL.

9. **Connect the nodes in the following order:**  
   - On form submission → Mapper → Youtube Transcriptor → Formator → AI Agent → Optimizer → Google Docs  
   - Connect Google Gemini Chat Model as the language model for AI Agent.

10. **Set up credentials:**  
    - RapidAPI credentials for Youtube Transcriptor (x-rapidapi-host and x-rapidapi-key)  
    - Google PaLM API credentials for Google Gemini Chat Model node  
    - Google Service Account credentials for Google Docs node

11. **Test the workflow:**  
    - Submit the form with a YouTube video ID  
    - Verify transcript retrieval, summarization, and document update.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                  |
|------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow tag is "Digital Marketing" indicating its use case focus.       | Tag metadata                                     |
| The Google Docs node requires setting the target document URL manually.      | Google Docs node configuration                    |
| System message in AI Agent defines output format strictly for reliable parsing.| AI prompt design best practice                    |
| RapidAPI service used for YouTube transcript retrieval may require subscription.| https://rapidapi.com/                             |
| Google PaLM API (Gemini) credentials must be set up with OAuth2 for access.  | https://developers.generativeai.google/          |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.