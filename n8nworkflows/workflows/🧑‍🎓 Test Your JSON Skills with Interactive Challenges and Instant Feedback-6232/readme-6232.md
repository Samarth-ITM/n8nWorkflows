🧑‍🎓 Test Your JSON Skills with Interactive Challenges and Instant Feedback

https://n8nworkflows.xyz/workflows/------test-your-json-skills-with-interactive-challenges-and-instant-feedback-6232


# 🧑‍🎓 Test Your JSON Skills with Interactive Challenges and Instant Feedback

### 1. Workflow Overview

This workflow is an interactive JSON knowledge test designed to help users practice and validate their understanding of basic JSON data types through a series of challenges with instant feedback. The workflow targets learners or automation developers who want to master JSON syntax and data structures within the n8n automation environment.

The workflow is logically divided into six main blocks corresponding to the core JSON data types:

- **1.1 String Challenge**: Validate a JSON string value.
- **1.2 Number Challenge**: Validate JSON numeric values (integer and decimal).
- **1.3 Boolean Challenge**: Validate JSON boolean values (`true` and `false`).
- **1.4 Null Challenge**: Validate a JSON null value.
- **1.5 Array Challenge**: Validate a JSON array with mixed types.
- **1.6 Object Challenge**: Validate nested JSON objects.

Each block includes instructional sticky notes, a "Test" node where users input their JSON, conditional checks to verify correctness, feedback sticky notes for success or failure, and nodes to guide the flow to the next challenge or show final success.

---

### 2. Block-by-Block Analysis

#### 1.1 String Challenge

- **Overview:** Introduces the concept of JSON strings by asking the user to input a JSON object with a key `"my_string"` and a string value `"I love automation"`. Validates the input and routes success or error feedback accordingly.

- **Nodes Involved:**  
  - Start Test! (Manual trigger)  
  - Test - String (Set node, user input)  
  - Check - String (If node, validation)  
  - Success - String (NoOp node)  
  - Error - String (StopAndError node)  
  - Feedback Correct1 (Sticky Note)  
  - Feedback Incorrect1 (Sticky Note)  
  - Answer - String (Set node, correct answer)  
  - Answer Note - String (Sticky Note)  
  - Instruction - String1 (Sticky Note)  

- **Node Details:**

  - **Start Test!**  
    - Type: Manual Trigger  
    - Role: Entry point to start the test workflow  
    - Output: Feeds into "Test - String" node  

  - **Test - String**  
    - Type: Set node  
    - Role: User modifies the JSON object here to include `"my_string"` key  
    - Configuration: Raw JSON mode with placeholder for `"my_string"` value  
    - Input: Trigger from "Start Test!"  
    - Output: To "Check - String"  

  - **Check - String**  
    - Type: If node  
    - Role: Validates that `"my_string"` equals `"I love automation"` (case insensitive, trimmed)  
    - Key Expression: `{{$json.my_string.trim().toLowerCase()}} === 'i love automation'`  
    - Output:  
      - True branch to "Success - String"  
      - False branch to "Error - String"  

  - **Success - String**  
    - Type: NoOp (No operation)  
    - Role: Marks success and continues workflow  
    - Output: To "Test - Number" (next challenge)  

  - **Error - String**  
    - Type: StopAndError  
    - Role: Stops workflow on failure, shows hint about string quotes  

  - **Feedback Correct1 / Feedback Incorrect1**  
    - Type: Sticky Notes  
    - Role: Display feedback messages including hints and encouragement  

  - **Answer - String**  
    - Type: Set node  
    - Role: Contains the correct JSON answer for reference  

  - **Instruction - String1**  
    - Type: Sticky Note  
    - Role: Provides instructions and task description  

- **Edge Cases / Failures:**  
  - User omitting quotes around string value  
  - Typographical errors in string  
  - Case sensitivity handled by trimming and lowercasing in condition  

---

#### 1.2 Number Challenge

- **Overview:** Tests user knowledge of JSON numbers by requiring a JSON object with keys `"product_id"` (integer) and `"price"` (decimal). Checks that numbers are not quoted.

- **Nodes Involved:**  
  - Test - Number (Set node)  
  - Check - Number (If node)  
  - Success - Number (NoOp node)  
  - Error - Number (StopAndError node)  
  - Feedback Correct2 (Sticky Note)  
  - Feedback Incorrect2 (Sticky Note)  
  - Answer - Number (Set node)  
  - Instruction - Number (Sticky Note)  

- **Node Details:**

  - **Test - Number**  
    - User inputs JSON with keys `product_id` and `price` (initially blank)  
    - Raw JSON mode  

  - **Check - Number**  
    - Validates `product_id === 12345` (number) and `price === 99.99` (number)  
    - Outputs success or error  

  - **Success - Number**  
    - Passes flow to "Test - Boolean" node  

  - **Error - Number**  
    - Stops workflow, shows hint about numbers not being quoted  

  - **Feedback Correct2 / Feedback Incorrect2**  
    - Provide encouragement or error hints  

  - **Answer - Number**  
    - Contains correct JSON answer for reference  

  - **Instruction - Number**  
    - Step description and animated GIF  

- **Edge Cases / Failures:**  
  - Numbers written as strings (quoted)  
  - Typo in numbers or decimal points  

---

#### 1.3 Boolean Challenge

- **Overview:** Validates JSON boolean values with keys `"is_active": true` and `"has_permission": false`.

- **Nodes Involved:**  
  - Test - Boolean (Set node)  
  - Check - Boolean (If node)  
  - Success - Boolean (NoOp node)  
  - Error - Boolean (StopAndError node)  
  - Feedback Correct3 (Sticky Note)  
  - Feedback Incorrect3 (Sticky Note)  
  - Answer - Boolean (Set node)  
  - Instruction - Boolean (Sticky Note)  

- **Node Details:**

  - **Test - Boolean**  
    - User input placeholder JSON with keys `"is_active"` and `"has_permission"`  

  - **Check - Boolean**  
    - Condition checks that `is_active === true` and `has_permission === false` (strict boolean match)  

  - **Success - Boolean**  
    - Flows to "Test - Null" node  

  - **Error - Boolean**  
    - Stops with error message about booleans without quotes  

  - **Answer - Boolean**  
    - Reference correct answer JSON  

  - **Instruction - Boolean**  
    - Instructions and GIF  

- **Edge Cases / Failures:**  
  - Booleans given as strings ("true", "false")  
  - Wrong boolean values or typos  

---

#### 1.4 Null Challenge

- **Overview:** Tests user on the JSON `null` value with a key `"middle_name": null`.

- **Nodes Involved:**  
  - Test - Null (Set node)  
  - Check - Null (If node)  
  - Success - Null (NoOp node)  
  - Error - Null (StopAndError node)  
  - Feedback Correct4 (Sticky Note)  
  - Feedback Incorrect7 (Sticky Note)  
  - Answer - Null (Set node)  
  - Instruction - Null (Sticky Note)  

- **Node Details:**

  - **Test - Null**  
    - Placeholder for `"middle_name"` with no value initially  

  - **Check - Null**  
    - Checks if `middle_name` is empty (`null`) and does not exist as a string  

  - **Success - Null**  
    - Flows to "Test - Array" node  

  - **Error - Null**  
    - Stops workflow with hint about `null` without quotes  

  - **Answer - Null**  
    - Reference JSON with `"middle_name": null`  

  - **Instruction - Null**  
    - Instructions and GIF  

- **Edge Cases / Failures:**  
  - Using quotes around `null`  
  - Using empty strings instead of `null`  

---

#### 1.5 Array Challenge

- **Overview:** Validates a JSON array assigned to key `"tags"` containing two strings and one number: `["n8n", "automation", 2024]`.

- **Nodes Involved:**  
  - Test - Array (Set node)  
  - Check - Array (If node)  
  - Success - Array (NoOp node)  
  - Error - Array (StopAndError node)  
  - Feedback Correct5 (Sticky Note)  
  - Feedback Incorrect5 (Sticky Note)  
  - Answer - Array (Set node)  
  - Instruction - Array (Sticky Note)  

- **Node Details:**

  - **Test - Array**  
    - User inputs JSON with `"tags"` key as array  

  - **Check - Array**  
    - Validates each array element:  
      - `tags[0] === "n8n"` (string)  
      - `tags[1] === "automation"` (string)  
      - `tags[2] === 2024` (number)  

  - **Success - Array**  
    - Flows to "Test - Object" node  

  - **Error - Array**  
    - Stops workflow with hints about order, types, and syntax  

  - **Answer - Array**  
    - Correct JSON answer  

  - **Instruction - Array**  
    - Instructions and GIF  

- **Edge Cases / Failures:**  
  - Wrong order of elements  
  - Using strings vs numbers incorrectly  
  - Syntax errors in array formatting  

---

#### 1.6 Object Challenge

- **Overview:** Tests user on nested JSON objects by requiring key `"user"` whose value is an object with keys `"name": "Alex"` and `"id": 987`.

- **Nodes Involved:**  
  - Test - Object (Set node)  
  - Check - Object (If node)  
  - Success - Object (NoOp node)  
  - Error - Object (StopAndError node)  
  - Feedback Correct6 (Sticky Note)  
  - Feedback Incorrect6 (Sticky Note)  
  - Answer - Object (Set node)  
  - Instruction - Object (Sticky Note)  
  - 🎉 SUCCESS 🎉 (HTML node)  
  - Congratulations! (Sticky Note)  

- **Node Details:**

  - **Test - Object**  
    - Placeholder for `"user"` object key  

  - **Check - Object**  
    - Validates nested object properties:  
      - `user.name === "Alex"` (string)  
      - `user.id === 987` (number)  

  - **Success - Object**  
    - Flows to final success HTML node  

  - **Error - Object**  
    - Stops workflow with hints about curly braces and types  

  - **Answer - Object**  
    - Reference JSON with nested object  

  - **Instruction - Object**  
    - Instructions and GIF  

  - **🎉 SUCCESS 🎉**  
    - Displays a styled HTML success card with congratulatory message and link to templates  

  - **Congratulations!**  
    - Sticky note congratulating user on passing the test  

- **Edge Cases / Failures:**  
  - Missing nested braces  
  - Wrong data types inside nested object  
  - Typographical errors  

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                        | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                                  |
|----------------------|--------------------|-------------------------------------|-----------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Start Test!          | Manual Trigger     | Entry point to start the test        |                       | Test - String             |                                                                                                              |
| Test - String        | Set                | User JSON input for string step      | Start Test!            | Check - String            |                                                                                                              |
| Check - String       | If                  | Validates string correctness         | Test - String          | Success - String, Error - String |                                                                                                              |
| Success - String     | NoOp                | Marks string success, continue        | Check - String (true)  | Test - Number             |                                                                                                              |
| Error - String       | StopAndError        | Stops on string error                 | Check - String (false) |                          |                                                                                                              |
| Feedback Correct1    | Sticky Note         | Correct feedback for string step     |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Feedback Incorrect1  | Sticky Note         | Incorrect feedback for string step   |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Answer - String      | Set                 | Correct answer JSON for string       |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Instruction - String1| Sticky Note         | Instructions for string step         |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Test - Number        | Set                 | User JSON input for number step      | Success - String       | Check - Number            |                                                                                                              |
| Check - Number       | If                  | Validates number correctness         | Test - Number          | Success - Number, Error - Number |                                                                                                              |
| Success - Number     | NoOp                | Marks number success, continue       | Check - Number (true)  | Test - Boolean            |                                                                                                              |
| Error - Number       | StopAndError        | Stops on number error                 | Check - Number (false) |                          |                                                                                                              |
| Feedback Correct2    | Sticky Note         | Correct feedback for number step     |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Feedback Incorrect2  | Sticky Note         | Incorrect feedback for number step   |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Answer - Number      | Set                 | Correct answer JSON for number       |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Instruction - Number | Sticky Note         | Instructions for number step         |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Test - Boolean       | Set                 | User JSON input for boolean step     | Success - Number       | Check - Boolean           |                                                                                                              |
| Check - Boolean      | If                  | Validates boolean correctness        | Test - Boolean         | Success - Boolean, Error - Boolean |                                                                                                              |
| Success - Boolean    | NoOp                | Marks boolean success, continue      | Check - Boolean (true) | Test - Null               |                                                                                                              |
| Error - Boolean      | StopAndError        | Stops on boolean error                | Check - Boolean (false)|                          |                                                                                                              |
| Feedback Correct3    | Sticky Note         | Correct feedback for boolean step    |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Feedback Incorrect3  | Sticky Note         | Incorrect feedback for boolean step  |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Answer - Boolean     | Set                 | Correct answer JSON for boolean      |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Instruction - Boolean| Sticky Note         | Instructions for boolean step        |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Test - Null          | Set                 | User JSON input for null step        | Success - Boolean      | Check - Null              |                                                                                                              |
| Check - Null         | If                  | Validates null correctness           | Test - Null            | Success - Null, Error - Null |                                                                                                              |
| Success - Null       | NoOp                | Marks null success, continue         | Check - Null (true)    | Test - Array              |                                                                                                              |
| Error - Null         | StopAndError        | Stops on null error                   | Check - Null (false)   |                          |                                                                                                              |
| Feedback Correct4    | Sticky Note         | Correct feedback for null step       |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Feedback Incorrect7  | Sticky Note         | Incorrect feedback for null step     |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Answer - Null        | Set                 | Correct answer JSON for null         |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Instruction - Null   | Sticky Note         | Instructions for null step           |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Test - Array         | Set                 | User JSON input for array step       | Success - Null         | Check - Array             |                                                                                                              |
| Check - Array        | If                  | Validates array correctness          | Test - Array           | Success - Array, Error - Array |                                                                                                              |
| Success - Array      | NoOp                | Marks array success, continue        | Check - Array (true)   | Test - Object             |                                                                                                              |
| Error - Array        | StopAndError        | Stops on array error                  | Check - Array (false)  |                          |                                                                                                              |
| Feedback Correct5    | Sticky Note         | Correct feedback for array step      |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Feedback Incorrect5  | Sticky Note         | Incorrect feedback for array step    |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Answer - Array       | Set                 | Correct answer JSON for array        |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Instruction - Array  | Sticky Note         | Instructions for array step          |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Test - Object        | Set                 | User JSON input for object step      | Success - Array        | Check - Object            |                                                                                                              |
| Check - Object       | If                  | Validates object correctness         | Test - Object          | Success - Object, Error - Object |                                                                                                              |
| Success - Object     | NoOp                | Marks object success, continue       | Check - Object (true)  | 🎉 SUCCESS 🎉             |                                                                                                              |
| Error - Object       | StopAndError        | Stops on object error                 | Check - Object (false) |                          |                                                                                                              |
| Feedback Correct6    | Sticky Note         | Correct feedback for object step     |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Feedback Incorrect6  | Sticky Note         | Incorrect feedback for object step   |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Answer - Object      | Set                 | Correct answer JSON for object       |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| Instruction - Object | Sticky Note         | Instructions for object step         |                       |                          | © 2025 Lucas Peyrin                                                                                            |
| 🎉 SUCCESS 🎉         | HTML                | Final success message display        | Success - Object       |                          | Well done! You're awesome © 2025 Lucas Peyrin                                                                  |
| Congratulations!     | Sticky Note         | Final congratulatory message         |                       |                          | © 2025 Lucas Peyrin                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create entry node:**  
   - Add a **Manual Trigger** node named "Start Test!"  

2. **Step 1: String Challenge**  
   - Add a **Set** node named "Test - String"  
     - Mode: Raw JSON  
     - JSON output placeholder: `{ "my_string": }` (empty value)  
   - Connect "Start Test!" → "Test - String"  
   - Add an **If** node "Check - String"  
     - Condition:  
       - Type: string equals  
       - Expression: `{{$json.my_string.trim().toLowerCase()}}` equals `"i love automation"`  
   - Connect "Test - String" → "Check - String"  
   - Add **NoOp** node "Success - String"  
     - Connect "Check - String" true → "Success - String"  
   - Add **StopAndError** node "Error - String"  
     - Error message: `Incorrect String. Hint: A string value must always be enclosed in double quotes " ". Check for typos!`  
     - Connect "Check - String" false → "Error - String"  
   - Connect "Success - String" → next step "Test - Number"  

3. **Step 2: Number Challenge**  
   - Add **Set** node "Test - Number"  
     - JSON output placeholder:  
       ```json
       {
         "product_id": ,
         "price": 
       }
       ```  
   - Connect "Success - String" → "Test - Number"  
   - Add **If** node "Check - Number"  
     - Conditions:  
       - `product_id` equals `12345` (number)  
       - `price` equals `99.99` (number)  
   - Connect "Test - Number" → "Check - Number"  
   - Add **NoOp** "Success - Number"  
   - Add **StopAndError** "Error - Number"  
     - Error message: `Incorrect Number. Hint: Numbers must not be in quotes. Check both the integer and the decimal value.`  
   - Connect "Check - Number" true → "Success - Number"  
   - Connect "Check - Number" false → "Error - Number"  
   - Connect "Success - Number" → "Test - Boolean"  

4. **Step 3: Boolean Challenge**  
   - Add **Set** node "Test - Boolean" (initially disabled)  
     - JSON output placeholder:  
       ```json
       {
         "is_active": ,
         "has_permission": 
       }
       ```  
   - Connect "Success - Number" → "Test - Boolean"  
   - Add **If** node "Check - Boolean"  
     - Conditions:  
       - `is_active` equals `true` (boolean)  
       - `has_permission` equals `false` (boolean)  
   - Connect "Test - Boolean" → "Check - Boolean"  
   - Add **NoOp** "Success - Boolean"  
   - Add **StopAndError** "Error - Boolean"  
     - Error message: `Incorrect Boolean. Hint: Boolean values are true or false and must be written without any quotes.`  
   - Connect "Check - Boolean" true → "Success - Boolean"  
   - Connect "Check - Boolean" false → "Error - Boolean"  
   - Connect "Success - Boolean" → "Test - Null"  

5. **Step 4: Null Challenge**  
   - Add **Set** node "Test - Null" (initially disabled)  
     - JSON output placeholder: `{ "middle_name": }` (null value)  
   - Connect "Success - Boolean" → "Test - Null"  
   - Add **If** node "Check - Null"  
     - Conditions:  
       - `middle_name` is empty (null)  
       - `middle_name` string does not exist  
   - Connect "Test - Null" → "Check - Null"  
   - Add **NoOp** "Success - Null"  
   - Add **StopAndError** "Error - Null"  
     - Error message: `Incorrect Null. Hint: The null value represents 'nothing' and is written as null without quotes.`  
   - Connect "Check - Null" true → "Success - Null"  
   - Connect "Check - Null" false → "Error - Null"  
   - Connect "Success - Null" → "Test - Array"  

6. **Step 5: Array Challenge**  
   - Add **Set** node "Test - Array" (initially disabled)  
     - JSON output placeholder: `{ "tags": }` (array)  
   - Connect "Success - Null" → "Test - Array"  
   - Add **If** node "Check - Array"  
     - Conditions:  
       - `tags[0]` equals `"n8n"` (string)  
       - `tags[1]` equals `"automation"` (string)  
       - `tags[2]` equals `2024` (number)  
   - Connect "Test - Array" → "Check - Array"  
   - Add **NoOp** "Success - Array"  
   - Add **StopAndError** "Error - Array"  
     - Error message: `Incorrect Array. Hint: Check the order of items, data types (string vs number), and the syntax: [ ] , .`  
   - Connect "Check - Array" true → "Success - Array"  
   - Connect "Check - Array" false → "Error - Array"  
   - Connect "Success - Array" → "Test - Object"  

7. **Step 6: Object Challenge**  
   - Add **Set** node "Test - Object" (initially disabled)  
     - JSON output placeholder:  
       ```json
       {
         "user": 
       }
       ```  
   - Connect "Success - Array" → "Test - Object"  
   - Add **If** node "Check - Object"  
     - Conditions:  
       - `user.name` equals `"Alex"` (string)  
       - `user.id` equals `987` (number)  
   - Connect "Test - Object" → "Check - Object"  
   - Add **NoOp** "Success - Object"  
   - Add **StopAndError** "Error - Object"  
     - Error message: `Incorrect Object. Hint: Remember to wrap the inner object in curly braces {} and check the data types inside it.`  
   - Connect "Check - Object" true → "Success - Object"  
   - Connect "Check - Object" false → "Error - Object"  

8. **Final Success Display**  
   - Add **HTML** node "🎉 SUCCESS 🎉"  
     - Content: Styled HTML page congratulating the user with a green checkmark and link to more templates  
   - Connect "Success - Object" → "🎉 SUCCESS 🎉"  

9. **Feedback and Instruction Sticky Notes**  
   - Add sticky notes at relevant positions with the instructions, hints, answer keys, and congratulatory messages as per the original workflow text and images.  

10. **Disable Test nodes initially**  
    - All "Test - ..." nodes start disabled except the first one to enforce stepwise progression.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| JSON Knowledge Test: Welcome message explains how to use this test—modify "Test - ..." nodes, execute workflow, and interpret green/red paths.                                                                                                                                                                                                                               | Sticky Note at workflow start                                                                            |
| The success HTML node contains a visually appealing card with a green checkmark and a link to [Lucas Peyrin's n8n templates](https://n8n.io/creators/lucaspeyrin).                                                                                                                                                                                                             | Final success node                                                                                        |
| Feedback mechanism includes hints emphasizing JSON syntax rules: strings need quotes, numbers must not be quoted, booleans are literals `true`/`false`, null is `null` without quotes, arrays require proper syntax and order, and nested objects require braces and correct types.                                                                                           | Across all error messages                                                                                 |
| Multiple animated GIFs are used in sticky notes to visually reinforce JSON concepts for each data type (strings, numbers, booleans, null, arrays, objects).                                                                                                                                                                                                                   | Instruction sticky notes                                                                                  |
| The workflow is authored and maintained by Lucas Peyrin, with copyright year 2025 consistently noted on all nodes and sticky notes.                                                                                                                                                                                                                                         | Metadata and sticky notes                                                                                 |
| User is encouraged to provide feedback or coaching inquiries via external forms linked in the large sticky note near the end of the workflow.                                                                                                                                                                                                                                | Sticky Note with feedback and coaching links: https://api.ia2s.app/form/templates/feedback?template=JSON%20Test |
| The workflow emphasizes progressive learning by enabling only one test node at a time and routing success to the next test node, ensuring stepwise mastery.                                                                                                                                                                                                                 | Workflow connections and node disable states                                                            |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.