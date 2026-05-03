Create PDF from Images for free via Google Slides and Google Drive

https://n8nworkflows.xyz/workflows/create-pdf-from-images-for-free-via-google-slides-and-google-drive-4540


# Create PDF from Images for free via Google Slides and Google Drive

### 1. Workflow Overview

This workflow automates the creation of a PDF document from images stored in a specified Google Drive folder by using Google Slides as an intermediary. It takes all images from a selected folder, sorts them by creation date, copies a Google Slides template, creates slides for each image, adds images to these slides, deletes the initial empty slide, converts the final presentation to PDF, and uploads the PDF back to Google Drive.

**Target Use Case:**  
Users who want to compile a set of images from Google Drive into a customized PDF document without manual slide creation, leveraging Google Slides for layout and image placement, and producing the PDF entirely through Google APIs.

The workflow is logically divided into these blocks:

- **1.1 Input & Initialization:** Manual trigger, setting PDF file name, copying the Slides template, and preparing the presentation.
- **1.2 Image Retrieval & Sorting:** Fetching all files from the folder, filtering for images, and sorting them by creation date.
- **1.3 Slide Creation & Image Insertion Loop:** For each image, create a slide, update image permissions, add the image to the slide.
- **1.4 Cleanup & PDF Generation:** Deleting the first empty slide, converting Slides to PDF, and uploading the final PDF file.
- **1.5 Auxiliary Notes:** Sticky notes providing detailed user instructions and context.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Initialization

**Overview:**  
This block starts the workflow manually, sets the desired PDF file name, copies a Google Slides template to create a new presentation, and retrieves details about this new presentation.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set Pdf File Name  
- CopyPdfTemplate  
- Get Created Presentation  
- Get Folder Id w Presentation

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: None required.  
  - Inputs/Outputs: Output triggers "Set Pdf File Name".  
  - Potential Failures: None; manual trigger node.

- **Set Pdf File Name**  
  - Type: Set  
  - Role: Defines the desired PDF filename as `presentation_title`.  
  - Configuration: Assigns string `[Your PDF Name Here]` to variable `presentation_title`.  
  - Inputs: From manual trigger.  
  - Outputs: To "CopyPdfTemplate".  
  - Edge Cases: User must customize the filename before running.

- **CopyPdfTemplate**  
  - Type: Google Drive - Copy File  
  - Role: Copies a predefined Google Slides template to serve as the base presentation.  
  - Configuration:  
    - Template file selected from a list (default ID: `1_Mfgb7lUD8tXRg3cXeIQXjQ9vgCG-TzwOoWVok_ROQ8`)  
    - Destination folder is empty (defaults to My Drive)  
    - New file named using `presentation_title`.  
  - Inputs: From "Set Pdf File Name".  
  - Outputs: To "Get Created Presentation".  
  - Credentials: Google Drive OAuth2.  
  - Edge Cases:  
    - If the template ID is invalid or inaccessible, copy will fail.  
    - Folder must be writable.

- **Get Created Presentation**  
  - Type: Google Slides - Get Presentation  
  - Role: Retrieves details (e.g., slide IDs, page size) of the newly copied presentation.  
  - Configuration: Uses copied presentation ID from "CopyPdfTemplate".  
  - Inputs: From "CopyPdfTemplate".  
  - Outputs: To "Get Folder Id w Presentation".  
  - Credentials: Google Slides OAuth2.  
  - Edge Cases: If the presentation is deleted or inaccessible immediately after copying, retrieval may fail.

- **Get Folder Id w Presentation**  
  - Type: HTTP Request (Google Drive API)  
  - Role: Extracts the parent folder ID of the copied presentation file.  
  - Configuration: Calls `GET https://www.googleapis.com/drive/v3/files/{presentationId}?fields=parents`.  
  - Inputs: From "Get Created Presentation".  
  - Outputs: To "Get All Files From the Folder".  
  - Credentials: Google Drive OAuth2.  
  - Edge Cases: If the presentation file lacks a parent folder or API quota limits are exceeded.

---

#### 1.2 Image Retrieval & Sorting

**Overview:**  
Fetches all files from the folder containing the copied presentation, filters for images with MIME type `image/png`, then sorts these images by creation date.

**Nodes Involved:**  
- Get All Files From the Folder  
- Filter: Only Images  
- Sort by Created Date

**Node Details:**

- **Get All Files From the Folder**  
  - Type: Google Drive - List Files/Folders  
  - Role: Lists all files in the folder identified previously (where images are stored).  
  - Configuration: Filters by folder ID extracted from previous node; returns all files.  
  - Inputs: From "Get Folder Id w Presentation".  
  - Outputs: To "Filter: Only Images".  
  - Credentials: Google Drive OAuth2.  
  - Edge Cases: Large folder with many files may cause performance issues; API rate limits.

- **Filter: Only Images**  
  - Type: Filter  
  - Role: Filters items to only include images of MIME type `image/png`.  
  - Configuration: Condition checks if `mimeType` equals `image/png`.  
  - Inputs: From "Get All Files From the Folder".  
  - Outputs: To "Sort by Created Date".  
  - Edge Cases:  
    - Only PNG images included by default.  
    - User must modify filter to add other image types (e.g., `image/jpeg` for JPG).  
    - May exclude valid image files if MIME types differ.

- **Sort by Created Date**  
  - Type: Sort  
  - Role: Sorts filtered images by their `createdTime` ascending.  
  - Configuration: Sort field is `createdTime`.  
  - Inputs: From "Filter: Only Images".  
  - Outputs: To "Loop Over Items".  
  - Edge Cases: If `createdTime` field is missing or inconsistent, sorting may be unreliable.

---

#### 1.3 Slide Creation & Image Insertion Loop

**Overview:**  
Iterates over the sorted images, for each image creates a blank slide, updates the image sharing permissions for public access, and inserts the image into the slide fitting the slide dimensions.

**Nodes Involved:**  
- Loop Over Items  
- Limit: 1Page  
- Create An Empty Slide  
- Update Image Permissions  
- Add Image To The Slide

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes images one by one in sequence to manage API calls and limits.  
  - Configuration: Default batch size (1 item per batch).  
  - Inputs: From "Sort by Created Date".  
  - Outputs: Two parallel outputs: one goes to "Limit: 1Page", the other to "Create An Empty Slide".  
  - Edge Cases: Large image sets may slow down processing.

- **Limit: 1Page**  
  - Type: Limit  
  - Role: Keeps the last item only; used to ensure the chain of slide creation completes properly.  
  - Inputs: From "Loop Over Items".  
  - Outputs: To "Delete First Empty Slide".  
  - Edge Cases: Ensures final steps only run after last image processed.

- **Create An Empty Slide**  
  - Type: HTTP Request (Google Slides API)  
  - Role: Sends a batch update request to add a new blank slide to the copied presentation.  
  - Configuration: POST to `https://slides.googleapis.com/v1/presentations/{presentationId}:batchUpdate` with JSON to create a blank slide.  
  - Inputs: From "Loop Over Items".  
  - Outputs: To "Update Image Permissions".  
  - Credentials: Google Slides OAuth2.  
  - Edge Cases: API limits, permission errors, or invalid presentation ID.

- **Update Image Permissions**  
  - Type: Google Drive - Share File  
  - Role: Updates the current image’s sharing permissions to `anyone with link can read` to allow Google Slides to access it.  
  - Configuration: File ID from current image; permission role `reader`, type `anyone`.  
  - Inputs: From "Create An Empty Slide".  
  - Outputs: To "Add Image To The Slide".  
  - Credentials: Google Drive OAuth2.  
  - Edge Cases: Permission update failures, file access errors.

- **Add Image To The Slide**  
  - Type: HTTP Request (Google Slides API)  
  - Role: Inserts the image URL into the newly created slide, scaling it to slide dimensions.  
  - Configuration:  
    - Uses batchUpdate POST to Google Slides API with `createImage` request.  
    - Image URL from image’s `webContentLink`.  
    - Slide size and object IDs dynamically retrieved from previous nodes.  
  - Inputs: From "Update Image Permissions".  
  - Outputs: Loops back to "Loop Over Items" to process next image.  
  - Credentials: Google Slides OAuth2.  
  - Edge Cases: URL accessibility, image size limits, API quota.

---

#### 1.4 Cleanup & PDF Generation

**Overview:**  
After all images have been added as slides, the initial empty slide is deleted, the Google Slides presentation is exported as PDF, and the PDF is uploaded to the same Google Drive folder.

**Nodes Involved:**  
- Delete First Empty Slide  
- Convert slides to PDF  
- Upload Final PDF File

**Node Details:**

- **Delete First Empty Slide**  
  - Type: HTTP Request (Google Slides API)  
  - Role: Removes the first slide (usually blank from the template) from the presentation.  
  - Configuration: POST to batchUpdate with `deleteObject` targeting first slide’s objectId.  
  - Inputs: From "Limit: 1Page".  
  - Outputs: To "Convert slides to PDF".  
  - Credentials: Google Slides OAuth2.  
  - Edge Cases: If slide objectId is missing or slide already deleted.

- **Convert slides to PDF**  
  - Type: Google Drive - Download File  
  - Role: Downloads the presentation file converted to PDF format.  
  - Configuration: File ID of copied presentation, conversion format set to PDF.  
  - Inputs: From "Delete First Empty Slide".  
  - Outputs: To "Upload Final PDF File".  
  - Credentials: Google Drive OAuth2.  
  - Edge Cases: File access issues, API conversion limits.

- **Upload Final PDF File**  
  - Type: Google Drive - Upload File  
  - Role: Uploads the generated PDF back into the same Google Drive folder as the images and copied presentation.  
  - Configuration:  
    - Filename from `presentation_title`.  
    - Folder ID from "Get Folder Id w Presentation".  
    - Uploads to "My Drive" or specified folder.  
  - Inputs: From "Convert slides to PDF".  
  - Credentials: Google Drive OAuth2.  
  - Edge Cases: Upload quota, folder write permissions.

---

#### 1.5 Auxiliary Notes

**Overview:**  
Sticky notes provide detailed instructions and important context for users about configuring the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**

- **Sticky Note**  
  - Purpose: Overall workflow explanation, usage tips, limitations, and cost info.  
  - Content highlights:  
    - How to set PDF filename  
    - Importance of Google account authentication  
    - Using a custom Google Slides template for page size  
    - Default image filtering and how to modify it  
    - Workflow limits for image/PDF size  
    - Free to use relying on Google API free tier.

- **Sticky Note2**  
  - Purpose: Instructions for configuring the Slides template and image folder in the "CopyPdfTemplate" node.  
  - Content highlights:  
    - Select Slides template file  
    - Select Google Drive folder for images  
    - Final PDF will be saved in same folder.

- **Sticky Note3**  
  - Purpose: Explains image filter configuration.  
  - Content highlights:  
    - Default filter is for PNG images  
    - Instructions on changing MIME type for other image formats like JPG.

- **Sticky Note4**  
  - Purpose: Explains the stepwise image-to-slide process.  
  - Content highlights:  
    - Fetching images  
    - Sorting by creation date  
    - For each image: create slide, update permissions, add image  
    - Deleting first empty slide after processing.

---

### 3. Summary Table

| Node Name                | Node Type                    | Functional Role                                      | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                                       |
|--------------------------|------------------------------|-----------------------------------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger               | Manual start trigger                                |                              | Set Pdf File Name              |                                                                                                                  |
| Set Pdf File Name         | Set                          | Sets the PDF filename variable                      | When clicking ‘Test workflow’ | CopyPdfTemplate               |                                                                                                                  |
| CopyPdfTemplate           | Google Drive - Copy File     | Copies Google Slides template to new presentation  | Set Pdf File Name             | Get Created Presentation       | Sticky Note2: Configure template & image folder; select template & folder in this node.                          |
| Get Created Presentation  | Google Slides - Get          | Retrieves details of copied presentation            | CopyPdfTemplate              | Get Folder Id w Presentation   |                                                                                                                  |
| Get Folder Id w Presentation | HTTP Request (Google Drive API) | Gets parent folder ID of presentation                | Get Created Presentation     | Get All Files From the Folder  |                                                                                                                  |
| Get All Files From the Folder | Google Drive - List Files   | Lists all files in the target folder                | Get Folder Id w Presentation | Filter: Only Images            |                                                                                                                  |
| Filter: Only Images       | Filter                       | Filters files to only include PNG images            | Get All Files From the Folder | Sort by Created Date          | Sticky Note3: Image filter info; update MIME type here to support other formats.                                |
| Sort by Created Date      | Sort                         | Sorts images by creation date                        | Filter: Only Images          | Loop Over Items                |                                                                                                                  |
| Loop Over Items           | Split In Batches             | Processes images one at a time                       | Sort by Created Date         | Limit: 1Page, Create An Empty Slide | Sticky Note4: Describes image-to-slide process loop.                                                        |
| Limit: 1Page              | Limit                        | Keeps last item for final cleanup                    | Loop Over Items              | Delete First Empty Slide       |                                                                                                                  |
| Create An Empty Slide     | HTTP Request (Google Slides API) | Creates new blank slide for each image               | Loop Over Items              | Update Image Permissions       |                                                                                                                  |
| Update Image Permissions  | Google Drive - Share File    | Sets image file sharing to public read              | Create An Empty Slide        | Add Image To The Slide         |                                                                                                                  |
| Add Image To The Slide    | HTTP Request (Google Slides API) | Adds image URL to created slide                       | Update Image Permissions     | Loop Over Items                |                                                                                                                  |
| Delete First Empty Slide  | HTTP Request (Google Slides API) | Deletes the initial blank slide                       | Limit: 1Page                | Convert slides to PDF          |                                                                                                                  |
| Convert slides to PDF     | Google Drive - Download File | Exports presentation as PDF                           | Delete First Empty Slide     | Upload Final PDF File          |                                                                                                                  |
| Upload Final PDF File     | Google Drive - Upload File   | Uploads the generated PDF to Drive                    | Convert slides to PDF        |                               |                                                                                                                  |
| Sticky Note               | Sticky Note                  | General workflow overview and instructions           |                              |                               | Sticky Note: Full workflow explanation and usage tips.                                                         |
| Sticky Note2              | Sticky Note                  | Template & folder configuration instructions         |                              |                               | Sticky Note2: Explains "CopyPdfTemplate" settings.                                                              |
| Sticky Note3              | Sticky Note                  | Image filter configuration instructions              |                              |                               | Sticky Note3: Explains image filtering by MIME type.                                                            |
| Sticky Note4              | Sticky Note                  | Image-to-slide process explanation                    |                              |                               | Sticky Note4: Describes slide creation and image insertion steps.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Start workflow manually.

2. **Add a Set node**  
   - Name: `Set Pdf File Name`  
   - Add field `presentation_title` as string, default value: `[Your PDF Name Here]`  
   - Connect output from manual trigger to this node.

3. **Add Google Drive node (Copy File)**  
   - Name: `CopyPdfTemplate`  
   - Operation: Copy  
   - File: Select your Google Slides template file (must be previously created with desired page size)  
   - Folder: Select folder containing source images (also destination for copied file)  
   - Name: Use expression `{{$json.presentation_title}}` to name the copied presentation  
   - Connect from `Set Pdf File Name` node.  
   - Configure Google Drive OAuth2 credentials.

4. **Add Google Slides node (Get Presentation)**  
   - Name: `Get Created Presentation`  
   - Operation: Get  
   - Presentation ID: Use expression `{{$json.id}}` from `CopyPdfTemplate` output  
   - Connect from `CopyPdfTemplate` node.  
   - Authenticate with Google Slides OAuth2.

5. **Add HTTP Request node**  
   - Name: `Get Folder Id w Presentation`  
   - Method: GET  
   - URL: `https://www.googleapis.com/drive/v3/files/{{$json.presentationId}}?fields=parents`  
   - Authentication: Google Drive OAuth2  
   - Connect from `Get Created Presentation`.

6. **Add Google Drive node (List Files/Folders)**  
   - Name: `Get All Files From the Folder`  
   - Resource: File/Folder  
   - Filter: Folder ID from `Get Folder Id w Presentation` output `parents[0]`  
   - Return all files  
   - Connect from `Get Folder Id w Presentation`.

7. **Add Filter node**  
   - Name: `Filter: Only Images`  
   - Condition: `mimeType` equals `image/png` (modify as needed for other image formats)  
   - Connect from `Get All Files From the Folder`.

8. **Add Sort node**  
   - Name: `Sort by Created Date`  
   - Sort by field: `createdTime` ascending  
   - Connect from `Filter: Only Images`.

9. **Add Split In Batches node**  
   - Name: `Loop Over Items`  
   - Batch size: 1 (default)  
   - Connect from `Sort by Created Date`.

10. **Add Limit node**  
    - Name: `Limit: 1Page`  
    - Keep: last item  
    - Connect from first output of `Loop Over Items`.

11. **Add HTTP Request node**  
    - Name: `Delete First Empty Slide`  
    - Method: POST  
    - URL: `https://slides.googleapis.com/v1/presentations/{{$json.presentationId}}:batchUpdate`  
    - Body (JSON):  
      ```json
      {
        "requests": [
          {
            "deleteObject": {
              "objectId": "{{$json.slides[0].objectId}}"
            }
          }
        ]
      }
      ```  
    - Authentication: Google Slides OAuth2  
    - Connect from `Limit: 1Page`.

12. **Add Google Drive node (Download File)**  
    - Name: `Convert slides to PDF`  
    - Operation: Download  
    - File ID: Use copied presentation ID  
    - Options: Conversion set to PDF for slides  
    - Connect from `Delete First Empty Slide`.

13. **Add Google Drive node (Upload File)**  
    - Name: `Upload Final PDF File`  
    - Name: Use expression `{{$json.presentation_title}}`  
    - Folder: Use folder ID from `Get Folder Id w Presentation`  
    - Connect from `Convert slides to PDF`.

14. **Add HTTP Request node**  
    - Name: `Create An Empty Slide`  
    - Method: POST  
    - URL: `https://slides.googleapis.com/v1/presentations/{{$json.presentationId}}:batchUpdate`  
    - Body (JSON):  
      ```json
      {
        "requests": [
          {
            "createSlide": {
              "slideLayoutReference": {
                "predefinedLayout": "BLANK"
              }
            }
          }
        ]
      }
      ```  
    - Authentication: Google Slides OAuth2  
    - Connect from second output of `Loop Over Items`.

15. **Add Google Drive node (Share File)**  
    - Name: `Update Image Permissions`  
    - Operation: Share  
    - File ID: Use current image ID from batch  
    - Permissions: Role = reader, type = anyone  
    - Connect from `Create An Empty Slide`.

16. **Add HTTP Request node**  
    - Name: `Add Image To The Slide`  
    - Method: POST  
    - URL: `https://slides.googleapis.com/v1/presentations/{{$json.presentationId}}:batchUpdate`  
    - Body (JSON):  
      Construct JSON dynamically to create image on slide using:  
      - Image URL: current image `webContentLink`  
      - Slide objectId: from `Create An Empty Slide` response  
      - Slide size: from `Get Created Presentation` page size info  
    - Authentication: Google Slides OAuth2  
    - Connect from `Update Image Permissions`.  
    - Output loops back to `Loop Over Items` to continue batch processing.

17. **Add Sticky Notes (Optional)**  
    - Add sticky note nodes with instructions as per user needs for workflow clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow relies on Google API quotas and permissions. Ensure your Google Drive and Slides OAuth2 credentials have appropriate scopes. Google Slides API has limitations on image sizes and number of slides; large batches may cause errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow performance and limits                                                                                   |
| Use a custom Google Slides template with desired page size via File > Page Setup. This controls the PDF page dimensions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Google Slides page size configuration                                                                             |
| For image formats other than PNG, update the Filter node condition with the correct MIME type (e.g., `image/jpeg` for JPG).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Image format customization                                                                                        |
| The workflow sets public read permission on images temporarily to allow Google Slides API to insert them via URL. Permissions can be revoked manually after workflow runs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Image permission management                                                                                        |
| The workflow is free to use and falls within Google API free tier limits for typical usage.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Cost information                                                                                                  |
| Google Slides API docs: https://developers.google.com/slides/api/guides/batchupdates                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Official Google Slides API documentation                                                                           |
| Google Drive API docs: https://developers.google.com/drive/api/v3/reference/files                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Official Google Drive API documentation                                                                            |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.