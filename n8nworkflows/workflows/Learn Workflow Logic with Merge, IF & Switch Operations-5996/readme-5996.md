Learn Workflow Logic with Merge, IF & Switch Operations

https://n8nworkflows.xyz/workflows/learn-workflow-logic-with-merge--if---switch-operations-5996


# Learn Workflow Logic with Merge, IF & Switch Operations

### 1. Workflow Overview

This workflow, titled **"Learn Workflow Logic with Merge, IF & Switch Operations"**, is an educational automation designed to demonstrate the core logical control nodes in n8n: **Merge**, **IF**, and **Switch** nodes. It uses the analogy of a package sorting center to help users understand how data (packages) can be combined, conditionally routed, and classified into multiple categories.

**Target Use Cases:**  
- Learning and experimenting with n8n's fundamental logic nodes.  
- Understanding data flow control patterns in automation.  
- Designing sorting or decision-making workflows that merge data streams, apply conditional logic, and route data to multiple outputs.

**Logical Blocks:**

- **1.1 Data Creation & Start:** Manual trigger to start the workflow and sets up initial package data (letters and parcel).  
- **1.2 First Merge:** Combines multiple package data streams into one unified list.  
- **1.3 IF Node Filtering:** Splits packages based on whether they are fragile or not, processing each path separately.  
- **1.4 Second Merge:** Recombines the fragile and non-fragile packages after handling instructions are added.  
- **1.5 Switch Node Sorting:** Routes all packages to appropriate destination "bins" based on their destination attribute.  
- **1.6 Final Output:** Collects all sorted packages into one endpoint node.

Each logical block corresponds to a step in the sorting and decision process, illustrating the practical use of these nodes in a workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Creation & Start

- **Overview:** This block initializes the workflow data, simulating packages on different conveyor belts. It includes a manual trigger and three Set nodes creating letters and a parcel with various properties.
- **Nodes Involved:**  
  - Start Sorting (Manual Trigger)  
  - Create Letter (Set)  
  - Create 2nd Letter (Set)  
  - Create Parcel (Set)  

- **Node Details:**

  - **Start Sorting**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow execution.  
    - Config: Default, no parameters.  
    - Inputs: None  
    - Outputs: Connected to the three Set nodes.  
    - Edge Cases: None typical; manual trigger requires user action.

  - **Create Letter**  
    - Type: Set  
    - Role: Creates a data item representing a letter package destined for London.  
    - Config: Sets `package_id` = "L-001", `type` = "letter", `destination` = "London".  
    - Inputs: From Start Sorting  
    - Outputs: Connects to Merge node.  
    - Edge Cases: None; static data sets.

  - **Create 2nd Letter**  
    - Type: Set  
    - Role: Creates a second letter destined for Tokyo.  
    - Config: `package_id` = "L-002", `type` = "letter", `destination` = "Tokyo".  
    - Inputs: From Start Sorting  
    - Outputs: Connects to Merge node.  
    - Edge Cases: None.

  - **Create Parcel**  
    - Type: Set  
    - Role: Creates a parcel package destined for New York, marked as fragile.  
    - Config: `package_id` = "P-001", `type` = "parcel", `destination` = "New York", `is_fragile` = true (boolean).  
    - Inputs: From Start Sorting  
    - Outputs: Connects to Merge node.  
    - Edge Cases: None.

---

#### 2.2 First Merge

- **Overview:** Combines the three separate package streams (two letters and one parcel) into a single stream for subsequent processing. Uses Append mode to concatenate data items.
- **Nodes Involved:**  
  - 1. Merge Node  

- **Node Details:**

  - **1. Merge Node**  
    - Type: Merge  
    - Role: Appends incoming data from multiple inputs into one unified array.  
    - Config: Number of inputs = 3 (matching the three Set nodes). Default Append mode (waits for all inputs before passing combined data).  
    - Inputs: From Create Letter, Create 2nd Letter, Create Parcel  
    - Outputs: To IF node for conditional routing.  
    - Edge Cases: If one input has no data or errors, the Merge may wait indefinitely or fail; ensure all inputs provide data or handle missing inputs.

---

#### 2.3 IF Node Filtering (Fragile vs Non-Fragile)

- **Overview:** Splits the combined package stream based on whether a package is fragile, using a boolean condition. This simulates a two-way sorting gate.
- **Nodes Involved:**  
  - 2. IF Node  
  - Add 'Fragile' Handling (Set)  
  - Add 'Standard' Handling (Set)  

- **Node Details:**

  - **2. IF Node**  
    - Type: IF  
    - Role: Checks if each package item has `is_fragile` property set to true.  
    - Config: Condition uses expression `{{$json.is_fragile}}` evaluated as boolean true. Loose type validation enabled to tolerate absence of property.  
    - Inputs: From 1. Merge Node  
    - Outputs:  
      - True path: To Add 'Fragile' Handling  
      - False path: To Add 'Standard' Handling  
    - Edge Cases: Packages missing `is_fragile` are routed false by default; expression errors if data format unexpected.

  - **Add 'Fragile' Handling**  
    - Type: Set  
    - Role: Appends `handling_instructions` = "Handle with care!" to fragile packages.  
    - Config: Adds field `handling_instructions` with string value "Handle with care!". Preserves other fields.  
    - Inputs: True output of IF node  
    - Outputs: To second Merge node  
    - Edge Cases: None typical.

  - **Add 'Standard' Handling**  
    - Type: Set  
    - Role: Adds `handling_instructions` = "Standard handling" for non-fragile packages.  
    - Config: Adds field `handling_instructions` with string value "Standard handling".  
    - Inputs: False output of IF node  
    - Outputs: To second Merge node  
    - Edge Cases: None.

---

#### 2.4 Second Merge (Recombining Fragile and Non-Fragile)

- **Overview:** Recombines the two separate streams (fragile and non-fragile) back into a single data stream for the next sorting step.
- **Nodes Involved:**  
  - Re-group All Packages (Merge)  

- **Node Details:**

  - **Re-group All Packages**  
    - Type: Merge  
    - Role: Combines the two processed streams back into one unified stream.  
    - Config: Default merge mode (Append) with two inputs.  
    - Inputs: From Add 'Fragile' Handling and Add 'Standard' Handling  
    - Outputs: To Switch node for destination-based sorting  
    - Edge Cases: Similar to first Merge node; all inputs must provide data.

---

#### 2.5 Switch Node Sorting (Multi-Destination Routing)

- **Overview:** Routes packages to different outputs based on the `destination` field, supporting multiple destinations plus a default fallback.
- **Nodes Involved:**  
  - 3. Switch Node  
  - Send to London Bin (Set)  
  - Send to New York Bin (Set)  
  - Send to Tokyo Bin (Set)  
  - Default Bin (Set)  

- **Node Details:**

  - **3. Switch Node**  
    - Type: Switch  
    - Role: Checks `destination` field and routes to matching outputs.  
    - Config:  
      - Output 0: Matches "London" (case-sensitive, strict)  
      - Output 1: Matches "New York"  
      - Output 2: Matches "Tokyo"  
      - Default output renamed "Default" for non-matching destinations.  
    - Inputs: From Re-group All Packages  
    - Outputs: Four outputs to respective Set nodes.  
    - Edge Cases: Case sensitivity means "london" (lowercase) would go to default output. Missing `destination` also routes to default.

  - **Send to London Bin**  
    - Type: Set  
    - Role: Adds `sorting_bin` = "A1 (London)" for London packages.  
    - Config: Adds field `sorting_bin` with specified string, preserves other fields.  
    - Inputs: Switch output 0  
    - Outputs: To Final Sorted Packages (NoOp)  
    - Edge Cases: None.

  - **Send to New York Bin**  
    - Type: Set  
    - Role: Adds `sorting_bin` = "B2 (New York)".  
    - Config: Similar to above.  
    - Inputs: Switch output 1  
    - Outputs: To Final Sorted Packages  
    - Edge Cases: None.

  - **Send to Tokyo Bin**  
    - Type: Set  
    - Role: Adds `sorting_bin` = "C3 (Tokyo)".  
    - Config: As above.  
    - Inputs: Switch output 2  
    - Outputs: To Final Sorted Packages  
    - Edge Cases: None.

  - **Default Bin**  
    - Type: Set  
    - Role: Adds `sorting_bin` = "Return to Sender" for unmatched destinations.  
    - Config: As above.  
    - Inputs: Default output of Switch  
    - Outputs: To Final Sorted Packages  
    - Edge Cases: Handles any unexpected or missing destinations.

---

#### 2.6 Final Output

- **Overview:** Collects all the sorted packages after destination assignment into a single endpoint, signaling the end of the workflow.
- **Nodes Involved:**  
  - Final Sorted Packages (NoOp)  

- **Node Details:**

  - **Final Sorted Packages**  
    - Type: NoOp  
    - Role: Serves as a visual end-point for all sorted packages. Does not modify data.  
    - Config: None  
    - Inputs: From all four Set nodes of destination bins  
    - Outputs: None  
    - Edge Cases: None.

---

#### 2.7 Sticky Notes (Documentation & Guidance)

- **Overview:** Multiple sticky note nodes provide inline documentation, explanations, and user instructions. They do not affect data flow but are critical for learning.

- **Sticky Note (Tutorial Intro)**  
  - Content: Introduces the workflow and analogy of package sorting. Explains the role of Merge, IF, and Switch nodes.  
  - Position: Top-left, near start.

- **Sticky Note1 (Merge Explanation)**  
  - Content: Explains the Merge node as combining streams, appending data.  
  - Positioned near first Merge node.

- **Sticky Note2 (IF Explanation)**  
  - Content: Describes IF node as a two-path sorting gate based on `is_fragile`.  
  - Near IF node.

- **Sticky Note3 (Second Merge Explanation)**  
  - Content: Explains why a second Merge after IF is necessary to recombine streams.  
  - Positioned near second Merge.

- **Sticky Note4 (Switch Explanation)**  
  - Content: Describes Switch node as multi-path routing by `destination`.  
  - Positioned near Switch node.

- **Sticky Note5 (Summary and Congratulations)**  
  - Content: Summarizes what was learned and congratulates the user.  
  - Positioned near final nodes.

- **Sticky Note10 (Feedback and Coaching Links)**  
  - Content: Invites user feedback and offers coaching and consulting services with multiple clickable links:  
    - Feedback form: https://api.ia2s.app/form/templates/feedback?template=Merge%20If%20Switch  
    - Coaching sessions: https://api.ia2s.app/form/templates/coaching?template=Merge%20If%20Switch  
    - Consulting inquiries: https://api.ia2s.app/form/templates/consulting?template=Merge%20If%20Switch  
  - Positioned top-right corner.

---

### 3. Summary Table

| Node Name             | Node Type        | Functional Role                            | Input Node(s)                       | Output Node(s)                              | Sticky Note                                                                                           |
|-----------------------|------------------|------------------------------------------|-----------------------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Start Sorting         | Manual Trigger   | Workflow entry trigger                    | None                              | Create Letter, Create 2nd Letter, Create Parcel |                                                                                                     |
| Create Letter         | Set              | Creates letter package (London)           | Start Sorting                    | 1. Merge Node                              |                                                                                                     |
| Create 2nd Letter     | Set              | Creates letter package (Tokyo)            | Start Sorting                    | 1. Merge Node                              |                                                                                                     |
| Create Parcel         | Set              | Creates parcel package (New York, fragile) | Start Sorting                    | 1. Merge Node                              |                                                                                                     |
| 1. Merge Node         | Merge            | Combines package streams                   | Create Letter, Create 2nd Letter, Create Parcel | 2. IF Node                                | Sticky Note1: Explains Merge node as appending data streams.                                        |
| 2. IF Node            | IF               | Splits packages by fragile property       | 1. Merge Node                   | Add 'Fragile' Handling (true), Add 'Standard' Handling (false) | Sticky Note2: Describes IF node as two-path sorting gate checking 'is_fragile'.                      |
| Add 'Fragile' Handling | Set              | Adds handling instructions for fragile    | 2. IF Node (true)               | Re-group All Packages                       |                                                                                                     |
| Add 'Standard' Handling | Set              | Adds handling instructions for non-fragile | 2. IF Node (false)              | Re-group All Packages                       |                                                                                                     |
| Re-group All Packages  | Merge            | Recombines fragile and non-fragile streams | Add 'Fragile' Handling, Add 'Standard' Handling | 3. Switch Node                             | Sticky Note3: Explains need for second Merge after IF.                                              |
| 3. Switch Node        | Switch           | Routes packages by destination             | Re-group All Packages            | Send to London Bin, Send to New York Bin, Send to Tokyo Bin, Default Bin | Sticky Note4: Explains Switch node as multi-path routing by destination.                            |
| Send to London Bin    | Set              | Assigns sorting bin for London packages    | 3. Switch Node (London output)   | Final Sorted Packages                       |                                                                                                     |
| Send to New York Bin  | Set              | Assigns sorting bin for New York packages  | 3. Switch Node (New York output) | Final Sorted Packages                       |                                                                                                     |
| Send to Tokyo Bin     | Set              | Assigns sorting bin for Tokyo packages     | 3. Switch Node (Tokyo output)    | Final Sorted Packages                       |                                                                                                     |
| Default Bin           | Set              | Assigns sorting bin for unmatched packages | 3. Switch Node (Default output)  | Final Sorted Packages                       |                                                                                                     |
| Final Sorted Packages | NoOp             | Final collection point for sorted packages | Send to London Bin, Send to New York Bin, Send to Tokyo Bin, Default Bin | None                                       | Sticky Note5: Congratulates user and summarizes learning points.                                    |
| Sticky Note           | Sticky Note      | Introductory tutorial and analogy          | None                             | None                                        | Primary workflow introduction and usage instructions.                                              |
| Sticky Note1          | Sticky Note      | Explains first Merge node                   | None                             | None                                        | See above.                                                                                          |
| Sticky Note2          | Sticky Note      | Explains IF node                            | None                             | None                                        | See above.                                                                                          |
| Sticky Note3          | Sticky Note      | Explains second Merge                       | None                             | None                                        | See above.                                                                                          |
| Sticky Note4          | Sticky Note      | Explains Switch node                        | None                             | None                                        | See above.                                                                                          |
| Sticky Note5          | Sticky Note      | Final summary and congratulations           | None                             | None                                        | See above.                                                                                          |
| Sticky Note10         | Sticky Note      | Feedback and coaching links                  | None                             | None                                        | Contains feedback and coaching links: https://api.ia2s.app/form/templates/feedback?template=Merge%20If%20Switch |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "Start Sorting"  
   - Purpose: To manually start the workflow.

2. **Create Three Set Nodes for Initial Packages:**  
   - Name: "Create Letter"  
     - Set fields:  
       - `package_id` = "L-001" (string)  
       - `type` = "letter" (string)  
       - `destination` = "London" (string)  
   - Name: "Create 2nd Letter"  
     - Set fields:  
       - `package_id` = "L-002"  
       - `type` = "letter"  
       - `destination` = "Tokyo"  
   - Name: "Create Parcel"  
     - Set fields:  
       - `package_id` = "P-001"  
       - `type` = "parcel"  
       - `destination` = "New York"  
       - `is_fragile` = true (boolean)

3. **Connect the Manual Trigger's output to all three Set nodes.**

4. **Add a Merge Node:**  
   - Name: "1. Merge Node"  
   - Set Number of Inputs: 3  
   - Mode: Append (default)  
   - Connect the three Set nodes outputs to this Merge node.

5. **Add an IF Node:**  
   - Name: "2. IF Node"  
   - Condition: Check if `is_fragile` property is true:  
     - Expression: `{{$json.is_fragile}}` (boolean true)  
     - Enable loose type validation to handle missing property gracefully.  
   - Connect the output of the Merge node to the IF node input.

6. **Create two Set Nodes for Handling Instructions:**  
   - Name: "Add 'Fragile' Handling"  
     - Field: `handling_instructions` = "Handle with care!" (string)  
     - Preserve other fields.  
   - Name: "Add 'Standard' Handling"  
     - Field: `handling_instructions` = "Standard handling" (string)  
     - Preserve other fields.

7. **Connect IF node outputs:**  
   - True path to "Add 'Fragile' Handling"  
   - False path to "Add 'Standard' Handling"

8. **Add a Merge Node:**  
   - Name: "Re-group All Packages"  
   - Default append mode with 2 inputs.  
   - Connect outputs of the two "Add Handling" Set nodes to the Merge node.

9. **Add a Switch Node:**  
   - Name: "3. Switch Node"  
   - Add rules based on `destination` field (string, case-sensitive, strict):  
     - Output 0: Equals "London"  
     - Output 1: Equals "New York"  
     - Output 2: Equals "Tokyo"  
   - Set fallback/default output renamed to "Default".  
   - Connect output of "Re-group All Packages" Merge node to this Switch node.

10. **Add four Set Nodes for destination bins:**  
    - Name: "Send to London Bin"  
      - Field: `sorting_bin` = "A1 (London)"  
    - Name: "Send to New York Bin"  
      - Field: `sorting_bin` = "B2 (New York)"  
    - Name: "Send to Tokyo Bin"  
      - Field: `sorting_bin` = "C3 (Tokyo)"  
    - Name: "Default Bin"  
      - Field: `sorting_bin` = "Return to Sender"

11. **Connect the outputs of the Switch node:**  
    - Output 0 to "Send to London Bin"  
    - Output 1 to "Send to New York Bin"  
    - Output 2 to "Send to Tokyo Bin"  
    - Default output to "Default Bin"

12. **Add a NoOp node:**  
    - Name: "Final Sorted Packages"  
    - Connect outputs of all four "Send to ... Bin" Set nodes to this NoOp node.

13. **(Optional) Add Sticky Notes:**  
    - Add sticky notes explaining the purpose of each block and the overall workflow.  
    - Position them near corresponding nodes for clarity.

**No credentials are required for this workflow as it uses only manual triggers, Set nodes, and logical nodes without external API calls.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is designed as an educational template to demonstrate three fundamental logical operations in n8n: Merge, IF, and Switch nodes, using a package sorting analogy for clarity.                                        | Workflow purpose                                                                                                 |
| Feedback and coaching opportunities are offered to help users improve their n8n skills or implement custom automation projects.                                                                                                 | Feedback and coaching links: [Feedback form](https://api.ia2s.app/form/templates/feedback?template=Merge%20If%20Switch), [Coaching](https://api.ia2s.app/form/templates/coaching?template=Merge%20If%20Switch), [Consulting](https://api.ia2s.app/form/templates/consulting?template=Merge%20If%20Switch) |
| The workflow demonstrates common automation patterns: split-process-merge and conditional routing, essential for building complex workflows.                                                                                     | Conceptual note                                                                                                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.