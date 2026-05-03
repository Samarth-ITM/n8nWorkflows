Transform Your Selfies into 3D Figurines with Nano Banana AI

https://n8nworkflows.xyz/workflows/transform-your-selfies-into-3d-figurines-with-nano-banana-ai-8571


# Transform Your Selfies into 3D Figurines with Nano Banana AI

### 1. Workflow Overview

This workflow, titled **"Transform Your Selfies into 3D Figurines with Nano Banana AI"**, automates the process of converting user-uploaded selfie images into high-quality 3D figurine designs using the Defapi API with Google's **Nano Banana AI** model. The workflow is designed for marketers, product designers, e-commerce businesses, and content creators who want to generate professional 3D figurine visualizations and collectible merchandise designs with minimal manual effort.

The workflow logic is split into the following functional blocks:

- **1.1 Input Reception**: User uploads a selfie image, provides a creative prompt describing the desired 3D figurine, and submits an API key via a form.
- **1.2 Data Preparation**: The uploaded image is converted into a base64-encoded data URI format, and user inputs are structured into a JSON object for API submission.
- **1.3 Image Generation Request**: The workflow sends a POST request to Defapi’s image generation endpoint with the prepared payload and API key.
- **1.4 Status Polling and Waiting**: After submitting the request, the workflow waits 10 seconds, then repeatedly polls the Defapi API to check if the image generation is complete.
- **1.5 Result Handling and Output Formatting**: Once the image generation is successful, the final image URL is extracted and formatted for display or download.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects user input through a web form including the selfie image file, an API key for Defapi authentication, and a creative prompt describing the desired output.

**Nodes Involved:**  
- Upload Image (Form Trigger)

**Node Details:**  
- **Upload Image**  
  - Type: Form Trigger  
  - Role: Collects image file (single image), API key (text), and creative prompt (text) from the user.  
  - Configuration:  
    - Form titled "Upload Image"  
    - Fields:  
      - Image: file upload, accepts image/*, required  
      - API Key: text input, required  
      - Prompt: text input, required  
  - Input: User-submitted form data via webhook  
  - Output: JSON data with binary image and form fields  
  - Potential Failures: Missing required fields, invalid image uploads, webhook errors, malformed input data  
  - Version Requirements: n8n Form Trigger node version 2.3+ for field configuration support

---

#### 2.2 Data Preparation

**Overview:**  
Converts the uploaded selfie image binary data into a base64-encoded string and constructs the JSON payload with the API key and prompt for subsequent API call.

**Nodes Involved:**  
- Convert to JSON (Code Node)

**Node Details:**  
- **Convert to JSON**  
  - Type: Code Node (JavaScript)  
  - Role: Converts binary image data to base64-encoded data URI; extracts and formats API key and prompt from input  
  - Configuration:  
    - Retrieves binary data buffer of the uploaded image  
    - Converts to base64 string with proper MIME type prefix (data:[mimeType];base64,[data])  
    - Extracts 'API Key' and 'Prompt' from form JSON fields  
    - Outputs an object containing `img_url` (base64 string), `api_key`, and `prompt`  
  - Input: Output from "Upload Image" node (binary image + JSON fields)  
  - Output: Structured JSON object for API submission  
  - Edge Cases: Missing or corrupted binary data, failed base64 encoding, missing API key or prompt fields  
  - Version Requirements: Code node with async/await support (n8n v0.157+ recommended)

---

#### 2.3 Image Generation Request

**Overview:**  
Sends a POST request to Defapi’s image generation API endpoint to initiate the 3D figurine creation process.

**Nodes Involved:**  
- Send Image Generation Request to Defapi.org API (HTTP Request)

**Node Details:**  
- **Send Image Generation Request to Defapi.org API**  
  - Type: HTTP Request  
  - Role: Submits the generation request including prompt, model selection, and base64 image to Defapi API  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.defapi.org/api/image/gen`  
    - Headers:  
      - Content-Type: application/json  
      - Authorization: Bearer token from user’s API key  
    - Body (JSON):  
      ```json
      {
        "prompt": "{{$json.prompt}}",
        "model": "google/nano-banana",
        "images": ["{{$json.img_url}}"]
      }
      ```  
  - Input: JSON from "Convert to JSON" node  
  - Output: JSON response containing task ID for status polling  
  - Edge Cases: Authentication failures (invalid API key), network timeouts, malformed JSON errors, API rate limits  
  - Version Requirements: HTTP Request node version 4.2+ for advanced header and query parameter support

---

#### 2.4 Status Polling and Waiting

**Overview:**  
This logic block waits for 10 seconds after the initial request, then repeatedly polls the API to check if the 3D image generation task has completed.

**Nodes Involved:**  
- Wait for Image Processing Completion (Wait Node)  
- Obtain the generated status (HTTP Request)  
- Check if Image Generation is Complete (IF Node)

**Node Details:**  
- **Wait for Image Processing Completion**  
  - Type: Wait Node  
  - Role: Delays workflow execution by 10 seconds before polling the status  
  - Configuration: Wait for 10 seconds  
  - Input: Output from "Send Image Generation Request" or from IF node for retry  
  - Output: Triggers "Obtain the generated status" node  
  - Edge Cases: Workflow timeouts if wait is too long; possible need to increase wait for slower responses  

- **Obtain the generated status**  
  - Type: HTTP Request  
  - Role: Sends GET request to Defapi API status endpoint to query task completion  
  - Configuration:  
    - URL: `https://api.defapi.org/api/task/query`  
    - Method: GET  
    - Query Parameter: task_id = `{{$json.data.task_id}}` (from generation request response)  
    - Headers:  
      - Content-Type: application/json  
      - Authorization: Bearer token from API key stored in "Convert to JSON" node  
  - Input: Output of "Wait for Image Processing Completion" node  
  - Output: JSON with status field (e.g., 'success', 'pending', 'failed')  
  - Edge Cases: Invalid or expired task_id, authentication errors, network failures  

- **Check if Image Generation is Complete**  
  - Type: IF Node  
  - Role: Checks if the status returned by "Obtain the generated status" is 'success'  
  - Configuration: Condition: `{{$json.data.status == 'success'}} == true`  
  - Input: Output of "Obtain the generated status" node  
  - Output:  
    - True branch: Proceed to formatting and displaying results  
    - False branch: Loop back to wait node to retry polling  
  - Edge Cases: Infinite loops if status never becomes 'success'; no handling for failure status explicitly  

---

#### 2.5 Result Handling and Output Formatting

**Overview:**  
Extracts the final generated 3D figurine image URL from the API response and formats it for display or download.

**Nodes Involved:**  
- Format and Display Image Results (Set Node)

**Node Details:**  
- **Format and Display Image Results**  
  - Type: Set Node  
  - Role: Maps the nested image URL from API response to a simple JSON property for easier consumption  
  - Configuration:  
    - Sets new field `image_url` with value `{{$json.data.result[0].image}}`  
  - Input: Output from IF node’s true branch (successful generation status)  
  - Output: JSON with `image_url` for downstream usage (e.g., display in UI or send via webhook)  
  - Edge Cases: Missing or empty result array, malformed API response  

---

### 3. Summary Table

| Node Name                             | Node Type          | Functional Role                            | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                                                   |
|-------------------------------------|--------------------|-------------------------------------------|----------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Upload Image                        | Form Trigger       | Receives user image, API key, and prompt | Webhook / User input             | Convert to JSON                     |                                                                                                                              |
| Convert to JSON                    | Code Node          | Converts image to base64 and formats data | Upload Image                    | Send Image Generation Request to Defapi.org API |                                                                                                                              |
| Send Image Generation Request to Defapi.org API | HTTP Request      | Sends generation request to Defapi API    | Convert to JSON                 | Wait for Image Processing Completion |                                                                                                                              |
| Wait for Image Processing Completion | Wait Node          | Waits 10 seconds between polling attempts | Send Image Generation Request to Defapi.org API, Check if Image Generation is Complete (False branch) | Obtain the generated status          |                                                                                                                              |
| Obtain the generated status        | HTTP Request       | Polls Defapi API for task status           | Wait for Image Processing Completion | Check if Image Generation is Complete |                                                                                                                              |
| Check if Image Generation is Complete | IF Node            | Checks if generation status is 'success'  | Obtain the generated status     | Format and Display Image Results (True), Wait for Image Processing Completion (False) |                                                                                                                              |
| Format and Display Image Results   | Set Node           | Formats final image URL for output          | Check if Image Generation is Complete (True) | (end of workflow)                   |                                                                                                                              |
| Sticky Note3                      | Sticky Note        | Workflow overview and detailed description | None                           | None                               | Contains full workflow overview, setup instructions, usage tips, and example prompt                                          |
| Sticky Note2                      | Sticky Note        | Displays example input photo                | None                           | None                               | Shows sample product image input                                                                                            |
| Sticky Note4                      | Sticky Note        | Displays example output image               | None                           | None                               | Shows sample creative 3D figurine output image                                                                              |
| Sticky Note                       | Sticky Note        | Provides detailed example prompt            | None                           | None                               | Contains example prompt text for creative input                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Upload Image" node**  
   - Type: Form Trigger  
   - Configure form titled "Upload Image"  
   - Add form fields:  
     - Image: file upload, accepts image/*, required  
     - API Key: text field, required  
     - Prompt: text field, required  
   - Save and note the webhook URL to access the form  
   
2. **Add "Convert to JSON" node**  
   - Type: Code Node (JavaScript)  
   - Paste the following JS code to convert uploaded image to base64 data URI and extract API key and prompt:
     ```javascript
     const results = {};
     const bin = $input.first().binary['Image'];
     const binBuffer = await this.helpers.getBinaryDataBuffer(0, 'Image');
     results.img_url = `data:${bin.mimeType};base64,${Buffer.from(binBuffer).toString('base64')}`;
     results.api_key = $input.first().json['API Key'];
     results.prompt = $input.first().json['Prompt'];
     return results;
     ```
   - Connect "Upload Image" node output to this node.

3. **Add "Send Image Generation Request to Defapi.org API" node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.defapi.org/api/image/gen`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: Bearer token from `{{ $json.api_key }}`  
   - Body (JSON):  
     ```json
     {
       "prompt": "{{$json.prompt}}",
       "model": "google/nano-banana",
       "images": ["{{$json.img_url}}"]
     }
     ```
   - Connect "Convert to JSON" node output to this node.

4. **Add "Wait for Image Processing Completion" node**  
   - Type: Wait  
   - Set wait time to 10 seconds  
   - Connect output of "Send Image Generation Request" node to this node.

5. **Add "Obtain the generated status" node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.defapi.org/api/task/query`  
   - Query Parameter: `task_id` = `{{$json.data.task_id}}` (extracted from previous response)  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: Bearer token from `{{ $('Convert to JSON').item.json.api_key }}` (or pass API key forward)  
   - Connect "Wait for Image Processing Completion" node output to this node.

6. **Add "Check if Image Generation is Complete" node**  
   - Type: IF  
   - Condition: Check if `{{$json.data.status}}` equals `'success'`  
   - Connect "Obtain the generated status" node output to this node.

7. **Add "Format and Display Image Results" node**  
   - Type: Set  
   - Add field `image_url` with value `{{$json.data.result[0].image}}`  
   - Connect "Check if Image Generation is Complete" node’s **true** output to this node.

8. **Loop for polling**  
   - Connect "Check if Image Generation is Complete" node’s **false** output back to the "Wait for Image Processing Completion" node to continue polling until completion.

9. **Save and activate the workflow**  
   - Test by accessing the form URL, uploading a well-lit selfie image, entering a descriptive prompt, and providing a valid Defapi API key.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow requires a Defapi account and API key, which can be obtained by signing up at [Defapi.org](https://defapi.org/model/google/nano-banana).                                                                                                                                                                                                                                                                      | Defapi API registration                          |
| Avoid using dark or poorly lit photos as input; the generated 3D figurine images will inherit poor lighting and appear dark.                                                                                                                                                                                                                                                                                               | Best practice for input photo quality            |
| Example prompt for high-quality output: Create a 1/7 scale commercialized figurine of the characters in the picture, in a realistic style, placed on a computer desk with a round transparent acrylic base and no text on the base. The computer screen shows the Zbrush modeling process of the figurine, next to a packaging box with rounded corners and transparent front window displaying the figure inside. | Example creative prompt content                    |
| Technical details: API endpoint `https://api.defapi.org/api/image/gen` (POST) for submission and `https://api.defapi.org/api/task/query` (GET) for status check; Bearer token authentication required.                                                                                                                                                                                                                         | API technical specification                      |
| For detailed workflow description and setup instructions, consult the sticky note titled "Sticky Note3" at the start of the workflow for comprehensive guidance.                                                                                                                                                                                                                                                          | Workflow overview sticky note                     |
| Sample input and output images are included as sticky notes for visual reference: Input photo (Sticky Note2) and Result image (Sticky Note4).                                                                                                                                                                                                                                                                               | Visual examples                                  |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow created with compliance to content policies. It contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.