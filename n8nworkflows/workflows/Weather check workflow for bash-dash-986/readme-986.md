Weather check workflow for bash-dash

https://n8nworkflows.xyz/workflows/weather-check-workflow-for-bash-dash-986


# Weather check workflow for bash-dash

### 1. Workflow Overview

This workflow provides current weather information for a specified city or a default city if none is provided. It is designed to serve as a backend for the [bash-dash](https://github.com/n8n-io/bash-dash) command-line dashboard, enabling users to retrieve real-time weather data via a simple HTTP request.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Receives the incoming HTTP request with an optional city query parameter.
- **1.2 City Determination:** Extracts the city name from the request or assigns a default city.
- **1.3 Weather Retrieval and Response Creation:** Fetches weather data from OpenWeatherMap API and formats a user-friendly response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives an HTTP GET request at the `/weather` webhook endpoint. It captures query parameters that may specify a city name.

**Nodes Involved:**  
- Webhook GET

**Node Details:**

- **Webhook GET**  
  - **Type:** Webhook (HTTP Inbound)  
  - **Role:** Entry point for external requests; listens on `/weather` path for GET requests.  
  - **Configuration:**  
    - Path: `weather`  
    - Response Mode: `lastNode` (response is taken from the last executed node)  
    - Response Property Name: `data` (used internally to hold the response data)  
  - **Key Expressions / Variables:**  
    - Accesses query parameter via `$json["query"]["parameter"]` in downstream nodes.  
  - **Input/Output:**  
    - Input: HTTP request  
    - Output: JSON object containing query parameters  
  - **Potential Failures:**  
    - Invalid HTTP request or method  
    - Webhook not reachable if port or URL is misconfigured  
  - **Version Requirements:** Compatible with n8n v0.148.0+ (webhook node stable)  
  - **Sub-workflow:** None

#### 1.2 City Determination

**Overview:**  
Extracts the city name from the incoming request query parameter. If not provided, assigns a default city (`berlin,de` by default). This value is passed forward for weather data retrieval.

**Nodes Involved:**  
- Set City

**Node Details:**

- **Set City**  
  - **Type:** Set Node (data transformation)  
  - **Role:** Defines the `city` variable, either from query parameter or default.  
  - **Configuration:**  
    - Sets a string field named `city` with the expression:  
      `{{$json["query"]["parameter"] || 'berlin,de'}}`  
    - This means if the query parameter is present, use it; otherwise, default to `"berlin,de"`.  
  - **Input/Output:**  
    - Input: JSON object from Webhook GET node  
    - Output: JSON object with a new `city` field  
  - **Potential Failures:**  
    - Expression failure if `$json["query"]["parameter"]` is undefined or malformed (however, fallback is provided)  
  - **Version Requirements:** None special  
  - **Sub-workflow:** None

#### 1.3 Weather Retrieval and Response Creation

**Overview:**  
Requests current weather data for the specified city from OpenWeatherMap and formats a concise, human-readable message including temperature and feels-like temperature.

**Nodes Involved:**  
- OpenWeatherMap  
- Create Response

**Node Details:**

- **OpenWeatherMap**  
  - **Type:** OpenWeatherMap node (API integration)  
  - **Role:** Fetches current weather data based on city name.  
  - **Configuration:**  
    - City Name: Bound to `{{$json["city"]}}` (city determined in previous node)  
    - Language: English (`en`)  
    - Credentials: Requires valid OpenWeatherMap API key (configured via n8n credentials)  
  - **Input/Output:**  
    - Input: JSON with `city` field  
    - Output: JSON containing weather data such as temperature, feels_like, city name, etc.  
  - **Potential Failures:**  
    - Authentication errors if API key is invalid or missing  
    - API rate limits or timeouts  
    - Invalid city names leading to API errors  
  - **Version Requirements:** Requires n8n with OpenWeatherMap node support (v0.140.0+)  
  - **Sub-workflow:** None

- **Create Response**  
  - **Type:** Set Node (data transformation)  
  - **Role:** Constructs a user-friendly string describing the weather status.  
  - **Configuration:**  
    - Sets a string field `data` with the expression:  
      ```
      =It has {{$json["main"]["temp"]}}°C and feels like {{$json["main"]["feels_like"]}}°C in {{$json["name"]}}
      ```  
    - This formats temperature and city name into a readable sentence.  
  - **Input/Output:**  
    - Input: JSON from OpenWeatherMap node with weather details  
    - Output: JSON with single `data` string field ready for HTTP response  
  - **Potential Failures:**  
    - Expression failure if expected fields (`main.temp`, `main.feels_like`, `name`) are missing  
  - **Version Requirements:** None special  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name       | Node Type          | Functional Role                    | Input Node(s)  | Output Node(s)   | Sticky Note                                                                                 |
|-----------------|--------------------|----------------------------------|----------------|------------------|---------------------------------------------------------------------------------------------|
| Webhook GET     | Webhook            | Receives HTTP request at /weather| (Start)        | Set City         |                                                                                             |
| Set City        | Set                | Extracts city or sets default    | Webhook GET    | OpenWeatherMap   |                                                                                             |
| OpenWeatherMap  | OpenWeatherMap     | Fetches current weather data     | Set City       | Create Response  | Requires valid OpenWeatherMap API credentials                                               |
| Create Response | Set                | Formats weather info into string | OpenWeatherMap | (Final response) |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook GET Node**  
   - Type: Webhook  
   - Name: `Webhook GET`  
   - HTTP Method: GET  
   - Path: `weather`  
   - Response Mode: `lastNode` (response is taken from the last executing node)  
   - Response Property Name: `data`  
   - No credentials required  
   - Position on canvas (optional): [500, 300]

2. **Create the Set City Node**  
   - Type: Set  
   - Name: `Set City`  
   - Add a string field named `city`  
   - Set its value to expression:  
     ```javascript
     {{$json["query"]["parameter"] || 'berlin,de'}}
     ```  
   - This captures the city parameter from the query or defaults to `"berlin,de"`  
   - Connect `Webhook GET` node's output to this node's input  
   - Position: [700, 300]

3. **Create the OpenWeatherMap Node**  
   - Type: OpenWeatherMap  
   - Name: `OpenWeatherMap`  
   - Credentials: Create or select existing OpenWeatherMap API credentials (requires API key)  
   - Set Parameters:  
     - City Name: Expression `{{$json["city"]}}` (from previous node)  
     - Language: `en`  
   - Connect `Set City` node output to this node input  
   - Position: [900, 300]

4. **Create the Create Response Node**  
   - Type: Set  
   - Name: `Create Response`  
   - Add a string field named `data`  
   - Set its value to this expression:  
     ```javascript
     =It has {{$json["main"]["temp"]}}°C and feels like {{$json["main"]["feels_like"]}}°C in {{$json["name"]}}
     ```  
   - Connect `OpenWeatherMap` node output to this node input  
   - Position: [1100, 300]

5. **Connect the nodes in order:**  
   `Webhook GET` → `Set City` → `OpenWeatherMap` → `Create Response`

6. **Save and activate the workflow.**

7. **Configure bash-dash (optional):**  
   - Set the command as:  
     ```bash
     commands[weather]="http://localhost:5678/webhook/weather"
     ```  
   - This will call the webhook and return the weather string.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                   |
|------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Default city is set to "berlin,de" but can be changed in "Set City" node    | Adjust default city by modifying the `city` value in the "Set City" node expression              |
| Example usage: `weather london`                                              | Command syntax for bash-dash invocation                                                         |
| Workflow intended to integrate with [bash-dash](https://github.com/n8n-io/bash-dash) | Project link for bash-dash CLI dashboard                                                        |
| Requires valid OpenWeatherMap API key                                       | Obtain API key at https://openweathermap.org/api                                                |

---

This documentation fully covers the workflow structure, node configurations, and setup instructions, enabling reproduction, debugging, and extension.