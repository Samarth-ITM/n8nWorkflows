Send a message via AWS SNS

https://n8nworkflows.xyz/workflows/send-a-message-via-aws-sns-501


# Send a message via AWS SNS

### 1. Workflow Overview

This workflow demonstrates how to send a message using the AWS SNS (Simple Notification Service) node in n8n. It is designed as a companion example for the AWS SNS node documentation, illustrating the basic usage of the node to publish a message to a specified SNS topic when triggered manually. The workflow consists of two logical blocks:

- **1.1 Input Reception:** A manual trigger node to start the workflow execution.
- **1.2 AWS SNS Publishing:** The AWS SNS node configured to send a predefined message and subject to an SNS topic.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow manually, allowing the user to trigger the sending of a message to AWS SNS on demand.
- **Nodes Involved:** 
  - On clicking 'execute'

- **Node Details:**

  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Technical Role:** Starts the workflow execution when manually activated.  
  - **Configuration Choices:** No parameters set, default manual trigger behavior.  
  - **Key Expressions or Variables:** None (static trigger)  
  - **Input Connections:** None (entry point)  
  - **Output Connections:** Connects to AWS SNS node  
  - **Version-Specific Requirements:** Compatible with all n8n versions supporting manual trigger nodes.  
  - **Edge Cases / Potential Failures:** None typical; if the workflow is not executed manually, the SNS message will not be sent.  
  - **Sub-workflow Reference:** None  

#### 1.2 AWS SNS Publishing

- **Overview:** This block sends a message with a subject to a specific AWS SNS topic using the AWS SNS node configured with appropriate credentials.
- **Nodes Involved:**  
  - AWS SNS

- **Node Details:**

  - **Node Name:** AWS SNS  
  - **Type:** AWS SNS (n8n-nodes-base.awsSns)  
  - **Technical Role:** Publishes a message to a defined AWS SNS topic.  
  - **Configuration Choices:**  
    - Topic: "n8n-rocks" (the SNS topic name)  
    - Message: "This is a test message" (content of the notification)  
    - Subject: "This is a test subject" (message subject line)  
  - **Key Expressions or Variables:** Static text used for message and subject; no dynamic expressions.  
  - **Input Connections:** Receives trigger from Manual Trigger node  
  - **Output Connections:** None (terminal node)  
  - **Version-Specific Requirements:** Requires n8n version supporting AWS SNS node (generally n8n version 0.92 or later).  
  - **Edge Cases / Potential Failures:**  
    - Authentication failure if AWS credentials are invalid or expired  
    - Network timeout or connectivity issues to AWS SNS endpoint  
    - Incorrect topic name or permissions may cause SNS publish failure  
  - **Sub-workflow Reference:** None  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                  | Input Node(s)          | Output Node(s) | Sticky Note                          |
|-------------------------|-------------------------|---------------------------------|-----------------------|----------------|------------------------------------|
| On clicking 'execute'    | Manual Trigger          | Initiates workflow manually     | -                     | AWS SNS        |                                    |
| AWS SNS                 | AWS SNS Node            | Sends message to AWS SNS topic  | On clicking 'execute'  | -              | Companion workflow for AWS SNS node docs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - Leave all parameters at default.  
   - Name this node: **On clicking 'execute'**.

2. **Create the AWS SNS Node**  
   - Add a new node of type **AWS SNS**.  
   - Configure the following parameters:  
     - **Topic**: Enter `n8n-rocks` (or your target SNS topic name).  
     - **Message**: Enter `This is a test message`.  
     - **Subject**: Enter `This is a test subject`.  
   - Name this node: **AWS SNS**.

3. **Connect the Nodes**  
   - Connect the output of **On clicking 'execute'** to the input of **AWS SNS**.

4. **Configure AWS Credentials**  
   - In n8n credentials, create or select an existing **AWS** credential with valid access key ID and secret access key.  
   - Assign this AWS credential to the **AWS SNS** node’s credentials field.

5. **Save and Execute**  
   - Save the workflow.  
   - Click **Execute Workflow** or the manual trigger button to send the message to AWS SNS.

---

### 5. General Notes & Resources

| Note Content                                      | Context or Link                                |
|--------------------------------------------------|-----------------------------------------------|
| Companion workflow for AWS SNS node docs          | This workflow serves as a practical example alongside the AWS SNS node documentation. |
| SNS topic name “n8n-rocks” used as example       | Replace with your own SNS topic in production. |

---

This document captures the full structure and logic of the "Send a message via AWS SNS" workflow, facilitating its reproduction, modification, and troubleshooting.