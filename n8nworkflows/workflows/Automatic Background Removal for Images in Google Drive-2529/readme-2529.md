Automatic Background Removal for Images in Google Drive

https://n8nworkflows.xyz/workflows/automatic-background-removal-for-images-in-google-drive-2529


# Automatic Background Removal for Images in Google Drive

### 1. Workflow Overview

This n8n workflow automates the removal of image backgrounds for photos uploaded to Google Drive, utilizing the PhotoRoom API for processing. It is designed for users who manage product images or similar visual content and want to streamline background removal and image formatting without manual editing.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Download:** Watches a specified Google Drive folder for new images and downloads them.
- **1.2 Image Metadata Extraction & Preparation:** Extracts image size information and sets configuration variables such as background color, padding, output size, and API credentials.
- **1.3 Output Size Decision Logic:** Determines whether to keep the original image size or use a fixed output size for processing.
- **1.4 Background Removal Processing:** Calls the PhotoRoom API with appropriate parameters to remove the background and add padding.
- **1.5 Upload Processed Images:** Saves the processed images back to a specified Google Drive folder with a modified filename.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Download

- **Overview:** This block monitors a Google Drive folder for new image files and downloads them for processing.
- **Nodes Involved:**  
  - Watch for new images  
  - Download Image

- **Node Details:**

  - **Watch for new images**  
    - *Type & Role:* Google Drive Trigger; initiates workflow on new file creation in a specified folder.  
    - *Configuration:* Triggers every minute; watches a user-specified folder (must be set via folder ID or URL).  
    - *Input/Output:* No input; outputs metadata about created files including file ID and name.  
    - *Potential Failures:* Auth errors due to invalid Google Drive credentials; folder access permission issues; network timeouts.

  - **Download Image**  
    - *Type & Role:* Google Drive Node; downloads the new image file using file ID.  
    - *Configuration:* Operation set to "download"; file ID dynamically set from trigger output.  
    - *Input/Output:* Input from trigger; outputs binary image data and metadata.  
    - *Potential Failures:* File access denied; file not found if deleted before download; large file size causing timeouts.

---

#### 1.2 Image Metadata Extraction & Preparation

- **Overview:** Extracts image size, splits image metadata for parallel processing, and sets configuration parameters including API key, output folder, and formatting options.
- **Nodes Involved:**  
  - Get Image Size  
  - Split Out  
  - Config

- **Node Details:**

  - **Get Image Size**  
    - *Type & Role:* Edit Image node; retrieves image dimensions and metadata.  
    - *Configuration:* Operation set to "information" to extract size details.  
    - *Input/Output:* Input binary image from previous node; outputs JSON with image size info.  
    - *Edge Cases:* Unsupported image formats; corrupted image files causing failure.

  - **Split Out**  
    - *Type & Role:* Split Out node; separates image metadata fields for conditional logic.  
    - *Configuration:* Splits out the "Geometry" field (image dimension data), includes binary data.  
    - *Input/Output:* Input from image metadata; outputs split data used later in merging.  
    - *Edge Cases:* Missing or malformed geometry data.

  - **Config**  
    - *Type & Role:* Set node; defines workflow-wide configuration variables.  
    - *Configuration:*  
      - `bg_color`: Background color for removal (default "white")  
      - `padding`: Padding around the image (default "5%")  
      - `keepInputSize`: Boolean to toggle output size mode (default true)  
      - `outputSize`: Fixed output size (default "1600x1600")  
      - `inputFileName`: Dynamically set from original filename  
      - `OutputDriveFolder`: Google Drive folder URL for saving processed images (user must set)  
      - `api-key`: PhotoRoom API key (user must set)  
    - *Input/Output:* Inputs image metadata; outputs config variables merged with payload.  
    - *Edge Cases:* Missing API key or output folder configuration will cause downstream failures.

---

#### 1.3 Output Size Decision Logic

- **Overview:** Decides which background removal method to use based on whether the original image size should be preserved or a fixed size should be applied.
- **Nodes Involved:**  
  - loop all over your images  
  - check which output size method is used  
  - Merge

- **Node Details:**

  - **loop all over your images**  
    - *Type & Role:* Split In Batches; processes images in batches for scalability.  
    - *Configuration:* Defaults with no special options.  
    - *Input/Output:* Receives combined data; outputs one image batch at a time.  
    - *Edge Cases:* Very large batches may slow processing; empty input arrays.

  - **check which output size method is used**  
    - *Type & Role:* If node; evaluates `keepInputSize` boolean from config.  
    - *Configuration:* Condition checks if `keepInputSize` is false to choose fixed size processing.  
    - *Input/Output:* Input batch data; outputs to either fixed size or original size processing nodes.  
    - *Edge Cases:* Misconfiguration of boolean may lead to wrong processing path.

  - **Merge**  
    - *Type & Role:* Merge node; combines outputs from different processing branches.  
    - *Configuration:* Mode set to "combine" by position.  
    - *Input/Output:* Inputs from Split Out and Config nodes; outputs combined data for looping.  
    - *Edge Cases:* Misaligned data arrays causing unexpected merges.

---

#### 1.4 Background Removal Processing

- **Overview:** Calls PhotoRoom API to remove the background with either fixed output size or original image size with padding.
- **Nodes Involved:**  
  - remove background fixed size  
  - remove background

- **Node Details:**

  - **remove background fixed size**  
    - *Type & Role:* HTTP Request; sends image to PhotoRoom API with fixed output size.  
    - *Configuration:*  
      - POST to `https://image-api.photoroom.com/v2/edit`  
      - Sends multipart form data including background color, padding, fixed output size, and image binary data.  
      - API key passed in header `x-api-key`.  
    - *Input/Output:* Input is image and config variables; output is processed image binary.  
    - *Edge Cases:* API key invalid; request timeouts; invalid image data causing API errors.

  - **remove background**  
    - *Type & Role:* HTTP Request; similar to above but uses dynamic output size from image metadata (`Geometry`).  
    - *Configuration:* Same as fixed size node but uses dynamic output size field.  
    - *Input/Output:* Input image with original size metadata; output processed image.  
    - *Edge Cases:* Same as above with additional risk if Geometry data is malformed.

---

#### 1.5 Upload Processed Images

- **Overview:** Uploads the background-removed images back to Google Drive in a specified folder with updated filenames.
- **Nodes Involved:**  
  - Upload Picture to Google Drive  
  - Upload Picture to Google Drive1

- **Node Details:**

  - **Upload Picture to Google Drive**  
    - *Type & Role:* Google Drive node; uploads fixed size processed images.  
    - *Configuration:*  
      - Uploads to user-specified Google Drive folder URL.  
      - Filename prepends "BG-Removed-" to the original filename (removes original extension, appends `.png`).  
      - Uses Google Drive OAuth2 credentials.  
    - *Input/Output:* Input is processed image binary; outputs file metadata.  
    - *Edge Cases:* Invalid folder path; upload failures; auth expiry.

  - **Upload Picture to Google Drive1**  
    - *Type & Role:* Google Drive node; uploads original size processed images.  
    - *Configuration:* Same as above node, same filename formatting.  
    - *Input/Output:* Input from dynamic size processing node.  
    - *Edge Cases:* Same as above.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                         | Input Node(s)                    | Output Node(s)                       | Sticky Note                                                      |
|-------------------------------|------------------------|---------------------------------------|---------------------------------|------------------------------------|-----------------------------------------------------------------|
| Watch for new images           | Google Drive Trigger   | Detect new images in Drive folder     | None                            | Download Image, Config              | Select Input Folder                                              |
| Download Image                | Google Drive           | Download image binary                  | Watch for new images             | Get Image Size                     |                                                                 |
| Get Image Size                | Edit Image             | Extract image dimensions               | Download Image                  | Split Out                         |                                                                 |
| Split Out                    | Split Out              | Split geometry data for processing     | Get Image Size                  | Merge                            |                                                                 |
| Config                       | Set                    | Configure parameters and API keys      | Watch for new images            | Merge                            | Configuration instructions and required API key setup          |
| Merge                        | Merge                  | Combine config and geometry data       | Split Out, Config               | loop all over your images          |                                                                 |
| loop all over your images    | Split In Batches       | Process images batch-wise               | Merge                          | check which output size method is used |                                                                 |
| check which output size method is used | If                     | Branch logic on output size preference | loop all over your images       | remove background fixed size, remove background |                                                                 |
| remove background fixed size | HTTP Request           | Call PhotoRoom API with fixed size     | check which output size method is used (true branch) | Upload Picture to Google Drive      |                                                                 |
| remove background            | HTTP Request           | Call PhotoRoom API with original size  | check which output size method is used (false branch) | Upload Picture to Google Drive1     |                                                                 |
| Upload Picture to Google Drive | Google Drive           | Upload processed image (fixed size)    | remove background fixed size    | loop all over your images           |                                                                 |
| Upload Picture to Google Drive1 | Google Drive           | Upload processed image (original size) | remove background              | loop all over your images           |                                                                 |
| Sticky Note1                 | Sticky Note            | Workflow overview and features         | None                           | None                             | About this workflow, features, usage, and API playground link   |
| Sticky Note2                 | Sticky Note            | Setup instructions and config notes    | None                           | None                             | Setup requirements and config instructions                      |
| Sticky Note3                 | Sticky Note            | Input folder selection guidance         | None                           | None                             | Select Input Folder                                              |
| Sticky Note4                 | Sticky Note            | Configuration details                   | None                           | None                             | Configuration instructions including API key and output folder |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node ("Watch for new images")**  
   - Set event to "fileCreated".  
   - Set polling interval to every minute.  
   - Configure to watch a specific folder (set folder ID or URL).  
   - Connect Google Drive OAuth2 credentials.

2. **Add Google Drive node ("Download Image")**  
   - Set operation to "download".  
   - Set file ID to `{{$json.id}}` from trigger output.  
   - Connect Google Drive OAuth2 credentials.  
   - Connect output of "Watch for new images" to this node.

3. **Add Edit Image node ("Get Image Size")**  
   - Set operation to "information".  
   - Connect output of "Download Image".

4. **Add Split Out node ("Split Out")**  
   - Set field to split out to "Geometry".  
   - Enable option to include binary data.  
   - Connect output of "Get Image Size".

5. **Add Set node ("Config")**  
   - Add variables:  
     - `bg_color` (string): default "white"  
     - `padding` (string): default "5%"  
     - `keepInputSize` (boolean): default true  
     - `outputSize` (string): default "1600x1600"  
     - `inputFileName` (string): expression `{{$json.originalFilename}}`  
     - `OutputDriveFolder` (string): user must input Google Drive folder URL  
     - `api-key` (string): user must input PhotoRoom API key  
   - Connect output of "Watch for new images" (parallel to "Download Image").

6. **Add Merge node ("Merge")**  
   - Set mode to "combine" by position.  
   - Connect "Split Out" as first input and "Config" as second input.

7. **Add Split In Batches node ("loop all over your images")**  
   - Default settings.  
   - Connect output of "Merge".

8. **Add If node ("check which output size method is used")**  
   - Condition: check if `keepInputSize` is false (boolean false).  
   - Connect output of "loop all over your images".

9. **Add HTTP Request node ("remove background fixed size")**  
   - Method: POST  
   - URL: `https://image-api.photoroom.com/v2/edit`  
   - Content Type: multipart-form-data  
   - Body parameters:  
     - `background.color`: `{{$json.bg_color}}`  
     - `imageFile`: binary data from input field named "data"  
     - `padding`: `{{$json.padding}}`  
     - `outputSize`: `{{$json.outputSize}}`  
   - Header parameters:  
     - `x-api-key`: `{{$json['api-key']}}`  
   - Connect IF node’s **true** branch to this node.

10. **Add HTTP Request node ("remove background")**  
    - Same configuration as node above, except set `outputSize` parameter to `{{$json.Geometry}}`.  
    - Connect IF node’s **false** branch to this node.

11. **Add Google Drive node ("Upload Picture to Google Drive")**  
    - Operation: upload  
    - Name: expression `BG-Removed-{{$json.inputFileName.split('.').slice(0, -1).join('.') }}.png`  
    - Drive ID: "My Drive"  
    - Folder ID: expression `{{$json.OutputDriveFolder}}`  
    - Connect Google Drive OAuth2 credentials.  
    - Connect output of "remove background fixed size" to this node.

12. **Add Google Drive node ("Upload Picture to Google Drive1")**  
    - Same configuration as above node.  
    - Connect output of "remove background" node.

13. **Connect outputs of both Upload nodes back to "loop all over your images" node’s second output** to continue batch processing.

14. **Test the workflow** by uploading an image into the configured Google Drive input folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow supports transparent or colored background removal with padding and output size customization.  | Workflow description and features section                                                      |
| PhotoRoom API Playground allows testing API parameters interactively.                                         | https://www.photoroom.com/api/playground                                                      |
| Users must set up Google Drive OAuth2 credentials and PhotoRoom API key before running the workflow.          | Setup instructions in Sticky Note2 and Config node                                            |
| For customization, users can modify background color, padding, output size, and output folder easily.         | Config Node and Sticky Note4                                                                   |
| Additional features can be integrated, such as AI-based product type analysis or adding captions with nodes.  | Suggested in Sticky Note1                                                                      |

---

This documentation fully describes the workflow’s logic, node configurations, and setup instructions, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.