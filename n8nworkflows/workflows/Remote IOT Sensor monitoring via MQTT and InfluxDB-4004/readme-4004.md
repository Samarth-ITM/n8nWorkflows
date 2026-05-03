Remote IOT Sensor monitoring via MQTT and InfluxDB

https://n8nworkflows.xyz/workflows/remote-iot-sensor-monitoring-via-mqtt-and-influxdb-4004


# Remote IOT Sensor monitoring via MQTT and InfluxDB

### 1. Workflow Overview

This workflow demonstrates how to monitor remote IoT sensor data using the MQTT protocol and store it into an InfluxDB time-series database via n8n automation. It is designed primarily for users who want a practical example of integrating MQTT-based IoT devices (such as an ESP32 microcontroller with a DHT22 temperature and humidity sensor) to collect sensor readings and persist them for real-time analysis or visualization.

The workflow is logically divided into three main blocks:

- **1.1 MQTT Input Reception:** Subscribes to a remote MQTT topic ("wokwi-weather") on a public Mosquitto MQTT broker to receive sensor data payloads published by the ESP32 device.

- **1.2 Payload Data Preparation:** Parses and formats the raw MQTT message payload into a line protocol string compliant with InfluxDB ingestion requirements.

- **1.3 Data Ingestion into InfluxDB:** Sends the formatted sensor data to a specified InfluxDB bucket using an authenticated HTTP POST request.

---

### 2. Block-by-Block Analysis

#### Block 1.1: MQTT Input Reception

- **Overview:**  
  This block establishes a live subscription to a remote MQTT topic to receive data published by the IoT sensor. It listens continuously for temperature and humidity data updates.

- **Nodes Involved:**  
  - Remote Sensor MQTT Trigger  
  - Sticky Note (comment explaining the MQTT trigger)

- **Node Details:**

  - **Remote Sensor MQTT Trigger**  
    - **Type:** MQTT Trigger node  
    - **Role:** Subscribes to the MQTT topic "wokwi-weather" via a configured Mosquitto MQTT broker to receive messages.  
    - **Configuration:**  
      - Topic: "wokwi-weather"  
      - MQTT credential: Custom MQTT account credential pointing to the test Mosquitto broker at "test.mosquitto.org".  
    - **Inputs:** None (trigger node)  
    - **Outputs:** Forwards incoming MQTT messages downstream.  
    - **Version requirements:** Requires n8n version supporting MQTT Trigger (v1 used here).  
    - **Potential Failures:**  
      - MQTT broker connectivity issues (network failure, broker down).  
      - Authentication errors if credential is misconfigured.  
      - Topic subscription failure if topic does not exist or broker access is restricted.  
    - **Sticky Note:** Describes the subscription to the "wokwi-weather" topic and origin of data from the remote ESP32 with DHT22 sensor.

---

#### Block 1.2: Payload Data Preparation

- **Overview:**  
  This block processes the raw MQTT message payload, validating and parsing the JSON-formatted temperature and humidity readings, and reformats them into the InfluxDB line protocol string required for ingestion.

- **Nodes Involved:**  
  - Payload data preparation node (Code node)  
  - Sticky Note1 (comment on the code node)

- **Node Details:**

  - **Payload data preparation node**  
    - **Type:** Code node (JavaScript)  
    - **Role:** Parses the incoming MQTT message payload (string), validates presence and type of temperature and humidity values, then formats them into a string like `"topic_name humidity=45,temp=22"` compliant with InfluxDB line protocol.  
    - **Configuration:**  
      - Custom JavaScript code that attempts JSON parsing of `$json.message` (payload from MQTT message).  
      - Throws explicit errors if parsing fails or expected fields are missing/invalid.  
      - Uses `$json.topic` to prepend topic name; defaults to "unknown-topic" if absent.  
    - **Key Expressions/Variables:**  
      - `$json.message` (input MQTT message payload as string)  
      - `$json.topic` (input MQTT topic)  
    - **Inputs:** Receives raw MQTT messages from the MQTT Trigger node.  
    - **Outputs:** Outputs JSON with a single field `payload` containing the formatted line protocol string for InfluxDB.  
    - **Version requirements:** Standard Code node, no special version dependencies.  
    - **Potential Failures:**  
      - Invalid JSON payload causing parse error.  
      - Missing or non-numeric temperature/humidity properties leading to validation error.  
      - Unhandled exceptions if input structure changes.  
    - **Sticky Note1:** Explains that this node extracts temperature and humidity values and ensures correct JSON formatting for database ingestion.

---

#### Block 1.3: Data Ingestion into InfluxDB

- **Overview:**  
  This block performs the ingestion of the properly formatted sensor data into an InfluxDB bucket by making an authenticated HTTP POST request.

- **Nodes Involved:**  
  - Data ingest to InfluxDB bucket (HTTP Request node)  
  - Sticky Note3 (comment on the HTTP request node)

- **Node Details:**

  - **Data ingest to InfluxDB bucket**  
    - **Type:** HTTP Request node  
    - **Role:** Sends a POST request to the configured InfluxDB API endpoint to write sensor data into the specified bucket.  
    - **Configuration:**  
      - URL: `http://localhost:8086/api/v2/write?orgID=<Organization ID>&bucket=<InfluxDB bucket name>&precision=s` (placeholders must be replaced with actual values)  
      - HTTP Method: POST  
      - Body: Raw content set to `={{ $json.payload }}` (the line protocol string from the code node)  
      - Headers: Authorization header with API token for InfluxDB access (`Token <API Token value>`)  
      - Content-Type: Raw body  
      - Sends body and headers with request.  
    - **Inputs:** Receives formatted payload from the code node.  
    - **Outputs:** Outputs the HTTP response (success or error) for downstream handling if needed.  
    - **Version requirements:** HTTP Request node v4.2 or higher recommended for header parameter support.  
    - **Potential Failures:**  
      - HTTP connectivity issues (InfluxDB server down or unreachable).  
      - Authentication failure due to invalid or missing API token.  
      - Incorrect organization ID or bucket name leading to data write failure.  
      - Malformed line protocol string causing InfluxDB to reject data.  
    - **Sticky Note3:** Specifies that this node posts temperature and humidity data to a local InfluxDB instance running on port 8086.

---

### 3. Summary Table

| Node Name                   | Node Type         | Functional Role                              | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                          |
|-----------------------------|-------------------|----------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note       | Comment on MQTT subscription                  | None                        | None                        | MQTT trigger subscribed to a topic called wokwi-weather via a Mosquitto MQTT broker. The trigger receives the temperature and humidity payloads from a DHT22 sensor connected to a remote ESP32 microcontroller |
| Remote Sensor MQTT Trigger  | MQTT Trigger      | Subscribes to MQTT topic to receive sensor data | None                        | Payload data preparation node |                                                                                                    |
| Sticky Note1                | Sticky Note       | Comment on payload extraction code            | None                        | None                        | Javascript code to extract the temperature and humidity values to ensure correct JSON format for the database |
| Payload data preparation node | Code              | Parses and reformats MQTT payload to InfluxDB line protocol | Remote Sensor MQTT Trigger  | Data ingest to InfluxDB bucket |                                                                                                    |
| Sticky Note3                | Sticky Note       | Comment on HTTP POST request to InfluxDB      | None                        | None                        | HTTP request node posts temperature and humidity data from the DHT22 sensor to the InfluxDB data bucket running on a local host http://localhost:8086 |
| Data ingest to InfluxDB bucket | HTTP Request      | Sends sensor data to InfluxDB bucket          | Payload data preparation node | None                        |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MQTT Trigger Node**  
   - Add an MQTT Trigger node named "Remote Sensor MQTT Trigger".  
   - Configure topic subscription to `"wokwi-weather"`.  
   - Set MQTT credentials: create or select existing MQTT credential configured for the Mosquitto broker at `test.mosquitto.org` (no authentication required).  
   - Leave other options default.  
   - Position node on canvas (e.g., left side).

2. **Create Code Node for Payload Preparation**  
   - Add a Code node named "Payload data preparation node".  
   - Connect the output of the MQTT Trigger node to the input of this Code node.  
   - In the Code node, paste the following JavaScript code:

     ```javascript
     // Parse incoming MQTT message payload as JSON
     let data;
     try {
       data = JSON.parse($json.message);
     } catch (e) {
       throw new Error("Invalid JSON in MQTT message");
     }

     const topic = $json.topic || "unknown-topic";

     if (typeof data.humidity !== "number" || typeof data.temp !== "number") {
       throw new Error("Missing or invalid humidity/temp in MQTT message");
     }

     const line = `${topic} humidity=${data.humidity},temp=${data.temp}`;

     return [{ json: { payload: line } }];
     ```

   - This code extracts and validates temperature and humidity, then formats them for InfluxDB ingestion.

3. **Create HTTP Request Node for InfluxDB Data Ingestion**  
   - Add an HTTP Request node named "Data ingest to InfluxDB bucket".  
   - Connect the output of the Code node to this HTTP Request node.  
   - Configure as follows:  
     - HTTP Method: POST  
     - URL: `http://localhost:8086/api/v2/write?orgID=<Organization ID>&bucket=<InfluxDB bucket name>&precision=s`  
       *(Replace `<Organization ID>` and `<InfluxDB bucket name>` with actual values from your InfluxDB setup.)*  
     - Body Content Type: Raw  
     - Body: Expression: `={{ $json.payload }}`  
     - Headers: Add header with name `Authorization` and value `Token <API Token value generated in InfluxDB>` (replace placeholder with your InfluxDB token).  
     - Enable sending of body and headers.  
   - Leave other options default.

4. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add Sticky Note nodes near each major node to document their purpose as in the original workflow:  
     - Near MQTT Trigger: "MQTT trigger subscribed to a topic called wokwi-weather via a Mosquitto MQTT broker. The trigger receives the temperature and humidity payloads from a DHT22 sensor connected to a remote ESP32 microcontroller."  
     - Near Code node: "Javascript code to extract the temperature and humidity values to ensure correct JSON format for the database."  
     - Near HTTP Request node: "HTTP request node posts temperature and humidity data from the DHT22 sensor to the InfluxDB data bucket running on a local host http://localhost:8086."

5. **Save and Activate Workflow**  
   - Verify connections: MQTT Trigger → Code node → HTTP Request node.  
   - Test workflow by simulating MQTT messages or running the Wokwi ESP32 simulator configured to publish to "wokwi-weather".  
   - Monitor InfluxDB bucket to confirm data ingestion.

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                      |
|---------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Wokwi IoT ESP32 simulator runs MicroPython code publishing temperature and humidity to MQTT topic "wokwi-weather". | https://wokwi.com/projects/                                        |
| To configure Wokwi simulator, set MQTT_CLIENT_ID="" and MQTT_BROKER="test.mosquitto.org" in MicroPython code lines 28-29. | Provided in workflow description                                   |
| InfluxDB setup requires: valid URL (e.g., localhost:8086), bucket name, organization ID, and API token with write permissions. | https://docs.influxdata.com/influxdb/v2.0/security/tokens/         |
| This workflow template is easily adaptable to other MQTT topics and payload formats by modifying subscription topic and code node parsing logic. | Workflow description                                               |

---

This documentation enables full understanding of the workflow's architecture, node configurations, potential failure points, and step-by-step instructions to recreate the integration of remote IoT sensor data ingestion via MQTT and InfluxDB using n8n.