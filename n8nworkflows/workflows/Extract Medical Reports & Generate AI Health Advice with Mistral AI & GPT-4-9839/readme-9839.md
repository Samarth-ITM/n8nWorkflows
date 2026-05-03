Extract Medical Reports & Generate AI Health Advice with Mistral AI & GPT-4

https://n8nworkflows.xyz/workflows/extract-medical-reports---generate-ai-health-advice-with-mistral-ai---gpt-4-9839


# Extract Medical Reports & Generate AI Health Advice with Mistral AI & GPT-4

### 1. Workflow Overview

This workflow automates the extraction of medical test results from diagnostic reports uploaded to Google Drive, and generates personalized AI-based health advice for out-of-range lab results using Mistral AI and GPT-4. It is designed for healthcare providers, diagnostic centers, or wellness platforms seeking to streamline medical data ingestion and deliver actionable patient recommendations.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Watches a specific Google Drive folder for new medical reports in PDF or image format and downloads the file for processing.
- **1.2 Document Type Detection & OCR Extraction:** Determines whether the file is a PDF or an image and uses Mistral AI to extract structured text accordingly.
- **1.3 Text Aggregation & AI Extraction:** Combines multi-page extracted text and employs GPT-4 to parse and extract detailed, structured medical test data as JSON.
- **1.4 Data Transformation & Validation:** Parses AI output into individual test result rows, standardizes data fields, and cleans reference ranges.
- **1.5 Out-of-Range Detection & Personalized Advice Generation:** Identifies abnormal test results by comparing values against reference intervals and uses GPT-4 to generate personalized dietary, lifestyle, and exercise advice.
- **1.6 Data Merging & Storage:** Merges AI-generated advice back into the medical test records and saves both all test results and flagged abnormal results with advice to Google Sheets for review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Monitors a designated Google Drive folder for newly uploaded medical reports (PDF or image files) and downloads the detected file for processing.

**Nodes Involved:**  
- Upload pdf/jpeg file to Google drive  
- Download file

**Node Details:**

- **Upload pdf/jpeg file to Google drive**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive folder for new file creation events, polling every minute.  
  - Configuration: Folder ID is set via a workflow variable `GOOGLE_DRIVE_FOLDER_ID`.  
  - Input: Triggered by new file upload.  
  - Output: Passes file metadata including file ID and MIME type.  
  - Edge Cases: Missing or incorrect folder ID may prevent triggering; permission issues with folder access may cause auth errors.

- **Download file**  
  - Type: Google Drive Node  
  - Role: Downloads the actual file content based on file ID from the trigger.  
  - Configuration: Uses dynamic expression `={{ $json.id }}` from trigger to identify the file to download.  
  - Input: Receives file metadata from trigger node.  
  - Output: Provides file binary content for further processing.  
  - Edge Cases: Download failure due to network issues or insufficient permissions.

---

#### 2.2 Document Type Detection & OCR Extraction

**Overview:**  
Determines whether the downloaded file is a PDF or an image and routes it to the appropriate Mistral AI node to extract text and structure using OCR.

**Nodes Involved:**  
- Check if PDF or Image  
- Extract from PDF  
- Extract from Image  
- Combine page Markdown

**Node Details:**

- **Check if PDF or Image**  
  - Type: IF Node  
  - Role: Evaluates the MIME type of the downloaded file to decide extraction route.  
  - Configuration: Compares MIME type to `application/pdf`.  
  - Input: Receives file metadata and content from Download file.  
  - Output: Routes to either Extract from PDF (if PDF) or Extract from Image (otherwise).  
  - Edge Cases: Unrecognized MIME types may need additional handling or cause routing failure.

- **Extract from PDF**  
  - Type: Mistral AI Node  
  - Role: Uses Mistral AI OCR capabilities optimized for PDFs to extract text and document structure.  
  - Configuration: Document type set to default (PDF).  
  - Input: Receives binary PDF file.  
  - Output: Produces JSON including page-wise Markdown text.  
  - Edge Cases: OCR accuracy depends on PDF quality; large PDFs might cause timeouts.

- **Extract from Image**  
  - Type: Mistral AI Node  
  - Role: Uses Mistral AI vision OCR to extract text from image files (JPEG, PNG).  
  - Configuration: Document type set to "image_url".  
  - Input: Receives binary image file.  
  - Output: Produces JSON including page-wise Markdown text.  
  - Edge Cases: Poor image quality may reduce OCR accuracy.

- **Combine page Markdown**  
  - Type: Code Node  
  - Role: Merges Markdown text from all pages of the extracted document into a single combined Markdown string.  
  - Key Code Logic: Iterates over `pages` array, concatenates each page’s `markdown` field separated by newlines.  
  - Input: Receives JSON with `pages` array.  
  - Output: Single JSON item with combined Markdown text.  
  - Edge Cases: Missing or empty pages array could produce empty output.

---

#### 2.3 Text Aggregation & AI Extraction

**Overview:**  
Feeds the combined Markdown text to GPT-4 to extract all individual medical test results and patient metadata in a granular JSON array.

**Nodes Involved:**  
- Extract Medical Data (AI)

**Node Details:**

- **Extract Medical Data (AI)**  
  - Type: OpenAI Node (LangChain)  
  - Role: Parses combined text using GPT-4 with a detailed prompt to extract all measurable test results as a JSON array, including patient and test details.  
  - Configuration:  
    - Model: GPT-4 (`gpt-4o`)  
    - Prompt: System message defines role as expert medical data extraction assistant; user message instructs detailed JSON output format with fields such as diagnostic centre, patient name, age, gender, registration date, sample type, test name, result, unit, reference range.  
  - Input: Combined Markdown text from previous node.  
  - Output: JSON array of test results/parameters.  
  - Edge Cases: GPT output might include formatting or partial JSON; prompt adherence issues could cause parsing errors.

---

#### 2.4 Data Transformation & Validation

**Overview:**  
Transforms the AI-generated JSON array into individual records (one item per test result), cleans the biological reference interval field for consistency, and prepares data for downstream processing.

**Nodes Involved:**  
- Parse AI output to one-per-row  
- Save All Test Results

**Node Details:**

- **Parse AI output to one-per-row**  
  - Type: Code Node  
  - Role: Extracts the AI JSON string from the GPT output, cleans markdown code fences, parses JSON, validates it is an array, and maps each element to a separate n8n item with normalized fields.  
  - Key Code Logic:  
    - Strips code fences (```), parses JSON, returns error JSON if parsing fails or data is not an array.  
    - Cleans `reference_range` field to retain only numbers, dots, hyphens, and spaces for standardization.  
    - Maps fields to standardized property names compatible with Google Sheets.  
  - Input: GPT-4 raw response.  
  - Output: Multiple n8n items, each representing one test result.  
  - Edge Cases: Parsing errors, malformed JSON, missing fields replaced by "N/A".

- **Save All Test Results**  
  - Type: Google Sheets Node  
  - Role: Appends all extracted test results to the "All Values" sheet for comprehensive record-keeping.  
  - Configuration: Defines columns for patient info, test name, result, unit, sample type, and reference interval.  
  - Input: Individual test result items.  
  - Output: None (data stored in Google Sheets).  
  - Edge Cases: Google Sheets API quota limits, invalid sheet name or missing permissions.

---

#### 2.5 Out-of-Range Detection & Personalized Advice Generation

**Overview:**  
Detects test results that are outside the provided biological reference intervals and generates AI-based personalized dietary, lifestyle, and exercise advice for each abnormal result.

**Nodes Involved:**  
- Out-of-Range Detection & Advice Fields  
- General Health Advice(AI)  
- Merge AI Response Back

**Node Details:**

- **Out-of-Range Detection & Advice Fields**  
  - Type: Code Node  
  - Role: Compares each test result numeric value against its reference range to identify abnormal values. Initializes empty advice fields for each item. Filters out-of-range results for further AI processing.  
  - Key Code Logic:  
    - Parses numeric values from result and reference ranges using regex.  
    - Supports ranges (e.g., "0.3 - 1.2"), less-than ("<2"), and greater-than (">5") formats.  
    - Returns only out-of-range items for AI advice generation.  
  - Input: Parsed test result items.  
  - Output: Out-of-range test result items with blank advice fields.  
  - Edge Cases: Non-numeric or missing reference ranges, malformed reference intervals, false negatives.

- **General Health Advice(AI)**  
  - Type: OpenAI Node (LangChain)  
  - Role: Uses GPT-4 to generate actionable personalized health advice in three categories: dietary, lifestyle, and exercise based on each abnormal test result and patient demographics.  
  - Configuration:  
    - Model: GPT-4 (`gpt-4o`)  
    - Prompt: System message defines role as physician-reviewed expert assistant; user message provides patient and test details dynamically with placeholders and instructs JSON-only output with advice fields.  
  - Input: Out-of-range test result items.  
  - Output: AI-generated JSON advice per test result.  
  - Edge Cases: GPT output parsing failures, incomplete advice generation.

- **Merge AI Response Back**  
  - Type: Code Node  
  - Role: Merges the AI-generated advice fields back into the full set of test results by matching array indices, updating each item with dietary, lifestyle, and exercise advice.  
  - Key Code Logic:  
    - Parses AI response JSON, removes code fences, extracts advice fields.  
    - Updates the corresponding item’s JSON with advice strings.  
  - Input: AI advice items and original test result items.  
  - Output: Complete test result items with appended health advice.  
  - Edge Cases: Mismatched array lengths, parsing errors, missing advice fields.

---

#### 2.6 Data Merging & Storage

**Overview:**  
Stores the enriched test result data, including personalized advice for abnormal results, into dedicated Google Sheets tabs for review and record management.

**Nodes Involved:**  
- Save Out-of-Range Results

**Node Details:**

- **Save Out-of-Range Results**  
  - Type: Google Sheets Node  
  - Role: Appends abnormal test results along with generated health advice to the "Out of Range Values" sheet for prioritized clinical review.  
  - Configuration: Defines columns for patient info, test details, result, unit, reference range, and all three advice fields.  
  - Input: Complete test result items with AI advice.  
  - Output: None (data stored in Google Sheets).  
  - Edge Cases: Google Sheets API limits, sheet access permissions, data formatting issues.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                                      | Input Node(s)                           | Output Node(s)                            | Sticky Note                                                                                                    |
|-----------------------------------|----------------------------|-----------------------------------------------------|---------------------------------------|------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Upload pdf/jpeg file to Google drive | Google Drive Trigger       | Watches Google Drive folder for new medical reports | None (trigger)                        | Download file                            | ## Google Drive Trigger<br>Monitors a specific Google Drive folder for newly uploaded medical reports (PDF/image files). Triggers the workflow when a new file is detected (checks every minute). |
| Download file                     | Google Drive Node           | Downloads detected file                              | Upload pdf/jpeg file to Google drive | Check if PDF or Image                    | ## Download File<br>Downloads the detected medical report file from Google Drive for processing.               |
| Check if PDF or Image             | IF Node                    | Routes based on file MIME type                       | Download file                        | Extract from PDF, Extract from Image     | ## Check if PDF/Image<br>Determines whether the uploaded file is a PDF or an image, routing to the appropriate extraction node. |
| Extract from PDF                  | Mistral AI Node             | OCR extraction from PDF                              | Check if PDF or Image                | Combine page Markdown                    | ## Extract from PDF<br>Uses Mistral AI to extract text and structure from PDF medical reports using OCR capabilities. |
| Extract from Image                | Mistral AI Node             | OCR extraction from image                            | Check if PDF or Image                | Combine page Markdown                    | ## Extract from Image<br>Uses Mistral AI to extract text from image-based medical reports (JPG, PNG, etc.) using vision capabilities. |
| Combine page Markdown             | Code Node                  | Merges text from all pages into single Markdown     | Extract from PDF, Extract from Image | Extract Medical Data (AI)                | ## Extract Medical Data (AI)<br>Uses GPT-4 to intelligently extract structured data from the medical report.<br>## Combine page Markdown<br>Merges markdown text from all pages of multi-page reports.<br>## Parse AI output to one-per-row<br>Transforms AI JSON array to individual records. |
| Extract Medical Data (AI)         | OpenAI Node (GPT-4)         | Extracts structured medical test data from text    | Combine page Markdown                | Parse AI output to one-per-row           |                                                                                                               |
| Parse AI output to one-per-row   | Code Node                  | Parses AI JSON array into individual test result items | Extract Medical Data (AI)             | Out-of-Range Detection & Advice Fields, Save All Test Results |                                                                                                               |
| Save All Test Results            | Google Sheets Node          | Stores all extracted test results                   | Parse AI output to one-per-row       | None                                     | ## Save All Test Results<br>Saves every extracted test result to the "All Values" sheet in Google Sheets.     |
| Out-of-Range Detection & Advice Fields | Code Node                  | Detects abnormal results and initializes advice fields | Parse AI output to one-per-row       | General Health Advice(AI)                 | ## Generate Health Advice (AI)<br>Uses GPT-4 to generate personalized recommendations.<br>## Out-of-Range Detection & Advice Fields<br>Detects abnormal values.<br>## Merge AI Response Back<br>Combines AI advice with test data. |
| General Health Advice(AI)        | OpenAI Node (GPT-4)         | Generates personalized health advice for abnormal results | Out-of-Range Detection & Advice Fields | Merge AI Response Back                   |                                                                                                               |
| Merge AI Response Back           | Code Node                  | Merges AI advice back into all test result items   | General Health Advice(AI)            | Save Out-of-Range Results                 |                                                                                                               |
| Save Out-of-Range Results        | Google Sheets Node          | Stores abnormal test results with advice            | Merge AI Response Back               | None                                     | ## Save Out-of-Range Results<br>Saves abnormal results with personalized advice to the "Out of Range Values" sheet. |
| Sticky Note                     | Sticky Note                 | Workflow explanation and guidance                    | None                                | None                                     | ## Google Drive Trigger<br>## Upload file (Pdf/Image)<br>Monitors a specific Google Drive folder for newly uploaded medical reports (PDF/image files). Triggers the workflow when a new file is detected (checks every minute). |
| Sticky Note1                    | Sticky Note                 | Guidance on Download File node                        | None                                | None                                     | ## Download File<br>Downloads the detected medical report file from Google Drive for processing.               |
| Sticky Note2                    | Sticky Note                 | Guidance on Check if PDF/Image node                   | None                                | None                                     | ## Check if PDF/Image<br>Determines whether the uploaded file is a PDF or an image, routing to the appropriate extraction node. |
| Sticky Note3                    | Sticky Note                 | Guidance on Extract from PDF and Image nodes          | None                                | None                                     | ## Extract from PDF and Image<br>Mistral AI OCR extraction of PDF or image medical reports.                   |
| Sticky Note4                    | Sticky Note                 | Workflow overview and instructions                    | None                                | None                                     | ## Try It Out!<br>Stepwise explanation of workflow usage and setup requirements.                             |
| Sticky Note5                    | Sticky Note                 | Guidance on saving results nodes                      | None                                | None                                     | ## Save Out-of-Range Results<br>Saves abnormal test results and advice.<br>## Save All Test Results<br>Saves all extracted data. |
| Sticky Note6                    | Sticky Note                 | Guidance on Extract Medical Data and parsing nodes   | None                                | None                                     | ## Extract Medical Data (AI)<br>GPT-4 extraction and parsing explanation.                                    |
| Sticky Note10                   | Sticky Note                 | Guidance on advice generation and merging nodes      | None                                | None                                     | ## Generate Health Advice (AI)<br>GPT-4 advice generation.<br>## Out-of-Range Detection & Advice Fields<br>Abnormality detection.<br>## Merge AI Response Back<br>Combines advice with data. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure to watch a specific folder by its ID (`GOOGLE_DRIVE_FOLDER_ID` variable or direct folder ID).  
   - Set event to `fileCreated` and polling interval to every minute.

2. **Add Google Drive Download File Node**  
   - Type: Google Drive node  
   - Operation: `download`  
   - Set File ID to expression `={{ $json.id }}` from trigger node output.  
   - Connect Google Drive Trigger output to this node.

3. **Add IF Node to Check File Type**  
   - Type: IF Node  
   - Condition: Check if `{{ $('Download file').item.json.mimeType }}` equals `"application/pdf"`.  
   - Connect Download File node output to IF node input.

4. **Add Mistral AI Node for PDF Extraction**  
   - Type: Mistral AI node  
   - Set Document Type to PDF (default).  
   - Connect IF node’s "true" output (PDF path) here.

5. **Add Mistral AI Node for Image Extraction**  
   - Type: Mistral AI node  
   - Set Document Type to `"image_url"`.  
   - Connect IF node’s "false" output (image path) here.

6. **Add Code Node to Combine Page Markdown**  
   - Type: Code Node  
   - Paste code to iterate over `pages` array and concatenate `markdown` fields with newlines.  
   - Connect both Mistral AI nodes’ outputs to this node.

7. **Add OpenAI Node to Extract Medical Data**  
   - Type: OpenAI Node (LangChain) with GPT-4 model (`gpt-4o`)  
   - System message: Define expert medical data extraction role.  
   - User message: Provide detailed prompt to extract all test results as a JSON array with required fields (diagnostic centre, patient name, age, gender, registration date, sample type, test name, result, unit, reference range).  
   - Input: Combined Markdown from previous node.

8. **Add Code Node to Parse AI Output to One-per-Row**  
   - Type: Code Node  
   - Paste code to remove markdown fences, parse JSON, validate array, clean reference ranges, and output individual items.  
   - Connect OpenAI node output here.

9. **Add Google Sheets Node to Save All Test Results**  
   - Type: Google Sheets node  
   - Operation: Append  
   - Sheet name: `"All Values"`  
   - Map columns: Age, Name, Unit, Result, Test Name, Sample Type, Biological reference Interval.  
   - Connect parse output node to this node.

10. **Add Code Node for Out-of-Range Detection & Advice Fields**  
    - Type: Code Node  
    - Paste code that compares each result with reference range, identifies out-of-range items, initializes advice fields, and outputs filtered items.  
    - Connect parse output node to this node.

11. **Add OpenAI Node for General Health Advice**  
    - Type: OpenAI Node (LangChain) with GPT-4 model (`gpt-4o`)  
    - System message: Define role as physician-reviewed expert assistant.  
    - User message: Provide prompt with placeholders for patient data and test result requesting JSON output with dietary, lifestyle, and exercise advice.  
    - Connect Out-of-Range Detection node output.

12. **Add Code Node to Merge AI Response Back**  
    - Type: Code Node  
    - Paste code to merge AI advice JSON fields back into full test result items by index.  
    - Connect General Health Advice node output.

13. **Add Google Sheets Node to Save Out-of-Range Results**  
    - Type: Google Sheets node  
    - Operation: Append  
    - Sheet name: `"Out of Range Values"`  
    - Map columns: Diagnostic Centre, Name, Age, Gender, Registered on, Sample Type, Test Name, Result, Unit, Biological reference Interval, Dietary advice, Lifestyle advice, Exercise advice.  
    - Connect Merge AI Response Back node output.

14. **Connect Nodes Appropriately**  
    - Trigger → Download file → Check if PDF or Image → Extract from PDF/Image → Combine page Markdown → Extract Medical Data (AI) → Parse AI output to one-per-row →  
      - Save All Test Results  
      - Out-of-Range Detection & Advice Fields → General Health Advice (AI) → Merge AI Response Back → Save Out-of-Range Results

15. **Configure Credentials**  
    - Google Drive OAuth2 credentials for trigger and download nodes.  
    - Mistral AI API key for OCR nodes.  
    - OpenAI API key with GPT-4 access for AI extraction and advice nodes.  
    - Google Sheets OAuth2 credentials for saving nodes.

16. **Set Workflow Variables**  
    - `GOOGLE_DRIVE_FOLDER_ID` (or hardcode folder ID) to monitor for incoming files.

17. **Test Workflow**  
    - Upload sample PDF/image medical reports to the monitored Google Drive folder.  
    - Verify data extraction in Google Sheets tabs `"All Values"` and `"Out of Range Values"` with AI-generated advice.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow uses Mistral AI for OCR extraction of text from both PDFs and images, preserving document structure for better AI parsing.                                                                                                                                                                                                          | Mistral AI OCR and vision node usage                                                                |
| GPT-4 is leveraged twice: first for detailed structured extraction of medical test results, then for personalized health advice generation based on abnormal values.                                                                                                                                                                              | OpenAI GPT-4 nodes with LangChain integration                                                      |
| Google Sheets contains two tabs: "All Values" for all extracted test results, and "Out of Range Values" for flagged abnormal tests with personalized advice.                                                                                                                                                                                      | Sheet setup requirement                                                                              |
| Important to ensure all API credentials are correctly configured for Google Drive, Mistral AI, OpenAI, and Google Sheets with OAuth2 authentication.                                                                                                                                                                                             | Credential configuration instructions                                                               |
| The workflow expects consistent, well-formed medical reports; poor scan/image quality or unusual report formats may reduce extraction accuracy.                                                                                                                                                                                                  | OCR and AI extraction limitations                                                                   |
| For support and advanced configuration, visit the n8n community forum or official documentation.                                                                                                                                                                                                                                                  | https://community.n8n.io/                                                                            |
| This workflow was designed by a senior technical analyst specializing in n8n workflows and AI integration.                                                                                                                                                                                                                                       | Project credit                                                                                       |


---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. The process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.