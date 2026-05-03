Automatically Discover & Enrich HubSpot Buying Groups With Surfe and Google Sheets

https://n8nworkflows.xyz/workflows/automatically-discover---enrich-hubspot-buying-groups-with-surfe-and-google-sheets-7258


# Automatically Discover & Enrich HubSpot Buying Groups With Surfe and Google Sheets

---
### 1. Workflow Overview

This n8n workflow automates the discovery and enrichment of HubSpot buying groups by integrating Surfe API and Google Sheets data, ultimately enriching HubSpot contacts related to deals and notifying users via Gmail. It targets sales and marketing teams seeking to enhance their CRM data with comprehensive contact details based on buying group criteria and company domain information.

The workflow logically breaks down into these key functional blocks:

- **1.1 Trigger and Deal Data Fetching:** Listens for new HubSpot deal creation events, then retrieves associated companies and deal details.
- **1.2 Criteria Extraction and Merging:** Reads buying group criteria from Google Sheets and merges it with company domain data extracted from HubSpot companies.
- **1.3 Surfe People Search and Enrichment:** Uses the Surfe API to search people matching criteria and performs bulk enrichment requests.
- **1.4 Enrichment Status Polling and Data Extraction:** Waits for Surfe enrichment completion, extracts enriched contact details, and filters them.
- **1.5 HubSpot Contact Upsert:** Creates or updates enriched contacts in HubSpot with the enriched data.
- **1.6 Notification Email Preparation and Sending:** Composes a summary email with enriched contacts and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Deal Data Fetching

- **Overview:** Detects new HubSpot deal creation events and fetches associated companies and deal information for subsequent enrichment.
- **Nodes Involved:**  
  - HubSpot Trigger  
  - GET deal associated companies from HUBSPOT  
  - HubSpot Get Company  
  - HubSpot get deal  
  - extract deal info

- **Node Details:**

  - **HubSpot Trigger**  
    - Type: HubSpot event trigger node  
    - Configuration: Listens to "deal.creation" events via HubSpot Developer API OAuth2  
    - Inputs: External event webhook  
    - Outputs: Emits deal creation event data including dealId  
    - Potential Failures: OAuth token expiration, webhook misconfiguration, network timeout  

  - **GET deal associated companies from HUBSPOT**  
    - Type: HTTP Request  
    - Configuration: Uses HubSpot OAuth2 API to get companies associated with the dealId from trigger; URL dynamically built with dealId  
    - Inputs: dealId from trigger  
    - Outputs: JSON list of associated companies  
    - Potential Failures: API rate limits, invalid dealId, auth errors  

  - **HubSpot Get Company**  
    - Type: HubSpot node (company resource get)  
    - Configuration: Retrieves detailed company info by company ID from previous node response  
    - Inputs: company ID from associated companies  
    - Outputs: Company details including domain property  
    - Potential Failures: Missing company ID, permission errors  

  - **HubSpot get deal**  
    - Type: HubSpot node (deal resource get)  
    - Configuration: Retrieves full deal details by dealId  
    - Inputs: dealId from trigger  
    - Outputs: Deal properties including portalId, dealName  
    - Potential Failures: Auth errors, deal not found  

  - **extract deal info**  
    - Type: Code node (JavaScript)  
    - Configuration: Extracts portalId, dealId, dealName from HubSpot get deal node output for use in later email notification  
    - Inputs: HubSpot get deal JSON  
    - Outputs: Simplified JSON with portalId, dealId, dealName  
    - Edge Cases: Missing or null properties handled by fallback nulls  

#### 1.2 Criteria Extraction and Merging

- **Overview:** Reads buying group criteria from Google Sheets, extracts company domains from HubSpot companies, and merges both datasets for enrichment requests.
- **Nodes Involved:**  
  - Google Sheets READ CRITERIAS  
  - extract companyDomains  
  - Merge

- **Node Details:**

  - **Google Sheets READ CRITERIAS**  
    - Type: Google Sheets node  
    - Configuration: Reads rows from the sheet "Your Buying Group Criterias" (gid=0) in a specified Google Sheets document  
    - Inputs: None (triggered downstream)  
    - Outputs: Rows containing departments, jobTitles, seniorities, countries  
    - Credential: Google Sheets OAuth2  
    - Failure Modes: Invalid document ID, permission denied, API limits  

  - **extract companyDomains**  
    - Type: Code node  
    - Configuration: Extracts and deduplicates company domain values from all input items (company info)  
    - Inputs: HubSpot Get Company node output(s)  
    - Outputs: JSON with unique list of companyDomains  
    - Edge Cases: Null or missing domain fields filtered out  

  - **Merge**  
    - Type: Merge node  
    - Configuration: Merges outputs from Google Sheets and extract companyDomains nodes  
    - Inputs: Buying group criteria and company domains  
    - Outputs: Combined JSON payload for subsequent search  
    - Notes: Ensures criteria and domains are combined correctly for API calls  

#### 1.3 Surfe People Search and Enrichment

- **Overview:** Performs a search for people matching criteria in Surfe, prepares a bulk enrichment request, and sends it to Surfe API.
- **Nodes Involved:**  
  - prepare JSON PAYLAOD for Person Search  
  - Search People in Companies  
  - Prepare JSON Payload Enrichment Request  
  - Surfe Bulk Enrichments API

- **Node Details:**

  - **prepare JSON PAYLAOD for Person Search**  
    - Type: Code node  
    - Configuration: Extracts and deduplicates criteria (departments, jobTitles, seniorities, countries) and companyDomains from all input items, formats JSON payload with limit 200  
    - Inputs: Merged data from criteria and companyDomains  
    - Outputs: JSON payload for Surfe people search API  
    - Edge Cases: Handles empty or missing criteria gracefully  

  - **Search People in Companies**  
    - Type: HTTP Request  
    - Configuration: POST to Surfe /v2/people/search API with JSON body from previous node; uses Bearer token authentication  
    - Inputs: JSON payload from code node  
    - Outputs: Search results with people array  
    - Failure Modes: Auth token invalid, API error, network issues  

  - **Prepare JSON Payload Enrichment Request**  
    - Type: Code node  
    - Configuration: Maps Surfe search response people to enrichment request format, including fields to enrich (email, mobile, jobHistory), and generates externalID for each person  
    - Inputs: Surfe people search response JSON  
    - Outputs: JSON payload for Surfe bulk enrichment API  
    - Edge Cases: Missing person fields defaulted to empty strings  

  - **Surfe Bulk Enrichments API**  
    - Type: HTTP Request  
    - Configuration: POST to Surfe /v2/people/enrich API with enrichment request JSON; Bearer token auth  
    - Inputs: JSON payload from previous node  
    - Outputs: Enrichment request response with enrichmentID  
    - Failure Modes: API limits, invalid payload, auth failure  

#### 1.4 Enrichment Status Polling and Data Extraction

- **Overview:** Polls Surfe enrichment status until completion, then extracts enriched contact details and filters contacts having phone or email.
- **Nodes Involved:**  
  - Surfe check enrichement status  
  - Is enrichment complete ?  
  - Wait 3 secondes  
  - Extract list of peoples from Surfe API response  
  - Filter: phone AND email

- **Node Details:**

  - **Surfe check enrichement status**  
    - Type: HTTP Request  
    - Configuration: GET request to Surfe API /v2/people/enrich/{enrichmentID} using Bearer token  
    - Inputs: enrichmentID from enrichment submission response  
    - Outputs: Enrichment status JSON  
    - Failure Modes: Network timeout, invalid enrichmentID, auth errors  

  - **Is enrichment complete ?**  
    - Type: If node  
    - Configuration: Checks if JSON.status equals "COMPLETED"  
    - True path: proceeds to extract people  
    - False path: loops back to Wait node to retry  
    - Edge Cases: Other status values not handled explicitly (e.g., FAILED)  

  - **Wait 3 secondes**  
    - Type: Wait node  
    - Configuration: Waits 3 seconds before polling Surfe status again  
    - Inputs: From False branch of If node  
    - Outputs: Triggers Surfe check again  

  - **Extract list of peoples from Surfe API response**  
    - Type: Code node  
    - Configuration: Extracts relevant person fields (id, firstName, lastName, email, phone, jobTitle, companyName, linkedinUrl, country, status) from Surfe enrichment response  
    - Inputs: Surfe enrichment completed response  
    - Outputs: List of contacts in simplified JSON format  
    - Edge Cases: Missing or empty fields gracefully handled  

  - **Filter: phone AND email**  
    - Type: Filter node  
    - Configuration: Passes only contacts that have non-empty phone OR email fields  
    - Inputs: Extracted people list  
    - Outputs: Contacts fulfilling contactability criteria  

#### 1.5 HubSpot Contact Upsert

- **Overview:** Creates or updates HubSpot contacts using enriched data, ensuring CRM reflects latest enriched contact information.
- **Nodes Involved:**  
  - HubSpot: Create or Update  
  - Merge1

- **Node Details:**

  - **HubSpot: Create or Update**  
    - Type: HubSpot node (contact create/update)  
    - Configuration: Uses email as unique identifier, sets contact fields such as country, jobTitle, firstName, lastName, websiteUrl (linkedInUrl), companyName, phoneNumber, mobilePhoneNumber  
    - Inputs: Filtered contacts with phone or email  
    - Outputs: HubSpot contact creation/update response  
    - Credential: HubSpot App Token  
    - Failure Modes: Duplicate emails, API errors, auth failures  

  - **Merge1**  
    - Type: Merge node  
    - Configuration: Merges contacts update output with extracted deal info for final notification  
    - Inputs: Outputs from HubSpot contact upsert and extract deal info node  
    - Outputs: Combined data for email preparation  

#### 1.6 Notification Email Preparation and Sending

- **Overview:** Prepares a detailed HTML and plain-text summary email of enriched contacts linked to the HubSpot deal and sends notification via Gmail.
- **Nodes Involved:**  
  - prepare email content  
  - Gmail

- **Node Details:**

  - **prepare email content**  
    - Type: Code node  
    - Configuration:  
      - Builds HTML table summarizing enriched contacts with links to HubSpot contact records and deal page  
      - Extracts deal info, buying group criteria, company domain  
      - Generates both HTML and plain-text versions of the message  
      - Handles missing data gracefully (e.g., unknown domains)  
    - Inputs: Merged contacts and deal info  
    - Outputs: JSON with subject, HTML message, and text message for email node  

  - **Gmail**  
    - Type: Gmail node  
    - Configuration: Sends email to a fixed address ("youdestinationremail@gmail.com") with subject and HTML body from previous node  
    - Credential: Gmail OAuth2  
    - Behavior: Executes once per workflow run (executeOnce: true)  
    - Failure Modes: SMTP errors, auth token expiration, invalid email address  

---

### 3. Summary Table

| Node Name                         | Node Type                | Functional Role                          | Input Node(s)                         | Output Node(s)                     | Sticky Note                                         |
|----------------------------------|--------------------------|----------------------------------------|-------------------------------------|----------------------------------|-----------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Manual start point (for testing)       | None                                | prepare JSON PAYLAOD for Person Search |                                                     |
| HubSpot Trigger                  | HubSpot Trigger          | Trigger on HubSpot deal creation       | None                                | GET deal associated companies from HUBSPOT |                                                     |
| GET deal associated companies from HUBSPOT | HTTP Request           | Get companies linked to deal           | HubSpot Trigger                    | HubSpot Get Company               |                                                     |
| HubSpot Get Company             | HubSpot                  | Retrieve company details                | GET deal associated companies from HUBSPOT | Google Sheets READ CRITERIAS      |                                                     |
| Google Sheets READ CRITERIAS    | Google Sheets            | Read buying group criteria              | HubSpot Get Company                 | Merge                           |                                                     |
| extract companyDomains           | Code                     | Extract unique company domains          | HubSpot Get Company                 | Merge                           |                                                     |
| Merge                          | Merge                    | Merge criteria with company domains     | Google Sheets READ CRITERIAS, extract companyDomains | prepare JSON PAYLAOD for Person Search |                                                     |
| prepare JSON PAYLAOD for Person Search | Code                     | Build search payload for Surfe people  | Merge                             | Search People in Companies        |                                                     |
| Search People in Companies      | HTTP Request             | Search people in Surfe API              | prepare JSON PAYLAOD for Person Search | Prepare JSON Payload Enrichment Request |                                                     |
| Prepare JSON Payload Enrichment Request | Code                     | Format enrichment request JSON          | Search People in Companies          | Surfe Bulk Enrichments API        |                                                     |
| Surfe Bulk Enrichments API      | HTTP Request             | Submit bulk enrichment request          | Prepare JSON Payload Enrichment Request | Surfe check enrichement status    |                                                     |
| Surfe check enrichement status  | HTTP Request             | Poll enrichment status                   | Surfe Bulk Enrichments API, Wait 3 secondes | Is enrichment complete ?          |                                                     |
| Is enrichment complete ?         | If                       | Check if enrichment is completed         | Surfe check enrichement status     | Extract list of peoples from Surfe API response (true), Wait 3 secondes (false) |                                                     |
| Wait 3 secondes                 | Wait                     | Pause before rechecking enrichment status | Is enrichment complete ? (false)   | Surfe check enrichement status    |                                                     |
| Extract list of peoples from Surfe API response | Code                     | Extract enriched contact data            | Is enrichment complete ? (true)    | Filter: phone AND email           |                                                     |
| Filter: phone AND email          | Filter                   | Filter contacts with phone or email     | Extract list of peoples from Surfe API response | HubSpot: Create or Update         |                                                     |
| HubSpot: Create or Update        | HubSpot                  | Upsert enriched contacts in HubSpot     | Filter: phone AND email             | Merge1                          |                                                     |
| HubSpot get deal                 | HubSpot                  | Fetch full deal details                  | HubSpot Trigger                    | extract deal info                 |                                                     |
| extract deal info               | Code                     | Extract portalId, dealId, dealName       | HubSpot get deal                   | Merge1                          |                                                     |
| Merge1                         | Merge                    | Merge contact upsert and deal info      | HubSpot: Create or Update, extract deal info | prepare email content             |                                                     |
| prepare email content           | Code                     | Build notification email content         | Merge1                           | Gmail                           |                                                     |
| Gmail                         | Gmail                    | Send notification email                   | prepare email content             | None                           | Notify end of all batches                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**  
   - Type: Manual Trigger  
   - Purpose: Manual start for testing or debugging.

2. **Create HubSpot Trigger Node**  
   - Event: "deal.creation"  
   - Credentials: HubSpot Developer API OAuth2  
   - Purpose: Trigger workflow on new deal creation.

3. **Add HTTP Request Node to Get Associated Companies**  
   - URL: `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.dealId }}/associations/companies`  
   - Method: GET  
   - Authentication: HubSpot OAuth2  
   - Input: dealId from HubSpot Trigger.

4. **Add HubSpot Node to Get Company Details**  
   - Resource: Company  
   - Operation: Get  
   - Company ID: `={{ $json.results[0].id }}` from previous node  
   - Authentication: HubSpot App Token.

5. **Add Google Sheets Node to Read Buying Group Criteria**  
   - Document ID: URL of your Google Sheet  
   - Sheet Name: `gid=0` or appropriate sheet  
   - Authentication: Google Sheets OAuth2.

6. **Add Code Node to Extract Unique Company Domains**  
   - Input: HubSpot Get Company output  
   - Extract and deduplicate `properties.domain.value` fields.

7. **Add Merge Node**  
   - Merge criteria from Google Sheets and company domains  
   - Mode: Append with proper input order.

8. **Add Code Node to Prepare JSON Payload for Surfe People Search**  
   - Extract and deduplicate criteria fields: departments, jobTitles, seniorities, countries, and companyDomains  
   - Compose JSON with these fields and limit=200.

9. **Add HTTP Request Node to Surfe People Search API**  
   - URL: `https://api.surfe.com/v2/people/search`  
   - Method: POST  
   - Body: JSON from previous node  
   - Authentication: Bearer Token (Surfe).

10. **Add Code Node to Prepare Enrichment Request Payload**  
    - Map Surfe search results to enrichment request format with include options for email, mobile, jobHistory  
    - Generate externalID as lowercase sanitized string.

11. **Add HTTP Request Node to Surfe Bulk Enrichments API**  
    - URL: `https://api.surfe.com/v2/people/enrich`  
    - Method: POST  
    - Body: JSON from previous node  
    - Authentication: Bearer Token.

12. **Add HTTP Request Node to Check Enrichment Status**  
    - URL: `https://api.surfe.com/v2/people/enrich/{{ $json.enrichmentID }}`  
    - Method: GET  
    - Authentication: Bearer Token.

13. **Add If Node to Check if Enrichment Status is "COMPLETED"**  
    - Condition: `$json.status === "COMPLETED"`  
    - True: Proceed  
    - False: Wait 3 seconds then re-check.

14. **Add Wait Node**  
    - Duration: 3 seconds  
    - Connect from False branch of If node back to Check Enrichment Status node.

15. **Add Code Node to Extract People from Enrichment Response**  
    - Extract fields: id, firstName, lastName, email, phone, jobTitle, companyName, companyWebsite, linkedinUrl, country, status.

16. **Add Filter Node to Filter People with Phone or Email**  
    - Condition: phone not empty OR email not empty.

17. **Add HubSpot Node to Create or Update Contacts**  
    - Resource: Contact  
    - Operation: Create or Update by Email  
    - Map fields: country, jobTitle, lastName, firstName, websiteUrl (linkedInUrl), companyName, phoneNumber, mobilePhoneNumber  
    - Authentication: HubSpot App Token.

18. **Add HubSpot Node to Get Deal Details**  
    - Resource: Deal  
    - Operation: Get  
    - ID: From trigger's dealId  
    - Authentication: HubSpot App Token.

19. **Add Code Node to Extract Deal Info**  
    - Extract portalId, dealId, dealName for email content.

20. **Add Merge Node**  
    - Merge outputs from HubSpot contact upsert and deal info nodes.

21. **Add Code Node to Prepare Email Content**  
    - Compose HTML and plain-text message summarizing enriched contacts  
    - Include links to HubSpot deal and contacts  
    - Include buying group criteria and company domain.

22. **Add Gmail Node to Send Notification Email**  
    - To: `youdestinationremail@gmail.com` (customize)  
    - Subject and message from previous node  
    - Credential: Gmail OAuth2  
    - Execute once per workflow run.

23. **Connect the nodes following the logical flow described, ensuring data and parameters propagate correctly.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow enriches HubSpot contacts based on buying group criteria and company domains using Surfe API and Google Sheets data.  | Workflow purpose                                                                                         |
| Gmail node sends notification email upon completion summarizing enriched contacts with direct HubSpot links.                   | Gmail notification feature                                                                               |
| HubSpot OAuth2 and App Token credentials used for different API calls (trigger and data retrieval vs. contact upsert).          | Credential management                                                                                     |
| Surfe API requires Bearer Token for authentication; ensure tokens are valid and have required scopes.                          | Surfe API integration details                                                                            |
| Google Sheets OAuth2 credentials must have read access to specified sheet containing buying group criteria.                    | Google Sheets integration                                                                                |
| API rate limits and token expiration are potential failure points; consider adding error handling and retry logic if needed.   | General integration best practices                                                                       |
| For more info on Surfe API endpoints used: https://docs.surfe.com/api                                                            | External API documentation link                                                                          |

---

**Disclaimer:** The provided text is based solely on an automated workflow created with n8n, an integration and automation tool. All processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.