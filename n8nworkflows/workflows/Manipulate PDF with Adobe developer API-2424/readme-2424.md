Manipulate PDF with Adobe developer API

https://n8nworkflows.xyz/workflows/manipulate-pdf-with-adobe-developer-api-2424


# Manipulate PDF with Adobe developer API

### 1. Workflow Overview

This workflow serves as a comprehensive, generic wrapper to interact with the Adobe PDF Services API, enabling various PDF manipulations such as splitting, combining, OCR, page operations, and content extraction. It encapsulates the multi-step Adobe API process, which includes authentication, asset registration, file upload, query execution, result polling, and downloading the transformed PDF or related data.

**Target Use Cases:**  
- Automating PDF transformations using Adobe’s API from other workflows or manual triggers.  
- Extracting clean PDF content for AI and Retrieval-Augmented Generation (RAG) systems, e.g., extracting tables as images for improved AI recognition.  
- Developers needing a reusable, modular integration with Adobe PDF Services.

**Logical Blocks:**  
- **1.1 Input Reception & Setup:** Triggering the workflow and preparing input data and query parameters.  
- **1.2 Authentication:** Obtaining a temporary access token from Adobe for API calls.  
- **1.3 Asset Registration & Upload:** Registering a new PDF asset and uploading the file to Adobe’s cloud storage.  
- **1.4 Query Processing:** Sending the transformation request to Adobe and managing asynchronous processing.  
- **1.5 Result Handling:** Polling for completion, downloading the output, and forwarding results back.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Setup  
**Overview:**  
This block receives the initial trigger to start the workflow and prepares the Adobe API query parameters alongside input PDF data.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Load a test pdf file (Dropbox node)  
- Adobe API Query (Set node)  
- Query + File (Merge node)  

**Node Details:**  
- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual testing.  
  - Inputs: None  
  - Outputs: Triggers the "Load a test pdf file" and "Adobe API Query" nodes.  
  - Failure modes: None significant; manual trigger.  

- **Load a test pdf file**  
  - Type: Dropbox (Download operation)  
  - Role: Downloads a test PDF file from Dropbox for processing.  
  - Configuration: Path set to a specific PDF in Dropbox, OAuth2 authentication used.  
  - Inputs: Triggered by manual trigger node.  
  - Outputs: Binary PDF data.  
  - Failure modes: OAuth2 token expiry, file not found, network errors.  

- **Adobe API Query**  
  - Type: Set node  
  - Role: Prepares the JSON payload and the target endpoint for the Adobe API request.  
  - Configuration:  
    - `endpoint`: set to `"extractpdf"` (example use case).  
    - `json_payload`: JSON object specifying what to extract, here “tables” and “text”.  
  - Inputs: None directly (manual trigger)  
  - Outputs: Passes JSON data downstream.  
  - Failure modes: Expression evaluation failure if JSON malformed.  

- **Query + File**  
  - Type: Merge (combine by position)  
  - Role: Combines the query parameters and the PDF file binary data into one data object for subsequent processing.  
  - Inputs: From "Adobe API Query" and "Load a test pdf file" nodes.  
  - Outputs: Merged JSON + binary object.  
  - Failure modes: Mismatched input positions or missing inputs.

---

#### 1.2 Authentication  
**Overview:**  
Obtains an OAuth access token from Adobe necessary for all subsequent API calls.

**Nodes Involved:**  
- Authenticartion (get token) (HTTP Request)  

**Node Details:**  
- **Authenticartion (get token)**  
  - Type: HTTP Request (POST)  
  - Role: Exchanges client credentials for a temporary access token.  
  - Configuration:  
    - URL: `https://pdf-services.adobe.io/token`  
    - Content-Type: `application/x-www-form-urlencoded`  
    - Uses a custom HTTP credential containing client_id and client_secret in the body.  
  - Inputs: Triggered after input preparation.  
  - Outputs: JSON object containing `access_token`.  
  - Failure modes: Invalid credentials, network errors, token expiration.  
  - Credential note: Requires a "Custom Auth" credential with body parameters client_id and client_secret.  

---

#### 1.3 Asset Registration & Upload  
**Overview:**  
Registers a new asset to host the PDF on Adobe servers and uploads the PDF binary data to this asset.

**Nodes Involved:**  
- Create Asset (HTTP Request)  
- Query + File + Asset information (Merge node)  
- Upload PDF File (asset) (HTTP Request)  

**Node Details:**  
- **Create Asset**  
  - Type: HTTP Request (POST)  
  - Role: Registers a new asset with Adobe by sending a POST request with mediaType "application/pdf".  
  - Configuration:  
    - URL: `https://pdf-services.adobe.io/assets`  
    - Auth: Header with `Authorization: Bearer {{access_token}}`  
    - Sends JSON body `{ "mediaType": "application/pdf" }`  
  - Inputs: Access token from Authentication node.  
  - Outputs: JSON response containing asset metadata including assetID and uploadUri.  
  - Failure modes: Auth errors, rate limits, malformed requests.  

- **Query + File + Asset information**  
  - Type: Merge (combine by position)  
  - Role: Combines the original query + file data with the newly created asset info.  
  - Inputs: From "Query + File" and "Create Asset" nodes.  
  - Outputs: Merged object including assetID and uploadUri for upload.  
  - Failure modes: Input mismatch, missing asset info.  

- **Upload PDF File (asset)**  
  - Type: HTTP Request (PUT)  
  - Role: Uploads the PDF binary data to the asset’s upload URI.  
  - Configuration:  
    - URL: dynamic, based on `uploadUri` from asset registration.  
    - Content-Type: binaryData  
    - Input data field: binary PDF file data from merged input.  
  - Inputs: Merged data containing uploadUri and binary file.  
  - Outputs: Confirmation of upload success.  
  - Failure modes: Upload failures, network issues, invalid uploadUri.

---

#### 1.4 Query Processing  
**Overview:**  
Submits the requested PDF transformation query to Adobe, waits for processing, and polls for completion.

**Nodes Involved:**  
- Process Query (HTTP Request)  
- Wait 5 second (Wait node)  
- Try to download the result (HTTP Request)  
- Switch (Switch node)  

**Node Details:**  
- **Process Query**  
  - Type: HTTP Request (POST)  
  - Role: Calls the Adobe API operation endpoint for the requested transformation.  
  - Configuration:  
    - URL: `https://pdf-services.adobe.io/operation/{{endpoint}}` where `endpoint` is dynamic.  
    - Body: JSON combining `assetID` and the original `json_payload`.  
    - Auth: Bearer token header.  
    - Specifies full HTTP response to get headers (used for polling).  
  - Inputs: Merged query + file + asset data.  
  - Outputs: HTTP response with 'location' header for polling result URL.  
  - Failure modes: Auth errors, invalid payload, server errors.  

- **Wait 5 second**  
  - Type: Wait node  
  - Role: Delay between polling attempts to avoid overloading Adobe.  
  - Configuration: 5 seconds delay.  
  - Inputs: Chain from Process Query or Switch node.  
  - Outputs: Triggers next polling attempt.  
  - Failure modes: None likely.  

- **Try to download the result**  
  - Type: HTTP Request (GET)  
  - Role: Polls the URL in 'location' header for processing status or final result.  
  - Configuration:  
    - URL: dynamic from `Process Query` response header location.  
    - Auth: Bearer token header.  
  - Inputs: After wait node.  
  - Outputs: JSON containing status and possibly download URL.  
  - Failure modes: 404 if not ready, network errors, token expiry.  

- **Switch**  
  - Type: Switch node  
  - Role: Routes output based on status field (`in progress`, `failed`, or else).  
  - Configuration: Checks JSON `status` field to determine next step.  
  - Inputs: From download attempt node.  
  - Outputs:  
    - If "in progress": loops back to Wait 5 second (poll again).  
    - If "failed" or other: forwards response to origin workflow (end).  
  - Failure modes: Missing or unexpected status fields.

---

#### 1.5 Result Handling  
**Overview:**  
Forwards the final output or error back to the calling workflow or user.

**Nodes Involved:**  
- Forward response to origin workflow (Set node)  

**Node Details:**  
- **Forward response to origin workflow**  
  - Type: Set node  
  - Role: Passes through the final JSON data from Adobe API to the outer workflow or caller.  
  - Configuration: Includes all fields as is without modification.  
  - Inputs: From Switch node on failure or success completion.  
  - Outputs: Final output of the workflow.  
  - Failure modes: None significant; pass-through node.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                        | Input Node(s)                            | Output Node(s)                          | Sticky Note                                                                                                                       |
|------------------------------|---------------------|-------------------------------------|-----------------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual workflow start trigger        | —                                       | Load a test pdf file, Adobe API Query  |                                                                                                                                  |
| Load a test pdf file          | Dropbox             | Downloads test PDF from Dropbox      | When clicking ‘Test workflow’            | Query + File                          |                                                                                                                                  |
| Adobe API Query               | Set                 | Prepares endpoint and payload JSON  | When clicking ‘Test workflow’            | Query + File                          |                                                                                                                                  |
| Query + File                 | Merge                | Merges query and PDF file data       | Adobe API Query, Load a test pdf file    | Authenticartion (get token), Query + File + Asset information |                                                                                                                                  |
| Authenticartion (get token)   | HTTP Request        | Retrieves Adobe API access token     | Query + File, Execute Workflow Trigger   | Create Asset                         | See Sticky Note3: Custom Auth credential with client_id and client_secret in body.                                               |
| Create Asset                 | HTTP Request         | Registers new PDF asset on Adobe     | Authenticartion (get token)               | Query + File + Asset information      |                                                                                                                                  |
| Query + File + Asset information | Merge             | Combines query, file, and asset info | Query + File, Create Asset                | Upload PDF File (asset)               |                                                                                                                                  |
| Upload PDF File (asset)       | HTTP Request        | Uploads PDF binary to Adobe asset    | Query + File + Asset information          | Process Query                        |                                                                                                                                  |
| Process Query                | HTTP Request         | Sends transformation query to Adobe | Upload PDF File (asset)                    | Wait 5 second                      |                                                                                                                                  |
| Wait 5 second               | Wait                 | Delays for 5 seconds between polls  | Process Query, Switch                      | Try to download the result            | Sticky Note2: "Wait for file do be processed"                                                                                     |
| Try to download the result    | HTTP Request        | Polls for query result or status    | Wait 5 second                             | Switch                              |                                                                                                                                  |
| Switch                      | Switch               | Routes based on processing status   | Try to download the result                 | Wait 5 second (if in progress), Forward response to origin workflow (if failed or complete) |                                                                                                                                  |
| Forward response to origin workflow | Set            | Passes final result to caller        | Switch                                   | —                                  |                                                                                                                                  |
| Execute Workflow Trigger      | Execute Workflow Trigger | Allows external workflows to call this wrapper | —                                 | Authenticartion (get token), Query + File + Asset information |                                                                                                                                  |
| Sticky Note                  | Sticky Note          | Documentation and guidance           | —                                       | —                                  | See section 5 for full sticky note contents                                                                                        |
| Sticky Note1                 | Sticky Note          | Development testing note             | —                                       | —                                  |                                                                                                                                  |
| Sticky Note2                 | Sticky Note          | Explains wait node                   | —                                       | —                                  | "Wait for file do be processed"                                                                                                  |
| Sticky Note3                 | Sticky Note          | Details credential for token request| —                                       | —                                  | Custom Auth credential example with client_id and client_secret body                                                             |
| Sticky Note4                 | Sticky Note          | Details credential for API queries  | —                                       | —                                  | Header Auth credential example with X-API-Key                                                                                    |
| Sticky Note5                 | Sticky Note          | Explains workflow input format      | —                                       | —                                  | Examples of endpoint and json_payload formats                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Node type: Manual Trigger  
   - Purpose: Start workflow manually for testing.

2. **Add Dropbox node to download test PDF:**  
   - Node type: Dropbox  
   - Operation: Download  
   - Path: Set to desired PDF file path in Dropbox.  
   - Authentication: OAuth2 with Dropbox credentials.

3. **Add Set node to prepare Adobe API query:**  
   - Node type: Set  
   - Fields:  
     - `endpoint`: string, e.g., `"extractpdf"`  
     - `json_payload`: JSON object with desired extraction params, e.g., `{ "renditionsToExtract": ["tables"], "elementsToExtract": ["text","tables"] }`

4. **Merge node to combine query and PDF file:**  
   - Node type: Merge  
   - Mode: Combine by position (mergeByPosition)  
   - Inputs: Connect outputs of Set node and Dropbox node.

5. **Add HTTP Request node for Adobe Authentication:**  
   - Node type: HTTP Request  
   - Method: POST  
   - URL: `https://pdf-services.adobe.io/token`  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Authentication: Use a **Custom Auth** credential configured as:  
     ```json
     {
       "headers": {
         "Content-Type": "application/x-www-form-urlencoded"
       },
       "body": {
         "client_id": "YOUR_CLIENT_ID",
         "client_secret": "YOUR_CLIENT_SECRET"
       }
     }
     ```
   - Input: Connect from Merge node.

6. **Add HTTP Request node to Create Asset:**  
   - Node type: HTTP Request  
   - Method: POST  
   - URL: `https://pdf-services.adobe.io/assets`  
   - Authentication: Use a **Header Auth** credential with header `Authorization: Bearer {{access_token}}` (access_token from previous node).  
   - Body: JSON `{ "mediaType": "application/pdf" }`  
   - Input: Connect from Authentication node.

7. **Add Merge node to combine query+file with asset info:**  
   - Node type: Merge  
   - Mode: Combine by position  
   - Inputs: From "Query + File" merge node and "Create Asset" node.

8. **Add HTTP Request node to Upload PDF File:**  
   - Node type: HTTP Request  
   - Method: PUT  
   - URL: Dynamic from asset info `uploadUri`  
   - Content-Type: binaryData  
   - Input Data Field: Set to binary PDF file data  
   - Input: Connect from previous Merge node.

9. **Add HTTP Request node to Process Query:**  
   - Node type: HTTP Request  
   - Method: POST  
   - URL: `https://pdf-services.adobe.io/operation/{{endpoint}}` (endpoint from merged data)  
   - Authentication: Header with Bearer token.  
   - Body: JSON merging `assetID` and `json_payload`.  
   - Input: Connect from Upload PDF node.

10. **Add Wait node:**  
    - Node type: Wait  
    - Duration: 5 seconds  
    - Input: Connect from Process Query node.

11. **Add HTTP Request node to Try Download Result:**  
    - Node type: HTTP Request  
    - Method: GET  
    - URL: From `location` header of Process Query response  
    - Authentication: Bearer token header  
    - Input: Connect from Wait node.

12. **Add Switch node:**  
    - Node type: Switch  
    - Check JSON field `status`  
    - Condition branches:  
      - If `in progress`: Connect back to Wait node (loop).  
      - If `failed` or others: Forward output to next node.

13. **Add Set node to Forward response:**  
    - Node type: Set  
    - Purpose: Pass final output downstream or back to calling workflow.  
    - Input: Connect from Switch node.

14. **Optional:** Add Execute Workflow Trigger node if you want this workflow callable from others.

15. **Create Credentials:**  
    - Custom Auth credential for token request with client_id and client_secret in body.  
    - Header Auth credential for other API calls with `X-API-Key` header (same as client_id).  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Adobe API Wrapper workflow documentation and official API docs: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/howtos/ and https://developer.adobe.com/document-services/docs/overview/pdf-extract-api/gettingstarted/                                                                                     | Sticky Note at workflow start                                                                                                                  |
| Credential setup for token request requires creating a Custom Auth credential with client_id and client_secret in the body, Content-Type `application/x-www-form-urlencoded`.                                                                                                                        | Sticky Note3                                                                                                                                                                        |
| Credential setup for all other Adobe API queries requires Header Auth credential with `X-API-Key` header set to client_id.                                                                                                                                                                       | Sticky Note4                                                                                                                                                                        |
| Workflow input expects an object with `endpoint` string, `json_payload` object (excluding assetID), and PDF binary data as input. Examples provided for `splitpdf` and `extractpdf` endpoints.                                                                                                     | Sticky Note5                                                                                                                                                                        |
| Typical use case: extracting tables as images from PDFs to forward to AI systems for improved recognition accuracy.                                                                                                                                                                              | Workflow overview section                                                                                                                                                           |
| The workflow is designed to be used as a sub-workflow via Execute Workflow node or manually triggered for testing.                                                                                                                                                                                | Node "Execute Workflow Trigger" presence                                                                                                                                             |
| Rate limits: Adobe free tier allows up to 500 PDF operations per month.                                                                                                                                                                                                                             | Workflow description                                                                                                                                                                |
| Polling delay is set to 5 seconds to avoid overwhelming Adobe API during processing.                                                                                                                                                                                                               | Sticky Note2                                                                                                                                                                        |

---

This document fully describes the workflow’s logic, node configuration, and integration points to enable seamless reproduction, modification, and error anticipation.