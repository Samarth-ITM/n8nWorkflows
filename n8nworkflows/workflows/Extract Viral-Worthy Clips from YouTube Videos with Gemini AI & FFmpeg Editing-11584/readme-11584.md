Extract Viral-Worthy Clips from YouTube Videos with Gemini AI & FFmpeg Editing

https://n8nworkflows.xyz/workflows/extract-viral-worthy-clips-from-youtube-videos-with-gemini-ai---ffmpeg-editing-11584


# Extract Viral-Worthy Clips from YouTube Videos with Gemini AI & FFmpeg Editing

---

### 1. Workflow Overview

This workflow automates the extraction of viral-worthy short clips from YouTube videos by leveraging AI analysis and advanced video editing tools. It targets content creators and marketers who want to quickly create engaging TikTok/Shorts-style clips from long-form videos, with professional cropping, trimming, and styled subtitles.

The workflow is logically organized into these main functional blocks:

- **1.1 Input Reception and Video Download**: Receives video details via a form or manual trigger, downloads the YouTube video and its transcript using yt-dlp.
- **1.2 Transcript Processing and Chunking**: Parses and splits the transcript file into manageable chunks for AI processing.
- **1.3 AI-Based Viral Clip Identification**: Uses Google Gemini AI to analyze transcript chunks and identify the best viral clip segments.
- **1.4 Clip Filtering and Preparation**: Aggregates AI outputs, filters top-scoring clips, and prepares clip metadata linked to the downloaded video.
- **1.5 Simple Clipping (Baseline)**: Extracts clips with ffmpeg without editing for initial clip creation.
- **1.6 Detailed Clip Analysis and Editing Planning**: Analyzes each clip with Google Gemini AI to generate precise editing instructions including cropping, trimming, and subtitles.
- **1.7 Execution of Editing Commands Sub-Workflow**: Runs the generated ffmpeg commands to apply the editing steps (crop, trim, subtitles) on clips sequentially.
- **1.8 Notification**: Sends an email notification when all clips are ready.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Video Download

**Overview:**  
Receives input via a form or manual trigger, downloads the YouTube video and its transcript using yt-dlp commands, and extracts the final downloaded video path.

**Nodes Involved:**  
- `On form submission`  
- `When clicking ‘Execute workflow’`  
- `video download with yt-dlp`  
- `get transcript from yt-dlp`  
- `get the downloaded video location`  
- `extract filepath`  

**Node Details:**

- **On form submission**  
  - Type: FormTrigger  
  - Role: Entry point via web form with fields for project name and YouTube URL.  
  - Configuration: Requires video title and YouTube URL.  
  - Output connects to `video download with yt-dlp` and `get transcript from yt-dlp`.  
  - Failure cases: Invalid URL input, webhook issues.

- **When clicking ‘Execute workflow’**  
  - Type: ManualTrigger  
  - Role: Alternative manual start.  
  - Connects similarly to download nodes.

- **video download with yt-dlp**  
  - Type: ExecuteCommand  
  - Role: Downloads YouTube video to `/data/clips/%(title)s.%(ext)s`.  
  - Command template uses input URL from form.  
  - Output: stdout log for further parsing.  
  - Failure: yt-dlp command errors, network issues.

- **get transcript from yt-dlp**  
  - Type: ExecuteCommand  
  - Role: Downloads English auto-generated subtitles (if available).  
  - Output: stdout log.

- **get the downloaded video location**  
  - Type: Code  
  - Role: Parses `video download with yt-dlp` stdout to extract file path using regex for "Merging formats into" or "Destination" lines.  
  - Output: JSON with `downloadedFile` path.  
  - Failure: If stdout format changes or download failed.

- **extract filepath**  
  - Type: Code  
  - Role: Similar parsing for transcript file path from `get transcript from yt-dlp` stdout.  
  - Output: JSON with `downloadedFile` path.

---

#### 1.2 Transcript Processing and Chunking

**Overview:**  
Reads the downloaded transcript file, parses SRT captions, cleans and segments text, and splits into chunks for AI consumption.

**Nodes Involved:**  
- `read srt from disk`  
- `Extract from File`  
- `formating of data`  
- `some more formating`  

**Node Details:**

- **read srt from disk**  
  - Type: ReadWriteFile  
  - Role: Reads transcript file content from disk using path from previous node.  
  - Output: raw SRT text.

- **Extract from File**  
  - Type: ExtractFromFile  
  - Role: Extracts text from binary file content.  
  - Operation: text extraction.

- **formating of data**  
  - Type: Code  
  - Role: Parses SRT text into JSON segments with start/end timestamps and cleaned text.  
  - Key logic: Cleans tags and timestamps, accumulates text per segment.

- **some more formating**  
  - Type: Code  
  - Role: Splits parsed captions into chunks of 150 segments for AI processing to avoid input size limits.

---

#### 1.3 AI-Based Viral Clip Identification

**Overview:**  
Sends transcript chunks to Google Gemini AI with instructions to identify 3-5 viral TikTok-style clips per chunk, returning JSON arrays of proposed clips.

**Nodes Involved:**  
- `viral clips identification`  
- `filter out top clips according to score`  

**Node Details:**

- **viral clips identification**  
  - Type: Google Gemini AI (Langchain)  
  - Role: Analyzes transcript chunks with a system prompt specialized for viral short clip detection.  
  - Input: JSON chunk of captions.  
  - Output: JSON array of clip objects with start, end, hook, and score fields.  
  - Failure: API errors, malformed input, response parse errors.

- **filter out top clips according to score**  
  - Type: Code  
  - Role: Parses AI JSON outputs, merges all clip suggestions, sorts by score, keeps top 10 clips.  
  - Outputs filtered clip objects for next processing.

---

#### 1.4 Clip Filtering and Preparation

**Overview:**  
Merges video metadata and AI clip data, producing actionable clip objects linked to the video file path.

**Nodes Involved:**  
- `wait for both branches to complete and merge`  
- `seperate actionable data items`  

**Node Details:**

- **wait for both branches to complete and merge**  
  - Type: Merge  
  - Role: Synchronizes video path extraction and AI clips filtering outputs.

- **seperate actionable data items**  
  - Type: Code  
  - Role: Combines video path with each clip's start/end/hook/score info into unified objects for clipping.

---

#### 1.5 Simple Clipping (Baseline)

**Overview:**  
Cuts raw clips from the original video using ffmpeg, maintaining original aspect ratio and quality, as a baseline before advanced editing.

**Nodes Involved:**  
- `simple clipping (still in orignal aspect ratio)`  
- `extract all clips paths`  
- `Loop Over Items`  
- `Loop Over Items1`  
- `Read clips from disk`  
- `extract clip file in base64`  
- `convert base64 to actual binary file`  

**Node Details:**

- **simple clipping (still in orignal aspect ratio)**  
  - Type: ExecuteCommand  
  - Role: Runs ffmpeg with `-ss` and `-to` to trim clips from original video path.  
  - Output file named by start and end timestamps.  
  - Executes per clip in batch.  
  - Failure: ffmpeg errors, invalid timestamps.

- **extract all clips paths**  
  - Type: Code  
  - Role: Parses ffmpeg stderr to extract output clip file paths for downstream processing.

- **Loop Over Items** and **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Sequential processing of clips for resource management and downstream nodes.

- **Read clips from disk**, **extract clip file in base64**, **convert base64 to actual binary file**  
  - Role: Reads clipped video files, converts to base64 text, then back to binary form for AI analysis input.

---

#### 1.6 Detailed Clip Analysis and Editing Planning

**Overview:**  
Runs detailed AI analysis per clip to generate a comprehensive editing plan: exact cropping, trimming, subtitle placement, styling, and other instructions.

**Nodes Involved:**  
- `Analyze the actual whole video`  
- `extract all actionable operations`  
- `Filterout the not required operations`  
- `Aggregate`  

**Node Details:**

- **Analyze the actual whole video**  
  - Type: Google Gemini AI (Langchain)  
  - Role: Analyzes clip video binary with strict JSON output format describing editing instructions (cropping, trimming, subtitles).  
  - Includes strict timestamp validation and formatting rules.

- **extract all actionable operations**  
  - Type: Code  
  - Role: Parses AI JSON response, creates a sequential pipeline of ffmpeg commands for each editing step (trim+crop combined, separate trim or crop, subtitles creation, burning subtitles, audio normalization).  
  - Handles edge cases like missing timestamps or invalid JSON.

- **Filterout the not required operations**  
  - Type: Filter  
  - Role: Filters out disabled tasks to keep only enabled editing commands.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all filtered operations into a single array for batch execution.

---

#### 1.7 Execution of Editing Commands Sub-Workflow

**Overview:**  
Calls a separate sub-workflow to sequentially execute each ffmpeg command generated in the previous step, applying all editing operations on the clips.

**Nodes Involved:**  
- `Call subworkflow`  
- `Loop Over Items1`  
- `Send a message`  
- `Read clips from disk`  

**Node Details:**

- **Call subworkflow**  
  - Type: ExecuteWorkflow  
  - Role: Invokes external workflow "editing" passing the array of editing command objects (`data` parameter).  
  - Ensures modularity and separation of concerns.

- **Loop Over Items1**  
  - Processes sub-workflow results for each edited clip.

- **Send a message**  
  - Type: Gmail  
  - Role: Sends notification email "clips are ready!" to configured Gmail account via OAuth2 credentials.

- **Read clips from disk**  
  - Reads final edited clips for any further processing or delivery.

---

#### 1.8 Notification

**Overview:**  
Notifies the user by email that all clip processing is complete.

**Nodes Involved:**  
- `Send a message` (same as above)

**Node Details:**  
- Uses Gmail OAuth2 credential to send plain text notification email.

---

### 3. Summary Table

| Node Name                          | Node Type                   | Functional Role                             | Input Node(s)                               | Output Node(s)                           | Sticky Note                                                                                                            |
|-----------------------------------|-----------------------------|--------------------------------------------|---------------------------------------------|-----------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | ManualTrigger               | Manual start of workflow                    |                                             | video download with yt-dlp, get transcript from yt-dlp |                                                                                                                        |
| On form submission                | FormTrigger                 | Web form start with video title & URL      |                                             | video download with yt-dlp, get transcript from yt-dlp |                                                                                                                        |
| video download with yt-dlp        | ExecuteCommand              | Downloads YouTube video                      | On form submission, When clicking ‘Execute workflow’ | get the downloaded video location       | ## Initial Download and identification of clips:- \n### used FFMPEG, Gemini & YT-DLP                                      |
| get transcript from yt-dlp        | ExecuteCommand              | Downloads YouTube transcript subtitles      | On form submission, When clicking ‘Execute workflow’ | extract filepath                        | ## Initial Download and identification of clips:- \n### used FFMPEG, Gemini & YT-DLP                                      |
| get the downloaded video location | Code                       | Parses video download stdout for file path | video download with yt-dlp                   | wait for both branches to complete and merge           | ## Initial Download and identification of clips:- \n### used FFMPEG, Gemini & YT-DLP                                      |
| extract filepath                  | Code                       | Parses transcript download stdout for path | get transcript from yt-dlp                    | read srt from disk                      | ## Initial Download and identification of clips:- \n### used FFMPEG, Gemini & YT-DLP                                      |
| read srt from disk                | ReadWriteFile              | Reads transcript file content               | extract filepath                             | Extract from File                      | ## Initial Download and identification of clips:- \n### used FFMPEG, Gemini & YT-DLP                                      |
| Extract from File                 | ExtractFromFile             | Extracts text from binary file               | read srt from disk                           | formating of data                      |                                                                                                                        |
| formating of data                 | Code                       | Parses and cleans SRT into JSON segments    | Extract from File                            | some more formating                    |                                                                                                                        |
| some more formating               | Code                       | Splits transcript segments into chunks      | formating of data                            | viral clips identification            |                                                                                                                        |
| viral clips identification       | Google Gemini AI            | AI identifies viral clip segments            | some more formating                          | filter out top clips according to score |                                                                                                                        |
| filter out top clips according to score | Code                | Parses and filters top scoring clips         | viral clips identification                   | wait for both branches to complete and merge           |                                                                                                                        |
| wait for both branches to complete and merge | Merge             | Merges video path and AI clip data           | get the downloaded video location, filter out top clips according to score | seperate actionable data items          |                                                                                                                        |
| seperate actionable data items   | Code                       | Combines clip data with video path           | wait for both branches to complete and merge | Loop Over Items                      |                                                                                                                        |
| simple clipping (still in orignal aspect ratio) | ExecuteCommand      | Baseline clip extraction with ffmpeg         | Loop Over Items                              | Loop Over Items                      | ## Clipping out                                                                                                         |
| extract all clips paths           | Code                       | Extracts output clip file paths from ffmpeg  | Loop Over Items                              | Loop Over Items1                     | ## Clipping out                                                                                                         |
| Loop Over Items                  | SplitInBatches             | Processes clips sequentially                  | seperate actionable data items                | extract all clips paths, simple clipping (still in orignal aspect ratio) |                                                                                                                        |
| Loop Over Items1                 | SplitInBatches             | Processes clipped files sequentially          | extract all clips paths                       | Send a message, Read clips from disk  |                                                                                                                        |
| Read clips from disk             | ReadWriteFile              | Reads clipped video file                       | Loop Over Items1                              | extract clip file in base64            |                                                                                                                        |
| extract clip file in base64      | ExtractFromFile             | Converts clip file to base64 for AI input     | Read clips from disk                          | convert base64 to actual binary file   |                                                                                                                        |
| convert base64 to actual binary file | ConvertToFile          | Converts base64 back to binary file            | extract clip file in base64                   | Analyze the actual whole video         |                                                                                                                        |
| Analyze the actual whole video  | Google Gemini AI            | AI generates detailed editing instructions    | convert base64 to actual binary file          | extract all actionable operations      | ## Analysis of each clip and extracting required editing operations                                                     |
| extract all actionable operations | Code                     | Parses AI instructions, builds ffmpeg tasks   | Analyze the actual whole video                 | Filterout the not required operations  | ## Analysis of each clip and extracting required editing operations                                                     |
| Filterout the not required operations | Filter                | Filters enabled editing commands               | extract all actionable operations             | Aggregate                            | ## Analysis of each clip and extracting required editing operations                                                     |
| Aggregate                      | Aggregate                  | Aggregates editing commands into one array     | Filterout the not required operations          | Call subworkflow                     |                                                                                                                        |
| Call subworkflow               | ExecuteWorkflow            | Calls external editing workflow                  | Aggregate                                      | Loop Over Items1                     | ## executing editing commands on the clips\nTake this into a seperate workflow, and configure the call sub-workflow node |
| if operation is subtitles       | If                         | Checks if operation is subtitle-related         | Loop Over Items2                               | find height & width, Execute operation on the clip |                                                                                                                        |
| find height & width             | ExecuteCommand              | Runs ffprobe for video dimensions               | if operation is subtitles                      | calculate relative subtitle size      |                                                                                                                        |
| calculate relative subtitle size | Code                      | Constructs ffmpeg command to burn styled subtitles | find height & width                          | burn subtitles                      |                                                                                                                        |
| burn subtitles                 | ExecuteCommand              | Executes ffmpeg command to burn subtitles        | calculate relative subtitle size               | Wait                                |                                                                                                                        |
| Wait                          | Wait                       | Waits 60 seconds between subtitle burn commands | burn subtitles                                 | Loop Over Items2                   |                                                                                                                        |
| Execute operation on the clip   | ExecuteCommand              | Runs ffmpeg commands for editing steps           | if operation is subtitles                      | Wait (according to how powerful your system is and how much ram you have) |                                                                                                                        |
| Wait (according to how powerful your system is and how much ram you have) | Wait | Wait node for system resource management          | Execute operation on the clip                   | Loop Over Items2                   |                                                                                                                        |
| Loop Over Items2               | SplitInBatches             | Processes editing operations sequentially          | Split Out, Wait                                 | if operation is subtitles, EDITING    |                                                                                                                        |
| Split Out                    | SplitOut                   | Splits array of editing tasks into individual items | EDITING                                       | Loop Over Items2                   |                                                                                                                        |
| EDITING                      | ExecuteWorkflowTrigger     | Triggers sub-workflow for clip editing tasks      | Loop Over Items2                                | Split Out                          |                                                                                                                        |
| Send a message                | Gmail                      | Sends notification email after processing clips   | Loop Over Items1                                |                                         |                                                                                                                        |
| Sticky Note                  | StickyNote                 | Workflow overview and instructions                |                                                 |                                         | # 🎬 AI-Powered YouTube Clip Creator ... See full sticky note content above for details                                  |
| Sticky Note1                 | StickyNote                 | Highlights initial download logic                   |                                                 |                                         | ## Initial Download and identification of clips:- \n### used FFMPEG, Gemini & YT-DLP                                      |
| Sticky Note2                  | StickyNote                 | Highlights clip analysis and editing operations      |                                                 |                                         | ## Analysis of each clip and extracting required editing operations                                                     |
| Sticky Note3                  | StickyNote                 | Labels clipping out stage                              |                                                 |                                         | ## Clipping out                                                                                                         |
| Sticky Note4                  | StickyNote                 | Notes on sub-workflow execution for editing          |                                                 |                                         | ## executing editing commands on the clips\nTake this into a seperate workflow, and configure the call sub-workflow node |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Type: `FormTrigger`  
   - Configure fields:  
     - `project name` (required)  
     - `yt URL` (required)  
   - Position: Start node  
   - Optional: Add `ManualTrigger` node as alternative start.

2. **Download YouTube Video**  
   - Node: `ExecuteCommand`  
   - Command: `yt-dlp -o "/data/clips/%(title)s.%(ext)s" "{{ $json["yt URL"] }}"`  
   - Connect from trigger.

3. **Download Transcript (Auto Subtitles)**  
   - Node: `ExecuteCommand`  
   - Command: `yt-dlp --write-auto-sub --sub-lang "en.*,live" --skip-download {{ $json["yt URL"] }} -o "/data/%(title)s"`  
   - Connect from trigger (parallel to video download).

4. **Extract Video File Path**  
   - Node: `Code`  
   - Parse `stdout` from video download node for lines indicating merged or destination file path.  
   - Output JSON with `downloadedFile` key.

5. **Extract Transcript File Path**  
   - Node: `Code`  
   - Similar parsing for transcript download stdout.

6. **Read Transcript File**  
   - Node: `ReadWriteFile`  
   - Use file path from transcript extraction node.

7. **Extract Transcript Text**  
   - Node: `ExtractFromFile`  
   - Operation: text.

8. **Parse and Format Transcript**  
   - Node: `Code`  
   - Parse SRT format into JSON segments with start/end times and cleaned text.

9. **Split Transcript into Chunks**  
   - Node: `Code`  
   - Split parsed captions into chunks of ~150 segments.

10. **Viral Clips Identification via AI**  
    - Node: `Google Gemini` (Langchain)  
    - Model: `models/gemini-2.5-flash`  
    - System prompt instructing to identify 3-5 viral clips with exact timestamps.  
    - Input: JSON chunks.  
    - Output: JSON arrays of clip candidates.

11. **Filter Top Clips by Score**  
    - Node: `Code`  
    - Parse AI outputs, merge, sort descending by score, keep top 10.

12. **Merge Video Info and Clip Data**  
    - Node: `Merge` (Wait for video path and clips data).  
    - Node: `Code`  
    - Combine video path with clips info into single objects.

13. **Simple Baseline Clipping**  
    - Node: `SplitInBatches` for sequential processing.  
    - Node: `ExecuteCommand`  
    - Command: `ffmpeg -ss {{$json.start}} -to {{$json.end}} -i "{{$json.videoPath}}" -c:v libx264 -preset fast -crf 22 -c:a aac -b:a 128k "/data/clips/{{ $json.start }}_{{ $json.end }}.mp4"`  
    - Node: `Code`  
    - Extract output clip paths from ffmpeg stderr.

14. **Read Clips and Prepare for AI Analysis**  
    - Nodes: `ReadWriteFile` → `ExtractFromFile` (binary to base64) → `ConvertToFile` (base64 to binary)

15. **Detailed Clip Analysis with AI**  
    - Node: `Google Gemini` (Langchain)  
    - Model: `models/gemini-2.5-flash`  
    - Input: clip video binary  
    - System prompt enforces strict JSON schema and timestamp rules for editing instructions.

16. **Parse AI Editing Instructions and Build ffmpeg Commands**  
    - Node: `Code`  
    - Parses AI JSON, creates sequential editing steps: trim+crop, trim, crop, generate subtitles (SRT), burn subtitles, audio normalization (optional), final copy.  
    - Handles time parsing, file naming, error cases.

17. **Filter Enabled Commands**  
    - Node: `Filter`  
    - Passes only enabled ffmpeg commands further.

18. **Aggregate Commands**  
    - Node: `Aggregate`  
    - Collects all commands into an array.

19. **Call Editing Sub-Workflow**  
    - Node: `ExecuteWorkflow`  
    - Pass array of editing commands as parameter `data` to sub-workflow "editing".  
    - (Sub-workflow should execute each command sequentially with wait nodes to manage system load.)

20. **Process Sub-Workflow Results**  
    - Node: `SplitInBatches`  
    - Node: `Send a message` (Gmail OAuth2)  
    - Sends notification email when clips are ready.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| 🎬 AI-Powered YouTube Clip Creator: transforms long-form YouTube videos into viral-ready short clips using AI analysis and professional editing. Requires self-hosted n8n with FFmpeg and yt-dlp installed. Credentials needed: Google Gemini API key and Gmail OAuth2. Workflow processes clips sequentially to avoid overload.                                                                                                                                                                                                                                                                                                                                | Sticky Note on workflow overview                                                                         |
| Setup Guides for Dependencies: FFmpeg installation [https://docs.n8n.io/integrations/community-nodes/installation/gui-install/#install-via-npm], yt-dlp installation [https://github.com/yt-dlp/yt-dlp#installation]                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note on requirements                                                                              |
| Google Gemini API key is required: [https://ai.google.dev/]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note on credentials                                                                               |
| Pro Tip: Adjust wait times depending on system RAM and CPU power to balance performance and stability.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note on performance                                                                               |
| Editing sub-workflow is a separate workflow referenced by this main workflow, responsible for executing ffmpeg commands generated by AI analysis. Ensure the sub-workflow accepts 'data' array input and processes editing tasks sequentially.                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note on sub-workflow usage                                                                        |

---

**Disclaimer**: The provided description and analysis are derived solely from an automated n8n workflow export. All data and operations comply with legal and ethical standards. No offensive or protected content is included.

---