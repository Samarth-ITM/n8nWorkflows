Build an Image Restoration Service with n8n & Gemini AI Image Editing

https://n8nworkflows.xyz/workflows/build-an-image-restoration-service-with-n8n---gemini-ai-image-editing-4815


# Build an Image Restoration Service with n8n & Gemini AI Image Editing

### 1. Workflow Overview

This workflow implements an automated vintage image restoration service leveraging n8n and Google’s Gemini AI Image Editing capabilities. It processes vintage photographs with damage such as cracks, tears, and missing sections, and restores them to near-original condition by removing imperfections and filling missing parts while preserving original colors and dimensions.

The workflow is logically divided into these blocks:

- **1.1 Input Image Preparation:** Downloads sample vintage images from predefined URLs and splits the array into individual image items for processing.
- **1.2 Image Data Extraction:** Converts downloaded images into base64 encoded strings suitable for AI input.
- **1.3 AI-Based Image Restoration:** Sends the extracted image data along with a prompt to Google’s Gemini Image Generation API to restore the images.
- **1.4 Post-Processing and Upload:** Converts AI output back into binary files and uploads the restored images to Google Drive.
- **1.5 Documentation and Notes:** Embedded sticky notes provide context, usage instructions, and visual examples of original vs. restored images.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Image Preparation

- **Overview:** Initializes the workflow with a manual trigger, sets URLs for sample vintage images, and splits them into individual processing items.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Sample Images (Set)  
  - Split Out (Split Out)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually.  
    - Config: No parameters, simple manual start.  
    - Input: None  
    - Output: Triggers next node (Sample Images)  
    - Failures: None expected, manual trigger.  

  - **Sample Images**  
    - Type: Set  
    - Role: Defines an array of 3 URLs pointing to vintage images hosted on Cloudinary.  
    - Config: Assigns an array property `url` with 3 image URLs.  
    - Input: Trigger data from manual trigger.  
    - Output: Passes array to Split Out node.  
    - Notes: URLs represent damaged vintage photographs for restoration demonstration.  
    - Failures: None expected.  

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array of URLs into individual items for parallel processing.  
    - Config: Splits on field `url`.  
    - Input: Receives array of URLs.  
    - Output: Outputs one item per URL to Download Image node.  
    - Failures: None expected.  

---

#### 2.2 Image Data Extraction

- **Overview:** Downloads each image from its URL and converts the binary data into a base64-encoded string suitable for AI processing.  
- **Nodes Involved:**  
  - Download Image (HTTP Request)  
  - Extract from File (Extract From File)  

- **Node Details:**

  - **Download Image**  
    - Type: HTTP Request  
    - Role: Downloads image data from the current URL.  
    - Config: URL dynamically set from incoming item’s `url` property using expression `={{ $json.url }}`; no additional headers or auth.  
    - Input: Receives single URL item from Split Out node.  
    - Output: Binary image data in `data` property.  
    - Failures: Possible network errors, invalid URLs, or 404s.  
    - Notes: Downloads raw image for conversion.  

  - **Extract from File**  
    - Type: Extract From File  
    - Role: Converts binary file data into a base64 string placed in JSON for API compatibility.  
    - Config: Operation set to `binaryToPropery` (binary data → JSON property).  
    - Input: Binary image data from Download Image node.  
    - Output: JSON with base64 string of image in `data` property.  
    - Failures: Could fail if binary data is corrupt or missing.  

---

#### 2.3 AI-Based Image Restoration

- **Overview:** Uses Google Gemini’s Image Generation API to restore damaged images by sending the base64 image and a restoration prompt, receiving back a restored image in base64 format.  
- **Nodes Involved:**  
  - Gemini Image Restoration (HTTP Request)  
  - Get Image Contents (Set)  

- **Node Details:**

  - **Gemini Image Restoration**  
    - Type: HTTP Request  
    - Role: Calls Google Gemini API to generate restored images from original damaged inputs.  
    - Config:  
      - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-preview-image-generation:generateContent`  
      - Method: POST  
      - Body: JSON including prompt instructing restoration ("You are a vintage image restoration agent...") and base64 image data extracted from previous node.  
      - Authentication: Uses predefined Google Palm API credentials.  
      - Response Modalities: Requests both TEXT and IMAGE outputs.  
    - Input: JSON with base64 image data and MIME type.  
    - Output: JSON containing candidates with restored image base64 data and MIME type.  
    - Failures:  
      - Authentication errors if credentials invalid.  
      - API rate limits or quota exceeded.  
      - Geo-restrictions causing model unavailability.  
      - Timeout or malformed request errors.  
    - Notes: The prompt can be customized for better restoration quality.  

  - **Get Image Contents**  
    - Type: Set  
    - Role: Extracts restored image base64 string and mime type from Gemini API response.  
    - Config: Uses expressions to locate first candidate's inline data in response JSON.  
    - Input: Gemini API response JSON.  
    - Output: Properties `image` (base64 string) and `mimeType`.  
    - Failures: Could fail if API response format changes or no image data present.  

---

#### 2.4 Post-Processing and Upload

- **Overview:** Converts the AI-generated base64 restored image back into binary format and uploads it to Google Drive.  
- **Nodes Involved:**  
  - Convert to File (Convert To File)  
  - Upload to Drive (Google Drive)  

- **Node Details:**

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts base64-encoded string back to binary format for file upload.  
    - Config:  
      - Operation: `toBinary`  
      - Source Property: `image` (base64 string)  
      - MIME Type: dynamically set from the `mimeType` property extracted earlier.  
    - Input: JSON with base64 image data.  
    - Output: Binary file data ready for upload.  
    - Failures: Conversion errors if base64 is malformed or missing.  
    - On Error: Configured to continue workflow even if conversion fails.  

  - **Upload to Drive**  
    - Type: Google Drive  
    - Role: Uploads restored image binary to Google Drive folder.  
    - Config:  
      - File name: dynamically named `file_restored_<index>.<fileExtension>`  
      - Drive: My Drive  
      - Folder: Root folder (modifiable)  
      - Credentials: OAuth2 Google Drive account configured.  
    - Input: Binary file from Convert to File node.  
    - Output: Upload confirmation with file metadata.  
    - Failures: Authentication errors, permission denied, quota limits.  

---

#### 2.5 Documentation and Notes

- **Overview:** Provides detailed instructions, usage notes, examples with before/after images, and warnings about API costs and geo-restrictions via sticky notes.  
- **Nodes Involved:**  
  - Sticky Note (multiple instances)  

- **Node Details:**  
  - Sticky notes contain markdown content linking to official docs, pricing, usage tips, and sample image comparisons.  
  - They do not affect workflow execution but serve as embedded documentation.  
  - Notable notes include:  
    - Explanation of HTTP Request usage for image downloads.  
    - Details on Gemini’s text and image editing power.  
    - Instructions on converting base64 to binary for Google Drive upload.  
    - Pricing and geo-restriction warnings.  
    - Visual tables comparing original vs. restored images with embedded image links.  

---

### 3. Summary Table

| Node Name                   | Node Type        | Functional Role                              | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                                     |
|-----------------------------|------------------|----------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger    | Manual start of workflow                      | None                         | Sample Images               |                                                                                                                |
| Sample Images               | Set              | Defines sample image URLs                     | When clicking ‘Execute workflow’ | Split Out                  | Sticky Note1: ## 1. Download Sample Images with HTTP Request doc link and integration tips                     |
| Split Out                  | Split Out        | Splits array of URLs into individual items   | Sample Images                 | Download Image              | Sticky Note1 (same as above)                                                                                   |
| Download Image             | HTTP Request     | Downloads images from URLs                     | Split Out                    | Extract from File           | Sticky Note1 (same as above)                                                                                   |
| Extract from File          | Extract From File | Converts binary image data to base64 string  | Download Image               | Gemini Image Restoration    | Sticky Note1 (same as above)                                                                                   |
| Gemini Image Restoration   | HTTP Request     | Sends image and prompt to Gemini API          | Extract from File            | Get Image Contents          | Sticky Note2: ## 2. Use Gemini LLM for image restoration with link to Gemini docs                              |
| Get Image Contents         | Set              | Extracts base64 restored image and MIME type | Gemini Image Restoration     | Convert to File             | Sticky Note2 (same as above)                                                                                   |
| Convert to File            | Convert To File  | Converts base64 restored image to binary      | Get Image Contents           | Upload to Drive             | Sticky Note3: ## 3. Upload to Google Drive with Google Drive node docs link                                    |
| Upload to Drive            | Google Drive     | Uploads restored binary image to Google Drive | Convert to File              | None                       | Sticky Note3 (same as above)                                                                                   |
| Sticky Note                | Sticky Note      | Displays original vs restored image comparison | None                        | None                       | Table of example images (original and restored)                                                                |
| Sticky Note1               | Sticky Note      | Introduction and HTTP Request explanation     | None                        | None                       | Contains usage info and integration suggestions                                                                |
| Sticky Note2               | Sticky Note      | Gemini API usage explanation                   | None                        | None                       | Details on Gemini image editing capabilities                                                                    |
| Sticky Note3               | Sticky Note      | Google Drive upload instructions               | None                        | None                       | Explains base64 to binary conversion and uploading process                                                     |
| Sticky Note4               | Sticky Note      | Workflow overview and cost warnings            | None                        | None                       | Full workflow explanation, pricing note, and usage tips                                                        |
| Sticky Note5               | Sticky Note      | Original vs restored image comparison          | None                        | None                       | Table with second sample image before/after                                                                    |
| Sticky Note6               | Sticky Note      | Original vs restored image comparison          | None                        | None                       | Table with third sample image before/after                                                                     |
| Sticky Note8               | Sticky Note      | Geo-restrictions warning                        | None                        | None                       | Advises on possible model availability issues based on region                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name it `When clicking ‘Execute workflow’`  
   - No parameters needed.  

2. **Add a Set node named `Sample Images`**  
   - Add a field `url` of type Array.  
   - Assign array values with 3 sample vintage image URLs:  
     ```
     [
       "https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/l8b3j1sf6ejx73z0awsh",
       "https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/htrjbmiozdrvxwdsreyt",
       "https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/ykftgxpozp2pr4sxpuoy"
     ]
     ```  
   - Connect the Manual Trigger output to this node.  

3. **Add a Split Out node named `Split Out`**  
   - Set `Field to Split Out` to `url`.  
   - Connect `Sample Images` output to this node.  

4. **Add HTTP Request node named `Download Image`**  
   - Set Method to `GET`.  
   - Set URL to expression: `={{ $json.url }}` to download each image URL.  
   - Connect `Split Out` output.  

5. **Add Extract From File node named `Extract from File`**  
   - Set Operation to `binaryToProperty` (convert binary data to base64 string in JSON).  
   - Connect `Download Image` output.  

6. **Add HTTP Request node named `Gemini Image Restoration`**  
   - Set Method to `POST`.  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-preview-image-generation:generateContent`  
   - Set Body Content Type to JSON.  
   - Use the following JSON body (adjust indentation as needed):  
     ```json
     {
       "contents": [
         {
           "parts": [
             {
               "text": "You are an vintage image restoration agent. Restore the given image by removing cracks and tears and fill in missing sections of the image. Retain the dimensions, colors and camera film type of the original image."
             },
             {
               "inline_data": {
                 "mime_type": "{{ $('Download Image').item.binary.data.mimeType }}",
                 "data": "{{ $json.data }}"
               }
             }
           ]
         }
       ],
       "generationConfig": {
         "responseModalities": ["TEXT", "IMAGE"]
       }
     }
     ```  
   - Enable authentication with Google Palm API credentials (requires previously set up OAuth2 credentials for Google Gemini API).  
   - Connect `Extract from File` output.  

7. **Add a Set node named `Get Image Contents`**  
   - Add two fields:  
     - `image` (string) with expression: `={{ $json.candidates[0].content.parts.find(part => part.inlineData).inlineData.data }}`  
     - `mimeType` (string) with expression: `={{ $json.candidates[0].content.parts.find(part => part.inlineData).inlineData.mimeType }}`  
   - Connect `Gemini Image Restoration` output.  

8. **Add Convert To File node named `Convert to File`**  
   - Operation: `toBinary`  
   - Source Property: `image` (the base64 string)  
   - MIME Type: set expression `={{ $json.mimeType }}`  
   - On error: set to continue workflow (optional).  
   - Connect `Get Image Contents` output.  

9. **Add Google Drive node named `Upload to Drive`**  
   - Operation: Upload file  
   - File Name: `file_restored_{{ $itemIndex }}.{{ $binary.data.fileExtension }}`  
   - Drive ID: My Drive (default or choose specific)  
   - Folder ID: Root or specify folder where to save files  
   - Use Google Drive OAuth2 credentials authorized for your Google account.  
   - Connect `Convert to File` output.  

10. **Optionally add Sticky Note nodes** at relevant positions with content for documentation and usage examples, referencing official docs and including markdown tables with before/after images.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow demonstrates a vintage image restoration service using Google Gemini’s multimodal AI capabilities for image editing, highlighting the new text-and-image-to-image generation feature.                                          | Gemini Image Editing API Docs: https://ai.google.dev/gemini-api/docs/image-generation#gemini-image-editing |
| Restored images are output as base64 strings; converting them back to binary is necessary for file upload to Google Drive or other storage solutions.                                                                                     | Google Drive Node Docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive     |
| Please be aware that as of the time of writing, each image generation with Gemini costs approximately $0.039 USD. Check official pricing for updates.                                                                                    | Gemini Pricing Info: https://ai.google.com/pricing                                                           |
| Geo-restrictions may apply; the Gemini Image Generation model is currently available only in select countries. Model-not-found errors may indicate regional unavailability.                                                                 |                                                                                                         |
| For community support, join the n8n Discord or visit the forum.                                                                                                                                                                            | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                           |
| HTTP Request node documentation is crucial for understanding image download implementation.                                                                                                                                               | HTTP Request Docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.