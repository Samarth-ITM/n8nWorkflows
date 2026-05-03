Learn Data Synchronization: Warehouse Inventory Audit Tutorial

https://n8nworkflows.xyz/workflows/learn-data-synchronization--warehouse-inventory-audit-tutorial-5999


# Learn Data Synchronization: Warehouse Inventory Audit Tutorial

### 1. Workflow Overview

This workflow demonstrates a practical data synchronization scenario between two warehouse inventories, designed as a tutorial to explain the usage of the **Compare Datasets** node in n8n. The use case is auditing and synchronizing inventory data between a **source of truth** warehouse (Warehouse A) and a secondary warehouse (Warehouse B) to ensure consistency.

The workflow is logically divided into the following blocks:

- **1.1 Input Data Setup:** Initialization of warehouse inventory data for Warehouse A (source of truth) and Warehouse B (to be synchronized).
- **1.2 Data Preparation:** Splitting the inventory arrays into individual product records to prepare for comparison.
- **1.3 Data Comparison (The Auditor):** Using the Compare Datasets node to audit the differences, matches, additions, and deletions between the two warehouses.
- **1.4 Post-Comparison Actions:** Routing outputs into four distinct branches representing different synchronization actions: Add, Update, Remove, or Do Nothing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Data Setup

- **Overview:** This block initializes the inventory data arrays for Warehouse A and Warehouse B. Warehouse A is treated as the authoritative source, while Warehouse B's data needs to be audited and synchronized accordingly.
- **Nodes Involved:**  
  - `Start Audit`  
  - `Warehouse A (Source of Truth)`  
  - `Warehouse B (To be Synced)`

- **Node Details:**

  - **Start Audit**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point to manually start the workflow.  
    - *Configuration:* No parameters; simply triggers workflow execution on demand.  
    - *Connections:* Outputs to both Warehouse A and Warehouse B data sets.  
    - *Failure Modes:* None expected; manual trigger.

  - **Warehouse A (Source of Truth)**  
    - *Type:* Set node  
    - *Role:* Defines the master inventory dataset representing Warehouse A.  
    - *Configuration:* Sets a single field `products` with an array of product objects (each with `product_id`, `name`, `stock`).  
    - *Key Data:*  
      ```json
      [
        {"product_id":"P-001","name":"Keyboard","stock":200},
        {"product_id":"P-002","name":"Mouse","stock":150},
        {"product_id":"P-003","name":"Webcam","stock":75}
      ]
      ```  
    - *Connections:* Outputs to the splitting node for Warehouse A products.  
    - *Failure Modes:* None expected; static dataset.

  - **Warehouse B (To be Synced)**  
    - *Type:* Set node  
    - *Role:* Defines the inventory dataset representing Warehouse B, which needs synchronization.  
    - *Configuration:* Sets a `products` field with an array of products, similar structure but differing stock and some missing/extra products.  
    - *Key Data:*  
      ```json
      [
        {"product_id":"P-001","name":"Keyboard","stock":200},
        {"product_id":"P-002","name":"Mouse","stock":100},
        {"product_id":"P-004","name":"Monitor","stock":50}
      ]
      ```  
    - *Connections:* Outputs to splitting node for Warehouse B products.  
    - *Failure Modes:* None expected; static dataset.

---

#### 2.2 Data Preparation

- **Overview:** This block processes the array of products from each warehouse into individual items, enabling item-by-item comparison.
- **Nodes Involved:**  
  - `Split Out Prducts (A)`  
  - `Split Out Prducts (B)`

- **Node Details:**

  - **Split Out Prducts (A)**  
    - *Type:* Split Out  
    - *Role:* Splits the `products` array from Warehouse A into individual product records for comparison.  
    - *Configuration:* Field to split out: `products`  
    - *Connections:* Input from Warehouse A node; output to The Auditor node.  
    - *Failure Modes:* If the `products` field is missing or empty, no items to compare; workflow may effectively be a no-op.

  - **Split Out Prducts (B)**  
    - *Type:* Split Out  
    - *Role:* Splits the `products` array from Warehouse B similarly.  
    - *Configuration:* Field to split out: `products`  
    - *Connections:* Input from Warehouse B node; output to The Auditor node.  
    - *Failure Modes:* Same as above; empty or missing field means no items to compare.

---

#### 2.3 Data Comparison (The Auditor)

- **Overview:** This is the core auditing block that compares inventory items from both warehouses using the product ID as the unique key. It categorizes the results into four outputs: addition needed, no changes, update needed, or removal needed.
- **Nodes Involved:**  
  - `The Auditor`  
  - `Sticky Note1` (explains the node's role and outputs)

- **Node Details:**

  - **The Auditor**  
    - *Type:* Compare Datasets node  
    - *Role:* Compares individual product records from Warehouse A and B.  
    - *Configuration:*  
      - Merge by fields: `product_id` (both datasets)  
      - Source of truth: Warehouse A (input A)  
      - On difference: Use input A version (Warehouse A's data prevails)  
    - *Outputs:* Four outputs representing:  
      1. Items found only in Warehouse A (need to add to B)  
      2. Items identical in both (no action)  
      3. Items present in both but different (update B)  
      4. Items only in Warehouse B (remove from B)  
    - *Connections:*  
      - Input 1: Split Out Products (A)  
      - Input 2: Split Out Products (B)  
      - Outputs to four respective NoOp nodes for demonstration.  
    - *Failure Modes:*  
      - Missing or malformed product_id fields break matching logic.  
      - Large datasets may increase processing time.  
      - Misconfiguration of source of truth or merge fields causes incorrect results.  

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Role:* Documentation explaining the Compare Datasets node logic and outputs.  
    - *Content Summary:* Explains use of product_id as key, Warehouse A as source of truth, and meaning of four outputs.  
    - *Positioned:* Near The Auditor node for clarity.

---

#### 2.4 Post-Comparison Actions

- **Overview:** This block routes the output from the auditor into four branches representing potential synchronization actions. Each branch currently uses No Operation (NoOp) nodes as placeholders. Sticky notes provide detailed explanations of each category’s meaning and expected action in a production scenario.
- **Nodes Involved:**  
  - `➕ Add to Warehouse B`  
  - `✅ All Good (Do Nothing)`  
  - `🔄 Update in Warehouse B`  
  - `❌ Remove from Warehouse B`  
  - `Sticky Note2`, `Sticky Note11`, `Sticky Note12`, `Sticky Note13`, `Sticky Note14`

- **Node Details:**

  - **➕ Add to Warehouse B**  
    - *Type:* NoOp  
    - *Role:* Placeholder for logic to add missing items from Warehouse A to B.  
    - *Input:* Output 1 from The Auditor (items in A but not in B)  
    - *Sticky Note11:* Explains this category means items missing from B, next step is creation in B.

  - **✅ All Good (Do Nothing)**  
    - *Type:* NoOp  
    - *Role:* Placeholder for items that match perfectly; no synchronization needed.  
    - *Input:* Output 2 from The Auditor (identical items)  
    - *Sticky Note2:* Explains the meaning and that no action is required.

  - **🔄 Update in Warehouse B**  
    - *Type:* NoOp  
    - *Role:* Placeholder for updating mismatched items in Warehouse B to match Warehouse A.  
    - *Input:* Output 3 from The Auditor (items differing between A and B)  
    - *Sticky Note12:* Explains the meaning and expected update action.

  - **❌ Remove from Warehouse B**  
    - *Type:* NoOp  
    - *Role:* Placeholder for removing extraneous items from Warehouse B that do not exist in Warehouse A.  
    - *Input:* Output 4 from The Auditor (items in B but not in A)  
    - *Sticky Note13:* Explains the meaning and expected removal action.

  - **Sticky Note14**  
    - *Type:* Sticky Note  
    - *Role:* Calls for user feedback and provides links to coaching and consulting services related to n8n.  
    - *Content:* Contains links to feedback form, coaching sessions, and consulting inquiries.  
    - *Positioned:* Adjacent to post-comparison nodes for user visibility.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                   | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                                                                                                                 |
|---------------------------|-----------------------|---------------------------------|---------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start Audit               | Manual Trigger        | Initiates workflow execution     | None                            | Warehouse A (Source of Truth), Warehouse B (To be Synced) |                                                                                                                                                                                                             |
| Warehouse A (Source of Truth) | Set                   | Defines master inventory data    | Start Audit                     | Split Out Prducts (A)              |                                                                                                                                                                                                             |
| Warehouse B (To be Synced) | Set                   | Defines secondary inventory data | Start Audit                     | Split Out Prducts (B)              |                                                                                                                                                                                                             |
| Split Out Prducts (A)     | Split Out             | Splits Warehouse A products array | Warehouse A (Source of Truth)   | The Auditor                      |                                                                                                                                                                                                             |
| Split Out Prducts (B)     | Split Out             | Splits Warehouse B products array | Warehouse B (To be Synced)      | The Auditor                      |                                                                                                                                                                                                             |
| The Auditor               | Compare Datasets      | Compares inventories, categorizes differences | Split Out Prducts (A), Split Out Prducts (B) | ➕ Add to Warehouse B, ✅ All Good (Do Nothing), 🔄 Update in Warehouse B, ❌ Remove from Warehouse B | Explains use of product_id as key, Warehouse A is source of truth, outputs four categories.                                                                                                                   |
| ➕ Add to Warehouse B     | NoOp                  | Placeholder to add missing items  | The Auditor (output 1)           | None                              | Explains items missing from B; next step is creation in B.                                                                                                                                                   |
| ✅ All Good (Do Nothing)  | NoOp                  | Placeholder for unchanged items   | The Auditor (output 2)           | None                              | Explains items identical; no action needed.                                                                                                                                                                 |
| 🔄 Update in Warehouse B  | NoOp                  | Placeholder for items to update   | The Auditor (output 3)           | None                              | Explains items differing; next step is updating B.                                                                                                                                                           |
| ❌ Remove from Warehouse B | NoOp                  | Placeholder for items to remove   | The Auditor (output 4)           | None                              | Explains items in B not in A; next step is removal.                                                                                                                                                          |
| Sticky Note1              | Sticky Note           | Explains The Auditor node logic   | None                            | None                              | Describes Compare Datasets node, fields matched, source of truth, and four outputs.                                                                                                                         |
| Sticky Note2              | Sticky Note           | Explains "All Good" output meaning | None                            | None                              | Explains no action needed for identical items.                                                                                                                                                              |
| Sticky Note11             | Sticky Note           | Explains "Add" output meaning     | None                            | None                              | Explains adding missing items to Warehouse B.                                                                                                                                                               |
| Sticky Note12             | Sticky Note           | Explains "Update" output meaning  | None                            | None                              | Explains updating differing items in Warehouse B.                                                                                                                                                           |
| Sticky Note13             | Sticky Note           | Explains "Remove" output meaning  | None                            | None                              | Explains removing extraneous items from Warehouse B.                                                                                                                                                        |
| Sticky Note14             | Sticky Note           | Requests feedback, offers coaching | None                            | None                              | Provides links for feedback, coaching, and consulting services related to n8n.                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Start Audit`  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually.

2. **Create Set Node for Warehouse A**  
   - Name: `Warehouse A (Source of Truth)`  
   - Type: Set  
   - Parameters:  
     - Add field `products` (Array) with the following data:  
       ```json
       [
         {"product_id":"P-001","name":"Keyboard","stock":200},
         {"product_id":"P-002","name":"Mouse","stock":150},
         {"product_id":"P-003","name":"Webcam","stock":75}
       ]
       ```  
   - Connect output of `Start Audit` to this node.

3. **Create Set Node for Warehouse B**  
   - Name: `Warehouse B (To be Synced)`  
   - Type: Set  
   - Parameters:  
     - Add field `products` (Array) with the following data:  
       ```json
       [
         {"product_id":"P-001","name":"Keyboard","stock":200},
         {"product_id":"P-002","name":"Mouse","stock":100},
         {"product_id":"P-004","name":"Monitor","stock":50}
       ]
       ```  
   - Connect output of `Start Audit` to this node as well.

4. **Create Split Out Node for Warehouse A**  
   - Name: `Split Out Prducts (A)`  
   - Type: Split Out  
   - Parameters:  
     - Field to split out: `products`  
   - Connect output of `Warehouse A (Source of Truth)` to this node.

5. **Create Split Out Node for Warehouse B**  
   - Name: `Split Out Prducts (B)`  
   - Type: Split Out  
   - Parameters:  
     - Field to split out: `products`  
   - Connect output of `Warehouse B (To be Synced)` to this node.

6. **Create Compare Datasets Node**  
   - Name: `The Auditor`  
   - Type: Compare Datasets  
   - Parameters:  
     - Merge By Fields: Map `product_id` from Input A and `product_id` from Input B.  
     - Source of Truth: Input A (Warehouse A).  
     - On Difference: Use Input A Version (Warehouse A data prevails).  
   - Connect outputs:  
     - Input 1 from `Split Out Prducts (A)`  
     - Input 2 from `Split Out Prducts (B)`

7. **Create Four No Operation Nodes for Outputs:**  
   - Names and connections:  
     - `➕ Add to Warehouse B`: Connect from output 1 of `The Auditor` (items missing in B).  
     - `✅ All Good (Do Nothing)`: Connect from output 2 (matching items).  
     - `🔄 Update in Warehouse B`: Connect from output 3 (differing items).  
     - `❌ Remove from Warehouse B`: Connect from output 4 (extra items in B).

8. **Add Sticky Notes for Documentation (Optional but Recommended):**  
   - Create sticky notes near relevant nodes with content explaining node roles and outputs.  
   - Include the long introductory note describing the tutorial and workflow context.  
   - Add notes explaining each NoOp node’s meaning and expected real-world action.  
   - Add a feedback and coaching invitation note near the output nodes.

9. **Run and Test:**  
   - Manually trigger the workflow via `Start Audit`.  
   - Inspect the outputs of each NoOp node to confirm items are categorized correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                              | Context or Link                                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow is a teaching resource demonstrating the use of the Compare Datasets node in n8n for data synchronization between two datasets. It uses a warehouse inventory analogy for clarity.                                                                                          | Workflow description                                                                                                 |
| Feedback and coaching offered by Lucas Peyrin to improve n8n skills and workflows. Links provided for feedback form, coaching sessions, and consulting inquiries.                                                                                                                         | [Feedback Form](https://api.ia2s.app/form/templates/feedback?template=Compare%20Datasets)  
[Book Coaching](https://api.ia2s.app/form/templates/coaching?template=Compare%20Datasets)  
[Consulting](https://api.ia2s.app/form/templates/consulting?template=Compare%20Datasets) |
| The Compare Datasets node treats input A as "source of truth" and uses `product_id` as the unique key for matching records. Differences resolve in favor of input A’s data.                                                                                                                 | Sticky Note1 content                                                                                                 |
| The four outputs correspond to: Add missing items, Confirm matched items, Update differing items, Remove extraneous items.                                                                                                                                                                | Sticky Note content adjacent to output nodes                                                                        |

---

**Disclaimer:**  
The provided text is extracted solely from a workflow automated with n8n, adhering strictly to content policies and containing no illegal, offensive, or protected elements. All manipulated data are legal and public.