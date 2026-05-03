Calculate the Centroid of a Set of Vectors

https://n8nworkflows.xyz/workflows/calculate-the-centroid-of-a-set-of-vectors-2839


# Calculate the Centroid of a Set of Vectors

### 1. Workflow Overview

This workflow calculates the centroid of a set of vectors provided via an HTTP GET request. It is designed to accept an array of numerical vectors, validate that all vectors have consistent dimensions, compute their centroid by averaging each dimension, and return the result or an error message if validation fails. The workflow is reusable and suitable for projects requiring vector centroid computations with input validation.

**Logical Blocks:**

- **1.1 Input Reception:** Receives the input vectors via a webhook GET request.
- **1.2 Data Extraction & Parsing:** Extracts and converts the input parameter into a usable JSON array.
- **1.3 Validation & Computation:** Validates vector dimensions and computes the centroid.
- **1.4 Response Delivery:** Returns the computed centroid or error message to the client.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives an HTTP GET request containing the vectors parameter. It acts as the entry point of the workflow.

- **Nodes Involved:**  
  - Receive Vectors (Webhook)

- **Node Details:**

  - **Receive Vectors**  
    - **Type & Role:** Webhook node; receives HTTP GET requests.  
    - **Configuration:**  
      - HTTP Method: GET  
      - Path: `centroid`  
      - Response Mode: Uses a separate Respond to Webhook node to send the response.  
    - **Key Expressions/Variables:**  
      - Accesses query parameter `vectors` as a string representing an array of vectors.  
    - **Input/Output:**  
      - No input nodes (start node).  
      - Outputs to "Extract & Parse Vectors" node.  
    - **Version Requirements:** n8n Webhook node version 2.  
    - **Potential Failures:**  
      - Missing or malformed `vectors` query parameter.  
      - HTTP request errors or timeouts.  
    - **Sticky Note Context:**  
      Describes the expected input format and example request URL.  
    - **Sub-Workflow:** None.

#### 1.2 Data Extraction & Parsing

- **Overview:**  
  Converts the incoming `vectors` query string into a proper JSON array for further processing.

- **Nodes Involved:**  
  - Extract & Parse Vectors (Set Node)

- **Node Details:**

  - **Extract & Parse Vectors**  
    - **Type & Role:** Set node; transforms and assigns data fields.  
    - **Configuration:**  
      - Assigns `vectors` field as an array by parsing the `vectors` query parameter from the webhook JSON input.  
      - Uses expression: `={{ $json.query.vectors }}` to extract the raw string.  
      - Converts string to array type internally.  
    - **Input/Output:**  
      - Input from "Receive Vectors".  
      - Output to "Validate & Compute Centroid".  
    - **Version Requirements:** Set node version 3.4.  
    - **Potential Failures:**  
      - If `vectors` parameter is missing or not a valid array string, may cause downstream errors.  
      - No explicit error handling here; relies on downstream validation.  
    - **Sticky Note Context:**  
      Explains its role in ensuring `vectors` is a valid array and shows expected output format.  
    - **Sub-Workflow:** None.

#### 1.3 Validation & Computation

- **Overview:**  
  Validates that all vectors have the same dimension and computes the centroid by averaging each dimension.

- **Nodes Involved:**  
  - Validate & Compute Centroid (Code Node)

- **Node Details:**

  - **Validate & Compute Centroid**  
    - **Type & Role:** Code node; executes JavaScript for validation and computation.  
    - **Configuration:**  
      - Reads `vectors` from input JSON.  
      - Checks if `vectors` is an array and non-empty.  
      - Checks that all vectors have the same length (dimension).  
      - Computes centroid by summing each dimension and dividing by number of vectors.  
      - Returns either `{ centroid: [...] }` or `{ error: "..." }`.  
    - **Key Code Snippet:**  
      ```javascript
      const input = items[0].json;
      const vectors = input.vectors;

      if (!Array.isArray(vectors) || vectors.length === 0) {
        return [{ json: { error: "Invalid input: Expected an array of vectors." } }];
      }

      const dimension = vectors[0].length;
      if (!vectors.every(v => v.length === dimension)) {
        return [{ json: { error: "Vectors have inconsistent dimensions." } }];
      }

      const centroid = new Array(dimension).fill(0);
      vectors.forEach(vector => {
        vector.forEach((val, index) => {
          centroid[index] += val;
        });
      });

      for (let i = 0; i < dimension; i++) {
        centroid[i] /= vectors.length;
      }

      return [{ json: { centroid } }];
      ```
    - **Input/Output:**  
      - Input from "Extract & Parse Vectors".  
      - Output to "Return Centroid Response".  
    - **Version Requirements:** Code node version 2.  
    - **Potential Failures:**  
      - Input vectors missing or malformed.  
      - Dimension mismatch error.  
      - Runtime errors in code execution (e.g., unexpected data types).  
    - **Sticky Note Context:**  
      Describes validation and computation logic with example success and error outputs.  
    - **Sub-Workflow:** None.

#### 1.4 Response Delivery

- **Overview:**  
  Sends the computed centroid or error message back to the client as the HTTP response.

- **Nodes Involved:**  
  - Return Centroid Response (Respond to Webhook Node)

- **Node Details:**

  - **Return Centroid Response**  
    - **Type & Role:** Respond to Webhook node; returns HTTP response to the original request.  
    - **Configuration:**  
      - No special parameters; returns whatever JSON it receives.  
    - **Input/Output:**  
      - Input from "Validate & Compute Centroid".  
      - No output nodes (terminal node).  
    - **Version Requirements:** Respond to Webhook node version 1.1.  
    - **Potential Failures:**  
      - If input JSON is malformed, response may be invalid.  
      - Network or client connection issues.  
    - **Sticky Note Context:**  
      Explains that it returns the final response with centroid or error message.  
    - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                     | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                          |
|-------------------------|------------------------|-----------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Receive Vectors         | Webhook                | Entry point; receives GET request | None                  | Extract & Parse Vectors   | Acts as the entry point receiving GET request with `vectors` parameter in JSON format. Example URL provided. |
| Extract & Parse Vectors | Set                    | Parses and converts input vectors | Receive Vectors       | Validate & Compute Centroid | Extracts `vectors` array from GET request, ensures valid array. Shows expected output example.      |
| Validate & Compute Centroid | Code                | Validates vectors and computes centroid | Extract & Parse Vectors | Return Centroid Response  | Performs vector dimension validation and centroid calculation. Shows success and error output examples. |
| Return Centroid Response | Respond to Webhook     | Sends final response to client    | Validate & Compute Centroid | None                    | Returns computed centroid or error message to client. Example response shown.                      |
| Sticky Note             | Sticky Note            | Documentation note                | None                  | None                      | Describes the role and expected behavior of "Extract & Parse Vectors" node.                        |
| Sticky Note1            | Sticky Note            | Documentation note                | None                  | None                      | Describes the role and behavior of "Validate & Compute Centroid" node.                            |
| Sticky Note2            | Sticky Note            | Documentation note                | None                  | None                      | Describes the role and behavior of "Return Centroid Response" node.                              |
| Sticky Note4            | Sticky Note            | Documentation note                | None                  | None                      | Describes the role and behavior of "Receive Vectors" node.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node** named `Receive Vectors`:  
   - Set HTTP Method to `GET`.  
   - Set Path to `centroid`.  
   - Set Response Mode to `Response Node` (to delegate response to another node).  
   - Save.

3. **Add a Set node** named `Extract & Parse Vectors`:  
   - Add a new field assignment:  
     - Name: `vectors`  
     - Type: Array  
     - Value: Expression `={{ $json.query.vectors }}`  
   - This extracts the `vectors` query parameter from the webhook input and converts it to an array.  
   - Connect `Receive Vectors` node output to this node input.

4. **Add a Code node** named `Validate & Compute Centroid`:  
   - Set Language to JavaScript.  
   - Paste the following script:
     ```javascript
     const input = items[0].json;
     const vectors = input.vectors;

     if (!Array.isArray(vectors) || vectors.length === 0) {
       return [{ json: { error: "Invalid input: Expected an array of vectors." } }];
     }

     const dimension = vectors[0].length;
     if (!vectors.every(v => v.length === dimension)) {
       return [{ json: { error: "Vectors have inconsistent dimensions." } }];
     }

     const centroid = new Array(dimension).fill(0);
     vectors.forEach(vector => {
       vector.forEach((val, index) => {
         centroid[index] += val;
       });
     });

     for (let i = 0; i < dimension; i++) {
       centroid[i] /= vectors.length;
     }

     return [{ json: { centroid } }];
     ```
   - Connect `Extract & Parse Vectors` node output to this node input.

5. **Add a Respond to Webhook node** named `Return Centroid Response`:  
   - No special parameters needed; it will respond with the JSON it receives.  
   - Connect `Validate & Compute Centroid` node output to this node input.

6. **Activate the workflow** and test by sending a GET request to:  
   ```
   https://<your-n8n-domain>/webhook/centroid?vectors=[[2,3,4],[4,5,6],[6,7,8]]
   ```
   - The response should be:  
     ```json
     {
       "centroid": [4,5,6]
     }
     ```
7. **Error Testing:**  
   - Send malformed or inconsistent vectors to verify error handling, e.g.:  
     ```
     ?vectors=[[1,2],[3,4,5]]
     ```
   - Expected error response:  
     ```json
     {
       "error": "Vectors have inconsistent dimensions."
     }
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed to be reusable and easily integrated into projects requiring vector centroid calculations. | General workflow purpose.                                                                       |
| Example GET request URL: `https://actions.singular-innovation.com/webhook-test/centroid?vectors=[[2,3,4],[4,5,6],[6,7,8]]` | Example usage for testing.                                                                      |
| The centroid calculation assumes numerical vectors and consistent dimensions; non-numeric or irregular inputs will cause errors. | Important input validation note.                                                                |
| Use tools like Postman or n8n's built-in webhook tester to send requests and verify outputs.          | Testing recommendation.                                                                         |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the "Calculate the Centroid of a Set of Vectors" n8n workflow.