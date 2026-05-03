Generate Creative ADV Images from References with Seedream v4 for Instagram & Facebook

https://n8nworkflows.xyz/workflows/generate-creative-adv-images-from-references-with-seedream-v4-for-instagram---facebook-8601


# Generate Creative ADV Images from References with Seedream v4 for Instagram & Facebook

### 1. Workflow Overview

This workflow automates the creation of creative advertising images (ADV) by leveraging multiple reference images and a textual prompt submitted via a web form. It uses the Seedream v4 AI model hosted on fal.ai to generate images based on the user’s input. After the image is generated, it automatically uploads the final creative to social media platforms Instagram and Facebook through the Upload-Post service.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Preparation:** Captures user inputs from a form, processes the reference images URLs into an array, and sets up data for the AI generation.
- **1.2 AI Image Generation Request:** Sends the prompt and reference images to the Seedream v4 AI API to generate the creative image.
- **1.3 Polling for Completion:** Waits and repeatedly checks the AI generation job status until the image is ready.
- **1.4 Post-Processing and Metadata Generation:** Retrieves the generated image, generates an SEO-friendly title using OpenAI GPT-4o-mini, and prepares the image for upload.
- **1.5 Upload to Social Media:** Uploads the generated image with the generated title to Instagram and Facebook via the Upload-Post API.
- **1.6 Storage:** Saves a copy of the generated image to Google Drive for archival.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

**Overview:**  
This block receives user inputs from a web form, parses a string of reference image URLs into an array, and sets up the prompt and reference images in a structured format for downstream processing.

**Nodes Involved:**  
- On form submission  
- Array  
- Set data

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; triggers the workflow on form submission with fields "Prompt" (text) and "Reference images" (textarea).  
  - Configuration: Form fields require user input; "Reference images" expects multiple URLs separated by commas or newlines.  
  - Input: User form data  
  - Output: JSON containing raw prompt and reference image string  
  - Edge cases: Missing or malformed URLs, empty strings, invalid prompt input  

- **Array**  
  - Type: Code  
  - Role: Parses the raw string of image URLs from the form into an array of strings.  
  - Configuration: JavaScript splits input by commas and newlines, trims empty entries.  
  - Key Expression: `const urlsArray = inputString.split(/,\s*|\r\n+/).filter(url => url.trim() !== "");`  
  - Input: Raw string from form submission  
  - Output: JSON with array of cleaned reference image URLs under property `reference_image_urls`  
  - Edge cases: Empty string inputs, invalid URL formats, trailing commas produce empty entries  

- **Set data**  
  - Type: Set  
  - Role: Prepares the data object for API consumption by assigning the prompt and reference images array to distinct JSON properties.  
  - Configuration: Sets `prompt` from form field and `reference_images` from the "Array" node output.  
  - Input: Output from "Array" node  
  - Output: JSON structured with `prompt` and `reference_images`  
  - Edge cases: Null or undefined values if prior parsing failed  

---

#### 2.2 AI Image Generation Request

**Overview:**  
This block sends a request to the Seedream v4 AI API to generate a new image based on the provided prompt and reference images.

**Nodes Involved:**  
- Create Video  
- Wait 60 sec.  

**Node Details:**

- **Create Video**  
  - Type: HTTP Request  
  - Role: Sends POST request to Seedream v4 endpoint to initiate image generation.  
  - Configuration:  
    - URL: `https://queue.fal.run/fal-ai/bytedance/seedream/v4/edit`  
    - Body: JSON including prompt, reference image URLs, image size (1280x1280), number of images (1), max images (1), safety checker enabled  
    - Headers: Content-Type application/json, Authorization with API key via HTTP header auth credential  
  - Key Expressions: Uses prompt and reference images from previous node via expressions  
  - Input: JSON with prompt and image URLs  
  - Output: JSON including a `request_id` for polling  
  - Edge cases: Authentication errors (invalid API key), network timeouts, malformed request body, API rate limits  

- **Wait 60 sec.**  
  - Type: Wait  
  - Role: Pauses workflow for 60 seconds to allow image generation to process on the server side.  
  - Configuration: Fixed 60-second wait  
  - Input: Triggered after "Create Video"  
  - Output: Passes execution to next node after delay  
  - Edge cases: Workflow timeout if external API takes longer than expected  

---

#### 2.3 Polling for Completion

**Overview:**  
This block polls the Seedream API for the generation status repeatedly until the image is marked as completed.

**Nodes Involved:**  
- Get status  
- Completed?  
- Wait 60 sec. (loop back)

**Node Details:**

- **Get status**  
  - Type: HTTP Request  
  - Role: Queries the status of the image generation job using the `request_id`.  
  - Configuration:  
    - URL dynamically constructed using `request_id` from "Create Video" output  
    - Authentication: Same HTTP header authorization as "Create Video"  
  - Input: `request_id` from "Create Video"  
  - Output: JSON containing status field (`COMPLETED`, `PENDING`, etc.)  
  - Edge cases: Network errors, invalid request_id, API errors  

- **Completed?**  
  - Type: If  
  - Role: Checks if the status is `COMPLETED`.  
  - Configuration: Condition matches `$json.status === "COMPLETED"`  
  - Input: Status response from "Get status"  
  - Output:  
    - True branch: proceeds to get result URL  
    - False branch: loops back to wait 60 sec. for re-polling  
  - Edge cases: Unexpected status values, empty or missing status field  

- **Wait 60 sec.** (loop)  
  - Same as previous wait node, used to pause before next polling iteration.  
  - Edge cases: Same as above  

---

#### 2.4 Post-Processing and Metadata Generation

**Overview:**  
Once the image is generated, this block retrieves the image URL, downloads the image file, and generates an SEO-optimized title for social media posting using OpenAI.

**Nodes Involved:**  
- Get Url Image  
- Generate title  
- Get File Image

**Node Details:**

- **Get Url Image**  
  - Type: HTTP Request  
  - Role: Retrieves detailed info of the completed image generation job, including the direct image URL.  
  - Configuration:  
    - URL constructed using `request_id` from prior nodes  
    - Auth: Same HTTP header authorization as previous fal.ai calls  
  - Input: `request_id`  
  - Output: JSON containing image `url` under `images.url`  
  - Edge cases: Missing URL, API errors, network failures  

- **Generate title**  
  - Type: OpenAI (LangChain)  
  - Role: Generates a catchy, SEO-friendly title for the image post based on the original prompt.  
  - Configuration:  
    - Model: GPT-4o-mini  
    - System prompt: Instructions for SEO optimized YouTube video titles, max 60 characters, language matching input  
    - User prompt: Uses the user’s original prompt text  
  - Input: Prompt text from "Set data" node  
  - Output: Generated title string in JSON path `message.content`  
  - Credentials: OpenAI API key with OpenRouter integration  
  - Edge cases: API quota exceeded, response formatting errors, network issues  

- **Get File Image**  
  - Type: HTTP Request  
  - Role: Downloads the image binary content from the URL obtained in "Get Url Image".  
  - Configuration: URL set from previous node’s `images.url` field  
  - Input: Image URL  
  - Output: Binary image data suitable for upload or storage  
  - Edge cases: URL unreachable, timeout, corrupted image data  

---

#### 2.5 Upload to Social Media

**Overview:**  
Uploads the generated image with the generated title to Instagram and Facebook via Upload-Post API.

**Nodes Involved:**  
- Upload Image  
- Post to Instagram  
- Post to Facebook

**Node Details:**

- **Upload Image**  
  - Type: Google Drive (OAuth2)  
  - Role: Saves a copy of the generated image binary to a specified Google Drive folder.  
  - Configuration:  
    - Drive: "My Drive"  
    - Folder: ID predefined (e.g., `1aHRwLWyrqfzoVC8HoB-YMrBvQ4tLC-NZ`)  
    - File name: Timestamp concatenated with original file name from image metadata  
  - Input: Binary image data from "Get File Image"  
  - Output: URL/location of the stored file  
  - Credentials: Google Drive OAuth2  
  - Edge cases: Auth expiration, quota limits, file naming conflicts  

- **Post to Instagram**  
  - Type: HTTP Request  
  - Role: Uploads image and title to Instagram via Upload-Post API.  
  - Configuration:  
    - URL: `https://api.upload-post.com/api/upload`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Body: title (generated title), user (placeholder YOUR_USERNAME), platform array with "instagram", image binary data under "data"  
    - Auth: API key via HTTP header auth  
  - Input: Title from "Generate title", binary image from "Get File Image"  
  - Output: Upload response JSON  
  - Edge cases: Invalid API keys, upload failures, rate limits  

- **Post to Facebook**  
  - Type: HTTP Request  
  - Role: Uploads same image and title to Facebook via Upload-Post API.  
  - Configuration: Same as Instagram node but platform array contains "facebook"  
  - Input/Output: Same as Instagram node  
  - Edge cases: Same as Instagram node  

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                          | Input Node(s)           | Output Node(s)                     | Sticky Note                                                                                                 |
|--------------------|-------------------------------|----------------------------------------|-------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------|
| Sticky Note3       | Sticky Note                   | Project overview and description       |                         |                                  | # Generate creative ADV image from your multiple image references ... with Seedream v4 AI                    |
| On form submission | Form Trigger                  | Entry point: Receive user prompt & refs|                         | Array                            |                                                                                                             |
| Array              | Code                         | Parse reference images URLs string to array | On form submission      | Set data                        |                                                                                                             |
| Set data           | Set                          | Prepare prompt and references for API  | Array                   | Create Video                    |                                                                                                             |
| Create Video       | HTTP Request                 | Request AI image generation (Seedream) | Set data                 | Wait 60 sec.                    | Set API Key created in Step 1                                                                                |
| Wait 60 sec.       | Wait                         | Pause 60 seconds before polling        | Create Video / Completed? | Get status / Completed? (loop)   |                                                                                                             |
| Get status         | HTTP Request                 | Poll AI generation status               | Wait 60 sec.             | Completed?                     |                                                                                                             |
| Completed?         | If                           | Check if generation completed           | Get status               | Get Url Image / Wait 60 sec.     |                                                                                                             |
| Get Url Image      | HTTP Request                 | Retrieve image URL of completed AI job  | Completed?               | Generate title                 |                                                                                                             |
| Generate title     | OpenAI (LangChain)            | Generate SEO-friendly title from prompt | Get Url Image            | Get File Image                 |                                                                                                             |
| Get File Image     | HTTP Request                 | Download generated image binary         | Generate title           | Upload Image, Post to Instagram, Post to Facebook |                                                                                                             |
| Upload Image       | Google Drive                 | Save image to Google Drive               | Get File Image           |                                  |                                                                                                             |
| Post to Instagram  | HTTP Request                 | Upload image & title to Instagram       | Get File Image, Generate title |                              | Set YOUR_USERNAME in Step 2                                                                                  |
| Post to Facebook   | HTTP Request                 | Upload image & title to Facebook        | Get File Image, Generate title |                              | Set YOUR_USERNAME in Step 2                                                                                  |
| Sticky Note5       | Sticky Note                   | Instruction about manual or scheduled start |                         |                                  | ## STEP 2 - MAIN FLOW: Start manually or schedule every 5 minutes                                            |
| Sticky Note6       | Sticky Note                   | Instructions for obtaining API key      |                         |                                  | ## STEP 1 - GET API KEY (YOURAPIKEY): Create fal.ai account, set header auth in "Create Image"                |
| Sticky Note7       | Sticky Note                   | Reminder to set API Key                  |                         |                                  | Set API Key created in Step 1                                                                                 |
| Sticky Note8       | Sticky Note                   | Instructions for Upload-Post API setup  |                         |                                  | ## STEP 2 - Upload video on Instagram and Facebook: API key, profiles, free plan limits                       |
| Sticky Note1       | Sticky Note                   | Example prompt and reference images with result |                         |                                  | Example on form submit: prompt and reference images URLs with resulting image preview                         |
| Sticky Note        | Sticky Note                   | Reminder to set your social media username |                         |                                  | Set YOUR_USERNAME in Step 2                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node "On form submission":**  
   - Trigger on form submissions with fields:  
     - Prompt (Required Text)  
     - Reference images (Required TextArea)  
   - Configure webhook ID (auto-generated).

2. **Create Code Node "Array":**  
   - Parse the "Reference images" string into an array by splitting on commas and newlines, removing empty entries:  
     ```js
     const inputString = $json["Reference images"];
     const urlsArray = inputString.split(/,\s*|\r\n+/).filter(url => url.trim() !== "");
     return [{ json: { reference_image_urls: urlsArray } }];
     ```
   - Connect "On form submission" → "Array".

3. **Create Set Node "Set data":**  
   - Assign `prompt` field with expression from form input prompt: `={{ $('On form submission').item.json.Prompt }}`  
   - Assign `reference_images` field as array from "Array" node: `={{ $json.reference_image_urls }}`  
   - Connect "Array" → "Set data".

4. **Create HTTP Request Node "Create Video":**  
   - Set method POST  
   - URL: `https://queue.fal.run/fal-ai/bytedance/seedream/v4/edit`  
   - Authentication: HTTP Header Auth with header "Authorization: Key YOURAPIKEY" (set via credentials)  
   - Headers: Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "prompt": "{{ $json.prompt }}",
       "image_urls": {{ $json.reference_images.toJsonString() }},
       "image_size": { "height": 1280, "width": 1280 },
       "num_images": 1,
       "max_images": 1,
       "enable_safety_checker": true
     }
     ```  
   - Connect "Set data" → "Create Video".

5. **Create Wait Node "Wait 60 sec.":**  
   - Wait 60 seconds  
   - Connect "Create Video" → "Wait 60 sec.".

6. **Create HTTP Request Node "Get status":**  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/bytedance/requests/{{ $('Create Video').item.json.request_id }}/status`  
   - Authentication: Same as "Create Video"  
   - Connect "Wait 60 sec." → "Get status".

7. **Create If Node "Completed?":**  
   - Condition: Check if `{{$json.status}} === "COMPLETED"`  
   - True output → "Get Url Image" node (next step)  
   - False output → Loop back to "Wait 60 sec." to poll again  
   - Connect "Get status" → "Completed?".

8. **Create HTTP Request Node "Get Url Image":**  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/bytedance/requests/{{ $json.request_id }}`  
   - Authentication: Same as above  
   - Connect True output of "Completed?" → "Get Url Image".

9. **Create OpenAI Node "Generate title":**  
   - Provider: OpenAI via LangChain  
   - Model: GPT-4o-mini  
   - Messages:  
     - System: SEO expert instructions for YouTube titles (max 60 chars, catchy, relevant keywords, same language)  
     - User: Input prompt from "Set data" node (`={{ $('Set data').item.json.prompt }}`)  
   - Credentials: OpenAI API key (OpenRouter recommended)  
   - Connect "Get Url Image" → "Generate title".

10. **Create HTTP Request Node "Get File Image":**  
    - Method: GET  
    - URL: `={{ $('Get Url Image').item.json.images.url }}`  
    - No authentication  
    - Connect "Generate title" → "Get File Image".

11. **Create Google Drive Node "Upload Image":**  
    - Operation: Upload file  
    - Folder ID: Set your desired Google Drive folder  
    - File name: Use timestamp + original file name: `={{ $now.format('yyyyLLddHHmmss') }}-{{ $('Get Url Image').item.json.video.file_name }}`  
    - Credentials: Google Drive OAuth2  
    - Connect "Get File Image" → "Upload Image".

12. **Create HTTP Request Node "Post to Instagram":**  
    - Method: POST  
    - URL: `https://api.upload-post.com/api/upload`  
    - Authentication: HTTP Header Auth with Upload-Post API key  
    - Content-Type: multipart/form-data  
    - Body parameters:  
      - title: `={{ $('Generate title').item.json.message.content }}`  
      - user: YOUR_USERNAME (replace placeholder)  
      - platform[]: instagram  
      - image: binary from "Get File Image" (`inputDataFieldName`: data)  
    - Connect "Get File Image" and "Generate title" outputs into this node (merge data).  

13. **Create HTTP Request Node "Post to Facebook":**  
    - Same as Instagram node but `platform[]` = facebook  
    - Connect same inputs as Instagram node.

14. **Add Sticky Notes:**  
    - Add descriptive sticky notes to explain API key setup, username placeholders, scheduling recommendations, and example inputs as per original workflow.

15. **Configure Execution Order and Testing:**  
    - Set the workflow to start on form submission  
    - Optionally add a Schedule Trigger node to run periodically (e.g., every 5 minutes) if needed  
    - Test with valid API keys and form inputs  

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Obtain API key for fal.ai Seedream v4 AI model at [https://fal.ai/](https://fal.ai/)                                                    | Step 1 API Key setup instructions (Sticky Note 6)                                               |
| Upload-Post service offers free 10 uploads/month; manage API keys and profiles at [https://www.upload-post.com/](https://www.upload-post.com/) | Social media upload service for Instagram & Facebook (Sticky Note 8)                            |
| OpenAI API is used to generate SEO-optimized titles; recommended model is GPT-4o-mini with OpenRouter integration                      | Title generation node configuration                                                             |
| Example prompt and reference images with result preview URL provided in sticky note                                                     | See Sticky Note1 for example usage with prompt and images                                       |
| Recommended scheduling interval for polling is 5 minutes to balance responsiveness and API rate limits                                 | Sticky Note5                                                                                     |
| Replace placeholder YOUR_USERNAME in social media upload nodes with your actual Upload-Post profile username                            | Sticky Note4 and Sticky Note                                                                     |

---

This comprehensive documentation covers every node, connection, configuration, and operational detail required to understand, recreate, and troubleshoot the "Generate creative ADV image from your multiple image references" workflow using n8n.