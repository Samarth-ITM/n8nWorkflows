Convert Newsletters into AI Podcasts with GPT-4o Mini and ElevenLabs

https://n8nworkflows.xyz/workflows/convert-newsletters-into-ai-podcasts-with-gpt-4o-mini-and-elevenlabs-6523


# Convert Newsletters into AI Podcasts with GPT-4o Mini and ElevenLabs

### 1. Workflow Overview

This workflow automates the conversion of unread newsletters received via email into an engaging AI-generated podcast episode featuring a conversational dialogue between two distinct AI personas. It leverages AI for script generation and text-to-speech (TTS) synthesis, ultimately sending the finished audio file back via email.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Fetch unread newsletter emails filtered by sender using Gmail.
- **1.2 Dialogue Script Generation:** Use OpenAI’s GPT-4o Mini model to transform newsletter text into a natural, conversational script between two AI voices.
- **1.3 Script Segmentation:** Split the generated dialogue into individual speaker segments for voice synthesis.
- **1.4 Voice Synthesis Loop:** Iterate over each segment, detect the speaker, clean the text, and generate audio clips using ElevenLabs TTS with distinct voices.
- **1.5 Audio Assembly:** Save audio chunks, prepare a file list for FFmpeg, and merge all audio segments into a single MP3 file.
- **1.6 Delivery:** Read the final merged audio and send it as an email attachment to the original newsletter recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow by fetching unread newsletters from the Gmail inbox, filtered by a specific sender address. It serves as the entry point of the entire process.

- **Nodes Involved:**  
  - Get Newsletter (Gmail Trigger)  
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **Get Newsletter**  
    - Type: Gmail Trigger  
    - Role: Polls Gmail every minute for unread emails from "demandcurve.com" (filter via `from:demandcurve.com`).  
    - Configuration: OAuth2 credentials for Gmail; polling every minute; extracts full email content including HTML/plain text.  
    - Input: None (trigger)  
    - Output: Email JSON including body text (used downstream)  
    - Edge Cases: Gmail API rate limits, auth token expiration, emails with no body content or unsupported formats.  
    - Notes: Can be replaced with a Webhook node for integration with external apps.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains the purpose and alternatives for this block.  
    - No technical config.

#### 2.2 Dialogue Script Generation

- **Overview:**  
  This block transforms the raw newsletter text into a scripted dialogue between two AI personas using the OpenAI GPT-4o Mini model.

- **Nodes Involved:**  
  - Generate Dialogue Script (OpenAI Chat)  
  - Sticky Note2 (Documentation)

- **Node Details:**

  - **Generate Dialogue Script**  
    - Type: OpenAI Chat (Langchain node)  
    - Role: Sends newsletter content to GPT-4o Mini model with a prompt to generate a conversational script featuring two voices (`voice1` and `voice2`).  
    - Configuration:  
      - Model: GPT-4o Mini for cost-effective, long-form generation.  
      - Prompt: Detailed instructions to create a lively, informal dialogue with specific voice personalities and a minimum length of 10,000 characters.  
      - Input Expression: `{{$('Get Newsletter').first().json.text}}` to pull newsletter body text.  
      - Output: Single text block with speaker tags (`voice1:`, `voice2:`).  
    - Input: Newsletter text JSON.  
    - Output: JSON with generated dialogue script.  
    - Edge Cases: API rate limits, prompt failures, incomplete text, or insufficient length; incorrect or missing newsletter text input.  

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Describes the creative goal and prompt structure for the dialogue generation.

#### 2.3 Script Segmentation

- **Overview:**  
  Splits the long dialogue script into smaller segments, each tagged by the speaking persona, to prepare for individual voice synthesis.

- **Nodes Involved:**  
  - Split script (Code/Function Node)  
  - Sticky Note3 (Documentation)

- **Node Details:**

  - **Split script**  
    - Type: Code (JavaScript)  
    - Role: Parses generated dialogue text and splits it at `voice1:` and `voice2:` tags into separate items for batch processing.  
    - Logic: Normalizes line breaks, regex split retaining speaker tags, filters out empty segments.  
    - Input: Script text from the OpenAI node.  
    - Output: Array of JSON items, each with a single speaker segment.  
    - Edge Cases: Unexpected script format, missing speaker tags, empty segments.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Explains the purpose of segmenting the script for subsequent TTS calls.

#### 2.4 Voice Synthesis Loop

- **Overview:**  
  Processes each dialogue segment in batches, cleans the text, detects the speaker, prepares the text for TTS, sends requests to ElevenLabs TTS API, and saves the resulting audio.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - Function Node – Clean Segment (Code)  
  - If (Conditional)  
  - Function Node – Prepare Text for TTS (voice1)  
  - Function Node – Prepare Text for TTS1 (voice2)  
  - Function Node – Prepare Text for TTS - Voice 1 (HTTP Request)  
  - Function Node – Prepare Text for TTS - Voice 2 (HTTP Request)  
  - Save Audio Chucks (ReadWriteFile)  
  - Sticky Note4 (Documentation)

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each dialogue segment to process sequentially or in controlled batches.  
    - Input: Array of segmented dialogue items.  
    - Output: Single item per iteration.  

  - **Function Node – Clean Segment**  
    - Type: Code (JavaScript)  
    - Role: Removes problematic characters such as quotation marks and line breaks from the segment text.  
    - Input: Segment JSON with `segment` string.  
    - Output: JSON with `cleanedText`.  
    - Edge Cases: Empty input, missing segment property.

  - **If**  
    - Type: Conditional  
    - Role: Checks if `cleanedText` contains `voice1:` to route the flow to the appropriate voice preparation node.  
    - Input: Cleaned segment text.  
    - Output: Branch to either voice1 or voice2 processing nodes.  
    - Edge Cases: Missing or malformed `cleanedText`.

  - **Function Node – Prepare Text for TTS (voice1)**  
    - Type: Code (JavaScript)  
    - Role: Strips the `voice1:` tag from text and prepares a clean string for TTS.  
    - Input: `cleanedText` containing `voice1:` label.  
    - Output: JSON with `modifiedString`.  
    - Edge Cases: Regex mismatch, missing voice label.

  - **Function Node – Prepare Text for TTS1 (voice2)**  
    - Same as above but for `voice2:` label.

  - **Function Node – Prepare Text for TTS - Voice 1 (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Sends `modifiedString` to ElevenLabs TTS API using voice1’s voice ID and specific voice settings (stability, similarity boost).  
    - Auth: Custom HTTP header with ElevenLabs API key.  
    - Endpoint: `https://api.elevenlabs.io/v1/text-to-speech/uYXf8XasLslADfZ2MB4u` (voice1 voice ID).  
    - Output: Binary MP3 audio file.  
    - Edge Cases: API failures, invalid API key, network timeout.

  - **Function Node – Prepare Text for TTS - Voice 2 (HTTP Request)**  
    - Same as above but for voice2’s voice ID: `https://api.elevenlabs.io/v1/text-to-speech/UgBBYS2sOqTuMpoF3BR0`.

  - **Save Audio Chucks**  
    - Type: ReadWriteFile  
    - Role: Saves each audio segment as a separate MP3 file named `audio_{{$itemIndex}}.mp3` in a temporary directory.  
    - Edge Cases: File system permissions, disk space.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Describes the iterative voice generation process and how to support the template by using ElevenLabs affiliate links.

#### 2.5 Audio Assembly

- **Overview:**  
  This block consolidates all generated audio chunks into a single MP3 file using FFmpeg concat method, preparing it for delivery.

- **Nodes Involved:**  
  - Generate `concat_list.txt` (Code)  
  - Save concat_list (ReadWriteFile)  
  - Join audio chucks and delete all files (Execute Command)  
  - Sticky Note5 (Documentation)

- **Node Details:**

  - **Generate `concat_list.txt`**  
    - Type: Code (JavaScript)  
    - Role: Builds a text file listing all audio chunk files in FFmpeg concat format (`file 'audio_1.mp3'` per line).  
    - Converts the text list into Base64 binary for saving.  
    - Input: List of audio chunk file names.  
    - Output: Binary data for `concat_list.txt`.  
    - Edge Cases: Missing audio files, inconsistent naming.

  - **Save concat_list**  
    - Type: ReadWriteFile  
    - Role: Writes the Base64-encoded `concat_list.txt` to the temporary directory.

  - **Join audio chucks and delete all files**  
    - Type: Execute Command  
    - Role: Runs an FFmpeg command to concatenate all audio chunks losslessly into `final_merged.mp3`.  
    - Command:  
      ```
      ffmpeg -y -f concat -safe 0 -i /newsletter2podcast/tmp/concat_list.txt -c copy /newsletter2podcast/tmp/final_merged.mp3
      find /newsletter2podcast/tmp/ -type f ! -name "final_merged.mp3" -delete
      ```  
    - Requirements: FFmpeg must be installed locally; not compatible with n8n Cloud or restricted environments.  
    - Edge Cases: FFmpeg not installed, file access errors, audio format mismatches.

  - **Sticky Note5**  
    - Type: Sticky Note  
    - Role: Details the FFmpeg concatenation process, usage notes, and troubleshooting tips.

#### 2.6 Delivery

- **Overview:**  
  The final block reads the merged audio file and sends it as an email attachment back to the newsletter recipient.

- **Nodes Involved:**  
  - read final_merged (ReadWriteFile)  
  - Send audio (Gmail)  
  - Sticky Note6 (Documentation)

- **Node Details:**

  - **read final_merged**  
    - Type: ReadWriteFile  
    - Role: Reads the final MP3 audio file from the temporary directory for sending.  
    - Output: Binary audio file data.

  - **Send audio**  
    - Type: Gmail Send  
    - Role: Sends an email to the original newsletter recipient with the merged podcast attached.  
    - Configuration:  
      - Recipient: Dynamic from `Get Newsletter` node (`to.text`).  
      - Subject: Prefixes "[Audio Version]" to original email subject.  
      - Body: HTML content greeting the user.  
      - Attachment: Binary MP3 file from previous node, property `data`, named `newsletter_audio.mp3`.  
      - Auth: Gmail OAuth2 credentials.  
    - Edge Cases: Email send failure, attachment size limits, invalid recipient address.

  - **Sticky Note6**  
    - Type: Sticky Note  
    - Role: Explains the final steps and customization options for delivery.

---

### 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                                  | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                          |
|----------------------------------|----------------------------|-------------------------------------------------|-------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Sticky Note                      | Sticky Note                | Workflow overview and general explanation       | None                          | None                                   | 🎧 Newsletter-to-Audio Conversation Flow: Explains the full workflow concept and inspiration.                        |
| Get Newsletter                   | Gmail Trigger              | Fetch unread newsletters from Gmail              | None                          | Generate Dialogue Script                | Describes entry point, can be replaced with Webhook for integration.                                                  |
| Sticky Note1                    | Sticky Note                | Explains Get Newsletter node                      | None                          | None                                   | Step 1: Input reception and alternatives.                                                                             |
| Generate Dialogue Script         | OpenAI Chat (Langchain)    | Create conversational script from newsletter     | Get Newsletter                | Split script                           | Step 2: Dialogue generation with GPT-4o Mini, prompt details.                                                        |
| Sticky Note2                    | Sticky Note                | Explains dialogue generation node                 | None                          | None                                   | Step 2: Script generation details.                                                                                    |
| Split script                   | Code (Function)            | Split script into speaker segments                 | Generate Dialogue Script      | Loop Over Items                        | Step 3: Splitting dialogue for TTS processing.                                                                        |
| Sticky Note3                    | Sticky Note                | Explains script segmentation                       | None                          | None                                   | Step 3: Script segmentation purpose.                                                                                  |
| Loop Over Items                 | SplitInBatches             | Iterate over dialogue segments for TTS            | Split script                 | Save Audio Chucks, Function Node – Clean Segment | Step 4–6: Loop for voice generation.                                                                                  |
| If                             | Conditional (If)           | Detect speaker to branch flow                       | Function Node – Clean Segment | Function Node – Prepare Text for TTS / Prepare Text for TTS1 | Step 4–6: Speaker detection.                                                                                           |
| Function Node – Clean Segment    | Code (Function)            | Clean segment text of quotes and line breaks       | Loop Over Items              | If                                    | Step 4–6: Text cleaning for TTS.                                                                                      |
| Function Node – Prepare Text for TTS  | Code (Function)            | Prepare cleaned text for voice1 TTS                 | If (voice1 branch)           | Function Node – Prepare Text for TTS - Voice 1 | Step 4–6: Remove voice1 label.                                                                                        |
| Function Node – Prepare Text for TTS1 | Code (Function)            | Prepare cleaned text for voice2 TTS                 | If (voice2 branch)           | Function Node – Prepare Text for TTS - Voice 2 | Step 4–6: Remove voice2 label.                                                                                        |
| Function Node – Prepare Text for TTS - Voice 1 | HTTP Request              | Send voice1 text to ElevenLabs TTS API              | Function Node – Prepare Text for TTS | Loop Over Items                    | Step 4–6: ElevenLabs TTS call for voice1.                                                                             |
| Function Node – Prepare Text for TTS - Voice 2 | HTTP Request              | Send voice2 text to ElevenLabs TTS API              | Function Node – Prepare Text for TTS1 | Loop Over Items                    | Step 4–6: ElevenLabs TTS call for voice2.                                                                             |
| Save Audio Chucks               | ReadWriteFile              | Save each audio chunk as MP3 file                   | Loop Over Items             | Generate `concat_list.txt`              | Step 4–6: Save audio segments to disk.                                                                                |
| Sticky Note4                    | Sticky Note                | Explains loop and voice generation process          | None                       | None                                   | Step 4–6: Details on voice generation with ElevenLabs.                                                                |
| Generate `concat_list.txt`       | Code (Function)            | Create FFmpeg concat list file content               | Save Audio Chucks           | Save concat_list                      | Step 7: Generate concat_list.txt for FFmpeg.                                                                          |
| Save concat_list                | ReadWriteFile              | Save the concat list file to disk                    | Generate `concat_list.txt`   | Join audio chucks and delete all files | Step 7: Save concat_list.txt file.                                                                                     |
| Join audio chucks and delete all files | Execute Command            | Merge audio files into one MP3 and cleanup          | Save concat_list             | read final_merged                     | Step 8: FFmpeg concat and cleanup.                                                                                    |
| Sticky Note5                    | Sticky Note                | Explains audio merge step with FFmpeg                | None                       | None                                   | Step 7–8: FFmpeg merge instructions and notes.                                                                        |
| read final_merged              | ReadWriteFile              | Read merged MP3 audio file                           | Join audio chucks and delete all files | Send audio                         | Step 9: Read final audio for sending.                                                                                 |
| Send audio                     | Gmail                      | Send final audio as email attachment                  | read final_merged           | None                                   | Step 9: Send podcast audio via Gmail with dynamic recipient and subject.                                               |
| Sticky Note6                   | Sticky Note                | Explains final sending step                            | None                       | None                                   | Step 9: Delivery details and customization notes.                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node "Get Newsletter":**  
   - Node type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail.  
   - Set filter to: `from:demandcurve.com` (or your preferred newsletter sender).  
   - Poll every minute or desired interval.  
   - Output: Email content including body text.

2. **Add OpenAI Chat node "Generate Dialogue Script":**  
   - Node type: OpenAI Chat (Langchain).  
   - Credentials: OpenAI API key configured.  
   - Model: `gpt-4o-mini`.  
   - Message prompt: Insert detailed prompt instructing to convert newsletter content into a conversation between two voices (`voice1` and `voice2`), with at least 10,000 characters.  
   - Input: Use expression to reference newsletter body text: `{{$('Get Newsletter').first().json.text}}`.

3. **Add Function node "Split script":**  
   - Node type: Code  
   - Code: Split text on `voice1:` and `voice2:` tags, keep tags, filter empty.  
   - Input: Output from "Generate Dialogue Script".

4. **Add SplitInBatches node "Loop Over Items":**  
   - To iterate over each dialogue segment individually.  
   - Input: Output from "Split script".

5. **Add Function node "Function Node – Clean Segment":**  
   - Remove quotes and line breaks from each segment.  
   - Input: Current batch item from "Loop Over Items".

6. **Add If node "If":**  
   - Condition: Check if `cleanedText` contains "voice1:" string.  
   - Branch 1: True → Process as voice1.  
   - Branch 2: False → Process as voice2.

7. **For Voice 1 branch:**  
   - Add Function node "Function Node – Prepare Text for TTS" to strip "voice1:" label.  
   - Add HTTP Request node "Function Node – Prepare Text for TTS - Voice 1":  
     - Method: POST  
     - URL: `https://api.elevenlabs.io/v1/text-to-speech/uYXf8XasLslADfZ2MB4u` (replace with your voice1 ID).  
     - Headers: `Content-Type: application/json` and `xi-api-key` with ElevenLabs API key.  
     - Body (JSON):  
       ```json
       {
         "text": "{{ $json.modifiedString }}",
         "model_id": "eleven_multilingual_v2",
         "voice_settings": {
           "stability": 0.5,
           "similarity_boost": 0.75
         }
       }
       ```

8. **For Voice 2 branch:**  
   - Add Function node "Function Node – Prepare Text for TTS1" to strip "voice2:" label.  
   - Add HTTP Request node "Function Node – Prepare Text for TTS - Voice 2":  
     - Same as voice1 but URL with voice2 ID: `https://api.elevenlabs.io/v1/text-to-speech/UgBBYS2sOqTuMpoF3BR0`.

9. **Add ReadWriteFile node "Save Audio Chucks":**  
   - Save the binary audio data from ElevenLabs.  
   - Filename: `/newsletter2podcast/tmp/audio_{{$itemIndex}}.mp3`.

10. **Connect output of "Save Audio Chucks" to Code node "Generate concat_list.txt":**  
    - Code to create FFmpeg concat list file referencing all saved audio chunk filenames.  
    - Output binary Base64 data for text.

11. **Save concat_list.txt via ReadWriteFile node "Save concat_list":**  
    - Path: `/newsletter2podcast/tmp/concat_list.txt`.

12. **Add Execute Command node "Join audio chucks and delete all files":**  
    - Command:  
      ```
      ffmpeg -y -f concat -safe 0 -i /newsletter2podcast/tmp/concat_list.txt -c copy /newsletter2podcast/tmp/final_merged.mp3
      find /newsletter2podcast/tmp/ -type f ! -name "final_merged.mp3" -delete
      ```  
    - Requires FFmpeg installed on host machine.

13. **Add ReadWriteFile node "read final_merged":**  
    - Reads `/newsletter2podcast/tmp/final_merged.mp3` for email attachment.

14. **Add Gmail Send node "Send audio":**  
    - OAuth2 credentials for Gmail.  
    - Recipient: Use expression `={{$('Get Newsletter').first().json.to.text}}`.  
    - Subject: `=[Audio Version] {{$('Get Newsletter').first().json.subject}}`.  
    - Message: HTML content informing user.  
    - Attach binary: property `data` with filename `newsletter_audio.mp3`.

15. **Connect nodes as per described flow:**  
    - Trigger → Generate Script → Split → Loop → Clean → If → Prepare Text → TTS → Save Audio → Generate concat list → Save concat list → Join audio → Read final audio → Send email.

16. **Credentials Setup:**  
    - Gmail OAuth2 for Gmail nodes.  
    - OpenAI API key for Generate Dialogue Script.  
    - ElevenLabs API key for HTTP Request nodes.

17. **Environment Requirements:**  
    - FFmpeg installed and accessible for audio merging.  
    - Sufficient disk space and permissions for temporary audio storage.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is inspired by Google’s NotebookLM approach to converting dense text into human-like dialogue. | Sticky Note in workflow overview.                                                                 |
| Support the template creator by signing up to ElevenLabs using their affiliate link (no extra cost).        | [Click here to support via ElevenLabs](https://try.elevenlabs.io/ds0cvdfiufax)                     |
| FFmpeg must be installed locally for the audio merging steps; this workflow cannot merge audio on n8n Cloud.| Sticky Note5 and block 2.5.                                                                        |
| Contact Luis Acosta at Luis.acosta@news2podcast.com for help or collaboration on advanced audio & AI projects.| Workflow overview sticky note.                                                                     |
| The prompt used in OpenAI node enforces a minimum length (~10,000 characters) and specific conversational style.| Sticky Note2.                                                                                      |
| ElevenLabs voices have customizable settings: stability and similarity boost adjust the voice characteristics.| HTTP Request nodes for TTS.                                                                        |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and extending the "Convert Newsletters into AI Podcasts with GPT-4o Mini and ElevenLabs" workflow. It covers all nodes, their configuration, and highlights potential failure points and environment requirements.