Nano Banana AI Product Image Creator via WhatsApp

https://n8nworkflows.xyz/workflows/nano-banana-ai-product-image-creator-via-whatsapp-8549


# Nano Banana AI Product Image Creator via WhatsApp

### 1. Workflow Overview

The **Nano Banana AI Product Image Creator via WhatsApp** workflow automates the process of receiving a product image via WhatsApp, analyzing it with AI to generate a premium advertisement-style enhancement, and sending the enhanced image back to the user. This workflow is designed for businesses or marketers who want to leverage AI-driven visual enhancement of product images directly through WhatsApp messaging.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Capture incoming WhatsApp messages containing images.
- **1.2 Media Retrieval and Preparation:** Extract the image from WhatsApp servers, download it, and convert it to base64 format.
- **1.3 AI Processing:** Use Google Gemini AI to analyze the image and generate design suggestions, then prepare and send a request to generate an enhanced image.
- **1.4 Output Delivery:** Convert the AI-generated image back to a file and send it to the user via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block listens for incoming WhatsApp messages, specifically filtering for messages containing images.
- **Nodes Involved:** `WhatsApp Message Received`

##### WhatsApp Message Received
- **Type:** WhatsApp Trigger node
- **Role:** Entry point for the workflow; listens for incoming WhatsApp messages.
- **Configuration:** 
  - Monitors only `"messages"` update events.
  - Requires WhatsApp Business API credentials.
- **Expressions/Variables:** 
  - Used downstream to extract image ID and sender information.
- **Input/Output:** No input; outputs message JSON including image data.
- **Version:** n8n v1 compatible.
- **Failure Modes:** 
  - Authentication errors if credentials invalid.
  - Missed messages if webhook not configured properly.
- **Notes:** Only triggers on messages, so non-message updates ignored.

#### 1.2 Media Retrieval and Preparation

- **Overview:** Retrieves the media URL for the image sent, downloads it, and converts it to base64 for AI input.
- **Nodes Involved:** `Get Media URL`, `Download Image File`, `Image to Base64`
  
##### Get Media URL
- **Type:** WhatsApp node
- **Role:** Fetches the download URL for the image using the media ID from the incoming message.
- **Configuration:** 
  - Operation: `mediaUrlGet`
  - Media ID from expression: `{{ $('WhatsApp Message Received').item.json.messages[0].image.id }}`
  - Requires WhatsApp API credentials.
- **Input:** Output from `WhatsApp Message Received`.
- **Output:** JSON containing media URL.
- **Failure Modes:** 
  - Invalid media ID.
  - Authorization failure with WhatsApp API.
  - Media not found errors.

##### Download Image File
- **Type:** HTTP Request node
- **Role:** Downloads the actual image file from WhatsApp servers using the media URL.
- **Configuration:** 
  - Method: GET
  - URL from previous node: `{{ $json.url }}`
  - Header includes `Authorization: Bearer YOUR_WHATSAPP_ACCESS_TOKEN` (must be replaced).
  - Response format set to file.
- **Input:** Output from `Get Media URL`.
- **Output:** Binary file data of the image.
- **Failure Modes:** 
  - Invalid or expired access token.
  - Network timeouts.
  - HTTP errors (404, 403).

##### Image to Base64
- **Type:** Extract From File node
- **Role:** Converts the binary image file into a base64 encoded string property.
- **Configuration:** Operation: `binaryToProperty`.
- **Input:** Binary data from `Download Image File`.
- **Output:** JSON with base64 property for AI nodes.
- **Failure Modes:** 
  - Corrupt binary data.
  - Conversion errors if file format unsupported.

#### 1.3 AI Processing

- **Overview:** Runs AI analysis on the base64 image to generate design suggestions, merges results, prepares payload, and generates enhanced image using Google Gemini AI.
- **Nodes Involved:** `AI Design Analysis`, `Combine Image & Analysis`, `Prepare API Payload`, `Generate Enhanced Image`

##### AI Design Analysis
- **Type:** Google Gemini AI node (Langchain integration)
- **Role:** Analyzes the image and caption to produce a detailed commercial photography concept and advertisement text in JSON format.
- **Configuration:** 
  - Model: `gemini-2.5-flash-preview-05-20`
  - Input type: Binary image from base64 conversion.
  - Prompt includes detailed instructions to generate premium ad copy and design suggestions using only user-provided caption text.
  - Requires Google Palm API credentials.
- **Input:** Binary image from `Download Image File`.
- **Output:** JSON with headline, subheadline, description, call to action, and detailed design suggestions.
- **Failure Modes:** 
  - API quota exceeded.
  - Invalid API key.
  - Prompt parsing or response format errors.

##### Combine Image & Analysis
- **Type:** Merge node
- **Role:** Combines base64 image data and AI analysis JSON into a single data stream.
- **Configuration:** Default merge settings (merge by index).
- **Input:** 
  - Base64 image data (from `Image to Base64`).
  - AI analysis results (from `AI Design Analysis`).
- **Output:** Combined data for payload preparation.
- **Failure Modes:** Data mismatch if one input missing.

##### Prepare API Payload
- **Type:** Code (JavaScript) node
- **Role:** Formats the combined data into the payload structure expected by the Google Gemini image generation API.
- **Configuration:** 
  - Custom JS code consolidates inputs into a single JSON object with `input1` and `input2`.
- **Input:** Combined output from `Combine Image & Analysis`.
- **Output:** JSON payload for image generation API.
- **Failure Modes:** 
  - Coding errors.
  - Empty or malformed input data.

##### Generate Enhanced Image
- **Type:** HTTP Request node
- **Role:** Sends a POST request to Google Gemini API to generate a premium enhanced product image.
- **Configuration:** 
  - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent`
  - Method: POST
  - Body: JSON constructed from previous node outputs, embedding text and base64 image data.
  - Authentication: Uses Google Palm API credentials.
- **Input:** Payload from `Prepare API Payload`.
- **Output:** AI-generated image data in base64 format within JSON response.
- **Failure Modes:** 
  - API rate limits.
  - Authentication failure.
  - Payload validation errors.
  - Network issues.

#### 1.4 Output Delivery

- **Overview:** Converts the AI-generated base64 image data back to a file and sends it back to the user on WhatsApp.
- **Nodes Involved:** `Convert Base64 to Image`, `Send Image`

##### Convert Base64 to Image
- **Type:** Convert to File node
- **Role:** Converts the base64 encoded AI-generated image data back into a binary image file.
- **Configuration:** 
  - Operation: `toBinary`
  - Source property: `candidates[0].content.parts[0].inlineData.data`
- **Input:** JSON response from `Generate Enhanced Image`.
- **Output:** Binary image file suitable for sending.
- **Failure Modes:** 
  - Invalid or missing base64 data.
  - Conversion failure.

##### Send Image
- **Type:** WhatsApp node
- **Role:** Sends the enhanced image file back to the original sender via WhatsApp.
- **Configuration:** 
  - Operation: Send
  - Message type: Image
  - Recipient phone number extracted dynamically from original WhatsApp message sender.
  - Uses WhatsApp API credentials.
- **Input:** Binary image file from `Convert Base64 to Image`.
- **Output:** Confirmation of sent message.
- **Failure Modes:** 
  - Invalid recipient number format.
  - WhatsApp API errors.
  - Network issues.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                           | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                   |
|-------------------------|---------------------------------|-----------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| WhatsApp Message Received| WhatsApp Trigger                | Listens for incoming WhatsApp messages | None                        | Get Media URL               | Listens for incoming WhatsApp messages with images. Requires WhatsApp Business API credentials.               |
| Get Media URL            | WhatsApp                       | Retrieves download URL for image        | WhatsApp Message Received   | Download Image File         | Gets download URL for the image sent by user. Requires WhatsApp API credentials.                              |
| Download Image File      | HTTP Request                   | Downloads image file from WhatsApp      | Get Media URL               | Image to Base64, AI Design Analysis | Downloads image file from WhatsApp servers. Requires valid access token.                                      |
| Image to Base64          | Extract From File              | Converts image file to base64            | Download Image File         | Combine Image & Analysis    | Converts downloaded image to base64 for AI processing.                                                        |
| AI Design Analysis       | Google Gemini (Langchain)      | Analyzes image, generates design ideas | Download Image File         | Combine Image & Analysis    | Analyzes image for design improvements using Google Gemini AI. Requires Google Palm API credentials.          |
| Combine Image & Analysis | Merge                         | Combines image data and AI analysis     | Image to Base64, AI Design Analysis | Prepare API Payload         | Combines base64 image data with AI analysis results.                                                          |
| Prepare API Payload      | Code (JavaScript)              | Formats payload for image generation API| Combine Image & Analysis    | Generate Enhanced Image     | Formats data for Google Gemini image generation API.                                                          |
| Generate Enhanced Image  | HTTP Request                  | Generates enhanced image using AI       | Prepare API Payload         | Convert Base64 to Image     | Calls Google Gemini API to create enhanced image. Requires Google Palm API credentials.                        |
| Convert Base64 to Image  | Convert to File                | Converts base64 AI image back to file   | Generate Enhanced Image     | Send Image                  | Converts AI-generated base64 image back to a file.                                                            |
| Send Image               | WhatsApp                      | Sends enhanced image back to user       | Convert Base64 to Image     | None                       | Sends the enhanced image to the original WhatsApp sender. Requires WhatsApp API credentials.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Message Received node**
   - Type: WhatsApp Trigger
   - Set updates to: `messages` only
   - Attach WhatsApp Business API credentials
   - Configure webhook ID (unique for your instance)

2. **Create Get Media URL node**
   - Type: WhatsApp
   - Resource: `media`
   - Operation: `mediaUrlGet`
   - Media ID: Use expression `={{ $('WhatsApp Message Received').item.json.messages[0].image.id }}`
   - Attach WhatsApp API credentials
   - Connect output of `WhatsApp Message Received` to input of this node

3. **Create Download Image File node**
   - Type: HTTP Request
   - Method: GET
   - URL: Use expression `={{ $json.url }}`
   - Add header: `Authorization: Bearer YOUR_WHATSAPP_ACCESS_TOKEN` (replace with your token)
   - Response format: File
   - Connect output of `Get Media URL` to input of this node

4. **Create Image to Base64 node**
   - Type: Extract From File
   - Operation: `binaryToProperty`
   - Connect output of `Download Image File` to input of this node

5. **Create AI Design Analysis node**
   - Type: Google Gemini (Langchain)
   - Resource: Image
   - Operation: Analyze
   - Model: `gemini-2.5-flash-preview-05-20`
   - Input Type: Binary
   - Text prompt: Use the detailed prompt provided to generate ad copy and design suggestions using the user’s caption
   - Attach Google Palm API credentials
   - Connect output of `Download Image File` to input of this node

6. **Create Combine Image & Analysis node**
   - Type: Merge
   - Use default merge settings (merge by index)
   - Connect outputs of `Image to Base64` (main input 1) and `AI Design Analysis` (main input 2) to this node

7. **Create Prepare API Payload node**
   - Type: Code (JavaScript)
   - Paste code to combine inputs into a single JSON object with keys `input1` and `input2`
   - Connect output of `Combine Image & Analysis` to input of this node

8. **Create Generate Enhanced Image node**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent`
   - Authentication: Use Google Palm API credentials
   - Request Body (JSON): Construct using expressions to embed text and base64 data from previous node, matching Google Gemini's API format
   - Connect output of `Prepare API Payload` to input of this node

9. **Create Convert Base64 to Image node**
   - Type: Convert to File
   - Operation: `toBinary`
   - Source Property: `candidates[0].content.parts[0].inlineData.data`
   - Connect output of `Generate Enhanced Image` to input of this node

10. **Create Send Image node**
    - Type: WhatsApp
    - Operation: Send
    - Message Type: Image
    - Phone Number ID: Your WhatsApp Business phone number ID
    - Recipient Phone Number: Use expression `=+{{ $('WhatsApp Message Received').first().json.contacts[0].wa_id }}`
    - Media Path: Use binary data from previous node
    - Attach WhatsApp API credentials for sending
    - Connect output of `Convert Base64 to Image` to input of this node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| Replace all placeholder tokens such as `YOUR_WHATSAPP_ACCESS_TOKEN`, `YOUR_INSTANCE_ID`, `YOUR_WEBHOOK_ID`, and credential IDs with your actual credentials and configuration values.                                                                                  | Setup instructions                          |
| Ensure WhatsApp Business API account has permission to receive and send media messages. Monitor API rate limits to avoid throttling.                                                                                                                                | WhatsApp Business API documentation         |
| Google Gemini AI requires Google Palm API credentials with access to image analysis and generation models. Keep API tokens secure and rotate regularly.                                                                                                              | Google Palm API documentation                |
| The detailed AI prompt is designed to create premium commercial product advertisements with strict rules to preserve the original product's integrity while enhancing visual appeal.                                                                                 | AI prompt embedded in node `AI Design Analysis` |
| Test with simple images initially to validate the workflow before scaling to production.                                                                                                                                                                            | Best practice                               |
| Workflow triggers only on images sent via WhatsApp; messages without images will not start the workflow.                                                                                                                                                             | Functional limitation                        |
| For more information on WhatsApp Business API and Google Gemini AI integration, refer to official docs: https://developers.facebook.com/docs/whatsapp and https://developers.generativelanguage.google | External resources                          |

---

**Disclaimer:** The text provided above is derived exclusively from an automated n8n workflow developed for integration and automation purposes. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data are legal and publicly accessible.