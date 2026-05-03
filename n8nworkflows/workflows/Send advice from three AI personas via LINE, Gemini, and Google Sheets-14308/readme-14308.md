Send advice from three AI personas via LINE, Gemini, and Google Sheets

https://n8nworkflows.xyz/workflows/send-advice-from-three-ai-personas-via-line--gemini--and-google-sheets-14308


# Send advice from three AI personas via LINE, Gemini, and Google Sheets

# 1. Workflow Overview

This workflow turns a LINE Messaging API bot into a multi-persona advice assistant powered by Google Gemini and backed by Google Sheets logging.

When a user sends a text message to the LINE bot, the workflow:

- receives the webhook event from LINE
- extracts the relevant message metadata
- ignores non-text or non-message events
- sends the user’s message to Gemini through a Basic LLM Chain
- requests one structured response containing advice from three personas
- parses the AI response into separate persona fields
- stores the interaction in Google Sheets
- replies to the user in LINE with a Flex Message carousel containing three cards:
  - 🔮 Fortune Teller
  - 💼 Business Coach
  - 😊 Best Friend

## 1.1 Input Reception and Basic Configuration

This block starts with a webhook endpoint that receives POST requests from LINE and injects local configuration values such as the LINE channel token and Google Sheets identifiers.

## 1.2 Event Parsing and Routing

This block normalizes the LINE payload, extracts the first event, identifies whether it is a text message, and routes execution either to a skip response or into the AI path.

## 1.3 AI Prompting and Structured Advice Generation

This block assembles the working context, connects a Google Gemini chat model to a Basic LLM Chain, and prompts the model to return advice in strict JSON format with three persona outputs.

## 1.4 Response Parsing, Logging, and LINE Reply

This block parses Gemini’s output, applies fallbacks if the JSON is malformed or incomplete, appends the conversation to Google Sheets, sends a Flex Message carousel back to LINE, and finally returns an HTTP 200-style webhook acknowledgment.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Basic Configuration

### Overview

This block receives incoming webhook calls from LINE and defines workflow-level configuration values required later in the execution. These values are stored directly in a Set node instead of being pulled from credentials or environment variables.

### Nodes Involved

- Receive LINE message
- Set config

### Node Details

#### Receive LINE message

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point webhook that accepts POST requests from LINE.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `line-three-advisors`
  - Response mode: `responseNode`
- **Key expressions or variables used:**
  - No expressions inside this node
  - Webhook ID is defined as `line-three-advisors-webhook`
- **Input and output connections:**
  - Input: none, this is an entry node
  - Output: `Set config`
- **Version-specific requirements:**
  - Uses webhook node type version `2`
  - Because response mode is `responseNode`, the workflow must explicitly terminate with `Respond to Webhook` nodes
- **Edge cases or potential failure types:**
  - LINE webhook URL misconfigured
  - Wrong HTTP method
  - Workflow inactive in production
  - No final response path causing webhook timeout
- **Sub-workflow reference:** none

#### Set config

- **Type and technical role:** `n8n-nodes-base.set`  
  Adds static configuration values to the execution data.
- **Configuration choices:**
  - Defines:
    - `LINE_CHANNEL_ACCESS_TOKEN`
    - `SHEET_ID`
    - `HISTORY_SHEET_NAME`
  - Default sheet name is `advice_history`
- **Key expressions or variables used:**
  - No dynamic expressions; plain assigned strings
- **Input and output connections:**
  - Input: `Receive LINE message`
  - Output: `Parse LINE event`
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases or potential failure types:**
  - Placeholder values left unchanged
  - Invalid LINE token causing downstream 401 errors
  - Wrong sheet ID causing Google Sheets lookup failure
- **Sub-workflow reference:** none

---

## 2.2 Event Parsing and Routing

### Overview

This block extracts the first LINE event from either the root payload or `body.events`, validates that it is a text message, and either exits early or prepares a clean context object for AI processing.

### Nodes Involved

- Parse LINE event
- Skip if no text event
- Respond OK (skip)
- Build context

### Node Details

#### Parse LINE event

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript normalization and extraction step for LINE webhook data.
- **Configuration choices:**
  - Checks for events in:
    - `input.events`
    - `input.body.events`
  - Extracts:
    - `userId`
    - `replyToken`
    - `messageText`
    - `eventType`
  - Returns `skip: true` when:
    - no events exist
    - event type is not `message`
    - message type is not `text`
- **Key expressions or variables used:**
  - `$input.first().json`
  - Optional chaining on event fields such as `event.source?.userId`
- **Input and output connections:**
  - Input: `Set config`
  - Output: `Skip if no text event`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Multiple LINE events are present but only the first is processed
  - Unexpected payload shape
  - Missing `replyToken` or `userId`
  - Non-text messages are intentionally skipped
- **Sub-workflow reference:** none

#### Skip if no text event

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the event should be skipped.
- **Configuration choices:**
  - Checks boolean condition:
    - `{{ $json.skip }}` is `true`
- **Key expressions or variables used:**
  - `={{ $json.skip }}`
- **Input and output connections:**
  - Input: `Parse LINE event`
  - True output: `Respond OK (skip)`
  - False output: `Build context`
- **Version-specific requirements:**
  - Uses If node version `2`
- **Edge cases or potential failure types:**
  - If the upstream node returns malformed `skip`, branch behavior may become incorrect
- **Sub-workflow reference:** none

#### Respond OK (skip)

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an immediate plain-text `OK` response for events the workflow intentionally ignores.
- **Configuration choices:**
  - Respond with text
  - Body: `OK`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input: `Skip if no text event` true branch
  - Output: none
- **Version-specific requirements:**
  - Uses Respond to Webhook version `1`
- **Edge cases or potential failure types:**
  - If omitted, LINE may retry skipped events due to timeout
- **Sub-workflow reference:** none

#### Build context

- **Type and technical role:** `n8n-nodes-base.code`  
  Consolidates parsed LINE event data with configuration fields into one object for downstream nodes.
- **Configuration choices:**
  - Pulls data from:
    - `Parse LINE event`
    - `Set config`
  - Outputs:
    - `userId`
    - `replyToken`
    - `messageText`
    - `SHEET_ID`
    - `HISTORY_SHEET_NAME`
    - `LINE_CHANNEL_ACCESS_TOKEN`
- **Key expressions or variables used:**
  - `$('Parse LINE event').first().json`
  - `$('Set config').first().json`
- **Input and output connections:**
  - Input: `Skip if no text event` false branch
  - Output: `Generate advice with Gemini`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Cross-node lookup fails if upstream data is unavailable
  - Missing config values propagate silently into later failures
- **Sub-workflow reference:** none

---

## 2.3 AI Prompting and Structured Advice Generation

### Overview

This block sends the user message to Gemini using a Basic LLM Chain and instructs the model to respond with JSON containing three persona-based advice texts.

### Nodes Involved

- Generate advice with Gemini
- Google Gemini Chat Model

### Node Details

#### Generate advice with Gemini

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Basic LLM Chain node that formats and submits the prompt to the connected language model.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Includes the user message via expression
  - Strictly requests JSON output with keys:
    - `fortune_teller`
    - `business_coach`
    - `best_friend`
  - Persona tone instructions:
    - Fortune Teller: mystical, warm, stars/fate/intuition, 3–4 sentences
    - Business Coach: logical, action-oriented, specific next steps, 3–4 sentences
    - Best Friend: casual, empathetic, “I totally get it!”, touch of humor, 3–4 sentences
- **Key expressions or variables used:**
  - `{{ $json.messageText }}`
- **Input and output connections:**
  - Main input: `Build context`
  - AI language model input: `Google Gemini Chat Model`
  - Main output: `Parse Gemini response`
- **Version-specific requirements:**
  - Uses node version `1.9`
  - Requires LangChain-compatible AI model connection
- **Edge cases or potential failure types:**
  - Gemini may return additional prose despite the JSON-only instruction
  - Rate limits or quota exhaustion
  - Credential/authentication failures
  - Prompt injection from user message could distort output format
- **Sub-workflow reference:** none

#### Google Gemini Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Supplies the actual Gemini model backend used by the Basic LLM Chain.
- **Configuration choices:**
  - Minimal explicit options; relies primarily on attached credential and default model settings
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via `ai_languageModel` connection into `Generate advice with Gemini`
- **Version-specific requirements:**
  - Uses node version `1`
  - Requires valid Google Gemini credential in n8n
- **Edge cases or potential failure types:**
  - Missing or invalid Gemini credential
  - Model availability issues
  - Region/quota limitations
- **Sub-workflow reference:** none

---

## 2.4 Response Parsing, Logging, and LINE Reply

### Overview

This block extracts the structured advice from Gemini output, fills in defaults when parsing fails, stores the interaction in Google Sheets, sends a LINE Flex carousel response, and completes the webhook cycle.

### Nodes Involved

- Parse Gemini response
- Save advice to Sheets
- Send Flex Message to LINE
- Respond OK

### Node Details

#### Parse Gemini response

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses AI output text to recover a JSON object and merges it with the saved context.
- **Configuration choices:**
  - Reads `text` from the Gemini chain result
  - Uses regex `/{[\s\S]*}/` to extract the first JSON-looking block
  - Attempts `JSON.parse`
  - On failure, uses default fallback advice strings for all personas
  - Re-attaches:
    - user metadata
    - LINE token
    - sheet identifiers
- **Key expressions or variables used:**
  - `$input.first().json.text`
  - `$('Build context').first().json`
- **Input and output connections:**
  - Input: `Generate advice with Gemini`
  - Output: `Save advice to Sheets`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Regex may match malformed JSON-like content
  - AI may output markdown fences or escaped JSON
  - Missing keys are replaced with fallback text
  - If chain output field changes from `text`, parsing fails
- **Sub-workflow reference:** none

#### Save advice to Sheets

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to a Google Sheets worksheet for conversation history.
- **Configuration choices:**
  - Operation: `append`
  - Document ID from expression: `{{ $json.SHEET_ID }}`
  - Sheet name from expression: `{{ $json.HISTORY_SHEET_NAME }}`
  - Defined column mapping:
    - `Timestamp` → `{{ $now.toISO() }}`
    - `User ID` → `{{ $json.userId }}`
    - `Message` → `{{ $json.messageText }}`
    - `Fortune Teller` → `{{ $json.fortune_teller }}`
    - `Business Coach` → `{{ $json.business_coach }}`
    - `Best Friend` → `{{ $json.best_friend }}`
- **Key expressions or variables used:**
  - `={{ $now.toISO() }}`
  - `={{ $json.userId }}`
  - `={{ $json.messageText }}`
  - persona field expressions
- **Input and output connections:**
  - Input: `Parse Gemini response`
  - Output: `Send Flex Message to LINE`
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.5`
  - Requires matching sheet headers to avoid mapping confusion
- **Edge cases or potential failure types:**
  - Invalid spreadsheet ID
  - Sheet tab missing or misnamed
  - OAuth permission errors
  - Header mismatch may break append mapping
- **Sub-workflow reference:** none

#### Send Flex Message to LINE

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a reply message to the LINE Messaging API using a manually crafted Flex Message payload.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.line.me/v2/bot/message/reply`
  - Sends headers:
    - `Authorization: Bearer {{ $json.LINE_CHANNEL_ACCESS_TOKEN }}`
    - `Content-Type: application/json`
  - JSON body contains:
    - `replyToken`
    - one `flex` message
    - a `carousel` with three `bubble` cards
  - Card colors:
    - Fortune Teller: `#6a0dad`
    - Business Coach: `#1a73e8`
    - Best Friend: `#34a853`
- **Key expressions or variables used:**
  - `{{ $json.replyToken }}`
  - `{{ $json.fortune_teller }}`
  - `{{ $json.business_coach }}`
  - `{{ $json.best_friend }}`
  - `{{ $json.LINE_CHANNEL_ACCESS_TOKEN }}`
- **Input and output connections:**
  - Input: `Save advice to Sheets`
  - Output: `Respond OK`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Expired or invalid LINE access token
  - Reply token expiration if processing takes too long
  - Flex payload validation errors from LINE
  - Long AI text may exceed LINE message size/layout constraints
- **Sub-workflow reference:** none

#### Respond OK

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Completes the webhook request after the LINE API reply call has been attempted.
- **Configuration choices:**
  - Respond with text
  - Body: `OK`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input: `Send Flex Message to LINE`
  - Output: none
- **Version-specific requirements:**
  - Uses Respond to Webhook version `1`
- **Edge cases or potential failure types:**
  - If previous nodes fail, webhook may not return successfully unless error handling is added
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive LINE message | Webhook | Receives POST webhook events from LINE |  | Set config | ## Overview<br>This workflow lets users send any worry or question to a LINE bot and instantly receive advice from three distinct AI personas — a Fortune Teller 🔮, a Business Coach 💼, and a Best Friend 😊 — all powered by Google Gemini.<br><br>Each conversation is automatically logged to Google Sheets for review and analysis.<br><br>### How it works<br>1. A user sends a text message to your LINE bot<br>2. The message is parsed and passed to Gemini via Basic LLM Chain<br>3. Gemini responds as three different personas in a single JSON reply<br>4. The advice is saved to Google Sheets<br>5. A LINE Flex Message carousel is sent back with each persona in a color-coded card<br><br>### Setup steps<br>1. Create a LINE Messaging API channel at https://developers.line.biz and copy your Channel Access Token<br>2. Get a free Gemini API key at https://aistudio.google.com<br>3. Create a Google Sheets spreadsheet with a sheet named `advice_history` — add these headers in row 1: `Timestamp`, `User ID`, `Message`, `Fortune Teller`, `Business Coach`, `Best Friend`<br>4. In the **Set config** node, paste your LINE token, Gemini API key, and Sheet ID<br>5. Set your LINE webhook URL to the n8n webhook path<br>6. Activate the workflow and test by messaging your LINE bot<br>## 1. Receive & configure<br>Webhook receives the LINE event.<br>Paste your credentials in **Set config**. |
| Set config | Set | Stores static LINE token and Sheets configuration | Receive LINE message | Parse LINE event | ## Overview<br>This workflow lets users send any worry or question to a LINE bot and instantly receive advice from three distinct AI personas — a Fortune Teller 🔮, a Business Coach 💼, and a Best Friend 😊 — all powered by Google Gemini.<br><br>Each conversation is automatically logged to Google Sheets for review and analysis.<br><br>### How it works<br>1. A user sends a text message to your LINE bot<br>2. The message is parsed and passed to Gemini via Basic LLM Chain<br>3. Gemini responds as three different personas in a single JSON reply<br>4. The advice is saved to Google Sheets<br>5. A LINE Flex Message carousel is sent back with each persona in a color-coded card<br><br>### Setup steps<br>1. Create a LINE Messaging API channel at https://developers.line.biz and copy your Channel Access Token<br>2. Get a free Gemini API key at https://aistudio.google.com<br>3. Create a Google Sheets spreadsheet with a sheet named `advice_history` — add these headers in row 1: `Timestamp`, `User ID`, `Message`, `Fortune Teller`, `Business Coach`, `Best Friend`<br>4. In the **Set config** node, paste your LINE token, Gemini API key, and Sheet ID<br>5. Set your LINE webhook URL to the n8n webhook path<br>6. Activate the workflow and test by messaging your LINE bot<br>## 1. Receive & configure<br>Webhook receives the LINE event.<br>Paste your credentials in **Set config**. |
| Parse LINE event | Code | Extracts first LINE event and normalizes text-message fields | Set config | Skip if no text event | ## 2. Parse & route<br>Extracts userId, replyToken, and message text.<br>Non-message events (follows, etc.) are skipped immediately. |
| Skip if no text event | If | Routes skipped events away from AI processing | Parse LINE event | Respond OK (skip); Build context | ## 2. Parse & route<br>Extracts userId, replyToken, and message text.<br>Non-message events (follows, etc.) are skipped immediately. |
| Respond OK (skip) | Respond to Webhook | Returns immediate OK for ignored events | Skip if no text event |  | ## 2. Parse & route<br>Extracts userId, replyToken, and message text.<br>Non-message events (follows, etc.) are skipped immediately. |
| Build context | Code | Combines parsed event fields with config values | Skip if no text event | Generate advice with Gemini | ## 2. Parse & route<br>Extracts userId, replyToken, and message text.<br>Non-message events (follows, etc.) are skipped immediately. |
| Generate advice with Gemini | Basic LLM Chain | Sends prompt requesting three persona responses in JSON | Build context; Google Gemini Chat Model | Parse Gemini response | ## 3. Generate advice with Gemini<br>Builds a structured prompt and calls Gemini.<br>The response is parsed into three persona fields. |
| Google Gemini Chat Model | Google Gemini Chat Model | Provides the Gemini language model to the LLM chain |  | Generate advice with Gemini | ## 3. Generate advice with Gemini<br>Builds a structured prompt and calls Gemini.<br>The response is parsed into three persona fields. |
| Parse Gemini response | Code | Extracts JSON from model output and applies fallback persona text | Generate advice with Gemini | Save advice to Sheets | ## 4. Log & reply<br>Saves the conversation to Google Sheets,<br>then sends a Flex Message carousel back to the user. |
| Save advice to Sheets | Google Sheets | Appends advice history row to spreadsheet | Parse Gemini response | Send Flex Message to LINE | ## 4. Log & reply<br>Saves the conversation to Google Sheets,<br>then sends a Flex Message carousel back to the user. |
| Send Flex Message to LINE | HTTP Request | Calls LINE reply API with Flex Message carousel payload | Save advice to Sheets | Respond OK | ## 4. Log & reply<br>Saves the conversation to Google Sheets,<br>then sends a Flex Message carousel back to the user. |
| Respond OK | Respond to Webhook | Completes webhook execution with plain-text OK | Send Flex Message to LINE |  | ## 4. Log & reply<br>Saves the conversation to Google Sheets,<br>then sends a Flex Message carousel back to the user. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   `Send advice from three AI personas via LINE, Gemini, and Google Sheets`.

2. **Add a Webhook node** named `Receive LINE message`.
   - Type: **Webhook**
   - HTTP Method: `POST`
   - Path: `line-three-advisors`
   - Response Mode: `Using Respond to Webhook Node`
   - Save the node.
   - This node becomes the URL you will later paste into the LINE Messaging API webhook settings.

3. **Add a Set node** named `Set config` and connect:
   - `Receive LINE message` → `Set config`
   - In the Set node, create these string fields:
     - `LINE_CHANNEL_ACCESS_TOKEN` = your LINE Messaging API channel access token
     - `SHEET_ID` = your Google Sheets spreadsheet ID
     - `HISTORY_SHEET_NAME` = `advice_history`

4. **Add a Code node** named `Parse LINE event` and connect:
   - `Set config` → `Parse LINE event`
   - Paste this logic conceptually:
     - read the incoming JSON
     - detect whether events are in `events` or `body.events`
     - if there are no events, return `skip: true`
     - extract the first event only
     - extract `userId`, `replyToken`, `messageText`, and `eventType`
     - if the event is not a text message, return `skip: true`
     - otherwise return `skip: false` with the extracted fields
   - Output fields should include:
     - `skip`
     - `userId`
     - `replyToken`
     - `messageText`
     - `eventType`

5. **Add an If node** named `Skip if no text event` and connect:
   - `Parse LINE event` → `Skip if no text event`
   - Configure the condition:
     - left value: `{{ $json.skip }}`
     - operation: boolean `true`
   - This means:
     - true branch = skip processing
     - false branch = continue to Gemini

6. **Add a Respond to Webhook node** named `Respond OK (skip)` and connect it to the true branch.
   - `Skip if no text event` true → `Respond OK (skip)`
   - Configure:
     - Respond With: `Text`
     - Response Body: `OK`

7. **Add another Code node** named `Build context` and connect it to the false branch.
   - `Skip if no text event` false → `Build context`
   - Configure it to merge:
     - parsed event data from `Parse LINE event`
     - static settings from `Set config`
   - Output fields:
     - `userId`
     - `replyToken`
     - `messageText`
     - `SHEET_ID`
     - `HISTORY_SHEET_NAME`
     - `LINE_CHANNEL_ACCESS_TOKEN`

8. **Add a Google Gemini Chat Model node** named `Google Gemini Chat Model`.
   - Type: **Google Gemini Chat Model**
   - Attach a valid Gemini credential.
   - If your n8n instance asks for model selection or options, use the default compatible chat model unless you have a specific preference.

9. **Add a Basic LLM Chain node** named `Generate advice with Gemini`.
   - Connect:
     - `Build context` → `Generate advice with Gemini`
     - `Google Gemini Chat Model` → AI model input on `Generate advice with Gemini`
   - Set prompt mode to define the prompt directly.
   - Use a prompt that:
     - includes the user message
     - asks for exactly one JSON object
     - requires the keys:
       - `fortune_teller`
       - `business_coach`
       - `best_friend`
   - Include persona instructions similar to:
     - Fortune Teller: mystical, fate and intuition, 3–4 sentences
     - Business Coach: practical steps, direct and motivating, 3–4 sentences
     - Best Friend: casual and empathetic, light humor, 3–4 sentences
   - Include the user message through an expression like:
     - `{{ $json.messageText }}`

10. **Add a Code node** named `Parse Gemini response`.
    - Connect:
      - `Generate advice with Gemini` → `Parse Gemini response`
    - Configure it to:
      - read the LLM output text
      - extract a JSON object using a regex or equivalent parsing strategy
      - parse the JSON
      - if parsing fails, fill default fallback strings for each persona
      - merge the result back with the context from `Build context`
    - Output fields should include:
      - `userId`
      - `replyToken`
      - `messageText`
      - `fortune_teller`
      - `business_coach`
      - `best_friend`
      - `SHEET_ID`
      - `HISTORY_SHEET_NAME`
      - `LINE_CHANNEL_ACCESS_TOKEN`

11. **Prepare your Google Sheet** before adding the Sheets node.
    - Create a spreadsheet in Google Sheets.
    - Copy its spreadsheet ID from the URL.
    - Create a worksheet/tab named:
      - `advice_history`
    - Add this header row exactly in row 1:
      - `Timestamp`
      - `User ID`
      - `Message`
      - `Fortune Teller`
      - `Business Coach`
      - `Best Friend`

12. **Add a Google Sheets node** named `Save advice to Sheets`.
    - Connect:
      - `Parse Gemini response` → `Save advice to Sheets`
    - Operation: `Append`
    - Credential: connect a Google Sheets OAuth2 credential
    - Spreadsheet/document ID: use expression `{{ $json.SHEET_ID }}`
    - Sheet name: use expression `{{ $json.HISTORY_SHEET_NAME }}`
    - Map the columns:
      - `Timestamp` → `{{ $now.toISO() }}`
      - `User ID` → `{{ $json.userId }}`
      - `Message` → `{{ $json.messageText }}`
      - `Fortune Teller` → `{{ $json.fortune_teller }}`
      - `Business Coach` → `{{ $json.business_coach }}`
      - `Best Friend` → `{{ $json.best_friend }}`

13. **Add an HTTP Request node** named `Send Flex Message to LINE`.
    - Connect:
      - `Save advice to Sheets` → `Send Flex Message to LINE`
    - Configure:
      - Method: `POST`
      - URL: `https://api.line.me/v2/bot/message/reply`
      - Send Headers: enabled
      - Send Body: enabled
      - Body Type: JSON
    - Headers:
      - `Authorization` = `Bearer {{ $json.LINE_CHANNEL_ACCESS_TOKEN }}`
      - `Content-Type` = `application/json`

14. **Configure the LINE reply JSON body** in `Send Flex Message to LINE`.
    - Include:
      - `replyToken` from `{{ $json.replyToken }}`
      - a `messages` array with one Flex Message
      - Flex `altText` such as `Advice from your three advisors!`
      - `contents.type = carousel`
      - three bubbles, one per persona
    - Bubble 1:
      - title `🔮 Fortune Teller`
      - body text = `{{ $json.fortune_teller }}`
      - header color `#6a0dad`
    - Bubble 2:
      - title `💼 Business Coach`
      - body text = `{{ $json.business_coach }}`
      - header color `#1a73e8`
    - Bubble 3:
      - title `😊 Best Friend`
      - body text = `{{ $json.best_friend }}`
      - header color `#34a853`
    - Set text components to wrap so long advice can span multiple lines.

15. **Add a Respond to Webhook node** named `Respond OK`.
    - Connect:
      - `Send Flex Message to LINE` → `Respond OK`
    - Configure:
      - Respond With: `Text`
      - Response Body: `OK`

16. **Configure credentials**.
    - **Google Gemini credential**
      - Create or attach a valid Gemini API credential in n8n.
      - Assign it to `Google Gemini Chat Model`.
    - **Google Sheets OAuth2 credential**
      - Create or attach a Google Sheets OAuth2 credential in n8n.
      - Assign it to `Save advice to Sheets`.
    - **LINE credential**
      - This workflow does not use a formal n8n credential node for LINE.
      - Instead, paste the LINE Channel Access Token into `Set config`.

17. **Configure LINE Messaging API webhook settings**.
    - In LINE Developers Console:
      - create or open your Messaging API channel
      - copy the webhook production URL from `Receive LINE message`
      - paste it into LINE’s webhook URL setting
      - enable webhook delivery

18. **Test with a real text message**.
    - Send a text message to the LINE bot.
    - Confirm:
      - the webhook executes
      - Gemini returns text
      - the parser extracts three persona fields
      - a new row appears in `advice_history`
      - LINE receives the Flex carousel reply

19. **Activate the workflow** once all tests pass.

## Rebuild Connection Order Summary

1. `Receive LINE message` → `Set config`  
2. `Set config` → `Parse LINE event`  
3. `Parse LINE event` → `Skip if no text event`  
4. `Skip if no text event` true → `Respond OK (skip)`  
5. `Skip if no text event` false → `Build context`  
6. `Build context` → `Generate advice with Gemini`  
7. `Google Gemini Chat Model` → AI model input of `Generate advice with Gemini`  
8. `Generate advice with Gemini` → `Parse Gemini response`  
9. `Parse Gemini response` → `Save advice to Sheets`  
10. `Save advice to Sheets` → `Send Flex Message to LINE`  
11. `Send Flex Message to LINE` → `Respond OK`

## Rebuild Constraints and Important Behavior

- Only the **first LINE event** in the webhook payload is processed.
- Only **text messages** proceed to Gemini.
- Non-text events return `OK` immediately.
- Gemini is expected to return strict JSON, but parsing fallback text is already built in.
- The Google Sheet headers should match the configured column names exactly.
- LINE reply tokens can expire quickly, so heavy AI latency may cause reply failures.
- Because the workflow uses `Respond to Webhook`, every execution path must end in a response node.

## Sub-workflow Setup

- This workflow does **not** invoke any sub-workflows.
- There are **no multiple n8n entry points** beyond the single webhook node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a LINE Messaging API channel and copy your Channel Access Token | https://developers.line.biz |
| Get a free Gemini API key | https://aistudio.google.com |
| Google Sheet tab name expected by the workflow | `advice_history` |
| Required Google Sheets header row | `Timestamp`, `User ID`, `Message`, `Fortune Teller`, `Business Coach`, `Best Friend` |
| Workflow purpose | Users send a worry or question to LINE and receive advice from three AI personas powered by Gemini |
| Persona set used in this workflow | Fortune Teller 🔮, Business Coach 💼, Best Friend 😊 |
| Customization note | You can replace the three personas in the Gemini prompt with others such as therapist, investor, or comedian |
| UI customization note | You can change Flex Message card colors in `Send Flex Message to LINE` |
| Tracking customization note | You can add extra columns to Google Sheets to log more metadata |