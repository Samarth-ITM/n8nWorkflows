Gate AI Slack DevOps bot actions with OpenAI and Permit.io RBAC

https://n8nworkflows.xyz/workflows/gate-ai-slack-devops-bot-actions-with-openai-and-permit-io-rbac-14087


# Gate AI Slack DevOps bot actions with OpenAI and Permit.io RBAC

# 1. Workflow Overview

This workflow implements a Slack-based DevOps bot that uses OpenAI to classify a user’s request and Permit.io to enforce role-based access control before any action is executed.

Typical use case:
- A team member mentions the bot in Slack with a DevOps request such as “deploy staging” or “restart production”.
- OpenAI extracts the intended `action` and `resource`.
- Permit.io decides whether the Slack user is authorized.
- If authorized, the workflow executes a downstream action.
- If denied, the workflow responds with the user’s available permissions and a list of users who can perform the requested action.

## 1.1 Trigger & Request Classification

This block receives Slack `app_mention` events and sends the raw message text to OpenAI for structured intent extraction.

## 1.2 Permission Gate

This block takes the classified `action` and `resource`, checks them against Permit.io RBAC policies, and branches according to the authorization result.

## 1.3 Allowed Path: Action Execution

If Permit.io returns `allow = true`, the workflow triggers a mock HTTP endpoint representing an infrastructure action, then posts a success message back to Slack.

## 1.4 Denied Path: User Guidance

If access is denied, the workflow queries Permit.io for:
- the current user’s permissions
- the users authorized for the requested action/resource

It then posts a denial message in Slack with escalation guidance.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Request Classification

### Overview
This block starts the workflow when the bot is mentioned in Slack. It then uses OpenAI to transform free-form text into a strict JSON object containing the requested DevOps action, resource, and a short summary.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Slack Trigger - Bot Mention
- Classify DevOps Intent

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation only.
- **Configuration choices:** Contains the overall workflow description, setup requirements, and customization guidance.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for the first functional block.
- **Configuration choices:** Explains that Slack triggers the workflow and OpenAI extracts `action` and `resource`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Slack Trigger - Bot Mention
- **Type and technical role:** `n8n-nodes-base.slackTrigger`; event-driven entry point.
- **Configuration choices:**
  - Trigger type: `app_mention`
  - Channel filter is effectively unset, so mentions are not restricted to a specific channel.
  - Uses Slack API credentials.
- **Key expressions or variables used:**
  - Downstream nodes use values such as:
    - `$('Slack Trigger - Bot Mention').item.json.user`
    - `$('Slack Trigger - Bot Mention').item.json.channel`
    - `$json.text`
- **Input and output connections:**
  - Entry point node; no input.
  - Output goes to `Classify DevOps Intent`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Slack app may be missing required scopes.
  - Slack event subscriptions may not be configured correctly.
  - Bot mentions may include formatting, user mentions, or extra text that complicates classification.
  - If the event payload lacks expected fields like `text`, downstream expressions may fail.
- **Sub-workflow reference:** None.

#### Classify DevOps Intent
- **Type and technical role:** `n8n-nodes-base.openAi`; LLM-based classification step.
- **Configuration choices:**
  - Resource: `chat`
  - Model: `gpt-4o`
  - Temperature: `0` for deterministic classification
  - System prompt strictly constrains outputs to JSON with fields:
    - `action`
    - `resource`
    - `summary`
  - Valid actions: `deploy`, `restart`, `view`, `rotate`
  - Valid resources: `staging`, `production`, `logs`, `secrets`
  - Unknown requests must return:
    - `action = unknown`
    - `resource = unknown`
- **Key expressions or variables used:**
  - User message input: `={{ $json.text }}`
  - Downstream nodes parse `message.content` with `JSON.parse(...)`
- **Input and output connections:**
  - Input from `Slack Trigger - Bot Mention`
  - Output to `Check Permission`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - OpenAI auth issues or quota/rate-limit failures.
  - Model may still return malformed JSON despite prompt constraints.
  - If the node response shape differs across n8n/OpenAI node versions, `message.content` may not exist as expected.
  - Ambiguous user requests may produce `unknown`, which then reaches Permit.io.
- **Sub-workflow reference:** None.

---

## 2.2 Permission Gate

### Overview
This block converts the LLM output into permission-check parameters and asks Permit.io whether the Slack user can perform the requested action on the requested resource. It then branches into allowed or denied logic.

### Nodes Involved
- Sticky Note2
- Check Permission
- Is Allowed?

### Node Details

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation for authorization logic.
- **Configuration choices:** Notes that Permit.io performs the RBAC check and the IF node routes based on the result.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Check Permission
- **Type and technical role:** `@permitio/n8n-nodes-permitio.permit`; authorization decision node.
- **Configuration choices:**
  - Default permission-check operation is used.
  - `user` is the Slack user ID from the trigger payload.
  - `action` and `resource` are extracted by parsing the JSON returned from OpenAI.
- **Key expressions or variables used:**
  - User: `={{ $('Slack Trigger - Bot Mention').item.json.user }}`
  - Action: `={{ JSON.parse($json.message.content).action }}`
  - Resource: `={{ JSON.parse($json.message.content).resource }}`
- **Input and output connections:**
  - Input from `Classify DevOps Intent`
  - Output to `Is Allowed?`
- **Version-specific requirements:**
  - Requires the Permit.io community node package `@permitio/n8n-nodes-permitio`.
  - Type version `1`.
- **Edge cases or potential failure types:**
  - Community node not installed in the n8n instance.
  - Permit.io credentials invalid or expired.
  - Slack user IDs not synced to Permit.io users.
  - JSON parsing fails if OpenAI output is malformed.
  - Unknown action/resource values may be denied or rejected depending on Permit.io setup.
- **Sub-workflow reference:** None.

#### Is Allowed?
- **Type and technical role:** `n8n-nodes-base.if`; boolean branch node.
- **Configuration choices:**
  - Condition checks whether `$json.allow` equals `true`.
  - True branch goes to action execution.
  - False branch goes to denial handling.
- **Key expressions or variables used:**
  - `={{ $json.allow }}`
- **Input and output connections:**
  - Input from `Check Permission`
  - True output to `Execute DevOps Action (Mock)`
  - False output to `Get User Permissions`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If Permit.io returns a different property name or a non-boolean value, branching may misbehave.
  - Missing `allow` field leads to a false branch or evaluation issues.
- **Sub-workflow reference:** None.

---

## 2.3 Allowed Path: Action Execution

### Overview
This block executes the requested operation through a mock HTTP endpoint and confirms success back to the originating Slack channel.

### Nodes Involved
- Sticky Note3
- Execute DevOps Action (Mock)
- Slack - Action Succeeded

### Node Details

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for the authorized execution path.
- **Configuration choices:** Advises replacing the mock HTTP request with a real infrastructure endpoint such as GitHub Actions, ArgoCD, or Jenkins.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Execute DevOps Action (Mock)
- **Type and technical role:** `n8n-nodes-base.httpRequest`; external action invoker.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://httpbin.org/post`
  - Sends a request body containing:
    - `action`
    - `resource`
    - `triggered_by`
  - This is explicitly a placeholder for a real DevOps system.
- **Key expressions or variables used:**
  - `={{ JSON.parse($('Classify DevOps Intent').item.json.message.content).action }}`
  - `={{ JSON.parse($('Classify DevOps Intent').item.json.message.content).resource }}`
  - `={{ $('Slack Trigger - Bot Mention').item.json.user }}`
- **Input and output connections:**
  - Input from `Is Allowed?` true branch
  - Output to `Slack - Action Succeeded`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Remote endpoint unavailable, slow, or returning non-2xx responses.
  - JSON parsing failure from OpenAI output.
  - Real infra endpoint may require authentication, headers, or a different payload structure.
  - No validation exists here to prevent execution on `unknown` actions if Permit.io allows them by configuration mistake.
- **Sub-workflow reference:** None.

#### Slack - Action Succeeded
- **Type and technical role:** `n8n-nodes-base.slack`; sends a message to Slack.
- **Configuration choices:**
  - Posts to the same channel from which the bot mention originated.
  - Text includes:
    - action
    - resource
    - summary
    - triggering Slack user mention
- **Key expressions or variables used:**
  - Channel: `={{ $('Slack Trigger - Bot Mention').item.json.channel }}`
  - Action/resource/summary parsed from `$('Classify DevOps Intent').item.json.message.content`
  - User mention from trigger payload
- **Input and output connections:**
  - Input from `Execute DevOps Action (Mock)`
  - No downstream output
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Slack auth issues or insufficient bot scopes.
  - Channel ID may be invalid or inaccessible to the bot.
  - Message formatting may break if parsed JSON fields are missing.
  - This message indicates success based on upstream node completion, not necessarily on semantic success of a real infra action unless the HTTP step is hardened.
- **Sub-workflow reference:** None.

---

## 2.4 Denied Path: User Guidance

### Overview
When access is denied, this block gathers the current user’s permitted actions and identifies users who are authorized for the requested operation. It then sends a Slack message explaining the denial and possible escalation path.

### Nodes Involved
- Sticky Note4
- Get User Permissions
- Get Authorized Users
- Slack - Permission Denied

### Node Details

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for the denied branch.
- **Configuration choices:** Explains that the branch helps the user understand what they can do and whom to contact.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Get User Permissions
- **Type and technical role:** `@permitio/n8n-nodes-permitio.permit`; permission introspection node.
- **Configuration choices:**
  - Operation: `getUserPermissions`
  - Resource types queried: `logs,staging,production,secrets`
  - Target user is the Slack user ID from the trigger
- **Key expressions or variables used:**
  - User: `={{ $('Slack Trigger - Bot Mention').item.json.user }}`
  - Static resource type list: `=logs,staging,production,secrets`
- **Input and output connections:**
  - Input from `Is Allowed?` false branch
  - Output to `Get Authorized Users`
- **Version-specific requirements:**
  - Requires the Permit.io community node.
  - Type version `1`.
- **Edge cases or potential failure types:**
  - Slack user missing in Permit.io.
  - Tenant structure may differ from the expected `__tenant:default`.
  - Returned permission format may vary depending on Permit.io configuration.
- **Sub-workflow reference:** None.

#### Get Authorized Users
- **Type and technical role:** `@permitio/n8n-nodes-permitio.permit`; retrieves users allowed to perform the requested operation.
- **Configuration choices:**
  - Operation: `getAuthorizedUsers`
  - Action and resource type are derived from the OpenAI classification output.
- **Key expressions or variables used:**
  - Action: `={{ JSON.parse($('Classify DevOps Intent').item.json.message.content).action }}`
  - Resource type: `={{ JSON.parse($('Classify DevOps Intent').item.json.message.content).resource }}`
- **Input and output connections:**
  - Input from `Get User Permissions`
  - Output to `Slack - Permission Denied`
- **Version-specific requirements:**
  - Requires the Permit.io community node.
  - Type version `1`.
- **Edge cases or potential failure types:**
  - Unknown or invalid action/resource may produce empty results.
  - Returned user object may be empty or differently structured.
  - If the Permit.io schema uses a different terminology than the classifier output, results may be misleading.
- **Sub-workflow reference:** None.

#### Slack - Permission Denied
- **Type and technical role:** `n8n-nodes-base.slack`; sends a denial response to Slack.
- **Configuration choices:**
  - Posts into the originating Slack channel.
  - Message includes:
    - the denied action and resource
    - the user’s current permissions
    - a list of authorized users who can help
- **Key expressions or variables used:**
  - Channel: `={{ $('Slack Trigger - Bot Mention').item.json.channel }}`
  - Denied action/resource from `Classify DevOps Intent`
  - Current permissions from `$('Get User Permissions').item.json['__tenant:default'].permissions.join(', ')`
  - Authorized users from `Object.keys($('Get Authorized Users').item.json.users || {}).join(', ')`
- **Input and output connections:**
  - Input from `Get Authorized Users`
  - No downstream output
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If `__tenant:default` does not exist, expression evaluation fails.
  - If `permissions` is not an array, `.join(', ')` fails.
  - If `users` is empty, the “Authorized users” line becomes blank.
  - Slack delivery may fail due to auth or channel access issues.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Global visual documentation for purpose, setup, and customization |  |  | ## Gate AI Slack bot actions with role-based permissions from Permit.io<br><br>### How it works<br>A Slack bot that handles DevOps requests and checks every action against Permit.io RBAC before execution.<br><br>1. Team member @mentions the bot with a request<br>2. OpenAI classifies the intent into action + resource<br>3. Permit.io checks if the user has permission<br>4. Allowed → executes the action, posts result to Slack<br>5. Denied → shows what the user can do and who to escalate to<br><br>### Setup<br>1. Install `@permitio/n8n-nodes-permitio` community node<br>2. Configure Slack app with `app_mentions:read`, `chat:write`, `channels:read`, `users:read`<br>3. Add OpenAI API key<br>4. In Permit.io, create resources (logs, staging, production, secrets) with actions (view, deploy, restart, rotate)<br>5. Create roles: viewer, developer, sre, admin<br>6. Sync Slack user IDs as users in Permit.io and assign roles<br><br>### Customization<br>- Replace the mock HTTP Request with your real infra endpoints<br>- Add ABAC conditions in Permit.io for time-based or amount-based rules without changing this workflow |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual annotation for trigger and classification block |  |  | ### 1. Trigger & Classify<br>Slack trigger captures the @mention, then OpenAI extracts the action and resource from natural language. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual annotation for authorization block |  |  | ### 2. Permission Gate<br>Permit.io checks RBAC policy. The IF node branches on the result. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual annotation for allowed execution branch |  |  | ### 3a. Allowed → Execute<br>Replace this HTTP Request with your actual infra endpoint (GitHub Actions, ArgoCD, Jenkins, etc.) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual annotation for denied/help branch |  |  | ### 3b. Denied → Help<br>Tells the user what they CAN do and who to escalate to. |
| Slack Trigger - Bot Mention | n8n-nodes-base.slackTrigger | Receives Slack bot mentions as workflow entry events |  | Classify DevOps Intent | ### 1. Trigger & Classify<br>Slack trigger captures the @mention, then OpenAI extracts the action and resource from natural language. |
| Classify DevOps Intent | n8n-nodes-base.openAi | Converts Slack text into structured action/resource JSON | Slack Trigger - Bot Mention | Check Permission | ### 1. Trigger & Classify<br>Slack trigger captures the @mention, then OpenAI extracts the action and resource from natural language. |
| Check Permission | @permitio/n8n-nodes-permitio.permit | Checks whether the Slack user may perform the classified action/resource | Classify DevOps Intent | Is Allowed? | ### 2. Permission Gate<br>Permit.io checks RBAC policy. The IF node branches on the result. |
| Is Allowed? | n8n-nodes-base.if | Branches execution based on Permit.io allow/deny result | Check Permission | Execute DevOps Action (Mock); Get User Permissions | ### 2. Permission Gate<br>Permit.io checks RBAC policy. The IF node branches on the result. |
| Execute DevOps Action (Mock) | n8n-nodes-base.httpRequest | Calls a placeholder endpoint representing the approved DevOps action | Is Allowed? | Slack - Action Succeeded | ### 3a. Allowed → Execute<br>Replace this HTTP Request with your actual infra endpoint (GitHub Actions, ArgoCD, Jenkins, etc.) |
| Slack - Action Succeeded | n8n-nodes-base.slack | Posts success confirmation to the original Slack channel | Execute DevOps Action (Mock) |  | ### 3a. Allowed → Execute<br>Replace this HTTP Request with your actual infra endpoint (GitHub Actions, ArgoCD, Jenkins, etc.) |
| Get User Permissions | @permitio/n8n-nodes-permitio.permit | Retrieves current permissions for the requesting user | Is Allowed? | Get Authorized Users | ### 3b. Denied → Help<br>Tells the user what they CAN do and who to escalate to. |
| Get Authorized Users | @permitio/n8n-nodes-permitio.permit | Finds users who are allowed to perform the denied action/resource | Get User Permissions | Slack - Permission Denied | ### 3b. Denied → Help<br>Tells the user what they CAN do and who to escalate to. |
| Slack - Permission Denied | n8n-nodes-base.slack | Sends denial details, user permissions, and escalation options to Slack | Get Authorized Users |  | ### 3b. Denied → Help<br>Tells the user what they CAN do and who to escalate to. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like: `Gate AI Slack bot actions with role-based permissions from Permit.io`.

2. **Add a global sticky note**
   - Add a **Sticky Note** node.
   - Paste a description of the workflow purpose, setup requirements, and customization notes.
   - Include these setup requirements:
     - install `@permitio/n8n-nodes-permitio`
     - configure Slack scopes: `app_mentions:read`, `chat:write`, `channels:read`, `users:read`
     - add OpenAI credentials
     - define Permit.io resources/actions/roles
     - sync Slack users into Permit.io

3. **Add a block note for the first section**
   - Add another **Sticky Note**.
   - Label it: `1. Trigger & Classify`.
   - Note that Slack captures the mention and OpenAI extracts action/resource.

4. **Create the Slack trigger node**
   - Add **Slack Trigger**.
   - Rename it to `Slack Trigger - Bot Mention`.
   - Set the trigger event to `app_mention`.
   - Leave channel filtering empty unless you want to restrict the bot to a specific channel.
   - Configure Slack credentials using your Slack app.
   - Ensure the app is installed in the workspace and event subscriptions are enabled.

5. **Create the OpenAI classification node**
   - Add an **OpenAI** node.
   - Rename it to `Classify DevOps Intent`.
   - Connect `Slack Trigger - Bot Mention` → `Classify DevOps Intent`.
   - Configure:
     - **Resource:** `Chat`
     - **Model:** `gpt-4o`
     - **Temperature:** `0`
   - Set the system message to instruct the model to output only JSON in this structure:
     - `action`
     - `resource`
     - `summary`
   - Restrict valid values:
     - actions: `deploy`, `restart`, `view`, `rotate`
     - resources: `staging`, `production`, `logs`, `secrets`
   - Add a user message using the expression:
     - `{{ $json.text }}`
   - Configure OpenAI credentials with a valid API key.

6. **Add a block note for the permission gate**
   - Add a **Sticky Note** labeled `2. Permission Gate`.
   - Note that Permit.io performs RBAC and an IF node branches on the result.

7. **Create the Permit.io permission check**
   - Add the Permit.io community node.
   - Rename it to `Check Permission`.
   - Connect `Classify DevOps Intent` → `Check Permission`.
   - Configure Permit.io credentials.
   - Set:
     - **User:** `{{ $('Slack Trigger - Bot Mention').item.json.user }}`
     - **Action:** `{{ JSON.parse($json.message.content).action }}`
     - **Resource:** `{{ JSON.parse($json.message.content).resource }}`
   - This assumes the OpenAI node returns the JSON text in `message.content`.

8. **Create the IF branch**
   - Add an **IF** node.
   - Rename it to `Is Allowed?`.
   - Connect `Check Permission` → `Is Allowed?`.
   - Add a boolean condition:
     - left value: `{{ $json.allow }}`
     - operator: equals
     - right value: `true`

9. **Add a block note for the allowed branch**
   - Add a **Sticky Note** labeled `3a. Allowed → Execute`.
   - Mention that the HTTP node is a placeholder and should later be replaced with GitHub Actions, ArgoCD, Jenkins, or another real backend.

10. **Create the execution node**
    - Add an **HTTP Request** node.
    - Rename it to `Execute DevOps Action (Mock)`.
    - Connect the **true** output of `Is Allowed?` to this node.
    - Configure:
      - **Method:** `POST`
      - **URL:** `https://httpbin.org/post`
      - **Send Body:** enabled
    - Add body parameters:
      - `action` = `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).action }}`
      - `resource` = `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).resource }}`
      - `triggered_by` = `{{ $('Slack Trigger - Bot Mention').item.json.user }}`
    - If replacing with a real endpoint later, also configure authentication, headers, and request schema as required.

11. **Create the success Slack message**
    - Add a **Slack** node.
    - Rename it to `Slack - Action Succeeded`.
    - Connect `Execute DevOps Action (Mock)` → `Slack - Action Succeeded`.
    - Use the same Slack credentials as the trigger or another valid Slack credential set.
    - Set it to post to a channel by ID.
    - Channel expression:
      - `{{ $('Slack Trigger - Bot Mention').item.json.channel }}`
    - Message text:
      - Include the action, resource, summary, and the requesting user.
      - Example expressions:
        - `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).action }}`
        - `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).resource }}`
        - `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).summary }}`
        - `<@{{ $('Slack Trigger - Bot Mention').item.json.user }}>`

12. **Add a block note for the denied branch**
    - Add a **Sticky Note** labeled `3b. Denied → Help`.
    - Mention that this branch explains what the user can do and who can help.

13. **Create the user-permissions lookup**
    - Add another Permit.io node.
    - Rename it to `Get User Permissions`.
    - Connect the **false** output of `Is Allowed?` to this node.
    - Set operation to `getUserPermissions`.
    - Set resource types to:
      - `logs,staging,production,secrets`
    - Set target user to:
      - `{{ $('Slack Trigger - Bot Mention').item.json.user }}`

14. **Create the authorized-users lookup**
    - Add another Permit.io node.
    - Rename it to `Get Authorized Users`.
    - Connect `Get User Permissions` → `Get Authorized Users`.
    - Set operation to `getAuthorizedUsers`.
    - Set:
      - **Action:** `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).action }}`
      - **Resource Type:** `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).resource }}`

15. **Create the denial Slack message**
    - Add another **Slack** node.
    - Rename it to `Slack - Permission Denied`.
    - Connect `Get Authorized Users` → `Slack - Permission Denied`.
    - Configure it to post to the original Slack channel:
      - `{{ $('Slack Trigger - Bot Mention').item.json.channel }}`
    - Build the message text with:
      - denied action/resource
      - current user mention
      - current permissions
      - list of users who can help
    - Use these expressions:
      - action: `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).action }}`
      - resource: `{{ JSON.parse($('Classify DevOps Intent').item.json.message.content).resource }}`
      - permissions: `{{ $('Get User Permissions').item.json['__tenant:default'].permissions.join(', ') }}`
      - authorized users: `{{ Object.keys($('Get Authorized Users').item.json.users || {}).join(', ') }}`

16. **Connect the full graph**
    - Main sequence:
      - `Slack Trigger - Bot Mention` → `Classify DevOps Intent` → `Check Permission` → `Is Allowed?`
    - Allowed branch:
      - `Is Allowed? (true)` → `Execute DevOps Action (Mock)` → `Slack - Action Succeeded`
    - Denied branch:
      - `Is Allowed? (false)` → `Get User Permissions` → `Get Authorized Users` → `Slack - Permission Denied`

17. **Set up credentials**
    - **Slack credentials**
      - Create or reuse Slack API credentials.
      - Install the Slack app in the target workspace.
      - Make sure the bot can read mentions and post in target channels.
    - **OpenAI credentials**
      - Add an OpenAI API credential with a valid API key.
    - **Permit.io credentials**
      - Install the Permit.io community node first.
      - Add Permit.io API credentials required by that node.

18. **Prepare Permit.io data model**
    - In Permit.io, create resource types:
      - `logs`
      - `staging`
      - `production`
      - `secrets`
    - Create actions:
      - `view`
      - `deploy`
      - `restart`
      - `rotate`
    - Create roles such as:
      - `viewer`
      - `developer`
      - `sre`
      - `admin`
    - Assign permissions to roles.
    - Create users whose IDs match Slack user IDs.
    - Assign roles to those users.

19. **Test the classifier**
    - Mention the bot with examples like:
      - “deploy staging”
      - “restart production”
      - “view logs”
      - “rotate secrets”
    - Also test unsupported inputs like:
      - “book a meeting”
    - Confirm the OpenAI node returns valid JSON text.

20. **Test the authorization flow**
    - Test with a Slack user who should be allowed.
    - Test with a Slack user who should be denied.
    - Verify:
      - allow path triggers the HTTP request and posts success
      - deny path lists permissions and authorized users

21. **Harden the workflow for production**
    - Add validation after the OpenAI node to catch malformed JSON.
    - Add fallback handling for `unknown` action/resource.
    - Replace `httpbin.org` with a real internal endpoint.
    - Consider adding retries or error branches around:
      - OpenAI
      - Permit.io
      - Slack posting
      - HTTP execution

### Sub-workflow setup
This workflow does **not** use sub-workflows and does **not** invoke any other workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Permit.io community node package: `@permitio/n8n-nodes-permitio` | Required before the Permit.io nodes can be used in n8n |
| Slack app should include scopes `app_mentions:read`, `chat:write`, `channels:read`, `users:read` | Slack integration setup |
| Permit.io should contain resources `logs`, `staging`, `production`, `secrets` and actions `view`, `deploy`, `restart`, `rotate` | RBAC model expected by the workflow |
| Suggested roles: `viewer`, `developer`, `sre`, `admin` | Permit.io role design |
| Slack user IDs should be synchronized into Permit.io as users | Required so the RBAC check can evaluate the requesting Slack user |
| Replace the mock HTTP request with your real infrastructure endpoint | Examples mentioned: GitHub Actions, ArgoCD, Jenkins |
| ABAC conditions can be added in Permit.io without changing this workflow | Useful for time-based or amount-based authorization rules |