Create & Share AI Videos with Telegram, Gemini & Post to TikTok, Instagram, Facebook

https://n8nworkflows.xyz/workflows/create---share-ai-videos-with-telegram--gemini---post-to-tiktok--instagram--facebook-10850


# Create & Share AI Videos with Telegram, Gemini & Post to TikTok, Instagram, Facebook

### 1. Workflow Overview

This workflow automates the creation and sharing of AI-generated videos and social media posts via Telegram, Google Gemini AI, Google Drive, and the Blotato platform, targeting TikTok, Instagram, and Facebook. It is designed to accept user input as text or voice through Telegram, generate video prompts and social media copy using AI, create videos with Google Gemini, upload media assets, post to social platforms via Blotato, and monitor publishing status. The workflow includes iterative user feedback and approval loops to refine the content before publishing.

**Logical Blocks:**

- **1.1 Input Reception:** Listening to Telegram messages (text or voice), converting voice notes to text.
- **1.2 AI Processing:** Using OpenAI and LangChain agents to generate video prompts and social media text with iterative approval.
- **1.3 Content Approval & Data Recording:** Parsing AI output, detecting user approval, saving prompts and posts to Google Sheets, and notifying users.
- **1.4 Video Generation & Upload:** Creating videos with Google Gemini API, uploading to Google Drive and Blotato.
- **1.5 Posting & Monitoring on TikTok:** Posting videos on TikTok via Blotato and monitoring the post status.
- **1.6 Posting & Monitoring on Instagram:** Posting videos on Instagram via Blotato and monitoring the post status.
- **1.7 Posting & Monitoring on Facebook:** Posting videos on Facebook via Blotato and monitoring the post status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives incoming Telegram messages, determines if the input is text or voice, and extracts the message text accordingly. Voice notes are downloaded and transcribed to text.

- **Nodes Involved:**  
  - Listen for incoming events  
  - Voice or Text  
  - A Voice?  
  - Get Voice File  
  - Speech to Text

- **Node Details:**  

1. **Listen for incoming events**  
   - Type: Telegram Trigger  
   - Role: Entry point listening for Telegram messages.  
   - Config: Triggers on new messages.  
   - Output: Telegram message JSON with text or voice data.  
   - Edge cases: Telegram API downtime, webhook misconfiguration.

2. **Voice or Text**  
   - Type: Set  
   - Role: Extracts text from Telegram message or sets empty string if none.  
   - Config: Sets `text` field to the message text or empty string.  
   - Input: From Telegram trigger.  
   - Output: JSON with `text` property.  
   - Edge cases: Empty or malformed messages.

3. **A Voice?**  
   - Type: If  
   - Role: Checks if the message text is empty (implying a voice note).  
   - Conditions: If `message.text` is empty string → true branch (voice), else false (text).  
   - Input: From Set node.  
   - Output: Branches to voice or text processing.  
   - Edge cases: Messages with empty text but no voice file.

4. **Get Voice File**  
   - Type: Telegram (API)  
   - Role: Retrieve the voice file from Telegram using file ID.  
   - Config: Uses `message.voice.file_id` from Telegram event.  
   - Input: From "A Voice?" node (voice branch).  
   - Edge cases: File not found, Telegram API errors.

5. **Speech to Text**  
   - Type: OpenAI (Audio Transcription)  
   - Role: Transcribe voice note to text using OpenAI Whisper.  
   - Input: Binary audio from "Get Voice File".  
   - Output: Transcribed text.  
   - Edge cases: Audio format unsupported, transcription failures, API limits.

---

#### 1.2 AI Processing

- **Overview:**  
Processes the input text with a LangChain AI Agent using OpenAI GPT-4.1-mini, managing conversational memory for iterative refinement and generating video prompts and social media text.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Window Buffer Memory  
  - Approved from user?  
  - Send questions or proposal to user

- **Node Details:**

1. **AI Agent**  
   - Type: LangChain Agent  
   - Role: Processes user text to generate video prompt and social media text; supports iterative feedback and final approval.  
   - Config: System message instructs the agent to create video prompts and social media text, handle refinement based on user feedback, and output final JSON only on approval.  
   - Input: Text from transcription or direct text input.  
   - Output: AI-generated text or JSON after approval.  
   - Edge cases: AI generation errors, incomplete or ambiguous output.

2. **OpenAI Chat Model**  
   - Type: LangChain OpenAI Chat Model  
   - Role: Provides the underlying GPT-4.1-mini model for the AI Agent.  
   - Config: Uses GPT-4.1-mini, no extra options.  
   - Input: Text prompts from AI Agent.  
   - Edge cases: API rate limits, response delays.

3. **Window Buffer Memory**  
   - Type: LangChain Memory Buffer Window  
   - Role: Maintains conversational context over last 10 entries per Telegram user (session key by user ID).  
   - Input: Connected to AI Agent for context.  
   - Edge cases: Memory overflow, session ID mismatches.

4. **Approved from user?**  
   - Type: If  
   - Role: Detects if user input contains approval keywords ("approved", "ok") to proceed with finalizing content.  
   - Conditions: Checks if AI Agent output contains `videoPrompt` or `socialMediaText` or starts with `{`.  
   - Output: True branch proceeds to parse AI output; false branch sends proposal text back to user for refinement.  
   - Edge cases: Missed approval keywords, false positives.

5. **Send questions or proposal to user**  
   - Type: Telegram  
   - Role: Sends AI-generated suggestions or questions back to the Telegram user for feedback.  
   - Input: AI Agent output when approval not detected.  
   - Edge cases: Telegram API failures.

---

#### 1.3 Content Approval & Data Recording

- **Overview:**  
Parses AI output JSON for video prompt and social text, saves content to Google Sheets, and informs the user that video creation is underway.

- **Nodes Involved:**  
  - Parse AI Output  
  - Save Prompt & Post-Text  
  - Inform user about processing

- **Node Details:**

1. **Parse AI Output**  
   - Type: Code  
   - Role: Extracts JSON fields `videoPrompt` and `socialMediaText` from AI Agent output text, handling markdown or format inconsistencies.  
   - Config: Custom JS code parsing JSON or extracting fields by regex fallback.  
   - Edge cases: Parsing errors, missing fields, malformed AI output.

2. **Save Prompt & Post-Text**  
   - Type: Google Sheets Append  
   - Role: Appends parsed video prompt and social media text as new rows into a specified Google Sheet.  
   - Config: Uses spreadsheet ID and sheet name, maps columns `prompt` and `text`.  
   - Edge cases: Sheets API errors, permission issues.

3. **Inform user about processing**  
   - Type: Telegram  
   - Role: Notifies user that video generation has started.  
   - Input: Triggered after parsing and saving data.  
   - Edge cases: Telegram API downtime.

---

#### 1.4 Video Generation & Upload

- **Overview:**  
Uses Google Gemini API to generate a video from the approved video prompt with a vertical aspect ratio, uploads the video to Google Drive, downloads it, and uploads it to Blotato for social media posting.

- **Nodes Involved:**  
  - Generate a video  
  - Upload video  
  - Download Video from Drive1  
  - Upload media1

- **Node Details:**

1. **Generate a video**  
   - Type: LangChain Google Gemini  
   - Role: Generate video using Gemini API with prompt from parsed AI output.  
   - Config: Model "veo-3.0-fast-generate-001", resource type "video", aspect ratio 9:16.  
   - Credentials: Google Palm API.  
   - Output: Binary video file.  
   - Edge cases: API errors, prompt invalidity, generation timeouts.

2. **Upload video**  
   - Type: Google Drive Upload  
   - Role: Uploads generated binary video to specified Google Drive folder.  
   - Config: Folder ID set to a dedicated video folder.  
   - Credentials: Google Drive OAuth2.  
   - Edge cases: Upload failures, quota exceeded.

3. **Download Video from Drive1**  
   - Type: Google Drive Download  
   - Role: Downloads uploaded video file using file ID for Blotato upload.  
   - Edge cases: File missing, permission errors.

4. **Upload media1**  
   - Type: Blotato Upload Media  
   - Role: Uploads video binary to Blotato media library for posting.  
   - Credentials: Blotato API.  
   - Edge cases: API errors, large file rejection.

---

#### 1.5 Posting & Monitoring on TikTok

- **Overview:**  
Creates a TikTok post on Blotato with the uploaded media and social media text, waits, and periodically checks the post status until published or failed, notifying the user accordingly.

- **Nodes Involved:**  
  - Create TikTok post  
  - Wait  
  - Check Post Status  
  - Published to TikTok?  
  - In Progress?  
  - Give Blotat other 5s :)  
  - Confirm publishing to Tiktok  
  - Send TikTok Error Message

- **Node Details:**

1. **Create TikTok post**  
   - Type: Blotato Create Post  
   - Role: Posts video and text to TikTok account via Blotato.  
   - Config: Platform "tiktok", includes AI-generated flag.  
   - Edge cases: Posting errors.

2. **Wait**  
   - Type: Wait  
   - Role: Pauses for 30 seconds before status check.  
   - Edge cases: Workflow timeouts.

3. **Check Post Status**  
   - Type: Blotato Get Post Status  
   - Role: Retrieves status of TikTok post by submission ID.  
   - Edge cases: API errors, invalid IDs.

4. **Published to TikTok?**  
   - Type: If  
   - Role: Checks if status is "published".  
   - Branches: Yes → Confirm publishing; No → In Progress check.

5. **In Progress?**  
   - Type: If  
   - Role: Checks if status is still "in-progress".  
   - Branches: Yes → Wait 5 seconds more; No → Send error message.

6. **Give Blotat other 5s :)**  
   - Type: Wait  
   - Role: Waits 5 seconds before re-check.  
   - Edge cases: Repeated delays.

7. **Confirm publishing to Tiktok**  
   - Type: Telegram  
   - Role: Notifies user with link to published TikTok video.  
   - Edge cases: Telegram API issues.

8. **Send TikTok Error Message**  
   - Type: Telegram  
   - Role: Notifies user of posting error.  
   - Edge cases: Telegram API issues.

---

#### 1.6 Posting & Monitoring on Instagram

- **Overview:**  
Creates an Instagram post on Blotato, waits, checks post status, and notifies user upon success or failure.

- **Nodes Involved:**  
  - Create post1 (Instagram)  
  - Wait1  
  - Get post1  
  - Published to Instagram?  
  - In Progress?1  
  - Give Blotat more time  
  - Confirm publishing to Instagram  
  - Send Instagram Error Message

- **Node Details:**

1. **Create post1**  
   - Type: Blotato Create Post  
   - Role: Posts video and text to Instagram account.  
   - Edge cases: Posting failures.

2. **Wait1**  
   - Type: Wait  
   - Role: Waits 30 seconds before status check.  
   - Edge cases: Workflow timeouts.

3. **Get post1**  
   - Type: Blotato Get Post Status  
   - Role: Retrieves Instagram post status.  
   - Edge cases: API errors.

4. **Published to Instagram?**  
   - Type: If  
   - Role: Checks if status is "published" (note: possible typo `"=published"` in condition).  
   - Edge cases: Condition mismatch causing false negatives.

5. **In Progress?1**  
   - Type: If  
   - Role: Checks if status is "in-progress".  
   - Branches: Wait more or send error.

6. **Give Blotat more time**  
   - Type: Wait  
   - Role: Waits 5 seconds before retry.  
   - Edge cases: Repeated delays.

7. **Confirm publishing to Instagram**  
   - Type: Telegram  
   - Role: Sends confirmation message with post URL.  
   - Edge cases: Telegram API failures.

8. **Send Instagram Error Message**  
   - Type: Telegram  
   - Role: Sends error message if post fails.  
   - Edge cases: Telegram API issues.

---

#### 1.7 Posting & Monitoring on Facebook

- **Overview:**  
Posts video and text to Facebook via Blotato, waits, monitors post status, and notifies user upon success or failure.

- **Nodes Involved:**  
  - Create Facebook post  
  - Wait2  
  - Get Facebook post  
  - Published to Facebook?  
  - In Progress?2  
  - Give Blotat other 5s :)2  
  - Confirm publishing to Facebook  
  - Send Facebook Error Message

- **Node Details:**

1. **Create Facebook post**  
   - Type: Blotato Create Post  
   - Role: Posts video and text to Facebook page.  
   - Edge cases: Posting errors.

2. **Wait2**  
   - Type: Wait  
   - Role: Waits 30 seconds before status check.  
   - Edge cases: Workflow timeouts.

3. **Get Facebook post**  
   - Type: Blotato Get Post Status  
   - Role: Retrieves Facebook post status.  
   - Edge cases: API errors.

4. **Published to Facebook?**  
   - Type: If  
   - Role: Checks if status is "published".  
   - Edge cases: False negatives due to string matching.

5. **In Progress?2**  
   - Type: If  
   - Role: Checks if status is "in-progress".  
   - Branches: Wait more or send error.

6. **Give Blotat other 5s :)2**  
   - Type: Wait  
   - Role: Waits 5 seconds before retry.  
   - Edge cases: Repeated delays.

7. **Confirm publishing to Facebook**  
   - Type: Telegram  
   - Role: Sends confirmation message with post URL.  
   - Edge cases: Telegram API issues.

8. **Send Facebook Error Message**  
   - Type: Telegram  
   - Role: Sends error message if post fails.  
   - Edge cases: Telegram API issues.

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                                  | Input Node(s)                         | Output Node(s)                              | Sticky Note                                                                                       |
|------------------------------|----------------------------------------|-------------------------------------------------|-------------------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------|
| Listen for incoming events    | Telegram Trigger                       | Receive Telegram messages                        | -                                   | Voice or Text                              | ## 1. Capture Input: Receives Telegram messages and converts voice notes to text using OpenAI Whisper |
| Voice or Text                | Set                                    | Extract text or prepare empty string             | Listen for incoming events           | A Voice?                                    | ## 1. Capture Input: Receives Telegram messages and converts voice notes to text using OpenAI Whisper |
| A Voice?                     | If                                     | Determine if input is voice or text               | Voice or Text                       | Get Voice File / AI Agent                   | ## 1. Capture Input: Receives Telegram messages and converts voice notes to text using OpenAI Whisper |
| Get Voice File               | Telegram API                           | Download voice file from Telegram                 | A Voice?                           | Speech to Text                              | ## 1. Capture Input: Receives Telegram messages and converts voice notes to text using OpenAI Whisper |
| Speech to Text               | OpenAI Audio Transcription             | Transcribe voice note to text                      | Get Voice File                     | AI Agent                                    | ## 1. Capture Input: Receives Telegram messages and converts voice notes to text using OpenAI Whisper |
| AI Agent                    | LangChain Agent                        | Generate video prompt and social media text      | Speech to Text / A Voice? (text branch) | Approved from user?                         | ## 2. AI Content Creation: Generates LinkedIn posts and handles revision requests through conversational memory |
| OpenAI Chat Model            | LangChain OpenAI Model                 | GPT-4.1-mini model for AI Agent                   | AI Agent                           | AI Agent                                    | ## 2. AI Content Creation: Generates LinkedIn posts and handles revision requests through conversational memory |
| Window Buffer Memory         | LangChain Memory Buffer                | Maintain conversational context                    | AI Agent                           | AI Agent                                    | ## 2. AI Content Creation: Generates LinkedIn posts and handles revision requests through conversational memory |
| Approved from user?          | If                                     | Detect user approval keywords                      | AI Agent                           | Parse AI Output / Send questions or proposal to user | ## 3. Approval & Publishing: Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Send questions or proposal to user | Telegram                          | Send AI suggestions back for user feedback        | Approved from user? (false branch) | -                                           | ## 3. Approval & Publishing: Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Parse AI Output              | Code                                   | Parse AI output JSON to extract video prompt and social text | Approved from user? (true branch) | Inform user about processing                | ## 3. Approval & Publishing: Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Save Prompt & Post-Text     | Google Sheets Append                   | Save video prompt and social media text           | Inform user about processing       | Generate a video                            | ## 3. Approval & Publishing: Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Inform user about processing | Telegram                              | Notify user video generation has started           | Parse AI Output                    | Save Prompt & Post-Text                     | ## 3. Approval & Publishing: Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Generate a video             | LangChain Google Gemini API            | Generate video from prompt                         | Save Prompt & Post-Text            | Upload video                               | ## 4. Video Generation and upload to Blotato: Generate your video mit Gemini-API, save it to your google drive and uploadd it to Blotato |
| Upload video                | Google Drive Upload                   | Upload generated video to Google Drive              | Generate a video                   | Download Video from Drive1                   | ## 4. Video Generation and upload to Blotato: Generate your video mit Gemini-API, save it to your google drive and uploadd it to Blotato |
| Download Video from Drive1  | Google Drive Download                 | Download video for upload to Blotato                 | Upload video                      | Upload media1                               | ## 4. Video Generation and upload to Blotato: Generate your video mit Gemini-API, save it to your google drive and uploadd it to Blotato |
| Upload media1               | Blotato Upload Media                 | Upload video media to Blotato                         | Download Video from Drive1         | Create post1, Create TikTok post, Create Facebook post | ## 4. Video Generation and upload to Blotato: Generate your video mit Gemini-API, save it to your google drive and uploadd it to Blotato |
| Create TikTok post          | Blotato Create Post                  | Create TikTok post with video and social text        | Upload media1                    | Wait                                        | ## 5. Post the video on TikTok and monitor the status |
| Wait                       | Wait                                  | Pause before checking TikTok post status              | Create TikTok post               | Check Post Status                           | ## 5. Post the video on TikTok and monitor the status |
| Check Post Status           | Blotato Get Post Status              | Check status of TikTok post                            | Wait                            | Published to TikTok?                        | ## 5. Post the video on TikTok and monitor the status |
| Published to TikTok?        | If                                    | Check if TikTok post is published                      | Check Post Status                | Confirm publishing to Tiktok / In Progress? | ## 5. Post the video on TikTok and monitor the status |
| In Progress?                | If                                    | Check if TikTok post is still in progress              | Published to TikTok?             | Give Blotat other 5s :) / Send TikTok Error Message | ## 5. Post the video on TikTok and monitor the status |
| Give Blotat other 5s :)     | Wait                                  | Wait 5 seconds before retrying TikTok post check       | In Progress?                    | Published to TikTok?                        | ## 5. Post the video on TikTok and monitor the status |
| Confirm publishing to Tiktok | Telegram                             | Notify user TikTok video is published                  | Published to TikTok?             | -                                           | ## 5. Post the video on TikTok and monitor the status |
| Send TikTok Error Message   | Telegram                             | Notify user of TikTok upload error                      | In Progress?                    | -                                           | ## 5. Post the video on TikTok and monitor the status |
| Create post1               | Blotato Create Post                  | Create Instagram post with video and social text       | Upload media1                   | Wait1                                       | ## 6. Post the video on Instagram and monitor the status |
| Wait1                      | Wait                                  | Pause before checking Instagram post status             | Create post1                   | Get post1                                    | ## 6. Post the video on Instagram and monitor the status |
| Get post1                  | Blotato Get Post Status              | Check Instagram post status                              | Wait1                          | Published to Instagram?                      | ## 6. Post the video on Instagram and monitor the status |
| Published to Instagram?     | If                                    | Check if Instagram post is published                     | Get post1                      | Confirm publishing to Instagram / In Progress?1 | ## 6. Post the video on Instagram and monitor the status |
| In Progress?1              | If                                    | Check if Instagram post is still in progress             | Published to Instagram?         | Give Blotat more time / Send Instagram Error Message | ## 6. Post the video on Instagram and monitor the status |
| Give Blotat more time       | Wait                                  | Wait 5 seconds before retrying Instagram post check      | In Progress?1                  | Published to Instagram?                      | ## 6. Post the video on Instagram and monitor the status |
| Confirm publishing to Instagram | Telegram                         | Notify user Instagram video is published                 | Published to Instagram?         | -                                           | ## 6. Post the video on Instagram and monitor the status |
| Send Instagram Error Message | Telegram                           | Notify user of Instagram upload error                     | In Progress?1                  | -                                           | ## 6. Post the video on Instagram and monitor the status |
| Create Facebook post        | Blotato Create Post                  | Create Facebook post with video and social text          | Upload media1                   | Wait2                                       | ## 7. Post the video on Facebook and monitor the status |
| Wait2                      | Wait                                  | Pause before checking Facebook post status                | Create Facebook post           | Get Facebook post                            | ## 7. Post the video on Facebook and monitor the status |
| Get Facebook post          | Blotato Get Post Status              | Check Facebook post status                                | Wait2                          | Published to Facebook?                        | ## 7. Post the video on Facebook and monitor the status |
| Published to Facebook?      | If                                    | Check if Facebook post is published                       | Get Facebook post              | Confirm publishing to Facebook / In Progress?2 | ## 7. Post the video on Facebook and monitor the status |
| In Progress?2              | If                                    | Check if Facebook post is still in progress               | Published to Facebook?         | Give Blotat other 5s :)2 / Send Facebook Error Message | ## 7. Post the video on Facebook and monitor the status |
| Give Blotat other 5s :)2    | Wait                                  | Wait 5 seconds before retrying Facebook post check        | In Progress?2                  | Published to Facebook?                        | ## 7. Post the video on Facebook and monitor the status |
| Confirm publishing to Facebook | Telegram                         | Notify user Facebook video is published                   | Published to Facebook?         | -                                           | ## 7. Post the video on Facebook and monitor the status |
| Send Facebook Error Message | Telegram                             | Notify user of Facebook upload error                       | In Progress?2                  | -                                           | ## 7. Post the video on Facebook and monitor the status |
| Send questions or proposal to user | Telegram                      | Send AI message back to user for feedback                 | Approved from user? (false)     | -                                           | ## 3. Approval & Publishing: Detects approval keywords, extracts final content, save the output in your google sheet and inform the user |
| Sticky Note                 | Sticky Note                          | Overview and usage notes                                | -                               | -                                           | Covers entire workflow overview and setup instructions.                                         |
| Sticky Note1                | Sticky Note                          | Input capture block summary                             | -                               | -                                           | ## 1. Capture Input                                                                                 |
| Sticky Note2                | Sticky Note                          | AI content creation summary                            | -                               | -                                           | ## 2. AI Content Creation                                                                          |
| Sticky Note8                | Sticky Note                          | Approval & Publishing summary                          | -                               | -                                           | ## 3. Approval & Publishing                                                                        |
| Sticky Note3                | Sticky Note                          | Video generation and upload summary                    | -                               | -                                           | ## 4. Video Generation and upload to Blotato                                                     |
| Sticky Note7                | Sticky Note                          | TikTok posting summary                                 | -                               | -                                           | ## 5. Post the video on TikTok and monitor the status                                            |
| Sticky Note9                | Sticky Note                          | Instagram posting summary                              | -                               | -                                           | ## 6. Post the video on Instagram and monitor the status                                         |
| Sticky Note10               | Sticky Note                          | Facebook posting summary                               | -                               | -                                           | ## 7. Post the video on Facebook and monitor the status                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Settings: Listen for "message" updates.  
   - Credential: Connect your Telegram Bot API.  
   - Position: Entry node.

2. **Create Set node "Voice or Text"**  
   - Extract `text` from incoming Telegram message JSON or set to empty string if none.

3. **Create If node "A Voice?"**  
   - Condition: Check if extracted text is empty string.  
   - True branch → voice processing; False branch → text processing.

4. **Voice branch:**  
   a. Create Telegram node "Get Voice File"  
      - Use `message.voice.file_id` to download voice file.  
      - Credential: Telegram Bot.  
   b. Create OpenAI node "Speech to Text"  
      - Operation: Audio transcribe using OpenAI Whisper.  
      - Credential: OpenAI API.

5. **Text branch:**  
   - Bypass to AI Agent.

6. **Create LangChain Window Buffer Memory node**  
   - Configure session key with `message.from.id`.  
   - Context window length: 10.

7. **Create LangChain OpenAI Chat Model node**  
   - Model: GPT-4.1-mini.

8. **Create LangChain AI Agent node**  
   - Input: Text from transcription or direct text input.  
   - System prompt instructs to create video prompt and social media post, handle iterative feedback, output JSON only on approval.  
   - Connect to OpenAI Chat Model and Window Buffer Memory nodes.

9. **Create If node "Approved from user?"**  
   - Condition: Check if AI output contains `videoPrompt` or `socialMediaText` or starts with `{`.  
   - True branch → parse AI output.  
   - False branch → send AI output back to user for feedback.

10. **Create Telegram node "Send questions or proposal to user"**  
    - Sends AI Agent message to user chat ID.

11. **Create Code node "Parse AI Output"**  
    - Parses JSON from AI Agent output text extracting `videoPrompt` and `socialMediaText`.  
    - Throws error if parsing fails or fields missing.

12. **Create Telegram node "Inform user about processing"**  
    - Notify user video generation has started.

13. **Create Google Sheets node "Save Prompt & Post-Text"**  
    - Append operation on your Google Sheet with columns: `prompt` and `text`.  
    - Credential: Google Sheets OAuth2.

14. **Create LangChain Google Gemini node "Generate a video"**  
    - Model: veo-3.0-fast-generate-001.  
    - Resource: video.  
    - Aspect ratio: 9:16.  
    - Credential: Google Palm API.  
    - Input prompt from parsed AI output.

15. **Create Google Drive Upload node "Upload video"**  
    - Upload generated video binary to dedicated folder.  
    - Credential: Google Drive OAuth2.

16. **Create Google Drive Download node "Download Video from Drive1"**  
    - Download video by file ID for Blotato upload.

17. **Create Blotato Upload Media node "Upload media1"**  
    - Upload downloaded video binary.  
    - Credential: Blotato API.

18. **Create Blotato Create Post nodes:**  
    - For TikTok ("Create TikTok post"), Instagram ("Create post1"), Facebook ("Create Facebook post").  
    - Use social media text from parsed AI output and media URL from upload response.  
    - Credentials: Blotato API.  
    - Platform and account IDs must be set accordingly.

19. **Create Wait nodes:**  
    - Wait 30 seconds after each post creation before status check (nodes: Wait, Wait1, Wait2).

20. **Create Blotato Get Post Status nodes:**  
    - Check status of TikTok ("Check Post Status"), Instagram ("Get post1"), Facebook ("Get Facebook post") posts.

21. **Create If nodes:**  
    - "Published to TikTok?", "In Progress?", "Published to Instagram?", "In Progress?1", "Published to Facebook?", "In Progress?2"  
    - Conditions check if post status is "published" or "in-progress".

22. **Create Wait nodes for polling delays:**  
    - "Give Blotat other 5s :)", "Give Blotat more time", "Give Blotat other 5s :)2"  
    - Wait for 5 seconds before rechecking status.

23. **Create Telegram notification nodes:**  
    - "Confirm publishing to Tiktok", "Confirm publishing to Instagram", "Confirm publishing to Facebook"  
    - Notify user with post URLs on successful publishing.

24. **Create Telegram error message nodes:**  
    - "Send TikTok Error Message", "Send Instagram Error Message", "Send Facebook Error Message"  
    - Notify user in case of upload or publishing failure.

25. **Connect all nodes as per dependencies described in sections 1.1 to 1.7.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow automates video generation and posting using Telegram input, Google Gemini for AI video creation, OpenAI for transcription and content generation, and Blotato for social media posting. | Workflow overview                                             |
| Blotato platform used for posting on TikTok, Instagram, Facebook.                                                                                                   | https://blotato.com/?ref=feras                                |
| Requires Telegram Bot API token, OpenAI API key (for GPT and Whisper), Google Palm API credentials, Google Drive OAuth2, Google Sheets OAuth2, and Blotato API keys. | Credential setup instructions                                 |
| Approval loop enables iterative content refinement via Telegram chat.                                                                                               | Block 1.2 and 1.3 descriptions                               |
| Video generation uses Gemini API model "veo-3.0-fast-generate-001" with vertical 9:16 aspect ratio.                                                                  | Video generation block                                        |
| Potential edge case: Instagram post status check condition uses `"=published"` which might be a typo; verify string matching for status conditions.                  | Blocks 1.6, potential bug                                     |
| Telegram notifications provide user feedback for each stage: proposal, processing, publication success, and errors.                                                 | User communication throughout workflow                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. The processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.