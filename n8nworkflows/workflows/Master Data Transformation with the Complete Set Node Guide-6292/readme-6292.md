Master Data Transformation with the Complete Set Node Guide

https://n8nworkflows.xyz/workflows/master-data-transformation-with-the-complete-set-node-guide-6292


# Master Data Transformation with the Complete Set Node Guide

---

### 1. Workflow Overview

This workflow, titled **Master Data Transformation with the Complete Set Node Guide**, serves as a comprehensive tutorial for mastering the **Set** node in n8n. It is designed for users who want to learn how to manipulate data within workflows by setting values, using expressions, handling complex data structures, cleaning outputs, and applying conditional logic. The workflow is structured into logical blocks that progressively demonstrate core features of the Set node, culminating in conditional branching and final data merging.

**Target Use Cases:**  
- Learning and mastering the Set node functionality  
- Data transformation and preparation  
- Conditional data manipulation within workflows  
- Producing clean, tailored outputs for downstream processes

**Logical Blocks:**  
- **1.1 Input Reception:** Manual or test data input triggers  
- **1.2 Basic Data Setting:** Setting simple data types (strings, numbers, booleans)  
- **1.3 Expression Usage:** Using expressions and referencing previous data dynamically  
- **1.4 Complex Data Handling:** Creating nested objects and arrays  
- **1.5 Data Cleaning:** Using “Keep Only Set” to trim output data  
- **1.6 Conditional Logic:** Branching with IF node and conditionally setting data  
- **1.7 Output Merging and Summary:** Combining branches and summarizing key concepts

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  The workflow starts with either a manual trigger or a test data set to initiate the tutorial, providing flexibility to run with static or custom input.

- **Nodes Involved:**  
  - Start (Manual Trigger)  
  - 0. Test Data Input  
  - When clicking ‘Execute workflow’ (Manual Trigger alternative)  
  - Tutorial Intro (Sticky Note)

- **Node Details:**  
  - **Start (Manual Trigger)**  
    - Type: Manual trigger node  
    - Role: Starts the workflow manually, passing a basic welcome message object with a timestamp  
    - Connections: Output to “1. Set Basic Values”  
    - Edge cases: None, simple manual start  
  - **0. Test Data Input**  
    - Type: Set node  
    - Role: Provides sample user data including age, score, preferences, and product interest  
    - Parameters: Static JSON data with fields like `user_age`, `score`, `is_active` etc.  
    - Connections: Alternative start node, can be connected to “1. Set Basic Values” instead of the manual trigger  
    - Edge cases: None, but user can modify data to test different branches  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual trigger  
    - Role: Alternative manual trigger to activate the test data input node  
    - Connections: Output to “0. Test Data Input”  
  - **Tutorial Intro**  
    - Type: Sticky Note  
    - Content: Overview of the tutorial workflow goals and instructions  

---

#### 2.2 Basic Data Setting

- **Overview:**  
  This block introduces setting basic data types such as strings, numbers, and booleans using the Set node.

- **Nodes Involved:**  
  - 1. Set Basic Values  
  - Basic Set Note (Sticky Note)

- **Node Details:**  
  - **1. Set Basic Values**  
    - Type: Set node  
    - Role: Sets example fields like `user_name` (string), `user_age` (number), `is_active` (boolean), etc.  
    - Configuration: Static values assigned directly to fields  
    - Connections: Input from Start node, output to “2. Set with Expressions” and “Age Check” nodes  
    - Edge cases: None expected, but user can change values to see effects downstream  
  - **Basic Set Note**  
    - Sticky note explaining data types and encouraging user experimentation  

---

#### 2.3 Expression Usage

- **Overview:**  
  Demonstrates how to use expressions within the Set node to dynamically calculate and interpolate values.

- **Nodes Involved:**  
  - 2. Set with Expressions  
  - Expression Note (Sticky Note)

- **Node Details:**  
  - **2. Set with Expressions**  
    - Type: Set node  
    - Role: Uses expressions like `{{ $json.user_name }}`, `{{ $now.toFormat('yyyy-MM-dd') }}`, and calculations like `{{ $json.score * 2 }}`  
    - Configuration: Fields set using JavaScript expressions wrapped in `{{ }}`  
    - Connections: Input from “1. Set Basic Values”, output to “3. Set Complex Data”  
    - Edge cases: Expression syntax errors or referencing missing fields could cause failures  
  - **Expression Note**  
    - Sticky note explaining expression syntax and examples  

---

#### 2.4 Complex Data Handling

- **Overview:**  
  Focuses on setting complex nested objects and arrays combining static and dynamic data.

- **Nodes Involved:**  
  - 3. Set Complex Data  
  - Complex Data Note (Sticky Note)  
  - 4. Set Clean Output  
  - Keep Only Note (Sticky Note)

- **Node Details:**  
  - **3. Set Complex Data**  
    - Type: Set node  
    - Role: Builds hierarchical JSON objects like `user_profile` with nested `personal` and `account` properties, and arrays like `user_permissions`  
    - Configuration: Mix of static values and expressions used to populate nested structures  
    - Connections: Input from “2. Set with Expressions”, output to “4. Set Clean Output”  
    - Edge cases: Complex expressions may fail if referencing undefined data  
  - **Complex Data Note**  
    - Sticky note explaining nested objects and arrays with code examples  
  - **4. Set Clean Output**  
    - Type: Set node  
    - Role: Uses “Keep Only Set” option enabled to output only newly set fields, removing all previous data  
    - Configuration: Option “Keep Only Set” enabled, fields reference nested object properties  
    - Connections: Input from “3. Set Complex Data”  
    - Output to “Age Check” node (implicitly by connections)  
    - Edge cases: Removing all other data may cause downstream nodes expecting previous fields to fail  
  - **Keep Only Note**  
    - Sticky note explaining the “Keep Only Set” option and its use cases  

---

#### 2.5 Conditional Logic

- **Overview:**  
  Demonstrates branching using an IF node based on user age and setting data conditionally in each branch.

- **Nodes Involved:**  
  - Age Check (IF node)  
  - 5a. Set Adult Data  
  - 5b. Set Young Adult Data  
  - Conditional Note (Sticky Note)  
  - Merge Branches

- **Node Details:**  
  - **Age Check**  
    - Type: IF node  
    - Role: Branches workflow based on condition `user_age > 25`  
    - Condition: Numeric greater-than comparison on `user_age` from input JSON  
    - Connections: TRUE outputs to “5a. Set Adult Data”, FALSE outputs to “5b. Set Young Adult Data”  
    - Edge cases: Missing or non-numeric `user_age` may cause condition failure  
  - **5a. Set Adult Data**  
    - Type: Set node  
    - Role: Sets data fields specific for adults, such as premium offers  
    - Connections: Input from TRUE branch of “Age Check”, output to “Merge Branches”  
  - **5b. Set Young Adult Data**  
    - Type: Set node  
    - Role: Sets data fields specific for young adults, like student discounts  
    - Connections: Input from FALSE branch of “Age Check”, output to “Merge Branches”  
  - **Conditional Note**  
    - Sticky note explaining the branching logic and encouraging testing different input ages  
  - **Merge Branches**  
    - Type: Merge node  
    - Role: Combines TRUE and FALSE branches back into a single output stream  
    - Configuration: Combine mode, no specific merge fields  
    - Connections: Inputs from both “5a” and “5b” Set nodes, output to “6. Tutorial Summary”  
    - Edge cases: Mismatched data schemas between branches might cause downstream issues  

---

#### 2.6 Output Summary

- **Overview:**  
  Final Set node summarizes all key concepts covered in the tutorial and prepares the final output data.

- **Nodes Involved:**  
  - 6. Tutorial Summary  
  - Completion Note (Sticky Note)

- **Node Details:**  
  - **6. Tutorial Summary**  
    - Type: Set node  
    - Role: Outputs a summary object or fields that recapitulate the tutorial’s lessons  
    - Connections: Input from “Merge Branches” node  
  - **Completion Note**  
    - Sticky note congratulating the user for completing the tutorial and suggesting next steps  

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                          | Input Node(s)                 | Output Node(s)                      | Sticky Note                                                                                       |
|-------------------------|-----------------------|----------------------------------------|------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Start (Manual Trigger)   | Manual Trigger        | Workflow manual start trigger           |                              | 1. Set Basic Values               | This node serves as your manual trigger. Click 'Execute Workflow' to start the tutorial.         |
| 0. Test Data Input       | Set                   | Provides sample input data for testing  | When clicking ‘Execute workflow’ | 1. Set Basic Values               | 🔧 **ALTERNATIVE START NODE** - Use this instead of 'Start (Manual Trigger)' for testing!         |
| When clicking ‘Execute workflow’ | Manual Trigger    | Alternative manual trigger for test data|                              | 0. Test Data Input                |                                                                                                 |
| Tutorial Intro           | Sticky Note           | Workflow overview and instructions      |                              |                                   | ## 🎯 Set Node Tutorial Workflow\n\nWelcome to the comprehensive Set node tutorial!              |
| 1. Set Basic Values      | Set                   | Sets basic string, number, boolean data | Start (Manual Trigger), 0. Test Data Input | 2. Set with Expressions, Age Check | This demonstrates setting basic data types: strings, numbers, booleans                            |
| Basic Set Note           | Sticky Note           | Explains basic Set operations            |                              |                                   | ## 📝 Basic Set Operations\nSets simple string, number, boolean values                           |
| 2. Set with Expressions  | Set                   | Demonstrates use of expressions          | 1. Set Basic Values           | 3. Set Complex Data               | Shows usage of expressions like {{ $json.field }} and calculations                               |
| Expression Note          | Sticky Note           | Explains expressions syntax and usage    |                              |                                   | ## 🔧 Expression Magic\nUsing {{ }} to reference previous data and functions                     |
| 3. Set Complex Data      | Set                   | Sets nested objects and arrays           | 2. Set with Expressions       | 4. Set Clean Output               | Demonstrates complex data structures with nested objects and arrays                              |
| Complex Data Note        | Sticky Note           | Explains complex data structures          |                              |                                   | ## 🏗️ Complex Data Structures\nObjects and arrays with mixed data                               |
| 4. Set Clean Output      | Set                   | Uses “Keep Only Set” to output clean data| 3. Set Complex Data           | Age Check                        | Shows the 'Keep Only Set' option which cleans output data                                       |
| Keep Only Note           | Sticky Note           | Explains “Keep Only Set” option           |                              |                                   | ## 🧹 Keep Only Set Option\nOnly new values kept, previous data removed                         |
| Age Check               | IF                    | Branches based on user_age > 25           | 1. Set Basic Values, 4. Set Clean Output | 5a. Set Adult Data (TRUE), 5b. Set Young Adult Data (FALSE) | Branching logic to demonstrate conditional Set nodes                                            |
| 5a. Set Adult Data       | Set                   | Sets data for adults (age > 25)           | Age Check (TRUE)              | Merge Branches                   | Conditional Set node for adults                                                                |
| 5b. Set Young Adult Data | Set                   | Sets data for young adults (age ≤ 25)     | Age Check (FALSE)             | Merge Branches                   | Conditional Set node for young adults                                                          |
| Conditional Note         | Sticky Note           | Explains conditional branching             |                              |                                   | ## 🔀 Conditional Set Nodes\nDifferent logic based on age condition                            |
| Merge Branches           | Merge                 | Combines conditional branches             | 5a. Set Adult Data, 5b. Set Young Adult Data | 6. Tutorial Summary              | Merges TRUE and FALSE branches back together                                                  |
| 6. Tutorial Summary      | Set                   | Final summary of tutorial concepts         | Merge Branches               |                                   | Final summary of the Set node tutorial                                                        |
| Completion Note          | Sticky Note           | Congratulates user and suggests next steps |                              |                                   | ## 🎓 Tutorial Complete!\nSummary and encouragement for further experiments                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Start Node:**  
   - Add a **Manual Trigger** node named `Start (Manual Trigger)`. No parameters required.  
   - This node will start the workflow manually.

2. **Create Test Data Input (Optional):**  
   - Add a **Set** node named `0. Test Data Input`.  
   - Configure static JSON data with fields:  
     ```
     score: 75  
     user_age: 20  
     is_active: false  
     user_name: "Bob"  
     user_email: "bob@example.com"  
     order_value: 150.75  
     preferences: { theme: "dark", notifications: { sms: false, email: true } }  
     items_purchased: ["item_A", "item_C", "item_F"]  
     product_interest: "Automation Software"  
     newsletter_subscribed: true  
     ```  
   - This node serves as an alternate start for testing.

3. **Create Tutorial Intro Sticky Note:**  
   - Add a **Sticky Note** node named `Tutorial Intro`.  
   - Add content explaining the tutorial goals and instructions on executing nodes step by step.

4. **Create 1. Set Basic Values:**  
   - Add a **Set** node named `1. Set Basic Values`.  
   - Configure to set:  
     - `user_name` as a string (e.g., "Alice")  
     - `user_age` as a number (e.g., 30)  
     - `is_active` as boolean (true/false)  
     - Other simple fields as examples.

5. **Create Basic Set Note:**  
   - Add a Sticky Note explaining the setting of basic data types.

6. **Connect the Start Nodes:**  
   - Connect `Start (Manual Trigger)` output to `1. Set Basic Values`.  
   - Optionally, connect `When clicking ‘Execute workflow’` manual trigger node to `0. Test Data Input`, and then connect `0. Test Data Input` to `1. Set Basic Values`.

7. **Create 2. Set with Expressions:**  
   - Add a **Set** node named `2. Set with Expressions`.  
   - Use expressions like:  
     - `user_name_upper` = `{{ $json.user_name.toUpperCase() }}`  
     - `current_date` = `{{ $now.toFormat('yyyy-MM-dd') }}`  
     - `score_doubled` = `{{ $json.score * 2 }}`  
   - Connect input from `1. Set Basic Values`.

8. **Create Expression Note Sticky Note:**  
   - Explain usage of expressions and syntax.

9. **Create 3. Set Complex Data:**  
   - Add a **Set** node named `3. Set Complex Data`.  
   - Configure nested objects like:  
     ```
     user_profile: {  
       personal: { name: {{ $json.user_name }}, age: {{ $json.user_age }}, email: "user@example.com" },  
       account: { is_active: {{ $json.is_active }}, score: {{ $json.score }} }  
     }  
     user_permissions: ["read", "write", "premium"]  
     score_history: [85.5, 171, 128.25]  
     ```  
   - Connect input from `2. Set with Expressions`.

10. **Create Complex Data Note Sticky Note:**  
    - Add explanations about nested objects and arrays.

11. **Create 4. Set Clean Output:**  
    - Add a **Set** node named `4. Set Clean Output`.  
    - Enable the **Keep Only Set** option.  
    - Set only a few selected fields, referencing nested data, e.g., `user_profile.personal.name`.  
    - Connect input from `3. Set Complex Data`.

12. **Create Keep Only Note Sticky Note:**  
    - Explain the “Keep Only Set” option and use cases.

13. **Create Age Check IF Node:**  
    - Add an **IF** node named `Age Check`.  
    - Set condition:  
      - Type: Number  
      - Operation: Greater than (>)  
      - Value: `25`  
      - Field: `user_age` from incoming JSON (`={{ $json.user_age }}`)  
    - Connect input from `1. Set Basic Values` and `4. Set Clean Output` (depending on your intended data flow; the provided workflow shows both feeding into Age Check).

14. **Create 5a. Set Adult Data:**  
    - Add a **Set** node named `5a. Set Adult Data`.  
    - Configure fields relevant to adults, e.g., `offer` = "Premium Membership".  
    - Connect input from TRUE branch of `Age Check`.

15. **Create 5b. Set Young Adult Data:**  
    - Add a **Set** node named `5b. Set Young Adult Data`.  
    - Configure fields relevant to young adults, e.g., `offer` = "Student Discount".  
    - Connect input from FALSE branch of `Age Check`.

16. **Create Conditional Note Sticky Note:**  
    - Explain the IF node splitting the workflow and different logic per branch.

17. **Create Merge Branch Node:**  
    - Add a **Merge** node named `Merge Branches`.  
    - Mode: Combine  
    - Connect inputs from `5a. Set Adult Data` and `5b. Set Young Adult Data`.  
    - Output connects to next node.

18. **Create 6. Tutorial Summary:**  
    - Add a **Set** node named `6. Tutorial Summary`.  
    - Summarize key tutorial concepts in fields or a message.

19. **Create Completion Note Sticky Note:**  
    - Add a sticky note congratulating the user and suggesting next steps.

20. **Connect Merge Branches output to 6. Tutorial Summary.**

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The Set node is versatile and supports strings, numbers, booleans, complex objects, arrays, and expressions. | Core concept throughout this tutorial workflow. |
| Expressions use `{{ }}` syntax to reference previous node data (`$json`) and built-in functions like `$now`.  | Expressions block and notes.                      |
| “Keep Only Set” option is crucial for producing clean outputs by removing previous data fields automatically. | Data cleaning block explanation.                  |
| Conditional logic with IF nodes allows branching workflows based on dynamic data conditions.                   | Conditional block explanation.                    |
| Workflow designed to be run step-by-step with manual execution to observe intermediate outputs.                | Tutorial Intro and Completion notes.              |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected elements. All data handled is legal and public.

---