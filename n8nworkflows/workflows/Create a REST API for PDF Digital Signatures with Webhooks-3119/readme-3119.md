Create a REST API for PDF Digital Signatures with Webhooks

https://n8nworkflows.xyz/workflows/create-a-rest-api-for-pdf-digital-signatures-with-webhooks-3119


# Create a REST API for PDF Digital Signatures with Webhooks

### 1. Workflow Overview

This workflow implements a complete REST API for digitally signing PDF documents using n8n webhooks. It is designed to provide secure, automated PDF signing capabilities accessible via standardized HTTP endpoints. The workflow supports uploading PDF files and digital certificates, generating new certificates, signing PDFs with certificates, and downloading signed documents.

**Target Use Cases:**
- Developers integrating PDF digital signature functionality into document workflows.
- Automation specialists creating API-driven document signing processes.
- Proof-of-concept and testing environments for digital signature systems.
- Learning and experimenting with n8n webhook and file handling features.

**Logical Blocks:**

- **1.1 Request Processing and Method Routing:** Receives incoming HTTP requests via webhooks, parses method parameters, and routes requests to appropriate handlers.
- **1.2 Document Upload Handling:** Validates and processes PDF document uploads.
- **1.3 Certificate Upload Handling:** Validates and processes digital certificate (PFX) uploads.
- **1.4 Certificate Generation:** Validates parameters and generates new digital certificates and keys.
- **1.5 PDF Signing:** Validates parameters and applies digital signatures to PDF documents.
- **1.6 Document Download Handling:** Processes requests to download signed or uploaded documents.
- **1.7 Response Checking and Formatting:** Validates operation success and formats API responses accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Request Processing and Method Routing

**Overview:**  
This block receives all incoming API requests via webhooks, sets file path variables, and routes requests based on the `method` parameter in the request body. It directs the workflow to upload, generate keys, sign PDFs, or download documents.

**Nodes Involved:**  
- API POST Endpoint  
- API GET Endpoint  
- set file path  
- Switch Operation  
- Prepare input params  

**Node Details:**

- **API POST Endpoint**  
  - Type: Webhook (POST)  
  - Role: Entry point for document and certificate operations (upload, generate keys, sign PDF)  
  - Config: Path `/docu-digi-sign`, accepts POST requests, response mode set to response node  
  - Inputs: External HTTP POST requests  
  - Outputs: Connected to `set file path` node  
  - Edge Cases: Invalid HTTP method, missing parameters, malformed JSON  

- **API GET Endpoint**  
  - Type: Webhook (GET)  
  - Role: Entry point for document download requests  
  - Config: Path `/docu-download`, accepts GET requests, response mode set to response node  
  - Inputs: External HTTP GET requests  
  - Outputs: Connected to `set file path` node  
  - Edge Cases: Missing or invalid query parameters  

- **set file path**  
  - Type: Set  
  - Role: Defines default storage paths for PDFs and keys (`/data/files/`)  
  - Config: Sets `pdfPath` and `keyPath` variables for downstream nodes  
  - Inputs: From both webhook nodes  
  - Outputs: Connected to `Switch Operation` node  
  - Edge Cases: Path permission issues, environment-specific path adjustments  

- **Switch Operation**  
  - Type: Switch  
  - Role: Routes requests based on `method` field in the request body (`upload`, `genKey`, `signPdf`, `download`)  
  - Config: Uses strict string equality on `$json.body.method`  
  - Inputs: From `set file path`  
  - Outputs:  
    - `upload` → `Prepare input params`  
    - `genKey` → `Validate Key Gen Params`  
    - `signPdf` → `Validate PDF Sign Params`  
    - `download` → `set downlowd file info`  
  - Edge Cases: Unknown method values, missing method parameter  

- **Prepare input params**  
  - Type: Set  
  - Role: Prepares binary file data and generates a unique filename for uploads  
  - Config: Extracts binary data from webhook input, creates unique filename based on timestamp and MIME type  
  - Inputs: From `Switch Operation` (upload path)  
  - Outputs: Connected to `Switch Upload Type`  
  - Edge Cases: Missing binary data, MIME type parsing errors  

---

#### 2.2 Document Upload Handling

**Overview:**  
Handles validation and processing of uploaded PDF documents, converting binary data to files and writing them to disk.

**Nodes Involved:**  
- Switch Upload Type  
- Validate PDF Upload  
- Convert PDF to File  
- Write PDF File to Disk  
- Read PDF File from Disk  
- Check PDF file is OK  

**Node Details:**

- **Switch Upload Type**  
  - Type: Switch  
  - Role: Routes upload requests based on `uploadType` (`pdfDoc` or `signKey`)  
  - Inputs: From `Prepare input params`  
  - Outputs:  
    - `pdfDoc` → `Validate PDF Upload`  
    - `signKey` → `Validate Key Upload` (see next block)  
  - Edge Cases: Unknown uploadType, missing uploadType parameter  

- **Validate PDF Upload**  
  - Type: Code  
  - Role: Validates presence of `fileData` for PDF uploads, sets output path with timestamp fallback  
  - Inputs: From `Switch Upload Type` (pdfDoc)  
  - Outputs: Connected to `Convert PDF to File`  
  - Edge Cases: Missing fileData, invalid fileName  

- **Convert PDF to File**  
  - Type: ConvertToFile  
  - Role: Converts base64 binary data to file format for writing  
  - Config: Uses filename from request body, MIME type from binary data  
  - Inputs: From `Validate PDF Upload`  
  - Outputs: Connected to `Write PDF File to Disk`  
  - Edge Cases: Binary data corruption, unsupported MIME types  

- **Write PDF File to Disk**  
  - Type: ReadWriteFile  
  - Role: Writes the converted PDF file to disk at the specified path  
  - Config: Writes to path constructed from `pdfPath` and unique filename  
  - Inputs: From `Convert PDF to File`  
  - Outputs: Connected to `Read PDF File from Disk`  
  - Edge Cases: Disk write permission errors, path not found  

- **Read PDF File from Disk**  
  - Type: ReadWriteFile  
  - Role: Reads back the written PDF file to confirm write success  
  - Inputs: From `Write PDF File to Disk`  
  - Outputs: Connected to `Check PDF file is OK`  
  - Edge Cases: File read errors, file missing after write  

- **Check PDF file is OK**  
  - Type: Set  
  - Role: Confirms that the file read matches the expected unique filename, sets success flag  
  - Inputs: From `Read PDF File from Disk`  
  - Outputs: Connected to `check success` (response block)  
  - Edge Cases: Filename mismatch, file integrity issues  

---

#### 2.3 Certificate Upload Handling

**Overview:**  
Validates and processes uploaded digital certificates (PFX files), converting and writing them to disk.

**Nodes Involved:**  
- Validate Key Upload  
- Convert PFX to File  
- Write PFX File to Disk  
- Read PFX File from Disk  
- Check PFX file is OK  

**Node Details:**

- **Validate Key Upload**  
  - Type: Code  
  - Role: Validates presence of `fileData` for key uploads, sets output path with timestamp fallback  
  - Inputs: From `Switch Upload Type` (signKey)  
  - Outputs: Connected to `Convert PFX to File`  
  - Edge Cases: Missing fileData, invalid fileName  

- **Convert PFX to File**  
  - Type: ConvertToFile  
  - Role: Converts base64 binary PFX data to file format for writing  
  - Config: Uses filename from request body, MIME type from binary data  
  - Inputs: From `Validate Key Upload`  
  - Outputs: Connected to `Write PFX File to Disk`  
  - Edge Cases: Binary data corruption, unsupported MIME types  

- **Write PFX File to Disk**  
  - Type: ReadWriteFile  
  - Role: Writes the converted PFX file to disk at the specified path  
  - Config: Writes to path constructed from `pdfPath` and unique filename (note: uses pdfPath, which may be a naming inconsistency)  
  - Inputs: From `Convert PFX to File`  
  - Outputs: Connected to `Read PFX File from Disk`  
  - Edge Cases: Disk write permission errors, path not found  

- **Read PFX File from Disk**  
  - Type: ReadWriteFile  
  - Role: Reads back the written PFX file to confirm write success  
  - Inputs: From `Write PFX File to Disk`  
  - Outputs: Connected to `Check PFX file is OK`  
  - Edge Cases: File read errors, file missing after write  

- **Check PFX file is OK**  
  - Type: Set  
  - Role: Confirms that the file read matches the expected unique filename, sets success flag  
  - Inputs: From `Read PFX File from Disk`  
  - Outputs: Connected to `check success` (response block)  
  - Edge Cases: Filename mismatch, file integrity issues  

---

#### 2.4 Certificate Generation

**Overview:**  
Validates input parameters for certificate generation and creates a new digital certificate and associated keys using node-forge. Saves generated files to disk.

**Nodes Involved:**  
- Validate Key Gen Params  
- Generate Keys  

**Node Details:**

- **Validate Key Gen Params**  
  - Type: Code  
  - Role: Checks for required parameters (`subjectCN`, `issuerCN`, `serialNumber`, `validFrom`, `validTo`, `password`) in the request body  
  - Sets default output directory `/tmp` if not provided  
  - Outputs paths for generated files with timestamp-based names  
  - Inputs: From `Switch Operation` (genKey path)  
  - Outputs: Connected to `Generate Keys`  
  - Edge Cases: Missing parameters, invalid date formats  

- **Generate Keys**  
  - Type: Code  
  - Role: Generates RSA key pair and self-signed certificate using node-forge  
  - Creates PEM files and PKCS#12 (PFX) file with password protection  
  - Saves files to disk at specified paths  
  - Returns success status and file paths/names  
  - Inputs: From `Validate Key Gen Params`  
  - Outputs: Connected to `check success` (response block)  
  - Edge Cases: Cryptographic errors, file write failures, invalid parameters  

---

#### 2.5 PDF Signing

**Overview:**  
Validates parameters for PDF signing and applies a digital signature to the specified PDF using the provided PFX certificate.

**Nodes Involved:**  
- Validate PDF Sign Params  
- Sign PDF  

**Node Details:**

- **Validate PDF Sign Params**  
  - Type: Code  
  - Role: Checks for required parameters (`inputPdf`, `pfxFile`, `pfxPassword`) in the request body  
  - Sets default directories `/tmp` if not provided  
  - Constructs full file paths for input PDF, PFX file, and output signed PDF with timestamp  
  - Inputs: From `Switch Operation` (signPdf path)  
  - Outputs: Connected to `Sign PDF`  
  - Edge Cases: Missing parameters, invalid file paths  

- **Sign PDF**  
  - Type: Code  
  - Role: Reads input PDF and PFX files from disk, adds a signature placeholder, signs the PDF using @signpdf libraries  
  - Writes signed PDF to output path  
  - Returns success status and output file info  
  - Inputs: From `Validate PDF Sign Params`  
  - Outputs: Connected to `check success` (response block)  
  - Edge Cases: File read/write errors, invalid PFX password, signing errors  

---

#### 2.6 Document Download Handling

**Overview:**  
Handles requests to download signed or uploaded documents by reading files from disk and responding with binary data and appropriate headers.

**Nodes Involved:**  
- set downlowd file info  
- Read download file from Disk  
- GET Respond to Webhook  

**Node Details:**

- **set downlowd file info**  
  - Type: Set  
  - Role: Determines full file path for download based on file extension and configured paths  
  - Inputs: From `Switch Operation` (download path)  
  - Outputs: Connected to `Read download file from Disk`  
  - Edge Cases: Missing or invalid filename, path resolution errors  

- **Read download file from Disk**  
  - Type: ReadWriteFile  
  - Role: Reads the requested file from disk to prepare for download  
  - Inputs: From `set downlowd file info`  
  - Outputs: Connected to `GET Respond to Webhook`  
  - Edge Cases: File not found, read permission errors  

- **GET Respond to Webhook**  
  - Type: RespondToWebhook  
  - Role: Sends the file content as a binary response with appropriate Content-Disposition header for attachment download  
  - Inputs: From `Read download file from Disk`  
  - Outputs: None (end of flow)  
  - Edge Cases: Response streaming errors  

---

#### 2.7 Response Checking and Formatting

**Overview:**  
Checks the success status of operations and routes to success or error response nodes, formatting the final API response.

**Nodes Involved:**  
- check success  
- Prepare Success Response  
- POST Success Response  
- POST Error Response  

**Node Details:**

- **check success**  
  - Type: If  
  - Role: Evaluates the `success` boolean in the JSON payload  
  - Inputs: From `Check PDF file is OK`, `Check PFX file is OK`, `Generate Keys`, `Sign PDF`  
  - Outputs:  
    - True → `Prepare Success Response`  
    - False → `POST Error Response`  
  - Edge Cases: Missing success field, unexpected data types  

- **Prepare Success Response**  
  - Type: Set  
  - Role: Sets `success` to true and includes the server file name in the response  
  - Inputs: From `check success` (true branch)  
  - Outputs: Connected to `POST Success Response`  
  - Edge Cases: Missing fileName field  

- **POST Success Response**  
  - Type: RespondToWebhook  
  - Role: Sends a successful JSON response back to the client  
  - Inputs: From `Prepare Success Response`  
  - Outputs: None (end of flow)  

- **POST Error Response**  
  - Type: RespondToWebhook  
  - Role: Sends an error JSON response back to the client  
  - Inputs: From `check success` (false branch)  
  - Outputs: None (end of flow)  

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                          | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                  |
|-------------------------|----------------------|----------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| API POST Endpoint       | Webhook (POST)       | Entry point for document/cert operations | -                                | set file path                   | # Request Processing and Method Routing                                                     |
| API GET Endpoint        | Webhook (GET)        | Entry point for document download       | -                                | set file path                   | # Request Processing and Method Routing                                                     |
| set file path           | Set                  | Defines storage paths                   | API POST Endpoint, API GET Endpoint | Switch Operation               | # Request Processing and Method Routing                                                     |
| Switch Operation        | Switch               | Routes request by method                | set file path                    | Prepare input params, Validate Key Gen Params, Validate PDF Sign Params, set downlowd file info | # Request Processing and Method Routing                                                     |
| Prepare input params    | Set                  | Prepares binary data and unique filename | Switch Operation (upload)         | Switch Upload Type              | # Document Management - Upload Certificate and Upload PDF                                   |
| Switch Upload Type      | Switch               | Routes upload by type                   | Prepare input params             | Validate PDF Upload, Validate Key Upload | # Document Management - Upload Certificate and Upload PDF                                   |
| Validate PDF Upload     | Code                 | Validates PDF upload parameters         | Switch Upload Type (pdfDoc)       | Convert PDF to File             | # Document Management - Upload Certificate and Upload PDF                                   |
| Convert PDF to File     | ConvertToFile        | Converts PDF binary data to file        | Validate PDF Upload              | Write PDF File to Disk          | # Document Management - Upload Certificate and Upload PDF                                   |
| Write PDF File to Disk  | ReadWriteFile        | Writes PDF file to disk                  | Convert PDF to File              | Read PDF File from Disk         | # Document Management - Upload Certificate and Upload PDF                                   |
| Read PDF File from Disk | ReadWriteFile        | Reads PDF file from disk                 | Write PDF File to Disk           | Check PDF file is OK            | # Document Management - Upload Certificate and Upload PDF                                   |
| Check PDF file is OK    | Set                  | Validates PDF file write success         | Read PDF File from Disk          | check success                  | # Document Management - Upload Certificate and Upload PDF                                   |
| Validate Key Upload     | Code                 | Validates certificate upload parameters | Switch Upload Type (signKey)      | Convert PFX to File            | # Document Management - Upload Certificate and Upload PDF                                   |
| Convert PFX to File     | ConvertToFile        | Converts PFX binary data to file         | Validate Key Upload              | Write PFX File to Disk          | # Document Management - Upload Certificate and Upload PDF                                   |
| Write PFX File to Disk  | ReadWriteFile        | Writes PFX file to disk                   | Convert PFX to File              | Read PFX File from Disk         | # Document Management - Upload Certificate and Upload PDF                                   |
| Read PFX File from Disk | ReadWriteFile        | Reads PFX file from disk                  | Write PFX File to Disk           | Check PFX file is OK            | # Document Management - Upload Certificate and Upload PDF                                   |
| Check PFX file is OK    | Set                  | Validates PFX file write success          | Read PFX File from Disk          | check success                  | # Document Management - Upload Certificate and Upload PDF                                   |
| Validate Key Gen Params | Code                 | Validates certificate generation params  | Switch Operation (genKey)         | Generate Keys                  | # Cryptographic Operations - Generate Certificate and Sign PDF                              |
| Generate Keys           | Code                 | Generates certificate and keys            | Validate Key Gen Params          | check success                  | # Cryptographic Operations - Generate Certificate and Sign PDF                              |
| Validate PDF Sign Params| Code                 | Validates PDF signing parameters          | Switch Operation (signPdf)        | Sign PDF                      | # Cryptographic Operations - Generate Certificate and Sign PDF                              |
| Sign PDF                | Code                 | Applies digital signature to PDF          | Validate PDF Sign Params         | check success                  | # Cryptographic Operations - Generate Certificate and Sign PDF                              |
| check success           | If                   | Checks operation success and routes response | Check PDF file is OK, Check PFX file is OK, Generate Keys, Sign PDF | Prepare Success Response, POST Error Response | # Response Checking and Formatting                                                         |
| Prepare Success Response| Set                  | Prepares success response data            | check success (true)             | POST Success Response          | # Response Checking and Formatting                                                         |
| POST Success Response   | RespondToWebhook     | Sends success response to client          | Prepare Success Response         | -                             | # Response Checking and Formatting                                                         |
| POST Error Response     | RespondToWebhook     | Sends error response to client            | check success (false)            | -                             | # Response Checking and Formatting                                                         |
| set downlowd file info  | Set                  | Prepares file path for download            | Switch Operation (download)       | Read download file from Disk   | # Document Management - Download document                                                  |
| Read download file from Disk | ReadWriteFile   | Reads file from disk for download          | set downlowd file info           | GET Respond to Webhook         | # Document Management - Download document                                                  |
| GET Respond to Webhook  | RespondToWebhook     | Sends file as binary download response     | Read download file from Disk     | -                             | # Document Management - Download document                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes:**
   - Create a **Webhook (POST)** node named `API POST Endpoint`:
     - Path: `docu-digi-sign`
     - HTTP Method: POST
     - Response Mode: Response Node
     - Enable binary data handling as needed.
   - Create a **Webhook (GET)** node named `API GET Endpoint`:
     - Path: `docu-download`
     - HTTP Method: GET
     - Response Mode: Response Node

2. **Set Storage Paths:**
   - Add a **Set** node named `set file path` connected from both webhook nodes.
   - Set variables:
     - `pdfPath` = `/data/files/`
     - `keyPath` = `/data/files/`

3. **Method Routing:**
   - Add a **Switch** node named `Switch Operation` connected from `set file path`.
   - Configure rules on `$json.body.method`:
     - `upload` → output 1
     - `genKey` → output 2
     - `signPdf` → output 3
     - `download` → output 4

4. **Prepare Upload Input Parameters:**
   - Add a **Set** node named `Prepare input params` connected from `Switch Operation` output 1 (`upload`).
   - Configure to:
     - Strip binary data.
     - Assign `fileData` from binary input `$('API POST Endpoint').item.binary`.
     - Assign `uniqueFileName` as `'file_' + $now.toMillis() + '.' + mime subtype from binary data`.
   - Include all other fields.

5. **Upload Type Routing:**
   - Add a **Switch** node named `Switch Upload Type` connected from `Prepare input params`.
   - Configure rules on `$json.body.uploadType`:
     - `pdfDoc` → output 1
     - `signKey` → output 2

6. **PDF Upload Validation and Processing:**
   - Add a **Code** node named `Validate PDF Upload` connected from `Switch Upload Type` output 1.
     - Validate presence of `fileData`.
     - Set default output directory `/tmp` if not provided.
     - Construct output path using `fileName` or timestamped default.
   - Add a **ConvertToFile** node named `Convert PDF to File` connected from `Validate PDF Upload`.
     - Configure to convert base64 to binary file.
     - Use filename and MIME type from input.
   - Add a **ReadWriteFile** node named `Write PDF File to Disk` connected from `Convert PDF to File`.
     - Write operation.
     - Filename: `pdfPath + uniqueFileName`.
   - Add a **ReadWriteFile** node named `Read PDF File from Disk` connected from `Write PDF File to Disk`.
     - Read operation.
     - File selector: filename just written.
   - Add a **Set** node named `Check PDF file is OK` connected from `Read PDF File from Disk`.
     - Set `success` boolean if filename matches expected.
     - Pass filename.

7. **Certificate Upload Validation and Processing:**
   - Add a **Code** node named `Validate Key Upload` connected from `Switch Upload Type` output 2.
     - Validate presence of `fileData`.
     - Set default output directory `/tmp` if not provided.
     - Construct output path using `fileName` or timestamped default.
   - Add a **ConvertToFile** node named `Convert PFX to File` connected from `Validate Key Upload`.
     - Convert base64 to binary file.
     - Use filename and MIME type from input.
   - Add a **ReadWriteFile** node named `Write PFX File to Disk` connected from `Convert PFX to File`.
     - Write operation.
     - Filename: `pdfPath + uniqueFileName` (consider correcting to `keyPath`).
   - Add a **ReadWriteFile** node named `Read PFX File from Disk` connected from `Write PFX File to Disk`.
     - Read operation.
     - File selector: filename just written.
   - Add a **Set** node named `Check PFX file is OK` connected from `Read PFX File from Disk`.
     - Set `success` boolean if filename matches expected.
     - Pass filename.

8. **Certificate Generation:**
   - Add a **Code** node named `Validate Key Gen Params` connected from `Switch Operation` output 2 (`genKey`).
     - Validate required parameters: `subjectCN`, `issuerCN`, `serialNumber`, `validFrom`, `validTo`, `password`.
     - Set default output directory `/tmp`.
     - Generate output file paths with timestamp.
   - Add a **Code** node named `Generate Keys` connected from `Validate Key Gen Params`.
     - Use `node-forge` to generate RSA key pair and self-signed certificate.
     - Create PEM and PFX files.
     - Save files to disk.
     - Return success status and file info.

9. **PDF Signing:**
   - Add a **Code** node named `Validate PDF Sign Params` connected from `Switch Operation` output 3 (`signPdf`).
     - Validate required parameters: `inputPdf`, `pfxFile`, `pfxPassword`.
     - Set default directories `/tmp`.
     - Construct full file paths for input PDF, PFX, and output signed PDF.
   - Add a **Code** node named `Sign PDF` connected from `Validate PDF Sign Params`.
     - Read input PDF and PFX files.
     - Add signature placeholder.
     - Sign PDF using `@signpdf` libraries.
     - Write signed PDF to disk.
     - Return success status and file info.

10. **Document Download Handling:**
    - Add a **Set** node named `set downlowd file info` connected from `Switch Operation` output 4 (`download`).
      - Determine full file path based on file extension and configured paths.
    - Add a **ReadWriteFile** node named `Read download file from Disk` connected from `set downlowd file info`.
      - Read operation for the requested file.
    - Add a **RespondToWebhook** node named `GET Respond to Webhook` connected from `Read download file from Disk`.
      - Respond with binary file content.
      - Set `Content-Disposition` header for attachment with filename.

11. **Response Checking and Formatting:**
    - Add an **If** node named `check success`.
      - Condition: `$json.success` is true.
      - Inputs: Connect from `Check PDF file is OK`, `Check PFX file is OK`, `Generate Keys`, `Sign PDF`.
      - True output → `Prepare Success Response`.
      - False output → `POST Error Response`.
    - Add a **Set** node named `Prepare Success Response` connected from `check success` true output.
      - Set `success` = true.
      - Set `serverFileName` = `$json.fileName`.
    - Add a **RespondToWebhook** node named `POST Success Response` connected from `Prepare Success Response`.
      - Sends success JSON response.
    - Add a **RespondToWebhook** node named `POST Error Response` connected from `check success` false output.
      - Sends error JSON response.

12. **Activate the Workflow:**
    - Import the workflow JSON or recreate nodes as above.
    - Ensure environment variable `NODE_FUNCTION_ALLOW_EXTERNAL` includes required packages.
    - Activate the workflow to enable webhooks.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow requires n8n version 0.214.0 or higher and Node.js 14+.                                           | Setup Instructions                                                                              |
| Environment variable `NODE_FUNCTION_ALLOW_EXTERNAL` must include: `"node-forge,@signpdf/signpdf,@signpdf/signer-p12,@signpdf/placeholder-plain"` | Setup Instructions                                                                              |
| The workflow demonstrates multipart/form-data handling for file uploads and JSON payload processing.           | Overview                                                                                       |
| Uses `node-forge` for cryptographic key and certificate generation.                                            | Cryptographic Operations                                                                        |
| Uses `@signpdf` libraries for PDF signing and placeholder insertion.                                           | Cryptographic Operations                                                                        |
| The workflow stores files under `/data/files/` by default; adjust paths and permissions as needed.             | Setup Instructions                                                                              |
| For testing, use HTTP clients like Postman or curl to send requests to `/docu-digi-sign` and `/docu-download`. | Use Case                                                                                       |
| Sticky notes in the workflow provide section headers for easier navigation.                                    | Workflow JSON                                                                                   |

---

This document provides a complete, detailed reference for understanding, reproducing, and modifying the "Basic PDF Digital Sign Service" workflow in n8n. It covers all nodes, logic flows, configurations, and potential edge cases to ensure robust implementation and integration.