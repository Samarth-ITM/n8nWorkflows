Clean and Log IoT Sensor Data to InfluxDB (Webhook | Function | HTTP)

https://n8nworkflows.xyz/workflows/clean-and-log-iot-sensor-data-to-influxdb--webhook---function---http--7248


# Clean and Log IoT Sensor Data to InfluxDB (Webhook | Function | HTTP)

### 1. Workflow Overview

This workflow is designed to receive raw IoT sensor data via an HTTP webhook, clean and validate the input data, and then log the processed measurements into an InfluxDB time-series database. It targets IoT scenarios where sensor readings (temperature, humidity, voltage) must be ingested reliably and stored for further analysis or monitoring.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accept sensor data via a POST webhook.
- **1.2 Configuration Setup:** Prepare InfluxDB connection parameters.
- **1.3 Data Cleaning and Validation:** Validate sensor readings against acceptable ranges and format data.
- **1.4 Data Logging:** Write cleaned data points into InfluxDB using an HTTP API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block exposes a webhook endpoint to receive incoming sensor data as JSON via HTTP POST requests. It immediately responds with a confirmation to the sender.

- **Nodes Involved:**  
  - Sensor Input

- **Node Details:**

  - **Sensor Input**  
    - Type: Webhook  
    - Role: Entry point for the workflow, listens for incoming IoT sensor data.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `/sensor-data`  
      - Response Data: Static JSON `{"status":"received"}` to acknowledge receipt.  
    - Inputs: None (trigger node)  
    - Outputs: Sends received JSON payload to "Set Config" node.  
    - Edge Cases / Failures:  
      - Missing or malformed JSON body could cause downstream errors.  
      - If the webhook URL changes, external devices must update their target endpoint.  
    - Version: Compatible with n8n webhook v1 node.

#### 1.2 Configuration Setup

- **Overview:**  
  This block sets all necessary InfluxDB connection parameters and measurement metadata as static variables for use in downstream nodes.

- **Nodes Involved:**  
  - Set Config

- **Node Details:**

  - **Set Config**  
    - Type: Set  
    - Role: Stores configuration values such as InfluxDB host URL, token, bucket, organization, and measurement name.  
    - Configuration:  
      - influxDbHost: URL of the InfluxDB server (e.g., `hosturl`)  
      - influxDbToken: Authentication token for InfluxDB API  
      - influxDbBucket: Target bucket for data storage  
      - influxDbOrg: InfluxDB organization name  
      - measurement: Measurement/table name for storing sensor data  
    - Inputs: Receives JSON from "Sensor Input" node  
    - Outputs: Passes enriched data to "Clean & Transform Data" node  
    - Edge Cases / Failures:  
      - If any config value is missing or incorrect, HTTP requests to InfluxDB will fail (auth errors, 404s).  
      - Token expiration or revocation will cause auth failures.  
    - Version: Compatible with n8n Set node v1.

#### 1.3 Data Cleaning and Validation

- **Overview:**  
  Validates incoming sensor data to ensure the values are within realistic ranges, rounds values to one decimal place, and normalizes the timestamp to ISO format. Throws errors on invalid data to prevent bad entries.

- **Nodes Involved:**  
  - Clean & Transform Data

- **Node Details:**

  - **Clean & Transform Data**  
    - Type: Function  
    - Role: Data validation and transformation  
    - Configuration:  
      - Validates temperature between -50 and 150 °C  
      - Validates humidity between 0 and 100 %  
      - Validates voltage between 0 and 500 V  
      - Throws error if any value is out of range  
      - Rounds temperature, humidity, voltage to one decimal place  
      - Converts timestamp to ISO string format  
    - Key Expressions:  
      - `isValid(value, min, max)` helper function to validate ranges  
      - Rounding via `Math.round(value * 10) / 10`  
      - ISO timestamp formatting via `new Date(data.timestamp).toISOString()`  
    - Inputs: Receives JSON with raw sensor data and config values  
    - Outputs: Cleaned JSON object with validated and formatted data  
    - Edge Cases / Failures:  
      - Throws error if sensor values are missing, non-numeric, or out of range.  
      - If timestamp is invalid or missing, `new Date()` could produce `Invalid Date` ISO string.  
      - Workflow will stop on error unless error handling is added.  
    - Version: Compatible with n8n Function node v1.

#### 1.4 Data Logging

- **Overview:**  
  Sends the cleaned sensor metrics to the InfluxDB HTTP API, writing data points into the specified measurement with proper authentication and query parameters.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Pushes cleaned sensor data into InfluxDB using the `/api/v2/write` endpoint.  
    - Configuration:  
      - URL: Dynamically set from `influxDbHost` config + `/api/v2/write`  
      - Method: POST  
      - Headers:  
        - Authorization: `Token <influxDbToken>`  
      - Query Parameters:  
        - bucket: from config  
        - org: from config  
        - precision: (optional, from config, but not set in provided JSON)  
      - Body: InfluxDB line protocol format, e.g.:  
        ```
        <measurement> temperature=23.4,humidity=45.6,voltage=3.7 <timestamp>
        ```  
      - Timestamp converted to epoch milliseconds (`Date.parse(...)`)  
      - Content Type: text/plain (raw body)  
    - Inputs: Receives cleaned JSON data from "Clean & Transform Data"  
    - Outputs: None connected (endpoint of workflow)  
    - Edge Cases / Failures:  
      - HTTP errors due to invalid token, unreachable host, malformed body.  
      - Timestamp precision mismatch could cause data misalignment.  
      - Missing config values cause request failures.  
      - Network timeouts or InfluxDB server errors.  
    - Version: HTTP Request node v4.2 features used (dynamic expressions in URL, headers, body, and query params).

---

### 3. Summary Table

| Node Name           | Node Type        | Functional Role                     | Input Node(s)     | Output Node(s)         | Sticky Note                                                                                                                                                      |
|---------------------|------------------|-----------------------------------|-------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sensor Input        | Webhook          | Receives raw sensor data (POST)   | None              | Set Config             |                                                                                                                                                                 |
| Set Config          | Set              | Defines InfluxDB connection params| Sensor Input      | Clean & Transform Data  |                                                                                                                                                                 |
| Clean & Transform Data | Function        | Validate and clean sensor data    | Set Config        | HTTP Request           |                                                                                                                                                                 |
| HTTP Request        | HTTP Request     | Send cleaned data to InfluxDB     | Clean & Transform Data | None                |                                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Sensor Input`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `sensor-data`  
   - Response: Set response data to JSON `{"status":"received"}`  
   - Save and activate webhook.

2. **Create Set Node for Configuration**  
   - Name: `Set Config`  
   - Type: Set  
   - Add string values:  
     - `influxDbHost`: your InfluxDB server URL (e.g., `http://localhost:8086`)  
     - `influxDbToken`: your InfluxDB API token  
     - `influxDbBucket`: target bucket name  
     - `influxDbOrg`: your organization name in InfluxDB  
     - `measurement`: measurement/table name (e.g., `sensor_data`)  
   - Connect output of `Sensor Input` to input of `Set Config`.

3. **Create Function Node for Cleaning and Validation**  
   - Name: `Clean & Transform Data`  
   - Type: Function  
   - Code:  
     ```javascript
     const data = $json.body;

     function isValid(value, min, max) {
       return typeof value === 'number' && value >= min && value <= max;
     }

     if (!isValid(data.temperature, -50, 150) || 
         !isValid(data.humidity, 0, 100) || 
         !isValid(data.voltage, 0, 500)) {
       throw new Error('Invalid sensor data range');
     }

     const cleaned = {
       temperature: Math.round(data.temperature * 10) / 10,
       humidity: Math.round(data.humidity * 10) / 10,
       voltage: Math.round(data.voltage * 10) / 10,
       timestamp: new Date(data.timestamp).toISOString()
     };

     return [{ json: cleaned }];
     ```
   - Connect output of `Set Config` to input of this node.

4. **Create HTTP Request Node to Write to InfluxDB**  
   - Name: `HTTP Request`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: Expression `{{$node["Set Config"].json["influxDbHost"]}}/api/v2/write`  
     - Method: POST  
     - Headers:  
       - Name: `Authorization`  
       - Value: `Token {{$node["Set Config"].json["influxDbToken"]}}`  
     - Query Parameters:  
       - bucket: `{{$node["Set Config"].json["influxDbBucket"]}}`  
       - org: `{{$node["Set Config"].json["influxDbOrg"]}}`  
       - precision: (optional - can be set to `ms` or `ns` if needed)  
     - Content Type: `text/plain` (raw)  
     - Body: Set to Expression:  
       ```
       {{$node["Set Config"].json["measurement"]}} temperature={{$json["temperature"]}},humidity={{$json["humidity"]}},voltage={{$json["voltage"]}} {{Date.parse($json["timestamp"])}}
       ```
   - Connect output of `Clean & Transform Data` to input of this node.

5. **Activate the Workflow** and test by sending a POST request with JSON body containing at least:  
   ```json
   {
     "temperature": 22.5,
     "humidity": 55.2,
     "voltage": 3.7,
     "timestamp": "2024-01-01T12:00:00Z"
   }
   ```

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                  |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| InfluxDB line protocol requires precise formatting for measurement, fields, and timestamp.         | https://docs.influxdata.com/influxdb/v2.0/write-data/developer-tools/line-protocol/ |
| Webhook node must be publicly accessible or tunneled for external IoT devices to send data.        | n8n documentation on Webhook node                |
| Configure InfluxDB API token with write permissions only for security best practices.               | InfluxDB API Token Management                     |
| Consider adding error handling nodes for robustness in production workflows (e.g., Catch node).    | n8n error handling best practices                  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.