Generate Unlimited E-Commerce Ad Creative with Nano Banana Image Generator

https://n8nworkflows.xyz/workflows/generate-unlimited-e-commerce-ad-creative-with-nano-banana-image-generator-8226


# Generate Unlimited E-Commerce Ad Creative with Nano Banana Image Generator

### 1. Workflow Overview

This workflow, titled **"The Recap AI - Nano Banana Influencer Ad Creative"**, is designed to generate unlimited e-commerce advertisement creatives by combining a product image with influencer reference images. It automates the process of creating contextual ad images where a product (e.g., a tumbler or cup) is visually integrated with an influencer photo, simulating a natural setting such as a person holding the product at a café. The generated images are then uploaded back to a designated Google Drive folder for easy access and further use.

The workflow can serve e-commerce marketers, social media managers, and digital advertisers aiming to efficiently create personalized ad creatives featuring products endorsed by influencers without manual graphic design.

It is logically divided into the following blocks:

- **1.1 Input Reception:** Receives a product image upload via a web form.
- **1.2 Influencer Image Retrieval:** Lists and iterates over influencer images stored in Google Drive.
- **1.3 Image Processing:** Downloads influencer images and converts both product and influencer images into base64 format.
- **1.4 AI Image Generation:** Calls Google Gemini API to generate a composite image based on the product and influencer images.
- **1.5 Post-Processing and Upload:** Converts the AI-generated image into binary format and uploads it to a target Google Drive folder.
- **1.6 Workflow Initialization and Notes:** Provides user instructions and setup notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by providing a web form interface for the user to upload a product image, which will be used later for generating ad creatives.

- **Nodes Involved:**  
  - `form_trigger`  
  - `product_image_to_base64`

- **Node Details:**  

  1. **form_trigger**  
     - *Type & Role:* Form Trigger node that exposes a webhook for user input via a custom form.  
     - *Configuration:*  
       - Form title: "Influencer Ad Creative Generator"  
       - Form field: Single file upload labeled "Image," required for submission.  
       - Description: "Select and upload an image of your product."  
     - *Input/Output:* Triggers on HTTP POST with uploaded image file; outputs the file in binary data under property "Image."  
     - *Failure Modes:* File upload failure, webhook execution errors, invalid file format.  
     - *Notes:* Essential entry point for the workflow.

  2. **product_image_to_base64**  
     - *Type & Role:* ExtractFromFile node; converts the binary product image file from the form into a base64 string for API use.  
     - *Configuration:*  
       - Operation: Convert binary property "Image" to a JSON property (base64 string).  
     - *Input/Output:* Input from `form_trigger`, outputs JSON with base64 data in `.data`.  
     - *Failure Modes:* Binary data missing or corrupt, failure in conversion.  
     - *Version:* v1  
     - *Connections:* Passes output to `list_influencer_images`.

#### 1.2 Influencer Image Retrieval

- **Overview:**  
  This block fetches all influencer reference images stored in a specified Google Drive folder and prepares them for iterative processing.

- **Nodes Involved:**  
  - `list_influencer_images`  
  - `iterate_influencer_images`

- **Node Details:**

  1. **list_influencer_images**  
     - *Type & Role:* Google Drive node; lists all files inside the configured folder containing influencer images.  
     - *Configuration:*  
       - Folder ID specified (hardcoded ID: "1HTaxyt9ZIlf3faATFlN4ujlTZge9_yg-").  
       - Returns all files without pagination.  
       - Credential: Google Drive OAuth2 account.  
     - *Input/Output:* Receives base64 product image data, outputs list of influencer image metadata.  
     - *Failure Modes:* Authentication failure, folder ID invalid, API rate limits.

  2. **iterate_influencer_images**  
     - *Type & Role:* SplitInBatches node; iterates over the influencer image list one by one for sequential processing.  
     - *Configuration:* Default batch size (1 item per batch).  
     - *Input/Output:* Input from `list_influencer_images`, outputs individual file metadata per batch.  
     - *Failure Modes:* Empty input array, batch processing errors.

#### 1.3 Image Processing

- **Overview:**  
  Downloads each influencer image from Google Drive and converts both product and influencer images into base64 strings for AI processing.

- **Nodes Involved:**  
  - `download_influencer_image`  
  - `influencer_image_to_base_64`

- **Node Details:**

  1. **download_influencer_image**  
     - *Type & Role:* Google Drive node; downloads each influencer image file by ID.  
     - *Configuration:*  
       - File ID dynamically set from current batch item (`={{ $json.id }}`).  
       - Operation: Download binary file.  
       - Credential: Google Drive OAuth2 account.  
     - *Input/Output:* Input from `iterate_influencer_images` (second output path), outputs binary influencer image.  
     - *Failure Modes:* Invalid file ID, download failures, permission errors.

  2. **influencer_image_to_base_64**  
     - *Type & Role:* ExtractFromFile node; converts the downloaded influencer image binary to base64 string.  
     - *Configuration:* Default binary property conversion.  
     - *Input/Output:* Input from `download_influencer_image`, outputs JSON base64 string in `.data`.  
     - *Failure Modes:* Missing binary data, conversion errors.

#### 1.4 AI Image Generation

- **Overview:**  
  Calls the Google Gemini API to generate a new image combining the product and influencer images with a naturalistic scenario prompt.

- **Nodes Involved:**  
  - `generate_image`  
  - `set_result`

- **Node Details:**

  1. **generate_image**  
     - *Type & Role:* HTTP Request node; submits a POST request to Google Gemini Image Generation API.  
     - *Configuration:*  
       - URL: Gemini v1beta model endpoint for image preview generation.  
       - Method: POST with JSON body containing:  
         - Text prompt describing the desired image composite (person holding product cup at café, friendly photo style, angled side view).  
         - Inline base64 product image (PNG).  
         - Inline base64 influencer image (JPEG).  
       - Authentication: HTTP Header Auth with Google Gemini credentials.  
       - Uses expressions to dynamically inject base64 images from previous nodes.  
     - *Input/Output:* Inputs base64 images, outputs JSON with image candidates containing base64-encoded generated image.  
     - *Failure Modes:* API authentication failure, network timeout, prompt or payload errors, malformed base64 data.

  2. **set_result**  
     - *Type & Role:* Set node; extracts the first generated image base64 string from the API response and assigns it to a JSON property `image_result`.  
     - *Configuration:* Uses an expression to filter and retrieve inline data from first candidate part.  
     - *Input/Output:* Input from `generate_image`, output JSON with `image_result`.  
     - *Failure Modes:* Response structure changes, missing data, expression evaluation errors.

#### 1.5 Post-Processing and Upload

- **Overview:**  
  Converts the generated image base64 string into binary format and uploads the image to a specified Google Drive folder for storage and use.

- **Nodes Involved:**  
  - `get_image`  
  - `upload_image`

- **Node Details:**

  1. **get_image**  
     - *Type & Role:* ConvertToFile node; transforms the base64 string property `image_result` into a binary file for upload.  
     - *Configuration:*  
       - Source property: `image_result` (string base64).  
       - Operation: Convert to binary.  
     - *Input/Output:* Input from `set_result`, outputs binary file data.  
     - *Failure Modes:* Invalid base64 string, conversion errors.

  2. **upload_image**  
     - *Type & Role:* Google Drive node; uploads the generated binary image file to a destination folder.  
     - *Configuration:*  
       - File name template: "Influencer Image #{{ $runIndex + 1 }}" (auto-increment per run).  
       - Destination folder ID: "1ZatlrK3cMUHkeel-HTeCFYDf1mdBRAWj" (hardcoded).  
       - Drive: "My Drive".  
       - Credential: Google Drive OAuth2 account.  
     - *Input/Output:* Input from `get_image`, outputs upload metadata.  
     - *Failure Modes:* Authentication, permission errors, quota limits, invalid folder ID.

  - *Connections:* Upon upload completion, the workflow loops back to process the next influencer image batch.

#### 1.6 Workflow Initialization and Notes

- **Overview:**  
  Provides setup instructions and clarifications for users via a sticky note displayed within the workflow editor.

- **Nodes Involved:**  
  - `Sticky Note`

- **Node Details:**

  1. **Sticky Note**  
     - *Type & Role:* Informational note for users.  
     - *Content:*  
       - Title: "Nano Banana Ad Creative Generator"  
       - Instructions:  
         1. Upload influencer reference images to the source Google Drive folder.  
         2. Create a Google Drive destination folder for output images.  
         3. Upload the product image via the form to be promoted.  
     - *Position:* Visible at the start of the workflow for easy reference.

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                             | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                         |
|---------------------------|----------------------|--------------------------------------------|--------------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| form_trigger              | Form Trigger         | Receives product image upload via form     | -                        | product_image_to_base64    |                                                                                                   |
| product_image_to_base64   | ExtractFromFile      | Converts product image binary to base64    | form_trigger             | list_influencer_images     |                                                                                                   |
| list_influencer_images    | Google Drive         | Lists influencer images from source folder | product_image_to_base64  | iterate_influencer_images  |                                                                                                   |
| iterate_influencer_images | SplitInBatches       | Iterates over influencer images             | list_influencer_images   | download_influencer_image  |                                                                                                   |
| download_influencer_image | Google Drive         | Downloads influencer image file             | iterate_influencer_images| influencer_image_to_base_64|                                                                                                   |
| influencer_image_to_base_64| ExtractFromFile     | Converts influencer image binary to base64 | download_influencer_image| generate_image             |                                                                                                   |
| generate_image            | HTTP Request         | Calls Google Gemini API to generate image   | influencer_image_to_base_64| set_result               |                                                                                                   |
| set_result                | Set                  | Extracts generated image base64 from API   | generate_image           | get_image                 |                                                                                                   |
| get_image                 | ConvertToFile        | Converts base64 generated image to binary  | set_result               | upload_image              |                                                                                                   |
| upload_image              | Google Drive         | Uploads generated image to destination folder| get_image               | iterate_influencer_images |                                                                                                   |
| Sticky Note               | Sticky Note          | Workflow setup and instructions             | -                        | -                         | ## Nano Banana Ad Creative Generator\n\n### Setup\n1. Upload influencer reference images to the source Google Drive Folder\n2. Create a Google Drive destination folder for your output\n3. Upload an image of your product you want promoted by the reference influencer images |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Form Trigger node** named `form_trigger`:  
   - Set form title to: "Influencer Ad Creative Generator"  
   - Add one form field:  
     - Type: File upload  
     - Label: "Image"  
     - Required: Yes  
   - This node exposes a webhook URL for product image upload.

3. **Add an ExtractFromFile node** named `product_image_to_base64`:  
   - Operation: Convert binary to property  
   - Binary Property Name: "Image"  
   - Connect input from `form_trigger`.

4. **Add a Google Drive node** named `list_influencer_images`:  
   - Operation: List files/folders  
   - Folder ID: Set to the source Google Drive folder containing influencer images (e.g., "1HTaxyt9ZIlf3faATFlN4ujlTZge9_yg-")  
   - Credentials: Configure Google Drive OAuth2 credentials  
   - Connect input from `product_image_to_base64`.

5. **Add a SplitInBatches node** named `iterate_influencer_images`:  
   - Default batch size 1  
   - Connect input from `list_influencer_images`.

6. **Add a Google Drive node** named `download_influencer_image`:  
   - Operation: Download file  
   - File ID: Set to current item ID (`={{ $json.id }}`)  
   - Credentials: Same Google Drive OAuth2 credentials  
   - Connect input from `iterate_influencer_images` (use the second output path for batch items).

7. **Add an ExtractFromFile node** named `influencer_image_to_base_64`:  
   - Operation: Convert binary to property (default binary property)  
   - Connect input from `download_influencer_image`.

8. **Add an HTTP Request node** named `generate_image`:  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent`  
   - Authentication: HTTP Header Auth with Google Gemini API credentials  
   - Body Content-Type: JSON  
   - JSON Body (use expression to inject base64 images):  
     ```json
     {
       "contents": [{
         "parts": [
           {
             "text": "Create an image where the cup/tumbler in image 1 is being held by the person in the 2nd image (like they are about to take a drink from the cup). The person should be sitting at a table at a cafe or coffee shop and is smiling warmly while looking at the camera. This is not a professional photo, it should feel like a friend is taking a picture of the person in the 2nd image. Only return the final generated image. The angle of the image should instead by slightly at an angle from the side (vary this angle)."
           },
           {
             "inline_data": {
               "mime_type": "image/png",
               "data": "{{ $node['product_image_to_base64'].json.data }}"
             }
           },
           {
             "inline_data": {
               "mime_type": "image/jpeg",
               "data": "{{ $node['influencer_image_to_base_64'].json.data }}"
             }
           }
         ]
       }]
     }
     ```  
   - Connect input from `influencer_image_to_base_64`.

9. **Add a Set node** named `set_result`:  
   - Add a field `image_result` (string) with value:  
     ```javascript
     {{$json.candidates[0].content.parts.filter(item => item.inlineData).first().inlineData.data}}
     ```  
   - Connect input from `generate_image`.

10. **Add a ConvertToFile node** named `get_image`:  
    - Operation: Convert string to binary file  
    - Source Property: `image_result`  
    - Connect input from `set_result`.

11. **Add a Google Drive node** named `upload_image`:  
    - Operation: Upload file  
    - File Name: "Influencer Image #{{ $runIndex + 1 }}"  
    - Destination Folder ID: Set to your output Google Drive folder (e.g., "1ZatlrK3cMUHkeel-HTeCFYDf1mdBRAWj")  
    - Drive: "My Drive"  
    - Credentials: Same Google Drive OAuth2 credentials  
    - Connect input from `get_image`.

12. **Connect `upload_image` output back to `iterate_influencer_images` input** to continue processing remaining influencer images.

13. **Add a Sticky Note node** named `Sticky Note` with content:  
    ```
    ## Nano Banana Ad Creative Generator
    
    ### Setup
    1. Upload influencer reference images to the source Google Drive Folder
    2. Create a Google Drive destination folder for your output
    3. Upload an image of your product you want promoted by the reference influencer images
    ```  
    Place it visibly near the start of your workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| The workflow leverages Google Gemini API for AI image generation: https://developers.generativeai.google/products/gemini                                   | Google Gemini API overview                                |
| Google Drive OAuth2 credentials must have read access to the influencer images folder and write access to the destination output folder.                      | Google Drive OAuth2 setup                                 |
| The image prompt is carefully crafted to produce casual, friendly photos with a natural angle rather than professional studio shots.                         | Prompt design for image generation                        |
| The workflow is designed to handle batch processing of influencer images, enabling scalable creative generation for multiple influencers automatically.       | Batch processing                                          |
| Uploaded product images must be in PNG format for best compatibility; influencer images are expected as JPEGs but can be adjusted in prompt MIME types.      | Input image format recommendations                        |
| Ensure quota and rate limits for Google Gemini and Google Drive APIs are monitored to prevent execution failures during bulk processing.                      | API quota management                                     |

---

This documentation enables developers and automation experts to fully understand, reproduce, and safely modify the "Nano Banana Influencer Ad Creative" workflow, while anticipating key points of failure and configuration requirements.