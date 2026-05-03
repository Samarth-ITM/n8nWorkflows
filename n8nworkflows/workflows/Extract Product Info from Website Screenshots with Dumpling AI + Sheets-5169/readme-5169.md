Extract Product Info from Website Screenshots with Dumpling AI + Sheets

https://n8nworkflows.xyz/workflows/extract-product-info-from-website-screenshots-with-dumpling-ai---sheets-5169


# Extract Product Info from Website Screenshots with Dumpling AI + Sheets

### 1. Workflow Overview

This workflow automates the extraction of product information from website screenshots using Dumpling AI's screenshot and image extraction APIs, integrating with Google Sheets for input and output management.

**Target Use Case:**  
Efficiently catalog product details such as name, price, discount, and star rating by processing URLs of product pages added to a Google Sheet, capturing screenshots, extracting visible information from these images, and appending or updating the results back into the spreadsheet.

**Logical Blocks:**

- **1.1 Input Reception:**  
  Detect new URLs added to a Google Sheet to trigger the workflow.

- **1.2 Screenshot Capture and Retrieval:**  
  Request Dumpling AI to capture a screenshot of the webpage, then download the image.

- **1.3 Image Processing and Text Extraction:**  
  Convert the downloaded image to base64 and send it to Dumpling AI to extract product information textually.

- **1.4 Data Formatting and Output:**  
  Format the extracted data and append or update it in the original Google Sheet alongside the URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Monitors a Google Sheet for newly added rows containing product page URLs, triggering downstream processing.

**Nodes Involved:**  
- Watch for New Screenshot URLs in Google Sheets

**Node Details:**

- **Watch for New Screenshot URLs in Google Sheets**  
  - Type: Google Sheets Trigger  
  - Role: Initiates workflow on new row addition  
  - Configuration:  
    - Event: `rowAdded`  
    - Polling frequency: every minute  
    - Target Sheet: Sheet1 in the specified Google Sheets document  
    - Document ID and Sheet Name are linked to a specific Google Sheet URL  
  - Credentials: Google Sheets OAuth2  
  - Input: N/A (trigger node)  
  - Output: Emits new row data, including a field `Site` with the product page URL  
  - Edge Cases:  
    - Authentication or permission issues with Google Sheets  
    - Network or API rate limits  
    - Empty or malformed rows  
  - Version: 1

#### 2.2 Screenshot Capture and Retrieval

**Overview:**  
Sends the product URL to Dumpling AI to generate a screenshot, then downloads the screenshot image for processing.

**Nodes Involved:**  
- Create Screenshot File via Dumpling AI  
- Download Screenshot Image File from Dumpling AI

**Node Details:**

- **Create Screenshot File via Dumpling AI**  
  - Type: HTTP Request  
  - Role: Requests a screenshot via Dumpling AI API  
  - Configuration:  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/screenshot`  
    - Body (JSON):  
      - `url`: dynamically set to the `Site` URL from the trigger node  
      - `blockCookieBanners`: true (to avoid cookie popups)  
      - `autoScroll`: true (to capture full page)  
    - Authentication: HTTP Header (Dumpling AI API key)  
  - Input: URL from Google Sheets trigger  
  - Output: JSON with `screenshotUrl` field pointing to the image URL  
  - Edge Cases:  
    - API limits or authentication failure  
    - Invalid or unreachable URL  
    - Delays in screenshot generation  
  - Version: 4.2

- **Download Screenshot Image File from Dumpling AI**  
  - Type: HTTP Request  
  - Role: Retrieves the screenshot image binary data  
  - Configuration:  
    - Method: GET (default)  
    - URL: dynamically set to the `screenshotUrl` from the previous node's output  
  - Input: Screenshot URL from Dumpling AI  
  - Output: Binary image data for further processing  
  - Edge Cases:  
    - URL expiration or invalid URL  
    - Network timeouts or failures  
  - Version: 4.2

#### 2.3 Image Processing and Text Extraction

**Overview:**  
Converts the downloaded image file into base64 format and sends it to Dumpling AI for extracting visible product information.

**Nodes Involved:**  
- Convert Image File to Base64  
- Extract Text from Screenshot with Dumpling AI

**Node Details:**

- **Convert Image File to Base64**  
  - Type: Extract Binary Data from File  
  - Role: Transforms the binary image into a base64 string within JSON  
  - Configuration:  
    - Operation: `binaryToProperty` (binary data converted to the `data` property)  
  - Input: Binary image from download node  
  - Output: JSON with base64-encoded image string under `data`  
  - Edge Cases:  
    - Corrupt or unsupported image format  
  - Version: 1

- **Extract Text from Screenshot with Dumpling AI**  
  - Type: HTTP Request  
  - Role: Sends base64 image and prompt to Dumpling AI's image extraction API  
  - Configuration:  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/extract-image`  
    - Body (JSON):  
      - `inputMethod`: "base64"  
      - `images`: array with base64 image string  
      - `prompt`: instructs to extract clearly visible product name, price, discount, and star rating without guessing  
      - `jsonMode`: true (expects structured JSON response)  
    - Authentication: HTTP Header (Dumpling AI API key)  
  - Input: Base64 image from previous node  
  - Output: JSON with `results` containing extracted product info  
  - Edge Cases:  
    - API authentication or rate limiting  
    - Image too complex or unclear for accurate extraction  
    - Malformed or unexpected response format  
  - Version: 4.2

#### 2.4 Data Formatting and Output

**Overview:**  
Formats the extracted product data for insertion into Google Sheets and appends or updates the sheet accordingly.

**Nodes Involved:**  
- Format Extracted Data for Google Sheets  
- Save extracted data to Google Sheets

**Node Details:**

- **Format Extracted Data for Google Sheets**  
  - Type: Set  
  - Role: Maps extracted `results` field to a JSON property named `results` for downstream use  
  - Configuration:  
    - Sets a single string property `results` with the value of `$json.results`  
  - Input: Extracted text JSON from Dumpling AI  
  - Output: JSON with `results` string for Google Sheets insertion  
  - Edge Cases:  
    - Missing or empty `results` field  
  - Version: 3.4

- **Save extracted data to Google Sheets**  
  - Type: Google Sheets  
  - Role: Inserts or updates a row in the Google Sheet with the original site URL and extracted product information  
  - Configuration:  
    - Operation: `appendOrUpdate`  
    - Sheet: Sheet1 (gid=0)  
    - Document: same as trigger node  
    - Columns:  
      - `Site`: set to original URL from trigger node  
      - `product`: set to extracted `results` string  
    - Matching on `Site` column to update existing entries or append new  
  - Credentials: Google Sheets OAuth2  
  - Input: JSON with formatted data  
  - Edge Cases:  
    - Permission or quota issues with Sheets  
    - Data formatting errors causing write failure  
  - Version: 4.6

---

### 3. Summary Table

| Node Name                               | Node Type                    | Functional Role                               | Input Node(s)                            | Output Node(s)                     | Sticky Note                                                                                      |
|----------------------------------------|------------------------------|-----------------------------------------------|-----------------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| Watch for New Screenshot URLs in Google Sheets | Google Sheets Trigger          | Trigger on new URL rows in Google Sheets      | N/A                                     | Create Screenshot File via Dumpling AI | ### 🖼️ Automated Product Data Extraction from Website Screenshots\n\nThis workflow starts when a new row is added in Google Sheets containing a product page URL. It triggers Dumpling AI to capture a screenshot of the site. The image is downloaded using an HTTP Request, then converted from binary to base64. Dumpling AI is called again to extract product information (like product name, price, discount, and ratings) directly from the screenshot. The Edit Fields node maps all the extracted details, and the final Google Sheets node updates your spreadsheet with the structured product data next to the original URL.\n\nUse this workflow to save hours of manual copy-paste work, speed up cataloging, and ensure your product database is always fresh and accurate.\n\n" |
| Create Screenshot File via Dumpling AI         | HTTP Request                 | Request screenshot from Dumpling AI            | Watch for New Screenshot URLs in Google Sheets | Download Screenshot Image File from Dumpling AI |                                                                                                 |
| Download Screenshot Image File from Dumpling AI | HTTP Request                 | Download screenshot image binary                | Create Screenshot File via Dumpling AI  | Convert Image File to Base64       |                                                                                                 |
| Convert Image File to Base64                      | Extract Binary Data from File | Convert image binary to base64 string           | Download Screenshot Image File from Dumpling AI | Extract Text from Screenshot with Dumpling AI |                                                                                                 |
| Extract Text from Screenshot with Dumpling AI    | HTTP Request                 | Extract product info text from base64 image    | Convert Image File to Base64             | Format Extracted Data for Google Sheets |                                                                                                 |
| Format Extracted Data for Google Sheets           | Set                         | Format extracted data for Google Sheets         | Extract Text from Screenshot with Dumpling AI | Save extracted data to Google Sheets |                                                                                                 |
| Save extracted data to Google Sheets               | Google Sheets               | Append or update product info in Google Sheets | Format Extracted Data for Google Sheets | N/A                              |                                                                                                 |
| Sticky Note                                       | Sticky Note                 | Documentation and overview                       | N/A                                     | N/A                              | ### 🖼️ Automated Product Data Extraction from Website Screenshots\n\nThis workflow starts when a new row is added in Google Sheets containing a product page URL. It triggers Dumpling AI to capture a screenshot of the site. The image is downloaded using an HTTP Request, then converted from binary to base64. Dumpling AI is called again to extract product information (like product name, price, discount, and ratings) directly from the screenshot. The Edit Fields node maps all the extracted details, and the final Google Sheets node updates your spreadsheet with the structured product data next to the original URL.\n\nUse this workflow to save hours of manual copy-paste work, speed up cataloging, and ensure your product database is always fresh and accurate.\n\n" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Credentials: Google Sheets OAuth2 (with access to your target spreadsheet)  
   - Set event to "rowAdded"  
   - Set polling frequency to "Every Minute"  
   - Specify the Document ID and Sheet Name (e.g., "URL-n8n" sheet, "Sheet1")  
   - This node listens for new URLs added in the sheet.

2. **Create HTTP Request Node (Create Screenshot File via Dumpling AI)**  
   - Type: HTTP Request  
   - Credentials: HTTP Header Auth with Dumpling AI API key  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/screenshot`  
   - Set Body Format to JSON  
   - Request body:  
     ```json
     {
       "url": "{{ $json.Site }}",
       "blockCookieBanners": true,
       "autoScroll": true
     }
     ```  
   - Connect from Google Sheets Trigger node.

3. **Create HTTP Request Node (Download Screenshot Image File)**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{ $json.screenshotUrl }}` (map screenshotUrl from previous node)  
   - Connect from the previous node.

4. **Create Extract Binary Data from File Node (Convert Image File to Base64)**  
   - Type: Extract Binary Data from File  
   - Operation: binaryToProperty  
   - Connect from Download Screenshot Image File node.

5. **Create HTTP Request Node (Extract Text from Screenshot with Dumpling AI)**  
   - Type: HTTP Request  
   - Credentials: HTTP Header Auth with Dumpling AI API key  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/extract-image`  
   - Body Format: JSON  
   - Request body:  
     ```json
     {
       "inputMethod": "base64",
       "images": ["{{$json.data}}"],
       "prompt": "Look at the image and extract the following details if they are visible: the product name, the price, any discount or sale information, and the star rating. Only include information that is clearly shown in the image. Do not guess or assume anything that is not visible.",
       "jsonMode": true
     }
     ```  
   - Connect from Convert Image File to Base64 node.

6. **Create Set Node (Format Extracted Data for Google Sheets)**  
   - Type: Set  
   - Add field `results` of type string  
   - Set value to `={{ $json.results }}` (map extracted results)  
   - Connect from Extract Text from Screenshot with Dumpling AI node.

7. **Create Google Sheets Node (Save extracted data to Google Sheets)**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2  
   - Document ID and Sheet Name same as trigger node (the sheet where URLs are stored)  
   - Operation: appendOrUpdate  
   - Define columns with mapping:  
     - `Site` = URL from Google Sheets Trigger node (`{{ $('Watch for New Screenshot URLs in Google Sheets').item.json.Site }}`)  
     - `product` = extracted data from Set node (`{{ $json.results }}`)  
   - Matching column: `Site` (to update existing rows)  
   - Connect from Format Extracted Data for Google Sheets node.

**Connection Order:**  
- Google Sheets Trigger → Create Screenshot → Download Screenshot → Convert to Base64 → Extract Text → Format Data → Google Sheets Append/Update

**Credential Setup:**  
- Google Sheets: OAuth2 with access to target spreadsheet  
- Dumpling AI: HTTP Header Auth with valid API key

**Default Values / Constraints:**  
- Poll every minute to detect new URLs  
- Prompt in image extraction node is strict to avoid guessing  
- Append or update based on `Site` to prevent duplicates

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| 🖼️ Automated Product Data Extraction from Website Screenshots: This workflow enables automatic capture of product page screenshots and extraction of key product details directly from images, boosting productivity and data accuracy. | Workflow sticky note content included in the first node and sticky note node.                                            |
| Dumpling AI API Documentation: https://app.dumplingai.com/docs/api                             | Reference for Dumpling AI screenshot and image extraction endpoints.                                                    |
| Google Sheets API and OAuth2 Setup Documentation: https://developers.google.com/sheets/api     | Required for setting up Google Sheets Trigger and Google Sheets nodes with proper credentials.                           |

---

**Disclaimer:**  
The provided content is generated from an n8n workflow automation tool. All data processed is legal and publicly accessible. The workflow complies with all content policies and contains no illegal or protected material.