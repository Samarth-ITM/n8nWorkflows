Batch Upscale Portrait Photos with Real-ESRGAN AI, Google Drive and Airtable

https://n8nworkflows.xyz/workflows/batch-upscale-portrait-photos-with-real-esrgan-ai--google-drive-and-airtable-8047


# Batch Upscale Portrait Photos with Real-ESRGAN AI, Google Drive and Airtable

### 1. Workflow Overview

This workflow automates the process of batch upscaling portrait photos stored in Airtable using the Real-ESRGAN AI model via the Replicate API, then uploads the enhanced images to a specified Google Drive folder. It is designed for users who maintain portrait photos in Airtable and want to improve their resolution automatically, organizing the outputs efficiently in Google Drive.

**Logical Blocks:**

- **1.1 Manual Trigger & Folder Creation:** Starts execution manually and creates a dedicated folder in Google Drive to store upscaled images.
- **1.2 Airtable Record Retrieval & URL Extraction:** Fetches a record from Airtable containing portrait photos and extracts the image URLs along with metadata.
- **1.3 Batch Processing Loop:** Iterates over each portrait image URL for sequential processing.
- **1.4 AI Upscaling via Replicate API:** Sends each image URL to the Real-ESRGAN model hosted on Replicate to generate an upscaled version.
- **1.5 Download & Prepare for Upload:** Downloads the upscaled image file and prepares metadata for upload.
- **1.6 Upload to Google Drive:** Uploads the upscaled images to the previously created Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & Folder Creation

- **Overview:** Initiates the workflow manually and creates a new folder in Google Drive where upscaled images will be stored.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Create folder

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Connected to "Create folder"  
    - Edge Cases: None specific; user must click to start.

  - **Create folder**  
    - Type: Google Drive node (create folder)  
    - Role: Creates a folder in Google Drive using a user-supplied folder name.  
    - Configuration:  
      - Folder name is provided via expression: `=< Folder Name >` (placeholder to be filled by user)  
      - Drive ID and Folder ID are set via list mode but default to empty (likely user must select correct Drive)  
      - Uses OAuth2 credentials for Google Drive (named "Vertical Google Drive account")  
    - Inputs: Triggered by manual trigger node  
    - Outputs: Folder metadata (including folder ID) passed downstream  
    - Edge Cases:  
      - Failure if folder name is empty or Drive access is unauthorized  
      - Permissions issues on Google Drive  
      - If folder already exists, the node creates a new folder with the same name (no conflict handling)  

#### 2.2 Airtable Record Retrieval & URL Extraction

- **Overview:** Retrieves a specific record from Airtable containing portrait photo data and extracts the URLs and metadata of the portraits for processing.
- **Nodes Involved:**  
  - Get Record from Pictures  
  - Extract Portrait URLS from Airtable Output

- **Node Details:**

  - **Get Record from Pictures**  
    - Type: Airtable node (get record)  
    - Role: Fetches one record from Airtable by ID to access portrait photo data.  
    - Configuration:  
      - Record ID placeholder `=< enter Record ID >` to be replaced by user  
      - Base and table are selected via ID mode (must be configured by user)  
      - Uses an Airtable API token credential ("Book your fantasy Airtable")  
    - Inputs: Receives folder info from "Create folder" node  
    - Outputs: Airtable record data sent to code node  
    - Edge Cases:  
      - Fail if record ID is invalid or missing  
      - Fail if Airtable API token is invalid or lacks access  
      - No pagination needed since a single record is fetched

  - **Extract Portrait URLS from Airtable Output**  
    - Type: Code node (JavaScript)  
    - Role: Parses Airtable record JSON to extract portrait image URLs and metadata into a simple array, attaching the Google Drive folder ID.  
    - Configuration:  
      - Custom JS code iterates over `PortraitFotoAuswahl` array in the Airtable data  
      - For each image, extracts URL, filename, size, width, height  
      - Injects the Drive folder ID from the "Create folder" node  
    - Inputs: Airtable record JSON  
    - Outputs: Array of JSON objects, each representing one portrait image with attached folder ID  
    - Edge Cases:  
      - If `PortraitFotoAuswahl` does not exist or is not an array, returns empty array (no images processed)  
      - Missing URL fields are skipped  
      - Assumes consistent Airtable schema  

#### 2.3 Batch Processing Loop

- **Overview:** Processes each extracted portrait image one-by-one through the workflow for upscaling and uploading.
- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches node  
    - Role: Iterates over input array items (portrait images) to process them sequentially.  
    - Configuration: Default batching (one item at a time)  
    - Inputs: Receives array of portrait images from code node  
    - Outputs: Each item sent individually to the next node in the chain  
    - Edge Cases:  
      - If input array is empty, loop does not run  
      - Supports error continuation downstream if any batch item fails  

#### 2.4 AI Upscaling via Replicate API

- **Overview:** Sends each portrait image URL to the Real-ESRGAN model on Replicate to generate an upscaled version with 2x scale.
- **Nodes Involved:**  
  - Replicate Upscaler  
  - Download Upscale  
  - Set GDrive Upload Folder ID

- **Node Details:**

  - **Replicate Upscaler**  
    - Type: HTTP Request node  
    - Role: Calls Replicate API to create a prediction job for upscaling the image.  
    - Configuration:  
      - POST to `https://api.replicate.com/v1/predictions`  
      - JSON body specifies model version `nightmareai/real-esrgan` with parameters:  
        - image: URL from current batch item  
        - scale: 2 (2x upscaling)  
        - face_enhance: false  
      - Auth: HTTP Header Auth with API token stored in "Replicate" credential  
      - Header includes `Prefer: wait` to wait for completion synchronously  
      - On error: continues regular output (avoids workflow halt)  
    - Inputs: Single portrait URL from batch item  
    - Outputs: Prediction response containing output URL  
    - Edge Cases:  
      - API rate limits or quota exceeded errors  
      - Timeout if prediction takes too long  
      - Invalid image URL or unsupported format  
      - Server errors from Replicate

  - **Download Upscale**  
    - Type: HTTP Request node  
    - Role: Downloads the upscaled image file from the URL provided by Replicate.  
    - Configuration:  
      - URL set dynamically from Replicate output JSON field `output`  
      - Response format: file (binary)  
      - On error: continues regular output  
    - Inputs: Replicate API output containing file URL  
    - Outputs: Binary file of upscaled image  
    - Edge Cases:  
      - Broken or expired URL  
      - Network timeouts or errors downloading file  
      - File size too large for workflow limits

  - **Set GDrive Upload Folder ID**  
    - Type: Code node (JavaScript)  
    - Role: Prepares metadata for uploading by attaching folder ID and original image info to the binary data.  
    - Configuration:  
      - Extracts original filename, index, and folder ID from batch item context  
      - Passes binary data unchanged with new JSON metadata  
    - Inputs: Binary file from download node plus batch metadata  
    - Outputs: JSON + binary ready for upload  
    - Edge Cases:  
      - Failure if metadata is missing or malformed  

#### 2.5 Upload to Google Drive

- **Overview:** Uploads the downloaded and processed upscaled image files to the designated Google Drive folder.
- **Nodes Involved:**  
  - Upload to Google Drive

- **Node Details:**

  - **Upload to Google Drive**  
    - Type: Google Drive node (file upload)  
    - Role: Uploads each upscaled portrait image file into the created Google Drive folder.  
    - Configuration:  
      - Filename dynamically constructed as `Upscaled_Picture + timestamp + .png` using current date/time  
      - Drive ID set to “My Drive”  
      - Folder ID dynamically set from metadata field `gdriveUploadFolderId`  
      - Uses OAuth2 credentials ("Vertical Google Drive account")  
      - On error: continues regular output (prevents workflow stop on individual upload failure)  
    - Inputs: Binary file and metadata from previous node  
    - Outputs: Confirmation and metadata of uploaded file  
    - Edge Cases:  
      - Drive permission denied or quota exceeded  
      - Invalid folder ID  
      - Filename collisions (overwrites prevented by timestamp)  

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                         | Input Node(s)                     | Output Node(s)                 | Sticky Note                                           |
|----------------------------------|-------------------------|---------------------------------------|----------------------------------|-------------------------------|-------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Starts workflow execution manually    | None                             | Create folder                 |                                                       |
| Create folder                    | Google Drive (folder)   | Creates Google Drive folder for output| When clicking ‘Execute workflow’ | Get Record from Pictures      | ## Add Folder Name                                    |
| Get Record from Pictures         | Airtable                | Retrieves Airtable record with images | Create folder                   | Extract Portrait URLS from Airtable Output | ## Enter Record from Stored Data                      |
| Extract Portrait URLS from Airtable Output | Code                 | Parses Airtable data to extract image URLs | Get Record from Pictures        | Loop Over Items               |                                                       |
| Loop Over Items                  | SplitInBatches          | Iterates over each portrait image     | Extract Portrait URLS from Airtable Output | Replicate Upscaler           |                                                       |
| Replicate Upscaler               | HTTP Request            | Sends images to Replicate AI for upscaling | Loop Over Items                 | Download Upscale             |                                                       |
| Download Upscale                | HTTP Request            | Downloads upscaled image file          | Replicate Upscaler               | Set GDrive Upload Folder ID   |                                                       |
| Set GDrive Upload Folder ID     | Code                    | Prepares metadata for Google Drive upload | Download Upscale                | Upload to Google Drive        |                                                       |
| Upload to Google Drive          | Google Drive (upload)   | Uploads upscaled images to Drive folder | Set GDrive Upload Folder ID    | Loop Over Items               |                                                       |
| Sticky Note                     | Sticky Note             | Info on workflow purpose               | None                            | None                         | ## Information  \nThis Node will Upscale Multiple Pictures from A Dataspace in Airtable \n\n1. it will create A folder in Drive\n2. it will take pictures stored in the airtable Dataspace\n3. it will run over Items to upscale in Replicate the pictures\n4. it will store the pictures in Drive\n\nPlease Upload the Pictures in Airtable in a column: PortraitFotoAuswahl |
| Sticky Note1                    | Sticky Note             | Instruction to enter record ID         | None                            | None                         | ## Enter Record from Stored Data                       |
| Sticky Note2                    | Sticky Note             | Instruction to add folder name         | None                            | None                         | ##  Add Folder Name                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Add “Manual Trigger” node named "When clicking ‘Execute workflow’". No parameters required.

2. **Add Google Drive “Create folder” Node**  
   - Connect from manual trigger node.  
   - Set operation to create folder.  
   - Configure folder name as an expression input (e.g., `< Folder Name >` placeholder).  
   - Select proper Google Drive via OAuth2 credential ("Vertical Google Drive account").  
   - Leave Drive ID empty to default or select your drive.  
   - Connect output to next node.

3. **Add Airtable “Get Record” Node**  
   - Connect from “Create folder”.  
   - Configure base and table by ID or name.  
   - Enter the record ID to fetch (replace placeholder).  
   - Use appropriate Airtable API token credentials ("Book your fantasy Airtable").  
   - Connect to next node.

4. **Add Code Node “Extract Portrait URLS from Airtable Output”**  
   - Connect from Airtable node.  
   - Paste JavaScript code to parse `PortraitFotoAuswahl` array from Airtable data, extract URLs and metadata, and append folder ID from the previous Google Drive folder creation node.  
   - Output array of JSON objects, each for one portrait image.  
   - Connect to next node.

5. **Add SplitInBatches Node “Loop Over Items”**  
   - Connect from code node.  
   - Leave default batch size (1).  
   - Connect output to Replicate node.

6. **Add HTTP Request Node “Replicate Upscaler”**  
   - Connect from “Loop Over Items” (first output).  
   - Set method POST, URL `https://api.replicate.com/v1/predictions`.  
   - In body, set JSON with model version and input: image URL from current batch item, scale=2, face_enhance=false.  
   - Set authentication to HTTP Header with your Replicate API key stored in credential "Replicate".  
   - Add header `Prefer: wait`.  
   - On error: continue regular output.  
   - Connect output to “Download Upscale”.

7. **Add HTTP Request Node “Download Upscale”**  
   - Connect from “Replicate Upscaler”.  
   - Set method GET.  
   - URL dynamically set from Replicate `output` field.  
   - Response format: file (binary).  
   - On error: continue regular output.  
   - Connect output to “Set GDrive Upload Folder ID”.

8. **Add Code Node “Set GDrive Upload Folder ID”**  
   - Connect from “Download Upscale”.  
   - JavaScript code to merge batch metadata (folder ID, original filename, index) with the binary data of the downloaded file.  
   - Output JSON + binary ready for upload.  
   - Connect to “Upload to Google Drive”.

9. **Add Google Drive “Upload to Google Drive” Node**  
   - Connect from “Set GDrive Upload Folder ID”.  
   - Configure to upload file:  
     - Filename: `Upscaled_Picture` + timestamp + `.png` via expression.  
     - Drive: “My Drive” or your Google Drive.  
     - Folder ID dynamically from incoming JSON field `gdriveUploadFolderId`.  
   - Use same Google Drive OAuth2 credential.  
   - On error: continue regular output.  
   - Connect output back to “Loop Over Items” (second output) to continue processing next item.

10. **Add Sticky Notes as Needed**  
    - Add informative sticky notes with instructions for folder name, record ID, and workflow purpose.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow batch upscales portrait photos stored in Airtable using Real-ESRGAN AI via Replicate, then uploads results to Drive. | Main workflow description                                                                       |
| Please upload portrait images in Airtable in a column named `PortraitFotoAuswahl` as an array of objects with `url` and metadata.  | Airtable column requirements                                                                    |
| Use the “Vertical Google Drive account” OAuth2 credentials for Google Drive nodes and “Book your fantasy Airtable” for Airtable.  | Credential references                                                                           |
| Replicate API requires an HTTP Header Auth credential with your API token configured under “Replicate”.                            | Replicate API credential setup                                                                  |
| The Real-ESRGAN model version used: `nightmareai/real-esrgan:f121d640bd286e1fdc67f9799164c1d5be36ff74576ee11c803ae5b665dd46aa`.      | Model version reference                                                                         |
| For more info on Real-ESRGAN: https://replicate.com/nightmareai/real-esrgan                                                          | Replicate model page                                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies and containing no illegal, offensive, or protected elements. All processed data is legal and public.