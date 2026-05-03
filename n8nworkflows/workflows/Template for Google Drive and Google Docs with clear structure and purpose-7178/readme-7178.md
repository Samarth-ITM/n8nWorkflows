Template for Google Drive and Google Docs with clear structure and purpose

https://n8nworkflows.xyz/workflows/template-for-google-drive-and-google-docs-with-clear-structure-and-purpose-7178


# Template for Google Drive and Google Docs with clear structure and purpose

### 1. Workflow Overview

This workflow automates the process of transferring images stored in a specific Google Drive folder into a Google Docs document, resizing them appropriately before insertion. It targets use cases where multiple images need to be batch-processed and embedded into a document, such as report generation, documentation, or content management workflows.

The workflow’s logical structure is divided into the following blocks:

- **1.1 Trigger and File Retrieval**: Starts manually and retrieves all image files from a designated Google Drive folder.
- **1.2 Batch Processing and Download**: Processes files in batches to avoid timeout issues, downloading each image file.
- **1.3 URI Filtering and Sorting**: Sorts the images numerically by filename and generates direct download URIs.
- **1.4 Image Resizing Preparation**: Prepares image insertion requests with size parameters based on filename conventions.
- **1.5 Image Insertion into Google Docs**: Inserts resized images into the specified Google Docs document via Google Docs API.
- **1.6 Batch Loop Control**: Uses batch loops and wait timers to pace the process and prevent timeouts or API limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and File Retrieval

- **Overview:**  
  This block initiates the workflow manually and searches a specified Google Drive folder for files to process.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Search File in Google Drive  
  - Batch Wait Timer-1

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user action (manual execution).  
    - Config: No parameters, simple trigger.  
    - Connections: Outputs to *Search File in Google Drive*.  
    - Potential Failures: None typical; user must trigger manually.

  - **Search File in Google Drive**  
    - Type: Google Drive (File/Folder resource)  
    - Role: Retrieves all files in a specific Google Drive folder.  
    - Config: Uses folder ID `1bMnN5da0iMxMPv1IgSlsRvXgdt5PGq84` (replaceable with user folder). Returns all files.  
    - Credentials: Google Drive OAuth2 credential configured (`garrynacov`).  
    - Connections: Outputs to *Batch Wait Timer-1*.  
    - Edge Cases: Folder ID invalid or inaccessible; no files found; Google Drive API limits.

  - **Batch Wait Timer-1**  
    - Type: Wait node  
    - Role: Acts as a buffer before download step to manage API pacing.  
    - Config: Default wait, no parameters set.  
    - Connections: Outputs to *Download File*.  
    - Edge Cases: None critical; ensures timing control.

#### 2.2 Batch Processing and Download

- **Overview:**  
  Processes files in batches to download each image file while managing rate limits and avoiding timeouts.

- **Nodes Involved:**  
  - Download File  
  - Batch Wait Timer-2  
  - Filtering URI  
  - Batch Wait Timer-3  
  - Resize Image  
  - Loop Over Items  
  - Insert Image to Google Doc

- **Node Details:**

  - **Download File**  
    - Type: Google Drive  
    - Role: Downloads each image file by ID.  
    - Config: File ID taken dynamically from input item (`{{$json.id}}`), filename passed as option.  
    - Credentials: Same Google Drive OAuth2 credential as above.  
    - Connections: Outputs to *Batch Wait Timer-2*.  
    - Edge Cases: File not found, access denied, download failure, large file causing timeout.

  - **Batch Wait Timer-2**  
    - Type: Wait node  
    - Role: Adds delay after download to prevent throttling.  
    - Config: Waits 3 seconds.  
    - Connections: Outputs to *Filtering URI*.  
    - Edge Cases: None significant.

  - **Filtering URI** (Code node)  
    - Type: Code (JavaScript)  
    - Role: Sorts image items in descending numeric order extracted from filenames; generates direct view URIs for images.  
    - Config: Custom JS sorts on numeric part of filename, constructs URI `https://drive.google.com/uc?export=view&id=<imageId>`.  
    - Connections: Outputs to *Batch Wait Timer-3*.  
    - Edge Cases: Files without numeric parts in name default to 0; malformed names may affect sorting.

  - **Batch Wait Timer-3**  
    - Type: Wait node  
    - Role: Waits 3 seconds before resizing images, pacing the workflow.  
    - Connections: Outputs to *Resize Image*.  
    - Edge Cases: None significant.

  - **Resize Image** (Code node)  
    - Type: Code (JavaScript)  
    - Role: Prepares Google Docs API request payloads to insert inline images with size parameters.  
    - Config: Checks if filename contains "thumb" to assign smaller size (200x150 pts), else larger size (400x250 pts). Constructs `insertInlineImage` requests with URI and size.  
    - Connections: Outputs to *Loop Over Items* (second output).  
    - Edge Cases: No filename field, fallback to empty string; unexpected filename conventions.

  - **Loop Over Items** (SplitInBatches node)  
    - Type: Split in Batches  
    - Role: Splits the image requests into batches for insertion, preventing timeouts.  
    - Config: Default batch size (not explicitly set, can be configured).  
    - Connections:  
      - First output: Empty (not used).  
      - Second output: To *Insert Image to Google Doc*.  
    - Edge Cases: Batch size too large may cause timeout; too small may slow processing.

  - **Insert Image to Google Doc** (HTTP Request node)  
    - Type: HTTP Request  
    - Role: Sends batchUpdate requests to Google Docs API to insert images.  
    - Config:  
      - URL: Google Docs API endpoint with hardcoded document ID: `1IkozrnIqirwn4mwbHdMl25f013P59Nvd-Fw8kg_Eyw4` (replaceable).  
      - Method: POST with body containing requests from previous node.  
      - Authentication: Uses Google Docs OAuth2 credential (`Google Docs account`).  
      - Timeout: 10 seconds.  
    - Connections: Outputs back to *Loop Over Items* (first output) to continue processing batches.  
    - Edge Cases: API quota exceeded, invalid document ID, network errors, malformed requests.

#### 2.3 Sticky Notes (Documentation and Guidance)

- **Overview:**  
  Sticky notes provide inline documentation and setup instructions visible in the workflow editor for user reference.

- **Nodes Involved:** All sticky note nodes:

  - **Sticky Note** (-1000,100)  
    - Content: Overview about filtering URI and image ID retrieval.

  - **Sticky Note1** (-240,100)  
    - Content: Brief note "Sending Images to Docs" with instruction "Double click".

  - **Sticky Note2** (-1000,-240)  
    - Content: Describes trigger and image download process.

  - **Sticky Note3** (-1380,-240)  
    - Content: Detailed “How it works” explanation with five steps outlining the workflow logic.

  - **Sticky Note4** (-1380,100)  
    - Content: Setup steps including credential connections, folder and document ID replacement instructions, and batch loop explanation.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                           | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                           |
|----------------------------|---------------------|-----------------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Starts the workflow manually             | —                           | Search File in Google Drive   |                                                                                                     |
| Search File in Google Drive | Google Drive        | Retrieves all files from target folder  | When clicking ‘Execute workflow’ | Batch Wait Timer-1            | Setup instructions about Folder & File Filter in Sticky Note4                                       |
| Batch Wait Timer-1          | Wait                | Timing buffer before download            | Search File in Google Drive  | Download File                |                                                                                                     |
| Download File              | Google Drive        | Downloads each image file by ID          | Batch Wait Timer-1           | Batch Wait Timer-2            | Trigger and Download Image from Google Drive (Sticky Note2)                                         |
| Batch Wait Timer-2          | Wait                | Adds delay after download                 | Download File                | Filtering URI                |                                                                                                     |
| Filtering URI              | Code                | Sorts files numerically and generates view URI | Batch Wait Timer-2           | Batch Wait Timer-3            | Filtering URI And Size Image (Sticky Note)                                                         |
| Batch Wait Timer-3          | Wait                | Adds delay before resizing                | Filtering URI                | Resize Image                 |                                                                                                     |
| Resize Image               | Code                | Prepares Google Docs image insertion requests with sizing | Batch Wait Timer-3           | Loop Over Items (second output) | Sending Images to Docs (Sticky Note1)                                                               |
| Loop Over Items            | Split In Batches    | Splits images into manageable batches    | Insert Image to Google Doc (loop back) | Insert Image to Google Doc    | How it works & Batch Loop explanation in Sticky Note3 and Sticky Note4                              |
| Insert Image to Google Doc | HTTP Request        | Inserts images into Google Docs document | Loop Over Items              | Loop Over Items (first output) | Setup instructions about Google Docs Document ID, Credentials, and batch processing in Sticky Note4 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manual start of workflow.

2. **Add Google Drive node to search files:**  
   - Name: `Search File in Google Drive`  
   - Resource: File/Folder  
   - Operation: List files  
   - Folder ID: Set to your target Google Drive folder ID (e.g., `1bMnN5da0iMxMPv1IgSlsRvXgdt5PGq84`)  
   - Return All: Yes  
   - Credentials: Connect Google Drive OAuth2 credential  
   - Connect output of manual trigger to this node.

3. **Add a Wait node:**  
   - Name: `Batch Wait Timer-1`  
   - Default configuration (no delay parameters set)  
   - Connect output of Search File node to this.

4. **Add Google Drive node to download files:**  
   - Name: `Download File`  
   - Operation: Download file  
   - File ID: Use expression `{{$json.id}}` dynamically from input  
   - File Name: Use expression `{{$json.name}}` to preserve name (optional)  
   - Credentials: Same as Search File  
   - Connect output of Batch Wait Timer-1 to here.

5. **Add Wait node:**  
   - Name: `Batch Wait Timer-2`  
   - Set `Amount` to 3 seconds  
   - Connect output of Download File to here.

6. **Add Code node for URI filtering and sorting:**  
   - Name: `Filtering URI`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     return items
       .sort((a, b) => {
         const getNumber = name => parseInt(name?.json?.name?.match(/\d+/)?.[0] || 0, 10);
         return getNumber(b) - getNumber(a); // Descending order
       })
       .map(item => ({
         json: {
           uri: `https://drive.google.com/uc?export=view&id=${item.json.id}`
         }
       }));
     ```  
   - Connect output of Batch Wait Timer-2 to here.

7. **Add Wait node:**  
   - Name: `Batch Wait Timer-3`  
   - Set `Amount` to 3 seconds  
   - Connect output of Filtering URI to here.

8. **Add Code node to prepare image insertion payload:**  
   - Name: `Resize Image`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     return items.map(item => {
       const name = item.json.name || "";
       const isThumbnail = name.includes("thumb");
       const width = isThumbnail ? 200 : 400;
       const height = isThumbnail ? 150 : 250;
       return {
         json: {
           requests: [{
             insertInlineImage: {
               uri: item.json.uri,
               location: { index: 1 },
               objectSize: {
                 height: { magnitude: height, unit: 'PT' },
                 width: { magnitude: width, unit: 'PT' }
               }
             }
           }]
         }
       };
     });
     ```  
   - Connect output of Batch Wait Timer-3 to here.

9. **Add SplitInBatches node:**  
   - Name: `Loop Over Items`  
   - Configure batch size as desired (default or adjust for your needs)  
   - Connect output of Resize Image to the second output of this node (main output 1 is empty).  

10. **Add HTTP Request node to insert images into Google Docs:**  
    - Name: `Insert Image to Google Doc`  
    - Method: POST  
    - URL: `https://docs.googleapis.com/v1/documents/{DOCUMENT_ID}:batchUpdate`  
      Replace `{DOCUMENT_ID}` with your Google Docs document ID.  
    - Authentication: Google Docs OAuth2 credential  
    - Body Parameters:  
      - Name: `requests`  
      - Value: Expression `{{$json.requests}}` (from the Resize Image node)  
    - Timeout: 10000 ms  
    - Connect main output of SplitInBatches node (Loop Over Items) to this node.

11. **Connect output of HTTP Request node back to SplitInBatches node:**  
    - This loop allows batch processing to continue until all images are inserted.

12. **Credentials Setup:**  
    - Google Drive OAuth2 credential with access to your Drive folder.  
    - Google Docs OAuth2 credential with access to your target document.

13. **Replace placeholders:**  
    - Folder ID in Search File node.  
    - Document ID in Insert Image to Google Doc node.

14. **Test the workflow:**  
    - Run manually.  
    - Monitor progress and logs for errors or timeouts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow includes batch processing and wait timers to avoid Google API rate limits and timeouts when handling multiple images.  | Important for large image sets to ensure reliable execution.                                             |
| Replace folder and document IDs with your own to adapt the workflow.                                                                  | See Sticky Note4 for setup instructions.                                                                 |
| Google OAuth2 credentials must be configured in n8n prior to running the workflow.                                                     | Credentials for Google Drive and Google Docs required.                                                   |
| Direct image URIs use the URL scheme: `https://drive.google.com/uc?export=view&id=<fileId>`.                                           | This allows Google Docs API to fetch images directly.                                                    |
| Naming convention impacts image size: filenames containing “thumb” are treated as thumbnails and resized smaller.                      | Customizable in the Resize Image code node.                                                              |
| For detailed Google Docs API documentation on batchUpdate requests, see: https://developers.google.com/docs/api/reference/rest/v1/documents/batchUpdate | Useful reference for modifying or extending image insertion logic.                                       |

---

**Disclaimer:** This document is generated from an n8n workflow automation. It adheres strictly to content policies and handles only public and legal data.