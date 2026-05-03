🧑‍🎓 Master Data Access Techniques with Progressive Expression Challenges

https://n8nworkflows.xyz/workflows/------master-data-access-techniques-with-progressive-expression-challenges-6236


# 🧑‍🎓 Master Data Access Techniques with Progressive Expression Challenges

### 1. Workflow Overview

This workflow is an interactive, stepwise practical test designed to teach and validate mastery of n8n expression syntax for accessing and manipulating JSON data structures. It targets users aiming to improve their skills in extracting and transforming data from nested JSON objects and arrays within n8n workflows.

The workflow is logically divided into the following blocks, each representing a distinct expression challenge:

- **1.1 Input Initialization:** Sets up the source JSON data used for testing.
- **1.2 Progressive Expression Challenges:** Series of test nodes where users write expressions to extract or convert specific data from the source.
- **1.3 Validation and Feedback:** Each test node is followed by conditional checks that validate the user’s expression, routing to success or error nodes with feedback.
- **1.4 Answer Keys:** Nodes containing correct expressions for reference.
- **1.5 Final Success:** Displays a final success message upon passing all tests.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** This block initializes the source data, a JSON object containing user details with nested structures (arrays, objects). It serves as the data source for all expression tests.
- **Nodes Involved:**  
  - *Start Test!* (Manual Trigger)  
  - *Source Data* (Set)  
  - *Sticky Note13* (Instructional Note)  
  - *Instruction - Basic Access1* (Visual reference sticky note with GIF)

- **Node Details:**

  - **Start Test!**  
    - *Type:* Manual Trigger  
    - *Role:* Initiates the workflow run manually.  
    - *Connections:* Outputs to *Source Data*.  
    - *Edge Cases:* None significant; user must trigger manually.

  - **Source Data**  
    - *Type:* Set  
    - *Role:* Defines a fixed JSON object with fields such as `name`, `city`, `level`, `tools` (array), `address` (nested object), and `tasks` (array of objects).  
    - *Configuration:* Assigns static values representing a fictional user "Clark Kent" with related data.  
    - *Output:* Single JSON item with all data fields for downstream use.  
    - *Edge Cases:* None; static data.

  - **Sticky Note13** and **Instruction - Basic Access1**  
    - *Type:* Sticky Note  
    - *Role:* Provide context, instructions, and visual aids to the user. No direct impact on workflow logic.

---

#### 2.2 Progressive Expression Challenges

This core block contains sequential steps where the user writes expressions to access or transform data, followed by automatic validation.

Each step follows a pattern:
- A *Test* node of type *Set* where the user inputs an expression.
- A *Check* node of type *If* that compares the expression output against expected correct values.
- A *Success* node (NoOp) indicating correct answers.
- An *Error* node (StopAndError) providing hints if the expression is incorrect.
- Answer nodes containing the correct solution expressions.
- Instruction sticky notes guiding the user.

The steps are:

##### 2.2.1 Step 1: Basic Access

- **Purpose:** Access a simple string field `city`.
- **Nodes:**  
  - Test - Basic Access (Set)  
  - Check - Basic Access (If)  
  - Success - Basic Access (NoOp)  
  - Error - Basic Access (StopAndError)  
  - Answer - Basic Access (Set)  
  - Answer Note - Basic Access (Sticky Note)  
  - Instruction - Basic Access (Sticky Note)  
  - Feedback Correct1 / Feedback Incorrect1 (Sticky Notes)

- **Key Configuration:**  
  - User must create field `user_city` with expression like `{{ $('Source Data').item.json.city }}`.  
  - Check node verifies equality with the source data city field and checks if the expression contains `.city`.

- **Edge Cases:**  
  - Expression not referencing the correct node or field.  
  - Expression syntax errors.

##### 2.2.2 Step 2: Array Access

- **Purpose:** Access the third element of the `tools` array.
- **Nodes:**  
  - Test - Array Access  
  - Check - Array Access  
  - Success - Array Access  
  - Error - Array Access  
  - Answer - Array Access  
  - Answer Note - Array Access  
  - Instruction - Array Access  
  - Feedback Correct2 / Feedback Incorrect2

- **Key Configuration:**  
  - User creates `third_tool` with expression accessing `tools[2]` (zero-indexed).  
  - Check node compares the output with source data’s third tool.

- **Edge Cases:**  
  - Off-by-one errors on array indexing.  
  - Syntax errors or incorrect path.

##### 2.2.3 Step 3: Nested Object Access

- **Purpose:** Access nested object property `address.street`.
- **Nodes:**  
  - Test - Nested Object  
  - Check - Nested Object  
  - Success - Nested Object  
  - Error - Nested Object  
  - Answer - Nested Object  
  - Answer Note - Nested Object  
  - Instruction - Nested Object  
  - Feedback Correct3 / Feedback Incorrect3

- **Key Configuration:**  
  - Expression to extract `street_address` as `{{ $('Source Data').item.json.address.street }}`.  
  - Check node verifies correctness and expression usage.

- **Edge Cases:**  
  - Incorrect chaining of object keys.  
  - Missing or mistyped keys.

##### 2.2.4 Step 4: Array of Objects Access

- **Purpose:** Access the `name` property of the second object in `tasks` array.
- **Nodes:**  
  - Test - Array of Objects  
  - Check - Array of Objects  
  - Success - Array of Objects  
  - Error - Array of Objects  
  - Answer - Array of Objects  
  - Answer Note - Array of Objects  
  - Instruction - Array of Objects  
  - Feedback Correct4 / Feedback Incorrect4

- **Key Configuration:**  
  - Expression for `second_task_name`: `{{ $('Source Data').item.json.tasks[1].name }}`.  
  - Check node confirms value and presence of `.tasks`.

- **Edge Cases:**  
  - Array indexing mistakes.  
  - Incorrect property access.

##### 2.2.5 Step 5: JS Function (String Transformation)

- **Purpose:** Transform the `name` field to uppercase.
- **Nodes:**  
  - Test - JS Function  
  - Check - JS Function  
  - Success - JS Function  
  - Error - JS Function  
  - Answer - JS Function  
  - Answer Note - JS Function  
  - Instruction - JS Function  
  - Feedback Correct5 / Feedback Incorrect5

- **Key Configuration:**  
  - Expression: `{{ $('Source Data').item.json.name.toUpperCase() }}`.  
  - Check confirms output matches uppercase conversion.

- **Edge Cases:**  
  - Missing `.toUpperCase()` method call.  
  - Syntax errors.

##### 2.2.6 Step 6: Final Challenge (String Interpolation)

- **Purpose:** Combine static text with expressions to create a summary string describing a task status.
- **Nodes:**  
  - Test - Final  
  - Check - Final  
  - Success - Final  
  - Error - Final  
  - Answer - Final  
  - Answer Note - Final  
  - Instruction - Final  
  - Feedback Correct6 / Feedback Incorrect6

- **Key Configuration:**  
  - Expression creating `summary`:  
    `The status of task '{{ $('Source Data').item.json.tasks[1].name }}' is {{ $('Source Data').item.json.tasks[1].status }}.`  
  - Check node verifies exact string match and presence of `.name` and `.status`.

- **Edge Cases:**  
  - Incorrect string concatenation or missing quotes.  
  - Syntax errors in expression interpolation.

---

#### 2.3 Final Success Display

- **Overview:** Presents a congratulatory HTML page upon successful completion of all tests.
- **Nodes Involved:**  
  - 🎉 SUCCESS 🎉 (HTML)  
  - Congratulations! (Sticky Note)  
  - Sticky Note16 (Visual animation sticky note)  
  - Sticky Note10 (Feedback & external links sticky note)

- **Node Details:**

  - **🎉 SUCCESS 🎉**  
    - *Type:* HTML  
    - *Role:* Displays a styled success message with a green checkmark, congratulating the user on mastering n8n expressions.  
    - *Output:* Visual UI element; no data output downstream.  
    - *Edge Cases:* None.

  - **Congratulations!** and other sticky notes  
    - Provide motivational and feedback content, external links to more templates, coaching, and consulting offers by Lucas Peyrin.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                      |
|-------------------------|---------------------|----------------------------------------|------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| Start Test!             | Manual Trigger      | Workflow trigger                       |                              | Source Data                      | © 2025 Lucas Peyrin                                                                             |
| Source Data             | Set                 | Source JSON data setup                  | Start Test!                  | Test - Basic Access              | © 2025 Lucas Peyrin                                                                             |
| Sticky Note13           | Sticky Note         | Introductory instructions               |                              |                                  | © 2025 Lucas Peyrin - Explains test overview and instructions                                   |
| Instruction - Basic Access1 | Sticky Note         | Visual GIF for source data reference    |                              |                                  | © 2025 Lucas Peyrin - Visual aid with GIF image                                                |
| Instruction - Basic Access | Sticky Note         | Step 1 instructions                     |                              |                                  | © 2025 Lucas Peyrin - Task description for Step 1                                              |
| Test - Basic Access     | Set                 | User input of expression for city      | Source Data                  | Check - Basic Access             | © 2025 Lucas Peyrin                                                                             |
| Check - Basic Access    | If                  | Validates user expression for city     | Test - Basic Access          | Success - Basic Access, Error - Basic Access | © 2025 Lucas Peyrin                                                                |
| Success - Basic Access  | NoOp                 | Path for correct expression             | Check - Basic Access         | Test - Array Access              | © 2025 Lucas Peyrin                                                                             |
| Error - Basic Access    | StopAndError        | Stops workflow and shows error for city| Check - Basic Access         |                                  | © 2025 Lucas Peyrin - Hint: Use format `{{ $('Source Data').item.json.city }}`                   |
| Answer - Basic Access   | Set                 | Contains correct city expression        |                              |                                  | © 2025 Lucas Peyrin - Reference answer                                                         |
| Answer Note - Basic Access | Sticky Note         | Answer key for Step 1                   |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Correct1       | Sticky Note         | Positive feedback for Step 1            |                              |                                  | © 2025 Lucas Peyrin - "Correct! Great start."                                                  |
| Feedback Incorrect1     | Sticky Note         | Hint for incorrect Step 1               |                              |                                  | © 2025 Lucas Peyrin - Reminder about correct expression format                                 |
| Instruction - Array Access | Sticky Note         | Step 2 instructions                     |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Test - Array Access     | Set                 | User expression for third array item   | Success - Basic Access       | Check - Array Access             | © 2025 Lucas Peyrin                                                                             |
| Check - Array Access    | If                  | Validates expression for third array item| Test - Array Access        | Success - Array Access, Error - Array Access | © 2025 Lucas Peyrin                                                                  |
| Success - Array Access  | NoOp                 | Path for correct expression             | Check - Array Access         | Test - Nested Object             | © 2025 Lucas Peyrin                                                                             |
| Error - Array Access    | StopAndError        | Stops workflow and shows error for array| Check - Array Access         |                                  | © 2025 Lucas Peyrin - Hint about zero-indexed arrays                                           |
| Answer - Array Access   | Set                 | Correct expression for third array item|                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Answer Note - Array Access | Sticky Note         | Answer key for Step 2                   |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Correct2       | Sticky Note         | Positive feedback for Step 2            |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Incorrect2     | Sticky Note         | Hint for incorrect Step 2               |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Instruction - Nested Object | Sticky Note         | Step 3 instructions                     |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Test - Nested Object    | Set                 | User expression for nested object       | Success - Array Access       | Check - Nested Object            | © 2025 Lucas Peyrin                                                                             |
| Check - Nested Object   | If                  | Validates expression for nested object  | Test - Nested Object         | Success - Nested Object, Error - Nested Object | © 2025 Lucas Peyrin                                                                |
| Success - Nested Object | NoOp                 | Path for correct expression             | Check - Nested Object        | Test - Array of Objects          | © 2025 Lucas Peyrin                                                                             |
| Error - Nested Object   | StopAndError        | Stops workflow and shows error for nested object | Check - Nested Object |                                  | © 2025 Lucas Peyrin - Hint about chaining keys                                                |
| Answer - Nested Object  | Set                 | Correct expression for nested object    |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Answer Note - Nested Object | Sticky Note         | Answer key for Step 3                   |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Correct3       | Sticky Note         | Positive feedback for Step 3            |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Incorrect3     | Sticky Note         | Hint for incorrect Step 3               |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Instruction - Array of Objects | Sticky Note         | Step 4 instructions                     |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Test - Array of Objects | Set                 | User expression for second object in array | Success - Nested Object   | Check - Array of Objects         | © 2025 Lucas Peyrin                                                                             |
| Check - Array of Objects| If                  | Validates expression for second object in array | Test - Array of Objects | Success - Array of Objects, Error - Array of Objects | © 2025 Lucas Peyrin                                                      |
| Success - Array of Objects | NoOp                 | Path for correct expression             | Check - Array of Objects     | Test - JS Function               | © 2025 Lucas Peyrin                                                                             |
| Error - Array of Objects| StopAndError        | Stops workflow and shows error for array of objects | Check - Array of Objects |                                  | © 2025 Lucas Peyrin - Hint about combining array and object access                            |
| Answer - Array of Objects| Set                 | Correct expression for second object in array |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Answer Note - Array of Objects | Sticky Note         | Answer key for Step 4                   |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Correct4       | Sticky Note         | Positive feedback for Step 4            |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Incorrect4     | Sticky Note         | Hint for incorrect Step 4               |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Instruction - JS Function| Sticky Note         | Step 5 instructions                     |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Test - JS Function      | Set                 | User expression for uppercase name      | Success - Array of Objects   | Check - JS Function              | © 2025 Lucas Peyrin                                                                             |
| Check - JS Function     | If                  | Validates uppercase string expression   | Test - JS Function           | Success - JS Function, Error - JS Function | © 2025 Lucas Peyrin                                                                |
| Success - JS Function   | NoOp                 | Path for correct expression             | Check - JS Function          | Test - Final                    | © 2025 Lucas Peyrin                                                                             |
| Error - JS Function     | StopAndError        | Stops workflow and shows error for JS function | Check - JS Function          |                                  | © 2025 Lucas Peyrin - Hint to add `.toUpperCase()`                                           |
| Answer - JS Function    | Set                 | Correct uppercase expression             |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Answer Note - JS Function| Sticky Note         | Answer key for Step 5                   |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Correct5       | Sticky Note         | Positive feedback for Step 5            |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Incorrect5     | Sticky Note         | Hint for incorrect Step 5               |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Instruction - Final     | Sticky Note         | Final challenge instructions            |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Test - Final            | Set                 | User expression for string interpolation | Success - JS Function        | Check - Final                   | © 2025 Lucas Peyrin                                                                             |
| Check - Final           | If                  | Validates final summary string           | Test - Final                | Success - Final, Error - Final   | © 2025 Lucas Peyrin                                                                             |
| Success - Final         | NoOp                 | Path for final success                   | Check - Final               | 🎉 SUCCESS 🎉                   | © 2025 Lucas Peyrin                                                                             |
| Error - Final           | StopAndError        | Stops workflow and shows error for final | Check - Final               |                                  | © 2025 Lucas Peyrin - Hint about combining static text and expressions                        |
| Answer - Final          | Set                 | Correct final expression                 |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Answer Note - Final     | Sticky Note         | Answer key for final step                |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Correct6       | Sticky Note         | Final positive feedback                  |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| Feedback Incorrect6     | Sticky Note         | Hint for incorrect final expression     |                              |                                  | © 2025 Lucas Peyrin                                                                             |
| 🎉 SUCCESS 🎉           | HTML                 | Final success message UI                  | Success - Final             |                                  | "Well done! You're awesome." + Link to more templates                                         |
| Congratulations!        | Sticky Note         | Congratulatory message                   |                              |                                  | "You've passed the test!" message                                                             |
| Sticky Note16           | Sticky Note         | Animated GIF visual                       |                              |                                  | Visual celebratory animation linked                                                           |
| Sticky Note10           | Sticky Note         | Feedback request, coaching and consulting offers |                              |                                  | Links to feedback form, coaching, consulting, and n8n Academy                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** called **Start Test!** to manually initiate the workflow.

2. **Create a Set node named Source Data**:  
   - Add fields:  
     - `name` (string): "Clark Kent"  
     - `city` (string): "Metropolis"  
     - `level` (number): 99  
     - `tools` (array string): `["Typewriter","Glasses","n8n"]`  
     - `address` (object string): `{"street":"123 Main St","zip":"10001"}`  
     - `tasks` (array of objects string): `[{"name":"Write Article","status":"Done"},{"name":"Review PR","status":"Pending"},{"name":"Save the World","status":"Pending"}]`  
   - Connect **Start Test!** output to **Source Data** input.

3. **Step 1: Basic Access**  
   - Create a Set node **Test - Basic Access**:  
     - Add field `user_city` (string) with expression: `={{ $('Source Data').item.json.city }}`  
   - Connect **Source Data** to **Test - Basic Access**.

   - Create an If node **Check - Basic Access**:  
     - Condition 1: `$json.user_city` equals `{{ $('Source Data').item.json.city }}`  
     - Condition 2: Expression string in Test node includes `.city` (boolean check)  
   - Connect **Test - Basic Access** to **Check - Basic Access**.

   - Create NoOp node **Success - Basic Access** and StopAndError node **Error - Basic Access**:  
     - Error node message: `"Incorrect. Hint: Use the format {{ $('Source Data').item.json.city }} to get the city value."`  
   - Connect **Check - Basic Access** success output to **Success - Basic Access**, failure output to **Error - Basic Access**.

4. **Step 2: Array Access**  
   - Create Set node **Test - Array Access** with field `third_tool` and expression: `={{ $('Source Data').item.json.tools[2] }}`  
   - Connect **Success - Basic Access** to **Test - Array Access**.

   - Create If node **Check - Array Access**:  
     - Check `$json.third_tool` equals `{{ $('Source Data').item.json.tools[2] }}`  
     - Check expression includes `.tools`  
   - Connect **Test - Array Access** to **Check - Array Access**.

   - Create NoOp **Success - Array Access** and StopAndError **Error - Array Access**:  
     - Error message: `"Incorrect. Hint: Remember that arrays are zero-indexed. The third item is at index [2]."`  
   - Connect **Check - Array Access** accordingly.

5. **Step 3: Nested Object Access**  
   - Set node **Test - Nested Object** with field `street_address` expression: `={{ $('Source Data').item.json.address.street }}`  
   - Connect **Success - Array Access** to **Test - Nested Object**.

   - If node **Check - Nested Object**:  
     - Check `$json.street_address` equals `{{ $('Source Data').item.json.address.street }}`  
     - Check expression includes `.address.street`  
   - Connect **Test - Nested Object** to **Check - Nested Object**.

   - NoOp **Success - Nested Object** and StopAndError **Error - Nested Object**:  
     - Error message: `"Incorrect. Hint: Chain the keys using dots, like ...json.address.street."`  
   - Connect **Check - Nested Object** accordingly.

6. **Step 4: Array of Objects Access**  
   - Set node **Test - Array of Objects** with field `second_task_name` expression: `={{ $('Source Data').item.json.tasks[1].name }}`  
   - Connect **Success - Nested Object** to **Test - Array of Objects**.

   - If node **Check - Array of Objects**:  
     - Check `$json.second_task_name` equals `{{ $('Source Data').item.json.tasks[1].name }}`  
     - Check expression includes `.tasks`  
   - Connect **Test - Array of Objects** to **Check - Array of Objects**.

   - NoOp **Success - Array of Objects** and StopAndError **Error - Array of Objects**:  
     - Error message: `"Incorrect. Hint: Combine array access [1] with object access .name."`  
   - Connect **Check - Array of Objects** accordingly.

7. **Step 5: JS Function String Transformation**  
   - Set node **Test - JS Function** with field `uppercase_name` expression: `={{ $('Source Data').item.json.name.toUpperCase() }}`  
   - Connect **Success - Array of Objects** to **Test - JS Function**.

   - If node **Check - JS Function**:  
     - Check `$json.uppercase_name` equals `{{ $('Source Data').item.json.name.toUpperCase() }}`  
     - Check expression includes `.toUpperCase()`  
   - Connect **Test - JS Function** to **Check - JS Function**.

   - NoOp **Success - JS Function** and StopAndError **Error - JS Function**:  
     - Error message: `"Incorrect. Hint: Add .toUpperCase() to the end of your expression, inside the {{ }}."`  
   - Connect **Check - JS Function** accordingly.

8. **Step 6: Final Challenge – String Interpolation**  
   - Set node **Test - Final** with field `summary` expression:  
     `=The status of task '{{ $('Source Data').item.json.tasks[1].name }}' is {{ $('Source Data').item.json.tasks[1].status }}.`  
   - Connect **Success - JS Function** to **Test - Final**.

   - If node **Check - Final**:  
     - Check `$json.summary` equals the exact string as above.  
     - Check expression includes both `.name` and `.status`  
   - Connect **Test - Final** to **Check - Final**.

   - NoOp **Success - Final** and StopAndError **Error - Final**:  
     - Error message: `"Incorrect. Hint: Combine static text and expressions like this: Some text '{{ expression1 }}' and '{{ expression2 }}'."`  
   - Connect **Check - Final** accordingly.

9. **Final Success Display**  
   - HTML node **🎉 SUCCESS 🎉**:  
     - Paste the provided styled HTML congratulatory content.  
   - Connect **Success - Final** to **🎉 SUCCESS 🎉**.

10. **Add all sticky notes** as instructional and feedback nodes at appropriate positions for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow and content © 2025 Lucas Peyrin. This practical test helps users validate understanding of n8n expressions with progressive difficulty.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Author credit                                                                                           |
| Animated GIF demonstrating data structure: ![Source](https://media4.giphy.com/media/v1.Y2lkPTc5MGI3NjExendjcXhicnQ1bzljNnVkdWlucThyanBvbndmcm8zd3FkdjJ6bGNsdSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/3ohzdZO0nAL1H2LdMA/giphy.gif)                                                                                                                                                                                                                                                                                                                                                                                                    | Instruction - Basic Access1 sticky note                                                                |
| Final success message includes link: [Check out more templates](https://n8n.io/creators/lucaspeyrin)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | 🎉 SUCCESS 🎉 HTML node                                                                                  |
| Feedback invitation and coaching/consulting offers with links:  
[Feedback Form](https://api.ia2s.app/form/templates/feedback?template=Expressions%20Test)  
[Book Coaching](https://api.ia2s.app/form/templates/coaching?template=Expressions%20Test)  
[Consulting Inquiry](https://api.ia2s.app/form/templates/consulting?template=Expressions%20Test)  
[n8n Academy](https://n8n.ac)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note10 providing community engagement and expert services                                       |
| Animated GIF linking to test skills page:  
[![Test Skills](https://supastudio.ia2s.app/storage/v1/object/public/assets/n8n/n8n_animation.gif)](https://n8n.io/creators/lucaspeyrin)                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note16                                                                                          |

---

**Disclaimer:** The text above is derived exclusively from an automated n8n workflow created with n8n, respecting all current content policies. It contains no illegal or offensive content. All data processed is legal and publicly available.