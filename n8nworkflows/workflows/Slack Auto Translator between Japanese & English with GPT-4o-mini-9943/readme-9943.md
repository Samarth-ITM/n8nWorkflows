Slack Auto Translator between Japanese & English with GPT-4o-mini

https://n8nworkflows.xyz/workflows/slack-auto-translator-between-japanese---english-with-gpt-4o-mini-9943


# Slack Auto Translator between Japanese & English with GPT-4o-mini

### 1. Workflow Overview

This workflow automates bidirectional translation between Japanese and English within Slack, triggered by a slash command `/trans`. It is designed to enable seamless communication for users who switch between these two languages, using GPT-4o-mini for precise AI-powered translation.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Receives slash command requests from Slack and sends an immediate acknowledgement to prevent timeout.
- **1.2 Language Detection & Preprocessing**: Parses the user input, detects the source language heuristically, and determines the target language, supporting manual override.
- **1.3 AI Translation**: Sends the prepared text to OpenAI’s GPT-4o-mini model with instructions to translate precisely without commentary.
- **1.4 Response Preparation & Posting**: Formats the translated text into a Slack-friendly message and posts it back to the originating Slack channel using Slack’s response URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Handles incoming HTTP POST requests from Slack’s `/trans` slash command, immediately acknowledges receipt to avoid Slack’s 3-second timeout, and passes the payload downstream.

- **Nodes Involved:**  
  - Webhook (Slash Command)  
  - Code (Ack)  
  - Respond to Webhook  
  - Sticky Note4 (comment)

- **Node Details:**  
  - **Webhook (Slash Command)**  
    - Type: Webhook  
    - Role: Listen for POST requests at `/slack/trans` path from Slack slash command `/trans`.  
    - Config: `POST` method, response mode set to "responseNode" to defer response sending.  
    - Input: Slack slash command payload (JSON)  
    - Output: Raw request payload forwarded to next node.  
    - Edge Cases: Invalid HTTP method, malformed payload, missing required Slack fields may cause failures.  
  - **Code (Ack)**  
    - Type: Code node (JavaScript)  
    - Role: Sends immediate acknowledgement text "Translating..." back to Slack to prevent timeout.  
    - Config: Returns JSON `{ ack: "Translating..." }` which Respond to Webhook uses as response body.  
    - Input: Triggered after Webhook node.  
    - Output: Acknowledgement JSON.  
    - Edge Cases: None significant; simple static response.  
  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends the acknowledgement HTTP 200 response with body "Translating..." and content-type set to `text/plain; charset=utf-8`.  
    - Connections: Input from Code (Ack), no output.  
    - Edge Cases: Network issues or Slack not receiving acknowledgment could cause retries or errors.  
  - **Sticky Note4**  
    - Content: "Receives /trans requests from Slack. Uses Production URL in Slack."  
    - Context: Highlights this node’s role as Slack integration entry point.

#### 2.2 Language Detection & Preprocessing

- **Overview:**  
  Parses the incoming Slack command text, trims whitespace, handles empty input with error message, detects whether input is Japanese or English using Unicode script detection, and supports user override to explicitly specify target language with prefixes (`en:` or `ja:`).

- **Nodes Involved:**  
  - Detect Language (Code)  
  - Sticky Note1 (comment)

- **Node Details:**  
  - **Detect Language (Code)**  
    - Type: Code node (JavaScript)  
    - Role:  
      - Extracts the text from Slack payload.  
      - Returns early with error if input is empty.  
      - Parses optional language override prefix (`en:` or `ja:`).  
      - Uses Unicode regex to detect Japanese scripts (Hiragana, Katakana, Kanji).  
      - Sets target language as English if input is Japanese, or Japanese if input is English, unless overridden.  
      - Outputs: `{ text, target, response_url, user_id, channel_id, error? }`  
    - Key Expressions:  
      - Regex match for override: `/^(\w{2}):\s*(.*)$/`  
      - Japanese detection regex: `/[\p{Script=Hiragana}\p{Script=Katakana}\p{Script=Han}]/u`  
    - Input: Raw Slack payload from webhook  
    - Output: Parsed and enriched object for translation  
    - Edge Cases:  
      - Empty input triggers error message.  
      - Unrecognized override codes ignored.  
      - Unicode regex requires Node.js 12+ for Unicode property escapes.  
  - **Sticky Note1**  
    - Content: "Detects JA/EN automatically. Supports overrides like \nen: こんにちは \nand handles empty input."  
    - Context: Clarifies detection logic and override support.

#### 2.3 AI Translation

- **Overview:**  
  Sends the detected text and target language to OpenAI GPT-4o-mini model with a carefully crafted system prompt instructing it to translate precisely without explanation, preserving punctuation and line breaks.

- **Nodes Involved:**  
  - OpenAI (Chat) - Translate  
  - Sticky Note2 (comment)

- **Node Details:**  
  - **OpenAI (Chat) - Translate**  
    - Type: OpenAI Chat Completion (Langchain integration)  
    - Role: Request GPT-4o-mini to translate text from source to target language.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Temperature: 0.2 (low randomness for consistent translations)  
      - Messages:  
        - System message with instructions including target language interpolation and input text.  
        - User message repeats input text (likely redundant but included).  
    - Credentials: OpenAI API credentials set up in n8n.  
    - Input: JSON with `text` and `target` from Detect Language node.  
    - Output: GPT translation response JSON.  
    - Edge Cases:  
      - API quota exceeded or authentication error  
      - Network timeout  
      - Unexpected response format  
  - **Sticky Note2**  
    - Content: "Uses gpt-4o-mini with temperature 0.2 for stable translations."  
    - Context: Notes model choice and parameter rationale.

#### 2.4 Response Preparation & Posting

- **Overview:**  
  Combines the original and translated texts into a formatted Slack message mentioning the user, then posts this message back to the Slack channel using the response URL provided in the slash command payload. The workflow merges parallel branches to synchronize.

- **Nodes Involved:**  
  - Merge (Combine Response)  
  - Prepare Response (Code)  
  - HTTP Request  
  - Sticky Note (comment)

- **Node Details:**  
  - **Merge (Combine Response)**  
    - Type: Merge  
    - Role: Combines outputs from the OpenAI translation and Detect Language nodes by position, synchronizing data for subsequent processing.  
    - Mode: Combine by position (index-based pairing).  
    - Input:  
      - Main 1: Output from OpenAI translation node  
      - Main 2: Output from Detect Language node  
    - Output: Combined JSON object with both translation and detection data.  
    - Edge Cases: Mismatched array lengths could cause missing or duplicated data.  
  - **Prepare Response**  
    - Type: Code node (JavaScript)  
    - Role:  
      - Extracts translated text safely from multiple possible GPT response fields.  
      - Retrieves original text and user_id from Detect Language node output.  
      - Creates a Slack message string that mentions the user, shows original text quoted, and then the translation on a new line.  
      - Supports alternative format (commented) to show only translation.  
      - Returns JSON with `reply` (formatted message) and `response_url`.  
    - Input: Combined data from Merge node.  
    - Output: JSON for HTTP Request node.  
    - Edge Cases: Missing translation fields fallback to "No content".  
  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Posts the translated reply back to Slack using the response URL.  
    - Config:  
      - URL: dynamic, from `response_url` field  
      - Method: POST  
      - Body parameters:  
        - `response_type`: "in_channel" (public message visible to all channel members)  
        - `text`: The formatted reply message  
      - Sends body as form parameters.  
    - Edge Cases:  
      - Invalid or expired response URL may cause failure to post message.  
      - Slack API rate limits or network errors.  
  - **Sticky Note**  
    - Content: "Posts the final translation back to Slack (in_channel by default).\nSwitch to ephemeral if you prefer private replies."  
    - Context: Explains message visibility setting and how to change it.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                          | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                   |
|---------------------------|---------------------------|----------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Webhook (Slash Command)   | Webhook                   | Receives `/trans` requests from Slack | None                         | Detect Language (Code), Code (Ack) | Receives /trans requests from Slack. Uses Production URL in Slack.                            |
| Code (Ack)                | Code                      | Sends immediate "Translating..." ack   | Webhook (Slash Command)       | Respond to Webhook           | Sends an immediate “Translating...” response to avoid Slack’s 3s timeout.                      |
| Respond to Webhook        | Respond to Webhook         | Responds HTTP 200 with ack text         | Code (Ack)                   | None                        |                                                                                               |
| Detect Language (Code)    | Code                      | Parses input, detects language, sets target | Webhook (Slash Command)       | OpenAI (Chat) - Translate, Merge (Combine Response) | Detects JA/EN automatically. Supports overrides like \nen: こんにちは \nand handles empty input. |
| OpenAI (Chat) - Translate | OpenAI Chat Completion    | Requests GPT-4o-mini translation        | Detect Language (Code)        | Merge (Combine Response)     | Uses gpt-4o-mini with temperature 0.2 for stable translations.                                |
| Merge (Combine Response)  | Merge                     | Combines detection and translation outputs | Detect Language (Code), OpenAI (Chat) - Translate | Prepare Response            |                                                                                               |
| Prepare Response          | Code                      | Formats Slack message with translation  | Merge (Combine Response)      | HTTP Request                |                                                                                               |
| HTTP Request              | HTTP Request              | Posts translated message back to Slack | Prepare Response             | None                        | Posts the final translation back to Slack (in_channel by default).\nSwitch to ephemeral if you prefer private replies. |
| Sticky Note4              | Sticky Note               | Comment on Webhook node                  | None                         | None                        | Receives /trans requests from Slack. Uses Production URL in Slack.                            |
| Sticky Note3              | Sticky Note               | Comment on Code (Ack) node               | None                         | None                        | Sends an immediate “Translating...” response to avoid Slack’s 3s timeout.                      |
| Sticky Note1              | Sticky Note               | Comment on Detect Language node          | None                         | None                        | Detects JA/EN automatically. Supports overrides like \nen: こんにちは \nand handles empty input. |
| Sticky Note2              | Sticky Note               | Comment on OpenAI node                   | None                         | None                        | Uses gpt-4o-mini with temperature 0.2 for stable translations.                                |
| Sticky Note               | Sticky Note               | Comment on HTTP Request node             | None                         | None                        | Posts the final translation back to Slack (in_channel by default).\nSwitch to ephemeral if you prefer private replies. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Webhook (Slash Command)`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `slack/trans`  
   - Response Mode: `responseNode` (deferred response)  
   - This node will receive JSON payloads from Slack slash command `/trans`.

2. **Create Code Node (Ack):**  
   - Name: `Code (Ack)`  
   - Type: Code  
   - Code (JavaScript):  
     ```js
     return [{ json: { ack: "Translating..." } }];
     ```
   - Connect Webhook node’s main output to this node.  
   - This node returns a quick acknowledgement to Slack.

3. **Create Respond to Webhook Node:**  
   - Name: `Respond to Webhook`  
   - Type: Respond to Webhook  
   - Response Code: 200  
   - Response Headers: `Content-Type: text/plain; charset=utf-8`  
   - Respond with: Text  
   - Response Body Expression: `{{$json["ack"]}}`  
   - Connect `Code (Ack)` output to this node.  
   - This node sends back the acknowledgement HTTP response to Slack.

4. **Create Code Node (Detect Language):**  
   - Name: `Detect Language (Code)`  
   - Type: Code  
   - Code (JavaScript):  
     ```js
     const body = $json.body || $json;
     const raw = (body.text || '').trim();

     if (!raw) {
       return [{ json: {
         text: '',
         target: '',
         response_url: body.response_url,
         user_id: body.user_id,
         channel_id: body.channel_id,
         error: 'Please provide text after /trans.'
       }}];
     }

     const m = raw.match(/^(\w{2}):\s*(.*)$/);
     let text = raw;
     let override = null;
     if (m && (m[1] === 'en' || m[1] === 'ja')) {
       override = m[1];
       text = m[2];
     }

     const hasJa = /[\p{Script=Hiragana}\p{Script=Katakana}\p{Script=Han}]/u.test(text);

     const target = override ?? (hasJa ? 'en' : 'ja');

     return [{
       json: {
         text,
         target,
         response_url: body.response_url,
         user_id: body.user_id,
         channel_id: body.channel_id
       }
     }];
     ```
   - Connect the main output of `Webhook (Slash Command)` to this node.  
   - This node extracts and prepares text and target language info.

5. **Create OpenAI Chat Node:**  
   - Name: `OpenAI (Chat) - Translate`  
   - Type: OpenAI Chat Completion (Langchain)  
   - Model: `gpt-4o-mini`  
   - Temperature: 0.2  
   - Messages:  
     - System role content:  
       ```
       You are a precise translator. Translate the input ONLY into the target language.
       - Preserve punctuation and line breaks.
       - Do NOT add explanations.
       - Keep style natural and concise.

       Target language: {{$json["target"]}}

       Input:
       {{$json["text"]}}
       ```
     - User role content: `{{$json["text"]}}`  
   - Credentials: Setup OpenAI API credentials in n8n and attach here.  
   - Connect the main output of `Detect Language (Code)` to this node.

6. **Create Merge Node:**  
   - Name: `Merge (Combine Response)`  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Connect:  
     - OpenAI Chat node main output to first input  
     - Detect Language node main output to second input  

7. **Create Code Node (Prepare Response):**  
   - Name: `Prepare Response`  
   - Type: Code  
   - Code (JavaScript):  
     ```js
     const j = $json;

     const translated =
       j.message?.content ??
       j.choices?.[0]?.message?.content ??
       j.data?.[0]?.content?.[0]?.text?.value ??
       j.response ??
       'No content';

     const detect = $node["Detect Language (Code)"].json;
     const original = detect.text;
     const userId = detect.user_id;

     const reply = `<@${userId}>\n> ${original}\n${translated}`;

     return [{ json: { reply, response_url: j.response_url } }];
     ```
   - Connect output of Merge node to this node.

8. **Create HTTP Request Node:**  
   - Name: `HTTP Request`  
   - Type: HTTP Request  
   - URL: Expression `{{$json["response_url"]}}`  
   - Method: POST  
   - Send Body: true  
   - Body Parameters (Form):  
     - `response_type`: `in_channel`  
     - `text`: `{{$json["reply"]}}`  
   - Connect output of `Prepare Response` to this node.  
   - This node sends the translated message back to Slack.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow immediately sends a “Translating...” message to Slack to avoid the platform’s 3-second timeout. | Slack API limitation for slash commands.         |
| Translation model chosen is GPT-4o-mini with low temperature 0.2 to ensure stable, precise translations.    | OpenAI GPT-4o-mini usage recommendation.         |
| Language detection uses Unicode script detection for Japanese characters and supports explicit target override via `en:` or `ja:` prefixes. | Heuristic to improve translation accuracy.       |
| Final Slack message posts publicly (`in_channel`) by default; can be changed to `ephemeral` for private replies. | Slack message visibility settings.                |
| Slack slash command must be configured with the webhook URL pointing to the deployed n8n webhook endpoint. | Slack App configuration requirement.              |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.