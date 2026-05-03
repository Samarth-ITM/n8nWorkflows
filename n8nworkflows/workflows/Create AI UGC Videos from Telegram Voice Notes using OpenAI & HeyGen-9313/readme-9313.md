Create AI UGC Videos from Telegram Voice Notes using OpenAI & HeyGen

https://n8nworkflows.xyz/workflows/create-ai-ugc-videos-from-telegram-voice-notes-using-openai---heygen-9313


# Create AI UGC Videos from Telegram Voice Notes using OpenAI & HeyGen

### 1. Workflow Overview

This workflow automates the process of creating AI-generated User-Generated Content (UGC) videos from voice notes received via Telegram. It integrates Telegram for input capture, OpenAI for voice memo transcription, and HeyGen for video generation, finally delivering the generated video URL back to the user on Telegram.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Captures incoming voice messages from Telegram and downloads the associated audio files.
- **1.2 Transcription:** Uses OpenAI's API to transcribe the downloaded voice memo into text.
- **1.3 Video Generation:** Prepares necessary identifiers and sends the transcription text to HeyGen's API to generate a video.
- **1.4 Video Status Polling & Completion:** Periodically checks the status of the video generation process until completion.
- **1.5 Output Delivery:** Sends the final video URL back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for new Telegram voice messages and downloads the attached audio file for further processing.

**Nodes Involved:**  
- Message Trigger  
- Downloading File

**Node Details:**

- **Message Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point; listens for new messages (voice notes) on a Telegram bot webhook.  
  - *Configuration:* Uses an existing Telegram bot webhook (webhookId defined). No filters applied, triggers on any message.  
  - *Input:* Incoming Telegram messages.  
  - *Output:* Message metadata and voice file reference.  
  - *Failure Modes:* Network errors, webhook misconfiguration, unsupported message types.  
  - *Version:* 1.2

- **Downloading File**  
  - *Type:* Telegram node (file download)  
  - *Role:* Downloads the Telegram voice note file based on the incoming message data.  
  - *Configuration:* Uses Telegram bot credentials; downloads the voice file referenced in the trigger node.  
  - *Input:* File ID from the Message Trigger node.  
  - *Output:* Local or accessible URL/path of the downloaded audio file.  
  - *Failure Modes:* File not found, permission issues, download timeout.  
  - *Version:* 1.2

---

#### 2.2 Transcription

**Overview:**  
Transcribes the downloaded audio voice memo into text using OpenAI’s transcription capabilities.

**Nodes Involved:**  
- Transcribing voice memo  
- Setting ID Fields

**Node Details:**

- **Transcribing voice memo**  
  - *Type:* OpenAI node (Langchain integration)  
  - *Role:* Converts audio input into text transcription using OpenAI’s speech-to-text models.  
  - *Configuration:* Uses OpenAI credentials; audio file passed as input. Model parameters set for transcription.  
  - *Input:* Audio file from Downloading File node.  
  - *Output:* Transcribed text string.  
  - *Failure Modes:* API rate limits, invalid audio format, transcription errors, network timeouts.  
  - *Version:* 1.8

- **Setting ID Fields**  
  - *Type:* Set node  
  - *Role:* Prepares and sets identifiers and metadata required for subsequent video generation request.  
  - *Configuration:* Sets key fields such as user ID, transcription text, or other metadata needed for HeyGen API.  
  - *Input:* Transcribed text from previous node.  
  - *Output:* Structured data with IDs and transcription ready for video generation.  
  - *Failure Modes:* Expression errors if fields missing or invalid data.  
  - *Version:* 3.4

---

#### 2.3 Video Generation

**Overview:**  
Sends the transcription and identifiers to HeyGen’s API to generate an AI video.

**Nodes Involved:**  
- Generating Video  
- Video Status Update

**Node Details:**

- **Generating Video**  
  - *Type:* HTTP Request node  
  - *Role:* Initiates video generation by calling HeyGen API with required parameters (text, user ID, etc.).  
  - *Configuration:* POST request to HeyGen endpoint with JSON body containing transcription and metadata; includes authentication headers.  
  - *Input:* Prepared ID fields and transcription from Setting ID Fields.  
  - *Output:* Response containing video generation job ID or status.  
  - *Failure Modes:* API auth errors, invalid request format, network failures.  
  - *Version:* 4.2

- **Video Status Update**  
  - *Type:* HTTP Request node  
  - *Role:* Polls HeyGen API to check the status of the video generation job.  
  - *Configuration:* GET or POST request with job ID; expects status response (e.g., processing, completed).  
  - *Input:* Video job ID from Generating Video node or previous polling.  
  - *Output:* Status data used to determine next steps.  
  - *Failure Modes:* Timeout, job ID invalid, API rate limits, network issues.  
  - *Version:* 4.2

---

#### 2.4 Video Status Polling & Completion

**Overview:**  
Implements waiting and conditional logic to poll the video status until it is completed.

**Nodes Involved:**  
- is Completed  
- 10s Buffer  
- Setting Output

**Node Details:**

- **is Completed**  
  - *Type:* If node  
  - *Role:* Checks if the video generation status returned by HeyGen indicates completion.  
  - *Configuration:* Condition evaluates the status field for completion flag. Routes flow accordingly.  
  - *Input:* Status response from Video Status Update.  
  - *Output:* Two outputs: ‘true’ (completed) and ‘false’ (not completed).  
  - *Failure Modes:* Expression errors if status field missing or malformed.  
  - *Version:* 2.2

- **10s Buffer**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow execution for 10 seconds before re-polling video status.  
  - *Configuration:* Fixed wait time of 10 seconds.  
  - *Input:* Triggered if video is not completed.  
  - *Output:* Triggers next Video Status Update request.  
  - *Failure Modes:* None typical; long waits may cause execution timeouts if limits imposed.  
  - *Version:* 1.1

- **Setting Output**  
  - *Type:* Set node  
  - *Role:* Prepares output data for final delivery, such as the video URL.  
  - *Configuration:* Extracts and sets the video URL and possibly other metadata for Telegram message.  
  - *Input:* Completed video status data.  
  - *Output:* Structured output including video URL.  
  - *Failure Modes:* Expression failures if URL missing.  
  - *Version:* 3.4

---

#### 2.5 Output Delivery

**Overview:**  
Sends the generated video URL back to the Telegram user who submitted the voice note.

**Nodes Involved:**  
- Sending Video URL

**Node Details:**

- **Sending Video URL**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends a message containing the video URL to the Telegram user.  
  - *Configuration:* Uses Telegram bot credentials; sends message to user ID captured earlier; message includes video URL.  
  - *Input:* Video URL and user ID from Setting Output node.  
  - *Output:* Confirmation of message sent.  
  - *Failure Modes:* Telegram API errors, user blocked bot, message formatting issues.  
  - *Version:* 1.2

---

### 3. Summary Table

| Node Name             | Node Type                    | Functional Role              | Input Node(s)          | Output Node(s)          | Sticky Note                    |
|-----------------------|------------------------------|-----------------------------|-----------------------|-------------------------|-------------------------------|
| Message Trigger       | Telegram Trigger             | Receive Telegram voice notes | (start)               | Downloading File         |                               |
| Downloading File      | Telegram                     | Download voice memo file     | Message Trigger       | Transcribing voice memo  |                               |
| Transcribing voice memo | OpenAI (Langchain)          | Transcribe audio to text     | Downloading File      | Setting ID Fields        |                               |
| Setting ID Fields     | Set                          | Prepare metadata for video   | Transcribing voice memo| Generating Video         |                               |
| Generating Video      | HTTP Request                 | Call HeyGen API to generate video | Setting ID Fields   | Video Status Update      |                               |
| Video Status Update   | HTTP Request                 | Poll HeyGen for video status | Generating Video / 10s Buffer | is Completed         |                               |
| is Completed          | If                           | Check if video generation complete | Video Status Update | Setting Output / 10s Buffer |                               |
| 10s Buffer            | Wait                         | Delay before next status poll| is Completed (false)  | Video Status Update      |                               |
| Setting Output        | Set                          | Prepare final video URL      | is Completed (true)   | Sending Video URL        |                               |
| Sending Video URL     | Telegram                     | Send video URL message       | Setting Output        | (end)                   |                               |
| Sticky Note           | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note1          | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note2          | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note3          | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note4          | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note5          | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note6          | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note7          | Sticky Note                  | Notes / comments             |                       |                         |                               |
| Sticky Note8          | Sticky Note                  | Notes / comments             |                       |                         |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials and webhook setup.  
   - No filters; triggers on any incoming message (voice notes expected).

2. **Add Telegram Download Node**  
   - Type: Telegram  
   - Configure to download files using your Telegram credentials.  
   - Input: Use the file ID from the Message Trigger node to download the voice note.

3. **Add OpenAI Node for Transcription**  
   - Type: OpenAI (Langchain integration)  
   - Configure with OpenAI API credentials.  
   - Input: Pass the downloaded audio file.  
   - Set model parameters appropriate for audio transcription.

4. **Add Set Node to Prepare IDs and Metadata**  
   - Type: Set  
   - Configure to create fields such as user ID, transcription text, and any other metadata required by HeyGen API.

5. **Add HTTP Request Node to Generate Video**  
   - Type: HTTP Request  
   - Configure as POST to HeyGen’s video generation endpoint.  
   - Include authentication headers (API key/token).  
   - Pass transcription and metadata in the JSON body.

6. **Add HTTP Request Node for Video Status Checking**  
   - Type: HTTP Request  
   - Configure to poll HeyGen’s status endpoint with the video job ID.  
   - Use GET or POST as required by HeyGen API.

7. **Add If Node to Check Completion**  
   - Type: If  
   - Configure to evaluate the status response field for completion.  
   - Two outputs: true (completed), false (not completed).

8. **Add Wait Node for 10 Seconds Delay**  
   - Type: Wait  
   - Configure to wait 10 seconds before polling status again.  
   - Connect false output of If node to this node.

9. **Connect Wait Node to Video Status Update Node**  
   - Connect output of Wait node back to Video Status Update node for polling loop.

10. **Add Set Node to Prepare Final Output**  
    - Type: Set  
    - Extract and set the final video URL and any additional info for Telegram message.

11. **Add Telegram Node to Send Video URL**  
    - Type: Telegram  
    - Configure to send a message to the originating user with the video URL.  
    - Use Telegram credentials.

12. **Connect Nodes Accordingly**  
    - Message Trigger -> Downloading File -> Transcribing voice memo -> Setting ID Fields -> Generating Video -> Video Status Update -> is Completed  
    - is Completed (true) -> Setting Output -> Sending Video URL  
    - is Completed (false) -> 10s Buffer -> Video Status Update

13. **Credentials Setup**  
    - Telegram Bot credentials for all Telegram nodes.  
    - OpenAI API credentials for transcription.  
    - HeyGen API credentials for HTTP Request nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                         |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The workflow integrates Telegram, OpenAI, and HeyGen APIs to automate AI UGC video creation.     | Workflow purpose                                        |
| Ensure API rate limits and authentication tokens are properly managed for OpenAI and HeyGen APIs.| Best practice for API integrations                      |
| HeyGen API endpoints and payload structures should be verified with current API documentation.    | HeyGen API docs (not included here)                     |
| Telegram bot must have appropriate permissions to receive voice messages and send media messages. | Telegram Bot API documentation                          |
| Consider adding error handling for network failures and invalid input cases for robustness.      | Workflow resilience                                     |

---

*Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*