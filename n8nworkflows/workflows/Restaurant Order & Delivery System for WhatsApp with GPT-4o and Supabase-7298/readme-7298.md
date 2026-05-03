Restaurant Order & Delivery System for WhatsApp with GPT-4o and Supabase

https://n8nworkflows.xyz/workflows/restaurant-order---delivery-system-for-whatsapp-with-gpt-4o-and-supabase-7298


# Restaurant Order & Delivery System for WhatsApp with GPT-4o and Supabase

---

### 1. Workflow Overview

This workflow implements a **Restaurant Order & Delivery System for WhatsApp**, leveraging GPT-4o and Supabase to provide an AI-powered conversational agent for taking and managing customer orders, verifying payments, and facilitating communication between customers and restaurant staff. It processes multi-modal WhatsApp messages (text, images, audio, documents, locations), maintains conversation history, and integrates external APIs to enrich and verify information.

**Target use cases:**
- Receiving and processing customer orders via WhatsApp with natural language understanding.
- Handling delivery or pickup preferences, including address verification via Google Maps.
- Analyzing payment receipts (images, PDFs) for payment confirmation.
- Transcribing voice messages to text for order processing.
- Sending menus, order confirmations, and payment QR codes.
- Managing conversation memory to maintain context.

**Logical blocks:**

- **1.1 Message Reception & Type Detection:** Entry point receiving WhatsApp messages and classifying them by type (text, image, voice, document, location).

- **1.2 Media Processing:** Download and process media files (images, audio, documents), including transcription, OCR, and PDF content extraction.

- **1.3 AI Conversation & Order Handling:** AI agent interacting with customers to take orders, using OpenAI GPT models and storing session context in Supabase (Postgres).

- **1.4 Order Structuring & Formatting:** Parsing AI output into structured order data and formatting it for restaurant staff notification and customer confirmation.

- **1.5 Payment Receipt Verification:** Analyzing submitted payment receipts for authenticity and matching with confirmed orders.

- **1.6 Location Handling:** Using Google Maps API to resolve and confirm delivery addresses.

- **1.7 Outgoing Communication:** Sending messages, images (menu, QR codes), and order confirmations back to customers and to restaurant staff.

- **1.8 Setup & Documentation Notes:** Sticky notes providing setup instructions, workflow overview, and operational guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Message Reception & Type Detection

**Overview:**  
Receives incoming WhatsApp messages and classifies them by content type to route processing accordingly.

**Nodes Involved:**  
- Message Received  
- Message Type (Switch)

**Node Details:**  
- **Message Received**: WhatsApp Trigger node that listens for incoming messages via WhatsApp Business API OAuth2 credentials. Triggers on all message updates.  
  - Input: WhatsApp webhook messages  
  - Output: Raw WhatsApp message JSON  
  - Potential failures: Webhook connectivity, auth errors.

- **Message Type**: Switch node that inspects the incoming message JSON to determine its type among Image, Document, Text, Location, Voice, or fallback.  
  - Conditions use existence checks on message subfields (e.g., presence of `image`, `document`, `text.body`, `location`, `audio`)  
  - Routes messages to specialized processing nodes per type  
  - Edge cases: Unexpected or unsupported message types fall to fallback output.

---

#### 1.2 Media Processing

**Overview:**  
Handles the download and processing of different media types, enabling further AI analysis or transcription.

**Nodes Involved:**  
- Get Image Url  
- Download Image  
- OpenAI (Image Analysis)  
- Get File Url  
- Download File  
- Extract from File1  
- Extract PDF Content Using Gemini Vision  
- Get Audio Url  
- Download Audio  
- Transcribe Audio  
- Audio (Set node)  
- File (Set node)  
- Image (Set node)  

**Node Details:**  
- **Get Image Url** (WhatsApp node): Retrieves downloadable URL for image media from WhatsApp servers.  
- **Download Image** (HTTP Request): Downloads image binary using authenticated header auth (WhatsApp token).  
- **OpenAI (Image Analysis)**: Uses GPT-4o model with image input to produce a detailed transcription and description of the image content, tailored for payment receipt analysis.  
  - Uses base64 image input  
  - System prompt instructs thorough, factual description with OCR of visible text.

- **Get File Url** / **Download File**: Similar approach for documents (PDFs), retrieving URL and downloading content.  
- **Extract from File1**: Converts binary PDF data to text property for processing.  
- **Extract PDF Content Using Gemini Vision**: Sends extracted PDF data to Google's Gemini API for content extraction.  
  - Authenticated with API key via HTTP query  
  - Returns JSON with extracted text for AI analysis.

- **Get Audio Url** / **Download Audio**: Retrieves URL and downloads audio messages.  
- **Transcribe Audio**: Uses OpenAI's transcription operation to convert audio to text.  
- **Audio, File, Image (Set nodes)**: Format extracted or transcribed text into a standardized `text` property for subsequent AI nodes.

**Potential failures:**  
- Media download errors due to expired tokens or network issues.  
- OCR or transcription inaccuracies from low-quality media.  
- API quota or auth failures on OpenAI or Gemini services.

---

#### 1.3 AI Conversation & Order Handling

**Overview:**  
Engages the customer in conversation to take and confirm orders, leveraging GPT-4o models and managing conversation memory in Supabase.

**Nodes Involved:**  
- AI Restaurant Agent (LangChain Agent)  
- OpenAI Chat Model2 (GPT-4o)  
- Postgres Chat Memory1 (Supabase)  

**Node Details:**  
- **AI Restaurant Agent**: Core LangChain agent node that receives customer message text (including location coordinates if provided). Uses system prompt defining its role as a warm, professional restaurant host guiding the order process step by step with multilingual support.  
  - Inputs conversation context from Postgres Chat Memory1  
  - Outputs the AI-generated response text  
  - Uses GPT-4o model linked via OpenAI API credentials.

- **Postgres Chat Memory1**: Stores and retrieves conversation context keyed by customer phone number (WhatsApp sender ID) in a Supabase Postgres DB. Maintains context window of last 300 tokens.  
  - Supports session continuity and personalized interaction.

- **OpenAI Chat Model2**: GPT-4o language model node connected as the language model for AI Restaurant Agent.

**Potential failures:**  
- API rate limits or auth issues with OpenAI.  
- Database connectivity problems with Supabase.  
- Incomplete or unexpected messages causing conversation flow breaks.

---

#### 1.4 Order Structuring & Formatting

**Overview:**  
Parses AI agent outputs into a structured order JSON and formats a detailed order message for restaurant staff and customer confirmation.

**Nodes Involved:**  
- Structured Output Parser  
- Order Format (LangChain Agent)  
- Code (JavaScript)  
- Message to cashier  
- Message to Customer2  
- WhatsApp Business Cloud  
- WhatsApp Business Cloud2  
- Wait  
- Response to Customer  
- If, If1, If2, Switch  

**Node Details:**  
- **Structured Output Parser**: Parses AI raw text into a strict JSON schema describing customer name, order type, product list (name and price), shipping cost, address, geolocation, total, payment method, and reference number.  
- **Order Format**: Agent querying Postgres memory to retrieve last confirmed order for session, ensuring all data completeness, returning structured JSON.  
- **Code**: Formats the structured order JSON into a rich text message with emojis and formatted fields for sending to restaurant staff and customer.  
- **Message to cashier**: Sends formatted order message to cashier/restaurant staff WhatsApp number.  
- **Message to Customer2**: Sends confirmation and order details to the customer.  
- **WhatsApp Business Cloud / WhatsApp Business Cloud2**: Send message nodes handling different media types (image, text) back to WhatsApp customers.  
- **Wait**: Delays response delivery to customer if necessary.  
- **Response to Customer**: Final message send confirming order or payment status.  
- **If / Switch nodes**: Control flow based on AI output content (e.g., detecting QR code requests, payment confirmation, welcome messages).

**Potential failures:**  
- Parsing errors if AI output deviates from schema.  
- Message sending failures due to WhatsApp API limits or invalid phone numbers.  
- Timing issues in message sequencing.

---

#### 1.5 Payment Receipt Verification

**Overview:**  
Analyzes submitted payment receipts (images or PDFs) to confirm payment correctness against stored orders.

**Nodes Involved:**  
- Receipt Analysis (LangChain Agent)  
- Postgres Chat Memory4  
- OpenAI Chat Model6  
- Switch  
- Forward receipt to Management  

**Node Details:**  
- **Receipt Analysis**: Agent designed as payment verifier, receives OCR text extracted from receipts, compares amount, date, bank, account holder info against last confirmed order in Postgres memory.  
  - Produces verification result indicating success or problems.  
- **Postgres Chat Memory4**: Provides conversation context and stored order data for verification.  
- **OpenAI Chat Model6**: GPT variant used for receipt analysis.  
- **Switch**: Routes messages based on verification results (Verified or Inconsistency).  
- **Forward receipt to Management**: Forwards the original receipt document to management WhatsApp number for manual review if needed.

**Potential failures:**  
- OCR inaccuracies causing mismatches.  
- Timing mismatches (e.g., receipt date different from current date).  
- Receipt format variations causing parsing problems.

---

#### 1.6 Location Handling

**Overview:**  
Processes customer location messages by reverse geocoding lat-long coordinates into human-readable addresses.

**Nodes Involved:**  
- Google Maps (HTTP Request)  
- Location (Set node)  
- AI Restaurant Agent (receives location context)  

**Node Details:**  
- **Google Maps**: Calls Google Maps Geocoding API with latitude and longitude from WhatsApp location message, returning formatted address and navigation points.  
- **Location**: Extracts and sets address, latitude, longitude from Google Maps API response to standardized properties.  
- **AI Restaurant Agent**: Uses location data to confirm delivery address with customer.

**Potential failures:**  
- Invalid or missing GPS coordinates.  
- Google Maps API quota or key errors.  
- Ambiguous or incomplete address results.

---

#### 1.7 Outgoing Communication

**Overview:**  
Sends responses, menus, order confirmations, and QR codes to customers and order details to restaurant staff via WhatsApp.

**Nodes Involved:**  
- WhatsApp Business Cloud  
- WhatsApp Business Cloud2  
- Message to Customer2  
- Message to cashier  
- Menu 1 (Google Drive)  
- Download file  
- Sticky Notes (for visual guidance)

**Node Details:**  
- **WhatsApp Business Cloud / WhatsApp Business Cloud2**: Send messages with text, images, or documents, using WhatsApp Business API credentials.  
- **Message to Customer2 / Message to cashier**: Sends order confirmation and payment verification messages.  
- **Menu 1**: Downloads menu images from Google Drive for sending on customer request.  
- **Download file**: Used for downloading files from Google Drive.  
- Sticky notes provide non-executable documentation hints.

**Potential failures:**  
- WhatsApp API limits or credential expiration.  
- File not found or access denied on Google Drive.  
- Incorrect phone numbers causing delivery failures.

---

#### 1.8 Setup & Documentation Notes

**Overview:**  
Sticky Note nodes provide clear instructions, overview, and setup guidance to operators and developers.

**Nodes Involved:**  
- Workflow Overview (sticky note)  
- Setup Guide (sticky note)  
- Other sticky notes: Recognize Image Type Receipts, Recognize voice messages, Recognize customer location, Recognize PDF type receipts, etc.

**Node Details:**  
- Contain descriptive text about workflow purpose, required credentials, setup steps, customization instructions.  
- Disabled sticky notes contain deprecated or future feature notes.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                             | Input Node(s)               | Output Node(s)                 | Sticky Note                                                  |
|-----------------------------|-----------------------------------|---------------------------------------------|-----------------------------|-------------------------------|--------------------------------------------------------------|
| Message Received            | WhatsApp Trigger                   | Entry point for incoming WhatsApp messages  | -                           | Message Type                  |                                                              |
| Message Type                | Switch                            | Classify message type to route processing   | Message Received            | Get Image Url, Get File Url, Text, Google Maps, Get Audio Url |                                                              |
| Get Image Url               | WhatsApp                          | Retrieve image download URL                  | Message Type (Image)        | Download Image                | Recognize Image Type Receipts                                 |
| Download Image              | HTTP Request                     | Download image binary                         | Get Image Url               | OpenAI                        |                                                              |
| OpenAI                     | LangChain openAI image analysis  | Analyze image and transcribe content         | Download Image              | Image (Set node)              |                                                              |
| Image                      | Set                              | Format AI image analysis output               | OpenAI                     | Receipt Analysis              |                                                              |
| Get File Url               | WhatsApp                          | Retrieve document download URL                | Message Type (Document)     | Download File                | Recognize PDF type receipts                                  |
| Download File              | HTTP Request                     | Download document binary                       | Get File Url                | Extract from File1             |                                                              |
| Extract from File1          | Extract from File                | Convert PDF binary to text property            | Download File              | Extract PDF Content Using Gemini Vision |                                                              |
| Extract PDF Content Using Gemini Vision | HTTP Request          | Extract PDF text content via Gemini API        | Extract from File1          | File (Set node)               |                                                              |
| File                       | Set                              | Format extracted PDF text                       | Extract PDF Content Using Gemini Vision | Forward receipt to Management, Receipt Analysis |                                                              |
| Forward receipt to Management| WhatsApp                         | Forward document receipt to management         | File                       | Receipt Analysis              |                                                              |
| Get Audio Url              | WhatsApp                          | Retrieve audio message download URL            | Message Type (Voice)        | Download Audio               | Recognize voice messages                                     |
| Download Audio             | HTTP Request                     | Download audio binary                           | Get Audio Url              | Transcribe Audio             |                                                              |
| Transcribe Audio           | LangChain openAI audio transcription | Convert audio to text                           | Download Audio             | Audio (Set node)              |                                                              |
| Audio                      | Set                              | Format transcribed audio text                   | Transcribe Audio           | AI Restaurant Agent           |                                                              |
| Text                       | Set                              | Format text message body                         | Message Type (Text)         | AI Restaurant Agent           |                                                              |
| Google Maps                | HTTP Request                     | Reverse geocode lat-long to address             | Message Type (Location)     | Location                     | Recognize customer location                                 |
| Location                   | Set                              | Format address and coordinates                  | Google Maps                | AI Restaurant Agent           |                                                              |
| AI Restaurant Agent        | LangChain Agent                  | Conversational AI agent for order taking       | Text, Audio, Location       | If, If1, If2                 | Chat with customer about order                              |
| If                        | If                              | Condition based on AI output (e.g., QR code requests) | AI Restaurant Agent         | Download file, Menu 1         |                                                              |
| If1                       | If                              | Condition for welcome message detection         | AI Restaurant Agent         | Menu 1                      |                                                              |
| If2                       | If                              | Condition for order confirmation with cash payment | AI Restaurant Agent         | Message to Customer2, Order Format, Wait |                                                              |
| Menu 1                    | Google Drive                    | Download menu image file                          | If1                        | WhatsApp Business Cloud      | Send Menu Image (disabled sticky note)                      |
| Download file             | Google Drive                    | Download file from Google Drive                   | If                         | WhatsApp Business Cloud2     |                                                              |
| WhatsApp Business Cloud   | WhatsApp                        | Send menu image or media                          | Menu 1                     | -                           |                                                              |
| WhatsApp Business Cloud2  | WhatsApp                        | Send media message                                | Download file              | -                           |                                                              |
| Message to Customer2      | WhatsApp                        | Send order confirmation message to customer      | If2, Code                  | -                           |                                                              |
| Message to cashier        | WhatsApp                        | Send formatted order to cashier                   | Code                       | -                           |                                                              |
| Wait                      | Wait                           | Delay message sending                             | If2                        | Response to Customer         |                                                              |
| Response to Customer      | WhatsApp                        | Send response message to customer                 | Wait                       | -                           |                                                              |
| Structured Output Parser  | LangChain Output Parser         | Parse AI raw text into structured order JSON     | OpenAI Chat Model1          | Order Format                 |                                                              |
| Order Format              | LangChain Agent                | Retrieve and format complete order from memory    | Structured Output Parser, Postgres Chat Memory | Code, If2                 | Order Confirmation                                           |
| Code                      | Function (Code)               | Create formatted order message text                | Order Format               | Message to cashier, Message to Customer2 |                                                              |
| Postgres Chat Memory1     | LangChain Memory (Postgres)   | Store/retrieve conversation context for AI Agent | AI Restaurant Agent        | AI Restaurant Agent          |                                                              |
| Postgres Chat Memory4     | LangChain Memory (Postgres)   | Provide memory for receipt analysis agent          | Receipt Analysis           | Receipt Analysis             |                                                              |
| OpenAI Chat Model1        | LangChain LM Chat OpenAI       | GPT-4o-mini for order parsing                      | -                         | Structured Output Parser     |                                                              |
| OpenAI Chat Model2        | LangChain LM Chat OpenAI       | GPT-4o for AI Restaurant Agent                     | -                         | AI Restaurant Agent          |                                                              |
| OpenAI Chat Model6        | LangChain LM Chat OpenAI       | GPT-o3-mini for Receipt Analysis agent             | -                         | Receipt Analysis             |                                                              |
| Receipt Analysis          | LangChain Agent                | Analyze payment receipts against order data        | Image, File                | Switch                      |                                                              |
| Switch                    | Switch                        | Route based on receipt verification result         | Receipt Analysis           | Message to cashier, Message to Customer2 |                                                              |
| Workflow Overview         | Sticky Note                   | Workflow purpose and features description           | -                         | -                           |                                                              |
| Setup Guide               | Sticky Note                   | Setup instructions and configuration guide          | -                         | -                           |                                                              |
| Sticky Note (various)     | Sticky Note                   | Functional notes on image, voice, location recognition | -                         | -                           |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node ("Message Received"):**  
   - Type: WhatsApp Trigger  
   - Credentials: WhatsApp OAuth2 (Business API)  
   - Trigger on message updates  
   - Purpose: Entry point for all incoming WhatsApp messages.

2. **Add a Switch Node ("Message Type") to classify message content:**  
   - Conditions: Check existence of `image`, `document`, `text.body`, `location`, `audio` fields in incoming message JSON.  
   - Outputs: Image, Document, Text, Location, Voice, and fallback.  
   - Connect input from "Message Received".

3. **For Image Messages:**  
   - Add "Get Image Url" (WhatsApp media URL retrieval) with media ID from message.  
   - Add "Download Image" HTTP Request node with header auth to download the image binary.  
   - Add "OpenAI" node for image analysis:  
     - Operation: Analyze image (base64 input)  
     - Model: GPT-4o  
     - System prompt: Detailed image description tailored to receipt transcription.  
   - Add "Image" Set node to format output text.  
   - Connect to "Receipt Analysis" agent.

4. **For Document Messages (PDF receipts):**  
   - Add "Get File Url" node to get document media URL.  
   - Add "Download File" HTTP Request node to download the file binary.  
   - Add "Extract from File" node to convert binary to text property.  
   - Add "Extract PDF Content Using Gemini Vision" HTTP Request node:  
     - URL: Gemini API endpoint  
     - POST JSON body with embedded base64 PDF data  
     - Auth: Gemini API key via query parameter.  
   - Add "File" Set node to format extracted text.  
   - Connect to "Forward receipt to Management" WhatsApp node for manual forwarding.  
   - Connect to "Receipt Analysis" agent.

5. **For Audio Messages:**  
   - Add "Get Audio Url" node for audio media URL.  
   - Add "Download Audio" HTTP Request with header auth to download audio.  
   - Add "Transcribe Audio" LangChain openAI node:  
     - Operation: transcribe audio  
     - Model: OpenAI audio transcription  
   - Add "Audio" Set node to format transcribed text.  
   - Connect to "AI Restaurant Agent".

6. **For Text Messages:**  
   - Add "Text" Set node extracting `text.body`.  
   - Connect to "AI Restaurant Agent".

7. **For Location Messages:**  
   - Add "Google Maps" HTTP Request node:  
     - URL: Google Maps Geocode API with lat-long from message  
     - Auth: Google Maps API key via query parameter.  
   - Add "Location" Set node extracting formatted address and coordinates.  
   - Connect to "AI Restaurant Agent".

8. **AI Restaurant Agent:**  
   - LangChain Agent node configured with system prompt defining restaurant order conversational flow.  
   - Input: message text or audio transcription or location data.  
   - Use OpenAI GPT-4o model credentials.  
   - Connect memory node "Postgres Chat Memory1" for conversation context using phone number as session key.  
   - Connect output to conditional "If" nodes for flow control.

9. **Order Parsing:**  
   - Add "Structured Output Parser" node with JSON schema to parse AI agent output into structured order data.  
   - Add "Order Format" LangChain Agent to query Postgres memory for last confirmed order data.  
   - Use the OpenAI GPT-4o-mini model for parsing.  
   - Connect output to "Code" node.

10. **Order Formatting ("Code" Node):**  
    - JavaScript code node formats order JSON into a multi-line WhatsApp message with emojis and detailed fields.  
    - Outputs formatted order message text.

11. **Send Messages:**  
    - "Message to cashier" WhatsApp node sends order details to restaurant staff number.  
    - "Message to Customer2" WhatsApp node sends order confirmation to customer.  
    - "Wait" node optionally delays messages for timing control.  
    - "Response to Customer" WhatsApp node sends final customer response.

12. **Receipt Analysis:**  
    - LangChain Agent node with system prompt instructing payment receipt verification using extracted text and Postgres memory.  
    - Uses OpenAI GPT-o3-mini model.  
    - Connects to Postgres Chat Memory4 for last order data.  
    - Output routed through a "Switch" node to handle Verified or Inconsistency results.  
    - If inconsistency detected, forward receipt to management WhatsApp number.

13. **Menu and QR Code Sending:**  
    - Add Google Drive node ("Menu 1") to download menu images.  
    - Connect to WhatsApp Business Cloud node to send menu images on request.  
    - Prepare similar nodes to send QR codes and specials (if applicable).

14. **Sticky Notes:**  
    - Add sticky notes for workflow overview and setup instructions (non-executable).  
    - Include notes on recognizing image receipts, voice messages, location handling, and PDF receipts.

15. **Credentials Setup:**  
    - WhatsApp Business API OAuth2 credentials for receiving and sending messages.  
    - OpenAI API key for GPT-4o and transcription.  
    - Supabase Postgres credentials for conversation memory.  
    - Google Drive OAuth2 for menu files.  
    - Google Maps API key for geocoding.  
    - Gemini API key for PDF content extraction.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow supports multi-language customer interaction: English, Spanish, French, Italian, Portuguese.                                      | AI Restaurant Agent system prompt                                                                                   |
| Setup requires valid WhatsApp Business API credentials and OpenAI API keys for GPT-4o and transcription features.                         | Setup Guide sticky note                                                                                              |
| Integration with Supabase Postgres for conversation memory enables session continuity and querying last confirmed orders.                  | Postgres Chat Memory nodes                                                                                           |
| Uses Google Maps Geocoding API for converting lat-long to addresses to confirm delivery location.                                          | Google Maps HTTP Request node                                                                                        |
| Payment receipt verification includes validating amount, date, bank, account holder, and receipt authenticity.                            | Receipt Analysis agent node                                                                                          |
| Menu images and QR codes are stored on Google Drive and delivered on customer request.                                                    | Google Drive nodes and WhatsApp Business Cloud nodes                                                                |
| AI prompts and system messages are customizable to adapt to specific restaurant branding, policies, and conversation flow preferences.    | Sticky notes "Workflow Overview" and "Setup Guide"                                                                   |
| The workflow incorporates error handling through conditional nodes and fallback routes for unsupported message types or verification fails. | Switch and If nodes throughout the workflow                                                                          |
| Video or external blog links are not included but recommended for detailed WhatsApp Business API and OpenAI integration guides.            | External documentation recommended                                                                                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow and fully complies with content policies. It contains no illegal, offensive, or protected content. All processed data is legal and public.

---