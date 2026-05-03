Recover Lost Binary Data Between Workflow Nodes with Code Node Pattern

https://n8nworkflows.xyz/workflows/recover-lost-binary-data-between-workflow-nodes-with-code-node-pattern-8780


# Recover Lost Binary Data Between Workflow Nodes with Code Node Pattern

### 1. Workflow Overview

This workflow demonstrates how to recover lost binary data between workflow nodes using a Code node pattern in n8n. It targets scenarios where binary files (such as images or PDFs) are fetched and processed across multiple steps, but intermediate nodes—like the Set node—do not preserve binary data by default. The workflow is designed to:

- Fetch a binary file (an image of the n8n logo) from a URL.
- Simulate the loss of binary data using a Set node that removes binary content while adding new data fields.
- Re-access and re-attach the original binary data from an earlier node despite its removal in the intermediate step.

The logical blocks of the workflow are:

- **1.1 Input Reception & Binary Data Fetch:** Get binary data (image) from an HTTP source.
- **1.2 Binary Data Removal Simulation:** Use a Set node to remove binary data and add new fields.
- **1.3 Binary Data Recovery:** Use a Code node to retrieve and re-attach binary data from a previous node despite removal.
- **1.4 Workflow Start & Documentation:** Manual trigger to initiate the workflow and sticky notes providing explanations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Binary Data Fetch

- **Overview:**  
  This block fetches an image (n8n logo) as binary data from a remote URL. It acts as the original source of the binary content used in the workflow.

- **Nodes Involved:**  
  - Get n8n Logo (Binary)

- **Node Details:**

  - **Get n8n Logo (Binary)**  
    - Type: HTTP Request node  
    - Role: Downloads the n8n logo image as a binary file from Wikimedia Commons.  
    - Configuration:  
      - URL set to the n8n logo PNG image link.  
      - No special HTTP options enabled.  
    - Input: Receives trigger from the Start node.  
    - Output: Emits item(s) containing the binary image data.  
    - Edge Cases:  
      - Network errors or URL unavailability may cause HTTP request failures.  
      - Large file sizes could impact performance but unlikely here due to image size.  
    - Notes: Contains a sticky note explaining it is the binary data source.

#### 1.2 Binary Data Removal Simulation

- **Overview:**  
  This block simulates the loss of binary data by using a Set node that overwrites the item’s data without preserving the binary property, mimicking common scenarios where binary data gets dropped.

- **Nodes Involved:**  
  - Remove Binary Data

- **Node Details:**

  - **Remove Binary Data**  
    - Type: Set node  
    - Role: Adds a new string field (`new_field` with value "test value") and does not preserve binary data.  
    - Configuration:  
      - Field `new_field` assigned a static string "test value".  
      - No "Keep Binary Data" option enabled.  
    - Input: Receives the binary data item from the HTTP Request node.  
    - Output: Emits items that *do not* contain the binary data anymore.  
    - Edge Cases:  
      - Loss of binary data means downstream nodes cannot access it unless recovered.  
    - Notes: Sticky note highlights that many n8n nodes behave this way by default.

#### 1.3 Binary Data Recovery

- **Overview:**  
  This critical block shows how to recover the lost binary data by referencing an earlier node's output directly, then re-attaching that binary data to the current item using helper methods.

- **Nodes Involved:**  
  - Re-Access Binary Data from Previous Node

- **Node Details:**

  - **Re-Access Binary Data from Previous Node**  
    - Type: Code node (JavaScript)  
    - Role: Re-attaches binary files from the specified previous node (`Get n8n Logo (Binary)`) to the current items, circumventing the loss caused by intermediate nodes.  
    - Configuration:  
      - The variable `previousNodeName` is set to `'Get n8n Logo (Binary)'`.  
      - Iterates over all incoming items, accesses the full item data from the previous node via `$(previousNodeName).item`.  
      - Checks for presence of binary data in the referenced item.  
      - For each binary file key, decodes base64 data to buffer, then prepares the binary data correctly using `this.helpers.prepareBinaryData()`.  
      - Attaches recovered binary data to the current item without overwriting existing binary fields.  
    - Input: Receives items without binary data from the Set node.  
    - Output: Emits items with binary data restored.  
    - Version Requirements: Uses n8n helpers available in recent versions (v0.153+ recommended).  
    - Edge Cases:  
      - If the referenced node name is incorrect or missing, no binary data is recovered.  
      - Binary data structure changes upstream may cause failures.  
      - Large binary files may slow down processing.  
    - Notes: Sticky note details the core concept and customization instructions.

#### 1.4 Workflow Start & Documentation

- **Overview:**  
  This block contains the manual trigger to start the workflow and sticky notes with detailed explanations, instructions, usage notes, and contact links from the creator.

- **Nodes Involved:**  
  - Start  
  - Sticky Note (0)  
  - Sticky Note (1)  
  - Sticky Note (2)  
  - Sticky Note (3)

- **Node Details:**

  - **Start**  
    - Type: Manual Trigger node  
    - Role: Entry point to trigger the workflow execution manually.  
    - Configuration: Default; no parameters needed.  
    - Output: Starts the chain by triggering the HTTP Request node.  

  - **Sticky Note (0)**  
    - Type: Sticky Note  
    - Role: Provides an overview, usage instructions, and contact info.  
    - Content Highlights:  
      - Explains the ability to re-access binary data despite intermediate node losses.  
      - Provides steps to run the workflow and how to reach out for support.  
      - Contains a link: [Get in Touch Here](https://api.ia2s.app/form/templates/academy).

  - **Sticky Note (1)**  
    - Provides explanation about the HTTP Request node fetching the binary image.

  - **Sticky Note (2)**  
    - Explains the Set node behavior removing binary data and why this scenario is important.

  - **Sticky Note (3)**  
    - Details how the Code node works to re-access binary data and customization tips.

---

### 3. Summary Table

| Node Name                             | Node Type          | Functional Role                       | Input Node(s)             | Output Node(s)                        | Sticky Note                                                                                             |
|-------------------------------------|--------------------|------------------------------------|---------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------|
| Start                               | Manual Trigger     | Workflow entry point                | —                         | Get n8n Logo (Binary)                |                                                                                                       |
| Get n8n Logo (Binary)                | HTTP Request       | Fetch binary image file             | Start                     | Remove Binary Data                   | This node fetches a binary image (the n8n logo) from a URL. It's the source of the binary data we'll be working with. |
| Remove Binary Data                   | Set                | Simulate binary data removal        | Get n8n Logo (Binary)      | Re-Access Binary Data from Previous Node | ⚠️ Important Concept: many n8n nodes like Set do not pass binary data to next nodes by default.         |
| Re-Access Binary Data from Previous Node | Code (JavaScript)   | Recover lost binary data from earlier node | Remove Binary Data        | —                                   | The magic happens here! Re-access binary data using `$(previousNodeName).item` and `this.helpers.prepareBinaryData()`. Customize `previousNodeName` as needed. |
| Sticky Note (0)                     | Sticky Note        | Documentation and instructions     | —                         | —                                   | Re-Access Binary Data explanation and contact form link: https://api.ia2s.app/form/templates/academy    |
| Sticky Note (1)                     | Sticky Note        | Explanation of HTTP Request node   | —                         | —                                   |                                                                                                       |
| Sticky Note (2)                     | Sticky Note        | Explanation of binary data removal | —                         | —                                   |                                                                                                       |
| Sticky Note (3)                     | Sticky Note        | Explanation of Code node recovery  | —                         | —                                   |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node named `Start`:**  
   - Default configuration; no parameters.  
   - This node will be the entry point.

2. **Create an HTTP Request node named `Get n8n Logo (Binary)`:**  
   - Connect the output of `Start` to this node.  
   - Set the HTTP Method to GET (default).  
   - Set the URL to:  
     `https://upload.wikimedia.org/wikipedia/commons/thumb/5/53/N8n-logo-new.svg/2560px-N8n-logo-new.svg.png`  
   - Ensure no special options are enabled (default).

3. **Create a Set node named `Remove Binary Data`:**  
   - Connect output of `Get n8n Logo (Binary)` to this node.  
   - Add a new field assignment:  
     - Name: `new_field`  
     - Type: String  
     - Value: `test value`  
   - Do **not** enable the "Keep Binary Data" option (default behavior removes binary data).

4. **Create a Code node named `Re-Access Binary Data from Previous Node`:**  
   - Connect the output of `Remove Binary Data` to this node.  
   - Set the language to JavaScript and use the following logic (adapted for your node name as needed):

```javascript
// IMPORTANT: Set this to the exact name of the node producing binary data
const previousNodeName = 'Get n8n Logo (Binary)';

const allItems = [];

for (const item of $input.all()) {

  // Access the full item from the previous node by name
  const previousNodeItem = $(previousNodeName).item;

  if (previousNodeItem && previousNodeItem.binary) {
    item.binary = item.binary || {};

    for (const key in previousNodeItem.binary) {
      const binary = previousNodeItem.binary[key];
      const myBuffer = Buffer.from(binary.data, 'base64');
      item.binary[key] = await this.helpers.prepareBinaryData(myBuffer, binary.fileName);
    }
  }

  allItems.push(item);
}

return allItems;
```

5. **Add Sticky Notes for Documentation (optional but recommended):**  
   - Add sticky notes near the respective nodes with explanations:  
     - Workflow overview and contact info.  
     - Explanation of HTTP Request as binary source.  
     - Explanation of the Set node removing binary data.  
     - Explanation of the Code node to recover binary data.

6. **Run the Workflow:**  
   - Trigger manually from `Start`.  
   - Observe the binary data is preserved and re-attached after the Set node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                |
|----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| This workflow demonstrates a key n8n automation technique: recovering binary data lost between nodes, essential for robust file workflows. | Workflow purpose and background                                |
| Contact and feedback form with AI-powered support: [Get in Touch Here](https://api.ia2s.app/form/templates/academy)                    | Support and coaching link                                      |
| Many n8n nodes (like Set) do not preserve binary data by default, so this pattern is crucial for workflows manipulating files.         | Important concept for users working with binary data           |
| Use the Code node helper `this.helpers.prepareBinaryData()` to correctly attach binary data for downstream nodes.                      | Technical implementation detail                                |

---

This documentation provides a complete, self-contained reference for understanding, reproducing, and extending the "Recover Lost Binary Data Between Workflow Nodes with Code Node Pattern" n8n workflow created by Lucas Peyrin.