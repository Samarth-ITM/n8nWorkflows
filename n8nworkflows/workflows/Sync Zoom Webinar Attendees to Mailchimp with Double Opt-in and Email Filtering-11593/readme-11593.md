Sync Zoom Webinar Attendees to Mailchimp with Double Opt-in and Email Filtering

https://n8nworkflows.xyz/workflows/sync-zoom-webinar-attendees-to-mailchimp-with-double-opt-in-and-email-filtering-11593


# Sync Zoom Webinar Attendees to Mailchimp with Double Opt-in and Email Filtering

### 1. Workflow Overview

This workflow automates the synchronization of Zoom webinar attendees into Mailchimp, implementing a double opt-in process and filtering out internal email addresses. It is designed for marketing teams who manage recurring webinars and want to automate lead management, contact updates, and tagging in Mailchimp.

The workflow logical structure includes these blocks:

- **1.1 Input Reception**: Manual trigger and setting webinar/occurrence IDs.
- **1.2 Data Retrieval from Zoom**: Fetch webinar registrants via Zoom API.
- **1.3 Data Extraction and Filtering**: Extract emails and filter out internal company addresses.
- **1.4 Mailchimp Contact Update or Creation**: Attempt to update existing Mailchimp members; if not found, create new members with double opt-in.
- **1.5 Lead Tagging in Mailchimp**: Tag new leads for marketing segmentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow manually and defines the input parameters required to query Zoom's API.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Type in IDs

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Config: No parameters, simply triggers the workflow manually.  
  - Input: None  
  - Output: Triggers "Type in IDs" node.  
  - Failure modes: None expected; user must initiate.

- **Type in IDs**  
  - Type: Set Node  
  - Role: Sets variables `webinar_id` and `occurence_id` for API calls.  
  - Config: Two string fields where user must input Zoom webinar series ID and occurrence ID.  
  - Expressions: None, static assigned strings by user.  
  - Input: Trigger from Manual Trigger  
  - Output: Passes IDs to the Zoom API request node.  
  - Failure modes: If empty or incorrect IDs are entered, subsequent API calls will fail or return empty.

---

#### 2.2 Data Retrieval from Zoom

**Overview:**  
Fetches registrants for the specified Zoom webinar and occurrence using Zoom's REST API.

**Nodes Involved:**  
- Get Webinar Attendees from Zoom

**Node Details:**  

- **Get Webinar Attendees from Zoom**  
  - Type: HTTP Request  
  - Role: Calls Zoom API endpoint for webinar registrants with pagination size 300.  
  - Config: URL built dynamically using `webinar_id` and `occurence_id` from input JSON; OAuth2 credential for Zoom API authentication.  
  - Expressions: URL uses `={{ $json.webinar_id }}` and `={{ $json.occurence_id }}` to inject variables.  
  - Input: Receives webinar and occurrence IDs from "Type in IDs" node.  
  - Output: JSON response with registrants data, passed to email extraction.  
  - Failure modes: OAuth2 token expiration, API rate limits, invalid IDs causing 404 or empty results.  
  - Version: Uses n8n HTTP Request version 4.2 with OAuth2 credential type "zoomOAuth2Api".

---

#### 2.3 Data Extraction and Filtering

**Overview:**  
Extracts email addresses from Zoom registrants and filters out internal company emails based on domain.

**Nodes Involved:**  
- Extract Registrant Emails  
- Filter Out Internal Emails

**Node Details:**  

- **Extract Registrant Emails**  
  - Type: Code (JavaScript)  
  - Role: Iterates over all Zoom API response items, extracts all registrant emails into individual items for further processing.  
  - Config: JS code loops through `$input.all()`, accesses `item.json.registrants`, collects emails, returns array of objects with `email` property.  
  - Input: Receives Zoom API response JSON array.  
  - Output: One item per email address extracted.  
  - Failure modes: Missing or malformed registrants array, no emails found resulting in empty output.  
  - Version: Code node v2.

- **Filter Out Internal Emails**  
  - Type: Filter  
  - Role: Removes any email addresses containing the internal domain `@yourcompanydomain.com`.  
  - Config: Condition uses "string does not contain" operator on `{{$json.email}}` with value `"@yourcompanydomain.com"` (case-sensitive).  
  - Input: List of emails from Extract Registrant Emails node.  
  - Output: Filtered list excluding internal emails.  
  - Failure modes: Case sensitivity may cause misses if internal emails have different casing; domain hardcoded and must be customized for user environment.  
  - Version: Filter node v2.2.

---

#### 2.4 Mailchimp Contact Update or Creation

**Overview:**  
Attempts to update existing Mailchimp members with attendee emails; if the member does not exist, creates them via double opt-in using hashed email.

**Nodes Involved:**  
- Update a member  
- If ID doesn't exist  
- MD5 Hash Email  
- Mailchimp - Send Double Opt in Email

**Node Details:**  

- **Update a member**  
  - Type: Mailchimp (n8n native node)  
  - Role: Updates member data in Mailchimp list for each filtered email.  
  - Config: Operation "update" with list ID placeholder `LIST_ID_HERE`, email from `{{$json.email}}`, no additional fields updated. Auth: OAuth2.  
  - Input: Filtered emails from "Filter Out Internal Emails".  
  - Output: Mailchimp member data (including `id` if exists).  
  - Failure modes: OAuth2 token issues, non-existent members may result in no update, node is set to continue on error.  
  - Version: Mailchimp node v1.

- **If ID doesn't exist**  
  - Type: If  
  - Role: Checks if Mailchimp member `id` exists in response; if not, triggers creation flow.  
  - Config: Condition checks if `{{$json.id}}` does not exist.  
  - Input: Output from "Update a member".  
  - Output: True branch to email hashing and creation, false branch ignored.  
  - Failure modes: Expression error if JSON structure changes, missing `id` may cause false positives.

- **MD5 Hash Email**  
  - Type: Code (JavaScript)  
  - Role: Computes MD5 hash of lowercase email required for Mailchimp API member identification and double opt-in.  
  - Config: Uses Node.js `crypto` module; retrieves email from filtered emails node dynamically via expression `$('Filter Out Internal Emails').item.json.email.toLowerCase()`.  
  - Input: True branch from "If ID doesn't exist".  
  - Output: Object with `hash` property containing MD5 hash string.  
  - Failure modes: Email missing or malformed causes errors in hashing; requires Node.js crypto module availability.  
  - Version: Code node v2.

- **Mailchimp - Send Double Opt in Email**  
  - Type: HTTP Request  
  - Role: Creates new Mailchimp member with status set to "pending" to trigger double opt-in email.  
  - Config: PUT request to Mailchimp API with hashed email in URL, JSON body includes email address and status `"pending"`. Auth: HTTP Basic with credential id `"abcd1234fgh567"`.  
  - Input: MD5 hash from previous node; email accessed via expression from filtered emails.  
  - Output: Response from Mailchimp confirming member creation or error.  
  - Failure modes: HTTP errors, invalid API key, rate limits, incorrect list ID, batch size handling with batch size 10.  
  - Version: HTTP Request node v4.2.

---

#### 2.5 Lead Tagging in Mailchimp

**Overview:**  
Tags newly added Mailchimp contacts with a "Leads" tag for marketing segmentation.

**Nodes Involved:**  
- Mailchimp - Add Leads Tag

**Node Details:**  

- **Mailchimp - Add Leads Tag**  
  - Type: HTTP Request  
  - Role: Adds tag "Leads" to the Mailchimp member identified by MD5 hashed email.  
  - Config: POST request to Mailchimp API endpoint `/lists/LIST_ID_HERE/members/{hash}/tags`, JSON body specifies tag name and active status. Auth: HTTP Basic same as double opt-in node.  
  - Input: From "Mailchimp - Send Double Opt in Email" node output, uses hash from "MD5 Hash Email" node via expression.  
  - Output: API response on tagging success or failure.  
  - Failure modes: API limits, invalid hash, missing member, authentication errors.  
  - Version: HTTP Request node v4.2.

---

### 3. Summary Table

| Node Name                      | Node Type         | Functional Role                                  | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                                          |
|--------------------------------|-------------------|-------------------------------------------------|----------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger    | Starts the workflow manually                     | None                             | Type in IDs                    | Sync Zoom Webinar Attendees to Mailchimp; step 1: Manual trigger                                                                     |
| Type in IDs                    | Set               | Sets webinar_id and occurrence_id for API call | When clicking ‘Execute workflow’ | Get Webinar Attendees from Zoom | Sync Zoom Webinar Attendees to Mailchimp; step 2: Set IDs                                                                            |
| Get Webinar Attendees from Zoom | HTTP Request      | Fetches registrants from Zoom API                | Type in IDs                     | Extract Registrant Emails      | Sync Zoom Webinar Attendees to Mailchimp; step 3: Get webinar attendees from Zoom                                                    |
| Extract Registrant Emails      | Code              | Extracts emails from Zoom registrants            | Get Webinar Attendees from Zoom | Filter Out Internal Emails     | Sync Zoom Webinar Attendees to Mailchimp; step 4: Extract registrant emails                                                          |
| Filter Out Internal Emails     | Filter            | Removes internal company emails                   | Extract Registrant Emails       | Update a member                | Sync Zoom Webinar Attendees to Mailchimp; step 5: Filter out internal emails                                                         |
| Update a member                | Mailchimp         | Updates Mailchimp member if exists                | Filter Out Internal Emails      | If ID doesn't exist            | Sync Zoom Webinar Attendees to Mailchimp; step 6: Attempt update member in Mailchimp                                                 |
| If ID doesn't exist            | If                | Checks if member exists in Mailchimp              | Update a member                 | MD5 Hash Email                | Sync Zoom Webinar Attendees to Mailchimp; step 7: Check if ID exists                                                                 |
| MD5 Hash Email                | Code              | Computes MD5 hash of email for Mailchimp double opt-in | If ID doesn't exist            | Mailchimp - Send Double Opt in Email | Sync Zoom Webinar Attendees to Mailchimp; step 8: MD5 hash email                                                                     |
| Mailchimp - Send Double Opt in Email | HTTP Request      | Creates new Mailchimp member with pending status | MD5 Hash Email                 | Mailchimp - Add Leads Tag      | Sync Zoom Webinar Attendees to Mailchimp; step 9: Send double opt-in email                                                          |
| Mailchimp - Add Leads Tag      | HTTP Request      | Adds "Leads" tag to new Mailchimp member          | Mailchimp - Send Double Opt in Email | None                         | Sync Zoom Webinar Attendees to Mailchimp; step 10: Tag new contacts as Leads                                                        |
| Sticky Note                   | Sticky Note       | Documentation block for workflow steps            | None                           | None                          | ## Sync Zoom Webinar Attendees to Mailchimp [Full step description inside]                                                           |
| Sticky Note1                  | Sticky Note       | Overview, use case, instructions, and help links | None                           | None                          | ## Sync Zoom Webinar Attendees to Mailchimp (workflow intro, usage, requirements, and support links including https://community.n8n.io/u/easy8.ai and https://www.youtube.com/@easy8ai) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No configuration needed.

2. **Create Set Node for IDs**  
   - Name: `Type in IDs`  
   - Type: Set  
   - Add two string fields:  
     - `webinar_id` (value: user inputs Zoom webinar series ID)  
     - `occurence_id` (value: user inputs Zoom occurrence ID)  
   - Connect Manual Trigger output to this node.

3. **Create HTTP Request Node to fetch Zoom Webinar Attendees**  
   - Name: `Get Webinar Attendees from Zoom`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.zoom.us/v2/webinars/{{$json.webinar_id}}/registrants?occurrence_id={{$json.occurence_id}}&page_size=300`  
   - Authentication: OAuth2 credential configured for Zoom (`zoomOAuth2Api`)  
   - Connect output of `Type in IDs` node to this node.

4. **Create Code Node to extract emails**  
   - Name: `Extract Registrant Emails`  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const items = $input.all();
     const emails = [];
     for (const item of items) {
       const registrants = item.json.registrants || [];
       for (const registrant of registrants) {
         if (registrant.email) {
           emails.push(registrant.email);
         }
       }
     }
     return emails.map(email => ({ json: { email } }));
     ```  
   - Connect output of `Get Webinar Attendees from Zoom` node.

5. **Create Filter Node to exclude internal emails**  
   - Name: `Filter Out Internal Emails`  
   - Type: Filter  
   - Condition:  
     - Field: `{{$json.email}}`  
     - Operation: "does not contain"  
     - Value: `@yourcompanydomain.com` (replace with your internal domain)  
     - Case sensitive: true  
   - Connect output of `Extract Registrant Emails` node.

6. **Create Mailchimp Node to update members**  
   - Name: `Update a member`  
   - Type: Mailchimp (built-in)  
   - Operation: Update  
   - List ID: Enter your Mailchimp List ID in place of `LIST_ID_HERE`  
   - Email: `{{$json.email}}`  
   - Authentication: OAuth2 configured with Mailchimp account  
   - Connect output of `Filter Out Internal Emails` node.  
   - Set on error to "Continue Regular Output" to handle non-existent members gracefully.

7. **Create If Node to check if member exists**  
   - Name: `If ID doesn't exist`  
   - Type: If  
   - Condition: Check if `{{$json.id}}` does not exist (operator: "not exists")  
   - Connect output of `Update a member` node.

8. **Create Code Node to MD5 hash email**  
   - Name: `MD5 Hash Email`  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const crypto = require('crypto');
     const email = $('Filter Out Internal Emails').item.json.email.toLowerCase();
     const hash = crypto.createHash('md5').update(email).digest('hex');
     return { hash };
     ```  
   - Set mode to "Run Once For Each Item"  
   - Connect true branch output of `If ID doesn't exist` node.

9. **Create HTTP Request Node to send double opt-in email**  
   - Name: `Mailchimp - Send Double Opt in Email`  
   - Type: HTTP Request  
   - Method: PUT  
   - URL: `https://us4.api.mailchimp.com/3.0/lists/LIST_ID_HERE/members/{{$json.hash}}`  
   - Authentication: HTTP Basic Auth credential with Mailchimp API key (create and select credential)  
   - Body: JSON  
     ```json
     {
       "email_address": "{{ $('Filter Out Internal Emails').item.json.email }}",
       "status_if_new": "pending",
       "status": "pending"
     }
     ```  
   - Connect output of `MD5 Hash Email` node.

10. **Create HTTP Request Node to add "Leads" tag**  
    - Name: `Mailchimp - Add Leads Tag`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://us4.api.mailchimp.com/3.0/lists/LIST_ID_HERE/members/{{ $('MD5 Hash Email').item.json.hash }}/tags`  
    - Authentication: Same HTTP Basic Auth as previous node  
    - Body JSON:  
      ```json
      {
        "tags": [
          {
            "name": "Leads",
            "status": "active"
          }
        ]
      }
      ```  
    - Connect output of `Mailchimp - Send Double Opt in Email` node.

11. **Add Sticky Notes**  
    - Create two sticky notes with the provided content describing workflow overview, use case, steps, and help links for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                               | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow pulls Zoom webinar attendee data, filters internal emails, updates or creates Mailchimp contacts with double opt-in, and tags new leads automatically. Ideal for marketing teams managing webinars and email lists.                                                                | Workflow purpose summary                                                                           |
| Replace placeholder values: `LIST_ID_HERE` with your Mailchimp List ID, `@yourcompanydomain.com` with your internal email domain, and enter valid Zoom webinar and occurrence IDs.                                                                                                        | Configuration instructions                                                                        |
| Requires Zoom and Mailchimp accounts with API access and appropriate OAuth2 or HTTP Basic credentials configured in n8n.                                                                                                                                                                | Credential setup requirements                                                                     |
| For help and community support: n8n community user "easy8.ai" at https://community.n8n.io/u/easy8.ai; Easy8.ai team contact; YouTube channel at https://www.youtube.com/@easy8ai                                                                                                         | Support and resources                                                                             |
| This workflow uses batch processing with batch size 10 for Mailchimp API calls to handle rate limits. Adjust batch size if necessary.                                                                                                                                                    | Performance and API rate limit consideration                                                     |

---

**Disclaimer:** The provided text is extracted from an automated n8n workflow. All operations comply with current content policies. Data processed is legal and publicly accessible.