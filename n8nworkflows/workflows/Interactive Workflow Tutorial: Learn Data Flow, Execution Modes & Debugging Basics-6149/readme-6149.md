Interactive Workflow Tutorial: Learn Data Flow, Execution Modes & Debugging Basics

https://n8nworkflows.xyz/workflows/interactive-workflow-tutorial--learn-data-flow--execution-modes---debugging-basics-6149


# Interactive Workflow Tutorial: Learn Data Flow, Execution Modes & Debugging Basics

### 1. Workflow Overview

This workflow is an interactive tutorial designed to teach users the fundamental concepts of n8n workflows, including data flow, execution modes, and debugging basics. It targets users who have programming experience (JavaScript, REST APIs, JSON) and want a live, practical introduction to n8n’s core operational model.

The workflow is structured into logical blocks that demonstrate fundamental n8n principles:

- **1.1 Introduction & Setup**: Initial sticky notes with instructions and a Form trigger to start the tutorial.
- **1.2 Basic Flow Control & Data Passing**: Demonstrations of node behavior, sequential execution, data passing, and branch merging.
- **1.3 Execution Modes & Looping**: Examples of item-based executions, looping using SplitInBatches, Aggregate and SplitOut nodes, and controlling execution flow count.
- **1.4 Advanced Data Referencing & Debugging**: Correct referencing of data from previous nodes, debugging with logs, and final wrap-up with congratulatory notes.

Each block uses a combination of Form nodes for interactive user input, Code nodes for dynamic data generation, Set nodes for data assignment, Merge nodes for combining data paths, and NoOp nodes for flow control and commentary.


---

### 2. Block-by-Block Analysis

#### 1.1 Introduction & Setup

**Overview:**  
This block introduces the user to the tutorial, sets expectations, and starts the workflow with a Form trigger that opens an interactive web form.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Form Start

**Node Details:**

- **Sticky Note, Sticky Note1, Sticky Note2**  
  - Type: Sticky Note  
  - Role: Provide instructional and preparatory information to the user.  
  - Configuration: Text content with links (e.g., YouTube videos), setup instructions, and encouragement to use dark theme.  
  - Input/Output: None (purely visual).  
  - Edge cases: None.

- **Form Start**  
  - Type: Form Trigger  
  - Role: Entry point trigger node that starts the workflow when the user clicks "Begin" on the web form.  
  - Configuration: Form titled "Let's Learn N8N! Lesson 1" with an HTML field providing notes and warnings about node inspection and execution.  
  - Webhook: Exposes a webhook URL for the form.  
  - Output: Triggers downstream nodes on form submission.  
  - Edge cases: Webhook failures, user closing browser unexpectedly during execution.

---

#### 1.2 Basic Flow Control & Data Passing

**Overview:**  
Demonstrates basic flow control (sequential execution), the effect of branching on execution order, data passing between nodes, and how to merge data streams.

**Nodes Involved:**  
- Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8, Sticky Note13  
- First NoOp node (FIRST)  
- Next Form (form navigation)  
- Do Thing 1, Do Thing 2 (forms)  
- Set Value 1e, Set Value 1f, Set Value 1g, Set Value 1g2, Set Value 1h (Set nodes assigning variables)  
- Value Passed Form, Can See It, Cant See It, Can See Both (forms displaying passed data)  
- Merge, Lets Fix It... (NoOp), Merge node  
- Do Something Unrelated (Set)  
- Reference Examples Form (Form node)

**Node Details:**

- **Sticky Notes 3 to 8 and 13**  
  - Serve as commentary explaining no-op nodes, form navigation, flow control (top path executes first), data passing as side effects, and data visibility depending on flow branches.  
  - Emphasize best practices for referencing previous node data explicitly.

- **FIRST (NoOp)**  
  - Role: Acts as a marker or placeholder to organize flow.

- **Next Form**  
  - Form node prompting user to continue after reading notes.

- **Do Thing 1, Do Thing 2**  
  - Form nodes representing two sequential tasks triggered from Next Form.  
  - Demonstrate branching and sequential execution order.

- **Set Value 1e, 1f, 1g, 1g2, 1h**  
  - Set nodes assigning string variables (my_value_1e, my_value_1f, etc.).  
  - Show how data is created and passed downstream.

- **Value Passed Form, Can See It, Cant See It, Can See Both**  
  - Form nodes to illustrate data availability downstream and across branches.  
  - "Cant See It" shows inability to access data from a different branch without merging.  
  - "Can See Both" demonstrates successful data merging.

- **Merge**  
  - Merge node set to "Combine" mode, combining data from multiple branches by position.  
  - Important for joining data streams from branched flows.

- **Lets Fix It... (NoOp)**  
  - Controls flow to the Set Value nodes after merging, showing fixing data visibility.

- **Do Something Unrelated**  
  - Set node assigning unrelated variable (city) to demonstrate isolated data scope.

- **Reference Examples Form**  
  - Shows correct syntax for referencing data from a specifically named node.  
  - Form description includes examples of variable interpolation with explicit node references.

**Edge Cases:**  
- Misunderstanding data visibility across branches.  
- Forgetting to merge branches before accessing combined data.  
- Incorrect variable referencing causing undefined values.  
- User skipping form steps.

---

#### 1.3 Execution Modes & Looping

**Overview:**  
Explores how n8n executes nodes per item or once for all items, demonstrates looping over batches, aggregation, splitting, and controlling execution frequency.

**Nodes Involved:**  
- Sticky Note10, Sticky Note11, Sticky Note12, Sticky Note14, Sticky Note15, Sticky Note16, Sticky Note17, Sticky Note21, Sticky Note22, Sticky Note23  
- Create Three Items, Create Four Items, Create Five Items, Create Set A, Create Set B, Create Set A1, Create Set B1 (Code nodes generating arrays of items)  
- Runs Three Times!, Runs Once!, Runs Five Times! (Code nodes with different execution modes)  
- Loop Over Items (SplitInBatches)  
- Aggregate Items, Aggregate Sets, Aggregate (Aggregate nodes combining items)  
- Split Out (SplitOut node)  
- Get This Item (Set node extracting current loop item and index)  
- 2.a Notes, 2.b Notes, 2.c Notes, 2e Notes1, 2c Notes, 2f Notes 1, 2f Notes 2 (Form nodes with explanations)  
- Only Once (NoOp with "Only Once" execution setting)  
- NOP, NOP2 (NoOp nodes for flow control)  
- FIRST TASK, SECOND TASK, THIRD, SECOND TASK, FINALLY (NoOp nodes used for flow sequencing)  
- Merge1 (Merge node with 3 inputs)  

**Node Details:**

- **Create Three/Four/Five/Set A/B/A1/B1 (Code)**  
  - Generate test data arrays representing users with id, name, email, active flag, and createdAt timestamp.  
  - Different ranges of IDs for diversity.

- **Runs Three Times!, Runs Five Times!**  
  - Code nodes set to run once per each item, logging to console, demonstrating per-item execution.  
  - Return each item as is.

- **Runs Once!**  
  - Code node configured to run once for all items, referencing the full array.  
  - Returns a single empty item to avoid multiple downstream executions.

- **Loop Over Items (SplitInBatches)**  
  - Controls batch size for processing items one at a time, showing looping behavior.

- **Aggregate Items, Aggregate Sets, Aggregate**  
  - Combine multiple items into a single array field.  
  - Demonstrate how aggregation reduces multiple items to one.

- **Split Out**  
  - Reverse of aggregate; splits array field into multiple executions.

- **Get This Item**  
  - Extracts current batch item and index from context for use downstream.

- **Only Once (NoOp)**  
  - Configured to execute only once regardless of incoming multiple items, demonstrating execution flow control.

- **FIRST TASK, SECOND TASK, THIRD, SECOND TASK, FINALLY (NoOp)**  
  - Control points to demonstrate sequential execution of multiple flows.

- **Merge1**  
  - Merge node with three inputs combining multiple data sets for final processing.

- **Sticky Notes & Form Notes**  
  - Provide detailed explanations and instructions related to execution modes, looping, aggregation, splitting, and debugging.

**Edge Cases:**  
- Misunderstanding item-based executions causing unexpected multiple runs of nodes.  
- Incorrect use of aggregation leading to unexpected data shapes.  
- Forgetting to manage execution count causing duplicate downstream operations.  
- User confusion navigating forms while workflow executes multiple times.

---

#### 1.4 Advanced Data Referencing & Debugging

**Overview:**  
Focuses on best practices for referencing data from previous nodes, using logs for debugging, and concluding the tutorial.

**Nodes Involved:**  
- Sticky Note15, Sticky Note16  
- Set Value 1h, Do Something Unrelated, Reference Examples Form, Logs, Congrats, End

**Node Details:**

- **Set Value 1h**  
  - Assigns variable `my_value_1h` used for demonstration of explicit referencing.

- **Do Something Unrelated**  
  - Assigns unrelated variable `city` to show isolated data.

- **Reference Examples Form**  
  - Form displaying examples of proper data referencing syntax:  
    - Incorrect: `{{ $json.my_value_1h }}`  
    - Correct: `{{ $('Set Value 1h').item.json.my_value_1h }}`

- **Logs (Form)**  
  - Instructs user to open the logs panel in n8n for workflow debugging.  
  - Encourages inspection of node inputs and outputs.

- **Congrats (Form)**  
  - Final congratulatory message encouraging users to apply learned concepts and explore further lessons.  
  - Includes a redirect URL to n8n creator’s page.

- **End (Form)**  
  - Completion node that redirects user to an external URL after finishing the lesson.

**Edge Cases:**  
- Misreferencing variables causing data access errors.  
- User unfamiliarity with logs panel leading to missed debugging opportunities.  
- Unexpected workflow termination or user navigation away before completion.

---

### 3. Summary Table

| Node Name            | Node Type         | Functional Role                               | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                         |
|----------------------|-------------------|-----------------------------------------------|--------------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------|
| Sticky Note          | Sticky Note       | Provides initial setup instructions           | None                                 | None                                 | ## Before we begin... - Create your n8n account - Watch [these 9 videos](https://www.youtube.com/watch?v=4BVTkqbn_tY&t=30s&ab_channel=n8n). - Have chatgpt (or windsurf etc.) open at all times... |
| Sticky Note1         | Sticky Note       | Introduces the lesson and prerequisites       | None                                 | None                                 | # Let's Learn n8n! Lesson 1 ... You should have familiarity with programming, JS, REST API, JSON.  |
| Sticky Note2         | Sticky Note       | Explains triggers and starting the workflow   | None                                 | None                                 | ## Triggers (Start Here) ... Go ahead and click EXECUTE WORKFLOW now.                             |
| Form Start           | Form Trigger      | Entry point trigger with initial form         | None                                 | THIRD, FIRST, SECOND                  |                                                                                                   |
| Sticky Note3         | Sticky Note       | Explains No Operation node purpose             | None                                 | None                                 | ## 1.a No Operation node ... handy to add comments or control layout.                             |
| Sticky Note4         | Sticky Note       | Explains Form Next node usage                    | None                                 | None                                 | ## 1.b Form Next ... used to control workflow execution by user interaction.                      |
| Next Form            | Form              | Presents next step form to user                 | FIRST                                | Do Thing 1, Do Thing 2                |                                                                                                   |
| Sticky Note5         | Sticky Note       | Explains flow control order                      | None                                 | None                                 | ## 1.c Flow Control ... TOP PATH executes first, then BOTTOM PATH.                               |
| Do Thing 1           | Form              | First form task in flow                          | Next Form                           | Set Value 1e                        |                                                                                                   |
| Do Thing 2           | Form              | Second form task in flow                         | Next Form                           | Set Value 1e                        |                                                                                                   |
| Set Value 1e         | Set               | Assigns variable my_value_1e                     | Do Thing 2                         | Value Passed Form                    |                                                                                                   |
| Sticky Note6         | Sticky Note       | Explains data passing as side effect             | None                                 | None                                 | ## 1.e Data ... Data is passed downstream as a side effect.                                      |
| Value Passed Form    | Form              | Shows passed value in form                        | Set Value 1e                       | Set Value 1f, Cant See It            |                                                                                                   |
| Set Value 1f         | Set               | Assigns variable my_value_1f                     | Value Passed Form                  | Can See It                         |                                                                                                   |
| Can See It           | Form              | Form that can access passed data                  | Set Value 1f                      | Lets Fix It...                     |                                                                                                   |
| Cant See It          | Form              | Form demonstrating inability to access data on other branch | Value Passed Form                  | Lets Fix It...                     |                                                                                                   |
| Sticky Note7         | Sticky Note       | Explains data visibility on branches              | None                                 | None                                 | ## 1.f Data & Flow ... If not downstream, you CAN'T access data.                                 |
| Lets Fix It...       | NoOp              | Controls flow to fix data visibility              | Can See It, Cant See It             | Set Value 1g2, Set Value 1g        |                                                                                                   |
| Set Value 1g         | Set               | Assigns my_value_1g for merged data example       | Lets Fix It...                    | Merge                            |                                                                                                   |
| Set Value 1g2        | Set               | Assigns my_value_1g2 for merged data example      | Lets Fix It...                    | Merge                            |                                                                                                   |
| Merge                | Merge             | Combines data from branches                         | Set Value 1g, Set Value 1g2        | Can See Both                     | ## 1.g Merge ... settings "Combine" to join data streams.                                       |
| Can See Both         | Form              | Form showing merged data access                     | Merge                             | Set Value 1h                     |                                                                                                   |
| Set Value 1h         | Set               | Assigns my_value_1h variable                         | Can See Both                     | Do Something Unrelated          |                                                                                                   |
| Do Something Unrelated| Set               | Assigns unrelated data "city"                        | Set Value 1h                     | Reference Examples Form          |                                                                                                   |
| Reference Examples Form| Form             | Shows correct and incorrect data referencing        | Do Something Unrelated            | Logs                           |                                                                                                   |
| Sticky Note8         | Sticky Note       | Explains Merge node usage                            | None                            | None                           | ## 1.g Merge ... options and use cases.                                                        |
| Sticky Note13        | Sticky Note       | Explains explicit data referencing best practices   | None                            | None                           | ## 1.h Data From Previous Nodes ... use explicit references with node names for clarity.        |
| Sticky Note15        | Sticky Note       | Explains use of Logs panel for debugging             | None                            | None                           | ## 1.i Logs ... vital debugging tool, inspect node data in logs.                              |
| Logs                 | Form              | Form instructing user to inspect logs panel           | Reference Examples Form          | None                           |                                                                                                   |
| Sticky Note16        | Sticky Note       | Additional notes on data referencing                  | None                            | None                           |                                                                                                   |
| Sticky Note10        | Sticky Note       | Explains looping concept using SplitInBatches          | None                            | None                           | ## 2.d Looping ... process items one at a time with Loop node.                                |
| Sticky Note11        | Sticky Note       | Section 2 header placeholder                           | None                            | None                           | ## 2.                                                                                         |
| Sticky Note12        | Sticky Note       | Section 3 header placeholder                           | None                            | None                           | ## 3.                                                                                         |
| Sticky Note14        | Sticky Note       | Completion note                                         | None                            | None                           | ## Completed Lesson 1                                                                        |
| Sticky Note17        | Sticky Note       | Explains multiple execution concepts                     | None                            | None                           | ## 2.a Each Item Becomes an Execution ... multiple executions for multiple items.             |
| Sticky Note21        | Sticky Note       | Explains Split node usage                               | None                            | None                           | ## 2.c Split ... split aggregated array back into items.                                    |
| Sticky Note22        | Sticky Note       | Explains "Only Once" execution setting                   | None                            | None                           | ## 2.e Only Once ... reduce multiple executions to a single one.                             |
| Sticky Note23        | Sticky Note       | Explains flow execution combining misconception         | None                            | None                           | ## 2.f Execution Flows Don't Combine ... use sequential flow steps instead.                  |
| Create Three Items   | Code              | Generates 3 test user items                              | SECOND                          | Runs Three Times!, Runs Once!, 2.a Notes |                                                                                                   |
| Runs Three Times!    | Code              | Runs once per item, logs user names                      | Create Three Items              | 2.a Notes                      |                                                                                                   |
| Runs Once!           | Code              | Runs once for all items, logs all user names             | Create Three Items              | 2.a Notes                      |                                                                                                   |
| 2.a Notes            | Form              | Notes on execution count and logging                      | Runs Three Times!               | Create Five Items              |                                                                                                   |
| Create Five Items    | Code              | Generates 5 test user items                              | 2.a Notes                     | Aggregate Items                |                                                                                                   |
| Aggregate Items      | Aggregate         | Aggregates 5 items into 1 item with array field           | Create Five Items              | 2.b Notes, Split Out           |                                                                                                   |
| 2.b Notes            | Form              | Notes on aggregate output                                  | Aggregate Items                | Create Five Items              |                                                                                                   |
| Split Out            | SplitOut          | Splits aggregated array back to multiple items            | Aggregate Items                | Runs Five Times!, 2.c Notes    |                                                                                                   |
| Runs Five Times!     | Code              | Runs once per item, logs each user                          | Split Out                     | 2.c Notes                     |                                                                                                   |
| 2.c Notes            | Form              | Notes on split and per-item execution                       | Runs Five Times!               | Create Four Items             |                                                                                                   |
| Create Four Items    | Code              | Generates 4 test user items                              | 2.c Notes                     | Loop Over Items               |                                                                                                   |
| Loop Over Items      | SplitInBatches    | Processes items one at a time, controls loop execution     | Create Four Items             | Only Once, Get This Item       |                                                                                                   |
| Only Once            | NoOp              | Limits execution downstream to one despite multiple inputs | Loop Over Items               | 2e Notes1                    |                                                                                                   |
| 2e Notes1            | Form              | Notes on "Only Once" setting                                | Only Once                     | NOP                         |                                                                                                   |
| NOP                  | NoOp              | Dummy node for flow control                                 | 2e Notes1                    | None                        |                                                                                                   |
| Get This Item        | Set               | Extracts current item and index from loop                    | Loop Over Items               | 2d Once for Each Form         |                                                                                                   |
| 2d Once for Each Form | Form              | Form shown once per item in loop                             | Get This Item                 | Loop Over Items               |                                                                                                   |
| Create Set A1        | Code              | Generates user items with id 0-4                            | NOP                          | Aggregate Sets                |                                                                                                   |
| Create Set B1        | Code              | Generates user items with id 5-10                            | NOP                          | Aggregate Sets                |                                                                                                   |
| Aggregate Sets       | Aggregate         | Aggregates multiple sets into one array                      | Create Set A1, Create Set B1  | 2f Notes 1                   |                                                                                                   |
| 2f Notes 1           | Form              | Notes on execution flow combination                           | Aggregate Sets               | FIRST TASK, SECOND TASK, FINALLY |                                                                                                   |
| FIRST TASK           | NoOp              | Controls first task flow                                      | 2f Notes 1                  | Create Set A                 |                                                                                                   |
| Create Set A         | Code              | Generates user items with id 0-4                            | FIRST TASK                  | Merge1                      |                                                                                                   |
| SECOND TASK          | NoOp              | Controls second task flow                                     | 2f Notes 1                  | Create Set B                 |                                                                                                   |
| Create Set B         | Code              | Generates user items with id 5-10                            | SECOND TASK                 | Merge1                      |                                                                                                   |
| FINALLY              | NoOp              | Final control node for flow sequencing                        | 2f Notes 1                  | Merge1                      |                                                                                                   |
| Merge1               | Merge             | Merges multiple sets from tasks                               | Create Set A, Create Set B, FINALLY | Aggregate                  |                                                                                                   |
| Aggregate            | Aggregate         | Aggregates merged data into a single array                     | Merge1                      | 2f Notes 2                  |                                                                                                   |
| 2f Notes 2           | Form              | Final notes showing combined result count                      | Aggregate                   | NOP2                        |                                                                                                   |
| NOP2                 | NoOp              | Final dummy node after completion                              | 2f Notes 2                 | 2f Notes 1                  |                                                                                                   |
| Sticky Note14        | Sticky Note       | Lesson completion message                                     | None                       | None                       | ## Completed Lesson 1                                                                         |
| THIRD                | NoOp              | Final flow control node after Form Start                       | Form Start                 | Congrats                    |                                                                                                   |
| Congrats              | Form              | Congratulatory message with next steps                         | THIRD                      | End                        |                                                                                                   |
| End                   | Form              | Final completion redirect to external resource                 | Congrats                   | None                       |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Introduction:**

   - Add three Sticky Note nodes with the texts provided in Sticky Note, Sticky Note1, Sticky Note2 positions.  
   - Configure colors and sizes as per original (optional for clarity).

2. **Create Form Trigger (Form Start):**

   - Add a Form Trigger node named "Form Start".  
   - Configure webhook ID (auto-generated), set form title to "Let's Learn N8N! Lesson 1".  
   - Add an HTML field with tutorial instructions about inspecting nodes and stopping execution.  
   - Connect no inputs (starting node).

3. **Create Initial Flow Control NoOp nodes:**

   - Add NoOp nodes named "FIRST", "SECOND", and "THIRD" for flow control.  
   - Connect "Form Start" main output to each of these nodes.

4. **Create First Interactive Form (Next Form):**

   - Add a Form node named "Next Form" with title "Let's Begin...", button "Continue", and a form description to encourage workspace exploration.  
   - Connect output of "FIRST" to "Next Form".

5. **Create Parallel Forms "Do Thing 1" and "Do Thing 2":**

   - Add two Form nodes named "Do Thing 1" and "Do Thing 2", each with their form titles and descriptions indicating execution order.  
   - Connect "Next Form" main output to both.

6. **Set Variable my_value_1e:**

   - Add a Set node named "Set Value 1e" assigning `my_value_1e` to string "10".  
   - Connect "Do Thing 2" output to this node.

7. **Create Value Passed Form:**

   - Add a Form node named "Value Passed Form" showing `{{ $json.my_value_1e }}` in its description.  
   - Connect "Set Value 1e" to this form.

8. **Set Variable my_value_1f:**

   - Add Set node "Set Value 1f" setting `my_value_1f` to "20".  
   - Connect "Value Passed Form" output to "Set Value 1f".

9. **Create Forms "Can See It" and "Cant See It":**

   - Create two Form nodes named "Can See It" and "Cant See It" with descriptions referencing `my_value_1f`.  
   - Connect "Set Value 1f" to "Can See It".  
   - Connect "Value Passed Form" to "Cant See It".

10. **Fix Data Visibility with NoOp and Set nodes:**

    - Add NoOp "Lets Fix It...". Connect "Can See It" and "Cant See It" to it.  
    - Add two Set nodes "Set Value 1g" and "Set Value 1g2" assigning strings "100" and "200" to their respective vars.  
    - Connect "Lets Fix It..." to both Set nodes.

11. **Merge Data:**

    - Add Merge node "Merge" with mode "Combine" and combineBy "Position".  
    - Connect "Set Value 1g" and "Set Value 1g2" into the Merge node.

12. **Form "Can See Both":**

    - Add Form node "Can See Both" with description referencing `my_value_1g` and `my_value_1g2`.  
    - Connect "Merge" to this form.

13. **Set Variable my_value_1h and Do Something Unrelated:**

    - Add Set node "Set Value 1h" assigning `my_value_1h` = "1001".  
    - Connect "Can See Both" to it.  
    - Add Set node "Do Something Unrelated" assigning `city` = "seattle".  
    - Connect "Set Value 1h" to this node.

14. **Form "Reference Examples Form":**

    - Add Form node with description showing incorrect and correct ways to reference `my_value_1h`.  
    - Connect "Do Something Unrelated" to this form.

15. **Form "Logs":**

    - Add Form node instructing user to open logs panel for debugging.  
    - Connect "Reference Examples Form" to "Logs".

16. **Create Code Nodes to Demonstrate Execution Modes:**

    - Add "Create Three Items" code node generating 3 user objects.  
    - Connect "SECOND" to this node.  
    - Add "Runs Three Times!" code node set to run once per item, logging item name. Connect "Create Three Items" to it.  
    - Add "Runs Once!" code node set to run once for all items, logging all names. Connect "Create Three Items" to it.  
    - Add form "2.a Notes" connected from "Runs Three Times!".

17. **Create Five Items & Aggregate:**

    - Add "Create Five Items" code node generating 5 users, connected from "2.a Notes".  
    - Add "Aggregate Items" node aggregating all items into "all_items". Connect "Create Five Items" to this.  
    - Add form "2.b Notes" connected from "Aggregate Items".

18. **Split Out & Per-Item Execution:**

    - Add "Split Out" node splitting "all_items". Connect "Aggregate Items" to it.  
    - Add "Runs Five Times!" code node running per item logging name, connected from "Split Out".  
    - Add form "2.c Notes" connected from "Runs Five Times!".

19. **Create Four Items & Loop Over Items:**

    - Add "Create Four Items" code node connected from "2.c Notes".  
    - Add "Loop Over Items" (SplitInBatches) connected from "Create Four Items".  
    - Connect "Loop Over Items" into two branches: "Only Once" NoOp and "Get This Item" Set node.

20. **Only Once & Notes:**

    - Configure "Only Once" NoOp node with setting to execute only once.  
    - Connect "Only Once" to form "2.e Notes1".  
    - Connect form "2.e Notes1" to NoOp "NOP".

21. **Get This Item & Once For Each Form:**

    - Configure "Get This Item" Set node extracting `this_item_index` and `this_item` from context.  
    - Connect it to form "2d Once for Each Form".  
    - Connect "2d Once for Each Form" back to "Loop Over Items" for continuous looping.

22. **Create Set A1, Set B1 & Aggregate Sets:**

    - Add "Create Set A1" code (items 0-4) connected from "NOP".  
    - Add "Create Set B1" code (items 5-10) connected from "NOP".  
    - Add "Aggregate Sets" node aggregating all data from these two code nodes.

23. **Flow Control and Final Merge:**

    - Connect "Aggregate Sets" to form "2f Notes 1".  
    - Connect "2f Notes 1" to three NoOp nodes: "FIRST TASK", "SECOND TASK", "FINALLY".  
    - Connect "FIRST TASK" to "Create Set A" code node (items 0-4).  
    - Connect "SECOND TASK" to "Create Set B" code node (items 5-10).  
    - Connect "FINALLY" directly to "Merge1" node with 3 inputs.  
    - Connect "Create Set A" and "Create Set B" to "Merge1" inputs.

24. **Final Aggregation and Notes:**

    - Add "Aggregate" node aggregating merged data from "Merge1".  
    - Connect to form "2f Notes 2".  
    - Connect "2f Notes 2" to NoOp "NOP2".

25. **Finishing Steps:**

    - Connect "Form Start" to "THIRD" NoOp.  
    - Connect "THIRD" to form "Congrats" with congratulatory message and link.  
    - Connect "Congrats" to form "End" with redirect URL to n8n creator page.

26. **Add all remaining Sticky Notes at appropriate positions for instructional content.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Before starting, users are encouraged to create an n8n account, watch a curated set of 9 videos, and have ChatGPT or alternative AI assistants open for help. Dark mode in n8n is recommended for better readability.                      | Sticky Note at beginning with link: https://www.youtube.com/watch?v=4BVTkqbn_tY&t=30s&ab_channel=n8n                                                             |
| This workflow is a live script designed to keep pace with the latest n8n version, unlike many outdated tutorials on YouTube.                                                                                                            | Sticky Note1 content                                                                                                                                               |
| Users must have basic knowledge of programming, JavaScript/TypeScript, REST APIs, and JSON for the best learning experience.                                                                                                            | Sticky Note1                                                                                                                                                      |
| The logs panel in n8n is a critical debugging tool where users can inspect inputs and outputs of each node after execution.                                                                                                              | Sticky Note15                                                                                                                                                     |
| Detailed explanation of execution modes: per-item execution vs. running once for all items; use of SplitInBatches for looping; Aggregate and SplitOut for data collection and splitting.                                                    | Sticky Notes 17, 21, 22, 23                                                                                                                                      |
| Best practice is to use explicit node referencing for data access to avoid confusion and errors.                                                                                                                                          | Sticky Note13, Reference Examples Form                                                                                                                            |
| The workflow concludes by encouraging users to create their own workflows and offers a link to further lessons and solutions to common problems.                                                                                         | Congrats form and link: https://n8n.io/creators/wyeth/                                                                                                           |

---

**Disclaimer:** The provided text and workflow are from an automated n8n workflow designed for legal and public content, strictly respecting all content policies and including no illegal or offensive material.