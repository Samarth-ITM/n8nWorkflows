Translate and dub spokesperson videos using Anthropic and deAPI

https://n8nworkflows.xyz/workflows/translate-and-dub-spokesperson-videos-using-anthropic-and-deapi-14662


# Translate and dub spokesperson videos using Anthropic and deAPI

# 1. Workflow Overview

This workflow localizes a spokesperson video into another language and replaces the on-screen presenter with a local presenter image, without requiring a new shoot. It reads a source video and a presenter image, transcribes the speech, translates the transcript with Anthropic, generates dubbed speech with deAPI, and then renders a lip-synced talking-head video using the new audio and image.

Typical use cases include:
- Marketing video localization
- Multilingual spokesperson content
- Regionalized product announcements
- Training or onboarding video adaptation

The workflow is organized into the following logical blocks:

## 1.1 Start & Configuration
A manual trigger starts the run, and a Set node defines the target language used later by both the translation and text-to-speech stages.

## 1.2 Input File Loading
Two file inputs are loaded in parallel:
- the original spokesperson video
- the local presenter reference image

## 1.3 Video Transcription
The source video is sent to deAPI for transcription, then the returned file content is converted into plain text for downstream AI processing.

## 1.4 Translation with AI Agent
An Anthropic-backed AI Agent translates the transcript into the selected target language while preserving spoken pacing and natural delivery.

## 1.5 Dubbed Audio Generation
The translated text is converted into speech with deAPI using a Qwen3 TTS model.

## 1.6 Lip-Synced Video Generation
The generated dubbed audio is merged with the presenter image and sent to deAPI to create a talking-head video whose appearance is guided by the local presenter image.

---

# 2. Block-by-Block Analysis

## 2.1 Start & Configuration

**Overview:**  
This block provides the workflow entry point and defines the localization target language. That language is referenced later by both the translation prompt and the TTS node.

**Nodes Involved:**
- Manual Trigger
- Set Fields

### Node: Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual workflow entry point for test or ad hoc executions.
- **Configuration choices:** No parameters are configured; it simply starts execution when the user clicks **Test Workflow**.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Set Fields`
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - No runtime failure expected.
  - Only usable interactively; not suitable for automated production triggers.
- **Sub-workflow reference:** None.

### Node: Set Fields
- **Type and technical role:** `n8n-nodes-base.set`; defines workflow variables.
- **Configuration choices:** Creates one string field:
  - `target_language = "Spanish"`
- **Key expressions or variables used:**
  - Later referenced as `$('Set Fields').item.json.target_language`
- **Input and output connections:**
  - Input: `Manual Trigger`
  - Output: `Read Source Video`, `Read Local Presenter Image`
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - If renamed, any expression referencing `Set Fields` by node name will break.
  - If the target language is unsupported or formatted unexpectedly, downstream TTS may fail or produce poor results.
- **Sub-workflow reference:** None.

---

## 2.2 Input File Loading

**Overview:**  
This block loads the two binary assets required for localization: the source video and the local presenter reference image. Both branches start from the same configuration node and proceed in parallel.

**Nodes Involved:**
- Read Source Video
- Read Local Presenter Image

### Node: Read Source Video
- **Type and technical role:** `n8n-nodes-base.readWriteFile`; reads a local file from disk into binary data.
- **Configuration choices:**
  - File path: `/path/to/your/spokesperson-video.mp4`
  - Output binary property name: `video`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Set Fields`
  - Output: `deAPI Transcribe Video`
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Invalid path or missing file
  - File permission issues
  - Unsupported or corrupted video format
  - Large file size may slow upload or exceed API limits
- **Sub-workflow reference:** None.

### Node: Read Local Presenter Image
- **Type and technical role:** `n8n-nodes-base.readWriteFile`; reads a local image into binary data.
- **Configuration choices:**
  - File path: `/path/to/your/local-presenter.jpg`
  - Output binary property name: `image`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Set Fields`
  - Output: `Merge Audio + Image` on input index 1
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Invalid path or missing file
  - Permission issues
  - Unsupported image format
  - Oversized image; sticky note indicates a practical max size of 10 MB
- **Sub-workflow reference:** None.

---

## 2.3 Video Transcription

**Overview:**  
This block submits the source video to deAPI for speech transcription and then extracts plain text from the returned file-based output. The result becomes the transcript used by the AI translation stage.

**Nodes Involved:**
- deAPI Transcribe Video
- Extract from File

### Node: deAPI Transcribe Video
- **Type and technical role:** `n8n-nodes-deapi.deapi`; sends the input video to deAPI’s transcription endpoint.
- **Configuration choices:**
  - Resource: `video`
  - Operation: `transcribe`
  - Source: `binary`
  - Binary property name: `video`
  - Wait timeout: `120`
- **Key expressions or variables used:** Uses the incoming binary property `video`.
- **Input and output connections:**
  - Input: `Read Source Video`
  - Output: `Extract from File`
- **Version-specific requirements:** Type version 1; requires the deAPI community/custom node to be installed in the n8n instance.
- **Edge cases or potential failure types:**
  - Invalid deAPI credentials
  - Upload failure or timeout
  - Unsupported video codec/container
  - API-side job delay exceeding 120-second wait timeout
  - Transcription output format changes that affect downstream extraction
- **Sub-workflow reference:** None.

### Node: Extract from File
- **Type and technical role:** `n8n-nodes-base.extractFromFile`; converts file content into plain text stored in JSON.
- **Configuration choices:**
  - Operation: `text`
  - Destination key: `text`
- **Key expressions or variables used:** Outputs extracted content as `$json.text`
- **Input and output connections:**
  - Input: `deAPI Transcribe Video`
  - Output: `AI Agent`
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or potential failure types:**
  - If the incoming data is not a text-readable file, extraction may fail
  - If deAPI returns JSON or another structure instead of text content, this node may not produce the expected transcript
  - Empty transcript results in weak translation output
- **Sub-workflow reference:** None.

---

## 2.4 Translation with AI Agent

**Overview:**  
This block translates the transcript into the selected target language using an AI Agent connected to Anthropic. The prompt is designed specifically for spoken video localization and asks for plain translated text only.

**Nodes Involved:**
- AI Agent
- Anthropic Chat Model

### Node: AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates LLM-based processing using a defined prompt and system instructions.
- **Configuration choices:**
  - Prompt type: `define`
  - Main input text:
    - asks to translate into the target language from `Set Fields`
    - requests output with no timestamps, line numbers, or formatting
    - includes transcript from `$json.text`
  - System message instructs the model to:
    - preserve tone, energy, and intent
    - adapt idioms and cultural references
    - keep sentence length similar for lip-sync compatibility
    - use spoken language
    - return only translated text
- **Key expressions or variables used:**
  - `{{ $('Set Fields').item.json.target_language }}`
  - `{{ $json.text }}`
- **Input and output connections:**
  - Main input: `Extract from File`
  - AI language model input: `Anthropic Chat Model`
  - Main output: `deAPI Generate Speech`
- **Version-specific requirements:** Type version 1.7; requires LangChain-capable n8n build with AI nodes available.
- **Edge cases or potential failure types:**
  - Missing or invalid Anthropic credentials
  - Prompt expression failure if `Set Fields` is renamed or transcript is missing
  - Model may still return formatting despite instructions
  - Very long transcript may exceed context size or raise cost/latency
  - Translation may not match TTS language naming expectations
- **Sub-workflow reference:** None.

### Node: Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; provides the language model backend for the AI Agent.
- **Configuration choices:**
  - Model: `claude-opus-4-6`
  - No additional options configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Output via `ai_languageModel` connection to `AI Agent`
- **Version-specific requirements:** Type version 1.3; requires Anthropic credentials and compatible AI node package support.
- **Edge cases or potential failure types:**
  - Credential/authentication failure
  - Model availability or quota limitations
  - Regional access or API account restrictions
- **Sub-workflow reference:** None.

---

## 2.5 Dubbed Audio Generation

**Overview:**  
This block takes the translated text and generates spoken audio in the target language using deAPI TTS. It converts the AI output into an audio asset that will later drive the lip-sync video generation.

**Nodes Involved:**
- deAPI Generate Speech

### Node: deAPI Generate Speech
- **Type and technical role:** `n8n-nodes-deapi.deapi`; TTS/audio generation via deAPI.
- **Configuration choices:**
  - Resource: `audio`
  - Operation: `generateSpeech`
  - Text: `={{ $json.output }}`
  - Model: `Qwen3_TTS_12Hz_1_7B_CustomVoice`
  - Qwen3 language: `={{ $('Set Fields').item.json.target_language }}`
  - Wait timeout: `120`
- **Key expressions or variables used:**
  - `{{ $json.output }}` from `AI Agent`
  - `{{ $('Set Fields').item.json.target_language }}`
- **Input and output connections:**
  - Input: `AI Agent`
  - Output: `Merge Audio + Image` on input index 0
- **Version-specific requirements:** Type version 1; requires installed deAPI node and valid deAPI credentials.
- **Edge cases or potential failure types:**
  - If AI Agent output is empty or malformed, audio generation may fail
  - Target language label may not match deAPI’s accepted language values
  - Timeout if synthesis takes too long
  - Audio format/output assumptions must remain compatible with the next node
- **Sub-workflow reference:** None.

---

## 2.6 Lip-Synced Video Generation

**Overview:**  
This block combines the dubbed audio with the local presenter image and sends both to deAPI to generate a talking-head video. The image is used as the first frame to guide the visual identity of the speaker.

**Nodes Involved:**
- Merge Audio + Image
- deAPI Generate From Audio

### Node: Merge Audio + Image
- **Type and technical role:** `n8n-nodes-base.merge`; combines two incoming items into one item by matching position.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Input 0: `deAPI Generate Speech`
  - Input 1: `Read Local Presenter Image`
  - Output: `deAPI Generate From Audio`
- **Version-specific requirements:** Type version 3.2.
- **Edge cases or potential failure types:**
  - If one branch does not produce an item, merge may not emit the expected combined item
  - If one branch outputs multiple items, pairings may become misaligned
  - Binary property collisions are possible in more complex scenarios, though here the properties differ (`image` and likely deAPI audio output)
- **Sub-workflow reference:** None.

### Node: deAPI Generate From Audio
- **Type and technical role:** `n8n-nodes-deapi.deapi`; generates a lip-synced video from audio and image guidance.
- **Configuration choices:**
  - Resource: `video`
  - Operation: `generateFromAudio`
  - Prompt:  
    `A person speaking naturally to the camera, subtle head movements and facial expressions, professional lighting, medium close-up shot, steady camera`
  - Options:
    - `frames = 241`
    - `firstFrame = image`
    - `waitTimeout = 300`
- **Key expressions or variables used:**
  - Uses merged binary/image data where `firstFrame` points to binary property `image`
- **Input and output connections:**
  - Input: `Merge Audio + Image`
  - Output: none; terminal node
- **Version-specific requirements:** Type version 1; requires deAPI credentials and the deAPI node package.
- **Edge cases or potential failure types:**
  - Missing image binary under property `image`
  - Missing or incompatible audio from upstream
  - Long rendering time exceeding 300-second timeout
  - Prompt may influence visual style inconsistently
  - Generated output quality may vary depending on image quality and audio duration
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Canvas documentation / overview note |  |  | ## Try It Out!<br>### Localize a spokesperson video into another language with a new presenter — no filming required.<br><br>This workflow transcribes a video, translates the speech, generates dubbed audio, creates a lip-synced video.<br><br>### How it works<br>1. **Manual Trigger** starts the workflow<br>2. **Set Fields** defines the target language<br>3. **Read Source Video** and **Read Local Presenter Image** load the input files in parallel<br>4. **deAPI Transcribe Video** extracts the original speech as text with timestamps<br>5. **AI Agent** translates the transcript into the target language<br>6. **deAPI Generate Speech** creates dubbed audio in the target language<br>7. **deAPI Generate From Audio** produces a lip-synced talking-head video from the dubbed audio, using the local presenter image as the first frame<br><br>### Requirements<br>- [deAPI](https://deapi.ai) account for transcription, TTS, video generation<br>- Anthropic account for the AI Agent (translation)<br>- A spokesperson video<br>- A reference image of the local presenter<br>- n8n instance must be on **HTTPS**<br><br>### Need Help?<br>Join the [n8n Discord](https://discord.gg/n8n) or ask in the [Forum](https://community.n8n.io/)!<br><br>Happy Automating! |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Canvas documentation / example note |  |  | ### Example Input<br><br>**Source Video:**<br>An 8-second clip of a presenter speaking in English<br><br>**Local Presenter Image:**<br>A photo of the person who should appear in the localized video<br><br>**Target Language:**<br>Spanish |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Canvas documentation / configuration note |  |  | ## 1. Start & Configure<br>Click **Test Workflow** to run.<br><br>The **Set Fields** node defines:<br>- **target_language** — the language for the localized video (e.g. Spanish, Japanese, French) |
| Sticky Note - Read Files | n8n-nodes-base.stickyNote | Canvas documentation / input file note |  |  | ## 2. Load Files<br>Reads both input files in parallel:<br><br>**Source Video** (top branch)<br>- Output field: `video`<br>- The original spokesperson video<br>- Formats: MP4, MPEG, MOV, AVI, WMV, OGG<br><br>**Local Presenter Image** (bottom branch)<br>- Output field: `image`<br>- Reference photo of the local presenter<br>- Formats: JPG, JPEG, PNG, GIF, BMP, WebP<br>- Max size: 10 MB<br><br>Update the **File Path** in each node. |
| Sticky Note - Transcribe | n8n-nodes-base.stickyNote | Canvas documentation / transcription note |  |  | ## 3. Transcribe<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Transcribe Video** uses **Whisper Large V3** to extract the spoken text from the video.<br><br>Timestamps are included so the AI can preserve pacing during translation. |
| Sticky Note - Translate | n8n-nodes-base.stickyNote | Canvas documentation / translation note |  |  | ## 4. Translate<br>The **AI Agent** translates the transcript into the target language.<br><br>It preserves the natural tone and pacing of the original speech, adapting idioms and cultural references for the target audience. |
| Sticky Note - TTS | n8n-nodes-base.stickyNote | Canvas documentation / speech generation note |  |  | ## 5. Generate Dubbed Speech<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Generate Speech** uses **Qwen3** to create natural-sounding speech from the translated text.<br><br>Swap for **Clone a Voice** to preserve the original speaker's voice characteristics. |
| Sticky Note - Audio to Video | n8n-nodes-base.stickyNote | Canvas documentation / video generation note |  |  | ## 6. Generate Lip-Synced Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking-head video synced to the dubbed speech.<br><br>The local presenter image is used as the first frame to guide the visual appearance. |
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual entry point for testing |  | Set Fields | ## 1. Start & Configure<br>Click **Test Workflow** to run.<br><br>The **Set Fields** node defines:<br>- **target_language** — the language for the localized video (e.g. Spanish, Japanese, French) |
| Set Fields | n8n-nodes-base.set | Defines target language parameter | Manual Trigger | Read Source Video; Read Local Presenter Image | ## 1. Start & Configure<br>Click **Test Workflow** to run.<br><br>The **Set Fields** node defines:<br>- **target_language** — the language for the localized video (e.g. Spanish, Japanese, French) |
| Read Source Video | n8n-nodes-base.readWriteFile | Loads source video into binary property `video` | Set Fields | deAPI Transcribe Video | ## 2. Load Files<br>Reads both input files in parallel:<br><br>**Source Video** (top branch)<br>- Output field: `video`<br>- The original spokesperson video<br>- Formats: MP4, MPEG, MOV, AVI, WMV, OGG<br><br>**Local Presenter Image** (bottom branch)<br>- Output field: `image`<br>- Reference photo of the local presenter<br>- Formats: JPG, JPEG, PNG, GIF, BMP, WebP<br>- Max size: 10 MB<br><br>Update the **File Path** in each node. |
| Read Local Presenter Image | n8n-nodes-base.readWriteFile | Loads presenter image into binary property `image` | Set Fields | Merge Audio + Image | ## 2. Load Files<br>Reads both input files in parallel:<br><br>**Source Video** (top branch)<br>- Output field: `video`<br>- The original spokesperson video<br>- Formats: MP4, MPEG, MOV, AVI, WMV, OGG<br><br>**Local Presenter Image** (bottom branch)<br>- Output field: `image`<br>- Reference photo of the local presenter<br>- Formats: JPG, JPEG, PNG, GIF, BMP, WebP<br>- Max size: 10 MB<br><br>Update the **File Path** in each node. |
| deAPI Transcribe Video | n8n-nodes-deapi.deapi | Transcribes spoken content from source video | Read Source Video | Extract from File | ## 3. Transcribe<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Transcribe Video** uses **Whisper Large V3** to extract the spoken text from the video.<br><br>Timestamps are included so the AI can preserve pacing during translation. |
| Extract from File | n8n-nodes-base.extractFromFile | Extracts transcript text into JSON field `text` | deAPI Transcribe Video | AI Agent | ## 3. Transcribe<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Transcribe Video** uses **Whisper Large V3** to extract the spoken text from the video.<br><br>Timestamps are included so the AI can preserve pacing during translation. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Translates transcript into target language | Extract from File; Anthropic Chat Model | deAPI Generate Speech | ## 4. Translate<br>The **AI Agent** translates the transcript into the target language.<br><br>It preserves the natural tone and pacing of the original speech, adapting idioms and cultural references for the target audience. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Provides Claude model to the AI Agent |  | AI Agent | ## 4. Translate<br>The **AI Agent** translates the transcript into the target language.<br><br>It preserves the natural tone and pacing of the original speech, adapting idioms and cultural references for the target audience. |
| deAPI Generate Speech | n8n-nodes-deapi.deapi | Converts translated text to dubbed speech audio | AI Agent | Merge Audio + Image | ## 5. Generate Dubbed Speech<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Generate Speech** uses **Qwen3** to create natural-sounding speech from the translated text.<br><br>Swap for **Clone a Voice** to preserve the original speaker's voice characteristics. |
| Merge Audio + Image | n8n-nodes-base.merge | Combines dubbed audio with presenter image | deAPI Generate Speech; Read Local Presenter Image | deAPI Generate From Audio | ## 6. Generate Lip-Synced Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking-head video synced to the dubbed speech.<br><br>The local presenter image is used as the first frame to guide the visual appearance. |
| deAPI Generate From Audio | n8n-nodes-deapi.deapi | Generates lip-synced talking-head video from audio and image | Merge Audio + Image |  | ## 6. Generate Lip-Synced Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking-head video synced to the dubbed speech.<br><br>The local presenter image is used as the first frame to guide the visual appearance. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like **Multilingual Video Localization**.
   - Ensure your n8n instance supports:
     - AI/LangChain nodes
     - the deAPI node package
     - HTTPS access, as noted in the workflow comments

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Keep default settings.
   - This will be the workflow’s starting point.

3. **Add a Set node named `Set Fields`**
   - Node type: **Set**
   - Add one string field:
     - `target_language`
     - example value: `Spanish`
   - Connect:
     - `Manual Trigger` → `Set Fields`

4. **Add the source video file reader**
   - Node type: **Read/Write Files from Disk**
   - Rename it to **Read Source Video**
   - Configure it to read a file from disk.
   - Set the file path to your source video, for example:
     - `/path/to/your/spokesperson-video.mp4`
   - In options, set the binary output property name to:
     - `video`
   - Connect:
     - `Set Fields` → `Read Source Video`

5. **Add the local presenter image reader**
   - Node type: **Read/Write Files from Disk**
   - Rename it to **Read Local Presenter Image**
   - Configure it to read a file from disk.
   - Set the file path to your image, for example:
     - `/path/to/your/local-presenter.jpg`
   - In options, set the binary output property name to:
     - `image`
   - Connect:
     - `Set Fields` → `Read Local Presenter Image`

6. **Create deAPI credentials**
   - Add credentials for the deAPI node.
   - Use your deAPI account/API key as required by the node.
   - Reuse the same credential in all deAPI nodes below.

7. **Add the transcription node**
   - Node type: **deAPI**
   - Rename it to **deAPI Transcribe Video**
   - Configure:
     - Resource: `video`
     - Operation: `transcribe`
     - Source: `binary`
     - Binary Property Name: `video`
     - Wait Timeout: `120`
   - Connect:
     - `Read Source Video` → `deAPI Transcribe Video`

8. **Add transcript extraction**
   - Node type: **Extract From File**
   - Rename it to **Extract from File**
   - Configure:
     - Operation: `text`
     - Destination Key: `text`
   - Connect:
     - `deAPI Transcribe Video` → `Extract from File`

9. **Create Anthropic credentials**
   - Add **Anthropic API** credentials in n8n.
   - Ensure the account has access to the Claude model you plan to use.

10. **Add the AI Agent**
    - Node type: **AI Agent**
    - Rename it to **AI Agent**
    - Set prompt mode to define/custom prompt.
    - In the main prompt text, use:
      ```text
      Translate the following transcript into {{ $('Set Fields').item.json.target_language }}.

      Return ONLY the translated text, without timestamps, line numbers, or formatting. Preserve the natural pacing and tone of the original speech.

      Transcript:
      {{ $json.text }}
      ```
    - Add this system message:
      ```text
      You are a professional translator specializing in video localization. Your goal is to produce natural-sounding translations that work well when spoken aloud.

      Key principles:
      - Preserve the tone, energy, and intent of the original speech
      - Adapt idioms and cultural references for the target audience
      - Keep sentences at a similar length to the original for lip-sync compatibility
      - Use natural spoken language, not formal written style
      - Return ONLY the translated text — no explanations, notes, or formatting
      ```
    - Connect:
      - `Extract from File` → `AI Agent`

11. **Add the Anthropic model node**
    - Node type: **Anthropic Chat Model**
    - Rename it to **Anthropic Chat Model**
    - Select model:
      - `claude-opus-4-6`
    - Attach Anthropic credentials.
    - Connect using the AI model port:
      - `Anthropic Chat Model` → `AI Agent` via the `ai_languageModel` connection

12. **Add dubbed speech generation**
    - Node type: **deAPI**
    - Rename it to **deAPI Generate Speech**
    - Configure:
      - Resource: `audio`
      - Operation: `generateSpeech`
      - Model: `Qwen3_TTS_12Hz_1_7B_CustomVoice`
      - Text:
        `={{ $json.output }}`
      - Qwen3 Language:
        `={{ $('Set Fields').item.json.target_language }}`
      - Wait Timeout: `120`
    - Attach the same deAPI credentials.
    - Connect:
      - `AI Agent` → `deAPI Generate Speech`

13. **Add a Merge node**
    - Node type: **Merge**
    - Rename it to **Merge Audio + Image**
    - Configure:
      - Mode: `combine`
      - Combine By: `combineByPosition`
    - Connect:
      - `deAPI Generate Speech` → `Merge Audio + Image` input 1/main input 0
      - `Read Local Presenter Image` → `Merge Audio + Image` input 2/main input 1

14. **Add final video generation**
    - Node type: **deAPI**
    - Rename it to **deAPI Generate From Audio**
    - Configure:
      - Resource: `video`
      - Operation: `generateFromAudio`
      - Prompt:
        `A person speaking naturally to the camera, subtle head movements and facial expressions, professional lighting, medium close-up shot, steady camera`
      - Options:
        - `frames = 241`
        - `firstFrame = image`
        - `waitTimeout = 300`
    - Attach deAPI credentials.
    - Connect:
      - `Merge Audio + Image` → `deAPI Generate From Audio`

15. **Validate binary property names**
    - Confirm the source video node outputs binary as `video`.
    - Confirm the presenter image node outputs binary as `image`.
    - Confirm the final deAPI video node expects the first frame from `image`.
    - If you rename these binary properties, update all dependent fields accordingly.

16. **Run a test execution**
    - Click **Test Workflow**.
    - Watch the execution in order:
      - Manual Trigger
      - Set Fields
      - File readers
      - deAPI transcription
      - text extraction
      - AI translation
      - speech generation
      - merge
      - video generation

17. **Verify expected outputs at each stage**
    - `Set Fields`: JSON contains `target_language`
    - `Read Source Video`: binary contains `video`
    - `Read Local Presenter Image`: binary contains `image`
    - `Extract from File`: JSON contains transcript under `text`
    - `AI Agent`: translated text typically appears under `output`
    - `deAPI Generate Speech`: audio asset returned
    - `Merge Audio + Image`: combined item contains both audio and image-related data
    - `deAPI Generate From Audio`: final localized video returned

18. **Adjust for production use if needed**
    - Replace `Manual Trigger` with a webhook, form, scheduler, or upstream workflow trigger.
    - Replace hardcoded file paths with uploaded binary data, cloud storage reads, or workflow inputs.
    - Add error handling nodes for:
      - missing files
      - unsupported language values
      - deAPI timeout/retry logic
      - validation of transcript and translated text before TTS

## Credential and setup requirements
- **deAPI account**
  - Required for transcription, speech generation, and video generation.
- **Anthropic account**
  - Required for the AI translation step.
- **n8n HTTPS deployment**
  - Explicitly noted in the workflow comments as a requirement.

## Input expectations
- **Source video**
  - Intended formats per note: MP4, MPEG, MOV, AVI, WMV, OGG
- **Presenter image**
  - Intended formats per note: JPG, JPEG, PNG, GIF, BMP, WebP
  - Practical limit noted: 10 MB
- **Target language**
  - Plain language string such as Spanish, Japanese, or French

## Important implementation constraints
- The AI Agent references the node name `Set Fields` directly in expressions. If you rename this node, update all expressions using it.
- The TTS node expects translated text in `$json.output`. If your AI node version returns a different field, adjust accordingly.
- The transcript extraction assumes deAPI transcription output can be interpreted as text by `Extract from File`. If deAPI returns JSON instead, replace this node with a JSON parsing step.
- The final generation depends on having both audio and image available in a single merged item.

## Sub-workflow setup
This workflow does **not** contain any Execute Workflow / sub-workflow nodes. There are no nested workflow dependencies to create.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| deAPI account required for transcription, TTS, and video generation | https://deapi.ai |
| deAPI product documentation referenced in workflow comments | https://docs.deapi.ai |
| Anthropic account required for translation via AI Agent | Anthropic API credentials in n8n |
| n8n instance must be on HTTPS | Workflow requirement noted in overview comment |
| Need help from the n8n community | https://discord.gg/n8n |
| Need help from the n8n forum | https://community.n8n.io/ |
| Example input: an 8-second English presenter clip, a local presenter photo, target language Spanish | Workflow canvas note |
| Voice cloning is suggested as an alternative to standard TTS if preserving original voice characteristics matters | Mentioned in the speech-generation note |