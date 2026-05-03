Generate & Edit Images with OpenAI GPT-Image-1 and Share via Telegram

https://n8nworkflows.xyz/workflows/generate---edit-images-with-openai-gpt-image-1-and-share-via-telegram-4391


# Generate & Edit Images with OpenAI GPT-Image-1 and Share via Telegram

### 1. Workflow Overview

This workflow enables users to generate or edit images using OpenAI’s GPT-Image-1 model based on prompts submitted via a web form, and then share the resulting images through Telegram. It supports two core use cases: generating new images from text prompts, or editing existing images with descriptive prompts. The workflow handles user input, interacts with OpenAI’s image generation and editing APIs, processes and converts image data, uploads the final image either to Google Drive or imgbb for hosting, sets appropriate sharing permissions, and finally sends the image with a caption to a Telegram chat.

Logical blocks:

- **1.1 Input Reception and Mode Selection**: Captures user input from a form including prompt, quality level, and image input (file or URL). Determines workflow path based on presence of image input.

- **1.2 Image Processing with OpenAI**: Depending on mode, either edits an existing image or generates a new image from prompt using OpenAI GPT-Image-1 API endpoints.

- **1.3 Image Conversion**: Converts OpenAI base64-encoded image data into binary PNG files for further processing.

- **1.4 Image Upload and Sharing Setup**: Uploads the processed image to either Google Drive or imgbb, sets access permissions, and prepares shareable links.

- **1.5 Telegram Notification**: Sends the final image along with caption and links to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Mode Selection

**Overview:**  
Receives form submissions containing the image prompt, quality selection, and either an uploaded input image or an image URL. Uses a switch node to determine whether to proceed with image editing, image retrieval by URL, or image generation.

**Nodes Involved:**  
- On form submission  
- Switch Mode  

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; listens for HTTP form submissions with basic authentication.  
  - *Configuration:* Form titled "GPT image to image" with fields: "Promt" (text, required), "quality" (dropdown: low, medium, high), "input image" (single file, accepts PNG/JPG/JPEG), "input image url" (text).  
  - *Input:* HTTP form data.  
  - *Output:* JSON containing form fields.  
  - *Failure types:* Authentication errors, missing required fields, malformed input.  
  - *Notes:* BasicAuth credential used for security.

- **Switch Mode**  
  - *Type:* Switch  
  - *Role:* Determines processing path based on presence of uploaded file and/or image URL.  
  - *Configuration:*  
    - Output "file": true if an uploaded file is present and image URL is also not empty.  
    - Output "file_url": true if image URL is present and not empty.  
    - Fallback: "extra" (no explicit use in workflow).  
  - *Input:* JSON from form submission.  
  - *Output:* Routes to either image editing, image retrieval, or generation.  
  - *Failure types:* Incorrect logic if fields are missing or malformed.

---

#### 1.2 Image Processing with OpenAI

**Overview:**  
Processes images using OpenAI's GPT-Image-1 model. For editing, sends an existing image and prompt to the edits endpoint. For generation, sends prompt and parameters to the generation endpoint. If an image URL is provided, it first downloads the image.

**Nodes Involved:**  
- Edit Image (OpenAI)  
- Image Generation  
- Get Input Image by URL  

**Node Details:**

- **Edit Image (OpenAI)**  
  - *Type:* HTTP Request  
  - *Role:* Calls OpenAI `/v1/images/edits` API to modify an existing image based on prompt.  
  - *Configuration:*  
    - POST to `https://api.openai.com/v1/images/edits`  
    - multipart-form-data with parameters:  
      - `image`: binary image from uploaded file or URL download  
      - `prompt`: text prompt from form  
      - `model`: "gpt-image-1"  
      - `n`: 1 image  
      - `size`: 1024x1024  
      - `quality`: from form input (low/medium/high)  
    - Authentication: OpenAI API credential.  
  - *Input:* Binary image data or downloaded image + prompt.  
  - *Output:* Base64-encoded edited image JSON.  
  - *Failures:* API errors (auth, rate limits), malformed images, network timeouts.

- **Image Generation**  
  - *Type:* HTTP Request  
  - *Role:* Calls OpenAI `/v1/images/generations` API to create an image from prompt.  
  - *Configuration:*  
    - POST to `https://api.openai.com/v1/images/generations`  
    - JSON body with parameters:  
      - `model`: "gpt-image-1"  
      - `prompt`: from form  
      - `n`: 1  
      - `size`: 1024x1024  
      - `moderation`: low  
      - `background`: auto  
      - `quality`: from form input  
    - Authentication: OpenAI API credential.  
  - *Input:* Prompt and quality.  
  - *Output:* Base64-encoded generated image JSON.  
  - *Failures:* API errors, invalid prompts, rate limits.

- **Get Input Image by URL**  
  - *Type:* HTTP Request  
  - *Role:* Downloads image from the user-provided URL for editing.  
  - *Configuration:*  
    - GET request to the URL specified in form field "input image url".  
  - *Input:* URL string.  
  - *Output:* Binary image data.  
  - *Failures:* Invalid URL, network errors, unsupported image formats.

---

#### 1.3 Image Conversion

**Overview:**  
Converts the base64 JSON image data returned by OpenAI into binary PNG files for uploading and further usage.

**Nodes Involved:**  
- Convert json binary to File  
- Convert json binary to File final  

**Node Details:**

- **Convert json binary to File**  
  - *Type:* ConvertToFile  
  - *Role:* Converts base64-encoded image data from generation response into binary file.  
  - *Configuration:*  
    - Source property: `data[0].b64_json`  
    - File name: current timestamp + ".png"  
    - MIME type: image/png  
  - *Input:* JSON with base64 image string.  
  - *Output:* Binary image file.  
  - *Failures:* Invalid base64 data causing conversion errors.

- **Convert json binary to File final**  
  - *Type:* ConvertToFile  
  - *Role:* Similar to above but used for edited images.  
  - *Configuration:* Same as above.  
  - *Input:* JSON from edit image response.  
  - *Output:* Binary image file.  
  - *Failures:* Same as above.

---

#### 1.4 Image Upload and Sharing Setup

**Overview:**  
Uploads the final image file to either Google Drive or imgbb image hosting service. If uploaded to Google Drive, it sets the file’s sharing permissions to be publicly readable and collects shareable links.

**Nodes Involved:**  
- SETUP API KEY  
- Upload Result Image to imgbb  
- Upload Result Image to Google Drive  
- Set Access Permissions  
- File Link  
- File Link1  

**Node Details:**

- **SETUP API KEY**  
  - *Type:* Set  
  - *Role:* Assigns the environment variable or API key for imgbb uploads.  
  - *Configuration:* Sets `IMGBB_API_KEY` with a stored value or placeholder.  
  - *Failures:* Missing or invalid API key.

- **Upload Result Image to imgbb**  
  - *Type:* HTTP Request  
  - *Role:* Uploads binary image to imgbb via their API.  
  - *Configuration:*  
    - POST to `https://api.imgbb.com/1/upload`  
    - Multipart form data with `image` binary and `key` API key.  
  - *Input:* Binary image from conversion node.  
  - *Output:* JSON response with hosted image URLs.  
  - *Failures:* API errors, invalid API key, network failures.

- **Upload Result Image to Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Uploads binary image file to Google Drive root folder.  
  - *Configuration:*  
    - Uses service account credentials  
    - File named by binary filename  
    - Uploads to root folder of “My Drive”  
  - *Input:* Binary image file.  
  - *Output:* Metadata including file ID and links.  
  - *Failures:* Authentication errors, permission issues, file size limits.

- **Set Access Permissions**  
  - *Type:* Google Drive node  
  - *Role:* Sets public read permission on uploaded Google Drive file for sharing.  
  - *Configuration:*  
    - Operation: share  
    - Role: reader  
    - Type: anyone  
  - *Input:* File ID from previous node.  
  - *Output:* Confirmation of permission set.  
  - *Failures:* Insufficient permissions, invalid file ID.

- **File Link**  
  - *Type:* Set  
  - *Role:* Extracts and stores shareable Google Drive links (`webViewLink` and `webContentLink`) for use in Telegram captions.  
  - *Input:* Google Drive upload response JSON.  
  - *Output:* JSON with `ViewLink` and `ContentLink` strings.

- **File Link1**  
  - *Type:* Set  
  - *Role:* Extracts hosted image URLs from imgbb upload response.  
  - *Input:* imgbb upload JSON.  
  - *Output:* JSON with `ViewLink` and `ContentLink`.

---

#### 1.5 Telegram Notification

**Overview:**  
Sends the final image to a specified Telegram chat with a caption containing the original prompt, quality, and a link to the hosted image.

**Nodes Involved:**  
- Send to Telegram  

**Node Details:**

- **Send to Telegram**  
  - *Type:* Telegram node  
  - *Role:* Sends a photo message to Telegram chat.  
  - *Configuration:*  
    - Operation: sendPhoto  
    - ChatId: configured Telegram chat identifier (replace `"YOUR_CHAT_ID"`)  
    - File: uses `ContentLink` from either Google Drive or imgbb  
    - Caption: Markdown formatted text including prompt, quality, and clickable link to `ViewLink`  
    - Parse mode: MarkdownV2 for proper formatting  
  - *Input:* JSON containing file URL and links from previous nodes.  
  - *Output:* Telegram API response.  
  - *Failures:* Invalid chat ID, network errors, malformed captions.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                                 | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                      |
|------------------------------|---------------------|------------------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission            | Form Trigger        | Receives user input via authenticated form     | -                            | Switch Mode                   |                                                                                                |
| Switch Mode                  | Switch              | Routes flow based on presence of image input   | On form submission            | Edit Image (OpenAI), Get Input Image by URL, Image Generation |                                                                                                |
| Get Input Image by URL        | HTTP Request        | Downloads image from URL for editing            | Switch Mode                  | Edit Image (OpenAI)            |                                                                                                |
| Edit Image (OpenAI)           | HTTP Request        | Calls OpenAI edit API with image and prompt    | Switch Mode, Get Input Image by URL | Convert json binary to File final | ✏️ Image to image generation (OpenAI) - Sends binary image to `/v1/images/edits` endpoint. Model: `gpt-image-1` |
| Convert json binary to File final | ConvertToFile    | Converts base64 edit output to binary PNG      | Edit Image (OpenAI)           | Upload Result Image to Google Drive | 🧾 Convert base64 to File - Converts `b64_json` to PNG binary image                            |
| Upload Result Image to Google Drive | Google Drive    | Uploads image binary to Google Drive            | Convert json binary to File final | Set Access Permissions        | ## Upload result file to Google Drive OR to [imgbb](https://api.imgbb.com/)                   |
| Set Access Permissions         | Google Drive        | Makes uploaded Google Drive image publicly readable | Upload Result Image to Google Drive | File Link                    |                                                                                                |
| File Link                    | Set                 | Extracts Google Drive shareable links           | Set Access Permissions        | Send to Telegram              |                                                                                                |
| Image Generation             | HTTP Request        | Calls OpenAI generation API with prompt         | Switch Mode                  | Convert json binary to File    | 🎨 Image Generation (OpenAI) - Sends POST to `/v1/images/generations` endpoint. Model: `gpt-image-1` |
| Convert json binary to File   | ConvertToFile       | Converts base64 generation output to binary PNG | Image Generation             | SETUP API KEY                 | 🧾 Convert base64 to File - Converts `b64_json` to PNG binary image                            |
| SETUP API KEY                | Set                 | Sets API key for imgbb upload                    | Convert json binary to File   | Upload Result Image to imgbb  | ## Upload result file to Google Drive OR to [imgbb](https://api.imgbb.com/)                   |
| Upload Result Image to imgbb  | HTTP Request        | Uploads image binary to imgbb                     | SETUP API KEY                | File Link1                   |                                                                                                |
| File Link1                   | Set                 | Extracts imgbb hosted image URLs                  | Upload Result Image to imgbb | Send to Telegram              |                                                                                                |
| Send to Telegram             | Telegram             | Sends the image with caption to Telegram chat   | File Link, File Link1         | -                             |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure form title: "GPT image to image"  
   - Fields:  
     - "Promt" (text, required)  
     - "quality" (dropdown: low, medium, high, required)  
     - "input image" (file, single, accept .png, .jpg, .jpeg)  
     - "input image url" (text)  
   - Enable Basic Authentication with credentials.  
   - Save webhook ID for external triggering.

2. **Create Switch Node ("Switch Mode"):**  
   - Add rules:  
     - Output "file": when `input image` is not null AND `input image url` is not empty  
     - Output "file_url": when `input image url` is not null AND not empty  
     - Fallback output: "extra" (no action connected)  
   - Connect "On form submission" main output to this switch node.

3. **Create HTTP Request Node ("Get Input Image by URL"):**  
   - Method: GET  
   - URL: expression `={{ $json['input image url'] }}`  
   - Connected from "Switch Mode" output "file_url".

4. **Create HTTP Request Node ("Edit Image (OpenAI)"):**  
   - Method: POST  
   - URL: `https://api.openai.com/v1/images/edits`  
   - Authentication: OpenAI API (set via credentials)  
   - Request type: multipart/form-data  
   - Body parameters:  
     - "image": formBinaryData from input binary field `data` or `input_image` depending on source  
     - "prompt": from JSON `Promt` field  
     - "model": "gpt-image-1"  
     - "n": 1  
     - "size": "1024x1024"  
     - "quality": from JSON `quality` field  
   - Connect inputs from "Switch Mode" output "file" and from "Get Input Image by URL".

5. **Create HTTP Request Node ("Image Generation"):**  
   - Method: POST  
   - URL: `https://api.openai.com/v1/images/generations`  
   - Authentication: OpenAI API  
   - JSON body parameters:  
     - "model": "gpt-image-1"  
     - "prompt": from JSON `Promt`  
     - "n": 1  
     - "size": "1024x1024"  
     - "moderation": "low"  
     - "background": "auto"  
     - "quality": from JSON `quality`  
   - Connect from "Switch Mode" fallback or appropriate output for new generation.

6. **Create ConvertToFile Nodes:**  
   - Node 1 ("Convert json binary to File"):  
     - Source property: `data[0].b64_json`  
     - File name: current timestamp + ".png"  
     - MIME type: image/png  
     - Connected from "Image Generation".  
   - Node 2 ("Convert json binary to File final"):  
     - Same setup as above.  
     - Connected from "Edit Image (OpenAI)".

7. **Create Set Node ("SETUP API KEY"):**  
   - Assign `IMGBB_API_KEY` with your imgbb API key string.  
   - Connect from "Convert json binary to File".

8. **Create HTTP Request Node ("Upload Result Image to imgbb"):**  
   - Method: POST  
   - URL: `https://api.imgbb.com/1/upload`  
   - Body type: multipart/form-data  
   - Parameters:  
     - "image": binary data from previous node  
     - "key": `IMGBB_API_KEY`  
   - Connect from "SETUP API KEY".

9. **Create Google Drive Node ("Upload Result Image to Google Drive"):**  
   - Operation: Upload file  
   - Drive: "My Drive"  
   - Folder: root  
   - File name: from binary file name  
   - Authentication: Google Service Account credentials  
   - Connect from "Convert json binary to File final".

10. **Create Google Drive Node ("Set Access Permissions"):**  
    - Operation: share  
    - Role: reader  
    - Type: anyone  
    - File ID: from uploaded Google Drive node output  
    - Connect from "Upload Result Image to Google Drive".

11. **Create Set Nodes ("File Link" and "File Link1"):**  
    - "File Link": extract `webViewLink` and `webContentLink` from Google Drive upload response.  
    - "File Link1": extract image URLs from imgbb upload response.  
    - Connect "File Link" from "Set Access Permissions".  
    - Connect "File Link1" from "Upload Result Image to imgbb".

12. **Create Telegram Node ("Send to Telegram"):**  
    - Operation: sendPhoto  
    - Chat ID: your Telegram chat ID  
    - File URL: from either "File Link" or "File Link1" `ContentLink`  
    - Caption: markdown with prompt, quality, and `[Result](ViewLink)`  
    - Parse mode: MarkdownV2  
    - Credentials: Telegram API credentials  
    - Connect from both "File Link" and "File Link1".

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                              |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| The workflow uses OpenAI GPT-Image-1 model for both image generation and editing endpoints.    | OpenAI Image Generation API Documentation                    |
| Images are uploaded either to Google Drive (via service account) or imgbb (requires API key). | imgbb API: https://api.imgbb.com/                            |
| Telegram bot credentials must be set up with appropriate permissions and chat ID configured.  | Telegram Bot API Documentation                               |
| The workflow includes basic HTTP Basic Authentication on the form trigger for security.       | n8n Form Trigger Node documentation                          |
| Image quality parameter controls the quality setting passed to OpenAI endpoints.               | Quality options: low, medium, high                            |
| MarkdownV2 parse mode is used in Telegram captions; special characters in prompt may need escaping. | Telegram MarkdownV2 formatting guidelines                    |

---

**Disclaimer:**  
The text provided is derived solely from an automated n8n workflow export. It fully complies with applicable content policies and contains no illegal or protected material. All data processed is public or user-provided and legal.