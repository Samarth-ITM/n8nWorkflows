Generate UGC Marketing Videos for eCommerce with Sora 2 and Gemini

https://n8nworkflows.xyz/workflows/generate-ugc-marketing-videos-for-ecommerce-with-sora-2-and-gemini-9432


# Generate UGC Marketing Videos for eCommerce with Sora 2 and Gemini

### 1. Workflow Overview

This workflow automates the creation of authentic User-Generated Content (UGC) style marketing videos for eCommerce products. Starting from a simple product image and its name, it leverages AI models to analyze the product, generate an ideal influencer persona, create multiple detailed UGC video scripts, produce matching video content, and upload the final videos to Google Drive.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** User submits product image and name through a web form.
- **1.2 Product Analysis & Persona Creation:** Uses OpenAI Vision API to analyze the product image and generate a detailed influencer persona profile.
- **1.3 Script Generation:** Uses Google Gemini 2.5 Pro to generate multiple authentic UGC video scripts with detailed, frame-by-frame descriptions.
- **1.4 Script Extraction & Iteration:** Parses the multi-script output, splits it into individual scripts, and iterates over them.
- **1.5 Frame Generation & Image Processing:** For each script, generates a custom first frame adapted to UGC style and resizes images for video format.
- **1.6 Video Production & Monitoring:** Uses Sora 2 video generation API to create 12-second videos from scripts and frames, polls for video completion status.
- **1.7 Upload & Storage:** Uploads completed videos to a specified Google Drive folder automatically.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Accepts user input via a web form with product image upload and product name entry to kick off the workflow.
- **Nodes Involved:**  
  - `form_trigger`
  - `convert_product_to_base64`
  - `convert_product_to_image`

- **Node Details:**

  - **form_trigger**  
    - *Type:* Form Trigger (Webhook-based)  
    - *Purpose:* Entry point; collects user-uploaded product image (single file) and product name (required text).  
    - *Configuration:* Form titled "eCommerce Product Video Generator" with two required fields: Product (file), Product Name (text)  
    - *Input:* User submits form  
    - *Output:* Binary product image plus JSON with product name  
    - *Failure modes:* Form submission errors, invalid file types, missing required fields.  
    - *Notes:* Generates a webhook URL to access the form.

  - **convert_product_to_base64**  
    - *Type:* Extract From File  
    - *Purpose:* Converts binary product image to base64 string for API consumption.  
    - *Input:* Binary file from form_trigger (property "Product")  
    - *Output:* JSON property `data` with base64-encoded image data  
    - *Failure modes:* File read errors, unsupported formats.

  - **convert_product_to_image**  
    - *Type:* Convert To File  
    - *Purpose:* Converts base64 string back into a binary image format for downstream nodes that require an image file input.  
    - *Input:* Base64 string from previous node  
    - *Output:* Binary data representing the image  
    - *Failure modes:* Conversion failures if base64 data corrupted.

#### 2.2 Product Analysis & Persona Creation

- **Overview:** Uses OpenAI's Vision API to analyze the product image and generate a detailed influencer persona profile to tailor marketing scripts.
- **Nodes Involved:**  
  - `analyze_product`  
  - `set_model_details`

- **Node Details:**

  - **analyze_product**  
    - *Type:* OpenAI Vision API (via LangChain OpenAI node)  
    - *Purpose:* Analyzes the uploaded product image and generates a highly detailed profile of an ideal influencer persona for UGC ads.  
    - *Input:* Base64 product image (`resource: image`), product name from form_trigger JSON  
    - *Configuration:* Custom prompt instructing the model to produce a 5-part persona profile (identity, appearance, personality, lifestyle, and justification) with explicit output structure. Uses GPT-4o latest.  
    - *Output:* Text content with persona profile  
    - *Failure modes:* API quota exceeded, image analysis failures, malformed prompt results.

  - **set_model_details**  
    - *Type:* Set Node  
    - *Purpose:* Extracts and stores the persona text output into a variable (`prompt`) for use in script generation.  
    - *Input:* Output from `analyze_product` (persona text)  
    - *Output:* JSON with `prompt` property containing persona profile  
    - *Failure modes:* None significant; depends on prior node output.

#### 2.3 Script Generation

- **Overview:** Builds a master prompt combining the persona profile and product details, then calls Google Gemini 2.5 Pro to generate multiple authentic 12-second UGC video scripts with detailed shot breakdowns.
- **Nodes Involved:**  
  - `set_build_video_prompts`  
  - `generate_ad_prompts`  
  - `gemini-2.5-pro`  
  - `extract_prompts`  
  - `prompts-parser`  
  - `split_prompts`

- **Node Details:**

  - **set_build_video_prompts**  
    - *Type:* Set Node  
    - *Purpose:* Constructs the master prompt for Gemini with instructions on UGC style, video structure, and input substitution including the persona profile and product name.  
    - *Input:* Persona profile (`prompt`) from `set_model_details`, product name from form_trigger  
    - *Output:* JSON property `prompt` with full Gemini prompt text  
    - *Failure modes:* Incorrect prompt interpolation.

  - **generate_ad_prompts**  
    - *Type:* HTTP Request  
    - *Purpose:* Sends the master prompt and product image to Google Gemini 2.5 Pro for script generation.  
    - *Input:* JSON with prompt text and base64 product image data  
    - *Output:* Gemini-generated raw text containing multiple scripts  
    - *Failure modes:* API authentication errors, rate limits, timeout.

  - **gemini-2.5-pro**  
    - *Type:* LangChain Chat Model (Google Gemini)  
    - *Purpose:* Invokes the Gemini 2.5 Pro AI language model to generate content according to prompt.  
    - *Input:* Prompt from `generate_ad_prompts`  
    - *Output:* Multi-part text with scripts  
    - *Failure modes:* API quota, connectivity.

  - **extract_prompts**  
    - *Type:* Chain LLM (LangChain)  
    - *Purpose:* Extracts full individual scripts from the multi-script text output as an array of strings.  
    - *Input:* Raw Gemini output text  
    - *Output:* JSON object with array property `prompts` containing each full script  
    - *Failure modes:* Parsing errors if format changes.

  - **prompts-parser**  
    - *Type:* Output Parser Structured  
    - *Purpose:* Validates and structures the extracted prompts array using JSON schema.  
    - *Input:* Output from `extract_prompts`  
    - *Output:* Validated JSON array of string scripts  
    - *Failure modes:* Schema mismatch.

  - **split_prompts**  
    - *Type:* Split Out  
    - *Purpose:* Splits the array of prompts into individual JSON items for iteration.  
    - *Input:* JSON array of prompts from `prompts-parser`  
    - *Output:* Individual prompt JSON objects  
    - *Failure modes:* Empty arrays, splitting errors.

#### 2.4 Script Iteration & Frame Generation

- **Overview:** Iterates over each extracted script, generates a custom first frame image adapted to UGC aspect ratio and style using Gemini image generation, and prepares images for video generation.
- **Nodes Involved:**  
  - `iterate_prompts`  
  - `generate_frame`  
  - `set_frame_result`  
  - `get_frame_image`  
  - `resize_image`

- **Node Details:**

  - **iterate_prompts**  
    - *Type:* Split In Batches  
    - *Purpose:* Processes each script prompt one by one for video generation steps.  
    - *Input:* Individual prompt JSON from `split_prompts`  
    - *Output:* Single script JSON for downstream nodes  
    - *Failure modes:* Batch size configuration issues.

  - **generate_frame**  
    - *Type:* HTTP Request  
    - *Purpose:* Sends a prompt to Google Gemini image preview model to adapt the original product image into a first frame that fits UGC aspect ratio (720x1280) without distorting elements.  
    - *Input:* JSON body with base64 product image, detailed prompt describing image adaptation instructions  
    - *Output:* Gemini-generated image data (base64 PNG)  
    - *Failure modes:* API limits, image generation errors.

  - **set_frame_result**  
    - *Type:* Set Node  
    - *Purpose:* Extracts base64 image data from Gemini's response and stores in JSON property `result`.  
    - *Input:* Gemini response containing inline image data  
    - *Output:* JSON with `result` base64 image string  
    - *Failure modes:* Missing or malformed image data.

  - **get_frame_image**  
    - *Type:* Convert To File  
    - *Purpose:* Converts base64 image string to binary format for video API consumption.  
    - *Input:* Base64 `result` from `set_frame_result`  
    - *Output:* Binary image data  
    - *Failure modes:* Conversion failures.

  - **resize_image**  
    - *Type:* Edit Image  
    - *Purpose:* Resizes the frame image to 720x1280 pixels ignoring aspect ratio for consistent video format.  
    - *Input:* Binary image from `get_frame_image`  
    - *Output:* Resized binary image  
    - *Failure modes:* Image processing errors.

#### 2.5 Video Production & Monitoring

- **Overview:** Uses Sora 2 API to generate 12-second videos from each script and corresponding first frame, then polls for completion and retrieves the final video.
- **Nodes Involved:**  
  - `generate_video`  
  - `delay`  
  - `get_video_status`  
  - `check_status`  
  - `get_video`

- **Node Details:**

  - **generate_video**  
    - *Type:* HTTP Request  
    - *Purpose:* Calls OpenAI's Sora 2 video generation API to create a 12-second UGC video using the script prompt and first frame image as input.  
    - *Input:* Form data including the current script prompt, model "sora-2," video duration 12 seconds, resolution 720x1280, and binary image as input reference  
    - *Output:* JSON response with video generation job ID  
    - *Failure modes:* API quota limits, invalid inputs, network errors.

  - **delay**  
    - *Type:* Wait  
    - *Purpose:* Waits 15 seconds between polling attempts to avoid excessive API calls.  
    - *Input:* Triggered after video generation or status check  
    - *Output:* Passes flow to status check  
    - *Failure modes:* None.

  - **get_video_status**  
    - *Type:* HTTP Request  
    - *Purpose:* Polls Sora 2 API for the video generation job status using job ID.  
    - *Input:* Job ID from previous `generate_video` or repeated calls  
    - *Output:* JSON with current status (e.g., 'completed') and video ID if done  
    - *Failure modes:* API errors, job not found.

  - **check_status**  
    - *Type:* If Node  
    - *Purpose:* Checks if video generation status is "completed."  
    - *Input:* Status from `get_video_status`  
    - *Output:* If completed, proceeds to `get_video`; else loops back to `delay` to retry  
    - *Failure modes:* Incorrect status value, logic errors.

  - **get_video**  
    - *Type:* HTTP Request  
    - *Purpose:* Retrieves the completed video content from Sora 2 API using video ID.  
    - *Input:* Video ID from `get_video_status`  
    - *Output:* Video binary content or download URL  
    - *Failure modes:* Video not ready, API errors.

#### 2.6 Upload & Storage

- **Overview:** Uploads the generated video files to a Google Drive folder, organizing by run index.
- **Nodes Involved:**  
  - `upload_video`  
  - `iterate_prompts` (main output connection)

- **Node Details:**

  - **upload_video**  
    - *Type:* Google Drive Node  
    - *Purpose:* Uploads the generated video file to a specified folder in the user's Google Drive.  
    - *Input:* Video content from `get_video`  
    - *Configuration:* Folder ID set to a specific Google Drive folder URL, file named dynamically as "UGC Video #<runIndex + 1>"  
    - *Output:* Confirmation of upload  
    - *Failure modes:* OAuth token expiration, insufficient permissions, quota limits.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                         | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                              |
|-----------------------|----------------------------------|---------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------|
| form_trigger          | Form Trigger                     | Input reception of product image/name | -                            | convert_product_to_base64     | ## Overview & Setup ... (full sticky note text in node)                                                |
| convert_product_to_base64 | Extract From File                | Convert product image to base64       | form_trigger                 | convert_product_to_image      |                                                                                                        |
| convert_product_to_image | Convert To File                  | Convert base64 string to binary image | convert_product_to_base64    | analyze_product              |                                                                                                        |
| analyze_product       | OpenAI Vision API (LangChain)    | Analyze product image and create persona profile | convert_product_to_image   | set_model_details            |                                                                                                        |
| set_model_details     | Set Node                        | Store persona profile for next steps  | analyze_product             | set_build_video_prompts      |                                                                                                        |
| set_build_video_prompts | Set Node                        | Build master prompt combining persona and product | set_model_details          | generate_ad_prompts          |                                                                                                        |
| generate_ad_prompts   | HTTP Request                    | Request Gemini to generate UGC scripts | set_build_video_prompts     | gemini-2.5-pro              |                                                                                                        |
| gemini-2.5-pro        | LangChain Chat Model (Google Gemini) | Generate scripts from prompt           | generate_ad_prompts         | extract_prompts             |                                                                                                        |
| extract_prompts       | Chain LLM (LangChain)           | Extract individual scripts as array   | gemini-2.5-pro              | prompts-parser              |                                                                                                        |
| prompts-parser        | Output Parser Structured         | Validate and format extracted scripts | extract_prompts             | split_prompts               |                                                                                                        |
| split_prompts         | Split Out                      | Split array of scripts into individual items | prompts-parser             | iterate_prompts             |                                                                                                        |
| iterate_prompts       | Split In Batches               | Iterate over each script for video generation | split_prompts              | generate_frame (branch 2) & generate_video (branch 1) |                                                                                                        |
| generate_frame        | HTTP Request                    | Generate adapted first frame image    | iterate_prompts (batch 2)   | set_frame_result            |                                                                                                        |
| set_frame_result      | Set Node                        | Extract base64 image data from response | generate_frame             | get_frame_image             |                                                                                                        |
| get_frame_image       | Convert To File                  | Convert base64 image to binary         | set_frame_result            | resize_image                |                                                                                                        |
| resize_image          | Edit Image                      | Resize image to 720x1280 for video    | get_frame_image             | generate_video              |                                                                                                        |
| generate_video        | HTTP Request                    | Generate 12-second UGC video via Sora 2 | resize_image & iterate_prompts (batch 1) | delay                      |                                                                                                        |
| delay                 | Wait                           | Wait before polling video status      | generate_video              | get_video_status            |                                                                                                        |
| get_video_status      | HTTP Request                    | Poll video generation status          | delay                      | check_status                |                                                                                                        |
| check_status          | If                             | Check if video generation completed   | get_video_status            | get_video / delay           |                                                                                                        |
| get_video             | HTTP Request                    | Retrieve completed video content       | check_status               | upload_video                |                                                                                                        |
| upload_video          | Google Drive                   | Upload completed video to Google Drive | get_video                  | iterate_prompts (continue)  |                                                                                                        |
| Sticky Note           | Sticky Note                    | Provides workflow overview and details | -                          | -                          | ## Sora 2 UGC eCommerce Video Generator ... (full sticky note text)                                    |
| Sticky Note1          | Sticky Note                    | Explains overview, use cases, and how workflow works | -                          | -                          | ## Overview & Setup ... (full sticky note text)                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("form_trigger")**  
   - Type: Form Trigger (Webhook)  
   - Configure form with title "eCommerce Product Video Generator"  
   - Add two required fields:  
     - File upload named "Product" (single file)  
     - Text input named "Product Name"

2. **Add an Extract From File Node ("convert_product_to_base64")**  
   - Operation: binaryToProperty  
   - Binary Property Name: "Product"  
   - Connect from `form_trigger`

3. **Add a Convert To File Node ("convert_product_to_image")**  
   - Operation: toBinary  
   - Source Property: "data" (base64 string from previous node)  
   - Connect from `convert_product_to_base64`

4. **Add OpenAI Vision Analysis Node ("analyze_product")**  
   - Use LangChain OpenAI node or OpenAI Vision API  
   - Model: GPT-4o latest  
   - Input Type: base64 image from `convert_product_to_image`  
   - Prompt: Detailed persona generation prompt including product name interpolation from `form_trigger`  
   - Connect from `convert_product_to_image`  
   - Configure OpenAI credentials (API key with Vision API access)

5. **Add a Set Node ("set_model_details")**  
   - Assign `prompt` property to the content output from `analyze_product` (persona text)  
   - Connect from `analyze_product`

6. **Add a Set Node ("set_build_video_prompts")**  
   - Assign `prompt` with a master prompt template that instructs Gemini to create three detailed UGC video scripts combining:  
     - Persona profile from `set_model_details.prompt`  
     - Product name from `form_trigger`  
   - Connect from `set_model_details`

7. **Add HTTP Request Node to Generate Ad Prompts ("generate_ad_prompts")**  
   - URL: Google Gemini 2.5 Pro API endpoint for content generation  
   - Method: POST  
   - Body: JSON including `prompt` from `set_build_video_prompts` and base64 product image from `convert_product_to_base64`  
   - Authentication: HTTP Header Auth for Google Gemini API  
   - Connect from `set_build_video_prompts`  
   - Set retry and wait between tries for robustness

8. **Add LangChain Gemini Chat Node ("gemini-2.5-pro")**  
   - Model: Gemini 2.5 Pro  
   - Connect from `generate_ad_prompts`

9. **Add Chain LLM Node ("extract_prompts")**  
   - Purpose: Extract full individual scripts from Gemini response text as an array  
   - Input: Text from `gemini-2.5-pro`  
   - Connect from `gemini-2.5-pro`

10. **Add Output Parser Node ("prompts-parser")**  
    - Use JSON schema to validate array of prompts extracted  
    - Connect from `extract_prompts`

11. **Add Split Out Node ("split_prompts")**  
    - Field to split out: `output.prompts`  
    - Connect from `prompts-parser`

12. **Add Split In Batches Node ("iterate_prompts")**  
    - Configure to process each prompt one by one  
    - Connect from `split_prompts`

13. **Add HTTP Request Node to Generate Adapted Frame ("generate_frame")**  
    - URL: Google Gemini 2.5 Flash Image Preview API  
    - Method: POST  
    - Body: JSON with prompt instructing Gemini to adapt product image to 720x1280 UGC aspect ratio, including base64 product image  
    - Authentication: HTTP Header Auth for Google Gemini API  
    - Connect from branch 2 (second output) of `iterate_prompts`

14. **Add Set Node ("set_frame_result")**  
    - Extract base64 image data from `generate_frame` response to property `result`  
    - Connect from `generate_frame`

15. **Add Convert To File Node ("get_frame_image")**  
    - Convert base64 `result` string to binary image  
    - Connect from `set_frame_result`

16. **Add Edit Image Node ("resize_image")**  
    - Resize image to 720x1280 ignoring aspect ratio  
    - Connect from `get_frame_image`

17. **Add HTTP Request Node to Generate Video ("generate_video")**  
    - URL: OpenAI Sora 2 Video Generation Endpoint  
    - Method: POST, multipart form-data  
    - Body parameters:  
      - `prompt`: script text from current batch (`iterate_prompts.prompt`)  
      - `model`: "sora-2"  
      - `seconds`: "12"  
      - `size`: "720x1280"  
      - `input_reference`: binary resized image from `resize_image`  
    - Authentication: OpenAI API credentials with Sora 2 access  
    - Connect from branch 1 (first output) of `iterate_prompts`

18. **Add Wait Node ("delay")**  
    - Wait 15 seconds before polling status  
    - Connect from `generate_video`

19. **Add HTTP Request Node to Get Video Status ("get_video_status")**  
    - URL: `https://api.openai.com/v1/videos/{{ $json.id }}`  
    - Method: GET  
    - Authentication: OpenAI API  
    - Connect from `delay`

20. **Add If Node ("check_status")**  
    - Condition: `$json.status == "completed"`  
    - Connect from `get_video_status`  
    - True branch connects to `get_video`  
    - False branch connects back to `delay` (loop polling)

21. **Add HTTP Request Node to Get Video Content ("get_video")**  
    - URL: `https://api.openai.com/v1/videos/{{ $json.id }}/content`  
    - Method: GET  
    - Authentication: OpenAI API  
    - Connect from true branch of `check_status`

22. **Add Google Drive Upload Node ("upload_video")**  
    - Set file name dynamically: "UGC Video #{{ $runIndex + 1 }}"  
    - Set folder ID to target Google Drive folder URL  
    - Connect from `get_video`  
    - Configure Google Drive OAuth2 credentials

23. **Connect output of `upload_video` back to `iterate_prompts` to continue processing next script**

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|----------------|
| This workflow generates authentic UGC-style marketing videos from a single product image, leveraging OpenAI Vision API, Google Gemini 2.5 Pro, and Sora 2 video generation. | Sticky Note on nodes near `form_trigger` and overview note in workflow |
| Sora 2 video generation typically takes 2-3 minutes per 12-second video; costs approx $0.50-1.00 USD per video (check OpenAI pricing). | Sticky Note1 |
| Videos generated are 720x1280 (9:16) format optimized for social media platforms like TikTok and Instagram Reels. | Sticky Note1 |
| Google Drive is used for automatic storage and organization of videos by campaign. | Sticky Note1 |
| Customize the persona prompt in `analyze_product` to target different demographics or marketing angles. | Sticky Note1 |
| Adjust video duration, script style, or output folder via respective nodes for different use cases. | Sticky Note1 |
| Workflow design ensures natural, unpolished UGC aesthetics with detailed frame-by-frame video script breakdowns. | Sticky Note1 |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow using AI services. It complies fully with content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.