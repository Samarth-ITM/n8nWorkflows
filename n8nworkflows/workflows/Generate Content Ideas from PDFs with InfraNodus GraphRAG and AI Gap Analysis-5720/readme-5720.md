Generate Content Ideas from PDFs with InfraNodus GraphRAG and AI Gap Analysis

https://n8nworkflows.xyz/workflows/generate-content-ideas-from-pdfs-with-infranodus-graphrag-and-ai-gap-analysis-5720


# Generate Content Ideas from PDFs with InfraNodus GraphRAG and AI Gap Analysis

---

### 1. Workflow Overview

This workflow facilitates the generation of content ideas and identification of knowledge gaps from PDF documents using InfraNodus’s GraphRAG AI technology. It is designed for users who upload PDF files, which are then processed to extract meaningful textual data. The extracted text is analyzed as a network graph of concepts to detect under-explored areas (content gaps). The AI then generates actionable ideas or questions bridging these gaps.

**Target Use Cases:**  
- Content creators and researchers aiming to discover novel content ideas from existing PDFs.  
- Teams looking to identify thematic gaps in documentation or literature collections.  
- Knowledge management workflows integrating AI-driven gap analysis.

**Logical Blocks:**  
- **1.1 Input Reception:** User uploads PDF files via a form trigger.  
- **1.2 Binary Preparation:** Uploaded files are converted/prepared as PDF binaries for text extraction.  
- **1.3 Text Extraction:** Extract plain text content from PDFs.  
- **1.4 Data Preparation:** Combine extracted text and generate parameters for InfraNodus analysis.  
- **1.5 AI Gap Analysis:** Call InfraNodus API to generate content gap insights and advice.  
- **1.6 Output Delivery:** Display AI-generated advice or questions back to the user via form response.

An optional alternative conversion method using ConvertAPI is included but disabled by default, providing higher quality PDF-to-text conversion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception  
**Overview:**  
This block captures user-uploaded PDF files via a web-accessible form endpoint. It initiates the workflow by receiving files for analysis.

**Nodes Involved:**  
- On form submission  
- Sticky Note (Step 1)

**Node Details:**  

- **On form submission**  
  - Type: `formTrigger`  
  - Role: Entry point for workflow; receives PDF files uploaded by user.  
  - Configuration: Form titled "Find Content Gaps in Your PDF Files" with a single file input field restricted to `.pdf`. Attribution disabled.  
  - Inputs: HTTP form submission with binary PDF files.  
  - Outputs: Emits items with binary data for downstream processing.  
  - Edge Cases: Upload of unsupported file types; large files causing timeouts or memory issues; no file uploaded.  
  - Notes: Exposed webhook URL enables public access.

- **Sticky Note (Step 1)**  
  - Provides user instruction highlighting this block’s role and public URL exposure.

---

#### 2.2 Binary Preparation  
**Overview:**  
Converts incoming binary data from the form into a consistent PDF binary format, preparing files for text extraction.

**Nodes Involved:**  
- Convert binary files to PDF  
- Sticky Note1

**Node Details:**  

- **Convert binary files to PDF**  
  - Type: `code`  
  - Role: Processes incoming binary items; restructures each binary file into individual items with a sanitized filename key for further operations.  
  - Configuration: JavaScript iterates over all binary properties, replaces spaces with underscores in keys, and outputs one item per binary.  
  - Inputs: Items with multiple binary fields from form submission.  
  - Outputs: New items each containing one PDF binary under a normalized key.  
  - Edge Cases: No binary data present; malformed binaries; large files causing processing delays.  
  - Version: n8n code node v2.  
  - Notes: Handles multiple files and binary keys gracefully.

- **Sticky Note1**  
  - Explains the purpose: converting uploaded binaries into PDF files to enable text extraction.

---

#### 2.3 Text Extraction  
**Overview:**  
Extracts plain text from each PDF binary file for analysis.

**Nodes Involved:**  
- Extract text from PDF files  
- Sticky Note2  
- (Optional) Convert File to PDF (disabled)  
- Sticky Note5 (optional better conversion)

**Node Details:**  

- **Extract text from PDF files**  
  - Type: `extractFromFile`  
  - Role: Extracts text content from PDFs.  
  - Configuration: Operates on binary data keyed by dynamic filename (`={{ $json.fileName }}`). No additional options set.  
  - Inputs: PDF binaries from previous code node.  
  - Outputs: JSON with extracted text property `text`.  
  - Edge Cases: Corrupted PDFs; unsupported PDF versions; empty or scanned-only PDFs without selectable text.  
  - Version: v1.  
  - Notes: Default extractor; may split text into short chunks.

- **Convert File to PDF** (disabled)  
  - Type: `httpRequest`  
  - Role: Optional, higher quality PDF to text conversion via ConvertAPI service, preserving paragraph structure.  
  - Configuration: POST request to ConvertAPI; multipart form-data with binary file; uses bearer token authentication.  
  - Inputs: Binary PDF from form.  
  - Outputs: Text as octet-stream response.  
  - Edge Cases: API rate limits; authentication failure; network errors.  
  - Notes: Requires ConvertAPI account and correct mapping downstream.

- **Sticky Note2**  
  - Describes the standard extraction step and optional ConvertAPI alternative.

- **Sticky Note5**  
  - Details benefits of using ConvertAPI for preserving document layout and better text quality.

---

#### 2.4 Data Preparation  
**Overview:**  
Aggregates all extracted text from multiple PDFs into a single string and generates a random parameter controlling depth of gap analysis.

**Nodes Involved:**  
- Prepare for InfraNodus  
- Sticky Note3

**Node Details:**  

- **Prepare for InfraNodus**  
  - Type: `code`  
  - Role: Concatenates all extracted text into one plain text string and generates a random integer (0-2) named `randomNum` used for gap depth in AI request.  
  - Configuration: JavaScript iterates over input items, appends their `text` field separated by double newlines, returns object with combined `text` and `randomNum`.  
  - Inputs: Multiple JSON items each with `text` field from text extraction node.  
  - Outputs: Single JSON object with properties `text` (string) and `randomNum` (integer).  
  - Edge Cases: Empty or missing text fields; very large combined texts causing API throttling.  
  - Version: v2.

- **Sticky Note3**  
  - Explains the role of combining text and setting gap depth parameter for InfraNodus.

---

#### 2.5 AI Gap Analysis  
**Overview:**  
Sends the combined text to InfraNodus GraphRAG API, requesting AI-generated content ideas highlighting structural gaps in the knowledge graph. The API response includes recommended ideas or prompts bridging distant concepts.

**Nodes Involved:**  
- InfraNodus GraphRAG AI Advice  
- Sticky Note4  
- InfraNodus AI Questions (disabled)  
- Sticky Note6

**Node Details:**  

- **InfraNodus GraphRAG AI Advice**  
  - Type: `httpRequest`  
  - Role: Calls InfraNodus API with combined text and parameters to receive content gap advice.  
  - Configuration: POST request to InfraNodus `/graphAndAdvice` endpoint with query parameters: `doNotSave=true`, `optimize=develop`, `includeGraph=false`, `includeGraphSummary=true`, and dynamic `gapDepth` from `randomNum`. Body includes parameters: `aiTopics=true`, `requestMode=idea`, and the combined text. Uses bearer token authentication with InfraNodus API key credential.  
  - Inputs: Single JSON with combined text and randomNum.  
  - Outputs: API JSON response including AI-generated ideas/advice array.  
  - Edge Cases: API authentication errors; rate limits; malformed text input; network timeouts.  
  - Notes: Disabled alternative node “InfraNodus AI Questions” exists to generate questions instead of ideas.

- **InfraNodus AI Questions** (disabled)  
  - Same API endpoint but with `requestMode=question` to generate AI questions rather than ideas.  
  - Disabled by default.

- **Sticky Note4**  
  - Instructions to insert InfraNodus API key and explanation of generating ideas vs. questions.

- **Sticky Note6**  
  - Notes on optionally forwarding or exposing the AI result in other apps or workflows.

---

#### 2.6 Output Delivery  
**Overview:**  
Presents the AI-generated content ideas or questions back to the user on the form submission page.

**Nodes Involved:**  
- Display on the Form to the User  
- Sticky Note7 (general explanation)

**Node Details:**  

- **Display on the Form to the User**  
  - Type: `form` node  
  - Role: Responds to the form submission with AI-generated advice text rendered in HTML.  
  - Configuration: Completion mode with response text formatted as an `<h3>` heading showing the first item from the AI advice array (`{{ $json.aiAdvice[0].text }}`).  
  - Inputs: JSON output from InfraNodus AI Advice node.  
  - Outputs: HTTP response to the user’s form submission.  
  - Edge Cases: Empty or malformed AI advice; response rendering issues.  
  - Version: v1.

- **Sticky Note7**  
  - Provides conceptual explanation of InfraNodus GraphRAG methodology and workflow purpose with a helpful diagram and links.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                                | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                                                                                                                                                             |
|-------------------------------|---------------------|-----------------------------------------------|---------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission             | formTrigger         | Entry point: accepts PDF file uploads         | —                         | Convert binary files to PDF  | Step 1: User uploads the PDF files for analysis. You can expose this endpoint publicly.                                                                                                                                                |
| Convert binary files to PDF    | code                | Normalize binaries into PDF files              | On form submission         | Extract text from PDF files  | Step 2: Convert uploaded binaries into PDF files so text can be extracted.                                                                                                                                                            |
| Extract text from PDF files    | extractFromFile     | Extract plain text from PDFs                    | Convert binary files to PDF| Prepare for InfraNodus       | Step 3: Extract plain text from PDF files. For better quality, optionally use ConvertAPI to preserve formatting.                                                                                                                     |
| Convert File to PDF            | httpRequest (disabled)| Optional higher quality PDF to text conversion | — (not connected)          | —                           | Optional: Better PDF conversion using ConvertAPI to preserve layout and paragraphs. Requires ConvertAPI account and mapping downstream.                                                                                              |
| Prepare for InfraNodus         | code                | Aggregate text and generate gap depth param    | Extract text from PDF files| InfraNodus GraphRAG AI Advice| Step 4: Combine extracted text and set gap depth for InfraNodus analysis.                                                                                                                                                            |
| InfraNodus GraphRAG AI Advice  | httpRequest         | API call to InfraNodus for content gap ideas   | Prepare for InfraNodus     | Display on the Form to the User | Step 5: Use InfraNodus GraphRAG AI to find content gaps and generate advice. Insert your InfraNodus API key here.                                                                                                                     |
| InfraNodus AI Questions (disabled) | httpRequest (disabled)| Alternative API call to generate AI questions | —                         | —                           | Optionally generate questions instead of ideas using InfraNodus API. Disabled by default.                                                                                                                                             |
| Display on the Form to the User| form                | Show AI-generated advice back to user          | InfraNodus GraphRAG AI Advice | —                         | Step 6: Display advice/questions to the user. Can be forwarded or embedded in other apps.                                                                                                                                             |
| Sticky Note                   | stickyNote          | Explanatory notes for Step 1                    | —                         | —                           | Step 1: User uploads the PDF files for analysis.                                                                                                                                                                                     |
| Sticky Note1                  | stickyNote          | Explanatory notes for Step 2                    | —                         | —                           | Step 2: Convert uploaded binaries into PDF files.                                                                                                                                                                                    |
| Sticky Note2                  | stickyNote          | Explanatory notes for Step 3                    | —                         | —                           | Step 3: Extract plain text from PDF files. Optionally use ConvertAPI.                                                                                                                                                                |
| Sticky Note3                  | stickyNote          | Explanatory notes for Step 4                    | —                         | —                           | Step 4: Combine extracted text and set gap depth for InfraNodus.                                                                                                                                                                    |
| Sticky Note4                  | stickyNote          | Explanatory notes for Step 5                    | —                         | —                           | Step 5: Use InfraNodus GraphRAG. Provide API key. Optionally generate questions.                                                                                                                                                      |
| Sticky Note5                  | stickyNote          | Optional better PDF conversion notes            | —                         | —                           | Optional: Use ConvertAPI for better PDF to text conversion preserving layout.                                                                                                                                                        |
| Sticky Note6                  | stickyNote          | Explanatory notes for Step 6                    | —                         | —                           | Step 6: Show advice/questions to user. Can forward to other workflows or apps.                                                                                                                                                       |
| Sticky Note7                  | stickyNote          | General explanation of InfraNodus GraphRAG     | —                         | —                           | Overview of InfraNodus method and how GraphRAG finds content gaps with AI. Includes diagram and official links: https://infranodus.com                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `On form submission` node**  
   - Type: `formTrigger`  
   - Configure form title: "Find Content Gaps in Your PDF Files"  
   - Add one field: File upload accepting `.pdf` files only  
   - Disable attribution append  
   - Save and note webhook URL for public access.

2. **Add `Convert binary files to PDF` code node**  
   - Paste the provided JavaScript code that iterates over binary fields, normalizes keys by replacing spaces with underscores, and outputs one item per binary PDF file.  
   - Connect output of `On form submission` to this node.

3. **Add `Extract text from PDF files` node**  
   - Type: `extractFromFile`  
   - Operation: `pdf`  
   - Binary Property Name: Set dynamically as `={{ $json.fileName }}`  
   - Connect output of `Convert binary files to PDF` to this node.

4. **(Optional) Add `Convert File to PDF` HTTP Request node for better conversion**  
   - Disabled by default  
   - Configure POST request to `https://v2.convertapi.com/convert/pdf/to/txt`  
   - Use multipart form-data with binary file input  
   - Authenticate using HTTP Bearer with ConvertAPI credentials  
   - Map response as text output  
   - To use, disconnect `Extract text from PDF files` and connect this node instead, adjusting downstream accordingly.

5. **Add `Prepare for InfraNodus` code node**  
   - Use JavaScript code that concatenates all input texts separated by double newlines into a single string.  
   - Generate a random integer `randomNum` from 0 to 2 for gap depth control.  
   - Return object with `text` and `randomNum` properties.  
   - Connect output of `Extract text from PDF files` (or optional ConvertAPI node) to this node.

6. **Add `InfraNodus GraphRAG AI Advice` HTTP Request node**  
   - POST to URL `https://infranodus.com/api/v1/graphAndAdvice` with query parameters:  
     - `doNotSave=true`  
     - `optimize=develop`  
     - `includeGraph=false`  
     - `includeGraphSummary=true`  
     - `gapDepth={{ $json.randomNum }}` (dynamic)  
   - Body parameters:  
     - `aiTopics` = `true`  
     - `requestMode` = `idea`  
     - `text` = `={{ $json.text }}`  
   - Authenticate with InfraNodus API key using HTTP Bearer Auth credential.  
   - Connect output of `Prepare for InfraNodus` to this node.

7. **(Optional) Create alternative `InfraNodus AI Questions` node** (disabled)  
   - Same as above but set `requestMode` = `question` to generate AI questions instead of ideas.

8. **Add `Display on the Form to the User` node**  
   - Type: `form` node (completion mode)  
   - Configure to respond with HTML text showing first AI advice: `<h3>{{ $json.aiAdvice[0].text }}</h3>`  
   - Connect output of `InfraNodus GraphRAG AI Advice` to this node.

9. **Add Sticky Notes with content as per steps 1 to 6 and general InfraNodus explanation**  
   - Position and color notes to match logical blocks for documentation clarity.

10. **Validate credentials:**  
    - Set up HTTP Bearer Auth credentials for InfraNodus API key.  
    - (Optional) Set up HTTP Bearer Auth credentials for ConvertAPI if using the optional conversion node.

11. **Test workflow:**  
    - Upload PDF files via the form URL.  
    - Confirm extracted text is concatenated.  
    - Verify AI advice is returned and displayed on form response.  
    - Troubleshoot for API errors, large files, or malformed inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                              | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| InfraNodus GraphRAG represents text as a network of concepts and identifies structural gaps where clusters are distant but contextually related. AI generates bridging ideas or questions. | https://infranodus.com                              |
| Optional higher quality PDF to text conversion can be achieved using ConvertAPI, which respects document layout and paragraphs better than standard extraction.                           | https://convertapi.com?ref=4l54n                     |
| Workflow form trigger webhook can be exposed publicly to allow organizational access to the content gap analysis service.                                                                  | n8n formTrigger node configuration                   |
| InfraNodus API key must be provided in the HTTP Bearer Auth credential for AI requests to succeed.                                                                                          | InfraNodus DeeMeeTree API Key credential             |
| AI-generated insights can be further integrated with other n8n workflows or displayed within custom applications via webhooks or iframes.                                                | Custom integrations via n8n or web apps               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---