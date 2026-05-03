Clone a voice and create talking avatar videos using Anthropic and deAPI

https://n8nworkflows.xyz/workflows/clone-a-voice-and-create-talking-avatar-videos-using-anthropic-and-deapi-14190


# Clone a voice and create talking avatar videos using Anthropic and deAPI

# 1. Workflow Overview

This workflow, titled **“Voice Clone Talking Avatar”**, creates a talking-avatar video from three core inputs:

- a short **reference audio clip** used to clone a voice,
- a **first frame image** used as the avatar’s visual anchor,
- and user-defined **speech text** plus a **video prompt**.

It combines **deAPI** services for voice cloning, prompt boosting, and video generation with an **Anthropic** model used through an n8n AI Agent to improve the final video prompt.

## 1.1 Start and User Input Definition

The workflow begins manually and sets the three main user-controlled values:

- `text`: what the avatar should say,
- `video_prompt`: visual description of the scene,
- `lang`: language for voice cloning.

This block provides all non-binary input parameters used later in the workflow.

## 1.2 File Ingestion

Two local files are loaded in parallel:

- the reference voice sample into binary field `audio`,
- the first frame image into binary field `image`.

These files become the binary payloads required by deAPI voice cloning and video generation.

## 1.3 Voice Cloning and Speech Synthesis

The reference audio and text are sent to deAPI to generate a new spoken audio track in the cloned voice.

The result is a synthesized audio binary that will drive the talking avatar video.

## 1.4 Merge and AI Prompt Optimization

The generated speech audio and the first frame image are logically combined, then passed into an AI Agent.

The AI Agent uses:
- an **Anthropic Chat Model** as its language model,
- and the **deAPI Video Prompt Booster** as a tool.

Its role is to transform the initial prompt into a more suitable talking-avatar generation prompt, emphasizing lip sync, facial motion, framing, and consistency with the first frame image.

## 1.5 Video Generation

Finally, deAPI generates a talking avatar video from:

- the cloned speech audio,
- the first frame image,
- and the AI-optimized prompt.

The output is a synthesized avatar video intended to match the provided text and image while using the cloned voice.

---

# 2. Block-by-Block Analysis

## Block 1 — Start & Configuration

### Overview

This block defines the workflow entry point and the structured text inputs used throughout the rest of the automation. It is the only execution trigger in the workflow.

### Nodes Involved

- Manual Trigger
- Set Fields

### Node Details

#### 1. Manual Trigger

- **Type and technical role:** `n8n-nodes-base.manualTrigger`
  - Starts the workflow manually from the editor.
- **Configuration choices:**
  - No custom parameters are configured.
- **Key expressions or variables used:**
  - None.
- **Input and output connections:**
  - No input.
  - Outputs to **Set Fields**.
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - No runtime failure expected beyond standard manual execution issues.
  - This trigger is not suitable for unattended or production event-driven execution.
- **Sub-workflow reference:**
  - None.

#### 2. Set Fields

- **Type and technical role:** `n8n-nodes-base.set`
  - Creates structured JSON fields used later by expression references.
- **Configuration choices:**
  - Defines three string fields:
    - `text`
    - `video_prompt`
    - `lang`
  - The node does not appear to remove all previous fields explicitly; it simply assigns new values.
- **Key expressions or variables used:**
  - Produces:
    - `json.text`
    - `json.video_prompt`
    - `json.lang`
- **Input and output connections:**
  - Input from **Manual Trigger**
  - Outputs to:
    - **Read Reference Audio**
    - **Read First Frame Image**
- **Version-specific requirements:**
  - `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - If later expressions expect these fields and they are renamed or removed, downstream nodes will fail.
  - Empty `text` may cause poor or rejected voice/video generation.
  - Unsupported or ambiguous `lang` values may produce poor cloning output or API-level validation errors.
- **Sub-workflow reference:**
  - None.

---

## Block 2 — Local File Loading

### Overview

This block reads the two required local assets from disk: the reference audio sample and the avatar’s first frame image. These are loaded into separate binary properties for downstream use.

### Nodes Involved

- Read Reference Audio
- Read First Frame Image

### Node Details

#### 3. Read Reference Audio

- **Type and technical role:** `n8n-nodes-base.readWriteFile`
  - Reads a local file into a binary property.
- **Configuration choices:**
  - `fileSelector` points to `/path/to/your/reference-audio.mp3`
  - Binary output property is set to `audio`
- **Key expressions or variables used:**
  - Produces `binary.audio`
- **Input and output connections:**
  - Input from **Set Fields**
  - Output to **deAPI Clone a Voice**
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Invalid file path
  - File not found
  - File permission denied
  - Unsupported format or oversized file for downstream API
  - Running n8n in an environment where the path does not exist inside the container/host
- **Sub-workflow reference:**
  - None.

#### 4. Read First Frame Image

- **Type and technical role:** `n8n-nodes-base.readWriteFile`
  - Reads a local image file into a binary property.
- **Configuration choices:**
  - `fileSelector` points to `/path/to/your/avatar-image.png`
  - Binary output property is set to `image`
- **Key expressions or variables used:**
  - Produces `binary.image`
- **Input and output connections:**
  - Input from **Set Fields**
  - Output to **Merge**
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Invalid file path
  - Missing file
  - Permission issues
  - Unsupported image format for downstream processing
  - Very large images may increase upload time or API rejection risk
- **Sub-workflow reference:**
  - None.

---

## Block 3 — Voice Cloning

### Overview

This block sends the reference audio, target text, and language to deAPI to synthesize new speech in the cloned voice. It is the first external API-dependent processing stage.

### Nodes Involved

- deAPI Clone a Voice

### Node Details

#### 5. deAPI Clone a Voice

- **Type and technical role:** `n8n-nodes-deapi.deapi`
  - Calls deAPI audio functionality to clone a voice and produce generated speech audio.
- **Configuration choices:**
  - `resource`: `audio`
  - Uses:
    - `lang` from **Set Fields**
    - `text` from **Set Fields**
    - `refAudio` from **Read Reference Audio** binary field `audio`
  - `options.waitTimeout`: `120`
- **Key expressions or variables used:**
  - `={{ $('Set Fields').item.json.lang }}`
  - `={{ $('Set Fields').item.json.text }}`
  - `={{ $('Read Reference Audio').item.binary.audio }}`
- **Input and output connections:**
  - Input from **Read Reference Audio**
  - Output to **Merge**
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires configured **deAPI** credentials.
- **Edge cases or potential failure types:**
  - Invalid deAPI credentials
  - API timeout if generation exceeds 120 seconds
  - Unsupported or poor-quality reference audio
  - Reference clip too short, too long, noisy, or empty
  - Expression failure if upstream binary field `audio` is missing
  - API-side validation errors for language or text content
- **Sub-workflow reference:**
  - None.

---

## Block 4 — Merge and AI Prompt Optimization

### Overview

This block joins the generated voice output with the image branch, then uses an AI Agent to produce a stronger video prompt tailored specifically for talking-avatar generation. The AI Agent relies on both a chat model and a deAPI prompt-boosting tool.

### Nodes Involved

- Merge
- AI Agent
- Anthropic Chat Model
- Video prompt booster in deAPI

### Node Details

#### 6. Merge

- **Type and technical role:** `n8n-nodes-base.merge`
  - Combines the two branches into a single item so downstream processing can proceed with synchronized context.
- **Configuration choices:**
  - `mode`: `combine`
  - `combineBy`: `combineByPosition`
- **Key expressions or variables used:**
  - None internally, but its success depends on both incoming branches emitting matching-position items.
- **Input and output connections:**
  - Input 1 from **deAPI Clone a Voice**
  - Input 2 from **Read First Frame Image**
  - Output to **AI Agent**
- **Version-specific requirements:**
  - `typeVersion: 3.2`
- **Edge cases or potential failure types:**
  - If one branch produces zero items, downstream processing may not occur as expected.
  - Position-based merging can misalign data if item counts differ in future modifications.
  - Binary properties may not automatically become the source used in downstream expressions if nodes continue referencing original upstream nodes directly.
- **Sub-workflow reference:**
  - None.

#### 7. AI Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`
  - Orchestrates LLM reasoning and tool usage to create a final boosted video prompt.
- **Configuration choices:**
  - `promptType`: `define`
  - Main user instruction asks for an optimized prompt for a talking avatar video.
  - Injects:
    - speech text from **Set Fields**
    - original visual prompt from **Set Fields**
  - System message defines strict behavior:
    - analyze tone,
    - refine for avatar behavior,
    - use `videoPromptBooster`,
    - return only final boosted prompt text.
- **Key expressions or variables used:**
  - `{{ $('Set Fields').item.json.text }}`
  - `{{ $('Set Fields').item.json.video_prompt }}`
  - Output later consumed as `{{$json.output}}`
- **Input and output connections:**
  - Main input from **Merge**
  - AI language model input from **Anthropic Chat Model**
  - AI tool input from **Video prompt booster in deAPI**
  - Main output to **deAPI Generate From Audio**
- **Version-specific requirements:**
  - `typeVersion: 1.7`
  - Requires compatible LangChain/AI features in the n8n version used.
- **Edge cases or potential failure types:**
  - Invalid Anthropic credentials
  - Tool call failure if deAPI tool credentials are invalid
  - LLM may fail to call the tool, though the prompt strongly instructs it to
  - Output may include extra text if the model does not strictly follow instructions
  - Missing upstream text/prompt values will reduce prompt quality or break expressions
- **Sub-workflow reference:**
  - None.

#### 8. Anthropic Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`
  - Provides the language model backend used by the AI Agent.
- **Configuration choices:**
  - Model selected: `claude-opus-4-6`
  - No notable advanced options configured.
- **Key expressions or variables used:**
  - None.
- **Input and output connections:**
  - No main input.
  - Connected to **AI Agent** via `ai_languageModel`.
- **Version-specific requirements:**
  - `typeVersion: 1.3`
  - Requires valid **Anthropic API** credentials.
  - Availability of `claude-opus-4-6` depends on account access and current Anthropic offerings.
- **Edge cases or potential failure types:**
  - Invalid credentials
  - Model unavailability
  - Quota/rate limit errors
  - Account access restrictions on the selected model
- **Sub-workflow reference:**
  - None.

#### 9. Video prompt booster in deAPI

- **Type and technical role:** `n8n-nodes-deapi.deapiTool`
  - Exposes a deAPI prompt-boosting function as a tool callable by the AI Agent.
- **Configuration choices:**
  - `resource`: `prompt`
  - `operation`: `boostVideo`
  - Prompt is provided through `$fromAI(...)`, meaning the AI Agent supplies the input dynamically.
  - `options.binaryPropertyName` references the first frame image binary from **Read First Frame Image**
- **Key expressions or variables used:**
  - `={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('Prompt', \`\`, 'string') }}`
  - `={{ $('Read First Frame Image').item.binary.image }}`
- **Input and output connections:**
  - No main input.
  - Connected to **AI Agent** via `ai_tool`.
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires valid **deAPI** credentials.
- **Edge cases or potential failure types:**
  - Invalid credentials
  - Tool invocation issues from the agent
  - If the image binary reference is invalid, visual-context boosting may fail
  - The parameter name `binaryPropertyName` is populated with a binary object expression rather than a literal property name; this likely works only if the node expects the binary payload directly, but it is a configuration area worth validating in a live test
- **Sub-workflow reference:**
  - None.

---

## Block 5 — Talking Avatar Video Generation

### Overview

This block creates the final video using the AI-generated prompt, the cloned speech audio, and the first frame image. It is the final external generation stage and the workflow’s terminal processing step.

### Nodes Involved

- deAPI Generate From Audio

### Node Details

#### 10. deAPI Generate From Audio

- **Type and technical role:** `n8n-nodes-deapi.deapi`
  - Calls deAPI video generation from audio, producing a talking-avatar-style result.
- **Configuration choices:**
  - `resource`: `video`
  - `operation`: `generateFromAudio`
  - `prompt` uses the AI Agent output
  - `audioBinaryProperty` references generated binary data from **deAPI Clone a Voice**
  - `options.firstFrame` references binary image from **Read First Frame Image**
  - `options.frames`: `241`
  - `options.waitTimeout`: `300`
- **Key expressions or variables used:**
  - `={{ $json.output }}`
  - `={{ $('deAPI Clone a Voice').item.binary.data }}`
  - `={{ $('Read First Frame Image').item.binary.image }}`
- **Input and output connections:**
  - Input from **AI Agent**
  - No downstream node; terminal node.
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires valid **deAPI** credentials.
- **Edge cases or potential failure types:**
  - Invalid or expired deAPI credentials
  - API timeout if generation exceeds 300 seconds
  - Invalid prompt if AI Agent output is empty or malformed
  - Missing cloned audio binary
  - Missing first frame image binary
  - High frame counts may increase generation time and failure likelihood
  - If the output of **AI Agent** is not stored in `json.output` in a future version, this expression may break
- **Sub-workflow reference:**
  - None.

---

## Non-executable Annotation Nodes

The following nodes are documentation-only nodes and do not affect execution:

- Sticky Note - Overview
- Sticky Note - Trigger
- Sticky Note - Read Files
- Sticky Note - Clone
- Sticky Note - Generate Video
- Sticky Note - Example

These nodes visually explain the workflow and should be preserved if the workflow is meant to remain understandable to end users.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual documentation for overall workflow purpose and requirements |  |  | ## Try It Out!<br>### Clone a voice from a short audio clip and generate a talking avatar video with a custom first frame.<br><br>This workflow reads a reference audio file and a first frame image, clones the voice to speak new text, and generates a talking avatar video using the image as the opening frame.<br><br>### How it works<br>1. **Manual Trigger** starts the workflow<br>2. **Set Fields** defines the text for the avatar to speak and the video prompt<br>3. **Read Reference Audio** loads a short voice sample into the `audio` binary field<br>4. **Read First Frame Image** loads the avatar image into the `image` binary field<br>5. **deAPI Clone a Voice** clones the voice and generates new speech<br>6. **Merge** combines the cloned audio and the first frame image<br>7. **AI Agent** crafts a talking-avatar-optimized prompt and boosts it with the **deAPI Video Prompt Booster** tool, using the first frame image for visual context<br>8. **deAPI Generate From Audio** creates a talking avatar video synced to the cloned speech, using the AI-crafted prompt and first frame image<br><br>### Requirements<br>- [deAPI](https://deapi.ai) account for voice cloning, prompt boosting, and video generation<br>- Anthropic account for the AI Agent<br>- A short reference audio file (3-10 seconds, MP3/WAV/FLAC/OGG)<br>- A first frame image for the avatar (PNG/JPG)<br>- n8n instance must be on **HTTPS**<br><br>### Need Help?<br>Join the [n8n Discord](https://discord.gg/n8n) or ask in the [Forum](https://community.n8n.io/)!<br><br>Happy Automating! |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Visual explanation of startup and input fields |  |  | ## 1. Start & Configure<br>Click **Test Workflow** to run.<br><br>The **Set Fields** node defines:<br>- **text** — what the avatar will say<br>- **video_prompt** — visual description for the avatar video<br>- **lang** — language for voice cloning |
| Sticky Note - Read Files | n8n-nodes-base.stickyNote | Visual explanation of input file requirements |  |  | ## 2. Load Files<br>Reads both input files in parallel into separate binary fields:<br><br>**Reference Audio** (top branch)<br>- Output field: `audio`<br>- Duration: 3-10 seconds<br>- Formats: MP3, WAV, FLAC, OGG, M4A<br>- Max size: 10 MB<br><br>**First Frame Image** (bottom branch)<br>- Output field: `image`<br>- The avatar image shown as the video's opening frame<br>- Formats: PNG, JPG<br><br>Update the **File Path** in each node. |
| Sticky Note - Clone | n8n-nodes-base.stickyNote | Visual explanation of the voice cloning step |  |  | ## 3. Clone Voice<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Clone a Voice** uses **Qwen3 TTS VoiceClone** to clone the voice from the reference audio. |
| Sticky Note - Generate Video | n8n-nodes-base.stickyNote | Visual explanation of merge, AI boost, and video generation |  |  | ## 4. Merge, AI Boost & Generate Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Merge** combines the cloned audio (`audio`) with the first frame image (`image`) into a single item.<br><br>**AI Agent** takes the user's video prompt and speech text, crafts a talking-avatar-optimized prompt focusing on lip sync, facial expressions, and natural movement, then uses the **Video Prompt Booster** tool with the first frame image for final optimization.<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking avatar video:<br>- Audio in `audio` drives the speech sync<br>- Image in `image` sets the opening frame<br>- AI-crafted prompt guides the visual scene |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Example input values for testing |  |  | ### Example Input<br><br>**Reference Audio:**<br>A 5-second clip of someone speaking naturally<br><br>**First Frame Image:**<br>A photo or AI-generated image of the avatar/presenter<br><br>**Text to Speak:**<br>"Welcome to our channel! Today we're going to explore the latest advancements in artificial intelligence and how they can help your business grow."<br><br>**Video Prompt:**<br>"A professional presenter speaking confidently in a modern, well-lit studio with a blurred tech background"<br><br>**Language:**<br>English |
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Set Fields | ## 1. Start & Configure<br>Click **Test Workflow** to run.<br><br>The **Set Fields** node defines:<br>- **text** — what the avatar will say<br>- **video_prompt** — visual description for the avatar video<br>- **lang** — language for voice cloning |
| Set Fields | n8n-nodes-base.set | Defines text, prompt, and language inputs | Manual Trigger | Read Reference Audio; Read First Frame Image | ## 1. Start & Configure<br>Click **Test Workflow** to run.<br><br>The **Set Fields** node defines:<br>- **text** — what the avatar will say<br>- **video_prompt** — visual description for the avatar video<br>- **lang** — language for voice cloning |
| Read Reference Audio | n8n-nodes-base.readWriteFile | Loads local reference audio into binary field `audio` | Set Fields | deAPI Clone a Voice | ## 2. Load Files<br>Reads both input files in parallel into separate binary fields:<br><br>**Reference Audio** (top branch)<br>- Output field: `audio`<br>- Duration: 3-10 seconds<br>- Formats: MP3, WAV, FLAC, OGG, M4A<br>- Max size: 10 MB<br><br>**First Frame Image** (bottom branch)<br>- Output field: `image`<br>- The avatar image shown as the video's opening frame<br>- Formats: PNG, JPG<br><br>Update the **File Path** in each node. |
| Read First Frame Image | n8n-nodes-base.readWriteFile | Loads local first-frame image into binary field `image` | Set Fields | Merge | ## 2. Load Files<br>Reads both input files in parallel into separate binary fields:<br><br>**Reference Audio** (top branch)<br>- Output field: `audio`<br>- Duration: 3-10 seconds<br>- Formats: MP3, WAV, FLAC, OGG, M4A<br>- Max size: 10 MB<br><br>**First Frame Image** (bottom branch)<br>- Output field: `image`<br>- The avatar image shown as the video's opening frame<br>- Formats: PNG, JPG<br><br>Update the **File Path** in each node. |
| deAPI Clone a Voice | n8n-nodes-deapi.deapi | Clones the voice and synthesizes speech audio | Read Reference Audio | Merge | ## 3. Clone Voice<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Clone a Voice** uses **Qwen3 TTS VoiceClone** to clone the voice from the reference audio. |
| Merge | n8n-nodes-base.merge | Combines cloned audio branch and image branch into one item | deAPI Clone a Voice; Read First Frame Image | AI Agent | ## 4. Merge, AI Boost & Generate Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Merge** combines the cloned audio (`audio`) with the first frame image (`image`) into a single item.<br><br>**AI Agent** takes the user's video prompt and speech text, crafts a talking-avatar-optimized prompt focusing on lip sync, facial expressions, and natural movement, then uses the **Video Prompt Booster** tool with the first frame image for final optimization.<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking avatar video:<br>- Audio in `audio` drives the speech sync<br>- Image in `image` sets the opening frame<br>- AI-crafted prompt guides the visual scene |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generates an optimized talking-avatar prompt using an LLM and a deAPI tool | Merge; Anthropic Chat Model; Video prompt booster in deAPI | deAPI Generate From Audio | ## 4. Merge, AI Boost & Generate Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Merge** combines the cloned audio (`audio`) with the first frame image (`image`) into a single item.<br><br>**AI Agent** takes the user's video prompt and speech text, crafts a talking-avatar-optimized prompt focusing on lip sync, facial expressions, and natural movement, then uses the **Video Prompt Booster** tool with the first frame image for final optimization.<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking avatar video:<br>- Audio in `audio` drives the speech sync<br>- Image in `image` sets the opening frame<br>- AI-crafted prompt guides the visual scene |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Provides the Claude model used by the AI Agent |  | AI Agent | ## 4. Merge, AI Boost & Generate Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Merge** combines the cloned audio (`audio`) with the first frame image (`image`) into a single item.<br><br>**AI Agent** takes the user's video prompt and speech text, crafts a talking-avatar-optimized prompt focusing on lip sync, facial expressions, and natural movement, then uses the **Video Prompt Booster** tool with the first frame image for final optimization.<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking avatar video:<br>- Audio in `audio` drives the speech sync<br>- Image in `image` sets the opening frame<br>- AI-crafted prompt guides the visual scene |
| Video prompt booster in deAPI | n8n-nodes-deapi.deapiTool | Tool callable by the AI Agent to strengthen the final video prompt |  | AI Agent | ## 4. Merge, AI Boost & Generate Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Merge** combines the cloned audio (`audio`) with the first frame image (`image`) into a single item.<br><br>**AI Agent** takes the user's video prompt and speech text, crafts a talking-avatar-optimized prompt focusing on lip sync, facial expressions, and natural movement, then uses the **Video Prompt Booster** tool with the first frame image for final optimization.<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking avatar video:<br>- Audio in `audio` drives the speech sync<br>- Image in `image` sets the opening frame<br>- AI-crafted prompt guides the visual scene |
| deAPI Generate From Audio | n8n-nodes-deapi.deapi | Generates the final talking avatar video from audio, image, and optimized prompt | AI Agent |  | ## 4. Merge, AI Boost & Generate Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>**Merge** combines the cloned audio (`audio`) with the first frame image (`image`) into a single item.<br><br>**AI Agent** takes the user's video prompt and speech text, crafts a talking-avatar-optimized prompt focusing on lip sync, facial expressions, and natural movement, then uses the **Video Prompt Booster** tool with the first frame image for final optimization.<br><br>**Generate From Audio** uses **LTX-2.3 22B** to create a talking avatar video:<br>- Audio in `audio` drives the speech sync<br>- Image in `image` sets the opening frame<br>- AI-crafted prompt guides the visual scene |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild procedure for recreating this workflow manually in n8n.

## Prerequisites

1. Have an **n8n instance running on HTTPS**.
2. Create and verify:
   - a **deAPI** account
   - an **Anthropic** account
3. Prepare two files accessible to the n8n runtime:
   - a short reference audio clip, ideally **3–10 seconds**
   - a first frame image in **PNG or JPG**
4. Confirm that file paths are valid from the machine/container where n8n runs.

## Step-by-step build

### 1. Create the trigger node

1. Add a **Manual Trigger** node.
2. Leave default settings unchanged.
3. This will be the workflow’s start node.

### 2. Create the input definition node

1. Add a **Set** node after **Manual Trigger**.
2. Name it **Set Fields**.
3. Add three string fields:
   - `text`
   - `video_prompt`
   - `lang`
4. Example values:
   - `text`:  
     `Welcome to our channel! Today we're going to explore the latest advancements in artificial intelligence and how they can help your business grow.`
   - `video_prompt`:  
     `A professional presenter speaking confidently in a modern, well-lit studio with a blurred tech background`
   - `lang`:  
     `English`
5. Connect **Manual Trigger → Set Fields**.

### 3. Create the reference audio file reader

1. Add a **Read/Write Files from Disk** node.
2. Name it **Read Reference Audio**.
3. Configure it to read a file from disk.
4. Set the file path to your actual reference audio file, for example:
   - `/path/to/your/reference-audio.mp3`
5. In options, set the binary output field to:
   - `audio`
6. Connect **Set Fields → Read Reference Audio**.

### 4. Create the first frame image reader

1. Add another **Read/Write Files from Disk** node.
2. Name it **Read First Frame Image**.
3. Configure it to read a file from disk.
4. Set the file path to your actual image file, for example:
   - `/path/to/your/avatar-image.png`
5. In options, set the binary output field to:
   - `image`
6. Connect **Set Fields → Read First Frame Image**.

### 5. Create deAPI credentials

1. In n8n credentials, create a **deAPI** credential.
2. Authenticate it with your deAPI account details.
3. Save it with a name such as:
   - `deAPI account`

You will use this credential in both deAPI nodes and the deAPI tool node.

### 6. Create the voice cloning node

1. Add a **deAPI** node.
2. Name it **deAPI Clone a Voice**.
3. Set:
   - **Resource**: `audio`
4. Configure parameters:
   - **Language**: expression  
     `{{ $('Set Fields').item.json.lang }}`
   - **Text**: expression  
     `{{ $('Set Fields').item.json.text }}`
   - **Reference Audio**: expression  
     `{{ $('Read Reference Audio').item.binary.audio }}`
5. In options, set:
   - `waitTimeout = 120`
6. Attach the **deAPI account** credential.
7. Connect **Read Reference Audio → deAPI Clone a Voice**.

### 7. Create the merge node

1. Add a **Merge** node.
2. Name it **Merge**.
3. Configure:
   - **Mode**: `Combine`
   - **Combine By**: `Position`
4. Connect:
   - **deAPI Clone a Voice → Merge** on input 1
   - **Read First Frame Image → Merge** on input 2

### 8. Create Anthropic credentials

1. In n8n credentials, create an **Anthropic API** credential.
2. Authenticate it with your Anthropic API key.
3. Save it with a name such as:
   - `Anthropic account`

### 9. Create the language model node

1. Add an **Anthropic Chat Model** node.
2. Name it **Anthropic Chat Model**.
3. Select model:
   - `claude-opus-4-6`
4. Attach the **Anthropic account** credential.

Note: if that exact model is unavailable in your environment, choose the nearest supported Claude model and update documentation accordingly.

### 10. Create the deAPI prompt booster tool node

1. Add a **deAPI Tool** node.
2. Name it **Video prompt booster in deAPI**.
3. Set:
   - **Resource**: `prompt`
   - **Operation**: `boostVideo`
4. Configure the prompt input to come from AI:
   - Use the AI-provided prompt field via `$fromAI(...)`
5. Configure image-related option so the first frame image is available to the tool:
   - expression used in the source workflow:  
     `{{ $('Read First Frame Image').item.binary.image }}`
6. Attach the **deAPI account** credential.

Important validation note:
- The workflow uses the field `options.binaryPropertyName` with a binary expression. Depending on your n8n/deAPI node version, this may need to be adjusted if the node expects a binary property name rather than a binary object. Test this node carefully after creation.

### 11. Create the AI Agent node

1. Add an **AI Agent** node.
2. Name it **AI Agent**.
3. Set **Prompt Type** to:
   - `Define`
4. In the main prompt text, use:

   ```text
   Create an optimized video generation prompt for a talking avatar video.

   The user wants the avatar to say:
   "{{ $('Set Fields').item.json.text }}"

   Their initial video prompt idea:
   "{{ $('Set Fields').item.json.video_prompt }}"

   Use the videoPromptBooster tool to optimize your final prompt. Return ONLY the boosted prompt text, nothing else.
   ```

5. In **System Message**, use:

   ```text
   You are a specialist in creating prompts for talking avatar videos. Your goal is to produce a single, highly descriptive video generation prompt that will result in a realistic talking avatar.

   Key principles for talking avatar prompts:
   - Emphasize natural lip synchronization and mouth movements matching speech
   - Include subtle facial expressions, eye blinks, and micro-expressions
   - Describe natural head movements (slight nods, tilts) that accompany speech
   - Specify consistent lighting and camera angle (typically front-facing, head-and-shoulders)
   - Maintain visual consistency with the provided first frame image
   - Match the tone of the speech content (e.g., enthusiastic, professional, casual)

   Workflow:
   1. Analyze the speech text to understand the tone and energy level
   2. Refine the user's video prompt idea to focus on talking avatar qualities
   3. Use the videoPromptBooster tool to optimize your refined prompt
   4. Return ONLY the final boosted prompt text — no explanations, no formatting
   ```

6. Connect:
   - **Merge → AI Agent** as the main input
   - **Anthropic Chat Model → AI Agent** as the AI language model connection
   - **Video prompt booster in deAPI → AI Agent** as the AI tool connection

### 12. Create the final deAPI video generation node

1. Add another **deAPI** node.
2. Name it **deAPI Generate From Audio**.
3. Set:
   - **Resource**: `video`
   - **Operation**: `generateFromAudio`
4. Configure:
   - **Prompt**: expression  
     `{{ $json.output }}`
   - **Audio Binary Property**: expression  
     `{{ $('deAPI Clone a Voice').item.binary.data }}`
5. In options, configure:
   - **Frames**: `241`
   - **First Frame**: expression  
     `{{ $('Read First Frame Image').item.binary.image }}`
   - **Wait Timeout**: `300`
6. Attach the **deAPI account** credential.
7. Connect **AI Agent → deAPI Generate From Audio**.

### 13. Optional: add visual notes for maintainability

To mirror the original workflow layout, add sticky notes documenting:

- overall workflow purpose and requirements,
- trigger/input field meanings,
- file requirements,
- deAPI cloning stage,
- merge/AI/video generation stage,
- sample input values.

These notes are not required for execution but improve maintainability.

### 14. Test the workflow

1. Click **Test Workflow**.
2. Confirm these checkpoints:
   - **Set Fields** emits `text`, `video_prompt`, and `lang`
   - **Read Reference Audio** outputs `binary.audio`
   - **Read First Frame Image** outputs `binary.image`
   - **deAPI Clone a Voice** returns generated speech audio, likely in `binary.data`
   - **AI Agent** returns the optimized prompt in `json.output`
   - **deAPI Generate From Audio** completes successfully and returns video output

---

## Credential configuration summary

### deAPI credential

Used by:
- **deAPI Clone a Voice**
- **Video prompt booster in deAPI**
- **deAPI Generate From Audio**

Expected purpose:
- authenticate requests for voice cloning,
- prompt boosting,
- video generation.

### Anthropic credential

Used by:
- **Anthropic Chat Model**

Expected purpose:
- authenticate Claude model access for the AI Agent.

---

## Input/output expectations by major node

### Read Reference Audio
- **Input:** JSON from Set Fields
- **Output:** same item with `binary.audio`

### Read First Frame Image
- **Input:** JSON from Set Fields
- **Output:** same item with `binary.image`

### deAPI Clone a Voice
- **Input:** language, text, reference audio
- **Output:** generated speech audio, expected in a binary field such as `data`

### AI Agent
- **Input:** merged item, text, initial prompt, model, tool
- **Output:** optimized final prompt, expected in `json.output`

### deAPI Generate From Audio
- **Input:** optimized prompt, speech audio, first frame image
- **Output:** final generated video

---

## Implementation cautions

1. **File path visibility**
   - If n8n runs in Docker, local paths must exist inside the container, not just on the host machine.

2. **HTTPS requirement**
   - The sticky notes specify that the n8n instance must be on HTTPS. This is especially relevant when integrations or hosted APIs require secure callback/tooling behavior.

3. **Binary field assumptions**
   - The workflow assumes:
     - reference audio is in `binary.audio`
     - first frame image is in `binary.image`
     - cloned speech result is in `binary.data`
   - Verify these after running each node.

4. **Agent output field**
   - The final video node expects the AI Agent result in `json.output`.
   - If your AI Agent node version emits a different field, update the expression accordingly.

5. **Tool parameter validation**
   - The deAPI tool node’s image option should be tested carefully because the workflow uses a direct binary expression in a field named like a property-name option.

6. **Merge dependence**
   - The merge mode is position-based. If you later expand the workflow to multiple items, ensure both branches stay aligned.

7. **Timeout tuning**
   - `120` seconds for voice cloning and `300` seconds for video generation may need to be increased depending on account tier, API load, media size, or output complexity.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| deAPI account is required for voice cloning, prompt boosting, and video generation | [https://deapi.ai](https://deapi.ai) |
| deAPI technical reference for clone/prompt/video operations | [https://docs.deapi.ai](https://docs.deapi.ai) |
| Anthropic account is required for the AI Agent language model | Anthropic platform account |
| n8n help community | [https://community.n8n.io/](https://community.n8n.io/) |
| n8n Discord community | [https://discord.gg/n8n](https://discord.gg/n8n) |
| Example reference audio guidance | Short natural speech sample, ideally 3–10 seconds |
| Example first frame image guidance | PNG or JPG image used as avatar opening frame |
| Example speech text | “Welcome to our channel! Today we're going to explore the latest advancements in artificial intelligence and how they can help your business grow.” |
| Example video prompt | “A professional presenter speaking confidently in a modern, well-lit studio with a blurred tech background” |
| Example language | English |

If you want, I can also convert this analysis into a more compact **agent-ingestible YAML or Markdown spec** for automated workflow reconstruction.