Upload Video & Image Ads to Meta Ads Manager from Google Sheets & Google Drive

https://n8nworkflows.xyz/workflows/upload-video---image-ads-to-meta-ads-manager-from-google-sheets---google-drive-5883


# Upload Video & Image Ads to Meta Ads Manager from Google Sheets & Google Drive

### 1. Workflow Overview

This workflow automates the upload and management of video and image ads to Meta Ads Manager by integrating Google Sheets, Google Drive, and the Meta Graph API. It targets advertisers and marketing teams who manage ad creatives stored in Google Drive and ad metadata in Google Sheets, streamlining ad creative uploads, processing, and recording results back into Google Sheets.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggering events from Google Drive folder updates or manual triggers to start the workflow.
- **1.2 File Retrieval and Metadata Extraction:** Fetching files from Google Drive and extracting relevant metadata.
- **1.3 Ads Retrieval from Google Sheets:** Getting the list of ads to process from Google Sheets.
- **1.4 Processing Loop:** Iterating through each ad, distinguishing between video and image assets.
- **1.5 Video Ad Handling:** Uploading and processing video ads, including status checking and creative creation.
- **1.6 Image Ad Handling:** Uploading images, handling single or multiple images, creating creatives, and ads.
- **1.7 Saving Results:** Writing back ad creation details to Google Sheets.
- **1.8 Control and Waiting:** Managing asynchronous processing with waits and conditional checks.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts the workflow either manually via a trigger or automatically when a Google Drive folder is updated.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Google Drive Folder Updated (Google Drive Trigger)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation for testing or manual runs.  
  - Inputs: None  
  - Outputs: Starts the workflow on manual trigger.  
  - Edge Cases: None expected; user manually triggers.  

- **Google Drive Folder Updated**  
  - Type: Google Drive Trigger  
  - Role: Listens for file changes in a specified Google Drive folder to kick off workflow automatically.  
  - Configuration: Folder ID or path to monitor.  
  - Inputs: None  
  - Outputs: Emits event when new files are added or updated.  
  - Edge Cases: Permissions error if folder access not granted, missed triggers if webhook fails.

---

#### 2.2 File Retrieval and Metadata Extraction

**Overview:**  
After trigger, fetches all files from Google Drive and extracts necessary metadata for processing.

**Nodes Involved:**  
- settings (Set node)  
- Fetch All Files (Google Drive)  
- Get Files Metadata (HTTP Request)  
- Extract File Specs (Set node)  
- Create Asset (Google Sheets)

**Node Details:**

- **settings**  
  - Type: Set  
  - Role: Initialize or set parameters needed for fetching files.  
  - Configuration: Sets variables like folder ID, file types, or filters.  
  - Inputs: Trigger outputs  
  - Outputs: Passes settings to file fetch node.

- **Fetch All Files**  
  - Type: Google Drive  
  - Role: Retrieves list of files from the specified Google Drive folder.  
  - Configuration: Folder ID and filters like file type or mime type.  
  - Inputs: settings node output  
  - Outputs: Array of files metadata.

- **Get Files Metadata**  
  - Type: HTTP Request  
  - Role: Obtains detailed metadata about files, possibly using Google Drive API beyond default node capabilities.  
  - Configuration: Request URL and headers for Google Drive API.  
  - Inputs: Files from Fetch All Files.  
  - Outputs: Detailed metadata for each file.

- **Extract File Specs**  
  - Type: Set  
  - Role: Processes and extracts relevant file specs (e.g., file name, type, id) from metadata for subsequent use.  
  - Configuration: Uses expressions to parse metadata.  
  - Inputs: Metadata from previous HTTP Request.  
  - Outputs: Cleaned and structured file info.

- **Create Asset**  
  - Type: Google Sheets  
  - Role: Optionally writes or updates asset info into Google Sheets for record-keeping.  
  - Configuration: Target spreadsheet and sheet, columns mapping.  
  - Inputs: File specs from Extract File Specs.  
  - Outputs: None (or confirmation).

**Edge Cases:**  
- API quota exceeded or Google Drive permission errors.  
- Empty folder or no new files.  
- Metadata extraction failures if API changes.

---

#### 2.3 Ads Retrieval from Google Sheets

**Overview:**  
Fetches the list of ads to process, including ad specs and asset references from a Google Sheet.

**Nodes Involved:**  
- settings_1 (Set)  
- Get Ads to Process (Google Sheets)  
- Loop Over Items (SplitInBatches)

**Node Details:**

- **settings_1**  
  - Type: Set  
  - Role: Initializes parameters such as spreadsheet ID, sheet name, or filters for ads retrieval.  
  - Inputs: Trigger or manual start.  
  - Outputs: Parameter object for Google Sheets node.

- **Get Ads to Process**  
  - Type: Google Sheets  
  - Role: Reads rows representing ads to be processed.  
  - Configuration: Spreadsheet ID, sheet, range, or filters.  
  - Inputs: settings_1 output.  
  - Outputs: List of ads with relevant fields like asset type, links, ad set IDs.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates through ads one at a time or in small batches to avoid rate limits.  
  - Configuration: Batch size, usually set to 1 for sequential processing.  
  - Inputs: List of ads.  
  - Outputs: Single ad item per iteration.

**Edge Cases:**  
- Empty or invalid rows in Google Sheets.  
- Rate limits or large data sets requiring batching.

---

#### 2.4 Processing Loop and Asset Type Switch

**Overview:**  
Determines whether each ad asset is video or image and routes processing accordingly.

**Nodes Involved:**  
- Video or Image Asset (Switch)  
- Upload Ad Video (Facebook Graph API)  
- Get image 1 (HTTP Request)  
- Upload Ad Image (Facebook Graph API)

**Node Details:**

- **Video or Image Asset**  
  - Type: Switch  
  - Role: Evaluates asset type field in ad data to route processing for video or image.  
  - Configuration: Condition on asset type field (e.g., ‘video’ vs ‘image’).  
  - Inputs: Single ad from Loop Over Items.  
  - Outputs: Two branches: video path and image path.

- **Upload Ad Video**  
  - Type: Facebook Graph API  
  - Role: Initiates upload of video asset to Meta Ads Manager.  
  - Configuration: Uses credentials with access to Meta API; parameters include video URL or file ID.  
  - Inputs: Video asset info.  
  - Outputs: Upload response, including video ID or status.  
  - Edge Cases: Upload failures, auth errors, video size or format issues.

- **Get image 1**  
  - Type: HTTP Request  
  - Role: Retrieves first image asset from a URL or source.  
  - Configuration: URL from ad data.  
  - Inputs: Image asset info.  
  - Outputs: Image binary or metadata.

- **Upload Ad Image**  
  - Type: Facebook Graph API  
  - Role: Uploads the primary image to Meta Ads Manager.  
  - Inputs: Image data from HTTP Request.  
  - Outputs: Image upload response.

**Edge Cases:**  
- Incorrect asset type field causing wrong routing.  
- Upload errors due to network or API limits.

---

#### 2.5 Video Ad Handling

**Overview:**  
Handles detailed video ad creation by uploading video, waiting for processing, creating creatives and ads, and saving results.

**Nodes Involved:**  
- Switch  
- Get Video  
- Code  
- Upload Ad Video1  
- Wait  
- Check if finished processing  
- If1  
- Get Video Preview Image  
- Create Video Ad Creative  
- Create Video Ad  
- Save Video Ad Details

**Node Details:**

- **Switch**  
  - Type: Switch  
  - Role: Decides next step based on video upload status.  
  - Inputs: Upload response.  
  - Outputs: Either wait or fetch video status.

- **Get Video**  
  - Type: HTTP Request  
  - Role: Checks video processing status via Meta API.  
  - Inputs: Video ID from upload.  
  - Outputs: Processing status response.

- **Code**  
  - Type: Code  
  - Role: Custom logic to parse video status and determine next action.  
  - Inputs: Status response.  
  - Outputs: Decision for next node.

- **Upload Ad Video1**  
  - Type: Facebook Graph API  
  - Role: Possibly retries or continues video upload process.  
  - Inputs: From code decision.  
  - Outputs: Upload status.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow to allow video processing on Meta side.  
  - Inputs: Control flow.  
  - Outputs: Continues after delay.

- **Check if finished processing**  
  - Type: Facebook Graph API  
  - Role: Polls video processing status.  
  - Inputs: Video ID.  
  - Outputs: Status info.

- **If1**  
  - Type: If  
  - Role: Conditional node checking if video processing is complete.  
  - Inputs: Status from previous node.  
  - Outputs: If complete, continue; else, wait and poll again.

- **Get Video Preview Image**  
  - Type: Facebook Graph API  
  - Role: Fetches thumbnail or preview image of the uploaded video asset.  
  - Inputs: Video ID.  
  - Outputs: Preview image URL or ID.

- **Create Video Ad Creative**  
  - Type: Facebook Graph API  
  - Role: Creates a video ad creative using the uploaded video asset.  
  - Inputs: Video asset ID and ad parameters.  
  - Outputs: Creative ID.

- **Create Video Ad**  
  - Type: Facebook Graph API  
  - Role: Creates the actual video ad linked to the creative.  
  - Inputs: Creative ID, ad set ID, campaign data.  
  - Outputs: Ad ID and status.

- **Save Video Ad Details**  
  - Type: Google Sheets  
  - Role: Saves details of the created video ad back to Google Sheets for tracking.  
  - Inputs: Ad ID, creative info, status.  
  - Outputs: Confirmation.

**Edge Cases:**  
- Video processing timeout or failure on Meta side.  
- API rate limits or credential issues.  
- Video format incompatibility.

---

#### 2.6 Image Ad Handling

**Overview:**  
Handles uploading one or two images, creating image creatives and ads, and saving results.

**Nodes Involved:**  
- Check if two or one image (If)  
- Get image 1 (HTTP Request)  
- Get image (HTTP Request)  
- Upload Ad Image (Facebook Graph API)  
- Upload Second Ad Image (Facebook Graph API)  
- Create Image Creative  
- Create Multiple Image Creative  
- Create Image Ad  
- Save Image Ad Details

**Node Details:**

- **Check if two or one image**  
  - Type: If  
  - Role: Determines if the ad uses one or two images to select appropriate upload path.  
  - Inputs: Ad data.  
  - Outputs: Branches for single or multiple images.

- **Get image 1**  
  - Type: HTTP Request  
  - Role: Downloads the first image asset from a URL.  
  - Inputs: Image URL.  
  - Outputs: Image binary.

- **Get image**  
  - Type: HTTP Request  
  - Role: Downloads second image asset if applicable.  
  - Inputs: Second image URL.  
  - Outputs: Image binary.

- **Upload Ad Image**  
  - Type: Facebook Graph API  
  - Role: Uploads first image asset to Meta Ads Manager.  
  - Inputs: Image binary.  
  - Outputs: Image ID.

- **Upload Second Ad Image**  
  - Type: Facebook Graph API  
  - Role: Uploads second image asset for multiple image ads.  
  - Inputs: Second image binary.  
  - Outputs: Image ID.

- **Create Image Creative**  
  - Type: Facebook Graph API  
  - Role: Creates a single image ad creative.  
  - Inputs: Image ID, ad parameters.  
  - Outputs: Creative ID.

- **Create Multiple Image Creative**  
  - Type: Facebook Graph API  
  - Role: Creates a multiple image ad creative using both uploaded images.  
  - Inputs: Both image IDs, ad parameters.  
  - Outputs: Creative ID.

- **Create Image Ad**  
  - Type: Facebook Graph API  
  - Role: Creates the actual image ad linked to the creative.  
  - Inputs: Creative ID, ad set ID, campaign data.  
  - Outputs: Ad ID and status.

- **Save Image Ad Details**  
  - Type: Google Sheets  
  - Role: Writes created image ad details back to Google Sheets for tracking.  
  - Inputs: Ad ID, creative info, status.  
  - Outputs: Confirmation.

**Edge Cases:**  
- Missing or invalid image URLs.  
- Upload failures or API errors.  
- Mismatch in expected number of images.

---

#### 2.7 Saving Results

**Overview:**  
Writes back to Google Sheets the details of created ads (both video and image) to maintain a record and enable tracking.

**Nodes Involved:**  
- Save Video Ad Details (Google Sheets)  
- Save Image Ad Details (Google Sheets)

**Node Details:**

- **Save Video Ad Details**  
  - Type: Google Sheets  
  - Role: Updates spreadsheet with video ad creation info such as ad ID, creative ID, status.  
  - Inputs: Video ad creation outputs.  
  - Outputs: Confirmation of write operation.

- **Save Image Ad Details**  
  - Type: Google Sheets  
  - Role: Updates spreadsheet with image ad creation info.  
  - Inputs: Image ad creation outputs.  
  - Outputs: Confirmation.

**Edge Cases:**  
- Google Sheets permission errors.  
- Write conflicts if multiple workflow runs write concurrently.

---

#### 2.8 Control and Waiting

**Overview:**  
Manages asynchronous tasks and timing, particularly for video processing checks, using wait nodes and conditional loops.

**Nodes Involved:**  
- Wait  
- Wait1  
- Check if finished processing  
- If1

**Node Details:**

- **Wait / Wait1**  
  - Type: Wait  
  - Role: Pauses execution to allow Meta to process uploaded videos before next status check.  
  - Inputs: Control flow from previous nodes.  
  - Outputs: Resumes workflow after delay.

- **Check if finished processing**  
  - Type: Facebook Graph API  
  - Role: Queries Meta API to confirm if video processing is complete.  
  - Inputs: Video ID.  
  - Outputs: Processing status.

- **If1**  
  - Type: If  
  - Role: Evaluates processing status; loops back to wait if processing incomplete or continues if done.  
  - Inputs: Status from Check node.  
  - Outputs: Loop control.

**Edge Cases:**  
- Infinite loops if video never finishes processing.  
- Timeout or max retry not set, may cause workflow hanging.

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                      | Input Node(s)                | Output Node(s)                     | Sticky Note                                        |
|-----------------------------|------------------------|------------------------------------|-----------------------------|----------------------------------|---------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger         | Manual start of workflow            | None                        | settings_1                       |                                                   |
| Google Drive Folder Updated  | Google Drive Trigger   | Auto-start on folder update         | None                        | settings                        |                                                   |
| settings                    | Set                    | Initialize fetch parameters         | Google Drive Folder Updated | Fetch All Files                 |                                                   |
| Fetch All Files             | Google Drive           | Fetch files from specified folder   | settings                    | Get Files Metadata              |                                                   |
| Get Files Metadata          | HTTP Request           | Get detailed file metadata          | Fetch All Files             | Extract File Specs              |                                                   |
| Extract File Specs          | Set                    | Extract essential file specs        | Get Files Metadata          | Create Asset                   |                                                   |
| Create Asset               | Google Sheets           | Log asset info to Google Sheets     | Extract File Specs          | None                          |                                                   |
| settings_1                 | Set                    | Initialize ad processing parameters | Manual Trigger/ Webhook     | Get Ads to Process             |                                                   |
| Get Ads to Process         | Google Sheets           | Retrieve ads to process              | settings_1                 | Loop Over Items                |                                                   |
| Loop Over Items            | SplitInBatches          | Process ads one by one               | Get Ads to Process          | Video or Image Asset           |                                                   |
| Video or Image Asset       | Switch                  | Route based on asset type            | Loop Over Items             | Upload Ad Video / Get image 1  |                                                   |
| Upload Ad Video            | Facebook Graph API      | Upload video asset                   | Video or Image Asset        | Switch                       |                                                   |
| Switch                    | Switch                  | Check upload status                  | Upload Ad Video             | Wait / Get Video              |                                                   |
| Wait                      | Wait                    | Pause for video processing          | Switch / Upload Ad Video1   | Check if finished processing  |                                                   |
| Get Video                 | HTTP Request            | Check video processing status       | Switch                     | Code                         |                                                   |
| Code                      | Code                    | Parse status and decide next step   | Get Video                  | Upload Ad Video1              |                                                   |
| Upload Ad Video1           | Facebook Graph API      | Retry or continue video upload      | Code                       | Wait                        |                                                   |
| Check if finished processing| Facebook Graph API      | Poll video processing status        | Wait / Wait1                | If1                         |                                                   |
| If1                       | If                      | Loop control based on status        | Check if finished processing| Get Video Preview Image / Wait1 |                                                   |
| Get Video Preview Image    | Facebook Graph API      | Get preview image for video         | If1                        | Create Video Ad Creative      |                                                   |
| Create Video Ad Creative   | Facebook Graph API      | Create video ad creative            | Get Video Preview Image     | Create Video Ad              |                                                   |
| Create Video Ad            | Facebook Graph API      | Create video ad                     | Create Video Ad Creative    | Save Video Ad Details         |                                                   |
| Save Video Ad Details      | Google Sheets           | Save video ad info                  | Create Video Ad             | Loop Over Items              |                                                   |
| Get image 1               | HTTP Request            | Download first image                | Video or Image Asset        | Upload Ad Image              |                                                   |
| Upload Ad Image           | Facebook Graph API      | Upload first image                  | Get image 1                | Check if two or one image    |                                                   |
| Check if two or one image | If                      | Decide if one or two images         | Upload Ad Image            | Get image / Create Image Creative |                                                   |
| Get image                 | HTTP Request            | Download second image               | Check if two or one image   | Upload Second Ad Image       |                                                   |
| Upload Second Ad Image    | Facebook Graph API      | Upload second image                 | Get image                  | Create Multiple Image Creative |                                                   |
| Create Image Creative     | Facebook Graph API      | Create single image creative        | Check if two or one image   | Create Image Ad              |                                                   |
| Create Multiple Image Creative| Facebook Graph API  | Create multiple image creative      | Upload Second Ad Image      | Create Image Ad              |                                                   |
| Create Image Ad           | Facebook Graph API      | Create image ad                    | Create Image Creative / Create Multiple Image Creative | Save Image Ad Details |                                                   |
| Save Image Ad Details     | Google Sheets           | Save image ad info                 | Create Image Ad             | Loop Over Items              |                                                   |
| Webhook                   | Webhook                 | Alternate manual or webhook starting point | None                   | settings_1                  |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: Allows manual start for testing.

2. **Create Google Drive Trigger Node:**  
   - Type: Google Drive Trigger  
   - Configure to watch specific Google Drive folder for updates.

3. **Create Set Node (settings):**  
   - Purpose: Define Google Drive folder ID and any filters for file fetching.

4. **Create Google Drive Node (Fetch All Files):**  
   - Configure to list files in the folder specified in `settings`.

5. **Create HTTP Request Node (Get Files Metadata):**  
   - Use Google Drive API to get detailed metadata for each file from previous node.

6. **Create Set Node (Extract File Specs):**  
   - Extract and prepare file information such as file name, ID, type for later use.

7. **Create Google Sheets Node (Create Asset):**  
   - Configure with credentials and target spreadsheet to save file asset info.

8. **Create Set Node (settings_1):**  
   - Set parameters for reading ads data from Google Sheets (spreadsheet ID, sheet name).

9. **Create Google Sheets Node (Get Ads to Process):**  
   - Configure to read rows representing ads to process.

10. **Create SplitInBatches Node (Loop Over Items):**  
    - Incoming data: ads list from Google Sheets  
    - Set batch size to 1 for sequential processing.

11. **Create Switch Node (Video or Image Asset):**  
    - Condition: check asset type field in ad data for "video" or "image".

12. **Video Path:**  
    - Create Facebook Graph API node (Upload Ad Video): upload video file.  
    - Create Switch node to check upload response status.  
    - For 'waiting' status, create Wait node with suitable delay.  
    - Create Facebook Graph API node to check video processing status.  
    - Create Code node to interpret status and decide next step.  
    - Loop upload and wait until processing complete (controlled by If node).  
    - Once complete, create Facebook Graph API nodes to get video preview image, create video ad creative, and create video ad.  
    - Finally, create Google Sheets node to save video ad details.

13. **Image Path:**  
    - Create HTTP Request node to download first image.  
    - Facebook Graph API node to upload first image.  
    - If node to check if one or two images present.  
    - If two images:  
      - HTTP Request node to download second image.  
      - Facebook Graph API node to upload second image.  
      - Facebook Graph API node to create multiple image creative.  
    - If one image:  
      - Facebook Graph API node to create single image creative.  
    - Facebook Graph API node to create image ad.  
    - Google Sheets node to save image ad details.

14. **Connect nodes following the logical flow described.**

15. **Create Webhook node as alternate start point if needed, connected to settings_1 node.**

16. **Configure credentials:**  
    - Google Drive OAuth2 credentials for Drive nodes.  
    - Google Sheets OAuth2 credentials for Sheets nodes.  
    - Facebook Graph API credentials with required permissions for ad account access.

17. **Confirm all nodes set with correct spreadsheet IDs, folder IDs, URLs, and parameters.**

18. **Test workflow with manual trigger and real data.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow automates interaction between Google Sheets, Google Drive, and Meta Ads Manager via API. | Core integration concept for ad automation.                   |
| Facebook Graph API nodes require setup of access tokens with appropriate scopes and permissions. | Meta for Developers: https://developers.facebook.com/docs/apis |
| Google Drive and Sheets nodes require OAuth2 credentials with appropriate scopes enabled.       | Google API Console: https://console.cloud.google.com/apis     |
| Use batching and wait nodes to avoid API rate limits and handle asynchronous video processing.   | n8n documentation on rate limits and execution control.       |

---

**Disclaimer:**  
The provided text is generated solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.