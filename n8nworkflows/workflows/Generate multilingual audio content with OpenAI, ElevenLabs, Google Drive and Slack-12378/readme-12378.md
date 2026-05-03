Generate multilingual audio content with OpenAI, ElevenLabs, Google Drive and Slack

https://n8nworkflows.xyz/workflows/generate-multilingual-audio-content-with-openai--elevenlabs--google-drive-and-slack-12378


# Generate multilingual audio content with OpenAI, ElevenLabs, Google Drive and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI-Powered Multilingual Audio Content Generator with Quality Control  
**Title provided:** Generate multilingual audio content with OpenAI, ElevenLabs, Google Drive and Slack

**Purpose:**  
Transforms an input Chinese script into multiple language versions (English, Spanish, French, German), generates audio files (TTS) for each language, performs a quality gate for the English translation before generating English audio, then packages results, uploads to Google Drive, and notifies a Slack channel.

**Target use cases:**
- E-learning localization and course narration in multiple languages
- Multilingual podcast/marketing audio distribution at scale

### Logical blocks
1. **1.1 Input & Configuration**
2. **1.2 Multilingual Translation**
3. **1.3 English Translation Quality Validation (Gate)**
4. **1.4 Audio Generation (TTS)**
5. **1.5 Audio Metrics + Packaging**
6. **1.6 Distribution (Google Drive) + Notification (Slack)**

---

## 2. Block-by-Block Analysis

### 2.1 Input & Configuration

**Overview:** Initializes the workflow manually and defines all runtime parameters (source text, API keys, IDs, thresholds) used by downstream nodes.

**Nodes involved:**
- Start Workflow
- Workflow Configuration

#### Node: Start Workflow
- **Type / role:** `Manual Trigger` — manual entry point for testing and ad-hoc runs.
- **Configuration:** No parameters.
- **Inputs/Outputs:**  
  - **Output →** Workflow Configuration
- **Edge cases/failures:** None (except user not executing).
- **Version notes:** Node v1.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central config object for the run.
- **Key fields set:**
  - `chineseScript` (string placeholder)
  - `elevenLabsApiKey` (string placeholder)
  - `elevenLabsVoiceId` (string placeholder)
  - `googleDriveFolderId` (string placeholder)
  - `slackChannel` (string placeholder)
  - `qualityThreshold` (number, default **7**)
- **Configuration choices:**
  - “Include other fields” enabled (passes through any incoming JSON).
- **Outputs:** Fan-out to all translation nodes:
  - **Output →** Translate to English, Translate to Spanish, Translate to French, Translate to German
- **Edge cases/failures:**
  - Missing/invalid IDs (Drive folder, Slack channel) cause downstream failures.
  - If placeholders are not replaced, OpenAI/ElevenLabs/Drive/Slack will fail.
- **Version notes:** Set node v3.4.

**Sticky note coverage (contextual):**
- “How It Works … translates … validates … generates … uploads … notifies …”
- “Setup Steps … credentials … ElevenLabs … threshold … Drive … Slack …”

---

### 2.2 Multilingual Translation

**Overview:** Translates the Chinese source into four target languages using OpenAI (LangChain OpenAI node). English translation is additionally routed into a quality-check gate before audio generation.

**Nodes involved:**
- Translate to English
- Translate to Spanish
- Translate to French
- Translate to German

#### Node: Translate to English
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call for translation (Chinese → English).
- **Model:** `gpt-4o` (selected by ID).
- **Instruction prompt:** Enforces professional, TTS-friendly translation; **must return only translated text**.
- **Key expression input:**
  - Content: `{{ $('Workflow Configuration').first().json.chineseScript }}`
- **Outputs:**
  - **Output →** Quality Check Translation
- **Credentials:** OpenAI credential “OpenAi account”.
- **Edge cases/failures:**
  - OpenAI auth/quota errors.
  - Model output not matching expectations (should be pure text, but could still include quotes/newlines—generally OK).
- **Version notes:** v2.1.

#### Node: Translate to Spanish / French / German
- **Type / role:** same OpenAI node, translating Chinese → respective language.
- **Model:** `gpt-4o`.
- **Key expression input:** same source Chinese script from Workflow Configuration.
- **Outputs:**
  - Spanish → Generate audio in Spanish
  - French → Generate audio in French
  - German → Generate audio in German
- **Credentials:** “OpenAi account”.
- **Edge cases/failures:** same as English translation.
- **Version notes:** v2.1.

**Sticky note coverage (contextual):**
- “Multilingual Translation & Validation … checks quality scores … ensures accuracy before audio …”

---

### 2.3 English Translation Quality Validation (Gate)

**Overview:** Scores the English translation using OpenAI and blocks English audio generation unless the score meets `qualityThreshold`.

**Nodes involved:**
- Quality Check Translation
- Check Quality Score

#### Node: Quality Check Translation
- **Type / role:** OpenAI node used as a structured evaluator (LLM-as-judge).
- **Model:** `gpt-4o`.
- **Instruction prompt:** Must return **ONLY** JSON: `{"score": 8, "issues": "..."}`
- **Key expression input:** builds a two-part text:
  - Original Chinese: `{{ $('Workflow Configuration').first().json.chineseScript }}`
  - English translation: `{{ $json.message.content }}`
- **Input connection:** from Translate to English.
- **Output connection:** to Check Quality Score.
- **Credentials:** “OpenAi account”.
- **Edge cases/failures:**
  - If the model returns non-JSON (or extra text), downstream `JSON.parse(...)` may fail.
  - Very long scripts may hit token limits or increase cost/latency.
- **Version notes:** v2.1.

#### Node: Check Quality Score
- **Type / role:** `If` — quality gate.
- **Conditions (AND):**
  1. `contains "score"` in `$('Quality Check Translation').item.json.message.content`
  2. Boolean expression:
     - `JSON.parse($('Quality Check Translation').item.json.message.content).score >= $('Workflow Configuration').first().json.qualityThreshold`
- **Outputs:**
  - **True path →** Generate English Audio (ElevenLabs)
  - **False path →** (not connected) workflow effectively stops for the English branch; other language branches still proceed.
- **Edge cases/failures:**
  - `JSON.parse(...)` throws if the content is not valid JSON (even if it contains “score”).
  - Using `.item` assumes matching item indexes; if parallel items are introduced later, this can misalign.
- **Version notes:** If node v2.3.

---

### 2.4 Audio Generation (TTS)

**Overview:** Produces audio files per language. English uses ElevenLabs via HTTP Request; Spanish/French/German use OpenAI “audio” resource.

**Nodes involved:**
- Generate English Audio (ElevenLabs)
- Generate audio in Spanish
- Generate audio in French
- Generate audio in German
- Add Language Metadata

#### Node: Generate English Audio (ElevenLabs)
- **Type / role:** `HTTP Request` — calls ElevenLabs Text-to-Speech endpoint and stores binary output.
- **Request:**
  - **POST** `https://api.elevenlabs.io/v1/text-to-speech/{{ voiceId }}`
  - `voiceId = $('Workflow Configuration').first().json.elevenLabsVoiceId`
- **Headers:**
  - `xi-api-key: {{ elevenLabsApiKey }}`
  - `Content-Type: application/json`
- **JSON body (expression):**
  - Uses translated English: `text: $json.message.content`
  - `model_id: "eleven_multilingual_v2"`
  - voice settings: stability 0.5, similarity_boost 0.75
- **Response handling:**
  - Response format: **File**
  - Output binary property: `audio_english`
- **Outputs:** → Add Language Metadata
- **Edge cases/failures:**
  - 401/403 if API key invalid.
  - 400 if voiceId invalid or body malformed.
  - Large text may exceed ElevenLabs limits / quota.
- **Version notes:** HTTP Request v4.3.

#### Node: Generate audio in Spanish / French / German
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` with `resource: "audio"` — OpenAI audio generation.
- **Configuration choices:**
  - Minimal config shown; relies on node defaults for voice/model unless configured elsewhere in UI.
- **Inputs:**
  - From the respective translation node.
- **Outputs:** → Calculate Audio Metrics
- **Credentials:** “OpenAi account 2”
- **Edge cases/failures:**
  - If no voice/model is configured by default in your n8n/OpenAI node settings, runs may fail or produce unexpected format.
  - Audio output format and binary property naming may vary by node version/settings; downstream metrics code assumes binary exists.
- **Version notes:** v2.1.

#### Node: Add Language Metadata
- **Type / role:** `Set` — enriches the English audio item with metadata for packaging.
- **Fields set:**
  - `language = "English"`
  - `audioFormat = "mp3"`
  - `generatedAt = {{ $now.toISO() }}`
  - `translationQuality = {{ $('Quality Check Translation').first().json.message.content }}`
- **Input:** from Generate English Audio (ElevenLabs).
- **Output:** → Combine All Audio Files
- **Edge cases/failures:**
  - If the quality-check branch didn’t run, referencing `Quality Check Translation` would normally be risky; here it always precedes English audio, so OK.
- **Version notes:** Set node v3.4.

**Sticky note coverage (contextual):**
- “Audio Generation … using ElevenLabs … professional voice-over …”

---

### 2.5 Audio Metrics + Packaging

**Overview:** Computes basic metrics per audio binary item, then aggregates all items (all languages) into one array for subsequent processing.

**Nodes involved:**
- Calculate Audio Metrics
- Combine All Audio Files

#### Node: Calculate Audio Metrics
- **Type / role:** `Code` — inspects each incoming item’s binary, derives file size and adds metadata.
- **Logic (interpreted):**
  - For each input item:
    - Takes the **first binary key** present.
    - Calculates base64 byte length → KB size (`audioSizeKB`)
    - Infers `language` from binary key containing `spanish`, `french`, `german`; otherwise `Unknown`
    - Sets `audioFormat = "mp3"`, `generatedAt = now`
  - Returns only items that have binary data.
- **Input:** from OpenAI audio generation nodes (Spanish/French/German).
- **Output:** → Combine All Audio Files
- **Edge cases/failures:**
  - If OpenAI audio nodes output binary keys that do not include `spanish/french/german`, language becomes `Unknown`.
  - If an item has multiple binaries, only the first is considered.
  - If binary is present but not base64 in `data`, size calculation may be wrong.
- **Version notes:** Code node v2.

#### Node: Combine All Audio Files
- **Type / role:** `Aggregate` — bundles all incoming item data.
- **Mode:** `aggregateAllItemData`
- **Destination field:** `allAudioFiles`
- **Inputs:**
  - From Add Language Metadata (English)
  - From Calculate Audio Metrics (Spanish/French/German)
- **Output:** → Upload to Google Drive
- **Edge cases/failures:**
  - Aggregation behavior depends on execution order and how many items arrive; if English branch is blocked by quality gate, only 3 language items may be aggregated.
- **Version notes:** Aggregate node v1.

---

### 2.6 Distribution (Google Drive) + Notification (Slack)

**Overview:** Uploads content to Google Drive and posts a Slack message summarizing the run.

**Nodes involved:**
- Upload to Google Drive
- Send Slack Notification

#### Node: Upload to Google Drive
- **Type / role:** `Google Drive` — uploads a file to a specified folder.
- **Filename expression:**
  - `chinese_translation_{{ $json.language }}_{{ $now.toFormat('yyyyMMdd_HHmmss') }}.mp3`
- **Drive:** “My Drive”
- **Folder:** ID from config: `{{ $('Workflow Configuration').first().json.googleDriveFolderId }}`
- **Input:** from Combine All Audio Files.
- **Output:** → Send Slack Notification
- **Important integration note:**  
  The node configuration shown does not explicitly map which **binary** to upload. In n8n, Drive upload typically needs a specific binary property (e.g., `data`) and a single item/file. Here, the prior Aggregate produces `allAudioFiles` array, which may not match Drive upload expectations without additional looping or splitting.
- **Edge cases/failures:**
  - Auth errors (OAuth) or missing folder permissions.
  - Upload fails if there is no binary on the current item or if multiple files are intended but not iterated.
- **Version notes:** Google Drive node v3.

#### Node: Send Slack Notification
- **Type / role:** `Slack` — sends a message to a channel via OAuth2.
- **Channel:** `{{ $('Workflow Configuration').first().json.slackChannel }}`
- **Message text includes:**
  - Original script preview: first 100 chars
  - “Languages Generated: English, Spanish, French, German”
  - Total files: `{{ $json.allAudioFiles.length }}`
  - Drive folder ID
- **Input:** from Upload to Google Drive.
- **Output:** none (end).
- **Edge cases/failures:**
  - Slack OAuth token revoked or missing `chat:write` scope.
  - Channel ID invalid or bot not in channel.
- **Version notes:** Slack node v2.4.

**Sticky note coverage (contextual):**
- “Packaging, Distribution & Notification … metrics … bundles … uploads … fast handoff …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Workflow | Manual Trigger | Manual entry point | — | Workflow Configuration | ## How It Works … translates … validates … generates … uploads … notifies … |
| Workflow Configuration | Set | Central runtime configuration | Start Workflow | Translate to English; Translate to Spanish; Translate to French; Translate to German | ## Setup Steps… credentials… ElevenLabs… threshold… Drive… Slack… |
| Translate to English | OpenAI (LangChain) | Chinese→English translation | Workflow Configuration | Quality Check Translation | ## Multilingual Translation & Validation … ensures translation accuracy … |
| Quality Check Translation | OpenAI (LangChain) | Scores English translation and returns JSON | Translate to English | Check Quality Score | ## Multilingual Translation & Validation … checks quality scores … |
| Check Quality Score | If | Quality gate for English audio | Quality Check Translation | Generate English Audio (ElevenLabs) | ## Multilingual Translation & Validation … avoiding rework. |
| Generate English Audio (ElevenLabs) | HTTP Request | English TTS via ElevenLabs | Check Quality Score (true) | Add Language Metadata | ## Audio Generation … ElevenLabs … professional voice-over … |
| Add Language Metadata | Set | Enrich English audio item | Generate English Audio (ElevenLabs) | Combine All Audio Files | ## Packaging, Distribution & Notification … bundles … uploads … |
| Translate to Spanish | OpenAI (LangChain) | Chinese→Spanish translation | Workflow Configuration | Generate audio in Spanish | ## Multilingual Translation & Validation … |
| Generate audio in Spanish | OpenAI (LangChain) Audio | Spanish TTS via OpenAI audio | Translate to Spanish | Calculate Audio Metrics | ## Audio Generation … |
| Translate to French | OpenAI (LangChain) | Chinese→French translation | Workflow Configuration | Generate audio in French | ## Multilingual Translation & Validation … |
| Generate audio in French | OpenAI (LangChain) Audio | French TTS via OpenAI audio | Translate to French | Calculate Audio Metrics | ## Audio Generation … |
| Translate to German | OpenAI (LangChain) | Chinese→German translation | Workflow Configuration | Generate audio in German | ## Multilingual Translation & Validation … |
| Generate audio in German | OpenAI (LangChain) Audio | German TTS via OpenAI audio | Translate to German | Calculate Audio Metrics | ## Audio Generation … |
| Calculate Audio Metrics | Code | Computes file size/metadata per audio item | Generate audio in Spanish; Generate audio in French; Generate audio in German | Combine All Audio Files | ## Packaging, Distribution & Notification … computes audio metrics … |
| Combine All Audio Files | Aggregate | Aggregates all items into `allAudioFiles` | Add Language Metadata; Calculate Audio Metrics | Upload to Google Drive | ## Packaging, Distribution & Notification … bundles all language files … |
| Upload to Google Drive | Google Drive | Uploads output to configured folder | Combine All Audio Files | Send Slack Notification | ## Packaging, Distribution & Notification … centralized access … |
| Send Slack Notification | Slack | Posts completion summary | Upload to Google Drive | — | ## Packaging, Distribution & Notification … fast handoff … |
| Sticky Note | Sticky Note | Comment block | — | — | ## Prerequisites … Use Cases … Customization … Benefits … |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Setup Steps … |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## How It Works … |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Packaging, Distribution & Notification … |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Audio Generation … |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## Multilingual Translation & Validation … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named:  
   “AI-Powered Multilingual Audio Content Generator with Quality Control” (or your preferred name).

2. **Add “Manual Trigger”** node named **Start Workflow**.

3. **Add a “Set”** node named **Workflow Configuration** and connect:  
   Start Workflow → Workflow Configuration.  
   Configure these fields:
   - `chineseScript` (String) – your Chinese source text
   - `elevenLabsApiKey` (String)
   - `elevenLabsVoiceId` (String)
   - `googleDriveFolderId` (String)
   - `slackChannel` (String, Slack channel ID)
   - `qualityThreshold` (Number) default 7  
   Enable **Include Other Fields**.

4. **Add 4 OpenAI (LangChain) nodes** for translations:
   - **Translate to English** (model `gpt-4o`)  
     Input content expression: `{{ $('Workflow Configuration').first().json.chineseScript }}`  
     Instructions: Chinese→English, return only translated text.
   - **Translate to Spanish** (same model, Chinese→Spanish)
   - **Translate to French** (Chinese→French)
   - **Translate to German** (Chinese→German)  
   Connect Workflow Configuration → each translation node (fan-out).  
   Configure **OpenAI credentials** for these translation nodes.

5. **Add English quality check:**
   - Add OpenAI (LangChain) node **Quality Check Translation** (model `gpt-4o`)  
     Instructions: return only JSON `{"score": x, "issues": "..."}`  
     Content:
     ```
     Original Chinese: {{ $('Workflow Configuration').first().json.chineseScript }}

     English Translation: {{ $json.message.content }}
     ```
     Connect: Translate to English → Quality Check Translation.
   - Add **If** node **Check Quality Score**. Connect: Quality Check Translation → Check Quality Score.  
     Add conditions (AND):
     - String contains: left `{{ $('Quality Check Translation').item.json.message.content }}` contains right `score`
     - Boolean true: left
       `{{ JSON.parse($('Quality Check Translation').item.json.message.content).score >= $('Workflow Configuration').first().json.qualityThreshold }}`

6. **Add English audio generation via ElevenLabs:**
   - Add **HTTP Request** node **Generate English Audio (ElevenLabs)** connected from **Check Quality Score (true)**.
   - Method: POST  
   - URL:
     `https://api.elevenlabs.io/v1/text-to-speech/{{ $('Workflow Configuration').first().json.elevenLabsVoiceId }}`
   - Headers:
     - `xi-api-key` = `{{ $('Workflow Configuration').first().json.elevenLabsApiKey }}`
     - `Content-Type` = `application/json`
   - Body type: JSON; body:
     ```
     {
       "text": {{$json.message.content}},
       "model_id": "eleven_multilingual_v2",
       "voice_settings": { "stability": 0.5, "similarity_boost": 0.75 }
     }
     ```
   - Response: **File**, binary property name: `audio_english`.

7. **Add metadata for English item:**
   - Add **Set** node **Add Language Metadata**, connect from Generate English Audio (ElevenLabs).
   - Set:
     - `language = "English"`
     - `audioFormat = "mp3"`
     - `generatedAt = {{ $now.toISO() }}`
     - `translationQuality = {{ $('Quality Check Translation').first().json.message.content }}`

8. **Add OpenAI audio generation for Spanish/French/German:**
   - Add 3 OpenAI (LangChain) nodes:
     - **Generate audio in Spanish** (Resource: **audio**)
     - **Generate audio in French** (Resource: **audio**)
     - **Generate audio in German** (Resource: **audio**)  
   - Connect:
     - Translate to Spanish → Generate audio in Spanish
     - Translate to French → Generate audio in French
     - Translate to German → Generate audio in German  
   - Set OpenAI credentials (can be separate, as in the JSON: “OpenAi account 2”).  
   - Ensure each node outputs binary audio (configure output format/voice/model as required in your environment).

9. **Add “Code” node** **Calculate Audio Metrics**:
   - Connect all three OpenAI audio nodes → Calculate Audio Metrics.
   - Paste logic that:
     - reads first binary key,
     - computes size,
     - infers language from the binary key,
     - appends `audioSizeKB`, `audioFormat`, `generatedAt`.

10. **Add “Aggregate” node** **Combine All Audio Files**:
   - Connect:
     - Add Language Metadata → Combine All Audio Files
     - Calculate Audio Metrics → Combine All Audio Files
   - Mode: aggregate all item data  
   - Destination field: `allAudioFiles`

11. **Add Google Drive upload** node **Upload to Google Drive**:
   - Connect: Combine All Audio Files → Upload to Google Drive
   - Credentials: Google Drive OAuth2
   - Drive: My Drive
   - Folder ID: `{{ $('Workflow Configuration').first().json.googleDriveFolderId }}`
   - File name expression:
     `chinese_translation_{{ $json.language }}_{{ $now.toFormat('yyyyMMdd_HHmmss') }}.mp3`
   - Important: ensure you configure the node to upload the intended **binary**. If you truly want to upload **multiple** language files, you will typically need to **Split Out Items** (or loop over `allAudioFiles`) and upload one file per item.

12. **Add Slack notification** node **Send Slack Notification**:
   - Connect: Upload to Google Drive → Send Slack Notification
   - Auth: Slack OAuth2 (needs `chat:write`)
   - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
   - Message text (as configured) referencing `allAudioFiles.length`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: AI translation API access (OpenAI/DeepL), ElevenLabs account with sufficient character quota. Use cases: E-learning course localization, podcast multilingual distribution. Customization: Add languages, modify quality score thresholds. Benefits: Reduces localization time by 95%, eliminates voice talent costs. | Sticky note “Prerequisites / Use Cases / Customization / Benefits” |
| Setup Steps: configure translation credentials, add ElevenLabs key + voice models, set threshold, connect Google Drive folder, configure Slack notifications. | Sticky note “Setup Steps” |
| Packaging/Distribution intent: compute metrics, bundle language files, upload package, notify team. | Sticky note “Packaging, Distribution & Notification” |
| Architectural note: English audio is gated by translation quality; Spanish/French/German audio is not gated in this workflow. | Workflow logic (node connections) |
| Integration note: Drive upload of multiple files likely needs an iteration step (split/loop) after aggregation. | Based on Aggregate → Google Drive design |

