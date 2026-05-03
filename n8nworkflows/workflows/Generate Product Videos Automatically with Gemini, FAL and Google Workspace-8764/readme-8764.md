Generate Product Videos Automatically with Gemini, FAL and Google Workspace

https://n8nworkflows.xyz/workflows/generate-product-videos-automatically-with-gemini--fal-and-google-workspace-8764


# Generate Product Videos Automatically with Gemini, FAL and Google Workspace

### 1. Workflow Overview

This workflow automates the generation of branded product advertisement videos based on new rows added to a Google Sheet. Upon detecting a new row containing an image link, it downloads the image, processes it through an AI image generation model (Gemini API) to create an ad-style variant, then converts that image into a short video clip using the FAL image-to-video API. The final video is uploaded to Google Drive, and its link is updated back into the original Google Sheet, completing a fully automated pipeline from spreadsheet entry to finished video asset.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Filtering:** Triggered by new rows in Google Sheets and filtering rows based on status.
- **1.2 Image Download and Processing:** Download the source image from Google Drive and prepare it for AI processing.
- **1.3 AI Image Generation:** Call Gemini API to generate a branded product image variant.
- **1.4 Image Upload:** Convert and upload the generated image back to Google Drive.
- **1.5 Video Creation via FAL:** Submit the generated image to FAL API for video creation, and poll until the video is ready.
- **1.6 Video Download and Upload:** Download the completed video and upload it to Google Drive.
- **1.7 Sheet Update:** Update the Google Sheet row with the video link and status.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception and Filtering

- **Overview:** This block listens for new rows added to a specific Google Sheet and filters them to process only rows with a status labeled "run".
- **Nodes Involved:** Google Sheets Trigger, Filter
- **Node Details:**

  - **Google Sheets Trigger**
    - Type: Trigger node for Google Sheets
    - Role: Detects new rows added to a configured Google Sheet.
    - Configuration: Polls every minute; listens on a specified sheet and document via IDs.
    - Inputs: None (trigger)
    - Outputs: Emits newly added row data.
    - Edge Cases: Potential OAuth token expiration; delayed polling interval may delay processing; missing or incorrect sheet/document IDs.

  - **Filter**
    - Type: Filter node
    - Role: Allows only rows where the `status` field equals "run" to proceed.
    - Configuration: Condition strictly compares `$json.status` to "run".
    - Inputs: From Google Sheets Trigger
    - Outputs: Only matching rows to downstream nodes.
    - Edge Cases: Status field missing or misspelled would block processing; case-sensitive comparison may cause mismatch.

---

#### Block 1.2: Image Download and Processing

- **Overview:** Downloads the product image from Google Drive based on the URL in the sheet, then extracts the binary data for further processing.
- **Nodes Involved:** Download file, Extract from File
- **Node Details:**

  - **Download file**
    - Type: Google Drive node
    - Role: Downloads the image file referenced by the URL in the sheet (`link_image`).
    - Configuration: Extracts fileId from the URL using regex; uses OAuth2 credentials; operation set to download.
    - Inputs: From Filter node (filtered row data)
    - Outputs: Binary data of the image file.
    - Edge Cases: Invalid or expired Google Drive link; permissions issues; network timeout; incorrect regex failing to extract file ID.

  - **Extract from File**
    - Type: Extract binary data node
    - Role: Converts the downloaded binary image file into a property (base64-encoded string) for API consumption.
    - Configuration: Operation set to binaryToProperty.
    - Inputs: Binary data from Download file node.
    - Outputs: JSON with base64-encoded image data.
    - Edge Cases: Corrupted file data; unsupported file types; large files causing performance issues.

---

#### Block 1.3: AI Image Generation with Gemini API

- **Overview:** Sends the base64 image and a prompt extracted from the sheet to the Gemini API to create an ad-style image variant.
- **Nodes Involved:** Create product image with model
- **Node Details:**

  - **Create product image with model**
    - Type: HTTP Request node
    - Role: Calls Gemini API to generate a new image based on input image and descriptive prompt.
    - Configuration:
      - URL: Gemini API endpoint for image content generation.
      - Method: POST with JSON body containing text prompt and inline base64 image data.
      - Headers: Includes `x-goog-api-key` from credentials.
      - Prompt text dynamically constructed from the `note` field in the Google Sheet.
    - Inputs: Output from Extract from File node.
    - Outputs: JSON response with generated image data.
    - Edge Cases: API key invalid or rate-limited; malformed request body; JSON expression errors in prompt construction; network errors; API downtime.

---

#### Block 1.4: Image Upload

- **Overview:** Converts the Gemini API response image data back to a binary file and uploads it to a designated Google Drive folder.
- **Nodes Involved:** Convert to File, Upload file
- **Node Details:**

  - **Convert to File**
    - Type: Convert to File node
    - Role: Converts inline base64 image data from Gemini API response into binary file format for upload.
    - Configuration: Source property set to `candidates[0].content.parts[0].inlineData.data` from the Gemini response.
    - Inputs: Gemini API response JSON.
    - Outputs: Binary file.
    - Edge Cases: Missing or malformed response data; base64 decode failures.

  - **Upload file**
    - Type: Google Drive node
    - Role: Uploads the binary image file to a specified images folder in Google Drive.
    - Configuration: Folder ID must be set for destination; uses OAuth2 credentials; names the file with the current timestamp.
    - Inputs: Binary file from Convert to File node.
    - Outputs: Metadata with Google Drive links (`webContentLink`).
    - Edge Cases: Invalid folder ID; upload failures; permission issues.

---

#### Block 1.5: Video Creation via FAL API and Polling

- **Overview:** Submits the uploaded image to the FAL image-to-video API to create a short video clip, then polls the API's response URL until the video generation is completed.
- **Nodes Involved:** Create video, Wait, Get Link Video
- **Node Details:**

  - **Create video**
    - Type: HTTP Request node
    - Role: Sends a POST request to FAL API with the uploaded image URL and prompt to generate a video.
    - Configuration:
      - URL: FAL image-to-video endpoint.
      - Headers: Authorization key from credentials.
      - Body: Includes `prompt` ("slow motion, " + `note` from filtered row) and `image_url` from uploaded image's `webContentLink`.
    - Inputs: Metadata from Upload file node.
    - Outputs: JSON with `response_url` to poll for video status.
    - Edge Cases: Invalid API key; malformed request; API rate limiting.

  - **Wait**
    - Type: Wait node
    - Role: Delays processing for 30 seconds between polls.
    - Configuration: Waits for 30 seconds.
    - Inputs: From Create video node.
    - Outputs: Triggers Get Link Video node.
    - Edge Cases: Fixed wait time may be inefficient; no built-in retry logic.

  - **Get Link Video**
    - Type: HTTP Request node
    - Role: Polls the FAL `response_url` to check video generation status.
    - Configuration:
      - URL dynamically set from `response_url`.
      - Headers: Authorization key from credentials.
    - Inputs: After Wait node.
    - Outputs: JSON containing video status and video URL when completed.
    - Edge Cases: Polling may receive errors; video may fail to generate; requires logic (not shown) to repeat polling if status not complete.

---

#### Block 1.6: Video Download and Upload

- **Overview:** Downloads the finished video from the FAL API response URL and uploads it to a designated Google Drive folder.
- **Nodes Involved:** Download video, Upload video
- **Node Details:**

  - **Download video**
    - Type: HTTP Request node
    - Role: Downloads the video file from the URL provided by FAL API.
    - Configuration: URL dynamically set from JSON path `video.url`.
    - Inputs: From Get Link Video node.
    - Outputs: Binary video data.
    - Edge Cases: Video URL invalid or inaccessible; timeout; large file size.

  - **Upload video**
    - Type: Google Drive node
    - Role: Uploads the binary video to a specified Google Drive folder for videos.
    - Configuration: Folder ID set for destination; OAuth2 credentials; file named by timestamp.
    - Inputs: Binary video from Download video node.
    - Outputs: Metadata including `webViewLink`.
    - Edge Cases: Upload failure; permission issues.

---

#### Block 1.7: Sheet Update

- **Overview:** Updates the original Google Sheet row with the status "finished" and adds the link to the uploaded video.
- **Nodes Involved:** Update row in sheet
- **Node Details:**

  - **Update row in sheet**
    - Type: Google Sheets node
    - Role: Updates the row matching the original `STT` with `status` set to "finished" and `link_video` set to the uploaded video's link.
    - Configuration:
      - Matches rows by `STT` column.
      - Writes back `status` = "finished", and the video URL from `webViewLink`.
      - Uses OAuth2 credentials.
    - Inputs: Metadata from Upload video node.
    - Outputs: None (final step).
    - Edge Cases: Row matching fails if `STT` is missing or duplicate; write failures due to permissions.

---

### 3. Summary Table

| Node Name                   | Node Type                     | Functional Role                         | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                                                     |
|-----------------------------|-------------------------------|---------------------------------------|--------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger        | Google Sheets Trigger          | Trigger workflow on new row addition  | None                     | Filter                    |                                                                                                                                |
| Filter                      | Filter                        | Filter rows with status = "run"       | Google Sheets Trigger    | Download file             |                                                                                                                                |
| Download file               | Google Drive                  | Download product image from Drive     | Filter                   | Extract from File         |                                                                                                                                |
| Extract from File           | Extract from File             | Convert binary image to base64 string | Download file            | Create product image with model |                                                                                                                                |
| Create product image with model | HTTP Request               | Call Gemini API for ad image generation | Extract from File        | Convert to File           |                                                                                                                                |
| Convert to File             | Convert to File               | Convert Gemini base64 image to binary | Create product image with model | Upload file               |                                                                                                                                |
| Upload file                 | Google Drive                  | Upload generated image to Drive       | Convert to File          | Create video              |                                                                                                                                |
| Create video                | HTTP Request                 | Submit image to FAL for video creation | Upload file              | Wait                      |                                                                                                                                |
| Wait                       | Wait                         | Wait 30 seconds before polling        | Create video             | Get Link Video            |                                                                                                                                |
| Get Link Video             | HTTP Request                 | Poll FAL API for video generation status | Wait                     | Download video            |                                                                                                                                |
| Download video             | HTTP Request                 | Download generated video file          | Get Link Video           | Upload video              |                                                                                                                                |
| Upload video               | Google Drive                  | Upload video to Drive                  | Download video           | Update row in sheet       |                                                                                                                                |
| Update row in sheet         | Google Sheets                | Update sheet row with video link/status | Upload video            | None                      |                                                                                                                                |
| Sticky Note — Overview      | Sticky Note                  | Workflow overview and purpose         | None                     | None                      | See content in sticky note section 1                                                                                          |
| Sticky Note — Setup         | Sticky Note                  | Setup checklist and critical configs  | None                     | None                      | See content in sticky note section 2                                                                                          |
| Sticky Note — Notes & Tips  | Sticky Note                  | Notes on security, polling, error handling | None                  | None                      | See content in sticky note section 3                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a `Google Sheets Trigger` node:**
   - Set event to "rowAdded".
   - Configure polling to run every minute.
   - Specify Google Sheet document ID and sheet name.
   - Connect Google Sheets Trigger OAuth2 credentials.

2. **Add a `Filter` node:**
   - Connect from Google Sheets Trigger.
   - Set condition: `$json.status` equals "run" (case-sensitive).
   - Only rows matching proceed further.

3. **Add a `Google Drive` node named "Download file":**
   - Operation: Download.
   - File ID: Extract from the `link_image` URL using regex to parse Drive file ID.
   - Connect Google Drive OAuth2 credentials.
   - Connect input from Filter node.

4. **Add `Extract from File` node:**
   - Operation: binaryToProperty.
   - Input from "Download file".
   - Converts binary image to base64 string.

5. **Add an `HTTP Request` node "Create product image with model":**
   - Method: POST.
   - URL: Gemini API endpoint for image content generation.
   - Headers: Add header `x-goog-api-key` with your Gemini API key from credentials.
   - Body (JSON):
     - Construct a payload with a text prompt: `"Create a image with model and product attached to create ads videos with this require: {{ note from filtered row }}"`.
     - Attach image data as base64 from the extracted property.
   - Input: Output from Extract from File node.

6. **Add a `Convert to File` node:**
   - Source property: Path to Gemini API base64 image data (`candidates[0].content.parts[0].inlineData.data`).
   - Input from Gemini HTTP request output.

7. **Add a `Google Drive` node "Upload file":**
   - Operation: Upload.
   - Destination folder ID: Set folder ID for images.
   - File name: Use current timestamp (`{{$now}}`).
   - Credentials: Google Drive OAuth2.
   - Input from Convert to File node.

8. **Add an `HTTP Request` node "Create video":**
   - Method: POST.
   - URL: FAL API image-to-video endpoint.
   - Headers: Authorization with your FAL API key from credentials.
   - Body (form or JSON):
     - `prompt`: `"slow motion, {{ note from filtered row }}"`.
     - `image_url`: Use `webContentLink` of uploaded image from Google Drive node.
   - Input: Output from Upload file node.

9. **Add a `Wait` node:**
   - Duration: 30 seconds.
   - Input from Create video node.

10. **Add an `HTTP Request` node "Get Link Video":**
    - Method: GET.
    - URL: Use `response_url` from Create video node output.
    - Headers: Authorization with FAL API key.
    - Input: From Wait node.

11. **Add an `HTTP Request` node "Download video":**
    - Method: GET.
    - URL: From `video.url` in Get Link Video response.
    - Input: From Get Link Video node.

12. **Add a `Google Drive` node "Upload video":**
    - Operation: Upload.
    - Folder ID: Set folder ID for videos.
    - File name: Use current timestamp.
    - Credentials: Google Drive OAuth2.
    - Input: Binary from Download video node.

13. **Add a `Google Sheets` node "Update row in sheet":**
    - Operation: Update.
    - Document ID and sheet name same as trigger.
    - Match by column `STT`.
    - Set columns:
      - `status`: "finished"
      - `link_video`: `webViewLink` from Upload video node.
    - Credentials: Google Sheets OAuth2.
    - Input: From Upload video node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Keep API keys (`x-goog-api-key` for Gemini and `Authorization` for FAL) securely stored in n8n credentials, never hardcode.      | Security best practice                                                                           |
| Poll FAL API `response_url` until video `status` is `completed` before downloading video to avoid errors or incomplete files.    | API usage note                                                                                   |
| If APIs require direct file URLs, use Google Drive direct download format: `uc?export=download&id=<FILE_ID>`.                     | Google Drive file access                                                                         |
| Customize Gemini and FAL prompts to match brand voice or creative requirements.                                                   | Creative flexibility                                                                             |
| Add error handling with `If`, `Wait`, and retry nodes; log failures to the sheet by updating `status` and `note` columns.        | Reliability and monitoring                                                                       |
| This workflow supports batch processing by pasting multiple rows with appropriate `status = run` and `link_image` URLs.           | Scalability                                                                                    |
| Google Sheets minimum columns: `STT` (identifier), `link_image`, `link_video`, `status`, and `note`.                             | Data schema requirements                                                                        |
| Video and image upload folders must exist in Google Drive with correct folder IDs configured in Upload nodes.                     | Google Drive configuration                                                                      |

---

**Disclaimer:** The text provided originates exclusively from an automated n8n workflow. The process adheres strictly to content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.