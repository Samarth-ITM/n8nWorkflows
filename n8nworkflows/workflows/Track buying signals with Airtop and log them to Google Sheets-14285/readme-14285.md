Track buying signals with Airtop and log them to Google Sheets

https://n8nworkflows.xyz/workflows/track-buying-signals-with-airtop-and-log-them-to-google-sheets-14285


# Track buying signals with Airtop and log them to Google Sheets

# 1. Workflow Overview

This workflow is a very compact sub-workflow designed to be called by another n8n workflow. Its purpose is to forward a set of buying-signal monitoring parameters to an Airtop agent named **“The Real-Time Buying Signals Agent”**, which then performs the actual monitoring and logging process, including writing results to Google Sheets.

Typical use cases include:

- Monitoring target accounts for buying signals such as:
  - hiring activity
  - LinkedIn post activity
  - company news
  - product launches
  - funding-related events
- Passing runtime options from a parent workflow
- Centralizing Airtop agent execution behind a reusable n8n sub-workflow

The workflow is organized into the following logical blocks:

## 1.1 Input Reception
This block receives all runtime parameters from a parent workflow through an **Execute Workflow Trigger** node. These inputs define what the Airtop agent should process and how far back it should search.

## 1.2 Airtop Agent Execution
This block invokes the Airtop agent with the values received from the trigger node. The Airtop agent is responsible for the actual buying-signal monitoring logic and downstream logging to Google Sheets.

## 1.3 Embedded Documentation
A sticky note provides operational context, including the intended workflow purpose, a video link, and an important prerequisite: the Airtop template must already be installed in the Airtop account.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block exposes the workflow as a callable sub-workflow. It defines the parameter interface expected from a parent workflow and makes those values available as JSON for downstream nodes.

### Nodes Involved
- **When Executed by Another Workflow**

### Node Details

#### When Executed by Another Workflow
- **Type and technical role:**  
  `n8n-nodes-base.executeWorkflowTrigger`  
  This is a sub-workflow entry node. It starts execution only when another workflow calls this workflow via an Execute Workflow-type mechanism.

- **Configuration choices:**  
  The node defines a structured input schema using workflow inputs. The following parameters are declared:
  - `googleSheetUrl`
  - `deduplicationWindowDays`
  - `enableHiringModule`
  - `enableLinkedInPostsModule`
  - `enableNewsModule`
  - `maxAccountsToProcess`
  - `maxJobPagesPerCompany`
  - `maxPostPagesPerCompany`
  - `newsDaysBack`
  - `postsDaysBack`

  Types explicitly set in the node:
  - Numbers:
    - `deduplicationWindowDays`
    - `maxAccountsToProcess`
    - `maxJobPagesPerCompany`
    - `maxPostPagesPerCompany`
    - `newsDaysBack`
    - `postsDaysBack`
  - Booleans:
    - `enableHiringModule`
    - `enableLinkedInPostsModule`
    - `enableNewsModule`
  - String/default text:
    - `googleSheetUrl`

- **Key expressions or variables used:**  
  This node itself does not use expressions; it emits input values as JSON fields such as:
  - `$json.googleSheetUrl`
  - `$json.newsDaysBack`
  - `$json.postsDaysBack`
  - etc.

- **Input and output connections:**  
  - **Input:** none, as this is the workflow entry point
  - **Output:** connected to **Run "The Real-Time Buying Signals Agent"**

- **Version-specific requirements (if any):**  
  Uses `typeVersion: 1.1`, which corresponds to the newer workflow-input behavior for sub-workflows. Reproduction should be done in a version of n8n that supports workflow input definitions on the Execute Workflow Trigger node.

- **Edge cases or potential failure types:**  
  - If the parent workflow does not provide `googleSheetUrl`, downstream Airtop execution may fail because Airtop marks it as required.
  - Type mismatches from the calling workflow may lead to malformed parameters or Airtop validation errors.
  - Null values may pass through unless explicitly guarded in the parent workflow.
  - Boolean fields may behave unexpectedly if provided as strings instead of actual booleans by the parent workflow.

- **Sub-workflow reference:**  
  This node indicates that the current workflow is intended to be used as a sub-workflow callable by another workflow.

---

## 2.2 Airtop Agent Execution

### Overview
This block forwards the received parameters to an Airtop agent template. The agent encapsulates the real business logic for scanning sources and logging buying signals to Google Sheets.

### Nodes Involved
- **Run "The Real-Time Buying Signals Agent"**

### Node Details

#### Run "The Real-Time Buying Signals Agent"
- **Type and technical role:**  
  `n8n-nodes-base.airtop`  
  This node calls an Airtop agent resource, specifically a preconfigured template agent.

- **Configuration choices:**  
  - **Resource:** `agent`
  - **Agent selected:** `The Real-Time Buying Signals Agent`
  - **Agent ID:** `85d35fb5-0610-4dd0-bb39-9f1ed310d24e`
  - **Credential used:** `Airtop Templates`

  The node maps incoming workflow fields into Airtop agent parameters using explicit field definitions.

- **Key expressions or variables used:**  
  The node passes through values from the trigger node with expressions:
  - `newsDaysBack = {{ $json.newsDaysBack }}`
  - `postsDaysBack = {{ $json.postsDaysBack }}`
  - `googleSheetUrl = {{ $json.googleSheetUrl }}`
  - `enableNewsModule = {{ $json.enableNewsModule }}`
  - `enableHiringModule = {{ $json.enableHiringModule }}`
  - `maxAccountsToProcess = {{ $json.maxAccountsToProcess }}`
  - `maxJobPagesPerCompany = {{ $json.maxJobPagesPerCompany }}`
  - `maxPostPagesPerCompany = {{ $json.maxPostPagesPerCompany }}`
  - `deduplicationWindowDays = {{ $json.deduplicationWindowDays }}`
  - `enableLinkedInPostsModule = {{ $json.enableLinkedInPostsModule }}`

- **Input and output connections:**  
  - **Input:** from **When Executed by Another Workflow**
  - **Output:** none shown in this workflow; this is the terminal operational node

- **Version-specific requirements (if any):**  
  Uses `typeVersion: 1`.  
  The Airtop node must be available in the n8n instance. This may require a recent n8n version and/or the Airtop integration package supported by the environment.

- **Edge cases or potential failure types:**  
  - Missing or invalid Airtop credentials
  - Airtop agent not installed in the linked Airtop account
  - Agent ID mismatch or deleted template
  - `googleSheetUrl` missing, null, or malformed
  - Invalid numeric values, such as negative values where unsupported by Airtop
  - Boolean parameters not passed in proper boolean form
  - Airtop API/network timeouts or rate limits
  - External failures inside the Airtop template, including:
    - LinkedIn/news source access issues
    - Google Sheets write failures handled inside Airtop rather than n8n
  - If the Google Sheet is not shared or accessible as expected by the Airtop process, logging may fail even if the n8n node itself starts correctly

- **Sub-workflow reference:**  
  This node invokes an external Airtop agent, not another n8n workflow.

---

## 2.3 Embedded Documentation

### Overview
This block contains only a sticky note, but it is operationally important. It explains the workflow’s purpose and documents an external prerequisite for successful execution.

### Nodes Involved
- **Sticky Note**

### Node Details

#### Sticky Note
- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  A non-executable documentation node used to annotate the workflow canvas.

- **Configuration choices:**  
  The note contains:
  - Workflow title
  - Short business description
  - Video link
  - Important setup requirement

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  None.

- **Version-specific requirements (if any):**  
  No meaningful version constraints beyond standard sticky note support.

- **Edge cases or potential failure types:**  
  None technically, but users may miss the prerequisite documented here:
  - The Airtop template **Real-Time Buying Signals Agent** must be installed in the Airtop account before execution.

- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Executed by Another Workflow | Execute Workflow Trigger | Receives runtime inputs from a parent workflow and starts the sub-workflow |  | Run "The Real-Time Buying Signals Agent" | ## Track Buying Signals with Airtop Automatically monitor target accounts across LinkedIn and news for hiring sprees, product launches, funding rounds, and other buying signals - logged straight to Google Sheets. Video Tutorial: https://www.youtube.com/watch?v=VYD7X3It2v8 Important: You need to install the Airtop template [Real-Time Buying Signals Agent](https://www.airtop.ai/templates/multi-source-b2b-buying-signal-monitor?utm_source=n8n-template) in your account before executing this n8n workflow. |
| Sticky Note | Sticky Note | Documents purpose, setup instructions, and external resource links |  |  | ## Track Buying Signals with Airtop Automatically monitor target accounts across LinkedIn and news for hiring sprees, product launches, funding rounds, and other buying signals - logged straight to Google Sheets. Video Tutorial: https://www.youtube.com/watch?v=VYD7X3It2v8 Important: You need to install the Airtop template [Real-Time Buying Signals Agent](https://www.airtop.ai/templates/multi-source-b2b-buying-signal-monitor?utm_source=n8n-template) in your account before executing this n8n workflow. |
| Run "The Real-Time Buying Signals Agent" | Airtop | Calls the Airtop buying-signals agent with runtime configuration | When Executed by Another Workflow |  | ## Track Buying Signals with Airtop Automatically monitor target accounts across LinkedIn and news for hiring sprees, product launches, funding rounds, and other buying signals - logged straight to Google Sheets. Video Tutorial: https://www.youtube.com/watch?v=VYD7X3It2v8 Important: You need to install the Airtop template [Real-Time Buying Signals Agent](https://www.airtop.ai/templates/multi-source-b2b-buying-signal-monitor?utm_source=n8n-template) in your account before executing this n8n workflow. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a new workflow.
   - Give it the title: **Track buying signals with Airtop and log them to Google Sheets**.

2. **Add the sub-workflow trigger node**
   - Add a node of type **Execute Workflow Trigger**.
   - Rename it to: **When Executed by Another Workflow**.

3. **Define workflow inputs on the trigger node**
   Add the following workflow inputs in the trigger configuration:
   - `googleSheetUrl`
   - `deduplicationWindowDays` as **Number**
   - `enableHiringModule` as **Boolean**
   - `enableLinkedInPostsModule` as **Boolean**
   - `enableNewsModule` as **Boolean**
   - `maxAccountsToProcess` as **Number**
   - `maxJobPagesPerCompany` as **Number**
   - `maxPostPagesPerCompany` as **Number**
   - `newsDaysBack` as **Number**
   - `postsDaysBack` as **Number**

4. **Optionally set test data for validation**
   For local testing in the sub-workflow editor, use values equivalent to the pinned data:
   - `newsDaysBack`: `30`
   - `postsDaysBack`: `7`
   - `googleSheetUrl`: provide a real Google Sheet URL
   - `enableNewsModule`: `true`
   - `enableHiringModule`: `true`
   - `maxAccountsToProcess`: `-1`
   - `maxJobPagesPerCompany`: `3`
   - `maxPostPagesPerCompany`: `1`
   - `deduplicationWindowDays`: `7`
   - `enableLinkedInPostsModule`: `true`

   Note: the source JSON shows `googleSheetUrl` as null in pinned data, but in practice the Airtop agent expects this field.

5. **Add the Airtop node**
   - Add a node of type **Airtop**.
   - Rename it to: **Run "The Real-Time Buying Signals Agent"**.

6. **Configure the Airtop node resource**
   - Set **Resource** to **Agent**.
   - Select the agent named **The Real-Time Buying Signals Agent**.
   - If your environment requires it, ensure it resolves to the Airtop agent ID:
     - `85d35fb5-0610-4dd0-bb39-9f1ed310d24e`

7. **Configure Airtop credentials**
   - Create or select Airtop API credentials.
   - The exported workflow uses a credential named: **Airtop Templates**.
   - Make sure the Airtop account tied to these credentials has the template installed:
     - [Real-Time Buying Signals Agent](https://www.airtop.ai/templates/multi-source-b2b-buying-signal-monitor?utm_source=n8n-template)

8. **Map the Airtop agent parameters**
   In the Airtop node, configure the agent parameters using expressions from the trigger node output:

   - `newsDaysBack` → `{{ $json.newsDaysBack }}`
   - `postsDaysBack` → `{{ $json.postsDaysBack }}`
   - `googleSheetUrl` → `{{ $json.googleSheetUrl }}`
   - `enableNewsModule` → `{{ $json.enableNewsModule }}`
   - `enableHiringModule` → `{{ $json.enableHiringModule }}`
   - `maxAccountsToProcess` → `{{ $json.maxAccountsToProcess }}`
   - `maxJobPagesPerCompany` → `{{ $json.maxJobPagesPerCompany }}`
   - `maxPostPagesPerCompany` → `{{ $json.maxPostPagesPerCompany }}`
   - `deduplicationWindowDays` → `{{ $json.deduplicationWindowDays }}`
   - `enableLinkedInPostsModule` → `{{ $json.enableLinkedInPostsModule }}`

9. **Preserve parameter types**
   - Do not force string conversion.
   - Keep numeric fields as numbers.
   - Keep boolean fields as booleans.
   - If your Airtop node has advanced options similar to the exported workflow:
     - disable automatic type conversion if not needed
     - disable convert-to-string behavior

10. **Connect the nodes**
    - Connect **When Executed by Another Workflow** → **Run "The Real-Time Buying Signals Agent"**

11. **Add the sticky note**
    - Add a **Sticky Note** node near the flow.
    - Paste this content:

    **Track Buying Signals with Airtop**

    Automatically monitor target accounts across LinkedIn and news for hiring sprees, product launches, funding rounds, and other buying signals - logged straight to Google Sheets.

    Video Tutorial:  
    https://www.youtube.com/watch?v=VYD7X3It2v8

    Important:  
    You need to install the Airtop template Real-Time Buying Signals Agent in your account before executing this n8n workflow.  
    https://www.airtop.ai/templates/multi-source-b2b-buying-signal-monitor?utm_source=n8n-template

12. **Activate or save the workflow**
    - Save the workflow.
    - This workflow does not need a standalone trigger because it is meant to be called by another workflow.

13. **Configure the parent workflow**
    Since this workflow is a sub-workflow, create or update a parent workflow that calls it using an Execute Workflow-type node.
    The parent workflow should pass these inputs:
    - `googleSheetUrl` as a valid Google Sheets URL
    - `deduplicationWindowDays` as number
    - `enableHiringModule` as boolean
    - `enableLinkedInPostsModule` as boolean
    - `enableNewsModule` as boolean
    - `maxAccountsToProcess` as number
    - `maxJobPagesPerCompany` as number
    - `maxPostPagesPerCompany` as number
    - `newsDaysBack` as number
    - `postsDaysBack` as number

14. **Validate runtime assumptions**
    Before production use, verify:
    - The Airtop agent exists and is accessible with the selected credentials
    - The Google Sheet URL is valid
    - The downstream Airtop template has permission and configuration necessary to log results
    - The boolean toggles match the modules you want enabled
    - Limits like `maxAccountsToProcess`, `maxJobPagesPerCompany`, and `maxPostPagesPerCompany` are acceptable to the Airtop template

15. **Test end-to-end**
    - Run the parent workflow or test the sub-workflow with mock input values.
    - Confirm the Airtop node executes successfully.
    - Verify that signals are written to the intended Google Sheet by the Airtop process.

### Sub-workflow setup summary
- **Entry mechanism:** Execute Workflow Trigger
- **Expected inputs:** 10 parameters listed above
- **Expected output:** No explicit n8n post-processing is defined in this workflow; the core effect is the Airtop agent execution and its external logging behavior

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video walkthrough for the workflow | https://www.youtube.com/watch?v=VYD7X3It2v8 |
| Required Airtop template: Real-Time Buying Signals Agent | https://www.airtop.ai/templates/multi-source-b2b-buying-signal-monitor?utm_source=n8n-template |
| The workflow itself does not write directly to Google Sheets through an n8n Google Sheets node; instead, it delegates that responsibility to the Airtop agent. | Operational architecture |
| This is a sub-workflow, not a standalone entry flow. It must be invoked by another workflow. | Implementation note |
| The provided pinned data suggests default-like test values, including `newsDaysBack=30`, `postsDaysBack=7`, `deduplicationWindowDays=7`, `maxJobPagesPerCompany=3`, and `maxPostPagesPerCompany=1`. | Testing and validation context |