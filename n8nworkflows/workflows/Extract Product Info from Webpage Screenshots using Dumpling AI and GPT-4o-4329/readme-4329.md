Extract Product Info from Webpage Screenshots using Dumpling AI and GPT-4o

https://n8nworkflows.xyz/workflows/extract-product-info-from-webpage-screenshots-using-dumpling-ai-and-gpt-4o-4329


# Extract Product Info from Webpage Screenshots using Dumpling AI and GPT-4o

### 1. Workflow Overview

This workflow automates the extraction of detailed product information from webpage screenshots using Dumpling AI and OpenAI's GPT-4o model. It is designed for scenarios where users track competitor product pages or e-commerce listings by URL, capture their full-page screenshots, extract all visible textual and UI data, and then distill structured product details for analysis or record-keeping.

The workflow is logically divided into two main blocks:

- **1.1 Screenshot Capture and Text Extraction:**  
  Starts with monitoring a Google Sheet for new URLs. Upon detecting a new URL, it triggers Dumpling AI to take a full-page screenshot, extract all visible content from the screenshot, download and save the screenshot to Google Drive, and log the screenshot URL back into the spreadsheet.

- **1.2 Product Data Extraction and Storage:**  
  Processes the extracted raw textual data through GPT-4o, which analyzes and extracts structured product information in JSON format. Each product entry is then split into individual records and saved into another Google Sheet for organized product data tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Screenshot Capture and Text Extraction

**Overview:**  
This block listens for new URLs added to a Google Sheet. For each URL, it uses Dumpling AI services to capture a full-page screenshot and extract all visible textual and UI information from that screenshot. The screenshot is then downloaded and stored in Google Drive, and its URL is logged back to the original Google Sheet.

**Nodes Involved:**  
- Trigger on New URL in Sheet  
- Take Full-Page Screenshot using Dumpling AI  
- Extract All Visible Data from Screenshot (Dumpling AI)  
- Download Screenshot File  
- Save Screenshot to Drive Folder  
- Log Screenshot URL to Spreadsheet  

**Node Details:**

- **Trigger on New URL in Sheet**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Watches a specific Google Sheet (“Competitors” document, “Sheet1”) for new rows added (new URLs).  
  - *Configuration:* Polls every minute for added rows. Uses OAuth2 credentials for Google Sheets access.  
  - *Input/Output:* No input; outputs the new row data including a URL field.  
  - *Edge cases:* Network connectivity or Google API rate limits may cause missed triggers or delays. Authentication expiry must be managed.

- **Take Full-Page Screenshot using Dumpling AI**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Dumpling AI’s screenshot API with the new URL to capture a full-page screenshot.  
  - *Configuration:*  
    - URL: https://app.dumplingai.com/api/v1/screenshot  
    - Body: JSON containing the URL from the trigger and `fullPage: true`.  
    - Authentication: HTTP header auth with preset credentials.  
  - *Input:* URL from previous node.  
  - *Output:* JSON containing screenshot metadata including a screenshot URL.  
  - *Edge cases:* API rate limits, invalid URLs, or Dumpling AI service downtime can cause failures.

- **Extract All Visible Data from Screenshot (Dumpling AI)**  
  - *Type:* HTTP Request  
  - *Role:* Uses Dumpling AI’s extract-image API to analyze the screenshot URL and extract all visible text and UI elements in a detailed, structured plain text format.  
  - *Configuration:*  
    - URL: https://app.dumplingai.com/api/v1/extract-image  
    - Body: JSON with image URL(s) and a detailed prompt instructing Dumpling AI to extract all visible information exactly as presented, preserving layout and content.  
    - Authentication: Same HTTP header auth credentials.  
  - *Input:* Screenshot URL from previous node.  
  - *Output:* JSON containing extracted textual content results.  
  - *Edge cases:* Large images may cause timeouts; incomplete or corrupted screenshot URLs may cause extraction errors.

- **Download Screenshot File**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the actual screenshot file from the URL provided by Dumpling AI.  
  - *Configuration:* Performs a GET request on the screenshot URL.  
  - *Input:* Screenshot URL extracted from the screenshot node’s output.  
  - *Output:* Binary file data of the screenshot.  
  - *Edge cases:* URL expiration or access issues; large file size may cause timeouts.

- **Save Screenshot to Drive Folder**  
  - *Type:* Google Drive  
  - *Role:* Uploads the downloaded screenshot file to a specific Google Drive folder (“Webpage-Screenshoot” folder in “My Drive”).  
  - *Configuration:* Uses OAuth2 credentials for Google Drive; target folder specified.  
  - *Input:* Binary screenshot file from previous node.  
  - *Output:* Metadata of the uploaded file (file ID, URL).  
  - *Edge cases:* Permission errors, quota limits on Drive storage.

- **Log Screenshot URL to Spreadsheet**  
  - *Type:* Google Sheets  
  - *Role:* Appends or updates a row in the original Google Sheet logging the original URL and the screenshot URL returned by Dumpling AI.  
  - *Configuration:* Uses “appendOrUpdate” operation keyed by URL to avoid duplicates; writes to “Sheet1” in the “Competitors” document.  
  - *Input:* Original URL and screenshot URL from prior nodes.  
  - *Output:* Confirmation of sheet update.  
  - *Edge cases:* Google Sheets API limits; concurrent writes causing conflicts.

#### 2.2 Product Data Extraction and Storage

**Overview:**  
This block takes the plain text extracted from the screenshot and feeds it into GPT-4o, which parses and extracts structured product information. The products are split into individual records and finally appended into a separate Google Sheet for further analysis.

**Nodes Involved:**  
- Extract Product Info from Screenshot Text with GPT-4o  
- Split Each Product into Individual Record  
- Save Products info to Google Sheet  

**Node Details:**

- **Extract Product Info from Screenshot Text with GPT-4o**  
  - *Type:* OpenAI (Langchain node)  
  - *Role:* Processes the extracted text with a system prompt instructing GPT-4o to extract product-related information into JSON format.  
  - *Configuration:*  
    - Model: GPT-4o  
    - System message prompt includes JSON schema with fields: name, ratings, purchased, price, deal, delivery, buyingOptions, colorOptions, badges.  
    - Input message includes the raw extracted text from Dumpling AI’s extraction node.  
    - Output: JSON parsed response with a top-level “products” array.  
  - *Input:* Text extracted from screenshot.  
  - *Output:* JSON object with structured product data.  
  - *Edge cases:* GPT hallucination, incomplete or ambiguous input text, API rate limits or quota errors.

- **Split Each Product into Individual Record**  
  - *Type:* Split Out  
  - *Role:* Splits the JSON array of products into individual output items, one per product, to enable separate processing or storage.  
  - *Configuration:* Field to split out is `message.content.products`.  
  - *Input:* JSON output from GPT-4o node.  
  - *Output:* Multiple items, each representing a single product.  
  - *Edge cases:* Empty or malformed arrays; missing fields.

- **Save Products info to Google Sheet**  
  - *Type:* Google Sheets  
  - *Role:* Appends each individual product record to a dedicated Google Sheet (“Sheet2” in “Competitors” document) with mapped columns for product attributes.  
  - *Configuration:*  
    - Columns include Product Name, price, purchased, ratings, deal, buyingOptions.  
    - Uses “append” operation (no deduplication).  
  - *Input:* Individual product JSON objects from Split Out node.  
  - *Output:* Confirmation of rows added.  
  - *Edge cases:* Google Sheets quota limits, missing or unexpected data types.

---

### 3. Summary Table

| Node Name                               | Node Type                   | Functional Role                                   | Input Node(s)                           | Output Node(s)                                  | Sticky Note                                                                                                        |
|----------------------------------------|-----------------------------|-------------------------------------------------|----------------------------------------|------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Trigger on New URL in Sheet             | Google Sheets Trigger       | Detect new URLs added to the source Google Sheet | None                                   | Take Full-Page Screenshot using Dumpling AI    | ### 📸 Automate Screenshot Capture & Text Extraction\nThis first part of the workflow begins by monitoring a Google Sheet for new URLs... |
| Take Full-Page Screenshot using Dumpling AI | HTTP Request                | Take full-page screenshot of the URL using Dumpling AI API | Trigger on New URL in Sheet            | Extract All Visible Data from Screenshot (Dumpling AI) |                                                                                                                     |
| Extract All Visible Data from Screenshot (Dumpling AI) | HTTP Request                | Extract all visible text and UI elements from screenshot | Take Full-Page Screenshot using Dumpling AI | Download Screenshot File                        |                                                                                                                     |
| Download Screenshot File                | HTTP Request                | Download the screenshot image file               | Extract All Visible Data from Screenshot (Dumpling AI) | Save Screenshot to Drive Folder                 |                                                                                                                     |
| Save Screenshot to Drive Folder         | Google Drive                | Upload screenshot file to Google Drive folder    | Download Screenshot File                | Log Screenshot URL to Spreadsheet                |                                                                                                                     |
| Log Screenshot URL to Spreadsheet       | Google Sheets               | Log URL and screenshot URL to original Google Sheet | Save Screenshot to Drive Folder         | Extract Product Info from Screenshot Text with GPT-4o |                                                                                                                     |
| Extract Product Info from Screenshot Text with GPT-4o | OpenAI (Langchain)          | Extract structured product info from raw text using GPT-4o | Log Screenshot URL to Spreadsheet       | Split Each Product into Individual Record        | ### 🛍️ Extract and Store Product Listings with GPT-4o\nAfter saving the screenshot, GPT-4o processes the extracted text... |
| Split Each Product into Individual Record | Split Out                   | Split JSON array of products into individual items | Extract Product Info from Screenshot Text with GPT-4o | Save Products info to Google Sheet               |                                                                                                                     |
| Save Products info to Google Sheet       | Google Sheets               | Append individual product records to dedicated Google Sheet | Split Each Product into Individual Record | None                                           |                                                                                                                     |
| Sticky Note                            | Sticky Note                 | Documentation note for first block                | None                                   | None                                           | ### 📸 Automate Screenshot Capture & Text Extraction\nThis first part of the workflow begins by monitoring a Google Sheet for new URLs... |
| Sticky Note1                           | Sticky Note                 | Documentation note for second block               | None                                   | None                                           | ### 🛍️ Extract and Store Product Listings with GPT-4o\nAfter saving the screenshot, GPT-4o receives the extracted text... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger node**  
   - Name: `Trigger on New URL in Sheet`  
   - Set event to `rowAdded` with polling every minute.  
   - Select your Google Sheets document (e.g., “Competitors”) and worksheet (e.g., “Sheet1”).  
   - Authenticate using OAuth2 credentials with Google Sheets API access.

2. **Add HTTP Request node for screenshot capture**  
   - Name: `Take Full-Page Screenshot using Dumpling AI`  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/screenshot`  
   - Body (raw JSON):  
     ```json
     {
       "url": "{{ $json.URL }}",
       "fullPage": true
     }
     ```  
   - Set content type to JSON and send body as JSON.  
   - Use HTTP Header Auth credentials with your Dumpling AI API key.

3. **Add HTTP Request node for text extraction**  
   - Name: `Extract All Visible Data from Screenshot (Dumpling AI)`  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/extract-image`  
   - Body (raw JSON):  
     ```json
     {
       "inputMethod": "url",
       "images": ["{{ $json.screenshotUrl }}"],
       "prompt": "You are a detail-oriented assistant. Analyze the screenshot provided and extract all visible information in a clear and organized way. Include all text content exactly as it appears, labels, buttons, icons, or form field names, any numbers, figures, or timestamps, section headers or groupings, notes or messages if any, and table data or list items if present. Present the result in a neatly structured format using plain text. Do not summarize, interpret, or skip anything. Maintain the original order and layout as much as possible for clarity."
     }
     ```  
   - Use same HTTP header auth credentials as previous.  
   - Set content type to JSON and send body as JSON.

4. **Add HTTP Request node to download the screenshot file**  
   - Name: `Download Screenshot File`  
   - Method: GET  
   - URL: `={{ $('Take Full-Page Screenshot using Dumpling AI').item.json.screenshotUrl }}`  
   - No authentication necessary (assuming public URL).  
   - Output should be binary data.

5. **Add Google Drive node to upload the screenshot**  
   - Name: `Save Screenshot to Drive Folder`  
   - Operation: Upload file  
   - Drive: My Drive  
   - Folder: Select or specify the folder ID (e.g., “Webpage-Screenshoot” folder)  
   - Input: binary data from previous node.  
   - Authenticate with Google Drive OAuth2 credentials.

6. **Add Google Sheets node to log screenshot URLs**  
   - Name: `Log Screenshot URL to Spreadsheet`  
   - Operation: Append or Update  
   - Document: Same “Competitors” Google Sheet  
   - Sheet: “Sheet1”  
   - Map columns:  
     - URL: `={{ $('Trigger on New URL in Sheet').item.json.URL }}`  
     - screenshot URL: `={{ $('Take Full-Page Screenshot using Dumpling AI').item.json.screenshotUrl }}`  
   - Authenticate with Google Sheets OAuth2 credentials.

7. **Add OpenAI node for product info extraction**  
   - Name: `Extract Product Info from Screenshot Text with GPT-4o`  
   - Model: `gpt-4o` (ensure your OpenAI credentials support this model)  
   - Message system prompt:  
     ```
     Extract only the product-related information from the text. For each product, return a JSON object with fields: "name", "ratings", "purchased", "price", "deal", "delivery", "buyingOptions", "colorOptions", and "badges". Include only available fields. Skip fields not mentioned. Ignore unrelated content. Return JSON with a top-level "products" array.  
     Example JSON output: {...} (as in original prompt)
     ```  
   - User message:  
     ```
     Here is the text: {{ $('Extract All Visible Data from Screenshot (Dumpling AI)').item.json.results }}
     ```  
   - Set Output to JSON format.  
   - Authenticate with OpenAI API credentials.

8. **Add Split Out node**  
   - Name: `Split Each Product into Individual Record`  
   - Field to split: `message.content.products` (path to products array in JSON output).

9. **Add Google Sheets node to save product data**  
   - Name: `Save Products info to Google Sheet`  
   - Operation: Append  
   - Document: “Competitors” Google Sheet  
   - Sheet: “Sheet2”  
   - Map columns:  
     - Product Name → `={{ $json.name }}`  
     - price → `={{ $json.price }}`  
     - purchased → `={{ $json.purchased }}`  
     - ratings → `={{ $json.ratings }}`  
     - deal → `={{ $json.deal }}`  
     - buyingOptions → `={{ $json.buyingOptions }}`  
   - Authenticate with Google Sheets OAuth2 credentials.

10. **Connect nodes in the following order:**  
    - Trigger on New URL in Sheet → Take Full-Page Screenshot using Dumpling AI → Extract All Visible Data from Screenshot (Dumpling AI) → Download Screenshot File → Save Screenshot to Drive Folder → Log Screenshot URL to Spreadsheet → Extract Product Info from Screenshot Text with GPT-4o → Split Each Product into Individual Record → Save Products info to Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses Dumpling AI for both screenshot capturing and detailed UI/text extraction, leveraging its specialized APIs.        | https://app.dumplingai.com/api/v1/screenshot and https://app.dumplingai.com/api/v1/extract-image                                          |
| GPT-4o is used to transform raw extracted text into structured JSON product data, enabling downstream data analytics or reporting.  | Requires OpenAI API access with GPT-4o model enabled                                                                                      |
| Google Sheets and Google Drive are used for input URLs, logging, and storage, requiring configured OAuth2 credentials for access.   | Make sure API quotas and permissions are set appropriately for reliable operation.                                                        |
| The structured prompt for GPT-4o is detailed to avoid hallucinations and ensure output adheres strictly to the requested JSON format. | The prompt includes a sample JSON structure to guide GPT's response.                                                                      |
| Sticky notes within the workflow visually document the two major blocks for user clarity and maintenance.                            | See nodes “Sticky Note” and “Sticky Note1” in the workflow JSON above.                                                                    |

---

This documentation enables a clear understanding of each component, how data flows through the workflow, and how to rebuild or troubleshoot it effectively.