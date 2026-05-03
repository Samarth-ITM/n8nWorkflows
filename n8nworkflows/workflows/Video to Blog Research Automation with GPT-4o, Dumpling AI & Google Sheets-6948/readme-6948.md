Video to Blog Research Automation with GPT-4o, Dumpling AI & Google Sheets

https://n8nworkflows.xyz/workflows/video-to-blog-research-automation-with-gpt-4o--dumpling-ai---google-sheets-6948


# Video to Blog Research Automation with GPT-4o, Dumpling AI & Google Sheets

---

### 1. Workflow Overview

This n8n workflow automates the process of turning uploaded videos into AI-generated blog research content using Dumpling AI, OpenAI's GPT-4o, and Google Sheets. It is designed for content creators, marketers, or researchers who want to automate transcription, keyword extraction, competitor research, and structured data storage from video content.

The workflow is logically divided into the following blocks:

- **1.1 Video Upload Monitoring and Download:** Watches a specific Google Drive folder for newly uploaded videos and downloads them.
- **1.2 Video Processing and Transcription:** Converts the downloaded video to Base64 format and sends it to Dumpling AI for detailed transcription with timestamps and speaker identification.
- **1.3 Keyword Extraction:** Uses OpenAI GPT-4o to extract exactly five high-impact SEO-relevant keywords from the transcript.
- **1.4 Competitor Research:** Passes extracted keywords to a Dumpling AI agent which performs research via Perplexity and Google Search APIs to gather relevant blog posts and topic summaries.
- **1.5 Results Formatting and Storage:** Formats the research results and appends them into a Google Sheets document for review and further use.

The workflow includes a sticky note summarizing its purpose and steps for clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Video Upload Monitoring and Download

- **Overview:**  
  This block monitors a specific Google Drive folder for new video files and downloads any newly created video files for further processing.

- **Nodes Involved:**  
  - Watch Uploaded Videos  
  - Download Video

- **Node Details:**

  - **Watch Uploaded Videos**  
    - Type: Google Drive Trigger  
    - Role: Watches for newly created files in a designated Google Drive folder (specified by folder ID).  
    - Configuration: Polls every minute; triggers only on new files created in the folder with ID `1qh0vXJpLpoeXsfe0OFlbClBwDYqApn1A` (named "Video").  
    - Inputs: None (trigger node).  
    - Outputs: Metadata of the new file (including file ID, name, MIME type).  
    - Edge Cases: Permissions errors if OAuth token is invalid or insufficient; possible delay if Google Drive API rate limits; only triggers on new files, not on modifications.  
    - Credentials: Google Drive OAuth2.

  - **Download Video**  
    - Type: Google Drive  
    - Role: Downloads the video file identified by the file ID from the trigger node.  
    - Configuration: Uses the file ID from the trigger node to download the file content as binary data.  
    - Inputs: File metadata from "Watch Uploaded Videos".  
    - Outputs: Binary data of the video file.  
    - Edge Cases: Download failures due to network issues or permissions; handling large files may cause timeouts or memory issues.  
    - Credentials: Google Drive OAuth2.

#### 2.2 Video Processing and Transcription

- **Overview:**  
  Converts the downloaded video binary to a Base64 string and sends it to Dumpling AI's transcription API to extract a detailed transcript including timestamps and speaker identification.

- **Nodes Involved:**  
  - Convert Video to Base64  
  - Transcribe with Dumpling AI

- **Node Details:**

  - **Convert Video to Base64**  
    - Type: Extract From File  
    - Role: Converts the binary video file into a Base64-encoded string property (`data`) for HTTP transmission.  
    - Configuration: Operation set to `binaryToProperty`, extracting the binary data into JSON property `data`.  
    - Inputs: Binary video file from "Download Video".  
    - Outputs: JSON data with Base64-encoded video string.  
    - Edge Cases: Large video files may cause performance degradation; encoding failures if binary data is corrupted.  

  - **Transcribe with Dumpling AI**  
    - Type: HTTP Request  
    - Role: Sends the Base64 video to Dumpling AI API for transcription.  
    - Configuration: POST request to `https://app.dumplingai.com/api/v1/extract-video` with JSON body containing the Base64 video and a prompt to transcribe word-for-word including timestamps and speaker identification.  
    - Inputs: Base64 video string from "Convert Video to Base64" node.  
    - Outputs: JSON transcription result under `results`.  
    - Authentication: HTTP Header Auth with Dumpling AI credentials.  
    - Edge Cases: API rate limits, authentication failures, large payload rejection, malformed JSON responses, transcription errors if video quality is poor.

#### 2.3 Keyword Extraction

- **Overview:**  
  Sends the full transcript received from Dumpling AI to OpenAI GPT-4o to generate exactly five SEO-focused keywords or phrases representing the main topics.

- **Nodes Involved:**  
  - Extract Keywords with OpenAI

- **Node Details:**

  - **Extract Keywords with OpenAI**  
    - Type: OpenAI (LangChain Node)  
    - Role: Uses GPT-4o to analyze the transcript and return five comma-separated keywords/phrases.  
    - Configuration:  
      - Model: `chatgpt-4o-latest`  
      - Messages:  
        - System prompt instructs to extract exactly five high-impact keywords relevant for SEO, content tagging, or summarization.  
        - User message contains the transcript from Dumpling AI transcription (`{{$json.results}}`).  
    - Inputs: JSON with transcription results from "Transcribe with Dumpling AI".  
    - Outputs: JSON with the message containing the keywords string.  
    - Credentials: OpenAI API key.  
    - Edge Cases: API rate limits, malformed transcript input, potential incomplete or off-topic keywords, response formatting errors.

#### 2.4 Competitor Research

- **Overview:**  
  Passes the extracted keywords to a Dumpling AI agent which performs competitor research by querying Perplexity and Google Search APIs, returning curated blog posts and topic summaries.

- **Nodes Involved:**  
  - Run Competitor Research via Dumpling AI

- **Node Details:**

  - **Run Competitor Research via Dumpling AI**  
    - Type: HTTP Request  
    - Role: Sends a prompt containing the extracted keywords to a Dumpling AI agent with agent ID `79e8a289-8e4a-46f5-8cc2-abeceb8ae9e4` to generate competitor research data.  
    - Configuration:  
      - POST to `https://app.dumplingai.com/api/v1/agents/generate-completion`  
      - JSON body includes the user message with keywords and agent ID.  
      - Parses JSON response automatically.  
    - Inputs: Keywords message content from "Extract Keywords with OpenAI".  
    - Outputs: JSON containing research results categorized as `topicsFromPerplexity` and `blogPostsFromGoogle`.  
    - Authentication: HTTP Header Auth with Dumpling AI credentials.  
    - Edge Cases: API limits, malformed input causing errors, JSON parsing issues, empty or irrelevant research results.

#### 2.5 Results Formatting and Storage

- **Overview:**  
  Formats the competitor research results into readable string summaries and appends them alongside keywords into a Google Sheets document for storage and easy access.

- **Nodes Involved:**  
  - Format Results for Google Sheets  
  - Append to Google Sheets

- **Node Details:**

  - **Format Results for Google Sheets**  
    - Type: Code (JavaScript)  
    - Role: Converts research results arrays (`topicsFromPerplexity` and `blogPostsFromGoogle`) into formatted multiline strings for spreadsheet storage.  
    - Configuration: Custom JS code that maps each item to strings with titles, summaries, notes, and URLs, then joins them with double line breaks.  
    - Inputs: JSON research results from "Run Competitor Research via Dumpling AI".  
    - Outputs: JSON with two string fields: `perplexityOutput`, `googleSearchOutput`.  
    - Edge Cases: Empty or missing data arrays, malformed input JSON.

  - **Append to Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends a new row with columns for Keywords, blog posts, and topics to a specified Google Sheets document and sheet.  
    - Configuration:  
      - Document ID and Sheet name specified via linked Google Sheets.  
      - Columns mapped to:  
        - Keywords: extracted keywords string  
        - blogPostsFromGoogle: formatted Google search blog posts string  
        - topicsFromPerplexity: formatted Perplexity topics string  
      - Append operation mode.  
    - Inputs: JSON from "Format Results for Google Sheets" and keywords from "Extract Keywords with OpenAI".  
    - Outputs: Confirmation of appended row.  
    - Credentials: Google Sheets OAuth2.  
    - Edge Cases: Permission errors, quota limits, invalid sheet or document IDs.

#### 2.6 Sticky Note

- **Overview:**  
  Provides a detailed summary and documentation of the workflow's purpose and steps within the n8n editor for user reference.

- **Node Involved:**  
  - Sticky Note

- **Node Details:**

  - Type: Sticky Note  
  - Content: Describes the overall workflow, its trigger, and stepwise operations clearly.  
  - Inputs/Outputs: None  
  - Position: Visually placed for user guidance.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                                     | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                                  |
|--------------------------------|----------------------------------|----------------------------------------------------|-------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Watch Uploaded Videos           | Google Drive Trigger              | Detect new video uploads in specified Google Drive folder | None                    | Download Video                  | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Download Video                 | Google Drive                     | Download the newly uploaded video file              | Watch Uploaded Videos    | Convert Video to Base64          | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Convert Video to Base64         | Extract From File                | Convert downloaded video binary to Base64 string    | Download Video           | Transcribe with Dumpling AI     | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Transcribe with Dumpling AI     | HTTP Request                    | Send Base64 video to Dumpling AI to get transcript  | Convert Video to Base64  | Extract Keywords with OpenAI     | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Extract Keywords with OpenAI    | OpenAI (LangChain Node)          | Extract 5 SEO keywords from transcript               | Transcribe with Dumpling AI | Run Competitor Research via Dumpling AI | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Run Competitor Research via Dumpling AI | HTTP Request              | Use Dumpling AI agent to perform competitor research | Extract Keywords with OpenAI | Format Results for Google Sheets | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Format Results for Google Sheets | Code (JavaScript)               | Format research results into strings for spreadsheet | Run Competitor Research via Dumpling AI | Append to Google Sheets          | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Append to Google Sheets         | Google Sheets                   | Append formatted data and keywords as new row       | Format Results for Google Sheets, Extract Keywords with OpenAI | None                            | ## 🎬 AI Video to Blog Research Automation: Workflow triggers on video upload, then downloads the video.     |
| Sticky Note                    | Sticky Note                     | Workflow description and documentation               | None                    | None                            | ## 🎬 AI Video to Blog Research Automation: This workflow triggers when a video is uploaded to a specific Google Drive folder. It performs downloading, transcription with Dumpling AI, keyword extraction with GPT-4o, competitor research with Dumpling AI, and saves results to Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node ("Watch Uploaded Videos"):**  
   - Node Type: Google Drive Trigger  
   - Event: `fileCreated`  
   - Poll interval: every minute  
   - Folder to watch: Folder ID `1qh0vXJpLpoeXsfe0OFlbClBwDYqApn1A` (your designated video folder)  
   - Credential: Google Drive OAuth2 with appropriate permissions.

2. **Add Google Drive Node ("Download Video"):**  
   - Node Type: Google Drive  
   - Operation: `Download`  
   - File ID: Use expression to get ID from trigger: `={{ $json["id"] }}`  
   - Credential: Same Google Drive OAuth2.

3. **Add Extract From File Node ("Convert Video to Base64"):**  
   - Node Type: Extract From File  
   - Operation: `binaryToProperty`  
   - Input: Use binary data from "Download Video" node  
   - Output Property: Default (`data`).

4. **Add HTTP Request Node ("Transcribe with Dumpling AI"):**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/extract-video`  
   - Authentication: HTTP Header Auth with Dumpling AI API key  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "inputMethod": "base64",
       "video": "{{$json.data}}",
       "prompt": "extract everything spoken in this video word for word. Include timestamps for each sentence, identify different speakers if possible, and return only the full transcription without any explanations or extra formatting."
     }
     ```
   - Send Body: true.

5. **Add OpenAI Node ("Extract Keywords with OpenAI"):**  
   - Node Type: OpenAI (LangChain)  
   - Model: `chatgpt-4o-latest`  
   - Credential: OpenAI API key  
   - Messages:  
     - System role: Instruct to extract exactly five SEO-relevant keywords, comma-separated, no extra formatting  
     - User role: Provide transcript from previous node: `=Here is the transcript: {{ $json.results }}`

6. **Add HTTP Request Node ("Run Competitor Research via Dumpling AI"):**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/agents/generate-completion`  
   - Authentication: HTTP Header Auth with Dumpling AI API key  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "messages": [
         {
           "role": "user",
           "content": "{{ $json.message.content }}"
         }
       ],
       "agentId": "79e8a289-8e4a-46f5-8cc2-abeceb8ae9e4",
       "parseJson": true
     }
     ```
   - Send Body: true.

7. **Add Code Node ("Format Results for Google Sheets"):**  
   - Node Type: Code (JavaScript)  
   - Input: Use JSON from previous node  
   - Code:  
     ```javascript
     return items.map(item => {
       const perplexity = item.json.parsedJson?.topicsFromPerplexity || [];
       const google = item.json.parsedJson?.blogPostsFromGoogle || [];

       const formattedPerplexity = perplexity.map(p => {
         return `Title: ${p.title}\nSummary: ${p.summary}\nNotes: ${p.notes}`;
       }).join('\n\n');

       const formattedGoogle = google.map(g => {
         return `Title: ${g.title}\nURL: ${g.url}\nSummary: ${g.summary}`;
       }).join('\n\n');

       return {
         json: {
           perplexityOutput: formattedPerplexity,
           googleSearchOutput: formattedGoogle
         }
       };
     });
     ```

8. **Add Google Sheets Node ("Append to Google Sheets"):**  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your target Google Sheets document ID (e.g., `10J1qtgITl76XcuNqUdv5lWcPRJt5sfuhHnYE6L2jsjo`)  
   - Sheet Name: e.g., `Sheet1`  
   - Columns to append:  
     - Keywords: Expression: `={{ $('Extract Keywords with OpenAI').item.json.message.content }}`  
     - topicsFromPerplexity: Expression: `={{ $json.perplexityOutput }}`  
     - blogPostsFromGoogle: Expression: `={{ $json.googleSearchOutput }}`  
   - Credential: Google Sheets OAuth2.

9. **Connect nodes in this order:**  
   - Watch Uploaded Videos → Download Video → Convert Video to Base64 → Transcribe with Dumpling AI → Extract Keywords with OpenAI → Run Competitor Research via Dumpling AI → Format Results for Google Sheets → Append to Google Sheets.

10. **Add Sticky Note (optional):**  
    - Create a Sticky Note node with the workflow summary for documentation within the editor.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow uses Dumpling AI for both transcription and competitor research. Ensure your API credentials are valid and have sufficient quota. | Dumpling AI API docs (not publicly linked in workflow) |
| OpenAI GPT-4o model is used specifically for keyword extraction; ensure your OpenAI API key has access to this model. | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4o |
| Google Drive OAuth2 and Google Sheets OAuth2 require scopes for file access and spreadsheet editing, respectively. | Google API scopes documentation |
| The workflow is designed to handle MP4 videos uploaded to a specific folder; other video formats or very large files may require adjustment. | Workflow design note |
| The Dumpling AI agent uses Perplexity and Google Search internally to gather blog post data; results depend on their API availability and response quality. | Workflow design note |
| For performance and error handling, consider adding error workflow branches or notifications for API failures or large file issues. | Best practice advice |
| This workflow exemplifies combining video content ingestion with AI-driven content research and structured output for content marketing workflows. | Conceptual note |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. It complies strictly with current content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.

---