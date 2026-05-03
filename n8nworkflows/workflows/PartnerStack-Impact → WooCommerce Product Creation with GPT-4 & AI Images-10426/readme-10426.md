PartnerStack/Impact → WooCommerce Product Creation with GPT-4 & AI Images

https://n8nworkflows.xyz/workflows/partnerstack-impact---woocommerce-product-creation-with-gpt-4---ai-images-10426


# PartnerStack/Impact → WooCommerce Product Creation with GPT-4 & AI Images

### 1. Workflow Overview

This workflow automates the creation and updating of WooCommerce products based on partner data sourced from PartnerStack/Impact, enhanced by AI-generated content and images using GPT-4 and AI image generation models. It targets e-commerce managers or marketing teams who want to streamline product listing creation with enriched descriptions and visuals automatically generated from partner inputs.

The logical blocks include:

- **1.1 Scheduled Data Retrieval:** Periodically trigger the workflow to fetch partner data rows from Google Sheets.
- **1.2 Filtering Active and Unpublished Partnerships:** Identify which partners have active, unpublished products.
- **1.3 Product Data Enrichment with AI:** Use GPT-4 powered agents to generate structured product details, detailed descriptions, and image prompts.
- **1.4 AI Image Generation and Processing:** Generate product images from prompts, resize, rename, and prepare images for upload.
- **1.5 Product Publication:** Upload images, update image metadata, publish the product to the WooCommerce website via HTTP requests, and update product categories.
- **1.6 Synchronization and Error Handling:** Update Google Sheets with product publication status and log errors if product data retrieval fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Retrieval

**Overview:**  
This block triggers the workflow on a schedule and retrieves partner data rows from Google Sheets, serving as the starting point for the process.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet

**Node Details:**  

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Initiates workflow periodically (default schedule not detailed, likely cron based)  
  - Configuration: No custom parameters provided; runs on a preset schedule  
  - Inputs: None  
  - Outputs: Connects to "Get row(s) in sheet" node  
  - Failure Modes: Trigger misconfiguration or disabled trigger stops workflow start

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves rows containing partner data  
  - Configuration: Connects to a specific Google Sheet (credentials preconfigured)  
  - Inputs: Trigger output  
  - Outputs: Connects to "Partnership Active and Not Published" filter node  
  - Failure Modes: Google Sheets API errors, auth failures, empty sheet

---

#### 1.2 Filtering Active and Unpublished Partnerships

**Overview:**  
Filters the data to process only those partnerships that are active and have not yet had their products published.

**Nodes Involved:**  
- Partnership Active and Not Published (Filter)  
- Limit

**Node Details:**  

- **Partnership Active and Not Published**  
  - Type: Filter  
  - Role: Filters Google Sheets rows to those eligible for product creation  
  - Configuration: Conditions to check partnership status and published flag  
  - Inputs: Partner data from Google Sheets  
  - Outputs: Connects to "Limit" node for processing a subset  
  - Failure Modes: Expression errors in filter logic, no matching rows

- **Limit**  
  - Type: Limit  
  - Role: Caps the number of rows processed per run to avoid overload  
  - Configuration: Limit count not explicitly detailed (likely defaults or set in node)  
  - Inputs: Filtered rows  
  - Outputs: Connects to "Get Product Details from Website" node  
  - Failure Modes: Misconfiguration can cause zero or excess processing

---

#### 1.3 Product Data Enrichment with AI

**Overview:**  
This block uses GPT-4 powered Langchain agents to generate structured product data, detailed descriptions, and image prompts based on partner input and existing product info.

**Nodes Involved:**  
- Get Product Details from Website (HTTP Request)  
- Parse Product Data (Code)  
- OpenAI Chat Model1, OpenAI Chat Model2, OpenAI Chat Model3 (Langchain LM Chat OpenAI)  
- Structured Output Parser1 (Langchain Output Parser Structured)  
- Product Basic Data (Langchain Agent)  
- Detailed Descriptoin Writer (Langchain Agent)  
- Image Prompt Generator (Langchain Agent)

**Node Details:**  

- **Get Product Details from Website**  
  - Type: HTTP Request  
  - Role: Retrieves existing product details from WooCommerce or similar endpoint  
  - Configuration: API endpoint URL, auth headers (likely OAuth2)  
  - Inputs: Limited partner rows  
  - Outputs: On success to "Parse Product Data," on failure to "Update about Error"  
  - Failure Modes: API errors, timeouts, invalid product IDs  
  - OnError: "Continue" allows workflow continuation after failure

- **Parse Product Data**  
  - Type: Code  
  - Role: Parses raw HTTP response into structured JSON for AI input  
  - Configuration: JavaScript or similar code for data extraction  
  - Inputs: HTTP response data  
  - Outputs: Feeds into "Product Basic Data" AI agent  
  - Failure Modes: Parsing errors, unexpected data format

- **OpenAI Chat Model1,2,3**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Provides GPT-4 chat completions to AI agents  
  - Configuration: Linked to appropriate prompts and agents; uses OpenAI credentials  
  - Inputs: Prompts or data from AI agents or previous nodes  
  - Outputs: Passes results to respective AI agents or parsers  
  - Failure Modes: API quota limits, auth errors, timeout, malformed prompts

- **Structured Output Parser1**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI model's structured JSON output for downstream processing  
  - Inputs: Output from OpenAI Chat Model1  
  - Outputs: Feeds "Product Basic Data" agent  
  - Failure Modes: Parsing errors if AI output is malformed

- **Product Basic Data**  
  - Type: Langchain Agent  
  - Role: Generates base product information (title, SKU, price, etc.)  
  - Inputs: Parsed product data  
  - Outputs: Feeds "Detailed Descriptoin Writer" agent  
  - Failure Modes: AI model failures, prompt misconfigurations

- **Detailed Descriptoin Writer**  
  - Type: Langchain Agent  
  - Role: Creates rich, marketing-oriented product descriptions  
  - Inputs: Basic product data  
  - Outputs: Feeds "Image Prompt Generator" agent  
  - Failure Modes: Same as above

- **Image Prompt Generator**  
  - Type: Langchain Agent  
  - Role: Generates textual prompts for AI image generation based on product descriptions  
  - Inputs: Detailed description  
  - Outputs: Feeds "Generate and Image1" node  
  - Failure Modes: Same as above

---

#### 1.4 AI Image Generation and Processing

**Overview:**  
Uses the generated image prompts to create AI images, then processes these images with resizing and renaming before upload.

**Nodes Involved:**  
- Generate and Image1 (Langchain OpenAI)  
- Resize Image1 (Edit Image) [disabled]  
- Rename Image (Code)

**Node Details:**  

- **Generate and Image1**  
  - Type: Langchain OpenAI (likely DALL·E or similar)  
  - Role: Generates product images from prompts  
  - Inputs: Image prompt text  
  - Outputs: Connects to "Resize Image1" (disabled)  
  - Failure Modes: Image generation quota, API errors, invalid prompts

- **Resize Image1** (Disabled)  
  - Type: Edit Image  
  - Role: Intended to resize images but currently disabled  
  - Inputs: Generated images  
  - Outputs: Connects to "Rename Image" node  
  - Failure Modes: Disabled, no effect

- **Rename Image**  
  - Type: Code  
  - Role: Renames or formats image filenames to match product naming conventions  
  - Inputs: Image data, possibly metadata  
  - Outputs: Feeds into "Upload Image"  
  - Failure Modes: Code errors, incorrect renaming logic

---

#### 1.5 Product Publication

**Overview:**  
Uploads images, updates image metadata, publishes the product with enriched data to WooCommerce, updates product images and categories, and finally updates the Google Sheet with product status.

**Nodes Involved:**  
- Upload Image (HTTP Request)  
- Update Image Metadata (HTTP Request)  
- Publish Product to Website (HTTP Request)  
- Update Product Image (HTTP Request)  
- Update Product Category (HTTP Request)  
- Update row in sheet (Google Sheets)

**Node Details:**  

- **Upload Image**  
  - Type: HTTP Request  
  - Role: Uploads renamed AI-generated images to website or CDN  
  - Inputs: Image file data  
  - Outputs: Feeds "Update Image Metadata"  
  - Failure Modes: Upload failures, auth errors, file size limits

- **Update Image Metadata**  
  - Type: HTTP Request  
  - Role: Updates image metadata (alt text, description) on server  
  - Inputs: Upload response data  
  - Outputs: Feeds "Publish Product to Website"  
  - Failure Modes: API errors, invalid metadata

- **Publish Product to Website**  
  - Type: HTTP Request  
  - Role: Creates or updates product entry on WooCommerce  
  - Inputs: Product data including updated images  
  - Outputs: Connects to "Update Product Image"  
  - Failure Modes: API errors, validation errors, product duplication

- **Update Product Image**  
  - Type: HTTP Request  
  - Role: Associates uploaded images with the product in WooCommerce  
  - Inputs: Product publication response  
  - Outputs: Connects to "Update Product Category"  
  - Failure Modes: API errors, image linkage failures

- **Update Product Category**  
  - Type: HTTP Request  
  - Role: Assigns product to appropriate categories on WooCommerce  
  - Inputs: Product image update response  
  - Outputs: Connects to "Update row in sheet"  
  - Failure Modes: API errors, invalid category IDs

- **Update row in sheet**  
  - Type: Google Sheets  
  - Role: Marks the partner's row as published and updates other status fields  
  - Inputs: Successful product creation data  
  - Outputs: End of workflow  
  - Failure Modes: Google Sheets update errors, concurrency issues

---

#### 1.6 Synchronization and Error Handling

**Overview:**  
Handles error logging back to Google Sheets if product detail retrieval from the website fails.

**Nodes Involved:**  
- Update about Error (Google Sheets)

**Node Details:**  

- **Update about Error**  
  - Type: Google Sheets  
  - Role: Updates the partner row with error messages or flags on failed product data retrieval  
  - Inputs: Failure path from "Get Product Details from Website"  
  - Outputs: End of workflow for error path  
  - Failure Modes: Google Sheets write errors

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                                  | Input Node(s)                  | Output Node(s)                     | Sticky Note                   |
|-------------------------------|----------------------------------|-------------------------------------------------|-------------------------------|----------------------------------|------------------------------|
| Schedule Trigger               | Schedule Trigger                 | Periodically triggers workflow                   | None                          | Get row(s) in sheet              |                              |
| Get row(s) in sheet           | Google Sheets                   | Retrieves partner data rows                       | Schedule Trigger              | Partnership Active and Not Published |                              |
| Partnership Active and Not Published | Filter                         | Filters active, unpublished partnerships          | Get row(s) in sheet           | Limit                           |                              |
| Limit                         | Limit                           | Limits number of rows processed                   | Partnership Active and Not Published | Get Product Details from Website |                              |
| Get Product Details from Website | HTTP Request                   | Gets existing product info from website          | Limit                         | Parse Product Data, Update about Error |                              |
| Parse Product Data            | Code                            | Parses product data HTTP response                 | Get Product Details from Website | Product Basic Data               |                              |
| OpenAI Chat Model1            | Langchain LM Chat OpenAI         | GPT-4 model for base product data                 | Structured Output Parser1      | Product Basic Data               |                              |
| Structured Output Parser1     | Langchain Output Parser Structured | Parses AI response into structured format         | OpenAI Chat Model1            | Product Basic Data               |                              |
| Product Basic Data            | Langchain Agent                  | Generates base product info                        | Parse Product Data, Structured Output Parser1 | Detailed Descriptoin Writer      |                              |
| OpenAI Chat Model3            | Langchain LM Chat OpenAI         | GPT-4 model for detailed description              | Detailed Descriptoin Writer    | Detailed Descriptoin Writer      |                              |
| Detailed Descriptoin Writer   | Langchain Agent                  | Writes detailed product descriptions               | Product Basic Data            | Image Prompt Generator           |                              |
| OpenAI Chat Model2            | Langchain LM Chat OpenAI         | GPT-4 model for image prompt generation            | Image Prompt Generator         | Image Prompt Generator           |                              |
| Image Prompt Generator        | Langchain Agent                  | Creates image generation prompts                   | Detailed Descriptoin Writer    | Generate and Image1              |                              |
| Generate and Image1           | Langchain OpenAI                 | Generates AI images from prompts                   | Image Prompt Generator         | Resize Image1                   |                              |
| Resize Image1 (disabled)      | Edit Image                      | Intended image resizing (disabled)                 | Generate and Image1            | Rename Image                   |                              |
| Rename Image                 | Code                            | Renames image files                                 | Resize Image1                 | Upload Image                   |                              |
| Upload Image                 | HTTP Request                   | Uploads images to server/CDN                         | Rename Image                 | Update Image Metadata           |                              |
| Update Image Metadata        | HTTP Request                   | Updates metadata of uploaded images                  | Upload Image                 | Publish Product to Website      |                              |
| Publish Product to Website   | HTTP Request                   | Publishes or updates product on WooCommerce         | Update Image Metadata         | Update Product Image            |                              |
| Update Product Image         | HTTP Request                   | Associates uploaded images with product             | Publish Product to Website    | Update Product Category         |                              |
| Update Product Category      | HTTP Request                   | Assigns product categories                            | Update Product Image          | Update row in sheet             |                              |
| Update row in sheet          | Google Sheets                   | Updates partner sheet with publication status       | Update Product Category       | None                          |                              |
| Update about Error           | Google Sheets                   | Logs errors when product data retrieval fails       | Get Product Details from Website (error) | None                          |                              |
| Sticky Note                  | Sticky Note                    | (Empty content)                                     | None                          | None                          |                              |
| Sticky Note1                 | Sticky Note                    | (Empty content)                                     | None                          | None                          |                              |
| Sticky Note11                | Sticky Note                    | (Empty content)                                     | None                          | None                          |                              |
| Sticky Note12                | Sticky Note                    | (Empty content)                                     | None                          | None                          |                              |
| Sticky Note2                 | Sticky Note                    | (Empty content)                                     | None                          | None                          |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger node:**  
   - Type: Schedule Trigger  
   - Configure schedule as desired (e.g., hourly, daily).

2. **Add Google Sheets node "Get row(s) in sheet":**  
   - Connect to Schedule Trigger.  
   - Configure to read rows from the partner data spreadsheet.  
   - Use Google Sheets credentials with read access.

3. **Add Filter node "Partnership Active and Not Published":**  
   - Connect to "Get row(s) in sheet".  
   - Set filter conditions to select rows where partnership is active and product is not published.

4. **Add Limit node:**  
   - Connect to Filter node.  
   - Set max number of rows to process per run (e.g., 5 or 10).

5. **Add HTTP Request node "Get Product Details from Website":**  
   - Connect to Limit node.  
   - Configure API endpoint to retrieve product info by partner/product ID.  
   - Set authentication (OAuth2 or API key).  
   - Enable "Continue on Error" to allow error handling.

6. **Add Code node "Parse Product Data":**  
   - Connect to successful output of "Get Product Details from Website".  
   - Write code to parse HTTP response JSON into structured data for AI input.

7. **Add Google Sheets node "Update about Error":**  
   - Connect to error output of "Get Product Details from Website".  
   - Configure to update partner row with error details.

8. **Add Langchain LM Chat OpenAI node "OpenAI Chat Model1":**  
   - Configure with OpenAI credentials.  
   - Set model to GPT-4.  
   - Connect to "Structured Output Parser1" node.

9. **Add Langchain Output Parser node "Structured Output Parser1":**  
   - Connect to "OpenAI Chat Model1".  
   - Configure to parse structured JSON output.

10. **Add Langchain Agent node "Product Basic Data":**  
    - Connect to "Structured Output Parser1" and "Parse Product Data".  
    - Configure prompt for generating base product info.

11. **Add Langchain LM Chat OpenAI nodes "OpenAI Chat Model3" and "OpenAI Chat Model2":**  
    - Both configured with GPT-4 and proper credentials.  
    - "OpenAI Chat Model3" connects to "Detailed Descriptoin Writer".  
    - "OpenAI Chat Model2" connects to "Image Prompt Generator".

12. **Add Langchain Agent nodes "Detailed Descriptoin Writer" and "Image Prompt Generator":**  
    - "Detailed Descriptoin Writer" receives input from "Product Basic Data" and "OpenAI Chat Model3".  
    - "Image Prompt Generator" receives input from "Detailed Descriptoin Writer" and "OpenAI Chat Model2".

13. **Add Langchain OpenAI node "Generate and Image1":**  
    - Connect to "Image Prompt Generator".  
    - Configure for image generation (DALL·E or similar).  
    - Use OpenAI credentials.

14. **Add Edit Image node "Resize Image1" (optional):**  
    - Connect to "Generate and Image1".  
    - Configure resizing parameters if needed (currently disabled).

15. **Add Code node "Rename Image":**  
    - Connect to "Resize Image1" or directly to "Generate and Image1" if resize disabled.  
    - Write code to rename images to desired filename format.

16. **Add HTTP Request node "Upload Image":**  
    - Connect to "Rename Image".  
    - Configure API endpoint and credentials to upload images.

17. **Add HTTP Request node "Update Image Metadata":**  
    - Connect to "Upload Image".  
    - Configure to update image alt text and descriptions.

18. **Add HTTP Request node "Publish Product to Website":**  
    - Connect to "Update Image Metadata".  
    - Configure to create/update product on WooCommerce website.

19. **Add HTTP Request node "Update Product Image":**  
    - Connect to "Publish Product to Website".  
    - Configure to associate images with the product.

20. **Add HTTP Request node "Update Product Category":**  
    - Connect to "Update Product Image".  
    - Configure to assign product categories.

21. **Add Google Sheets node "Update row in sheet":**  
    - Connect to "Update Product Category".  
    - Configure to mark the product as published in the Google Sheet.

22. **Add error handling path:**  
    - Connect error output of "Get Product Details from Website" to "Update about Error" Google Sheets node.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow leverages GPT-4 and Langchain agents for advanced natural language understanding.       | See OpenAI GPT-4 and Langchain official docs                    |
| Image generation uses OpenAI's DALL·E or equivalent via Langchain integration.                    | OpenAI DALL·E API documentation                                 |
| Google Sheets API credentials must have read/write access for partner data and status updates.   | Google Sheets API documentation                                 |
| WooCommerce API requires authentication (OAuth2 or API key) with permissions to create products. | WooCommerce REST API docs                                       |
| The Resize Image node is currently disabled but can be enabled for image dimension control.      | n8n Edit Image node documentation                               |
| Error handling updates sheet rows with error messages to allow manual follow-up.                 | Best practice for monitoring automated workflows                |

---

This structured and detailed documentation enables accurate understanding, replication, and modification of the workflow by developers and AI agents alike.