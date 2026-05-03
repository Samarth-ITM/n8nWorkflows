Generate audio documentaries from web articles with ElevenLabs and BrowserAct

https://n8nworkflows.xyz/workflows/generate-audio-documentaries-from-web-articles-with-elevenlabs-and-browseract-12347


# Generate audio documentaries from web articles with ElevenLabs and BrowserAct

## 1. Workflow Overview

**Purpose:**  
This workflow turns a web article link sent to a Telegram bot into an **audio documentary**. It (1) classifies the user message, (2) scrapes article content via **BrowserAct**, (3) rewrites it into a narrative audio script + Telegram caption + filename using an LLM (via **OpenRouter/Claude**), (4) generates speech with **ElevenLabs**, and (5) delivers the audio back to Telegram. If no valid link is provided, it falls back to a conversational response and/or asks for a URL.

**Target use cases:**
- “Turn this article into an audio story/podcast”
- “Summarize/read this page” (only if a URL is present)
- Regular chat and “no link provided” handling in Telegram

### 1.1 Input Reception (Telegram)
Receives messages from Telegram and forwards them into AI-based intent recognition.

### 1.2 Intent Recognition + Routing
Classifies the message into one of:
- `Article_Request` (must include a valid URL)
- `Chat`
- `NoData` (including “article request but missing link”)

Routes to either the article pipeline or chat fallback.

### 1.3 Article Extraction (BrowserAct)
Runs a BrowserAct workflow template to extract the article’s text/content.

### 1.4 Script + Caption + Filename Generation (LLM via OpenRouter)
Transforms extracted content into:
- `elevenlabtext` (plain text TTS script)
- `Telegram` (HTML-formatted caption)
- `Audioname` (mp3 filename)

### 1.5 Voice Production + Delivery (ElevenLabs → Telegram)
Generates an audio file from `elevenlabtext` and sends it to the user with the caption + filename.

### 1.6 Conversational Fallback (Gemini)
If the message is `Chat` or `NoData`, responds conversationally or prompts the user to send a link.

---

## 2. Block-by-Block Analysis

### Block A — Input Reception (Telegram)
**Overview:** Receives incoming Telegram messages that initiate the workflow.  
**Nodes Involved:** `User Sends Message to Bot`

#### Node: User Sends Message to Bot
- **Type / Role:** `telegramTrigger` — entry point trigger on new Telegram messages.
- **Configuration (interpreted):**
  - Listens to update type: `message`
- **Key data produced:**
  - `{{$json.message.text}}` (message body)
  - `{{$json.message.chat.id}}` (chat id used for replies)
- **Connections:**
  - **Output →** `Validate user Input`
- **Edge cases / failures:**
  - Telegram credential issues (bot token invalid/revoked)
  - Bot not added to chat / blocked by user
  - Non-text messages (stickers, photos) may not have `message.text` (could break downstream expressions)

---

### Block B — Intent Recognition + Structured Parsing + Routing
**Overview:** Uses an LLM to classify user input into `Article_Request`, `Chat`, or `NoData`, then routes accordingly.  
**Nodes Involved:** `Validate user Input`, `Google Gemini1`, `Structured Output Parser`, `Check For Input Type`

#### Node: Validate user Input
- **Type / Role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — prompts an LLM to output classification JSON only.
- **Configuration choices:**
  - **Input text:** `={{ $json.message.text }}`
  - **System message:** Strict JSON-only classifier with priority rules:
    - Article intent requires URL → `{"Type":"Article_Request","Link":"..."}`  
    - Article intent without URL → `{"Type":"NoData","Link":"Null"}`
    - Greetings/small talk → `{"Type":"Chat","Link":"Null"}`
  - **Has output parser:** enabled (expects structured JSON)
- **AI model connection:**
  - Uses **Google Gemini1** as `ai_languageModel`
  - Uses **Structured Output Parser** as `ai_outputParser`
- **Output shape (typical):**
  - `{{$json.output.Type}}`
  - `{{$json.output.Link}}`
- **Connections:**
  - **Main output →** `Check For Input Type`
- **Edge cases / failures:**
  - `message.text` missing (non-text Telegram update)
  - LLM returns invalid JSON despite instructions (mitigated by parser auto-fix, but not guaranteed)
  - URL extraction errors or partial URLs
  - Model throttling / quota errors (Gemini API)

#### Node: Google Gemini1
- **Type / Role:** `lmChatGoogleGemini` — provides the model for intent classification.
- **Configuration choices:**
  - Default options (no explicit model parameters shown)
- **Connections:**
  - **ai_languageModel →** `Validate user Input` and `Structured Output Parser`
- **Edge cases / failures:**
  - Credential/quota errors
  - Safety filters possibly refusing some content (less likely here, but possible)

#### Node: Structured Output Parser
- **Type / Role:** Structured output parser (`outputParserStructured`) — converts LLM output into a JSON object.
- **Configuration choices:**
  - `autoFix: true` (attempts to correct malformed JSON)
  - Schema example: `{"Type":"Article_Request","Link":"extracted_link"}`
- **Connections:**
  - **ai_outputParser →** `Validate user Input`
- **Edge cases / failures:**
  - If the model returns content that cannot be repaired into valid JSON, downstream switch conditions will fail (`$json.output` may be missing)

#### Node: Check For Input Type
- **Type / Role:** `switch` — routes based on `{{$json.output.Type}}`
- **Configuration choices:**
  - Rule 1: if `output.Type == "Article_Request"` → Article pipeline
  - Rule 2: if `output.Type == "Chat"` → Chat fallback
  - Rule 3: if `output.Type == "NoData"` → Chat fallback (prompt for link)
- **Connections:**
  - **Output 0 (Article_Request) →** `Notify User` and `Get Article Data from BrowserAct` (two parallel main connections)
  - **Output 1 (Chat) →** `Chatting With User`
  - **Output 2 (NoData) →** `Chatting With User`
- **Edge cases / failures:**
  - If `output.Type` is missing or unexpected (typo), no route matches
  - Strict comparison is case-sensitive; `article_request` would not match

---

### Block C — Article Extraction (BrowserAct)
**Overview:** When an article link is detected, BrowserAct runs a predefined workflow to scrape/extract article text.  
**Nodes Involved:** `Notify User`, `Get Article Data from BrowserAct`

#### Node: Notify User
- **Type / Role:** `telegram` (sendMessage) — immediate acknowledgment to the user.
- **Configuration choices:**
  - **Text:** `Understood. Please give me a moment to complete that.`
  - **chatId expression:** `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - Parse mode: HTML (though message contains no HTML tags)
- **Connections:** None (it’s a side-message running in parallel with scraping)
- **Edge cases / failures:**
  - Telegram send errors (bot blocked, invalid chat id)
  - The expression depends on the trigger node item being available (it is in normal single-thread runs; in more complex multi-item contexts, using `$('User Sends Message...')` can be fragile)

#### Node: Get Article Data from BrowserAct
- **Type / Role:** `browserAct` — executes a BrowserAct hosted workflow to extract article content.
- **Configuration choices:**
  - Mode/type: `WORKFLOW`
  - Timeout: `7200` seconds (2 hours) for long-running scraping
  - BrowserAct `workflowId`: `69338962262095598`
  - Input mapping:
    - `Target_link` is set to `={{ $json.output.Link }}`
  - `open_incognito_mode: false`
- **Expected output (based on downstream usage):**
  - `{{$json.output.string}}` appears to contain extracted article text/content as a string
- **Connections:**
  - **Main output →** `Write Script for ElevenLabs`
- **Edge cases / failures:**
  - BrowserAct credential issues / workflow not found
  - Workflow template changed in BrowserAct so output structure no longer includes `output.string`
  - Link paywalls / bot detection / dynamic sites causing extraction failure
  - Very long extraction results could exceed LLM context limits downstream

---

### Block D — Script + Caption + Filename Generation (OpenRouter/Claude)
**Overview:** Converts scraped content into an ElevenLabs-friendly script plus Telegram caption and an mp3 filename, using structured JSON output parsing.  
**Nodes Involved:** `Write Script for ElevenLabs`, `OpenRouter`, `Structured Output`

#### Node: Write Script for ElevenLabs
- **Type / Role:** LangChain Agent — generates structured creative output for audio + Telegram distribution.
- **Configuration choices:**
  - **Input text:** `Article or Story Data :  {{ $json.output.string }}`
  - **System message:** Strong constraints:
    - Produce:
      - `elevenlabtext` (plain text, no markdown, 1500–3500 chars)
      - `Telegram` caption (Telegram HTML tags only, < 900 chars)
      - `Audioname` (no spaces, underscores, `.mp3`)
    - Output must be a **single valid JSON array** with one object
    - No double quotes inside string values (use single quotes)
  - **Has output parser:** enabled
- **AI model connection:**
  - Uses **OpenRouter** (`anthropic/claude-haiku-4.5`)
  - Uses **Structured Output** parser
- **Output shape (typical):**
  - `{{$json.output[0].elevenlabtext}}`
  - `{{$json.output[0].Telegram}}`
  - `{{$json.output[0].Audioname}}`
- **Connections:**
  - **Main output →** `Convert text to speech`
- **Edge cases / failures:**
  - If BrowserAct output is huge, model may truncate or fail
  - JSON constraint is strict; model may still introduce invalid JSON (parser auto-fix helps but not perfect)
  - Character limits: Telegram caption must remain below Telegram’s limit (configured target < 900)
  - Content policy or model refusal is possible depending on article content

#### Node: OpenRouter
- **Type / Role:** `lmChatOpenRouter` — provides the LLM used for script generation.
- **Configuration choices:**
  - Model: `anthropic/claude-haiku-4.5`
- **Connections:**
  - **ai_languageModel →** `Write Script for ElevenLabs` and `Structured Output`
- **Edge cases / failures:**
  - OpenRouter API key invalid / insufficient credits
  - Model availability or rate limits

#### Node: Structured Output
- **Type / Role:** Structured output parser — parses the JSON array output.
- **Configuration choices:**
  - `autoFix: true`
  - Schema example: `[{"elevenlabtext":"...","Telegram":"...","Audioname":"..."}]`
- **Connections:**
  - **ai_outputParser →** `Write Script for ElevenLabs`
- **Edge cases / failures:**
  - If the model returns non-repairable JSON, downstream nodes referencing `$json.output[0]` will fail

---

### Block E — Voice Production + Delivery (ElevenLabs → Telegram)
**Overview:** Sends the generated narrative script to ElevenLabs for speech synthesis and delivers the resulting audio file to Telegram with caption and filename.  
**Nodes Involved:** `Convert text to speech`, `Send an audio file To User`

#### Node: Convert text to speech
- **Type / Role:** ElevenLabs node (`@elevenlabs/n8n-nodes-elevenlabs.elevenLabs`) — text-to-speech generation.
- **Configuration choices:**
  - Resource: `speech`
  - **Text:** `={{ $json.output[0].elevenlabtext }}`
  - Voice: `Liam - Energetic, Social Media Creator` (voice id `TX3LPaxmHKxFdv7VOQHJ`)
  - Model: `Eleven Flash v2.5` (`eleven_flash_v2_5`)
  - Language code: `en`
- **Connections:**
  - **Main output →** `Send an audio file To User`
- **Edge cases / failures:**
  - ElevenLabs auth/credit issues
  - Text too long for the selected model limits (the workflow attempts to constrain length, but failures still possible)
  - Special characters or markup could be read aloud (system message forbids markdown to reduce this risk)

#### Node: Send an audio file To User
- **Type / Role:** `telegram` (sendAudio) — sends the generated audio binary to the originating chat.
- **Configuration choices:**
  - Operation: `sendAudio`
  - `binaryData: true` (expects audio data from previous node as binary)
  - **chatId:** `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - **caption:** `{{ $('Write Script for ElevenLabs').first().json.output[0].Telegram }}`
  - **fileName:** `{{ $('Write Script for ElevenLabs').first().json.output[0].Audioname }}`
  - Parse mode: HTML (caption uses Telegram HTML)
- **Connections:** None (terminal delivery)
- **Edge cases / failures:**
  - If ElevenLabs node output binary property name doesn’t match what Telegram expects, sending fails (may require setting the correct binary field in Telegram node depending on n8n version)
  - Caption HTML must be valid Telegram HTML (unsupported tags break formatting)
  - Telegram file size limits (audio too large)
  - Reliance on `first()` referencing can be fragile in multi-item executions

---

### Block F — Conversational Fallback (Chat / NoData)
**Overview:** If the user didn’t provide a valid link, the workflow replies conversationally or asks for a URL.  
**Nodes Involved:** `Chatting With User`, `Google Gemini`, `Answer the User`

#### Node: Chatting With User
- **Type / Role:** LangChain Agent — produces a single raw-text response.
- **Configuration choices:**
  - Input text: `Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System message:
    - If `NoData`: ask user to provide the article link
    - If `Chat`: respond normally
    - Output must be raw text (no markdown fences)
- **AI model connection:**
  - Uses **Google Gemini** as `ai_languageModel`
- **Connections:**
  - **Main output →** `Answer the User`
- **Edge cases / failures:**
  - Same `message.text` missing issue
  - Model may still produce extra formatting; Telegram node uses HTML parse mode which can misinterpret stray `<` characters

#### Node: Google Gemini
- **Type / Role:** `lmChatGoogleGemini` — model for chat fallback.
- **Configuration choices:** default options
- **Connections:**
  - **ai_languageModel →** `Chatting With User`
- **Edge cases / failures:** quota/auth, safety filters

#### Node: Answer the User
- **Type / Role:** `telegram` (sendMessage) — sends the fallback chat response.
- **Configuration choices:**
  - Text: `={{ $json.output }}`
  - chatId: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - Parse mode: HTML
- **Connections:** none (terminal for chat path)
- **Edge cases / failures:**
  - If `$json.output` is not plain text (e.g., object), Telegram send fails
  - HTML parse mode can cause issues if the model outputs invalid HTML-like text

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation | Sticky Note | Workflow documentation and requirements |  |  | ## ⚡ Workflow Overview & Setup… Requirements… Links: https://docs.browseract.com |
| Sticky Note | Sticky Note | Video reference |  |  | @[youtube](YuxjfB87F0E) |
| Step 1 Explanation | Sticky Note | Explains intent recognition |  |  | ### 🧠 Step 1: Intent Recognition… |
| Step 2 Explanation | Sticky Note | Explains scraping & scripting |  |  | ### ✍️ Step 2: Scraping & Scripting… |
| Step 3 Explanation | Sticky Note | Explains production & delivery |  |  | ### 🎙️ Step 3: Production & Delivery… |
| Step 4 Explanation | Sticky Note | Explains chat/no-link fallback |  |  | ### 💬 Step 2-2: Conversational Fallback… |
| User Sends Message to Bot | telegramTrigger | Entry point: receive Telegram message |  | Validate user Input | ### 🧠 Step 1: Intent Recognition… |
| Validate user Input | LangChain Agent | Classify intent + extract URL into JSON | User Sends Message to Bot | Check For Input Type | ### 🧠 Step 1: Intent Recognition… |
| Google Gemini1 | Google Gemini Chat Model | LLM for intent classification |  | Validate user Input; Structured Output Parser | ### 🧠 Step 1: Intent Recognition… |
| Structured Output Parser | Structured Output Parser | Parse classifier JSON |  | Validate user Input | ### 🧠 Step 1: Intent Recognition… |
| Check For Input Type | Switch | Route Article vs Chat vs NoData | Validate user Input | Notify User; Get Article Data from BrowserAct; Chatting With User | ### 🧠 Step 1: Intent Recognition… |
| Notify User | Telegram (sendMessage) | Acknowledge request immediately | Check For Input Type |  | ### ✍️ Step 2: Scraping & Scripting… |
| Get Article Data from BrowserAct | BrowserAct | Scrape/extract article data from URL | Check For Input Type | Write Script for ElevenLabs | ### ✍️ Step 2: Scraping & Scripting… |
| OpenRouter | OpenRouter Chat Model | LLM (Claude) for script generation |  | Write Script for ElevenLabs; Structured Output | ### ✍️ Step 2: Scraping & Scripting… |
| Structured Output | Structured Output Parser | Parse JSON array for script/caption/name |  | Write Script for ElevenLabs | ### ✍️ Step 2: Scraping & Scripting… |
| Write Script for ElevenLabs | LangChain Agent | Generate ElevenLabs script + Telegram caption + filename | Get Article Data from BrowserAct | Convert text to speech | ### ✍️ Step 2: Scraping & Scripting… |
| Convert text to speech | ElevenLabs | Generate audio from script text | Write Script for ElevenLabs | Send an audio file To User | ### 🎙️ Step 3: Production & Delivery… |
| Send an audio file To User | Telegram (sendAudio) | Deliver audio file with caption + filename | Convert text to speech |  | ### 🎙️ Step 3: Production & Delivery… |
| Google Gemini | Google Gemini Chat Model | LLM for chat fallback |  | Chatting With User | ### 💬 Step 2-2: Conversational Fallback… |
| Chatting With User | LangChain Agent | Compose fallback response / ask for URL | Check For Input Type | Answer the User | ### 💬 Step 2-2: Conversational Fallback… |
| Answer the User | Telegram (sendMessage) | Send fallback text response | Chatting With User |  | ### 💬 Step 2-2: Conversational Fallback… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   “Generate audio documentaries from web articles with ElevenLabs & BrowserAct”.

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Updates: `message`
   - Credential: connect your **Telegram bot** credential.
   - This is the entry node.

3. **Add Intent Classification Agent**
   - Node: **AI Agent (LangChain)**
   - Name: “Validate user Input”
   - Text field: set expression to `{{$json.message.text}}`
   - Prompt type: “Define”
   - System message: paste the classifier rules (must output raw JSON only).
   - Enable **Output Parser** (structured).

4. **Add Google Gemini model for classification**
   - Node: **Google Gemini Chat Model**
   - Name: “Google Gemini1”
   - Connect:
     - `Google Gemini1` → (ai_languageModel) → `Validate user Input`

5. **Add Structured Output Parser for classification**
   - Node: **Structured Output Parser**
   - Name: “Structured Output Parser”
   - Auto-fix: enabled
   - Schema example: `{"Type":"Article_Request","Link":"extracted_link"}`
   - Connect:
     - `Structured Output Parser` → (ai_outputParser) → `Validate user Input`

6. **Add Switch for routing**
   - Node: **Switch**
   - Name: “Check For Input Type”
   - Add 3 rules (String equals):
     - `{{$json.output.Type}}` equals `Article_Request`
     - `{{$json.output.Type}}` equals `Chat`
     - `{{$json.output.Type}}` equals `NoData`
   - Connect:
     - `Validate user Input` → `Check For Input Type`

7. **Article path: send “please wait” message**
   - Node: **Telegram**
   - Name: “Notify User”
   - Operation: sendMessage
   - Text: “Understood. Please give me a moment to complete that.”
   - chatId expression: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
   - Parse mode: HTML
   - Connect:
     - Switch output for `Article_Request` → `Notify User`

8. **Article path: BrowserAct extraction**
   - Node: **BrowserAct**
   - Name: “Get Article Data from BrowserAct”
   - Type: `WORKFLOW`
   - Workflow ID: set to your BrowserAct workflow (in this workflow: `69338962262095598`)
   - Timeout: `7200`
   - Map input variable (e.g., `Target_link`) to expression: `{{$json.output.Link}}`
   - Credential: connect your **BrowserAct API** credential.
   - Connect:
     - Switch output for `Article_Request` → `Get Article Data from BrowserAct`

9. **Add LLM for scripting (OpenRouter)**
   - Node: **OpenRouter Chat Model**
   - Name: “OpenRouter”
   - Model: `anthropic/claude-haiku-4.5`
   - Credential: connect your **OpenRouter** credential.

10. **Add Structured Output Parser for scripting**
   - Node: **Structured Output Parser**
   - Name: “Structured Output”
   - Auto-fix: enabled
   - Schema example: `[{"elevenlabtext":"...","Telegram":"...","Audioname":"..."}]`

11. **Add Scriptwriter Agent**
   - Node: **AI Agent (LangChain)**
   - Name: “Write Script for ElevenLabs”
   - Text input: `Article or Story Data : {{ $json.output.string }}`
   - Prompt type: “Define”
   - System message: include:
     - narrative constraints for `elevenlabtext` (plain text, no markdown, length bounds)
     - Telegram HTML caption constraints (<900 chars, allowed tags only)
     - filename constraints (underscores, `.mp3`, no spaces)
     - strict JSON array output with one object and no double quotes inside values
   - Enable **Output Parser** (structured).
   - Connect:
     - `OpenRouter` → (ai_languageModel) → `Write Script for ElevenLabs`
     - `Structured Output` → (ai_outputParser) → `Write Script for ElevenLabs`
     - `Get Article Data from BrowserAct` → `Write Script for ElevenLabs`

12. **Add ElevenLabs TTS**
   - Node: **ElevenLabs**
   - Name: “Convert text to speech”
   - Resource: Speech / Text-to-speech
   - Text: `{{$json.output[0].elevenlabtext}}`
   - Voice: select your preferred voice (workflow uses “Liam…”)
   - Model: “Eleven Flash v2.5”
   - Language code: `en`
   - Credential: connect **ElevenLabs API** credential
   - Connect:
     - `Write Script for ElevenLabs` → `Convert text to speech`

13. **Send audio to Telegram**
   - Node: **Telegram**
   - Name: “Send an audio file To User”
   - Operation: `sendAudio`
   - Enable **Binary Data**
   - chatId: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
   - caption: `{{ $('Write Script for ElevenLabs').first().json.output[0].Telegram }}`
   - fileName: `{{ $('Write Script for ElevenLabs').first().json.output[0].Audioname }}`
   - Parse mode: HTML
   - Connect:
     - `Convert text to speech` → `Send an audio file To User`

14. **Chat/NoData fallback: Gemini + Telegram reply**
   - Node: **Google Gemini Chat Model**
   - Name: “Google Gemini”
   - Credential: Gemini/PaLM
   - Node: **AI Agent (LangChain)**
     - Name: “Chatting With User”
     - Text: `Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
     - System message: ask for a link if NoData; otherwise respond normally; output raw text only.
     - Connect `Google Gemini` → (ai_languageModel) → `Chatting With User`
   - Node: **Telegram** (sendMessage)
     - Name: “Answer the User”
     - Text: `{{$json.output}}`
     - chatId: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
     - Parse mode: HTML
   - Connect:
     - Switch output `Chat` → `Chatting With User`
     - Switch output `NoData` → `Chatting With User`
     - `Chatting With User` → `Answer the User`

15. **(Optional) Add Sticky Notes**
   - Add sticky notes for documentation, step explanations, and the BrowserAct docs links.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow converts Telegram article URLs into engaging audio documentaries via BrowserAct + LLM rewriting + ElevenLabs voiceover. Requires credentials: Telegram, BrowserAct, Google Gemini, OpenRouter, ElevenLabs. | From “Documentation” sticky note |
| Mandatory: BrowserAct API + saved template “AI Summarization & Eleven Labs Podcast Generation”. | From “Documentation” sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | `@[youtube](YuxjfB87F0E)` (YouTube ID: YuxjfB87F0E) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.