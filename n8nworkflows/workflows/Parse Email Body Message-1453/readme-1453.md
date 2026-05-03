Parse Email Body Message

https://n8nworkflows.xyz/workflows/parse-email-body-message-1453


# Parse Email Body Message

### 1. Workflow Overview

This workflow, titled **"Email body parser by aprenden8n.com"**, is designed to parse structured email body content received from user form submissions on websites. Its main purpose is to extract key-value pairs from the email's body text based on predefined labels, enabling easy access to user-submitted data without manual processing or complex parsing logic.

**Target Use Cases:**  
- Automating data extraction from standardized email notifications triggered by form submissions.  
- Feeding parsed data into downstream systems for CRM, marketing automation, or support ticket generation.  
- Serving as a reusable snippet or sub-workflow for any email body parsing task that follows a label:value structure.

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger initiates the workflow for testing or integration.  
- **1.2 Data Preparation:** Setting the raw email body and the list of labels to parse.  
- **1.3 Parsing Logic:** Processing the email body using a JavaScript function to extract labeled values.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block provides a manual trigger node to start the workflow execution. It is intended for manual testing or integration points where an explicit trigger is required.

**Nodes Involved:**  
- On clicking 'execute'

**Node Details:**  
- **Type and Role:** Manual Trigger node; initiates workflow execution on user action.  
- **Configuration:** No specific parameters set; default manual trigger behavior.  
- **Expressions/Variables:** None.  
- **Input/Output Connections:** Outputs to "Set values" node.  
- **Version Requirements:** Compatible with n8n v0.152.0+ (Manual Trigger is standard).  
- **Potential Failures:** None expected; user must manually trigger execution.  
- **Sub-workflow:** None.

---

#### 1.2 Data Preparation

**Overview:**  
This block sets the input data necessary for parsing: the raw email body text and the list of labels to extract. It acts as a container for variables that can be dynamically replaced or set as expressions in real scenarios.

**Nodes Involved:**  
- Set values

**Node Details:**  
- **Type and Role:** Set node; used to define static or dynamic variables for the workflow.  
- **Configuration Choices:**  
  - Defines two string fields:  
    - `body`: Contains the multiline string representing the email body.  
    - `labels`: A comma-separated string of labels to detect and parse (case insensitive).  
- **Expressions/Variables:** Values are hardcoded but can be replaced with expressions referencing previous nodes or external inputs.  
- **Input/Output Connections:** Receives input from "On clicking 'execute'", outputs to "Email Parser Snippet".  
- **Version Requirements:** Standard Set node, no special requirements.  
- **Potential Failures:** Misconfiguration of labels or body will lead to incomplete or failed parsing downstream.  
- **Sub-workflow:** None.

---

#### 1.3 Parsing Logic

**Overview:**  
This block contains the core parsing logic using a Function Item node. It processes the email body and extracts values for each label defined, returning an object where keys are labels and values are parsed content.

**Nodes Involved:**  
- Email Parser Snippet

**Node Details:**  
- **Type and Role:** Function Item node; executes JavaScript code on each item individually.  
- **Configuration Choices:**  
  - The JavaScript code splits the provided labels string into an array.  
  - For each label, it constructs a RegExp that matches the label followed by a colon or space and captures the corresponding value.  
  - Special handling for the last label assumes it may contain multiline values until the end of the string.  
  - The matching is case-insensitive.  
  - The result is an object with each label as a key and the extracted trimmed string as value.  
- **Key Expressions/Variables:**  
  - `item.labels` (input labels string)  
  - `item.body` (email body string)  
  - RegExp dynamically built per label for extraction.  
- **Input/Output Connections:** Receives input from "Set values", outputs parsed object as a new item.  
- **Version Requirements:** Requires n8n version supporting Function Item node (available from n8n v0.153.0+).  
- **Edge Cases or Failures:**  
  - If labels do not match exactly or case-insensitively, values will be missing.  
  - Multiline values are supported only for the last label; values spanning multiple lines for other labels will not parse fully.  
  - If the email body structure differs significantly, parsing fails.  
  - Regular expression failures if special characters appear in labels without escape.  
- **Sub-workflow:** Can be extracted as an independent snippet to be called from other workflows, accepting `body` and `labels` as parameters.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                 | Input Node(s)        | Output Node(s)        | Sticky Note                                                                                                                |
|----------------------|--------------------|--------------------------------|----------------------|-----------------------|----------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger     | Initiates workflow execution    |                      | Set values            |                                                                                                                            |
| Set values           | Set                | Defines email body and labels   | On clicking 'execute' | Email Parser Snippet  | Define `body` as the email content and `labels` as comma-separated keys to parse; replace with expressions as needed.       |
| Email Parser Snippet | Function Item      | Parses email body into key-value pairs | Set values           |                       | Label detection is case insensitive; multiline values are supported only for the last label. Can be reused or called generically. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - No parameters needed. This node starts the workflow manually.

2. **Add a Set node**  
   - Name: `Set values`  
   - Connect the output of `On clicking 'execute'` to this node.  
   - Add two string fields:  
     - `body`: Paste or set the email body content here. For example:  
       ```
       Name: Miquel
       Email: miquel@aprenden8n.com
       Subject: Welcome aboard
       Message: Hi Miquel!

       Thank you for your signup!
       ```  
     - `labels`: Enter the labels you want to parse, comma-separated (case insensitive). For example: `Name,Email,Subject,Message`

3. **Add a Function Item node**  
   - Name: `Email Parser Snippet`  
   - Connect the output of `Set values` to this node.  
   - Paste the following JavaScript code into the Function Item node’s code editor:
     ```javascript
     var obj = {};
     var labels = item.labels.split(",");
     item.labels.split(",").forEach(function(label) {
       var re = labels.indexOf(label) === labels.length - 1 ? "\\b" + label + "\\b[: ]+([^$]+)" : "\\b" + label + "\\b[: ]+([^\\n$]+)";
       var found = item.body.match(new RegExp(re, "i"));
       if (found && found.length > 1) {
         obj[label] = found[1].trim();
       }
     });

     return obj;
     ```
   - This code extracts values for each label from the body, handling multiline values only for the last label.

4. **Activate and test the workflow**  
   - Trigger manually via the `On clicking 'execute'` node.  
   - Inspect output of `Email Parser Snippet` to verify parsed data is correct.

**Note:**  
- To use this snippet in other workflows, pass `body` and `labels` as input parameters to this snippet node or create a sub-workflow with these inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| We are **Aprende n8n**, the first n8n Spanish course for all n8n lovers.                                                    | https://aprenden8n.com                                                                           |
| The snippet can parse any email body with labeled fields separated by colons or spaces.                                    | Documentation and usage instructions included in the workflow description.                       |
| For help or comments, contact: miquel@aprenden8n.com                                                                       |                                                                                                 |
| Join the Spanish n8n community for support and learning.                                                                    | https://t.me/comunidadn8n                                                                        |
| Limitations: The parser only supports multiline values for the last label in the email body.                               | Important to consider for email templates with multiline data.                                  |
| Recommended to integrate this snippet into existing workflows and adapt the `body` and `labels` fields dynamically.       | Allows flexible and reusable parsing in diverse scenarios.                                      |

---

This documentation should enable advanced users and AI agents to fully understand, recreate, and modify the "Email body parser by aprenden8n.com" workflow with confidence.