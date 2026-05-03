Classify Event Photos from Attendees with Gemma AI, Google Drive & Sheets

https://n8nworkflows.xyz/workflows/classify-event-photos-from-attendees-with-gemma-ai--google-drive---sheets-6575


# Classify Event Photos from Attendees with Gemma AI, Google Drive & Sheets

### 1. Workflow Overview

This workflow automates the classification and categorization of event photos uploaded by attendees via a web form. It targets event organizers who want to efficiently collect, organize, and tag photos contributed by guests for easier management and sharing. The workflow integrates AI-powered image classification, image optimization, cloud storage, and data logging in a spreadsheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures photo uploads and user information via an n8n Form Trigger.
- **1.2 Image Preparation:** Extracts files, obtains metadata, resizes images for efficient processing, and converts images to base64 format required for AI inference.
- **1.3 AI Processing:** Sends images to the Featherless.ai API with Google’s Gemma multimodal model to classify photos into predefined categories and extracts the resulting tags.
- **1.4 Cloud Storage:** Uploads the original photos to a specific Google Drive folder for archival and sharing.
- **1.5 Data Logging:** Combines the classification data and storage links, then appends this information to a Google Sheets table to allow filtering, sorting, and later use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving photo uploads and the uploader's name from guests through a user-friendly form interface. It supports multiple file uploads and is optimized for mobile use.

- **Nodes Involved:**  
  - On form submission  
  - Files to Items

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Starts the workflow on form submission.  
    - Configuration:  
      - Form title: "Photo Uploads"  
      - Fields: "Name" (required text), "Files" (required file input accepting jpg, png)  
      - Button label: "Upload"  
      - Ignoring bot submissions enabled  
    - Inputs: None (trigger node)  
    - Outputs: Captures form data including files as binary data  
    - Edge cases: Form submission without required fields; unsupported file formats; bot submissions (filtered)  
    - Sticky note explains the ease of using the form trigger and mobile compatibility.

  - **Files to Items**  
    - Type: SplitOut  
    - Role: Splits the multiple uploaded files into individual workflow items for parallel processing.  
    - Configuration: Splitting based on binary file field.  
    - Inputs: Output from form submission  
    - Outputs: One item per file  
    - Edge cases: Empty or corrupted files; large file numbers impacting throughput.

#### 2.2 Image Preparation

- **Overview:**  
  This block processes each image file individually by extracting metadata, resizing images to a max width of 512 pixels to optimize AI processing speed, and converting images to base64 strings for API compatibility.

- **Nodes Involved:**  
  - Get File Meta  
  - Get Image Info  
  - Resize Image  
  - Convert Image to Base64

- **Node Details:**

  - **Get File Meta**  
    - Type: Set  
    - Role: Extracts JSON metadata from the binary file data.  
    - Configuration: Raw JSON output from binary property (dynamic).  
    - Inputs: Individual file items from Files to Items  
    - Outputs: JSON metadata for further use  
    - Edge cases: Missing or malformed binary data.

  - **Get Image Info**  
    - Type: Edit Image  
    - Role: Retrieves image dimensions and properties.  
    - Configuration: Operation set to "information" on each file.  
    - Inputs: From Get File Meta (via Files to Items)  
    - Outputs: Image metadata including size, used for resizing decision  
    - Edge cases: Unsupported image formats; corrupted images.

  - **Resize Image**  
    - Type: Edit Image  
    - Role: Resizes images to a width of max 512 px, maintaining aspect ratio.  
    - Configuration:  
      - Width: If original width > 512, resize to 512, else keep original width  
      - Height: Scaled proportionally  
      - Input binary data dynamically referenced per item  
    - Inputs: From Get Image Info  
    - Outputs: Resized image binary data  
    - Edge cases: Extremely small or zero dimension images; processing failures.

  - **Convert Image to Base64**  
    - Type: Extract From File  
    - Role: Converts binary image data to base64 strings required for AI API.  
    - Configuration: Converts binary property dynamically per item  
    - Inputs: From Resize Image  
    - Outputs: JSON with base64 encoded image data, mimeType  
    - Edge cases: Conversion errors; large image sizes causing delays.

- **Sticky note details:** Emphasizes resizing images to optimize AI processing speed and concurrency.

#### 2.3 AI Processing

- **Overview:**  
  Sends the base64-encoded images to Featherless.ai’s API using the Google Gemma multimodal model to classify photos into a fixed list of event-related categories. The output is parsed into arrays of category tags.

- **Nodes Involved:**  
  - Classify Photo and Suggest Tags  
  - Extract Categories

- **Node Details:**

  - **Classify Photo and Suggest Tags**  
    - Type: HTTP Request  
    - Role: Performs AI inference by calling Featherless.ai's chat completion endpoint.  
    - Configuration:  
      - URL: https://api.featherless.ai/v1/chat/completions  
      - Method: POST  
      - Authentication: Predefined Featherless API credential and HTTP header auth for API key  
      - JSON body: Sends a message array including the base64 image URL and instructions to classify into predefined categories (e.g., Group Shot, Candid, Smiles, etc.)  
      - Batching enabled (batch size 1, interval 3000ms) to respect API rate limits  
    - Inputs: Base64 images from Convert Image to Base64  
    - Outputs: AI response JSON with classification text  
    - Edge cases: API authentication failure; network timeouts; unexpected API responses; invalid base64 images.

  - **Extract Categories**  
    - Type: Set  
    - Role: Parses the comma-separated classification string from AI response into an array of categories.  
    - Configuration: Splits response content on commas, trims spaces  
    - Inputs: AI response from HTTP request  
    - Outputs: Array of categories for further merging  
    - Edge cases: Unexpected or empty classification strings; malformed JSON.

- **Sticky note details:** Describes the power of multimodal LLMs and Featherless.ai’s subscription model benefits.

#### 2.4 Cloud Storage

- **Overview:**  
  Uploads each original photo file to a designated Google Drive folder, naming files uniquely with uploader name, execution ID, and index to avoid collisions.

- **Nodes Involved:**  
  - Upload file

- **Node Details:**

  - **Upload file**  
    - Type: Google Drive  
    - Role: Uploads photo files to a specific folder in Google Drive for storage and sharing.  
    - Configuration:  
      - Folder: Fixed folder ID "1flK9aO20jw8Npf1EioiOl42i6FFLBGrX" labeled "99. Event Photos"  
      - Filename: Dynamically constructed "[Name]_[ExecutionID]_[Index].[extension]" to ensure uniqueness  
      - Drive: "My Drive"  
      - Input data field: Uses binary file field per item  
    - Inputs: Original files from Files to Items (parallel to AI processing branch)  
    - Outputs: Metadata including Google Drive URL for the uploaded file  
    - Credentials: Google Drive OAuth2 account  
    - Edge cases: Authentication errors; exceeding Google Drive quota; file size limits.

- **Sticky note details:** Explains the choice of Google Drive for photo storage with flexibility to substitute other services.

#### 2.5 Data Logging

- **Overview:**  
  Merges AI classification data with Google Drive file metadata and user submission details, then appends a structured record into a Google Sheets table for searchable and filterable storage.

- **Nodes Involved:**  
  - Merge  
  - Merge1  
  - Create Payload  
  - Append to Sheet

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines data streams from AI classification and image metadata for each image.  
    - Configuration: Combine mode by position to align items from both branches.  
    - Inputs:  
      - From Convert Image to Base64 branch (AI classification data)  
      - From Get File Meta branch (image metadata)  
    - Outputs: Combined JSON for each image.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines categories extracted from AI with Google Drive file upload metadata.  
    - Configuration: Combine mode by position.  
    - Inputs:  
      - Extract Categories node output (categories array)  
      - Upload file node output (Google Drive file metadata)  
    - Outputs: Merged data for payload creation.

  - **Create Payload**  
    - Type: Set  
    - Role: Prepares the final data object to append to Google Sheets.  
    - Configuration:  
      - Fields set:  
        - url: Google Drive webViewLink  
        - catergories (note spelling): array of categories  
        - submittedBy: uploader name from form submission  
        - submittedAt: ISO timestamp of form submission  
        - filename: original file name  
    - Inputs: Merged data from Merge1  
    - Outputs: Payload for Google Sheets append.

  - **Append to Sheet**  
    - Type: Google Sheets  
    - Role: Appends one row per photo entry to a predefined Google Sheets document and sheet.  
    - Configuration:  
      - Document ID: "1TpXQyhUq6tB8MLJ3maeWwswjut9wERZ8pSk_3kKhc58"  
      - Sheet Name: "Sheet2" (gid 1533965902)  
      - Columns mapped: url, filename, categories (joined as comma-separated string), submittedBy, submittedAt  
      - Operation: Append  
    - Credentials: Google Sheets OAuth2 account  
    - Inputs: Payload from Create Payload  
    - Edge cases: Authentication errors; spreadsheet limits; network issues.

- **Sticky note details:** Highlights the convenience of using Google Sheets for filtering and sorting event photo data, including a sample sheet URL.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                                   | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                                                        |
|-------------------------------|--------------------|-------------------------------------------------|------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger       | Captures uploaded photos and user name          | None                         | Files to Items                   | ## 1. Allow Guests to Upload via Form Trigger. Mobile-friendly form interface. [Learn more about the form trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger) |
| Files to Items                | SplitOut           | Splits multiple uploaded files into separate items | On form submission           | Get File Meta, Get Image Info, Upload file |                                                                                                                                   |
| Get File Meta                | Set                | Extracts metadata from binary file data          | Files to Items                | Merge                           |                                                                                                                                   |
| Get Image Info               | Edit Image         | Retrieves image dimensions and info              | Files to Items                | Resize Image                    | ## 2. Resize the Images for Faster Processing. Optimize images for AI speed. [Learn more about the Edit Image node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage) |
| Resize Image                 | Edit Image         | Resizes images to max width 512px                 | Get Image Info               | Convert Image to Base64          | See above                                                                                                                       |
| Convert Image to Base64      | Extract From File  | Converts resized image binaries to base64 string | Resize Image                 | Merge                          | See above                                                                                                                       |
| Merge                       | Merge              | Combines base64 image data with file metadata    | Convert Image to Base64, Get File Meta | Classify Photo and Suggest Tags |                                                                                                                                   |
| Classify Photo and Suggest Tags | HTTP Request       | Sends image to Featherless AI API for classification | Merge                       | Extract Categories              | ## 3. Categorise Photos using Gemma Model via Featherless.ai. [Featherless.ai](https://featherless.ai/register?referrer=HJUUTA6M) subscription offers unlimited tokens. |
| Extract Categories           | Set                | Parses AI response into array of categories      | Classify Photo and Suggest Tags | Merge1                         | See above                                                                                                                       |
| Upload file                 | Google Drive       | Uploads original photo to Google Drive folder     | Files to Items                | Merge1                         | ## 4. Store a Copy of the Photo in Google Drive. [Learn more about the Edit Image node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage) |
| Merge1                      | Merge              | Combines categories with Google Drive file info  | Extract Categories, Upload file | Create Payload                 |                                                                                                                                   |
| Create Payload              | Set                | Creates final data object for Google Sheets logging | Merge1                      | Append to Sheet                |                                                                                                                                   |
| Append to Sheet             | Google Sheets      | Appends photo info and categories to Google Sheets | Create Payload              | None                          | ## 5. Save Entry Into Google Sheets Table. [Learn more about the Google Sheet node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/). Sample sheet: https://docs.google.com/spreadsheets/d/1TpXQyhUq6tB8MLJ3maeWwswjut9wERZ8pSk_3kKhc58/edit?usp=sharing |
| Sticky Note                 | Sticky Note        | Informational content about form trigger          | None                         | None                          | See above                                                                                                                       |
| Sticky Note1                | Sticky Note        | Notes on image resizing and optimization          | None                         | None                          | See above                                                                                                                       |
| Sticky Note2                | Sticky Note        | Notes on AI classification and Featherless usage  | None                         | None                          | See above                                                                                                                       |
| Sticky Note3                | Sticky Note        | Notes on Google Sheets logging                      | None                         | None                          | See above                                                                                                                       |
| Sticky Note4                | Sticky Note        | Notes on Google Drive storage                       | None                         | None                          | See above                                                                                                                       |
| Sticky Note5                | Sticky Note        | Overview and project description                    | None                         | None                          | ## Classify User-Uploaded Photos for Public Events using [Featherless.ai](https://featherless.ai/register?referrer=HJUUTA6M) with workflow summary and usage notes. Final result link included. |
| Sticky Note9                | Sticky Note        | Branding banner image                               | None                         | None                          | ![](https://cdn.subworkflow.ai/n8n-templates/banner_595x311.png#full-width)                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure form title: "Photo Uploads"  
   - Add two fields:  
     - Name (Text, Required)  
     - Files (File upload, Required, accept jpg, png)  
   - Enable bot filtering  
   - Set button label: "Upload"  
   - Save and note the webhook URL for guests to submit photos.

2. **Add SplitOut Node ("Files to Items"):**  
   - Type: SplitOut  
   - Configure to split the `$binary` field (which contains uploaded files) into individual items.

3. **Add Set Node ("Get File Meta"):**  
   - Type: Set  
   - Set mode: Raw  
   - Output the first binary file's JSON metadata dynamically with expression:  
     `={{ $binary[$binary.keys()[0]] }}`

4. **Add Edit Image Node ("Get Image Info"):**  
   - Operation: Information  
   - Data Property: Use dynamic field for each file e.g. `=Files_{{ $itemIndex }}`

5. **Add Edit Image Node ("Resize Image"):**  
   - Operation: Resize  
   - Width: `={{ $json.size.width > 512 ? 512 : $json.size.width }}`  
   - Height: `={{ $json.size.width > 512 ? (512 / $json.size.width) * $json.size.height : $json.size.height }}`  
   - Data Property: `=Files_{{ $itemIndex }}`

6. **Add Extract From File Node ("Convert Image to Base64"):**  
   - Operation: Binary to Property  
   - Binary Property Name: `=Files_{{ $itemIndex }}`

7. **Add Merge Node ("Merge"):**  
   - Mode: Combine  
   - Combine By: Position  
   - Connect inputs from "Convert Image to Base64" and "Get File Meta"

8. **Add HTTP Request Node ("Classify Photo and Suggest Tags"):**  
   - Method: POST  
   - URL: `https://api.featherless.ai/v1/chat/completions`  
   - Authentication: Use Featherless API credentials and HTTP header auth with your API key.  
   - Body (JSON):  
     ```json
     {
       "model": "google/gemma-3-27b-it",
       "messages": [
         {
           "role": "user",
           "content": [
             {
               "type": "image_url",
               "image_url": {
                 "url": "data:{{ $json.mimeType }};base64,{{ $json.data }}"
               }
             },
             {
               "type": "text",
               "text": "Classify the image into one or more of following categories:Group Shot,Candid,Smiles,Food & Drink,Decorations,Fun,Laughter,Music,Dance Floor,Conversations,Networking,Toasting,Activity,Portrait,Venue,Best Dressed,Selfie,Cheers,Memories. If none of these categories apply, return \"Uncategorised\". Output your response as a comma-limited list only. Do not explain your reasoning."
             }
           ]
         }
       ]
     }
     ```
   - Enable batching with batch size 1 and interval 3000ms.

9. **Add Set Node ("Extract Categories"):**  
   - Create an array field `categories` by splitting AI response content:  
     `={{ $json.choices[0].message.content.split(',') }}`

10. **Add Google Drive Node ("Upload file"):**  
    - Operation: Upload  
    - Folder ID: `1flK9aO20jw8Npf1EioiOl42i6FFLBGrX` (replace with your folder)  
    - File Name: Use expression to generate unique name:  
      ```javascript
      {{$('On form submission').item.json.Name + '_' + $execution.id + '_' + $itemIndex + '.' + $binary[$binary.keys()[0]].fileName.split('.')[1]}}
      ```  
    - Drive: "My Drive"  
    - Input Data Field Name: Bind per file, e.g., `Files_{{ $itemIndex }}`  
    - Credential: Google Drive OAuth2

11. **Add Merge Node ("Merge1"):**  
    - Mode: Combine  
    - Combine By: Position  
    - Inputs: From "Extract Categories" and "Upload file"

12. **Add Set Node ("Create Payload"):**  
    - Create fields:  
      - url: `={{ $json.webViewLink }}`  
      - catergories: `={{ $json.categories }}` (note the spelling as in workflow)  
      - submittedBy: `={{ $('On form submission').first().json.Name }}`  
      - submittedAt: `={{ $('On form submission').item.json.submittedAt.toDateTime().toISO() }}`  
      - filename: `={{ $json.name }}`

13. **Add Google Sheets Node ("Append to Sheet"):**  
    - Operation: Append  
    - Document ID: Your Google Sheet ID (e.g., `1TpXQyhUq6tB8MLJ3maeWwswjut9wERZ8pSk_3kKhc58`)  
    - Sheet Name: Sheet2 or your target sheet  
    - Mapping columns:  
      - url (string)  
      - filename (string)  
      - categories (string, join array with commas): `={{ $json.catergories.map(x => x.trim()).join(',') }}`  
      - submittedBy (string)  
      - submittedAt (string)  
    - Credential: Google Sheets OAuth2

14. **Connect Nodes in Order:**  
    - On form submission → Files to Items  
    - Files to Items → Get File Meta, Get Image Info, Upload file (parallel)  
    - Get File Meta → Merge (input 2)  
    - Get Image Info → Resize Image → Convert Image to Base64 → Merge (input 1)  
    - Merge → Classify Photo and Suggest Tags → Extract Categories → Merge1 (input 1)  
    - Upload file → Merge1 (input 2)  
    - Merge1 → Create Payload → Append to Sheet

15. **Add Sticky Notes:**  
    - Add informational sticky notes describing each block and links to documentation as per original workflow for clarity and onboarding.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Uses Featherless.ai for multimodal LLM inference with Google Gemma model to classify images by event-related categories. Subscription-based with unlimited tokens. | https://featherless.ai/register?referrer=HJUUTA6M |
| Google Drive is used for photo storage; can be replaced by alternatives (e.g., SharePoint). | n8n Google Drive node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/ |
| Google Sheets stores photo metadata and classification for easy filtering and sharing. | Sample sheet: https://docs.google.com/spreadsheets/d/1TpXQyhUq6tB8MLJ3maeWwswjut9wERZ8pSk_3kKhc58/edit?usp=sharing |
| Form Trigger node enables mobile-friendly, easy photo upload with user identification. | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger |
| Edit Image node used to optimize image size for faster AI processing. | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage |
| Join n8n community for help: Discord https://discord.com/invite/XPKeKXeB7d, Forum https://community.n8n.io/ | n8n support channels |

---

**Disclaimer:**  
The provided content is derived exclusively from an automated workflow created using n8n, adhering strictly to applicable content policies and handling only legal, public data.