Get AI insights from Databox in Slack using OpenAI

https://n8nworkflows.xyz/workflows/get-ai-insights-from-databox-in-slack-using-openai-14322


# Get AI insights from Databox in Slack using OpenAI

# 1. Workflow Overview

This workflow lets Slack users ask questions about business metrics by mentioning a bot in a Slack channel. The workflow sends the Slack message to an AI agent, which uses Databox through MCP to retrieve relevant live data and then posts a concise answer back into Slack.

Typical use cases include:
- Checking recent ad performance
- Comparing current vs. previous periods
- Asking for high-level KPI summaries
- Getting a short AI-generated insight directly in a Slack conversation

The workflow is organized into three main logical blocks.

## 1.1 Slack Input Reception

This block listens for Slack bot mentions in a specific channel. When a user mentions the bot, the message text and channel context are captured and passed to the AI layer.

## 1.2 AI Processing with Databox MCP

This block contains the AI agent, its language model, and the Databox MCP tool. The agent receives the user’s Slack question, follows a defined retrieval strategy, calls Databox MCP tools to discover accounts/data sources and fetch metrics, then formats a Slack-friendly response.

## 1.3 Slack Response Delivery

This block posts the AI-generated response back to Slack using the same channel from which the mention originated.

---

# 2. Block-by-Block Analysis

## 2.1 Slack Input Reception

### Overview
This block is the workflow entry point. It waits for `app_mention` events from Slack and filters them to a configured Slack channel before forwarding the event payload to the AI agent.

### Nodes Involved
- Sticky Note
- Sticky Note 1
- Slack Trigger

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation/annotation only.
- **Configuration choices:** Provides the overall purpose of the workflow, supported data sources, and prerequisite services.
- **Key expressions or variables used:** None.
- **Input and output connections:** None; visual documentation only.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note 1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; setup guidance for Slack.
- **Configuration choices:** Explains Slack app creation, permission scopes, webhook registration, and channel setup.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Slack Trigger
- **Type and technical role:** `n8n-nodes-base.slackTrigger`; webhook-based event trigger for incoming Slack bot events.
- **Configuration choices:**
  - Trigger listens for `app_mention`
  - Restricted to a specific channel via `channelId`
  - Uses a Slack API credential containing bot token and signing secret
- **Key expressions or variables used:**
  - Receives Slack event payload fields such as:
    - `$json.text`
    - `$json.channel`
- **Input and output connections:**
  - Entry point node; no input
  - Outputs to **Databox AI Assistant**
- **Version-specific requirements:**
  - Uses node type version `1`
  - Requires Slack event subscription support in the workspace and correct signing secret handling in n8n
- **Edge cases or potential failure types:**
  - Invalid Slack signing secret
  - Missing `app_mentions:read` permission
  - Slack webhook verification failure if workflow is inactive
  - Channel mismatch if the configured channel ID is wrong
  - Bot not invited to the channel
  - Slack event delivery retries if n8n does not acknowledge quickly
- **Sub-workflow reference:** None.

---

## 2.2 AI Processing with Databox MCP

### Overview
This is the core logic block. The AI agent receives the Slack mention text, uses an OpenAI chat model for reasoning and response generation, and calls Databox MCP tools to discover accounts, inspect available data sources, and retrieve metric data relevant to the user’s question.

### Nodes Involved
- Sticky Note 2
- Databox AI Assistant
- OpenAI Chat Model
- Databox MCP Tool

### Node Details

#### Sticky Note 2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; setup guidance for the AI and Databox integration.
- **Configuration choices:** Documents OpenAI credential setup, Databox OAuth2 authorization, and a video reference for MCP connection.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Databox AI Assistant
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrating AI agent that takes user input, reasons over it, invokes tools, and generates the final answer.
- **Configuration choices:**
  - Prompt type is explicitly defined
  - Input text is mapped from the Slack event text
  - A detailed system message instructs the agent to:
    1. Call `list_accounts`
    2. Call `list_data_sources`
    3. Identify relevant sources and metrics
    4. Fetch data using `load_metric_data` or `ask_genie`
    5. Calculate comparisons/trends
    6. Return a short Slack-friendly answer
  - Formatting guidance emphasizes:
    - concise responses
    - bold formatting
    - emojis
    - percent change for comparisons
    - currency/rate formatting
    - a short recommendation at the end
    - no disclaimers/sign-offs
    - reasonable assumptions for ambiguous questions
  - `alwaysOutputData` is enabled
  - `retryOnFail` is disabled
- **Key expressions or variables used:**
  - Input text: `={{ $json.text }}`
  - Final generated answer is later exposed as `$json.output`
- **Input and output connections:**
  - Main input from **Slack Trigger**
  - AI language model input from **OpenAI Chat Model**
  - AI tool input from **Databox MCP Tool**
  - Main output to **Post message**
- **Version-specific requirements:**
  - Uses node type version `3`
  - Requires n8n build with LangChain/AI agent support and MCP tool compatibility
- **Edge cases or potential failure types:**
  - User question too vague or too broad
  - Tool call failure from Databox MCP
  - LLM quota/rate-limit/authentication issues
  - Unexpected output structure if model/tool interaction fails
  - Long or malformed Slack messages causing poor prompt interpretation
  - Agent may make assumptions if the prompt is ambiguous
  - Since retries are disabled, transient failures are not retried automatically
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model provider used by the AI agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - `maxTokens`: `4096`
  - Built-in tools are not configured here; external tool use is delegated through the agent/tool nodes
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**
  - Connected to **Databox AI Assistant** via `ai_languageModel`
- **Version-specific requirements:**
  - Uses node type version `1.3`
  - Requires valid OpenAI API credentials
  - Requires model availability in the connected OpenAI account
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model access not enabled
  - OpenAI rate limiting or quota exhaustion
  - Token limits if prompts/tool transcripts grow large
  - Regional or account-level restrictions
- **Sub-workflow reference:** None.

#### Databox MCP Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`; MCP client tool exposing Databox actions to the AI agent.
- **Configuration choices:**
  - Endpoint URL: `https://mcp.databox.com/mcp`
  - Authentication: `OAuth2` via `mcpOAuth2Api`
  - No extra options configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Connected to **Databox AI Assistant** via `ai_tool`
- **Version-specific requirements:**
  - Uses node type version `1.2`
  - Requires n8n support for MCP client tools
  - Requires valid Databox OAuth2 authorization
- **Edge cases or potential failure types:**
  - OAuth2 authorization failure
  - Databox MCP endpoint unavailability
  - User’s Databox account missing required connected data sources
  - Tool actions returning no data for the requested period
  - Schema/tool-action changes on the Databox MCP side
- **Sub-workflow reference:** None.

---

## 2.3 Slack Response Delivery

### Overview
This block publishes the agent’s final answer back to Slack. It dynamically targets the same Slack channel from the original event and posts the generated text as a message.

### Nodes Involved
- Sticky Note 3
- Post message

### Node Details

#### Sticky Note 3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; setup guidance for outgoing Slack messages.
- **Configuration choices:** Notes that the same bot credential should be reused and suggests optionally adding error handling.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Post message
- **Type and technical role:** `n8n-nodes-base.slack`; sends a Slack message.
- **Configuration choices:**
  - Operation targets a channel
  - Message text comes from the agent’s output
  - Channel ID is dynamically taken from the Slack trigger event
  - Uses the same Slack credential as the trigger
- **Key expressions or variables used:**
  - Text: `={{ $json.output }}`
  - Channel ID: `={{ $('Slack Trigger').item.json.channel }}`
- **Input and output connections:**
  - Main input from **Databox AI Assistant**
  - No downstream nodes
- **Version-specific requirements:**
  - Uses node type version `2.2`
  - Requires Slack bot token with `chat:write`
- **Edge cases or potential failure types:**
  - Invalid or expired Slack token
  - Missing `chat:write` scope
  - Bot not present in the target channel
  - Empty or null `$json.output`
  - Expression failure if Slack Trigger data is unavailable in the execution context
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | High-level workflow documentation and prerequisites |  |  | ## Ask Your Business Data in Slack and Get AI-powered Insights via Databox MCP<br>Get instant answers about your business metrics directly in Slack. Mention the bot with any question - "How did Google Ads perform this week?" or "What's our YouTube watch time vs. last month?" - and the AI agent pulls live data from Databox via MCP and replies in the channel.<br>### How it works<br>`Slack Trigger (Bot Mention)` → `AI Agent (queries Databox MCP)` → `Reply in Slack`<br>### What you need<br>- Databox account with data sources connected → Free plan available: https://databox.com/?ref=n8n<br>- A custom Slack app (see setup instructions below)<br>- OpenAI or Claude API key<br>### Supported Data Sources<br>Google Ads • Google Analytics • Facebook Ads • YouTube • HubSpot • Salesforce • Stripe • And 100+ more → https://databox.com/integrations |
| Sticky Note 1 | n8n-nodes-base.stickyNote | Slack app and trigger setup guidance |  |  | ## 1️⃣ Slack App + Trigger Setup<br>### What this section does<br>Listens for @mentions of your Slack bot in a specific channel and forwards the question to the AI Agent.<br>### What you need to do ⚠️<br>### Step 1 - Create a Slack app<br>1. Go to https://api.slack.com/apps and click **Create an App** → From scratch<br>2. Name it (e.g. "Databox Assistant") and select your workspace<br>3. Go to **OAuth & Permissions** → under Bot Token Scopes, add:<br>- `app_mentions:read`<br>- `chat:write`<br>4. Click **Install to Workspace** and authorize - you'll get a **Bot User OAuth Token** (starts with `xoxb-`)<br>5. Go to **Basic Information** → copy the **Signing Secret**<br>### Step 2 - Create credential in n8n<br>1. In n8n, go to **Credentials** → Add credential → search **Slack API**<br>2. Paste the **Bot User OAuth Token** into Access Token<br>3. Paste the **Signing Secret** into Signature Secret<br>4. Save and select this credential in both the **Slack Trigger** and **Reply in Thread** nodes<br>### Step 3 - Register the webhook<br>1. Copy the **Production URL** from the Slack Trigger node (expand Webhook URLs at the top)<br>2. Make the workflow **Active** (toggle top right in n8n)<br>3. In your Slack app, go to **Event Subscriptions** → toggle On<br>4. Paste the webhook URL - Slack will verify it automatically<br>5. Under **Subscribe to bot events**, add `app_mention`<br>6. Click **Save Changes**<br>### Step 4 - Add bot to channel<br>1. In Slack, go to the channel you want to monitor<br>2. Type `/invite @YourBotName` to add the bot<br>3. Update the **Channel to Watch** field in this trigger node to your channel ID<br>- Find your channel ID: right-click channel → View channel details → scroll to bottom |
| Sticky Note 2 | n8n-nodes-base.stickyNote | AI and Databox MCP setup guidance |  |  | ## 2️⃣ AI Agent + Databox MCP Setup<br>### What this section does<br>The AI Agent reads the user's Slack question, queries Databox via MCP for the relevant data, and formats a concise Slack-friendly reply.<br>### What you need to do ⚠️<br>1. Click **OpenAI Chat Model** → add your `API key` credential<br>- You can replace this with the **Anthropic Chat Model** node<br>2. Click **Databox MCP Tool** → set Authentication to `OAuth2` → authorize with your Databox account<br>3. Ensure the data sources relevant to your team's questions are connected in Databox<br>### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |
| Sticky Note 3 | n8n-nodes-base.stickyNote | Slack output guidance |  |  | ## 3️⃣ Slack Message<br>### What this section does<br>The AI agent delivers in Slack, augmented with intelligent analysis to provide deeper insights and actionable information.<br>### What you need to do ⚠️<br>- Use the **same Slack API credential** as in the Slack Trigger node (the bot token, not a personal OAuth account)<br>- Ensure your bot has been invited to the channel with `/invite @YourBotName`<br>- **Optional**: Add an error message node if the agent returns no data |
| Slack Trigger | n8n-nodes-base.slackTrigger | Receives Slack bot mentions in a specific channel |  | Databox AI Assistant | ## 1️⃣ Slack App + Trigger Setup<br>### What this section does<br>Listens for @mentions of your Slack bot in a specific channel and forwards the question to the AI Agent.<br>### What you need to do ⚠️<br>### Step 1 - Create a Slack app<br>1. Go to https://api.slack.com/apps and click **Create an App** → From scratch<br>2. Name it (e.g. "Databox Assistant") and select your workspace<br>3. Go to **OAuth & Permissions** → under Bot Token Scopes, add:<br>- `app_mentions:read`<br>- `chat:write`<br>4. Click **Install to Workspace** and authorize - you'll get a **Bot User OAuth Token** (starts with `xoxb-`)<br>5. Go to **Basic Information** → copy the **Signing Secret**<br>### Step 2 - Create credential in n8n<br>1. In n8n, go to **Credentials** → Add credential → search **Slack API**<br>2. Paste the **Bot User OAuth Token** into Access Token<br>3. Paste the **Signing Secret** into Signature Secret<br>4. Save and select this credential in both the **Slack Trigger** and **Reply in Thread** nodes<br>### Step 3 - Register the webhook<br>1. Copy the **Production URL** from the Slack Trigger node (expand Webhook URLs at the top)<br>2. Make the workflow **Active** (toggle top right in n8n)<br>3. In your Slack app, go to **Event Subscriptions** → toggle On<br>4. Paste the webhook URL - Slack will verify it automatically<br>5. Under **Subscribe to bot events**, add `app_mention`<br>6. Click **Save Changes**<br>### Step 4 - Add bot to channel<br>1. In Slack, go to the channel you want to monitor<br>2. Type `/invite @YourBotName` to add the bot<br>3. Update the **Channel to Watch** field in this trigger node to your channel ID<br>- Find your channel ID: right-click channel → View channel details → scroll to bottom |
| Databox AI Assistant | @n8n/n8n-nodes-langchain.agent | Orchestrates LLM reasoning and Databox MCP tool usage | Slack Trigger; OpenAI Chat Model; Databox MCP Tool | Post message | ## 2️⃣ AI Agent + Databox MCP Setup<br>### What this section does<br>The AI Agent reads the user's Slack question, queries Databox via MCP for the relevant data, and formats a concise Slack-friendly reply.<br>### What you need to do ⚠️<br>1. Click **OpenAI Chat Model** → add your `API key` credential<br>- You can replace this with the **Anthropic Chat Model** node<br>2. Click **Databox MCP Tool** → set Authentication to `OAuth2` → authorize with your Databox account<br>3. Ensure the data sources relevant to your team's questions are connected in Databox<br>### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |
| Databox MCP Tool | @n8n/n8n-nodes-langchain.mcpClientTool | Exposes Databox MCP actions to the AI agent |  | Databox AI Assistant | ## 2️⃣ AI Agent + Databox MCP Setup<br>### What this section does<br>The AI Agent reads the user's Slack question, queries Databox via MCP for the relevant data, and formats a concise Slack-friendly reply.<br>### What you need to do ⚠️<br>1. Click **OpenAI Chat Model** → add your `API key` credential<br>- You can replace this with the **Anthropic Chat Model** node<br>2. Click **Databox MCP Tool** → set Authentication to `OAuth2` → authorize with your Databox account<br>3. Ensure the data sources relevant to your team's questions are connected in Databox<br>### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the chat model used by the AI agent |  | Databox AI Assistant | ## 2️⃣ AI Agent + Databox MCP Setup<br>### What this section does<br>The AI Agent reads the user's Slack question, queries Databox via MCP for the relevant data, and formats a concise Slack-friendly reply.<br>### What you need to do ⚠️<br>1. Click **OpenAI Chat Model** → add your `API key` credential<br>- You can replace this with the **Anthropic Chat Model** node<br>2. Click **Databox MCP Tool** → set Authentication to `OAuth2` → authorize with your Databox account<br>3. Ensure the data sources relevant to your team's questions are connected in Databox<br>### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |
| Post message | n8n-nodes-base.slack | Sends the AI response back to Slack | Databox AI Assistant |  | ## 3️⃣ Slack Message<br>### What this section does<br>The AI agent delivers in Slack, augmented with intelligent analysis to provide deeper insights and actionable information.<br>### What you need to do ⚠️<br>- Use the **same Slack API credential** as in the Slack Trigger node (the bot token, not a personal OAuth account)<br>- Ensure your bot has been invited to the channel with `/invite @YourBotName`<br>- **Optional**: Add an error message node if the agent returns no data |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **Ask your business data in Slack and get AI-powered insights**.

2. **Create the Slack app before configuring nodes**
   - Go to `https://api.slack.com/apps`
   - Create a new app from scratch
   - Choose your Slack workspace
   - In **OAuth & Permissions**, add these Bot Token Scopes:
     - `app_mentions:read`
     - `chat:write`
   - Install the app to the workspace
   - Copy:
     - the **Bot User OAuth Token** (`xoxb-...`)
     - the **Signing Secret**

3. **Create the Slack credential in n8n**
   - Add a **Slack API** credential
   - Set:
     - **Access Token** = Slack Bot User OAuth Token
     - **Signature Secret** = Slack Signing Secret
   - Save the credential
   - This same credential will be reused in both Slack nodes

4. **Add the Slack Trigger node**
   - Node type: **Slack Trigger**
   - Configure:
     - **Trigger event**: `app_mention`
     - **Channel to Watch / Channel ID**: set the Slack channel ID you want to monitor
   - Select the Slack API credential created earlier
   - Keep default options unless you need additional filtering

5. **Activate the workflow temporarily to register the webhook**
   - Open the Slack Trigger node and copy its **Production URL**
   - In Slack app settings:
     - Go to **Event Subscriptions**
     - Turn subscriptions on
     - Paste the Production URL into the request URL field
     - Add the bot event: `app_mention`
   - Save changes
   - Invite the bot to the target channel in Slack using `/invite @BotName`

6. **Create the OpenAI credential**
   - Add an **OpenAI API** credential in n8n
   - Paste a valid API key
   - Ensure the account has access to the chosen model, here `gpt-4o`

7. **Add the OpenAI Chat Model node**
   - Node type: **OpenAI Chat Model**
   - Configure:
     - **Model**: `gpt-4o`
     - **Max Tokens**: `4096`
   - Attach the OpenAI credential
   - Do not add built-in tools here

8. **Create the Databox credential**
   - Add the credential required for **MCP OAuth2 API**
   - Authorize it against your Databox account
   - Ensure your Databox workspace already has the relevant data sources connected

9. **Add the Databox MCP Tool node**
   - Node type: **MCP Client Tool**
   - Configure:
     - **Endpoint URL**: `https://mcp.databox.com/mcp`
     - **Authentication**: `OAuth2`
   - Select the Databox OAuth2 credential
   - Leave additional options empty unless your environment requires them

10. **Add the AI Agent node**
    - Node type: **AI Agent**
    - Configure the main text input as:
      - `={{ $json.text }}`
    - Set prompt mode to a defined prompt/system message
    - Use this system behavior:
      - It is a Databox data assistant
      - It should first inspect available accounts and data sources
      - It should fetch relevant data for the question
      - It should default to the last 7 days when no period is given
      - It should calculate trends/comparisons if requested
      - It should output short Slack-friendly text
      - It should use bold formatting, emojis, and one short insight/recommendation
      - It should avoid disclaimers or sign-offs
      - It should make a reasonable assumption if the user is ambiguous

11. **Paste the system message into the AI Agent**
    - Configure the system message with the following logic:
      - Call `list_accounts`
      - Call `list_data_sources`
      - Identify relevant source(s) and metrics
      - Fetch with `load_metric_data` or `ask_genie`
      - Calculate requested comparisons/trends
      - Format a concise answer for Slack

12. **Connect the model to the AI Agent**
    - Connect **OpenAI Chat Model** to **Databox AI Assistant** using the AI language model connection.

13. **Connect the Databox tool to the AI Agent**
    - Connect **Databox MCP Tool** to **Databox AI Assistant** using the AI tool connection.

14. **Connect the Slack Trigger to the AI Agent**
    - Main connection:
      - **Slack Trigger** → **Databox AI Assistant**

15. **Add the outgoing Slack node**
    - Node type: **Slack**
    - Operation: send/post a message to a channel
    - Configure:
      - **Text**: `={{ $json.output }}`
      - **Channel ID**: `={{ $('Slack Trigger').item.json.channel }}`
    - Reuse the same Slack API credential as the Slack Trigger

16. **Connect the AI Agent to the Slack output node**
    - Main connection:
      - **Databox AI Assistant** → **Post message**

17. **Optionally add documentation sticky notes**
    - Add sticky notes for:
      - overall workflow purpose
      - Slack setup instructions
      - Databox/OpenAI setup instructions
      - Slack output notes

18. **Test the workflow**
    - Activate the workflow
    - In Slack, mention the bot in the configured channel, for example:
      - `@Databox Assistant How did Google Ads perform this week?`
      - `@Databox Assistant Compare YouTube watch time vs last month`
    - Confirm that:
      - the trigger fires
      - the AI agent invokes Databox MCP
      - a response is posted back to Slack

19. **Validate expected input/output behavior**
    - **Input expectation:** Slack event payload with at least:
      - `text`
      - `channel`
    - **AI Agent expectation:** produces an `output` field containing final response text
    - **Slack output expectation:** posts `output` into the originating Slack channel

20. **Recommended hardening**
    - Add an **IF** node or fallback branch before Slack posting if `output` is empty
    - Add an error handling path for:
      - OpenAI failure
      - Databox authorization failure
      - no metrics returned
    - Consider posting to the original thread instead of the channel if threaded behavior is desired

### Sub-workflow Setup
- This workflow does **not** use any Execute Workflow or callable sub-workflow node.
- There are no sub-workflow dependencies to build separately.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Databox free plan available | https://databox.com/?ref=n8n |
| Supported integrations include Google Ads, Google Analytics, Facebook Ads, YouTube, HubSpot, Salesforce, Stripe, and 100+ more | https://databox.com/integrations |
| Slack app creation and management | https://api.slack.com/apps |
| Video reference for connecting Databox MCP Tool in n8n | YouTube reference: `892KtXhv-vI` |
| The workflow is active in the provided export | Operational state from workflow metadata |
| Workflow tags: Databox MCP, Slack, Automation | Metadata/context |