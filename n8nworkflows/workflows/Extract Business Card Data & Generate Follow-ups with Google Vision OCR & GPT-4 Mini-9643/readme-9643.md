Extract Business Card Data & Generate Follow-ups with Google Vision OCR & GPT-4 Mini

https://n8nworkflows.xyz/workflows/extract-business-card-data---generate-follow-ups-with-google-vision-ocr---gpt-4-mini-9643


# Extract Business Card Data & Generate Follow-ups with Google Vision OCR & GPT-4 Mini

### 1. Workflow Overview

This workflow automates the processing of business card images sent via Telegram. It extracts contact and company details using Google Vision OCR, then applies AI (GPT-4 Mini) to structure the data, generate personalized follow-up emails, WhatsApp drafts, and internal notes including fit scoring and next actions. Finally, it stores all information in Google Sheets and optionally sends an email copy to the user.

Logical blocks:

- **1.1 Input Reception:** Trigger and acquisition of business card image from Telegram.
- **1.2 Image Processing & OCR:** Conversion of image to Base64 and text extraction using Google Vision API.
- **1.3 AI Data Structuring & Content Generation:** AI parses raw OCR text and user context to extract structured contact info and draft personalized communications.
- **1.4 Data Cleaning & Validation:** Clean and parse AI JSON output; verify presence of key fields (email).
- **1.5 Data Storage and Notification:** Append processed data to Google Sheets and optionally send email notification to user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Starts the workflow upon receiving a message (photo or caption) on Telegram, downloading any attached files for further processing.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Listens for incoming Telegram messages with images or text; downloads message attachments (business card photos).  
    - Configuration:  
      - Updates: message  
      - Download: Enabled (to retrieve image files)  
      - Webhook: Configured via Telegram webhook URL linked to Telegram bot token  
    - Inputs: External Telegram message events  
    - Outputs: JSON containing message data and binary image file  
    - Edge cases: Message without image; network or webhook misconfiguration; Telegram API rate limits  

#### 1.2 Image Processing & OCR

- **Overview:**  
Converts the received business card image to Base64 format and sends it to Google Vision API for text extraction.

- **Nodes Involved:**  
  - Convert Image to Base64  
  - HTTP Request (Google Vision OCR)  
  - Extract Raw Text)

- **Node Details:**

  - **Convert Image to Base64**  
    - Type: Code  
    - Role: Converts binary image data from Telegram Trigger into Base64 string for API consumption.  
    - Configuration: Custom JavaScript reads binary data and encodes it as Base64, stored in JSON property `base64`.  
    - Input: Binary image from Telegram Trigger  
    - Output: JSON with `base64` string  
    - Edge cases: Missing or corrupted binary data, encoding failures  

  - **HTTP Request (Google Vision OCR)**  
    - Type: HTTP Request  
    - Role: Posts Base64 image to Google Vision API to perform TEXT_DETECTION OCR.  
    - Configuration:  
      - URL: `https://vision.googleapis.com/v1/images:annotate?key=YOUR_GOOGLE_VISION_API_KEY` (user must replace key)  
      - Method: POST  
      - Headers: Content-Type application/json  
      - Body: JSON specifying image content and feature type TEXT_DETECTION  
    - Input: JSON with Base64 image  
    - Output: JSON OCR response from Google Vision  
    - Edge cases: API key invalid or missing, quota exceeded, network issues, malformed request  

  - **Extract Raw Text)**  
    - Type: Code  
    - Role: Extracts plain text from Google Vision OCR response (`fullTextAnnotation.text`) and passes only the raw text forward.  
    - Configuration: JS code that safely extracts text or returns empty string if missing.  
    - Input: OCR JSON response  
    - Output: JSON with `raw_text` string  
    - Edge cases: Missing or empty OCR results, malformed response JSON  

#### 1.3 AI Data Structuring & Content Generation

- **Overview:**  
Uses GPT-4 Mini via the LangChain AI Agent node to parse raw OCR text and user caption, extracting structured contact info, scoring fit, and generating personalized email and WhatsApp drafts.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Sends prompt containing raw OCR text, Telegram caption, and user’s personal info to AI for processing and data extraction according to detailed instructions.  
    - Configuration:  
      - Prompt includes explicit instructions to extract fields (name, company, role, emails, phones, country, website), infer company type and industry, generate internal summary, automation ideas, fit score and reason, email subject and HTML body, WhatsApp draft, next actions, and opportunities summary.  
      - Output must be strict JSON matching exact schema.  
    - Input: JSON with `raw_text`, Telegram caption, user info; AI language model connected as resource.  
    - Output: AI-generated JSON string result.  
    - Edge cases: AI response not JSON, incomplete or incorrect fields, API failures, rate limits  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4 Mini model for AI Agent to use.  
    - Configuration: Model set to `gpt-4.1-mini`.  
    - Inputs and outputs: Connected as AI language model resource for AI Agent node.  
    - Edge cases: API key issues, model availability, request limits  

#### 1.4 Data Cleaning & Validation

- **Overview:**  
Cleans AI output by removing formatting, parsing JSON, normalizing arrays, and verifying the presence of at least one valid email address.

- **Nodes Involved:**  
  - Clean AI Output  
  - If (Check for Email Found)

- **Node Details:**

  - **Clean AI Output**  
    - Type: Code  
    - Role:  
      - Strips markdown/code fences from AI string output  
      - Parses JSON strictly and throws error if invalid  
      - Normalizes emails and phones arrays, trims entries, formats phone numbers  
      - Extracts key fields including subject and HTML body for email  
    - Input: AI Agent output string  
    - Output: Cleaned JSON object with all expected fields (name, company, emails, phones, etc.)  
    - Edge cases: Parsing failure throws explicit error; missing fields default to empty strings or arrays  

  - **If (Check for Email Found)**  
    - Type: If Condition  
    - Role: Checks if the first email in extracted data is non-empty to decide if data is valid and worth further processing.  
    - Configuration: Condition: `$json.emails[0]` is not empty.  
    - Input: Clean AI Output JSON  
    - True Output: Proceed to append data to sheet  
    - False Output: Stops or skips further processing  
    - Edge cases: Empty or incorrectly formatted emails array  

#### 1.5 Data Storage and Notification

- **Overview:**  
Saves validated contact and AI-generated data to Google Sheets and optionally sends a copy of the personalized email to the user via Gmail.

- **Nodes Involved:**  
  - Append row in sheet  
  - Send a message  

- **Node Details:**

  - **Append row in sheet (Google Sheets)**  
    - Type: Google Sheets node  
    - Role: Appends a new row with extracted and generated data into a specified Google Sheet for record keeping.  
    - Configuration:  
      - Google account OAuth2 credentials required  
      - Document ID: user must specify Google Sheet ID  
      - Sheet name: default "Sheet1" or as configured  
      - Columns mapped to JSON fields: Name, Email, Country, Website, Fit Score, Email Body, Fit Reason, User Input (caption), Company Name, Next Actions (3 items), Phone Number, Email Subject, Opportunities (5 items), Position/Role, Alternate Email, WhatsApp Message, Opportunities Summary, Alternate Phone Number  
    - Input: Clean, validated JSON from previous step plus Telegram caption from trigger  
    - Output: Confirmation of append operation  
    - Edge cases: Credentials expire, Sheet ID incorrect, quota issues, write conflicts  

  - **Send a message (Gmail)**  
    - Type: Gmail node  
    - Role: Optionally sends the generated personalized email to the user’s email address as a notification or copy.  
    - Configuration:  
      - Gmail OAuth2 credentials connected  
      - Send To: static email (hi@adityamalur.com)  
      - Subject: from AI output’s "Email Subject"  
      - Message body: from AI output’s "Email Body" (HTML content)  
      - Append Attribution: disabled  
    - Input: Append row in sheet output (passes email subject/body)  
    - Output: Email sent confirmation  
    - Edge cases: Credential expiration, quota limits, invalid email addresses  

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                                     | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                                                              |
|----------------------|--------------------------------|----------------------------------------------------|---------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger      | Telegram Trigger               | Receives business card image and caption from Telegram | (start)                   | Convert Image to Base64  | Starts the workflow on Telegram message with photo. Setup notes on connecting Telegram bot and webhook.                                 |
| Convert Image to Base64 | Code                          | Converts image binary to Base64 string for OCR API | Telegram Trigger          | HTTP Request            | Converts image to Base64 for Google Vision. No credentials needed.                                                                        |
| HTTP Request         | HTTP Request                  | Sends Base64 image to Google Vision API for OCR     | Convert Image to Base64    | Extract Raw Text)        | Sends image to Google Vision OCR API. Setup includes API key and JSON body format.                                                        |
| Extract Raw Text)     | Code                          | Extracts plain OCR text from Google Vision response | HTTP Request              | AI Agent                | Extracts raw OCR text for AI input.                                                                                                       |
| AI Agent             | LangChain AI Agent            | Parses OCR text and user context to extract structured data and generate follow-up content | Extract Raw Text)         | Clean AI Output          | Core AI processing node. Uses GPT-4 Mini to structure info and compose emails/messages.                                                   |
| OpenAI Chat Model     | LangChain OpenAI Chat Model   | Provides GPT-4 Mini model for AI Agent              | (resource to AI Agent)    | AI Agent                | Configured with GPT-4 Mini model.                                                                                                         |
| Clean AI Output       | Code                          | Cleans and parses AI JSON output, normalizes data   | AI Agent                  | If                      | Cleans AI output, removes markdown, parses JSON, normalizes arrays.                                                                       |
| If                   | If Condition                  | Checks that at least one valid email is extracted   | Clean AI Output           | Append row in sheet      | Checks presence of email before saving data.                                                                                              |
| Append row in sheet   | Google Sheets                 | Saves extracted and generated data to Google Sheets | If                       | Send a message           | Saves all data to Google Sheet. Setup includes Google OAuth2 and sheet mapping.                                                           |
| Send a message        | Gmail                         | Optionally sends personalized email copy to user    | Append row in sheet       | (end)                   | Sends generated email to user. Setup requires Gmail OAuth2 credentials.                                                                   |
| Sticky Note           | Sticky Note                   | Informational notes on workflow design and setup    | (none)                   | (none)                  | Various notes explaining each workflow block and node configuration, including credentials setup, API keys, and usage instructions.      |
| Sticky Note1          | Sticky Note                   | Explanation of Telegram Trigger node                 | (none)                   | (none)                  | Detailed setup instructions for Telegram Trigger node.                                                                                    |
| Sticky Note2          | Sticky Note                   | Explanation of Base64 conversion node                | (none)                   | (none)                  | Explains purpose and setup of image to Base64 conversion.                                                                                 |
| Sticky Note3          | Sticky Note                   | Explanation of Google Vision OCR node                | (none)                   | (none)                  | Setup details for Google Vision OCR HTTP Request node.                                                                                    |
| Sticky Note4          | Sticky Note                   | Explanation of Raw Text extraction node              | (none)                   | (none)                  | Describes role of extracting plain OCR text for AI input.                                                                                 |
| Sticky Note5          | Sticky Note                   | Explanation of AI Agent node                          | (none)                   | (none)                  | Describes AI processing role, prompt structure, and key configuration.                                                                    |
| Sticky Note6          | Sticky Note                   | Explanation of Clean AI Output node                   | (none)                   | (none)                  | Describes cleaning and parsing AI output JSON.                                                                                            |
| Sticky Note7          | Sticky Note                   | Explanation of email check (If) node                  | (none)                   | (none)                  | Explains condition to verify presence of extracted email.                                                                                 |
| Sticky Note8          | Sticky Note                   | Explanation of Google Sheets append node              | (none)                   | (none)                  | Setup instructions for saving data to Google Sheets.                                                                                      |
| Sticky Note9          | Sticky Note                   | Explanation of Gmail send message node                | (none)                   | (none)                  | Explains Gmail node configuration for optional email notification.                                                                        |
| Sticky Note11         | Sticky Note                   | Project credits and author info                        | (none)                   | (none)                  | Credits to Aditya Malur with LinkedIn link.                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: message  
     - Enable "Download" to true  
   - Connect your Telegram bot with BotFather token and set webhook URL as provided by n8n.

2. **Add Code node "Convert Image to Base64":**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const binaryData = items[0].binary.data;
     const base64Image = binaryData.data.toString('base64');
     items[0].json.base64 = base64Image;
     return items;
     ```  
   - Connect output of Telegram Trigger to this node.

3. **Add HTTP Request node for Google Vision OCR:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://vision.googleapis.com/v1/images:annotate?key=YOUR_GOOGLE_VISION_API_KEY` (replace with your key)  
   - Headers: `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "requests": [
         {
           "image": {
             "content": "{{ $json.base64 }}"
           },
           "features": [
             {
               "type": "TEXT_DETECTION"
             }
           ]
         }
       ]
     }
     ```  
   - Connect "Convert Image to Base64" output here.

4. **Add Code node "Extract Raw Text)":**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const rawText = $json.responses?.[0]?.fullTextAnnotation?.text || "";
     return [{ json: { raw_text: rawText } }];
     ```  
   - Connect output of HTTP Request node.

5. **Add LangChain AI Agent node:**  
   - Type: LangChain AI Agent  
   - Parameters:  
     - Text: Use a prompt that includes:  
       - Raw OCR text: `{{ $json.raw_text }}`  
       - Telegram caption: `{{ $('Telegram Trigger').item.json.message.caption }}`  
       - Your personal details (name, title, email, phone, LinkedIn, Calendly, positioning) as provided in the original prompt  
       - Detailed instructions to extract structured fields, generate summaries, email drafts, etc., as per the original workflow’s prompt.  
     - System message: Set to instruct the AI to output strict JSON in the defined schema, factual, concise, structured.  
   - Connect "Extract Raw Text)" output here.

6. **Set up OpenAI Chat Model node:**  
   - Type: LangChain OpenAI Chat Model  
   - Parameters:  
     - Model: Select `gpt-4.1-mini`  
   - Connect this node as the `ai_languageModel` resource input of AI Agent node.

7. **Add Code node "Clean AI Output":**  
   - Type: Code  
   - JavaScript code as per original to:  
     - Strip markdown/code fences  
     - Parse JSON output strictly  
     - Normalize emails and phones arrays  
     - Extract and prepare subject and htmlBody fields  
   - Connect output of AI Agent node here.

8. **Add If node "Check for Email Found":**  
   - Type: If Condition  
   - Condition: Check if `{{$json.emails[0]}}` is not empty  
   - Connect output of "Clean AI Output" here.

9. **Add Google Sheets node "Append row in sheet":**  
   - Type: Google Sheets  
   - Operation: Append  
   - Connect Google OAuth2 credentials for your Google account  
   - Parameters:  
     - Document ID: your Google Sheet ID  
     - Sheet Name: e.g., "Sheet1"  
     - Map columns to JSON fields: Name, Email, Country, Website, Fit Score, Email Body, Fit Reason, User Input (Telegram caption), Company, Next Actions (3 items), Phone Number, Email Subject, Opportunities (up to 5), Role, Alternate Email, WhatsApp Draft, Opportunities Summary, Alternate Phone Number  
   - Connect "true" output of If node here.

10. **Add Gmail node "Send a message":**  
    - Type: Gmail  
    - Connect Gmail OAuth2 credentials  
    - Parameters:  
      - Send To: your email (e.g., hi@adityamalur.com)  
      - Subject: `{{$json['Email Subject']}}`  
      - Message: `{{$json['Email Body']}}` (HTML)  
    - Connect output of Google Sheets node here.

11. **Wire nodes accordingly:**  
    - Telegram Trigger → Convert Image to Base64 → HTTP Request → Extract Raw Text) → AI Agent → Clean AI Output → If  
    - If (true) → Append row in sheet → Send a message  
    - AI Agent node connected to OpenAI Chat Model as AI resource  

12. **Test the workflow:**  
    - Send a business card photo with optional caption on Telegram bot.  
    - Check logs for errors (e.g., API keys, parsing errors).  
    - Confirm data appends in Google Sheets and email receipt.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                               |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow created by Aditya Malur, AI Powered Mentor & Consultant. LinkedIn: https://www.linkedin.com/in/aditya-malur/ | Project author and contact                                      |
| To connect your Telegram bot with n8n, use BotFather to create a bot and set webhook with n8n URL. | Telegram bot setup instructions                                |
| Google Vision API key required for OCR; ensure API billing and quota are configured.                | https://cloud.google.com/vision/docs/quickstart                |
| OpenAI API key required for GPT-4 Mini usage; manage rate limits and costs accordingly.             | https://platform.openai.com/docs/api-reference                 |
| Google Sheets OAuth2 credentials must have write access to target spreadsheet.                      | Google Sheets API and OAuth2 setup                             |
| Gmail OAuth2 credentials required to send emails via Gmail node.                                   | Gmail API setup with OAuth2                                    |
| AI prompt enforces strict JSON output; failure to comply will cause workflow errors at parsing stage. | Important for AI Agent prompt design                           |
| Phone numbers are cleaned and prefixed with apostrophes to preserve formatting in Sheets.           | Data normalization detail                                      |
| Workflow includes detailed sticky notes explaining each block and node setup for maintainability.  | Embedded documentation inside workflow                         |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow designed using n8n, adhering to all applicable content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.