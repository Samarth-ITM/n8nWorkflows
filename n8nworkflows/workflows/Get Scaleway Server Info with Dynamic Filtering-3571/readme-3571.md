Get Scaleway Server Info with Dynamic Filtering

https://n8nworkflows.xyz/workflows/get-scaleway-server-info-with-dynamic-filtering-3571


# Get Scaleway Server Info with Dynamic Filtering

### 1. Workflow Overview

This workflow, titled **Get Scaleway Server Info with Dynamic Filtering**, is designed to retrieve and filter server information from Scaleway cloud infrastructure. It targets developers, system administrators, and DevOps engineers who need to quickly access detailed server data across multiple zones and server types (instances and baremetal). The workflow supports dynamic filtering by tags, server name, public IP, or zone, enabling integration into monitoring, inventory, or automation systems.

The workflow logic is structured into the following functional blocks:

- **1.1 Input Reception:**  
  Receives HTTP POST requests via a secured webhook with search criteria specifying how to filter the server data.

- **1.2 Configuration and Preparation:**  
  Extracts input parameters, sets authentication tokens, and defines zones for instances and baremetal servers.

- **1.3 Server Data Collection:**  
  Iterates over configured zones to fetch server data from Scaleway’s API endpoints for both instances and baremetal servers.

- **1.4 Data Aggregation and Normalization:**  
  Processes and consolidates raw API responses into a uniform server data format with key attributes.

- **1.5 Dynamic Filtering:**  
  Routes the aggregated data to filtering routines based on the requested filter type (`tags`, `name`, `public_ip`, or `zone`).

- **1.6 Response Handling:**  
  Returns filtered results or error messages via webhook responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow on receiving an HTTP POST request with basic authentication. It captures the search parameters for filtering server data.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger  
    - Role: Entry point for the workflow, listens for POST requests at a specified path with basicAuth security.  
    - Configuration:  
      - HTTP Method: POST  
      - Authentication: Basic Authentication (credentials configured)  
      - Response Mode: Response node (delays response until workflow completes)  
    - Inputs: External HTTP POST request with JSON body containing `search_by` and `search`.  
    - Outputs: Passes incoming request data to the next node.  
    - Edge Cases: Authentication failure, invalid or missing JSON body, unsupported HTTP methods.

#### 2.2 Configuration and Preparation

- **Overview:**  
  Extracts and sets key parameters from the webhook input, including the search criteria, Scaleway API token, and zone lists for instances and baremetal servers.

- **Nodes Involved:**  
  - Edit Fields  
  - Split Out ZONE_INSTANCE

- **Node Details:**

  - **Edit Fields**  
    - Type: Set node  
    - Role: Extracts `search_by` and `search` from the webhook JSON body and sets workflow variables including the Scaleway API token and zone arrays.  
    - Configuration:  
      - Assigns:  
        - `search_by` = `{{$json.body.search_by}}`  
        - `search` = `{{$json.body.search}}`  
        - `Scaleway-X-Auth-Token` = placeholder string (to be replaced with a real token)  
        - `ZONE_INSTANCE` = array of instance zones (e.g., "fr-par-1", "nl-ams-1")  
        - `ZONE_BAREMETAL` = array of baremetal zones (stringified array)  
    - Inputs: Data from Webhook node  
    - Outputs: Passes enriched data downstream  
    - Edge Cases: Missing or malformed input fields, token not replaced with real value.

  - **Split Out ZONE_INSTANCE**  
    - Type: Split Out  
    - Role: Splits the `ZONE_INSTANCE` array into individual items to process each zone separately.  
    - Configuration: Splits on field `ZONE_INSTANCE`  
    - Inputs: From Edit Fields  
    - Outputs: One item per zone string  
    - Edge Cases: Empty or invalid zone list.

#### 2.3 Server Data Collection

- **Overview:**  
  Iterates over each instance zone, fetches server data from Scaleway’s instance API, and conditionally fetches baremetal server data if the zone is included in the baremetal zones list.

- **Nodes Involved:**  
  - Loop Over Zone Instance  
  - Get scw instance by zone  
  - If ZONE_BAREMETAL in ZONE_INSTANCE  
  - Get scw baremetal by zone

- **Node Details:**

  - **Loop Over Zone Instance**  
    - Type: Split In Batches  
    - Role: Iterates over each zone output from Split Out ZONE_INSTANCE to process sequentially.  
    - Configuration: Default batch size (processes one zone at a time)  
    - Inputs: From Split Out ZONE_INSTANCE  
    - Outputs: Passes current zone to next nodes  
    - Edge Cases: Large zone lists may increase runtime.

  - **Get scw instance by zone**  
    - Type: HTTP Request  
    - Role: Fetches instance server data for the current zone from Scaleway API.  
    - Configuration:  
      - URL: `https://api.scaleway.com/instance/v1/zones/{{ $json.ZONE_INSTANCE }}/servers` (zone interpolated)  
      - Method: GET  
      - Headers:  
        - `X-Auth-Token`: from Edit Fields node  
        - `Content-Type`: application/json  
    - Inputs: Current zone from Loop Over Zone Instance  
    - Outputs: JSON response with server instances  
    - Edge Cases: API authentication errors, network timeouts, empty responses.

  - **If ZONE_BAREMETAL in ZONE_INSTANCE**  
    - Type: If  
    - Role: Checks if the current zone is in the baremetal zones list to decide whether to fetch baremetal servers.  
    - Configuration:  
      - Condition: Check if `ZONE_BAREMETAL` array contains current zone string  
    - Inputs: Current zone from Loop Over Zone Instance  
    - Outputs:  
      - True: proceeds to Get scw baremetal by zone  
      - False: loops back to Loop Over Zone Instance (skips baremetal fetch)  
    - Edge Cases: Misconfigured zone lists causing missed data.

  - **Get scw baremetal by zone**  
    - Type: HTTP Request  
    - Role: Fetches baremetal server data for the current zone from Scaleway API.  
    - Configuration:  
      - URL: `https://api.scaleway.com/baremetal/v1/zones/{{ $json.ZONE_INSTANCE }}/servers`  
      - Method: GET  
      - Headers: same as instance request  
    - Inputs: Current zone from If node  
    - Outputs: JSON response with baremetal servers  
    - Edge Cases: Same as instance request.

#### 2.4 Data Aggregation and Normalization

- **Overview:**  
  Aggregates all server data collected from multiple zones and server types, normalizes the data structure, and extracts key server attributes for filtering.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Processes all incoming server data items, extracts relevant fields, and returns a unified array of server objects with normalized attributes.  
    - Key Logic:  
      - Validates input is an array  
      - Extracts fields: `name`, `tags` (joined string or "No tags"), `public_ip` (IPv4 prioritized, else IPv6, else "No IP"), `type` (commercial_type or offer_name), `state` (state or status), `zone`, `user` (from multiple possible fields or defaults to "root")  
      - Handles both instance and baremetal server data formats  
    - Inputs: Multiple items from Loop Over Zone Instance and API requests  
    - Outputs: Array of normalized server JSON objects  
    - Edge Cases: Unexpected API response formats, missing fields, empty server lists.

#### 2.5 Dynamic Filtering

- **Overview:**  
  Routes the normalized server data to filtering routines based on the `search_by` parameter, filtering the server list by tags, name, public IP, or zone. If the filter is invalid, returns an error.

- **Nodes Involved:**  
  - Switch  
  - Code search Tags  
  - Code search Name  
  - Code search public_ip  
  - Code search zone  
  - Respond Error

- **Node Details:**

  - **Switch**  
    - Type: Switch  
    - Role: Routes data based on the exact value of `search_by` extracted earlier.  
    - Configuration:  
      - Cases for `tags`, `name`, `public_ip`, `zone` (case-sensitive, strict validation)  
      - Fallback output for invalid or missing `search_by`  
    - Inputs: Normalized server data from Code node  
    - Outputs: To respective filtering Code nodes or error response  
    - Edge Cases: Case sensitivity may cause unexpected routing; missing or null `search_by`.

  - **Code search Tags**  
    - Type: Code  
    - Role: Filters servers whose `tags` string includes the `search` keyword.  
    - Inputs: Server list from Code node  
    - Outputs: Filtered server list  
    - Edge Cases: Tags field missing or empty, case sensitivity in matching.

  - **Code search Name**  
    - Type: Code  
    - Role: Filters servers whose `name` includes the `search` keyword.  
    - Inputs/Outputs: Same as above  
    - Edge Cases: Missing name fields.

  - **Code search public_ip**  
    - Type: Code  
    - Role: Filters servers whose `public_ip` includes the `search` keyword.  
    - Inputs/Outputs: Same  
    - Edge Cases: Servers with "No IP" string.

  - **Code search zone**  
    - Type: Code  
    - Role: Intended to filter servers by zone, but currently filters by `public_ip` (likely a bug).  
    - Inputs/Outputs: Same  
    - Edge Cases: Incorrect filtering logic; should be updated to filter by `zone`.

  - **Respond Error**  
    - Type: Respond to Webhook  
    - Role: Returns a JSON error message if `search_by` is invalid, listing valid filter options.  
    - Inputs: From Switch fallback output  
    - Outputs: Error JSON response  
    - Edge Cases: None.

#### 2.6 Response Handling

- **Overview:**  
  Sends the filtered server data or error messages back to the requester via webhook response nodes.

- **Nodes Involved:**  
  - Respond to Webhook (multiple instances)

- **Node Details:**

  - **Respond to Webhook1, Respond to Webhook2, Respond to Webhook3, Respond to Webhook, Respond to Webhook4**  
    - Type: Respond to Webhook  
    - Role: Return filtered server data corresponding to each filter type (`name`, `public_ip`, `zone`, `tags`) or fallback.  
    - Configuration: Responds with all incoming items as JSON.  
    - Inputs: From respective Code search nodes or Switch fallback  
    - Outputs: HTTP response to original webhook request  
    - Edge Cases: Large payloads, response timeouts.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                                | Input Node(s)               | Output Node(s)                         | Sticky Note                                                                                                          |
|---------------------------|------------------------|-----------------------------------------------|-----------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook Trigger        | Receives HTTP POST with search criteria       | External HTTP request       | Edit Fields                           |                                                                                                                      |
| Edit Fields               | Set                    | Extracts search params, sets token and zones  | Webhook                    | Split Out ZONE_INSTANCE                |                                                                                                                      |
| Split Out ZONE_INSTANCE    | Split Out              | Splits instance zones into individual items   | Edit Fields                | Loop Over Zone Instance                |                                                                                                                      |
| Loop Over Zone Instance    | Split In Batches       | Iterates over zones for data fetching          | Split Out ZONE_INSTANCE     | Code, Get scw instance by zone        |                                                                                                                      |
| Get scw instance by zone   | HTTP Request           | Fetches instance servers from Scaleway API    | Loop Over Zone Instance     | If ZONE_BAREMETAL in ZONE_INSTANCE    |                                                                                                                      |
| If ZONE_BAREMETAL in ZONE_INSTANCE | If             | Checks if zone includes baremetal servers     | Get scw instance by zone    | Get scw baremetal by zone, Loop Over Zone Instance |                                                                                                                      |
| Get scw baremetal by zone  | HTTP Request           | Fetches baremetal servers from Scaleway API   | If ZONE_BAREMETAL in ZONE_INSTANCE | Loop Over Zone Instance           |                                                                                                                      |
| Code                      | Code                   | Aggregates and normalizes server data          | Loop Over Zone Instance, Get scw baremetal by zone | Switch                        |                                                                                                                      |
| Switch                    | Switch                 | Routes data based on `search_by` filter type   | Code                       | Code search Tags, Name, public_ip, zone, Respond Error |                                                                                                                      |
| Code search Tags          | Code                   | Filters servers by tags                         | Switch                     | Respond to Webhook                    |                                                                                                                      |
| Code search Name          | Code                   | Filters servers by name                         | Switch                     | Respond to Webhook1                   |                                                                                                                      |
| Code search public_ip     | Code                   | Filters servers by public IP                    | Switch                     | Respond to Webhook2                   |                                                                                                                      |
| Code search zone          | Code                   | Filters servers by zone (bug: filters by public_ip) | Switch                 | Respond to Webhook3                   |                                                                                                                      |
| Respond Error             | Respond to Webhook     | Returns error JSON for invalid `search_by`     | Switch (fallback)           | -                                   |                                                                                                                      |
| Respond to Webhook        | Respond to Webhook     | Returns filtered data for tags                  | Code search Tags            | -                                   |                                                                                                                      |
| Respond to Webhook1       | Respond to Webhook     | Returns filtered data for name                  | Code search Name            | -                                   |                                                                                                                      |
| Respond to Webhook2       | Respond to Webhook     | Returns filtered data for public_ip             | Code search public_ip       | -                                   |                                                                                                                      |
| Respond to Webhook3       | Respond to Webhook     | Returns filtered data for zone                   | Code search zone            | -                                   |                                                                                                                      |
| Respond to Webhook4       | Respond to Webhook     | Fallback response for Switch node                | Switch                     | -                                   |                                                                                                                      |
| Get Scalway Machines      | HTTP Request           | External usage example node to call webhook     | -                         | Loop Over Items                      |                                                                                                                      |
| Loop Over Items           | Split In Batches       | Processes batches of items from Get Scalway Machines | Get Scalway Machines     | -                                   |                                                                                                                      |
| Sticky Note               | Sticky Note            | Technical documentation content                  | -                         | -                                   | Contains detailed workflow description and operation overview                                                       |
| Sticky Note1              | Sticky Note            | Usage instructions for external app integration | -                         | -                                   | Explains how to send POST requests and process results                                                              |
| Sticky Note2              | Sticky Note            | Instructions for replacing Scaleway API token   | -                         | -                                   | Provides link to Scaleway API token creation documentation: https://www.scaleway.com/en/developers/api/              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: unique identifier (e.g., `a6767312-3a4c-4819-b4fe-a03c9e0ade5c`)  
   - Authentication: Basic Auth (configure credentials with username/password)  
   - Response Mode: Response Node (delays response until workflow finishes)  

2. **Create Set Node (Edit Fields)**  
   - Type: Set  
   - Add fields:  
     - `search_by` (string): Expression `{{$json.body.search_by}}`  
     - `search` (string): Expression `{{$json.body.search}}`  
     - `Scaleway-X-Auth-Token` (string): Replace placeholder with your Scaleway API token  
     - `ZONE_INSTANCE` (array): Zones for instances, e.g., `["fr-par-1", "fr-par-2", "fr-par-3", "nl-ams-1", "nl-ams-2", "nl-ams-3", "pl-waw-1", "pl-waw-2", "pl-waw-3"]`  
     - `ZONE_BAREMETAL` (string): JSON string of baremetal zones, e.g., `["fr-par-1", "fr-par-2", "nl-ams-1", "nl-ams-2", "pl-waw-2", "pl-waw-3"]`  

3. **Create Split Out Node (Split Out ZONE_INSTANCE)**  
   - Type: Split Out  
   - Field to split out: `ZONE_INSTANCE`  

4. **Create Split In Batches Node (Loop Over Zone Instance)**  
   - Type: Split In Batches  
   - Default batch size (1)  

5. **Create HTTP Request Node (Get scw instance by zone)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.scaleway.com/instance/v1/zones/{{ $json.ZONE_INSTANCE }}/servers`  
   - Headers:  
     - `X-Auth-Token`: Expression `{{$json["Scaleway-X-Auth-Token"]}}`  
     - `Content-Type`: `application/json`  

6. **Create If Node (If ZONE_BAREMETAL in ZONE_INSTANCE)**  
   - Type: If  
   - Condition: Check if `ZONE_BAREMETAL` array contains current zone string  
   - Use expression:  
     - Left Value: `{{$json.ZONE_BAREMETAL}}` (parsed as array)  
     - Operator: Contains  
     - Right Value: `{{$json.ZONE_INSTANCE}}`  

7. **Create HTTP Request Node (Get scw baremetal by zone)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.scaleway.com/baremetal/v1/zones/{{ $json.ZONE_INSTANCE }}/servers`  
   - Headers: same as instance request  

8. **Create Code Node (Code)**  
   - Type: Code  
   - JavaScript code:  
     - Extracts and normalizes server data from all incoming items  
     - Implements helper functions `extractServers`, `getPublicIPs`, `getUser`  
     - Returns array of normalized server objects with keys: `name`, `tags`, `public_ip`, `type`, `state`, `zone`, `user`  

9. **Create Switch Node (Switch)**  
   - Type: Switch  
   - Property to check: `{{$json.search_by}}` (from Edit Fields node)  
   - Cases:  
     - `tags`  
     - `name`  
     - `public_ip`  
     - `zone`  
   - Fallback: to error response node  

10. **Create Code Nodes for Filtering:**  
    - **Code search Tags:** Filters servers where `tags` includes `search` keyword  
    - **Code search Name:** Filters servers where `name` includes `search` keyword  
    - **Code search public_ip:** Filters servers where `public_ip` includes `search` keyword  
    - **Code search zone:** Filters servers where `zone` includes `search` keyword (correct the existing bug)  

11. **Create Respond to Webhook Nodes:**  
    - One for each filter type (`tags`, `name`, `public_ip`, `zone`)  
    - One for error response  
    - Configure to respond with all incoming items as JSON  

12. **Connect Nodes:**  
    - Webhook → Edit Fields → Split Out ZONE_INSTANCE → Loop Over Zone Instance  
    - Loop Over Zone Instance → Get scw instance by zone → If ZONE_BAREMETAL in ZONE_INSTANCE  
    - If True → Get scw baremetal by zone → Loop Over Zone Instance (loop continues)  
    - If False → Loop Over Zone Instance (loop continues)  
    - Loop Over Zone Instance and Get scw baremetal by zone → Code (aggregation) → Switch  
    - Switch → respective Code search nodes → respective Respond to Webhook nodes  
    - Switch fallback → Respond Error node  

13. **Credential Setup:**  
    - Configure Basic Auth credentials for Webhook node  
    - No special credentials for HTTP Request nodes; token passed in headers from Edit Fields node  

14. **Testing:**  
    - Replace Scaleway API token in Edit Fields node  
    - Send POST request to webhook URL with JSON body:  
      ```json
      {
        "search_by": "tags",
        "search": "Apiv1"
      }
      ```  
    - Verify filtered server list is returned  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| To ensure the workflow functions correctly, replace the placeholder `Scaleway-X-Auth-Token` in the Edit Fields node with your personal Scaleway API token. If you do not have a token, create one via the Scaleway console.       | https://www.scaleway.com/en/developers/api/                                                        |
| The workflow supports dynamic filtering by `tags`, `name`, `public_ip`, or `zone`. The `zone` filtering code node currently contains a bug where it filters by `public_ip` instead of `zone`. This should be corrected.          |                                                                                                    |
| The workflow uses basic authentication on the webhook endpoint to secure access. Ensure credentials are managed properly and not exposed publicly.                                                                             |                                                                                                    |
| The workflow can be integrated into external applications or other workflows by sending POST requests with the appropriate JSON payload to the webhook URL.                                                                    | Sticky Note1 content                                                                                 |
| The workflow aggregates data from multiple zones and server types, normalizing different API response formats into a unified structure for easier filtering and processing.                                                     | Sticky Note content                                                                                  |

---

This documentation provides a comprehensive reference to understand, reproduce, and modify the **Get Scaleway Server Info with Dynamic Filtering** workflow, including its architecture, node details, and integration considerations.