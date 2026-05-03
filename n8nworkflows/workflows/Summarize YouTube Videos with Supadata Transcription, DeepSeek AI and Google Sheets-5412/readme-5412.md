Summarize YouTube Videos with Supadata Transcription, DeepSeek AI and Google Sheets

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-with-supadata-transcription--deepseek-ai-and-google-sheets-5412


# Summarize YouTube Videos with Supadata Transcription, DeepSeek AI and Google Sheets

### 1. Workflow Overview

This workflow automates the process of summarizing YouTube videos by leveraging transcription from Supadata, natural language processing with DeepSeek AI, and data organization in Google Sheets. It is designed for users who want to efficiently extract structured summaries, key points, and main topics from video content, then update a tracking spreadsheet with these insights.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Data Retrieval**: Initiated manually to fetch pending YouTube video URLs from a Google Sheet.
- **1.2 Transcript Generation**: Uses Supadata API to obtain the video transcript for each URL.
- **1.3 Transcript Storage**: Updates the Google Sheet with the retrieved transcript.
- **1.4 AI-Based Summarization**: Sends the transcript to DeepSeek AI to generate a structured summary, key points, and topics.
- **1.5 Summary Post-Processing**: Parses and cleans the AI output to proper JSON and prepares data for storage.
- **1.6 Summary Storage and Workflow Looping**: Updates the Google Sheet with the summary and marks the status as done, then loops to process the next pending URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Retrieval

- **Overview:**  
  Starts the workflow on manual trigger and fetches the next YouTube video URL marked as "Pending" from a Google Sheet to be processed.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get URL to Transcript (Google Sheets Read)

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution  
    - Configuration: No parameters; triggered manually  
    - Outputs: Triggers "Get URL to Transcript"  
    - Failure Types: None; manual trigger  
  - **Get URL to Transcript**  
    - Type: Google Sheets  
    - Role: Reads first row where "Status" equals "Pending" to get URL for processing  
    - Configuration:  
      - Document ID and Sheet ID set to specific Google Sheet  
      - Filter applied on “Status” column = "Pending"  
      - Return first match only  
    - Inputs: Trigger from manual node or loop back from "Adding Summary to file"  
    - Outputs: Passes URL JSON to "Generating transcript"  
    - Failure Types: Google Sheets API errors, no matching rows found (workflow might stall if no "Pending")  
    - Credentials: Google Sheets OAuth2  

#### 2.2 Transcript Generation

- **Overview:**  
  Requests the transcript of the YouTube video URL from Supadata API with a text format response.

- **Nodes Involved:**  
  - Generating transcript (HTTP Request)  
  - Adding transcript to file (Google Sheets Write)

- **Node Details:**  
  - **Generating transcript**  
    - Type: HTTP Request  
    - Role: Calls Supadata API to get YouTube transcript  
    - Configuration:  
      - URL constructed dynamically using URL from previous node  
      - Response format: JSON  
      - Header includes `x-api-key` with placeholder for API key  
    - Inputs: URL JSON from "Get URL to Transcript"  
    - Outputs: JSON transcript content to "Adding transcript to file"  
    - Failure Types: HTTP errors (401 if API key invalid, 429 rate limiting, 500 server errors), network errors  
    - Notes: API key must be provided for successful request  
  - **Adding transcript to file**  
    - Type: Google Sheets  
    - Role: Updates the Google Sheet row with the transcript text under the "Transcript" column  
    - Configuration:  
      - Document ID and Sheet ID same as previous sheet node  
      - Matching row by "Url" column  
      - Updates "Transcript" with API response content  
    - Inputs: Transcript JSON from HTTP request  
    - Outputs: Passes JSON with transcript to "Generate Summary"  
    - Failure Types: Google Sheets API errors, data mapping issues  
    - Credentials: Google Sheets OAuth2  

#### 2.3 AI-Based Summarization

- **Overview:**  
  Sends the transcript text to DeepSeek AI's chat model to generate a structured summary in JSON format focusing on core topics, key points, and main ideas.

- **Nodes Involved:**  
  - Generate Summary (Langchain LLM Chain)  
  - DeepSeek Chat Model (Langchain Chat Model)

- **Node Details:**  
  - **DeepSeek Chat Model**  
    - Type: Langchain Chat Model Node (DeepSeek)  
    - Role: Processes the prompt to generate a summary using DeepSeek API  
    - Configuration: Uses DeepSeek API credentials  
    - Inputs: Transcript JSON from "Adding transcript to file"  
    - Outputs: Raw AI response to "Generate Summary"  
    - Failure Types: API authentication errors, timeouts, rate limits, malformed input  
    - Credentials: DeepSeek API key  
  - **Generate Summary**  
    - Type: Langchain LLM Chain  
    - Role: Defines the prompt template specifying the summarization instructions and expected JSON output format  
    - Configuration:  
      - Prompt instructs summarization with strict JSON output, focusing on clarity, key points, and neutrality  
      - Uses transcript from previous node as variable `{{ $json.Transcript }}`  
    - Inputs: AI chat model response  
    - Outputs: Parsed JSON summary to "Clear code" node  
    - Failure Types: Prompt failures, incorrect response format, parsing errors  

#### 2.4 Summary Post-Processing

- **Overview:**  
  Parses the raw JSON summary output from AI, cleans formatting, extracts summary, key points, and topics into separate fields for easier storage.

- **Nodes Involved:**  
  - Clear code (Code Node)

- **Node Details:**  
  - **Clear code**  
    - Type: Code (JavaScript)  
    - Role: Cleans AI output text, removes backticks or markdown artifacts, parses JSON, joins arrays into strings  
    - Key Logic: Parses the first array element, extracts `summary`, joins `key_points` with newlines, joins `topics` with commas  
    - Inputs: Raw AI JSON text from "Generate Summary"  
    - Outputs: Cleaned JSON with fields `summary`, `key_points`, `topics` to "Adding Summary to file"  
    - Failure Types: JSON parse errors, unexpected output format, runtime exceptions  

#### 2.5 Summary Storage and Workflow Looping

- **Overview:**  
  Updates the Google Sheet row with the cleaned summary, key points, and topics, and marks the video as processed ("Done"). Then triggers the next iteration to process remaining pending URLs.

- **Nodes Involved:**  
  - Adding Summary to file (Google Sheets Write)  
  - Get URL to Transcript (Google Sheets Read) [loop back]

- **Node Details:**  
  - **Adding Summary to file**  
    - Type: Google Sheets  
    - Role: Updates the Google Sheet row matching the URL with:  
      - Status = "Done"  
      - Summary text formatted with sections for Summary, Key Points, Topics  
    - Configuration:  
      - Document ID and Sheet ID consistent with other Google Sheets nodes  
      - Matching by "Url" column  
      - Writes multi-line summary text for readability  
    - Inputs: Cleaned summary JSON from "Clear code"  
    - Outputs: Loops back to "Get URL to Transcript" for next pending item  
    - Failure Types: Google Sheets API errors, data mapping issues  
    - Credentials: Google Sheets OAuth2  

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                      | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                                                                    |
|---------------------------|------------------------------------|------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts workflow on manual command  | —                           | Get URL to Transcript         |                                                                                                                                                               |
| Get URL to Transcript      | Google Sheets                      | Reads next pending YouTube URL     | When clicking ‘Execute workflow’, Adding Summary to file | Generating transcript          |                                                                                                                                                               |
| Generating transcript      | HTTP Request                      | Calls Supadata API for transcript  | Get URL to Transcript         | Adding transcript to file      | ## Generating Transcript \nHere we get transcript.\nI use supadata.ai. There are 100 free credit                                                               |
| Adding transcript to file  | Google Sheets                     | Updates sheet with transcript      | Generating transcript         | DeepSeek Chat Model            |                                                                                                                                                               |
| DeepSeek Chat Model        | Langchain Chat Model (DeepSeek)  | Sends transcript to DeepSeek AI    | Adding transcript to file     | Generate Summary               |                                                                                                                                                               |
| Generate Summary           | Langchain LLM Chain               | Defines prompt and processes summary | DeepSeek Chat Model           | Clear code                    | ## Generating Summary\nHere we generating summary                                                                                                            |
| Clear code                | Code Node (JavaScript)             | Cleans and parses AI output        | Generate Summary              | Adding Summary to file         |                                                                                                                                                               |
| Adding Summary to file     | Google Sheets                     | Updates sheet with summary and status | Clear code                   | Get URL to Transcript          |                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Execute workflow’  
   - Type: Manual Trigger  
   - No configuration needed.

2. **Create Google Sheets Node to Get URL**  
   - Name: Get URL to Transcript  
   - Operation: Read (Return First Match)  
   - Sheet: Use specific Google Sheet ID and Sheet name (same throughout workflow)  
   - Filter: Status column = "Pending"  
   - Matching: Return first matching row only  
   - Connect input from Manual Trigger and later from Adding Summary to file node (for looping)  
   - Set Google Sheets OAuth2 credential.

3. **Create HTTP Request Node for Transcript**  
   - Name: Generating transcript  
   - HTTP Method: GET  
   - URL: `https://api.supadata.ai/v1/youtube/transcript?url={{ $json.Url }}&text=true`  
   - Headers: Add header `x-api-key` with your Supadata API key  
   - Response Format: JSON  
   - Connect input from Get URL to Transcript node.

4. **Create Google Sheets Node to Add Transcript**  
   - Name: Adding transcript to file  
   - Operation: Update  
   - Match rows by "Url" column  
   - Update "Transcript" column with `={{ $json.content }}` (response from HTTP Request)  
   - Connect input from Generating transcript node.  
   - Set Google Sheets OAuth2 credential.

5. **Create DeepSeek Chat Model Node**  
   - Name: DeepSeek Chat Model  
   - Use DeepSeek API credentials  
   - Connect input from Adding transcript to file node.

6. **Create Langchain LLM Chain Node for Summary**  
   - Name: Generate Summary  
   - Prompt:  
     ```
     Summarize the following video transcript in a structured, clear, and concise manner. Focus on the key points, main ideas, and essential takeaways. Avoid unnecessary details and redundant information. Ensure that the summary is logically organized and easy to understand.

     Guidelines for summarization:

     Identify core topics: Capture the main subject of the discussion.

     Extract key points: Highlight the most important arguments, insights, and conclusions.

     Use clear and precise language: Ensure readability and logical flow.

     Maintain neutrality: Avoid opinions or interpretations that are not in the transcript.

     Format for clarity: If the transcript includes multiple sections or topics, structure the summary using bullet points or short paragraphs.

     Expected Output Format:
     Return the summary as a structured text response, using either paragraphs or bullet points, depending on the complexity of the content. If applicable, include subheadings for better organization.
     Summarize the following video transcript in a structured JSON format.

     Focus on key points, main ideas, and important takeaways.
     Here is the transcript: {{ $json.Transcript }}
     ```
   - Configure to parse output as JSON  
   - Connect input from DeepSeek Chat Model node.

7. **Create Code Node to Clean Summary**  
   - Name: Clear code  
   - Language: JavaScript  
   - Code:  
     ```javascript
     return items.map(item => {
       const rawText = $input.first().json.text;

       const cleaned = rawText
         .replace(/^```json\s*/, '')
         .replace(/\s*```$/, '')
         .trim();

       let parsed;
       try {
         parsed = JSON.parse(cleaned);
       } catch (e) {
         throw new Error('Error parsing field "text".');
       }

       const data = parsed[0];

       return {
         json: {
           summary: data.summary,
           key_points: data.key_points.join('\n'),
           topics: data.topics.join(', ')
         }
       };
     });
     ```
   - Connect input from Generate Summary node.

8. **Create Google Sheets Node to Add Summary**  
   - Name: Adding Summary to file  
   - Operation: Update  
   - Match rows by "Url" column  
   - Update columns:  
     - "Url": `={{ $('Get URL to Transcript').item.json.Url }}`  
     - "Status": "Done"  
     - "Summary":  
       ```
       Summary:
       {{ $json.summary }} 
       Key Points:
       {{ $json.key_points }}
       Topics:
       {{ $json.topics }}
       ```
   - Connect input from Clear code node.  
   - Set Google Sheets OAuth2 credential.  
   - Connect output back to Get URL to Transcript node to process next pending URL.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                         |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------|
| Supadata.ai offers a free quota of 100 credits for transcription API usage.                    | Supadata API Documentation                              |
| DeepSeek AI integration is used via Langchain nodes requiring an API key credential.          | DeepSeek API & Langchain n8n integration documentation  |
| Google Sheets document ID and sheet ID must remain consistent across all Google Sheets nodes. | Google Sheets API and OAuth2 setup                       |
| The summary prompt enforces strict JSON output format to facilitate parsing and storage.      | Prompt engineering best practices                        |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly follows applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.