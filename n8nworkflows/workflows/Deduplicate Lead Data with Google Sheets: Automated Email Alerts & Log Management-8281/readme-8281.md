Deduplicate Lead Data with Google Sheets: Automated Email Alerts & Log Management

https://n8nworkflows.xyz/workflows/deduplicate-lead-data-with-google-sheets--automated-email-alerts---log-management-8281


# Deduplicate Lead Data with Google Sheets: Automated Email Alerts & Log Management

### 1. Workflow Overview

This workflow automates the intake, deduplication, and CRM synchronization of lead data submitted via Google Forms and recorded in Google Sheets. Its main purpose is to ensure that new leads are properly added to the CRM (GoHighLevel) while preventing duplicate contacts by checking existing lead records. It also maintains detailed logging of both new and duplicate leads for audit and analytics purposes.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception**: Detects new lead submissions from Google Forms via Google Sheets trigger.
- **1.2 Duplicate Detection**: Searches the master lead database to identify if the lead is new or duplicate.
- **1.3 Decision Routing**: Routes processing based on duplication status.
- **1.4 New Lead Handling**: Creates new contacts in GoHighLevel and logs the new lead data.
- **1.5 Duplicate Lead Handling**: Updates existing contacts in GoHighLevel and logs duplicate lead submissions.
- **1.6 Data Processing and Logging**: Processes API responses and manages detailed logging of lead activities.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block continuously monitors a Google Sheets spreadsheet linked to a Google Form for new lead submissions. It triggers the workflow automatically every minute when new data is detected.

- **Nodes Involved:**  
  - Google Sheets Trigger1

- **Node Details:**  
  - **Google Sheets Trigger1**  
    - Type: Trigger Node (Google Sheets Trigger)  
    - Configuration: Polls a specific Google Sheets document (via URL) every minute, monitoring the sheet where Google Form responses are recorded.  
    - Key Variables: Captures all form submission fields such as First Name, Last Name, Email, Phone, Source, Lead Score, etc.  
    - Input: None (trigger node)  
    - Output: Emits new lead data on each trigger event  
    - Potential Failures: Authentication errors with Google Sheets API, rate limiting, or spreadsheet access permission issues.

---

#### 2.2 Duplicate Detection

- **Overview:**  
  Searches an existing leads database spreadsheet to find if the incoming lead’s email address already exists, serving as a duplicate detection mechanism.

- **Nodes Involved:**  
  - Lookup Lead1

- **Node Details:**  
  - **Lookup Lead1**  
    - Type: Data Query Node (Google Sheets)  
    - Configuration: Reads the master lead database spreadsheet and searches for an existing email matching the incoming lead’s email address. Uses the first sheet (gid=0) of a specified Google Sheets document.  
    - Key Expressions: Uses email address from the trigger node’s data to query the database.  
    - Input: Receives new lead data from Google Sheets Trigger1  
    - Output: Returns existing lead data if a match is found, empty if no match.  
    - Potential Failures: API authentication issues, spreadsheet access, empty or malformed email fields.

---

#### 2.3 Decision Routing

- **Overview:**  
  Routes workflow execution based on whether the lead is new or duplicate by comparing emails from the lookup result.

- **Nodes Involved:**  
  - Check from the data base?1  
  - Check the duplication

- **Node Details:**  
  - **Check from the data base?1**  
    - Type: Conditional Logic Node (If Node)  
    - Configuration: Checks if the email from the incoming lead is different from the email found in the lookup result to decide if the lead is new or duplicate.  
    - Input: Leads data from Lookup Lead1  
    - Output: Two branches — True (duplicate lead), False (new lead)  
    - Potential Failures: Expression evaluation errors, null or missing email fields.

  - **Check the duplication**  
    - Type: Filter Node  
    - Configuration: Applies an additional validation filter to confirm the email is truly new before proceeding to create a new contact in CRM.  
    - Input: New lead data from Check from the data base?1 (false path)  
    - Output: Filtered data for new leads  
    - Potential Failures: Filtering errors, misconfiguration of filter conditions.

---

#### 2.4 New Lead Handling

- **Overview:**  
  Creates a new contact in GoHighLevel CRM for validated new leads and logs the new lead data into the master database.

- **Nodes Involved:**  
  - Create Contact (GHL)1  
  - Log New Lead1

- **Node Details:**  
  - **Create Contact (GHL)1**  
    - Type: HTTP Request Node  
    - Configuration: Sends a POST request to GoHighLevel API to create a new contact with form data including first name, last name, email, phone, source, custom fields (lead score, submission date), and tags. Uses predefined GoHighLevel API credentials.  
    - Input: Validated new lead data from Check the duplication filter  
    - Output: API response containing created contact details  
    - Potential Failures: API authentication failure, invalid data format, network issues, API rate limiting.

  - **Log New Lead1**  
    - Type: Google Sheets Node  
    - Configuration: Appends the new lead data along with the GoHighLevel contact ID to the master leads database spreadsheet, enabling future duplicate detection.  
    - Input: Response from Create Contact (GHL)1  
    - Output: Confirmation of data append operation  
    - Potential Failures: Google Sheets API errors, permission issues, data mapping errors.

---

#### 2.5 Duplicate Lead Handling

- **Overview:**  
  Updates existing contacts in GoHighLevel CRM when duplicate leads are detected and logs the duplicate activity.

- **Nodes Involved:**  
  - Update Contact (GHL)1  
  - Code  
  - Log Duplicate Lead1

- **Node Details:**  
  - **Update Contact (GHL)1**  
    - Type: HTTP Request Node  
    - Configuration: Sends an update request to GoHighLevel API to refresh existing contact information using email as identifier. Uses HTTP header authentication with GoHighLevel API credentials.  
    - Input: Duplicate lead data from Check from the data base?1 (true path)  
    - Output: API response with updated contact details  
    - Potential Failures: API authentication failure, contact not found, network issues.

  - **Code**  
    - Type: Code Node (JavaScript)  
    - Configuration: Processes the API response to extract contacts array, filters contacts matching a target email (hardcoded or from JSON), and prepares the data for logging.  
    - Input: Response from Update Contact (GHL)1  
    - Output: Filtered contact data for logging  
    - Potential Failures: JavaScript errors, missing or malformed JSON fields, empty contact arrays.

  - **Log Duplicate Lead1**  
    - Type: Google Sheets Node  
    - Configuration: Updates the existing lead record in a dedicated duplicates tracking spreadsheet by adding the latest submission timestamp and incrementing a counter to track duplicate occurrences.  
    - Input: Processed contact data from Code node  
    - Output: Confirmation of update operation  
    - Potential Failures: Google Sheets API errors, concurrency conflicts, data mapping issues.

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role               | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                    |
|------------------------|-------------------------|------------------------------|------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger1  | Google Sheets Trigger   | Lead Form Monitor             | None                   | Lookup Lead1               | Monitors Google Forms responses sheet for new submissions every minute. Triggers when new lead data is added. |
| Lookup Lead1           | Google Sheets           | Duplicate Detection Scanner   | Google Sheets Trigger1 | Check from the data base?1  | Searches the leads database spreadsheet to check if an email already exists.                                 |
| Check from the data base?1 | If Node              | Duplicate Decision Router     | Lookup Lead1           | Check the duplication, Update Contact (GHL)1 | Routes the workflow based on whether the lead is new or a duplicate.                                         |
| Check the duplication   | Filter                  | New Lead Validator            | Check from the data base?1 (false) | Create Contact (GHL)1      | Performs additional validation to ensure leads are truly new before CRM creation.                            |
| Create Contact (GHL)1   | HTTP Request            | New Contact Creator           | Check the duplication  | Log New Lead1              | Creates new contact in GoHighLevel CRM with form submission data.                                            |
| Log New Lead1           | Google Sheets           | New Lead Database Writer      | Create Contact (GHL)1  | None                      | Records new lead information in the leads database spreadsheet along with the GoHighLevel contact ID.        |
| Update Contact (GHL)1   | HTTP Request            | Duplicate Contact Updater     | Check from the data base?1 (true) | Code                       | Updates existing contact in GoHighLevel CRM with new information from the form.                              |
| Code                   | Code (JavaScript)       | Contact Data Processor        | Update Contact (GHL)1  | Log Duplicate Lead1        | Processes CRM response to extract and format contact information for logging.                               |
| Log Duplicate Lead1     | Google Sheets           | Duplicate Activity Logger     | Code                   | None                      | Updates existing lead record to track duplicate submissions, timestamps, and counts.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Google Sheets Trigger** node named `Google Sheets Trigger1`.  
   - Configure it to monitor the Google Sheets document linked to your Google Form responses. Use the URL mode to specify the exact sheet.  
   - Set polling to every minute.  
   - Authenticate with Google Sheets OAuth2 credentials.  

2. **Add Duplicate Lookup Node:**  
   - Add a **Google Sheets** node named `Lookup Lead1`.  
   - Configure it to read the master lead database spreadsheet (documentId and sheetName).  
   - Set operation to search for leads by email address from the trigger node.  
   - Authenticate with Google Sheets OAuth2 credentials.  
   - Connect `Google Sheets Trigger1` output to this node.

3. **Add Decision Node:**  
   - Add an **If** node named `Check from the data base?1`.  
   - Configure condition: Compare incoming email (`Google Sheets Trigger1` email) with the email found in `Lookup Lead1`.  
   - Condition: If they are not equal (meaning no match), route to false (new lead); else true (duplicate lead).  
   - Connect `Lookup Lead1` output to this node.

4. **Add Filter for New Leads:**  
   - Add a **Filter** node named `Check the duplication`.  
   - Configure it to perform an additional email check to filter truly new leads.  
   - Connect the false (new lead) path from `Check from the data base?1` to this node.

5. **Create Contact in CRM:**  
   - Add an **HTTP Request** node named `Create Contact (GHL)1`.  
   - Set method to POST. URL: `https://rest.gohighlevel.com/v1/contacts/`.  
   - Set JSON body with dynamic expressions to include first name, last name, email, phone, source, custom fields (lead score, submission date), and tags.  
   - Use GoHighLevel API credentials.  
   - Connect output of `Check the duplication` to this node.

6. **Log New Lead:**  
   - Add a **Google Sheets** node named `Log New Lead1`.  
   - Configure it to append new rows to the master lead database spreadsheet, including contact info and GoHighLevel contact ID from the previous node's response.  
   - Authenticate Google Sheets OAuth2.  
   - Connect output of `Create Contact (GHL)1` to this node.

7. **Update Existing Contact for Duplicates:**  
   - Add an **HTTP Request** node named `Update Contact (GHL)1`.  
   - Configure it for updating contacts: Using GET or PUT depending on API, with URL constructed using email parameter.  
   - Use HTTP Header authentication with GoHighLevel API credentials.  
   - Connect the true (duplicate lead) path from `Check from the data base?1` to this node.

8. **Process Update Response:**  
   - Add a **Code** node named `Code`.  
   - Insert JavaScript to parse the contacts array from the previous node's response, filter contacts matching the target email, and prepare the data for logging.  
   - Connect output of `Update Contact (GHL)1` to this node.

9. **Log Duplicate Lead:**  
   - Add a **Google Sheets** node named `Log Duplicate Lead1`.  
   - Configure it to update an existing leads duplicate tracking spreadsheet: update timestamp and increment duplicate counter.  
   - Authenticate Google Sheets OAuth2.  
   - Connect output of `Code` node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow includes detailed sticky notes for each node explaining their type, purpose, and detailed description for better maintenance and understanding by operators.     | Embedded in the workflow as Sticky Note nodes adjacent to functional nodes.                        |
| Google Sheets Trigger polling is set to every minute to balance prompt lead intake and API usage limits.                                                                       | Consider adjusting polling frequency based on form submission volume and Google API quotas.       |
| GoHighLevel API requires valid API credentials and proper permission scopes for contact creation and update operations.                                                       | See GoHighLevel API documentation for authentication and endpoint details: https://rest.gohighlevel.com/ |
| The workflow assumes email is the unique identifier for leads and duplicates. If your use case requires other identifiers, adjust filter and condition expressions accordingly. | Customization might be needed for different CRM or lead management systems.                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.