Implement Recursive Algorithms with Sub-workflows: Towers of Hanoi Demo

https://n8nworkflows.xyz/workflows/implement-recursive-algorithms-with-sub-workflows--towers-of-hanoi-demo-8656


# Implement Recursive Algorithms with Sub-workflows: Towers of Hanoi Demo

### 1. Workflow Overview

This workflow demonstrates how to implement the recursive algorithm for the classic Towers of Hanoi puzzle using n8n's sub-workflows. It serves as a proof of concept for recursion handling within n8n by:

- Recursively solving the Towers of Hanoi problem for a given number of discs.
- Using sub-workflows to model recursive calls.
- Maintaining and updating stacks (rods) representing disc positions.
- Logging each move and formatting the final solution as human-readable instructions.

The workflow logically separates into these blocks:

- **1.1 Input Reception and Initialization:** Accepts the number of discs and initializes stacks representing rods A, B, and C.
- **1.2 Recursive Decision (Base Case and Recursive Calls):** Checks if only one disc remains (base case) or recurses on smaller subproblems.
- **1.3 Disc Movement and Stack Updates:** Moves discs between stacks and logs the steps.
- **1.4 Recursive Sub-workflow Invocations:** Calls the main workflow recursively with updated parameters to simulate recursive calls.
- **1.5 Result Formatting:** Converts the logged moves into a formatted textual solution.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block sets the initial number of discs, enforces constraints on the allowed disc count, and initializes the stacks representing rods A, B, and C with their respective discs.

- **Nodes Involved:**  
  - Start (Manual Trigger)  
  - Set number of discs  
  - Create stacks  

- **Node Details:**

  - **Start**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually.  
    - Input/Output: No input; triggers the workflow start.  
    - Edge cases: None.

  - **Set number of discs**  
    - Type: Set  
    - Role: Defines a numeric variable `numberOfDiscs` (default 4).  
    - Configuration: Sets `numberOfDiscs` to 4 initially.  
    - Input: Triggered by Start node.  
    - Output: Passes data to `Create stacks`.  
    - Edge cases: None.

  - **Create stacks**  
    - Type: Code (JavaScript)  
    - Role: Validates `numberOfDiscs` (min 1, max 5) and creates three stacks (A, B, C) with discs initialized on stack A in descending order. Also initializes an empty logs array.  
    - Key logic:  
      - Limits discs between 1 and 5 to avoid recursion depth issues.  
      - Prepares `stackA`, `stackB`, `stackC` objects with `name` and `items` arrays.  
      - `stackA.items` contains discs [numberOfDiscs ... 1].  
      - `stackB.items` and `stackC.items` are empty arrays.  
      - Logs array is empty.  
    - Input: Receives `numberOfDiscs` from previous node.  
    - Output: Passes stacks and logs to sub-workflow `A B C`.  
    - Edge cases: Input number outside allowed range adjusted automatically.

---

#### 2.2 Recursive Decision (Base Case and Recursive Calls)

- **Overview:**  
  Decides whether the current recursion step is the base case (one disc) or requires further recursion. Directs flow accordingly.

- **Nodes Involved:**  
  - A B C (Execute Workflow - recursive call)  
  - Max 1 disc (If node)  
  - X to Z (Code)  
  - X Z Y (Execute Workflow - recursive call)  
  - Y X Z (Execute Workflow - recursive call)  
  - Update (Code)  
  - Update (Code)  

- **Node Details:**

  - **A B C**  
    - Type: Execute Workflow (Sub-workflow call)  
    - Role: Recursively calls the main workflow with inputs: current stacks (A, B, C), logs, and `numberOfDiscs`.  
    - Configuration: Waits for sub-workflow to finish to continue processing.  
    - Input: Receives initialized stacks and logs from `Create stacks`.  
    - Output: Passes results to `Max 1 disc`.  
    - Edge cases: Recursive calls can fail due to stack overflow if max recursion depth exceeded.

  - **Max 1 disc**  
    - Type: If  
    - Role: Checks if `numberOfDiscs` ≤ 1 (base case).  
    - Configuration: Condition `numberOfDiscs <= 1`.  
    - Input: Receives current state from `A B C`.  
    - Output:  
      - True branch: `X to Z` (move disc directly).  
      - False branch: `X Z Y` (recursive call with swapped stacks).  
    - Edge cases: Expression evaluation errors if input is missing or malformed.

  - **X to Z**  
    - Type: Code  
    - Role: Moves one disc from stackX to stackZ and logs the move.  
    - Logic:  
      - Pops disc from `stackX.items`, pushes to `stackZ.items`.  
      - Appends a string `"stackX.name stackZ.name disc"` to logs.  
    - Input: Receives stacks and logs from `Max 1 disc` true branch.  
    - Output: Passes updated stacks/logs to `Y X Z`.  
    - Edge cases: Popping from empty stack will cause errors; ensure input stacks are valid.

  - **X Z Y**  
    - Type: Execute Workflow (Sub-workflow call)  
    - Role: Recursive call with `numberOfDiscs - 1` and stacks reordered to (stackX, stackZ, stackY).  
    - Input: Receives updated stacks and logs from `Max 1 disc` false branch.  
    - Output: Passes results to `Update` node.  
    - Edge cases: Recursive calls may cause stack depth issues.

  - **Y X Z**  
    - Type: Execute Workflow (Sub-workflow call)  
    - Role: Recursive call with `numberOfDiscs - 1` and stacks reordered to (stackY, stackX, stackZ).  
    - Input: Receives updated stacks and logs from `X to Z`.  
    - Output: Passes results to `Update` node.  
    - Edge cases: Same as other recursion calls.

  - **Update** (two instances: one after `X Z Y`, one after `Y X Z`)  
    - Type: Code  
    - Role: Corrects stack assignments and increments `numberOfDiscs` to restore state after recursive calls.  
    - Logic differences:  
      - One swaps stacks Y and Z to reset ordering after `X Z Y`.  
      - The other swaps stacks X and Y after `Y X Z`.  
    - Input: Receives results from respective sub-workflow calls.  
    - Output: Passes updated data back to `X to Z` or `Solution`.  
    - Edge cases: Errors if input data structure is unexpected.

---

#### 2.3 Disc Movement and Stack Updates

- **Overview:**  
  Handles the actual movement of discs between stacks and keeps a log of moves.

- **Nodes Involved:**  
  - X to Z (Code)  
  -  X to Z (Code) [duplicate node name but different instance]  
  - Update (Code)  
  -  Update (Code)  

- **Node Details:**

  - **X to Z** (first instance)  
    - See section 2.2 for details.

  - ** X to Z** (second instance)  
    - Type: Code  
    - Role: Same as the first `X to Z`, moves disc from stackX to stackZ and logs the move.  
    - Positioned differently in the flow but functionally equivalent.  
    - Edge cases: Same as above.

  - **Update** and ** Update**  
    - See section 2.2 for details.

---

#### 2.4 Result Formatting

- **Overview:**  
  Transforms the collected logs of disc movements into a readable, formatted text output describing each move step-by-step.

- **Nodes Involved:**  
  - Solution (Code)  

- **Node Details:**

  - **Solution**  
    - Type: Code  
    - Role: Converts the array of logged moves into a formatted multiline string describing each move in human-readable form.  
    - Logic:  
      - Splits each log entry into from, to, and disc.  
      - Formats first move with "Move disc X from A to B,".  
      - Subsequent moves formatted with shorthand like "then A 2→ C," etc.  
      - Inserts line breaks and commas for readability.  
    - Input: Receives logs from recursive sub-workflow results (`A B C`).  
    - Output: Provides final `solution` string output.  
    - Edge cases: Empty logs or malformed entries may cause formatting issues.

---

#### 2.5 Supporting Sticky Notes (Documentation)

- **Overview:**  
  Sticky notes provide contextual explanations, input/output expectations, and algorithm details to assist users understanding the workflow.

- **Nodes Involved:**  
  - user input  
  - setup  
  - result  
  - solution  
  - swap  
  - move  
  - update  
  - Towers of Hanoi (main description note)  

- **Node Details:**  
  - These nodes are purely informational with no execution role.  
  - Contain sample inputs, outputs, algorithm description, and step explanations.  
  - Provide links to further resources such as GitHub and Wikipedia on recursion.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                       | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                                |
|---------------------|-------------------------|------------------------------------|-----------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Start               | Manual Trigger          | Starts the workflow manually        | -                     | Set number of discs      |                                                                                                                                            |
| Set number of discs  | Set                     | Sets initial numberOfDiscs (4)      | Start                 | Create stacks            |                                                                                                                                            |
| Create stacks       | Code                    | Validates disc count and initializes stacks A, B, C with discs on A | Set number of discs      | A B C                    |                                                                                                                                            |
| A B C               | Execute Workflow        | Recursive call to main workflow with stacks and logs | Create stacks          | Max 1 disc               |                                                                                                                                            |
| Max 1 disc          | If                      | Checks base case (n ≤ 1)             | A B C                  | X to Z (true), X Z Y (false) |                                                                                                                                            |
| X to Z              | Code                    | Moves one disc from stackX to stackZ and logs move | Max 1 disc (true)       | Y X Z                    |                                                                                                                                            |
| Y X Z               | Execute Workflow        | Recursive call with stacks reordered after move | X to Z                  | Update                   |                                                                                                                                            |
| Update              | Code                    | Restores stacks & increments numberOfDiscs after Y X Z | Y X Z                  | X to Z                   | "### Node steps\n- **numberOfDiscs** += 1\n- YXZ to XYZ\n  - stack = **stackY**\n  - **stackY** = **stackX**\n  - **stackX** = stack"          |
| X Z Y               | Execute Workflow        | Recursive call with stacks reordered before move | Max 1 disc (false)      | Update                   |                                                                                                                                            |
| Update              | Code                    | Restores stacks & increments numberOfDiscs after X Z Y | X Z Y                  | X to Z                   | "### Node steps\n- **numberOfDiscs** += 1\n- Reset stacks (XZY to XYZ)\n  - stack = **stackY**\n  - **stackY** = **stackZ**\n  - **stackZ** = stack" |
|  X to Z             | Code                    | Moves disc from stackX to stackZ and logs move | Update                 | Y X Z                    | "### Node steps\n- disc = **stackX**.pop()\n- **stackZ**.push(disc)"                                                                       |
| Solution            | Code                    | Formats logs into human-readable solution string | A B C                   | -                       |                                                                                                                                            |
| user input          | Sticky Note             | Describes input: numberOfDiscs = 4  | -                     | -                       | "### Input\n- **numberOfDiscs:** 4 "                                                                                                       |
| setup               | Sticky Note             | Describes initial stacks and discs  | -                     | -                       | "### Setup\n- **numberOfDiscs:** 4  \n- **stackA**\n  - **name:** A  \n  - **items:** 4, 3, 2, 1  \n- **stackB**\n  - **name:** B  \n  - **items:** -  \n- **stackC**\n  - **name:** C  \n  - **items:** -  \n" |
| result              | Sticky Note             | Shows expected final stacks after solving | -                     | -                       | "### Result\n- **numberOfDiscs:** 4  \n- **stackX**\n  - **name:** A  \n  - **items:** -  \n- **stackY**\n  - **name:** B  \n  - **items:** -  \n- **stackZ**\n  - **name:** C  \n  - **items:** 4, 3, 2, 1" |
| solution            | Sticky Note             | Example formatted solution output   | -                     | -                       | "### Solution\nMove disc 1 from A to B,\nthen A 2→ C,\nB 1→ C, A 3→ B,\nC 1→ A, C 2→ B,\nA 1→ B, A 4→ C,\nB 1→ C, B 2→ A,\nC 1→ A, B 3→ C,\nA 1→ B, A 2→ C,\nB 1→ C." |
| swap                | Sticky Note             | Explains input stack swaps in recursion | -                     | -                       | "### Node input\n- **numberOfDiscs** -1 \n- stackX = **stackX** \n- stackY = **stackZ** \n- stackZ = **stackY** "                              |
| move                | Sticky Note             | Explains disc move operations        | -                     | -                       | "### Node steps\n- disc = **stackX**.pop()\n- **stackZ**.push(disc)"                                                                        |
| update              | Sticky Note             | Explains stack resetting steps       | -                     | -                       | "### Node steps\n- **numberOfDiscs** += 1\n- YXZ to XYZ\n  - stack = **stackY**\n  - **stackY** = **stackX**\n  - **stackX** = stack"          |
| Towers of Hanoi     | Sticky Note             | Full algorithm description, notes, and resource links | -                     | -                       | See section 5 for full content and links.                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node ("Start")**  
   - Type: Manual Trigger  
   - No configuration needed.

2. **Create Set Node ("Set number of discs")**  
   - Set parameter: `numberOfDiscs` = 4 (Number type)  
   - Connect "Start" → "Set number of discs".

3. **Create Code Node ("Create stacks")**  
   - JavaScript:  
     - Enforce min 1 and max 5 discs.  
     - Initialize `stackA` with discs [numberOfDiscs ... 1].  
     - Initialize empty `stackB` and `stackC`.  
     - Initialize empty `logs` array.  
   - Connect "Set number of discs" → "Create stacks".

4. **Create Execute Workflow Node ("A B C")**  
   - Set to call same workflow (recursive call).  
   - Pass inputs:  
     - `numberOfDiscs`  
     - `stackX` = `stackA`  
     - `stackY` = `stackB`  
     - `stackZ` = `stackC`  
     - `logs`  
   - Enable "Wait for sub-workflow" to true.  
   - Connect "Create stacks" → "A B C".

5. **Create If Node ("Max 1 disc")**  
   - Condition: `numberOfDiscs <= 1` (number comparison)  
   - Connect "A B C" → "Max 1 disc".

6. **Create Code Node ("X to Z")**  
   - JavaScript:  
     ```js
     const disc = $json.stackX.items.pop();
     $json.stackZ.items.push(disc);
     $json.logs.push(`${$json.stackX.name} ${$json.stackZ.name} ${disc}`);
     return $input.item;
     ```  
   - Connect "Max 1 disc" True branch → "X to Z".

7. **Create Execute Workflow Node ("Y X Z")**  
   - Set to call same workflow recursively.  
   - Pass inputs:  
     - `numberOfDiscs` = current `numberOfDiscs - 1`  
     - `stackX` = previous `stackY`  
     - `stackY` = previous `stackX`  
     - `stackZ` = previous `stackZ`  
     - `logs`  
   - Wait for sub-workflow result.  
   - Connect "X to Z" → "Y X Z".

8. **Create Code Node ("Update")**  
   - JavaScript:  
     ```js
     $json.numberOfDiscs += 1;
     const stack = $json.stackY;
     $json.stackY = $json.stackX;
     $json.stackX = stack;
     return $input.item;
     ```  
   - Connect "Y X Z" → "Update".

9. **Create Code Node (" X to Z")** (note the leading space in name)  
   - Same code as first "X to Z" node for moving disc and logging.  
   - Connect "Update" → " X to Z".

10. **Create Execute Workflow Node ("X Z Y")**  
    - Set to call same workflow recursively.  
    - Pass inputs:  
      - `numberOfDiscs` = current `numberOfDiscs - 1`  
      - `stackX` = previous `stackX`  
      - `stackY` = previous `stackZ`  
      - `stackZ` = previous `stackY`  
      - `logs`  
    - Wait for sub-workflow result.  
    - Connect "Max 1 disc" False branch → "X Z Y".

11. **Create Code Node ("Update")** (second instance)  
    - JavaScript:  
      ```js
      $json.numberOfDiscs += 1;
      const stack = $json.stackY;
      $json.stackY = $json.stackZ;
      $json.stackZ = stack;
      return $input.item;
      ```  
    - Connect "X Z Y" → this "Update".

12. **Connect "Update" → " X to Z"** (the space-prefixed node).

13. **Create Code Node ("Solution")**  
    - JavaScript:  
      ```js
      function logsToSentence(logs) {
        return logs.map((entry, index) => {
          const [from, to, disc] = entry.split(" ");
          let text;
          if (index === 0) {
            text = `Move disc ${disc} from ${from} to ${to}`;
          } else if (index === 1) {
            text = `then ${from} ${disc}→ ${to}`;
          } else {
            text = `${from} ${disc}→ ${to}`;
          }
          text += ",";
          if (index === 0 || index % 2 === 1) {
            text += "\n";
          } else {
            text += " ";
          }
          return text;
        }).join("").slice(0, -2) + ".";
      }
      return { solution: logsToSentence($input.first().json.logs) };
      ```  
    - Connect "A B C" → "Solution".

14. **Add Sticky Notes** for documentation as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ## Towers of Hanoi<br>This workflow demonstrates recursion in n8n using sub-workflows to solve the Towers of Hanoi puzzle.<br>### Algorithm<br>```procedure Hanoi(n, X, Y, Z):<br>  if n == 1:<br>    move disk from X to Z<br>  else:<br>    Hanoi(n-1, X, Z, Y)<br>    move disk from X to Z<br>    Hanoi(n-1, Y, X, Z)<br>```<br>Notes:<br>- Variables X, Y, Z represent rods; swapping them adjusts positions.<br>- Recursive calls return updated stacks, requiring stack resetting nodes.<br>- Termination condition is important to avoid infinite recursion.<br>- Links:<br>  - [n8n Dashboard Restart workspace](https://app.n8n.cloud/manage)<br>  - [GitHub Workflow repo](https://github.com/adibaba/n8n.workflows)<br>  - [Recursion Wikipedia (Towers of Hanoi)](https://en.wikipedia.org/w/index.php?title=Recursion_(computer_science)&oldid=1301600240#Towers_of_Hanoi) | Main workflow sticky note with full description and resources.                                                                                                                                                                  |

---

**Disclaimer:** The provided text is solely derived from an automated workflow created using n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.