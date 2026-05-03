Start an AI coding agent from Linear issues with CloudCLI

https://n8nworkflows.xyz/workflows/start-an-ai-coding-agent-from-linear-issues-with-cloudcli-14071


# Start an AI coding agent from Linear issues with CloudCLI

# 1. Workflow Overview

This workflow automatically launches an AI coding session in a CloudCLI development environment whenever a **new Linear issue** is created, then posts the implementation result and session-access links back to the same Linear issue.

Its primary use case is to turn issue creation in Linear into an automated engineering handoff: a freshly created issue becomes a task for an AI coding agent such as Claude Code, Cursor CLI, or Codex running inside a prepared CloudCLI environment.

## 1.1 Input Reception and Event Filtering
The workflow starts from a **Linear Trigger** listening to issue events. Since Linear can emit multiple issue-related events, an **IF** node ensures only `"create"` actions continue through the workflow.

## 1.2 Prompt Construction
Once a new issue is confirmed, the workflow builds a structured prompt for the AI coding agent. It extracts the issue identifier, title, description, and internal issue ID into reusable fields.

## 1.3 Cloud Environment Resolution and Agent Execution
The workflow then retrieves the selected CloudCLI environment details, including connection metadata and the live access URL. With that environment context, it launches the AI coding agent inside the target CloudCLI environment using the generated prompt.

## 1.4 Result Extraction and Feedback to Linear
After the AI agent completes, the workflow parses the agent output and session metadata, generates deep links for Web UI, VS Code, and Cursor access, and posts a formatted comment back to the originating Linear issue.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and New-Issue Filtering

### Overview
This block receives issue events from Linear and prevents accidental reruns on issue updates. Only newly created issues are allowed to proceed.

### Nodes Involved
- Linear Trigger
- Is New Issue?

### Node Details

#### 1. Linear Trigger
- **Type and technical role:** `n8n-nodes-base.linearTrigger`  
  Event source node that starts the workflow when Linear emits an issue-related webhook event.
- **Configuration choices:**
  - Resource monitored: `issue`
  - Uses a webhook-based trigger
- **Key expressions or variables used:**
  - No custom expression in the node itself
  - Downstream logic expects fields like:
    - `$json.action`
    - `$json.data.id`
    - `$json.data.identifier`
    - `$json.data.title`
    - `$json.data.description`
- **Input and output connections:**
  - No input; this is an entry point
  - Output → `Is New Issue?`
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires a working Linear connection in n8n
- **Edge cases or potential failure types:**
  - Linear credential/authentication failure
  - Webhook registration issues
  - Event payload shape changes
  - Missing `description` or other optional issue fields in payload
- **Sub-workflow reference:**
  - None

#### 2. Is New Issue?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional gate that allows only issue creation events to continue.
- **Configuration choices:**
  - Condition checks whether `{{$json.action}}` equals `"create"`
  - Strict type validation is enabled
  - Case-sensitive string comparison
- **Key expressions or variables used:**
  - `={{ $json.action }}`
- **Input and output connections:**
  - Input ← `Linear Trigger`
  - True output → `Compose Agent Prompt`
  - False output is unused
- **Version-specific requirements:**
  - `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - If Linear sends a different action label than expected, new issues may be filtered out
  - If `action` is missing or null, the comparison fails and the item stops
- **Sub-workflow reference:**
  - None

---

## Block 2 — Prompt Construction

### Overview
This block converts the Linear issue into a prompt suitable for an AI coding agent. It also preserves the issue ID and human-readable identifier for later posting.

### Nodes Involved
- Compose Agent Prompt

### Node Details

#### 3. Compose Agent Prompt
- **Type and technical role:** `n8n-nodes-base.set`  
  Data-shaping node that creates normalized fields used by later nodes.
- **Configuration choices:**
  - Creates three fields:
    - `agentPrompt`
    - `issueId`
    - `issueIdentifier`
  - The prompt contains the issue identifier, title, description, and implementation instructions
- **Key expressions or variables used:**
  - `{{ $json.data.identifier }}`
  - `{{ $json.data.title }}`
  - `{{ $json.data.description }}`
  - `={{ $json.data.id }}`
  - `={{ $json.data.identifier }}`
- **Input and output connections:**
  - Input ← `Is New Issue?`
  - Output → `Get Environment Details`
- **Version-specific requirements:**
  - `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - If `description` is empty, the prompt may contain a blank section
  - If `title` or `identifier` is missing, the prompt quality degrades or expression evaluation may fail
  - Prompt may need customization for repository-specific guidance
- **Sub-workflow reference:**
  - None

---

## Block 3 — Cloud Environment Resolution and Agent Execution

### Overview
This block identifies the target CloudCLI environment and then runs the AI coding agent inside it. It is the execution core of the workflow.

### Nodes Involved
- Get Environment Details
- Run AI Coding Agent

### Node Details

#### 4. Get Environment Details
- **Type and technical role:** `@cloudcli-ai/n8n-nodes-cloud-cli.cloudCli`  
  CloudCLI community node used to retrieve metadata about a selected environment.
- **Configuration choices:**
  - Operation: `get`
  - Environment is selected through the node’s `environmentId` field
  - The configured environment becomes the execution target for the agent
- **Key expressions or variables used:**
  - No dynamic value is currently populated in the JSON export for the environment selector; this must be chosen manually in n8n
- **Input and output connections:**
  - Input ← `Compose Agent Prompt`
  - Output → `Run AI Coding Agent`
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires installation of the verified community node:
    - `@cloudcli-ai/n8n-nodes-cloud-cli`
- **Edge cases or potential failure types:**
  - Community node not installed
  - Missing or invalid CloudCLI API credentials
  - Environment not selected
  - Environment deleted, unavailable, or stopped
  - Returned payload missing expected fields such as `id`, `access_url`, `subdomain`, or `name`
- **Sub-workflow reference:**
  - None

#### 5. Run AI Coding Agent
- **Type and technical role:** `@cloudcli-ai/n8n-nodes-cloud-cli.cloudCli`  
  CloudCLI agent-execution node that sends the composed task prompt to an AI coding agent inside the selected environment.
- **Configuration choices:**
  - Resource: `agent`
  - Message: uses the `agentPrompt` created earlier
  - Agent environment ID: taken from the output of `Get Environment Details`
  - No extra options configured in `additionalOptions`
- **Key expressions or variables used:**
  - `={{ $('Compose Agent Prompt').item.json.agentPrompt }}`
  - `={{ $('Get Environment Details').item.json.id }}`
- **Input and output connections:**
  - Input ← `Get Environment Details`
  - Output → `Extract Agent Result`
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires compatible CloudCLI node and valid credentials
- **Edge cases or potential failure types:**
  - Agent execution timeout
  - Prompt too vague or too long
  - Environment inaccessible or misconfigured
  - Unexpected response structure from agent run
  - API quota/credit issues on CloudCLI side
- **Sub-workflow reference:**
  - None

---

## Block 4 — Result Extraction and Posting Back to Linear

### Overview
This block parses the CloudCLI agent response, derives session continuation links, and writes a status comment to the original Linear issue. It turns raw execution output into reviewer-friendly delivery information.

### Nodes Involved
- Extract Agent Result
- Post Results to Linear

### Node Details

#### 6. Extract Agent Result
- **Type and technical role:** `n8n-nodes-base.set`  
  Transformation node that parses agent events and builds reusable output fields for the Linear comment.
- **Configuration choices:**
  - Extracts:
    - `agentResult`
    - `sessionId`
    - `accessUrl`
    - `vscodeUrl`
    - `cursorUrl`
    - `issueId`
    - `issueIdentifier`
  - Uses array search logic against the CloudCLI event stream
  - Builds VS Code and Cursor deep links from environment metadata
- **Key expressions or variables used:**
  - `={{ $json.events.find(e => e.type === 'claude-response' && e.data && e.data.type === 'result').data.result }}`
  - `={{ $json.events.find(e => e.type === 'session-created').sessionId }}`
  - `={{ $('Get Environment Details').item.json.access_url }}`
  - `=vscode://vscode-remote/ssh-remote+{{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai/workspace/{{ $('Get Environment Details').item.json.name.replace(/[^a-zA-Z0-9-]/g, '') }}?windowId=_blank`
  - `=cursor://vscode-remote/ssh-remote+{{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai/workspace/{{ $('Get Environment Details').item.json.name.replace(/[^a-zA-Z0-9-]/g, '') }}?windowId=_blank`
  - `={{ $('Compose Agent Prompt').item.json.issueId }}`
  - `={{ $('Compose Agent Prompt').item.json.issueIdentifier }}`
- **Input and output connections:**
  - Input ← `Run AI Coding Agent`
  - Output → `Post Results to Linear`
- **Version-specific requirements:**
  - `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - `.find(...)` can return `undefined`; accessing `.data.result` or `.sessionId` would then fail
  - The event type names assume a Claude-style response event (`claude-response`); other agents may return different event structures
  - If environment names sanitize poorly, generated deep links may not open correctly
  - Missing `subdomain`, `access_url`, or `name` fields will break link generation
- **Sub-workflow reference:**
  - None

#### 7. Post Results to Linear
- **Type and technical role:** `n8n-nodes-base.linear`  
  Action node that creates a comment on the originating Linear issue.
- **Configuration choices:**
  - Resource: `comment`
  - `issueId` comes from the extracted issue metadata
  - Comment body includes:
    - Issue identifier
    - Agent result text
    - Web UI link
    - VS Code deep link
    - Cursor deep link
    - SSH resume instructions with session ID
    - Automation signature
- **Key expressions or variables used:**
  - `{{ $json.issueIdentifier }}`
  - `{{ $json.agentResult }}`
  - `{{ $json.accessUrl }}`
  - `{{ $json.vscodeUrl }}`
  - `{{ $json.cursorUrl }}`
  - `{{ $('Get Environment Details').item.json.subdomain }}`
  - `{{ $json.sessionId }}`
  - `={{ $json.issueId }}`
- **Input and output connections:**
  - Input ← `Extract Agent Result`
  - No downstream connection
- **Version-specific requirements:**
  - `typeVersion: 1.1`
  - Requires Linear credentials with permission to post comments
- **Edge cases or potential failure types:**
  - Invalid or expired Linear credentials
  - Comment formatting issues if result text contains unusual Markdown
  - Comment may become excessively long if agent output is very verbose
  - If issue ID is invalid or deleted, comment creation fails
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Linear Trigger | `n8n-nodes-base.linearTrigger` | Receives issue events from Linear |  | Is New Issue? | ## 1. Capture New Linear Issues<br><br>The Linear Trigger fires on all issue events (create and update). The IF node filters for newly created issues only by checking that the action equals "create".<br><br>This prevents the agent from re-running every time someone updates the issue. |
| Is New Issue? | `n8n-nodes-base.if` | Filters for issue creation events only | Linear Trigger | Compose Agent Prompt | ## 1. Capture New Linear Issues<br><br>The Linear Trigger fires on all issue events (create and update). The IF node filters for newly created issues only by checking that the action equals "create".<br><br>This prevents the agent from re-running every time someone updates the issue. |
| Compose Agent Prompt | `n8n-nodes-base.set` | Builds AI prompt and stores issue identifiers | Is New Issue? | Get Environment Details | ## 2. Compose Agent Prompt<br><br>The issue title and description are formatted into a prompt for the AI agent. Customize this to include your project's coding standards, conventions, or any context the agent should know. |
| Get Environment Details | `@cloudcli-ai/n8n-nodes-cloud-cli.cloudCli` | Retrieves selected CloudCLI environment metadata | Compose Agent Prompt | Run AI Coding Agent | ## 3. Get Environment & Run Agent<br>[CloudCLI Docs](https://developer.cloudcli.ai)<br><br>The Get Environment node fetches your environment's details including its live access URL. The agent then runs the task inside that environment's isolated container.<br><br>Select your target environment in the "Get Environment Details" node. The agent auto-detects the project path from the environment name. |
| Run AI Coding Agent | `@cloudcli-ai/n8n-nodes-cloud-cli.cloudCli` | Executes the AI coding task in CloudCLI | Get Environment Details | Extract Agent Result | ## 3. Get Environment & Run Agent<br>[CloudCLI Docs](https://developer.cloudcli.ai)<br><br>The Get Environment node fetches your environment's details including its live access URL. The agent then runs the task inside that environment's isolated container.<br><br>Select your target environment in the "Get Environment Details" node. The agent auto-detects the project path from the environment name. |
| Extract Agent Result | `n8n-nodes-base.set` | Parses CloudCLI output and generates access links | Run AI Coding Agent | Post Results to Linear | ## 4. Post Results to Linear<br><br>The agent's result text, session duration, cost, and multiple ways to continue the session are extracted and posted back to the Linear issue as a comment:<br><br>- **Web UI** link to the CloudCLI dashboard<br>- **VS Code / Cursor** deep links that open the environment directly via Remote SSH<br>- **SSH command** with `claude -r` to resume the agent session from terminal<br><br>Reviewers pick whichever entry point they prefer and continue where the agent left off. |
| Post Results to Linear | `n8n-nodes-base.linear` | Posts completion summary and session links back to Linear | Extract Agent Result |  | ## 4. Post Results to Linear<br><br>The agent's result text, session duration, cost, and multiple ways to continue the session are extracted and posted back to the Linear issue as a comment:<br><br>- **Web UI** link to the CloudCLI dashboard<br>- **VS Code / Cursor** deep links that open the environment directly via Remote SSH<br>- **SSH command** with `claude -r` to resume the agent session from terminal<br><br>Reviewers pick whichever entry point they prefer and continue where the agent left off. |
| Sticky Note - Overview | `n8n-nodes-base.stickyNote` | Canvas documentation for workflow purpose and setup |  |  | ## Start an AI Coding Agent (Claude Code, Gemini or Codex) from Linear Issues with CloudCLI<br><br>### When a Linear issue is created, this workflow sends it to a CloudCLI cloud dev environment where an AI agent handles the implementation, then posts the results and a live session link back to Linear for review.<br><br>### How it works<br>1. A Linear webhook fires when an issue event occurs.<br>2. The workflow filters for newly created issues only.<br>3. The issue title and description are composed into an agent prompt.<br>4. CloudCLI fetches the target environment details (including its live access URL).<br>5. The AI coding agent (Claude Code, Cursor CLI, or Codex) runs the task.<br>6. The agent's output, VS Code/Cursor deep links, SSH resume command, and environment URL are posted back to Linear as a comment.<br><br>### Set up steps<br>1. Install the CloudCLI verified community node from the n8n nodes panel.<br>2. Connect your Linear account credentials.<br>3. Connect your CloudCLI API credentials (get your key at [cloudcli.ai](https://cloudcli.ai)).<br>4. Select which CloudCLI environment to use in the "Get Environment Details" node.<br>5. Customize the prompt template in "Compose Agent Prompt" to fit your codebase.<br><br>### Requirements<br>- Linear account<br>- CloudCLI account with API key ([cloudcli.ai](https://cloudcli.ai))<br>- A running CloudCLI environment with your repo cloned<br><br>### Need Help?<br>[n8n Discord](https://discord.com/invite/XPKeKXeB7d) \| [n8n Forum](https://community.n8n.io/) \| [CloudCLI Docs](https://developer.cloudcli.ai) |
| Sticky Note - Step 1 | `n8n-nodes-base.stickyNote` | Canvas documentation for trigger/filter stage |  |  | ## 1. Capture New Linear Issues<br><br>The Linear Trigger fires on all issue events (create and update). The IF node filters for newly created issues only by checking that the action equals "create".<br><br>This prevents the agent from re-running every time someone updates the issue. |
| Sticky Note - Step 2 | `n8n-nodes-base.stickyNote` | Canvas documentation for prompt composition |  |  | ## 2. Compose Agent Prompt<br><br>The issue title and description are formatted into a prompt for the AI agent. Customize this to include your project's coding standards, conventions, or any context the agent should know. |
| Sticky Note - Step 3 | `n8n-nodes-base.stickyNote` | Canvas documentation for environment retrieval and agent execution |  |  | ## 3. Get Environment & Run Agent<br>[CloudCLI Docs](https://developer.cloudcli.ai)<br><br>The Get Environment node fetches your environment's details including its live access URL. The agent then runs the task inside that environment's isolated container.<br><br>Select your target environment in the "Get Environment Details" node. The agent auto-detects the project path from the environment name. |
| Sticky Note - Step 4 | `n8n-nodes-base.stickyNote` | Canvas documentation for Linear feedback stage |  |  | ## 4. Post Results to Linear<br><br>The agent's result text, session duration, cost, and multiple ways to continue the session are extracted and posted back to the Linear issue as a comment:<br><br>- **Web UI** link to the CloudCLI dashboard<br>- **VS Code / Cursor** deep links that open the environment directly via Remote SSH<br>- **SSH command** with `claude -r` to resume the agent session from terminal<br><br>Reviewers pick whichever entry point they prefer and continue where the agent left off. |
| Sticky Note - Community Node | `n8n-nodes-base.stickyNote` | Canvas documentation for required CloudCLI community node |  |  | ### Community Node<br>This workflow uses the **CloudCLI** verified community node (`@cloudcli-ai/n8n-nodes-cloud-cli`). Install it from the n8n nodes panel. [Learn more](https://docs.n8n.io/integrations/community-nodes/installation/verified-install/). |

---

# 4. Reproducing the Workflow from Scratch

1. **Install the required community node**
   - In n8n, open the nodes panel or community node management area.
   - Install the verified community package:
     - `@cloudcli-ai/n8n-nodes-cloud-cli`
   - Confirm the CloudCLI node appears in your node list.

2. **Prepare credentials**
   - Create or connect **Linear credentials** in n8n.
   - Create or connect **CloudCLI API credentials** using your API key from [cloudcli.ai](https://cloudcli.ai).
   - Ensure the Linear credential can:
     - receive issue events
     - create comments on issues
   - Ensure the CloudCLI credential can:
     - fetch environment details
     - launch agent sessions

3. **Create the trigger node**
   - Add a node: **Linear Trigger**
   - Node type: `n8n-nodes-base.linearTrigger`
   - Set the resource to:
     - `issue`
   - Attach your Linear credentials.
   - If prompted, register or activate the webhook.

4. **Add the event filter**
   - Add an **IF** node named **Is New Issue?**
   - Connect:
     - `Linear Trigger` → `Is New Issue?`
   - Configure one condition:
     - Left value: `={{ $json.action }}`
     - Operator: `equals`
     - Right value: `create`
   - Keep case sensitivity enabled as in the exported workflow.

5. **Add the prompt-building node**
   - Add a **Set** node named **Compose Agent Prompt**
   - Connect:
     - `Is New Issue?` true output → `Compose Agent Prompt`
   - Add these fields:
     - `agentPrompt` as string:
       ```text
       You are working on Linear issue {{ $json.data.identifier }}.

       Task: {{ $json.data.title }}

       Details:
       {{ $json.data.description }}

       Implement the changes described above. Write clean, well-documented code. When finished, provide a summary of what was changed and any notes for the reviewer.
       ```
     - `issueId` as string:
       - `={{ $json.data.id }}`
     - `issueIdentifier` as string:
       - `={{ $json.data.identifier }}`
   - Leave other options at default unless you want to retain or remove additional fields.

6. **Add the environment lookup**
   - Add a **CloudCLI** node named **Get Environment Details**
   - Connect:
     - `Compose Agent Prompt` → `Get Environment Details`
   - Configure:
     - Operation: `get`
   - Select the target CloudCLI environment in the `environmentId` field.
   - Attach your CloudCLI credentials.
   - This environment should already be running and should contain the cloned repository the AI agent will modify.

7. **Add the agent execution node**
   - Add another **CloudCLI** node named **Run AI Coding Agent**
   - Connect:
     - `Get Environment Details` → `Run AI Coding Agent`
   - Configure:
     - Resource: `agent`
     - Message:
       - `={{ $('Compose Agent Prompt').item.json.agentPrompt }}`
     - Agent Environment ID:
       - `={{ $('Get Environment Details').item.json.id }}`
   - Keep `additionalOptions` empty unless you want to tune agent behavior.

8. **Add the result extraction node**
   - Add a **Set** node named **Extract Agent Result**
   - Connect:
     - `Run AI Coding Agent` → `Extract Agent Result`
   - Add these fields:

   - `agentResult`:
     - `={{ $json.events.find(e => e.type === 'claude-response' && e.data && e.data.type === 'result').data.result }}`

   - `sessionId`:
     - `={{ $json.events.find(e => e.type === 'session-created').sessionId }}`

   - `accessUrl`:
     - `={{ $('Get Environment Details').item.json.access_url }}`

   - `vscodeUrl`:
     - `=vscode://vscode-remote/ssh-remote+{{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai/workspace/{{ $('Get Environment Details').item.json.name.replace(/[^a-zA-Z0-9-]/g, '') }}?windowId=_blank`

   - `cursorUrl`:
     - `=cursor://vscode-remote/ssh-remote+{{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai/workspace/{{ $('Get Environment Details').item.json.name.replace(/[^a-zA-Z0-9-]/g, '') }}?windowId=_blank`

   - `issueId`:
     - `={{ $('Compose Agent Prompt').item.json.issueId }}`

   - `issueIdentifier`:
     - `={{ $('Compose Agent Prompt').item.json.issueIdentifier }}`

9. **Add the Linear comment node**
   - Add a **Linear** node named **Post Results to Linear**
   - Connect:
     - `Extract Agent Result` → `Post Results to Linear`
   - Configure:
     - Resource: `comment`
     - Issue ID:
       - `={{ $json.issueId }}`
     - Comment:
       ```markdown
       **CloudCLI agent completed work on {{ $json.issueIdentifier }}**

       {{ $json.agentResult }}

       ---

       **Continue this session:**
       - Web UI: [Open in CloudCLI]({{ $json.accessUrl }})
       - VS Code: [Open in VS Code]({{ $json.vscodeUrl }})
       - Cursor: [Open in Cursor]({{ $json.cursorUrl }})
       - SSH + resume: `ssh {{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai` then run `claude -r` to resume session `{{ $json.sessionId }}`

       *Automated by n8n + CloudCLI*
       ```
   - Attach your Linear credentials.

10. **Test with a new Linear issue**
    - Activate the workflow.
    - Create a new issue in Linear.
    - Confirm the execution path is:
      - `Linear Trigger` → `Is New Issue?` → `Compose Agent Prompt` → `Get Environment Details` → `Run AI Coding Agent` → `Extract Agent Result` → `Post Results to Linear`
    - Verify a comment is added to the issue with:
      - agent result text
      - CloudCLI web link
      - VS Code link
      - Cursor link
      - SSH resume command

11. **Recommended hardening before production**
    - Add fallback handling if `description` is empty.
    - Add validation if `events` does not contain:
      - `claude-response`
      - `session-created`
    - Consider a fallback branch for failed agent runs.
    - Consider truncating or summarizing long AI output before posting to Linear comments.
    - If using a non-Claude agent, adjust the event parsing in **Extract Agent Result** to match that agent’s response format.

## Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does **not** expose multiple workflow entry points beyond the single **Linear Trigger**.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| CloudCLI API key is obtained from CloudCLI | [https://cloudcli.ai](https://cloudcli.ai) |
| CloudCLI developer documentation | [https://developer.cloudcli.ai](https://developer.cloudcli.ai) |
| n8n verified community node installation guidance for CloudCLI node | [https://docs.n8n.io/integrations/community-nodes/installation/verified-install/](https://docs.n8n.io/integrations/community-nodes/installation/verified-install/) |
| n8n Discord | [https://discord.com/invite/XPKeKXeB7d](https://discord.com/invite/XPKeKXeB7d) |
| n8n Forum | [https://community.n8n.io/](https://community.n8n.io/) |
| Workflow intent: when a Linear issue is created, send it to a CloudCLI dev environment where an AI agent implements the task and returns results plus session links to Linear | From the overview sticky note |
| Supported/mentioned agent options in the canvas notes: Claude Code, Gemini, Codex, and Cursor CLI | From the overview and Step 3 notes |
| Operational requirement: the CloudCLI environment should already be running and have the repository cloned | From the overview sticky note |