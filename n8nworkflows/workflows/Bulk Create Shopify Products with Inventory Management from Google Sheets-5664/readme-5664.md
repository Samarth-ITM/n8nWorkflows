Bulk Create Shopify Products with Inventory Management from Google Sheets

https://n8nworkflows.xyz/workflows/bulk-create-shopify-products-with-inventory-management-from-google-sheets-5664


# Bulk Create Shopify Products with Inventory Management from Google Sheets

### 1. Workflow Overview

This workflow automates the bulk creation of Shopify products from a Google Sheets spreadsheet while managing inventory levels at the store's default location. It is designed for efficiently importing new product data into Shopify, skipping products that already exist based on their unique slug (handle). The workflow is well-suited for staging or test environments where rapid product setup and inventory initialization are needed.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Fetching product data from a Google Sheets document.
- **1.2 Location Retrieval:** Obtaining the default location ID from Shopify for inventory assignment.
- **1.3 Product Existence Check:** Querying Shopify to identify if each product already exists by handle.
- **1.4 Product Creation & Inventory Management:** Creating new products, enabling inventory tracking, and setting inventory levels.
- **1.5 Batch Processing Control:** Iterating over each product row in batches for scalable processing.

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception (Google Sheets Fetch)

- **Overview:**  
  Fetches all product data rows from a specified Google Sheet to be processed for Shopify product creation.

- **Nodes Involved:**  
  - Start Workflow  
  - Shopify, GetLocations  
  - Google Sheet, Fetch Products

- **Node Details:**

  **Start Workflow**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually.  
  - Config: No parameters, triggers the workflow manually.  
  - Inputs: None  
  - Outputs: Shopify, GetLocations  
  - Edge Cases: None (manual start)  

  **Shopify, GetLocations**  
  - Type: GraphQL  
  - Role: Retrieves the first Shopify location to assign inventory.  
  - Config: GraphQL query fetching first location details; endpoint points to Shopify Admin API 2025-04 version.  
  - Credentials: Shopify GraphQL Header Authentication with API key/token.  
  - Inputs: From Start Workflow node.  
  - Outputs: Google Sheet, Fetch Products  
  - Edge Cases:  
    - API authentication failure  
    - No locations returned (empty store location)  
    - Network timeout  

  **Google Sheet, Fetch Products**  
  - Type: Google Sheets  
  - Role: Reads all product rows from the specified sheet "Products".  
  - Config: Uses OAuth2 credentials linked to Google Sheets; reads entire sheet by name and document ID.  
  - Inputs: From Shopify, GetLocations  
  - Outputs: Loop Over Items node  
  - Edge Cases:  
    - Sheet or document ID incorrect  
    - API quota exceeded  
    - Empty or malformed data rows  

---

#### Block 1.2: Batch Processing Control

- **Overview:**  
  Splits the fetched product data into batches to process each product individually, enabling scalable iteration.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each product row one at a time (default batch size is 1).  
  - Config: Default settings, no batch size explicitly set (assumed 1).  
  - Inputs: From Google Sheet, Fetch Products  
  - Outputs: Shopify, ProductQuery (for product existence check), Finished (end of processing)  
  - Edge Cases:  
    - Empty input data results in no iterations  
    - Large datasets may require batch size tuning for performance  

---

#### Block 1.3: Product Existence Check

- **Overview:**  
  Checks if each product already exists in Shopify by querying the product handle (slug).

- **Nodes Involved:**  
  - Shopify, ProductQuery  
  - If product exists

- **Node Details:**

  **Shopify, ProductQuery**  
  - Type: GraphQL  
  - Role: Queries Shopify for a product by handle to determine existence.  
  - Config: GraphQL query with variable `handle` bound to the current item’s slug field.  
  - Credentials: Shopify GraphQL Header Authentication.  
  - Inputs: Loop Over Items  
  - Outputs: If product exists  
  - Edge Cases:  
    - Authentication failure  
    - Shopify API limits or errors  
    - Missing or malformed slug fields causing query errors  

  **If product exists**  
  - Type: If Boolean  
  - Role: Routes workflow based on product existence.  
  - Config: Condition checks if the GraphQL query response contains a product (`productByHandle` exists).  
  - Inputs: Shopify, ProductQuery  
  - Outputs: Loop Over Items (if product exists) to skip creation; Shopify, CreateProduct (if product does not exist)  
  - Edge Cases:  
    - Null or undefined responses could cause condition errors  
    - Logic assumes exact match on handle; differences in casing or formatting may affect detection  

---

#### Block 1.4: Product Creation & Inventory Management

- **Overview:**  
  Creates new Shopify products, enables inventory tracking, and sets the stock quantity at the default location.

- **Nodes Involved:**  
  - Shopify, CreateProduct  
  - Shopify, Enable InventoryTracking  
  - Shopify, Set InventoryLevel

- **Node Details:**

  **Shopify, CreateProduct**  
  - Type: GraphQL  
  - Role: Creates a new product in Shopify with provided metadata and media.  
  - Config: Mutation includes product fields such as title, descriptionHtml, vendor, productType, status, handle, and media with a placeholder image URL. Variables are populated from the current batch item.  
  - Credentials: Shopify GraphQL Header Authentication.  
  - Inputs: If product exists (false condition branch)  
  - Outputs: Shopify, Enable InventoryTracking  
  - Edge Cases:  
    - User errors returned from Shopify (e.g., invalid fields, duplicate handles)  
    - Network or API failure  
    - Missing required fields in input data  

  **Shopify, Enable InventoryTracking**  
  - Type: GraphQL  
  - Role: Enables inventory tracking and shipping requirements on the product's inventory item.  
  - Config: Mutation updates inventoryItem using ID from the newly created product variant; sets `tracked` and `requiresShipping` to true.  
  - Credentials: Shopify GraphQL Header Authentication.  
  - Inputs: Shopify, CreateProduct  
  - Outputs: Shopify, Set InventoryLevel  
  - Edge Cases:  
    - InventoryItem ID missing or invalid  
    - API errors during mutation  
    - Permissions issues  

  **Shopify, Set InventoryLevel**  
  - Type: GraphQL  
  - Role: Sets the on-hand inventory quantity for the product at the store’s default location.  
  - Config: Mutation uses inventoryItem ID and location ID from previous nodes; quantity is set from the product's `stock_on_hand` field.  
  - Credentials: Shopify GraphQL Header Authentication.  
  - Inputs: Shopify, Enable InventoryTracking  
  - Outputs: Loop Over Items (continues processing next product)  
  - Edge Cases:  
    - Incorrect or missing inventoryItem or location IDs  
    - Quantity field missing or invalid (e.g., negative values)  
    - API or permission errors  

---

#### Block 1.5: Workflow Completion

- **Overview:**  
  Marks the completion of batch processing.

- **Nodes Involved:**  
  - Finished

- **Node Details:**

  **Finished**  
  - Type: No Operation (NoOp)  
  - Role: Acts as an endpoint to denote the end of processing for a batch iteration.  
  - Config: No parameters.  
  - Inputs: Loop Over Items (first output branch)  
  - Outputs: None  
  - Edge Cases: None  

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                            | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                                                           |
|-----------------------------|---------------------|-------------------------------------------|--------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Start Workflow              | Manual Trigger      | Entry point to start the workflow manually | None                     | Shopify, GetLocations           |                                                                                                                                        |
| Shopify, GetLocations       | GraphQL             | Fetch default Shopify location for inventory | Start Workflow           | Google Sheet, Fetch Products    |                                                                                                                                        |
| Google Sheet, Fetch Products| Google Sheets       | Retrieve product data rows from Google Sheet | Shopify, GetLocations    | Loop Over Items                 |                                                                                                                                        |
| Loop Over Items             | SplitInBatches      | Iterate over each product row individually | Google Sheet, Fetch Products | Finished, Shopify, ProductQuery |                                                                                                                                        |
| Shopify, ProductQuery       | GraphQL             | Check if product exists in Shopify by handle | Loop Over Items          | If product exists               | Sticky Note1: "Check to see if product exists"                                                                                        |
| If product exists           | If Boolean          | Branch based on product existence          | Shopify, ProductQuery    | Loop Over Items (exists), Shopify, CreateProduct (not exists) |                                                                                                                                        |
| Shopify, CreateProduct      | GraphQL             | Create new product in Shopify              | If product exists (false branch) | Shopify, Enable InventoryTracking | Sticky Note2: "Create the Product - Create product. - Enable inventory tracking - Set inventory quantity"                              |
| Shopify, Enable InventoryTracking | GraphQL       | Enable inventory tracking on the product   | Shopify, CreateProduct   | Shopify, Set InventoryLevel     | Sticky Note2 (duplicated content)                                                                                                    |
| Shopify, Set InventoryLevel | GraphQL             | Set inventory quantity at store location    | Shopify, Enable InventoryTracking | Loop Over Items               | Sticky Note2 (duplicated content)                                                                                                    |
| Finished                   | No Operation (NoOp) | Marks completion of processing iteration    | Loop Over Items          | None                           |                                                                                                                                        |
| Sticky Note                | Sticky Note         | Documentation and setup guidance            | None                     | None                           | "Create Products in Shopify from a Google Sheet\n\nThis workflow creates products in your Shopify store from a google sheet..." (full detailed note) |
| Sticky Note1               | Sticky Note         | Documentation for product existence check   | None                     | None                           | "Check to see if product exists"                                                                                                     |
| Sticky Note2               | Sticky Note         | Documentation for product creation steps    | None                     | None                           | "Create the Product\n- Create product.\n- Enable inventory tracking \n- Set inventory quantity"                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `Start Workflow` to initiate the workflow.

2. **Create a GraphQL node** named `Shopify, GetLocations`:  
   - Set the endpoint to your Shopify Admin API GraphQL URL (e.g., `https://<your-store>.myshopify.com/admin/api/2025-04/graphql.json`).  
   - Add the following query to retrieve the first store location:  
     ```graphql
     query {
       locations(first:1, reverse:true) {
         edges {
           node {
             id
             name
             address {
               address1
               address2
               city
               country
               zip
               province
             }
           }
         }
       }
     }
     ```  
   - Configure authentication using Shopify HTTP header auth credentials (API token).  
   - Connect `Start Workflow` to this node.

3. **Create a Google Sheets node** named `Google Sheet, Fetch Products`:  
   - Connect it to `Shopify, GetLocations`.  
   - Use Google Sheets OAuth2 credentials.  
   - Set the document ID to your Google Sheet containing product data.  
   - Set the sheet name to the relevant tab (e.g., "Products").  
   - Ensure the first row contains column headers matching product fields: title, description, company, category, status, slug, price, compare_at_price, sku, stock_on_hand.

4. **Create a SplitInBatches node** named `Loop Over Items`:  
   - Connect it to `Google Sheet, Fetch Products`.  
   - Default batch size is 1 (process products one by one).

5. **Create a GraphQL node** named `Shopify, ProductQuery` to check product existence:  
   - Connect it to `Loop Over Items`.  
   - Use the Shopify Admin API endpoint and HTTP header auth as before.  
   - Use this GraphQL query with a variable for the handle:  
     ```graphql
     query Products ($handle: String!) {
       productByHandle(handle: $handle) {
         id
         title
         variants(first:20) {
           edges {
             node {
               id
               title
               sku
               price
               inventoryItem {
                 id
               }
             }
           }
         }
         updatedAt
         createdAt
       }
     }
     ```  
   - Set variables as:  
     ```json
     {
       "handle": "={{ $json.slug }}"
     }
     ```

6. **Create an If node** named `If product exists`:  
   - Connect it to `Shopify, ProductQuery`.  
   - Configure a condition checking whether `{{$json.data.productByHandle}}` exists (object exists).  
   - If true (product exists), connect back to `Loop Over Items` to skip creation.  
   - If false (product does not exist), connect to `Shopify, CreateProduct`.

7. **Create a GraphQL node** named `Shopify, CreateProduct`:  
   - Connect it to the false branch of `If product exists`.  
   - Use the Shopify Admin API endpoint and HTTP header auth.  
   - Use this mutation to create a product:  
     ```graphql
     mutation productCreate($product: ProductCreateInput!, $media: [CreateMediaInput!]) {
       productCreate(product: $product, media: $media) {
         product {
           id
           title
           descriptionHtml
           vendor
           productType
           status
           handle
           variants(first:10) {
             edges {
               node {
                 id
                 sku
                 displayName
                 inventoryItem {
                   id
                   sku
                   tracked
                   requiresShipping
                 }
               }
             }
           }
           media(first: 10) {
             edges {
               node {
                 alt
                 mediaContentType
                 status
                 id
                 preview {
                   image {
                     url
                   }
                   status
                 }
               }
             }
           }
           options {
             id
             name
             position
             optionValues {
               id
               name
               hasVariants
             }
           }
         }
         userErrors {
           field
           message
         }
       }
     }
     ```  
   - Set variables referencing current batch item fields:  
     ```json
     {
       "product": {
         "title": "={{$json.title}}",
         "descriptionHtml": "={{$json.description}}",
         "vendor": "={{$json.company}}",
         "productType": "={{$json.category}}",
         "status": "={{$json.status}}",
         "handle": "={{$json.slug}}"
       },
       "media": [{
         "alt": "alt tag",
         "mediaContentType": "IMAGE",
         "originalSource": "https://placehold.co/800x600.png"
       }]
     }
     ```

8. **Create a GraphQL node** named `Shopify, Enable InventoryTracking`:  
   - Connect it to `Shopify, CreateProduct`.  
   - Use mutation to update inventory item:  
     ```graphql
     mutation inventoryItemUpdate($id: ID!, $input: InventoryItemInput!) {
       inventoryItemUpdate(id: $id, input: $input) {
         inventoryItem {
           id
           unitCost {
             amount
           }
           tracked
         }
         userErrors {
           message
         }
       }
     }
     ```  
   - Set variables referencing inventory item ID from created product variant:  
     ```json
     {
       "id": "={{$json.data.productCreate.product.variants.edges[0].node.inventoryItem.id}}",
       "input": {
         "tracked": true,
         "requiresShipping": true
       }
     }
     ```

9. **Create a GraphQL node** named `Shopify, Set InventoryLevel`:  
   - Connect it to `Shopify, Enable InventoryTracking`.  
   - Use mutation to set inventory quantities:  
     ```graphql
     mutation inventorySetOnHandQuantities($input: InventorySetOnHandQuantitiesInput!) {
       inventorySetOnHandQuantities(input: $input) {
         userErrors {
           field
           message
         }
       }
     }
     ```  
   - Set variables with inventoryItemId and locationId from previous nodes and quantity from Google Sheet:  
     ```json
     {
       "input": {
         "reason": "correction",
         "setQuantities": [{
           "inventoryItemId": "={{$json.data.inventoryItemUpdate.inventoryItem.id}}",
           "locationId": "={{$node['Shopify, GetLocations'].item.json.data.locations.edges[0].node.id}}",
           "quantity": "={{$json.stock_on_hand}}"
         }]
       }
     }
     ```

10. **Connect output of `Shopify, Set InventoryLevel` back to `Loop Over Items`** to continue processing next product.

11. **Create a NoOp node** named `Finished`:  
    - Connect the first output of `Loop Over Items` to this node to mark workflow completion.

12. **Credentials Configuration:**  
    - Google Sheets OAuth2 credentials with access to the target spreadsheet.  
    - Shopify HTTP Header Authentication credentials including a valid API token with GraphQL Admin API rights.

13. **Optional:** Add sticky notes for documentation at strategic points, e.g., product existence check, product creation steps, and general workflow overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow creates products in your Shopify store from a Google Sheet. It also enables inventory tracking and sets the quantity of an inventory item at your store's default location. It is ideal for populating test or staging stores with sample data. The Google Sheet should have columns for `title`, `description`, `company`, `category`, `status` (ACTIVE, DRAFT, ARCHIVE), `slug` (product handle), `price`, `compare_at_price`, `sku`, and `stock_on_hand`. Update all GraphQL endpoint URLs to your Shopify store URL and ensure API versions match your store. | See Sticky Note content in the workflow; Shopify Admin API 2025-04 documentation for GraphQL: https://shopify.dev/api/admin-graphql |
| The workflow skips products that already exist by matching the product handle (slug), ensuring no duplicates are created.                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow logic block "If product exists"                                                                                     |
| Placeholder media image URLs are used in product creation. Replace with actual image URLs if required.                                                                                                                                                                                                                                                                                                                                                                                                                          | Shopify, CreateProduct node configuration                                                                                     |
| Shopify API rate limits and permissions can cause failures; ensure the API token has the appropriate scopes (product creation, inventory management). Handle API errors gracefully in production environments.                                                                                                                                                                                                                                                                                                                   | General API and authentication considerations                                                                                 |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.