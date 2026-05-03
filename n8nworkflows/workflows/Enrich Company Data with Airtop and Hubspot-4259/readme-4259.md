Enrich Company Data with Airtop and Hubspot

https://n8nworkflows.xyz/workflows/enrich-company-data-with-airtop-and-hubspot-4259


# Enrich Company Data with Airtop and Hubspot

### 1. Workflow Overview

This workflow, titled **"Enrich Company Data with Airtop and Hubspot"**, automates the process of enriching company information based on user input from a form or another workflow. Its main purpose is to enhance CRM records by retrieving detailed company profiles, calculating an Ideal Customer Profile (ICP) score, and updating or creating company records in HubSpot. It is designed primarily for onboarding, qualification, and CRM enrichment scenarios.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts input parameters either via a web form submission or from another workflow trigger. It unifies the inputs for consistent downstream processing.
- **1.2 Corporate Email Validation:** Filters out non-corporate email domains to avoid processing generic or invalid company data.
- **1.3 Company Information Enrichment:** Calls a sub-workflow that extracts detailed company data from Airtop, leveraging LinkedIn connections and domain information, and calculates the ICP score.
- **1.4 Data Aggregation and Mapping:** Aggregates results from enrichment, maps and restructures data fields to match the HubSpot format.
- **1.5 HubSpot Integration:** Sends the enriched and mapped company data to a dedicated sub-workflow that upserts (inserts or updates) the company record in HubSpot.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures input data either from a web form submission or when triggered by another workflow, then standardizes these inputs into a unified format for consistent processing.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow  
- Unify Params

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Receives data from a web form titled "New contact signup" with required fields: "Airtop Profile (connected to Linkedin)", "Company domain", and optional "Company LinkedIn".  
  - *Configuration:* Webhook-based trigger with defined form fields and description.  
  - *Inputs:* External form submission  
  - *Outputs:* JSON containing submitted fields  
  - *Failure cases:* Webhook unavailability, incomplete form submission, or missing required fields.

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Allows this workflow to be triggered programmatically by another workflow with input parameters.  
  - *Configuration:* Defines expected inputs matching form fields.  
  - *Inputs:* Workflow call  
  - *Outputs:* JSON with passed parameters  
  - *Failure cases:* Missing required inputs, workflow call errors.

- **Unify Params**  
  - *Type:* Set  
  - *Role:* Normalizes inputs from either trigger to a consistent parameter set with keys: "Airtop Profile (connected to Linkedin)", "Company domain", and "Company LinkedIn".  
  - *Configuration:* Assigns values directly from incoming JSON fields to standardized names for downstream use.  
  - *Inputs:* Output from either trigger node  
  - *Outputs:* JSON with unified parameter keys  
  - *Failure cases:* Missing fields in input JSON, expression evaluation errors.

---

#### 2.2 Corporate Email Validation

**Overview:**  
Filters out entries whose company domain corresponds to common personal or educational email providers to ensure only legitimate corporate domains proceed.

**Nodes Involved:**  
- Is corporate email?

**Node Details:**  

- **Is corporate email?**  
  - *Type:* Filter  
  - *Role:* Excludes domains matching generic email providers (e.g., gmail, yahoo, outlook) or ending with ".edu".  
  - *Configuration:* Uses two conditions combined with AND:  
    - Domain does **not** match regex of common providers  
    - Domain does **not** end with ".edu"  
  - *Inputs:* Unified parameters with "Company domain"  
  - *Outputs:* Passes only corporate domains to next node  
  - *Failure cases:* False negatives if domain format varies, regex mismatches, empty or malformed domain fields.

---

#### 2.3 Company Information Enrichment

**Overview:**  
Executes a dedicated sub-workflow to extract detailed company information and calculate an ICP score by leveraging Airtop data connected with LinkedIn profiles.

**Nodes Involved:**  
- Company info  
- Aggregate

**Node Details:**  

- **Company info**  
  - *Type:* Execute Workflow  
  - *Role:* Runs the sub-workflow "AIRTOP — Extract company data and calculate ICP" with inputs: company domain, LinkedIn URL, and Airtop profile.  
  - *Configuration:* Passes current JSON parameters as inputs; conversion to string enabled to avoid type conflicts.  
  - *Inputs:* From "Is corporate email?" node  
  - *Outputs:* Enriched data including company profile, ICP score, and LinkedIn info  
  - *Failure cases:* Sub-workflow errors, API credential issues, missing or invalid input data, timeout.  
  - *Sub-workflow reference:* ID "Ipc5s7U9UzuqexMI"

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Combines all items from the previous node into a single aggregated object, useful if multiple data points come from sub-workflow.  
  - *Configuration:* Uses "aggregateAllItemData" method with default settings.  
  - *Inputs:* From "Company info"  
  - *Outputs:* Single aggregated JSON object  
  - *Failure cases:* Empty input, aggregation errors.

---

#### 2.4 Data Aggregation and Mapping

**Overview:**  
Transforms the aggregated raw data into a structured format aligned with HubSpot’s expected fields for company records.

**Nodes Involved:**  
- Map information

**Node Details:**  

- **Map information**  
  - *Type:* Set  
  - *Role:* Extracts and maps various fields from the aggregated data into a flat structure with keys such as company_profile, city, name, country, tagline, employee_count, icp_score, linkedin, state, website, and company_domain.  
  - *Configuration:* Uses expressions to select nested data from the aggregated array at specific indices, e.g., company overview, location details, ICP score, and LinkedIn URL.  
  - *Inputs:* From "Aggregate"  
  - *Outputs:* JSON with mapped company information ready for HubSpot upsert  
  - *Failure cases:* Index out of range if expected data missing, expression errors, null fields.

---

#### 2.5 HubSpot Integration

**Overview:**  
Sends the mapped company data to another sub-workflow responsible for upserting the company record into HubSpot CRM.

**Nodes Involved:**  
- Save company Hubspot

**Node Details:**  

- **Save company Hubspot**  
  - *Type:* Execute Workflow  
  - *Role:* Invokes the sub-workflow "AIRTOP — Upsert company in Hubspot" with mapped company data fields.  
  - *Configuration:* Passes all relevant mapped fields (city, name, country, tagline, website, LinkedIn, ICP score, employee count, company profile, domain, state) as inputs. String conversion is enabled.  
  - *Inputs:* From "Map information"  
  - *Outputs:* Success or failure confirmation from HubSpot upsert  
  - *Failure cases:* API authentication errors, rate limits, invalid field mappings, sub-workflow errors.  
  - *Sub-workflow reference:* ID "I5o15J4MI1P1GRjh"

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                                  | Input Node(s)              | Output Node(s)         | Sticky Note                                         |
|---------------------------|------------------------|-------------------------------------------------|----------------------------|------------------------|-----------------------------------------------------|
| On form submission        | Form Trigger           | Accepts input from web form                      | (external)                 | Unify Params            | ## Input Parameters<br>Run this workflow using a form or from another workflow |
| When Executed by Another Workflow | Execute Workflow Trigger | Accepts input from another workflow              | (external)                 | Unify Params            | ## Input Parameters<br>Run this workflow using a form or from another workflow |
| Unify Params             | Set                    | Normalizes input parameters                       | On form submission, When Executed by Another Workflow | Is corporate email?     |                                                     |
| Is corporate email?      | Filter                 | Filters out non-corporate email domains          | Unify Params               | Company info            | ## Keep only corporate emails                        |
| Company info             | Execute Workflow       | Runs sub-workflow to enrich company info and calculate ICP | Is corporate email?        | Aggregate               | ## Retrive company info                              |
| Aggregate                | Aggregate              | Aggregates multiple items into one                | Company info               | Map information         | ## Retrive company info                              |
| Map information          | Set                    | Maps aggregated data to HubSpot field structure  | Aggregate                  | Save company Hubspot    | ## Save info in Hubspot                              |
| Save company Hubspot     | Execute Workflow       | Upserts company record in HubSpot CRM             | Map information            | (end)                   | ## Save info in Hubspot                              |
| Sticky Note1             | Sticky Note            | Comment: Keep only corporate emails               |                            |                        | ## Keep only corporate emails                        |
| Sticky Note2             | Sticky Note            | Comment: Retrieve company info                     |                            |                        | ## Retrive company info                              |
| Sticky Note3             | Sticky Note            | Comment: Save info in Hubspot                       |                            |                        | ## Save info in Hubspot                              |
| Sticky Note4             | Sticky Note            | Comment: Input Parameters description              |                            |                        | ## Input Parameters<br>Run this workflow using a form or from another workflow |
| Sticky Note              | Sticky Note            | README and detailed project description            |                            |                        | README with use case, setup requirements, and next steps |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Form Trigger** node named "On form submission". Configure the form with title "New contact signup" and fields:  
     - "Airtop Profile (connected to Linkedin)" (required)  
     - "Company domain" (required)  
     - "Company LinkedIn" (optional)  
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow". Configure it to accept inputs:  
     - "Airtop Profile (connected to Linkedin)"  
     - "Company domain"  
     - "Company LinkedIn"

2. **Normalize Inputs:**  
   - Add a **Set** node named "Unify Params". Configure it to assign:  
     - "Airtop Profile (connected to Linkedin)" = `{{$json["Airtop Profile (connected to Linkedin)"]}}`  
     - "Company domain" = `{{$json["Company domain"]}}`  
     - "Company LinkedIn" = `{{$json["Company LinkedIn"]}}`  
   - Connect outputs of both trigger nodes to "Unify Params" node.

3. **Filter Corporate Emails:**  
   - Add a **Filter** node named "Is corporate email?".  
   - Configure conditions:  
     - "Company domain" does **not** match regex `\b(gmail|hotmail|yahoo|outlook|icloud|aol|protonmail|live|msn|ya)\b` (case sensitive)  
     - AND "Company domain" does **not** end with `.edu`  
   - Connect "Unify Params" output to this filter node.

4. **Company Data Enrichment:**  
   - Add an **Execute Workflow** node named "Company info".  
   - Set it to run the sub-workflow with ID `"Ipc5s7U9UzuqexMI"` (the Airtop extract and ICP calculator).  
   - Map inputs: pass "Company domain", "Company LinkedIn", and "Airtop Profile (connected to Linkedin)" from the filter output. Enable string conversion as needed.  
   - Connect the "Is corporate email?" node's true output to this node.

5. **Aggregate Data:**  
   - Add an **Aggregate** node named "Aggregate".  
   - Configure it to aggregate all item data into a single object (default "aggregateAllItemData").  
   - Connect output of "Company info" to this node.

6. **Map Data for HubSpot:**  
   - Add a **Set** node named "Map information".  
   - Map the following fields using expressions on the aggregated data:  
     - `company_profile` = `{{$json[""][0].company_profile.overview}}`  
     - `city` = `{{$json[""][0].company_profile.location.city}}`  
     - `company_domain` = `{{$node["Is corporate email?"].item.json["Company domain"]}}`  
     - `name` = `{{$json[""][0].company_profile.name}}`  
     - `country` = `{{$json[""][0].company_profile.location.country}}`  
     - `tagline` = `{{$json[""][0].company_profile.tagline}}`  
     - `employee_count` = `{{$json[""][0].scale.employee_count}}`  
     - `icp_score` = `{{$json[""][2].icp_company_score}}`  
     - `linkedin` = `{{$json[""][1].company_linkedin}}`  
     - `state` = `{{$json[""][0].company_profile.location.state}}`  
     - `website` = `{{$json[""][0].company_profile.website}}`  
   - Connect "Aggregate" output to this node.

7. **HubSpot Upsert:**  
   - Add an **Execute Workflow** node named "Save company Hubspot".  
   - Configure it to run the sub-workflow with ID `"I5o15J4MI1P1GRjh"` (handles upserting company data to HubSpot).  
   - Map all fields from "Map information" node as inputs. Enable string conversion as needed.  
   - Connect "Map information" output to this node.

8. **Additional Setup:**  
   - Ensure credentials for Airtop API and HubSpot OAuth2 are configured and linked in the respective sub-workflows.  
   - Verify the sub-workflows "AIRTOP — Extract company data and calculate ICP" and "AIRTOP — Upsert company in Hubspot" are properly set up with expected input/output schemas.  
   - Test the workflow using the form or via manual trigger from another workflow with test data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| README detailing use case: Automates company data enrichment and HubSpot integration for onboarding and CRM enrichment. Inputs include email-derived domain, LinkedIn, and Airtop Profile. Outputs enriched company profile, ICP score, and updates HubSpot. Setup requires Airtop API key, Airtop profile with LinkedIn auth, and HubSpot integration. Suggested use includes triggering on form submission or batch enrichment. Further flexibility to adapt for other CRMs like Salesforce or Pipedrive.                                                                                                                                                                      | Embedded sticky note in the workflow; see the large README node for complete project description.                                                                   |
| Airtop Profile must be connected and authenticated with LinkedIn to ensure accurate enrichment. Ensure API keys and credentials remain valid to avoid authentication errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Related to "Company info" node and sub-workflow inputs.                                                                                                             |
| Regex for corporate email filtering excludes common personal domains such as gmail, yahoo, outlook, icloud, etc., and educational domains ending with ".edu". Adjust these filters if business requirements change.                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Related to "Is corporate email?" filter node.                                                                                                                       |
| HubSpot integration requires OAuth2 credentials with correct scopes to allow company object upsertion. Check HubSpot API limits and error logs in case of failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Related to "Save company Hubspot" sub-workflow node.                                                                                                                |
| Links for setup: Airtop API keys and profiles - https://portal.airtop.ai/api-keys and https://portal.airtop.ai/browser-profiles                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Mentioned in README sticky note.                                                                                                                                     |

---

This completes the structured reference documentation for the "Enrich Company Data with Airtop and Hubspot" n8n workflow. It enables expert users and automation agents to fully understand, reproduce, and troubleshoot the workflow.