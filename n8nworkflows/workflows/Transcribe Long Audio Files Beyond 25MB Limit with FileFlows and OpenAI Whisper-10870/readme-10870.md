Transcribe Long Audio Files Beyond 25MB Limit with FileFlows and OpenAI Whisper

https://n8nworkflows.xyz/workflows/transcribe-long-audio-files-beyond-25mb-limit-with-fileflows-and-openai-whisper-10870


# Transcribe Long Audio Files Beyond 25MB Limit with FileFlows and OpenAI Whisper

### 1. Workflow Overview

This workflow enables transcription of long audio files that exceed the 25 MB size limit of the OpenAI Whisper API. It is designed for processing lengthy recordings such as meetings, podcasts, interviews, or conferences by splitting large audio files into smaller chunks, transcribing each segment individually, then merging all transcriptions into a single text file, and finally delivering the transcription to the user by email.

The workflow is logically divided into the following blocks:

- **1.1 Upload & Chunk (Stage 1):** Receives user input via a web form, chunks the uploaded audio file into manageable 4 MiB pieces, and uploads these chunks to a FileFlows server.

- **1.2 Audio Splitting (Stage 2):** Uses FileFlows to split the uploaded audio file into 15-minute segments to ensure each segment is below OpenAI’s 25 MB limit.

- **1.3 Transcription (Stage 3):** Processes each audio segment by transcribing it with OpenAI Whisper API, handles rate limits and errors, and merges all segment transcripts into a single text.

- **1.4 Delivery (Stage 4):** Converts the final transcription into a text file and sends it as an email attachment to the user. Also handles error notifications by email.

---

### 2. Block-by-Block Analysis

#### 1.1 Upload & Chunk (Stage 1)

**Overview:**  
This block handles user input from a web form, chunks the uploaded audio file into 4 MiB pieces, and uploads these chunks to a FileFlows server via HTTP requests.

**Nodes Involved:**  
- GET Form  
- Configuration  
- Make 4MiB Chunks  
- Loop Over Chunks  
- Upload Chunk  
- Result  
- Chunk  
- Filter temporary files  
- If succeed  
- Send Error1  

**Node Details:**  

- **GET Form**  
  - Type: Form Trigger  
  - Role: Entry point; receives audio file (.mp3) and user email via a web form path `/audio-transcription`.  
  - Config: Form with file and email fields; confirmation message on submission.  
  - Inputs: HTTP webhook trigger  
  - Outputs: Passes form data to Configuration node  
  - Edge cases: Missing or invalid file, unsupported file type, invalid email  
  - Version: 2.3

- **Configuration**  
  - Type: Set  
  - Role: Stores constants such as chunk size (4 MiB), FileFlows API URL, and Flow UID for splitting.  
  - Config: Assigns chunk_size = 4 * 1024 * 1024 bytes, fileflows_url, and flowUid.  
  - Inputs: GET Form output  
  - Outputs: Passes configuration to Make 4MiB Chunks  
  - Edge cases: Misconfigured URLs or UIDs will cause failures downstream.  
  - Version: 3.4

- **Make 4MiB Chunks**  
  - Type: Code  
  - Role: Splits the uploaded binary audio file into 4 MiB chunks encoded in base64 for upload.  
  - Config: Uses JavaScript to slice Buffer from binary data; respects optional filename override.  
  - Inputs: Configuration output with binary file data  
  - Outputs: Array of chunk items each containing binary chunk data and metadata (chunkNumber, totalChunks)  
  - Edge cases: No binary data received, incorrect field naming, memory issues with large files  
  - Version: 2

- **Loop Over Chunks**  
  - Type: Split In Batches  
  - Role: Processes each 4 MiB chunk sequentially to avoid overload.  
  - Config: Default batch size (typically 1)  
  - Inputs: Chunks from previous node  
  - Outputs: Single chunk to Upload Chunk node per iteration  
  - Edge cases: Batch processing failures; blocked or slow HTTP requests  

- **Upload Chunk**  
  - Type: HTTP Request  
  - Role: Uploads each chunk to FileFlows API endpoint `/api/library-file/upload` using multipart/form-data.  
  - Config: Sends chunk data and metadata (fileName, chunkNumber, totalChunks) alongside binary chunk.  
  - Inputs: Loop Over Chunks output  
  - Outputs: Response with uploaded file identifier or error detail  
  - Edge cases: Network failures, authentication issues if FileFlows requires it, malformed requests  
  - Version: 4.2

- **Result**  
  - Type: No Operation (NoOp)  
  - Role: Placeholder to capture the response from upload operations for further filtering.  
  - Inputs: Loop Over Chunks output  
  - Outputs: Passes data to Filter temporary files node  
  - Version: 1

- **Filter temporary files**  
  - Type: Filter  
  - Role: Filters out temporary files (those ending with `.temp`) from the list of uploaded files reported by FileFlows.  
  - Config: Condition excludes files with `.temp` suffix in their `data` field.  
  - Inputs: Result node output  
  - Outputs: Passes filtered valid files to If succeed node  
  - Edge cases: No files passed if all end with `.temp`—workflow will branch to error  
  - Version: 2.2

- **If succeed**  
  - Type: If  
  - Role: Checks if FileFlows responded with valid uploaded file(s) by testing existence of `data` field.  
  - Config: Condition checks if `data` exists and is not empty.  
  - Inputs: Filter temporary files output  
  - Outputs: Success branch → Split audio file node; Failure branch → Send Error1 node  
  - Edge cases: Network error or malformed response leading to false negatives  
  - Version: 2.2

- **Send Error1**  
  - Type: Gmail (Email)  
  - Role: Sends an error email to user if the file upload or splitting initiation failed.  
  - Config: Uses user’s email from form; standard error message about file splitting issue.  
  - Inputs: If succeed’s failure branch  
  - Credentials: Gmail OAuth2 configured  
  - Edge cases: Email sending failures, invalid email addresses  
  - Version: 2.1

- **Chunk**  
  - Type: NoOp  
  - Role: Placeholder node connected after Loop Over Chunks for clarity or future expansion.  
  - Version: 1

---

#### 1.2 Audio Splitting (Stage 2)

**Overview:**  
This block uses FileFlows API to initiate splitting uploaded audio files into 15-minute segments, preparing them for transcription.

**Nodes Involved:**  
- Split audio file  
- Wait  

**Node Details:**  

- **Split audio file**  
  - Type: HTTP Request  
  - Role: Calls FileFlows API `/api/library-file/manually-add` to trigger the audio splitting using a predefined flow UID and file references.  
  - Config: Sends JSON body with FlowUid, Files array containing uploaded file IDs, and a callback URL to resume workflow upon completion.  
  - Inputs: If succeed node success branch  
  - Outputs: Invokes Wait node to pause workflow until splitting completes  
  - Edge cases: Incorrect Flow UID, server unavailability, or invalid file references  
  - Version: 4.2

- **Wait**  
  - Type: Wait (Webhook Resume)  
  - Role: Waits up to 30 minutes for FileFlows to complete splitting, resumes workflow via webhook triggered by FileFlows callback.  
  - Config: Wait duration 30 minutes max, HTTP POST resume method, webhook ID linked for callback  
  - Inputs: Split audio file node  
  - Outputs: Passes audio file segments to Split Audio node  
  - Edge cases: Timeout if callback not received, webhook misconfiguration  
  - Version: 1.1

---

#### 1.3 Transcription (Stage 3)

**Overview:**  
This block processes each 15-minute audio segment: splitting the batch, transcribing with OpenAI Whisper API, handling rate limits and errors, and finally merging all segment transcripts.

**Nodes Involved:**  
- Split Audio  
- Loop Over Segments  
- OpenAI  
- Rate Limit Delay  
- Send Error  
- Result transcription  
- Segment  
- Merge transcription  

**Node Details:**  

- **Split Audio**  
  - Type: Code  
  - Role: Converts binary segment data from webhook resume into individual items, preserving full binary structure.  
  - Config: Iterates over all binary entries, emits one item per audio segment with metadata and binary data under the property "Audio".  
  - Inputs: Wait node output  
  - Outputs: Array of audio segment items to Loop Over Segments  
  - Edge cases: Binary data missing or malformed, empty input  
  - Version: 2

- **Loop Over Segments**  
  - Type: Split In Batches  
  - Role: Processes audio segments sequentially to avoid API rate limits or memory overload.  
  - Inputs: Split Audio output  
  - Outputs: One audio segment item passed to OpenAI node per iteration  
  - Edge cases: Batch processing delays or failures  
  - Version: 3

- **OpenAI**  
  - Type: OpenAI (LangChain)  
  - Role: Transcribes each audio segment using OpenAI Whisper API.  
  - Config: Language set to French (`fr`), operation `transcribe`, binary input property named "Audio".  
  - Inputs: Loop Over Segments output  
  - Outputs: Transcription text for each segment  
  - Credentials: OpenAI API key required  
  - Edge cases: API rate limits, network failures, auth errors, transcription errors  
  - Version: 1.8  
  - On error: Continues with error output branch to Send Error node

- **Rate Limit Delay**  
  - Type: Wait  
  - Role: Adds delay after each transcription to respect OpenAI API rate limits.  
  - Inputs: OpenAI main success output  
  - Outputs: Loops back to Loop Over Segments for next batch  
  - Edge cases: Delay too short may cause throttling; too long delays workflow unnecessarily  
  - Version: 1.1

- **Send Error**  
  - Type: Gmail (Email)  
  - Role: Sends error email to user if transcription fails due to model or API issues.  
  - Inputs: OpenAI error output  
  - Credentials: Gmail OAuth2 required  
  - Edge cases: Email sending failure, invalid email address  
  - Version: 2.1

- **Result transcription**  
  - Type: NoOp  
  - Role: Collects all successful transcription outputs for final merging.  
  - Inputs: Loop Over Segments success output  
  - Outputs: Passes to Merge transcription node  
  - Version: 1

- **Segment**  
  - Type: NoOp  
  - Role: Placeholder node connected to OpenAI node output for clarity or downstream processing.  
  - Version: 1

- **Merge transcription**  
  - Type: Code  
  - Role: Concatenates the `text` field from all segment transcriptions into one single string.  
  - Config: Loops over all input items, concatenates the `json.text` fields, returns resulting transcription string under `transcription` property.  
  - Inputs: Result transcription node  
  - Outputs: Passes merged transcription to Convert to File node  
  - Edge cases: Missing or incorrect transcription text fields, empty segments  
  - Version: 2

---

#### 1.4 Delivery (Stage 4)

**Overview:**  
This block converts the merged transcription string to a text file and emails it to the user. It also manages error notifications.

**Nodes Involved:**  
- Convert to File  
- Send Email with Transcription  
- Send Error (also used in Stage 3)  

**Node Details:**  

- **Convert to File**  
  - Type: Convert To File  
  - Role: Converts the merged transcription text string into a UTF-8 encoded `.txt` file named `transcription.txt`.  
  - Config: Source property is `transcription` from previous node output  
  - Inputs: Merge transcription output  
  - Outputs: Binary file attachment to email node  
  - Version: 1.1

- **Send Email with Transcription**  
  - Type: Gmail (Email)  
  - Role: Sends an email to the user with the transcription text file attached.  
  - Config: Uses email from form submission, message body informing completion, subject line.  
  - Credentials: Gmail OAuth2 configured  
  - Inputs: Convert to File output  
  - Edge cases: Email sending failure, invalid email, attachment issues  
  - Version: 2.1

- **Send Error**  
  - (Described above) Used also here to notify user of transcription model errors.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                         | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                                                      |
|----------------------------|----------------------------|---------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| GET Form                   | Form Trigger               | Entry point; gets audio file & email  | (Webhook)                   | Configuration                  |                                                                                                                                |
| Configuration              | Set                        | Holds configs: chunk size, URLs, UIDs | GET Form                    | Make 4MiB Chunks               |                                                                                                                                |
| Make 4MiB Chunks           | Code                       | Splits uploaded file into 4 MiB chunks| Configuration               | Loop Over Chunks               |                                                                                                                                |
| Loop Over Chunks           | Split In Batches           | Processes chunks sequentially          | Make 4MiB Chunks            | Upload Chunk, Chunk            |                                                                                                                                |
| Upload Chunk               | HTTP Request               | Uploads each chunk to FileFlows        | Loop Over Chunks            | Result                        |                                                                                                                                |
| Result                     | NoOp                       | Placeholder for upload response        | Upload Chunk                | Filter temporary files         |                                                                                                                                |
| Filter temporary files     | Filter                     | Filters out temporary `.temp` files    | Result                      | If succeed                    |                                                                                                                                |
| If succeed                 | If                         | Checks if upload succeeded              | Filter temporary files      | Split audio file, Send Error1  |                                                                                                                                |
| Send Error1                | Gmail                      | Error notification for upload/split   | If succeed (fail branch)    | -                            |                                                                                                                                |
| Chunk                      | NoOp                       | Placeholder after Loop Over Chunks     | Loop Over Chunks            | -                            |                                                                                                                                |
| Split audio file           | HTTP Request               | Triggers FileFlows audio splitting     | If succeed (success branch) | Wait                         |                                                                                                                                |
| Wait                       | Wait (Webhook Resume)      | Waits for splitting completion         | Split audio file            | Split Audio                   |                                                                                                                                |
| Split Audio                | Code                       | Formats binary segments for processing | Wait                       | Loop Over Segments            |                                                                                                                                |
| Loop Over Segments         | Split In Batches           | Processes segments sequentially        | Split Audio                 | OpenAI, Result transcription  |                                                                                                                                |
| OpenAI                     | OpenAI (LangChain)         | Transcribes each segment                | Loop Over Segments          | Rate Limit Delay, Send Error  |                                                                                                                                |
| Rate Limit Delay           | Wait                       | Rate-limits API calls                   | OpenAI                     | Loop Over Segments            |                                                                                                                                |
| Send Error                 | Gmail                      | Error notification for transcription   | OpenAI (error branch)       | -                            |                                                                                                                                |
| Result transcription       | NoOp                       | Collects transcription results         | Loop Over Segments          | Merge transcription           |                                                                                                                                |
| Segment                    | NoOp                       | Placeholder after OpenAI                | OpenAI                     | -                            |                                                                                                                                |
| Merge transcription        | Code                       | Concatenates all segment transcriptions| Result transcription        | Convert to File               |                                                                                                                                |
| Convert to File            | Convert To File            | Converts text to .txt file              | Merge transcription         | Send Email with Transcription |                                                                                                                                |
| Send Email with Transcription | Gmail                   | Emails final transcription to user     | Convert to File             | -                            |                                                                                                                                |
| Overview                   | Sticky Note                | Documentation on purpose and usage     | -                           | -                            | Long-Form Audio Transcription: problem, solution, stages, use cases, docs: https://github.com/JulienDelRio/My-Interesting-n8n-Workflows/tree/main/Full%20audio%20transcription%20with%20FileFlows%20and%20OpenAI |
| Stage 1 Upload             | Sticky Note                | Describes upload & chunk stage         | -                           | -                            |                                                                                                                                |
| Stage 2 Splitting          | Sticky Note                | Describes audio splitting with FileFlows| -                           | -                            |                                                                                                                                |
| Stage 3 Transcription      | Sticky Note                | Describes transcription with OpenAI   | -                           | -                            |                                                                                                                                |
| Stage 4 Delivery           | Sticky Note                | Describes packaging and email delivery | -                           | -                            |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `GET Form` node**  
   - Type: Form Trigger  
   - Configure webhook path: `audio-transcription`  
   - Add form fields: file (accept `.mp3`, required), email (required)  
   - Configure confirmation message: "Your file has been received; an email will be sent to you upon completion of transcription or in case of error."

2. **Create `Configuration` node**  
   - Type: Set  
   - Add variables:  
     - `chunk_size`: Number, value = 4 * 1024 * 1024 (4 MiB)  
     - `fileflows_url`: String, e.g., `http://172.19.0.2:5000` (replace with your FileFlows URL)  
     - `flowUid`: String, e.g., `65fad732-0ac3-4f6d-ac10-2d31ef84c154` (replace with your FileFlows flow UID)

3. **Connect `GET Form` → `Configuration`**

4. **Create `Make 4MiB Chunks` node**  
   - Type: Code  
   - Paste JS code that:  
     - Extracts binary file from form data  
     - Slices file into 4 MiB chunks (based on `chunk_size` from Configuration)  
     - Outputs array of chunk items with metadata and base64-encoded chunk binary  
   - Connect `Configuration` → `Make 4MiB Chunks`

5. **Create `Loop Over Chunks` node**  
   - Type: Split In Batches  
   - Accept default batch size (1)  
   - Connect `Make 4MiB Chunks` → `Loop Over Chunks`

6. **Create `Upload Chunk` node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `={{ $('Configuration').item.json.fileflows_url }}/api/library-file/upload`  
   - Content-Type: multipart/form-data  
   - Body parameters:  
     - `fileName` = `{{$json["fileName"]}}`  
     - `chunkNumber` = `{{$json["chunkNumber"]}}`  
     - `totalChunks` = `{{$json["totalChunks"]}}`  
     - `file` = binary chunk data (parameterType: formBinaryData, inputDataFieldName: "chunk")  
   - Connect `Loop Over Chunks` → `Upload Chunk`

7. **Create `Result` node**  
   - Type: NoOp  
   - Connect `Upload Chunk` → `Result`

8. **Create `Filter temporary files` node**  
   - Type: Filter  
   - Condition: Exclude files whose `data` field ends with `.temp`  
   - Connect `Result` → `Filter temporary files`

9. **Create `If succeed` node**  
   - Type: If  
   - Condition: Check if `$json.data` exists and is not empty (string exists)  
   - Connect `Filter temporary files` → `If succeed`

10. **Create `Send Error1` node**  
    - Type: Gmail  
    - Configure Gmail OAuth2 credentials  
    - Recipient: `={{ $('GET Form').first().json.email }}`  
    - Subject: "Your transcription encountered an issue"  
    - Message: "Hi,\n\nWe encountered an issue to split your file.\nPlease retry in a moment.\n\nBest"  
    - Connect `If succeed` failure branch → `Send Error1`

11. **Create `Split audio file` node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `={{ $('Configuration').item.json.fileflows_url }}/api/library-file/manually-add`  
    - Body type: JSON  
    - JSON Body:  
      ```json
      {
        "FlowUid": "{{ $('Configuration').first().json.flowUid }}",
        "Files": ["{{ $json.data }}"],
        "CustomVariables": {
          "callbackUrl": "{{$execution.resumeUrl}}"
        }
      }
      ```  
    - Connect `If succeed` success branch → `Split audio file`

12. **Create `Wait` node**  
    - Type: Wait (Webhook Resume)  
    - Configure webhook ID and resume method POST  
    - Resume time max 30 minutes  
    - Connect `Split audio file` → `Wait`

13. **Create `Split Audio` node**  
    - Type: Code  
    - JS code to iterate over binary files and output one item per audio segment with binary property "Audio"  
    - Connect `Wait` → `Split Audio`

14. **Create `Loop Over Segments` node**  
    - Type: Split In Batches  
    - Accept default batch size (1)  
    - Connect `Split Audio` → `Loop Over Segments`

15. **Create `OpenAI` node**  
    - Type: OpenAI (LangChain)  
    - Operation: Transcribe audio  
    - Language: French (`fr`) or remove for auto-detection  
    - Binary Property Name: `Audio`  
    - Credentials: Configure OpenAI API credentials  
    - Connect `Loop Over Segments` → `OpenAI`

16. **Create `Rate Limit Delay` node**  
    - Type: Wait  
    - Default delay (e.g., a few seconds) to avoid API throttling  
    - Connect `OpenAI` success output → `Rate Limit Delay`  
    - Connect `Rate Limit Delay` → `Loop Over Segments` (loop for next batch)

17. **Create `Send Error` node**  
    - Type: Gmail  
    - Recipient: `={{ $('GET Form').first().json.email }}`  
    - Subject: "Your transcription encountered an issue"  
    - Message: "Hi,\n\nWe encountered an issue with the translation model.\nPlease retry in a moment.\n\nBest"  
    - Credentials: Gmail OAuth2 configured  
    - Connect `OpenAI` error output → `Send Error`

18. **Create `Result transcription` node**  
    - Type: NoOp  
    - Connect `Loop Over Segments` success output → `Result transcription`

19. **Create `Merge transcription` node**  
    - Type: Code  
    - JS code to concatenate all `json.text` fields from input items into a single string `transcription`  
    - Connect `Result transcription` → `Merge transcription`

20. **Create `Convert to File` node**  
    - Type: Convert To File  
    - Operation: toText  
    - Source Property: `transcription`  
    - FileName: `transcription.txt`  
    - Encoding: UTF-8  
    - Connect `Merge transcription` → `Convert to File`

21. **Create `Send Email with Transcription` node**  
    - Type: Gmail  
    - Recipient: `={{ $('GET Form').first().json.email }}`  
    - Subject: "Your transcription is ready"  
    - Message: "Hi,\n\nYour audio transcription is complete and attached to this email.\n\nBest regards"  
    - Attachments: Use binary output from Convert to File  
    - Credentials: Gmail OAuth2 configured  
    - Connect `Convert to File` → `Send Email with Transcription`

22. **Add Sticky Notes** (optional) for documentation and clarity per stage, referencing provided content and links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Long-Form Audio Transcription overcomes OpenAI Whisper 25 MB limit by chunking, splitting, transcribing, merging, and emailing. Use cases include meetings, podcasts, and interviews. Documentation and FileFlows workflow available at https://github.com/JulienDelRio/My-Interesting-n8n-Workflows/tree/main/Full%20audio%20transcription%20with%20FileFlows%20and%20OpenAI | Main overview and usage instructions in sticky note `Overview`. FileFlows workflow for splitting audio: https://github.com/JulienDelRio/My-Interesting-n8n-Workflows/blob/main/Full%20audio%20transcription%20with%20FileFlows%20and%20OpenAI/FileFlows%20-%20Split%20audio%20for%20n8n.json |
| FileFlows setup requires installation of FFmpeg and configuration of storage for media segments (e.g., `/media/segments/`).                                                                                                                                                                                                                         | Refer to workflow sticky notes and GitHub repo for setup guidance.                                                                                                                                                      |
| Gmail OAuth2 credentials must be configured and assigned to all Gmail nodes for email delivery and error notifications.                                                                                                                                                                                                                            | Credentials setup required in n8n.                                                                                                                                                                                      |
| OpenAI API key must be configured and assigned to the OpenAI node. Language defaults to French (`fr`) but can be adjusted or removed for automatic language detection.                                                                                                                                                                            | Credentials setup required in n8n.                                                                                                                                                                                      |

---

This structured documentation fully describes the workflow’s logic, enabling advanced users and automation agents to understand, reproduce, and maintain the transcription pipeline efficiently.