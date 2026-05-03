Check if a Twitch Stream is Live

https://n8nworkflows.xyz/workflows/check-if-a-twitch-stream-is-live-2684


# Check if a Twitch Stream is Live

### 1. Workflow Overview

This workflow checks whether a specified Twitch user is currently live streaming or offline. It is designed primarily for Twitch streamers or community managers who want to automate notifications or integrations based on a streamer's live status.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts or sets the Twitch username to check.
- **1.2 Twitch API Query:** Uses a GraphQL query to Twitch’s API to fetch live stream data for the given user.
- **1.3 Live Status Determination:** Evaluates the API response to decide if the user is live or offline.

This structure enables easy integration with other workflows, such as sending alerts or social media posts when a stream goes live.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block provides the Twitch username to be checked. It uses a static input node for simplicity but can be replaced by dynamic input such as a list or external trigger.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)  
  - `Document` (Set Node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type and Role:* Manual trigger node to start the workflow manually.  
    - *Configuration:* No parameters set; simply triggers the workflow on demand.  
    - *Inputs/Outputs:* No input connections; output connected to `Document` node.  
    - *Edge Cases:* None, as it is manual trigger.  
    - *Sub-workflow:* None.

  - **Document**  
    - *Type and Role:* Set node to hold the Twitch username string.  
    - *Configuration:* Contains one field named `twitch` with value `"YOUR-TWITCH-USERNAME"` (placeholder).  
    - *Expressions:* The value is accessed downstream via `$('Document').item.json.twitch`.  
    - *Inputs/Outputs:* Input from Manual Trigger; output to `Twitch GraphQL` node.  
    - *Edge Cases:* If the username is invalid or empty, the workflow query will fail or return no data.  
    - *Sub-workflow:* None.

---

#### 2.2 Twitch API Query

- **Overview:**  
  Queries Twitch’s GraphQL API to retrieve stream information for the specified user. Returns details such as stream title, viewer count, and game.

- **Nodes Involved:**  
  - `Twitch GraphQL`

- **Node Details:**

  - **Twitch GraphQL**  
    - *Type and Role:* GraphQL node configured to query Twitch API.  
    - *Configuration:*  
      - Endpoint: `https://gql.twitch.tv/gql`  
      - Query:  
        ```graphql
        {
          user(login: "{{ $('Document').item.json.twitch }}") {
            stream {
              id
              viewersCount
              title
              type
              game {
                id
              }
            }
          }
        }
        ```  
      - Variables: none (empty object `"="`)  
      - Header: Fixed `client-id` set to `"kimne78kx3ncx6brgo4mv6wki5h1ko"` (anonymous client-id used by Twitch public web)  
    - *Expressions:* Uses expression to fetch username dynamically from `Document` node.  
    - *Inputs/Outputs:* Input from `Document`; output to `Is Online` node.  
    - *Edge Cases:*  
      - Twitch API rate limiting or downtime may cause errors.  
      - Invalid or non-existent username results in `stream` being `null`.  
      - Network errors or malformed queries may cause failures.  
    - *Sub-workflow:* None.

---

#### 2.3 Live Status Determination

- **Overview:**  
  Checks the Twitch API response to determine if the user is live. If the `stream` property is not null, the user is live; otherwise, offline.

- **Nodes Involved:**  
  - `Is Online`

- **Node Details:**

  - **Is Online**  
    - *Type and Role:* IF node that evaluates whether the `stream` property is empty or not.  
    - *Configuration:*  
      - Condition: Checks if `{{$json.data.user.stream}}` is NOT empty (not null).  
      - Case sensitive and strict validation enabled to ensure precise matching.  
    - *Inputs/Outputs:* Input from `Twitch GraphQL`; outputs:  
      - True branch if user is live (stream data present)  
      - False branch if user is offline (stream is null)  
    - *Edge Cases:*  
      - If API response structure changes, condition may fail.  
      - Null or undefined values handled by the notEmpty operator.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                 | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                          |
|--------------------------|---------------------|--------------------------------|----------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts the workflow manually    |                            | Document                |                                                                                                    |
| Document                 | Set                 | Holds Twitch username to check  | When clicking ‘Test workflow’ | Twitch GraphQL          | The document node serves as sample source for `twitch` username to check                            |
| Twitch GraphQL           | GraphQL             | Queries Twitch API for stream   | Document                   | Is Online               | the value of `client-id` parameter is a fixed known value used by twitch for anonymous call used in their website |
| Is Online                | IF                  | Determines if user is live      | Twitch GraphQL             |                         | we need only to check the value of `stream` if `null` to know if the user offline. Any value will denote the user is online |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Add a "Manual Trigger" node.
   - No parameters needed.
   - This node will start the workflow manually.

2. **Create Set Node for Twitch Username**
   - Add a "Set" node named `Document`.
   - Add a string field named `twitch`.  
   - Set its value to the Twitch username you want to check, e.g., `"YOUR-TWITCH-USERNAME"`.
   - Connect the output of the Manual Trigger node to this Set node.

3. **Create GraphQL Node to Query Twitch API**
   - Add a "GraphQL" node named `Twitch GraphQL`.
   - Set the "Endpoint" parameter to `https://gql.twitch.tv/gql`.
   - In the "Query" field, insert the following query (use expression to get username from `Document` node):  
     ```graphql
     {
       user(login: "{{ $('Document').item.json.twitch }}") {
         stream {
           id
           viewersCount
           title
           type
           game {
             id
           }
         }
       }
     }
     ```
   - Leave "Variables" as an empty JSON object (`{}` or `=` in n8n format).
   - Add an HTTP header parameter with:  
     - Name: `client-id`  
     - Value: `kimne78kx3ncx6brgo4mv6wki5h1ko`  
   - Connect the output of the `Document` node to this GraphQL node.

4. **Create IF Node to Check Live Status**
   - Add an "IF" node named `Is Online`.
   - Configure it with the following condition:  
     - Mode: Advanced (version 2)  
     - Condition: Check if the expression `{{$json.data.user.stream}}` is NOT empty.  
     - Operator: `notEmpty`  
     - This checks if the `stream` object exists, indicating the user is live.  
   - Connect the output of the `Twitch GraphQL` node to this IF node.

5. **Workflow Connections**
   - Connect nodes in sequence:  
     `When clicking ‘Test workflow’` → `Document` → `Twitch GraphQL` → `Is Online`.

6. **Optional Adjustments**
   - Replace the static username in the `Document` node with dynamic input if checking multiple Twitch users.
   - Add further nodes downstream of the IF node to handle live/offline branches, e.g., sending notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The value of `client-id` parameter is a fixed known value used by Twitch for anonymous API calls used on their website.                                                     | Sticky Note on `Twitch GraphQL` node                                                               |
| You can replace the `Document` node with a list or other data source to check multiple Twitch channels in batch.                                                           | Workflow description, Setup Instructions                                                           |
| For detailed Twitch API stream data structure, see the official Twitch API documentation: [Get Streams](https://dev.twitch.tv/docs/api/reference/#get-streams)             | Official Twitch API Documentation                                                                  |
| `stream` property being `null` means the user is offline; any object value means the user is live. This simplifies live status checks to a single property evaluation.       | Sticky Note on `Is Online` node                                                                     |
| This workflow is compatible for integration with other workflows to post live alerts to social media platforms or notify communities via Twitter/X, Bluesky, Discord, etc. | Common Use Cases section                                                                            |

---

This documentation enables full understanding, reproduction, and extension of the Twitch live status checking workflow in n8n.