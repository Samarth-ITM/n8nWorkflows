Extract Domain and verify email syntax on the go

https://n8nworkflows.xyz/workflows/extract-domain-and-verify-email-syntax-on-the-go-2239


# Extract Domain and verify email syntax on the go

### 1. Workflow Overview

This workflow is designed to assist email marketing professionals with two key tasks: extracting the domain part of an email address and validating the email syntax without requiring custom code. It leverages n8n's built-in string functions to perform these operations in a straightforward, no-code manner.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the starting trigger for the workflow and provides sample data for demonstration, which should be replaced in production with the user's actual email data source.
- **1.2 Email Processing:** Processes each email by extracting the domain and validating the email syntax using native n8n JSON functions.
- **1.3 Output Preparation:** Prepares the processed data with the original email, its extracted domain, and a boolean indicating validity.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and provides sample email data for demonstration purposes. It is designed to be replaced with a real data source in production.

- **Nodes Involved:**  
  - When clicking "Test workflow" (Manual Trigger)  
  - Generate random data (Debug Helper)  
  - Sticky Note1 (Instructional note)

- **Node Details:**

  - **When clicking "Test workflow"**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually  
    - Configuration: Default, no parameters set  
    - Inputs: None  
    - Outputs: Triggers "Generate random data" node  
    - Edge Cases: None, as it is manual; workflow will not run without manual trigger  
    - Sub-workflow: None

  - **Generate random data**  
    - Type: Debug Helper  
    - Role: Provides sample data with preset emails for testing  
    - Configuration: Generates random email data defined explicitly by pinned data (an array of objects with "email" and "confirmed" fields)  
    - Inputs: Triggered by manual trigger node  
    - Outputs: Passes email data to "Set these fields to extract domain"  
    - Edge Cases: The sample data includes some invalid emails (missing '@'), useful to test validation  
    - Sub-workflow: None

  - **Sticky Note1**  
    - Type: Sticky Note (Visual aid)  
    - Role: Instruction to replace "Generate random data" node with actual data source  
    - Configuration: Contains text "Make sure you replace the Generate random data with your actual data"  
    - Inputs/Outputs: None  
    - Edge Cases: None

---

#### 1.2 Email Processing

- **Overview:**  
  This block performs the core logic: extracting the domain from each email and verifying if the email syntax is valid using n8n's native JSON functions.

- **Nodes Involved:**  
  - Set these fields to extract domain (Set node)  
  - Sticky Note (Instructional note)

- **Node Details:**

  - **Set these fields to extract domain**  
    - Type: Set  
    - Role: Adds three fields to each item:  
      - "Valid EmailIs email": boolean result of email syntax validation  
      - "Extract Domain": domain part extracted from the email  
      - "email": original email string  
    - Configuration:  
      - Uses expressions on the incoming JSON field `email` with native string functions:  
        - `{{$json.email.isEmail()}}` — validates email syntax  
        - `{{$json.email.extractDomain()}}` — extracts domain from email  
        - `{{$json.email}}` — passes through email string  
    - Inputs: Receives data from "Generate random data"  
    - Outputs: Enriches data with validation and domain fields  
    - Edge Cases:  
      - If the email field is missing or not a string, expressions may fail or return false/empty  
      - Domain extraction will fail or return empty string if email is malformed  
    - Version-specific: Uses expression syntax supported in n8n v0.152.0+ (for `.isEmail()` and `.extractDomain()`)  
    - Sub-workflow: None

  - **Sticky Note**  
    - Type: Sticky Note (Visual aid)  
    - Role: Explains the workflow purpose and usage instructions  
    - Configuration: Contains the text:  
      ```
      ## Email Validation and extract domain
      ** This workflow is aimed at making email validation and domain extract using the native functionalities in n8n

      ** Replace the debugger node with your actual data source to validate your own emails
      ```
    - Inputs/Outputs: None  
    - Edge Cases: None

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                       | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                   |
|-------------------------------|--------------------|-------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When clicking "Test workflow"  | Manual Trigger     | Initiates the workflow manually     | None                       | Generate random data         |                                                                                              |
| Generate random data           | Debug Helper       | Provides sample email data           | When clicking "Test workflow" | Set these fields to extract domain | Sticky Note1: Make sure you replace the Generate random data with your actual data            |
| Set these fields to extract domain | Set              | Extracts domain and validates email | Generate random data         | None                        | Sticky Note: Workflow purpose and usage instructions                                         |
| Sticky Note                   | Sticky Note        | Explains workflow purpose            | None                       | None                        | See content in node details                                                                   |
| Sticky Note1                  | Sticky Note        | Instruction to replace sample data   | None                       | None                        | See content in node details                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named `When clicking "Test workflow"`.  
   - No configuration needed.

2. **Create Debug Helper Node for Sample Data**  
   - Add a "Debug Helper" node named `Generate random data`.  
   - Set "Category" to `randomData`.  
   - Set "Random Data Type" to `email`.  
   - Pin the following JSON data as static input to simulate emails:  
     ```json
     [
       {"email":"Megan.Lueilwitz@yahoo.com","confirmed":true},
       {"email":"Tommie70@yahoo.com","confirmed":true},
       {"email":"Joanna.Fisher@yahoo.com","confirmed":false},
       {"email":"Terrence.Hettinger@yahoo.com","confirmed":false},
       {"email":"Eddie.Bradtke@hotmail.com","confirmed":false},
       {"email":"Marcus.Considine64@yahoo.com","confirmed":true},
       {"email":"Constance.Markshotmail.com","confirmed":false},
       {"email":"Dominick.Corwin@yahoo.com","confirmed":true},
       {"email":"Ellen54@yahoo.com","confirmed":true},
       {"email":"Angel.Hartmann40@hotmail.com","confirmed":false}
     ]
     ```
   - Connect `When clicking "Test workflow"` → `Generate random data`.

3. **Create Set Node to Extract Domain and Validate Email**  
   - Add a "Set" node named `Set these fields to extract domain`.  
   - In the "Value to Set" section, add three fields:  
     - Name: `Valid EmailIs email` (string)  
       Value: `={{ $json.email.isEmail() }}` (expression)  
     - Name: `Extract Domain` (string)  
       Value: `={{ $json.email.extractDomain() }}` (expression)  
     - Name: `email` (string)  
       Value: `={{ $json.email }}` (expression)  
   - Connect `Generate random data` → `Set these fields to extract domain`.

4. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add one sticky note near the start node explaining the workflow purpose and usage instructions.  
   - Add another sticky note near the sample data node reminding to replace it with actual data source.

5. **Run the Workflow**  
   - Manually trigger the workflow.  
   - Observe the output of the Set node showing the original email, its validation status, and extracted domain.

6. **Replace Sample Data with Actual Data Source**  
   - Replace or remove the `Generate random data` node.  
   - Connect your actual data source (e.g., HTTP Request, CSV Reader, or any input node) to the Set node.  
   - Ensure your data contains an `email` field.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                         |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow demonstrates native n8n expression functions `.isEmail()` and `.extractDomain()` introduced in recent versions. | n8n official docs: https://docs.n8n.io/nodes/expressions/ |
| Replace the sample "Generate random data" node with your actual email data source for production use.                 | Workflow sticky note and best practice                  |
| The workflow assumes each input item has an `email` field of type string for processing.                             | Data schema requirement                                 |
| Invalid emails (e.g., missing '@') will result in `Valid EmailIs email` as `false` and empty `Extract Domain`.        | Edge case to verify data quality                         |

---

This document fully describes the "Extract Domain and verify email syntax on the go" workflow, enabling reproduction, modification, and error anticipation without access to the original JSON file.