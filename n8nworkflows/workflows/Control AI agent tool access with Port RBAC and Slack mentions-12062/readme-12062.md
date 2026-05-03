Control AI agent tool access with Port RBAC and Slack mentions

https://n8nworkflows.xyz/workflows/control-ai-agent-tool-access-with-port-rbac-and-slack-mentions-12062


# Control AI agent tool access with Port RBAC and Slack mentions

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Control AI agent tool access with Port RBAC and Slack mentions  
**Workflow name (in JSON):** Access Control for AI Agents (RBAC) using Port and Slack

This workflow enforces **role-based access control (RBAC)** for an n8n AI Agent invoked from **Slack @mentions**. It looks up the Slack user in **Port** (using an `rbacUser` blueprint keyed by email), retrieves the user’s `allowed_tools`, dynamically **filters the agent’s connected tools at runtime**, and replies back in Slack. Unauthorized tools are replaced with a safe stub that instructs the agent to respond: *“You are not authorized to use this tool”.*

### Logical blocks
1. **1.1 Slack Event Intake & Identity Resolution**  
   Trigger on Slack mention, fetch Slack user profile (email).
2. **1.2 Port Authentication & RBAC Lookup**  
   Get Port access token, query Port for the user entity by email.
3. **1.3 User Existence Gate + Input Shaping**  
   If user not found: notify in Slack. If found: format fields for the agent (name, roles, allowed tools).
4. **1.4 Tool Access Control Layer (Runtime Tool Filtering)**  
   Replace unauthorized tools with a “not authorized” DynamicTool.
5. **1.5 Agent Execution (LLM + Memory + Tools) & Response Posting**  
   Run AI Agent with OpenAI model, memory, filtered tools; post output back to Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Slack Event Intake & Identity Resolution

**Overview:** Receives Slack `app_mention` events, then retrieves the user’s Slack profile to obtain a stable identifier (email) for RBAC lookup in Port.

**Nodes involved:**
- Slack Trigger
- Get user’s slack profile

#### Node: Slack Trigger
- **Type / Role:** `n8n-nodes-base.slackTrigger` — Entry point; listens to Slack events.
- **Key configuration:**
  - Trigger: `app_mention`
  - Channel: `YOUR_CHANNEL_ID` (must match where the bot is mentioned)
- **Inputs/Outputs:** No inputs; outputs Slack event payload including `user`, `text`, `channel`.
- **Credentials:** Slack API credential required (OAuth / bot token depending on n8n setup).
- **Edge cases / failures:**
  - Bot not invited to channel → no events.
  - Missing Slack scopes (e.g., events:read, app_mentions:read) → trigger fails.
  - Channel ID mismatch → events may not fire as expected.

#### Node: Get user’s slack profile
- **Type / Role:** `n8n-nodes-base.slack` (User → Get Profile) — Resolves the Slack user to an email.
- **Key configuration:**
  - Operation: `user.getProfile`
  - User ID: expression `={{ $json.user }}`
- **Inputs/Outputs:**
  - Input: Slack Trigger event.
  - Output: Slack profile payload; used later as `...item.json.email` in the Port lookup URL.
- **Credentials:** Slack API credential.
- **Edge cases / failures:**
  - Email not available due to Slack workspace settings or missing scope (`users:read.email`) → Port lookup will break.
  - `user` missing in payload (rare) → expression failure.

---

### 2.2 Port Authentication & RBAC Lookup

**Overview:** Authenticates to Port and retrieves the RBAC user entity by Slack email from the `rbacUser` blueprint.

**Nodes involved:**
- Get Port access token
- Get user permission from Port

#### Node: Get Port access token
- **Type / Role:** `n8n-nodes-base.httpRequest` — Fetches a Port access token.
- **Key configuration:**
  - Method: `POST`
  - URL: `https://api.port.io/v1/auth/access_token`
  - JSON body includes:
    - `clientId`: `YOUR_PORT_CLIENT_ID`
    - `clientSecret`: `YOUR_PORT_CLIENT_SECRET`
  - “Send Body”: enabled, Body type: JSON
- **Inputs/Outputs:**
  - Input: from Slack profile node.
  - Output: Port token response (typically includes an access token).
- **Edge cases / failures:**
  - Wrong client credentials → 401/403.
  - Port outage / timeout → HTTP node error.
- **Important integration note:**  
  The next node (“Get user permission from Port”) is configured to use **HTTP Bearer Auth credentials** (generic credential type). Ensure that credential is set up correctly (see reproduction section). If you intend to use the token returned by this node dynamically, you would typically map it into the Authorization header; this workflow instead uses a static bearer credential configuration.

#### Node: Get user permission from Port
- **Type / Role:** `n8n-nodes-base.httpRequest` — Reads the RBAC entity for the user.
- **Key configuration:**
  - URL (expression):  
    `https://api.port.io/v1/blueprints/rbacUser/entities/{{ $('Get user\'s slack profile').item.json.email }}`
  - Authentication: `genericCredentialType` → `httpBearerAuth`
- **Inputs/Outputs:**
  - Input: from “Get Port access token”.
  - Output: Port API response (expected to include `ok` boolean and `entity` data if found).
- **Edge cases / failures:**
  - If Slack email is empty → URL malformed or Port returns not found.
  - If blueprint `rbacUser` does not exist → 404.
  - If identifier in Port is not the email → not found (`ok` false).
  - Bearer token invalid/expired → 401.

---

### 2.3 User Existence Gate + Input Shaping

**Overview:** Branches behavior depending on whether Port found the user. If not found, sends an error message to Slack. If found, formats data for downstream nodes.

**Nodes involved:**
- Unknown user (IF)
- Send a message
- Set input

#### Node: Unknown user
- **Type / Role:** `n8n-nodes-base.if` — Checks whether Port lookup succeeded.
- **Key configuration:**
  - Condition: boolean check that `$json.ok` is **false**.
- **Inputs/Outputs:**
  - Input: Port lookup response.
  - Outputs:
    - **True branch** (user unknown): to “Send a message”
    - **False branch** (user exists): to “Set input”
- **Edge cases / failures:**
  - If Port response does not contain `ok` → expression/validation issue.
  - If Port uses a different response shape (API version change) → condition may misroute.

#### Node: Send a message
- **Type / Role:** `n8n-nodes-base.slack` — Notifies when the user is not found in Port.
- **Key configuration:**
  - Text: `User not found in Port. Please contact your administrator.`
  - Channel: `YOUR_CHANNEL_ID` (static)
- **Inputs/Outputs:** Receives “unknown user” branch; posts message.
- **Credentials:** Slack API credential.
- **Edge cases / failures:**
  - Posting to wrong channel ID or missing `chat:write` scope.

#### Node: Set input
- **Type / Role:** `n8n-nodes-base.set` — Normalizes user data for the agent layer.
- **Key configuration (assigned fields):**
  - `name` = `={{ $json.entity.identifier }}`
  - `granted_roles` = `={{ $json.entity.relations.roles || [] }}`
  - `allowed_tools` = `={{ $json.entity.properties.allowed_tools || [] }}`
- **Inputs/Outputs:**
  - Input: Port entity response (known user path).
  - Output: simplified JSON used by Agent, Memory, and Permission filter.
- **Edge cases / failures:**
  - If `entity` missing → expression failure.
  - If `allowed_tools` is not an array in Port → tool filtering will misbehave (e.g., `.includes` not working as intended).

---

### 2.4 Tool Access Control Layer (Runtime Tool Filtering)

**Overview:** Intercepts the set of tools connected to the agent, compares each tool name against `allowed_tools`, and replaces unauthorized tools with a DynamicTool that returns a fixed denial instruction.

**Nodes involved:**
- Check permissions

#### Node: Check permissions
- **Type / Role:** `@n8n/n8n-nodes-langchain.code` — Produces an `ai_tool` output list for the agent.
- **Key configuration:**
  - Expects tool connections via `ai_tool` input.
  - Reads allowed tools from: `$input.item.json.allowed_tools`
  - Code behavior:
    - Gets connected tools: `await this.getInputConnectionData('ai_tool', 0)`
    - For each tool:
      - If tool name is in `allowed_tools`, keep it.
      - Otherwise replace with `DynamicTool` of same name/description and `func` returning:  
        `"Tell the user 'You are not authorized to use this tool'."`
- **Inputs/Outputs:**
  - Input: `ai_tool` tools from tool nodes (Calculator, Wikipedia, AWS S3, PagerDuty).
  - Output: filtered tools to the AI Agent `ai_tool` input.
- **Version-specific requirements:**
  - Uses LangChain tool APIs (`@langchain/core/tools`). Requires compatible n8n LangChain node versions (here `typeVersion: 1`).
- **Edge cases / failures:**
  - If `allowed_tools` is undefined/not array → `.includes()` may throw.
  - Tool name mismatch: if Port stores tool names differently than `tool.getName()` returns, access control will deny unintentionally.
  - If no tools connected on `ai_tool` input, connectedTools may be empty and the agent may fail to answer (by design, given system prompt).

---

### 2.5 Agent Execution (LLM + Memory + Tools) & Response Posting

**Overview:** Runs an AI Agent instructed to use only provided tools, with per-user memory and a filtered toolset. Posts the agent’s final output back to the Slack channel where the mention occurred.

**Nodes involved:**
- OpenAI Chat Model
- Chat Memory
- AI Agent
- Send output message
- Tool nodes: calculator, Wikipedia, Create a bucket in AWS S3, Create an incident in PagerDuty

#### Node: OpenAI Chat Model
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM backend for the agent.
- **Key configuration:**
  - Model: `gpt-4o`
- **Inputs/Outputs:**
  - Output as `ai_languageModel` into AI Agent.
- **Credentials:** OpenAI API credential.
- **Edge cases / failures:**
  - Invalid API key / quota exceeded.
  - Model not available in the account/region.

#### Node: Chat Memory
- **Type / Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Stores short window of conversation per user.
- **Key configuration:**
  - Session ID type: `customKey`
  - Session key: `={{ $json.name }}`
- **Inputs/Outputs:**
  - Receives formatted user JSON from “Set input”.
  - Outputs as `ai_memory` into AI Agent.
- **Edge cases / failures:**
  - If `name` is empty/non-unique, memory collisions can occur across users.

#### Node: AI Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates tool usage to answer the Slack request.
- **Key configuration:**
  - User text input: `={{ $('Slack Trigger').item.json.text }}`
  - System message enforces:
    - current user name injected from `$json.name`
    - *must only use provided tools*
    - show allowed tools list (`$json.allowed_tools`)
    - `returnIntermediateSteps: true`
- **Inputs/Outputs:**
  - Inputs:
    - Main input: from “Set input” (contains `name`, `allowed_tools`, etc.)
    - `ai_languageModel`: from OpenAI Chat Model
    - `ai_memory`: from Chat Memory
    - `ai_tool`: from “Check permissions” (filtered tools)
  - Output: includes `output` used for Slack reply.
- **Edge cases / failures:**
  - If user asks something that cannot be done by tools, system prompt forces the agent to explain it can’t.
  - If tool stubs are used, agent should respond with authorization denial message (depends on agent compliance).

#### Node: Send output message
- **Type / Role:** `n8n-nodes-base.slack` — Posts agent output back to Slack.
- **Key configuration:**
  - Text: `={{ $json.output }}`
  - Channel: `={{ $('Slack Trigger').item.json.channel }}`
- **Inputs/Outputs:** Input from AI Agent; posts to originating channel.
- **Edge cases / failures:**
  - Missing `output` key → message is blank or expression error.
  - Slack rate limits for high-volume usage.

#### Tool node: calculator
- **Type / Role:** `@n8n/n8n-nodes-langchain.toolCalculator` — Arithmetic tool.
- **Connectivity:** Outputs `ai_tool` → Check permissions.
- **Edge cases:** Minimal; primarily agent/tool calling format issues.

#### Tool node: Wikipedia
- **Type / Role:** `@n8n/n8n-nodes-langchain.toolWikipedia` — Wikipedia lookup tool.
- **Connectivity:** Outputs `ai_tool` → Check permissions.
- **Edge cases:**
  - Network errors / Wikipedia blocking/rate limits.
  - Ambiguous queries.

#### Tool node: Create a bucket in AWS S3
- **Type / Role:** `n8n-nodes-base.awsS3Tool` — Allows agent to create S3 buckets.
- **Key configuration:**
  - Resource: `bucket`, Operation implied by node: create bucket
  - Bucket name from AI: `BucketName` via `$fromAI(...)`
  - Region from AI: `Region` via `$fromAI(...)`
- **Credentials:** AWS IAM credentials.
- **Connectivity:** Outputs `ai_tool` → Check permissions.
- **Edge cases / failures:**
  - Bucket name already taken (global namespace) → error.
  - Region invalid, permission denied (IAM), SCP restrictions → error.
  - Agent may propose unsafe names; consider adding validation.

#### Tool node: Create an incident in PagerDuty
- **Type / Role:** `n8n-nodes-base.pagerDutyTool` — Allows agent to create PagerDuty incidents.
- **Key configuration:**
  - Operation: create incident
  - Service ID: `YOUR_PAGERDUTY_SERVICE_ID`
  - Fields from AI: `Email`, `Title` via `$fromAI(...)`
- **Credentials:** PagerDuty API token.
- **Connectivity:** Outputs `ai_tool` → Check permissions.
- **Edge cases / failures:**
  - Wrong service ID, insufficient API token scopes → error.
  - Agent-generated content may need policy checks (title/description quality).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Slack Trigger | slackTrigger | Slack event entry point (app mention) | — | Get user's slack profile | Listens to messages directly sent to the Slack bot |
| Get user's slack profile | slack | Fetch Slack user profile (email) | Slack Trigger | Get Port access token | Listens to messages directly sent to the Slack bot |
| Get Port access token | httpRequest | Authenticate to Port | Get user's slack profile | Get user permission from Port |  |
| Get user permission from Port | httpRequest | Lookup RBAC user entity in Port | Get Port access token | Unknown user | Checks if the user was found in Port |
| Unknown user | if | Branch if Port user not found | Get user permission from Port | Send a message; Set input | Checks if the user was found in Port |
| Send a message | slack | Notify user missing from Port | Unknown user | — | Checks if the user was found in Port |
| Set input | set | Normalize user data for agent | Unknown user | AI Agent | Collects input and formats it using required keys |
| OpenAI Chat Model | lmChatOpenAi | LLM for agent | — | AI Agent | AI agent with the instruction to always use the connected tools to respond to the user's request |
| Chat Memory | memoryBufferWindow | Per-user short-term memory | Set input | AI Agent | AI agent with the instruction to always use the connected tools to respond to the user's request |
| AI Agent | agent | Tool-using agent responding to Slack | Set input; Check permissions; OpenAI Chat Model; Chat Memory | Send output message | AI agent with the instruction to always use the connected tools to respond to the user's request |
| Send output message | slack | Send agent response to originating channel | AI Agent | — | AI agent with the instruction to always use the connected tools to respond to the user's request |
| Check permissions | langchain.code | Filter tools based on `allowed_tools` | calculator; Wikipedia; Create a bucket in AWS S3; Create an incident in PagerDuty | AI Agent | Uses list of allowed tools gathered from Port to check for permissions and replaces denied tools with a fixed instruction to return a message to the user. |
| calculator | toolCalculator | Arithmetic tool for agent | — | Check permissions | Uses list of allowed tools gathered from Port to check for permissions and replaces denied tools with a fixed instruction to return a message to the user. |
| Wikipedia | toolWikipedia | Wikipedia lookup tool for agent | — | Check permissions | Uses list of allowed tools gathered from Port to check for permissions and replaces denied tools with a fixed instruction to return a message to the user. |
| Create a bucket in AWS S3 | awsS3Tool | S3 bucket creation tool for agent | — | Check permissions | Uses list of allowed tools gathered from Port to check for permissions and replaces denied tools with a fixed instruction to return a message to the user. |
| Create an incident in PagerDuty | pagerDutyTool | PagerDuty incident creation tool for agent | — | Check permissions | Uses list of allowed tools gathered from Port to check for permissions and replaces denied tools with a fixed instruction to return a message to the user. |
| Sticky Note | stickyNote | Documentation / setup notes | — | — | ## AI Agent Access Control (Port + Slack)\n\nThis workflow adds role-based access control to AI agents. Users @mention the bot in Slack, and the workflow checks their permissions in Port before letting the agent use any tools.\n\n### How it works\n1. Slack trigger picks up @mentions and gets the user's email.\n2. Authenticates with Port and looks up the user in the rbacUser blueprint.\n3. If the user exists, reads their allowed_tools array.\n4. The LangChain code node filters tools at runtime, swapping any unauthorized tool with a \"not authorized\" stub.\n5. AI agent runs with only permitted tools, then posts the response back to Slack.\n\n### Setup\n- [ ] Connect your Slack account and set the channel ID.\n- [ ] Add your OpenAI API key.\n- [ ] Get a free Port account at port.io.\n- [ ] Create an rbacUser blueprint in Port with an allowed_tools property (string array).\n- [ ] Add user entities with their email as identifier and allowed tools listed.\n- [ ] Replace the Port client ID and secret in the \"Get Port access token\" node.\n- [ ] Connect any tool credentials you want to use (PagerDuty, AWS, etc.).\n- [ ] Invite the bot to your Slack channel. |
| Sticky Note2 | stickyNote | Comment | — | — | Uses list of allowed tools gathered from Port to check for permissions and replaces denied tools with a fixed instruction to return a message to the user. |
| Sticky Note4 | stickyNote | Comment | — | — | AI agent with the instruction to always use the connected tools to respond to the user's request |
| Sticky Note5 | stickyNote | Comment | — | — | Collects input and formats it using required keys |
| Sticky Note11 | stickyNote | Comment | — | — | Listens to messages directly sent to the Slack bot |
| Sticky Note12 | stickyNote | Comment | — | — | Checks if the user was found in Port |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Access Control for AI Agents (RBAC) using Port and Slack** (or your preferred title).

2. **Add Slack Trigger**
   - Node: **Slack Trigger**
   - Event: `app_mention`
   - Set **Channel ID** to the channel you want to monitor (replace `YOUR_CHANNEL_ID`).
   - **Credentials:** Connect Slack (ensure scopes for events + reading user profile/email).

3. **Add Slack “Get user profile”**
   - Node: **Slack**
   - Resource: `user`
   - Operation: `getProfile`
   - User: expression `{{$json.user}}`
   - Connect: **Slack Trigger → Get user's slack profile**

4. **Add Port token request**
   - Node: **HTTP Request**
   - Method: `POST`
   - URL: `https://api.port.io/v1/auth/access_token`
   - Body: JSON, for example:
     - `clientId`: your Port client id
     - `clientSecret`: your Port client secret
   - Connect: **Get user's slack profile → Get Port access token**

5. **Create/Configure Port Bearer credential**
   - In **Credentials**, create **HTTP Bearer Auth**:
     - Bearer token: your Port API token/access token mechanism.
   - Note: This workflow uses a bearer credential on the *lookup* call. If you want the token returned by step 4 to be used dynamically, modify the lookup node to set `Authorization: Bearer {{$json.accessToken}}` (or the correct field), instead of static credentials.

6. **Add Port RBAC user lookup**
   - Node: **HTTP Request**
   - Method: `GET`
   - URL (expression):  
     `https://api.port.io/v1/blueprints/rbacUser/entities/{{ $('Get user\'s slack profile').item.json.email }}`
   - Authentication: **Generic credential type → HTTP Bearer Auth**
   - Connect: **Get Port access token → Get user permission from Port**

7. **Add IF node to detect unknown users**
   - Node: **IF**
   - Condition: boolean, left value `{{$json.ok}}`, check “is false”.
   - Connect: **Get user permission from Port → Unknown user**

8. **Unknown-user Slack response**
   - Node: **Slack**
   - Operation: post message (send message)
   - Text: `User not found in Port. Please contact your administrator.`
   - Channel: set to your channel (static) or use the trigger’s channel.
   - Connect: **Unknown user (true) → Send a message**

9. **Set node to format input for agent**
   - Node: **Set**
   - Add fields:
     - `name` (string): `{{$json.entity.identifier}}`
     - `granted_roles` (array): `{{$json.entity.relations.roles || []}}`
     - `allowed_tools` (array): `{{$json.entity.properties.allowed_tools || []}}`
   - Connect: **Unknown user (false) → Set input**

10. **Add OpenAI Chat Model**
    - Node: **OpenAI Chat Model (LangChain)**
    - Model: `gpt-4o`
    - **Credentials:** OpenAI API key
    - This node will connect to the agent via `ai_languageModel`.

11. **Add Chat Memory**
    - Node: **Chat Memory (Buffer Window)**
    - Session id type: `customKey`
    - Session key: `{{$json.name}}`
    - Connect: **Set input → Chat Memory** (main)
    - Then connect **Chat Memory → AI Agent** using the `ai_memory` connection.

12. **Add tool nodes you want the agent to potentially use**
    - Add **Calculator Tool** (LangChain)
    - Add **Wikipedia Tool** (LangChain)
    - Add **AWS S3 Tool**
      - Resource: bucket
      - Configure “Bucket name” and “Region” using `$fromAI()` (n8n AI tool parameter feature).
      - **Credentials:** AWS IAM with permission to create buckets.
    - Add **PagerDuty Tool**
      - Operation: create incident
      - Service ID: your PagerDuty service id
      - Fields like Email/Title from `$fromAI()`
      - **Credentials:** PagerDuty API token.

13. **Add “Check permissions” LangChain Code node**
    - Node: **LangChain Code**
    - Configure it to accept `ai_tool` input and output `ai_tool`.
    - Paste logic equivalent to:
      - Read `allowed_tools` from the current input item
      - For each connected tool, keep or replace with denial stub
    - Connect each tool’s `ai_tool` output into **Check permissions** `ai_tool` input.
    - Connect **Check permissions** `ai_tool` output to **AI Agent** `ai_tool` input.

14. **Add AI Agent**
    - Node: **AI Agent (LangChain)**
    - Text: `{{ $('Slack Trigger').item.json.text }}`
    - System message (example aligned with workflow):
      - Include user name: `{{ $json.name }}`
      - Enforce: must only use provided tools; no general knowledge
      - Include allowed tools list: `{{ $json.allowed_tools }}`
    - Enable `returnIntermediateSteps` if you want tool traces.
    - Connections:
      - Main input: **Set input → AI Agent**
      - `ai_languageModel`: **OpenAI Chat Model → AI Agent**
      - `ai_memory`: **Chat Memory → AI Agent**
      - `ai_tool`: **Check permissions → AI Agent**

15. **Add Slack “Send output message”**
    - Node: **Slack**
    - Text: `{{$json.output}}`
    - Channel: `{{ $('Slack Trigger').item.json.channel }}`
    - Connect: **AI Agent → Send output message**

16. **Port data model requirement (outside n8n)**
    - In Port, create blueprint: `rbacUser`
    - Create property: `allowed_tools` as **string array**
    - Create user entities where:
      - **identifier** = user email (must match Slack profile email)
      - `properties.allowed_tools` lists exact tool names as returned by the tools (e.g., `calculator`, `Wikipedia`, etc.; verify actual tool names in your environment)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get a free Port account at port.io. | From the workflow’s embedded notes |
| Create an `rbacUser` blueprint in Port with an `allowed_tools` property (string array). | Port RBAC data model prerequisite |
| Invite the Slack bot to your Slack channel and set the correct Channel ID. | Slack configuration prerequisite |
| Replace Port client ID/secret in “Get Port access token”; connect Slack/OpenAI/AWS/PagerDuty credentials. | Credential setup prerequisite |