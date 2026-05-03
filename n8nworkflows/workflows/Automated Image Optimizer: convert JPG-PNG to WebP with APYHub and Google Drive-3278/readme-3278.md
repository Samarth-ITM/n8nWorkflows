Automated Image Optimizer: convert JPG/PNG to WebP with APYHub and Google Drive

https://n8nworkflows.xyz/workflows/automated-image-optimizer--convert-jpg-png-to-webp-with-apyhub-and-google-drive-3278


# Automated Image Optimizer: convert JPG/PNG to WebP with APYHub and Google Drive

### 1. Workflow Overview

This workflow automates the conversion of images from JPG or PNG formats to the WebP format using the APYHub API. It is designed to streamline image optimization by retrieving image URLs from a Google Sheet, converting them via API calls, updating the sheet with conversion results, and uploading the converted images to Google Drive for easy access and management.

**Target Use Cases:**  
- Web developers and SEO specialists aiming to optimize website images for faster loading and better search rankings.  
- Content managers automating bulk image format conversions.  
- Teams integrating cloud storage and API-based image processing in their workflows.

**Logical Blocks:**

- **1.1 Input Reception and Initialization:**  
  Starts the workflow manually and sets the API key for APYHub.

- **1.2 Image Retrieval and Metadata Extraction:**  
  Fetches image URLs from Google Sheets and extracts file extensions.

- **1.3 Image Type Determination and Routing:**  
  Routes images based on their extension (JPG/JPEG or PNG) to the appropriate conversion node.

- **1.4 Image Conversion via APYHub API:**  
  Converts images to WebP format using HTTP POST requests to APYHub.

- **1.5 Google Sheet Update:**  
  Updates the Google Sheet with the converted image URL and marks the conversion as done.

- **1.6 Download Converted Image:**  
  Downloads the converted WebP image file from the URL returned by the API.

- **1.7 Upload to Google Drive:**  
  Uploads the downloaded WebP image to a specified Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  This block initiates the workflow manually and sets the API key required for APYHub API authentication.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set API KEY (Set Node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: No parameters; triggers workflow execution.  
    - Inputs: None  
    - Outputs: Connects to Set API KEY node.  
    - Edge Cases: None typical; user must manually trigger.

  - **Set API KEY**  
    - Type: Set  
    - Role: Stores the APYHub API key as a workflow variable.  
    - Configuration: Assigns a string variable `apikey` with the API key value.  
    - Inputs: From Manual Trigger  
    - Outputs: Connects to Get images node.  
    - Edge Cases: API key must be valid; invalid or expired keys will cause API request failures downstream.

---

#### 1.2 Image Retrieval and Metadata Extraction

- **Overview:**  
  Retrieves image URLs from a Google Sheet and extracts the filename and extension from each URL for further processing.

- **Nodes Involved:**  
  - Get images (Google Sheets)  
  - Get extension (Code)

- **Node Details:**

  - **Get images**  
    - Type: Google Sheets  
    - Role: Reads rows from a Google Sheet containing image URLs and status flags.  
    - Configuration:  
      - Document ID and Sheet Name specified (e.g., `1upj3EDLwU1N7NHWWV3DhwMuE6ty39tIK5z5lCVDWWuM`, sheet `gid=0`).  
      - Filters rows where `DONE` column is empty (i.e., images not yet converted).  
      - Credentials: Google Sheets OAuth2.  
    - Inputs: From Set API KEY  
    - Outputs: Connects to Get extension node.  
    - Edge Cases:  
      - Empty or missing rows cause no processing.  
      - Google Sheets API quota or auth errors possible.

  - **Get extension**  
    - Type: Code (JavaScript)  
    - Role: Parses each image URL to extract the filename and file extension (jpg, jpeg, png).  
    - Configuration:  
      - Loops over all input items.  
      - Extracts filename and extension from URL by splitting on `/`, `#`, `?`, and `.`.  
      - Adds `FILENAME` and `EXTENSION` fields to each item’s JSON.  
    - Inputs: From Get images  
    - Outputs: Connects to JPG or PNG? node.  
    - Edge Cases:  
      - URLs without extensions or malformed URLs may cause incorrect parsing.  
      - Case sensitivity not handled here; handled later.

---

#### 1.3 Image Type Determination and Routing

- **Overview:**  
  Determines whether each image is JPG/JPEG or PNG and routes the workflow accordingly for conversion.

- **Nodes Involved:**  
  - JPG or PNG? (Switch)

- **Node Details:**

  - **JPG or PNG?**  
    - Type: Switch  
    - Role: Routes items based on the `EXTENSION` field.  
    - Configuration:  
      - Three conditions:  
        - `EXTENSION` equals "jpg" (case sensitive) → output "jpeg"  
        - `EXTENSION` equals "jpeg" → output "jpeg"  
        - `EXTENSION` equals "png" → output "png"  
      - Routes JPG/JPEG to "From JPG to WEBP" node.  
      - Routes PNG to "PNG to WEBP" node.  
    - Inputs: From Get extension  
    - Outputs:  
      - "jpeg" output connects to From JPG to WEBP node.  
      - "png" output connects to PNG to WEBP node.  
    - Edge Cases:  
      - Extensions other than jpg, jpeg, png are ignored (no route).  
      - Case sensitivity may cause missed matches if extension case varies.

---

#### 1.4 Image Conversion via APYHub API

- **Overview:**  
  Converts the images to WebP format by sending POST requests to the APYHub API endpoints for JPG or PNG images.

- **Nodes Involved:**  
  - From JPG to WEBP (HTTP Request)  
  - PNG to WEBP (HTTP Request)

- **Node Details:**

  - **From JPG to WEBP**  
    - Type: HTTP Request  
    - Role: Converts JPG/JPEG images to WebP via APYHub API.  
    - Configuration:  
      - POST to `https://api.apyhub.com/convert/image/jpeg/webp/url?output=test-sample`  
      - JSON body: `{ "url": "<original image URL>" }`  
      - Headers:  
        - `Content-Type: application/json`  
        - `api-token`: taken from `Set API KEY` node variable `apikey`  
      - Sends JSON body and headers.  
    - Inputs: From JPG or PNG? switch (jpeg output)  
    - Outputs: Connects to Update Sheet node.  
    - Edge Cases:  
      - API token invalid or missing → 401 Unauthorized.  
      - API rate limits or downtime.  
      - Invalid image URLs cause conversion failure.

  - **PNG to WEBP**  
    - Type: HTTP Request  
    - Role: Converts PNG images to WebP via APYHub API.  
    - Configuration:  
      - POST to `https://api.apyhub.com/convert/image/png/webp/url?output=test-sample`  
      - JSON body: `{ "url": "<original image URL>" }`  
      - Headers:  
        - `Content-Type: application/json`  
        - **Note:** Header name is `apy-token` (likely a typo; should be `api-token` for consistency).  
      - Sends JSON body and headers.  
    - Inputs: From JPG or PNG? switch (png output)  
    - Outputs: Connects to Update Sheet node.  
    - Edge Cases:  
      - Header name inconsistency may cause auth failure.  
      - Same API-related errors as JPG conversion.

---

#### 1.5 Google Sheet Update

- **Overview:**  
  Updates the Google Sheet row with the URL of the converted WebP image and marks the conversion as done.

- **Nodes Involved:**  
  - Update Sheet (Google Sheets)

- **Node Details:**

  - **Update Sheet**  
    - Type: Google Sheets  
    - Role: Updates the `TO` column with the converted image URL and sets `DONE` to "x".  
    - Configuration:  
      - Uses the `row_number` from the original `Get images` node to update the correct row.  
      - Columns updated:  
        - `TO` = API response field `data` (converted image URL)  
        - `DONE` = "x" (marking completion)  
      - Credentials: Google Sheets OAuth2.  
      - Document ID and Sheet Name same as Get images.  
    - Inputs: From conversion HTTP Request nodes.  
    - Outputs: Connects to Get file image node.  
    - Edge Cases:  
      - Google Sheets API errors or quota limits.  
      - Row number mismatch or missing data could cause update failure.

---

#### 1.6 Download Converted Image

- **Overview:**  
  Downloads the converted WebP image file from the URL provided by the APYHub API.

- **Nodes Involved:**  
  - Get file image (HTTP Request)

- **Node Details:**

  - **Get file image**  
    - Type: HTTP Request  
    - Role: Downloads the converted WebP image as a file.  
    - Configuration:  
      - URL taken from the `TO` field (converted image URL) in the JSON.  
      - Response format set to "file" to download the binary data.  
    - Inputs: From Update Sheet node.  
    - Outputs: Connects to Upload image node.  
    - Edge Cases:  
      - Broken or expired URLs cause download failure.  
      - Network timeouts or slow responses.

---

#### 1.7 Upload to Google Drive

- **Overview:**  
  Uploads the downloaded WebP image to a specified folder in Google Drive.

- **Nodes Involved:**  
  - Upload image (Google Drive)

- **Node Details:**

  - **Upload image**  
    - Type: Google Drive  
    - Role: Uploads the WebP image file to Google Drive.  
    - Configuration:  
      - File name set dynamically as `<FILENAME>.webp` from the `Get extension` node.  
      - Drive ID: "My Drive" (default Google Drive root)  
      - Folder ID specified (e.g., `1XyUSYXdNrZIw0XyZ3YpuaxGJjOaARyEJ`)  
      - Credentials: Google Drive OAuth2.  
    - Inputs: From Get file image node.  
    - Outputs: None (end of workflow).  
    - Edge Cases:  
      - Google Drive API quota or permission errors.  
      - Folder ID invalid or missing permissions.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                              | Input Node(s)                 | Output Node(s)             | Sticky Note                                                                                                  |
|-------------------------|--------------------|----------------------------------------------|------------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Starts workflow manually                      | None                         | Set API KEY                |                                                                                                              |
| Set API KEY             | Set                | Stores APYHub API key                         | When clicking ‘Test workflow’ | Get images                 |                                                                                                              |
| Get images              | Google Sheets      | Retrieves image URLs from Google Sheet        | Set API KEY                  | Get extension              | ## PRELIMINARY STEP - Get your FREE API KEY from [APYHub](https://apyhub.com//) - Clone [this sheet](https://docs.google.com/spreadsheets/d/1upj3EDLwU1N7NHWWV3DhwMuE6ty39tIK5z5lCVDWWuM/edit?usp=sharing) and insert the URL of your images (only jpg, jpeg and png format) in the column "FROM" |
| Get extension           | Code               | Extracts filename and extension from URL      | Get images                   | JPG or PNG?                |                                                                                                              |
| JPG or PNG?             | Switch             | Routes images based on extension               | Get extension                | From JPG to WEBP, PNG to WEBP |                                                                                                              |
| From JPG to WEBP        | HTTP Request       | Converts JPG/JPEG images to WebP via APYHub   | JPG or PNG? (jpeg output)    | Update Sheet               |                                                                                                              |
| PNG to WEBP             | HTTP Request       | Converts PNG images to WebP via APYHub         | JPG or PNG? (png output)     | Update Sheet               |                                                                                                              |
| Update Sheet            | Google Sheets      | Updates Google Sheet with converted URL and status | From JPG to WEBP, PNG to WEBP | Get file image             |                                                                                                              |
| Get file image          | HTTP Request       | Downloads converted WebP image file            | Update Sheet                 | Upload image               |                                                                                                              |
| Upload image            | Google Drive       | Uploads WebP image to Google Drive folder      | Get file image               | None                      |                                                                                                              |
| Sticky Note             | Sticky Note        | Workflow description and purpose               | None                         | None                      | ## Convert image from jpg/png to webp This workflow automates the process of converting images from **JPG/PNG** format to **WEBP** using the **APYHub API**. It retrieves image URLs from a **Google Sheet**, converts the images, and uploads the converted files to **Google Drive**. This workflow is a powerful tool for automating image conversion tasks, saving time and ensuring that images are efficiently converted and stored in the desired format. |
| Sticky Note1            | Sticky Note        | Setup instructions and preliminary notes       | None                         | None                      | ## PRELIMINARY STEP - Get your FREE API KEY from [APYHub](https://apyhub.com//) - Clone [this sheet](https://docs.google.com/spreadsheets/d/1upj3EDLwU1N7NHWWV3DhwMuE6ty39tIK5z5lCVDWWuM/edit?usp=sharing) and insert the URL of your images (only jpg, jpeg and png format) in the column "FROM" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for API Key:**  
   - Name: `Set API KEY`  
   - Type: Set  
   - Add a string field named `apikey`.  
   - Set the value to your APYHub API key (e.g., `"APY*****************************"`).  
   - Connect output of Manual Trigger to this node.

3. **Create Google Sheets Node to Get Images:**  
   - Name: `Get images`  
   - Type: Google Sheets  
   - Operation: Read rows (default).  
   - Document ID: Your Google Sheet ID (e.g., `1upj3EDLwU1N7NHWWV3DhwMuE6ty39tIK5z5lCVDWWuM`).  
   - Sheet Name: Specify the sheet (e.g., `gid=0`).  
   - Filter: Only rows where `DONE` column is empty (filter on `DONE` column).  
   - Credentials: Use Google Sheets OAuth2 credentials.  
   - Connect output of `Set API KEY` node to this node.

4. **Create Code Node to Extract Filename and Extension:**  
   - Name: `Get extension`  
   - Type: Code (JavaScript)  
   - Paste the following code:

     ```javascript
     for (const item of $input.all()) {
       const url = item.json.FROM;
       const filenameWithExtension = url.split('/').pop().split(/[#?]/)[0];
       const extension = filenameWithExtension.split('.').pop();
       const filename = filenameWithExtension.substring(0, filenameWithExtension.length - extension.length - 1);
       item.json.FILENAME = filename;
       item.json.EXTENSION = extension;
     }
     return $input.all();
     ```

   - Connect output of `Get images` node to this node.

5. **Create Switch Node to Route by Extension:**  
   - Name: `JPG or PNG?`  
   - Type: Switch  
   - Add three rules:  
     - Condition 1: `EXTENSION` equals `jpg` (case sensitive), output key: `jpeg`  
     - Condition 2: `EXTENSION` equals `jpeg`, output key: `jpeg`  
     - Condition 3: `EXTENSION` equals `png`, output key: `png`  
   - Connect output of `Get extension` node to this node.

6. **Create HTTP Request Node for JPG to WEBP Conversion:**  
   - Name: `From JPG to WEBP`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apyhub.com/convert/image/jpeg/webp/url?output=test-sample`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `api-token`: Expression referencing `Set API KEY` node’s `apikey` field (`={{ $('Set API KEY').item.json.apikey }}`)  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "url": "{{ $json.FROM }}"
     }
     ```  
   - Connect the `jpeg` output of `JPG or PNG?` node to this node.

7. **Create HTTP Request Node for PNG to WEBP Conversion:**  
   - Name: `PNG to WEBP`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apyhub.com/convert/image/png/webp/url?output=test-sample`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `api-token`: Expression referencing `Set API KEY` node’s `apikey` field (`={{ $('Set API KEY').item.json.apikey }}`)  
     *(Note: Fix header name from `apy-token` to `api-token` to avoid auth issues.)*  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "url": "{{ $json.FROM }}"
     }
     ```  
   - Connect the `png` output of `JPG or PNG?` node to this node.

8. **Create Google Sheets Node to Update Sheet:**  
   - Name: `Update Sheet`  
   - Type: Google Sheets  
   - Operation: Update row  
   - Document ID and Sheet Name: Same as `Get images` node.  
   - Columns to update:  
     - `TO`: Set to the API response field containing the converted image URL (`={{ $json.data }}`)  
     - `DONE`: Set to `"x"`  
     - `row_number`: Use the row number from the original `Get images` node (`={{ $('Get images').item.json.row_number }}`)  
   - Credentials: Google Sheets OAuth2  
   - Connect outputs of both `From JPG to WEBP` and `PNG to WEBP` nodes to this node.

9. **Create HTTP Request Node to Download Converted Image:**  
   - Name: `Get file image`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Expression referencing the `TO` field (`={{ $json.TO }}`)  
   - Response Format: File (binary)  
   - Connect output of `Update Sheet` node to this node.

10. **Create Google Drive Node to Upload Image:**  
    - Name: `Upload image`  
    - Type: Google Drive  
    - Operation: Upload file  
    - File Name: Expression using filename from `Get extension` node plus `.webp` extension (`={{ $('Get extension').item.json.FILENAME }}.webp`)  
    - Drive ID: `My Drive` (default)  
    - Folder ID: Specify your target Google Drive folder ID (e.g., `1XyUSYXdNrZIw0XyZ3YpuaxGJjOaARyEJ`)  
    - Credentials: Google Drive OAuth2  
    - Connect output of `Get file image` node to this node.

11. **Verify and Test:**  
    - Save the workflow.  
    - Click “Test workflow” to run the process.  
    - Monitor for errors and verify Google Sheet updates and Google Drive uploads.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Using WebP images improves SEO by faster loading, better Core Web Vitals, improved mobile performance, higher rankings, and reduced server load. | SEO benefits of WebP images.                                                                              |
| Get your FREE API KEY from APYHub to enable image conversion: https://apyhub.com/                                                         | APYHub API key registration.                                                                              |
| Clone the sample Google Sheet to use with this workflow: https://docs.google.com/spreadsheets/d/1upj3EDLwU1N7NHWWV3DhwMuE6ty39tIK5z5lCVDWWuM/edit?usp=sharing | Sample Google Sheet template with columns `FROM`, `TO`, and `DONE`.                                       |
| Ensure Google Sheets and Google Drive OAuth2 credentials are properly configured in n8n for seamless integration.                        | Credential setup instructions in n8n documentation.                                                      |
| Note: The header name in the PNG to WEBP HTTP Request node should be corrected from `apy-token` to `api-token` to avoid authentication errors. | Important fix to avoid API authentication failure.                                                       |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Convert image from jpg/png to webp" workflow in n8n, including all nodes, configurations, and integration points.