Text-to-Image Generator with Google Sheets & Drive Integration using Flux AI

https://n8nworkflows.xyz/workflows/text-to-image-generator-with-google-sheets---drive-integration-using-flux-ai-5900


# Text-to-Image Generator with Google Sheets & Drive Integration using Flux AI

### 1. Workflow Overview

This workflow automates the generation of images from text prompts stored in a Google Sheet using the Flux AI Text-to-Image API. It is designed for users who want to bulk-generate images from textual descriptions without manual effort, integrating Google Sheets for prompt management and Google Drive for image storage.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initialization:** Starts manually and reads prompts from a Google Sheet.
- **1.2 Batch Processing & Filtering:** Processes prompts in batches, filtering to avoid reprocessing.
- **1.3 AI Image Generation & Conversion:** Sends prompts to Flux AI, receives base64 images, converts to binary.
- **1.4 Data Logging & Upload:** Logs generated image data back to Google Sheets, uploads images to Google Drive.
- **1.5 Error Handling & Delay:** Handles API errors by updating the sheet, and includes wait periods to throttle API calls.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:** This block starts the workflow manually and reads the list of text prompts from a Google Sheet that acts as the data source.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `Google Sheets2`
- **Node Details:**

  1. **When clicking ‘Execute workflow’**
     - Type: Manual Trigger
     - Role: Starts the workflow on demand.
     - Configuration: No parameters; simply waits for manual execution.
     - Input: None
     - Output: Triggers the next node.
     - Edge Cases: None; manual trigger.

  2. **Google Sheets2**
     - Type: Google Sheets (Read)
     - Role: Reads the prompts from the first sheet (gid=0) of a specified Google Sheets document.
     - Configuration:
       - Document ID: URL mode (must be set to the target Google Sheet URL).
       - Sheet Name: "Sheet1"
       - Authentication: Service Account credentials.
     - Input: Trigger from manual node.
     - Output: Sends rows of prompt data downstream.
     - Edge Cases:
       - Failure if document ID is missing or authentication fails.
       - Empty sheet returns no data, halting further processing.

#### 1.2 Batch Processing & Filtering

- **Overview:** Processes the prompts in manageable batches and filters to process only those prompts that have not yet been processed (i.e., no existing image in Drive).
- **Nodes Involved:** `Loop Over Items`, `If2`
- **Node Details:**

  1. **Loop Over Items**
     - Type: Split In Batches
     - Role: Processes each prompt individually or in batches to avoid API overload.
     - Configuration:
       - Default batch size (not explicitly set).
       - Reset option disabled (does not reset batch index).
     - Input: Rows from `Google Sheets2`.
     - Output: Passes batches to conditional check.
     - Edge Cases:
       - Large datasets require appropriate batch size adjustment.
       - Processing stops if input is empty.

  2. **If2**
     - Type: If (Condition)
     - Role: Checks two conditions:
       - The prompt field is not empty.
       - The `drive path` field is empty (meaning no image generated yet).
     - Configuration:
       - Left Value: `{{$json.Prompt}}` not empty.
       - Left Value: `{{$json['drive path']}}` empty.
     - Input: Batches from `Loop Over Items`.
     - Output:
       - True branch: proceed to generate image.
       - False branch: skip processing, return to batch loop.
     - Edge Cases:
       - Missing or malformed fields may cause condition evaluation errors.
       - Case sensitive and strict validation ensures exact matches.
  
#### 1.3 AI Image Generation & Conversion

- **Overview:** Sends eligible prompts to the Flux AI Text-to-Image API, receives base64 encoded images, and converts them into binary files for later upload.
- **Nodes Involved:** `HTTP Request1`, `If1`, `Code1`
- **Node Details:**

  1. **HTTP Request1**
     - Type: HTTP Request (POST)
     - Role: Calls the Flux AI API to generate images based on the prompt.
     - Configuration:
       - URL: `https://text-to-image-flux-ai.p.rapidapi.com/flux.php`
       - Method: POST
       - Headers:
         - `x-rapidapi-host`: `text-to-image-flux-ai.p.rapidapi.co`
         - `x-rapidapi-key`: **User must provide their API key**
       - Body: JSON containing `"prompt": {{$json.Prompt}}`
       - On Error: Continue to error output to allow error handling downstream.
     - Input: Prompts passing `If2` condition.
     - Output:
       - Main (success): base64 image data.
       - Error: routed to error handler.
     - Edge Cases:
       - Network errors, authentication failures (invalid API key).
       - API rate limiting.
       - Unexpected API response format.
  
  2. **If1**
     - Type: If (Error Check)
     - Role: Checks if the API response contains an `error` field.
     - Configuration:
       - Condition: `{{$json.error}}` exists.
     - Input: Response from `HTTP Request1`.
     - Output:
       - True: route to error logging.
       - False: continue to image processing.
     - Edge Cases:
       - Partial or malformed error responses.
  
  3. **Code1**
     - Type: Code (JavaScript)
     - Role: Converts the base64 encoded image string into a binary buffer suitable for file upload.
     - Configuration:
       - Extracts `image_base64` from input JSON.
       - Removes data URI prefix if present.
       - Converts base64 string to a Buffer.
       - Outputs binary data with file metadata (`mimeType: image/jpeg`, `fileName: output.jpg`).
     - Input: Successful API response.
     - Output: Binary data for downstream nodes.
     - Edge Cases:
       - Missing or invalid base64 string.
       - Incorrect MIME type assumptions.

#### 1.4 Data Logging & Upload

- **Overview:** Stores the generated image data in Google Sheets for logging and uploads the binary image file to Google Drive.
- **Nodes Involved:** `Google Sheets1`, `Google Drive1`, `Google Sheets4`
- **Node Details:**

  1. **Google Sheets1**
     - Type: Google Sheets (Append)
     - Role: Appends a new row with `Prompt`, base64 image data, and generation timestamp.
     - Configuration:
       - Document ID and Sheet Name same as reading node.
       - Columns: `Prompt`, `Base64`, `Generated Date`.
       - Authentication: Service Account.
     - Input: Binary data from `Code1` with JSON fields.
     - Output: Triggers next upload node.
     - Edge Cases:
       - Append failure due to permissions or quota.
       - Data size limits on base64 strings.
  
  2. **Google Drive1**
     - Type: Google Drive (Upload)
     - Role: Uploads the binary image file to a specified Google Drive folder.
     - Configuration:
       - File name from binary file metadata.
       - Drive ID and Folder ID: must be set by user.
       - Authentication: Service Account.
     - Input: Binary image from `Code1`.
     - Output: Triggers wait node.
     - Edge Cases:
       - Upload failures due to permissions or quota.
       - Folder ID or Drive ID misconfiguration.
  
  3. **Google Sheets4**
     - Type: Google Sheets (Append or Update)
     - Role: Updates the Google Sheet with error details if image generation fails.
     - Configuration:
       - Uses `Prompt` as matching column.
       - Updates `Base64` column with error message.
       - Document ID and sheet same as above.
     - Input: Errors from `If1` node.
     - Output: Loops back to wait node.
     - Edge Cases:
       - Failure to update sheet on error.
       - Multiple errors for same prompt.

#### 1.5 Error Handling & Delay

- **Overview:** Manages error routing and introduces wait periods between batch processing to prevent API rate limits.
- **Nodes Involved:** `Wait`
- **Node Details:**

  1. **Wait**
     - Type: Wait
     - Role: Pauses workflow execution for 10 seconds between batches.
     - Configuration:
       - Amount: 10 seconds.
     - Input: Triggered after image upload or error logging.
     - Output: Returns to batch loop node.
     - Edge Cases:
       - Wait node failure is rare but could stall workflow.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                      | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                      |
|------------------------------|---------------------|------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Starts the workflow manually       | None                            | Google Sheets2                  | Starts the workflow manually when you click “Execute workflow.” Initiates the entire process.                   |
| Google Sheets2               | Google Sheets (Read) | Reads text prompts from Google Sheet | When clicking ‘Execute workflow’ | Loop Over Items                 | Reads the list of text prompts from a Google Sheet. Fetches data that drives the image generation process.      |
| Loop Over Items              | Split In Batches     | Processes prompts in batches       | Google Sheets2                  | If2 (true branch), Loop Over Items (false branch) | Processes each prompt individually in manageable batches to avoid API overload. Allows sequential/batch processing.|
| If2                         | If                   | Checks prompt presence & drive path | Loop Over Items                 | HTTP Request1 (true), Loop Over Items (false) | Checks if prompt is present and image not already generated (drive path empty). Prevents duplicate processing.    |
| HTTP Request1               | HTTP Request (POST)   | Calls Flux AI API to generate image | If2                            | Code1 (success), If1 (error)    | Sends prompt to Text To Image Flux AI API; receives base64 encoded image. Calls external AI service.             |
| If1                         | If                   | Checks for API response errors     | HTTP Request1                  | Google Sheets4 (error), Code1 (no error) | Checks if API response contains an error; routes flow accordingly.                                               |
| Code1                       | Code (JavaScript)    | Converts base64 image to binary    | HTTP Request1                  | Google Sheets1, Google Drive1   | Converts base64 image string into binary format suitable for upload. Prepares image data for Google Drive upload.|
| Google Sheets1              | Google Sheets (Append) | Logs generated image base64 and timestamp | Code1                         | Wait                          | Appends generated base64 image data and timestamp to Google Sheet for tracking.                                 |
| Google Drive1               | Google Drive (Upload) | Uploads binary image to Drive      | Code1                         | Wait                          | Uploads generated image file to specified Google Drive folder. Stores images for access and sharing.            |
| Google Sheets4              | Google Sheets (AppendOrUpdate) | Logs errors in Google Sheet       | If1 (error branch)             | Wait                          | Updates Google Sheet with error info if image generation fails. Tracks failures for debugging or reprocessing.  |
| Wait                       | Wait                 | Pauses workflow to avoid API overload | Google Drive1, Google Sheets1, Google Sheets4 | Loop Over Items               | Adds 10-second pause between batches to prevent API rate limits or overload.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: `When clicking ‘Execute workflow’`
   - Type: Manual Trigger
   - No special configuration.

2. **Add Google Sheets Node to Read Prompts**
   - Name: `Google Sheets2`
   - Type: Google Sheets (Read)
   - Set Document ID to your target Google Sheet URL.
   - Sheet Name: `Sheet1` (gid=0)
   - Authentication: Use a Service Account with access to the sheet.
   - Connect output from Manual Trigger.

3. **Add Split In Batches Node**
   - Name: `Loop Over Items`
   - Type: Split In Batches
   - Default batch size (optional: customize for performance).
   - Connect input from Google Sheets2.

4. **Add If Node for Filtering**
   - Name: `If2`
   - Type: If
   - Condition 1: `{{$json.Prompt}}` is not empty.
   - Condition 2: `{{$json['drive path']}}` is empty.
   - Connect true output to next step, false output back to Loop Over Items (to skip processed prompts).

5. **Add HTTP Request Node for Image Generation**
   - Name: `HTTP Request1`
   - Type: HTTP Request (POST)
   - URL: `https://text-to-image-flux-ai.p.rapidapi.com/flux.php`
   - Headers:
     - `x-rapidapi-host`: `text-to-image-flux-ai.p.rapidapi.co`
     - `x-rapidapi-key`: **Your RapidAPI Key here**
   - Body Parameters: JSON with `"prompt": {{$json.Prompt}}`
   - On Error: Continue error output.
   - Connect input from If2 (true branch).

6. **Add If Node for Error Checking**
   - Name: `If1`
   - Type: If
   - Condition: `{{$json.error}}` exists.
   - Connect success output to Code node, error output to Google Sheets error logging.

7. **Add Code Node to Convert Base64**
   - Name: `Code1`
   - Type: Code (JavaScript)
   - Paste code that extracts base64 from `$input.first().json.image_base64`, strips data URI prefix if present, converts to binary buffer.
   - Set output binary with:
     - `mimeType`: `image/jpeg`
     - `fileName`: `output.jpg`
   - Connect input from HTTP Request1 (success branch).

8. **Add Google Sheets Node to Log Image Data**
   - Name: `Google Sheets1`
   - Type: Google Sheets (Append)
   - Document ID and Sheet Name: same as reading node.
   - Append columns: `Prompt`, `Base64` (from HTTP Request), `Generated Date` (use expression `$now`).
   - Authentication: Service Account.
   - Connect input from Code1.

9. **Add Google Drive Node to Upload Image**
   - Name: `Google Drive1`
   - Type: Google Drive (Upload)
   - Set Drive and Folder IDs to target upload location.
   - File Name: from binary file metadata.
   - Authentication: Service Account.
   - Connect input from Code1.

10. **Add Google Sheets Node to Log Errors**
    - Name: `Google Sheets4`
    - Type: Google Sheets (AppendOrUpdate)
    - Document ID and Sheet Name: same as above.
    - Matching Column: `Prompt`
    - Columns to update: `Base64` with `{{$json.error}}` message.
    - Authentication: Service Account.
    - Connect input from If1 (error branch).

11. **Add Wait Node**
    - Name: `Wait`
    - Type: Wait
    - Duration: 10 seconds.
    - Connect inputs from Google Sheets1, Google Drive1, and Google Sheets4.
    - Output connects back to `Loop Over Items` to process next batch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow automates image generation from text prompts stored in Google Sheets, integrates with Flux AI API, stores images in Google Drive, and logs results and errors back to Sheets.                                                                                                                     | Project description embedded in workflow sticky notes.                                                       |
| Ensure you replace `"your key"` in HTTP Request1 with a valid RapidAPI key registered for the Text To Image Flux AI service.                                                                                                                                                                                 | API key required for external service authentication.                                                        |
| Google Service Account credentials must have access to both Google Sheets and Google Drive APIs with appropriate permissions.                                                                                                                                                                                | Credential setup required in n8n for Google API nodes.                                                       |
| The workflow includes error handling for API failures and rate limiting via the Wait node to ensure robustness.                                                                                                                                                                                              | Best practice for integrating rate-limited external APIs.                                                    |
| For large datasets, consider adjusting batch sizes in the Split In Batches node to optimize processing time and resource usage.                                                                                                                                                                              | Performance tuning note.                                                                                       |
| Flux AI’s API base64 image response format is expected; if the API response changes, Code1 node needs adjustment accordingly.                                                                                                                                                                                 | Maintenance note for API response format changes.                                                             |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.