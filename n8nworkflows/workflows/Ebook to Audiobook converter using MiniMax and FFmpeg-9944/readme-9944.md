Ebook to Audiobook converter using MiniMax and FFmpeg

https://n8nworkflows.xyz/workflows/ebook-to-audiobook-converter-using-minimax-and-ffmpeg-9944


# Ebook to Audiobook converter using MiniMax and FFmpeg

---

### 1. Workflow Overview

This workflow automates the conversion of an eBook (PDF) into an audiobook by extracting text, converting text segments into audio using the MiniMax TTS API, and merging audio pieces into a single audio file with FFmpeg. It is designed for self-hosted n8n instances where FFmpeg is installed.

**Target Use Cases:**  
- Users who want to convert PDF eBooks into audiobooks automatically.  
- Batch processing of text chunks to manage API rate limits and audio file size.  
- Merging multiple audio chunks into a seamless audiobook.  
- Uploading the final audiobook to Google Drive for easy access and sharing.

**Logical Blocks:**

- **1.1 Input Reception and Text Extraction:** Receives PDF upload, extracts raw text.  
- **1.2 Text Processing and Chunking:** Cleans and splits text into manageable chunks for TTS.  
- **1.3 Text-to-Speech Conversion:** Calls MiniMax API to convert text chunks to audio URLs, downloads audio files.  
- **1.4 Audio File Management:** Names, saves audio chunks, prepares a concat list for FFmpeg.  
- **1.5 Audio Merging:** Uses FFmpeg to concatenate audio chunks into one final MP3.  
- **1.6 Final Output Upload:** Uploads the merged audiobook to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Text Extraction

- **Overview:**  
  This block receives the user's PDF file upload via a form and extracts its textual content.

- **Nodes Involved:**  
  - FORM  
  - EXTRACT TEXT  
  - SPLITS THE TEXT ACCORGING TO RULES (prepares for next block)

- **Node Details:**  

  - **FORM**  
    - Type: Form Trigger  
    - Role: Captures user PDF upload through an interactive form.  
    - Config: Single file upload field labeled "UPLOAD", required.  
    - Inputs: External HTTP request (form submission).  
    - Outputs: Passes binary PDF file data to next node.  
    - Edge Cases: Missing file upload; unsupported file types.  
    - Version: 2.3  

  - **EXTRACT TEXT**  
    - Type: Extract From File  
    - Role: Extracts text from the uploaded PDF binary data.  
    - Config: Operation set to 'pdf', input from FORM node's binary property "UPLOAD".  
    - Inputs: Binary PDF file.  
    - Outputs: JSON with extracted raw text.  
    - Edge Cases: Corrupted PDF, extraction failures, large file size handling.  
    - Version: 1  

  - **SPLITS THE TEXT ACCORGING TO RULES**  
    - Type: Code  
    - Role: Cleans and splits extracted text into chunks of max 500 characters, preserving sentence boundaries.  
    - Key Logic:  
      - Cleans newline and escape characters.  
      - Splits text into sentences, then chunks while respecting character limits.  
      - Outputs array of objects `{ order: <number>, text: <chunk> }`.  
    - Inputs: Extracted raw text JSON.  
    - Outputs: JSON array of text chunks for processing.  
    - Edge Cases: Very long sentences, empty text, encoding issues.  
    - Version: 2  

---

#### 2.2 Text-to-Speech Conversion and Batch Processing

- **Overview:**  
  Processes text chunks in batches of 5, converts each chunk to audio via MiniMax TTS API, and downloads the resulting audio files.

- **Nodes Involved:**  
  - Loop Over Text chunks (5) at a time  
  - MINIMAX TTS  
  - WAITS FOR 5 SECONDS  
  - CONVERTS URL TO AUDIO FILES  
  - GIVES INDEXES TO AUDIO FILES  
  - Save Audio Chucks

- **Node Details:**  

  - **Loop Over Text chunks (5) at a time**  
    - Type: Split In Batches  
    - Role: Controls batch size of 5 text chunks to throttle API calls and resource usage.  
    - Config: batchSize = 5, no special options.  
    - Inputs: JSON array of text chunks.  
    - Outputs: Batches of 5 text chunks.  
    - Edge Cases: Last batch smaller than 5, empty input.  
    - Version: 3  

  - **MINIMAX TTS**  
    - Type: HTTP Request  
    - Role: Sends each text chunk to MiniMax speech-02-hd model for TTS conversion.  
    - Config:  
      - POST to `https://api.replicate.com/v1/models/minimax/speech-02-hd/predictions`  
      - JSON body includes text, pitch=0, speed=1, volume=1, 128kbps mono, emotion=happy, voice_id=Friendly_Person, sample_rate=32000, English normalization and boost enabled.  
      - Auth: HTTP Bearer with token ("Bearer YOUR_TOKEN_HERE account" placeholder).  
      - Header "Prefer: wait" to wait for prediction completion synchronously.  
    - Inputs: Text chunk JSON.  
    - Outputs: JSON with an `output` property containing URL to audio file.  
    - Edge Cases: API rate limits, auth errors, network timeouts, malformed responses.  
    - Version: 4.2  

  - **WAITS FOR 5 SECONDS**  
    - Type: Wait  
    - Role: Adds a 5-second delay between batch executions to respect API rate limits or pacing.  
    - Config: No parameters, fixed 5 seconds.  
    - Inputs/Outputs: Passes data unchanged.  
    - Version: 1.1  

  - **CONVERTS URL TO AUDIO FILES**  
    - Type: HTTP Request  
    - Role: Downloads the audio file from the URL provided by MiniMax TTS node.  
    - Config: URL parameterized as `={{ $json.output }}` to fetch the audio file binary.  
    - Inputs: JSON with URL.  
    - Outputs: Binary audio file data.  
    - Edge Cases: Broken links, HTTP errors, large files, network issues.  
    - Version: 4.2  

  - **GIVES INDEXES TO AUDIO FILES**  
    - Type: Code  
    - Role: Renames the binary audio files with unique indexed keys like `audio 0`, `audio 1`... and sets file names accordingly.  
    - Logic: Iterates items, copies binary data, renames keys and filenames to `audio <index>.mp3`.  
    - Inputs: Binary audio files.  
    - Outputs: Binary audio files with renamed keys.  
    - Edge Cases: Empty input, binary data missing.  
    - Version: 2  

  - **Save Audio Chucks**  
    - Type: Read/Write File  
    - Role: Writes each audio chunk to the `/tmp/audio <index>.mp3` path on the local file system.  
    - Config: Write operation, filename pattern `/tmp/audio {{$itemIndex}}.mp3`, data from binary property `audio {{$itemIndex}}`.  
    - Inputs: Binary audio files with renamed keys.  
    - Outputs: Passes data for next step.  
    - Edge Cases: File write permission errors, disk full, invalid binary data.  
    - Version: 1  

---

#### 2.3 Audio File Management and Merging

- **Overview:**  
  Generates the FFmpeg concat list file, merges audio chunks into a single file, and reads the merged file for upload.

- **Nodes Involved:**  
  - Generate `concat_list.txt`  
  - Save concat_list  
  - Join audio chucks and delete all files  
  - read final_merged  
  - Uploads Ebook (final upload)

- **Node Details:**  

  - **Generate `concat_list.txt`**  
    - Type: Code  
    - Role: Creates a text file listing all audio chunk file paths in FFmpeg concat format.  
    - Logic: Iterates incoming items, extracts `fileName` from each, concatenates lines like `file '/tmp/audio X.mp3'`. Then encodes the list as Base64 binary to write as a text file.  
    - Inputs: Items containing file paths from Save Audio Chucks node.  
    - Outputs: Single item with binary `concat_list.txt` file ready for saving.  
    - Edge Cases: Missing file paths, path inconsistencies.  
    - Version: 2  

  - **Save concat_list**  
    - Type: Read/Write File  
    - Role: Writes the binary `concat_list.txt` file to `/tmp/concat_list.txt`.  
    - Config: Write operation, fixed filename `/tmp/concat_list.txt`.  
    - Inputs: Binary concat list file.  
    - Outputs: Passes data for merging.  
    - Edge Cases: File write permission errors.  
    - Version: 1  

  - **Join audio chucks and delete all files**  
    - Type: Execute Command  
    - Role: Executes FFmpeg command to concatenate all audio chunks into `/tmp/final_merged.mp3`.  
    - Command:  
      ```
      ffmpeg -y -f concat -safe 0 -i /tmp/concat_list.txt -c copy /tmp/final_merged.mp3
      ```  
    - Inputs: concat_list.txt file saved on disk.  
    - Outputs: None directly, but creates merged MP3 on disk.  
    - Requirements: Must run on self-hosted n8n with FFmpeg installed; not supported on n8n cloud.  
    - Edge Cases: Missing FFmpeg, invalid concat list, file locking, disk space.  
    - Version: 1  

  - **read final_merged**  
    - Type: Read/Write File  
    - Role: Reads the merged MP3 file `/tmp/final_merged.mp3` into binary data for upload.  
    - Config: File selector `/tmp/final_merged.mp3`.  
    - Inputs: Triggered after FFmpeg execution.  
    - Outputs: Binary MP3 data.  
    - Edge Cases: File missing, read permission denied.  
    - Version: 1  

  - **Uploads Ebook**  
    - Type: Google Drive  
    - Role: Uploads the final merged audiobook MP3 to a specified folder on Google Drive.  
    - Config:  
      - Filename: `audiobook.mp3`  
      - Drive: "My Drive"  
      - Folder ID: `11eIODBtLZwiUjUVJvK97_Z42uPmEnMMu` (Audiobook folder)  
      - Credentials: OAuth2 Google Drive API (credential ID and name referenced)  
    - Inputs: Binary MP3 data.  
    - Outputs: Google Drive upload metadata.  
    - Edge Cases: Auth failures, quota exceeded, network errors.  
    - Version: 3  

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                   | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                       |
|-------------------------------|------------------------|--------------------------------------------------|-------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| FORM                          | Form Trigger           | Receives PDF upload                              | (external)                    | EXTRACT TEXT                     |                                                                                                 |
| EXTRACT TEXT                  | Extract From File      | Extracts text from uploaded PDF                   | FORM                          | SPLITS THE TEXT ACCORGING TO RULES |                                                                                                 |
| SPLITS THE TEXT ACCORGING TO RULES | Code                   | Cleans and splits extracted text into chunks    | EXTRACT TEXT                  | Loop Over Text chunks (5) at a time | Sticky Note1: Breaks large text into paragraphs and translates in batches of 5 paragraphs each   |
| Loop Over Text chunks (5) at a time | Split In Batches       | Processes text chunks in batches of 5            | SPLITS THE TEXT ACCORGING TO RULES, WAITS FOR 5 SECONDS | CONVERTS URL TO AUDIO FILES, MINIMAX TTS   |                                                                                                 |
| MINIMAX TTS                   | HTTP Request           | Converts text chunks to audio URLs using MiniMax | Loop Over Text chunks (5) at a time | WAITS FOR 5 SECONDS              |                                                                                                 |
| WAITS FOR 5 SECONDS           | Wait                   | Adds delay between batches                         | MINIMAX TTS                   | Loop Over Text chunks (5) at a time |                                                                                                 |
| CONVERTS URL TO AUDIO FILES   | HTTP Request           | Downloads audio files from URLs                    | Loop Over Text chunks (5) at a time | GIVES INDEXES TO AUDIO FILES      |                                                                                                 |
| GIVES INDEXES TO AUDIO FILES  | Code                   | Renames audio files with indexed keys             | CONVERTS URL TO AUDIO FILES    | Save Audio Chucks                |                                                                                                 |
| Save Audio Chucks             | Read/Write File        | Saves audio chunks to local disk                   | GIVES INDEXES TO AUDIO FILES   | Generate `concat_list.txt`       |                                                                                                 |
| Generate `concat_list.txt`    | Code                   | Creates FFmpeg concat list file                    | Save Audio Chucks             | Save concat_list                 | Sticky Note2: Uses FFmpeg to combine audio files; only works on self-hosted n8n with FFmpeg      |
| Save concat_list              | Read/Write File        | Writes concat list file to disk                     | Generate `concat_list.txt`     | Join audio chucks and delete all files |                                                                                                 |
| Join audio chucks and delete all files | Execute Command        | Merges audio chunks into one MP3 with FFmpeg      | Save concat_list              | read final_merged               |                                                                                                 |
| read final_merged             | Read/Write File        | Reads final merged MP3 file                        | Join audio chucks and delete all files | Uploads Ebook                   |                                                                                                 |
| Uploads Ebook                 | Google Drive           | Uploads final audiobook to Google Drive           | read final_merged             | None                          | Sticky Note3: Uploads the ebook to Drive. [Result](https://drive.google.com/file/d/12aVR2p-ZQ2DyqXCUgJPouzy-acoAB7WO/view?usp=sharing) |
| Sticky Note                   | Sticky Note            | Describes Ebook Extraction Module                  |                                |                                  | Sticky Note: Sample PDF link and ebook image                                                    |
| Sticky Note1                  | Sticky Note            | Describes Ebook to Audiobook Conversion Module     |                                |                                  | Contains example JSON of paragraph batching                                                    |
| Sticky Note2                  | Sticky Note            | Describes Audio Merging Module using FFmpeg        |                                |                                  | Contains FFmpeg command snippet                                                                |
| Sticky Note3                  | Sticky Note            | Describes Uploads Ebook node                        |                                |                                  | Link to resulting output on Google Drive                                                       |
| Sticky Note4                  | Sticky Note            | YouTube Demo Video                                 |                                |                                  | [Watch the YouTube Demo Video](https://img.youtube.com/vi/xKqkjXIPZoM/0.jpg)                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create FORM node**  
   - Type: Form Trigger  
   - Configure:  
     - Form Title: "Ebook to Audiobook"  
     - Form Fields: Add one file upload field  
       - Label: "UPLOAD"  
       - Multiple files: No  
       - Required: Yes  
     - Description: "Upload your Ebook here"  
   - No credentials needed.

2. **Connect FORM to EXTRACT TEXT**  
   - Type: Extract From File  
   - Configure:  
     - Operation: PDF  
     - Binary Property Name: "UPLOAD" (from FORM node)  
   - Output: Extracted raw text JSON.

3. **Connect EXTRACT TEXT to SPLITS THE TEXT ACCORGING TO RULES**  
   - Type: Code  
   - Paste JavaScript to clean and split text into chunks max 500 characters, preserving sentences:  
     ```js
     const rawText = $input.first().json.text || "";
     function cleanText(str) {
         return str
             .replace(/\\n/g, " ")
             .replace(/\\/g, "\\\\")
             .replace(/"/g, '\\"')
             .replace(/(\r\n|\r|\n)+/g, " ")
             .replace(/\s+/g, " ")
             .trim();
     }
     const text = cleanText(rawText);
     const sentences = text.split(/(?<=[.!?])\s+/).filter(Boolean);
     const maxChars = 500;
     let parts = [];
     let chunk = "";
     for (let sentence of sentences) {
         if (sentence.length > maxChars) {
             let start = 0;
             while (start < sentence.length) {
                 const sub = sentence.slice(start, start + maxChars);
                 if (chunk) parts.push(chunk);
                 parts.push(sub);
                 chunk = "";
                 start += maxChars;
             }
             continue;
         }
         const space = chunk ? " " : "";
         if ((chunk + space + sentence).length > maxChars) {
             if (chunk) parts.push(chunk);
             chunk = sentence;
         } else {
             chunk += space + sentence;
         }
     }
     if (chunk) parts.push(chunk);
     return parts.map((p, i) => ({ json: { order: i + 1, text: p } }));
     ```
   - Output: JSON array of text chunks.

4. **Connect SPLITS THE TEXT ACCORGING TO RULES to Loop Over Text chunks (5) at a time**  
   - Type: Split In Batches  
   - Configure batch size: 5.

5. **From Loop Over Text chunks (5) at a time, create two parallel outputs:**  
   - Connect first output to MINIMAX TTS node.  
   - Connect second output to CONVERTS URL TO AUDIO FILES node.

6. **Configure MINIMAX TTS node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/models/minimax/speech-02-hd/predictions`  
   - Authentication: HTTP Bearer Token (set your MiniMax API token)  
   - Headers: Add header "Prefer" with value "wait"  
   - Body: JSON with parameters:  
     ```json
     {
       "input": {
         "text": "{{ $json.text }}",
         "pitch": 0,
         "speed": 1,
         "volume": 1,
         "bitrate": 128000,
         "channel": "mono",
         "emotion": "happy",
         "voice_id": "Friendly_Person",
         "sample_rate": 32000,
         "language_boost": "English",
         "english_normalization": true
       }
     }
     ```
   - Send as JSON body.

7. **Connect MINIMAX TTS to WAITS FOR 5 SECONDS**  
   - Type: Wait  
   - Duration: 5 seconds.

8. **Connect WAITS FOR 5 SECONDS back to Loop Over Text chunks (5) at a time**  
   - This creates a controlled loop ensuring a 5-second pause between batches.

9. **Configure CONVERTS URL TO AUDIO FILES node**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{ $json.output }}` (dynamic from MiniMax TTS output)  
   - No authentication.

10. **Connect CONVERTS URL TO AUDIO FILES to GIVES INDEXES TO AUDIO FILES**  
    - Type: Code  
    - JavaScript code to rename binary properties and file names to `audio <index>.mp3`:  
      ```js
      return items.map((item, index) => {
        const newItem = { json: {}, binary: {} };
        newItem.json = { ...item.json };
        for (let key in item.binary) {
          const newKey = `audio ${index}`;
          newItem.binary[newKey] = { ...item.binary[key] };
          newItem.binary[newKey].fileName = `${newKey}.mp3`;
        }
        return newItem;
      });
      ```

11. **Connect GIVES INDEXES TO AUDIO FILES to Save Audio Chucks**  
    - Type: Read/Write File  
    - Operation: Write  
    - Filename: `/tmp/audio {{$itemIndex}}.mp3`  
    - Data Property: `audio {{$itemIndex}}` (binary).

12. **Connect Save Audio Chucks to Generate `concat_list.txt`**  
    - Type: Code  
    - JavaScript to create FFmpeg concat list and convert to Base64 for writing:  
      ```js
      const items = $input.all();
      let concatListText = '';
      items.forEach((item) => {
        const filePath = item.json.fileName;
        if (filePath) concatListText += `file '${filePath}'\n`;
      });
      const buffer = Buffer.from(concatListText, 'utf-8');
      const base64Data = buffer.toString('base64');
      return [{
        json: {},
        binary: {
          data: {
            data: base64Data,
            mimeType: 'text/plain',
            fileName: 'concat_list.txt'
          }
        }
      }];
      ```

13. **Connect Generate `concat_list.txt` to Save concat_list**  
    - Type: Read/Write File  
    - Operation: Write  
    - Filename: `/tmp/concat_list.txt`.

14. **Connect Save concat_list to Join audio chucks and delete all files**  
    - Type: Execute Command  
    - Command:  
      ```
      ffmpeg -y -f concat -safe 0 -i /tmp/concat_list.txt -c copy /tmp/final_merged.mp3
      ```
    - Requirements: Self-hosted n8n with FFmpeg installed.

15. **Connect Join audio chucks and delete all files to read final_merged**  
    - Type: Read/Write File  
    - Operation: Read  
    - File Selector: `/tmp/final_merged.mp3`.

16. **Connect read final_merged to Uploads Ebook**  
    - Type: Google Drive  
    - Configure:  
      - Filename: `audiobook.mp3`  
      - Drive: "My Drive"  
      - Folder ID: Your target folder ID  
      - Credentials: Google Drive OAuth2 (setup required).

17. **Add Sticky Notes** describing each major module and provide links or instructions as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Sample PDF file for testing: [Little Red Riding Hood.pdf](https://www.laburnumps.vic.edu.au/uploaded_files/media/little_red_riding_hood.pdf) | Ebook Extraction Module sticky note                                                                                     |
| FFmpeg must be installed on self-hosted n8n for audio merging; not supported on n8n cloud.                                                 | Audio Merging Module sticky note                                                                                         |
| Final audiobook upload example: [Google Drive Link](https://drive.google.com/file/d/12aVR2p-ZQ2DyqXCUgJPouzy-acoAB7WO/view?usp=sharing)    | Uploads Ebook sticky note                                                                                                |
| Watch the demo video on YouTube: [Demo Video Thumbnail](https://img.youtube.com/vi/xKqkjXIPZoM/0.jpg)                                      | Sticky Note4                                                                                                            |
| MiniMax API requires Bearer token authentication; ensure token is valid and has sufficient quota.                                          | MINIMAX TTS node authentication note                                                                                    |
| Text chunking respects sentence boundaries and limits chunks to 500 characters to optimize TTS quality and API constraints.                | SPLITS THE TEXT ACCORGING TO RULES node code description                                                                |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively using the n8n automation tool and comply with all applicable content policies. No illegal, offensive, or protected elements are included. All processed data are legal and publicly accessible.

---