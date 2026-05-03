Automated Product Ad Image Creation with OpenAI, Gemini & Google Workspace

https://n8nworkflows.xyz/workflows/automated-product-ad-image-creation-with-openai--gemini---google-workspace-8627


# Automated Product Ad Image Creation with OpenAI, Gemini & Google Workspace

---

### 1. Workflow Overview

This n8n workflow automates the creation of promotional product advertisement images by leveraging AI models (OpenAI GPT-4O-MINI and Google Gemini) alongside Google Workspace services. It is designed to produce ad-ready images combining product and influencer/model photos, enriched with AI-generated stylistic guidance.

The workflow logically divides into three main blocks:

- **1.1 Trigger & Data Preparation**: Automatically triggers on a schedule, fetches product and model image URLs from a Google Sheet for the current date, downloads these images from Google Drive, and converts them into a format suitable for AI processing.

- **1.2 AI Analysis & Image Generation**: Uses OpenAI to analyze the product image to generate a detailed promotional description. Then, it calls the Google Gemini model via HTTP request to generate a new combined ad image utilizing both product and influencer images, followed by cleanup and conversion of the AI output.

- **1.3 Save & Update**: Uploads the AI-generated ad image back to a specified Google Drive folder and updates the Google Sheet with a link to the new image and a status indicating readiness to publish.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Preparation

- **Overview**: This block initiates the workflow on a schedule, retrieves the relevant product and model image URLs for the current date from a Google Sheet, downloads these images from Google Drive, and converts the binary files into text/base64 suitable for AI input.

- **Nodes Involved**:  
  - Schedule Trigger  
  - Get the Raw (Google Sheets)  
  - Download Product Image (Google Drive)  
  - Convert Binary to Text (Product Image)  
  - Download Influencer Image (Google Drive)  
  - influencer_image_Convert_to_Text (Influencer Image Conversion)  

- **Node Details**:

  - **Schedule Trigger**  
    - *Type*: Schedule Trigger  
    - *Role*: Initiates the workflow automatically on a recurring interval (default daily).  
    - *Configuration*: Uses default interval with no complex filter, triggering once per day.  
    - *Connections*: Output → Get the Raw node.  
    - *Failure cases*: Misconfiguration of schedule; no direct data errors.  

  - **Get the Raw (Google Sheets)**  
    - *Type*: Google Sheets (Get Row)  
    - *Role*: Retrieves the product and influencer image URLs corresponding to the current date.  
    - *Configuration*:  
      - Filters based on a concatenated date string from the item fields (`Month Day, Year`).  
      - Retrieves from a defined spreadsheet and sheet (`Sheet1`).  
    - *Expressions*: Uses date fields dynamically to filter rows.  
    - *Connections*: Output → Download Product Image.  
    - *Failures*: Authentication errors, missing or malformed date fields, no matching rows found.  

  - **Download Product Image (Google Drive)**  
    - *Type*: Google Drive - Download  
    - *Role*: Downloads the product image file from Google Drive using the URL from the sheet.  
    - *Configuration*: Uses the file ID extracted via expression from the `Product url` field.  
    - *Connections*: Output → Convert Binary to Text and Analyze image nodes.  
    - *Failures*: File not found, permission errors, invalid URL or file ID.  

  - **Convert Binary to Text (Product Image Conversion)**  
    - *Type*: Extract From File (binaryToPropery)  
    - *Role*: Converts the downloaded product image binary data into base64 text for AI input.  
    - *Connections*: Output → Download Influencer Image node.  
    - *Failures*: Corrupted file data, conversion failures.  

  - **Download Influencer Image (Google Drive)**  
    - *Type*: Google Drive - Download  
    - *Role*: Downloads the influencer/model image file from Google Drive using the URL from the sheet.  
    - *Configuration*: Uses file ID derived from the `Model url` field.  
    - *Connections*: Output → influencer_image_Convert_to_Text.  
    - *Failures*: Similar to Download Product Image node.  

  - **influencer_image_Convert_to_Text**  
    - *Type*: Extract From File (binaryToPropery)  
    - *Role*: Converts the downloaded influencer image binary data into base64 text for AI input.  
    - *Connections*: Output → Image Generation node.  
    - *Failures*: Conversion errors, corrupted file.  

---

#### 2.2 AI Analysis & Image Generation

- **Overview**: This block analyzes the product image to generate a detailed advertising description via OpenAI, then calls the Google Gemini model to generate a new ad image combining product and influencer images. The output is cleaned and converted into a usable image file.

- **Nodes Involved**:  
  - Analyze image (OpenAI)  
  - Image Generation (HTTP Request to OpenRouter Gemini)  
  - base64 cleanup (Code Node)  
  - Convert to File  

- **Node Details**:

  - **Analyze image (OpenAI)**  
    - *Type*: OpenAI (LangChain Node)  
    - *Role*: Analyzes the product image (provided as base64) to generate a detailed ad-ready visual description focusing on product presentation, styling, lighting, background, and mood, abstracted from specific model details.  
    - *Configuration*:  
      - Model: GPT-4O-MINI  
      - Operation: Image analysis with base64 input  
      - Prompt: Explicit instructions to generate ad-focused visual description.  
    - *Connections*: Output → Image Generation.  
    - *Failures*: API authentication, request timeout, improper base64 input, model limits.  

  - **Image Generation (HTTP Request)**  
    - *Type*: HTTP Request  
    - *Role*: Calls the OpenRouter API using Google Gemini model to generate a new ad image combining product and influencer images and the AI-generated prompt.  
    - *Configuration*:  
      - Endpoint: `https://openrouter.ai/api/v1/chat/completions`  
      - Method: POST  
      - Body: JSON containing model name, prompt from previous node, and two image inputs (product and influencer) in base64.  
      - Headers: Authorization Bearer token, Content-Type application/json.  
    - *Expressions*: Uses outputs from Analyze image and Convert Binary to Text nodes.  
    - *Connections*: Output → base64 cleanup.  
    - *Failures*: API key issues, rate limits, malformed JSON, network errors.  

  - **base64 cleanup (Code Node)**  
    - *Type*: Code (JavaScript)  
    - *Role*: Cleans the raw base64 image string from the AI response by removing the `data:image/png;base64,` prefix.  
    - *Configuration*: Custom JS code looping through items to ensure clean base64 data for file conversion.  
    - *Connections*: Output → Convert to File.  
    - *Failures*: Unexpected response structure, missing data paths.  

  - **Convert to File**  
    - *Type*: Convert To File  
    - *Role*: Converts cleaned base64 string into a binary file (image) for upload.  
    - *Configuration*: Source property set to the cleaned base64 string path in JSON (`choices[0].images[0].image_url.url`).  
    - *Connections*: Output → upload_image (Google Drive Upload).  
    - *Failures*: Conversion failures, corrupted data.  

---

#### 2.3 Save & Update

- **Overview**: This block uploads the newly generated ad image to Google Drive and updates the Google Sheet with the image's link and publish status.

- **Nodes Involved**:  
  - upload_image (Google Drive Upload)  
  - Update Spreadsheet with Results (Google Sheets AppendOrUpdate)  

- **Node Details**:

  - **upload_image (Google Drive Upload)**  
    - *Type*: Google Drive - Upload  
    - *Role*: Uploads the AI-generated ad image file to a specified Google Drive folder.  
    - *Configuration*:  
      - File name format: `Ad Image {{ $runIndex + 1 }}` (dynamic naming)  
      - Target folder ID specified for output images  
      - Drive set to "My Drive"  
    - *Connections*: Output → Update Spreadsheet with Results.  
    - *Failures*: Permission denied, folder not found, file size limits.  

  - **Update Spreadsheet with Results (Google Sheets AppendOrUpdate)**  
    - *Type*: Google Sheets (Append or Update Row)  
    - *Role*: Updates the Google Sheet row corresponding to the processed date by adding the link to the generated ad image and marking the status as "Publish".  
    - *Configuration*:  
      - Matches rows by `Date` column.  
      - Updates or appends columns: Date, Publish ("Publish"), and "Ad Image Ready To Post" (with uploaded file webContentLink).  
      - Uses same spreadsheet and sheet as initial data fetch.  
    - *Connections*: Terminal node (no outputs).  
    - *Failures*: Authentication, sheet access, row matching errors.  

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                                  | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                                                                        |
|-------------------------------|-----------------------------|-------------------------------------------------|------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger            | Initiates workflow daily                         |                              | Get the Raw                    | Step 1: Trigger & Data Preparation - runs daily scheduling                                                                                                       |
| Get the Raw                   | Google Sheets (Get Row)      | Fetches product and influencer image URLs       | Schedule Trigger             | Download Product Image          | Step 1: Trigger & Data Preparation - fetch product/model URLs for current date                                                                                   |
| Download Product Image         | Google Drive (Download)      | Downloads product image                          | Get the Raw                  | Convert Binary to Text, Analyze image | Step 1: Trigger & Data Preparation - download product image                                                                                                      |
| Convert Binary to Text         | Extract From File            | Converts product image binary to base64          | Download Product Image       | Download Influencer Image       | Step 1: Trigger & Data Preparation - convert product image for AI input                                                                                          |
| Download Influencer Image      | Google Drive (Download)      | Downloads influencer/model image                 | Convert Binary to Text       | influencer_image_Convert_to_Text | Step 1: Trigger & Data Preparation - download influencer image                                                                                                   |
| influencer_image_Convert_to_Text | Extract From File            | Converts influencer image binary to base64       | Download Influencer Image    | Image Generation               | Step 1: Trigger & Data Preparation - convert influencer image for AI input                                                                                       |
| Analyze image                 | OpenAI (LangChain)           | Generates ad-focused detailed product description| Download Product Image       | Image Generation               | Step 2: AI Analysis & Image Generation - generate ad description from product image                                                                              |
| Image Generation              | HTTP Request                 | Calls Gemini model to generate combined ad image | influencer_image_Convert_to_Text, Analyze image | base64 cleanup               | Step 2: AI Analysis & Image Generation - generate ad image combining product and influencer images                                                               |
| base64 cleanup               | Code                        | Cleans base64 prefix from AI response            | Image Generation             | Convert to File                | Step 2: AI Analysis & Image Generation - clean AI image output                                                                                                   |
| Convert to File               | Convert To File              | Converts cleaned base64 string to binary file    | base64 cleanup              | upload_image                  | Step 2: AI Analysis & Image Generation - convert cleaned base64 to file                                                                                          |
| upload_image                 | Google Drive (Upload)        | Uploads generated ad image to Google Drive       | Convert to File             | Update Spreadsheet with Results | Step 3: Save & Update - upload generated ad image                                                                                                               |
| Update Spreadsheet with Results | Google Sheets (Append/Update)| Updates sheet row with image link and publish status | upload_image                |                                | Step 3: Save & Update - update spreadsheet with publishing info                                                                                                |
| Sticky Note                  | Sticky Note                 | Documentation note                               |                              |                                | Step 1: Trigger & Data Preparation summary                                                                                                                      |
| Sticky Note1                 | Sticky Note                 | Documentation note                               |                              |                                | Step 2: AI Analysis & Image Generation summary                                                                                                                  |
| Sticky Note2                 | Sticky Note                 | Documentation note                               |                              |                                | Step 3: Save & Update summary                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily (default interval).  

2. **Add a Google Sheets node ("Get the Raw")**  
   - Operation: Get Row (or similar)  
   - Connect input from Schedule Trigger  
   - Set document ID to your Google Sheet ID with product/model URLs  
   - Set sheet name to "Sheet1" or your sheet name  
   - Apply a filter on the `Date` column using dynamic expression: `={{ $json.Month }} {{ $json['Day of month'] }}, {{ $json.Year }}` to match current date row.  
   - Set authentication credentials for Google Sheets OAuth2.  

3. **Add a Google Drive node ("Download Product Image")**  
   - Operation: Download  
   - Connect input from "Get the Raw"  
   - Set file ID from the `Product url` field (using expression to extract ID from URL)  
   - Provide Google Drive OAuth2 credentials.  

4. **Add an Extract From File node ("Convert Binary to Text")**  
   - Operation: binaryToPropery  
   - Connect input from "Download Product Image"  
   - No additional parameters needed.  

5. **Add a Google Drive node ("Download Influencer Image")**  
   - Operation: Download  
   - Connect input from "Convert Binary to Text"  
   - Set file ID from the `Model url` field (using expression)  
   - Use Google Drive OAuth2 credentials.  

6. **Add an Extract From File node ("influencer_image_Convert_to_Text")**  
   - Operation: binaryToPropery  
   - Connect input from "Download Influencer Image"  
   - No extra configuration required.  

7. **Add an OpenAI node ("Analyze image")**  
   - Resource: Image  
   - Operation: Analyze  
   - Model: GPT-4O-MINI  
   - Input Type: base64  
   - Text prompt: Provide detailed instructions to generate an ad-ready description focusing on product presentation, lighting, styling, environment, mood, excluding specific model details.  
   - Connect input from "Download Product Image" (to get base64 product image)  
   - Configure OpenAI credentials accordingly.  

8. **Add an HTTP Request node ("Image Generation")**  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Headers:  
     - Authorization: Bearer YOUR_OPENROUTER_API_KEY  
     - Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "model": "google/gemini-2.5-flash-image-preview",
       "prompt": "{{ $json.content }}",
       "image_inputs": [
         {
           "image": "{{ $json.data }}",
           "mime_type": "image/png"
         },
         {
           "image": "{{ $('Convert Binary to Text').item.json.data }}",
           "mime_type": "image/jpeg"
         }
       ]
     }
     ```  
   - Connect inputs from "Analyze image" (for prompt) and "influencer_image_Convert_to_Text" (for influencer image base64).  

9. **Add a Code node ("base64 cleanup")**  
   - Paste the provided JS code that removes the `data:image/png;base64,` prefix from the AI-generated image URL.  
   - Connect input from "Image Generation".  

10. **Add a Convert to File node ("Convert to File")**  
    - Operation: toBinary  
    - Source property: `choices[0].images[0].image_url.url` (path to cleaned base64)  
    - Connect input from "base64 cleanup".  

11. **Add a Google Drive node ("upload_image")**  
    - Operation: Upload  
    - File name: `Ad Image {{ $runIndex + 1 }}` (dynamic)  
    - Drive: "My Drive"  
    - Folder ID: Your designated output folder in Google Drive  
    - Connect input from "Convert to File"  
    - Provide Google Drive OAuth2 credentials.  

12. **Add a Google Sheets node ("Update Spreadsheet with Results")**  
    - Operation: AppendOrUpdate Row  
    - Document ID and Sheet Name: same as initial Google Sheet  
    - Matching column: `Date`  
    - Update columns:  
      - `Date` (copy from original row)  
      - `Publish` set to string "Publish"  
      - `Ad Image Ready To Post` set to `webContentLink` from uploaded file  
    - Connect input from "upload_image" node  
    - Provide Google Sheets OAuth2 credentials.  

13. **Connect the nodes as per the flow described:**  
    - Schedule Trigger → Get the Raw → Download Product Image → Convert Binary to Text → Download Influencer Image → influencer_image_Convert_to_Text → Image Generation → base64 cleanup → Convert to File → upload_image → Update Spreadsheet with Results  
    - Also, "Download Product Image" connects to "Analyze image" for AI prompt generation  
    - "Analyze image" output connects to "Image Generation"  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow leverages OpenAI GPT-4O-MINI for text-based image analysis and Google Gemini (via OpenRouter API) for image generation.                      | OpenAI API and OpenRouter API credentials required                                                   |
| Google Workspace OAuth2 credentials needed for Sheets and Drive nodes (Download & Upload).                                                                | Google Cloud Console OAuth2 setup                                                                      |
| The workflow is designed for daily automated runs, ideal for e-commerce or marketing teams to streamline ad image creation.                              | Workflow can be adapted for different schedules or manual triggers                                    |
| Sticky notes in the workflow provide stepwise documentation of each logical block for user clarity.                                                     | Useful for onboarding and maintenance                                                                 |
| OpenRouter API key and Google Drive folder IDs must be replaced with actual values before running the workflow.                                          | https://openrouter.ai/docs/                                                                            |
| The workflow avoids including specific model details in AI prompts to keep generated ads generic and reusable.                                           | Enhances versatility of generated images                                                              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---