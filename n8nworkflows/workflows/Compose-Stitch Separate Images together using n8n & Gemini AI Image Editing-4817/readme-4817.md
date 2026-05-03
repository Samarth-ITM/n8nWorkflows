Compose/Stitch Separate Images together using n8n & Gemini AI Image Editing

https://n8nworkflows.xyz/workflows/compose-stitch-separate-images-together-using-n8n---gemini-ai-image-editing-4817


# Compose/Stitch Separate Images together using n8n & Gemini AI Image Editing

### 1. Workflow Overview

This n8n workflow demonstrates how to compose or "stitch" separate images together using Google’s Gemini AI Image Editing model. The workflow targets users who want to generate new imagery by combining multiple existing images with AI assistance, maintaining the original image styles and assets. Typical use cases include creating consistent storyboard scenes, marketing materials with product assets, or fashion try-ons.

The workflow is logically divided into the following blocks:

- **1.1 Input Acquisition:** Fetch source images via HTTP requests.
- **1.2 Image Preparation:** Convert downloaded images from binary to base64 strings for AI consumption.
- **1.3 AI Image Composition:** Use Gemini Image Generation API to produce a composed image from the base64 inputs and textual prompt.
- **1.4 Result Processing and Upload:** Convert Gemini’s base64 output back to binary file and upload it to Google Drive.
- **1.5 Annotations and Documentation:** Sticky notes provide instructional content and visual references throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Acquisition

**Overview:**  
This block fetches three separate images from cloud storage URLs via HTTP GET requests. These images serve as the source assets to be composed by Gemini AI.

**Nodes Involved:**  
- Character (HTTP Request)  
- Dalmation (HTTP Request)  
- Sofa (HTTP Request)

**Node Details:**

- **Character**  
  - Type: HTTP Request  
  - Role: Download the character image asset.  
  - Configuration: Simple GET request to a Cloudinary URL delivering an image in an optimized format (`f_auto,q_auto`).  
  - Inputs: Manual Trigger node initiates this request.  
  - Outputs: Binary image data.  
  - Potential failures: HTTP request failure (network issues, 404), format incompatibility.

- **Dalmation**  
  - Type: HTTP Request  
  - Role: Download the dalmation image asset.  
  - Configuration: Similar GET request to Cloudinary with image optimizations.  
  - Inputs: Manual trigger.  
  - Outputs: Binary image data.  
  - Potential failures: Same as Character node.

- **Sofa**  
  - Type: HTTP Request  
  - Role: Download the sofa image asset.  
  - Configuration: HTTP GET request to Cloudinary URL.  
  - Inputs: Manual trigger.  
  - Outputs: Binary image data.  
  - Potential failures: Same as above.

---

#### 1.2 Image Preparation

**Overview:**  
Converts the downloaded binary images to base64 strings and aggregates them into one dataset, preparing the data structure required by Gemini’s API.

**Nodes Involved:**  
- Merge1 (Merge)  
- Extract from File (ExtractFromFile)  
- Aggregate1 (Aggregate)  
- Sticky Note6 (documentation)

**Node Details:**

- **Merge1**  
  - Type: Merge  
  - Role: Combines the three separate image streams into one array of items, setting `numberInputs` to 3.  
  - Inputs: Character, Dalmation, Sofa nodes.  
  - Outputs: Aggregated stream of three image items.  
  - Edge cases: Mismatch in input counts could cause unexpected output.

- **Extract from File**  
  - Type: ExtractFromFile  
  - Role: Converts binary image data from each input item into base64 string properties for easier handling.  
  - Configuration: `Operation` set to `binaryToProperty`, extracting binary data into JSON properties.  
  - Inputs: Output of Merge1.  
  - Outputs: JSON with base64 data.  
  - Failure modes: Binary data missing or corrupt binary could cause extraction failure.

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates all items into a single item containing an array of the base64 image data objects.  
  - Configuration: Uses `aggregateAllItemData`.  
  - Inputs: Output of Extract from File.  
  - Outputs: Single aggregated JSON item with an array of base64 images.  
  - Edge cases: Empty input results in empty aggregation.

- **Sticky Note6**  
  - Type: Sticky Note  
  - Role: Documentation about converting images to base64 strings with a link to node docs.

---

#### 1.3 AI Image Composition

**Overview:**  
This block sends the aggregated base64 images and a text prompt instructing Gemini AI on how to compose the images into a new image output.

**Nodes Involved:**  
- Gemini Image Compose (HTTP Request)  
- Get Image Contents (Set)  
- Sticky Note2 (documentation)

**Node Details:**

- **Gemini Image Compose**  
  - Type: HTTP Request  
  - Role: Calls Google Gemini's text and image-to-image editing API to generate composed image content.  
  - Configuration:  
    - Method: POST  
    - URL: Gemini API endpoint for image generation.  
    - Body: JSON including:  
      - Text prompt describing the scene ("The character is sitting on the sofa...").  
      - Inline base64 image data for 3 images with MIME type `image/png`.  
      - Generation config requesting both TEXT and IMAGE in response.  
    - Authentication: Uses predefined Google PaLM API credentials.  
  - Inputs: Aggregated base64 images from Aggregate1.  
  - Outputs: JSON with generated image data as base64 string.  
  - Potential failures: Authentication errors, model unavailability (geo-restrictions), malformed request, rate limits, network timeouts.

- **Get Image Contents**  
  - Type: Set  
  - Role: Extracts the generated image base64 string and MIME type from Gemini’s response for further processing.  
  - Configuration: Extracts first candidate’s content inline data base64 string and MIME type.  
  - Inputs: Output of Gemini Image Compose.  
  - Outputs: JSON containing `image` (base64 string) and `mimeType`.  
  - Edge cases: No candidates in response or unexpected response structure leads to extraction errors.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Documentation explaining the power of Gemini’s text-and-image-to-image editing for composing images.

---

#### 1.4 Result Processing and Upload

**Overview:**  
Converts the base64 composed image from Gemini back into a binary file and uploads it to Google Drive.

**Nodes Involved:**  
- Convert to File (ConvertToFile)  
- Upload to Drive (Google Drive)  
- Sticky Note3 (documentation)  
- Sticky Note (Final Output image display)

**Node Details:**

- **Convert to File**  
  - Type: ConvertToFile  
  - Role: Converts base64 image string to binary file format.  
  - Configuration:  
    - Sets MIME type dynamically based on extracted mimeType from previous node.  
    - Operation: `toBinary` converting `image` property to binary data.  
  - Inputs: JSON from Get Image Contents.  
  - Outputs: Binary file data for upload.  
  - Edge cases: MIME type missing or incorrect causes conversion failure.

- **Upload to Drive**  
  - Type: Google Drive  
  - Role: Uploads the binary image file to Google Drive root folder.  
  - Configuration:  
    - File name: Pattern `file_restored_{{ $itemIndex }}.{{ $binary.data.fileExtension }}` dynamically naming the file.  
    - Folder: Root drive folder.  
    - Credentials: Uses OAuth2 Google Drive account.  
  - Inputs: Binary file from Convert to File.  
  - Outputs: Upload confirmation metadata.  
  - Potential failures: Credential expiration, quota limits, API errors, network issues.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Documentation about converting base64 to binary for upload and Google Drive node usage.

- **Sticky Note (Final Output)**  
  - Type: Sticky Note  
  - Role: Visual reference to the final composed image output.

---

#### 1.5 Annotations and Documentation

**Overview:**  
Sticky notes provide explanations, example images, usage instructions, and warnings about geo-restrictions for Gemini.

**Nodes Involved:**  
- Sticky Note1 (Character image preview)  
- Sticky Note4 (Dalmation image preview)  
- Sticky Note5 (Sofa image preview)  
- Sticky Note7 (General Usage Instructions)  
- Sticky Note8 (Geo Restrictions Warning)

**Node Details:**

- **Sticky Note1,4,5**  
  - Show the original source images for visual reference.

- **Sticky Note7**  
  - Provides a detailed explanation of workflow purpose, steps, requirements, and usage tips.  
  - Contains links to Discord and n8n Forum for help.

- **Sticky Note8**  
  - Warns about geographical restrictions on Gemini Image Generation API availability.

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                      |
|------------------------------|--------------------|----------------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Entry point to start workflow          |                               | Character, Dalmation, Sofa     |                                                                                                                 |
| Character                    | HTTP Request       | Download character image               | When clicking ‘Execute workflow’ | Merge1                        | Sticky Note1 - Character image preview                                                                          |
| Dalmation                   | HTTP Request       | Download dalmation image               | When clicking ‘Execute workflow’ | Merge1                        | Sticky Note4 - Dalmation image preview                                                                          |
| Sofa                         | HTTP Request       | Download sofa image                    | When clicking ‘Execute workflow’ | Merge1                        | Sticky Note5 - Sofa image preview                                                                                 |
| Merge1                       | Merge              | Combine 3 image streams into array     | Character, Dalmation, Sofa      | Extract from File              |                                                                                                                 |
| Extract from File            | ExtractFromFile    | Convert binary image to base64 string  | Merge1                        | Aggregate1                    | Sticky Note6 - Explains base64 conversion                                                                        |
| Aggregate1                   | Aggregate          | Aggregate all items into one array     | Extract from File              | Gemini Image Compose          |                                                                                                                 |
| Gemini Image Compose         | HTTP Request       | Call Gemini AI to compose images       | Aggregate1                   | Get Image Contents            | Sticky Note2 - Explains Gemini image editing                                                                     |
| Get Image Contents           | Set                | Extract base64 image and MIME type     | Gemini Image Compose          | Convert to File               |                                                                                                                 |
| Convert to File              | ConvertToFile      | Convert base64 string to binary file   | Get Image Contents            | Upload to Drive               | Sticky Note3 - Explains conversion to binary and upload                                                          |
| Upload to Drive              | Google Drive       | Upload final binary image to Google Drive | Convert to File              |                               |                                                                                                                 |
| Sticky Note1                 | Sticky Note        | Documentation: Character image preview |                               |                               |                                                                                                                 |
| Sticky Note2                 | Sticky Note        | Documentation: Gemini image compose    |                               |                               |                                                                                                                 |
| Sticky Note3                 | Sticky Note        | Documentation: Upload to Google Drive  |                               |                               |                                                                                                                 |
| Sticky Note4                 | Sticky Note        | Documentation: Dalmation image preview |                               |                               |                                                                                                                 |
| Sticky Note5                 | Sticky Note        | Documentation: Sofa image preview      |                               |                               |                                                                                                                 |
| Sticky Note6                 | Sticky Note        | Documentation: Base64 conversion       |                               |                               |                                                                                                                 |
| Sticky Note7                 | Sticky Note        | Documentation: Workflow overview, instructions |                               |                               | Contains detailed instructions, usage notes, links to Discord and Forum                                         |
| Sticky Note8                 | Sticky Note        | Warning: Gemini API geo restrictions   |                               |                               |                                                                                                                 |
| Sticky Note2                 | Sticky Note        | Explanation of Gemini’s text-and-image editing |                               |                               | Contains link: https://ai.google.dev/gemini-api/docs/image-generation#gemini-image-editing                      |
| Sticky Note3                 | Sticky Note        | Google Drive node documentation        |                               |                               | Contains link: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive                     |
| Sticky Note7                 | Sticky Note        | Help and usage links                    |                               |                               | Contains links: Discord https://discord.com/invite/XPKeKXeB7d, Forum https://community.n8n.io/                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Entry point to start the workflow.

2. **Add HTTP Request Nodes for Input Images**  
   - Create three HTTP Request nodes named `Character`, `Dalmation`, and `Sofa`.  
   - For each, set Method to GET and URL to the respective Cloudinary image URL:  
     - Character: `https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/qwitjyhejockuxfvkk5g`  
     - Dalmation: `https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/gxcc0xnghpbthwt8fpec`  
     - Sofa: `https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/nyghscauxtcfl4dve6of`  
   - Connect Manual Trigger node output to each of these three nodes.

3. **Add Merge Node**  
   - Type: Merge  
   - Name: `Merge1`  
   - Settings: Set `numberInputs` to 3 to combine the three image requests into one stream.  
   - Connect outputs of `Character`, `Dalmation`, and `Sofa` nodes to inputs 0,1,2 of Merge1 respectively.

4. **Add Extract From File Node**  
   - Type: ExtractFromFile  
   - Operation: `binaryToProperty` (converts binary data to base64 string property).  
   - Connect Merge1 output to this node.

5. **Add Aggregate Node**  
   - Type: Aggregate  
   - Aggregate: `aggregateAllItemData` (combine multiple items into one JSON array).  
   - Connect Extract from File output to Aggregate node.

6. **Add HTTP Request Node for Gemini Image Compose**  
   - Name: `Gemini Image Compose`  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-preview-image-generation:generateContent`  
   - Authentication: Use Google PaLM API credentials (set up OAuth2 or API key according to Google API).  
   - Body (JSON):  
   ```json
   {
     "contents": [
       {
         "parts": [
           {
             "text": "The character is sitting on the sofa, and the dalmation is lying on the blanket sleeping on the floor. Retain the image style of the character image."
           },
           {
             "inline_data": {
               "mime_type": "image/png",
               "data": "{{ $json.data[0].data }}"
             }
           },
           {
             "inline_data": {
               "mime_type": "image/png",
               "data": "{{ $json.data[1].data }}"
             }
           },
           {
             "inline_data": {
               "mime_type": "image/png",
               "data": "{{ $json.data[2].data }}"
             }
           }
         ]
       }
     ],
     "generationConfig": {
       "responseModalities": [
         "TEXT",
         "IMAGE"
       ]
     }
   }
   ```  
   - Connect Aggregate output to this node.

7. **Add Set Node to Extract Image Contents**  
   - Name: `Get Image Contents`  
   - Add two fields:  
     - `image` with expression: `={{ $json.candidates[0].content.parts.find(part => part.inlineData).inlineData.data }}`  
     - `mimeType` with expression: `={{ $json.candidates[0].content.parts.find(part => part.inlineData).inlineData.mimeType }}`  
   - Connect Gemini Image Compose output to this node.

8. **Add Convert To File Node**  
   - Operation: `toBinary`  
   - Source Property: `image`  
   - MIME Type: Set dynamically with expression `={{ $json.mimeType }}`  
   - Connect Get Image Contents output to this node.

9. **Add Google Drive Upload Node**  
   - Set credentials for Google Drive OAuth2.  
   - File Name: Use expression `file_restored_{{ $itemIndex }}.{{ $binary.data.fileExtension }}`  
   - Folder ID: Set to root or desired folder.  
   - Connect Convert To File output to this node.

10. **Add Sticky Notes** (optional but recommended for documentation)  
    - Add notes for:  
      - Workflow overview and instructions  
      - Per image preview (Character, Dalmation, Sofa)  
      - Explanation of base64 conversion  
      - Gemini image editing description with link: https://ai.google.dev/gemini-api/docs/image-generation#gemini-image-editing  
      - Google Drive node info with link: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive  
      - Geo restriction warning about Gemini API availability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Use cases include storyboarding, marketing materials, and fashion try-ons that require consistent image style and asset retention.                                                                                              | General workflow purpose                                                                           |
| Gemini Image Generation API supports text-and-image-to-image editing, enabling powerful composition instructions combining multiple images.                                                                                     | https://ai.google.dev/gemini-api/docs/image-generation#gemini-image-editing                        |
| Google Drive node is used for demonstration but any other file storage destination can replace it.                                                                                                                               | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive                      |
| Gemini Image Generation API may be geo-restricted; users outside supported regions may receive model not found errors.                                                                                                           | Sticky Note8 warning                                                                              |
| Community support and further help is available on the n8n Discord and Forum.                                                                                                                                                      | Discord: https://discord.com/invite/XPKeKXeB7d<br>Forum: https://community.n8n.io/                 |

---

*Disclaimer: The provided text is exclusively from an automated workflow built with n8n, respecting all current content policies. It contains no illegal, offensive, or protected material. All data manipulated is legal and public.*