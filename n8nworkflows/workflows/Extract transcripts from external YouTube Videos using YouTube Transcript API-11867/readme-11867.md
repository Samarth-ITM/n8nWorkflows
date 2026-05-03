Extract transcripts from external YouTube Videos using YouTube Transcript API

https://n8nworkflows.xyz/workflows/extract-transcripts-from-external-youtube-videos-using-youtube-transcript-api-11867


# Extract transcripts from external YouTube Videos using YouTube Transcript API

### 1. Workflow Overview

This workflow extracts clean transcripts from any public YouTube video by utilizing the external YouTube Transcript API service available at youtube-transcript.io. It is designed as a sub-workflow triggered by another workflow, receiving a YouTube video ID as input and returning a cleaned transcript with metadata such as language, word count, and character count.

The workflow is logically grouped into these blocks:

- **1.1 Input Reception:** Accepts the video ID as input from a parent workflow.
- **1.2 API Request:** Sends a POST request to youtube-transcript.io to fetch the transcript data.
- **1.3 Parsing & Validation:** Parses the API response safely, handling multiple possible response formats.
- **1.4 Transcript Availability Decision:** Checks if the transcript is present.
- **1.5 Transcript Cleaning:** Cleans and normalizes the transcript text if available.
- **1.6 Error Handling:** Provides a structured fallback and stops execution if no transcript is found.
- **1.7 Final Output:** Outputs structured data including transcript and metrics.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block receives the YouTube video ID as input when the workflow is executed by another workflow. It sets the video ID as a variable for downstream use.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Set Variables

**Node Details:**  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry trigger for sub-workflow execution  
  - Configuration: Accepts a single input parameter `youtubeVideoId`  
  - Inputs: External workflow trigger  
  - Outputs: Passes input data to next node  
  - Edge Cases: Missing or invalid `youtubeVideoId` will cause downstream failures  
  - Version: 1.1  

- **Set Variables**  
  - Type: Set  
  - Role: Stores `youtubeVideoId` variable from input JSON for consistent reference  
  - Configuration: Assigns `youtubeVideoId` using expression `={{ $json.youtubeVideoId }}`  
  - Inputs: From trigger node  
  - Outputs: Passes assigned variable to next node  
  - Edge Cases: No special cases; variable must exist  
  - Version: 3.4  

---

#### 2.2 API Request

**Overview:**  
Sends a POST request to the external YouTube Transcript API (youtube-transcript.io) to request transcript data for the specified video ID.

**Nodes Involved:**  
- Get Transcript (YouTube Transcript API)

**Node Details:**  

- **Get Transcript (YouTube Transcript API)**  
  - Type: HTTP Request  
  - Role: Fetches transcript data from youtube-transcript.io API  
  - Configuration:  
    - Method: POST  
    - URL: `https://www.youtube-transcript.io/api/transcripts`  
    - Headers: Content-Type: application/json  
    - Body: JSON with `ids` array containing the video ID from `Set Variables` node (`{{ $('Set Variables').item.json.youtubeVideoId }}`)  
    - Response: Full response with raw text; prevents errors on HTTP failures (`neverError=true`)  
    - Retry: Enabled with 5 seconds wait between tries to handle transient errors  
  - Inputs: Receives video ID from Set Variables  
  - Outputs: Full HTTP response forwarded to parsing node  
  - Edge Cases: API downtime, invalid video ID, rate limits, network timeouts  
  - Version: 4.2  

---

#### 2.3 Parsing & Validation

**Overview:**  
Parses the API response safely, handling various possible response formats and extracting transcript text and language metadata.

**Nodes Involved:**  
- Parse API Response

**Node Details:**  

- **Parse API Response**  
  - Type: Code  
  - Role: Robust JSON parsing and normalization of transcript data  
  - Configuration:  
    - Uses try-catch to parse response body safely  
    - Supports multiple transcript shapes: `text`, `transcript`, or nested `tracks[].transcript[]`  
    - Extracts language info from multiple possible fields  
    - Returns structured JSON with keys: `videoId`, `language`, `text`, `status`, `source`  
  - Inputs: Full HTTP response from API Request node  
  - Outputs: Parsed transcript data or error JSON if parsing fails  
  - Edge Cases: Malformed JSON, unexpected response structure, missing transcript text  
  - Version: 2  

---

#### 2.4 Transcript Availability Decision

**Overview:**  
Evaluates whether a transcript was found in the parsed response and routes processing accordingly.

**Nodes Involved:**  
- IF Has Transcript?

**Node Details:**  

- **IF Has Transcript?**  
  - Type: If  
  - Role: Conditional routing based on presence of transcript text  
  - Configuration:  
    - Condition: `$json.text !== null && $json.text !== undefined && $json.text !== ''`  
    - True branch: transcript exists  
    - False branch: transcript missing  
  - Inputs: Parsed data from previous node  
  - Outputs: Routes to either cleaning or fallback node  
  - Edge Cases: Empty strings or whitespace-only transcripts considered missing  
  - Version: 2  

---

#### 2.5 Transcript Cleaning

**Overview:**  
Processes and normalizes the raw transcript text to produce clean output suitable for further AI or data processing.

**Nodes Involved:**  
- Clean Transcript

**Node Details:**  

- **Clean Transcript**  
  - Type: Code  
  - Role: Cleans and normalizes transcript text  
  - Configuration:  
    - Converts transcript to string if necessary  
    - Replaces newlines with spaces, collapses multiple whitespaces to single spaces  
    - Removes common music tags like `[Music]`, `[Música]` (case-insensitive)  
    - Computes `wordCount` and `charCount` metrics  
    - Returns structured JSON with cleaned text and metadata  
  - Inputs: Transcript JSON from IF node (true branch)  
  - Outputs: Clean transcript JSON  
  - Edge Cases: Non-string transcripts, transcripts with embedded tags or special characters  
  - Version: 2  

---

#### 2.6 Error Handling

**Overview:**  
Provides fallback structured response and stops workflow execution with an error if no transcript was found.

**Nodes Involved:**  
- No Transcript Fallback  
- Stop and Error

**Node Details:**  

- **No Transcript Fallback**  
  - Type: Code  
  - Role: Generates standardized error JSON when transcript is unavailable  
  - Configuration:  
    - Uses video ID from `Set Variables` node  
    - Defaults language to `es` if not specified  
    - Provides error message and suggestion to try alternative languages or fallback ASR/Whisper methods  
  - Inputs: IF node (false branch)  
  - Outputs: Error JSON to Stop node  
  - Edge Cases: Missing preferred language; fallback to Spanish used  
  - Version: 2  

- **Stop and Error**  
  - Type: Stop and Error  
  - Role: Terminates workflow with an error message  
  - Configuration: Error message: "There's no transcript for this YouTube video (Transcript API)."  
  - Inputs: From No Transcript Fallback  
  - Outputs: Workflow stops with error  
  - Edge Cases: None (terminal node)  
  - Version: 1  

---

#### 2.7 Final Output

**Overview:**  
The final output is the cleaned transcript data structured consistently for downstream use, including video ID, language, cleaned text, word and character counts, status, and source.

**Nodes Involved:**  
- Clean Transcript (final output node)

**Node Details:**  
- See Clean Transcript node above.

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                       | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                          |
|-------------------------------|--------------------------|-------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger  | Sub-workflow entry point, input reception | External trigger                 | Set Variables                   | ## 1. Input Processing - Receives execution as a sub-workflow - Defines `youtubeVideoId` and `preferredLanguage` Example: { "youtubeVideoId": "xObjAdhDxBE" }                                                                        |
| Set Variables                 | Set                      | Assigns youtubeVideoId variable     | When Executed by Another Workflow | Get Transcript (YouTube Transcript API) | ## 1. Input Processing - Receives execution as a sub-workflow - Defines `youtubeVideoId` and `preferredLanguage` Example: { "youtubeVideoId": "xObjAdhDxBE" }                                                                        |
| Get Transcript (YouTube Transcript API) | HTTP Request             | Calls youtube-transcript.io API to fetch transcript | Set Variables                   | Parse API Response             | ## 2. Transcript API Request - Calls **youtube-transcript.io** - Does not require YouTube OAuth - Returns the response (Full Response + text) - The next node performs robust parsing                                                  |
| Parse API Response            | Code                     | Parses and normalizes API response  | Get Transcript (YouTube Transcript API) | IF Has Transcript?             | ## 3. Parsing & Cleaning - Parses JSON even when the response arrives as a string (using safe parsing / try-catch) - Supports multiple response shapes (`text` / `transcript` / `tracks`) - Basic cleaning                             |
| IF Has Transcript?            | If                       | Checks for transcript presence      | Parse API Response               | Clean Transcript, No Transcript Fallback | ## ❌ Error Handling Path - When no transcript is available: - Returns a structured error (**No Transcript Fallback**) - Ends the execution with **Stop and Error** - Suggests an ASR/Whisper fallback                               |
| Clean Transcript             | Code                     | Cleans transcript text and computes metrics | IF Has Transcript? (true branch) | (Final output)                 | ## 3. Parsing & Cleaning - Parses JSON even when the response arrives as a string (using safe parsing / try-catch) - Supports multiple response shapes (`text` / `transcript` / `tracks`) - Basic cleaning                             |
| No Transcript Fallback       | Code                     | Generates error JSON when no transcript found | IF Has Transcript? (false branch) | Stop and Error                | ## ❌ Error Handling Path - When no transcript is available: - Returns a structured error (**No Transcript Fallback**) - Ends the execution with **Stop and Error** - Suggests an ASR/Whisper fallback                               |
| Stop and Error               | Stop and Error           | Terminates workflow with error      | No Transcript Fallback          | None                          | ## ❌ Error Handling Path - When no transcript is available: - Returns a structured error (**No Transcript Fallback**) - Ends the execution with **Stop and Error** - Suggests an ASR/Whisper fallback                               |
| Sticky Note - Overview       | Sticky Note              | Documentation overview              | None                           | None                          | ## 🚀 Try It Out! YouTube Transcript API Extractor (Any Public Video) - Extracts a clean transcript from a videoId using youtube-transcript.io. Use cases include AI summaries, SEO, content pipelines. Workflow steps are documented. |
| Sticky Note - Input          | Sticky Note              | Input block explanation             | None                           | None                          | ## 1. Input Processing - Receives execution as a sub-workflow - Defines `youtubeVideoId` and `preferredLanguage` Example: { "youtubeVideoId": "xObjAdhDxBE" }                                                                          |
| Sticky Note - API            | Sticky Note              | API Request block explanation       | None                           | None                          | ## 2. Transcript API Request - Calls **youtube-transcript.io** - Does not require YouTube OAuth - Returns the response (Full Response + text) - The next node performs robust parsing                                                  |
| Sticky Note - Processing     | Sticky Note              | Parsing and cleaning explanation    | None                           | None                          | ## 3. Parsing & Cleaning - Parses JSON even when the response arrives as a string (using safe parsing / try-catch) - Supports multiple response shapes (`text` / `transcript` / `tracks`) - Basic cleaning                             |
| Sticky Note - Output         | Sticky Note              | Final output format explanation     | None                           | None                          | ## ✅ Final Output Format - Success: { "videoId": "xObjAdhDxBE", "language": "es", "text": "Clean transcript text...", "wordCount": 1234, "charCount": 5678, "status": "success", "source": "youtube_transcript_api" }                   |
| Sticky Note - Errors         | Sticky Note              | Error handling path explanation     | None                           | None                          | ## ❌ Error Handling Path - When no transcript is available: - Returns a structured error (**No Transcript Fallback**) - Ends the execution with **Stop and Error** - Suggests an ASR/Whisper fallback                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add **Execute Workflow Trigger** node.
   - Name it: `When Executed by Another Workflow`.
   - Configure input parameter: `youtubeVideoId` (string).
   - This node will receive the video ID when called from another workflow.

2. **Set Variables Node:**
   - Add a **Set** node named `Set Variables`.
   - Connect it from the trigger node.
   - Assign a variable named `youtubeVideoId` with the expression `={{ $json.youtubeVideoId }}`.
   - This ensures consistent access to the video ID downstream.

3. **HTTP Request Node:**
   - Add an **HTTP Request** node named `Get Transcript (YouTube Transcript API)`.
   - Connect it from `Set Variables`.
   - Configure as follows:
     - Method: POST
     - URL: `https://www.youtube-transcript.io/api/transcripts`
     - Headers: Add `Content-Type` header with value `application/json`
     - Body Content: JSON with key `ids` containing an array with the video ID:
       ```json
       {
         "ids": ["{{ $('Set Variables').item.json.youtubeVideoId }}"]
       }
       ```
     - Set "Send Body" to true and specify body format as JSON.
     - Under Options, set Response to "Full Response" and "Never Error" to true.
     - Enable Retry on Fail with 5 seconds wait between tries.
   - No special credentials are required for this public API.

4. **Code Node for Parsing:**
   - Add a **Code** node named `Parse API Response`.
   - Connect from the HTTP Request node.
   - Paste the JavaScript code that:
     - Extracts the raw response body safely.
     - Parses JSON with try-catch.
     - Handles multiple transcript response formats.
     - Extracts transcript text and language.
     - Returns a JSON object with `videoId`, `language`, `text`, `status`, and `source`.
   - This node normalizes the API response.

5. **If Node for Transcript Check:**
   - Add an **If** node named `IF Has Transcript?`.
   - Connect from the Parse node.
   - Configure condition:
     - Expression: `{{$json.text !== null && $json.text !== undefined && $json.text !== ''}}`
     - Operator: Boolean equals true
   - True branch proceeds to transcript cleaning.
   - False branch proceeds to error handling.

6. **Code Node for Cleaning Transcript:**
   - Add a **Code** node named `Clean Transcript`.
   - Connect from the If node’s true branch.
   - Paste JavaScript code that:
     - Normalizes transcript text to string.
     - Replaces newlines with spaces.
     - Collapses multiple spaces.
     - Removes `[Music]`, `[Música]`, `[Musica]` tags case-insensitively.
     - Calculates `wordCount` and `charCount`.
     - Returns cleaned transcript JSON with metadata.
   - This node produces the final successful output.

7. **Code Node for Transcript Missing Fallback:**
   - Add a **Code** node named `No Transcript Fallback`.
   - Connect from the If node’s false branch.
   - Paste JavaScript code that:
     - Returns JSON with error message, fallback language (default `es`), and suggestion.
     - Includes `videoId`, `status: 'no_transcript'`, and `source`.

8. **Stop and Error Node:**
   - Add a **Stop and Error** node named `Stop and Error`.
   - Connect from the No Transcript Fallback node.
   - Configure error message: `There's no transcript for this YouTube video (Transcript API).`
   - This node terminates workflow execution on error.

9. **Add Sticky Note nodes:**
   - Add sticky notes with the documented content as per the original workflow, placed near relevant nodes for usability and documentation purposes.

10. **Testing:**
    - Test the workflow by triggering it with a valid `youtubeVideoId` (e.g., `xObjAdhDxBE`).
    - Verify the output JSON includes `text`, `language`, `wordCount`, `charCount`, and `status: success`.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow uses the free and public API at youtube-transcript.io, which does not require YouTube OAuth. | API Documentation at https://youtube-transcript.io             |
| Clean transcripts are useful for AI-driven content analysis such as summarization or sentiment analysis. | Use case examples include SEO, content pipelines, and indexing. |
| The workflow is designed as a sub-workflow to be called from other workflows with `youtubeVideoId` input. | Facilitates batch processing or integration with other automations. |
| Error handling suggests fallback to ASR/Whisper transcription methods if no transcript is found.          | Whisper ASR: OpenAI Whisper models or other speech-to-text services can be integrated as fallback. |

---

**Disclaimer:**  
The content provided is generated exclusively from an n8n workflow designed for legal and public data sources, respecting all applicable content policies.