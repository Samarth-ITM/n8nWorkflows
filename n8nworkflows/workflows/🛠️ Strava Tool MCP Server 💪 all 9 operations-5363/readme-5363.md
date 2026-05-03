🛠️ Strava Tool MCP Server 💪 all 9 operations

https://n8nworkflows.xyz/workflows/----strava-tool-mcp-server----all-9-operations-5363


# 🛠️ Strava Tool MCP Server 💪 all 9 operations

### 1. Workflow Overview

This workflow, titled **"Strava Tool MCP Server"**, serves as a comprehensive server-side automation interface to perform a full suite of operations on Strava activities. It targets developers, data analysts, or automation engineers who want to manage Strava data programmatically via n8n, supporting all 9 main activity-related operations exposed by the Strava Tool node.

The workflow is logically divided into two main blocks:

- **1.1 MCP Trigger Input Reception:**  
  This block accepts incoming requests or commands through the MCP Trigger node, acting as the single entry point for the workflow to start processing.

- **1.2 Strava Activity Operations:**  
  This block contains nine distinct Strava Tool nodes, each configured for one of the key operations on Strava activities. These nodes receive inputs from the MCP Trigger and return the respective Strava API responses.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Input Reception

- **Overview:**  
  This block contains the single MCP Trigger node that listens for incoming API requests or commands. It acts as the workflow’s entry point, routing requests to the appropriate Strava action nodes based on the operation requested.

- **Nodes Involved:**  
  - Strava Tool MCP Server

- **Node Details:**

  - **Strava Tool MCP Server**  
    - *Type and Role:* MCP Trigger node from the Langchain MCP package; serves as the external API trigger or webhook.  
    - *Configuration:* No special parameters configured; uses default webhook settings with a fixed webhook ID.  
    - *Expressions/Variables:* None explicitly used; it passes incoming data to downstream nodes.  
    - *Input/Output:* No input nodes; outputs are connected to all Strava Tool nodes via the ‘ai_tool’ output channel.  
    - *Version Requirements:* n8n version supporting MCP Trigger and Langchain MCP package.  
    - *Edge Cases:* Potential failures include webhook connectivity issues, malformed incoming requests, or unsupported operation commands.  
    - *Sub-workflow:* None.

#### 2.2 Strava Activity Operations

- **Overview:**  
  This block comprises nine Strava Tool nodes, each implementing a specific Strava API operation related to activities. Every node receives input from the MCP Trigger and performs one of these operations: create, get, update activities, and fetch related data such as comments, kudos, laps, streams, zones, and multiple activities.

- **Nodes Involved:**  
  - Create an activity  
  - Get an activity  
  - Update an activity  
  - Get many activities  
  - Get all activity comments  
  - Get all activity kudos  
  - Get all activity laps  
  - Get all activity streams  
  - Get all activity zones

- **Node Details:**

  - **Create an activity**  
    - *Type and Role:* Strava Tool node configured to create a new activity on Strava.  
    - *Configuration:* Uses Strava API credentials; parameters expected to include activity details (type, distance, time, etc.).  
    - *Expressions/Variables:* Receives data from MCP Trigger inputs.  
    - *Input/Output:* Input from MCP Trigger; no output nodes specified (likely endpoints or response handling elsewhere).  
    - *Edge Cases:* Authentication failure, invalid input data, Strava API rate limits, missing required fields.  
    - *Sub-workflow:* None.

  - **Get an activity**  
    - *Type and Role:* Strava Tool node to retrieve details of a single activity by ID.  
    - *Configuration:* Requires activity ID input.  
    - *Edge Cases:* Nonexistent activity ID, permissions error, API timeouts.

  - **Update an activity**  
    - *Type and Role:* Strava Tool node to update an existing activity's information.  
    - *Configuration:* Requires activity ID and update parameters.  
    - *Edge Cases:* Read-only activities, invalid update data, permission errors.

  - **Get many activities**  
    - *Type and Role:* Strava Tool node to fetch multiple activities in batch.  
    - *Configuration:* May use pagination or filtering parameters.  
    - *Edge Cases:* Large data volume leading to timeouts or rate limiting.

  - **Get all activity comments**  
    - *Type and Role:* Strava Tool node to fetch all comments on a specific activity.  
    - *Configuration:* Requires activity ID.  
    - *Edge Cases:* Empty comment lists, permission errors.

  - **Get all activity kudos**  
    - *Type and Role:* Strava Tool node to retrieve all kudos for an activity.  
    - *Configuration:* Requires activity ID.  
    - *Edge Cases:* No kudos present, API limits.

  - **Get all activity laps**  
    - *Type and Role:* Strava Tool node to get lap data for an activity.  
    - *Configuration:* Requires activity ID.  
    - *Edge Cases:* Activities without laps, data availability.

  - **Get all activity streams**  
    - *Type and Role:* Strava Tool node to retrieve detailed streams (GPS, heart rate, etc.) for an activity.  
    - *Configuration:* Requires activity ID and stream types.  
    - *Edge Cases:* Empty streams, partial data, rate limits.

  - **Get all activity zones**  
    - *Type and Role:* Strava Tool node to get zone data (power, heart rate zones) for an activity.  
    - *Configuration:* Requires activity ID.  
    - *Edge Cases:* Zone data not available for all activities.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                   | Input Node(s)          | Output Node(s)                             | Sticky Note |
|---------------------------|----------------------------------|---------------------------------|-----------------------|--------------------------------------------|-------------|
| Workflow Overview 0       | Sticky Note                      | Documentation placeholder       |                       |                                            |             |
| Strava Tool MCP Server    | MCP Trigger                      | Entry point; receives requests  |                       | Create an activity, Get an activity, Get all activity comments, Get all activity kudos, Get all activity laps, Get many activities, Get all activity streams, Get all activity zones, Update an activity |             |
| Create an activity        | Strava Tool                     | Create new Strava activity      | Strava Tool MCP Server |                                            |             |
| Get an activity           | Strava Tool                     | Retrieve single activity details| Strava Tool MCP Server |                                            |             |
| Get all activity comments | Strava Tool                     | Fetch all comments on activity  | Strava Tool MCP Server |                                            |             |
| Get all activity kudos    | Strava Tool                     | Retrieve all kudos for activity | Strava Tool MCP Server |                                            |             |
| Get all activity laps     | Strava Tool                     | Retrieve all laps of activity   | Strava Tool MCP Server |                                            |             |
| Get many activities       | Strava Tool                     | Fetch multiple activities       | Strava Tool MCP Server |                                            |             |
| Get all activity streams  | Strava Tool                     | Retrieve all streams data       | Strava Tool MCP Server |                                            |             |
| Get all activity zones    | Strava Tool                     | Retrieve zone data for activity | Strava Tool MCP Server |                                            |             |
| Update an activity        | Strava Tool                     | Update existing activity        | Strava Tool MCP Server |                                            |             |
| Sticky Note 1             | Sticky Note                      | Documentation placeholder       |                       |                                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n, name it "Strava Tool MCP Server".

2. **Add an MCP Trigger node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name it `Strava Tool MCP Server`  
   - Use default webhook settings or configure with a fixed webhook ID if needed.  
   - No additional parameters required.

3. **Add Strava Tool nodes (9 total), one per operation:**  
   For each node:  
   - Node Type: `n8n-nodes-base.stravaTool`  
   - Connect its input to the MCP Trigger node's output (the ai_tool output).  
   - Name and configure as follows:

   - **Create an activity:**  
     - Operation: Create Activity  
     - Set required parameters such as activity type, name, distance, start_date, etc.  
     - Link Strava OAuth2 credentials.

   - **Get an activity:**  
     - Operation: Get Activity  
     - Input: activity ID parameter from MCP trigger input.

   - **Update an activity:**  
     - Operation: Update Activity  
     - Input: activity ID and update parameters.

   - **Get many activities:**  
     - Operation: Get Many Activities  
     - Support pagination or filtering parameters as needed.

   - **Get all activity comments:**  
     - Operation: Get All Activity Comments  
     - Input: activity ID.

   - **Get all activity kudos:**  
     - Operation: Get All Activity Kudos  
     - Input: activity ID.

   - **Get all activity laps:**  
     - Operation: Get All Activity Laps  
     - Input: activity ID.

   - **Get all activity streams:**  
     - Operation: Get All Activity Streams  
     - Input: activity ID, stream types.

   - **Get all activity zones:**  
     - Operation: Get All Activity Zones  
     - Input: activity ID.

4. **Configure Strava credentials for all Strava Tool nodes:**  
   - Use OAuth2 authentication with valid Strava API credentials.  
   - Ensure the token has the necessary scopes for reading and writing activities.

5. **Connect all Strava Tool nodes' input to the MCP Trigger node's 'ai_tool' output.**

6. **Optionally, add sticky notes for documentation:**  
   - Add notes titled "Workflow Overview" and "Sticky Note 1" as placeholders or for additional comments.

7. **Set workflow timezone to America/New_York** if relevant (optional).

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                         |
|------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The workflow supports the full range of Strava activity operations, useful for comprehensive automation and data management. | Workflow description and use case.                      |
| Ensure that Strava OAuth2 credentials have sufficient permissions and are refreshed as needed to avoid auth errors.          | Authentication best practice.                           |
| Strava API rate limits may cause failures under heavy load; consider implementing retries or throttling.                     | API usage considerations.                               |
| No sticky note content was provided in this workflow.                                            | Placeholder sticky notes present for user documentation. |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.