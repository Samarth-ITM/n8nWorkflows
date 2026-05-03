Text-to-Image Generation with Flux AI, Google Drive Storage & Sheets Logging

https://n8nworkflows.xyz/workflows/text-to-image-generation-with-flux-ai--google-drive-storage---sheets-logging-5929


# Text-to-Image Generation with Flux AI, Google Drive Storage & Sheets Logging

### 1. Workflow Overview

This workflow automates the generation of images from user-submitted text prompts using the Flux AI Text-to-Image Generator API. It covers the entire process from receiving user input via a form, generating an image from the prompt, converting and storing the image in Google Drive, and logging the prompt and image metadata in Google Sheets. It also includes error detection and logging mechanisms.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception (Form Submission):** Captures the user's text prompt input.
- **1.2 AI Image Generation (HTTP Request to Flux AI API):** Sends the prompt to the Flux AI API and receives a base64-encoded image.
- **1.3 Image Processing (Base64 Decoding):** Converts the base64 image string into a binary image file.
- **1.4 Storage (Google Drive Upload):** Uploads the generated image file to a specified Google Drive folder.
- **1.5 Logging (Google Sheets – Success Log):** Records the prompt, image filename, and generation date in a Google Sheet.
- **1.6 Error Handling (If Node and Google Sheets – Error Log):** Detects generation errors and logs failed prompts and error messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (Form Submission)

- **Overview:**  
  This block triggers the workflow by capturing user input from a web form titled "Text To Image Flux AI". It collects a required textarea input named "Prompt" containing the description for image generation.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **Node Name:** On form submission  
  - **Type:** Form Trigger  
  - **Configuration:**  
    - Webhook ID set to listen for form submissions  
    - Form titled "Text To Image Flux AI"  
    - Single required textarea field labeled "Prompt" with a detailed placeholder example describing a complex image prompt  
  - **Expressions/Variables:** Captures user input as `$json.Prompt`  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Connects to HTTP Request node  
  - **Version Requirements:** n8n version supporting formTrigger node, v2.2+  
  - **Edge Cases:** Missing prompt is prevented by required field enforcement; malformed input is passed downstream for validation  
  - **Sub-Workflow:** None

#### 2.2 AI Image Generation (HTTP Request to Flux AI API)

- **Overview:**  
  Sends the user prompt to the Flux AI Text-to-Image Generator API via RapidAPI, requesting image generation. Receives a base64-encoded image response or an error.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **Node Name:** HTTP Request  
  - **Type:** HTTP Request (POST)  
  - **Configuration:**  
    - URL: `https://text-to-image-generator-flux.p.rapidapi.com/flux.php`  
    - Headers include `x-rapidapi-host` and `x-rapidapi-key` (API key to be set securely)  
    - Body parameter includes prompt taken dynamically via expression `={{ $json.Prompt }}`  
    - On error, the node continues execution to allow error handling downstream  
  - **Expressions/Variables:** Sends prompt as body parameter  
  - **Input Connections:** From "On form submission"  
  - **Output Connections:** Two outputs:  
    - Success output to "Code" node for processing image  
    - Error or any output to "If" node for error checking  
  - **Version Requirements:** n8n version supporting HTTP Request v4.2+  
  - **Edge Cases:**  
    - API key invalid or rate limited (authentication errors)  
    - Network timeouts  
    - Malformed prompt causing API to return error field  
  - **Sub-Workflow:** None

#### 2.3 Image Processing (Base64 Decoding)

- **Overview:**  
  Converts the base64-encoded image string from the API response into a binary image file suitable for upload and further processing.

- **Nodes Involved:**  
  - Code

- **Node Details:**  
  - **Node Name:** Code  
  - **Type:** Code (JavaScript)  
  - **Configuration:**  
    - Extracts `image_base64` from input JSON  
    - Removes any data URI prefix if present, isolating the pure base64 string  
    - Converts base64 string to a Buffer object representing binary data  
    - Sets mime type as `image/jpeg` and file name as `output.jpg`  
  - **Expressions/Variables:** Uses `$input.first().json.image_base64`  
  - **Input Connections:** From HTTP Request (success output)  
  - **Output Connections:** To Google Sheets (logging) and Google Drive (upload) nodes  
  - **Version Requirements:** n8n supporting Code node v2  
  - **Edge Cases:**  
    - Missing or malformed base64 string  
    - Unexpected image format (currently hardcoded to jpeg)  
  - **Sub-Workflow:** None

#### 2.4 Storage (Google Drive Upload)

- **Overview:**  
  Uploads the decoded binary image file to a specified Google Drive folder for secure storage and sharing.

- **Nodes Involved:**  
  - Google Drive

- **Node Details:**  
  - **Node Name:** Google Drive  
  - **Type:** Google Drive  
  - **Configuration:**  
    - File name dynamically set as `okoutput.jpg` (prefix "ok" + filename from Code node)  
    - Drive ID and Folder ID to be set by user (via URL mode)  
    - Authentication using Google Service Account credentials  
  - **Expressions/Variables:** Uses `$binary.data.fileName` for naming  
  - **Input Connections:** From Code node  
  - **Output Connections:** None (end of branch)  
  - **Version Requirements:** Google Drive node v3+ with service account auth  
  - **Edge Cases:**  
    - Invalid or expired Google credentials  
    - Folder ID not accessible or missing  
    - Upload failures due to file size or network issues  
  - **Sub-Workflow:** None

#### 2.5 Logging (Google Sheets – Success Log)

- **Overview:**  
  Logs the successful image generation event by appending a row with the user prompt, image filename, and generation date to a Google Sheet.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Node Name:** Google Sheets  
  - **Type:** Google Sheets  
  - **Configuration:**  
    - Operation: Append row  
    - Sheet name: "Sheet1" (gid=0)  
    - Document ID: to be set by user (Google Sheets URL)  
    - Columns mapped:  
      - "Prompt" from form submission (`$('On form submission').item.json.Prompt`)  
      - "Image" from binary file name (`$binary.data.fileName`)  
      - "Generated Date" from form submission timestamp (`$('On form submission').item.json.submittedAt`)  
    - Authentication: Service Account via Google API credentials  
  - **Expressions/Variables:** Referenced as above  
  - **Input Connections:** From Code node  
  - **Output Connections:** None (end of branch)  
  - **Version Requirements:** Google Sheets node v4.6+ supporting service accounts  
  - **Edge Cases:**  
    - Invalid document ID or sheet name  
    - Authentication errors  
    - Append failures if schema mismatched  
  - **Sub-Workflow:** None

#### 2.6 Error Handling (If Node and Google Sheets – Error Log)

- **Overview:**  
  Detects if the API returned an error and, if so, logs the error details along with the prompt in a separate Google Sheet for troubleshooting.

- **Nodes Involved:**  
  - If  
  - Google Sheets5 (Error Log)

- **Node Details:**  

  - **Node Name:** If  
  - **Type:** If  
  - **Configuration:**  
    - Condition: Checks if `$json.error` exists (non-empty)  
  - **Input Connections:** From HTTP Request (error output)  
  - **Output Connections:** True output connects to "Google Sheets5" node  
  - **Edge Cases:**  
    - False negatives if error field is not properly set by API  
    - Workflow continues even if error is detected due to `continueErrorOutput` on HTTP Request  

  - **Node Name:** Google Sheets5  
  - **Type:** Google Sheets  
  - **Configuration:**  
    - Operation: Append or update row based on prompt matching  
    - Sheet name: "Sheet1" (gid=0)  
    - Document ID: to be set by user (Google Sheets URL)  
    - Columns mapped:  
      - "Prompt" from JSON  
      - "Base64" set to error message (`$json.error`)  
    - Authentication: Service Account via Google API credentials  
  - **Input Connections:** From If node (true output)  
  - **Output Connections:** None (end of branch)  
  - **Edge Cases:**  
    - Document or sheet access rights  
    - Authentication failures  
    - Append or update conflicts if prompt matching fails  
  - **Sub-Workflow:** None

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                     | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                                           |
|---------------------|-----------------------|-----------------------------------|-----------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger          | Captures user prompt input        | None                  | HTTP Request               | ### 1. 📝 On Form Submission - Accepts user input for a creative text prompt. - 🔍 Example prompt provided.            |
| HTTP Request        | HTTP Request           | Sends prompt to Flux AI API        | On form submission    | Code, If                   | ### 2. 🌐 HTTP Request — Flux AI API - Sends prompt to API via RapidAPI. - Returns base64 image.                       |
| Code                | Code                   | Decodes base64 image to binary    | HTTP Request          | Google Sheets, Google Drive | ### 3. 🧪 Code Node — Base64 Decoder - Converts base64 to .jpg binary file.                                            |
| Google Drive        | Google Drive            | Uploads image to Google Drive      | Code                  | None                       | ### 4. 📁 Google Drive - Uploads generated image to Drive.                                                             |
| Google Sheets       | Google Sheets           | Logs successful generation         | Code                  | None                       | ### 5. 📊 Google Sheets — Success Log - Logs prompt, filename, and date.                                               |
| If                  | If                      | Detects errors in API response     | HTTP Request          | Google Sheets5             | ### 6. ⚠️ IF Node — Error Detection - Checks for generation failure to route errors.                                   |
| Google Sheets5      | Google Sheets           | Logs errors for failed generation  | If                    | None                       | ### 7. 📉 Google Sheets — Error Log - Logs failed prompts and errors.                                                  |
| Sticky Note         | Sticky Note             | Documentation and overview         | None                  | None                       | # 🎨 AI Image Generator with Flux AI and Google integrations; API link included.                                       |
| Sticky Note1        | Sticky Note             | Documentation for form submission  | None                  | None                       | ### 1. 📝 On Form Submission - No technical knowledge needed.                                                          |
| Sticky Note2        | Sticky Note             | Documentation for HTTP request     | None                  | None                       | ### 2. 🌐 HTTP Request — Flux AI API - Seamless integration with image generation API.                                 |
| Sticky Note3        | Sticky Note             | Documentation for code node        | None                  | None                       | ### 3. 🧪 Code Node — Base64 Decoder - Prepares image for upload.                                                       |
| Sticky Note4        | Sticky Note             | Documentation for Google Drive     | None                  | None                       | ### 4. 📁 Google Drive - Secure cloud storage.                                                                          |
| Sticky Note5        | Sticky Note             | Documentation for Google Sheets    | None                  | None                       | ### 5. 📊 Google Sheets — Success Log - Tracks all generated images.                                                   |
| Sticky Note6        | Sticky Note             | Documentation for IF node          | None                  | None                       | ### 6. ⚠️ IF Node — Error Detection - Prevents workflow halt on errors.                                                |
| Sticky Note7        | Sticky Note             | Documentation for error logging    | None                  | None                       | ### 7. 📉 Google Sheets — Error Log - Helps identify errors like malformed prompts.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure with webhook to listen for submissions  
   - Title form as "Text To Image Flux AI"  
   - Add one required textarea field named "Prompt" with a descriptive placeholder  
   - Save and activate webhook

2. **Add HTTP Request Node ("HTTP Request"):**  
   - Type: HTTP Request (POST)  
   - Set URL to `https://text-to-image-generator-flux.p.rapidapi.com/flux.php`  
   - Set headers:  
     - `x-rapidapi-host`: `text-to-image-generator-flux.p.rapidapi.com`  
     - `x-rapidapi-key`: *Your RapidAPI key* (store securely in credentials or environment variable)  
   - Set body parameters with key "prompt" and value expression `={{ $json.Prompt }}`  
   - Enable "Send Body" and "Send Headers" options  
   - Set error handling to "Continue on Error"  
   - Connect output 0 to next node for success, output 1 to error handling node

3. **Add Code Node ("Code"):**  
   - Type: Code (JavaScript)  
   - Paste code to extract base64 string, remove prefixes, convert to Buffer with mimeType "image/jpeg", and name file "output.jpg"  
   - Connect success output from HTTP Request to this node  
   - Output binary data for image file

4. **Add Google Drive Node ("Google Drive"):**  
   - Type: Google Drive  
   - Configure with Google Service Account credentials  
   - Set File Name to expression `=ok{{ $binary.data.fileName }}` (prefix “ok” plus filename)  
   - Set Drive ID and Folder ID according to your Google Drive structure (paste URLs in URL mode)  
   - Connect output of Code node to Google Drive node

5. **Add Google Sheets Node ("Google Sheets"):**  
   - Type: Google Sheets  
   - Configure with Google Service Account credentials  
   - Set Operation to "Append"  
   - Choose target Google Sheet and sheet name "Sheet1" (gid=0)  
   - Map columns:  
     - Prompt: `={{ $('On form submission').item.json.Prompt }}`  
     - Image: `={{ $binary.data.fileName }}`  
     - Generated Date: `={{ $('On form submission').item.json.submittedAt }}`  
   - Connect output of Code node to this node

6. **Add If Node ("If"):**  
   - Type: If  
   - Condition: Check if `$json.error` exists (non-empty)  
   - Connect error output of HTTP Request node to this If node

7. **Add Google Sheets Node for Error Logging ("Google Sheets5"):**  
   - Type: Google Sheets  
   - Configure with Google Service Account credentials  
   - Set Operation to "Append or Update" based on matching "Prompt" column  
   - Choose different Google Sheet or tab for error logs, sheet name "Sheet1" (gid=0)  
   - Map columns:  
     - Prompt: `={{ $json.Prompt }}`  
     - Base64 (used for error text): `={{ $json.error }}`  
   - Connect true output of If node to this node

8. **Add Sticky Note Nodes (Optional):**  
   - Add Sticky Note nodes containing documentation and overview text for each logical block  
   - Position them appropriately for visual clarity in the workflow editor

9. **Credentials Setup:**  
   - Create and configure Google Service Account credentials with access to Google Drive and Sheets  
   - Store RapidAPI key securely and reference in HTTP Request headers

10. **Testing and Activation:**  
    - Test the workflow by submitting a prompt via the form  
    - Verify image generation, upload to Google Drive, and logging in Google Sheets  
    - Trigger error scenarios by sending invalid prompts or disconnecting API credentials to test error logging

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Workflow utilizes the Flux AI Text-to-Image Generator API accessible via RapidAPI platform.                                                              | [Flux AI API on RapidAPI](https://rapidapi.com/skdeveloper/api/text-to-image-generator-flux)                                |
| The workflow handles errors gracefully by logging them without stopping execution, aiding troubleshooting.                                               | Error Handling section in workflow                                                                                          |
| Google Drive and Google Sheets are integrated using a Google Service Account for seamless cloud storage and logging without user intervention.           | Requires Google Cloud service account setup with Drive and Sheets API access                                               |
| Example prompt given in form placeholder illustrates complex descriptive text for rich image generation.                                                  | Input Reception block                                                                                                       |
| The image is always saved as a JPEG file named "output.jpg" prefixed with "ok" when uploaded to Google Drive.                                            | Image Processing and Storage blocks                                                                                        |
| All API keys and credentials should be securely stored and never exposed in plain text within the workflow.                                              | Security best practice                                                                                                      |
| Sticky notes in workflow provide inline documentation and overview, useful for maintainers and new users.                                                | Workflow documentation nodes                                                                                               |

---

**Disclaimer:** The text provided is exclusively derived from an n8n workflow automation using publicly available APIs and legal content. No illegal, offensive, or protected elements are included. All data handled is lawful and public.