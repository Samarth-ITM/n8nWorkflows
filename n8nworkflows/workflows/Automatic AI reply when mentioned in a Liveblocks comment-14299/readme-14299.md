Automatic AI reply when mentioned in a Liveblocks comment

https://n8nworkflows.xyz/workflows/automatic-ai-reply-when-mentioned-in-a-liveblocks-comment-14299


# Automatic AI reply when mentioned in a Liveblocks comment

# 1. Workflow Overview

This workflow automatically posts an AI-generated reply in a Liveblocks comment thread when the AI assistant is mentioned in a newly created comment.

Its main use case is collaborative applications using **Liveblocks Comments**, where users can mention an AI participant such as `@AI Assistant`. When a new comment is created, the workflow checks whether the AI user ID (`__AI_AGENT`) was mentioned. If yes, it fetches the full thread, sends the thread context to an Anthropic model through an n8n AI Agent node, and posts the generated reply back into the same thread as a new comment.

## 1.1 Event Reception

The workflow starts from a **Liveblocks Trigger** configured for the `commentCreated` event. This receives webhook payloads whenever a new comment is added in Liveblocks.

## 1.2 Comment Inspection

After receiving the event, the workflow fetches the full created comment and checks whether the AI user ID appears in the comment’s `mentionedUserIds` array.

## 1.3 Thread Context Retrieval and AI Generation

If the AI is mentioned, the workflow retrieves the entire thread so the language model can understand the discussion context. The thread comments are then passed into an **AI Agent** node connected to an **Anthropic Chat Model**.

## 1.4 Automated Reply Publication

The generated response is posted back to the same Liveblocks thread as a plain-text comment authored by the synthetic user ID `__AI_AGENT`.

---

# 2. Block-by-Block Analysis

## Block 1: Event Reception

### Overview
This block receives webhook events from Liveblocks whenever a comment is created. It is the only entry point in the workflow and drives all downstream logic.

### Nodes Involved
- Liveblocks Trigger

### Node Details

#### Liveblocks Trigger
- **Type and technical role:** `CUSTOM.liveblocksTrigger`  
  Trigger node that listens for Liveblocks webhook events.
- **Configuration choices:**
  - Configured to listen only to `commentCreated`
  - Uses a **Liveblocks Webhook Signing Secret** credential to validate webhook authenticity
- **Key expressions or variables used:**
  - No expressions in the node itself
  - Downstream nodes use the incoming payload fields under `$json.event.data`
- **Input and output connections:**
  - Input: none, as it is a trigger
  - Output: connected to **Get a comment**
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Depends on the custom Liveblocks trigger node being installed and available in the n8n instance
- **Edge cases or potential failure types:**
  - Invalid webhook signature
  - Webhook endpoint not publicly reachable
  - Liveblocks webhook misconfigured for the wrong event type
  - Trigger URL mismatch between local and public endpoint
- **Sub-workflow reference:**
  - None

---

## Block 2: Comment Inspection

### Overview
This block retrieves the newly created comment and determines whether the AI assistant was mentioned. It prevents unnecessary AI calls when the AI user is not tagged.

### Nodes Involved
- Get a comment
- If

### Node Details

#### Get a comment
- **Type and technical role:** `CUSTOM.liveblocks`  
  Liveblocks API action node used here to fetch the full comment object.
- **Configuration choices:**
  - `resource`: `comment`
  - `operation`: `getComment`
  - `roomId`: taken from the trigger payload
  - `threadId`: taken from the trigger payload
  - `commentId`: taken from the trigger payload
- **Key expressions or variables used:**
  - `={{ $json.event.data.roomId }}`
  - `={{ $json.event.data.threadId }}`
  - `={{ $json.event.data.commentId }}`
- **Input and output connections:**
  - Input: **Liveblocks Trigger**
  - Output: **If**
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires a **Liveblocks API** credential with access to the target project
- **Edge cases or potential failure types:**
  - Invalid or expired Liveblocks API credentials
  - Comment deleted before retrieval
  - Trigger payload missing expected identifiers
  - Permission mismatch between webhook project and API credential project
- **Sub-workflow reference:**
  - None

#### If
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional node used to determine whether the AI assistant ID is present in the comment mentions.
- **Configuration choices:**
  - Uses an **array contains** condition
  - Checks whether `mentionedUserIds` contains `__AI_AGENT`
  - Strict validation enabled by the node condition settings
- **Key expressions or variables used:**
  - `={{ $json.mentionedUserIds }}`
  - Comparison target: `__AI_AGENT`
- **Input and output connections:**
  - Input: **Get a comment**
  - True output: **Get a thread**
  - False output: no connection; execution stops
- **Version-specific requirements:**
  - `typeVersion: 2.3`
  - Uses condition format with version 3 condition settings
- **Edge cases or potential failure types:**
  - `mentionedUserIds` absent or null
  - Data type mismatch if `mentionedUserIds` is not returned as an array
  - AI mention format in the frontend not matching `__AI_AGENT`
- **Sub-workflow reference:**
  - None

---

## Block 3: Thread Context Retrieval and AI Generation

### Overview
If the AI assistant was mentioned, this block fetches the entire thread and asks an AI model to generate a reply based on all comments in context.

### Nodes Involved
- Get a thread
- AI Agent
- Anthropic Chat Model

### Node Details

#### Get a thread
- **Type and technical role:** `CUSTOM.liveblocks`  
  Liveblocks API action node used to retrieve the entire thread object and its comments.
- **Configuration choices:**
  - `resource`: `thread`
  - `operation`: `getThread`
  - `roomId`: from the previous node output
  - `threadId`: from the previous node output
- **Key expressions or variables used:**
  - `={{ $json.roomId }}`
  - `={{ $json.threadId }}`
- **Input and output connections:**
  - Input: **If** true branch
  - Output: **AI Agent**
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires the same Liveblocks API credential as other Liveblocks API nodes
- **Edge cases or potential failure types:**
  - Thread no longer exists
  - Permission or credential issues
  - Large thread payloads causing longer execution or prompt size pressure downstream
- **Sub-workflow reference:**
  - None

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI orchestration node that builds the prompt and requests a completion from the connected language model.
- **Configuration choices:**
  - `promptType`: `define`
  - Prompt instructs the agent that:
    - it is replying to comments in a thread
    - it has been mentioned in the latest comment
    - it must use the entire thread context
    - it must return the reply as a string
  - No extra tools or options are configured
- **Key expressions or variables used:**
  - Includes the full thread comments using:
    - `{{ JSON.stringify($json.comments, null, 2) }}`
- **Input and output connections:**
  - Main input: **Get a thread**
  - AI language model input: **Anthropic Chat Model**
  - Main output: **Create a comment**
- **Version-specific requirements:**
  - `typeVersion: 3.1`
  - Requires n8n’s LangChain/AI nodes support in the instance version
- **Edge cases or potential failure types:**
  - Model output may be empty or malformed
  - Token/context limit issues if thread history grows too large
  - Prompt injection in user comments affecting model behavior
  - Unexpected thread schema if `comments` is absent
- **Sub-workflow reference:**
  - None

#### Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Provides the language model used by the AI Agent.
- **Configuration choices:**
  - Model selected: `claude-sonnet-4-6`
  - No additional model options configured
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input: none on main channel
  - AI output connection: connected to **AI Agent** via `ai_languageModel`
- **Version-specific requirements:**
  - `typeVersion: 1.3`
  - Requires valid **Anthropic API** credentials
  - Model availability depends on the Anthropic account and n8n node version
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model unavailability or account entitlement mismatch
  - API rate limiting
  - Timeout or upstream service interruption
- **Sub-workflow reference:**
  - None

---

## Block 4: Automated Reply Publication

### Overview
This block posts the generated AI response back into the same Liveblocks thread as a new plain-text comment under the AI assistant identity.

### Nodes Involved
- Create a comment

### Node Details

#### Create a comment
- **Type and technical role:** `CUSTOM.liveblocks`  
  Liveblocks API action node used to create a new comment in an existing thread.
- **Configuration choices:**
  - `resource`: `comment`
  - Uses the original `roomId` and `threadId`
  - Sets `createComment_userId` to `__AI_AGENT`
  - Uses `plainText` mode for the body
  - Comment body is the AI output string
- **Key expressions or variables used:**
  - `={{ $('Get a comment').item.json.roomId }}`
  - `={{ $('Get a comment').item.json.threadId }}`
  - `={{ $json.output }}`
- **Input and output connections:**
  - Input: **AI Agent**
  - Output: none
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires valid Liveblocks API credentials
- **Edge cases or potential failure types:**
  - Empty `output` from AI Agent
  - API validation failure if text exceeds platform limits
  - User ID `__AI_AGENT` not recognized by the application’s user mapping
  - Potential duplicate replies if webhook retries occur and no deduplication exists
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Liveblocks Trigger | `CUSTOM.liveblocksTrigger` | Receives Liveblocks `commentCreated` webhook events |  | Get a comment | ## Automatic AI reply when mentioned in a Liveblocks comment<br>This example uses [Liveblocks Comments](https://liveblocks.io/comments), collaborative commenting components for React. When an AI assistant is mentioned in a thread (e.g. "@AI Assistant"), it will automatically leave a response.<br>Using [webhooks](https://liveblocks.io/docs/platform/webhooks), this workflow is triggered when a comment is created in a thread. If the agent's ID (`"__AI_AGENT"`) was mentioned, an AI agent is given the entire thread context, and asked to create a response. This response is then added, and users will see it appear in their apps in real time.<br>### Setup<br>- [ ] Download the [Next.js comments example](https://liveblocks.io/examples/comments/nextjs-comments), and run it with a secret key.<br>- [ ] Find `database.ts` inside the example and uncomment the AI assistant user.<br>- [ ] Insert the secret key from the project into n8n nodes: "Get a comment", "Get a thread", "Create a comment".<br>- [ ] Go to the [Liveblocks dashboard](https://liveblocks.io/dashboard), open your project and go to "Webhooks". Create a new webhook in your project using a placeholder URL, and selecting "commentCreated" events.<br>- [ ] Copy your webhook secret from this page and paste it into the "Liveblocks Trigger" node.<br>- [ ] Expose the webhook URL from the trigger, for example with `localtunnel` or `ngrok`. Copy the production URL from the "Liveblocks Trigger" and replace `localhost:5678` with the new URL.<br>- [ ] Your workflow is now set up! Tag @AI Assistant in the application to trigger it.<br>#### Localtunnel<br>`npx localtunnel --port 5678 --subdomain your-name-here`<br>This creates a URL like:<br>`https://honest-months-fix.loca.lt`<br>The URL you need for the dashboard looks like this:<br>`https://honest-months-fix.loca.lt/webhook/9cc66974-aaaf-4720-b557-1267105ca78b/webhook`<br>## Comment created<br>When a comment is created in a room. |
| Get a comment | `CUSTOM.liveblocks` | Fetches the newly created comment | Liveblocks Trigger | If | ## Check if AI mentioned<br>Check if a comment mentions the AI agent. Their ID is `"__AI_AGENT"` in this demo. |
| If | `n8n-nodes-base.if` | Checks whether `mentionedUserIds` contains `__AI_AGENT` | Get a comment | Get a thread | ## Check if AI mentioned<br>Check if a comment mentions the AI agent. Their ID is `"__AI_AGENT"` in this demo. |
| Get a thread | `CUSTOM.liveblocks` | Fetches the complete thread for context | If | AI Agent | ## Fetch thread and generate<br>If `true`, fetch the entire comment thread, pass it to AI so it understands the full context, and ask it to write up a response. |
| AI Agent | `@n8n/n8n-nodes-langchain.agent` | Builds the prompt and generates the reply | Get a thread; Anthropic Chat Model | Create a comment | ## Fetch thread and generate<br>If `true`, fetch the entire comment thread, pass it to AI so it understands the full context, and ask it to write up a response. |
| Anthropic Chat Model | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Supplies the Anthropic model used by the AI Agent |  | AI Agent | ## Fetch thread and generate<br>If `true`, fetch the entire comment thread, pass it to AI so it understands the full context, and ask it to write up a response. |
| Create a comment | `CUSTOM.liveblocks` | Posts the AI reply back into the thread | AI Agent |  | ## Add response to thread<br>The response is then added as a comment in the same thread. The user ID is set to `"__AI_AGENT"`. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  |  |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  |  |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  |  |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  |  |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation/comment node |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Automatic AI reply when mentioned in a Liveblocks comment**.

2. **Add the trigger node**
   - Create a **Liveblocks Trigger** node.
   - Configure it to listen for:
     - `commentCreated`
   - Add credentials:
     - **Liveblocks Webhook Signing Secret**
   - Save the node and note the webhook URL generated by n8n.

3. **Prepare Liveblocks on the platform side**
   - In the **Liveblocks dashboard**, open the target project.
   - Go to **Webhooks**.
   - Create a webhook for the `commentCreated` event.
   - Paste the n8n production webhook URL.
   - Copy the webhook signing secret into the trigger credential in n8n.

4. **Expose the webhook if developing locally**
   - If n8n runs locally, expose it via a public tunnel such as `localtunnel` or `ngrok`.
   - Example:
     - `npx localtunnel --port 5678 --subdomain your-name-here`
   - Replace the local host in the n8n webhook URL with the public tunnel domain.
   - Register that public URL in the Liveblocks dashboard webhook.

5. **Add a Liveblocks API node to retrieve the created comment**
   - Create a **Liveblocks** node and name it **Get a comment**.
   - Configure:
     - `resource`: **comment**
     - `operation`: **getComment**
     - `roomId`: `{{ $json.event.data.roomId }}`
     - `threadId`: `{{ $json.event.data.threadId }}`
     - `commentId`: `{{ $json.event.data.commentId }}`
   - Add a **Liveblocks API** credential using your Liveblocks project secret/API access.
   - Connect:
     - **Liveblocks Trigger** → **Get a comment**

6. **Add the conditional check**
   - Create an **If** node.
   - Configure a condition:
     - Left value: `{{ $json.mentionedUserIds }}`
     - Operator: **array contains**
     - Right value: `__AI_AGENT`
   - Keep strict validation enabled if available.
   - Connect:
     - **Get a comment** → **If**

7. **Add a node to fetch the entire thread**
   - Create another **Liveblocks** node and name it **Get a thread**.
   - Configure:
     - `resource`: **thread**
     - `operation`: **getThread**
     - `roomId`: `{{ $json.roomId }}`
     - `threadId`: `{{ $json.threadId }}`
   - Use the same **Liveblocks API** credential.
   - Connect:
     - **If** true output → **Get a thread**
   - Leave the false output unconnected so execution ends when the AI is not mentioned.

8. **Add the AI Agent node**
   - Create an **AI Agent** node.
   - Set:
     - `promptType`: **define**
   - In the prompt text, use:
     - You are an AI assistant that can reply to comments in a thread. You have been mentioned in the latest comment and must reply.
       
       Here is the entire thread:
       
       ```
       {{ JSON.stringify($json.comments, null, 2) }}
       ```
       
       You were mentioned in the last comment. Write your reply as a string.
   - Connect:
     - **Get a thread** → **AI Agent**

9. **Add the language model**
   - Create an **Anthropic Chat Model** node.
   - Select the model:
     - **Claude Sonnet 4.6** (`claude-sonnet-4-6`)
   - Add **Anthropic API** credentials.
   - Connect the model to the AI Agent using the **AI language model** connection:
     - **Anthropic Chat Model** → **AI Agent**

10. **Add the node that posts the AI reply**
    - Create a **Liveblocks** node and name it **Create a comment**.
    - Configure:
      - `resource`: **comment**
      - comment creation operation
      - `roomId`: `{{ $('Get a comment').item.json.roomId }}`
      - `threadId`: `{{ $('Get a comment').item.json.threadId }}`
      - `createComment_userId`: `__AI_AGENT`
      - `createComment_bodyMode`: **plainText**
      - `createComment_bodyText`: `{{ $json.output }}`
    - Use the same **Liveblocks API** credential.
    - Connect:
      - **AI Agent** → **Create a comment**

11. **Set up the application-side AI user**
    - Ensure the frontend/demo app recognizes a user with ID `__AI_AGENT`.
    - In the referenced Liveblocks demo, this is done by uncommenting the AI assistant user in `database.ts`.
    - If your application uses another identifier, update:
      - the **If** node condition
      - the **Create a comment** user ID
      - your frontend mention system

12. **Activate the workflow**
    - Turn the workflow on.
    - Create a comment in a Liveblocks thread mentioning the AI user.
    - Verify the webhook fires, the thread is fetched, and the AI reply appears in the same thread.

13. **Recommended safeguards when rebuilding**
    - Add deduplication if Liveblocks may retry webhook deliveries.
    - Consider truncating or summarizing long threads before sending them to the model.
    - Optionally validate that `$json.output` is non-empty before posting the comment.
    - Optionally add error handling branches for API failures.

## Credential Configuration Summary

### Liveblocks Webhook Signing Secret
- Used by: **Liveblocks Trigger**
- Purpose: validates incoming webhook signatures from Liveblocks

### Liveblocks API
- Used by:
  - **Get a comment**
  - **Get a thread**
  - **Create a comment**
- Purpose: reads and writes comments/threads in the Liveblocks project

### Anthropic API
- Used by: **Anthropic Chat Model**
- Purpose: generates the reply text through Claude Sonnet 4.6

## Input/Output Expectations

### Trigger payload expectation
The workflow expects the webhook payload to contain:
- `event.data.roomId`
- `event.data.threadId`
- `event.data.commentId`

### Comment object expectation
The **Get a comment** node should return:
- `mentionedUserIds`
- `roomId`
- `threadId`

### Thread object expectation
The **Get a thread** node should return:
- `comments` array containing thread history

### AI output expectation
The **AI Agent** node is expected to expose:
- `output` as the generated reply string

### No sub-workflows
This workflow does **not** invoke any sub-workflow node and has only one entry point.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Liveblocks Comments product page | https://liveblocks.io/comments |
| Liveblocks webhooks documentation | https://liveblocks.io/docs/platform/webhooks |
| Liveblocks dashboard | https://liveblocks.io/dashboard |
| Next.js comments example referenced by the workflow notes | https://liveblocks.io/examples/comments/nextjs-comments |
| Localtunnel command shown in the workflow notes: `npx localtunnel --port 5678 --subdomain your-name-here` | Useful for exposing a local n8n webhook publicly |
| Example public webhook URL pattern from the workflow note: `https://honest-months-fix.loca.lt/webhook/9cc66974-aaaf-4720-b557-1267105ca78b/webhook` | Replace host with your public tunnel domain and keep the n8n webhook path |