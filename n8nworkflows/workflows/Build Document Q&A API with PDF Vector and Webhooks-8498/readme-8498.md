Build Document Q&A API with PDF Vector and Webhooks

https://n8nworkflows.xyz/workflows/build-document-q-a-api-with-pdf-vector-and-webhooks-8498


# Build Document Q&A API with PDF Vector and Webhooks

### 1. Workflow Overview

This workflow implements a **Document Q&A API** designed to provide intelligent answers to user questions based on uploaded or referenced documents. It exposes a RESTful webhook endpoint where clients can submit questions along with document references (URLs or IDs). The workflow then leverages a PDF Vector AI node (GPT-4 powered) to parse the document, find relevant information, generate answers, and return structured JSON responses with answers, citations, and confidence scores.

The workflow’s logic is organized into the following blocks:

- **1.1 Input Reception and Validation:** Receives incoming API requests via webhook, validates the request payload ensuring necessary parameters are present, and prepares metadata like session IDs and timestamps.
- **1.2 Conditional Routing:** Checks if the request is valid; if invalid, prepares an error response immediately.
- **1.3 AI Processing:** For valid requests, uses the PDF Vector node to query the document and generate an answer.
- **1.4 Response Formatting and Delivery:** Formats the AI’s output into a consistent JSON response structure (success or error) and sends it back to the API caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block handles the initial reception of HTTP POST requests to the `/doc-qa` webhook endpoint. It validates that the incoming request contains either a document URL or document ID and a question. It also generates a session ID if one is not provided and timestamps the request.

**Nodes Involved:**  
- Webhook  
- Validate Request

**Node Details:**  

- **Webhook**  
  - Type: Webhook trigger  
  - Role: Entry point exposing `POST /doc-qa` as the API endpoint  
  - Configuration: Path set to `doc-qa`, HTTP method POST, webhook ID is `doc-qa-webhook`  
  - Input: External HTTP POST request with JSON payload  
  - Output: Passes request body to the next node  
  - Edge Cases: Invalid HTTP methods or malformed requests will not trigger this node  

- **Validate Request**  
  - Type: Code node (JavaScript)  
  - Role: Validates the presence of required fields (`documentUrl` or `documentId`, and `question`) and enriches the payload with `sessionId` and `timestamp`  
  - Configuration: Custom JS code that checks fields, accumulates errors, and returns validation status as `valid` boolean  
  - Key Expressions:  
    ```js
    const body = $input.first().json.body;
    const errors = [];
    if (!body.documentUrl && !body.documentId) errors.push('Either documentUrl or documentId is required');
    if (!body.question) errors.push('Question is required');
    const sessionId = body.sessionId || `session-${Date.now()}`;
    return [{ json: { ...body, sessionId, valid: errors.length === 0, errors, timestamp: new Date().toISOString() } }];
    ```  
  - Input: Raw request JSON  
  - Output: JSON enriched with validation info and errors array  
  - Edge Cases: Missing fields cause `valid` to be false with error messages; malformed JSON payloads may cause runtime errors  

#### 2.2 Conditional Routing

**Overview:**  
Determines whether the request is valid. If yes, routes to AI processing; if not, sends an error response.

**Nodes Involved:**  
- Valid Request? (If node)  
- Format Error Response

**Node Details:**  

- **Valid Request?**  
  - Type: If node  
  - Role: Branches workflow based on validation result  
  - Configuration: Checks if `$json.valid === true`  
  - Input: JSON from Validate Request  
  - Output: Two branches: main[0] for valid requests, main[1] for invalid requests  

- **Format Error Response**  
  - Type: Code node  
  - Role: Constructs a standardized error JSON response  
  - Configuration: Returns `success: false`, error messages, generic message, and timestamp  
  - Key Expressions:  
    ```js
    const errors = $json.errors || ['An error occurred processing your request'];
    return [{ json: { success: false, errors, message: 'Invalid request', timestamp: new Date().toISOString() } }];
    ```  
  - Input: JSON with errors from Validate Request  
  - Output: Error response JSON  
  - Edge Cases: Missing or malformed error array defaults to generic error message  

#### 2.3 AI Processing

**Overview:**  
Processes valid requests by querying the PDF Vector AI node to answer the question based on the provided document.

**Nodes Involved:**  
- PDF Vector - Ask Question  
- Format Success Response

**Node Details:**  

- **PDF Vector - Ask Question**  
  - Type: PDF Vector node (custom AI integration)  
  - Role: Sends a prompt to GPT-4 powered AI to parse the document and answer the question  
  - Configuration:  
    - `url`: Uses the `documentUrl` from the incoming JSON  
    - `prompt`: A template string: `"Answer the following question about this document or image: {{ $json.question }}"`  
    - `resource`: Document  
    - `inputType`: URL  
    - `operation`: ask  
  - Input: Validated request JSON including question and document URL  
  - Output: AI-generated answer with citations  
  - Edge Cases:  
    - Network errors or invalid URLs may cause failures  
    - AI timeouts or quota limits could occur  
    - Missing or malformed document content may result in incomplete answers  

- **Format Success Response**  
  - Type: Code node  
  - Role: Builds a success response JSON including the AI answer, confidence score, session info, and metadata  
  - Configuration:  
    - Reads the AI answer  
    - Calculates a confidence score based on answer length and presence of keywords ("specifically", "according to")  
    - Caps confidence at 1.0  
    - Includes session ID, question, document URL, processing time, and credits used  
  - Key Expressions:  
    ```js
    const answer = $json.answer;
    const request = $node['Validate Request'].json;
    let confidence = 0.8;
    if (answer.length > 100) confidence += 0.1;
    if (answer.toLowerCase().includes('specifically') || answer.toLowerCase().includes('according to')) confidence += 0.1;
    confidence = Math.min(confidence, 1.0);
    return [{
      json: {
        success: true,
        data: { answer, confidence, sessionId: request.sessionId, documentUrl: request.documentUrl, question: request.question },
        metadata: { processedAt: new Date().toISOString(), responseTime: Date.now() - new Date(request.timestamp).getTime(), creditsUsed: 1 }
      }
    }];
    ```  
  - Input: AI response JSON  
  - Output: Formatted success JSON response  
  - Edge Cases: Empty or null answers may reduce confidence; unexpected input structure may cause errors  

#### 2.4 Response Formatting and Delivery

**Overview:**  
Sends the final JSON response (either success or error) back to the client via the webhook response node.

**Nodes Involved:**  
- Send Response

**Node Details:**  

- **Send Response**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP response to the original API caller  
  - Configuration:  
    - Response content type: `application/json`  
    - Response body: Serialized JSON from previous node (`Format Success Response` or `Format Error Response`)  
  - Input: JSON response object  
  - Output: HTTP response  
  - Edge Cases: Large responses might exceed limits; network issues could interrupt response delivery  

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                   | Input Node(s)       | Output Node(s)            | Sticky Note                                                                                   |
|-------------------------|---------------------------|---------------------------------|---------------------|---------------------------|-----------------------------------------------------------------------------------------------|
| API Overview            | Sticky Note               | Documentation overview           | —                   | —                         | ## 🤖 Document Q&A API RESTful service for document intelligence: webhook, AI, JSON response |
| Webhook                 | Webhook                   | Entry point / API endpoint       | —                   | Validate Request          | API endpoint for document Q&A                                                                |
| Validate Request        | Code                      | Validate and prepare input       | Webhook             | Valid Request?            | Validate and prepare request                                                                 |
| Valid Request?          | If                        | Branch on request validity       | Validate Request     | PDF Vector - Ask Question, Format Error Response |                                                                                               |
| PDF Vector - Ask Question| PDF Vector AI node        | AI document question answering   | Valid Request? (true)| Format Success Response   | Get answer from document                                                                     |
| Format Success Response | Code                      | Format successful API response   | PDF Vector           | Send Response             | Prepare successful response                                                                  |
| Format Error Response   | Code                      | Format error API response        | Valid Request? (false)| Send Response             | Prepare error response                                                                       |
| Send Response           | Respond to Webhook        | Return JSON response to client   | Format Success Response, Format Error Response | —                         | Send API response                                                                            |
| Request Format          | Sticky Note               | Documentation on API request     | —                   | —                         | ## 📥 API Request POST to /document-qa with question, maxTokens, file                        |
| Q&A Processing          | Sticky Note               | Documentation on AI usage        | —                   | —                         | ## 🔍 AI Processing PDF Vector parses doc, finds sections, answers, cites                     |
| Response Format         | Sticky Note               | Documentation on API response    | —                   | —                         | ## 📤 API Response JSON with success, answer, sources, confidence                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook`  
   - HTTP Method: POST  
   - Path: `doc-qa`  
   - Save credentials if prompted  
   - This node will receive requests to the `/doc-qa` endpoint  

2. **Add a Code Node for Validation**  
   - Name: `Validate Request`  
   - Type: Code (JavaScript)  
   - Connect `Webhook` main output to this node’s input  
   - Paste the following JS code:  
     ```js
     const body = $input.first().json.body;
     const errors = [];
     if (!body.documentUrl && !body.documentId) errors.push('Either documentUrl or documentId is required');
     if (!body.question) errors.push('Question is required');
     const sessionId = body.sessionId || `session-${Date.now()}`;
     return [{
       json: {
         ...body,
         sessionId,
         valid: errors.length === 0,
         errors,
         timestamp: new Date().toISOString()
       }
     }];
     ```  
   - Set node version to 2 if available for better execution environment  

3. **Add an If Node to Branch on Validity**  
   - Name: `Valid Request?`  
   - Type: If  
   - Connect `Validate Request` main output to this node  
   - Condition: Boolean check `$json.valid === true`  
   - Set two outputs:  
     - Output 1 (true): Valid requests  
     - Output 2 (false): Invalid requests  

4. **Add PDF Vector Node for AI Processing**  
   - Name: `PDF Vector - Ask Question`  
   - Type: PDF Vector (requires PDF Vector credentials)  
   - Connect `Valid Request?` output 1 (true) to this node  
   - Configure:  
     - Resource: Document  
     - Operation: Ask  
     - Input Type: URL  
     - URL: `={{ $json.documentUrl }}`  
     - Prompt: `Answer the following question about this document or image: {{ $json.question }}`  
   - Ensure credentials for PDF Vector / OpenAI GPT-4 are configured  

5. **Add Code Node to Format Success Response**  
   - Name: `Format Success Response`  
   - Type: Code (JavaScript)  
   - Connect `PDF Vector - Ask Question` main output here  
   - Use this JS code:  
     ```js
     const answer = $json.answer;
     const request = $node['Validate Request'].json;
     let confidence = 0.8;
     if (answer.length > 100) confidence += 0.1;
     if (answer.toLowerCase().includes('specifically') || answer.toLowerCase().includes('according to')) confidence += 0.1;
     confidence = Math.min(confidence, 1.0);
     return [{
       json: {
         success: true,
         data: {
           answer,
           confidence,
           sessionId: request.sessionId,
           documentUrl: request.documentUrl,
           question: request.question
         },
         metadata: {
           processedAt: new Date().toISOString(),
           responseTime: Date.now() - new Date(request.timestamp).getTime(),
           creditsUsed: 1
         }
       }
     }];
     ```  
   - Set node version 2 for best compatibility  

6. **Add Code Node to Format Error Response**  
   - Name: `Format Error Response`  
   - Type: Code (JavaScript)  
   - Connect `Valid Request?` output 2 (false) here  
   - Use this JS code:  
     ```js
     const errors = $json.errors || ['An error occurred processing your request'];
     return [{
       json: {
         success: false,
         errors,
         message: 'Invalid request',
         timestamp: new Date().toISOString()
       }
     }];
     ```  
   - Use version 2 if available  

7. **Add Respond to Webhook Node**  
   - Name: `Send Response`  
   - Type: Respond to Webhook  
   - Connect both `Format Success Response` and `Format Error Response` outputs to this node  
   - Configure response:  
     - Respond With: JSON  
     - Response Body: `={{ JSON.stringify($json) }}`  
     - Response Headers: Content-Type: application/json  

8. **Verify Credentials**  
   - Ensure OpenAI or PDF Vector API credentials are configured for the PDF Vector node  
   - No additional credentials required for webhook or code nodes  

9. **Test the API**  
   - Send POST requests to `/webhook/doc-qa` with JSON body containing:  
     ```json
     {
       "documentUrl": "https://example.com/sample.pdf",
       "question": "What is the main topic of the document?"
     }
     ```  
   - Expect a JSON response with `success`, `answer`, `confidence`, and metadata  

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow demonstrates sub-second response times for document Q&A APIs using vector search. | Workflow sticky note “API Overview”                                                                 |
| The AI processing block uses GPT-4 via PDF Vector integration for document understanding.         | Sticky note “Q&A Processing”                                                                        |
| API request format expects a POST to `/document-qa` with JSON body including question and file.  | Sticky note “Request Format”                                                                         |
| API response includes success boolean, answer, sources, and confidence score.                    | Sticky note “Response Format”                                                                        |
| For enhanced security and production readiness, consider adding authentication and rate limiting.| General best practice not included in this workflow                                                  |
| PDF Vector node requires valid API credentials and internet access to external document URLs.    | Refer to n8n documentation on PDF Vector node and credential setup                                  |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.