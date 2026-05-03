Generate Shopify Product Listings from Images with Gemini AI and Airtable

https://n8nworkflows.xyz/workflows/generate-shopify-product-listings-from-images-with-gemini-ai-and-airtable-10008


# Generate Shopify Product Listings from Images with Gemini AI and Airtable

### 1. Workflow Overview

This workflow automates the generation of Shopify product listings directly from digital artwork images using AI-powered analysis and Airtable for data management. It is designed for digital artists, print-on-demand sellers, and e-commerce managers who want to streamline product creation from image assets.

The workflow is structured into four main logical blocks:

- **1.1 Input Reception & Data Filtering**: Fetch raw image entries from Airtable, filter unused images, and prepare for analysis.  
- **1.2 AI-Based Image Analysis & Data Enrichment**: Download images from Google Drive, analyze images with OpenAI’s GPT-based image analysis, and update Airtable with structured image metadata.  
- **1.3 Product Content Generation Using AI and Shopify Collections**: Fetch Shopify collections, refine collection data, and generate SEO-optimized product details from analyzed image data using AI (Gemini and LangChain chains). Update Airtable with generated product details.  
- **1.4 Shopify Product Posting & Status Update**: Retrieve generated product details from Airtable and create Shopify products via the API, then update Airtable to mark products as posted.  

Batch processing and loop nodes enable efficient handling of multiple images and products. The workflow can be triggered manually or automated through webhooks or schedules.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Filtering

**Overview:**  
This block initiates the workflow, fetching raw image entries from Airtable marked as 'Unused' and filtering them for processing.

**Nodes involved:**  
- start  
- get_raw_image_table_data  
- filter_raw_row  
- do nothing  

**Node Details:**  

- **start**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Input: None  
  - Output: Initiates request to fetch raw image data.  
  - Edge cases: None.  

- **get_raw_image_table_data**  
  - Type: Airtable Node (Search Operation)  
  - Role: Fetches all entries from Airtable base "shopify_digital_product", table "raw_data".  
  - Credentials: Airtable API token with access to the base.  
  - Input: Trigger from start node.  
  - Output: All raw image records.  
  - Edge cases: API rate limits, invalid credentials, empty base.  

- **filter_raw_row**  
  - Type: Switch Node  
  - Role: Splits records based on their `status` field into two outputs: "Unused" and "Used".  
  - Expression: `{{$json.status}}` equals "Unused" or "Used".  
  - Input: Airtable data from previous node.  
  - Output:  
    - Unused images proceed for analysis.  
    - Used images route to "do nothing" (no operation).  
  - Edge cases: Missing or malformed `status` field, case sensitivity issues.  

- **do nothing**  
  - Type: No Operation  
  - Role: Placeholder for unused images that are already processed.  
  - Input: "Used" filtered data.  
  - Output: None.  

---

#### 2.2 AI-Based Image Analysis & Data Enrichment

**Overview:**  
Downloads each unused image from Google Drive, analyzes the image using OpenAI’s GPT-based image analysis (LangChain OpenAI node), extracts structured metadata, and updates the Airtable record marking the image as "Used".

**Nodes involved:**  
- loop_image_analyzation  
- download_image  
- analyze_image  
- update_image_data  

**Node Details:**  

- **loop_image_analyzation**  
  - Type: SplitInBatches  
  - Role: Processes images one or multiple at a time to avoid overloading API or exceeding rate limits.  
  - Input: Unused images from filter.  
  - Output: Individual image records for sequential processing.  
  - Edge cases: Large batch sizes may cause throttling or timeouts.  

- **download_image**  
  - Type: Google Drive (Download)  
  - Role: Downloads image file using `drive_file_id` from each Airtable record.  
  - Credentials: Google OAuth2 with access to Google Drive.  
  - Inputs: `drive_file_id` expression from current item.  
  - Output: Binary image data for analysis.  
  - Edge cases: Invalid file ID, expired tokens, file missing or permissions error.  

- **analyze_image**  
  - Type: LangChain OpenAI Image Analysis  
  - Role: Uses GPT-4o model to analyze the image content and extract structured JSON details about characters, series, poster text, style, and mood.  
  - Credentials: OpenAI API Paid plan.  
  - Configuration: Instruction prompt limits hallucinations, restricts to visible details, expects JSON output with keys such as `character_name`, `series_name`, `category`, `poster_text`, `poster_type`.  
  - Input: Base64 image from Google Drive node.  
  - Output: AI-generated structured JSON analysis.  
  - Edge cases: API timeout, invalid image format, incomplete or ambiguous image content leading to null fields.  

- **update_image_data**  
  - Type: Airtable Update  
  - Role: Updates the original Airtable raw image record with AI analyzed data (`image_data` field) and sets `status` to "Used".  
  - Credentials: Airtable API token.  
  - Matching column: `drive_file_id`.  
  - Edge cases: Update conflicts, missing record, API limits.  

---

#### 2.3 Product Content Generation Using AI and Shopify Collections

**Overview:**  
Fetches Shopify collections for the store, refines collection data, retrieves analyzed images, generates Shopify product details using AI (Gemini and LangChain chain), and updates Airtable product details records.

**Nodes involved:**  
- store_id  
- get_collection_data  
- refine_collection_output  
- get_analyzed_row  
- limit_  
- Basic LLM Chain  
- Structured Output Parser  
- Auto-fixing Output Parser  
- update_product_details  
- product_info_creation  
- limit_2  
- get_product_table  
- If  
- do nothing1  

**Node Details:**  

- **store_id**  
  - Type: Set  
  - Role: Sets the Shopify store identifier string used in API calls.  
  - Output: JSON with `store_id` field, e.g., "1qzkpy-1s".  

- **get_collection_data**  
  - Type: HTTP Request  
  - Role: Fetches Shopify custom collections via Shopify Admin API using the store ID.  
  - Credentials: Shopify OAuth Access Token.  
  - URL constructed dynamically using the store ID.  
  - Output: Raw Shopify collections JSON response.  
  - Edge cases: API rate limits, invalid tokens, network errors.  

- **refine_collection_output**  
  - Type: Code (JavaScript)  
  - Role: Simplifies collections array to only include `id` and `title` for easier matching downstream.  
  - Input: Raw Shopify collections JSON.  
  - Output: Array of simplified collection objects.  

- **get_analyzed_row**  
  - Type: Airtable Search  
  - Role: Fetches raw image records from Airtable where `status` is "Used" (i.e., analyzed images ready for product generation).  
  - Credentials: Airtable API token.  
  - Output: List of analyzed images.  
  - Executes once per workflow run.  

- **limit_**  
  - Type: Limit  
  - Role: Limits the number of analyzed images processed simultaneously to avoid API overload.  
  - Input: Analyzed image records.  

- **Basic LLM Chain**  
  - Type: LangChain Chain LLM (OpenAI + Gemini AI)  
  - Role: Uses Gemini AI model combined with LangChain to generate Shopify product details from analyzed image data and available Shopify collections.  
  - Input: Structured analyzed image data + collection info (title and id) as context.  
  - Output: JSON product details including title, description, category, SEO metadata.  
  - Edge cases: AI output parsing errors, hallucinations (mitigated by prompt instructions), API failures.  

- **Structured Output Parser**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Validates and parses AI output into JSON schema with expected product fields.  
  - Input: Raw AI output text.  
  - Output: Parsed JSON object.  

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser (Auto-fixing)  
  - Role: Attempts to auto-correct minor format issues in AI output to produce valid JSON.  
  - Input: Output from Structured Output Parser.  
  - Output: Cleaned JSON for product detail fields.  

- **update_product_details**  
  - Type: Airtable Update  
  - Role: Updates Airtable `product_details` table with generated product info and marks status as `generated`.  
  - Matching column: `drive_file_id`.  
  - Edge cases: Airtable API update errors, data conflicts.  

- **product_info_creation**  
  - Type: SplitInBatches  
  - Role: Processes product records in batches for efficient downstream processing.  
  - Output: Batches product data for posting to Shopify.  

- **limit_2**  
  - Type: Limit  
  - Role: Limits the number of product data batches processed at once for API safety.  

- **get_product_table**  
  - Type: Airtable Search  
  - Role: Fetches product details from Airtable.  
  - Output: Product data for further validation or posting.  

- **If**  
  - Type: If Condition  
  - Role: Checks if the product record's `drive_file_id` is empty, to decide whether to proceed or do nothing.  
  - Output: Routes to "do nothing1" or to next steps accordingly.  

- **do nothing1**  
  - Type: No Operation  
  - Role: Placeholder to bypass empty or invalid product entries.  

---

#### 2.4 Shopify Product Posting & Status Update

**Overview:**  
Takes generated Shopify product details from Airtable, creates products via Shopify API, and updates Airtable to mark products as posted.

**Nodes involved:**  
- get_analyzed_row2  
- Create a product  
- update_product_update_status  
- update_drive_file_id  
- Wait  

**Node Details:**  

- **get_analyzed_row2**  
  - Type: Airtable Search  
  - Role: Retrieves product records from Airtable where `product_status` is "generated" ready for Shopify posting.  
  - Credentials: Airtable API token.  
  - Executes once per run.  

- **Create a product**  
  - Type: Shopify Node (Create Product)  
  - Role: Creates a new product in Shopify using the generated product details (title, description, SEO handle, etc.).  
  - Credentials: Shopify OAuth Access Token.  
  - Input: Product details from Airtable node.  
  - Edge cases: API rate limits, invalid data, network errors.  

- **update_product_update_status**  
  - Type: Airtable Update  
  - Role: Updates the product record in Airtable, changing `product_status` to "posted" after successful Shopify creation.  
  - Matching column: `product_id`.  
  - Edge cases: Update conflicts, Airtable API errors.  

- **update_drive_file_id**  
  - Type: Airtable Create  
  - Role: Creates or updates a record in Airtable product details table with the `drive_file_id` of the Shopify product, linking image and product.  
  - Edge cases: Duplicate entries, missing IDs.  

- **Wait**  
  - Type: Wait (Webhook)  
  - Role: Pause or synchronize workflow steps if necessary; also linked to store_id to trigger collection fetching.  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                      | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                |
|-------------------------|--------------------------------|-----------------------------------------------------|-----------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| start                   | Manual Trigger                 | Entry point                                         | -                     | get_raw_image_table_data    | # Digital Art to Shopify Automation  This template automates the process of turning digital art files into fully structured Shopify products using n8n, OpenAI, Airtable, Google Drive, and Shopify. The workflow fetches raw digital artwork, analyzes it for categories and descriptions using AI, generates SEO-optimized product content, and posts it to Shopify. Ideal for digital artists and e-commerce managers.                                                                |
| get_raw_image_table_data| Airtable (Search)              | Fetch raw image records                             | start                 | filter_raw_row             |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| filter_raw_row          | Switch                        | Filter images by status (Unused/Used)               | get_raw_image_table_data| loop_image_analyzation, do nothing |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| do nothing              | No Operation                  | Placeholder for processed images                     | filter_raw_row         | -                          |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| loop_image_analyzation  | SplitInBatches                | Batch processing of images for analysis             | filter_raw_row         | limit_2, limit_1            |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| download_image          | Google Drive (Download)       | Downloads image file from Google Drive               | loop_image_analyzation | analyze_image               |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| analyze_image           | LangChain OpenAI (Image Analyze) | AI analyzes image content and extracts JSON metadata| download_image         | update_image_data           |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| update_image_data       | Airtable (Update)             | Update Airtable with AI image analysis data         | analyze_image          | loop_image_analyzation      |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| store_id                | Set                          | Sets Shopify store ID for API calls                  | Wait                   | get_collection_data         |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| get_collection_data     | HTTP Request (Shopify API)    | Fetch Shopify collections                            | store_id               | refine_collection_output    |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| refine_collection_output| Code (JS)                    | Simplifies Shopify collections to id & title         | get_collection_data    | product_info_creation       |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| get_analyzed_row        | Airtable (Search)             | Fetch analyzed images from Airtable                   | product_info_creation   | limit_                     |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| limit_                  | Limit                        | Limits number of analyzed images processed            | get_analyzed_row        | Basic LLM Chain             |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Basic LLM Chain         | LangChain Chain LLM           | Generates Shopify product details from AI analysis   | limit_                  | update_product_details      |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Structured Output Parser| LangChain Output Parser       | Parses AI output into structured JSON                | Basic LLM Chain         | Auto-fixing Output Parser   |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Auto-fixing Output Parser| LangChain Output Parser      | Auto-corrects AI output JSON format                   | Structured Output Parser| Basic LLM Chain             |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| update_product_details  | Airtable (Update)             | Updates Airtable product details with generated data | Basic LLM Chain         | product_info_creation       |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| product_info_creation   | SplitInBatches                | Batch processing of product records                   | update_product_details  | get_analyzed_row2, get_analyzed_row |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| limit_2                 | Limit                        | Limits number of products processed in batch          | loop_image_analyzation  | get_product_table           |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| get_product_table       | Airtable (Search)             | Fetch product details for validation                   | limit_2                 | If                         |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| If                      | If Condition                 | Checks if product's drive_file_id is empty            | get_product_table       | do nothing1, update_drive_file_id |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| do nothing1             | No Operation                  | Skip processing if no drive_file_id                    | If                      | -                          |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| get_analyzed_row2       | Airtable (Search)             | Fetch product details marked as 'generated'           | product_info_creation   | Create a product            |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Create a product        | Shopify Node (Create Product) | Creates product in Shopify using generated data       | get_analyzed_row2       | update_product_update_status|                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| update_product_update_status | Airtable (Update)         | Marks product status as 'posted' after Shopify creation| Create a product        | -                          |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| update_drive_file_id    | Airtable (Create)             | Creates/updates product details record with drive_file_id | If                      | Wait                       |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Wait                    | Wait (Webhook)                | Synchronizes workflow steps, triggers collection fetch | update_drive_file_id     | store_id                   |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| do nothing              | No Operation                  | Placeholder for unused images                          | filter_raw_row         | -                          |                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Sticky Note             | Sticky Note                  | Overview and user guide                                | -                      | -                          | # Digital Art to Shopify Automation\n\n## Overview :\nThis template automates the process of turning digital art files into fully structured Shopify products using n8n, OpenAI, Airtable, Google Drive, and Shopify. The workflow fetches raw digital artwork, analyzes it for categories and descriptions using AI, generates SEO-optimized product content, and posts it to Shopify.\n\n## Ideal for : \nDigital artists, print-on-demand sellers, e-commerce store managers, and anyone looking to automate Shopify product creation for digital art.\n\n## SEO Keywords : \nDigital art automation, Shopify product creation, n8n workflow template, AI image analysis, Airtable to Shopify integration, OpenAI product description generator, automated e-commerce workflow.\n\n\n\n# User Guide\nThis workflow automates image analysis, product detail generation, and Shopify product posting.\n\n---\n## Step 1: Upload Images\n- Upload your digital artwork or poster images to **Google Drive**.  \n- In **Airtable Raw Image Table**, add new entries with:\n  - `drive_file_id` → Google Drive file ID  \n  - `status` → set as `Unused`  \n\n---\n## Step 2: Image Analysis (Subworkflow 1)\n- Fetch `Unused` images from Airtable.  \n- Download images from Google Drive.  \n- AI analyzes images to extract:\n  - Characters\n  - Series\n  - Poster text\n  - Style\n  - Category\n  - Visual mood  \n- Updates analyzed data back to Airtable.  \n- Marks the image `Used` after processing.  \n\n---\n## Step 3: Product Generation (Subworkflow 2)\n- Fetch analyzed image data from Airtable.  \n- Retrieve available Shopify collections for your store.  \n- AI generates Shopify-ready product details:\n  - `product_title`\n  - `product_description` (SEO-friendly, 4–6 sentences)\n  - `product_category` and matched collection ID\n  - SEO fields: `SEL_page_title`, `SEL_meta_description`, `SEL_url_handle`  \n- Updates generated product details in Airtable and marks as `generated`.  \n\n---\n## Step 4: Posting Products to Shopify\n- Fetch `generated` product details from Airtable.  \n- Automatically create products in Shopify using the API.  \n- Update Airtable product status to `posted` once successfully added.  \n\n---\n## Step 5: Batch Processing & Automation\n- Workflow processes images and products in **batches** for efficiency.  \n- Can be triggered automatically via schedule or webhook.  \n|
| Sticky Note1            | Sticky Note                  | Explains digital image analysis block                 | -                      | -                          | # Digital Image Analysis\n\n## Purpose:\nAutomatically fetch raw digital images, analyze their content using AI, and update the results in Airtable for further processing.\n\n## How It Works:\nThis workflow retrieves all unused images from Airtable, downloads them from Google Drive, and uses an AI model to analyze key details \nlike characters, series, category, poster text, and type. After analysis, the results are updated back into Airtable, and images are \nmarked as processed, creating a seamless loop for continuous image handling.|
| Sticky Note2            | Sticky Note                  | Explains Shopify product creation block                 | -                      | -                          | # Shopify Product Creation\n\n## Purpose:\nConvert AI-analyzed image data into structured Shopify product details and post them automatically to your Shopify store.\n\n## How It Works:\nHere we will fetch analyzed image data from Airtable.\nIt retrieves available Shopify collections to match products with the most relevant category.\n\n## AI (LLM) generates structured product details including:\n- Product title\n- Description (SEO-friendly and engaging)\n- Product category and matched collection ID\n- SEO metadata (page title, meta description, URL handle)\nGenerated product details are updated back into Airtable.\n\n## The workflow then posts the product to Shopify automatically.\nFinally, the workflow updates the product status in Airtable to indicate it has been posted.\n|
| Sticky Note4            | Sticky Note                  | Author contact and expertise details                   | -                      | -                          | # Author Details\n\n## Manish Kumar\n### Expert Designer & Automation Engineer for Custom Shopify Apps\n\nI specialize in designing seamless customer experiences and automating workflows for Shopify, web, and business platforms. From custom apps integration to process optimization, my solutions empower teams to work smarter and scale faster.\n\n## Contact:\n### 📧 [manipritraj@gmail.com](mailto:manipritraj@gmail.com)\n### 📞 +91-9334888899\n|

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Node**  
   - Name: `start`  
   - Type: Manual Trigger (n8n native)  

2. **Create Airtable Node for Raw Images**  
   - Name: `get_raw_image_table_data`  
   - Type: Airtable (Search)  
   - Credentials: Provide Airtable API token with access to base "shopify_digital_product".  
   - Base ID: `appYnPOyDUwImADqj`  
   - Table ID: `tbl1s3jAxiwbfaQQO` (raw_data)  
   - Operation: Search (no filter)  

3. **Add Switch Node to Filter on `status`**  
   - Name: `filter_raw_row`  
   - Conditions:  
     - If `$json.status` equals `"Unused"` output to `Unused` branch  
     - If `$json.status` equals `"Used"` output to `Used` branch  

4. **Create No Operation Node for Used**  
   - Name: `do nothing`  
   - Type: No Operation  

5. **Add SplitInBatches Node for Processing Unused Images**  
   - Name: `loop_image_analyzation`  
   - Type: SplitInBatches  
   - Batch size: default or 1 for safe API calls  

6. **Create Google Drive Node to Download Image**  
   - Name: `download_image`  
   - Type: Google Drive (Download)  
   - Credentials: Google OAuth2 with Drive access  
   - Parameter: `fileId` set to `={{ $json.drive_file_id }}`  

7. **Add LangChain OpenAI Node for Image Analysis**  
   - Name: `analyze_image`  
   - Type: LangChain OpenAI (Image Analyze)  
   - Credentials: OpenAI API Paid  
   - Model: `chatgpt-4o-latest`  
   - InputType: Base64 (from downloaded image)  
   - Prompt: As defined, instructing to analyze poster content and output strict JSON with character_name, series_name, category, poster_text, poster_type, avoiding hallucination.  

8. **Add Airtable Update Node to Save Analysis**  
   - Name: `update_image_data`  
   - Type: Airtable Update  
   - Credentials: Same Airtable token  
   - Base/Table: same as raw image table  
   - Matching field: `drive_file_id`  
   - Update fields:  
     - `image_data` with AI output  
     - `status` set to `"Used"`  

9. **Loop Back from `update_image_data` to `loop_image_analyzation`** for continuous batch processing.  

10. **Add Set Node to Define Store ID**  
    - Name: `store_id`  
    - Type: Set  
    - Field: `store_id` = `1qzkpy-1s` (example)  

11. **HTTP Request Node to Fetch Shopify Collections**  
    - Name: `get_collection_data`  
    - Type: HTTP Request  
    - URL: `https://{{ $json.store_id }}.myshopify.com/admin/api/2025-10/custom_collections.json?limit=50`  
    - Authentication: Shopify OAuth Access Token  
    - Credentials: Shopify access token OAuth2  

12. **Code Node to Refine Collections**  
    - Name: `refine_collection_output`  
    - Type: Code (JavaScript)  
    - Code: Map collections to `{id, title}` only.  

13. **Airtable Search Node for Analyzed Images**  
    - Name: `get_analyzed_row`  
    - Base/Table: Raw image table  
    - Filter formula: `{status} = 'Used'`  

14. **Limit Node to Control Batch Size**  
    - Name: `limit_`  
    - Type: Limit  

15. **LangChain Chain LLM Node for Product Generation**  
    - Name: `Basic LLM Chain`  
    - Model: Gemini AI (Google Palm) + LangChain  
    - Input: Pass analyzed image data and refined collection data  
    - Prompt: As per workflow, to generate Shopify product details in JSON format with SEO fields.  

16. **Add Structured Output Parser Node**  
    - Name: `Structured Output Parser`  
    - JSON Schema Example: as defined in the workflow for product details  

17. **Add Auto-fixing Output Parser Node**  
    - Name: `Auto-fixing Output Parser`  
    - Wraps structured parser to auto-correct format  

18. **Airtable Update Node for Product Details**  
    - Name: `update_product_details`  
    - Base/Table: product_details table  
    - Matching: `drive_file_id`  
    - Fields: product_id (unique generated), product_title, description, category, SEO fields, product_status = "generated"  

19. **SplitInBatches Node to Process Products**  
    - Name: `product_info_creation`  

20. **Limit Node to Control Product Batch Size**  
    - Name: `limit_2`  

21. **Airtable Search Node for Product Details**  
    - Name: `get_product_table`  
    - Base/Table: product_details table  

22. **If Node to Check for Empty drive_file_id**  
    - Name: `If`  
    - Condition: If `drive_file_id` is empty, route to `do nothing1`; else continue.  

23. **No Operation Node**  
    - Name: `do nothing1`  

24. **Airtable Search Node for Products to Post**  
    - Name: `get_analyzed_row2`  
    - Base/Table: product_details table  
    - Filter: `{product_status} = 'generated'`  

25. **Shopify Node to Create Product**  
    - Name: `Create a product`  
    - Credentials: Shopify OAuth Access Token  
    - Fields: title, handle, body_html, product_type from Airtable JSON  

26. **Airtable Update Node to Mark Product Posted**  
    - Name: `update_product_update_status`  
    - Base/Table: product_details  
    - Matching: `product_id`  
    - Update `product_status` to "posted"  

27. **Airtable Create Node to Update drive_file_id**  
    - Name: `update_drive_file_id`  
    - Base/Table: product_details (optional for linking)  

28. **Wait Node**  
    - Name: `Wait`  
    - Purpose: Synchronize workflow and trigger collection fetching  

29. **Connect all nodes as per the workflow graph**  
    - Follow connections in the design to ensure correct data flow and batch processing.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow leverages Google Gemini AI and OpenAI GPT-4o for advanced image analysis and product content generation.                             | AI Model integration details                                                                    |
| Airtable is the central data store for raw images, analyzed metadata, and generated product details, facilitating state and batch management.     | Airtable base: shopify_digital_product                                                          |
| Google Drive is used for storing and accessing source image files referenced by Airtable IDs.                                                      | Google Drive OAuth2 credentials required                                                       |
| Shopify API access requires OAuth2 credentials with permissions to read collections and create products.                                           | Shopify Admin API 2025-10 version used                                                           |
| Batch processing nodes (`SplitInBatches`, `Limit`) prevent hitting API rate limits and allow scalable processing.                                 | Important for large image/product sets                                                          |
| The workflow includes rich prompt engineering to minimize hallucinations and enforce strict JSON output from AI models.                          | Prompt templates embedded in LangChain nodes                                                    |
| Error handling edge cases include: API rate limits, token expirations, empty or malformed data fields, and network timeouts.                      | Recommended to add retry and alert mechanisms externally                                        |
| Author Contact: Manish Kumar, expert in Shopify automation and custom app development. Email: manipritraj@gmail.com, Phone: +91-9334888899.         | For consulting or custom workflow adaptation                                                    |

---

**Disclaimer:** The provided text is exclusively derived from a complete n8n workflow automation. The processing respects all applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.