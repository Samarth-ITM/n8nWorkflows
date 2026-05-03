Create Stylized Product Photography with Airtable & Gemini Nano Banana

https://n8nworkflows.xyz/workflows/create-stylized-product-photography-with-airtable---gemini-nano-banana-10250


# Create Stylized Product Photography with Airtable & Gemini Nano Banana

### 1. Workflow Overview

This workflow automates the creation of stylized product photography by combining original product images with creative templates using Airtable as the data source and Google Gemini (Nano Banana) for AI-driven image editing. It is designed for e-commerce use cases where multiple stylized images need to be generated for products based on predefined templates containing textual prompts and reference images.

**Key logical blocks:**

- **1.1 Input Reception & Job Initialization:** Receive a job request via webhook, fetch job details from Airtable, and mark the job as "in progress."
- **1.2 Product and Template Data Retrieval:** Retrieve product images and templates (including their reference images) specified in the job.
- **1.3 Reference Image Handling:** Download all reference images linked to each template.
- **1.4 Image Combination and Generation:** Combine each product image with each template and generate new stylized images using Google Gemini, selecting the generation method based on the number of reference images.
- **1.5 Result Storage and Job Completion:** Upload generated images to Airtable Results table, update records accordingly, and mark the job as "done."

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Job Initialization

- **Overview:** Triggered by an external webhook carrying a job ID; fetches the job record from Airtable and updates its status to "In progress."
- **Nodes involved:**  
  - Webhook  
  - parameters (Set node)  
  - Get Job (Airtable)  
  - Update job status (Airtable)  
  - Clean up (Set node)  
- **Node Details:**

  - **Webhook**
    - Type: Webhook (HTTP trigger)  
    - Configuration: Path set to `736029f4-2d85-409f-8841-1ca9a010e385`  
    - Inputs: External HTTP request with query parameter `recordId` (job ID)  
    - Outputs: Passes query parameters downstream  
    - Edge cases: Missing or invalid job ID in request could cause failure  

  - **parameters (Set)**
    - Type: Set  
    - Configuration: Sets static parameters for Airtable Base ID and Result Image field ID  
    - Inputs: From Webhook node  
    - Outputs: Parameters object used to configure Airtable nodes  
    - Edge cases: Incorrect base or field IDs will cause Airtable errors  

  - **Get Job (Airtable)**
    - Type: Airtable node (read)  
    - Configuration: Fetch record by ID (from webhook query param) in "Jobs" table  
    - Credentials: Airtable Personal Access Token  
    - Inputs: From parameters node  
    - Outputs: Full job record including product images and templates fields  
    - Edge cases: Record not found or API rate limits/errors  

  - **Update job status (Airtable)**
    - Type: Airtable node (update)  
    - Configuration: Updates the same job record, setting `Status` field to "In progress"  
    - Inputs: Output of Get Job  
    - Credentials: Airtable credentials  
    - Edge cases: Update failure or permission issues  

  - **Clean up (Set)**
    - Type: Set  
    - Configuration: Extracts arrays of product IDs and template IDs, plus job ID for downstream use  
    - Inputs: Output of Get Job (job record JSON)  
    - Outputs: Structured variables for further processing  
    - Edge cases: Missing or empty products/templates fields  

---

#### Block 1.2: Product and Template Data Retrieval

- **Overview:** Split and fetch detailed records for each product and template referenced in the job.
- **Nodes involved:**  
  - Split products (SplitOut)  
  - Get products (Airtable)  
  - Edit products (Set)  
  - Download product image (HTTP Request)  
  - Split templates (SplitOut)  
  - Get templates (Airtable)  
  - Edit templates (Set)  
  - Loop over templates (SplitInBatches)  
  - Merge templates data (Merge - combine)  
- **Node Details:**

  - **Split products**
    - Type: SplitOut  
    - Configuration: Splits the array of product IDs to process each individually  
    - Inputs: products array from Clean up  
    - Outputs: Individual product IDs per item  
    - Edge cases: Empty array input  

  - **Get products (Airtable)**
    - Type: Airtable read node  
    - Configuration: Fetch product record by ID  
    - Credentials: Airtable  
    - Inputs: product ID from Split products  
    - Outputs: Product record including image URL  
    - Edge cases: Missing product record, API errors  

  - **Edit products (Set)**
    - Type: Set  
    - Configuration: Extract product ID and product image URL from record  
    - Inputs: Get products output  
    - Outputs: JSON with product_id and product_image_url fields  

  - **Download product image (HTTP Request)**
    - Type: HTTP Request  
    - Configuration: Downloads product image binary data from URL  
    - Inputs: product_image_url from Edit products  
    - Outputs: Binary data of product image for downstream generation  
    - Edge cases: Image URL inaccessible, HTTP errors, timeouts  

  - **Split templates**
    - Type: SplitOut  
    - Configuration: Splits array of template IDs into individual items  
    - Inputs: templates array from Clean up  
    - Outputs: Single template IDs per item  

  - **Get templates (Airtable)**
    - Type: Airtable read node  
    - Configuration: Fetch template record by ID  
    - Inputs: template ID from Split templates  
    - Outputs: Template record including prompt text and reference images array  

  - **Edit templates (Set)**
    - Type: Set  
    - Configuration: Extracts template_id, prompt text, and reference_images array  
    - Inputs: Get templates output  

  - **Loop over templates (SplitInBatches)**
    - Type: SplitInBatches  
    - Configuration: Processes templates one by one in batches (default batch size)  
    - Inputs: Edit templates output  

  - **Merge templates data (Merge)**
    - Type: Merge (combine)  
    - Configuration: Combines data streams by position for parallel processing of templates  

---

#### Block 1.3: Reference Image Handling

- **Overview:** For each template, download all reference images required for AI generation.
- **Nodes involved:**  
  - Split reference images (SplitOut)  
  - Get reference image (Airtable)  
  - Edit reference image (Set)  
  - Download reference image (HTTP Request)  
  - Aggregate (Aggregate)  
- **Node Details:**

  - **Split reference images**
    - Type: SplitOut  
    - Configuration: Splits the reference_images array from each template to individual IDs  

  - **Get reference image (Airtable)**
    - Type: Airtable read node  
    - Configuration: Fetch reference image record by ID  
    - Inputs: reference image ID from Split reference images  
    - Outputs: Reference image record including image URL  

  - **Edit reference image (Set)**
    - Type: Set  
    - Configuration: Extracts image ID and image URL fields  

  - **Download reference image (HTTP Request)**
    - Type: HTTP Request  
    - Configuration: Downloads binary data of the reference image from URL  

  - **Aggregate (Aggregate)**
    - Type: Aggregate  
    - Configuration: Aggregates multiple downloaded reference images back into a single array for the template  
    - Includes binaries to preserve image data  

---

#### Block 1.4: Image Combination and Generation

- **Overview:** Combine each product image with each template and generate stylized images using Google Gemini Nano Banana, selecting generation method depending on number of reference images.
- **Nodes involved:**  
  - All combinations (Merge) - combines product and template data  
  - Switch - routes to generation node based on count of reference images  
  - Generate with 1 ref (Google Gemini)  
  - Generate with 2 refs (Google Gemini)  
  - Generate with 3 refs (Google Gemini)  
  - Merge (Merge) - merges generation results  
  - Extract from File (ExtractFromFile)  
  - Collect variables (Set)  
- **Node Details:**

  - **All combinations (Merge)**
    - Type: Merge (combineAll)  
    - Configuration: Combines all product-template pairs to generate all permutations  
    - Inputs: Edited product data and merged template data  

  - **Switch**
    - Type: Switch  
    - Configuration: Checks the number of reference images in the combined data  
    - Routes:
      - 1 reference image → "Generate with 1 ref" node  
      - 2 reference images → "Generate with 2 refs" node  
      - 3 or more → "Generate with 3 refs" node  
    - Edge cases: Missing or malformed reference images array  

  - **Generate with 1 ref / 2 refs / 3 refs (Google Gemini)**
    - Type: Google Gemini (Langchain node) for image editing  
    - Configuration:
      - Inputs: Binary product image data plus 1, 2, or 3 binary reference images  
      - Prompt text from template  
      - Outputs edited image as binary property `edited`  
    - Credentials: Google Gemini API key  
    - Edge cases: API limits, invalid prompt, image processing errors  

  - **Merge**
    - Type: Merge (numberInputs=3)  
    - Configuration: Merges results from all three generation nodes back into a single stream  

  - **Extract from File (ExtractFromFile)**
    - Type: ExtractFromFile  
    - Configuration: Converts binary edited image file to base64 string and preserves source JSON  
    - Inputs: Binary `edited` property from Gemini generate nodes  

  - **Collect variables (Set)**
    - Type: Set  
    - Configuration: Collects base64 output, MIME type, template ID, product ID, and job ID for uploading and record creation  

---

#### Block 1.5: Result Storage and Job Completion

- **Overview:** Create new records in Airtable "Results" table for each generated image, upload images as attachments, and update job status to "Done."
- **Nodes involved:**  
  - Create Result record (Airtable)  
  - Upload attachment (HTTP Request)  
  - Update job status (done) (Airtable)  
- **Node Details:**

  - **Create Result record (Airtable)**
    - Type: Airtable create node  
    - Configuration: Creates a new record in the "Results" table with references to Job, Template, Product Image, timestamp, and status "Pending"  
    - Inputs: Variables from Collect variables node  
    - Credentials: Airtable  

  - **Upload attachment (HTTP Request)**
    - Type: HTTP Request POST  
    - Configuration: Uploads the generated image (base64) to the attachment field of the newly created Result record  
    - Uses Airtable API endpoint with base ID, record ID, and attachment field ID  
    - Authentication: Airtable token  
    - Edge cases: Upload failure, large file size, API rate limiting  

  - **Update job status (done) (Airtable)**
    - Type: Airtable update node  
    - Configuration: Updates the job record status to "Done" after all uploads complete  

---

### 3. Summary Table

| Node Name              | Node Type                  | Functional Role                                  | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                                               |
|------------------------|----------------------------|-------------------------------------------------|----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                | Webhook                    | Receive job ID to trigger workflow              | -                          | parameters                  |                                                                                                                                           |
| parameters             | Set                        | Set static parameters (Airtable base ID, fields)| Webhook                    | Get Job                     | ## Configure ➤ set Airtable base ID ➤ set Airtable field ID for generated images                                                            |
| Get Job                | Airtable                   | Fetch Job record by ID                           | parameters                 | Update job status, Clean up  | ## Step 1. Fetch Job record by ID Also set Job's status to 'in progress' in Airtable                                                       |
| Update job status      | Airtable                   | Update Job status to "In progress"               | Get Job                    | Clean up                    | ## Step 1. Fetch Job record by ID Also set Job's status to 'in progress' in Airtable                                                       |
| Clean up               | Set                        | Extract products, templates arrays, job ID       | Get Job                    | Split products, Split templates |                                                                                                                                           |
| Split products         | SplitOut                   | Split product IDs for individual processing      | Clean up                   | Get products                | ## Step 2a. Get original product image - Fetch Product records specified in Job - Download product images                                  |
| Get products           | Airtable                   | Fetch product record by ID                        | Split products             | Edit products               | ## Step 2a. Get original product image - Fetch Product records specified in Job - Download product images                                  |
| Edit products          | Set                        | Extract product ID and image URL                  | Get products               | Download product image      | ## Step 2a. Get original product image - Fetch Product records specified in Job - Download product images                                  |
| Download product image | HTTP Request               | Download product image binary                      | Edit products              | All combinations            | ## Step 2a. Get original product image - Fetch Product records specified in Job - Download product images                                  |
| Split templates        | SplitOut                   | Split template IDs for individual processing      | Clean up                   | Get templates               | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Get templates          | Airtable                   | Fetch template record by ID                        | Split templates            | Edit templates              | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Edit templates         | Set                        | Extract template ID, prompt, reference images     | Get templates              | Loop over templates, Merge templates data | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Loop over templates    | SplitInBatches             | Process templates one by one                       | Edit templates             | Merge templates data, Split reference images | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Merge templates data   | Merge (combine)            | Combine template data streams                      | Loop over templates        | All combinations            | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Split reference images | SplitOut                   | Split reference images array                        | Loop over templates        | Get reference image         | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Get reference image    | Airtable                   | Fetch reference image record by ID                 | Split reference images     | Edit reference image        | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Edit reference image   | Set                        | Extract reference image ID and URL                 | Get reference image        | Download reference image    | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Download reference image | HTTP Request             | Download reference image binary                     | Edit reference image       | Aggregate                   | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| Aggregate              | Aggregate                  | Aggregate downloaded reference images into array  | Download reference image   | Loop over templates         | ## Step 2b. Get templates with reference images - Fetch Template records specified in Job (each template contains prompt and references)  |
| All combinations       | Merge (combineAll)         | Create all product-template combinations           | Download product image, Merge templates data | Switch                     | ## Step 3. Generate new product images - Create all combinations of original images and templates - Pick generation node based on count   |
| Switch                 | Switch                     | Select generation node based on number of references | All combinations           | Generate with 1/2/3 refs    | ## Step 3. Generate new product images - Create all combinations of original images and templates - Pick generation node based on count   |
| Generate with 1 ref    | Google Gemini (Langchain)  | Generate image with one reference image            | Switch (output 1)          | Merge                      | ## Step 3. Generate new product images - Create all combinations of original images and templates - Pick generation node based on count   |
| Generate with 2 refs   | Google Gemini (Langchain)  | Generate image with two reference images           | Switch (output 2)          | Merge                      | ## Step 3. Generate new product images - Create all combinations of original images and templates - Pick generation node based on count   |
| Generate with 3 refs   | Google Gemini (Langchain)  | Generate image with three or more reference images | Switch (output 3)          | Merge                      | ## Step 3. Generate new product images - Create all combinations of original images and templates - Pick generation node based on count   |
| Merge                  | Merge                      | Merge generation outputs from all generation nodes | Generate with 1/2/3 refs  | Extract from File           | ## Step 3. Generate new product images - Create all combinations of original images and templates - Pick generation node based on count   |
| Extract from File      | ExtractFromFile            | Convert edited image binary to base64 string       | Merge                      | Collect variables           | ## Step 4. Create new Result records in Airtable - For each generated image, create new Result record - Upload generated image             |
| Collect variables      | Set                        | Collect variables required for record creation and upload | Extract from File          | Create Result record        | ## Step 4. Create new Result records in Airtable - For each generated image, create new Result record - Upload generated image             |
| Create Result record   | Airtable                   | Create new Result record in Airtable                | Collect variables          | Upload attachment           | ## Step 4. Create new Result records in Airtable - For each generated image, create new Result record - Upload generated image             |
| Upload attachment      | HTTP Request               | Upload generated image file to Airtable attachment field | Create Result record       | Update job status (done)    | ## Step 4. Create new Result records in Airtable - For each generated image, create new Result record - Upload generated image             |
| Update job status (done) | Airtable                 | Mark Job record as "Done" after processing          | Upload attachment          | -                          | ## Step 4. Create new Result records in Airtable - For each generated image, create new Result record - Upload generated image             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node:**  
   - Type: Webhook  
   - Path: `736029f4-2d85-409f-8841-1ca9a010e385`  
   - Purpose: Receive HTTP requests containing query param `recordId` (job ID).  

2. **Create Set node "parameters":**  
   - Type: Set  
   - Assign static values:  
     - `airtableBaseId` = your Airtable Base ID (e.g., `app3umoCF1t88Mg0q`)  
     - `airtableResultImageFieldId` = field ID for attachment in "Results" table (e.g., `fldzhCsleyyBrPyKj`)  
   - Connect from Webhook node.  

3. **Create Airtable "Get Job" node:**  
   - Type: Airtable (read)  
   - Credentials: Airtable Personal Access Token  
   - Base ID: `={{ $json.airtableBaseId }}` (from parameters)  
   - Table: "Jobs"  
   - Record ID: `={{ $('Webhook').item.json.query.recordId }}`  
   - Connect from parameters node.  

4. **Create Airtable "Update job status" node:**  
   - Type: Airtable (update)  
   - Credentials: Airtable  
   - Base ID: `={{ $('parameters').item.json.airtableBaseId }}`  
   - Table: "Jobs"  
   - Update record ID: `={{ $json.id }}` (from Get Job)  
   - Update field "Status" to "In progress"  
   - Connect from Get Job node.  

5. **Create Set node "Clean up":**  
   - Type: Set  
   - Assign:  
     - `job_id` = `={{ $json.id }}`  
     - `products` = `={{ $json["Product Images"] }}` (array of product record IDs)  
     - `templates` = `={{ $json.Templates }}` (array of template record IDs)  
   - Connect from Get Job node.  

6. **Create SplitOut node "Split products":**  
   - Type: SplitOut  
   - Field to split out: `products`  
   - Connect from Clean up node.  

7. **Create Airtable "Get products" node:**  
   - Type: Airtable (read)  
   - Credentials: Airtable  
   - Base ID: `={{ $('parameters').item.json.airtableBaseId }}`  
   - Table: "Product Images"  
   - Record ID: `={{ $json.products }}` (from Split products)  
   - Connect from Split products node.  

8. **Create Set node "Edit products":**  
   - Type: Set  
   - Assign:  
     - `product_id` = `={{ $json.id }}`  
     - `product_image_url` = `={{ $json.Image[0].url }}`  
   - Connect from Get products node.  

9. **Create HTTP Request node "Download product image":**  
   - Type: HTTP Request  
   - URL: `={{ $json.product_image_url }}`  
   - Response format: File (binary)  
   - Output property: `product_data`  
   - Connect from Edit products node.  

10. **Create SplitOut node "Split templates":**  
    - Type: SplitOut  
    - Field to split out: `templates`  
    - Connect from Clean up node.  

11. **Create Airtable "Get templates" node:**  
    - Type: Airtable (read)  
    - Credentials: Airtable  
    - Base ID: `={{ $('parameters').item.json.airtableBaseId }}`  
    - Table: "Templates"  
    - Record ID: `={{ $json.templates }}`  
    - Connect from Split templates node.  

12. **Create Set node "Edit templates":**  
    - Type: Set  
    - Assign:  
      - `template_id` = `={{ $json.id }}`  
      - `prompt` = `={{ $json["Text Prompt"] }}`  
      - `reference_images` = `={{ $json["Reference Images"] }}`  
    - Connect from Get templates node.  

13. **Create SplitInBatches node "Loop over templates":**  
    - Type: SplitInBatches  
    - Connect from Edit templates node.  

14. **Create SplitOut node "Split reference images":**  
    - Type: SplitOut  
    - Field to split out: `reference_images`  
    - Connect from Loop over templates node.  

15. **Create Airtable "Get reference image" node:**  
    - Type: Airtable (read)  
    - Credentials: Airtable  
    - Base ID: `={{ $('parameters').item.json.airtableBaseId }}`  
    - Table: "Reference Images"  
    - Record ID: `={{ $json.reference_images }}`  
    - Connect from Split reference images node.  

16. **Create Set node "Edit reference image":**  
    - Type: Set  
    - Assign:  
      - `image_id` = `={{ $json.id }}`  
      - `image_url` = `={{ $json.Image[0].url }}`  
    - Connect from Get reference image node.  

17. **Create HTTP Request node "Download reference image":**  
    - Type: HTTP Request  
    - URL: `={{ $json.image_url }}`  
    - Connect from Edit reference image node.  

18. **Create Aggregate node "Aggregate":**  
    - Type: Aggregate  
    - Aggregate all item data including binaries  
    - Destination field: `reference_images`  
    - Connect from Download reference image node.  

19. **Connect Aggregate node back to Loop over templates node (second output) to continue processing each template with reference images aggregated.**

20. **Create Merge node "Merge templates data":**  
    - Type: Merge (combine)  
    - Combine by position  
    - Connect from Loop over templates node (first output) and Aggregate node.  

21. **Create Merge node "All combinations":**  
    - Type: Merge (combineAll)  
    - Connect from Download product image node and Merge templates data node.  

22. **Create Switch node "Switch":**  
    - Type: Switch  
    - Rules based on length of `reference_images` array:  
      - length = 1 → output 1  
      - length = 2 → output 2  
      - length >= 3 → output 3  
    - Connect from All combinations node.  

23. **Create three Google Gemini nodes for generation:**  
    - "Generate with 1 ref": inputs product image binary + 1 reference image binary, prompt text; output edited image binary  
    - "Generate with 2 refs": inputs product image binary + 2 reference images binary  
    - "Generate with 3 refs": inputs product image binary + 3 reference images binary  
    - Credentials: Google Gemini API key  
    - Connect outputs of Switch node accordingly.  

24. **Create Merge node "Merge":**  
    - Merge numberInputs=3  
    - Connect outputs of all three Gemini generate nodes.  

25. **Create ExtractFromFile node "Extract from File":**  
    - Convert binary edited image to base64 string under `output_base64`  
    - Connect from Merge node.  

26. **Create Set node "Collect variables":**  
    - Assign variables:  
      - `outputBase64` = `={{ $json.output_base64 }}`  
      - `mimeType` = `={{ $json.mimeType }}`  
      - `templateId` = `={{ $('All combinations').item.json.template_id }}`  
      - `productId` = `={{ $('All combinations').item.json.product_id }}`  
      - `jobId` = `={{ $('Get Job').item.json.id }}`  
    - Connect from Extract from File node.  

27. **Create Airtable "Create Result record" node:**  
    - Table: "Results"  
    - Fields:  
      - Jobs: array with jobId  
      - Template: array with templateId  
      - Product Image: array with productId  
      - Timestamp: current time (`{{$now}}`)  
      - Status: "Pending"  
    - Connect from Collect variables node.  

28. **Create HTTP Request node "Upload attachment":**  
    - POST to Airtable content API endpoint:  
      `https://content.airtable.com/v0/{{ airtableBaseId }}/{{ recordId }}/{{ airtableResultImageFieldId }}/uploadAttachment`  
    - Body parameters: contentType, file (base64), filename  
    - Authentication: Airtable token  
    - Connect from Create Result record node.  

29. **Create Airtable "Update job status (done)" node:**  
    - Update job record, setting Status field to "Done"  
    - Connect from Upload attachment node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| AI Product Photography using Airtable and Gemini (Nano Banana). This workflow automates generating stylized product photos for e-commerce by combining real product shots with reusable templates. It uses Google Gemini for image editing and Airtable as a database. Tables needed include Product Images, Reference Images, Templates, Jobs, and Results. Requirements: Airtable PAT, Google Gemini API key. | Sticky Note on "Sticky Note6" node in workflow summary                                                              |
| Workflow setup instructions include configuring Airtable base and field IDs, adding credentials for Airtable and Google Gemini, and ensuring tables exist with proper fields such as prompts and reference images.                                                                                                                                                                           | Sticky Note on "Sticky Note4" and related setup nodes                                                               |
| Airtable API limits and attachment upload constraints should be considered. Generated images are uploaded as base64-encoded attachments to Airtable’s attachment fields.                                                                                                                                                                                                                  | Implicit in Upload attachment node configuration                                                                     |
| Google Gemini Nano Banana is used with Langchain integration nodes for image editing, supporting up to three reference images.                                                                                                                                                                                                                                                            | Nodes "Generate with 1/2/3 refs"                                                                                     |
| The workflow is structured to handle batch processing and dynamic routing based on template complexity and reference image count.                                                                                                                                                                                                                                                         | Observed in SplitInBatches, Switch, and Merge nodes                                                                   |
| For troubleshooting, ensure that all API credentials are valid, and monitor for Airtable API rate limits or Google Gemini quota limits. Validate that all referenced record IDs exist in Airtable before triggering the workflow.                                                                                                                                                              | General best practices                                                                                               |
| Airtable table URLs used in nodes for reference: Jobs (https://airtable.com/app3umoCF1t88Mg0q/tblN7ikAWki5s5jcd), Product Images (https://airtable.com/app3umoCF1t88Mg0q/tblZeNi5MVPOEj7Hp), Templates (https://airtable.com/app3umoCF1t88Mg0q/tblGJpUnqFpLMlVFe), Reference Images (https://airtable.com/app3umoCF1t88Mg0q/tbl4STKAfNWhTsJjl), Results (https://airtable.com/app3umoCF1t88Mg0q/tblSoB3qTeeKXjsov) | Sticky Note content and node cachedResultUrl parameters                                                             |

---

**Disclaimer:** The text analyzed and documented is exclusively from an automated workflow created with n8n. The workflow respects all content policies and contains no illegal or protected content. Only public and legally permissible data is processed.