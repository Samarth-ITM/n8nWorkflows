Enrich lead that booked a call on Calendly and save it on Hubspot

https://n8nworkflows.xyz/workflows/workflow-2129-1747220446346.png


# Enrich lead that booked a call on Calendly and save it on Hubspot

### 1. Workflow Overview

This workflow automates the enrichment and CRM updating process for new leads who book calls via Calendly. Its primary goal is to maximize call effectiveness by gathering detailed lead and company information automatically, reducing manual research effort.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception**: Triggered by a new Calendly booking event.
- **1.2 Email Filtering**: Filters out common personal email domains to focus on business leads.
- **1.3 Email Enrichment**: Uses Clearbit to enrich the lead’s email with detailed personal and employment data.
- **1.4 Company Enrichment & CRM Lookup**: If the lead is associated with a company, enrich company data via Clearbit and search the company in Hubspot CRM.
- **1.5 CRM Company Upsert**: Create or update the company record in Hubspot depending on its existence.
- **1.6 CRM Contact Upsert**: Create or update the contact (lead) in Hubspot, associating it with the company if applicable.
- **1.7 No-Op for Missing Contact**: Graceful handling if no contact data is found.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for new call bookings on Calendly and initiates the workflow with the booking data.

**Nodes Involved:**  
- Calendly Trigger

**Node Details:**  
- **Calendly Trigger**  
  - Type: Trigger node for Calendly webhook events  
  - Configuration: Listens to `invitee.created` event, which fires when a new invitee schedules a call  
  - Credentials: Uses configured Calendly API credentials  
  - Input/Output: No input; outputs booking payload including invitee email and details  
  - Edge Cases:  
    - Webhook misconfiguration or credential failure will prevent triggers  
    - Rate limits or network errors on Calendly API  
  - Version: v1

---

#### 2.2 Email Filtering

**Overview:**  
Filters out leads with personal email domains (e.g., gmail.com, yahoo.com) to focus enrichment on business emails.

**Nodes Involved:**  
- Filter out personal emails

**Node Details:**  
- **Filter out personal emails**  
  - Type: Filter node  
  - Configuration: Checks invitee email against a list of common personal domains using multiple "notContains" conditions  
  - Key Expression: `{{$json.payload.email}}`  
  - Input: Calendly Trigger output  
  - Output: Passes only emails not containing personal domains  
  - Edge Cases:  
    - Emails with uncommon personal domains may pass through  
    - If email field missing or malformed, filter may fail silently  
  - Version: v2

---

#### 2.3 Email Enrichment

**Overview:**  
Enriches the filtered email using Clearbit’s person enrichment API to retrieve detailed personal and employment information.

**Nodes Involved:**  
- Enrich email

**Node Details:**  
- **Enrich email**  
  - Type: Clearbit node  
  - Configuration: Enrich person resource by email from payload  
  - Expression: Email from `{{$json.payload.email}}`  
  - Credentials: Clearbit API key  
  - Input: Filter out personal emails output  
  - Output: Enriched person data including employment details  
  - Error Handling: Continues workflow on failure (`onError` set to `continueErrorOutput`)  
  - Edge Cases:  
    - API rate limiting or quota exceeded  
    - Email not found or no enrichment data available  
    - Network timeouts  
  - Version: v1

---

#### 2.4 Company Enrichment & CRM Lookup

**Overview:**  
If the enriched person data includes a company domain, this block enriches the company data and searches for the company in Hubspot CRM.

**Nodes Involved:**  
- If person has a company  
- Enrich company  
- Search company

**Node Details:**  
- **If person has a company**  
  - Type: If node  
  - Configuration: Checks if `employment.domain` from Clearbit person data is not null  
  - Expression: `{{$json.employment.domain}} != null`  
  - Input: Enrich email output  
  - Output:  
    - True: proceeds to company enrichment  
    - False: skips to lead upsert  
  - Edge Cases: Employment data missing or incomplete  
  - Version: v2

- **Enrich company**  
  - Type: Clearbit node  
  - Configuration: Enrich company resource by domain from person's employment domain  
  - Expression: `{{$json.employment.domain}}`  
  - Credentials: Clearbit API key  
  - Input: If person has a company (true branch)  
  - Output: Detailed company information  
  - Edge Cases: Similar to person enrichment, including no company data found  
  - Version: v1

- **Search company**  
  - Type: Hubspot node  
  - Configuration: Searches Hubspot CRM for company by domain, limit 1  
  - Expression: Domain from company enrichment output  
  - Credentials: Hubspot OAuth2 credentials  
  - Input: Enrich company output  
  - Output: Company search results (existing CRM record or empty)  
  - Edge Cases:  
    - API rate limiting  
    - No matching company in CRM  
  - Version: v2

---

#### 2.5 CRM Company Upsert

**Overview:**  
Based on whether the company exists in Hubspot, either creates a new company record or updates the existing one with enriched data.

**Nodes Involved:**  
- if company does not exist on CRM  
- Create company  
- Update company

**Node Details:**  
- **if company does not exist on CRM**  
  - Type: If node  
  - Configuration: Checks if search result is empty (no existing company)  
  - Expression: Checks if input JSON object is empty  
  - Input: Search company output  
  - Output:  
    - True: create new company  
    - False: update existing company  
  - Edge Cases: Empty or malformed search response  
  - Version: v2

- **Create company**  
  - Type: Hubspot node  
  - Configuration: Creates company resource with fields mapped from Clearbit company enrichment output (name, twitter bio, description, founded year, country, twitter handle, money raised, followers, domain, employee count)  
  - Credentials: Hubspot OAuth2  
  - Input: if company does not exist output (true)  
  - Output: Created company ID and data  
  - Edge Cases: API errors, validation errors on fields  
  - Version: v2

- **Update company**  
  - Type: Hubspot node  
  - Configuration: Updates company resource by ID with fields from Clearbit enrichment (similar to create but excludes some fields like year founded)  
  - Credentials: Hubspot OAuth2  
  - Input: if company does not exist output (false)  
  - Output: Updated company data  
  - Edge Cases: Invalid company ID, API errors  
  - Version: v2

---

#### 2.6 CRM Contact Upsert

**Overview:**  
Upserts the contact (lead) in Hubspot, associating it with the relevant company if found/created.

**Nodes Involved:**  
- Upsert contact  
- Upsert lead  
- Contact not found, do nothing

**Node Details:**  
- **Upsert contact**  
  - Type: Hubspot node  
  - Configuration: Upserts contact using enriched email, associates it with company ID (from created/updated company)  
  - Expression: Email from Enrich email node; company ID from prior node outputs  
  - Credentials: Hubspot OAuth2  
  - Input: Create company output (for new company)  
  - Output: Upserted contact data  
  - Edge Cases: Email missing, API errors, association errors  
  - Version: v2

- **Upsert lead**  
  - Type: Hubspot node  
  - Configuration: Upserts contact with enriched email and personal name data (first and last name) for leads without company  
  - Credentials: Hubspot OAuth2  
  - Input: If person has a company (false branch)  
  - Output: Upserted contact data  
  - Edge Cases: Missing names, email errors  
  - Version: v2

- **Contact not found, do nothing**  
  - Type: NoOp node  
  - Configuration: Does nothing; placeholder for contacts not qualifying for enrichment  
  - Input: Enrich email output (false branch in If person has a company)  
  - Version: v1

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                          | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                         |
|--------------------------------|---------------------|----------------------------------------|---------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Calendly Trigger               | Calendly Trigger    | Receive booking event                   | None                      | Filter out personal emails   |                                                                                                   |
| Filter out personal emails     | Filter              | Exclude personal emails                 | Calendly Trigger          | Enrich email                | Make sure to map the email field from the data your booking tool provides                         |
| Enrich email                  | Clearbit            | Enrich person data by email             | Filter out personal emails | If person has a company; Contact not found, do nothing |                                                                                                   |
| If person has a company        | If                  | Check if lead has company domain        | Enrich email              | Enrich company; Upsert lead |                                                                                                   |
| Enrich company                | Clearbit            | Enrich company data                      | If person has a company   | Search company              | Map all data found about the company that you interested in                                      |
| Search company                | Hubspot             | Search company in Hubspot CRM            | Enrich company            | if company does not exist on CRM |                                                                                                   |
| if company does not exist on CRM | If                  | Check if company exists in CRM           | Search company            | Create company; Update company |                                                                                                   |
| Create company                | Hubspot             | Create new company in CRM                | if company does not exist on CRM | Upsert contact              |                                                                                                   |
| Update company                | Hubspot             | Update existing company in CRM           | if company does not exist on CRM |                             |                                                                                                   |
| Upsert contact               | Hubspot             | Upsert contact with company association | Create company            |                             |                                                                                                   |
| Upsert lead                  | Hubspot             | Upsert lead without company             | If person has a company (false branch) |                             |                                                                                                   |
| Contact not found, do nothing | NoOp                | Graceful handling if no enrichment      | Enrich email (false branch) |                             |                                                                                                   |
| Sticky Note                  | Sticky Note         | Setup instructions                      | None                      | None                        | 1. Add `Clearbit`, `Hubspot`, and `Calendly` credentials 2. Click on `Test workflow` 3. Book meeting on Calendly so the event starts the workflow |
| Sticky Note1                 | Sticky Note         | Booking tool replacement suggestion    | None                      | None                        | Replace this node with your booking tool of choice                                                |
| Sticky Note2                 | Sticky Note         | Company data mapping instruction       | None                      | None                        | Map all data found about the company that you interested in                                      |
| Sticky Note3                 | Sticky Note         | Email field mapping instruction         | None                      | None                        | Make sure to map the email field from the data your booking tool provides                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Calendly Trigger Node**  
   - Type: Calendly Trigger  
   - Configure to listen for event `invitee.created`  
   - Set up Calendly API credentials (OAuth2)  
   - Position: Start of workflow

2. **Add Filter Node "Filter out personal emails"**  
   - Type: Filter  
   - Condition: Email does NOT contain any of these domains: `@gmail.com`, `@yahoo.com`, `@outlook.com`, `@hotmail.com`, `@icloud.com`, `@mail.com`, `@aol.com`, `@zoho.com`, `@gmx`  
   - Use expression: `{{$json.payload.email}}` for email field  
   - Connect output of Calendly Trigger to this filter

3. **Add Clearbit Node "Enrich email"**  
   - Type: Clearbit  
   - Resource: Person  
   - Email: `{{$json.payload.email}}` from Filter output  
   - Credentials: Clearbit API key  
   - Error handling: Set `onError` to continue (continueErrorOutput)  
   - Connect output of filter to this node

4. **Add If Node "If person has a company"**  
   - Type: If  
   - Condition: Check if `{{$json.employment.domain}}` is not null  
   - Connect output of "Enrich email" to this node

5. **Add Clearbit Node "Enrich company"**  
   - Type: Clearbit  
   - Resource: Company  
   - Domain: `{{$json.employment.domain}}` from "Enrich email" output  
   - Credentials: Clearbit API key  
   - Connect "true" branch of If node to this node

6. **Add Hubspot Node "Search company"**  
   - Type: Hubspot  
   - Operation: Search by domain  
   - Domain: `{{$json.domain}}` from "Enrich company" output  
   - Limit: 1  
   - Credentials: Hubspot OAuth2  
   - Connect output of "Enrich company" to this node

7. **Add If Node "if company does not exist on CRM"**  
   - Type: If  
   - Condition: Check if input JSON is empty (no matching company found)  
   - Connect output of "Search company" to this node

8. **Add Hubspot Node "Create company"**  
   - Type: Hubspot  
   - Resource: Company  
   - Map fields from "Enrich company" output, including:  
     - Name, Twitter Bio, Description, Founded Year, Country, Twitter Handle, Total Money Raised, Twitter Followers, Domain, Number of Employees  
   - Credentials: Hubspot OAuth2  
   - Connect "true" branch of If node to this node

9. **Add Hubspot Node "Update company"**  
   - Type: Hubspot  
   - Resource: Company  
   - Operation: Update  
   - Company ID from CRM search result  
   - Map fields as in Create, excluding founded year  
   - Credentials: Hubspot OAuth2  
   - Connect "false" branch of If node to this node

10. **Add Hubspot Node "Upsert contact"**  
    - Type: Hubspot  
    - Operation: Upsert contact by email  
    - Email: `{{$('Enrich email').item.json.email}}`  
    - Associate with company ID from Create or Update company node output  
    - Credentials: Hubspot OAuth2  
    - Connect output of "Create company" to this node

11. **Add Hubspot Node "Upsert lead"**  
    - Type: Hubspot  
    - Operation: Upsert contact by email  
    - Email and Name fields from "Enrich email" output  
    - Credentials: Hubspot OAuth2  
    - Connect "false" branch of "If person has a company" node to this node

12. **Add NoOp Node "Contact not found, do nothing"**  
    - Type: NoOp  
    - Connect "false" branch of "Enrich email" (for any errors or missing data) to this node

13. **Add Sticky Notes**  
    - Add setup instructions  
    - Add notes on replacing the booking tool if desired  
    - Add reminders to map email and company fields carefully

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow credits to n8n community template for lead enrichment automation                                                | https://n8n.io/workflows/2129                                                                   |
| Clearbit API: Used for enrichment of contact and company data. Requires API key setup                                    | https://clearbit.com/docs                                                                       |
| Hubspot OAuth2 credentials needed for CRM search, create, update, and upsert operations                                  | https://developers.hubspot.com/docs/api/oauth                                                    |
| Calendly Trigger webhook requires setting webhook URL in Calendly developer settings                                    | https://developer.calendly.com/api-docs/v2#webhooks                                            |
| Personal email domains filtering may need adjustment depending on target audience                                        |                                                                                                |

---

This completes the detailed reference and reproduction guide for the "Enrich lead that booked a call on Calendly and save it on Hubspot" workflow.