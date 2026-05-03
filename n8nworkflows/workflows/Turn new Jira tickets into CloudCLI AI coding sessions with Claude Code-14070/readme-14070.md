Turn new Jira tickets into CloudCLI AI coding sessions with Claude Code

https://n8nworkflows.xyz/workflows/turn-new-jira-tickets-into-cloudcli-ai-coding-sessions-with-claude-code-14070


# Turn new Jira tickets into CloudCLI AI coding sessions with Claude Code

# 1. Workflow Overview

This workflow listens for newly created Jira issues, converts each issue into an AI coding task, runs that task inside a CloudCLI development environment, and posts the resulting implementation summary plus session continuation links back into Jira.

Its main use case is automated engineering handoff: a newly opened Jira ticket immediately becomes a CloudCLI AI coding session powered by Claude Code, with enough output and session metadata for a developer or reviewer to continue work manually in the browser, VS Code, Cursor, or SSH.

## 1.1 Input Reception: Jira Issue Capture
The workflow starts from a Jira trigger configured for issue creation events. It extracts the issue key, summary, and description into simplified fields for downstream use.

## 1.2 Prompt Construction
The extracted Jira fields are transformed into a structured agent prompt. This prompt tells the AI coding agent what ticket it is addressing, what the task is, and what quality/output expectations to follow.

## 1.3 Cloud Environment Resolution and Agent Execution
The workflow fetches details for a selected CloudCLI environment, then sends the generated prompt to a CloudCLI agent execution node using the Claude provider. The environment metadata is reused later to build access links.

## 1.4 Result Extraction and Jira Feedback
After the CloudCLI agent finishes, the workflow parses the returned event stream to extract the final result text and session ID. It then builds multiple continuation links and posts everything back to the originating Jira ticket as a formatted comment.

## 1.5 Documentation / Visual Guidance Layer
Several sticky notes document setup, requirements, and each step of the process. These notes do not affect runtime execution but are important for maintainers reproducing or modifying the workflow.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block receives the Jira webhook event when a new issue is created and normalizes the payload into a smaller set of fields. It isolates the workflow from Jira’s more complex event structure and makes downstream expressions simpler.

### Nodes Involved
- New Jira Issue Created
- Extract Ticket Details

### Node Details

#### 1. New Jira Issue Created
- **Type and technical role:** `n8n-nodes-base.jiraTrigger`  
  Entry-point trigger node that listens for Jira Cloud issue creation events.
- **Configuration choices:**
  - Event selected: `jira:issue_created`
  - No additional fields configured
  - Uses a Jira webhook internally
- **Key expressions or variables used:** None in parameters
- **Input and output connections:**
  - Input: none, this is a trigger
  - Output: sends Jira event payload to **Extract Ticket Details**
- **Version-specific requirements:**
  - Node type version: `1.1`
  - Requires Jira Trigger support in the n8n version being used
- **Edge cases or potential failure types:**
  - Jira credential or webhook registration failures
  - Missing permissions on the Jira project
  - Event payload shape differing across Jira configurations
  - If the workflow is inactive, webhook delivery may not behave as expected depending on setup
- **Sub-workflow reference:** None

#### 2. Extract Ticket Details
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps selected values from the Jira payload into explicit fields for reuse.
- **Configuration choices:**
  - Creates:
    - `ticketKey` from `issue.key`
    - `ticketSummary` from `issue.fields.summary`
    - `ticketDescription` from `issue.fields.description`
- **Key expressions or variables used:**
  - `{{ $json.issue.key }}`
  - `{{ $json.issue.fields.summary }}`
  - `{{ $json.issue.fields.description }}`
- **Input and output connections:**
  - Input: **New Jira Issue Created**
  - Output: **Compose Agent Prompt**
- **Version-specific requirements:**
  - Node type version: `3.4`
  - Uses assignment-based Set node format available in newer n8n versions
- **Edge cases or potential failure types:**
  - `description` may not always be a plain string in Jira Cloud; some Jira setups return Atlassian Document Format objects
  - Null or empty summary/description values may produce weak prompts
  - Expression resolution fails if `issue` or `fields` is missing from the event
- **Sub-workflow reference:** None

---

## Block 2 — Prompt Construction

### Overview
This block turns the normalized Jira issue data into a single natural-language instruction for the coding agent. It also carries forward the ticket key so the workflow can later post back to the correct Jira issue.

### Nodes Involved
- Compose Agent Prompt

### Node Details

#### 3. Compose Agent Prompt
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds the AI task prompt and preserves the Jira ticket key.
- **Configuration choices:**
  - Creates `agentPrompt` as a multiline string containing:
    - Jira ticket key
    - Task summary
    - Detailed description
    - Instructions to implement changes cleanly and provide reviewer notes
  - Reassigns `ticketKey` from upstream for downstream comment posting
- **Key expressions or variables used:**
  - `{{ $json.ticketKey }}`
  - `{{ $json.ticketSummary }}`
  - `{{ $json.ticketDescription }}`
- **Input and output connections:**
  - Input: **Extract Ticket Details**
  - Output: **Get Environment Details**
- **Version-specific requirements:**
  - Node type version: `3.4`
- **Edge cases or potential failure types:**
  - If `ticketDescription` is structured JSON instead of plain text, the resulting prompt may contain `[object Object]` or malformed content
  - Very large descriptions may exceed prompt or provider limits
  - Missing fields produce vague prompts and lower-quality agent output
- **Sub-workflow reference:** None

---

## Block 3 — Cloud Environment Resolution and Agent Execution

### Overview
This block retrieves metadata for a selected CloudCLI environment and then runs the AI coding task inside that environment. It is the operational core of the workflow and depends on valid CloudCLI credentials plus a prepared remote workspace.

### Nodes Involved
- Get Environment Details
- Run AI Coding Agent

### Node Details

#### 4. Get Environment Details
- **Type and technical role:** `@cloudcli-ai/n8n-nodes-cloud-cli.cloudCli`  
  CloudCLI community node used to fetch a single environment record and its metadata.
- **Configuration choices:**
  - Resource: `environment`
  - Operation: `get`
  - `environmentId` selected from the UI list; currently the exported JSON shows an empty selected value, so this must be configured manually
- **Key expressions or variables used:**
  - No expression in the configured environment selector value as exported
- **Input and output connections:**
  - Input: **Compose Agent Prompt**
  - Output: **Run AI Coding Agent**
- **Version-specific requirements:**
  - Node type version: `1`
  - Requires installation of the verified CloudCLI community node package: `@cloudcli-ai/n8n-nodes-cloud-cli`
- **Edge cases or potential failure types:**
  - Community node not installed
  - CloudCLI credentials missing or invalid
  - No environment selected
  - Selected environment deleted, stopped, or inaccessible
  - API/network failures
- **Sub-workflow reference:** None

#### 5. Run AI Coding Agent
- **Type and technical role:** `@cloudcli-ai/n8n-nodes-cloud-cli.cloudCli`  
  Executes an AI coding agent task within the specified CloudCLI environment.
- **Configuration choices:**
  - Resource: `agent`
  - Operation: `execute`
  - Provider: `claude`
  - Message comes from the previously composed prompt
  - Environment ID is dynamically taken from **Get Environment Details**
  - No additional options configured
- **Key expressions or variables used:**
  - `{{ $('Compose Agent Prompt').item.json.agentPrompt }}`
  - `{{ $('Get Environment Details').item.json.id }}`
- **Input and output connections:**
  - Input: **Get Environment Details**
  - Output: **Extract Agent Result**
- **Version-specific requirements:**
  - Node type version: `1`
  - Depends on the CloudCLI node package and CloudCLI API compatibility
- **Edge cases or potential failure types:**
  - Provider unavailable or unsupported in the account/environment
  - Timeout during agent execution
  - Empty or malformed prompt
  - Environment ID lookup failure
  - Agent event payload shape changing between CloudCLI versions
  - Cost or quota restrictions on the CloudCLI side
- **Sub-workflow reference:** None

---

## Block 4 — Result Extraction and Jira Feedback

### Overview
This block parses the CloudCLI agent execution output, derives session continuation links, and publishes a formatted Jira comment. It is where most payload assumptions live, so changes in CloudCLI response shape or environment metadata can affect it.

### Nodes Involved
- Extract Agent Result
- Post Results to Jira

### Node Details

#### 6. Extract Agent Result
- **Type and technical role:** `n8n-nodes-base.set`  
  Reads the agent response event list, extracts the useful artifacts, and constructs deep links for follow-up access.
- **Configuration choices:**
  - Creates `agentResult` by searching `events` for a `claude-response` event whose nested `data.type` is `result`
  - Creates `sessionId` by searching `events` for a `session-created` event
  - Copies `accessUrl` from **Get Environment Details**
  - Builds `vscodeUrl` using the CloudCLI environment subdomain and a sanitized environment name
  - Builds `cursorUrl` similarly
  - Restores `ticketKey` from **Compose Agent Prompt**
- **Key expressions or variables used:**
  - `{{ $json.events.find(e => e.type === 'claude-response' && e.data && e.data.type === 'result').data.result }}`
  - `{{ $json.events.find(e => e.type === 'session-created').sessionId }}`
  - `{{ $('Get Environment Details').item.json.access_url }}`
  - `{{ $('Get Environment Details').item.json.subdomain }}`
  - `{{ $('Get Environment Details').item.json.name.replace(/[^a-zA-Z0-9-]/g, '') }}`
  - `{{ $('Compose Agent Prompt').item.json.ticketKey }}`
- **Input and output connections:**
  - Input: **Run AI Coding Agent**
  - Output: **Post Results to Jira**
- **Version-specific requirements:**
  - Node type version: `3.4`
- **Edge cases or potential failure types:**
  - If `events` is missing or not an array, `.find(...)` fails
  - If no matching `claude-response` result event exists, attempting `.data.result` on `undefined` causes an expression error
  - If no `session-created` event exists, session ID extraction fails
  - Missing `subdomain`, `name`, or `access_url` from environment details breaks generated links
  - Sanitizing the environment name may produce a path that does not match the actual workspace path
- **Sub-workflow reference:** None

#### 7. Post Results to Jira
- **Type and technical role:** `n8n-nodes-base.jira`  
  Adds a comment to the original Jira issue containing the AI result and continuation paths.
- **Configuration choices:**
  - Resource: `issueComment`
  - Operation: `add`
  - `issueKey` from the extracted `ticketKey`
  - Comment uses Jira wiki markup and includes:
    - Completion heading
    - Agent result text
    - CloudCLI Web UI link
    - VS Code deep link
    - Cursor deep link
    - SSH command with `claude -r` resume guidance
    - Automation signature
  - `wikiMarkup` enabled
- **Key expressions or variables used:**
  - `{{ $json.ticketKey }}`
  - `{{ $json.agentResult }}`
  - `{{ $json.accessUrl }}`
  - `{{ $json.vscodeUrl }}`
  - `{{ $json.cursorUrl }}`
  - `{{ $('Get Environment Details').item.json.subdomain }}`
  - `{{ $json.sessionId }}`
- **Input and output connections:**
  - Input: **Extract Agent Result**
  - Output: none
- **Version-specific requirements:**
  - Node type version: `1`
- **Edge cases or potential failure types:**
  - Jira authentication or permission errors when adding comments
  - Jira issue key invalid or not visible to the credentialed user
  - Wiki markup rendering differences across Jira Cloud configurations
  - Excessively long agent results may exceed Jira comment size expectations
  - Deep links may not open correctly if clients are not installed locally
- **Sub-workflow reference:** None

---

## Block 5 — Documentation and Visual Guidance

### Overview
This workflow includes multiple sticky notes that describe the architecture, setup, requirements, and each processing stage. They do not execute but are important contextual metadata for maintainers and anyone recreating the workflow.

### Nodes Involved
- Sticky Note - Overview
- Sticky Note - Step 1
- Sticky Note - Step 2
- Sticky Note - Step 3
- Sticky Note - Step 4
- Sticky Note - Community Node

### Node Details

#### 8. Sticky Note - Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Provides high-level explanation, setup checklist, requirements, and support links.
- **Configuration choices:**
  - Large note positioned at the top-left of the canvas
  - Includes CloudCLI account/setup guidance and external links
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### 9. Sticky Note - Step 1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the Jira trigger stage
  - Reminds user to update webhook events for the target project
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### 10. Sticky Note - Step 2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the prompt composition stage
  - Encourages customization with project conventions and coding standards
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### 11. Sticky Note - Step 3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents environment lookup and CloudCLI agent execution
  - Includes link to CloudCLI Docs: `https://developer.cloudcli.ai`
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### 12. Sticky Note - Step 4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents Jira feedback stage
  - Explains the Web UI, VS Code/Cursor deep links, and SSH resume path
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

#### 13. Sticky Note - Community Node
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Warns that the workflow depends on a verified community node
  - Includes installation link for community nodes
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Canvas documentation and setup guidance |  |  | ## Turn New Jira Tickets into Claude Code, Gemini, Cursor or Codex Sessions with CloudCLI<br>### When a Jira ticket is created, this workflow sends it to a CloudCLI cloud dev environment where an AI agent handles the implementation, then posts the results and a live session link back to Jira for review.<br>### How it works<br>1. A Jira webhook fires when a new issue is created in your project.<br>2. The ticket summary and description are composed into an agent prompt.<br>3. CloudCLI fetches the target environment details (including its live access URL).<br>4. The AI coding agent (Claude Code, Cursor CLI, or Codex) runs the task.<br>5. The agent's output, VS Code/Cursor deep links, SSH resume command, and environment URL are posted back to Jira as a comment so reviewers can pick up exactly where the agent left off.<br>### Set up steps<br>1. Install the CloudCLI verified community node from the n8n nodes panel.<br>2. Connect your Jira Cloud credentials.<br>3. Connect your CloudCLI API credentials (get your key at [cloudcli.ai](https://cloudcli.ai)).<br>4. Select which CloudCLI environment to use in the "Get Environment Details" node.<br>5. Customize the prompt template in "Compose Agent Prompt" to fit your codebase.<br>### Requirements<br>- Jira Cloud account<br>- CloudCLI account with API key ([cloudcli.ai](https://cloudcli.ai))<br>- A running CloudCLI environment with your repo cloned<br>### Need Help?<br>[n8n Discord](https://discord.com/invite/XPKeKXeB7d) \| [n8n Forum](https://community.n8n.io/) \| [CloudCLI Docs](https://developer.cloudcli.ai) |
| Sticky Note - Step 1 | n8n-nodes-base.stickyNote | Documents the Jira capture stage |  |  | ## 1. Capture New Jira Ticket<br>The workflow triggers when a new issue is created in your Jira project. The ticket key, summary, and description are extracted for use as the agent's task.<br>Update the webhook events in the Jira Trigger to match your project. |
| Sticky Note - Step 2 | n8n-nodes-base.stickyNote | Documents prompt preparation |  |  | ## 2. Compose Agent Prompt<br>The ticket details are formatted into a prompt for the AI agent. Customize this to include your project's coding standards, conventions, or any context the agent should know. |
| Sticky Note - Step 3 | n8n-nodes-base.stickyNote | Documents CloudCLI environment lookup and execution |  |  | ## 3. Get Environment & Run Agent<br>[CloudCLI Docs](https://developer.cloudcli.ai)<br>The Get Environment node fetches your environment's details including its live access URL. The agent then runs the task inside that environment's isolated container.<br>Select your target environment in the "Get Environment Details" node. The agent auto-detects the project path from the environment name. |
| Sticky Note - Step 4 | n8n-nodes-base.stickyNote | Documents result posting to Jira |  |  | ## 4. Post Results to Jira<br>The agent's result text, session duration, cost, and multiple ways to continue the session are extracted and posted back to the Jira ticket as a comment:<br>- **Web UI** link to the CloudCLI dashboard<br>- **VS Code / Cursor** deep links that open the environment directly via Remote SSH<br>- **SSH command** with `claude -r` to resume the agent session from terminal<br>Reviewers pick whichever entry point they prefer and continue where the agent left off. |
| Sticky Note - Community Node | n8n-nodes-base.stickyNote | Notes dependency on CloudCLI verified community node |  |  | ### Community Node<br>This workflow uses the **CloudCLI** verified community node (`@cloudcli-ai/n8n-nodes-cloud-cli`). Install it from the n8n nodes panel. [Learn more](https://docs.n8n.io/integrations/community-nodes/installation/verified-install/). |
| New Jira Issue Created | n8n-nodes-base.jiraTrigger | Triggers on new Jira issue creation |  | Extract Ticket Details | ## 1. Capture New Jira Ticket<br>The workflow triggers when a new issue is created in your Jira project. The ticket key, summary, and description are extracted for use as the agent's task.<br>Update the webhook events in the Jira Trigger to match your project. |
| Extract Ticket Details | n8n-nodes-base.set | Extracts key Jira fields into simple variables | New Jira Issue Created | Compose Agent Prompt | ## 1. Capture New Jira Ticket<br>The workflow triggers when a new issue is created in your Jira project. The ticket key, summary, and description are extracted for use as the agent's task.<br>Update the webhook events in the Jira Trigger to match your project. |
| Compose Agent Prompt | n8n-nodes-base.set | Builds the AI coding instruction text | Extract Ticket Details | Get Environment Details | ## 2. Compose Agent Prompt<br>The ticket details are formatted into a prompt for the AI agent. Customize this to include your project's coding standards, conventions, or any context the agent should know. |
| Get Environment Details | @cloudcli-ai/n8n-nodes-cloud-cli.cloudCli | Fetches selected CloudCLI environment metadata | Compose Agent Prompt | Run AI Coding Agent | ## 3. Get Environment & Run Agent<br>[CloudCLI Docs](https://developer.cloudcli.ai)<br>The Get Environment node fetches your environment's details including its live access URL. The agent then runs the task inside that environment's isolated container.<br>Select your target environment in the "Get Environment Details" node. The agent auto-detects the project path from the environment name. |
| Run AI Coding Agent | @cloudcli-ai/n8n-nodes-cloud-cli.cloudCli | Executes the Claude coding task in CloudCLI | Get Environment Details | Extract Agent Result | ## 3. Get Environment & Run Agent<br>[CloudCLI Docs](https://developer.cloudcli.ai)<br>The Get Environment node fetches your environment's details including its live access URL. The agent then runs the task inside that environment's isolated container.<br>Select your target environment in the "Get Environment Details" node. The agent auto-detects the project path from the environment name. |
| Extract Agent Result | n8n-nodes-base.set | Parses CloudCLI output and builds continuation links | Run AI Coding Agent | Post Results to Jira | ## 4. Post Results to Jira<br>The agent's result text, session duration, cost, and multiple ways to continue the session are extracted and posted back to the Jira ticket as a comment:<br>- **Web UI** link to the CloudCLI dashboard<br>- **VS Code / Cursor** deep links that open the environment directly via Remote SSH<br>- **SSH command** with `claude -r` to resume the agent session from terminal<br>Reviewers pick whichever entry point they prefer and continue where the agent left off. |
| Post Results to Jira | n8n-nodes-base.jira | Adds the agent result as a Jira issue comment | Extract Agent Result |  | ## 4. Post Results to Jira<br>The agent's result text, session duration, cost, and multiple ways to continue the session are extracted and posted back to the Jira ticket as a comment:<br>- **Web UI** link to the CloudCLI dashboard<br>- **VS Code / Cursor** deep links that open the environment directly via Remote SSH<br>- **SSH command** with `claude -r` to resume the agent session from terminal<br>Reviewers pick whichever entry point they prefer and continue where the agent left off. |

---

# 4. Reproducing the Workflow from Scratch

1. **Install the required community node**
   - In n8n, open the nodes panel or community nodes administration area.
   - Install the verified package:
     - `@cloudcli-ai/n8n-nodes-cloud-cli`
   - Confirm the CloudCLI node appears in the node picker.

2. **Prepare credentials**
   - Create or connect **Jira Cloud credentials** for both:
     - the Jira Trigger node
     - the Jira node used for posting comments
   - Create **CloudCLI API credentials** using your API key from [https://cloudcli.ai](https://cloudcli.ai).
   - Ensure the selected Jira user can:
     - receive issue events
     - comment on issues in the target project
   - Ensure the CloudCLI account has access to the target environment.

3. **Create the trigger node**
   - Add a **Jira Trigger** node.
   - Name it: **New Jira Issue Created**
   - Configure:
     - Event: `jira:issue_created`
   - Attach Jira Cloud credentials.
   - If needed, adjust the trigger event list to match your Jira use case.
   - This node is the workflow entry point.

4. **Add the ticket extraction node**
   - Add a **Set** node after the trigger.
   - Name it: **Extract Ticket Details**
   - Create these string fields:
     1. `ticketKey` = `{{ $json.issue.key }}`
     2. `ticketSummary` = `{{ $json.issue.fields.summary }}`
     3. `ticketDescription` = `{{ $json.issue.fields.description }}`
   - Connect:
     - **New Jira Issue Created** → **Extract Ticket Details**

5. **Add the prompt composition node**
   - Add another **Set** node.
   - Name it: **Compose Agent Prompt**
   - Create these string fields:
     1. `agentPrompt`
        - Set it to:
          ```text
          You are working on Jira ticket {{ $json.ticketKey }}.

          Task: {{ $json.ticketSummary }}

          Details:
          {{ $json.ticketDescription }}

          Implement the changes described above. Write clean, well-documented code. When finished, provide a summary of what was changed and any notes for the reviewer.
          ```
     2. `ticketKey` = `{{ $json.ticketKey }}`
   - Connect:
     - **Extract Ticket Details** → **Compose Agent Prompt**

6. **Add the CloudCLI environment lookup node**
   - Add a **CloudCLI** node.
   - Name it: **Get Environment Details**
   - Configure:
     - Resource: `environment`
     - Operation: `get`
     - Environment: choose your target CloudCLI environment from the selector
   - Attach CloudCLI credentials.
   - Important:
     - The exported workflow contains no environment selected, so this must be set manually.
     - The chosen environment should already be running and contain the repository to work on.
   - Connect:
     - **Compose Agent Prompt** → **Get Environment Details**

7. **Add the CloudCLI agent execution node**
   - Add another **CloudCLI** node.
   - Name it: **Run AI Coding Agent**
   - Configure:
     - Resource: `agent`
     - Operation: `execute`
     - Provider: `claude`
     - Message:
       `{{ $('Compose Agent Prompt').item.json.agentPrompt }}`
     - Agent Environment ID:
       `{{ $('Get Environment Details').item.json.id }}`
     - Additional Options: leave default/empty unless you need custom agent behavior
   - Attach the same CloudCLI credentials.
   - Connect:
     - **Get Environment Details** → **Run AI Coding Agent**

8. **Add the result extraction node**
   - Add a **Set** node.
   - Name it: **Extract Agent Result**
   - Create these string fields:
     1. `agentResult`  
        `{{ $json.events.find(e => e.type === 'claude-response' && e.data && e.data.type === 'result').data.result }}`
     2. `sessionId`  
        `{{ $json.events.find(e => e.type === 'session-created').sessionId }}`
     3. `accessUrl`  
        `{{ $('Get Environment Details').item.json.access_url }}`
     4. `vscodeUrl`  
        `vscode://vscode-remote/ssh-remote+{{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai/workspace/{{ $('Get Environment Details').item.json.name.replace(/[^a-zA-Z0-9-]/g, '') }}?windowId=_blank`
     5. `cursorUrl`  
        `cursor://vscode-remote/ssh-remote+{{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai/workspace/{{ $('Get Environment Details').item.json.name.replace(/[^a-zA-Z0-9-]/g, '') }}?windowId=_blank`
     6. `ticketKey`  
        `{{ $('Compose Agent Prompt').item.json.ticketKey }}`
   - Connect:
     - **Run AI Coding Agent** → **Extract Agent Result**

9. **Add the Jira comment node**
   - Add a **Jira** node.
   - Name it: **Post Results to Jira**
   - Configure:
     - Resource: `issueComment`
     - Operation: `add`
     - Issue Key:
       `{{ $json.ticketKey }}`
     - Enable **Wiki Markup**
     - Comment:
       ```text
       *CloudCLI agent completed work on {{ $json.ticketKey }}*

       {{ $json.agentResult }}

       ----

       *Continue this session:*
       * Web UI: [Open in CloudCLI|{{ $json.accessUrl }}]
       * VS Code: [Open in VS Code|{{ $json.vscodeUrl }}]
       * Cursor: [Open in Cursor|{{ $json.cursorUrl }}]
       * SSH + resume: ssh {{ $('Get Environment Details').item.json.subdomain }}@ssh.cloudcli.ai then run claude -r to resume session {{ $json.sessionId }}

       _Automated by n8n + CloudCLI_
       ```
   - Attach Jira Cloud credentials.
   - Connect:
     - **Extract Agent Result** → **Post Results to Jira**

10. **Optionally add the documentation sticky notes**
    - Add sticky notes to match the original workflow’s visual organization:
      - Overview and setup note
      - Step 1 note for Jira capture
      - Step 2 note for prompt composition
      - Step 3 note for environment and agent execution
      - Step 4 note for Jira result posting
      - Community node dependency note
    - These are optional for execution but useful for maintainability.

11. **Test the trigger and payload shape**
    - Activate or test the Jira Trigger.
    - Create a new issue in the target Jira project.
    - Confirm the trigger payload contains:
      - `issue.key`
      - `issue.fields.summary`
      - `issue.fields.description`
    - If `description` is an object instead of plain text, insert a transformation node before prompt composition to convert it to text.

12. **Validate the CloudCLI response structure**
    - Run the workflow with a test issue.
    - Inspect **Run AI Coding Agent** output.
    - Confirm:
      - `events` exists
      - there is a `session-created` event
      - there is a `claude-response` event whose `data.type` is `result`
    - If not, adjust the extraction expressions in **Extract Agent Result**.

13. **Activate the workflow**
    - Once the trigger, CloudCLI environment selection, credentials, and Jira posting are verified, activate the workflow for production use.

## Rebuild Constraints and Important Implementation Notes
- The workflow has **one entry point**: **New Jira Issue Created**
- There are **no sub-workflows**
- The workflow depends on **cross-node expressions** using `$('Node Name').item.json...`
- The CloudCLI environment must be selected manually; the export does not contain a usable environment ID
- The workflow assumes the agent provider is **Claude**
- The workflow assumes the CloudCLI execution output includes an `events` array with specific event types

## Recommended Hardening Improvements During Rebuild
If you want a more robust production version, consider:
1. Adding an **IF** or **Code** node after Jira extraction to handle empty descriptions.
2. Converting Jira rich-text descriptions into plain text before prompt composition.
3. Adding error branches for:
   - CloudCLI API failures
   - agent execution timeouts
   - missing result events
   - Jira comment failures
4. Storing raw CloudCLI response metadata for audit/debugging.
5. Truncating large results before posting to Jira.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| CloudCLI account and API key are required for the CloudCLI nodes. | https://cloudcli.ai |
| CloudCLI developer documentation is referenced directly in the workflow notes. | https://developer.cloudcli.ai |
| n8n verified community node installation guidance is relevant because this workflow depends on `@cloudcli-ai/n8n-nodes-cloud-cli`. | https://docs.n8n.io/integrations/community-nodes/installation/verified-install/ |
| n8n community help channel referenced in the workflow. | https://discord.com/invite/XPKeKXeB7d |
| n8n forum referenced in the workflow. | https://community.n8n.io/ |
| The workflow title indicates support positioning for Claude Code, Gemini, Cursor, or Codex sessions through CloudCLI, but the actual configured agent provider in this exported workflow is `claude`. | Workflow behavior / implementation note |
| The sticky note mentions posting session duration and cost, but the implemented extraction node does not actually extract duration or cost fields. | Behavior mismatch between documentation note and configured nodes |
| The workflow assumes the CloudCLI environment name can be sanitized and used as part of the workspace path for VS Code/Cursor deep links. Validate this in your environment before relying on it operationally. | Implementation note |