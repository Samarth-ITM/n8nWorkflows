Generate Images with GPT-image-1 and Store in Google Drive with Cost Tracking

https://n8nworkflows.xyz/workflows/generate-images-with-gpt-image-1-and-store-in-google-drive-with-cost-tracking-3710


# Generate Images with GPT-image-1 and Store in Google Drive with Cost Tracking

### 1. Workflow Overview

This workflow automates the generation of images based on chat input prompts using OpenAI's `gpt-image-1` API, then stores the generated images in Google Drive and logs relevant metadata and cost tracking information in Google Sheets. It is designed for use cases where users want to generate AI images dynamically and maintain organized records of images and associated usage costs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives chat input prompts via a webhook trigger.
- **1.2 Image Generation:** Calls OpenAI's image generation API to create images from the prompt.
- **1.3 Image Processing Loop:** Splits the returned image array and processes each image individually.
- **1.4 Image File Conversion and Upload:** Converts base64 image data to files and uploads them to Google Drive.
- **1.5 Metadata Preparation and Storage:** Prepares image metadata and appends it to a Google Sheets document.
- **1.6 Cost Tracking:** Aggregates token usage and estimated costs, then records them in a separate Google Sheets tab.
- **1.7 Manual Trigger (Optional):** A disabled manual trigger node for testing the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives chat input prompts via a webhook, initiating the workflow when a new chat message is received.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger` (Webhook trigger for chat input)  
    - Configuration: Listens for incoming chat messages; webhook ID is preset.  
    - Expressions/Variables: Outputs JSON containing `chatInput` used downstream.  
    - Input: External webhook call  
    - Output: Passes chat input JSON to the next node (HTTP Request)  
    - Edge cases: Webhook misconfiguration, network issues, or malformed input could cause failure.

---

#### 2.2 Image Generation

- **Overview:**  
  Sends the chat input prompt to OpenAI's `gpt-image-1` API to generate images with specified parameters.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Configuration:  
      - URL: `https://api.openai.com/v1/images/generations`  
      - Method: POST  
      - Authentication: Uses predefined OpenAI API credentials  
      - Body Parameters:  
        - model: `gpt-image-1`  
        - prompt: `={{ $json.chatInput }}` (dynamic from webhook)  
        - output_format: `jpeg`  
        - quality: `low`  
        - output_compression: 80 (parsed integer)  
        - size: `1024x1024`  
        - n: 1 (number of images)  
        - moderation: `low`  
    - Input: JSON with chat prompt  
    - Output: JSON containing generated images in base64 and usage data (tokens)  
    - Edge cases: API authentication failure, rate limits, invalid prompt, or network timeout.

---

#### 2.3 Image Processing Loop

- **Overview:**  
  Splits the array of generated images and processes each image individually in batches of one.

- **Nodes Involved:**  
  - Split Out  
  - Loop Over Items

- **Node Details:**

  - **Split Out**  
    - Type: `n8n-nodes-base.splitOut`  
    - Configuration: Splits the `data` field containing image array, including binary data.  
    - Input: JSON with image array from HTTP Request  
    - Output: Individual image objects for further processing  
    - Edge cases: Empty or malformed image array could cause failure.

  - **Loop Over Items**  
    - Type: `n8n-nodes-base.splitInBatches`  
    - Configuration: Batch size set to 1 to process images one at a time.  
    - Input: Individual image items from Split Out  
    - Output: Sequential processing of each image  
    - Edge cases: Batch processing errors or workflow timeouts on large datasets.

---

#### 2.4 Image File Conversion and Upload

- **Overview:**  
  Converts each base64-encoded image to a binary file and uploads it to a specified Google Drive folder.

- **Nodes Involved:**  
  - Edit Fields-file_name  
  - Convert to File  
  - Google Drive  
  - Edit Fields1

- **Node Details:**

  - **Edit Fields-file_name**  
    - Type: `n8n-nodes-base.set`  
    - Configuration: Sets two fields:  
      - `file_name`: Timestamp string formatted as `yyyyMMddHHmmSSS` (unique filename)  
      - `b64_json`: Base64 image string from current item  
    - Input: Single image JSON from Loop Over Items  
    - Output: JSON with file name and base64 data  
    - Edge cases: Timestamp generation failure or missing base64 data.

  - **Convert to File**  
    - Type: `n8n-nodes-base.convertToFile`  
    - Configuration: Converts `b64_json` string to binary file with filename from timestamp.  
    - Input: JSON with base64 image string  
    - Output: Binary file data for upload  
    - Edge cases: Invalid base64 data or conversion errors.

  - **Google Drive**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Configuration:  
      - Uploads binary file to a specific folder ID in "My Drive"  
      - File name dynamically set as `chatgpt_created_by_n8n_{{created_timestamp}}`  
      - Credentials: OAuth2 Google Drive account  
    - Input: Binary file from Convert to File  
    - Output: JSON metadata of uploaded file (including `id`, `webViewLink`, `thumbnailLink`)  
    - Edge cases: Authentication failure, permission issues, quota limits, or upload errors.

  - **Edit Fields1**  
    - Type: `n8n-nodes-base.set`  
    - Configuration: Extracts and sets fields from Google Drive response:  
      - `id`, `webViewLink`, `thumbnailLink`  
      - Also passes along `file_name` from previous node  
    - Input: Google Drive upload metadata  
    - Output: Prepared metadata for Google Sheets insertion  
    - Edge cases: Missing fields or API response changes.

---

#### 2.5 Metadata Preparation and Storage

- **Overview:**  
  Appends image metadata including prompt, image link, and thumbnail to a Google Sheets document.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: `n8n-nodes-base.googleSheets`  
    - Configuration:  
      - Operation: Append row  
      - Document ID: Predefined Google Sheets document  
      - Sheet Name: Tab with gid=0 (named "工作表1")  
      - Columns mapped:  
        - `prompt`: from chat input  
        - `image`: Google Drive `webViewLink`  
        - `image_thumb`: Google Sheets formula to embed thumbnail image using `thumbnailLink`  
      - Cell format: USER_ENTERED  
      - Credentials: OAuth2 Google Sheets account  
    - Input: Metadata from Edit Fields1 and original prompt  
    - Output: Confirmation of row append  
    - Edge cases: Authentication failure, sheet access issues, or invalid formula syntax.

---

#### 2.6 Cost Tracking

- **Overview:**  
  Aggregates token usage and estimated costs from the OpenAI API response and appends this data to a separate Google Sheets tab for cost tracking.

- **Nodes Involved:**  
  - Aggregate  
  - Google Sheets1

- **Node Details:**

  - **Aggregate**  
    - Type: `n8n-nodes-base.aggregate`  
    - Configuration: Aggregates all item data from the batch processing (collects usage info).  
    - Input: Output from Loop Over Items (processed images)  
    - Output: Aggregated data for cost calculation  
    - Edge cases: Empty input or aggregation errors.

  - **Google Sheets1**  
    - Type: `n8n-nodes-base.googleSheets`  
    - Configuration:  
      - Operation: Append row  
      - Document ID: Same Google Sheets document as above  
      - Sheet Name: Tab with gid=929800828 (named "usage")  
      - Columns mapped:  
        - `prompt`: from chat input  
        - `datetime`: formatted creation timestamp from HTTP Request response  
        - `input token`, `output token`: token counts from OpenAI usage data  
        - `input estimated price`, `output estimated price`, `total estimated price`: calculated costs based on token counts and fixed rates (input tokens at $10 per million, output tokens at $40 per million)  
      - Credentials: OAuth2 Google Sheets account  
    - Input: Aggregated usage data and prompt  
    - Output: Confirmation of row append  
    - Edge cases: Authentication failure, sheet access issues, or calculation errors.

---

#### 2.7 Manual Trigger (Optional)

- **Overview:**  
  A disabled manual trigger node for testing the workflow manually.

- **Nodes Involved:**  
  - When clicking 'Test workflow'

- **Node Details:**

  - **When clicking 'Test workflow'**  
    - Type: `n8n-nodes-base.manualTrigger`  
    - Configuration: Disabled by default  
    - Input: Manual trigger by user  
    - Output: Starts the workflow at HTTP Request node  
    - Edge cases: Disabled state prevents accidental triggering.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                      | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                  |
|---------------------------|----------------------------------|------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Receives chat input prompt          | External webhook             | HTTP Request               | Use Chat to input prompts for image generation                                                               |
| HTTP Request              | n8n-nodes-base.httpRequest       | Calls OpenAI image generation API  | When chat message received   | Split Out                  |                                                                                                              |
| Split Out                 | n8n-nodes-base.splitOut          | Splits image array into items      | HTTP Request                | Loop Over Items            | Process image array data with Loop; images uploaded to Drive and saved as rows in Sheet with links and thumbs |
| Loop Over Items           | n8n-nodes-base.splitInBatches    | Processes images one by one        | Split Out                   | Aggregate, Edit Fields-file_name |                                                                                                              |
| Edit Fields-file_name     | n8n-nodes-base.set               | Sets filename and base64 image     | Loop Over Items             | Convert to File            |                                                                                                              |
| Convert to File           | n8n-nodes-base.convertToFile     | Converts base64 to binary file     | Edit Fields-file_name       | Google Drive               |                                                                                                              |
| Google Drive              | n8n-nodes-base.googleDrive       | Uploads image file to Drive        | Convert to File             | Edit Fields1               |                                                                                                              |
| Edit Fields1              | n8n-nodes-base.set               | Prepares metadata for Sheets       | Google Drive                | Google Sheets              |                                                                                                              |
| Google Sheets             | n8n-nodes-base.googleSheets      | Appends image metadata to Sheet    | Edit Fields1                | Loop Over Items            |                                                                                                              |
| Aggregate                 | n8n-nodes-base.aggregate         | Aggregates usage data for cost     | Loop Over Items             | Google Sheets1             | After processing, save Cost to Sheet                                                                         |
| Google Sheets1            | n8n-nodes-base.googleSheets      | Appends cost tracking data         | Aggregate                   |                            |                                                                                                              |
| When clicking 'Test workflow' | n8n-nodes-base.manualTrigger   | Manual trigger for testing         |                             | HTTP Request               |                                                                                                              |
| Sticky Note               | n8n-nodes-base.stickyNote        | Notes and instructions             |                             |                            | Use Chat to input prompts for image generation                                                               |
| Sticky Note1              | n8n-nodes-base.stickyNote        | Notes on image processing loop     |                             |                            | Process image array data with Loop; images uploaded to Drive and saved as rows in Sheet with links and thumbs |
| Sticky Note2              | n8n-nodes-base.stickyNote        | Notes on cost saving                |                             |                            | After processing, save Cost to Sheet                                                                         |
| Sticky Note4              | n8n-nodes-base.stickyNote        | Creator contact info                |                             |                            | Created by darrell_tw_ with social links                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a `@n8n/n8n-nodes-langchain.chatTrigger` node named `When chat message received`.  
   - Configure webhook ID or leave default to receive chat input prompts.

2. **Add HTTP Request Node:**  
   - Add `HTTP Request` node connected from `When chat message received`.  
   - Set method to POST, URL to `https://api.openai.com/v1/images/generations`.  
   - Use OpenAI API credentials (OAuth2 or API key).  
   - Set body parameters:  
     - model: `gpt-image-1`  
     - prompt: `={{ $json.chatInput }}`  
     - output_format: `jpeg`  
     - quality: `low`  
     - output_compression: 80  
     - size: `1024x1024`  
     - n: 1  
     - moderation: `low`

3. **Add Split Out Node:**  
   - Add `Split Out` node connected from `HTTP Request`.  
   - Configure to split field `data` including binary data.

4. **Add Loop Over Items Node:**  
   - Add `Split In Batches` node named `Loop Over Items` connected from `Split Out`.  
   - Set batch size to 1.

5. **Add Set Node for Filename:**  
   - Add `Set` node named `Edit Fields-file_name` connected from `Loop Over Items`.  
   - Set fields:  
     - `file_name`: `={{ $now.format("yyyyMMddHHmmSSS") }}`  
     - `b64_json`: `={{ $json.b64_json }}`

6. **Add Convert to File Node:**  
   - Add `Convert to File` node connected from `Edit Fields-file_name`.  
   - Configure to convert `b64_json` to binary file with filename from `file_name`.

7. **Add Google Drive Node:**  
   - Add `Google Drive` node connected from `Convert to File`.  
   - Configure to upload binary file to a specific folder in "My Drive".  
   - Use Google Drive OAuth2 credentials.  
   - Set file name dynamically, e.g., `chatgpt_created_by_n8n_{{created_timestamp}}`.

8. **Add Set Node for Metadata:**  
   - Add `Set` node named `Edit Fields1` connected from `Google Drive`.  
   - Set fields:  
     - `id`, `webViewLink`, `thumbnailLink` from Google Drive response.  
     - `file_name` from `Edit Fields-file_name`.

9. **Add Google Sheets Node for Image Metadata:**  
   - Add `Google Sheets` node connected from `Edit Fields1`.  
   - Configure to append rows to a Google Sheets document (specify document ID and sheet tab).  
   - Map columns:  
     - `prompt`: from chat input  
     - `image`: `webViewLink`  
     - `image_thumb`: formula embedding thumbnail image using `thumbnailLink`.  
   - Use Google Sheets OAuth2 credentials.

10. **Connect Google Sheets output back to Loop Over Items:**  
    - Connect `Google Sheets` node output back to `Loop Over Items` to continue processing.

11. **Add Aggregate Node:**  
    - Add `Aggregate` node connected from `Loop Over Items`.  
    - Configure to aggregate all item data for usage and cost calculation.

12. **Add Google Sheets Node for Cost Tracking:**  
    - Add `Google Sheets` node named `Google Sheets1` connected from `Aggregate`.  
    - Configure to append rows to a separate sheet tab for usage tracking.  
    - Map columns:  
      - `prompt` from chat input  
      - `datetime` formatted from HTTP Request creation time  
      - `input token`, `output token` from OpenAI usage data  
      - `input estimated price`, `output estimated price`, `total estimated price` calculated based on token counts and fixed rates.  
    - Use Google Sheets OAuth2 credentials.

13. **(Optional) Add Manual Trigger Node:**  
    - Add `Manual Trigger` node named `When clicking 'Test workflow'` for manual testing.  
    - Connect its output to `HTTP Request` node.  
    - Keep disabled by default.

14. **Add Sticky Notes:**  
    - Add sticky notes with instructions and tips as per original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Created by darrell_tw_, an engineer focused on AI and Automation                                | [X](https://x.com/darrell_tw_), [Threads](https://www.threads.net/@darrell_tw_), [Instagram](https://www.instagram.com/darrell_tw_/), [Website](https://www.darrelltw.com/) |
| Use Chat to input prompts for image generation                                                  | Sticky note inside workflow                                                                        |
| Process image array data with Loop; images uploaded to Drive and saved as rows in Sheet with links and thumbnails | Sticky note inside workflow                                                                        |
| After processing, save Cost to Sheet                                                           | Sticky note inside workflow                                                                        |
| Setup steps: Connect OpenAI, Google Drive, Google Sheets credentials; set folder and sheet IDs; pre-configured field mappings; setup takes 5-10 minutes | Workflow description                                                                               |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the "Generate Images with GPT-image-1 and Store in Google Drive with Cost Tracking" workflow. It covers all nodes, their configurations, data flow, and potential failure points.