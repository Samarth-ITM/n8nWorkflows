Create children’s AI story videos from drawings and auto-publish to YouTube with Blotato

https://n8nworkflows.xyz/workflows/create-children-s-ai-story-videos-from-drawings-and-auto-publish-to-youtube-with-blotato-14171


# Create children’s AI story videos from drawings and auto-publish to YouTube with Blotato

# 1. Workflow Overview

This automation turns a child’s uploaded drawing into a complete AI-generated story video and publishes it to YouTube via Blotato. It is designed as a **3-part pipeline**, even though all nodes are present in the provided JSON. The workflow uses AI to detect characters in a drawing, generate character artwork, write a children’s story, render scene images, animate them into video clips, synthesize narration, assemble the final video, and publish it.

## 1.1 Overall Purpose

Target use cases:
- Turn children’s sketches into animated bedtime/story videos
- Generate consistent characters from a rough drawing
- Automatically build short-form or YouTube-friendly children’s content
- Chain multiple AI/image/video/audio services through n8n

## 1.2 Architectural Split

The workflow is explicitly intended to be split into **three separate n8n workflows**:

1. **Part 1 — From Drawing to Story: Characters, Images & Narrative**
   - Accepts a drawing from a form
   - Extracts characters from the image
   - Generates standalone character images
   - Writes a structured story in chunks
   - Sends the story + character assets to Part 2 by webhook

2. **Part 2 — From Characters to Scenes: Rendering the Visual Story**
   - Receives title, chunks, and character image URLs
   - Downloads character images and converts them to base64
   - Generates one illustrated scene per story chunk
   - Uploads scenes to Cloudinary
   - Sends the scene package to Part 3 by webhook

3. **Part 3 — From Scenes to Screen: Prompts, Audio & Final Assembly**
   - Receives scene image URLs and scripts
   - Generates animation prompts and short narration
   - Calls Atlas Cloud/Kling image-to-video
   - Polls until videos are ready
   - Synthesizes narration with ElevenLabs
   - Uploads audio to Cloudinary
   - Assembles final video with Shotstack
   - Generates YouTube metadata
   - Publishes to YouTube with Blotato

## 1.3 Logical Blocks

### Block A — Input Reception and Drawing Preparation
Handles form submission and converts the uploaded file into binary/base64-compatible input for image AI analysis.

### Block B — Character Extraction from Drawing
Uses Gemini Vision to interpret the uploaded drawing and return structured JSON describing all visible characters, the setting, and the mood.

### Block C — Character Prompt Generation and Character Image Rendering
Uses Claude to generate one image prompt per character, then uses Gemini image generation to create isolated character illustrations and stores them in Cloudinary.

### Block D — Story Generation
Builds structured story context from the generated characters and uses Claude to produce a short children’s story split into timed chunks.

### Block E — Scene Workflow Handoff
Packages story chunks and character image references, then posts them to the second workflow.

### Block F — Scene Rendering Workflow
Receives story data, converts character images into inline image references, generates one scene per chunk, uploads scenes to Cloudinary, and sends the result onward.

### Block G — Video Prompting and Video Clip Generation
Receives scene image URLs, writes animation prompts and shortened narration, then submits image-to-video jobs to Atlas Cloud.

### Block H — Video Polling and Narration Generation
Polls Atlas Cloud until videos complete, creates ElevenLabs narration for each scene, uploads audio files, and maps output URLs.

### Block I — Final Assembly and Publishing
Builds a Shotstack timeline from generated videos and narration, renders the final MP4, creates title/description text, and publishes to YouTube via Blotato. TikTok is configured but disabled.

---

# 2. Block-by-Block Analysis

## Block A — Input Reception and Drawing Preparation

### Overview
This block captures the child’s drawing through an n8n form and converts the uploaded file into a format suitable for downstream AI services. It prepares both binary image data and base64 text representations used later.

### Nodes Involved
- On form submission
- Extract from File
- Convert to File

### Node Details

#### 1. On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — workflow entry point for user submission
- **Configuration choices:**
  - Form title: **SketchTales**
  - One required file field named **Image**
  - Label shown to user: **Upload your child's drawing**
  - Single file only
- **Key variables / expressions:** none
- **Input connections:** none; entry trigger
- **Output connections:** to **Extract from File**
- **Version-specific notes:** typeVersion `2.4`
- **Potential failures / edge cases:**
  - No file uploaded
  - Unsupported file format
  - Trigger unavailable if workflow inactive
- **Sub-workflow reference:** Part 1 entry point

#### 2. Extract from File
- **Type / role:** `n8n-nodes-base.extractFromFile` — converts binary file input into a property
- **Configuration choices:**
  - Operation: `binaryToPropery`
  - Binary property name: `Image`
- **Key variables / expressions:** extracts uploaded image into JSON field data
- **Input connections:** from **On form submission**
- **Output connections:** to **Convert to File**
- **Version-specific notes:** typeVersion `1.1`
- **Potential failures / edge cases:**
  - Missing binary property `Image`
  - File decoding failure
  - Large file memory usage
- **Sub-workflow reference:** none

#### 3. Convert to File
- **Type / role:** `n8n-nodes-base.convertToFile` — converts JSON/base64-like content back into binary for image-model input
- **Configuration choices:**
  - Operation: `toBinary`
  - Source property: `data`
- **Key variables / expressions:** source property `data`
- **Input connections:** from **Extract from File**
- **Output connections:** to **Analyze an image**
- **Version-specific notes:** typeVersion `1.1`
- **Potential failures / edge cases:**
  - `data` property missing
  - Invalid base64 or incompatible file content
- **Sub-workflow reference:** none

---

## Block B — Character Extraction from Drawing

### Overview
This block analyzes the uploaded drawing with Gemini image understanding, normalizes the output field, and parses the returned JSON into one item per character. It transforms freeform vision output into structured story-building inputs.

### Nodes Involved
- Analyze an image
- Edit Fields
- Parse Characters

### Node Details

#### 4. Analyze an image
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — image analysis with Gemini
- **Configuration choices:**
  - Resource: `image`
  - Operation: `analyze`
  - Input type: `binary`
  - Model: `models/gemini-3-pro-preview`
  - Prompt instructs model to identify:
    - all characters/creatures/people
    - simple child-friendly names
    - visual descriptions
    - setting
    - mood
  - Requires exact JSON-only response
- **Key variables / expressions:** prompt output expected in `content.parts[0].text`
- **Input connections:** from **Convert to File**
- **Output connections:** to **Edit Fields**
- **Version-specific notes:** typeVersion `1.1`
- **Potential failures / edge cases:**
  - Google Gemini credential/auth failure
  - Model availability mismatch
  - Model may return markdown or malformed JSON despite instruction
  - Very abstract or low-quality drawings may produce weak extraction
- **Sub-workflow reference:** none

#### 5. Edit Fields
- **Type / role:** `n8n-nodes-base.set` — narrows the payload to the relevant text field
- **Configuration choices:**
  - Keeps/assigns `content.parts[0].text = {{$json.content.parts[0].text}}`
- **Key variables / expressions:** `{{ $json.content.parts[0].text }}`
- **Input connections:** from **Analyze an image**
- **Output connections:** to **Parse Characters**
- **Version-specific notes:** typeVersion `3.4`
- **Potential failures / edge cases:**
  - Missing `content.parts[0].text`
  - Response structure changes from Gemini node version/model
- **Sub-workflow reference:** none

#### 6. Parse Characters
- **Type / role:** `n8n-nodes-base.code` — parses Gemini JSON text and emits one item per character
- **Configuration choices:**
  - Removes possible ```json fences
  - Parses JSON
  - Reads `data.characters`
  - Returns one item per character with:
    - `character_index`
    - `character_name`
    - `character_desc`
    - `character_count`
- **Key variables / expressions:**
  - `$input.first().json`
  - `response.content.parts[0].text`
  - `JSON.parse(text)`
- **Input connections:** from **Edit Fields**
- **Output connections:** to **Build Character Prompt**
- **Version-specific notes:** typeVersion `2`
- **Potential failures / edge cases:**
  - Invalid JSON returned by Gemini
  - Missing `characters` array
  - Empty character list
  - Unused variables `charList` and `jsonFormat` are built but not returned; harmless but misleading
- **Sub-workflow reference:** none

---

## Block C — Character Prompt Generation and Character Image Rendering

### Overview
This block groups parsed character descriptions, asks Claude to create one image-generation prompt per character, generates reference character artwork using Gemini image generation, and uploads the resulting files to Cloudinary.

### Nodes Involved
- Build Character Prompt
- Generate Character Prompts
- Structured Output Parser2
- Anthropic Chat Model1
- Split Character Prompts
- Nano Banana Payload
- Nano Banana Generate
- Extract Character Image
- Character Image to File
- Upload an asset from file data1
- Map Character URLs

### Node Details

#### 7. Build Character Prompt
- **Type / role:** `n8n-nodes-base.code` — aggregates character items into a single prompt context
- **Configuration choices:**
  - Builds:
    - `character_count`
    - `character_list`
    - `json_format`
  - `json_format` dynamically matches number of characters
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **Parse Characters**
- **Output connections:** to **Generate Character Prompts**
- **Version-specific notes:** typeVersion `2`
- **Potential failures / edge cases:**
  - Empty item list
  - Very large character descriptions may reduce LLM reliability
- **Sub-workflow reference:** none

#### 8. Generate Character Prompts
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — structured LLM chain for prompt generation
- **Configuration choices:**
  - Uses dynamic text:
    - count of characters
    - character descriptions
    - exact JSON schema stub
  - System message enforces:
    - image-to-image prompt style
    - front-facing, full body, white background
    - consistent visual style
    - one character per prompt
    - JSON-only output
  - `hasOutputParser: true`
- **Key variables / expressions:**
  - `{{ $json.character_count }}`
  - `{{ $json.character_list }}`
  - `{{ $json.json_format }}`
- **Input connections:** from **Build Character Prompt**
- **Output connections:** to **Split Character Prompts**
- **AI connections:** from **Anthropic Chat Model1** and **Structured Output Parser2**
- **Version-specific notes:** typeVersion `1.9`
- **Potential failures / edge cases:**
  - Anthropic auth/model availability issues
  - Parser mismatch if response deviates from schema
  - Character count mismatch in returned list
- **Sub-workflow reference:** none

#### 9. Structured Output Parser2
- **Type / role:** `outputParserStructured` — validates character prompt JSON structure
- **Configuration choices:**
  - Expected schema:
    - `prompts[]`
    - each item has `character_index`, `character_name`, `prompt`
- **Key variables / expressions:** schema example only
- **Input connections:** AI parser input to **Generate Character Prompts**
- **Output connections:** none on main path
- **Version-specific notes:** typeVersion `1.3`
- **Potential failures / edge cases:**
  - Returned JSON does not conform
- **Sub-workflow reference:** none

#### 10. Anthropic Chat Model1
- **Type / role:** Anthropic chat model provider for character prompt generation
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Key variables / expressions:** none
- **Input connections:** AI language-model connection to **Generate Character Prompts**
- **Output connections:** none on main path
- **Version-specific notes:** typeVersion `1.3`
- **Potential failures / edge cases:**
  - API key issues
  - model deprecation/region restrictions
- **Sub-workflow reference:** none

#### 11. Split Character Prompts
- **Type / role:** `n8n-nodes-base.code` — expands structured LLM result into one item per character prompt
- **Configuration choices:**
  - Reads `output.prompts`
  - Emits:
    - `character_index`
    - `character_name`
    - `prompt`
- **Key variables / expressions:** `$input.first().json.output.prompts`
- **Input connections:** from **Generate Character Prompts**
- **Output connections:** to **Nano Banana Payload**
- **Version-specific notes:** typeVersion `2`
- **Potential failures / edge cases:**
  - Missing `output.prompts`
  - Parser/model inconsistency
- **Sub-workflow reference:** none

#### 12. Nano Banana Payload
- **Type / role:** `n8n-nodes-base.code` — builds Gemini image generation API payload for each character
- **Configuration choices:**
  - Pulls original drawing base64 from **Extract from File**
  - Creates multimodal payload with:
    - prompt text
    - inline original image reference
    - image generation config 16:9 and 1K
- **Key variables / expressions:**
  - `$('Extract from File').first().json.data`
  - `item.json.prompt`
- **Input connections:** from **Split Character Prompts**
- **Output connections:** to **Nano Banana Generate**
- **Version-specific notes:** typeVersion `2`
- **Potential failures / edge cases:**
  - Assumes uploaded file is PNG via hardcoded `image/png`
  - If source drawing is JPEG/webp, MIME mismatch may affect API behavior
  - Missing base64 `data`
- **Sub-workflow reference:** none

#### 13. Nano Banana Generate
- **Type / role:** `n8n-nodes-base.httpRequest` — direct Gemini image-generation API call
- **Configuration choices:**
  - POST to `.../models/gemini-3-pro-image-preview:generateContent`
  - JSON body from `{{$json.payload}}`
  - Uses predefined Google Gemini credential
  - Adds `Content-Type: application/json`
  - Retries on fail with 5-second wait
- **Key variables / expressions:** `={{ $json.payload }}`
- **Input connections:** from **Nano Banana Payload**
- **Output connections:** to **Extract Character Image**
- **Version-specific notes:** typeVersion `4.3`
- **Potential failures / edge cases:**
  - API auth/quota errors
  - Model output shape changes
  - No image returned in `candidates[0].content.parts[0].inlineData.data`
  - Long generation latency
- **Sub-workflow reference:** none

#### 14. Extract Character Image
- **Type / role:** `n8n-nodes-base.set` — extracts generated image base64 from API response
- **Configuration choices:**
  - Sets `data = {{$json.candidates[0].content.parts[0].inlineData.data}}`
- **Key variables / expressions:** response path above
- **Input connections:** from **Nano Banana Generate**
- **Output connections:** to **Character Image to File**
- **Version-specific notes:** typeVersion `3.4`
- **Potential failures / edge cases:**
  - Missing `inlineData`
  - Candidate order may vary
- **Sub-workflow reference:** none

#### 15. Character Image to File
- **Type / role:** `n8n-nodes-base.convertToFile` — converts base64 image data to binary file
- **Configuration choices:**
  - Operation: `toBinary`
  - Source property: `data`
- **Key variables / expressions:** source property `data`
- **Input connections:** from **Extract Character Image**
- **Output connections:** to **Upload an asset from file data1**
- **Version-specific notes:** typeVersion `1.1`
- **Potential failures / edge cases:**
  - Invalid base64
- **Sub-workflow reference:** none

#### 16. Upload an asset from file data1
- **Type / role:** `n8n-nodes-cloudinary.cloudinary` — uploads generated character files to Cloudinary
- **Configuration choices:**
  - Operation: `uploadFile`
- **Key variables / expressions:** uses incoming binary file
- **Input connections:** from **Character Image to File**
- **Output connections:** to **Map Character URLs**
- **Version-specific notes:** Cloudinary community node, typeVersion `1`
- **Potential failures / edge cases:**
  - Cloudinary auth/limits
  - file too large
  - unsupported file metadata
- **Sub-workflow reference:** none

#### 17. Map Character URLs
- **Type / role:** `n8n-nodes-base.code` — maps uploaded Cloudinary URLs back to the original character names
- **Configuration choices:**
  - Reads all Cloudinary outputs
  - Reads all parsed characters from **Parse Characters**
  - Emits:
    - `character_index`
    - `character_name`
    - `character_image_url`
- **Key variables / expressions:**
  - `$input.all()`
  - `$('Parse Characters').all()`
- **Input connections:** from **Upload an asset from file data1**
- **Output connections:** to **Build Story Context**
- **Version-specific notes:** typeVersion `2`
- **Potential failures / edge cases:**
  - Assumes Cloudinary output order matches character order
  - If one upload fails or order changes, names and URLs may be mismatched
- **Sub-workflow reference:** none

---

## Block D — Story Generation

### Overview
This block compiles the generated character assets into a story context and asks Claude to write a short children’s story in timed chunks. The result includes narration, scene prompts, and character references for each chunk.

### Nodes Involved
- Build Story Context
- Generate Story
- Structured Output Parser
- Anthropic Chat Model
- Split Story Chunks
- Extract Unique Character URLs
- Build Scene Payload

### Node Details

#### 18. Build Story Context
- **Type / role:** `n8n-nodes-base.code` — aggregates character names and image URLs
- **Configuration choices:**
  - Builds:
    - `character_count`
    - `character_list`
    - `character_map`
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **Map Character URLs**
- **Output connections:** to **Generate Story**
- **Version-specific notes:** typeVersion `2`
- **Potential failures / edge cases:**
  - Empty character list
- **Sub-workflow reference:** none

#### 19. Generate Story
- **Type / role:** `chainLlm` — creates structured children’s story
- **Configuration choices:**
  - Prompt asks for a children’s story using supplied characters
  - System message enforces:
    - 4–6 chunks
    - warm magical language for ages 3–7
    - chunk-level `script`, `image_prompt`, `characters_used`, `duration_seconds`
    - image prompt style consistency
    - JSON-only output
  - Has structured output parser
- **Key variables / expressions:**
  - `{{ $json.character_count }}`
  - `{{ $json.character_list }}`
- **Input connections:** from **Build Story Context**
- **Output connections:** to **Split Story Chunks**
- **AI connections:** from **Anthropic Chat Model** and **Structured Output Parser**
- **Version-specific notes:** typeVersion `1.9`
- **Potential failures / edge cases:**
  - Non-conforming JSON
  - `characters_used` indexes may reference invalid characters
  - duration may not match actual read time
- **Sub-workflow reference:** none

#### 20. Structured Output Parser
- **Type / role:** output schema validator for story JSON
- **Configuration choices:**
  - Schema with `title` and `chunks[]`
- **Input connections:** AI parser input to **Generate Story**
- **Output connections:** none on main path
- **Version-specific notes:** `1.3`
- **Potential failures / edge cases:**
  - malformed response
- **Sub-workflow reference:** none

#### 21. Anthropic Chat Model
- **Type / role:** Anthropic model provider for story generation
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Input connections:** AI language-model input to **Generate Story**
- **Output connections:** none on main path
- **Version-specific notes:** `1.3`
- **Potential failures / edge cases:** auth, quota, model changes
- **Sub-workflow reference:** none

#### 22. Split Story Chunks
- **Type / role:** `code` — expands story chunks into one item per scene
- **Configuration choices:**
  - Reads `output`
  - Maps:
    - `title`
    - `chunk_number`
    - `script`
    - `image_prompt`
    - `duration_seconds`
    - `characters_used`
    - `character_images`
  - Pulls image URLs from **Map Character URLs**
- **Key variables / expressions:**
  - `$input.first().json.output`
  - `$('Map Character URLs').all()`
- **Input connections:** from **Generate Story**
- **Output connections:** to **Extract Unique Character URLs**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Invalid character indices in `characters_used`
  - Indexing assumes 1-based array order
- **Sub-workflow reference:** none

#### 23. Extract Unique Character URLs
- **Type / role:** `code` — deduplicates character URLs used across all chunks
- **Configuration choices:**
  - Uses `Set` to deduplicate
  - Returns list as `character_index`, `image_url`
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **Split Story Chunks**
- **Output connections:** to **Build Scene Payload**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Re-indexes deduplicated URLs, which can break original character numbering
- **Sub-workflow reference:** none

#### 24. Build Scene Payload
- **Type / role:** `code` — prepares payload sent to Part 2 scene-rendering workflow
- **Configuration choices:**
  - Reads story from **Generate Story**
  - Reads character list from **Extract Unique Character URLs**
  - Builds:
    - `title`
    - `characters`
    - `chunks`
  - Each chunk includes `character_images`
- **Key variables / expressions:**
  - `$('Generate Story').first().json.output`
  - `$('Extract Unique Character URLs').all()`
- **Input connections:** from **Extract Unique Character URLs**
- **Output connections:** to **Trigger Scene Workflow**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Important logic bug risk:
    - it assumes `character_index` in deduplicated list still matches `characters_used`
    - `find(c => c.character_index === i)` may fail or map to wrong image if indexes shifted
- **Sub-workflow reference:** sends data to Part 2

---

## Block E — Scene Workflow Handoff

### Overview
This is the bridge from Part 1 to Part 2. It sends the prepared story/chunk/character payload to a separate workflow via webhook.

### Nodes Involved
- Trigger Scene Workflow

### Node Details

#### 25. Trigger Scene Workflow
- **Type / role:** `httpRequest` — invokes the scene-rendering workflow
- **Configuration choices:**
  - Method: `POST`
  - URL: test webhook on n8n Cloud
  - Sends whole JSON body
- **Key variables / expressions:** `={{ $json }}`
- **Input connections:** from **Build Scene Payload**
- **Output connections:** none
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - Uses `/webhook-test/` URL, not production `/webhook/`
  - Receiving workflow must be active or in test mode
  - Network and cross-workflow schema mismatches
- **Sub-workflow reference:** invokes Part 2 webhook `9a2bdfcb-57ae-4a8f-a3dc-78382b9334a0`

---

## Block F — Scene Rendering Workflow

### Overview
This workflow receives the story package, downloads character images, converts them to base64, injects them into image generation requests, creates one scene image per story chunk, uploads them to Cloudinary, and forwards them to the video workflow.

### Nodes Involved
- Webhook
- Split Scene Chunks
- Extract Character URLs
- Download Character Images
- Character Images to Base64
- Build Base64 Map
- Build Scene Image Payloads
- Nano Banana Scene Generate
- Extract Scene Image
- Scene Image to File
- Upload an asset from file data
- Map Scene URLs
- Trigger Video Workflow

### Node Details

#### 26. Webhook
- **Type / role:** `webhook` — Part 2 entry point
- **Configuration choices:**
  - POST webhook
  - Path matches UUID-like identifier
- **Input connections:** none; entry trigger
- **Output connections:** to **Split Scene Chunks**
- **Version-specific notes:** `2.1`
- **Potential failures / edge cases:**
  - Trigger path mismatch
  - inactive workflow
- **Sub-workflow reference:** Part 2 entry point

#### 27. Split Scene Chunks
- **Type / role:** `code` — expands incoming `body.chunks`
- **Configuration choices:**
  - Returns one item per chunk
  - Preserves title, prompt, duration, character image list
- **Key variables / expressions:** `$input.first().json.body`
- **Input connections:** from **Webhook**
- **Output connections:** to **Extract Character URLs**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:** missing `body.chunks`
- **Sub-workflow reference:** none

#### 28. Extract Character URLs
- **Type / role:** `code` — expands `body.characters`
- **Configuration choices:**
  - Emits `character_index` and `image_url`
- **Key variables / expressions:** `$('Webhook').first().json.body`
- **Input connections:** from **Split Scene Chunks**
- **Output connections:** to **Download Character Images**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Depends on webhook body still being accessible by expression
  - Missing `characters`
- **Sub-workflow reference:** none

#### 29. Download Character Images
- **Type / role:** `httpRequest` — downloads character images from remote URLs
- **Configuration choices:**
  - URL from `{{$json.image_url}}`
- **Key variables / expressions:** `={{ $json.image_url }}`
- **Input connections:** from **Extract Character URLs**
- **Output connections:** to **Character Images to Base64**
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - 404/403 from Cloudinary or external URL
  - rate limiting
- **Sub-workflow reference:** none

#### 30. Character Images to Base64
- **Type / role:** `extractFromFile` — converts downloaded binary images into JSON/base64
- **Configuration choices:**
  - Operation: `binaryToPropery`
- **Input connections:** from **Download Character Images**
- **Output connections:** to **Build Base64 Map**
- **Version-specific notes:** `1.1`
- **Potential failures / edge cases:**
  - binary property naming assumptions
- **Sub-workflow reference:** none

#### 31. Build Base64 Map
- **Type / role:** `code` — aggregates all character images into a single base64 map
- **Configuration choices:**
  - Returns `character_base64[]` with:
    - `character_index`
    - `base64`
    - `mimetype`
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **Character Images to Base64**
- **Output connections:** to **Build Scene Image Payloads**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Again assumes item order equals character order
- **Sub-workflow reference:** none

#### 32. Build Scene Image Payloads
- **Type / role:** `code` — creates multimodal Gemini image-generation request for each scene
- **Configuration choices:**
  - Reads chunk list from webhook body
  - Reads base64 character references from **Build Base64 Map**
  - For each scene:
    - starts with image prompt text
    - appends inline images for all characters used
    - sets image generation config to 16:9, 1K
- **Key variables / expressions:**
  - `$('Build Base64 Map').first().json.character_base64`
  - `$('Webhook').first().json.body`
- **Input connections:** from **Build Base64 Map**
- **Output connections:** to **Nano Banana Scene Generate**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - `characters_used` may not match available character map
  - missing base64 for one character silently skips that character
- **Sub-workflow reference:** none

#### 33. Nano Banana Scene Generate
- **Type / role:** `httpRequest` — Gemini image generation for story scenes
- **Configuration choices:**
  - Same endpoint as character generation
  - Google credential auth
  - retries enabled
- **Key variables / expressions:** `={{ $json.payload }}`
- **Input connections:** from **Build Scene Image Payloads**
- **Output connections:** to **Extract Scene Image**
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:** same as character generation
- **Sub-workflow reference:** none

#### 34. Extract Scene Image
- **Type / role:** `set` — extracts generated scene image base64
- **Configuration choices:**
  - `data = {{$json.candidates[0].content.parts[0].inlineData.data}}`
- **Input connections:** from **Nano Banana Scene Generate**
- **Output connections:** to **Scene Image to File**
- **Version-specific notes:** `3.4`
- **Potential failures / edge cases:** missing inline image field
- **Sub-workflow reference:** none

#### 35. Scene Image to File
- **Type / role:** `convertToFile` — converts generated scene image to binary file
- **Configuration choices:** source property `data`
- **Input connections:** from **Extract Scene Image**
- **Output connections:** to **Upload an asset from file data**
- **Version-specific notes:** `1.1`
- **Potential failures / edge cases:** invalid base64
- **Sub-workflow reference:** none

#### 36. Upload an asset from file data
- **Type / role:** Cloudinary upload for generated scene images
- **Configuration choices:** `uploadFile`
- **Input connections:** from **Scene Image to File**
- **Output connections:** to **Map Scene URLs**
- **Version-specific notes:** `1`
- **Potential failures / edge cases:** Cloudinary limits/auth
- **Sub-workflow reference:** none

#### 37. Map Scene URLs
- **Type / role:** `code` — pairs generated scene image URLs with original chunks
- **Configuration choices:**
  - Reads webhook body chunks
  - Assumes Cloudinary outputs align by index
  - Builds `title` and `chunks[]` with `scene_image_url`
- **Key variables / expressions:**
  - `$input.all()`
  - `$('Webhook').first().json.body`
- **Input connections:** from **Upload an asset from file data**
- **Output connections:** to **Trigger Video Workflow**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - index order mismatch
- **Sub-workflow reference:** none

#### 38. Trigger Video Workflow
- **Type / role:** `httpRequest` — invokes Part 3
- **Configuration choices:**
  - POST JSON to test webhook URL
- **Key variables / expressions:** `={{ $json }}`
- **Input connections:** from **Map Scene URLs**
- **Output connections:** none
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - uses `/webhook-test/`
  - receiving workflow activation mismatch
- **Sub-workflow reference:** invokes Part 3 webhook `ec089d71-c7c3-482a-b11d-f7c24ec16fbc`

---

## Block G — Video Prompting and Video Clip Generation

### Overview
This block receives scene images, generates motion prompts plus shorter narration, merges those prompts back onto the original scene data, and submits image-to-video generation requests to Atlas Cloud.

### Nodes Involved
- Webhook1
- Split Video Chunks
- Build Narration Context
- Generate Video Prompts
- Structured Output Parser1
- Anthropic Chat Model2
- Merge Video Prompts
- Build Kling Payloads
- Atlas Cloud Generate Video
- Wait
- Atlas Cloud Poll Result
- If

### Node Details

#### 39. Webhook1
- **Type / role:** `webhook` — Part 3 entry point
- **Configuration choices:** POST webhook
- **Input connections:** none
- **Output connections:** to **Split Video Chunks**
- **Version-specific notes:** `2.1`
- **Potential failures / edge cases:** inactive path / wrong endpoint
- **Sub-workflow reference:** Part 3 entry point

#### 40. Split Video Chunks
- **Type / role:** `code` — expands incoming scene package and stores title/chunks in workflow static data
- **Configuration choices:**
  - Stores:
    - `staticData.title`
    - `staticData.chunks`
  - Emits one item per chunk
- **Key variables / expressions:**
  - `$input.first().json.body`
  - `$getWorkflowStaticData('global')`
- **Input connections:** from **Webhook1**
- **Output connections:** to **Build Narration Context**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - static data persistence issues in parallel/test runs
  - missing body structure
- **Sub-workflow reference:** none

#### 41. Build Narration Context
- **Type / role:** `code` — composes all scenes into one textual prompt context
- **Configuration choices:**
  - Builds `chunk_count` and `chunk_list`
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **Split Video Chunks**
- **Output connections:** to **Generate Video Prompts**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:** empty input
- **Sub-workflow reference:** none

#### 42. Generate Video Prompts
- **Type / role:** `chainLlm` — creates image-to-video prompts and short narration
- **Configuration choices:**
  - System instructions enforce:
    - motion/camera descriptions
    - prompt begins with “Use the provided image as the first frame.”
    - narration max 18 words
    - output as structured JSON
- **Key variables / expressions:**
  - `{{ $json.chunk_count }}`
  - `{{ $json.chunk_list }}`
- **Input connections:** from **Build Narration Context**
- **Output connections:** to **Merge Video Prompts**
- **AI connections:** from **Anthropic Chat Model2** and **Structured Output Parser1**
- **Version-specific notes:** `1.9`
- **Potential failures / edge cases:**
  - parser mismatch
  - generated narration too long despite instruction
- **Sub-workflow reference:** none

#### 43. Structured Output Parser1
- **Type / role:** output parser for video prompts
- **Configuration choices:** expects `scenes[]` with `chunk_number`, `video_prompt`, `narration`
- **Input connections:** AI parser to **Generate Video Prompts**
- **Output connections:** none on main path
- **Version-specific notes:** `1.3`
- **Potential failures / edge cases:** malformed output
- **Sub-workflow reference:** none

#### 44. Anthropic Chat Model2
- **Type / role:** model provider for video prompt generation
- **Configuration choices:** `claude-sonnet-4-5-20250929`
- **Input connections:** AI LM to **Generate Video Prompts**
- **Output connections:** none
- **Version-specific notes:** `1.3`
- **Potential failures / edge cases:** auth/model issues
- **Sub-workflow reference:** none

#### 45. Merge Video Prompts
- **Type / role:** `code` — merges generated motion prompts back with scene image URLs and timings
- **Configuration choices:**
  - Reads LLM output scenes
  - Reads original webhook body
  - Emits:
    - `chunk_number`
    - `script`
    - `duration_seconds`
    - `scene_image_url`
    - `video_prompt`
- **Key variables / expressions:**
  - `$input.first().json.output.scenes`
  - `$('Webhook1').first().json.body`
- **Input connections:** from **Generate Video Prompts**
- **Output connections:** to **Build Kling Payloads**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - scene ordering mismatch
- **Sub-workflow reference:** none

#### 46. Build Kling Payloads
- **Type / role:** `code` — builds Atlas Cloud/Kling request per scene
- **Configuration choices:**
  - Model: `kwaivgi/kling-v3.0-pro/image-to-video`
  - Includes:
    - `cfg_scale: 0.5`
    - `duration`
    - `image`
    - optional `end_image` from next scene
    - `prompt`
    - negative prompt blocking lip-sync/talking
    - `sound: true`
  - Stores body as stringified JSON
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **Merge Video Prompts**
- **Output connections:** to **Atlas Cloud Generate Video**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - final scene has no `end_image` by design
  - API may expect object rather than JSON string depending on configuration
- **Sub-workflow reference:** none

#### 47. Atlas Cloud Generate Video
- **Type / role:** `httpRequest` — submits video generation job
- **Configuration choices:**
  - POST to Atlas Cloud generate endpoint
  - Headers include bearer token placeholder `YOUR_TOKEN_HERE`
  - Body from `{{$json.body}}`
- **Key variables / expressions:** `={{ $json.body }}`
- **Input connections:** from **Build Kling Payloads**
- **Output connections:** to **Wait**
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - token placeholder must be replaced
  - body may be double-encoded JSON string
  - rate limiting / queue delays
- **Sub-workflow reference:** none

#### 48. Wait
- **Type / role:** `wait` — pauses before polling generation result
- **Configuration choices:**
  - Wait 11 minutes
- **Input connections:** from **Atlas Cloud Generate Video** and false branch from **If**
- **Output connections:** to **Atlas Cloud Poll Result**
- **Version-specific notes:** `1.1`
- **Potential failures / edge cases:**
  - fixed wait may be too short or wastefully long
- **Sub-workflow reference:** none

#### 49. Atlas Cloud Poll Result
- **Type / role:** `httpRequest` — polls generation status
- **Configuration choices:**
  - GET `prediction/{{ $json.data.id }}`
  - Bearer token header placeholder
- **Key variables / expressions:** `=https://api.atlascloud.ai/api/v1/model/prediction/{{ $json.data.id }}`
- **Input connections:** from **Wait**
- **Output connections:** to **If**
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - missing `data.id`
  - pending/failed status handling incomplete
- **Sub-workflow reference:** none

#### 50. If
- **Type / role:** conditional status check
- **Configuration choices:**
  - Condition: `{{$json.data.status}} == "completed"`
  - True branch -> **Extract Video URLs**
  - False branch -> **Wait**
- **Key variables / expressions:** `={{ $json.data.status }}`
- **Input connections:** from **Atlas Cloud Poll Result**
- **Output connections:** true to **Extract Video URLs**, false back to **Wait**
- **Version-specific notes:** `2.3`
- **Potential failures / edge cases:**
  - no explicit handling for `failed`, `canceled`, or missing status, causing endless loop
- **Sub-workflow reference:** none

---

## Block H — Video Polling and Narration Generation

### Overview
Once video clips are complete, this block extracts their URLs, generates one short narrated audio clip per scene with ElevenLabs, uploads those audio files, and prepares them for final assembly.

### Nodes Involved
- Extract Video URLs
- Prepare Narration Loop
- Loop Over Items
- ElevenLabs Narrate
- Upload an asset from file data2
- Wait1
- Extract Audio URLs

### Node Details

#### 51. Extract Video URLs
- **Type / role:** `code` — maps Atlas outputs into scene video URLs
- **Configuration choices:**
  - Emits `chunk_number` and `video_url`
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **If**
- **Output connections:** to **Prepare Narration Loop**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Assumes `item.json.data.outputs[0]` exists
  - Ignores original chunk numbers and reindexes sequentially
- **Sub-workflow reference:** none

#### 52. Prepare Narration Loop
- **Type / role:** `code` — prepares ElevenLabs request payloads from generated narration text
- **Configuration choices:**
  - Reads `Generate Video Prompts` output
  - For each scene builds JSON body:
    - `text`
    - `model_id: eleven_multilingual_v2`
    - voice settings
- **Key variables / expressions:** `$('Generate Video Prompts').first().json.output.scenes`
- **Input connections:** from **Extract Video URLs**
- **Output connections:** to **Loop Over Items**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Disconnect between video completion order and narration order
- **Sub-workflow reference:** none

#### 53. Loop Over Items
- **Type / role:** `splitInBatches` — iterative processing of narration/audio upload
- **Configuration choices:** default batch loop behavior
- **Input connections:** from **Prepare Narration Loop** and from **Wait1**
- **Output connections:**
  - main output 0 -> **Extract Audio URLs**
  - main output 0 -> **ElevenLabs Narrate**
- **Version-specific notes:** `3`
- **Potential failures / edge cases:**
  - Unusual wiring: same output feeds both downstream extraction and narration request path
  - Can be confusing to maintain
- **Sub-workflow reference:** none

#### 54. ElevenLabs Narrate
- **Type / role:** `httpRequest` — text-to-speech generation
- **Configuration choices:**
  - POST to a fixed voice endpoint
  - Response format: file
  - Headers:
    - `xi-api-key: YOUR_API_HERE`
    - `Content-Type: application/json`
  - Body from `{{$json.body}}`
- **Key variables / expressions:** `={{ $json.body }}`
- **Input connections:** from **Loop Over Items**
- **Output connections:** to **Upload an asset from file data2**
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - API key placeholder must be replaced
  - fixed voice ID may not exist for account
  - body may be JSON string rather than object
- **Sub-workflow reference:** none

#### 55. Upload an asset from file data2
- **Type / role:** Cloudinary upload for narration audio
- **Configuration choices:**
  - `uploadFile`
  - `resource_type_file: raw`
- **Input connections:** from **ElevenLabs Narrate**
- **Output connections:** to **Wait1**
- **Version-specific notes:** `1`
- **Potential failures / edge cases:** raw file upload auth/limits
- **Sub-workflow reference:** none

#### 56. Wait1
- **Type / role:** `wait` — resumes loop iteration
- **Configuration choices:** no explicit time set; acts as resumable wait node
- **Input connections:** from **Upload an asset from file data2**
- **Output connections:** to **Loop Over Items**
- **Version-specific notes:** `1.1`
- **Potential failures / edge cases:** depends on how n8n resumes split-in-batches loops
- **Sub-workflow reference:** none

#### 57. Extract Audio URLs
- **Type / role:** `code` — maps uploaded audio file URLs
- **Configuration choices:**
  - Emits `chunk_number` and `audio_url`
- **Key variables / expressions:** `$input.all()`
- **Input connections:** from **Loop Over Items**
- **Output connections:** to **Build Shotstack Payload**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Because it receives loop items, may not only receive completed uploads depending on execution semantics
  - Reindexes by array order, not original chunk number
- **Sub-workflow reference:** none

---

## Block I — Final Assembly and Publishing

### Overview
This block combines rendered video clips and narration audio into a single Shotstack timeline, renders the final MP4, generates title/description metadata, and publishes to YouTube. A TikTok node is present but disabled.

### Nodes Involved
- Build Shotstack Payload
- Build Shotstack Timeline
- Shotstack Render
- Wait2
- Shotstack Poll Result
- Generate Video Prompts1
- Structured Output Parser3
- Anthropic Chat Model3
- Youtube
- Tiktok

### Node Details

#### 58. Build Shotstack Payload
- **Type / role:** `code` — combines chunk metadata, video URLs, and audio URLs
- **Configuration choices:**
  - Reads:
    - audio items from input
    - video items from **Extract Video URLs**
    - chunks from **Split Video Chunks**
    - title from workflow static data
  - Throws explicit errors if any input group is empty
- **Key variables / expressions:**
  - `$('Extract Video URLs').all()`
  - `$('Split Video Chunks').all()`
  - `$getWorkflowStaticData('global')`
- **Input connections:** from **Extract Audio URLs**
- **Output connections:** to **Build Shotstack Timeline**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - order mismatch between chunks/videos/audio
  - static data collisions between concurrent runs
- **Sub-workflow reference:** none

#### 59. Build Shotstack Timeline
- **Type / role:** `code` — builds Shotstack render payload
- **Configuration choices:**
  - Creates two tracks:
    - video clips
    - audio clips
  - Uses `start` offsets accumulated from `duration_seconds`
  - Output config:
    - format `mp4`
    - resolution `hd`
- **Key variables / expressions:** `$input.first().json`
- **Input connections:** from **Build Shotstack Payload**
- **Output connections:** to **Shotstack Render**
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - video/audio real durations may differ from declared chunk durations
  - no title card or transitions
- **Sub-workflow reference:** none

#### 60. Shotstack Render
- **Type / role:** `httpRequest` — submits final render job
- **Configuration choices:**
  - POST render endpoint
  - `x-api-key: YOUR_API_HERE`
  - body from `{{$json.body}}`
- **Key variables / expressions:** `={{ $json.body }}`
- **Input connections:** from **Build Shotstack Timeline**
- **Output connections:** to **Wait2**
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - placeholder key must be replaced
  - body may be JSON string
- **Sub-workflow reference:** none

#### 61. Wait2
- **Type / role:** `wait` — pauses before polling Shotstack render
- **Configuration choices:** 1 minute
- **Input connections:** from **Shotstack Render**
- **Output connections:** to **Shotstack Poll Result**
- **Version-specific notes:** `1.1`
- **Potential failures / edge cases:**
  - no retry loop if render still queued/in-progress after 1 minute
- **Sub-workflow reference:** none

#### 62. Shotstack Poll Result
- **Type / role:** `httpRequest` — fetches render result
- **Configuration choices:**
  - GET `render/{{ $json.response.id }}`
  - `x-api-key` header
- **Key variables / expressions:** `=https://api.shotstack.io/edit/v1/render/{{ $json.response.id }}`
- **Input connections:** from **Wait2**
- **Output connections:** to **Generate Video Prompts1**
- **Version-specific notes:** `4.3`
- **Potential failures / edge cases:**
  - no conditional loop if render not completed
- **Sub-workflow reference:** none

#### 63. Generate Video Prompts1
- **Type / role:** `chainLlm` — generates publishing title and description
- **Configuration choices:**
  - Uses story title and all scene scripts
  - Requests JSON with `video_title` and `video_description`
  - Has structured output parser
- **Key variables / expressions:**
  - `{{ $('Webhook1').item.json.body.title }}`
  - `{{ $('Split Video Chunks').all().map(...) }}`
- **Input connections:** from **Shotstack Poll Result**
- **Output connections:** to **Youtube** and **Tiktok**
- **AI connections:** from **Anthropic Chat Model3** and **Structured Output Parser3**
- **Version-specific notes:** `1.9`
- **Potential failures / edge cases:**
  - parser schema is oddly named `scenes[]` though content is metadata; see below
- **Sub-workflow reference:** none

#### 64. Structured Output Parser3
- **Type / role:** output parser for publishing metadata
- **Configuration choices:**
  - Expects:
    - `scenes[]`
      - `video_title`
      - `description`
- **Input connections:** AI parser input to **Generate Video Prompts1**
- **Output connections:** none
- **Version-specific notes:** `1.3`
- **Potential failures / edge cases:**
  - Schema does not match the prompt text, which asks for:
    - `{"video_title":"...","video_description":"..."}`
  - This is a likely bug and may break metadata generation
- **Sub-workflow reference:** none

#### 65. Anthropic Chat Model3
- **Type / role:** model provider for publish metadata
- **Configuration choices:** `claude-sonnet-4-5-20250929`
- **Input connections:** AI LM input to **Generate Video Prompts1**
- **Output connections:** none
- **Version-specific notes:** `1.3`
- **Potential failures / edge cases:** auth/model issues
- **Sub-workflow reference:** none

#### 66. Youtube
- **Type / role:** `@blotato/n8n-nodes-blotato.blotato` — publishes final video to YouTube through Blotato
- **Configuration choices:**
  - Platform: `youtube`
  - Account: `Dr. Firas (PlayKids - English)`
  - Text/description from `{{$json.output.scenes[0].description}}`
  - Media URL from `$('Shotstack Poll Result').item.json.response.url`
  - Title from `{{$json.output.scenes[0].video_title}}`
  - Made for kids = true
  - Notify subscribers = false
- **Key variables / expressions:**
  - `={{ $json.output.scenes[0].description }}`
  - `={{ $('Shotstack Poll Result').item.json.response.url }}`
  - `={{ $json.output.scenes[0].video_title }}`
- **Input connections:** from **Generate Video Prompts1**
- **Output connections:** none
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Depends on parser bug above; data path likely invalid if metadata schema differs
  - Blotato account auth/publishing permissions required
- **Sub-workflow reference:** none

#### 67. Tiktok
- **Type / role:** Blotato TikTok publisher
- **Configuration choices:**
  - Disabled
  - Platform: `tiktok`
  - Media URL from Shotstack
  - Privacy: `SELF_ONLY`
- **Key variables / expressions:** title path same metadata output structure
- **Input connections:** from **Generate Video Prompts1**
- **Output connections:** none
- **Version-specific notes:** `2`
- **Potential failures / edge cases:**
  - Disabled node
  - same metadata schema mismatch risk
- **Sub-workflow reference:** none

---

## Additional Non-Execution / Annotation Nodes

### Sticky Note2
Contains global project context:
- workflow split into 3 separate workflows
- tutorial link: `@[youtube](bmx4IHX5lhI)`
- setup steps
- required services:
  - [Blotato](https://blotato.com/?ref=firas)
  - [AtlasCloud](https://www.atlascloud.ai?ref=8QKPJE)
  - Anthropic
  - ElevenLabs
  - Shotstack
  - Nano Banana

### Sticky Note3
Marks Part 1:
- `Part 1 — From Drawing to Story: Characters, Images & Narrative`

### Sticky Note
Marks Part 2:
- `Part 2 — From Characters to Scenes: Rendering the Visual Story`

### Sticky Note1
Marks Part 3:
- `IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | formTrigger | Form-based workflow entry for drawing upload |  | Extract from File | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Extract from File | extractFromFile | Convert uploaded drawing binary into property/base64 data | On form submission | Convert to File | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Convert to File | convertToFile | Rebuild binary file for Gemini image analysis | Extract from File | Analyze an image | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Analyze an image | googleGemini | Vision analysis of child drawing into structured character JSON | Convert to File | Edit Fields | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Edit Fields | set | Normalize Gemini response text field | Analyze an image | Parse Characters | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Parse Characters | code | Parse JSON and emit one item per character | Edit Fields | Build Character Prompt | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Build Character Prompt | code | Aggregate character descriptions for prompt generation | Parse Characters | Generate Character Prompts | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Anthropic Chat Model1 | lmChatAnthropic | LLM provider for character prompt generation |  | Generate Character Prompts | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Structured Output Parser2 | outputParserStructured | Enforce schema for character prompt output |  | Generate Character Prompts | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Generate Character Prompts | chainLlm | Generate one image prompt per character | Build Character Prompt | Split Character Prompts | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Split Character Prompts | code | Expand LLM prompt list into one item per character | Generate Character Prompts | Nano Banana Payload | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Nano Banana Payload | code | Build Gemini image-generation payload for each character | Split Character Prompts | Nano Banana Generate | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Nano Banana Generate | httpRequest | Generate character image from prompt + source drawing | Nano Banana Payload | Extract Character Image | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Extract Character Image | set | Extract generated character image base64 | Nano Banana Generate | Character Image to File | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Character Image to File | convertToFile | Convert generated character image to binary file | Extract Character Image | Upload an asset from file data1 | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Upload an asset from file data1 | cloudinary | Upload generated character image to Cloudinary | Character Image to File | Map Character URLs | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Map Character URLs | code | Map Cloudinary character image URLs back to names | Upload an asset from file data1 | Build Story Context | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Build Story Context | code | Aggregate character references for story generation | Map Character URLs | Generate Story | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Anthropic Chat Model | lmChatAnthropic | LLM provider for story generation |  | Generate Story | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Structured Output Parser | outputParserStructured | Enforce schema for story output |  | Generate Story | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Generate Story | chainLlm | Generate children’s story chunks and scene prompts | Build Story Context | Split Story Chunks | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Split Story Chunks | code | Expand structured story into per-scene items | Generate Story | Extract Unique Character URLs | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Extract Unique Character URLs | code | Deduplicate character image URLs used in story | Split Story Chunks | Build Scene Payload | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Build Scene Payload | code | Prepare payload for scene-rendering workflow | Extract Unique Character URLs | Trigger Scene Workflow | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Trigger Scene Workflow | httpRequest | Send story package to Part 2 via webhook | Build Scene Payload |  | Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Webhook | webhook | Part 2 webhook entry point |  | Split Scene Chunks | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Split Scene Chunks | code | Expand incoming chunk list for scene workflow | Webhook | Extract Character URLs | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Extract Character URLs | code | Expand incoming character image URL list | Split Scene Chunks | Download Character Images | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Download Character Images | httpRequest | Download character images by URL | Extract Character URLs | Character Images to Base64 | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Character Images to Base64 | extractFromFile | Convert downloaded character images to base64 | Download Character Images | Build Base64 Map | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Build Base64 Map | code | Build indexed base64 map of character images | Character Images to Base64 | Build Scene Image Payloads | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Build Scene Image Payloads | code | Build multimodal payload for scene image generation | Build Base64 Map | Nano Banana Scene Generate | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Nano Banana Scene Generate | httpRequest | Generate scene illustrations with character references | Build Scene Image Payloads | Extract Scene Image | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Extract Scene Image | set | Extract generated scene image base64 | Nano Banana Scene Generate | Scene Image to File | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Scene Image to File | convertToFile | Convert generated scene image to binary | Extract Scene Image | Upload an asset from file data | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Upload an asset from file data | cloudinary | Upload generated scene image to Cloudinary | Scene Image to File | Map Scene URLs | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Map Scene URLs | code | Map scene image URLs back onto story chunks | Upload an asset from file data | Trigger Video Workflow | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Trigger Video Workflow | httpRequest | Send scene package to Part 3 via webhook | Map Scene URLs |  | Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Webhook1 | webhook | Part 3 webhook entry point |  | Split Video Chunks | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Split Video Chunks | code | Expand scene package and store global title/chunks | Webhook1 | Build Narration Context | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Build Narration Context | code | Aggregate all scenes for video prompt generation | Split Video Chunks | Generate Video Prompts | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Anthropic Chat Model2 | lmChatAnthropic | LLM provider for video prompt generation |  | Generate Video Prompts | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Structured Output Parser1 | outputParserStructured | Enforce schema for video prompts and short narration |  | Generate Video Prompts | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Generate Video Prompts | chainLlm | Generate animation prompts and short narration per scene | Build Narration Context | Merge Video Prompts | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Merge Video Prompts | code | Merge generated prompts with original scene data | Generate Video Prompts | Build Kling Payloads | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Build Kling Payloads | code | Build Atlas Cloud/Kling video generation payloads | Merge Video Prompts | Atlas Cloud Generate Video | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Atlas Cloud Generate Video | httpRequest | Submit image-to-video jobs to Atlas Cloud | Build Kling Payloads | Wait | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Wait | wait | Delay before polling video generation result | Atlas Cloud Generate Video, If | Atlas Cloud Poll Result | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Atlas Cloud Poll Result | httpRequest | Poll Atlas Cloud prediction status | Wait | If | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| If | if | Check whether video generation completed | Atlas Cloud Poll Result | Extract Video URLs, Wait | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Extract Video URLs | code | Extract generated video URLs from Atlas result | If | Prepare Narration Loop | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Prepare Narration Loop | code | Build ElevenLabs narration payloads | Extract Video URLs | Loop Over Items | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Loop Over Items | splitInBatches | Iterate narration/audio generation workflow | Prepare Narration Loop, Wait1 | Extract Audio URLs, ElevenLabs Narrate | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| ElevenLabs Narrate | httpRequest | Generate narration audio file | Loop Over Items | Upload an asset from file data2 | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Upload an asset from file data2 | cloudinary | Upload narration audio to Cloudinary | ElevenLabs Narrate | Wait1 | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Wait1 | wait | Resume loop after each uploaded narration file | Upload an asset from file data2 | Loop Over Items | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Extract Audio URLs | code | Collect Cloudinary audio URLs | Loop Over Items | Build Shotstack Payload | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Build Shotstack Payload | code | Combine chunks, video URLs, and audio URLs | Extract Audio URLs | Build Shotstack Timeline | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Build Shotstack Timeline | code | Build Shotstack multi-track render timeline | Build Shotstack Payload | Shotstack Render | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Shotstack Render | httpRequest | Submit final video render job | Build Shotstack Timeline | Wait2 | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Wait2 | wait | Delay before polling Shotstack render | Shotstack Render | Shotstack Poll Result | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Shotstack Poll Result | httpRequest | Poll final assembled video render | Wait2 | Generate Video Prompts1 | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Anthropic Chat Model3 | lmChatAnthropic | LLM provider for YouTube metadata generation |  | Generate Video Prompts1 | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Structured Output Parser3 | outputParserStructured | Enforce schema for publish title/description |  | Generate Video Prompts1 | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Generate Video Prompts1 | chainLlm | Generate video title and description | Shotstack Poll Result | Youtube, Tiktok | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Youtube | blotato | Publish final MP4 to YouTube via Blotato | Generate Video Prompts1 |  | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Tiktok | blotato | Optional TikTok publishing via Blotato, currently disabled | Generate Video Prompts1 |  | IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |
| Sticky Note2 | stickyNote | Global notes and setup guidance |  |  | # 🎬 From Drawing to Story — Auto-Publish AI Video to YouTube with Blotato |
| Sticky Note3 | stickyNote | Annotation for Part 1 |  |  | ## Part 1 — From Drawing to Story: Characters, Images & Narrative |
| Sticky Note | stickyNote | Annotation for Part 2 |  |  | ## Part 2 — From Characters to Scenes: Rendering the Visual Story |
| Sticky Note1 | stickyNote | Annotation for Part 3 |  |  | ## IPart 3 — From Scenes to Screen: Prompts, Audio & Final Assembly |

---

# 4. Reproducing the Workflow from Scratch

Below is the most practical way to rebuild this system: as **three separate workflows**.

## Part 1 — Drawing to Characters and Story

1. **Create a new workflow** in n8n named something like `Part 1 - Drawing to Story`.

2. **Add a Form Trigger node**
   - Node type: **On form submission / Form Trigger**
   - Form title: `SketchTales`
   - Add one field:
     - Name: `Image`
     - Type: `file`
     - Label: `Upload your child's drawing`
     - Required: yes
     - Multiple files: no

3. **Add Extract from File**
   - Operation: `Binary to Property`
   - Binary property name: `Image`

4. **Add Convert to File**
   - Operation: `To Binary`
   - Source property: `data`

5. **Connect**
   - Form Trigger → Extract from File → Convert to File

6. **Add Google Gemini node**
   - Type: `Google Gemini`
   - Resource: `image`
   - Operation: `analyze`
   - Input type: `binary`
   - Model: `gemini-3-pro-preview` or nearest available equivalent
   - Prompt: instruct Gemini to extract all characters, fun names, descriptions, setting, mood, and return JSON only
   - Configure **Google Gemini / PaLM credential**

7. **Add Set node named Edit Fields**
   - Assign field:
     - `content.parts[0].text` = `{{$json.content.parts[0].text}}`

8. **Add Code node named Parse Characters**
   - Parse `content.parts[0].text`
   - Remove markdown fences if needed
   - `JSON.parse(...)`
   - Emit one item per character with:
     - `character_index`
     - `character_name`
     - `character_desc`
     - `character_count`

9. **Connect**
   - Convert to File → Analyze an image → Edit Fields → Parse Characters

10. **Add Code node named Build Character Prompt**
    - Aggregate all character items
    - Build:
      - `character_count`
      - `character_list`
      - `json_format`

11. **Add Anthropic Chat Model node**
    - Model: `Claude Sonnet 4.5` or currently available equivalent
    - Configure **Anthropic credential**

12. **Add Structured Output Parser node**
    - Schema example:
      - object with `prompts[]`
      - each prompt has `character_index`, `character_name`, `prompt`

13. **Add Basic LLM Chain node named Generate Character Prompts**
    - Prompt should:
      - ask for one image prompt per character
      - enforce same style across all characters
      - require exact JSON output
    - Connect Anthropic model to LM port
    - Connect parser to output parser port

14. **Add Code node named Split Character Prompts**
    - Read `output.prompts`
    - Emit one item per character prompt

15. **Add Code node named Nano Banana Payload**
    - For each prompt create a JSON body for Gemini image generation
    - Include:
      - prompt text
      - original drawing as `inline_data`
      - response modalities `TEXT`, `IMAGE`
      - image aspect ratio `16:9`
      - image size `1K`
    - Preferably use the real MIME type instead of hardcoded `image/png`

16. **Add HTTP Request node named Nano Banana Generate**
    - Method: `POST`
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent`
    - Authentication: predefined credential type using Google Gemini credential
    - Header: `Content-Type: application/json`
    - Send JSON body from payload
    - Enable retries

17. **Add Set node named Extract Character Image**
    - Assign `data = {{$json.candidates[0].content.parts[0].inlineData.data}}`

18. **Add Convert to File named Character Image to File**
    - Operation: `To Binary`
    - Source property: `data`

19. **Add Cloudinary node named Upload an asset from file data1**
    - Operation: `uploadFile`
    - Configure **Cloudinary credential**

20. **Add Code node named Map Character URLs**
    - Match Cloudinary output URLs to parsed character names
    - Output:
      - `character_index`
      - `character_name`
      - `character_image_url`

21. **Connect the whole character image branch**
    - Parse Characters → Build Character Prompt → Generate Character Prompts → Split Character Prompts → Nano Banana Payload → Nano Banana Generate → Extract Character Image → Character Image to File → Upload an asset from file data1 → Map Character URLs

22. **Add Code node named Build Story Context**
    - Aggregate all mapped character image records
    - Build:
      - `character_count`
      - `character_list`
      - `character_map`

23. **Add another Anthropic Chat Model**
    - Same credential/model family

24. **Add another Structured Output Parser**
    - Schema:
      - `title`
      - `chunks[]`
        - `chunk_number`
        - `script`
        - `image_prompt`
        - `characters_used`
        - `duration_seconds`

25. **Add LLM Chain named Generate Story**
    - Prompt asks for a children’s story in 4–6 chunks
    - Rules:
      - exact character names
      - warm magical tone
      - image prompts start with “Using the attached character images as reference, create a scene showing...”
      - valid JSON only

26. **Add Code node named Split Story Chunks**
    - Read `Generate Story.output`
    - Expand chunks to one item per scene
    - Attach `character_images` by looking up used character indices from `Map Character URLs`

27. **Add Code node named Extract Unique Character URLs**
    - Deduplicate all character image URLs across chunks

28. **Add Code node named Build Scene Payload**
    - Build final payload:
      - `title`
      - `characters[]`
      - `chunks[]`
    - Keep original character indexing if possible; avoid reindexing bug

29. **Add HTTP Request node named Trigger Scene Workflow**
    - Method: `POST`
    - URL: webhook endpoint of Part 2
    - Send JSON body = current item

30. **Connect story branch**
    - Map Character URLs → Build Story Context → Generate Story → Split Story Chunks → Extract Unique Character URLs → Build Scene Payload → Trigger Scene Workflow

---

## Part 2 — Characters to Scenes

31. **Create a second workflow** named `Part 2 - Render Scenes`.

32. **Add Webhook node**
    - Method: `POST`
    - Path: choose a unique path
    - This is the endpoint called by Part 1

33. **Add Code node named Split Scene Chunks**
    - Read `body.chunks`
    - Emit one item per chunk

34. **Add Code node named Extract Character URLs**
    - Read `body.characters`
    - Emit one item per character with `character_index`, `image_url`

35. **Add HTTP Request node named Download Character Images**
    - URL = `{{$json.image_url}}`
    - Response as file/binary

36. **Add Extract from File node named Character Images to Base64**
    - Operation: `Binary to Property`

37. **Add Code node named Build Base64 Map**
    - Aggregate downloaded character images into array:
      - `character_index`
      - `base64`
      - `mimetype`

38. **Add Code node named Build Scene Image Payloads**
    - For each chunk:
      - start with `image_prompt`
      - append each used character image as `inline_data`
      - set Gemini image generation config:
        - response modalities: TEXT + IMAGE
        - aspect ratio 16:9
        - image size 1K

39. **Add HTTP Request node named Nano Banana Scene Generate**
    - Same Gemini endpoint and auth as Part 1 image generation

40. **Add Set node named Extract Scene Image**
    - `data = {{$json.candidates[0].content.parts[0].inlineData.data}}`

41. **Add Convert to File named Scene Image to File**
    - Source property `data`

42. **Add Cloudinary node named Upload an asset from file data**
    - Upload generated scene image

43. **Add Code node named Map Scene URLs**
    - Combine original chunk metadata with uploaded `scene_image_url`

44. **Add HTTP Request node named Trigger Video Workflow**
    - POST to Part 3 webhook endpoint
    - Send JSON body with title and chunks

45. **Connect**
    - Webhook → Split Scene Chunks → Extract Character URLs → Download Character Images → Character Images to Base64 → Build Base64 Map → Build Scene Image Payloads → Nano Banana Scene Generate → Extract Scene Image → Scene Image to File → Upload an asset from file data → Map Scene URLs → Trigger Video Workflow

---

## Part 3 — Scenes to Final Video and Publishing

46. **Create a third workflow** named `Part 3 - Video Assembly and Publishing`.

47. **Add Webhook node named Webhook1**
    - Method: `POST`
    - Path: unique path
    - This is the endpoint called by Part 2

48. **Add Code node named Split Video Chunks**
    - Read `body.chunks`
    - Store `title` and `chunks` in workflow static data
    - Emit one item per chunk

49. **Add Code node named Build Narration Context**
    - Build aggregate text listing each scene’s script, image prompt, and duration

50. **Add Anthropic Chat Model + Structured Output Parser**
    - Parser schema:
      - `scenes[]`
        - `chunk_number`
        - `video_prompt`
        - `narration`

51. **Add LLM Chain named Generate Video Prompts**
    - Prompt asks for:
      - one animation prompt per scene
      - one shortened narration line per scene
    - Enforce:
      - prompt starts with `Use the provided image as the first frame.`
      - max ~18 words narration

52. **Add Code node named Merge Video Prompts**
    - Merge generated prompts with original scene image URLs and durations

53. **Add Code node named Build Kling Payloads**
    - For each item build Atlas payload:
      - model `kwaivgi/kling-v3.0-pro/image-to-video`
      - `cfg_scale: 0.5`
      - `duration`
      - `image`
      - optional `end_image`
      - `prompt`
      - `negative_prompt`
      - `sound: true`

54. **Add HTTP Request named Atlas Cloud Generate Video**
    - POST `https://api.atlascloud.ai/api/v1/model/generateVideo`
    - Headers:
      - `Authorization: Bearer <YOUR_TOKEN>`
      - `Content-Type: application/json`
    - Use actual secret, not inline literal in production

55. **Add Wait node**
    - 11 minutes

56. **Add HTTP Request named Atlas Cloud Poll Result**
    - GET `https://api.atlascloud.ai/api/v1/model/prediction/{{ $json.data.id }}`
    - Same bearer token header

57. **Add If node**
    - Condition: status equals `completed`
    - True → continue
    - False → back to Wait
    - Ideally add explicit failure branch for `failed`

58. **Add Code node named Extract Video URLs**
    - Map completed outputs into `video_url`

59. **Add Code node named Prepare Narration Loop**
    - Read `Generate Video Prompts.output.scenes`
    - Build ElevenLabs JSON body for each narration

60. **Add Split in Batches node named Loop Over Items**
    - Use to process one narration item at a time

61. **Add HTTP Request node named ElevenLabs Narrate**
    - POST `https://api.elevenlabs.io/v1/text-to-speech/<VOICE_ID>`
    - Response format: file
    - Headers:
      - `xi-api-key: <YOUR_KEY>`
      - `Content-Type: application/json`

62. **Add Cloudinary node named Upload an asset from file data2**
    - Upload audio as raw file

63. **Add Wait node named Wait1**
    - Use to resume loop if needed

64. **Wire narration loop**
    - Prepare Narration Loop → Loop Over Items → ElevenLabs Narrate → Upload an asset from file data2 → Wait1 → Loop Over Items

65. **Add Code node named Extract Audio URLs**
    - Collect Cloudinary audio URLs
    - Prefer preserving original chunk numbers

66. **Add Code node named Build Shotstack Payload**
    - Combine chunk metadata, video URLs, and audio URLs
    - Read title from static data

67. **Add Code node named Build Shotstack Timeline**
    - Build timeline with 2 tracks:
      - video clips
      - audio clips
    - Set output to MP4 HD

68. **Add HTTP Request named Shotstack Render**
    - POST `https://api.shotstack.io/edit/v1/render`
    - Headers:
      - `x-api-key: <YOUR_KEY>`
      - `Content-Type: application/json`

69. **Add Wait node named Wait2**
    - 1 minute

70. **Add HTTP Request named Shotstack Poll Result**
    - GET `https://api.shotstack.io/edit/v1/render/{{ $json.response.id }}`
    - Same Shotstack API key

71. **Add metadata generation chain**
    - Anthropic model
    - Structured output parser
    - LLM chain named `Generate Video Prompts1`
    - Prompt should generate:
      - `video_title`
      - `video_description`

72. **Important correction**
    - Fix parser schema so it matches the prompt:
      - expected output should be:
        - `video_title`
        - `video_description`
      - not `scenes[]`

73. **Add Blotato node named Youtube**
    - Platform: `youtube`
    - Connect **Blotato credential**
    - Select the desired account
    - Title = generated `video_title`
    - Description/text = generated `video_description`
    - Media URL = final Shotstack URL
    - Set:
      - made for kids = true
      - notify subscribers = false

74. **Optional TikTok node**
    - Add Blotato TikTok node if needed
    - This template includes one but it is disabled

75. **Connect final section**
    - Extract Audio URLs → Build Shotstack Payload → Build Shotstack Timeline → Shotstack Render → Wait2 → Shotstack Poll Result → Generate Video Prompts1 → Youtube

---

## Required Credentials

Configure these before testing:

1. **Google Gemini / PaLM API**
   - Used by:
     - Analyze an image
     - Nano Banana Generate
     - Nano Banana Scene Generate

2. **Anthropic API**
   - Used by:
     - Generate Character Prompts
     - Generate Story
     - Generate Video Prompts
     - Generate Video Prompts1

3. **Cloudinary**
   - Used for:
     - Character image uploads
     - Scene image uploads
     - Narration audio uploads

4. **Atlas Cloud**
   - Used via HTTP Request bearer token
   - Replace `YOUR_TOKEN_HERE`

5. **ElevenLabs**
   - Used via HTTP Request
   - Replace `YOUR_API_HERE`
   - Confirm voice ID is valid

6. **Shotstack**
   - Used via HTTP Request
   - Replace `YOUR_API_HERE`

7. **Blotato**
   - Used by YouTube/TikTok nodes
   - Requires linked social account access

---

## Recommended Corrections Before Production Use

1. Replace all `/webhook-test/` URLs with production `/webhook/` URLs.
2. Fix `Structured Output Parser3` schema mismatch.
3. Preserve original character indices instead of reindexing deduplicated URLs.
4. Preserve original chunk numbers for audio/video mapping.
5. Add failure handling for Atlas `failed` and Shotstack non-complete responses.
6. Use credentials/secrets instead of hardcoded placeholder headers.
7. Store actual MIME type for uploaded drawing rather than hardcoded `image/png`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| From Drawing to Story — Auto-Publish AI Video to YouTube with Blotato | Global project note |
| Transform a hand-drawn sketch into a fully narrated, animated video — automatically generated and published to YouTube. | Global project note |
| This template is split into 3 separate workflows. Each part must be imported into its own workflow in n8n. | Global project note |
| Tutorial video reference | `@[youtube](bmx4IHX5lhI)` |
| Part 1 — Analyze the drawing, generate character images & write the story | Workflow architecture note |
| Part 2 — Generate scene images with character consistency | Workflow architecture note |
| Part 3 — Generate videos, narrate with AI audio & auto-publish to YouTube | Workflow architecture note |
| Blotato — YouTube / TikTok publishing | https://blotato.com/?ref=firas |
| AtlasCloud — Kling Pro 3.0 video generation | https://www.atlascloud.ai?ref=8QKPJE |
| Anthropic — Claude AI for story and prompt generation | Credential requirement |
| ElevenLabs — narration audio | Credential requirement |
| Shotstack — final video assembly | Credential requirement |
| Nano Banana — image generation | Credential requirement |
| Configure credentials for each service in n8n before testing | Setup note |
| Set up a form trigger with a file upload field for the drawing | Setup note |
| Deploy the 3 workflows in order and connect them via webhooks | Setup note |
| Run a test submission with a simple sketch to validate the full pipeline | Setup note |

If you want, I can also convert this into a **clean implementation spec per workflow (Part 1 / Part 2 / Part 3)** with corrected logic and identified fixes applied.