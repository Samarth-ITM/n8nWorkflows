Convert RSS Feeds into a Podcast with Google Gemini, Kokoro TTS, and FFmpeg

https://n8nworkflows.xyz/workflows/convert-rss-feeds-into-a-podcast-with-google-gemini--kokoro-tts--and-ffmpeg-6945


# Convert RSS Feeds into a Podcast with Google Gemini, Kokoro TTS, and FFmpeg

### 1. Workflow Overview

This workflow, titled **Daily Sports Digest**, automates the transformation of daily sports news from multiple RSS feeds into a polished podcast episode. It is designed to run once per day, fetching fresh content, generating both a textual digest and a scripted two-person podcast dialogue using Google Gemini AI models, converting the dialogue into audio chunks via Kokoro TTS API, and finally merging these chunks into a single MP3 file with FFmpeg. The finished podcast is sent directly to a Telegram chat.

The workflow logic can be grouped into five main functional blocks:

- **1.1 Fetch & Filter Daily News:** Daily trigger initiates RSS feed reading, merging, cleaning, and filtering articles published only in the last 24 hours.
- **1.2 Generate AI Content (Digest & Script):** Two parallel Google Gemini AI agents create (a) a clean text digest summary sent to Telegram, and (b) a conversational podcast script between two AI voices.
- **1.3 Split Script & Generate Audio Chunks:** The podcast script is split by speaker lines, cleaned, routed, and sent to two Kokoro TTS API endpoints to generate individual MP3 audio chunks.
- **1.4 Save, Prepare & Merge Audio:** Audio chunks are saved locally; a concat list file is generated for FFmpeg, which merges the chunks into a final seamless MP3. Temporary files are cleaned up.
- **1.5 Read Merged Audio & Send Final Podcast:** The merged podcast file is read and sent as audio to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Fetch & Filter Daily News

**Overview:**  
This block triggers daily, reads multiple RSS feeds for sports news, cleans and standardizes fields, merges feeds, and filters articles to only those published in the previous calendar day.

**Nodes Involved:**  
- Daily Trigger  
- Fetch RSS 1: Folha de SP  
- Fetch RSS 2: GE  
- Clean Up Fields (for each RSS feed)  
- Merge News Sources  
- Filter for Last 24hrs' News  
- Prepare Data for AI

**Node Details:**  

- **Daily Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow daily at 8:00 AM server time.  
  - Configuration: Interval trigger at hour 8.  
  - Edge Cases: Trigger time zone depends on server; ensure it matches target audience time zone.  

- **Fetch RSS 1: Folha de SP / Fetch RSS 2: GE**  
  - Type: RSS Feed Read  
  - Role: Reads RSS feeds from Brazilian sports news sources.  
  - Configuration: URLs to respective RSS feed endpoints.  
  - Edge Cases: RSS feed unavailability or format changes could cause empty or malformed data.  

- **Clean Up Fields / Clean Up Fields1**  
  - Type: Set  
  - Role: Cleans and normalizes article fields: removes bracketed tags from titles, extracts publication date, cleans links, and converts ISO date to timestamp.  
  - Key Expressions: Uses regex replacements on titles and links, converts isoDate to Unix timestamp (ms).  
  - Edge Cases: Unexpected RSS feed formats could break regex or date parsing.  

- **Merge News Sources**  
  - Type: Merge  
  - Role: Combines articles from both RSS feeds into a single data stream.  
  - Edge Cases: Ensures no data loss; if feeds are empty, output will be empty.  

- **Filter for Last 24hrs' News**  
  - Type: Filter  
  - Role: Keeps only articles with isoDate timestamps after start of yesterday and before start of today, ensuring only previous calendar day’s news is processed.  
  - Key Expressions:  
    - `isoDate > (midnight today - 24 hrs)`  
    - `isoDate < midnight today`  
  - Edge Cases: Timezone handling critical to match local date boundaries; inaccurate server time can cause missed articles.  

- **Prepare Data for AI**  
  - Type: Code  
  - Role: Formats filtered articles into a single text block with markdown-style separators, extracting title, content, and link; generates formatted date string for the digest title.  
  - Key Expressions: Uses JavaScript date formatting with timezone `'America/Sao_Paulo'`.  
  - Edge Cases: Null or missing fields could cause formatting issues.

---

#### 2.2 Generate AI Content (Digest & Script)

**Overview:**  
Parallel generation of (a) a concise, English-language sports news digest and (b) a scripted podcast dialogue between two AI personas, both using Google Gemini AI models. Also creates a temporary directory for storing audio files.

**Nodes Involved:**  
- Create Temp Directory  
- Google Gemini Chat Model (for Digest)  
- Generate Text Digest  
- Send Text Digest  
- Google Gemini Chat Model1 (for Podcast Script)  
- Generate Podcast Script

**Node Details:**  

- **Create Temp Directory**  
  - Type: Execute Command  
  - Role: Ensures `/tmp/dailydigest` directory exists for saving audio files.  
  - Command: `mkdir -p /tmp/dailydigest`  
  - Edge Cases: Permission issues on server could cause failure.  

- **Google Gemini Chat Model (Digest) / Google Gemini Chat Model1 (Script)**  
  - Type: Langchain Google Gemini Chat Model  
  - Role: AI language model nodes configured to use `"models/gemini-2.5-pro"`.  
  - Credentials: Uses Google Palm API credentials.  
  - Edge Cases: API quota limits, authentication errors, or network issues.  

- **Generate Text Digest**  
  - Type: Langchain Agent  
  - Role: Takes formatted news and date, generates a witty, engaging sports digest in English using MarkdownV2 formatting.  
  - Key Expressions: Prompt instructs AI on style, output format, and content.  
  - Input: Output from Prepare Data for AI node.  
  - Output: Markdown formatted digest text.  
  - Edge Cases: AI generation failure or incomplete output.  

- **Send Text Digest**  
  - Type: Telegram  
  - Role: Sends the generated digest text to a specified Telegram chat with Markdown formatting enabled.  
  - Credentials: Telegram API credentials required.  
  - Edge Cases: Network errors, invalid chat ID, Telegram rate limits.  

- **Generate Podcast Script**  
  - Type: Langchain Agent  
  - Role: Transforms the news digest text into a two-voice conversational podcast script with detailed character instructions for voice1 and voice2.  
  - Key Expressions: Long prompt emphasizing natural dialogue, informal tone, and specific structure.  
  - Output: Single large dialogue text block.  
  - Edge Cases: AI response length limits, malformed output.  

---

#### 2.3 Split Script & Generate Audio Chunks

**Overview:**  
Processes the AI-generated podcast script by splitting it into speaker-labeled segments, cleaning text, routing to proper TTS voice nodes, and generating MP3 audio chunks via Kokoro TTS API.

**Nodes Involved:**  
- Split Script by Speaker  
- Loop Through Segments  
- Clean Dialogue Segment  
- Route to Correct Voice (If)  
- Prepare Text for TTS (voice1)  
- Prepare Text for TTS1 (voice2)  
- Generate Audio (Voice 1)  
- Generate Audio (Voice 2)

**Node Details:**  

- **Split Script by Speaker**  
  - Type: Code  
  - Role: Uses regex to split script on occurrences of "voice1:" or "voice2:", returning each segment preserving speaker label.  
  - Output: Multiple items, one per dialogue segment.  
  - Edge Cases: Unexpected script formats may cause splits to fail or produce empty segments.  

- **Loop Through Segments**  
  - Type: SplitInBatches  
  - Role: Loops through dialogue segments for sequential processing.  
  - Edge Cases: Large scripts may require batch size tuning for performance.  

- **Clean Dialogue Segment**  
  - Type: Code  
  - Role: Removes quotation marks and newlines from segment text to sanitize input for TTS.  
  - Key Expressions: Regex replacements for `"`, `“`, `”`, and newlines.  
  - Edge Cases: Missing input text throws error; malformed segments cause failure.  

- **Route to Correct Voice (If)**  
  - Type: If  
  - Role: Checks cleaned text for presence of "voice1:" tag to determine routing.  
  - Expression: Contains check for "voice1:".  
  - Output: Routes to Prepare Text for TTS (voice1) if true, else to Prepare Text for TTS1 (voice2).  
  - Edge Cases: Tags missing or mistyped cause routing errors.  

- **Prepare Text for TTS / Prepare Text for TTS1**  
  - Type: Code  
  - Role: Removes speaker tags ("voice1:" or "voice2:") from cleaned text, trims whitespace, and confirms tag removal with debug logs.  
  - Edge Cases: Unexpected text formats may leave tags unremoved; logs help debug.  

- **Generate Audio (Voice 1) / Generate Audio (Voice 2)**  
  - Type: HTTP Request  
  - Role: Calls Kokoro TTS API to generate MP3 audio from cleaned dialogue text for each AI voice.  
  - Configuration:  
    - URL: `https://tts-kokoro.mfxikq.easypanel.host/api/v1/audio/speech`  
    - Method: POST  
    - Body Params: model, voice (distinct per node), speed=1, response_format=mp3, input text  
  - Retry on Fail enabled for reliability.  
  - Edge Cases: API key missing or invalid, network timeout, API quota exceeded.  
  - Note: Requires user to insert valid Kokoro API key in headers.  
  - Voices can be changed by modifying the `voice` parameter.

---

#### 2.4 Save, Prepare & Merge Audio

**Overview:**  
Saves each generated MP3 chunk to disk, creates an FFmpeg concat list file, merges all chunks into a single MP3, and cleans temporary files.

**Nodes Involved:**  
- Save Audio Chunk to Disk  
- Generate FFmpeg Concat List  
- Save Concat List to Disk  
- Merge Audio & Clean Up

**Node Details:**  

- **Save Audio Chunk to Disk**  
  - Type: Read/Write File  
  - Role: Saves each binary MP3 audio chunk to `/tmp/dailydigest/dailydigest_{{index}}.mp3`.  
  - Edge Cases: File system permissions or disk space issues.  

- **Generate FFmpeg Concat List**  
  - Type: Code  
  - Role: Creates a text file with lines formatted for FFmpeg concat: `file '/path/to/file'` for each saved chunk.  
  - Output: Binary base64-encoded text file named `concat_list.txt`.  
  - Edge Cases: Missing file paths cause incomplete concat list.  

- **Save Concat List to Disk**  
  - Type: Read/Write File  
  - Role: Writes `concat_list.txt` to `/tmp/dailydigest/`.  
  - Edge Cases: File system issues.  

- **Merge Audio & Clean Up**  
  - Type: Execute Command  
  - Role: Runs FFmpeg command to merge audio files losslessly using concat list, outputs `/tmp/dailydigest/final_merged.mp3`. Then deletes all temporary files except the final MP3.  
  - Command:  
    ```
    ffmpeg -y -f concat -safe 0 -i /tmp/dailydigest/concat_list.txt -c copy /tmp/dailydigest/final_merged.mp3
    find /tmp/dailydigest/ -type f ! -name "final_merged.mp3" -delete
    ```  
  - Edge Cases: FFmpeg not installed or accessible, command failure, file permission issues.

---

#### 2.5 Read Merged Audio & Send Final Podcast

**Overview:**  
Reads the final combined MP3 file from disk and sends it as an audio message to a Telegram chat with a dynamically generated filename.

**Nodes Involved:**  
- Read Final Merged MP3  
- Send Podcast to Telegram

**Node Details:**  

- **Read Final Merged MP3**  
  - Type: Read/Write File  
  - Role: Reads `/tmp/dailydigest/final_merged.mp3` into binary format for upload.  
  - Edge Cases: File missing or permission denied.  

- **Send Podcast to Telegram**  
  - Type: Telegram  
  - Role: Sends the MP3 audio file as a Telegram audio message to a configured chat ID.  
  - Parameters:  
    - `chatId`: User-specified Telegram chat ID.  
    - `fileName`: Dynamic filename like `Daily Digest - dd/MM/yyyy.mp3` using Sao Paulo timezone.  
  - Credentials: Telegram API account.  
  - Edge Cases: Invalid chat ID, Telegram API failures, file size limits.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                          | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
|---------------------------|----------------------------------------|----------------------------------------|------------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger             | Schedule Trigger                       | Triggers workflow daily at 8 AM         |                                    | Fetch RSS 1: Folha de SP, Fetch RSS 2: GE | ## 📰 Step 1: Fetch & Filter Daily News: This section acts as the data pipeline. It triggers once a day, fetches the latest articles from multiple RSS feeds, merges them, and filters the list to keep only articles published on the previous calendar day. This provides a clean, relevant dataset for the AI.                                                                                                                                                                                                                                                                                                                                                                             |
| Fetch RSS 1: Folha de SP  | RSS Feed Read                         | Reads sports news RSS feed               | Daily Trigger                     | Clean Up Fields                   | See above note for Step 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Fetch RSS 2: GE           | RSS Feed Read                         | Reads sports news RSS feed               | Daily Trigger                     | Clean Up Fields1                  | See above note for Step 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Clean Up Fields           | Set                                  | Cleans and normalizes RSS feed fields   | Fetch RSS 1                      | Merge News Sources               | See above note for Step 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Clean Up Fields1          | Set                                  | Cleans and normalizes RSS feed fields   | Fetch RSS 2                      | Merge News Sources               | See above note for Step 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Merge News Sources        | Merge                                | Merges multiple RSS feeds into one list | Clean Up Fields, Clean Up Fields1 | Filter for Last 24hrs' News      | See above note for Step 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Filter for Last 24hrs' News | Filter                             | Filters articles published yesterday    | Merge News Sources              | Prepare Data for AI              | See above note for Step 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Prepare Data for AI       | Code                                  | Formats articles and date for AI input  | Filter for Last 24hrs' News       | Create Temp Directory, Generate Text Digest | See above note for Step 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Create Temp Directory     | Execute Command                       | Creates directory for audio storage     | Prepare Data for AI               | Generate Podcast Script           | ## ✍️ Step 2: Generate AI Content (Digest & Script): Ensures folder /tmp/dailydigest exists for audio chunks                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Google Gemini Chat Model  | Langchain AI Model                   | AI model generating text digest         | Prepare Data for AI               | Generate Text Digest              | Part of AI content generation block                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Generate Text Digest      | Langchain Agent                      | Generates English sports digest summary | Google Gemini Chat Model         | Send Text Digest                 | See above note for Step 2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Send Text Digest          | Telegram                            | Sends digest text to Telegram chat      | Generate Text Digest              |                                 | See above note for Step 2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Google Gemini Chat Model1 | Langchain AI Model                   | AI model generating podcast script      | Create Temp Directory            | Generate Podcast Script          | Part of AI content generation block                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Generate Podcast Script   | Langchain Agent                      | Creates two-voice conversational script | Google Gemini Chat Model1        | Split Script by Speaker          | See above note for Step 2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Split Script by Speaker   | Code                                 | Splits script into voice1/voice2 segments | Generate Podcast Script          | Loop Through Segments            | ## ✍️ Step 3: Split Script & Generate Audio Chunks: Core audio generation engine                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Loop Through Segments     | SplitInBatches                      | Loops through each dialogue segment     | Split Script by Speaker          | Save Audio Chunk to Disk, Clean Dialogue Segment | See above note for Step 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Clean Dialogue Segment    | Code                                 | Cleans quotes and line breaks from text | Loop Through Segments            | Route to Correct Voice           | See above note for Step 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Route to Correct Voice    | If                                   | Routes segment to voice1 or voice2 TTS  | Clean Dialogue Segment           | Prepare Text for TTS, Prepare Text for TTS1 | See above note for Step 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Prepare Text for TTS      | Code                                 | Removes "voice1:" prefix, trims text    | Route to Correct Voice           | Generate Audio (Voice 1)         | See above note for Step 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Prepare Text for TTS1     | Code                                 | Removes "voice2:" prefix, trims text    | Route to Correct Voice           | Generate Audio (Voice 2)         | See above note for Step 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Generate Audio (Voice 1)  | HTTP Request                        | Calls Kokoro TTS to generate MP3 for voice1 | Prepare Text for TTS            | Loop Through Segments            | See above note for Step 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Generate Audio (Voice 2)  | HTTP Request                        | Calls Kokoro TTS to generate MP3 for voice2 | Prepare Text for TTS1           | Loop Through Segments            | See above note for Step 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Save Audio Chunk to Disk  | Read/Write File                   | Saves MP3 chunk to disk                   | Loop Through Segments            | Generate FFmpeg Concat List      | ##  🎛️  Step 4: Save, Prepare, & Merge Audio: Manages audio files and merging                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Generate FFmpeg Concat List | Code                             | Creates concat list file for FFmpeg      | Save Audio Chunk to Disk         | Save Concat List to Disk         | See above note for Step 4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Save Concat List to Disk  | Read/Write File                   | Saves FFmpeg concat list text file       | Generate FFmpeg Concat List      | Merge Audio & Clean Up           | See above note for Step 4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Merge Audio & Clean Up    | Execute Command                   | Runs FFmpeg to merge audio and cleans temp files | Save Concat List to Disk       | Read Final Merged MP3            | See above note for Step 4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Read Final Merged MP3     | Read/Write File                   | Reads final merged MP3 into binary        | Merge Audio & Clean Up           | Send Podcast to Telegram         | ## 📤  Step 5: Read Merged Audio & Send Final Podcast: Final delivery to Telegram                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Send Podcast to Telegram  | Telegram                           | Sends final MP3 audio to Telegram chat   | Read Final Merged MP3            |                                 | See above note for Step 5                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note1              | Sticky Note                       | Overview and general explanation          |                                    |                                   | Contains full workflow purpose, how it works, customization options, and inspiration link: https://n8n.io/workflows/6523-convert-newsletters-into-ai-podcasts-with-gpt-4o-mini-and-elevenlabs/                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Sticky Note                | Sticky Note                      | Step 1 detailed explanation                |                                    |                                   | Explains Fetch & Filter Daily News logic and customization                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Sticky Note2              | Sticky Note                       | Step 2 detailed explanation                |                                    |                                   | Explains AI content generation, prompt details, directory setup, and customization                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Sticky Note3              | Sticky Note                       | Step 3 detailed explanation                |                                    |                                   | Explains splitting script, audio generation, TTS service setup, and customization                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Sticky Note4              | Sticky Note                       | Step 4 detailed explanation                |                                    |                                   | Explains saving chunks, generating concat list, merging audio with FFmpeg, and cleanup                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Sticky Note5              | Sticky Note                       | Step 5 detailed explanation                |                                    |                                   | Explains reading final MP3 and sending it via Telegram, including filename customization                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Daily Trigger**  
- Create a **Schedule Trigger** node named `Daily Trigger`.  
- Configure to trigger once daily at 8:00 AM (adjust time as needed).  

**Step 2: Fetch RSS Feeds**  
- Create two **RSS Feed Read** nodes:  
  - `Fetch RSS 1: Folha de SP` with URL `https://feeds.folha.uol.com.br/esporte/rss091.xml`  
  - `Fetch RSS 2: GE` with URL `https://ge.globo.com/rss/ge/`  
- Connect `Daily Trigger` to both RSS nodes in parallel.  

**Step 3: Clean and Normalize RSS Data**  
- For each RSS node, add a **Set** node (`Clean Up Fields` and `Clean Up Fields1`) to:  
  - Remove bracketed tags from titles (`title.replace(/\[PACK\].*/, "").replace(/\[.*?\]/g, "").trim()`)  
  - Copy `pubDate` as-is  
  - Clean links with regex replace  
  - Keep `content` field  
  - Convert `isoDate` string to timestamp (number) using `new Date($json.isoDate).getTime()`  
- Connect each RSS node to its corresponding clean node.  

**Step 4: Merge and Filter Articles**  
- Add a **Merge** node `Merge News Sources`, connect both clean nodes as inputs.  
- Add a **Filter** node `Filter for Last 24hrs' News`:  
  - Condition 1: `isoDate` > `(midnight today - 24h)`  
  - Condition 2: `isoDate` < `midnight today`  
- Connect merge to filter node.  

**Step 5: Prepare Data for AI**  
- Add a **Code** node `Prepare Data for AI` that:  
  - Aggregates all filtered news items into a markdown block with title, content, and link.  
  - Formats today’s date in `"MMMM d, yyyy"` format in Sao Paulo timezone.  
- Connect filter node to this code node.  

**Step 6: Create Temporary Directory**  
- Add an **Execute Command** node `Create Temp Directory` with command: `mkdir -p /tmp/dailydigest`  
- Connect `Prepare Data for AI` to this node.  

**Step 7: Generate Text Digest**  
- Add a **Langchain Agent** node `Generate Text Digest`:  
  - Use Google Gemini Chat Model with `"models/gemini-2.5-pro"`.  
  - Prompt: Provided detailed prompt that instructs creation of English sports digest in MarkdownV2.  
- Connect `Prepare Data for AI` to this node.  

**Step 8: Send Text Digest to Telegram**  
- Add a **Telegram** node `Send Text Digest`:  
  - Configure with Telegram API credentials.  
  - Set chat ID.  
  - Send message with digest output, enabling Markdown parsing.  
- Connect `Generate Text Digest` to this node.  

**Step 9: Generate Podcast Script**  
- Add another **Langchain Agent** node `Generate Podcast Script` using Google Gemini Chat Model1.  
- Use prompt instructing generation of a two-person conversational dialogue script in English with roles voice1 and voice2.  
- Connect `Create Temp Directory` to this node.  

**Step 10: Split Script by Speaker**  
- Add a **Code** node `Split Script by Speaker` that splits the script text at occurrences of "voice1:" or "voice2:" and outputs each segment as one item.  
- Connect `Generate Podcast Script` to this node.  

**Step 11: Loop Through Segments**  
- Add a **SplitInBatches** node `Loop Through Segments` to process segments one-by-one.  
- Connect `Split Script by Speaker` to this node.  

**Step 12: Clean Dialogue Segment**  
- Add a **Code** node `Clean Dialogue Segment` to:  
  - Remove quotes and newlines from the segment text.  
- Connect `Loop Through Segments` to this node.  

**Step 13: Route to Correct Voice**  
- Add an **If** node `Route to Correct Voice`:  
  - Condition: If cleaned text contains "voice1:" (case-sensitive).  
  - True output: to `Prepare Text for TTS` (voice1)  
  - False output: to `Prepare Text for TTS1` (voice2)  
- Connect `Clean Dialogue Segment` to this node.  

**Step 14: Prepare Text for TTS (voice1 and voice2)**  
- Add two **Code** nodes, `Prepare Text for TTS` and `Prepare Text for TTS1`, each removing their respective speaker tags ("voice1:" or "voice2:") and trimming whitespace.  
- Connect `Route to Correct Voice` outputs to these nodes respectively.  

**Step 15: Generate Audio with Kokoro TTS**  
- Add two **HTTP Request** nodes, `Generate Audio (Voice 1)` and `Generate Audio (Voice 2)` configured as:  
  - POST to `https://tts-kokoro.mfxikq.easypanel.host/api/v1/audio/speech`  
  - Body parameters: model=`model_q8f16`, voice=`am_liam` (voice1) or `af_heart` (voice2), speed=1, response_format=mp3, input text from respective code node.  
  - Add header `X-API-KEY` with your Kokoro API key.  
  - Enable retry on failure.  
- Connect `Prepare Text for TTS` and `Prepare Text for TTS1` to their respective HTTP nodes.  
- Connect back both HTTP nodes to `Loop Through Segments` to continue processing.  

**Step 16: Save Audio Chunks**  
- Add a **Read/Write File** node `Save Audio Chunk to Disk`:  
  - Operation: write  
  - Filename: `/tmp/dailydigest/dailydigest_{{$itemIndex}}.mp3`  
  - Save binary data from TTS nodes.  
- Connect `Loop Through Segments` to this node.  

**Step 17: Generate FFmpeg Concat List**  
- Add a **Code** node `Generate FFmpeg Concat List` that creates a text file with lines of the form:  
  `file '/tmp/dailydigest/dailydigest_0.mp3'` for each saved chunk.  
- Encode as base64 binary to write as text file.  
- Connect `Save Audio Chunk to Disk` to this node.  

**Step 18: Save Concat List to Disk**  
- Add a **Read/Write File** node `Save Concat List to Disk`:  
  - Operation: write  
  - Filename: `/tmp/dailydigest/concat_list.txt`  
- Connect `Generate FFmpeg Concat List` to this node.  

**Step 19: Merge Audio and Cleanup**  
- Add an **Execute Command** node `Merge Audio & Clean Up` with command:  
  ```
  ffmpeg -y -f concat -safe 0 -i /tmp/dailydigest/concat_list.txt -c copy /tmp/dailydigest/final_merged.mp3
  find /tmp/dailydigest/ -type f ! -name "final_merged.mp3" -delete
  ```  
- Connect `Save Concat List to Disk` to this node.  

**Step 20: Read Final MP3**  
- Add a **Read/Write File** node `Read Final Merged MP3`:  
  - Operation: read  
  - File Selector: `/tmp/dailydigest/final_merged.mp3`  
- Connect `Merge Audio & Clean Up` to this node.  

**Step 21: Send Podcast to Telegram**  
- Add a **Telegram** node `Send Podcast to Telegram`:  
  - Operation: sendAudio  
  - Binary data: enabled  
  - Chat ID: your Telegram chat ID  
  - Filename: dynamic expression `Daily Digest - {{ $now.setZone('America/Sao_Paulo').toFormat('dd/LL/yyyy') }}.mp3`  
- Connect `Read Final Merged MP3` to this node.  
- Configure Telegram API credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                          |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Workflow inspired by an n8n example converting newsletters into AI podcasts using GPT-4o-mini and ElevenLabs TTS. | https://n8n.io/workflows/6523-convert-newsletters-into-ai-podcasts-with-gpt-4o-mini-and-elevenlabs/                         |
| The workflow uses timezone 'America/Sao_Paulo' consistently for date formatting and filtering.             | Important for correct date filtering and naming conventions.                                                             |
| Kokoro TTS API requires user to obtain and insert their own API key in the HTTP Request nodes' headers.   | Refer to Kokoro API documentation for key generation and available voices.                                                |
| Telegram nodes require a valid bot token and chat ID with necessary permissions.                           | Ensure Telegram bot is added to chat and has permission to send messages and audio.                                       |
| FFmpeg command requires FFmpeg to be installed and accessible on the server or container running n8n.     | Installation of FFmpeg is a prerequisite for audio merging.                                                               |

---

This completes the detailed reference document for the **Daily Sports Digest** n8n workflow, enabling advanced users and AI agents to thoroughly understand, reproduce, and adapt the process end-to-end.