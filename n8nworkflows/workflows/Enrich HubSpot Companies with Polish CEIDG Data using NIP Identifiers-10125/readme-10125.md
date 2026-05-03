Enrich HubSpot Companies with Polish CEIDG Data using NIP Identifiers

https://n8nworkflows.xyz/workflows/enrich-hubspot-companies-with-polish-ceidg-data-using-nip-identifiers-10125


# Enrich HubSpot Companies with Polish CEIDG Data using NIP Identifiers

### 1. Workflow Overview

This workflow is designed to automatically enrich HubSpot company records with official Polish company data fetched from the CEIDG (Central Register and Information on Economic Activity) database using a company’s NIP (Polish Tax Identification Number). It targets Polish sales teams, marketing agencies, and CRM administrators who want to ensure data accuracy and reduce manual entry in HubSpot.

The workflow is logically divided into the following functional blocks:

- **1.1 Trigger Input Reception:** Watches for changes to the NIP property on HubSpot company records.
- **1.2 Input Validation:** Checks if a valid NIP value is provided before proceeding.
- **1.3 External Data Retrieval:** Queries the CEIDG API using the NIP to fetch official company data.
- **1.4 Response Verification:** Validates that the CEIDG API returned company records.
- **1.5 Data Transformation:** Maps and restructures the CEIDG API response into a format suitable for HubSpot company updates.
- **1.6 HubSpot Company Update:** Applies the enriched data to the corresponding HubSpot company record.
- **1.7 Error Handling:** Marks HubSpot records with an error note if the NIP is invalid or no data is found.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Input Reception

- **Overview:**  
  Listens for any changes to the "nip" property on HubSpot company records to start the workflow.

- **Nodes Involved:**  
  - When NIP property changes

- **Node Details:**

  - **When NIP property changes**  
    - Type: HubSpot Trigger  
    - Role: Initiates workflow upon company property change event.  
    - Configuration: Triggered specifically on the "nip" property change event for company objects in HubSpot. Uses a webhook with ID `7a4af938-f0f4-49cb-84f1-de35a042d791`.  
    - Inputs: None (event-driven)  
    - Outputs: Triggers "Check if NIP exists" node  
    - Edge cases: Missed webhook events if webhook setup is incomplete or network issues occur; requires valid HubSpot app token credentials.  
    - Sub-workflow: None

#### 2.2 Input Validation

- **Overview:**  
  Ensures the NIP property is not empty before proceeding, preventing unnecessary API calls.

- **Nodes Involved:**  
  - Check if NIP exists

- **Node Details:**

  - **Check if NIP exists**  
    - Type: If node  
    - Role: Conditional gate that checks if the NIP string is not empty.  
    - Configuration: Condition checks if `propertyValue` (the new NIP) is non-empty string.  
    - Inputs: Trigger data from the HubSpot Trigger node  
    - Outputs:  
      - If true: proceeds to "Fetch company data from CEIDG"  
      - If false: ends workflow (no next node)  
    - Edge cases: Unexpected data structure could cause expression evaluation failures; empty or whitespace-only NIP values can cause false negatives.

#### 2.3 External Data Retrieval

- **Overview:**  
  Fetches official company data from the CEIDG API using the provided NIP value.

- **Nodes Involved:**  
  - Fetch company data from CEIDG

- **Node Details:**

  - **Fetch company data from CEIDG**  
    - Type: HTTP Request  
    - Role: Queries CEIDG v3 API endpoint to retrieve company details by NIP.  
    - Configuration:  
      - URL: `https://dane.biznes.gov.pl/api/ceidg/v3/firmy`  
      - Query parameter: `nip` set to the NIP from the trigger (`propertyValue`)  
      - Authentication: HTTP Bearer token (predefined credential containing CEIDG API token)  
      - On error: continues workflow without failure (to allow error handling downstream)  
    - Inputs: Output from "Check if NIP exists" node  
    - Outputs: To "Check if data retrieved" node  
    - Edge cases: Network timeouts, invalid tokens, API rate limits, malformed responses, or empty results.  
    - Sub-workflow: None

#### 2.4 Response Verification

- **Overview:**  
  Checks if the CEIDG API response contains any company records before continuing.

- **Nodes Involved:**  
  - Check if data retrieved

- **Node Details:**

  - **Check if data retrieved**  
    - Type: If node  
    - Role: Verifies that the API response includes at least one company (`count > 0`).  
    - Configuration: Checks if `$json.count` > 0  
    - Inputs: JSON output from CEIDG API response  
    - Outputs:  
      - True: proceeds to "Transform data for HubSpot"  
      - False: proceeds to "Mark error in HubSpot"  
    - Edge cases: API response missing `count` field or invalid JSON structure causing condition failure.

#### 2.5 Data Transformation

- **Overview:**  
  Maps and restructures the CEIDG company data to HubSpot-friendly property formats for updating records.

- **Nodes Involved:**  
  - Transform data for HubSpot

- **Node Details:**

  - **Transform data for HubSpot**  
    - Type: Code node (JavaScript)  
    - Role: Extracts relevant fields from the CEIDG API response and prepares the object with properties for HubSpot update.  
    - Configuration:  
      - Validates the presence of company data in `firmy` array.  
      - Extracts first company entry.  
      - Retrieves address components (`ulica`, `budynek`, `lokal`, `miasto`, `kod`, `wojewodztwo`).  
      - Constructs street address string including optional unit number.  
      - Extracts owner NIP and REGON identifiers.  
      - Retrieves other properties: phone, website, company name, business start date.  
      - Obtains `companyId` from the original HubSpot trigger data for update linkage.  
      - Constructs a description field summarizing source and identifiers.  
    - Inputs: CEIDG API response and original HubSpot trigger data (via `$item(0).$node[...]`)  
    - Outputs: JSON with `companyId` and a nested `properties` object for HubSpot update  
    - Edge cases: Missing or malformed input data; multiple companies in response (only first used); undefined nested objects causing runtime errors.

#### 2.6 HubSpot Company Update

- **Overview:**  
  Updates the HubSpot company record with enriched data from CEIDG.

- **Nodes Involved:**  
  - Update company in HubSpot

- **Node Details:**

  - **Update company in HubSpot**  
    - Type: HubSpot node  
    - Role: Updates a company record with mapped properties.  
    - Configuration:  
      - Resource: Company  
      - Operation: Update  
      - Company ID: dynamically set from the previous transform node output (`companyId`)  
      - Properties updated:  
        - name, city, postalCode, websiteUrl, phoneNumber, stateRegion, countryRegion, streetAddress  
        - custom property `ceidg_notes` for description/notes  
      - Authentication: HubSpot App Token credentials  
    - Inputs: Output from "Transform data for HubSpot" node  
    - Outputs: None (end of success path)  
    - Edge cases: Invalid companyId, failed authentication, API limits, invalid property values.

#### 2.7 Error Handling

- **Overview:**  
  Marks the HubSpot company record with an error note if no valid data was retrieved or NIP was invalid.

- **Nodes Involved:**  
  - Mark error in HubSpot

- **Node Details:**

  - **Mark error in HubSpot**  
    - Type: HubSpot node  
    - Role: Updates the company record to add an error note in the `ceidg_notes` property.  
    - Configuration:  
      - Resource: Company  
      - Operation: Update  
      - Company ID: from "Check if NIP exists" node output (`companyId`)  
      - Sets `ceidg_notes` property to `"Invalid NIP / No data found in CEIDG database"`  
      - Authentication: HubSpot App Token credentials  
    - Inputs: From "Check if NIP exists" or "Check if data retrieved" nodes  
    - Outputs: None (end of error path)  
    - Edge cases: Failure to update HubSpot due to invalid ID or auth failure.

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                      | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                          |
|-------------------------------|--------------------|-----------------------------------|------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| When NIP property changes      | HubSpot Trigger    | Trigger workflow on NIP property change | None                         | Check if NIP exists             | ## Trigger Monitors HubSpot for changes to the NIP property on company records.                    |
| Check if NIP exists            | If                 | Validate NIP is present           | When NIP property changes    | Fetch company data from CEIDG / Mark error in HubSpot | ## Validation Checks if the NIP field has a value. If empty, the workflow ends here.               |
| Fetch company data from CEIDG  | HTTP Request       | Retrieve company data from CEIDG API | Check if NIP exists          | Check if data retrieved         | ## CEIDG API Query Fetches official company data from the Polish government database.              |
| Check if data retrieved        | If                 | Verify API returned data          | Fetch company data from CEIDG| Transform data for HubSpot / Mark error in HubSpot | ## Data Verification Checks if the API returned any company records.                               |
| Transform data for HubSpot     | Code (JavaScript)  | Map CEIDG data to HubSpot format | Check if data retrieved      | Update company in HubSpot       | ## Transform Data Maps CEIDG API response to HubSpot company properties.                           |
| Update company in HubSpot      | HubSpot            | Update company with CEIDG data    | Transform data for HubSpot   | None                           | ## Success Path Updates the HubSpot company record with all enriched data from CEIDG.             |
| Mark error in HubSpot          | HubSpot            | Mark error note on company record | Check if NIP exists / Check if data retrieved | None                  | ## Error Path If NIP invalid or no data found, adds a note to the HubSpot company.                 |
| Workflow Description           | Sticky Note        | Documentation                    | None                         | None                           | Full workflow description, usage instructions, setup, and customization notes                      |
| Step 1: HubSpot Trigger        | Sticky Note        | Documentation                    | None                         | None                           | ## Trigger Monitors HubSpot for changes to the NIP property on company records.                    |
| Step 2: Validate Input         | Sticky Note        | Documentation                    | None                         | None                           | ## Validation Checks if the NIP field has a value.                                                 |
| Step 3: Fetch Data             | Sticky Note        | Documentation                    | None                         | None                           | ## CEIDG API Query Fetches official company data from the Polish government database.              |
| Step 4: Verify Response        | Sticky Note        | Documentation                    | None                         | None                           | ## Data Verification Checks if the API returned any company records.                               |
| Step 5: Map Fields             | Sticky Note        | Documentation                    | None                         | None                           | ## Transform Data Maps CEIDG API response to HubSpot company properties.                           |
| Step 6: Update Success         | Sticky Note        | Documentation                    | None                         | None                           | ## Success Path Updates the HubSpot company record with all enriched data from CEIDG.             |
| Error Handling                | Sticky Note        | Documentation                    | None                         | None                           | ## Error Path If NIP is invalid or not found in CEIDG, this node adds a note to the HubSpot company.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create HubSpot Trigger Node**  
   - Type: HubSpot Trigger  
   - Configure event: company.propertyChange  
   - Property to watch: `nip`  
   - Add webhook (copy webhook ID or create new)  
   - Connect no input nodes; output connects to "Check if NIP exists" node  
   - Set HubSpot credentials (App Token or Developer API credentials)

2. **Create "Check if NIP exists" Node**  
   - Type: If node  
   - Condition: String operation `isNotEmpty` on `{{$json.propertyValue}}` (the NIP value from trigger)  
   - Connect input from HubSpot Trigger node  
   - True output connects to "Fetch company data from CEIDG" node  
   - False output ends workflow (no connection)

3. **Create "Fetch company data from CEIDG" Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://dane.biznes.gov.pl/api/ceidg/v3/firmy`  
   - Query Parameter: `nip` = `{{$json.propertyValue}}`  
   - Authentication: HTTP Bearer Token (create credential with your CEIDG API token)  
   - On error: Continue workflow without failure  
   - Connect input from "Check if NIP exists" (true branch)  
   - Connect output to "Check if data retrieved" node

4. **Create "Check if data retrieved" Node**  
   - Type: If node  
   - Condition: Numeric check: `$json.count > 0`  
   - Connect input from "Fetch company data from CEIDG"  
   - True output connects to "Transform data for HubSpot"  
   - False output connects to "Mark error in HubSpot"

5. **Create "Transform data for HubSpot" Node**  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that:  
     - Validates CEIDG response content  
     - Extracts first company data  
     - Builds street address  
     - Extracts owner NIP and REGON  
     - Retrieves companyId from HubSpot trigger data  
     - Returns JSON with `companyId` and mapped `properties`  
   - Connect input from "Check if data retrieved" (true branch)  
   - Connect output to "Update company in HubSpot"

6. **Create "Update company in HubSpot" Node**  
   - Type: HubSpot  
   - Resource: Company  
   - Operation: Update  
   - Company ID: Set expression to `{{$json.companyId}}` from code node output  
   - Set properties fields:  
     - name → `{{$json.properties.name}}`  
     - city → `{{$json.properties.city}}`  
     - postalCode → `{{$json.properties.zip}}`  
     - websiteUrl → `{{$json.properties.website}}`  
     - phoneNumber → `{{$json.properties.phone}}`  
     - stateRegion → `{{$json.properties.state}}`  
     - countryRegion → `{{$json.properties.country}}` (hardcoded "Poland")  
     - streetAddress → `{{$json.properties.address}}`  
     - custom property `ceidg_notes` → `{{$json.properties.description}}`  
   - Connect input from "Transform data for HubSpot"  
   - Set HubSpot credentials

7. **Create "Mark error in HubSpot" Node**  
   - Type: HubSpot  
   - Resource: Company  
   - Operation: Update  
   - Company ID: Expression from `$('Check if NIP exists').item.json.companyId` or from trigger data  
   - Set property `ceidg_notes` to string: `"Invalid NIP / No data found in CEIDG database"`  
   - Connect input from:  
     - "Check if NIP exists" (false branch)  
     - "Check if data retrieved" (false branch)  
   - Set HubSpot credentials

8. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Create sticky notes describing each step/block as per the workflow description to improve maintainability.

9. **Activate the workflow**  
   - Ensure all credentials (HubSpot and CEIDG) are properly set and tested  
   - Activate the workflow in your n8n instance

10. **Test the workflow**  
    - Modify or add a NIP property on any HubSpot company record  
    - Verify that the company record is enriched or marked with an error note accordingly

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                          | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Polish CEIDG API requires a free token, which can be obtained by registering at https://dane.biznes.gov.pl/. This token must be configured as an HTTP Bearer Token credential in n8n for the HTTP Request node.                                                                                        | https://dane.biznes.gov.pl/                                                                                   |
| HubSpot custom properties "nip" (single-line text) and "ceidg_notes" (multi-line text) must be created in your HubSpot account under Company properties before using this workflow.                                                                                                                  | HubSpot Settings > Properties > Company Properties                                                            |
| The workflow currently processes one company record per NIP update event. For batch processing, consider adding a schedule trigger and looping logic.                                                                                                                                                 | n8n workflow customization suggestion                                                                          |
| CEIDG database covers sole proprietorships; for other company types (e.g., limited liability companies), a different API (KRS) is required. This workflow only handles CEIDG data.                                                                                                                     | Polish business registers clarification                                                                        |
| API rate limits and network reliability must be considered when running this workflow in production. Implement appropriate retry or throttling mechanisms if needed.                                                                                                                                 | Best practices for API integrations                                                                            |
| To extend functionality, you can add additional mapping fields or integrate notification nodes (email, Slack) after successful updates.                                                                                                                                                             | Suggested workflow improvements                                                                                 |
| HubSpot API authentication can be done via App Token or OAuth 2.0 Developer credentials; ensure the chosen method has sufficient scopes to update company properties.                                                                                                                                | HubSpot API authentication instructions                                                                        |

---

**Disclaimer:**  
The content provided is strictly derived from an automated workflow created in n8n, a no-code automation tool. It complies fully with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.