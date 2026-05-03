Create, update and get a subscriber using the MailerLite node

https://n8nworkflows.xyz/workflows/create--update-and-get-a-subscriber-using-the-mailerlite-node-751


# Create, update and get a subscriber using the MailerLite node

### 1. Workflow Overview

This workflow demonstrates how to manage a subscriber in MailerLite using the n8n MailerLite node. It showcases the sequential process of creating a new subscriber, updating their custom field data, and then retrieving the updated subscriber information. The workflow targets use cases involving subscriber management for email marketing campaigns or CRM integrations.

Logical blocks:

- **1.1 Input Trigger**: Manual trigger to start the workflow.
- **1.2 Subscriber Creation**: Creates a new subscriber with basic details.
- **1.3 Subscriber Update**: Updates the newly created subscriber’s custom fields.
- **1.4 Subscriber Retrieval**: Retrieves the updated subscriber details for confirmation or further processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview**: This block initiates the workflow execution manually.
- **Nodes Involved**:  
  - On clicking 'execute'

- **Node Details**:  
  - **Node Name**: On clicking 'execute'  
  - **Type & Role**: Manual Trigger node — starts workflow on manual execution.  
  - **Configuration**: No parameters set; waits for manual trigger.  
  - **Expressions/Variables**: None.  
  - **Input/Output Connections**: Outputs to the "MailerLite" node.  
  - **Version**: Version 1.  
  - **Edge Cases**: No inherent failure modes; workflow will not proceed unless manually triggered.  
  - **Sub-workflow**: None.

#### 1.2 Subscriber Creation

- **Overview**: Creates a new subscriber in MailerLite with email and name fields.
- **Nodes Involved**:  
  - MailerLite

- **Node Details**:  
  - **Node Name**: MailerLite  
  - **Type & Role**: MailerLite node configured to create a subscriber.  
  - **Configuration**:  
    - Email: fixed value `harshil@n8n.io`  
    - Additional Fields: Name set to "Harshil"  
    - Operation: Default operation is create (implicit)  
  - **Expressions/Variables**: None; static values used.  
  - **Input/Output Connections**: Receives input from Manual Trigger; outputs to "MailerLite1".  
  - **Version**: Version 1.  
  - **Edge Cases**:  
    - API authentication failure if credentials invalid.  
    - Duplicate subscriber handling depends on MailerLite API behavior (may cause error or update).  
    - Network timeouts or request limits.  
  - **Sub-workflow**: None.

#### 1.3 Subscriber Update

- **Overview**: Updates the subscriber’s custom field "city" to "Berlin".
- **Nodes Involved**:  
  - MailerLite1

- **Node Details**:  
  - **Node Name**: MailerLite1  
  - **Type & Role**: MailerLite node performing update operation on subscriber.  
  - **Configuration**:  
    - Operation: "update"  
    - SubscriberId: Expression retrieves the email from the previous node output (`{{$node["MailerLite"].json["email"]}}`)  
    - Update Fields: Custom field "city" set to "Berlin"  
  - **Expressions/Variables**: Uses expression to fetch subscriber ID from prior node output.  
  - **Input/Output Connections**: Input from "MailerLite"; output to "MailerLite2".  
  - **Version**: Version 1.  
  - **Edge Cases**:  
    - Subscriber ID not found or invalid causes API error.  
    - If custom field ID is incorrect or missing in MailerLite account, update will fail.  
    - Auth or network issues could cause failure.  
  - **Sub-workflow**: None.

#### 1.4 Subscriber Retrieval

- **Overview**: Retrieves the subscriber data after update to verify changes.
- **Nodes Involved**:  
  - MailerLite2

- **Node Details**:  
  - **Node Name**: MailerLite2  
  - **Type & Role**: MailerLite node performing get operation on subscriber.  
  - **Configuration**:  
    - Operation: "get"  
    - SubscriberId: Expression using subscriber email from "MailerLite" node output (`{{$node["MailerLite"].json["email"]}}`)  
  - **Expressions/Variables**: Fetching subscriber ID from previous node output.  
  - **Input/Output Connections**: Input from "MailerLite1"; no outputs (end of chain).  
  - **Version**: Version 1.  
  - **Edge Cases**:  
    - Subscriber not found error if subscriber was not created or deleted.  
    - Auth or network issues.  
  - **Sub-workflow**: None.

---

### 3. Summary Table

| Node Name          | Node Type          | Functional Role                  | Input Node(s)         | Output Node(s)     | Sticky Note                                  |
|--------------------|--------------------|--------------------------------|-----------------------|--------------------|----------------------------------------------|
| On clicking 'execute' | Manual Trigger     | Starts workflow manually         | -                     | MailerLite          |                                              |
| MailerLite         | MailerLite Node     | Create subscriber with email/name | On clicking 'execute' | MailerLite1         |                                              |
| MailerLite1        | MailerLite Node     | Update subscriber custom fields  | MailerLite            | MailerLite2         |                                              |
| MailerLite2        | MailerLite Node     | Get subscriber details           | MailerLite1           | -                   |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name it `On clicking 'execute'`.  
   - No parameters need to be set.

3. **Add a MailerLite node:**  
   - Name it `MailerLite`.  
   - Set operation to create (default).  
   - Set Email field to `harshil@n8n.io`.  
   - Under Additional Fields, add Name with value `Harshil`.  
   - Select or create MailerLite API credentials with valid API key.  
   - Connect output of `On clicking 'execute'` to input of `MailerLite`.

4. **Add another MailerLite node:**  
   - Name it `MailerLite1`.  
   - Set operation to `update`.  
   - Set SubscriberId to an expression: `{{$node["MailerLite"].json["email"]}}` (gets email from previous node).  
   - Under Update Fields > Custom Fields, add a custom field value:  
     - Field ID: `city` (ensure this matches your MailerLite custom field ID for city).  
     - Value: `Berlin`.  
   - Use the same MailerLite API credentials as above.  
   - Connect output of `MailerLite` to input of `MailerLite1`.

5. **Add a third MailerLite node:**  
   - Name it `MailerLite2`.  
   - Set operation to `get`.  
   - Set SubscriberId to expression: `{{$node["MailerLite"].json["email"]}}`.  
   - Use the same MailerLite API credentials.  
   - Connect output of `MailerLite1` to input of `MailerLite2`.

6. **Save the workflow and execute manually by clicking the trigger node's Execute button.**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                    |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Ensure the MailerLite API key used has appropriate permissions to create, update, and read subscribers. | MailerLite API credentials setup                                  |
| The custom field ID `city` must exist in your MailerLite account for the update operation to succeed. | MailerLite custom fields documentation                            |
| The workflow uses static email `harshil@n8n.io`; adapt this to dynamic inputs for real use cases.    | Adapt for production use                                          |
| MailerLite node documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.mailerLite/ | Official n8n MailerLite node docs                                 |

---

This documentation fully describes the workflow’s structure, node configurations, data flows, and considerations for robustness and reproduction.