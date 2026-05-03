Scrape & Import Shoe Products to Shopify with BrowserAct (with Variants & Images)

https://n8nworkflows.xyz/workflows/scrape---import-shoe-products-to-shopify-with-browseract--with-variants---images--9946


# Scrape & Import Shoe Products to Shopify with BrowserAct (with Variants & Images)

### 1. Workflow Overview

This workflow automates the end-to-end process of scraping shoe product data from arbitrary URLs and importing the data as fully detailed Shopify products, including variants and images. It is optimized for Nike shoes but adaptable to other products with similar data structure. It integrates multiple services and API calls to deliver a seamless product import pipeline.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger and retrieval of product URLs from Google Sheets.
- **1.2 Product Scraping:** Iterative scraping of each product URL using BrowserAct's scraping API workflow.
- **1.3 Data Parsing & Transformation:** Processing raw scraped data into structured JSON for Shopify.
- **1.4 Shopify Base Product Creation:** Creating the main product entry in Shopify with title, description, and tags.
- **1.5 Adding Product Options:** Adding product options (e.g., "Shoe Size") to the Shopify product.
- **1.6 Creating Variants:** Creating individual product variants for each available size.
- **1.7 Uploading Images:** Uploading all product images to the Shopify product.
- **1.8 Notifications:** Sending a Slack alert upon completion of processing each product.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow via manual trigger and obtains the list of product URLs from a Google Sheet.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `Get List`  
  - `Loop Over Items`
  
- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start workflow manually.  
    - Input: None  
    - Output: Triggers the next node (`Get List`)  
    - Edge Cases: None typical; manual trigger means no scheduled automation errors.  

  - **Get List**  
    - Type: Google Sheets  
    - Role: Reads product page URLs from a specified Google Sheet and worksheet.  
    - Key Configurations: Document ID and Sheet GID point to a sheet with a column named `Product Link`.  
    - Input: Trigger output  
    - Output: List of URLs to process.  
    - Credentials: Google Sheets OAuth2 with appropriate read access.  
    - Edge Cases: Sheet missing or misconfigured, missing `Product Link` column, OAuth token expiry/errors.  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes each product URL one by one to control flow and API calls.  
    - Input: Array of product links from Google Sheets.  
    - Output: Sends each URL individually downstream for scraping and processing.  
    - Edge Cases: Large lists may require pagination or throttling; reset option is disabled to preserve state.  

---

#### 2.2 Product Scraping

- **Overview:** For each product URL, invoke BrowserAct workflow to scrape product details asynchronously and retrieve results.
- **Nodes Involved:**  
  - `Run a workflow` (BrowserAct)  
  - `Get workflow Data` (BrowserAct)  

- **Node Details:**

  - **Run a workflow**  
    - Type: BrowserAct (Community Node)  
    - Role: Initiates a BrowserAct scraping workflow with the product URL as input.  
    - Key Configurations: Uses BrowserAct workflow ID `"57519762985270138"` and passes the URL via parameter `Target_Product_Url`.  
    - Input: Single product URL from Loop Over Items  
    - Output: Task ID for the scraping job.  
    - Credentials: BrowserAct API key with access to the scraping workflow.  
    - Edge Cases: API limits, workflow ID changes, network errors, invalid URLs.  

  - **Get workflow Data**  
    - Type: BrowserAct  
    - Role: Polls BrowserAct for scraping results, waits for completion up to 900 seconds.  
    - Input: Task ID from previous node  
    - Output: Raw scraped product data as JSON string.  
    - Credentials: Same BrowserAct API key.  
    - Edge Cases: Timeout exceeding max wait time, incomplete data, API errors.  

---

#### 2.3 Data Parsing & Transformation

- **Overview:** Converts raw scraped product data (stringified JSON) into structured JSON objects with separate arrays for sizes and images.
- **Nodes Involved:**  
  - `Parse Data` (Code node)  

- **Node Details:**

  - **Parse Data**  
    - Type: Code (JavaScript)  
    - Role: Parses the BrowserAct output string, splits sizes and images into arrays of objects, and reformats product data for Shopify.  
    - Key Expressions:  
      - Parses input JSON string: `JSON.parse($input.first().json.output.string)`  
      - Splits `Size` string by commas into array of size objects.  
      - Splits `Images` string by `||` delimiter into array of image URL objects.  
    - Input: Raw JSON string from BrowserAct scraping result.  
    - Output: Structured JSON with fields: Title, Price, Description, Category, sizes (array), images (array).  
    - Edge Cases: Malformed JSON strings, missing fields, empty size or image strings.  

---

#### 2.4 Shopify Base Product Creation

- **Overview:** Creates a new Shopify product with main details and tags extracted from the parsed data.
- **Nodes Involved:**  
  - `Create a product` (Shopify)  
  - `Set Option` (Set Node)  
  - `Add Option` (HTTP Request)  

- **Node Details:**

  - **Create a product**  
    - Type: Shopify node  
    - Role: Creates the base product using title, description, and category tags.  
    - Key Configurations:  
      - Product resource creation with fields: Title, Body HTML (description), Tags (category), Vendor set as "NIKE".  
    - Input: Parsed product JSON from `Parse Data`.  
    - Output: Shopify product object including product ID.  
    - Credentials: Shopify access token with write permissions.  
    - Edge Cases: Shopify API rate limits, invalid product data, permission errors.  

  - **Set Option**  
    - Type: Set node  
    - Role: Defines the product option name as "Shoe Size" for the upcoming API call.  
    - Configuration: Static assignment of string "Shoe Size" to field `Option`.  
    - Input: Output from `Create a product`.  
    - Output: Data enriched with option name.  

  - **Add Option**  
    - Type: HTTP Request  
    - Role: Updates the newly created Shopify product to add the product option "Shoe Size".  
    - Configuration: PUT request to Shopify Products API endpoint with product ID, setting options array with the option name.  
    - Input: Output from `Set Option`.  
    - Output: Updated product data.  
    - Credentials: Shopify access token.  
    - Edge Cases: API failures, invalid option names, Shopify API version compatibility (uses 2025-01).  

---

#### 2.5 Creating Variants

- **Overview:** For each shoe size, create a product variant in Shopify linked to the base product.
- **Nodes Involved:**  
  - `Merge` (SQL Join)  
  - `Split Shoe Size` (SplitOut)  
  - `Add Variant` (HTTP Request)  

- **Node Details:**

  - **Merge**  
    - Type: Merge (combineBySql)  
    - Role: Joins the parsed product data with Shopify product ID to consolidate data for variant creation.  
    - Configuration: SQL LEFT JOIN query combining input from `Parse Data` and `Create a product`.  
    - Input: Parsed product data and Shopify product data.  
    - Output: Combined data with product ID accessible.  

  - **Split Shoe Size**  
    - Type: SplitOut  
    - Role: Splits the array of sizes into individual items for one-by-one variant creation.  
    - Configuration: Splits field `sizes`, includes `input1.product->id` for reference.  
    - Input: Merged product data.  
    - Output: Individual size records for variant creation.  

  - **Add Variant**  
    - Type: HTTP Request  
    - Role: Creates a variant for each size via Shopify API POST to the variants endpoint.  
    - Configuration:  
      - URL dynamically constructed using product ID.  
      - JSON body includes option1 (size), price (from merged data), SKU constructed with product ID and size string sanitized.  
    - Input: Single size object from `Split Shoe Size`.  
    - Output: Shopify variant creation response.  
    - OnError: Continue workflow on error to avoid interruption (e.g., duplicate variant errors).  
    - Credentials: Shopify access token.  
    - Edge Cases: SKU conflicts, price missing, API limits.  

---

#### 2.6 Uploading Images

- **Overview:** Upload each product image URL to Shopify associated with the product.
- **Nodes Involved:**  
  - `Split Images` (SplitOut)  
  - `Add Images` (HTTP Request)  
  - `Merge1` (Merge)  

- **Node Details:**

  - **Split Images**  
    - Type: SplitOut  
    - Role: Splits the array of image URLs into individual items for uploading.  
    - Configuration: Splits field `images`, includes `input1.product->id`.  
    - Input: Output from `Merge` node.  
    - Output: Individual image records.  

  - **Add Images**  
    - Type: HTTP Request  
    - Role: Uploads each image to Shopify using the images API endpoint.  
    - Configuration:  
      - URL uses product ID dynamically.  
      - JSON body contains image source URL.  
    - Input: Single image object from `Split Images`.  
    - Output: Shopify image upload response.  
    - Credentials: Shopify access token.  
    - Edge Cases: Invalid URLs, API limits, image size restrictions.  

  - **Merge1**  
    - Type: Merge  
    - Role: Combines outputs from `Add Variant` and `Add Images` to collate results.  
    - Input: Outputs from variant and image creation nodes.  
    - Output: Combined data for further processing or notification.  

---

#### 2.7 Notifications

- **Overview:** Sends a Slack message notification after processing each product.
- **Nodes Involved:**  
  - `Send a message` (Slack)  

- **Node Details:**

  - **Send a message**  
    - Type: Slack  
    - Role: Posts a message "The Products Updated" to a specified Slack channel.  
    - Configuration: Uses Slack OAuth2 credentials, posts to channel ID `C09KLV9DJSX`.  
    - Input: Triggered once per product due to batch processing.  
    - Edge Cases: Slack API auth errors, channel permissions.  

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                          | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                                                                                                                                           |
|-----------------------------|---------------------------|----------------------------------------|----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Entry point to start workflow manually | None                       | Get List                    |                                                                                                                                                                                                                                     |
| Get List                    | Google Sheets             | Reads product URLs from Google Sheet   | When clicking ‘Execute workflow’ | Loop Over Items             |                                                                                                                                                                                                                                     |
| Loop Over Items             | SplitInBatches            | Processes URLs one at a time            | Get List                   | Send a message, Run a workflow | Sticky Note - Scrape & Transform: Explains looping through URLs and scraping per product.                                                                                                                                             |
| Run a workflow              | BrowserAct                | Starts scraping workflow with URL      | Loop Over Items             | Get workflow Data           | Sticky Note - Scrape & Transform                                                                                                                                                                                                     |
| Get workflow Data           | BrowserAct                | Polls for scraping results              | Run a workflow              | Parse Data                  | Sticky Note - Scrape & Transform                                                                                                                                                                                                     |
| Parse Data                  | Code                      | Transforms raw scraping data into structured JSON | Get workflow Data           | Create a product, Merge     | Sticky Note - Scrape & Transform                                                                                                                                                                                                     |
| Create a product            | Shopify                   | Creates base Shopify product             | Parse Data                  | Set Option                  | Sticky Note - Create Base Product                                                                                                                                                                                                    |
| Set Option                 | Set                       | Sets product option name "Shoe Size"    | Create a product            | Add Option                  | Sticky Note - Create Base Product                                                                                                                                                                                                    |
| Add Option                 | HTTP Request (Shopify API) | Adds product option to Shopify product  | Set Option                  | Merge                       | Sticky Note - Create Base Product                                                                                                                                                                                                    |
| Merge                      | Merge (combineBySql)      | Joins Shopify product ID with scraped data | Add Option, Parse Data      | Split Shoe Size, Split Images | Sticky Note - Add Variants & Images: Describes merging data before splitting for variants and images.                                                                                                                               |
| Split Shoe Size            | SplitOut                  | Splits sizes array into individual items | Merge                      | Add Variant                 | Sticky Note - Add Variants & Images                                                                                                                                                                                                 |
| Add Variant                | HTTP Request (Shopify API) | Creates Shopify product variants for sizes | Split Shoe Size            | Merge1                      | Sticky Note - Add Variants & Images                                                                                                                                                                                                 |
| Split Images               | SplitOut                  | Splits images array into individual items | Merge                      | Add Images                  | Sticky Note - Add Variants & Images                                                                                                                                                                                                 |
| Add Images                 | HTTP Request (Shopify API) | Uploads product images to Shopify        | Split Images                | Merge1                      | Sticky Note - Add Variants & Images                                                                                                                                                                                                 |
| Merge1                     | Merge                     | Combines variant and image creation results | Add Variant, Add Images     | Loop Over Items             |                                                                                                                                                                                                                                     |
| Send a message             | Slack                     | Sends Slack notification per product    | Loop Over Items             | None                       | Sticky Note - Scrape & Transform: Sends alert to Slack channel                                                                                                                                                                      |
| Sticky Note - Intro        | StickyNote                | Documentation and overview               | None                       | None                       | Provides workflow introduction, requirements, and help links.                                                                                                                                                                       |
| Sticky Note - How to Use   | StickyNote                | Documentation on setting up and usage   | None                       | None                       | Instructions for credential setup and usage.                                                                                                                                                                                        |
| Sticky Note - Need Help    | StickyNote                | Documentation with helpful links        | None                       | None                       | Contains video tutorials and resources for BrowserAct and n8n integration.                                                                                                                                                         |
| Sticky Note - Scrape & Transform | StickyNote            | Documents scraping and data parsing block | None                       | None                       | Describes scraping loop, data transformation, and Slack notification.                                                                                                                                                               |
| Sticky Note - Create Base Product | StickyNote            | Documents base product creation block   | None                       | None                       | Describes Shopify product creation and option addition.                                                                                                                                                                            |
| Sticky Note - Add Variants & Images | StickyNote          | Documents variants and images block      | None                       | None                       | Describes product variants and images upload process.                                                                                                                                                                              |
| Sticky Note                 | StickyNote                | Empty sticky note                        | None                       | None                       |                                                                                                                                                                                                                                     |
| Sticky Note1                | StickyNote                | Empty sticky note                        | None                       | None                       |                                                                                                                                                                                                                                     |
| Sticky Note2                | StickyNote                | Empty sticky note                        | None                       | None                       |                                                                                                                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Google Sheets Node ("Get List")**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2 with read access.  
   - Parameters:  
     - Document ID: Your Google Sheet ID containing product URLs.  
     - Sheet Name/GID: The worksheet ID or name.  
   - Configure to read rows with a column named `Product Link`.  
   - Connect Manual Trigger output to this node.

3. **Add SplitInBatches Node ("Loop Over Items")**  
   - Type: SplitInBatches  
   - Purpose: Process each product URL individually.  
   - Configuration: Reset option disabled.  
   - Connect Google Sheets node output to this node.

4. **Add BrowserAct Node ("Run a workflow")**  
   - Type: BrowserAct (Community node)  
   - Credentials: BrowserAct API key with access to your scraping workflows.  
   - Parameters:  
     - Workflow ID: Use the BrowserAct workflow designed for scraping product pages (e.g., `"57519762985270138"`).  
     - Input Parameters: Map `Target_Product_Url` to `{{$json["Product Link"]}}`.  
     - Disable saving browser data for speed.  
   - Connect Loop Over Items output to this node.

5. **Add BrowserAct Node ("Get workflow Data")**  
   - Type: BrowserAct  
   - Credentials: Same as above.  
   - Parameters:  
     - Operation: Get Task  
     - Task ID: Use the task ID from previous node.  
     - Max Wait Time: 900 seconds to allow scraping to complete.  
     - Wait for Finish: true.  
   - Connect "Run a workflow" output to this node.

6. **Add Code Node ("Parse Data")**  
   - Type: Code (JavaScript)  
   - Purpose: Transform raw scrape output into structured JSON.  
   - Code snippet:  
     ```js
     const jsonString = $input.first().json.output.string;
     const productArray = JSON.parse(jsonString);
     return productArray.map(product => {
       const sizesList = product.Size.split(',').map(size => ({ "Size": size.trim().replace(" ", "-") }));
       const imagesList = product.Images.split('||').map(url => ({ "Image": url.trim() }));
       return { json: {
         "Title": product.Title,
         "Price": product.Price,
         "Description": product.Description,
         "Category": product.Category,
         "sizes": sizesList,
         "images": imagesList
       }};
     });
     ```  
   - Connect "Get workflow Data" output to this node.

7. **Add Shopify Node ("Create a product")**  
   - Type: Shopify  
   - Credentials: Shopify Access Token with product write permission.  
   - Parameters:  
     - Resource: Product  
     - Operation: Create  
     - Fields:  
       - Title: `={{ $json.Title }}`  
       - Body HTML: `={{ $json.Description }}`  
       - Tags: `={{ $json.Category }}`  
       - Vendor: "NIKE" (static)  
   - Connect "Parse Data" output to this node.

8. **Add Set Node ("Set Option")**  
   - Type: Set  
   - Purpose: Define product option name for Shopify product.  
   - Assign field `Option` with value `"Shoe Size"`.  
   - Connect "Create a product" output to this node.

9. **Add HTTP Request Node ("Add Option")**  
   - Type: HTTP Request  
   - Credentials: Shopify Access Token  
   - Parameters:  
     - Method: PUT  
     - URL:  
       `https://{shopify-store-domain}/admin/api/2025-01/products/{{ $('Create a product').item.json.id }}.json`  
     - Body Type: JSON  
     - Body:  
       ```json
       {
         "product": {
           "id": "{{ $('Create a product').item.json.id }}",
           "options": [{ "name": "{{ $json.Option }}" }]
         }
       }
       ```  
   - Connect "Set Option" output to this node.

10. **Add Merge Node ("Merge")**  
    - Type: Merge (combineBySql)  
    - Purpose: Join parsed product data and Shopify product creation result.  
    - SQL Query:  
      ```
      SELECT
        input2.*,
        input1.product.id
      FROM input2
      LEFT JOIN input1
      ```  
    - Connect "Add Option" output and "Parse Data" output as inputs.

11. **Add SplitOut Node ("Split Shoe Size")**  
    - Type: SplitOut  
    - Parameters:  
      - Field to split out: `sizes`  
      - Include fields: `input1.product->id`  
    - Connect "Merge" output to this node.

12. **Add HTTP Request Node ("Add Variant")**  
    - Type: HTTP Request  
    - Credentials: Shopify Access Token  
    - Parameters:  
      - Method: POST  
      - URL:  
        `https://{shopify-store-domain}/admin/api/2025-01/products/{{ $json["input1.product->id"] }}/variants.json`  
      - Body Type: JSON  
      - Body:  
        ```json
        {
          "variant": {
            "option1": "{{ $json.sizes.Size }}",
            "price": "{{ $('Merge').item.json.Price }}",
            "sku": "{{ $json['input1.product->id'] }}-SKU-{{ $json.sizes.Size.replace('.', '-').replace(/\s+/g, '-') }}"
          }
        }
        ```  
      - On Error: Continue workflow  
    - Connect "Split Shoe Size" output to this node.

13. **Add SplitOut Node ("Split Images")**  
    - Type: SplitOut  
    - Parameters:  
      - Field to split out: `images`  
      - Include fields: `input1.product->id`  
    - Connect "Merge" output to this node.

14. **Add HTTP Request Node ("Add Images")**  
    - Type: HTTP Request  
    - Credentials: Shopify Access Token  
    - Parameters:  
      - Method: POST  
      - URL:  
        `https://{shopify-store-domain}/admin/api/2025-01/products/{{ $json["input1.product->id"] }}/images.json`  
      - Body Type: JSON  
      - Body:  
        ```json
        {
          "image": {
            "src": "{{ $json.images.Image }}"
          }
        }
        ```  
    - Connect "Split Images" output to this node.

15. **Add Merge Node ("Merge1")**  
    - Type: Merge  
    - Purpose: Combine outputs of variant and image creation.  
    - Connect "Add Variant" and "Add Images" outputs to this node.

16. **Connect "Merge1" output back to "Loop Over Items"**  
    - This allows processing the next item in the batch.

17. **Add Slack Node ("Send a message")**  
    - Type: Slack  
    - Credentials: Slack OAuth2 with permission to post in target channel.  
    - Parameters:  
      - Channel: Channel ID (e.g., `C09KLV9DJSX`)  
      - Message: "The Products Updated"  
    - Connect first output of "Loop Over Items" (batch processing) to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow requires a BrowserAct API account and the BrowserAct n8n Community Node for web scraping.                                                                                                                                  | [n8n Nodes BrowserAct](https://www.npmjs.com/package/n8n-nodes-browseract-workflows)                           |
| The workflow uses a BrowserAct template named “Bulk Product Scraping From (URLs) and uploading to Shopify (Optimised for shoe - NIKE -> Shopify)”.                                                                                       | BrowserAct user account/template setup                                                                       |
| For best results, ensure your Shopify API credentials have permissions for products, variants, and images API operations.                                                                                                                | Shopify API docs                                                                                               |
| Slack notifications are optional but recommended for monitoring progress.                                                                                                                                                                 | Slack OAuth2 API                                                                                                |
| Video tutorials and support are available on BrowserAct's YouTube channel and Discord.                                                                                                                                                    | [How to Find Your BrowseAct API Key & Workflow ID](https://www.youtube.com/watch?v=pDjoZWEsZlE) <br> [Discord](https://discord.com/invite/UpnCKd7GaU) <br> [BrowserAct Blog](https://www.browseract.com/blog) |

---

**Disclaimer:**  
The text provided is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.