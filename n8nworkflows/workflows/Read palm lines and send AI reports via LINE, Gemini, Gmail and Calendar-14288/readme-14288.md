Read palm lines and send AI reports via LINE, Gemini, Gmail and Calendar

https://n8nworkflows.xyz/workflows/read-palm-lines-and-send-ai-reports-via-line--gemini--gmail-and-calendar-14288


# Read palm lines and send AI reports via LINE, Gemini, Gmail and Calendar

# 1. Workflow Overview

This workflow turns a LINE bot into an AI-powered palm reading service. When a user sends a palm photo to the LINE bot, the workflow downloads the image, sends it to Google Gemini for interpretation, extracts structured output from Gemini’s response, replies to the user in LINE, emails a detailed report through Gmail, and creates a Google Calendar event for suggested lucky days.

Primary use cases:
- AI-based palm image reading from LINE
- Automatic multi-channel delivery of results
- Lightweight automation that combines chat, vision analysis, email, and calendar registration

Logical blocks in the workflow:

## 1.1 Trigger & Configuration
The workflow starts from a LINE webhook, then normalizes incoming event data and stores configuration values such as the LINE access token, Gmail addresses, and Google Calendar ID for downstream reuse.

## 1.2 Message Type Validation
The workflow checks whether the incoming LINE message is an image. If not, it sends a friendly fallback message asking the user to upload a palm photo.

## 1.3 Image Retrieval & AI Analysis
If the message is an image, the workflow downloads the binary image from LINE’s content API and passes it to a Google Gemini chat model through the LangChain chain node using an image-aware prompt.

## 1.4 Response Parsing & Multi-Channel Delivery
Gemini’s text output is parsed into reusable fields. Then three parallel actions are launched:
- reply in LINE with a short summary,
- send the full reading via Gmail,
- create a Google Calendar event containing lucky day information.

## 1.5 Webhook Completion
A Respond to Webhook node returns a simple `OK` response to the original HTTP request.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Configuration

### Overview
This block receives the incoming POST request from LINE and transforms the raw webhook payload into named fields used throughout the workflow. It also acts as the central place for manual configuration of static values such as tokens and destination addresses.

### Nodes Involved
- Receive LINE webhook
- Set config variables

### Node Details

#### Receive LINE webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry-point node that receives HTTP POST requests from LINE.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `line-webhook`
  - Response mode: `responseNode`, meaning the workflow does not auto-respond and must use a Respond to Webhook node later.
- **Key expressions or variables used:** None in this node.
- **Input and output connections:**
  - No input; this is an entry point.
  - Output goes to **Set config variables**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - LINE webhook misconfiguration in LINE Developer Console
  - Wrong HTTP method
  - Workflow inactive, unreachable, or blocked by network restrictions
  - Payload shape differences causing downstream expression failures
- **Sub-workflow reference:** None.

#### Set config variables
- **Type and technical role:** `n8n-nodes-base.set`; creates normalized fields and stores static configuration values.
- **Configuration choices:**
  - Defines:
    - `LINE_CHANNEL_ACCESS_TOKEN`
    - `GMAIL_FROM`
    - `GMAIL_TO`
    - `GOOGLE_CALENDAR_ID`
    - `replyToken`
    - `messageType`
    - `imageId`
    - `userId`
  - Static values are placeholders and must be replaced manually.
  - Dynamic values are extracted from `body.events[0]` in the LINE payload.
- **Key expressions or variables used:**
  - `{{ $json.body.events[0].replyToken }}`
  - `{{ $json.body.events[0].message.type }}`
  - `{{ $json.body.events[0].message.id }}`
  - `{{ $json.body.events[0].source.userId }}`
- **Input and output connections:**
  - Input from **Receive LINE webhook**
  - Output to **Check if message is image**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If `events` is empty or missing, expressions fail
  - Non-message LINE events may not contain `message.type` or `message.id`
  - Placeholder config values left unchanged will break later API calls
  - `GMAIL_FROM` is stored but not actually used by the Gmail node in this workflow
- **Sub-workflow reference:** None.

---

## 2.2 Message Type Validation

### Overview
This block decides whether to process the request as a valid palm image or reject it with a friendly LINE message. It prevents unnecessary API calls to Gemini for unsupported message types.

### Nodes Involved
- Check if message is image
- Reply with error for non-image message

### Node Details

#### Check if message is image
- **Type and technical role:** `n8n-nodes-base.if`; conditional branch node.
- **Configuration choices:**
  - Checks whether `messageType` equals `image`.
  - Strict string comparison is enabled through the node’s condition settings.
- **Key expressions or variables used:**
  - `{{ $json.messageType }}`
- **Input and output connections:**
  - Input from **Set config variables**
  - True output to **Download image from LINE**
  - False output to **Reply with error for non-image message**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If `messageType` is undefined, the condition resolves false and routes to the non-image branch
  - Some LINE event types may not be message events at all
- **Sub-workflow reference:** None.

#### Reply with error for non-image message
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a reply message back to LINE’s reply API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.line.me/v2/bot/message/reply`
  - Sends JSON body with the LINE `replyToken`
  - Returns a fixed text prompting the user to send a palm photo
  - Uses Bearer authentication via request header
- **Key expressions or variables used:**
  - `{{ $('Set config variables').item.json.replyToken }}`
  - `{{ $('Set config variables').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
- **Input and output connections:**
  - Input from false branch of **Check if message is image**
  - No output connected to **Respond to webhook**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid or expired LINE channel access token
  - Reply token expiration; LINE reply tokens are short-lived
  - If this branch runs, the workflow currently does **not** connect to **Respond to webhook**, so the original webhook request may remain unacknowledged depending on execution behavior
- **Sub-workflow reference:** None.

---

## 2.3 Image Retrieval & AI Analysis

### Overview
This block downloads the uploaded image from LINE and submits it to Google Gemini with a structured prompt requesting three output sections: a short LINE-ready response, a detailed report, and lucky day suggestions.

### Nodes Involved
- Download image from LINE
- Google Gemini Chat Model
- Analyze palm lines with Gemini

### Node Details

#### Download image from LINE
- **Type and technical role:** `n8n-nodes-base.httpRequest`; fetches the binary image content from LINE’s content API.
- **Configuration choices:**
  - URL uses the LINE message ID
  - Response format is set to `file`
  - Authorization header uses Bearer token
- **Key expressions or variables used:**
  - `https://api-data.line.me/v2/bot/message/{{ $json.imageId }}/content`
  - `Bearer {{ $json.LINE_CHANNEL_ACCESS_TOKEN }}`
- **Input and output connections:**
  - Input from true branch of **Check if message is image**
  - Output to **Analyze palm lines with Gemini**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid access token
  - Expired or invalid message ID
  - LINE content API temporary failures
  - Large file or unsupported image format handling depends on Gemini/model capabilities
- **Sub-workflow reference:** None.

#### Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; provides the language/multimodal model used by the chain node.
- **Configuration choices:**
  - Temperature: `0.7`
  - Max output tokens: `1500`
  - Uses a Google Gemini API credential
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - AI language model connection into **Analyze palm lines with Gemini**
- **Version-specific requirements:**
  - Type version `1`
  - Requires the LangChain-compatible n8n AI nodes to be available in the instance
  - Requires a valid Google Gemini / PaLM API credential
- **Edge cases or potential failure types:**
  - Credential/authentication errors
  - Model quota exhaustion
  - Region or API availability limitations
  - Response truncation if output exceeds token limit
- **Sub-workflow reference:** None.

#### Analyze palm lines with Gemini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; orchestrates the prompt and image input sent to the Gemini model.
- **Configuration choices:**
  - Prompt instructs Gemini to act as a professional palm reader
  - Requires three explicitly labeled sections:
    - `[LINE reply (within 100 characters)]`
    - `[Detailed report]`
    - `[Lucky day suggestions]`
  - Uses `imageBinary` as a human message input
  - Prompt mode is `define`
- **Key expressions or variables used:**
  - The output parser later relies on the exact bracketed section headers, so prompt stability matters
- **Input and output connections:**
  - Main input from **Download image from LINE**
  - AI model input from **Google Gemini Chat Model**
  - Main output to **Parse Gemini response**
- **Version-specific requirements:** Type version `1.4`.
- **Edge cases or potential failure types:**
  - If Gemini does not follow the required section format, downstream regex extraction may fail or fallback defaults will be used
  - Vision analysis quality depends on image quality, orientation, lighting, and palm visibility
  - Prompt can produce non-date lucky day text that is not suitable for direct calendar automation
- **Sub-workflow reference:** None.

---

## 2.4 Response Parsing & Multi-Channel Delivery

### Overview
This block extracts useful text fields from Gemini’s output and distributes the results through three channels in parallel: LINE, Gmail, and Google Calendar.

### Nodes Involved
- Parse Gemini response
- Reply to LINE with summary
- Send detailed report via Gmail
- Register lucky days in Google Calendar

### Node Details

#### Parse Gemini response
- **Type and technical role:** `n8n-nodes-base.set`; prepares final text fields for downstream nodes.
- **Configuration choices:**
  - Stores full Gemini output in `geminiText`
  - Builds a formatted LINE message in `lineMessage`
  - Uses regex to extract the `[LINE reply ...]` section
  - Falls back to a default success message if the section is missing
- **Key expressions or variables used:**
  - `{{ $json.text }}`
  - `{{ $json.text.match(/\[LINE reply[^\]]*\]([\s\S]*?)(?=\[|$)/)?.[1]?.trim() ?? 'Your palm has been read! Check your email for the full report ✨' }}`
- **Input and output connections:**
  - Input from **Analyze palm lines with Gemini**
  - Outputs in parallel to:
    - **Reply to LINE with summary**
    - **Send detailed report via Gmail**
    - **Register lucky days in Google Calendar**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If `$json.text` is missing, expressions fail
  - Regex depends on Gemini preserving bracketed section names
  - If Gemini output is malformed, `lineMessage` may use fallback text
- **Sub-workflow reference:** None.

#### Reply to LINE with summary
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends the short summary back to the user via LINE.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.line.me/v2/bot/message/reply`
  - JSON body includes the original `replyToken`
  - Text message uses `lineMessage`
  - Authorization header uses stored LINE token
- **Key expressions or variables used:**
  - `{{ $('Set config variables').item.json.replyToken }}`
  - `{{ $json.lineMessage }}`
  - `{{ $('Set config variables').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
- **Input and output connections:**
  - Input from **Parse Gemini response**
  - Output to **Respond to webhook**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Reply token may expire if AI processing takes too long
  - LINE text length limits may matter if Gemini output is too verbose
  - Token/auth failures
- **Sub-workflow reference:** None.

#### Send detailed report via Gmail
- **Type and technical role:** `n8n-nodes-base.gmail`; sends an HTML email with the full Gemini output.
- **Configuration choices:**
  - Recipient is `GMAIL_TO` from the Set node
  - Subject: `🔮 Palm Reading Detailed Report`
  - Sender name: `Palm Reading Bot`
  - Email body includes:
    - heading
    - thank-you note
    - full Gemini text inside a `<pre>` block
    - footer note
- **Key expressions or variables used:**
  - `{{ $('Set config variables').item.json.GMAIL_TO }}`
  - `{{ $('Parse Gemini response').item.json.geminiText }}`
- **Input and output connections:**
  - Input from **Parse Gemini response**
  - Output to **Respond to webhook**
- **Version-specific requirements:**
  - Type version `2.1`
  - Requires Gmail OAuth2 credential
- **Edge cases or potential failure types:**
  - Gmail OAuth token expiry or revoked consent
  - Gmail sending restrictions or quota
  - HTML rendering differences
  - `GMAIL_FROM` exists in config but is not mapped into this node
- **Sub-workflow reference:** None.

#### Register lucky days in Google Calendar
- **Type and technical role:** `n8n-nodes-base.googleCalendar`; creates a calendar event.
- **Configuration choices:**
  - Calendar selected by ID from `GOOGLE_CALENDAR_ID`
  - Start time: current time plus 7 days
  - End time: current time plus 7 days plus 1 hour
  - Description is extracted from the `[Lucky day suggestions]` section of Gemini output
- **Key expressions or variables used:**
  - `{{ $now.plus(7, 'days').toISO() }}`
  - `{{ $now.plus(7, 'days').plus(1, 'hours').toISO() }}`
  - `{{ $('Set config variables').item.json.GOOGLE_CALENDAR_ID }}`
  - `{{ $('Parse Gemini response').item.json.geminiText.match(/\[Lucky day suggestions\]([\s\S]*?)(?=\[|$)/)?.[1]?.trim() ?? 'Lucky day from palm reading' }}`
- **Input and output connections:**
  - Input from **Parse Gemini response**
  - Output to **Respond to webhook**
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires Google Calendar OAuth2 credential
- **Edge cases or potential failure types:**
  - The node creates only one event, not three separate events, even though the prompt asks for three lucky days
  - Suggested dates are not parsed into actual calendar dates; the event is always scheduled 7 days from now
  - Calendar permission issues
  - Invalid calendar ID
  - Time zone behavior depends on n8n instance settings
- **Sub-workflow reference:** None.

---

## 2.5 Webhook Completion

### Overview
This block returns an HTTP response back to LINE after downstream processing reaches it. It acknowledges the webhook with a simple text payload.

### Nodes Involved
- Respond to webhook

### Node Details

#### Respond to webhook
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; final HTTP response node for workflows using webhook response mode.
- **Configuration choices:**
  - Respond with text
  - Response body: `OK`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Inputs from:
    - **Reply to LINE with summary**
    - **Send detailed report via Gmail**
    - **Register lucky days in Google Calendar**
  - No outputs
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Multiple upstream nodes converge here; depending on execution timing, only one execution path can practically complete the webhook response first
  - The non-image branch does not connect here, so unsupported-message requests may not receive a proper webhook response
  - If any upstream node fails before reaching this node, the webhook may remain unanswered
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive LINE webhook | Webhook | Receives LINE bot webhook POST requests |  | Set config variables | ## 1️⃣ Trigger & config\nReceives the LINE webhook POST and immediately sets all credentials and event data into named fields for easy reference downstream. |
| Set config variables | Set | Stores static config and extracts key fields from the LINE payload | Receive LINE webhook | Check if message is image | ## 1️⃣ Trigger & config\nReceives the LINE webhook POST and immediately sets all credentials and event data into named fields for easy reference downstream. |
| Check if message is image | If | Branches between supported image messages and unsupported message types | Set config variables | Download image from LINE; Reply with error for non-image message | ## 2️⃣ Image check\nOnly image messages are supported. Text or sticker messages are caught here and receive a friendly prompt asking the user to send a photo of their palm instead. |
| Reply with error for non-image message | HTTP Request | Sends a fallback LINE reply for unsupported message types | Check if message is image |  | ## 2️⃣ Image check\nOnly image messages are supported. Text or sticker messages are caught here and receive a friendly prompt asking the user to send a photo of their palm instead. |
| Download image from LINE | HTTP Request | Downloads the image binary from the LINE content API | Check if message is image | Analyze palm lines with Gemini | ## 3️⃣ AI analysis\nThe palm image is fetched from LINE's content API and passed to Google Gemini. The prompt asks for a LINE-ready short summary, a full line-by-line reading, and three lucky date suggestions in YYYY-MM-DD format. |
| Google Gemini Chat Model | Google Gemini Chat Model | Provides the Gemini multimodal model for the AI chain |  | Analyze palm lines with Gemini | ## 3️⃣ AI analysis\nThe palm image is fetched from LINE's content API and passed to Google Gemini. The prompt asks for a LINE-ready short summary, a full line-by-line reading, and three lucky date suggestions in YYYY-MM-DD format. |
| Analyze palm lines with Gemini | LangChain LLM Chain | Sends palm image plus prompt to Gemini and receives structured text output | Download image from LINE; Google Gemini Chat Model | Parse Gemini response | ## 3️⃣ AI analysis\nThe palm image is fetched from LINE's content API and passed to Google Gemini. The prompt asks for a LINE-ready short summary, a full line-by-line reading, and three lucky date suggestions in YYYY-MM-DD format. |
| Parse Gemini response | Set | Extracts reusable text fields from Gemini output | Analyze palm lines with Gemini | Reply to LINE with summary; Send detailed report via Gmail; Register lucky days in Google Calendar | ## 4️⃣ Multi-channel output\nAfter parsing the Gemini response, three actions run in parallel:\n- Short summary → LINE reply\n- Full report → Gmail HTML email\n- Lucky dates → Google Calendar events |
| Reply to LINE with summary | HTTP Request | Sends the short palm reading summary back to LINE | Parse Gemini response | Respond to webhook | ## 4️⃣ Multi-channel output\nAfter parsing the Gemini response, three actions run in parallel:\n- Short summary → LINE reply\n- Full report → Gmail HTML email\n- Lucky dates → Google Calendar events |
| Send detailed report via Gmail | Gmail | Emails the full report in HTML format | Parse Gemini response | Respond to webhook | ## 4️⃣ Multi-channel output\nAfter parsing the Gemini response, three actions run in parallel:\n- Short summary → LINE reply\n- Full report → Gmail HTML email\n- Lucky dates → Google Calendar events |
| Register lucky days in Google Calendar | Google Calendar | Creates a calendar event containing lucky day suggestions | Parse Gemini response | Respond to webhook | ## 4️⃣ Multi-channel output\nAfter parsing the Gemini response, three actions run in parallel:\n- Short summary → LINE reply\n- Full report → Gmail HTML email\n- Lucky dates → Google Calendar events |
| Respond to webhook | Respond to Webhook | Returns `OK` to the original webhook request | Reply to LINE with summary; Send detailed report via Gmail; Register lucky days in Google Calendar |  |  |
| Sticky Note — Overview | Sticky Note | Canvas documentation for purpose, setup, and customization |  |  | ## 🔮 Palm Reading Bot — Overview\n### How it works\nThis workflow turns your LINE account into an AI-powered palm reading bot. When a user sends a palm photo via LINE, the image is downloaded and analyzed by Google Gemini, which identifies the life line, heart line, head line, and fate line. The results are split into three outputs: a short reply sent back to LINE, a detailed HTML report delivered by Gmail, and up to three lucky days automatically registered in Google Calendar.\n### Setup steps\n1. Add your LINE Channel Access Token to the **Set config variables** node\n2. Set your Gmail address for both `GMAIL_FROM` and `GMAIL_TO`\n3. Set your Google Calendar ID (usually your Gmail address)\n4. Connect your Google Gemini API credential to the **Google Gemini Chat Model** node\n5. Connect your Gmail OAuth2 credential to the **Send detailed report via Gmail** node\n6. Connect your Google Calendar OAuth2 credential to the **Register lucky days in Google Calendar** node\n7. Set the LINE webhook URL in your LINE Developer Console to point to this workflow's webhook URL\n### Customization\n- Edit the Gemini prompt in **Analyze palm lines with Gemini** to change the reading style or language\n- Swap Gmail for any email node to match your mail provider\n- Adjust the calendar event timing (default: 7 days from now) in the Calendar node |
| Sticky Note — Trigger section | Sticky Note | Canvas note describing trigger/config block |  |  | ## 1️⃣ Trigger & config\nReceives the LINE webhook POST and immediately sets all credentials and event data into named fields for easy reference downstream. |
| Sticky Note — Image check section | Sticky Note | Canvas note describing image validation block |  |  | ## 2️⃣ Image check\nOnly image messages are supported. Text or sticker messages are caught here and receive a friendly prompt asking the user to send a photo of their palm instead. |
| Sticky Note — AI analysis section | Sticky Note | Canvas note describing AI analysis block |  |  | ## 3️⃣ AI analysis\nThe palm image is fetched from LINE's content API and passed to Google Gemini. The prompt asks for a LINE-ready short summary, a full line-by-line reading, and three lucky date suggestions in YYYY-MM-DD format. |
| Sticky Note — Output section | Sticky Note | Canvas note describing output block |  |  | ## 4️⃣ Multi-channel output\nAfter parsing the Gemini response, three actions run in parallel:\n- Short summary → LINE reply\n- Full report → Gmail HTML email\n- Lucky dates → Google Calendar events |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Webhook node** named **Receive LINE webhook**.
   - Method: `POST`
   - Path: `line-webhook`
   - Response mode: `Using Respond to Webhook node`
3. **Add a Set node** named **Set config variables** and connect it after the webhook.
   - Add these string fields:
     - `LINE_CHANNEL_ACCESS_TOKEN` = your LINE Messaging API channel access token
     - `GMAIL_FROM` = your Gmail address
     - `GMAIL_TO` = your Gmail address or recipient address
     - `GOOGLE_CALENDAR_ID` = your Google Calendar ID, often the Gmail address
   - Add these expression-based fields:
     - `replyToken` = `{{ $json.body.events[0].replyToken }}`
     - `messageType` = `{{ $json.body.events[0].message.type }}`
     - `imageId` = `{{ $json.body.events[0].message.id }}`
     - `userId` = `{{ $json.body.events[0].source.userId }}`
4. **Add an If node** named **Check if message is image**.
   - Condition: `messageType` equals `image`
   - Connect **Set config variables** to this node.
5. **Add an HTTP Request node** named **Reply with error for non-image message** for the false branch.
   - Method: `POST`
   - URL: `https://api.line.me/v2/bot/message/reply`
   - Send headers: enabled
   - Headers:
     - `Authorization` = `Bearer {{ $('Set config variables').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
     - `Content-Type` = `application/json`
   - Send body as JSON
   - Body:
     - `replyToken` from `{{ $('Set config variables').item.json.replyToken }}`
     - one text message saying:
       - `🙏 Please send a photo of your palm!\nThis bot only supports image messages.`
   - Connect the false output of the If node to this node.
6. **Add an HTTP Request node** named **Download image from LINE** for the true branch.
   - Method: `GET`
   - URL: `https://api-data.line.me/v2/bot/message/{{ $json.imageId }}/content`
   - Send headers: enabled
   - Header:
     - `Authorization` = `Bearer {{ $json.LINE_CHANNEL_ACCESS_TOKEN }}`
   - Response format: `File`
   - Connect the true output of the If node to this node.
7. **Add a Google Gemini Chat Model node** named **Google Gemini Chat Model**.
   - Select a valid Google Gemini API credential.
   - Temperature: `0.7`
   - Max output tokens: `1500`
8. **Add a LangChain LLM Chain node** named **Analyze palm lines with Gemini**.
   - Connect **Download image from LINE** to its main input.
   - Connect **Google Gemini Chat Model** to its AI language model input.
   - Set prompt mode to define text manually.
   - Use a prompt equivalent to:
     - Ask Gemini to act as a professional palm reader
     - Request exactly three sections:
       - `[LINE reply (within 100 characters)]`
       - `[Detailed report]`
       - `[Lucky day suggestions]`
     - Ask for:
       - life line reading and health fortune
       - heart line reading and love fortune
       - head line reading and career fortune
       - fate line reading and overall fortune
       - summary
       - 3 lucky dates in `YYYY-MM-DD` format with reasons
   - Add a human message of type **image binary** so the downloaded binary image is sent to the model.
9. **Add a Set node** named **Parse Gemini response** and connect it after the LLM chain.
   - Create `geminiText` = `{{ $json.text }}`
   - Create `lineMessage` with a formatted expression that:
     - prepends `🔮 Palm Reading Result`
     - extracts the text after the `[LINE reply ...]` heading using regex
     - falls back to: `Your palm has been read! Check your email for the full report ✨`
10. **Add an HTTP Request node** named **Reply to LINE with summary**.
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/reply`
    - Headers:
      - `Authorization` = `Bearer {{ $('Set config variables').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
      - `Content-Type` = `application/json`
    - Body as JSON:
      - `replyToken` = `{{ $('Set config variables').item.json.replyToken }}`
      - `messages[0].type` = `text`
      - `messages[0].text` = `{{ $json.lineMessage }}`
    - Connect **Parse Gemini response** to this node.
11. **Add a Gmail node** named **Send detailed report via Gmail**.
    - Operation: send email
    - Select a Gmail OAuth2 credential
    - To: `{{ $('Set config variables').item.json.GMAIL_TO }}`
    - Subject: `🔮 Palm Reading Detailed Report`
    - Sender name: `Palm Reading Bot`
    - Message format: HTML
    - HTML body should include:
      - a title
      - a thank-you line
      - the full Gemini output inside a `<pre>` block using `{{ $('Parse Gemini response').item.json.geminiText }}`
      - an automated footer note
    - Connect **Parse Gemini response** to this node.
12. **Add a Google Calendar node** named **Register lucky days in Google Calendar**.
    - Operation: create event
    - Select a Google Calendar OAuth2 credential
    - Calendar ID: `{{ $('Set config variables').item.json.GOOGLE_CALENDAR_ID }}`
    - Start: `{{ $now.plus(7, 'days').toISO() }}`
    - End: `{{ $now.plus(7, 'days').plus(1, 'hours').toISO() }}`
    - Description: use regex to extract the `[Lucky day suggestions]` section from `geminiText`, with fallback text
    - Connect **Parse Gemini response** to this node.
13. **Add a Respond to Webhook node** named **Respond to webhook**.
    - Respond with: `Text`
    - Response body: `OK`
14. **Connect the three output nodes** to **Respond to webhook**:
    - **Reply to LINE with summary** → **Respond to webhook**
    - **Send detailed report via Gmail** → **Respond to webhook**
    - **Register lucky days in Google Calendar** → **Respond to webhook**
15. **Recommended fix:** also connect **Reply with error for non-image message** to **Respond to webhook**.
    - This is not present in the JSON, but it is strongly recommended so all webhook executions return a valid HTTP response.
16. **Create and connect credentials**:
    - **Google Gemini API credential** for the Gemini model node
    - **Gmail OAuth2 credential** for the Gmail node
    - **Google Calendar OAuth2 credential** for the calendar node
17. **Activate the workflow** once tested.
18. **Copy the production webhook URL** from the Webhook node.
19. **Configure LINE Developer Console**:
    - Set the Messaging API webhook URL to the n8n webhook URL
    - Enable webhooks for the LINE bot
20. **Test end to end**:
    - Send a text message in LINE and verify the fallback response
    - Send a palm image and verify:
      - LINE summary reply
      - Gmail delivery
      - Google Calendar event creation

### Credential configuration details

- **LINE**
  - This workflow does not use an n8n credential object for LINE.
  - The channel access token is hardcoded in the **Set config variables** node.
  - Better practice: move it to an environment variable or credential-supported custom approach.

- **Google Gemini**
  - Use a valid Google Gemini / PaLM API credential compatible with the Gemini model node.
  - Ensure the API has vision-capable access if required by your n8n version/model configuration.

- **Gmail**
  - Use OAuth2 and authorize the target Gmail account.
  - Make sure the account is permitted to send email through the Gmail API.

- **Google Calendar**
  - Use OAuth2 and authorize calendar access.
  - Ensure the chosen account can create events in the calendar identified by `GOOGLE_CALENDAR_ID`.

### Input/output expectations

- **Expected webhook input**
  - LINE Messaging API webhook payload with `body.events[0]`
  - Must include a message event
  - For image processing, `body.events[0].message.type` must be `image`

- **Expected AI output**
  - Gemini should return clearly labeled sections:
    - `[LINE reply ...]`
    - `[Detailed report]`
    - `[Lucky day suggestions]`

- **Current calendar behavior**
  - Despite asking Gemini for three lucky dates, the workflow creates one calendar event scheduled 7 days ahead and stores the lucky day suggestions in the description only.
  - If you want one event per suggested date, you must add parsing logic plus item splitting before the Google Calendar node.

- **No sub-workflows**
  - This workflow does not call or expose any sub-workflow nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow turns your LINE account into an AI-powered palm reading bot. When a user sends a palm photo via LINE, the image is downloaded and analyzed by Google Gemini, which identifies the life line, heart line, head line, and fate line. The results are split into three outputs: a short reply sent back to LINE, a detailed HTML report delivered by Gmail, and up to three lucky days automatically registered in Google Calendar. | Overview note on the canvas |
| Add your LINE Channel Access Token to the **Set config variables** node. | Setup note |
| Set your Gmail address for both `GMAIL_FROM` and `GMAIL_TO`. | Setup note |
| Set your Google Calendar ID, usually your Gmail address. | Setup note |
| Connect your Google Gemini API credential to the **Google Gemini Chat Model** node. | Setup note |
| Connect your Gmail OAuth2 credential to the **Send detailed report via Gmail** node. | Setup note |
| Connect your Google Calendar OAuth2 credential to the **Register lucky days in Google Calendar** node. | Setup note |
| Set the LINE webhook URL in your LINE Developer Console to point to this workflow's webhook URL. | Deployment note |
| Edit the Gemini prompt in **Analyze palm lines with Gemini** to change the reading style or language. | Customization note |
| Swap Gmail for any email node to match your mail provider. | Customization note |
| Adjust the calendar event timing, default 7 days from now, in the Calendar node. | Customization note |

## Important implementation notes
- `GMAIL_FROM` is configured but not used by the Gmail node.
- The non-image branch does not currently return through **Respond to webhook**.
- The calendar node does not create events on Gemini’s suggested dates; it creates a single event 7 days in the future and places the lucky-day text in the description.
- The workflow assumes the first LINE event only: `events[0]`. If LINE sends batched events, all others are ignored.
- The regex-based parsing depends on Gemini preserving the requested section headers exactly enough to match.