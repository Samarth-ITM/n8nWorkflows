IP Geolocation & HTTP Port Scanning with Google Sheets

https://n8nworkflows.xyz/workflows/ip-geolocation---http-port-scanning-with-google-sheets-8674


# IP Geolocation & HTTP Port Scanning with Google Sheets

---

### 1. Workflow Overview

This workflow automates the enrichment and monitoring of IP addresses entered into a Google Sheets document. When a new IP address is added to the sheet, the workflow performs two main tasks:  
1. **Geolocation Data Enrichment:** It fetches detailed IP geolocation information such as ISP, city, country, latitude, longitude, and organization using an external API.  
2. **HTTP Port Scanning:** It tests connectivity on common HTTP ports (80, 443, 8080, 8000, 3000) by attempting HTTP requests to the IP on each port and records whether each port is open or closed.

**Target Use Cases:**  
- Network monitoring and security auditing.  
- Maintaining an IP intelligence database.  
- Security research or automated asset inventory.

**Logical Blocks:**  
- **1.1 Input Reception:** Triggered by new IP rows added to Google Sheets.  
- **1.2 IP Geolocation Retrieval:** Calls external API to get IP metadata.  
- **1.3 Data Update in Google Sheets:** Updates the sheet with geolocation results.  
- **1.4 HTTP Ports Definition and Splitting:** Defines ports to scan and splits them into individual requests.  
- **1.5 HTTP Port Connectivity Testing:** Performs HTTP requests to each port on the IP.  
- **1.6 Results Aggregation and Update:** Aggregates scan results and updates the sheet with port status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow whenever a new row containing an IP address is added to the specified Google Sheets document.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**  

  - **Google Sheets Trigger**  
    - Type: Trigger node, listens for new rows added to Google Sheets.  
    - Configuration: Watches the sheet with ID `19MSjNyjzs1FeRI5_QLiVk8Hi9JNt5HD1bNulfpB6SzI` and the first sheet (`gid=0`). Polls every minute.  
    - Credentials: Uses OAuth2 credentials for Google Sheets access.  
    - Inputs: None (trigger).  
    - Outputs: Emits the new row data that contains the IP address.  
    - Edge Cases:  
      - Possible failures include credential expiration or revoked access.  
      - New rows without valid IP addresses may trigger the flow but could cause downstream errors.  
      - Rate limits on Google API polling.  
    - Sticky Note: Explains this node triggers on new IP address rows.

#### 1.2 IP Geolocation Retrieval

- **Overview:**  
  Fetches geolocation data for the IP address from the free IP geolocation API at `ip-api.com`.

- **Nodes Involved:**  
  - GetIP_Info

- **Node Details:**  

  - **GetIP_Info**  
    - Type: HTTP Request node.  
    - Configuration: Performs GET request to `http://ip-api.com/json/{{ $json.IP }}`, dynamically injecting the IP from the trigger data.  
    - Inputs: Receives IP address JSON from Google Sheets Trigger.  
    - Outputs: JSON response with fields like query (IP), isp, lat, lon, org, city, country.  
    - Version: n8n HTTP Request v4.2.  
    - Edge Cases:  
      - API failures due to rate limit or downtime.  
      - Invalid IP addresses causing error responses or empty data.  
      - Network timeouts.  
    - Sticky Note: "Get IP Geolocation Data."

#### 1.3 Data Update in Google Sheets

- **Overview:**  
  Updates the original Google Sheets row with the enriched geolocation data.

- **Nodes Involved:**  
  - Update_IP_Info_row  
  - Edit Fields

- **Node Details:**  

  - **Update_IP_Info_row**  
    - Type: Google Sheets node (operation: update).  
    - Configuration: Matches rows by IP column and updates columns: IP, ISP, Lat, Lon, Org, City, Country.  
    - Inputs: Receives enriched IP data from GetIP_Info node.  
    - Outputs: Passes data forward.  
    - Credentials: Uses Google Sheets OAuth2 credentials.  
    - Edge Cases:  
      - Matching failure if IP column is missing or duplicated.  
      - Google API errors or quota limits.  
      - Data format mismatches.  

  - **Edit Fields**  
    - Type: Set node to prepare port scan data.  
    - Configuration: Sets a fixed array for `ports` field: `[80,443,8080,8000,3000]` alongside passing IP field.  
    - Inputs: Data from Update_IP_Info_row.  
    - Outputs: Passes IP with ports array.  
    - Edge Cases: None significant here.  
    - Sticky Note: Explains defining HTTP ports list and converting array to items.

#### 1.4 HTTP Ports Definition and Splitting

- **Overview:**  
  Converts the array of ports into individual items to enable scanning each port separately.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**  

  - **Split Out**  
    - Type: Split Out node.  
    - Configuration: Splits the `ports` array field into multiple items, including the IP field in each item.  
    - Inputs: Receives IP and ports array from Edit Fields.  
    - Outputs: Emits individual items each containing one port and the IP.  
    - Edge Cases:  
      - Empty or missing ports array would cause no outputs.  
    - Sticky Note: Notes converting array to list for HTTP requests.

#### 1.5 HTTP Port Connectivity Testing

- **Overview:**  
  Performs HTTP GET requests to test if the IP address accepts connections on each specified port.

- **Nodes Involved:**  
  - CheckHttpPort

- **Node Details:**  

  - **CheckHttpPort**  
    - Type: HTTP Request node.  
    - Configuration: Requests URL `http://{{ $json.IP }}:{{$json.ports}}/` with 10-second timeout, allowing unauthorized certificates.  
    - Inputs: Individual IP-port items from Split Out.  
    - Outputs: Result or error for each port test.  
    - Error Handling: Continues workflow even on request errors (e.g., connection refused).  
    - Edge Cases:  
      - Timeouts if port is filtered or host unreachable.  
      - SSL certificate warnings are ignored for HTTPS ports.  
      - Network errors or malformed URLs.  
    - Sticky Note: Notes testing HTTP port connectivity.

#### 1.6 Results Aggregation and Update

- **Overview:**  
  Aggregates all port test results into a single JSON object indicating which ports are open, then updates the Google Sheet with the results.

- **Nodes Involved:**  
  - PutAll_in_OneItem  
  - Update_HTTP_Ports_State

- **Node Details:**  

  - **PutAll_in_OneItem**  
    - Type: Code node (JavaScript).  
    - Configuration:  
      - Iterates over all HTTP request results.  
      - For each port, checks if an error message starts with an HTTP status code (regex).  
      - If error indicates HTTP code, or no error, marks port as open (`true`), else `false`.  
      - Returns an object with port numbers as keys and booleans as values.  
    - Inputs: Multiple HTTP request results from CheckHttpPort.  
    - Outputs: One aggregated result object.  
    - Edge Cases:  
      - Errors not matching regex treated as port closed.  
      - Unexpected error message formats may cause incorrect status.  

  - **Update_HTTP_Ports_State**  
    - Type: Google Sheets node (update operation).  
    - Configuration: Matches rows by IP and updates columns for each port with boolean status.  
    - Inputs: Aggregated port scan result from PutAll_in_OneItem and IP from Split Out (first item).  
    - Outputs: None further (end of workflow).  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Edge Cases:  
      - Sheet update failures or mismatched IP keys.  
      - Data type mismatch if booleans are not accepted as strings.

---

### 3. Summary Table

| Node Name              | Node Type                  | Functional Role                        | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                            |
|------------------------|----------------------------|--------------------------------------|-----------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger   | Google Sheets Trigger       | Trigger on new IP row added          | None                  | GetIP_Info              | triggers whenever a new row containing an IP address is added to your Google Sheet                                       |
| GetIP_Info             | HTTP Request                | Fetch IP geolocation data             | Google Sheets Trigger | Update_IP_Info_row       | **Get IP Geolocation Data**                                                                                            |
| Update_IP_Info_row      | Google Sheets (update)      | Update sheet row with geolocation    | GetIP_Info            | Edit Fields             |                                                                                                                        |
| Edit Fields            | Set                        | Define ports array for scanning       | Update_IP_Info_row     | Split Out               | **Define HTTP ports list for scan** Convert the array to items list in order to execute http request for each port     |
| Split Out              | Split Out                  | Split ports array into separate items | Edit Fields            | CheckHttpPort           |                                                                                                                        |
| CheckHttpPort          | HTTP Request                | Test HTTP connectivity per port      | Split Out             | PutAll_in_OneItem       | **Test HTTP Port Connectivity**                                                                                        |
| PutAll_in_OneItem      | Code                       | Aggregate port scan results           | CheckHttpPort         | Update_HTTP_Ports_State  | **Aggregate Port Scan Results** Check is the port open for each port                                                   |
| Update_HTTP_Ports_State | Google Sheets (update)      | Update sheet row with port states    | PutAll_in_OneItem, Split Out (for IP) | None                    |                                                                                                                        |
| Sticky Note            | Sticky Note                 | Documentation and guidance            | None                  | None                    | ## Automate IP geolocation and HTTP port scanning with Google Sheets trigger This n8n template automatically enriches IP addresses with geolocation data and performs HTTP port scanning when new IPs are added to a Google Sheets document. Perfect for network monitoring, security research, or maintaining an IP intelligence database. **Sample Sheet** [Link](https://docs.google.com/spreadsheets/d/19MSjNyjzs1FeRI5_QLiVk8Hi9JNt5HD1bNulfpB6SzI/edit?usp=sharing) |
| Sticky Note1           | Sticky Note                 | Documentation for Google Sheets Trigger | None                  | None                    | triggers whenever a new row containing an IP address is added to your Google Sheet                                      |
| Sticky Note2           | Sticky Note                 | Documentation for GetIP_Info node    | None                  | None                    | **Get IP Geolocation Data**                                                                                            |
| Sticky Note3           | Sticky Note                 | Documentation for Edit Fields node   | None                  | None                    | **Define HTTP ports list for scan** Convert the array to items list in order to execute http request for each port     |
| Sticky Note4           | Sticky Note                 | Documentation for CheckHttpPort node | None                  | None                    | **Test HTTP Port Connectivity**                                                                                        |
| Sticky Note5           | Sticky Note                 | Documentation for PutAll_in_OneItem node | None                  | None                    | **Aggregate Port Scan Results** Check is the port open for each port                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node**  
   - Type: Google Sheets Trigger  
   - Credentials: Set up OAuth2 credentials with Google account access.  
   - Configuration:  
     - Document ID: `19MSjNyjzs1FeRI5_QLiVk8Hi9JNt5HD1bNulfpB6SzI`  
     - Sheet Name / GID: `gid=0` (Sheet1)  
     - Event: `rowAdded`  
     - Poll interval: every minute  

2. **Create HTTP Request node "GetIP_Info"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `http://ip-api.com/json/{{ $json.IP }}` (use expression to inject IP)  
   - No authentication required  
   - Timeout and retries: default  

3. **Create Google Sheets node "Update_IP_Info_row"**  
   - Type: Google Sheets (operation: update)  
   - Credentials: Use Google Sheets OAuth2 credentials.  
   - Document ID and Sheet Name same as trigger.  
   - Mapping: Match by column `IP`.  
   - Update columns: IP, ISP, Lat, Lon, Org, City, Country with corresponding data from `GetIP_Info` response fields.  

4. **Create Set node "Edit Fields"**  
   - Type: Set  
   - Set field `ports` to array: `[80,443,8080,8000,3000]`  
   - Include IP field from previous node output.  

5. **Create Split Out node "Split Out"**  
   - Type: Split Out  
   - Field to split out: `ports`  
   - Include other fields: IP  
   - Result: one item per port with IP and port field.  

6. **Create HTTP Request node "CheckHttpPort"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `http://{{ $json.IP }}:{{$json.ports }}/` (expression)  
   - Timeout: 10000 ms (10 seconds)  
   - Allow unauthorized certificates: true (to avoid SSL errors)  
   - Error handling: On error continue (to capture closed ports)  

7. **Create Code node "PutAll_in_OneItem"**  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```javascript
     function startsWithHttpCode(error) {
       return /^\s*\d{3}\b/.test(error);
     }

     const result = {};
     let index = 0;
     for (const item of $input.all()) {
       const portItem = $("Split Out").itemMatching(index).json;

       if(item.json.error)
         result[portItem.ports] = startsWithHttpCode(item.json.error.message);
       else
         result[portItem.ports] = true;
       index++;
     }

     return {result};
     ```
   - Purpose: Aggregate port scan results marking ports true (open) or false (closed).  

8. **Create Google Sheets node "Update_HTTP_Ports_State"**  
   - Type: Google Sheets (update)  
   - Credentials: Google Sheets OAuth2  
   - Document ID and Sheet Name same as trigger.  
   - Matching: by IP column.  
   - Columns to update: Port_80, Port_443, PORT_3000, Port_8000, Port_8080 with corresponding boolean values from aggregated result object.  

9. **Connect nodes in order:**  
   Google Sheets Trigger → GetIP_Info → Update_IP_Info_row → Edit Fields → Split Out → CheckHttpPort → PutAll_in_OneItem → Update_HTTP_Ports_State  

10. **Add sticky notes for documentation and clarity** (optional, for team collaboration).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Automate IP geolocation and HTTP port scanning with Google Sheets trigger. This n8n template automatically enriches IP addresses with geolocation data and performs HTTP port scanning when new IPs are added to a Google Sheets document. Perfect for network monitoring, security research, or maintaining an IP intelligence database. Sample Sheet: [Google Sheet Sample](https://docs.google.com/spreadsheets/d/19MSjNyjzs1FeRI5_QLiVk8Hi9JNt5HD1bNulfpB6SzI/edit?usp=sharing) | Workflow sticky note                                                                                 |

---

**Disclaimer:** The provided text is derived solely from an n8n automated workflow. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---