Batch ID Photo Converter & Enhancer With Google Drive & Nano Banana API

https://n8nworkflows.xyz/workflows/batch-id-photo-converter---enhancer-with-google-drive---nano-banana-api-9245


# Batch ID Photo Converter & Enhancer With Google Drive & Nano Banana API

---

### 1. Workflow Overview

This workflow automates a batch process to enhance and convert ID photos stored in a Google Drive input folder using the Nano Banana AI model via Defapi.org API, then uploads the enhanced photos back to a Google Drive output folder. It is designed for users who need to process multiple ID photos with AI enhancement while managing files on Google Drive. The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Collects user input via a form, including Google Drive folder URLs and an optional image enhancement prompt.
- **1.2 Google Drive Image Retrieval:** Searches the specified input folder on Google Drive and prepares image data for processing.
- **1.3 AI Processing Request:** Sends each image to the Nano Banana AI model via Defapi.org API and polls for completion status.
- **1.4 Result Handling and Upload:** Downloads the enhanced images from the API response and uploads them to the specified Google Drive output folder.
- **1.5 Workflow Orchestration and Control:** Includes wait nodes and conditional checks to handle asynchronous AI processing.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  Collects user input via a form trigger, capturing Google Drive folder URLs for input and output, plus an optional detailed prompt for image enhancement.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - **Type:** Form Trigger  
    - **Role:** Entry point to capture user input dynamically.  
    - **Configuration:**  
      - Form title: "Set Google Drive"  
      - Fields:  
        - Google Drive - Input Folder URL (required)  
        - Google Drive - Output Folder URL (required)  
        - Prompt (textarea, optional, with detailed default placeholder prompt describing image enhancement requirements)  
    - **Input:** HTTP form submission  
    - **Output:** JSON with form fields  
    - **Edge cases:** Missing required fields will block submission; prompt fallback uses placeholder if empty.  

---

#### Block 1.2: Google Drive Image Retrieval

- **Overview:**  
  Retrieves all files from the specified Google Drive input folder and prepares the data for the AI processing request.

- **Nodes Involved:**  
  - Search files and folders  
  - Code in JavaScript

- **Node Details:**

  - **Search files and folders**  
    - **Type:** Google Drive (File/Folder List)  
    - **Role:** Lists all files in the Google Drive folder provided by user input.  
    - **Configuration:**  
      - Folder ID: Extracted from the input folder URL submitted in the form  
      - Return all files without pagination limit  
      - Selects fields relevant for images: mimeType, name, id, webViewLink, imageMediaMetadata, webContentLink, thumbnailLink  
    - **Credentials:** Google Drive OAuth2  
    - **Input:** Triggered by form submission  
    - **Output:** List of files metadata  
    - **Edge cases:** Invalid folder URL, permission denied, empty folder returns empty array.  

  - **Code in JavaScript**  
    - **Type:** Code (JavaScript)  
    - **Role:** Prepares an array of objects containing the prompt, image URL, and filename for each retrieved file.  
    - **Configuration:**  
      - For each input item (file), copies the prompt from the form or uses placeholder text.  
      - Extracts `webContentLink` as the image URL and the filename.  
    - **Input:** List of files from Google Drive node  
    - **Output:** Array of JSON objects formatted for the API request  
    - **Edge cases:** Missing or malformed URLs, empty inputs, reliance on form prompt presence.

---

#### Block 1.3: AI Processing Request

- **Overview:**  
  Submits the images with prompts to the Defapi.org API (Nano Banana model) for AI-based enhancement, waits for processing, and polls for completion.

- **Nodes Involved:**  
  - Send Image Generation Request to Defapi.org API  
  - Wait for Image Processing Completion  
  - Obtain the generated status  
  - Check if Image Generation is Complete

- **Node Details:**

  - **Send Image Generation Request to Defapi.org API**  
    - **Type:** HTTP Request  
    - **Role:** Sends a POST request to Defapi.org to initiate image generation.  
    - **Configuration:**  
      - URL: `https://api.defapi.org/api/image/gen`  
      - Method: POST  
      - Body (JSON): Contains `prompt` from prepared data, model name `"google/nano-banana"`, and `images` array with the image URL  
      - Authentication: HTTP Bearer token (Defapi account credentials)  
      - Headers: Content-Type application/json  
    - **Input:** Prepared image and prompt data from JavaScript node  
    - **Output:** Task ID for generation status query  
    - **Edge cases:** API authentication failure, invalid image URL, request timeout.

  - **Wait for Image Processing Completion**  
    - **Type:** Wait  
    - **Role:** Pauses workflow for 10 seconds before polling status.  
    - **Configuration:** Fixed 10 seconds wait  
    - **Input:** After sending generation request  
    - **Output:** Triggers next status check  
    - **Edge cases:** Fixed wait time may be suboptimal; longer processing times require multiple loops.

  - **Obtain the generated status**  
    - **Type:** HTTP Request  
    - **Role:** Queries Defapi.org API for image generation task status.  
    - **Configuration:**  
      - URL: `https://api.defapi.org/api/task/query`  
      - Method: GET with query parameter `task_id` from previous response  
      - Authentication: HTTP Bearer token (Defapi account)  
      - Headers: Content-Type application/json  
    - **Input:** Task ID from generation request  
    - **Output:** JSON with generation status and results  
    - **Edge cases:** Invalid task ID, network errors, API rate limits.

  - **Check if Image Generation is Complete**  
    - **Type:** If  
    - **Role:** Determines if generation is complete by checking that status is not `"pending"`.  
    - **Configuration:** Condition: `$json.data.status != "pending"`  
    - **Input:** Status response JSON  
    - **Output:**  
      - True branch: proceeds to result handling  
      - False branch: loops back to wait again  
    - **Edge cases:** Unexpected status values, API errors, infinite loops if status stuck.

---

#### Block 1.4: Result Handling and Upload

- **Overview:**  
  Once processing is complete, formats the results for display, downloads the enhanced image, and uploads it to the Google Drive output folder.

- **Nodes Involved:**  
  - Format and Display Image Results  
  - HTTP Request  
  - Upload file

- **Node Details:**

  - **Format and Display Image Results**  
    - **Type:** Set  
    - **Role:** Prepares markdown content with generation status, message, and image URL for downstream usage or display.  
    - **Configuration:**  
      - Sets `markdown_content` combining status, text message, and image URL in markdown syntax  
      - Sets `image` field with the generated image URL  
    - **Input:** Generation status with result data  
    - **Output:** JSON with formatted content and image URL  
    - **Edge cases:** Missing or malformed result data.

  - **HTTP Request**  
    - **Type:** HTTP Request  
    - **Role:** Downloads the generated image file from the URL.  
    - **Configuration:**  
      - URL: the generated image URL from previous node (`$json.image`)  
      - Response format: file (binary)  
    - **Input:** Image URL  
    - **Output:** Binary file data for upload  
    - **Edge cases:** Broken image URL, download failures, large file size.

  - **Upload file**  
    - **Type:** Google Drive  
    - **Role:** Uploads the downloaded enhanced image to the Google Drive output folder specified by user.  
    - **Configuration:**  
      - Drive: My Drive  
      - Folder ID: Extracted from form input output folder URL  
    - **Credentials:** Google Drive OAuth2  
    - **Input:** Binary file from HTTP Request  
    - **Output:** Confirmation of upload with file metadata  
    - **Edge cases:** Invalid folder URL, permission errors, quota limits.

---

#### Block 1.5: Workflow Orchestration and Control

- **Overview:**  
  Controls timing and looping for asynchronous AI processing, ensuring images are processed fully before proceeding.

- **Nodes Involved:**  
  - Wait for Image Processing Completion  
  - Check if Image Generation is Complete  
  - Obtain the generated status

- **Node Details:**  
  These nodes are described in previous blocks, but as a group they enable:

  - Initial wait after submission  
  - Polling the Defapi API for status  
  - Conditional looping to wait again if processing not done

- **Edge cases:**  
  Potential infinite loops if task stays pending indefinitely; no max retry limit configured.

---

### 3. Summary Table

| Node Name                            | Node Type               | Functional Role                         | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                                         |
|------------------------------------|-------------------------|---------------------------------------|-------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| On form submission                 | Form Trigger            | Collects user inputs (folders, prompt)| -                             | Search files and folders                    |                                                                                                                     |
| Search files and folders           | Google Drive            | Retrieves files from input folder      | On form submission            | Code in JavaScript                          |                                                                                                                     |
| Code in JavaScript                 | Code                    | Prepares image and prompt data         | Search files and folders      | Send Image Generation Request to Defapi.org API |                                                                                                                     |
| Send Image Generation Request to Defapi.org API | HTTP Request            | Sends generation request to Nano Banana API | Code in JavaScript            | Wait for Image Processing Completion        |                                                                                                                     |
| Wait for Image Processing Completion| Wait                    | Waits 10 seconds before status check   | Send Image Generation Request | Obtain the generated status                  |                                                                                                                     |
| Obtain the generated status       | HTTP Request            | Polls API for generation status        | Wait for Image Processing Completion | Check if Image Generation is Complete       |                                                                                                                     |
| Check if Image Generation is Complete | If                      | Checks if generation is complete       | Obtain the generated status   | Format and Display Image Results (True branch); Wait for Image Processing Completion (False branch) |                                                                                                                     |
| Format and Display Image Results  | Set                     | Formats markdown and image URL          | Check if Image Generation is Complete | HTTP Request                              |                                                                                                                     |
| HTTP Request                     | HTTP Request            | Downloads generated image file          | Format and Display Image Results | Upload file                                |                                                                                                                     |
| Upload file                      | Google Drive            | Uploads enhanced image to output folder| HTTP Request                 | -                                          |                                                                                                                     |
| Sticky Note1                     | Sticky Note             | Explains Input Image Load block steps  | -                             | -                                          | ## Load Images From Google Drive Input Dir 1. On form submission - Collects Google Drive folder URLs and optional prompt 2. Search files and folders - Retrieves all files from the input folder 3. Code in JavaScript - Prepares image data and prompt for API request |
| Sticky Note2                     | Sticky Note             | Shows Google Drive Input Folder screenshot | -                             | -                                          | ## Google Drive Input Dir ![Image](https://i.imgur.com/7VcUtl6.jpeg)                                                  |
| Sticky Note3                     | Sticky Note             | Shows Google Drive Output Folder screenshot | -                             | -                                          | ## Google Drive Output Dir ![Image](https://i.imgur.com/j5mWcqz.jpeg)                                                 |
| Sticky Note4                     | Sticky Note             | Explains AI Processing block steps     | -                             | -                                          | ## Send Image to Nano Banana API (Defapi.org) 4. Send Image Generation Request to Defapi.org API - Submits generation request for each image 5. Wait for Image Processing Completion - Waits 10 seconds before checking status 6. Obtain the generated status - Polls API for completion status 7. Check if Image Generation is Complete - Checks if status is not "pending" |
| Sticky Note5                     | Sticky Note             | Explains Result Handling and Upload block steps | -                             | -                                          | ## Download Images And Upload to Google Drive 8. Format and Display Image Results - Formats result with markdown and image URL 9. HTTP Request - Downloads the generated image file 10. Upload file - Uploads the enhanced photo to the output folder |
| Sticky Note                      | Sticky Note             | Explains Google Drive folder setup     | -                             | -                                          | ## How to use drive - Prepare two folders. - One for input: https://drive.google.com/drive/folders/xxxxxxx - One for output: https://drive.google.com/drive/folders/yyyyyy - Maker your folders public |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named `On form submission`  
   - Set form title: "Set Google Drive"  
   - Add form fields:  
     - "Google Drive - Input Folder URL" (required, single line)  
     - "Google Drive - Output Folder URL" (required, single line)  
     - "Prompt" (textarea, optional, with the detailed default prompt text for ID photo enhancement)  

2. **Add a Google Drive node** named `Search files and folders`  
   - Operation: List files/folders  
   - Set folder ID by extracting from `On form submission` input field "Google Drive - Input Folder URL" (use expression to parse URL and get folder ID)  
   - Enable "Return All"  
   - Select fields: mimeType, name, id, webViewLink, imageMediaMetadata, webContentLink, thumbnailLink  
   - Connect `On form submission` output to this node input  
   - Use Google Drive OAuth2 credentials  

3. **Add a Code node** named `Code in JavaScript`  
   - Use JavaScript to iterate over all input items from `Search files and folders`  
   - For each item, create an object containing:  
     - `prompt`: from form prompt or placeholder text  
     - `image`: `webContentLink` of the file  
     - `name`: file name  
   - Return an array of these objects  
   - Connect `Search files and folders` to this node  

4. **Add an HTTP Request node** named `Send Image Generation Request to Defapi.org API`  
   - Method: POST  
   - URL: `https://api.defapi.org/api/image/gen`  
   - Body format: JSON  
   - Body content:  
     ```json
     {
       "prompt": "={{$json.prompt}}",
       "model": "google/nano-banana",
       "images": ["{{$json.image}}"]
     }
     ```  
   - Authentication: HTTP Bearer with Defapi credentials  
   - Headers: Content-Type: application/json  
   - Connect `Code in JavaScript` output to this node  

5. **Add a Wait node** named `Wait for Image Processing Completion`  
   - Set wait time: 10 seconds  
   - Connect `Send Image Generation Request to Defapi.org API` output to this node  

6. **Add an HTTP Request node** named `Obtain the generated status`  
   - Method: GET  
   - URL: `https://api.defapi.org/api/task/query`  
   - Add query parameter `task_id` with value from previous node response `{{$json.data.task_id}}`  
   - Authentication: HTTP Bearer with Defapi credentials  
   - Headers: Content-Type: application/json  
   - Connect `Wait for Image Processing Completion` output to this node  

7. **Add an If node** named `Check if Image Generation is Complete`  
   - Condition: Check if `{{$json.data.status}}` NOT equal to `"pending"`  
   - Connect `Obtain the generated status` output to this node  

8. **Connect the If node** branches as follows:  
   - **True branch:** Connect to a Set node named `Format and Display Image Results`  
   - **False branch:** Loop back to the `Wait for Image Processing Completion` node to retry  

9. **Add a Set node** named `Format and Display Image Results`  
   - Add fields:  
     - `markdown_content` (string): formatted markdown with status, message, and embedded image  
     - `image` (string): URL of generated image from `{{$json.data.result[0].image}}`  
   - Connect `Check if Image Generation is Complete` True branch to this node  

10. **Add an HTTP Request node** named `HTTP Request`  
    - Method: GET  
    - URL: `{{$json.image}}` from previous node output  
    - Response format: File (binary)  
    - Connect `Format and Display Image Results` to this node  

11. **Add a Google Drive node** named `Upload file`  
    - Operation: Upload file  
    - Drive: "My Drive"  
    - Folder ID: Extracted from form output folder URL (`{{$json['Google Drive - Output Folder URL']}}`)  
    - Input: binary data from `HTTP Request` node  
    - Connect `HTTP Request` output to this node  
    - Use Google Drive OAuth2 credentials  

12. **Add Sticky Notes** for documentation and clarity on Google Drive folder setup, workflow blocks, and process instructions as shown in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Prepare two Google Drive folders: One for input and one for output. Make these folders public to ensure n8n access.               | Sticky Note: https://drive.google.com/drive/folders/xxxxxxx (input), https://drive.google.com/drive/folders/yyyyyy (output) |
| Use Nano Banana AI model via Defapi.org for professional portrait enhancement with detailed prompt instructions included in form. | API Documentation: https://defapi.org/api/image/gen                                                        |
| The workflow uses a polling mechanism with a 10-second wait interval to handle asynchronous image generation status checking.     | Potential enhancement: Add max retry or timeout logic to avoid infinite loops.                             |
| Screenshots in sticky notes illustrate folder setup in Google Drive UI for user reference.                                         | Sticky Note images: Input folder ![Drive Input](https://i.imgur.com/7VcUtl6.jpeg), Output folder ![Drive Output](https://i.imgur.com/j5mWcqz.jpeg) |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---