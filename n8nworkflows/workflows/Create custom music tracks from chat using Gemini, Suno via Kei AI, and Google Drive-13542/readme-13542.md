Create custom music tracks from chat using Gemini, Suno via Kei AI, and Google Drive

https://n8nworkflows.xyz/workflows/create-custom-music-tracks-from-chat-using-gemini--suno-via-kei-ai--and-google-drive-13542


# Create custom music tracks from chat using Gemini, Suno via Kei AI, and Google Drive

## 1. Workflow Overview

**Workflow name:** Suno Music Generation Chatbot  
**Purpose:** Provide an AI chat experience that collects song requirements (title, style, lyrics, negative tags), generates music via **Suno through the Kie.ai API**, then **downloads and uploads** the resulting audio tracks to **Google Drive**.

### 1.1 Chat Input & Conversation Orchestration
A chat trigger starts a Gemini-powered agent (“Music Producer Agent”) that collects required fields and can call tools (songwriting or web search) while maintaining short-term memory.

### 1.2 Output Validation & JSON Normalization
The agent’s output is checked for JSON code-fence formatting, parsed and cleaned, then normalized again using an LLM chain + structured output parser to enforce strict JSON formatting.

### 1.3 Async Music Generation (Kie.ai) + Webhook Resume
A music generation request is sent to Kie.ai with a **callback URL** (n8n resume URL). The workflow **waits** and resumes when Kie.ai calls back.

### 1.4 Results Processing, Download, and Google Drive Upload
After resume, the workflow fetches generation records, splits results into individual items, downloads each audio file, and uploads it to a configured Drive folder with timestamped filenames.

---

## 2. Block-by-Block Analysis

### Block 1 — Chat Input & Agent Orchestration

**Overview:** Receives chat messages, runs a Gemini agent to collect song parameters, and optionally calls tools to write lyrics or search existing lyrics.  
**Nodes involved:**  
- When chat message received  
- Music Producer Agent  
- Google Gemini Chat Model  
- Simple Memory  
- Songwriter  
- Google Gemini Chat Model1  
- Search songs

#### Node: When chat message received
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point that receives chat input.
- **Key config:** `responseMode: lastNode` (final chat response is taken from the last executed node in the chain).
- **Connections:** Outputs to **Music Producer Agent**.
- **Edge cases:** Webhook/chat channel configuration issues; if used in embedded chat, permissions/routing must be correct.

#### Node: Music Producer Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — main conversational orchestrator.
- **Key config choices:**
  - **System message** instructs agent to collect 4 fields: `title`, `style`, `negativeTags`, `prompt`.
  - Tool routing rules:
    - Original lyrics → call tool **“Cantautore”** (implemented here by **Songwriter** node).
    - Existing song lyrics → call tool **“search song”** (implemented here by **Search songs** node).
  - **Critical formatting constraints** for `prompt`: single line, no `\n`, no double quotes `"`, separators via space or `/`, safe for JSON embedding.
  - `hasOutputParser: true` (agent is expected to emit a structured JSON-only response once ready).
- **Inputs:**
  - Main chat input from trigger.
  - **AI Language Model** from **Google Gemini Chat Model**.
  - **Memory** from **Simple Memory**.
  - **Tools**: **Songwriter**, **Search songs**.
- **Outputs:** Main output to **is Song?**
- **Edge cases / failure types:**
  - Agent may still return extra text, markdown fences, or malformed JSON.
  - Tool calls can fail (auth/network) and derail completion.
  - User may not provide enough details; agent loops conversationally.

#### Node: Google Gemini Chat Model
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — Gemini chat LLM for agent and JSON fixer chain.
- **Key config:** Uses Google PaLM/Gemini credentials.
- **Connections:** Provides `ai_languageModel` to **Music Producer Agent** and **Fix Json Structure**.
- **Edge cases:** Invalid API key, quota limits, model availability/region restrictions.

#### Node: Simple Memory
- **Type / Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term conversation memory.
- **Key config:** `contextWindowLength: 10` (keeps last 10 interaction turns).
- **Connections:** Feeds `ai_memory` to **Music Producer Agent** and **Songwriter**.
- **Edge cases:** Memory window too short may cause the agent to forget earlier constraints; too long increases token usage.

#### Node: Songwriter
- **Type / Role:** `@n8n/n8n-nodes-langchain.agentTool` — tool that generates original lyrics.
- **Key config:**
  - Tool input text: `{{$fromAI('Prompt__User_Message_', '', 'string')}}`
  - System message: strict song structure and “lyrics only” output.
- **Model:** Uses **Google Gemini Chat Model1** via `ai_languageModel`.
- **Connections:** Exposed as `ai_tool` to **Music Producer Agent**.
- **Edge cases:**
  - Lyrics may include quotes or line breaks; the agent is instructed to remove them later, but this is a common failure point.
  - If $fromAI variables are not populated as expected, tool input may be empty.

#### Node: Google Gemini Chat Model1
- **Type / Role:** Second Gemini LLM node dedicated to lyric-writing tool.
- **Connections:** Supplies `ai_languageModel` to **Songwriter**.
- **Edge cases:** Same credential/quota issues as the main Gemini node.

#### Node: Search songs
- **Type / Role:** `n8n-nodes-gemini-search.geminiSearchToolTool` — web search tool for fetching existing song lyrics.
- **Key config:**
  - Query: `{{$fromAI('Query', '', 'string')}}`
  - Optional context flags: `Enable_URL_Context_Tool`, `Enable_Organization_Context`
  - Tool description in Italian: used to find lyrics of an existing song.
- **Credentials:** Gemini Search API credential.
- **Connections:** Exposed as `ai_tool` to **Music Producer Agent**.
- **Edge cases:**
  - Search results may contain copyrighted text; handle according to your policies and intended use.
  - Tool may return verbose/multiline output; agent must compress/sanitize to single-line lyrics.

**Sticky notes applying to this block:**
- “## Music Producer Chatbot using Gemini + Suno (via Kei AI) with Google Drive Upload …”
- “## STEP 1 - Chatbot …”
- “## MY NEW YOUTUBE CHANNEL … https://youtube.com/@n3witalia …”

---

### Block 2 — Validation & Formatting

**Overview:** Verifies the agent output is JSON code-fenced, parses it, and then re-normalizes into strict structured JSON using an LLM chain plus a structured output parser.  
**Nodes involved:**  
- is Song?  
- Parser  
- Fix Json Structure  
- Structured Output Parser  
- Google Gemini Chat Model (already described above; reused here)

#### Node: is Song?
- **Type / Role:** `n8n-nodes-base.if` — validates that the agent output starts with a JSON code block.
- **Condition:** `{{$json.output}} startsWith "```json"`
- **Connections:** True path goes to **Parser**. (No false-path output is wired.)
- **Edge cases:**
  - If the agent returns plain JSON without fences, or any other formatting, the workflow stops here because the “false” branch is not connected.
  - If `$json.output` is missing, the condition may evaluate unexpectedly.

#### Node: Parser
- **Type / Role:** `n8n-nodes-base.code` — strips markdown fences, parses JSON, and sanitizes fields.
- **Key logic:**
  - Removes ```json and ``` fences.
  - Unescapes `\\'` to `'`.
  - `JSON.parse()` then:
    - Uses `parsedData.properties || parsedData` (supports schema-like wrappers).
    - Enforces required fields: `title` and `prompt`.
    - Produces a normalized item:
      - `title` (trimmed string)
      - `style` (optional trimmed string)
      - `prompt` (trimmed string)
      - `negativeTags` (optional trimmed string)
  - Throws explicit error on parsing failure: `Errore nel parsing JSON: ...`
- **Connections:** Outputs to **Fix Json Structure**.
- **Edge cases:**
  - Any unescaped double quotes in lyrics will break parsing.
  - If agent returns JSON with trailing commas or non-JSON constructs, parsing fails.
  - Multiline lyrics can still be present; this node does not remove newline characters explicitly—only trims.

#### Node: Fix Json Structure
- **Type / Role:** `@n8n/n8n-nodes-langchain.chainLlm` — LLM-based normalization into final structured JSON.
- **Key config:**
  - Input text: `{{ JSON.stringify($json) }}` (wraps parsed fields back into a JSON string for the LLM).
  - Prompt message: “Based on the info you have... you have to transform it into a structured json”
  - `hasOutputParser: true`
- **Connections:** Outputs to **Create song**.
- **Dependencies:**
  - Uses **Google Gemini Chat Model** as the language model.
  - Uses **Structured Output Parser** to enforce schema.
- **Edge cases:**
  - If the model returns non-conforming output, the structured parser will fail.
  - If `prompt` still contains disallowed characters (quotes/newlines), Kie.ai request may fail later.

#### Node: Structured Output Parser
- **Type / Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces schema for final output.
- **Schema fields:** `title`, `style`, `prompt`, `negativeTags` as strings.
- **Connections:** Supplies `ai_outputParser` to **Fix Json Structure**.
- **Edge cases:** Schema mismatch will error; empty strings are allowed by schema but may be rejected downstream by the generation API.

**Sticky note applying to this block:**
- “## STEP2 - Validation & Formatting …”

---

### Block 3 — Song Generation (Async) via Kie.ai + Resume

**Overview:** Sends a generation request to Kie.ai (Suno wrapper), waits for asynchronous completion via webhook resume, then fetches record info using the taskId.  
**Nodes involved:**  
- Create song  
- Wait  
- Get songs

#### Node: Create song
- **Type / Role:** `n8n-nodes-base.httpRequest` — calls Kie.ai generation endpoint.
- **Endpoint:** `POST https://api.kie.ai/api/v1/generate`
- **Auth:** Bearer token via `httpBearerAuth` credential (“Kie AI”).
- **Body (JSON) key fields:**
  - `model: "V4_5PLUS"`
  - `customMode: true`
  - `instrumental: false`
  - `title: {{$json.output.title}}`
  - `prompt: {{$json.output.prompt}}`
  - `style: {{$json.output.style}}`
  - `negativeTags: {{$json.output.negativeTags}}`
  - `vocalGender: "m"`
  - Weights: `styleWeight`, `weirdnessConstraint`, `audioWeight` set to `0.65`
  - **Async callback:** `callBackUrl: {{$execution.resumeUrl}}`
- **Connections:** Outputs to **Wait**.
- **Edge cases / failure types:**
  - 401/403 if bearer token invalid.
  - 400 if prompt/title invalid (length limits, illegal characters, etc.).
  - If `$json.output.*` fields are missing (parser/LLM failure), request body may contain undefined/empty values.
  - Callback URL must be reachable publicly for Kie.ai to resume the workflow.

#### Node: Wait
- **Type / Role:** `n8n-nodes-base.wait` — pauses workflow until webhook resume.
- **Key config:** `resume: webhook`, `httpMethod: POST`
- **Connections:** On resume, continues to **Get songs**.
- **Edge cases:**
  - If n8n is not publicly accessible, Kie.ai cannot POST to resume URL.
  - Firewalls/proxies can block the resume callback.
  - If callback never arrives, execution remains waiting.

#### Node: Get songs
- **Type / Role:** `n8n-nodes-base.httpRequest` — fetches completed generation record info.
- **Endpoint:** `GET https://api.kie.ai/api/v1/generate/record-info`
- **Auth:** Bearer token via same Kie AI credential.
- **Query parameter:** `taskId = {{$('Create song').item.json.data.taskId}}`
- **Connections:** Outputs to **Get response**.
- **Edge cases:**
  - If `taskId` is missing (Create song failed), this request fails.
  - API might return “processing” state depending on timing; workflow assumes completion upon callback.

**Sticky note applying to this block:**
- “## STEP 3 - Song Generation …”

---

### Block 4 — Process Results, Download, Upload to Google Drive

**Overview:** Extracts the list of generated tracks, iterates them, downloads each audio file, and uploads it to a specific Drive folder.  
**Nodes involved:**  
- Get response  
- Split Out  
- Loop Over Items  
- Get single song  
- Upload song

#### Node: Get response
- **Type / Role:** `n8n-nodes-base.set` — extracts the array of song results from the Kie.ai response.
- **Key config:** Sets `response` (array) to: `{{$json.data.response.sunoData}}`
- **Connections:** Outputs to **Split Out**.
- **Edge cases:** If Kie.ai response shape changes or `sunoData` is missing, downstream splitting will fail or produce no items.

#### Node: Split Out
- **Type / Role:** `n8n-nodes-base.splitOut` — converts an array field into individual items.
- **Key config:** `fieldToSplitOut: "response"`
- **Connections:** Outputs to **Loop Over Items**.
- **Edge cases:** If `response` is not an array, node errors.

#### Node: Loop Over Items
- **Type / Role:** `n8n-nodes-base.splitInBatches` — iterates items in batches (looping construct).
- **Key config:** Default batch settings (not customized).
- **Connections:**
  - “Next batch” output is connected to **Get single song** (index 1 in JSON wiring).
  - After **Upload song**, it loops back into **Loop Over Items** to continue.
- **Edge cases:**
  - If no items, loop may end immediately.
  - Large result sets could require careful batch sizing (not configured here).

#### Node: Get single song
- **Type / Role:** `n8n-nodes-base.httpRequest` — downloads the audio file from a URL per track.
- **URL:** `{{$json.sourceStreamAudioUrl}}`
- **Connections:** Outputs to **Upload song**.
- **Edge cases:**
  - If URL is missing/expired, download fails.
  - Depending on n8n HTTP Request settings, you may need “Download”/binary response options; if misconfigured, Drive upload won’t have binary data.

#### Node: Upload song
- **Type / Role:** `n8n-nodes-base.googleDrive` — uploads downloaded audio to Drive.
- **Credentials:** Google Drive OAuth2 (“Google Drive account (n3w.it)”).
- **Target folder:** Folder ID `1iT2rs_A22QESeiTMH1cyk-zIO5xM8OYw` (named “Kie AI” in cached selection).
- **Filename expression:** `{{$now.format('yyyyLLddHHiiss')}}_{{ $binary.data.fileName }}`
- **Connections:** Outputs back to **Loop Over Items** for the next item.
- **Edge cases:**
  - OAuth token expired / missing scopes → 401.
  - If binary field name isn’t `data` or `fileName` missing, naming/upload fails.
  - Folder permissions: must have write access.

**Sticky note applying to this block:**
- “## STEP 4 - Process Results … Upload songs to Google Drive”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat entry point | — | Music Producer Agent | ## Music Producer Chatbot using Gemini + Suno (via Kei AI) with Google Drive Upload …; ## STEP 1 - Chatbot …; ## MY NEW YOUTUBE CHANNEL … https://youtube.com/@n3witalia … |
| Music Producer Agent | `@n8n/n8n-nodes-langchain.agent` | Conversational collection + tool orchestration | When chat message received; Google Gemini Chat Model (LM); Simple Memory (memory); Songwriter/Search songs (tools) | is Song? | ## Music Producer Chatbot using Gemini + Suno (via Kei AI) with Google Drive Upload …; ## STEP 1 - Chatbot … |
| Google Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Primary LLM for agent + JSON fixer chain | — | (ai_languageModel) Music Producer Agent; Fix Json Structure | ## Music Producer Chatbot using Gemini + Suno (via Kei AI) with Google Drive Upload …; ## STEP 1 - Chatbot … |
| Simple Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Conversation memory window | — | (ai_memory) Music Producer Agent; Songwriter | ## STEP 1 - Chatbot … |
| Songwriter | `@n8n/n8n-nodes-langchain.agentTool` | Tool: generate original lyrics | Google Gemini Chat Model1 (LM); Simple Memory (memory) | (ai_tool) Music Producer Agent | ## STEP 1 - Chatbot … |
| Google Gemini Chat Model1 | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM for lyric-writing tool | — | (ai_languageModel) Songwriter | ## STEP 1 - Chatbot … |
| Search songs | `n8n-nodes-gemini-search.geminiSearchToolTool` | Tool: search web for existing lyrics | — | (ai_tool) Music Producer Agent | ## STEP 1 - Chatbot … |
| is Song? | `n8n-nodes-base.if` | Validate output starts with ```json | Music Producer Agent | Parser (true branch) | ## STEP2 - Validation & Formatting … |
| Parser | `n8n-nodes-base.code` | Strip fences + JSON.parse + clean fields | is Song? | Fix Json Structure | ## STEP2 - Validation & Formatting … |
| Fix Json Structure | `@n8n/n8n-nodes-langchain.chainLlm` | Normalize into strict structured JSON | Parser; Google Gemini Chat Model (LM); Structured Output Parser (parser) | Create song | ## STEP2 - Validation & Formatting … |
| Structured Output Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce schema (title/style/prompt/negativeTags) | — | (ai_outputParser) Fix Json Structure | ## STEP2 - Validation & Formatting … |
| Create song | `n8n-nodes-base.httpRequest` | Call Kie.ai generate endpoint (async) | Fix Json Structure | Wait | ## STEP 3 - Song Generation … |
| Wait | `n8n-nodes-base.wait` | Pause until webhook resume (callback) | Create song | Get songs | ## STEP 3 - Song Generation … |
| Get songs | `n8n-nodes-base.httpRequest` | Fetch record info by taskId | Wait | Get response | ## STEP 3 - Song Generation … |
| Get response | `n8n-nodes-base.set` | Extract `sunoData` array into `response` | Get songs | Split Out | ## STEP 4 - Process Results … Upload songs to Google Drive |
| Split Out | `n8n-nodes-base.splitOut` | Split array into items | Get response | Loop Over Items | ## STEP 4 - Process Results … Upload songs to Google Drive |
| Loop Over Items | `n8n-nodes-base.splitInBatches` | Iterate each generated track | Split Out; Upload song (loop-back) | Get single song | ## STEP 4 - Process Results … Upload songs to Google Drive |
| Get single song | `n8n-nodes-base.httpRequest` | Download audio file from track URL | Loop Over Items | Upload song | ## STEP 4 - Process Results … Upload songs to Google Drive |
| Upload song | `n8n-nodes-base.googleDrive` | Upload audio to Drive folder | Get single song | Loop Over Items | ## STEP 4 - Process Results … Upload songs to Google Drive |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation/comment | — | — |  |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation/comment | — | — |  |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation/comment | — | — |  |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation/comment | — | — |  |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation/comment | — | — |  |
| Sticky Note9 | `n8n-nodes-base.stickyNote` | Documentation/comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Suno Music Generation Chatbot”**.

2. **Add Chat Trigger**
   - Node: **When chat message received**
   - Type: *LangChain → Chat Trigger*
   - Set **Options → Response Mode** = `lastNode`

3. **Add Memory**
   - Node: **Simple Memory**
   - Type: *LangChain → Memory Buffer Window*
   - Set **Context Window Length** = `10`

4. **Add Gemini model for main agent**
   - Node: **Google Gemini Chat Model**
   - Type: *LangChain → Google Gemini Chat Model*
   - Create/attach **Google Gemini (PaLM) API credentials** (API key)

5. **Add Tools**
   1) **Songwriter tool**
   - Node: **Songwriter**
   - Type: *LangChain → Agent Tool*
   - Set **Tool Description** to match “singer-songwriter generates lyrics”
   - Set **Text** to: `{{$fromAI('Prompt__User_Message_', '', 'string')}}`
   - Paste the provided system message (song structure rules)

   2) **Gemini model for Songwriter**
   - Node: **Google Gemini Chat Model1**
   - Type: *LangChain → Google Gemini Chat Model*
   - Use same Gemini credentials (or separate)
   - Connect `ai_languageModel` → **Songwriter**

   3) **Search tool**
   - Node: **Search songs**
   - Type: *Gemini Search Tool*
   - Set **Query**: `{{$fromAI('Query', '', 'string')}}`
   - Configure Gemini Search credentials (API key/token as required by the node)

6. **Add the main Agent**
   - Node: **Music Producer Agent**
   - Type: *LangChain → Agent*
   - Paste the system message that:
     - Collects `title`, `style`, `negativeTags`, `prompt`
     - Calls **Songwriter** for original lyrics, **Search songs** for existing lyrics
     - Enforces prompt formatting (single line, no quotes)
     - Outputs **only** JSON
   - Enable/keep **Has Output Parser** (as in the workflow)
   - Connect:
     - Chat Trigger (main) → Agent (main)
     - **Google Gemini Chat Model** → Agent (`ai_languageModel`)
     - **Simple Memory** → Agent (`ai_memory`)
     - **Songwriter** → Agent (`ai_tool`)
     - **Search songs** → Agent (`ai_tool`)

7. **Add validation IF**
   - Node: **is Song?**
   - Type: *IF*
   - Condition: String → `{{$json.output}}` **startsWith** ` ```json `
   - Connect Agent (main) → IF (main)

8. **Add parsing Code node**
   - Node: **Parser**
   - Type: *Code*
   - Paste the JS parsing/cleaning logic (strip code fences, JSON.parse, enforce `title` and `prompt`)
   - Connect IF (true) → Parser

9. **Add structured output enforcement**
   1) Node: **Structured Output Parser**
   - Type: *LangChain → Structured Output Parser*
   - Schema (manual): object with string fields `title`, `style`, `prompt`, `negativeTags`

   2) Node: **Fix Json Structure**
   - Type: *LangChain → LLM Chain*
   - Input text: `{{ JSON.stringify($json) }}`
   - Provide a prompt instructing transformation into structured JSON
   - Enable **Has Output Parser**
   - Connect:
     - Parser → Fix Json Structure (main)
     - **Google Gemini Chat Model** → Fix Json Structure (`ai_languageModel`)
     - **Structured Output Parser** → Fix Json Structure (`ai_outputParser`)

10. **Add Kie.ai generation request**
   - Node: **Create song**
   - Type: *HTTP Request*
   - Method: `POST`
   - URL: `https://api.kie.ai/api/v1/generate`
   - Auth: **HTTP Bearer Auth** credential (create credential “Kie AI” with your token)
   - Body type: JSON, include:
     - `callBackUrl: {{$execution.resumeUrl}}`
     - `title: {{$json.output.title}}`
     - `prompt: {{$json.output.prompt}}`
     - `style: {{$json.output.style}}`
     - `negativeTags: {{$json.output.negativeTags}}`
     - plus your chosen model/weights (e.g., `V4_5PLUS`, 0.65 weights)
   - Connect Fix Json Structure → Create song

11. **Add Wait (webhook resume)**
   - Node: **Wait**
   - Type: *Wait*
   - Resume: `webhook`
   - HTTP Method: `POST`
   - Connect Create song → Wait
   - Ensure your n8n instance is publicly reachable so Kie.ai can call the resume URL.

12. **Add “Get songs” record-info request**
   - Node: **Get songs**
   - Type: *HTTP Request*
   - Method: GET (default)
   - URL: `https://api.kie.ai/api/v1/generate/record-info`
   - Auth: same **Kie AI** bearer credential
   - Query param `taskId`: `{{$('Create song').item.json.data.taskId}}`
   - Connect Wait → Get songs

13. **Extract array results**
   - Node: **Get response** (Set)
   - Set field `response` (type array) to: `{{$json.data.response.sunoData}}`
   - Connect Get songs → Get response

14. **Split and loop**
   - Node: **Split Out**
   - Type: *Split Out*
   - Field to split: `response`
   - Connect Get response → Split Out

   - Node: **Loop Over Items**
   - Type: *Split In Batches*
   - Connect Split Out → Loop Over Items

15. **Download each track**
   - Node: **Get single song**
   - Type: *HTTP Request*
   - URL: `{{$json.sourceStreamAudioUrl}}`
   - Configure response as **binary download** if needed in your n8n version (so Drive node receives binary).
   - Connect Loop Over Items → Get single song

16. **Upload to Google Drive**
   - Node: **Upload song**
   - Type: *Google Drive*
   - Operation: Upload (file)
   - Credentials: Google Drive OAuth2
   - Folder: pick/enter folder ID `1iT2rs_A22QESeiTMH1cyk-zIO5xM8OYw` (or your own)
   - Name: `{{$now.format('yyyyLLddHHiiss')}}_{{ $binary.data.fileName }}`
   - Connect Get single song → Upload song
   - Connect Upload song → Loop Over Items (to continue batches)

17. **Activate the workflow** and test through the chat trigger UI. Verify:
   - Agent returns JSON as expected
   - Kie.ai callback successfully resumes execution
   - Audio downloads produce binary data
   - Files appear in the Drive folder

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Kie.ai API used for music generation (Suno via Kei AI). Callback URL required for async completion. | https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5 |
| Channel link and cover image reference from sticky note. | https://youtube.com/@n3witalia ; image: https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |
| Setup prerequisites listed in sticky note: Gemini, Gemini Search, Kie.ai bearer token, Google Drive OAuth2; expose Wait webhook publicly. | Applies to overall deployment |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.