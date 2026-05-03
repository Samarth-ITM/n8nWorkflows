Generate Professional product images | AI marketing campaign Generator

https://n8nworkflows.xyz/workflows/generate-professional-product-images---ai-marketing-campaign-generator-6285


# Generate Professional product images | AI marketing campaign Generator

---

### 1. Workflow Overview

This workflow titled **"Generate Professional product images | AI marketing campaign Generator"** automates the creation of a comprehensive AI-driven marketing campaign for a product, focusing on generating professional product images tailored for multiple digital marketing assets.

**Purpose:**  
To enable users to upload a product image along with product details and automatically generate a tailored marketing campaign including visual assets such as Instagram posts, stories, website banners, ad creatives, and testimonial graphics. The workflow combines AI-based visual and textual analysis, image editing via AI, and asset management through Google Drive.

**Target Use Cases:**  
- E-commerce brands and marketers wanting automated, AI-enhanced product marketing content.  
- Marketing teams aiming to streamline campaign asset creation based on product image and descriptions.  
- Agencies or freelancers creating scalable marketing visuals across multiple platforms.

**Logical Blocks:**  
1.1 Input Reception and Initial Data Preparation  
1.2 AI-driven Product and Marketing Campaign Analysis  
1.3 Asset Generation via AI Image Editing  
1.4 Asset Distribution and Storage  
1.5 Error Handling and Data Validation

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Initial Data Preparation

**Overview:**  
This block collects input from a user-submitted form including a product image and product-related textual data. It validates and processes the input, extracting the binary image data and product details to prepare a dynamic brand profile for AI processing.

**Nodes Involved:**  
- On form submission  
- Google Drive (initial upload)  
- Edit Fields

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point collecting product image and details via a form titled "🚀 AI Marketing Campaign Generator V2".  
  - Configuration:  
    - Fields include product image (file, required), product name (required), tagline, description (textarea, required), category (select, required), highlighted benefit (textarea).  
  - Input: User form submission via webhook  
  - Output: JSON form data + binary image data  
  - Edge cases: Missing required fields, file upload failures, webhook timeouts.

- **Google Drive (initial upload)**  
  - Type: Google Drive node  
  - Role: Uploads the raw product image to a specified Google Drive folder ("Product Images") for storage/backup.  
  - Configuration: Uses Google Drive OAuth2 credentials; folderId points to dedicated folder.  
  - Input: Binary image data from form submission  
  - Output: Confirmation of file upload  
  - Edge cases: Authentication failure, quota issues, file format or size restrictions.

- **Edit Fields**  
  - Type: Code (JS) node  
  - Role: Processes the form data and binary image input to prepare a structured dynamic brand profile JSON object. Converts the image to base64 for AI processing, validates presence of image data.  
  - Key logic:  
    - Extracts all form fields safely.  
    - Attempts multiple keys to locate binary image data; throws error if none found.  
    - Converts binary image to base64 string.  
    - Constructs a dynamic brand profile object including metadata and processing flags.  
  - Input: JSON + binary from form submission  
  - Output: JSON dynamic brand profile + binary image data under key `data`  
  - Edge cases: Missing image binary, improper binary format, buffer conversion errors.

---

#### 1.2 AI-driven Product and Marketing Campaign Analysis

**Overview:**  
This block performs advanced AI analysis combining the uploaded product image and textual data to generate a comprehensive marketing campaign strategy and creative directions for multiple asset types.

**Nodes Involved:**  
- Image & Text Analyzer (Langchain AI Agent)  
- Structured Output Parser1  
- Code (post AI processing)

**Node Details:**

- **Image & Text Analyzer**  
  - Type: Langchain AI Agent  
  - Role: Executes visual and textual AI analysis to derive product insights, brand strategy, campaign strategy, and detailed asset creative directions.  
  - Configuration:  
    - Receives prompt combining image + product text.  
    - Instructed to produce structured JSON with exactly 5 assets: Instagram Post, Instagram Story, Website Banner, Ad Creative, Testimonial Graphic.  
    - Includes instructions to analyze visual colors, styles, and branding from image data.  
    - Passthrough binary images enabled for AI visual analysis.  
  - Input: Dynamic brand profile + image binary from Edit Fields  
  - Output: Structured JSON marketing campaign plan with assets  
  - Edge cases: AI model timeouts, parsing errors, incomplete AI output, malformed JSON.

- **Structured Output Parser1**  
  - Type: Langchain Structured Output Parser  
  - Role: Validates and parses AI JSON output according to a strict schema defining productAnalysis, brandStrategy, campaignStrategy, and assets arrays with required properties.  
  - Configuration: Manual JSON schema enforcing types and required fields.  
  - Input: AI text output from Image & Text Analyzer  
  - Output: Parsed JSON structure for downstream nodes  
  - Edge cases: Parsing failures, schema mismatches, missing required fields.

- **Code (post AI processing)**  
  - Type: Code (JS) node  
  - Role: Processes the parsed AI output to extract and prepare individual marketing assets, attaches product info, and prepares data with correct binary for image editing.  
  - Logic includes:  
    - Extracting assets array, brand strategy, campaign strategy safely.  
    - Validates presence of assets; throws error if none.  
    - Prepares output items for each asset with metadata and binary product image for editing.  
  - Input: Parsed AI structured JSON + original form data + binary image  
  - Output: Array of asset objects, each ready for image generation  
  - Edge cases: Missing assets, binary data missing, runtime JS errors.

---

#### 1.3 Asset Generation via AI Image Editing

**Overview:**  
This block generates photorealistic marketing images for each asset type by sending the product image and creative directions to the OpenAI Image Edit API.

**Nodes Involved:**  
- Switch  
- Instagram_Post HTTP Request  
- Instagram_Story HTTP Request  
- Website_Banner HTTP Request  
- Ad_Creative HTTP Request  
- Testimonials HTTP Request  
- Convert to File, Convert to File1, Convert to File2, Convert to File3, Convert to File4

**Node Details:**

- **Switch**  
  - Type: Switch node  
  - Role: Routes each asset from the processed assets array to the appropriate HTTP request node based on `assetType`.  
  - Configuration: Cases for "Instagram Post", "Instagram Story", "Website Banner", "Ad Creative", "Testimonial Graphic"  
  - Input: Array of assets  
  - Output: One branch per asset type  
  - Edge cases: AssetType missing or unknown, no matching case.

- **Instagram_Post, Instagram_Story, Website_Banner, Ad_Creative, Testimonials**  
  - Type: HTTP Request node  
  - Role: Call OpenAI Image Edit API to generate photorealistic images per asset creative direction.  
  - Configuration commonalities:  
    - URL: https://api.openai.com/v1/images/edits  
    - Method: POST, multipart/form-data  
    - Auth: OpenAI API credential  
    - Body parameters:  
      - model: "gpt-image-1"  
      - image: binary from previous node ("image" field)  
      - prompt: dynamically constructed with product name, background tone, surface type, accent, lighting, camera angle, brand tone, overlay text  
      - size: auto  
    - Timeout set to 90 seconds (90000 ms)  
  - Input: Asset metadata + product image binary  
  - Output: JSON with base64 image data  
  - Edge cases: API rate limits, timeouts, invalid image formats, authentication errors.

- **Convert to File, Convert to File1, Convert to File2, Convert to File3, Convert to File4**  
  - Type: Convert to File node  
  - Role: Converts base64-encoded images from HTTP response to binary files for saving.  
  - Configuration: Source property "data[0].b64_json" (OpenAI image edit response format)  
  - Input: HTTP Request node output  
  - Output: Binary image files for storage  
  - Edge cases: Missing or malformed base64 data, conversion failures.

---

#### 1.4 Asset Distribution and Storage

**Overview:**  
This block uploads the generated asset images to designated Google Drive folders for organized storage.

**Nodes Involved:**  
- Google Drive2  
- Google Drive3  
- Google Drive4  
- Google Drive5  
- Google Drive6

**Node Details:**

- **Google Drive2, Google Drive3, Google Drive4, Google Drive5, Google Drive6**  
  - Type: Google Drive node  
  - Role: Uploads converted binary images to the "Product Images" folder in Google Drive. Each node corresponds to a specific asset type folder:  
    - Google Drive6: Instagram Post  
    - Google Drive2: Instagram Story  
    - Google Drive3: Website Banner  
    - Google Drive4: Ad Creative  
    - Google Drive5: Testimonials  
  - Configuration:  
    - Uses same Google Drive OAuth2 credentials as initial upload  
    - FolderId set to shared Product Images folder (could be subdivided per asset)  
  - Input: Binary image from convertToFile nodes  
  - Output: Upload confirmation  
  - Edge cases: Authentication issues, file overwrite conflicts, quota limits.

---

#### 1.5 Error Handling and Data Validation

**Overview:**  
Error handling is embedded primarily in the Code nodes and AI output parsing to ensure clean data flow and meaningful errors if inputs or AI outputs are missing or malformed.

**Nodes Involved:**  
- Edit Fields (validation of image binary)  
- Code (post AI processing) (validation of AI output and assets)  
- Structured Output Parser1 (schema validation)  
- Form Trigger (required fields)

**Edge Cases:**  
- Missing or corrupt product image upload  
- AI model failures or incomplete responses  
- Google Drive upload failures  
- HTTP Request timeouts or API errors  
- Expression errors in prompt construction

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                        | Input Node(s)        | Output Node(s)             | Sticky Note                         |
|---------------------|----------------------------------|-------------------------------------|----------------------|----------------------------|-----------------------------------|
| On form submission   | Form Trigger                     | User input form capture              | -                    | Google Drive, Edit Fields  | ## Product Image and Characteristics Form |
| Google Drive        | Google Drive                     | Upload raw product image             | On form submission   | -                          | ## Product Image and Characteristics Form |
| Edit Fields          | Code                            | Prepare brand profile and validate  | On form submission, Google Drive | Image & Text Analyzer     | ## Advanced AI marketing strategist |
| Image & Text Analyzer| Langchain AI Agent              | AI product & marketing analysis     | Edit Fields          | Code                       | ## Advanced AI marketing strategist |
| Structured Output Parser1 | Langchain Output Parser     | Validate & parse AI output JSON     | Image & Text Analyzer| Image & Text Analyzer       |                                   |
| Code                 | Code                            | Prepare assets array for generation | Image & Text Analyzer| Switch                     |                                   |
| Switch               | Switch                         | Route assets by type                 | Code                 | Instagram_Post, Instagram_Story, Website_Banner, Ad_Creative, Testimonials |                                   |
| Instagram_Post       | HTTP Request                   | Generate Instagram Post image       | Switch               | Convert to File             | ## Instagram Post Generation      |
| Instagram_Story      | HTTP Request                   | Generate Instagram Story image      | Switch               | Convert to File1            | ## Instagram Story Generation     |
| Website_Banner       | HTTP Request                   | Generate Website Banner image       | Switch               | Convert to File2            | ## Website Banner Generation      |
| Ad_Creative          | HTTP Request                   | Generate Ad Creative image           | Switch               | Convert to File3            | ## Ad Creative Generation         |
| Testimonials         | HTTP Request                   | Generate Testimonial Graphic image  | Switch               | Convert to File4            | ## Testimonials Generation        |
| Convert to File      | ConvertToFile                  | Convert base64 to binary (Post)     | Instagram_Post       | Google Drive6               | ## Instagram Post Generation      |
| Convert to File1     | ConvertToFile                  | Convert base64 to binary (Story)    | Instagram_Story      | Google Drive2               | ## Instagram Story Generation     |
| Convert to File2     | ConvertToFile                  | Convert base64 to binary (Banner)   | Website_Banner       | Google Drive3               | ## Website Banner Generation      |
| Convert to File3     | ConvertToFile                  | Convert base64 to binary (Ad)       | Ad_Creative          | Google Drive4               | ## Ad Creative Generation         |
| Convert to File4     | ConvertToFile                  | Convert base64 to binary (Testimonial)| Testimonials        | Google Drive5               | ## Testimonials Generation        |
| Google Drive2        | Google Drive                   | Upload Instagram Story image        | Convert to File1     | -                          |                                   |
| Google Drive3        | Google Drive                   | Upload Website Banner image         | Convert to File2     | -                          |                                   |
| Google Drive4        | Google Drive                   | Upload Ad Creative image            | Convert to File3     | -                          |                                   |
| Google Drive5        | Google Drive                   | Upload Testimonial image            | Convert to File4     | -                          |                                   |
| Google Drive6        | Google Drive                   | Upload Instagram Post image         | Convert to File      | -                          |                                   |
| Sticky Note          | Sticky Note                   | Annotation                         | -                    | -                          | ## Product Image and Characteristics Form |
| Sticky Note1         | Sticky Note                   | Annotation                         | -                    | -                          | ## Instagram Post Generation      |
| Sticky Note2         | Sticky Note                   | Annotation                         | -                    | -                          | ## Instagram Story Generation     |
| Sticky Note3         | Sticky Note                   | Annotation                         | -                    | -                          | ## Website Banner Generation      |
| Sticky Note4         | Sticky Note                   | Annotation                         | -                    | -                          | ## Ad Creative Generation         |
| Sticky Note5         | Sticky Note                   | Annotation                         | -                    | -                          | ## Testimonials Generation        |
| Sticky Note6         | Sticky Note                   | Annotation                         | -                    | -                          | ## Advanced AI marketing strategist |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "On form submission"**  
   - Type: Form Trigger  
   - Configure path (auto-generated or custom)  
   - Title: "🚀 AI Marketing Campaign Generator V2"  
   - Fields:  
     - Product Image (file, required)  
     - Product Name (text, required)  
     - Product Tagline (text)  
     - Product Description (textarea, required)  
     - Product Category (select, required)  
     - Highlighted Benefit (textarea)  
   - Save credentials and webhook.

2. **Add Google Drive Node: "Google Drive"**  
   - Type: Google Drive  
   - Credentials: Google Drive OAuth2  
   - Drive: "My Drive"  
   - Folder ID: Target product image folder  
   - Input Data Field Name: "Product_Image" (match form binary field)  
   - Connect output of "On form submission" to Google Drive node.

3. **Add Code Node: "Edit Fields"**  
   - Type: Code  
   - Paste JS code to extract form fields, locate binary image, convert to base64, build dynamic brand profile object including product info and flags.  
   - Input: Output of "On form submission"  
   - Output: JSON + binary with key 'data' for image binary  
   - Connect output of "On form submission" to "Edit Fields".

4. **Add Langchain AI Agent Node: "Image & Text Analyzer"**  
   - Use Langchain AI Agent node supporting image input  
   - Set prompt to analyze both visual and text data (paste the detailed prompt), instructing to produce structured JSON with product analysis, brand strategy, campaign strategy, and 5 marketing assets.  
   - Enable passthroughBinaryImages  
   - Connect output of "Edit Fields" to this node.

5. **Add Langchain Structured Output Parser Node: "Structured Output Parser1"**  
   - Configure with manual JSON schema matching expected AI output (productAnalysis, brandStrategy, campaignStrategy, assets array with specified properties).  
   - Connect output of "Image & Text Analyzer" to this parser node.

6. **Add Code Node: "Code" for AI Output Processing**  
   - Paste JS code to safely extract assets, product info, and prepare output array with each asset including metadata and binary product image.  
   - Connect output of "Structured Output Parser1" to this node.

7. **Add Switch Node: "Switch"**  
   - Configure rules to check `$json.assetType` for each of the 5 asset types:  
     - Instagram Post  
     - Instagram Story  
     - Website Banner  
     - Ad Creative  
     - Testimonial Graphic  
   - Connect output of "Code" node to "Switch".

8. **For each asset type, add HTTP Request Node (OpenAI Image Edit API):**  
   - Nodes: Instagram_Post, Instagram_Story, Website_Banner, Ad_Creative, Testimonials  
   - Method: POST  
   - URL: https://api.openai.com/v1/images/edits  
   - Authentication: OpenAI API credentials  
   - Content-Type: multipart/form-data  
   - Body parameters:  
     - model: "gpt-image-1"  
     - image: from input binary (`image` field)  
     - prompt: dynamically constructed using expressions referencing asset metadata fields (product name, backgroundTone, surfaceType, accentProp, lighting, cameraAngle, brand tone, overlay text)  
     - size: auto  
   - Timeout: 90000 ms  
   - Connect corresponding output branch of "Switch" node to each HTTP Request node.

9. **For each HTTP Request node, add Convert to File Node:**  
   - Operation: toBinary  
   - Source property: "data[0].b64_json" (OpenAI image response)  
   - Connect output of each HTTP Request node to its Convert to File node.

10. **For each Convert to File node, add Google Drive node to upload the generated image:**  
    - Use same Google Drive OAuth2 credentials  
    - Drive: "My Drive"  
    - Folder ID: Same "Product Images" folder or subfolders as desired  
    - Connect Convert to File outputs to respective Google Drive nodes:  
      - Convert to File -> Google Drive6 (Instagram Post)  
      - Convert to File1 -> Google Drive2 (Instagram Story)  
      - Convert to File2 -> Google Drive3 (Website Banner)  
      - Convert to File3 -> Google Drive4 (Ad Creative)  
      - Convert to File4 -> Google Drive5 (Testimonials)

11. **Add Sticky Notes for documentation at appropriate positions** (optional but recommended for clarity).

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The OpenAI Image Edit API uses the `"gpt-image-1"` model to generate photorealistic images with editing. | Official docs: https://platform.openai.com/docs/api-reference/images/create-edit                  |
| Google Drive OAuth2 credentials must be set up with appropriate scopes for file upload and listing.      | Google Drive API docs: https://developers.google.com/drive/api/v3/about-sdk                       |
| The Langchain AI Agent node is configured to analyze both image and text inputs for a comprehensive output. | Langchain for n8n: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/           |
| The workflow expects exact asset types for routing; adding new asset types requires updating the Switch. | Important for workflow scalability and maintenance.                                              |
| Handling large image files could cause timeouts; consider adjusting HTTP request timeout or image sizes. | Timeout currently set to 90000 ms (90 seconds) for OpenAI image edits.                            |
| The dynamic prompt construction uses n8n expression syntax (e.g., `{{ $json.propertyName }}`) to customize AI calls. | See n8n expressions: https://docs.n8n.io/nodes/expressions/                                      |

---

**Disclaimer:**  
This document is generated based exclusively on an n8n workflow JSON export. All content respects legal and ethical guidelines. The workflow handles only legal and public data.

---