Import Odoo Product Images from Google Drive

https://n8nworkflows.xyz/workflows/import-odoo-product-images-from-google-drive-3181


# Import Odoo Product Images from Google Drive

### 1. Workflow Overview

This workflow automates the synchronization of product images from a designated Google Drive folder into Odoo’s product catalog. It is tailored for businesses and developers who want to eliminate manual uploads and ensure product images in Odoo are always up-to-date.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Triggering**: Initiates the workflow either on a schedule or manually.
- **1.2 File Discovery & Filtering**: Finds image files in Google Drive and filters them by extension.
- **1.3 Metadata Extraction & Validation**: Parses file names to extract model type and SKU.
- **1.4 Routing by Model Type**: Directs images to either product templates or individual products processing.
- **1.5 Odoo Record Matching**: Searches Odoo for matching product or template records by SKU.
- **1.6 Image Download & Conversion**: Downloads images from Google Drive and converts them to Base64.
- **1.7 Odoo Image Update**: Updates the corresponding Odoo records with new images in multiple resolutions.
- **1.8 Cleanup of Processed Files**: Moves processed images to a “done” folder and deletes any old duplicates.
- **1.9 Completion Announcement**: Sends a summary notification via Google Chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Triggering

- **Overview:** Starts the workflow either on a scheduled interval (every 10 minutes) or manually via a trigger node.
- **Nodes Involved:** `Schedule Trigger`, `Click Manual`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Configuration: Runs every 10 minutes.
    - Inputs: None (trigger node)
    - Outputs: Initiates the workflow by triggering the `Find Files` node.
    - Edge Cases: Scheduling misconfiguration or disabled node will prevent automatic runs.

  - **Click Manual**
    - Type: Manual Trigger
    - Configuration: Allows manual execution.
    - Inputs: None
    - Outputs: Starts the workflow by triggering `Find Files`.
    - Edge Cases: Manual trigger requires user interaction.

#### 2.2 File Discovery & Filtering

- **Overview:** Retrieves all files from a specific Google Drive folder and filters them to only include `.png` and `.jpg` images.
- **Nodes Involved:** `Find Files`, `Filter Images`
- **Node Details:**

  - **Find Files**
    - Type: Google Drive (File/Folder)
    - Configuration: Lists all files in the folder with ID `1VG-7mRW8tsmJelW5FTeoj2jXeObMvan6` within the drive `0AGL-iqy2wxM8Uk9PVA`.
    - Credentials: Google Drive OAuth2 Administrator
    - Inputs: Trigger from `Schedule Trigger` or `Click Manual`
    - Outputs: Passes all files to `Filter Images`.
    - Edge Cases: API rate limits, folder access permissions, empty folder.

  - **Filter Images**
    - Type: Filter
    - Configuration: Filters files where the name ends with `.png` or `.jpg` (case-sensitive).
    - Inputs: Files from `Find Files`
    - Outputs: Filtered image files to `Decorate Images`.
    - Edge Cases: Files with uppercase extensions or other image formats will be excluded.

#### 2.3 Metadata Extraction & Validation

- **Overview:** Parses file names to extract the model type (`template` or `product`) and SKU, validating the naming convention.
- **Nodes Involved:** `Decorate Images`
- **Node Details:**

  - **Decorate Images**
    - Type: Code (JavaScript)
    - Configuration: Splits the file name by underscores and dots to assign `model` and `sku` properties.
      - Example: `template_ABC123.png` → model: `template`, sku: `ABC123`
    - Inputs: Filtered images from `Filter Images`
    - Outputs: Decorated items with metadata to `Switch`, `Move Images`, and `Search Old Images`.
    - Edge Cases: Files not following the naming convention will have incorrect or missing metadata, potentially causing downstream failures.

#### 2.4 Routing by Model Type

- **Overview:** Routes images to different processing branches based on whether they are product templates or individual products.
- **Nodes Involved:** `Switch`
- **Node Details:**

  - **Switch**
    - Type: Switch
    - Configuration: Routes based on `model` property:
      - If `model` equals `template`, route to `Find Templates`.
      - If `model` equals `product`, route to `Find Products`.
    - Inputs: Decorated images from `Decorate Images`
    - Outputs: Two branches for templates and products.
    - Edge Cases: Files with invalid or unexpected `model` values will not be processed.

#### 2.5 Odoo Record Matching

- **Overview:** Searches Odoo for product template or product records matching the SKU extracted from the file name.
- **Nodes Involved:** `Find Templates`, `Find Products`
- **Node Details:**

  - **Find Templates**
    - Type: Odoo
    - Configuration: Searches `product.template` model for records with `default_code` equal to SKU.
    - Limit: 1 record
    - Inputs: From `Switch` (template branch)
    - Outputs: Passes found template record to `Download Images Templates`
    - Credentials: Odoo API with appropriate access
    - Edge Cases: No matching template found results in no update; API errors or permission issues possible.

  - **Find Products**
    - Type: Odoo
    - Configuration: Searches `product.product` model for records with `default_code` equal to SKU.
    - Limit: 1 record
    - Inputs: From `Switch` (product branch)
    - Outputs: Passes found product record to `Download Images Products`
    - Credentials: Odoo API
    - Edge Cases: Same as `Find Templates`.

#### 2.6 Image Download & Conversion

- **Overview:** Downloads the image file from Google Drive and converts the binary data to Base64 for Odoo compatibility.
- **Nodes Involved:** `Download Images Templates`, `Convert Base64 Images Templates`, `Download Images Products`, `Convert Base64 Images Products`
- **Node Details:**

  - **Download Images Templates / Download Images Products**
    - Type: Google Drive (Download)
    - Configuration: Downloads the file by ID from the filtered image.
    - Inputs: From `Find Templates` or `Find Products`
    - Outputs: Binary image data to respective `Convert Base64` nodes.
    - Credentials: Google Drive OAuth2 Administrator
    - Edge Cases: Download failures, file not found, permission issues.

  - **Convert Base64 Images Templates / Convert Base64 Images Products**
    - Type: Extract From File
    - Configuration: Converts binary image data to Base64 string stored in JSON property `data`.
    - Inputs: Binary data from download nodes
    - Outputs: Base64 encoded image data to `Update Images Templates` or `Update Images Products`.
    - Edge Cases: Binary data corruption or conversion errors.

#### 2.7 Odoo Image Update

- **Overview:** Updates the corresponding Odoo product or template record with the new image in multiple resolution fields.
- **Nodes Involved:** `Update Images Templates`, `Update Images Products`
- **Node Details:**

  - **Update Images Templates**
    - Type: Odoo
    - Configuration: Updates `product.template` record identified by ID with Base64 image data in fields: `image_1920`, `image_1024`, `image_512`, `image_256`, `image_128`.
    - Inputs: Base64 data from `Convert Base64 Images Templates`
    - Outputs: None (end of template branch)
    - Credentials: Odoo API
    - Edge Cases: Update failures, invalid record ID, API errors.

  - **Update Images Products**
    - Type: Odoo
    - Configuration: Same as above but for `product.product` model.
    - Inputs: Base64 data from `Convert Base64 Images Products`
    - Outputs: None (end of product branch)
    - Credentials: Odoo API
    - Edge Cases: Same as above.

#### 2.8 Cleanup of Processed Files

- **Overview:** Moves processed images to a "done" folder in Google Drive and deletes any old duplicate images.
- **Nodes Involved:** `Move Images`, `Sum Images`, `Search Old Images`, `Drop Old Images`
- **Node Details:**

  - **Move Images**
    - Type: Google Drive (Move)
    - Configuration: Moves the processed file to folder ID `1NqxzbwarAZ1BtkoyM-T8NNcO5m_cmO1V` in drive `0AAaxIiOTPGeCUk9PVA`.
    - Inputs: From `Decorate Images`
    - Outputs: Passes moved files to `Sum Images`
    - Credentials: Google Drive OAuth2 Administrator
    - Edge Cases: Move failures, permission issues.

  - **Sum Images**
    - Type: Summarize
    - Configuration: Counts the total number of processed images by summarizing the `id` field.
    - Inputs: From `Move Images`
    - Outputs: Passes summary count to `Announce`
    - Edge Cases: Empty input results in zero count.

  - **Search Old Images**
    - Type: Google Drive (File/Folder)
    - Configuration: Searches the "done" folder for files with the same name as the currently processed image.
    - Inputs: From `Decorate Images`
    - Outputs: Passes found old files to `Drop Old Images`
    - Credentials: Google Drive OAuth2 Administrator
    - Edge Cases: No old files found results in no deletion.

  - **Drop Old Images**
    - Type: Google Drive (Delete File)
    - Configuration: Deletes the old duplicate file by ID.
    - Inputs: From `Search Old Images`
    - Outputs: None
    - Credentials: Google Drive OAuth2 Administrator
    - Edge Cases: Deletion failures, file already deleted.

#### 2.9 Completion Announcement

- **Overview:** Sends a notification message to a Google Chat space summarizing the number of images processed.
- **Nodes Involved:** `Announce`
- **Node Details:**

  - **Announce**
    - Type: Google Chat
    - Configuration: Sends a message to space `spaces/AAAAt6xI1aY` with text:  
      `"Product images done onto Google Drive (total : {{ $json.count_id }})."`
    - Inputs: From `Sum Images`
    - Credentials: Google Chat OAuth2 Administrator
    - Edge Cases: Message delivery failure, invalid webhook or OAuth credentials.

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                      | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                   |
|-----------------------------|--------------------------|------------------------------------|------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Click Manual                | Manual Trigger           | Manual workflow start              | None                         | Find Files                      |                                                                                              |
| Schedule Trigger            | Schedule Trigger         | Scheduled workflow start           | None                         | Find Files                      |                                                                                              |
| Find Files                 | Google Drive             | List files in input folder         | Click Manual, Schedule Trigger | Filter Images                   |                                                                                              |
| Filter Images              | Filter                   | Filter files by `.png` or `.jpg`  | Find Files                   | Decorate Images                 |                                                                                              |
| Decorate Images            | Code                     | Extract model and SKU from filename| Filter Images                | Switch, Move Images, Search Old Images |                                                                                              |
| Switch                    | Switch                   | Route by model type (template/product) | Decorate Images             | Find Templates, Find Products   |                                                                                              |
| Find Templates            | Odoo                     | Find product template by SKU       | Switch (template branch)     | Download Images Templates       |                                                                                              |
| Download Images Templates  | Google Drive             | Download template image            | Find Templates               | Convert Base64 Images Templates |                                                                                              |
| Convert Base64 Images Templates | Extract From File       | Convert template image to Base64   | Download Images Templates    | Update Images Templates         |                                                                                              |
| Update Images Templates    | Odoo                     | Update template images in Odoo     | Convert Base64 Images Templates | None                          |                                                                                              |
| Find Products             | Odoo                     | Find product by SKU                | Switch (product branch)      | Download Images Products        |                                                                                              |
| Download Images Products   | Google Drive             | Download product image             | Find Products                | Convert Base64 Images Products  |                                                                                              |
| Convert Base64 Images Products | Extract From File       | Convert product image to Base64    | Download Images Products     | Update Images Products          |                                                                                              |
| Update Images Products     | Odoo                     | Update product images in Odoo      | Convert Base64 Images Products | None                          |                                                                                              |
| Move Images               | Google Drive             | Move processed images to "done" folder | Decorate Images             | Sum Images                     |                                                                                              |
| Sum Images                | Summarize                | Count total processed images       | Move Images                  | Announce                       |                                                                                              |
| Search Old Images         | Google Drive             | Search for old duplicates in "done" folder | Decorate Images             | Drop Old Images                |                                                                                              |
| Drop Old Images           | Google Drive             | Delete old duplicate images        | Search Old Images            | None                          |                                                                                              |
| Announce                  | Google Chat              | Notify completion with summary     | Sum Images                   | None                          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node:
     - Set interval to every 10 minutes.
   - Add a **Manual Trigger** node for manual execution.

2. **Create Google Drive File Listing:**
   - Add a **Google Drive** node named `Find Files`.
   - Set resource to `fileFolder`.
   - Configure to list all files in the input folder:
     - Drive ID: `0AGL-iqy2wxM8Uk9PVA`
     - Folder ID: `1VG-7mRW8tsmJelW5FTeoj2jXeObMvan6`
   - Use Google Drive OAuth2 credentials.
   - Connect outputs of both triggers to this node.

3. **Filter Image Files:**
   - Add a **Filter** node named `Filter Images`.
   - Set conditions to pass files where `name` ends with `.png` OR `.jpg` (case-sensitive).
   - Connect `Find Files` output to this node.

4. **Extract Metadata from File Names:**
   - Add a **Code** node named `Decorate Images`.
   - Use JavaScript code to split file name by underscores and dots, setting:
     - `model` = first part (e.g., `template` or `product`)
     - `sku` = remainder joined by underscores
   - Connect `Filter Images` output to this node.

5. **Route by Model Type:**
   - Add a **Switch** node named `Switch`.
   - Add two rules:
     - If `model` equals `template`, route to `Find Templates`.
     - If `model` equals `product`, route to `Find Products`.
   - Connect `Decorate Images` output to this node.

6. **Find Odoo Records:**
   - Add an **Odoo** node named `Find Templates`:
     - Resource: `custom`
     - Operation: `getAll`
     - Custom Resource: `product.template`
     - Filter: `default_code` equals `sku`
     - Limit: 1
     - Use Odoo API credentials.
   - Add an **Odoo** node named `Find Products`:
     - Same as above but for `product.product`.
   - Connect `Switch` outputs to respective nodes.

7. **Download Images from Google Drive:**
   - Add two **Google Drive** nodes:
     - `Download Images Templates`
     - `Download Images Products`
   - Set operation to `download`.
   - File ID: Use the filtered image ID from `Filter Images`.
   - Binary property name: `data`.
   - Use Google Drive OAuth2 credentials.
   - Connect `Find Templates` to `Download Images Templates`.
   - Connect `Find Products` to `Download Images Products`.

8. **Convert Images to Base64:**
   - Add two **Extract From File** nodes:
     - `Convert Base64 Images Templates`
     - `Convert Base64 Images Products`
   - Operation: `binaryToProperty`
   - Connect `Download Images Templates` to `Convert Base64 Images Templates`.
   - Connect `Download Images Products` to `Convert Base64 Images Products`.

9. **Update Odoo Records with Images:**
   - Add two **Odoo** nodes:
     - `Update Images Templates`
       - Operation: `update`
       - Custom Resource: `product.template`
       - Custom Resource ID: from `Find Templates` result
       - Fields to update: `image_1920`, `image_1024`, `image_512`, `image_256`, `image_128` with Base64 data.
     - `Update Images Products`
       - Same as above but for `product.product`.
   - Connect `Convert Base64 Images Templates` to `Update Images Templates`.
   - Connect `Convert Base64 Images Products` to `Update Images Products`.

10. **Move Processed Images:**
    - Add a **Google Drive** node named `Move Images`.
    - Operation: `move`
    - File ID: from `Decorate Images`
    - Destination folder ID: `1NqxzbwarAZ1BtkoyM-T8NNcO5m_cmO1V` in drive `0AAaxIiOTPGeCUk9PVA`
    - Use Google Drive OAuth2 credentials.
    - Connect `Decorate Images` output to this node.

11. **Summarize Processed Images:**
    - Add a **Summarize** node named `Sum Images`.
    - Summarize field: `id` (count)
    - Connect `Move Images` output to this node.

12. **Search and Delete Old Images:**
    - Add a **Google Drive** node named `Search Old Images`.
    - Operation: `fileFolder` with query string set to the current file name.
    - Folder ID: same as "done" folder.
    - Use Google Drive OAuth2 credentials.
    - Connect `Decorate Images` output to this node.

    - Add a **Google Drive** node named `Drop Old Images`.
    - Operation: `deleteFile`
    - File ID: from `Search Old Images`
    - Use Google Drive OAuth2 credentials.
    - Connect `Search Old Images` output to this node.

13. **Announce Completion:**
    - Add a **Google Chat** node named `Announce`.
    - Space ID: `spaces/AAAAt6xI1aY`
    - Message text: `"Product images done onto Google Drive (total : {{ $json.count_id }})."`
    - Authentication: OAuth2 with Google Chat Administrator credentials.
    - Connect `Sum Images` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Images must be named strictly as `template_{sku}.png/jpg` or `product_{sku}.png/jpg` for correct parsing. | Naming convention critical for metadata extraction and routing.                                 |
| Google Drive folder IDs must be configured correctly for input and done folders.                           | Input folder: `1VG-7mRW8tsmJelW5FTeoj2jXeObMvan6`, Done folder: `1NqxzbwarAZ1BtkoyM-T8NNcO5m_cmO1V` |
| Odoo API credentials require access to `product.template` and `product.product` models.                    | Ensure API user has write permissions for image fields.                                        |
| Scheduler interval is set to 10 minutes but can be adjusted as needed.                                    |                                                                                               |
| Google Chat webhook requires OAuth2 authentication with proper scopes.                                    |                                                                                               |
| Workflow designed to handle both manual and scheduled executions.                                        |                                                                                               |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Import Odoo Product Images from Google Drive" workflow, including all nodes, configurations, and integration points.