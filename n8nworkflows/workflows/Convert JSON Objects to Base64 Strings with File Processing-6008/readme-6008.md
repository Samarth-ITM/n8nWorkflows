Convert JSON Objects to Base64 Strings with File Processing

https://n8nworkflows.xyz/workflows/convert-json-objects-to-base64-strings-with-file-processing-6008


# Convert JSON Objects to Base64 Strings with File Processing

### 1. Workflow Overview

This n8n workflow titled **"Convert JSON Objects to Base64 Strings with File Processing"** demonstrates how to convert a JSON object into a Base64-encoded string using n8n's internal file and binary data processing nodes. The primary use case is preparing JSON payloads for APIs, webhooks, or SaaS integrations that require Base64 encoding.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggering the workflow and creating example JSON data.
- **1.2 JSON Stringification:** Converting the JSON object into a raw string.
- **1.3 Binary Conversion:** Transforming the JSON string into binary format.
- **1.4 Base64 Encoding:** Extracting the Base64 string representation from the binary data.

Additionally, sticky notes provide contextual explanations and recommendations, including a suggestion to encapsulate the core encoding steps into a reusable sub-workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow execution manually and generates a sample JSON data object that will be encoded.

- **Nodes Involved:**  
  - Manual Execution  
  - Create Json Data  
  - Sticky Note (Create Example JSON data)

- **Node Details:**

  - **Manual Execution**  
    - *Type & Role:* Trigger node to start the workflow manually.  
    - *Configuration:* No parameters needed.  
    - *Input/Output:* No input; outputs trigger to "Create Json Data".  
    - *Failures:* Minimal risk; typical errors could be UI or permission-related.

  - **Create Json Data**  
    - *Type & Role:* Set node creating a raw JSON object with various data types (string, number, boolean, null, arrays, nested objects).  
    - *Configuration:* JSON object hardcoded in `jsonOutput` parameter with diverse data for testing encoding robustness.  
    - *Key Expressions:* None beyond static JSON.  
    - *Input/Output:* Receives trigger from Manual Execution; outputs JSON data to "Convert JSON to String".  
    - *Failures:* JSON syntax errors if modified incorrectly.

  - **Sticky Note (Create Example JSON data)**  
    - *Type & Role:* Documentation and guidance only.  
    - *Content:* "## Create Example JSON data" describing the purpose of this block.  
    - *No connections.*

#### 2.2 JSON Stringification

- **Overview:**  
  Converts the structured JSON object into a string representation using JavaScript’s `JSON.stringify`.

- **Nodes Involved:**  
  - Convert JSON to String  
  - Sticky Note1 (Stringify JSON and Save to Binary)

- **Node Details:**

  - **Convert JSON to String**  
    - *Type & Role:* Set node that applies a stringification expression to the input JSON.  
    - *Configuration:* Uses an assignment to create a new field `json_text` with the value `={{ JSON.stringify($json) }}` which converts the entire JSON input to a string.  
    - *Input/Output:* Input from "Create Json Data"; output to "Convert String to Binary".  
    - *Failures:* Expression failures if input JSON is invalid or too large; stringification edge cases with circular references (not present here).  
    - *Version:* Uses n8n v3.4 features for expression and JSON output.

  - **Sticky Note1 (Stringify JSON and Save to Binary)**  
    - *Type & Role:* Documentation explaining this step’s role in stringifying JSON and preparing for binary conversion.

#### 2.3 Binary Conversion

- **Overview:**  
  Converts the JSON string into a binary file representation, enabling n8n to handle it as file data for further Base64 processing.

- **Nodes Involved:**  
  - Convert String to Binary  
  - Sticky Note2 (Convert Binary Data to Base64 Encoded string)

- **Node Details:**

  - **Convert String to Binary**  
    - *Type & Role:* ConvertToFile node that transforms the `json_text` string field into a binary file named `encoded_text`.  
    - *Configuration:*  
      - Encoding set to `utf8`.  
      - Source property for conversion: `json_text`.  
      - Output binary property: `encoded_text`.  
    - *Input/Output:* Input from "Convert JSON to String"; output to "Extract Base64 from Binary".  
    - *Failures:* Encoding issues if `json_text` is empty or invalid; memory limits on very large strings.  
    - *Version:* Uses operation `"toText"` with encoding.

  - **Sticky Note2 (Convert Binary Data to Base64 Encoded string)**  
    - *Role:* Explains that this block prepares the binary data for Base64 extraction.

#### 2.4 Base64 Encoding

- **Overview:**  
  Extracts the Base64-encoded string from the binary file content, the final step producing a Base64 string representation of the original JSON.

- **Nodes Involved:**  
  - Extract Base64 from Binary  
  - Sticky Note3 (Encode JSON to Base64 String)

- **Node Details:**

  - **Extract Base64 from Binary**  
    - *Type & Role:* ExtractFromFile node that converts binary data to a Base64 string stored in a new field.  
    - *Configuration:*  
      - Operation: `binaryToPropery` (convert binary content to property).  
      - Binary Property Name: `encoded_text`.  
      - Destination key for Base64 string: `base64_text`.  
    - *Input/Output:* Input from "Convert String to Binary"; no outputs connected (end of workflow).  
    - *Failures:* Binary property missing or corrupted data; output property overwrite risks.  
    - *Version:* v1 node.

  - **Sticky Note3 (Encode JSON to Base64 String)**  
    - *Role:* Provides a detailed explanation of the entire workflow’s goal.  
    - *Content Summary:* Workflow converts JSON to Base64 string using file processing; suggests encapsulating the three green nodes (`Convert JSON to String`, `Convert String to Binary`, and `Extract Base64 from Binary`) into a reusable sub-workflow for modular use.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                     | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                          |
|---------------------------|------------------------|-----------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------|
| Manual Execution          | Manual Trigger         | Workflow start trigger             | -                        | Create Json Data            |                                                                                                                      |
| Create Json Data          | Set                    | Create sample JSON object          | Manual Execution          | Convert JSON to String      |                                                                                                                      |
| Convert JSON to String    | Set                    | Stringify JSON object to string    | Create Json Data          | Convert String to Binary    | ## Stringify JSON and Save to Binary                                                                                 |
| Convert String to Binary  | ConvertToFile          | Convert JSON string to binary file | Convert JSON to String    | Extract Base64 from Binary  | ## Convert Binary Data to Base64 Encoded string                                                                       |
| Extract Base64 from Binary| ExtractFromFile        | Extract Base64 string from binary  | Convert String to Binary  | -                          | ## Encode JSON to Base64 String<br>This example workflow demonstrates how to convert a JSON object into a base64-encoded string... |
| Sticky Note              | Sticky Note            | Documentation and explanation      | -                        | -                          | ## Create Example JSON data                                                                                           |
| Sticky Note1             | Sticky Note            | Documentation and explanation      | -                        | -                          | ## Stringify JSON and Save to Binary                                                                                 |
| Sticky Note2             | Sticky Note            | Documentation and explanation      | -                        | -                          | ## Convert Binary Data to Base64 Encoded string                                                                       |
| Sticky Note3             | Sticky Note            | Documentation and explanation      | -                        | -                          | ## Encode JSON to Base64 String<br>This example workflow demonstrates how to convert a JSON object into a base64-encoded string... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `Manual Execution`.  
   - No special configuration needed.

2. **Create JSON Data Node**  
   - Add a **Set** node named `Create Json Data`.  
   - Set mode to raw and paste the following JSON object into the `jsonOutput` field:  
     ```json
     {
       "string": "Hello, world!",
       "number": 42,
       "float": 3.14,
       "booleanTrue": true,
       "booleanFalse": false,
       "nullValue": null,
       "array": [1, "two", false, null],
       "nestedObject": {
         "id": 1,
         "name": "Nested",
         "attributes": {
           "active": true,
           "tags": ["test", "sample"]
         }
       },
       "arrayOfObjects": [
         { "type": "A", "value": 10 },
         { "type": "B", "value": 20 }
       ],
       "emptyArray": [],
       "emptyObject": {}
     }
     ```  
   - Connect output of `Manual Execution` to this node.

3. **Convert JSON to String Node**  
   - Add a **Set** node named `Convert JSON to String`.  
   - Configure an assignment:  
     - Name: `json_text`  
     - Type: String  
     - Value: Use expression `{{ JSON.stringify($json) }}` to convert the entire JSON input to string.  
   - Connect output of `Create Json Data` to this node.

4. **Convert String to Binary Node**  
   - Add a **ConvertToFile** node named `Convert String to Binary`.  
   - Configure:  
     - Operation: `toText`  
     - Source Property: `json_text`  
     - Binary Property Name: `encoded_text`  
     - Encoding: `utf8`  
   - Connect output of `Convert JSON to String` to this node.

5. **Extract Base64 from Binary Node**  
   - Add an **ExtractFromFile** node named `Extract Base64 from Binary`.  
   - Configure:  
     - Operation: `binaryToPropery`  
     - Binary Property Name: `encoded_text`  
     - Destination Key: `base64_text`  
   - Connect output of `Convert String to Binary` to this node.

6. **Add Sticky Notes for Documentation (Optional)**  
   - Add four **Sticky Note** nodes with the following contents and positions to document the workflow stages:  
     - "## Create Example JSON data" near the `Create Json Data` node.  
     - "## Stringify JSON and Save to Binary" near `Convert JSON to String`.  
     - "## Convert Binary Data to Base64 Encoded string" near `Convert String to Binary`.  
     - "## Encode JSON to Base64 String\nThis example workflow demonstrates how to convert a JSON object into a base64-encoded string using n8n’s built-in file processing capabilities. This is a common requirement when working with APIs, webhooks, or SaaS integrations that expect payloads to be base64-encoded.\n\n**Put the 3 nodes in green into a Sub and make a reusable base64 encoder in your projects.**" spanning above the three encoding nodes.

7. **Workflow Settings**  
   - Set execution order to v1 (default).  
   - Test workflow by executing the manual trigger and verify that the final node outputs a `base64_text` property containing the Base64 string of the original JSON.

8. **Optional Sub-Workflow Creation**  
   - To reuse encoding steps, create a sub-workflow that includes:  
     - `Convert JSON to String`  
     - `Convert String to Binary`  
     - `Extract Base64 from Binary`  
   - Input: JSON data object  
   - Output: Base64 string in `base64_text` property.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| This workflow shows a practical method to encode JSON objects to Base64 strings using n8n’s file and binary processing nodes.                                                                                                | Main workflow purpose                                       |
| Suggestion to encapsulate the three core encoding nodes into a reusable sub-workflow for convenience and modularity.                                                                                                         | Sticky Note3 content                                        |
| Base64 encoding of JSON is commonly needed for API integrations, webhooks, or any system requiring payloads to be transmitted in Base64-encoded format.                                                                      | Workflow description                                        |

---

**Disclaimer:** The provided content is generated exclusively from an n8n workflow and complies with all content policies. It contains no illegal, offensive, or protected material. All handled data is legal and public.