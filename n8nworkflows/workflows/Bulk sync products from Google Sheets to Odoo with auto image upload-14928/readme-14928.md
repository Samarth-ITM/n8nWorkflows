Bulk sync products from Google Sheets to Odoo with auto image upload

https://n8nworkflows.xyz/workflows/bulk-sync-products-from-google-sheets-to-odoo-with-auto-image-upload-14928


# Bulk sync products from Google Sheets to Odoo with auto image upload

# Workflow Analysis: Bulk Sync Products from Google Sheets to Odoo with Auto Image Upload

## 1. Workflow Overview
This workflow automates the synchronization of product data from a Google Sheet into an Odoo ERP system. It is designed to handle bulk imports while preventing duplicates by validating barcodes, automatically assigning product categories, and managing product imagery via external URLs.

The logic is organized into three primary functional blocks:
- **1.1 Fetch & Verify:** Monitors Google Sheets for new entries and checks for existing product barcodes in Odoo to avoid duplicates.
- **1.2 Category & Image Processing:** Resolves the internal Odoo category ID and handles the downloading and encoding of product images.
- **1.3 Creation & Status Update:** Creates the product record in Odoo and updates the original spreadsheet to mark the row as "Done," featuring a built-in retry mechanism for API failures.

---

## 2. Block-by-Block Analysis

### 2.1 Fetch & Verify
**Overview:** This block acts as the entry point, ensuring that only new data is processed and that products already existing in Odoo are skipped.

- **Nodes Involved:** `On New Row in Sheets`, `Process Row Batch`, `Check Odoo for Barcode`, `If Product Missing`.
- **Node Details:**
    - **On New Row in Sheets (Google Sheets Trigger):** 
        - *Role:* Polls a specific spreadsheet every minute for new rows.
        - *Config:* Linked to a specific Document ID and Sheet GID.
    - **Process Row Batch (Split In Batches):** 
        - *Role:* Controls the flow of data to prevent API rate limiting or timeouts.
    - **Check Odoo for Barcode (Odoo):** 
        - *Role:* Searches for a product using the `barcode` field.
        - *Technical:* Uses `customResource: product.product` with a filter matching the barcode from the sheet.
    - **If Product Missing (If):** 
        - *Role:* Logic gate to determine if the product needs to be created.
        - *Expression:* Checks if the `id` returned by the Odoo search is empty. If empty $\rightarrow$ True (Proceed to create); if not empty $\rightarrow$ False (Return to batch).

### 2.2 Category, Image Processing & Creation
**Overview:** This block prepares the necessary metadata (Category ID and Base64 image string) required by Odoo's API to create a complete product template.

- **Nodes Involved:** `Get Odoo Category`, `Has Image URL?`, `Download Image`, `Convert Image to Base64`, `Create Product (With Image)`, `Create Product (No Image)`.
- **Node Details:**
    - **Get Odoo Category (Odoo):** 
        - *Role:* Finds the internal Odoo ID for the category name provided in the sheet.
        - *Technical:* Filters `product.category` by name.
    - **Has Image URL? (If):** 
        - *Role:* Checks if the column "الصوره" contains a value.
    - **Download Image (HTTP Request):** 
        - *Role:* Fetches the image file from the provided URL.
        - *Config:* Set to `responseFormat: file`.
    - **Convert Image to Base64 (Extract From File):** 
        - *Role:* Converts the binary image data into a Base64 string (required by Odoo).
        - *Output:* Maps the result to a variable `final_image`.
    - **Create Product (With/Without Image) (Odoo):** 
        - *Role:* Creates the record in `product.template`.
        - *Key Mappings:* Name $\rightarrow$ Name, Barcode $\rightarrow$ barcode, Sales Price $\rightarrow$ list_price, Cost $\rightarrow$ standard_price, Category $\rightarrow$ categ_id. The "With Image" version adds `final_image` to `image_1920`.

### 2.3 Status & Error Recovery
**Overview:** Finalizes the process by updating the source sheet and providing a safety loop for transient network errors.

- **Nodes Involved:** `Mark as Done in Sheet`, `Wait on Error`.
- **Node Details:**
    - **Mark as Done in Sheet (Google Sheets):** 
        - *Role:* Updates the "الحاله" (Status) column to "Done" for the processed row.
        - *Matching:* Uses the `Name` column as the unique identifier for the update.
        - *Error Handling:* Configured to `continueErrorOutput` to trigger the recovery path.
    - **Wait on Error (Wait):** 
        - *Role:* Pauses execution for 1 minute if the Google Sheet update fails before attempting to process the batch again.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On New Row in Sheets | Google Sheets Trigger | Entry point / Data fetch | - | Process Row Batch | Phase 1: Fetch & Verify |
| Process Row Batch | Split In Batches | Batch control / Loop | On New Row, Mark as Done, Wait on Error | Check Odoo for Barcode | Phase 1: Fetch & Verify |
| Check Odoo for Barcode | Odoo | Duplicate validation | Process Row Batch | If Product Missing | Phase 1: Fetch & Verify |
| If Product Missing | If | Route: Create vs Skip | Check Odoo for Barcode | Get Odoo Category, Process Row Batch | Phase 1: Fetch & Verify |
| Get Odoo Category | Odoo | Category ID resolution | If Product Missing | Has Image URL? | Phase 2: Category, Image Processing & Creation |
| Has Image URL? | If | Image presence check | Get Odoo Category | Download Image, Create Product (No Image) | Phase 2: Category, Image Processing & Creation |
| Download Image | HTTP Request | Binary file fetch | Has Image URL? | Convert Image to Base64 | Phase 2: Category, Image Processing & Creation |
| Convert Image to Base64 | Extract From File | Binary $\rightarrow$ Base64 | Download Image | Create Product (With Image) | Phase 2: Category, Image Processing & Creation |
| Create Product (With Image) | Odoo | Record creation (Full) | Convert Image to Base64 | Mark as Done in Sheet | Phase 2: Category, Image Processing & Creation |
| Create Product (No Image) | Odoo | Record creation (Basic) | Has Image URL? | Mark as Done in Sheet | Phase 2: Category, Image Processing & Creation |
| Mark as Done in Sheet | Google Sheets | Status update | Create Product (With/No Image) | Process Row Batch, Wait on Error | Phase 3: Status & Error Recovery |
| Wait on Error | Wait | Retry delay | Mark as Done in Sheet | Process Row Batch | Phase 3: Status & Error Recovery |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Input and Verification (Phase 1)
1. **Create Google Sheets Trigger**: Set event to `rowAdded`, select the target document and sheet. Set polling to every minute.
2. **Create Split In Batches Node**: Connect the trigger to this node to manage data processing in chunks.
3. **Create Odoo Node (Search)**: 
   - Resource: `custom` $\rightarrow$ `product.product`.
   - Operation: `getAll`.
   - Filter: `barcode` equals `{{ $json.Barcode }}`.
4. **Create If Node**: Check if `{{ $json.id }}` is empty.
   - **True Path**: Connect to Step 2.
   - **False Path**: Connect back to "Process Row Batch".

### Step 2: Data Enrichment and Creation (Phase 2)
5. **Create Odoo Node (Category)**:
   - Resource: `custom` $\rightarrow$ `product.category`.
   - Operation: `getAll`.
   - Filter: `name` equals `{{ $('Process Row Batch').item.json.category }}`.
6. **Create If Node (Image Check)**: Check if the column `الصوره` is not empty.
7. **Create HTTP Request Node (True Path)**: Set URL to `{{ $('On New Row in Sheets').item.json['الصوره'] }}` and response format to `File`.
8. **Create Extract From File Node**: Set operation to `Binary to Property` and destination key to `final_image`.
9. **Create Odoo Node (Create With Image)**:
   - Resource: `custom` $\rightarrow$ `product.template`.
   - Operation: `Create`.
   - Map: `name`, `barcode`, `list_price` (Sales Price), `standard_price` (Cost), `categ_id` (from Step 5), and `image_1920` (from Step 8).
10. **Create Odoo Node (Create No Image - False Path from Step 6)**:
    - Same as Step 9, but omit the `image_1920` field.

### Step 3: Closing the Loop (Phase 3)
11. **Create Google Sheets Node (Update)**:
    - Operation: `Update`.
    - Matching Column: `Name`.
    - Update: Set `الحاله` to `Done`.
    - Error Handling: Change "On Error" settings to `Continue (output error)`.
12. **Create Wait Node**: Set to 1 minute. Connect the error output of the Google Sheets node here.
13. **Closing Connection**: Connect the "Wait" node back to the "Process Row Batch" node to resume the loop.

### Credential Requirements
- **Google Sheets OAuth2**: Required for Trigger and Update nodes.
- **Odoo API**: Required for all Odoo nodes (ensure the user has access to `product.product`, `product.template`, and `product.category`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Odoo Logic** | The workflow uses `product.product` for searching (variants) but `product.template` for creation. |
| **Image Handling** | Images must be publicly accessible via URL for the HTTP Request node to download them. |
| **Retry Logic** | The loop from "Wait on Error" back to "Process Row Batch" ensures that a temporary Google API outage doesn't stop the entire sync process. |