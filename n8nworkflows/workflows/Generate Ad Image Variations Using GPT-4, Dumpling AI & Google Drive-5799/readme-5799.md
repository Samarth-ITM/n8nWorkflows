Generate Ad Image Variations Using GPT-4, Dumpling AI & Google Drive

https://n8nworkflows.xyz/workflows/generate-ad-image-variations-using-gpt-4--dumpling-ai---google-drive-5799


# Generate Ad Image Variations Using GPT-4, Dumpling AI & Google Drive

### 1. Workflow Overview

This workflow automates the generation of 10 subtle, AI-driven visual variations of a reference ad image for direct-to-consumer (DTC) brands, leveraging GPT-4, Dumpling AI, and Google Drive integrations. It is designed to help marketers and creatives test different ad image styles while preserving the core product framing and subject.

**Target Use Case:**  
Generate controlled, performance-testable variations of existing product ad images by modifying background, lighting, mood, and color treatments. The workflow is tailored for e-commerce brands seeking to optimize Facebook/Instagram ad creatives without creating entirely new compositions.

**Logical Blocks:**

- **1.1 Input Reception & Upload**: Collect brand info and reference image from user; upload to Google Drive.  
- **1.2 Image & Brand Style Analysis**: Analyze the uploaded image’s visual style and the brand website’s aesthetics using GPT-4 models.  
- **1.3 Prompt Generation for Variations**: Use a LangChain agent (GPT-4o) to generate 10 tightly related image variation prompts based on the analyses.  
- **1.4 Prompt Processing & Image Generation Loop**: Split prompts, download base image, and generate image variations via Dumpling AI in a loop.  
- **1.5 Logging**: Record generated image URLs into a Google Sheet for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Upload

- **Overview:**  
  Collects brand name, brand website URL, and a reference ad image from the user through a web form, then uploads the image to a specified Google Drive folder.

- **Nodes Involved:**  
  - Submit Brand Info + Image  
  - Upload Ad Image to Google Drive  
  - Download Ad Image for Analysis

- **Node Details:**

  1. **Submit Brand Info + Image**  
     - **Type:** Form Trigger  
     - **Role:** Entry point for user input, capturing brand name, website, and a single image file.  
     - **Config:** Form titled "Ad Image Generator" with required fields for Brand Name, Brand Website, and one Ad Image upload.  
     - **Inputs:** User web submission.  
     - **Outputs:** JSON with the submitted fields and image binary data.  
     - **Failure modes:** User cancellation, missing required fields, file upload errors.

  2. **Upload Ad Image to Google Drive**  
     - **Type:** Google Drive - Upload File  
     - **Role:** Uploads the submitted ad image file to Google Drive under a specific folder ("n8n Testing").  
     - **Config:** Filename preserves original; folderId set to a known Drive folder; uses OAuth2 credentials.  
     - **Inputs:** Image binary data from form trigger node.  
     - **Outputs:** JSON containing uploaded file metadata including file ID.  
     - **Failure modes:** Google Drive auth errors, quota limits, network issues.

  3. **Download Ad Image for Analysis**  
     - **Type:** Google Drive - Download File  
     - **Role:** Downloads the uploaded image to prepare it for AI visual style analysis.  
     - **Config:** Uses uploaded file’s ID from previous step.  
     - **Inputs:** File ID from upload node output.  
     - **Outputs:** Base64 or binary image data for AI node.  
     - **Failure modes:** File not found, permissions errors, network failures.

---

#### 2.2 Image & Brand Style Analysis

- **Overview:**  
  Two GPT-4 powered analyses run sequentially: one describes the visual style of the reference image; the other analyzes the brand’s website aesthetics.

- **Nodes Involved:**  
  - Describe Visual Style of Image  
  - Analyze Brand Website Style

- **Node Details:**

  1. **Describe Visual Style of Image**  
     - **Type:** LangChain OpenAI Node (Image Analysis)  
     - **Role:** Produces a textual description of the image’s visual style, including subject matter, composition, lighting, and camera angle.  
     - **Config:** Uses GPT-4o model; input type is base64 image; prompt requests detailed yet concise visual description.  
     - **Inputs:** Base64 image from Google Drive download.  
     - **Outputs:** Text description of image style.  
     - **Failure modes:** OpenAI API limits, image decoding errors, model timeout.

  2. **Analyze Brand Website Style**  
     - **Type:** LangChain OpenAI Node (Chat Completion)  
     - **Role:** Analyzes the brand’s website content focusing on visual aesthetics like color palette, photography style, mood, and layout patterns.  
     - **Config:** Uses GPT-4 model; prompt includes brand name and website URL extracted from form data; instructs to focus only on visual brand identity.  
     - **Inputs:** Brand website URL and name from form; prior image style description output chained as context.  
     - **Outputs:** Text summary of visual brand style.  
     - **Failure modes:** OpenAI API errors, website content inaccessible, or non-visual content.

---

#### 2.3 Prompt Generation for Variations

- **Overview:**  
  A specialized LangChain Agent uses combined image and brand style analyses to generate 10 tightly related image variation prompts, focusing on subtle creative changes without altering product framing.

- **Nodes Involved:**  
  - LangChain Agent: Generate Variation Prompts  
  - GPT-4o (Connected to LangChain Agent)  
  - Parse Prompts into JSON Array

- **Node Details:**

  1. **LangChain Agent: Generate Variation Prompts**  
     - **Type:** LangChain Agent Node  
     - **Role:** Generates 10 image variation prompts in JSON array format based on input data.  
     - **Config:** System message includes brand info, website, visual aesthetic description, and reference image description. Instructions emphasize subtle variations for ad testing (lighting, background, mood). Output strictly JSON array of objects with `prompt` properties.  
     - **Inputs:** Brand/website info, image style, brand style from previous nodes; GPT-4o language model node connected as the LM backend.  
     - **Outputs:** Raw JSON text with 10 prompts.  
     - **Failure modes:** Parsing issues, model response formatting errors, API failures.

  2. **GPT-4o (Connected to LangChain Agent)**  
     - **Type:** LangChain LM Chat Node  
     - **Role:** Provides GPT-4o-mini model as language model backend for the LangChain Agent node.  
     - **Config:** Model set to gpt-4o-mini; uses OpenAI credentials.  
     - **Inputs:** Connected internally by LangChain Agent.  
     - **Outputs:** Language model completions.  
     - **Failure modes:** OpenAI API limits, unauthorized credentials.

  3. **Parse Prompts into JSON Array**  
     - **Type:** LangChain Output Parser (Structured JSON)  
     - **Role:** Parses the raw JSON string output from the LangChain Agent into usable JSON array format for splitting.  
     - **Config:** JSON schema example provided to guide parsing.  
     - **Inputs:** Raw text from LangChain Agent.  
     - **Outputs:** Parsed JSON array of prompt objects.  
     - **Failure modes:** Invalid JSON output, parsing exceptions.

---

#### 2.4 Prompt Processing & Image Generation Loop

- **Overview:**  
  Splits the 10 prompts into individual items, downloads the base image for each variation cycle, and calls Dumpling AI to generate each image variation in a controlled batch loop.

- **Nodes Involved:**  
  - Split: One Prompt per Item  
  - Download Base Image for Each Variation  
  - Loop: Process Image Variations  
  - Dumpling AI: Generate Image Variation

- **Node Details:**

  1. **Split: One Prompt per Item**  
     - **Type:** Split Out  
     - **Role:** Splits the JSON array of prompts into separate workflow items for parallel processing.  
     - **Config:** Field to split is `output` (the parsed JSON array).  
     - **Inputs:** Parsed JSON array from prompt parser.  
     - **Outputs:** Individual prompt objects.  
     - **Failure modes:** Empty array, field naming errors.

  2. **Download Base Image for Each Variation**  
     - **Type:** Google Drive - Download File  
     - **Role:** Downloads the original reference image again for each image variation generation cycle (likely needed by Dumpling AI).  
     - **Config:** Uses the uploaded image ID from earlier Google Drive upload node.  
     - **Inputs:** File ID from upload node.  
     - **Outputs:** Base image binary/base64 data for Dumpling API.  
     - **Failure modes:** File not found, rate limits.

  3. **Loop: Process Image Variations**  
     - **Type:** Split In Batches  
     - **Role:** Controls iteration over each prompt/image variation pair to manage API calls and processing load.  
     - **Config:** Defaults to batch size 1 (one prompt per iteration).  
     - **Inputs:** Items from split node and base image download.  
     - **Outputs:** Processed variation items.  
     - **Failure modes:** Infinite loops, batch size misconfigurations.

  4. **Dumpling AI: Generate Image Variation**  
     - **Type:** HTTP Request  
     - **Role:** Sends prompt text to Dumpling AI API to generate a new AI image variation.  
     - **Config:** POST request to `https://app.dumplingai.com/api/v1/generate-ai-image` with JSON body containing the prompt. Uses HTTP Header authentication with supplied API key.  
     - **Inputs:** Prompt text from current batch item.  
     - **Outputs:** JSON response with generated image URL.  
     - **Failure modes:** API rate limits, network errors, invalid prompt format, auth failures.

---

#### 2.5 Logging

- **Overview:**  
  Logs each generated image URL into a Google Sheet for record-keeping and subsequent review.

- **Nodes Involved:**  
  - Log Image Variation URLs to Google Sheets

- **Node Details:**

  1. **Log Image Variation URLs to Google Sheets**  
     - **Type:** Google Sheets - Append Row  
     - **Role:** Appends each generated image URL as a new row in a predefined Google Sheet.  
     - **Config:** Target sheet with single column "Image URL"; uses OAuth2 credentials.  
     - **Inputs:** URL from Dumpling AI node JSON response.  
     - **Outputs:** Confirmation of row appended.  
     - **Failure modes:** Sheet permission errors, quota exceeded, network issues.

---

### 3. Summary Table

| Node Name                          | Node Type                               | Functional Role                                 | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                                  |
|-----------------------------------|---------------------------------------|------------------------------------------------|----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Submit Brand Info + Image          | Form Trigger                          | Collect user input (brand info + image)        | —                                | Upload Ad Image to Google Drive     |                                                                                                              |
| Upload Ad Image to Google Drive    | Google Drive (Upload)                 | Upload user image to Drive folder               | Submit Brand Info + Image         | Download Ad Image for Analysis       |                                                                                                              |
| Download Ad Image for Analysis     | Google Drive (Download)               | Download uploaded image for AI analysis         | Upload Ad Image to Google Drive   | Describe Visual Style of Image       |                                                                                                              |
| Describe Visual Style of Image     | LangChain OpenAI (Image Analysis)    | Analyze image visual style with GPT-4o          | Download Ad Image for Analysis    | Analyze Brand Website Style          |                                                                                                              |
| Analyze Brand Website Style        | LangChain OpenAI (Chat Completion)   | Analyze brand website visual aesthetics         | Describe Visual Style of Image    | LangChain Agent: Generate Variation Prompts |                                                                                                              |
| LangChain Agent: Generate Variation Prompts | LangChain Agent                  | Generate 10 image variation prompts              | Analyze Brand Website Style, GPT-4o (Connected) | Split: One Prompt per Item            |                                                                                                              |
| GPT-4o (Connected to LangChain Agent) | LangChain LM Chat OpenAI           | GPT-4o-mini model backend for LangChain Agent   | —                                | LangChain Agent: Generate Variation Prompts |                                                                                                              |
| Parse Prompts into JSON Array      | LangChain Output Parser (Structured) | Parse LangChain Agent output into JSON array    | LangChain Agent: Generate Variation Prompts | Split: One Prompt per Item            |                                                                                                              |
| Split: One Prompt per Item         | Split Out                           | Split JSON array of prompts into individual items | Parse Prompts into JSON Array     | Download Base Image for Each Variation |                                                                                                              |
| Download Base Image for Each Variation | Google Drive (Download)           | Download original image for each variation      | Split: One Prompt per Item        | Loop: Process Image Variations       |                                                                                                              |
| Loop: Process Image Variations     | Split In Batches                    | Iterate over each prompt/image pair in batches  | Download Base Image for Each Variation | Dumpling AI: Generate Image Variation, Log Image Variation URLs to Google Sheets |                                                                                                              |
| Dumpling AI: Generate Image Variation | HTTP Request                     | Generate new image variation via Dumpling AI API | Loop: Process Image Variations (Batch output) | Log Image Variation URLs to Google Sheets |                                                                                                              |
| Log Image Variation URLs to Google Sheets | Google Sheets (Append)           | Log generated image URLs in Google Sheet        | Dumpling AI: Generate Image Variation | Loop: Process Image Variations       |                                                                                                              |
| Sticky Note                       | Sticky Note                         | Documentation note explaining workflow purpose | —                                | —                                   | ### 🖼️ Ad Image Variation Generator (Using GPT-4 + Dumpling AI) This workflow creates 10 subtle creative variations of a reference ad image to test performance across visual styles, lighting, background, and tone — while preserving the product's framing and subject. --- 🔧 How It Works 1. User submits brand name, website, and reference ad image via a form. 2. The image is uploaded to Google Drive. 3. GPT-4o analyzes the image’s visual style (composition, subject, lighting). 4. GPT-4 analyzes the brand website to understand overall visual identity. 5. A LangChain AI Agent uses both to generate 10 variation prompts. 6. Each prompt is passed to Dumpling AI to generate a new ad image. 7. All image URLs are logged in Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Submit Brand Info + Image")**  
   - Type: Form Trigger  
   - Configure form title: "Ad Image Generator"  
   - Add three required form fields:  
     - Brand Name (text)  
     - Brand Website (text)  
     - Ad Image (file upload, single file only)  
   - Save and activate this node.

2. **Add Google Drive Upload Node ("Upload Ad Image to Google Drive")**  
   - Connect from Form Trigger node.  
   - Set operation to upload file.  
   - For filename, use expression: `{{$json["Ad Image"].filename}} (Original)`  
   - Set Drive to "My Drive" or appropriate drive.  
   - Set Folder ID to your target Google Drive folder (e.g., "n8n Testing").  
   - Set input data field to "Ad_Image" (the uploaded file from form).  
   - Apply Google Drive OAuth2 credentials.

3. **Add Google Drive Download Node ("Download Ad Image for Analysis")**  
   - Connect from Upload node.  
   - Operation: download file.  
   - File ID: use expression linking to upload node output file ID.  
   - Apply same Google Drive OAuth2 credentials.

4. **Add LangChain OpenAI Node ("Describe Visual Style of Image")**  
   - Connect from Download node.  
   - Set resource to "image" and operation to "analyze".  
   - Model: GPT-4o (or GPT-4 variant).  
   - Text prompt: "Describe the visual style, subject matter, and composition of this image..." (include lighting and camera angle).  
   - Input type: base64 (use image downloaded).  
   - Apply OpenAI API credentials.

5. **Add LangChain OpenAI Node ("Analyze Brand Website Style")**  
   - Connect from previous node.  
   - Model: GPT-4.  
   - Set up chat messages with content:  
     - Role: system or user  
     - Content: Instruction to analyze brand website’s visual aesthetic (color palette, photography style, mood, etc.).  
     - Inject brand website and name dynamically from form data using expressions.  
   - Apply OpenAI API credentials.

6. **Add LangChain Agent Node ("LangChain Agent: Generate Variation Prompts")**  
   - Connect from "Analyze Brand Website Style".  
   - Provide system message combining: brand name, website, visual style descriptions from prior nodes.  
   - Task: generate 10 JSON prompts describing subtle image variations (background, lighting, mood) preserving product framing.  
   - Set output to return a JSON array of prompts with property "prompt".  
   - Use GPT-4o-mini model node connected as LM backend.

7. **Add LangChain LM Chat Node ("GPT-4o (Connected to LangChain Agent)")**  
   - Configure with GPT-4o-mini model.  
   - Apply OpenAI credentials.  
   - Connect this node to LangChain Agent node as the language model input.

8. **Add LangChain Output Parser Node ("Parse Prompts into JSON Array")**  
   - Connect to LangChain Agent output.  
   - Configure with JSON schema example matching expected prompt array structure.

9. **Add Split Out Node ("Split: One Prompt per Item")**  
   - Connect from parser output.  
   - Configure to split on the "output" field (the parsed array of prompts).

10. **Add Google Drive Download Node ("Download Base Image for Each Variation")**  
    - Connect from Split node.  
    - Download original uploaded image by linking file ID from initial upload node.

11. **Add Split In Batches Node ("Loop: Process Image Variations")**  
    - Connect from base image download node.  
    - Configure batch size to 1 (process one prompt/image pair at a time).

12. **Add HTTP Request Node ("Dumpling AI: Generate Image Variation")**  
    - Connect as second output of Split In Batches node (batch processing output).  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`  
    - Body (JSON): `{ "model": "FLUX.1-pro", "input": { "prompt": "{{ $json.prompt }}" } }`  
    - Authentication: HTTP Header Auth with Dumpling AI API key.  
    - Apply credentials.

13. **Add Google Sheets Append Node ("Log Image Variation URLs to Google Sheets")**  
    - Connect from Dumpling AI node.  
    - Operation: Append row.  
    - Target Sheet: specify Google Sheet document and sheet name (e.g., Sheet1).  
    - Columns: map "Image URL" to `{{$json.url}}` from Dumpling AI response.  
    - Apply Google Sheets OAuth2 credentials.

14. **Connect Google Sheets node back to the Split In Batches node**  
    - To allow batch loop continuation.

15. **Add Sticky Note**  
    - Document workflow purpose and summary as provided.

16. **Test & Activate Workflow**  
    - Verify all credentials are valid and tested.  
    - Run test submissions and check output image URLs in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow uses GPT-4o (GPT-4 optimized) and GPT-4 models via LangChain integration to combine natural language understanding with image analysis.                                                                                                               | n8n LangChain AI nodes documentation                                                                         |
| Dumpling AI is employed for image generation via a secure HTTP API requiring key-based authentication.                                                                                                                                                             | https://app.dumplingai.com                                                                                   |
| Google Drive is used for secure storage and retrieval of images; Google Sheets serves as a log for generated image URLs to facilitate review and further processing.                                                                                                | Google Workspace API documentation                                                                            |
| The workflow emphasizes subtle variations (lighting, background, mood) for ad performance testing, avoiding major compositional changes or branding overlays.                                                                                                      | Sticky Note content in workflow                                                                               |
| Ensure API quotas and rate limits for OpenAI and Dumpling AI are monitored to avoid throttling during batch image generation.                                                                                                                                      | OpenAI and Dumpling AI API usage guidelines                                                                   |
| The workflow assumes the user submits a single image per run; to scale for multiple images simultaneously, workflow modifications are necessary.                                                                                                                  | n8n multi-item processing best practices                                                                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.