Enrich Person Data and Update CRM with Airtop and Hubspot

https://n8nworkflows.xyz/workflows/enrich-person-data-and-update-crm-with-airtop-and-hubspot-4255


# Enrich Person Data and Update CRM with Airtop and Hubspot

### 1. Workflow Overview

This workflow automates the enrichment of a person’s professional data using key identifiers and updates their record in HubSpot CRM. It targets sales, marketing, and recruitment teams who require enhanced contact insights to improve engagement and qualification. The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Accepts person data via a form submission or an external workflow trigger.
- **1.2 Parameter Unification:** Normalizes and consolidates input parameters for downstream processing.
- **1.3 AI-Powered Data Enrichment:** Invokes a sub-workflow to extract detailed person information and calculate an Ideal Customer Profile (ICP) score.
- **1.4 Data Aggregation and Formatting:** Aggregates enriched data, formats fields, and maps values to align with HubSpot schema.
- **1.5 CRM Update:** Calls another sub-workflow to update the contact record in HubSpot with enriched data.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block captures the initial input, either through a form submission or via an external workflow trigger, collecting essential person identifiers to begin the enrichment process.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow  

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing person data submitted via web form.  
  - *Configuration:*  
    - Form titled "Enrich person data"  
    - Four required fields: Person name, Work email, Airtop Profile (linked to LinkedIn), Hubspot object id  
    - Description explains purpose to enrich contact info  
  - *Input/Output:* No input; outputs form data to "Unify Params" node  
  - *Edge Cases:* Missing required fields, webhook connectivity issues, invalid form inputs  
  - *Version:* 2.2  

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Allows this workflow to be triggered programmatically by another workflow, passing in the same four parameters as above  
  - *Configuration:* Defines expected inputs matching form fields  
  - *Input/Output:* Receives input from caller workflow; outputs unified parameters downstream  
  - *Edge Cases:* Missing or malformed input parameters, execution permission errors  
  - *Version:* 1.1  

---

#### 2.2 Parameter Unification

**Overview:**  
This block consolidates input data fields into normalized variables for consistent referencing in downstream nodes.

**Nodes Involved:**  
- Unify Params  

**Node Details:**

- **Unify Params**  
  - *Type:* Set  
  - *Role:* Maps incoming JSON fields to standardized variable names to unify input data structure  
  - *Configuration:*  
    - Assigns `object_id` ← Hubspot object id  
    - Assigns `name` ← Person name  
    - Assigns `email` ← Work email  
    - Assigns `airtop_profile` ← Airtop Profile (connected to LinkedIn)  
  - *Input/Output:* Receives raw input from form or trigger; outputs normalized JSON for enrichment  
  - *Expressions:* Uses expressions like `={{ $json["Person name"] }}` for field assignment  
  - *Edge Cases:* Missing or null input fields, unexpected input JSON structure  
  - *Version:* 3.4  

---

#### 2.3 AI-Powered Data Enrichment

**Overview:**  
Invokes an external sub-workflow that leverages Airtop and AI capabilities to extract detailed person insights and compute an ICP score.

**Nodes Involved:**  
- Extract person info and calculate ICP  
- Aggregate  

**Node Details:**

- **Extract person info and calculate ICP**  
  - *Type:* Execute Workflow  
  - *Role:* Calls sub-workflow “AIRTOP — Extract person data and calculate ICP” to enrich person details  
  - *Configuration:*  
    - Passes parameters: `work_email`, `person_name`, `Airtop_profile` mapped from unified params  
    - Mapping mode set to define parameters explicitly  
  - *Input/Output:* Receives unified inputs; outputs enriched data array  
  - *Edge Cases:* Sub-workflow failures, API key or authentication errors inside sub-workflow, timeout if enrichment takes too long  
  - *Version:* 1.2  
  - *Sub-workflow:* Yes — external enrichment and scoring logic  

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Combines all items returned by enrichment workflow into a single data object for easier field mapping  
  - *Configuration:* Aggregate all item data  
  - *Input/Output:* Receives multiple enriched data items; outputs aggregated single JSON  
  - *Edge Cases:* Empty input arrays, aggregation errors due to malformed data  
  - *Version:* 1  

---

#### 2.4 Data Aggregation and Formatting

**Overview:**  
This block extracts relevant fields from aggregated data and formats them into the final structure expected by HubSpot, including calculated scores and profile URLs.

**Nodes Involved:**  
- Edit Fields  

**Node Details:**

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Assigns and formats enriched fields for HubSpot update, mapping each specific data point from aggregated enrichment results and unified params  
  - *Configuration:*  
    - Maps fields like:  
      - `email` ← unified param email  
      - `current_company`, `linkedin_company_url`, `linkedin_connections`, `linkedin_followers`, `linkedin_about`, `icp_person_score`, `seniority_level`, `technical_depth`, `ai_interest_level` ← from aggregated sub-workflow output  
      - `linkedin_url` ← from a different enrichment array element  
      - `object_id` ← unified param object ID  
      - `location`, `full_name` ← from aggregated data  
    - Uses complex expressions referencing multiple indexes of the aggregated JSON array  
  - *Input/Output:* Receives aggregated data; outputs final enriched and formatted JSON for CRM update  
  - *Edge Cases:* Index out of range if enrichment sub-workflow returns unexpected array size, missing fields causing expression errors  
  - *Version:* 3.4  

---

#### 2.5 CRM Update

**Overview:**  
Executes a sub-workflow to update the corresponding contact record in HubSpot with all enriched and formatted data.

**Nodes Involved:**  
- Save data in Hubspot  

**Node Details:**

- **Save data in Hubspot**  
  - *Type:* Execute Workflow  
  - *Role:* Calls external sub-workflow “AIRTOP — Update Hubspot contact” to push enriched data to HubSpot CRM  
  - *Configuration:*  
    - No direct parameters mapped here; expects input via JSON forwarded from “Edit Fields” node  
  - *Input/Output:* Receives enriched contact data; performs update in HubSpot; no further output  
  - *Edge Cases:* HubSpot API authentication errors, object ID not found errors, data validation errors on HubSpot side  
  - *Version:* 1.2  
  - *Sub-workflow:* Yes — external HubSpot update logic  

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                         | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                      |
|------------------------------|----------------------------|---------------------------------------|--------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| On form submission            | Form Trigger               | Entry point via form submission        | None                           | Unify Params                 | ## Input Parameters<br>** Run this workflow using a form or from another workflow              |
| When Executed by Another Workflow | Execute Workflow Trigger   | Entry point via external workflow call | None                           | Unify Params                 | ## Input Parameters<br>** Run this workflow using a form or from another workflow              |
| Unify Params                 | Set                        | Normalize input parameters             | On form submission, When Executed by Another Workflow | Extract person info and calculate ICP |                                                                                                |
| Extract person info and calculate ICP | Execute Workflow          | Enrich person info and calculate ICP  | Unify Params                   | Aggregate                   | ## Retrieve person info                                                                        |
| Aggregate                   | Aggregate                  | Combine enriched data items            | Extract person info and calculate ICP | Edit Fields                 | ## Save info in Hubspot                                                                       |
| Edit Fields                 | Set                        | Format and map enriched fields         | Aggregate                      | Save data in Hubspot        | ## Save info in Hubspot                                                                       |
| Save data in Hubspot        | Execute Workflow           | Update HubSpot contact record          | Edit Fields                   | None                        | ## Save info in Hubspot                                                                       |
| Sticky Note                 | Sticky Note                | Informational note                     | None                           | None                        | ## Input Parameters<br>** Run this workflow using a form or from another workflow              |
| Sticky Note1                | Sticky Note                | Informational note                     | None                           | None                        | ## Retrieve person info                                                                        |
| Sticky Note2                | Sticky Note                | Informational note                     | None                           | None                        | ## Save info in Hubspot                                                                       |
| Sticky Note7                | Sticky Note                | Detailed README and project overview  | None                           | None                        | README with detailed use case, setup requirements, and next steps <br>Links:<br>- Airtop Profile: https://portal.airtop.ai/browser-profiles<br>- Airtop API Key: https://portal.airtop.ai/api-keys |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Form Trigger** node named "On form submission".  
     - Configure form title: "Enrich person data".  
     - Add required fields:  
       - Person name (string, required)  
       - Work email (string, required)  
       - Airtop Profile (connected to LinkedIn) (string, required)  
       - Hubspot object id (string, required)  
     - Add description: "This automation searches for the contact's info and enriches the data accordingly".  
     - Save webhook URL for form integration.  
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow".  
     - Define workflow inputs matching the form fields: Person name, Work email, Airtop Profile, Hubspot object id.

2. **Create Parameter Unification Node:**  
   - Add a **Set** node named "Unify Params".  
   - Map incoming data fields to normalized variables:  
     - `object_id` ← `Hubspot object id`  
     - `name` ← `Person name`  
     - `email` ← `Work email`  
     - `airtop_profile` ← `Airtop Profile (connected to LinkedIn)`

3. **Create Enrichment Sub-workflow Call:**  
   - Add an **Execute Workflow** node named "Extract person info and calculate ICP".  
   - Set workflow to call: "AIRTOP — Extract person data and calculate ICP" (workflow ID required).  
   - Map inputs:  
     - `work_email` ← `email` from "Unify Params"  
     - `person_name` ← `name` from "Unify Params"  
     - `Airtop_profile` ← `airtop_profile` from "Unify Params"

4. **Add Aggregation Node:**  
   - Add an **Aggregate** node named "Aggregate".  
   - Configure to aggregate all incoming items into a single object.

5. **Add Data Formatting Node:**  
   - Add a **Set** node named "Edit Fields".  
   - Assign fields with expressions referencing "Unify Params" and aggregated data:  
     - `email` ← from unified params  
     - `current_company` ← aggregated data element index 2, field `current_or_last_employer`  
     - `linkedin_company_url` ← aggregated data element index 2, `linkedin_company_url`  
     - `linkedin_connections` ← aggregated data element index 2, `number_of_connections` (number)  
     - `linkedin_followers` ← aggregated data element index 2, `number_of_followers` (number)  
     - `linkedin_url` ← aggregated data element index 1, path `data.modelResponse`  
     - `linkedin_about` ← aggregated data element index 2, `about_section_text`  
     - `icp_person_score` ← aggregated data element index 2, `icp_score` (number)  
     - `seniority_level` ← aggregated data element index 2, `seniority_level`  
     - `technical_depth` ← aggregated data element index 2, `technical_depth`  
     - `ai_interest_level` ← aggregated data element index 2, `ai_interest_level`  
     - `object_id` ← from unified params  
     - `location` ← aggregated data element index 0, `location`  
     - `full_name` ← aggregated data element index 0, `name`

6. **Add HubSpot Update Sub-workflow Call:**  
   - Add an **Execute Workflow** node named "Save data in Hubspot".  
   - Configure to call the workflow "AIRTOP — Update Hubspot contact" (workflow ID required).  
   - Pass the formatted enriched data from "Edit Fields" node as input.

7. **Connect Nodes:**  
   - Connect "On form submission" and "When Executed by Another Workflow" nodes to "Unify Params".  
   - Connect "Unify Params" to "Extract person info and calculate ICP".  
   - Connect "Extract person info and calculate ICP" to "Aggregate".  
   - Connect "Aggregate" to "Edit Fields".  
   - Connect "Edit Fields" to "Save data in Hubspot".

8. **Credential Setup:**  
   - Ensure Airtop API credentials are configured in the enrichment sub-workflow.  
   - Ensure HubSpot OAuth2 credentials and permissions are configured in the HubSpot update sub-workflow.

9. **Testing & Validation:**  
   - Test form submission with valid data.  
   - Test triggering via external workflow.  
   - Validate data enrichment completeness and HubSpot record update success.  
   - Handle and log errors in sub-workflows for troubleshooting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This automation enriches a person’s profile using Airtop linked to LinkedIn and updates HubSpot contacts, ideal for sales, marketing, and recruitment teams. | Workflow purpose and use case                                 |
| Setup requires Airtop API Key and a logged-in Airtop Profile linked to LinkedIn.                                                                             | https://portal.airtop.ai/api-keys<br>https://portal.airtop.ai/browser-profiles |
| HubSpot CRM must have object ID accessible for contact updates.                                                                                            | HubSpot CRM setup requirement                                |
| Next steps include integrating this workflow with lead generation and CRM triggers. Custom ICP scoring logic can be adapted as needed.                     | Workflow extension ideas                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies. All processed data is legal and public.