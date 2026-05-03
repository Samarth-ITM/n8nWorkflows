Fetch Scriptures Dynamically from get Bible API

https://n8nworkflows.xyz/workflows/fetch-scriptures-dynamically-from-get-bible-api-2818


# Fetch Scriptures Dynamically from get Bible API

### 1. Workflow Overview

The **GetBible Query Workflow** is a modular, self-contained n8n workflow designed to dynamically fetch scripture passages from the GetBible API based on structured JSON input. It acts as an intermediary layer that accepts references, translation, and API version parameters, processes them, queries the API, and returns the scripture data in a standardized JSON format consistent with the API’s native response.

**Target Use Cases:**  
- Bible study applications requiring dynamic scripture retrieval  
- Theological research tools needing structured verse data  
- Dynamic content generation involving scripture references  
- Sermon preparation automation  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives structured JSON input with scripture references, translation, and API version.  
- **1.2 Input Normalization:** Converts the array of references into a semicolon-separated string suitable for API querying.  
- **1.3 API Query:** Constructs the API URL dynamically and sends an HTTP request to GetBible API.  
- **1.4 Response Formatting:** Wraps the API response into a JSON object under a `result` key for standardized output.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block serves as the entry point for the workflow. It is designed to be triggered externally by another workflow or manually for testing, accepting a JSON object with scripture references, translation, and API version.

- **Nodes Involved:**  
  - Entry (Execute Workflow Trigger)

- **Node Details:**

  - **Entry**  
    - Type: `Execute Workflow Trigger`  
    - Role: Receives JSON input to start the workflow.  
    - Configuration: Uses a JSON example input for testing, mimicking the expected external call format.  
    - Key Variables:  
      - `references`: Array of scripture references (e.g., ["1 John 3:16", "Jn 3:16"])  
      - `translation`: Bible translation code (e.g., "kjv")  
      - `version`: API version string (e.g., "v2")  
    - Input Connections: None (trigger node)  
    - Output Connections: Passes data to `ModelJson` node  
    - Edge Cases:  
      - Missing or malformed JSON input could cause downstream errors.  
      - If `references` is missing or not an array, handled in next block.  
    - Version Requirements: n8n version supporting `Execute Workflow Trigger` node (v1.1 or later)

#### 1.2 Input Normalization

- **Overview:**  
  Converts the input `references` array into a semicolon-separated string, which matches the GetBible API’s expected query format. Provides a default reference if input is missing or malformed.

- **Nodes Involved:**  
  - ModelJson (Code node)

- **Node Details:**

  - **ModelJson**  
    - Type: `Code` (JavaScript)  
    - Role: Processes input JSON to normalize scripture references into a single string.  
    - Configuration:  
      - Checks if `references` exists and is an array.  
      - Joins array elements with `; ` separator.  
      - Defaults to `"John 3:16"` if no valid references provided.  
    - Key Expressions:  
      - `Array.isArray(item.json.references)`  
      - `item.json.references.join('; ')`  
    - Input Connections: From `Entry` node  
    - Output Connections: To `API Query to GetBible` node  
    - Edge Cases:  
      - Handles missing or invalid `references` gracefully by defaulting.  
      - If input is large, ensure string length limits for URL are respected (not explicitly handled).  
    - Version Requirements: n8n version supporting Code node v2

#### 1.3 API Query

- **Overview:**  
  Constructs the HTTP request URL dynamically using input parameters and sends a GET request to the GetBible API to fetch scripture data.

- **Nodes Involved:**  
  - API Query to GetBible (HTTP Request node)

- **Node Details:**

  - **API Query to GetBible**  
    - Type: `HTTP Request`  
    - Role: Queries the GetBible API with constructed URL.  
    - Configuration:  
      - URL template: `https://query.getbible.net/{{ $json.version || 'v2' }}/{{ $json.translation || 'kjv' }}/{{ $json.references }}`  
      - HTTP Method: GET (default)  
      - No authentication required (public API)  
      - No additional headers or options specified  
    - Key Expressions:  
      - Uses n8n expressions to dynamically build URL from JSON fields.  
    - Input Connections: From `ModelJson` node  
    - Output Connections: To `Map API Respons to Result` node  
    - Edge Cases:  
      - Network errors, timeouts, or API downtime may cause failures.  
      - Invalid references or unsupported translations may return errors or empty results.  
      - URL length limits could be exceeded with very large inputs.  
    - Version Requirements: n8n version supporting HTTP Request node v4.2

#### 1.4 Response Formatting

- **Overview:**  
  Wraps the raw API response JSON into an object under the key `result` to maintain consistent output formatting aligned with the GetBible API response structure.

- **Nodes Involved:**  
  - Map API Respons to Result (Set node)

- **Node Details:**

  - **Map API Respons to Result**  
    - Type: `Set`  
    - Role: Transforms the HTTP response by assigning it to a `result` property.  
    - Configuration:  
      - Sets a new JSON field `result` with the entire API response JSON as its value.  
    - Input Connections: From `API Query to GetBible` node  
    - Output Connections: None (end of workflow)  
    - Edge Cases:  
      - If API response is empty or malformed, the `result` field will reflect that directly.  
      - No additional error handling or validation performed here.  
    - Version Requirements: n8n version supporting Set node v3.4

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                  | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                                   |
|-------------------------|----------------------------|---------------------------------|---------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note             | Sticky Note                | Documentation and overview      | None                | None                     | Contains detailed documentation, usage instructions, JSON input/output examples, and support links.          |
| Entry                   | Execute Workflow Trigger   | Receives JSON input to start    | None                | ModelJson                |                                                                                                              |
| ModelJson               | Code                       | Normalizes references array into semicolon-separated string | Entry               | API Query to GetBible    |                                                                                                              |
| API Query to GetBible   | HTTP Request               | Queries GetBible API with constructed URL | ModelJson           | Map API Respons to Result |                                                                                                              |
| Map API Respons to Result | Set                      | Wraps API response in `result` field | API Query to GetBible | None                     |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add an `Execute Workflow Trigger` node:**  
   - Name: `Entry`  
   - Configure to accept JSON input.  
   - For testing, set the JSON example input to:  
     ```json
     {
       "references": [
         "1 John 3:16",
         "Jn 3:16",
         "James 3:16",
         "Rom 3:16"
       ],
       "translation": "kjv",
       "version": "v2"
     }
     ```  
   - This node will serve as the workflow entry point.

3. **Add a `Code` node:**  
   - Name: `ModelJson`  
   - Set to execute once per workflow run.  
   - Paste the following JavaScript code:  
     ```javascript
     for (let item of $input.all()) {
       if (Array.isArray(item.json.references)) {
         item.json.references = item.json.references.join('; ');
       } else {
         item.json.references = 'John 3:16';
       }
     }
     return $input.all();
     ```  
   - Connect `Entry` node output to this node input.

4. **Add an `HTTP Request` node:**  
   - Name: `API Query to GetBible`  
   - Set HTTP Method to GET.  
   - Set URL to an expression:  
     ```
     https://query.getbible.net/{{ $json.version || 'v2' }}/{{ $json.translation || 'kjv' }}/{{ $json.references }}
     ```  
   - No authentication or additional headers needed.  
   - Connect `ModelJson` node output to this node input.

5. **Add a `Set` node:**  
   - Name: `Map API Respons to Result`  
   - Add a new field assignment:  
     - Field Name: `result`  
     - Type: `Object`  
     - Value: `={{ $json }}` (assign entire incoming JSON response)  
   - Connect `API Query to GetBible` node output to this node input.

6. **Save the workflow.**

7. **Test the workflow:**  
   - Trigger the workflow manually or call it from another workflow passing the JSON input as specified.  
   - The output will be a JSON object with the scripture data under the `result` key, formatted as per the GetBible API.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow is designed for seamless integration and requires no additional configuration beyond JSON input.         | Workflow overview and usage instructions                   |
| Supported translations can be found at: https://api.getbible.net/v2/translations.json                                 | Translation reference                                      |
| API documentation is available at: https://getbible.net/docs                                                          | API usage and advanced features                            |
| Support and community help: https://git.vdm.dev/getBible/support                                                       | Support desk for troubleshooting and questions            |
| The workflow returns responses identical in structure to the GetBible API, ensuring compatibility with existing tools. | Integration and usage notes                                |

---

This completes the comprehensive reference documentation for the **GetBible Query Workflow**.