Generate AI Photos with Gemini & Auto-Post to Facebook, Instagram & X with Approval

https://n8nworkflows.xyz/workflows/generate-ai-photos-with-gemini---auto-post-to-facebook--instagram---x-with-approval-11138


# Generate AI Photos with Gemini & Auto-Post to Facebook, Instagram & X with Approval

### 1. Workflow Overview

This n8n workflow automates the creation and publication of AI-generated photos with social media posts, integrating Telegram for user interaction, Google Gemini for image generation, OpenAI for AI content creation and voice transcription, Google Drive and Sheets for storage, and Blotato API for posting on Facebook, Instagram, and X (Twitter). It includes an approval loop to refine AI-generated content interactively before publishing.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**  
  Captures user input from Telegram via text messages or voice notes, converting voice to text.

- **1.2 AI Content Creation and Approval**  
  Uses OpenAI GPT-4 and LangChain to generate video prompts and social media text, supports iterative refinement based on user feedback, and detects final approval.

- **1.3 Data Storage and Image Generation**  
  Saves approved prompts and social media text to Google Sheets, generates AI photos using Google Gemini API.

- **1.4 User Approval of Generated Image**  
  Sends the generated photo to the user via Telegram, requests approval to proceed.

- **1.5 Media Upload and Social Posting**  
  Uploads approved images to Google Drive and Blotato, then creates posts on Facebook, Instagram, and X.

- **1.6 Post Status Monitoring and User Notification**  
  Checks posting status for each platform, waits and retries if necessary, and notifies the user of publishing success or errors.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives messages from Telegram users, determining whether input is text or voice. If voice, downloads the voice file and transcribes it to text using OpenAI Whisper.

**Nodes Involved:**  
- Listen for incoming events  
- Voice or Text  
- A Voice?  
- Get Voice File  
- Speech to Text

**Node Details:**

- **Listen for incoming events**  
  - Type: Telegram Trigger  
  - Role: Entry point capturing Telegram messages (text or voice)  
  - Config: Listens to "message" updates, uses Telegram API credentials  
  - Inputs: Telegram webhook  
  - Outputs: Raw Telegram message JSON  
  - Edge Cases: Missing message, unsupported message types, webhook failures

- **Voice or Text**  
  - Type: Set  
  - Role: Extract text from message or empty string if voice  
  - Config: Sets `text` field to message text if available or empty string  
  - Inputs: Telegram message JSON  
  - Outputs: JSON with `text` field  
  - Edge Cases: Empty or malformed messages

- **A Voice?**  
  - Type: If  
  - Role: Checks if message text is empty (indicating voice message)  
  - Config: Condition checks if `message.text` is empty string  
  - Inputs: JSON with `text` field  
  - Outputs: Two branches: Yes (voice), No (text)  
  - Edge Cases: False negatives if text empty for other reasons

- **Get Voice File**  
  - Type: Telegram Node  
  - Role: Retrieves voice file from Telegram using file ID  
  - Config: Uses file ID from the incoming message's voice property  
  - Inputs: Telegram message JSON with voice file ID  
  - Outputs: Binary audio file  
  - Edge Cases: File not found, API errors

- **Speech to Text**  
  - Type: OpenAI (Audio Transcription)  
  - Role: Transcribes voice audio to text using OpenAI Whisper  
  - Config: Audio resource, "transcribe" operation, uses OpenAI API credentials  
  - Inputs: Binary audio file from Telegram  
  - Outputs: Transcribed text JSON  
  - Edge Cases: Poor audio quality, API errors, rate limits

---

#### 1.2 AI Content Creation and Approval

**Overview:**  
Processes user text input with LangChain and OpenAI GPT-4 to generate video prompts and social media texts. Supports iterative refinement through conversational memory. Detects user approval and extracts final JSON output.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Window Buffer Memory  
- Approved from user?  
- Parse AI Output  
- Send questions or proposal to user  
- Inform user about processing

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Conversational AI generating video prompt and social media text, handles refinement requests  
  - Config: Uses system message instructing the bot to analyze, create, and refine prompts, only outputs final JSON on approval  
  - Inputs: Text input from user (transcribed or typed)  
  - Outputs: Text response or JSON with `videoPrompt` and `socialMediaText` fields  
  - Edge Cases: Unexpected user input, JSON parsing errors, incomplete responses

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat  
  - Role: Provides GPT-4.1-mini model for AI Agent  
  - Config: Model set to "gpt-4.1-mini" with OpenAI credentials  
  - Inputs: Prompt from AI Agent  
  - Outputs: Generated text for AI Agent  
  - Edge Cases: API errors, timeouts, rate limits

- **Window Buffer Memory**  
  - Type: LangChain Memory Buffer (Window)  
  - Role: Maintains conversational context up to 10 previous exchanges keyed by Telegram user ID  
  - Config: Uses Telegram user ID as session key  
  - Inputs: Conversation history  
  - Outputs: Context for AI Agent  
  - Edge Cases: Memory overflow, session key errors

- **Approved from user?**  
  - Type: If  
  - Role: Detects if AI Agent output contains final JSON ("videoPrompt" or "socialMediaText") indicating user approval  
  - Config: Condition checks if output contains keywords or starts with '{'  
  - Inputs: AI Agent output text  
  - Outputs: Yes (approved), No (needs refinement)  
  - Edge Cases: False positives, malformed JSON

- **Parse AI Output**  
  - Type: Code  
  - Role: Extracts and parses JSON from AI Agent output text, ensuring presence of required fields  
  - Config: JavaScript code cleans markdown, extracts JSON, validates fields  
  - Inputs: AI Agent output text  
  - Outputs: Parsed JSON with `videoPrompt` and `socialMediaText`  
  - Edge Cases: Parsing failure, missing fields, malformed JSON

- **Send questions or proposal to user**  
  - Type: Telegram Node  
  - Role: Sends AI-generated text or proposal back to user for feedback or approval  
  - Config: Sends AI Agent output text to Telegram chat  
  - Inputs: AI Agent output text, Telegram chat ID  
  - Outputs: Telegram message sent  
  - Edge Cases: Telegram API errors, message formatting issues

- **Inform user about processing**  
  - Type: Telegram Node  
  - Role: Notifies user that video/photo generation is starting after approval  
  - Config: Sends fixed message "Okay. Your video is being prepared now..."  
  - Inputs: Telegram chat ID  
  - Outputs: Telegram message sent  
  - Edge Cases: Telegram API errors

---

#### 1.3 Data Storage and Image Generation

**Overview:**  
Stores approved prompts and social media text in Google Sheets, generates AI images using Google Gemini API.

**Nodes Involved:**  
- Save Prompt & Post-Text  
- Generate an image

**Node Details:**

- **Save Prompt & Post-Text**  
  - Type: Google Sheets  
  - Role: Appends approved `videoPrompt` and `socialMediaText` to a specific Google Sheet for record-keeping  
  - Config: Defines columns "prompt" and "text", appends data to sheet with gid=0 in a specified Google Sheets document  
  - Inputs: Parsed AI output JSON  
  - Outputs: Confirmation of append operation  
  - Edge Cases: Credential expiration, rate limits, sheet access errors

- **Generate an image**  
  - Type: Google Gemini (LangChain node)  
  - Role: Generates AI photo based on approved `videoPrompt` using Gemini-2.5 Flash Image model  
  - Config: Uses `videoPrompt` as prompt, specified Gemini model, Google Palm API credentials  
  - Inputs: Prompt text from Google Sheets output (via workflow connections)  
  - Outputs: Binary image data and URL  
  - Edge Cases: API errors, model unavailability, prompt errors

---

#### 1.4 User Approval of Generated Image

**Overview:**  
Sends the generated image to the user on Telegram for approval and waits for user confirmation before proceeding.

**Nodes Involved:**  
- Send a photo message  
- Send message and wait for response  
- If1

**Node Details:**

- **Send a photo message**  
  - Type: Telegram  
  - Role: Sends generated image as photo message to Telegram user  
  - Config: Sends binary image data, targets user's chat ID  
  - Inputs: Image binary data, Telegram chat ID  
  - Outputs: Telegram message sent  
  - Edge Cases: Image size limits, Telegram errors

- **Send message and wait for response**  
  - Type: Telegram  
  - Role: Sends "Good?" message and waits for user's approval or changes  
  - Config: Uses Telegram "sendAndWait" with double approval option  
  - Inputs: Telegram chat ID  
  - Outputs: User's reply text  
  - Edge Cases: User fails to respond, timeouts, unexpected answers

- **If1**  
  - Type: If  
  - Role: Checks if user approved image (boolean `approved` in response)  
  - Config: Condition checks `data.approved == true`  
  - Inputs: User response JSON  
  - Outputs: Yes (approved), No (reject or retry)  
  - Edge Cases: Missing approval flag, ambiguous responses

---

#### 1.5 Media Upload and Social Posting

**Overview:**  
Uploads approved images to Google Drive and Blotato, then creates posts on Facebook, Instagram, and X with the approved text and media URL.

**Nodes Involved:**  
- Upload image1  
- Download image from Drive  
- Upload media1  
- Create instagram Post  
- Create x post  
- Create FB post

**Node Details:**

- **Upload image1**  
  - Type: Google Drive  
  - Role: Uploads generated image binary to specific Google Drive folder  
  - Config: Uploads to folder ID "1RA8NwbhB8Ts1A96FO5zCA4Ta_WJglJyI" with name "data"  
  - Inputs: Image binary data  
  - Outputs: File metadata including file ID  
  - Edge Cases: Drive quota limits, permissions issues

- **Download image from Drive**  
  - Type: Google Drive  
  - Role: Downloads image file from Google Drive to get binary data for upload to Blotato  
  - Config: Downloads file by ID from previous upload  
  - Inputs: File ID from Upload image1  
  - Outputs: Binary image data  
  - Edge Cases: File not found, API errors

- **Upload media1**  
  - Type: Blotato Media Upload  
  - Role: Uploads image binary to Blotato media library  
  - Config: Uses binary data property from previous node  
  - Inputs: Binary image data  
  - Outputs: Post submission ID and media URL for posting  
  - Edge Cases: API errors, upload failures

- **Create instagram Post**  
  - Type: Blotato Post Creation  
  - Role: Creates Instagram post using Blotato API with social media text and uploaded media URL  
  - Config: Uses Instagram account ID, post content text, media URL  
  - Inputs: Media URL from Upload media1, social media text from AI output  
  - Outputs: Submission status  
  - Edge Cases: API errors, posting limits

- **Create x post**  
  - Type: Blotato Post Creation  
  - Role: Creates X (Twitter) post similarly with provided text and media  
  - Config: Uses Twitter account ID, post text, media URL  
  - Inputs: Same as Instagram post  
  - Outputs: Submission status  
  - Edge Cases: API errors, rate limits

- **Create FB post**  
  - Type: Blotato Post Creation  
  - Role: Creates Facebook post with text and media on specified Facebook page  
  - Config: Uses Facebook account and page IDs, post content text, media URL  
  - Inputs: Same as other posts  
  - Outputs: Submission status  
  - Edge Cases: API errors, page permissions

---

#### 1.6 Post Status Monitoring and User Notification

**Overview:**  
Monitors the publishing status on all three platforms, waits and retries if posts are "in-progress," and finally notifies the user of successful publishing or errors.

**Nodes Involved:**  
- Wait2  
- Check Post Status1  
- Published to Facebook ?  
- Confirm publishing to Facebook  
- Send Facebook Error Message  
- Wait1  
- Get post1  
- Published to Instagram?  
- Confirm publishing to Instagram  
- Send Instagram Error Message  
- Wait  
- Check Post Status  
- Published to X?  
- Confirm publishing to X  
- Send X Error Message  
- In Progress?  
- Give Blotat other 5s :)  
- In Progress?1  
- Give Blotat more time  
- In Progress?2  
- Give Blotat other 5s :)1

**Node Details:**

- **Wait2, Wait1, Wait**  
  - Type: Wait  
  - Role: Delays execution to allow Blotato time to process posting  
  - Config: Various wait times (e.g., 10s, 30s)  
  - Inputs: Post submission IDs  
  - Outputs: Triggers status check nodes  
  - Edge Cases: Wait too short or too long affecting responsiveness

- **Check Post Status1, Check Post Status, Get post1**  
  - Type: Blotato API (Get)  
  - Role: Retrieves current status of post submissions by postSubmissionId  
  - Config: Uses postSubmissionId from previous nodes  
  - Inputs: Submission IDs  
  - Outputs: JSON with status field (e.g., "in-progress", "published")  
  - Edge Cases: API errors, invalid IDs

- **Published to Facebook ?, Published to Instagram?, Published to X?**  
  - Type: If  
  - Role: Checks if post status is "published"  
  - Config: Condition equals "published" on status field  
  - Inputs: Post status JSON  
  - Outputs: Yes (published), No ("in-progress" or error) branches  
  - Edge Cases: Unexpected status values

- **Confirm publishing to Facebook, Confirm publishing to Instagram, Confirm publishing to X**  
  - Type: Telegram  
  - Role: Sends confirmation message with public URL of the published post to user  
  - Config: Message templates with post URL, sends to user's Telegram chat  
  - Inputs: Published post JSON with publicUrl, chat ID  
  - Outputs: Telegram messages sent  
  - Edge Cases: Telegram API errors, missing URLs

- **Send Facebook Error Message, Send Instagram Error Message, Send X Error Message**  
  - Type: Telegram  
  - Role: Notifies user of errors during posting process  
  - Config: Sends fixed error message to Telegram chat  
  - Inputs: Chat ID  
  - Outputs: Telegram messages sent  
  - Edge Cases: API errors

- **In Progress?, In Progress?1, In Progress?2**  
  - Type: If  
  - Role: Checks if status is "in-progress" to continue wait/retry loop  
  - Config: Condition equals "in-progress"  
  - Inputs: Post status JSON  
  - Outputs: Yes (wait and retry), No (error or success paths)  
  - Edge Cases: Infinite loops if status never changes, API failures

- **Give Blotat other 5s :), Give Blotat more time, Give Blotat other 5s :)1**  
  - Type: Wait  
  - Role: Additional wait to allow Blotato more time before re-checking status  
  - Config: Default wait time (usually 5 seconds)  
  - Inputs: Triggered after "in-progress" detection  
  - Outputs: Triggers next status check  
  - Edge Cases: Excessive waiting delaying user notification

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                                | Input Node(s)                 | Output Node(s)                                       | Sticky Note                                                                                                                   |
|-------------------------------|-------------------------------------|-----------------------------------------------|------------------------------|-----------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Listen for incoming events     | Telegram Trigger                    | Receives Telegram messages                     | -                            | Voice or Text                                       | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper                       |
| Voice or Text                 | Set                                 | Extracts text or empty string from message    | Listen for incoming events    | A Voice?                                            | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper                       |
| A Voice?                     | If                                  | Checks if input is voice or text               | Voice or Text                | Get Voice File (voice) / AI Agent (text)            | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper                       |
| Get Voice File                | Telegram                            | Downloads voice file                           | A Voice?                     | Speech to Text                                      | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper                       |
| Speech to Text               | OpenAI (Audio Transcription)        | Transcribes voice audio to text                | Get Voice File               | AI Agent                                           | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper                       |
| AI Agent                     | LangChain Agent                     | Processes input, generates/refines prompts     | Speech to Text / A Voice?    | Approved from user?                                | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory             |
| OpenAI Chat Model            | LangChain OpenAI Chat               | Provides GPT-4 model for AI Agent              | AI Agent                     | AI Agent                                           | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory             |
| Window Buffer Memory         | LangChain Memory Buffer             | Maintains conversation context                 | Listen for incoming events   | AI Agent                                           | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory             |
| Approved from user?          | If                                 | Detects final approval from AI Agent output    | AI Agent                     | Parse AI Output / Send questions or proposal to user | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Parse AI Output              | Code                               | Extracts and validates JSON from AI output     | Approved from user?          | Inform user about processing                        | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Send questions or proposal to user | Telegram                      | Sends AI-generated text for feedback           | Approved from user?          | -                                                   | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Inform user about processing | Telegram                           | Notifies user that generation is starting      | Parse AI Output              | Save Prompt & Post-Text                            | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Save Prompt & Post-Text      | Google Sheets                     | Saves approved prompts and social media text  | Inform user about processing | Generate an image                                  | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Generate an image            | Google Gemini (LangChain node)   | Generates AI photo from prompt                  | Save Prompt & Post-Text      | Upload image1, Send a photo message                | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Send a photo message         | Telegram                         | Sends generated image to user                   | Generate an image            | Send message and wait for response                 | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Send message and wait for response | Telegram                   | Requests user approval on generated image      | Send a photo message         | If1                                                | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| If1                         | If                               | Checks if user approved the image               | Send message and wait for response | Upload media1                                      | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Upload image1               | Google Drive                    | Uploads approved image to Google Drive          | If1                         | Download image from Drive                           | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Download image from Drive   | Google Drive                    | Downloads image binary from Google Drive        | Upload image1               | If1                                                | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Upload media1               | Blotato Media Upload           | Uploads image to Blotato media library           | If1 / Download image from Drive | Create instagram Post, Create x post, Create FB post | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Create instagram Post       | Blotato Post Creation          | Creates Instagram post using Blotato            | Upload media1               | Wait1                                              | ## 6. Post the Photo  on Instagram and monitor the status                                                                       |
| Create x post               | Blotato Post Creation          | Creates X (Twitter) post                         | Upload media1               | Wait                                               | ## 6. Post the Photo on X and monitor the status                                                                                 |
| Create FB post              | Blotato Post Creation          | Creates Facebook post                            | Upload media1               | Wait2                                              | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Wait2                       | Wait                          | Waits 30 seconds before checking Facebook post status | Create FB post              | Check Post Status1                                 | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Check Post Status1          | Blotato API (Get)              | Checks Facebook post status                      | Wait2                       | Published to Facebook ?                            | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Published to Facebook ?     | If                            | Checks if Facebook post is published            | Check Post Status1          | Confirm publishing to Facebook / In Progress?2    | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Confirm publishing to Facebook | Telegram                    | Notifies user of Facebook post URL               | Published to Facebook ?     | -                                                  | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Send Facebook Error Message | Telegram                      | Notifies user of Facebook posting error          | In Progress?2               | -                                                  | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Wait1                       | Wait                          | Waits 10 seconds before checking Instagram post status | Create instagram Post       | Get post1                                          | ## 7. Post the Photo  on Instagram and monitor the status                                                                         |
| Get post1                   | Blotato API (Get)              | Checks Instagram post status                     | Wait1                       | Published to Instagram?                            | ## 7. Post the Photo  on Instagram and monitor the status                                                                         |
| Published to Instagram?     | If                            | Checks if Instagram post is published            | Get post1                   | Confirm publishing to Instagram / In Progress?1   | ## 7. Post the Photo  on Instagram and monitor the status                                                                         |
| Confirm publishing to Instagram | Telegram                   | Notifies user of Instagram post URL               | Published to Instagram?     | -                                                  | ## 7. Post the Photo  on Instagram and monitor the status                                                                         |
| Send Instagram Error Message | Telegram                     | Notifies user of Instagram posting error          | In Progress?1               | -                                                  | ## 7. Post the Photo  on Instagram and monitor the status                                                                         |
| Wait                        | Wait                          | Waits 30 seconds before checking X post status   | Create x post               | Check Post Status                                  | ## 6. Post the Photo on X and monitor the status                                                                                 |
| Check Post Status           | Blotato API (Get)              | Checks X post status                             | Wait                        | Published to X?                                   | ## 6. Post the Photo on X and monitor the status                                                                                 |
| Published to X?             | If                            | Checks if X post is published                     | Check Post Status           | Confirm publishing to X / In Progress?             | ## 6. Post the Photo on X and monitor the status                                                                                 |
| Confirm publishing to X     | Telegram                      | Notifies user of X post URL                        | Published to X?             | -                                                  | ## 6. Post the Photo on X and monitor the status                                                                                 |
| Send X Error Message        | Telegram                      | Notifies user of X posting error                   | In Progress?                | -                                                  | ## 6. Post the Photo on X and monitor the status                                                                                 |
| In Progress?                | If                            | Checks if X post status is "in-progress"          | Published to X?             | Give Blotat other 5s :) / Send X Error Message    | ## 6. Post the Photo on X and monitor the status                                                                                 |
| Give Blotat other 5s :)     | Wait                          | Waits additional 5 seconds before retrying X post | In Progress?                | Check Post Status                                  | ## 6. Post the Photo on X and monitor the status                                                                                 |
| In Progress?1               | If                            | Checks if Instagram post status is "in-progress" | Published to Instagram?     | Give Blotat more time / Send Instagram Error Message | ## 7. Post the Photo  on Instagram and monitor the status                                                                         |
| Give Blotat more time       | Wait                          | Waits additional 5 seconds before retrying Instagram post | In Progress?1               | Published to Instagram?                            | ## 7. Post the Photo  on Instagram and monitor the status                                                                         |
| In Progress?2               | If                            | Checks if Facebook post status is "in-progress"   | Published to Facebook ?     | Give Blotat other 5s :)1 / Send Facebook Error Message | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Give Blotat other 5s :)1    | Wait                          | Waits additional 5 seconds before retrying Facebook post | In Progress?2               | Published to Facebook ?                            | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Sticky Note                 | Sticky Note                    | Workflow overview and instructions               | -                          | -                                                  | ## 🤖 Social Media Photo Creation Bot with Approval Loop<br>Transform your Telegram into a **AI Photo Creator**... (full content) |
| Sticky Note1                | Sticky Note                    | Block 1 description                               | -                          | -                                                  | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper                       |
| Sticky Note2                | Sticky Note                    | Block 2 description                               | -                          | -                                                  | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory             |
| Sticky Note8                | Sticky Note                    | Block 3 description                               | -                          | -                                                  | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Sticky Note3                | Sticky Note                    | Block 4 description                               | -                          | -                                                  | ## 4. Photo Generation and upload to [Blotato](https://blotato.com/?ref=feras)<br>Generate your photo with Gemini-API, approve it via Telegram, save it to your google drive and upload it to Blotato |
| Sticky Note7                | Sticky Note                    | Block 5 description                               | -                          | -                                                  | ## 5. Post the Photo on Facebook and monitor the status                                                                          |
| Sticky Note9                | Sticky Note                    | Block 6 description                               | -                          | -                                                  | ## 6. Post the Photo on X and monitor the status                                                                                 |
| Sticky Note10               | Sticky Note                    | Block 7 description                               | -                          | -                                                  | ## 7. Post the Photo  on Instagram and monitor the status                                                                        |

---

### 4. Reproducing the Workflow from Scratch

**1. Create Telegram Trigger Node**  
- Type: Telegram Trigger  
- Configure to listen for "message" updates  
- Set webhook ID (auto-generated)  
- Connect your Telegram Bot API credentials

**2. Create "Voice or Text" Set Node**  
- Type: Set  
- Set field "text" to expression: `{{ $json?.message?.text || "" }}`  
- Connect output of Telegram Trigger to this node

**3. Create "A Voice?" If Node**  
- Type: If  
- Condition: Check if `{{ $json.message.text }}` is empty string  
- Connect output of "Voice or Text" node

**4. Create "Get Voice File" Telegram Node**  
- Type: Telegram  
- Operation: Get file by file ID  
- File ID expression: `{{ $('Listen for incoming events').item.json.message.voice.file_id }}`  
- Connect from "A Voice?" (Yes branch)

**5. Create "Speech to Text" OpenAI Node**  
- Type: OpenAI  
- Resource: Audio  
- Operation: Transcribe  
- Connect from "Get Voice File" node  
- Use OpenAI API credentials with Whisper enabled

**6. Connect "Speech to Text" and "A Voice?" (No branch) to "AI Agent" node**  
- Create LangChain Agent node:  
  - Text input: `{{ $json.text }}`  
  - System message: Provide instructions to generate video prompt and social media text, handle approval loop  
  - Use LangChain OpenAI Chat Model node with GPT-4.1-mini model credentials connected as AI language model  
  - Use Window Buffer Memory node keyed by Telegram user ID for conversational context

**7. Create "Approved from user?" If Node**  
- Check if AI Agent output contains approval keywords or starts with "{"  
- Connect from AI Agent node

**8. Create "Parse AI Output" Code Node**  
- JavaScript to extract and parse JSON from AI Agent output, validate required fields  
- Connect from "Approved from user?" (Yes branch)

**9. Create "Send questions or proposal to user" Telegram Node**  
- Sends AI Agent output text to Telegram user for feedback  
- Connect from "Approved from user?" (No branch)

**10. Create "Inform user about processing" Telegram Node**  
- Sends confirmation message to user about video/photo being prepared  
- Connect from "Parse AI Output" node

**11. Create "Save Prompt & Post-Text" Google Sheets Node**  
- Append row with columns "prompt" and "text" mapped from parsed AI output  
- Connect from "Inform user about processing" node  
- Use Google Sheets OAuth2 credentials and specify document and sheet

**12. Create "Generate an image" Google Gemini Node**  
- Prompt: videoPrompt from parsed AI output  
- Model: Gemini 2.5 Flash Image  
- Use Google Palm API credentials  
- Connect from "Save Prompt & Post-Text" node

**13. Create "Send a photo message" Telegram Node**  
- Send generated image binary to user chat ID  
- Connect from "Generate an image" node

**14. Create "Send message and wait for response" Telegram Node**  
- Send message "Good?" and wait for user approval  
- Use double approval option  
- Connect from "Send a photo message" node

**15. Create "If1" Node**  
- Checks if user response contains `approved=true`  
- Connect from "Send message and wait for response" node

**16. Create "Upload image1" Google Drive Node**  
- Upload binary image to specified Google Drive folder  
- Connect from "If1" (Yes branch)  
- Use Google Drive OAuth2 credentials

**17. Create "Download image from Drive" Google Drive Node**  
- Download uploaded image binary for further processing  
- Connect from "Upload image1" node

**18. Create "Upload media1" Blotato Node**  
- Upload image binary to Blotato media library  
- Connect from "Download image from Drive" node  
- Use Blotato API credentials

**19. Create "Create instagram Post" Blotato Node**  
- Create Instagram post with text and media URL  
- Connect from "Upload media1" node  
- Use Instagram account ID and Blotato credentials

**20. Create "Create x post" Blotato Node**  
- Create X (Twitter) post with text and media URL  
- Connect from "Upload media1" node  
- Use X account ID and Blotato credentials

**21. Create "Create FB post" Blotato Node**  
- Create Facebook post with text and media URL for Facebook page  
- Connect from "Upload media1" node  
- Use Facebook account and page IDs, Blotato credentials

**22. Create "Wait" Nodes**  
- Add wait nodes (10s, 30s, 5s as per original workflow) connected appropriately after post creation nodes

**23. Create "Check Post Status" Blotato Nodes**  
- Query post status using postSubmissionId  
- Connect from wait nodes

**24. Create "Published to ..." If Nodes**  
- Check if post status is "published" for each platform  
- Connect from post status check nodes

**25. Create "Confirm publishing to ..." Telegram Nodes**  
- Notify user with post URL when published  
- Connect from "Published to ..." If Nodes (Yes branch)

**26. Create "Send ... Error Message" Telegram Nodes**  
- Notify user on posting error  
- Connect from "Published to ..." If Nodes (No branch leading to error message)

**27. Create "In Progress?" If Nodes and Wait Loops**  
- Detect "in-progress" status to retry checking after waiting  
- Connect wait nodes and status checks accordingly

**28. Add Sticky Notes**  
- Add descriptive sticky notes as per original for documentation purposes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Workflow transforms Telegram messages into AI-generated photos with social media posts, supporting approval loops and automated posting on Facebook, Instagram, and X. Requires configured bots and API credentials for OpenAI, Google Gemini, Google Drive/Sheets, and Blotato. | Workflow purpose and setup instructions |
| Blotato platform is used for media upload and social posting API: https://blotato.com/?ref=feras                                                                                                                                               | External platform link                   |
| Google Gemini API used for image generation - requires proper Google Cloud credentials                                                                                                                                                         | API dependency                          |
| OpenAI Whisper used for speech-to-text transcription of voice messages                                                                                                                                                                         | Technology detail                      |
| Telegram Bot setup requires API token from @BotFather                                                                                                                                                                                          | Telegram bot setup                     |

---

**Disclaimer:** The text analyzed and documented originates exclusively from an automated workflow created with n8n, respecting all applicable content policies and legal constraints. All data processed are legal and publicly accessible.