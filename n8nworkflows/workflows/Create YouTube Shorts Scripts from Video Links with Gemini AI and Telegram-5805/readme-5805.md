Create YouTube Shorts Scripts from Video Links with Gemini AI and Telegram

https://n8nworkflows.xyz/workflows/create-youtube-shorts-scripts-from-video-links-with-gemini-ai-and-telegram-5805


# Create YouTube Shorts Scripts from Video Links with Gemini AI and Telegram

### 1. Workflow Overview

This workflow automates the creation of YouTube Shorts scripts from video links sent via Telegram, leveraging AI-powered transcription and language modeling. The core use case is to enable users to submit a YouTube video URL to a Telegram bot, which then transcribes the video, generates a concise and engaging trivia-style script suitable for YouTube Shorts using Google Gemini AI, and returns the script back to the user in Telegram.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Transcription:** Receives a YouTube video URL via Telegram, extracts the video ID, and uses the Supadata transcription service to obtain the transcript of the video.
- **1.2 Script Preparation & AI Processing:** Maps and formats the raw transcript text, then invokes the Google Gemini language model to generate a polished, trivia-rich YouTube Shorts script.
- **1.3 Output Delivery:** Parses the AI output into structured JSON and sends the final script and title back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Transcription

- **Overview:**  
  This block captures the user’s YouTube video URL sent through Telegram, triggers transcription via Supadata, and prepares the transcript data for AI processing.

- **Nodes Involved:**  
  - Input URL (telegramTrigger)  
  - Make Transcribe (supadata)  
  - Script mapping (set)  
  - Sticky Note (Get Video Link and Transcribe Video)

- **Node Details:**

  - **Input URL**  
    - *Type:* Telegram Trigger  
    - *Role:* Listens for incoming Telegram messages containing YouTube video URLs.  
    - *Config:* Listens to all "message" updates; uses Telegram bot credentials named "youtube_script_maker_bot".  
    - *Input/Output:* No input (trigger node). Outputs message JSON with video link text and chat metadata.  
    - *Edge Cases:* Invalid or malformed video URLs, unsupported message types, Telegram API downtime.

  - **Make Transcribe**  
    - *Type:* Supadata node for transcription  
    - *Role:* Extracts the video ID from the Telegram message text and requests the transcription from Supadata API.  
    - *Config:* The "videoId" parameter is dynamically set from the Telegram message text (`={{ $json.message.text }}`). Uses valid Supadata API credentials.  
    - *Input:* Receives video URL text from "Input URL".  
    - *Output:* Returns transcription content structured as an array of transcript segments.  
    - *Edge Cases:* API key invalid or expired, video ID not found, transcription unavailable, API timeouts.

  - **Script mapping**  
    - *Type:* Set node  
    - *Role:* Concatenates all transcript segments (`$json.content.map(item => item.text).join('\n')`) into a single string `full_script`.  
    - *Config:* Uses JavaScript expression to merge transcript texts.  
    - *Input:* Receives raw transcript content array from "Make Transcribe".  
    - *Output:* Outputs a unified `full_script` string for downstream AI processing.  
    - *Edge Cases:* Empty transcript content, unexpected transcript data structure.

  - **Sticky Note** (Get Video Link and Transcribe Video)  
    - *Type:* Sticky Note  
    - *Role:* Documentation block indicating the purpose of this section.  
    - *Content:* "## Get Video Link and Transcribe Video"

---

#### 2.2 Script Preparation & AI Processing

- **Overview:**  
  This block takes the unified transcript text and uses Google Gemini AI to generate a concise, engaging YouTube Shorts script, then parses the AI output into structured JSON.

- **Nodes Involved:**  
  - Google Gemini Chat Model (lmChatGoogleGemini)  
  - Create Script (chainLlm)  
  - Parsing (outputParserStructured)  
  - Sticky Note1 (AI PARSING & Send Script)  
  - Sticky Note2 (Template Info & Contact)

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type:* Google Gemini language model node  
    - *Role:* Provides the AI language model used for script generation.  
    - *Config:* Uses the "models/gemini-2.5-pro" model with Google PaLM API credentials.  
    - *Input:* Connected to "Create Script" as the AI language model provider.  
    - *Output:* Returns AI-generated text output to the "Create Script" node.  
    - *Edge Cases:* API quota exceeded, authentication errors, network failures.

  - **Create Script**  
    - *Type:* Chain LLM node  
    - *Role:* Crafts the prompt for the AI, instructing it to create a trivia-based YouTube Shorts script from the transcript text.  
    - *Config:*  
      - Prompt includes: professional scriptwriter role, conditions for content and style, a required output format with "title" and "script" fields.  
      - Input text is bound to `full_script` from the "Script mapping" node.  
      - Output is parsed by a structured output parser downstream.  
    - *Input:* Receives the concatenated transcript string `full_script`.  
    - *Output:* Produces raw AI output for parsing.  
    - *Edge Cases:* Prompt misinterpretation, incomplete or irrelevant script generation, timeout.

  - **Parsing**  
    - *Type:* Langchain structured output parser  
    - *Role:* Converts the AI-generated string into structured JSON containing "title" and "script".  
    - *Config:* Uses a JSON schema example with "title" and "script" fields to validate and parse the AI output.  
    - *Input:* Connected to "Create Script" node's parsed output.  
    - *Output:* Structured JSON with keys `output.title` and `output.script`.  
    - *Edge Cases:* Output not matching schema, parsing errors.

  - **Sticky Note1** (AI PARSING & Send Script)  
    - *Type:* Sticky Note  
    - *Role:* Describes this block’s function in AI parsing and script delivery.  
    - *Content:* "## AI PARSING & Send Script"

  - **Sticky Note2** (Template Info & Contact)  
    - *Type:* Sticky Note  
    - *Role:* Provides information about the template being free and contact information for requests.  
    - *Content:* "## This Template is Free✨\n\n### Have any request, contact me [Here](https://x.com/taiki_16_k)"

---

#### 2.3 Output Delivery

- **Overview:**  
  Sends the finalized YouTube Shorts script and title back to the originating Telegram user.

- **Nodes Involved:**  
  - Send Summary (telegram)  
  - Sticky Note3 (API & Setup Instructions)

- **Node Details:**

  - **Send Summary**  
    - *Type:* Telegram node  
    - *Role:* Sends the generated script and title as a message back to the Telegram user who initiated the request.  
    - *Config:*  
      - Text message concatenates `output.title` and `output.script` fields.  
      - Dynamically sets `chatId` from the original Telegram message (`$('Input URL').item.json.message.chat.id`) to reply to the correct user.  
      - Disables default Telegram attribution append.  
      - Uses "youtube_script_maker_bot" Telegram API credentials.  
    - *Input:* Receives structured AI output from "Parsing".  
    - *Output:* Sends a message to Telegram; no further outputs.  
    - *Edge Cases:* Telegram API limits, invalid chat ID, message formatting issues.

  - **Sticky Note3** (API & Setup Instructions)  
    - *Type:* Sticky Note  
    - *Role:* Lists required API keys and basic setup instructions for the workflow.  
    - *Content:*  
      ```
      ## Required API Keys:
      - **Supadata KEY**: Get your transcription API key [here](https://supadata.com/)

      ### Setup Instructions:
      1. Send URL to your bot to start Transcribing
      ```

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role                        | Input Node(s)          | Output Node(s)         | Sticky Note                                                  |
|-----------------------|-------------------------------|-------------------------------------|------------------------|------------------------|--------------------------------------------------------------|
| Input URL             | Telegram Trigger              | Receive YouTube URL via Telegram    | —                      | Make Transcribe        | ## Required API Keys: ... Setup Instructions ...             |
| Make Transcribe       | Supadata Transcription        | Get video transcript from Supadata  | Input URL              | Script mapping         | ## Get Video Link and Transcribe Video                       |
| Script mapping        | Set                          | Combine transcript text to string   | Make Transcribe         | Create Script          | ## Get Video Link and Transcribe Video                       |
| Google Gemini Chat Model | Google Gemini LM Chat        | AI language model for script gen    | Create Script (lmChat)  | Create Script          | ## AI PARSING & Send Script                                  |
| Create Script         | Chain LLM                     | Generate YouTube Shorts script      | Script mapping, Gemini  | Parsing                | ## AI PARSING & Send Script                                  |
| Parsing               | Langchain Output Parser       | Parse AI output into JSON structure | Create Script           | Send Summary           | ## AI PARSING & Send Script                                  |
| Send Summary          | Telegram                      | Send generated script back to user  | Parsing                 | —                      | ## AI PARSING & Send Script                                  |
| Sticky Note           | Sticky Note                  | Documentation for Input & Transcribe| —                      | —                      | ## Get Video Link and Transcribe Video                       |
| Sticky Note1          | Sticky Note                  | Documentation for AI Parsing & Send | —                      | —                      | ## AI PARSING & Send Script                                  |
| Sticky Note2          | Sticky Note                  | Template info and contact           | —                      | —                      | ## This Template is Free✨ Have any request, contact me [Here](https://x.com/taiki_16_k) |
| Sticky Note3          | Sticky Note                  | API key requirements and setup     | —                      | —                      | ## Required API Keys: ... Setup Instructions ...             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node ("Input URL")**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates.  
   - Credentials: Link to your Telegram bot API credentials ("youtube_script_maker_bot").  
   - Purpose: Receive YouTube video URL messages from users.  

2. **Create Supadata node ("Make Transcribe")**  
   - Type: Supadata (transcription) node  
   - Parameters:  
     - Operation: "getTranscript"  
     - videoId: Expression = `{{$json.message.text}}` (extract video ID from Telegram message)  
   - Credentials: Link to your Supadata API key.  
   - Connect the output of "Input URL" to this node.  

3. **Create Set node ("Script mapping")**  
   - Type: Set  
   - Purpose: Concatenate transcript segments into a single string.  
   - Parameters:  
     - Create field `full_script` with value: `={{ $json.content.map(item => item.text).join('\n') }}`  
   - Connect output of "Make Transcribe" to this node.  

4. **Create Google Gemini Chat Model node ("Google Gemini Chat Model")**  
   - Type: Google Gemini Language Model Chat node  
   - Parameters:  
     - Model Name: `models/gemini-2.5-pro`  
   - Credentials: Link to your Google PaLM API credentials.  

5. **Create Chain LLM node ("Create Script")**  
   - Type: Chain LLM  
   - Parameters:  
     - Text: `=#background information\n{{ $json.full_script }}`  
     - Messages: Use the provided prompt instructing the AI to generate a concise, trivia-style YouTube Shorts script (see overview).  
     - Enable output parser.  
   - Connect "Script mapping" node output to this node's input.  
   - Connect "Google Gemini Chat Model" node as the AI language model for this node.  

6. **Create Output Parser node ("Parsing")**  
   - Type: Langchain Output Parser Structured  
   - Parameters: Use JSON schema example with fields "title" and "script".  
   - Connect output of "Create Script" node to this node.  

7. **Create Telegram node ("Send Summary")**  
   - Type: Telegram  
   - Parameters:  
     - Text: `={{ $json.output.title }}\n\n{{ $json.output.script }}`  
     - Chat ID: `={{ $('Input URL').item.json.message.chat.id }}` (to reply to user)  
     - Disable append attribution.  
   - Credentials: Use the same Telegram bot credentials as in step 1.  
   - Connect output of "Parsing" node to this node.  

8. **Add Sticky Notes as documentation** (optional but recommended)  
   - Add notes describing each functional block: input & transcription, AI parsing & send script, and API keys/setup instructions.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Required API Keys: Supadata KEY needed for transcription. Obtain at https://supadata.com/           | Supadata transcription API                       |
| Template is free. For feature requests or support, contact [Here](https://x.com/taiki_16_k)        | Author contact via Twitter/X                      |
| Send the YouTube video URL to the Telegram bot to trigger transcription and script generation flow | User instructions                                |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected elements. All processed data is legal and publicly available.