Generate AI music and publish YouTube videos automatically with Blotato, OpenAI, ElevenLabs, and Shotstack

https://n8nworkflows.xyz/workflows/generate-ai-music-and-publish-youtube-videos-automatically-with-blotato--openai--elevenlabs--and-shotstack-14985


# Generate AI music and publish YouTube videos automatically with Blotato, OpenAI, ElevenLabs, and Shotstack

Good, I've verified all the connections and node relationships. Now let me produce the comprehensive document. 1. Workflow Overview

This workflow fully automates the creation and publishing of AI-generated music videos on YouTube. Starting from a user-facing form, it collects music preferences (duration, description, vocal type), then uses OpenAI to optimize those inputs into structured API parameters for ElevenLabs music generation. In parallel, it generates a YouTube thumbnail via Atlas Cloud's image generation API. The audio file is uploaded to Cloudinary for a persistent URL, and the thumbnail image URL is extracted after an asynchronous generation cycle. Both assets are merged and passed to Shotstack, which renders a final MP4 video (static thumbnail + audio). After polling for render completion, the final video URL is published to YouTube via the Blotato social media management node.

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Input Reception | Collect user preferences via an n8n form trigger |
| 2 | AI Music Prompt Optimization | Transform raw user input into structured ElevenLabs API parameters using an AI Agent with a JSON output parser |
| 3 | Music Generation & Audio Hosting | Call the ElevenLabs Music API to generate audio and upload it to Cloudinary for a persistent URL |
| 4 | AI Thumbnail Generation | Generate an optimized image prompt via AI, call Atlas Cloud image generation, poll for completion, and extract the final image URL |
| 5 | Asset Merge & URL Preparation | Merge audio and image URLs and normalize them into a clean payload for video rendering |
| 6 | Video Rendering | Submit audio + image to Shotstack for MP4 rendering and poll until the render completes |
| 7 | YouTube Publishing | Upload the rendered video to YouTube via Blotato with the SEO-optimized title |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Input Reception

**Overview:** A web-based form serves as the sole entry point. It collects three fields from the user: music duration (dropdown), a free-text music description (textarea), and the vocal type preference (radio button). Upon submission, the form triggers the workflow.

**Nodes Involved:**
- Music Generation Form

**Node Details:**

| Attribute | Value |
|-----------|-------|
| **Node** | Music Generation Form |
| **Type** | `n8n-nodes-base.formTrigger` (v2.5) |
| **Technical Role** | Webhook-based entry trigger; exposes a form UI at the n8n webhook endpoint |
| **Configuration** | Form title: *"AI Music Generator for YouTube"*; Form description: *"Generate custom AI music and publish it automatically to YouTube"*; Attribution append disabled; Three fields: (1) `duration` — dropdown with options 30s / 1min / 2min / 3min / 5min; (2) `description` — textarea with placeholder; (3) `lyrics_type` — radio with options Instrumental / With Lyrics |
| **Key Expressions / Variables** | Output fields: `$json.duration`, `$json.description`, `$json.lyrics_type` |
| **Input Connections** | None (entry node) |
| **Output Connections** | → Generate ElevenLabs Prompt |
| **Edge Cases / Failure Types** | Webhook URL must be reachable (check n8n base URL and tunnel settings if self-hosted); form fields are not validated server-side beyond type — an empty description will pass through; no file upload, so binary data issues are impossible at this stage |

---

#### Block 2 — AI Music Prompt Optimization

**Overview:** An AI Agent powered by OpenAI gpt-4o-mini takes the raw form data and rewrites it into four structured fields required by downstream nodes: an optimized English music prompt, duration in milliseconds, an instrumental flag, and an SEO-optimized YouTube title. A structured JSON output parser guarantees the response conforms to the expected schema.

**Nodes Involved:**
- Generate ElevenLabs Prompt
- OpenAI gpt-4o-mini
- JSON Output Parser

**Node Details:**

**Generate ElevenLabs Prompt**

| Attribute | Value |
|-----------|-------|
| **Type** | `@n8n/n8n-nodes-langchain.agent` (v3.1) |
| **Technical Role** | AI Agent that orchestrates prompt optimization; connects to a language model and an output parser |
| **Configuration** | Prompt type: *Define*; System message: instructs the agent to act as a "music prompt optimizer" and return a JSON with `prompt`, `duration_ms`, `force_instrumental`, and `youtube_title`; Output parser enabled (`hasOutputParser: true`) |
| **Key Expressions** | User prompt text: `"Durée : {{ $json.duration }}\nLyrics type : {{ $json.lyrics_type }}\nDescription :{{ $json.description }}"` |
| **Input Connections** | ← Music Generation Form (main input); ← OpenAI gpt-4o-mini (ai_languageModel); ← JSON Output Parser (ai_outputParser) |
| **Output Connections** | → Generate Music - ElevenLabs |
| **Output Structure** | `$json.output.prompt`, `$json.output.duration_ms`, `$json.output.force_instrumental`, `$json.output.youtube_title` |
| **Edge Cases / Failure Types** | If OpenAI returns malformed JSON, the output parser will throw a parsing error; temperature is 0.3 so output is relatively deterministic but can still occasionally omit a field; API rate limits or credential expiry will cause the agent to fail; French labels in the prompt ("Durée") may confuse the model for non-French contexts — the system message is in English so the model typically handles it |

**OpenAI gpt-4o-mini**

| Attribute | Value |
|-----------|-------|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) |
| **Technical Role** | Language model sub-node providing chat completion to the AI Agent |
| **Configuration** | Model: `gpt-4o-mini` (selected from list); Temperature: 0.3 (low, for structured/consistent output); No built-in tools enabled |
| **Credentials** | OpenAI API credentials (referenced as `OpenAi account`) |
| **Input Connections** | None (sub-node, connected via `ai_languageModel` to the Agent) |
| **Output Connections** | → Generate ElevenLabs Prompt (ai_languageModel channel) |
| **Edge Cases / Failure Types** | Invalid or expired OpenAI API key; quota exceeded; model name change by OpenAI; temperature too high could produce non-JSON output |

**JSON Output Parser**

| Attribute | Value |
|-----------|-------|
| **Type** | `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) |
| **Technical Role** | Enforces structured JSON output from the AI Agent; parses and validates against the provided JSON schema |
| **Configuration** | Schema type: *Manual*; Schema defines four properties: `prompt` (string, "Optimized music prompt"), `duration_ms` (number, "Duration in milliseconds"), `force_instrumental` (boolean, "True for instrumental, false for vocals"), `youtube_title` (string, "SEO-optimized YouTube title") |
| **Input Connections** | None (sub-node, connected via `ai_outputParser` to the Agent) |
| **Output Connections** | → Generate ElevenLabs Prompt (ai_outputParser channel) |
| **Edge Cases / Failure Types** | If the LLM omits any of the four required fields, the parser will raise a validation error; if the LLM outputs a string duration instead of a number, parsing fails; retry logic in the Agent may auto-correct, but not guaranteed |

---

#### Block 3 — Music Generation & Audio Hosting

**Overview:** The optimized parameters from Block 2 are sent to the ElevenLabs Music API, which returns a binary audio file. Simultaneously, the raw audio binary is uploaded to Cloudinary to obtain a persistent, publicly accessible URL required by Shotstack (which cannot consume transient n8n binary data).

**Nodes Involved:**
- Generate Music - ElevenLabs
- Upload an asset from file data

**Node Details:**

**Generate Music - ElevenLabs**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.4) |
| **Technical Role** | POST request to ElevenLabs Music generation endpoint; returns binary audio file |
| **Configuration** | URL: `https://api.elevenlabs.io/v1/music`; Method: POST; Auth: Generic Credential → Header Auth (expects header `xi-api-key` or similar); Response format: *File* (binary); Body parameters: `prompt` = `$('Generate ElevenLabs Prompt').item.json.output.prompt`, `duration_ms` = `$('Generate ElevenLabs Prompt').item.json.output.duration_ms`, `force_instrumental` = `$('Generate ElevenLabs Prompt').item.json.output.force_instrumental` |
| **Credentials** | HTTP Header Auth credential (referenced as `Header Auth account`) — must contain the ElevenLabs API key |
| **Input Connections** | ← Generate ElevenLabs Prompt |
| **Output Connections** | → Generate Image Prompt; → Upload an asset from file data |
| **Edge Cases / Failure Types** | ElevenLabs API key invalid or expired; music generation timeout (longer durations like 5min may take longer than default HTTP timeout); rate limits on the ElevenLabs free tier; response may be empty if prompt is rejected by content filters; binary data may exceed n8n memory limits for very long audio |

**Upload an asset from file data**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-cloudinary.cloudinary` (v1) |
| **Technical Role** | Uploads the binary audio file received from ElevenLabs to Cloudinary, returning a persistent public URL |
| **Configuration** | Operation: *Upload file*; Resource type: *video* (Cloudinary treats audio as video resource); No additional fields configured |
| **Credentials** | Cloudinary API credentials (referenced as `Cloudinary account 7`) — requires Cloud Name, API Key, and API Secret |
| **Input Connections** | ← Generate Music - ElevenLabs |
| **Output Connections** | → Merge Audio & Image Paths (input 0) |
| **Output Structure** | `$json.secure_url` or `$json.url` (Cloudinary public URL); `$json.duration` (may or may not be populated depending on Cloudinary metadata) |
| **Edge Cases / Failure Types** | Cloudinary credentials misconfigured; upload fails if binary is too large; if the previous node's binary data is named unexpectedly, Cloudinary may receive an empty payload; Cloudinary free tier storage limits; the `duration` field returned by Cloudinary may be null for some audio formats |

---

#### Block 4 — AI Thumbnail Generation

**Overview:** After the music is generated, an AI Agent creates a visual prompt based on the YouTube title and music description. This prompt is sent to Atlas Cloud's image generation model. Because image generation is asynchronous, the workflow extracts a prediction ID, waits 2 minutes, then polls the status endpoint. Once complete, a Code node extracts the final image URL from the response.

**Nodes Involved:**
- Generate Image Prompt
- OpenAI gpt-4o-mini Image
- Generate Thumbnail - Atlas Cloud
- Extract Prediction ID
- Wait
- Check Image Status
- Code clear imageUrl

**Node Details:**

**Generate Image Prompt**

| Attribute | Value |
|-----------|-------|
| **Type** | `@n8n/n8n-nodes-langchain.agent` (v3.1) |
| **Technical Role** | AI Agent that produces a detailed visual prompt for AI image generation, tailored for YouTube thumbnails |
| **Configuration** | Prompt type: *Define*; System message: instructs the agent to act as a "visual prompt expert" creating 16:9, 1280×720 YouTube thumbnail prompts with vivid color and composition details; No output parser attached (free-text output) |
| **Key Expressions** | User prompt text: `"Based on this YouTube title: {{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }} and music description: {{ $('Music Generation Form').item.json.description }}, create a visual prompt for a YouTube thumbnail."` |
| **Input Connections** | ← Generate Music - ElevenLabs (main input); ← OpenAI gpt-4o-mini Image (ai_languageModel) |
| **Output Connections** | → Generate Thumbnail - Atlas Cloud |
| **Output Structure** | `$json.output` — free-text visual prompt string |
| **Edge Cases / Failure Types** | Higher temperature (0.7) means more creative but potentially less relevant outputs; no structured output parser, so the output is raw text — downstream nodes must handle it as-is; if the YouTube title contains special characters, they may need escaping in the prompt |

**OpenAI gpt-4o-mini Image**

| Attribute | Value |
|-----------|-------|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) |
| **Technical Role** | Language model sub-node for the image prompt generation Agent |
| **Configuration** | Model: `gpt-4o-mini`; Temperature: 0.7 (higher creativity for visual prompts); No built-in tools |
| **Credentials** | OpenAI API credentials (same `OpenAi account` credential as the other OpenAI node, or a separate one) |
| **Input Connections** | None (sub-node) |
| **Output Connections** | → Generate Image Prompt (ai_languageModel channel) |
| **Edge Cases / Failure Types** | Same OpenAI credential and quota risks as Block 2; higher temperature increases risk of off-topic or excessively verbose prompts |

**Generate Thumbnail - Atlas Cloud**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.4) |
| **Technical Role** | Submits an asynchronous image generation request to Atlas Cloud using the `google/nano-banana-2/text-to-image` model |
| **Configuration** | URL: `https://api.atlascloud.ai/api/v1/model/generateImage`; Method: POST; Body parameters: `prompt` = `$('Generate Image Prompt').item.json.output`, `aspect_ratio` = `"16:9"`, `model` = `"google/nano-banana-2/text-to-image"`, `output_format` = `"png"`, `resolution` = `"1k"`; Header: `Authorization: Bearer YOUR_TOKEN_HERE` (must be replaced with actual Atlas Cloud token) |
| **Credentials** | None stored as n8n credential — the token is hardcoded in headers (placeholder `YOUR_TOKEN_HERE` must be replaced) |
| **Input Connections** | ← Generate Image Prompt |
| **Output Connections** | → Extract Prediction ID |
| **Edge Cases / Failure Types** | Placeholder token must be replaced before first run; Atlas Cloud API key may expire; async model means immediate response contains a prediction ID, not the image itself; model name may change or be deprecated; rate limits on Atlas Cloud free tier; response timeout unlikely since response is quick (just returns ID) |

**Extract Prediction ID**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Custom JavaScript to pull the prediction ID from the Atlas Cloud response for subsequent polling |
| **Configuration** | JS Code: reads `$input.first().json`, extracts `response.data.id` into `predictionId`, returns as new item |
| **Key Expressions / Variables** | Output: `$json.predictionId` |
| **Input Connections** | ← Generate Thumbnail - Atlas Cloud |
| **Output Connections** | → Wait |
| **Edge Cases / Failure Types** | If Atlas Cloud changes its response structure (e.g., `data.id` becomes `id`), this node will return `undefined` and downstream polling will fail with a 404; if the response contains an error instead of data, the extraction will produce an invalid ID |

**Wait**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.wait` (v1.1) |
| **Technical Role** | Pauses execution for 2 minutes to allow Atlas Cloud's asynchronous image generation to complete |
| **Configuration** | Wait unit: *minutes*; Amount: 2; Webhook-based resume (generated webhook ID: `acb76754-179a-4e26-8a1f-e4235d688a1c`) |
| **Input Connections** | ← Extract Prediction ID |
| **Output Connections** | → Check Image Status |
| **Edge Cases / Failure Types** | If image generation takes longer than 2 minutes, the status check may return "processing" and the workflow will proceed with incomplete data; n8n must be configured to support webhooks for wait nodes; if the n8n instance is restarted during the wait, the execution may be lost; the webhook ID is auto-generated and unique per workflow import |

**Check Image Status**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.4) |
| **Technical Role** | Polls the Atlas Cloud prediction status endpoint to retrieve the final image result |
| **Configuration** | URL: `https://api.atlascloud.ai/api/v1/model/prediction/{{ $('Extract Prediction ID').item.json.predictionId }}`; Method: GET (default); Header: `Authorization: Bearer YOUR_TOKEN_HERE` (same placeholder as above) |
| **Credentials** | None stored as n8n credential — token in header |
| **Input Connections** | ← Wait |
| **Output Connections** | → Code clear imageUrl |
| **Output Structure** | Response expected to contain `data.outputs[0]` — the final image URL |
| **Edge Cases / Failure Types** | If polling happens too early (image not ready), the `outputs` array may be empty; placeholder token must match the one used in the generation request; 404 if prediction ID is invalid; no retry logic — a single poll is performed |

**Code clear imageUrl**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Extracts the final image URL from the nested Atlas Cloud response structure |
| **Configuration** | JS Code: maps each item, extracting `item.json?.data?.outputs?.[0]` into `imageUrl`; returns null if not found |
| **Key Expressions / Variables** | Output: `$json.imageUrl` |
| **Input Connections** | ← Check Image Status |
| **Output Connections** | → Merge Audio & Image Paths (input 1) |
| **Edge Cases / Failure Types** | If Atlas Cloud response structure changes, `outputs[0]` may not exist; `imageUrl` will be `null` and downstream Shotstack render will fail with an invalid image source; no validation or retry logic |

---

#### Block 5 — Asset Merge & URL Preparation

**Overview:** The Cloudinary audio URL (from Block 3) and the Atlas Cloud image URL (from Block 4) arrive at the Merge node on separate inputs. The merged data is then normalized by a Code node that extracts clean `audioUrl`, `imageUrl`, and `duration` fields into a single, predictable payload for the Shotstack rendering request.

**Nodes Involved:**
- Merge Audio & Image Paths
- Code clean urls

**Node Details:**

**Merge Audio & Image Paths**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.merge` (v3.2) |
| **Technical Role** | Combines the two parallel branches (Cloudinary audio upload and Atlas Cloud image URL extraction) into a single item |
| **Configuration** | Default merge mode (combine/append based on v3.2 defaults — combines items by position) |
| **Input Connections** | Input 0 ← Upload an asset from file data; Input 1 ← Code clear imageUrl |
| **Output Connections** | → Code clean urls |
| **Edge Cases / Failure Types** | If one branch completes significantly faster, the merge will wait for the other; if one branch produces zero items, the merge may produce an empty result depending on mode; mismatched item counts can cause misalignment (audio paired with wrong image) |

**Code clean urls**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Custom JavaScript that scans all merged items, identifies the audio item (by `secure_url` or `url` fields) and the image item (by `imageUrl` field), and produces a single consolidated output |
| **Configuration** | JS Code: searches `items` for audio (has `secure_url` or `url`) and image (has `imageUrl`); returns `{ audioUrl, imageUrl, duration }` where `duration` defaults to 10 if not present in Cloudinary response |
| **Key Expressions / Variables** | Output: `$json.audioUrl`, `$json.imageUrl`, `$json.duration` |
| **Input Connections** | ← Merge Audio & Image Paths |
| **Output Connections** | → Shotstack Render |
| **Edge Cases / Failure Types** | If neither item has the expected fields, returns an empty array and the workflow stops; `duration` defaulting to 10 seconds is a fallback — if the actual music is 5 minutes, the video will only show 10 seconds of image unless Cloudinary properly returns duration; the code assumes a single audio item and a single image item — multiple items could cause incorrect pairing |

---

#### Block 6 — Video Rendering

**Overview:** The clean asset URLs are submitted to Shotstack's render API, which combines the static thumbnail image with the audio track into an MP4 video. After submitting the render job, the workflow waits 1 minute and then polls Shotstack for the final video URL.

**Nodes Involved:**
- Shotstack Render
- Wait2
- Shotstack Poll Result

**Node Details:**

**Shotstack Render**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.3) |
| **Technical Role** | POST request to Shotstack's render endpoint; defines a timeline with a soundtrack (audio) and a single image clip displayed for the audio's duration |
| **Configuration** | URL: `https://api.shotstack.io/edit/v1/render`; Method: POST; Body specification: *JSON*; JSON body includes: `timeline.soundtrack.src` = `{{$json.audioUrl}}`; `timeline.tracks[0].clips[0].asset.type` = `"image"`, `.src` = `{{$json.imageUrl}}`, `.start` = 0, `.length` = `{{$json.duration}}`; `output.format` = `"mp4"`, `output.resolution` = `"sd"`; Headers: `x-api-key: YOUR_API` (must be replaced), `Content-Type: application/json` |
| **Credentials** | None stored as n8n credential — API key in header (placeholder `YOUR_API` must be replaced with actual Shotstack API key) |
| **Input Connections** | ← Code clean urls |
| **Output Connections** | → Wait2 |
| **Output Structure** | `$json.response.id` — the Shotstack render ID |
| **Edge Cases / Failure Types** | Placeholder API key must be replaced; invalid audio URL (e.g., Cloudinary URL expired or requires auth) will cause render failure; very long audio may exceed Shotstack's timeline limits; `sd` resolution produces 360p output — may not meet YouTube quality expectations; Shotstack free tier has rendering limits; if `$json.duration` is 10 (the default fallback), long audio will be truncated to 10 seconds of video |

**Wait2**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.wait` (v1.1) |
| **Technical Role** | Pauses execution for 1 minute to allow Shotstack to render the video |
| **Configuration** | Wait unit: *minutes*; Amount: 1; Webhook-based resume (webhook ID: `67b0f6a7-a489-47c3-b6ac-ef4af886f22e`) |
| **Input Connections** | ← Shotstack Render |
| **Output Connections** | → Shotstack Poll Result |
| **Edge Cases / Failure Types** | 1 minute may be insufficient for longer videos; n8n instance must support webhook-based wait; execution may be lost on n8n restart; no retry if the wait webhook is missed |

**Shotstack Poll Result**

| Attribute | Value |
|-----------|-------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.3) |
| **Technical Role** | GET request to check the Shotstack render status and retrieve the final video URL |
| **Configuration** | URL: `https://api.shotstack.io/edit/v1/render/{{ $json.response.id }}`; Method: GET (default); Header: `x-api-key: YOUR_API` (same placeholder as Shotstack Render) |
| **Credentials** | None stored — header-based auth |
| **Input Connections** | ← Wait2 |
| **Output Connections** | → Create post Youtube |
| **Output Structure** | `$json.response.url` — the final rendered video URL (used by Blotato) |
| **Edge Cases / Failure Types** | If polling happens before render completes, `response.url` may be empty or `response.status` may be "rendering" — no status check logic exists, the workflow blindly passes the URL to YouTube; placeholder API key must be replaced; 404 if render ID is invalid; no retry on failed poll |

---

#### Block 7 — YouTube Publishing

**Overview:** The final rendered video URL is published to YouTube using the Blotato social media management integration. The SEO-optimized title generated by the AI Agent in Block 2 is used as both the video title and the post text.

**Nodes Involved:**
- Create post Youtube

**Node Details:**

**Create post Youtube**

| Attribute | Value |
|-----------|-------|
| **Type** | `@blotato/n8n-nodes-blotato.blotato` (v2) |
| **Technical Role** | Publishes a video to YouTube via the Blotato platform |
| **Configuration** | Platform: *youtube*; Account ID: `30379` (hardcoded, cached result: "Dr. Firas (PlayKids - English)"); Post content text: `$('Generate ElevenLabs Prompt').item.json.output.youtube_title`; Post content media URLs: `$json.response.url` (Shotstack render result); YouTube-specific option — Title: `$('Generate ElevenLabs Prompt').item.json.output.youtube_title` |
| **Credentials** | Blotato API credentials (referenced as `Blotato account`) |
| **Input Connections** | ← Shotstack Poll Result |
| **Output Connections** | None (terminal node) |
| **Edge Cases / Failure Types** | Blotato API key invalid or expired; YouTube account disconnected from Blotato; Blotato npm package must be installed (`npm install @blotato/n8n-nodes-blotato`); video URL from Shotstack may be a temporary URL that expires — if Blotato downloads it asynchronously, it may expire before download; the Account ID is hardcoded to a specific Blotato account — must be changed for other users; no description, tags, or thumbnail are set on the YouTube upload beyond the title |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|----------------|-----------------|-------------|
| Music Generation Form | n8n-nodes-base.formTrigger (v2.5) | Webhook entry point; collects user music preferences | None | Generate ElevenLabs Prompt | ## 🎵 Generate Content Inputs — Creates the base inputs for the workflow including prompts and music generation. |
| Generate ElevenLabs Prompt | @n8n/n8n-nodes-langchain.agent (v3.1) | AI Agent transforming user input into optimized ElevenLabs parameters and YouTube title | Music Generation Form; OpenAI gpt-4o-mini (ai_languageModel); JSON Output Parser (ai_outputParser) | Generate Music - ElevenLabs | ## 🎵 Generate Content Inputs — Creates the base inputs for the workflow including prompts and music generation. |
| OpenAI gpt-4o-mini | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.3) | Language model for music prompt Agent | None (sub-node) | Generate ElevenLabs Prompt (ai_languageModel) | ## 🎵 Generate Content Inputs — Creates the base inputs for the workflow including prompts and music generation. |
| JSON Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured (v1.3) | Enforces structured JSON output schema for music prompt Agent | None (sub-node) | Generate ElevenLabs Prompt (ai_outputParser) | ## 🎵 Generate Content Inputs — Creates the base inputs for the workflow including prompts and music generation. |
| Generate Music - ElevenLabs | n8n-nodes-base.httpRequest (v4.4) | Calls ElevenLabs Music API to generate audio file | Generate ElevenLabs Prompt | Generate Image Prompt; Upload an asset from file data | ## 🎵 Generate Content Inputs — Creates the base inputs for the workflow including prompts and music generation. |
| Upload an asset from file data | n8n-nodes-cloudinary.cloudinary (v1) | Uploads generated audio binary to Cloudinary for persistent URL | Generate Music - ElevenLabs | Merge Audio & Image Paths (input 0) | |
| Generate Image Prompt | @n8n/n8n-nodes-langchain.agent (v3.1) | AI Agent creating a visual prompt for YouTube thumbnail | Generate Music - ElevenLabs; OpenAI gpt-4o-mini Image (ai_languageModel) | Generate Thumbnail - Atlas Cloud | ## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets. |
| OpenAI gpt-4o-mini Image | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.3) | Language model for image prompt Agent | None (sub-node) | Generate Image Prompt (ai_languageModel) | ## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets. |
| Generate Thumbnail - Atlas Cloud | n8n-nodes-base.httpRequest (v4.4) | Submits async image generation request to Atlas Cloud | Generate Image Prompt | Extract Prediction ID | ## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets. |
| Extract Prediction ID | n8n-nodes-base.code (v2) | Extracts prediction ID from Atlas Cloud response for polling | Generate Thumbnail - Atlas Cloud | Wait | ## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets. |
| Wait | n8n-nodes-base.wait (v1.1) | Waits 2 minutes for Atlas Cloud image generation to complete | Extract Prediction ID | Check Image Status | ## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets. |
| Check Image Status | n8n-nodes-base.httpRequest (v4.4) | Polls Atlas Cloud for image generation result | Wait | Code clear imageUrl | ## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets. |
| Code clear imageUrl | n8n-nodes-base.code (v2) | Extracts final image URL from Atlas Cloud polling response | Check Image Status | Merge Audio & Image Paths (input 1) | ## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets. |
| Merge Audio & Image Paths | n8n-nodes-base.merge (v3.2) | Merges Cloudinary audio URL and Atlas Cloud image URL branches | Upload an asset from file data (input 0); Code clear imageUrl (input 1) | Code clean urls | |
| Code clean urls | n8n-nodes-base.code (v2) | Normalizes merged data into clean audioUrl, imageUrl, and duration fields | Merge Audio & Image Paths | Shotstack Render | ## 🎬 Video Rendering — Builds and renders the final video using Shotstack. |
| Shotstack Render | n8n-nodes-base.httpRequest (v4.3) | Submits video render request to Shotstack (image + audio → MP4) | Code clean urls | Wait2 | ## 🎬 Video Rendering — Builds and renders the final video using Shotstack. |
| Wait2 | n8n-nodes-base.wait (v1.1) | Waits 1 minute for Shotstack render to complete | Shotstack Render | Shotstack Poll Result | ## 🎬 Video Rendering — Builds and renders the final video using Shotstack. |
| Shotstack Poll Result | n8n-nodes-base.httpRequest (v4.3) | Polls Shotstack for final rendered video URL | Wait2 | Create post Youtube | ## 🎬 Video Rendering — Builds and renders the final video using Shotstack. |
| Create post Youtube | @blotato/n8n-nodes-blotato.blotato (v2) | Publishes rendered video to YouTube via Blotato | Shotstack Poll Result | None | ## 📺 YouTube Publishing — Publishes the generated video automatically to YouTube. |

---

### 4. Reproducing the Workflow from Scratch

**Prerequisites:**
- n8n instance (self-hosted or cloud) with webhook support enabled
- Active accounts and API keys for: OpenAI, ElevenLabs, Atlas Cloud, Cloudinary, Shotstack, Blotato
- Blotato community node installed: `npm install @blotato/n8n-nodes-blotato`
- FFmpeg is **not** required in this version (Shotstack replaces FFmpeg-based video creation)

---

**Step 1 — Create the Form Trigger**

1. Add a **Form Trigger** node (`n8n-nodes-base.formTrigger`).
2. Set **Form Title** to: `AI Music Generator for YouTube`.
3. Set **Form Description** to: `Generate custom AI music and publish it automatically to YouTube`.
4. Disable **Append Attribution** in options.
5. Add three form fields:
   - **Duration** — type: Dropdown; options: `30s`, `1min`, `2min`, `3min`, `5min`; field name: `duration`.
   - **Music Description** — type: Textarea; placeholder: `Describe the style, mood, and feel of the music you want...`; field name: `description`.
   - **Vocal Type** — type: Radio Buttons; options: `Instrumental`, `With Lyrics`; field name: `lyrics_type`.
6. Save and note the generated webhook URL.

---

**Step 2 — Add the JSON Output Parser**

1. Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`).
2. Set **Schema Type** to: *Manual*.
3. Paste the following JSON schema:

```json
{
  "type": "object",
  "properties": {
    "prompt": {
      "type": "string",
      "description": "Optimized music prompt"
    },
    "duration_ms": {
      "type": "number",
      "description": "Duration in milliseconds"
    },
    "force_instrumental": {
      "type": "boolean",
      "description": "True for instrumental, false for vocals"
    },
    "youtube_title": {
      "type": "string",
      "description": "SEO-optimized YouTube title"
    }
  }
}
```

---

**Step 3 — Add the OpenAI Chat Model (Music)**

1. Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`).
2. Set **Model** to: `gpt-4o-mini` (select from list).
3. Set **Temperature** to: `0.3`.
4. Configure **OpenAI API credentials** — enter your OpenAI API key.

---

**Step 4 — Add the Music Prompt AI Agent**

1. Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`).
2. Set **Prompt Type** to: *Define*.
3. Set the **User Prompt** expression to:

```
Durée : {{ $json.duration }}
Lyrics type : {{ $json.lyrics_type }}
Description :{{ $json.description }}
```

4. Set the **System Message** to:

> You are a music prompt optimizer. Transform user input into optimized ElevenLabs music generation parameters. Return a JSON object with: 1) prompt (string): an optimized musical description in English, 2) duration_ms (number): duration in milliseconds, 3) force_instrumental (boolean): true if instrumental, false if with lyrics, 4) youtube_title (string): an SEO-optimized YouTube title based on the music description.

5. Enable **Has Output Parser** (toggle on).
6. Connect the **Form Trigger** output → Agent main input.
7. Connect the **OpenAI Chat Model** → Agent `ai_languageModel` input.
8. Connect the **JSON Output Parser** → Agent `ai_outputParser` input.

---

**Step 5 — Add the ElevenLabs Music HTTP Request**

1. Add an **HTTP Request** node.
2. Set **URL** to: `https://api.elevenlabs.io/v1/music`.
3. Set **Method** to: `POST`.
4. Set **Authentication** to: *Generic Credential Type* → *Header Auth*.
5. Create a **Header Auth** credential with your ElevenLabs API key (header name is typically `xi-api-key` with the API key as value).
6. Set **Response Format** to: *File* (binary).
7. Enable **Send Body** and add three parameters:
   - `prompt` → expression: `{{ $('Generate ElevenLabs Prompt').item.json.output.prompt }}`
   - `duration_ms` → expression: `{{ $('Generate ElevenLabs Prompt').item.json.output.duration_ms }}`
   - `force_instrumental` → expression: `{{ $('Generate ElevenLabs Prompt').item.json.output.force_instrumental }}`
8. Connect the **AI Agent** output → this HTTP Request.

---

**Step 6 — Add the Cloudinary Upload Node**

1. Add a **Cloudinary** node (`n8n-nodes-cloudinary.cloudinary`).
2. Set **Operation** to: *Upload file*.
3. Set **Resource Type** to: *video*.
4. Configure **Cloudinary API credentials** (Cloud Name, API Key, API Secret).
5. Connect the **ElevenLabs HTTP Request** output → Cloudinary node.

---

**Step 7 — Add the OpenAI Chat Model (Image)**

1. Add a second **OpenAI Chat Model** node.
2. Set **Model** to: `gpt-4o-mini`.
3. Set **Temperature** to: `0.7`.
4. Use the same or a different **OpenAI API credential**.

---

**Step 8 — Add the Image Prompt AI Agent**

1. Add a second **AI Agent** node.
2. Set **Prompt Type** to: *Define*.
3. Set the **User Prompt** expression to:

```
Based on this YouTube title: {{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }} and music description: {{ $('Music Generation Form').item.json.description }}, create a visual prompt for a YouTube thumbnail.
```

4. Set the **System Message** to:

> You are a visual prompt expert for AI image generation. Create a detailed, vivid prompt in English for generating a YouTube thumbnail (16:9 aspect ratio, 1280x720). The image should be eye-catching, professional, and related to the music theme. Focus on colors, composition, and visual elements that work well for YouTube thumbnails.

5. Do **not** enable an output parser (free-text output).
6. Connect the **ElevenLabs HTTP Request** output → this Agent's main input.
7. Connect the **second OpenAI Chat Model** → this Agent's `ai_languageModel` input.

---

**Step 9 — Add the Atlas Cloud Image Generation HTTP Request**

1. Add an **HTTP Request** node.
2. Set **URL** to: `https://api.atlascloud.ai/api/v1/model/generateImage`.
3. Set **Method** to: `POST`.
4. Enable **Send Headers** and add: `Authorization` → `Bearer YOUR_ATLAS_CLOUD_TOKEN` (replace placeholder).
5. Enable **Send Body** and add five parameters:
   - `prompt` → expression: `{{ $('Generate Image Prompt').item.json.output }}`
   - `aspect_ratio` → `16:9`
   - `model` → `google/nano-banana-2/text-to-image`
   - `output_format` → `png`
   - `resolution` → `1k`
6. Connect the **Image Prompt AI Agent** output → this HTTP Request.

---

**Step 10 — Add the Extract Prediction ID Code Node**

1. Add a **Code** node.
2. Set the **JavaScript** to:

```javascript
const response = $input.first().json;

return [{
  json: {
    predictionId: response.data.id
  }
}];
```

3. Connect the **Atlas Cloud HTTP Request** output → this Code node.

---

**Step 11 — Add the First Wait Node**

1. Add a **Wait** node.
2. Set **Unit** to: *minutes*.
3. Set **Amount** to: `2`.
4. Connect the **Extract Prediction ID** output → this Wait node.

---

**Step 12 — Add the Check Image Status HTTP Request**

1. Add an **HTTP Request** node.
2. Set **URL** expression to: `https://api.atlascloud.ai/api/v1/model/prediction/{{ $('Extract Prediction ID').item.json.predictionId }}`.
3. Enable **Send Headers** and add: `Authorization` → `Bearer YOUR_ATLAS_CLOUD_TOKEN` (same token as Step 9).
4. Connect the **Wait** node output → this HTTP Request.

---

**Step 13 — Add the Code clear imageUrl Node**

1. Add a **Code** node.
2. Set the **JavaScript** to:

```javascript
return items.map(item => {
  const imageUrl = item.json?.data?.outputs?.[0] || null;
  return {
    json: {
      imageUrl
    }
  };
});
```

3. Connect the **Check Image Status** output → this Code node.

---

**Step 14 — Add the Merge Node**

1. Add a **Merge** node (`n8n-nodes-base.merge`, v3.2).
2. Leave default settings (combine mode).
3. Connect **Upload an asset from file data** (Cloudinary) → Merge **input 0**.
4. Connect **Code clear imageUrl** → Merge **input 1**.

---

**Step 15 — Add the Code clean urls Node**

1. Add a **Code** node.
2. Set the **JavaScript** to:

```javascript
const audioItem = items.find(item => item.json.secure_url || item.json.url);
const imageItem = items.find(item => item.json.imageUrl);

if (!audioItem || !imageItem) {
  return [];
}

return [
  {
    json: {
      audioUrl: audioItem.json.secure_url || audioItem.json.url,
      imageUrl: imageItem.json.imageUrl,
      duration: audioItem.json.duration || 10
    }
  }
];
```

3. Connect the **Merge** output → this Code node.

---

**Step 16 — Add the Shotstack Render HTTP Request**

1. Add an **HTTP Request** node (use v4.3 if available).
2. Set **URL** to: `https://api.shotstack.io/edit/v1/render`.
3. Set **Method** to: `POST`.
4. Set **Specify Body** to: *JSON*.
5. Enable **Send Headers** and add:
   - `x-api-key` → `YOUR_SHOTSTACK_API_KEY` (replace placeholder).
   - `Content-Type` → `application/json`.
6. Set the **JSON Body** expression to:

```json
{
  "timeline": {
    "soundtrack": {
      "src": "{{$json.audioUrl}}"
    },
    "tracks": [
      {
        "clips": [
          {
            "asset": {
              "type": "image",
              "src": "{{$json.imageUrl}}"
            },
            "start": 0,
            "length": {{$json.duration}}
          }
        ]
      }
    ]
  },
  "output": {
    "format": "mp4",
    "resolution": "sd"
  }
}
```

7. Connect the **Code clean urls** output → this HTTP Request.

---

**Step 17 — Add the Second Wait Node**

1. Add a **Wait** node.
2. Set **Unit** to: *minutes*.
3. Set **Amount** to: `1`.
4. Connect the **Shotstack Render** output → this Wait node.

---

**Step 18 — Add the Shotstack Poll Result HTTP Request**

1. Add an **HTTP Request** node.
2. Set **URL** expression to: `https://api.shotstack.io/edit/v1/render/{{ $json.response.id }}`.
3. Enable **Send Headers** and add: `x-api-key` → `YOUR_SHOTSTACK_API_KEY` (same as Step 16).
4. Connect the **second Wait** node output → this HTTP Request.

---

**Step 19 — Add the Blotato YouTube Publish Node**

1. Ensure the Blotato community node is installed: `npm install @blotato/n8n-nodes-blotato`.
2. Add a **Blotato** node (`@blotato/n8n-nodes-blotato.blotato`).
3. Set **Platform** to: *youtube*.
4. Select your **Account ID** from the list (or enter manually — the template uses `30379` for "Dr. Firas (PlayKids - English)").
5. Set **Post Content Text** expression to: `{{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }}`.
6. Set **Post Content Media URLs** expression to: `{{ $json.response.url }}`.
7. Set **YouTube Title Option** expression to: `{{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }}`.
8. Configure **Blotato API credentials** (your Blotato account API key).
9. Connect the **Shotstack Poll Result** output → this Blotato node.

---

**Step 20 — Add Sticky Notes (Optional)**

1. Add a **Sticky Note** covering the form and AI Agent area with content: `## 🎵 Generate Content Inputs — Creates the base inputs for the workflow including prompts and music generation.`
2. Add a **Sticky Note** covering the image generation area with content: `## 🖼️ AI Thumbnail Generation — Generates AI thumbnail using AtlasCloud and prepares image assets.`
3. Add a **Sticky Note** covering the Shotstack area with content: `## 🎬 Video Rendering — Builds and renders the final video using Shotstack.`
4. Add a **Sticky Note** covering the Blotato node with content: `## 📺 YouTube Publishing — Publishes the generated video automatically to YouTube.`
5. Optionally add the large overview sticky note with full workflow description, tutorial video link, credentials list, and setup steps (see Section 5 for the full content).

---

**Step 21 — Activate and Test**

1. Replace all placeholder API keys (`YOUR_TOKEN_HERE`, `YOUR_API`) with actual values in the Atlas Cloud and Shotstack HTTP Request headers.
2. Ensure all credentials (OpenAI, ElevenLabs Header Auth, Cloudinary, Blotato) are properly configured.
3. Activate the workflow.
4. Submit the form with test values (e.g., "1min", "relaxing lo-fi beats", "Instrumental").
5. Monitor execution — watch for errors at each API call stage.
6. Verify the YouTube video appears on the connected channel.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Full workflow overview: "Generate AI Music & Publish to YouTube Automatically with Blotato" — by Dr. Firas. Covers the complete pipeline from form input through AI music generation, thumbnail creation, video rendering, and YouTube publishing. | Main sticky note |
| Video tutorial for the workflow | [https://www.youtube.com/watch?v=yit-6fuKAVU](https://www.youtube.com/watch?v=yit-6fuKAVU) |
| OpenAI API keys | [https://platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| ElevenLabs API key (affiliate link) | [https://elevenlabs.io/?from=drfirass2382](https://elevenlabs.io/?from=drfirass2382) |
| Atlas Cloud API key (affiliate link) | [https://www.atlascloud.ai?ref=8QKPJE](https://www.atlascloud.ai?ref=8QKPJE) |
| Blotato platform (affiliate link) | [https://blotato.com/?ref=firas](https://blotato.com/?ref=firas) |
| Blotato n8n community node installation command | `npm install @blotato/n8n-nodes-blotato` |
| Shotstack API documentation | [https://shotstack.io/docs/](https://shotstack.io/docs/) |
| The workflow uses `gpt-4o-mini` at two different temperatures (0.3 for structured music parameters, 0.7 for creative image prompts). Using the same OpenAI credential for both is acceptable. | Architecture note |
| Atlas Cloud image generation is asynchronous — the workflow only polls **once** after a 2-minute wait. If the image takes longer, the workflow will proceed with a null or incomplete URL. Consider adding an IF/Loop pattern for robust polling. | Reliability note |
| Shotstack rendering is also asynchronous — the workflow polls **once** after 1 minute. Longer videos (3–5 min) may require more time. Consider increasing the wait or adding a polling loop. | Reliability note |
| The `Code clean urls` node defaults `duration` to 10 seconds if Cloudinary does not return duration metadata. This means for a 5-minute track, the Shotstack video will only render 10 seconds unless Cloudinary properly returns the audio duration field. Verify that Cloudinary's `duration` field is populated for your audio format, or hardcode the duration from the user's form selection. | Critical edge case |
| All API keys for Atlas Cloud and Shotstack are embedded directly as header parameters (not as n8n credentials). For better security and maintainability, migrate them to n8n credential objects (HTTP Header Auth or generic credential types). | Security best practice |
| The Blotato Account ID (`30379`) is hardcoded to the template author's YouTube channel. Users must change this to their own Blotato-connected YouTube account ID. | Configuration note |
| The workflow uses Shotstack at `sd` resolution (360p). For YouTube quality expectations, consider upgrading to `hd` (720p) or `1080` (1080p) in the Shotstack render JSON body. | Quality note |
| The workflow does not set a YouTube video description, tags, or custom thumbnail during upload. The Blotato node only sets the title and media URL. Additional metadata would require extending the Blotato node configuration or using the YouTube Data API directly. | Feature gap |
| FFmpeg is mentioned in the overview sticky note but is **not** used in this version of the workflow — Shotstack replaces the local FFmpeg video creation step. | Discrepancy note |
| The form trigger's webhook ID and the two Wait node webhook IDs are auto-generated on import. If you rebuild from scratch, n8n will generate new IDs automatically. | Technical note |