Generate product descriptions from images and publish to Shopify using AI

https://n8nworkflows.xyz/workflows/generate-product-descriptions-from-images-and-publish-to-shopify-using-ai-11695


# Generate product descriptions from images and publish to Shopify using AI

### 1. Workflow Overview

This workflow automates the generation of detailed product descriptions from product images and associated metadata, then publishes the new product listing to a Shopify store. Its primary use case is to streamline ecommerce product listing creation by leveraging AI image analysis and content generation, combined with image hosting and Shopify API integration.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Image Processing:** Receives product data and images via a webhook, uploads the image to an external hosting service (imgbb), and sends the image along with textual product info to Google Gemini AI for analysis and content generation.

- **1.2 AI Content Parsing and Response:** Processes the AI-generated content to extract structured JSON data describing the product, including title, description, tags, bullet points, and SEO metadata. Then responds back to the initial webhook caller with this generated content.

- **1.3 Product Assembly and Shopify Publishing:** Combines the AI-generated product data with the uploaded image URL and additional input fields, aggregates all information into a Shopify product JSON format, and creates the product via Shopify's REST Admin API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Image Processing

- **Overview:**  
  This block begins the workflow by receiving product details and an image via a webhook. It then uploads the image to imgbb (an image hosting service) and simultaneously sends the image plus some textual product data to Google Gemini AI for analysis. This dual approach ensures both image hosting and AI content generation occur in parallel.

- **Nodes Involved:**  
  - Webhook  
  - Analyze image (Google Gemini)  
  - imgbb (HTTP Request)  
  - Merge  

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook (Trigger node)  
    - Configuration: Listens for POST requests at path `/387b3de0-7931-478b-b7d0-9d12c013afc6`  
    - Role: Entry point to receive incoming product data including `product_name`, `material_type`, `file` (image binary), vendor, product_type, options, and variants.  
    - Inputs: Incoming HTTP request with JSON body and image file as binary property `file`  
    - Outputs: Passes received data to "Analyze image" and "imgbb" nodes  
    - Edge cases: Missing or malformed input data; invalid image file; incorrect content-type in the request

  - **Analyze image (Google Gemini AI)**  
    - Type: AI image analysis node using Google Gemini (PaLM) API  
    - Configuration:  
      - Model: `models/gemini-2.5-flash`  
      - Input: Binary image from webhook's `file` property  
      - Additional text prompt includes product name and material type from webhook JSON body  
      - Output requested in JSON format  
    - Role: Analyzes the image and associated text to generate product-related content  
    - Inputs: Binary image + product text properties  
    - Outputs: AI-generated content in a structured text format (often JSON embedded in Markdown)  
    - Credentials: Google Gemini API credentials required  
    - Edge cases: API rate limits, authentication errors, malformed AI response, timeout, or empty results

  - **imgbb (HTTP Request)**  
    - Type: HTTP POST Request  
    - Configuration:  
      - URL: `https://api.imgbb.com/1/upload`  
      - Auth: HTTP Basic Auth with API key for imgbb  
      - Sends the same binary image file as multipart/form-data under parameter `image`  
      - Query parameter `expiration=600` (image expires in 10 minutes)  
    - Role: Upload image to imgbb to get a public URL for Shopify product image  
    - Inputs: Binary image from webhook  
    - Outputs: JSON with image URL hosted on imgbb  
    - Edge cases: Authentication failure, upload failure, network issues, API limits

  - **Merge**  
    - Type: Merge node (combine mode)  
    - Configuration: Combines outputs from "Analyze image" (index 0) and "imgbb" (index 1)  
    - Role: Combines AI-generated content and image URL into a single data structure for downstream processing  
    - Inputs: Two streams from Analyze image and imgbb nodes  
    - Outputs: Single combined data stream  
    - Edge cases: Data mismatch or missing expected fields from either input

---

#### 2.2 AI Content Parsing and Response

- **Overview:**  
  This block parses the AI-generated text output to extract JSON product details cleanly, removes Markdown formatting, and returns the generated product data as a response to the webhook caller.

- **Nodes Involved:**  
  - Code in JavaScript  
  - Respond to Webhook  
  - Edit Fields  

- **Node Details:**

  - **Code in JavaScript**  
    - Type: Code node executing JavaScript  
    - Configuration:  
      - Cleans AI text by removing Markdown code block wrappers (e.g., ```json ... ```)  
      - Attempts to parse cleaned text as JSON  
      - If parsing fails, stores raw text as HTML string under `html` property  
      - Returns parsed JSON object for further processing  
    - Inputs: AI-generated content from "Analyze image" node  
    - Outputs: Parsed JSON product content  
    - Edge cases: Malformed AI output not parseable as JSON, empty content, unexpected text format causing code errors

  - **Respond to Webhook**  
    - Type: Respond node for webhook  
    - Configuration: Sends back all incoming items as response  
    - Role: Provides immediate feedback to the webhook sender with the generated product data  
    - Inputs: Parsed JSON from Code node  
    - Outputs: HTTP response to initial webhook request  
    - Edge cases: Large response payload causing timeout, failure to respond if prior nodes error

  - **Edit Fields**  
    - Type: Set node  
    - Configuration:  
      - Assigns multiple product metadata fields under `data` property such as title, description, tags, bullet points, alt_text, meta_title, meta_description  
      - Values sourced from parsed JSON content of AI response  
    - Role: Structure and rename fields to match Shopify product JSON schema for downstream use  
    - Inputs: Parsed AI product data from Code node (via Respond node)  
    - Outputs: Structured product data for aggregation and Shopify publishing  
    - Edge cases: Missing expected fields in AI output, null or undefined values

---

#### 2.3 Product Assembly and Shopify Publishing

- **Overview:**  
  This final block aggregates the structured product data and image URL, composes the full Shopify product JSON, and creates the product via Shopify API.

- **Nodes Involved:**  
  - Aggregate  
  - HTTP Request (Shopify API)  
  - Sticky Notes (for documentation)

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate node  
    - Configuration: Aggregates incoming data fields under `data` key to prepare a single object  
    - Role: Consolidates product data and image URL into one object for Shopify JSON body  
    - Inputs: Combined AI product content and image URL from Merge node  
    - Outputs: Aggregated product data structure

  - **HTTP Request (Shopify API)**  
    - Type: HTTP POST Request  
    - Configuration:  
      - URL: Shopify Admin API endpoint for product creation (`/admin/api/2025-07/products.json`)  
      - Method: POST  
      - Headers include `X-Shopify-Access-Token` with Shopify OAuth token  
      - JSON body constructed by combining aggregated product fields and image URL  
      - Product fields populated dynamically using expressions referencing previous nodes:  
        - Title, description, tags, meta title, meta description from AI data  
        - Vendor, product_type, options, variants from webhook input  
        - Image src from imgbb URL  
      - Product is published immediately (`published: true`)  
    - Role: Creates the new product listing on Shopify with AI-generated content and uploaded image  
    - Inputs: Aggregated product JSON data  
    - Outputs: Shopify API response containing product creation confirmation or error  
    - Credentials: Shopify store access token with product write permissions  
    - Edge cases: Authentication failure, API rate limiting, malformed JSON body, missing required fields, network errors

  - **Sticky Notes**  
    - Provide documentation and instructions for the workflow setup, including API keys and webhook configuration.

---

### 3. Summary Table

| Node Name          | Node Type                   | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                       |
|--------------------|-----------------------------|----------------------------------------|------------------------|------------------------|------------------------------------------------------------------------------------------------------------------|
| Webhook            | HTTP Webhook                | Receive product data and image          | -                      | Analyze image, imgbb    | Setup webhook URL in source system; expects product details, image, and metadata                                  |
| Analyze image      | Google Gemini AI (LangChain) | AI image analysis and content generation | Webhook                | Code in JavaScript      | Connect to Google Gemini (PaLM) API for AI content generation                                                    |
| imgbb              | HTTP Request                | Upload image to hosting service         | Webhook                | Merge                  | Add your imgbb API key for image upload                                                                           |
| Merge              | Merge                      | Combine AI content and image URL        | Analyze image, imgbb    | Aggregate               |                                                                                                                  |
| Code in JavaScript  | Code Node                  | Parse AI-generated content JSON         | Analyze image          | Respond to Webhook      |                                                                                                                  |
| Respond to Webhook  | Respond to Webhook          | Return generated product data to caller | Code in JavaScript      | Edit Fields             |                                                                                                                  |
| Edit Fields        | Set Node                   | Structure AI output fields for Shopify  | Respond to Webhook      | Merge                   |                                                                                                                  |
| Aggregate          | Aggregate Node             | Consolidate product data and image URL  | Merge                   | HTTP Request            |                                                                                                                  |
| HTTP Request       | HTTP Request (Shopify API) | Create product on Shopify                | Aggregate               | -                       | Connect Shopify via access token; ensure API permissions to create products                                      |
| Sticky Note        | Sticky Note                 | Documentation and setup instructions    | -                      | -                       | ## AI Product Listing on Shopify with Image Analysis ... [Setup instructions included]                            |
| Sticky Note1       | Sticky Note                 | Documentation for block 1 setup          | -                      | -                       | ## 1. Get data and generate content and upload img to online storage                                             |
| Sticky Note2       | Sticky Note                 | Documentation for block 2 setup          | -                      | -                       | ## 2. provide generated data back to user                                                                        |
| Sticky Note3       | Sticky Note                 | Documentation for block 3 setup          | -                      | -                       | ## 3. list item to shopify                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook node**  
   - Type: HTTP Webhook  
   - HTTP Method: POST  
   - Path: `387b3de0-7931-478b-b7d0-9d12c013afc6` (or any unique path)  
   - Purpose: Receive incoming HTTP POST requests with product data and image file  
   - No credentials needed  
   - Expect JSON body with fields: `product_name`, `material_type`, `vendor`, `product_type`, `options`, `variants`, and a binary file property `file` containing the product image  

2. **Create the Analyze image node (Google Gemini AI)**  
   - Type: `@n8n/n8n-nodes-langchain.googleGemini`  
   - Operation: Analyze  
   - Resource: Image  
   - Model: Select `models/gemini-2.5-flash`  
   - Input Type: Binary  
   - Binary Property Name: `file` (from webhook)  
   - Text parameter: Set to a multiline expression incorporating `$json.body.text`, `$json.body.product_name`, and `$json.body.material_type` from webhook JSON  
   - Output format requested: JSON  
   - Connect Google Gemini (PaLM) API credentials with required access  

3. **Create the imgbb HTTP Request node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.imgbb.com/1/upload`  
   - Authentication: HTTP Basic Auth with your imgbb API key  
   - Body Content Type: Multipart-form-data  
   - Body Parameters: Add parameter `image` as binary data from property `file`  
   - Query Parameters: `expiration=600` (optional, image expiration in seconds), `key` with your imgbb API key  
   - Connect appropriate credentials or set API key directly  
   - This uploads the image and returns a JSON with a public URL  

4. **Create the Merge node**  
   - Type: Merge  
   - Mode: Combine  
   - Join Mode: Keep Everything  
   - Fields to match: `data` (default)  
   - Connect inputs from both "Analyze image" (index 0) and "imgbb" (index 1) nodes  

5. **Create the Code node for JSON parsing**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the JavaScript code provided:  
     - It cleans AI text from Markdown wrappers  
     - Attempts JSON.parse on cleaned text  
     - On failure, stores raw content as HTML string  
   - Input: Output from Analyze image node  
   - Output: Parsed JSON product data  

6. **Create Respond to Webhook node**  
   - Type: Respond to Webhook  
   - Respond With: All Incoming Items  
   - Connect input from Code node  
   - Purpose: Return generated product data immediately to the webhook client  

7. **Create Edit Fields (Set) node**  
   - Type: Set  
   - Assign fields under `data`:  
     - title, description, tags, bullet_points, alt_text, meta_title, meta_description  
   - Values sourced from parsed JSON output fields from Code node  
   - Connect input from Respond to Webhook node  

8. **Connect Edit Fields output to Merge node input (index 0)**  
   - This allows combining AI-generated product data with image URL  

9. **Create Aggregate node**  
   - Type: Aggregate  
   - Fields to aggregate: `data` field  
   - Connect input from Merge node  
   - Purpose: Consolidate all product data into one object before Shopify API call  

10. **Create HTTP Request node for Shopify product creation**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://{your-shop-name}.myshopify.com/admin/api/2025-07/products.json`  
    - Authentication: Set header parameter `X-Shopify-Access-Token` with your Shopify Admin API access token  
    - Body Content Type: JSON  
    - JSON Body: Build product JSON using expressions referencing aggregated `data` fields and webhook inputs, including:  
      - title  
      - body_html (description)  
      - vendor  
      - product_type  
      - tags  
      - published: true  
      - options, variants (from webhook data)  
      - images array with `src` set to imgbb image URL  
      - SEO fields: meta_title, meta_description  
    - Connect input from Aggregate node  

11. **Add Sticky Notes**  
    - Add descriptive sticky notes for each block as per documentation  
    - Include setup instructions for API credentials and webhook configuration  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                          | Context or Link                                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates AI-driven product description generation from images and posts listings to Shopify. Requires Google Gemini API, imgbb API, and Shopify Admin API credentials set up properly.        | Workflow Overview                                                                                                                 |
| Setup checklist: Configure webhook URL in source system; add Google Gemini (PaLM) API credentials; add imgbb API key; add Shopify Admin API access token with product write permissions.               | Sticky Note content                                                                                                               |
| Google Gemini model used: `models/gemini-2.5-flash` – ensure access and quota on Google Cloud Platform.                                                                                              | Analyze image node configuration                                                                                                 |
| imgbb image upload uses expiration parameter (600 seconds) to limit image lifetime; adjust as needed.                                                                                                | imgbb node configuration                                                                                                         |
| Shopify API version used: `2025-07` – keep Shopify API version updated to avoid deprecation issues.                                                                                                  | HTTP Request node (Shopify)                                                                                                      |
| The AI-generated content parsing code handles cases where AI may return non-JSON by storing fallback HTML text. Ensure that webhook clients can handle such fallback gracefully.                      | Code in JavaScript node, error handling                                                                                          |
| Shopify product creation requires all mandatory fields (title, vendor, variants, etc.) – ensure incoming webhook data contains these or adjust node logic accordingly.                              | HTTP Request node, webhook input expectations                                                                                    |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.