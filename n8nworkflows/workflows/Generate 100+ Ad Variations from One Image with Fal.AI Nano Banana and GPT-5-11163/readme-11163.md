Generate 100+ Ad Variations from One Image with Fal.AI Nano Banana and GPT-5

https://n8nworkflows.xyz/workflows/generate-100--ad-variations-from-one-image-with-fal-ai-nano-banana-and-gpt-5-11163


# Generate 100+ Ad Variations from One Image with Fal.AI Nano Banana and GPT-5

### 1. Workflow Overview

This workflow automates the generation of 100+ advertisement image variations starting from a single reference image, leveraging AI models and cloud integrations. It is designed for marketers and creative teams who want to scale their ad creative output efficiently using AI-powered image analysis, prompt rewriting, and image generation.

The workflow is logically divided into five main functional blocks:

- **1.1 Input Reception & Brand Settings:** Collects user inputs via a form, uploads the reference inspiration image to Google Drive, and downloads it for processing.
- **1.2 AI Analysis & Recipe Selection:** Uses OpenAI’s GPT-4o to analyze the reference image and describe it comprehensively. Then maps form inputs and generates multiple rewritten prompt variants using GPT-5, applying a chosen creative recipe.
- **1.3 Generation: Batch Processing with Fal.AI:** Splits the variants into batch items, submits each as a generation request to Fal.AI’s Nano Banana model, polls for completion, and retrieves generated images.
- **1.4 Delivery: Cloud Upload & Finalization:** Downloads generated images and uploads them into a dedicated Google Drive output folder.
- **1.5 Final Step Placeholder:** Provides a placeholder for any post-processing or additional steps after the batch processing loop completes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Brand Settings

**Overview:**  
This block triggers the workflow based on a user-submitted form capturing company name, ad inspiration image, number of ads, creative recipe, copywriting inspiration, and user comments. It uploads the inspiration image to Google Drive and downloads it for internal processing.

**Nodes Involved:**  
- On form submission  
- Upload file (Google Drive)  
- Download file to process (Google Drive)  
- Describe ad selected (OpenAI GPT-4o)  
- Extract from File (binary to base64 conversion)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing user inputs via a form with fields for company name, ad inspiration (file upload), number of ads, recipe selection, copywriting example, and comments.  
  - Inputs: User form fields  
  - Outputs: JSON with form data for downstream processing  
  - Edge cases: Missing or invalid form data; file upload failures; webhook issues.

- **Upload file**  
  - Type: Google Drive node (upload)  
  - Role: Uploads the user’s inspiration image file to a specified Google Drive folder ("Source files") for persistence and tracking.  
  - Config: Uses Google Drive OAuth2 credentials; folderId set to source files folder  
  - Inputs: File binary data from form submission  
  - Outputs: Metadata about uploaded file (e.g., file ID)  
  - Edge cases: Permissions errors, upload failures, incorrect folder IDs.

- **Download file to process**  
  - Type: Google Drive node (download)  
  - Role: Downloads the uploaded inspiration image file from Google Drive to process it in the workflow.  
  - Config: Uses file ID from previous node; credentials as above  
  - Outputs: Binary file data for image analysis and conversion  
  - Edge cases: File not found, permission denied, network errors.

- **Describe ad selected**  
  - Type: OpenAI node with image analysis (GPT-4o Mini)  
  - Role: Analyzes the downloaded image (base64 format) to produce an extremely comprehensive text description.  
  - Config: Model GPT-4o Mini, operation "analyze", inputType "base64", binary property "data"  
  - Inputs: Binary image data (base64)  
  - Outputs: JSON text description of the image  
  - Edge cases: API rate limits, malformed image data, model errors.

- **Extract from File**  
  - Type: Binary to property converter  
  - Role: Converts the binary image data to a base64 string property "image_base64" for downstream use in prompt generation and API calls.  
  - Inputs: Binary image data  
  - Outputs: Base64 string in JSON field  
  - Edge cases: Data corruption, conversion failures.

---

#### 2.2 AI Analysis & Recipe Selection

**Overview:**  
Maps form answers and AI-generated image description into workflow variables, then uses GPT-5 to generate multiple prompt variants for ad creatives according to a selected creative recipe and user instructions.

**Nodes Involved:**  
- Map form answers (Set)  
- Create variants of the prompt (OpenAI GPT-5)  
- Split Out variants (SplitOut)  
- Merge with image inspiration (Merge)

**Node Details:**

- **Map form answers**  
  - Type: Set node  
  - Role: Extracts and remaps form submission data and the GPT-4o image description into workflow variables used for prompt rewriting.  
  - Key variables set: company_name, number_ads, ad_creative_recipe, user_comments, image_description, copywriting inspiration  
  - Inputs: JSON from "Describe ad selected" and form submission nodes  
  - Outputs: Structured JSON with all parameters for prompt generation  
  - Edge cases: Missing keys or undefined variables causing expression errors.

- **Create variants of the prompt**  
  - Type: OpenAI GPT-5 text generation  
  - Role: Generates a JSON array of ad creative prompt variants (number defined by user) based on original image description, user instructions, creative recipe methodology, and copywriting inspiration.  
  - Config: GPT-5 model, system and user role messages guiding prompt rewriting with detailed instructions and examples for recipe methods ("The metaphor bake", "The pointer platter", "The comparison combo", or none).  
  - Outputs: JSON object with "variants" array of strings representing detailed image generation prompts  
  - Edge cases: API errors, malformed JSON output, generation timeouts.

- **Split Out variants**  
  - Type: SplitOut node  
  - Role: Splits the JSON array of variants into individual items for batch processing.  
  - Config: Splits on the field "output[0].content[0].text.variants"  
  - Outputs: Multiple items each containing one variant prompt  
  - Edge cases: Empty or malformed array causing downstream errors.

- **Merge with image inspiration**  
  - Type: Merge node (combine)  
  - Role: Combines each split variant with the original image base64 data and other metadata into a single item for subsequent image generation requests.  
  - Inputs: From Split Out variants and Extract from File nodes  
  - Outputs: Combined JSON items with prompt and image base64 string fields  
  - Edge cases: Data mismatches, out-of-sync inputs leading to missing data.

---

#### 2.3 Generation: Batch Processing with Fal.AI

**Overview:**  
Processes each ad prompt variant individually by submitting image generation requests to Fal.AI Nano Banana model via HTTP API, waits for completion, and retrieves generated images.

**Nodes Involved:**  
- Loop Over Items (splitInBatches)  
- Set variables for Ad generation (Set)  
- Submit Request to generate image (HTTP Request)  
- Wait 10 seconds (Wait)  
- Get Image status (HTTP Request)  
- Check if image is ready (If)  
- Get image url (HTTP Request)  
- Download in n8n (HTTP Request)

**Node Details:**

- **Loop Over Items**  
  - Type: splitInBatches  
  - Role: Controls the batch processing loop, handling one prompt variant at a time through the generation pipeline.  
  - Config: Default batch size (1) to process items sequentially  
  - Inputs: Combined prompt and image items from previous block  
  - Outputs: Single item per iteration for image generation request  
  - Edge cases: Infinite loops if batch not properly ended; batch size misconfiguration.

- **Set variables for Ad generation**  
  - Type: Set node  
  - Role: Prepares the request payload variables: "prompt" (the ad variant) and "image" (base64) for the generation API call.  
  - Outputs: JSON with "prompt" and "image" fields for HTTP request  
  - Edge cases: Null or empty prompt/image causing API errors.

- **Submit Request to generate image**  
  - Type: HTTP Request  
  - Role: Sends POST request to Fal.AI Nano Banana API to generate an image based on the prompt and base64 reference image.  
  - Config:  
    - URL: `https://queue.fal.run/fal-ai/nano-banana/edit`  
    - Method: POST  
    - Body: JSON with "prompt" and "image_urls" fields (image base64 embedded)  
    - Headers: Content-Type application/json, Authorization with Fal.AI API Key  
  - Outputs: JSON response including a "request_id" to poll status  
  - Edge cases: Auth failure, malformed body, API downtime.

- **Wait 10 seconds**  
  - Type: Wait node  
  - Role: Delays next step to allow image generation processing time before status polling.  
  - Config: 10 seconds delay  
  - Edge cases: Insufficient wait time causing premature status check.

- **Get Image status**  
  - Type: HTTP Request  
  - Role: Polls Fal.AI API for status of image generation request using request_id.  
  - Config:  
    - URL: `https://queue.fal.run/fal-ai/nano-banana/requests/{{request_id}}/status`  
    - Headers: Authorization with API Key  
  - Outputs: JSON with status field (e.g., "COMPLETED")  
  - Edge cases: Request ID expired, network errors.

- **Check if image is ready**  
  - Type: If node  
  - Role: Checks if Fal.AI status is "COMPLETED".  
  - Outputs:  
    - True branch: proceed to get image URL  
    - False branch: loop back to wait and poll again  
  - Edge cases: Stuck in loop if status never changes or error states not handled.

- **Get image url**  
  - Type: HTTP Request  
  - Role: Retrieves the URL of the generated image from Fal.AI once generation is completed.  
  - Config:  
    - URL: `https://queue.fal.run/fal-ai/nano-banana/requests/{{request_id}}`  
    - Headers: Authorization with API Key  
  - Outputs: JSON including image URL and file metadata  
  - Edge cases: Missing URL, permission denied.

- **Download in n8n**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file from the URL provided by Fal.AI.  
  - Config: URL dynamically set from previous node's output  
  - Outputs: Binary image file data for upload  
  - Edge cases: Download failures, timeouts.

---

#### 2.4 Delivery: Cloud Upload & Finalization

**Overview:**  
Uploads each downloaded generated ad image to a designated Google Drive output folder for storage and team access.

**Nodes Involved:**  
- Upload generated ad to output folder (Google Drive)  
- Loop Over Items (loop back to process next item)

**Node Details:**

- **Upload generated ad to output folder**  
  - Type: Google Drive node (upload)  
  - Role: Uploads the generated image binary to a specific Google Drive folder ("Output files").  
  - Config:  
    - File name derived by stripping file extension from downloaded file name  
    - Uses Google Drive OAuth2 credentials with write access  
    - Folder ID set to output folder  
  - Inputs: Binary image file from download node  
  - Outputs: Metadata of uploaded file  
  - Edge cases: Permission denied, quota exceeded, folder ID misconfiguration.

---

#### 2.5 Final Step Placeholder

**Overview:**  
A No Operation node named "Replace Me" is placed as a placeholder for future workflow extension after batch processing completes.

**Nodes Involved:**  
- Replace Me (NoOp)

**Node Details:**

- **Replace Me**  
  - Type: NoOp (no operation)  
  - Role: Placeholder for additional steps after all image variants are processed.  
  - Inputs/Outputs: Connected from batch loop but currently does nothing.  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                                      | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                                      |
|-------------------------------|--------------------------------|-----------------------------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                   | Workflow trigger; collects user inputs              | -                               | Upload file                       | ## Stage 1: Trigger: Input & Brand Settings                                                                     |
| Upload file                   | Google Drive Upload            | Uploads inspiration image to Drive source folder    | On form submission              | Download file to process          | See note on Google Drive connection setup                                                                       |
| Download file to process      | Google Drive Download          | Downloads uploaded inspiration image                | Upload file                    | Describe ad selected, Extract from File |                                                                                                                 |
| Describe ad selected          | OpenAI GPT-4o Mini (Image)    | Analyzes image to generate detailed description     | Download file to process        | Map form answers                 | ## Stage 2: AI Logic: Analysis & Recipe Selection                                                                |
| Extract from File             | Binary to Property Converter   | Converts binary image to base64 string               | Download file to process        | Merge with image inspiration     |                                                                                                                 |
| Map form answers             | Set                           | Maps form inputs + image description to variables   | Describe ad selected            | Create variants of the prompt     |                                                                                                                 |
| Create variants of the prompt | OpenAI GPT-5 Text Generation  | Generates multiple prompt variants using creative recipe | Map form answers              | Split Out variants               | ## Stage 3: Generation: Batch Processing with Fal.ai                                                             |
| Split Out variants            | SplitOut                      | Splits prompt variants array into individual items  | Create variants of the prompt   | Merge with image inspiration     |                                                                                                                 |
| Merge with image inspiration | Merge (Combine)               | Combines prompt variants with base64 image           | Split Out variants, Extract from File | Loop Over Items              |                                                                                                                 |
| Loop Over Items              | SplitInBatches                | Batch processing loop over prompt variants           | Merge with image inspiration    | Set variables for Ad generation, Replace Me |                                                                                                                 |
| Set variables for Ad generation | Set                         | Prepares prompt and image for generation API         | Loop Over Items                 | Submit Request to generate image |                                                                                                                 |
| Submit Request to generate image | HTTP Request               | Submits generation request to Fal.AI Nano Banana API | Set variables for Ad generation | Wait 10 seconds                |                                                                                                                 |
| Wait 10 seconds              | Wait                         | Pauses to allow image generation                      | Submit Request to generate image | Get Image status               | ## Stage 4.1: Get creative ad generation status                                                                   |
| Get Image status             | HTTP Request                 | Polls Fal.AI API for image generation status          | Wait 10 seconds                | Check if image is ready          |                                                                                                                 |
| Check if image is ready      | If                           | Checks if image generation is complete                | Get Image status               | Get image url (True), Wait 10 seconds (False) |                                                                                                                 |
| Get image url                | HTTP Request                 | Retrieves generated image URL from Fal.AI             | Check if image is ready (True) | Download in n8n                 |                                                                                                                 |
| Download in n8n              | HTTP Request                 | Downloads generated image file                         | Get image url                  | Upload generated ad to output folder | ## Stage 4.2: Download file and upload to Google Drive                                                            |
| Upload generated ad to output folder | Google Drive Upload     | Uploads generated image to Drive output folder        | Download in n8n                | Loop Over Items                 | ## Stage 4: Delivery: Drive Upload & Final Polish                                                                 |
| Replace Me                  | NoOp                         | Placeholder for post-processing steps                  | Loop Over Items                | -                             | ## Stage 5: final step. You can add other steps here when the loop is done                                         |
| Sticky Note1                | Sticky Note                  | Comment node                                          | -                             | -                             | ## Stage 4.1: Get creative ad generation status                                                                   |
| Sticky Note2                | Sticky Note                  | Project description, instructions, and cost estimates | -                             | -                             | Scale Your Creative Strategy: 100+ Ads from 1 Image, includes setup instructions and support contact              |
| Sticky Note3                | Sticky Note                  | Stage 2 header                                        | -                             | -                             | ## Stage 2: AI Logic: Analysis & Recipe Selection                                                                  |
| Sticky Note4                | Sticky Note                  | Stage 3 header                                        | -                             | -                             | ## Stage 3: Generation: Batch Processing with Fal.ai                                                               |
| Sticky Note5                | Sticky Note                  | Stage 4.2 header                                      | -                             | -                             | ## Stage 4.2: Download file and upload to Google Drive                                                             |
| Sticky Note6                | Sticky Note                  | Stage 5 header                                       | -                             | -                             | ## Stage 5: final step. You can add other steps here when the loop is done                                         |
| Sticky Note7                | Sticky Note                  | Stage 1 header                                       | -                             | -                             | ## Stage 1: Trigger: Input & Brand Settings                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure form fields:  
     - "What is your company name?" (string, required)  
     - "Can you provide an ad inspiration you have?" (file upload, single, required)  
     - "How many ads you want?" (number, required)  
     - "Which Ad creative recipe do you want?" (dropdown with options: The metaphor bake, The pointer platter, The comparison combo, None of them. let AI figure out what is best)  
     - "Provide an example of copy writting for inspiration" (string, required)  
     - "Can you give some description of what you want the ad to have?" (string, required)  
   - Save and note the webhook URL for triggering.

2. **Add Google Drive Upload node ("Upload file"):**  
   - Type: Google Drive  
   - Operation: Upload  
   - Credentials: Google Drive OAuth2 with read/write permissions  
   - Folder: Set `folderId` to your source files folder ID  
   - Input data field name: map to the file upload field from form trigger  
   - Connect "On form submission" → "Upload file"

3. **Add Google Drive Download node ("Download file to process"):**  
   - Type: Google Drive  
   - Operation: Download  
   - Use the file ID from "Upload file" output  
   - Connect "Upload file" → "Download file to process"

4. **Add OpenAI node ("Describe ad selected"):**  
   - Type: OpenAI (Langchain)  
   - Model: GPT-4o Mini (or GPT-4o)  
   - Operation: Analyze image  
   - Input type: base64 binary property "data" (from downloaded file)  
   - Text parameter: "describe the image extremely comprehensively. don't leave anything behind"  
   - Credentials: OpenAI API Key  
   - Connect "Download file to process" → "Describe ad selected"

5. **Add Extract from File node:**  
   - Type: Extract from File  
   - Operation: Binary to Property  
   - Destination key: "image_base64"  
   - Connect "Download file to process" → "Extract from File"

6. **Add Set node ("map form answers"):**  
   - Map variables using expressions from form submission and "Describe ad selected":  
     - company_name  
     - number_ads  
     - ad_creative_recipe  
     - user_comments  
     - image_description (from GPT-4o output)  
     - Provide an example of copy writting for inspiration  
   - Connect "Describe ad selected" → "map form answers"

7. **Add OpenAI node ("Create variants of the prompt"):**  
   - Model: GPT-5  
   - Configure system and user messages with detailed instructions for generating JSON array of variants based on creative recipe and inputs  
   - Output format: strict JSON with key "variants" (array)  
   - Credentials: OpenAI API Key  
   - Connect "map form answers" → "Create variants of the prompt"

8. **Add SplitOut node ("Split Out variants"):**  
   - Field to split: "output[0].content[0].text.variants"  
   - Connect "Create variants of the prompt" → "Split Out variants"

9. **Add Merge node ("Merge with image inspiration"):**  
   - Mode: Combine all  
   - Inputs: from "Split Out variants" and "Extract from File" (to combine prompt variant and base64 image)  
   - Connect both → "Merge with image inspiration"

10. **Add splitInBatches node ("Loop Over Items"):**  
    - Process items one by one (default batch size 1)  
    - Connect "Merge with image inspiration" → "Loop Over Items"

11. **Add Set node ("Set variables for Ad generation"):**  
    - Assign "prompt" from current item variant  
    - Assign "image" from base64 image string  
    - Connect "Loop Over Items" → "Set variables for Ad generation"

12. **Add HTTP Request node ("Submit Request to generate image"):**  
    - Method: POST  
    - URL: https://queue.fal.run/fal-ai/nano-banana/edit  
    - Headers: Content-Type application/json, Authorization: Key <YOUR_API_KEY> (replace with your Fal.AI API key)  
    - Body (JSON):  
      ```json
      {
        "prompt": "{{ $json.prompt }}",
        "image_urls": ["data:image/png;base64,{{ $json.image }}"]
      }
      ```  
    - Connect "Set variables for Ad generation" → "Submit Request to generate image"

13. **Add Wait node ("Wait 10 seconds"):**  
    - Delay: 10 seconds  
    - Connect "Submit Request to generate image" → "Wait 10 seconds"

14. **Add HTTP Request node ("Get Image status"):**  
    - Method: GET  
    - URL: https://queue.fal.run/fal-ai/nano-banana/requests/{{ $json.request_id }}/status  
    - Headers: Authorization: Key <YOUR_API_KEY>  
    - Connect "Wait 10 seconds" → "Get Image status"

15. **Add If node ("Check if image is ready"):**  
    - Condition: $json.status equals "COMPLETED"  
    - True → "Get image url" node  
    - False → loop back to "Wait 10 seconds" node for re-polling  
    - Connect "Get Image status" → "Check if image is ready"

16. **Add HTTP Request node ("Get image url"):**  
    - Method: GET  
    - URL: https://queue.fal.run/fal-ai/nano-banana/requests/{{ $json.request_id }}  
    - Headers: Authorization: Key <YOUR_API_KEY>  
    - Connect "Check if image is ready" (True) → "Get image url"

17. **Add HTTP Request node ("Download in n8n"):**  
    - Method: GET  
    - URL: {{ $json.images[0].url }} (dynamic from previous node)  
    - Connect "Get image url" → "Download in n8n"

18. **Add Google Drive Upload node ("Upload generated ad to output folder"):**  
    - Operation: Upload  
    - File name: derived from downloaded file name, removing extension  
    - Folder: your output folder ID  
    - Credentials: Google Drive OAuth2 with write access  
    - Connect "Download in n8n" → "Upload generated ad to output folder"

19. **Connect "Upload generated ad to output folder" → "Loop Over Items":**  
    - To continue batch processing next item until all variants processed

20. **Add NoOp node ("Replace Me"):**  
    - Connect as final node from "Loop Over Items" for any future workflow extensions after batch completes

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses Fal.AI Nano Banana model for AI image generation; sign up at https://fal.ai to get API keys and details.         | Fal.AI API documentation and signup                                                            |
| OpenAI usage requires API keys for GPT-4o Mini and GPT-5 models; configure in n8n credentials.                                      | OpenAI API key setup                                                                            |
| Google Drive credentials require OAuth2 with read/write access to specified folders for input inspiration images and output ads.  | Google Drive OAuth2 credential configuration                                                   |
| Estimated cost: ~$0.04 per generated image variant; ~$4 for 100 ads using Fal.AI Nano Banana.                                       | Cost estimation from Sticky Note2                                                              |
| Support contact: Reach out on X (Twitter) @maxrojasdelgado for troubleshooting and questions.                                       | https://x.com/maxrojasdelgado                                                                   |
| The creative recipe selection influences prompt rewriting style with examples provided in system prompt for GPT-5.                 | Creative recipes: The metaphor bake, The pointer platter, The comparison combo, or None          |
| The workflow processes image variants sequentially, polling Fal.AI API status with a 10-second delay to avoid API rate limiting.  | Loop control and polling mechanism                                                             |
| All generated images are uploaded to a dedicated Google Drive folder for easy access and sharing with teams.                       | Output folder management                                                                         |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow built with adherence to current content policies. It contains no illegal or protected content. All processed data is legal and public.