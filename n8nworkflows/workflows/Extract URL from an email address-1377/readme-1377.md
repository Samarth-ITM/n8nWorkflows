Extract URL from an email address

https://n8nworkflows.xyz/workflows/extract-url-from-an-email-address-1377


# Extract URL from an email address

### 1. Workflow Overview

This workflow is designed to extract the domain part from an email address. It focuses on parsing a given email string to isolate and output the domain (the substring after the "@" symbol). The workflow is straightforward and suitable for use cases where domain extraction from emails is required, such as data normalization, validation, or categorization.

The logical blocks included are:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Email Preparation:** Setting a sample email address for processing.
- **1.3 Domain Extraction:** Parsing the email to extract the domain name.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow on manual execution, acting as the entry point.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually.  
    - Configuration: Default settings; no parameters required.  
    - Expressions: None.  
    - Input: None (workflow start).  
    - Output: Triggers the next node ('Sample email').  
    - Edge Cases: None, unless the manual trigger is not executed.  

#### 1.2 Email Preparation

- **Overview:**  
  Prepares a sample email address by setting a JSON variable to be used for domain extraction.

- **Nodes Involved:**  
  - Sample email

- **Node Details:**  

  - **Sample email**  
    - Type: Set  
    - Role: Defines the input email address for the workflow.  
    - Configuration: Sets a single string field `email` with the value `"email@domain2.com"`.  
    - Expressions: None; static value.  
    - Input: Receives trigger from 'On clicking execute'.  
    - Output: Passes JSON with the email field to 'Extract domain name'.  
    - Edge Cases: If the email is malformed or missing, the next node may fail or return invalid results.  

#### 1.3 Domain Extraction

- **Overview:**  
  Extracts the domain portion from the email string by splitting at the "@" character.

- **Nodes Involved:**  
  - Extract domain name

- **Node Details:**  

  - **Extract domain name**  
    - Type: Function  
    - Role: Processes the input email string to isolate the domain.  
    - Configuration: JavaScript code snippet that:  
      - Reads the `email` field from input JSON.  
      - Extracts the substring after the last "@" character as the domain.  
      - Returns JSON containing only the `domain` field.  
    - Key Expression:  
      ```js
      var email = $json["email"];
      var domain = email.substring(email.lastIndexOf("@") + 1);
      return [{ json: { domain } }];
      ```  
    - Input: Receives JSON with `email` from 'Sample email'.  
    - Output: JSON with `domain` field (e.g., "domain2.com").  
    - Edge Cases:  
      - If the email string does not contain "@", `lastIndexOf("@")` returns -1, possibly causing incorrect substring extraction (resulting in the entire string or empty domain).  
      - No validation on email format; malformed emails may yield incorrect domains.  
      - No error handling for empty or null email values.  
    - Version Requirements: Compatible with n8n version supporting Function node (standard feature).  
    - Sub-workflow: None.  

---

### 3. Summary Table

| Node Name            | Node Type       | Functional Role         | Input Node(s)        | Output Node(s)          | Sticky Note                   |
|----------------------|-----------------|------------------------|----------------------|-------------------------|-------------------------------|
| On clicking 'execute' | Manual Trigger  | Workflow start trigger | None                 | Sample email            |                               |
| Sample email         | Set             | Defines sample email   | On clicking 'execute' | Extract domain name     |                               |
| Extract domain name  | Function        | Extracts domain part   | Sample email          | None                    |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No additional configuration needed.

2. **Create a Set node:**  
   - Name: `Sample email`  
   - Type: Set  
   - Configure to keep only set fields.  
   - Add a string field named `email` with value `email@domain2.com`.  
   - Connect output of `On clicking 'execute'` to input of `Sample email`.

3. **Create a Function node:**  
   - Name: `Extract domain name`  
   - Type: Function  
   - Paste the following JavaScript code into the function editor:  
     ```js
     var email = $json["email"];
     var domain = email.substring(email.lastIndexOf("@") + 1);
     return [{ json: { domain } }];
     ```  
   - Connect output of `Sample email` to input of `Extract domain name`.

4. **Save and execute:**  
   - Trigger the workflow by clicking "Execute" on the manual trigger node.  
   - The final output will contain the extracted domain from the input email.

5. **Credentials:**  
   - No credentials required for this workflow.

---

### 5. General Notes & Resources

| Note Content                              | Context or Link                |
|-------------------------------------------|-------------------------------|
| Workflow is a simple demonstration of string manipulation in n8n to extract email domains. | Useful for preprocessing or filtering email data. |
| Ensure email inputs are valid to avoid incorrect extraction results. | Validation can be added in future enhancements. |