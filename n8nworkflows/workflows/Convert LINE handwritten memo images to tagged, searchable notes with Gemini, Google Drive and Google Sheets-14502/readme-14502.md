Convert LINE handwritten memo images to tagged, searchable notes with Gemini, Google Drive and Google Sheets

https://n8nworkflows.xyz/workflows/convert-line-handwritten-memo-images-to-tagged--searchable-notes-with-gemini--google-drive-and-google-sheets-14502


# Convert LINE handwritten memo images to tagged, searchable notes with Gemini, Google Drive and Google Sheets

# 1. Workflow Overview

This workflow turns handwritten memo images sent through a LINE bot into structured notes that can be searched later by tag. It combines LINE Messaging API, Google Drive, Google Sheets, and Google Gemini to receive an image, extract readable content, generate a title/summary/tags, archive the image, store the structured note, and let users retrieve notes with `#tagname` or list all tags with `#taglist`.

## 1.1 Input Reception and Environment Setup
The workflow starts from a LINE webhook, then injects configuration data such as the LINE access token. From there, it decides whether the incoming message is an image, a tag-list command, a tag-search command, or an unsupported message.

## 1.2 Image Memo Processing
If the message is an image, the workflow immediately replies with a short processing message, downloads the binary image from LINE, uploads it to Google Drive, restores the binary payload for AI use, and sends the image to Gemini for OCR-style extraction and summarization.

## 1.3 OCR Parsing and Failure Handling
The AI response is parsed and normalized into safe JSON. If OCR fails or the model indicates unreadable text, the workflow pushes a failure notice to the user instead of saving data.

## 1.4 Data Storage and Completion Notification
If OCR succeeds, the structured note is appended to Google Sheets with timestamp, title, summary, tags, and the Google Drive image link. A push message is then sent to the user confirming the saved memo.

## 1.5 Tag List Retrieval
If the user sends `#taglist`, the workflow reads all memo rows from Google Sheets, extracts all distinct tags, sorts them, and pushes the list back to the user.

## 1.6 Tag Search Retrieval
If the user sends a text starting with `#`, the workflow extracts the tag, loads all saved memos, filters rows whose tags contain that value, formats the first five matches, and pushes the results to the user.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Routing

### Overview
This block receives the LINE webhook event, injects the LINE access token into the execution context, and routes the flow based on message type and command text. It is the main branching layer for all later logic.

### Nodes Involved
- `LINE_Receive_Webhook`
- `Config_Set_Environment`
- `Check_Image_Message`
- `Check_Tag_List_Command`
- `Check_Tag_Command`
- `LINE_Reply_NoImage`

### Node Details

#### LINE_Receive_Webhook
- **Type / role:** Webhook node; entry point for LINE Messaging API events.
- **Configuration choices:**  
  - HTTP method is `POST`.
  - Webhook path is dynamically set but effectively resolves to a fixed UUID-like path.
- **Key expressions / variables used:**  
  - Downstream nodes reference `body.events[0]` for message metadata, reply token, user ID, and message ID/type/text.
- **Input / output connections:**  
  - No input; outputs to `Config_Set_Environment`.
- **Version-specific requirements:**  
  - Uses webhook node type version `2.1`.
- **Edge cases / failures:**  
  - Invalid webhook URL in LINE console.
  - Non-LINE payloads missing `body.events[0]`.
  - Multiple events in one webhook are ignored because the workflow only uses the first event.
- **Sub-workflow reference:** None.

#### Config_Set_Environment
- **Type / role:** Set node; stores environment-like values used later.
- **Configuration choices:**  
  - Creates a field `LINE_ACCESS_TOKEN` with placeholder value `YOUR_ACCESS_TOKEN`.
- **Key expressions / variables used:**  
  - Exposes `LINE_ACCESS_TOKEN` to all HTTP LINE API calls.
- **Input / output connections:**  
  - Input from `LINE_Receive_Webhook`; output to `Check_Image_Message`.
- **Version-specific requirements:**  
  - Set node version `3.4`.
- **Edge cases / failures:**  
  - If not replaced with a real LINE Channel Access Token, every LINE API request will fail with 401/403.
- **Sub-workflow reference:** None.

#### Check_Image_Message
- **Type / role:** IF node; determines whether the inbound message is an image.
- **Configuration choices:**  
  - Checks `$('LINE_Receive_Webhook').item.json.body.events[0].message.type === "image"`.
  - Contains a second empty equals condition, which appears redundant.
- **Key expressions / variables used:**  
  - `body.events[0].message.type`
- **Input / output connections:**  
  - Input from `Config_Set_Environment`
  - True output -> `LINE_Reply_Processing`
  - False output -> `Check_Tag_List_Command`
- **Version-specific requirements:**  
  - IF node version `2.3`, condition engine version `3`.
- **Edge cases / failures:**  
  - If `message` or `type` is missing, the expression may resolve unexpectedly.
  - The redundant blank condition could confuse maintainers, though it likely evaluates harmlessly.
- **Sub-workflow reference:** None.

#### Check_Tag_List_Command
- **Type / role:** IF node; checks whether the text is exactly `#taglist`.
- **Configuration choices:**  
  - Compares `message.text` to `#taglist`.
- **Key expressions / variables used:**  
  - `$('LINE_Receive_Webhook').item.json.body.events[0].message.text`
- **Input / output connections:**  
  - Input from `Check_Image_Message`
  - True output -> `Sheets_Get_All_Memos_For_TagList`
  - False output -> `Check_Tag_Command`
- **Version-specific requirements:**  
  - IF node version `2.3`.
- **Edge cases / failures:**  
  - If the inbound non-image message is not text, `message.text` may be missing.
  - Command is case-sensitive and exact; `#TagList` or extra spaces will not match.
- **Sub-workflow reference:** None.

#### Check_Tag_Command
- **Type / role:** IF node; detects any text command beginning with `#`.
- **Configuration choices:**  
  - Uses `startsWith("#")` on `message.text`.
- **Key expressions / variables used:**  
  - `$('LINE_Receive_Webhook').item.json.body.events[0].message.text`
- **Input / output connections:**  
  - Input from `Check_Tag_List_Command`
  - True output -> `Extract_Tag`
  - False output -> `LINE_Reply_NoImage`
- **Version-specific requirements:**  
  - IF node version `2.3`.
- **Edge cases / failures:**  
  - Non-text payloads may not contain `message.text`.
  - Any `#...` string is treated as a tag search, including malformed values like `#`.
- **Sub-workflow reference:** None.

#### LINE_Reply_NoImage
- **Type / role:** HTTP Request; replies to unsupported non-image/non-command messages.
- **Configuration choices:**  
  - POST to `https://api.line.me/v2/bot/message/reply`
  - Sends JSON body with `replyToken` and instructional text.
  - Uses Bearer auth header from `Config_Set_Environment`.
- **Key expressions / variables used:**  
  - `replyToken`
  - `LINE_ACCESS_TOKEN`
- **Input / output connections:**  
  - Input from false branch of `Check_Tag_Command`
  - No downstream output.
- **Version-specific requirements:**  
  - HTTP Request node version `4.3`.
- **Edge cases / failures:**  
  - LINE reply tokens expire quickly; delays or retries may cause invalid reply token errors.
  - Invalid token or content-type issues produce LINE API auth/validation errors.
- **Sub-workflow reference:** None.

---

## 2.2 Image Retrieval and Archival

### Overview
This block handles valid image messages. It acknowledges the request to the user, fetches the image binary from LINE, uploads the file to Google Drive, then restores the binary stream for AI analysis because the Drive node output does not preserve the original binary in the shape expected downstream.

### Nodes Involved
- `LINE_Reply_Processing`
- `LINE_Download_ImageContent`
- `Drive_Upload_Image`
- `Restore_Image_Binary`

### Node Details

#### LINE_Reply_Processing
- **Type / role:** HTTP Request; immediate reply to let the user know processing has started.
- **Configuration choices:**  
  - POST to LINE reply endpoint.
  - Sends text: `Processing… please wait a moment.`
- **Key expressions / variables used:**  
  - `replyToken`
  - `LINE_ACCESS_TOKEN`
- **Input / output connections:**  
  - Input from true branch of `Check_Image_Message`
  - Output to `LINE_Download_ImageContent`
- **Version-specific requirements:**  
  - HTTP Request node version `4.3`.
- **Edge cases / failures:**  
  - Reply token expiration if webhook handling is delayed.
  - If this request fails, image processing still cannot continue because it is directly chained.
- **Sub-workflow reference:** None.

#### LINE_Download_ImageContent
- **Type / role:** HTTP Request; downloads original image binary from LINE content API.
- **Configuration choices:**  
  - GET `https://api-data.line.me/v2/bot/message/{messageId}/content`
  - Response format set to `file`.
  - Authorization header uses the LINE access token.
- **Key expressions / variables used:**  
  - `$('LINE_Receive_Webhook').item.json.body.events[0].message.id`
  - `$('Config_Set_Environment').item.json.LINE_ACCESS_TOKEN`
- **Input / output connections:**  
  - Input from `LINE_Reply_Processing`
  - Output to `Drive_Upload_Image`
- **Version-specific requirements:**  
  - HTTP Request node version `4.3`.
- **Edge cases / failures:**  
  - Invalid/expired LINE token.
  - Message content no longer available.
  - Large file download or timeout issues.
- **Sub-workflow reference:** None.

#### Drive_Upload_Image
- **Type / role:** Google Drive node; archives the image.
- **Configuration choices:**  
  - Uploads to `My Drive` in folder `LINE_PIC`.
  - File name is `{messageId}.jpg`.
  - Uses a configured Google Drive OAuth2 credential.
- **Key expressions / variables used:**  
  - File name from LINE message ID.
  - Later nodes use `webViewLink` from this node.
- **Input / output connections:**  
  - Input from `LINE_Download_ImageContent`
  - Output to `Restore_Image_Binary`
- **Version-specific requirements:**  
  - Google Drive node version `3`.
- **Edge cases / failures:**  
  - OAuth credential missing/expired.
  - Folder no longer exists or permission denied.
  - Binary property naming issues if the inbound file is not attached as expected.
- **Sub-workflow reference:** None.

#### Restore_Image_Binary
- **Type / role:** Code node; reattaches the original binary file to the current item.
- **Configuration choices:**  
  - Reads the first current item.
  - Replaces `item.binary` with `$node["LINE_Download_ImageContent"].binary`.
- **Key expressions / variables used:**  
  - `$node["LINE_Download_ImageContent"].binary`
- **Input / output connections:**  
  - Input from `Drive_Upload_Image`
  - Output to `AI_OCR_Analyze_Image`
- **Version-specific requirements:**  
  - Code node version `2`.
- **Edge cases / failures:**  
  - If the HTTP file download did not produce binary data, the AI image input will fail.
  - Assumes only one current item.
- **Sub-workflow reference:** None.

---

## 2.3 AI OCR and Structured Extraction

### Overview
This block sends the memo image to Gemini through an LLM chain configured for OCR-like extraction. The prompt forces pure JSON output containing `title`, `summary`, and up to three `tags`.

### Nodes Involved
- `AI_Model_Gemini`
- `AI_OCR_Analyze_Image`

### Node Details

#### AI_Model_Gemini
- **Type / role:** Google Gemini chat model node; language model provider for the chain.
- **Configuration choices:**  
  - `temperature: 0`
  - `topP: 1`
  - `maxOutputTokens: 2048`
  - Uses Google Gemini/PaLM API credential.
- **Key expressions / variables used:**  
  - No custom expressions; serves as the model backend.
- **Input / output connections:**  
  - Connected to `AI_OCR_Analyze_Image` via AI language model port.
- **Version-specific requirements:**  
  - LangChain Gemini node version `1`.
- **Edge cases / failures:**  
  - Invalid API key/credential.
  - Model quota/rate limiting.
  - Vision capability depends on the installed node version and service support.
- **Sub-workflow reference:** None.

#### AI_OCR_Analyze_Image
- **Type / role:** LangChain LLM Chain node; prompts the model to read the image and return normalized JSON.
- **Configuration choices:**  
  - Prompt explicitly says:
    - If unreadable, output JSON with summary `Unable to read the text.`
    - Otherwise return JSON only with `title`, `summary`, `tags`
    - No explanations or preamble
  - Message input includes a `HumanMessagePromptTemplate` of type `imageBinary`.
- **Key expressions / variables used:**  
  - Uses current item binary as image input.
- **Input / output connections:**  
  - Main input from `Restore_Image_Binary`
  - AI model input from `AI_Model_Gemini`
  - Main output to `Data_Parse_OCR_JSON`
- **Version-specific requirements:**  
  - Chain LLM node version `1.7`.
  - Requires a n8n version supporting binary image prompts in LangChain nodes.
- **Edge cases / failures:**  
  - Model may still wrap output in markdown fences despite prompt instructions.
  - Handwriting may be unreadable.
  - Image too large/low quality/oriented incorrectly.
- **Sub-workflow reference:** None.

---

## 2.4 OCR Parsing, Validation, and Failure Branch

### Overview
This block cleans and parses the AI response into safe fields, then checks whether OCR effectively failed. It protects the rest of the workflow from malformed JSON or inconsistent model outputs.

### Nodes Involved
- `Data_Parse_OCR_JSON`
- `AI_Check_OCR_Failure`
- `LINE_Push_NoSummary`

### Node Details

#### Data_Parse_OCR_JSON
- **Type / role:** Code node; sanitizes and parses AI response text into normalized JSON.
- **Configuration choices:**  
  - Reads `$json.text`.
  - Removes ```json fences and generic triple backticks.
  - Extracts the first JSON object substring.
  - Tries `JSON.parse`.
  - On parse failure, falls back to:
    - `title: ""`
    - `summary: raw.slice(0, 100)`
    - `tags: []`
  - Ensures defaults:
    - missing title -> `NoTitle`
    - missing summary -> `NoSummary`
    - invalid tags -> `[]`
    - limits tags to 3 strings
  - Merges parsed values back into original item and preserves binary.
- **Key expressions / variables used:**  
  - `$json["text"]`
  - `$input.first()`
- **Input / output connections:**  
  - Input from `AI_OCR_Analyze_Image`
  - Output to `AI_Check_OCR_Failure`
- **Version-specific requirements:**  
  - Code node version `2`.
- **Edge cases / failures:**  
  - If the model returns no JSON at all, fallback summary may contain arbitrary text.
  - If model returns malformed UTF-8 or unexpected structure, parsing may still degrade.
- **Sub-workflow reference:** None.

#### AI_Check_OCR_Failure
- **Type / role:** IF node; checks whether OCR was declared unreadable.
- **Configuration choices:**  
  - Tests whether `$json.summary === "Unable to read the text."`
- **Key expressions / variables used:**  
  - `$json.summary`
- **Input / output connections:**  
  - Input from `Data_Parse_OCR_JSON`
  - True output -> `LINE_Push_NoSummary`
  - False output -> `Sheets_Append_Row`
- **Version-specific requirements:**  
  - IF node version `2.3`.
- **Edge cases / failures:**  
  - Depends on exact string match.
  - If AI expresses failure differently and parser doesn’t normalize it, unreadable notes may still be saved.
- **Sub-workflow reference:** None.

#### LINE_Push_NoSummary
- **Type / role:** HTTP Request; informs the user OCR failed.
- **Configuration choices:**  
  - Uses LINE push endpoint instead of reply endpoint.
  - Message advises that text may be too small and suggests retaking the photo in a well-lit area.
- **Key expressions / variables used:**  
  - User ID from `source.userId`
  - `LINE_ACCESS_TOKEN`
- **Input / output connections:**  
  - Input from true branch of `AI_Check_OCR_Failure`
  - No downstream output.
- **Version-specific requirements:**  
  - HTTP Request node version `4.3`.
- **Edge cases / failures:**  
  - Push requires the bot to be allowed to push to the user.
  - If user ID is unavailable in the webhook context, push fails.
- **Sub-workflow reference:** None.

---

## 2.5 Storage and Completion Message

### Overview
When OCR succeeds, this block writes the memo record to Google Sheets and sends the user a completion message containing the generated title, tags, and summary.

### Nodes Involved
- `Sheets_Append_Row`
- `LINE_Push_Completion_Message`

### Node Details

#### Sheets_Append_Row
- **Type / role:** Google Sheets node; appends one memo record.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: `OCR_data`
  - Sheet/tab: `template`
  - Defines columns manually:
    - `date` -> current timestamp formatted `yyyy-MM-dd HH:mm:ss`
    - `title` -> parsed title
    - `summary` -> parsed summary
    - `webUrl` -> Google Drive `webViewLink`
    - `tags` -> tags joined by `, `
- **Key expressions / variables used:**  
  - `$now.format(...)`
  - `$json.title`
  - `$json.summary`
  - `$json.tags.join(", ")`
  - `$('Drive_Upload_Image').item.json.webViewLink`
- **Input / output connections:**  
  - Input from false branch of `AI_Check_OCR_Failure`
  - Output to `LINE_Push_Completion_Message`
- **Version-specific requirements:**  
  - Google Sheets node version `4.7`.
- **Edge cases / failures:**  
  - Column names in the target sheet must match configured schema.
  - OAuth permission issues or deleted spreadsheet.
  - `tags.join()` assumes `tags` is an array; parser enforces this.
- **Sub-workflow reference:** None.

#### LINE_Push_Completion_Message
- **Type / role:** HTTP Request; sends save confirmation to the user.
- **Configuration choices:**  
  - POST to LINE push endpoint.
  - Includes title, tags, summary, and reminder about `#tagname` and `#taglist`.
- **Key expressions / variables used:**  
  - `source.userId`
  - `$json.title`
  - `$json.tags`
  - `$json.summary`
  - `LINE_ACCESS_TOKEN`
- **Input / output connections:**  
  - Input from `Sheets_Append_Row`
  - No downstream output.
- **Version-specific requirements:**  
  - HTTP Request node version `4.3`.
- **Edge cases / failures:**  
  - Push permission or auth issues.
  - If the user blocked the bot, push may fail.
- **Sub-workflow reference:** None.

---

## 2.6 Tag List Generation

### Overview
This block supports the `#taglist` command. It loads all stored memo rows, extracts unique tags from the comma-separated `tags` column, sorts them, and returns a formatted list.

### Nodes Involved
- `Sheets_Get_All_Memos_For_TagList`
- `Build_TagList`
- `LINE_Push_TagList`

### Node Details

#### Sheets_Get_All_Memos_For_TagList
- **Type / role:** Google Sheets node; retrieves all stored memo rows.
- **Configuration choices:**  
  - Reads from spreadsheet `OCR_data`, sheet `template`.
  - No filtering configured.
- **Key expressions / variables used:**  
  - None beyond selected document/sheet references.
- **Input / output connections:**  
  - Input from true branch of `Check_Tag_List_Command`
  - Output to `Build_TagList`
- **Version-specific requirements:**  
  - Google Sheets node version `4.7`.
- **Edge cases / failures:**  
  - Empty sheet returns no rows.
  - OAuth or permission errors.
- **Sub-workflow reference:** None.

#### Build_TagList
- **Type / role:** Code node; produces a unique sorted tag list.
- **Configuration choices:**  
  - Iterates through all rows.
  - Reads `item.json.tags` as a comma-separated string.
  - Splits on commas, trims values, removes empties, adds to a `Set`, sorts the result.
  - Returns:
    - `tagList` array
    - `tagText` string joined with newline
- **Key expressions / variables used:**  
  - `$input.all()`
- **Input / output connections:**  
  - Input from `Sheets_Get_All_Memos_For_TagList`
  - Output to `LINE_Push_TagList`
- **Version-specific requirements:**  
  - Code node version `2`.
- **Edge cases / failures:**  
  - If stored tags use inconsistent separators, extraction quality drops.
  - Case-sensitive duplicates are not normalized, so `Study` and `study` may both appear.
- **Sub-workflow reference:** None.

#### LINE_Push_TagList
- **Type / role:** HTTP Request; sends the tag list back to the user.
- **Configuration choices:**  
  - Uses LINE push endpoint.
  - Message body text is `【Taglist】\n` plus the generated tag list.
  - Authorization header is set, but content-type header is omitted; n8n generally handles JSON mode correctly.
- **Key expressions / variables used:**  
  - `source.userId`
  - `$json.tagText`
  - `LINE_ACCESS_TOKEN`
- **Input / output connections:**  
  - Input from `Build_TagList`
  - No downstream output.
- **Version-specific requirements:**  
  - HTTP Request node version `4.3`.
- **Edge cases / failures:**  
  - Long tag lists may exceed LINE message length limits.
  - Empty tag list results in a near-empty message.
- **Sub-workflow reference:** None.

---

## 2.7 Tag Search

### Overview
This block supports `#tagname` searches. It extracts the requested tag from the message, retrieves memo rows from Google Sheets, filters by partial tag match, formats up to five hits, and pushes the result to the user.

### Nodes Involved
- `Extract_Tag`
- `Sheets_Get_All_Memos`
- `Filter_By_Tag`
- `Build_Search_Result`
- `LINE_Push_TagResults`

### Node Details

#### Extract_Tag
- **Type / role:** Code node; converts the inbound `#tag` message into a plain tag string.
- **Configuration choices:**  
  - Reads `message.text`
  - Removes `#`
  - Trims whitespace
  - Returns `{ tag }`
- **Key expressions / variables used:**  
  - `$('LINE_Receive_Webhook').first().json.body.events[0].message.text`
- **Input / output connections:**  
  - Input from true branch of `Check_Tag_Command`
  - Output to `Sheets_Get_All_Memos`
- **Version-specific requirements:**  
  - Code node version `2`.
- **Edge cases / failures:**  
  - `#` alone becomes an empty search tag.
  - Multi-word tags are allowed as literal text after `#`.
- **Sub-workflow reference:** None.

#### Sheets_Get_All_Memos
- **Type / role:** Google Sheets node; fetches all rows for later filtering.
- **Configuration choices:**  
  - Reads from the same spreadsheet `OCR_data`, sheet `template`.
- **Key expressions / variables used:**  
  - None.
- **Input / output connections:**  
  - Input from `Extract_Tag`
  - Output to `Filter_By_Tag`
- **Version-specific requirements:**  
  - Google Sheets node version `4.7`.
- **Edge cases / failures:**  
  - Large sheet means every search reads the full dataset, which may become slow at scale.
- **Sub-workflow reference:** None.

#### Filter_By_Tag
- **Type / role:** Code node; selects matching rows.
- **Configuration choices:**  
  - Reads search term from `Extract_Tag`.
  - Reads all incoming rows.
  - Keeps rows where `row.json.tags.toLowerCase().includes(searchTag.toLowerCase())`
  - Returns at most 5 results.
  - `alwaysOutputData: true` ensures downstream node still runs with no matches.
- **Key expressions / variables used:**  
  - `$items("Extract_Tag")[0].json.tag`
  - `$input.all()`
- **Input / output connections:**  
  - Input from `Sheets_Get_All_Memos`
  - Output to `Build_Search_Result`
- **Version-specific requirements:**  
  - Code node version `2`.
- **Edge cases / failures:**  
  - Uses substring matching, not exact tag matching; searching `meet` matches `meeting`.
  - Empty tag searches may match almost everything.
- **Sub-workflow reference:** None.

#### Build_Search_Result
- **Type / role:** Code node; formats the filtered rows into one message payload.
- **Configuration choices:**  
  - Filters out empty rows.
  - If none exist, returns: `No matching notes were found.\nSend #taglist to see a list of available tags.`
  - Otherwise builds a text block showing:
    - result number
    - title
    - summary
    - tags
- **Key expressions / variables used:**  
  - `$input.all()`
- **Input / output connections:**  
  - Input from `Filter_By_Tag`
  - Output to `LINE_Push_TagResults`
- **Version-specific requirements:**  
  - Code node version `2`.
- **Edge cases / failures:**  
  - Formatting assumes tags may be either array or string.
  - Large summaries may produce long LINE messages.
- **Sub-workflow reference:** None.

#### LINE_Push_TagResults
- **Type / role:** HTTP Request; sends the search result text to the user.
- **Configuration choices:**  
  - POST to LINE push endpoint.
  - Body is created via `JSON.stringify(...)`.
  - Includes content-type header and bearer token.
- **Key expressions / variables used:**  
  - `source.userId`
  - `$json.message`
  - `LINE_ACCESS_TOKEN`
- **Input / output connections:**  
  - Input from `Build_Search_Result`
  - No downstream output.
- **Version-specific requirements:**  
  - HTTP Request node version `4.3`.
- **Edge cases / failures:**  
  - Long message text may exceed LINE limits.
  - Push permission/auth problems.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| LINE_Receive_Webhook | n8n-nodes-base.webhook | Receives LINE Messaging API events |  | Config_Set_Environment | ## Receive message from LINE  This workflow starts when a user sends a message to the LINE bot. The LINE Messaging API sends an event to the webhook endpoint in n8n. The event contains: • userId • message type (image or text) • messageId This information is used to determine how the workflow should proceed. |
| Config_Set_Environment | n8n-nodes-base.set | Stores LINE access token for later API calls | LINE_Receive_Webhook | Check_Image_Message | ## Detect message type The workflow checks whether the incoming message is: • Image (handwritten memo) • Text command Image messages trigger the OCR pipeline. Text messages are treated as commands such as tag search. |
| Check_Image_Message | n8n-nodes-base.if | Routes image messages into OCR flow | Config_Set_Environment | LINE_Reply_Processing; Check_Tag_List_Command | ## Detect message type The workflow checks whether the incoming message is: • Image (handwritten memo) • Text command Image messages trigger the OCR pipeline. Text messages are treated as commands such as tag search. |
| LINE_Reply_Processing | n8n-nodes-base.httpRequest | Replies immediately that processing has started | Check_Image_Message | LINE_Download_ImageContent | ## Retrieve image from LINE The workflow retrieves the image file using the LINE Messaging API. The image binary data will be used for OCR processing and stored later. |
| LINE_Download_ImageContent | n8n-nodes-base.httpRequest | Downloads image binary from LINE content API | LINE_Reply_Processing | Drive_Upload_Image | ## Retrieve image from LINE The workflow retrieves the image file using the LINE Messaging API. The image binary data will be used for OCR processing and stored later. |
| Drive_Upload_Image | n8n-nodes-base.googleDrive | Uploads memo image to Google Drive | LINE_Download_ImageContent | Restore_Image_Binary | ## Save image to Google Drive The memo image is stored in Google Drive for long-term access. This allows users to open the original handwritten memo later from the stored link. |
| Restore_Image_Binary | n8n-nodes-base.code | Restores original image binary after Drive upload | Drive_Upload_Image | AI_OCR_Analyze_Image | ## Save image to Google Drive The memo image is stored in Google Drive for long-term access. This allows users to open the original handwritten memo later from the stored link. |
| AI_Model_Gemini | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini model to OCR chain |  | AI_OCR_Analyze_Image (AI model input) | ## AI OCR processing The uploaded image is analyzed using AI OCR. The AI extracts: • memo title • summary • tags • main text The result is returned as structured JSON. |
| AI_OCR_Analyze_Image | @n8n/n8n-nodes-langchain.chainLlm | Sends image to AI and requests JSON OCR summary | Restore_Image_Binary; AI_Model_Gemini | Data_Parse_OCR_JSON | ## AI OCR processing The uploaded image is analyzed using AI OCR. The AI extracts: • memo title • summary • tags • main text The result is returned as structured JSON. |
| Data_Parse_OCR_JSON | n8n-nodes-base.code | Parses and normalizes AI JSON output | AI_OCR_Analyze_Image | AI_Check_OCR_Failure | ## Parse AI response The JSON response from the AI is validated and parsed. This step ensures the workflow receives structured data before saving it into the database. |
| AI_Check_OCR_Failure | n8n-nodes-base.if | Detects unreadable-image response | Data_Parse_OCR_JSON | LINE_Push_NoSummary; Sheets_Append_Row | ## Parse AI response The JSON response from the AI is validated and parsed. This step ensures the workflow receives structured data before saving it into the database. |
| LINE_Push_NoSummary | n8n-nodes-base.httpRequest | Pushes OCR failure message to user | AI_Check_OCR_Failure |  | ## Parse AI response The JSON response from the AI is validated and parsed. This step ensures the workflow receives structured data before saving it into the database. |
| Sheets_Append_Row | n8n-nodes-base.googleSheets | Appends structured memo record to Google Sheets | AI_Check_OCR_Failure | LINE_Push_Completion_Message | ## Store memo data The memo data is stored in Google Sheets. Each record contains: • title • summary • tags • timestamp • image link. |
| LINE_Push_Completion_Message | n8n-nodes-base.httpRequest | Pushes save confirmation to user | Sheets_Append_Row |  | ## Send confirmation message After the memo is successfully processed and stored, a confirmation message is sent to the user via LINE. The message confirms that the memo has been saved. |
| Check_Tag_List_Command | n8n-nodes-base.if | Detects `#taglist` command | Check_Image_Message | Sheets_Get_All_Memos_For_TagList; Check_Tag_Command | ## Generate tag list The workflow collects all stored tags and generates a unique tag list. This helps users quickly see which topics are available in the memo database. |
| Sheets_Get_All_Memos_For_TagList | n8n-nodes-base.googleSheets | Loads all memo rows for tag-list generation | Check_Tag_List_Command | Build_TagList | ## Generate tag list The workflow collects all stored tags and generates a unique tag list. This helps users quickly see which topics are available in the memo database. |
| Build_TagList | n8n-nodes-base.code | Extracts unique sorted tags from sheet rows | Sheets_Get_All_Memos_For_TagList | LINE_Push_TagList | ## Generate tag list The workflow collects all stored tags and generates a unique tag list. This helps users quickly see which topics are available in the memo database. |
| LINE_Push_TagList | n8n-nodes-base.httpRequest | Pushes full tag list to user | Build_TagList |  | ## Generate tag list The workflow collects all stored tags and generates a unique tag list. This helps users quickly see which topics are available in the memo database. |
| Check_Tag_Command | n8n-nodes-base.if | Detects generic `#tag` search commands | Check_Tag_List_Command | Extract_Tag; LINE_Reply_NoImage | ## Tag search command Users can search memos by sending a tag command. Example #study #meeting The workflow filters memos that contain the specified tag and returns the results to the user. |
| Extract_Tag | n8n-nodes-base.code | Removes `#` and returns search tag | Check_Tag_Command | Sheets_Get_All_Memos | ## Tag search command Users can search memos by sending a tag command. Example #study #meeting The workflow filters memos that contain the specified tag and returns the results to the user. |
| Sheets_Get_All_Memos | n8n-nodes-base.googleSheets | Loads all memo rows for search | Extract_Tag | Filter_By_Tag | ## Tag search command Users can search memos by sending a tag command. Example #study #meeting The workflow filters memos that contain the specified tag and returns the results to the user. |
| Filter_By_Tag | n8n-nodes-base.code | Filters memo rows by tag substring match | Sheets_Get_All_Memos | Build_Search_Result | ## Tag search command Users can search memos by sending a tag command. Example #study #meeting The workflow filters memos that contain the specified tag and returns the results to the user. |
| Build_Search_Result | n8n-nodes-base.code | Formats search hits into one text response | Filter_By_Tag | LINE_Push_TagResults | ## Tag search command Users can search memos by sending a tag command. Example #study #meeting The workflow filters memos that contain the specified tag and returns the results to the user. |
| LINE_Push_TagResults | n8n-nodes-base.httpRequest | Pushes tag-search results to user | Build_Search_Result |  | ## Tag search command Users can search memos by sending a tag command. Example #study #meeting The workflow filters memos that contain the specified tag and returns the results to the user. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `LINE AI Handwritten Memo OCR & Tag Search System`.

2. **Add a Webhook node** named `LINE_Receive_Webhook`.
   - Method: `POST`
   - Path: use a unique path, for example the UUID-style path from the workflow.
   - Save the workflow and copy the production/test webhook URL.

3. **Configure LINE Messaging API** outside n8n.
   - Create a LINE Messaging API channel.
   - Enable webhook delivery.
   - Paste the n8n webhook URL into the LINE bot webhook setting.
   - Obtain a Channel Access Token.

4. **Add a Set node** named `Config_Set_Environment`.
   - Add field:
     - `LINE_ACCESS_TOKEN` = your real LINE Channel Access Token
   - Connect `LINE_Receive_Webhook -> Config_Set_Environment`.

5. **Add an IF node** named `Check_Image_Message`.
   - Condition: `LINE_Receive_Webhook.body.events[0].message.type` equals `image`
   - Connect `Config_Set_Environment -> Check_Image_Message`.

6. **Add an HTTP Request node** named `LINE_Reply_Processing`.
   - Method: `POST`
   - URL: `https://api.line.me/v2/bot/message/reply`
   - Body type: JSON
   - JSON body:
     - `replyToken`: from webhook event
     - `messages[0].type`: `text`
     - `messages[0].text`: `Processing… please wait a moment.`
   - Headers:
     - `Authorization: Bearer {{LINE_ACCESS_TOKEN}}`
     - `Content-type: application/json`
   - Connect true output of `Check_Image_Message -> LINE_Reply_Processing`.

7. **Add an HTTP Request node** named `LINE_Download_ImageContent`.
   - Method: `GET`
   - URL: `https://api-data.line.me/v2/bot/message/{messageId}/content`
   - Replace `{messageId}` with expression from webhook: `body.events[0].message.id`
   - Enable response as file/binary.
   - Header:
     - `Authorization: Bearer {{LINE_ACCESS_TOKEN}}`
   - Connect `LINE_Reply_Processing -> LINE_Download_ImageContent`.

8. **Create Google Drive OAuth2 credentials** in n8n.
   - Authorize access to the target Google account.
   - Create or choose a Drive folder for archived memo images.

9. **Add a Google Drive node** named `Drive_Upload_Image`.
   - Operation: upload file
   - File name: `{messageId}.jpg`
   - Drive: `My Drive`
   - Folder: choose your memo archive folder
   - Use the Google Drive credential
   - Connect `LINE_Download_ImageContent -> Drive_Upload_Image`.

10. **Add a Code node** named `Restore_Image_Binary`.
    - Use this code:
      ```javascript
      const item = $input.first();
      item.binary = $node["LINE_Download_ImageContent"].binary;
      return [item];
      ```
    - Connect `Drive_Upload_Image -> Restore_Image_Binary`.

11. **Create Google Gemini credentials** in n8n.
    - Use a valid Google Gemini / PaLM API credential supported by your n8n version.

12. **Add a Google Gemini Chat Model node** named `AI_Model_Gemini`.
    - Temperature: `0`
    - Top P: `1`
    - Max output tokens: `2048`

13. **Add a LangChain LLM Chain node** named `AI_OCR_Analyze_Image`.
    - Prompt type: define manually
    - Input message type: image binary
    - Prompt should instruct the model to:
      - read handwritten text from image
      - if unreadable, return:
        ```json
        {
          "title": "",
          "summary": "Unable to read the text.",
          "tags": []
        }
        ```
      - otherwise return JSON only:
        ```json
        {
          "title": "",
          "summary": "",
          "tags": ["", "", ""]
        }
        ```
    - Connect:
      - `Restore_Image_Binary -> AI_OCR_Analyze_Image`
      - `AI_Model_Gemini` to the AI model input port of `AI_OCR_Analyze_Image`

14. **Add a Code node** named `Data_Parse_OCR_JSON`.
    - Paste the parsing/sanitizing code from the workflow logic:
      - remove markdown fences
      - extract JSON
      - parse safely
      - default missing `title`, `summary`, `tags`
      - preserve binary
    - Connect `AI_OCR_Analyze_Image -> Data_Parse_OCR_JSON`.

15. **Add an IF node** named `AI_Check_OCR_Failure`.
    - Condition: `summary` equals `Unable to read the text.`
    - Connect `Data_Parse_OCR_JSON -> AI_Check_OCR_Failure`.

16. **Add an HTTP Request node** named `LINE_Push_NoSummary`.
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/push`
    - Body type: JSON
    - Send to `source.userId` from webhook
    - Message text should explain the image could not be read and suggest retaking it in good light
    - Headers:
      - `Authorization: Bearer {{LINE_ACCESS_TOKEN}}`
      - `Content-type: application/json`
    - Connect true branch of `AI_Check_OCR_Failure -> LINE_Push_NoSummary`.

17. **Prepare a Google Sheet** for memo storage.
    - Create a spreadsheet, e.g. `OCR_data`
    - Create a sheet/tab, e.g. `template`
    - Add columns:
      - `date`
      - `title`
      - `summary`
      - `webUrl`
      - `tags`

18. **Create Google Sheets OAuth2 credentials** in n8n.
    - Authorize access to the target spreadsheet.

19. **Add a Google Sheets node** named `Sheets_Append_Row`.
    - Operation: `append`
    - Select your spreadsheet and sheet
    - Map fields:
      - `date` = current timestamp formatted as `yyyy-MM-dd HH:mm:ss`
      - `title` = parsed title
      - `summary` = parsed summary
      - `webUrl` = `Drive_Upload_Image.webViewLink`
      - `tags` = join tags array with `, `
    - Connect false branch of `AI_Check_OCR_Failure -> Sheets_Append_Row`.

20. **Add an HTTP Request node** named `LINE_Push_Completion_Message`.
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/push`
    - Send to `source.userId`
    - Include title, tags, summary, and instructions for `#tagname` / `#taglist`
    - Headers:
      - `Authorization: Bearer {{LINE_ACCESS_TOKEN}}`
      - `Content-type: application/json`
    - Connect `Sheets_Append_Row -> LINE_Push_Completion_Message`.

21. **Add an IF node** named `Check_Tag_List_Command`.
    - Condition: `message.text` equals `#taglist`
    - Connect false branch of `Check_Image_Message -> Check_Tag_List_Command`.

22. **Add a Google Sheets node** named `Sheets_Get_All_Memos_For_TagList`.
    - Read rows from the same spreadsheet/sheet
    - Connect true branch of `Check_Tag_List_Command -> Sheets_Get_All_Memos_For_TagList`.

23. **Add a Code node** named `Build_TagList`.
    - Logic:
      - iterate through all rows
      - split each `tags` string by commas
      - trim and deduplicate with a Set
      - sort tags
      - return `tagList` and `tagText`
    - Connect `Sheets_Get_All_Memos_For_TagList -> Build_TagList`.

24. **Add an HTTP Request node** named `LINE_Push_TagList`.
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/push`
    - Send `【Taglist】\n` followed by the generated `tagText`
    - Header:
      - `Authorization: Bearer {{LINE_ACCESS_TOKEN}}`
    - Connect `Build_TagList -> LINE_Push_TagList`.

25. **Add an IF node** named `Check_Tag_Command`.
    - Condition: `message.text` starts with `#`
    - Connect false branch of `Check_Tag_List_Command -> Check_Tag_Command`.

26. **Add a Code node** named `Extract_Tag`.
    - Logic:
      - get inbound text
      - remove `#`
      - trim
      - return `{ tag }`
    - Connect true branch of `Check_Tag_Command -> Extract_Tag`.

27. **Add a Google Sheets node** named `Sheets_Get_All_Memos`.
    - Read all rows from the same sheet
    - Connect `Extract_Tag -> Sheets_Get_All_Memos`.

28. **Add a Code node** named `Filter_By_Tag`.
    - Logic:
      - get `searchTag` from `Extract_Tag`
      - keep rows where stored `tags` string includes `searchTag` case-insensitively
      - return first 5 rows
    - Enable “always output data”
    - Connect `Sheets_Get_All_Memos -> Filter_By_Tag`.

29. **Add a Code node** named `Build_Search_Result`.
    - Logic:
      - if no rows, output “No matching notes were found...”
      - otherwise format results with title, summary, and tags
    - Connect `Filter_By_Tag -> Build_Search_Result`.

30. **Add an HTTP Request node** named `LINE_Push_TagResults`.
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/push`
    - Send formatted search result text to `source.userId`
    - Headers:
      - `Authorization: Bearer {{LINE_ACCESS_TOKEN}}`
      - `Content-Type: application/json`
    - Connect `Build_Search_Result -> LINE_Push_TagResults`.

31. **Add an HTTP Request node** named `LINE_Reply_NoImage`.
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/reply`
    - Use `replyToken`
    - Return instructional text saying handwritten memo images are supported, `#tagname` searches tags, and `#taglist` lists tags
    - Headers:
      - `Authorization: Bearer {{LINE_ACCESS_TOKEN}}`
      - `Content-type: application/json`
    - Connect false branch of `Check_Tag_Command -> LINE_Reply_NoImage`.

32. **Optionally add sticky notes** matching the original canvas sections:
    - Receive message from LINE
    - Detect message type
    - Retrieve image from LINE
    - Save image to Google Drive
    - AI OCR processing
    - Parse AI response
    - Store memo data
    - Send confirmation message
    - Generate tag list
    - Tag search command

33. **Test the workflow** with these cases:
    - image message
    - unreadable image
    - plain text message
    - `#taglist`
    - `#existingtag`
    - `#unknowntag`

34. **Activate the workflow** and switch the LINE webhook to the production URL if needed.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow converts handwritten memo images sent via LINE into structured, searchable knowledge using AI. Users send a handwritten memo photo, the workflow performs OCR, summarizes the content, generates tags, stores results in Google Sheets, and archives the image in Google Drive. | Overall workflow purpose |
| Main features: AI OCR for handwritten memo recognition; automatic title and summary generation; tag extraction; image archiving in Google Drive; structured storage in Google Sheets; tag-based memo search via LINE; tag list generation. | Overall workflow capabilities |
| User commands: send an image to analyze and save it automatically. Send `#tagname` to search memos containing that tag. Send `#taglist` to display all tags currently stored. | User interaction model |
| Setup requirements before use: create a LINE Messaging API channel; obtain a Channel Access Token; prepare a Google Sheet for memo storage; prepare a Google Drive folder for image storage; set credentials in the Config node and Google nodes. | Deployment prerequisites |
| OCR accuracy may vary depending on handwriting quality. Very small or unclear text may not be recognized. The workflow is optimized for quick memo capture and organization. | Operational notes |