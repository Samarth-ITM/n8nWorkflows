Create Detailed Security Tickets from SentinelOne Threats with MITRE Analysis

https://n8nworkflows.xyz/workflows/create-detailed-security-tickets-from-sentinelone-threats-with-mitre-analysis-7312


# Create Detailed Security Tickets from SentinelOne Threats with MITRE Analysis

### 1. Workflow Overview

This workflow automates the creation of detailed security tickets in an Autotask system based on incoming SentinelOne threat alerts. It enriches the threat data with MITRE ATT&CK analysis, aligns threat intelligence with client company data, and systematically maps ticket fields before generating tickets for security incident tracking and response.

Logical blocks are structured as follows:

- **1.1 Input Reception:** Capture new SentinelOne threat data via webhook.
- **1.2 Threat Intelligence Extraction:** Parse and extract key threat details from the webhook payload.
- **1.3 Autotask Data Retrieval & Rate Limiting:** Sequentially retrieve Autotask users, client companies, and ticket field definitions with controlled pacing to avoid API rate limits.
- **1.4 Data Processing & Mapping:** Split and process client companies, parse ticket field options, and map client company data to ticket fields.
- **1.5 Ticket Creation:** Create security tickets in Autotask with enriched and mapped data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block waits for new threat alerts from SentinelOne, triggering the workflow upon receiving data.

- **Nodes Involved:**  
  - New SentinelOne Threat (Webhook)

- **Node Details:**  
  - **New SentinelOne Threat**  
    - Type: Webhook  
    - Role: Entry point to receive threat alerts from SentinelOne in real-time.  
    - Configuration: Listens for incoming HTTP POST requests; no specific parameters set, implying default webhook settings.  
    - Inputs: External HTTP calls from SentinelOne.  
    - Outputs: Passes raw threat payload downstream.  
    - Edge Cases: Missing or malformed webhook payloads, security of webhook (e.g., no explicit authentication shown).  
    - Version: Uses typeVersion 2 (n8n webhook node latest stable version).  

#### 1.2 Threat Intelligence Extraction

- **Overview:**  
  Parses the raw SentinelOne threat payload to extract relevant threat intelligence data for further processing.

- **Nodes Involved:**  
  - Extract Threat Intelligence (Code)

- **Node Details:**  
  - **Extract Threat Intelligence**  
    - Type: Code (JavaScript)  
    - Role: Extracts and structures threat-related data fields such as threat name, severity, MITRE tactics, affected endpoints, etc.  
    - Configuration: Custom JavaScript code processes webhook JSON payload.  
    - Inputs: Data from the "New SentinelOne Threat" webhook node.  
    - Outputs: Structured threat intelligence data for subsequent API requests.  
    - Edge Cases: Unexpected or incomplete payload structure, potential runtime errors in code.  
    - Version: typeVersion 2 (code node).  

#### 1.3 Autotask Data Retrieval & Rate Limiting

- **Overview:**  
  Sequentially fetches necessary Autotask system data (users, client companies, ticket fields) with intentional delays to comply with API rate limits.

- **Nodes Involved:**  
  - Fetch Autotask Users (HTTP Request)  
  - Rate Limit Delay 1 (Wait)  
  - Load Client Companies (HTTP Request)  
  - Wait (Wait)  
  - Process Company Data (SplitOut)  
  - Retrieve Ticket Fields (HTTP Request)  
  - Parse Field Options (Code)  
  - Rate Limit Delay 2 (Wait)

- **Node Details:**  
  - **Fetch Autotask Users**  
    - Type: HTTP Request  
    - Role: Retrieves users from Autotask for ticket assignment or referencing.  
    - Configuration: Configured with Autotask API endpoint and credentials (not detailed in JSON).  
    - Inputs: Output from Extract Threat Intelligence.  
    - Outputs: User list passed downstream.  
    - Edge Cases: API authentication failure, timeout, empty user list.  
    - Version: 4.2 (latest HTTP Request node version).  

  - **Rate Limit Delay 1**  
    - Type: Wait  
    - Role: Introduces delay post user fetch to avoid API throttling.  
    - Configuration: Default wait time (parameters empty).  
    - Inputs: After Fetch Autotask Users.  
    - Outputs: Triggers Load Client Companies.  
    - Edge Cases: Excessive delay may slow workflow; no delay risks rate limiting.  

  - **Load Client Companies**  
    - Type: HTTP Request  
    - Role: Retrieves list of client companies from Autotask for ticket association.  
    - Configuration: Autotask API call with credentials.  
    - Inputs: After Rate Limit Delay 1.  
    - Outputs: Client companies data.  
    - Edge Cases: API errors, empty results.  

  - **Wait**  
    - Type: Wait  
    - Role: Additional delay prior to processing client companies, likely for pacing.  
    - Configuration: Default wait time.  
    - Inputs: After Load Client Companies.  
    - Outputs: Triggers Process Company Data.  

  - **Process Company Data**  
    - Type: SplitOut  
    - Role: Splits client companies array into individual records for separate processing.  
    - Configuration: Default split.  
    - Inputs: After Wait node.  
    - Outputs: Individual company data items.  

  - **Retrieve Ticket Fields**  
    - Type: HTTP Request  
    - Role: Fetches metadata about ticket fields from Autotask, necessary for accurate ticket creation.  
    - Configuration: Autotask API call.  
    - Inputs: After Process Company Data.  
    - Outputs: Ticket field definitions.  

  - **Parse Field Options**  
    - Type: Code (JavaScript)  
    - Role: Parses and structures options for ticket fields (e.g., dropdown choices).  
    - Inputs: Output from Retrieve Ticket Fields.  
    - Outputs: Parsed field options for mapping.  

  - **Rate Limit Delay 2**  
    - Type: Wait  
    - Role: Final delay ensuring API calls to Autotask are paced before ticket creation.  

#### 1.4 Data Processing & Mapping

- **Overview:**  
  Maps client company data with ticket field options and prepares the payload for ticket creation.

- **Nodes Involved:**  
  - Map Client Company (Code)

- **Node Details:**  
  - **Map Client Company**  
    - Type: Code (JavaScript)  
    - Role: Combines client company data with parsed ticket field options and threat intelligence to generate a valid ticket payload.  
    - Inputs: After Rate Limit Delay 2.  
    - Outputs: Structured ticket creation data.  
    - Edge Cases: Data mismatches, missing fields, mapping errors.  

#### 1.5 Ticket Creation

- **Overview:**  
  Sends the prepared ticket data to Autotask to create a new security ticket.

- **Nodes Involved:**  
  - Create Security Ticket (HTTP Request)

- **Node Details:**  
  - **Create Security Ticket**  
    - Type: HTTP Request  
    - Role: Posts ticket data to Autotask API to instantiate a new security incident ticket.  
    - Configuration: Autotask API endpoint, authentication credentials, payload built from prior code node.  
    - Inputs: From Map Client Company node.  
    - Outputs: Autotask API response confirming ticket creation.  
    - Edge Cases: API errors, validation failures, connectivity issues.  

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                         | Input Node(s)             | Output Node(s)               | Sticky Note |
|-------------------------|------------------|---------------------------------------|---------------------------|-----------------------------|-------------|
| New SentinelOne Threat  | Webhook          | Entry point, receives SentinelOne alerts | (external webhook)        | Extract Threat Intelligence  |             |
| Extract Threat Intelligence | Code          | Extracts threat data from webhook payload | New SentinelOne Threat   | Fetch Autotask Users         |             |
| Fetch Autotask Users    | HTTP Request     | Retrieves Autotask users               | Extract Threat Intelligence | Rate Limit Delay 1          |             |
| Rate Limit Delay 1      | Wait             | Delays to prevent API rate limits     | Fetch Autotask Users      | Load Client Companies        |             |
| Load Client Companies   | HTTP Request     | Retrieves client companies from Autotask | Rate Limit Delay 1      | Wait                        |             |
| Wait                    | Wait             | Additional pacing delay                | Load Client Companies     | Process Company Data         |             |
| Process Company Data    | SplitOut         | Splits company data array into items  | Wait                      | Retrieve Ticket Fields       |             |
| Retrieve Ticket Fields  | HTTP Request     | Fetches ticket field metadata          | Process Company Data      | Parse Field Options          |             |
| Parse Field Options     | Code             | Parses ticket field options             | Retrieve Ticket Fields    | Rate Limit Delay 2           |             |
| Rate Limit Delay 2      | Wait             | Final delay before ticket creation     | Parse Field Options       | Map Client Company           |             |
| Map Client Company      | Code             | Maps company and threat data to ticket fields | Rate Limit Delay 2   | Create Security Ticket       |             |
| Create Security Ticket  | HTTP Request     | Creates security ticket in Autotask    | Map Client Company        |                             |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: "New SentinelOne Threat"  
   - Type: Webhook  
   - Configure to receive POST requests from SentinelOne alerts. No authentication set by default; configure security as required.

2. **Add Code Node:**  
   - Name: "Extract Threat Intelligence"  
   - Type: Code (JavaScript)  
   - Connect input from "New SentinelOne Threat"  
   - Implement code to parse webhook JSON, extracting fields like threat name, severity, MITRE ATT&CK tactics, affected assets, timestamp, etc.

3. **Add HTTP Request Node:**  
   - Name: "Fetch Autotask Users"  
   - Connect input from "Extract Threat Intelligence"  
   - Configure with Autotask API URL to fetch users (e.g., GET /users)  
   - Set OAuth2 or API key credentials for Autotask.

4. **Add Wait Node:**  
   - Name: "Rate Limit Delay 1"  
   - Connect input from "Fetch Autotask Users"  
   - Configure delay duration (e.g., 1-2 seconds) to avoid API throttling.

5. **Add HTTP Request Node:**  
   - Name: "Load Client Companies"  
   - Connect input from "Rate Limit Delay 1"  
   - Configure with Autotask API endpoint to retrieve client companies (e.g., GET /companies)  
   - Use same credentials as above.

6. **Add Wait Node:**  
   - Name: "Wait"  
   - Connect input from "Load Client Companies"  
   - Configure delay to pace data processing.

7. **Add SplitOut Node:**  
   - Name: "Process Company Data"  
   - Connect input from "Wait"  
   - Configure to split array of companies into individual items.

8. **Add HTTP Request Node:**  
   - Name: "Retrieve Ticket Fields"  
   - Connect input from "Process Company Data"  
   - Configure to get ticket field metadata from Autotask (e.g., GET /ticketFields)  
   - Use credentials.

9. **Add Code Node:**  
   - Name: "Parse Field Options"  
   - Connect input from "Retrieve Ticket Fields"  
   - Implement code to parse options for fields (dropdowns, enums) for proper ticket field mapping.

10. **Add Wait Node:**  
    - Name: "Rate Limit Delay 2"  
    - Connect input from "Parse Field Options"  
    - Configure delay similar to prior wait nodes.

11. **Add Code Node:**  
    - Name: "Map Client Company"  
    - Connect input from "Rate Limit Delay 2"  
    - Implement code to map client company data combined with parsed ticket fields and threat intelligence into a ticket payload.

12. **Add HTTP Request Node:**  
    - Name: "Create Security Ticket"  
    - Connect input from "Map Client Company"  
    - Configure to POST ticket data to Autotask (e.g., POST /tickets)  
    - Use proper credentials and ensure payload matches Autotask API schema.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                          |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Workflow integrates SentinelOne threat alerts with Autotask ticketing to automate security incident response.  | Project purpose                         |
| Careful API rate limit handling is implemented via multiple Wait nodes to prevent request throttling.          | Operational stability                   |
| MITRE ATT&CK analysis is embedded in threat extraction to enhance ticket context and prioritization.           | Security enrichment                     |
| Ensure Autotask API credentials are configured securely (OAuth2 recommended).                                  | Credential management best practice     |
| For webhook security, consider adding authentication or secret verification to the SentinelOne webhook node.    | Security best practice                   |

---

**Disclaimer:**  
The provided text is derived exclusively from an n8n automated workflow. All data handled is legal and public, strictly respecting content policies. No illegal, offensive, or protected content is included.