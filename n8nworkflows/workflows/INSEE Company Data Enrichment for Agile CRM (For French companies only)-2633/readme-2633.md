INSEE Company Data Enrichment for Agile CRM (For French companies only)

https://n8nworkflows.xyz/workflows/insee-company-data-enrichment-for-agile-crm--for-french-companies-only--2633


# INSEE Company Data Enrichment for Agile CRM (For French companies only)

### 1. Workflow Overview

This workflow is designed to enrich company data stored in Agile CRM for French companies by leveraging the French INSEE OpenData SIREN database. It extracts all companies from Agile CRM, queries the INSEE API to retrieve official company details such as address and SIREN (government ID), and updates the Agile CRM entries with this enriched data. It includes a safeguard feature to avoid overwriting records marked as read-only.

The workflow is logically structured into these blocks:

- **1.1 Trigger & Initialization:** Starts the workflow manually or on schedule, setting the INSEE API key.
- **1.2 Data Extraction from Agile CRM:** Retrieves all company entries from Agile CRM.
- **1.3 ReadOnly Filtering:** Filters out companies flagged as read-only to prevent accidental updates.
- **1.4 INSEE Data Querying:** Queries the INSEE API first to find the company by name, then requests detailed company data.
- **1.5 Data Merging:** Combines Agile CRM data with INSEE results.
- **1.6 CRM Update:** Updates the Agile CRM company entry with enriched data.
- **1.7 Support and Admin Nodes:** Sticky notes for documentation and a no-op node used for routing.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

**Overview:**  
This block starts the workflow either manually via a manual trigger or automatically via a schedule trigger and sets the INSEE API key for subsequent API calls.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger  
- Set Insee API Key

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual testing and execution of the workflow.  
  - Configuration: Default manual trigger settings with no parameters.  
  - Input: None  
  - Output: Triggers the workflow chain by connecting to "Set Insee API Key".  
  - Edge Cases: None specific, but manual trigger requires user action.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically runs the workflow at defined intervals.  
  - Configuration: Runs at default interval (unspecified in JSON, likely default every minute or hourly).  
  - Input: None  
  - Output: Triggers "Set Insee API Key".  
  - Edge Cases: None specific; scheduling conflicts or time zone issues could arise.

- **Set Insee API Key**  
  - Type: Set Node  
  - Role: Stores the INSEE API key as a workflow variable for header injection in API requests.  
  - Configuration: Single string assignment for the header `X-INSEE-Api-Key-Integration`, default placeholder `"put-your-insee-api-key-here"`.  
  - Key expressions: None, static assignment.  
  - Input: Manual or scheduled trigger output.  
  - Output: Passes data to "Get all Compagnies from Agile CRM".  
  - Edge Cases: Failure if API key is missing or invalid; no automatic validation here.

#### 1.2 Data Extraction from Agile CRM

**Overview:**  
Fetches all company records currently stored in Agile CRM to prepare for enrichment.

**Nodes Involved:**  
- Get all Compagnies from Agile CRM

**Node Details:**  

- **Get all Compagnies from Agile CRM**  
  - Type: Agile CRM node  
  - Role: Retrieves all company entities from Agile CRM.  
  - Configuration: Resource = company, Operation = getAll (fetch all companies).  
  - Credentials: Uses Agile CRM OAuth2 credentials configured under "AgileCRM account".  
  - Input: Receives from "Set Insee API Key".  
  - Output: Passes all company data to the filtering node.  
  - Edge Cases: API rate limits, authentication failures, empty result sets.

#### 1.3 ReadOnly Filtering

**Overview:**  
Filters out companies marked as read-only by checking a custom property "RO" with value "1" to avoid overwriting them.

**Nodes Involved:**  
- FilterOut all Company that have the ReadOnly Key set  
- clean_route (NoOp node)

**Node Details:**  

- **FilterOut all Company that have the ReadOnly Key set**  
  - Type: Code node (JavaScript)  
  - Role: Filters out companies flagged with the custom property `"RO" = "1"`.  
  - Configuration: JS code iterates all input items, inspects `properties` array for property with name "RO" and value "1", excludes those items.  
  - Key Expressions: Accesses `$input.all()`, filters on `properties.some(...)`.  
  - Input: From "Get all Compagnies from Agile CRM".  
  - Output: Two outputs:  
    - Main output (filtered companies) connected to "Find Company in SIREN database".  
    - Secondary output connected to "clean_route" (used for routing or future extension).  
  - Edge Cases: Companies missing properties array or without "RO" property will be included; malformed properties may cause exceptions.

- **clean_route**  
  - Type: NoOp  
  - Role: Placeholder node for routing or extension; connected from second output of filter node.  
  - Configuration: None (empty).  
  - Input: From filter node’s second output.  
  - Output: Connected to merge node as a fallback route.  
  - Edge Cases: None.

#### 1.4 INSEE Data Querying

**Overview:**  
Queries the INSEE OpenData API to find the company by name, then requests detailed company data using the SIREN and NIC identifiers.

**Nodes Involved:**  
- Find Company in SIREN database  
- Request all data from SIREN database

**Node Details:**  

- **Find Company in SIREN database**  
  - Type: HTTP Request  
  - Role: Search companies in INSEE API by company name (`denominationUniteLegale`).  
  - Configuration:  
    - Method: GET (default)  
    - URL: `https://api.insee.fr/api-sirene/3.11/siren?q=periode(denominationUniteLegale:"{{ $json.denominationUniteLegale }}")`  
    - Headers: `accept: application/json`, `X-INSEE-Api-Key-Integration` injected from "Set Insee API Key".  
    - On error: continue (do not stop workflow on error).  
  - Key Expressions: URL interpolates `denominationUniteLegale` from input JSON.  
  - Input: From filter node output (filtered companies).  
  - Output: Passes to "Request all data from SIREN database".  
  - Edge Cases: API rate limit, no company found (empty results), invalid API key, malformed company name causing query errors.

- **Request all data from SIREN database**  
  - Type: HTTP Request  
  - Role: Fetch detailed company establishment data from INSEE API using SIREN and NIC.  
  - Configuration:  
    - URL: `https://api.insee.fr/api-sirene/3.11/siret/{{ $json.unitesLegales[0].siren }}{{ $json.unitesLegales[0].periodesUniteLegale[0].nicSiegeUniteLegale }}`  
    - Headers same as above  
    - Method: GET (default)  
  - Key Expressions: Builds URL from JSON path inside INSEE response: `unitesLegales[0].siren` and `periodesUniteLegale[0].nicSiegeUniteLegale`.  
  - Input: From previous INSEE search node.  
  - Output: Passes detailed data to merge node.  
  - Edge Cases: Missing or malformed JSON paths, empty arrays, API errors.

#### 1.5 Data Merging

**Overview:**  
Combines the original Agile CRM company data with the detailed INSEE API data by matching company names.

**Nodes Involved:**  
- Merge data from CRM and SIREN database with enriched for the CRM

**Node Details:**  

- **Merge data from CRM and SIREN database with enriched for the CRM**  
  - Type: Merge  
  - Role: Combines two data streams by matching company names.  
  - Configuration:  
    - Mode: Combine (joins two inputs)  
    - Advanced mode enabled  
    - Merge by fields: `denominationUniteLegale` (left input) with `etablissement.uniteLegale.denominationUniteLegale` (right input)  
  - Input:  
    - First input: From "FilterOut all Company that have the ReadOnly Key set" (original CRM data)  
    - Second input: From "Request all data from SIREN database" (INSEE detailed data)  
  - Output: Passes merged data to "Enrich CRM with INSEE Data".  
  - Edge Cases:  
    - Mismatched company names could cause missing merges  
    - Empty right input causes incomplete data  
    - Case sensitivity or whitespace differences in names may reduce merge accuracy

#### 1.6 CRM Update

**Overview:**  
Updates the Agile CRM company entry with enriched information such as official address and SIREN number.

**Nodes Involved:**  
- Enrich CRM with INSEE Data

**Node Details:**  

- **Enrich CRM with INSEE Data**  
  - Type: Agile CRM node  
  - Role: Updates company in Agile CRM with new address and SIREN data.  
  - Configuration:  
    - Resource: company  
    - Operation: update  
    - Company ID: extracted dynamically from merged data `{{$json.companyId}}`  
    - Additional Fields:  
      - Address: composed from detailed INSEE data fields (`complementAdresseEtablissement`, `typeVoieEtablissement`, `libelleVoieEtablissement`, `codePostalEtablissement`, `libelleCommuneEtablissement`) assigned as office address subtype.  
      - Custom Properties: sets `"SIREN"` custom field with INSEE `siren` value, subtype TEXT.  
    - Credentials: Agile CRM OAuth2 credentials.  
  - Input: From merge node.  
  - Output: None connected (end of chain).  
  - Edge Cases:  
    - Missing or partial INSEE data fields may cause incomplete address  
    - API update failures due to authentication or data validation errors  
    - If company ID is missing or invalid, update will fail

#### 1.7 Support and Admin Nodes

**Overview:**  
Contains sticky notes for user guidance and a no-op node used in routing.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- clean_route (already covered)

**Node Details:**  

- **Sticky Note**  
  - Content: Describes workflow purpose: enrich Agile CRM data with INSEE OpenData API, updating official address and SIREN.  
  - Role: Documentation for users.

- **Sticky Note1**  
  - Content: Setup instructions including credentials setup, custom fields creation ("SIREN" and "RO"), testing, scheduling, and activation.  
  - Role: Setup guidance for administrators.

- **Sticky Note2**  
  - Content: Reminder that workflow can be triggered by either manual or scheduled trigger.  
  - Role: Guidance on triggers.

- **Sticky Note3**  
  - Content: Notes on read-only flag ("RO") usage to prevent overwriting and instructions on how to make the record readonly post-update by adding the "RO" custom property with value "1".  
  - Role: User operational notes to customize workflow behavior.

---

### 3. Summary Table

| Node Name                                      | Node Type          | Functional Role                             | Input Node(s)                           | Output Node(s)                                   | Sticky Note                                                                                                          |
|------------------------------------------------|--------------------|--------------------------------------------|---------------------------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                   | Manual Trigger     | Manual workflow start                       | None                                  | Set Insee API Key                                | 👆 You can use any of those two Trigger to start the process.                                                        |
| Schedule Trigger                                | Schedule Trigger   | Scheduled workflow start                    | None                                  | Set Insee API Key                                | 👆 You can use any of those two Trigger to start the process.                                                        |
| Set Insee API Key                               | Set                | Store INSEE API key for HTTP headers       | When clicking ‘Test workflow’, Schedule Trigger | Get all Compagnies from Agile CRM            | Setup instructions: add Agile CRM & INSEE credentials; create custom fields "SIREN" and "RO".                         |
| Get all Compagnies from Agile CRM               | Agile CRM          | Retrieve all companies                      | Set Insee API Key                     | FilterOut all Company that have the ReadOnly Key set | Setup instructions: add Agile CRM & INSEE credentials; create custom fields "SIREN" and "RO".                         |
| FilterOut all Company that have the ReadOnly Key set | Code               | Filter out read-only companies              | Get all Compagnies from Agile CRM    | Find Company in SIREN database, clean_route      | Notes on read-only flag usage ("RO") to prevent overwriting entries.                                                 |
| clean_route                                     | NoOp               | Routing placeholder for filtered-out data  | FilterOut all Company that have the ReadOnly Key set | Merge data from CRM and SIREN database with enriched for the CRM |                                                                                                                      |
| Find Company in SIREN database                   | HTTP Request       | Search company in INSEE by company name    | FilterOut all Company that have the ReadOnly Key set | Request all data from SIREN database             |                                                                                                                      |
| Request all data from SIREN database             | HTTP Request       | Get detailed company data from INSEE API   | Find Company in SIREN database        | Merge data from CRM and SIREN database with enriched for the CRM |                                                                                                                      |
| Merge data from CRM and SIREN database with enriched for the CRM | Merge              | Combine Agile CRM and INSEE data            | clean_route, Request all data from SIREN database | Enrich CRM with INSEE Data                        |                                                                                                                      |
| Enrich CRM with INSEE Data                       | Agile CRM          | Update company entry with enriched data     | Merge data from CRM and SIREN database with enriched for the CRM | None                                            | Notes: To make record readonly after update, add custom property "RO" = 1 in this node.                              |
| Sticky Note                                     | Sticky Note        | Documentation of workflow purpose           | None                                  | None                                            | Enrich CRM data with French INSEE OpenData API; updates official address and SIREN.                                  |
| Sticky Note1                                    | Sticky Note        | Setup instructions                          | None                                  | None                                            | Setup steps: credentials, custom fields, testing, scheduling, activating workflow.                                   |
| Sticky Note2                                    | Sticky Note        | Trigger usage note                          | None                                  | None                                            | You can use either manual or scheduled trigger to start the workflow.                                               |
| Sticky Note3                                    | Sticky Note        | ReadOnly flag usage notes                   | None                                  | None                                            | Workflow writes over entries unless "RO" = 1; to lock post-update, add "RO" = 1 in last update node.                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. No special configuration needed.  
   - Add a **Schedule Trigger** node named `Schedule Trigger`. Set desired interval (e.g., daily or hourly).

2. **Set INSEE API Key:**  
   - Add a **Set** node named `Set Insee API Key`.  
   - Add one field: `X-INSEE-Api-Key-Integration` (string), default value `"put-your-insee-api-key-here"`.  
   - Connect both trigger nodes to this node.

3. **Retrieve Companies from Agile CRM:**  
   - Add an **Agile CRM** node named `Get all Compagnies from Agile CRM`.  
   - Configure resource as `company`, operation as `getAll`.  
   - Assign Agile CRM OAuth2 credentials.  
   - Connect output of `Set Insee API Key` to this node.

4. **Filter ReadOnly Companies:**  
   - Add a **Code** node named `FilterOut all Company that have the ReadOnly Key set`.  
   - Paste the following JS code:

   ```javascript
   const input = $input.all();
   const output = input.filter(item => {
       const properties = item.json.properties || [];
       return !properties.some(property => property.name === "RO" && property.value === "1");
   }).map(item => {
       const companyId = item.json.id;
       const denominationUniteLegale = item.json.properties[0]?.value || null;
       return {
           json: {
               companyId,
               denominationUniteLegale
           }
       };
   });
   return output;
   ```

   - Connect output of `Get all Compagnies from Agile CRM` to this node.

5. **Add NoOp Node:**  
   - Add a **NoOp** node named `clean_route`.  
   - Connect second output of `FilterOut all Company that have the ReadOnly Key set` to `clean_route`.

6. **Search Company in INSEE SIREN Database:**  
   - Add an **HTTP Request** node named `Find Company in SIREN database`.  
   - Configure:  
     - URL: `https://api.insee.fr/api-sirene/3.11/siren?q=periode(denominationUniteLegale:"{{ $json.denominationUniteLegale }}")`  
     - HTTP Method: GET  
     - Headers:  
       - `accept: application/json`  
       - `X-INSEE-Api-Key-Integration`: `={{ $('Set Insee API Key').all()[0].json['X-INSEE-Api-Key-Integration'] }}`  
     - Error Handling: Continue on error.  
   - Connect first output of `FilterOut all Company that have the ReadOnly Key set` to this node.

7. **Request Detailed Company Data from INSEE:**  
   - Add an **HTTP Request** node named `Request all data from SIREN database`.  
   - Configure:  
     - URL: `https://api.insee.fr/api-sirene/3.11/siret/{{ $json.unitesLegales[0].siren }}{{ $json.unitesLegales[0].periodesUniteLegale[0].nicSiegeUniteLegale }}`  
     - HTTP Method: GET  
     - Headers same as previous node.  
   - Connect output of `Find Company in SIREN database` to this node.

8. **Merge CRM and INSEE Data:**  
   - Add a **Merge** node named `Merge data from CRM and SIREN database with enriched for the CRM`.  
   - Set mode to `Combine`.  
   - Enable `Advanced` options.  
   - Set merge by fields:  
     - Field1: `denominationUniteLegale` (from first input)  
     - Field2: `etablissement.uniteLegale.denominationUniteLegale` (from second input)  
   - Connect `clean_route` node to first input (index 0).  
   - Connect `Request all data from SIREN database` node to second input (index 1).

9. **Update Agile CRM with Enriched Data:**  
   - Add an **Agile CRM** node named `Enrich CRM with INSEE Data`.  
   - Configure:  
     - Resource: `company`  
     - Operation: `update`  
     - Company ID: `={{ $json.companyId }}`  
     - Additional Fields:  
       - Address Options:  
         - Address Properties:  
           - Address:  
             ```
             {{ $json.etablissement.adresseEtablissement.complementAdresseEtablissement }}
             {{ $json.etablissement.adresseEtablissement.typeVoieEtablissement }} {{ $json.etablissement.adresseEtablissement.libelleVoieEtablissement }}
             {{ $json.etablissement.adresseEtablissement.codePostalEtablissement }} {{ $json.etablissement.adresseEtablissement.libelleCommuneEtablissement }}
             ```  
           - Subtype: `office`  
       - Custom Properties:  
         - Name: `SIREN`  
         - Value: `={{ $json.etablissement.siren }}`  
         - Subtype: `TEXT`  
     - Credentials: Agile CRM OAuth2 credentials.  
   - Connect output of `Merge data from CRM and SIREN database with enriched for the CRM` to this node.

10. **Add Sticky Notes for Documentation (Optional but Recommended):**  
    - Add sticky notes to describe workflow purpose, setup instructions, trigger usage, and read-only flag usage as per the original workflow for user guidance.

11. **Validate & Test:**  
    - Fill in your actual API keys and credentials.  
    - Ensure Agile CRM has custom fields named:  
      - "SIREN" (Text Field)  
      - "RO" (Number Field)  
    - Test manually via the manual trigger node.  
    - Schedule automation as required.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow enriches Agile CRM company data using the French INSEE OpenData API, which provides official government company information including addresses and SIREN numbers.                                                                           | https://portail-api.insee.fr/                      |
| To prevent overwriting data for specific companies, use the "RO" custom field set to 1. To lock a record after update, add the "RO" property with value 1 in the last update node.                                                                         | Workflow usage note                                |
| Ensure Agile CRM custom fields: "SIREN" (Text) and "RO" (Number) are created before running the workflow.                                                                                                                                                 | Agile CRM Admin Settings                           |
| This workflow supports both manual and scheduled triggers, allowing flexible execution.                                                                                                                                                                   | Trigger usage note                                 |
| The INSEE API requires an API key; register and obtain this key at the INSEE portal prior to running the workflow.                                                                                                                                          | https://portail-api.insee.fr/                      |

---

This documentation fully describes the "INSEE Company Data Enrichment for Agile CRM" workflow, enabling thorough understanding, error anticipation, and full reproduction in n8n without access to the original JSON.