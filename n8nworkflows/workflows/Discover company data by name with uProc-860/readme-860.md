Discover company data by name with uProc

https://n8nworkflows.xyz/workflows/discover-company-data-by-name-with-uproc-860


# Discover company data by name with uProc

### 1. Workflow Overview

This n8n workflow automates the enrichment of company data by retrieving detailed company information using the uProc **Get Company by Name** tool. It is designed to support business processes such as signup enrichment, CRM data augmentation, or marketing campaigns by providing verified company details based on a company name and country.

**Target Use Cases:**  
- Enriching company profiles during signup or lead capture  
- Preparing company contact data for outreach campaigns  
- Integrating company location and contact info into CRMs or databases  

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger and creation of the company query item (name and country)  
- **1.2 Company Data Lookup:** Querying uProc’s API to retrieve company data by name and country  
- **1.3 Result Validation:** Checking if the uProc response contains valid company data  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and sets up the input data — the company name and country — which will be submitted to the uProc API.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Create Company Item (Function Item)

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow when the user manually triggers execution  
  - *Configuration:* No parameters; simple manual start  
  - *Inputs:* None  
  - *Outputs:* One output connection to "Create Company Item"  
  - *Edge Cases:* None typical; failsafe for manual use only  
  - *Version:* Compatible with all n8n versions supporting Manual Trigger  

- **Create Company Item**  
  - *Type:* Function Item  
  - *Role:* Constructs the input data object with company name and country  
  - *Configuration:* Hardcoded values for `company` = "Killia technologies" and `country` = "Spain"  
  - *Key Expressions:*  
    - `item.company = "Killia technologies";`  
    - `item.country = "Spain";`  
  - *Inputs:* From Manual Trigger  
  - *Outputs:* To "Get Company by Name" node  
  - *Edge Cases:* If company or country fields are empty or invalid, subsequent API calls may fail or return no data  
  - *Version:* No special requirements  

#### 1.2 Company Data Lookup

**Overview:**  
This block calls the uProc API to search for companies matching the provided name and country, returning detailed location and contact data.

**Nodes Involved:**  
- Get Company by Name (uProc node)

**Node Details:**

- **Get Company by Name**  
  - *Type:* uProc node (custom integration)  
  - *Role:* Queries uProc’s "getCompanyByName" tool with company name and country parameters  
  - *Configuration:*  
    - `name` set dynamically from `Create Company Item` node’s `company` field  
    - `country` set dynamically from `Create Company Item` node’s `country` field  
    - Tool selected: "getCompanyByName" under "company" group  
    - Credentials: uses stored uProc API credentials named "miquel-uproc"  
  - *Key Expressions:*  
    - `={{$node["Create Company Item"].json["company"]}}`  
    - `={{$node["Create Company Item"].json["country"]}}`  
  - *Inputs:* From "Create Company Item"  
  - *Outputs:* To "Company Found?" node  
  - *Version:* Requires n8n version supporting uProc node and credential management  
  - *Edge Cases:*  
    - API authentication errors if credentials are missing or invalid  
    - No results returned if the company is not present on Google Maps or internet sources  
    - Timeout or connectivity issues with uProc API  
    - Unexpected response formats may cause expression errors  

#### 1.3 Result Validation

**Overview:**  
This block checks whether the uProc API returned a valid company result by verifying the presence of a company name in the response.

**Nodes Involved:**  
- Company Found? (If)

**Node Details:**

- **Company Found?**  
  - *Type:* If (Conditional)  
  - *Role:* Verifies if the `name` field inside the `message` object of the uProc response contains any non-empty string  
  - *Configuration:* String condition using regex `.+` on `{{$node["Get Company by Name"].json["message"]["name"]}}`  
  - *Inputs:* From "Get Company by Name"  
  - *Outputs:* Not connected further in this workflow (end of flow)  
  - *Version:* No special requirements  
  - *Edge Cases:*  
    - If `message` or `name` is missing or empty, condition evaluates to false  
    - Expression errors if response JSON structure changes or is malformed  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                              |
|---------------------|---------------------|----------------------------------------|-----------------------|------------------------|---------------------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger      | Starts workflow manually                | -                     | Create Company Item     |                                                                                                         |
| Create Company Item  | Function Item       | Provides company name and country input| On clicking 'execute'  | Get Company by Name     | You can replace this node with other services returning company names and countries (Hubspot, Sheets...) |
| Get Company by Name  | uProc API node      | Calls uProc API to retrieve company data| Create Company Item    | Company Found?          | Requires uProc credentials (Email and API Key) configured in n8n. Uses uProc Get Company by Name tool.    |
| Company Found?       | If (Conditional)    | Checks if company data was found       | Get Company by Name    | -                      | Condition verifies presence of company name in response message.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it** "Get Company by Name".

2. **Add a Manual Trigger node:**  
   - Name: `On clicking 'execute'`  
   - No parameters needed.

3. **Add a Function Item node:**  
   - Name: `Create Company Item`  
   - Function code:  
     ```javascript
     item.company = "Killia technologies";
     item.country = "Spain";
     return item;
     ```
   - Connect output of `On clicking 'execute'` to input of `Create Company Item`.

4. **Add a uProc node:**  
   - Name: `Get Company by Name`  
   - Select tool: `getCompanyByName` (under "company" group)  
   - Set parameters using expressions:  
     - `name`: `{{$node["Create Company Item"].json["company"]}}`  
     - `country`: `{{$node["Create Company Item"].json["country"]}}`  
   - No additional options required by default.  
   - Under Credentials, select or create new uProc API credentials (Email and API Key) from your uProc account ([Integration section](https://app.uproc.io/#/settings/integration)).  
   - Connect output of `Create Company Item` to input of `Get Company by Name`.

5. **Add an If node for validation:**  
   - Name: `Company Found?`  
   - Condition type: String  
   - Condition: Check if `{{$node["Get Company by Name"].json["message"]["name"]}}` matches regex `.+` (meaning any non-empty string).  
   - Connect output of `Get Company by Name` to input of `Company Found?`.

6. **Workflow end:**  
   - The workflow can be extended by connecting outputs of `Company Found?` to further processing nodes such as CRM integration, Google Sheets, or marketing campaign triggers.

7. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires valid uProc API credentials (Email and API Key) configured in n8n.                                       | uProc Integration section: https://app.uproc.io/#/settings/integration                         |
| The "Create Company Item" node is designed for easy replacement to use other data sources like Hubspot, Google Sheets, or Typeform.| You can adapt this workflow to any source that outputs company names and countries.            |
| The uProc tool combines Google Maps and internet email research; no results are returned if the company is not present on Google Maps.| uProc Get Company by Name tool documentation: https://app.uproc.io/#/tools/processor/get/company/by-name |
| Returned company data fields include name, email, cif, address, city, state, county, country, zipcode, phone, website, latitude, longitude.| Useful for CRM enrichment and campaign targeting.                                              |
| Workflow screenshot included in original documentation for visual reference.                                                    | Image file reference: workflow-screenshot (fileId:359)                                        |

---

This document fully describes the workflow structure, logic, and reproduction steps to enable both advanced users and automation agents to understand, reproduce, and adapt the workflow confidently.