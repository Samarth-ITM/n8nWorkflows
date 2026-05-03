Merge Google Drive PDFs with dynamic cover pages and watermark using Autype

https://n8nworkflows.xyz/workflows/merge-google-drive-pdfs-with-dynamic-cover-pages-and-watermark-using-autype-13896


# Merge Google Drive PDFs with dynamic cover pages and watermark using Autype

# 1. Workflow Overview

This workflow takes all PDF files from a specific Google Drive folder, generates a custom title page for each file, merges each title page in front of its corresponding PDF, applies a company watermark to the final combined document, and uploads the result back to Google Drive.

Typical use cases include:
- Building a consolidated document package from many PDFs
- Creating branded document bundles with standardized cover pages
- Producing archive packs with metadata-based separators
- Preparing client-facing or internal review binders

The workflow is organized into the following logical blocks.

## 1.1 Trigger and Source File Discovery
The workflow starts manually, then queries a Google Drive folder to retrieve all files that will be used to build title pages and later downloaded for merging.

## 1.2 Title Page Definition and Rendering
A Code node transforms the Google Drive file metadata into an Autype rendering configuration. Autype then renders all title pages as a single multi-page PDF, and that generated PDF is uploaded into Autype’s file storage so individual pages can later be extracted.

## 1.3 Per-Document Processing Loop
The workflow creates one loop item per source PDF, carrying the Drive file ID, file name, title page index, and the uploaded title-pages PDF ID. It then iterates through each document, downloading the source PDF, uploading it to Autype, extracting the corresponding title page from the generated title-pages PDF, and collecting the two file IDs as a merge pair.

## 1.4 Final Merge, Watermark, and Storage
After the loop completes, the workflow builds an ordered interleaved merge list, merges everything into one PDF, applies a watermark, and uploads the final PDF to Google Drive.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Source File Discovery

**Overview:**  
This block starts the workflow manually and lists the files from a chosen Google Drive folder. The returned Google Drive metadata becomes the source of truth for title page content and subsequent file downloads.

**Nodes Involved:**  
- Run Workflow
- List PDFs in Folder

### Node Details

#### Run Workflow
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for testing or on-demand execution.
- **Configuration choices:** No parameters are configured. It simply starts the workflow when the user clicks execute.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - No input node
  - Outputs to `List PDFs in Folder`
- **Version-specific requirements:** Type version 1; standard manual trigger behavior.
- **Edge cases or potential failure types:** None at node level. Main limitation is that it is not automated; the workflow runs only when manually triggered.
- **Sub-workflow reference:** None.

#### List PDFs in Folder
- **Type and technical role:** `n8n-nodes-base.googleDrive`; lists files/folders from Google Drive.
- **Configuration choices:**  
  - Resource: `fileFolder`
  - Folder filter enabled using a fixed folder ID placeholder: `YOUR_FOLDER_ID`
  - `returnAll: true`, so it fetches all matching items
  - Additional fields set to request `*`, meaning broad metadata is requested
- **Key expressions or variables used:** None in the current setup; folder ID is static.
- **Input and output connections:**  
  - Input from `Run Workflow`
  - Output to `Build Title Pages JSON`
- **Version-specific requirements:** Type version 3 of the Google Drive node.
- **Edge cases or potential failure types:**  
  - Invalid Google Drive OAuth2 credentials
  - Folder ID not found or inaccessible
  - Large folder contents can increase execution time
  - Despite the node name, there is no explicit MIME type filter in the configuration shown; if the folder contains non-PDF files, they may also be returned unless filtered elsewhere
  - Missing metadata fields such as `owners`, `createdTime`, or `modifiedTime`
- **Sub-workflow reference:** None.

---

## 2.2 Title Page Definition and Rendering

**Overview:**  
This block converts the list of Google Drive files into an Autype render configuration that creates one title page per document. The rendered title-page PDF is then uploaded to Autype so its individual pages can be extracted during the loop.

**Nodes Involved:**  
- Build Title Pages JSON
- Render Title Pages PDF
- Upload Title Pages PDF

### Node Details

#### Build Title Pages JSON
- **Type and technical role:** `n8n-nodes-base.code`; transforms Drive metadata into the JSON configuration required by Autype’s rendering endpoint.
- **Configuration choices:**  
  - Uses JavaScript to collect all incoming items: `$input.all().map(i => i.json)`
  - Builds one `section` per file with:
    - `type: 'page'`
    - `align: 'center'`
    - Content elements including:
      - Document title from file name with `.pdf` suffix removed
      - Created date
      - Last modified date
      - Owner display name
      - Document position indicator such as `Document 2 of 5`
  - Builds final config with:
    - `document.type = 'pdf'`
    - `document.size = 'A4'`
    - `sections = [...]`
  - Returns a single item containing:
    - `config`: stringified JSON
    - `files`: simplified array with `{ id, name, index }`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `file.name.replace(/\.pdf$/i, '')`
  - `new Date(file.createdTime).toLocaleDateString('en-US', ...)`
  - `new Date(file.modifiedTime).toLocaleDateString('en-US', ...)`
  - Fallbacks to `'Unknown'`
- **Input and output connections:**  
  - Input from `List PDFs in Folder`
  - Output to `Render Title Pages PDF`
- **Version-specific requirements:** Type version 2 of Code node.
- **Edge cases or potential failure types:**  
  - Empty folder results in `sections = []`; downstream rendering may fail or produce an empty PDF
  - Non-PDF file names will still be processed; only the `.pdf` suffix removal is conditional
  - Invalid or missing date fields could produce `Unknown`
  - Localized date formatting depends on runtime support for `toLocaleDateString`
  - Very large file counts may create a large config payload
- **Sub-workflow reference:** None.

#### Render Title Pages PDF
- **Type and technical role:** `n8n-nodes-autype.autype`; renders a PDF using an Autype configuration.
- **Configuration choices:**  
  - `config` is provided from the previous node with expression: `{{ $json.config }}`
  - `downloadOutput: true`, meaning the rendered PDF binary is returned for immediate reuse
- **Key expressions or variables used:**  
  - `={{ $json.config }}`
- **Input and output connections:**  
  - Input from `Build Title Pages JSON`
  - Output to `Upload Title Pages PDF`
- **Version-specific requirements:**  
  - Type version 1 of the Autype community node
  - Requires the `n8n-nodes-autype` community node installed on self-hosted n8n
  - Requires valid Autype API credentials
- **Edge cases or potential failure types:**  
  - Invalid Autype API key
  - Malformed config JSON
  - Render API limits or timeout on very large title-page batches
  - If zero sections are provided, output may fail or be unusable
- **Sub-workflow reference:** None.

#### Upload Title Pages PDF
- **Type and technical role:** `n8n-nodes-autype.autype`; uploads a file into Autype’s file storage/tooling context.
- **Configuration choices:**  
  - Resource: `file`
  - No additional parameters shown, so it relies on incoming binary from the render step
- **Key expressions or variables used:** None explicitly.
- **Input and output connections:**  
  - Input from `Render Title Pages PDF`
  - Output to `Prepare Loop Items`
- **Version-specific requirements:** Same Autype node requirements as above.
- **Edge cases or potential failure types:**  
  - Missing binary data from prior render
  - Upload size limits
  - Authentication failure
  - Unexpected binary property naming if the community node expects a specific field
- **Sub-workflow reference:** None.

---

## 2.3 Per-Document Processing Loop

**Overview:**  
This block prepares one loop item per original Google Drive file, then processes each file individually. For every document, it downloads the source PDF, uploads it to Autype, extracts the matching title page from the generated title-page PDF, and stores both file IDs so the final merge can preserve the correct order.

**Nodes Involved:**  
- Prepare Loop Items
- Loop Over Documents
- Download PDF from Drive
- Upload Document to Autype
- Extract Title Page
- Collect Merge Pair

### Node Details

#### Prepare Loop Items
- **Type and technical role:** `n8n-nodes-base.code`; converts the single title-page upload result plus stored file list into one item per document.
- **Configuration choices:**  
  - Reads the uploaded title-pages file ID from the current input:
    - `const titlePagesFileId = $input.first().json.id;`
  - Reads original files array from `Build Title Pages JSON`:
    - `$('Build Title Pages JSON').first().json.files`
  - Emits one item per file with:
    - `driveFileId`
    - `fileName`
    - `titlePageNumber` as `index + 1`
    - `titlePagesFileId`
- **Key expressions or variables used:**  
  - `$input.first().json.id`
  - `$('Build Title Pages JSON').first().json.files`
- **Input and output connections:**  
  - Input from `Upload Title Pages PDF`
  - Output to `Loop Over Documents`
- **Version-specific requirements:** Type version 2 of Code node.
- **Edge cases or potential failure types:**  
  - If title-pages upload fails or returns no `id`, extraction later cannot work
  - If `Build Title Pages JSON` returned empty files, this emits no items
  - Cross-node expression references require node names to remain unchanged
- **Sub-workflow reference:** None.

#### Loop Over Documents
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; loops through document items one by one.
- **Configuration choices:**  
  - Default options only; no custom batch size is shown
  - In this pattern, the node has:
    - One output feeding the per-document processing branch
    - A loop-back connection from `Collect Merge Pair` into the node to request the next item
    - A completion path from the node to `Build Final Merge List`
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  - Input from `Prepare Loop Items`
  - Main processing output to `Download PDF from Drive`
  - Completion output to `Build Final Merge List`
  - Loop-back input from `Collect Merge Pair`
- **Version-specific requirements:** Type version 3.
- **Edge cases or potential failure types:**  
  - Incorrect loop wiring can cause infinite loops or premature completion
  - Empty input list causes the completion branch to run immediately
  - If any item fails mid-loop and error handling is not configured, execution stops before final merge
- **Sub-workflow reference:** None.

#### Download PDF from Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`; downloads the current file from Google Drive as binary.
- **Configuration choices:**  
  - Operation: `download`
  - File ID is dynamic: `{{ $json.driveFileId }}`
- **Key expressions or variables used:**  
  - `={{ $json.driveFileId }}`
- **Input and output connections:**  
  - Input from `Loop Over Documents`
  - Output to `Upload Document to Autype`
- **Version-specific requirements:** Google Drive node version 3.
- **Edge cases or potential failure types:**  
  - File deleted or moved after initial listing
  - Permission denied
  - Google API quota or transient network failure
  - Non-PDF content can still be downloaded if the folder contained other file types
- **Sub-workflow reference:** None.

#### Upload Document to Autype
- **Type and technical role:** `n8n-nodes-autype.autype`; uploads the current source file to Autype for downstream document tool operations.
- **Configuration choices:**  
  - Resource: `file`
  - Expects binary file input from Google Drive download
- **Key expressions or variables used:** None explicitly.
- **Input and output connections:**  
  - Input from `Download PDF from Drive`
  - Output to `Extract Title Page`
- **Version-specific requirements:** Autype community node, type version 1.
- **Edge cases or potential failure types:**  
  - Missing binary data
  - Unsupported file type if non-PDF files entered the flow
  - API size limits
  - Authentication failure
- **Sub-workflow reference:** None.

#### Extract Title Page
- **Type and technical role:** `n8n-nodes-autype.autype`; extracts a single page from the previously generated title-pages PDF.
- **Configuration choices:**  
  - Resource: `documentTools`
  - Operation: `pages`
  - `fileId` comes from the prepared loop item: the uploaded title-pages PDF ID
  - `pages` is the current title page number converted to string
- **Key expressions or variables used:**  
  - `={{ $('Loop Over Documents').item.json.titlePageNumber.toString() }}`
  - `={{ $('Loop Over Documents').item.json.titlePagesFileId }}`
- **Input and output connections:**  
  - Input from `Upload Document to Autype`
  - Output to `Collect Merge Pair`
- **Version-specific requirements:** Autype node version 1.
- **Edge cases or potential failure types:**  
  - If the title-pages PDF has fewer pages than expected, extraction fails
  - Cross-node `.item` references depend on loop execution context
  - If the uploaded title-pages file ID is invalid or expired, extraction fails
- **Sub-workflow reference:** None.

#### Collect Merge Pair
- **Type and technical role:** `n8n-nodes-base.code`; collects the Autype file IDs needed for final interleaved merging.
- **Configuration choices:**  
  - Reads:
    - `titlePageFileId` from current node output (`$json.outputFileId`)
    - `docFileId` from `Upload Document to Autype`
    - `fileName` from `Loop Over Documents`
  - Returns a single item containing those three fields
- **Key expressions or variables used:**  
  - `$json.outputFileId`
  - `$('Upload Document to Autype').item.json.id`
  - `$('Loop Over Documents').item.json.fileName`
- **Input and output connections:**  
  - Input from `Extract Title Page`
  - Output loops back to `Loop Over Documents`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**  
  - Missing `outputFileId` from extraction
  - Cross-node `.item` references break if node names change or execution context shifts
  - File name is currently collected but not used downstream except as possible metadata/debugging
- **Sub-workflow reference:** None.

---

## 2.4 Final Merge, Watermark, and Storage

**Overview:**  
This block runs after the loop has finished and all merge pairs are available. It builds the final ordered list of file IDs, merges all pages into one PDF, applies a watermark, and saves the final result to Google Drive.

**Nodes Involved:**  
- Build Final Merge List
- Merge All PDFs
- Add Company Watermark
- Save Final PDF to Drive

### Node Details

#### Build Final Merge List
- **Type and technical role:** `n8n-nodes-base.code`; aggregates all collected merge pairs into the exact order required by the merge operation.
- **Configuration choices:**  
  - Reads all incoming items with `$input.all()`
  - Builds an array `fileIds`
  - For each item, appends:
    1. `titlePageFileId`
    2. `docFileId`
  - Returns one item with a comma-separated `fileIds` string
- **Key expressions or variables used:**  
  - `$input.all()`
  - `fileIds.join(',')`
- **Input and output connections:**  
  - Input from the completion path of `Loop Over Documents`
  - Output to `Merge All PDFs`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**  
  - If no items arrive, resulting merge string may be empty
  - Assumes all prior loop items completed successfully
  - Ordering is based on loop completion order, which in this design should match input sequence because processing is sequential
- **Sub-workflow reference:** None.

#### Merge All PDFs
- **Type and technical role:** `n8n-nodes-autype.autype`; merges multiple uploaded PDFs in the provided order.
- **Configuration choices:**  
  - Resource: `documentTools`
  - `fileIds` is passed as a comma-separated string from prior node
- **Key expressions or variables used:**  
  - `={{ $json.fileIds }}`
- **Input and output connections:**  
  - Input from `Build Final Merge List`
  - Output to `Add Company Watermark`
- **Version-specific requirements:** Autype node version 1.
- **Edge cases or potential failure types:**  
  - Empty or malformed `fileIds` string
  - One or more Autype file IDs missing, invalid, or no longer available
  - Merge size/page-count limits
- **Sub-workflow reference:** None.

#### Add Company Watermark
- **Type and technical role:** `n8n-nodes-autype.autype`; applies a text watermark across the merged PDF and returns the resulting binary.
- **Configuration choices:**  
  - Resource: `documentTools`
  - Operation: `watermark`
  - `fileId` comes from `Merge All PDFs.outputFileId`
  - Watermark text: `Your Company Name`
  - `downloadOutput: true` so the final watermarked PDF is returned as binary for upload
  - Watermark styling:
    - Color: `#2563EB`
    - Opacity: `0.6`
    - Font size: `14`
    - Rotation: `45`
- **Key expressions or variables used:**  
  - `={{ $('Merge All PDFs').item.json.outputFileId }}`
- **Input and output connections:**  
  - Input from `Merge All PDFs`
  - Output to `Save Final PDF to Drive`
- **Version-specific requirements:** Autype node version 1.
- **Edge cases or potential failure types:**  
  - Missing `outputFileId` from merge result
  - Watermark rendering limits for very large PDFs
  - Text may not match intended placement exactly; the sticky note says “top-right area” but the configured parameters only explicitly show rotation, opacity, color, and font size
- **Sub-workflow reference:** None.

#### Save Final PDF to Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads the final watermarked PDF back to Google Drive.
- **Configuration choices:**  
  - Target drive: `My Drive`
  - Target folder ID: `YOUR_FOLDER_ID`
  - File name pattern: `merged-documents-YYYY-MM-DD.pdf`
  - Uses expression based on `$now.format('yyyy-MM-dd')`
  - The node configuration implies upload/create behavior using incoming binary
- **Key expressions or variables used:**  
  - `=merged-documents-{{ $now.format('yyyy-MM-dd') }}.pdf`
- **Input and output connections:**  
  - Input from `Add Company Watermark`
  - No downstream node
- **Version-specific requirements:** Google Drive node version 3.
- **Edge cases or potential failure types:**  
  - Invalid folder permissions
  - Duplicate naming behavior depends on node defaults and Drive behavior
  - Missing binary data from watermark node
  - Large file upload limits or timeouts
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Workflow | Manual Trigger | Manual start of the workflow |  | List PDFs in Folder | ## Merge All PDFs from Google Drive with Generated Title Pages and Watermark<br>### This workflow reads every PDF from a Google Drive folder, generates a title page for each document (showing name, dates, and owner), merges everything into a single PDF with interleaved title pages, adds a company watermark, and uploads the result back to Drive.<br><br>Click "Test Workflow" to run it once. Each document gets a professional cover sheet before its content.<br><br>### How it works<br>1. **List PDFs in Folder** — Searches a Google Drive folder for all PDF files.<br>2. **Build Title Pages JSON** — A Code node creates an Autype Render JSON config with one page section per document (name, created date, modified date, owner, document number).<br>3. **Render Title Pages PDF** — Autype renders all title pages as a single multi-page PDF in one API call.<br>4. **Upload Title Pages PDF** — Uploads the rendered PDF to Autype Tools for page extraction.<br>5. **Loop Over Documents** — For each PDF:<br>   - Downloads from Google Drive<br>   - Uploads to Autype Tools<br>   - Extracts the corresponding title page from the title pages PDF<br>   - Collects the title page file ID and document file ID as a merge pair<br>6. **Build Final Merge List** — Interleaves file IDs: title page 1, doc 1, title page 2, doc 2, etc.<br>7. **Merge All PDFs** — Combines everything into one document in order.<br>8. **Add Company Watermark** — Stamps the company name in blue on every page (top-right area, 60% opacity).<br>9. **Save Final PDF to Drive** — Uploads the result as `merged-documents-YYYY-MM-DD.pdf` to the same Google Drive folder.<br><br>### Requirements<br>* **Autype account** — Sign up at [app.autype.com](https://app.autype.com) and go to **Settings → API Keys** to generate your API key.<br>* **Google Drive** — Connect your Google account via OAuth2 in n8n credentials.<br>* **n8n-nodes-autype** — Install the Autype community node via **Settings → Community Nodes** in your self-hosted n8n instance.<br><br>> ⚠️ Community node: requires self-hosted n8n. Not available on n8n Cloud. |
| List PDFs in Folder | Google Drive | List source files from a Drive folder | Run Workflow | Build Title Pages JSON | ### 1. List & Generate Title Pages<br>Searches the Google Drive folder for all PDFs. A Code node builds the Autype Render JSON with one title page section per document. Autype renders all title pages in a single API call as one multi-page PDF. |
| Build Title Pages JSON | Code | Build Autype render config and preserve file list metadata | List PDFs in Folder | Render Title Pages PDF | ### 1. List & Generate Title Pages<br>Searches the Google Drive folder for all PDFs. A Code node builds the Autype Render JSON with one title page section per document. Autype renders all title pages in a single API call as one multi-page PDF. |
| Render Title Pages PDF | Autype | Render all title pages into one PDF | Build Title Pages JSON | Upload Title Pages PDF | ### 1. List & Generate Title Pages<br>Searches the Google Drive folder for all PDFs. A Code node builds the Autype Render JSON with one title page section per document. Autype renders all title pages in a single API call as one multi-page PDF. |
| Upload Title Pages PDF | Autype | Upload rendered title-pages PDF to Autype storage/tools | Render Title Pages PDF | Prepare Loop Items |  |
| Prepare Loop Items | Code | Create one loop item per document with page-number mapping | Upload Title Pages PDF | Loop Over Documents |  |
| Loop Over Documents | Split In Batches | Iterate through each document and emit completion branch after all items | Prepare Loop Items; Collect Merge Pair | Build Final Merge List; Download PDF from Drive |  |
| Download PDF from Drive | Google Drive | Download current source PDF | Loop Over Documents | Upload Document to Autype | ### 2. Loop: Download, Upload, Extract Title Page<br>For each document: download from Drive, upload to Autype, extract the matching title page (by page number) from the title pages PDF, and collect both file IDs as a merge pair. |
| Upload Document to Autype | Autype | Upload current document to Autype | Download PDF from Drive | Extract Title Page | ### 2. Loop: Download, Upload, Extract Title Page<br>For each document: download from Drive, upload to Autype, extract the matching title page (by page number) from the title pages PDF, and collect both file IDs as a merge pair. |
| Extract Title Page | Autype | Extract matching title page from generated title-pages PDF | Upload Document to Autype | Collect Merge Pair | ### 2. Loop: Download, Upload, Extract Title Page<br>For each document: download from Drive, upload to Autype, extract the matching title page (by page number) from the title pages PDF, and collect both file IDs as a merge pair. |
| Collect Merge Pair | Code | Store title-page and document file IDs for final merge | Extract Title Page | Loop Over Documents | ### 2. Loop: Download, Upload, Extract Title Page<br>For each document: download from Drive, upload to Autype, extract the matching title page (by page number) from the title pages PDF, and collect both file IDs as a merge pair. |
| Build Final Merge List | Code | Interleave title-page and document file IDs into merge order | Loop Over Documents | Merge All PDFs | ### 3. Merge, Watermark & Save<br>After the loop, file IDs are interleaved (title1, doc1, title2, doc2...) and merged into one PDF. A blue company-name watermark is added to every page. The final PDF is uploaded to Google Drive. |
| Merge All PDFs | Autype | Merge all title pages and source PDFs into one document | Build Final Merge List | Add Company Watermark | ### 3. Merge, Watermark & Save<br>After the loop, file IDs are interleaved (title1, doc1, title2, doc2...) and merged into one PDF. A blue company-name watermark is added to every page. The final PDF is uploaded to Google Drive. |
| Add Company Watermark | Autype | Apply text watermark to merged PDF | Merge All PDFs | Save Final PDF to Drive | ### 3. Merge, Watermark & Save<br>After the loop, file IDs are interleaved (title1, doc1, title2, doc2...) and merged into one PDF. A blue company-name watermark is added to every page. The final PDF is uploaded to Google Drive. |
| Save Final PDF to Drive | Google Drive | Upload final watermarked PDF to Google Drive | Add Company Watermark |  | ### 3. Merge, Watermark & Save<br>After the loop, file IDs are interleaved (title1, doc1, title2, doc2...) and merged into one PDF. A blue company-name watermark is added to every page. The final PDF is uploaded to Google Drive. |
| Sticky Note | Sticky Note | Documentation note on overall workflow |  |  |  |
| Sticky Note1 | Sticky Note | Documentation note for title-page generation block |  |  |  |
| Sticky Note2 | Sticky Note | Documentation note for per-document loop block |  |  |  |
| Sticky Note3 | Sticky Note | Documentation note for final merge block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Merge all PDFs from a Google Drive folder with generated title pages and watermark using Autype`.
   - Ensure your n8n instance is self-hosted if you plan to use the Autype community node.

2. **Install the Autype community node**
   - In n8n, go to **Settings → Community Nodes**.
   - Install `n8n-nodes-autype`.
   - This is required because the workflow depends on the `n8n-nodes-autype.autype` node.
   - This community node is not available on n8n Cloud.

3. **Create credentials**
   - **Google Drive OAuth2**
     - Create or select Google OAuth2 credentials in n8n.
     - Grant access to the Drive containing the source and destination folder.
   - **Autype API**
     - Create an Autype account at [https://app.autype.com](https://app.autype.com).
     - Generate an API key in **Settings → API Keys**.
     - Add it to n8n as Autype credentials.

4. **Add the entry node**
   - Create a **Manual Trigger** node.
   - Rename it to **Run Workflow**.

5. **Add the Google Drive listing node**
   - Create a **Google Drive** node and rename it **List PDFs in Folder**.
   - Configure:
     - Resource: `File/Folder`
     - Operation: listing/search equivalent for files in a folder
     - Folder filter: set the folder ID to your source folder
     - Return All: enabled
     - Fields: include all metadata if available (`*`)
   - Attach Google Drive credentials.
   - Connect **Run Workflow → List PDFs in Folder**.

6. **Add the title-page configuration builder**
   - Create a **Code** node named **Build Title Pages JSON**.
   - Paste this logic conceptually:
     - Read all listed files
     - For each file, build one Autype section representing a centered page
     - Include:
       - Title from file name without `.pdf`
       - Created date
       - Modified date
       - Owner display name
       - Position string such as `Document X of Y`
     - Build the final Autype config object with PDF/A4 settings
     - Return:
       - `config` as a stringified JSON
       - `files` as a simplified mapping array
   - Connect **List PDFs in Folder → Build Title Pages JSON**.

7. **Add the Autype render node**
   - Create an **Autype** node named **Render Title Pages PDF**.
   - Configure:
     - Use the render/config mode supported by the Autype node
     - Set `config` to `{{ $json.config }}`
     - Enable `downloadOutput`
   - Attach Autype credentials.
   - Connect **Build Title Pages JSON → Render Title Pages PDF**.

8. **Add the Autype upload node for title pages**
   - Create another **Autype** node named **Upload Title Pages PDF**.
   - Configure:
     - Resource: `file`
   - This node should consume the binary output from the render node and upload it to Autype.
   - Connect **Render Title Pages PDF → Upload Title Pages PDF**.

9. **Add the loop-item preparation node**
   - Create a **Code** node named **Prepare Loop Items**.
   - Configure it to:
     - Read the uploaded title-pages PDF file ID from the input item
     - Read the `files` array from **Build Title Pages JSON**
     - Return one item per source file with:
       - `driveFileId`
       - `fileName`
       - `titlePageNumber` = index + 1
       - `titlePagesFileId`
   - Connect **Upload Title Pages PDF → Prepare Loop Items**.

10. **Add the loop controller**
    - Create a **Split In Batches** node named **Loop Over Documents**.
    - Keep default options unless you want to customize batch size.
    - Connect **Prepare Loop Items → Loop Over Documents**.

11. **Add the Drive download node**
    - Create a **Google Drive** node named **Download PDF from Drive**.
    - Configure:
      - Operation: `download`
      - File ID: `{{ $json.driveFileId }}`
    - Attach Google Drive credentials.
    - Connect **Loop Over Documents → Download PDF from Drive**.

12. **Add the Autype upload node for current document**
    - Create an **Autype** node named **Upload Document to Autype**.
    - Configure:
      - Resource: `file`
    - It should upload the binary PDF downloaded from Drive.
    - Connect **Download PDF from Drive → Upload Document to Autype**.

13. **Add the title-page extraction node**
    - Create an **Autype** node named **Extract Title Page**.
    - Configure:
      - Resource: `documentTools`
      - Operation: `pages`
      - File ID: `{{ $('Loop Over Documents').item.json.titlePagesFileId }}`
      - Pages: `{{ $('Loop Over Documents').item.json.titlePageNumber.toString() }}`
    - Connect **Upload Document to Autype → Extract Title Page**.

14. **Add the merge-pair collector**
    - Create a **Code** node named **Collect Merge Pair**.
    - Configure it to return:
      - `titlePageFileId` from the extraction result
      - `docFileId` from **Upload Document to Autype**
      - `fileName` from **Loop Over Documents**
    - Connect **Extract Title Page → Collect Merge Pair**.

15. **Create the loop-back connection**
    - Connect **Collect Merge Pair → Loop Over Documents**.
    - This is what advances the Split In Batches node to the next item.

16. **Create the completion branch**
    - From **Loop Over Documents**, add the completion/output path to a new **Code** node named **Build Final Merge List**.
    - Configure it to:
      - Read all incoming collected items
      - Append each pair in this order:
        1. `titlePageFileId`
        2. `docFileId`
      - Join them into a comma-separated string in a field called `fileIds`

17. **Add the merge node**
    - Create an **Autype** node named **Merge All PDFs**.
    - Configure:
      - Resource: `documentTools`
      - Merge operation
      - File IDs: `{{ $json.fileIds }}`
    - Connect **Build Final Merge List → Merge All PDFs**.

18. **Add the watermark node**
    - Create an **Autype** node named **Add Company Watermark**.
    - Configure:
      - Resource: `documentTools`
      - Operation: `watermark`
      - File ID: `{{ $('Merge All PDFs').item.json.outputFileId }}`
      - Text: `Your Company Name`
      - Enable `downloadOutput`
      - Watermark options:
        - Color: `#2563EB`
        - Opacity: `0.6`
        - Font size: `14`
        - Rotation: `45`
    - Connect **Merge All PDFs → Add Company Watermark**.

19. **Add the final Google Drive upload node**
    - Create a **Google Drive** node named **Save Final PDF to Drive**.
    - Configure it to upload/create a file using incoming binary.
    - Set:
      - Drive: `My Drive`
      - Folder ID: your destination folder ID, which in the original workflow is the same placeholder folder
      - File name: `merged-documents-{{ $now.format('yyyy-MM-dd') }}.pdf`
    - Attach Google Drive credentials.
    - Connect **Add Company Watermark → Save Final PDF to Drive**.

20. **Optional: add sticky notes**
    - Add a large general note describing the whole workflow and requirements.
    - Add one note above the title-page generation area.
    - Add one note above the loop area.
    - Add one note above the merge/watermark/save area.

21. **Validate binary handling**
    - The workflow setting uses `binaryMode: separate`.
    - Ensure each binary-producing node passes a valid binary property expected by the next node.
    - If your node versions differ, verify the binary property names in execution data.

22. **Test with a small folder first**
    - Use a folder containing 2–3 PDFs.
    - Confirm:
      - The title-page PDF is generated
      - Each extracted page corresponds to the right document
      - The final merge order is correct
      - The watermark appears as expected
      - The output uploads to the correct folder

23. **Recommended hardening before production**
    - Add an explicit PDF MIME type filter in the Google Drive list step
   - Add an IF node or Code validation to stop if no files are found
    - Add error handling for failed Drive download or Autype upload
    - Optionally sort files explicitly before generating title pages if deterministic ordering matters

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and is **not** itself shown as a callable sub-workflow. There are no Execute Workflow nodes or external workflow dependencies.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Autype account is required to render, upload, extract, merge, and watermark PDFs. API keys are created in the Autype application settings. | [https://app.autype.com](https://app.autype.com) |
| The Autype node used here is a community node and requires a self-hosted n8n deployment. | n8n Settings → Community Nodes |
| Google Drive OAuth2 credentials must have access to the source and destination folder. | n8n credentials configuration |
| The workflow description claims it searches for PDFs, but the shown Google Drive node configuration does not explicitly filter by MIME type. Add a MIME type filter if strict PDF-only behavior is required. | Implementation note |
| The final file is named using the current date: `merged-documents-YYYY-MM-DD.pdf`. | Save Final PDF to Drive naming pattern |
| The title pages include file metadata: name, created date, modified date, owner, and document index. | Build Title Pages JSON logic |
| The watermark text is currently hard-coded as `Your Company Name` and should be replaced for real use. | Add Company Watermark node |