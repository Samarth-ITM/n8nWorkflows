Google Sheets to SinergiaCRM: Automate Event Participant Registration

https://n8nworkflows.xyz/workflows/google-sheets-to-sinergiacrm--automate-event-participant-registration-8088


# Google Sheets to SinergiaCRM: Automate Event Participant Registration

### 1. Workflow Overview

This workflow automates the registration of event participants from a Google Sheet into SinergiaCRM. It listens for new rows added to a specified Google Sheet and processes only those marked for CRM import and not yet processed. The workflow checks if a contact exists in SinergiaCRM by matching the unique identification number (NIF). If the contact exists, it creates a relationship and event registration linked to that contact. If not, it creates the contact first, then proceeds with the relationship and registration. Finally, the workflow marks the row in Google Sheets as processed to avoid duplicates.

**Logical blocks:**

- **1.1 Input Reception and Filtering:** Detect new Google Sheet rows and filter by flags indicating if the row should be sent to CRM and if it was already processed.
- **1.2 Contact Lookup and Data Preparation:** Search for existing contact in SinergiaCRM by NIF and merge CRM data with the original input.
- **1.3 Conditional Processing Based on Contact Existence:** Branch workflow based on whether the contact exists.
- **1.4 Contact Creation and Relationship/Registration:** Create a new contact if none exists, then create relationships and registrations in SinergiaCRM.
- **1.5 Relationship and Registration for Existing Contacts:** If contact exists, directly create relationships and registrations.
- **1.6 Update Google Sheet Row Status:** Mark the processed row in Google Sheets as "Yes" under the Processed column to indicate completion.
- **1.7 Supporting Notes and Troubleshooting:** Several sticky notes provide setup instructions and troubleshooting tips.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Filtering

- **Overview:** Triggered by new rows added to a Google Sheet, this block filters incoming data to only process rows flagged for CRM and not yet processed.
- **Nodes Involved:** 
  - Google Sheets Trigger
  - IF: to CRM == Yes
  - IF: Not processed == No

- **Node Details:**

1. **Google Sheets Trigger**  
   - Type: Trigger node for Google Sheets  
   - Config: Listens for new rows added ("rowAdded") on a specific sheet (Sheet1) of a Google Sheet document with ID `1XRjRmPPv5ToZ5ZsFQyX1o5g8pWg-rod0yOmkKO-Ec3A`.  
   - Poll interval: every minute  
   - Credentials: OAuth2 for Google Sheets  
   - Input: N/A (trigger)  
   - Output: New row data with all columns  
   - Failures: Auth errors, network timeouts, sheet access issues  

2. **IF: to CRM == Yes**  
   - Type: Conditional node  
   - Config: Checks if the "To CRM" column equals "Yes" (case-sensitive, strict match)  
   - Input: Output of Google Sheets Trigger  
   - Output: Passes data forward only if condition met  
   - Failures: Expression evaluation errors if "To CRM" field missing  

3. **IF: Not processed == No**  
   - Type: Conditional node  
   - Config: Checks if "Processed" column equals "No" (case-sensitive, strict match)  
   - Input: Output of previous IF node  
   - Output: Passes data forward only if not processed  
   - Failures: Expression evaluation errors if "Processed" field missing  

---

#### 1.2 Contact Lookup and Data Preparation

- **Overview:** Searches SinergiaCRM Contacts module for existing contact by NIF and enriches input data with CRM contact ID and attributes.
- **Nodes Involved:** 
  - Find person by NIF
  - Combinar ID del CRM (Merge node)
  - Edit Fields (Set node)

- **Node Details:**

1. **Find person by NIF**  
   - Type: SinergiaCRM API node  
   - Config: Queries Contacts module filtering on `stic_identification_number_c` field equal to the NIF from the Google Sheet row  
   - Credentials: SinergiaCRM OAuth2  
   - Input: Filtered row data from input block  
   - Output: Contact data if found, empty otherwise  
   - Failures: API errors, auth failures, missing NIF field, no results  

2. **Combinar ID del CRM**  
   - Type: Merge node (Combine mode, enrichInput2)  
   - Config: Merges CRM contact data with original input using `stic_identification_number_c` and NIF as keys  
   - Input: Two inputs: (1) original filtered data, (2) CRM search results  
   - Output: Enriched data containing contact ID and attributes  
   - Failures: Merge mismatches if keys don’t align  

3. **Edit Fields**  
   - Type: Set node  
   - Config: Extracts and assigns key fields (First name, Last name, NIF, Email, Relation type, Relation date, Registration date, Event ID, id) from merged JSON for downstream use  
   - Input: Merged data  
   - Output: Cleaned and structured JSON for next decisions  
   - Failures: Missing properties if merge failed  

---

#### 1.3 Conditional Processing Based on Contact Existence

- **Overview:** Branches processing based on whether the contact exists in CRM. If contact found, proceeds to relationship and registration creation; if not, creates contact first.
- **Nodes Involved:** 
  - IF: Person exist

- **Node Details:**

1. **IF: Person exist**  
   - Type: Conditional node  
   - Config: Checks if the `id` field (contact ID) is non-empty in the prepared data  
   - Input: Output of Edit Fields  
   - Output: Two branches: True (contact exists), False (contact does not exist)  
   - Failures: Expression errors if `id` field missing or null  

---

#### 1.4 Contact Creation and Relationship/Registration (Contact does not exist branch)

- **Overview:** Creates a new contact in SinergiaCRM, then creates relationship and registration records for that contact.
- **Nodes Involved:** 
  - SinergiaCRM: Create Contact
  - SinergiaCRM: Create Relationship1
  - SinergiaCRM: Create Registration1
  - Google Sheets: Mark Procesado = Sí1

- **Node Details:**

1. **SinergiaCRM: Create Contact**  
   - Type: SinergiaCRM API node (create operation)  
   - Config: Creates a new Contact record with first name, last name, email, NIF (mapped to `stic_identification_number_c`), and identification type set to "nif"  
   - Credentials: SinergiaCRM OAuth2  
   - Input: Data from False branch of IF: Person exist  
   - Output: Newly created contact data including CRM contact ID  
   - Failures: Validation errors, auth errors, duplicate NIF handling  

2. **SinergiaCRM: Create Relationship1**  
   - Type: SinergiaCRM API node (create operation)  
   - Config: Creates a new relationship record linking the newly created contact (`id` from Create Contact) with relation type, start date from input data, assigned user id fixed to "2"  
   - Credentials: SinergiaCRM OAuth2  
   - Input: Output of Create Contact node  
   - Output: Relationship record data  
   - Failures: Missing contact ID, API errors  

3. **SinergiaCRM: Create Registration1**  
   - Type: SinergiaCRM API node (create operation)  
   - Config: Creates a registration record for the contact just created, linking to event (event ID taken from `Create Contact` node erroneously, see note below), setting participation type, attendees=1, status=confirmed, registration date from relation date field, assigned user id "2"  
   - Credentials: SinergiaCRM OAuth2  
   - Input: Output of Create Relationship1  
   - Output: Registration record data  
   - Failures: Incorrect event ID usage (see notes in troubleshooting), API errors  

4. **Google Sheets: Mark Procesado = Sí1**  
   - Type: Google Sheets update node  
   - Config: Updates the row matched by NIF, setting Processed column to "Yes" to mark completion  
   - Credentials: Google Sheets OAuth2  
   - Input: Output of Create Registration1  
   - Output: Updated Google Sheet row confirmation  
   - Failures: Sheet update errors, row mismatch, auth issues  

---

#### 1.5 Relationship and Registration for Existing Contacts (Contact exists branch)

- **Overview:** For contacts found in CRM, creates relationship and registration records linked to the existing contact, then marks Google Sheets row as processed.
- **Nodes Involved:** 
  - SinergiaCRM: Create Relationship
  - SinergiaCRM: Create Registration
  - Google Sheets: Mark Procesado = Sí

- **Node Details:**

1. **SinergiaCRM: Create Relationship**  
   - Type: SinergiaCRM API node (create operation)  
   - Config: Creates relationship record with start date, relationship type, contact ID from merged data, assigned user id "2"  
   - Credentials: SinergiaCRM OAuth2  
   - Input: True branch of IF: Person exist  
   - Output: Relationship record data  
   - Failures: Missing contact ID, API errors  

2. **SinergiaCRM: Create Registration**  
   - Type: SinergiaCRM API node (create operation)  
   - Config: Creates registration record linked to existing contact (ID from IF node), event ID from input data, participation type "attendant", status "confirmed", attendees 1, registration date from input, assigned user id "2"  
   - Credentials: SinergiaCRM OAuth2  
   - Input: Output of Create Relationship  
   - Output: Registration record data  
   - Failures: API errors, missing event ID  

3. **Google Sheets: Mark Procesado = Sí**  
   - Type: Google Sheets update node  
   - Config: Updates Processed column to "Yes" for matching row by NIF  
   - Credentials: Google Sheets OAuth2  
   - Input: Output of Create Registration  
   - Output: Confirmation of update  
   - Failures: Sheet update errors, row mismatch, auth issues  

---

#### 1.6 Supporting Notes and Troubleshooting

- **Sticky Note** nodes provide:
  - Workflow overview and key steps
  - Google Sheet columns required and their expected values
  - SinergiaCRM module and credential requirements
  - Conditional logic explanation for contact existence
  - Instructions on marking rows as processed to prevent duplicates
  - Troubleshooting tips including field name matching, NIF formatting, and API error checking

---

### 3. Summary Table

| Node Name                       | Node Type                      | Functional Role                      | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                      |
|--------------------------------|--------------------------------|------------------------------------|-------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Google Sheets Trigger           | n8n-nodes-base.googleSheetsTrigger | Detect new Google Sheet rows       | N/A                           | IF: to CRM == Yes                  | See Sticky Note1 (Google Sheet Setup)                                                           |
| IF: to CRM == Yes              | n8n-nodes-base.if               | Filter rows to process only if "To CRM" = Yes | Google Sheets Trigger          | IF: Not processed == No            | See Sticky Note1 (Google Sheet Setup)                                                           |
| IF: Not processed == No        | n8n-nodes-base.if               | Filter out rows already processed (Processed = No) | IF: to CRM == Yes             | Find person by NIF (main), Combinar ID del CRM (alt) | See Sticky Note1 (Google Sheet Setup)                                                           |
| Find person by NIF             | n8n-nodes-sinergiacrm.sinergiaCrm | Search existing contact by NIF in CRM | IF: Not processed == No        | Combinar ID del CRM                | See Sticky Note2 (SinergiaCRM Requirements)                                                     |
| Combinar ID del CRM            | n8n-nodes-base.merge            | Merge CRM contact data with input  | Find person by NIF, IF: Not processed == No | Edit Fields                       | See Sticky Note2 (SinergiaCRM Requirements)                                                     |
| Edit Fields                   | n8n-nodes-base.set              | Prepare and clean data for next steps | Combinar ID del CRM           | IF: Person exist                   | See Sticky Note3 (Contact Check Logic)                                                          |
| IF: Person exist              | n8n-nodes-base.if               | Branch if contact exists or not    | Edit Fields                   | SinergiaCRM: Create Relationship (true), SinergiaCRM: Create Contact (false) | See Sticky Note3 (Contact Check Logic)                                                          |
| SinergiaCRM: Create Contact   | n8n-nodes-sinergiacrm.sinergiaCrm | Create new contact if not found    | IF: Person exist (false)      | SinergiaCRM: Create Relationship1   | See Sticky Note2 (SinergiaCRM Requirements)                                                     |
| SinergiaCRM: Create Relationship | n8n-nodes-sinergiacrm.sinergiaCrm | Create relationship for existing contact | IF: Person exist (true)       | SinergiaCRM: Create Registration   | See Sticky Note2 (SinergiaCRM Requirements)                                                     |
| SinergiaCRM: Create Relationship1 | n8n-nodes-sinergiacrm.sinergiaCrm | Create relationship for new contact | SinergiaCRM: Create Contact   | SinergiaCRM: Create Registration1  | See Sticky Note2 (SinergiaCRM Requirements)                                                     |
| SinergiaCRM: Create Registration | n8n-nodes-sinergiacrm.sinergiaCrm | Create event registration for existing contact | SinergiaCRM: Create Relationship | Google Sheets: Mark Procesado = Sí | See Sticky Note2 (SinergiaCRM Requirements)                                                     |
| SinergiaCRM: Create Registration1 | n8n-nodes-sinergiacrm.sinergiaCrm | Create event registration for new contact | SinergiaCRM: Create Relationship1 | Google Sheets: Mark Procesado = Sí1 | See Sticky Note2 (SinergiaCRM Requirements)                                                     |
| Google Sheets: Mark Procesado = Sí | n8n-nodes-base.googleSheets    | Mark processed rows in Google Sheet | SinergiaCRM: Create Registration | N/A                               | See Sticky Note4 (Mark Row as Processed)                                                        |
| Google Sheets: Mark Procesado = Sí1 | n8n-nodes-base.googleSheets    | Mark processed rows in Google Sheet | SinergiaCRM: Create Registration1 | N/A                               | See Sticky Note4 (Mark Row as Processed)                                                        |
| Sticky Note                    | n8n-nodes-base.stickyNote       | Documentation                       | N/A                           | N/A                               | Workflow overview and key steps                                                                 |
| Sticky Note1                   | n8n-nodes-base.stickyNote       | Documentation                       | N/A                           | N/A                               | Google Sheet Setup instructions                                                                 |
| Sticky Note2                   | n8n-nodes-base.stickyNote       | Documentation                       | N/A                           | N/A                               | SinergiaCRM modules and credential requirements                                                 |
| Sticky Note3                   | n8n-nodes-base.stickyNote       | Documentation                       | N/A                           | N/A                               | Contact existence logic explanation                                                             |
| Sticky Note4                   | n8n-nodes-base.stickyNote       | Documentation                       | N/A                           | N/A                               | Marking rows as processed in Google Sheets                                                      |
| Sticky Note5                   | n8n-nodes-base.stickyNote       | Documentation                       | N/A                           | N/A                               | Troubleshooting tips                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**  
   - Type: Google Sheets Trigger  
   - Event: "rowAdded"  
   - Document ID: `1XRjRmPPv5ToZ5ZsFQyX1o5g8pWg-rod0yOmkKO-Ec3A`  
   - Sheet Name: `gid=0` (Sheet1)  
   - Polling: Every minute  
   - Credentials: Google Sheets OAuth2 configured  

2. **Add IF node "IF: to CRM == Yes":**  
   - Condition: Check if field `To CRM` equals `"Yes"` (case-sensitive)  
   - Connect input from Google Sheets Trigger  

3. **Add IF node "IF: Not processed == No":**  
   - Condition: Check if field `Processed` equals `"No"` (case-sensitive)  
   - Connect input from "IF: to CRM == Yes" (true output)  

4. **Add SinergiaCRM node "Find person by NIF":**  
   - Module: Contacts  
   - Operation: List with filter  
   - Filter: `stic_identification_number_c` equals `{{$json.NIF}}`  
   - Credentials: SinergiaCRM OAuth2 configured  
   - Connect input from "IF: Not processed == No" (true output)  

5. **Add Merge node "Combinar ID del CRM":**  
   - Mode: Combine, Join Mode: enrichInput2  
   - Merge by fields: `attributes.stic_identification_number_c` (from CRM) and `NIF` (from Google Sheets)  
   - Connect inputs: (1) Output of "Find person by NIF", (2) Output of "IF: Not processed == No" (true output)  

6. **Add Set node "Edit Fields":**  
   - Assign fields from merged data: First name, Last name, NIF, Email, Relation type, Relation date, Registration date (set equal to Relation date), Event ID, id  
   - Connect input from "Combinar ID del CRM"  

7. **Add IF node "IF: Person exist":**  
   - Condition: Field `id` is not empty (contact exists)  
   - Connect input from "Edit Fields"  

8. **For existing contact branch (true):**  
   - Add SinergiaCRM node "SinergiaCRM: Create Relationship":  
     - Module: stic_Contacts_Relationships  
     - Operation: Create  
     - Data fields:  
       - start_date = `{{$json["Relation date"]}}`  
       - relationship_type = `{{$json["Relation type"]}}`  
       - stic_contacts_relationships_contactscontacts_ida = `{{$json.id}}`  
       - assigned_user_id = `"2"` (adjust if needed)  
     - Credentials: SinergiaCRM OAuth2  
     - Connect input from IF: Person exist (true output)  

   - Add SinergiaCRM node "SinergiaCRM: Create Registration":  
     - Module: stic_Registrations  
     - Operation: Create  
     - Data fields:  
       - stic_registrations_contactscontacts_ida = `{{$json.id}}` (from IF node)  
       - stic_registrations_stic_eventsstic_events_ida = `{{$json["Event ID"]}}`  
       - participation_type = `"attendant"`  
       - attendees = `1`  
       - assigned_user_id = `"2"`  
       - status = `"confirmed"`  
       - registration_date = `{{$json["Registration date"]}}`  
     - Credentials: SinergiaCRM OAuth2  
     - Connect input from Create Relationship  

   - Add Google Sheets node "Google Sheets: Mark Procesado = Sí":  
     - Operation: Update row  
     - Sheet and Document same as trigger  
     - Matching column: NIF  
     - Update column Processed = "Yes"  
     - Credentials: Google Sheets OAuth2  
     - Connect input from Create Registration  

9. **For contact not found branch (false):**  
   - Add SinergiaCRM node "SinergiaCRM: Create Contact":  
     - Module: Contacts  
     - Operation: Create  
     - Data fields:  
       - first_name = `{{$json["First name"]}}`  
       - last_name = `{{$json["Last name"]}}`  
       - email1 = `{{$json.Email}}`  
       - stic_identification_type_c = `"nif"`  
       - stic_identification_number_c = `{{$json.NIF}}`  
     - Credentials: SinergiaCRM OAuth2  
     - Connect input from IF: Person exist (false output)  

   - Add SinergiaCRM node "SinergiaCRM: Create Relationship1":  
     - Module: stic_Contacts_Relationships  
     - Operation: Create  
     - Data fields:  
       - start_date = `{{$json["Relation date"]}}` (from IF node)  
       - relationship_type = `{{$json["Relation type"]}}`  
       - stic_contacts_relationships_contactscontacts_ida = `{{$json.id}}` (from newly created contact)  
       - assigned_user_id = `"2"`  
     - Credentials: SinergiaCRM OAuth2  
     - Connect input from Create Contact  

   - Add SinergiaCRM node "SinergiaCRM: Create Registration1":  
     - Module: stic_Registrations  
     - Operation: Create  
     - Data fields:  
       - stic_registrations_contactscontacts_ida = `{{$json.id}}` (from Create Contact node)  
       - stic_registrations_stic_eventsstic_events_ida = `{{$json.id}}` **[Note: This uses contact ID instead of event ID, verify and correct if necessary]**  
       - participation_type = `"attendant"`  
       - attendees = `1`  
       - assigned_user_id = `"2"`  
       - status = `"confirmed"`  
       - registration_date = `{{$json["Relation date"]}}`  
     - Credentials: SinergiaCRM OAuth2  
     - Connect input from Create Relationship1  

   - Add Google Sheets node "Google Sheets: Mark Procesado = Sí1":  
     - Operation: Update row  
     - Sheet and Document same as trigger  
     - Matching column: NIF  
     - Update Processed = "Yes"  
     - Credentials: Google Sheets OAuth2  
     - Connect input from Create Registration1  

10. **Add Sticky Note nodes** for documentation and troubleshooting as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow listens for new Google Sheet rows and processes only those marked with To CRM = "Yes" and Processed = "No". | Workflow Overview (Sticky Note)                                                                   |
| Google Sheet must have columns: First name, Last name, NIF, Email, Relation type, Relation date, Event ID, Registration date, To CRM, Processed. | Google Sheet Setup (Sticky Note1)                                                                 |
| SinergiaCRM instance must have modules: Contacts, stic_Contacts_Relationships, stic_Registrations. Custom fields like stic_identification_number_c should exist. | SinergiaCRM Requirements (Sticky Note2)                                                           |
| IF node checks if contact exists by presence of `id`. Merging uses NIF as key.                                  | Contact Check Logic (Sticky Note3)                                                                 |
| Rows marked Processed = "Yes" after successful registration to prevent duplicates.                              | Mark Row as Processed (Sticky Note4)                                                              |
| Troubleshooting: Ensure spreadsheet column names match exactly; verify NIF formatting; confirm custom fields in CRM; check logs for API errors. | Troubleshooting Tips (Sticky Note5)                                                                |
| Assigned user ID hardcoded as "2" in relationship and registration nodes; change if necessary to reflect your CRM user. | SinergiaCRM Requirements (Sticky Note2)                                                           |
| Possible issue: In "SinergiaCRM: Create Registration1", event ID is set incorrectly to contact ID; verify and correct. | User should adjust manually during reproduction                                                    |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.