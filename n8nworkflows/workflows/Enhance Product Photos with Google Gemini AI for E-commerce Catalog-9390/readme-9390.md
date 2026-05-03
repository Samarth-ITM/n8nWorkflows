Enhance Product Photos with Google Gemini AI for E-commerce Catalog

https://n8nworkflows.xyz/workflows/enhance-product-photos-with-google-gemini-ai-for-e-commerce-catalog-9390


# Enhance Product Photos with Google Gemini AI for E-commerce Catalog

### 1. Workflow Overview

This workflow automates the enhancement of product photos for an e-commerce catalog by leveraging Google Gemini AI's image editing capabilities. It monitors a specific Google Drive folder for new or updated image files, processes these images by transforming them according to a detailed prompt, and saves the enhanced images back to a designated Google Drive folder. Throughout the process, it maintains a log in a Google Sheet to track the status and metadata of each processed image.

**Target Use Cases:**

- Batch processing of product photos to standardize style and quality.
- Automating image enhancement workflows for e-commerce catalogs.
- Extensible to other image transformation scenarios (e.g., team photos).

**Logical Blocks:**

- **1.1 Input Reception:** Detect new or updated image files in a monitored Google Drive folder.
- **1.2 Configuration Setup:** Define workflow-wide parameters such as Google Sheet ID, output folder, and image transformation prompt.
- **1.3 Processing Logging:** Create and update entries in Google Sheets to track processing status.
- **1.4 Image Downloading:** Download the detected image file from Google Drive.
- **1.5 AI Image Transformation:** Use the Google Gemini AI to enhance the image based on the specified prompt.
- **1.6 Saving Output:** Upload the transformed image back to the Google Drive output folder.
- **1.7 Status Update:** Update the Google Sheet entry to reflect success or failure of processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block watches a specific Google Drive folder for newly created or updated files, triggering the workflow when changes occur.

**Nodes Involved:**  
- File Created  
- File Updated  
- Set File ID

**Node Details:**

- **File Created**  
  - Type: Google Drive Trigger  
  - Role: Trigger workflow on new file creation.  
  - Config: Watches a specific folder (folder ID to be configured) every 5 minutes for any new files of any type.  
  - Input/Output: Trigger node, outputs file metadata JSON.  
  - Failure modes: API rate limits, folder ID misconfiguration, Google OAuth token expiration.

- **File Updated**  
  - Type: Google Drive Trigger  
  - Role: Trigger workflow on file updates.  
  - Config: Watches the same folder every 5 minutes for file updates.  
  - Input/Output: Trigger node, outputs updated file metadata JSON.  
  - Failure modes: Same as File Created node.

- **Set File ID**  
  - Type: Set  
  - Role: Extracts and sets key file properties (ID, name, MIME type, URL, name without extension) for downstream use.  
  - Config: Uses expressions to parse file metadata.  
  - Inputs: From either File Created or File Updated triggers.  
  - Outputs: JSON with standardized keys (`input_file_id`, `input_file_name`, etc.).  
  - Failure modes: Expression evaluation errors if input data missing or malformed.

---

#### 1.2 Configuration Setup

**Overview:**  
Defines workflow-wide constants such as the Google Sheet ID for logging, destination Google Drive folder ID for output images, and the text prompt guiding image transformation.

**Nodes Involved:**  
- Workflow Configuration

**Node Details:**

- **Workflow Configuration**  
  - Type: Set  
  - Role: Holds configurable parameters as variables for the workflow.  
  - Config:  
    - `google_sheet_id`: ID of the Google Sheet used for logging.  
    - `dest_folder_id`: ID of the Google Drive folder where enhanced images are saved.  
    - `text_prompt`: Detailed instructions for image enhancement (background removal, lighting, color correction, etc.).  
  - Inputs: From Set File ID node.  
  - Outputs: JSON containing configuration parameters.  
  - Failure modes: Incorrect or missing IDs will cause downstream node failures.

---

#### 1.3 Processing Logging

**Overview:**  
Manages creating and updating rows in a Google Sheet to track the status and metadata of each image processing job.

**Nodes Involved:**  
- Create Entry  
- Update Entry to Done  
- Update Entry to Error

**Node Details:**

- **Create Entry**  
  - Type: Google Sheets (Append or Update)  
  - Role: Adds a new row at the start of processing with status "Not Started" and timestamps.  
  - Config:  
    - Sheet named `Photos`.  
    - Columns: File name, Status, Start Time, Input File URL.  
    - Uses `Input File` column as unique row identifier.  
  - Inputs: From Workflow Configuration node.  
  - Outputs: Passes data for next steps.  
  - Failure modes: Google Sheets API errors, permission issues, incorrect Sheet ID.

- **Update Entry to Done**  
  - Type: Google Sheets (Append or Update)  
  - Role: Updates the row to mark processing as "Completed" with end time and links to input/output files.  
  - Config: Same sheet and matching column as Create Entry.  
  - Inputs: From Save image node (successful output).  
  - Failure modes: Same as Create Entry.

- **Update Entry to Error**  
  - Type: Google Sheets (Append or Update)  
  - Role: Updates the row to mark processing as "Error" with end time and file links, triggered on processing failure.  
  - Inputs: From Edit Image node error output.  
  - Failure modes: Same as above.

---

#### 1.4 Image Downloading

**Overview:**  
Downloads the image file identified by the trigger from Google Drive to prepare it for AI processing.

**Nodes Involved:**  
- Download Image

**Node Details:**

- **Download Image**  
  - Type: Google Drive (Download)  
  - Role: Retrieves the binary content of the image file.  
  - Config: Uses `input_file_id` from Set File ID node.  
  - Inputs: From Create Entry node (which follows Workflow Configuration).  
  - Outputs: Binary data of the image under property `input_file_data`.  
  - Failure modes: File not found, permissions issues, network timeout.

---

#### 1.5 AI Image Transformation

**Overview:**  
Uses Google Gemini AI to edit the image based on the detailed prompt, resulting in an enhanced product photo.

**Nodes Involved:**  
- Edit Image

**Node Details:**

- **Edit Image**  
  - Type: Google Gemini (Langchain node)  
  - Role: Performs image editing using Google Gemini AI.  
  - Config:  
    - Input: Binary image data (`input_file_data`).  
    - Prompt: Text from `Workflow Configuration.text_prompt`.  
    - Output: Binary enhanced image data (`output_file_data`).  
    - On error: Continues execution to error handling node.  
  - Credentials: Requires Google Gemini (PaLM) API credentials configured in the node.  
  - Inputs: From Download Image node.  
  - Outputs: Binary edited image data.  
  - Failure modes: API authentication failures, timeouts, prompt errors, rate limits.

---

#### 1.6 Saving Output

**Overview:**  
Uploads the enhanced image back to a specified Google Drive folder with a modified file name.

**Nodes Involved:**  
- Save image

**Node Details:**

- **Save image**  
  - Type: Google Drive (Upload)  
  - Role: Uploads the transformed image to the destination folder.  
  - Config:  
    - File name: Original name without extension + `_clean` + original extension.  
    - Destination folder ID from `Workflow Configuration.dest_folder_id`.  
    - Input data: Binary `output_file_data` from Edit Image node.  
  - Inputs: From Edit Image node (success output).  
  - Outputs: Metadata of uploaded file including `webContentLink`.  
  - Failure modes: Permission denied, file size limits, network errors.

---

#### 1.7 Status Update

**Overview:**  
Finalizes the workflow by updating the Google Sheet entry to reflect the result of the image processing job.

**Nodes Involved:**  
- Update Entry to Done  
- Update Entry to Error

**Node Details:**

- **Update Entry to Done**  
  - As described in Block 1.3 above, triggered after successful save.

- **Update Entry to Error**  
  - As described in Block 1.3 above, triggered on processing error from Edit Image node.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                                | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                                                                                                                                                 |
|---------------------|---------------------------|------------------------------------------------|---------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| File Created        | Google Drive Trigger       | Trigger on new files in target folder           | -                         | Set File ID                   | ## 1. Watch for New and Updated Images in Google Drive Folder [Google Drive Trigger docs](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googledrivetrigger/) [Google Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/) Other sources possible. |
| File Updated        | Google Drive Trigger       | Trigger on updated files in target folder       | -                         | Set File ID                   | Same as File Created                                                                                                                                                                                                        |
| Set File ID         | Set                       | Extracts and sets file metadata variables       | File Created, File Updated | Workflow Configuration        |                                                                                                                                                                                                                             |
| Workflow Configuration | Set                     | Defines workflow parameters (Sheet ID, folder, prompt) | Set File ID                | Create Entry                  | ## ⚙️ Update the Workflow Configuration Node - Set `google_sheet_id`, `dest_folder_id`, and `text_prompt` (customizable)                                                                                                   |
| Create Entry        | Google Sheets              | Logs new processing job with status 'Not Started' | Workflow Configuration     | Download Image                | ## 2. Create Entry in Google Sheet - Tracks progress with unique URL identifier                                                                                                                                             |
| Download Image      | Google Drive               | Downloads image binary data                       | Create Entry               | Edit Image                   |                                                                                                                                                                                                                             |
| Edit Image          | Google Gemini (Langchain)  | AI image transformation                          | Download Image             | Save image / Update Entry to Error | ## 3. Transform the Image [Google Gemini node docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.googlegemini/) [Image generation docs](https://ai.google.dev/gemini-api/docs/image-generation) |
| Save image          | Google Drive               | Uploads transformed image to output folder       | Edit Image (success)       | Update Entry to Done          |                                                                                                                                                                                                                             |
| Update Entry to Done | Google Sheets              | Updates log entry to 'Completed'                  | Save image                 | -                           | ## 4. Update entry in Google Sheet                                                                                                                                                                                          |
| Update Entry to Error| Google Sheets              | Updates log entry to 'Error'                       | Edit Image (error)         | -                           |                                                                                                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Triggers:**
   - Add two Google Drive Trigger nodes named `File Created` and `File Updated`.
   - Configure both to watch the same target folder by its folder ID.
   - Set polling interval to every 5 minutes.
   - Ensure Google Drive OAuth2 credentials are properly configured.

2. **Add a Set Node to Extract File Metadata:**
   - Create a `Set File ID` node.
   - Use expressions to assign the following variables from trigger data:  
     - `input_file_id` = `{{$json.id}}`  
     - `input_file_name` = `{{$json.name}}`  
     - `input_file_name_no_ext` = `{{$json.name.split('.').slice(0, -1).join('.')}}`  
     - `input_mime_type` = `{{$json.mimeType}}`  
     - `input_file_url` = `{{$json.webContentLink}}`
   - Connect outputs of both triggers to this node.

3. **Add a Configuration Set Node:**
   - Create `Workflow Configuration` node as Set.
   - Define variables:  
     - `google_sheet_id` (string): Google Sheet ID for logging.  
     - `dest_folder_id` (string): Google Drive output folder ID.  
     - `text_prompt` (string): Detailed prompt for image editing (copy from workflow or customize).  
   - Connect `Set File ID` node output to this node.

4. **Create Google Sheets Append/Update Node (Create Entry):**
   - Add `Create Entry` Google Sheets node.
   - Configure to append or update in sheet named `Photos`.  
   - Map columns: `File name`, `Status` (set to "Not Started"), `Start Time` (current timestamp), `Input File` (file URL).  
   - Set `Input File` as the unique matching column.  
   - Connect `Workflow Configuration` output to this node.

5. **Add Google Drive Download Node:**
   - Create `Download Image` node.
   - Set operation to download file by ID from `input_file_id` variable.  
   - Configure to output binary data under `input_file_data`.  
   - Connect `Create Entry` output to this node.

6. **Add Google Gemini AI Edit Image Node:**
   - Add `Edit Image` node (Google Gemini Langchain).
   - Set resource to `image`, operation to `edit`.  
   - Configure images input to pass binary `input_file_data`.  
   - Set prompt to `text_prompt` variable from configuration.  
   - Configure output binary property as `output_file_data`.  
   - Attach Google Gemini (PaLM) credentials.  
   - Connect `Download Image` output to this node.

7. **Add Google Drive Upload Node:**
   - Add `Save image` node.
   - Configure upload destination folder ID from `dest_folder_id`.  
   - Set file name to `{{input_file_name_no_ext}}_clean.{{fileExtension}}` (extract extension as needed).  
   - Input binary data from `output_file_data`.  
   - Connect successful output of `Edit Image` node here.

8. **Add Google Sheets Append/Update Node for Success Status:**
   - Add `Update Entry to Done` node.
   - Configure to update the same sheet and row as `Create Entry` based on `Input File` URL.  
   - Update columns: `Status` = "Completed", `End Time` = current timestamp, `Output File` = the uploaded file web link.  
   - Connect output of `Save image` node to this node.

9. **Add Google Sheets Append/Update Node for Error Status:**
   - Add `Update Entry to Error` node.
   - Configure similar to success node but with `Status` = "Error".  
   - Connect error output of `Edit Image` node to this node.

10. **Connect Triggers to Set File ID Node:**
    - Connect both `File Created` and `File Updated` nodes to `Set File ID`.

11. **Establish Remaining Connections:**
    - `Set File ID` → `Workflow Configuration` → `Create Entry` → `Download Image` → `Edit Image` → `Save image` → `Update Entry to Done`.
    - Error output of `Edit Image` → `Update Entry to Error`.

12. **Credential Setup:**
    - Configure Google Drive and Google Sheets OAuth2 credentials and assign to respective nodes.
    - Configure Google Gemini (PaLM) API credentials and assign to the `Edit Image` node.

13. **Final Configuration:**
    - Update folder IDs and Google Sheet ID in the `Workflow Configuration` node.
    - Ensure the Google Sheet named `Photos` exists with required headers: `File name`, `Status`, `Start Time`, `End Time`, `Input File`, `Output File`.
    - Verify the Google Drive folders exist and are accessible by the credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini AI's image editing capabilities to transform product photos by removing backgrounds, adjusting lighting, and enhancing color, preserving the product’s integrity and text readability.                                                                                                                                                                         | See [Google Gemini Image Generation Docs](https://ai.google.dev/gemini-api/docs/image-generation)                                |
| The Gemini node `Edit Image` replaces a more complex HTTP request setup, simplifying the workflow for image editing while still supporting fine-grained configuration if needed.                                                                                                                                                                                                                | Technical note in Sticky Note1                                                                                                   |
| The Google Sheet for logging must have a sheet named `Photos` with headers: File name, Status, Start Time, End Time, Input File, Output File. The Google account used must have Editor access to this sheet.                                                                                                                                                                                     | Sticky Note7 contains detailed Google Sheets configuration instructions                                                          |
| Credential setup is critical: all Google nodes require Google OAuth2 credentials; the Gemini AI node requires Google Gemini (PaLM) API credentials from Google AI Studio.                                                                                                                                                                                                                      | See sticky notes 5 and the official n8n docs on [Google Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/) and [Google Gemini Credentials](https://docs.n8n.io/integrations/builtin/credentials/googleai/) |
| Folder and Sheet IDs can be found in URLs of the Google Drive folders and Google Sheets documents respectively; update the workflow configuration accordingly.                                                                                                                                                                                                                               | Sticky Note9 explains how to find IDs                                                                                             |
| This workflow architecture is extensible to other batch image processing use cases, such as unifying team photos, and can be adapted by modifying the prompt or input sources.                                                                                                                                                                                                                 | See Sticky Note16 (flowers example) and Sticky Note18 (general use cases)                                                        |
| For support or customization requests, the author can be contacted on LinkedIn or via the n8n community forum.                                                                                                                                                                                                                                                                               | [LinkedIn Profile](https://www.linkedin.com/in/ytkaczyk/) and [n8n Forum](https://community.n8n.io/)                              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.