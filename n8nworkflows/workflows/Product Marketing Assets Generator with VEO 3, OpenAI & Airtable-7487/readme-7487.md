Product Marketing Assets Generator with VEO 3, OpenAI & Airtable

https://n8nworkflows.xyz/workflows/product-marketing-assets-generator-with-veo-3--openai---airtable-7487


# Product Marketing Assets Generator with VEO 3, OpenAI & Airtable

### 1. Workflow Overview

This workflow, titled **Product Marketing Assets Generator with VEO 3, OpenAI & Airtable**, automates the generation of marketing images and videos based on campaign concepts stored in Airtable, leveraging AI services such as OpenAI's GPT models and KIE AI's VEO 3 video and GPT-4o image generation APIs. It serves marketing teams aiming to produce creative visual assets (images and videos) quickly by transforming campaign briefs and uploaded product images into AI-generated marketing materials.

The workflow is logically divided into two primary branches based on the requested action:
- **Image Generation Flow**
- **Video Generation Flow**

Each branch has clearly defined processing stages:

- **1.1 Input Reception & Routing**: Receives webhook requests and routes to image or video generation based on the query parameter.
- **1.2 Data Retrieval from Airtable**: Fetches campaign concepts and instructions from Airtable.
- **1.3 Image Analysis & Preparation (Image Flow)**: Processes uploaded images to describe product details using OpenAI’s image analysis.
- **1.4 AI Prompt Generation**: Uses OpenAI GPT models to generate detailed prompts for image or video generation.
- **1.5 Asset Generation via KIE AI APIs**: Calls external APIs to generate images or videos based on prompts and input assets.
- **1.6 Result Handling & Airtable Update**: Polls for generation completion, updates Airtable records with results or errors.
- **1.7 Wait & Retry Logic**: Handles asynchronous processing by waiting and retrying data retrieval.

Supporting utilities like code nodes for URL formatting and batch processing of image assets are integrated to streamline data handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing

- **Overview:**  
  Receives incoming webhook requests with parameters defining whether to generate images or videos, then routes the flow accordingly.

- **Nodes Involved:**  
  - Webhook  
  - Switch

- **Node Details:**

  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point for external requests; listens on path `/test-button`.  
    - Configuration: No authentication, expects query parameters `action` and `recordId`.  
    - Inputs: HTTP request from external caller.  
    - Outputs: Forwards JSON query data to Switch node.  
    - Failures: Missing parameters, invalid HTTP methods, or network issues.  
    - Version: 2.1

  - **Switch**  
    - Type: Switch  
    - Role: Routes workflow based on `action` query parameter (`imageGeneration` or `videoGeneration`).  
    - Configuration: Strict string comparison on `$json.query.action`.  
    - Inputs: Webhook output.  
    - Outputs: Branch 1: Image generation flow; Branch 2: Video generation flow.  
    - Failures: Unexpected or missing `action` values cause no further processing.  
    - Version: 3.2

---

#### 2.2 Data Retrieval from Airtable

- **Overview:**  
  Fetch campaign concept and instructions for image or video generation from Airtable using the provided record ID.

- **Nodes Involved:**  
  - Get Concept Record ID - Image Generation  
  - Get Concept Record ID - Image Generation - 2  
  - Get a Concept Record ID - Video Generation

- **Node Details:**

  - **Get Concept Record ID - Image Generation**  
    - Type: Airtable  
    - Role: Retrieves campaign concept record by `recordId` for image generation branch.  
    - Configuration: Uses Airtable base `appH2OEGW78g0qSOw`, table `Concept Generation`.  
    - Inputs: Record ID from webhook query.  
    - Outputs: Campaign data for image flow.  
    - Failures: Invalid ID, Airtable API errors, authentication failures.  
    - Version: 2.1  
    - Credentials: Airtable API token named "Airtable Agenia Outbound".

  - **Get Concept Record ID - Image Generation - 2**  
    - Type: Airtable  
    - Role: Retrieves the same record again for additional details needed downstream (e.g., API key, image size).  
    - Configuration: Same as above, uses ID from first Airtable node output.  
    - Failures: Same as above.  
    - Version: 2.1

  - **Get a Concept Record ID - Video Generation**  
    - Type: Airtable  
    - Role: Retrieves campaign concept record by `recordId` for video generation branch.  
    - Configuration: Same Airtable base and table.  
    - Inputs: Record ID from webhook query.  
    - Outputs: Campaign data for video flow.  
    - Failures: Same as image flow.  
    - Version: 2.1

---

#### 2.3 Image Analysis & Preparation (Image Flow)

- **Overview:**  
  Splits uploaded asset files, analyzes each image with OpenAI’s vision model to extract detailed product descriptions, and aggregates results.

- **Nodes Involved:**  
  - Split Out Uploaded Asset  
  - Loop Over Image Asset  
  - Analyze image  
  - Aggregate Content Output

- **Node Details:**

  - **Split Out Uploaded Asset**  
    - Type: SplitOut  
    - Role: Extracts individual uploaded assets from Airtable record array.  
    - Configuration: Splits on field `Upload Assets`.  
    - Inputs: Airtable record JSON.  
    - Outputs: Single asset objects.  
    - Failures: Missing or malformed asset list.  
    - Version: 1

  - **Loop Over Image Asset**  
    - Type: SplitInBatches  
    - Role: Processes assets one by one to avoid API rate limits.  
    - Configuration: Default batch size, no concurrency controls.  
    - Inputs: Split Out node output.  
    - Outputs: Single asset per iteration.  
    - Failures: Large batch sizes may cause timeouts.  
    - Version: 3

  - **Analyze image**  
    - Type: OpenAI (Langchain) – Image Analysis  
    - Role: Uses GPT-4o model to analyze product image and generate a detailed description focusing on the product only.  
    - Configuration: Model `chatgpt-4o-latest`; prompt instructs to ignore background.  
    - Inputs: Image URL from asset.  
    - Outputs: Text description of product in image.  
    - Failures: API auth errors, rate limits, malformed URLs.  
    - Version: 1.8  
    - Credentials: OpenAI API

  - **Aggregate Content Output**  
    - Type: Aggregate  
    - Role: Aggregates all descriptions and URLs from analyzed images into arrays for downstream processing.  
    - Configuration: Aggregates fields `content` and `url`.  
    - Inputs: Loop iterations outputs.  
    - Outputs: Aggregated arrays.  
    - Failures: Empty input sets cause empty aggregation.  
    - Version: 1

---

#### 2.4 AI Prompt Generation

- **Overview:**  
  Generates detailed AI prompts for image or video generation using OpenAI GPT models, integrating user instructions, campaign data, and product descriptions.

- **Nodes Involved:**  
  - Generate System Prompt Images GPT-Image  
  - Generate System Prompt Video VEO3  
  - Think - GPT IMAG1 (tool node)  
  - Think - VEO3 (tool node)

- **Node Details:**

  - **Generate System Prompt Images GPT-Image**  
    - Type: OpenAI (Langchain)  
    - Role: Generates a structured, stringified JSON prompt for GPT-4o image generation model based on campaign concept, user instructions, and product descriptions.  
    - Configuration: Model `gpt-5`; system prompt details prompt engineering instructions; output is JSON string with key `image_prompt`.  
    - Inputs: Airtable data, aggregated product descriptions.  
    - Outputs: JSON with detailed image generation prompt.  
    - Failures: Model API errors, JSON parsing issues.  
    - Version: 1.8  
    - Credentials: OpenAI API

  - **Generate System Prompt Video VEO3**  
    - Type: OpenAI (Langchain)  
    - Role: Generates detailed cinematic video prompt as a stringified JSON for VEO3 model, based on user video instructions, campaign objectives, image descriptions, and aspect ratio.  
    - Configuration: Model `gpt-5-2025-08-07`; system prompt includes strict JSON output format; output key `Video_prompt`.  
    - Inputs: Airtable video data, formatted image URLs.  
    - Outputs: JSON with video generation prompt.  
    - Failures: Same as image prompt generation.  
    - Version: 1.8  
    - Credentials: OpenAI API

  - **Think - GPT IMAG1** and **Think - VEO3**  
    - Type: ToolThink (Langchain)  
    - Role: These nodes enable invoking the "Think" tool to validate or double check prompt generation outputs before sending to generation APIs.  
    - Inputs: Connected to respective prompt generation nodes.  
    - Outputs: Passed downstream.  
    - Failures: Tool unavailability or errors.  
    - Version: 1.1

---

#### 2.5 Asset Generation via KIE AI APIs

- **Overview:**  
  Calls external KIE AI APIs to generate images (GPT-4o) or videos (VEO3) using the generated prompts and user-uploaded asset URLs.

- **Nodes Involved:**  
  - Generate 4o Images  
  - Generate Videos VEO 3

- **Node Details:**

  - **Generate 4o Images**  
    - Type: HTTP Request  
    - Role: Sends POST request to KIE AI GPT-4o image generation endpoint with prompt, image URLs, requested sizes, and number of variants.  
    - Configuration: Authorization header uses API key from Airtable; body includes prompt and URLs formatted by prior code node.  
    - Inputs: Prompt JSON, formatted URLs, campaign parameters.  
    - Outputs: API response with taskId for polling.  
    - Failures: API auth failure, invalid URLs, or prompt format errors.  
    - Version: 4.2

  - **Generate Videos VEO 3**  
    - Type: HTTP Request  
    - Role: Sends POST request to KIE AI VEO 3 video generation endpoint with prompt, image URLs, model, aspect ratio, and watermark info.  
    - Configuration: Uses Bearer token from Airtable API key, JSON body with stringified prompt and URLs.  
    - Inputs: Prompt JSON, formatted URLs, campaign parameters.  
    - Outputs: API response with taskId.  
    - Failures: Similar auth or format issues as images.  
    - Version: 4.2

---

#### 2.6 Result Handling & Airtable Update

- **Overview:**  
  Polls the KIE AI API for completion of generation tasks, updates Airtable records with generated asset URLs or marks status as failed.

- **Nodes Involved:**  
  - Wait 180s  
  - Retrieve 4o Image  
  - If Success  
  - Create or update a record (Airtable)  
  - Update Image Generation Failed  
  - Wait 300s Videos  
  - Retrieve Video - 1080p - video  
  - If Success Video  
  - Update Video Assets Results  
  - Update Getting Failed  
  - Update Status Video Generation Failed

- **Node Details:**

  - **Wait 180s / Wait 300s Videos**  
    - Type: Wait  
    - Role: Pauses workflow for polling intervals (180s for images, 240s for videos) to allow generation to complete asynchronously.  
    - Inputs: Triggered after initial request success.  
    - Outputs: Triggers retrieval nodes.  
    - Failures: Timeout limits or external interruptions.  
    - Version: 1.1

  - **Retrieve 4o Image**  
    - Type: HTTP Request  
    - Role: Polls KIE AI image generation status endpoint with taskId; fetches result URLs if ready.  
    - Inputs: taskId from image generation node.  
    - Outputs: API response with status and URLs.  
    - Failures: Network errors, invalid taskId, auth errors.  
    - Version: 4.2

  - **If Success**  
    - Type: If  
    - Role: Checks if HTTP response code is 200 (success).  
    - Outputs: True branch proceeds to wait or update record, false branch updates failure status in Airtable.  
    - Failures: Incorrect response codes or missing fields.  
    - Version: 2.2

  - **Create or update a record**  
    - Type: Airtable  
    - Role: Updates Airtable record with new image generation status, URLs, prompts, and descriptions.  
    - Inputs: Output from successful retrieval of generated images.  
    - Failures: Airtable API errors, data validation errors.  
    - Version: 2.1

  - **Update Image Generation Failed**  
    - Type: Airtable  
    - Role: Marks the image generation as failed in the Airtable record if generation or retrieval fails.  
    - Inputs: Workflow failure branches.  
    - Failures: Airtable API issues.  
    - Version: 2.1

  - **Retrieve Video - 1080p - video**  
    - Type: HTTP Request  
    - Role: Polls KIE AI video generation status endpoint with taskId; fetches video URLs.  
    - Inputs: taskId from video generation node.  
    - Outputs: API response with video URLs.  
    - Failures: Similar to image retrieval.  
    - Version: 4.2

  - **If Success Video**  
    - Type: If  
    - Role: Checks for HTTP 200 success on video retrieval.  
    - Outputs: True branch updates Airtable with video URLs, false branch updates failure status.  
    - Failures: Similar to image If node.  
    - Version: 2.2

  - **Update Video Assets Results**  
    - Type: Airtable  
    - Role: Updates Airtable record with video generation success status and video URLs.  
    - Inputs: Successful video retrieval.  
    - Failures: Airtable API errors.  
    - Version: 2.1

  - **Update Getting Failed** & **Update Status Video Generation Failed**  
    - Type: Airtable  
    - Role: Mark video status as failed in Airtable when retrieval or generation fails.  
    - Inputs: Failure branches from If nodes.  
    - Failures: Airtable issues.  
    - Version: 2.1

---

#### 2.7 Utilities & Data Formatting

- **Overview:**  
  Helper nodes to format URLs and manage batching for API limits.

- **Nodes Involved:**  
  - Aggregate Image URLs  
  - Aggregate - Images URLs  
  - Code (URL Formatter)  
  - Code - Format URLs  
  - Split Out - Images Assets  
  - Limit 1 Image - API Limit

- **Node Details:**

  - **Aggregate Image URLs / Aggregate - Images URLs**  
    - Type: Aggregate  
    - Role: Collects URL arrays from image assets or generated assets for further processing.  
    - Failures: Empty inputs yield empty outputs.  
    - Version: 1

  - **Code / Code - Format URLs**  
    - Type: Code (JavaScript)  
    - Role: Splits comma-separated URL strings into arrays and formats them as quoted strings for JSON inclusion in API requests.  
    - Failures: Unexpected input formats, null or undefined fields.  
    - Version: 2

  - **Split Out - Images Assets / Split Out - Upload Assets**  
    - Type: SplitOut  
    - Role: Extract arrays of image or upload assets from Airtable records for batch processing.  
    - Failures: Missing or malformed arrays.  
    - Version: 1

  - **Limit 1 Image - API Limit**  
    - Type: Limit  
    - Role: Restricts processing to one image at a time to respect API limits.  
    - Failures: None expected.  
    - Version: 1

---

### 3. Summary Table

| Node Name                         | Node Type                | Functional Role                                      | Input Node(s)                           | Output Node(s)                               | Sticky Note                                                     |
|----------------------------------|--------------------------|-----------------------------------------------------|---------------------------------------|----------------------------------------------|----------------------------------------------------------------|
| Webhook                          | Webhook                  | Entry point, receives requests                       | -                                     | Switch                                       |                                                                |
| Switch                          | Switch                   | Routes based on action (image or video)              | Webhook                               | Get Concept Record ID - Image Generation; Get a Concept Record ID - Video Generation |                                                                |
| Get Concept Record ID - Image Generation | Airtable                 | Fetch campaign concept for image generation          | Switch (imageGeneration branch)       | Split Out Uploaded Asset                      |                                                                |
| Split Out Uploaded Asset          | SplitOut                 | Splits uploaded asset array                           | Get Concept Record ID - Image Generation | Loop Over Image Asset                         | "Image File Processing"                                         |
| Loop Over Image Asset             | SplitInBatches           | Processes each image asset individually              | Split Out Uploaded Asset               | Analyze image; Aggregate Content Output      | "Image File Processing"                                         |
| Analyze image                    | OpenAI (Langchain)       | Analyzes images to describe product                   | Loop Over Image Asset                  | Loop Over Image Asset                         | "Image Analyst LLM"                                             |
| Aggregate Content Output          | Aggregate                | Aggregates image descriptions and URLs               | Analyze image                         | Get Concept Record ID - Image Generation - 2 | "Image File Processing"                                         |
| Get Concept Record ID - Image Generation - 2 | Airtable                 | Fetch additional campaign details                     | Aggregate Content Output               | Split Out - Upload Assets                     |                                                                |
| Split Out - Upload Assets         | SplitOut                 | Splits upload assets for further processing          | Get Concept Record ID - Image Generation - 2 | Aggregate Image URLs                        | "Image File Processing"                                         |
| Aggregate Image URLs              | Aggregate                | Aggregates upload asset URLs                          | Split Out - Upload Assets              | Code                                         | "Image File Processing"                                         |
| Code                            | Code                     | Formats URLs as quoted strings for JSON              | Aggregate Image URLs                   | Generate System Prompt Images GPT-Image      |                                                                |
| Generate System Prompt Images GPT-Image | OpenAI (Langchain)       | Generates AI prompt JSON for image generation        | Code; Get Concept Record ID - Image Generation - 2 | Generate 4o Images                        | "Image Prompt Generator"                                        |
| Generate 4o Images               | HTTP Request             | Calls KIE AI API to generate images                   | Generate System Prompt Images GPT-Image | If Success                                  | "KIE AI Image"                                                 |
| If Success                      | If                       | Checks success of image generation API call           | Generate 4o Images                    | Wait 180s; Update Image Generation Failed    |                                                                |
| Wait 180s                      | Wait                     | Waits before polling image generation results         | If Success                          | Retrieve 4o Image                             |                                                                |
| Retrieve 4o Image               | HTTP Request             | Polls image generation status and retrieves results  | Wait 180s                           | Create or update a record                     | "KIE AI Image"                                                 |
| Create or update a record       | Airtable                 | Updates Airtable with generated image info           | Retrieve 4o Image                   | -                                            |                                                                |
| Update Image Generation Failed  | Airtable                 | Marks image generation as failed in Airtable         | If Success (false branch)            | -                                            |                                                                |
| Get a Concept Record ID - Video Generation | Airtable                 | Fetch campaign concept for video generation           | Switch (videoGeneration branch)      | Split Out - Images Assets                      |                                                                |
| Split Out - Images Assets        | SplitOut                 | Splits image assets array for video flow              | Get a Concept Record ID - Video Generation | Limit 1 Image - API Limit                   |                                                                |
| Limit 1 Image - API Limit        | Limit                    | Limits processing to one image                         | Split Out - Images Assets             | Aggregate - Images URLs                        |                                                                |
| Aggregate - Images URLs          | Aggregate                | Aggregates image URLs for video prompt generation     | Limit 1 Image - API Limit             | Code - Format URLs                            |                                                                |
| Code - Format URLs              | Code                     | Formats URLs as quoted strings for video prompt       | Aggregate - Images URLs               | Generate System Prompt Video VEO3             |                                                                |
| Generate System Prompt Video VEO3 | OpenAI (Langchain)       | Generates detailed video prompt JSON for VEO3         | Code - Format URLs; Get a Concept Record ID - Video Generation | Generate Videos VEO 3                   | "Video Prompt Generator"                                       |
| Generate Videos VEO 3           | HTTP Request             | Calls KIE AI API to generate videos                    | Generate System Prompt Video VEO3    | If success                                    | "KIE AI Video"                                                 |
| If success                     | If                       | Checks success of video generation API call            | Generate Videos VEO 3                | Wait 300s Videos; Update Status Video Generation Failed |                                                                |
| Wait 300s Videos               | Wait                     | Waits before polling video generation results          | If success                        | Retrieve Video - 1080p - video                 |                                                                |
| Retrieve Video - 1080p - video  | HTTP Request             | Polls video generation status and retrieves results   | Wait 300s Videos                    | If Success Video                              | "KIE AI Video"                                                 |
| If Success Video               | If                       | Checks success of video retrieval                       | Retrieve Video - 1080p - video       | Update Video Assets Results; Update Getting Failed |                                                                |
| Update Video Assets Results     | Airtable                 | Updates Airtable with generated video info            | If Success Video                   | -                                            |                                                                |
| Update Getting Failed           | Airtable                 | Marks video status as failed in Airtable               | If Success Video (false branch)    | -                                            |                                                                |
| Update Status Video Generation Failed | Airtable                 | Marks video generation as failed in Airtable           | If success (false branch)           | -                                            |                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Name: `Webhook`  
   - Path: `test-button`  
   - Accepts GET/POST requests with query parameters: `action` and `recordId`.

2. **Add a Switch node**  
   - Name: `Switch`  
   - Connect `Webhook` output to `Switch` input.  
   - Configure rules:  
     - If `$json.query.action` equals `imageGeneration` → output 1  
     - If `$json.query.action` equals `videoGeneration` → output 2

3. **Image Generation Branch Setup**

   a. **Get campaign concept record (image)**  
      - Node: Airtable  
      - Name: `Get Concept Record ID - Image Generation`  
      - Base: `appH2OEGW78g0qSOw`  
      - Table: `Concept Generation`  
      - Record ID: `={{ $json.query.recordId }}`  
      - Credentials: Airtable token named "Airtable Agenia Outbound"  
      - Connect Switch output 1 to this node.

   b. **Split uploaded assets array**  
      - Node: SplitOut  
      - Name: `Split Out Uploaded Asset`  
      - Field to split: `Upload Assets`  
      - Connect from `Get Concept Record ID - Image Generation`.

   c. **Loop over each image asset**  
      - Node: SplitInBatches  
      - Name: `Loop Over Image Asset`  
      - Connect from `Split Out Uploaded Asset`.

   d. **Analyze each image with OpenAI**  
      - Node: OpenAI (Langchain)  
      - Name: `Analyze image`  
      - Model: `chatgpt-4o-latest`  
      - Operation: `analyze` (image resource)  
      - Prompt: "Describe the product and brand in this image in full detail. Fully ignore the background. Focus ONLY on the product."  
      - Image URLs: `={{ $json.url }}`  
      - Credentials: OpenAI API account  
      - Connect from `Loop Over Image Asset`.

   e. **Aggregate content output**  
      - Node: Aggregate  
      - Name: `Aggregate Content Output`  
      - Fields to aggregate: `content`, `url`  
      - Connect from `Analyze image`.

   f. **Retrieve campaign concept record again**  
      - Node: Airtable  
      - Name: `Get Concept Record ID - Image Generation - 2`  
      - Use ID from first Airtable node's output: `={{ $('Get Concept Record ID - Image Generation').first().json.id }}`  
      - Connect from `Aggregate Content Output`.

   g. **Split again upload assets**  
      - Node: SplitOut  
      - Name: `Split Out - Upload Assets`  
      - Field to split: `Upload Assets`  
      - Connect from `Get Concept Record ID - Image Generation - 2`.

   h. **Aggregate image URLs**  
      - Node: Aggregate  
      - Name: `Aggregate Image URLs`  
      - Field to aggregate: `url`  
      - Connect from `Split Out - Upload Assets`.

   i. **Format URLs for API**  
      - Node: Code (JavaScript)  
      - Name: `Code`  
      - Code:  
        ```js
        const urlString = $json.url;
        const formattedUrls = urlString
          .flatMap(urlString => (typeof urlString === "string" ? urlString.split(',') : []))
          .map(url => `"${url.trim()}"`)
          .join(',\n');
        return { formattedUrls };
        ```  
      - Connect from `Aggregate Image URLs`.

   j. **Generate system prompt for images (OpenAI GPT-5)**  
      - Node: OpenAI (Langchain)  
      - Name: `Generate System Prompt Images GPT-Image`  
      - Model: `gpt-5`  
      - Messages: System prompt instructing to generate detailed image prompt JSON string; user prompt includes campaign concept, instructions, and aggregated product descriptions.  
      - Output: JSON with key `image_prompt`.  
      - Connect from `Code`.

   k. **Generate 4o images via KIE AI API**  
      - Node: HTTP Request  
      - Name: `Generate 4o Images`  
      - Method: POST  
      - URL: `https://api.kie.ai/api/v1/gpt4o-image/generate`  
      - Body: JSON with `filesUrl` (formatted URLs), `prompt` (image_prompt), `size`, and `nVariants` from Airtable record.  
      - Headers: Authorization Bearer token from Airtable API key field.  
      - Connect from `Generate System Prompt Images GPT-Image`.

   l. **Check if generation request succeeded**  
      - Node: If  
      - Name: `If Success`  
      - Condition: `$json.code === 200`  
      - Connect from `Generate 4o Images`.

   m. **On success, wait 180s before polling**  
      - Node: Wait  
      - Name: `Wait 180s`  
      - Amount: 180 seconds  
      - Connect from `If Success` true branch.

   n. **Poll for generated images**  
      - Node: HTTP Request  
      - Name: `Retrieve 4o Image`  
      - Method: GET  
      - URL: `https://api.kie.ai/api/v1/gpt4o-image/record-info` with query param `taskId` from generation response.  
      - Headers: Authorization Bearer token from Airtable API key.  
      - Connect from `Wait 180s`.

   o. **Update Airtable with generated images**  
      - Node: Airtable  
      - Name: `Create or update a record`  
      - Operation: Upsert by record ID  
      - Fields updated: Image status = Success, Images Assets URLs, Images Prompt, Images Description, Run = false.  
      - Connect from `Retrieve 4o Image`.

   p. **On failure, update Airtable status**  
      - Node: Airtable  
      - Name: `Update Image Generation Failed`  
      - Operation: Update record’s Image Status = Failed, Run = false.  
      - Connect from `If Success` false branch and failure branches.

4. **Video Generation Branch Setup**

   a. **Get campaign concept record (video)**  
      - Node: Airtable  
      - Name: `Get a Concept Record ID - Video Generation`  
      - Base and Table same as image branch  
      - Record ID: `={{ $json.query.recordId }}`  
      - Connect from Switch output 2.

   b. **Split image assets for video**  
      - Node: SplitOut  
      - Name: `Split Out - Images Assets`  
      - Field to split: `Images Assets`  
      - Connect from `Get a Concept Record ID - Video Generation`.

   c. **Limit to 1 image for API**  
      - Node: Limit  
      - Name: `Limit 1 Image - API Limit`  
      - Connect from `Split Out - Images Assets`.

   d. **Aggregate image URLs**  
      - Node: Aggregate  
      - Name: `Aggregate - Images URLs`  
      - Field to aggregate: `url`  
      - Connect from `Limit 1 Image - API Limit`.

   e. **Format URLs for JSON**  
      - Node: Code  
      - Name: `Code - Format URLs`  
      - Same JavaScript formatting as image flow.  
      - Connect from `Aggregate - Images URLs`.

   f. **Generate system prompt for video**  
      - Node: OpenAI (Langchain)  
      - Name: `Generate System Prompt Video VEO3`  
      - Model: `gpt-5-2025-08-07`  
      - System prompt instructs generating a stringified JSON for video generation.  
      - Input includes video instructions, campaign objectives, image descriptions, aspect ratio.  
      - Connect from `Code - Format URLs` and `Get a Concept Record ID - Video Generation`.

   g. **Generate video via KIE AI API**  
      - Node: HTTP Request  
      - Name: `Generate Videos VEO 3`  
      - Method: POST  
      - URL: `https://api.kie.ai/api/v1/veo/generate`  
      - Body: JSON with prompt (stringified), image URLs, model, watermark, aspect ratio.  
      - Headers: Authorization Bearer token from Airtable API key.  
      - Connect from `Generate System Prompt Video VEO3`.

   h. **Check if video generation request succeeded**  
      - Node: If  
      - Name: `If success`  
      - Condition: `$json.code === 200`  
      - Connect from `Generate Videos VEO 3`.

   i. **On success, wait 240s before polling**  
      - Node: Wait  
      - Name: `Wait 300s Videos` (actually 240s)  
      - Connect from `If success` true branch.

   j. **Poll for generated video**  
      - Node: HTTP Request  
      - Name: `Retrieve Video - 1080p - video`  
      - Method: GET  
      - URL: `https://api.kie.ai/api/v1/veo/record-info` with taskId query param.  
      - Headers: Authorization Bearer token.  
      - Connect from `Wait 300s Videos`.

   k. **Check if video retrieval succeeded**  
      - Node: If  
      - Name: `If Success Video`  
      - Condition: `$json.code === 200`  
      - Connect from `Retrieve Video - 1080p - video`.

   l. **Update Airtable with video results**  
      - Node: Airtable  
      - Name: `Update Video Assets Results`  
      - Update Video Status = success, Videos Assets array with URL.  
      - Connect from `If Success Video` true branch.

   m. **On failure, update Airtable video status**  
      - Node: Airtable  
      - Name: `Update Getting Failed` and `Update Status Video Generation Failed`  
      - Update Video Status = failed.  
      - Connect from `If Success Video` false branch and `If success` false branch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ![AI-powered-campaign-concept-generation-banner.png](https://i.postimg.cc/ZngnWjcH/AI-powered-campaign-concept-generation-banner.png)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Banner image linked in Sticky Note9                                                                |
| Airtable Base Link: https://airtable.com/appcbxXtgUQglcF9r/shrUXt5kkivWK7lKL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Setup instructions and community video available on Skool: https://www.skool.com/ruben-ai          |
| Sticky notes label key nodes and blocks for easier understanding, e.g., “Image Analyst LLM”, “Image Prompt Generator”, “KIE AI Image”, “Video Prompt Generator”, “KIE AI Video”, “Image File Processing” - these aid in visually grouping functional areas.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Visual guide for workflow grouping                                                                |

---

**Disclaimer:**  
This documentation is based exclusively on the provided n8n workflow JSON export. It respects all relevant content policies, handling only legal, public data with no inclusion of protected or offensive material.