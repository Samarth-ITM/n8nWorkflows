Look up a person using their email in Clearbit

https://n8nworkflows.xyz/workflows/look-up-a-person-using-their-email-in-clearbit-484


# Look up a person using their email in Clearbit

### 1. Workflow Overview

This workflow is designed to look up detailed information about a person using their email address via the Clearbit API. It targets use cases where quick enrichment of contact data is required, such as in sales, marketing, or customer support contexts.

The workflow consists of two primary logical blocks:

- **1.1 Manual Trigger:** Allows the user to manually start the workflow.
- **1.2 Clearbit Person Lookup:** Performs the API call to Clearbit to retrieve personal data based on the provided email.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block initiates the workflow execution manually by the user. It serves as an entry point for the workflow.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Starts the workflow on user command.  
  - **Configuration:** No parameters; simple manual start.  
  - **Key Expressions/Variables:** None.  
  - **Input Connections:** None (entry node).  
  - **Output Connections:** Connects to the Clearbit node.  
  - **Version Requirements:** n8n version supporting manual triggers (all modern versions).  
  - **Potential Failures:** None expected, as this node only waits for manual execution.  
  - **Sub-workflow Reference:** None.

#### 1.2 Clearbit Person Lookup

- **Overview:**  
  This block queries Clearbit's Person API to retrieve detailed information about a person based on their email address.

- **Nodes Involved:**  
  - Clearbit

- **Node Details:**  
  - **Node Name:** Clearbit  
  - **Type:** Clearbit Node (API integration)  
  - **Technical Role:** Makes an authenticated API call to Clearbit’s "person" resource using the email address.  
  - **Configuration:**  
    - **Resource:** Person  
    - **Email:** Empty by default (requires user input or parameterization before execution).  
    - **Additional Fields:** None configured.  
    - **Credentials:** Uses Clearbit API credentials (must be configured in n8n).  
  - **Key Expressions/Variables:** The email parameter is the key input, currently empty and must be set for the workflow to function.  
  - **Input Connections:** Receives trigger from the manual trigger node.  
  - **Output Connections:** None (end of workflow).  
  - **Version Requirements:** Compatible with n8n versions supporting Clearbit node (generally from n8n v0.150+).  
  - **Potential Failures:**  
    - Authentication errors if Clearbit API key is invalid or missing.  
    - API rate limiting or network errors.  
    - Empty or invalid email input leading to API errors.  
    - Timeout or unexpected API response structure.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role           | Input Node(s)         | Output Node(s) | Sticky Note                                |
|-----------------------|--------------------|--------------------------|-----------------------|----------------|--------------------------------------------|
| On clicking 'execute'  | Manual Trigger     | Workflow start trigger   | None                  | Clearbit       |                                            |
| Clearbit              | Clearbit API       | Person lookup by email   | On clicking 'execute'  | None           | Requires Clearbit API credentials configured |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - Name it: `On clicking 'execute'`.  
   - No additional parameters need configuring.

2. **Create Clearbit Node**  
   - Add a new node of type **Clearbit**.  
   - Name it: `Clearbit`.  
   - Set the **Resource** to `Person`.  
   - Enter the email address to lookup in the **Email** field. If you want to make this dynamic, set it via an expression or external input before running.  
   - Leave **Additional Fields** empty unless you want to specify extra query parameters (none required here).  
   - Under **Credentials**, select or create the Clearbit API credential with your API key. This is mandatory for the API call to succeed.

3. **Connect Nodes**  
   - Connect the output of `On clicking 'execute'` to the input of `Clearbit`.

4. **Save and Activate the Workflow**  
   - Optionally, activate the workflow for scheduled or webhook trigger.  
   - Otherwise, you can run it manually using the manual trigger.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                               |
|------------------------------------------------------------------------------|---------------------------------------------------------------|
| Clearbit API documentation: https://clearbit.com/docs                         | Official API reference for understanding person lookup calls |
| Ensure Clearbit API key has sufficient permissions and is valid              | Credential management in n8n                                 |
| The email input must be set before execution for meaningful results          | Can be set via expression, parameter, or manual input        |
| This workflow is ideal for sales, marketing, or CRM enrichment workflows     | Use to augment contact records                                 |