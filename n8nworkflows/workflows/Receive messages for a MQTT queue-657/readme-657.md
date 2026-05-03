Receive messages for a MQTT queue

https://n8nworkflows.xyz/workflows/receive-messages-for-a-mqtt-queue-657


# Receive messages for a MQTT queue

### 1. Workflow Overview

This workflow is designed to **receive and process messages from an MQTT queue**. Its primary use case is to listen to a specified MQTT topic and trigger downstream automation whenever a new message arrives. The workflow currently includes a single logical block:

- **1.1 MQTT Message Reception:** Listens to an MQTT broker topic to receive messages in real time.

---

### 2. Block-by-Block Analysis

#### 1.1 MQTT Message Reception

- **Overview:**  
  This block handles incoming messages from an MQTT broker by triggering the workflow each time a new message is published on the configured MQTT topic.

- **Nodes Involved:**  
  - MQTT Trigger

- **Node Details:**

  - **Node Name:** MQTT Trigger  
  - **Type and Technical Role:**  
    An event trigger node that subscribes to an MQTT topic and initiates the workflow when new messages are received.  
  - **Configuration Choices:**  
    - Uses MQTT credentials labeled "mqtt" for broker authentication.  
    - No additional options configured, so default subscription parameters apply (e.g., subscribing to all messages on the specified topic).  
  - **Key Expressions or Variables:**  
    None explicitly specified; the node outputs the raw message payload received from the MQTT broker.  
  - **Input and Output Connections:**  
    - Input: None (trigger node).  
    - Output: Emits the message data downstream (though no nodes are connected in this workflow).  
  - **Version-specific Requirements:**  
    - Uses typeVersion 1, compatible with n8n versions supporting this node type.  
  - **Edge Cases or Potential Failures:**  
    - Authentication failure if MQTT credentials are invalid or expired.  
    - Connectivity issues if the MQTT broker is unreachable or the network is down.  
    - Subscription failure if the topic does not exist or permissions are insufficient.  
    - Message format errors if the incoming payload is malformed or unexpected.  
  - **Sub-workflow Reference:**  
    None; this is a standalone node within the main workflow.

---

### 3. Summary Table

| Node Name    | Node Type                  | Functional Role       | Input Node(s) | Output Node(s) | Sticky Note |
|--------------|----------------------------|----------------------|---------------|----------------|-------------|
| MQTT Trigger | n8n-nodes-base.mqttTrigger | Receive MQTT messages | None          | None           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n** and name it "Receive messages for a MQTT queue".

2. **Add an "MQTT Trigger" node:**
   - Drag the "MQTT Trigger" node onto the canvas.
   - Configure the node as follows:
     - **Credentials:** Select or create MQTT credentials that connect to your MQTT broker.
       - Credentials require broker URL, port, and authentication details as needed (username/password or certificates).
     - **Options:** Leave default unless you want to specify topics, QoS, or other subscription parameters.
   - This node will start the workflow whenever a message is received on the MQTT topic.

3. **No additional nodes or connections are required** for this basic setup.

4. **Activate the workflow** to start listening for MQTT messages.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                   |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------|
| To configure MQTT credentials, ensure your broker details and authentication are correctly set. | n8n documentation: https://docs.n8n.io/nodes/n8n-nodes-base.mqttTrigger/ |
| This workflow currently only receives messages; further processing nodes can be added downstream.|                                                  |

---

This concise workflow serves as the foundation for any MQTT message-driven automation in n8n, awaiting extension with processing, filtering, or integration nodes.