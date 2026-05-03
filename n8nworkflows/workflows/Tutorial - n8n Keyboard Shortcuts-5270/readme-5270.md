Tutorial - n8n Keyboard Shortcuts

https://n8nworkflows.xyz/workflows/tutorial---n8n-keyboard-shortcuts-5270


# Tutorial - n8n Keyboard Shortcuts

### 1. Workflow Overview

This workflow is an interactive tutorial designed to teach users the essential keyboard shortcuts and basic interactions within the n8n workflow automation platform. Its primary use case is educational: guiding new or intermediate users through practical exercises that demonstrate how to efficiently navigate, manipulate, and manage nodes in the n8n canvas.

The workflow is logically divided into four main chapters, each covering a distinct aspect of node and canvas interaction:

- **1.1 Chapter 1: Node Basics** — Focuses on fundamental node interactions such as renaming, editing, deactivating, and duplicating nodes.
- **1.2 Chapter 2: Canvas Navigation & Selection** — Teaches users how to navigate around the canvas, select multiple nodes, and control zoom levels.
- **1.3 Chapter 3: Advanced Actions** — Introduces more powerful features like tidying up nodes, creating sub-workflows, and copy-pasting nodes.
- **1.4 Chapter 4: Execution & Debugging** — Covers workflow execution controls and debugging, including pinning data and opening execution panels.

Supporting these chapters are sticky notes acting as instructional guides and tasks, and a series of no-op and set nodes that serve as targets for keyboard shortcut practice. The workflow also includes a basic conditional flow triggered by a schedule node, though this is minimal and primarily structural.

---

### 2. Block-by-Block Analysis

#### 2.1 Chapter 1: Node Basics

- **Overview:**  
  This block introduces users to fundamental node operations such as renaming, editing values, deactivating nodes, and duplicating nodes. It sets the foundation for interacting with individual nodes.

- **Nodes Involved:**  
  - Sticky Notes: Introduction, Chapter 1 Header, Sticky Note13, Sticky Note14, Sticky Note15, Sticky Note16  
  - Nodes: ➡️ First, Rename Me!, ➡️ Now, Edit Me! (Set node), ➡️ This one is extra. Deactivate Me!, ➡️ We need another one. Duplicate Me!  

- **Node Details:**  
  - **➡️ First, Rename Me!**  
    - Type: NoOp (placeholder node)  
    - Role: Target node for renaming shortcut (`Space`)  
    - Connections: Leads to ➡️ Now, Edit Me!  
    - Edge cases: None; no execution logic.

  - **➡️ Now, Edit Me!**  
    - Type: Set node  
    - Role: Demonstrates editing node parameters (changing string value)  
    - Parameters: Sets variable `my_value` to "Hello World" (default)  
    - Connections: Leads to ➡️ This one is extra. Deactivate Me!  
    - Edge cases: Expression or assignment errors if configuration is altered improperly.

  - **➡️ This one is extra. Deactivate Me!**  
    - Type: NoOp (placeholder)  
    - Role: Demonstrates node deactivation shortcut (`D`)  
    - Connections: Leads to ➡️ We need another one. Duplicate Me!  
    - Edge cases: None; visual deactivation only.

  - **➡️ We need another one. Duplicate Me!**  
    - Type: NoOp (placeholder)  
    - Role: Demonstrates duplicating a node (`Ctrl + D`)  
    - Connections: Leads to ➡️ Copy Me  
    - Edge cases: None.

  - **➡️ Copy Me**  
    - Type: NoOp (placeholder)  
    - Role: Demonstrates copy (`Ctrl + C`), cut (`Ctrl + X`), and paste (`Ctrl + V`) operations  
    - Connections: Leads to ➡️ Delete me!  
    - Edge cases: Clipboard issues or copy-paste delays.

- **Sticky Notes:** Provide step-by-step tasks for using the shortcuts, e.g., renaming with `Space`, editing with `Enter`, deactivating with `D`, duplicating with `Ctrl+D`, and copying/pasting.

---

#### 2.2 Chapter 2: Canvas Navigation & Selection

- **Overview:**  
  This block teaches users how to efficiently move around the canvas and select nodes using multiple methods, including keyboard and mouse. It covers node adding, multiple selection, and zoom controls.

- **Nodes Involved:**  
  - Sticky Notes: Chapter 2 Header, Sticky Note17, Sticky Note18, Sticky Note19, Sticky Note20, Sticky Note27  
  - Nodes: ➡️ Let's add a new node, ➡️ Select these three nodes, Node 2, Node 3, Node A, Node B, Node C, ➡️ Now select everything, ➡️ Zoom out to see it all  

- **Node Details:**  
  - **➡️ Let's add a new node**  
    - Type: NoOp  
    - Role: Demonstrates adding a new node via node search (`Tab` key)  
    - Connections: None (end node)  

  - **➡️ Select these three nodes**  
    - Type: NoOp  
    - Role: Targets selection of multiple nodes (Node 2, Node 3, Node A)  
    - Connections: Leads to Node 2  

  - **Node 2 → Node 3 → Node A → Node B → Node C**  
    - Type: NoOp  
    - Role: Demonstrates a chain of nodes to be selected and navigated  
    - Connections: Sequential main connections from Node 2 to Node 3, Node A, Node B, Node C  
    - Edge cases: None; no active processing.

  - **➡️ Now select everything**  
    - Type: NoOp  
    - Role: Demonstrates `Ctrl + A` select all nodes and sticky notes shortcut  
    - Connections: Leads to ➡️ Zoom out to see it all  

  - **➡️ Zoom out to see it all**  
    - Type: NoOp  
    - Role: Demonstrates zooming out and fitting workflow to view (`1` key and `Ctrl + mouse wheel`)  
    - Connections: None  

- **Sticky Notes:** Describe multiple methods for selection (Ctrl-click, drag box, Ctrl+arrow keys), zoom shortcuts, and node navigation via arrow keys or edit panel.

---

#### 2.3 Chapter 3: Advanced Actions

- **Overview:**  
  This block focuses on advanced canvas operations: tidying up node layout, creating sub-workflows, and controlling canvas zoom for better workflow organization.

- **Nodes Involved:**  
  - Sticky Notes: Chapter 3 Header, Sticky Note21, Sticky Note26, Sticky Note  
  - Nodes: ➡️ Delete me!, ➡️ Add a Sticky Node  

- **Node Details:**  
  - **➡️ Delete me!**  
    - Type: NoOp  
    - Role: Demonstrates deleting a node with `Del` and undo/redo shortcuts (`Ctrl + Z`, `Ctrl + Shift + Z`)  
    - Connections: Leads to ➡️ Add a Sticky Node  

  - **➡️ Add a Sticky Node**  
    - Type: NoOp  
    - Role: Demonstrates adding a sticky note (`Shift + S`)  
    - Connections: None  

- **Sticky Notes:**  
  - Instructions for tidy up (`Shift + Alt + T`), creating sub-workflows (`Alt + X`), copying/cutting/pasting nodes (`Ctrl + C/X/V`), and deleting with undo/redo.

---

#### 2.4 Chapter 4: Execution & Debugging

- **Overview:**  
  This block guides users through execution and debugging shortcuts, including manual triggering, pinning data, and toggling execution data panels.

- **Nodes Involved:**  
  - Sticky Notes: Chapter 4 Header, Sticky Note22, Sticky Note24, Conclusion  
  - Nodes: Execute Set Node (Manual Trigger), ➡️ Pin My Data (Set node), ➡️ Open the Execution Data Panel  

- **Node Details:**  
  - **Execute Set Node**  
    - Type: Manual Trigger  
    - Role: Allows manual start of execution for testing  
    - Connections: Leads to ➡️ Pin My Data  

  - **➡️ Pin My Data**  
    - Type: Set node  
    - Role: Sets a variable `status` with value "Data is ready!" and demonstrates data pinning in execution panel  
    - Connections: Leads to ➡️ Open the Execution Data Panel  

  - **➡️ Open the Execution Data Panel**  
    - Type: NoOp  
    - Role: Demonstrates toggling execution data panel visibility with keyboard shortcuts (`L`, `O`, `I`)  
    - Connections: None  

- **Sticky Notes:**  
  - Steps for running the workflow, pinning data (`P`), and managing execution panel visibility.

---

#### 2.5 Miscellaneous Nodes

- **Test IF, Test Node, Test Node (1), Test Node (2), Schedule Trigger**  
  - These nodes form a simple scheduled trigger flow with an IF condition branching into two NoOp nodes.  
  - Likely included as placeholders or for internal testing; not directly involved in the tutorial flow.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                            |
|-------------------------------|---------------------|-----------------------------------------------|------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger    | Initiates schedule-based trigger               |                        | Test Node                 |                                                                                                                        |
| Test Node                     | NoOp                | Placeholder node after schedule                 | Schedule Trigger       | Test IF                   |                                                                                                                        |
| Test IF                       | IF                  | Conditional logic branching                      | Test Node              | Test Node (1), Test Node (2) |                                                                                                                        |
| Test Node (1)                 | NoOp                | Branch 1 after IF                                | Test IF                |                           |                                                                                                                        |
| Test Node (2)                 | NoOp                | Branch 2 after IF                                | Test IF                |                           |                                                                                                                        |
| Introduction                  | Sticky Note         | Workflow introduction and instructions          |                        |                           | # Tutorial - n8n Keyboard Shortcuts! Welcome! This workflow is an interactive playground...                              |
| Chapter 1 Header              | Sticky Note         | Header for Chapter 1                             |                        |                           | ## Chapter 1: Node Basics. Let's start with the fundamentals of interacting with a single node.                         |
| ➡️ First, Rename Me!          | NoOp                | Node to practice renaming                        |                        | ➡️ Now, Edit Me!          | **Task:** Select this node. Press **`Space`** to rename it. Call it "My First Node" and press Enter.                    |
| Sticky Note13                 | Sticky Note         | Explains renaming task                           |                        |                           | **Task:** Select this node. Press **`Space`** to rename it. Call it "My First Node" and press Enter.                    |
| ➡️ Now, Edit Me!              | Set                 | Node to practice editing node parameters        | ➡️ First, Rename Me!   | ➡️ This one is extra. Deactivate Me! | **Task:** Select this Set node. Press **`Enter`** to open its settings panel. Change the value from "Hello World" to your name! |
| Sticky Note14                 | Sticky Note         | Explains editing task                            |                        |                           | **Task:** Select this Set node. Press **`Enter`** to open its settings panel. Change the value from "Hello World" to your name! |
| ➡️ This one is extra. Deactivate Me! | NoOp          | Node to practice deactivating a node             | ➡️ Now, Edit Me!       | ➡️ We need another one. Duplicate Me! | **Task:** Select this node. Press **`D`** to deactivate it. Notice how it turns grey and would be skipped during execution. |
| Sticky Note15                 | Sticky Note         | Explains deactivation task                       |                        |                           | **Task:** Select this node. Press **`D`** to deactivate it. Notice how it turns grey and would be skipped during execution. |
| ➡️ We need another one. Duplicate Me! | NoOp          | Node to practice duplicating a node              | ➡️ This one is extra. Deactivate Me! | ➡️ Copy Me                | **Task:** Select this node. Press **`Ctrl + D`** to create an exact copy of it.                                         |
| Sticky Note16                 | Sticky Note         | Explains duplicating task                        |                        |                           | **Task:** Select this node. Press **`Ctrl + D`** to create an exact copy of it.                                         |
| ➡️ Copy Me                   | NoOp                | Node to practice copy/cut/paste                   | ➡️ We need another one. Duplicate Me! | ➡️ Delete me!              | **Task:** 1. Select this node and press **`Ctrl + C`** or **`Ctrl + X`**. 2. Click anywhere on the canvas. 3. Press **`Ctrl + V`**. |
| Sticky Note21                 | Sticky Note         | Explains copy/cut/paste task                      |                        |                           | **Task:** 1. Select this node and press **`Ctrl + C`** or **`Ctrl + X`**. 2. Click anywhere on the canvas. 3. Press **`Ctrl + V`**. |
| ➡️ Delete me!                | NoOp                | Node to practice deleting and undo/redo          | ➡️ Copy Me             | ➡️ Add a Sticky Node      | **Task:** 1. Select this node. 2. Press **`Del`** to delete. 3. Press **`Ctrl + Z`** to undo. 4. Press **`Ctrl + Shift + Z`** to redo. |
| Sticky Note25                 | Sticky Note         | Explains delete and undo/redo task                |                        |                           | **Task:** 1. Select this node. 2. Press **`Del`** to delete. 3. Press **`Ctrl + Z`** to undo. 4. Press **`Ctrl + Shift + Z`** to redo. |
| ➡️ Add a Sticky Node         | NoOp                | Node to practice adding sticky notes              | ➡️ Delete me!          |                           | **Task:** Add a sticky note. Press **`Shift + S`** to add a new Sticky Note.                                           |
| Sticky Note26                 | Sticky Note         | Explains adding sticky notes                       |                        |                           | **Task:** Add a sticky note. Press **`Shift + S`** to add a new Sticky Note.                                           |
| Chapter 2 Header             | Sticky Note         | Header for Chapter 2                               |                        |                           | ## Chapter 2: Canvas Navigation & Selection. Now let's learn how to move around the canvas and manage multiple nodes.    |
| ➡️ Let's add a new node      | NoOp                | Demonstrates adding nodes via node search          |                        |                           | **Task:** Press **`Tab`**, type "Set" and press Enter to add a new Set Node.                                           |
| Sticky Note17                | Sticky Note         | Explains adding nodes task                          |                        |                           | **Task:** Press **`Tab`** anywhere on the canvas. This opens the node search menu. Type "Set" and press Enter.           |
| ➡️ Select these three nodes  | NoOp                | Demonstrates selecting multiple nodes               |                        | Node 2                    | **Task:** Select multiple nodes via Ctrl-click, drag box, or Ctrl + arrow keys.                                        |
| Sticky Note18                | Sticky Note         | Explains multiple selection methods                 |                        |                           | **Task:** Select multiple nodes. Methods: Ctrl-click, drag box, Ctrl + arrow keys.                                    |
| Node 2                       | NoOp                | Part of the selection chain                          | ➡️ Select these three nodes | Node 3                    |                                                                                                                        |
| Node 3                       | NoOp                | Part of the selection chain                          | Node 2                 | Node A                    |                                                                                                                        |
| Node A                       | NoOp                | Part of the selection chain                          | Node 3                 | Node B                    |                                                                                                                        |
| Node B                       | NoOp                | Part of the selection chain                          | Node A                 | Node C                    |                                                                                                                        |
| Node C                       | NoOp                | Part of selection chain                              | Node B                 |                           |                                                                                                                        |
| ➡️ Now select everything     | NoOp                | Demonstrates select all nodes and sticky notes       |                        | ➡️ Zoom out to see it all | **Task:** Press **`Ctrl + A`** to select all nodes and sticky notes on the canvas.                                     |
| Sticky Note19                | Sticky Note         | Explains select all task                             |                        |                           | **Task:** Press **`Ctrl + A`** to select every single node and sticky note on the canvas.                              |
| ➡️ Zoom out to see it all    | NoOp                | Demonstrates zooming and fitting workflow to view    | ➡️ Now select everything |                           | **Task:** Press **`1`** to fit workflow to screen. Bonus: Hold **`Ctrl`** and scroll mouse wheel to zoom.               |
| Sticky Note20                | Sticky Note         | Explains zoom and fit task                           |                        |                           | **Task:** Press **`1`** to fit entire workflow. Hold **`Ctrl`** + mouse wheel to zoom in/out.                          |
| Chapter 3 Header             | Sticky Note         | Header for Chapter 3                                |                        |                           | ## Chapter 3: Advanced Actions. Tidying, sub-workflows, and zoom control.                                             |
| Sticky Note                  | Sticky Note         | Explains tidying and sub-workflow creation           |                        |                           | **Task:** Select nodes, press **`Shift + Alt + T`** to tidy up. Press **`Alt + X`** to create sub-workflow.             |
| Chapter 4 Header             | Sticky Note         | Header for Chapter 4                                |                        |                           | ## Chapter 4: Execution & Debugging. Essential shortcuts for testing workflows.                                        |
| Execute Set Node             | Manual Trigger      | Manual start for execution                           |                        | ➡️ Pin My Data            |                                                                                                                        |
| ➡️ Pin My Data               | Set                 | Pinning execution data                               | Execute Set Node       | ➡️ Open the Execution Data Panel | **Task:** After execution, select this node and press **`P`** to pin its data.                                        |
| Sticky Note22                | Sticky Note         | Explains data pinning task                           |                        |                           | **Task:** Click "Execute Workflow", then select this Set node and press **`P`** to pin data.                         |
| ➡️ Open the Execution Data Panel | NoOp            | Opening/hiding execution data panel                  | ➡️ Pin My Data         |                           | **Task:** Press **`L`** to open execution panel, **`O`** to toggle outputs, **`I`** to toggle inputs.                  |
| Sticky Note24                | Sticky Note         | Explains execution panel shortcuts                   |                        |                           | **Task:** Press **`L`** to open execution panel. Press **`O`** and **`I`** to toggle outputs and inputs.              |
| Conclusion                   | Sticky Note         | Workflow conclusion and final tips                   |                        |                           | ## 🎉 You've Mastered the Basics! Most important shortcut: **`Ctrl + S`** to save your workflow.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Introductory Sticky Notes**  
   - Add Sticky Notes with content:  
     - Introduction (welcome and instructions)  
     - Chapter 1 Header (Node Basics)  
     - Chapter 2 Header (Canvas Navigation & Selection)  
     - Chapter 3 Header (Advanced Actions)  
     - Chapter 4 Header (Execution & Debugging)  
     - Conclusion (final tips)  

2. **Chapter 1: Node Basics Nodes & Tasks**  
   - Add NoOp node named "➡️ First, Rename Me!"  
   - Add Set node named "➡️ Now, Edit Me!"  
     - Configure Set node to set parameter `my_value` as string with default "Hello World"  
   - Add NoOp node "➡️ This one is extra. Deactivate Me!"  
   - Add NoOp node "➡️ We need another one. Duplicate Me!"  
   - Add NoOp node "➡️ Copy Me"  
   - Add NoOp node "➡️ Delete me!"  
   - Add NoOp node "➡️ Add a Sticky Node"  
   - Connect nodes sequentially:  
     `➡️ First, Rename Me!` → `➡️ Now, Edit Me!` → `➡️ This one is extra. Deactivate Me!` → `➡️ We need another one. Duplicate Me!` → `➡️ Copy Me` → `➡️ Delete me!` → `➡️ Add a Sticky Node`

3. **Chapter 1 Sticky Notes Tasks**  
   - Add sticky notes near respective nodes describing keyboard shortcuts:  
     - Rename with `Space`  
     - Edit with `Enter`  
     - Deactivate with `D`  
     - Duplicate with `Ctrl + D`  
     - Copy/Cut/Paste with `Ctrl + C/X/V`  
     - Delete and Undo/Redo with `Del`, `Ctrl + Z`, `Ctrl + Shift + Z`  
     - Add Sticky Note with `Shift + S`  

4. **Chapter 2: Canvas Navigation & Selection**  
   - Add NoOp nodes:  
     - "➡️ Let's add a new node"  
     - "➡️ Select these three nodes"  
     - "Node 2", "Node 3"  
     - "Node A", "Node B", "Node C" (linked sequentially: Node 2 → Node 3 → Node A → Node B → Node C)  
     - "➡️ Now select everything"  
     - "➡️ Zoom out to see it all"  
   - Connect:  
     `➡️ Select these three nodes` → `Node 2` → `Node 3` → `Node A` → `Node B` → `Node C`  
     `➡️ Now select everything` → `➡️ Zoom out to see it all`  

5. **Chapter 2 Sticky Notes**  
   - Add sticky notes explaining:  
     - Adding nodes with `Tab` key and search  
     - Multiple selection methods (Ctrl-click, drag, Ctrl+arrow keys)  
     - Select all with `Ctrl + A`  
     - Zooming and fitting with `1` and `Ctrl + mouse wheel`  
     - Moving between nodes with arrow keys and edit panel  

6. **Chapter 3: Advanced Actions**  
   - Add NoOp nodes:  
     - "➡️ Delete me!" (connected from `➡️ Copy Me`)  
     - "➡️ Add a Sticky Node" (connected from `➡️ Delete me!`)  
   - Add sticky notes describing:  
     - Tidy up with `Shift + Alt + T`  
     - Create sub-workflows with `Alt + X`  
     - Copy/Paste/Delete and Undo/Redo shortcuts  

7. **Chapter 4: Execution & Debugging**  
   - Add Manual Trigger node "Execute Set Node"  
   - Add Set node "➡️ Pin My Data"  
     - Set parameter `status` to string "Data is ready!"  
   - Add NoOp node "➡️ Open the Execution Data Panel"  
   - Connect:  
     `Execute Set Node` → `➡️ Pin My Data` → `➡️ Open the Execution Data Panel`  
   - Add sticky notes explaining:  
     - Manual execution trigger  
     - Pinning data with `P`  
     - Opening execution panel with `L`, toggle outputs with `O`, inputs with `I`  

8. **Miscellaneous / Testing Nodes (Optional)**  
   - Add Schedule Trigger node ("Schedule Trigger")  
   - Add NoOp node ("Test Node") → IF node ("Test IF")  
   - IF branches to two NoOp nodes ("Test Node (1)" and "Test Node (2)") for conditional flow demonstration  

9. **Final Touches**  
   - Arrange all nodes and sticky notes in a logical top-to-bottom layout reflecting chapters and flow.  
   - Assign colors to sticky notes for visual clarity (e.g., blue for intro, green for navigation, orange for advanced, purple for debugging).  
   - Verify all connections and test keyboard shortcuts in n8n UI as per sticky note instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The tutorial emphasizes the `Ctrl + S` shortcut as the most important for saving workflows frequently.         | Conclusion sticky note                                                                              |
| This workflow is designed as an interactive playground; users must follow sticky note instructions sequentially.| Introduction sticky note                                                                            |
| Useful keyboard shortcuts: `Space` (rename), `Enter` (edit), `D` (deactivate), `Ctrl+D` (duplicate), `Ctrl+C/X/V` (copy/cut/paste), `Del` (delete), `Ctrl+Z` / `Ctrl+Shift+Z` (undo/redo), `Shift+S` (add sticky note), `Tab` (node search), `Ctrl+A` (select all), `1` (zoom fit), `Shift+Alt+T` (tidy up), `Alt+X` (sub-workflow creation), `P` (pin data), `L/O/I` (execution panel toggling). | Entire workflow; summarized for reference                                                            |
| For more keyboard shortcuts and n8n usage tips, consult the official n8n documentation and blog.               | Official n8n docs: https://docs.n8n.io/ and blog: https://n8n.io/blog/                               |

---

This document fully describes the structure, nodes, and instructional content of the "Tutorial - n8n Keyboard Shortcuts" workflow, allowing both human users and automated agents to understand, reproduce, and modify the workflow confidently.