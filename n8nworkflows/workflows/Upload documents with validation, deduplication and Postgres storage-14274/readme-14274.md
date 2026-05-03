Upload documents with validation, deduplication and Postgres storage

https://n8nworkflows.xyz/workflows/upload-documents-with-validation--deduplication-and-postgres-storage-14274


# Upload documents with validation, deduplication and Postgres storage

# 1. Workflow Overview

This workflow receives a single uploaded tax-related document through either an n8n form or an HTTP webhook, validates the file, computes metadata and a SHA-256 content hash, checks PostgreSQL for duplicates, and either returns a duplicate notice or stores a new document record and returns success.

Typical use cases include:
- intake of invoices, receipts, W-2s, 1099s, and related tax files
- pre-processing before OCR, classification, or downstream accounting workflows
- controlled upload endpoints that reject oversized or unsupported file types
- duplicate prevention based on file content rather than filename

## 1.1 Input Reception
Two entry points are provided:
- a **Form Trigger** for user-friendly browser uploads
- a **Webhook** for programmatic POST uploads

Both paths converge immediately into shared processing.

## 1.2 Workflow Configuration and Validation
The workflow sets central configuration values such as maximum file size, allowed MIME types, and a storage base path placeholder. It then validates:
- file size
- MIME type

If either validation fails, the workflow returns a structured error response.

## 1.3 Metadata Generation and Duplicate Detection
For valid files, the workflow:
- extracts binary file details
- computes a SHA-256 hash from file contents
- generates a UUID document ID
- assembles metadata
- queries PostgreSQL to determine whether the file already exists

## 1.4 Duplicate Handling
If a matching file hash is found in the database, the workflow returns a duplicate response with the existing document ID instead of creating a new record.

## 1.5 Record Storage and Success Response
If the file is new, the workflow inserts a document row into PostgreSQL with status `received` and returns a success payload containing the newly generated document ID.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block exposes two independent upload entry points: one interactive form and one API webhook. Both feed into the same downstream validation and metadata pipeline.

### Nodes Involved
- Document Upload Form
- Webhook Upload

### Node Details

#### 1. Document Upload Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Browser-based form entry point that receives a single uploaded file.
- **Configuration choices:**
  - Form title: `Tax Document Upload`
  - Description: upload tax-related documents for automated processing
  - One required file field labeled `Select Document`
  - Accepts only `.pdf, .png, .jpg, .jpeg`
  - `multipleFiles: false`
  - Attribution disabled
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input; this is an entry node
  - Outputs to **Workflow Configuration**
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - User may submit unsupported file extension despite UI restrictions if client behavior is bypassed
  - Missing binary field naming assumptions can break downstream nodes if form payload structure differs from expected `data`
  - Large file uploads may fail before reaching logic depending on n8n/server limits
- **Sub-workflow reference:** None

#### 2. Webhook Upload
- **Type and technical role:** `n8n-nodes-base.webhook`  
  API endpoint for programmatic file upload via HTTP POST.
- **Configuration choices:**
  - Path: `document-upload`
  - Method: `POST`
  - Response mode: `lastNode`
  - `rawBody: true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input; this is an entry node
  - Outputs to **Workflow Configuration**
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Raw-body mode may require careful client request formatting to ensure binary data lands in `$binary.data`
  - If incoming webhook data is not mapped to binary property `data`, downstream validation and hashing will fail
  - Content-Type mismatches or multipart parsing issues can prevent file extraction
  - Because response mode is `lastNode`, execution path must always end in a node that emits output
- **Sub-workflow reference:** None

---

## Block 2 — Workflow Configuration and Validation

### Overview
This block centralizes file validation settings and applies size and MIME type checks. It prevents unsupported or oversized files from moving further into processing.

### Nodes Involved
- Workflow Configuration
- Check File Size
- Check MIME Type
- Return Validation Error

### Node Details

#### 3. Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Injects reusable workflow-level configuration values into the item JSON.
- **Configuration choices:**
  - `maxFileSizeMB = 50`
  - `allowedMimeTypes = "application/pdf,image/png,image/jpeg"`
  - `storageBasePath = "/tmp/tax-documents"`
  - Keeps other incoming fields via `includeOtherFields: true`
- **Key expressions or variables used:** None internally; values are referenced by downstream expressions
- **Input and output connections:**
  - Inputs from **Document Upload Form** and **Webhook Upload**
  - Outputs to **Check File Size**
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - `allowedMimeTypes` is stored as a comma-separated string, not an array; this works only because the subsequent IF node uses `contains` loosely against a string
  - `storageBasePath` is not used anywhere else in this workflow
  - If future branches overwrite these JSON fields, validation may behave unexpectedly
- **Sub-workflow reference:** None

#### 4. Check File Size
- **Type and technical role:** `n8n-nodes-base.if`  
  Verifies that uploaded binary size is less than or equal to configured limit.
- **Configuration choices:**
  - Compares `$binary.data.fileSize` to `maxFileSizeMB * 1024 * 1024`
  - Numeric `lte` condition
- **Key expressions or variables used:**
  - `={{ $binary.data.fileSize }}`
  - `={{ $('Workflow Configuration').item.json.maxFileSizeMB * 1024 * 1024 }}`
- **Input and output connections:**
  - Input from **Workflow Configuration**
  - True output to **Check MIME Type**
  - False output to **Return Validation Error**
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - If `$binary.data.fileSize` is missing or not numeric, the condition may evaluate incorrectly or fail
  - Depending on node source, file size may be string-typed; loose validation helps but is not perfect
  - If actual server upload limits are below 50 MB, uploads may fail before this node is reached
- **Sub-workflow reference:** None

#### 5. Check MIME Type
- **Type and technical role:** `n8n-nodes-base.if`  
  Verifies that the uploaded file MIME type is allowed.
- **Configuration choices:**
  - Uses array-style `contains` operation
  - Left value is the configured `allowedMimeTypes`
  - Right value is `$binary.data.mimeType`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').item.json.allowedMimeTypes }}`
  - `={{ $binary.data.mimeType }}`
- **Input and output connections:**
  - Input from **Check File Size**
  - True output to **Generate Document Metadata**
  - False output to **Return Validation Error**
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - The configured allowed MIME types are stored as a string rather than a real array
  - This may still match because of substring containment, but it is less strict than an actual array comparison
  - MIME types such as `image/jpg` would fail because only `image/jpeg` is allowed
  - Clients may upload a file with a misleading extension but accurate MIME type, or vice versa
- **Sub-workflow reference:** None

#### 6. Return Validation Error
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a structured error response for invalid file uploads.
- **Configuration choices:**
  - `status = "error"`
  - `message = "File validation failed: invalid size or file type"`
  - `allowed_types = "PDF, PNG, JPEG"`
  - `max_size_mb` pulled from configuration
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').item.json.maxFileSizeMB }}`
- **Input and output connections:**
  - Inputs from false branch of **Check File Size** and false branch of **Check MIME Type**
  - No downstream connection; acts as terminal output for those branches
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - For the webhook path, this returns only payload data; it does not explicitly set HTTP status code
  - `max_size_mb` is defined as type `string` even though source value is numeric
- **Sub-workflow reference:** None

---

## Block 3 — Metadata Generation and Duplicate Detection

### Overview
This block transforms the uploaded binary into a metadata record, computes a content hash for deduplication, and queries PostgreSQL to see whether the file has already been received.

### Nodes Involved
- Generate Document Metadata
- Check for Duplicate
- Is Duplicate?

### Node Details

#### 7. Generate Document Metadata
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript node that extracts binary details, computes SHA-256 hash, creates a document UUID, and returns structured metadata while preserving the binary.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Uses Node.js `crypto`
  - Reads uploaded file from `$binary.data`
  - Decodes base64 file contents into a Buffer
  - Creates:
    - `document_id`
    - `file_name`
    - `file_size`
    - `mime_type`
    - `file_hash`
    - `uploaded_at`
    - `source`
- **Key expressions or variables used:**
  - `$binary.data`
  - `crypto.createHash('sha256').update(buffer).digest('hex')`
  - `crypto.randomUUID()`
  - `new Date().toISOString()`
  - `const source = $node.name.includes('Form') ? 'form' : 'webhook';`
- **Input and output connections:**
  - Input from **Check MIME Type**
  - Output to **Check for Duplicate**
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - The `source` logic is flawed: `$node.name` in this code node is the current node name (`Generate Document Metadata`), so it will not contain `Form`; it will always resolve to `webhook`
  - If `$binary.data.data` is absent or malformed base64, Buffer conversion and hash generation will fail
  - `require('crypto')` assumes standard code-node runtime support
  - Memory usage increases with large files because the whole binary is converted into a buffer
- **Sub-workflow reference:** None

#### 8. Check for Duplicate
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Executes a SELECT query to find an existing document by exact file hash.
- **Configuration choices:**
  - Operation: `executeQuery`
  - SQL: `SELECT document_id, status FROM documents WHERE file_hash = $1 LIMIT 1`
  - Query replacement uses current item `file_hash`
- **Key expressions or variables used:**
  - `={{ $json.file_hash }}`
- **Input and output connections:**
  - Input from **Generate Document Metadata**
  - Output to **Is Duplicate?**
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**
  - Requires valid PostgreSQL credentials and reachable database
  - Fails if table `documents` does not exist or schema differs
  - Query-replacement formatting must align with n8n Postgres node expectations
  - If multiple rows exist for the same hash, `LIMIT 1` hides data quality issues
- **Sub-workflow reference:** None

#### 9. Is Duplicate?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the previous PostgreSQL query returned any rows.
- **Configuration choices:**
  - Checks whether `$('Check for Duplicate').item.json.length > 0`
- **Key expressions or variables used:**
  - `={{ $('Check for Duplicate').item.json.length }}`
- **Input and output connections:**
  - Input from **Check for Duplicate**
  - True output to **Return Duplicate Response**
  - False output to **Insert Document Record**
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - This logic assumes the Postgres node exposes results as an array under `.json.length`
  - In many n8n configurations, query results are emitted as individual items, not as an array attached to one item; if so, this condition will be incorrect
  - Duplicate detection behavior should be tested carefully against actual Postgres node output format in your n8n version
- **Sub-workflow reference:** None

---

## Block 4 — Duplicate Handling

### Overview
If a prior document with the same content hash is found, this block returns a duplicate response rather than inserting a new record.

### Nodes Involved
- Return Duplicate Response

### Node Details

#### 10. Return Duplicate Response
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates the payload for duplicate uploads.
- **Configuration choices:**
  - `status = "duplicate"`
  - `message = "Document already exists (duplicate detected via file hash)"`
  - `existing_document_id` taken from duplicate query output
- **Key expressions or variables used:**
  - `={{ $('Check for Duplicate').item.json[0].document_id }}`
- **Input and output connections:**
  - Input from true branch of **Is Duplicate?**
  - No downstream connection; terminal branch
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Assumes duplicate query output is an array and that first element exists at `.json[0]`
  - If Postgres returns itemized rows instead, this expression will fail
  - Does not emit an HTTP 409 status; only returns JSON-like data
- **Sub-workflow reference:** None

---

## Block 5 — Record Storage and Success Response

### Overview
If the document is not a duplicate, this block inserts a new database row and returns a success payload with the generated document ID.

### Nodes Involved
- Insert Document Record
- Return Success Response

### Node Details

#### 11. Insert Document Record
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts document metadata into the `documents` table.
- **Configuration choices:**
  - Operation: `executeQuery`
  - SQL inserts:
    - `document_id`
    - `file_hash`
    - `file_name`
    - `file_size`
    - `mime_type`
    - `source`
    - fixed status `received`
    - `created_at = NOW()`
  - Returns inserted row with `RETURNING *`
- **Key expressions or variables used:**
  - `={{ $('Generate Document Metadata').item.json.document_id }}`
  - `={{ $('Generate Document Metadata').item.json.file_hash }}`
  - `={{ $('Generate Document Metadata').item.json.file_name }}`
  - `={{ $('Generate Document Metadata').item.json.file_size }}`
  - `={{ $('Generate Document Metadata').item.json.mime_type }}`
  - `={{ $('Generate Document Metadata').item.json.source }}`
- **Input and output connections:**
  - Input from false branch of **Is Duplicate?**
  - Output to **Return Success Response**
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**
  - Requires existing `documents` table with compatible schema
  - The query replacement string must map correctly to six parameters; formatting mistakes can cause SQL binding errors
  - If a uniqueness constraint exists on `file_hash` and a race condition occurs, insert may fail even after duplicate check
  - Since `uploaded_at` is generated earlier but not stored, audit consistency may differ from returned metadata
- **Sub-workflow reference:** None

#### 12. Return Success Response
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates the final success payload after insertion.
- **Configuration choices:**
  - `status = "success"`
  - `message = "Document uploaded successfully"`
  - `document_id` from generated metadata
- **Key expressions or variables used:**
  - `={{ $('Generate Document Metadata').item.json.document_id }}`
- **Input and output connections:**
  - Input from **Insert Document Record**
  - No downstream connection; terminal branch
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Returns generated ID, not necessarily the inserted row’s ID if upstream expressions become inconsistent
  - Does not explicitly include inserted database status or timestamps
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Document Upload Form | Form Trigger | Browser form upload entry point |  | Workflow Configuration | ## Upload Input<br>Accepts document via form or webhook upload. |
| Webhook Upload | Webhook | API upload entry point |  | Workflow Configuration | ## Upload Input<br>Accepts document via form or webhook upload. |
| Workflow Configuration | Set | Defines validation and storage configuration | Document Upload Form, Webhook Upload | Check File Size | ## Configuration and Validation<br>Defines file size limits, allowed types, and storage settings and Checks file size and MIME type before processing. |
| Check File Size | If | Validates maximum file size | Workflow Configuration | Check MIME Type, Return Validation Error | ## Configuration and Validation<br>Defines file size limits, allowed types, and storage settings and Checks file size and MIME type before processing. |
| Check MIME Type | If | Validates allowed MIME type | Check File Size | Generate Document Metadata, Return Validation Error | ## Configuration and Validation<br>Defines file size limits, allowed types, and storage settings and Checks file size and MIME type before processing. |
| Generate Document Metadata | Code | Computes hash, UUID, and metadata from binary file | Check MIME Type | Check for Duplicate | ## Metadata Generation and Duplicate check<br>Creates document ID, hash, and extracts file details and Checks if file already exists using hash comparison. |
| Check for Duplicate | Postgres | Queries existing document by file hash | Generate Document Metadata | Is Duplicate? | ## Metadata Generation and Duplicate check<br>Creates document ID, hash, and extracts file details and Checks if file already exists using hash comparison. |
| Is Duplicate? | If | Branches between duplicate response and insert path | Check for Duplicate | Return Duplicate Response, Insert Document Record |  |
| Return Duplicate Response | Set | Returns duplicate result payload | Is Duplicate? |  | ## Duplicate Handling<br>Returns response if duplicate document is detected. |
| Insert Document Record | Postgres | Stores new document metadata in PostgreSQL | Is Duplicate? | Return Success Response | ## Success Response and Record Storage<br>Stores document metadata in Postgres database and<br>Returns success message with document ID. |
| Return Success Response | Set | Returns success payload with document ID | Insert Document Record |  | ## Success Response and Record Storage<br>Stores document metadata in Postgres database and<br>Returns success message with document ID. |
| Return Validation Error | Set | Returns validation failure payload | Check File Size, Check MIME Type |  | ## Error Handling<br>Returns error if file validation fails. |
| Sticky Note1 | Sticky Note | Documentation annotation |  |  |  |
| Sticky Note | Sticky Note | Documentation annotation |  |  |  |
| Sticky Note2 | Sticky Note | Documentation annotation |  |  |  |
| Sticky Note3 | Sticky Note | Documentation annotation |  |  |  |
| Sticky Note4 | Sticky Note | Documentation annotation |  |  |  |
| Sticky Note5 | Sticky Note | Documentation annotation |  |  |  |
| Sticky Note6 | Sticky Note | Documentation annotation |  |  |  |
| Sticky Note7 | Sticky Note | Documentation annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Upload documents with validation, deduplication and Postgres storage`.

2. **Add a Form Trigger node** named `Document Upload Form`.
   - Node type: **Form Trigger**
   - Set form title to `Tax Document Upload`
   - Set description to:  
     `Upload tax-related documents (invoices, 1099, W-2, receipts) for automated processing`
   - Add one form field:
     - Type: `File`
     - Label: `Select Document`
     - Required: enabled
     - Multiple files: disabled
     - Accepted file types: `.pdf, .png, .jpg, .jpeg`
   - In options, disable attribution if desired.

3. **Add a Webhook node** named `Webhook Upload`.
   - Node type: **Webhook**
   - HTTP Method: `POST`
   - Path: `document-upload`
   - Response mode: `Last Node`
   - Enable raw body in options
   - Note: for reliable binary upload handling, confirm how your n8n instance maps webhook uploads into binary properties. The rest of the workflow expects the uploaded file in `$binary.data`.

4. **Add a Set node** named `Workflow Configuration`.
   - Node type: **Set**
   - Keep incoming fields
   - Add these fields:
     - `maxFileSizeMB` as Number = `50`
     - `allowedMimeTypes` as String = `application/pdf,image/png,image/jpeg`
     - `storageBasePath` as String = `/tmp/tax-documents`
   - Connect:
     - `Document Upload Form` → `Workflow Configuration`
     - `Webhook Upload` → `Workflow Configuration`

5. **Add an IF node** named `Check File Size`.
   - Node type: **If**
   - Configure one numeric condition:
     - Left value: `{{ $binary.data.fileSize }}`
     - Operation: `less than or equal`
     - Right value: `{{ $('Workflow Configuration').item.json.maxFileSizeMB * 1024 * 1024 }}`
   - Connect:
     - `Workflow Configuration` → `Check File Size`

6. **Add another IF node** named `Check MIME Type`.
   - Node type: **If**
   - Configure one condition checking allowed MIME types:
     - Left value: `{{ $('Workflow Configuration').item.json.allowedMimeTypes }}`
     - Operation: `contains`
     - Right value: `{{ $binary.data.mimeType }}`
   - Connect:
     - True branch of `Check File Size` → `Check MIME Type`

7. **Add a Set node** named `Return Validation Error`.
   - Node type: **Set**
   - Add these fields:
     - `status` = `error`
     - `message` = `File validation failed: invalid size or file type`
     - `allowed_types` = `PDF, PNG, JPEG`
     - `max_size_mb` = `{{ $('Workflow Configuration').item.json.maxFileSizeMB }}`
   - Connect:
     - False branch of `Check File Size` → `Return Validation Error`
     - False branch of `Check MIME Type` → `Return Validation Error`

8. **Add a Code node** named `Generate Document Metadata`.
   - Node type: **Code**
   - Mode: `Run Once for Each Item`
   - Paste JavaScript that:
     - reads `$binary.data`
     - converts base64 content into a `Buffer`
     - computes SHA-256 hash using `crypto`
     - generates UUID with `crypto.randomUUID()`
     - extracts `file_name`, `file_size`, `mime_type`
     - sets `uploaded_at`
     - returns JSON metadata plus preserved binary
   - Use the same logic as the workflow, but preferably improve source detection. A safer approach is to set a source flag before the merge point or inspect `$execution`/trigger context explicitly.
   - Connect:
     - True branch of `Check MIME Type` → `Generate Document Metadata`

9. **Add PostgreSQL credentials** in n8n before using Postgres nodes.
   - Create or select a **Postgres credential**
   - Provide:
     - host
     - port
     - database
     - user
     - password
     - SSL options if required

10. **Prepare the PostgreSQL table** externally.
    A compatible table should exist, for example with columns:
    - `document_id` text or uuid primary key
    - `file_hash` text
    - `file_name` text
    - `file_size` bigint or integer
    - `mime_type` text
    - `source` text
    - `status` text
    - `created_at` timestamp
    Recommended additions:
    - unique index on `file_hash`
    - optional `updated_at`
    - optional `uploaded_at`

11. **Add a Postgres node** named `Check for Duplicate`.
    - Node type: **Postgres**
    - Operation: `Execute Query`
    - SQL:
      `SELECT document_id, status FROM documents WHERE file_hash = $1 LIMIT 1`
    - Use query replacement with:
      `{{ $json.file_hash }}`
    - Attach the Postgres credential
    - Connect:
      - `Generate Document Metadata` → `Check for Duplicate`

12. **Add an IF node** named `Is Duplicate?`.
    - Node type: **If**
    - Recreate the original logic if you want fidelity:
      - Left value: `{{ $('Check for Duplicate').item.json.length }}`
      - Operation: `greater than`
      - Right value: `0`
    - Connect:
      - `Check for Duplicate` → `Is Duplicate?`
    - Important: test this carefully. In many n8n/Postgres setups, a better duplicate test may be needed depending on query output shape.

13. **Add a Set node** named `Return Duplicate Response`.
    - Node type: **Set**
    - Add fields:
      - `status` = `duplicate`
      - `message` = `Document already exists (duplicate detected via file hash)`
      - `existing_document_id` = `{{ $('Check for Duplicate').item.json[0].document_id }}`
    - Connect:
      - True branch of `Is Duplicate?` → `Return Duplicate Response`
    - Important: if your Postgres node returns rows as normal items rather than an array, adjust this expression accordingly.

14. **Add a Postgres node** named `Insert Document Record`.
    - Node type: **Postgres**
    - Operation: `Execute Query`
    - SQL:
      ```sql
      INSERT INTO documents
      (document_id, file_hash, file_name, file_size, mime_type, source, status, created_at)
      VALUES ($1, $2, $3, $4, $5, $6, 'received', NOW())
      RETURNING *
      ```
    - Configure query replacements using:
      - `{{ $('Generate Document Metadata').item.json.document_id }}`
      - `{{ $('Generate Document Metadata').item.json.file_hash }}`
      - `{{ $('Generate Document Metadata').item.json.file_name }}`
      - `{{ $('Generate Document Metadata').item.json.file_size }}`
      - `{{ $('Generate Document Metadata').item.json.mime_type }}`
      - `{{ $('Generate Document Metadata').item.json.source }}`
    - Attach same Postgres credential
    - Connect:
      - False branch of `Is Duplicate?` → `Insert Document Record`

15. **Add a Set node** named `Return Success Response`.
    - Node type: **Set**
    - Add fields:
      - `status` = `success`
      - `message` = `Document uploaded successfully`
      - `document_id` = `{{ $('Generate Document Metadata').item.json.document_id }}`
    - Connect:
      - `Insert Document Record` → `Return Success Response`

16. **Add sticky notes** if you want the same visual documentation.
   Suggested note contents:
   - `## Upload Input`  
     `Accepts document via form or webhook upload.`
   - `## Configuration and Validation`  
     `Defines file size limits, allowed types, and storage settings and Checks file size and MIME type before processing.`
   - `## Error Handling`  
     `Returns error if file validation fails.`
   - `## Metadata Generation and Duplicate check`  
     `Creates document ID, hash, and extracts file details and Checks if file already exists using hash comparison.`
   - `## Duplicate Handling`  
     `Returns response if duplicate document is detected.`
   - `## Success Response and Record Storage`  
     `Stores document metadata in Postgres database and Returns success message with document ID.`
   - `## How it works` plus setup notes if desired

17. **Test the form path.**
   - Upload a supported file under 50 MB
   - Confirm:
     - validation passes
     - metadata is generated
     - Postgres row is inserted
     - success payload is returned

18. **Test the duplicate path.**
   - Upload the exact same file again
   - Confirm:
     - hash matches existing record
     - duplicate branch is taken
     - existing document ID is returned

19. **Test validation failures.**
   - Upload unsupported MIME type or oversized file
   - Confirm:
     - error branch is reached
     - structured error payload is returned

20. **Activate the workflow** after confirming both trigger endpoints behave correctly.

## Recommended implementation improvements during rebuild
If you are rebuilding this for production, consider these changes:
1. Store `allowedMimeTypes` as an actual array, not a comma-separated string.
2. Fix source detection by setting source earlier:
   - add one Set node after the form trigger with `source=form`
   - add one Set node after the webhook with `source=webhook`
   - merge into shared path
3. Verify actual Postgres output format and adjust duplicate-check expressions.
4. Add explicit HTTP response codes for webhook consumers:
   - `200` success
   - `409` duplicate
   - `400` validation error
5. Add a unique DB constraint on `file_hash` to prevent race-condition duplicates.
6. If file storage is intended, add a binary write/storage step because `storageBasePath` is currently unused.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow handles secure document uploads from a form or webhook. Uploaded files are first validated for size and type to ensure they meet defined limits. Once validated, the workflow generates metadata including a unique document ID, file hash, and file details. The file hash is used to check for duplicates in the database. If a duplicate is found, the workflow returns an existing record instead of storing it again. If the file is new, its metadata is stored in Postgres with a “received” status. A success response is returned with the document ID for tracking. | Workflow-level note |
| Setup notes: 1. Configure form or webhook for uploads 2. Set max file size and allowed types 3. Connect Postgres database 4. Create documents table 5. Test with sample uploads 6. Activate the workflow | Workflow-level note |

## Additional implementation notes
- `storageBasePath` is defined but unused; no physical file persistence occurs in this workflow.
- Duplicate detection is based on content hash, not filename, so renamed identical files are still treated as duplicates.
- The current code node’s source detection likely always resolves incorrectly; test and correct if source attribution matters.
- The workflow contains two entry points and no sub-workflows.