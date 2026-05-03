Learn JSON Basics with an Interactive Step-by-Step Tutorial for Beginners

https://n8nworkflows.xyz/workflows/learn-json-basics-with-an-interactive-step-by-step-tutorial-for-beginners-5170


# Learn JSON Basics with an Interactive Step-by-Step Tutorial for Beginners

### 1. Workflow Overview

This workflow is an interactive step-by-step tutorial designed to teach beginners the basics of JSON (JavaScript Object Notation) using n8n’s nodes. It breaks down JSON concepts into fundamental building blocks such as strings, numbers, booleans, null, arrays, and objects, illustrating each with concrete examples and explanations. The workflow culminates with a final node that synthesizes all the previous data types into one comprehensive JSON object, demonstrating how to use expressions to access and combine data dynamically.

**Target Use Cases:**  
- Educational tool for newcomers to JSON and n8n automation  
- Demonstration of JSON data types and structure within n8n  
- Reference for building JSON-related workflows and understanding data passing via expressions

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger to start the tutorial  
- **1.2 JSON Basics Setup:** Nodes illustrating basic JSON components (Key/Value, String, Number, Boolean, Null, Array, Object) with explanatory sticky notes  
- **1.3 Expression Usage:** Demonstrates referencing previous node data with expressions  
- **1.4 Final Synthesis:** Combines all previous JSON data into a summary object  
- **1.5 Documentation:** Sticky notes providing detailed explanations for each concept

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Starts the workflow manually by user interaction to enable stepwise exploration.

**Nodes Involved:**  
- Execute to Start (Manual Trigger)

**Node Details:**  
- **Execute to Start**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow run when the user executes it manually  
  - Configuration: Default manual trigger, no parameters  
  - Input: None (manual start)  
  - Output: Starts the chain to the “Key & Value” node  
  - Edge Cases: Workflow will not run unless manually triggered; no authentication issues here

#### 1.2 JSON Basics Setup

**Overview:**  
Sequential nodes set example JSON values for different data types and structures with corresponding sticky notes explaining each concept.

**Nodes Involved:**  
- Key & Value  
- String  
- Number  
- Boolean  
- Null  
- Array  
- Object  
- Sticky Note (multiple, each linked visually to specific nodes)

**Node Details:**  

- **Key & Value**  
  - Type: Set  
  - Role: Defines basic key/value pairs foundational to JSON  
  - Configuration: Sets two string keys “key” and “another_key” with values “value” and “another_value” respectively  
  - Inputs: From “Execute to Start”  
  - Outputs: To “String” node  
  - Edge Cases: No dynamic expressions; simple static assignment

- **String**  
  - Type: Set  
  - Role: Demonstrates a JSON string value enclosed in double quotes  
  - Configuration: Assigns “json_example_string” with a textual string explaining JSON string syntax  
  - Inputs: From “Key & Value”  
  - Outputs: To “Number” node  
  - Edge Cases: None, static string

- **Number**  
  - Type: Set  
  - Role: Illustrates JSON numeric values, integer and float  
  - Configuration: Sets “json_example_integer” as 10 (integer) and “json_example_float” as 12.5 (float) without quotes  
  - Inputs: From “String”  
  - Outputs: To “Boolean”  
  - Edge Cases: Numeric type correctness critical; expressions must not convert numbers to strings

- **Boolean**  
  - Type: Set  
  - Role: Shows boolean true/false values in JSON  
  - Configuration: Sets “json_example_boolean” as true (boolean, no quotes)  
  - Inputs: From “Number”  
  - Outputs: To “Null”  
  - Edge Cases: Boolean must be lowercase and unquoted to be valid JSON boolean

- **Null**  
  - Type: Set  
  - Role: Represents JSON null value  
  - Configuration: Sets “json_example_null” as null (no quotes)  
  - Inputs: From “Boolean”  
  - Outputs: To “Array”  
  - Edge Cases: Differentiates from empty string or zero; intentional absence of value

- **Array**  
  - Type: Set  
  - Role: Demonstrates JSON array containing mixed data types  
  - Configuration: Assigns “json_example_array” with an array of mixed types: string, number, boolean, null  
  - Inputs: From “Null”  
  - Outputs: To “Object”  
  - Edge Cases: Correct JSON array syntax with brackets and commas required

- **Object**  
  - Type: Set  
  - Role: Shows a complex JSON object combining keys with various data types, including nested objects and arrays  
  - Configuration: Sets “json_example_object” containing keys with string, array, boolean, integer, and nested sub-object  
  - Inputs: From “Array”  
  - Outputs: To “Using JSON (Expressions)”  
  - Edge Cases: Nested references require correct syntax; watch for valid JSON object structure

- **Sticky Notes** (Multiple nodes)  
  - Type: Sticky Note  
  - Role: Provide detailed textual explanations for JSON concepts presented by each data node  
  - Positions correspond visually near their related nodes for tutorial clarity  
  - No inputs or outputs; purely informational  
  - Edge Cases: None

#### 1.3 Expression Usage

**Overview:**  
Illustrates how to use n8n expressions to extract and manipulate JSON data from previous nodes dynamically.

**Nodes Involved:**  
- Using JSON (Expressions)  
- Sticky Note8

**Node Details:**  

- **Using JSON (Expressions)**  
  - Type: Set  
  - Role: Demonstrates extracting values using expressions from prior nodes, showing dynamic referencing  
  - Configuration:  
    - `message`: Static string concatenated with expression pulling integer value from “Number” node  
    - `sub_key`: Extracts nested key from “Object” node’s sub_object  
    - `array_second_item`: Gets second item from the “Object” node’s array  
  - Inputs: From “Object”  
  - Outputs: To “Final Exam”  
  - Edge Cases: Expression errors if referenced node or key is missing; care with indexing arrays

- **Sticky Note8**  
  - Type: Sticky Note  
  - Role: Explains the importance of using expressions `{{ }}` to dynamically reference JSON data between nodes  
  - Position: Near “Using JSON (Expressions)” node  
  - Edge Cases: None

#### 1.4 Final Synthesis

**Overview:**  
Combines all previously defined JSON data types into one final object using expressions, summarizing the tutorial.

**Nodes Involved:**  
- Final Exam  
- Sticky Note9

**Node Details:**  

- **Final Exam**  
  - Type: Set  
  - Role: Aggregates all previous JSON data fields into a single JSON object using expressions  
  - Configuration: Sets keys for string, number, boolean, null, array, and object fields, each pulling data from respective nodes via expressions  
  - Inputs: From “Using JSON (Expressions)”  
  - Outputs: None (end of workflow)  
  - Edge Cases: Expression failures if any prior nodes are missing or output malformed data

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Congratulates the user and highlights the final integration of all JSON concepts learned  
  - Position: Adjacent to “Final Exam” node  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name                | Node Type         | Functional Role                                  | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                     |
|--------------------------|-------------------|-------------------------------------------------|------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Execute to Start          | Manual Trigger    | Starts workflow manually                         | -                      | Key & Value              |                                                                                                |
| Key & Value              | Set               | Defines basic key/value pairs                     | Execute to Start         | String                   | Sticky Note1 (The Heart of JSON: Key & Value)                                                  |
| String                   | Set               | Demonstrates JSON string type                     | Key & Value             | Number                   | Sticky Note2 (Data Type: String)                                                               |
| Number                   | Set               | Demonstrates JSON numbers (integer and float)    | String                  | Boolean                  | Sticky Note3 (Data Type: Number)                                                               |
| Boolean                  | Set               | Demonstrates JSON boolean type                    | Number                  | Null                     | Sticky Note4 (Data Type: Boolean)                                                              |
| Null                     | Set               | Demonstrates JSON null type                       | Boolean                 | Array                    | Sticky Note7 (Data Type: Null)                                                                 |
| Array                    | Set               | Demonstrates JSON array type                      | Null                    | Object                   | Sticky Note5 (Data Type: Array)                                                                |
| Object                   | Set               | Demonstrates JSON object with nested structures  | Array                   | Using JSON (Expressions) | Sticky Note6 (Data Type: Object)                                                               |
| Using JSON (Expressions) | Set               | Shows how to use expressions to access JSON data | Object                  | Final Exam               | Sticky Note8 (THE KEY STEP: Using JSON in n8n!)                                               |
| Final Exam               | Set               | Combines all data types into one final JSON object | Using JSON (Expressions) | -                        | Sticky Note9 (FINAL EXAM: Putting It All Together)                                            |
| Sticky Note              | Sticky Note       | Tutorial intro and explanation                    | -                      | -                        | Sticky Note (Welcome and JSON basics introduction)                                            |
| Sticky Note1             | Sticky Note       | Explanation of key/value pairs                    | -                      | -                        | Sticky Note1 (The Heart of JSON: Key & Value)                                                  |
| Sticky Note2             | Sticky Note       | Explanation of string data type                    | -                      | -                        | Sticky Note2 (Data Type: String)                                                               |
| Sticky Note3             | Sticky Note       | Explanation of number data type                    | -                      | -                        | Sticky Note3 (Data Type: Number)                                                               |
| Sticky Note4             | Sticky Note       | Explanation of boolean data type                   | -                      | -                        | Sticky Note4 (Data Type: Boolean)                                                              |
| Sticky Note5             | Sticky Note       | Explanation of array data type                     | -                      | -                        | Sticky Note5 (Data Type: Array)                                                                |
| Sticky Note6             | Sticky Note       | Explanation of object data type                    | -                      | -                        | Sticky Note6 (Data Type: Object)                                                               |
| Sticky Note7             | Sticky Note       | Explanation of null data type                      | -                      | -                        | Sticky Note7 (Data Type: Null)                                                                 |
| Sticky Note8             | Sticky Note       | Explanation of using expressions for JSON data   | -                      | -                        | Sticky Note8 (THE KEY STEP: Using JSON in n8n!)                                               |
| Sticky Note9             | Sticky Note       | Final summary and congratulations                  | -                      | -                        | Sticky Note9 (FINAL EXAM: Putting It All Together)                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `Execute to Start`  
   - Purpose: To start the workflow manually  
   - No parameters needed

2. **Create a Set Node**  
   - Name: `Key & Value`  
   - Connect input from `Execute to Start`  
   - Assign two string key/value pairs:  
     - `key`: `"value"`  
     - `another_key`: `"another_value"`

3. **Create a Set Node**  
   - Name: `String`  
   - Connect input from `Key & Value`  
   - Assign one string:  
     - `json_example_string`: `"This is a simple string. In JSON, it's always enclosed in double quotes."`

4. **Create a Set Node**  
   - Name: `Number`  
   - Connect input from `String`  
   - Assign two numbers:  
     - `json_example_integer`: `10` (integer, no quotes)  
     - `json_example_float`: `12.5` (float, no quotes)

5. **Create a Set Node**  
   - Name: `Boolean`  
   - Connect input from `Number`  
   - Assign one boolean:  
     - `json_example_boolean`: `true` (boolean type, no quotes)

6. **Create a Set Node**  
   - Name: `Null`  
   - Connect input from `Boolean`  
   - Assign one null:  
     - `json_example_null`: `null` (explicit null, no quotes)

7. **Create a Set Node**  
   - Name: `Array`  
   - Connect input from `Null`  
   - Assign one array with mixed types:  
     - `json_example_array`: `["first element", 2, false, null]` (array syntax)

8. **Create a Set Node**  
   - Name: `Object`  
   - Connect input from `Array`  
   - Assign one complex object:  
     ```json
     {
       "key": "value",
       "array": [1,2,3],
       "boolean": false,
       "integer": 123,
       "sub_object": {
         "sub_key": "Find me!"
       }
     }
     ```
     - Field name: `json_example_object`

9. **Create a Set Node**  
   - Name: `Using JSON (Expressions)`  
   - Connect input from `Object`  
   - Assign fields using expressions:  
     - `message`: `"Hello, the number from the tutorial is: {{ $('Number').item.json.json_example_integer }}"`  
     - `sub_key`: `={{ $json.json_example_object.sub_object.sub_key }}`  
     - `array_second_item`: `={{ $json.json_example_object.array[1] }}`

10. **Create a Set Node**  
    - Name: `Final Exam`  
    - Connect input from `Using JSON (Expressions)`  
    - Assign fields with expressions referencing all previous nodes:  
      - `summary_string`: `={{ $('String').item.json.json_example_string }}`  
      - `summary_number`: `={{ $('Number').item.json.json_example_integer }}`  
      - `summary_boolean`: `={{ $('Boolean').item.json.json_example_boolean }}`  
      - `summary_null`: `={{ $('Null').item.json.json_example_null }}`  
      - `summary_array`: `={{ $('Array').item.json.json_example_array }}`  
      - `summary_object`: `={{ $('Object').item.json.json_example_object }}`

11. **Add Sticky Notes** (Optional but recommended for tutorial clarity)  
    - Add sticky notes near each data node explaining the JSON concept it represents.  
    - Use markdown formatting for headings and bullet points.  
    - Position notes near corresponding nodes visually.

12. **Connect Nodes Sequentially**  
    - `Execute to Start` → `Key & Value` → `String` → `Number` → `Boolean` → `Null` → `Array` → `Object` → `Using JSON (Expressions)` → `Final Exam`

13. **Save and Execute Workflow**  
    - Use manual trigger to start  
    - Inspect output data at each node to verify JSON structures and understand the tutorial flow

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Welcome! This workflow teaches JSON basics, the language apps and n8n nodes use to exchange info. Start by executing the workflow and exploring each node's output and sticky notes. | Introduction sticky note at workflow start                                                                    |
| Expressions `{{ }}` allow dynamic referencing of data from other nodes, crucial for building flexible workflows in n8n.       | Sticky Note8 explaining expressions usage                                                                     |
| JSON data types covered: string, number, boolean, null, array, object — foundational for API integrations and data manipulation workflows. | Multiple sticky notes throughout the workflow                                                                 |
| Final node consolidates all learned data into one JSON object, demonstrating practical use of expressions and data aggregation. | Sticky Note9 congratulating users on completing the tutorial                                                  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.