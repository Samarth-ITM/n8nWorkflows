Scan business cards from LINE with Gemini to Sheets, Slack, and Gmail

https://n8nworkflows.xyz/workflows/scan-business-cards-from-line-with-gemini-to-sheets--slack--and-gmail-14212


# Scan business cards from LINE with Gemini to Sheets, Slack, and Gmail

# 1. Workflow Overview

This workflow receives a message from a LINE bot, checks whether the message contains an image of a business card, sends that image to Google Gemini for OCR-style structured extraction, stores the extracted contact data in Google Sheets, notifies a Slack channel, optionally sends a thank-you email through Gmail, and finally replies to the LINE user.

Its main use case is lightweight contact capture from business cards, especially for LINE-centric workflows. It is designed for semi-automatic CRM intake: a user sends a business card photo to a LINE bot, and the workflow turns that image into structured contact data plus internal and external follow-up actions.

## 1.1 Input Reception and Immediate Acknowledgement

The workflow starts from a LINE webhook. It immediately returns `OK` to the webhook caller using a dedicated response node, while the rest of the workflow continues asynchronously.

## 1.2 Input Validation and Runtime Configuration

After receiving the request, the workflow loads several configurable values such as the LINE channel access token, Slack channel, Google Sheet ID, and reply messages. It then validates that the incoming LINE message is an image.

## 1.3 Image Retrieval and AI Extraction

If the message is an image, the workflow downloads the image binary from LINE and submits it to a Gemini-powered LangChain node that is prompted to return only structured JSON representing business card fields.

## 1.4 JSON Cleanup and Spreadsheet Persistence

Because LLM responses may include markdown fences or invalid JSON formatting, a Code node sanitizes and parses the Gemini output. The resulting structured fields are appended as a new row in Google Sheets.

## 1.5 Internal Notification, Optional Email, and Final LINE Reply

Once the row is saved, the workflow sends a Slack notification to the team. It then checks whether an email address was extracted. If so, it sends a thank-you email through Gmail. Finally, it replies to the original LINE user confirming successful registration. If the original LINE message was not an image, the workflow instead replies asking the user to send a business card image.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Receive & validate

**Overview:**  
This block receives the LINE webhook payload, responds immediately to avoid webhook timeout issues, loads workflow configuration values, and checks whether the message type is `image`. It is the entry point and gatekeeper for all downstream processing.

**Nodes Involved:**  
- LINE Webhook  
- Respond to LINE immediately  
- Set config values  
- Is image message?  
- Reply no-image message on LINE

### Node: LINE Webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for POST requests from the LINE Messaging API.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `line-webhook`
  - Response mode: `responseNode`, meaning a separate Respond to Webhook node controls the HTTP response
- **Key expressions or variables used:**  
  Downstream nodes read values from:
  - `$('LINE Webhook').item.json.body.events[0].message.type`
  - `$('LINE Webhook').item.json.body.events[0].message.id`
  - `$('LINE Webhook').item.json.body.events[0].replyToken`
- **Input and output connections:**  
  - No input node
  - Outputs to:
    - Respond to LINE immediately
    - Set config values
- **Version-specific requirements:**  
  Uses `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - LINE may send payloads with no `events[0]`
  - Non-message events may not include `message`
  - Signature validation is not implemented here; webhook authenticity depends on external setup
  - If LINE expects a fast response, any missing Respond node could cause retries
- **Sub-workflow reference:** None

### Node: Respond to LINE immediately

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an immediate HTTP response back to LINE.
- **Configuration choices:**  
  - Responds with text
  - Response body: `OK`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from LINE Webhook
  - No downstream output
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Only works when the webhook node is configured with `responseMode: responseNode`
  - If omitted or misconfigured, LINE may consider the webhook failed
- **Sub-workflow reference:** None

### Node: Set config values

- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a configuration object used by downstream nodes.
- **Configuration choices:**  
  Assigns the following values:
  - `config.lineChannelAccessToken`
  - `config.slackChannel`
  - `config.googleSheetId`
  - `config.thankYouEmailSubject`
  - `config.replyMessageSuccess`
  - `config.replyMessageNoImage`
- **Key expressions or variables used:**  
  Produces:
  - `$('Set config values').item.json.config.lineChannelAccessToken`
  - `$('Set config values').item.json.config.slackChannel`
  - `$('Set config values').item.json.config.googleSheetId`
  - etc.
- **Input and output connections:**  
  - Input from LINE Webhook
  - Output to Is image message?
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - Placeholder values (`User Key`) must be replaced
  - Storing secrets in a Set node is functional but not ideal for security; credentials or environment variables are safer
- **Sub-workflow reference:** None

### Node: Is image message?

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the LINE message type equals `image`.
- **Configuration choices:**  
  Condition:
  - Left value: `{{ $('LINE Webhook').item.json.body.events[0].message.type }}`
  - Operator: equals
  - Right value: `image`
- **Key expressions or variables used:**  
  `$('LINE Webhook').item.json.body.events[0].message.type`
- **Input and output connections:**  
  - Input from Set config values
  - True output to Download image from LINE
  - False output to Reply no-image message on LINE
- **Version-specific requirements:**  
  Uses `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If `events[0].message.type` is missing, the expression may fail
  - LINE events such as follow/join/postback will not match the expected structure
- **Sub-workflow reference:** None

### Node: Reply no-image message on LINE

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls LINE’s reply API to tell the user to send an image.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.line.me/v2/bot/message/reply`
  - Raw JSON body including:
    - `replyToken` from the webhook
    - Text message from `config.replyMessageNoImage`
  - Headers:
    - `Authorization: Bearer <LINE token>`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `$('LINE Webhook').item.json.body.events[0].replyToken`
  - `$('Set config values').item.json.config.replyMessageNoImage`
  - `$('Set config values').item.json.config.lineChannelAccessToken`
- **Input and output connections:**  
  - Input from false branch of Is image message?
  - No downstream output
- **Version-specific requirements:**  
  Uses `typeVersion: 4.2`
- **Edge cases or potential failure types:**  
  - Reply tokens expire quickly; delayed executions may fail
  - Invalid LINE token returns auth errors
  - Malformed raw JSON body causes API rejection
- **Sub-workflow reference:** None

---

## 2.2 Block: Extract contact data

**Overview:**  
This block downloads the original image binary from LINE and sends it to Gemini Vision via a LangChain chain node. The prompt enforces a strict JSON schema for business card extraction.

**Nodes Involved:**  
- Download image from LINE  
- Google Gemini Chat Model  
- Extract business card data with Gemini

### Node: Download image from LINE

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the image content associated with the LINE message.
- **Configuration choices:**  
  - URL built from LINE message ID:
    `https://api-data.line.me/v2/bot/message/{messageId}/content`
  - Response format: `file`
  - Authorization header uses the LINE channel access token
- **Key expressions or variables used:**  
  - `$('LINE Webhook').item.json.body.events[0].message.id`
  - `$('Set config values').item.json.config.lineChannelAccessToken`
- **Input and output connections:**  
  - Input from true branch of Is image message?
  - Output to Extract business card data with Gemini
- **Version-specific requirements:**  
  Uses `typeVersion: 4.2`
- **Edge cases or potential failure types:**  
  - Invalid or expired message ID
  - LINE token auth failure
  - Binary data retrieval issues
  - If LINE sends a non-image payload despite the check, Gemini processing may fail later
- **Sub-workflow reference:** None

### Node: Google Gemini Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Provides the Gemini language/vision model backend for the chain node.
- **Configuration choices:**  
  - Minimal explicit options shown
  - Used as the language model connection for the extraction chain
- **Key expressions or variables used:** None directly
- **Input and output connections:**  
  - AI language model connection into Extract business card data with Gemini
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
  - Requires the n8n LangChain nodes package and a valid Gemini credential/API key
- **Edge cases or potential failure types:**  
  - Invalid Gemini API key
  - Model quota/rate limit
  - Regional availability restrictions
  - Vision/image handling support depends on selected model and credential setup
- **Sub-workflow reference:** None

### Node: Extract business card data with Gemini

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Sends the image and instruction prompt to Gemini to extract structured business card data.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Text prompt instructs the model to return only a valid JSON object
  - Required fields:
    - `full_name`
    - `company`
    - `department`
    - `title`
    - `email`
    - `phone`
    - `address`
    - `website`
    - `notes`
  - Includes additional rules:
    - Empty string for missing values
    - Include country code for phone if visible
    - Romanize Japanese names if only kanji/kana is present
    - Return only JSON
  - Messages configuration includes `HumanMessagePromptTemplate` with `imageBinary`
- **Key expressions or variables used:**  
  - Prompt contains `{{ $json.data }}`  
    This appears intended to reference the binary/image context, but the actual image transfer is primarily handled by the `imageBinary` message type.
- **Input and output connections:**  
  - Main input from Download image from LINE
  - AI model input from Google Gemini Chat Model
  - Output to Parse Gemini JSON response
- **Version-specific requirements:**  
  Uses `typeVersion: 1.9`
  - Requires compatible LangChain/Gemini node versions that support image binary messages
- **Edge cases or potential failure types:**  
  - Model may still return markdown or prose despite the prompt
  - OCR quality depends on image quality, orientation, glare, and multilingual layout
  - `{{ $json.data }}` may not contain meaningful text content depending on upstream binary handling
  - If the model does not support the image modality, extraction fails
- **Sub-workflow reference:** None

---

## 2.3 Block: Save to Sheets

**Overview:**  
This block cleans Gemini’s response, parses it into JSON, and appends the resulting contact data into a target Google Sheet. It transforms potentially unstable LLM output into a consistent persistence format.

**Nodes Involved:**  
- Parse Gemini JSON response  
- Save contact to Google Sheets

### Node: Parse Gemini JSON response

- **Type and technical role:** `n8n-nodes-base.code`  
  Sanitizes the model output and attempts JSON parsing.
- **Configuration choices:**  
  JavaScript logic:
  - Reads raw text from `$input.item.json.text`
  - Removes markdown code fences such as ```json
  - Parses JSON
  - On parse failure, returns a fallback object with blank fields and `notes` containing the raw response
- **Key expressions or variables used:**  
  - `$input.item.json.text`
- **Input and output connections:**  
  - Input from Extract business card data with Gemini
  - Output to Save contact to Google Sheets
- **Version-specific requirements:**  
  Uses `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If the Gemini node returns a different field than `text`, parsing will fail
  - Partial or malformed JSON becomes a fallback record
  - Parse failure does not stop the workflow; it stores a mostly empty row with diagnostic notes
- **Sub-workflow reference:** None

### Node: Save contact to Google Sheets

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the extracted contact data as a new row.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet name: `シート1`
  - Document ID from `config.googleSheetId`
  - Explicit column mapping:
    - Timestamp
    - Full name
    - Company
    - Department
    - Title
    - Email
    - Phone
    - Address
    - Website
    - Notes
  - Timestamp uses current time formatted as `yyyy-MM-dd HH:mm:ss`
- **Key expressions or variables used:**  
  - `{{ $json.full_name }}`
  - `{{ $json.company }}`
  - `{{ $json.department }}`
  - `{{ $json.title }}`
  - `{{ $json.email }}`
  - `{{ $json.phone }}`
  - `{{ $json.address }}`
  - `{{ $json.website }}`
  - `{{ $json.notes }}`
  - `{{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`
  - `{{ $('Set config values').item.json.config.googleSheetId }}`
- **Input and output connections:**  
  - Input from Parse Gemini JSON response
  - Outputs to:
    - Notify team on Slack
    - Has email address?
- **Version-specific requirements:**  
  Uses `typeVersion: 4.5`
  - Requires Google Sheets credentials with write access
- **Edge cases or potential failure types:**  
  - Spreadsheet ID may be wrong
  - Sheet `シート1` must exist
  - Column names must match the configured mapping
  - OAuth/service-account permissions must allow append
  - Timestamp timezone depends on n8n instance settings
- **Sub-workflow reference:** None

---

## 2.4 Block: Notify & confirm

**Overview:**  
This block informs the internal team through Slack, checks whether an email address is available, optionally triggers an external thank-you email, and confirms success back to the LINE user.

**Nodes Involved:**  
- Notify team on Slack  
- Has email address?  
- Send thank-you email via Gmail  
- Reply success message on LINE

### Node: Notify team on Slack

- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a formatted summary of the captured contact into a Slack channel.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Select by channel name
  - Channel ID/name comes from `config.slackChannel`
  - Message includes:
    - Full name
    - Company
    - Title
    - Email
    - Phone
    - Google Sheets link
- **Key expressions or variables used:**  
  - `$('Parse Gemini JSON response').item.json.full_name`
  - `$('Parse Gemini JSON response').item.json.company`
  - `$('Parse Gemini JSON response').item.json.title`
  - `$('Parse Gemini JSON response').item.json.email`
  - `$('Parse Gemini JSON response').item.json.phone`
  - `$('Set config values').item.json.config.googleSheetId`
  - `$('Set config values').item.json.config.slackChannel`
- **Input and output connections:**  
  - Input from Save contact to Google Sheets
  - Output to Send thank-you email via Gmail
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**  
  - Channel name may not resolve correctly
  - OAuth scopes may be insufficient
  - Message formatting errors if fields are null/undefined
  - The extra connection to Gmail means this node can trigger email independently of the IF branch
- **Sub-workflow reference:** None

### Node: Has email address?

- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the extracted email field is non-empty.
- **Configuration choices:**  
  Condition:
  - Left value: `{{ $('Parse Gemini JSON response').item.json.email }}`
  - Operator: not equals
  - Right value: empty string
- **Key expressions or variables used:**  
  `$('Parse Gemini JSON response').item.json.email`
- **Input and output connections:**  
  - Input from Save contact to Google Sheets
  - True output to Send thank-you email via Gmail
  - False output to Reply success message on LINE
- **Version-specific requirements:**  
  Uses `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Whitespace-only emails pass unless trimmed elsewhere
  - Invalid email syntax still passes if non-empty
- **Sub-workflow reference:** None

### Node: Send thank-you email via Gmail

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a thank-you email to the extracted address.
- **Configuration choices:**  
  - Recipient: `{{ $json.Email }}`
  - Subject from `config.thankYouEmailSubject`
  - Plain text email body addressing the extracted full name
- **Key expressions or variables used:**  
  - `{{ $json.Email }}`
  - `{{ $('Parse Gemini JSON response').item.json.full_name }}`
  - `{{ $('Set config values').item.json.config.thankYouEmailSubject }}`
- **Input and output connections:**  
  - Inputs from:
    - Notify team on Slack
    - True branch of Has email address?
  - Output to Reply success message on LINE
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`
  - Requires Gmail OAuth credentials with send permission
- **Edge cases or potential failure types:**  
  - Important design issue: this node uses `{{ $json.Email }}` from the current item, which works if the incoming item is from Google Sheets append output and contains `Email`, but may fail or behave differently depending on whether the triggering input is from Slack or IF branch data shape
  - The node has two incoming paths; depending on execution semantics, it may execute twice or with unexpected item structure
  - Invalid email format can cause Gmail API rejection
  - Gmail sending quotas or OAuth issues may block delivery
- **Sub-workflow reference:** None

### Node: Reply success message on LINE

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a LINE reply confirming the business card was registered.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.line.me/v2/bot/message/reply`
  - Raw JSON body with:
    - `replyToken`
    - Text from `config.replyMessageSuccess`
  - Headers:
    - `Authorization: Bearer <LINE token>`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `$('LINE Webhook').item.json.body.events[0].replyToken`
  - `$('Set config values').item.json.config.replyMessageSuccess`
  - `$('Set config values').item.json.config.lineChannelAccessToken`
- **Input and output connections:**  
  - Inputs from:
    - False branch of Has email address?
    - Send thank-you email via Gmail
  - No downstream output
- **Version-specific requirements:**  
  Uses `typeVersion: 4.2`
- **Edge cases or potential failure types:**  
  - Reply token can only be used once and expires quickly
  - Because the workflow already sends an immediate webhook response, this later LINE reply is a separate API call and must happen within LINE’s reply-token validity window
  - If both branches or duplicate executions reach this node, LINE may reject repeated use of the same reply token
- **Sub-workflow reference:** None

---

## 2.5 Documentation / annotation nodes

**Overview:**  
These nodes are sticky notes used for visual explanation and operational guidance inside the n8n canvas. They do not affect execution but are important for maintainability.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6

### Node: Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation block containing workflow purpose, setup steps, and customization hints.
- **Configuration choices:**  
  Large introductory note covering the whole workflow.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

### Node: Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes the “Receive & validate” area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes the “Extract contact data” area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes the “Save to Sheets” area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes the notification and confirmation area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note5

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Highlights the Gmail sending behavior.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note6

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Highlights the Slack notification behavior.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| LINE Webhook | Webhook | Receives POST events from LINE Messaging API |  | Respond to LINE immediately; Set config values | Got a stack of meishi from your last networking event? Just snap a photo and send it to LINE — this workflow handles the rest. |
| Respond to LINE immediately | Respond to Webhook | Returns immediate `OK` HTTP response to LINE | LINE Webhook |  | ## Receive & validate |
| Set config values | Set | Defines runtime configuration values and user-facing messages | LINE Webhook | Is image message? | ## Receive & validate |
| Is image message? | If | Checks whether the incoming LINE message is an image | Set config values | Download image from LINE; Reply no-image message on LINE | ## Receive & validate |
| Download image from LINE | HTTP Request | Downloads the original image binary from LINE content API | Is image message? | Extract business card data with Gemini | ## Extract contact data |
| Google Gemini Chat Model | Google Gemini Chat Model | Provides the Gemini model used by the extraction chain |  | Extract business card data with Gemini | ## Extract contact data |
| Extract business card data with Gemini | LangChain LLM Chain | Sends image + prompt to Gemini and requests strict JSON extraction | Download image from LINE; Google Gemini Chat Model | Parse Gemini JSON response | ## Extract contact data |
| Parse Gemini JSON response | Code | Removes markdown fences and parses Gemini output into JSON | Extract business card data with Gemini | Save contact to Google Sheets | ## Save to Sheets |
| Save contact to Google Sheets | Google Sheets | Appends extracted contact data as a new row | Parse Gemini JSON response | Notify team on Slack; Has email address? | ## Save to Sheets |
| Notify team on Slack | Slack | Posts a summary of the new contact to Slack | Save contact to Google Sheets | Send thank-you email via Gmail | ## Notify & confirm |
| Has email address? | If | Checks whether an extracted email exists | Save contact to Google Sheets | Send thank-you email via Gmail; Reply success message on LINE | ## Notify & confirm |
| Send thank-you email via Gmail | Gmail | Sends a thank-you email to the extracted contact | Notify team on Slack; Has email address? | Reply success message on LINE | ## Notify & confirm |
| Reply success message on LINE | HTTP Request | Sends success confirmation back to the LINE user | Has email address?; Send thank-you email via Gmail |  | ## Notify & confirm |
| Reply no-image message on LINE | HTTP Request | Sends a fallback message when the LINE event is not an image | Is image message? |  | ## Receive & validate |
| Sticky Note | Sticky Note | General workflow description, setup, and customization guidance |  |  | Got a stack of meishi from your last networking event? Just snap a photo and send it to LINE — this workflow handles the rest. |
| Sticky Note1 | Sticky Note | Visual annotation for input validation block |  |  | ## Receive & validate |
| Sticky Note2 | Sticky Note | Visual annotation for extraction block |  |  | ## Extract contact data |
| Sticky Note3 | Sticky Note | Visual annotation for Sheets block |  |  | ## Save to Sheets |
| Sticky Note4 | Sticky Note | Visual annotation for notify/confirm block |  |  | ## Notify & confirm |
| Sticky Note5 | Sticky Note | Visual annotation for Gmail behavior |  |  | ## 📧 Send Email\nIf an email address was found on the business card, Gmail automatically sends a thank-you email to the new contact. |
| Sticky Note6 | Sticky Note | Visual annotation for Slack behavior |  |  | ## 🔔 Slack Notification\nPosts the extracted contact details to a Slack channel to keep the whole team in the loop. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Scan business cards with LINE and Gemini, then send thank-you emails via Gmail`.
   - In workflow settings, keep binary mode compatible with separate binary storage if your instance uses that mode.

2. **Add the LINE trigger node**
   - Add a **Webhook** node.
   - Name it `LINE Webhook`.
   - Set:
     - **HTTP Method**: `POST`
     - **Path**: `line-webhook`
     - **Response Mode**: `Using Respond to Webhook Node`
   - Save the node and copy its test/production webhook URL.
   - In the LINE Developer Console, configure your Messaging API webhook to use this URL.

3. **Add the immediate webhook response**
   - Add a **Respond to Webhook** node.
   - Name it `Respond to LINE immediately`.
   - Set:
     - **Respond With**: `Text`
     - **Response Body**: `OK`
   - Connect `LINE Webhook -> Respond to LINE immediately`.

4. **Add a configuration node**
   - Add a **Set** node.
   - Name it `Set config values`.
   - Create string fields:
     - `config.lineChannelAccessToken`
     - `config.slackChannel`
     - `config.googleSheetId`
     - `config.thankYouEmailSubject`
     - `config.replyMessageSuccess`
     - `config.replyMessageNoImage`
   - Example values:
     - `config.lineChannelAccessToken`: your LINE channel access token
     - `config.slackChannel`: Slack channel name
     - `config.googleSheetId`: target spreadsheet ID
     - `config.thankYouEmailSubject`: e.g. `Thank you for meeting us`
     - `config.replyMessageSuccess`: `名刺を登録しました！Google Sheetsに保存されています。`
     - `config.replyMessageNoImage`: `名刺の画像を送ってください。`
   - Connect `LINE Webhook -> Set config values`.

5. **Add message-type validation**
   - Add an **If** node.
   - Name it `Is image message?`.
   - Configure a string condition:
     - Left value: `{{ $('LINE Webhook').item.json.body.events[0].message.type }}`
     - Operation: `equals`
     - Right value: `image`
   - Connect `Set config values -> Is image message?`.

6. **Add fallback reply for non-image input**
   - Add an **HTTP Request** node.
   - Name it `Reply no-image message on LINE`.
   - Set:
     - **Method**: `POST`
     - **URL**: `https://api.line.me/v2/bot/message/reply`
     - **Send Headers**: enabled
     - Headers:
       - `Authorization`: `Bearer {{ $('Set config values').item.json.config.lineChannelAccessToken }}`
       - `Content-Type`: `application/json`
     - **Send Body**: enabled
     - **Content Type**: `Raw`
     - **Raw Content Type**: `application/json`
     - **Body**:
       ```json
       {
         "replyToken": "{{ $('LINE Webhook').item.json.body.events[0].replyToken }}",
         "messages": [
           {
             "type": "text",
             "text": "{{ $('Set config values').item.json.config.replyMessageNoImage }}"
           }
         ]
       }
       ```
   - Connect the **false** output of `Is image message?` to this node.

7. **Add LINE image download**
   - Add an **HTTP Request** node.
   - Name it `Download image from LINE`.
   - Set:
     - **Method**: `GET`
     - **URL**:
       `https://api-data.line.me/v2/bot/message/{{ $('LINE Webhook').item.json.body.events[0].message.id }}/content`
     - **Send Headers**: enabled
     - Header:
       - `Authorization`: `Bearer {{ $('Set config values').item.json.config.lineChannelAccessToken }}`
     - **Response Format**: `File`
   - Connect the **true** output of `Is image message?` to this node.

8. **Add Gemini model node**
   - Add a **Google Gemini Chat Model** node from the LangChain nodes.
   - Name it `Google Gemini Chat Model`.
   - Create or attach Gemini credentials using a Google Gemini API key.
   - Leave default options unless you need a specific model variant that supports image input.

9. **Add the Gemini extraction chain**
   - Add a **Basic LLM Chain / Chain LLM** node from the LangChain nodes.
   - Name it `Extract business card data with Gemini`.
   - Set prompt mode to define the prompt manually.
   - Paste a prompt equivalent to:
     - You are a business card OCR assistant.
     - Return only a valid JSON object.
     - Include fields:
       `full_name`, `company`, `department`, `title`, `email`, `phone`, `address`, `website`, `notes`
     - Use empty strings for missing values.
     - Romanize Japanese names if needed.
   - In the messages section, add a human message configured as **image binary**.
   - Connect:
     - `Download image from LINE -> Extract business card data with Gemini`
     - `Google Gemini Chat Model -> Extract business card data with Gemini` using the AI language model connection.

10. **Add JSON parsing and cleanup**
    - Add a **Code** node.
    - Name it `Parse Gemini JSON response`.
    - Use JavaScript like:
      ```javascript
      const raw = $input.item.json.text;
      const cleaned = raw.replace(/```json\\n?/g, '').replace(/```/g, '').trim();
      try {
        const parsed = JSON.parse(cleaned);
        return { json: parsed };
      } catch(e) {
        return { json: {
          full_name: '',
          company: '',
          department: '',
          title: '',
          email: '',
          phone: '',
          address: '',
          website: '',
          notes: 'Parse error: ' + raw
        }};
      }
      ```
    - Connect `Extract business card data with Gemini -> Parse Gemini JSON response`.

11. **Prepare the Google Sheet**
    - In Google Sheets, create a spreadsheet with a worksheet named `シート1`.
    - Create columns:
      - Timestamp
      - Full name
      - Company
      - Department
      - Title
      - Email
      - Phone
      - Address
      - Website
      - Notes
    - Share it with the credentialed account if using service account auth.

12. **Add Google Sheets append node**
    - Add a **Google Sheets** node.
    - Name it `Save contact to Google Sheets`.
    - Credentials: Google Sheets OAuth2 or service account.
    - Set:
      - **Operation**: `Append`
      - **Document ID**: `{{ $('Set config values').item.json.config.googleSheetId }}`
      - **Sheet Name**: `シート1`
      - Map fields explicitly:
        - Timestamp → `{{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`
        - Full name → `{{ $json.full_name }}`
        - Company → `{{ $json.company }}`
        - Department → `{{ $json.department }}`
        - Title → `{{ $json.title }}`
        - Email → `{{ $json.email }}`
        - Phone → `{{ $json.phone }}`
        - Address → `{{ $json.address }}`
        - Website → `{{ $json.website }}`
        - Notes → `{{ $json.notes }}`
    - Connect `Parse Gemini JSON response -> Save contact to Google Sheets`.

13. **Add Slack notification**
    - Add a **Slack** node.
    - Name it `Notify team on Slack`.
    - Credentials: Slack OAuth2 with permission to post messages.
    - Configure message posting to a channel selected by name.
    - Set channel value to:
      `{{ $('Set config values').item.json.config.slackChannel }}`
    - Set message text similar to:
      - New business card registered
      - Include full name, company, title, email, phone
      - Include a Sheets URL using spreadsheet ID
    - Connect `Save contact to Google Sheets -> Notify team on Slack`.

14. **Add email-presence check**
    - Add an **If** node.
    - Name it `Has email address?`.
    - Configure:
      - Left value: `{{ $('Parse Gemini JSON response').item.json.email }}`
      - Operation: `not equals`
      - Right value: empty string
    - Connect `Save contact to Google Sheets -> Has email address?`.

15. **Add Gmail send node**
    - Add a **Gmail** node.
    - Name it `Send thank-you email via Gmail`.
    - Credentials: Gmail OAuth2 with send permission.
    - Set:
      - **To**: `{{ $json.Email }}`
      - **Subject**: `{{ $('Set config values').item.json.config.thankYouEmailSubject }}`
      - **Email Type**: `Text`
      - **Body**:
        `Dear {{ $('Parse Gemini JSON response').item.json.full_name }}, ...`
    - Connect:
      - True output of `Has email address? -> Send thank-you email via Gmail`
      - `Notify team on Slack -> Send thank-you email via Gmail`
   - **Important:** This exact structure exists in the source workflow, but it is risky because the Gmail node has two incoming connections and depends on `{{ $json.Email }}`. A safer rebuild is:
      - either connect only from `Has email address?`
      - or ensure the incoming item always contains a field named `Email`
   - If you want to reproduce the source exactly, keep both incoming connections.

16. **Add success reply to LINE**
    - Add an **HTTP Request** node.
    - Name it `Reply success message on LINE`.
    - Set:
      - **Method**: `POST`
      - **URL**: `https://api.line.me/v2/bot/message/reply`
      - **Send Headers**: enabled
      - Headers:
        - `Authorization`: `Bearer {{ $('Set config values').item.json.config.lineChannelAccessToken }}`
        - `Content-Type`: `application/json`
      - **Send Body**: enabled
      - **Content Type**: `Raw`
      - **Raw Content Type**: `application/json`
      - **Body**:
        ```json
        {
          "replyToken": "{{ $('LINE Webhook').item.json.body.events[0].replyToken }}",
          "messages": [
            {
              "type": "text",
              "text": "{{ $('Set config values').item.json.config.replyMessageSuccess }}"
            }
          ]
        }
        ```
    - Connect:
      - False output of `Has email address? -> Reply success message on LINE`
      - `Send thank-you email via Gmail -> Reply success message on LINE`

17. **Add optional sticky notes for maintainability**
    - Create sticky notes for:
      - Receive & validate
      - Extract contact data
      - Save to Sheets
      - Notify & confirm
      - Gmail behavior
      - Slack behavior
    - Also add a large overview note with setup steps and customization guidance if you want parity with the source workflow.

18. **Configure credentials**
    - **LINE**
      - This workflow stores the access token in the Set node, not in a credential object.
      - Ensure the bot has Messaging API enabled.
      - Paste the n8n webhook URL into LINE Developers.
    - **Google Gemini**
      - Create Gemini API credentials in n8n.
      - Use a model that supports image input.
    - **Google Sheets**
      - Use OAuth2 or service account credentials.
      - Ensure write access to the spreadsheet.
    - **Slack**
      - Use OAuth2 with channel posting access.
      - Make sure the bot/app is invited to the target channel.
    - **Gmail**
      - Use OAuth2 with mail send permissions.

19. **Test with real inputs**
    - First send a non-image LINE message and verify the fallback reply.
    - Then send a business card image and verify:
      - webhook acknowledges immediately
      - image downloads
      - Gemini extracts fields
      - Sheets row is appended
      - Slack message is posted
      - Gmail sends if email exists
      - LINE success reply is sent

20. **Recommended hardening adjustments**
    - Add a guard for missing `events[0]`
    - Validate email format before Gmail
    - Avoid double-triggering Gmail by removing one of its incoming connections
    - Consider using a **Push message** endpoint instead of a **Reply** endpoint if downstream processing may exceed LINE reply-token validity
    - Move secrets out of the Set node into credentials or environment variables

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Got a stack of meishi from your last networking event? Just snap a photo and send it to LINE — this workflow handles the rest. | General workflow positioning |
| The LINE webhook picks up your image, checks it's actually a photo, then passes it to Gemini Vision for OCR and structured extraction. | Workflow behavior summary |
| Extracted fields are appended to Google Sheets, Slack gets a summary, Gmail sends a thank-you email if possible, and LINE confirms the save. | Operational summary |
| Setup step: add LINE Messaging API credentials in n8n and paste the webhook URL into the LINE Developer Console. | LINE integration setup |
| Setup step: add a Google Gemini API key; the free tier may be sufficient. | Gemini setup |
| Setup step: share the target Google Sheet with the service account email, or connect via OAuth. | Google Sheets setup |
| Setup step: open the “Notify team on Slack” node and set your channel name. | Slack setup |
| Setup step: connect Gmail via OAuth in the credentials panel. | Gmail setup |
| Setup step: send a test business card photo to your LINE bot and check the Sheet. | Validation step |
| Customization: adjust the Gemini prompt to capture extra fields such as LinkedIn URL or department. | Prompt extension |
| Customization: adjust the Slack message or Gmail body to match your team tone. | Message customization |

## Important implementation note

The current graph contains a likely logic flaw:

- `Notify team on Slack` also connects to `Send thank-you email via Gmail`
- `Has email address?` also connects to `Send thank-you email via Gmail`

This means Gmail can be triggered from two paths, and the current `To` field uses `{{ $json.Email }}`, which depends on the exact incoming item structure. For a production-safe version, it is better to trigger Gmail only from `Has email address?` and use a stable expression such as `{{ $('Parse Gemini JSON response').item.json.email }}` for the recipient.