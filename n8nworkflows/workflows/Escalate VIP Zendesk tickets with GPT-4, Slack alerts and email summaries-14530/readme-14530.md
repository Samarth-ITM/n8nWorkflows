Escalate VIP Zendesk tickets with GPT-4, Slack alerts and email summaries

https://n8nworkflows.xyz/workflows/escalate-vip-zendesk-tickets-with-gpt-4--slack-alerts-and-email-summaries-14530


# Escalate VIP Zendesk tickets with GPT-4, Slack alerts and email summaries

# 1. Workflow Overview

This workflow automates triage for newly created Zendesk tickets and applies different escalation paths depending on whether the requester is marked as a VIP customer.

Its main purpose is to:
- detect new Zendesk tickets,
- identify VIP tickets using a Zendesk tag,
- summarize VIP issues with GPT-4,
- store VIP ticket details in Airtable,
- alert support staff in Slack,
- and send grouped email summaries for non-VIP tickets.

The workflow is organized into the following logical blocks.

## 1.1 Ticket Intake and VIP Classification
The workflow starts from a Zendesk trigger and immediately checks whether the new ticket contains the `vip` tag.

## 1.2 VIP Ticket Analysis and Tracking
If the ticket is a VIP ticket, the workflow sends the ticket content to OpenAI for a short summary and recommended next steps, then stores the resulting record in Airtable.

## 1.3 Slack Availability Check and VIP Escalation
After storing the VIP ticket, the workflow fetches Slack users, checks them one by one for presence, and decides whether to notify an active user directly or fall back to posting in a shared support channel.

## 1.4 Non-VIP Ticket Preparation and Email Summary
If the ticket is not VIP, the workflow normalizes the ticket fields, aggregates non-VIP tickets into one payload, and emails a summary through Gmail.

## 1.5 Documentation and In-Canvas Guidance
Several sticky notes document the intended business process, setup requirements, and node-level behavior directly in the workflow canvas.

---

# 2. Block-by-Block Analysis

## 2.1 Ticket Intake and VIP Classification

### Overview
This block listens for new Zendesk tickets and decides whether each ticket should follow the VIP escalation path or the non-VIP summary path. The decision is based solely on the presence of the `vip` tag in the ticket tags array.

### Nodes Involved
- New ticket received
- is the VIP Customer?

### Node Details

#### New ticket received
- **Type and role:** `n8n-nodes-base.zendeskTrigger`  
  Entry-point trigger that starts the workflow whenever a new Zendesk ticket event is received.
- **Configuration choices:**
  - Uses **OAuth2** authentication.
  - No special trigger options are configured.
- **Key expressions or variables used:**  
  None in this node itself. It emits Zendesk ticket fields such as:
  - `id`
  - `subject`
  - `description`
  - `priority`
  - `status`
  - `tags`
  - `requester_id`
  - `url`
  - `created_at`
- **Input and output connections:**
  - No input; this is a trigger.
  - Output goes to **is the VIP Customer?**
- **Version-specific requirements:**  
  Uses **typeVersion 1** for the Zendesk Trigger node.
- **Edge cases / failure types:**
  - OAuth2 credential failure or token expiration.
  - Zendesk webhook delivery issues.
  - Missing expected fields in the Zendesk payload.
  - Ticket creation events may differ depending on Zendesk setup or app permissions.
- **Sub-workflow reference:**  
  None.

#### is the VIP Customer?
- **Type and role:** `n8n-nodes-base.if`  
  Branching node that checks whether the incoming ticket is tagged as VIP.
- **Configuration choices:**
  - Uses an **array contains** condition.
  - Condition checks whether `tags` contains `vip`.
  - Strict type validation is enabled by the IF node condition engine.
- **Key expressions or variables used:**
  - Left value: `={{ $json.tags }}`
  - Right value: `vip`
- **Input and output connections:**
  - Input from **New ticket received**
  - True output goes to **Intelligent Ticket Analysis**
  - False output goes to **Prepare non VIP ticket data**
- **Version-specific requirements:**  
  Uses **IF node typeVersion 2.2** with condition format version 2.
- **Edge cases / failure types:**
  - If `tags` is missing or not an array, the condition may not behave as intended.
  - Tag capitalization matters in practice if source values differ; this node expects exact `vip`.
  - Tickets with equivalent but differently formatted tags like `VIP` or `vip_customer` will not match.
- **Sub-workflow reference:**  
  None.

---

## 2.2 VIP Ticket Analysis and Tracking

### Overview
This block handles VIP tickets by generating an AI-based issue summary and support guidance, then storing the ticket and summary in Airtable for tracking and audit purposes.

### Nodes Involved
- Intelligent Ticket Analysis
- Store VIP Ticket details

### Node Details

#### Intelligent Ticket Analysis
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`  
  Calls OpenAI to analyze the VIP ticket and produce a concise summary plus suggested next steps for a support agent.
- **Configuration choices:**
  - Model: **gpt-4-turbo**
  - Prompt is built as a single response content template.
  - No built-in tools enabled.
  - No extra model options configured.
- **Key expressions or variables used:**
  - `{{$json["subject"]}}`
  - `{{$json["description"]}}`
  - `{{$json["priority"]}}`
  - Prompt intent:
    - summarize issue in 2–3 lines,
    - provide clear next steps for support.
- **Input and output connections:**
  - Input from **is the VIP Customer?** true branch
  - Output goes to **Store VIP Ticket details**
- **Version-specific requirements:**  
  Uses **typeVersion 2** of the LangChain OpenAI node. This requires the corresponding n8n AI/node package support.
- **Edge cases / failure types:**
  - OpenAI credential or quota issues.
  - Model availability changes, especially if `gpt-4-turbo` is deprecated or renamed.
  - Long or malformed Zendesk descriptions may increase token usage or cause truncation.
  - Output structure is nested and later nodes depend on:
    - `output[0].content[0].text`
    If the node output schema changes, downstream expressions will fail.
- **Sub-workflow reference:**  
  None.

#### Store VIP Ticket details
- **Type and role:** `n8n-nodes-base.airtable`  
  Creates a new Airtable record containing key VIP ticket details and the AI-generated summary.
- **Configuration choices:**
  - Operation: **create**
  - Base: `n8n Demo`
  - Table: `Customer Ticket`
  - Mapping mode: define fields explicitly
  - Conversion options disabled
- **Key expressions or variables used:**
  - `Link`: `={{ $('is the VIP Customer?').item.json.url }}`
  - `Subject`: `={{ $('is the VIP Customer?').item.json.subject }}`
  - `Priority`: `={{ $('is the VIP Customer?').item.json.priority }}`
  - `Ticket Id`: `={{ $('is the VIP Customer?').item.json.id }}`
  - `Request Id`: `={{ $('is the VIP Customer?').item.json.requester_id }}`
  - `Issue Summary`: `={{ $('Intelligent Ticket Analysis').item.json.output[0].content[0].text }}`
- **Input and output connections:**
  - Input from **Intelligent Ticket Analysis**
  - Output goes to **Get support team members**
- **Version-specific requirements:**  
  Uses **typeVersion 2.1** of the Airtable node.
- **Edge cases / failure types:**
  - Airtable auth failure or revoked token.
  - Base/table mismatch with configured schema.
  - Field names in Airtable must match exactly:
    - `Ticket Id`
    - `Subject`
    - `Priority`
    - `Request Id`
    - `Issue Summary`
    - `Link`
  - If the OpenAI output path is empty or changed, record creation may fail or store blank summary text.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Slack Availability Check and VIP Escalation

### Overview
This block attempts to route the VIP alert intelligently. It fetches Slack users, iterates through them, checks presence status for each, and either sends a direct alert to an active user or posts the alert into a shared support channel if the checked user is not active.

### Nodes Involved
- Get support team members
- Check each team member
- Get a user's presence status
- Is someone Active?
- Notify directly Active user
- Notify support channel

### Important Logic Note
Although the intent is to find an available support team member, the current implementation has limitations:

1. **Get support team members** fetches all Slack users, not a filtered support-only subset.
2. **Notify directly Active user** sends to a hardcoded Slack user ID, not the currently iterated active user.
3. **Notify support channel** is triggered whenever the checked user is not active, meaning the workflow may post fallback alerts repeatedly during iteration unless batch behavior and execution timing happen to stop earlier.
4. The loopback from both notification nodes into **Check each team member** suggests iterative processing, but there is no explicit stop-on-first-active pattern.

So the block expresses the intended design, but in practice it may not behave as a true “find first available agent” flow.

### Node Details

#### Get support team members
- **Type and role:** `n8n-nodes-base.slack`  
  Retrieves Slack users to be inspected for availability.
- **Configuration choices:**
  - Resource: `user`
  - Operation: `getAll`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - Input from **Store VIP Ticket details**
  - Output goes to **Check each team member**
- **Version-specific requirements:**  
  Uses Slack node **typeVersion 2.3**.
- **Edge cases / failure types:**
  - Slack auth or permission scope issues.
  - Workspace may return bots, deactivated users, or many users that are unrelated to support.
  - Pagination/rate limiting may affect large workspaces.
- **Sub-workflow reference:**  
  None.

#### Check each team member
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates over Slack users one by one.
- **Configuration choices:**
  - Default batching behavior, no custom batch size/options visible.
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**
  - Input from **Get support team members**
  - Also receives loopback input from:
    - **Notify directly Active user**
    - **Notify support channel**
  - Secondary batch-processing output goes to **Get a user's presence status**
- **Version-specific requirements:**  
  Uses **typeVersion 3**.
- **Edge cases / failure types:**
  - Loop configuration can create repeated processing patterns if termination is not carefully controlled.
  - If no items are returned from Slack, downstream presence checks will not run.
  - Depending on execution semantics, repeated fallback channel notifications may occur.
- **Sub-workflow reference:**  
  None.

#### Get a user's presence status
- **Type and role:** `n8n-nodes-base.slack`  
  Queries Slack for presence of the current iterated user.
- **Configuration choices:**
  - Resource: `user`
  - Operation: `getPresence`
  - User ID comes from the current item’s `id`
- **Key expressions or variables used:**
  - User: `={{ $json.id }}`
- **Input and output connections:**
  - Input from **Check each team member**
  - Output goes to **Is someone Active?**
- **Version-specific requirements:**  
  Uses Slack node **typeVersion 2.3**.
- **Edge cases / failure types:**
  - Presence data may not be available for all users or plans.
  - Bot/user scope limitations may prevent presence lookup.
  - If the current Slack item lacks `id`, expression evaluation fails.
- **Sub-workflow reference:**  
  None.

#### Is someone Active?
- **Type and role:** `n8n-nodes-base.if`  
  Evaluates whether the current Slack user’s presence is `active`.
- **Configuration choices:**
  - String equality comparison on `presence`
  - Strict validation enabled
- **Key expressions or variables used:**
  - Left value: `={{ $json.presence }}`
  - Right value: `active`
- **Input and output connections:**
  - Input from **Get a user's presence status**
  - True output goes to **Notify directly Active user**
  - False output goes to **Notify support channel**
- **Version-specific requirements:**  
  Uses **IF node typeVersion 2.2**.
- **Edge cases / failure types:**
  - If presence is absent, null, or uses another value such as `away`, the condition falls through to the false branch.
  - This logic only distinguishes `active` from everything else.
- **Sub-workflow reference:**  
  None.

#### Notify directly Active user
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a direct Slack message for the VIP alert.
- **Configuration choices:**
  - Sends to a **specific hardcoded Slack user ID**.
  - Message includes ticket metadata and AI-generated summary.
  - Link-to-workflow attribution disabled.
- **Key expressions or variables used:**
  - Ticket ID: `{{ $('is the VIP Customer?').item.json.id }}`
  - Subject: `{{ $('is the VIP Customer?').item.json.subject }}`
  - Priority: `{{ $('is the VIP Customer?').item.json.priority }}`
  - Requester ID: `{{ $('is the VIP Customer?').item.json.requester_id }}`
  - AI summary: `{{$node["Intelligent Ticket Analysis"].json["output"][0]["content"][0]["text"]}}`
  - Zendesk link: `{{ $('is the VIP Customer?').item.json.url }}`
  - User ID configured as: `=U05JTMVC7JL`
- **Input and output connections:**
  - Input from **Is someone Active?** true branch
  - Output loops back to **Check each team member**
- **Version-specific requirements:**  
  Uses Slack node **typeVersion 2.3**.
- **Edge cases / failure types:**
  - Hardcoded user may not be the active user being checked.
  - Slack DM permission issues.
  - If the target user leaves the workspace or changes availability expectations, the logic becomes misleading.
  - Loopback may continue processing additional users even after a DM was already sent.
- **Sub-workflow reference:**  
  None.

#### Notify support channel
- **Type and role:** `n8n-nodes-base.slack`  
  Posts a VIP alert message into a Slack channel when the checked user is not active.
- **Configuration choices:**
  - Sends to a fixed channel ID.
  - Link-to-workflow attribution disabled.
- **Key expressions or variables used:**
  - Ticket ID: `{{ $('is the VIP Customer?').item.json.id }}`
  - Subject: `{{ $('is the VIP Customer?').item.json.subject }}`
  - Priority: `{{ $('is the VIP Customer?').item.json.priority }}`
  - Requester ID: `{{ $('is the VIP Customer?').item.json.requester_id }}`
  - AI summary: `{{$node["Intelligent Ticket Analysis"].json["output"][0]["content"][0]["text"]}}`
  - Zendesk link: `{{ $('is the VIP Customer?').item.json.url }}`
  - Channel ID configured as: `C09S57E2JQ2`
- **Input and output connections:**
  - Input from **Is someone Active?** false branch
  - Output loops back to **Check each team member**
- **Version-specific requirements:**  
  Uses Slack node **typeVersion 2.3**.
- **Edge cases / failure types:**
  - Multiple repeated channel alerts may occur if many checked users are inactive.
  - Channel permission/scope issues.
  - Message formatting depends on AI output path remaining valid.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Non-VIP Ticket Preparation and Email Summary

### Overview
This block handles non-VIP tickets by extracting relevant fields, aggregating them into a grouped structure, and sending a plain-text email summary to the support team.

### Nodes Involved
- Prepare non VIP ticket data
- Combine all VIP ticket
- Notify for non VIP customer

### Important Naming Note
The node named **Combine all VIP ticket** is misleading. Based on its placement and data source, it actually aggregates **non-VIP** ticket items before emailing them.

### Node Details

#### Prepare non VIP ticket data
- **Type and role:** `n8n-nodes-base.set`  
  Normalizes and maps selected Zendesk fields into a simpler schema for email formatting.
- **Configuration choices:**
  - Creates explicit assigned fields:
    - `ticket_id`
    - `subject`
    - `priority`
    - `status`
    - `description`
    - `url`
    - `created_at`
- **Key expressions or variables used:**
  - `={{ $json.id }}`
  - `={{ $json.subject }}`
  - `={{ $json.priority }}`
  - `={{ $json.status }}`
  - `={{ $json.description }}`
  - `={{ $json.url }}`
  - `={{ $json.created_at }}`
- **Input and output connections:**
  - Input from **is the VIP Customer?** false branch
  - Output goes to **Combine all VIP ticket**
- **Version-specific requirements:**  
  Uses **Set node typeVersion 3.4**.
- **Edge cases / failure types:**
  - Missing input fields from Zendesk lead to empty mapped values.
  - Large descriptions may make the email long and less readable.
- **Sub-workflow reference:**  
  None.

#### Combine all VIP ticket
- **Type and role:** `n8n-nodes-base.aggregate`  
  Aggregates all incoming non-VIP ticket items into one data structure for a single email.
- **Configuration choices:**
  - Aggregate mode: `aggregateAllItemData`
  - Produces a `data` array containing all items.
- **Key expressions or variables used:**  
  No direct expressions in configuration.
- **Input and output connections:**
  - Input from **Prepare non VIP ticket data**
  - Output goes to **Notify for non VIP customer**
- **Version-specific requirements:**  
  Uses **Aggregate node typeVersion 1**.
- **Edge cases / failure types:**
  - If only one ticket is present, aggregation still works but may be unnecessary.
  - If execution runs per trigger event, this may aggregate only items from the current execution, not across separate ticket events.
  - Node name may confuse maintainers because it does not match actual role.
- **Sub-workflow reference:**  
  None.

#### Notify for non VIP customer
- **Type and role:** `n8n-nodes-base.gmail`  
  Sends a plain-text Gmail summary containing all aggregated non-VIP tickets.
- **Configuration choices:**
  - Subject: `New Non-VIP Support Tickets ({{ $json.data.length }})`
  - Email body loops over `$json.data`
  - Attribution disabled
  - Email type: `text`
- **Key expressions or variables used:**
  - Subject: `=New Non-VIP Support Tickets ({{ $json.data.length }})`
  - Message body:
    - `{{ $json.data.map(ticket => ... ).join('\n') }}`
- **Input and output connections:**
  - Input from **Combine all VIP ticket**
  - No downstream nodes
- **Version-specific requirements:**  
  Uses Gmail node **typeVersion 2.1**.
- **Edge cases / failure types:**
  - Gmail OAuth2 credential issues.
  - Recipient settings are not visible in the provided JSON, so actual delivery target should be verified in the node UI.
  - Large aggregated content may exceed practical email size/readability limits.
  - If `$json.data` is missing, the expression fails.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Documentation and In-Canvas Guidance

### Overview
These nodes are non-executing sticky notes that explain workflow behavior, setup steps, and intent. They are essential for maintainability because they clarify business logic and expected integrations.

### Nodes Involved
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10
- Sticky Note11
- Sticky Note12
- Sticky Note13

### Node Details

#### Sticky Note7
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the Zendesk trigger purpose.
- **Configuration choices:**  
  Content explains that the automation starts on new Zendesk support tickets.
- **Input and output connections:**  
  None.
- **Edge cases / failure types:**  
  None; documentation only.
- **Sub-workflow reference:**  
  None.

#### Sticky Note8
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents VIP classification logic.
- **Configuration choices:**  
  Content explains that VIP status is determined by the `vip` tag.
- **Input and output connections:**  
  None.
- **Edge cases / failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note9
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the full VIP support handling block.
- **Configuration choices:**  
  Explains AI summarization, Airtable tracking, Slack team scan, and availability-based routing.
- **Input and output connections:**  
  None.
- **Edge cases / failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note10
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents private Slack escalation to an available senior support agent.
- **Configuration choices:**  
  Descriptive only.
- **Input and output connections:**  
  None.
- **Edge cases / failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note11
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents fallback posting to a shared Slack channel.
- **Configuration choices:**  
  Descriptive only.
- **Input and output connections:**  
  None.
- **Edge cases / failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note12
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Documents the non-VIP email summary branch.
- **Configuration choices:**  
  Explains field cleanup, aggregation, and single-email summary behavior.
- **Input and output connections:**  
  None.
- **Edge cases / failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note13
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Provides full process description and setup checklist.
- **Configuration choices:**  
  Includes overall workflow explanation and six setup steps for Zendesk, Slack, OpenAI, Airtable, Gmail, and Zendesk tagging.
- **Input and output connections:**  
  None.
- **Edge cases / failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New ticket received | Zendesk Trigger | Starts workflow on new Zendesk ticket creation |  | is the VIP Customer? | Starts the automation whenever a new support ticket is received in Zendesk. |
| is the VIP Customer? | IF | Checks whether the ticket has the `vip` tag | New ticket received | Intelligent Ticket Analysis; Prepare non VIP ticket data | Checks whether the ticket belongs to a VIP customer based on the “vip” tag. |
| Intelligent Ticket Analysis | OpenAI (LangChain) | Summarizes VIP ticket and suggests next steps | is the VIP Customer? | Store VIP Ticket details | ## Customer Support Process & Ticket Management Flow  This part of the workflow uses AI to understand VIP tickets, saves the ticket details for tracking and then checks the support team on Slack to see who is currently available. The system goes through each team member one by one, checks who is online and decides the next step based on availability so the ticket can be handled quickly. |
| Store VIP Ticket details | Airtable | Saves VIP ticket metadata and AI summary | Intelligent Ticket Analysis | Get support team members | ## Customer Support Process & Ticket Management Flow  This part of the workflow uses AI to understand VIP tickets, saves the ticket details for tracking and then checks the support team on Slack to see who is currently available. The system goes through each team member one by one, checks who is online and decides the next step based on availability so the ticket can be handled quickly. |
| Get support team members | Slack | Retrieves Slack users for availability checks | Store VIP Ticket details | Check each team member | ## Customer Support Process & Ticket Management Flow  This part of the workflow uses AI to understand VIP tickets, saves the ticket details for tracking and then checks the support team on Slack to see who is currently available. The system goes through each team member one by one, checks who is online and decides the next step based on availability so the ticket can be handled quickly. |
| Check each team member | Split In Batches | Iterates through Slack users | Get support team members; Notify support channel; Notify directly Active user | Get a user's presence status | ## Customer Support Process & Ticket Management Flow  This part of the workflow uses AI to understand VIP tickets, saves the ticket details for tracking and then checks the support team on Slack to see who is currently available. The system goes through each team member one by one, checks who is online and decides the next step based on availability so the ticket can be handled quickly. |
| Get a user's presence status | Slack | Checks Slack presence for current user | Check each team member | Is someone Active? | ## Customer Support Process & Ticket Management Flow  This part of the workflow uses AI to understand VIP tickets, saves the ticket details for tracking and then checks the support team on Slack to see who is currently available. The system goes through each team member one by one, checks who is online and decides the next step based on availability so the ticket can be handled quickly. |
| Is someone Active? | IF | Tests whether current Slack user presence is `active` | Get a user's presence status | Notify directly Active user; Notify support channel | ## Customer Support Process & Ticket Management Flow  This part of the workflow uses AI to understand VIP tickets, saves the ticket details for tracking and then checks the support team on Slack to see who is currently available. The system goes through each team member one by one, checks who is online and decides the next step based on availability so the ticket can be handled quickly. |
| Notify directly Active user | Slack | Sends direct VIP alert in Slack | Is someone Active? | Check each team member | Sends a private Slack message to an available senior support agent about the VIP ticket. |
| Notify support channel | Slack | Posts fallback VIP alert to Slack channel | Is someone Active? | Check each team member | Posts the VIP ticket alert in a shared Slack channel if no agent is active. |
| Prepare non VIP ticket data | Set | Maps non-VIP ticket fields for email aggregation | is the VIP Customer? | Combine all VIP ticket | ## Non-VIP Ticket Email Process  **Prepare Non-VIP Ticket Data:** Cleans and formats ticket information so it can be combined into one email. **Combine All Non-VIP Tickets:** Groups multiple non-VIP tickets together so only one email is sent instead of many. **Notify for non VIP customer:** Sends a single summary email listing all non-VIP tickets for the support team to review. |
| Combine all VIP ticket | Aggregate | Aggregates non-VIP tickets into a single data array | Prepare non VIP ticket data | Notify for non VIP customer | ## Non-VIP Ticket Email Process  **Prepare Non-VIP Ticket Data:** Cleans and formats ticket information so it can be combined into one email. **Combine All Non-VIP Tickets:** Groups multiple non-VIP tickets together so only one email is sent instead of many. **Notify for non VIP customer:** Sends a single summary email listing all non-VIP tickets for the support team to review. |
| Notify for non VIP customer | Gmail | Sends email summary for aggregated non-VIP tickets | Combine all VIP ticket |  | ## Non-VIP Ticket Email Process  **Prepare Non-VIP Ticket Data:** Cleans and formats ticket information so it can be combined into one email. **Combine All Non-VIP Tickets:** Groups multiple non-VIP tickets together so only one email is sent instead of many. **Notify for non VIP customer:** Sends a single summary email listing all non-VIP tickets for the support team to review. |
| Sticky Note7 | Sticky Note | Canvas documentation |  |  | Starts the automation whenever a new support ticket is received in Zendesk. |
| Sticky Note8 | Sticky Note | Canvas documentation |  |  | Checks whether the ticket belongs to a VIP customer based on the “vip” tag. |
| Sticky Note9 | Sticky Note | Canvas documentation |  |  | ## Customer Support Process & Ticket Management Flow  This part of the workflow uses AI to understand VIP tickets, saves the ticket details for tracking and then checks the support team on Slack to see who is currently available. The system goes through each team member one by one, checks who is online and decides the next step based on availability so the ticket can be handled quickly. |
| Sticky Note10 | Sticky Note | Canvas documentation |  |  | Sends a private Slack message to an available senior support agent about the VIP ticket. |
| Sticky Note11 | Sticky Note | Canvas documentation |  |  | Posts the VIP ticket alert in a shared Slack channel if no agent is active. |
| Sticky Note12 | Sticky Note | Canvas documentation |  |  | ## Non-VIP Ticket Email Process  **Prepare Non-VIP Ticket Data:** Cleans and formats ticket information so it can be combined into one email. **Combine All Non-VIP Tickets:** Groups multiple non-VIP tickets together so only one email is sent instead of many. **Notify for non VIP customer:** Sends a single summary email listing all non-VIP tickets for the support team to review. |
| Sticky Note13 | Sticky Note | Canvas documentation |  |  | ## How it Works  This workflow automatically listens for new support tickets from Zendesk. When a ticket is created, it checks whether the customer is marked as a VIP. If the customer is a VIP, the workflow uses AI to understand the problem, creates a short summary with next steps, saves the ticket details for tracking and sends an alert to an available support agent on Slack or to a team channel if no one is online. If the customer is not a VIP, the workflow collects all such tickets and sends one combined email to the support team. This helps VIP customers get faster attention while keeping regular ticket notifications simple and organized.  ## Setup steps  **1.** Connect your Zendesk account to receive new ticket events.  **2.** Set up Slack to fetch users, check online status and send messages.  **3.** Connect OpenAI to summarize VIP tickets clearly.  **4.** Connect Airtable to store VIP ticket details.  **5.** Configure Gmail to send summary emails for non-VIP tickets.  **6.** Make sure VIP customers are correctly tagged in Zendesk. |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence in n8n.

## Prerequisites
Before creating nodes, prepare these credentials and resources:
1. **Zendesk OAuth2** account with permission to receive ticket events.
2. **Slack API** credential with scopes for:
   - reading users,
   - reading user presence,
   - sending DMs/messages,
   - posting to channels.
3. **OpenAI API** credential with access to `gpt-4-turbo` or an equivalent available model.
4. **Airtable Personal Access Token** with create access to the target base/table.
5. **Gmail OAuth2** credential for sending support summary emails.
6. An Airtable table with fields:
   - `Ticket Id`
   - `Subject`
   - `Priority`
   - `Request Id`
   - `Issue Summary`
   - `Link`

## Build Steps

1. **Create a Zendesk Trigger node**
   - Name it **New ticket received**.
   - Choose **Zendesk Trigger**.
   - Set authentication to **OAuth2**.
   - Connect your Zendesk credential.
   - Keep default options unless your Zendesk environment requires filtering.

2. **Create an IF node for VIP detection**
   - Name it **is the VIP Customer?**
   - Add one condition:
     - Left value: `{{ $json.tags }}`
     - Operator: **Array contains**
     - Right value: `vip`
   - Connect **New ticket received** → **is the VIP Customer?**

3. **Create the VIP AI node**
   - Name it **Intelligent Ticket Analysis**.
   - Use the **OpenAI / LangChain OpenAI** node.
   - Select model **gpt-4-turbo**.
   - In the prompt/content field, paste:
     - “You are a support agent assistant. A VIP customer has submitted this ticket:
       Ticket Subject: {{ $json["subject"] }}
       Ticket Description: {{ $json["description"] }}
       Priority: {{ $json["priority"] }}
       Summarize the issue in 2-3 lines and give clear next steps the support agent should take.”
   - Connect the **true** branch of **is the VIP Customer?** to **Intelligent Ticket Analysis**.

4. **Create the Airtable node for VIP storage**
   - Name it **Store VIP Ticket details**.
   - Choose **Airtable**.
   - Operation: **Create**.
   - Select your base and table.
   - Set mapping mode to manual/define fields.
   - Map fields:
     - `Link` = `{{ $('is the VIP Customer?').item.json.url }}`
     - `Subject` = `{{ $('is the VIP Customer?').item.json.subject }}`
     - `Priority` = `{{ $('is the VIP Customer?').item.json.priority }}`
     - `Ticket Id` = `{{ $('is the VIP Customer?').item.json.id }}`
     - `Request Id` = `{{ $('is the VIP Customer?').item.json.requester_id }}`
     - `Issue Summary` = `{{ $('Intelligent Ticket Analysis').item.json.output[0].content[0].text }}`
   - Connect **Intelligent Ticket Analysis** → **Store VIP Ticket details**.

5. **Create a Slack node to list users**
   - Name it **Get support team members**.
   - Choose **Slack**.
   - Resource: **User**
   - Operation: **Get All**
   - Connect **Store VIP Ticket details** → **Get support team members**.
   - Note: this fetches all Slack users. If you want actual support-team-only behavior, add filtering later.

6. **Create a Split In Batches node**
   - Name it **Check each team member**.
   - Keep default settings.
   - Connect **Get support team members** → **Check each team member**.

7. **Create a Slack presence lookup node**
   - Name it **Get a user's presence status**.
   - Slack resource: **User**
   - Operation: **Get Presence**
   - User field: `{{ $json.id }}`
   - Connect the batch-processing output of **Check each team member** → **Get a user's presence status**.

8. **Create an IF node to check activity**
   - Name it **Is someone Active?**
   - Condition:
     - Left value: `{{ $json.presence }}`
     - Operator: **equals**
     - Right value: `active`
   - Connect **Get a user's presence status** → **Is someone Active?**

9. **Create the Slack DM node**
   - Name it **Notify directly Active user**.
   - Slack operation: send message to a **user**.
   - Select mode: **user**
   - Set a user ID manually to the target senior support agent:
     - `U05JTMVC7JL` in this workflow
   - Message text:
     - include ticket ID, subject, priority, requester ID, AI summary, and Zendesk link using the same expressions as the source workflow.
   - Connect the **true** branch of **Is someone Active?** → **Notify directly Active user**.

10. **Create the fallback Slack channel alert**
    - Name it **Notify support channel**.
    - Slack operation: send message to a **channel**.
    - Set channel ID to your support channel.
      - In this workflow: `C09S57E2JQ2`
    - Use the same alert text structure as the DM node.
    - Connect the **false** branch of **Is someone Active?** → **Notify support channel**.

11. **Loop the Slack branch back into the batch node**
    - Connect **Notify directly Active user** → **Check each team member**
    - Connect **Notify support channel** → **Check each team member**
    - This reproduces the original looping structure exactly.
    - Important: this design may send repeated channel notifications or continue processing after finding an active user. If rebuilding for production, consider replacing this with a controlled stop condition.

12. **Create a Set node for non-VIP data**
    - Name it **Prepare non VIP ticket data**.
    - Connect the **false** branch of **is the VIP Customer?** → **Prepare non VIP ticket data**
    - Add fields:
      - `ticket_id` as number = `{{ $json.id }}`
      - `subject` as string = `{{ $json.subject }}`
      - `priority` as string = `{{ $json.priority }}`
      - `status` as string = `{{ $json.status }}`
      - `description` as string = `{{ $json.description }}`
      - `url` as string = `{{ $json.url }}`
      - `created_at` as string = `{{ $json.created_at }}`

13. **Create an Aggregate node**
    - Name it **Combine all VIP ticket** to match the original workflow, even though it actually handles non-VIP tickets.
    - Operation/mode: **Aggregate all item data**
    - Connect **Prepare non VIP ticket data** → **Combine all VIP ticket**

14. **Create a Gmail node**
    - Name it **Notify for non VIP customer**
    - Connect **Combine all VIP ticket** → **Notify for non VIP customer**
    - Email type: **text**
    - Subject:
      - `New Non-VIP Support Tickets ({{ $json.data.length }})`
    - Message body:
      - Build a plain text summary using:
        - ticket ID
        - subject
        - priority
        - status
        - created date
        - description
        - Zendesk link
      - Use a map/join expression over `$json.data`
    - Disable attribution if preferred.
    - Set the recipient in the Gmail node UI as appropriate for your support mailbox or team alias.

15. **Add optional sticky notes**
    - Recreate the documentation notes if you want the same maintainability aids:
      - Zendesk trigger explanation
      - VIP tag explanation
      - VIP AI + Slack process
      - private Slack alert note
      - channel fallback note
      - non-VIP email process
      - global setup notes

## Credential Configuration Summary

### Zendesk
- Use OAuth2.
- Ensure the trigger can subscribe to new ticket events.
- Validate webhook/callback settings if self-hosting n8n.

### Slack
- Use one Slack credential for all Slack nodes.
- Required capabilities typically include:
  - listing users,
  - reading presence,
  - sending DMs,
  - posting channel messages.
- If presence API is unavailable in your workspace/app setup, this block must be redesigned.

### OpenAI
- Add an OpenAI API credential.
- Select a currently supported model. If `gpt-4-turbo` is unavailable, use the nearest compatible replacement and adjust output testing accordingly.

### Airtable
- Use a Personal Access Token.
- Confirm the target base/table exist and exact field names match.

### Gmail
- Use OAuth2.
- Confirm the sender account is authorized to send to the intended recipients.
- Verify any organization email restrictions.

## Reproduction Warnings
1. The non-VIP aggregation happens only within the current workflow execution. Since the trigger is per ticket, this may often produce one email per ticket rather than true cross-execution batching.
2. The Slack “active user” branch does not DM the iterated active Slack user; it DMs a fixed user ID.
3. The fallback channel post can occur multiple times in the current loop design.
4. The Airtable and Slack messages depend on the nested OpenAI output path:
   - `output[0].content[0].text`
   Test this after model/node upgrades.

## Recommended Improvements if Rebuilding
If you want a production-grade version while preserving intent:
1. Filter Slack users to a support-team subset.
2. DM `{{ $json.id }}` or the current active user instead of a hardcoded ID.
3. Stop iterating after the first active user is found.
4. Send channel fallback only once after no active users are found.
5. Replace per-execution non-VIP aggregation with scheduled batching if you want digest emails.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically listens for new support tickets from Zendesk. When a ticket is created, it checks whether the customer is marked as a VIP. If the customer is a VIP, the workflow uses AI to understand the problem, creates a short summary with next steps, saves the ticket details for tracking and sends an alert to an available support agent on Slack or to a team channel if no one is online. If the customer is not a VIP, the workflow collects all such tickets and sends one combined email to the support team. This helps VIP customers get faster attention while keeping regular ticket notifications simple and organized. | Overall workflow purpose |
| Connect your Zendesk account to receive new ticket events. | Setup guidance |
| Set up Slack to fetch users, check online status and send messages. | Setup guidance |
| Connect OpenAI to summarize VIP tickets clearly. | Setup guidance |
| Connect Airtable to store VIP ticket details. | Setup guidance |
| Configure Gmail to send summary emails for non-VIP tickets. | Setup guidance |
| Make sure VIP customers are correctly tagged in Zendesk. | Setup guidance |