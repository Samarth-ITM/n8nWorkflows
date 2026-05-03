Create .SRT Subtitles & .LRC Lyrics from Audio with Whisper AI and GPT-5-nano

https://n8nworkflows.xyz/workflows/create--srt-subtitles----lrc-lyrics-from-audio-with-whisper-ai-and-gpt-5-nano-9589


# Create .SRT Subtitles & .LRC Lyrics from Audio with Whisper AI and GPT-5-nano

### 1. Workflow Overview

This n8n workflow automates the transformation of uploaded audio files (primarily vocal tracks in MP3 format) into professional subtitle (.SRT) and lyric (.LRC) files using advanced AI models and custom timestamp alignment. It is designed to serve musicians, record labels, content creators, and video editors who require accurate timed transcriptions formatted naturally for singing or rapping.

The workflow is logically divided into the following blocks:

- **1.1 Audio Input & Transcription:**  
  Receives audio file uploads via a form trigger and uses OpenAI’s Whisper API to produce a verbose transcription with word-level timestamps.

- **1.2 AI Lyrics Segmentation:**  
  Processes raw transcription text with GPT-5-nano to segment into natural lyric lines (2–8 words each) without altering the text.

- **1.3 Quality Control Checkpoint:**  
  Conditional routing for manual quality review and correction of lyrics, allowing upload of corrected lyrics or skipping directly to file generation.

- **1.4 Smart Timestamp Alignment:**  
  Uses a custom diff-match algorithm with Levenshtein distance to align manually corrected lyrics back to original Whisper timestamps, handling insertions, deletions, and modifications intelligently.

- **1.5 Subtitle & Lyrics File Preparation:**  
  Generates formatted .SRT and .LRC subtitle files from aligned lyrics and timestamps, applying timing adjustments for readability and synchronization.

- **1.6 Export Ready Files:**  
  Converts generated subtitle contents into downloadable files named dynamically from the original audio filename.

---

### 2. Block-by-Block Analysis

#### 2.1 Audio Input & Transcription

- **Overview:**  
  Accepts an MP3 audio upload via a web form and sends it to OpenAI Whisper API for transcription with detailed word timestamps.

- **Nodes Involved:**  
  - AudioInput  
  - WhisperTranscribe

- **Node Details:**  

  - **AudioInput**  
    - Type: Form Trigger  
    - Role: Entry point to upload audio file and select quality check option (YES/NO)  
    - Config: Accepts single MP3 file, max 25MB; requires QualityCheck radio selection  
    - Inputs: External user upload  
    - Outputs: Binary audio file + form data JSON  
    - Edge cases: File size limit, unsupported formats, missing API key for downstream nodes  

  - **WhisperTranscribe**  
    - Type: HTTP Request  
    - Role: Calls OpenAI Whisper API for transcription  
    - Config: POST to `https://api.openai.com/v1/audio/transcriptions` with multipart-form data; model: whisper-1; response format: verbose JSON including word timestamps  
    - Header Authorization: Must include valid OpenAI API key  
    - Input: Binary audio file from AudioInput  
    - Output: JSON transcription with word-level timing  
    - Edge cases: API authorization errors, network timeouts, malformed audio file  

---

#### 2.2 AI Lyrics Segmentation

- **Overview:**  
  Uses GPT-5-nano chat model to convert raw transcription text into natural lyrics lines, preserving wording but formatting for singing or rap phrasing.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - PostProcessing

- **Node Details:**  

  - **OpenAI Chat Model**  
    - Type: Langchain AI Chat (OpenAI)  
    - Role: Runs GPT-5-nano with prompt to reformat transcription text into lyric-like short lines  
    - Config: Model set to "gpt-5-nano", no additional options  
    - Inputs: Transcription text from WhisperTranscribe  
    - Outputs: Reformatted lyric text  
    - Edge cases: API quota limits, prompt failures, latency  

  - **PostProcessing**  
    - Type: Langchain Chain LLM  
    - Role: Applies the GPT-5-nano prompt on transcription text  
    - Key expression: `{{$json.text}}` as input text; prompt instructs to keep lines short (2-8 words), natural phrasing, no wording changes  
    - Inputs: JSON transcription text  
    - Outputs: Segmented lyric lines in JSON text field  
    - Edge cases: Unexpected output format, empty results  

---

#### 2.3 Quality Control Checkpoint

- **Overview:**  
  Provides a conditional manual quality check: user can choose to skip or upload corrected lyrics to improve timestamp alignment.

- **Nodes Involved:**  
  - RoutingQualityCheck  
  - TranscribedLyrics  
  - QualityCheck  
  - DiffMatch + SrcPrep

- **Node Details:**  

  - **RoutingQualityCheck**  
    - Type: If Node  
    - Role: Routes workflow based on user’s QualityCheck form value ("YES" or "NO")  
    - Condition: If QualityCheck == "YES" → proceed to manual correction; else auto proceed  
    - Inputs: PostProcessing output + AudioInput form data  
    - Outputs: Two branches - manual correction or auto processing  
    - Edge cases: Missing or invalid form input  

  - **TranscribedLyrics**  
    - Type: Convert To File  
    - Role: Converts raw GPT-processed lyrics text into downloadable TXT for manual correction  
    - Config: Text source from `text` property; binary filename includes original audio filename prefixed with TRANSCRIBED_  
    - Inputs: PostProcessing or RoutingQualityCheck outputs  
    - Outputs: Binary text file for download  
    - Edge cases: File conversion errors  

  - **QualityCheck**  
    - Type: Wait (Form Trigger Resume)  
    - Role: Pauses workflow, waits for user to upload corrected lyrics TXT file  
    - Config: Single file input, required, accepts only .txt files  
    - Webhook ID provided for resuming  
    - Inputs: Waits for corrected lyrics upload  
    - Outputs: Binary corrected lyrics file  
    - Edge cases: Upload timeout, wrong file type, empty file  

  - **DiffMatch + SrcPrep**  
    - Type: Code  
    - Role: Aligns corrected lyrics with original Whisper timestamps using Levenshtein distance and custom diff algorithm  
    - Logic:  
      - Reads original Whisper words with timestamps  
      - Reads corrected lyrics text (uploaded or fallback to original)  
      - Performs fuzzy alignment to handle insertions, deletions, modifications  
      - Generates aligned words array with timestamps and types (match/modified/inserted)  
      - Produces improved .LRC and .SRT content with timestamp deduplication and duration adjustments  
      - Calculates statistics of alignment quality  
    - Inputs: Binary corrected lyrics from QualityCheck or raw lyrics if skipped  
    - Outputs: JSON with aligned words, stats, corrected lyrics, LRC and SRT content  
    - Edge cases: Large mismatch between corrected and original, missing timestamps, empty corrections, performance on long lyrics  

---

#### 2.4 Timestamp Matching and Subtitle Preparation

- **Overview:**  
  Processes aligned lyrics to finalize timed segments and formats .SRT and .LRC files.

- **Nodes Involved:**  
  - TimestampMatching  
  - SubtitlesPreparation  
  - SRT  
  - LRC

- **Node Details:**  

  - **TimestampMatching**  
    - Type: Code  
    - Role: Matches lyric lines to Whisper word timestamps with fuzzy matching threshold  
    - Logic:  
      - Splits GPT-segmented lyrics lines  
      - Normalizes words for fuzzy matching (minimum 70% word match ratio)  
      - Assigns start/end timestamps per line from Whisper timestamps or approximates fallback  
    - Inputs: Corrected lyrics text from RoutingQualityCheck (manual or auto), Whisper transcription words  
    - Outputs: Timed lyric segments array  
    - Edge cases: Lines with too few matching words, heavily distorted transcripts  

  - **SubtitlesPreparation**  
    - Type: Code  
    - Role: Converts timed segments into properly formatted .SRT and .LRC content strings  
    - Logic:  
      - Formats timestamps into SRT style (HH:MM:SS,ms) and LRC style ([mm:ss.cs])  
      - Enforces duration constraints and adjusts overlaps for readability  
      - Returns SRT content, LRC content, plain text, and segment count  
    - Inputs: Timed segments from TimestampMatching, corrected lyrics text  
    - Outputs: Formatted subtitle strings  
    - Edge cases: Incorrect timestamps, formatting failures  

  - **SRT**  
    - Type: Convert To File  
    - Role: Converts SRT content string into downloadable .SRT file binary  
    - Config: Filename dynamically set as "SrtFile_OriginalAudioFilename"  
    - Inputs: SRT content from SubtitlesPreparation or DiffMatch + SrcPrep  
    - Outputs: Binary .SRT file  
    - Edge cases: File naming conflicts, encoding errors  

  - **LRC**  
    - Type: Convert To File  
    - Role: Converts LRC content string into downloadable .LRC file binary  
    - Config: Filename dynamically set as "LRC_FILE_OriginalAudioFilename"  
    - Inputs: LRC content from SubtitlesPreparation or DiffMatch + SrcPrep  
    - Outputs: Binary .LRC file  
    - Edge cases: Same as SRT node  

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                                 | Input Node(s)        | Output Node(s)                     | Sticky Note                                                                                                             |
|----------------------|----------------------------------|------------------------------------------------|----------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| AudioInput           | Form Trigger                     | Upload audio file and choose quality check     | —                    | WhisperTranscribe                | ## AUDIO INPUT & TRANSCRIPTION<br>Upload your vocal track (MP3) and let Whisper AI transcribe it with precise timestamps. Works best with clean vocal recordings. |
| WhisperTranscribe    | HTTP Request                    | Transcribe audio with Whisper API               | AudioInput           | PostProcessing                  |                                                                                                                         |
| OpenAI Chat Model    | Langchain AI Chat (OpenAI)       | GPT-5-nano lyrics segmentation                   | WhisperTranscribe    | PostProcessing                  | ## AI LYRICS SEGMENTATION<br>GPT-5-nano formats raw transcription into natural lyric lines (2-8 words per line) while preserving original wording.                 |
| PostProcessing       | Langchain Chain LLM              | Apply GPT-5-nano prompt to segment lyrics       | WhisperTranscribe, OpenAI Chat Model | RoutingQualityCheck           |                                                                                                                         |
| RoutingQualityCheck  | If Node                         | Route based on quality check choice              | PostProcessing       | TranscribedLyrics, TimestampMatching | ## QUALITY CONTROL CHECKPOINT<br>Choose your path:<br>- Auto: Skip to file generation<br>- Manual: Download TXT, make corrections, re-upload for timestamp matching |
| TranscribedLyrics    | Convert To File                 | Export raw lyrics for manual correction          | RoutingQualityCheck  | QualityCheck                   |                                                                                                                         |
| QualityCheck         | Wait (Form Trigger Resume)       | Wait for corrected lyrics upload                 | TranscribedLyrics    | DiffMatch + SrcPrep            |                                                                                                                         |
| DiffMatch + SrcPrep  | Code                           | Align corrected lyrics with timestamps, generate final subtitles and lyrics | QualityCheck          | SRT, LRC                      | ## SMART TIMESTAMP ALIGNMENT<br>Advanced diff & matching algorithm aligns your corrections with original Whisper timestamps using Levenshtein distance.               |
| TimestampMatching    | Code                           | Match lyric lines to Whisper timestamps          | RoutingQualityCheck  | SubtitlesPreparation           |                                                                                                                         |
| SubtitlesPreparation | Code                           | Format timed segments into .SRT and .LRC strings | TimestampMatching    | SRT, LRC                      |                                                                                                                         |
| SRT                  | Convert To File                 | Export .SRT subtitle file                         | SubtitlesPreparation, DiffMatch + SrcPrep | —                            | ## EXPORT READY FILES<br>Generate professional subtitle files:<br>- .SRT for YouTube & video platforms<br>- .LRC for Musixmatch & streaming services                 |
| LRC                  | Convert To File                 | Export .LRC lyrics file                           | SubtitlesPreparation, DiffMatch + SrcPrep | —                            |                                                                                                                         |
| Sticky Note          | Sticky Note                    | Explains Quality Control checkpoint               | —                    | —                              | ## QUALITY CONTROL CHECKPOINT<br>Choose your path:<br>- Auto: Skip to file generation<br>- Manual: Download TXT, make corrections, re-upload for timestamp matching |
| Sticky Note1         | Sticky Note                    | Explains AI lyrics segmentation                   | —                    | —                              | ## AI LYRICS SEGMENTATION<br>GPT-5-nano formats raw transcription into natural lyric lines (2-8 words per line) while preserving original wording.                 |
| Sticky Note2         | Sticky Note                    | Explains Audio Input & Transcription              | —                    | —                              | ## AUDIO INPUT & TRANSCRIPTION<br>Upload your vocal track (MP3) and let Whisper AI transcribe it with precise timestamps. Works best with clean vocal recordings. |
| Sticky Note3         | Sticky Note                    | Explains export file formats                       | —                    | —                              | ## EXPORT READY FILES<br>Generate professional subtitle files:<br>- .SRT for YouTube & video platforms<br>- .LRC for Musixmatch & streaming services                 |
| Sticky Note4         | Sticky Note                    | Explains smart timestamp alignment                | —                    | —                              | ## SMART TIMESTAMP ALIGNMENT<br>Advanced diff & matching algorithm aligns your corrections with original Whisper timestamps using Levenshtein distance.               |
| Sticky Note5         | Sticky Note                    | General overview and key features of the workflow| —                    | —                              | ## 🎵 SUBTITLE & LYRICS GENERATOR WITH WHISPER AI<br>Transform vocal tracks into professional subtitle and lyrics files with AI-powered transcription and intelligent segmentation.<br><br>KEY FEATURES:<br>- Whisper AI transcription with word-level timestamps<br>- GPT-5-nano intelligent lyrics segmentation (2-8 words/line)<br>- Optional quality check with manual correction workflow<br>- Smart timestamp alignment using Levenshtein distance<br>- Dual output: .SRT (YouTube/video) + .LRC (streaming/Musixmatch)<br>- No disk storage - download files directly<br>- Supports multiple languages via ISO codes<br><br>PERFECT FOR:<br>Musicians • Record Labels • Content Creators • Video Editors<br><br>⚡ Upload MP3 → AI processes → Download professional subtitle files |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger: AudioInput**  
   - Type: Form Trigger  
   - Configure webhook to receive form data  
   - Add two form fields:  
     - File upload field labeled "Audio File", accept `.mp3`, required, single file  
     - Radio field labeled "QualityCheck" with options "YES" and "NO", required  
   - Set form title: "Upload audio file (max 25mb)" and description accordingly  
   - Position as start node  

2. **Add HTTP Request: WhisperTranscribe**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.openai.com/v1/audio/transcriptions`  
   - Content-Type: multipart/form-data  
   - Add body parameters:  
     - `file`: formBinaryData from AudioInput's uploaded audio file  
     - `model`: whisper-1  
     - `response_format`: verbose_json  
     - `timestamp_granularities[]`: word  
   - Add header parameter:  
     - Authorization: `Bearer YOUR API KEY` (replace accordingly)  
   - Connect AudioInput output to WhisperTranscribe input  

3. **Add Langchain AI Chat Node: OpenAI Chat Model**  
   - Type: Langchain AI Chat (OpenAI)  
   - Model: gpt-5-nano  
   - Credentials: OpenAI API key configured  
   - No special options  
   - Connect WhisperTranscribe output to this node's input  

4. **Add Langchain Chain LLM: PostProcessing**  
   - Type: Chain LLM  
   - Set input text expression: `{{$json.text}}` (from WhisperTranscribe)  
   - Add prompt message:  
     "You are helping with preparing song lyrics for musicians. Take the following transcription and split it into lyric-like lines. Keep lines short (2–8 words), natural for singing/rap phrasing, and do not change the wording."  
   - Connect OpenAI Chat Model output to PostProcessing input  

5. **Add If Node: RoutingQualityCheck**  
   - Type: If  
   - Condition: If `{{$node["AudioInput"].json["QualityCheck"]}}` equals "YES"  
   - Connect PostProcessing output to this node  

6. **Add Convert To File: TranscribedLyrics**  
   - Type: Convert To File  
   - Operation: toText  
   - Source Property: `text` from PostProcessing  
   - Binary Property Name: `=TRANSCRIBED_{{ $node["AudioInput"].json["Audio File"].filename }}`  
   - Connect RoutingQualityCheck "YES" output to this node  

7. **Add Wait Node: QualityCheck**  
   - Type: Wait (Form Trigger Resume)  
   - Configure webhook to wait for a file upload  
   - Form field: Single file upload, label "Corrected lyrics", accept `.txt`, required  
   - Connect TranscribedLyrics output to QualityCheck input  

8. **Add Code Node: DiffMatch + SrcPrep**  
   - Type: Code  
   - Paste provided JavaScript code for diff-match alignment, timestamp interpolation, and subtitle generation  
   - Input: Binary corrected lyrics (from QualityCheck upload) or fallback to original  
   - Output: JSON with alignedWords, stats, correctedLyrics, lrcContent, srtContent  
   - Connect QualityCheck output to this node  

9. **Add Code Node: TimestampMatching**  
   - Type: Code  
   - Paste provided JavaScript code to fuzzy match lyric lines to Whisper word timestamps  
   - Input: Output from RoutingQualityCheck "NO" path (PostProcessing) and Whisper transcription words  
   - Output: Timed lyric segments  
   - Connect RoutingQualityCheck "NO" output and WhisperTranscribe output accordingly  

10. **Add Code Node: SubtitlesPreparation**  
    - Type: Code  
    - Paste provided JavaScript code to format .SRT and .LRC files from timed segments  
    - Input: TimestampMatching output  
    - Output: srtContent, lrcContent, segmentCount, plainText  
    - Connect TimestampMatching output to this node  

11. **Add Convert To File Nodes: SRT and LRC**  
    - Type: Convert To File (twice)  
    - Both operation: toText  
    - SRT node sourceProperty: `srtContent`  
    - SRT binaryPropertyName: `=SrtFile_{{ $node["AudioInput"].json["Audio File"].filename }}`  
    - LRC node sourceProperty: `lrcContent`  
    - LRC binaryPropertyName: `=LRC_FILE_{{ $node["AudioInput"].json["Audio File"].filename }}`  
    - Connect SubtitlesPreparation output to both SRT and LRC nodes (for auto flow)  
    - Also connect DiffMatch + SrcPrep output to both SRT and LRC nodes (for manual correction flow)  

12. **Set Credentials**  
    - Configure OpenAI API credentials for WhisperTranscribe and OpenAI Chat Model nodes  

13. **Add Sticky Notes** (Optional for documentation in the editor)  
    - Add all sticky notes content as per their descriptions positioned near relevant nodes  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 🎵 SUBTITLE & LYRICS GENERATOR WITH WHISPER AI: Full workflow overview and key features summary | General project description, perfect for musicians, labels, content creators, video editors         |
| QUALITY CONTROL CHECKPOINT: Choice between automatic processing or manual correction workflow     | Explains user decision point on quality control and correction upload                               |
| AI LYRICS SEGMENTATION: GPT-5-nano splits transcription into natural lyric lines                 | Details AI model usage for lyric formatting                                                        |
| AUDIO INPUT & TRANSCRIPTION: Upload MP3, Whisper AI transcription with word timestamps           | Describes input expectations and transcription accuracy requirements                               |
| SMART TIMESTAMP ALIGNMENT: Uses Levenshtein distance for aligning corrected lyrics with timestamps | Critical explanation of alignment algorithm improving timestamp accuracy after manual edits        |
| EXPORT READY FILES: Generates .SRT and .LRC files suitable for YouTube, Musixmatch, streaming     | Clarifies final output formats and their intended platforms                                        |

---

This documentation provides a thorough, structured explanation and stepwise guide for reproducing, understanding, and troubleshooting the entire workflow for generating subtitle and lyric files from audio using Whisper AI and GPT-5-nano in n8n.