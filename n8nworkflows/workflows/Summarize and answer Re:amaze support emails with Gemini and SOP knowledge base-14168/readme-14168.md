Summarize and answer Re:amaze support emails with Gemini and SOP knowledge base

https://n8nworkflows.xyz/workflows/summarize-and-answer-re-amaze-support-emails-with-gemini-and-sop-knowledge-base-14168


# Summarize and answer Re:amaze support emails with Gemini and SOP knowledge base

# 1. Workflow Overview

This workflow automates first-line customer email handling for Re:amaze conversations using Gemini and an embedded SOP-style knowledge base. It fetches incoming messages from Re:amaze, keeps only the most recent valid customer messages, classifies each email into a support category, generates a controlled response grounded in documented company policies, and posts the reply back to Re:amaze.

It is designed for support teams that want semi-structured, production-oriented AI email handling with safeguards such as deduplication, customer-message filtering, category-driven behavior, and knowledge-base-constrained responses.

## 1.1 Entry and Conversation Retrieval
The workflow starts manually for testing, then calls the Re:amaze API to retrieve messages. It expands the API response into individual items and sorts them so the newest messages are considered first.

## 1.2 Deduplication and Customer Message Filtering
After retrieval, the workflow removes duplicate conversation entries using the conversation slug, then filters out non-customer or non-public messages. This prevents duplicate replies and avoids responding to internal or agent-originated messages.

## 1.3 Email Intent Classification
Each valid customer email is sent to Gemini for single-label classification. The classifier returns one category from a fixed taxonomy such as pricing, setup, security, escalation, spam, or misdirected.

## 1.4 AI Reply Generation with SOP Knowledge Base
The classified email is passed into an AI Agent. The agent uses:
- a Gemini chat model,
- a structured output parser,
- a code-based knowledge base tool containing SOP and policy facts.

The agent is instructed to rely on the documentation as the source of truth and return JSON with:
- `category`
- `draft_reply`

## 1.5 Response Preparation and Re:amaze Reply
The draft reply is escaped for safe JSON transport and then sent back to the corresponding Re:amaze conversation via API as a public customer-visible message.

---

# 2. Block-by-Block Analysis

## 2.1 Entry and Conversation Retrieval

### Overview
This block initializes the workflow and pulls message data from Re:amaze. It transforms the API response into one item per message and orders messages so the latest records are processed first.

### Nodes Involved
- Manual Trigger (Test Workflow)
- Fetch Conversations from Re:amaze API
- Split Conversations into Individual Messages
- Sort Messages by Latest First

### Node Details

#### Manual Trigger (Test Workflow)
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for testing and development.
- **Configuration choices:** No parameters configured.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - No input
  - Output → `Fetch Conversations from Re:amaze API`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime beyond manual execution context.
- **Sub-workflow reference:** None.

#### Fetch Conversations from Re:amaze API
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Re:amaze REST API to fetch messages.
- **Configuration choices:**  
  - Method defaults to GET
  - URL: `https://divyanshu.reamaze.com/api/v1/messages`
  - Uses generic credential authentication with HTTP Basic Auth
  - Sends `Accept: application/json`
- **Key expressions or variables used:** None in URL or headers.
- **Input and output connections:**  
  - Input ← `Manual Trigger (Test Workflow)`
  - Output → `Split Conversations into Individual Messages`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Invalid Re:amaze subdomain
  - Basic Auth credential failure
  - 401/403 authorization errors
  - 429 rate limiting
  - API response shape changes
  - Empty result set
- **Sub-workflow reference:** None.

#### Split Conversations into Individual Messages
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the `messages` array into one n8n item per message.
- **Configuration choices:**  
  - `fieldToSplitOut`: `messages`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input ← `Fetch Conversations from Re:amaze API`
  - Output → `Sort Messages by Latest First`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - If `messages` is missing or not an array, the node may fail or emit no usable items
  - Nested API schema changes can break downstream assumptions
- **Sub-workflow reference:** None.

#### Sort Messages by Latest First
- **Type and technical role:** `n8n-nodes-base.sort`  
  Orders items so newest messages are processed first.
- **Configuration choices:**  
  - Sort field: `created_at`
  - Order: descending
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input ← `Split Conversations into Individual Messages`
  - Output → `Remove Duplicate Conversations (Prevent Double Reply)`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Missing or malformed `created_at`
  - Lexicographic sorting issues if timestamps are not normalized
- **Sub-workflow reference:** None.

---

## 2.2 Deduplication and Customer Message Filtering

### Overview
This block ensures the workflow replies only once per conversation and only to legitimate customer-originated public messages. It reduces the risk of duplicate replies and avoids reacting to staff or internal notes.

### Nodes Involved
- Remove Duplicate Conversations (Prevent Double Reply)
- Filter Only Customer Emails

### Node Details

#### Remove Duplicate Conversations (Prevent Double Reply)
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Deduplicates items based on conversation identity.
- **Configuration choices:**  
  - Compare mode: selected fields
  - Field to compare: `conversation.slug`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input ← `Sort Messages by Latest First`
  - Output → `Filter Only Customer Emails`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing `conversation.slug`
  - Different messages in same conversation intentionally collapsed to one item
  - Depends on the preceding sort to preserve the latest message as the retained one
- **Sub-workflow reference:** None.

#### Filter Only Customer Emails
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters items to keep only customer-visible inbound emails.
- **Configuration choices:**  
  Three AND conditions are required:
  1. `origin == 1`
  2. `user['staff?'] == false`
  3. `visibility == 0`
- **Key expressions or variables used:**  
  - `{{ $json.origin }}`
  - `{{ $json.user['staff?'] }}`
  - `{{ $json.visibility }}`
- **Input and output connections:**  
  - Input ← `Remove Duplicate Conversations (Prevent Double Reply)`
  - True output → `Email Category Classifier`
  - False output is unused
- **Version-specific requirements:** Type version `2.3`, conditions format version `3`.
- **Edge cases or potential failure types:**  
  - Missing `user` object or missing `staff?` key
  - Numeric/string type mismatch if API changes field types
  - Misinterpretation of `origin` codes if Re:amaze changes semantics
  - Internal notes with unexpected visibility values
- **Sub-workflow reference:** None.

---

## 2.3 Email Intent Classification

### Overview
This block assigns a single support category to each customer email. The category then determines how the downstream agent searches documentation and frames the reply.

### Nodes Involved
- Email Category Classifier

### Node Details

#### Email Category Classifier
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Direct Gemini model invocation for intent classification.
- **Configuration choices:**  
  - Model: `models/gemini-2.5-flash`
  - JSON output enabled
  - Prompt instructs the model to classify into exactly one of:
    - pricing
    - setup
    - security
    - hr
    - escalate_sales
    - escalate_support
    - escalate_legal
    - spam
    - misdirected
  - The prompt includes:
    - `Subject: {{ $json.conversation.subject }}`
    - `Body: {{ $json.body }}`
  - Instruction: “Return only the category.”
- **Key expressions or variables used:**  
  - `{{ $json.conversation.subject }}`
  - `{{ $json.body }}`
- **Input and output connections:**  
  - Input ← `Filter Only Customer Emails`
  - Output → `AI Support Agent`
- **Version-specific requirements:** Type version `1.1`; requires configured Google Gemini/PaLM credential in n8n.
- **Edge cases or potential failure types:**  
  - Authentication or quota errors from Gemini
  - Model returns unexpected formatting instead of a bare category
  - Subject or body missing
  - Ambiguous emails that fit multiple classes
  - JSON output may still require downstream interpretation if model output shape changes
- **Sub-workflow reference:** None.

---

## 2.4 AI Reply Generation with SOP Knowledge Base

### Overview
This block generates the actual customer response. The AI agent receives the email content and category, can call a local code-based knowledge tool, and is constrained to produce structured JSON containing the final category and draft reply.

### Nodes Involved
- AI Support Agent
- LLM Response Generator
- Structured Response
- 📚 Knowledge Base

### Node Details

#### AI Support Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based agent orchestrating the LLM, output parser, and tool usage.
- **Configuration choices:**  
  - Prompt type: defined prompt
  - Main user prompt includes:
    - sender email from `Filter Only Customer Emails`
    - body from `Filter Only Customer Emails`
    - category from classifier output via `{{ $json.content.parts[0].text }}`
  - System message enforces:
    - documentation-first answering
    - no invention of features or policies
    - category-specific behavior
    - escalation only when documentation explicitly requires it
    - JSON output with exact schema:
      ```json
      {
        "category": "<final category>",
        "draft_reply": "<customer support reply>"
      }
      ```
  - Output parser is enabled
- **Key expressions or variables used:**  
  - `{{ $('Filter Only Customer Emails').item.json.user.email }}`
  - `{{ $('Filter Only Customer Emails').item.json.body }}`
  - `{{ $json.content.parts[0].text }}`
- **Input and output connections:**  
  - Main input ← `Email Category Classifier`
  - AI language model input ← `LLM Response Generator`
  - AI output parser input ← `Structured Response`
  - AI tool input ← `📚 Knowledge Base`
  - Main output → `Prepare Final Response`
- **Version-specific requirements:** Type version `3.1`; depends on LangChain-compatible nodes available in the n8n instance.
- **Edge cases or potential failure types:**  
  - If classifier output is not in `content.parts[0].text`, the category may be blank or invalid
  - If tool invocation fails, the agent may produce poor output or fail
  - Structured parser can reject malformed JSON
  - Hallucination risk is reduced by prompt, but not mathematically eliminated
  - Cross-item referencing with `$('Filter Only Customer Emails').item...` assumes aligned execution context
- **Sub-workflow reference:** None.

#### LLM Response Generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Gemini chat model used by the agent as its language model.
- **Configuration choices:**  
  - Temperature: `0`
  - Top P: `1`
  - This makes output more deterministic.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model output → `AI Support Agent`
- **Version-specific requirements:** Type version `1`; requires Google Gemini credential.
- **Edge cases or potential failure types:**  
  - Credential/quota/model access issues
  - Timeout on long agent/tool interactions
  - Model behavior changes across Gemini releases
- **Sub-workflow reference:** None.

#### Structured Response
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces structured JSON output for the agent response.
- **Configuration choices:**  
  - JSON schema example:
    ```json
    {
      "category": "...",
      "draft_reply": "..."
    }
    ```
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI output parser output → `AI Support Agent`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid JSON from agent
  - Missing required fields
  - Escaping issues inside generated reply text
- **Sub-workflow reference:** None.

#### 📚 Knowledge Base
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  Custom code tool serving as the agent’s SOP and policy knowledge source.
- **Configuration choices:**  
  - JavaScript tool with in-memory `docs` array
  - Covers categories:
    - pricing
    - setup
    - security
    - escalate_support
    - escalate_sales
    - escalate_legal
    - hr
    - spam
    - misdirected
  - Reads `const category = $json.category || ""`
  - Produces:
    - `category`
    - `category_context`
    - `full_context`
- **Key expressions or variables used:**  
  - `$json.category`
- **Input and output connections:**  
  - AI tool output → `AI Support Agent`
- **Version-specific requirements:** Type version `1.3`; code execution must be allowed in the environment.
- **Edge cases or potential failure types:**  
  - The tool expects a `category` field, but the agent may not always provide it cleanly during tool calls
  - If category is empty or invalid, `category_context` becomes empty
  - Knowledge base is hard-coded, so updates require workflow edits
  - Placeholder emails like `user@example.com` must be replaced before production use
- **Sub-workflow reference:** None.

---

## 2.5 Response Preparation and Re:amaze Reply

### Overview
This block sanitizes the generated draft reply for JSON transport and posts it back into the correct Re:amaze conversation. It is the final delivery stage of the automation.

### Nodes Involved
- Prepare Final Response
- Send Reply to Customer (Re:amaze API)

### Node Details

#### Prepare Final Response
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a cleaned string version of the AI-generated reply.
- **Configuration choices:**  
  - Adds field `clean_reply`
  - Expression escapes line breaks and double quotes:
    - replaces newline characters with `\\n`
    - replaces `"` with `\"`
- **Key expressions or variables used:**  
  - `{{ $json.output.draft_reply.replace(/\n/g, "\\n").replace(/"/g, '\\"') }}`
- **Input and output connections:**  
  - Input ← `AI Support Agent`
  - Output → `Send Reply to Customer (Re:amaze API)`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Fails if `output.draft_reply` does not exist
  - Double-escaping may occur depending on downstream serialization behavior
  - Special characters beyond quotes/newlines are not specifically normalized
- **Sub-workflow reference:** None.

#### Send Reply to Customer (Re:amaze API)
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the AI-generated reply back to Re:amaze as a public message.
- **Configuration choices:**  
  - Method: `POST`
  - URL:
    `https://divyanshu.reamaze.com/api/v1/conversations/{{ $('Filter Only Customer Emails').item.json.conversation.slug }}/messages`
  - Raw JSON body:
    ```json
    {
      "message": {
        "body": "{{ $json.clean_reply }}",
        "visibility": 0
      }
    }
    ```
  - Authentication: HTTP Basic Auth
  - Headers:
    - `Accept: application/json`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `{{ $('Filter Only Customer Emails').item.json.conversation.slug }}`
  - `{{ $json.clean_reply }}`
- **Input and output connections:**  
  - Input ← `Prepare Final Response`
  - No downstream nodes
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Conversation slug missing or stale
  - Re:amaze auth failure
  - Invalid JSON body due to escaping errors
  - API rejection if body is empty
  - Duplicate reply risk still exists across separate workflow runs if upstream retrieval overlaps over time
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger (Test Workflow) | n8n-nodes-base.manualTrigger | Manual workflow entry point for testing |  | Fetch Conversations from Re:amaze API | ## Try It Out!<br>### This template demonstrates a production-ready AI support automation system built specifically for Re:amaze, enabling automated email handling, intent classification & SOP-based decisioning (Knowledge Base).<br><br>🔹 Ideal for:<br>- Customer support teams handling high volumes of email queries (e.g., e-commerce, SaaS)<br>- Businesses looking to automate SOP-based responses with AI while integrating actions like order lookup or issue logging<br><br>### What this workflow does<br>- Fetches incoming customer emails from Re:amaze<br>- Filters only customer messages (ignores agent replies)<br>- Classifies email intent (e.g. security, pricing, setup etc.)<br>- Matches the request against a structured SOP knowledge base<br>- Generates a response using AI based strictly on SOP<br>- Sends reply back to the customer automatically<br><br><br>### Key Features:<br>- Prevents duplicate replies using deduplication logic<br>- Ensures responses follow SOP (reduces hallucination)<br>- Scalable for high-volume email handling<br><br>### Requirements<br>- Re:amaze API access (for fetching & replying to conversations)<br>- AI model (e.g., OpenAI / Gemini configured in n8n)<br>- SOP knowledge base (Google Sheets or Code tool)<br><br>### How to set up<br>1. Add your Re:amaze API credentials in HTTP nodes<br><br>To get the API Key, you have to login to the Re:amaze plateform(https://www.reamaze.com/)<br><br>Here is the API documentation -https://www.reamaze.com/api/get_messages<br><br>2. Configure your AI model credentials<br>3. Connect your SOP knowledge base (Google Sheets or internal tool)<br>4. Test the workflow using sample customer emails<br><br>🚀 This template can be extended to support multi-channel support (chat, tickets, etc.)<br><br>### Need Help?<br>ask in the [Forum](https://community.n8n.io/)!<br><br>Happy Hacking! |
| Fetch Conversations from Re:amaze API | n8n-nodes-base.httpRequest | Fetches message data from Re:amaze | Manual Trigger (Test Workflow) | Split Conversations into Individual Messages | ## 📥 Fetch & Prepare Conversations<br><br>This section retrieves conversations from Re:amaze using API.<br><br>Steps:<br>- Fetch all conversations<br>- Split them into individual messages<br>- Sort messages so the latest ones are processed first<br><br>This ensures the workflow always works on the most recent customer queries. |
| Split Conversations into Individual Messages | n8n-nodes-base.splitOut | Splits API message array into individual items | Fetch Conversations from Re:amaze API | Sort Messages by Latest First | ## 📥 Fetch & Prepare Conversations<br><br>This section retrieves conversations from Re:amaze using API.<br><br>Steps:<br>- Fetch all conversations<br>- Split them into individual messages<br>- Sort messages so the latest ones are processed first<br><br>This ensures the workflow always works on the most recent customer queries. |
| Sort Messages by Latest First | n8n-nodes-base.sort | Sorts messages descending by creation time | Split Conversations into Individual Messages | Remove Duplicate Conversations (Prevent Double Reply) | ## 📥 Fetch & Prepare Conversations<br><br>This section retrieves conversations from Re:amaze using API.<br><br>Steps:<br>- Fetch all conversations<br>- Split them into individual messages<br>- Sort messages so the latest ones are processed first<br><br>This ensures the workflow always works on the most recent customer queries. |
| Remove Duplicate Conversations (Prevent Double Reply) | n8n-nodes-base.removeDuplicates | Keeps one message per conversation slug | Sort Messages by Latest First | Filter Only Customer Emails | ## ⚠️ Deduplication & Filtering<br><br>This section ensures only valid customer messages are processed.<br><br>- Removes duplicate conversations to prevent multiple replies<br>- Filters only customer emails using:<br>  origin = 1 → Customer<br>  origin = 7 → Agent (ignored)<br><br>This guarantees:<br>✔ No duplicate replies<br>✔ No replies to internal/agent messages |
| Filter Only Customer Emails | n8n-nodes-base.if | Keeps only customer-originated public non-staff messages | Remove Duplicate Conversations (Prevent Double Reply) | Email Category Classifier | ## ⚠️ Deduplication & Filtering<br><br>This section ensures only valid customer messages are processed.<br><br>- Removes duplicate conversations to prevent multiple replies<br>- Filters only customer emails using:<br>  origin = 1 → Customer<br>  origin = 7 → Agent (ignored)<br><br>This guarantees:<br>✔ No duplicate replies<br>✔ No replies to internal/agent messages |
| Email Category Classifier | @n8n/n8n-nodes-langchain.googleGemini | Classifies email into one allowed intent category | Filter Only Customer Emails | AI Support Agent | ## 🧠 Email Intent Classification<br>The model analyzes the email and predicts a category.<br><br>Supported categories:<br>• setup<br>• pricing<br>• security<br>• hr<br>• escalate_support<br>• escalate_sales<br>• escalate_legal<br>• spam<br>• misdirected<br><br>This category guides how the AI agent responds. |
| AI Support Agent | @n8n/n8n-nodes-langchain.agent | Generates SOP-grounded structured support reply | Email Category Classifier; LLM Response Generator; Structured Response; 📚 Knowledge Base | Prepare Final Response | ## 🤖 AI Response Generation & Reply<br><br>This section generates and sends the final response.<br><br>- AI uses SOP-based knowledge to create accurate replies<br>- Ensures responses are consistent and controlled (no hallucination)<br>- Formats the response payload<br>- Sends reply back to customer via Re:amaze API<br><br>Result:<br>✔ Automated, high-quality customer support replies<br>✔ Consistent tone and policy adherence |
| LLM Response Generator | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini chat model for the AI agent |  | AI Support Agent | ## 🤖 AI Response Generation & Reply<br><br>This section generates and sends the final response.<br><br>- AI uses SOP-based knowledge to create accurate replies<br>- Ensures responses are consistent and controlled (no hallucination)<br>- Formats the response payload<br>- Sends reply back to customer via Re:amaze API<br><br>Result:<br>✔ Automated, high-quality customer support replies<br>✔ Consistent tone and policy adherence |
| Structured Response | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON response schema for the AI agent |  | AI Support Agent | ## 🤖 AI Response Generation & Reply<br><br>This section generates and sends the final response.<br><br>- AI uses SOP-based knowledge to create accurate replies<br>- Ensures responses are consistent and controlled (no hallucination)<br>- Formats the response payload<br>- Sends reply back to customer via Re:amaze API<br><br>Result:<br>✔ Automated, high-quality customer support replies<br>✔ Consistent tone and policy adherence |
| 📚 Knowledge Base | @n8n/n8n-nodes-langchain.toolCode | Supplies SOP/policy facts as an agent tool |  | AI Support Agent | ## 🤖 AI Response Generation & Reply<br><br>This section generates and sends the final response.<br><br>- AI uses SOP-based knowledge to create accurate replies<br>- Ensures responses are consistent and controlled (no hallucination)<br>- Formats the response payload<br>- Sends reply back to customer via Re:amaze API<br><br>Result:<br>✔ Automated, high-quality customer support replies<br>✔ Consistent tone and policy adherence |
| Prepare Final Response | n8n-nodes-base.set | Escapes generated reply for safe JSON posting | AI Support Agent | Send Reply to Customer (Re:amaze API) | ## 🤖 AI Response Generation & Reply<br><br>This section generates and sends the final response.<br><br>- AI uses SOP-based knowledge to create accurate replies<br>- Ensures responses are consistent and controlled (no hallucination)<br>- Formats the response payload<br>- Sends reply back to customer via Re:amaze API<br><br>Result:<br>✔ Automated, high-quality customer support replies<br>✔ Consistent tone and policy adherence |
| Send Reply to Customer (Re:amaze API) | n8n-nodes-base.httpRequest | Posts reply back to the Re:amaze conversation | Prepare Final Response |  | ## 🤖 AI Response Generation & Reply<br><br>This section generates and sends the final response.<br><br>- AI uses SOP-based knowledge to create accurate replies<br>- Ensures responses are consistent and controlled (no hallucination)<br>- Formats the response payload<br>- Sends reply back to customer via Re:amaze API<br><br>Result:<br>✔ Automated, high-quality customer support replies<br>✔ Consistent tone and policy adherence |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation for overall workflow purpose and setup |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation for fetch and preparation block |  |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for deduplication and filtering block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for classification block |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for AI reply and send block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Email Automation`.
   - Keep execution order as default unless you specifically need another mode.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name: `Manual Trigger (Test Workflow)`
   - No additional configuration required.

3. **Add the first HTTP Request node for Re:amaze retrieval**
   - Node type: `HTTP Request`
   - Name: `Fetch Conversations from Re:amaze API`
   - Method: `GET`
   - URL:  
     `https://YOUR_SUBDOMAIN.reamaze.com/api/v1/messages`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Basic Auth`
   - Add header:
     - `Accept: application/json`
   - Create/select Re:amaze Basic Auth credentials.
     - Use your Re:amaze API-compatible username/key according to your account setup.
   - Connect:
     - `Manual Trigger (Test Workflow)` → `Fetch Conversations from Re:amaze API`

4. **Add a Split Out node**
   - Node type: `Split Out`
   - Name: `Split Conversations into Individual Messages`
   - Field to split out: `messages`
   - Connect:
     - `Fetch Conversations from Re:amaze API` → `Split Conversations into Individual Messages`

5. **Add a Sort node**
   - Node type: `Sort`
   - Name: `Sort Messages by Latest First`
   - Sort by field:
     - `created_at`
   - Order:
     - `Descending`
   - Connect:
     - `Split Conversations into Individual Messages` → `Sort Messages by Latest First`

6. **Add a Remove Duplicates node**
   - Node type: `Remove Duplicates`
   - Name: `Remove Duplicate Conversations (Prevent Double Reply)`
   - Compare mode: `Selected Fields`
   - Field to compare:
     - `conversation.slug`
   - Connect:
     - `Sort Messages by Latest First` → `Remove Duplicate Conversations (Prevent Double Reply)`

7. **Add an IF node to filter valid customer messages**
   - Node type: `IF`
   - Name: `Filter Only Customer Emails`
   - Use `AND` combinator.
   - Add conditions:
     1. `{{ $json.origin }}` equals `1` as number
     2. `{{ $json.user['staff?'] }}` equals `false` as boolean
     3. `{{ $json.visibility }}` equals `0` as number
   - Connect:
     - `Remove Duplicate Conversations (Prevent Double Reply)` → `Filter Only Customer Emails`
   - Only the **true** branch is used in this workflow.

8. **Add a Gemini classification node**
   - Node type: `Google Gemini` from LangChain/n8n AI nodes
   - Name: `Email Category Classifier`
   - Model: `models/gemini-2.5-flash`
   - Enable JSON output if available in your installed version.
   - Add one message with content equivalent to:
     - classify the email into one category only:
       `pricing`, `setup`, `security`, `hr`, `escalate_sales`, `escalate_support`, `escalate_legal`, `spam`, `misdirected`
     - include:
       - subject: `{{ $json.conversation.subject }}`
       - body: `{{ $json.body }}`
     - instruct: `Return only the category.`
   - Configure Google Gemini credentials.
   - Connect:
     - `Filter Only Customer Emails` true output → `Email Category Classifier`

9. **Add an AI Agent node**
   - Node type: `AI Agent`
   - Name: `AI Support Agent`
   - Prompt type: `Define below`
   - Main prompt text:
     - Include customer email, sender address, body, and category
     - Use expressions:
       - `{{ $('Filter Only Customer Emails').item.json.user.email }}`
       - `{{ $('Filter Only Customer Emails').item.json.body }}`
       - `{{ $json.content.parts[0].text }}`
   - Add a system message enforcing:
     - documentation-first answering
     - no invented facts
     - pricing/setup/security specifics when applicable
     - escalation only when documentation says so
     - spam and misdirected behavior
     - exact JSON output with:
       - `category`
       - `draft_reply`

10. **Add the chat model used by the agent**
    - Node type: `Google Gemini Chat Model`
    - Name: `LLM Response Generator`
    - Set:
      - Temperature = `0`
      - Top P = `1`
    - Use the same or another valid Gemini credential.
    - Connect its AI model output to:
      - `AI Support Agent`

11. **Add a structured output parser**
    - Node type: `Structured Output Parser`
    - Name: `Structured Response`
    - Use a schema example like:
      ```json
      {
        "category": "...",
        "draft_reply": "..."
      }
      ```
    - Connect its AI output parser output to:
      - `AI Support Agent`

12. **Add the code-based knowledge tool**
    - Node type: `Tool Code`
    - Name: `📚 Knowledge Base`
    - Paste JavaScript that:
      - defines a `docs` array with SOP entries for pricing, setup, security, escalations, HR, spam, and misdirected
      - reads `const category = $json.category || ""`
      - builds:
        - `category_context`
        - `full_context`
      - returns:
        - `category`
        - `category_context`
        - `full_context`
    - At minimum, reproduce these categories and facts:
      - pricing plans: Starter, Professional, Enterprise
      - nonprofit discount
      - setup instructions and sync modes
      - security facts: encryption, SOC2, GDPR, HIPAA availability, SSO, IP whitelisting, audit logs, SOC2 report request
      - escalation contacts for support, sales, legal, HR
      - spam and misdirected handling
    - Replace placeholder email addresses such as `user@example.com` with real operational contacts before using in production.
    - Connect its AI tool output to:
      - `AI Support Agent`

13. **Connect the classifier to the agent**
    - Main connection:
      - `Email Category Classifier` → `AI Support Agent`

14. **Add a Set node to prepare the final reply**
    - Node type: `Set`
    - Name: `Prepare Final Response`
    - Add a string field:
      - Name: `clean_reply`
      - Value:
        `{{ $json.output.draft_reply.replace(/\n/g, "\\n").replace(/"/g, '\\"') }}`
    - Connect:
      - `AI Support Agent` → `Prepare Final Response`

15. **Add the final HTTP Request node to post the reply**
    - Node type: `HTTP Request`
    - Name: `Send Reply to Customer (Re:amaze API)`
    - Method: `POST`
    - URL:
      `https://YOUR_SUBDOMAIN.reamaze.com/api/v1/conversations/{{ $('Filter Only Customer Emails').item.json.conversation.slug }}/messages`
    - Authentication: `Generic Credential Type`
    - Generic Auth Type: `HTTP Basic Auth`
    - Send headers:
      - `Accept: application/json`
      - `Content-Type: application/json`
    - Send body: enabled
    - Content type: raw
    - Raw content type: `application/json`
    - Request body:
      ```json
      {
        "message": {
          "body": "{{ $json.clean_reply }}",
          "visibility": 0
        }
      }
      ```
    - Reuse the same Re:amaze credential if appropriate.
    - Connect:
      - `Prepare Final Response` → `Send Reply to Customer (Re:amaze API)`

16. **Optionally add sticky notes for maintainability**
    - Add notes documenting:
      - overall workflow purpose and requirements
      - fetch and prepare section
      - deduplication and filtering section
      - classification section
      - AI reply generation and delivery section

17. **Configure credentials**
    - **Re:amaze**
      - Create an `HTTP Basic Auth` credential
      - Use valid account/API authentication details for your Re:amaze workspace
    - **Google Gemini**
      - Create a `Google Gemini (PaLM) API` credential or the equivalent supported by your n8n version
      - Ensure the selected Gemini models are accessible in your account/project

18. **Test with representative customer emails**
    - Run the workflow manually.
    - Confirm:
      - API returns messages
      - split and sorting work correctly
      - duplicates are removed as expected
      - only inbound customer messages pass the IF node
      - classifier returns one allowed category
      - agent output is valid structured JSON
      - posted reply appears in the expected Re:amaze conversation

19. **Production hardening recommendations**
    - Replace the manual trigger with a scheduled trigger or webhook-driven ingestion if needed
    - Add error handling branches for:
      - API failures
      - LLM failures
      - parser errors
      - missing conversation slug
    - Consider persistent deduplication across executions, not just within a single run
    - Log agent category, reply body, and conversation slug for auditing

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows.  
The `📚 Knowledge Base` node is a tool node, not a sub-workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Re:amaze platform login is required to obtain API access details | https://www.reamaze.com/ |
| Re:amaze API documentation for message retrieval | https://www.reamaze.com/api/get_messages |
| n8n Community Forum for support and questions | https://community.n8n.io/ |
| The workflow is presented as a production-ready AI support automation example for Re:amaze | General project context |
| The design emphasizes deduplication, customer-only filtering, and SOP-grounded responses to reduce hallucination risk | General architecture note |
| The current knowledge base is embedded directly in a code tool and must be edited in the workflow to change plan details or escalation contacts | Implementation note |
| Placeholder escalation addresses such as `user@example.com` should be replaced with real business contacts before deployment | Deployment note |