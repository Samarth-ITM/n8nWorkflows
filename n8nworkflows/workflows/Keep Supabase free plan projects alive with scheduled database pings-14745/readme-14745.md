Keep Supabase free plan projects alive with scheduled database pings

https://n8nworkflows.xyz/workflows/keep-supabase-free-plan-projects-alive-with-scheduled-database-pings-14745


# Keep Supabase free plan projects alive with scheduled database pings

# 1. Workflow Overview

This workflow keeps a Supabase project on the free plan active by periodically writing small “ping” records into a database table. Its main purpose is to avoid inactivity-based pausing by simulating light database usage every few days.

It has two entry points:
- a scheduled trigger that runs automatically every 4 days
- a manual trigger for on-demand testing

The workflow is logically divided into the following blocks:

## 1.1 Trigger and Batch Initialization

This block starts the workflow either automatically or manually, then creates a fixed batch of 25 items. These items are not business records; they exist purely to drive repeated ping operations.

## 1.2 Randomized Ping Execution Loop

This block iterates through the generated items one at a time, assigns a random wait time between 20 and 60 seconds for each iteration, pauses, and then inserts a row into Supabase. The loop continues until all 25 iterations are processed.

## 1.3 Documentation and Setup Notes

This block consists of sticky notes embedded in the canvas. They document the workflow purpose, setup requirements, and warnings for anyone maintaining or reusing the workflow.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Batch Initialization

### Overview

This block provides the workflow’s two execution entry points and prepares the repeated workload. It converts a single trigger event into 25 synthetic items so the downstream loop can perform multiple database writes per run.

### Nodes Involved

- Schedule Trigger
- When clicking ‘Execute workflow’
- Code in JavaScript
- Sticky Note
- Sticky Note3

### Node Details

#### Schedule Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Automatic entry point that starts the workflow on a recurring interval.
- **Configuration choices:**  
  Configured to run every 4 days.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input. Outputs to **Code in JavaScript**.
- **Version-specific requirements:**  
  Uses type version `1.3`.
- **Edge cases or potential failure types:**  
  - Workflow must be activated, otherwise the schedule will not run.
  - Instance timezone may affect the perceived run schedule.
  - If n8n is down at the scheduled time, execution behavior depends on hosting/runtime setup.
- **Sub-workflow reference:**  
  None.

#### When clicking ‘Execute workflow’

- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for testing the workflow directly from the editor.
- **Configuration choices:**  
  Default manual trigger configuration.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input. Outputs to **Code in JavaScript**.
- **Version-specific requirements:**  
  Uses type version `1`.
- **Edge cases or potential failure types:**  
  - Only runs when manually executed in the n8n editor.
  - Does not replace activation of the scheduled version.
- **Sub-workflow reference:**  
  None.

#### Code in JavaScript

- **Type and technical role:** `n8n-nodes-base.code`  
  Generates an array of 25 items to drive the loop.
- **Configuration choices:**  
  JavaScript returns 25 objects, each with:
  - `json.value = 1`
- **Key expressions or variables used:**  
  The code is effectively:
  - create an array of length 25
  - map each element into `{ json: { value: 1 } }`
- **Input and output connections:**  
  Input from **Schedule Trigger** and **When clicking ‘Execute workflow’**.  
  Output to **Loop Over Items**.
- **Version-specific requirements:**  
  Uses code node type version `2`.
- **Edge cases or potential failure types:**  
  - JavaScript syntax errors would stop execution.
  - If the item count is increased significantly, the run duration also increases because of downstream waits.
- **Sub-workflow reference:**  
  None.

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note.
- **Configuration choices:**  
  Content explains trigger and batch setup:
  - Starts the workflow on a schedule or manually
  - Generates 25 items to drive the ping loop
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses type version `1`.
- **Edge cases or potential failure types:**  
  None; informational only.
- **Sub-workflow reference:**  
  None.

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Main canvas documentation note with setup and customization guidance.
- **Configuration choices:**  
  Includes:
  - workflow purpose
  - detailed operation steps
  - setup checklist
  - customization guidance
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses type version `1`.
- **Edge cases or potential failure types:**  
  None; informational only.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Randomized Ping Execution Loop

### Overview

This block processes the 25 generated items sequentially. For each item, it generates a random delay, waits for that duration, inserts a timestamped row into Supabase, and then returns control to the batch loop until all items are exhausted.

### Nodes Involved

- Loop Over Items
- Code in JavaScript1
- Wait
- Create a row
- Sticky Note1
- Sticky Note2

### Node Details

#### Loop Over Items

- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Controls iterative batch processing, here effectively one item at a time.
- **Configuration choices:**  
  No custom options are set. It is used as a loop controller.
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  Input from **Code in JavaScript**.  
  Second output goes to **Code in JavaScript1** for each iteration.  
  Input back from **Create a row** to continue the loop.  
  First output is unused in this workflow.
- **Version-specific requirements:**  
  Uses type version `3`.
- **Edge cases or potential failure types:**  
  - Misunderstanding the two outputs can break the loop design.
  - If upstream produces zero items, the loop body will not execute.
  - If downstream fails mid-loop, remaining items will not be processed.
- **Sub-workflow reference:**  
  None.

#### Code in JavaScript1

- **Type and technical role:** `n8n-nodes-base.code`  
  Enriches each loop item with a random number of seconds to wait.
- **Configuration choices:**  
  Maps all incoming items and adds:
  - `seconds = random integer between 20 and 60`
- **Key expressions or variables used:**  
  The code computes:
  - `Math.floor(Math.random() * (60 - 20 + 1)) + 20`
- **Input and output connections:**  
  Input from **Loop Over Items**.  
  Output to **Wait**.
- **Version-specific requirements:**  
  Uses code node type version `2`.
- **Edge cases or potential failure types:**  
  - JavaScript syntax/runtime errors would stop execution.
  - If the wait range is edited incorrectly, it may produce invalid or negative durations.
- **Sub-workflow reference:**  
  None.

#### Wait

- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses each loop iteration before the database write.
- **Configuration choices:**  
  Wait amount is dynamically set from the current item:
  - `={{ $json.seconds }}`
- **Key expressions or variables used:**  
  - `$json.seconds`
- **Input and output connections:**  
  Input from **Code in JavaScript1**.  
  Output to **Create a row**.
- **Version-specific requirements:**  
  Uses type version `1.1`.
- **Edge cases or potential failure types:**  
  - If `seconds` is missing or non-numeric, the node may fail.
  - Long waits increase total execution time substantially.
  - Wait nodes may behave differently depending on execution mode and persistence settings in self-hosted n8n.
- **Sub-workflow reference:**  
  None.

#### Create a row

- **Type and technical role:** `n8n-nodes-base.supabase`  
  Inserts a row into a Supabase table.
- **Configuration choices:**  
  - Operation: create row
  - Table: `ping`
  - Field set:
    - `created_at = {{ $now }}`
  - Credential used: Supabase API credential named **Supabase Lumizone**
- **Key expressions or variables used:**  
  - `$now`
- **Input and output connections:**  
  Input from **Wait**.  
  Output back to **Loop Over Items** to continue processing the remaining items.
- **Version-specific requirements:**  
  Uses type version `1`.
- **Edge cases or potential failure types:**  
  - Authentication failure if Supabase credentials are missing or invalid.
  - Table `ping` must exist.
  - Column `created_at` must exist and accept the provided timestamp format.
  - Row-level security or API permission settings may block inserts.
  - Network/API errors can interrupt the loop.
- **Sub-workflow reference:**  
  None.

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note for the loop section.
- **Configuration choices:**  
  Explains:
  - iteration over each item
  - random interval between 20 and 60 seconds
  - writing a row to Supabase
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses type version `1`.
- **Edge cases or potential failure types:**  
  None; informational only.
- **Sub-workflow reference:**  
  None.

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual warning label near the Supabase insertion area.
- **Configuration choices:**  
  Contains only the text:
  - `## Warning`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses type version `1`.
- **Edge cases or potential failure types:**  
  None; informational only, though the content is very minimal.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Documentation and Setup Notes

### Overview

This block contains embedded operational guidance. It explains the goal of the workflow, how to configure Supabase, and what to customize if the default ping strategy is not appropriate.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### Sticky Note

- See section 2.1.

#### Sticky Note1

- See section 2.2.

#### Sticky Note2

- See section 2.2.

#### Sticky Note3

- See section 2.1.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point every 4 days |  | Code in JavaScript | ## Trigger and batch setup\nStarts the workflow on a schedule or manually, then generates 25 items to drive the ping loop. |
| Create a row | n8n-nodes-base.supabase | Inserts a timestamp row into Supabase `ping` table | Wait | Loop Over Items | ## Randomised ping loop\nIterates over each item, waits a random interval between 20 and 60 seconds, then writes a row to Supabase.\n## Warning |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual workflow entry point for testing |  | Code in JavaScript | ## Trigger and batch setup\nStarts the workflow on a schedule or manually, then generates 25 items to drive the ping loop. |
| Code in JavaScript | n8n-nodes-base.code | Generates 25 loop-driving items | Schedule Trigger, When clicking ‘Execute workflow’ | Loop Over Items | ## Trigger and batch setup\nStarts the workflow on a schedule or manually, then generates 25 items to drive the ping loop. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Controls sequential iteration through generated items | Code in JavaScript, Create a row | Code in JavaScript1 | ## Randomised ping loop\nIterates over each item, waits a random interval between 20 and 60 seconds, then writes a row to Supabase. |
| Code in JavaScript1 | n8n-nodes-base.code | Adds random wait duration to each item | Loop Over Items | Wait | ## Randomised ping loop\nIterates over each item, waits a random interval between 20 and 60 seconds, then writes a row to Supabase. |
| Wait | n8n-nodes-base.wait | Delays each loop iteration using random seconds | Code in JavaScript1 | Create a row | ## Randomised ping loop\nIterates over each item, waits a random interval between 20 and 60 seconds, then writes a row to Supabase. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for trigger setup |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for ping loop |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Warning label near insert node |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Full workflow description, setup checklist, customization guidance |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure interval mode to run every `4 days`
   - This will be the automatic entry point

3. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Keep default settings
   - This will be the testing entry point when using “Execute workflow”

4. **Add a Code node named `Code in JavaScript`**
   - Node type: `Code`
   - Language: JavaScript
   - Configure it to return 25 items, each shaped like:
     - `json.value = 1`
   - Equivalent logic:
     - create 25 objects
     - return them as workflow items
   - Connect:
     - `Schedule Trigger` → `Code in JavaScript`
     - `Manual Trigger` → `Code in JavaScript`

5. **Add a Split In Batches node named `Loop Over Items`**
   - Node type: `Loop Over Items` / `Split In Batches`
   - Leave default options
   - Connect:
     - `Code in JavaScript` → `Loop Over Items`

6. **Add a second Code node named `Code in JavaScript1`**
   - Node type: `Code`
   - Language: JavaScript
   - Configure it to add a new field called `seconds` to each incoming item
   - Set `seconds` to a random integer between `20` and `60`
   - Connect:
     - second/main loop output of `Loop Over Items` → `Code in JavaScript1`

7. **Add a Wait node**
   - Node type: `Wait`
   - Set the wait amount dynamically using an expression:
     - `{{ $json.seconds }}`
   - Ensure the node is configured to wait by amount, using the incoming value
   - Connect:
     - `Code in JavaScript1` → `Wait`

8. **Add a Supabase node named `Create a row`**
   - Node type: `Supabase`
   - Operation: create/insert row
   - Table name: `ping`
   - Add one field:
     - `created_at` = `{{ $now }}`
   - Connect:
     - `Wait` → `Create a row`

9. **Close the loop**
   - Connect:
     - `Create a row` → `Loop Over Items`
   - This sends control back to the loop controller so the next item is processed

10. **Create the Supabase table**
    - In Supabase, create a table named `ping`
    - Add a column named `created_at`
    - Ensure the column type supports timestamps
    - Make sure the API role used by n8n has permission to insert rows

11. **Configure Supabase credentials in n8n**
    - Open the `Create a row` node
    - Create or select a `Supabase API` credential
    - Provide the required project URL and API key/token
    - Validate that the credential can access the target project

12. **Add optional sticky notes for maintainability**
    - Add a note for trigger/batch setup:
      - “Starts the workflow on a schedule or manually, then generates 25 items to drive the ping loop.”
    - Add a note for the randomized loop:
      - “Iterates over each item, waits a random interval between 20 and 60 seconds, then writes a row to Supabase.”
    - Add a warning note near the Supabase node
    - Add a larger note summarizing:
      - purpose
      - setup checklist
      - customization options

13. **Test manually**
    - Use the manual trigger
    - Confirm:
      - 25 items are generated
      - each item gets a `seconds` value
      - waits occur
      - rows are inserted into Supabase

14. **Activate the workflow**
    - Turn workflow activation on
    - The schedule will then run every 4 days automatically

## Expected runtime behavior

- One workflow run creates 25 rows in total
- Each row is delayed by a random 20–60 second wait
- Total execution time can be several minutes because waits are cumulative
- The inserted timestamp comes from n8n’s current runtime clock via `$now`

## Constraints and implementation notes

- If you increase the number of generated items, total run time increases proportionally.
- If you reduce the wait interval too much, the activity may look more artificial.
- If the `ping` table has strict validation, ensure `created_at` accepts the incoming value.
- If you use Row Level Security in Supabase, verify insert permissions for the API key in use.

## Sub-workflow setup

This workflow does **not** use any sub-workflow nodes and does not invoke other workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Supabase free plan keep-alive workflow: schedule runs every 4 days, generates 25 items, randomizes 20–60 second delays, and inserts into `ping.created_at`. | General workflow purpose |
| Setup requirements: add Supabase credentials to the Create a row node, create a `ping` table with `created_at`, and activate the workflow. | Operational setup |
| Customization: adjust the Schedule Trigger interval or change the generated item count in the first Code node. | Maintenance and scaling |
| The workflow contains no external links, no branding links, and no referenced videos. | General observation |