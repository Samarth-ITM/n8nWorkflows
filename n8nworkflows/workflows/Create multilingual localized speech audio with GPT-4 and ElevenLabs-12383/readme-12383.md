Create multilingual localized speech audio with GPT-4 and ElevenLabs

https://n8nworkflows.xyz/workflows/create-multilingual-localized-speech-audio-with-gpt-4-and-elevenlabs-12383


# Create multilingual localized speech audio with GPT-4 and ElevenLabs

## 1. Workflow Overview

**Purpose:**  
This workflow takes a source **Chinese text**, generates **culturally localized translations** for multiple target languages using **GPT-4.1-mini**, then creates **speech audio files** for each language via **ElevenLabs Text-to-Speech**, returning per-language audio binaries plus metadata.

**Typical use cases:** global marketing assets, multilingual e-learning narration, content studio localization pipelines.

### 1.1 Controlled Input & Configuration
Manual execution with a single configuration node that stores the source text, target language list, and ElevenLabs credentials/voice.

### 1.2 Cultural Localization (LLM + Structured Output)
A LangChain Agent (“Localization Agent”) uses an OpenAI chat model and a structured output parser to produce a JSON array of translations (language, languageCode, translatedText, culturalNotes).

### 1.3 Prepare Iteration List
The translations array is mapped into a `languages` array for iteration.

### 1.4 Multilingual Audio Generation Loop (Batch Loop → ElevenLabs → Post-process → Repeat)
A `SplitInBatches` loop iterates over languages, calls ElevenLabs TTS for each item, processes the binary response, then loops back until all languages are processed.

### 1.5 Voice/Speech Optimization Tools (Defined, but not wired into the main loop output)
Two “agent tools” exist (speech text optimization and voice parameter selection) backed by OpenAI + structured parsing, but they are **only attached as tools to agents** and **not explicitly used** in the main node-to-node data path. As a result, fields required by the ElevenLabs request (e.g., `stability`) may be missing unless the agent injects them into each translation item (see edge cases).

---

## 2. Block-by-Block Analysis

### Block 1 — Controlled Workflow Initiation & Configuration
**Overview:** Starts the workflow manually and defines all runtime configuration values (source text, target languages, ElevenLabs credentials).  
**Nodes involved:** `Start Workflow`, `Workflow Configuration`

#### Node: Start Workflow
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration:** No parameters; execution starts when user clicks “Execute workflow”.
- **Outputs:** Sends a single empty item to `Workflow Configuration`.
- **Failure modes:** None typical (only platform-level issues).

#### Node: Workflow Configuration
- **Type / role:** Set (`n8n-nodes-base.set`) — central configuration holder.
- **Key fields set:**
  - `chineseText`: placeholder string for the Chinese source content.
  - `targetLanguages`: `"English,Spanish,French,German,Japanese"` (comma-separated names).
  - `elevenLabsApiKey`: placeholder for ElevenLabs API key.
  - `elevenLabsVoiceId`: placeholder for ElevenLabs voice ID.
- **Expressions/variables used downstream:**
  - `{{$json.chineseText}}` (LLM input)
  - `{{$json.targetLanguages}}` (LLM system message)
  - `{{$('Workflow Configuration').first().json.elevenLabsApiKey}}` (HTTP header)
  - `{{$('Workflow Configuration').first().json.elevenLabsVoiceId}}` (ElevenLabs URL)
- **Connections:** Receives from `Start Workflow`, outputs to `Localization Agent`.
- **Edge cases / failures:**
  - Missing/invalid API key or voice ID → ElevenLabs request fails (401/403/404).
  - `targetLanguages` format is free-form; the agent may return unexpected language codes if ambiguous.

---

### Block 2 — Cultural Localization & Adaptation (LLM Agent with Structured Output)
**Overview:** Uses GPT-4.1-mini to localize Chinese content into multiple languages with cultural adaptation, returning structured JSON.  
**Nodes involved:** `Localization Agent`, `OpenAI Model - Localization`, `Structured Output - Translations`, plus attached agent tools (see Block 3)

#### Node: Localization Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompting, model call, and structured output parsing.
- **Key configuration:**
  - **Input text:** `={{ $json.chineseText }}`
  - **System message:** instructs cultural localization for target languages from `={{ $json.targetLanguages }}`, and asks for structured JSON including:
    - `language`, `languageCode` (ISO 639-1), `translatedText`, `culturalNotes`
  - `promptType: define`
  - `hasOutputParser: true` (must produce schema-compliant output)
- **AI connections (special ports):**
  - **Language model:** `OpenAI Model - Localization` (via `ai_languageModel`)
  - **Output parser:** `Structured Output - Translations` (via `ai_outputParser`)
  - **Tools available to this agent:**  
    - `Speech Optimization Agent Tool` (via `ai_tool`)  
    - `Voice Parameter Agent Tool` (via `ai_tool`)
- **Main output:** A JSON object expected to contain `translations: [ ... ]`.
- **Edge cases / failures:**
  - Model returns JSON that doesn’t match schema → structured parser fails (node error).
  - Language list is a comma-separated string; agent may interpret names inconsistently (e.g., “Spanish” vs “Español”).
  - If tools are invoked, results might not be merged into the final `translations` array unless explicitly prompted/handled (important for downstream TTS parameters).

#### Node: OpenAI Model - Localization
- **Type / role:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) used by the Localization Agent.
- **Model:** `gpt-4.1-mini`
- **Credentials:** `OpenAi account` (OpenAI API credential)
- **Connections:** Only to `Localization Agent` via `ai_languageModel`.
- **Failure modes:**
  - Invalid/expired OpenAI credential.
  - Rate limits / quota exhaustion.
  - Model availability changes.

#### Node: Structured Output - Translations
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)
- **Schema (manual):**
  - Root object with `translations` array
  - Each item: `language` (string), `languageCode` (string), `translatedText` (string), `culturalNotes` (string)
- **Connections:** Feeds parsing back into `Localization Agent` via `ai_outputParser`.
- **Failure modes:**
  - Missing required properties or wrong types (e.g., languageCode not string).
  - Non-JSON output from the model.

---

### Block 3 — Auxiliary AI Tools (Speech text optimization + Voice parameter selection)
**Overview:** Two agent tools are defined to refine speech text and compute ElevenLabs voice settings per language. They are available to agents but are **not explicitly used in the main data path**.  
**Nodes involved:** `Speech Optimization Agent Tool`, `OpenAI Model - Speech Optimization`, `Voice Parameter Agent Tool`, `OpenAI Model - Voice Parameters`, `Structured Output - Voice Params`

#### Node: Speech Optimization Agent Tool
- **Type / role:** Agent Tool (`@n8n/n8n-nodes-langchain.agentTool`) — callable tool for an agent.
- **Input binding:** `text: ={{ $fromAI("text_to_optimize") }}`
  - This means the calling agent must provide a tool argument named `text_to_optimize`.
- **System message:** Refines text for natural spoken delivery; returns **ONLY optimized text**.
- **Connected model:** `OpenAI Model - Speech Optimization` via `ai_languageModel`.
- **Connected to:** Available as a tool to `Localization Agent` via `ai_tool`.
- **Edge cases / failures:**
  - If the calling agent doesn’t pass `text_to_optimize`, the tool may error or produce empty output.
  - Tool output is not automatically injected into downstream nodes unless the agent includes it in final structured output.

#### Node: OpenAI Model - Speech Optimization
- **Type / role:** OpenAI Chat Model (`lmChatOpenAi`) for the speech optimization tool.
- **Model:** `gpt-4.1-mini`
- **Credentials:** `OpenAi account`
- **Connections:** Only to `Speech Optimization Agent Tool`.
- **Failure modes:** Same OpenAI constraints as above.

#### Node: Voice Parameter Agent Tool
- **Type / role:** Agent Tool (`agentTool`) — callable tool to recommend ElevenLabs `voice_settings`.
- **Input binding:** `text: ={{ $fromAI("language") }}`
  - The calling agent must provide a tool argument named `language`.
- **System message:** Returns JSON with:
  - `stability` (0–1), `similarity_boost` (0–1), `style` (0–1), `use_speaker_boost` (boolean)
- **Output parsing:** `hasOutputParser: true` with `Structured Output - Voice Params`.
- **Connected model:** `OpenAI Model - Voice Parameters`.
- **Connected to:** Available as a tool to `Localization Agent` via `ai_tool`.
- **Critical downstream dependency:** The ElevenLabs HTTP node expects these fields on each language item (`$json.stability`, etc.). Currently, the translation schema **does not include them**, so they will be missing unless the agent adds them anyway (see Block 5 edge cases).
- **Failure modes:**
  - Tool output schema mismatch → parser error.
  - Tool output not merged into translation items → ElevenLabs call may send null/undefined settings.

#### Node: OpenAI Model - Voice Parameters
- **Type / role:** OpenAI Chat Model for voice parameter tool.
- **Model:** `gpt-4.1-mini`
- **Credentials:** `OpenAi account`
- **Connections:** Only to `Voice Parameter Agent Tool`.
- **Failure modes:** OpenAI credential/rate limits.

#### Node: Structured Output - Voice Params
- **Type / role:** Structured Output Parser for voice parameters.
- **Schema:** object with numeric `stability`, `similarity_boost`, `style`, boolean `use_speaker_boost`.
- **Connections:** To `Voice Parameter Agent Tool` via `ai_outputParser`.
- **Failure modes:** Missing or out-of-range values (schema checks type, not range; range issues manifest later in ElevenLabs quality/validation).

---

### Block 4 — Prepare Languages Array
**Overview:** Converts the LLM’s `translations` array into a field used for looping.  
**Nodes involved:** `Prepare Languages Array`

#### Node: Prepare Languages Array
- **Type / role:** Set (`n8n-nodes-base.set`) — creates `languages`.
- **Key assignment:**
  - `languages` (array) = `={{ $json.translations }}`
- **Connections:** Input from `Localization Agent`, output to `Loop Over Languages`.
- **Edge cases / failures:**
  - If `translations` is missing or not an array, `languages` will be invalid and the loop will not behave as intended.

---

### Block 5 — Multilingual Audio Generation Loop (SplitInBatches → ElevenLabs → Process → Aggregate → Repeat)
**Overview:** Iterates through each language item, generates audio from ElevenLabs, attaches metadata, and loops until all items are processed.  
**Nodes involved:** `Loop Over Languages`, `Generate Audio with ElevenLabs`, `Process Audio Response`, `Aggregate Results`

#### Node: Loop Over Languages
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iteration controller.
- **Configuration:** Default options (batch size default depends on n8n UI; not explicitly set here).
- **Connections:**
  - Input from `Prepare Languages Array`
  - Output to `Generate Audio with ElevenLabs`
  - Receives loop-back input from `Aggregate Results` (to fetch next batch)
- **Important behavioral note:** For correct looping, the incoming items to `SplitInBatches` must be the list to iterate. Here, `Prepare Languages Array` produces an item containing `languages`, but `SplitInBatches` typically splits **incoming items**, not an array field—so this may require that the translations already be individual items rather than nested in `languages`. If not, you usually add an `Item Lists` or `Function` node to “split out” the array into items.
- **Edge cases / failures:**
  - If translations are in a single item array field, this node may only process one item total (and not iterate translations as intended) unless configured/preceded correctly.

#### Node: Generate Audio with ElevenLabs
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls ElevenLabs TTS and returns audio as binary.
- **Method/URL:**
  - `POST`
  - `https://api.elevenlabs.io/v1/text-to-speech/{{ $('Workflow Configuration').first().json.elevenLabsVoiceId }}`
- **Headers:**
  - `xi-api-key: {{ $('Workflow Configuration').first().json.elevenLabsApiKey }}`
  - `Content-Type: application/json`
- **Body parameters (JSON):**
  - `text: {{ $json.translatedText }}`
  - `model_id: eleven_multilingual_v2`
  - `voice_settings: {{ { stability: $json.stability, similarity_boost: $json.similarity_boost, style: $json.style, use_speaker_boost: $json.use_speaker_boost } }}`
- **Response handling:** `responseFormat: file` (binary audio in `binary.data`)
- **Connections:** Input from `Loop Over Languages`, output to `Process Audio Response`.
- **Edge cases / failures:**
  - Missing `translatedText` → 400 error.
  - Missing voice settings fields → may default poorly, or API may reject if null/invalid.
  - Invalid voice ID → 404.
  - Rate limits / concurrency constraints on ElevenLabs.
  - Large text length may exceed ElevenLabs limits.

#### Node: Process Audio Response
- **Type / role:** Code (`n8n-nodes-base.code`) — normalizes output and adds metadata.
- **Mode:** `runOnceForEachItem`
- **Logic summary:**
  - Reads `languageCode` and `language` from `item.json`
  - Builds `filename = audio_<languageCode>.mp3`
  - Returns:
    - `json`: `{ language, languageCode, filename, success: true }`
    - `binary`: passes through `item.binary` (audio data)
- **Connections:** Input from `Generate Audio with ElevenLabs`, output to `Aggregate Results`.
- **Edge cases / failures:**
  - If `languageCode` missing → filename becomes `audio_undefined.mp3`
  - If ElevenLabs returned error JSON instead of file → `item.binary` may be missing.

#### Node: Aggregate Results
- **Type / role:** Set (`n8n-nodes-base.set`) — maps output fields for a consistent per-language result and drives the loop.
- **Key assignments:**
  - `audioFile` (string) = `={{ $binary.data }}` (note: this is a binary reference, not actual file content)
  - `language` = `={{ $json.language }}`
  - `languageCode` = `={{ $json.languageCode }}`
  - `filename` = `={{ $json.filename }}`
- **Connections:**
  - Input from `Process Audio Response`
  - Output back to `Loop Over Languages` (loop continuation)
- **Edge cases / failures:**
  - `audioFile` set as string from `$binary.data` can be misleading; for file delivery you typically keep `binary` intact or use “Move Binary Data” / “Write Binary File” / storage upload.
  - No final “done” node: results are produced per iteration, but there is no explicit final aggregation into a single array or zip/package.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Workflow | Manual Trigger | Manual entry point | — | Workflow Configuration | ## Controlled Workflow Initiation\n**What:** Manual trigger initiates workflow with source content configuration\n**Why:** Allows controlled execution with customizable input parameters and language selections |
| Workflow Configuration | Set | Stores text, language list, ElevenLabs settings | Start Workflow | Localization Agent | ## Setup Steps\n1. Configure OpenAI API credentials in all AI agent nodes\n2. Set up ElevenLabs account, obtain API key\n3. Define target languages list in \"Workflow Configuration\" node using ISO language codes\n4. Customize localization prompts in AI agents to match brand voice and content type\n5. Adjust voice parameter ranges and optimization criteria based on audio requirements |
| Localization Agent | LangChain Agent | Cultural localization + structured JSON translations | Workflow Configuration | Prepare Languages Array | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| OpenAI Model - Localization | OpenAI Chat Model | LLM backend for localization | — (AI port) | — (to agent via AI port) | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| Structured Output - Translations | Structured Output Parser | Enforces translation JSON schema | — (AI port) | — (to agent via AI port) | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| Speech Optimization Agent Tool | Agent Tool | Tool to rewrite text for better speech | — (AI port) | — | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| OpenAI Model - Speech Optimization | OpenAI Chat Model | LLM backend for speech optimization tool | — (AI port) | — | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| Voice Parameter Agent Tool | Agent Tool | Tool to compute ElevenLabs voice settings | — (AI port) | — | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| OpenAI Model - Voice Parameters | OpenAI Chat Model | LLM backend for voice parameter tool | — (AI port) | — | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| Structured Output - Voice Params | Structured Output Parser | Enforces voice params JSON schema | — (AI port) | — | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| Prepare Languages Array | Set | Creates `languages` array from translations | Localization Agent | Loop Over Languages | ## Multilingual Audio Generation Loop\n**What:** Loop processes each language with optimized ElevenLabs settings\n**Why:** Generates consistent-quality audio across all languages with appropriate vocal characteristics |
| Loop Over Languages | Split In Batches | Iterates over languages | Prepare Languages Array; Aggregate Results (loop-back) | Generate Audio with ElevenLabs | ## Multilingual Audio Generation Loop\n**What:** Loop processes each language with optimized ElevenLabs settings\n**Why:** Generates consistent-quality audio across all languages with appropriate vocal characteristics |
| Generate Audio with ElevenLabs | HTTP Request | Calls ElevenLabs TTS and returns audio binary | Loop Over Languages | Process Audio Response | ## Multilingual Audio Generation Loop\n**What:** Loop processes each language with optimized ElevenLabs settings\n**Why:** Generates consistent-quality audio across all languages with appropriate vocal characteristics |
| Process Audio Response | Code | Adds metadata and filename; passes binary through | Generate Audio with ElevenLabs | Aggregate Results | ## Multilingual Audio Generation Loop\n**What:** Loop processes each language with optimized ElevenLabs settings\n**Why:** Generates consistent-quality audio across all languages with appropriate vocal characteristics |
| Aggregate Results | Set | Normalizes per-language output; drives loop continuation | Process Audio Response | Loop Over Languages | ## Multilingual Audio Generation Loop\n**What:** Loop processes each language with optimized ElevenLabs settings\n**Why:** Generates consistent-quality audio across all languages with appropriate vocal characteristics |
| Sticky Note | Sticky Note | Comment: setup steps | — | — | ## Setup Steps\n1. Configure OpenAI API credentials in all AI agent nodes\n2. Set up ElevenLabs account, obtain API key\n3. Define target languages list in \"Workflow Configuration\" node using ISO language codes\n4. Customize localization prompts in AI agents to match brand voice and content type\n5. Adjust voice parameter ranges and optimization criteria based on audio requirements |
| Sticky Note1 | Sticky Note | Comment: how it works | — | — | ## How It Works\nThis workflow delivers intelligent multilingual audio content creation for global marketing teams, e-learning providers, and content production studios. It solves the complex challenge of generating culturally adapted, professionally voiced translations optimized for each target language. The system begins with AI-powered localization that adapts source content for cultural context, idioms, and regional preferences rather than literal translation. Specialized AI agents then optimize speech parameters (pace, tone, emphasis) and voice characteristics (pitch, timbre, style) specific to each language's phonetic requirements. The workflow prepares language arrays and loops through each target language, generating optimized audio via ElevenLabs with customized voice parameters. All audio files are processed, formatted with metadata, and aggregated into a complete deliverable package, transforming single-source content into publication-ready multilingual audio assets. |
| Sticky Note2 | Sticky Note | Comment: prerequisites/use cases/benefits | — | — | ## Prerequisites\nOpenAI API access with GPT-4 capabilities, active ElevenLabs subscription with multi-voice access.\n## Use Cases\nGlobal product launch campaigns, international e-learning course production\n## Customization\nModify AI prompts for industry-specific terminology, add quality validation checkpoints\n## Benefits\nAchieves native-quality audio across languages, reduces production time by 80% |
| Sticky Note3 | Sticky Note | Comment: loop explanation | — | — | ## Multilingual Audio Generation Loop\n**What:** Loop processes each language with optimized ElevenLabs settings\n**Why:** Generates consistent-quality audio across all languages with appropriate vocal characteristics |
| Sticky Note5 | Sticky Note | Comment: localization explanation | — | — | ## Cultural Localization & Adaptation\n**What:** Localization agent adapts content for cultural context per language\n**Why:** Ensures messages resonate authentically rather than sounding mechanically translated |
| Sticky Note6 | Sticky Note | Comment: manual trigger explanation | — | — | ## Controlled Workflow Initiation\n**What:** Manual trigger initiates workflow with source content configuration\n**Why:** Allows controlled execution with customizable input parameters and language selections |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named: *Multilingual Speech Localization and Audio Generation Pipeline* (or your preferred name).

2) **Add “Manual Trigger”** node  
   - Node name: `Start Workflow`

3) **Add “Set”** node  
   - Node name: `Workflow Configuration`  
   - Add fields:
     - `chineseText` (string): your Chinese source text
     - `targetLanguages` (string): e.g. `English,Spanish,French,German,Japanese`
     - `elevenLabsApiKey` (string): your ElevenLabs API key (or use n8n credentials/vault instead)
     - `elevenLabsVoiceId` (string): your ElevenLabs voice ID
   - Connect: `Start Workflow` → `Workflow Configuration`

4) **Add “OpenAI Chat Model” (LangChain)** node  
   - Node name: `OpenAI Model - Localization`  
   - Model: `gpt-4.1-mini`  
   - Credentials: configure **OpenAI API** credential in n8n and select it here.

5) **Add “Structured Output Parser” (LangChain)** node  
   - Node name: `Structured Output - Translations`  
   - Schema (manual): an object with a `translations` array of objects containing:
     - `language` (string)
     - `languageCode` (string)
     - `translatedText` (string)
     - `culturalNotes` (string)

6) **Add “AI Agent” (LangChain Agent)** node  
   - Node name: `Localization Agent`  
   - Text: `={{ $json.chineseText }}`  
   - System message: instruct cultural localization for languages in `={{ $json.targetLanguages }}` and demand the JSON structure required by the translations schema.
   - Enable/attach:
     - **Language model**: connect `OpenAI Model - Localization` to agent’s **AI Language Model** port
     - **Output parser**: connect `Structured Output - Translations` to agent’s **AI Output Parser** port
   - Connect main flow: `Workflow Configuration` → `Localization Agent`

7) **(Optional but recommended) Add the speech optimization tool**
   - Add `OpenAI Model - Speech Optimization` (OpenAI chat, model `gpt-4.1-mini`, OpenAI credential)
   - Add `Speech Optimization Agent Tool`
     - Text input: `={{ $fromAI("text_to_optimize") }}`
     - System message: return only optimized speech-ready text
     - Connect model to tool via AI port
   - Connect tool to `Localization Agent` via the agent’s **AI Tool** port
   - Note: To make it effective, adjust the localization agent prompt to call this tool per language and store the result into `translatedText`.

8) **(Optional but strongly recommended) Add the voice-parameter tool**
   - Add `OpenAI Model - Voice Parameters` (OpenAI chat, model `gpt-4.1-mini`)
   - Add `Structured Output - Voice Params` with schema:
     - `stability` (number), `similarity_boost` (number), `style` (number), `use_speaker_boost` (boolean)
   - Add `Voice Parameter Agent Tool`
     - Text input: `={{ $fromAI("language") }}`
     - Enable output parser, connect `Structured Output - Voice Params`
     - Connect model to tool via AI port
   - Connect this tool to `Localization Agent` via **AI Tool**
   - Important: Modify the localization agent’s structured output schema (or add a second processing step) so that each translation item also includes these voice settings, otherwise ElevenLabs call won’t have the fields.

9) **Add “Set”** node  
   - Node name: `Prepare Languages Array`  
   - Add field: `languages` (array) = `={{ $json.translations }}`  
   - Connect: `Localization Agent` → `Prepare Languages Array`

10) **Ensure translations are individual items for looping**
   - If needed, add an `Item Lists` node (or Code node) to **Split Out Items** from `languages` into separate items.  
   - In the provided workflow JSON, this step is not explicit; implement it if `Split In Batches` only sees one item.

11) **Add “Split In Batches”** node  
   - Node name: `Loop Over Languages`  
   - Connect: `Prepare Languages Array` (or your “split out items” node) → `Loop Over Languages`

12) **Add “HTTP Request”** node (ElevenLabs TTS)
   - Node name: `Generate Audio with ElevenLabs`
   - Method: `POST`
   - URL: `=https://api.elevenlabs.io/v1/text-to-speech/{{ $('Workflow Configuration').first().json.elevenLabsVoiceId }}`
   - Headers:
     - `xi-api-key`: `={{ $('Workflow Configuration').first().json.elevenLabsApiKey }}`
     - `Content-Type`: `application/json`
   - Body (JSON):
     - `text`: `={{ $json.translatedText }}`
     - `model_id`: `eleven_multilingual_v2`
     - `voice_settings`: `={{ { stability: $json.stability, similarity_boost: $json.similarity_boost, style: $json.style, use_speaker_boost: $json.use_speaker_boost } }}`
   - Response: set to **File** (binary)
   - Connect: `Loop Over Languages` → `Generate Audio with ElevenLabs`

13) **Add “Code”** node
   - Node name: `Process Audio Response`
   - Mode: *Run once for each item*
   - Code: create `filename = audio_<languageCode>.mp3`, pass through `binary`, return metadata (`language`, `languageCode`, `filename`, `success: true`)
   - Connect: `Generate Audio with ElevenLabs` → `Process Audio Response`

14) **Add “Set”** node  
   - Node name: `Aggregate Results`
   - Map fields:
     - `audioFile` = `={{ $binary.data }}`
     - `language` = `={{ $json.language }}`
     - `languageCode` = `={{ $json.languageCode }}`
     - `filename` = `={{ $json.filename }}`
   - Connect: `Process Audio Response` → `Aggregate Results`

15) **Close the loop**
   - Connect: `Aggregate Results` → `Loop Over Languages`  
   This enables fetching the next batch until complete.

16) **Credentials**
   - **OpenAI**: configure once in n8n Credentials; select it in all OpenAI model nodes.
   - **ElevenLabs**: this workflow uses a raw API key stored in `Workflow Configuration`. In production, prefer n8n credentials/secrets and reference them in expressions.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## How It Works: End-to-end multilingual localization + speech/audio generation via GPT + ElevenLabs; loop per language; output audio with metadata | Sticky note “How It Works” (overview/positioning) |
| ## Prerequisites: OpenAI GPT-4 access + ElevenLabs subscription; suggested use cases and benefits | Sticky note “Prerequisites” |
| ## Setup Steps: configure OpenAI creds; ElevenLabs API key; define target languages; customize prompts; adjust parameter ranges | Sticky note “Setup Steps” |
| ## Cultural Localization & Adaptation: explains why localization agent is used (authentic resonance vs literal translation) | Sticky note “Cultural Localization & Adaptation” |
| ## Multilingual Audio Generation Loop: explains per-language processing with optimized settings | Sticky note “Multilingual Audio Generation Loop” |
| ## Controlled Workflow Initiation: manual trigger rationale | Sticky note “Controlled Workflow Initiation” |

**Disclaimer (provided by user):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.