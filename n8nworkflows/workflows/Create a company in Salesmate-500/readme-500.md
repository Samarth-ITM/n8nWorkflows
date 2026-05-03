Create a company in Salesmate

https://n8nworkflows.xyz/workflows/create-a-company-in-salesmate-500


# Create a company in Salesmate

### 1. Workflow Overview

This workflow is designed to create a new company record in Salesmate CRM. It is triggered manually and then invokes the Salesmate node to add a company entity. The workflow serves as a simple integration example for users who want to automate company creation in Salesmate within n8n.

Logical blocks included:  
- **1.1 Manual Trigger:** Initiates the workflow execution manually.  
- **1.2 Salesmate Company Creation:** Connects to Salesmate API and creates a company resource with optional fields.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:** This block starts the workflow execution manually by the user.  
- **Nodes Involved:**  
  - *On clicking 'execute'*  
- **Node Details:**  
  - Type: Manual Trigger  
  - Role: Entry point to initiate workflow on demand.  
  - Configuration: No parameters configured; simple manual trigger.  
  - Expressions/Variables: None.  
  - Input: None (trigger node).  
  - Output: Connects to the Salesmate node.  
  - Version Requirements: Compatible with all n8n versions supporting manual triggers.  
  - Edge Cases: None specific; relies on manual user action.  
  - Sub-workflow: None.

#### 1.2 Salesmate Company Creation

- **Overview:** This block sends a request to Salesmate’s API to create a new company record. It uses the Salesmate node configured for the company resource.  
- **Nodes Involved:**  
  - *Salesmate*  
- **Node Details:**  
  - Type: Salesmate node (API integration)  
  - Role: Creates a company entity in Salesmate CRM.  
  - Configuration Choices:  
    - Resource: Company  
    - Operation: Create (implied by the node usage)  
    - Name and Owner fields: left blank, implying minimal data sent.  
    - Additional fields: empty object, meaning no extra company details are provided.  
  - Expressions/Variables: None configured, static empty fields.  
  - Input: Receives trigger from manual node.  
  - Output: Outputs API response from Salesmate.  
  - Version Requirements: Requires valid Salesmate API credentials configured in n8n.  
  - Edge Cases:  
    - API authentication failure if credentials missing or invalid.  
    - Failure if mandatory fields are required by Salesmate but not provided.  
    - Network issues or API downtime causing request failure.  
  - Sub-workflow: None.

---

### 3. Summary Table

| Node Name              | Node Type              | Functional Role           | Input Node(s)          | Output Node(s) | Sticky Note                                  |
|------------------------|------------------------|--------------------------|-----------------------|----------------|----------------------------------------------|
| On clicking 'execute'  | Manual Trigger         | Workflow entry point     | None                  | Salesmate      |                                              |
| Salesmate              | Salesmate API Node     | Create company in Salesmate | On clicking 'execute' | None           | Requires valid Salesmate API credentials.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a node of type *Manual Trigger*.  
   - No special parameters needed.  
   - This node will serve as the workflow entry point.

2. **Add Salesmate Node:**  
   - Add a node of type *Salesmate*.  
   - Set **Resource** to `Company`.  
   - Leave **Name**, **Owner**, and **Additional Fields** empty (or configure as needed).  
   - Configure credentials: Create or select existing Salesmate API credentials in n8n. This requires API key/token from Salesmate.  
   - No expressions or dynamic values are necessary for this minimal setup.

3. **Connect Nodes:**  
   - Connect the output of the *Manual Trigger* node to the input of the *Salesmate* node.

4. **Activate and Test:**  
   - Save the workflow.  
   - Click ‘Execute’ on the manual trigger node to run the workflow and create a company in Salesmate.  
   - Monitor the output of the Salesmate node for success or error messages.

---

### 5. General Notes & Resources

| Note Content                                                       | Context or Link                                      |
|-------------------------------------------------------------------|-----------------------------------------------------|
| This workflow requires valid Salesmate API credentials to function properly. | Salesmate API docs: https://developers.salesmate.io |
| Minimal example; extend by adding company details in Additional Fields for richer data. | Salesmate node fields documentation in n8n          |
| Screenshot available in original workflow description (not provided here). | Workflow screenshot reference (fileId:99)           |