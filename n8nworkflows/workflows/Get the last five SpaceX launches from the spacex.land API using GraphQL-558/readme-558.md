Get the last five SpaceX launches from the spacex.land API using GraphQL

https://n8nworkflows.xyz/workflows/get-the-last-five-spacex-launches-from-the-spacex-land-api-using-graphql-558


# Get the last five SpaceX launches from the spacex.land API using GraphQL

### 1. Workflow Overview

This workflow is designed to fetch details about the last five SpaceX launches by querying the SpaceX GraphQL API hosted at `spacex.land`. Its primary use case is to demonstrate how to integrate and query a GraphQL endpoint within n8n, serving as a companion example for the GraphQL node documentation. The workflow consists of two logical blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 GraphQL Query Execution:** Execution of a GraphQL query to retrieve detailed launch data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:**  
  This block serves to manually start the workflow. It provides control over when the data retrieval process begins, allowing users to execute the workflow on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Role:** Initiates workflow execution manually via the n8n editor UI.  
  - **Configuration:** No parameters required; an empty configuration.  
  - **Expressions/Variables:** None.  
  - **Input Connections:** None (entry node).  
  - **Output Connections:** Connects to the GraphQL node.  
  - **Version Requirements:** Compatible with all n8n versions that support manual triggers.  
  - **Potential Failures:** None expected; manual trigger is stable.  
  - **Sub-workflow:** None.

#### 2.2 GraphQL Query Execution

- **Overview:**  
  This block executes a complex GraphQL query to the SpaceX API endpoint to fetch rich, nested data about the five most recent launches. It processes the query and returns the response as a string.

- **Nodes Involved:**  
  - GraphQL

- **Node Details:**  
  - **Node Name:** GraphQL  
  - **Type:** GraphQL Request Node (n8n-nodes-base.graphql)  
  - **Role:** Sends a GraphQL query to an external API and retrieves the response.  
  - **Configuration:**  
    - **Query:** A multiline GraphQL query requesting the last five launches, including mission name, local launch date, launch site, article and video links, rocket details (rocket name, core reuse data, payload types and masses), and associated ships with their names, home ports, and images.  
    - **Endpoint:** `https://api.spacex.land/graphql/`  
    - **Request Format:** JSON  
    - **Response Format:** String (raw JSON string)  
    - **Headers:** None (no authentication required)  
  - **Expressions/Variables:** Static query string; no dynamic expressions.  
  - **Input Connections:** Receives trigger from Manual Trigger node.  
  - **Output Connections:** None (end node).  
  - **Version Requirements:** Requires n8n version supporting GraphQL node with `string` response format.  
  - **Potential Failures:**  
    - Network connectivity or endpoint unavailability.  
    - API schema changes that invalidate the query.  
    - Timeout if the API is slow to respond.  
    - Malformed query errors if query is modified incorrectly.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role       | Input Node(s)           | Output Node(s) | Sticky Note                                                                                   |
|---------------------|----------------------------|----------------------|-------------------------|----------------|----------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger             | Workflow start trigger| None                    | GraphQL        |                                                                                              |
| GraphQL             | GraphQL Request Node        | Executes GraphQL query| On clicking 'execute'    | None           | This node fetches the last five SpaceX launches from the spacex.land GraphQL API using a detailed nested query. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a new node of type **Manual Trigger** (under `Triggers`) and label it "On clicking 'execute'".  
   - No configuration needed. This node will start the workflow on manual execution.

2. **Create GraphQL Node:**  
   - Add a new node of type **GraphQL** (under `HTTP Request` category).  
   - Label it "GraphQL".  
   - Configure the node as follows:  
     - **Endpoint URL:** `https://api.spacex.land/graphql/`  
     - **Request Format:** JSON  
     - **Response Format:** String  
     - **GraphQL Query:** Paste the following GraphQL query exactly:

       ```graphql
       {
         launchesPast(limit: 5) {
           mission_name
           launch_date_local
           launch_site {
             site_name_long
           }
           links {
             article_link
             video_link
           }
           rocket {
             rocket_name
             first_stage {
               cores {
                 flight
                 core {
                   reuse_count
                   status
                 }
               }
             }
             second_stage {
               payloads {
                 payload_type
                 payload_mass_kg
                 payload_mass_lbs
               }
             }
           }
           ships {
             name
             home_port
             image
           }
         }
       }
       ```

     - Leave **Header Parameters** empty (no authentication required).  
   - No additional credentials are needed for this public API.

3. **Connect Nodes:**  
   - Connect the output of "On clicking 'execute'" node to the input of the "GraphQL" node.

4. **Save and Execute:**  
   - Save the workflow.  
   - Click the "Execute Workflow" button, then click "Execute Node" on the manual trigger node to run the entire workflow and fetch data.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                          |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow serves as a companion example for the n8n GraphQL node documentation, illustrating nested queries. | n8n GraphQL Node Docs (https://docs.n8n.io/nodes/n8n-nodes-base.graphql/) |
| The SpaceX GraphQL API used is publicly accessible at https://api.spacex.land/graphql/                            | SpaceX API Homepage (https://spacex.land/)                |

---

This documentation fully describes the workflow for fetching recent SpaceX launches using a GraphQL query and manual trigger in n8n. It should enable users or automation agents to understand, reproduce, and adapt the workflow reliably.