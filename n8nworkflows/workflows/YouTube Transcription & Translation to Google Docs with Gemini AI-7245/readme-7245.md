YouTube Transcription & Translation to Google Docs with Gemini AI

https://n8nworkflows.xyz/workflows/youtube-transcription---translation-to-google-docs-with-gemini-ai-7245


# YouTube Transcription & Translation to Google Docs with Gemini AI

### 1. Workflow Overview

This workflow automates the process of transcribing a YouTube video, summarizing and translating the transcript into a specified language, and then saving the results into a Google Docs document. It is designed for users who want to quickly obtain multilingual summaries and translations of YouTube video content, stored conveniently in Google Docs for further reference or sharing.

The logic is divided into the following functional blocks:

- **1.1 Input Reception and Formatting:** Receives webhook input containing the YouTube URL, target language, and summary preference, and extracts necessary parameters.
- **1.2 Video Transcription Retrieval:** Calls an external API to obtain the full transcription of the YouTube video.
- **1.3 Transcript Processing:** Merges segmented transcription entries into a single text string.
- **1.4 AI Processing (Summarization & Translation):** Uses Google Gemini AI model to generate a translated and summarized version of the transcript in the target language.
- **1.5 Google Docs Output Generation:** Creates a new Google Docs document and appends the AI-generated summary and translation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Formatting

**Overview:**  
This block initiates the workflow by receiving an HTTP POST request with parameters and formats them to extract the YouTube video ID and user preferences.

**Nodes Involved:**  
- Trigger YouTube Processing Request (Webhook)  
- Format Webhook Input (Code)

**Node Details:**

- **Trigger YouTube Processing Request**  
  - *Type:* Webhook  
  - *Role:* Entry point accepting POST requests at path `/youtube`.  
  - *Configuration:* Expects JSON body with keys: `youtube_url`, `language`, `enable_summary`.  
  - *Input:* HTTP POST request.  
  - *Output:* Passes raw JSON to next node.  
  - *Edge Cases:* Missing or malformed URL, unsupported HTTP method, webhook authentication not configured (could allow unauthorized access).  

- **Format Webhook Input**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Extracts 11-character YouTube video ID from URL and packages data into structured JSON with `videoId`, `language`, `enable_summary`, and `originalUrl`.  
  - *Key Expressions:* Uses regex `/v=|\/)([0-9A-Za-z_-]{11})/` to extract video ID.  
  - *Input:* JSON from webhook.  
  - *Output:* JSON object with formatted parameters.  
  - *Edge Cases:* No valid video ID found (null value), invalid URL formats, missing keys in input JSON. Could lead to downstream failures if not handled.  

---

#### 2.2 Video Transcription Retrieval

**Overview:**  
Sends the extracted video ID to Supadata’s transcription API to retrieve the full transcript of the YouTube video.

**Nodes Involved:**  
- Transcribe YouTube Video (HTTP Request)

**Node Details:**

- **Transcribe YouTube Video**  
  - *Type:* HTTP Request  
  - *Role:* Fetches transcription data from Supadata API using the video ID.  
  - *Configuration:*  
    - URL: `https://api.supadata.ai/v1/youtube/transcript?`  
    - Query Parameter: `videoId` from formatted input  
    - Header: `x-api-key` (must be set with a valid API key)  
  - *Input:* JSON with `videoId`.  
  - *Output:* JSON containing transcription segments in `content` array.  
  - *Edge Cases:* API key missing/invalid (401 Unauthorized), video ID invalid (404 or empty response), rate limiting, network timeouts.  
  - *Version:* HTTP Request v4.2 used for advanced query and header control.  

---

#### 2.3 Transcript Processing

**Overview:**  
Consolidates segmented transcription into a single coherent string to prepare for AI processing.

**Nodes Involved:**  
- Combine Transcription Content (Code)

**Node Details:**

- **Combine Transcription Content**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Takes the `content` array from the transcription API and concatenates all `text` fields into one string `mergedTranscript`.  
  - *Key Expressions:* Maps over `$json["content"]` extracting `text`, then joins with spaces.  
  - *Input:* JSON with transcription segments.  
  - *Output:* JSON with single string field `mergedTranscript`.  
  - *Edge Cases:* Empty or malformed `content` array, missing `text` fields, large transcript size causing memory issues.  

---

#### 2.4 AI Processing (Summarization & Translation)

**Overview:**  
Uses the Google Gemini AI model to produce a translated and summarized version of the transcript in the user-specified language.

**Nodes Involved:**  
- Summarize and Translate Text (Basic LLM Chain)  
- Google Gemini To Summarize and Translate Text (Google Gemini Chat Model)

**Node Details:**

- **Summarize and Translate Text**  
  - *Type:* Basic LLM Chain (Langchain)  
  - *Role:* Defines the prompt template to instruct the AI to translate and summarize the transcript.  
  - *Configuration:*  
    - Prompt includes instructions to generate JSON output with keys `translation` and `summary`, both in the target language.  
    - Injects variables: target language and merged transcript from previous nodes.  
    - Enforces output format as valid JSON without markdown or code blocks.  
  - *Input:* `mergedTranscript` and `language` parameters.  
  - *Output:* JSON string response from AI with translation and summary fields.  
  - *Edge Cases:* AI generating invalid JSON, incomplete output, or unexpected language.  
  - *Version:* v1.7 for Langchain node.  

- **Google Gemini To Summarize and Translate Text**  
  - *Type:* Google Gemini Chat Model (Langchain)  
  - *Role:* Actual AI model node that executes the prompt chain using Google Gemini (PaLM).  
  - *Configuration:*  
    - Uses model "models/gemini-1.5-flash".  
    - Requires Google Palm API credentials.  
  - *Input:* Receives prompt from the Langchain node.  
  - *Output:* AI-generated text fulfilling the prompt.  
  - *Edge Cases:* API quota limits, network issues, invalid credentials, or model downtime.  
  - *Sub-workflow:* Integrated as AI language model node inside Summarize and Translate Text chain.  

---

#### 2.5 Google Docs Output Generation

**Overview:**  
Creates a new Google Docs document named after the video ID and target language, then inserts the summarized translation text.

**Nodes Involved:**  
- Create Output Document (Google Docs)  
- Append Summary and Translation (Google Docs)

**Node Details:**

- **Create Output Document**  
  - *Type:* Google Docs node  
  - *Role:* Creates a new Google Docs document in a specified folder.  
  - *Configuration:*  
    - Title is dynamically generated as `{videoId}_{language}`.  
    - Folder ID is fixed: `"1FM-kd1_Xi2ikKpK7xAULFWYc_qVH2wwc"`.  
    - Uses OAuth2 credentials for Google Docs.  
  - *Input:* Receives output from AI processing node.  
  - *Output:* Document metadata including document URL/ID.  
  - *Edge Cases:* Insufficient permissions, invalid folder ID, credential expiration.  

- **Append Summary and Translation**  
  - *Type:* Google Docs node  
  - *Role:* Updates the newly created document by inserting the AI-generated summary and translation text at the document location (beginning).  
  - *Configuration:*  
    - Uses document URL/ID from previous node.  
    - Inserts text from `Summarize and Translate Text` node output field `text`.  
    - Requires same OAuth2 credentials as above.  
  - *Input:* Document ID and AI output text.  
  - *Output:* Confirmation of document update.  
  - *Edge Cases:* Document locked or deleted, network issues, invalid text format.  

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                                | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                   |
|-------------------------------|----------------------------------|-----------------------------------------------|----------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Trigger YouTube Processing Request | Webhook                         | Entry point receiving POST with YouTube URL, language, summary flag | -                          | Format Webhook Input           | # **Node Breakdown & Descriptions:** Starts workflow with webhook input.                      |
| Format Webhook Input           | Code                             | Extracts video ID and formats input JSON      | Trigger YouTube Processing Request | Transcribe YouTube Video       | Reformats incoming data for downstream use.                                                 |
| Transcribe YouTube Video      | HTTP Request                     | Calls Supadata API to get YouTube transcription | Format Webhook Input         | Combine Transcription Content  | Requires valid API key; fetches transcription segments.                                     |
| Combine Transcription Content | Code                             | Merges transcription segments into one string | Transcribe YouTube Video     | Summarize and Translate Text   | Consolidates transcript text for AI input.                                                  |
| Summarize and Translate Text | Basic LLM Chain (Langchain)       | Defines prompt for AI translation and summary | Combine Transcription Content | Create Output Document         | Sends merged transcript and language to AI chain.                                          |
| Google Gemini To Summarize and Translate Text | Google Gemini Chat Model         | Executes AI prompt for Gemini model            | Summarize and Translate Text (AI input) | Summarize and Translate Text (output) | Uses Google Gemini AI for translation and summarization.                                    |
| Create Output Document        | Google Docs                      | Creates new Google Docs document for output   | Summarize and Translate Text | Append Summary and Translation | Creates doc titled by video ID and language in fixed folder.                                 |
| Append Summary and Translation | Google Docs                      | Inserts AI-generated content into Google Doc  | Create Output Document       | -                             | Updates document with translated and summarized text.                                       |
| Sticky Note                  | Sticky Note                      | Workflow title display                         | -                          | -                             | "## YouTube Transcription, Summarization & Translation to Google Docs"                      |
| Sticky Note1                 | Sticky Note                      | Node breakdown and descriptions                | -                          | -                             | Detailed node descriptions and workflow overview.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Trigger YouTube Processing Request`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/youtube`  
   - Purpose: Receive JSON payload with `youtube_url`, `language`, and `enable_summary`.  

2. **Add Code Node**  
   - Name: `Format Webhook Input`  
   - Paste JavaScript code to extract the YouTube video ID from the URL and format input JSON:  
     ```js
     const url = $json["body"]["youtube_url"];
     const match = url.match(/(?:v=|\/)([0-9A-Za-z_-]{11})/);
     const videoId = match ? match[1] : null;

     return [
       {
         json: {
           videoId,
           language: $json["body"]["language"],
           enable_summary: $json["body"]["enable_summary"],
           originalUrl: url
         }
       }
     ];
     ```  
   - Connect output of webhook to this node.  

3. **Add HTTP Request Node**  
   - Name: `Transcribe YouTube Video`  
   - URL: `https://api.supadata.ai/v1/youtube/transcript?`  
   - HTTP Method: GET (default)  
   - Query Parameter: `videoId` set to expression `={{ $json.videoId }}`  
   - Header Parameter: `x-api-key` with your Supadata API key  
   - Connect output of `Format Webhook Input` to this node.  

4. **Add Code Node**  
   - Name: `Combine Transcription Content`  
   - JavaScript code to merge transcription text:  
     ```js
     const data = $json["content"];
     const mergedText = data.map(entry => entry.text).join(" ");
     return [
       {
         json: {
           mergedTranscript: mergedText
         }
       }
     ];
     ```  
   - Connect output of `Transcribe YouTube Video` to this node.  

5. **Add Basic LLM Chain Node**  
   - Name: `Summarize and Translate Text`  
   - Prompt Type: Define (custom prompt)  
   - Prompt template (copy the prompt with variables):  
     ```
     This is a transcription that needs processing.

     Please do the following:

     1. Generate a complete and accurate translation of the transcript into the specified language.
     2. Generate a concise and clear summary of the transcript’s key points in the same language.

     Target Language:
     {{ $('Combine Transcription Content').item.json.language }}

     Transcript:
     {{ $('Combine Transcription Content').item.json.mergedTranscript }}

     Output Format:
     Please return the result strictly in this JSON structure, with both values written in the target language:

     {
       "translation": "Full translated version of the transcript in the specified language.",
       "summary": "Clear and concise summary of the transcript in the same language."
     }

     Make sure:
     - Both "translation" and "summary" keys are present.
     - The summary should not be skipped or embedded inside the translation.
     - Do NOT include markdown formatting (no triple backticks or code blocks).
     - Output must be valid JSON only.
     ```  
   - Connect output of `Combine Transcription Content` to this node.  

6. **Add Google Gemini Chat Model Node**  
   - Name: `Google Gemini To Summarize and Translate Text`  
   - Model Name: `models/gemini-1.5-flash`  
   - Credentials: Configure and select your Google Palm API credentials with appropriate access to Gemini models.  
   - Connect this node as AI language model input to `Summarize and Translate Text` node.  

7. **Add Google Docs Node**  
   - Name: `Create Output Document`  
   - Operation: Create Document  
   - Title: Set expression `={{ $('Format Webhook Input').item.json.videoId }}_{{ $('Format Webhook Input').item.json.language }}`  
   - Folder ID: `"1FM-kd1_Xi2ikKpK7xAULFWYc_qVH2wwc"` (replace with your folder if needed)  
   - Credentials: Google Docs OAuth2 account with write permissions  
   - Connect main output of `Summarize and Translate Text` to this node.  

8. **Add Google Docs Node**  
   - Name: `Append Summary and Translation`  
   - Operation: Update Document  
   - Document URL: Use expression `={{ $json.id }}` from `Create Output Document` node output  
   - Actions: Insert Text at Location (start of document)  
   - Text: Expression `={{ $('Summarize and Translate Text').item.json.text }}` (ensure this matches the field containing AI output text)  
   - Credentials: Same Google Docs OAuth2 account as above  
   - Connect main output of `Create Output Document` to this node.  

9. **Add Sticky Notes (Optional)**  
   - Add descriptive sticky notes for workflow title and node explanations as per original workflow for documentation clarity.  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses Supadata API for YouTube transcription which requires a valid API key and account setup.  | https://supadata.ai                                                                              |
| The AI summarization and translation leverages Google Gemini (PaLM) via Langchain integration in n8n.         | Requires Google Cloud account with PaLM API access                                              |
| Google Docs OAuth2 credentials need appropriate scopes (`https://www.googleapis.com/auth/documents`).        | See Google Cloud Console API & Services documentation                                           |
| The workflow is triggered by an HTTP POST request with JSON body containing `youtube_url`, `language`, and `enable_summary`. | Use tools like Postman or curl to test webhook endpoint.                                        |
| Regex for video ID extraction supports standard YouTube URL formats but may fail on uncommon or shortened URLs. | Consider extending regex or validating input URLs before processing.                            |
| The AI output prompt enforces JSON-only output to facilitate parsing and insertion into Google Docs.         | Any deviations or formatting issues from AI may cause downstream errors.                         |

---

**Disclaimer:** This document is generated from an automated n8n workflow export. All data processed is legal and publicly accessible. The workflow respects all content policies and contains no illegal, offensive, or protected elements.