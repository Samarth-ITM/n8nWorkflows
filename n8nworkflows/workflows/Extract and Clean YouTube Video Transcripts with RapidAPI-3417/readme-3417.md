Extract and Clean YouTube Video Transcripts with RapidAPI

https://n8nworkflows.xyz/workflows/extract-and-clean-youtube-video-transcripts-with-rapidapi-3417


# Extract and Clean YouTube Video Transcripts with RapidAPI

### 1. Workflow Overview

This workflow automates the extraction and cleaning of YouTube video transcripts using the YouTube Transcript API available on RapidAPI. It is designed for users who want to retrieve subtitles from YouTube videos, clean and format the transcript text, and output structured transcript data for further use such as analysis, summarization, or repurposing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the YouTube video URL from the user via a form trigger.
- **1.2 Transcript Retrieval:** Sends a POST request to the YouTube Transcript API to fetch the transcript data.
- **1.3 Transcript Processing:** Cleans and formats the raw transcript text, handling different API response formats and error cases.
- **1.4 Output Formatting:** Structures the cleaned transcript into a user-friendly format for output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block provides a user interface to input the YouTube video URL. It acts as the workflow’s entry point.

- **Nodes Involved:**  
  - `YoutubeVideoURL`

- **Node Details:**  
  - **Node Name:** YoutubeVideoURL  
  - **Type:** Form Trigger  
  - **Technical Role:** Captures user input via a web form to trigger the workflow.  
  - **Configuration:**  
    - Form titled "Youtube Video Transcriber"  
    - Single required field labeled "Youtube Video Url"  
  - **Expressions/Variables:** The form field value is accessed as `$json['Youtube Video Url']` in downstream nodes.  
  - **Input/Output Connections:** No input nodes; output connected to `extractTranscript`.  
  - **Version Requirements:** Uses Form Trigger v2.2, requires n8n version supporting this node type.  
  - **Potential Failures:** User may input invalid or malformed URLs; no validation beyond required field is configured.  
  - **Sub-workflow:** None.

---

#### 2.2 Transcript Retrieval

- **Overview:**  
  Sends a POST request to the YouTube Transcript API via RapidAPI to retrieve the transcript for the provided YouTube video URL.

- **Nodes Involved:**  
  - `extractTranscript`

- **Node Details:**  
  - **Node Name:** extractTranscript  
  - **Type:** HTTP Request  
  - **Technical Role:** Makes an authenticated API call to fetch transcript data.  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://youtube-transcript3.p.rapidapi.com/api/transcript`  
    - Body Parameters: includes `url` set dynamically from the form input `$json['Youtube Video Url']`.  
    - Query Parameters: includes a static `videoId` parameter set to `"ZacjOVVgoLY"` (likely a placeholder or example; should be dynamic or removed).  
    - Headers:  
      - `x-rapidapi-host`: `youtube-transcript3.p.rapidapi.com`  
      - `x-rapidapi-key`: `"your_api_key"` (to be replaced with actual API key)  
      - `Content-Type`: `application/json`  
  - **Expressions/Variables:** Uses expression to pass the video URL from the form input.  
  - **Input/Output Connections:** Input from `YoutubeVideoURL`; output to `processTranscript`.  
  - **Version Requirements:** HTTP Request node v3.  
  - **Potential Failures:**  
    - Authentication errors if API key is invalid or missing.  
    - Network timeouts or API rate limits.  
    - API returning no transcript or error responses.  
  - **Sub-workflow:** None.

---

#### 2.3 Transcript Processing

- **Overview:**  
  Processes the raw transcript data returned by the API, cleans the text by removing unwanted characters and formatting it for readability, and handles cases where no transcript is available.

- **Nodes Involved:**  
  - `processTranscript`

- **Node Details:**  
  - **Node Name:** processTranscript  
  - **Type:** Function  
  - **Technical Role:** Custom JavaScript code to parse and clean transcript data.  
  - **Configuration:**  
    - Checks if transcript data exists in either `transcript` or `text` fields.  
    - Supports different API response formats: array of segments or single text string.  
    - Concatenates transcript segments into a single string.  
    - Cleans transcript by removing extra spaces and normalizing punctuation spacing.  
    - Returns JSON with:  
      - `success` flag  
      - `videoUrl` (from input or fallback)  
      - `rawTranscript` (original text)  
      - `cleanedTranscript` (processed text)  
      - `duration`, `offset`, `language` if available  
  - **Expressions/Variables:** Uses `$input.first().json` to access API response.  
  - **Input/Output Connections:** Input from `extractTranscript`; output to `cleanedTranscript`.  
  - **Version Requirements:** Function node v1.  
  - **Potential Failures:**  
    - Missing or malformed API response data causing runtime errors.  
    - Unexpected transcript formats not handled by the code.  
    - Errors in expression evaluation or JavaScript exceptions.  
  - **Sub-workflow:** None.

---

#### 2.4 Output Formatting

- **Overview:**  
  Structures the cleaned transcript data into a simple JSON field for output or further use.

- **Nodes Involved:**  
  - `cleanedTranscript`

- **Node Details:**  
  - **Node Name:** cleanedTranscript  
  - **Type:** Set  
  - **Technical Role:** Assigns the cleaned transcript string to a field named `transcript` for output.  
  - **Configuration:**  
    - Sets `transcript` field to the value of `cleanedTranscript` from the previous node’s JSON.  
  - **Expressions/Variables:** Uses expression `={{ $json.cleanedTranscript }}` to assign value.  
  - **Input/Output Connections:** Input from `processTranscript`; no further output nodes.  
  - **Version Requirements:** Set node v3.4.  
  - **Potential Failures:** If `cleanedTranscript` is missing or undefined, output will be empty or invalid.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name         | Node Type       | Functional Role                | Input Node(s)     | Output Node(s)    | Sticky Note                                                                                  |
|-------------------|-----------------|-------------------------------|-------------------|-------------------|----------------------------------------------------------------------------------------------|
| YoutubeVideoURL    | Form Trigger    | User input of YouTube video URL | None              | extractTranscript |                                                                                              |
| extractTranscript  | HTTP Request    | Fetch transcript from API      | YoutubeVideoURL   | processTranscript | Replace `"your_api_key"` with your actual RapidAPI key.                                     |
| processTranscript  | Function        | Clean and format transcript    | extractTranscript | cleanedTranscript | Handles different transcript formats; returns error message if no transcript is available.  |
| cleanedTranscript  | Set             | Format output transcript field | processTranscript | None              | Outputs cleaned transcript as `transcript` field.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Name: `YoutubeVideoURL`  
   - Type: Form Trigger (v2.2)  
   - Configure form title: "Youtube Video Transcriber"  
   - Add one required field: Label "Youtube Video Url" (string input)  
   - This node will trigger the workflow when the form is submitted.

2. **Create HTTP Request Node**  
   - Name: `extractTranscript`  
   - Type: HTTP Request (v3)  
   - Method: POST  
   - URL: `https://youtube-transcript3.p.rapidapi.com/api/transcript`  
   - Body Parameters: Add parameter `url` with value expression `{{$json["Youtube Video Url"]}}`  
   - Query Parameters: (Optional) Remove or dynamically set `videoId` if needed; currently set to `"ZacjOVVgoLY"` (can be omitted)  
   - Headers:  
     - `x-rapidapi-host`: `youtube-transcript3.p.rapidapi.com`  
     - `x-rapidapi-key`: Replace `"your_api_key"` with your actual RapidAPI key (store securely in credentials or environment variables)  
     - `Content-Type`: `application/json`  
   - Connect input from `YoutubeVideoURL`.

3. **Create Function Node**  
   - Name: `processTranscript`  
   - Type: Function (v1)  
   - Paste the following JavaScript code:  
     ```javascript
     const data = $input.first().json;

     if (!data.transcript && !data.text) {
       return {
         json: {
           success: false,
           message: 'No transcript available for this video',
           videoUrl: $input.first().json.body?.videoUrl || 'Unknown'
         }
       };
     }

     let transcriptText = '';

     if (data.transcript) {
       if (Array.isArray(data.transcript)) {
         data.transcript.forEach(segment => {
           if (segment.text) {
             transcriptText += segment.text + ' ';
           }
         });
       } else if (typeof data.transcript === 'string') {
         transcriptText = data.transcript;
       }
     } else if (data.text) {
       transcriptText = data.text;
     }

     const cleanedTranscript = transcriptText
       .replace(/\s+/g, ' ')
       .replace(/\s([.,!?])/g, '$1')
       .trim();

     return {
       json: {
         success: true,
         videoUrl: $input.first().json.body?.videoUrl || 'From transcript',
         rawTranscript: data.text || data.transcript,
         cleanedTranscript,
         duration: data.duration,
         offset: data.offset,
         language: data.lang
       }
     };
     ```
   - Connect input from `extractTranscript`.

4. **Create Set Node**  
   - Name: `cleanedTranscript`  
   - Type: Set (v3.4)  
   - Add assignment:  
     - Field Name: `transcript`  
     - Type: String  
     - Value: Expression `={{ $json.cleanedTranscript }}`  
   - Connect input from `processTranscript`.

5. **Connect Nodes**  
   - `YoutubeVideoURL` → `extractTranscript` → `processTranscript` → `cleanedTranscript`

6. **Credential Setup**  
   - Obtain a RapidAPI account and subscribe to the YouTube Transcript API.  
   - Store your API key securely in n8n credentials or environment variables.  
   - Replace `"your_api_key"` in the HTTP Request node with the actual key or use an expression to reference credentials.

7. **Activate Workflow**  
   - Save and activate the workflow.  
   - Access the form trigger URL to input a YouTube video URL.  
   - The workflow will output the cleaned transcript.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The YouTube Transcript API is accessed via RapidAPI; ensure you have an active subscription. | [YouTube Transcript API on RapidAPI](https://rapidapi.com/solid-api-solid-api-default/api/youtube-transcript3/playground) |
| n8n installation and setup guide available at official docs.                                  | [n8n Documentation](https://docs.n8n.io/)                                                         |
| Customize the Function node to modify transcript cleaning rules or output format as needed.    | See "Customization Guide" section in the workflow description.                                    |
| Consider adding database nodes or AI summarization nodes for extended functionality.          | Workflow can be extended with MySQL/PostgreSQL/Firebase nodes or OpenAI nodes for summaries.      |

---

This documentation provides a complete and detailed reference to understand, reproduce, and customize the YouTube Transcript Extraction workflow in n8n.