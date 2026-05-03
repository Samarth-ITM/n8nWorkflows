Create, update, and get an issue on Taiga

https://n8nworkflows.xyz/workflows/workflow-685-1747220141016.png


# Create, update, and get an issue on Taiga

### 1. Workflow Overview

This workflow automates the lifecycle management of an issue on Taiga, a project management platform. It demonstrates how to create a new issue in a specified Taiga project, update the newly created issue with additional information, and finally retrieve the updated issue details. The workflow is triggered manually and consists of three main logical blocks:

- **1.1 Input Trigger:** Manual trigger to start the workflow.
- **1.2 Issue Creation:** Creates a new issue in a given Taiga project.
- **1.3 Issue Update and Retrieval:** Updates the created issue’s description and then fetches the updated issue data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block initiates the workflow execution manually, allowing the user to start the process when desired.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Configuration:** No parameters configured; it simply waits for manual activation.  
  - **Key Expressions/Variables:** None  
  - **Input Connections:** None (entry node)  
  - **Output Connections:** Connects to the "Taiga" node to begin issue creation.  
  - **Version Requirements:** Compatible with all n8n versions supporting manual trigger nodes.  
  - **Potential Failures:** None, except user not triggering execution.  
  - **Sub-workflow:** None

---

#### 1.2 Issue Creation

- **Overview:**  
  This block creates a new issue in the Taiga project specified by project ID and subject. The newly created issue's ID and project details are passed downstream for update and retrieval.

- **Nodes Involved:**  
  - Taiga (first instance)

- **Node Details:**  
  - **Name:** Taiga  
  - **Type:** Taiga node (n8n-nodes-base.taiga)  
  - **Configuration:**  
    - Operation: Create an issue (default implied by lack of explicit operation)  
    - Project ID: 385605  
    - Subject: "n8n-docs"  
    - Additional fields: None  
  - **Key Expressions/Variables:** None in this node; static values used for projectId and subject.  
  - **Input Connections:** From manual trigger node "On clicking 'execute'"  
  - **Output Connections:** Outputs JSON containing the newly created issue, connected to "Taiga1" update node.  
  - **Credentials:** Uses "taigaCloudApi" credential (OAuth/token for Taiga Cloud API)  
  - **Version Requirements:** Requires Taiga node support with create issue operation.  
  - **Potential Failures:**  
    - Authentication errors if credentials expire or are invalid  
    - API rate limits or network issues  
    - Invalid project ID causing creation failure  
  - **Sub-workflow:** None

---

#### 1.3 Issue Update and Retrieval

- **Overview:**  
  This block updates the description of the created issue, then retrieves the updated issue details to confirm changes and provide output.

- **Nodes Involved:**  
  - Taiga1 (update operation)  
  - Taiga2 (get operation)

- **Node Details:**  
  - **Name:** Taiga1  
  - **Type:** Taiga node  
  - **Configuration:**  
    - Operation: Update  
    - Issue ID: Dynamically retrieved from previous node: `{{$node["Taiga"].json["id"]}}`  
    - Project ID: Dynamically from previous node: `{{$node["Taiga"].json["project"]}}`  
    - Update Fields: Description set to "This ticket is for the documentation for the Taiga node"  
  - **Key Expressions/Variables:** Uses expressions to reference the newly created issue’s ID and project from the first Taiga node output.  
  - **Input Connections:** From "Taiga" (issue creation node)  
  - **Output Connections:** Connected to "Taiga2" node for retrieval  
  - **Credentials:** Same "taigaCloudApi" credential as before  
  - **Version Requirements:** Taiga node version supporting update operation  
  - **Potential Failures:**  
    - Issue ID or project ID missing or incorrect expression evaluation failure  
    - Authentication or permission errors  
    - Network/API errors  
  - **Sub-workflow:** None

  - **Name:** Taiga2  
  - **Type:** Taiga node  
  - **Configuration:**  
    - Operation: Get  
    - Issue ID: Same dynamic expression as update node `{{$node["Taiga"].json["id"]}}`  
  - **Key Expressions/Variables:** References issue ID from the first Taiga node output  
  - **Input Connections:** From "Taiga1" node  
  - **Output Connections:** None (end of workflow)  
  - **Credentials:** Same "taigaCloudApi" credential  
  - **Version Requirements:** Taiga node version supporting get operation  
  - **Potential Failures:**  
    - Issue not found if update failed or invalid ID  
    - Authentication or API errors  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role           | Input Node(s)          | Output Node(s)     | Sticky Note |
|---------------------|--------------------------------|--------------------------|-----------------------|--------------------|-------------|
| On clicking 'execute'| Manual Trigger                 | Initiate workflow        | None                  | Taiga              |             |
| Taiga               | Taiga Node                    | Create issue             | On clicking 'execute' | Taiga1             |             |
| Taiga1              | Taiga Node                    | Update issue description | Taiga                 | Taiga2             |             |
| Taiga2              | Taiga Node                    | Retrieve updated issue   | Taiga1                | None               |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node**  
   - Node Name: `On clicking 'execute'`  
   - No configuration needed.

3. **Add a Taiga node to create an issue**  
   - Node Name: `Taiga`  
   - Credentials: Set up and select `taigaCloudApi` credential (OAuth2 or API token for Taiga Cloud).  
   - Parameters:  
     - Operation: (default, create issue)  
     - Project ID: `385605`  
     - Subject: `n8n-docs`  
     - Additional Fields: leave empty.

4. **Connect `On clicking 'execute'` node output to `Taiga` node input.**

5. **Add a second Taiga node to update the issue**  
   - Node Name: `Taiga1`  
   - Credentials: same `taigaCloudApi` credential.  
   - Parameters:  
     - Operation: `update`  
     - Issue ID: Use expression `{{$node["Taiga"].json["id"]}}` to get the ID from the previous node.  
     - Project ID: Use expression `{{$node["Taiga"].json["project"]}}` from previous node.  
     - Update Fields: Set description to `"This ticket is for the documentation for the Taiga node"`.

6. **Connect `Taiga` node output to `Taiga1` node input.**

7. **Add a third Taiga node to get the updated issue**  
   - Node Name: `Taiga2`  
   - Credentials: same `taigaCloudApi` credential.  
   - Parameters:  
     - Operation: `get`  
     - Issue ID: Use expression `{{$node["Taiga"].json["id"]}}`.

8. **Connect `Taiga1` node output to `Taiga2` node input.**

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                    |
|-----------------------------------------------------------------------------------------------|----------------------------------|
| Taiga API credentials require valid Taiga Cloud API token or OAuth2 setup for authentication. | Taiga node credential setup       |
| Ensure project ID `385605` exists and user has permissions to create/update issues in it.     | Taiga project prerequisite        |
| Expressions referencing previous node outputs rely on consistent JSON structure from Taiga API.| Expression usage in update/get nodes |
| Taiga node supports multiple operations; this workflow demonstrates create, update, and get.  | n8n Taiga node documentation      |