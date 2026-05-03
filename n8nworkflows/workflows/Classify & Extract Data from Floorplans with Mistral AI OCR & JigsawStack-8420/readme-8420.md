Classify & Extract Data from Floorplans with Mistral AI OCR & JigsawStack

https://n8nworkflows.xyz/workflows/classify---extract-data-from-floorplans-with-mistral-ai-ocr---jigsawstack-8420


# Classify & Extract Data from Floorplans with Mistral AI OCR & JigsawStack

### 1. Workflow Overview

This workflow is designed to automatically classify and extract data from floorplan files uploaded by users, using AI-powered OCR and classification services (Mistral AI and JigsawStack). It targets use cases in real estate, architecture, and construction, where users submit PDFs or image files of floorplans for automated validation and measurement extraction.

The workflow is logically divided into two main phases:  
- **1.1 Input Reception & Consent Validation:** Handles incoming uploads, checks GDPR consent, and prepares individual files for processing.  
- **1.2 File Type-based Processing & Filtering:** Differentiates PDFs from images, applies file size and page count restrictions, and extracts text from PDFs.  
- **1.3 Heuristic Confidence Scoring:** Uses heuristic rules on extracted PDF text or file metadata to calculate a confidence score indicating the likelihood the file is a floorplan.  
- **1.4 AI Classification:** For PDFs, sends extracted text; for images, sends a temporary public URL to JigsawStack’s classification API to determine if the file is a floorplan.  
- **1.5 Confidence-based Routing & User Response:** Routes files based on confidence thresholds, responding to the user with appropriate messages for rejection, manual review, or acceptance.  
- **1.6 File Storage:** Uploads image files to JigsawStack storage before classification.  
- **1.7 Documentation, Notes & Fail-safes:** Sticky notes document logic, user messaging, and operational constraints such as file size limits and manual review policies.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Consent Validation

- **Overview:** Receives upload requests via webhook and verifies GDPR consent before continuing processing.  
- **Nodes Involved:** `Webhook – Receive Upload`, `Check – GDPR Consent`, `Respond – Consent Required`, `Process – Multiple File Uploads`  

- **Node Details:**  
  - **Webhook – Receive Upload**  
    - Type: Webhook  
    - Role: Entry point for incoming POST requests with floorplan files.  
    - Config: Path `fp-mvp`, accepts POST, basic authentication enabled.  
    - Inputs: External HTTP requests  
    - Outputs: To GDPR consent check  
    - Failures: Auth failure, invalid HTTP method  
  - **Check – GDPR Consent**  
    - Type: IF  
    - Role: Validates that user consent fields `GDPR_check` or `GDPR_check2` are true.  
    - Config: Expression checks JSON body fields for boolean true.  
    - Inputs: Webhook output  
    - Outputs: Branches to process files if consent; respond with consent required otherwise.  
    - Edge Cases: Missing or false consent fields, loose type validation to handle multiple variants.  
  - **Respond – Consent Required**  
    - Type: Respond to Webhook  
    - Role: Sends JSON message asking for consent if missing.  
    - Config: HTML message with GDPR notice and deletion timeframe.  
    - Inputs: From GDPR check (false branch)  
    - Outputs: Ends workflow for that request.  
  - **Process – Multiple File Uploads**  
    - Type: Code  
    - Role: Splits multi-file uploads into individual items preserving binary data.  
    - Config: JS loops through binary properties, outputs one item per file.  
    - Inputs: From GDPR check (true branch)  
    - Outputs: Individual files to file type checking  
    - Edge Cases: Skips items without binary data; preserves original binary keys.

#### 2.2 File Type-based Processing & Filtering

- **Overview:** Identifies whether each file is an image or PDF, extracts text from PDFs, and checks size and page limits.  
- **Nodes Involved:** `Check – File Type (PDF/Image)`, `Upload – JigsawStack (Storage)`, `Extract – PDF Metadata/Text`, `Check – File Size & Pages`, `Respond – File Too Large`  

- **Node Details:**  
  - **Check – File Type (PDF/Image)**  
    - Type: IF  
    - Role: Routes files based on extension; image files to upload, PDFs to text extraction.  
    - Config: Checks file extensions against image list (png, jpg, bmp, etc.)  
    - Inputs: From file splitting node  
    - Outputs: Image files → Upload node; PDFs → Extract text node  
  - **Upload – JigsawStack (Storage)**  
    - Type: HTTP Request  
    - Role: Uploads image files to JigsawStack storage to obtain public URLs for classification.  
    - Config: POST binary data to storage endpoint with overwrite and temp URL enabled; HTTP header auth with Jigsaw API key.  
    - Inputs: Image files from file type check  
    - Outputs: To image classification node  
    - Edge Cases: Retry on failure, max 2 tries, 3-second wait between tries, continues on error to avoid blocking workflow.  
  - **Extract – PDF Metadata/Text**  
    - Type: Extract From File  
    - Role: Extracts both original PDF and extracted text for analysis.  
    - Config: Operation set to PDF, uses binary property named after file key.  
    - Inputs: PDFs from file type check  
    - Outputs: To file size/pages check  
  - **Check – File Size & Pages**  
    - Type: IF  
    - Role: Validates file size under 10MB and PDFs under 10 pages (note: sticky note says 20 pages limit but code uses 10).  
    - Config: Parses file size string with units (MB, KB, bytes); checks page count for PDFs.  
    - Inputs: Extracted metadata from PDF extraction node  
    - Outputs: Passes valid files to heuristic analysis; oversized files to file too large response.  
    - Edge Cases: Unknown file size formats default to false (reject).  
  - **Respond – File Too Large**  
    - Type: Respond to Webhook  
    - Role: Sends user a message explaining file limits and suggesting splitting large files.  
    - Config: Fixed JSON message.  
    - Inputs: From file size check (fail branch)  
    - Outputs: Ends workflow for that request.

#### 2.3 Heuristic Confidence Scoring

- **Overview:** Applies heuristic text analysis to PDF extracted text to generate a confidence score (0-1) that the file is a floorplan.  
- **Nodes Involved:** `Analyze – Confidence Score (Heuristics)`  

- **Node Details:**  
  - **Analyze – Confidence Score (Heuristics)**  
    - Type: Code  
    - Role: Analyzes text for keywords (Dutch and English room names), measurement units (m², mm), technical symbols, floorplan-related words, and project metadata.  
    - Config: Runs once per item, assigns weighted scores for each feature found, caps total confidence at 1.0.  
    - Key expressions: Uses regex and array filters to detect keywords and symbols.  
    - Inputs: From file size/pages check  
    - Outputs: JSON enriched with confidence score and detailed analysis reasons, passes binary unchanged.  
    - Edge Cases: Empty or missing text results in low confidence.  
    - Failures: Code errors in expressions or missing properties.

#### 2.4 Confidence-based Routing & User Response

- **Overview:** Routes files based on confidence score thresholds and returns user-facing messages accordingly.  
- **Nodes Involved:** `Route – Confidence Levels`, `Respond – Low Quality/Drop`, `Classify – PDF Text`, `Respond – Classification Result (PDF)`, `Classify – Image Files`, `Respond – Classification Result (Image)`, `No Operation, do nothing`  

- **Node Details:**  
  - **Route – Confidence Levels**  
    - Type: Switch  
    - Role: Routes files into multiple confidence brackets: <0.2, 0.2-0.4, 0.4-0.6, 0.6-0.85, ≥0.85  
    - Config: Conditions compare `$json.confidence` with thresholds, renaming outputs.  
    - Inputs: From heuristic confidence scoring  
    - Outputs: Very low confidence → respond low quality; low confidence → respond low quality; uncertain confidence → call PDF classification; high confidence → call PDF classification; very high confidence → no operation (accepted).  
    - Edge Cases: Fallback output "needs_review" routes uncertain cases to manual review.  
  - **Respond – Low Quality/Drop**  
    - Type: Respond to Webhook  
    - Role: Sends rejection message for low confidence or non-floorplan detection.  
    - Config: Conditional message based on confidence or classification prediction.  
    - Inputs: From route node for low confidence branches  
    - Outputs: Ends workflow for that request  
  - **Classify – PDF Text**  
    - Type: HTTP Request  
    - Role: Sends extracted PDF text to JigsawStack classification API for floorplan vs. not floorplan determination.  
    - Config: POST JSON with text dataset and labels, authenticated with Jigsaw API key.  
    - Inputs: From route node for uncertain or high confidence PDFs  
    - Outputs: To PDF classification response node  
    - Edge Cases: API failures, malformed text data  
  - **Respond – Classification Result (PDF)**  
    - Type: Respond to Webhook  
    - Role: Sends success or error message based on classification result from JigsawStack for PDFs.  
    - Config: Checks first prediction label equals "floorplan" for success message.  
    - Inputs: From PDF classification  
    - Outputs: Ends workflow  
  - **Classify – Image Files**  
    - Type: HTTP Request  
    - Role: Sends image file public URL (from Jigsaw storage) to JigsawStack classification API.  
    - Config: POST JSON with image dataset and labels, authenticated.  
    - Inputs: From image upload to Jigsaw storage  
    - Outputs: To image classification response node  
    - Edge Cases: HTTP errors, invalid URLs, API limits  
  - **Respond – Classification Result (Image)**  
    - Type: Respond to Webhook  
    - Role: Sends user success or error message based on image classification result.  
    - Config: Similar logic to PDF classification response node.  
  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Placeholder node for very high confidence branch where no further action is needed at this stage.  
    - Inputs: From route node for very high confidence  
    - Outputs: None

#### 2.5 Documentation and Sticky Notes

- **Overview:** Provides comprehensive documentation, user messaging guidelines, and operational notes via sticky notes visible in the editor.  
- **Nodes Involved:** All sticky note nodes: `📘 Workflow Documentation`, `Flow Summary Note`, `Low Quality Endpoint Note`, `Manual Review Note`, `Continue to JIG Note`, `Classification Success Note`, `File Limit Note`  

- **Node Details:**  
  - These sticky notes explain phases, confidence thresholds, user responses, file size constraints, and next steps in the workflow.  
  - They provide context for operators and developers but do not affect execution flow.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                        | Input Node(s)                 | Output Node(s)                       | Sticky Note                                             |
|-----------------------------|-------------------------|-------------------------------------|------------------------------|------------------------------------|---------------------------------------------------------|
| Webhook – Receive Upload     | Webhook                 | Entry point for file upload          | (external)                   | Check – GDPR Consent               |                                                         |
| Check – GDPR Consent         | IF                      | Validates user consent               | Webhook – Receive Upload     | Process – Multiple File Uploads, Respond – Consent Required |                                                         |
| Respond – Consent Required   | Respond to Webhook      | Responds if consent missing          | Check – GDPR Consent         | (end)                             |                                                         |
| Process – Multiple File Uploads | Code                 | Splits multi-file upload items       | Check – GDPR Consent         | Check – File Type (PDF/Image)      |                                                         |
| Check – File Type (PDF/Image)| IF                      | Routes files by extension            | Process – Multiple File Uploads | Upload – JigsawStack, Extract – PDF Metadata/Text |                                                         |
| Upload – JigsawStack (Storage) | HTTP Request           | Uploads images to Jigsaw storage     | Check – File Type (PDF/Image) | Classify – Image Files             |                                                         |
| Extract – PDF Metadata/Text  | Extract From File       | Extracts PDF text and metadata       | Check – File Type (PDF/Image) | Check – File Size & Pages          |                                                         |
| Check – File Size & Pages    | IF                      | Validates file size and page count   | Extract – PDF Metadata/Text  | Analyze – Confidence Score (Heuristics), Respond – File Too Large |                                                         |
| Respond – File Too Large     | Respond to Webhook      | Notifies user when file is too large | Check – File Size & Pages    | (end)                             |                                                         |
| Analyze – Confidence Score (Heuristics) | Code           | Heuristic scoring of PDF text        | Check – File Size & Pages    | Route – Confidence Levels          |                                                         |
| Route – Confidence Levels    | Switch                  | Routes files by confidence score     | Analyze – Confidence Score   | Respond – Low Quality/Drop, Classify – PDF Text, No Operation |                                                         |
| Respond – Low Quality/Drop   | Respond to Webhook      | Sends rejection or low quality message | Route – Confidence Levels   | (end)                             | Low Quality Endpoint Note                                |
| Classify – PDF Text          | HTTP Request            | Classifies PDF text via JigsawStack  | Route – Confidence Levels    | Respond – Classification Result (PDF) |                                                         |
| Respond – Classification Result (PDF) | Respond to Webhook | Sends PDF classification result to user | Classify – PDF Text          | (end)                             |                                                         |
| Classify – Image Files       | HTTP Request            | Classifies image via JigsawStack     | Upload – JigsawStack         | Respond – Classification Result (Image) |                                                         |
| Respond – Classification Result (Image) | Respond to Webhook | Sends image classification result to user | Classify – Image Files     | (end)                             |                                                         |
| No Operation, do nothing     | NoOp                    | Placeholder for accepted files       | Route – Confidence Levels    | (none)                            | Continue to JIG Note, Manual Review Note, Classification Success Note |
| 📘 Workflow Documentation    | Sticky Note             | Full workflow documentation          | (none)                      | (none)                            |                                                         |
| Flow Summary Note            | Sticky Note             | Summarizes flow logic                | (none)                      | (none)                            |                                                         |
| Low Quality Endpoint Note    | Sticky Note             | Explains low confidence rejection   | (none)                      | (none)                            | Low Quality Endpoint Note                                |
| Manual Review Note           | Sticky Note             | Explains manual review process       | (none)                      | (none)                            | Manual Review Note                                      |
| Continue to JIG Note         | Sticky Note             | Explains acceptance and next steps  | (none)                      | (none)                            | Continue to JIG Note                                    |
| Classification Success Note  | Sticky Note             | Explains high confidence success    | (none)                      | (none)                            | Classification Success Note                             |
| File Limit Note              | Sticky Note             | Explains file size/page limits       | (none)                      | (none)                            | File Limit Note                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook – Receive Upload`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `fp-mvp`  
   - Authentication: Basic Auth (configure with your username/password)  
   - Response Mode: `responseNode`  

2. **Add IF Node for GDPR Consent**  
   - Name: `Check – GDPR Consent`  
   - Type: IF  
   - Condition: Boolean expression true if either `$json.body.GDPR_check` or `$json.body.GDPR_check2` is true.  
   - If True: Proceed, else respond with consent required.  

3. **Add Respond to Webhook Node for Consent Required**  
   - Name: `Respond – Consent Required`  
   - Type: Respond to Webhook  
   - Response Body (JSON): HTML message asking user to provide consent, note GDPR compliance and data deletion after 10 minutes.  

4. **Add Code Node to Process Multiple File Uploads**  
   - Name: `Process – Multiple File Uploads`  
   - Type: Code  
   - JavaScript: Loop over `items`, for each binary file create separate item preserving binary key and metadata (filename, mimetype, extension, size).  

5. **Add IF Node to Check File Type**  
   - Name: `Check – File Type (PDF/Image)`  
   - Type: IF  
   - Condition: Check if file extension (lowercased) is one of: `png, jpg, jpeg, bmp, tiff, webp, gif` → image branch; else PDF branch.  

6. **For Image Files Branch:**  
   - Add HTTP Request Node: `Upload – JigsawStack (Storage)`  
     - Method: POST  
     - URL: `https://api.jigsawstack.com/v1/store/file`  
     - Authentication: HTTP Header Auth with Jigsaw API key  
     - Query Parameters: `key` = filename, `overwrite` = true, `temp_public_url`= true  
     - Headers: `Content-Type` = mimetype  
     - Send Binary Data: true, binary property name = fileKey  
     - Retry on fail: yes, 2 tries, 3 sec wait  
   - Add HTTP Request Node: `Classify – Image Files`  
     - Method: POST  
     - URL: `https://api.jigsawstack.com/v1/classification`  
     - Auth: Jigsaw API key  
     - Body JSON: dataset with type image and value = `{{ $json.temp_public_url }}`, labels: "floorplan", "not floorplan"  
   - Add Respond to Webhook Node: `Respond – Classification Result (Image)`  
     - Response JSON: If first prediction is "floorplan" → success message, else error message.  

7. **For PDF Files Branch:**  
   - Add Extract From File Node: `Extract – PDF Metadata/Text`  
     - Operation: PDF  
     - Binary Property Name: `{{ $json.fileKey }}`  
     - Keep Source: Both  
   - Add IF Node: `Check – File Size & Pages`  
     - Condition 1: File size under 10MB (check units KB, MB, bytes)  
     - Condition 2: If PDF, pages under 10 (note sticky note says 20 pages, adjust as needed)  
     - True branch: continue to analyze  
     - False branch: Respond with file too large message  
   - Add Respond to Webhook Node: `Respond – File Too Large`  
     - Response JSON: Message indicating file exceeds limits and suggesting splitting/extracting pages  
   - Add Code Node: `Analyze – Confidence Score (Heuristics)`  
     - JavaScript: Analyze extracted text for keywords, measurement units, symbols, floorplan terms and project metadata; assign weighted confidence score up to 1.0  
     - Output JSON includes confidence score and detailed reasons  
   - Add Switch Node: `Route – Confidence Levels`  
     - Conditions: Route based on confidence thresholds:  
       - <0.2 very likely not floorplan  
       - 0.2-0.4 likely not floorplan  
       - 0.4-0.6 uncertain low confidence  
       - 0.6-0.85 uncertain high confidence  
       - ≥0.85 very likely floorplan  
   - For <0.4 branches:  
     - Add Respond to Webhook: `Respond – Low Quality/Drop`  
       - Message: Low confidence rejection or non-floorplan message  
   - For uncertain or high confidence branches:  
     - Add HTTP Request: `Classify – PDF Text`  
       - Method: POST  
       - URL: `https://api.jigsawstack.com/v1/classification`  
       - Auth: Jigsaw API key  
       - Body: JSON dataset with text value = extracted text string, labels "floorplan" and "not floorplan"  
     - Add Respond to Webhook: `Respond – Classification Result (PDF)`  
       - Message: Success or error based on prediction results  
   - For very high confidence (≥0.85) branch:  
     - Add No Operation Node: `No Operation, do nothing`  

8. **Credential Setup:**  
   - Create HTTP Header Auth credentials for JigsawStack API key (used in all HTTP Request nodes to JigsawStack).  
   - Setup Basic Auth credentials for webhook endpoint.  

9. **Add Sticky Notes for Documentation and User Messaging:**  
   - Add sticky notes with content describing flow summary, quality thresholds, file size limits, manual review policy, and user-facing messages.  
   - Position notes near relevant nodes for clarity.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow designed to reject non-floorplans early to save API costs and reduce manual workload. | Internal workflow documentation |
| User-facing messages include clear instructions for consent, file size limits, and upload quality. | Sticky notes in workflow |
| API keys are never hardcoded; use n8n credential manager for secure storage. | Best practice |
| Supported file types: PDFs and images (JPG, PNG, BMP, TIFF, WebP, GIF). | Workflow description |
| Confidence thresholds adjustable to tune strictness of filtering. | Workflow documentation sticky note |
| Workflow can be extended with measurement extraction in a paid version using Mistral AI OCR. | Workflow description |
| Uses JigsawStack API for file storage and classification: https://jigsawstack.com | External API reference |
| GDPR compliance ensured by explicit consent check before processing. | Workflow logic |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. All processing complies with applicable content policies and handles only legal, public data.