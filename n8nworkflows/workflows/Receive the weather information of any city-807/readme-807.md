Receive the weather information of any city

https://n8nworkflows.xyz/workflows/receive-the-weather-information-of-any-city-807


# Receive the weather information of any city

### 1. Workflow Overview

This workflow is designed to receive a city name via an HTTP webhook request and respond with the current weather information for that city. It targets use cases where external applications or users need to fetch live weather data dynamically by providing a city name as a query parameter.

The workflow is logically divided into the following blocks:  
- **1.1 Input Reception:** Receiving the city name from an external request via a webhook node.  
- **1.2 Weather Data Retrieval:** Querying the OpenWeatherMap API to get current weather details for the given city.  
- **1.3 Data Formatting:** Extracting and formatting key weather information (temperature and description) to return as the response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures incoming HTTP requests that contain the city name as a query parameter. It acts as the entry point for the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Node Name:** Webhook  
  - **Type:** HTTP Webhook  
  - **Configuration:**  
    - Path: A unique webhook ID string ("45690b6a-2b01-472d-8839-5e83a74858e5") ensures a specific URL endpoint.  
    - Response Mode: Set to `lastNode`, meaning the output of the last executed node in the workflow will be sent as the HTTP response.  
    - Response Data: `allEntries` - all data from the last node's execution is sent in response.  
  - **Key Expressions:**  
    - The city name is expected at the path `query.city` in the incoming request JSON: `{{$node["Webhook"].json["query"]["city"]}}`.  
  - **Input / Output:**  
    - Input: Receives HTTP requests (GET or POST) with query parameters.  
    - Output: Forwards data to the OpenWeatherMap node.  
  - **Potential Failures:**  
    - Missing or malformed query parameters (no city provided).  
    - Incorrect HTTP method if restricted on the webhook side (not specified here).  
    - Webhook URL exposure or security concerns if publicly accessible.  
  - **Version Requirements:** None.  

#### 1.2 Weather Data Retrieval

- **Overview:**  
  This block queries the OpenWeatherMap API to retrieve current weather details for the city received from the webhook.

- **Nodes Involved:**  
  - OpenWeatherMap

- **Node Details:**  
  - **Node Name:** OpenWeatherMap  
  - **Type:** OpenWeatherMap API node  
  - **Configuration:**  
    - City Name: Dynamically set using expression referencing the webhook input: `={{$node["Webhook"].json["query"]["city"]}}`.  
    - Credentials: Requires valid OpenWeatherMap API credentials configured in n8n.  
  - **Input / Output:**  
    - Input: Receives city name from Webhook node.  
    - Output: Provides weather data JSON including temperature, weather description, and more.  
  - **Potential Failures:**  
    - Invalid or missing API credentials leading to authentication errors.  
    - API rate limits exceeded or downtime.  
    - Invalid city names resulting in API errors or empty responses.  
    - Network timeouts or connectivity issues.  
  - **Version Requirements:** None.  
  - **Notes:** This node depends on a properly configured OpenWeatherMap API credential in n8n.  

#### 1.3 Data Formatting

- **Overview:**  
  This block extracts relevant information (temperature and weather description) from the OpenWeatherMap response and formats it for returning to the user.

- **Nodes Involved:**  
  - Set

- **Node Details:**  
  - **Node Name:** Set  
  - **Type:** Data Transformation (Set)  
  - **Configuration:**  
    - Keeps only the fields defined below (`keepOnlySet` is true).  
    - Defines two output fields:  
      - `temp`: Extracted from `main.temp` field of the OpenWeatherMap response.  
      - `description`: Extracted from the first element of the `weather` array's `description` property.  
    - Expressions used:  
      - `={{$node["OpenWeatherMap"].json["main"]["temp"]}}`  
      - `={{$node["OpenWeatherMap"].json["weather"][0]["description"]}}`  
  - **Input / Output:**  
    - Input: Receives full weather JSON from OpenWeatherMap node.  
    - Output: Sends simplified JSON containing only temperature and description.  
  - **Potential Failures:**  
    - Missing expected fields in API response if city lookup failed or API changed.  
    - Expression errors if input data structure differs.  
  - **Version Requirements:** None.  

---

### 3. Summary Table

| Node Name       | Node Type            | Functional Role          | Input Node(s) | Output Node(s) | Sticky Note                                  |
|-----------------|----------------------|-------------------------|---------------|----------------|----------------------------------------------|
| Webhook         | HTTP Webhook         | Receive city input      | -             | OpenWeatherMap |                                              |
| OpenWeatherMap  | OpenWeatherMap API   | Retrieve weather data   | Webhook       | Set            | Requires configured OpenWeatherMap credential |
| Set             | Set (Data Transform) | Format response data    | OpenWeatherMap| -              |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Set the webhook path to a unique identifier (e.g., "45690b6a-2b01-472d-8839-5e83a74858e5").  
   - Configure response mode to `lastNode` and response data to `allEntries`.  
   - This node will receive HTTP requests with a query parameter named `city`.  

2. **Create OpenWeatherMap Node**  
   - Type: OpenWeatherMap  
   - Connect Webhook node's output to this node's input.  
   - In parameters, set `cityName` to expression: `{{$node["Webhook"].json["query"]["city"]}}`.  
   - Configure OpenWeatherMap credentials with a valid API key. You must create or import these credentials in n8n.  

3. **Create Set Node**  
   - Type: Set  
   - Connect OpenWeatherMap node's output to this node's input.  
   - Enable `keepOnlySet` to true to output only selected fields.  
   - Define two fields under "Values":  
     - Name: `temp` - Value: `={{$node["OpenWeatherMap"].json["main"]["temp"]}}`  
     - Name: `description` - Value: `={{$node["OpenWeatherMap"].json["weather"][0]["description"]}}`  

4. **Finalizing Workflow**  
   - Ensure the nodes are connected: Webhook → OpenWeatherMap → Set.  
   - Activate the workflow.  
   - Test by sending an HTTP GET request to the webhook URL including a `city` query parameter, for example:  
     `https://<your-n8n-domain>/webhook/45690b6a-2b01-472d-8839-5e83a74858e5?city=London`  
   - The response will contain JSON with temperature and weather description for the specified city.  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                    |
|------------------------------------------------------------------------------|---------------------------------------------------|
| Ensure that the OpenWeatherMap API key is valid and has sufficient quota.    | https://openweathermap.org/api                     |
| The workflow returns the last node's output as the HTTP response automatically. | n8n webhook response mode documentation            |
| For security, consider restricting webhook access or adding authentication.   | n8n webhook security best practices                |