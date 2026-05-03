Generate AI music and publish to YouTube automatically with Blotato

https://n8nworkflows.xyz/workflows/generate-ai-music-and-publish-to-youtube-automatically-with-blotato-14538


# Generate AI music and publish to YouTube automatically with Blotato

### 1. Workflow Overview

This workflow automates the end-to-end process of generating AI music, producing a matching YouTube thumbnail, combining them into a video, and publishing the result to YouTube via the Blotato platform. It is triggered by a public form that collects musical preferences from a user, then orchestrates a multi-service pipeline across OpenAI (prompt optimization), ElevenLabs (music generation), Atlas Cloud (thumbnail image generation), Cloudinary (audio asset hosting), and Shotstack (video rendering) before publishing the final video on YouTube.

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1.1 | Input Reception | Collects user input (duration, description, vocal type) via a public web form |
| 1.2 | AI Prompt Optimization | Uses an AI Agent with GPT-4o-mini and a structured JSON output parser to transform raw user input into optimized ElevenLabs music generation parameters and an SEO-optimized YouTube title |
| 1.3 | Music Generation | Calls the ElevenLabs Music API with the optimized parameters to produce an audio file |
| 1.4 | Audio Asset Upload | Uploads the generated audio binary to Cloudinary for public URL hosting |
| 1.5 | Thumbnail Generation | Uses a second AI Agent to create a visual prompt, then calls Atlas Cloud's text-to-image API (with polling for async prediction), and extracts the final image URL |
| 1.6 | Asset Merge & URL Cleaning | Merges the audio Cloudinary URL and image URL, normalizes them, and prepares structured data for video rendering |
| 1.7 | Video Rendering | Sends the audio URL and image URL to Shotstack to render an MP4 video (static image + audio), then polls for render completion |
| 1.8 | YouTube Publishing | Uses the Blotato node to publish the rendered video to YouTube with the SEO-optimized title |

---

### 2. Block-by-Block Analysis

#### Block 1.1 — Input Reception

**Overview:** A public web form collects the user's desired music duration, a free-text description of the music style/mood, and whether the music should be instrumental or include lyrics. The form serves as the sole entry point for the workflow.

**Nodes Involved:**
- Music Generation Form

**Node Details:**

**Music Generation Form**
- **Type:** `n8n-nodes-base.formTrigger` (v2.5)
- **Technical Role:** Webhook-based form entry point. Produces a JSON payload from form submissions.
- **Configuration:**
  - Form Title: "AI Music Generator for YouTube"
  - Form Description: "Generate custom AI music and publish it automatically to YouTube"
  - Attribution appending: disabled
  - Fields:
    - `duration` (dropdown): options "30s", "1min", "2min", "3min", "5min"
    - `description` (textarea): placeholder "Describe the style, mood, and feel of the music you want..."
    - `lyrics_type` (radio): options "Instrumental" or "With Lyrics"
- **Key Expressions:** None — output is plain form field values.
- **Input:** None (trigger node).
- **Output:** Connects to **Generate ElevenLabs Prompt**.
- **Edge Cases:**
  - If the user submits without selecting a duration or lyrics_type, the form may pass `undefined` or empty strings downstream, causing the AI Agent to produce unexpected results.
  - Webhook URL must be accessible (public n8n instance or tunnel) for the form to work.

---

#### Block 1.2 — AI Prompt Optimization

**Overview:** An AI Agent powered by GPT-4o-mini transforms raw user form data into a structured JSON object containing an optimized ElevenLabs music prompt, duration in milliseconds, a boolean for instrumental mode, and an SEO-optimized YouTube title. A structured output parser guarantees the response conforms to the expected schema.

**Nodes Involved:**
- Generate ElevenLabs Prompt
- OpenAI gpt-4o-mini
- JSON Output Parser

**Node Details:**

**Generate ElevenLabs Prompt**
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v3.1)
- **Technical Role:** AI Agent node that combines a language model, a system message, and an output parser to produce structured JSON from free-form input.
- **Configuration:**
  - Prompt Type: "define" (static prompt defined in node)
  - Text expression:
    ```
    Durée : {{ $json.duration }}
    Lyrics type : {{ $json.lyrics_type }}
    Description : {{ $json.description }}
    ```
  - System Message: Instructs the agent to act as a "music prompt optimizer" and return a JSON object with keys: `prompt` (string), `duration_ms` (number), `force_instrumental` (boolean), `youtube_title` (string).
  - Output Parser: Enabled (`hasOutputParser: true`), connected to JSON Output Parser.
- **Key Expressions:** References `$json.duration`, `$json.lyrics_type`, `$json.description` from the form trigger output.
- **Input:** From **Music Generation Form**.
- **Output:** Connects to **Generate Music - ElevenLabs**. The output is accessed as `$('[NodeName]').item.json.output.<field>`.
- **Edge Cases:**
  - If the LLM fails to produce valid JSON or omits a required key, the structured output parser will throw a parsing error, halting execution.
  - If the duration text "30s" is not properly converted to milliseconds by the LLM, the downstream ElevenLabs call may fail or produce unexpected-length audio.

**OpenAI gpt-4o-mini**
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3)
- **Technical Role:** Language model provider sub-node for the AI Agent.
- **Configuration:**
  - Model: `gpt-4o-mini` (selected from list)
  - Temperature: 0.3 (low creativity for consistent structured output)
  - Built-in tools: none
- **Credentials:** OpenAI API credential (referenced as "OpenAi account").
- **Input/Output:** Connected as `ai_languageModel` to **Generate ElevenLabs Prompt** (not part of main flow).
- **Edge Cases:**
  - Invalid or expired API key causes 401 authentication errors.
  - Rate limits or quota exhaustion can cause 429 errors.

**JSON Output Parser**
- **Type:** `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3)
- **Technical Role:** Enforces structured output schema on the AI Agent response.
- **Configuration:**
  - Schema Type: "manual"
  - Input Schema (JSON Schema):
    ```json
    {
      "type": "object",
      "properties": {
        "prompt": { "type": "string", "description": "Optimized music prompt" },
        "duration_ms": { "type": "number", "description": "Duration in milliseconds" },
        "force_instrumental": { "type": "boolean", "description": "True for instrumental, false for vocals" },
        "youtube_title": { "type": "string", "description": "SEO-optimized YouTube title" }
      }
    }
    ```
- **Input/Output:** Connected as `ai_outputParser` to **Generate ElevenLabs Prompt** (not part of main flow).
- **Edge Cases:**
  - If the LLM returns extra keys or wrong types, the parser will attempt coercion or fail. Missing required keys cause a validation error.

---

#### Block 1.3 — Music Generation

**Overview:** Sends the optimized prompt, duration, and instrumental flag to the ElevenLabs Music API, which returns an audio file (binary). This is the core music creation step.

**Nodes Involved:**
- Generate Music - ElevenLabs

**Node Details:**

**Generate Music - ElevenLabs**
- **Type:** `n8n-nodes-base.httpRequest` (v4.4)
- **Technical Role:** Synchronous HTTP POST to ElevenLabs Music API; receives audio binary as response.
- **Configuration:**
  - URL: `https://api.elevenlabs.io/v1/music`
  - Method: POST
  - Authentication: Generic Credential Type → HTTP Header Auth (expects `xi-api-key` or similar header)
  - Body Parameters (3):
    - `prompt`: `={{ $('Generate ElevenLabs Prompt').item.json.output.prompt }}`
    - `duration_ms`: `={{ $('Generate ElevenLabs Prompt').item.json.output.duration_ms }}`
    - `force_instrumental`: `={{ $('Generate ElevenLabs Prompt').item.json.output.force_instrumental }}`
  - Response Format: `file` (binary audio data stored as workflow binary)
- **Credentials:** HTTP Header Auth credential (referenced as "Header Auth account").
- **Input:** From **Generate ElevenLabs Prompt**.
- **Output:** Connects to **Generate Image Prompt** (Branch B) and **Upload an asset from file data** (Branch A).
- **Edge Cases:**
  - Invalid API key → 401 error.
  - Exceeded ElevenLabs quota → 402/429 error.
  - If `duration_ms` is not a valid number or out of range (ElevenLabs supports specific ranges), the API may reject the request.
  - The response is binary; if the API returns an error JSON instead, the binary parsing may fail.

---

#### Block 1.4 — Audio Asset Upload

**Overview:** Uploads the generated audio binary file to Cloudinary as a video resource type to obtain a publicly accessible URL. This URL is needed by Shotstack to combine audio and image.

**Nodes Involved:**
- Upload an asset from file data

**Node Details:**

**Upload an asset from file data**
- **Type:** `n8n-nodes-cloudinary.cloudinary` (v1)
- **Technical Role:** Uploads binary data from the previous step to Cloudinary and returns metadata including `secure_url` and `duration`.
- **Configuration:**
  - Operation: `uploadFile`
  - Resource Type: `video` (Cloudinary treats audio as video resource type)
  - Additional Fields: none
- **Credentials:** Cloudinary API credential (referenced as "Cloudinary account 7").
- **Input:** From **Generate Music - ElevenLabs** (receives the binary audio file).
- **Output:** Connects to **Merge Audio & Image Paths** (input 0). Output includes `secure_url`, `url`, `duration`, and other Cloudinary metadata.
- **Edge Cases:**
  - Invalid Cloudinary credentials → 401.
  - Cloud storage quota exceeded → upload failure.
  - If no binary data is passed (e.g., ElevenLabs returned an error JSON body), the upload will fail with a missing-file error.

---

#### Block 1.5 — Thumbnail Generation

**Overview:** A second AI Agent creates a vivid visual prompt based on the YouTube title and music description. That prompt is sent to Atlas Cloud's text-to-image API (async prediction model). After submitting, the workflow extracts the prediction ID, waits 2 minutes, then polls for the result and extracts the final image URL.

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
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v3.1)
- **Technical Role:** AI Agent that generates a visual prompt suitable for AI image generation (YouTube thumbnail).
- **Configuration:**
  - Prompt Type: "define"
  - Text expression:
    ```
    Based on this YouTube title: {{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }} and music description: {{ $('Music Generation Form').item.json.description }}, create a visual prompt for a YouTube thumbnail.
    ```
  - System Message: Instructs the agent to act as a "visual prompt expert for AI image generation" and create a detailed prompt for a 16:9 (1280×720) YouTube thumbnail with focus on colors, composition, and visual elements.
  - Output Parser: Not connected (free-form text output).
- **Key Expressions:** References `youtube_title` from the first AI Agent and `description` from the form.
- **Input:** From **Generate Music - ElevenLabs**.
- **Output:** Connects to **Generate Thumbnail - Atlas Cloud**. Output accessed as `$('Generate Image Prompt').item.json.output`.
- **Edge Cases:**
  - If the LLM returns an empty or very short prompt, the generated image may be low quality.

**OpenAI gpt-4o-mini Image**
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3)
- **Technical Role:** Language model provider sub-node for the image prompt AI Agent.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Temperature: 0.7 (higher creativity for vivid visual prompts)
- **Credentials:** OpenAI API credential (same "OpenAi account").
- **Input/Output:** Connected as `ai_languageModel` to **Generate Image Prompt**.

**Generate Thumbnail - Atlas Cloud**
- **Type:** `n8n-nodes-base.httpRequest` (v4.4)
- **Technical Role:** Sends the visual prompt to Atlas Cloud's text-to-image API as an asynchronous prediction request.
- **Configuration:**
  - URL: `https://api.atlascloud.ai/api/v1/model/generateImage`
  - Method: POST
  - Body Parameters:
    - `prompt`: `={{ $('Generate Image Prompt').item.json.output }}`
    - `aspect_ratio`: `16:9`
    - `model`: `google/nano-banana-2/text-to-image`
    - `output_format`: `png`
    - `resolution`: `1k`
  - Header Parameters:
    - `Authorization`: `Bearer YOUR_TOKEN_HERE` (must be replaced with actual Atlas Cloud API token)
- **Credentials:** None configured at node level — auth is inline in headers (placeholder token).
- **Input:** From **Generate Image Prompt**.
- **Output:** Connects to **Extract Prediction ID**. Returns a JSON response containing `data.id` (prediction ID).
- **Edge Cases:**
  - Placeholder token `YOUR_TOKEN_HERE` will cause 401 — must be replaced.
  - If the model name changes or is deprecated, the API will return an error.
  - Async prediction: the image is not ready immediately; the response only contains a prediction ID.

**Extract Prediction ID**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Custom JavaScript code that extracts the prediction ID from the Atlas Cloud response.
- **Configuration:**
  - JS Code:
    ```javascript
    const response = $input.first().json;
    return [{
      json: {
        predictionId: response.data.id
      }
    }];
    ```
- **Input:** From **Generate Thumbnail - Atlas Cloud**.
- **Output:** Connects to **Wait**. Output is `{ predictionId: "<id>" }`.
- **Edge Cases:**
  - If `response.data.id` is undefined (e.g., API returned an error instead of a prediction), the output will have `predictionId: undefined`, and the subsequent polling request will fail.

**Wait**
- **Type:** `n8n-nodes-base.wait` (v1.1)
- **Technical Role:** Pauses execution for 2 minutes to allow the Atlas Cloud image generation to complete before polling.
- **Configuration:**
  - Wait Duration: 2 minutes
  - Webhook ID: `acb76754-179a-4e26-8a1f-e4235d688a1c` (auto-generated for resuming)
- **Input:** From **Extract Prediction ID**.
- **Output:** Connects to **Check Image Status**.
- **Edge Cases:**
  - If the n8n instance is restarted during the wait, the execution may not resume.
  - 2 minutes may be insufficient for complex images; the poll may return a "processing" status without a URL.

**Check Image Status**
- **Type:** `n8n-nodes-base.httpRequest` (v4.4)
- **Technical Role:** Polls the Atlas Cloud prediction endpoint to check if image generation is complete.
- **Configuration:**
  - URL: `https://api.atlascloud.ai/api/v1/model/prediction/{{ $('Extract Prediction ID').item.json.predictionId }}`
  - Method: GET (default)
  - Header Parameters:
    - `Authorization`: `Bearer YOUR_TOKEN_HERE` (must be replaced)
- **Input:** From **Wait**.
- **Output:** Connects to **Code clear imageUrl**. Returns full prediction status JSON.
- **Edge Cases:**
  - If the prediction is still processing, `data.outputs[0]` may be null or absent — the downstream code node will output `imageUrl: null`.
  - There is no retry loop — if the image isn't ready after 2 minutes, the workflow proceeds with a null URL and will fail at video rendering.

**Code clear imageUrl**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Extracts the final generated image URL from the polled Atlas Cloud response.
- **Configuration:**
  - JS Code:
    ```javascript
    return items.map(item => {
      const imageUrl = item.json?.data?.outputs?.[0] || null;
      return { json: { imageUrl } };
    });
    ```
- **Input:** From **Check Image Status**.
- **Output:** Connects to **Merge Audio & Image Paths** (input 1). Output is `{ imageUrl: "<url>" }` or `{ imageUrl: null }`.
- **Edge Cases:**
  - If `imageUrl` is null, downstream video rendering via Shotstack will fail because the image source will be invalid.

---

#### Block 1.6 — Asset Merge & URL Cleaning

**Overview:** Merges the Cloudinary audio upload result (input 0) and the Atlas Cloud image URL (input 1), then a code node extracts and normalizes the audio URL, image URL, and duration into a clean object for the video rendering step.

**Nodes Involved:**
- Merge Audio & Image Paths
- Code clean urls

**Node Details:**

**Merge Audio & Image Paths**
- **Type:** `n8n-nodes-base.merge` (v3.2)
- **Technical Role:** Combines two input streams (audio metadata from Cloudinary and image URL from Atlas Cloud) into a single item.
- **Configuration:** Default merge mode (Append/Combine — no special settings).
- **Input 0:** From **Upload an asset from file data** (Cloudinary — contains `secure_url`, `url`, `duration`, etc.)
- **Input 1:** From **Code clear imageUrl** (contains `imageUrl`)
- **Output:** Connects to **Code clean urls**. Merged item contains all keys from both inputs.
- **Edge Cases:**
  - If one branch completes significantly earlier than the other, the Merge node waits for both.
  - If one branch fails entirely, the Merge node may never receive both inputs, causing execution to stall or produce incomplete data.

**Code clean urls**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Extracts the audio URL, image URL, and duration from the merged data into a clean, minimal object.
- **Configuration:**
  - JS Code:
    ```javascript
    const audioItem = items.find(item => item.json.secure_url || item.json.url);
    const imageItem = items.find(item => item.json.imageUrl);
    if (!audioItem || !imageItem) { return []; }
    return [{
      json: {
        audioUrl: audioItem.json.secure_url || audioItem.json.url,
        imageUrl: imageItem.json.imageUrl,
        duration: audioItem.json.duration || 10
      }
    }];
    ```
- **Key Logic:** Falls back to `url` if `secure_url` is unavailable; defaults duration to 10 seconds if not present in Cloudinary metadata.
- **Input:** From **Merge Audio & Image Paths**.
- **Output:** Connects to **Shotstack Render**. Output is `{ audioUrl, imageUrl, duration }`.
- **Edge Cases:**
  - If neither `audioItem` nor `imageItem` is found, returns an empty array (`[]`), which halts the workflow with no items.
  - The fallback duration of 10 seconds may not match the actual music duration — this affects the Shotstack video length (image display duration).

---

#### Block 1.7 — Video Rendering

**Overview:** Sends the audio URL, image URL, and duration to Shotstack to render an MP4 video (static image + audio soundtrack). After submitting the render request, the workflow waits 1 minute, then polls Shotstack for the render result.

**Nodes Involved:**
- Shotstack Render
- Wait2
- Shotstack Poll Result

**Node Details:**

**Shotstack Render**
- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Submits a render request to the Shotstack editing API with a timeline containing a soundtrack (audio URL) and a single image clip.
- **Configuration:**
  - URL: `https://api.shotstack.io/edit/v1/render`
  - Method: POST
  - Body Specification: JSON (inline)
  - JSON Body expression:
    ```json
    {
      "timeline": {
        "soundtrack": { "src": "{{ $json.audioUrl }}" },
        "tracks": [{
          "clips": [{
            "asset": { "type": "image", "src": "{{ $json.imageUrl }}" },
            "start": 0,
            "length": {{ $json.duration }}
          }]
        }]
      },
      "output": { "format": "mp4", "resolution": "sd" }
    }
    ```
  - Header Parameters:
    - `x-api-key`: `YOUR_API` (must be replaced with actual Shotstack API key)
    - `Content-Type`: `application/json`
- **Credentials:** None configured at node level — API key is inline in headers (placeholder).
- **Input:** From **Code clean urls**.
- **Output:** Connects to **Wait2**. Returns JSON with `response.id` (render ID).
- **Edge Cases:**
  - Placeholder API key `YOUR_API` causes 403 Forbidden.
  - If `audioUrl` or `imageUrl` is null/invalid, Shotstack will return a render error.
  - `duration` must be a number — if it's a string, the JSON body may be malformed.

**Wait2**
- **Type:** `n8n-nodes-base.wait` (v1.1)
- **Technical Role:** Pauses execution for 1 minute to allow Shotstack to process the render.
- **Configuration:**
  - Wait Duration: 1 minute
  - Webhook ID: `67b0f6a7-a489-47c3-b6ac-ef4af886f22e`
- **Input:** From **Shotstack Render**.
- **Output:** Connects to **Shotstack Poll Result**.
- **Edge Cases:**
  - 1 minute may be insufficient for longer videos (e.g., 5-minute music); the poll may return a "rendering" status without a final URL.
  - Instance restart during wait may prevent resumption.

**Shotstack Poll Result**
- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Checks the Shotstack render status and retrieves the final video URL once rendering is complete.
- **Configuration:**
  - URL: `https://api.shotstack.io/edit/v1/render/{{ $json.response.id }}`
  - Method: GET (default)
  - Header Parameters:
    - `x-api-key`: `YOUR_API` (must be replaced)
- **Input:** From **Wait2**.
- **Output:** Connects to **Create post Youtube**. Returns JSON including `response.url` (the final MP4 video URL).
- **Edge Cases:**
  - If the render is still in progress, `response.url` may be null or absent.
  - No retry logic — a single poll is performed. If timing is off, the workflow proceeds with incomplete data.

---

#### Block 1.8 — YouTube Publishing

**Overview:** Uses the Blotato integration node to publish the rendered video to YouTube, setting the SEO-optimized title generated earlier by the AI Agent.

**Nodes Involved:**
- Create post Youtube

**Node Details:**

**Create post Youtube**
- **Type:** `@blotato/n8n-nodes-blotato.blotato` (v2)
- **Technical Role:** Publishes the rendered video to YouTube via the Blotato platform API.
- **Configuration:**
  - Platform: `youtube`
  - Account ID: `30379` (pre-selected, linked to "Dr. Firas (PlayKids - English)")
  - Post Content Text (YouTube description): `={{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }}`
  - Post Content Media URLs: `={{ $json.response.url }}` (Shotstack render result video URL)
  - YouTube Option Title: `={{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }}`
- **Credentials:** Blotato API credential (referenced as "Blotato account").
- **Input:** From **Shotstack Poll Result**.
- **Output:** None (terminal node).
- **Edge Cases:**
  - If `response.url` is null (render not complete), the YouTube upload will fail.
  - Blotato account must have YouTube connected and authorized.
  - The Blotato community node (`@blotato/n8n-nodes-blotato`) must be installed (`npm install @blotato/n8n-nodes-blotato`).
  - Both the post text and title use the same `youtube_title` — no separate description field is configured.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Music Generation Form | n8n-nodes-base.formTrigger | Entry point: collects user music preferences via web form | — | Generate ElevenLabs Prompt | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Generate ElevenLabs Prompt | @n8n/n8n-nodes-langchain.agent | AI Agent: transforms form input into structured ElevenLabs music params + YouTube title | Music Generation Form; OpenAI gpt-4o-mini (ai_languageModel); JSON Output Parser (ai_outputParser) | Generate Music - ElevenLabs | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| OpenAI gpt-4o-mini | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model for the ElevenLabs prompt AI Agent | — (sub-node) | Generate ElevenLabs Prompt (ai_languageModel) | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| JSON Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON schema on AI Agent output | — (sub-node) | Generate ElevenLabs Prompt (ai_outputParser) | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Generate Music - ElevenLabs | n8n-nodes-base.httpRequest | Calls ElevenLabs Music API to generate audio binary from optimized prompt | Generate ElevenLabs Prompt | Generate Image Prompt; Upload an asset from file data | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Upload an asset from file data | n8n-nodes-cloudinary.cloudinary | Uploads audio binary to Cloudinary for public URL hosting | Generate Music - ElevenLabs | Merge Audio & Image Paths (input 0) | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Generate Image Prompt | @n8n/n8n-nodes-langchain.agent | AI Agent: creates visual prompt for YouTube thumbnail generation | Generate Music - ElevenLabs; OpenAI gpt-4o-mini Image (ai_languageModel) | Generate Thumbnail - Atlas Cloud | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| OpenAI gpt-4o-mini Image | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model for the image prompt AI Agent | — (sub-node) | Generate Image Prompt (ai_languageModel) | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Generate Thumbnail - Atlas Cloud | n8n-nodes-base.httpRequest | Submits visual prompt to Atlas Cloud text-to-image API (async prediction) | Generate Image Prompt | Extract Prediction ID | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Extract Prediction ID | n8n-nodes-base.code | Extracts the prediction ID from Atlas Cloud's async response | Generate Thumbnail - Atlas Cloud | Wait | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Wait | n8n-nodes-base.wait | Pauses workflow for 2 minutes to allow Atlas Cloud image generation | Extract Prediction ID | Check Image Status | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Check Image Status | n8n-nodes-base.httpRequest | Polls Atlas Cloud prediction endpoint to check if image is ready | Wait | Code clear imageUrl | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Code clear imageUrl | n8n-nodes-base.code | Extracts the final image URL from the Atlas Cloud prediction response | Check Image Status | Merge Audio & Image Paths (input 1) | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Merge Audio & Image Paths | n8n-nodes-base.merge | Combines Cloudinary audio data and Atlas Cloud image URL into a single item | Upload an asset from file data (input 0); Code clear imageUrl (input 1) | Code clean urls | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Code clean urls | n8n-nodes-base.code | Extracts and normalizes audioUrl, imageUrl, and duration from merged data | Merge Audio & Image Paths | Shotstack Render | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Shotstack Render | n8n-nodes-base.httpRequest | Submits render request to Shotstack to combine audio + image into MP4 video | Code clean urls | Wait2 | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Wait2 | n8n-nodes-base.wait | Pauses workflow for 1 minute to allow Shotstack video rendering | Shotstack Render | Shotstack Poll Result | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Shotstack Poll Result | n8n-nodes-base.httpRequest | Polls Shotstack for render completion and retrieves final video URL | Wait2 | Create post Youtube | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Create post Youtube | @blotato/n8n-nodes-blotato.blotato | Publishes the rendered video to YouTube via Blotato with SEO title | Shotstack Poll Result | — (terminal) | # 🚀 Generate AI Music & Publish to YouTube Automatically with Blotato ## (By Dr. Firas) This workflow generates AI music and publishes it to YouTube automatically --- ## 🎥 Tutorial : @[youtube](yit-6fuKAVU) --- ## ⚙️ How It Works 1. Music Generation Form - Collects user input (duration, description, vocal type) 2. Generate ElevenLabs Prompt - AI Agent transforms input into optimized parameters 3. Generate Music - ElevenLabs - Calls ElevenLabs API to create the music audio 4. Generate Image Prompt - AI Agent creates a visual prompt for the thumbnail 5. Generate Thumbnail - Atlas Cloud - Generates a 16:9 YouTube thumbnail image 6. Save Audio File & Save Image File - Writes files to /tmp/ directory 7. FFmpeg - Create Video - Combines static image + audio into MP4 video 8. Read Final Video - Loads the final MP4 file 9. Publish to YouTube - Blotato - Uploads video to YouTube with title and description --- ## 🔐 Credentials Needed Required Credentials: 1. OpenAI API Key - Type: OpenAI API credentials - Get it at: https://platform.openai.com/api-keys - Used for: AI Agent nodes (prompt generation) 2. ElevenLabs API Key - Type: Header Auth (X-API-Key) - Get it at: **[elevenlabs](https://elevenlabs.io/?from=drfirass2382)** - Used for: Generate Music HTTP Request 3. Atlas Cloud API Key - Type: Header Auth (Authorization: Bearer) - Get it at: **[AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)** - Used for: Generate Thumbnail HTTP Request 4. Blotato API Key - Type: Blotato credentials - Get it at: **[Blotato](https://blotato.com/?ref=firas)** - Used for: YouTube video upload - Install: npm install @blotato/n8n-nodes-blotato 5. SSH Credentials (for FFmpeg) - Type: SSH credentials (localhost) - Required: FFmpeg installed on n8n server --- ## 🚀 Setup Steps 1- Add OpenAI credentials for prompt and content generation 2- Add ElevenLabs credentials for music generation 3- Add AtlasCloud credentials for image and thumbnail generation 4- Add Shotstack credentials for video rendering 5- Configure Upload & Merge nodes (audio + image paths) 6- Add YouTube credentials and configure post settings 7- Run a test and activate the workflow 🚀 |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation sticky note covering the entire workflow canvas | — | — | (content captured in all rows above) |

---

### 4. Reproducing the Workflow from Scratch

1. **Install required community nodes.** In your n8n instance, go to Settings → Community Nodes and install:
   - `n8n-nodes-cloudinary` (Cloudinary node)
   - `@blotato/n8n-nodes-blotato` (Blotato node)
   - The LangChain nodes (`@n8n/n8n-nodes-langchain`) are built into n8n v1.x+.

2. **Create the workflow** and name it "Generate AI Music & Publish to YouTube Automatically with Blotato".

3. **Add the Form Trigger node:**
   - Type: `Form Trigger`
   - Form Title: "AI Music Generator for YouTube"
   - Form Description: "Generate custom AI music and publish it automatically to YouTube"
   - Disable "Append Attribution"
   - Add three form fields:
     - Field 1: Name `duration`, Type `Dropdown`, Label "Music Duration", Options: "30s", "1min", "2min", "3min", "5min"
     - Field 2: Name `description`, Type `Textarea`, Label "Music Description", Placeholder: "Describe the style, mood, and feel of the music you want..."
     - Field 3: Name `lyrics_type`, Type `Radio`, Label "Vocal Type", Options: "Instrumental", "With Lyrics"

4. **Add the first AI Agent node (Generate ElevenLabs Prompt):**
   - Type: `AI Agent`
   - Prompt Type: "Define"
   - Text (expression): `Durée : {{ $json.duration }}\nLyrics type : {{ $json.lyrics_type }}\nDescription : {{ $json.description }}`
   - System Message: "You are a music prompt optimizer. Transform user input into optimized ElevenLabs music generation parameters. Return a JSON object with: 1) prompt (string): an optimized musical description in English, 2) duration_ms (number): duration in milliseconds, 3) force_instrumental (boolean): true if instrumental, false if with lyrics, 4) youtube_title (string): an SEO-optimized YouTube title based on the music description."
   - Enable Output Parser: Yes
   - Connect: From **Music Generation Form** → **Generate ElevenLabs Prompt**

5. **Add the OpenAI language model sub-node (OpenAI gpt-4o-mini):**
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o-mini`
   - Temperature: 0.3
   - Credentials: Create or select an OpenAI API credential
   - Connect: As `ai_languageModel` to **Generate ElevenLabs Prompt**

6. **Add the JSON Output Parser sub-node:**
   - Type: `Structured Output Parser`
   - Schema Type: "Manual"
   - Define the schema with four properties: `prompt` (string), `duration_ms` (number), `force_instrumental` (boolean), `youtube_title` (string)
   - Connect: As `ai_outputParser` to **Generate ElevenLabs Prompt**

7. **Add the ElevenLabs HTTP Request node (Generate Music - ElevenLabs):**
   - Type: `HTTP Request`
   - Method: POST
   - URL: `https://api.elevenlabs.io/v1/music`
   - Authentication: Generic Credential Type → HTTP Header Auth
   - Create an HTTP Header Auth credential with your ElevenLabs API key (header name: `xi-api-key`)
   - Body Parameters (Send Body: true):
     - `prompt`: `={{ $('Generate ElevenLabs Prompt').item.json.output.prompt }}`
     - `duration_ms`: `={{ $('Generate ElevenLabs Prompt').item.json.output.duration_ms }}`
     - `force_instrumental`: `={{ $('Generate ElevenLabs Prompt').item.json.output.force_instrumental }}`
   - Response Format: File (binary)
   - Connect: From **Generate ElevenLabs Prompt** → **Generate Music - ElevenLabs**

8. **Add the second AI Agent node (Generate Image Prompt):**
   - Type: `AI Agent`
   - Prompt Type: "Define"
   - Text (expression): `Based on this YouTube title: {{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }} and music description: {{ $('Music Generation Form').item.json.description }}, create a visual prompt for a YouTube thumbnail.`
   - System Message: "You are a visual prompt expert for AI image generation. Create a detailed, vivid prompt in English for generating a YouTube thumbnail (16:9 aspect ratio, 1280x720). The image should be eye-catching, professional, and related to the music theme. Focus on colors, composition, and visual elements that work well for YouTube thumbnails."
   - Output Parser: None
   - Connect: From **Generate Music - ElevenLabs** → **Generate Image Prompt**

9. **Add the second OpenAI language model sub-node (OpenAI gpt-4o-mini Image):**
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o-mini`
   - Temperature: 0.7
   - Credentials: Same OpenAI API credential
   - Connect: As `ai_languageModel` to **Generate Image Prompt**

10. **Add the Cloudinary Upload node (Upload an asset from file data):**
    - Type: `Cloudinary`
    - Operation: `Upload File`
    - Resource Type: `video`
    - Additional Fields: none
    - Credentials: Create or select a Cloudinary API credential (Cloud Name, API Key, API Secret)
    - Connect: From **Generate Music - ElevenLabs** → **Upload an asset from file data**

11. **Add the Atlas Cloud image generation HTTP Request node (Generate Thumbnail - Atlas Cloud):**
    - Type: `HTTP Request`
    - Method: POST
    - URL: `https://api.atlascloud.ai/api/v1/model/generateImage`
    - Send Body: true
    - Body Parameters:
      - `prompt`: `={{ $('Generate Image Prompt').item.json.output }}`
      - `aspect_ratio`: `16:9`
      - `model`: `google/nano-banana-2/text-to-image`
      - `output_format`: `png`
      - `resolution`: `1k`
    - Send Headers: true
    - Header Parameters:
      - `Authorization`: `Bearer <YOUR_ATLAS_CLOUD_API_TOKEN>` (replace with your actual token)
    - Connect: From **Generate Image Prompt** → **Generate Thumbnail - Atlas Cloud**

12. **Add the Code node (Extract Prediction ID):**
    - Type: `Code`
    - JS Code:
      ```javascript
      const response = $input.first().json;
      return [{
        json: {
          predictionId: response.data.id
        }
      }];
      ```
    - Connect: From **Generate Thumbnail - Atlas Cloud** → **Extract Prediction ID**

13. **Add the first Wait node (Wait):**
    - Type: `Wait`
    - Duration: 2 minutes
    - Connect: From **Extract Prediction ID** → **Wait**

14. **Add the Check Image Status HTTP Request node:**
    - Type: `HTTP Request`
    - Method: GET
    - URL (expression): `https://api.atlascloud.ai/api/v1/model/prediction/{{ $('Extract Prediction ID').item.json.predictionId }}`
    - Send Headers: true
    - Header Parameters:
      - `Authorization`: `Bearer <YOUR_ATLAS_CLOUD_API_TOKEN>` (same token as step 11)
    - Connect: From **Wait** → **Check Image Status**

15. **Add the Code node (Code clear imageUrl):**
    - Type: `Code`
    - JS Code:
      ```javascript
      return items.map(item => {
        const imageUrl = item.json?.data?.outputs?.[0] || null;
        return { json: { imageUrl } };
      });
      ```
    - Connect: From **Check Image Status** → **Code clear imageUrl**

16. **Add the Merge node (Merge Audio & Image Paths):**
    - Type: `Merge`
    - Mode: Default (Append/Combine)
    - Connect Input 0: From **Upload an asset from file data** → **Merge Audio & Image Paths** (input 0)
    - Connect Input 1: From **Code clear imageUrl** → **Merge Audio & Image Paths** (input 1)

17. **Add the Code node (Code clean urls):**
    - Type: `Code`
    - JS Code:
      ```javascript
      const audioItem = items.find(item => item.json.secure_url || item.json.url);
      const imageItem = items.find(item => item.json.imageUrl);
      if (!audioItem || !imageItem) { return []; }
      return [{
        json: {
          audioUrl: audioItem.json.secure_url || audioItem.json.url,
          imageUrl: imageItem.json.imageUrl,
          duration: audioItem.json.duration || 10
        }
      }];
      ```
    - Connect: From **Merge Audio & Image Paths** → **Code clean urls**

18. **Add the Shotstack Render HTTP Request node:**
    - Type: `HTTP Request`
    - Method: POST
    - URL: `https://api.shotstack.io/edit/v1/render`
    - Specify Body: JSON
    - JSON Body (expression):
      ```
      {
        "timeline": {
          "soundtrack": { "src": "{{$json.audioUrl}}" },
          "tracks": [{
            "clips": [{
              "asset": { "type": "image", "src": "{{$json.imageUrl}}" },
              "start": 0,
              "length": {{$json.duration}}
            }]
          }]
        },
        "output": { "format": "mp4", "resolution": "sd" }
      }
      ```
    - Send Headers: true
    - Header Parameters:
      - `x-api-key`: `<YOUR_SHOTSTACK_API_KEY>` (replace with your actual key)
      - `Content-Type`: `application/json`
    - Connect: From **Code clean urls** → **Shotstack Render**

19. **Add the second Wait node (Wait2):**
    - Type: `Wait`
    - Duration: 1 minute
    - Connect: From **Shotstack Render** → **Wait2**

20. **Add the Shotstack Poll Result HTTP Request node:**
    - Type: `HTTP Request`
    - Method: GET
    - URL (expression): `https://api.shotstack.io/edit/v1/render/{{ $json.response.id }}`
    - Send Headers: true
    - Header Parameters:
      - `x-api-key`: `<YOUR_SHOTSTACK_API_KEY>` (same key as step 18)
    - Connect: From **Wait2** → **Shotstack Poll Result**

21. **Add the Blotato YouTube publish node (Create post Youtube):**
    - Type: `Blotato`
    - Platform: `youtube`
    - Account: Select your connected YouTube account (or enter account ID)
    - Post Content Text: `={{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }}`
    - Post Content Media URLs: `={{ $json.response.url }}`
    - YouTube Title Option: `={{ $('Generate ElevenLabs Prompt').item.json.output.youtube_title }}`
    - Credentials: Create or select a Blotato API credential
    - Connect: From **Shotstack Poll Result** → **Create post Youtube**

22. **Add the Sticky Note** (optional, for documentation):
    - Place a large sticky note on the canvas with the content from the workflow's sticky note (setup steps, credentials needed, how it works).

23. **Verify all connections:**
    - Form → Generate ElevenLabs Prompt → Generate Music - ElevenLabs → (splits to) Generate Image Prompt AND Upload an asset from file data
    - Generate Image Prompt → Generate Thumbnail - Atlas Cloud → Extract Prediction ID → Wait → Check Image Status → Code clear imageUrl → Merge (input 1)
    - Upload an asset from file data → Merge (input 0)
    - Merge → Code clean urls → Shotstack Render → Wait2 → Shotstack Poll Result → Create post Youtube
    - Sub-nodes: OpenAI gpt-4o-mini → Generate ElevenLabs Prompt (ai_languageModel); JSON Output Parser → Generate ElevenLabs Prompt (ai_outputParser); OpenAI gpt-4o-mini Image → Generate Image Prompt (ai_languageModel)

24. **Configure all credentials:**
    - OpenAI API Key at https://platform.openai.com/api-keys
    - ElevenLabs API Key at https://elevenlabs.io/?from=drfirass2382
    - Atlas Cloud API Key at https://www.atlascloud.ai?ref=8QKPJE
    - Cloudinary API credentials (Cloud Name, API Key, API Secret)
    - Shotstack API Key (replace `YOUR_API` placeholders)
    - Blotato API Key at https://blotato.com/?ref=firas

25. **Test the workflow** by submitting the form with sample data, then activate it for production use.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video walkthrough of the workflow | https://www.youtube.com/watch?v=yit-6fuKAVU |
| Workflow authored by Dr. Firas | Credited in sticky note title |
| ElevenLabs referral link for API key | https://elevenlabs.io/?from=drfirass2382 |
| Atlas Cloud referral link for API key | https://www.atlascloud.ai?ref=8QKPJE |
| Blotato referral link for API key and node installation | https://blotato.com/?ref=firas |
| Blotato community node installation command | `npm install @blotato/n8n-nodes-blotato` |
| OpenAI API key management | https://platform.openai.com/api-keys |
| The Atlas Cloud and Shotstack API tokens are hardcoded as placeholders (`YOUR_TOKEN_HERE`, `YOUR_API`) in the HTTP Request headers — these must be manually replaced before the workflow can run. | Applies to nodes: Generate Thumbnail - Atlas Cloud, Check Image Status, Shotstack Render, Shotstack Poll Result |
| The image generation pipeline uses async polling with a fixed 2-minute wait. If Atlas Cloud takes longer, the poll may return a "still processing" status with no image URL, causing downstream failure. Consider adding a loop or increasing wait time for longer generations. | Applies to: Wait → Check Image Status → Code clear imageUrl |
| The video rendering pipeline uses a fixed 1-minute wait for Shotstack. Longer audio (e.g., 5 minutes of music) may require more rendering time. A single poll after 1 minute may return an incomplete render. | Applies to: Wait2 → Shotstack Poll Result |
| The Code clean urls node defaults `duration` to 10 seconds if Cloudinary does not return a duration value. This fallback may produce a video with an image displayed for only 10 seconds even if the audio is longer, resulting in a mismatch. | Applies to: Code clean urls |
| The YouTube post uses the same `youtube_title` for both the video title and the description text. No separate description field is configured. | Applies to: Create post Youtube |
| The workflow uses Cloudinary as an intermediary for audio hosting (uploading the ElevenLabs binary to get a public URL). This is required because Shotstack needs a publicly accessible audio URL. | Applies to: Upload an asset from file data |
| The n8n binary mode is set to "separate" in workflow settings, meaning binary data is kept separate from JSON data throughout the pipeline. | Workflow-level setting |
| The workflow is currently inactive (`active: false`) and must be manually activated after setup. | Workflow-level setting |