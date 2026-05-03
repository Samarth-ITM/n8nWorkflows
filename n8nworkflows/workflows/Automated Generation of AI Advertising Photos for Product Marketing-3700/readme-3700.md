Automated Generation of AI Advertising Photos for Product Marketing

https://n8nworkflows.xyz/workflows/automated-generation-of-ai-advertising-photos-for-product-marketing-3700


# Automated Generation of AI Advertising Photos for Product Marketing

### 1. Workflow Overview

This workflow automates the enhancement of standard product images into professional product photography featuring human models, leveraging AI capabilities. It is designed for marketers or e-commerce operators who want to transform basic product photos into visually appealing images that include human interaction or modeling, improving product presentation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Image Download:** Reads product image URLs from a Google Sheets spreadsheet and downloads the images.
- **1.2 Image Analysis:** Uses OpenAI to analyze each downloaded image to identify the product type.
- **1.3 Prompt Generation:** Creates specialized AI prompts for generating professional product photography, ensuring human models are included appropriately.
- **1.4 AI Image Generation:** Sends the original image and generated prompt to OpenAI’s image editing endpoint to create enhanced photos.
- **1.5 Output Handling:** Converts AI-generated images to files, uploads them to Google Drive, and updates the Google Sheet with new image URLs and prompts.
- **1.6 Optional Simple Image Generation:** A separate, manual-triggered sub-flow to generate images directly from text prompts without product image input.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Image Download

- **Overview:**  
  This block reads product image URLs from a Google Sheets document and downloads each image for further processing.

- **Nodes Involved:**  
  - When clicking 'Test workflow' (Manual Trigger)  
  - Read Image URLs (Google Sheets)  
  - Download Images (HTTP Request)  
  - Merge (Merge node)  
  - Sticky Note (Extract Product Images from Template)

- **Node Details:**

  - **When clicking 'Test workflow'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or execution.  
    - Inputs: None  
    - Outputs: Triggers "Read Image URLs" node.  
    - Edge cases: None specific, but workflow won’t run without manual trigger.

  - **Read Image URLs**  
    - Type: Google Sheets  
    - Role: Reads rows from a Google Sheet containing product image URLs under the column "Image-URL".  
    - Configuration: Reads from a specific spreadsheet and sheet (gid=0).  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Trigger from manual node  
    - Outputs: JSON items with "Image-URL" field.  
    - Edge cases: Empty or invalid URLs, Google Sheets API errors, OAuth token expiration.

  - **Download Images**  
    - Type: HTTP Request  
    - Role: Downloads each image using the URL from the sheet.  
    - Configuration: URL set dynamically from each item’s "Image-URL" field.  
    - Inputs: Image URLs from "Read Image URLs"  
    - Outputs: Binary image data for each URL.  
    - Edge cases: Broken URLs, HTTP errors, timeouts, unsupported image formats.

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from "Download Images" and "Product Photography Prompt" (later in flow) by position to keep data aligned.  
    - Configuration: Mode "combine", combine by position.  
    - Inputs: From "Download Images" and "Product Photography Prompt"  
    - Outputs: Combined data for next steps.  
    - Edge cases: Mismatched item counts causing misalignment.

  - **Sticky Note**  
    - Content: "## Extract Product Images from Template"  
    - Role: Documentation for this block.

---

#### 2.2 Image Analysis

- **Overview:**  
  This block analyzes each downloaded image to generate a brief description of the product it contains, which is used to tailor the photography prompt.

- **Nodes Involved:**  
  - Analyze Images (OpenAI Image Analysis)  
  - Sticky Note1 (Analyze Images and Create Prompt for Product Photography)

- **Node Details:**

  - **Analyze Images**  
    - Type: OpenAI Image Analysis (Langchain OpenAI node)  
    - Role: Sends base64-encoded image to OpenAI to get a short description (less than 5 words) of the product.  
    - Configuration: Model "gpt-4o-mini", operation "analyze", input type "base64".  
    - Inputs: Binary image data from "Download Images" converted to base64 internally.  
    - Outputs: JSON with "content" field containing the brief description.  
    - Credentials: OpenAI API key  
    - Edge cases: API rate limits, invalid image data, model errors, network issues.

---

#### 2.3 Prompt Generation

- **Overview:**  
  Generates a detailed AI prompt for image generation based on the product description, ensuring the product is shown with a human model under specific conditions.

- **Nodes Involved:**  
  - Product Photography Prompt (Langchain Chain LLM)  
  - OpenAI Chat Model (GPT-4.1-mini)  
  - Sticky Note1 (shared with Image Analysis block)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain Chat Model (OpenAI)  
    - Role: Provides advanced language model capabilities to assist prompt generation.  
    - Configuration: Model "gpt-4.1-mini".  
    - Inputs: Connected to "Product Photography Prompt" node as AI language model.  
    - Outputs: Used internally by "Product Photography Prompt".  
    - Credentials: OpenAI API key  
    - Edge cases: API limits, authentication errors.

  - **Product Photography Prompt**  
    - Type: Langchain Chain LLM  
    - Role: Constructs a specialized prompt for AI image generation based on the analyzed product description.  
    - Configuration:  
      - Input text: "Image description: {{ $json.content }}" (from "Analyze Images")  
      - Instruction message: Detailed instructions to always include a human model, specify realism, lighting, angles, and cinematic grain.  
      - Output: Final prompt string only, no extra text or quotes.  
    - Inputs: Product description from "Analyze Images" and AI model from "OpenAI Chat Model".  
    - Outputs: Text prompt for image generation.  
    - Edge cases: Expression evaluation errors, incomplete or ambiguous product descriptions.

---

#### 2.4 AI Image Generation

- **Overview:**  
  Uses OpenAI’s image editing API to generate enhanced product photos based on the original image and the generated prompt.

- **Nodes Involved:**  
  - Send Image with Prompt to OpenAI (HTTP Request)  
  - Convert Base64 to File (ConvertToFile)  
  - Sticky Note2 (gpt-image-1 creates the Product Photography)

- **Node Details:**

  - **Send Image with Prompt to OpenAI**  
    - Type: HTTP Request  
    - Role: Calls OpenAI’s image editing endpoint to create new images using the prompt and original image.  
    - Configuration:  
      - URL: https://api.openai.com/v1/images/edits  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Parameters: model "gpt-image-1", prompt from "Product Photography Prompt", image binary data, quality "high", size "1536x1024"  
    - Inputs: Original image binary and prompt text  
    - Outputs: Base64-encoded edited image data  
    - Credentials: OpenAI API key  
    - Edge cases: API errors, invalid image or prompt, network timeouts.

  - **Convert Base64 to File**  
    - Type: ConvertToFile  
    - Role: Converts base64 image data from OpenAI response into a binary file format for upload.  
    - Configuration: Source property "data[0].b64_json"  
    - Inputs: Base64 image from "Send Image with Prompt to OpenAI"  
    - Outputs: Binary file data  
    - Edge cases: Invalid base64 data, conversion failures.

---

#### 2.5 Output Handling

- **Overview:**  
  Uploads the generated images to Google Drive and updates the Google Sheet with new image URLs and prompts.

- **Nodes Involved:**  
  - Upload to Drive (Google Drive)  
  - Insert Image URL in Table (Google Sheets)  
  - Sticky Note3 (Output is uploaded to Drive and the Image URLs are saved in the table)

- **Node Details:**

  - **Upload to Drive**  
    - Type: Google Drive  
    - Role: Uploads the converted image file to a specified Google Drive folder.  
    - Configuration:  
      - Folder ID: Specific folder for product images  
      - File name: Uses product description from "Analyze Images" content field  
      - Drive: "My Drive"  
    - Inputs: Binary file from "Convert Base64 to File"  
    - Outputs: Metadata including webViewLink (URL) of uploaded image  
    - Credentials: Google Drive OAuth2  
    - Edge cases: Permission errors, quota limits, network issues.

  - **Insert Image URL in Table**  
    - Type: Google Sheets  
    - Role: Updates the original Google Sheet row matching "Image-URL" with new columns: "Output" (uploaded image URL) and "Prompt" (generated prompt).  
    - Configuration:  
      - Matching column: "Image-URL"  
      - Columns updated: Output, Prompt  
      - Sheet and document IDs same as input sheet  
    - Inputs: Uploaded image URL, prompt text, original image URL  
    - Outputs: Confirmation of update  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Row matching failures, API errors, OAuth token expiration.

---

#### 2.6 Optional Simple Image Generation

- **Overview:**  
  A separate manual-triggered workflow branch to generate images directly from a fixed text prompt without product image input.

- **Nodes Involved:**  
  - Manual Trigger (When clicking 'Test workflow')  
  - Generate Image (HTTP Request to OpenAI)  
  - Convert to File (ConvertToFile)  
  - Sticky Note4 (Simple Image Generation reminder)

- **Node Details:**

  - **Generate Image**  
    - Type: HTTP Request  
    - Role: Calls OpenAI’s image generation endpoint with a fixed prompt for a children’s book style image.  
    - Configuration:  
      - URL: https://api.openai.com/v1/images/generations  
      - Method: POST  
      - Parameters: model "gpt-image-1", prompt fixed text  
    - Inputs: Manual trigger  
    - Outputs: Base64-encoded generated image data  
    - Credentials: OpenAI API key  
    - Edge cases: API errors, rate limits.

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts base64 image data to binary file.  
    - Configuration: Source property "data[0].b64_json"  
    - Inputs: Base64 image from "Generate Image"  
    - Outputs: Binary file data  
    - Edge cases: Conversion errors.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                                   | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                 |
|-------------------------------|----------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|---------------------------------------------------------------------------------------------|
| When clicking 'Test workflow'  | Manual Trigger                   | Starts the workflow manually                      | None                          | Read Image URLs, Generate Image  |                                                                                             |
| Read Image URLs                | Google Sheets                   | Reads product image URLs from spreadsheet        | When clicking 'Test workflow'  | Download Images                 |                                                                                             |
| Download Images               | HTTP Request                    | Downloads images from URLs                        | Read Image URLs               | Analyze Images, Merge           |                                                                                             |
| Analyze Images                | OpenAI Image Analysis (Langchain) | Analyzes images to generate product descriptions | Download Images               | Product Photography Prompt      | ## Analyze Images and Create Prompt for Product Photography                                 |
| Product Photography Prompt    | Langchain Chain LLM             | Generates AI prompt for image generation         | Analyze Images, OpenAI Chat Model | Merge                         | ## Analyze Images and Create Prompt for Product Photography                                 |
| OpenAI Chat Model             | Langchain Chat Model            | Provides advanced language model for prompt      | None (used internally)         | Product Photography Prompt      |                                                                                             |
| Merge                        | Merge                          | Combines image data and prompt data               | Download Images, Product Photography Prompt | Send Image with Prompt to OpenAI |                                                                                             |
| Send Image with Prompt to OpenAI | HTTP Request                  | Sends image and prompt to OpenAI for editing     | Merge                         | Convert Base64 to File          | ## gpt-image-1 creates the Product Photography                                              |
| Convert Base64 to File        | ConvertToFile                  | Converts base64 image data to binary file         | Send Image with Prompt to OpenAI | Upload to Drive               | ## gpt-image-1 creates the Product Photography                                              |
| Upload to Drive               | Google Drive                   | Uploads generated images to Google Drive          | Convert Base64 to File        | Insert Image URL in Table       | ## Output is uploaded to Drive and the Image URLs are saved in the table                    |
| Insert Image URL in Table     | Google Sheets                  | Updates spreadsheet with new image URLs and prompts | Upload to Drive              | None                          | ## Output is uploaded to Drive and the Image URLs are saved in the table                    |
| Generate Image               | HTTP Request                   | Generates images from fixed text prompt (optional) | When clicking 'Test workflow' | Convert to File                | ## Simple Image Generation\n### Don't forget the manual trigger ;)                          |
| Convert to File              | ConvertToFile                  | Converts base64 generated image to binary file    | Generate Image                | None                          | ## Simple Image Generation\n### Don't forget the manual trigger ;)                          |
| Sticky Note                  | Sticky Note                   | Documentation block                               | None                          | None                          | ## Extract Product Images from Template                                                     |
| Sticky Note1                 | Sticky Note                   | Documentation block                               | None                          | None                          | ## Analyze Images and Create Prompt for Product Photography                                 |
| Sticky Note2                 | Sticky Note                   | Documentation block                               | None                          | None                          | ## gpt-image-1 creates the Product Photography                                              |
| Sticky Note3                 | Sticky Note                   | Documentation block                               | None                          | None                          | ## Output is uploaded to Drive and the Image URLs are saved in the table                    |
| Sticky Note4                 | Sticky Note                   | Documentation block                               | None                          | None                          | ## Simple Image Generation\n### Don't forget the manual trigger ;)                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking 'Test workflow'"  
   - Purpose: To manually start the workflow.

2. **Create Google Sheets Node to Read Image URLs**  
   - Name: "Read Image URLs"  
   - Operation: Read rows from your Google Sheet  
   - Configure:  
     - Document ID: Your Google Sheet ID containing product images  
     - Sheet Name: The sheet or gid (e.g., "gid=0")  
     - Credentials: Google Sheets OAuth2  
   - Output: Rows with column "Image-URL"

3. **Create HTTP Request Node to Download Images**  
   - Name: "Download Images"  
   - Method: GET  
   - URL: Set dynamically to `={{ $json["Image-URL"] }}`  
   - Output: Binary image data  
   - Connect input from "Read Image URLs"

4. **Create OpenAI Image Analysis Node**  
   - Name: "Analyze Images"  
   - Type: OpenAI Image Analysis (Langchain OpenAI node)  
   - Model: "gpt-4o-mini"  
   - Operation: Analyze image  
   - Input Type: base64  
   - Text prompt: "Briefly explain in less than 5 words what this image is about."  
   - Credentials: OpenAI API key  
   - Connect input from "Download Images"

5. **Create Langchain Chain LLM Node for Prompt Generation**  
   - Name: "Product Photography Prompt"  
   - Input Text: `=Image description: {{ $json.content }}` (from "Analyze Images")  
   - Messages:  
     - Instruction:  
       ```
       Create a short prompt for an AI image generator that receives a photo of a product to ultimately produce professional product photography.

       If the product is wearable, it must be worn by a human model with visible face; if it's not wearable, it must be held or interacted with by a model.

       The product must ALWAYS be shown together with a human model with the model's face visible.

       Ensure that instructions for optimal realism, best lighting, best angle, best colors, best model positioning, etc. are included according to the product type.

       Always formulate the prompt to refer to the product as "this [PRODUCT]" so the AI image generator knows that an input photo of the product is being submitted.

       Always add subtle grain for a cinematic look.
       The description of the product will be sent to you. Respond exclusively with the final prompt, nothing else, not even quotation marks.
       ```  
   - Model: Use "OpenAI Chat Model" node with "gpt-4.1-mini" as language model  
   - Connect input from "Analyze Images" and link to "OpenAI Chat Model"

6. **Create Merge Node**  
   - Name: "Merge"  
   - Mode: Combine  
   - Combine By: Position  
   - Connect inputs from "Download Images" and "Product Photography Prompt"

7. **Create HTTP Request Node to Send Image and Prompt to OpenAI**  
   - Name: "Send Image with Prompt to OpenAI"  
   - Method: POST  
   - URL: https://api.openai.com/v1/images/edits  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - model: "gpt-image-1"  
     - prompt: `={{ $json.text }}` (from "Product Photography Prompt")  
     - image[]: binary data from "Download Images"  
     - quality: "high"  
     - size: "1536x1024"  
   - Credentials: OpenAI API key  
   - Connect input from "Merge"

8. **Create ConvertToFile Node**  
   - Name: "Convert Base64 to File"  
   - Operation: toBinary  
   - Source Property: "data[0].b64_json" (from OpenAI response)  
   - Connect input from "Send Image with Prompt to OpenAI"

9. **Create Google Drive Node to Upload Image**  
   - Name: "Upload to Drive"  
   - Operation: Upload file  
   - Folder ID: Your Google Drive folder ID for product images  
   - File Name: Use product description from "Analyze Images" content field  
   - Drive: "My Drive"  
   - Credentials: Google Drive OAuth2  
   - Connect input from "Convert Base64 to File"

10. **Create Google Sheets Node to Update Table**  
    - Name: "Insert Image URL in Table"  
    - Operation: Update row  
    - Document ID and Sheet Name: Same as "Read Image URLs"  
    - Matching Column: "Image-URL"  
    - Columns to update:  
      - Output: `={{ $json.webViewLink }}` (from Google Drive upload)  
      - Prompt: `={{ $('Product Photography Prompt').item.json.text }}`  
      - Image-URL: `={{ $('Read Image URLs').item.json['Image-URL'] }}`  
    - Credentials: Google Sheets OAuth2  
    - Connect input from "Upload to Drive"

11. **Optional: Create Simple Image Generation Branch**  
    - Manual Trigger (reuse "When clicking 'Test workflow'")  
    - HTTP Request Node "Generate Image"  
      - POST to https://api.openai.com/v1/images/generations  
      - Parameters: model "gpt-image-1", prompt fixed text (e.g., "A childrens book drawing of a veterinarian using a stethoscope to listen to the heartbeat of a baby otter.")  
      - Credentials: OpenAI API key  
    - ConvertToFile Node "Convert to File"  
      - Source Property: "data[0].b64_json"  
    - Connect "Generate Image" output to "Convert to File"

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow requires OpenAI API key with access to "gpt-image-1" model for image editing and generation | OpenAI API documentation: https://platform.openai.com/docs/models/gpt-image-1                                   |
| Google Sheets must have columns: "Image-URL", "Prompt", and "Output"                                | Spreadsheet example link: https://docs.google.com/spreadsheets/d/17zQUytFekDK305wvgxYdEYm4N5QEQ1mrwsfccNn872I   |
| Google Drive folder must be created and folder ID configured in "Upload to Drive" node              | Example folder link: https://drive.google.com/drive/folders/1mAV3g0eR5XZ2wknZTbcfZOkLlq8GZryP                    |
| The prompt generation enforces inclusion of human models with visible faces for realism            | Ensures marketing images meet professional standards for product presentation                                   |
| Manual trigger node is necessary to start the workflow                                              | Reminder in Sticky Note4: "Don't forget the manual trigger ;)"                                                  |
| The workflow merges image data and prompt data by position, so item counts must match              | Mismatched data can cause errors or misaligned processing                                                       |

---

This documentation fully describes the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.