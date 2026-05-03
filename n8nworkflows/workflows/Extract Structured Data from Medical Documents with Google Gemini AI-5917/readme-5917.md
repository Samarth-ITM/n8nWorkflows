Extract Structured Data from Medical Documents with Google Gemini AI

https://n8nworkflows.xyz/workflows/extract-structured-data-from-medical-documents-with-google-gemini-ai-5917


# Extract Structured Data from Medical Documents with Google Gemini AI

### 1. Workflow Overview

This workflow automates the extraction of structured data from medical documents using Google Gemini AI. It is designed for applications such as medical billing automation, insurance claim processing, and clinical data extraction. The workflow processes an input image URL of a medical document, performs classification of the document type, extracts text with OCR, and converts the extracted data into a standardized structured JSON format compliant with medical taxonomy and regulatory requirements.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Validation:** Receives and validates incoming requests with medical document image URLs and optional hints.
- **1.2 Image Acquisition & Preparation:** Downloads the document image and converts it into a suitable format (base64) for AI processing.
- **1.3 AI Classification & Text Extraction:** Calls Google Gemini AI to classify the document type and extract all visible text with language detection.
- **1.4 Structured Data Generation:** Uses Google Gemini AI again to convert extracted raw text into a structured medical document schema.
- **1.5 Finalization & API Response:** Prepares a comprehensive response including structured data, quality metrics, token usage, and processing metadata, then returns it to the requester.

Supporting sticky notes provide documentation, setup instructions, and technical details throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

**Overview:**  
This block handles incoming HTTP requests, validates the payload, and extracts necessary parameters for downstream processing.

**Nodes Involved:**  
- Webhook Input  
- Parse Input

**Node Details:**

- **Webhook Input**  
  - Type: Webhook  
  - Role: Entry point to receive POST requests to `/webhook/analyze-medical-document` with JSON payload.  
  - Configuration: Accepts JSON body containing `image_url` (mandatory), `expected_type` (optional document type hint), and `language_hint` (optional language hint).  
  - Input: HTTP request  
  - Output: JSON body forwarded to next node  
  - Failure Modes: Missing or malformed payload, invalid HTTP method, webhook configuration errors.

- **Parse Input**  
  - Type: Set  
  - Role: Extracts and validates input JSON parameters; sets default values for optional fields.  
  - Configuration: Assigns `image_url` from `$json.body.image_url`, `expected_type` defaults to `'unknown'` if not provided, `language_hint` defaults to `'auto'`.  
  - Input: JSON from Webhook Input  
  - Output: Cleaned parameters for image processing  
  - Failure Modes: Missing `image_url` could cause downstream failures if not handled; expression failures if input format unexpected.

---

#### 2.2 Image Acquisition & Preparation

**Overview:**  
Downloads the medical document image from the URL and converts the binary image data into a base64-encoded string suitable for AI API input.

**Nodes Involved:**  
- Download Image  
- Extract to Base64  
- Prepare for AI

**Node Details:**

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads the image from the provided URL.  
  - Configuration: Uses `$json.image_url` dynamically; expects image data in binary file format. Supports JPEG, PNG, WebP.  
  - Input: JSON with `image_url`  
  - Output: Binary image data with MIME type metadata  
  - Failure Modes: Network errors, invalid URL, unsupported image formats, timeout, HTTP errors (404, 403).

- **Extract to Base64**  
  - Type: Extract From File  
  - Role: Converts downloaded binary image to base64 string and preserves original binary.  
  - Configuration: Operation is `binaryToProperty`; base64 stored under `image_data`.  
  - Input: Binary image from Download Image  
  - Output: JSON with base64 string and original binary  
  - Failure Modes: Binary data corruption, unsupported file types.

- **Prepare for AI**  
  - Type: Set  
  - Role: Organizes data for AI API call, extracts MIME type, includes all relevant fields.  
  - Configuration: Extracts MIME type from binary metadata, keeps all other fields.  
  - Input: JSON with base64 image and metadata  
  - Output: JSON ready for Google Gemini API consumption  
  - Failure Modes: Missing MIME type, data inconsistency.

---

#### 2.3 AI Classification & Text Extraction

**Overview:**  
Calls Google Gemini AI API to classify the document type, extract text using OCR with multi-language detection, and provide confidence scores.

**Nodes Involved:**  
- Gemini Classify Extract  
- Parse AI Results

**Node Details:**

- **Gemini Classify Extract**  
  - Type: HTTP Request  
  - Role: Sends image data and instructions to Google Gemini 2.0 Flash model for classification and text extraction.  
  - Configuration:  
    - POST request to Gemini API endpoint.  
    - JSON body includes base64 image data, MIME type, detailed prompt specifying classification categories and extraction tasks, expected type hint, and language hint.  
    - Generation config sets temperature to 0.1 (low randomness), max tokens 4096, requests JSON output with a strict schema.  
    - Uses predefined Google Gemini API credentials.  
  - Input: Prepared JSON with image and parameters  
  - Output: AI response JSON with classification, extracted text, languages, and image quality metrics  
  - Failure Modes: API authentication errors, rate limit exceeded, malformed request, model errors, timeout, invalid API key.

- **Parse AI Results**  
  - Type: Set  
  - Role: Parses the JSON text response from Gemini, extracting classification output and preparing for structured data generation.  
  - Configuration: Parses JSON string found in the first candidate's content part.  
  - Input: Gemini API raw response  
  - Output: Parsed classification and extraction results as JSON object  
  - Failure Modes: JSON parse errors if AI response malformed, missing fields.

---

#### 2.4 Structured Data Generation

**Overview:**  
Transforms the extracted text into a detailed structured medical document JSON compliant with taxonomy, regulatory requirements, and including metadata and quality metrics.

**Nodes Involved:**  
- Gemini Structure Data

**Node Details:**

- **Gemini Structure Data**  
  - Type: HTTP Request  
  - Role: Calls Google Gemini AI again to transform raw extracted text and classification into a final structured medical document schema.  
  - Configuration:  
    - POST to Gemini API with base64 image and a prompt that instructs the model to create a final validated structured output based on document type and extracted text.  
    - Schema includes fields such as documentId, metadata (createdDate, providerId, patientId, etc.), content (amount, medications, diagnosis, etc.), and quality metrics.  
    - Parameters tuned for high accuracy and compliance (temperature 0.1, max tokens 8192).  
    - Uses same Google Gemini API credentials.  
  - Input: Parsed classification results combined with base64 image  
  - Output: JSON with fully structured and validated medical document data  
  - Failure Modes: API errors, exceeding token limits, incomplete data, invalid schema response.

---

#### 2.5 Finalization & API Response

**Overview:**  
Prepares the final output, aggregates token usage and metadata, and returns the standardized JSON response to the API caller.

**Nodes Involved:**  
- Finalize Track  
- API Response

**Node Details:**

- **Finalize Track**  
  - Type: Set  
  - Role: Parses the final structured JSON from Gemini, calculates token usage from both AI calls, adds processing metadata including timestamps and workflow version.  
  - Configuration:  
    - Parses final result JSON from Gemini Structure Data response.  
    - Collects token usage metrics from both classification and structuring API calls.  
    - Adds metadata such as workflow version, processing timestamp, stages completed, and models used.  
  - Input: Gemini structured data response and previous usage metadata  
  - Output: JSON object with final result, token usage, and metadata  
  - Failure Modes: JSON parse errors, missing usage data.

- **API Response**  
  - Type: Respond To Webhook  
  - Role: Sends the final JSON response back to the original HTTP requestor.  
  - Configuration: Responds with JSON containing success status, structured medical document result, token usage, and processing metadata.  
  - Input: Finalized data from Finalize Track  
  - Output: HTTP response to client  
  - Failure Modes: Network errors, webhook response failures.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                               | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                 |
|-----------------------|-------------------------|-----------------------------------------------|-------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Sticky Note           | Sticky Note             | Provides overview and use case summary         |                         |                         | # Medical Document AI Analysis<br>Extract structured data from medical documents using Google Gemini AI ...                   |
| Sticky Note1          | Sticky Note             | Setup instructions and API key requirements    |                         |                         | # Setup Requirements<br>What you need:<br>• Google Gemini API Key (from Google AI Studio)...                                  |
| Sticky Note3          | Sticky Note             | Example input/output JSON for the API           |                         |                         | # API Usage<br>Input:<br>{ "image_url": "https://example.com/receipt.jpg" }<br>Output sample JSON ...                          |
| Webhook Input         | Webhook                 | Receives medical document analysis requests    |                         | Parse Input              | Receives medical document analysis requests<br>Accepts JSON payload with:<br>• image_url<br>• expected_type (optional)...    |
| Parse Input           | Set                     | Validates and extracts input parameters         | Webhook Input            | Download Image           | Validates and extracts input parameters<br>Sets defaults for optional fields                                                  |
| Download Image        | HTTP Request            | Downloads image from URL                         | Parse Input              | Extract to Base64        | Downloads image from provided URL<br>Supports JPEG, PNG, WebP formats                                                       |
| Extract to Base64     | Extract From File       | Converts binary image to base64                  | Download Image           | Prepare for AI           | Converts binary image to base64 format<br>Prepares data for AI processing                                                    |
| Prepare for AI        | Set                     | Organizes data for AI processing                  | Extract to Base64        | Gemini Classify Extract  | Extracts MIME type and combines data<br>Validates data completeness                                                         |
| Gemini Classify Extract| HTTP Request            | Calls Google Gemini AI for classification and OCR| Prepare for AI           | Parse AI Results         | Document classification and text extraction using Google Gemini 2.0 Flash<br>Optimized for medical documents                |
| Parse AI Results      | Set                     | Parses classification and text extraction results| Gemini Classify Extract  | Gemini Structure Data    | Processes Gemini classification results<br>Parses JSON response                                                             |
| Gemini Structure Data | HTTP Request            | Converts extracted text into structured medical data| Parse AI Results         | Finalize Track           | Converts extracted text into structured medical data<br>Ensures regulatory compliance                                        |
| Finalize Track        | Set                     | Prepares final response with metrics and metadata| Gemini Structure Data    | API Response             | Prepares final response with comprehensive metrics<br>Calculates token usage, adds timestamps                                |
| API Response          | Respond To Webhook      | Returns standardized JSON response to client    | Finalize Track           |                         | Returns structured medical document data<br>Includes quality and usage metrics                                               |
| Sticky Note5          | Sticky Note             | AI model technical details                       |                         |                         | # Technical Details<br>AI Model: Google Gemini 2.0 Flash<br>Performance metrics                                               |
| Sticky Note7          | Sticky Note             | Quick start guide                               |                         |                         | # Quick Start<br>Test in 3 steps:<br>1. Import workflow<br>2. Add Gemini API credentials<br>3. Send test request              |
| Sticky Note2          | Sticky Note             | Reminder to replace Gemini API Key               |                         |                         | ## Replace with your Gemini API Key                                                                                          |
| Sticky Note4          | Sticky Note             | Reminder to replace Gemini API Key               |                         |                         | ## Replace with your Gemini API Key                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Input Node**  
   - Type: Webhook  
   - Set path to `analyze-medical-document`  
   - HTTP Method: POST  
   - Response Mode: Response Node  
   - Description: Receives medical document analysis requests with JSON payload containing at least `image_url`.

2. **Add a Set Node (Parse Input)**  
   - Connect from Webhook Input  
   - Extract and assign:  
     - `image_url` = `{{$json.body.image_url}}` (string)  
     - `expected_type` = `{{$json.body.expected_type || 'unknown'}}` (string, default 'unknown')  
     - `language_hint` = `{{$json.body.language_hint || 'auto'}}` (string, default 'auto')  
   - Purpose: Validate and prepare input parameters.

3. **Add HTTP Request Node (Download Image)**  
   - Connect from Parse Input  
   - URL: `{{$json.image_url}}`  
   - Response Format: File (binary)  
   - Purpose: Download image from provided URL.

4. **Add Extract From File Node (Extract to Base64)**  
   - Connect from Download Image  
   - Operation: `binaryToProperty`  
   - Destination Key: `image_data`  
   - Keep original binary as well (`keepSource = both`)  
   - Purpose: Convert image binary to base64 string for API.

5. **Add Set Node (Prepare for AI)**  
   - Connect from Extract to Base64  
   - Assign `mime_type` from binary metadata: `{{$binary.data.mimeType}}`  
   - Include all other fields  
   - Purpose: Organize data for Google Gemini API request.

6. **Add HTTP Request Node (Gemini Classify Extract)**  
   - Connect from Prepare for AI  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
   - Authentication: Use Google Gemini API credentials (set up in n8n credentials)  
   - Body Type: JSON  
   - Body: Compose JSON with:  
     - `contents` array containing:  
       - `inline_data` with `mime_type` and base64 `data`  
       - `text` with prompt instructing classification and text extraction, including taxonomy and hints from input (`expected_type`, `language_hint`)  
     - `generationConfig` with low temperature (0.1), topP 0.8, maxOutputTokens 4096, `response_mime_type` application/json, and a strict response schema for classification and extraction results.  
   - Purpose: Classify document and extract all visible text.

7. **Add Set Node (Parse AI Results)**  
   - Connect from Gemini Classify Extract  
   - Parse JSON from string response field: `JSON.parse($json.candidates[0].content.parts[0].text)`  
   - Assign to `classification_output`  
   - Purpose: Extract structured classification and text extraction data from AI response.

8. **Add HTTP Request Node (Gemini Structure Data)**  
   - Connect from Parse AI Results  
   - Method: POST  
   - URL: Same Gemini API endpoint as before  
   - Authentication: Google Gemini API credentials  
   - Body: JSON with:  
     - `contents` containing:  
       - `inline_data` with `mime_type` and base64 `data` (from earlier Prepare for AI node)  
       - `text` prompt instructing to create final structured medical document data conforming to taxonomy using classification and extracted text from previous step.  
     - `generationConfig` with temperature 0.1, maxOutputTokens 8192, `response_mime_type` application/json, and a strict output schema describing detailed medical document fields and quality metrics.  
   - Purpose: Generate fully structured and validated medical document data.

9. **Add Set Node (Finalize Track)**  
   - Connect from Gemini Structure Data  
   - Parse final JSON: `JSON.parse($json.candidates[0].content.parts[0].text)` assigned to `final_result`  
   - Aggregate token usage from both Gemini API calls for cost monitoring  
   - Add metadata: workflow version, processing timestamp, stages completed, AI models used  
   - Purpose: Prepare final data package including metrics.

10. **Add Respond to Webhook Node (API Response)**  
    - Connect from Finalize Track  
    - Respond with JSON containing:  
      - `success: true`  
      - `result`: final structured medical document data  
      - `token_usage`: aggregated tokens from AI calls  
      - `processing_metadata`: metadata object  
    - Purpose: Return standardized API response.

11. **Set Up Credentials**  
    - Configure Google Gemini API credentials in n8n (OAuth2 or API key as required by Google AI Studio).  
    - Ensure credentials are referenced in HTTP Request nodes calling Gemini API.

12. **Add Sticky Notes (Optional for Documentation)**  
    - Add notes at strategic points with setup instructions, API usage examples, technical details, quick start guides, and reminders to replace API keys.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow uses Google Gemini 2.0 Flash model optimized for medical document processing with superior OCR and multi-language support.                                                                                             | See Sticky Note5                                                                                             |
| Setup requires Google Gemini API Key from Google AI Studio and configuring credentials in n8n.                                                                                                                                 | See Sticky Note1 and Sticky Note2/Sticky Note4                                                              |
| Estimated cost per document ranges from $0.01 to $0.05 depending on token usage; monitor API quotas carefully to avoid service disruption.                                                                                    | See Sticky Note1                                                                                             |
| Input JSON example: `{ "image_url": "https://example.com/receipt.jpg" }` with optional `expected_type` and `language_hint` for improved classification accuracy.                                                               | See Sticky Note3                                                                                             |
| Output is a structured JSON document compliant with detailed medical taxonomy schema including metadata, content details, and quality metrics suitable for billing, insurance, and clinical data workflows.                      | Described in Gemini Structure Data node schema                                                              |
| Quick start instructions: Import workflow, add Gemini API credentials, send POST request to webhook with image URL.                                                                                                            | See Sticky Note7                                                                                             |
| Consider adding database storage, batch processing, or custom validation for production deployments.                                                                                                                          | Suggested in Sticky Note7                                                                                     |

---

**Disclaimer:** The provided text is exclusively sourced from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.