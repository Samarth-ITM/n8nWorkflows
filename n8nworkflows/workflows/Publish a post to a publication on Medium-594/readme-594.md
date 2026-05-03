Publish a post to a publication on Medium

https://n8nworkflows.xyz/workflows/publish-a-post-to-a-publication-on-medium-594


# Publish a post to a publication on Medium

### 1. Workflow Overview

This workflow automates the process of publishing a post directly to a Medium publication via the Medium API. It is designed for users who want to streamline content posting to a Medium publication without manual intervention through the Medium website.

**Logical Blocks:**

- **1.1 Trigger Block:** Manual initiation of the workflow.
- **1.2 Medium Publishing Block:** Sends post data to Medium’s API to publish a post in a specified publication.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:**  
  Acts as the manual starting point of the workflow. When the user clicks execute, the workflow begins.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Technical Role:** Starts the workflow on user action; no inputs required.  
  - **Configuration:** Default manual trigger with no parameters.  
  - **Expressions/Variables:** None.  
  - **Input connections:** None (entry node).  
  - **Output connections:** Connects to the Medium node.  
  - **Version requirements:** Compatible with all current n8n versions.  
  - **Potential Failures:** None expected unless manual execution is aborted.  
  - **Sub-workflows:** None.

#### 1.2 Medium Publishing Block

- **Overview:**  
  Publishes a post to a specified Medium publication using the Medium API credentials configured in n8n.

- **Nodes Involved:**  
  - Medium

- **Node Details:**  
  - **Type:** Medium node (API integration node)  
  - **Technical Role:** Sends post content to Medium’s publication endpoint.  
  - **Configuration:**
    - **Title:** Empty by default; must be populated to create a meaningful post.
    - **Content:** Empty by default; expects content in Medium-supported format (HTML, Markdown, or plain text).
    - **Publication:** Set to `true` indicating the post is to be published to a publication rather than a personal profile.
    - **Content Format:** Empty by default; should be set to specify the content type (e.g., 'html', 'markdown').
    - **Publication ID:** Empty by default; required to identify the target publication on Medium.
    - **Additional Fields:** Optional metadata or parameters.
  - **Expressions/Variables:** None set; expected to be configured dynamically in use.
  - **Input connections:** Receives trigger from manual execution node.
  - **Output connections:** None (end node).
  - **Version requirements:** Requires n8n Medium node v1 or higher and valid Medium API credentials configured under `mediumApi`.
  - **Potential Failures:**
    - Authentication errors if credentials are invalid or expired.
    - API errors if required fields (title, content, publicationId) are missing or incorrect.
    - Network timeouts or Medium service unavailability.
  - **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                      | Input Node(s)        | Output Node(s) | Sticky Note                      |
|-----------------------|---------------------|------------------------------------|----------------------|----------------|---------------------------------|
| On clicking 'execute'  | Manual Trigger      | Initiates workflow manually         | None                 | Medium         |                                 |
| Medium                | Medium API Node      | Publishes post to Medium publication | On clicking 'execute' | None           |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - Leave default settings; no parameters required.  
   - This node will start the workflow on manual execution.

2. **Create Medium Node**  
   - Add a new node of type **Medium**.  
   - Configure the following parameters:  
     - **Title:** Set this to the desired post title or use an expression to reference incoming data.  
     - **Content:** Provide the post content in accepted format (HTML/Markdown/plain text).  
     - **Publication:** Set to `true` to target a publication.  
     - **Content Format:** Specify format as `html` or `markdown` based on content.  
     - **Publication ID:** Enter the Medium publication ID where the post will be published.  
     - **Additional Fields:** Add any optional parameters if needed.
   - Under **Credentials**, select or create a Medium API credential with valid OAuth2 token or API key.

3. **Connect Nodes**  
   - Connect the output of the Manual Trigger node to the Medium node input.

4. **Save and Test**  
   - Save the workflow and click execute to manually trigger publishing a post to the specified Medium publication.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|-----------------------------------------------------------------------------------------------|-----------------------------------------|
| Medium API requires OAuth2 authentication; ensure credentials in n8n are valid and authorized. | Medium API documentation: https://github.com/Medium/medium-api-docs |
| Publication ID can be found via Medium API or Medium account interface for targeted publishing. | Medium publications overview            |
| This workflow requires manual trigger; can be adapted to listen to data input nodes for automation. | n8n Manual Trigger node documentation   |

---

This documentation provides a full understanding of the "Publish post to a publication" workflow, enabling users to replicate, maintain, or extend it effectively.