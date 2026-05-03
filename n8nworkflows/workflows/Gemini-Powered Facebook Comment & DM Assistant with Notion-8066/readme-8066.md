Gemini-Powered Facebook Comment & DM Assistant with Notion

https://n8nworkflows.xyz/workflows/gemini-powered-facebook-comment---dm-assistant-with-notion-8066


# Gemini-Powered Facebook Comment & DM Assistant with Notion

### 1. Workflow Overview

This workflow automates the moderation and response process for Facebook comments and direct messages (DMs) using AI powered by Google Gemini and integrates with Notion as a knowledge base and logging system. It targets social media managers and customer support teams who want to efficiently handle Facebook page comments with context-aware, sentiment-sensitive replies and DM follow-ups, while tracking processed interactions.

**Logical Blocks:**

- **1.1 Input Reception & Parsing:** Receives Facebook webhook events for new comments, parses comment data and metadata.
- **1.2 Duplicate & Source Check:** Checks if the comment is already processed or if it originates from the page itself to avoid loops.
- **1.3 Knowledge Base Retrieval & Formatting:** Queries Notion databases for relevant knowledge base info and formats it for AI consumption.
- **1.4 AI Comment Analysis & Response Generation:** Uses a Gemini-powered AI agent to generate JSON output with reply text, sentiment classification, and DM message.
- **1.5 Response Handling & Moderation Actions:** Posts public replies, sends DMs if required, hides inappropriate comments, and logs all processed comments in Notion.
- **1.6 Conditional Flow Control:** Uses multiple IF nodes for branching on comment type, sentiment, and reply content presence to determine subsequent actions.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Parsing

**Overview:**  
This block receives incoming Facebook webhook POST requests for comments, extracts and structures relevant comment details for downstream processing.

**Nodes Involved:**  
- Webhook1  
- If2  
- Parse Comms  
- Code2

**Node Details:**

- **Webhook1**  
  - *Type:* Webhook  
  - *Role:* Entry point receiving Facebook webhook events on a specified POST path.  
  - *Config:* HTTP POST, custom path, no query options.  
  - *Inputs:* External Facebook webhook requests.  
  - *Outputs:* Raw webhook JSON output.  
  - *Edge Cases:* Incorrect webhook signature, missing fields, or delayed webhook delivery could cause failure.

- **If2**  
  - *Type:* IF  
  - *Role:* Filters webhook events to only process newly added comments ("item" = "comment" and "verb" = "add").  
  - *Config:* Checks two conditions combined with AND.  
  - *Inputs:* Webhook1 output.  
  - *Outputs:* Routes only valid new comment events forward.  
  - *Edge Cases:* Other event types ignored; misconfiguration may skip valid comments.

- **Parse Comms**  
  - *Type:* Set  
  - *Role:* Extracts and assigns parsed variables from webhook JSON into structured fields (page_id, post_id, comment_id, comment_message, sender_id, etc.).  
  - *Config:* Uses expressions to parse deeply nested JSON fields.  
  - *Inputs:* If2 main output.  
  - *Outputs:* Parsed comment data JSON.  
  - *Edge Cases:* Missing fields or malformed webhook JSON may cause expression errors.

- **Code2**  
  - *Type:* Code (JavaScript)  
  - *Role:* Further processes parsed data to extract raw comment ID suffix for simplified reference.  
  - *Config:* Splits comment_id on underscore and keeps last segment.  
  - *Inputs:* Parse Comms output.  
  - *Outputs:* Adds comment_id_raw field to JSON.  
  - *Edge Cases:* Unexpected comment_id format may cause errors.

---

#### 2.2 Duplicate & Source Check

**Overview:**  
Verifies if the comment has been processed before or if it originates from the page itself, to avoid redundant processing or self-triggered loops.

**Nodes Involved:**  
- Check Database  
- Who's there? (IF node)

**Node Details:**

- **Check Database**  
  - *Type:* Notion Database Query  
  - *Role:* Queries Notion "Processed Facebook Comments" database filtering by comment ID and sender ID equal to page ID (indicating a previously processed or self comment).  
  - *Config:* Filters use "equals" condition on "Comment ID" and "sender id" rich text fields.  
  - *Inputs:* Code2 output (comment details).  
  - *Outputs:* List of matching database pages (empty if new comment).  
  - *Edge Cases:* Notion API rate limits, credential errors, or schema changes could break query.

- **Who's there?**  
  - *Type:* IF  
  - *Role:* Checks if the comment sender ID equals the page ID (comment from page itself).  
  - *Config:* Simple equality check with loose validation.  
  - *Inputs:* Arrange KB output (see next block for connection).  
  - *Outputs:* Skips AI processing for comments from the page itself.  
  - *Edge Cases:* Incorrect page ID or missing sender ID could cause false negatives.

---

#### 2.3 Knowledge Base Retrieval & Formatting

**Overview:**  
Fetches knowledge base entries from Notion, processes and formats them into different textual representations for AI prompt context.

**Nodes Involved:**  
- Fetch KB  
- Arrange KB

**Node Details:**

- **Fetch KB**  
  - *Type:* HTTP Request  
  - *Role:* Queries Notion API to retrieve up to 100 pages from the configured Knowledge Base database.  
  - *Config:* POST request with JSON body specifying page_size=100, authenticated via Notion credentials.  
  - *Inputs:* Check Database output.  
  - *Outputs:* Raw Notion pages JSON.  
  - *Edge Cases:* API limits, database permission issues, or schema changes.

- **Arrange KB**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes Notion API response, extracting and cleaning text properties from each page into multiple formats: flattened text (for AI prompt), raw text (for DMs), JSON strings, and lists.  
  - *Config:* Includes helper function to extract text from various Notion property types (title, rich_text, select, date, etc.), removes quotes, normalizes whitespace, and formats key-value pairs.  
  - *Inputs:* Fetch KB output.  
  - *Outputs:* JSON object with multiple database representations: databaseText, databaseRaw, databaseJSON, etc.  
  - *Edge Cases:* Unexpected property types, empty pages, or invalid JSON could cause failure.

---

#### 2.4 AI Comment Analysis & Response Generation

**Overview:**  
Uses Google Gemini powered language model with a LangChain AI Agent to analyze the comment, classify sentiment, and generate a public reply and private DM message.

**Nodes Involved:**  
- Comment  
- Google Gemini Chat Model  
- AI Agent  
- Separate rep&sent

**Node Details:**

- **Comment**  
  - *Type:* Facebook Graph API (GET)  
  - *Role:* Retrieves full comment details from Facebook using comment_id.  
  - *Config:* Uses Graph API v23.0, GET method, authenticated with Facebook API credentials.  
  - *Inputs:* "Who's there?" negative branch (i.e., comment not from page itself).  
  - *Outputs:* Comment details JSON for AI input.  
  - *Edge Cases:* API errors, deleted comments, or permission issues.

- **Google Gemini Chat Model**  
  - *Type:* LangChain LM Chat Google Gemini  
  - *Role:* Provides the underlying AI language model interface with temperature set to 0.1 for deterministic output.  
  - *Config:* Connected as AI language model credentialed with Google Gemini API.  
  - *Inputs:* AI Agent node.  
  - *Outputs:* AI-generated JSON response string.  
  - *Edge Cases:* API quota limits, network issues, or malformed responses.

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Constructs prompt with instructions and injects comment and knowledge base context to generate JSON response including reply, sentiment, and dm_message fields.  
  - *Config:* Complex prompt template with placeholders for COMPANY_NAME, INDUSTRY_TYPE, TARGET_LANGUAGE, etc. Parses comment from "Parse Comms" node.  
  - *Inputs:* Comment node output.  
  - *Outputs:* Raw AI JSON string output.  
  - *Edge Cases:* Parsing errors, invalid JSON output from AI, or prompt misconfiguration.

- **Separate rep&sent**  
  - *Type:* Set  
  - *Role:* Parses AI output JSON string to extract reply, sentiment, and dm_message into separate fields.  
  - *Config:* Uses JSON.parse with regex to remove ```json markdown code fences.  
  - *Inputs:* AI Agent output.  
  - *Outputs:* Structured fields for reply, sentiment, dm_message.  
  - *Edge Cases:* Invalid or incomplete AI output JSON.

---

#### 2.5 Response Handling & Moderation Actions

**Overview:**  
Based on sentiment and reply content, posts public replies, sends DMs, hides inappropriate comments, and logs the comment processing result in Notion.

**Nodes Involved:**  
- Appropriate? (IF)  
- Reply  
- Add to DB  
- If4 (IF)  
- Messages  
- If3 (IF)  
- Reply1  
- Delete Comms  
- Add to DB1  
- Messages1  
- Delete Comms1

**Node Details:**

- **Appropriate?**  
  - *Type:* IF  
  - *Role:* Checks if sentiment is "normal" or "price" to post a public reply; otherwise, routes to moderation steps.  
  - *Config:* OR condition on sentiment field.  
  - *Inputs:* Separate rep&sent output.  
  - *Outputs:*  
    - True: Post public reply  
    - False: Moderation flow (e.g., complaints, offensive).  
  - *Edge Cases:* Unexpected sentiment values.

- **Reply**  
  - *Type:* Facebook Graph API (POST)  
  - *Role:* Posts the public reply message on the comment.  
  - *Config:* Uses comment_id for node, message from reply field, Graph API v23.0.  
  - *Inputs:* Appropriate? true branch.  
  - *Outputs:* Confirmation response JSON.  
  - *Edge Cases:* API errors, comment deleted meanwhile.

- **Add to DB**  
  - *Type:* Notion Database Page Create  
  - *Role:* Logs the processed comment ID and status "Done" in Notion database.  
  - *Config:* Writes "Comment ID" and "Response Status" fields.  
  - *Inputs:* Reply node output.  
  - *Outputs:* Confirmation of DB entry creation.  
  - *Edge Cases:* API limits, schema mismatch.

- **If4**  
  - *Type:* IF  
  - *Role:* Checks if sentiment is "complement" to decide whether to send DM or not.  
  - *Config:* Sentiment equals "complement".  
  - *Inputs:* Add to DB output.  
  - *Outputs:*  
    - True: Skip DM  
    - False: Send DM  
  - *Edge Cases:* Sentiment misclassification.

- **Messages**  
  - *Type:* HTTP Request (Facebook Messages API)  
  - *Role:* Sends private DM message corresponding to the comment.  
  - *Config:* POST to Facebook messages endpoint with recipient comment_id and message text from dm_message.  
  - *Inputs:* If4 false branch.  
  - *Outputs:* Message send confirmation.  
  - *Edge Cases:* Message permissions, user blocking, API limits.

- **If3**  
  - *Type:* IF  
  - *Role:* Checks if reply text is non-empty to decide on posting reply or hiding comment.  
  - *Config:* Not empty condition on reply field.  
  - *Inputs:* Appropriate? false branch (for complaints, offensive, etc.).  
  - *Outputs:*  
    - True: Post reply (Reply1 node)  
    - False: Hide comment (Delete Comms1 node).  
  - *Edge Cases:* Empty or malformed reply.

- **Reply1**  
  - *Type:* Facebook Graph API (POST)  
  - *Role:* Posts public reply for complaints or other handled sentiments.  
  - *Config:* Same as Reply node but on different flow branch.  
  - *Inputs:* If3 true branch.  
  - *Outputs:* Reply confirmation.  
  - *Edge Cases:* Same as Reply.

- **Delete Comms1**  
  - *Type:* Facebook Graph API (POST)  
  - *Role:* Hides inappropriate or empty-reply comments by setting is_hidden=true.  
  - *Config:* POST to comment node with is_hidden parameter.  
  - *Inputs:* If3 false branch.  
  - *Outputs:* Hide confirmation.  
  - *Edge Cases:* Permissions, comment already deleted.

- **Delete Comms**  
  - *Type:* Facebook Graph API (POST)  
  - *Role:* Hides comment after posting reply in "normal" sentiment flow.  
  - *Config:* Same as Delete Comms1.  
  - *Inputs:* Reply1 output.  
  - *Outputs:* Confirmation.  
  - *Edge Cases:* Same as Delete Comms1.

- **Add to DB1**  
  - *Type:* Notion Database Page Create  
  - *Role:* Logs processed comment after hiding in DB.  
  - *Config:* Same as Add to DB.  
  - *Inputs:* Delete Comms output.  
  - *Outputs:* Confirmation.  
  - *Edge Cases:* Same as Add to DB.

- **Messages1**  
  - *Type:* HTTP Request (Facebook Messages API)  
  - *Role:* Sends DM message after hiding comment for negative sentiment.  
  - *Config:* Uses JSON body with recipient comment_id and message text from dm_message.  
  - *Inputs:* Add to DB1 output.  
  - *Outputs:* Confirmation.  
  - *Edge Cases:* Same as Messages.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                                | Input Node(s)                | Output Node(s)               | Sticky Note                                      |
|---------------------|----------------------------------|------------------------------------------------|------------------------------|------------------------------|-------------------------------------------------|
| Webhook1            | Webhook                          | Receives Facebook webhook events               | External webhook             | If2                          |                                                 |
| If2                 | IF                               | Filters for new comment add events              | Webhook1                    | Parse Comms                  |                                                 |
| Parse Comms         | Set                              | Parses comment and metadata fields               | If2                         | Code2                       |                                                 |
| Code2               | Code (JavaScript)                 | Extracts raw comment ID suffix                    | Parse Comms                 | Check Database               |                                                 |
| Check Database      | Notion Database Query             | Checks if comment already processed or self     | Code2                       | Fetch KB                    |                                                 |
| Fetch KB            | HTTP Request                     | Queries Notion Knowledge Base                     | Check Database              | Arrange KB                  |                                                 |
| Arrange KB          | Code (JavaScript)                 | Formats KB entries for AI prompt and DM          | Fetch KB                    | Who's there?                |                                                 |
| Who's there?        | IF                               | Checks if comment sender is the page itself      | Arrange KB                  | Comment (false branch)       |                                                 |
| Comment             | Facebook Graph API (GET)          | Fetches full comment details                       | Who's there? false branch   | AI Agent                    |                                                 |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | Provides AI language model interface               | AI Agent                    | AI Agent                    |                                                 |
| AI Agent            | LangChain Agent                  | Generates JSON reply, sentiment, dm_message       | Comment                     | Separate rep&sent           |                                                 |
| Separate rep&sent   | Set                              | Parses AI JSON output into separate fields         | AI Agent                    | Appropriate?                |                                                 |
| Appropriate?        | IF                               | Checks if sentiment allows public reply            | Separate rep&sent           | Reply (true), If3 (false)   |                                                 |
| Reply               | Facebook Graph API (POST)         | Posts public reply on comment                       | Appropriate? true branch    | Add to DB                   |                                                 |
| Add to DB           | Notion Database Page Create      | Logs processed comment with status "Done"          | Reply                       | If4                        |                                                 |
| If4                 | IF                               | Checks if sentiment is "complement" to skip DM     | Add to DB                   | Messages (false branch)     |                                                 |
| Messages            | HTTP Request (Facebook Messages) | Sends DM message to commenter                        | If4 false branch            |                            |                                                 |
| If3                 | IF                               | Checks if reply text is non-empty                     | Appropriate? false branch   | Reply1 (true), Delete Comms1 (false) |                                           |
| Reply1              | Facebook Graph API (POST)         | Posts reply for complaints/negative sentiment        | If3 true branch             | Delete Comms                |                                                 |
| Delete Comms        | Facebook Graph API (POST)         | Hides comment after reply                             | Reply1                      | Add to DB1                  |                                                 |
| Add to DB1          | Notion Database Page Create      | Logs processed comment after hiding                   | Delete Comms                | Messages1                   |                                                 |
| Messages1           | HTTP Request (Facebook Messages) | Sends DM after hiding comment                          | Add to DB1                  |                            |                                                 |
| Delete Comms1       | Facebook Graph API (POST)         | Hides comment when no reply is provided               | If3 false branch            | Add to DB1                  |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: [YOUR_WEBHOOK_PATH] (e.g., facebook-moderator-webhook)  
   - Save and activate webhook.

2. **Add IF Node (If2):**  
   - Conditions:  
     - `$json.body.entry[0].changes[0].value.item` equals "comment"  
     - `$json.body.entry[0].changes[0].value.verb` equals "add"  
   - Connect Webhook1 main output to If2 input.

3. **Add Set Node (Parse Comms):**  
   - Assign fields by extracting from webhook JSON:  
     - page_id = `$json.body.entry[0].id`  
     - post_id = `$json.body.entry[0].changes[0].value.post_id`  
     - comment_id = `$json.body.entry[0].changes[0].value.comment_id`  
     - comment_message = `$json.body.entry[0].changes[0].value.message`  
     - parent_id, comm_time, field, sender_id similarly parsed.  
   - Connect If2 true output to Parse Comms.

4. **Add Code Node (Code2):**  
   - JavaScript code: extract raw comment ID suffix from comment_id.  
   - Connect Parse Comms to Code2.

5. **Add Notion Node (Check Database):**  
   - Operation: getAll database pages from "Processed Facebook Comments" database.  
   - Filters: Comment ID equals `{{$json.comment_id}}` AND sender id equals "[YOUR_PAGE_ID]"  
   - Credentials: Notion API credentials.  
   - Connect Code2 to Check Database.

6. **Add HTTP Request Node (Fetch KB):**  
   - URL: `https://api.notion.com/v1/databases/[YOUR_KNOWLEDGE_BASE_DATABASE_ID]/query`  
   - Method: POST  
   - Body JSON: `{ "page_size": 100 }`  
   - Authentication: Notion API credentials.  
   - Connect Check Database to Fetch KB.

7. **Add Code Node (Arrange KB):**  
   - JavaScript code to parse Notion response, extract text fields, and produce multiple formatted outputs (databaseText, databaseRaw, etc.).  
   - Connect Fetch KB to Arrange KB.

8. **Add IF Node (Who's there?):**  
   - Condition: `$json.sender_id` equals `[YOUR_PAGE_ID]`  
   - Connect Arrange KB to Who's there?.  
   - True branch: end or ignore self comments.  
   - False branch: continue.

9. **Add Facebook Graph API Node (Comment):**  
   - Operation: GET comment by ID.  
   - Node: `{{$json.body.entry[0].changes[0].value.comment_id}}`  
   - Connect Who's there? false branch to Comment.

10. **Add LangChain AI Agent Node (AI Agent):**  
    - Configure prompt template with placeholders for company, language, and instructions. Inject comment message and knowledge base text.  
    - Connect Comment output to AI Agent input.

11. **Add Google Gemini LM Chat Node:**  
    - Model: Google Gemini Chat  
    - Temperature: 0.1  
    - Credentials: Google Gemini API credentials.  
    - Connect AI Agent ai_languageModel input to this node.

12. **Add Set Node (Separate rep&sent):**  
    - Parse AI Agent JSON output string to extract reply, sentiment, and dm_message fields using `JSON.parse`.  
    - Connect AI Agent output to Separate rep&sent.

13. **Add IF Node (Appropriate?):**  
    - Condition: sentiment equals "normal" OR "price"  
    - Connect Separate rep&sent output to Appropriate?.

14. **Add Facebook Graph API Node (Reply):**  
    - Operation: POST comment reply with message `{{$json.reply}}`.  
    - Node: comment_id from Parse Comms.  
    - Connect Appropriate? true branch to Reply.

15. **Add Notion Node (Add to DB):**  
    - Create page in "Processed Facebook Comments" with Comment ID and Response Status "Done".  
    - Connect Reply output to Add to DB.

16. **Add IF Node (If4):**  
    - Condition: sentiment equals "complement"  
    - Connect Add to DB output to If4.

17. **Add HTTP Request Node (Messages):**  
    - POST to Facebook messages endpoint with recipient.comment_id and message.text from dm_message field.  
    - Connect If4 false branch to Messages.

18. **Add IF Node (If3):**  
    - Condition: reply is not empty  
    - Connect Appropriate? false branch to If3.

19. **Add Facebook Graph API Node (Reply1):**  
    - POST comment reply (for complaints or other).  
    - Connect If3 true branch to Reply1.

20. **Add Facebook Graph API Node (Delete Comms):**  
    - POST to hide comment (is_hidden=true).  
    - Connect Reply1 output to Delete Comms.

21. **Add Notion Node (Add to DB1):**  
    - Log processed comment after hiding.  
    - Connect Delete Comms output to Add to DB1.

22. **Add HTTP Request Node (Messages1):**  
    - POST DM message after hiding comment.  
    - Connect Add to DB1 output to Messages1.

23. **Add Facebook Graph API Node (Delete Comms1):**  
    - POST to hide comment if no reply was generated.  
    - Connect If3 false branch to Delete Comms1.

24. **Connect Delete Comms1 output to Add to DB1** to log hidden comments.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini via LangChain for advanced AI natural language processing and response generation.                      | Gemini API credential required for AI Agent nodes.                                              |
| Facebook Graph API v23.0 is used for comment moderation, replies, and messaging.                                                          | Ensure Facebook app has required permissions (pages_manage_metadata, pages_read_engagement, pages_messaging). |
| Notion API is used to maintain a processed comments database and knowledge base for informed AI replies.                                 | Notion API credentials and correct database IDs are required.                                  |
| The AI prompt template uses placeholders like [COMPANY_NAME], [INDUSTRY_TYPE], [TARGET_LANGUAGE], etc., which must be replaced per deployment.| Customization needed to fit company branding and language requirements.                          |
| To avoid loops, comments from the page itself are filtered out early.                                                                     | Critical to prevent infinite processing cycles.                                                |
| The workflow hides comments with offensive or spam content automatically without public reply.                                            | Moderation compliance and content policy enforcement.                                          |
| The AI output is expected strictly in JSON format with fields reply, sentiment, dm_message to avoid parsing issues.                      | AI output validation is essential to avoid runtime errors.                                     |
| Facebook Messenger API for sending DMs uses comment_id as recipient identifier.                                                           | Ensure access and permissions for messaging via Facebook API.                                  |

---

**disclaimer** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.