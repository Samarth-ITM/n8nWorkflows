IOT device control with MQTT and webhook

https://n8nworkflows.xyz/workflows/iot-device-control-with-mqtt-and-webhook-4211


# IOT device control with MQTT and webhook

### 1. Workflow Overview

This workflow orchestrates control commands for IoT devices (such as an ESP32 microcontroller) using MQTT messaging triggered by HTTP requests via a webhook. It is designed to enable real-time control of GPIO pins on IoT hardware through web-based interactions. The workflow is structured into three logical blocks:

- **1.1 Input Reception:** Captures HTTP requests containing pin control commands from a web interface.
- **1.2 Data Preparation:** Processes and formats the incoming data to create a valid MQTT message payload.
- **1.3 MQTT Publishing:** Sends the formatted message to an MQTT broker, targeting a specific topic to command the IoT device.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP requests on a specific webhook path and captures the user's command (e.g., turning a pin "on" or "off") from the query parameters.

- **Nodes Involved:**  
  - IOT control Webhook

- **Node Details:**  

  - **Node:** IOT control Webhook  
    - Type: Webhook (n8n-nodes-base.webhook)  
    - Role: Entry point of the workflow; receives HTTP GET or POST requests from the IoT control webpage.  
    - Configuration:  
      - Webhook path set to `/pin-control`  
      - No additional options enabled  
    - Key Expressions / Variables:  
      - Accesses `{{$json.query.value}}` to read the command value from the URL query parameters.  
    - Input: External HTTP request  
    - Output: Passes the captured query data to the next node  
    - Potential Failures:  
      - Missing or malformed query parameters may result in undefined payloads downstream.  
      - Unauthorized or unexpected HTTP requests (no authentication configured).  
    - Notes:  
      - When the "on" or "off" button is clicked on the IoT control webpage, this webhook triggers the workflow with the selected value.  
    - Sub-workflow: None

#### 2.2 Data Preparation

- **Overview:**  
  Processes the incoming webhook data and prepares a structured payload for MQTT publishing, specifically extracting and assigning the pin control command.

- **Nodes Involved:**  
  - Set data for MQTT message payload

- **Node Details:**  

  - **Node:** Set data for MQTT message payload  
    - Type: Set (n8n-nodes-base.set)  
    - Role: Constructs the MQTT message payload by extracting the command from the webhook input and assigning it to a new JSON field `pin`.  
    - Configuration:  
      - Keeps only the set data fields (`keepOnlySet: true`)  
      - Sets a string field named `pin` with the value `={{ $json.query.value }}`  
    - Input: Output from the webhook node containing query parameters  
    - Output: JSON with a single field `pin` containing the command string ("on" or "off")  
    - Potential Failures:  
      - Expression evaluation fails if `query.value` is missing or undefined.  
      - Unexpected data types or empty inputs may generate invalid MQTT payloads.  
    - Notes:  
      - The node prepares the received data as the message payload for the MQTT topic.  
    - Sub-workflow: None

#### 2.3 MQTT Publishing

- **Overview:**  
  Publishes the prepared payload to the MQTT broker on the topic `pin-control`, thus commanding the physical IoT device.

- **Nodes Involved:**  
  - MQTT Publish Topic Node

- **Node Details:**  

  - **Node:** MQTT Publish Topic Node  
    - Type: MQTT (n8n-nodes-base.mqtt)  
    - Role: Sends the control message to the MQTT broker under the topic `pin-control` with the payload extracted from the previous node.  
    - Configuration:  
      - Topic set to `pin-control`  
      - Message set dynamically as `={{ $json.pin }}` (the value set in the previous node)  
      - Does not send input data automatically (only the explicitly set message is sent)  
      - No additional options enabled  
    - Credentials: Uses a configured MQTT account (credential ID `xtd75tjk1hKlQOba`)  
    - Input: Receives a JSON object containing the `pin` field from the Set node  
    - Output: Publishes to MQTT, no further nodes connected  
    - Potential Failures:  
      - MQTT broker connection issues (network failure, authentication failure)  
      - Topic or message misconfiguration resulting in no action on the device  
      - Broker unavailability or timeouts  
    - Notes:  
      - Publishes MQTT topic "pin-control" with the payload data to control the GPIO on the ESP32 microcontroller.  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name                     | Node Type         | Functional Role                          | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                          |
|-------------------------------|-------------------|----------------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| IOT control Webhook            | Webhook           | Receives HTTP requests from the IoT control webpage | External HTTP Request     | Set data for MQTT message payload | When the "on" or "off" button is clicked on the IOT control webpage the webhook gets the selected value and triggers the workflow |
| Set data for MQTT message payload | Set               | Prepares the MQTT message payload from webhook data | IOT control Webhook       | MQTT Publish Topic Node      | The set node prepares the receive data as a message payload for the MQTT topic                      |
| MQTT Publish Topic Node        | MQTT              | Publishes the MQTT message to control the IoT device | Set data for MQTT message payload | None                       | Publishes MQTT topic "pin-control" with the payload data to control the GPIO on the ESP32 microcontroller |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Add a new node of type **Webhook**.  
   - Set the webhook path to `pin-control`.  
   - Leave other options default.  
   - This node will receive HTTP requests from the IoT control webpage whose query parameter `value` contains the pin command (e.g., "on" or "off").

2. **Create Set Node:**  
   - Add a **Set** node connected to the Webhook node.  
   - Configure it to keep only the set data (`keepOnlySet` = true).  
   - Add a string field named `pin`.  
   - Set its value to the expression `{{$json.query.value}}` to extract the command from the webhook’s query parameters.

3. **Create MQTT Node:**  
   - Add an **MQTT** node connected to the Set node.  
   - Set the topic to `pin-control`.  
   - Set the message field to the expression `{{$json.pin}}` to send the command extracted in the previous step.  
   - Configure the MQTT node to **not send input data automatically** (sendInputData = false).  
   - Select or create MQTT credentials that point to your MQTT broker (e.g., username, password, broker URL).  

4. **Connect Nodes:**  
   - Connect the Webhook node’s output to the Set node’s input.  
   - Connect the Set node’s output to the MQTT node’s input.

5. **Test the Workflow:**  
   - Activate the workflow.  
   - Send an HTTP GET or POST request to `https://<your-n8n-instance>/webhook/pin-control?value=on` (or `off`).  
   - Verify that the MQTT broker receives a message on topic `pin-control` with payload "on" or "off".  
   - Confirm that the IoT device reacts correctly to the MQTT message.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                   |
|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow is designed to work seamlessly with IoT devices like ESP32 microcontrollers that listen to MQTT topics for GPIO control. | Project context                                   |
| For secure deployments, consider adding authentication/authorization on the webhook to prevent unauthorized access. | Security best practices                            |
| MQTT credentials must be properly configured in n8n for the MQTT node to publish messages successfully.          | n8n documentation on MQTT credential setup       |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.