YouTube Video Transcription & Summary to Telegram with GPT-4o

https://n8nworkflows.xyz/workflows/youtube-video-transcription---summary-to-telegram-with-gpt-4o-4964


# YouTube Video Transcription & Summary to Telegram with GPT-4o

### 1. Workflow Overview

This workflow automates the process of transcribing a YouTube video URL sent to a Telegram bot, then generating a structured and SEO-optimized summary of the video content using GPT-4o, and finally sending a detailed summary back to the user via Telegram. It targets content creators, marketers, and online business owners who want to repurpose YouTube content into engaging blog posts and social media content efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Video Transcription:** Receives a YouTube video URL from a Telegram user and obtains the video transcript.
- **1.2 AI Processing and Content Generation:** Uses LangChain with GPT-4o to analyze the transcript and generate a comprehensive SEO-friendly summary and blog post structure.
- **1.3 Output Delivery via Telegram:** Parses the AI output and sends a formatted summary message back to the Telegram user.
- **1.4 Auxiliary and Disabled Nodes:** Includes a disabled HTTP Request node originally intended for transcription via Deepgram, and sticky notes with instructions and credits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Video Transcription

**Overview:**  
This block starts the workflow by triggering on a Telegram message containing a YouTube URL, then fetches the transcript of the video using the Supadata API.

**Nodes Involved:**  
- Input URL (Telegram Trigger)  
- Make Transcribe (Supadata API)  
- HTTP Request (Deepgram API, disabled)  
- Sticky Note (Instructional)

**Node Details:**  

- **Input URL**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for incoming Telegram messages containing YouTube URLs to start the workflow.  
  - *Configuration:* Watches for "message" updates; uses Telegram API credentials linked to `@motuyoutubebot`.  
  - *Key expression:* Extracts the text message containing the YouTube URL (`{{$json.message.text}}`).  
  - *Input:* Incoming Telegram message event.  
  - *Output:* JSON object with message data including chat ID and video URL text.  
  - *Edge cases:* Invalid or no URL in message may cause downstream errors. Telegram API rate limits or connectivity issues possible.

- **Make Transcribe**  
  - *Type:* Supadata API node  
  - *Role:* Retrieves the transcription text of the YouTube video URL received.  
  - *Configuration:* Uses the input URL (`={{ $json.message.text }}`) as `videoId` parameter and performs `getTranscript` operation.  
  - *Credentials:* Supadata API key required.  
  - *Input:* YouTube URL from `Input URL` node.  
  - *Output:* JSON containing the transcription content under `content`.  
  - *Edge cases:* API failures, invalid URL, transcription not available, or quota exceeded.

- **HTTP Request** (Disabled)  
  - *Type:* HTTP Request  
  - *Role:* Intended to transcribe audio via Deepgram API (alternative to Supadata).  
  - *Configuration:* POST to Deepgram with URL and headers for authentication; currently disabled and not connected.  
  - *Edge cases:* Disabled, so no effect; if enabled, possible auth failures or request errors.

- **Sticky Note**  
  - *Role:* Provides instructions on required API keys (Deepgram and Supadata) and basic usage notes.  
  - *Content:* Notes where to get API keys and how to use the bot (send URL to bot to start transcription).

---

#### 2.2 AI Processing and Content Generation

**Overview:**  
This block uses LangChain and OpenAI GPT-4o-mini to analyze the transcript, generate a detailed summary, SEO-optimized blog content, and social media posts in a structured JSON format.

**Nodes Involved:**  
- Summarize (LangChain Chain LLM)  
- OpenAI (OpenAI LLM Chat)  
- Parsing (Structured Output Parser)  
- Sticky Note1 (Instructional)

**Node Details:**  

- **Summarize**  
  - *Type:* LangChain Chain LLM  
  - *Role:* Core node that sends the transcription to an AI prompt instructing GPT-4o-mini to generate a structured SEO blog post summary and related content.  
  - *Configuration:*  
    - Uses a detailed system prompt defining a two-step process: create bullet point summary and transform into SEO blog post with specified tone, structure, and SEO guidelines.  
    - Passes the transcription content dynamically (`{{ $json.content }}`) for analysis.  
    - Prompt includes detailed content and SEO elements to be generated.  
  - *Input:* Transcription text from `Make Transcribe` node.  
  - *Output:* AI-generated structured text in JSON format per the example schema.  
  - *Edge cases:* AI response failure, rate limit, prompt syntax issues, or incomplete transcription may affect output quality.

- **OpenAI**  
  - *Type:* OpenAI Chat LLM (GPT-4o-mini)  
  - *Role:* Provides the underlying GPT-4o-mini model for the LangChain node to process the prompt.  
  - *Credentials:* OpenAI API key configured (`n8n - Money manager Khairul`).  
  - *Input:* Prompt from `Summarize` node.  
  - *Output:* AI text completion used by `Summarize`.  
  - *Edge cases:* API quota exceeded, connectivity issues, or latency.

- **Parsing**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses the AI's output into a well-defined JSON structure matching the provided schema.  
  - *Configuration:* Uses a JSON schema example defining video analysis, content summary, social media content, blog content, and automation metadata.  
  - *Input:* Raw AI output from `OpenAI` node.  
  - *Output:* Structured JSON object for downstream use.  
  - *Edge cases:* Parsing errors if AI output deviates from schema, incomplete or malformed JSON.

- **Sticky Note1**  
  - *Role:* Provides a brief explanation of this block's function: AI parsing and sending summary via OpenAI.  
  - *Content:* "Using OpenAI, extract the value from transcript and send in bullet point."

---

#### 2.3 Output Delivery via Telegram

**Overview:**  
This block formats and sends the AI-generated summary back to the Telegram user as a message with clear sections for insights, tips, problems, solutions, and highlights.

**Nodes Involved:**  
- Send Summary (Telegram node)  
- Sticky Note2 (Credits and contact)

**Node Details:**  

- **Send Summary**  
  - *Type:* Telegram node (sendMessage)  
  - *Role:* Sends the formatted summary message including video analysis title, main themes, key insights, actionable tips, problems identified, solutions provided, and highlights back to the Telegram chat that initiated the workflow.  
  - *Configuration:*  
    - Uses Telegram API credentials linked to `@motuyoutubebot`.  
    - Text is dynamically composed with mustache-like expressions accessing the parsed JSON from AI output (`$json.output.video_analysis.title`, etc.).  
    - Chat ID is taken from the original Telegram input message to reply to the correct user.  
    - Message uses unicode box drawing characters and emojis for readability and emphasis.  
  - *Input:* Structured AI output from `Parsing` node.  
  - *Output:* Telegram message sent to user.  
  - *Edge cases:* Telegram API rate limits, invalid chat ID, message size limits, or formatting issues.

- **Sticky Note2**  
  - *Role:* Contains credit and contact information for the workflow author, with links for support and donations.  
  - *Content:* "This Template is Free... Have any request, contact me [Here](https://shop.khmuhtadin.com/pages/contact) or care to give [coffee](coff.ee/khmuhtadin) would be awesome."

---

### 3. Summary Table

| Node Name     | Node Type                         | Functional Role                          | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                                   |
|---------------|----------------------------------|----------------------------------------|---------------------|--------------------|---------------------------------------------------------------------------------------------------------------|
| Input URL     | Telegram Trigger                 | Receives YouTube URL from Telegram     | -                   | Make Transcribe     | ## Get Input and Transcribe Video<br>Required API Keys:<br>- Deepgram API: https://deepgram.com/<br>- Supadata KEY: https://supadata.com/<br>Setup Instructions: Send URL to your bot to start Transcribing |
| Make Transcribe | Supadata API                   | Fetches transcript of YouTube video    | Input URL           | Summarize          | ## Get Input and Transcribe Video (see above)                                                                |
| HTTP Request  | HTTP Request (Disabled)          | Alternative transcription (Deepgram)  | Input URL           | -                  | ## Get Input and Transcribe Video (see above)                                                                |
| Summarize     | LangChain Chain LLM              | Generates structured SEO blog and summary | Make Transcribe    | Send Summary        | ## AI PARSING & Send Summary<br>Using OpenAI, extract the value from transcript and send in bullet point       |
| OpenAI        | OpenAI Chat LLM (GPT-4o-mini)   | Language model for AI processing       | Summarize           | Parsing             | ## AI PARSING & Send Summary (see above)                                                                      |
| Parsing       | LangChain Structured Output Parser | Parses AI output into JSON structure | OpenAI              | Summarize           | ## AI PARSING & Send Summary (see above)                                                                      |
| Send Summary  | Telegram node (sendMessage)      | Sends formatted summary to Telegram    | Summarize           | -                   | ## AI PARSING & Send Summary (see above)                                                                      |
| Sticky Note   | Sticky Note                     | Instructions for Input and Transcription | -                 | -                   | ## Get Input and Transcribe Video (see above)                                                                 |
| Sticky Note1  | Sticky Note                     | Explanation of AI Parsing block         | -                   | -                   | ## AI PARSING & Send Summary (see above)                                                                      |
| Sticky Note2  | Sticky Note                     | Credits and contact info                 | -                   | -                   | This Template is Free. Have any request, contact me [Here](https://shop.khmuhtadin.com/pages/contact) or care to give [coffee](coff.ee/khmuhtadin) would be awesome |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Name: `Input URL`  
   - Configure to listen for "message" updates.  
   - Connect Telegram API credentials associated with your bot (`@motuyoutubebot`).  
   - Position: (0,0)  
   - Output: Receives user messages with YouTube URLs.

2. **Create Supadata Node**  
   - Type: Supadata API node (custom or HTTP Request if custom node unavailable)  
   - Name: `Make Transcribe`  
   - Operation: `getTranscript`  
   - Parameter: `videoId` set to expression `{{$json.message.text}}` (passes YouTube URL from Telegram message).  
   - Credentials: Configure Supadata API key.  
   - Position: (260,0)  
   - Connect input from `Input URL` node.

3. *(Optional)* **Create HTTP Request Node for Deepgram** (disabled by default)  
   - Type: HTTP Request  
   - Name: `HTTP Request`  
   - Method: POST to `https://api.deepgram.com/v1/listen`  
   - Headers: Authorization with `Token YOUR-TOKEN-HERE` (replace with your Deepgram key)  
   - Body: JSON with parameter `"url": {{$json.message.text}}`  
   - Query Parameters: model=`nova-2`, smart_format=`true`  
   - Disabled by default, no connections.  
   - Position: (280,-220)

4. **Create LangChain Chain LLM Node**  
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Name: `Summarize`  
   - Parameters:  
     - `text`: prompt instructing the AI to:  
       - Step 1: Create bullet point summary from transcription  
       - Step 2: Transform summary into SEO blog post as per detailed guidelines in the prompt (tone, structure, keywords, calls to action)  
     - Use expression to pass transcription content: `{{$json.content}}`  
   - Position: (560,0)  
   - Connect input from `Make Transcribe`.

5. **Create OpenAI Chat Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: `OpenAI`  
   - Model: Select `"gpt-4o-mini"` from list mode selector  
   - Credentials: OpenAI API key configured (e.g., `n8n - Money manager Khairul`)  
   - Position: (560,180)  
   - Connect input from `Summarize`.

6. **Create LangChain Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Name: `Parsing`  
   - Configure JSON schema example to parse AI output into structured fields including video_analysis, content_summary, social_media_content, blog_content, automation_data (use provided schema example).  
   - Position: (740,180)  
   - Connect input from `OpenAI`.

7. **Connect Output Parser back to Summarize**  
   - Connect `Parsing` node output back to `Summarize` node's second input to handle parsed output.

8. **Create Telegram Node to Send Message**  
   - Type: Telegram (sendMessage) node  
   - Name: `Send Summary`  
   - Text: Compose dynamic message using expressions from parsed AI output, including title, key insights, tips, problems, solutions, and highlights (use the provided multiline template).  
   - Chat ID: Use expression `{{$node["Input URL"].json.message.chat.id}}` to reply to the user.  
   - Credentials: Telegram API with same bot credentials as before.  
   - Position: (1000,0)  
   - Connect input from `Summarize`.

9. **Add Sticky Notes for Documentation**  
   - Create Sticky Note nodes with instructions for API keys, usage, and credits as in the original workflow.

10. **Workflow Settings**  
    - Execution order: `v1` (linear)  
    - Ensure all credentials (Telegram, Supadata, OpenAI) are set and tested.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Deepgram API offers a free $200 credit for transcription services.                                              | https://deepgram.com/                                                                           |
| Supadata transcription API key required for video transcript extraction.                                         | https://supadata.com/                                                                           |
| Contact or support for this workflow template available at:                                                     | https://shop.khmuhtadin.com/pages/contact                                                      |
| Donations or support via coffee appreciated:                                                                    | https://coff.ee/khmuhtadin                                                                     |
| Workflow uses GPT-4o-mini model via LangChain integration in n8n for advanced language processing.               | OpenAI GPT-4o-mini is a lightweight powerful model suitable for structured content generation. |
| The AI prompt is carefully crafted to produce SEO-optimized blog content with tone and structure guidelines.    | Enables repurposing video content into multiple digital formats easily.                         |

---

**Disclaimer:**  
The provided text and workflow are generated solely from an automated n8n workflow. All data processed is legal and public, and the workflow complies with content policies.