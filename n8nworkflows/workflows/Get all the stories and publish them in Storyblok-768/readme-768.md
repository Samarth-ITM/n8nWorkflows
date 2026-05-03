Get all the stories and publish them in Storyblok

https://n8nworkflows.xyz/workflows/get-all-the-stories-and-publish-them-in-storyblok-768


# Get all the stories and publish them in Storyblok

### 1. Workflow Overview

This workflow is designed to interact with the Storyblok content management system via its Management API. Its primary purpose is to retrieve all stories within a specified space whose slugs start with the prefix "release" and then publish each of those stories automatically. This is particularly useful for content teams or developers who need to automate the publishing process for a batch of related stories based on naming conventions.

The workflow consists of three logical blocks:

- **1.1 Manual Trigger:** Initiates the workflow execution manually.
- **1.2 Retrieve Stories:** Queries the Storyblok Management API to get all stories starting with the prefix "release".
- **1.3 Publish Stories:** Iterates over each retrieved story and publishes it via the Storyblok Management API.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block allows the workflow to be started manually by a user within the n8n interface.

- **Nodes Involved:**  
  - `On clicking 'execute'`

- **Node Details:**  

  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Configuration:** No parameters, serves simply to initiate the workflow.  
  - **Expressions/Variables:** None.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Connected to the `Storyblok` node to trigger the data retrieval process.  
  - **Version Requirements:** Compatible with n8n versions supporting manual trigger nodes.  
  - **Potential Failures:** None expected, except workflow execution interruptions or user cancellation.  
  - **Sub-workflow:** Not applicable.

#### 1.2 Retrieve Stories

- **Overview:**  
  This block queries the Storyblok Management API to retrieve all stories from a specific space that have slugs starting with "release".

- **Nodes Involved:**  
  - `Storyblok`

- **Node Details:**  

  - **Node Name:** Storyblok  
  - **Type:** Storyblok Node (n8n-nodes-base.storyblok)  
  - **Configuration:**  
    - Space ID set to `96940` (the target Storyblok space).  
    - Source set to `managementApi` to use the management API endpoint.  
    - Operation set to `getAll` to retrieve all matching stories.  
    - Filter applied with `starts_with` set to `"release"` to filter stories by slug prefix.  
  - **Expressions/Variables:**  
    - Static space ID `96940`.  
  - **Inputs:**  
    - Triggered by the manual trigger node.  
  - **Outputs:**  
    - Outputs an array of stories matching the filter criteria, each containing story metadata including `id`.  
    - Connected to the next node `Storyblok1` for publishing.  
  - **Credentials:**  
    - Uses the Storyblok Management API credential named `storyblok-tanay`.  
  - **Version Requirements:**  
    - Requires API credential properly configured with management API access.  
  - **Potential Failures:**  
    - Authentication failure if credentials are invalid or expired.  
    - API rate limiting or network errors.  
    - No stories found (empty output).  
  - **Sub-workflow:** Not applicable.

#### 1.3 Publish Stories

- **Overview:**  
  This block publishes each story retrieved from the previous node by invoking the Storyblok Management API publish operation on each story ID.

- **Nodes Involved:**  
  - `Storyblok1`

- **Node Details:**  

  - **Node Name:** Storyblok1  
  - **Type:** Storyblok Node (n8n-nodes-base.storyblok)  
  - **Configuration:**  
    - Dynamically sets the space ID by referencing the output of the previous node: `={{$node["Storyblok"].parameter["space"]}}`.  
    - Operation set to `publish` to publish the story.  
    - Story ID dynamically set using the expression: `={{$node["Storyblok"].json["id"]}}`, iterating over each story output by the previous node.  
  - **Expressions/Variables:**  
    - Uses expressions to dynamically define space and story IDs per item.  
  - **Inputs:**  
    - Receives an array of stories from the previous node, processes each separately (via n8n’s item iteration).  
  - **Outputs:**  
    - Outputs the result of each publish operation, including success status and metadata.  
  - **Credentials:**  
    - Uses the same Storyblok Management API credential as before.  
  - **Version Requirements:**  
    - Requires support for dynamic expressions and Storyblok Management API publish operation in n8n.  
  - **Potential Failures:**  
    - Authentication issues.  
    - Publishing errors if the story is already published or locked.  
    - Network or API errors.  
    - Handling of empty input should be considered if the previous node returns no stories.  
  - **Sub-workflow:** Not applicable.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                            | Input Node(s)          | Output Node(s)         | Sticky Note                               |
|---------------------|-------------------------|-------------------------------------------|-----------------------|------------------------|-------------------------------------------|
| On clicking 'execute'| Manual Trigger          | Starts workflow manually                   | None                  | Storyblok              |                                           |
| Storyblok           | Storyblok API Node       | Retrieves all stories starting with "release" | On clicking 'execute' | Storyblok1             |                                           |
| Storyblok1          | Storyblok API Node       | Publishes each retrieved story             | Storyblok             | None                   |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type **Manual Trigger** and name it `On clicking 'execute'`.  
   - No parameters need to be set. This node will start the workflow manually.

2. **Add Storyblok Node to Retrieve Stories**  
   - Add a **Storyblok** node and name it `Storyblok`.  
   - Configure credentials to use your Storyblok Management API credential (e.g., `storyblok-tanay`).  
   - Set the following parameters:  
     - **Space:** `96940` (or your target Storyblok space ID).  
     - **Source:** `managementApi`.  
     - **Operation:** `getAll`.  
     - **Filters:** Set `starts_with` to `release` to filter stories by slug prefix.  
   - Connect the output of `On clicking 'execute'` node to the input of this node.

3. **Add Storyblok Node to Publish Stories**  
   - Add another **Storyblok** node and name it `Storyblok1`.  
   - Use the same Storyblok Management API credentials as the previous node.  
   - Set the parameters:  
     - **Space:** Use expression to dynamically inherit the space from the previous node: `={{$node["Storyblok"].parameter["space"]}}`.  
     - **Source:** `managementApi`.  
     - **Operation:** `publish`.  
     - **Story ID:** Use expression to get the current item's story ID from the previous node: `={{$node["Storyblok"].json["id"]}}`.  
   - Connect the output of the `Storyblok` node to the input of this node. This will ensure each retrieved story is processed individually for publishing.

4. **Save and Activate the Workflow**  
   - Save your workflow and activate it if automatic execution is desired.  
   - Alternatively, trigger manually using the `On clicking 'execute'` node.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                   |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Storyblok Management API documentation provides detailed info on operations such as `getAll` and `publish`. | https://www.storyblok.com/docs/api/management                     |
| Ensure that the Storyblok Management API credential has proper scopes and is valid to prevent authentication errors. | Credential setup in n8n under Storyblok Management API           |
| Filtering by `starts_with` is case-sensitive and depends on story slugs; verify slug naming conventions in your content. | Storyblok content slug management                                 |
| Publishing stories via API may trigger content delivery; handle rate limits and error retries as needed. | API rate limits and error handling best practices                |

---

This document fully describes the workflow structure, node configurations, and stepwise instructions to recreate and understand the automation. It also highlights potential failure points and integration considerations to ensure reliable operation.