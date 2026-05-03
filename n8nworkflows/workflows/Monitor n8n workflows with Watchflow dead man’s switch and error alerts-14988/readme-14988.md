Monitor n8n workflows with Watchflow dead man’s switch and error alerts

https://n8nworkflows.xyz/workflows/monitor-n8n-workflows-with-watchflow-dead-man-s-switch-and-error-alerts-14988


# Monitor n8n workflows with Watchflow dead man’s switch and error alerts

Good, now I have all the information verified. Let me compose the complete document. 1. Workflow Overview

This workflow demonstrates **n8n monitoring best practices** using the Watchflow platform. It implements three complementary observability patterns for any n8n automation:

1. **Dead Man's Switch** – Ensures at least one item is successfully processed on schedule. If a trigger fails silently or zero data is processed, no success heartbeat is sent, and Watchflow raises an alert.
2. **Performance Transparency** – Aggregates the exact count of processed items and sends that metric to Watchflow, making it impossible for a "successful but empty" execution to go unnoticed.
3. **Instant Incident Alerting** – Uses n8n's global `Error Trigger` to catch any unexpected node failure and immediately report the full error context (including a direct execution link) to Watchflow.

The workflow is structured as a simulated ETL pipeline (extract → transform → load) followed by an aggregation-and-validation step that conditionally sends a success ping. A parallel error-handling branch captures failures globally.

### Logical Blocks

| Block | Purpose |
|---|---|
| **1.1 Input Reception** | Manual trigger to start the simulated pipeline |
| **1.2 Data Source (Extraction)** | Generate a static array of items simulating extracted records |
| **1.3 Data Transformation** | Split the array into individual items and assign labels |
| **1.4 Data Load Simulation** | No-op node representing a final sync step |
| **1.5 Aggregation & Validation** | Count all processed items and decide whether to report success |
| **1.6 Success Reporting** | Send a success heartbeat with item-count metric to Watchflow |
| **1.7 Global Error Handling & Failure Reporting** | Catch any unhandled error and report it to Watchflow instantly |

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception

**Overview:** A manual trigger node that allows the user to start the workflow on demand. It is the sole entry point for the simulated pipeline.

**Nodes Involved:**
- When clicking 'Execute workflow'

**Node Details — When clicking 'Execute workflow'**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.manualTrigger` (v1) |
| Role | Entry point; fires execution when the user clicks "Execute workflow" in the n8n editor |
| Configuration | No parameters required |
| Input | None (trigger node) |
| Output | Single empty item, connected to **Get Items** |
| Edge Cases | None; this is a simple trigger |
| Sticky Note | "Run this workflow manually" |

---

### Block 1.2 — Data Source (Extraction)

**Overview:** Simulates the extraction phase of an ETL pipeline. A Set node creates a static array of nine numeric values representing extracted records, and a SplitOut node fans the array into individual items for downstream processing.

**Nodes Involved:**
- Get Items
- Split Out Items

**Node Details — Get Items**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.set` (v3.4) |
| Role | Creates a mock data payload for the pipeline |
| Configuration | Single assignment: field `data`, type `array`, value `[1,2,3,4,5,6,7,8,9]` |
| Key Expressions | Hard-coded array literal; no dynamic expression |
| Input | From **When clicking 'Execute workflow'** |
| Output | One item containing `{ data: [1,2,3,4,5,6,7,8,9] }` |
| Edge Cases | If modified to use a real data source (e.g., an HTTP Request), the source could return an empty array, which the downstream validation will catch |
| Sticky Note | "Dummy Data Source — Your dummy workflow data extraction source: In this step, we exemplary fetch 20 items from a source to simulate a real-world data stream for processing and monitoring." |

**Node Details — Split Out Items**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.splitOut` (v1) |
| Role | Expands the single-item array field into multiple items, one per element |
| Configuration | `fieldToSplitOut`: `data` (the array field created by Get Items) |
| Key Expressions | Static field name reference |
| Input | From **Get Items** (single item with `data` array) |
| Output | Nine items, each containing `{ data: <number> }` where `<number>` is an element from the original array |
| Edge Cases | If the source array is empty, SplitOut produces zero output items; downstream nodes receive nothing |
| Sticky Note | Same as Get Items ("Dummy Data Source") |

---

### Block 1.3 — Data Transformation

**Overview:** Simulates business-logic transformation. Each individual item receives a human-readable label string derived from its data value.

**Nodes Involved:**
- Set Label

**Node Details — Set Label**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.set` (v3.4) |
| Role | Adds a `label` field to each item to represent processed output |
| Configuration | Single assignment: field `label`, type `string`, value `=Item #{{ $json.data }}` |
| Key Expressions | `$json.data` references the current item's `data` field (the numeric value from the split) |
| Input | From **Split Out Items** (multiple items, each with `{ data: <number> }`) |
| Output | Multiple items, each with `{ data: <number>, label: "Item #<number>" }` |
| Edge Cases | If `data` is undefined or null for any item, the expression will produce `Item #undefined`; ensure upstream nodes always provide the expected structure |
| Sticky Note | "Dummy Data Transformation — Dummy data transformation: This node simulates your business logic by processing the extracted items and preparing them for the final monitoring step." |

---

### Block 1.4 — Data Load Simulation

**Overview:** Represents the final step in the simulated pipeline where processed data would be synced to a target system (CRM, database, etc.). In this template, it is a no-op pass-through.

**Nodes Involved:**
- No Operation, do nothing

**Node Details — No Operation, do nothing**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.noOp` (v1) |
| Role | Placeholder for a real "load" step (e.g., HTTP Request, database insert). Passes all items through unchanged. |
| Configuration | No parameters |
| Input | From **Set Label** |
| Output | Same items, unchanged, forwarded to **Aggregate** |
| Edge Cases | None for the no-op itself; in a production workflow, replace with an actual integration node that could fail and be caught by the Error Trigger |
| Sticky Note | "Dummy Data Load — This node simulates the final step where your processed data is synced to a target system (like a CRM or Database), confirming the functional success of the workflow." |

---

### Block 1.5 — Aggregation & Validation

**Overview:** Collects all processed items into a single aggregated result, then checks whether at least one item was processed. This is the core of the dead man's switch logic.

**Nodes Involved:**
- Aggregate
- If

**Node Details — Aggregate**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.aggregate` (v1) |
| Role | Combines all items back into one, collecting the `data` field values into an array for metric reporting |
| Configuration | `aggregate`: `aggregateAllItemData` — merges all input items' data into a single output item |
| Key Expressions | None (built-in aggregation) |
| Input | From **No Operation, do nothing** |
| Output | Single item containing `{ data: [1, 2, 3, 4, 5, 6, 7, 8, 9] }` (or whatever values flowed through) |
| Edge Cases | If zero items arrive (e.g., the source returned an empty set), the Aggregate node produces no output item, and the If node is never reached — which is exactly the dead man's switch behavior |
| Sticky Note | "Workflow Monitoring & Data Aggregation — This node uses an Aggregate pattern to collect all processed items into a single count. By calculating the total amount of successful syncs, you can send precise metrics to Watchflow to ensure your workflow didn't just 'run,' but actually delivered results." |

**Node Details — If**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.if` (v2.2) |
| Role | Validates that the aggregated data contains at least one item before sending a success ping |
| Configuration | Single condition, combinator `and`, condition type `boolean` `equals`; left value: `={{ $json.data.length > 0 }}`, right value: `={{true}}` |
| Key Expressions | `$json.data.length > 0` — evaluates whether the aggregated data array has any elements |
| Input | From **Aggregate** |
| Output (true branch) | To **Mark job as successful** |
| Output (false branch) | Not connected — intentionally. If zero items were processed, no success ping is sent, and Watchflow's dead man's switch will fire. |
| Edge Cases | If the Aggregate node produces no item at all (zero input items), the If node is never executed, which also results in no success ping. This is the desired dead man's switch behavior. |
| Sticky Note | Same as Aggregate ("Workflow Monitoring & Data Aggregation") |

---

### Block 1.6 — Success Reporting

**Overview:** Sends a success heartbeat to Watchflow, including the exact count of processed items as a metric. This heartbeat confirms to Watchflow that the workflow ran and processed data.

**Nodes Involved:**
- Mark job as successful

**Node Details — Mark job as successful**
| Property | Detail |
|---|---|
| Type | `@watchflow/n8n-nodes-watchflow.watchflow` (v1) |
| Role | Sends a success heartbeat to Watchflow with item-count metrics |
| Configuration | Operation: `success` (default); Key: `sample`; Name: `Sample Monitor`; Data: expression `={ "items": {{ $('Aggregate').item.json.data.length }} }` |
| Key Expressions | `$('Aggregate').item.json.data.length` — references the output of the Aggregate node to retrieve the count of processed items |
| Credentials | `Watchflow account` (Watchflow API credential, stored as `test-creds`) |
| Input | From **If** (true branch) |
| Output | None (terminal node in the success path) |
| Edge Cases | If the Watchflow API key is invalid or the Watchflow service is unreachable, this node will fail. Since it sits after the If node, the global Error Trigger will catch such a failure and route it to the failure reporting block. |
| Sticky Note | "Workflow Monitoring & Data Aggregation" (same sticky note as Aggregate and If) |

---

### Block 1.7 — Global Error Handling & Failure Reporting

**Overview:** A global safety net that catches any unhandled error from any node in the workflow and immediately reports it to Watchflow with full execution context. This block operates independently from the main pipeline — it is triggered by n8n's built-in error workflow mechanism.

**Nodes Involved:**
- Error Trigger
- Mark job as failed

**Node Details — Error Trigger**
| Property | Detail |
|---|---|
| Type | `n8n-nodes-base.errorTrigger` (v1) |
| Role | Catches any unhandled error from any node in the workflow; acts as a global error handler |
| Configuration | No parameters — n8n automatically populates the output with error details (error message, execution ID, workflow ID, failed node name) |
| Input | None (trigger node, activated by n8n's error workflow mechanism) |
| Output | To **Mark job as failed** |
| Edge Cases | This node only fires if it is configured as the workflow's Error Workflow in Settings. In a production setup, you can save this error-handling sequence as its own standalone workflow and reference it as the default Error Workflow across all your other workflows. |
| Sticky Note | "Generic Error Trigger — This dedicated module listens for any unexpected node failure across your entire workflow. It captures the specific error message and the failed node name, immediately triggering an incident alert in Watchflow so you can debug the exact execution link without searching through logs." |

**Node Details — Mark job as failed**
| Property | Detail |
|---|---|
| Type | `@watchflow/n8n-nodes-watchflow.watchflow` (v1) |
| Role | Sends a failure heartbeat to Watchflow with full execution context |
| Configuration | Operation: `fail`; Key: `PDF_GENERATION`; Name: `Pdf Generation`; Error: `={{ JSON.stringify($json.execution) }}` |
| Key Expressions | `JSON.stringify($json.execution)` — serializes the entire execution context received from the Error Trigger into a string for Watchflow |
| Credentials | `Watchflow account` (Watchflow API credential, stored as `test-creds`) |
| Input | From **Error Trigger** |
| Output | None (terminal node in the error path) |
| Edge Cases | The key `PDF_GENERATION` and name `Pdf Generation` are example values from the template; in production, replace them with a key and name that match the actual Watchflow monitor you have configured. If the Watchflow service is down, the failure report cannot be sent — consider adding a fallback notification (e.g., email or Slack) for resilience. |
| Sticky Note | Same as Error Trigger ("Generic Error Trigger") |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | manualTrigger | Manual entry point for the workflow | — | Get Items | Run this workflow manually |
| Get Items | set (v3.4) | Generates a static array of nine numeric items simulating extracted data | When clicking 'Execute workflow' | Split Out Items | Dummy Data Source — Your dummy workflow data extraction source: In this step, we exemplary fetch 20 items from a source to simulate a real-world data stream for processing and monitoring. |
| Split Out Items | splitOut (v1) | Expands the array field into individual items | Get Items | Set Label | Dummy Data Source — Your dummy workflow data extraction source: In this step, we exemplary fetch 20 items from a source to simulate a real-world data stream for processing and monitoring. |
| Set Label | set (v3.4) | Assigns a human-readable label to each item representing transformation | Split Out Items | No Operation, do nothing | Dummy Data Transformation — Dummy data transformation: This node simulates your business logic by processing the extracted items and preparing them for the final monitoring step. |
| No Operation, do nothing | noOp (v1) | Placeholder pass-through simulating the "load" step of ETL | Set Label | Aggregate | Dummy Data Load — This node simulates the final step where your processed data is synced to a target system (like a CRM or Database), confirming the functional success of the workflow. |
| Aggregate | aggregate (v1) | Merges all items into a single aggregated result for metric reporting | No Operation, do nothing | If | Workflow Monitoring & Data Aggregation — This node uses an Aggregate pattern to collect all processed items into a single count. By calculating the total amount of successful syncs, you can send precise metrics to Watchflow to ensure your workflow didn't just "run," but actually delivered results. |
| If | if (v2.2) | Validates that at least one item was processed before sending a success ping | Aggregate | Mark job as successful (true branch only) | Workflow Monitoring & Data Aggregation — This node uses an Aggregate pattern to collect all processed items into a single count. By calculating the total amount of successful syncs, you can send precise metrics to Watchflow to ensure your workflow didn't just "run," but actually delivered results. |
| Mark job as successful | @watchflow/n8n-nodes-watchflow.watchflow (v1) | Sends a success heartbeat with item-count metric to Watchflow | If | — (terminal) | Workflow Monitoring & Data Aggregation — This node uses an Aggregate pattern to collect all processed items into a single count. By calculating the total amount of successful syncs, you can send precise metrics to Watchflow to ensure your workflow didn't just "run," but actually delivered results. |
| Error Trigger | errorTrigger (v1) | Global error handler that catches any unhandled node failure | — (triggered by n8n error mechanism) | Mark job as failed | Generic Error Trigger — This dedicated module listens for any unexpected node failure across your entire workflow. It captures the specific error message and the failed node name, immediately triggering an incident alert in Watchflow so you can debug the exact execution link without searching through logs. |
| Mark job as failed | @watchflow/n8n-nodes-watchflow.watchflow (v1) | Sends a failure heartbeat with full execution context to Watchflow | Error Trigger | — (terminal) | Generic Error Trigger — This dedicated module listens for any unexpected node failure across your entire workflow. It captures the specific error message and the failed node name, immediately triggering an incident alert in Watchflow so you can debug the exact execution link without searching through logs. |

---

## 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the workflow in a new n8n instance:

### Prerequisites
1. Install the Watchflow community node: in n8n, go to **Settings → Community Nodes**, search for `@watchflow/n8n-nodes-watchflow`, and install it.
2. Create a **Watchflow API** credential: sign up at [watchflow.io](https://watchflow.io), go to **Settings → API Key**, copy your key, then in n8n go to **Credentials → New → Watchflow API**, paste the key, and save.
3. (Optional) Create an **n8n API** credential for Watchflow to link executions: in n8n go to **Settings → n8n API**, generate a key, then add the credential in Watchflow with the base URL appended with `/v1/api` (e.g., `https://n8n.yourdomain.com/v1/api`).

### Step-by-Step Instructions

1. **Create a new workflow** named "n8n Monitoring best practices" (or your preferred name).

2. **Add the Manual Trigger node:**
   - Click **Add Node → Triggers → Manual Trigger**.
   - Name it: `When clicking 'Execute workflow'`.
   - No parameters needed.

3. **Add the Get Items node (Set node):**
   - Click **Add Node → Actions → Set**.
   - Name it: `Get Items`.
   - In **Assignments**, click **Add Assignment**:
     - Name: `data`
     - Type: `Array`
     - Value: `[1,2,3,4,5,6,7,8,9]`
   - Connect: `When clicking 'Execute workflow'` → `Get Items`.

4. **Add the Split Out Items node:**
   - Click **Add Node → Actions → Split Out**.
   - Name it: `Split Out Items`.
   - In **Field to Split Out**, enter: `data`
   - Connect: `Get Items` → `Split Out Items`.

5. **Add the Set Label node (Set node):**
   - Click **Add Node → Actions → Set**.
   - Name it: `Set Label`.
   - In **Assignments**, click **Add Assignment**:
     - Name: `label`
     - Type: `String`
     - Value (expression): `=Item #{{ $json.data }}`
   - Connect: `Split Out Items` → `Set Label`.

6. **Add the No Operation node:**
   - Click **Add Node → Actions → No Operation**.
   - Name it: `No Operation, do nothing`.
   - No parameters needed.
   - Connect: `Set Label` → `No Operation, do nothing`.

7. **Add the Aggregate node:**
   - Click **Add Node → Actions → Aggregate**.
   - Name it: `Aggregate`.
   - Set **Aggregate** to: `Aggregate All Item Data`.
   - Connect: `No Operation, do nothing` → `Aggregate`.

8. **Add the If node:**
   - Click **Add Node → Logic → IF**.
   - Name it: `If`.
   - Under **Conditions**, configure:
     - Combinator: `And`
     - Add condition:
       - Left value (expression): `={{ $json.data.length > 0 }}`
       - Operator: `Boolean → Equals`
       - Right value (expression): `={{true}}`
   - Do **not** connect the false branch — this is intentional for the dead man's switch pattern.
   - Connect: `Aggregate` → `If` (input).

9. **Add the Mark job as successful node (Watchflow node):**
   - Click **Add Node → Actions → Watchflow** (available after installing the community node).
   - Name it: `Mark job as successful`.
   - Configure:
     - **Operation**: `Success` (default)
     - **Key**: `sample`
     - **Name**: `Sample Monitor`
     - **Data** (expression): `={ "items": {{ $('Aggregate').item.json.data.length }} }`
   - In **Credentials**, select your Watchflow API credential.
   - Connect: `If` → (true output) → `Mark job as successful`.

10. **Add the Error Trigger node:**
    - Click **Add Node → Triggers → Error Trigger**.
    - Name it: `Error Trigger`.
    - No parameters needed.

11. **Add the Mark job as failed node (Watchflow node):**
    - Click **Add Node → Actions → Watchflow**.
    - Name it: `Mark job as failed`.
    - Configure:
      - **Operation**: `Fail`
      - **Key**: `PDF_GENERATION` *(replace with your own Watchflow monitor key in production)*
      - **Name**: `Pdf Generation` *(replace with your own monitor name)*
      - **Error** (expression): `={{ JSON.stringify($json.execution) }}`
    - In **Credentials**, select your Watchflow API credential.
    - Connect: `Error Trigger` → `Mark job as failed`.

12. **Configure the Error Workflow (recommended):**
    - Save the workflow.
    - In **Workflow Settings**, set the **Error Workflow** field to point to this workflow itself (or save the error-handling nodes as a separate standalone workflow and reference it).
    - Alternatively, go to **Settings → Error Workflow** on any other workflow and select this workflow as the default error handler.

13. **Add sticky notes for documentation (optional but recommended):**
    - Add a sticky note at the top with the workflow title and a description of the three monitoring patterns (Dead Man's Switch, Performance Transparency, Instant Incident Alerting).
    - Add sticky notes over each functional section with the descriptions provided in the Summary Table.

14. **Activate the workflow:**
    - If you want the Error Trigger to work, the workflow must be **Active**.
    - Click the **Active** toggle in the top-right corner.

### Verification Steps
- Click **Execute Workflow** to run it manually.
- Confirm that the `If` node receives the aggregated data and routes to `Mark job as successful`.
- In your Watchflow dashboard, verify that a success heartbeat appears with the correct item count (`items: 9`).
- To test the error path, temporarily cause a node to fail (e.g., add a Function node that throws an error). Confirm that the Error Trigger fires and `Mark job as failed` sends a failure report to Watchflow.
- To test the dead man's switch, modify the `Get Items` node to produce an empty array (`[]`). Confirm that the `If` node's condition evaluates to false (or no items reach it at all), no success ping is sent, and Watchflow raises a dead man's switch alert after the expected interval.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Watchflow official site for sign-up, API keys, and documentation | [https://watchflow.io](https://watchflow.io) |
| n8n Monitoring Use Cases guide — detailed explanation of dead man's switch, performance transparency, and incident alerting patterns | [https://www.watchflow.io/use-cases/n8n-monitoring/](https://www.watchflow.io/use-cases/n8n-monitoring/) |
| Install the Watchflow community node via npm: `npm install @watchflow/n8n-nodes-watchflow` | Community node setup |
| When adding n8n API credentials in Watchflow, append `/v1/api` to your base URL (e.g., `https://n8n.yourdomain.com/v1/api`) | Watchflow credential tip |
| The `key` and `name` fields in the Watchflow nodes (`sample` / `Sample Monitor` for success, `PDF_GENERATION` / `Pdf Generation` for failure) are template examples. Replace them with your own monitor identifiers created in the Watchflow dashboard before production use. | Configuration customization |
| The Error Trigger node only functions when the workflow is **Active**. It must also be set as the Error Workflow in n8n Settings (either globally or per-workflow) to catch errors from other workflows. | Error Trigger requirement |
| The false branch of the `If` node is intentionally left disconnected. This is the core of the dead man's switch: if zero items are processed, no success heartbeat is sent, and Watchflow triggers an alert after the expected interval elapses. | Dead man's switch design |
| The `Mark job as failed` node sends the entire execution object (`JSON.stringify($json.execution)`) to Watchflow, which includes the execution ID, workflow ID, error message, and failed node name. This enables Watchflow to provide a direct link to the exact failed execution in your n8n instance. | Error context richness |
| For production resilience, consider adding a secondary notification channel (e.g., Slack, email, or PagerDuty) after `Mark job as failed` to ensure you are alerted even if the Watchflow service is temporarily unavailable. | Reliability best practice |