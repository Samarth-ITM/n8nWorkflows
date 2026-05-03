Parse and Extract Data from Documents/Images with Mistral OCR

https://n8nworkflows.xyz/workflows/parse-and-extract-data-from-documents-images-with-mistral-ocr-3102


# Parse and Extract Data from Documents/Images with Mistral OCR

### 1. Workflow Overview

This workflow demonstrates how to parse and extract data from documents and images using the Mistral OCR API. It targets use cases involving multi-page PDFs and single images, such as bank statements or financial reports, to extract structured data including tables and textual content. The workflow showcases three main approaches:

- **Example 1: Publicly Hosted Files** — Directly sending public URLs of PDFs or images to Mistral OCR for processing.
- **Example 2: Privately Hosted Files via Mistral Cloud** — Uploading files securely to Mistral Cloud, generating signed URLs to protect access, and then processing these URLs with Mistral OCR.
- **Example 3: Document Understanding via Chat Completion** — Using Mistral’s AI chat models to query documents directly for insights without explicit extraction.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Manual trigger and setting URLs or importing files from Google Drive.
- **1.2 Secure Upload & URL Generation:** Uploading files to Mistral Cloud and generating signed URLs.
- **1.3 OCR Processing:** Sending document or image URLs to Mistral OCR API for text extraction.
- **1.4 Document Understanding:** Querying documents or images using Mistral’s chat completion API.
- **1.5 Informational Sticky Notes:** Contextual notes explaining workflow usage and examples.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and sets up the input documents either by specifying public URLs or by importing files from Google Drive.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Document URL  
- Image URL  
- Import PDF  
- Import Image

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually.  
  - Inputs: None  
  - Outputs: Triggers two parallel paths to set document and image URLs.  
  - Edge Cases: None

- **Document URL**  
  - Type: Set  
  - Role: Defines a public URL string for a PDF document.  
  - Configuration: Assigns a string URL to the variable `url`.  
  - Inputs: Trigger node  
  - Outputs: Passes URL to OCR nodes.  
  - Edge Cases: URL must be publicly accessible.

- **Image URL**  
  - Type: Set  
  - Role: Defines a public URL string for an image file.  
  - Configuration: Assigns a string URL to the variable `url`.  
  - Inputs: Trigger node  
  - Outputs: Passes URL to OCR nodes.  
  - Edge Cases: URL must be publicly accessible.

- **Import PDF**  
  - Type: Google Drive  
  - Role: Downloads a PDF file from Google Drive by file ID.  
  - Configuration: Operation set to "download" with a specific file ID.  
  - Inputs: None (manual trigger path)  
  - Outputs: Binary data of the PDF file.  
  - Credentials: Google Drive OAuth2  
  - Edge Cases: File ID must be valid and accessible; OAuth token must be valid.

- **Import Image**  
  - Type: Google Drive  
  - Role: Downloads an image file from Google Drive by file ID.  
  - Configuration: Operation set to "download" with a specific file ID.  
  - Inputs: None (manual trigger path)  
  - Outputs: Binary data of the image file.  
  - Credentials: Google Drive OAuth2  
  - Edge Cases: File ID must be valid and accessible; OAuth token must be valid.

---

#### 1.2 Secure Upload & URL Generation

**Overview:**  
This block uploads files to Mistral Cloud storage to secure them and generates signed URLs with limited expiry for private access.

**Nodes Involved:**  
- Mistral Upload  
- Mistral Signed URL  
- Mistral Upload1  
- Mistral Signed URL1

**Node Details:**

- **Mistral Upload**  
  - Type: HTTP Request  
  - Role: Uploads the PDF binary data to Mistral Cloud with purpose "ocr".  
  - Configuration: POST to `https://api.mistral.ai/v1/files` with multipart-form-data including the file binary and purpose parameter.  
  - Inputs: Binary data from "Import PDF" node.  
  - Outputs: JSON response containing file ID.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Upload failure due to network, auth errors, or invalid file data.

- **Mistral Signed URL**  
  - Type: HTTP Request  
  - Role: Requests a signed URL for the uploaded file with 24-hour expiry.  
  - Configuration: GET to `https://api.mistral.ai/v1/files/{{ $json.id }}/url` with query parameter expiry=24.  
  - Inputs: JSON from "Mistral Upload" node containing file ID.  
  - Outputs: JSON containing signed URL.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Invalid file ID, expired or revoked credentials.

- **Mistral Upload1**  
  - Type: HTTP Request  
  - Role: Uploads the image binary data to Mistral Cloud with purpose "ocr".  
  - Configuration: Same as "Mistral Upload" but input from "Import Image".  
  - Inputs: Binary data from "Import Image" node.  
  - Outputs: JSON response with file ID.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Same as "Mistral Upload".

- **Mistral Signed URL1**  
  - Type: HTTP Request  
  - Role: Requests a signed URL for the uploaded image file with 24-hour expiry.  
  - Configuration: Same as "Mistral Signed URL" but input from "Mistral Upload1".  
  - Inputs: JSON from "Mistral Upload1" node.  
  - Outputs: JSON containing signed URL.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Same as "Mistral Signed URL".

---

#### 1.3 OCR Processing

**Overview:**  
This block sends document or image URLs to the Mistral OCR API to extract text and tables, leveraging Mistral’s latest OCR model.

**Nodes Involved:**  
- Mistral DOC OCR1  
- Mistral DOC OCR  
- Mistral IMAGE OCR1  
- Mistral IMAGE OCR

**Node Details:**

- **Mistral DOC OCR1**  
  - Type: HTTP Request  
  - Role: Sends a public document URL to Mistral OCR for processing.  
  - Configuration: POST to `https://api.mistral.ai/v1/ocr` with JSON body specifying model "mistral-ocr-latest" and document type "document_url" with URL from input.  
  - Inputs: JSON from "Document URL" node.  
  - Outputs: OCR result JSON.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Invalid URL, network errors, API rate limits.

- **Mistral DOC OCR**  
  - Type: HTTP Request  
  - Role: Sends a signed URL of uploaded PDF to Mistral OCR for processing, including base64 images.  
  - Configuration: POST with JSON body similar to "Mistral DOC OCR1" but includes `"include_image_base64": true`.  
  - Inputs: JSON from "Mistral Signed URL" node.  
  - Outputs: OCR result JSON.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Signed URL expiry, API errors.

- **Mistral IMAGE OCR1**  
  - Type: HTTP Request  
  - Role: Sends a public image URL to Mistral OCR for processing.  
  - Configuration: POST to OCR endpoint with JSON specifying model and image URL.  
  - Inputs: JSON from "Image URL" node.  
  - Outputs: OCR result JSON.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Invalid URL, unsupported image format.

- **Mistral IMAGE OCR**  
  - Type: HTTP Request  
  - Role: Sends signed URL of uploaded image to Mistral OCR for processing.  
  - Configuration: POST with JSON body specifying model and image URL from signed URL.  
  - Inputs: JSON from "Mistral Signed URL1" node.  
  - Outputs: OCR result JSON.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Signed URL expiry, API errors.

---

#### 1.4 Document Understanding

**Overview:**  
This block uses Mistral’s chat completion API to ask questions about documents or images, enabling semantic understanding beyond raw extraction.

**Nodes Involved:**  
- Document URL1  
- Document Understanding  
- Image URL1  
- Document Mis-Understanding?

**Node Details:**

- **Document URL1**  
  - Type: Set  
  - Role: Sets a public PDF document URL and a query string asking about total deposits.  
  - Configuration: Assigns `url` and `query` variables.  
  - Inputs: Manual trigger  
  - Outputs: JSON with URL and query.  
  - Edge Cases: URL accessibility, query relevance.

- **Document Understanding**  
  - Type: HTTP Request  
  - Role: Sends a chat completion request to Mistral’s small language model with document URL and query.  
  - Configuration: POST to `https://api.mistral.ai/v1/chat/completions` with JSON body containing model "mistral-small-latest", messages array with user role, text query, and document URL. Limits set for images and pages.  
  - Inputs: JSON from "Document URL1".  
  - Outputs: AI-generated answer JSON.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Model limitations, query ambiguity, API errors.

- **Image URL1**  
  - Type: Set  
  - Role: Sets a public image URL and a query string asking about total deposits.  
  - Configuration: Assigns `url` and `query` variables.  
  - Inputs: Manual trigger  
  - Outputs: JSON with URL and query.  
  - Edge Cases: URL accessibility, query relevance.

- **Document Mis-Understanding?**  
  - Type: HTTP Request  
  - Role: Sends a chat completion request to Mistral’s Pixtral large model for image-based document understanding.  
  - Configuration: POST to chat completions endpoint with model "pixtral-large-latest", messages with user role, text query, and image URL. Limits set for images and pages.  
  - Inputs: JSON from "Image URL1".  
  - Outputs: AI-generated answer JSON.  
  - Credentials: Mistral Cloud API key  
  - Edge Cases: Pixtral model quality issues, image complexity, API errors.

---

#### 1.5 Informational Sticky Notes

**Overview:**  
Sticky notes provide contextual explanations and guidance for users on workflow usage, examples, and requirements.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3

**Node Details:**

- **Sticky Note**  
  - Content: Explains Example 1 - using publicly hosted files for Mistral OCR.  
  - Position: Near input URL nodes.

- **Sticky Note1**  
  - Content: Explains Example 2 - using Mistral Cloud for private and secure file hosting with signed URLs.  
  - Position: Near upload and signed URL nodes.

- **Sticky Note2**  
  - Content: Overview of Mistral OCR capabilities, pricing, and requirements.  
  - Position: Top-left corner, general info.

- **Sticky Note3**  
  - Content: Explains Example 3 - document understanding via chat models and limitations with images.  
  - Position: Near document understanding nodes.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                  | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                          |
|-------------------------|---------------------|-------------------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts the workflow manually                      | -                           | Document URL, Image URL     |                                                                                                    |
| Document URL            | Set                 | Sets public PDF document URL                      | When clicking ‘Test workflow’| Mistral DOC OCR1           | Example 1: Publicly Hosted Files - default usage for public URLs                                   |
| Image URL               | Set                 | Sets public image URL                             | When clicking ‘Test workflow’| Mistral IMAGE OCR1         | Example 1: Publicly Hosted Files - default usage for public URLs                                   |
| Import PDF              | Google Drive        | Downloads PDF file from Google Drive              | -                           | Mistral Upload             |                                                                                                    |
| Import Image            | Google Drive        | Downloads image file from Google Drive            | -                           | Mistral Upload1            |                                                                                                    |
| Mistral Upload          | HTTP Request        | Uploads PDF to Mistral Cloud                       | Import PDF                  | Mistral Signed URL         | Example 2: Privately Hosted via Mistral Cloud - secure upload and access                            |
| Mistral Signed URL      | HTTP Request        | Generates signed URL for uploaded PDF             | Mistral Upload              | Mistral DOC OCR            | Example 2: Privately Hosted via Mistral Cloud - secure upload and access                            |
| Mistral Upload1         | HTTP Request        | Uploads image to Mistral Cloud                     | Import Image                | Mistral Signed URL1        | Example 2: Privately Hosted via Mistral Cloud - secure upload and access                            |
| Mistral Signed URL1     | HTTP Request        | Generates signed URL for uploaded image            | Mistral Upload1             | Mistral IMAGE OCR          | Example 2: Privately Hosted via Mistral Cloud - secure upload and access                            |
| Mistral DOC OCR1        | HTTP Request        | Sends public PDF URL to Mistral OCR                | Document URL                | (end)                     | Example 1: Publicly Hosted Files - default usage for public URLs                                   |
| Mistral DOC OCR         | HTTP Request        | Sends signed URL PDF to Mistral OCR with base64 images | Mistral Signed URL          | (end)                     | Example 2: Privately Hosted via Mistral Cloud - secure upload and access                            |
| Mistral IMAGE OCR1      | HTTP Request        | Sends public image URL to Mistral OCR              | Image URL                   | (end)                     | Example 1: Publicly Hosted Files - default usage for public URLs                                   |
| Mistral IMAGE OCR       | HTTP Request        | Sends signed URL image to Mistral OCR              | Mistral Signed URL1         | (end)                     | Example 2: Privately Hosted via Mistral Cloud - secure upload and access                            |
| Document URL1           | Set                 | Sets public PDF URL and query for document understanding | When clicking ‘Test workflow’| Document Understanding     | Example 3: No need for Extraction? Talk Directly with the File!                                    |
| Document Understanding  | HTTP Request        | Queries document via Mistral chat completion API   | Document URL1               | (end)                     | Example 3: No need for Extraction? Talk Directly with the File!                                    |
| Image URL1              | Set                 | Sets public image URL and query for document understanding | When clicking ‘Test workflow’| Document Mis-Understanding? | Example 3: No need for Extraction? Talk Directly with the File!                                    |
| Document Mis-Understanding? | HTTP Request    | Queries image via Pixtral chat completion API      | Image URL1                  | (end)                     | Example 3: No need for Extraction? Talk Directly with the File!                                    |
| Sticky Note             | Sticky Note         | Explains Example 1 usage                            | -                           | -                          | Example 1: Publicly Hosted Files - default usage for public URLs                                   |
| Sticky Note1            | Sticky Note         | Explains Example 2 usage                            | -                           | -                          | Example 2: Privately Hosted via Mistral Cloud - secure upload and access                            |
| Sticky Note2            | Sticky Note         | General info on Mistral OCR capabilities and pricing | -                           | -                          | General overview and requirements for Mistral OCR                                                 |
| Sticky Note3            | Sticky Note         | Explains Example 3 usage and limitations           | -                           | -                          | Example 3: No need for Extraction? Talk Directly with the File!                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Test workflow’`.

2. **Create two Set nodes**:
   - `Document URL`: Assign a string variable `url` with a public PDF URL (e.g., `"https://pub-d4aa9be14ae34d6ebcebe06f13af667b.r2.dev/multimodal_bank_statement_scan.pdf"`).
   - `Image URL`: Assign a string variable `url` with a public image URL (e.g., `"https://pub-d4aa9be14ae34d6ebcebe06f13af667b.r2.dev/multimodal_bank_statement_2.png"`).

3. **Connect the Manual Trigger node outputs** to both `Document URL` and `Image URL`.

4. **Create two Google Drive nodes**:
   - `Import PDF`: Set operation to "download" and specify the PDF file ID to download.
   - `Import Image`: Set operation to "download" and specify the image file ID to download.

5. **Create two HTTP Request nodes for uploading files to Mistral Cloud**:
   - `Mistral Upload`:  
     - Method: POST  
     - URL: `https://api.mistral.ai/v1/files`  
     - Content-Type: multipart-form-data  
     - Body parameters:  
       - `purpose`: "ocr"  
       - `file`: binary data from `Import PDF` node  
     - Authentication: Use Mistral Cloud API credentials.

   - `Mistral Upload1`: Same as above but input binary from `Import Image`.

6. **Create two HTTP Request nodes to generate signed URLs**:
   - `Mistral Signed URL`:  
     - Method: GET  
     - URL: `https://api.mistral.ai/v1/files/{{ $json.id }}/url` (use expression to insert file ID)  
     - Query parameter: `expiry=24` (hours)  
     - Header: `Accept: application/json`  
     - Authentication: Mistral Cloud API credentials.

   - `Mistral Signed URL1`: Same as above but input from `Mistral Upload1`.

7. **Create four HTTP Request nodes for OCR processing**:
   - `Mistral DOC OCR1`:  
     - Method: POST  
     - URL: `https://api.mistral.ai/v1/ocr`  
     - Body (JSON):  
       ```json
       {
         "model": "mistral-ocr-latest",
         "document": {
           "type": "document_url",
           "document_url": "{{ $json.url }}"
         }
       }
       ```  
     - Input: from `Document URL` node  
     - Authentication: Mistral Cloud API credentials.

   - `Mistral DOC OCR`:  
     - Same as above but input from `Mistral Signed URL` node and JSON body includes `"include_image_base64": true`.

   - `Mistral IMAGE OCR1`:  
     - Method: POST  
     - URL: `https://api.mistral.ai/v1/ocr`  
     - Body (JSON):  
       ```json
       {
         "model": "mistral-ocr-latest",
         "document": {
           "type": "image_url",
           "image_url": "{{ $json.url }}"
         }
       }
       ```  
     - Input: from `Image URL` node  
     - Authentication: Mistral Cloud API credentials.

   - `Mistral IMAGE OCR`:  
     - Same as above but input from `Mistral Signed URL1` node.

8. **Create two Set nodes for document understanding queries**:
   - `Document URL1`: Assign variables:  
     - `url`: public PDF URL  
     - `query`: e.g., "what is the total number of deposits?"  
   - `Image URL1`: Assign variables:  
     - `url`: public image URL  
     - `query`: same or similar question.

9. **Create two HTTP Request nodes for chat completion queries**:
   - `Document Understanding`:  
     - Method: POST  
     - URL: `https://api.mistral.ai/v1/chat/completions`  
     - Body (JSON):  
       ```json
       {
         "model": "mistral-small-latest",
         "messages": [
           {
             "role": "user",
             "content": [
               { "type": "text", "text": "{{ $json.query }}" },
               { "type": "document_url", "document_url": "{{ $json.url }}" }
             ]
           }
         ],
         "document_image_limit": 8,
         "document_page_limit": 64
       }
       ```  
     - Input: from `Document URL1`  
     - Authentication: Mistral Cloud API credentials.

   - `Document Mis-Understanding?`:  
     - Method: POST  
     - URL: same as above  
     - Body (JSON):  
       ```json
       {
         "model": "pixtral-large-latest",
         "messages": [
           {
             "role": "user",
             "content": [
               { "type": "text", "text": "{{ $json.query }}" },
               { "type": "image_url", "image_url": "{{ $json.url }}" }
             ]
           }
         ],
         "document_image_limit": 8,
         "document_page_limit": 64
       }
       ```  
     - Input: from `Image URL1`  
     - Authentication: Mistral Cloud API credentials.

10. **Connect nodes accordingly**:
    - `When clicking ‘Test workflow’` → `Document URL` and `Image URL` and also → `Document URL1` and `Image URL1`.
    - `Import PDF` → `Mistral Upload` → `Mistral Signed URL` → `Mistral DOC OCR`.
    - `Import Image` → `Mistral Upload1` → `Mistral Signed URL1` → `Mistral IMAGE OCR`.
    - `Document URL` → `Mistral DOC OCR1`.
    - `Image URL` → `Mistral IMAGE OCR1`.
    - `Document URL1` → `Document Understanding`.
    - `Image URL1` → `Document Mis-Understanding?`.

11. **Add Sticky Notes** with the provided content near relevant nodes for user guidance.

12. **Configure credentials**:
    - Google Drive OAuth2 for `Import PDF` and `Import Image`.
    - Mistral Cloud API key for all HTTP Request nodes interacting with Mistral endpoints.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Mistral OCR is designed specifically for parsing PDFs and images, supporting multi-page documents and images up to 10k pixels.          | General workflow description                                                                                     |
| Pricing is very competitive at $0.001 per page, with batching options available for cost savings at the expense of processing time.       | Workflow description                                                                                            |
| Official Mistral OCR API documentation: [https://docs.mistral.ai/capabilities/document/#tag/ocr/operation/ocr_v1_ocr_post](https://docs.mistral.ai/capabilities/document/#tag/ocr/operation/ocr_v1_ocr_post) | API reference                                                                                                   |
| For privacy, uploading files to Mistral Cloud and using signed URLs is recommended over public URLs.                                      | Workflow description and Sticky Note1                                                                           |
| Mistral’s chat completion models enable querying documents semantically, but image-based querying via Pixtral is less reliable currently. | Sticky Note3                                                                                                    |

---

This comprehensive documentation enables advanced users and AI agents to fully understand, reproduce, and extend the workflow while anticipating potential integration issues and edge cases.