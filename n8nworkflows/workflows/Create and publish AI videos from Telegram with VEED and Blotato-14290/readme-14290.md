Create and publish AI videos from Telegram with VEED and Blotato

https://n8nworkflows.xyz/workflows/create-and-publish-ai-videos-from-telegram-with-veed-and-blotato-14290


# Create and publish AI videos from Telegram with VEED and Blotato

## 1. Workflow Overview

This workflow creates a conversational AI video assistant on Telegram. A user chats with a Telegram bot, the bot uses an OpenAI-powered agent plus VEED MCP tools to help the user generate an AI talking-head video, and once a final video URL is detected, the workflow asks whether the video should also be published to social media through Blotato.

### 1.1 Telegram Input Reception
The workflow starts when a Telegram user sends a message to the bot. That message becomes the prompt for the AI agent.

### 1.2 AI Conversation, Memory, and VEED Tool Use
An AI agent receives the Telegram text, uses an OpenAI chat model, stores short conversation history per Telegram chat, and calls VEED MCP tools to list avatars, list voices, preview a video, create it, and poll generation status.

### 1.3 User Response Delivery and Video URL Detection
The agent’s response is sent back to Telegram. Then a code node parses the response to look for the exact `VIDEO_READY:` marker followed by a VEED URL.

### 1.4 Conditional Approval Request
If a final video URL is found, the workflow sends a Telegram approval prompt with Approve/Reject buttons and waits up to 24 hours for a response.

### 1.5 Publishing Decision
If the user approves, the workflow prepares a caption, uploads the video URL to Blotato, and fans out publication requests to nine social platforms in parallel. If the user rejects, a Telegram confirmation message is sent instead.

### 1.6 Parallel Social Posting and Final Confirmation
Each Blotato platform node posts the uploaded media to its configured destination. A merge node collects completion from all configured branches and then sends a final Telegram summary.

---

## 2. Block-by-Block Analysis

## 2.1 Telegram Input Reception

**Overview:**  
This block receives incoming Telegram messages and passes them into the AI conversation pipeline. It is the main entry point of the workflow.

**Nodes Involved:**  
- `📩 Telegram Trigger`

### Node Details

#### `📩 Telegram Trigger`
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`  
  Trigger node that listens for incoming Telegram updates.
- **Configuration choices:**  
  - Subscribed to `message` updates only.
  - No additional filters are configured.
- **Key expressions or variables used:**  
  None in parameters, but downstream nodes reference:
  - `{{$json.message.text}}`
  - `{{$json.message.chat.id}}`
- **Input and output connections:**  
  - Input: none, entry point
  - Output: `🤖 AI Video Agent`
- **Version-specific requirements:**  
  - Type version `1.2`
  - Requires Telegram bot credentials and a valid webhook configuration handled by n8n.
- **Edge cases or potential failure types:**  
  - Bot token invalid or revoked
  - Telegram webhook registration issues
  - User sends non-text messages; downstream agent expects `message.text`
  - Private/group chat permissions may affect delivery
- **Sub-workflow reference:**  
  None

---

## 2.2 AI Conversation, Memory, and VEED Tool Orchestration

**Overview:**  
This is the core intelligence block. The AI agent interprets user requests, maintains recent context per Telegram chat, and invokes VEED MCP tools to browse assets and generate videos.

**Nodes Involved:**  
- `🤖 AI Video Agent`
- `🧠 OpenAI Chat Model`
- `💾 Conversation Memory`
- `🎬 VEED MCP Tools`

### Node Details

#### `🤖 AI Video Agent`
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Agent node that orchestrates the conversation and decides when to call tools.
- **Configuration choices:**  
  - Prompt source: `={{ $json.message.text }}`
  - `maxIterations`: `15`
  - Custom system prompt defines:
    - assistant behavior
    - VEED tool usage rules
    - required conversation flow
    - final `VIDEO_READY:` output convention
    - Telegram formatting restrictions
- **Key expressions or variables used:**  
  - Input text: `{{$json.message.text}}`
- **Input and output connections:**  
  - Main input: `📩 Telegram Trigger`
  - AI language model input: `🧠 OpenAI Chat Model`
  - AI memory input: `💾 Conversation Memory`
  - AI tool input: `🎬 VEED MCP Tools`
  - Main output: `📤 Send Agent Response`
- **Version-specific requirements:**  
  - Type version `3.1`
  - Depends on LangChain-capable n8n version with AI node support.
- **Edge cases or potential failure types:**  
  - User sends empty/non-text message
  - LLM may fail to call required VEED tools correctly
  - Tool-call loops may exceed max iterations
  - The agent may omit the `VIDEO_READY:` marker, which prevents downstream publishing
  - Responses may be too verbose for Telegram
- **Sub-workflow reference:**  
  None

#### `🧠 OpenAI Chat Model`
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the LLM used by the agent.
- **Configuration choices:**  
  - Model: `gpt-5-nano`
  - No special model options configured
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output via AI language model port to `🤖 AI Video Agent`
- **Version-specific requirements:**  
  - Type version `1.2`
  - Requires OpenAI credentials
  - Model availability depends on the connected OpenAI account and n8n node support.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Quota/rate-limit errors
  - Model access not enabled in account
  - Upstream tool outputs may exceed context limits if conversation gets long
- **Sub-workflow reference:**  
  None

#### `💾 Conversation Memory`
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Stores rolling chat history so the AI can maintain state across messages.
- **Configuration choices:**  
  - Session ID type: custom key
  - Session key: `={{ $('📩 Telegram Trigger').item.json.message.chat.id }}`
  - Context window length: `20`
- **Key expressions or variables used:**  
  - Telegram chat ID as memory session key
- **Input and output connections:**  
  - Output via AI memory port to `🤖 AI Video Agent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - If the trigger payload lacks `message.chat.id`, memory sessioning breaks
  - Windowed memory only retains the last 20 interactions, so earlier selections may be forgotten
- **Sub-workflow reference:**  
  None

#### `🎬 VEED MCP Tools`
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes VEED MCP tools to the AI agent over an MCP endpoint.
- **Configuration choices:**  
  - Endpoint URL: `https://www.veed.io/api/v1/mcp`
  - Authentication: MCP OAuth2 API credentials
  - No additional options configured
- **Key expressions or variables used:**  
  None in node parameters
- **Input and output connections:**  
  - Output via AI tool port to `🤖 AI Video Agent`
- **Version-specific requirements:**  
  - Type version `1.2`
  - Requires VEED MCP OAuth2 credentials configured in n8n
- **Edge cases or potential failure types:**  
  - OAuth2 auth failure or expired token
  - MCP endpoint unavailable
  - Tool schema mismatch or remote tool errors
  - VEED credit exhaustion
  - Slow generation status polling
- **Sub-workflow reference:**  
  None

---

## 2.3 User Response Delivery and Video URL Detection

**Overview:**  
This block sends the AI-generated reply to Telegram, then scans it for the special `VIDEO_READY:` marker. That marker is the handoff between conversational generation and optional publication.

**Nodes Involved:**  
- `📤 Send Agent Response`
- `📋 Extract Video URL`
- `✅ Has Video URL?`

### Node Details

#### `📤 Send Agent Response`
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the agent’s plain response back to the Telegram user.
- **Configuration choices:**  
  - Operation: send message
  - Text: `={{ $json.output }}`
  - Chat ID: `={{ $('📩 Telegram Trigger').item.json.message.chat.id }}`
  - `onError`: continue regular output
- **Key expressions or variables used:**  
  - `{{$json.output}}`
  - `{{ $('📩 Telegram Trigger').item.json.message.chat.id }}`
- **Input and output connections:**  
  - Input: `🤖 AI Video Agent`
  - Output: `📋 Extract Video URL`
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - Telegram formatting issues if LLM outputs malformed text
  - Message length limits
  - `continueRegularOutput` means downstream parsing still happens even if Telegram delivery fails
- **Sub-workflow reference:**  
  None

#### `📋 Extract Video URL`
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript parser that extracts the final VEED video URL from the agent output.
- **Configuration choices:**  
  - Reads agent output from `$('🤖 AI Video Agent').item.json.output`
  - Regex used: `/VIDEO_READY:\s*(https?:\/\/[^\s)>\]]+)/i`
  - Returns:
    - `video_url`
    - `has_video`
    - `agent_output`
    - `chat_id`
- **Key expressions or variables used:**  
  - `$('🤖 AI Video Agent').item.json.output`
  - `$('📩 Telegram Trigger').item.json.message.chat.id`
- **Input and output connections:**  
  - Input: `📤 Send Agent Response`
  - Output: `✅ Has Video URL?`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - If the LLM does not include `VIDEO_READY:` exactly, no URL is captured
  - Regex may miss malformed or wrapped URLs
  - If multiple URLs appear, only the first matching URL after the marker is used
- **Sub-workflow reference:**  
  None

#### `✅ Has Video URL?`
- **Type and technical role:** `n8n-nodes-base.if`  
  Boolean gate that checks whether a final video URL was found.
- **Configuration choices:**  
  - Strict boolean validation
  - Condition: `={{ $json.has_video }}` is `true`
- **Key expressions or variables used:**  
  - `{{$json.has_video}}`
- **Input and output connections:**  
  - Input: `📋 Extract Video URL`
  - True output: intended to `📩 Publish to Social Media?`
  - False output: no branch configured
- **Version-specific requirements:**  
  - Type version `2.2`
- **Edge cases or potential failure types:**  
  - If `has_video` is not strictly boolean, condition may fail due to strict validation
  - If no URL is present, the workflow simply stops after sending the AI response
- **Sub-workflow reference:**  
  None

---

## 2.4 Conditional Approval Request

**Overview:**  
When a final video URL exists, the workflow asks the user whether to publish it. The Telegram node pauses execution and waits up to 24 hours for an approval or rejection.

**Nodes Involved:**  
- `📩 Publish to Social Media?`
- `✅ Approved?`

### Node Details

#### `📩 Publish to Social Media?`
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends an interactive Telegram approval message and waits for the response.
- **Configuration choices:**  
  - Operation: `sendAndWait`
  - Chat ID: `={{ $json.chat_id }}`
  - Message text includes video URL
  - Approval type: `double` (Approve / Reject style interaction)
  - Limit wait time: 24 hours
- **Key expressions or variables used:**  
  - `{{$json.chat_id}}`
  - `{{ "Your video is ready!\n\n" + $json.video_url + "\n\nWould you like to publish it to social media via Blotato?" }}`
- **Input and output connections:**  
  - Input: `✅ Has Video URL?`
  - Output: `✅ Approved?`
- **Version-specific requirements:**  
  - Type version `1.2`
  - Requires an n8n setup that supports waiting/resume behavior for Telegram send-and-wait operations.
- **Edge cases or potential failure types:**  
  - User does not respond within 24 hours
  - Resume webhook/wait state issues
  - Telegram interaction permissions or message delivery issues
- **Sub-workflow reference:**  
  None

#### `✅ Approved?`
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the approval response returned `approved = true`.
- **Configuration choices:**  
  - Strict boolean validation
  - Condition: `={{ $json.data.approved }}` is `true`
- **Key expressions or variables used:**  
  - `{{$json.data.approved}}`
- **Input and output connections:**  
  - Input: `📩 Publish to Social Media?`
  - Intended true output: `⚙️ Set Caption`
  - Intended false output: `📤 Send Skipped Message`
- **Version-specific requirements:**  
  - Type version `2.2`
- **Edge cases or potential failure types:**  
  - If Telegram wait response payload shape changes, `data.approved` may be missing
  - Timeout/no response may not produce the expected approval object
- **Sub-workflow reference:**  
  None

**Important implementation note:**  
The JSON wiring shows both `⚙️ Set Caption` and `📤 Send Skipped Message` attached under the same `main` output array for `✅ Approved?`. In intended n8n behavior, an IF node usually splits true and false into separate outputs. Functionally, this workflow is clearly designed so:
- `true` → `⚙️ Set Caption`
- `false` → `📤 Send Skipped Message`

When rebuilding, configure the true and false branches explicitly.

---

## 2.5 Publishing Preparation and Blotato Upload

**Overview:**  
If the user approves publishing, this block derives a social caption from the AI output and uploads the video to Blotato so the returned media URL can be reused across all platform-specific post nodes.

**Nodes Involved:**  
- `⚙️ Set Caption`
- `📤 Upload Video to Blotato`

### Node Details

#### `⚙️ Set Caption`
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates normalized fields used by all social publishing nodes.
- **Configuration choices:**  
  - Sets `caption` to first 200 characters of the original agent output
  - Sets `video_url` from the extracted VEED URL
- **Key expressions or variables used:**  
  - `{{ $('📋 Extract Video URL').item.json.agent_output.substring(0, 200) }}`
  - `{{ $('📋 Extract Video URL').item.json.video_url }}`
- **Input and output connections:**  
  - Input: intended true branch of `✅ Approved?`
  - Output: `📤 Upload Video to Blotato`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - Caption may include the `VIDEO_READY:` marker if the output is not cleaned
  - Caption truncation may cut off words or formatting
  - If `agent_output` is missing, expression fails
- **Sub-workflow reference:**  
  None

#### `📤 Upload Video to Blotato`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Uploads the external media URL into Blotato as a media resource before platform posting.
- **Configuration choices:**  
  - Resource: `media`
  - Media URL: `={{ $json.video_url }}`
- **Key expressions or variables used:**  
  - `{{$json.video_url}}`
- **Input and output connections:**  
  - Input: `⚙️ Set Caption`
  - Outputs in parallel to:
    - `TikTok`
    - `YouTube`
    - `Instagram`
    - `LinkedIn`
    - `Facebook`
    - `Twitter (X)`
    - `Threads`
    - `Bluesky`
    - `Pinterest`
- **Version-specific requirements:**  
  - Type version `2`
  - Requires Blotato community node installation
  - Requires Blotato API credentials
- **Edge cases or potential failure types:**  
  - Community node not installed
  - Invalid API key
  - Unsupported or inaccessible media URL
  - Media fetch/upload timeout
- **Sub-workflow reference:**  
  None

---

## 2.6 Parallel Social Media Publishing

**Overview:**  
This block posts the uploaded media to nine social platforms using Blotato. Each platform node receives the uploaded media URL returned from the Blotato media upload step.

**Nodes Involved:**  
- `TikTok`
- `YouTube`
- `Instagram`
- `LinkedIn`
- `Facebook`
- `Twitter (X)`
- `Threads`
- `Bluesky`
- `Pinterest`
- `Merge`

### Node Details

#### `TikTok`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes a post to TikTok via Blotato.
- **Configuration choices:**  
  - Platform: `tiktok`
  - Account ID selected from credential-backed list
  - Post text: caption from `⚙️ Set Caption`
  - Media URLs: uploaded media URL from previous node (`$json.url`)
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 0
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Missing account ID
  - TikTok account not connected in Blotato
  - Platform restrictions on captions or video format
- **Sub-workflow reference:**  
  None

#### `YouTube`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes a YouTube video via Blotato.
- **Configuration choices:**  
  - Platform: `youtube`
  - Account ID from list
  - Post text uses caption
  - Media URL uses uploaded media
  - Title: first 100 chars of caption
  - Privacy: `private`
  - Notify subscribers: `false`
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
  - `{{ $('⚙️ Set Caption').item.json.caption.substring(0, 100) }}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 1
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Missing title/account configuration
  - YouTube metadata limits
  - Upload or transcoding delays on platform side
- **Sub-workflow reference:**  
  None

#### `Instagram`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes to Instagram via Blotato.
- **Configuration choices:**  
  - Account ID from list
  - Post text from caption
  - Media URL from uploaded media
  - Platform parameter not explicitly shown, likely defaults to Instagram in this node setup
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 2
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Missing/incorrect account selection
  - Instagram media format constraints
  - If platform default behavior differs between node versions, explicitly setting platform may be safer when rebuilding
- **Sub-workflow reference:**  
  None

#### `LinkedIn`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes to LinkedIn via Blotato.
- **Configuration choices:**  
  - Platform: `linkedin`
  - Account ID from list
  - Caption and media URL mapped in the same pattern
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 3
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Account not connected
  - Business/profile posting restrictions
- **Sub-workflow reference:**  
  None

#### `Facebook`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes to Facebook via Blotato.
- **Configuration choices:**  
  - Platform: `facebook`
  - Account ID from list
  - Facebook Page ID from list
  - Caption and media URL mapped
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 4
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Missing Page ID
  - Account connected but page permissions missing
- **Sub-workflow reference:**  
  None

#### `Twitter (X)`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes to X/Twitter via Blotato.
- **Configuration choices:**  
  - Platform: `twitter`
  - Account ID from list
  - Caption and media URL mapped
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 5
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Character limits or media policy restrictions
  - Account authorization issues
- **Sub-workflow reference:**  
  None

#### `Threads`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes to Threads via Blotato.
- **Configuration choices:**  
  - Platform: `threads`
  - Account ID from list
  - Caption and media URL mapped
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 6
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Account connection or platform-specific media restrictions
- **Sub-workflow reference:**  
  None

#### `Bluesky`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes to Bluesky via Blotato.
- **Configuration choices:**  
  - Platform: `bluesky`
  - Account ID from list
  - Caption and media URL mapped
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 7
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Account connection issues
  - Platform API limitations
- **Sub-workflow reference:**  
  None

#### `Pinterest`
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Publishes to Pinterest via Blotato.
- **Configuration choices:**  
  - Platform: `pinterest`
  - Account ID from list
  - Pinterest Board ID entered as ID mode
  - Caption and media URL mapped
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Set Caption').item.json.caption }}`
  - `{{$json.url}}`
- **Input and output connections:**  
  - Input: `📤 Upload Video to Blotato`
  - Output: `Merge` input 8
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Missing board ID
  - Board permissions or unsupported media format
- **Sub-workflow reference:**  
  None

#### `Merge`
- **Type and technical role:** `n8n-nodes-base.merge`  
  Collects all platform branches before sending a final success message.
- **Configuration choices:**  
  - Mode: `chooseBranch`
  - Number of inputs: `9`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Inputs from all nine platform nodes
  - Output: `📤 Send Publish Summary`
- **Version-specific requirements:**  
  - Type version `3.2`
- **Edge cases or potential failure types:**  
  - In `chooseBranch` mode, behavior is not equivalent to “wait for all branches”; depending on execution semantics, this may continue when one branch succeeds rather than after all nine finish
  - If the intent is true aggregation after all posts complete, a different merge configuration may be preferable
- **Sub-workflow reference:**  
  None

---

## 2.7 Final User Notifications

**Overview:**  
This block sends either a “publishing skipped” message or a “publishing complete” summary back to the Telegram user.

**Nodes Involved:**  
- `📤 Send Skipped Message`
- `📤 Send Publish Summary`

### Node Details

#### `📤 Send Skipped Message`
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Confirms to the user that no social posting will occur.
- **Configuration choices:**  
  - Fixed text explaining publication was skipped
  - Chat ID comes from extracted Telegram context
- **Key expressions or variables used:**  
  - `{{ $('📋 Extract Video URL').item.json.chat_id }}`
- **Input and output connections:**  
  - Input: intended false branch of `✅ Approved?`
  - Output: none
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - Chat ID unavailable if earlier extraction failed
- **Sub-workflow reference:**  
  None

#### `📤 Send Publish Summary`
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a final success confirmation after social posting.
- **Configuration choices:**  
  - Message includes the final VEED video URL
  - Chat ID comes from extracted Telegram context
- **Key expressions or variables used:**  
  - `{{ 'Your video has been published to all configured platforms via Blotato!\n\nVideo: ' + $('📋 Extract Video URL').item.json.video_url }}`
  - `{{ $('📋 Extract Video URL').item.json.chat_id }}`
- **Input and output connections:**  
  - Input: `Merge`
  - Output: none
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - Message may be misleading if some platform nodes were disabled or failed while another branch reached the merge
  - Depends on merge behavior matching expected “all done” semantics
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Setup | n8n-nodes-base.stickyNote | Documentation / setup note |  |  | # Create Telegram AI Video Agent with VEED MCP<br>A conversational AI video agent accessible through Telegram. Users chat with a bot to create AI talking-head videos using VEED's MCP tools, then optionally publish to social media via Blotato.<br><br>## How It Works<br>1. **Message**: User sends a message to the Telegram bot<br>2. **AI Agent**: OpenAI-powered agent interprets the request and maintains conversation context<br>3. **VEED Tools**: Agent uses VEED MCP tools to browse characters/voices and create videos<br>4. **Response**: Agent sends the result back via Telegram<br>5. **Approval**: If a video URL is detected, user is asked to approve publishing<br>6. **Upload**: Video is uploaded to Blotato<br>7. **Publish**: Video is posted to 9 platforms in parallel (TikTok, YouTube, Instagram, LinkedIn, Facebook, X, Threads, Bluesky, Pinterest)<br><br>## Requirements<br>- Telegram Bot Token (via @BotFather)<br>- OpenAI API Key<br>- VEED MCP OAuth2 credentials<br>- Blotato: Install `@blotato/n8n-nodes-blotato` community node + API key (paid plan required)<br><br>**Author:** VEED.io |
| Sticky Note - How It Works | n8n-nodes-base.stickyNote | Documentation / conversation flow note |  |  | ## Conversation Flow<br>1. User describes the video they want to create<br>2. Agent lists available AI characters using `list_characters`<br>3. User picks a character<br>4. Agent lists voices using `list_voices` (filtered by locale)<br>5. User picks a voice<br>6. Agent confirms details using `confirm_fabric_video`<br>7. Agent creates the video using `create_fabric_video`<br>8. Agent tracks progress with `get_generation_status`<br>9. Agent delivers the video URL when complete<br>10. User is asked: "Publish to social media?"<br>11. If approved, video is uploaded to Blotato then posted to all platforms in parallel |
| Sticky Note - VEED Tools | n8n-nodes-base.stickyNote | Documentation / VEED capabilities note |  |  | ## Available VEED MCP Tools<br>- `list_workspaces` - Show available workspaces<br>- `list_characters` - Browse AI avatar characters<br>- `list_voices` - Browse voices by locale<br>- `confirm_fabric_video` - Preview video before creating<br>- `create_fabric_video` - Create a talking-head video<br>- `get_generation_status` - Check video progress<br>- `get_credit_balance` - Check remaining credits |
| Sticky Note - Blotato Setup | n8n-nodes-base.stickyNote | Documentation / Blotato setup note |  |  | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Sticky Note - Approval Flow | n8n-nodes-base.stickyNote | Documentation / publish approval note |  |  | ## Approval & Publishing Flow<br>After the AI agent responds, the workflow checks if the output contains a VEED video URL using the `VIDEO_READY:` marker.<br><br>If a video is found:<br>1. Telegram sends an approval prompt with Approve/Reject buttons<br>2. Workflow pauses and waits up to 24 hours for a response<br>3. **Approve**: Video is uploaded to Blotato, then posted to all 9 platforms in parallel<br>4. **Reject**: User gets a confirmation that publishing was skipped<br><br>The caption is extracted from the first 200 characters of the agent's response. Edit the **Set Caption** node to customize. |
| 📩 Telegram Trigger | n8n-nodes-base.telegramTrigger | Receives Telegram user messages |  | 🤖 AI Video Agent | # Create Telegram AI Video Agent with VEED MCP<br>A conversational AI video agent accessible through Telegram. Users chat with a bot to create AI talking-head videos using VEED's MCP tools, then optionally publish to social media via Blotato.<br><br>## How It Works<br>1. **Message**: User sends a message to the Telegram bot<br>2. **AI Agent**: OpenAI-powered agent interprets the request and maintains conversation context<br>3. **VEED Tools**: Agent uses VEED MCP tools to browse characters/voices and create videos<br>4. **Response**: Agent sends the result back via Telegram<br>5. **Approval**: If a video URL is detected, user is asked to approve publishing<br>6. **Upload**: Video is uploaded to Blotato<br>7. **Publish**: Video is posted to 9 platforms in parallel (TikTok, YouTube, Instagram, LinkedIn, Facebook, X, Threads, Bluesky, Pinterest)<br><br>## Requirements<br>- Telegram Bot Token (via @BotFather)<br>- OpenAI API Key<br>- VEED MCP OAuth2 credentials<br>- Blotato: Install `@blotato/n8n-nodes-blotato` community node + API key (paid plan required)<br><br>**Author:** VEED.io |
| 🤖 AI Video Agent | @n8n/n8n-nodes-langchain.agent | Conversational agent orchestrating VEED tool usage | 📩 Telegram Trigger; 🧠 OpenAI Chat Model; 💾 Conversation Memory; 🎬 VEED MCP Tools | 📤 Send Agent Response | ## Conversation Flow<br>1. User describes the video they want to create<br>2. Agent lists available AI characters using `list_characters`<br>3. User picks a character<br>4. Agent lists voices using `list_voices` (filtered by locale)<br>5. User picks a voice<br>6. Agent confirms details using `confirm_fabric_video`<br>7. Agent creates the video using `create_fabric_video`<br>8. Agent tracks progress with `get_generation_status`<br>9. Agent delivers the video URL when complete<br>10. User is asked: "Publish to social media?"<br>11. If approved, video is uploaded to Blotato then posted to all platforms in parallel |
| 🧠 OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing the AI agent |  | 🤖 AI Video Agent | ## Conversation Flow<br>1. User describes the video they want to create<br>2. Agent lists available AI characters using `list_characters`<br>3. User picks a character<br>4. Agent lists voices using `list_voices` (filtered by locale)<br>5. User picks a voice<br>6. Agent confirms details using `confirm_fabric_video`<br>7. Agent creates the video using `create_fabric_video`<br>8. Agent tracks progress with `get_generation_status`<br>9. Agent delivers the video URL when complete<br>10. User is asked: "Publish to social media?"<br>11. If approved, video is uploaded to Blotato then posted to all platforms in parallel |
| 💾 Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Keeps recent per-chat memory |  | 🤖 AI Video Agent | ## Conversation Flow<br>1. User describes the video they want to create<br>2. Agent lists available AI characters using `list_characters`<br>3. User picks a character<br>4. Agent lists voices using `list_voices` (filtered by locale)<br>5. User picks a voice<br>6. Agent confirms details using `confirm_fabric_video`<br>7. Agent creates the video using `create_fabric_video`<br>8. Agent tracks progress with `get_generation_status`<br>9. Agent delivers the video URL when complete<br>10. User is asked: "Publish to social media?"<br>11. If approved, video is uploaded to Blotato then posted to all platforms in parallel |
| 🎬 VEED MCP Tools | @n8n/n8n-nodes-langchain.mcpClientTool | Exposes VEED MCP tools to the agent |  | 🤖 AI Video Agent | ## Available VEED MCP Tools<br>- `list_workspaces` - Show available workspaces<br>- `list_characters` - Browse AI avatar characters<br>- `list_voices` - Browse voices by locale<br>- `confirm_fabric_video` - Preview video before creating<br>- `create_fabric_video` - Create a talking-head video<br>- `get_generation_status` - Check video progress<br>- `get_credit_balance` - Check remaining credits |
| 📤 Send Agent Response | n8n-nodes-base.telegram | Sends AI response back to Telegram | 🤖 AI Video Agent | 📋 Extract Video URL |  |
| 📋 Extract Video URL | n8n-nodes-base.code | Parses `VIDEO_READY:` URL from AI output | 📤 Send Agent Response | ✅ Has Video URL? | ## Approval & Publishing Flow<br>After the AI agent responds, the workflow checks if the output contains a VEED video URL using the `VIDEO_READY:` marker.<br><br>If a video is found:<br>1. Telegram sends an approval prompt with Approve/Reject buttons<br>2. Workflow pauses and waits up to 24 hours for a response<br>3. **Approve**: Video is uploaded to Blotato, then posted to all 9 platforms in parallel<br>4. **Reject**: User gets a confirmation that publishing was skipped<br><br>The caption is extracted from the first 200 characters of the agent's response. Edit the **Set Caption** node to customize. |
| ✅ Has Video URL? | n8n-nodes-base.if | Checks whether agent returned a final video URL | 📋 Extract Video URL | 📩 Publish to Social Media? | ## Approval & Publishing Flow<br>After the AI agent responds, the workflow checks if the output contains a VEED video URL using the `VIDEO_READY:` marker.<br><br>If a video is found:<br>1. Telegram sends an approval prompt with Approve/Reject buttons<br>2. Workflow pauses and waits up to 24 hours for a response<br>3. **Approve**: Video is uploaded to Blotato, then posted to all 9 platforms in parallel<br>4. **Reject**: User gets a confirmation that publishing was skipped<br><br>The caption is extracted from the first 200 characters of the agent's response. Edit the **Set Caption** node to customize. |
| 📩 Publish to Social Media? | n8n-nodes-base.telegram | Sends approval prompt and waits for reply | ✅ Has Video URL? | ✅ Approved? | ## Approval & Publishing Flow<br>After the AI agent responds, the workflow checks if the output contains a VEED video URL using the `VIDEO_READY:` marker.<br><br>If a video is found:<br>1. Telegram sends an approval prompt with Approve/Reject buttons<br>2. Workflow pauses and waits up to 24 hours for a response<br>3. **Approve**: Video is uploaded to Blotato, then posted to all 9 platforms in parallel<br>4. **Reject**: User gets a confirmation that publishing was skipped<br><br>The caption is extracted from the first 200 characters of the agent's response. Edit the **Set Caption** node to customize. |
| ✅ Approved? | n8n-nodes-base.if | Splits approval vs rejection flow | 📩 Publish to Social Media? | ⚙️ Set Caption; 📤 Send Skipped Message | ## Approval & Publishing Flow<br>After the AI agent responds, the workflow checks if the output contains a VEED video URL using the `VIDEO_READY:` marker.<br><br>If a video is found:<br>1. Telegram sends an approval prompt with Approve/Reject buttons<br>2. Workflow pauses and waits up to 24 hours for a response<br>3. **Approve**: Video is uploaded to Blotato, then posted to all 9 platforms in parallel<br>4. **Reject**: User gets a confirmation that publishing was skipped<br><br>The caption is extracted from the first 200 characters of the agent's response. Edit the **Set Caption** node to customize. |
| 📤 Send Skipped Message | n8n-nodes-base.telegram | Confirms publishing was skipped | ✅ Approved? |  | ## Approval & Publishing Flow<br>After the AI agent responds, the workflow checks if the output contains a VEED video URL using the `VIDEO_READY:` marker.<br><br>If a video is found:<br>1. Telegram sends an approval prompt with Approve/Reject buttons<br>2. Workflow pauses and waits up to 24 hours for a response<br>3. **Approve**: Video is uploaded to Blotato, then posted to all 9 platforms in parallel<br>4. **Reject**: User gets a confirmation that publishing was skipped<br><br>The caption is extracted from the first 200 characters of the agent's response. Edit the **Set Caption** node to customize. |
| ⚙️ Set Caption | n8n-nodes-base.set | Prepares caption and video URL for publishing | ✅ Approved? | 📤 Upload Video to Blotato | ## Approval & Publishing Flow<br>After the AI agent responds, the workflow checks if the output contains a VEED video URL using the `VIDEO_READY:` marker.<br><br>If a video is found:<br>1. Telegram sends an approval prompt with Approve/Reject buttons<br>2. Workflow pauses and waits up to 24 hours for a response<br>3. **Approve**: Video is uploaded to Blotato, then posted to all 9 platforms in parallel<br>4. **Reject**: User gets a confirmation that publishing was skipped<br><br>The caption is extracted from the first 200 characters of the agent's response. Edit the **Set Caption** node to customize. |
| 📤 Upload Video to Blotato | @blotato/n8n-nodes-blotato.blotato | Uploads VEED video into Blotato media storage | ⚙️ Set Caption | TikTok; YouTube; Instagram; LinkedIn; Facebook; Twitter (X); Threads; Bluesky; Pinterest | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| TikTok | @blotato/n8n-nodes-blotato.blotato | Publishes video to TikTok | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| YouTube | @blotato/n8n-nodes-blotato.blotato | Publishes video to YouTube | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Instagram | @blotato/n8n-nodes-blotato.blotato | Publishes video to Instagram | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| LinkedIn | @blotato/n8n-nodes-blotato.blotato | Publishes video to LinkedIn | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Facebook | @blotato/n8n-nodes-blotato.blotato | Publishes video to Facebook Page | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Twitter (X) | @blotato/n8n-nodes-blotato.blotato | Publishes video to X / Twitter | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Threads | @blotato/n8n-nodes-blotato.blotato | Publishes video to Threads | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Bluesky | @blotato/n8n-nodes-blotato.blotato | Publishes video to Bluesky | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Pinterest | @blotato/n8n-nodes-blotato.blotato | Publishes video to Pinterest | 📤 Upload Video to Blotato | Merge | ## Blotato Setup<br>1. Install the `@blotato/n8n-nodes-blotato` community node in n8n<br>2. Sign up at [blotato.com](https://blotato.com) (paid plan required)<br>3. Connect your social media accounts in the Blotato dashboard<br>4. Go to Settings > API and copy your API key<br>5. In n8n, create Blotato API credentials with your key<br>6. Configure each platform node with your `accountId` (select from the dropdown list)<br><br>## Pre-configured Platforms<br>All 9 platforms are wired and ready. Configure the `accountId` in each platform node:<br>- TikTok<br>- YouTube (also set title & privacy)<br>- Instagram<br>- LinkedIn<br>- Facebook (also set Page ID)<br>- Twitter / X<br>- Threads<br>- Bluesky<br>- Pinterest (also set Board ID)<br><br>Disable any platform node you don't need. |
| Merge | n8n-nodes-base.merge | Consolidates social posting branches | TikTok; YouTube; Instagram; LinkedIn; Facebook; Twitter (X); Threads; Bluesky; Pinterest | 📤 Send Publish Summary |  |
| 📤 Send Publish Summary | n8n-nodes-base.telegram | Sends final publication confirmation | Merge |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   Name it something like: `Create & Publish AI Videos from Telegram Chat with VEED and Blotato`.

2. **Create Telegram credentials.**
   - Create a Telegram bot with `@BotFather`.
   - Copy the bot token.
   - In n8n, create Telegram credentials for both:
     - Telegram Trigger
     - Telegram message send nodes

3. **Create OpenAI credentials.**
   - In n8n, add OpenAI API credentials.
   - Ensure the target model is available to your account, ideally `gpt-5-nano` if supported.

4. **Create VEED MCP OAuth2 credentials.**
   - Add MCP OAuth2 API credentials in n8n.
   - Configure them for the VEED MCP endpoint:
     - `https://www.veed.io/api/v1/mcp`
   - Complete OAuth authorization.

5. **Install the Blotato community node.**
   - Install `@blotato/n8n-nodes-blotato` in your n8n instance.
   - Restart n8n if required.

6. **Create Blotato API credentials.**
   - Sign up at [blotato.com](https://blotato.com).
   - Connect your social media accounts in the Blotato dashboard.
   - Copy the API key from Settings > API.
   - Create Blotato credentials in n8n.

7. **Add the entry trigger node.**
   - Create a `Telegram Trigger` node.
   - Name it `📩 Telegram Trigger`.
   - Set update types to `message`.

8. **Add the AI agent node.**
   - Create an `AI Agent` node.
   - Name it `🤖 AI Video Agent`.
   - Set the input text to:
     - `={{ $json.message.text }}`
   - Set `maxIterations` to `15`.
   - Paste the system message logic from the workflow:
     - explain VEED tools
     - enforce script confirmation
     - require `get_generation_status` after creation
     - require `VIDEO_READY: <url>` on its own line for the final video URL
     - define Telegram-safe formatting
   - Connect `📩 Telegram Trigger` → `🤖 AI Video Agent`.

9. **Add the OpenAI model node.**
   - Create an `OpenAI Chat Model` node.
   - Name it `🧠 OpenAI Chat Model`.
   - Select model `gpt-5-nano`.
   - Connect it to the AI agent’s language model port.

10. **Add the memory node.**
    - Create a `Memory Buffer Window` node.
    - Name it `💾 Conversation Memory`.
    - Set session ID type to `customKey`.
    - Set session key to:
      - `={{ $('📩 Telegram Trigger').item.json.message.chat.id }}`
    - Set context window length to `20`.
    - Connect it to the AI agent’s memory port.

11. **Add the VEED MCP tool node.**
    - Create an `MCP Client Tool` node.
    - Name it `🎬 VEED MCP Tools`.
    - Set endpoint URL to:
      - `https://www.veed.io/api/v1/mcp`
    - Set authentication to MCP OAuth2 API.
    - Attach VEED MCP credentials.
    - Connect it to the AI agent’s tool port.

12. **Add the Telegram response node.**
    - Create a `Telegram` node for sending a message.
    - Name it `📤 Send Agent Response`.
    - Set text to:
      - `={{ $json.output }}`
    - Set chat ID to:
      - `={{ $('📩 Telegram Trigger').item.json.message.chat.id }}`
    - Set error handling to continue on error.
    - Connect `🤖 AI Video Agent` → `📤 Send Agent Response`.

13. **Add the code parser node.**
    - Create a `Code` node.
    - Name it `📋 Extract Video URL`.
    - Use JavaScript mode.
    - Add logic that:
      - reads `$('🤖 AI Video Agent').item.json.output`
      - extracts URL with regex looking for `VIDEO_READY:`
      - returns `video_url`, `has_video`, `agent_output`, `chat_id`
    - Connect `📤 Send Agent Response` → `📋 Extract Video URL`.

14. **Add the first IF node for URL detection.**
    - Create an `If` node.
    - Name it `✅ Has Video URL?`
    - Add a boolean condition:
      - Left value: `={{ $json.has_video }}`
      - Operation: `is true`
    - Connect `📋 Extract Video URL` → `✅ Has Video URL?`

15. **Add the Telegram approval node.**
    - Create a `Telegram` node.
    - Name it `📩 Publish to Social Media?`
    - Set operation to `sendAndWait`.
    - Set chat ID to:
      - `={{ $json.chat_id }}`
    - Set message to:
      - `={{ "Your video is ready!\n\n" + $json.video_url + "\n\nWould you like to publish it to social media via Blotato?" }}`
    - Enable approval buttons with double approval type.
    - Set maximum wait time to 24 hours.
    - Connect the **true** output of `✅ Has Video URL?` → `📩 Publish to Social Media?`

16. **Add the approval decision IF node.**
    - Create another `If` node.
    - Name it `✅ Approved?`
    - Add boolean condition:
      - Left value: `={{ $json.data.approved }}`
      - Operation: `is true`
    - Connect `📩 Publish to Social Media?` → `✅ Approved?`

17. **Add the rejection message node.**
    - Create a `Telegram` send node.
    - Name it `📤 Send Skipped Message`.
    - Set text to:
      - `No problem! Your video won't be published to social media. You can still share it manually using the link above.`
    - Set chat ID to:
      - `={{ $('📋 Extract Video URL').item.json.chat_id }}`
    - Connect the **false** output of `✅ Approved?` → `📤 Send Skipped Message`.

18. **Add the caption preparation node.**
    - Create a `Set` node.
    - Name it `⚙️ Set Caption`.
    - Add field `caption` as string:
      - `={{ $('📋 Extract Video URL').item.json.agent_output.substring(0, 200) }}`
    - Add field `video_url` as string:
      - `={{ $('📋 Extract Video URL').item.json.video_url }}`
    - Connect the **true** output of `✅ Approved?` → `⚙️ Set Caption`.

19. **Add the Blotato media upload node.**
    - Create a `Blotato` node.
    - Name it `📤 Upload Video to Blotato`.
    - Choose resource `media`.
    - Set media URL to:
      - `={{ $json.video_url }}`
    - Attach Blotato credentials.
    - Connect `⚙️ Set Caption` → `📤 Upload Video to Blotato`.

20. **Create the TikTok posting node.**
    - Add a `Blotato` node named `TikTok`.
    - Platform: `tiktok`
    - Select `accountId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

21. **Create the YouTube posting node.**
    - Add a `Blotato` node named `YouTube`.
    - Platform: `youtube`
    - Select `accountId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Title:
      - `={{ $('⚙️ Set Caption').item.json.caption.substring(0, 100) }}`
    - Privacy status: `private`
    - Notify subscribers: `false`
    - Connect from `📤 Upload Video to Blotato`.

22. **Create the Instagram posting node.**
    - Add a `Blotato` node named `Instagram`.
    - Select Instagram account.
    - If your installed node version requires it, explicitly set platform to `instagram`.
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

23. **Create the LinkedIn posting node.**
    - Add a `Blotato` node named `LinkedIn`.
    - Platform: `linkedin`
    - Select `accountId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

24. **Create the Facebook posting node.**
    - Add a `Blotato` node named `Facebook`.
    - Platform: `facebook`
    - Select `accountId`
    - Select `facebookPageId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

25. **Create the X posting node.**
    - Add a `Blotato` node named `Twitter (X)`.
    - Platform: `twitter`
    - Select `accountId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

26. **Create the Threads posting node.**
    - Add a `Blotato` node named `Threads`.
    - Platform: `threads`
    - Select `accountId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

27. **Create the Bluesky posting node.**
    - Add a `Blotato` node named `Bluesky`.
    - Platform: `bluesky`
    - Select `accountId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

28. **Create the Pinterest posting node.**
    - Add a `Blotato` node named `Pinterest`.
    - Platform: `pinterest`
    - Select `accountId`
    - Set `pinterestBoardId`
    - Post text:
      - `={{ $('⚙️ Set Caption').item.json.caption }}`
    - Post media URLs:
      - `={{ $json.url }}`
    - Connect from `📤 Upload Video to Blotato`.

29. **Add the merge node.**
    - Create a `Merge` node.
    - Name it `Merge`.
    - Set mode to `chooseBranch`.
    - Set number of inputs to `9`.
    - Connect each social node into a separate merge input:
      - TikTok → 0
      - YouTube → 1
      - Instagram → 2
      - LinkedIn → 3
      - Facebook → 4
      - Twitter (X) → 5
      - Threads → 6
      - Bluesky → 7
      - Pinterest → 8

30. **Add the final Telegram summary node.**
    - Create a `Telegram` send node.
    - Name it `📤 Send Publish Summary`.
    - Text:
      - `={{ 'Your video has been published to all configured platforms via Blotato!\n\nVideo: ' + $('📋 Extract Video URL').item.json.video_url }}`
    - Chat ID:
      - `={{ $('📋 Extract Video URL').item.json.chat_id }}`
    - Connect `Merge` → `📤 Send Publish Summary`.

31. **Optionally add sticky notes for maintainability.**
    - Add notes describing:
      - setup requirements
      - conversation flow
      - VEED tools
      - approval flow
      - Blotato setup

32. **Test the Telegram conversation path first.**
    - Confirm:
      - bot receives messages
      - agent replies
      - VEED tools are callable
      - memory persists per Telegram chat
      - final completed responses include `VIDEO_READY: <url>`

33. **Test the approval flow.**
    - Ensure a generated message with a valid marker triggers the send-and-wait node.
    - Approve and reject separately.

34. **Test one social platform before enabling all nine.**
    - Configure a single Blotato platform node first.
    - Disable the others until successful.
    - Then re-enable the rest.

35. **Recommended implementation corrections during rebuild.**
    - Wire `✅ Approved?` with explicit true and false outputs.
    - Consider replacing `Merge` mode if you truly need to wait for all 9 posting branches before sending the final summary.
    - Consider cleaning the caption so `VIDEO_READY:` is not included in social text.

### Credential configuration summary
- **Telegram:** Bot token from BotFather
- **OpenAI:** API key with access to selected model
- **VEED MCP OAuth2:** OAuth2 credentials authorized against VEED MCP
- **Blotato:** API key from Blotato dashboard

### Sub-workflow setup
This workflow does **not** call any sub-workflow and has **one main entry point**:
- `📩 Telegram Trigger`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: VEED.io | Workflow branding / credit |
| Blotato account and API setup are required, and a paid plan is mentioned | [https://blotato.com](https://blotato.com) |
| Community node required for social publishing | `@blotato/n8n-nodes-blotato` |
| Telegram bot token must be created through BotFather | Telegram setup requirement |
| VEED MCP endpoint used by the workflow | `https://www.veed.io/api/v1/mcp` |
| The workflow depends on the AI agent emitting the exact `VIDEO_READY:` marker for downstream publishing to trigger | Core integration constraint |
| The current JSON suggests an ambiguous IF-node branch wiring for approval handling; rebuild with explicit true/false outputs | Implementation caution |
| The current merge mode is `chooseBranch`, which may not guarantee waiting for all platform posts before notifying the user | Execution semantics caution |