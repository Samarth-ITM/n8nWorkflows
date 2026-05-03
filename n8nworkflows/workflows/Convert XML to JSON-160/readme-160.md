Convert XML to JSON

https://n8nworkflows.xyz/workflows/convert-xml-to-json-160


# Convert XML to JSON

### 1. Workflow Overview

This workflow is designed to convert XML data into JSON format while preserving XML attributes in a dedicated key. It targets use cases where XML inputs need to be transformed into JSON for easier processing downstream, ensuring that attribute data is not lost but instead stored under a separate key for clarity and structured access.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow and provide the XML data.
- **1.2 XML to JSON Conversion:** Transformation of the XML string into a JSON object, explicitly preserving XML attributes under a designated key.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and sets the XML data to be processed. It allows the user to start the conversion process on-demand and inject the XML content as a string variable.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set

- **Node Details:**

  - **On clicking 'execute'**  
    - Type and Role: Manual Trigger node; starts workflow execution manually.  
    - Configuration: No parameters needed; simply provides a trigger event.  
    - Inputs: None (trigger node).  
    - Outputs: Connected to the 'Set' node.  
    - Potential Failures: None, except user not triggering execution.  
    - Version: Standard trigger available in n8n v0.100+.

  - **Set**  
    - Type and Role: Sets the XML data string that will be converted.  
    - Configuration:  
      - Sets a single string field named `xml` with the sample XML content:  
        ```xml
        <?xml version="1.0" encoding="utf-8"?> 
        <ORDERS05>   
          <IDOC BEGIN="1">     
            <EDI_DC40 SEGMENT="1">       
              <TABNAM>EDI_DC40</TABNAM>     
            </EDI_DC40>   
          </IDOC> 
        </ORDERS05>
        ```  
      - `keepOnlySet` is enabled, meaning only this field is passed forward.  
    - Inputs: Trigger node output.  
    - Outputs: Connects to the 'XML' node.  
    - Potential Failures: None expected unless manual changes introduce invalid XML.  
    - Version: No specific requirements.

#### 1.2 XML to JSON Conversion

- **Overview:**  
  Converts the XML string provided by the previous block into a JSON object. It explicitly separates XML attributes into a key named `$` without merging them into parent elements, and keeps the root element explicit.

- **Nodes Involved:**  
  - XML

- **Node Details:**

  - **XML**  
    - Type and Role: XML node; performs XML parsing and conversion to JSON.  
    - Configuration:  
      - `dataPropertyName` set to `xml` to specify which field contains the XML string.  
      - Options:  
        - `attrkey` set to `$` — XML attributes are stored under this key in resulting JSON.  
        - `mergeAttrs` set to `false` — prevents merging attributes into parent elements.  
        - `explicitRoot` set to `true` — keeps the root XML node explicitly in JSON output.  
    - Inputs: Receives JSON with `xml` string from the 'Set' node.  
    - Outputs: JSON object with XML data converted, preserving attributes under `$`.  
    - Potential Failures:  
      - XML parsing errors if input XML is malformed.  
      - Large XML payloads could cause performance/timeouts.  
      - Misconfiguration of `dataPropertyName` causing node to not find input XML.  
    - Version: Available in n8n since v0.100+.

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role         | Input Node(s)          | Output Node(s)         | Sticky Note                                                      |
|------------------------|-------------------------|------------------------|------------------------|------------------------|-----------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger          | Start workflow         | -                      | Set                    |                                                                 |
| Set                    | Set                     | Provide XML data       | On clicking 'execute'   | XML                    |                                                                 |
| XML                    | XML                     | Convert XML to JSON    | Set                    | -                      | Attributes stored under `$`; merging attributes disabled; root explicit |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Add a "Manual Trigger" node to serve as the workflow start point.
   - No configuration needed.
   - Position it at the start.

2. **Create Set Node:**
   - Add a "Set" node.
   - Connect the output of the Manual Trigger node to this Set node.
   - Configure the Set node to:
     - Enable "Keep Only Set".
     - Add a new field:
       - Type: String
       - Name: `xml`
       - Value: Paste the XML string:
         ```xml
         <?xml version="1.0" encoding="utf-8"?> 
         <ORDERS05>   
           <IDOC BEGIN="1">     
             <EDI_DC40 SEGMENT="1">       
               <TABNAM>EDI_DC40</TABNAM>     
             </EDI_DC40>   
           </IDOC> 
         </ORDERS05>
         ```

3. **Create XML Node:**
   - Add an "XML" node.
   - Connect the output of the Set node to this XML node.
   - Configure the XML node:
     - Set `Data Property Name` to `xml` (matches the field set in the Set node).
     - Under Options:
       - `Attribute Key` = `$`
       - `Merge Attributes` = false (do not merge attributes into parent nodes)
       - `Explicit Root` = true (keep root node in JSON output)

4. **Save and Activate Workflow:**
   - Save the workflow.
   - Optionally activate or run manually by clicking "Execute Workflow".

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                   |
|------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The XML node options ensure attributes are preserved under the `$` key as per n8n standard.    | n8n XML node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.xml/ |
| This workflow is useful when downstream processes require JSON with clear attribute separation. | Typical use: API integrations, data transformation pipelines.    |

---

This documentation fully describes the workflow "Convert XML to JSON" for professional use and modification.