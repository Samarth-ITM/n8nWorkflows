Process Documents with OCR, Analytics & Google Drive using PDF Vector

https://n8nworkflows.xyz/workflows/process-documents-with-ocr--analytics---google-drive-using-pdf-vector-8505


# Process Documents with OCR, Analytics & Google Drive using PDF Vector

### 1. Workflow Overview

This workflow automates the processing of documents stored in a specified Google Drive folder. It performs Optical Character Recognition (OCR) and analytics on PDF, Word, and image files using PDF Vector technology and compiles detailed real-time analytics reports on processing performance and quality. The workflow is designed for batch processing and continuous monitoring of document processing metrics for operational insights.

Logical blocks included:

- **1.1 Input Reception:** Triggering and fetching documents from Google Drive.
- **1.2 Validation & Prioritization:** Filtering and prioritizing files based on type and size.
- **1.3 Batch Processing:** Splitting files into manageable batches and preparing individual items.
- **1.4 Document Processing:** Processing each document or image via PDF Vector node.
- **1.5 Result Tracking:** Analyzing processing outcomes and quality metrics.
- **1.6 Analytics Reporting:** Aggregating results into a comprehensive analytics report.
- **1.7 Informational Notes:** Sticky notes providing contextual information on analytics and metrics.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts processing manually via trigger and lists documents in a specified Google Drive folder for subsequent handling.

**Nodes Involved:**  
- Manual Trigger  
- List Documents

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Initiates the workflow manually for batch processing.  
  - Configuration: No parameters; user manually starts the workflow.  
  - Inputs: None  
  - Outputs: Connected to List Documents node  
  - Edge Cases: None expected; manual start avoids unintended execution.

- **List Documents**  
  - Type: Google Drive node  
  - Role: Lists up to 100 files in a given Google Drive folder (requires folder ID replacement).  
  - Configuration:  
    - Operation: list  
    - Query: Files in folder `'FOLDER_ID_HERE'` that are not trashed  
    - Fields retrieved: id, name, mimeType, size, webViewLink, createdTime  
  - Inputs: Manual Trigger  
  - Outputs: Validated & queued files node  
  - Credentials: Requires Google Drive OAuth2 credentials  
  - Edge Cases:  
    - Folder ID must be replaced with a valid folder.  
    - API quota or permission errors possible.  
    - Empty folder returns no files.

---

#### 2.2 Validation & Prioritization

**Overview:**  
Validates file types and sizes, categorizes files into valid or invalid queues, and prioritizes processing based on file size.

**Nodes Involved:**  
- Validate & Queue Files

**Node Details:**

- **Validate & Queue Files**  
  - Type: Code (JavaScript) node  
  - Role: Applies business logic to determine valid files for processing and assign priorities.  
  - Configuration:  
    - Supported formats: PDF, Word (doc/docx), and common images (jpeg, png, gif).  
    - Size limit: 50MB max; files above are invalidated.  
    - Assigns priority:  
      - High if <5MB  
      - Medium if between 5MB and 20MB  
      - Low otherwise  
    - Calculates estimated credits based on size for PDFs (2 credits per MB), flat 1 credit otherwise.  
    - Outputs: An object with arrays of valid and invalid files, plus processing stats.  
  - Inputs: List Documents  
  - Outputs: Process in Batches  
  - Edge Cases:  
    - Files with unsupported mime types or oversized files go to invalid queue.  
    - Potential for incorrect MIME type detection.  
    - Large file size numbers might cause float precision issues.

---

#### 2.3 Batch Processing

**Overview:**  
Splits the valid files into batches of 5 for manageable processing, then prepares individual file items for detailed processing.

**Nodes Involved:**  
- Process in Batches  
- Split Out Files  
- Split Items

**Node Details:**

- **Process in Batches**  
  - Type: SplitInBatches node  
  - Role: Manages batch size for downstream processing to prevent overload.  
  - Configuration: Batch size set to 5 files per batch.  
  - Inputs: Validate & Queue Files  
  - Outputs: Split Out Files and Generate Analytics Report (for analytics after batching)  
  - Edge Cases:  
    - Batch size too large may cause timeouts or API rate limiting.

- **Split Out Files**  
  - Type: Set node  
  - Role: Converts the batch object into a single attribute (`processingBatch`) for splitting.  
  - Configuration: Assigns entire batch JSON to a single field.  
  - Inputs: Process in Batches  
  - Outputs: Split Items  
  - Edge Cases: None significant.

- **Split Items**  
  - Type: SplitOut node  
  - Role: Splits the batch into individual file items for per-file processing.  
  - Configuration: Field to split: `processingBatch.valid` (array of valid files).  
  - Inputs: Split Out Files  
  - Outputs: PDF Vector - Process Document/Image  
  - Edge Cases: Empty batch arrays produce no output items.

---

#### 2.4 Document Processing

**Overview:**  
Processes each individual document or image using PDF Vector’s OCR and NLP capabilities, supporting automatic LLM usage.

**Nodes Involved:**  
- PDF Vector - Process Document/Image

**Node Details:**

- **PDF Vector - Process Document/Image**  
  - Type: PDF Vector node (custom integration)  
  - Role: Parses documents/images from URL, performs OCR, and optionally uses Large Language Models (LLMs).  
  - Configuration:  
    - Resource: document  
    - Operation: parse  
    - Input type: URL (uses Google Drive webViewLink)  
    - LLM usage: auto (automatic decision to use LLM)  
  - Inputs: Split Items  
  - Outputs: Track Processing Results  
  - Continue On Fail: true (workflow continues even if processing fails)  
  - Edge Cases:  
    - Network issues or invalid URLs cause errors.  
    - OCR failures or unsupported document contents.  
    - LLM API rate limits or authentication errors.

---

#### 2.5 Result Tracking

**Overview:**  
Analyzes each processed file’s results, evaluating success, processing time, credits used, content quality, and error details.

**Nodes Involved:**  
- Track Processing Results

**Node Details:**

- **Track Processing Results**  
  - Type: Code node  
  - Role: Extracts processing metadata and performs quality checks on the output content.  
  - Configuration:  
    - Measures processing time based on execution timestamps  
    - Determines success based on absence of errors  
    - Calculates quality checks: content presence, reasonable word count, encoding correctness, credit efficiency  
    - Computes overall quality score (percentage)  
    - Returns a detailed summary object per file.  
  - Inputs: PDF Vector - Process Document/Image  
  - Outputs: Collect Batch Results  
  - Edge Cases:  
    - Missing timestamps or content can skew metrics.  
    - Edge cases for files with minimal content or encoding anomalies.

---

#### 2.6 Analytics Reporting

**Overview:**  
Aggregates all batch results, computes comprehensive metrics, success rates, error counts, performance highlights, and generates a formatted markdown report with actionable recommendations.

**Nodes Involved:**  
- Collect Batch Results  
- Generate Analytics Report

**Node Details:**

- **Collect Batch Results**  
  - Type: Aggregate node  
  - Role: Aggregates all individual processed file results into a single dataset for reporting.  
  - Configuration: Aggregate all item data together.  
  - Inputs: Track Processing Results  
  - Outputs: Generate Analytics Report  
  - Edge Cases: Empty input produces empty aggregate.

- **Generate Analytics Report**  
  - Type: Code node  
  - Role: Processes aggregated results and initial validation stats to produce detailed analytics and a human-readable report.  
  - Configuration:  
    - Calculates overview stats (files processed, success, failure, time, credits, quality scores)  
    - Breaks down by file type (pdf, word, image) with averages and success rates  
    - Tracks error types and counts  
    - Identifies fastest/slowest and most/least credit-efficient files  
    - Generates markdown report with recommendations based on thresholds (e.g., success rate < 90%)  
  - Inputs: Collect Batch Results and initial validation stats (from Validate & Queue Files)  
  - Outputs: End of processing data with analytics and report  
  - Edge Cases: Divisions by zero, empty datasets, unexpected error messages.

---

#### 2.7 Informational Notes (Sticky Notes)

**Overview:**  
Provides contextual information about the workflow’s analytics capabilities, tracked metrics, and output destinations.

**Nodes Involved:**  
- Analytics Overview  
- Metrics Tracked  
- Dashboard Output

**Node Details:**

- **Analytics Overview**  
  - Type: Sticky Note  
  - Content: Describes real-time analytics features such as tracking workflows, calculating KPIs every 30 minutes, monitoring success/failure, analyzing trends, and updating dashboards automatically.

- **Metrics Tracked**  
  - Type: Sticky Note  
  - Content: Lists key metrics tracked (documents/hour, processing time, error rates, API usage, cost) over a 30-day rolling window.

- **Dashboard Output**  
  - Type: Sticky Note  
  - Content: Lists output channels for analytics (Google Sheets, Tableau, Power BI, Slack alerts) with real-time update emphasis.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                      | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                 |
|-------------------------------|-------------------------|------------------------------------|--------------------------|------------------------------|---------------------------------------------------------------------------------------------|
| Manual Trigger                | Manual Trigger          | Initiates workflow manually        | None                     | List Documents               | Start batch processing                                                                     |
| List Documents               | Google Drive            | Lists files in specified folder    | Manual Trigger           | Validate & Queue Files       | Replace FOLDER_ID_HERE with your Google Drive folder ID                                    |
| Validate & Queue Files       | Code                    | Validates files and prioritizes    | List Documents           | Process in Batches           | Validate and prioritize files                                                              |
| Process in Batches           | SplitInBatches          | Splits files into batches of 5     | Validate & Queue Files   | Split Out Files, Generate Analytics Report | Process 5 files at a time                                                                   |
| Split Out Files              | Set                     | Prepares batch object for splitting| Process in Batches       | Split Items                  | Prepare individual files                                                                   |
| Split Items                 | SplitOut                | Splits batch into individual files | Split Out Files          | PDF Vector - Process Document/Image |                                                                                             |
| PDF Vector - Process Document/Image | PDF Vector              | Processes document/image OCR & NLP | Split Items              | Track Processing Results     | Process document or image                                                                   |
| Track Processing Results     | Code                    | Analyzes processing result quality | PDF Vector - Process Document/Image | Collect Batch Results         | Analyze results                                                                           |
| Collect Batch Results        | Aggregate                | Aggregates batch results           | Track Processing Results | Generate Analytics Report    | Aggregate batch results                                                                    |
| Generate Analytics Report    | Code                    | Generates detailed analytics report| Collect Batch Results     | None                        | Create analytics dashboard                                                                 |
| Analytics Overview           | Sticky Note              | Overview of analytics capabilities | None                     | None                        | ## 📊 Real-Time Analytics\n\nDocument processing metrics:\n• **Tracks** all workflows in database\n• **Calculates** KPIs every 30 minutes\n• **Monitors** success/failure rates\n• **Analyzes** trends & patterns\n• **Updates** dashboards automatically |
| Metrics Tracked             | Sticky Note              | Lists key tracked metrics          | None                     | None                        | ## 📈 Key Metrics\n\n**Tracking:**\n• Documents/hour\n• Processing time\n• Error rates\n• API usage\n• Cost analysis\n\n💡 30-day rolling window |
| Dashboard Output            | Sticky Note              | Lists analytics output destinations | None                     | None                        | ## 📊 Visualizations\n\n**Outputs to:**\n• Google Sheets\n• Tableau\n• Power BI\n• Slack alerts\n\n✨ Real-time updates! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - Position on canvas: start of the flow

2. **Create Google Drive Node (List Documents)**  
   - Type: Google Drive  
   - Operation: List  
   - Limit: 100  
   - Fields: id, name, mimeType, size, webViewLink, createdTime  
   - Query: `'FOLDER_ID_HERE' in parents and trashed=false` (replace `FOLDER_ID_HERE` with actual folder ID)  
   - Credentials: Set Google Drive OAuth2 credentials  
   - Connect Manual Trigger → List Documents

3. **Create Code Node (Validate & Queue Files)**  
   - Type: Code  
   - Language: JavaScript  
   - Paste provided validation script (validates file types, size, priority, and estimated credits)  
   - Connect List Documents → Validate & Queue Files

4. **Create SplitInBatches Node (Process in Batches)**  
   - Type: SplitInBatches  
   - Batch Size: 5  
   - Connect Validate & Queue Files → Process in Batches

5. **Create Set Node (Split Out Files)**  
   - Type: Set  
   - Add assignment: `processingBatch` = `={{ $json }}` (assign entire batch object)  
   - Connect Process in Batches → Split Out Files

6. **Create SplitOut Node (Split Items)**  
   - Type: SplitOut  
   - Field To Split Out: `processingBatch.valid`  
   - Connect Split Out Files → Split Items

7. **Create PDF Vector Node (PDF Vector - Process Document/Image)**  
   - Type: PDF Vector (custom node)  
   - Resource: Document  
   - Operation: Parse  
   - Input Type: URL  
   - URL: `={{ $json.webViewLink }}`  
   - Use LLM: Auto  
   - Enable “Continue On Fail”  
   - Connect Split Items → PDF Vector - Process Document/Image  
   - Credentials: Configure API credentials as needed for PDF Vector

8. **Create Code Node (Track Processing Results)**  
   - Type: Code  
   - JavaScript code: Provided result tracking code (calculates success, quality, timing, credits)  
   - Connect PDF Vector - Process Document/Image → Track Processing Results

9. **Create Aggregate Node (Collect Batch Results)**  
   - Type: Aggregate  
   - Operation: Aggregate All Item Data  
   - Connect Track Processing Results → Collect Batch Results

10. **Create Code Node (Generate Analytics Report)**  
    - Type: Code  
    - Paste the provided analytics report generation script  
    - Connect Collect Batch Results → Generate Analytics Report  
    - Also connect Process in Batches → Generate Analytics Report (to pass initial stats)  

11. **Add Sticky Notes for Documentation:**  
    - Create three sticky notes with the provided content:  
      - Analytics Overview (near start)  
      - Metrics Tracked (near bottom left)  
      - Dashboard Output (near bottom right)  

12. **Verify Credential Configurations:**  
    - Google Drive node requires OAuth2 credentials with read permissions on the target folder.  
    - PDF Vector node requires API credentials for OCR and LLM services.  
    - No other external credentials needed.

13. **Test Workflow:**  
    - Manually trigger the workflow.  
    - Confirm files are fetched, validated, processed, and analytics generated.  
    - Monitor for errors in unsupported file types or large files.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Real-time analytics update dashboards every 30 minutes with KPIs, error rates, and trend analysis. | Sticky note "Analytics Overview"                                                                 |
| Key metrics tracked include documents/hour, processing time, error rates, API usage, and cost.      | Sticky note "Metrics Tracked"                                                                     |
| Outputs analytics to Google Sheets, Tableau, Power BI, and Slack alerts with real-time updates.     | Sticky note "Dashboard Output"                                                                    |
| Replace placeholder folder ID in Google Drive node query with your actual Google Drive folder ID.   | Node "List Documents" note                                                                        |
| Batch processing size is set to 5 to balance throughput and rate limits.                             | Node "Process in Batches" note                                                                    |

---

This structured documentation enables a comprehensive understanding of the entire workflow, facilitates reproduction or modification, and highlights critical error handling points and integration dependencies.