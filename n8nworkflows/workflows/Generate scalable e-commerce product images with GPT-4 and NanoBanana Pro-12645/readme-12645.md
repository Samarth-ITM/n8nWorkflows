Generate scalable e-commerce product images with GPT-4 and NanoBanana Pro

https://n8nworkflows.xyz/workflows/generate-scalable-e-commerce-product-images-with-gpt-4-and-nanobanana-pro-12645


# Generate scalable e-commerce product images with GPT-4 and NanoBanana Pro

## 1. Workflow Overview

**Workflow name:** 💥 Create scalable e-commerce product images using NanoBanana Pro  
**Stated title:** Generate scalable e-commerce product images with GPT-4 and NanoBanana Pro

**Purpose:**  
Collect exactly **3 reference images** via an n8n Form, analyze each image with **OpenAI Vision (GPT‑4o)**, generate a structured **photoshoot prompt** via a LangChain Agent using **GPT‑4.1‑mini**, then call **AtlasCloud NanoBanana Pro** to generate a new edited/stylized product image. The final generated image is downloaded, uploaded to **Google Drive**, and logged to **Google Sheets** along with source URLs and descriptions.

**Typical use cases:**
- Rapid generation of consistent studio-style product imagery from a few reference photos
- Scaling content creation for e-commerce listings (fashion/editorial style output)
- Producing repeatable “house style” outputs driven by a stable system prompt

### 1.1 Input Reception & Validation
- Receives 3 uploaded files and optional notes.
- Validates that all 3 images are present; otherwise returns a structured error payload.

### 1.2 Per-image Parallel Processing (Storage + Vision + Public URL)
- Splits the 3 uploads into 3 items.
- For each item (in parallel):
  - Uploads original image to **Google Drive**
  - Sends the image to **OpenAI Vision** for a detailed description
  - Uploads the image to **tmpfiles.org** to obtain a publicly accessible URL (used by NanoBanana)

### 1.3 Aggregation & Prompt Generation
- Merges the 3 branches “by position”.
- Aggregates the three descriptions and URLs into a single consolidated JSON object.
- Uses a LangChain Agent with an OpenAI chat model + structured output parser to produce `image_prompt` JSON.

### 1.4 Image Generation (NanoBanana Pro) & Result Logging
- Calls AtlasCloud `generateImage` with:
  - Public reference image URLs
  - The generated prompt
  - Resolution/aspect/output options
- Waits (async pattern), then polls prediction result.
- Downloads the final PNG (binary), uploads it to Google Drive, then appends a row in Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Form Input & Validation
**Overview:** Accepts 3 images via an n8n form and ensures all required binaries exist before proceeding. If not, returns a JSON error message.

**Nodes involved:**
- Form Trigger (3 images)
- If
- Error Response - Missing Files
- Normalize binary names

#### Node: Form Trigger (3 images)
- **Type / role:** `n8n-nodes-base.formTrigger` — entry point that exposes an n8n-hosted form for file uploads.
- **Key configuration:**
  - Title: “Upload 3 Images for Analysis”
  - Button label: “Analyze Images”
  - Fields:
    - `image1` (file, required)
    - `Image2` (file, required) **note the capital I**
    - `Image3` (file, required) **note the capital I**
    - `notes` (text, optional)
  - Accepts common image formats: jpg/jpeg/png/gif/bmp/webp
- **Outputs:** One item containing form fields in `json` and uploaded files in `binary`.
- **Potential failures / edge cases:**
  - Users upload fewer than 3 files (handled by the If node).
  - Users upload unsupported file types (blocked by form accept list).
  - Large files may hit n8n upload limits / reverse proxy limits.

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — validates that all required binary properties exist.
- **Configuration (interpreted):**
  - AND condition with three “exists” checks:
    - `$binary.image1` exists
    - `$binary.Image2` exists
    - `$binary.Image3` exists
- **Routing:**
  - **True** → Normalize binary names
  - **False** → Error Response - Missing Files
- **Potential failures / edge cases:**
  - Case sensitivity: field names must match exactly (`Image2`/`Image3` vs `image2`/`image3`).

#### Node: Error Response - Missing Files
- **Type / role:** `n8n-nodes-base.set` — creates an error payload (does not terminate automatically unless your execution consumer treats it as such).
- **Configuration:**
  - Sets `json.error = "Please upload 3 images (image1, image2, image3)."`
- **Outputs:** A single JSON item with `error`.
- **Edge cases:**
  - Message references `image2/image3` lowercase, but the actual form uses `Image2/Image3` (capitalized). This is only a wording mismatch but may confuse users.

#### Node: Normalize binary names
- **Type / role:** `n8n-nodes-base.set` — attempts to normalize binaries into `image0/image1/image2`.
- **Configuration:**
  - Adds JSON fields:
    - `image0 = $binary.image1`
    - `image1 = $binary.Image2`
    - `image2 = $binary.Image3`
  - `includeOtherFields = true`
- **Important behavior note:** This node sets **JSON object fields**, not the `binary` section. Downstream, the “Split images” node still expects the originals in `item.binary.image1/Image2/Image3`.
- **Edge cases:**
  - This node currently does not actually rename the binary keys; it only copies references into JSON. If the intent was to rename binary properties, a different approach is needed (e.g., Code node to rewrite `item.binary`).

---

### Block 2.2 — Split & Parallel Per-Image Processing
**Overview:** Turns the single form submission into 3 items (one per image) and processes each item in parallel: Drive upload, GPT-4o vision analysis, and tmpfiles upload for a public URL.

**Nodes involved:**
- Split images
- Upload file
- Analyze image
- Build Public Image URL
- Merge

#### Node: Split images
- **Type / role:** `n8n-nodes-base.code` — splits 1 incoming item into 3 items.
- **Key logic (interpreted):**
  - Validates presence of `item.binary.image1`, `item.binary.Image2`, `item.binary.Image3`
  - Returns three items:
    - Item 1: `json.imageNumber = 1`, `binary.image = item.binary.image1`
    - Item 2: `json.imageNumber = 2`, `binary.image = item.binary.Image2`
    - Item 3: `json.imageNumber = 3`, `binary.image = item.binary.Image3`
- **Connections:**
  - Output → (in parallel) Upload file, Analyze image, Build Public Image URL
- **Potential failures / edge cases:**
  - Throws an error if any of the binaries are missing (hard fail).
  - Depends on exact binary key casing from the form.

#### Node: Upload file
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads each image to Google Drive.
- **Key configuration:**
  - Operation: Upload (implicit by node)
  - Input binary property: `image`
  - File name: `{{$binary.image.fileName}}`
  - Drive ID: placeholder `<__PLACEHOLDER_VALUE__Google DRIVE Document ID___>`
  - Folder ID: placeholder `<__PLACEHOLDER_VALUE__Google DRIVE Document ID___>`
  - Credentials: Google Drive OAuth2
- **Outputs:** Drive file metadata (commonly includes `webViewLink` / `webContentLink` depending on Drive settings).
- **Potential failures / edge cases:**
  - OAuth scope/consent problems, expired refresh token.
  - Folder/Drive ID wrong or not writable.
  - `webContentLink` may not exist unless file sharing/permissions allow it; may require “Anyone with the link” permissions if used externally.

#### Node: Analyze image
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI “analyze image”) — vision description per image.
- **Key configuration:**
  - Resource: Image
  - Operation: Analyze
  - Input type: base64 from binary
  - Binary property: `image`
  - Model: `gpt-4o`
  - Prompt text: “Describe the image in detail…”
- **Outputs:** An OpenAI response object (structure can vary by node version).
- **Potential failures / edge cases:**
  - OpenAI auth errors, quota exceeded, model access issues.
  - Large images can increase latency/cost; may time out if n8n limits are low.
  - Response structure changes may break downstream extraction.

#### Node: Build Public Image URL
- **Type / role:** `n8n-nodes-base.httpRequest` — uploads each image to tmpfiles.org to get a public URL.
- **Key configuration:**
  - POST `https://tmpfiles.org/api/v1/upload`
  - Multipart form-data with `file` from binary property `image`
  - Response format: JSON
- **Outputs:** JSON including `data.url` (tmpfiles link).
- **Potential failures / edge cases:**
  - tmpfiles rate limits / availability.
  - Returned URL is not always in direct-download form; workflow later converts it.
  - Public hosting is third-party; links may expire.

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines outputs from the three parallel branches.
- **Key configuration:**
  - Mode: Combine
  - Combine by: Position
  - Number of inputs: 3
- **Input mapping (as wired):**
  - Input 0: Upload file
  - Input 1: Build Public Image URL
  - Input 2: Analyze image
- **Outputs:** One item per image position where `json` becomes a composite object with keys `"0"`, `"1"`, `"2"` holding each branch’s JSON.
- **Potential failures / edge cases:**
  - If one branch returns fewer items or errors, the merge may misalign or produce incomplete composites.

---

### Block 2.3 — Aggregate Results & Create a Structured Prompt
**Overview:** Consolidates all three descriptions and URLs into one JSON object, then generates a structured `image_prompt` using an agent with a system prompt and output schema.

**Nodes involved:**
- Aggregate descriptions
- OpenAI Chat Model
- Structured Output Parser
- Generate Image Prompt

#### Node: Aggregate descriptions
- **Type / role:** `n8n-nodes-base.code` — extracts per-image info from the merged structure and formats final fields.
- **Key behaviors:**
  - Extracts OpenAI text from `it.json["0"]` (comment notes: “Après Merge by position, OpenAI est sous la clé "0"”).
    - **But in this workflow wiring, OpenAI is actually merged into key `"2"` (input index 2).**
  - Extracts Drive URLs from `it.json.webContentLink || it.json.webViewLink`
    - **But after Merge, Drive JSON is under key `"0"`, not top-level.**
  - Extracts tmpfiles URL from `it.json.data.url`
    - **But tmpfiles JSON is under key `"1"`, not top-level.**
  - Converts tmpfiles URL format:
    - `http://tmpfiles.org/<id>/<name>` → `https://tmpfiles.org/dl/<id>/<name>`
  - Produces a single consolidated output with:
    - `image1Description/image2Description/image3Description`
    - `image1Url/image2Url/image3Url` (Drive)
    - `imageUrls` (array of Drive URLs)
    - `image1Url_public/...` and `imageUrls_public` (array for NanoBanana)
    - `allDescriptions` (multi-paragraph combined string)
- **Connections:**
  - Output → Generate Image Prompt
- **Major edge cases / likely bug:**
  - Because of mismatched keys after `Merge`, this code will often output “No description” / “No link”.
  - This in turn can make NanoBanana calls fail (missing/invalid `images` array), or generate poor prompts (no descriptions).

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for the agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - Responses API disabled
- **Connections:**
  - Connected to `Generate Image Prompt` via `ai_languageModel`.
- **Potential failures / edge cases:**
  - Same OpenAI issues as above (auth/quota/model availability).
  - If model changes output style, parser may fail.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON output schema.
- **Schema example:**
  - `{ "image_prompt": "string" }`
- **Connections:**
  - Connected to `Generate Image Prompt` via `ai_outputParser`.
- **Potential failures / edge cases:**
  - If the agent outputs non-JSON or missing `image_prompt`, parsing fails.

#### Node: Generate Image Prompt
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates a “fashion studio photoshoot” prompt from `allDescriptions`.
- **Configuration (interpreted):**
  - User message (`text`) includes:
    - `{{ $json.allDescriptions }}`
  - System message: fixed template; agent must output a prompt variation with only:
    - hand positioning changes
    - how product is worn/placed
    - everything else unchanged
  - `hasOutputParser = true` (must comply with schema providing `image_prompt`)
- **Connections:**
  - Main output → NanoBanana: Create Image
- **Potential failures / edge cases:**
  - If `allDescriptions` is empty due to upstream extraction bug, prompt may be generic or invalid.
  - Agent may violate “change only X/Y” constraint; parser will still accept if JSON matches.

---

### Block 2.4 — NanoBanana Generation, Polling, Download, Storage, Logging
**Overview:** Sends prompt + reference image URLs to AtlasCloud NanoBanana Pro, waits, retrieves the prediction outputs, downloads the final PNG, uploads it to Drive, and logs metadata to Sheets.

**Nodes involved:**
- NanoBanana: Create Image
- Wait
- Download Edited Image
- Download Final PNG (binary)
- Upload file Nanobanana
- Append row in sheet

#### Node: NanoBanana: Create Image
- **Type / role:** `n8n-nodes-base.httpRequest` — calls AtlasCloud image generation API.
- **Key configuration:**
  - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
  - Headers:
    - `Authorization: Bearer <atlascloud Key placeholder>`
    - `Content-Type: application/json`
  - Body includes:
    - `model: "google/nano-banana-pro/edit"`
    - `aspect_ratio: "1:1"`
    - `resolution: "2k"`
    - `output_format: "png"`
    - `enable_sync_mode: false` (async)
    - `enable_base64_output: false`
    - `images`: `JSON.stringify($('Aggregate descriptions').item.json.imageUrls_public)`
    - `prompt`: from `$json.output.image_prompt` with escaping for newline/quotes
- **Connections:**
  - Output → Wait
- **Potential failures / edge cases:**
  - If `imageUrls_public` is empty or not publicly reachable, API may error.
  - Async mode requires correct polling; workflow uses a fixed Wait time rather than checking status repeatedly.
  - Prompt escaping is defensive; however embedding a JSON-stringified array directly into JSON body can be risky if it becomes a string rather than an array (depends on expression evaluation). Ideally `images` should be an array, not a string.

#### Node: Wait
- **Type / role:** `n8n-nodes-base.wait` — pauses execution for a configured number of minutes.
- **Key configuration:**
  - Unit: minutes
  - **No explicit duration value is shown** (must be set in the node UI; otherwise behavior may be default/invalid).
- **Connections:**
  - Output → Download Edited Image
- **Edge cases:**
  - If the prediction takes longer than the wait period, the next poll may return “still processing” or no outputs.
  - Too long wait wastes runtime; too short increases failures.

#### Node: Download Edited Image
- **Type / role:** `n8n-nodes-base.httpRequest` — polls prediction status/result.
- **Key configuration:**
  - GET `https://api.atlascloud.ai/api/v1/model/prediction/{{ $json.data.id }}`
  - Auth header Bearer token
  - Response: JSON
- **Connections:**
  - Output → Download Final PNG (binary)
- **Potential failures / edge cases:**
  - If `$json.data.id` is missing (upstream API error), URL becomes invalid.
  - If prediction not ready, `data.outputs` may be empty.

#### Node: Download Final PNG (binary)
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads the generated PNG as binary.
- **Key configuration:**
  - URL: `{{ $json.data.outputs[0] }}`
  - Response format: file (binary)
- **Connections:**
  - Output → Upload file Nanobanana
- **Potential failures / edge cases:**
  - `data.outputs[0]` missing or not a direct file URL.
  - Large file download may time out.

#### Node: Upload file Nanobanana
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads the final image to Google Drive.
- **Key configuration:**
  - File name: `{{ $json.data.id }}`
  - Drive ID + Folder ID: placeholders
  - **Binary input field is not explicitly set here.**
    - Many n8n nodes default to `data` or require specifying the binary property; ensure this node is configured to upload the binary produced by “Download Final PNG (binary)”.
- **Connections:**
  - Output → Append row in sheet
- **Potential failures / edge cases:**
  - Misconfigured binary property name leads to “no binary data” error.
  - Missing file extension (name is only the id) may be inconvenient; consider adding `.png`.

#### Node: Append row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — logs the run in a Google Sheet.
- **Key configuration:**
  - Operation: Append to a sheet tab (by ID placeholder)
  - Document ID: placeholder
  - Columns mapped:
    - `status = "nanobanana_done"`
    - `image_1 = {{ $('Aggregate descriptions').item.json.image1Url }}`
    - `image_2 = {{ $('Aggregate descriptions').item.json.image1Url }}` **likely a mistake (should be image2Url)**
    - `image_3 = {{ $('Aggregate descriptions').item.json.image3Url }}`
    - `description_all = {{ $('Aggregate descriptions').item.json.allDescriptions }}`
    - `image_nanobanana = {{ $json.webContentLink }}`
- **Connections:** terminal node.
- **Potential failures / edge cases:**
  - Permissions to edit the sheet, wrong sheet/tab IDs.
  - `image_nanobanana` depends on Drive upload output providing `webContentLink`; may be absent depending on Drive API response & sharing permissions.
  - Column mapping includes removed columns (marked removed in schema); ensure your actual sheet matches.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form Trigger (3 images) | n8n-nodes-base.formTrigger | Collect 3 images + notes via form | — | If | ## 🚀 AI Image Generation Workflow / (Notion link and setup notes) |
| If | n8n-nodes-base.if | Validate presence of 3 uploaded binaries | Form Trigger (3 images) | Normalize binary names; Error Response - Missing Files | ## Step 1: Input Validation … For each image in parallel … |
| Error Response - Missing Files | n8n-nodes-base.set | Output error payload when uploads missing | If (false) | — | ## Step 1: Input Validation … |
| Normalize binary names | n8n-nodes-base.set | Copy binary refs into JSON (attempted normalization) | If (true) | Split images | ## Step 1: Input Validation … |
| Split images | n8n-nodes-base.code | Split 1 item into 3 items (one per image) | Normalize binary names | Upload file; Analyze image; Build Public Image URL | ## Step 1: Input Validation … |
| Upload file | n8n-nodes-base.googleDrive | Upload each source image to Google Drive | Split images | Merge (input 0) | ## Step 1: Input Validation … |
| Analyze image | @n8n/n8n-nodes-langchain.openAi | Vision analysis (GPT-4o) per image | Split images | Merge (input 2) | ## Step 1: Input Validation … |
| Build Public Image URL | n8n-nodes-base.httpRequest | Upload each image to tmpfiles.org for public URL | Split images | Merge (input 1) | ## Step 1: Input Validation … |
| Merge | n8n-nodes-base.merge | Combine branch outputs by position (3 inputs) | Upload file; Build Public Image URL; Analyze image | Aggregate descriptions | ## Step 1: Input Validation … |
| Aggregate descriptions | n8n-nodes-base.code | Consolidate 3 descriptions + URLs into one JSON object | Merge | Generate Image Prompt | ## Step 2: Image Generation … |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM used by the agent (gpt-4.1-mini) | — (AI connection) | Generate Image Prompt (ai_languageModel) | ## Step 2: Image Generation … |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce `{image_prompt}` JSON output | — (AI connection) | Generate Image Prompt (ai_outputParser) | ## Step 2: Image Generation … |
| Generate Image Prompt | @n8n/n8n-nodes-langchain.agent | Create final prompt from aggregated descriptions | Aggregate descriptions + AI model/parser | NanoBanana: Create Image | ## Step 2: Image Generation … |
| NanoBanana: Create Image | n8n-nodes-base.httpRequest | Request image generation (AtlasCloud NanoBanana Pro) | Generate Image Prompt | Wait | ## Step 2: Image Generation … |
| Wait | n8n-nodes-base.wait | Pause before polling prediction | NanoBanana: Create Image | Download Edited Image |  |
| Download Edited Image | n8n-nodes-base.httpRequest | Poll prediction status/details | Wait | Download Final PNG (binary) |  |
| Download Final PNG (binary) | n8n-nodes-base.httpRequest | Download generated PNG as binary | Download Edited Image | Upload file Nanobanana |  |
| Upload file Nanobanana | n8n-nodes-base.googleDrive | Upload generated PNG to Google Drive | Download Final PNG (binary) | Append row in sheet |  |
| Append row in sheet | n8n-nodes-base.googleSheets | Log run metadata to Google Sheets | Upload file Nanobanana | — |  |
| Note - Workflow Overview | n8n-nodes-base.stickyNote | Comment block | — | — | (content: Step 1 overview) |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block + Notion link | — | — | (content includes: https://automatisation.notion.site/Create-scalable-e-commerce-product-images-from-photos-using-NanoBanana-Pro-2e33d6550fd9808e8891f7d606b49df7?source=copy_link) |
| Note - Workflow Overview1 | n8n-nodes-base.stickyNote | Comment block | — | — | (content: Step 2 overview) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: “💥 Create scalable e-commerce product images using NanoBanana Pro” (or your preferred name).

2) **Add node: Form Trigger** (`Form Trigger`)  
   - Form title: “Upload 3 Images for Analysis”  
   - Description: “Please upload exactly 3 images for AI-powered analysis”  
   - Button label: “Analyze Images”  
   - Add fields:
     - `image1` (File, required, single file, accept `.jpg,.jpeg,.png,.gif,.bmp,.webp`)
     - `Image2` (File, required, single file, same accept list)
     - `Image3` (File, required, single file, same accept list)
     - `notes` (Text, optional)

3) **Add node: If** (`IF`) and connect: **Form Trigger → If**  
   - Condition group: AND  
   - Add three conditions (Boolean → Exists):
     - Left value: `{{$binary.image1}}`
     - Left value: `{{$binary.Image2}}`
     - Left value: `{{$binary.Image3}}`

4) **Add node: Set** named “Error Response - Missing Files” and connect **If (false) → Error Response**  
   - Set field: `error` (String) = `Please upload 3 images (image1, image2, image3).`

5) **Add node: Set** named “Normalize binary names” and connect **If (true) → Normalize binary names**  
   - Enable “Include Other Fields”  
   - Add fields:
     - `image0` (Object) = `{{$binary.image1}}`
     - `image1` (Object) = `{{$binary.Image2}}`
     - `image2` (Object) = `{{$binary.Image3}}`

6) **Add node: Code** named “Split images” and connect **Normalize binary names → Split images**  
   - Paste logic that emits 3 items, each having `binary.image` set to one of the uploaded images, and `json.imageNumber` 1..3.
   - Ensure it references correct binary keys: `image1`, `Image2`, `Image3`.

7) **Add node: Google Drive** named “Upload file” and connect **Split images → Upload file**  
   - Credentials: Google Drive OAuth2
   - Operation: Upload
   - Binary property: `image`
   - File name expression: `{{$binary.image.fileName}}`
   - Set Drive and Folder IDs to your target locations.

8) **Add node: OpenAI (Image Analyze)** named “Analyze image” and connect **Split images → Analyze image**  
   - Credentials: OpenAI API
   - Resource: Image → Analyze
   - Model: `gpt-4o`
   - Binary property: `image`
   - Prompt: “Describe the image in detail…” (as in workflow)

9) **Add node: HTTP Request** named “Build Public Image URL” and connect **Split images → Build Public Image URL**  
   - Method: POST
   - URL: `https://tmpfiles.org/api/v1/upload`
   - Content type: multipart/form-data
   - Body parameter:
     - Name: `file`
     - Type: Binary
     - Binary property: `image`
   - Response format: JSON

10) **Add node: Merge** named “Merge”  
   - Mode: Combine  
   - Combine by: Position  
   - Number of inputs: 3  
   - Connect:
     - **Upload file → Merge (input 0)**
     - **Build Public Image URL → Merge (input 1)**
     - **Analyze image → Merge (input 2)**

11) **Add node: Code** named “Aggregate descriptions” and connect **Merge → Aggregate descriptions**  
   - Implement extraction that matches your merge structure:
     - Drive result is under `json["0"]`
     - tmpfiles result is under `json["1"]`
     - OpenAI vision result is under `json["2"]`
   - Output a single item containing:
     - `image1Description/image2Description/image3Description`
     - `imageUrls_public` (array of public URLs formatted for direct download)
     - `allDescriptions`

12) **Add LangChain components:**
   - **OpenAI Chat Model** node (`lmChatOpenAi`)
     - Model: `gpt-4.1-mini`
     - Credentials: OpenAI API
   - **Structured Output Parser** node
     - Schema: `{ "image_prompt": "string" }`

13) **Add node: Agent** named “Generate Image Prompt” and connect **Aggregate descriptions → Generate Image Prompt**  
   - Set the agent “promptType” to define a custom prompt
   - User text should include `{{$json.allDescriptions}}`
   - Set the system message to the fixed fashion photoshoot template and constraints
   - Connect:
     - **OpenAI Chat Model → Generate Image Prompt** (AI language model connection)
     - **Structured Output Parser → Generate Image Prompt** (AI output parser connection)

14) **Add node: HTTP Request** named “NanoBanana: Create Image” and connect **Generate Image Prompt → NanoBanana: Create Image**  
   - Method: POST
   - URL: `https://api.atlascloud.ai/api/v1/model/generateImage`
   - Headers:
     - `Authorization: Bearer <YOUR_ATLASCLOUD_KEY>`
     - `Content-Type: application/json`
   - Body fields:
     - `model`: `google/nano-banana-pro/edit`
     - `aspect_ratio`: `1:1`
     - `resolution`: `2k`
     - `output_format`: `png`
     - `enable_sync_mode`: `false`
     - `enable_base64_output`: `false`
     - `images`: array from `{{$('Aggregate descriptions').item.json.imageUrls_public}}`
     - `prompt`: from `{{$json.output.image_prompt}}`

15) **Add node: Wait** and connect **NanoBanana: Create Image → Wait**  
   - Unit: minutes
   - Set a practical duration (e.g., 1–5 minutes depending on typical generation time).

16) **Add node: HTTP Request** named “Download Edited Image” and connect **Wait → Download Edited Image**  
   - Method: GET
   - URL: `https://api.atlascloud.ai/api/v1/model/prediction/{{$json.data.id}}`
   - Headers: same Bearer token
   - Response format: JSON

17) **Add node: HTTP Request** named “Download Final PNG (binary)” and connect **Download Edited Image → Download Final PNG (binary)**  
   - URL: `{{$json.data.outputs[0]}}`
   - Response format: File (binary)

18) **Add node: Google Drive** named “Upload file Nanobanana” and connect **Download Final PNG (binary) → Upload file Nanobanana**  
   - Credentials: Google Drive OAuth2
   - Operation: Upload
   - Ensure the node is set to upload the binary property produced by the download node (configure the binary field name accordingly, e.g. `data` or the actual property name n8n produced).
   - File name: `{{$json.data.id}}.png` (recommended)

19) **Add node: Google Sheets** named “Append row in sheet” and connect **Upload file Nanobanana → Append row in sheet**  
   - Credentials: Google Sheets OAuth2
   - Operation: Append
   - Set Spreadsheet ID and Sheet Tab
   - Map columns:
     - `status`: `nanobanana_done`
     - `image_1`: `{{$('Aggregate descriptions').item.json.image1Url}}`
     - `image_2`: `{{$('Aggregate descriptions').item.json.image2Url}}` (recommended correction)
     - `image_3`: `{{$('Aggregate descriptions').item.json.image3Url}}`
     - `description_all`: `{{$('Aggregate descriptions').item.json.allDescriptions}}`
     - `image_nanobanana`: use the Drive upload output link field available in your environment (often `webViewLink`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Documentation link provided in sticky note | https://automatisation.notion.site/Create-scalable-e-commerce-product-images-from-photos-using-NanoBanana-Pro-2e33d6550fd9808e8891f7d606b49df7?source=copy_link |
| Setup reminders from workflow notes: add Google Drive/Sheets OAuth, OpenAI key, AtlasCloud key; replace `<__PLACEHOLDER_VALUE__>`; ensure Drive folders writable; input limited to 3 images | From workflow sticky notes (“Setup steps”) |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.