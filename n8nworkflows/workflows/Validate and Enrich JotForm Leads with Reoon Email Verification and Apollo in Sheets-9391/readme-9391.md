Validate and Enrich JotForm Leads with Reoon Email Verification and Apollo in Sheets

https://n8nworkflows.xyz/workflows/validate-and-enrich-jotform-leads-with-reoon-email-verification-and-apollo-in-sheets-9391


# Validate and Enrich JotForm Leads with Reoon Email Verification and Apollo in Sheets

### 1. Workflow Overview

This n8n workflow is designed to automate the capture, validation, and enrichment of leads submitted via a JotForm contact form. It targets use cases where lead quality and data enrichment are critical, such as sales outreach, marketing campaigns, and CRM data management. The workflow integrates form submissions with email verification and enrichment APIs, and stores results in Google Sheets for further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures new JotForm submissions as they occur.
- **1.2 Initial Data Storage:** Logs raw submission data into Google Sheets before validation.
- **1.3 Email Validation:** Sends submitted email addresses to the Reoon API to verify deliverability, spam status, and other quality metrics.
- **1.4 Validation Status Storage:** Updates the Google Sheet with email verification results.
- **1.5 Safety Filtering:** Filters contacts to pass only those emails verified as safe for further enrichment.
- **1.6 Contact Enrichment:** Queries the Apollo API to enrich safe contacts with professional information (LinkedIn, job title, company).
- **1.7 Enrichment Storage:** Updates the Google Sheet with enriched contact data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow whenever a new submission is received from a specified JotForm contact form.

- **Nodes Involved:**  
  - Trigger: JotForm Submission

- **Node Details:**  
  - **Trigger: JotForm Submission**  
    - *Type & Role:* JotForm webhook trigger node; listens for new form submissions.  
    - *Configuration:* Monitors a specific JotForm form identified by form ID `252806830916461`. Requires JotForm API credentials.  
    - *Expressions / Variables:* Extracts all submitted form fields such as email, phone, first and last names, and message.  
    - *Inputs:* None (trigger).  
    - *Outputs:* Passes submission data downstream.  
    - *Edge Cases / Failures:* Potential webhook misconfiguration, invalid form ID, or credential expiration. Network failures at submission time.  
    - *Sub-workflow:* None.

---

#### 1.2 Initial Data Storage

- **Overview:**  
  Stores the raw submission data into a Google Sheet to ensure all leads are captured before validation.

- **Nodes Involved:**  
  - Sheets: Create Contact Record

- **Node Details:**  
  - **Sheets: Create Contact Record**  
    - *Type & Role:* Google Sheets node; appends a new row with form data.  
    - *Configuration:* Appends data to a sheet named "Contact Form" within a Google Sheet document identified by ID `1RBZ_VcxxkNqU6weYv5OxX7OWj1Y46WLHM1sbLw-iCVc`. Columns mapped include email, phone, message, last_name, first_name, and multiple fields for validation/enrichment status (initially empty).  
    - *Expressions:* Values referenced from the trigger JSON, e.g., `{{$json['E-mail']}}`, `{{$json['Phone']}}`, etc.  
    - *Inputs:* From JotForm Submission node output.  
    - *Outputs:* Passes raw contact data downstream for email verification.  
    - *Edge Cases / Failures:* Google Sheets API rate limits, credential expiration, mismatched schema or sheet ID errors.  
    - *Sub-workflow:* None.

---

#### 1.3 Email Validation

- **Overview:**  
  Sends the submitted email address to the Reoon API to verify its deliverability, spam trap status, disposable nature, and other email quality metrics.

- **Nodes Involved:**  
  - API: Email Verification (Reoon)

- **Node Details:**  
  - **API: Email Verification (Reoon)**  
    - *Type & Role:* HTTP Request node; sends GET request to Reoon email verification API.  
    - *Configuration:* URL `https://emailverifier.reoon.com/api/v1/verify` with query parameters: `email` from the input JSON and `mode` set to "power" for comprehensive verification. Uses HTTP Query Authentication with credentials.  
    - *Expressions:* `email` parameter is dynamically set as `{{$json.email}}`.  
    - *Inputs:* From sheets node with raw contact data.  
    - *Outputs:* Returns detailed verification results including status, spam trap flags, deliverability, etc.  
    - *Edge Cases / Failures:* API rate limits, invalid API key, network timeouts, malformed email addresses.  
    - *Sub-workflow:* None.

---

#### 1.4 Validation Status Storage

- **Overview:**  
  Updates the original Google Sheet row for the contact with the email verification results received from Reoon.

- **Nodes Involved:**  
  - Sheets: Save Verification Status

- **Node Details:**  
  - **Sheets: Save Verification Status**  
    - *Type & Role:* Google Sheets node; updates the existing contact row with validation results.  
    - *Configuration:* Updates the same "Contact Form" sheet using `email` as the matching column. Columns updated include status, domain, spam trap flags, deliverability, disposable email status, and other verification fields. The `processed_at` column is set with a timestamp placeholder `"="` (likely to be replaced with a timestamp expression).  
    - *Expressions:* Values mapped from the Reoon API response JSON, e.g., `{{$json.status}}`, `{{$json.is_spamtrap}}`.  
    - *Inputs:* From Reoon API node.  
    - *Outputs:* Passes updated contact data downstream for filtering.  
    - *Edge Cases / Failures:* Sheet update conflicts, missing matching email, credential issues, API response inconsistencies.  
    - *Sub-workflow:* None.

---

#### 1.5 Safety Filtering

- **Overview:**  
  Filters out contacts whose email status is not marked as "safe", ensuring only verified and trustworthy emails proceed to enrichment.

- **Nodes Involved:**  
  - Filter: Safe Emails Only

- **Node Details:**  
  - **Filter: Safe Emails Only**  
    - *Type & Role:* If node; implements conditional logic.  
    - *Configuration:* Checks if the field `status` equals `"safe"`. Only those contacts pass to the next step.  
    - *Expressions:* Condition is `{{$json.status}} == "safe"`.  
    - *Inputs:* From Sheets: Save Verification Status node.  
    - *Outputs:*  
      - True branch: passes data to Apollo enrichment node.  
      - False branch: stops workflow for unsafe emails.  
    - *Edge Cases / Failures:* Missing or malformed status field, casing issues if not handled properly.  
    - *Sub-workflow:* None.

---

#### 1.6 Contact Enrichment

- **Overview:**  
  Enriches safe contacts by querying Apollo.io API to retrieve professional details such as job title, LinkedIn URL, and company information.

- **Nodes Involved:**  
  - API: Contact Enrichment (Apollo)

- **Node Details:**  
  - **API: Contact Enrichment (Apollo)**  
    - *Type & Role:* HTTP Request node; POST request to Apollo people matching API.  
    - *Configuration:* URL `https://api.apollo.io/api/v1/people/match` with query parameters including `email` (from input JSON), `reveal_personal_emails=false`, and `reveal_phone_number=false`. Sends headers to accept JSON and no-cache. Uses HTTP Header Authentication with configured Apollo credentials.  
    - *Expressions:* `email` parameter dynamically set as `{{$json.email}}`.  
    - *Inputs:* From Filter node true branch (safe emails only).  
    - *Outputs:* Returns detailed professional profile under `person` object.  
    - *Edge Cases / Failures:* API rate limits, invalid API key, no matching person found, network issues.  
    - *Sub-workflow:* None.

---

#### 1.7 Enrichment Storage

- **Overview:**  
  Updates the Google Sheet with enriched data from Apollo for each safe contact.

- **Nodes Involved:**  
  - Sheets: Save Enriched Data

- **Node Details:**  
  - **Sheets: Save Enriched Data**  
    - *Type & Role:* Google Sheets node; updates the "Contact Form" sheet by matching on `email`.  
    - *Configuration:* Updates columns such as enriched_firstname, enriched_lastname, enriched_linkedin_url, enriched_title, enriched_organization_name, and processed_at date (formatted as yyyy-MM-dd).  
    - *Expressions:* Values extracted from `person` object in the Apollo API response, e.g., `{{$json.person.first_name}}`.  
    - *Inputs:* From Apollo enrichment node.  
    - *Outputs:* Final node in the workflow; updates contact data with enriched information.  
    - *Edge Cases / Failures:* No matching email in sheet, partial enrichment data, credential issues, API inconsistencies.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                   | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                      |
|-------------------------------|---------------------------|---------------------------------|--------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------|
| Trigger: JotForm Submission    | JotForm Trigger           | Listen for JotForm submissions  | None                           | Sheets: Create Contact Record           | **Step 1: Form Submission** Triggers when someone submits your JotForm contact form. Make sure to update the form ID with your own JotForm ID. |
| Sheets: Create Contact Record  | Google Sheets (Append)    | Store raw submission data       | Trigger: JotForm Submission    | API: Email Verification (Reoon)         | **Step 2: Initial Storage** Creates a new row in Google Sheets with the basic form data. This ensures we capture all submissions before validation. |
| API: Email Verification (Reoon)| HTTP Request              | Validate email via Reoon API    | Sheets: Create Contact Record  | Sheets: Save Verification Status        | **Step 3: Email Verification** Sends email to Reoon API for comprehensive validation. Checks for deliverability, spam traps, disposable emails, and more. |
| Sheets: Save Verification Status| Google Sheets (Update)    | Update sheet with verification results | API: Email Verification (Reoon) | Filter: Safe Emails Only                | **Step 4: Safety Filter** Only emails marked as "safe" proceed to enrichment. This protects your sender reputation and ensures data quality. |
| Filter: Safe Emails Only       | If                        | Filter safe emails only         | Sheets: Save Verification Status| API: Contact Enrichment (Apollo)         |                                                                                                 |
| API: Contact Enrichment (Apollo)| HTTP Request             | Enrich contact via Apollo API   | Filter: Safe Emails Only       | Sheets: Save Enriched Data               | **Step 5: Contact Enrichment** Queries Apollo.io to find professional information including LinkedIn profile, job title, and company name. |
| Sheets: Save Enriched Data     | Google Sheets (Update)    | Store enriched contact data     | API: Contact Enrichment (Apollo)| None                                   |                                                                                                 |
| Sticky Note                   | Sticky Note               | Documentation / Instructions    | None                          | None                                   | ## 📋 JotForm Contact Validation & Enrichment This workflow automatically validates and enriches contact form submissions. [Contains setup instructions and links.] |
| Sticky Note1                  | Sticky Note               | Documentation                  | None                          | None                                   | **Step 1: Form Submission** Triggers when someone submits your JotForm contact form. Make sure to update the form ID with your own JotForm ID. |
| Sticky Note2                  | Sticky Note               | Documentation                  | None                          | None                                   | ⚠️ **IMPORTANT - Update These Settings:** Google Sheet ID, JotForm ID, API Credentials, Form Fields. Quick links to Google Sheet template, Reoon API, Apollo API. |
| Sticky Note3                  | Sticky Note               | Documentation                  | None                          | None                                   | **Step 2: Initial Storage** Creates a new row in Google Sheets with the basic form data. This ensures we capture all submissions before validation. |
| Sticky Note4                  | Sticky Note               | Documentation                  | None                          | None                                   | **Step 3: Email Verification** Sends email to Reoon API for comprehensive validation. Checks for deliverability, spam traps, disposable emails, and more. |
| Sticky Note5                  | Sticky Note               | Documentation                  | None                          | None                                   | **Step 4: Safety Filter** Only emails marked as "safe" proceed to enrichment. This protects your sender reputation and ensures data quality. |
| Sticky Note6                  | Sticky Note               | Documentation                  | None                          | None                                   | **Step 5: Contact Enrichment** Queries Apollo.io to find professional information including LinkedIn profile, job title, and company name. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: JotForm Submission**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Set the form ID to the ID of your contact form (e.g., `252806830916461`).  
   - This node listens for new form submissions.

2. **Add Google Sheets Node: Create Contact Record**  
   - Type: Google Sheets (Append operation)  
   - Connect input from JotForm Submission node.  
   - Set credentials to your Google Sheets OAuth2 account.  
   - Choose your Google Sheet document by ID (copy and update to your own).  
   - Select the sheet/tab named "Contact Form" (gid=0).  
   - Configure columns to append with form data fields: email, phone, message, last_name, first_name, plus all validation/enrichment fields initially empty.  
   - Use expressions like `{{$json['E-mail']}}`, `{{$json['Phone']}}`.

3. **Add HTTP Request Node: API Email Verification (Reoon)**  
   - Type: HTTP Request (GET)  
   - Connect input from Sheets: Create Contact Record.  
   - Set URL: `https://emailverifier.reoon.com/api/v1/verify`  
   - Add query parameters:  
     - `email` = `{{$json.email}}`  
     - `mode` = `"power"`  
   - Configure HTTP Query Authentication credentials with your Reoon API account.  
   - Ensure network access and API key validity.

4. **Add Google Sheets Node: Save Verification Status**  
   - Type: Google Sheets (Update operation)  
   - Connect input from API: Email Verification node.  
   - Use the same Google Sheet and sheet/tab as before.  
   - Use `email` as the matching column to update rows.  
   - Map verification fields from Reoon API response (status, is_spamtrap, is_catch_all, etc.) into corresponding columns.  
   - Set `processed_at` field (recommended to use expression like `{{$now.toISO()}}` or similar to store timestamp).  
   - Use the same Sheets OAuth2 credentials.

5. **Add If Node: Filter Safe Emails Only**  
   - Connect input from Sheets: Save Verification Status.  
   - Configure condition: `status` equals `"safe"`.  
   - True branch will continue to enrichment; false branch ends workflow for unsafe emails.

6. **Add HTTP Request Node: API Contact Enrichment (Apollo)**  
   - Connect input from If node (true branch).  
   - Set HTTP Request type POST.  
   - URL: `https://api.apollo.io/api/v1/people/match`  
   - Add query parameters:  
     - `email` = `{{$json.email}}`  
     - `reveal_personal_emails` = `false`  
     - `reveal_phone_number` = `false`  
   - Add header parameters:  
     - `Cache-Control: no-cache`  
     - `accept: application/json`  
   - Configure HTTP Header Authentication with Apollo API credentials.

7. **Add Google Sheets Node: Save Enriched Data**  
   - Connect input from API: Contact Enrichment node.  
   - Use same Google Sheet and sheet/tab.  
   - Use `email` as matching column.  
   - Map enrichment fields from Apollo response (`person.first_name`, `person.last_name`, `person.linkedin_url`, `person.title`, `person.organization.name`).  
   - Set `processed_at` field with current date formatted as `yyyy-MM-dd` using an expression like `{{$now.toFormat('yyyy-MM-dd')}}`.  
   - Use Sheets OAuth2 credentials.

8. **Add Sticky Notes for Documentation (Optional)**  
   - Add sticky notes with instructions, setup steps, and links for easy maintenance and onboarding.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Copy the Google Sheet template before use to maintain schema consistency and avoid data loss.                 | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1RBZ_VcxxkNqU6weYv5OxX7OWj1Y46WLHM1sbLw-iCVc/copy)                     |
| Signup and configure API credentials for JotForm, Reoon, and Apollo before running the workflow.              | JotForm Pricing: https://www.jotform.com/pricing/?partner=naveenchoudhary Reoon: https://emailverifier.reoon.com/ Apollo: https://www.apollo.io/ |
| Ensure your JotForm fields match exactly the field names used in the Google Sheets mapping to avoid errors.   | Maintain consistent naming conventions for form fields and sheet columns.                                                              |
| Monitor API rate limits and errors in Reoon and Apollo to prevent workflow failures.                           | Use n8n error handling features or retries if necessary.                                                                                |
| Use timestamps to track when each record was processed for audit and troubleshooting.                         | `processed_at` fields use ISO or formatted date strings for consistency.                                                               |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated n8n workflow export. All content complies with applicable content policies and contains no illegal or protected data. All data processed is legal and public.