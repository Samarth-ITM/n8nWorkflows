Generate Meeting Minutes from Videos with Whisper, Ollama LLM and Notion

https://n8nworkflows.xyz/workflows/generate-meeting-minutes-from-videos-with-whisper--ollama-llm-and-notion-9158


# Generate Meeting Minutes from Videos with Whisper, Ollama LLM and Notion

### 1. Workflow Overview

This workflow automates the process of generating structured meeting minutes from video recordings. It is designed to handle video files placed in a specific folder, transcribe their audio content using a local Whisper model, summarize the transcription into actionable meeting notes using a local Ollama LLM model, and finally organize and store the results in Notion and Google Drive. Additionally, it sends the notes to a Discord channel for team visibility.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Preparation:** Watches a folder for new video files, waits for file readiness, filters supported formats, and manages Google Drive uploads.
- **1.2 Audio Extraction and Transcription:** Converts video to WAV audio, runs Whisper transcription with language detection or forced language based on filename, and saves the transcript.
- **1.3 Meeting Notes Generation:** Uses an Ollama local language model to generate structured meeting minutes from the transcript.
- **1.4 Notion Page Creation and Content Append:** Creates a Notion page, inserts a link to the original video, and appends the generated meeting notes.
- **1.5 Notification:** Sends the meeting notes as messages to a Discord channel.
- **1.6 Utility and Support:** Includes wait nodes to handle file readiness and error-tolerant command execution, plus helper scripts for audio extraction and transcription.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and File Preparation

**Overview:**  
This block monitors a specified folder for new video files, waits until files are fully copied and unlocked, filters for MKV files, creates a dedicated Google Drive folder, uploads the video to that folder, and prepares the file for transcription.

**Nodes Involved:**  
- File (Local File Trigger)  
- Wait (Wait node)  
- Execute Command (runs PowerShell script to wait for file unlock)  
- If (Checks if file is unlocked)  
- Wait1 (Wait node; retry wait)  
- Filter (Filter node to allow only MKV files)  
- Create sub-folder (Google Drive folder creation)  
- Read MKV From Disk (Reads video file from disk)  
- Upload video file (Uploads video to Google Drive)  
- Sticky Notes (Sticky Note1, Sticky Note, Sticky Note6, Sticky Note8 - documenting process and helper script)

**Node Details:**

- **File (Local File Trigger):**  
  - Type: Trigger  
  - Watches folder `G:\OBS\videos` for newly added files.  
  - Configured to await write finish to avoid partial reads.  
  - Outputs file path to subsequent nodes.

- **Wait:**  
  - Pauses workflow 3 seconds to allow for initial file preparation.  
  - Output connects to Execute Command.

- **Execute Command (wait-for-file.ps1):**  
  - Runs a PowerShell script that attempts exclusive read lock on the file path provided by the File node.  
  - Outputs "0" if file is ready/unlocked, "1" otherwise.  
  - On error, continues normal output to avoid halting workflow.

- **If:**  
  - Checks if the Execute Command's stdout equals "0" (file unlocked).  
  - If true, proceeds to Filter node.  
  - If false, loops back to Wait1 for retry.

- **Wait1:**  
  - Waits 3 seconds before retrying file unlock check.

- **Filter:**  
  - Filters input files to only allow those with `.mkv` extension.  
  - Prevents unsupported file types from continuing.

- **Create sub-folder (Google Drive):**  
  - Creates a subfolder named after the base filename (without extension) inside a parent "Meetings" folder (ID provided).  
  - Uses Google Drive API with OAuth2 credentials.

- **Read MKV From Disk:**  
  - Reads the MKV video file from local disk for upload.

- **Upload video file:**  
  - Uploads the video file to the created Google Drive folder.  
  - Uses OAuth2 Google Drive credentials.  
  - Outputs the Google Drive file metadata including a sharable link.

- **Sticky Notes:**  
  - Document the file-waiting mechanism and scripts used (wait-for-file.ps1).  
  - Note on upload process.

**Edge Cases / Failure Modes:**  
- File locked or incomplete → retries wait.  
- Non-MKV files → filtered out.  
- Google Drive API auth failures or quota limits.  
- File read/write errors.  
- PowerShell script failures (caught and logged).

---

#### 1.2 Audio Extraction and Transcription

**Overview:**  
Converts the uploaded video file to WAV format using a Python script, then runs a local Whisper transcription script that extracts timestamps and supports automatic or forced language detection based on filename suffix.

**Nodes Involved:**  
- Create Wav (Execute Command, runs create_wav.py)  
- Transcribe Local (Execute Command, runs transcribe_return.py)  
- Save Transcript File (Execute Command)  
- Set Transcript Data (Set node)  
- Sticky Notes (Sticky Note2, Sticky Note9, Sticky Note10)

**Node Details:**

- **Create Wav:**  
  - Runs `create_wav.py` with the video file path as argument.  
  - Converts the video audio track to WAV with constraints to keep file size under 25MB.  
  - Uses ffmpeg with dynamic bitrate and codec selection.

- **Transcribe Local:**  
  - Runs `transcribe_return.py` with the video file path as argument.  
  - Extracts audio, transcribes with Whisper model "small".  
  - Supports language detection or forced language based on filename suffix (e.g., .es for Spanish).  
  - Outputs transcript with timestamps to stdout.

- **Save Transcript File:**  
  - Saves the transcript output to a `.transcript.txt` file alongside the original video file.

- **Set Transcript Data:**  
  - Extracts transcript from previous node output.  
  - Adds additional metadata fields: transcript text, "scripts" label, and Google Drive link to uploaded video.  
  - Prepares data for meeting notes generation.

- **Sticky Notes:**  
  - Contain Python scripts `create_wav.py` and `transcribe_return.py` with detailed explanations and usage.  
  - Document the transcription approach and audio extraction.

**Edge Cases / Failure Modes:**  
- ffmpeg errors during WAV conversion.  
- Whisper transcription failure or timeouts.  
- Language detection might be incorrect if filename suffix is missing or ambiguous.  
- File system permission errors saving transcript.  
- Script runtime errors or missing Python environment.

---

#### 1.3 Meeting Notes Generation

**Overview:**  
Generates professional meeting notes from the transcript using a local Ollama language model. The prompt is carefully structured for summarization into sections including topics, decisions, action items, blockers, and next steps.

**Nodes Involved:**  
- Create Notes (LangChain LLM node using Ollama)  
- Save Notes File (Execute Command)  
- Sticky Note4

**Node Details:**

- **Create Notes:**  
  - Type: LangChain LLM node with Ollama integration.  
  - Model: `gpt-oss:20b` specified with large context and batch sizes.  
  - Prompt instructs the model to produce concise, bullet-pointed meeting minutes referencing the transcript and video link.  
  - Uses transcript and file path variables in prompt via expressions.  
  - Runs once per transcript.

- **Save Notes File:**  
  - Saves the generated meeting notes text to a `.notes.txt` file on disk.

- **Sticky Notes:**  
  - Describe the notes generation process using local Ollama model.

**Edge Cases / Failure Modes:**  
- Ollama model not responsive or errors.  
- Prompt formatting mistakes causing incomplete results.  
- Output size exceeding limits.  
- File write permission issues.

---

#### 1.4 Notion Page Creation and Content Append

**Overview:**  
Creates a new Notion page for the meeting notes, adds a heading block with a link to the uploaded video, and appends the meeting notes content in blocks. The notes are split into smaller chunks to fit Notion block size limits.

**Nodes Involved:**  
- Create a page (Notion node)  
- Paste Video URL (Notion node)  
- Create Notes (LangChain LLM node) [from previous block]  
- Parse for Notion (Code node, splits notes text into 2000-character chunks)  
- Loop Over Items (SplitInBatches node)  
- Append a block (Notion node)  
- Sticky Note3, Sticky Note5

**Node Details:**

- **Create a page:**  
  - Creates a new page inside a specific parent page ID in Notion.  
  - Page title extracted from video filename (filename without extension).  
  - Uses Notion API credentials.

- **Paste Video URL:**  
  - Appends a heading block to the newly created page.  
  - Text is a clickable link labeled with the video filename pointing to the Google Drive URL.

- **Parse for Notion:**  
  - Custom JavaScript splits the notes text into chunks of max 2000 characters to comply with Notion block size limits.  
  - Returns an array of chunk objects.

- **Loop Over Items:**  
  - Processes each chunk in separate batches to append them to Notion.

- **Append a block:**  
  - For each chunk, appends a rich text block to the Notion page with the chunk content.

- **Sticky Notes:**  
  - Describe the Notion page creation and appending logic.

**Edge Cases / Failure Modes:**  
- Notion API quota or rate limits.  
- Invalid page or block IDs.  
- Chunk splitting errors resulting in incomplete notes.  
- Network errors in Notion API calls.

---

#### 1.5 Notification

**Overview:**  
Sends the generated meeting notes chunks as messages to a Discord channel via webhook for team awareness.

**Nodes Involved:**  
- Discord  
- Loop Over Items [shared with Notion append loop]

**Node Details:**

- **Discord:**  
  - Posts each chunk of the meeting notes as a separate message in a Discord channel.  
  - Uses a configured Discord webhook credential.  
  - Message content is the chunk string from the loop node.

- **Loop Over Items:**  
  - The same node that batches the notes chunks feeds both the Notion append and Discord message nodes.

**Edge Cases / Failure Modes:**  
- Discord webhook errors or rate limits.  
- Large messages split incorrectly.  
- Webhook credential invalid or revoked.

---

#### 1.6 Utility and Support

**Overview:**  
Includes various wait nodes and sticky notes documenting helper scripts and workflow logic for maintainability.

**Nodes Involved:**  
- Wait, Wait1 (Wait nodes for polling)  
- Sticky Notes (Sticky Note1, Sticky Note6, Sticky Note7, Sticky Note8, Sticky Note9, Sticky Note10)  
- gpt-oss 4090 (Ollama LLM base node)  

**Node Details:**  
- Wait nodes used to delay retries for file readiness and ensure smooth execution.  
- Sticky notes hold embedded helper script code and explanations for the PowerShell file lock check and Python transcription scripts.  
- The gpt-oss 4090 node acts as the backend language model invoked by the Create Notes node.

**Edge Cases:**  
- Delays may lead to longer processing times on slow file copies.  
- Documentation notes help troubleshooting.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                             | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                 |
|---------------------|---------------------------|---------------------------------------------|------------------------|---------------------------|--------------------------------------------------------------------------------------------|
| File                | Local File Trigger        | Trigger workflow on new video file           |                        | Wait                      | ## New File Trigger                                                                        |
| Wait                | Wait                      | Wait 3 seconds before checking file lock    | File                   | Execute Command            | ## Wait for file copy to finish                                                            |
| Execute Command      | Execute Command           | Run PowerShell to check file lock            | Wait                   | If                        | ## wait-for-file.ps1 script included in sticky note                                        |
| If                  | If                        | Check if file lock check returned "0"        | Execute Command         | Filter, Wait1              |                                                                                            |
| Wait1                | Wait                      | Wait 3 seconds before retrying file lock     | If (false branch)       | Execute Command            |                                                                                            |
| Filter              | Filter                    | Allow only MKV files                          | If (true branch)        | Create sub-folder          |                                                                                            |
| Create sub-folder    | Google Drive              | Create Google Drive folder for meeting       | Filter                  | Read MKV From Disk         | ## Upload to Google Drive                                                                  |
| Read MKV From Disk   | Read/Write File           | Read the video file for upload                | Create sub-folder       | Upload video file          |                                                                                            |
| Upload video file    | Google Drive              | Upload video to Google Drive                  | Read MKV From Disk      | Create Wav                 |                                                                                            |
| Create Wav           | Execute Command           | Convert video audio to WAV                     | Upload video file       | Transcribe Local           | ## Create Wav and Transcribe with Local Whisper Model                                      |
| Transcribe Local     | Execute Command           | Transcribe audio with Whisper                  | Create Wav              | Save Transcript File       | ## transcribe_return.py script included in sticky note                                    |
| Save Transcript File | Execute Command           | Save transcript text to disk                    | Transcribe Local        | Set Transcript Data        |                                                                                            |
| Set Transcript Data  | Set                       | Prepare transcript and metadata for notes     | Save Transcript File    | Create a page              |                                                                                            |
| Create a page        | Notion                    | Create a new Notion page for meeting notes     | Set Transcript Data     | Paste Video URL            | ## Generate Notion Page                                                                    |
| Paste Video URL      | Notion                    | Add video link heading block to Notion page    | Create a page           | Create Notes               |                                                                                            |
| Create Notes        | LangChain LLM (Ollama)    | Generate structured meeting minutes            | Paste Video URL         | Save Notes File            | ## Create Notes with local Ollama Model and parse the info for Notion                     |
| Save Notes File      | Execute Command           | Save meeting notes text to disk                 | Create Notes            | Parse for Notion           |                                                                                            |
| Parse for Notion     | Code                      | Split notes text into Notion-compatible chunks | Save Notes File         | Loop Over Items            |                                                                                            |
| Loop Over Items      | SplitInBatches            | Batch process notes chunks for Notion and Discord | Parse for Notion     | Append a block, Discord    | ## Generate Notion Page and send notification                                             |
| Append a block       | Notion                    | Append notes chunks as blocks to Notion page    | Loop Over Items (batch) | Discord                   |                                                                                            |
| Discord              | Discord                   | Post meeting notes chunks to Discord channel    | Loop Over Items (batch) |                           |                                                                                            |
| gpt-oss 4090         | LangChain Local LLM       | Local Ollama model backend                      | Create Notes (ai_languageModel) | Create Notes (ai_languageModel) |                                                                                            |
| Sticky Note1         | Sticky Note               | Documentation: Wait for file copy to finish    |                        |                           | ## Wait for file copy to finish                                                            |
| Sticky Note           | Sticky Note               | Documentation: Upload to Google Drive           |                        |                           | ## Upload to Google Drive                                                                  |
| Sticky Note2         | Sticky Note               | Documentation: Create WAV and Transcribe steps |                        |                           | ## Create Wav and Transcribe with Local Whisper Model                                      |
| Sticky Note3         | Sticky Note               | Documentation: Generate Notion Page              |                        |                           | ## Generate Notion Page                                                                    |
| Sticky Note4         | Sticky Note               | Documentation: Create Notes with Ollama and parse |                      |                           | ## Create Notes with local Ollama Model and parse the info for Notion                     |
| Sticky Note5         | Sticky Note               | Documentation: Generate Notion Page and send notification |                  |                           | ## Generate Notion Page and send notification                                             |
| Sticky Note6         | Sticky Note               | Documentation: New File Trigger                   |                        |                           | ## New File Trigger                                                                        |
| Sticky Note7         | Sticky Note               | Documentation: Helper Scripts                      |                        |                           | ## Helper Scripts                                                                          |
| Sticky Note8         | Sticky Note               | Embedded PowerShell script for file locking check |                      |                           | ## wait-for-file.ps1                                                                       |
| Sticky Note9         | Sticky Note               | Embedded create_wav.py script                       |                        |                           | ## create_wav.py                                                                           |
| Sticky Note10        | Sticky Note               | Embedded transcribe_return.py script                |                        |                           | ## transcribe_return.py                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Local File Trigger node** named `File`:  
   - Set folder path to monitor: `G:\OBS\videos`  
   - Events: Add  
   - Enable "Await Write Finish" for safe file completion detection.

2. **Add a Wait node** named `Wait`:  
   - Wait 3 seconds.  
   - Connect `File` main output to this node.

3. **Add an Execute Command node** named `Execute Command`:  
   - Command:  
     `powershell.exe -ExecutionPolicy Bypass -File "G:\obs\tools\wait-for-file.ps1" -FilePath "{{ $('Wait').item.json.path }}"`  
   - Set onError to "continueRegularOutput".  
   - Connect `Wait` main output to this node.

4. **Add an If node** named `If`:  
   - Condition: Check if `{{$json.stdout}}` equals `"0"` (string).  
   - Connect `Execute Command` main output to this node.

5. **Add a Wait node** named `Wait1`:  
   - Wait 3 seconds.  
   - Connect `If` false branch to `Wait1`.

6. **Connect `Wait1` main output back to `Execute Command`** to retry file locking check.

7. **Add a Filter node** named `Filter`:  
   - Condition: Input file path ends with `.mkv`.  
   - Connect `If` true branch to this node.

8. **Add a Google Drive node** named `Create sub-folder`:  
   - Operation: Create folder.  
   - Folder name: Base filename of input file (strip extension).  
   - Parent folder ID: The "Meetings" folder ID (e.g., `1EXauY-BVnK-A8-t4fJ_M0C9RghTzIFsO`).  
   - Use appropriate Google Drive OAuth2 credentials.  
   - Connect `Filter` main output to this node.

9. **Add a Read/Write File node** named `Read MKV From Disk`:  
   - File path: Use input file path.  
   - Connect `Create sub-folder` main output to this node.

10. **Add a Google Drive node** named `Upload video file`:  
    - Operation: Upload file.  
    - File name: Extract from input file path.  
    - Parent folder ID: Use output folder ID from `Create sub-folder`.  
    - Input data: Use binary data from `Read MKV From Disk`.  
    - Use Google Drive OAuth2 credentials.  
    - Connect `Read MKV From Disk` main output to this node.

11. **Add an Execute Command node** named `Create Wav`:  
    - Command:  
      `python G:\OBS\tools\create_wav.py "{{ $('File').item.json.path }}"`  
    - Connect `Upload video file` main output to this node.

12. **Add an Execute Command node** named `Transcribe Local`:  
    - Command:  
      `python G:\OBS\tools\transcribe_return.py "{{ $('File').item.json.path }}"`  
    - Connect `Create Wav` main output to this node.

13. **Add an Execute Command node** named `Save Transcript File`:  
    - Command:  
      `echo {{ JSON.stringify($json.stdout) }} > "{{ $('File').item.json.path.replace(/\.[^/.]+$/, '.transcript.txt') }}"`  
    - Connect `Transcribe Local` main output to this node.

14. **Add a Set node** named `Set Transcript Data`:  
    - Set variables:  
      - transcript = `{{ $('Transcribe Local').item.json.stdout }}`  
      - scripts = `"Script"`  
      - drive_link = `{{ $('Upload video file').item.json.webViewLink }}`  
    - Connect `Save Transcript File` main output to this node.

15. **Add a Notion node** named `Create a page`:  
    - Operation: Create page.  
    - Parent page ID: Use your Notion parent page ID (e.g., `26ddc80914ee803a9896f7a313c44125`).  
    - Title: Extract filename without extension from `File` node path.  
    - Use Notion API credentials.  
    - Connect `Set Transcript Data` main output to this node.

16. **Add a Notion node** named `Paste Video URL`:  
    - Operation: Append block.  
    - Block ID: Use newly created page ID from `Create a page`.  
    - Block content: Heading 3, text with link labeled by video filename, linking to Google Drive file.  
    - Use Notion API credentials.  
    - Connect `Create a page` main output to this node.

17. **Add a LangChain LLM node** named `Create Notes`:  
    - Model: Ollama model `gpt-oss:20b` with large context size.  
    - Prompt: Detailed meeting notes summarization prompt including transcript and video file path interpolations.  
    - Execute once.  
    - Use Ollama API credentials.  
    - Connect `Paste Video URL` main output to this node.

18. **Add an Execute Command node** named `Save Notes File`:  
    - Command:  
      `echo {{ JSON.stringify($json.text) }} > "{{ $('File').item.json.path.replace(/\.[^/.]+$/, '.notes.txt') }}"`  
    - Connect `Create Notes` main output to this node.

19. **Add a Code node** named `Parse for Notion`:  
    - JavaScript code: Split notes text into chunks of max 2000 characters, return array of chunk objects.  
    - Connect `Save Notes File` main output to this node.

20. **Add a SplitInBatches node** named `Loop Over Items`:  
    - Use default batch size.  
    - Connect `Parse for Notion` main output to this node.

21. **Add a Notion node** named `Append a block`:  
    - Operation: Append block.  
    - Block ID: Use page ID from `Create a page`.  
    - Block content: Append chunk text from batches as rich text blocks.  
    - Use Notion API credentials.  
    - Connect `Loop Over Items` first output to this node.

22. **Add a Discord node** named `Discord`:  
    - Authentication: Webhook.  
    - Content: Chunk text from batches.  
    - Use Discord webhook credentials.  
    - Connect `Loop Over Items` second output to this node.

23. **Add sticky notes for documentation** (optional) with embedded helper scripts for `wait-for-file.ps1`, `create_wav.py`, and `transcribe_return.py`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The PowerShell script `wait-for-file.ps1` attempts to open a file exclusively to detect if it is still being copied or locked by another process. It returns "0" if the file is free and "1" otherwise. This helps avoid processing incomplete files.                                                                                                                                                                                                                                              | Embedded in Sticky Note8                                                                          |
| The Python script `create_wav.py` converts the input video to a WAV audio file with constraints to keep the file size below 25 MB, choosing optimal codec and sample rate automatically.                                                                                                                                                                                                                                                                                                               | Embedded in Sticky Note9                                                                          |
| The Python script `transcribe_return.py` extracts audio from video, transcribes using Whisper model "small", supports forced language detection from filename suffix (e.g., `.es` or `.en`), and outputs timestamped transcript to stdout. It cleans up temporary audio files after processing.                                                                                                                                                                                                        | Embedded in Sticky Note10                                                                         |
| The workflow uses a local Ollama model (`gpt-oss:20b`) for generating meeting notes to keep data processing local and private.                                                                                                                                                                                                                                                                                                                                                                    | Ollama API credentials node (`gpt-oss 4090`)                                                    |
| Google Drive folder IDs and Notion parent page IDs must be configured according to your environment. Make sure OAuth2 credentials for Google Drive and Notion API are properly set up.                                                                                                                                                                                                                                                                                                               | Configured in Google Drive and Notion nodes                                                    |
| Discord webhook URL configured in the Discord node allows automatic posting of meeting notes to a team channel for collaboration.                                                                                                                                                                                                                                                                                                                                                                  | Discord webhook credential node                                                                 |
| The workflow tolerates errors in command execution nodes by continuing workflow execution to avoid halts. This allows retries and better fault tolerance.                                                                                                                                                                                                                                                                                                                                          | Execute Command nodes have `onError` set to continueRegularOutput                              |
| For best results, ensure Python environment has `ffmpeg-python`, `whisper`, and `ffmpeg` installed and accessible in the system path.                                                                                                                                                                                                                                                                                                                                                             | Required for custom scripts                                                                     |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow and complies with current content policies. It contains no illegal, offensive, or protected material. All data handled is legal and public.