Image-Based Data Extraction API using Gemini AI

https://n8nworkflows.xyz/workflows/image-based-data-extraction-api-using-gemini-ai-3149


# Image-Based Data Extraction API using Gemini AI

### 1. Workflow Overview

This workflow, titled **"Image-Based Data Extraction API using Gemini AI"**, implements a no-code API endpoint designed to extract structured data from images using AI-powered OCR technology. It is tailored for scenarios requiring automated text extraction from various image types such as ID cards, invoices, receipts, business cards, and scanned documents.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives an HTTP GET request containing an image URL and extraction parameters.
- **1.2 Image Retrieval and Conversion:** Downloads the image from the provided URL and converts it into a base64-encoded format suitable for AI processing.
- **1.3 AI Processing with Gemini API:** Sends the base64 image and extraction instructions to the Gemini AI API (Flash Lite model) to perform OCR and extract requested fields.
- **1.4 Output Formatting and Response:** Parses the AI response to isolate the required data fields and returns the structured JSON result to the API caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block exposes a webhook endpoint that accepts incoming API requests. It captures the image URL and extraction requirements from the request body.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (HTTP endpoint)  
    - Configuration:  
      - Path: `data-extractor`  
      - HTTP Method: GET (implied by usage)  
      - Response Mode: `responseNode` (response is sent by a downstream node)  
    - Expressions/Variables: Accesses request body parameters such as `image_url`, `Requirement`, and `properties`.  
    - Input: External HTTP request  
    - Output: Passes request data downstream  
    - Edge Cases:  
      - Missing or invalid `image_url` parameter  
      - Malformed JSON in request body  
      - Unsupported HTTP methods  
    - Notes: This node is the API entry point.

#### 2.2 Image Retrieval and Conversion

- **Overview:**  
  Downloads the image from the URL provided in the webhook request and converts the binary image data into a base64-encoded string for AI processing.

- **Nodes Involved:**  
  - Get image from URL  
  - Transform image to base64

- **Node Details:**

  - **Get image from URL**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: Dynamically set to `{{$json.body.image_url}}` from webhook input  
      - Method: GET (default)  
      - No additional options configured  
    - Input: Receives webhook data  
    - Output: Binary image data  
    - Edge Cases:  
      - Invalid or unreachable URL  
      - Non-image content returned  
      - Timeout or network errors  

  - **Transform image to base64**  
    - Type: Extract From File  
    - Configuration:  
      - Operation: `binaryToProperty` (converts binary data to a JSON property)  
      - Destination Key: `data1`  
      - Encoding: ASCII (base64 encoded string)  
    - Input: Binary image data from previous node  
    - Output: JSON with base64 string under `data1`  
    - Edge Cases:  
      - Binary data missing or corrupted  
      - Encoding errors  

#### 2.3 AI Processing with Gemini API

- **Overview:**  
  Sends the base64-encoded image and extraction instructions to the Gemini AI API (Flash Lite model) to perform OCR and extract the requested structured data fields.

- **Nodes Involved:**  
  - Call Gemini API (Flash Lite) with Image

- **Node Details:**

  - **Call Gemini API (Flash Lite) with Image**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-lite:generateContent`  
      - Method: POST  
      - Body Type: JSON  
      - Body Content:  
        - `contents`: Array with two user roles:  
          1. Inline image data with base64 string (`{{$json.data1}}`) and MIME type `image/jpeg`  
          2. Text prompt `"check this"` (likely a placeholder or trigger)  
        - `systemInstruction`: Text from webhook input `Requirement` field  
        - `generationConfig`: Parameters controlling AI output such as temperature, topK, topP, max tokens, response MIME type (`application/json`), and a dynamic response schema derived from webhook `properties` field  
      - Authentication: Uses predefined Google Palm API credentials named "Gemini API Srinivasan Online"  
    - Input: JSON with base64 image and instructions  
    - Output: AI-generated JSON response containing extracted data candidates  
    - Expressions/Variables:  
      - Uses expressions to dynamically insert base64 image and extraction requirements  
      - Dynamically constructs response schema from input properties  
    - Edge Cases:  
      - API authentication failure  
      - Rate limiting or quota exceeded  
      - Invalid or malformed request body  
      - AI model errors or timeouts  
      - Unexpected response format  
    - Version-specific: Requires n8n version supporting HTTP Request node v4.2 and Google Palm API credentials  
    - Notes: This is the core AI OCR processing step.

#### 2.4 Output Formatting and Response

- **Overview:**  
  Parses the AI response to extract the first candidate's content, converts it from stringified JSON to a JSON object, and sends it back as the API response.

- **Nodes Involved:**  
  - Edit fields to output required data alone  
  - Respond to Webhook

- **Node Details:**

  - **Edit fields to output required data alone**  
    - Type: Set  
    - Configuration:  
      - Assigns a new field `result` with the value parsed from the first candidate's content text:  
        `={{ $json.candidates[0].content.parts[0].text.parseJson() }}`  
      - Removes all other fields by default (only `result` remains)  
    - Input: AI JSON response  
    - Output: JSON with a single `result` field containing structured extracted data  
    - Edge Cases:  
      - Missing or empty `candidates` array  
      - Parsing errors if AI response text is not valid JSON  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration:  
      - Sends the output of the previous node as the HTTP response  
    - Input: Final JSON data to return  
    - Output: HTTP response to the original API caller  
    - Edge Cases:  
      - Network errors while sending response  
      - Large payloads causing timeout  

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                      |
|----------------------------------|---------------------|----------------------------------------|-------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook                          | Webhook             | API entry point, receives request      | External HTTP request    | Get image from URL             | See Sticky Note2 for detailed API endpoint description and use cases                            |
| Get image from URL               | HTTP Request        | Downloads image from provided URL      | Webhook                 | Transform image to base64      |                                                                                                 |
| Transform image to base64        | Extract From File   | Converts binary image to base64 string | Get image from URL      | Call Gemini API (Flash Lite)  |                                                                                                 |
| Call Gemini API (Flash Lite) with Image | HTTP Request        | Sends image and instructions to AI     | Transform image to base64 | Edit fields to output required data alone |                                                                                                 |
| Edit fields to output required data alone | Set                 | Parses AI response and extracts data   | Call Gemini API          | Respond to Webhook             |                                                                                                 |
| Respond to Webhook               | Respond to Webhook  | Sends final JSON response to caller    | Edit fields to output    | External HTTP response         |                                                                                                 |
| Sticky Note                     | Sticky Note         | Sample API call example                 |                         |                               | Contains cURL example for API usage                                                             |
| Sticky Note1                    | Sticky Note         | Sample output example                   |                         |                               | Shows example JSON output                                                                        |
| Sticky Note2                    | Sticky Note         | Workflow overview and API description  |                         |                               | Describes workflow purpose, use cases, and processing steps                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `data-extractor`  
   - Response Mode: `responseNode`  
   - Accepts GET requests with JSON body containing:  
     - `image_url` (string)  
     - `Requirement` (string)  
     - `properties` (JSON schema object)  

2. **Create HTTP Request Node: "Get image from URL"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Expression `{{$json.body.image_url}}` (from webhook input)  
   - No authentication or special headers  
   - Connect Webhook → Get image from URL  

3. **Create Extract From File Node: "Transform image to base64"**  
   - Type: Extract From File  
   - Operation: `binaryToProperty`  
   - Destination Key: `data1`  
   - Encoding: ASCII (base64)  
   - Connect Get image from URL → Transform image to base64  

4. **Create HTTP Request Node: "Call Gemini API (Flash Lite) with Image"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-lite:generateContent`  
   - Authentication: Use Google Palm API credentials (OAuth2 or API key)  
   - Body Content (JSON):  
     ```json
     {
       "contents": [
         {
           "role": "user",
           "parts": [
             {
               "inlineData": {
                 "data": "{{$json.data1}}",
                 "mimeType": "image/jpeg"
               }
             }
           ]
         },
         {
           "role": "user",
           "parts": [
             {
               "text": "check this"
             }
           ]
         }
       ],
       "systemInstruction": {
         "role": "user",
         "parts": [
           {
             "text": "{{ $('Webhook').first().json.body.Requirement }}"
           }
         ]
       },
       "generationConfig": {
         "temperature": 1,
         "topK": 40,
         "topP": 0.95,
         "maxOutputTokens": 8192,
         "responseMimeType": "application/json",
         "responseSchema": {
           "type": "object",
           "properties": {{ $('Webhook').first().json.body.properties.toJsonString() }}
         }
       }
     }
     ```  
   - Connect Transform image to base64 → Call Gemini API (Flash Lite) with Image  

5. **Create Set Node: "Edit fields to output required data alone"**  
   - Type: Set  
   - Remove all fields except one new field:  
     - Name: `result`  
     - Value (expression): `={{ $json.candidates[0].content.parts[0].text.parseJson() }}`  
   - Connect Call Gemini API → Edit fields to output required data alone  

6. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Connect Edit fields to output required data alone → Respond to Webhook  

7. **Add Sticky Notes (Optional for Documentation)**  
   - Add a sticky note near the Webhook node with sample cURL API call (see Sticky Note content)  
   - Add a sticky note near the Respond to Webhook node with sample JSON output  

8. **Credentials Setup**  
   - Configure Google Palm API credentials with valid API key or OAuth2 token for Gemini API access  
   - Assign these credentials to the "Call Gemini API (Flash Lite) with Image" node  

9. **Activate Workflow**  
   - Save and activate the workflow  
   - Test by sending a GET request with JSON body to `/webhook/data-extractor`  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sample API call using cURL: includes image URL, extraction requirement, and JSON schema for expected fields                                                  | See Sticky Note near Webhook node                                                                |
| Sample output JSON demonstrating extracted fields such as PAN Number, Name, Date of Birth, and Valid (boolean)                                              | See Sticky Note near Respond to Webhook node                                                     |
| Workflow converts images to base64 before sending to AI, ensuring compatibility with Gemini API input format                                                | Workflow design detail                                                                           |
| Uses Gemini API Flash Lite model for OCR and structured data extraction                                                                                      | Requires Google Palm API credentials                                                             |
| Supports customizable extraction schema via `properties` JSON parameter in API request                                                                     | Enables flexible field extraction                                                                |
| Suitable for integration with CRM, ERP, document management, and other automation systems                                                                    | Use case guidance                                                                                |
| For more information on Gemini API and Google Palm API, refer to official Google Cloud documentation                                                       | https://cloud.google.com/vertex-ai/docs/generative-ai/models/gemini                             |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.