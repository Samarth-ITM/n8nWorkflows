Find leads from LinkedIn posts using Airtop agents

https://n8nworkflows.xyz/workflows/find-leads-from-linkedin-posts-using-airtop-agents-14009


# Find leads from LinkedIn posts using Airtop agents

# 1. Workflow Overview

This workflow is a compact sub-workflow designed to be called by another n8n workflow. Its purpose is to take a LinkedIn profile URL and pass it to an Airtop agent that identifies potential leads from engagement on relevant LinkedIn posts. The workflow acts mainly as a wrapper around a prebuilt Airtop template/agent.

Typical use cases include:

- Enriching outbound lead generation from LinkedIn activity
- Turning post commenters or engagers into prospect lists
- Embedding Airtop lead discovery inside a larger sales automation pipeline
- Standardizing lead extraction behind a reusable sub-workflow interface

## 1.1 Input Reception

The workflow starts with an **Execute Workflow Trigger** node that defines the expected inputs from a parent workflow. These inputs include the LinkedIn profile URL and optional controls such as lookback period, post limits, lead limits, output format, and an optional Prospeo API key.

## 1.2 Airtop Agent Execution

The second block sends the collected inputs to an Airtop agent named **Turn LinkedIn Engagements to Pipeline**. This agent performs the actual lead-finding logic externally in Airtop. The n8n workflow itself does not inspect LinkedIn or parse content directly; it delegates the work to the agent.

## 1.3 Embedded Documentation / Operator Guidance

A sticky note documents the business purpose of the workflow, links to a video, and highlights an important prerequisite: the Airtop template must already be installed in the Airtop account used by the workflow credentials.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview

This block defines the sub-workflow’s input contract. It allows a parent workflow to call this workflow with a structured set of fields that will later be mapped into the Airtop agent invocation.

### Nodes Involved

- **When Executed by Another Workflow**

### Node Details

#### 1. When Executed by Another Workflow

- **Type and technical role:**  
  `n8n-nodes-base.executeWorkflowTrigger`  
  Entry-point trigger for a sub-workflow. It exposes named input parameters that can be passed from a parent workflow using an Execute Workflow node.

- **Configuration choices:**  
  The node defines the following expected workflow inputs:
  - `linkedinUrl`
  - `prospeoApiKey`
  - `daysLookback` (number)
  - `maxLeads` (number)
  - `maxPosts` (number)
  - `outputType`

  Only `daysLookback`, `maxLeads`, and `maxPosts` are explicitly typed as numbers in the input schema. The other fields are treated as generic/string-like inputs.

- **Key expressions or variables used:**  
  This node does not use expressions itself, but it exposes values later consumed as:
  - `$json.linkedinUrl`
  - `$json.prospeoApiKey`
  - `$json.daysLookback`
  - `$json.maxLeads`
  - `$json.maxPosts`
  - `$json.outputType`

- **Input and output connections:**  
  - **Input:** none; this is the workflow entry point
  - **Output:** connected to **Run "Turn LinkedIn Engagements to Pipeline"**

- **Version-specific requirements (if any):**  
  Uses type version **1.1**, which supports structured workflow input definitions for sub-workflows.

- **Edge cases or potential failure types:**  
  - If the workflow is executed directly rather than from another workflow, test data may be needed unless manually pinned.
  - If `linkedinUrl` is missing, downstream Airtop execution may fail because the Airtop agent schema marks this as required.
  - If parent workflow passes non-numeric values into numeric inputs, type mismatches may occur depending on caller behavior and n8n coercion.
  - Null values are possible, as shown in the pinned sample data.

- **Sub-workflow reference:**  
  This node itself is the sub-workflow trigger; it does not invoke another workflow.

---

## Block 2 — Airtop Agent Execution

### Overview

This block maps incoming workflow inputs into an Airtop agent call. It applies default values for optional parameters and sends the final parameter set to a specific Airtop agent installed in the connected Airtop account.

### Nodes Involved

- **Run "Turn LinkedIn Engagements to Pipeline"**

### Node Details

#### 2. Run "Turn LinkedIn Engagements to Pipeline"

- **Type and technical role:**  
  `n8n-nodes-base.airtop`  
  Executes an Airtop agent with a defined parameter payload.

- **Configuration choices:**  
  - **Resource:** `agent`
  - **Selected agent ID:** `381de7ea-0dd9-431b-95d3-76afc7963e1a`
  - **Displayed cached agent name:** `Turn LinkedIn Engagements to Pipeline`
  - **Credential used:** Airtop API credential named **Airtop Templates**

  The node uses manually defined parameter mapping rather than automatic matching. It sends these values to the Airtop agent:

  - `maxLeads`: defaults to `10` if not provided
  - `maxPosts`: defaults to `3` if not provided
  - `outputType`: defaults to `'json'` if not provided
  - `linkedinUrl`: taken directly from input
  - `daysLookback`: defaults to `30` if not provided
  - `prospeoApiKey`: defaults to empty string if not provided

- **Key expressions or variables used:**  
  - `={{ $json.maxLeads || 10 }}`
  - `={{ $json.maxPosts || 3 }}`
  - `={{ $json.outputType || 'json' }}`
  - `={{ $json.linkedinUrl }}`
  - `={{ $json.daysLookback || 30 }}`
  - `={{ $json.prospeoApiKey || '' }}`

  Important expression behavior:
  - The use of `||` means falsy values are replaced by defaults. For example:
    - `0` for `maxLeads`, `maxPosts`, or `daysLookback` would be replaced by the default rather than preserved
    - an empty string for `outputType` or `prospeoApiKey` becomes the fallback value

- **Input and output connections:**  
  - **Input:** from **When Executed by Another Workflow**
  - **Output:** none shown in the workflow JSON; this is the terminal operational node

- **Version-specific requirements (if any):**  
  Uses type version **1** of the Airtop node. The exact available UI fields and agent schema rendering may vary by n8n/Airtop integration version.

- **Edge cases or potential failure types:**  
  - **Authentication failure:** invalid or expired Airtop API credentials
  - **Agent availability failure:** the target Airtop template/agent is not installed in the connected Airtop account
  - **Invalid input:** malformed or unsupported LinkedIn URL
  - **Template mismatch:** if the configured Airtop agent schema changes upstream, this node’s mapped fields may no longer align
  - **Rate limits / remote execution errors:** Airtop service latency, request rejection, or timeout
  - **Prospeo dependency issues:** if the Airtop template expects a valid `prospeoApiKey` for email enrichment and it is blank or invalid, parts of the downstream Airtop process may degrade or fail
  - **Output handling ambiguity:** because no downstream node exists, whatever data Airtop returns becomes the workflow output; consumers should verify the return structure before relying on exact fields
  - **Falsy-value defaulting:** explicit zero values cannot be preserved due to `||` fallback logic

- **Sub-workflow reference:**  
  This node does not invoke an n8n sub-workflow. It invokes an external Airtop agent/template.

---

## Block 3 — Embedded Documentation / Operator Guidance

### Overview

This block contains a sticky note meant for human operators. It explains the business purpose, links to a video resource, and states the critical prerequisite needed before the workflow can run successfully.

### Nodes Involved

- **Sticky Note**

### Node Details

#### 3. Sticky Note

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Visual documentation node; it does not participate in execution.

- **Configuration choices:**  
  The note contains:
  - A heading describing the workflow’s goal
  - A YouTube video reference
  - An important prerequisite about installing the Airtop template:
    `https://www.airtop.ai/templates/linkedin-lead-generation-from-post-commenters-old-3`

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  None.

- **Version-specific requirements (if any):**  
  Uses sticky note type version **1**.

- **Edge cases or potential failure types:**  
  - No runtime effect
  - Operational risk if the note is ignored: the workflow may fail because the Airtop template is not installed

- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Executed by Another Workflow | Execute Workflow Trigger | Receives inputs from a parent workflow and starts this sub-workflow |  | Run "Turn LinkedIn Engagements to Pipeline" | Automatically Turn LinkedIn Comments into New Leads |
| Run "Turn LinkedIn Engagements to Pipeline" | Airtop | Sends LinkedIn and control parameters to the Airtop lead-generation agent | When Executed by Another Workflow |  | Automatically Turn LinkedIn Comments into New Leads |
|  |  |  |  |  | The Airtop agent will take a LinkedIn Profile and find leads from relevant posts. |
|  |  |  |  |  | Video: https://www.youtube.com/watch?v=28javM1Djng |
|  |  |  |  |  | Important: You need to install the Airtop template "Turn LinkedIn Engagements to Pipeline" in your account before executing this n8n workflow. Link: https://www.airtop.ai/templates/linkedin-lead-generation-from-post-commenters-old-3 |
| Sticky Note | Sticky Note | Visual documentation and prerequisite notice |  |  | Automatically Turn LinkedIn Comments into New Leads |
|  |  |  |  |  | The Airtop agent will take a LinkedIn Profile and find leads from relevant posts. |
|  |  |  |  |  | Video: https://www.youtube.com/watch?v=28javM1Djng |
|  |  |  |  |  | Important: You need to install the Airtop template "Turn LinkedIn Engagements to Pipeline" in your account before executing this n8n workflow. Link: https://www.airtop.ai/templates/linkedin-lead-generation-from-post-commenters-old-3 |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **Turn LinkedIn Engagements to Pipeline**
   - Keep it inactive until credentials and Airtop template setup are verified.

2. **Add the trigger node**
   - Add node: **When Executed by Another Workflow**
   - Node type: **Execute Workflow Trigger**
   - In the workflow inputs section, add these inputs:
     1. `linkedinUrl`
     2. `prospeoApiKey`
     3. `daysLookback` with type **Number**
     4. `maxLeads` with type **Number**
     5. `maxPosts` with type **Number**
     6. `outputType`
   - This makes the workflow callable as a sub-workflow from another n8n workflow.

3. **Add the Airtop execution node**
   - Add node: **Airtop**
   - Rename it to: **Run "Turn LinkedIn Engagements to Pipeline"**
   - Connect **When Executed by Another Workflow** → **Run "Turn LinkedIn Engagements to Pipeline"**

4. **Configure Airtop credentials**
   - In the Airtop node, create or select an **Airtop API** credential
   - Use the credential appropriate for the Airtop account where the template is installed
   - In the source workflow, the credential is named **Airtop Templates**, but you may use any name locally

5. **Install the required Airtop template in Airtop first**
   - Before configuring the node fully, ensure the Airtop template exists in the connected Airtop account
   - Required template: **Turn LinkedIn Engagements to Pipeline**
   - Reference link:  
     `https://www.airtop.ai/templates/linkedin-lead-generation-from-post-commenters-old-3`

6. **Select the Airtop resource and agent**
   - In the Airtop node:
     - Set **Resource** to **Agent**
     - Select the agent/template **Turn LinkedIn Engagements to Pipeline**
   - If your UI exposes an agent list, choose the installed template
   - If your environment requires a specific agent ID, use the matching installed agent from your Airtop account rather than hardcoding the ID from this export unless it matches your environment

7. **Configure the Airtop parameter mapping**
   - Choose the mode equivalent to **Define Below**
   - Add the following parameter mappings:

   - **maxLeads**
     - Expression: `{{$json.maxLeads || 10}}`
     - Meaning: default to 10 if omitted or falsy

   - **maxPosts**
     - Expression: `{{$json.maxPosts || 3}}`
     - Meaning: default to 3 if omitted or falsy

   - **outputType**
     - Expression: `{{$json.outputType || 'json'}}`
     - Meaning: default output mode is `json`

   - **linkedinUrl**
     - Expression: `{{$json.linkedinUrl}}`
     - Meaning: required source LinkedIn profile URL

   - **daysLookback**
     - Expression: `{{$json.daysLookback || 30}}`
     - Meaning: default to 30 days if omitted or falsy

   - **prospeoApiKey**
     - Expression: `{{$json.prospeoApiKey || ''}}`
     - Meaning: send empty string if no Prospeo key is supplied

8. **Be aware of expression behavior**
   - Using `||` means these values are replaced by defaults when falsy:
     - `0`
     - `null`
     - `undefined`
     - empty string
   - If you need to preserve `0` as a valid value, use a nullish-coalescing-style expression in n8n instead of `||`, for example with an explicit conditional.

9. **Leave the Airtop node as the last node**
   - No further node is required if you want the Airtop response to become the sub-workflow output
   - In n8n, the last executed node’s output is typically what a parent workflow can consume

10. **Add the documentation sticky note**
    - Add a **Sticky Note** node near the workflow
    - Suggested content:

      - **Automatically Turn LinkedIn Comments into New Leads**
      - The Airtop agent will take a LinkedIn Profile and find leads from relevant posts.
      - Video: `https://www.youtube.com/watch?v=28javM1Djng`
      - Important: You need to install the Airtop template `Turn LinkedIn Engagements to Pipeline` in your account before executing this n8n workflow.
      - Template link: `https://www.airtop.ai/templates/linkedin-lead-generation-from-post-commenters-old-3`

11. **Test with sample data**
    - Use sample inputs such as:
      - `linkedinUrl`: `https://www.linkedin.com/in/antonosika/`
      - `daysLookback`: leave blank or set `30`
      - `maxLeads`: leave blank or set `10`
      - `maxPosts`: leave blank or set `3`
      - `outputType`: leave blank or set `json`
      - `prospeoApiKey`: optional
    - Execute the workflow from a parent workflow or use test/pinned data in the sub-workflow editor.

12. **Create a parent workflow caller if needed**
    - In a separate workflow, add an **Execute Workflow** node
    - Select this workflow
    - Pass the six inputs by name:
      - `linkedinUrl`
      - `prospeoApiKey`
      - `daysLookback`
      - `maxLeads`
      - `maxPosts`
      - `outputType`
    - Ensure `linkedinUrl` is always supplied

13. **Validate the returned structure**
    - Inspect the Airtop node output after execution
    - Confirm:
      - the expected lead list is returned
      - the output format matches `outputType`
      - any downstream workflow can parse the response reliably

14. **Activate only after prerequisite checks**
    - Confirm the Airtop credential works
    - Confirm the template is installed in the same Airtop account
    - Confirm the parent workflow sends a valid LinkedIn URL
    - Then activate the workflow if it is meant for reuse

## Sub-workflow setup expectations

Since this workflow is itself a sub-workflow, its interface should be treated as:

### Required input
- `linkedinUrl`

### Optional inputs
- `prospeoApiKey`
- `daysLookback`
- `maxLeads`
- `maxPosts`
- `outputType`

### Expected output
- The output returned by the Airtop agent
- Exact schema depends on the Airtop template version and output mode

### External dependency
- Installed Airtop template: **Turn LinkedIn Engagements to Pipeline**
- Airtop API credential
- Potential optional Prospeo API key, depending on the enrichment behavior of the Airtop template

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically Turn LinkedIn Comments into New Leads | Workflow purpose |
| The Airtop agent will take a LinkedIn Profile and find leads from relevant posts. | Operator guidance |
| Video resource | https://www.youtube.com/watch?v=28javM1Djng |
| Required Airtop template: Turn LinkedIn Engagements to Pipeline | https://www.airtop.ai/templates/linkedin-lead-generation-from-post-commenters-old-3 |
| Important operational prerequisite: install the Airtop template in the Airtop account before running the workflow | Airtop dependency/setup |