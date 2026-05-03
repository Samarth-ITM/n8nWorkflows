Turn Ideas into Movies with DeepSeek, RunwayML, ElevenLabs & Creatomate

https://n8nworkflows.xyz/workflows/turn-ideas-into-movies-with-deepseek--runwayml--elevenlabs---creatomate-8222


# Turn Ideas into Movies with DeepSeek, RunwayML, ElevenLabs & Creatomate

### 1. Workflow Overview

This workflow, titled **"Turn Ideas into Movies with DeepSeek, RunwayML, ElevenLabs & Creatomate"**, automates the process of transforming a creative story idea into a fully rendered cinematic video. It is designed for content creators, marketers, and storytellers who want to rapidly prototype and produce multimedia stories using AI-driven tools. The workflow integrates multiple AI services and platforms to handle narrative generation, image synthesis, voiceover creation, music composition, and final video rendering, orchestrating these complex processes into a unified pipeline.

The workflow logic is grouped into the following functional blocks:

- **1.1 Creative Brief Reception & Archiving**: Intake and archive the initial story concept.
- **1.2 Story & Scene Generation (Narrative Processing)**: Generate a structured cinematic story and scenes.
- **1.3 Visual Content Creation**: Produce image prompts, generate concept art, assemble storyboards, and create video clips.
- **1.4 Audio Content Creation**: Compose background music and generate voiceover lines.
- **1.5 Post-Production Orchestration**: Synchronize all media assets and render the final video.
- **1.6 Final Delivery & Announcement**: Fetch the rendered video and announce it on Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Creative Brief Reception & Archiving

- **Overview:** This block receives the initial story idea via a webhook, archives the creative brief in a Notion database, and prepares the input for downstream processing.
- **Nodes Involved:** 
  - Creative Brief Intake (Webhook)
  - Creative Archive (Notion)
  - Production Sync Hub (Merge)
- **Node Details:**

  - **Creative Brief Intake**  
    - *Type:* Webhook  
    - *Role:* Entry point for receiving the story concept via HTTP POST (e.g., from Slack).  
    - *Config:* Path set to `/storytelling`, accepts POST requests.  
    - *Input/Output:* Receives raw text; outputs JSON with the creative brief.  
    - *Edge Cases:* Invalid payloads, unauthorized access, or webhook downtime.  
    - *Sticky Note:* Central node for collecting the creative brief and archiving project data. Connected to Creative Archive → Production Sync Hub.

  - **Creative Archive**  
    - *Type:* Notion  
    - *Role:* Archives the incoming creative brief into a Notion page for record-keeping.  
    - *Config:* Uses Notion API, stores text content and titles pages with brief metadata.  
    - *Input/Output:* Input from webhook; outputs JSON with archived data.  
    - *Edge Cases:* Notion API quota limits, authentication errors.  
    - *Sticky Note:* Central node for collecting the creative brief and archiving project data.

  - **Production Sync Hub**  
    - *Type:* Merge  
    - *Role:* Combines multiple inputs downstream; here, it collects data from archive and later nodes.  
    - *Config:* Combine mode set to "combine by position".  
    - *Edge Cases:* Missing inputs, merge conflicts.

---

#### 1.2 Story & Scene Generation (Narrative Processing)

- **Overview:** Generates a cinematic story outline and six detailed scenes using a Narrative LLM (DeepSeek), then distributes this text to branches for visuals, music, and voiceover.
- **Nodes Involved:** 
  - Narrative LLM Core (DeepSeek)
  - Narrative Director (Chain LLM)
  - Visual LLM Core (DeepSeek)
  - Visual Art Director (Chain LLM)
  - Visual Parser (Output Parser Structured)
  - Score LLM Core (DeepSeek)
  - Score Parser (Output Parser Structured)
  - Score Composer (Chain LLM)
  - Voice LLM Core (DeepSeek)
  - Voice Parser (Output Parser Structured)
  - Narration Artist (Chain LLM)
- **Node Details:**

  - **Narrative LLM Core**  
    - *Type:* DeepSeek LLM Chat  
    - *Role:* Underlying language model API call for narrative generation.  
    - *Config:* API key for DeepSeek.  
    - *Edge Cases:* API limits, network issues.

  - **Narrative Director**  
    - *Type:* Chain LLM  
    - *Role:* Processes the creative brief into a 30-word story summary and six concise scene descriptions.  
    - *Config:* Scripted prompt with strict formatting rules and tone instructions; inputs raw text from webhook.  
    - *Output:* Structured story and scenes text.  
    - *Edge Cases:* Prompt formatting errors, unexpected LLM output.  
    - *Sticky Note:* Generates the base story (script) split into scenes from the initial prompt; connects to Visual + Voice branches.

  - **Visual LLM Core**  
    - *Type:* DeepSeek LLM Chat  
    - *Role:* Language model for visual prompt generation.  
    - *Config:* Same DeepSeek API key.  
    - *Edge Cases:* Same as Narrative LLM Core.

  - **Visual Art Director**  
    - *Type:* Chain LLM  
    - *Role:* Converts story text into six vivid image prompts for storyboard scenes.  
    - *Config:* Strict JSON output format with six image prompt objects, emphasis on character consistency and cinematic style.  
    - *Input:* Story text from Narrative Director.  
    - *Output:* JSON array of image prompts.  
    - *Edge Cases:* Parsing errors if output is malformed.  
    - *Sticky Note:* Transforms narrative text into visual prompts; connected to Scene Breakdown → Concept Art Studio.

  - **Visual Parser**  
    - *Type:* Output Parser Structured  
    - *Role:* Parses the JSON image prompt array from Visual Art Director.  
    - *Config:* Example JSON schema with six image_prompt objects for validation.  
    - *Edge Cases:* Malformed JSON leading to parsing failure.

  - **Score LLM Core**  
    - *Type:* DeepSeek LLM Chat  
    - *Role:* Language model generating a music prompt based on story tone and mood.  
    - *Edge Cases:* Same as above.

  - **Score Parser**  
    - *Type:* Output Parser Structured  
    - *Role:* Parses the JSON music prompt.  
    - *Config:* Single field music_prompt in JSON.  
    - *Edge Cases:* Parsing errors.

  - **Score Composer**  
    - *Type:* Chain LLM  
    - *Role:* Generates a descriptive prompt for background music with pacing matching the story.  
    - *Output:* JSON with music_prompt string.  
    - *Edge Cases:* Unexpected output format.  
    - *Sticky Note:* Creates background music and soundtrack; connected to Orchestration Engine → Track Data Manager.

  - **Voice LLM Core**  
    - *Type:* DeepSeek LLM Chat  
    - *Role:* Language model for voiceover script generation.  
    - *Edge Cases:* Same as above.

  - **Voice Parser**  
    - *Type:* Output Parser Structured  
    - *Role:* Parses JSON array of voiceover lines for each scene.  
    - *Config:* Six voiceover objects with "voiceover" text.  
    - *Edge Cases:* Parsing failures.

  - **Narration Artist**  
    - *Type:* Chain LLM  
    - *Role:* Generates voiceover scripts for six scenes, each under 5 seconds.  
    - *Output:* JSON array of voiceover lines.  
    - *Sticky Note:* Generates realistic voice-over from story text; connected to Dialogue Segmenter → Line Delivery Manager.

---

#### 1.3 Visual Content Creation

- **Overview:** Takes image prompts, generates concept art images, waits for rendering, fetches assets, assembles storyboards into videos, and manages rendering queues.
- **Nodes Involved:** 
  - Scene Breakdown (Split Out)
  - Concept Art Studio (HTTP Request - Replicate API)
  - Render Queue (Wait)
  - Asset Fetcher (HTTP Request)
  - Storyboard Assembler (HTTP Request - RunwayML API)
  - Encoding Queue (Wait)
  - Media Retriever (HTTP Request)
  - Metadata Curator (Set)
- **Node Details:**

  - **Scene Breakdown**  
    - *Type:* Split Out  
    - *Role:* Splits the JSON array of image prompts into individual items for processing.  
    - *Edge Cases:* Empty arrays or malformed input.

  - **Concept Art Studio**  
    - *Type:* HTTP Request  
    - *Role:* Sends image prompt to Replicate API (black-forest-labs/flux-pro) to generate artwork images.  
    - *Config:* POST with width=768, height=1280, prompt from image_prompt field; uses API key.  
    - *Edge Cases:* API rate limiting, generation failures.

  - **Render Queue**  
    - *Type:* Wait  
    - *Role:* Pauses workflow (30 seconds) to allow time for image generation processing.  
    - *Edge Cases:* Insufficient wait time causing premature downstream requests.

  - **Asset Fetcher**  
    - *Type:* HTTP Request  
    - *Role:* Fetches generated image asset from URL output by Concept Art Studio.  
    - *Edge Cases:* Broken URLs or network errors.

  - **Storyboard Assembler**  
    - *Type:* HTTP Request  
    - *Role:* Uses RunwayML API to convert images into short video clips (5 sec each) with cinematic aspect ratio.  
    - *Config:* POST with model=gen4_turbo, ratio=720:1280, duration=5, promptText from input, promptImage from fetched asset; includes API key and version header.  
    - *Edge Cases:* API downtime, invalid inputs.

  - **Encoding Queue**  
    - *Type:* Wait  
    - *Role:* Waits 4 minutes to allow video encoding to complete.  
    - *Edge Cases:* Delays longer than wait time causing missing assets.

  - **Media Retriever**  
    - *Type:* HTTP Request  
    - *Role:* Fetches video task status and final video URLs from RunwayML API by task ID.  
    - *Edge Cases:* Task failure or timeout.

  - **Metadata Curator**  
    - *Type:* Set  
    - *Role:* Assigns video URLs to indexed keys (video1…video6) for downstream orchestration.  
    - *Edge Cases:* Missing or partial video URLs.

---

#### 1.4 Audio Content Creation

- **Overview:** Processes voiceover scripts, segments dialogue lines, manages batch processing for synthesis, uploads voice files to Dropbox, and prepares links for usage.
- **Nodes Involved:** 
  - Dialogue Segmenter (Split Out)
  - Line Delivery Manager (Split In Batches)
  - Voice Synthesis Studio (HTTP Request - ElevenLabs API)
  - Voiceover Collector (Merge)
  - Voice Upload Loop (Split In Batches)
  - Dropbox Uploader
  - Dropbox Link Generator (HTTP Request)
  - Voiceover Mapper (Set)
  - File Renamer (NoOp)
- **Node Details:**

  - **Dialogue Segmenter**  
    - *Type:* Split Out  
    - *Role:* Splits voiceover JSON array into individual lines.  
    - *Edge Cases:* Empty or malformed input.

  - **Line Delivery Manager**  
    - *Type:* Split In Batches  
    - *Role:* Processes voice lines in batches (default batch size) to control API load.  
    - *Outputs:* Two outputs: one loops back for upload; the other to synthesis.  
    - *Edge Cases:* Batch size misconfiguration causing rate limits.

  - **Voice Synthesis Studio**  
    - *Type:* HTTP Request  
    - *Role:* Sends text lines to ElevenLabs text-to-speech API to generate mp3 voice files.  
    - *Config:* POST with text from voiceover JSON, model_id set to "eleven_multilingual_v2", output format mp3_44100_128.  
    - *Edge Cases:* API quota, authentication failure.

  - **Voiceover Collector**  
    - *Type:* Merge  
    - *Role:* Collects synthesized voice files from batches into one stream for upload.  
    - *Edge Cases:* Missing items or partial merges.

  - **Voice Upload Loop**  
    - *Type:* Split In Batches  
    - *Role:* Controls upload of voice files one by one to Dropbox.  
    - *Edge Cases:* Upload failures or Dropbox API errors.

  - **Dropbox Uploader**  
    - *Type:* Dropbox  
    - *Role:* Uploads mp3 files to Dropbox folder `/voiceovers/` with indexed filenames.  
    - *Edge Cases:* Dropbox API rate limits, auth issues.

  - **Dropbox Link Generator**  
    - *Type:* HTTP Request  
    - *Role:* Requests temporary shareable download links for uploaded voice files using Dropbox API.  
    - *Edge Cases:* Link expiration or API errors.

  - **Voiceover Mapper**  
    - *Type:* Set  
    - *Role:* Maps generated Dropbox links to indexed voiceover keys (voiceover0…voiceover5).  
    - *Edge Cases:* Missing or broken links.

  - **File Renamer**  
    - *Type:* NoOp  
    - *Role:* Acts as a pass-through node for controlling flow.  
    - *Edge Cases:* None.

  - *Sticky Note:* Generates realistic voice-over; connected to Dialogue Segmenter → Line Delivery Manager → Voice Synthesis Studio.

---

#### 1.5 Post-Production Orchestration

- **Overview:** Aggregates all media assets (voiceovers, videos, background music), synchronizes them for final rendering using Creatomate, and manages rendering queues.
- **Nodes Involved:** 
  - Orchestration Engine (HTTP Request - Replicate API for music)
  - Track Data Manager (Set)
  - Production Sync Hub (Merge)
  - Post-Production Orchestrator (Code)
  - Final Cut Renderer (HTTP Request - Creatomate API)
  - Rendering Buffer (Wait)
- **Node Details:**

  - **Orchestration Engine**  
    - *Type:* HTTP Request  
    - *Role:* Submits music prompt to Replicate API to generate background music track.  
    - *Edge Cases:* API limits, generation errors.

  - **Track Data Manager**  
    - *Type:* Set  
    - *Role:* Assigns the music stream URL to the `bgm` key.  
    - *Edge Cases:* Missing URL from API response.

  - **Production Sync Hub**  
    - *Type:* Merge  
    - *Role:* Combines video URLs, voiceover links, and background music into a single dataset.  
    - *Edge Cases:* Partial merges or missing data.

  - **Post-Production Orchestrator**  
    - *Type:* Code  
    - *Role:* Custom JavaScript consolidates all assets into a single JSON payload with keys: voiceover0-5, video1-6, bgm.  
    - *Edge Cases:* Missing keys, null values.

  - **Final Cut Renderer**  
    - *Type:* HTTP Request  
    - *Role:* Sends consolidated data to Creatomate API for final video rendering using a specified template.  
    - *Config:* Uses JSON body with template_id and modifications mapping assets to template sources.  
    - *Edge Cases:* API errors, invalid template ID.

  - **Rendering Buffer**  
    - *Type:* Wait  
    - *Role:* Waits 2 minutes for rendering to complete.  
    - *Edge Cases:* Render time variations.

  - *Sticky Note:* Main synchronization hub where text, visuals, voice, and music meet; connected to Post-Production Orchestrator → Final Cut Renderer.

---

#### 1.6 Final Delivery & Announcement

- **Overview:** Retrieves the final rendered video from Creatomate, then posts a release announcement to a Slack channel.
- **Nodes Involved:** 
  - Final Delivery Fetcher (HTTP Request)
  - Release Announcement (Slack)
- **Node Details:**

  - **Final Delivery Fetcher**  
    - *Type:* HTTP Request  
    - *Role:* Polls Creatomate API for final render status and fetches the video URL.  
    - *Edge Cases:* Render failure, API downtime.

  - **Release Announcement**  
    - *Type:* Slack  
    - *Role:* Sends a message to a configured Slack channel with the video link.  
    - *Config:* Uses Slack API with specific channel ID and text containing video URL.  
    - *Edge Cases:* Slack API rate limits, invalid channel ID.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                           | Input Node(s)                         | Output Node(s)                                  | Sticky Note                                                                                           |
|-----------------------|----------------------------------|-----------------------------------------|-------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Creative Brief Intake  | Webhook                          | Receives story concept                   | —                                   | Creative Archive                               | Central node for collecting creative brief and archiving project data.                             |
| Creative Archive       | Notion                           | Archives creative brief                  | Creative Brief Intake               | Narrative Director, Production Sync Hub        | Central node for collecting the creative brief and archiving project data.                         |
| Narrative LLM Core     | DeepSeek LLM Chat                | Language model for narrative generation | Narrative Director                 | Narrative Director                              |                                                                                                    |
| Narrative Director     | Chain LLM                       | Generates story summary & scenes        | Creative Archive, Narrative LLM Core| Visual Art Director, Score Composer, Narration Artist | Generates base story split into scenes; connected to Visual + Voice branches.                      |
| Visual LLM Core        | DeepSeek LLM Chat                | Language model for visual prompt gen    | Visual Art Director               | Visual Art Director                              |                                                                                                    |
| Visual Art Director    | Chain LLM                       | Creates image prompts from story text   | Narrative Director, Visual LLM Core| Visual Parser                                  | Transforms narrative into visual prompts; connected to Scene Breakdown → Concept Art Studio.       |
| Visual Parser          | Output Parser Structured         | Parses visual prompts JSON               | Visual Art Director               | Scene Breakdown                                 |                                                                                                    |
| Scene Breakdown        | Split Out                       | Splits image prompt array                | Visual Parser                    | Concept Art Studio                              |                                                                                                    |
| Concept Art Studio     | HTTP Request (Replicate API)     | Generates concept art images             | Scene Breakdown                 | Render Queue                                    | Transforms narrative text into visual prompts; connected to Scene Breakdown → Concept Art Studio.  |
| Render Queue           | Wait                            | Waits for image generation               | Concept Art Studio              | Asset Fetcher                                   |                                                                                                    |
| Asset Fetcher          | HTTP Request                    | Fetches generated image asset            | Render Queue                   | Storyboard Assembler                            |                                                                                                    |
| Storyboard Assembler   | HTTP Request (RunwayML API)      | Converts images to video clips           | Asset Fetcher                  | Encoding Queue                                  |                                                                                                    |
| Encoding Queue         | Wait                            | Waits for video encoding                 | Storyboard Assembler           | Media Retriever                                 |                                                                                                    |
| Media Retriever        | HTTP Request                    | Retrieves video URLs                      | Encoding Queue                | Metadata Curator                                |                                                                                                    |
| Metadata Curator       | Set                             | Maps video URLs to keys                   | Media Retriever               | Production Sync Hub                             |                                                                                                    |
| Score LLM Core         | DeepSeek LLM Chat                | Generates music prompt                    | Narrative Director             | Score Composer                                  |                                                                                                    |
| Score Parser           | Output Parser Structured         | Parses music prompt JSON                  | Score Composer                | Orchestration Engine                            | Creates background music and soundtrack; connected to Orchestration Engine → Track Data Manager.   |
| Score Composer         | Chain LLM                       | Creates music prompt from story text     | Score LLM Core                | Orchestration Engine                            |                                                                                                    |
| Orchestration Engine   | HTTP Request (Replicate API)     | Generates background music                | Score Composer                | Track Data Manager                              |                                                                                                    |
| Track Data Manager     | Set                             | Assigns music stream URL                  | Orchestration Engine          | Production Sync Hub                             |                                                                                                    |
| Voice LLM Core         | DeepSeek LLM Chat                | Generates voiceover scripts               | Narrative Director             | Narration Artist                                |                                                                                                    |
| Voice Parser           | Output Parser Structured         | Parses voiceover JSON array               | Narration Artist              | Dialogue Segmenter                              |                                                                                                    |
| Narration Artist       | Chain LLM                       | Creates voiceover lines                    | Voice LLM Core                | Dialogue Segmenter                              | Generates realistic voice-over; connected to Dialogue Segmenter → Line Delivery Manager.           |
| Dialogue Segmenter     | Split Out                       | Splits voiceover lines                    | Narration Artist              | Line Delivery Manager                           |                                                                                                    |
| Line Delivery Manager  | Split In Batches                | Manages batch processing of voice lines  | Dialogue Segmenter            | Voiceover Collector, Voice Synthesis Studio    |                                                                                                    |
| Voice Synthesis Studio | HTTP Request (ElevenLabs API)    | Synthesizes voiceover audio               | Line Delivery Manager          | Line Delivery Manager                           |                                                                                                    |
| Voiceover Collector    | Merge                          | Collects synthesized voice audio          | Line Delivery Manager          | Voice Upload Loop                               |                                                                                                    |
| Voice Upload Loop      | Split In Batches                | Uploads voice files batch-wise            | Voiceover Collector           | Dropbox Uploader, File Renamer                  |                                                                                                    |
| Dropbox Uploader       | Dropbox                        | Uploads mp3 voice files                    | Voice Upload Loop             | Dropbox Link Generator                          |                                                                                                    |
| Dropbox Link Generator | HTTP Request                   | Generates temporary links for voice files| Dropbox Uploader             | Voiceover Mapper                                |                                                                                                    |
| Voiceover Mapper       | Set                             | Maps voiceover URLs                       | Dropbox Link Generator        | Production Sync Hub                             |                                                                                                    |
| File Renamer           | NoOp                           | Pass-through for flow control              | Voice Upload Loop             | Voice Upload Loop                               |                                                                                                    |
| Production Sync Hub    | Merge                          | Synchronizes all media assets              | Creative Archive, Metadata Curator, Track Data Manager, Voiceover Mapper | Post-Production Orchestrator         | Main synchronization hub; connected to Post-Production Orchestrator → Final Cut Renderer.          |
| Post-Production Orchestrator | Code                      | Aggregates all assets into single JSON    | Production Sync Hub           | Final Cut Renderer                              |                                                                                                    |
| Final Cut Renderer     | HTTP Request (Creatomate API)   | Sends data for final video rendering       | Post-Production Orchestrator  | Rendering Buffer                                |                                                                                                    |
| Rendering Buffer       | Wait                            | Waits for final rendering                  | Final Cut Renderer            | Final Delivery Fetcher                          |                                                                                                    |
| Final Delivery Fetcher | HTTP Request                   | Retrieves final rendered video URL         | Rendering Buffer             | Release Announcement                            |                                                                                                    |
| Release Announcement   | Slack                         | Sends Slack message with video link         | Final Delivery Fetcher        | —                                              |                                                                                                    |
| Sticky Note            | Sticky Note                    | Workflow feature and architecture notes    | —                           | —                                              | Features: Story generation, visuals, voice, music, automated rendering, Slack trigger.              |
| Sticky Note1           | Sticky Note                    | Describes narrative generation block       | —                           | —                                              | Generates base story split into scenes from initial prompt; connected to Narrative Director.       |
| Sticky Note2           | Sticky Note                    | Describes visual prompt and production flow| —                           | —                                              | Transforms narrative text into visual prompts; connected to Scene Breakdown → Concept Art Studio.  |
| Sticky Note3           | Sticky Note                    | Describes music generation block            | —                           | —                                              | Creates background music and soundtrack; connected to Orchestration Engine → Track Data Manager.   |
| Sticky Note4           | Sticky Note                    | Describes voiceover generation block        | —                           | —                                              | Generates realistic voice-over from story text; connected to Dialogue Segmenter → Line Delivery Manager. |
| Sticky Note5           | Sticky Note                    | Describes post-production orchestration     | —                           | —                                              | Main synchronization hub; connected to Post-Production Orchestrator → Final Cut Renderer.          |
| Sticky Note6           | Sticky Note                    | Describes brief intake and archiving block  | —                           | —                                              | Central node for collecting brief and archiving data; connected to Creative Archive → Production Sync Hub. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Creative Brief Intake"**  
   - Type: Webhook  
   - Path: `/storytelling`  
   - Method: POST  
   - Purpose: Receive story idea input (e.g., Slack message).  

2. **Create Notion Node: "Creative Archive"**  
   - Type: Notion  
   - Credentials: Connect your Notion API key.  
   - Configure to create a page with title based on webhook data (user_name, channel_name).  
   - Store the story text in a content block.  
   - Connect: Webhook → Creative Archive.

3. **Create DeepSeek API Credential**  
   - Store API key for DeepSeek.  

4. **Create DeepSeek LLM Chat Node: "Narrative LLM Core"**  
   - Credentials: Use DeepSeek API key.  
   - Purpose: Provide core narrative generation model.  

5. **Create Chain LLM Node: "Narrative Director"**  
   - Configure prompt to:  
     - Take raw text from webhook JSON.  
     - Generate 30-word story summary + 6 scene descriptions.  
     - Follow strict formatting rules (main character naming, tone, output format).  
   - Connect: Creative Archive → Narrative Director, Narrative LLM Core → Narrative Director.

6. **Create DeepSeek LLM Chat Node: "Visual LLM Core"**  
   - Credentials: DeepSeek API key.  

7. **Create Chain LLM Node: "Visual Art Director"**  
   - Prompt: Use story text to generate 6 image prompts in JSON format.  
   - Strong instructions on visual style and character continuity.  
   - Connect: Narrative Director → Visual Art Director, Visual LLM Core → Visual Art Director.

8. **Create Output Parser Structured Node: "Visual Parser"**  
   - JSON schema: Array of six objects with `image_prompt` string.  
   - Connect: Visual Art Director → Visual Parser.

9. **Create Split Out Node: "Scene Breakdown"**  
   - Field to split: `output` (the array of image prompts).  
   - Connect: Visual Parser → Scene Breakdown.

10. **Create HTTP Request Node: "Concept Art Studio"**  
    - URL: `https://api.replicate.com/v1/models/black-forest-labs/flux-pro/predictions`  
    - Method: POST  
    - Body: JSON with width=768, height=1280, prompt from `image_prompt`.  
    - Auth: Replicate API key via HTTP Header Auth.  
    - Connect: Scene Breakdown → Concept Art Studio.

11. **Create Wait Node: "Render Queue"**  
    - Wait for 30 seconds.  
    - Connect: Concept Art Studio → Render Queue.

12. **Create HTTP Request Node: "Asset Fetcher"**  
    - URL: From Concept Art Studio output (image URL).  
    - Connect: Render Queue → Asset Fetcher.

13. **Create HTTP Request Node: "Storyboard Assembler"**  
    - URL: `https://api.dev.runwayml.com/v1/image_to_video`  
    - Method: POST  
    - Body: JSON with model="gen4_turbo", ratio="720:1280", duration=5, promptText from input, promptImage from Asset Fetcher output.  
    - Auth: RunwayML API key via HTTP Header Auth.  
    - Header: X-Runway-Version=2024-11-06  
    - Connect: Asset Fetcher → Storyboard Assembler.

14. **Create Wait Node: "Encoding Queue"**  
    - Wait for 4 minutes for encoding.  
    - Connect: Storyboard Assembler → Encoding Queue.

15. **Create HTTP Request Node: "Media Retriever"**  
    - URL: `https://api.dev.runwayml.com/v1/tasks/{{ $json.id }}`  
    - Method: GET  
    - Auth: RunwayML API key.  
    - Connect: Encoding Queue → Media Retriever.

16. **Create Set Node: "Metadata Curator"**  
    - Assign video URLs from Media Retriever output to keys video1...video6.  
    - Connect: Media Retriever → Metadata Curator.

17. **Create DeepSeek LLM Chat Node: "Score LLM Core"**  
    - Credentials: DeepSeek API key.  

18. **Create Chain LLM Node: "Score Composer"**  
    - Prompt: Generate JSON music prompt based on story text.  
    - Connect: Narrative Director → Score Composer, Score LLM Core → Score Composer.

19. **Create Output Parser Structured Node: "Score Parser"**  
    - JSON schema for music_prompt string.  
    - Connect: Score Composer → Score Parser.

20. **Create HTTP Request Node: "Orchestration Engine"**  
    - URL: Replicate API for music generation.  
    - Method: POST  
    - Body: JSON with version ID, input prompt from music_prompt, duration=30.  
    - Auth: Replicate API key.  
    - Connect: Score Parser → Orchestration Engine.

21. **Create Set Node: "Track Data Manager"**  
    - Assign bgm key to music stream URL from Orchestration Engine output.  
    - Connect: Orchestration Engine → Track Data Manager.

22. **Create DeepSeek LLM Chat Node: "Voice LLM Core"**  
    - Credentials: DeepSeek API key.

23. **Create Chain LLM Node: "Narration Artist"**  
    - Prompt: Generate voiceover scripts (6 lines, max 5 sec each) from story text.  
    - Connect: Narrative Director → Narration Artist, Voice LLM Core → Narration Artist.

24. **Create Output Parser Structured Node: "Voice Parser"**  
    - JSON schema array of six voiceover objects.  
    - Connect: Narration Artist → Voice Parser.

25. **Create Split Out Node: "Dialogue Segmenter"**  
    - Split voiceover array into lines.  
    - Connect: Voice Parser → Dialogue Segmenter.

26. **Create Split In Batches Node: "Line Delivery Manager"**  
    - Batch process voice lines for API calls and upload management.  
    - Connect: Dialogue Segmenter → Line Delivery Manager.

27. **Create HTTP Request Node: "Voice Synthesis Studio"**  
    - URL: ElevenLabs TTS API  
    - Method: POST  
    - Body: Text from voiceover line, model_id `eleven_multilingual_v2`, output mp3_44100_128.  
    - Auth: ElevenLabs API key.  
    - Connect: Line Delivery Manager → Voice Synthesis Studio.

28. **Create Merge Node: "Voiceover Collector"**  
    - Collect voice mp3 results.  
    - Connect: Line Delivery Manager → Voiceover Collector.

29. **Create Split In Batches Node: "Voice Upload Loop"**  
    - Batch upload voice files one by one to Dropbox.  
    - Connect: Voiceover Collector → Voice Upload Loop.

30. **Create Dropbox Node: "Dropbox Uploader"**  
    - Upload mp3 files to `/voiceovers/voice_{index}.mp3`.  
    - Credentials: Dropbox API key.  
    - Connect: Voice Upload Loop → Dropbox Uploader.

31. **Create HTTP Request Node: "Dropbox Link Generator"**  
    - URL: `https://api.dropboxapi.com/2/files/get_temporary_link`  
    - Method: POST  
    - Body: JSON with path from uploaded file.  
    - Auth: OAuth2 API for Dropbox.  
    - Connect: Dropbox Uploader → Dropbox Link Generator.

32. **Create Set Node: "Voiceover Mapper"**  
    - Map Dropbox links to voiceover0...voiceover5 keys.  
    - Connect: Dropbox Link Generator → Voiceover Mapper.

33. **Create NoOp Node: "File Renamer"**  
    - Pass-through for flow control.  
    - Connect: Voice Upload Loop → File Renamer.

34. **Create Merge Node: "Production Sync Hub"**  
    - Combine outputs from Creative Archive, Metadata Curator (videos), Track Data Manager (music), and Voiceover Mapper.  
    - Connect: Creative Archive, Metadata Curator, Track Data Manager, Voiceover Mapper → Production Sync Hub.

35. **Create Code Node: "Post-Production Orchestrator"**  
    - Aggregate all assets into a single JSON object with keys for voiceovers, videos, and bgm.  
    - Connect: Production Sync Hub → Post-Production Orchestrator.

36. **Create HTTP Request Node: "Final Cut Renderer"**  
    - URL: Creatomate API endpoint `/v1/renders`  
    - Method: POST  
    - Body: JSON with template_id and modifications mapping all assets to template sources.  
    - Auth: Creatomate API key.  
    - Connect: Post-Production Orchestrator → Final Cut Renderer.

37. **Create Wait Node: "Rendering Buffer"**  
    - Wait 2 minutes for rendering.  
    - Connect: Final Cut Renderer → Rendering Buffer.

38. **Create HTTP Request Node: "Final Delivery Fetcher"**  
    - URL: `https://api.creatomate.com/v1/renders/{{ $json.id }}`  
    - Method: GET  
    - Auth: Creatomate API key.  
    - Connect: Rendering Buffer → Final Delivery Fetcher.

39. **Create Slack Node: "Release Announcement"**  
    - Send message with video URL to designated Slack channel.  
    - Credentials: Slack API key.  
    - Connect: Final Delivery Fetcher → Release Announcement.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Features include story generation (DeepSeek LLM), visuals (Replicate + RunwayML), voice & music (ElevenLabs + Replicate), automated rendering with Creatomate, all triggered via Slack command `/render`. | Sticky Note at workflow overview area.                                                            |
| Narrative generation block strictly formats story output with a 30-word summary and six scene descriptions to ensure consistent downstream processing. | Sticky Note1 — Narrative Director area.                                                           |
| Visual prompt generation emphasizes cinematic style continuity and main character consistency to enable coherent storyboards. | Sticky Note2 — Visual Art Director area.                                                          |
| Background music generated by Replicate AI music models, matching story mood and pacing.                      | Sticky Note3 — Score Composer area.                                                                |
| Voiceover generation uses ElevenLabs TTS API, with batch processing and Dropbox storage for access and reuse. | Sticky Note4 — Narration Artist area.                                                             |
| Post-production orchestrates all media assets into Creatomate templates for final video rendering.            | Sticky Note5 — Production Sync Hub and Post-Production Orchestrator area.                          |
| Creative brief and project data archived in Notion for traceability and project management.                   | Sticky Note6 — Creative Archive area.                                                             |
| Slack integration for notifications; ensure Slack channel ID and API credentials are correctly configured.    | Release Announcement node setup.                                                                  |
| API keys for DeepSeek, Replicate, RunwayML, ElevenLabs, Creatomate, Dropbox, and Slack must be provisioned and tested for each service prior to workflow execution. | Credential configuration section.                                                                 |

---

This documentation fully describes all functional nodes, data flows, and key configurations necessary to understand, reproduce, or modify the "Turn Ideas into Movies" workflow. It also highlights potential failure points and integration dependencies for robust operation.