Generating Vintage Polaroid Style Photos with Gemini AI

https://n8nworkflows.xyz/workflows/generating-vintage-polaroid-style-photos-with-gemini-ai-8887


# Generating Vintage Polaroid Style Photos with Gemini AI

### 1. Workflow Overview

This workflow automates the generation of vintage Polaroid-style photographs using Google's Gemini AI via the Defapi API. It targets photographers, content creators, and social media users who want to transform their digital photos into authentic vintage instant-camera-style images enriched with film grain, vignetting, and color cast effects.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Collect user inputs including two image uploads, a creative prompt, and an API key via a web form.
- **1.2 Image Preparation**: Convert uploaded binary images into base64-encoded data URLs and prepare the request payload.
- **1.3 Image Generation Request**: Submit the images and prompt to the Defapi API for AI-based image transformation.
- **1.4 Status Polling and Waiting**: Wait and poll the Defapi API until the image generation task completes.
- **1.5 Output Formatting**: Extract and format the final generated vintage photo URL for display or further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Handles the user-facing form submission capturing two image files, a text prompt describing the desired vintage effect, and the user's Defapi API key.
- **Nodes Involved:**  
  - Upload 2 Images (Form Trigger)
  - Sticky Note2 (Input Photo 1 illustration)  
  - Sticky Note5 (Input Photo 2 illustration)
  - Sticky Note (Example prompt guidance)

- **Node Details:**

  - **Upload 2 Images**  
    - Type: Form Trigger  
    - Role: Listens for HTTP form submissions containing two image files, a prompt string, and an API key.  
    - Configuration: Four form fields —  
      - "API Key" (required text)  
      - "Prompt" (required text)  
      - "Image 01" (required file upload, single image, accepts image MIME types)  
      - "Image 02" (required file upload, single image, accepts image MIME types)  
    - Input: User HTTP form submission  
    - Output: Binary data for images and JSON for text fields  
    - Edge cases: User may upload unsupported file types or omit required fields; form validation is assumed to handle this.  
    - Notes: Provides the web-accessible entry point for the workflow.

  - **Sticky Notes** (2, 5, and 1)  
    - Type: Sticky Note  
    - Role: Provide visual documentation and example images for user guidance and internal clarity.  
    - No input/output connections.  
    - Sticky Note1 explains the purpose of the "Convert to JSON" node.  
    - Sticky Note2 and Sticky Note5 show example photos to be used as inputs.  
    - Sticky Note (Example prompt) contains detailed prompt instructions to guide users in crafting prompts for vintage photo generation.

#### 2.2 Image Preparation

- **Overview:** Converts the two uploaded images from binary file format into base64-encoded data URLs and packages all inputs for the API request.
- **Nodes Involved:**  
  - Convert to JSON  
  - Sticky Note1 (documentation for this node)

- **Node Details:**

  - **Convert to JSON**  
    - Type: Code Node (JavaScript)  
    - Role: Reads the binary image data from the form submission, converts each image into a base64-encoded data URL string, and extracts the prompt and API key from JSON form fields.  
    - Configuration: Custom JavaScript code that uses n8n helpers to access binary data buffers and encodes them.  
    - Key Expressions: Uses `$input.first().binary['Image_01']` and `$input.first().binary['Image_02']` to access images, and `$input.first().json['API Key']` and `['Prompt']` for text fields.  
    - Input from: Upload 2 Images  
    - Output to: Send Image Generation Request to Defapi.org API  
    - Edge cases: Potential failures if binary data is missing, corrupted, or improperly named; base64 encoding errors; expression errors if field names differ.

#### 2.3 Image Generation Request

- **Overview:** Sends a POST request to the Defapi API to initiate the vintage photo generation task with the prepared images and prompt.
- **Nodes Involved:**  
  - Send Image Generation Request to Defapi.org API  
  - Sticky Note3 (Main workflow documentation)

- **Node Details:**

  - **Send Image Generation Request to Defapi.org API**  
    - Type: HTTP Request  
    - Role: Submits the generation job to Defapi API endpoint `https://api.defapi.org/api/image/gen`  
    - Configuration:  
      - Method: POST  
      - Headers: Content-Type: application/json; Authorization: Bearer token from API key  
      - Body: JSON containing:  
        - `prompt`: user prompt string  
        - `model`: "google/nano-banana" (Gemini AI model)  
        - `images`: array of two base64 data URLs from previous node  
    - Input from: Convert to JSON  
    - Output to: Wait for Image Processing Completion  
    - Edge cases: HTTP errors, authentication failures (invalid or missing API key), malformed JSON, API rate limits, network timeouts.

#### 2.4 Status Polling and Waiting

- **Overview:** Waits an initial 10 seconds, then polls the Defapi API to check if the image generation task has completed successfully.
- **Nodes Involved:**  
  - Wait for Image Processing Completion  
  - Obtain the generated status  
  - Check if Image Generation is Complete  

- **Node Details:**

  - **Wait for Image Processing Completion**  
    - Type: Wait  
    - Role: Pauses execution for 10 seconds to allow initial processing time.  
    - Configuration: Wait 10 seconds  
    - Input from: Send Image Generation Request to Defapi.org API or from IF node (retry path)  
    - Output to: Obtain the generated status  
    - Edge cases: Excessive waiting if API is slow, potential workflow timeout.

  - **Obtain the generated status**  
    - Type: HTTP Request  
    - Role: Sends GET request to `https://api.defapi.org/api/task/query` to obtain current status of the generation task.  
    - Configuration:  
      - Query Parameter: `task_id` obtained from previous generation response (`{{$json.data.task_id}}`)  
      - Headers: Content-Type: application/json; Authorization: Bearer token from API key (expression references previous node "Convert to JSON")  
      - Method: GET (default)  
    - Input from: Wait for Image Processing Completion  
    - Output to: Check if Image Generation is Complete  
    - Edge cases: HTTP errors, invalid or expired task_id, auth errors, API downtime.

  - **Check if Image Generation is Complete**  
    - Type: IF  
    - Role: Evaluates if the task status equals "success" (indicating completion).  
    - Configuration: Condition checks if `{{$json.data.status == 'success'}}` evaluates true.  
    - Input from: Obtain the generated status  
    - Outputs:  
      - True branch: proceed to format and display results  
      - False branch: loops back to Wait for Image Processing Completion to retry  
    - Edge cases: Infinite loop if status never changes to success; possible need for max retry limit (not implemented here).

#### 2.5 Output Formatting

- **Overview:** Extracts the URL of the generated vintage photo from the API response and formats it for display or downstream use.
- **Nodes Involved:**  
  - Format and Display Image Results  
  - Sticky Note4 (Result Image illustration)

- **Node Details:**

  - **Format and Display Image Results**  
    - Type: Set  
    - Role: Creates a new JSON field `image_url` containing the URL of the generated image extracted from API response (`{{$json.data.result[0].image}}`).  
    - Input from: Check if Image Generation is Complete (true branch)  
    - Output: Final output with the vintage photo URL  
    - Edge cases: Missing or malformed response data; no images generated; response array empty.

---

### 3. Summary Table

| Node Name                                 | Node Type           | Functional Role                                | Input Node(s)                   | Output Node(s)                        | Sticky Note                                                                                                           |
|-------------------------------------------|---------------------|-----------------------------------------------|--------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Upload 2 Images                           | Form Trigger        | Receive user inputs: 2 images, prompt, API key | -                              | Convert to JSON                     |                                                                                                                       |
| Sticky Note2                              | Sticky Note         | Illustrate Input Photo 1                       | -                              | -                                   | ## Input Photo 1 ![Product](https://i.imgur.com/s9KaIO2.png)                                                           |
| Sticky Note5                              | Sticky Note         | Illustrate Input Photo 2                       | -                              | -                                   | ## Input Photo 2 ![Product](https://i.imgur.com/t4GA0OJ.jpeg)                                                           |
| Sticky Note                               | Sticky Note         | Example prompt guidance                        | -                              | -                                   | ## Example prompt Take a picture with a Polaroid camera. The photo should look like a normal photo, without any clear subjects or props. such as a flash from a dark room, spread throughout the photo.In terms of color, it exhibits rich saturation and a vintage color cast, with soft tones, low contrast, and often accompanied by vignetting. The texture features a distinct film grain. Do not change the faces. Replace the background behind the two people with a white curtain. Make them being close to each other.The face should be clear. Their skin should be normal color. |
| Convert to JSON                          | Code                | Convert binary images to base64 data URLs; package inputs | Upload 2 Images                 | Send Image Generation Request to Defapi.org API | ## Convert to JSON Convert binary data of image to base64-style data url.                                              |
| Send Image Generation Request to Defapi.org API | HTTP Request        | Submit generation request to Defapi API       | Convert to JSON                 | Wait for Image Processing Completion | # Generate Vintage Polaroid Style Photo with Gemini AI [Workflow overview and instructions]                            |
| Wait for Image Processing Completion    | Wait                 | Pause 10 seconds before status check          | Send Image Generation Request / Check if Image Generation is Complete (false path) | Obtain the generated status           |                                                                                                                       |
| Obtain the generated status              | HTTP Request         | Poll Defapi API for generation status         | Wait for Image Processing Completion | Check if Image Generation is Complete |                                                                                                                       |
| Check if Image Generation is Complete   | IF                   | Evaluate if generation status is 'success'    | Obtain the generated status     | Format and Display Image Results (true), Wait for Image Processing Completion (false) |                                                                                                                       |
| Format and Display Image Results         | Set                  | Extract and set final image URL output         | Check if Image Generation is Complete (true) | -                                   | ## Result Image ![Creative](https://i.imgur.com/1jYGhjc.jpeg)                                                           |
| Sticky Note3                             | Sticky Note          | Main workflow documentation and instructions  | -                              | -                                   | # Generate Vintage Polaroid Style Photo with Gemini AI [Full workflow overview, setup, technical details, customization tips] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "Generate Vintage Polaroid Style Photo with Gemini AI".**

2. **Add a Form Trigger node named "Upload 2 Images":**  
   - Configure form fields:  
     - "API Key": Text, required  
     - "Prompt": Text, required  
     - "Image 01": File upload, accept `image/*`, single file, required  
     - "Image 02": File upload, accept `image/*`, single file, required  
   - This node will generate a webhook URL for form submission.

3. **Add a Code node named "Convert to JSON" and connect it to "Upload 2 Images":**  
   - Paste the following JavaScript code to convert the images to base64 data URLs and extract text inputs:

   ```javascript
   const results = {};

   const getImageDataUrl = async (name) => {
       const bin = $input.first().binary[name];
       const binBuffer = await this.helpers.getBinaryDataBuffer(0, name);
       return `data:${bin.mimeType};base64,${Buffer.from(binBuffer).toString('base64')}`
   }

   results.img_url_01 = await getImageDataUrl('Image_01')
   results.img_url_02 = await getImageDataUrl('Image_02')
   results.api_key = $input.first().json['API Key']
   results.prompt = $input.first().json['Prompt']

   return results;
   ```

4. **Add an HTTP Request node named "Send Image Generation Request to Defapi.org API", connected to "Convert to JSON":**  
   - Set method to POST  
   - URL: `https://api.defapi.org/api/image/gen`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: `Bearer {{$json.api_key}}`  
   - Body (JSON):  
   ```json
   {
     "prompt": "{{$json.prompt}}",
     "model": "google/nano-banana",
     "images": ["{{$json.img_url_01}}", "{{$json.img_url_02}}"]
   }
   ```  
   - Ensure "Send Body" is enabled and body format is JSON.

5. **Add a Wait node named "Wait for Image Processing Completion", connected to "Send Image Generation Request to Defapi.org API":**  
   - Set to wait 10 seconds.

6. **Add an HTTP Request node named "Obtain the generated status", connected to "Wait for Image Processing Completion":**  
   - Method: GET  
   - URL: `https://api.defapi.org/api/task/query`  
   - Add Query Parameter:  
     - Name: `task_id`  
     - Value: `={{$json.data.task_id}}` (reference task_id from prior response)  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: `Bearer {{ $('Convert to JSON').item.json.api_key }}` (reference API key from code node)

7. **Add an IF node named "Check if Image Generation is Complete", connected to "Obtain the generated status":**  
   - Condition: Check if `{{$json.data.status == 'success'}}` is true (string equals "success")  
   - True branch connected to next step  
   - False branch connected back to "Wait for Image Processing Completion" (to retry polling)

8. **Add a Set node named "Format and Display Image Results", connected to the true output of the IF node:**  
   - Add a new field named `image_url` of type string  
   - Set value to `{{$json.data.result[0].image}}` (extract first image URL from results)

9. **Optionally add sticky notes for documentation and user guidance as per original workflow for clarity and maintenance.**

10. **Save and activate the workflow.**

11. **Test the workflow:**  
    - Use the form URL from the form trigger node to submit two images, a prompt, and your Defapi API key.  
    - The workflow will process and eventually output a vintage Polaroid-style image URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sign up and obtain an API key at Defapi.org for access to the Gemini AI image generation model.               | https://defapi.org/model/google/gemini-2.5-flash-image                                           |
| The example prompt encourages specifying film grain, vintage color cast, low contrast, and vignetting effects. | Included in Sticky Note and main workflow description                                           |
| Use well-lit, clearly exposed input photos for best AI results; multiple generations may be needed.            | Workflow overview notes                                                                        |
| The workflow uses Bearer token authentication with the API key for all Defapi API calls.                       | Technical details section                                                                      |
| The workflow waits 10 seconds between polling attempts to avoid excessive API requests.                        | Technical details section                                                                      |
| Generated images may occasionally have artifacts; prompt tweaking and multiple attempts improve quality.       | Customization tips in workflow documentation                                                  |

---

**Disclaimer:** The content above is derived solely from an automated n8n workflow. It complies strictly with content policies and contains only legal, public data.