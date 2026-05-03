Generate 8-second product ad videos from Drive images with Gemini and Veo

https://n8nworkflows.xyz/workflows/generate-8-second-product-ad-videos-from-drive-images-with-gemini-and-veo-13920


# Generate 8-second product ad videos from Drive images with Gemini and Veo

# 1. Workflow Overview

This workflow generates an 8-second advertising video from a single product image stored in Google Drive. It analyzes the image with Gemini, converts that analysis into a structured short-form ad prompt, sends the prompt plus the source image to Google Veo as a long-running video generation request, polls until the video is ready, downloads the MP4, and uploads the final result back to Google Drive.

Typical use cases:
- Rapid creation of short product ad videos from static product shots
- AI-assisted ad concept generation for e-commerce or social media
- Automated creative production pipelines using Google Drive as input/output storage

## 1.1 Input Reception and Image Preparation

The workflow starts manually, downloads a selected image from Google Drive, and prepares two parallel forms of the same asset:
- binary image data for Gemini image analysis
- base64-encoded image data for the Veo API request

## 1.2 Image Understanding and Prompt Creation

Gemini first analyzes the uploaded product image and produces a structured visual brief. A second LLM chain then converts that brief into an advertising-oriented video prompt, with a structured output parser enforcing a JSON field called `video_prompt`.

## 1.3 Prompt/Image Merge and Video Generation Request

The workflow merges:
- the base64 image from the extraction branch
- the structured video prompt from the AI branch

It then sends both to the Veo long-running prediction endpoint with generation parameters such as aspect ratio, resolution, and duration.

## 1.4 Long-Running Job Polling

Because video generation is asynchronous, the workflow polls the operation URL returned by the Veo request. It waits 30 seconds between checks and loops until a generated video URI becomes available.

## 1.5 Video Download and Drive Upload

Once the video URI exists, the workflow downloads the generated MP4 and uploads it into a target Google Drive folder with a timestamped filename.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Image Preparation

### Overview
This block triggers the workflow, retrieves the source image from Google Drive, and prepares the file for downstream AI and API usage. It splits into two branches: one for image analysis and one for base64 encoding.

### Nodes Involved
- When clicking 'Test workflow'
- Download ad image
- Extract Model Image
- Section – Input image

### Node Details

#### 1) When clicking 'Test workflow'
- **Type and role:** `Manual Trigger`; starts the workflow manually from the editor.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Download ad image**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None beyond normal manual execution constraints.
- **Sub-workflow reference:** None.

#### 2) Download ad image
- **Type and role:** `Google Drive`; downloads the source product image.
- **Configuration choices:**
  - Operation: `download`
  - File selected explicitly by file ID
  - Downloaded binary stored under property `ad_img`
- **Key expressions or variables used:** None in expressions; file chosen from Drive picker.
- **Input and output connections:**
  - Input from **When clicking 'Test workflow'**
  - Outputs to **Extract Model Image** and **Creative Visualiser**
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failures:**
  - Missing or revoked Google Drive OAuth2 credentials
  - File no longer exists or access revoked
  - Wrong file type relative to downstream hardcoded MIME assumption (`image/png` later)
- **Sub-workflow reference:** None.

#### 3) Extract Model Image
- **Type and role:** `Extract From File`; converts binary image content into a property for reuse.
- **Configuration choices:**
  - Operation: binary to property
  - Source binary property: `ad_img`
  - Destination field: `ad_img_base64`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from **Download ad image**
  - Output to **Merge**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Missing binary property `ad_img`
  - Non-binary or corrupted binary input
  - Large images may increase payload size to Veo
- **Sub-workflow reference:** None.

#### 4) Section – Input image
- **Type and role:** `Sticky Note`; documentation only.
- **Configuration choices:** Comment: “Download the input image from Drive and convert it to base64.”
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Image Understanding and Prompt Creation

### Overview
This block uses Gemini to analyze the product image, then passes the resulting description into a second AI chain that turns the analysis into a concise advertising-oriented video script/prompt. The structured parser ensures the chain returns a predictable `video_prompt` field.

### Nodes Involved
- Creative Visualiser
- Google Gemini Chat Model1
- Product Video Prompt
- Structured Output Parser
- Section – Create video prompt
- Sample Promt

### Node Details

#### 1) Creative Visualiser
- **Type and role:** `Google Gemini` image analysis node; performs visual analysis on the source image.
- **Configuration choices:**
  - Resource: `image`
  - Operation: `analyze`
  - Input type: `binary`
  - Binary property: `ad_img`
  - Model: `models/gemini-2.5-flash`
  - Prompt instructs Gemini to return a “Visual Technical Brief” with strict sections:
    - ENTITY
    - VISUAL ATTRIBUTES
    - LIGHTING SETUP
    - BACKGROUND & VIBE
    - TECHNICAL SPEC
- **Key expressions or variables used:** Binary input `ad_img`.
- **Input and output connections:**
  - Input from **Download ad image**
  - Output to **Product Video Prompt**
- **Version-specific requirements:** Type version `1.1`; requires Gemini-compatible credentials in n8n.
- **Edge cases / failures:**
  - Gemini credential/auth issues
  - Unsupported or damaged image
  - Model output may still vary despite strict formatting instructions
  - If binary property name changes, node fails
- **Sub-workflow reference:** None.

#### 2) Google Gemini Chat Model1
- **Type and role:** `Google Gemini Chat Model`; acts as the LLM backend for the chain node.
- **Configuration choices:** Default options; no explicit model override shown here.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - AI language model connection into **Product Video Prompt**
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Credential issues
  - Model/provider rate limiting
  - Output inconsistency if model defaults change
- **Sub-workflow reference:** None.

#### 3) Product Video Prompt
- **Type and role:** `LangChain LLM Chain`; transforms the visual analysis into a short ad-oriented prompt.
- **Configuration choices:**
  - Prompt type: define
  - Input text: `Image description: {{ $json.content.parts[0].values() }}`
  - Uses a long system-style instruction asking the model to behave as a Creative Director and Copywriter
  - Requires English output
  - Constrains pacing for an 8-second ad
  - Asks for tone, visuals, dialogue, and audio/SFX
  - Has an output parser attached
- **Key expressions or variables used:**
  - `{{ $json.content.parts[0].values() }}`
- **Input and output connections:**
  - Main input from **Creative Visualiser**
  - AI language model input from **Google Gemini Chat Model1**
  - AI output parser input from **Structured Output Parser**
  - Main output to **Merge**
- **Version-specific requirements:** Type version `1.6`.
- **Edge cases / failures:**
  - Expression fragility: `content.parts[0].values()` assumes a specific Gemini output shape
  - Prompt text contains contradictory commentary: sticky note says “8s Vietnamese script” but node prompt says “Use natural English”
  - If parsed output does not match parser schema, chain may fail
- **Sub-workflow reference:** None.

#### 4) Structured Output Parser
- **Type and role:** `Structured Output Parser`; forces chain output into a schema.
- **Configuration choices:**
  - Manual schema
  - Requires JSON object with:
    - `video_prompt` string
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - AI output parser connection to **Product Video Prompt**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - The chain prompt asks for multi-section output, but the parser requires only `video_prompt`
  - If the model returns non-JSON or extraneous text, parsing can fail
- **Sub-workflow reference:** None.

#### 5) Section – Create video prompt
- **Type and role:** `Sticky Note`; explains the function of this block.
- **Configuration choices:** Comment says Gemini turns the image brief into an 8s Vietnamese script.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 6) Sample Promt
- **Type and role:** `Set`; disabled helper node containing sample prompts in pinned data.
- **Configuration choices:** Disabled; no effective runtime role.
- **Key expressions or variables used:** None in active config.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:** None, because disabled.
- **Sub-workflow reference:** None.

---

## 2.3 Prompt/Image Merge and Video Generation Request

### Overview
This block combines the AI-generated prompt and the base64-encoded image into one item, then sends them to the Veo long-running video generation endpoint.

### Nodes Involved
- Merge
- Generate Video
- Section – Generate video

### Node Details

#### 1) Merge
- **Type and role:** `Merge`; combines both branches into a single item.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `position`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input 1 from **Extract Model Image**
  - Input 2 from **Product Video Prompt**
  - Output to **Generate Video**
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases / failures:**
  - Position-based merge assumes both branches emit matching item counts in matching order
  - If one branch fails or returns no item, merge result may be empty or malformed
- **Sub-workflow reference:** None.

#### 2) Generate Video
- **Type and role:** `HTTP Request`; calls Veo long-running generation API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-fast-generate-preview:predictLongRunning`
  - Authentication: generic header auth
  - Header: `Content-Type: application/json`
  - JSON body includes:
    - `image.bytesBase64Encoded` from `{{ $json.ad_img_base64 }}`
    - `image.mimeType` hardcoded to `image/png`
    - `prompt` from `{{ JSON.stringify($json.output.video_prompt) }}`
    - parameters:
      - `aspectRatio`: `16:9`
      - `resolution`: `720p`
      - `durationSeconds`: `8`
      - `sampleCount`: `1`
- **Key expressions or variables used:**
  - `{{ $json.ad_img_base64 }}`
  - `{{ JSON.stringify($json.output.video_prompt) }}`
- **Input and output connections:**
  - Input from **Merge**
  - Output to **Get URL Download**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Invalid or expired API key in header auth credential
  - Request body mismatch with current Veo API requirements
  - Hardcoded `image/png` may be wrong if Drive file is JPEG/WebP
  - Large base64 payloads may hit API size limits
  - Preview model names may change or be region-restricted
- **Sub-workflow reference:** None.

#### 3) Section – Generate video
- **Type and role:** `Sticky Note`; explains the generation step.
- **Configuration choices:** Comment: “Start the Veo job using the image + prompt.”
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

## 2.4 Long-Running Job Polling

### Overview
This block repeatedly checks the Veo operation endpoint until the generated video URI becomes available. It uses a wait loop and an IF condition to branch between “ready” and “not ready yet”.

### Nodes Involved
- Get URL Download
- Wait
- If

### Node Details

#### 1) Get URL Download
- **Type and role:** `HTTP Request`; polls the operation resource returned by Veo.
- **Configuration choices:**
  - URL expression: `https://generativelanguage.googleapis.com/v1beta/{{ $json.name }}`
  - Method defaults to GET
  - Authentication: generic header auth
  - Header includes `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ $json.name }}`
- **Input and output connections:**
  - Input from **Generate Video**
  - Also input from **If** false branch to continue polling
  - Output to **Wait**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Assumes initial Veo response contains operation name in `name`
  - If later loop payload shape differs, URL expression may break
  - Auth failures or quota limits on repeated polling
  - Polling interval may be too short or too long for production usage
- **Sub-workflow reference:** None.

#### 2) Wait
- **Type and role:** `Wait`; pauses execution between polling attempts.
- **Configuration choices:**
  - Amount: `30`
  - Default unit in this node/version is typically seconds unless otherwise set in UI
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from **Get URL Download**
  - Output to **If**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Long-running executions may exceed environment-level limits
  - Wait node resumes through n8n’s internal wait/resume mechanism; misconfigured instance URLs or webhook settings can affect resumptions in some setups
- **Sub-workflow reference:** None.

#### 3) If
- **Type and role:** `If`; checks whether the generated video URI exists yet.
- **Configuration choices:**
  - Condition type: string `notEmpty`
  - Checked value: `{{ $json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
- **Key expressions or variables used:**
  - `{{ $json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
- **Input and output connections:**
  - Input from **Wait**
  - True output to **Get Dowload Video**
  - False output back to **Get URL Download**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - Expression may fail if intermediate objects are missing
  - If API response format changes, readiness check breaks
  - Infinite loop possible if job never completes and no timeout guard is added
- **Sub-workflow reference:** None.

---

## 2.5 Video Download and Drive Upload

### Overview
Once Veo reports a final asset URI, this block downloads the generated video and uploads it into Google Drive.

### Nodes Involved
- Get Dowload Video
- Upload to Drive
- Section – Download & upload

### Node Details

#### 1) Get Dowload Video
- **Type and role:** `HTTP Request`; downloads the generated video from the URI returned by Veo.
- **Configuration choices:**
  - URL expression: `{{ $json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
  - Authentication: generic header auth
- **Key expressions or variables used:**
  - `{{ $json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
- **Input and output connections:**
  - Input from **If** true branch
  - Output to **Upload to Drive**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Node name contains a typo (“Dowload”)
  - If response format is not set to file in node UI, upload step may not receive expected binary data
  - Download URL may expire
  - Auth may be required depending on endpoint behavior
- **Sub-workflow reference:** None.

#### 2) Upload to Drive
- **Type and role:** `Google Drive`; uploads the generated video to a Drive folder.
- **Configuration choices:**
  - Filename expression: `{{ 'ad_video_' + $now.toMillis() + '.mp4' }}`
  - Drive: `My Drive`
  - Destination folder explicitly selected
  - `onError`: continue regular output
- **Key expressions or variables used:**
  - `{{ 'ad_video_' + $now.toMillis() + '.mp4' }}`
- **Input and output connections:**
  - Input from **Get Dowload Video**
  - No downstream node
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failures:**
  - OAuth permission issues
  - Wrong binary property if HTTP download node does not expose file as expected
  - Folder access may be lost
  - Because `onError` is set to continue, upload failures may not stop the workflow and can be missed unless execution logs are inspected
- **Sub-workflow reference:** None.

#### 3) Section – Download & upload
- **Type and role:** `Sticky Note`; explains the finalization block.
- **Configuration choices:** Comment: “Poll until ready, download MP4, upload to Drive.”
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

## 2.6 Global Documentation Note

### Overview
This is a top-level documentation note describing the workflow’s purpose, setup steps, and tunable parameters.

### Nodes Involved
- Main overview

### Node Details

#### 1) Main overview
- **Type and role:** `Sticky Note`; project overview and setup instructions.
- **Configuration choices:** Contains functional summary and setup checklist:
  1. Connect Google Drive, Gemini, and Veo API credentials
  2. Set the input image in **Download ad image**
  3. Set output folder in **Upload to Drive**
  4. Optionally adjust `aspectRatio`, `resolution`, `durationSeconds` in **Generate Video**
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Test workflow' | Manual Trigger | Manual entry point for execution |  | Download ad image | ## AI Product Advertising Video<br>### How it works<br>This workflow generates an 8-second product advertising video from a single input image. It downloads the image from Google Drive, converts it to base64 for the API request, analyzes it with Gemini (Creative Visualiser), then turns the description into a short video script/prompt. The prompt + image are sent to Veo to start a long-running video generation job. The workflow polls until a video URI is available, downloads the MP4, and uploads it back to Google Drive.<br>### Setup<br>1) Connect credentials used in this workflow: Google Drive + Google Gemini, and an API key for the Veo HTTP requests.<br>2) Set the input image file in **Download ad image**.<br>3) Set the output folder in **Upload to Drive**.<br>4) (Optional) Adjust `aspectRatio`, `resolution`, and `durationSeconds` in **Generate Video**, then execute the workflow. |
| Download ad image | Google Drive | Download source image from Drive as binary | When clicking 'Test workflow' | Extract Model Image; Creative Visualiser | ## Input image<br>Download the input image from Drive and convert it to base64. |
| Extract Model Image | Extract From File | Convert binary image to base64 property | Download ad image | Merge | ## Input image<br>Download the input image from Drive and convert it to base64. |
| Creative Visualiser | Google Gemini | Analyze source image and produce visual brief | Download ad image | Product Video Prompt | ## Create video prompt<br>Gemini turns the image brief into an 8s Vietnamese script. |
| Google Gemini Chat Model1 | Google Gemini Chat Model | LLM backend for ad prompt chain |  | Product Video Prompt | ## Create video prompt<br>Gemini turns the image brief into an 8s Vietnamese script. |
| Product Video Prompt | LangChain LLM Chain | Convert visual brief into structured video prompt | Creative Visualiser | Merge | ## Create video prompt<br>Gemini turns the image brief into an 8s Vietnamese script. |
| Structured Output Parser | Structured Output Parser | Enforce schema with `video_prompt` string |  | Product Video Prompt | ## Create video prompt<br>Gemini turns the image brief into an 8s Vietnamese script. |
| Sample Promt | Set | Disabled sample prompt holder |  |  |  |
| Merge | Merge | Combine image base64 and AI prompt into one item | Extract Model Image; Product Video Prompt | Generate Video |  |
| Generate Video | HTTP Request | Start Veo long-running video generation job | Merge | Get URL Download | ## Generate video<br>Start the Veo job using the image + prompt. |
| Get URL Download | HTTP Request | Poll Veo operation status endpoint | Generate Video; If | Wait | ## Download & upload<br>Poll until ready, download MP4, upload to Drive. |
| Wait | Wait | Pause between polling attempts | Get URL Download | If | ## Download & upload<br>Poll until ready, download MP4, upload to Drive. |
| If | If | Check if generated video URI is available | Wait | Get Dowload Video; Get URL Download | ## Download & upload<br>Poll until ready, download MP4, upload to Drive. |
| Get Dowload Video | HTTP Request | Download generated MP4/video asset | If | Upload to Drive | ## Download & upload<br>Poll until ready, download MP4, upload to Drive. |
| Upload to Drive | Google Drive | Upload generated video file back to Drive | Get Dowload Video |  | ## Download & upload<br>Poll until ready, download MP4, upload to Drive. |
| Main overview | Sticky Note | Global documentation and setup instructions |  |  | ## AI Product Advertising Video<br>### How it works<br>This workflow generates an 8-second product advertising video from a single input image. It downloads the image from Google Drive, converts it to base64 for the API request, analyzes it with Gemini (Creative Visualiser), then turns the description into a short video script/prompt. The prompt + image are sent to Veo to start a long-running video generation job. The workflow polls until a video URI is available, downloads the MP4, and uploads it back to Google Drive.<br>### Setup<br>1) Connect credentials used in this workflow: Google Drive + Google Gemini, and an API key for the Veo HTTP requests.<br>2) Set the input image file in **Download ad image**.<br>3) Set the output folder in **Upload to Drive**.<br>4) (Optional) Adjust `aspectRatio`, `resolution`, and `durationSeconds` in **Generate Video**, then execute the workflow. |
| Section – Input image | Sticky Note | Visual documentation for input block |  |  | ## Input image<br>Download the input image from Drive and convert it to base64. |
| Section – Create video prompt | Sticky Note | Visual documentation for prompt creation block |  |  | ## Create video prompt<br>Gemini turns the image brief into an 8s Vietnamese script. |
| Section – Generate video | Sticky Note | Visual documentation for video generation block |  |  | ## Generate video<br>Start the Veo job using the image + prompt. |
| Section – Download & upload | Sticky Note | Visual documentation for polling and upload block |  |  | ## Download & upload<br>Poll until ready, download MP4, upload to Drive. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Use AI to generate advertising Video (EN)`.
   - Keep execution order at default unless you specifically need `v1`.
   - If your n8n instance handles binary data separately, ensure binary mode is compatible with file download/upload flows.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Name it: `When clicking 'Test workflow'`
   - No additional configuration needed.

3. **Add a Google Drive node to download the source image**
   - Node type: **Google Drive**
   - Name it: `Download ad image`
   - Operation: **Download**
   - Authenticate with **Google Drive OAuth2**
   - Select the source file ID from Drive
   - In options, set binary property name to `ad_img`
   - Connect:
     - `When clicking 'Test workflow'` → `Download ad image`

4. **Add an Extract From File node to create a base64 field**
   - Node type: **Extract From File**
   - Name it: `Extract Model Image`
   - Operation: **Binary to Property**
   - Source binary property: `ad_img`
   - Destination key: `ad_img_base64`
   - Connect:
     - `Download ad image` → `Extract Model Image`

5. **Add an image analysis Gemini node**
   - Node type: **Google Gemini**
   - Name it: `Creative Visualiser`
   - Resource: **Image**
   - Operation: **Analyze**
   - Input type: **Binary**
   - Binary property name: `ad_img`
   - Model: `models/gemini-2.5-flash`
   - Authenticate with **Google Gemini / PaLM API credentials**
   - Paste the visual analysis instruction prompt that asks for:
     - ENTITY
     - VISUAL ATTRIBUTES
     - LIGHTING SETUP
     - BACKGROUND & VIBE
     - TECHNICAL SPEC
   - Connect:
     - `Download ad image` → `Creative Visualiser`

6. **Add a Gemini chat model node for the LangChain chain**
   - Node type: **Google Gemini Chat Model**
   - Name it: `Google Gemini Chat Model1`
   - Authenticate with the same or another valid Gemini credential
   - Default model options are acceptable unless you want stricter determinism.

7. **Add a Structured Output Parser node**
   - Node type: **Structured Output Parser**
   - Name it: `Structured Output Parser`
   - Schema type: **Manual**
   - Use a schema requiring:
     - object
     - property `video_prompt` of type string
     - mark `video_prompt` as required

8. **Add a LangChain chain node for prompt generation**
   - Node type: **Basic LLM Chain / Chain LLM**
   - Name it: `Product Video Prompt`
   - Prompt type: **Define**
   - Set the input text to:
     - `Image description: {{ $json.content.parts[0].values() }}`
   - Add the long instruction message that defines the model as a Creative Director/Copywriter and asks for ad-style output.
   - Important: because the parser expects only `video_prompt`, adjust the prompt if needed so the model returns a single field matching the parser. The current exported workflow may be logically inconsistent here.
   - Enable output parser support.
   - Connect:
     - Main: `Creative Visualiser` → `Product Video Prompt`
     - AI language model: `Google Gemini Chat Model1` → `Product Video Prompt`
     - AI output parser: `Structured Output Parser` → `Product Video Prompt`

9. **Add a Merge node**
   - Node type: **Merge**
   - Name it: `Merge`
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - `Extract Model Image` → `Merge` input 1
     - `Product Video Prompt` → `Merge` input 2
   - This assumes both branches return one item in the same order.

10. **Add the Veo generation HTTP Request node**
    - Node type: **HTTP Request**
    - Name it: `Generate Video`
    - Method: **POST**
    - URL:
      - `https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-fast-generate-preview:predictLongRunning`
    - Authentication: **Generic Credential Type**
    - Auth type: **HTTP Header Auth**
    - Credential should include your Gemini/Veo API key, typically as `x-goog-api-key` or whatever your saved credential defines.
    - Enable **Send Headers**
    - Add header:
      - `Content-Type: application/json`
    - Body type: **JSON**
    - JSON body should include:
      - `instances[0].image.bytesBase64Encoded` = `{{$json.ad_img_base64}}`
      - `instances[0].image.mimeType` = `image/png`
      - `instances[0].prompt` = `{{ JSON.stringify($json.output.video_prompt) }}`
      - `parameters.aspectRatio` = `16:9`
      - `parameters.resolution` = `720p`
      - `parameters.durationSeconds` = `8`
      - `parameters.sampleCount` = `1`
    - Connect:
      - `Merge` → `Generate Video`

11. **Add the polling HTTP Request node**
    - Node type: **HTTP Request**
    - Name it: `Get URL Download`
    - Method: **GET**
    - URL expression:
      - `https://generativelanguage.googleapis.com/v1beta/{{ $json.name }}`
    - Use the same HTTP header auth credential as the generation request
    - Optionally add `Content-Type: application/json`
    - Connect:
      - `Generate Video` → `Get URL Download`

12. **Add a Wait node**
    - Node type: **Wait**
    - Name it: `Wait`
    - Set amount to `30`
    - Use seconds unless your UI indicates another unit
    - Connect:
      - `Get URL Download` → `Wait`

13. **Add an If node for readiness testing**
    - Node type: **If**
    - Name it: `If`
    - Condition:
      - Value 1: `{{ $json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
      - Operator: **is not empty**
    - Connect:
      - `Wait` → `If`

14. **Loop unfinished jobs back to polling**
    - Connect the **false** output of `If` back to `Get URL Download`
    - This creates the polling loop.

15. **Add the final video download HTTP Request**
    - Node type: **HTTP Request**
    - Name it: `Get Dowload Video`
    - Method: **GET**
    - URL expression:
      - `{{ $json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
    - Use the same header auth credential if required
    - Prefer configuring the response format as **File** or equivalent in your n8n version so the next Google Drive upload receives binary data properly
    - Connect:
      - **true** output of `If` → `Get Dowload Video`

16. **Add a Google Drive upload node**
    - Node type: **Google Drive**
    - Name it: `Upload to Drive`
    - Operation: **Upload**
    - Authenticate with **Google Drive OAuth2**
    - Choose destination Drive: `My Drive`
    - Select the destination folder
    - File name expression:
      - `{{ 'ad_video_' + $now.toMillis() + '.mp4' }}`
    - Ensure the node uses the binary property produced by `Get Dowload Video`
    - Set error handling to continue if you want behavior matching the export (`onError: continueRegularOutput`)
    - Connect:
      - `Get Dowload Video` → `Upload to Drive`

17. **Add documentation sticky notes**
    - Add a sticky note named `Main overview` with setup and behavior summary
    - Add `Section – Input image`
    - Add `Section – Create video prompt`
    - Add `Section – Generate video`
    - Add `Section – Download & upload`

18. **Optionally add the disabled sample prompt node**
    - Node type: **Set**
    - Name it: `Sample Promt`
    - Leave it disabled
    - Use it only as a local prompt bank if desired

19. **Configure credentials**
    - **Google Drive OAuth2**
      - Must have permission to read the source file and write into the destination folder
    - **Google Gemini / PaLM API**
      - Required for `Creative Visualiser` and `Google Gemini Chat Model1`
    - **HTTP Header Auth**
      - Required for Veo and Gemini REST calls
      - Typically stores the API key in a header expected by Google’s generative language endpoints

20. **Test the workflow**
    - Run manually
    - Verify:
      - source image downloads correctly
      - Gemini image analysis returns valid content
      - `Product Video Prompt` returns structured data matching parser schema
      - Veo request returns an operation object with `name`
      - polling eventually returns `generatedSamples[0].video.uri`
      - final download produces binary video data
      - upload succeeds to the target Drive folder

21. **Recommended hardening before production**
    - Detect the source image MIME type dynamically instead of hardcoding `image/png`
    - Add a max retry count or timeout in the polling loop
    - Align the LLM prompt with the parser schema so output is guaranteed
    - Add explicit error branches or notifications for upload and API failures
    - Log the Veo operation name and final URI for troubleshooting

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow description in the sticky note says the script is generated from a single image, analyzed by Gemini, then turned into a short video prompt for Veo. | Main overview |
| Setup instructions embedded in the workflow: connect Google Drive, Google Gemini, and an API key for Veo HTTP requests. | Main overview |
| Input file must be set directly in `Download ad image`. | Main overview |
| Output folder must be set directly in `Upload to Drive`. | Main overview |
| Optional generation parameters to tune: `aspectRatio`, `resolution`, `durationSeconds`. | Main overview |
| There is a wording inconsistency: one sticky note says Gemini creates an “8s Vietnamese script,” while the active chain prompt explicitly requests “natural English.” | Prompt generation block |
| The disabled `Sample Promt` node contains pinned example prompt text for product photography with human models, but it is not connected to execution. | Helper content only |

## Additional implementation cautions
- The workflow contains no sub-workflow nodes and no additional entry points beyond the manual trigger.
- The polling loop has no explicit stop condition other than successful video readiness.
- The final download/upload path may require response-format tuning in the HTTP node depending on your n8n version.
- The expression `{{$json.content.parts[0].values()}}` is fragile and should be validated against actual Gemini output in your environment.