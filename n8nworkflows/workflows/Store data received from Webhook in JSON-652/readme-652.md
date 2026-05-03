Store data received from Webhook in JSON

https://n8nworkflows.xyz/workflows/store-data-received-from-webhook-in-json-652


# Store data received from Webhook in JSON

### 1. Workflow Overview

This workflow is designed to retrieve a random cocktail recipe from the CocktailDB API and save the received data as a JSON file locally. It targets use cases where automated data collection and storage from public APIs are required, such as data archiving, offline analysis, or integration with other systems.

The workflow includes the following logical blocks:

- **1.1 Trigger Initiation:** Manual start of the workflow.
- **1.2 Data Retrieval:** Fetch a random cocktail recipe JSON from the CocktailDB API.
- **1.3 Data Conversion:** Convert retrieved JSON data into binary format for file writing.
- **1.4 Data Storage:** Write the binary data to a local JSON file named `cocktail.json`.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Initiation

- **Overview:** Initiates the workflow manually by user interaction.
- **Nodes Involved:**  
  - `On clicking 'execute'`

- **Node Details:**

  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Role:** Entry point to start the workflow on demand.  
  - **Configuration:** No parameters needed; triggers when clicked in the n8n editor or via API call.  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connects to `HTTP Request` node  
  - **Version-specific Notes:** Compatible with all n8n versions supporting manual trigger nodes.  
  - **Potential Failures:** None expected; manual trigger is straightforward.  
  - **Sub-workflows:** None

#### 1.2 Data Retrieval

- **Overview:** Sends an HTTP GET request to the CocktailDB API to fetch a random cocktail recipe in JSON format.
- **Nodes Involved:**  
  - `HTTP Request`

- **Node Details:**

  - **Node Name:** HTTP Request  
  - **Type:** HTTP Request  
  - **Role:** Connects to external API, retrieves JSON data.  
  - **Configuration:**  
    - URL: `https://www.thecocktaildb.com/api/json/v1/1/random.php`  
    - Method: GET (default)  
    - No additional headers or authentication required (public API).  
  - **Expressions/Variables:** None used; static URL.  
  - **Inputs:** From `On clicking 'execute'`  
  - **Outputs:** Sends JSON response to `Move Binary Data` node  
  - **Version-specific Notes:** Ensure n8n supports TLS connections and HTTP request node version 1 or higher.  
  - **Potential Failures:**  
    - Network connectivity issues  
    - API downtime or rate limiting  
    - Invalid or malformed JSON response (unlikely with this API)  
  - **Sub-workflows:** None

#### 1.3 Data Conversion

- **Overview:** Converts the JSON data from the HTTP response into binary format, suitable for file writing.
- **Nodes Involved:**  
  - `Move Binary Data`

- **Node Details:**

  - **Node Name:** Move Binary Data  
  - **Type:** Move Binary Data  
  - **Role:** Transforms JSON output into binary data field for the next node.  
  - **Configuration:**  
    - Mode: `jsonToBinary` – converts JSON body to binary data for file output.  
    - Options: default (no special options set).  
  - **Expressions/Variables:** None.  
  - **Inputs:** From `HTTP Request`  
  - **Outputs:** Sends binary data to `Write Binary File` node  
  - **Version-specific Notes:** Requires n8n version that supports binary data manipulation (generally version 0.100+).  
  - **Potential Failures:**  
    - Failure if input JSON is empty or malformed (very unlikely here)  
    - Expression evaluation errors if JSON structure is unexpected  
  - **Sub-workflows:** None

#### 1.4 Data Storage

- **Overview:** Writes the binary data into a file named `cocktail.json` in the local file system.
- **Nodes Involved:**  
  - `Write Binary File`

- **Node Details:**

  - **Node Name:** Write Binary File  
  - **Type:** Write Binary File  
  - **Role:** Persist binary data into a local file.  
  - **Configuration:**  
    - File Name: `cocktail.json` (static filename)  
    - Path: Default local execution directory unless otherwise specified in the environment.  
  - **Expressions/Variables:** None.  
  - **Inputs:** From `Move Binary Data`  
  - **Outputs:** None (terminal node)  
  - **Version-specific Notes:** Requires write permissions to the local file system where n8n is running.  
  - **Potential Failures:**  
    - File system permission errors  
    - Disk space issues  
    - Path not writable or invalid  
  - **Sub-workflows:** None

---

### 3. Summary Table

| Node Name              | Node Type         | Functional Role          | Input Node(s)          | Output Node(s)         | Sticky Note                      |
|------------------------|-------------------|-------------------------|-----------------------|------------------------|---------------------------------|
| On clicking 'execute'   | Manual Trigger    | Workflow initiation     | -                     | HTTP Request           |                                 |
| HTTP Request           | HTTP Request      | Fetch cocktail JSON      | On clicking 'execute' | Move Binary Data       |                                 |
| Move Binary Data       | Move Binary Data  | Convert JSON to binary   | HTTP Request          | Write Binary File      |                                 |
| Write Binary File      | Write Binary File | Save data to local file  | Move Binary Data      | -                      |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n with the name: *Store the data received from the CocktailDB API in JSON*.

2. **Add a Manual Trigger node:**
   - Name it `On clicking 'execute'`.
   - No parameters need to be set.
   - This node acts as the starting point.

3. **Add an HTTP Request node:**
   - Name it `HTTP Request`.
   - Set the **URL** to: `https://www.thecocktaildb.com/api/json/v1/1/random.php`
   - Method: GET (default)
   - No authentication or headers are required.
   - Connect the output of `On clicking 'execute'` to the input of this node.

4. **Add a Move Binary Data node:**
   - Name it `Move Binary Data`.
   - Set **Mode** to `jsonToBinary`.
   - Leave options as default.
   - Connect the output of `HTTP Request` to the input of this node.

5. **Add a Write Binary File node:**
   - Name it `Write Binary File`.
   - Set **File Name** to `cocktail.json`.
   - Ensure that the n8n execution environment has write permissions to the target directory.
   - Connect the output of `Move Binary Data` to this node.
   - No further parameters required unless a specific path is desired (then configure accordingly).

6. **Activate the workflow** (optional) and run manually using the manual trigger to test.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                    |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| The CocktailDB API is a free and public API for cocktail recipes and details.                    | https://www.thecocktaildb.com/api.php                             |
| Writing files requires the n8n instance to have appropriate file system permissions.             | n8n documentation on file system nodes: https://docs.n8n.io/nodes/n8n-nodes-base.writeBinaryFile/ |
| Manual Trigger nodes are useful for testing and manual execution during development.             |                                                                   |

---

This documentation facilitates understanding, modification, and reproduction of the workflow, while highlighting potential error sources and system prerequisites.