Analyze Liveblocks comment attachments with Anthropic Claude AI

https://n8nworkflows.xyz/workflows/analyze-liveblocks-comment-attachments-with-anthropic-claude-ai-14300


# Analyze Liveblocks comment attachments with Anthropic Claude AI

# 1. Workflow Overview

This workflow listens for new comments created in a Liveblocks thread and automatically responds when the AI assistant user is mentioned. If the comment contains supported attachments, it analyzes those files with Anthropic Claude first and uses the analysis as context for the reply. If there are no attachments, it generates a direct text response.

## 1.1 Event Reception and Comment Retrieval

The workflow starts from a Liveblocks webhook trigger for `commentCreated` events. It then fetches the full comment object from Liveblocks so downstream logic can rely on the complete comment payload rather than only the webhook event envelope.

## 1.2 Mention Detection and Routing

The workflow checks whether the created comment mentions the AI assistant, identified here as `__AI_AGENT`. If not, execution stops. If yes, it branches again depending on whether attachments exist.

## 1.3 No-Attachment AI Reply

If the AI is mentioned but the comment has no attachments, the workflow sends the plain comment text to an Anthropic-backed AI Agent and posts the generated reply back into the same Liveblocks thread as user `__AI_AGENT`.

## 1.4 Attachment Enumeration and Retrieval

If the AI is mentioned and attachments are present, the workflow expands the attachments array into one item per attachment and fetches metadata for each attachment, including its URL and MIME type.

## 1.5 Attachment Analysis by Type

Each attachment is routed by MIME type:
- PDFs go to Anthropic document analysis
- Images go to Anthropic image analysis
- Unsupported file types are converted into a structured fallback message

## 1.6 Merge Analyses and Generate Final AI Response

All attachment-analysis outputs are merged into a single list, wrapped into one object, and passed to an AI Agent. The agent receives both the original user comment and the attachment-analysis results, then writes a final response.

## 1.7 Post Response to Liveblocks

The generated AI reply is posted into the original thread using the Liveblocks comment creation node, with the author explicitly set to `__AI_AGENT`.

---

# 2. Block-by-Block Analysis

## 2.1 Event Reception and Comment Retrieval

**Overview:**  
This block receives Liveblocks webhook events whenever a comment is created, then retrieves the full comment payload from the Liveblocks API. This ensures all later steps have access to fields such as attachments, mentions, room ID, thread ID, and plain-text body.

**Nodes Involved:**  
- Liveblocks Trigger  
- Get a comment

### Node: Liveblocks Trigger
- **Type and technical role:** `CUSTOM.liveblocksTrigger`  
  Webhook trigger node for Liveblocks events.
- **Configuration choices:**  
  Configured to listen only to the `commentCreated` event.
- **Key expressions or variables used:**  
  None in parameters; outputs webhook payload under `event`.
- **Input and output connections:**  
  Entry point of the workflow. Outputs to **Get a comment**.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Invalid webhook signing secret
  - Liveblocks webhook misconfiguration
  - Public webhook URL not reachable
  - Event shape changes causing downstream expression failures
- **Sub-workflow reference:**  
  None.

### Node: Get a comment
- **Type and technical role:** `CUSTOM.liveblocks`  
  Fetches a specific comment from Liveblocks.
- **Configuration choices:**  
  Resource is `comment`, operation is `getComment`.  
  Uses IDs from the webhook event:
  - `roomId = $json.event.data.roomId`
  - `threadId = $json.event.data.threadId`
  - `commentId = $json.event.data.commentId`
- **Key expressions or variables used:**  
  - `={{ $json.event.data.roomId }}`
  - `={{ $json.event.data.threadId }}`
  - `={{ $json.event.data.commentId }}`
- **Input and output connections:**  
  Input from **Liveblocks Trigger**. Output to **If**.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Missing event fields
  - Expired or invalid Liveblocks API credentials
  - Comment deleted before fetch
  - Permission issues on room/thread/comment
- **Sub-workflow reference:**  
  None.

---

## 2.2 Mention Detection and Attachment Routing

**Overview:**  
This block determines whether the AI assistant was mentioned and whether the comment has attachments. It routes execution either to a simple text-response branch or the attachment-analysis branch.

**Nodes Involved:**  
- If  
- If1

### Node: If
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional gate to ensure the workflow only continues when the AI assistant is mentioned.
- **Configuration choices:**  
  Checks whether `mentionedUserIds` contains `__AI_AGENT`.
- **Key expressions or variables used:**  
  - `={{ $json.mentionedUserIds }}`
  - Right value: `__AI_AGENT`
- **Input and output connections:**  
  Input from **Get a comment**.  
  True output goes to **If1**.  
  False output is unconnected, so execution ends.
- **Version-specific requirements:**  
  `typeVersion: 2.3`, conditions format version 3.
- **Edge cases or potential failure types:**  
  - `mentionedUserIds` missing or not an array
  - Type validation may reject malformed payloads
  - If the AI user ID differs from `__AI_AGENT`, nothing will happen
- **Sub-workflow reference:**  
  None.

### Node: If1
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether attachments are present while also re-checking the mention condition.
- **Configuration choices:**  
  Two conditions combined with `and`:
  1. `attachments` is not empty
  2. `mentionedUserIds` contains `__AI_AGENT`
- **Key expressions or variables used:**  
  - `={{ $json.attachments }}`
  - `={{ $json.mentionedUserIds }}`
- **Input and output connections:**  
  Input from **If**.  
  True output goes to **Code in JavaScript**.  
  False output goes to **AI Agent1**.
- **Version-specific requirements:**  
  `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - `attachments` missing or null
  - Redundant mention check may fail if fetched schema changes
  - Empty array correctly routes to no-attachment branch
- **Sub-workflow reference:**  
  None.

---

## 2.3 No-Attachment AI Reply

**Overview:**  
If the AI is mentioned but there are no attachments, the workflow sends the user’s plain-text comment directly to an Anthropic-powered AI Agent. The result is posted back to the same thread.

**Nodes Involved:**  
- AI Agent1  
- Anthropic Chat Model1  
- Create a comment

### Node: AI Agent1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain agent node that generates a reply from the comment text.
- **Configuration choices:**  
  Uses a custom prompt:
  - `Respond to the users message: {{ $('Get a comment').item.json.bodyPlain }}`
  Prompt type is `define`.
- **Key expressions or variables used:**  
  - `{{ $('Get a comment').item.json.bodyPlain }}`
- **Input and output connections:**  
  Main input from **If1** false branch.  
  Language model input from **Anthropic Chat Model1**.  
  Main output to **Create a comment**.
- **Version-specific requirements:**  
  `typeVersion: 3.1`.
- **Edge cases or potential failure types:**  
  - `bodyPlain` missing
  - Anthropic model errors
  - Output format differences depending on node version
- **Sub-workflow reference:**  
  None.

### Node: Anthropic Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Supplies the language model used by **AI Agent1**.
- **Configuration choices:**  
  Model selected: `claude-sonnet-4-5-20250929`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected via `ai_languageModel` to **AI Agent1**.
- **Version-specific requirements:**  
  `typeVersion: 1.3`.
- **Edge cases or potential failure types:**  
  - Invalid Anthropic API credentials
  - Model availability or account access issues
  - Rate limiting or token limits
- **Sub-workflow reference:**  
  None.

### Node: Create a comment
- **Type and technical role:** `CUSTOM.liveblocks`  
  Creates a reply comment in the original Liveblocks thread.
- **Configuration choices:**  
  Resource `comment`; create operation implied by comment creation fields.  
  Uses:
  - `roomId` from **Get a comment**
  - `threadId` from **Get a comment**
  - `createComment_userId = __AI_AGENT`
  - plain-text body mode
  - body text from `{{$json.output}}`
- **Key expressions or variables used:**  
  - `={{ $('Get a comment').first().json.roomId }}`
  - `={{ $('Get a comment').first().json.threadId }}`
  - `={{ $json.output }}`
- **Input and output connections:**  
  Input from **AI Agent1**. No downstream node.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Missing `output` field if agent response schema changes
  - Invalid room/thread IDs
  - Permission errors creating comments
- **Sub-workflow reference:**  
  None.

---

## 2.4 Attachment Enumeration and Retrieval

**Overview:**  
When attachments exist, this block converts the comment’s attachments array into one item per attachment and retrieves detailed attachment metadata from Liveblocks. This prepares each file for type-based analysis.

**Nodes Involved:**  
- Code in JavaScript  
- Get attachment1

### Node: Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`  
  Transforms the attachments array into a list of items, each containing only an attachment ID.
- **Configuration choices:**  
  JavaScript code:
  - reads the first input item
  - maps `attachments` into `{ id: item.id }`
- **Key expressions or variables used:**  
  Uses `$input.first().json.attachments`.
- **Input and output connections:**  
  Input from **If1** true branch. Output to **Get attachment1**.
- **Version-specific requirements:**  
  `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - `attachments` absent or not an array
  - attachment object missing `id`
  - empty array produces no downstream items
- **Sub-workflow reference:**  
  None.

### Node: Get attachment1
- **Type and technical role:** `CUSTOM.liveblocks`  
  Retrieves metadata for each attachment, including URL and MIME type.
- **Configuration choices:**  
  Resource `attachment`.  
  Uses:
  - `roomId` from **Get a comment**
  - `attachmentId` from current item
- **Key expressions or variables used:**  
  - `={{ $('Get a comment').item.json.roomId }}`
  - `={{ $json.id }}`
- **Input and output connections:**  
  Input from **Code in JavaScript**. Output to **Switch**.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Missing attachment ID
  - Attachment deleted or inaccessible
  - URL unavailable or expired depending on Liveblocks behavior
- **Sub-workflow reference:**  
  None.

---

## 2.5 Attachment Analysis by Type

**Overview:**  
This block routes each attachment according to MIME type and either analyzes it with Anthropic or emits a fallback message for unsupported formats. It is the core file-processing segment of the workflow.

**Nodes Involved:**  
- Switch  
- Analyze document1  
- Analyze image  
- Edit Fields

### Node: Switch
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes attachments to different outputs based on MIME type.
- **Configuration choices:**  
  Rules:
  1. exact match `application/pdf` → PDF branch
  2. starts with `image/` → image branch  
  Fallback output is `extra`, which goes to unsupported-file handling.
- **Key expressions or variables used:**  
  - `={{ $json.mimeType }}`
- **Input and output connections:**  
  Input from **Get attachment1**.  
  Output 0 → **Analyze document1**  
  Output 1 → **Analyze image**  
  Fallback/extra → **Edit Fields**
- **Version-specific requirements:**  
  `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - `mimeType` missing
  - Nonstandard MIME labels
  - Files that Anthropic theoretically supports but are not covered by these rules
- **Sub-workflow reference:**  
  None.

### Node: Analyze document1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.anthropic`  
  Sends PDF attachments to Anthropic document analysis.
- **Configuration choices:**  
  Resource `document`; model `claude-sonnet-4-6`; document URL comes from attachment metadata.
- **Key expressions or variables used:**  
  - `={{ $json.url }}`
- **Input and output connections:**  
  Input from **Switch** PDF branch. Output to **Merge** input 0.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - URL inaccessible to Anthropic
  - File too large
  - Unsupported PDF structure
  - Authentication or rate-limit errors
- **Sub-workflow reference:**  
  None.

### Node: Analyze image
- **Type and technical role:** `@n8n/n8n-nodes-langchain.anthropic`  
  Sends image attachments to Anthropic image analysis.
- **Configuration choices:**  
  Resource `image`; model `claude-sonnet-4-6`; image URL comes from the fetched attachment item.
- **Key expressions or variables used:**  
  - `={{ $('Get attachment1').item.json.url }}`
- **Input and output connections:**  
  Input from **Switch** image branch. Output to **Merge** input 1.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - URL inaccessible to Anthropic
  - Unsupported image format or oversized image
  - Model/account limitations
- **Sub-workflow reference:**  
  None.

### Node: Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a fallback analysis result for unsupported file types.
- **Configuration choices:**  
  Creates/overwrites:
  - `mimeType`
  - `name`
  - `message = "Cannot analyze files of this type"`
- **Key expressions or variables used:**  
  - `={{ $json.mimeType }}`
  - `={{ $json.name }}`
- **Input and output connections:**  
  Input from **Switch** fallback branch. Output to **Merge** input 2.
- **Version-specific requirements:**  
  `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - Missing `mimeType` or `name`
  - Downstream consumers may expect a different schema than Anthropic outputs
- **Sub-workflow reference:**  
  None.

---

## 2.6 Merge Analyses and Prepare Agent Input

**Overview:**  
This block recombines all attachment-analysis outputs into one list and wraps them as a single object. That object becomes the structured context passed into the final AI reply generation step.

**Nodes Involved:**  
- Merge  
- Code in JavaScript1

### Node: Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Collects results from PDF analysis, image analysis, and unsupported-file fallback streams.
- **Configuration choices:**  
  Configured with `numberInputs: 3`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input 0 from **Analyze document1**  
  Input 1 from **Analyze image**  
  Input 2 from **Edit Fields**  
  Output to **Code in JavaScript1**
- **Version-specific requirements:**  
  `typeVersion: 3.2`.
- **Edge cases or potential failure types:**  
  - Different input schemas across branches
  - Merge timing/behavior depends on node semantics in current n8n version
  - Empty branches may affect expected result aggregation
- **Sub-workflow reference:**  
  None.

### Node: Code in JavaScript1
- **Type and technical role:** `n8n-nodes-base.code`  
  Wraps all merged analysis items into a single object with a `result` array.
- **Configuration choices:**  
  JavaScript code:
  - `return { result: $input.all() };`
- **Key expressions or variables used:**  
  Uses `$input.all()`.
- **Input and output connections:**  
  Input from **Merge**. Output to **AI Agent**.
- **Version-specific requirements:**  
  `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - If upstream produces no items, result may be empty
  - Returned shape differs from standard array-returning patterns, so compatibility depends on code-node behavior in this version
- **Sub-workflow reference:**  
  None.

---

## 2.7 Final AI Response with Attachment Context

**Overview:**  
This block asks an AI Agent to craft a final response using both the original user comment and the structured attachment-analysis results. The response is then posted back into the same Liveblocks thread.

**Nodes Involved:**  
- AI Agent  
- Anthropic Chat Model  
- Create a comment1

### Node: AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Generates the final AI response using file-analysis context and the original comment.
- **Configuration choices:**  
  Prompt includes:
  - instruction that the AI is responding to a user comment
  - JSON stringified analysis results from `$json.result`
  - original comment text from **Get a comment**
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.result, null, 2) }}`
  - `{{ $('Get a comment').item.json.bodyPlain }}`
- **Input and output connections:**  
  Main input from **Code in JavaScript1**.  
  Language model input from **Anthropic Chat Model**.  
  Main output to **Create a comment1**.
- **Version-specific requirements:**  
  `typeVersion: 3.1`.
- **Edge cases or potential failure types:**  
  - Prompt formatting issue: mixed quote delimiters in the stored text may be confusing, though usually harmless
  - Very large attachment-analysis payload may exceed model limits
  - Missing `bodyPlain` or `result`
- **Sub-workflow reference:**  
  None.

### Node: Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Provides the language model for **AI Agent**.
- **Configuration choices:**  
  Model selected: `claude-sonnet-4-5-20250929`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected via `ai_languageModel` to **AI Agent**.
- **Version-specific requirements:**  
  `typeVersion: 1.3`.
- **Edge cases or potential failure types:**  
  - Invalid Anthropic credentials
  - Rate limiting
  - Model availability mismatch between account and selected model
- **Sub-workflow reference:**  
  None.

### Node: Create a comment1
- **Type and technical role:** `CUSTOM.liveblocks`  
  Posts the final AI-generated response back to the original thread.
- **Configuration choices:**  
  Resource `comment`; uses:
  - room ID from **Get a comment**
  - thread ID from **Get a comment**
  - author user ID `__AI_AGENT`
  - plain-text response body from agent output
- **Key expressions or variables used:**  
  - `={{ $('Get a comment').first().json.roomId }}`
  - `={{ $('Get a comment').first().json.threadId }}`
  - `={{ $json.output }}`
- **Input and output connections:**  
  Input from **AI Agent**. No downstream node.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Missing `output`
  - Permission errors writing into thread
  - Duplicate replies if webhook retries and execution is not deduplicated
- **Sub-workflow reference:**  
  None.

---

## 2.8 Documentation and In-Canvas Notes

**Overview:**  
These sticky notes document the purpose of each major section, setup requirements, and external resources. They do not affect execution but are important for maintainability and reproduction.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note8  
- Sticky Note9  
- Sticky Note10  
- Sticky Note12

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation for the trigger block.
- **Configuration choices:**  
  Content: “Comment created. When a comment is created in a room.”
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation for mention/attachment checks.
- **Configuration choices:**  
  Explains that the workflow checks for attachments and for mention of `__AI_AGENT`.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation for attachment fetching.
- **Configuration choices:**  
  Explains looping through attachments and fetching URLs.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation for posting the AI reply.
- **Configuration choices:**  
  Explains that the response is added as a comment in the same thread with user ID `__AI_AGENT`.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Large overview/setup note for the whole workflow.
- **Configuration choices:**  
  Includes implementation overview, setup checklist, Liveblocks documentation links, demo app link, dashboard link, and localtunnel example.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Annotation for attachment analysis routing.
- **Configuration choices:**  
  Explains supported vs unsupported file analysis behavior.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Annotation for the merge stage.
- **Configuration choices:**  
  Explains that analyses are merged back into one list.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Annotation for AI response generation.
- **Configuration choices:**  
  Explains that AI writes a response using analysis context.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note12
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Annotation for the no-attachment branch.
- **Configuration choices:**  
  Explains that AI responds directly when there are no attachments.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Liveblocks Trigger | CUSTOM.liveblocksTrigger | Receives `commentCreated` webhook events from Liveblocks |  | Get a comment | ## Comment created<br>When a comment is created in a room.<br><br>## Analyzing uploaded Liveblocks comments attachments with AI<br>This example uses [Liveblocks Comments](https://liveblocks.io/comments), collaborative commenting components for React. When an AI assistant is mentioned in a thread (e.g. "@AI Assistant"), it will automatically leave a response. Additionally, it will analyze any PDf or image attachments in the comments, and use them to help it respond. |
| Get a comment | CUSTOM.liveblocks | Fetches the full created comment from Liveblocks | Liveblocks Trigger | If | ## Check if AI mentioned and attachment uploaded<br>Check if a comment has an attachment, and if it mentions the AI agent. Their ID is `"__AI_AGENT"` in this demo.<br><br>## Analyzing uploaded Liveblocks comments attachments with AI<br>Setup includes inserting the secret key into "Get a comment", "Get a thread", and "Create a comment". |
| If | n8n-nodes-base.if | Checks whether `mentionedUserIds` contains `__AI_AGENT` | Get a comment | If1 | ## Check if AI mentioned and attachment uploaded<br>Check if a comment has an attachment, and if it mentions the AI agent. Their ID is `"__AI_AGENT"` in this demo. |
| If1 | n8n-nodes-base.if | Routes to attachment analysis or direct reply branch | If | Code in JavaScript, AI Agent1 | ## Check if AI mentioned and attachment uploaded<br>Check if a comment has an attachment, and if it mentions the AI agent. Their ID is `"__AI_AGENT"` in this demo. |
| Code in JavaScript | n8n-nodes-base.code | Expands attachments into one item per attachment ID | If1 | Get attachment1 | ## Fetch attachment IDs<br>If there are attachments, loop through every attachment and fetch its URL. |
| Get attachment1 | CUSTOM.liveblocks | Fetches metadata for each attachment | Code in JavaScript | Switch | ## Fetch attachment IDs<br>If there are attachments, loop through every attachment and fetch its URL. |
| Switch | n8n-nodes-base.switch | Routes files by MIME type | Get attachment1 | Analyze document1, Analyze image, Edit Fields | ## Analyze attachments<br>If Anthropic can analyze the attachment (i.e PDF or image), feed it through an analyzer. Otherwise, say you can't analyze the file type. |
| Analyze document1 | @n8n/n8n-nodes-langchain.anthropic | Analyzes PDF attachments | Switch | Merge | ## Analyze attachments<br>If Anthropic can analyze the attachment (i.e PDF or image), feed it through an analyzer. Otherwise, say you can't analyze the file type. |
| Analyze image | @n8n/n8n-nodes-langchain.anthropic | Analyzes image attachments | Switch | Merge | ## Analyze attachments<br>If Anthropic can analyze the attachment (i.e PDF or image), feed it through an analyzer. Otherwise, say you can't analyze the file type. |
| Edit Fields | n8n-nodes-base.set | Produces fallback output for unsupported file types | Switch | Merge | ## Analyze attachments<br>If Anthropic can analyze the attachment (i.e PDF or image), feed it through an analyzer. Otherwise, say you can't analyze the file type. |
| Merge | n8n-nodes-base.merge | Recombines analysis outputs from all branches | Analyze document1, Analyze image, Edit Fields | Code in JavaScript1 | ## Merge the analyses<br>Merge the analyses back into a single list |
| Code in JavaScript1 | n8n-nodes-base.code | Wraps all analysis items into a `result` array | Merge | AI Agent | ## Merge the analyses<br>Merge the analyses back into a single list |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generates a final reply using comment text plus attachment analysis | Code in JavaScript1; Anthropic Chat Model (AI model input) | Create a comment1 | ## Create AI response<br>Ask AI to write a response to the original comment, with its analysis context |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Provides chat model for attachment-aware response generation |  | AI Agent | ## Create AI response<br>Ask AI to write a response to the original comment, with its analysis context |
| Create a comment1 | CUSTOM.liveblocks | Posts attachment-aware AI response to the thread | AI Agent |  | ## Add response to thread<br>The response is then added as a comment in the same thread. The user ID is set to `"__AI_AGENT"`. |
| AI Agent1 | @n8n/n8n-nodes-langchain.agent | Generates a direct reply when no attachments are present | If1; Anthropic Chat Model1 (AI model input) | Create a comment | ## Ignore attachments and respond<br>If there are no attachments, AI generates a response to the comment, and it's added to the thread. The user ID is set to `"__AI_AGENT"`. |
| Anthropic Chat Model1 | @n8n/n8n-nodes-langchain.lmChatAnthropic | Provides chat model for no-attachment replies |  | AI Agent1 | ## Ignore attachments and respond<br>If there are no attachments, AI generates a response to the comment, and it's added to the thread. The user ID is set to `"__AI_AGENT"`. |
| Create a comment | CUSTOM.liveblocks | Posts direct AI response to the thread | AI Agent1 |  | ## Ignore attachments and respond<br>If there are no attachments, AI generates a response to the comment, and it's added to the thread. The user ID is set to `"__AI_AGENT"`. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note12 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it a name similar to:  
   `Analyzing uploaded Liveblocks comments attachments with AI`.

2. **Add a Liveblocks Trigger node** named **Liveblocks Trigger**.
   - Type: `CUSTOM.liveblocksTrigger`
   - Event: `commentCreated`
   - Credentials: configure a **Liveblocks Webhook Signing Secret**
   - Keep additional fields empty unless your Liveblocks setup requires more filtering.

3. **Configure Liveblocks webhooks in the Liveblocks dashboard**.
   - Create a webhook for `commentCreated`
   - Use the production webhook URL from the n8n trigger
   - If running locally, expose n8n with a public tunnel such as `ngrok` or `localtunnel`

4. **Add a Liveblocks node** named **Get a comment**.
   - Type: `CUSTOM.liveblocks`
   - Resource: `comment`
   - Operation: `getComment`
   - Credentials: Liveblocks API credential
   - Set:
     - `roomId = {{ $json.event.data.roomId }}`
     - `threadId = {{ $json.event.data.threadId }}`
     - `commentId = {{ $json.event.data.commentId }}`
   - Connect **Liveblocks Trigger → Get a comment**

5. **Add an If node** named **If**.
   - Condition: array contains
   - Left value: `{{ $json.mentionedUserIds }}`
   - Right value: `__AI_AGENT`
   - Connect **Get a comment → If**
   - Leave the false branch unconnected so non-mentioned comments stop.

6. **Add another If node** named **If1**.
   - Combine conditions with `AND`
   - Condition 1: array `attachments` is not empty
     - Left value: `{{ $json.attachments }}`
   - Condition 2: array `mentionedUserIds` contains `__AI_AGENT`
     - Left value: `{{ $json.mentionedUserIds }}`
     - Right value: `__AI_AGENT`
   - Connect **If (true) → If1**

## Build the no-attachment branch

7. **Add an AI Agent node** named **AI Agent1**.
   - Type: `@n8n/n8n-nodes-langchain.agent`
   - Prompt type: `define`
   - Text:
     `Respond to the users message: {{ $('Get a comment').item.json.bodyPlain }}`
   - Connect **If1 false → AI Agent1**

8. **Add an Anthropic Chat Model node** named **Anthropic Chat Model1**.
   - Type: `@n8n/n8n-nodes-langchain.lmChatAnthropic`
   - Credentials: Anthropic API
   - Model: `claude-sonnet-4-5-20250929`
   - Connect this node to **AI Agent1** via the AI language model port.

9. **Add a Liveblocks node** named **Create a comment**.
   - Type: `CUSTOM.liveblocks`
   - Resource: `comment`
   - Configure it for comment creation
   - Set:
     - `roomId = {{ $('Get a comment').first().json.roomId }}`
     - `threadId = {{ $('Get a comment').first().json.threadId }}`
     - `createComment_userId = __AI_AGENT`
     - `createComment_bodyMode = plainText`
     - `createComment_bodyText = {{ $json.output }}`
   - Connect **AI Agent1 → Create a comment**

## Build the attachment-processing branch

10. **Add a Code node** named **Code in JavaScript**.
    - Type: `n8n-nodes-base.code`
    - Language: JavaScript
    - Code:
      ```javascript
      return $input.first().json.attachments.map((item) => ({ id: item.id }));
      ```
    - Connect **If1 true → Code in JavaScript**

11. **Add a Liveblocks node** named **Get attachment1**.
    - Type: `CUSTOM.liveblocks`
    - Resource: `attachment`
    - Credentials: Liveblocks API
    - Set:
      - `roomId = {{ $('Get a comment').item.json.roomId }}`
      - `attachmentId = {{ $json.id }}`
    - Connect **Code in JavaScript → Get attachment1**

12. **Add a Switch node** named **Switch**.
    - Type: `n8n-nodes-base.switch`
    - Create rule 1:
      - `{{ $json.mimeType }}` equals `application/pdf`
    - Create rule 2:
      - `{{ $json.mimeType }}` starts with `image/`
    - Set fallback output to `extra`
    - Connect **Get attachment1 → Switch**

13. **Add an Anthropic node** named **Analyze document1**.
    - Type: `@n8n/n8n-nodes-langchain.anthropic`
    - Resource: `document`
    - Credentials: Anthropic API
    - Model: `claude-sonnet-4-6`
    - Document URLs:
      - `{{ $json.url }}`
    - Connect **Switch output 0 → Analyze document1**

14. **Add another Anthropic node** named **Analyze image**.
    - Type: `@n8n/n8n-nodes-langchain.anthropic`
    - Resource: `image`
    - Credentials: Anthropic API
    - Model: `claude-sonnet-4-6`
    - Image URLs:
      - `{{ $('Get attachment1').item.json.url }}`
    - Connect **Switch output 1 → Analyze image**

15. **Add a Set node** named **Edit Fields**.
    - Type: `n8n-nodes-base.set`
    - Add fields:
      - `mimeType` = `{{ $json.mimeType }}`
      - `name` = `{{ $json.name }}`
      - `message` = `Cannot analyze files of this type`
    - Connect **Switch fallback/extra → Edit Fields**

16. **Add a Merge node** named **Merge**.
    - Type: `n8n-nodes-base.merge`
    - Set `numberInputs` to `3`
    - Connect:
      - **Analyze document1 → Merge input 0**
      - **Analyze image → Merge input 1**
      - **Edit Fields → Merge input 2**

17. **Add a Code node** named **Code in JavaScript1**.
    - Type: `n8n-nodes-base.code`
    - Use:
      ```javascript
      return { result: $input.all() };
      ```
    - Connect **Merge → Code in JavaScript1**

18. **Add an AI Agent node** named **AI Agent**.
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Prompt type: `define`
    - Text:
      ```text
      You're responding to a comment left by a users. There are some file attachments attached to the comment. You've just done an analysis of the images, this is your analysis:

      ```
      {{ JSON.stringify($json.result, null, 2) }}
      ```

      Respond to the users prompt using the file analysis reply. This is what the user asked you:

      """
      {{ $('Get a comment').item.json.bodyPlain }}
      ''''
      ```
    - Connect **Code in JavaScript1 → AI Agent**

19. **Add an Anthropic Chat Model node** named **Anthropic Chat Model**.
    - Type: `@n8n/n8n-nodes-langchain.lmChatAnthropic`
    - Credentials: Anthropic API
    - Model: `claude-sonnet-4-5-20250929`
    - Connect it to **AI Agent** through the AI language model port.

20. **Add a Liveblocks node** named **Create a comment1**.
    - Type: `CUSTOM.liveblocks`
    - Resource: `comment`
    - Configure it for comment creation
    - Set:
      - `roomId = {{ $('Get a comment').first().json.roomId }}`
      - `threadId = {{ $('Get a comment').first().json.threadId }}`
      - `createComment_userId = __AI_AGENT`
      - `createComment_bodyMode = plainText`
      - `createComment_bodyText = {{ $json.output }}`
    - Connect **AI Agent → Create a comment1**

## Credential setup

21. **Create Liveblocks API credentials** for:
    - **Get a comment**
    - **Get attachment1**
    - **Create a comment**
    - **Create a comment1**

22. **Create Liveblocks Webhook Signing Secret credentials** for:
    - **Liveblocks Trigger**

23. **Create Anthropic API credentials** for:
    - **Analyze document1**
    - **Analyze image**
    - **Anthropic Chat Model**
    - **Anthropic Chat Model1**

## Optional but recommended hardening

24. Add error handling or guardrails for:
    - comments with malformed attachment arrays
    - inaccessible attachment URLs
    - unsupported file types
    - webhook retries causing duplicate AI comments

25. Consider improving the final agent prompt to:
    - mention both PDFs and images, not only images
    - use consistent quote delimiters
    - instruct the model to mention unsupported files when relevant

26. Activate the workflow and test with:
    - a plain text comment mentioning the AI
    - a comment mentioning the AI with a PDF
    - a comment mentioning the AI with an image
    - a comment mentioning the AI with an unsupported file type

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Liveblocks Comments product page | https://liveblocks.io/comments |
| Liveblocks webhooks documentation | https://liveblocks.io/docs/platform/webhooks |
| Next.js comments example to test the workflow | https://liveblocks.io/examples/comments/nextjs-comments |
| Liveblocks dashboard for webhook and project setup | https://liveblocks.io/dashboard |
| The AI assistant user ID used throughout the workflow is `__AI_AGENT` | Global workflow assumption |
| The workflow requires a Comments app installed in Liveblocks and webhook setup in the project dashboard | Environment prerequisite |
| The setup note says to uncomment the AI assistant user in `database.ts` inside the Next.js example | Demo app preparation |
| The setup note says to expose the n8n webhook publicly with tools like `localtunnel` or `ngrok` | Local development requirement |
| Example localtunnel command: `npx localtunnel --port 5678 --subdomain your-name-here` | Local webhook exposure |
| Example public URL pattern for webhook exposure: `https://honest-months-fix.loca.lt/webhook/.../webhook` | Replace local n8n URL with public tunnel URL |
| The sticky note mentions inserting the secret key into nodes “Get a comment”, “Get a thread”, and “Create a comment”, but this workflow does not actually include a “Get a thread” node | Documentation inconsistency in canvas note |
| The workflow title in the canvas refers to analyzing PDF and image attachments with Anthropic and responding in real time in the same thread | Overall workflow context |