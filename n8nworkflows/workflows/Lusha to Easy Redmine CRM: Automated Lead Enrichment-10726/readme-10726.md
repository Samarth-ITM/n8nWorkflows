Lusha to Easy Redmine CRM: Automated Lead Enrichment

https://n8nworkflows.xyz/workflows/lusha-to-easy-redmine-crm--automated-lead-enrichment-10726


# Lusha to Easy Redmine CRM: Automated Lead Enrichment

---

### 1. Workflow Overview

This workflow automates the enrichment of lead data stored in Easy Redmine CRM by fetching additional contact and company information from the Lusha API. It is designed to help sales and marketing teams maintain up-to-date and enriched lead profiles, reducing manual data entry and improving lead quality for outreach.

The workflow consists of the following logical blocks:

- **1.1 Schedule Trigger:** Initiates the workflow on a daily schedule.
- **1.2 Easy Redmine Lead Retrieval:** Fetches multiple lead records from Easy Redmine CRM using a saved query.
- **1.3 Data Splitting:** Splits the fetched lead array into individual lead items for processing.
- **1.4 Lusha Data Fetch:** Requests enriched contact and company data from the Lusha API for each lead.
- **1.5 Lead Filtering:** Filters out leads for which Lusha did not return valid data.
- **1.6 Data Transformation:** Transforms and cleans Lusha data into a format compatible with Easy Redmine CRM.
- **1.7 CRM Update:** Updates the enriched lead data back into Easy Redmine CRM via HTTP PUT requests.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block initiates the workflow execution once per day at a specified hour (10:00 AM).

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Node:** Schedule Trigger  
  - **Type:** n8n-nodes-base.scheduleTrigger  
  - **Configuration:** Trigger set to run daily at 10:00 AM (hour 10).  
  - **Expressions/Variables:** None.  
  - **Input:** None (starting node).  
  - **Output:** Triggers next node "Get Leads from Easy Redmine".  
  - **Version Requirements:** n8n version supporting scheduleTrigger v1.2.  
  - **Potential Failures:** Scheduler misconfiguration, n8n downtime.  
  - **Sub-workflows:** None.

---

#### 2.2 Easy Redmine Lead Retrieval

- **Overview:**  
  Retrieves a collection of lead records from Easy Redmine CRM using a predefined query ID.

- **Nodes Involved:**  
  - Get Leads from Easy Redmine

- **Node Details:**  
  - **Node:** Get Leads from Easy Redmine  
  - **Type:** @easysoftware/n8n-nodes-easy-redmine.easyRedmine  
  - **Configuration:**  
    - Resource: easy_leads  
    - leadQueryId: 1234 (saved filter in Easy Redmine to select relevant leads, e.g., leads created today)  
    - Credentials: Easy Redmine API credentials with appropriate permissions.  
  - **Expressions/Variables:** None dynamic; static query ID.  
  - **Input:** From Schedule Trigger.  
  - **Output:** Lead records array to "Split Out".  
  - **Version Requirements:** Requires Easy Redmine node version 1 or higher with API access.  
  - **Potential Failures:** API authentication errors, invalid query ID, network issues, permission denied.  
  - **Sub-workflows:** None.

---

#### 2.3 Data Splitting

- **Overview:**  
  Splits the array of leads into individual lead items for further processing and enrichment.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**  
  - **Node:** Split Out  
  - **Type:** n8n-nodes-base.splitOut  
  - **Configuration:**  
    - Field to split out: "easy_leads" (field containing array of lead items)  
  - **Expressions/Variables:** None.  
  - **Input:** From "Get Leads from Easy Redmine".  
  - **Output:** Individual lead JSON objects forwarded to "Get Data from Lusha".  
  - **Version Requirements:** v1 or higher.  
  - **Potential Failures:** Empty lead array causing no output, invalid or missing field "easy_leads".  
  - **Sub-workflows:** None.

---

#### 2.4 Lusha Data Fetch

- **Overview:**  
  For each lead, sends an authenticated HTTP GET request to the Lusha API to retrieve enriched personal and company data.

- **Nodes Involved:**  
  - Get Data from Lusha

- **Node Details:**  
  - **Node:** Get Data from Lusha  
  - **Type:** n8n-nodes-base.httpRequest  
  - **Configuration:**  
    - HTTP Method: GET  
    - URL: https://api.lusha.com/v2/person  
    - Query Parameters (populated dynamically):  
      - email = {{$json.email}}  
      - filterBy = "emailAddresses"  
      - firstName = {{$json.first_name}}  
      - lastName = {{$json.last_name}}  
      - companyName = {{$json.company_name}}  
    - Headers: Content-Type: application/json  
    - Authentication: HTTP Header Auth using stored Lusha API key credentials  
  - **Expressions/Variables:** Uses lead fields dynamically for query parameters.  
  - **Input:** Individual lead JSON from "Split Out".  
  - **Output:** Lusha API response JSON to "Filter Leads Found in Lusha".  
  - **Version Requirements:** HTTP Request node v4.2 or higher; Lusha API access.  
  - **Potential Failures:** API rate limits, authentication failure, invalid parameters, network timeout, empty or malformed responses.  
  - **Sub-workflows:** None.

---

#### 2.5 Lead Filtering

- **Overview:**  
  Filters out any leads where Lusha returned an error or no data, allowing only successful enrichments to proceed.

- **Nodes Involved:**  
  - Filter Leads Found in Lusha

- **Node Details:**  
  - **Node:** Filter Leads Found in Lusha  
  - **Type:** n8n-nodes-base.filter  
  - **Configuration:**  
    - Condition: Exclude items where the field "contact.error.name" exists (i.e., error present).  
  - **Expressions/Variables:** Checks for non-existence of error in response JSON.  
  - **Input:** From "Get Data from Lusha".  
  - **Output:** Only successful enrichment data to "Contact Data Transformation for CRM".  
  - **Version Requirements:** Filter node v2.2 or higher.  
  - **Potential Failures:** Unexpected JSON structure, missing expected fields causing false negatives or positives.  
  - **Sub-workflows:** None.

---

#### 2.6 Data Transformation

- **Overview:**  
  Transforms the nested Lusha API response into a flattened, CRM-compatible JSON structure, extracting key contact and company information.

- **Nodes Involved:**  
  - Contact Data Transformation for CRM

- **Node Details:**  
  - **Node:** Contact Data Transformation for CRM  
  - **Type:** n8n-nodes-base.code (JavaScript)  
  - **Configuration:**  
    - Executes custom JavaScript to:  
      - Skip records with errors.  
      - Extract primary and secondary emails and phones.  
      - Convert company size and revenue ranges (arrays) to upper bounds.  
      - Extract and flatten fields like name, job title, location, LinkedIn, previous job, and meta info.  
  - **Key Expressions:** Uses JavaScript map over all input items to produce transformed output.  
  - **Input:** Filtered Lusha data from "Filter Leads Found in Lusha".  
  - **Output:** Transformed lead data JSON to "Update Leads in Easy Redmine CRM".  
  - **Version Requirements:** Code node v2 or higher supporting ES6 syntax.  
  - **Potential Failures:** Code errors due to unexpected data shape, null references, or missing fields.  
  - **Sub-workflows:** None.

---

#### 2.7 CRM Update

- **Overview:**  
  Sends HTTP PUT requests to Easy Redmine CRM to update each lead record with enriched phone, employee count, and LinkedIn URL.

- **Nodes Involved:**  
  - Update Leads in Easy Redmine CRM

- **Node Details:**  
  - **Node:** Update Leads in Easy Redmine CRM  
  - **Type:** n8n-nodes-base.httpRequest  
  - **Configuration:**  
    - HTTP Method: PUT  
    - URL: Dynamic; constructed as https://easy-redmine-application.com/easy_leads/{{ lead_id }}.json where lead_id is extracted from the original lead ("Split Out").  
    - Request Body (JSON):  
      ```json
      {
        "easy_lead": {
          "mobile_phone": "{{ phone or empty string }}",
          "number_of_employees": "companySize or 0",
          "custom_fields": [
            {
              "id": 737,
              "value": "{{ linkedinUrl or null }}"
            }
          ]
        }
      }
      ```  
    - Batching enabled with batch size 1 (to avoid API overload).  
    - Authentication: HTTP Header Auth with Easy Redmine API credentials.  
  - **Expressions/Variables:** Uses expressions to safely handle missing phone or company size, and populates custom field 737 with LinkedIn URL.  
  - **Input:** Transformed lead data from "Contact Data Transformation for CRM".  
  - **Output:** Final output; no further nodes.  
  - **Version Requirements:** HTTP Request node v4.2 or higher.  
  - **Potential Failures:** API authentication errors, invalid lead IDs, malformed JSON body, network issues, rate limiting.  
  - **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name                        | Node Type                               | Functional Role                                  | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|---------------------------------|---------------------------------------|-------------------------------------------------|---------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                | n8n-nodes-base.scheduleTrigger        | Initiates workflow daily at 10:00 AM            | None                            | Get Leads from Easy Redmine      |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Get Leads from Easy Redmine     | @easysoftware/n8n-nodes-easy-redmine.easyRedmine | Fetches lead records from Easy Redmine CRM       | Schedule Trigger                | Split Out                       | **1) Schedule Trigger** — Triggers the workflow on a time-based schedule  \n**2) Get Leads from Easy Redmine** — Fetches multiple lead records from Easy Redmine for one day (Today)                                                                                                                                                                                                                                                                                                                                                                                  |
| Split Out                      | n8n-nodes-base.splitOut                | Splits lead array into individual lead items     | Get Leads from Easy Redmine     | Get Data from Lusha             | **3) Split Out** — Splits the fetched data into individual items for processing                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Get Data from Lusha             | n8n-nodes-base.httpRequest             | Fetches enriched lead data from Lusha API        | Split Out                      | Filter Leads Found in Lusha     | **4) Get Data from Lusha** — Retrieves data from Lusha via HTTP request                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Filter Leads Found in Lusha     | n8n-nodes-base.filter                  | Filters out leads without valid Lusha data       | Get Data from Lusha             | Contact Data Transformation for CRM | **5) Filter Leads Found in Lusha** — Excludes rows where the data field is empty                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Contact Data Transformation for CRM | n8n-nodes-base.code                    | Transforms and cleans Lusha data for CRM update  | Filter Leads Found in Lusha     | Update Leads in Easy Redmine CRM | **6) Contact Data Transformation for CRM** — Transforms and cleans the data fields                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Update Leads in Easy Redmine CRM | n8n-nodes-base.httpRequest             | Updates enriched lead data back into Easy Redmine CRM | Contact Data Transformation for CRM | None                          | **7) Update Leads in Easy Redmine CRM** — Sends updated contact and company info (phone, employees, LinkedIn) to Easy Redmine CRM                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note                    | n8n-nodes-base.stickyNote              | Documentation note                               | None                           | None                           | ## Automated Lead Enrichment from Lusha to Easy Redmine CRM  \n**Try out using a native Easy Redmine node to enrich CRM lead data with Lusha fields.**\n\n### About Workflow  \nThis workflow fetches enriched Leads data from Lusha, cleans and formats it, and pushes it to Easy Redmine CRM. It is triggered on a schedule and runs fully automated once set up.\n\n### Use Case  \nDesigned for sales and marketing teams using Easy Redmine CRM. It automates the update of Leads with enriched Lusha data—saving manual effort and ensuring updated info like phone, employee count, and LinkedIn.\n\n### How it works  \n- Time-based trigger  \n=> Runs on a schedule defined in Schedule Trigger  \n- Data fetch  \n=> Pulls lead data using “get-many-easy_leads”  \n- Split Out\n=> Splits array → filters empty phone rows → transforms employee count (e.g., \"1,000-5,000\" → 5000)  \n- HTTP Request \n=> Gets GET information from Lusha  \n- Final output step  \n=> Sends PUT requests to Easy Redmine CRM to update lead records\n\n### How to use  \n- Set the **Schedule Trigger** to define how often the workflow should run  \n- In the **\"Get Leads from Easy Redmine\"** node, apply a saved filter in Easy Redmine to target the correct lead records  (ID Query)\n- Ensure the Lusha API connection is working and returns enriched data for the selected leads  \n- Adjust the **\"Contact Data Transformation for CRM\"** node to clean and map fields as needed—e.g., format employee numbers, remove unwanted characters, or align with CRM field structure  \n- Run tests on a small data sample to confirm that updates are correctly applied in Easy Redmine CRM before enabling full automation\n\n\n### Requirements  \n- Easy Redmine application\n=> ideally technical user for API calls with specific permissions\n- Lusha access \n\n### Need Help? \n- Reach out through n8n community => https://community.n8n.io/u/easy8.ai\n- Contant our team directly => Easy8.ai\n- Visit our youtube channel => https://www.youtube.com/@easy8ai \n |
| Sticky Note1                   | n8n-nodes-base.stickyNote              | Documentation note                               | None                           | None                           | ##  Automated Lead Enrichment from Lusha to Easy Redmine CRM  \n**1) Schedule Trigger** — Triggers the workflow on a time-based schedule  \n**2) Get Leads from Easy Redmine** — Fetches multiple lead records from Easy Redmine for one day (Today)\n**3) Split Out** — Splits the fetched data into individual items for processing  \n**4) Get Data from Lusha** — Retrieves data from Lusha via HTTP request  \n**5) Filter Leads Found in Lusha** — Excludes rows where the data field is empty  \n**6) Contact Data Transformation for CRM** — Transforms and cleans the data fields\n**7) Update Leads in Easy Redmine CRM** — Sends updated contact and company info (phone, employees, LinkedIn) to Easy Redmine CRM  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger daily at 10:00 AM (hour 10).  
   - No credentials needed.  
   - Connect this node’s output to "Get Leads from Easy Redmine".

2. **Create Get Leads from Easy Redmine Node**  
   - Type: Easy Redmine node (@easysoftware/n8n-nodes-easy-redmine.easyRedmine)  
   - Parameters:  
     - Resource: easy_leads  
     - leadQueryId: 1234 (replace with your saved query ID in Easy Redmine)  
   - Credentials: Assign Easy Redmine API credentials with API access and required permissions.  
   - Connect input to Schedule Trigger node.  
   - Connect output to "Split Out".

3. **Create Split Out Node**  
   - Type: Split Out  
   - Parameters: Field to split out: "easy_leads" (field name containing array of leads in the previous node's output).  
   - Connect input from "Get Leads from Easy Redmine".  
   - Connect output to "Get Data from Lusha".

4. **Create Get Data from Lusha Node**  
   - Type: HTTP Request  
   - Parameters:  
     - HTTP Method: GET  
     - URL: https://api.lusha.com/v2/person  
     - Query Parameters:  
       - email = {{$json.email}}  
       - filterBy = "emailAddresses"  
       - firstName = {{$json.first_name}}  
       - lastName = {{$json.last_name}}  
       - companyName = {{$json.company_name}}  
     - Headers: Content-Type = application/json  
     - Authentication: HTTP Header Auth  
   - Credentials: Add Lusha API key credentials (HTTP Header Auth).  
   - Connect input from "Split Out".  
   - Connect output to "Filter Leads Found in Lusha".

5. **Create Filter Leads Found in Lusha Node**  
   - Type: Filter  
   - Parameters:  
     - Condition: Exclude items where "contact.error.name" exists (i.e., is not empty).  
   - Connect input from "Get Data from Lusha".  
   - Connect output to "Contact Data Transformation for CRM".

6. **Create Contact Data Transformation for CRM Node**  
   - Type: Code (JavaScript)  
   - Parameters: Paste the following JavaScript code:

```javascript
const items = $input.all();

const transformedItems = items.map((item) => {
  const contact = item.json.contact;
  
  // Skip if there's an error
  if (contact.error) {
    return {
      json: {
        error: contact.error,
        isCreditCharged: contact.isCreditCharged
      }
    };
  }
  
  const data = contact.data;
  
  // Transform company size array [min, max] to just the max number
  let companySize = null;
  if (data.company?.companySize && Array.isArray(data.company.companySize)) {
    companySize = data.company.companySize[1]; // Take the upper bound
  }
  
  // Transform revenue range array [min, max] to just the max number
  let revenueRange = null;
  if (data.company?.revenueRange && Array.isArray(data.company.revenueRange)) {
    revenueRange = data.company.revenueRange[1]; // Take the upper bound
  }
  
  // Extract primary email (first in array)
  const primaryEmail = data.emailAddresses?.[0]?.email || null;
  const primaryEmailType = data.emailAddresses?.[0]?.emailType || null;
  const primaryEmailConfidence = data.emailAddresses?.[0]?.emailConfidence || null;
  
  // Extract primary phone (first in array)
  const primaryPhone = data.phoneNumbers?.[0]?.number || null;
  const primaryPhoneType = data.phoneNumbers?.[0]?.phoneType || null;
  
  // Extract secondary phone (second in array)
  const secondaryPhone = data.phoneNumbers?.[1]?.number || null;
  const secondaryPhoneType = data.phoneNumbers?.[1]?.phoneType || null;
  
  // Build flattened object for CRM
  return {
    json: {
      // Person Info
      firstName: data.firstName || null,
      lastName: data.lastName || null,
      fullName: data.fullName || null,
      personId: data.personId || null,
      
      // Contact Details
      email: primaryEmail,
      emailType: primaryEmailType,
      emailConfidence: primaryEmailConfidence,
      phone: primaryPhone,
      phoneType: primaryPhoneType,
      secondaryPhone: secondaryPhone,
      secondaryPhoneType: secondaryPhoneType,
      linkedinUrl: data.socialLinks?.linkedin || null,
      
      // Job Info
      jobTitle: data.jobTitle?.title || null,
      jobDepartment: data.jobTitle?.departments?.[0] || null,
      jobSeniority: data.jobTitle?.seniority || null,
      jobStartDate: data.jobStartDate || null,
      
      // Location
      country: data.location?.country || null,
      countryIso2: data.location?.country_iso2 || null,
      city: data.location?.city || null,
      continent: data.location?.continent || null,
      isEuContact: data.location?.is_eu_contact || false,
      
      // Company Info
      companyName: data.company?.name || null,
      companyId: data.companyId || null,
      companyDomain: data.company?.domains?.homepage || null,
      companyWebsite: data.company?.homepageUrl || null,
      companyDescription: data.company?.description || null,
      companySize: companySize,
      companyRevenue: revenueRange,
      companyIndustry: data.company?.mainIndustry || null,
      companySubIndustry: data.company?.subIndustry || null,
      companyCity: data.company?.location?.city || null,
      companyCountry: data.company?.location?.country || null,
      companyLinkedin: data.company?.social?.linkedin || null,
      companyLogoUrl: data.company?.logoUrl || null,
      
      // Previous Job Info
      previousCompanyName: data.previousJob?.company?.name || null,
      previousCompanyDomain: data.previousJob?.company?.domain || null,
      previousJobTitle: data.previousJob?.jobTitle?.title || null,
      
      // Meta
      isCreditCharged: contact.isCreditCharged,
      updateDate: data.updateDate || null
    }
  };
});

return transformedItems;
```

   - Connect input from "Filter Leads Found in Lusha".  
   - Connect output to "Update Leads in Easy Redmine CRM".

7. **Create Update Leads in Easy Redmine CRM Node**  
   - Type: HTTP Request  
   - Parameters:  
     - HTTP Method: PUT  
     - URL: `=https://easy-redmine-application.com/easy_leads/{{ $('Split Out').item.json.id }}.json` (replace domain with your Easy Redmine instance URL)  
     - Request Body (JSON):  
```json
{
  "easy_lead": {
    "mobile_phone": {{ $json.phone ? JSON.stringify($json.phone) : '""' }},
    "number_of_employees": {{ $json.companySize != null ? $json.companySize : 0 }},
    "custom_fields": [
      {
        "id": 737,
        "value": {{ $json.linkedinUrl ? JSON.stringify($json.linkedinUrl) : null }}
      }
    ]
  }
}
```
     - Enable batching with batch size 1 to avoid overload.  
     - Authentication: HTTP Header Auth with Easy Redmine API credentials set.  
   - Connect input from "Contact Data Transformation for CRM".  
   - No output connection needed (final step).

8. **Add Sticky Notes (optional)**  
   - Add two sticky notes with the provided documentation content to assist users.

9. **Activate and Test**  
   - Test the workflow with a small set of leads to verify successful enrichment and updates.  
   - Monitor API rate limits and errors during execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                        |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| Try out using a native Easy Redmine node to enrich CRM lead data with Lusha fields. This workflow fetches enriched Leads data from Lusha, cleans and formats it, and pushes it to Easy Redmine CRM. It is triggered on a schedule and runs fully automated once set up. Designed for sales and marketing teams using Easy Redmine CRM to automate lead enrichment and save manual effort.                                                                                              | Sticky Note content in workflow.                                     |
| For help or questions, reach out via the n8n community: https://community.n8n.io/u/easy8.ai or contact Easy8.ai directly. Also visit their YouTube channel: https://www.youtube.com/@easy8ai.                                                                                                                                                                                                                                                                                                                               | Support and community resources.                                    |
| Ensure the Easy Redmine API user has sufficient permissions to query and update leads. Lusha API access requires an active subscription and API key with quota for enrichment requests. Test on small samples before full automation to avoid unexpected data overwrites or API quota exhaustion.                                                                                                                                                                                                                           | Usage notes and requirements.                                       |
| The workflow transforms ranged fields from Lusha (e.g., company size and revenue) by taking the upper bound value to simplify CRM integration. It also flatten nested data for better compatibility with Easy Redmine custom fields.                                                                                                                                                                                                                                                                                           | Data transformation rationale.                                      |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---