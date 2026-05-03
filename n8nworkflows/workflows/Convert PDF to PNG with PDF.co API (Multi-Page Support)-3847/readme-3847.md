Convert PDF to PNG with PDF.co API (Multi-Page Support)

https://n8nworkflows.xyz/workflows/convert-pdf-to-png-with-pdf-co-api--multi-page-support--3847


# Convert PDF to PNG with PDF.co API (Multi-Page Support)

### 1. Workflow Overview

This workflow automates the conversion of each page of a multi-page PDF document into separate high-quality PNG images using the PDF.co API. It is designed to handle multi-page PDFs efficiently by splitting the PDF into pages, converting each page to an image, and then aggregating the images.

**Target use cases:**  
- Automating PDF page-to-image conversion for multi-page PDFs.  
- Processing batches of PDFs to generate PNG images per page.  
- Integrating PDF conversion into larger document processing pipelines.

**Logical blocks:**

- **1.1 Input Acquisition:** Fetch example PDF URLs from a webpage, extract and select relevant PDFs to process.  
- **1.2 PDF Download & Upload Preparation:** Download PDFs, obtain presigned upload URLs from PDF.co, upload PDFs to these URLs.  
- **1.3 PDF Splitting & Conversion:** Split multi-page PDFs into individual pages, convert each page to PNG via PDF.co API.  
- **1.4 Image Download & Aggregation:** Download generated PNG images, aggregate them into a zipped file.  
- **1.5 Workflow Trigger:** Manual trigger node to start the process.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Acquisition

**Overview:**  
This block retrieves example PDF links from a webpage, parses the HTML to extract PDF URLs, limits the selection to two relevant examples, and prepares them for download.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- GET example PDF files  
- Extract PDF links  
- Split links into items  
- Use two relevant PDF examples

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually.  
  - Inputs: None (trigger only)  
  - Outputs: Triggers “GET example PDF files”  
  - Failure modes: None typical; manual trigger.

- **GET example PDF files**  
  - Type: HTTP Request  
  - Role: Fetches a webpage containing sample PDF links.  
  - Configuration: HTTP GET to a predefined URL hosting sample PDFs.  
  - Outputs: HTML content containing PDF links.  
  - Failure modes: HTTP errors, network timeouts.

- **Extract PDF links**  
  - Type: HTML Extract  
  - Role: Parses HTML to extract PDF URLs.  
  - Configuration: CSS selector or XPath to select anchor tags or links pointing to PDFs.  
  - Inputs: HTML from previous node.  
  - Outputs: List of PDF URLs.  
  - Failure modes: Parsing errors if HTML structure changes.

- **Split links into items**  
  - Type: SplitOut  
  - Role: Splits array of PDF URLs into individual items for iteration.  
  - Inputs: List of URLs.  
  - Outputs: Single URL per item.  
  - Failure modes: None typical.

- **Use two relevant PDF examples**  
  - Type: Limit  
  - Role: Limits the number of URLs processed to two for demonstration.  
  - Inputs: Stream of URLs.  
  - Outputs: First two URLs only.  
  - Failure modes: None typical.

---

#### 2.2 PDF Download & Upload Preparation

**Overview:**  
Downloads the selected PDF files, obtains a presigned upload URL from PDF.co for each PDF, and uploads the PDFs to PDF.co storage for processing.

**Nodes Involved:**  
- Download example PDF Files  
- Pass through binary  
- Get Presigned Upload URL (PDF.co)  
- Combine binary and json data  
- Upload PDF to Presigned URL  
- Strip binary data  
- Combine data

**Node Details:**

- **Download example PDF Files**  
  - Type: HTTP Request  
  - Role: Downloads the actual PDF file from the URL.  
  - Configuration: HTTP GET with binary response enabled.  
  - Inputs: PDF URL from “Use two relevant PDF examples”.  
  - Outputs: PDF binary data.  
  - Failure modes: HTTP errors, file too large, network issues.

- **Pass through binary**  
  - Type: Set  
  - Role: Passes the binary data unchanged for further processing.  
  - Configuration: No changes, just passes binary data forward.  
  - Inputs: PDF binary.  
  - Outputs: Binary data forwarded.  
  - Failure modes: None typical.

- **Get Presigned Upload URL (PDF.co)**  
  - Type: HTTP Request  
  - Role: Requests a presigned URL from PDF.co API to upload the PDF.  
  - Configuration: HTTP POST/GET to PDF.co endpoint with API Key in headers for authentication.  
  - Inputs: Triggered per file.  
  - Outputs: JSON containing upload URL and other metadata.  
  - Failure modes: Authentication errors (invalid API Key), network timeout, API quota exceeded.

- **Combine binary and json data**  
  - Type: Merge  
  - Role: Combines the binary PDF data and JSON response containing the presigned URL into a single item.  
  - Inputs: Binary PDF data and JSON metadata from presigned URL node.  
  - Outputs: Combined data for upload.  
  - Failure modes: Data mismatch or missing inputs.

- **Upload PDF to Presigned URL**  
  - Type: HTTP Request  
  - Role: Uploads the PDF binary data to the presigned URL provided by PDF.co.  
  - Configuration: HTTP PUT or POST depending on presigned URL requirements, binary data as payload, no authentication needed here (presigned URL).  
  - Inputs: Combined binary and JSON data.  
  - Outputs: Upload status response.  
  - Failure modes: Upload failure, URL expired.

- **Strip binary data**  
  - Type: Set  
  - Role: Removes binary data from the output to reduce payload size for subsequent nodes.  
  - Inputs: Upload response with binary attached.  
  - Outputs: Non-binary JSON data.  
  - Failure modes: None typical.

- **Combine data**  
  - Type: Merge  
  - Role: Aggregates upload confirmation data with other inputs to prepare for splitting PDF.  
  - Inputs: Upload response and possibly other metadata.  
  - Outputs: Combined item for next block.  
  - Failure modes: Mismatched inputs.

---

#### 2.3 PDF Splitting & Conversion

**Overview:**  
Splits each multi-page PDF into individual pages, converts each page to PNG images using the PDF.co API, and supports batch processing of pages.

**Nodes Involved:**  
- Split multipage PDF files  
- Loop over pdf files  
- Split out urls  
- Convert PDF to PNG (PDF.co)

**Node Details:**

- **Split multipage PDF files**  
  - Type: HTTP Request  
  - Role: Calls PDF.co API to split multi-page PDF into single-page PDFs or URLs per page.  
  - Configuration: POST request to PDF.co split endpoint with the uploaded PDF URL and API Key.  
  - Inputs: PDF upload confirmation and URL.  
  - Outputs: URLs or references to individual page PDFs.  
  - Failure modes: API errors, invalid PDF, network issues.

- **Loop over pdf files**  
  - Type: SplitInBatches  
  - Role: Iterates over each page PDF URL to process them individually.  
  - Configuration: Batch size configurable (default 1).  
  - Inputs: List of single-page PDFs URLs.  
  - Outputs: Single page URL per iteration.  
  - Failure modes: Batch size misconfiguration.

- **Split out urls**  
  - Type: SplitOut  
  - Role: Splits array of URLs for processing.  
  - Inputs: URLs of PNG images or pages.  
  - Outputs: Single URL per item.  
  - Failure modes: None typical.

- **Convert PDF to PNG (PDF.co)**  
  - Type: HTTP Request  
  - Role: Converts a single-page PDF URL to PNG using PDF.co API.  
  - Configuration: POST request with PDF URL, conversion parameters (PNG output, quality settings), API Key in headers.  
  - Inputs: URL of single-page PDF.  
  - Outputs: URL(s) to PNG image(s).  
  - Failure modes: API errors, invalid parameters, quota limits, timeouts.

---

#### 2.4 Image Download & Aggregation

**Overview:**  
Downloads the converted PNG images from PDF.co, aggregates the binary images, compresses them into a ZIP file for output.

**Nodes Involved:**  
- Download PNG from PDF.co  
- Combine binaries  
- Compress into zip file

**Node Details:**

- **Download PNG from PDF.co**  
  - Type: HTTP Request  
  - Role: Downloads the PNG images (binary data) by fetching URLs returned by the conversion API.  
  - Configuration: HTTP GET with binary response enabled.  
  - Inputs: PNG image URLs from conversion node.  
  - Outputs: PNG binary data.  
  - Failure modes: HTTP errors, missing files.

- **Combine binaries**  
  - Type: Aggregate  
  - Role: Aggregates all PNG binary data into a collection for compression.  
  - Inputs: Multiple binary PNG files.  
  - Outputs: Aggregated binary data array.  
  - Failure modes: Memory issues if too many files.

- **Compress into zip file**  
  - Type: Compression  
  - Role: Compresses aggregated PNG binaries into a single ZIP archive.  
  - Inputs: Aggregated binaries.  
  - Outputs: ZIP binary file.  
  - Failure modes: Compression failure, memory limits.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                             | Input Node(s)                      | Output Node(s)                     | Sticky Note                                |
|-------------------------------|--------------------|---------------------------------------------|----------------------------------|----------------------------------|--------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Starts the workflow manually                 | -                                | GET example PDF files             |                                            |
| GET example PDF files         | HTTP Request       | Fetch webpage with sample PDF links          | When clicking ‘Test workflow’    | Extract PDF links                |                                            |
| Extract PDF links             | HTML Extract       | Extracts PDF URLs from HTML                   | GET example PDF files            | Split links into items           |                                            |
| Split links into items        | SplitOut           | Splits array of PDF URLs into single items   | Extract PDF links                | Use two relevant PDF examples    |                                            |
| Use two relevant PDF examples | Limit              | Limits processing to two PDFs                  | Split links into items           | Download example PDF Files       |                                            |
| Download example PDF Files    | HTTP Request       | Downloads PDF binary files                      | Use two relevant PDF examples    | Pass through binary, Get Presigned Upload URL (PDF.co) |                                            |
| Pass through binary           | Set                | Passes binary data unchanged                   | Download example PDF Files       | Combine binary and json data     |                                            |
| Get Presigned Upload URL (PDF.co) | HTTP Request       | Gets presigned upload URL for PDF                | Download example PDF Files       | Combine binary and json data     |                                            |
| Combine binary and json data  | Merge              | Merges binary PDF and presigned URL metadata | Pass through binary, Get Presigned Upload URL (PDF.co) | Upload PDF to Presigned URL, Strip binary data |                                            |
| Upload PDF to Presigned URL   | HTTP Request       | Uploads PDF binary to presigned URL            | Combine binary and json data     | Combine data                    |                                            |
| Strip binary data             | Set                | Removes binary data to reduce payload          | Upload PDF to Presigned URL       | Combine data                    |                                            |
| Combine data                 | Merge              | Combines upload response with metadata          | Upload PDF to Presigned URL, Strip binary data | Split multipage PDF files      |                                            |
| Split multipage PDF files     | HTTP Request       | Splits multi-page PDF into single page PDFs    | Combine data                    | Loop over pdf files              |                                            |
| Loop over pdf files           | SplitInBatches     | Iterates over single-page PDFs for processing  | Split multipage PDF files         | Compress into zip file, Split out urls |                                            |
| Split out urls                | SplitOut           | Splits PNG URLs into single items               | Loop over pdf files              | Convert PDF to PNG (PDF.co)      |                                            |
| Convert PDF to PNG (PDF.co)   | HTTP Request       | Converts single PDF page to PNG via API         | Split out urls                  | Download PNG from PDF.co         |                                            |
| Download PNG from PDF.co      | HTTP Request       | Downloads PNG binary images                      | Convert PDF to PNG (PDF.co)       | Combine binaries                |                                            |
| Combine binaries              | Aggregate          | Aggregates PNG binaries for compression          | Download PNG from PDF.co          | Loop over pdf files             |                                            |
| Compress into zip file        | Compression        | Compresses aggregated PNGs into ZIP              | Loop over pdf files              | -                              |                                            |
| Sticky Note                  | Sticky Note        | -                                               | -                              | -                              |                                            |
| Sticky Note1                 | Sticky Note        | -                                               | -                              | -                              |                                            |
| Sticky Note2                 | Sticky Note        | -                                               | -                              | -                              |                                            |
| Sticky Note3                 | Sticky Note        | -                                               | -                              | -                              |                                            |
| Sticky Note4                 | Sticky Note        | -                                               | -                              | -                              |                                            |
| Sticky Note5                 | Sticky Note        | -                                               | -                              | -                              |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: “When clicking ‘Test workflow’”  
   - Purpose: To manually start the workflow.

2. **Add an HTTP Request node:**  
   - Name: “GET example PDF files”  
   - Method: GET  
   - URL: The webpage URL hosting sample PDF links.  
   - Response: Set to return raw response (HTML).  
   - Connect from Manual Trigger.

3. **Add an HTML Extract node:**  
   - Name: “Extract PDF links”  
   - Input: From “GET example PDF files”.  
   - Extraction: Use CSS selectors or XPath targeting `<a>` tags or links ending with `.pdf` to extract URLs.

4. **Add a SplitOut node:**  
   - Name: “Split links into items”  
   - Input: From “Extract PDF links”.  
   - Purpose: Split array of URLs into individual items.

5. **Add a Limit node:**  
   - Name: “Use two relevant PDF examples”  
   - Input: From “Split links into items”.  
   - Limit: Set to 2 to process only two PDFs.

6. **Add an HTTP Request node:**  
   - Name: “Download example PDF Files”  
   - Method: GET  
   - URL: Use expression to set the URL from the limited PDF URLs.  
   - Response: Set to “File” or binary to download the PDF content.  
   - Input: From “Use two relevant PDF examples”.

7. **Add a Set node:**  
   - Name: “Pass through binary”  
   - Purpose: Pass the downloaded binary PDF unchanged.  
   - Input: From “Download example PDF Files”.

8. **Add an HTTP Request node:**  
   - Name: “Get Presigned Upload URL (PDF.co)”  
   - Method: POST or GET depending on PDF.co API.  
   - URL: PDF.co API endpoint to get presigned upload URL.  
   - Headers: Add `x-api-key` with your PDF.co API Key.  
   - Input: From “Download example PDF Files”.

9. **Add a Merge node:**  
   - Name: “Combine binary and json data”  
   - Mode: Merge by index or key.  
   - Inputs: From “Pass through binary” (binary data) and “Get Presigned Upload URL (PDF.co)” (JSON).  
   - Purpose: Combine binary PDF with presigned URL info.

10. **Add an HTTP Request node:**  
    - Name: “Upload PDF to Presigned URL”  
    - Method: PUT or POST based on presigned URL requirements.  
    - URL: Use presigned URL from merge node data.  
    - Body Content: Binary PDF data.  
    - Authentication: None (presigned URL).  
    - Input: From “Combine binary and json data”.

11. **Add a Set node:**  
    - Name: “Strip binary data”  
    - Purpose: Remove binary data from output to keep payload light.  
    - Input: From “Upload PDF to Presigned URL”.

12. **Add a Merge node:**  
    - Name: “Combine data”  
    - Mode: Merge inputs.  
    - Inputs: From “Upload PDF to Presigned URL” and “Strip binary data”.  
    - Purpose: Prepare data for splitting PDF.

13. **Add an HTTP Request node:**  
    - Name: “Split multipage PDF files”  
    - Method: POST  
    - URL: PDF.co API endpoint for splitting PDFs.  
    - Body: Include URL of uploaded PDF and parameters to split pages.  
    - Headers: Include `x-api-key` with API Key.  
    - Input: From “Combine data”.

14. **Add a SplitInBatches node:**  
    - Name: “Loop over pdf files”  
    - Input: From “Split multipage PDF files”.  
    - Purpose: Iterate over each single-page PDF URL.  
    - Batch Size: Default 1.

15. **Add a SplitOut node:**  
    - Name: “Split out urls”  
    - Input: From “Loop over pdf files”.  
    - Purpose: Split URLs of PNG images for individual processing.

16. **Add an HTTP Request node:**  
    - Name: “Convert PDF to PNG (PDF.co)”  
    - Method: POST  
    - URL: PDF.co API endpoint for PDF to PNG conversion.  
    - Body: Include single-page PDF URL and conversion parameters (PNG output, quality).  
    - Headers: Include `x-api-key` with API Key.  
    - Input: From “Split out urls”.

17. **Add an HTTP Request node:**  
    - Name: “Download PNG from PDF.co”  
    - Method: GET  
    - URL: Use PNG image URL from conversion output.  
    - Response: Binary (PNG images).  
    - Input: From “Convert PDF to PNG (PDF.co)”.

18. **Add an Aggregate node:**  
    - Name: “Combine binaries”  
    - Input: From “Download PNG from PDF.co”.  
    - Purpose: Collect all PNG files to prepare for compression.

19. **Connect “Combine binaries” to “Loop over pdf files” as next step:**  
    - This enables looping until all pages processed.

20. **Add a Compression node:**  
    - Name: “Compress into zip file”  
    - Input: From “Loop over pdf files”.  
    - Compression: ZIP format with all PNG binaries.  
    - Output: Single ZIP file containing all page images.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                        |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Create a PDF.co Account and obtain your API Key to authenticate all PDF.co API requests.                 | https://www.pdf.co/                                   |
| PDF.co API documentation and introduction for developer reference.                                      | https://developer.pdf.co/api/introduction/index.html  |
| To customize workflow for your PDFs, replace example PDF fetching logic with your own PDF retrieval method (upload, URL, storage). | Workflow description                                  |
| All API requests are made securely over HTTPS with API Key authentication for privacy and access control. | Security best practice                                |

---

This documentation enables clear understanding, reproduction, and modification of the PDF to PNG multi-page conversion workflow using n8n and PDF.co API. It anticipates potential errors like network issues, API authentication failures, and data parsing problems, and provides a modular breakdown for maintainability and extensibility.