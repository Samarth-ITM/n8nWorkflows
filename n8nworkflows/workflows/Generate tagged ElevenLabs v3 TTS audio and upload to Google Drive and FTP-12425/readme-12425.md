Generate tagged ElevenLabs v3 TTS audio and upload to Google Drive and FTP

https://n8nworkflows.xyz/workflows/generate-tagged-elevenlabs-v3-tts-audio-and-upload-to-google-drive-and-ftp-12425


# Generate tagged ElevenLabs v3 TTS audio and upload to Google Drive and FTP

## 1. Workflow Overview

**Purpose:**  
This workflow turns raw text into **highly expressive ElevenLabs v3 (alpha) speech** by first **injecting ElevenLabs v3 audio tags** using an AI agent (Claude Sonnet 4.5 + Context7 reference), then **generates an MP3** with ElevenLabs, and finally **uploads the audio to Google Drive and an FTP server** (e.g., BunnyCDN).

**Primary use cases:**
- Producing tagged narration for horror/storytelling (as explicitly instructed in the agent system message)
- Building an automation endpoint to generate and distribute voice assets from text
- Supporting multiple input methods: manual run, webhook API, and n8n chat trigger

### 1.1 Input Reception (3 entry points)
- Manual execution → fixed text from a Set node  
- Webhook POST → external systems send text  
- Chat trigger → conversational input within n8n chat

### 1.2 AI Tagging (LLM agent + official reference tool)
- An agent takes input text and returns the **same text enriched with ElevenLabs v3 audio tags**, using:
  - **Anthropic Claude Sonnet 4.5** as language model
  - **Context7 MCP tool** to consult the official tag guide (per instructions)

### 1.3 Text-to-Speech (ElevenLabs v3)
- Tagged text is sent to ElevenLabs “speech” endpoint using **eleven_v3 (alpha)** + configured voice settings.

### 1.4 Distribution (Drive + FTP)
- The produced MP3 (binary) is uploaded to:
  - A specific Google Drive folder
  - A configured FTP path

---

## 2. Block-by-Block Analysis

### Block 1 — Multiple Inputs / Triggers
**Overview:**  
Provides three ways to feed text into the workflow: manual execution (via Set node), webhook POST, or chat message input. All converge into the Audio Tagger Agent.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set text
- Webhook
- When chat message received

#### Node: **When clicking ‘Execute workflow’**
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — starts execution from the UI.
- **Configuration choices:** No parameters; standard manual trigger.
- **Connections:**
  - **Output →** Set text
- **Edge cases / failures:** None typical (only runs when user executes).
- **Version notes:** typeVersion 1.

#### Node: **Set text**
- **Type / role:** Set node (`n8n-nodes-base.set`) — injects a default text field for manual runs.
- **Configuration choices:**
  - Creates a string field named **`testo`** with value **`YOUR_TEXT`**.
  - Intended for you to replace `YOUR_TEXT` with your actual content.
- **Key variables/expressions:**
  - Output field: `$json.testo`
- **Connections:**
  - **Input ←** When clicking ‘Execute workflow’
  - **Output →** Audio Tagger Agent
- **Edge cases / failures:**
  - If you forget to replace `YOUR_TEXT`, you’ll synthesize placeholder content.
- **Version notes:** typeVersion 3.4.

#### Node: **Webhook**
- **Type / role:** Webhook (`n8n-nodes-base.webhook`) — external HTTP entry point.
- **Configuration choices:**
  - **Method:** POST
  - **Path:** `b2e00463-dec7-4dbd-9ce6-f323ef6876d0`
- **Connections:**
  - **Output →** Audio Tagger Agent
- **Edge cases / failures:**
  - Payload may not include the expected text field; the agent relies on `$json.testo || $json.chatInput` (see agent node), so you likely need to send `{"testo":"..."}` unless you modify expressions.
  - Webhook authentication/rate limits are not configured here (depends on n8n instance settings).
- **Version notes:** typeVersion 2.1.

#### Node: **When chat message received**
- **Type / role:** Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) — receives messages from n8n’s chat interface/integrations.
- **Configuration choices:**
  - Uses internal webhookId `2f67f228-de90-412d-9588-f0815d7e56c6`
  - Default options (none specified)
- **Connections:**
  - **Output →** Audio Tagger Agent
- **Edge cases / failures:**
  - If chat input field naming differs from `chatInput` in your environment, the agent may not pick it up.
- **Version notes:** typeVersion 1.4.

**Sticky note covering this block (applies to these nodes):**  
“## STEP 1 - Multiple Inputs  
The workflow accepts text input through multiple triggers - manual execution via "Set text" node (In this case set the manual text in this node), webhook POST requests, or chat message inputs. This text is passed to the Audio Tagger Agent.”

---

### Block 2 — AI Audio Tagging (Agent + LLM + Context7)
**Overview:**  
Transforms the input text into *the same text* augmented with ElevenLabs v3 audio tags. The agent is explicitly instructed not to paraphrase and to consult Context7 for correct tag usage.

**Nodes involved:**
- Audio Tagger Agent
- Anthropic Chat Model
- Context7

#### Node: **Anthropic Chat Model**
- **Type / role:** Anthropic chat LLM connector (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) — provides the language model used by the agent.
- **Configuration choices:**
  - **Model:** `claude-sonnet-4-5-20250929` (display: “Claude Sonnet 4.5”)
  - Default options
- **Credentials:** “Anthropic account”
- **Connections:**
  - **Output (ai_languageModel) →** Audio Tagger Agent
- **Edge cases / failures:**
  - Invalid/expired Anthropic API key
  - Model availability changes (named model might be unavailable in some regions/accounts)
  - Rate limits/timeouts for large prompts (system message is long)
- **Version notes:** typeVersion 1.3.

#### Node: **Context7**
- **Type / role:** MCP Client Tool (`@n8n/n8n-nodes-langchain.mcpClientTool`) — tool the agent can call to reference external docs.
- **Configuration choices:**
  - **Endpoint URL:** `https://mcp.context7.com/mcp`
  - **Authentication:** Header Auth (via n8n credential “httpHeaderAuth”)
- **Credentials:** “Context7”
- **Connections:**
  - **Output (ai_tool) →** Audio Tagger Agent
- **Edge cases / failures:**
  - Missing/incorrect `CONTEXT7_API_KEY` header → tool calls fail
  - Network/DNS/TLS issues reaching Context7 MCP endpoint
- **Version notes:** typeVersion 1.2.

#### Node: **Audio Tagger Agent**
- **Type / role:** Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates LLM + tool usage and returns tagged text.
- **Configuration choices (interpreted):**
  - **Input text expression:** `{{ $json.testo || $json.chatInput }}`
    - Prioritizes `testo` (manual/webhook) and falls back to `chatInput` (chat trigger).
  - **System message:** A detailed specification:
    - Keep same language, no paraphrasing, only insert tags + minimal punctuation
    - Use tags for pauses, rhythm, emphasis, whispering, emotion, breathing, etc.
    - Consult Context7 to ensure correct ElevenLabs v3 tag syntax
    - Horror-film storytelling emphasis
  - **Prompt type:** “define” (node’s promptType)
- **Connections:**
  - **Inputs:**
    - Main input from: Set text, Webhook, When chat message received
    - LLM input from: Anthropic Chat Model (ai_languageModel)
    - Tool input from: Context7 (ai_tool)
  - **Main output →** Convert text to speech
- **Outputs / data shape expectations:**
  - Downstream node expects tagged text in **`$json.output`** (as used by ElevenLabs node).
- **Edge cases / failures:**
  - If neither `testo` nor `chatInput` exists → agent receives `null/undefined` → poor output or failure.
  - Agent output field naming can differ depending on node version/config; here downstream assumes `output`. If agent returns e.g. `text`, you must adjust the ElevenLabs node expression.
  - Overuse/incorrect tags could reduce TTS quality or cause ElevenLabs parsing issues; Context7 is meant to mitigate this.
- **Version notes:** typeVersion 3.1.

**Sticky note covering this block (applies to these nodes):**  
“## STEP 2 - AI Agent  
AI-Powered Audio Tagging: The Audio Tagger Agent uses Claude Sonnet 4.5 to analyze the input text and intelligently insert ElevenLabs v3 audio tags with MCP Context7 validation  
- Get your FREE [Context7 API Key](https://context7.com/sign-in?redirect_url=%2Fdashboard)  
Set MCP connection:  
"headers": {  
      "CONTEXT7_API_KEY": "YOUR_API_KEY"  
    }”

---

### Block 3 — ElevenLabs v3 Text-to-Speech
**Overview:**  
Converts the agent-produced tagged text into speech using ElevenLabs “eleven_v3 (alpha)” model and a specific voice ID with tuned voice settings.

**Nodes involved:**
- Convert text to speech

#### Node: **Convert text to speech**
- **Type / role:** ElevenLabs node (`@elevenlabs/n8n-nodes-elevenlabs.elevenLabs`) — generates audio from text.
- **Configuration choices:**
  - **Resource:** speech
  - **Text:** `{{ $json.output }}` (expects agent output in `output`)
  - **Voice:** ID `TX3LPaxmHKxFdv7VOQHJ` (configured as an expression in the “id” mode)
  - **Model:** `eleven_v3` (alpha)
  - **Voice settings (JSON string):**
    - `stability: 1`
    - `similarity_boost: 1`
    - `style: 0.8`
    - `use_speaker_boost: true`
    - `speed: 1`
- **Credentials:** “ElevenLabs account”
- **Connections:**
  - **Input ←** Audio Tagger Agent
  - **Output →** Upload file to Drive, Upload to FTP (fan-out)
- **Outputs / data shape:**
  - Produces binary audio in `$binary.data` (expected by downstream upload nodes), including `fileName`.
- **Edge cases / failures:**
  - Wrong/expired ElevenLabs API key
  - Voice ID not available to your account
  - Eleven v3 alpha model availability changes
  - If `$json.output` is empty or too long, request may fail
  - If tags are malformed, audio generation could error or sound incorrect
- **Version notes:** typeVersion 1.

**Sticky note covering this block:**  
“## STEP 3 - Text-to-Speech  
The tagged text is sent to ElevenLabs’ v3 (alpha) model, which converts it into speech using a specific voice with customized voice settings including stability, similarity boost, style, speaker boost, and speed controls  
- Install Elevenlabs Community node  
- Set up your [ElevenLabs API Key](https://try.elevenlabs.io/ahkbf00hocnu)  
- Get Voice ID”

---

### Block 4 — Upload MP3 to Google Drive and FTP
**Overview:**  
Takes the MP3 binary produced by ElevenLabs and uploads it to a specified Google Drive folder and to an FTP server path.

**Nodes involved:**
- Upload file to Drive
- Upload to FTP

#### Node: **Upload file to Drive**
- **Type / role:** Google Drive node (`n8n-nodes-base.googleDrive`) — uploads a file to Drive.
- **Configuration choices:**
  - **Operation:** upload (implied by “Upload file” behavior)
  - **File name:** `{{ $binary.data.fileName }}`
  - **Drive:** “My Drive”
  - **Folder ID:** `1J7_S0zPgUukmKWJglMepvf429vfQsidL` (cached name “Elevenlabs”)
- **Credentials:** “Google Drive account (n3w.it)” (OAuth2)
- **Connections:**
  - **Input ←** Convert text to speech
  - **Output:** none (terminal branch)
- **Edge cases / failures:**
  - OAuth token expired/insufficient scopes
  - Folder ID not accessible to the authenticated user
  - If `$binary.data.fileName` missing, upload may fail or use a default name
- **Version notes:** typeVersion 3.

#### Node: **Upload to FTP**
- **Type / role:** FTP node (`n8n-nodes-base.ftp`) — uploads the MP3 to an FTP server.
- **Configuration choices:**
  - **Operation:** upload
  - **Remote path expression:** `=/YOUR_PATH/{{ $binary.data.fileName }}`
    - You must replace `/YOUR_PATH/` with your real destination directory.
- **Credentials:** “FTP BunnyCDN”
- **Connections:**
  - **Input ←** Convert text to speech
  - **Output:** none (terminal branch)
- **Edge cases / failures:**
  - Wrong FTP host/user/password or missing permissions
  - Remote path not existing (depending on server, it may not auto-create directories)
  - Binary missing or `fileName` missing
- **Version notes:** typeVersion 1.

**Sticky note covering this block:**  
“## STEP 4 - Upload Mp3  
- Configure Google Drive OAuth2 credentials with access to the target folder  
- Set up FTP credentials for BunnyCDN or alternative storage”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set text | ## STEP 1 - Multiple Inputs; The workflow accepts text input through multiple triggers - manual execution via "Set text" node (In this case set the manual text in this node), webhook POST requests, or chat message inputs. This text is passed to the Audio Tagger Agent. |
| Set text | Set | Define manual text (`testo`) | When clicking ‘Execute workflow’ | Audio Tagger Agent | ## STEP 1 - Multiple Inputs; The workflow accepts text input through multiple triggers - manual execution via "Set text" node (In this case set the manual text in this node), webhook POST requests, or chat message inputs. This text is passed to the Audio Tagger Agent. |
| Webhook | Webhook | HTTP POST entry point | — | Audio Tagger Agent | ## STEP 1 - Multiple Inputs; The workflow accepts text input through multiple triggers - manual execution via "Set text" node (In this case set the manual text in this node), webhook POST requests, or chat message inputs. This text is passed to the Audio Tagger Agent. |
| When chat message received | Chat Trigger (LangChain) | Chat entry point | — | Audio Tagger Agent | ## STEP 1 - Multiple Inputs; The workflow accepts text input through multiple triggers - manual execution via "Set text" node (In this case set the manual text in this node), webhook POST requests, or chat message inputs. This text is passed to the Audio Tagger Agent. |
| Anthropic Chat Model | LangChain Anthropic Chat Model | LLM backend for agent | — | Audio Tagger Agent | ## STEP 2 - AI Agent; AI-Powered Audio Tagging… Get your FREE [Context7 API Key](https://context7.com/sign-in?redirect_url=%2Fdashboard) … header `CONTEXT7_API_KEY`. |
| Context7 | LangChain MCP Client Tool | Tool access to ElevenLabs v3 tag reference | — | Audio Tagger Agent | ## STEP 2 - AI Agent; AI-Powered Audio Tagging… Get your FREE [Context7 API Key](https://context7.com/sign-in?redirect_url=%2Fdashboard) … header `CONTEXT7_API_KEY`. |
| Audio Tagger Agent | LangChain Agent | Insert ElevenLabs v3 audio tags into text | Set text; Webhook; When chat message received; (AI) Anthropic Chat Model; (Tool) Context7 | Convert text to speech | ## STEP 2 - AI Agent; AI-Powered Audio Tagging… Get your FREE [Context7 API Key](https://context7.com/sign-in?redirect_url=%2Fdashboard) … header `CONTEXT7_API_KEY`. |
| Convert text to speech | ElevenLabs | Generate MP3 from tagged text (eleven_v3 alpha) | Audio Tagger Agent | Upload file to Drive; Upload to FTP | ## STEP 3 - Text-to-Speech; Install Elevenlabs Community node; [ElevenLabs API Key](https://try.elevenlabs.io/ahkbf00hocnu); Get Voice ID |
| Upload file to Drive | Google Drive | Upload MP3 to Drive folder | Convert text to speech | — | ## STEP 4 - Upload Mp3; Configure Google Drive OAuth2 credentials… Set up FTP credentials… |
| Upload to FTP | FTP | Upload MP3 to FTP path | Convert text to speech | — | ## STEP 4 - Upload Mp3; Configure Google Drive OAuth2 credentials… Set up FTP credentials… |
| Sticky Note | Sticky Note | Commentary | — | — | (This node is itself a note) ## ElevenLabs v3 AI Audio Tagger Agent with TTS and upload to FTP & Drive … (full note content in section 5) |
| Sticky Note1 | Sticky Note | Commentary | — | — | (This node is itself a note) ## STEP 1 - Multiple Inputs … |
| Sticky Note2 | Sticky Note | Commentary | — | — | (This node is itself a note) ## STEP 2 - AI Agent … [Context7 API Key](https://context7.com/sign-in?redirect_url=%2Fdashboard) … |
| Sticky Note3 | Sticky Note | Commentary | — | — | (This node is itself a note) ## STEP 3 - Text-to-Speech … [ElevenLabs API Key](https://try.elevenlabs.io/ahkbf00hocnu) … |
| Sticky Note4 | Sticky Note | Commentary | — | — | (This node is itself a note) ## STEP 4 - Upload Mp3 … |
| Sticky Note5 | Sticky Note | Commentary / branding | — | — | (This node is itself a note) [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia) … image link preserved |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Elevenlabs v3 Audio Tags Agent”** (or your preferred name)

2) **Add input trigger (manual path)**
- Add node: **Manual Trigger**
  - Name: “When clicking ‘Execute workflow’”
- Add node: **Set**
  - Name: “Set text”
  - Add a string field:
    - Field name: `testo`
    - Value: `YOUR_TEXT` (replace with your actual text)
- Connect: Manual Trigger → Set text

3) **Add additional input trigger (Webhook)**
- Add node: **Webhook**
  - Method: **POST**
  - Path: choose a unique path (you can reuse the UUID-style path if desired)
- Connect: Webhook → Audio Tagger Agent (you’ll add the agent next)
- Payload recommendation: send JSON like:
  - `{"testo":"your text here"}`  
  (or modify the agent’s input expression to match your payload)

4) **Add additional input trigger (Chat)**
- Add node: **When chat message received** (LangChain Chat Trigger)
  - Leave defaults unless your chat integration requires changes
- Connect: Chat Trigger → Audio Tagger Agent

5) **Add the AI model (Anthropic)**
- Add node: **Anthropic Chat Model** (LangChain)
  - Select model: **Claude Sonnet 4.5** (or closest available)
- Create/attach credentials:
  - **Anthropic API** credential with your API key
- Connect: Anthropic Chat Model (ai_languageModel output) → Audio Tagger Agent (ai_languageModel input)

6) **Add Context7 MCP tool**
- Add node: **MCP Client Tool**
  - Name: “Context7”
  - Endpoint URL: `https://mcp.context7.com/mcp`
  - Auth: **Header Auth**
- Create/attach credentials (Header Auth):
  - Set header: `CONTEXT7_API_KEY: YOUR_API_KEY`
  - Get key: https://context7.com/sign-in?redirect_url=%2Fdashboard
- Connect: Context7 (ai_tool output) → Audio Tagger Agent (ai_tool input)

7) **Add the Agent**
- Add node: **Agent** (LangChain)
  - Name: “Audio Tagger Agent”
  - **Text input expression:** `{{ $json.testo || $json.chatInput }}`
  - Set the **System Message** to match the behavior you want (copy the intent from the workflow):
    - “You are an Audio Tagger for ElevenLabs (model v3)… return ONLY tagged text… use Context7… horror film story…”
  - Ensure the agent’s output field is accessible downstream (this workflow assumes `output`).
- Connect inputs:
  - Set text → Audio Tagger Agent
  - Webhook → Audio Tagger Agent
  - When chat message received → Audio Tagger Agent

8) **Add ElevenLabs Text-to-Speech**
- Install **ElevenLabs community node** if required by your n8n environment.
- Add node: **ElevenLabs**
  - Resource: **speech**
  - Model: **eleven_v3 (alpha)** (or v3 if later promoted)
  - Text expression: `{{ $json.output }}` (adjust if your agent output uses a different field)
  - Voice: set your **Voice ID**
  - Voice settings JSON (example from workflow):
    ```json
    {
      "stability": 1,
      "similarity_boost": 1,
      "style": 0.8,
      "use_speaker_boost": true,
      "speed": 1
    }
    ```
- Create/attach credentials:
  - ElevenLabs API key (link provided in notes): https://try.elevenlabs.io/ahkbf00hocnu
- Connect: Audio Tagger Agent → Convert text to speech

9) **Add Google Drive upload**
- Add node: **Google Drive**
  - Operation: upload file
  - File name: `{{ $binary.data.fileName }}`
  - Drive: “My Drive”
  - Folder: choose your target folder (store the folder ID)
- Create/attach credentials:
  - Google Drive OAuth2 (ensure it has permission to write into the chosen folder)
- Connect: Convert text to speech → Upload file to Drive

10) **Add FTP upload**
- Add node: **FTP**
  - Operation: upload
  - Path: `/YOUR_PATH/{{ $binary.data.fileName }}`
    - Replace `/YOUR_PATH/` with your actual FTP destination folder
- Create/attach credentials:
  - FTP host/user/password (e.g., BunnyCDN FTP)
- Connect: Convert text to speech → Upload to FTP

11) **Test**
- Manual run:
  - Put real text in “Set text”
  - Execute workflow
- Webhook test:
  - POST JSON `{"testo":"..."}` to the webhook URL
- Verify:
  - Drive file appears in the target folder
  - FTP file exists at the expected path

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “ElevenLabs v3 AI Audio Tagger Agent with TTS and upload to FTP & Drive… Setup steps: configure Anthropic, ElevenLabs, Google Drive OAuth2, FTP, Context7; update Set text; adjust FTP path and Drive folder; confirm webhook path; modify voice parameters.” | Workflow overview note (Sticky Note) |
| Get your FREE Context7 API Key | https://context7.com/sign-in?redirect_url=%2Fdashboard |
| Context7 MCP header format: `CONTEXT7_API_KEY: YOUR_API_KEY` | Used by the Context7 credential (Header Auth) |
| ElevenLabs API Key link (as provided) | https://try.elevenlabs.io/ahkbf00hocnu |
| Subscribe to YouTube channel | https://youtube.com/@n3witalia |
| YouTube cover image link (as provided) | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |