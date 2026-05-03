Create AI-Enhanced Facebook Posts from Amazon Affiliate Links with Gemini

https://n8nworkflows.xyz/workflows/create-ai-enhanced-facebook-posts-from-amazon-affiliate-links-with-gemini-7422


# Create AI-Enhanced Facebook Posts from Amazon Affiliate Links with Gemini

---

### 1. Workflow Overview

This workflow automates the creation and posting of AI-enhanced Facebook posts based on Amazon affiliate product links. It is designed for Amazon affiliate marketers who want to streamline generating engaging Facebook captions and professional product images, then automatically post them to Facebook while tracking progress in a Google Sheet.

**Target Use Cases:**  
- Amazon affiliate marketers seeking automated content creation and social media posting  
- Social media managers automating product promotion on Facebook  
- Marketers integrating AI-generated captions and images into their workflow  

**Logical Blocks:**  
- **1.1 Input Reception:** Google Sheets Trigger to receive new Amazon product links  
- **1.2 ASIN Extraction:** AI extracts ASIN from provided Amazon URLs  
- **1.3 Product Data Retrieval:** Fetch Amazon product details via RapidAPI  
- **1.4 AI Content Generation:** Generate Facebook caption and image prompt using OpenRouter AI models  
- **1.5 Image Processing:** Download product image, upload to Google Gemini API, generate enhanced image, convert for upload  
- **1.6 Facebook Posting:** Upload final image with caption to Facebook via Facebook Graph API  
- **1.7 Update Tracking:** Mark the product post as “Done ✅” in Google Sheets  
- **1.8 Workflow Control:** Includes wait nodes to manage timing and flow  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Listens for new rows added to a specific Google Sheet, triggering the workflow with Amazon product URLs.

**Nodes Involved:**  
- Google Sheets Trigger

**Node Details:**  
- **Google Sheets Trigger**  
  - Type: Trigger node for Google Sheets events  
  - Configured to trigger on “rowAdded” event in the sheet named “Sheet1” (gid=0) of document ID `1unoIMG4dKLP1Fw0euo64deo_FyBJp4viN4_Sy4LT0Dc`  
  - Inputs: None (triggered externally)  
  - Outputs: Triggers downstream ASIN extraction  
  - Edge Cases: Trigger misfires if sheet permissions/connection fail or if column “Product Link” is missing  
  - Credential: Requires Google Sheets OAuth2 credentials  

---

#### 2.2 ASIN Extraction

**Overview:**  
Extracts the Amazon ASIN number from the product URL using AI.

**Nodes Involved:**  
- OpenRouter Chat Model2  
- asin number

**Node Details:**  
- **OpenRouter Chat Model2**  
  - Type: AI language model (OpenRouter Chat)  
  - Model: `google/gemini-2.0-flash-exp:free`  
  - Role: Provides AI inference capability for downstream nodes  
- **asin number**  
  - Type: Langchain AI Agent node  
  - Configuration: Uses prompt to extract ASIN number from value in `Product Link` field from Google Sheets Trigger  
  - Expression: `You only have to give the asin number from this amazon product url: {{ $json['Product Link'] }}`  
  - Inputs: Product Link from Google Sheets Trigger  
  - Outputs: ASIN string for next step  
  - Edge Cases: URL invalid or malformed will cause incorrect or no ASIN extraction  
  - Version: Requires Langchain integration and OpenRouter credentials  

---

#### 2.3 Product Data Retrieval

**Overview:**  
Fetch detailed product information from Amazon using ASIN via RapidAPI.

**Nodes Involved:**  
- Amazon Product Details

**Node Details:**  
- **Amazon Product Details**  
  - Type: HTTP Request node  
  - Method: GET  
  - URL: `https://real-time-amazon-data.p.rapidapi.com/product-details?asin={{ $json.output }}&country=US`  
  - Headers: Includes `x-rapidapi-host` and `x-rapidapi-key` (user must replace `YOUR_API_KEY` with valid RapidAPI key)  
  - Input: ASIN from asin number node  
  - Output: Product details JSON including title, photos, about_product, product_url  
  - Edge Cases: API key invalid, rate limits, ASIN not found, network failures  
  - Version: HTTP Request v4.2  

---

#### 2.4 AI Content Generation

**Overview:**  
Generate engaging Facebook post captions and AI image generation prompts based on product details.

**Nodes Involved:**  
- OpenRouter Chat Model1  
- FB caption  
- OpenRouter Chat Model  
- Image Prompt Generate

**Node Details:**  
- **OpenRouter Chat Model1**  
  - Type: AI language model node  
  - Model: `google/gemini-2.0-flash-exp:free`  
  - Role: Provides AI inference for caption generation node  
- **FB caption**  
  - Type: Langchain AI Agent  
  - Configuration: Prompt to generate a short, casual, high-converting Facebook caption with CTA and affiliate link; includes emojis, no symbols, 1-3 lines max  
  - Uses product title, product features (3rd item in about_product array), and affiliate link from Amazon Product Details  
  - Input: Product details JSON  
  - Output: Caption text for Facebook post  
  - Edge Cases: AI may generate irrelevant or off-topic captions if product data incomplete  
- **OpenRouter Chat Model**  
  - AI language model node supporting image prompt generation  
- **Image Prompt Generate**  
  - Langchain AI Agent  
  - Prompt generates a short, realistic image generation prompt focusing on product photography style with human model, lighting, angle, cinematic grain  
  - Input: Product title and first about_product description  
  - Output: Text prompt for image generation  
  - Edge Cases: Misinterpretation of product type could cause poor image prompts  

---

#### 2.5 Image Processing

**Overview:**  
Download product image, upload to Google Gemini API, generate enhanced image, and convert it for Facebook upload.

**Nodes Involved:**  
- Product Image  
- Image Upload To Server  
- ai image generator  
- Convert to File

**Node Details:**  
- **Product Image**  
  - HTTP Request node  
  - Downloads product photo URL from Amazon Product Details (`data.product_photo`)  
  - Outputs binary image data  
  - Edge Cases: Image URL invalid or inaccessible  
- **Image Upload To Server**  
  - HTTP Request node  
  - Uploads binary image to Google Gemini API for processing  
  - Uses headers `X-Goog-Upload-Command`, `X-Goog-Upload-Header-Content-Length`, `Content-Type` set to `image/png`  
  - Input: Binary image data from Product Image node  
  - Output: File URI and metadata from Google Gemini  
  - Edge Cases: API key invalid, upload failure, content type mismatch  
- **ai image generator**  
  - HTTP Request node  
  - Sends a generation request to Google Gemini model `gemini-2.0-flash-exp` with uploaded file URI and image prompt text from Image Prompt Generate node  
  - Configured with temperature, topK, topP, max tokens, and response modalities for Text and Image  
  - Output: AI-generated image+text content in JSON  
  - Edge Cases: API request failure, rate limiting, invalid prompt  
- **Convert to File**  
  - ConvertToFile node  
  - Converts AI-generated inline image data into binary format suitable for Facebook upload  
  - Input: JSON property with base64 image data  
  - Output: Binary file data for Facebook Graph API upload  

---

#### 2.6 Facebook Posting

**Overview:**  
Uploads the AI-generated image with the AI-generated caption to Facebook.

**Nodes Involved:**  
- Wait  
- Facebook Graph API

**Node Details:**  
- **Wait**  
  - Wait node pauses workflow for 10 seconds before posting (likely to manage API rate limits or processing delays)  
- **Facebook Graph API**  
  - Type: Facebook Graph API node  
  - Operation: POST photo to edge `photos` on node `me` (current user/page)  
  - Sends binary image data (`data` binary property) along with `message` parameter containing Facebook caption from FB caption node  
  - Graph API version: v22.0  
  - Inputs: Binary image from Convert to File, caption from FB caption  
  - Outputs: Facebook post response JSON  
  - Edge Cases: Facebook permission/auth failures, rate limiting, binary upload errors  
  - Credential: Requires Facebook OAuth2 credentials with publish permissions  

---

#### 2.7 Update Tracking

**Overview:**  
After successful Facebook upload, marks the Google Sheet row as completed for tracking.

**Nodes Involved:**  
- Google Sheets

**Node Details:**  
- **Google Sheets**  
  - Operation: Update  
  - Updates “Facebook Upload” column to value “Done ✅” for the processed product link row  
  - Matching column: “Product Link”  
  - Inputs: Product Link from Google Sheets Trigger, confirmation from Facebook Graph API  
  - Outputs: Updated row data  
  - Edge Cases: Google Sheets permission issues or mismatch in product link causing failure to update  

---

#### 2.8 Workflow Control & Metadata

**Overview:**  
Includes sticky notes and additional OpenRouter Chat Model nodes for AI infrastructure.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note5

**Node Details:**  
- Sticky notes contain project purpose, usage instructions, setup guide, and a YouTube tutorial link, aiding user understanding and onboarding.  
- OpenRouter Chat Model nodes (multiple instances) provide AI model infrastructure for different AI agent nodes.  

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                              | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                              |
|-----------------------|----------------------------------|----------------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger  | Google Sheets Trigger             | Entry point: watches for new product links   | None                       | asin number                |                                                                                                        |
| asin number           | Langchain AI Agent                | Extract ASIN from Amazon URL                  | Google Sheets Trigger       | Amazon Product Details     |                                                                                                        |
| Amazon Product Details | HTTP Request                     | Fetch product details from RapidAPI           | asin number                 | FB caption                 |                                                                                                        |
| FB caption            | Langchain AI Agent                | Generate Facebook caption for product         | Amazon Product Details      | Product Image              |                                                                                                        |
| Product Image         | HTTP Request                     | Download product image                         | FB caption                  | Image Upload To Server     |                                                                                                        |
| Image Upload To Server | HTTP Request                     | Upload image to Google Gemini API              | Product Image               | Image Prompt Generate      |                                                                                                        |
| Image Prompt Generate | Langchain AI Agent                | Generate AI image prompt text                   | Image Upload To Server      | ai image generator         |                                                                                                        |
| ai image generator    | HTTP Request                     | Generate enhanced product image from prompt    | Image Prompt Generate       | Convert to File            |                                                                                                        |
| Convert to File       | ConvertToFile                    | Convert AI image data to binary                 | ai image generator          | Wait                      |                                                                                                        |
| Wait                  | Wait                             | Wait 10 seconds before posting                  | Convert to File             | Facebook Graph API         |                                                                                                        |
| Facebook Graph API    | Facebook Graph API                | Upload photo + caption to Facebook              | Wait                       | Google Sheets              |                                                                                                        |
| Google Sheets         | Google Sheets                    | Mark product as posted in sheet                 | Facebook Graph API          | None                      |                                                                                                        |
| OpenRouter Chat Model2 | AI Language Model (OpenRouter)  | AI model for ASIN extraction                     | None                       | asin number                |                                                                                                        |
| OpenRouter Chat Model1 | AI Language Model (OpenRouter)  | AI model for Facebook caption generation        | None                       | FB caption                 |                                                                                                        |
| OpenRouter Chat Model  | AI Language Model (OpenRouter)  | AI model for image prompt generation             | None                       | Image Prompt Generate      |                                                                                                        |
| Sticky Note           | Sticky Note                     | Workflow purpose and overview                   | None                       | None                      | --- Purpose of This Agent: Automates product detail fetching, AI caption & image generation, Facebook upload, and Google Sheets tracking. |
| Sticky Note1          | Sticky Note                     | Instructions on workflow usage                   | None                       | None                      | --- How to Use: Add link to sheet, auto ASIN extraction, data fetch, AI caption & image gen, Facebook upload, mark done.             |
| Sticky Note2          | Sticky Note                     | Setup guide with API keys and credential setup  | None                       | None                      | --- Setup Guide: Google Sheets setup, RapidAPI key, OpenRouter credentials, Gemini API key, Facebook Graph API credentials.          |
| Sticky Note5          | Sticky Note                     | YouTube tutorial link                            | None                       | None                      | --- Start here: Step-by-Step Youtube Tutorial: https://youtu.be/7Gz4C0XbBK8?si=RtW-ATYRF07C3o7_                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up Google Sheets Trigger:**  
   - Create a Google Sheet with columns “Product Link” and “Facebook Upload”.  
   - Add a Google Sheets Trigger node in n8n, set to watch for “rowAdded” events on your sheet (document ID and sheet name/gid).  
   - Connect your Google Sheets OAuth2 credentials.

2. **Add AI Node for ASIN Extraction:**  
   - Add OpenRouter Chat Model node configured with model `google/gemini-2.0-flash-exp:free` for ASIN extraction.  
   - Add Langchain AI Agent node with prompt: “You only have to give the asin number from this amazon product url: {{ $json['Product Link'] }}”.  
   - Connect Google Sheets Trigger output to Langchain AI Agent input.

3. **Fetch Amazon Product Details:**  
   - Add HTTP Request node named “Amazon Product Details”.  
   - Configure GET request to: `https://real-time-amazon-data.p.rapidapi.com/product-details?asin={{ $json.output }}&country=US`.  
   - Add headers: `x-rapidapi-host: real-time-amazon-data.p.rapidapi.com`, `x-rapidapi-key: YOUR_API_KEY`, `country: US`.  
   - Connect ASIN extraction output to this node’s input.

4. **Generate Facebook Caption (AI):**  
   - Add OpenRouter Chat Model node for caption generation.  
   - Add Langchain AI Agent node configured with prompt to generate a short, engaging Facebook caption with product title, features, and affiliate link.  
   - Connect Amazon Product Details node to this AI Agent.

5. **Download Product Image:**  
   - Add HTTP Request node “Product Image”.  
   - Set URL to `{{ $('Amazon Product Details').item.json.data.product_photo }}`.  
   - Connect from FB caption node (to ensure caption generated before image download).

6. **Upload Image to Google Gemini:**  
   - Add HTTP Request node “Image Upload To Server”.  
   - Configure POST to `https://generativelanguage.googleapis.com/upload/v1beta/files?key=YOUR_API_KEY`.  
   - Set headers for upload command and content-type (image/png).  
   - Use binary data input from Product Image node.

7. **Generate AI Image Prompt:**  
   - Add OpenRouter Chat Model node for image prompt generation.  
   - Add Langchain AI Agent node with prompt to create a short, realistic image generation prompt including human model instructions.  
   - Connect Image Upload To Server node output to this AI Agent.

8. **Generate Enhanced Image via Gemini:**  
   - Add HTTP Request node “ai image generator”.  
   - POST to `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent?key=YOUR_API_KEY`.  
   - Body includes file URI from previous upload, prompt text from image prompt node, and generation config (temperature, topK, topP, max tokens, response modalities Text & Image).  
   - Connect output of Image Prompt Generate node.

9. **Convert AI Image to Binary:**  
   - Add ConvertToFile node “Convert to File”.  
   - Convert the AI-generated inline image data (base64) to binary for Facebook upload.  
   - Connect from ai image generator node.

10. **Add Wait Node:**  
    - Add Wait node set to 10 seconds to allow processing delays.

11. **Post to Facebook:**  
    - Add Facebook Graph API node.  
    - Configure POST method on edge `photos` for node `me`.  
    - Send binary image data in property `data`.  
    - Add query parameter `message` with caption text from FB caption node.  
    - Connect Wait node output to Facebook node input.  
    - Connect Facebook OAuth2 credentials with publish permissions.

12. **Update Google Sheet Status:**  
    - Add Google Sheets node to update “Facebook Upload” column to “Done ✅” for the processed product link.  
    - Configure matching column to “Product Link”.  
    - Connect Facebook Graph API node output.

13. **Add Sticky Notes:**  
    - Create sticky notes with purpose, instructions, setup guide, and tutorial link for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Setup requires API keys for RapidAPI (Amazon data), OpenRouter (AI caption and prompt), Google Gemini (image upload and generation).       | https://rapidapi.com/, https://aistudio.google.com/apikey                                       |
| Facebook OAuth2 credentials must have permission to post photos to user/page timeline.                                                      |                                                                                                 |
| Workflow automatically triggers on new Google Sheet rows with product links and updates status upon completion.                            |                                                                                                 |
| YouTube tutorial for step-by-step setup and usage: [Amazon Affiliate Marketing Automation](https://youtu.be/7Gz4C0XbBK8?si=RtW-ATYRF07C3o7_) |                                                                                                 |
| Workflow includes best practices for AI prompts to ensure engaging captions and realistic product photography AI generation.               |                                                                                                 |

---

**Disclaimer:** The provided text originates exclusively from an n8n automated workflow. All data processed are legal and public, respecting content policies and containing no illegal, offensive, or protected material.

---