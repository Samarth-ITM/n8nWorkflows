Convert YouTube Videos to Audio Summaries with Decodo, Gemini 2.5 & Telegram

https://n8nworkflows.xyz/workflows/convert-youtube-videos-to-audio-summaries-with-decodo--gemini-2-5---telegram-10981


# Convert YouTube Videos to Audio Summaries with Decodo, Gemini 2.5 & Telegram

### 1. Workflow Overview

This workflow automates the conversion of YouTube video URLs into concise audio summaries delivered via Telegram. It targets users who want quick, spoken overviews of YouTube content without watching the full videos. The workflow is structured into these logical blocks:

- **1.1 Input Reception & Validation:** Receives a YouTube URL via Telegram, verifies its format, and acknowledges processing.
- **1.2 Transcript Extraction:** Uses Decodo API to fetch the video’s transcript, with error handling for unavailable captions.
- **1.3 Transcript Formatting:** Converts the raw transcript JSON into clean text suitable for AI summarization.
- **1.4 AI Summarization:** Uses Google Gemini 2.5 language model to analyze and summarize the transcript into a structured JSON containing genre, title, summary, and a script for text-to-speech.
- **1.5 Audio Generation:** Converts the AI-generated script into an MP3 audio file using OpenAI’s Text-to-Speech.
- **1.6 Delivery:** Sends the audio summary and a formatted text summary back to the user on Telegram.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Validation

**Overview:**  
This block listens for Telegram messages, validates that the message contains a proper YouTube URL, and sends a preliminary “working on it” message to the user, or an error message if validation fails.

**Nodes Involved:**  
- Telegram Trigger  
- Config  
- Check: Is Valid URL?  
- Send Working on it Message  
- Send Error Message  

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point; triggers workflow on receiving a Telegram message from a specific user ID.  
  - Config: Listens to "message" updates; restricts input to user ID `66336158`.  
  - Input: Telegram messages  
  - Output: Passes message JSON downstream.  
  - Edge Cases: Messages from unauthorized users are ignored.  
  - Notes: Requires a Telegram API credential.

- **Config**  
  - Type: Set Node  
  - Role: Defines configuration parameter `output_language` with default "English".  
  - Input: From Telegram Trigger  
  - Output: Adds config JSON for downstream use.  
  - Edge Cases: Misconfiguration could cause language mismatch in AI prompt.

- **Check: Is Valid URL?**  
  - Type: If Node  
  - Role: Validates message text with regex to confirm it is a YouTube URL (standard or short link).  
  - Condition: Regex checks for typical YouTube URL patterns containing 11-character video ID.  
  - Input: Telegram message text  
  - Output:  
    - True branch: Valid URL → continue workflow  
    - False branch: Invalid URL → send error message.  
  - Edge Cases:  
    - URLs with extra parameters still accepted.  
    - Non-YouTube URLs or malformed URLs rejected.  
  - Version: Uses strict type validation in n8n v2.2+.

- **Send Working on it Message**  
  - Type: Telegram Node  
  - Role: Sends an immediate feedback message indicating transcript fetching and audio generation is in progress.  
  - Config: Text message with emoji; chat ID dynamically set from Telegram Trigger’s chat context.  
  - Input: Valid URL branch of validation node  
  - Output: Passes control to transcript fetching.  
  - Edge Cases: Telegram API errors (rate limits, network issues).

- **Send Error Message**  
  - Type: Telegram Node  
  - Role: Sends an error message to user when URL validation fails.  
  - Config: Text warns user about invalid URL format and instructs on correct URL formats.  
  - Input: Invalid URL branch from validation  
  - Edge Cases: Telegram API errors.

---

#### 2.2 Transcript Extraction

**Overview:**  
Fetches the transcript JSON from the specified YouTube video ID using Decodo API and checks for transcript availability errors.

**Nodes Involved:**  
- Decodo (Fetch Transcript)  
- Check for Transcript Error  
- Send Transcript Failed Message  

**Node Details:**  

- **Decodo (Fetch Transcript)**  
  - Type: Decodo Node  
  - Role: Retrieves YouTube transcript via Decodo API using extracted video ID.  
  - Config: Extracts video ID from URL using regex; requests transcript in English.  
  - Credentials: Requires Decodo API key.  
  - Output: JSON with transcript content or error message.  
  - Edge Cases: No captions available, music videos, API limit reached, invalid video ID.

- **Check for Transcript Error**  
  - Type: If Node  
  - Role: Detects if Decodo response contains an error field indicating transcript extraction failure.  
  - Condition: Checks if `$json.results[0].content.error` exists and is true.  
  - Output:  
    - True branch: Send transcript failure message  
    - False branch: Continue to format transcript.  
  - Edge Cases: Unexpected JSON structure or missing fields.

- **Send Transcript Failed Message**  
  - Type: Telegram Node  
  - Role: Notifies user that transcript extraction failed and provides error message from Decodo.  
  - Config: Markdown formatted message with transcript failure reason.  
  - Input: Error branch from Check for Transcript Error.  
  - Edge Cases: Telegram API failures.

---

#### 2.3 Transcript Formatting

**Overview:**  
Transforms the nested transcript JSON into a clean, concatenated text string suitable for AI summarization.

**Nodes Involved:**  
- Code: Format Transcript  

**Node Details:**  

- **Code: Format Transcript**  
  - Type: Code Node (JavaScript)  
  - Role: Parses the Decodo transcript JSON array, extracts text segments, and joins them into a single string.  
  - Logic:  
    - Checks if transcript content is an array (valid)  
    - Maps each segment’s text snippet and concatenates with spaces  
    - Returns a JSON object with `transcript` string  
  - Input: Transcript JSON from Decodo  
  - Output: Clean text transcript for AI input  
  - Edge Cases: Empty or invalid transcript content returns empty string.

---

#### 2.4 AI Summarization

**Overview:**  
Uses Google Gemini 2.5 to analyze the transcript and generate structured JSON output with genre, title, summary, and a text-to-speech script in the configured output language.

**Nodes Involved:**  
- AI Summarizer  
- Google Gemini Chat Model (LM)  
- Structured Output Parser  

**Node Details:**  

- **AI Summarizer**  
  - Type: Langchain Agent Node  
  - Role: Sends the cleaned transcript to the Gemini 2.5 model to produce a structured summary and script.  
  - Config:  
    - Input text includes the raw transcript.  
    - System prompt instructs model to:  
      - Detect genre and tone  
      - Extract core content without fluff  
      - Produce a JSON output with fields: genre, title, summary, script  
      - Write all in configured output language (from Config node)  
      - Script starts with “This is a summary of the video...”  
      - Keep script under 300 words  
  - Output Parser: Enabled, expects strict JSON as defined by Structured Output Parser.  
  - Input: Formatted transcript string  
  - Output: JSON object with summarized metadata and text-to-speech script  
  - Edge Cases: AI model timeout, invalid JSON output, language mismatch.

- **Google Gemini Chat Model**  
  - Type: Langchain Google PaLM Chat Model  
  - Role: Provides the underlying AI model interface for the AI Summarizer agent.  
  - Credentials: Google PaLM API key required.  
  - Input/Output: Connected internally to AI Summarizer’s AI language model input.  
  - Edge Cases: Authentication failures, quota exceeded.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI output to enforce JSON schema: genre, title, summary, script.  
  - Auto-fix: Enabled to correct minor format issues.  
  - Edge Cases: Parsing errors if AI output deviates from expected schema.

---

#### 2.5 Audio Generation

**Overview:**  
Converts the AI-generated script text into an MP3 audio file using OpenAI’s Text-to-Speech API.

**Nodes Involved:**  
- OpenAI TTS (Create Audio)  

**Node Details:**  

- **OpenAI TTS (Create Audio)**  
  - Type: Langchain OpenAI Node (Audio resource)  
  - Role: Takes the script text and generates an MP3 audio file using OpenAI’s `tts-1` model.  
  - Input: `script` field from AI Summarizer’s JSON output  
  - Output: Binary audio data (MP3)  
  - Credentials: OpenAI API key required  
  - Edge Cases: Network errors, API limits, invalid input text.

---

#### 2.6 Delivery

**Overview:**  
Sends the generated audio file and a formatted text summary back to the Telegram user.

**Nodes Involved:**  
- Send Audio Summary  
- Send Text Summary  

**Node Details:**  

- **Send Audio Summary**  
  - Type: Telegram Node  
  - Role: Sends the MP3 audio file to the user’s chat as an audio message.  
  - Config: Uses binary audio data from OpenAI TTS node, filename “Summary.mp3”, chat ID from Telegram Trigger.  
  - Output: Passes control to Send Text Summary node.  
  - Edge Cases: Telegram file size limits, network errors.

- **Send Text Summary**  
  - Type: Telegram Node  
  - Role: Sends a formatted HTML message containing:  
    - Bolded video title  
    - Hashtagged genre tag (spaces removed)  
    - Blockquoted short summary  
  - Chat ID from Telegram Trigger  
  - Edge Cases: HTML parsing issues in Telegram, messaging API errors.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                     | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                  |
|-----------------------------|-----------------------------------|-----------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                  | Entry point for Telegram messages |                            | Config                        |                                                                                                              |
| Config                     | Set                              | Defines output language parameter | Telegram Trigger           | Check: Is Valid URL?           | ⚙️ Configuration: set `output_language` (e.g., English, Spanish)                                            |
| Check: Is Valid URL?         | If                               | Validates YouTube URL format      | Config                     | Send Working on it Message, Send Error Message | 🛡️ Input Validation: regex check for valid YouTube URL                                                      |
| Send Working on it Message   | Telegram                         | Sends processing status message   | Check: Is Valid URL? (true) | Decodo (Fetch Transcript)      |                                                                                                              |
| Send Error Message           | Telegram                         | Sends invalid URL error message   | Check: Is Valid URL? (false)|                               |                                                                                                              |
| Decodo (Fetch Transcript)    | Decodo API                      | Fetches video transcript          | Send Working on it Message | Check for Transcript Error     | 📥 Transcript Extraction: Decodo fetches transcript; 80% discount coupon provided                            |
| Check for Transcript Error   | If                               | Detects transcript extraction errors | Decodo (Fetch Transcript)  | Send Transcript Failed Message, Code: Format Transcript |                                                                                                              |
| Send Transcript Failed Message | Telegram                       | Notifies user of transcript failure | Check for Transcript Error (true) |                               |                                                                                                              |
| Code: Format Transcript      | Code (JavaScript)                | Converts transcript JSON to text  | Check for Transcript Error (false) | AI Summarizer                 | 📝 Formatting: flattens Decodo transcript JSON to clean text                                                |
| AI Summarizer                | Langchain Agent (Google Gemini) | Summarizes transcript into JSON   | Code: Format Transcript     | OpenAI TTS (Create Audio)      | 🧠 AI Summarization: Gemini 2.5 generates structured summary and script                                     |
| Google Gemini Chat Model     | Langchain LM                    | Underlying AI model for summarizer| AI Summarizer (ai_languageModel) | Structured Output Parser      |                                                                                                              |
| Structured Output Parser     | Langchain Output Parser          | Parses AI output into JSON schema | Google Gemini Chat Model    | AI Summarizer (ai_outputParser)|                                                                                                              |
| OpenAI TTS (Create Audio)    | Langchain OpenAI (Audio)        | Converts script text to MP3 audio | AI Summarizer              | Send Audio Summary             | 🎙️ Audio Generation: OpenAI TTS produces MP3 audio                                                         |
| Send Audio Summary           | Telegram                         | Sends audio message to user       | OpenAI TTS (Create Audio)   | Send Text Summary              | ✈️ Delivery: sends audio and formatted text summary                                                        |
| Send Text Summary            | Telegram                         | Sends formatted text summary      | Send Audio Summary          |                               |                                                                                                              |
| Sticky Note                 | Sticky Note                     | Documentation notes               |                            |                               | 🎙️ Workflow overview and core function descriptions                                                        |
| Sticky Note1                | Sticky Note                     | Documentation notes               |                            |                               | ⚙️ Configuration details                                                                                      |
| Sticky Note2                | Sticky Note                     | Documentation notes               |                            |                               | 🛡️ Input validation explanation                                                                              |
| Sticky Note3                | Sticky Note                     | Documentation notes               |                            |                               | 📥 Transcript extraction details, Decodo coupon code                                                        |
| Sticky Note4                | Sticky Note                     | Documentation notes               |                            |                               | 📝 Transcript formatting explanation                                                                        |
| Sticky Note5                | Sticky Note                     | Documentation notes               |                            |                               | 🧠 AI summarization with Gemini 2.5 details                                                                 |
| Sticky Note6                | Sticky Note                     | Documentation notes               |                            |                               | 🎙️ Audio generation with OpenAI TTS details                                                                 |
| Sticky Note7                | Sticky Note                     | Documentation notes               |                            |                               | ✈️ Delivery details                                                                                           |
| Sticky Note8                | Sticky Note                     | Documentation notes               |                            |                               | 🎁 Decodo 80% discount coupon with signup link                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure to listen for “message” updates  
   - Restrict user IDs to `66336158`  
   - Set credentials with your Telegram Bot API key  

2. **Create Config Node (Set):**  
   - Type: Set  
   - Add string parameter `output_language` with default value "English"  
   - Connect Telegram Trigger node’s main output to Config node input  

3. **Create "Check: Is Valid URL?" Node (If):**  
   - Type: If  
   - Condition: Check if Telegram message text matches regex for YouTube URLs:  
     `(?:https?:\/\/)?(?:www\.)?(?:youtube\.com\/watch\?v=|youtu\.be\/)[a-zA-Z0-9_-]{11}(?:[?&].*)?$`  
   - Set strict type validation and case sensitivity  
   - Connect Config node output to this node input  

4. **Create "Send Working on it Message" Node (Telegram):**  
   - Type: Telegram  
   - Text: `"Found it! 🎬 fetching transcript and generating audio... this may take a minute."`  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Disable attribution  
   - Connect true branch of URL validation to this node  

5. **Create "Send Error Message" Node (Telegram):**  
   - Type: Telegram  
   - Text: `"❌ That doesn't look like a valid YouTube video link. Please send a link like youtube.com/watch?v=... or youtu.be/..."`  
   - Chat ID: same as above  
   - Disable attribution  
   - Connect false branch of URL validation to this node  

6. **Create "Decodo (Fetch Transcript)" Node:**  
   - Type: Decodo  
   - Operation: youtube_transcript  
   - Language code: "en"  
   - Video ID: Extracted using regex from Telegram message text:  
     `={{ $('Telegram Trigger').item.json.message.text.match(/(?:youtu\.be\/|youtube\.com\/(?:.*v=|.*\/|shorts\/))([a-zA-Z0-9_-]{11})/)[1] }}`  
   - Use Decodo API credentials  
   - Connect "Send Working on it Message" node output to this node  

7. **Create "Check for Transcript Error" Node (If):**  
   - Type: If  
   - Condition: Check if `$json.results[0].content.error` is true  
   - Connect Decodo output to this node  

8. **Create "Send Transcript Failed Message" Node (Telegram):**  
   - Type: Telegram  
   - Text:  
     ```
     ⚠️ **Transcript Failed**  
     I couldn't extract text from this video. It might not have captions available.  
     **Reason:**  
     {{ $json.results[0].content.error.message }}
     ```  
   - Markdown parse mode  
   - Chat ID from Telegram Trigger  
   - Connect true branch of error check to this node  

9. **Create "Code: Format Transcript" Node (Code):**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const content = $input.first().json.results?.[0]?.content;

     if (!Array.isArray(content)) {
         return [{ json: { transcript: "" } }];
     }

     const fullText = content.map(item => {
         return item.transcriptSegmentRenderer?.snippet?.runs?.[0]?.text || "";
     }).join(' ');

     return [{ json: { transcript: fullText } }];
     ```  
   - Connect false branch of transcript error check to this node  

10. **Create "AI Summarizer" Node (Langchain Agent):**  
    - Type: Langchain Agent  
    - Input Text: `=**Input Transcript:**\n{{ $json.transcript }}`  
    - System Prompt: Use detailed instructions including:  
      - Expert Audio Summarizer role  
      - Output in `{{ $('Config').item.json.output_language }}`  
      - JSON schema with genre, title, summary, script  
      - Script starts with "This is a summary of the video..."  
      - Script under 300 words  
    - Enable structured JSON output parser  
    - Connect Code node output to this node  

11. **Create "Google Gemini Chat Model" Node (Langchain LM):**  
    - Type: Langchain Google PaLM Chat Model  
    - Connect as AI language model input to AI Summarizer node  
    - Use Google PaLM API credentials  

12. **Create "Structured Output Parser" Node (Langchain Parser):**  
    - Type: Structured Output Parser  
    - Use JSON schema example matching AI Summarizer output  
    - Enable auto-fix  
    - Connect as output parser in AI Summarizer node  

13. **Create "OpenAI TTS (Create Audio)" Node (Langchain OpenAI):**  
    - Type: Langchain OpenAI node  
    - Resource: Audio  
    - Input: `={{ $json.output.script }}` (script from AI Summarizer)  
    - Use OpenAI API credentials  
    - Connect AI Summarizer output to this node  

14. **Create "Send Audio Summary" Node (Telegram):**  
    - Type: Telegram  
    - Operation: Send audio  
    - Chat ID: From Telegram Trigger  
    - Binary data enabled, file name set to "Summary.mp3"  
    - Connect OpenAI TTS output to this node  

15. **Create "Send Text Summary" Node (Telegram):**  
    - Type: Telegram  
    - Text (HTML parse mode):  
      ```
      <b>{{ $('AI Summarizer').item.json.output.title }}</b>

      #{{ $('AI Summarizer').item.json.output.genre.replace(/ /g, '') }}

      <blockquote>{{ $('AI Summarizer').item.json.output.summary }}</blockquote>
      ```  
    - Chat ID from Telegram Trigger  
    - Disable attribution  
    - Connect Send Audio Summary output to this node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow leverages Decodo API with an exclusive 80% discount coupon code `ATTAN8N` for the 23k Advanced Scraping API plan. | [Decodo Signup & Coupon](https://visit.decodo.com/c/6679292/3071239/17480)                      |
| The AI Summarizer uses Google Gemini 2.5, selected for its large context window suitable for long transcripts.                 |                                                                                               |
| The transcript extraction handles YouTube videos with captions only; music videos or no-caption videos trigger user alerts.    |                                                                                               |
| The system prompt enforces output in a configurable language, allowing multilingual summaries and audio generation.            |                                                                                               |
| Telegram messages are sent in HTML or Markdown as appropriate; all Telegram API interactions require an authorized bot token.  |                                                                                               |
| The audio generation uses OpenAI's `tts-1` model for fast and high-quality MP3 conversion of the script text.                   |                                                                                               |

---

This documentation fully describes the workflow’s logic, node configuration, and dependencies, enabling advanced users or automation agents to understand, reproduce, and modify the system effectively.