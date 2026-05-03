Product Ingredient Safety Analyzer with Google Gemini via WhatsApp

https://n8nworkflows.xyz/workflows/product-ingredient-safety-analyzer-with-google-gemini-via-whatsapp-9098


# Product Ingredient Safety Analyzer with Google Gemini via WhatsApp

### 1. Workflow Overview

This workflow, titled **Product Ingredient Safety Analyzer with Google Gemini via WhatsApp**, enables users to send product images or text descriptions via WhatsApp and receive an AI-powered safety analysis of ingredients. It is designed to assess the safety of ingredients in various product categories such as food, cosmetics, pharmaceuticals, personal care, and household items.

The workflow is logically divided into two main processing branches based on the incoming WhatsApp message type:

- **1.1 Image Branch:** Handles image messages containing product photos. It downloads the image, extracts text via OCR using Google Document AI, analyzes the extracted ingredient list using Google Gemini AI, and sends the safety analysis back to the user.

- **1.2 Text Branch:** Handles text messages containing product names, brands, or ingredient lists. It sends the text directly to Google Gemini AI for analysis and returns the results.

Supporting this primary logic are nodes managing WhatsApp triggers, routing, media handling, AI interaction, and message sending.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing

- **Overview:**  
  Captures incoming WhatsApp messages and routes them according to message type (image or text).

- **Nodes Involved:**  
  - WhatsApp Trigger  
  - Route By Message Type

- **Node Details:**

  - **WhatsApp Trigger**  
    - *Type:* WhatsApp Trigger  
    - *Role:* Listens for new WhatsApp messages using WhatsApp Business API credentials.  
    - *Config:* Listens for "messages" updates only.  
    - *Inputs:* Webhook invocations from WhatsApp.  
    - *Outputs:* Passes message JSON to next node.  
    - *Failures:* Credential errors, webhook setup errors, WhatsApp API downtime.  
    - *Notes:* Requires proper WhatsApp Business API credentials.

  - **Route By Message Type (Switch Node)**  
    - *Type:* Switch  
    - *Role:* Routes flow based on WhatsApp message type (`image` or `text`).  
    - *Config:*  
      - Output 0: If message type equals "image"  
      - Output 1: If message type equals "text"  
    - *Inputs:* Receives full WhatsApp message JSON from trigger.  
    - *Outputs:*  
      - Output 0 → Image processing branch  
      - Output 1 → Text processing branch  
    - *Failures:* Expression errors if message type missing or malformed.  
    - *Notes:* Loose type validation enabled to handle variations.

#### 2.2 Image Processing Branch

- **Overview:**  
  Processes product images sent via WhatsApp: retrieves media URL, downloads the image, converts to base64, extracts text via OCR, analyzes ingredients via AI, then sends analysis back to user.

- **Nodes Involved:**  
  - Get Image Media URL  
  - Download Image File  
  - Convert Image to Base64  
  - Extract Text via OCR  
  - Analyze Image Ingredients (AI Agent)  
  - Gemini Model (Image Branch)  
  - JSON Parser (Image)  
  - Send Analysis of Image

- **Node Details:**

  - **Get Image Media URL**  
    - *Type:* WhatsApp API (media)  
    - *Role:* Retrieves the URL for the image media from WhatsApp servers.  
    - *Config:* Uses media ID from incoming WhatsApp message JSON (`messages[0].image.id`).  
    - *Credentials:* WhatsApp API credentials required.  
    - *Failures:* Invalid media ID, expired media, API errors.

  - **Download Image File**  
    - *Type:* HTTP Request  
    - *Role:* Downloads image file from the media URL.  
    - *Config:* Uses URL from previous node; expects binary file response.  
    - *Credentials:* WhatsApp API credentials for authentication.  
    - *Failures:* Network errors, URL expiry, invalid response format.

  - **Convert Image to Base64**  
    - *Type:* Extract From File  
    - *Role:* Converts downloaded binary image to base64 string for OCR input.  
    - *Config:* Converts binary property `data` to base64, stores in `data1` JSON property.  
    - *Failures:* Binary data missing or corrupt.

  - **Extract Text via OCR**  
    - *Type:* HTTP Request  
    - *Role:* Sends base64 image to Google Document AI processor for OCR text extraction.  
    - *Config:*  
      - POST to Google Document AI endpoint with base64 content and MIME type from WhatsApp message.  
      - Uses Google Service Account credentials.  
    - *Outputs:* OCR text accessible via `$json.document.text`.  
    - *Failures:* Credentials errors, API quota, incorrect processor ID, malformed request.

  - **Analyze Image Ingredients (Langchain Agent)**  
    - *Type:* Langchain Agent (Custom AI agent)  
    - *Role:* Analyzes OCR text and optional image caption to classify ingredient safety.  
    - *Config:*  
      - System message instructs AI to return structured JSON with `is_ingredients` and `message`.  
      - Input includes user message (image caption) and OCR text.  
      - Enforces strict JSON response format.  
    - *Failures:* AI model timeout, malformed AI response, invalid JSON output.

  - **Gemini Model (Image Branch)**  
    - *Type:* Langchain Google Gemini LM Chat  
    - *Role:* Executes underlying Google Gemini language model call supporting the agent.  
    - *Credentials:* Google Gemini API credentials required.  
    - *Failures:* API quota limits, credential issues, network failures.

  - **JSON Parser (Image)**  
    - *Type:* Langchain Output Parser Structured  
    - *Role:* Parses AI JSON response ensuring it matches expected schema (`is_ingredients`, `message`).  
    - *Failures:* Parsing errors if AI response invalid JSON.

  - **Send Analysis of Image**  
    - *Type:* WhatsApp API (send message)  
    - *Role:* Sends the AI analysis message back to the WhatsApp user who sent the image.  
    - *Config:* Uses recipient phone number from trigger contact info; sends text body from AI output.  
    - *Credentials:* WhatsApp API credentials required.  
    - *Failures:* Invalid phone number, API errors, message throttling.

#### 2.3 Text Processing Branch

- **Overview:**  
  Processes text messages containing product information by sending text to AI for safety analysis and returning the response.

- **Nodes Involved:**  
  - Analyze Text Query (AI Agent)  
  - Gemini Model (Text Branch)  
  - JSON Parser (Text)  
  - Send Analysis of Text

- **Node Details:**

  - **Analyze Text Query (Langchain Agent)**  
    - *Type:* Langchain Agent (Custom AI agent)  
    - *Role:* Analyzes text message content for product ingredient safety and conversational support.  
    - *Config:*  
      - System prompt includes detailed instructions on JSON response format and analysis criteria for multiple product categories.  
      - Input is the text body from WhatsApp message (`messages[0].text.body`).  
    - *Failures:* AI timeout, incomplete responses, invalid JSON output.

  - **Gemini Model (Text Branch)**  
    - *Type:* Langchain Google Gemini LM Chat  
    - *Role:* Executes underlying Google Gemini language model call for text analysis.  
    - *Credentials:* Google Gemini API credentials.  
    - *Failures:* API errors, quota limits.

  - **JSON Parser (Text)**  
    - *Type:* Langchain Output Parser Structured  
    - *Role:* Parses AI JSON response according to schema (`is_ingredients`, `message`).  
    - *Failures:* JSON parsing errors.

  - **Send Analysis of Text**  
    - *Type:* WhatsApp API (send message)  
    - *Role:* Sends AI-generated safety analysis message back to the user.  
    - *Config:* Uses phone number from WhatsApp trigger contact info; sends AI output message.  
    - *Credentials:* WhatsApp API credentials.  
    - *Failures:* Phone number issues, API errors.

#### 2.4 Documentation & Notes

- **Sticky Note**  
  - Contains a comprehensive summary of the workflow structure, credential requirements, node roles, and AI processing guidelines.  
  - Useful for maintenance, onboarding, and audit.  
  - Covers both image and text branches and key system messages.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                         | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                         |
|-------------------------|------------------------------------|---------------------------------------|----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| WhatsApp Trigger        | WhatsApp Trigger                   | Receive WhatsApp messages              | (Webhook)                  | Route By Message Type          | See sticky note covering overall workflow structure                                              |
| Route By Message Type   | Switch                            | Route messages by type (image/text)   | WhatsApp Trigger           | Get Image Media URL, Analyze Text Query | See sticky note covering overall workflow structure                                          |
| Get Image Media URL     | WhatsApp API (media)              | Retrieve URL for image media           | Route By Message Type (image) | Download Image File           | See sticky note covering overall workflow structure                                              |
| Download Image File     | HTTP Request                     | Download image file from URL           | Get Image Media URL        | Convert Image to Base64        | See sticky note covering overall workflow structure                                              |
| Convert Image to Base64 | Extract From File                | Convert binary image to base64 string | Download Image File        | Extract Text via OCR           | See sticky note covering overall workflow structure                                              |
| Extract Text via OCR    | HTTP Request                    | OCR text extraction via Google Document AI | Convert Image to Base64   | Analyze Image Ingredients      | See sticky note covering overall workflow structure                                              |
| Analyze Image Ingredients | Langchain Agent                | Analyze OCR text for ingredient safety| Extract Text via OCR       | Send Analysis of Image         | See sticky note covering overall workflow structure                                              |
| Gemini Model (Image Branch) | Langchain LM Chat Google Gemini | Google Gemini LM call for image branch | Analyze Image Ingredients  | Analyze Image Ingredients (loopback) | See sticky note covering overall workflow structure                                          |
| JSON Parser (Image)     | Langchain Output Parser Structured | Parse AI JSON output for image branch | Gemini Model (Image Branch)| Analyze Image Ingredients      | See sticky note covering overall workflow structure                                              |
| Send Analysis of Image  | WhatsApp API (send message)      | Send analysis message via WhatsApp    | Analyze Image Ingredients  | (End)                        | See sticky note covering overall workflow structure                                              |
| Analyze Text Query      | Langchain Agent                  | Analyze text messages for safety      | Route By Message Type (text) | Send Analysis of Text        | See sticky note covering overall workflow structure                                              |
| Gemini Model (Text Branch) | Langchain LM Chat Google Gemini | Google Gemini LM call for text branch | Analyze Text Query         | Analyze Text Query (loopback) | See sticky note covering overall workflow structure                                              |
| JSON Parser (Text)      | Langchain Output Parser Structured | Parse AI JSON output for text branch  | Gemini Model (Text Branch) | Analyze Text Query             | See sticky note covering overall workflow structure                                              |
| Send Analysis of Text   | WhatsApp API (send message)      | Send analysis message via WhatsApp    | Analyze Text Query         | (End)                        | See sticky note covering overall workflow structure                                              |
| Sticky Note             | Sticky Note                      | Documentation and overview             | -                          | -                             | Comprehensive workflow summary including flow, AI instructions, and credentials needed           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure to listen for "messages" update events.  
   - Assign WhatsApp Trigger API credentials.  
   - This node will receive incoming WhatsApp messages.

2. **Create Switch Node "Route By Message Type"**  
   - Type: Switch  
   - Add two rules:  
     - Rule 1 (Output 0): If `{{$json.messages[0].type}}` equals `"image"`  
     - Rule 2 (Output 1): If `{{$json.messages[0].type}}` equals `"text"`  
   - Connect WhatsApp Trigger output to this switch node input.

3. **Image Branch Setup:**

   3.1 **Get Image Media URL**  
       - Type: WhatsApp API (media)  
       - Operation: Get media URL  
       - Media ID: `{{$json.messages[0].image.id}}`  
       - Assign WhatsApp API credentials.  
       - Connect Output 0 of Switch node to this node.

   3.2 **Download Image File**  
       - Type: HTTP Request  
       - Method: GET  
       - URL: `{{$json.url}}` (from previous node)  
       - Response format: File (binary)  
       - Authentication: Use WhatsApp API credentials.  
       - Connect from Get Image Media URL.

   3.3 **Convert Image to Base64**  
       - Type: Extract From File  
       - Operation: Binary to Property  
       - Binary Property Name: `data` (default binary data key)  
       - Destination Key: `data1` (store base64 string here)  
       - Connect from Download Image File.

   3.4 **Extract Text via OCR (Google Document AI)**  
       - Type: HTTP Request  
       - Method: POST  
       - URL: `https://us-documentai.googleapis.com/v1/projects/YOUR_GOOGLE_PROJECT_ID/locations/us/processors/YOUR_PROCESSOR_ID:process`  
       - Body (JSON):  
         ```json
         {
           "rawDocument": {
             "content": "{{ $('Convert Image to Base64').item.json.data1 }}",
             "mimeType": "{{ $('WhatsApp Trigger').item.json.messages[0].image.mime_type }}"
           }
         }
         ```  
       - Authentication: Google Service Account credentials for Document AI.  
       - Connect from Convert Image to Base64.

   3.5 **Analyze Image Ingredients (Langchain Agent)**  
       - Type: Langchain Agent  
       - Text:  
         ```
         user message :- {{ $('WhatsApp Trigger').item.json.messages[0].image.caption || "null" }}

         ocr :-  {{ $json.document.text }}
         ```  
       - System message: Use detailed instructions for product ingredient analysis with strict JSON response format (see node description).  
       - Connect from Extract Text via OCR.

   3.6 **Gemini Model (Image Branch)**  
       - Type: Langchain Google Gemini LM Chat  
       - Credentials: Google Gemini API credentials.  
       - Connect Langchain Agent to this node as AI language model.

   3.7 **JSON Parser (Image)**  
       - Type: Langchain Output Parser Structured  
       - JSON Schema Example:  
         ```json
         {
           "is_ingredients": 0,
           "message": "your response message here"
         }
         ```  
       - Connect Gemini Model output to this parser node.

   3.8 **Send Analysis of Image**  
       - Type: WhatsApp API (send message)  
       - Operation: Send  
       - Text Body: `{{$json.output.message}}` (from AI output)  
       - Recipient Phone Number: `+{{$('WhatsApp Trigger').item.json.contacts[0].wa_id}}`  
       - Phone Number ID: Your WhatsApp Phone Number ID  
       - Assign WhatsApp API credentials.  
       - Connect from Analyze Image Ingredients node output.

4. **Text Branch Setup:**

   4.1 **Analyze Text Query (Langchain Agent)**  
       - Type: Langchain Agent  
       - Text: `{{$json.messages[0].text.body}}`  
       - System message: Use detailed product safety analyzer instructions for text input with required JSON response format (see node description).  
       - Connect Output 1 of Switch node here.

   4.2 **Gemini Model (Text Branch)**  
       - Type: Langchain Google Gemini LM Chat  
       - Credentials: Google Gemini API credentials.  
       - Connect Langchain Agent to this node as AI language model.

   4.3 **JSON Parser (Text)**  
       - Type: Langchain Output Parser Structured  
       - JSON Schema Example same as image branch.  
       - Connect Gemini Model output here.

   4.4 **Send Analysis of Text**  
       - Type: WhatsApp API (send message)  
       - Operation: Send  
       - Text Body: `{{$json.output.message}}`  
       - Recipient Phone Number: `+{{$('WhatsApp Trigger').item.json.contacts[0].wa_id}}`  
       - Phone Number ID: Your WhatsApp Phone Number ID  
       - Assign WhatsApp API credentials.  
       - Connect from Analyze Text Query node output.

5. **Add Documentation Sticky Note**  
   - Add a Sticky Note node describing the overall workflow, AI instructions, credentials needed, and flow structure for future reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 📋 Product Ingredient Safety Analyzer with AI via WhatsApp. The workflow splits processing into image and text branches. The switch node routes WhatsApp messages by type. Images trigger OCR via Google Document AI; text messages are analyzed directly. Both use Google Gemini language model for AI safety analysis with strict JSON output format. Responses are sent back via WhatsApp API. Credentials needed: WhatsApp Business API, Google Service Account for Document AI, Google Gemini API. See sticky note in workflow for full details.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow internal sticky note for documentation                                                    |
| Google Document AI setup requires creating a processor in your GCP project and enabling the Document AI API. Replace `YOUR_GOOGLE_PROJECT_ID` and `YOUR_PROCESSOR_ID` in the OCR node URL with your actual project and processor identifiers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | https://cloud.google.com/document-ai/docs/quickstarts                                              |
| Google Gemini API credentials must be created and linked properly with n8n Langchain nodes for AI interaction. Ensure quota limits and billing are configured in Google Cloud.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | https://developers.generativeai.google/api/gemini/guides/getting-started                           |
| WhatsApp Business API credentials require WhatsApp Business account approval and phone number registration. Phone Number ID must match your WhatsApp Business phone number.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | https://developers.facebook.com/docs/whatsapp/getting-started                                      |
| AI response JSON format is critical for downstream parsing. If AI returns invalid JSON, the JSON parser node will fail. Monitor AI output logs and adjust model prompts if necessary.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |                                                                                                    |
| Emoji usage and WhatsApp formatting (*bold*, _italic_) are embedded in AI system prompts for user-friendly messages.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                    |

---

**Disclaimer:** This content is generated exclusively from an n8n automation workflow. All data processed is legal and public, respecting content policies. No illegal or protected elements are included.