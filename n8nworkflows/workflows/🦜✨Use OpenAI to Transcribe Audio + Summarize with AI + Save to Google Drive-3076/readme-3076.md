🦜✨Use OpenAI to Transcribe Audio + Summarize with AI + Save to Google Drive

https://n8nworkflows.xyz/workflows/---use-openai-to-transcribe-audio---summarize-with-ai---save-to-google-drive-3076


# 🦜✨Use OpenAI to Transcribe Audio + Summarize with AI + Save to Google Drive

### 1. Workflow Overview

This workflow automates the transcription of audio files stored in Google Drive, generates AI-powered structured summaries and markdown reports, and saves all outputs back to Google Drive. It optionally includes a human approval step before transcription begins, ensuring oversight. Notifications are sent upon completion via email and Telegram.

**Target Use Cases:**  
- Content teams processing voice memos or interviews  
- Researchers needing searchable transcripts and summaries  
- Administrators managing meeting recordings  

**Logical Blocks:**  
- **1.1 Input Reception & Approval (Optional Human-in-the-Loop)**: Trigger on new audio files in Google Drive and optionally request user approval via Gmail before proceeding.  
- **1.2 Audio File Retrieval & Filtering**: Search Google Drive folder, filter for .m4a audio files, and download the latest file.  
- **1.3 Audio Transcription**: Use OpenAI Whisper to transcribe audio into text.  
- **1.4 AI-Powered Transcript Summarization**: Generate two AI summaries—one structured JSON report and one detailed JSON-based technical summary—then convert to markdown.  
- **1.5 File Naming & Storage**: Generate timestamped filenames and save raw transcripts, JSON reports, and markdown files to Google Drive.  
- **1.6 Notification Dispatch**: Send completion alerts with document links via Gmail and Telegram.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Approval (Optional Human-in-the-Loop)

- **Overview:**  
  This block triggers the workflow when a new audio file is created in a specific Google Drive folder. It optionally sends an approval request email to a user, pausing the workflow until approval or timeout.

- **Nodes Involved:**  
  - On File Created Trigger (disabled)  
  - Gmail User for Approval (disabled)  
  - Search Google Drive  

- **Node Details:**  
  - **On File Created Trigger**  
    - Type: Google Drive Trigger  
    - Role: Watches a specific Google Drive folder for new audio files (MIME type: audio).  
    - Config: Folder ID set to "Audio Recordings" folder; polls every minute.  
    - Disabled: Yes (optional activation).  
    - Edge Cases: Missed triggers if polling interval too long; Google Drive API quota limits.  
  - **Gmail User for Approval**  
    - Type: Gmail node with sendAndWait operation  
    - Role: Sends an approval email to a configured user and waits up to 45 minutes for response.  
    - Config: Email address from environment variable; double approval type; customizable timeout.  
    - Disabled: Yes (optional activation).  
    - Edge Cases: User non-response leads to timeout; email delivery failures; OAuth token expiration.  
  - **Search Google Drive**  
    - Type: Google Drive node (fileFolder resource)  
    - Role: Lists files in the target folder to find audio files for processing.  
    - Config: Folder ID set to "Audio Recordings".  
    - Edge Cases: Folder permission issues; API rate limits.  

---

#### 2.2 Audio File Retrieval & Filtering

- **Overview:**  
  Searches the target Google Drive folder, filters files to only .m4a extension, limits to the most recent file, and downloads it for transcription.

- **Nodes Involved:**  
  - Filter by .m4a extension  
  - Limit to last file  
  - Download audio file  

- **Node Details:**  
  - **Filter by .m4a extension**  
    - Type: Filter node  
    - Role: Filters incoming files to only those whose names end with ".m4a" (case-insensitive).  
    - Config: Condition on filename endsWith ".m4a".  
    - Edge Cases: Files with incorrect extensions; case sensitivity handled.  
  - **Limit to last file**  
    - Type: Limit node  
    - Role: Keeps only the last (most recent) file from filtered results.  
    - Config: Keep last item only.  
    - Edge Cases: Empty input list; multiple files with same timestamp.  
  - **Download audio file**  
    - Type: Google Drive node  
    - Role: Downloads the selected audio file content for transcription.  
    - Config: Uses file ID from previous node.  
    - Edge Cases: File access revoked; download failures; large file size/timeouts.  

---

#### 2.3 Audio Transcription

- **Overview:**  
  Transcribes the downloaded audio file into text using OpenAI's Whisper model.

- **Nodes Involved:**  
  - Transcribe with OpenAI  
  - Set Config  

- **Node Details:**  
  - **Transcribe with OpenAI**  
    - Type: OpenAI node (LangChain integration)  
    - Role: Performs audio transcription using OpenAI's Whisper.  
    - Config: Resource set to "audio", operation "transcribe".  
    - Credentials: OpenAI API key required.  
    - Edge Cases: API rate limits; transcription errors; unsupported audio formats.  
  - **Set Config**  
    - Type: Set node  
    - Role: Assigns transcription text and current datetime to variables for downstream use.  
    - Config: Sets `text` from transcription output and `datetime` to current timestamp.  
    - Edge Cases: Missing transcription text; datetime format issues.  

---

#### 2.4 AI-Powered Transcript Summarization

- **Overview:**  
  Generates two AI summaries from the transcript: a structured JSON report and a detailed JSON-based technical summary. Converts the latter into markdown format for readability.

- **Nodes Involved:**  
  - Summarize to JSON  
  - Summarize to Structured JSON  
  - Convert JSON to Markdown  

- **Node Details:**  
  - **Summarize to JSON**  
    - Type: OpenAI node (LangChain)  
    - Role: Creates a detailed structured transcript summary with sections like executive summary, analysis, and observations.  
    - Config: Uses GPT-4o-mini model; system prompt defines role, task, rules, and formatting.  
    - Output: JSON format.  
    - Edge Cases: Model hallucination; JSON parsing errors; prompt misinterpretation.  
  - **Summarize to Structured JSON**  
    - Type: OpenAI node (LangChain)  
    - Role: Produces a concise JSON report with key points, action items, sentiment, and glossary.  
    - Config: GPT-4o-mini; strict JSON output required.  
    - Edge Cases: Same as above.  
  - **Convert JSON to Markdown**  
    - Type: OpenAI node (LangChain)  
    - Role: Converts the detailed JSON summary into markdown text for human-friendly reading.  
    - Config: GPT-4o-mini; system prompt instructs to output markdown only.  
    - Edge Cases: Markdown formatting errors; incomplete conversion.  

---

#### 2.5 File Naming & Storage

- **Overview:**  
  Generates timestamped filenames for JSON and markdown reports, saves these files and the raw transcript text to Google Drive in the designated folder.

- **Nodes Involved:**  
  - Get Filename for JSON  
  - Get Filename for Markdown  
  - Save JSON file to Google Drive  
  - Save Markdown file to Google Drive  
  - Save Raw Transcript to Google Drive  
  - Get JSON File Meta  
  - Get Markdown File Meta  
  - Prepare Response JSON  
  - Prepare Response Markdown  

- **Node Details:**  
  - **Get Filename for JSON / Markdown**  
    - Type: Set nodes  
    - Role: Compose filenames combining file ID, name, and timestamp with appropriate extensions (.json, .md).  
    - Edge Cases: Filename collisions; invalid characters.  
  - **Save JSON file to Google Drive**  
    - Type: Google Drive node  
    - Role: Saves the structured JSON report as a text file in the target folder.  
    - Config: Folder ID set; content from AI summary node.  
    - Edge Cases: Write permission errors; API limits.  
  - **Save Markdown file to Google Drive**  
    - Type: Google Drive node  
    - Role: Saves the markdown report similarly.  
    - Edge Cases: Same as above.  
  - **Save Raw Transcript to Google Drive**  
    - Type: Google Drive node  
    - Role: Saves the raw transcript text as a .txt file.  
    - Edge Cases: Same as above.  
  - **Get JSON File Meta / Get Markdown File Meta**  
    - Type: Google Drive nodes  
    - Role: Retrieve metadata including webViewLink for saved files to include in notifications.  
    - Edge Cases: File not found; permission issues.  
  - **Prepare Response JSON / Markdown**  
    - Type: Set nodes  
    - Role: Store metadata objects for merging and notification dispatch.  
    - Edge Cases: Missing metadata.  

---

#### 2.6 Notification Dispatch

- **Overview:**  
  Sends notifications with links to the generated documents via Telegram and Gmail.

- **Nodes Involved:**  
  - Merge All Paths  
  - Email Content Formatter  
  - Send Gmail Message  
  - Send Telegram Message  

- **Node Details:**  
  - **Merge All Paths**  
    - Type: Merge node  
    - Role: Combines outputs from JSON metadata, markdown metadata, and raw transcript save nodes to synchronize notification data.  
    - Config: Combine by position, 3 inputs.  
    - Edge Cases: Missing inputs causing incomplete merges.  
  - **Email Content Formatter**  
    - Type: OpenAI node (LangChain)  
    - Role: Generates an HTML email body with links to the JSON and markdown reports.  
    - Config: GPT-4o-mini; prompt includes styling and formatting instructions.  
    - Edge Cases: HTML rendering issues; missing links.  
  - **Send Gmail Message**  
    - Type: Gmail node  
    - Role: Sends the formatted email to a configured recipient.  
    - Config: Recipient email from environment variable; subject and message from formatter.  
    - Edge Cases: Email delivery failures; OAuth token expiration.  
  - **Send Telegram Message**  
    - Type: Telegram node  
    - Role: Sends a Telegram message with report links to a configured chat ID.  
    - Config: Chat ID from environment variable; message includes webViewLinks.  
    - Edge Cases: Telegram API limits; invalid chat ID.  

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                | Input Node(s)                       | Output Node(s)                          | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|-----------------------------------------------|-----------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------|
| On File Created Trigger      | Google Drive Trigger             | Trigger workflow on new audio file             | —                                 | Gmail User for Approval                 | ## Wait for Google Drive Trigger and Send for User Approval to Proceed (Human in the Loop) (optional) |
| Gmail User for Approval      | Gmail                           | Send approval email and wait for user response| On File Created Trigger            | Search Google Drive                    |                                                                                                |
| Search Google Drive          | Google Drive                    | List files in target folder                     | Gmail User for Approval / Start Workflow | Filter by .m4a extension               | ## 2️⃣ Search and Download Audio File from Google Drive 💡Note:  Adjust Filter and Limit settings for your needs |
| Filter by .m4a extension    | Filter                         | Filter files to .m4a extension                  | Search Google Drive                | Limit to last file                     |                                                                                                |
| Limit to last file           | Limit                         | Keep only the most recent file                  | Filter by .m4a extension           | Download audio file                    |                                                                                                |
| Download audio file          | Google Drive                   | Download selected audio file                     | Limit to last file                 | Transcribe with OpenAI                 |                                                                                                |
| Transcribe with OpenAI       | OpenAI (LangChain)             | Transcribe audio to text                         | Download audio file                | Set Config                           | ## 3️⃣ Transcribe Audio                                                                        |
| Set Config                  | Set                           | Store transcription text and timestamp          | Transcribe with OpenAI             | Summarize to JSON, Summarize to Structured JSON, Save Raw Transcript to Google Drive |                                                                                                |
| Summarize to JSON            | OpenAI (LangChain)             | Generate detailed structured transcript summary | Set Config                       | Convert JSON to Markdown               | ## 4️⃣ Process Transcript and Generate Structured JSON Report                                 |
| Summarize to Structured JSON | OpenAI (LangChain)             | Generate concise structured JSON report         | Set Config                       | Get Filename for JSON                  |                                                                                                |
| Convert JSON to Markdown     | OpenAI (LangChain)             | Convert JSON summary to markdown                 | Summarize to JSON                 | Get Filename for Markdown              | ## 5️⃣ Process Transcript and Generate Structured JSON -> Markdown Report                     |
| Get Filename for JSON        | Set                           | Generate filename for JSON report                | Summarize to Structured JSON      | Save JSON file to Google Drive         |                                                                                                |
| Get Filename for Markdown    | Set                           | Generate filename for markdown report            | Convert JSON to Markdown           | Save Markdown file to Google Drive     |                                                                                                |
| Save JSON file to Google Drive | Google Drive                | Save JSON report file                            | Get Filename for JSON              | Get JSON File Meta                     |                                                                                                |
| Save Markdown file to Google Drive | Google Drive            | Save markdown report file                        | Get Filename for Markdown          | Get Markdown File Meta                 |                                                                                                |
| Save Raw Transcript to Google Drive | Google Drive            | Save raw transcript text file                    | Set Config                       | Merge All Paths                       | ## 6️⃣ Save Raw Transcript to Google Drive                                                    |
| Get JSON File Meta           | Google Drive                   | Retrieve metadata for JSON file                   | Save JSON file to Google Drive     | Prepare Response JSON                  |                                                                                                |
| Get Markdown File Meta       | Google Drive                   | Retrieve metadata for markdown file               | Save Markdown file to Google Drive | Prepare Response Markdown              |                                                                                                |
| Prepare Response JSON        | Set                           | Prepare JSON metadata for merging                 | Get JSON File Meta                | Merge All Paths                       |                                                                                                |
| Prepare Response Markdown    | Set                           | Prepare markdown metadata for merging             | Get Markdown File Meta            | Merge All Paths                       |                                                                                                |
| Merge All Paths              | Merge                         | Combine metadata and raw transcript save outputs | Prepare Response JSON, Prepare Response Markdown, Save Raw Transcript to Google Drive | Send Telegram Message, Email Content Formatter | ## 7️⃣ Send Transcription Report Links to User                                                |
| Email Content Formatter      | OpenAI (LangChain)             | Generate HTML email content with report links    | Merge All Paths                   | Send Gmail Message                    |                                                                                                |
| Send Gmail Message           | Gmail                         | Send email notification with report links        | Email Content Formatter           | —                                    |                                                                                                |
| Send Telegram Message        | Telegram                      | Send Telegram notification with report links     | Merge All Paths                   | —                                    |                                                                                                |
| Start Workflow               | Manual Trigger                | Manual start for testing or manual runs           | —                                 | Search Google Drive                   | ## 1️⃣ Start Transcription Service                                                            |
| Sticky Note (various)        | Sticky Note                   | Visual documentation blocks                        | —                                 | —                                    | See individual sticky note contents in node details                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Type: Google Drive Trigger  
   - Configure to watch the folder "Audio Recordings" (set folder ID) for new files of MIME type audio.  
   - Set polling interval (e.g., every minute).  
   - Optional: Disable if manual trigger preferred.

2. **Create Approval Node (Optional):**  
   - Type: Gmail node  
   - Operation: sendAndWait  
   - Recipient: Set email address via environment variable or static value.  
   - Subject: "💡New Audio File Created - Approve Transcription Service"  
   - Message: Inform user of new audio file and request approval.  
   - Approval type: double approval  
   - Timeout: 45 minutes (customizable)  
   - Connect trigger node output to this node.

3. **Search Google Drive for Files:**  
   - Type: Google Drive node  
   - Operation: List files in folder "Audio Recordings" (folder ID required).  
   - Connect approval node output (or trigger if approval disabled) to this node.

4. **Filter for .m4a Files:**  
   - Type: Filter node  
   - Condition: File name ends with ".m4a" (case-insensitive).  
   - Connect from Search Google Drive node.

5. **Limit to Last File:**  
   - Type: Limit node  
   - Keep last item only.  
   - Connect from Filter node.

6. **Download Audio File:**  
   - Type: Google Drive node  
   - Operation: Download file content using file ID from previous node.  
   - Connect from Limit node.

7. **Transcribe Audio:**  
   - Type: OpenAI node (LangChain)  
   - Resource: Audio  
   - Operation: Transcribe  
   - Credentials: OpenAI API key configured.  
   - Connect from Download audio file node.

8. **Set Config Variables:**  
   - Type: Set node  
   - Assign `text` from transcription output, `datetime` to current timestamp.  
   - Connect from Transcribe node.

9. **Summarize to JSON:**  
   - Type: OpenAI node (LangChain)  
   - Model: GPT-4o-mini  
   - Prompt: Detailed structured transcript summary with executive summary, analysis, observations.  
   - JSON output enabled.  
   - Connect from Set Config node.

10. **Summarize to Structured JSON:**  
    - Type: OpenAI node (LangChain)  
    - Model: GPT-4o-mini  
    - Prompt: Concise JSON report with key points, action items, sentiment.  
    - JSON output enabled.  
    - Connect from Set Config node.

11. **Convert JSON to Markdown:**  
    - Type: OpenAI node (LangChain)  
    - Model: GPT-4o-mini  
    - Prompt: Convert detailed JSON summary to markdown text only.  
    - Connect from Summarize to JSON node.

12. **Generate Filenames:**  
    - Create two Set nodes:  
      - For JSON filename: Combine file ID, name, and timestamp with ".json" extension.  
      - For Markdown filename: Same pattern with ".md" extension.  
    - Connect Summarize to Structured JSON to JSON filename node.  
    - Connect Convert JSON to Markdown to Markdown filename node.

13. **Save JSON Report to Google Drive:**  
    - Type: Google Drive node  
    - Operation: Create file from text  
    - Folder: "Audio Recordings" folder ID  
    - Content: Output of Summarize to Structured JSON node (stringified JSON).  
    - Filename: From JSON filename node.  
    - Connect from JSON filename node.

14. **Save Markdown Report to Google Drive:**  
    - Same as above but content from Convert JSON to Markdown node and filename from Markdown filename node.  
    - Connect from Markdown filename node.

15. **Save Raw Transcript to Google Drive:**  
    - Type: Google Drive node  
    - Operation: Create file from text  
    - Folder: "Audio Recordings" folder ID  
    - Content: Transcription text from Set Config node.  
    - Filename: Combine file ID, name, timestamp with ".txt" extension.  
    - Connect from Set Config node.

16. **Get Metadata for JSON and Markdown Files:**  
    - Two Google Drive nodes to search files by filename (JSON and Markdown respectively).  
    - Retrieve fields: id, webViewLink, name.  
    - Connect from Save JSON and Save Markdown nodes respectively.

17. **Prepare Response Objects:**  
    - Two Set nodes to store metadata objects for JSON and Markdown files.  
    - Connect from respective Get File Meta nodes.

18. **Merge All Paths:**  
    - Type: Merge node  
    - Mode: Combine by position  
    - Inputs: Prepare Response JSON, Prepare Response Markdown, Save Raw Transcript to Google Drive.  
    - Connect all three nodes to this merge node.

19. **Email Content Formatter:**  
    - Type: OpenAI node (LangChain)  
    - Model: GPT-4o-mini  
    - Prompt: Generate HTML email with links to JSON and Markdown reports.  
    - Connect from Merge All Paths node.

20. **Send Gmail Message:**  
    - Type: Gmail node  
    - Recipient: Configured email address.  
    - Subject: "Audio Transcribed and Reports Generated"  
    - Message: Output from Email Content Formatter.  
    - Connect from Email Content Formatter.

21. **Send Telegram Message:**  
    - Type: Telegram node  
    - Chat ID: From environment variable.  
    - Message: Text including webViewLinks from JSON and Markdown metadata.  
    - Connect from Merge All Paths node.

22. **Manual Trigger (Optional):**  
    - Add manual trigger node for testing or manual runs.  
    - Connect to Search Google Drive node to start workflow manually.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow supports optional human approval step with 45-minute timeout and double approval type.                | Configurable in "Gmail User for Approval" node parameters.                                       |
| Google Drive folder ID "1Wqd4zEEb847gFYKoDBbNnXsWEc-kCAm2" used throughout for audio input and output storage. | Replace with your own folder ID in all Google Drive nodes.                                      |
| OpenAI model used: GPT-4o-mini for transcription and summarization tasks.                                      | Requires OpenAI API key credential setup.                                                        |
| Email and Telegram recipients configured via environment variables: EMAIL_ADDRESS_JOE and TELEGRAM_CHAT_ID.    | Set these environment variables in n8n for notifications to work correctly.                      |
| Markdown and JSON reports include webViewLink for easy access to Google Drive files.                            | Ensures users can directly open documents from notifications.                                   |
| To customize file types, adjust the filter node condition for extensions like .mp3 or .wav.                    | Located in "Filter by .m4a extension" node.                                                      |
| To extend notifications, add SMS or Microsoft Teams nodes after the merge node.                                | Workflow modularly supports adding more notification channels.                                  |
| For deeper AI analysis, modify prompts in "Summarize to JSON" and "Summarize to Structured JSON" nodes.        | Prompts are in the messages parameter of these OpenAI nodes.                                    |
| Workflow includes multiple sticky notes for visual guidance in the editor.                                     | Sticky notes provide section headers and tips for each logical block.                           |

---

This document fully describes the workflow structure, node configurations, and logic flow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.