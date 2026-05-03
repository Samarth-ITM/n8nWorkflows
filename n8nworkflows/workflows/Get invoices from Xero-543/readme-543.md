Get invoices from Xero

https://n8nworkflows.xyz/workflows/get-invoices-from-xero-543


# Get invoices from Xero

### 1. Workflow Overview

This workflow is designed to retrieve all invoices from a Xero accounting organization using the Xero node in n8n. It serves as a companion example for users integrating with Xero via OAuth2 authentication in n8n, demonstrating how to fetch invoice data programmatically.

The workflow consists of two logical blocks:

- **1.1 Manual Trigger**: Initiates the workflow manually.
- **1.2 Xero Invoice Retrieval**: Connects to the Xero API using OAuth2 credentials and fetches all invoices from the specified organization.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview**:  
  Provides a manual start point for the workflow, allowing users to execute the workflow on demand.

- **Nodes Involved**:  
  - On clicking 'execute'

- **Node Details**:  

  - **Node Name**: On clicking 'execute'  
  - **Type**: Manual Trigger  
  - **Role**: Initiates workflow execution  
  - **Configuration**: Default manual trigger configuration with no parameters required.  
  - **Key Expressions/Variables**: None  
  - **Input Connections**: None (entry node)  
  - **Output Connections**: Connects to the Xero node  
  - **Version-Specific Requirements**: Compatible with all recent n8n versions  
  - **Potential Failure Modes**: None likely, as this node only triggers execution manually  
  - **Sub-workflow Reference**: None  

#### 1.2 Xero Invoice Retrieval

- **Overview**:  
  This node connects to the Xero API using OAuth2 credentials and retrieves all invoices for the specified organization.

- **Nodes Involved**:  
  - Xero

- **Node Details**:  

  - **Node Name**: Xero  
  - **Type**: Xero (API integration node)  
  - **Role**: Fetches invoices from Xero accounting platform  
  - **Configuration Choices**:  
    - Operation: `getAll` — retrieves all records in the resource category.  
    - Resource: Implicitly set to "Invoices" (since the workflow is about invoices) — the node defaults to invoices or requires manual selection.  
    - Organization ID: `"ab7e9014-5d01-418f-a64c-dbb6bf5ba2ea"` — specifies the Xero organization to query.  
  - **Credentials**: Uses the `n8n_xero` OAuth2 credential set, which must be preconfigured with valid Xero API access tokens.  
  - **Key Expressions/Variables**: None explicitly used, parameters are static.  
  - **Input Connections**: Receives trigger signal from the manual trigger node.  
  - **Output Connections**: None (end node)  
  - **Version-Specific Requirements**: Xero node requires n8n version supporting OAuth2 for Xero API.  
  - **Potential Failure Modes**:  
    - Authentication errors if OAuth2 token expired or revoked.  
    - API rate limits or network timeouts.  
    - Incorrect or invalid organization ID leading to empty or error responses.  
  - **Sub-workflow Reference**: None  

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role           | Input Node(s)        | Output Node(s) | Sticky Note                                         |
|----------------------|--------------------|--------------------------|----------------------|----------------|-----------------------------------------------------|
| On clicking 'execute' | Manual Trigger     | Start workflow manually  | None                 | Xero           |                                                     |
| Xero                 | Xero node          | Retrieve all invoices    | On clicking 'execute' | None           | Companion workflow for Xero node docs                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type "Manual Trigger".  
   - Leave default settings (no parameters needed).  
   - Name it `On clicking 'execute'`.

2. **Create Xero Node**  
   - Add a node of type "Xero".  
   - Set the operation to `getAll`.  
   - Ensure the resource is set to `Invoices` (default or explicit).  
   - Enter the Organization ID: `ab7e9014-5d01-418f-a64c-dbb6bf5ba2ea`.  
   - Select or create credentials: configure OAuth2 credentials for Xero API, named `n8n_xero`. This requires:  
     - Client ID and Client Secret from Xero developer portal.  
     - OAuth2 token configuration and consent.  
   - Name this node `Xero`.

3. **Connect Nodes**  
   - Connect the output of the `On clicking 'execute'` node to the input of the `Xero` node.

4. **Save and Execute**  
   - Save the workflow.  
   - Manually trigger the workflow by clicking the execute button on the manual trigger node.  
   - The Xero node will retrieve all invoices for the specified organization.

---

### 5. General Notes & Resources

| Note Content                                                             | Context or Link                                        |
|--------------------------------------------------------------------------|-------------------------------------------------------|
| This workflow serves as a companion example for Xero node documentation. | Refer to the official n8n Xero node docs for details: https://docs.n8n.io/integrations/n8n-nodes-base.xero/ |
| Ensure OAuth2 credentials are correctly set up in n8n for Xero API access. | Xero developer portal: https://developer.xero.com/    |

---

This documentation fully describes the "Get invoices from Xero" workflow, enabling users to understand, reproduce, and troubleshoot the process without requiring the original JSON export.