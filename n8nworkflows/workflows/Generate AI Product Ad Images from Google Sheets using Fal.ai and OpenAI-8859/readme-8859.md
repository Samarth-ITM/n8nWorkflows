Generate AI Product Ad Images from Google Sheets using Fal.ai and OpenAI

https://n8nworkflows.xyz/workflows/generate-ai-product-ad-images-from-google-sheets-using-fal-ai-and-openai-8859


# Generate AI Product Ad Images from Google Sheets using Fal.ai and OpenAI

### 1. Workflow Overview

This n8n workflow automates the generation of AI-based product advertisement images by integrating Google Sheets, OpenAI, and Fal.ai services. It is designed for marketing teams or digital content creators who want to produce multiple distinct ad image prompts and corresponding visuals efficiently from product data maintained in Google Sheets.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception and Data Retrieval:** Triggered manually, it fetches product rows from a Google Sheet where the status is "Create".
- **1.2 Image Preparation and Analysis:** Downloads the product image from Google Drive and analyzes it using OpenAI’s image analysis capabilities to generate structured insights.
- **1.3 AI Prompt Generation:** Uses LangChain’s AI agent with OpenAI models to generate multiple distinct ad image prompts based on the product image, description, and optional campaign data.
- **1.4 Prompt Parsing and Splitting:** Parses the AI-generated JSON output and splits the multiple prompts for individual processing.
- **1.5 Image Generation via Fal.ai:** Sends each prompt to Fal.ai’s image generation API (nano banana model), checks processing status, and retrieves the generated image.
- **1.6 File Upload and Status Update:** Uploads the generated image back to Google Drive and updates the Google Sheet row with the prompt, status, and output URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Retrieval

**Overview:**  
This block triggers the workflow manually and retrieves product rows from a Google Sheet where the status is "Create" to process them.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get row(s) in sheet

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Config: No parameters; trigger is manual.  
  - Inputs: None  
  - Outputs: To "Get row(s) in sheet" node  
  - Potential Failures: None (manual trigger)  

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves rows with status "Create" from the "product" sheet in a specific Google Sheets document.  
  - Config: Filter rows with `status` equal to "Create"; targets sheet with gid=0; uses Google Sheets OAuth2 credentials.  
  - Inputs: From manual trigger  
  - Outputs: To "Download file" node  
  - Potential Failures: Authentication errors, sheet access issues, empty result if no rows match.  

---

#### 2.2 Image Preparation and Analysis

**Overview:**  
Downloads the product image from Google Drive and analyzes it using OpenAI’s image analysis model specialized for advertising to produce structured product insights.

**Nodes Involved:**  
- Download file  
- Analyze image

**Node Details:**  

- **Download file**  
  - Type: Google Drive  
  - Role: Downloads the product image file from a Google Drive URL extracted from the sheet data.  
  - Config: File ID extracted dynamically from the product image URL in the sheet; uses Google Drive OAuth2 credentials.  
  - Inputs: Rows from Google Sheets  
  - Outputs: Base64 image data to "Analyze image"  
  - Edge Cases: Invalid URLs, permission denied for file, file not found.  

- **Analyze image**  
  - Type: OpenAI Image Analysis (LangChain)  
  - Role: Performs AI-based image analysis to identify product type, features, style, target audience, and ad creative angles, outputting JSON.  
  - Config: GPT-4o-mini model; multi-point instruction prompt specialized for ad image analysis; input type is base64 image data.  
  - Inputs: Base64 image from "Download file"  
  - Outputs: JSON insights to "AI Agent"  
  - Edge Cases: Image too complex or unclear, API timeout, malformed output.  

---

#### 2.3 AI Prompt Generation

**Overview:**  
Generates multiple distinct production-ready advertising image prompts using AI, combining the product image analysis, description, campaign context, and brand constraints.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**  

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Receives product image URL, name, description, campaign, model target, aspect ratio, number of variations, brand notes, and constraints to generate N distinct ad image prompts in JSON format.  
  - Config: System prompt with detailed instructions for prompt generation; uses GPT-4.1-mini via OpenAI Chat Model.  
  - Inputs: JSON insights from "Analyze image" and Google Sheets data via expressions.  
  - Outputs: Structured JSON to "Split Out" after parsing.  
  - Edge Cases: Missing data fields, ambiguous campaign, API limits, JSON formatting errors.  

- **OpenAI Chat Model**  
  - Type: LangChain Chat OpenAI  
  - Role: Provides the underlying language model (GPT-4.1-mini) used by the AI Agent for prompt generation.  
  - Config: Model selection, OpenAI API credentials.  
  - Inputs/Outputs: Connected internally to AI Agent.  
  - Edge Cases: API authentication failure, throttling.  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI Agent output JSON into structured data for downstream use.  
  - Config: Example JSON schema matching expected output with scenes and prompts.  
  - Inputs: Raw AI Agent output  
  - Outputs: Parsed JSON to AI Agent node main output  
  - Edge Cases: Parsing failure, malformed JSON.  

---

#### 2.4 Prompt Parsing and Splitting

**Overview:**  
Splits the array of generated prompt scenes into individual items for sequential image generation.

**Nodes Involved:**  
- Split Out  
- Append row in sheet  
- Edit Fields

**Node Details:**  

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array field `output.scenes` from the AI Agent output into individual messages for parallel processing.  
  - Config: Splitting on field `output.scenes`.  
  - Inputs: AI Agent output JSON  
  - Outputs: Individual scenes to "Append row in sheet"  
  - Edge Cases: Empty array, missing field.  

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Appends a new row to the "ad_image" sheet with the generated prompt, status "Ready", scene reference, and product name for each prompt.  
  - Config: Defines columns for prompt, status, scene_ref, product_name; uses Google Sheets OAuth2 credentials.  
  - Inputs: Individual scene JSON from "Split Out"  
  - Outputs: To "Edit Fields"  
  - Edge Cases: Sheet access issues, data format mismatch.  

- **Edit Fields**  
  - Type: Set  
  - Role: Extracts and constructs the product image URL from Google Drive file ID for API compatibility.  
  - Config: Expression extracts file ID from various URL formats and builds a direct view URL.  
  - Inputs: Original sheet data and appended row data  
  - Outputs: To "Call Fal.ai API (nannoBanana)"  
  - Edge Cases: Malformed URLs, missing ID extraction.  

---

#### 2.5 Image Generation via Fal.ai

**Overview:**  
Submits each prompt and image URL to Fal.ai API to generate advertising images, then polls for generation status and retrieves the final image.

**Nodes Involved:**  
- Call Fal.ai API (nannoBanana)  
- Get image status  
- If  
- Wait  
- Get the image1  
- HTTP Request

**Node Details:**  

- **Call Fal.ai API (nannoBanana)**  
  - Type: HTTP Request  
  - Role: Posts prompt and product image URL to Fal.ai nano banana model API to request image generation.  
  - Config: POST method; JSON body includes prompt, image URL, number of images=1, output_format=jpeg; uses Fal AI HTTP header auth credentials.  
  - Inputs: Edited fields from previous node  
  - Outputs: To "Get image status"  
  - Edge Cases: API errors, auth failure, malformed JSON.  

- **Get image status**  
  - Type: HTTP Request  
  - Role: Polls Fal.ai API with status URL to check image generation progress.  
  - Config: Uses URL from previous response; authenticated with Fal AI credentials.  
  - Inputs: From "Call Fal.ai API" or "Wait"  
  - Outputs: To "If" conditional node  
  - Edge Cases: Timeout, network errors, invalid URLs.  

- **If**  
  - Type: If (conditional)  
  - Role: Checks if image generation status returned by Fal.ai is "COMPLETED".  
  - Config: Condition equals "COMPLETED" on status field.  
  - Inputs: From "Get image status"  
  - Outputs: If true → "Get the image1", else → "Wait"  
  - Edge Cases: Unexpected status values, missing field.  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 10 seconds before re-checking image status.  
  - Config: Fixed 10 seconds wait  
  - Inputs: From "If" false branch  
  - Outputs: Back to "Get image status"  
  - Edge Cases: Delay in image generation, infinite loop if status never completes.  

- **Get the image1**  
  - Type: HTTP Request  
  - Role: Retrieves the completed generated image URL from Fal.ai request once ready.  
  - Config: URL constructed with Fal.ai API and request ID; authenticated with Fal AI credentials; retry enabled on failure.  
  - Inputs: From "If" true branch  
  - Outputs: To "HTTP Request" node  
  - Edge Cases: Image URL not available, API errors.  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file from Fal.ai URL.  
  - Config: Uses URL from previous step; authenticated with Fal AI credentials.  
  - Inputs: From "Get the image1"  
  - Outputs: To "Upload file"  
  - Edge Cases: Download failure, network issues.  

---

#### 2.6 File Upload and Status Update

**Overview:**  
Uploads the generated ad image to a designated Google Drive folder and updates the corresponding Google Sheet row with the image URL and status.

**Nodes Involved:**  
- Upload file  
- Update row in sheet

**Node Details:**  

- **Upload file**  
  - Type: Google Drive  
  - Role: Uploads the generated JPEG ad image to a specific Google Drive folder, naming it with product name and scene reference.  
  - Config: Folder ID specified; file name includes product name and scene reference; input data is the downloaded image; uses Google Drive OAuth2 credentials.  
  - Inputs: Image binary data from "HTTP Request"  
  - Outputs: To "Update row in sheet"  
  - Edge Cases: Upload failures, quota limits.  

- **Update row in sheet**  
  - Type: Google Sheets  
  - Role: Updates the appended row in "ad_image" sheet with status "Complete" and the Google Drive webViewLink URL of the uploaded image.  
  - Config: Uses "scene_ref" as matching column; updates status and output_url; uses Google Sheets OAuth2 credentials.  
  - Inputs: Uploaded file metadata from "Upload file" and appended row info  
  - Outputs: None (end of flow)  
  - Edge Cases: Sheet update conflicts, missing row, permission issues.  

---

### 3. Summary Table

| Node Name                     | Node Type                                      | Functional Role                                | Input Node(s)               | Output Node(s)                     | Sticky Note                                               |
|-------------------------------|-----------------------------------------------|-----------------------------------------------|-----------------------------|----------------------------------|-----------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                                | Workflow start trigger                         |                             | Get row(s) in sheet               |                                                           |
| Get row(s) in sheet            | Google Sheets                                 | Retrieve product rows with status "Create"    | When clicking ‘Execute workflow’ | Download file                     |                                                           |
| Download file                 | Google Drive                                  | Download product image from Drive URL          | Get row(s) in sheet          | Analyze image                    |                                                           |
| Analyze image                 | OpenAI Image Analysis (LangChain)             | Generate structured product image insights    | Download file                | AI Agent                        |                                                           |
| AI Agent                     | LangChain AI Agent                            | Generate multiple ad image prompts from data | Analyze image                | Split Out                      |                                                           |
| OpenAI Chat Model             | LangChain Chat OpenAI                         | Provide GPT-4.1-mini model for AI Agent       |                             | AI Agent (ai_languageModel)     |                                                           |
| Structured Output Parser      | LangChain Structured Output Parser            | Parse AI Agent JSON output                      | AI Agent                    | AI Agent (ai_outputParser)       |                                                           |
| Split Out                    | Split Out                                    | Split prompt array into individual scenes      | AI Agent                    | Append row in sheet              |                                                           |
| Append row in sheet           | Google Sheets                                 | Append prompts as new rows with status "Ready"| Split Out                   | Edit Fields                    |                                                           |
| Edit Fields                  | Set                                           | Extract and construct image URL for API       | Append row in sheet          | Call Fal.ai API (nannoBanana)   |                                                           |
| Call Fal.ai API (nannoBanana) | HTTP Request                                 | Submit prompt + image to Fal.ai for generation| Edit Fields                 | Get image status                |                                                           |
| Get image status             | HTTP Request                                  | Poll Fal.ai for image generation status        | Call Fal.ai API (nannoBanana), Wait | If                         |                                                           |
| If                          | If                                            | Check if Fal.ai image generation completed     | Get image status             | Get the image1 (true), Wait (false) |                                                           |
| Wait                        | Wait                                           | Delay before polling again                      | If (false branch)            | Get image status               |                                                           |
| Get the image1              | HTTP Request                                  | Retrieve generated image URL from Fal.ai       | If (true branch)             | HTTP Request                   |                                                           |
| HTTP Request                | HTTP Request                                  | Download generated image                        | Get the image1               | Upload file                   |                                                           |
| Upload file                 | Google Drive                                  | Upload generated image to Drive                 | HTTP Request                 | Update row in sheet            |                                                           |
| Update row in sheet         | Google Sheets                                 | Update sheet row with status and image URL     | Upload file                  |                              |                                                           |
| Sticky Note (disabled)       | Sticky Note                                   | Workflow block description                      |                             |                              | Multiple sticky notes document zones and product image previews |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add Google Sheets Node to Get Rows**  
   - Name: "Get row(s) in sheet"  
   - Operation: Read rows from sheet "product" (gid=0)  
   - Filter rows where `status` equals "Create".  
   - Set Google Sheets OAuth2 credentials.  
   - Connect output of manual trigger to this node.

3. **Add Google Drive Node to Download File**  
   - Name: "Download file"  
   - Operation: Download file by file ID.  
   - File ID extracted dynamically from `product_image_url` field using expressions (see Edit Fields node logic).  
   - Use Google Drive OAuth2 credentials.  
   - Connect from "Get row(s) in sheet".

4. **Add OpenAI Image Analysis Node**  
   - Name: "Analyze image"  
   - Resource: Image  
   - Operation: Analyze  
   - Model: GPT-4o-mini  
   - Input Type: Base64 (pass file data from previous node)  
   - Text prompt: Use the detailed multi-point instruction for product image analysis.  
   - Use OpenAI credentials.  
   - Connect from "Download file".

5. **Add LangChain AI Agent Node**  
   - Name: "AI Agent"  
   - Text: Complex prompt combining product fields and analysis results to generate N image prompts in JSON.  
   - System message: Detailed instructions for advertising image prompt generation with schema.  
   - Use OpenAI account credentials.  
   - Connect from "Analyze image".

6. **Create OpenAI Chat Model Node**  
   - Name: "OpenAI Chat Model"  
   - Model: GPT-4.1-mini  
   - Connect ai_languageModel output to AI Agent input.

7. **Add Structured Output Parser Node**  
   - Name: "Structured Output Parser"  
   - JSON schema example matching expected AI output.  
   - Connect ai_outputParser output to AI Agent node.

8. **Add Split Out Node**  
   - Name: "Split Out"  
   - Field to split: `output.scenes` (array of prompts)  
   - Connect from AI Agent main output.

9. **Add Google Sheets Node to Append Row**  
   - Name: "Append row in sheet"  
   - Operation: Append to sheet "ad_image" (gid=1544343606)  
   - Columns: prompt, status (set "Ready"), scene_ref, product_name  
   - Use Google Sheets OAuth2 credentials.  
   - Connect from "Split Out".

10. **Add Set Node to Edit Fields**  
    - Name: "Edit Fields"  
    - Add assignment: Extract Google Drive file ID from `product_image_url` and build direct view link `https://drive.google.com/uc?export=view&id=FILEID`.  
    - Include all other fields unchanged.  
    - Connect from "Append row in sheet".

11. **Add HTTP Request Node to Call Fal.ai API**  
    - Name: "Call Fal.ai API (nannoBanana)"  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/{{ model_target }}/edit` where model_target is from sheet data.  
    - Body (JSON): prompt + notes + image_urls (array with product image URL), num_images=1, output_format=jpeg  
    - Authentication: HTTP Header Auth with Fal AI credentials  
    - Connect from "Edit Fields".

12. **Add HTTP Request Node to Get Image Status**  
    - Name: "Get image status"  
    - URL: Use status_url from previous Fal.ai response  
    - Authentication: HTTP Header Auth with Fal AI credentials  
    - Connect from "Call Fal.ai API" and also from "Wait" node below.

13. **Add If Node**  
    - Name: "If"  
    - Condition: `$json.status == "COMPLETED"`  
    - Connect from "Get image status".  
    - True branch → "Get the image1" node  
    - False branch → "Wait" node

14. **Add Wait Node**  
    - Name: "Wait"  
    - Duration: 10 seconds  
    - Connect from "If" false branch.  
    - Output back to "Get image status" to poll again.

15. **Add HTTP Request Node to Get the Image**  
    - Name: "Get the image1"  
    - URL: `https://queue.fal.run/fal-ai/nano-banana/requests/{{ request_id }}`  
    - Authentication: HTTP Header Auth with Fal AI credentials  
    - Retry on failure enabled  
    - Connect from "If" true branch.

16. **Add HTTP Request Node to Download Generated Image**  
    - Name: "HTTP Request"  
    - URL: Use image URL from "Get the image1" response  
    - Authentication: HTTP Header Auth with Fal AI credentials  
    - Connect from "Get the image1".

17. **Add Google Drive Node to Upload File**  
    - Name: "Upload file"  
    - Folder ID: Google Drive folder for output images  
    - File name: `{{ product_name }}_{{ scene_ref }}.jpeg`  
    - Input data field: binary data from previous node  
    - Use Google Drive OAuth2 credentials  
    - Connect from "HTTP Request".

18. **Add Google Sheets Node to Update Row**  
    - Name: "Update row in sheet"  
    - Operation: Update row in "ad_image" sheet matching on `scene_ref`  
    - Update fields: set `status` to "Complete", `output_url` to Google Drive file webViewLink  
    - Use Google Sheets OAuth2 credentials  
    - Connect from "Upload file".

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                                     |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Sticky notes in the workflow visually document zones: Zone 1 (Create Image - AI prompt generation), Zone 2 (Create Image - Fal.ai generation). | Visible in node canvas, disabled sticky notes for workflow explanation and product image previews.                  |
| Product images are hosted on Google Drive with direct URL extraction logic in “Edit Fields” node.          | See expressions extracting file IDs from various URL formats for Google Drive.                                      |
| Fal.ai API (nano banana model) requires HTTP header authentication; credentials must be set accordingly.   | Credential named "Fal AI" configured for HTTP header authentication.                                               |
| OpenAI API usage involves multiple models: GPT-4o-mini for image analysis, GPT-4.1-mini for prompt generation. | Requires OpenAI API key with access to these models.                                                               |
| Campaign context supports seasonal/promotional themes like Halloween, Christmas, sales events (8.8, 10.10). | Used by AI Agent to incorporate subtle thematic elements in prompts.                                               |
| Google Sheets document ID: `1XAKLXfhRqPB7RjA8neUHYGwF4FKpM38edJUONHxt9pE` used for product and ad_image sheets.| Access and permission to these sheets must be granted for the workflow to function.                                 |

---

This detailed documentation enables understanding, reproduction, and modification of the workflow by advanced users or automation agents without referring to the original JSON.