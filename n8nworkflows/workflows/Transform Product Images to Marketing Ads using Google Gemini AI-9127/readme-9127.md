Transform Product Images to Marketing Ads using Google Gemini AI

https://n8nworkflows.xyz/workflows/transform-product-images-to-marketing-ads-using-google-gemini-ai-9127


# Transform Product Images to Marketing Ads using Google Gemini AI

---

### 1. Workflow Overview

This workflow automates the transformation of simple product photos into professional marketing images using Google Gemini AI via OpenRouter. It targets e-commerce sellers, social media marketers, small business owners, and content creators who want high-quality marketing visuals without costly photoshoots or graphic design.

**Logical Blocks:**

- **1.1 Input Reception:** Handles user upload of product images through a web form.
- **1.2 Image Preparation:** Converts uploaded binary image data into a base64-encoded format suitable for AI input.
- **1.3 AI Processing:** Sends the prepared image and a detailed prompt to Google Gemini AI to generate a professional marketing image.
- **1.4 Image Extraction & Conversion:** Extracts the image URL from AI response, converts it into a downloadable binary file.
- **1.5 Output Delivery:** Provides the user with the generated marketing image ready for download.

Supporting documentation and instructional sticky notes guide users through upload, AI generation, and download steps.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the user's product photo upload via a web form trigger, ensuring the file type and requirements are met.

- **Nodes Involved:**  
  - Upload Product Image (FormTrigger)  
  - Upload Instructions (Sticky Note)

- **Node Details:**  

  - **Upload Product Image**  
    - Type: Form Trigger  
    - Role: Entry point for user input; accepts a single JPG or PNG product image file.  
    - Configuration:  
      - Form title: “Product Photo to Marketing Image”  
      - Field: Single file upload, accepted types JPG and PNG, required field.  
    - Inputs: External HTTP trigger via webhook  
    - Outputs: Binary image data for downstream processing  
    - Failure Modes: Invalid file type or missing file; webhook errors  

  - **Upload Instructions**  
    - Type: Sticky Note  
    - Role: Provides user guidance on ideal image characteristics for upload  
    - Content: Tips on lighting, resolution, background simplicity  
    - No inputs or outputs  

#### 2.2 Image Preparation

- **Overview:**  
  Converts the uploaded binary image into base64 encoding and formats it as a data URL for AI consumption.

- **Nodes Involved:**  
  - Convert Image to Base64 (ExtractFromFile)  
  - Format Image URL for AI (Code)  
  - AI Generation Info (Sticky Note)

- **Node Details:**  

  - **Convert Image to Base64**  
    - Type: ExtractFromFile  
    - Role: Converts binary image to base64 string stored in JSON property  
    - Configuration: Operation “binaryToProperty” on binary property named “Image”  
    - Inputs: Binary image from form trigger  
    - Outputs: JSON with base64 string  
    - Failure Modes: Corrupted binary data, missing binary property  

  - **Format Image URL for AI**  
    - Type: Code (JavaScript)  
    - Role: Wraps base64 string into a data URL (`data:image/png;base64,...`) for AI input  
    - Key Expression: Constructs `url` property with base64 prefix  
    - Inputs: JSON with base64 string  
    - Outputs: JSON with `url` string property  
    - Failure Modes: Null or malformed base64 strings causing invalid data URL  

  - **AI Generation Info**  
    - Type: Sticky Note  
    - Role: Explains AI processing step and credential requirements  
    - No inputs or outputs  

#### 2.3 AI Processing

- **Overview:**  
  Sends a prompt along with the formatted image URL to Google Gemini 2.5 Flash model via OpenRouter API to generate a professional marketing image.

- **Nodes Involved:**  
  - AI Marketing Image Generator (HTTP Request)

- **Node Details:**  

  - **AI Marketing Image Generator**  
    - Type: HTTP Request  
    - Role: Sends chat completion request with image URL and detailed prompt to AI model  
    - Configuration:  
      - Method: POST  
      - URL: `https://openrouter.ai/api/v1/chat/completions`  
      - Model: `google/gemini-2.5-flash-image-preview`  
      - Body: JSON including user prompt with instructions for professional marketing image generation and the data URL of the product image  
      - Authentication: Uses predefined OpenRouter API credentials  
    - Inputs: JSON with `url` property containing base64 data URL  
    - Outputs: JSON response from AI containing generated image URL and content  
    - Failure Modes: Network errors, API authentication errors, rate limits, malformed requests, API downtime  
    - Version Requirements: Compatible with n8n versions supporting HTTP Request v4.2 and OpenRouter API authentication  

#### 2.4 Image Extraction & Conversion

- **Overview:**  
  Extracts the generated marketing image URL from the AI response JSON, splits MIME type and base64 data, and converts it into a downloadable binary file.

- **Nodes Involved:**  
  - Extract Image Data (Set)  
  - Convert to Downloadable File (ConvertToFile)  
  - Download Info (Sticky Note)

- **Node Details:**  

  - **Extract Image Data**  
    - Type: Set  
    - Role: Parses AI response JSON to extract:  
      - Full image URL string  
      - Base64 portion of the URL  
      - MIME type from the data URL  
      - Content text from AI response  
      - Filename with `.png` extension  
    - Key Expressions: Uses JSON path and string split operations to parse data URL  
    - Inputs: AI response JSON  
    - Outputs: JSON properties with extracted data for next node  
    - Failure Modes: Missing or malformed AI response fields, missing image URL, unexpected data URL format  

  - **Convert to Downloadable File**  
    - Type: ConvertToFile  
    - Role: Converts extracted base64 string into binary file for download  
    - Configuration: Source property is `base` (the base64 image data)  
    - Inputs: JSON with base64 image data  
    - Outputs: Binary file data ready for client download  
    - Failure Modes: Invalid base64 string, conversion errors  

  - **Download Info**  
    - Type: Sticky Note  
    - Role: Provides end-user information about the image download process and file readiness  
    - No inputs or outputs  

#### 2.5 Output Delivery

- **Overview:**  
  Presents the generated marketing image to the user via a form node that responds with a downloadable file and a custom completion message.

- **Nodes Involved:**  
  - Download Marketing Image (Form)

- **Node Details:**  

  - **Download Marketing Image**  
    - Type: Form  
    - Role: Final node to deliver AI-generated image as a downloadable file to the user  
    - Configuration:  
      - Operation: Completion  
      - RespondWith: Return binary file  
      - Completion Title: “Your Marketing Image is Ready!”  
      - Completion Message: Guidance on using the image in ads, social media, or product listings  
    - Inputs: Binary file data from previous conversion node  
    - Outputs: HTTP response with downloadable file  
    - Failure Modes: Webhook response errors, file transmission errors  

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                         |
|-------------------------|-------------------|----------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Upload Product Image     | Form Trigger      | User uploads product photo             | —                           | Convert Image to Base64     |                                                                                                                     |
| Upload Instructions     | Sticky Note       | User guidance on upload requirements   | —                           | —                          | 📤 Step 1: Upload Your Product Photo - tips on image quality and format                                             |
| Convert Image to Base64  | ExtractFromFile   | Convert image binary to base64 string  | Upload Product Image         | Format Image URL for AI     |                                                                                                                     |
| Format Image URL for AI  | Code              | Format base64 as data URL for AI input | Convert Image to Base64      | AI Marketing Image Generator |                                                                                                                     |
| AI Generation Info       | Sticky Note       | Information about AI step and creds    | —                           | —                          | 🤖 Step 2: AI Generation - uses Google Gemini 2.5 Flash via OpenRouter                                              |
| AI Marketing Image Generator | HTTP Request | Send prompt and image to AI for generation | Format Image URL for AI      | Extract Image Data          |                                                                                                                     |
| Extract Image Data       | Set               | Parse AI response for image URL & data | AI Marketing Image Generator | Convert to Downloadable File |                                                                                                                     |
| Convert to Downloadable File | ConvertToFile | Convert base64 string to binary file   | Extract Image Data           | Download Marketing Image    |                                                                                                                     |
| Download Info           | Sticky Note       | Info about download process             | —                           | —                          | 💾 Step 3: Process & Download - generated image ready for marketing use                                            |
| Download Marketing Image | Form              | Deliver downloadable marketing image   | Convert to Downloadable File | —                          | Your AI-generated marketing image will download automatically; use in ads, social media, or product listings        |
| Workflow Documentation  | Sticky Note       | Workflow purpose, usage, and setup     | —                           | —                          | ## Transform Product Photos into Marketing Images with AI - detailed description, setup, tips, and customization    |
| Setup Guide             | Sticky Note       | Quick setup instructions                | —                           | —                          | ## 🔧 Quick Setup Guide - prerequisites, configuration steps, customization examples                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a New Workflow** in n8n named “Transform Product Photos into Marketing Images with AI”.

2. **Create the Input Reception Block:**

   - Add a **Form Trigger** node:
     - Name: `Upload Product Image`
     - Configure with webhook to accept HTTP POST requests.
     - Set form title: “Product Photo to Marketing Image”
     - Add a single file upload field:
       - Label: “Product Image”
       - Accept types: jpg, png
       - Required: true

   - Add a **Sticky Note** node:
     - Name: `Upload Instructions`
     - Content:  
       ```
       ## 📤 Step 1: Upload Your Product Photo

       Upload a clear photo of your product (JPG or PNG format). The AI works best with:
       - Well-lit product images
       - Clear product visibility
       - Simple backgrounds
       - High resolution (recommended)
       ```
     - Position near the form trigger for visibility.

3. **Create the Image Preparation Block:**

   - Add an **ExtractFromFile** node:
     - Name: `Convert Image to Base64`
     - Operation: `binaryToProperty`
     - Binary Property Name: `Image`
     - Connect `Upload Product Image` → `Convert Image to Base64`

   - Add a **Code** node:
     - Name: `Format Image URL for AI`
     - Language: JavaScript
     - Code:
       ```javascript
       const items = $input.all();
       const updatedItems = items.map((item) => {
         const base64Url = item?.json?.data;
         const url = `data:image/png;base64,${base64Url}`;
         return { url };
       });
       return updatedItems;
       ```
     - Connect `Convert Image to Base64` → `Format Image URL for AI`

   - Add a **Sticky Note** node:
     - Name: `AI Generation Info`
     - Content:  
       ```
       ## 🤖 Step 2: AI Generation

       The AI analyzes your product and creates a professional marketing scene with models, backgrounds, and professional composition.

       **Note**: This uses Google Gemini 2.5 Flash via OpenRouter. Make sure your API credentials are configured.
       ```
     - Position near AI node.

4. **Create the AI Processing Block:**

   - Add an **HTTP Request** node:
     - Name: `AI Marketing Image Generator`
     - HTTP Method: POST
     - URL: `https://openrouter.ai/api/v1/chat/completions`
     - Authentication: Set to use OpenRouter API credentials (create credentials with your OpenRouter API key).
     - Body Content-Type: JSON
     - Body JSON (use expression to embed image url):
       ```json
       {
         "model": "google/gemini-2.5-flash-image-preview",
         "messages": [
           {
             "role": "user",
             "content": [
               {
                 "type": "text",
                 "text": "Transform this product image into a professional marketing photograph. Create a scene that includes: 1) A professional model naturally showcasing or using the product, 2) An aesthetically pleasing background that complements the product's style and target market, 3) Professional lighting and composition, 4) Strategic placement highlighting the product's key features, 5) A lifestyle context that helps potential customers envision using the product. The final image should look like a high-end advertisement suitable for e-commerce, social media, or print campaigns."
               },
               {
                 "type": "image_url",
                 "image_url": {
                   "url": "={{ $json.url }}"
                 }
               }
             ]
           }
         ]
       }
       ```
     - Connect `Format Image URL for AI` → `AI Marketing Image Generator`

5. **Create the Image Extraction & Conversion Block:**

   - Add a **Set** node:
     - Name: `Extract Image Data`
     - Assign fields:
       - `data`: `={{ $json.choices[0].message.images[0].image_url.url }}`
       - `base`: `={{ $json.choices[0].message.images[0].image_url.url.split(",")[1] }}`
       - `content`: `={{ $json.choices[0].message.content }}`
       - `mime`: `={{ $json.choices[0].message.images[0].image_url.url.split(",")[0].split(":")[1] }}`
       - `fileNme`: `.png`
     - Connect `AI Marketing Image Generator` → `Extract Image Data`

   - Add a **ConvertToFile** node:
     - Name: `Convert to Downloadable File`
     - Operation: `toBinary`
     - Binary Property Source: `base`
     - Connect `Extract Image Data` → `Convert to Downloadable File`

   - Add a **Sticky Note** node:
     - Name: `Download Info`
     - Content:  
       ```
       ## 💾 Step 3: Process & Download

       The generated image is processed and prepared for download. You'll receive a high-quality PNG file ready for use in your marketing materials.
       ```
     - Position near conversion nodes.

6. **Create the Output Delivery Block:**

   - Add a **Form** node:
     - Name: `Download Marketing Image`
     - Operation: `completion`
     - Respond With: `returnBinary`
     - Completion Title: “Your Marketing Image is Ready!”
     - Completion Message: “Your AI-generated marketing image will automatically download to your device. Use it in your ads, social media, or product listings!”
     - Connect `Convert to Downloadable File` → `Download Marketing Image`

7. **Add General Workflow Documentation Sticky Note:**

   - Add a **Sticky Note** node:
     - Name: `Workflow Documentation`
     - Content:  
       ```
       ## Transform Product Photos into Marketing Images with AI

       **Made by [Biznova](https://www.biznova.tech/en)** | **[TikTok](https://www.tiktok.com/@biznova_tech)**

       ---

       ### 🎯 Who's it for
       E-commerce sellers, social media marketers, small business owners, and content creators who need professional product advertising images without expensive photoshoots or graphic designers.

       ### ✨ What it does
       This workflow automatically transforms simple product photos into polished, professional marketing images featuring:
       - Professional models showcasing your product
       - Aesthetically pleasing, contextual backgrounds
       - Professional lighting and composition
       - Lifestyle scenes that help customers envision using the product
       - Commercial-ready quality suitable for ads and e-commerce

       ### 🚀 How it works
       1. Upload your basic product photo via the web form
       2. AI analyzes your product and generates a complete marketing scene
       3. Download your professional marketing image automatically
       4. Use it immediately in ads, social media, or product listings

       ### ⚙️ Setup Requirements
       1. **OpenRouter Account**: Create a free account at [openrouter.ai](https://openrouter.ai)
       2. **API Key**: Generate your API key from the OpenRouter dashboard
       3. **Add Credentials**: Configure the OpenRouter API credentials in the "AI Marketing Image Generator" node
       4. **Test**: Upload a sample product image to test the workflow

       ### 🎨 How to customize
       - **Edit the prompt** in the "AI Marketing Image Generator" node to match your brand style
       - **Adjust file formats** in the upload form (currently accepts JPG/PNG)
       - **Modify the response message** in the final form node
       - **Add your branding** by including brand colors or style preferences in the prompt

       ### 💡 Pro Tips
       - Use high-resolution product images for best results
       - Test different prompt variations to find your ideal style
       - Save successful prompts for consistent brand imagery
       - Batch process multiple products by running the workflow multiple times
       ```
     - Position clearly visible in the editor.

8. **Create OpenRouter API Credential:**

   - Within n8n credentials manager, create new credential type for OpenRouter API:
     - Provide your API key from [openrouter.ai](https://openrouter.ai)
     - Name the credential (e.g., `new_key`)
     - Assign this credential to the HTTP Request node

9. **Activate the Workflow** and test by uploading a product image.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Made by [Biznova](https://www.biznova.tech/en) with detailed documentation and setup instructions                   | Workflow Documentation sticky note                                                                   |
| OpenRouter AI platform used for Google Gemini AI access; sign up required at [openrouter.ai](https://openrouter.ai)  | Setup Guide sticky note and AI Generation Info sticky note                                           |
| Workflow supports JPG and PNG image formats for upload                                                                | Upload Product Image node configuration                                                             |
| Tips for best image quality: well-lit, simple backgrounds, high resolution                                            | Upload Instructions sticky note                                                                      |
| Customization options include prompt editing, file type adjustments, and branding additions                           | Workflow Documentation and Setup Guide sticky notes                                                 |
| Google Gemini 2.5 Flash Image Preview is the AI model used for generating marketing images                            | AI Marketing Image Generator node configuration                                                      |
| Output marketing images are high-quality PNG files suitable for e-commerce ads, social media, and print campaigns     | Download Info and Download Marketing Image nodes                                                    |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.

---